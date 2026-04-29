# Architecture Decision Record (ADR) - 003: 前端 ms-ng-view 全面 向 4 层整洁架构演进

## 1. 标题 (Title)
ms-ng-view 前端工程全模块向 4 层整洁架构演进及 API URL 集中化管理

## 2. 状态 (Status)
已接受 (Accepted) - 2026-04-28

## 3. 背景 (Context)
在 `ms-ng-view` 的开发初期，`Chat`、`Knowledge` 和 `Event` 模块承载了过多的职责。组件内部直接调用 `HttpClient` 处理业务逻辑，且存在大量的硬编码 API URL 字符串。这种模式导致：
* **难以维护与测试**：业务逻辑与视图耦合，网络处理与状态管理混杂。
* **URL 碎片化**：API 路径散落在各个 Adapter/Service 中，难以统一管理和维护。
* **违反整洁架构原则**：领域知识被局限在 UI 框架的生命周期内。

## 4. 决策 (Decision)
我们决定对 `ms-ng-view` 进行全量架构重构，将所有业务模块拆分为严格的 4 层架构，并引入 `URLConfig`：

1. **全面 4 层架构 (4-Layer Architecture)**：
   - **Domain Layer**：定义业务模型与 Repository 接口，确保业务逻辑纯度。
   - **Adapter Layer**：封装 `HttpClient` 与硬件访问。引入 `URLConfig` 消除硬编码。
   - **Use Case Layer**：使用 Angular Signals 管理页面状态，编排异步流程。
   - **UI Layer**：作为“傻瓜组件 (Dumb Component)”，仅负责渲染和交互。

2. **API URL 集中化 (URL Centralization)**：
   - 新增 `src/app/core/constants/url.config.ts`。
   - 所有 Adapter 必须通过 `URLConfig` 引用接口路径，严禁在 Adapter 内部出现硬编码 URL。

3. **清理遗留代码与技术债**：
    - 彻底废弃并删除 `src/app/core/services/` 下的胖服务（如 `KnowledgeService`, `EventService`），由 `UseCase` 取而代之。
    - 删除 `src/components/` 及其子目录（`mobile-list`, `mobile-detail`），所有功能组件现统一归口至 `src/app/features/`。

## 5. 影响 (Consequences)

### 正面影响 (Positive)
* **业务边界清晰**：每一层具有单一职责。
* **URL 一致性**：集中管理 URL 使得后端接口变动时的修改工作量极小化。
* **隔离框架与外部依赖**：Adapter 层的防腐作用得到增强。
* **极大地提高了可测试性**：支持对 Use Case 进行纯逻辑单元测试。

### 负面影响 (Negative)
* **初次重构成本高**：需要对存量代码进行大规模迁移。
* **样板代码增加**：每一项业务逻辑都需要穿透多层。

## 6. 验证 (Verification)
* **自动化测试**：2026-04-28，运行 `npm test`，全工程 12 个测试套件（34 个测试用例）全部通过 (PASS)。涵盖了 Use Case 的状态流转、Adapter 的 URL 构造及拦截器的 Auth 注入。
* **现代测试标准**：所有测试已成功从弃用的 `HttpClientTestingModule` 迁移至 `provideHttpClientTesting()`。
