# 特性定义：AI 美食助手 (AI Food Assistant)

## 业务背景
用户在日常聊天中经常面临“今天吃什么”的决策难题。为了提升系统的人性化体验并增加实用场景，我们将原有的“创作音乐”建议块重构为“今天吃什么” AI 美食助手。该功能旨在为用户提供从灵感发现到烹饪指导的一站式服务。

## 技术方案
### 1. 前端组件化架构
- **Portal 模式**: 采用 `CookPortalComponent` 独立子模块，避免 `ChatComponent` 逻辑膨胀。
- **动态交互**: 实现美食分类网格（12类）与“为您推荐”菜谱卡片。点击任意项可自动构造查询词并触发消息发送。
- **状态感知**: 集成 `isCookMode` 信号，动态切换输入框占位符（Placeholder）并显示“美食模式”状态徽章。

### 2. RAG 与 知识库集成
- **关联知识库**: 自动关联 ID 为 `topic_recipe_001` 的“菜谱”知识库。
- **大厨人格 (Chef Persona)**: 在 `ms-py-agent` 的 Graph 流程中，根据 `topic_id` 动态注入系统级提示词，使 AI 以“拥有20年经验的中餐大厨”身份提供服务。

### 3. 数据存储与迁移
- **数据库**: 在 `ms-java-biz` 中新增 `recipes`（菜谱业务数据）与 `user_favorites`（用户收藏）表。
- **自动初始化**: 通过 Flyway `V1.3` 脚本自动预置“菜谱”知识库主题及初始 Mock 菜谱。

## 涉及范围
- **ms-ng-view**:
    - `CookPortalComponent`: 核心展示与分类选择逻辑。
    - `ChatComponent`: 模式切换与 Suggesion 按钮重构。
- **ms-java-biz**:
    - `V1.3__recipe_and_food_schema.sql`: 数据库 Schema 定义。
- **ms-py-agent**:
    - `app/services/chat_graph.py`: 注入 Chef 专家人格。
    - `app/agent/state.py`: 增加 `topic_id` 状态支持。

## 接口定义
- **API 合约**: [/docs/api-contracts/MsJavaBiz-PregramerRecipes-v1.yaml](file:///Users/pei/projects/docs/api-contracts/MsJavaBiz-PregramerRecipes-v1.yaml)
- **对话接口**: `POST /api/agent/chat` (通过透传 `topic_id` 实现美食专用 RAG 与人格切换)。

## UI 设计
- **风格**: 现代简约，配合 AI 生成的高质量拼贴背景图。
- **交互**: 全自动发送逻辑，点击即回复，支持流式对话渲染。
