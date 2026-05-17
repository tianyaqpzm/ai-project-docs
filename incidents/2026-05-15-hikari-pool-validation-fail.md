# 故障分析报告 (RCA): HikariCP 数据库连接池验证失效

## 1. 故障描述
*   **故障现象**：Java 业务服务 (`ms-java-biz`) 日志中出现大量 `HikariPool-1 - Failed to validate connection ... (This connection has been closed.)` 警告。
*   **级联影响**：导致 Python Agent 服务在调用 Java 接口时出现 `httpx.ReadTimeout`。
*   **环境背景**：应用与 PostgreSQL 数据库通过 Tailscale VPN 跨网段连接。

## 2. 根因分析 (Root Cause)
1.  **VPN 链路回收机制 (Idle Pruning)**：
    中间网络设备（VPN 节点/NAT）为了节省状态表资源，对长时间无数据包的 TCP 连接进行了回收。经观察，其回收窗口约在 **15-20秒**。
2.  **HikariCP 心跳机制冲突**：
    HikariCP 规定 `keepalive-time` 的最小值必须为 **30,000ms (30秒)**。
    *   当配置值 < 30s 时，心跳功能被自动禁用（Disabled）。
    *   当配置值 = 30s 时，心跳发送频率（30s）依然落后于 VPN 的回收频率（20s）。
    这导致池中的空闲连接在心跳包还没发出前就已经被网络层物理切断。
3.  **验证成本与请求堆积**：
    当应用请求数据库时，HikariCP 尝试验证池中已损坏的连接，验证失败导致重试。在并发请求下，连接池线程被阻塞，导致 API 响应超时。

## 3. 解决方案 (Resolution)
1.  **实施 Zero-Idle 策略**：
    设置 `spring.datasource.hikari.minimum-idle=0`。不再试图在不稳定的链路上保留空闲连接，而是“随用随建”，彻底避免对旧连接的验证失败。
2.  **心跳加固**：
    将 `keepalive-time` 修正为 **30000** (30秒)，确保保活机制在合法范围内开启。
3.  **生命周期压缩**：
    将 `max-lifetime` 缩短至 **240000** (4分钟)，确保连接在可能出现逻辑失效前被主动回收。
4.  **强制驱动参数**：
    在 JDBC URL 中强制加入 `sslmode=require`、`tcpKeepAlive=true` 和 `connectTimeout=30`，提高传输层稳定性。

## 4. 预防措施 & 经验沉淀
*   **规范沉淀**：在 `.agent/rules/ai-code-ws.md` 中增加“跨网段/VPN 数据库连接配置标准”。
## 5. 衍生故障：SSE 长连接“不完整分块读取” (Incomplete Chunked Read)

### 5.1 故障描述
*   **故障现象**：Python Agent 报错 `httpx.RemoteProtocolError: peer closed connection without sending complete message body (incomplete chunked read)`。
*   **触发时机**：通常发生在 Agent 闲置或 VPN 网络出现瞬时抖动时。

### 5.2 根因分析
1.  **Tomcat 保活窗口过窄**：
    Spring Boot 默认的 Tomcat `keep-alive-timeout` 为 **20秒**。当网络由于 VPN 抖动导致 10s 的心跳包出现延迟时，Tomcat 极易先于心跳判定连接超时并主动关闭 TCP 窗口。
2.  **协议状态不一致**：
    Tomcat 关闭连接时，可能未能发送 SSE 所需的完整分块结束符，导致 Python `httpx` 客户端在解析流时认为数据被异常截断。

### 5.3 进阶加固方案
1.  **Tomcat 传输层调优** (`ms-java-biz`)：
    *   `server.tomcat.keep-alive-timeout=60000` (60s)：确保护航时间远大于应用心跳频率。
    *   `server.tomcat.max-keep-alive-requests=-1`：取消长连接请求数限制。
2.  **Python 客户端韧性增强** (`ms-py-agent`)：
    *   **自动重连 (Lazy Reconnect)**：在 `MCPToolRegistry` 中实现 `_ensure_connected` 逻辑，在每次工具调用前探测连接状态。
    *   **重试机制**：在 `call_tool` 中捕获 `RemoteProtocolError`，标记 Server 失效并立即触发一次“连接-重试”循环。
    *   **读取超时放宽**：设置 `sse_read_timeout=600.0`。
3.  **网关容错窗口调整** (`ms-java-gateway`)：
    *   将 Nacos 注册/配置超时统一提升至 **30秒**。
    *   将网关转发 `connect-timeout` 提升至 **10秒**。
