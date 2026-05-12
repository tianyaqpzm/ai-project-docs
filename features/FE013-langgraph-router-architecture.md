# FE013 LangGraph 路由架构重构与多 Agent 扩展

> 创建时间：2026-05-10  
> 工程范围：ms-py-agent  
> 状态：✅ 已完成（A2A 远端 Agent HTTP 调用待实现）

---

## 一、背景与目标

### 问题
原有 `ms-py-agent` 采用单体 LangGraph（`graph.py`）架构：
- 所有意图处理逻辑混合在同一个图中，扩展成本高
- MCP 工具列表硬编码为 `tools=[]`，无法动态加载
- 无法对接外部 Agent 服务（Java Agent / 第三方 AI 服务）

### 目标
1. 将单体图拆分为**路由图 + 子图**架构，实现意图驱动的动态分发
2. 引入官方 `mcp` SDK，实现工具 Schema 的动态加载（替代硬编码）
3. 预留两种多 Agent 对接模式的完整扩展点

---

## 二、业务逻辑

### 路由流程

```
用户输入
    │
    ▼ router_node（关键词优先 + LLM fallback）
    │
    ├─ intent = "rag"          → rag_subgraph（本地，已实现）
    ├─ intent = "coding"       → coding_subgraph（本地，Dummy 占位）
    ├─ intent = "general"      → general_subgraph（本地，Dummy 占位）
    └─ intent = "remote_agent" → remote_agent_subgraph（A2A，结构完整，HTTP 待实现）
```

### 意图分类策略（关键词优先级）

| 优先级 | 规则 | Intent |
|--------|------|--------|
| 1 | 委托 / 转交 / 外部agent | remote_agent |
| 2 | 代码 / 编程 / bug / debug | coding |
| 3 | 查 / 搜 / 文档 / 知识库 | rag |
| 4 | LLM 分类（Prompt 从 ms-java-biz 动态拉取） | rag / coding / general / remote_agent |
| 5 | 默认兜底 | general |

### Prompt 动态化
- LLM Fallback 分类的 System Prompt 从 `ms-java-biz` `/rest/biz/v1/prompts/router_intent_classifier` 接口动态获取
- ms-java-biz 不可达时降级到内置 `_FALLBACK_ROUTER_PROMPT`

---

## 三、涉及子工程与接口变动

| 子工程 | 变动类型 | 说明 |
|--------|----------|------|
| **ms-py-agent** | ✅ 核心重构 | 新增路由架构，MCPToolRegistry，A2A 骨架 |
| **ms-java-biz** | 🔵 需预置 Prompt | 需在 Prompt 管理中新增 `slug=router_intent_classifier` |
| **ms-java-gateway** | 🟢 无影响 | 路由规则不变 |
| **ms-ng-view** | 🟢 无影响 | 前端协议不变 |

---

## 四、新增文件清单

```
ms-py-agent/
├── app/agent/
│   ├── state.py                        # GlobalState + 子图 State + RemoteAgentSubState
│   ├── router/
│   │   ├── node.py                     # 意图分类节点（关键词 + LLM + Prompt 动态化）
│   │   └── graph.py                    # Global Router Graph（4 个子图节点）
│   └── subgraphs/
│       ├── rag_graph.py                # RAG 子图（完整，MCPToolRegistry 集成）
│       ├── coding_graph.py             # Coding 子图（Dummy 占位）
│       ├── general_graph.py            # General 子图（Dummy 占位）
│       └── remote_agent_graph.py       # A2A 子图（结构完整，HTTP 调用 pass）
├── app/core/
│   ├── mcp_registry.py                 # MCPToolRegistry 单例（官方 mcp SDK）
│   └── mcp_initialization.py           # 生命周期：filesystem + java-biz 注册
└── tests/
    └── test_router_graph.py            # 21 个验收测试（全通过）
```

---

## 五、两种多 Agent 对接模式

### 模式 A：本地子图（已实现）

进程内执行，子图以 `StateGraph` 形式嵌入路由图。  
扩展方式：在 `app/agent/subgraphs/` 下新建 `xxx_graph.py`，在 `router/graph.py` 注册。

### 模式 B：A2A 远端 Agent（骨架完整，HTTP 待实现）

通过 HTTP 委托给外部 Agent（遵循 [A2A 协议](https://google.github.io/A2A/)）。

**实现剩余工作**（仅需解注释 + 实现 3 个函数）：

```python
# app/agent/subgraphs/remote_agent_graph.py

async def _send_a2a_task(...)    # ← 取消注释，实现 httpx POST /a2a/tasks/send
async def _poll_a2a_result(...)  # ← 取消注释，实现 SSE 流式轮询
_INTENT_TO_AGENT = {             # ← 添加 intent → Nacos 服务名映射
    "coding": "ms-coding-agent",
}
```

---

## 六、GlobalState 扩展字段（A2A 协作）

| 字段 | 类型 | 写入方 | 用途 |
|------|------|--------|------|
| `handled_by` | `Optional[str]` | 各子图 | 标识哪个 agent 处理（`local:rag` / `remote:服务名@url`） |
| `a2a_task_id` | `Optional[str]` | remote_agent_node | 远端任务 ID，用于查询/取消 |
| `artifacts` | `List[Dict]` | remote_agent_node | 远端 Agent 返回的结构化产出物 |
| `handoff_context` | `Optional[str]` | 本地子图 | 多跳推理中传递给远端 Agent 的中间上下文 |

---

## 七、MCPToolRegistry 核心规范

- 唯一入口：`app/core/mcp_registry.py` 的 `mcp_tool_registry` 单例
- 支持 Stdio（filesystem 等本地 MCP Server）和 SSE（ms-java-biz 等远端服务）
- 工具列表在 `lifespan` startup 时动态加载，子图通过 `get_langchain_tools()` 获取
- 旧 `mcp_clients` 注册表保留以兼容 `chat_graph.py`

---

## 八、验收标准

- [x] 21/21 测试通过（`tests/test_router_graph.py`）
- [x] "帮我查文档" → intent=rag，100% 命中
- [x] "帮我写Python" → intent=coding，100% 命中
- [x] "委托外部agent" → intent=remote_agent，100% 命中
- [x] remote_agent_node 在 Nacos 不可达时返回错误提示而非抛异常
- [x] GlobalState 包含 4 个 A2A 扩展字段
- [ ] ms-java-biz 预置 `router_intent_classifier` Prompt（待运营配置）
- [ ] A2A HTTP 调用实现（待远端 Agent 服务就绪后实现）
