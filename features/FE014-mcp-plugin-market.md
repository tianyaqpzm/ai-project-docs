# 特性定义：FE014-mcp-plugin-market

## 业务逻辑
用户可以通过前端界面查看系统支持的 MCP 服务端列表，并根据需要通过 Switch 开关启用或禁用特定的能力集。

## 涉及子工程
- **ms-java-biz**: 负责存储和管理插件元数据及状态。
- **ms-py-agent**: 负责根据启用状态连接 MCP 服务端。
- **ms-ng-view**: 提供展示和操作的 UI 界面。

## 接口变动
### ms-java-biz
- `GET /rest/biz/v1/mcp-plugins`: 获取插件列表。
- `PATCH /rest/biz/v1/mcp-plugins/{id}/toggle`: 切换启用状态。

### ms-py-agent
- 启动逻辑：从 `ms-java-biz` 获取列表并注册。

## 实施计划
详见 [mcp_plugin_market_plan.md](file:///Users/pei/.gemini/antigravity/brain/70596837-4653-41ff-9787-1b755c54b99a/mcp_plugin_market_plan.md)

## 架构落地建议（基于模式二的实现路径）
如果决定采用“前端勾选”的模式，在后端实现上需要考虑动态的会话（Session）与工具（Tools）组装。一个典型的全栈实现思路如下：

### 1. 元数据管理 (关系型数据库)
在主业务数据库（例如 PostgreSQL）中设计三张表：
- **ms_mcp_server_registry**：记录你系统支持的所有 MCP 列表（ID, 名称, SSE_URL, 描述）。
- **ms_mcp_tool_cache**：通过定时任务去这些 MCP 拉取并缓存 `tools/list` 的 Schema，避免每次对话都去现拉。
- **ms_user_preference**：记录用户 ID 和他们勾选启用的 `MCP_ID` 等配置信息。

### 2. 运行时工具动态注入
当用户发起对话请求时：
- 后端拦截请求，查询 `ms_user_preference`。
- 组装该用户启用的所有 MCP Tools Schema。
- 将这些 Schema 与用户的 Prompt 一起打包发给大模型。

### 3. 连接复用与路由 (缓存层)
因为对接的是远端 SSE，建立连接有开销。
- 可以在后端维护一个连接池来管理长连接。
- 当大模型返回 `tool_call` 时，后端根据工具名称，路由到对应的活跃 SSE Client，发送 JSON-RPC 请求执行动作，拿到结果后再塞回给大模型。

### 4. 技术细节：PostgreSQL UUID 映射方案 (Solution 1)
在 `ms-java-biz` 与 PostgreSQL 的交互中，针对 `UUID` 类型的处理采用了“全局 UUID 映射方案”：
- **DO 实体层**：统一使用 `java.util.UUID` 类型，确保与数据库 Schema 1:1 诚实对应。
- **全局配置**：在 `MybatisPlusConfig` 中通过 `ConfigurationCustomizer` 全局注册了 `PostgresUuidTypeHandler`。
- **自定义处理器**：`PostgresUuidTypeHandler` 继承自 `BaseTypeHandler<UUID>`，在参数绑定时显式指定 `java.sql.Types.OTHER`，解决了 PostgreSQL 严格类型校验导致的 `operator does not exist: uuid = character varying` 错误。
- **架构收益**：
    - **简洁性**：Repository 层无需手动进行 `toString()` 或 `UUID.fromString()` 转换，代码更纯粹。
    - **健壮性**：利用 MyBatis-Plus 的 `autoResultMap` 机制，自动处理查询结果的 UUID 封装。
    - **性能**：避免了不必要的字符串解析开销。

### 5. 技术细节：PostgreSQL JSONB 映射方案
在处理 `ms_mcp_server_registry` 表的 `config` 字段（JSONB 类型）时，遇到了 `BadSqlGrammarException`：
- **根因分析**：MyBatis-Plus 默认的 `JacksonTypeHandler` 将 `JsonNode` 序列化为字符串并使用 `ps.setString()`。PostgreSQL 对 `JSONB` 字段有严格要求，不接受 `VARCHAR` 类型，必须显式指定为 `Types.OTHER`。
- **解决方案**：自定义 `PostgresJsonbTypeHandler` 继承自 `JacksonTypeHandler`，重写 `setNonNullParameter`：
  ```java
  ps.setObject(i, toJson(parameter), java.sql.Types.OTHER);
  ```
- **配置方式**：在 `DO` 实体类的 `config` 字段上通过 `@TableField(typeHandler = PostgresJsonbTypeHandler.class)` 指定。

### 6. 技术细节：MyBatis-Plus 扫描冲突处理
在启用全局 `UUID` 映射而引入 `MybatisPlusConfig` 时，导致了 `ConflictingBeanDefinitionException`：
- **根因分析**：递归通配符扫描 `**.mapper` 过于宽泛，将 Spring Data MongoDB 的 Repository (`MongoEventMapper`) 也误识别为 MyBatis Mapper，导致 Bean 定义冲突。
- **解决方案**：在 `@MapperScan` 配置中增加注解过滤，仅扫描标记了 `@Mapper` 的接口：
  ```java
  @MapperScan(
      basePackages = "com.dark.aiagent.infrastructure.persistence.**.mapper",
      annotationClass = org.apache.ibatis.annotations.Mapper.class
  )
  ```
### 7. 方案对比与选型 (UUID 处理)

在开发过程中，针对 PostgreSQL 的 UUID 类型映射问题，我们对比了两种技术方案：

| 维度 | **方案 1：全局 UUID 映射 (当前采用)** | **方案 2：String 映射 + 手动转换** |
| :--- | :--- | :--- |
| **Java 类型 (DO)** | `java.util.UUID` | `java.lang.String` |
| **转换逻辑** | **框架自动处理** (MyBatis TypeHandler) | **业务手动处理** (Repository 层) |
| **代码整洁度** | **高**：Repository 无冗余转换逻辑 | **中**：充斥大量的 `toString`/`fromString` |
| **类型安全** | **强**：编译期保证符合 UUID 规范 | **弱**：String 可能包含非法 UUID 格式 |
| **DB 兼容性** | **原生支持**：通过 `Types.OTHER` 适配 PG | **需额外处理**：不加 Handler 会报类型不匹配 |
| **维护成本** | **低**：配置一次，全局生效 | **高**：每个表都需要维护转换逻辑 |

**选型结论：**
最终决定采用 **方案 1**。虽然方案 2 在某些旧框架中兼容性较好，但方案 1 能够使“基础设施层”真实反映“数据库 Schema”，且通过 MyBatis 全局配置解决了 `autoResultMap` 的识别问题。这符合 DDD（领域驱动设计）中 DO 应当作为数据库镜像的原则，极大提升了代码的可读性和可维护性。

---

## 架构决策：为什么采用 Java 后端代理模式？

在处理远程 SSE MCP 服务时，系统选择通过 `ms-java-biz` 进行代理转发，而非由 `ms-py-agent` 直接连接，基于以下核心考量：

1. **统一工具注册表 (Registry & Caching)**：
   - Java 后端作为“单一真理来源”，负责发现、验证并缓存所有 MCP 工具的 Schema。
   - Agent 启动时仅需一次请求即可获取全量已启用的工具集，避免了与数十个远端服务器分别握手带来的巨大延迟。

2. **安全隔离与凭证管理**：
   - 远程 MCP 访问所需的 API Key 或认证凭据统一加密存储在 Java 端数据库中。
   - 代理模式确保了敏感信息不会流向 Agent 层，实现了凭据的闭环管理。

3. **业务权限控制 (User Preference Integration)**：
   - 系统支持按用户粒度启用/禁用插件。Java 代理层可以方便地集成 `ms_user_preference`，确保 AI 只能调用当前用户授权范围内的工具。

4. **协议抽象与稳定性**：
   - 代理层屏蔽了底层传输协议（SSE, Stdio, HTTP）的差异，为 Agent 提供统一的内部 RPC 接口。
   - 集中式的连接池管理能够更高效地处理长连接的重连与保活逻辑。

5. **全链路审计与流控**：
   - 所有的工具执行请求都会经过 Java 代理层，便于进行统一的日志审计、成本计算以及针对外部 API 的频率限制（Rate Limiting）。

---

---

## 待办事项 (TODO)
- [ ] **自动化工具刷新机制**：当前在 `ms-java-biz` 完成 Schema 抓取并缓存后，`ms-py-agent` 无法实时感知。目前采用重启 `ms-py-agent` 的方式临时规避。后续需实现 Java 后端在同步完成后主动调用 Python 端的刷新接口（例如 `POST /rest/agent/v1/mcp/refresh`），实现工具集的“即插即用”和热更新。
- [ ] **用户权限与 X-User-Id 隔离**：当前 `ms-java-biz` 的 `listEnabled` 接口为了简化测试，暂时忽略了 `X-User-Id` 的校验，默认返回所有已启用的插件。后续需恢复基于用户偏好的过滤逻辑，并确保 `X-User-Id` 在多服务间链路透传的可靠性。

