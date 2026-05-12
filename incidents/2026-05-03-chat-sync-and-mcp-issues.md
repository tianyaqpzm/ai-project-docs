# 故障复盘：Chat 会话同步与 MCP 跨服务通信失效 (2026-05-03)

## 问题描述
用户反馈 `/chat` 页面产生新对话时，侧边栏未能实时同步新增会话。在排查过程中，进一步发现了 Java 后端 SQL 异常、MCP 工具调用 403 权限错误以及系统空指针异常。

## 根因分析 (RCA)

### 1. 数据同步失效 (Chat Session Sync)
*   **现象**：Python 端发送消息后，数据库 `ms_chat_session` 表未更新。
*   **根因**：
    *   **命名冲突**：`ms-py-agent` 的 ORM 模型指向了错误的表名 `chat_messages`，而 Java 端的同步触发器监听的是 `ms_chat_message`。
    *   **竞态条件**：Python 端在发出流式结束信号 `[DONE]` 后才异步保存历史，导致前端刷新请求早于数据库写入。
*   **解决**：对齐 Python 模型表名，并将保存逻辑提前至 `[DONE]` 信号之前。

### 2. SQL 类型映射错误 (BadSqlGrammarException)
*   **现象**：Java 后端提示无法将 `TIMESTAMPTZ` 转换为 `LocalDateTime`。
*   **根因**：PostgreSQL 的带时区时间戳字段必须映射到 Java 的 `OffsetDateTime` 而非 `LocalDateTime`。
*   **解决**：统一修改 Prompt 模块相关实体的字段类型。

### 3. MCP 通信权限与兼容性
*   **403 Forbidden**：`ms-py-agent` 调用 Java MCP 接口时未透传 `Authorization` Token，且 Java 端安全白名单未覆盖 `/mcp/sse` 端点。
*   **NullPointerException**：Java `McpController` 在处理不带 `sessionId` 的无状态请求时，因 `ConcurrentHashMap` 不支持 null 键而崩溃。
*   **LangChain 报错**：`@tool` 装饰器不支持动态设置 `name` 参数。
*   **解决**：
    *   Java 端放开 `/mcp/**` 白名单，并实现 `McpController` 的双模响应（SSE + Stateless HTTP）。
    *   Python 端实现 Token Relay（Token 透传），并改用 `StructuredTool.from_function` 创建工具。

## 预防措施与经验沉淀
1.  **多工程模型同步**：跨语言开发时，数据库表名、字段名及触发器逻辑必须以 `ms-java-biz` 定义的规范为准。
2.  **Token 透传规范**：所有 Agent 发起的下行业务请求必须显式携带并透传用户的 Auth Token。
3.  **类型兼容性检查**：在涉及 `TIMESTAMPTZ` 字段时，Java 端强制要求使用 `OffsetDateTime`。
4.  **动态刷新机制**：数据库连接等敏感配置在 Nacos 变更后，程序应具备自动重置单例连接池的能力。
