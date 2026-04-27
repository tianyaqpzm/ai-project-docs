# 故障记录：GitHub Actions CI 环境兼容性与依赖冲突

**日期：** 2026-04-26
**状态：** 已解决
**根本原因：** CI 环境缺乏基础设施（MongoDB/OIDC）、依赖库版本冲突。

## 问题现象
1.  **ms-java-gateway**: 在 CI 中启动失败，报错无法连接 OIDC Provider 的 `.well-known` 端点。
2.  **ms-java-biz**: 
    - 数据库连接超时：无法连接 `localhost:27017` (MongoDB)。
    - 运行时错误：`NoSuchMethodError`，提示 `GenerationParamBuilder.searchOptions` 不存在。
    - **ApplicationContext 加载失败**：在 IDE 运行测试时报 `IllegalStateException: ApplicationContext failure threshold exceeded`。堆栈显示 `SeparateChatAssistantConfig` 因缺少 `MongoTemplate` 无法初始化。
3.  **ms-ng-view**:
    - **依赖冲突**: GitHub Actions 报错 `@angular/common@21.1.1` 与要求的 `21.2.10` 不匹配。
    - **Builder 丢失**: 执行 `ng test` 时报 `Cannot find builder "@angular-builders/jest:jest"`。
    - **CSS 语法错误**: 样式编译报错 `Expected identifier but found "!"`（Tailwind 4 特性冲突）。
    - **测试配置失效**: 更新后报 `Unknown file extension ".ts"` 和 `Base providers already called`。
    - **CI 参数报错**: GitHub Actions 执行报 `Error: Unknown argument: watchAll`。

## 根本原因分析
1.  **基础设施缺失**: GitHub Actions 默认不提供 MongoDB 实例。本地开发环境往往有运行中的容器或进程，导致这种依赖在本地不明显，但在 CI 中导致 Context 加载失败。
2.  **网络隔离与 OIDC 自动发现**: CI 环境无法随机访问外部域名。`ms-java-gateway` 在主 `application.yml` 中配置了 `issuer-uri`，由于 Spring Boot 的配置合并机制，`test` profile 会继承该值。只要 `issuer-uri` 存在，Spring Security OAuth2 客户端就会在启动时强制尝试连接该地址进行 OIDC Discovery，导致在隔离的 CI 环境中阻塞并超时。
3.  **BOM 冲突管理**: `Spring Cloud Alibaba` 的 BOM 将 `dashscope-sdk-java` 锁定在了较低版本（2.12.x），而 `LangChain4j` 社区版 starter 需要更高版本。
4.  **Builder API 变更**: 新版 `@angular-builders/jest` 将内部 builder 名称从 `:jest` 改为 `:run`，且同步修改了配置项（`jestConfig` 变为 `config`）。
5.  **ESM 与 TS 配置加载缺陷**: 在 `type: module` 的 Angular 项目中，Node.js 无法直接通过 builder 调用的 `require` 加载 `.ts` 格式的 Jest 配置文件。
6.  **自动化环境冲突**: 现代 Angular Jest Builder 默认开启了 `zoneless` 模式并自动处理 `TestBed` 初始化，与原有 `setup-jest.ts` 中的手动初始化代码产生了“双重初始化”冲突。
7.  **CSS 标准符合性**: Tailwind 4 的重要性修饰符 `!` 在进入 CSS 文件作为选择器时，未按 CSS 标准进行转义（需写作 `.\!w-8`），导致 Angular 编译器解析失败。
8.  **CLI 参数不一致**: 新版 Angular CLI 与 Jest Builder 强制要求使用 `--watch=false` 而非旧版的 `--watchAll=false`，导致原有的 CI 脚本参数不合法。
9.  **业务配置与基础设施耦合**: 业务 Bean 强依赖 `MongoTemplate` 导致基础设施缺失时全量崩溃。
## 解决方案
### ms-ng-view
- **对齐 Angular 版本**: 将所有 `@angular/*` 核心包更新至 `21.2.10`，周边包更新至 `21.2.8`，解决了 `@angular/platform-browser-dynamic` 引发的 Peer Dependency 冲突。
- **Builder 配置迁移**: 在 `angular.json` 中将测试 builder 更新为 `@angular-builders/jest:run`，并将 `jestConfig` 重命名为 `config`。
- **ESM 兼容性修正**: 将 `jest.config.ts` 转换为 `jest.config.js` (ESM)，解决 Node.js 在 `type: module` 下无法直接 load TS 配置的问题。
- **消除初始化冲突**: 显式关闭 builder 的 `zoneless` 模式（匹配项目 Zone.js 现状），并清空 `setup-jest.ts` 中的手动 `setupZoneTestEnv()` 调用，交由 builder 自动管理。
- **CSS 选择器转义**: 在 CSS 文件中对含有 `!` 的类选择器进行转义（如 `.\!w-8`），修复 Angular 编译器的解析错误。
- **CI 加固**: 在构建流中使用 `--legacy-peer-deps` 兼容社区插件的版本偏移；修正测试参数为 `--watch=false` 以适配新版 CLI。
### ms-java-gateway
- **Mock OIDC 端点**: 在 `application-test.yml` 中显式指定 `authorization-uri`、`token-uri` 等虚拟端点，以尝试阻止 Spring Security 拉取外部元数据。
- **配置隔离 (Profile Isolation)**: 进一步发现仅 Mock 端点不足以阻止继承自 `application.yml` 的 `issuer-uri` 触发 Discovery。因此采取最终方案：将 `issuer-uri` 移动到 `application-prod.yml`，确保在运行单元测试（`test` profile）时该属性彻底消失，从而跳过远程拉取逻辑。

### ms-java-biz
- **Mock MongoDB**: 使用 `Mockito` 在测试环境下创建 `MongoTemplate` 的 Mock Bean，并注入 MappingContext 以满足 Spring Data Repositories 的初始化需求。
- **锁定 SDK 版本**: 在 `pom.xml` 中以直连依赖方式声明 `dashscope-sdk-java:2.18.5`。直接依赖的优先级高于 BOM 的依赖管理。
- **环境隔离与测试策略**: 
    - 在 `application-test.yml` 中排除 MongoDB 相关的自动配置类。
    - **迁移 `@WebMvcTest`**: 将 Controller 层的测试（如 `KnowledgeControllerTest`）迁移至 `@WebMvcTest`。这不仅避免了不必要的 Context 加载，还彻底隔离了业务配置类的副作用。
    - **补全 Mock Bean**: 对于必须使用 `@SpringBootTest` 的类（如 `AiApplicationTests`），通过 `@MockBean` 注入 `MongoTemplate` 并模拟 `getConverter()` 方法。这满足了业务配置类（如 `SeparateChatAssistantConfig`）对 Bean 的强依赖，确保上下文能顺利刷新。

## 经验教训
- **基础设施 Mock 化**: 除非是集成测试，否则单元/切面测试应尽量 Mock 掉数据库和外部 API。
- **BOM 冲突预防**: 当同时使用多个厚重的 BOM（如 Spring Cloud Alibaba + LangChain4j）时，需警惕重叠 SDK 的版本。对于报错 `NoSuchMethodError` 的库，应第一时间检查 `mvn dependency:tree` 并手动锁定版本。
- **Angular 版本一致性**: 在 Angular 工程中，核心包（`core`, `common`, `compiler` 等）的版本必须严格一致。当升级个别包（如 `platform-browser-dynamic`）时，必须全量对齐，并注意检查 `Registry` 中 `material` 和 `cli` 的匹配版本。
- **CI 友好型配置**: `authorization-uri` 等关键地址在测试 profile 下应指向 mock 地址，避免启动阶段的耗时与网络波动。
- **优先使用切片测试 (Slice Testing)**: 除非需要测试跨层集成逻辑，否则应优先使用 `@WebMvcTest` 或 `@DataJpaTest`。这不仅能提供更好的隔离性，还能避免因个别业务配置类的依赖问题导致整个测试套件不可用。
