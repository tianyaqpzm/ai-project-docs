# FE011-聊天页面 Markdown 渲染 (Chat Markdown Rendering)

## 状态
已完成 (Completed) - 2026-05-03

## 背景
在大模型对话场景中，AI 返回的内容通常包含 Markdown 格式的排版（如代码块、列表、表格、加粗等）。
原有的 `ms-ng-view` 聊天页面仅使用简单的文本插值 (`{{ msg.content }}`)，导致内容缺乏排版，阅读体验较差。

## 目标
*   在 `ms-ng-view` 中引入专业的 Markdown 渲染引擎。
*   支持流式数据的实时渲染。
*   提供基础的样式支持（包括深色模式适配）。
*   保持高性能渲染，避免页面卡顿。
*   **[新增] 支持引用标签 (Citation Tags) 的交互展示（鼠标悬停提示）。**
*   **[新增] 支持参考来源 (Reference Sources) 的折叠面板展示。**

## 技术选型
*   **渲染库**: `ngx-markdown` (版本适配 Angular 21)
*   **解析引擎**: `marked` (ngx-markdown 的核心依赖)
*   **交互逻辑**: 事件委托 (Event Delegation) + Angular Signals

## 实施方案

### 1. 基础依赖与配置
*   安装 `ngx-markdown` 和 `marked`。
*   在 `main.ts` 中通过 `provideMarkdown()` 进行全局注册，确保 standalone 架构下的可用性。

### 2. 组件集成
*   **组件**: `ChatComponent` (`src/app/features/chat/chat.component.ts`)
*   **变更**: 
    *   在 `imports` 中引入 `MarkdownComponent`。
    *   在 `chat.component.html` 中，将模型消息显示区域由插值表达式替换为 `<markdown [data]="msg.content | citation"></markdown>`。
    *   引入自定义 `CitationPipe` 用于预处理文本，将 `[^n^]` 转换为自定义 HTML 标签。
    *   使用事件委托在 Markdown 容器上监听 `mouseover` 事件，触发引用悬浮窗。

### 3. 引用与来源支持
*   **Citation Tags**: 通过 `CitationPipe` 将 `[^1^]` 格式转换为 `<span class="citation-tag">`，并配合自定义浮层展示来源。
*   **Reference Sources**: 在 AI 回答下方增加 `<details>` 折叠面板，循环渲染 `msg.sources` 数组中的文档标题与摘要。

### 4. 样式增强
在 `chat.component.css` 中为 `.markdown-body` 增加了深度选择器样式，支持：
*   **标题 (H1-H3)**: 合适的字号与间距。
*   **列表 (UL/OL)**: 标准的缩进与符号。
*   **代码 (Code/Pre)**: 具有背景色区分的代码块，适配深色模式。
*   **引用 (Blockquote)**: 侧边边框装饰。
*   **表格 (Table)**: 完整的边框与斑马纹支持。
*   **引用标签**: 蓝色胶囊状设计，悬停时产生阴影与位移。
*   **参考面板**: 采用 Tailwind `group` 类实现动画箭头及 Hover 效果。

## 涉及范围

| 子工程 | 变更点 |
|--------|----------|
| **ms-ng-view** | 引入新依赖，创建 `CitationPipe`，更新 `ChatUseCase` 状态模型，增强 `ChatComponent` UI。 |
| **ms-py-agent** | 更新 `ChatState` 增加 `sources` 字段，RAG 检索后将文档信息透传至前端 SSE。 |

## 验证结论
*   **渲染效果**: 复杂 Markdown 格式均能正确展示。
*   **性能**: 流式输出过程中，页面渲染平稳，无明显掉帧。
*   **样式适配**: 在浅色/深色模式下，Markdown 元素均清晰可见。
*   **交互性**: 引用标签可悬停显示来源，底部面板可手动折叠/展开。
*   **一致性**: 后端 RAG 结果与前端引用索引序号严格对齐。
