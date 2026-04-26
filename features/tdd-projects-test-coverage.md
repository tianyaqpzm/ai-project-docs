# TDD: 全工程测试用例看护

## 背景

全系统 4 个子工程当前测试覆盖极其薄弱：

| 工程 | 角色 | 现有测试 | 风险等级 |
|------|------|---------|---------|
| api-gateway | 统一入口（认证/路由） | ❌ 零覆盖 | 🔴 极高 |
| ai-langchain4j | MCP Server（业务工具） | ⚠️ 7 个探索性测试 | 🟡 高 |
| python-agent | Agent Brain（编排/LLM） | ⚠️ 2 个验证脚本 | 🟡 高 |
| timekeeper | 前端 UI（Angular） | ❌ 零覆盖 | 🟠 中 |

## 测试策略总览

按「架构守护 + 核心功能」两个维度为每个工程规划用例。

---

## 🔵 工程一：api-gateway (Java / Spring WebFlux)

> **测试框架**: JUnit 5 + WebTestClient + spring-security-test

### 架构守护 (5 用例)

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| GA-01 | SecurityConfig Bean 成功创建 | 构造器注入 `IgnoreWhiteProperties`，`@Value` 属性可解析 |
| GA-02 | JwtAuthenticationFilter 注册为 GlobalFilter | `GlobalFilter` + `Ordered`，order = -100 |
| GA-03 | RedirectSaveFilter 注册为 WebFilter | `@Order(HIGHEST_PRECEDENCE)` |
| GA-04 | IgnoreWhiteProperties YAML list 绑定 | 绑定 7 个 URL pattern |
| GA-05 | 禁止 Servlet 栈 | classpath 不含 `DispatcherServlet` |

### 核心功能 (15 用例)

**JWT 认证过滤器 (8)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| GJ-01 | 白名单精确路径放行 | `/actuator/health` 不检查 Token |
| GJ-02 | 白名单通配符放行 | `/oauth2/**` 匹配 |
| GJ-03 | Cookie JWT → 放行 + X-User 头注入 | `X-User-Id`, `X-User-Name`, `X-User-Avatar` |
| GJ-04 | Bearer Header JWT → 放行 | 兜底策略（API 调用） |
| GJ-05 | 无 Token → 401 JSON | 包含 `url` 和 `message` |
| GJ-06 | 过期 Token → 401 | `exp` 在过去 |
| GJ-07 | 签名错误 → 401 | 不同 secret |
| GJ-08 | Claim 缺失 → 头为空字符串 | `name`/`picture` null |

**安全配置 (4)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| GS-01 | 白名单可匿名访问 | 返回非 401 |
| GS-02 | 非白名单返回 401 JSON | body 含 `url`, `message` |
| GS-03 | CSRF 已禁用 | POST 不需 CSRF Token |
| GS-04 | Session Cookie 名 = DARK_SESSION | 配置验证 |

**重定向过滤器 (3)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| GR-01 | OAuth2 + redirect → 存 Session | Session 有 `CUSTOM_REDIRECT_URI` |
| GR-02 | redirect 含 /logout → 跳过 | 不存 Session |
| GR-03 | 非 OAuth2 路径 → 放行 | 不操作 Session |

---

## 🟢 工程二：ai-langchain4j (Java / Spring MVC)

> **测试框架**: JUnit 5 + MockMvc + Mockito

### 架构守护 (4 用例)

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| LA-01 | McpController 自动装配所有 McpTool | `toolRegistry` size 与 Bean 数量一致 |
| LA-02 | McpTool 策略模式验证 | 所有 Tool 实现接口且有唯一 name |
| LA-03 | McpProtocol record DTO 序列化 | `JsonRpcResponse` JSON 输出包含 jsonrpc/id/result |
| LA-04 | MongoChatMemoryStore 实现 ChatMemoryStore | 接口契约验证 |

### 核心功能 (12 用例)

**MCP 协议 (6)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| LM-01 | GET /mcp/sse 返回 SSE endpoint 事件 | `event: endpoint`，data 含 `sessionId` |
| LM-02 | initialize → 返回 protocolVersion + serverInfo | 协议版本 `2024-11-05` |
| LM-03 | tools/list → 返回所有已注册工具 | 工具数量 ≥ 1 |
| LM-04 | tools/call 正确路由到目标 Tool | `query_order` 执行返回 ToolResult |
| LM-05 | tools/call 工具不存在 → 返回错误 | JSON-RPC error code |
| LM-06 | notifications/initialized → 无响应 | 静默处理 |

**业务工具 (3)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| LT-01 | OrderQueryTool 执行成功 | 传 orderId 返回 success ToolResult |
| LT-02 | OrderQueryTool 参数缺失 → 返回错误 | `isError = true` |
| LT-03 | ToolResult.success / .error 工厂方法 | 验证 content 结构 |

**知识库 API (3)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| LK-01 | GET /topics → 返回主题列表 | 200 + JSON array |
| LK-02 | POST /topics → 创建主题 | 自动生成 ID |
| LK-03 | DELETE /topics/{id} → 级联清理文档 | 主题和关联文档同时删除 |

---

## 🟡 工程三：python-agent (Python / FastAPI)

> **测试框架**: pytest + pytest-asyncio + httpx (TestClient)

### 架构守护 (4 用例)

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| PA-01 | AgentState TypedDict 字段完整 | 包含 messages, context, current_step, tool_outputs |
| PA-02 | ChatState TypedDict 字段验证 | messages 使用 `add_messages` reducer |
| PA-03 | Config 类属性完整 | 所有必需的 env 变量有默认值 |
| PA-04 | MCP Client 继承体系 | `SSEMCPClient → MCPClient`, `NacosSSEMCPClient → SSEMCPClient` |

### 核心功能 (12 用例)

**安全层 (4)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| PS-01 | _extract_token 优先 Cookie | Cookie 优先于 Header |
| PS-02 | _extract_token 兜底 Bearer | 无 Cookie 时用 Header |
| PS-03 | 有效 JWT → 返回 CurrentUser | sub/name/picture 正确解析 |
| PS-04 | 过期 JWT → 401 | ExpiredSignatureError |

**LangGraph 图结构 (4)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| PG-01 | should_continue 有 tool_calls → "tools" | 路由到工具节点 |
| PG-02 | should_continue 无 tool_calls → END | 路由到终止 |
| PG-03 | workflow 图结构完整 | 包含 agent + tools 节点 |
| PG-04 | save_chat_history 正常保存 | user + ai 消息持久化 |

**MCP Client (4)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| PC-01 | SSEMCPClient.list_tools 构造正确 JSON-RPC | method = "tools/list" |
| PC-02 | SSEMCPClient.call_tool 构造正确 JSON-RPC | params 含 name + arguments |
| PC-03 | NacosSSEMCPClient._resolve_url 解析服务地址 | base_url 正确拼接 |
| PC-04 | get_all_tools 聚合多个客户端工具 | 合并结果 + client_name 标签 |

---

## 🟠 工程四：timekeeper (Angular / TypeScript)

> **测试框架**: Jest (已从 Karma/Jasmine 迁移)

### 架构守护 (3 用例)

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| TA-01 | AuthService providedIn root | 全局单例 |
| TA-02 | apiUrlInterceptor 是纯函数 | `HttpInterceptorFn` 类型验证 |
| TA-03 | authGuard 是 CanActivateFn | 函数式守卫 |

### 核心功能 (10 用例)

**AuthService (4)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| TAS-01 | saveToken → localStorage 存储 + isLoggedIn = true | 状态同步 |
| TAS-02 | removeToken → localStorage 清除 + isLoggedIn = false | 状态同步 |
| TAS-03 | hasToken 排除 'undefined'/'null' 字符串 | 边界情况 |
| TAS-04 | extractTokenFromUrl 从 URL 提取并清理 | URL 清理 + 存储 |

**HTTP Interceptor (3)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| TI-01 | 非 http 开头的请求拼接 baseUrl | URL 重写 |
| TI-02 | Token 存在时注入 Authorization Header | `Bearer {token}` |
| TI-03 | 401 响应触发重定向（仅一次） | 锁机制防并发 |

**UserService (3)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| TU-01 | initialize 调用 extractTokenFromUrl | 启动时自动提取 |
| TU-02 | initialize 有 Token 时调用 getCurrentUser | /user/me API 调用 |
| TU-03 | initialize Token 过期时清除 Token | 容错降级 |

**LanguageService (3)**

| ID | 测试用例 | 验证点 |
|----|---------|--------|
| TL-01 | 初始加载默认语言 | `localStorage` 为空时默认为 `zh` |
| TL-02 | toggleLanguage 切换逻辑 | `zh` ↔ `en` 切换并更新 `localStorage` |
| TL-03 | setLanguage 手动更新 | 指定语言更新信号状态 |

---

## 覆盖度总结

| 工程 | 架构守护 | 核心功能 | 合计 |
|------|---------|---------|------|
| api-gateway | 5 | 15 | **20** |
| ai-langchain4j | 4 | 12 | **16** |
| python-agent | 4 | 12 | **16** |
| timekeeper | 3 | 13 | **16** |
| **总计** | **16** | **52** | **68** |
