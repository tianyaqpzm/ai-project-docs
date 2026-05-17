# 架构决策记录 (ADR) - 008: 统一 API 路由命名空间设计决策

* **状态**: 🟢 已接受 (Accepted)
* **日期**: 2026-05-17
* **作者**: Antigravity

---

## 1. 背景 (Context)

在早期的开发迭代中，系统的接口前缀存在多次分裂与多头演进，导致接口调用和网关配置越来越复杂：
1. **Python Agent 服务 (`ms-py-agent`)**:
   - 对话接口使用了前缀 `/rest/dark/v1/agent/chat`。
   - 知识库相关接口散落在前缀 `/rest/kb/v1/...`（如 `/rest/kb/v1/documents/ingest`）下。
2. **Java 业务服务 (`ms-java-biz`)**:
   - 一部分高内聚的基础和管理类接口使用了 `/rest/biz/v1/`（如 `prompts`、`mcp-plugins`）。
   - 另一部分常规业务接口却依旧采用 `/rest/dark/v1/` 的前缀（如 `tasks`、`time-limit-events`、`knowledge`、`history`、`user` 等）。

这种分裂导致 API 网关路由匹配中必须使用复合路径（逗号分隔）和顺序依赖判定，这极大增加了后续路由配置维护成本，且在多人协作时容易因命名空间混杂而编写出错误的冗余路由（例如 `/rest/agent/v1/agent/...`）。

---

## 2. 决策 (Decision)

为了在系统架构层彻底终结接口命名的混乱，我们决定推行**微服务群 API 路由命名空间大一统**的架构设计：

### 2.1 Python Agent 服务 统一底座
将 `ms-py-agent` 提供的所有 API 接口，无一例外地全部统一收归至前缀 **`/rest/agent/v1/`** 之下。
* 核心聊天接口变更为：`/rest/agent/v1/chat`。
* 知识库文档导入接口变更为：`/rest/agent/v1/documents/ingest`。
* 知识库增量构建接口变更为：`/rest/agent/v1/knowledge/build`（精炼移除了多余的 `/agent` 重复前缀）。
* 知识库检索与提问接口变更为：`/rest/agent/v1/retrieve` 与 `/rest/agent/v1/ask`。

### 2.2 Java 业务服务 统一底座
将 `ms-java-biz` 提供的所有对外业务及管理类接口，统一收归至前缀 **`/rest/biz/v1/`** 之下。
* 原先所有以 `/rest/dark/v1/` 开头的控制器端点，一律重构前缀为 `/rest/biz/v1/`：
  - `/rest/dark/v1/tasks` ➡️ `/rest/biz/v1/tasks`
  - `/rest/dark/v1/time-limit-events` ➡️ `/rest/biz/v1/time-limit-events`
  - `/rest/dark/v1/knowledge` ➡️ `/rest/biz/v1/knowledge`
  - `/rest/dark/v1/user` ➡️ `/rest/biz/v1/user`
  - `/rest/dark/v1/history` ➡️ `/rest/biz/v1/history`
  - `/rest/dark/v1/upload` ➡️ `/rest/biz/v1/upload`

### 2.3 API 网关谓词 极简化
大幅简化 `ms-java-gateway` 的配置，抛弃原先所有繁琐的路径定义和顺序判定：
* `agent-route` ➡️ 仅匹配 `/rest/agent/v1/**` ➡️ 转发给 `ms-py-agent`
* `biz-v1-route` ➡️ 仅匹配 `/rest/biz/v1/**` ➡️ 转发给 `ms-java-biz`

---

## 3. 影响与评估 (Consequences)

* **积极影响 (Positive)**:
  - **清晰明了**: 凡是 `/rest/agent/v1/` 开头的均指向 Python Agent 大脑，凡是 `/rest/biz/v1/` 开头的均指向 Java 业务端。
  - **低维护成本**: 网关（Gateway）配置大幅清爽、瘦身，消除了逗号分割、前导空格、以及任何特定路由条目的覆盖风险。
  - **高确定性**: 前端 `URLConfig` 全部模块化，极易扩展，杜绝硬编码接口路径。
* **消极影响 (Negative)**:
  - 需要在一次性的重构中全局适配前端、后端控制层和测试文件，不过该成本已通过 100% 绿灯的自动化测试成功抹平。
