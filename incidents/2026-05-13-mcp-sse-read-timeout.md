# Incident Report: MCP SSE Connection Instability & Agent Loop Protection

**Date:** 2026-05-13
**Updated:** 2026-05-14
**Status:** Resolved
**Project:** ms-py-agent, ms-java-biz

## 1. 问题描述 (Issue Description)
用户在调用 MCP 工具时遭遇：
1. **连接假死**: Agent 陷入长时间无响应，日志显示 `httpx.ReadTimeout`。
2. **连接猝死**: 日志抛出 `peer closed connection (incomplete chunked read)`，导致工具调用失败。
3. **空回复**: 当子图触发循环熔断时，由于最后一条消息是空内容的 `tool_calls`，导致前端没有答复。

## 2. 根因分析 (Root Cause Analysis)

### 2.1 MCP SSE 通信超时
- **原因:** MCP SDK 默认 `read_timeout` 仅 5s（httpx 默认值）。
- **细节:** 复杂工具（如大目录扫描或复杂 SQL）执行耗时常超过 5s，导致流读取线程断开。

### 2.2 链路闲置断开 (Idle Timeout)
- **原因:** SSE 长连接期间如果长时间无数据交换，中间网关（Gateway/Nginx）会强行切断 TCP 连接。
- **细节:** 缺乏应用层心跳（Keep-alive），导致 Python 端遭遇 `incomplete chunked read` 异常。

### 2.3 熔断逻辑不完整
- **原因:** 之前的 `MAX_TOOL_ITERATIONS` 仅简单 `END` 流程，未考虑 LLM 处于“待回复”状态。

## 3. 修复措施 (Remedial Actions)

### 3.1 增加多层通信超时保护
- **SDK 层**: 在 `sse_client` 连接时设置 `timeout=30s` (连接) 和 `sse_read_timeout=300s` (流读)。
- **调用层**: 在 `call_tool` 中使用 `asyncio.wait_for` (35s) 进行二次包裹，并配置 SDK 的 `read_timeout_seconds=30` 参数，确保卡死的工具调用能被准确截断。

### 3.2 实现双端链路保活 (Heartbeat)
- **Java 端 (Server)**: 
    - **心跳机制**: 在 `McpController` 中启用 `@Scheduled` 任务，每 15s 发送一次 **SSE 注释 (Comment)** 心跳。改用注释模式避免了客户端产生 "Unknown event" 警告。
    - **超时配置**: 在 `application.yaml` 中将 `spring.mvc.async.request-timeout` 设置为 **1 小时**，彻底解决 Spring 默认 30s 异步超时导致的连接猝死。
- **Python 端 (Client)**: 引入 **自动重连机制**。捕获连接关闭异常后标记失效，在下次调用前自动执行 `_ensure_connected` 重新握手。

### 3.3 增强异常处理鲁棒性
- **Java 端**: 优化 `GlobalExceptionHandler`，增加 **SSE 感知** 逻辑。当请求为 `text/event-stream` 时，异常发生后返回纯文本而非 JSON，解决了 `HttpMessageNotWritableException` (No converter for ErrorResponse) 的冲突问题。

### 3.4 引入“强制总结”节点 (Final Answer Node)
- **流程优化**: 将 `_should_continue` 路由从 `END` 改为指向 `final_answer` 节点。
- **策略**: 该节点不绑定工具，注入系统指令强制 LLM 根据当前已有信息给出最终答复，彻底解决“空回复”问题。

## 4. 经验教训 (Lessons Learned)
1. **默认值陷阱**: 必须显式审视第三方网络库的默认超时设置，长连接场景严禁使用通用 HTTP 短超时。
2. **状态机安全阀**: 所有的自动循环（Loop）结构必须具备硬性的“安全阀”（迭代上限），防止逻辑雪崩。
3. **调用可见性**: 工具调用必须记录耗时日志，以便在异常时快速定位是“网络超时”还是“后端卡死”。
4. **长连接必须有心跳**: 跨服务/跨网关的长连接若无心跳保活，极易被中间层静默断开。

## 5. 关联变更 (Related Changes)
- `app/core/mcp_registry.py`: 超时控制 + 自动重连 + 35s 保护。
- `app/agent/subgraphs/*.py`: `MAX_TOOL_ITERATIONS` + `final_answer` 节点。
- `app/services/prompt_service.py`: 增加 Prompt 缓存以降低循环损耗。
- `ms-java-biz/.../McpController.java`: SSE 心跳与清理逻辑。
- `ms-java-biz/.../AiApplication.java`: 开启定时任务支持。
