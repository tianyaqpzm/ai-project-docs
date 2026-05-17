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
- **动态自发现启动**：启动时调用 `ms-java-biz` 的 `/enabled` 接口，一键拉取并自动连接所有启用的 MCP Server（包括 Java 自身和第三方插件），实现“零配置”加载。

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

## 架构决策：为什么从“Java 代理”转向“直接连接 + 自发现”？

系统最初采用 Java 代理模式，但随着插件数量增加，为了追求更高的性能和更低的架构复杂度，现演进为**“直接连接 + 自发现 (Direct Connection & Self-Discovery)”**模式：

1. **统一自发现 (Unified Discovery)**：
   - `ms-java-biz` 仍作为配置中心，通过 `/rest/biz/v1/mcp-plugins/enabled` 接口下发所有活跃 MCP 服务端点。
   - **自发现注入**：Java 服务将自身的业务工具端点（`/mcp/sse`）也作为插件动态注入到列表中。

2. **高性能直连 (Direct Connection)**：
   - `ms-py-agent` 拿到列表后，直接与各个 MCP Server（包括 Java、FileSystem 等）建立 SSE/Stdio 连接。
   - **架构收益**：减少了 Java 层的中转开销，降低了工具调用的端到端延迟，同时减轻了 Java 服务的 IO 压力。

3. **单一注册中心 (Registry of Truth)**：
   - 所有的插件元数据、用户开启偏好均统一存储在 Java 端数据库中，Python 端不再持有任何硬编码的工具列表。

4. **安全与透传**：
   - 跨服务调用时，Agent 负责从 `RunnableConfig` 中提取 `Authorization` 头并透明透传给各个端点，确保安全性。

### 方案对比：Java 代理 vs. 直接连接

| 维度 | **方案 A：Java 代理模式 (初期)** | **方案 B：直接连接 + 自发现 (当前)** |
| :--- | :--- | :--- |
| **工具加载** | Java 聚合所有工具，Agent 仅连接 Java | Agent 动态获取列表，并行连接所有 Server |
| **性能延迟** | **高**：工具调用需经过 Java 转发 (Python -> Java -> MCP) | **低**：Agent 与 MCP Server 建立直连 (Python -> MCP) |
| **架构复杂度** | **高**：Java 需要实现复杂的异步代理和结果封装 | **低**：利用官方 MCP SDK 建立标准连接，逻辑更纯粹 |
| **扩展性** | **受限**：新增插件类型可能需要修改 Java 代理逻辑 | **极强**：支持 Stdio/SSE 等多种模式，Agent 自动适配 |
| **工具冗余** | **有**：Agent 会同时看到 Java 内部工具和其代理的工具 | **无**：每个工具均有唯一的来源，工具集更整洁 |
| **安全性** | Java 集中管理 Key，对 Agent 隐藏端点 | Agent 持有端点信息，需确保各端点间的鉴权透传 |

### 方案对比：Java 代理 vs. 直接连接 vs. 按需加载

| 维度 | **方案 A：Java 代理 (初期)** | **方案 B：直接连接 (V1)** | **方案 C：按需加载 (V2)** |
| :--- | :--- | :--- | :--- |
| **工具加载** | Java 聚合，Agent 仅连 Java | Agent 列表获取并直连 | **对话触发同步** |
| **性能延迟** | 高：Java 转发 | 低：Agent 直连 | **最低：无预加载压力** |
| **架构复杂度** | 高 | 低 | **极简：去生命周期阻塞** |
| **资源占用** | 内存与连接固定 | 高：连接常驻 | **优：仅活跃用户占用** |
| **工具冗余** | 有 | 无 | 无 |

---

## 架构演进：按需延迟加载 (On-Demand Lazy Loading)

为了追求极致的系统启动速度并支持用户偏好的实时生效，系统在“直接连接”模式的基础上进一步演进为**“按需延迟加载”**架构：

### 1. 核心变更：从“启动预加载”到“请求时同步”
- **移除 Lifespan 阻塞**：`ms-py-agent` 启动时不再尝试建立任何 MCP 连接。这使得 Agent 服务能够实现毫秒级启动，不再受远端 MCP 服务响应速度的影响。
- **对话触发同步**：在 `/chat` 接口的事件流（Event Stream）启动之初，Agent 会根据 `current_user.id` 异步调用 `sync_mcp_with_user` 逻辑。

### 2. 增量同步机制 (Incremental Sync)
- **幂等性维护**：`MCPToolRegistry` 经过重构，支持 `setup()` 方法的多次幂等调用。
- **连接复用**：对于已经在活跃连接池中的 Server，同步过程会直接跳过，保持连接稳定。
- **动态上下线**：
    - **新增**：如果用户新开启了某个插件，同步逻辑会立即建立新连接并注入工具。
    - **移除**：如果用户禁用了插件，同步逻辑会优雅地断开连接并从工具集中移除，实现真正的“热更新”。

### 3. 系统级插件保护 (System Plugin Guard)
- **逻辑下沉**：在 `ms-java-biz` 层，`listEnabled` 接口被赋予了识别“系统插件”的能力。
- **强制包含**：即使用户的偏好设置中未显式勾选（例如新用户），`system=true` 的插件（如 `java-biz` 本身）也会被强制返回。这确保了 Agent 始终拥有查订单、读知识库等核心业务能力。

---

### 方案演进对比：启动加载 vs. 按需加载

| 维度 | **V1：启动加载 (Lifespan)** | **V2：按需加载 (On-Demand)** |
| :--- | :--- | :--- |
| **启动耗时** | **长**：需等待所有 MCP Server 握手完成 | **极短**：0 毫秒 MCP 开销，秒开启动 |
| **生效时机** | 重启 Agent 后生效 | **即时**：点击发送消息后立即同步 |
| **资源占用** | **高**：未使用的插件连接也会一直占用 | **优**：仅在用户活跃时建立并复用连接 |
| **多租户支持** | **差**：全局配置，无法区分用户偏好 | **强**：基于 UserID 的个性化工具集 |

---

## 待办事项 (TODO)
- [x] **自动化工具刷新机制**：已通过“按需同步”逻辑彻底解决。现在每次对话开始前都会拉取最新插件状态，无需重启服务或手动刷新。
- [x] **用户权限与 X-User-Id 隔离**：已通过 Java 端的 `listEnabled(userId)` 过滤和 Python 端的 `sync_mcp_with_user(userId)` 实现。
- [ ] **连接生命周期管理 (TTL)**：目前连接在建立后会持续保持。未来需考虑在用户长时间不活跃后，自动断开并清理闲置连接以释放系统资源。
