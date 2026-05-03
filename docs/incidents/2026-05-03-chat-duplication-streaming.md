# RCA: 聊天消息重复与非流式显示问题复盘

**日期**: 2026-05-03
**问题描述**: 
1. 聊天界面中出现大量重复的人类消息和 AI 回复。
2. 聊天回复无法实现“打字机”式的流式显示，而是在数秒后一次性弹出。

---

## 1. 聊天消息重复问题

### 根因分析 (Root Cause)
1. **前端状态解锁过早**: `isThinking` 状态在收到 SSE 流的第一个字符时就被重置为 `false`。这导致发送按钮在 AI 仍在生成期间就变回可用状态。用户在等待过程中若再次点击，会触发重复的 API 请求。
2. **编辑/重试导致的后验失同步**: 
   - 当用户点击“编辑”或“重试”时，前端仅在本地 UI 删除了该条消息，但后端 **LangGraph 的 Checkpointer (数据库)** 仍保留着完整历史。
   - 由于使用的是同一个 `session_id`，LangGraph 会将“新”发送的消息直接追加在已有的（但在 UI 上被隐藏的）消息之后。
   - 刷新页面后，前端通过 `getHistory` 接口拉取到真实的后端状态，导致重复消息暴露。

### 解决方案 (Resolution)
1. **修复前端状态逻辑**: 将 `isThinking` 的重置移动到 SSE 流的 `complete` 和 `error` 回调中，确保生成彻底结束前按钮不可用。
2. **（待办）后端状态截断**: 未来需增加一个同步机制，在用户点击重试时，通过 API 显式告知后端截断到特定的消息索引。

---

## 2. 非流式显示问题

### 根因分析 (Root Cause)
1. **LLM 配置缺失**: `LLMFactory` 在初始化 `ChatOpenAI` 和 `ChatGoogleGenerativeAI` 时未显式开启 `streaming=True`。
2. **后端监听事件错误**: API 路由层监听的是 `on_chain_end` 事件（节点结束事件），而非 `on_chat_model_stream`（Token 级别事件）。
3. **前端 SSE 解析器缺陷**: 
   - 之前的前端逻辑在处理 Angular `HttpClient` 的 `partialText`（累积响应体）时，每次都会从头解析所有行，导致严重的冗余计算和 UI 闪烁。
   - 缺乏行缓冲区 (Line Buffer)，无法处理被网络 TCP 包截断的非完整 JSON 行。

### 解决方案 (Resolution)
1. **开启 LLM 流式模式**: 在 `LLMFactory` 构造函数中加入 `streaming=True`。
2. **升级 API 监听逻辑**: 在 `astream_events` 循环中捕获 `on_chat_model_stream` 事件并实时 `yield` 增量 Token。
3. **重构前端适配器**: 
   - 引入 `lastProcessedIndex` 记录已处理偏移。
   - 实现 `lineBuffer` 机制，确保只有收到完整的换行符后才进行 JSON 解析，保证了流式数据的健壮性。
   - 修改 `ChatUseCase` 逻辑，将更新模式从“覆盖”改为“追加”。

---

## 经验总结 (Lessons Learned)
1. **SSE 状态机对齐**: 流式场景下的 `Loading/Thinking` 状态必须严格绑定到流的 `FIN` 标志（complete/error），不能依赖于数据的到达。
2. **状态化后端同步**: 在使用 LangGraph 等带状态（Checkpointer）的后端时，前端的“逻辑删除/撤销”操作必须有对应的后端状态清理机制，不能简单地在 UI 上做掩耳盗铃。
3. **健壮的 SSE 解析**: 前端解析 SSE 时应始终考虑数据被截断的可能性，缓冲区机制是流式开发的标配。
