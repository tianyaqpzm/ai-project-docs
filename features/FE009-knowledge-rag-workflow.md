# FE009: 知识库 RAG 流程分步配置增强

## 特性描述
该特性旨在将知识库的文档处理流程（Ingestion Pipeline）标准化为 5 个核心 RAG 阶段：数据准备 (Data Preparation)、构建索引 (Indexing)、检索优化 (Retrieval)、生成集成 (Generation) 及系统评估 (Evaluation)。通过前端分步导航界面，用户可以精细化控制每个阶段的策略，并将配置持久化至后端。

## 涉及子工程
- **ms-ng-view**: 提供 5 步流程配置 UI，收集并提交全量 RAG 参数。
- **ms-java-biz**: 核心业务层，负责 RAG 配置的持久化 (JSONB) 与任务分发。
- **ms-py-agent**: 执行层，接收参数并动态调用不同的 Embedding 模型与向量存储策略。

## 核心逻辑

### 1. 五步流程定义 (RAG Stages)
| 阶段 | 关键参数 | 业务含义 |
| :--- | :--- | :--- |
| **数据准备** | `chunkSize`, `chunkOverlap`, `separators` | 处理多格式文档并执行语义切块优化。 |
| **构建索引** | `embeddingModel`, `vectorStore` | 向量化并将数据存入指定数据库（如 PostgreSQL）。 |
| **检索优化** | `topK`, `scoreThreshold`, `enableHybridSearch`, `alphaWeight` | 提升召回率，支持稠密与稀疏结合的混合检索。 |
| **生成集成** | `generationModel`, `temperature`, `maxTokens`, `systemPrompt` | 控制 LLM 生成时的上下文输入与输出格式。 |
| **系统评估** | `enableEvaluation`, `evaluationMetrics` | 引入评估指标（如 Faithfulness）量化 RAG 质量。 |

### 2. 数据持久化 (Java DDD)
- **值对象**: `KnowledgeConfig` (Record) 统一管理上述所有字段。
- **存储方式**: 在 `ms_knowledge_document` 表中通过 `config_json` (JSONB) 字段存储，保证了配置的灵活性与强类型。

### 3. 全链路通信
- 前端通过 `KnowledgeUseCase` 发起请求，Payload 包含上述所有 RAG 字段。
- Java 后端解析 Payload 并映射为 `IngestDocumentRequest` DTO。
- 通过 HTTP 调用 Python Agent 的 `/documents/ingest` 接口，透传配置。

### 4. 前端组件化架构 (Frontend Modularity)
- **子组件拆分**: 为降低模板复杂度，将 5 个配置阶段封装为独立的 KL 系列子组件（位于 `knowledge-embedding/components/`）。
- **触发接口 (Trigger Interfaces)**: 每个子组件通过 Angular `input` 接收状态，并通过 `output` (EventEmitter) 实时向上同步配置变更，确保主组件持有最新的全局 RAG 策略。
- **Standalone 规范**: 采用 Angular Standalone 组件架构，实现逻辑的高度解耦与复用。

## UI 交互设计
- **水平进度条**: 展示 5 个阶段的层级关系。
- **状态感知圆圈**: 步骤圆圈根据当前进度动态切换；已完成的步骤显示 **对号 (Checkmark)** 图标，当前步骤高亮展示。
- **备注 Tooltip**: 在步骤标题旁提供 **帮助图标 (?)**，鼠标悬停可查看该 RAG 阶段的技术含义与配置建议。
- **动态终端日志**: 模拟系统执行过程，实时反馈每个 RAG 阶段的构建状态，支持流式日志展示。

## 影响评估
- **数据库**: `ms_knowledge_document` 表的 `config_json` 字段内容复杂度增加，但 Schema 保持向后兼容。
- **Python 端**: 需支持动态加载 `LLMFactory` 中的 Embedding 模型。
