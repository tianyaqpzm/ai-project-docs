# RCA: ms-java-biz 启动时 schema 相对路径刷新导致 MCP 发现失败

## 1. 故障现场与现象

在 `ms-java-biz` 启动时，控制台输出以下错误日志：
```
2026-05-17T12:54:26.768Z ERROR 38323 --- [ms-java-biz] [ctor-http-nio-2] c.d.a.i.mcp.McpSchemaFetcher             : Failed to fetch schema for server f08f8942-5914-49b8-86a6-325c73ff1563: POST failed: <!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
```
由于此错误，内置插件 `java-biz`（UUID: `f08f8942-5914-49b8-86a6-325c73ff1563`）在启动时无法成功拉取工具列表（Schema）并完成缓存。

---

## 2. 根因分析 (Root Cause Analysis)

### 2.1 相对路径处理错误
1. 本地内置的 `java-biz` 插件在数据库（`ms_mcp_server_registry` 表）中注册的配置为：
   `{"url": "/mcp/sse", "messages_url": "/mcp/messages"}`
   这里的 `url` 为相对路径 `"/mcp/sse"`。
2. 应用启动时，会通过 `McpSchemaFetcher.fetchAndCache` 异步刷新所有启用的 SSE 插件 Schema：
   - 针对以 `/` 开头的相对路径，`McpSchemaFetcher` 会补全为绝对路径 `finalUrl = "http://localhost:8080/mcp/sse"`，并通过 `vanillaWebClientBuilder` 克隆出带有 `baseUrl(finalUrl)` 的 `client` 实例来发起 GET 请求建立 SSE 连接。
   - 当接收到 SSE 返回的 `endpoint` 事件时，事件数据 `data` 包含了真正的消息接收端点（例如 `"/mcp/messages?sessionId=..."`）。
   - 此时，代码调用 `resolveUrl(sseUrl, data)` 将端点解析为绝对路径。
   - **关键 Bug 在这里**：代码传入 `resolveUrl` 的第一个参数是相对路径 `sseUrl` (`"/mcp/sse"`)，而非补全后的绝对路径 `finalUrl`。
3. `new URI("/mcp/sse").resolve("/mcp/messages?sessionId=...")` 解析得到的返回结果仍然是相对路径 `"/mcp/messages?sessionId=..."`。

### 2.2 默认网络路由回退
1. 在随后的 `sendPostRequest(url, ...)` 中，`url` 为相对路径 `"/mcp/messages?sessionId=..."`。
2. 该方法内部通过 `vanillaWebClientBuilder.clone().build()` 创建了一个**不带 `baseUrl`** 的全新 WebClient 实例，并直接对这个相对路径发起 POST 请求。
3. Reactor Netty 面对一个非绝对的相对 URL 时，由于没有配置 `baseUrl` 且缺少主机名，在执行发送时回退到了本地的默认 HTTP 端口（通常是 macOS 内置且监听的 80 端口，这里运行着 macOS 默认的 Apache 服务器）。
4. 本地 Apache 服务并没有处理 `/mcp/messages` 的逻辑，因此直接返回了一个 404/405 的标准 Apache HTML 报错页面：
   `<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">...`
5. 最终，WebClient 将该 HTML 作为错误信息抛出，导致刷新 Schema 宣告失败。

---

## 3. 解决方案 (Resolution)

为了避免硬编码并增强配置的灵活性，采取了配置属性化的方案进行重构：

### 3.1 引入 `app.mcp` 相关配置
在 `application.yaml` 中新增了 `app.mcp.local-server-base-url` 属性，默认值回退到本服务的监听端口：
```yaml
app:
  mcp:
    local-server-base-url: ${LOCAL_SERVER_BASE_URL:http://localhost:${server.port}}
```

并在 `com.dark.aiagent.config` 下创建了配置类 `McpProperties`：
```java
@Component
@ConfigurationProperties(prefix = "app.mcp")
public class McpProperties {
    private String localServerBaseUrl;
    // getter / setter
}
```

### 3.2 注入并使用配置解析 URL
在 `McpSchemaFetcher.java` 与 `McpProxyService.java` 中，注入 `McpProperties` 并实现了一个健壮的 `resolveLocalUrl` 方法，用于消除任何硬编码的 `"http://localhost:"` 字符串：
```java
    private String resolveLocalUrl(String sseUrl) {
        if (!sseUrl.startsWith("/")) {
            return sseUrl;
        }
        String baseUrl = mcpProperties.getLocalServerBaseUrl();
        if (baseUrl == null || baseUrl.isEmpty()) {
            baseUrl = "http://localhost:" + serverPort;
        }
        if (baseUrl.endsWith("/")) {
            baseUrl = baseUrl.substring(0, baseUrl.length() - 1);
        }
        return baseUrl + sseUrl;
    }
```

### 3.3 修正基准路径参数
在 `McpSchemaFetcher.java` 的 `fetchAndCache` 中，调用 `resolveLocalUrl` 生成 `finalUrl`：
```java
    public void fetchAndCache(UUID serverId, String sseUrl) {
        final String finalUrl = resolveLocalUrl(sseUrl);
```
并在 SSE 事件监听的 `endpoint` 分支中，改用 `finalUrl` 绝对路径作为基准解析相对端点：
```java
                    if ("endpoint".equals(eventType) && data != null) {
                        String url = resolveUrl(finalUrl, data);
```

而在 `McpProxyService.java` 中，也通过 `resolveLocalUrl` 处理相对路径，彻底保持了两个核心组件的一致性。

---

## 4. 验证结果

1. **单元测试验证**：
   - 相应地更新了 `McpSchemaFetcherTest` 和 `McpProxyServiceTest`，Mock 并注入了 `McpProperties`。
   - 运行 `mvn test -Dtest=McpSchemaFetcherTest,McpProxyServiceTest` 成功，2 个测试全部 100% 运行通过，无任何 Failure/Error。
2. **逻辑推导与向前兼容**：
   - 远程绝对路径（如 `https://mcp-pg-sse.122577.xyz/sse`）：因为不以 `/` 开头，`resolveLocalUrl` 仍等于 `sseUrl`，原样向前兼容。
   - 本地相对路径（如 `/mcp/sse`）：`resolveLocalUrl` 会被动态补全为 `http://localhost:8080/mcp/sse`（或配置的自定义 URL），从而解析出正确的 `http://localhost:8080/mcp/messages?sessionId=...`，保障了内置插件在任意网络环境下的高可用与可配性。
