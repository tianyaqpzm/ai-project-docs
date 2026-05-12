# FE012-Prompt 管理 (Prompt Management) 特性

## 1. 业务逻辑与目标
Prompt 管理特性的核心目标是为系统提供动态的提示词模板管理能力。通过将 Prompt 从代码中抽离到数据库中，业务人员和开发人员可以在不修改代码和重启服务的情况下，动态调整大模型（LLM）的提示词策略。

该特性支持以下核心能力：
- **模板管理 (Template Management)**：定义不同业务场景的 Prompt 模板（如 `System`, `User`, `Tool`）。
- **版本控制 (Version Control)**：每个模板支持多个版本历史记录。
- **状态激活 (Activation)**：允许在多个版本中灵活切换当前生产环境生效的版本。
- **动态变量提取**：支持在提示词中定义并校验动态变量（Variables）。

## 2. 涉及的子工程
该特性是一个全栈功能，跨越以下子工程：

- **ms-java-biz (后端业务服务)**
  - 负责 Prompt 模板与版本的持久化（PostgreSQL JSONB）。
  - 基于领域驱动设计（DDD），定义 `PromptTemplate` 作为聚合根。
  - 提供对外的 RESTful 管理 API 以及供 Agent 调用的运行时接口。

- **ms-ng-view (前端管理控制台)**
  - 基于 Angular 21 和 4 层整洁架构（Domain, Use Case, Adapter, UI）实现。
  - 提供响应式的管理界面，包含侧边栏模板列表、版本历史追踪和 Markdown 风格的编辑器。

- **ms-py-agent (智能编排服务)** *(后续集成)*
  - 运行时通过调用 `ms-java-biz` 获取已激活的 Prompt 内容，并组装上下文发送给 LLM。

## 3. 接口变动
新增了以下针对 Prompt 管理的 API，详细定义请参考 `docs/api-contracts/MsJavaBiz-PromptManagement-v1.yaml`。

### 3.1 Management API (后台管理)
- `GET /rest/biz/v1/prompts/templates`：获取所有 Prompt 模板列表。
- `GET /rest/biz/v1/prompts/templates/{templateId}/versions`：获取指定模板的版本历史。
- `POST /rest/biz/v1/prompts/templates/{templateId}/versions`：为指定模板创建一个新版本。
- `PATCH /rest/biz/v1/prompts/versions/{versionId}/activate`：激活指定版本，并自动将该模板的其他版本设为未激活。

### 3.2 Runtime API (Agent 运行时)
- `GET /rest/biz/v1/prompts/runtime/{slug}`：根据模板 slug 获取当前已激活版本的提示词内容及配置。

## 4. 架构与设计决策
1. **聚合与一致性**：激活版本（Activate Version）操作在领域层被设计为对 `PromptTemplate` 的状态更新，通过事务保证原子性。
2. **全局错误码规范**：该特性率先引入并遵循了全新的 `DEP_01xx` 全局业务错误码规范（见 `ErrorCodeRegistry.md`），确保了微服务间的错误传递清晰且不冲突。
3. **前端 4 层架构**：在 `ms-ng-view` 中严格执行了 Domain -> Use Case -> Adapter -> UI 的分层模型，业务逻辑在 Use Case 层闭环，UI 组件保持傻瓜化。

## 5. UI/UX 优化与交互增强
1. **沉浸式编辑体验**：
   - 移除了输入框选中时的蓝色高亮边框，减少视觉干扰，让用户专注于提示词内容。
   - 实现了**版本号智能建议**：选中旧版本克隆时，系统自动预测并填充下一个逻辑版本号（如 `v1.0.1` -> `v1.0.2`）。
2. **直观的状态反馈**：
   - **选中态增强**：侧边栏模板列表和右侧版本历史列表均增加了蓝色左侧边框及加粗高亮，清晰指示当前操作上下文。
   - **即时通知**：集成 `MatSnackBar`，在版本保存、激活等关键操作完成后，在页面顶部提供即时结果反馈。
3. **流程自动化**：修复了“点击模板后需再次点击版本”的逻辑，改为选中模板后自动加载并渲染其“已激活”版本的内容。

## 6. 技术优化与稳定性保证
1. **网关路径修正**：解决了 `/rest/biz/v1/**` 前缀的 404 转发问题，统一了业务 API 的网关路由规则。
2. **连接池稳定性**：针对 PostgreSQL 数据库连接频繁关闭的问题，将 HikariCP 的 `max-lifetime` 调优至 10 分钟，确保长连接的稳定性。
3. **工程质量**：前端关键组件实现了 100% 的 **TSDoc 注释覆盖**，符合项目长期维护的文档化标准。
