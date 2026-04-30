# 特性定义：Prompt 管理系统 (Prompt Management System)

## 1. 背景与目标
在复杂的 AI Agent 系统中，Prompt（提示词）是核心资产。硬编码在代码中的 Prompt 存在以下问题：
- **修改困难**：每次优化 Prompt 都需要重新构建和部署服务。
- **缺乏版本控制**：无法回滚到之前的 Prompt 版本。
- **调试不便**：无法在不影响生产环境的情况下测试新版 Prompt。

本特性旨在构建一套“模板+版本”双维度的 Prompt 管理系统，实现 Prompt 的动态下发、版本迭代与模型参数解耦。

## 2. 涉及子工程
- **ms-java-biz**: 负责 Prompt 的元数据持久化、版本管理以及对外暴露获取接口。
- **ms-py-agent**: 作为消费端，通过 Nacos 发现并调用 Java 后端接口，动态渲染并应用 Prompt。

## 3. 技术方案

### 3.1 数据库设计 (PostgreSQL)
采用双表结构，确保灵活性与历史可追溯性。

#### 模板主表 (`ms_prompt_template`)
定义 Prompt 的业务标识（Slug）与基本分类。
- `slug`: 业务唯一标识（如 `chef_persona_rag`），代码通过此字段引用。
- `type`: 类型（System / User / Tool）。

#### 版本明细表 (`ms_prompt_version`)
存储具体的 Prompt 内容、变量定义及模型配置。
- `content`: 包含 `{{variable}}` 占位符的原文。
- `model_config`: 存储模型参数（如 `temperature`, `model_name`）的 JSON 字段。
- `variables`: 存储变量定义的 JSON 字段。
- `is_active`: 标记当前生产环境生效版本。

### 3.2 动态渲染流程
1. **获取**: Python Agent 在执行 `agent_node` 时，根据业务场景（如 `topic_id`）确定 `prompt_slug`。
2. **请求**: 通过 `PromptService` 向 Java 后端发起请求获取生效版本。
3. **渲染**: 使用 `render_prompt` 方法将业务上下文（如 RAG 检索结果）填入模板占位符。
4. **执行**: 将最终 Prompt 注入 LLM 消息序列。

## 4. API 定义
接口路径：`GET /rest/biz/v1/prompts/{slug}`

**响应示例**:
```json
{
  "slug": "chef_persona_rag",
  "version": "v1.0.0",
  "content": "你是一位大厨... Context: {{context}}",
  "modelConfig": {
    "model": "gemini-1.5-pro",
    "temperature": 0.7
  },
  "variables": ["context"]
}
```

## 5. 影响评估
- **灵活性**: 显著提升，Prompt 调优无需发版。
- **性能**: 引入了跨服务调用，后续可考虑在 Python 端增加本地缓存或通过 Nacos 配置推送优化。
- **扩展性**: 支持不同模型使用不同的 Prompt 版本，为多模型适配打下基础。

## 6. 验证计划
- [x] 数据库表结构创建 (`V1.5`)。
- [x] Java 后端接口实现与连通性测试。
- [x] Python Agent 成功从后端拉取并渲染 Prompt。
- [x] 验证 Emoji 字符能够正常存储与显示。
