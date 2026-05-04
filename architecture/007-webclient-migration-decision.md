# ADR 007: 采用 WebClient 替代 RestTemplate 进行跨服务调用

## 状态
已通过 (Accepted)

## 上下文 (Context)
在微服务架构中，`ms-java-biz` 需要频繁调用 `ms-py-agent` 及其他外部服务。传统的 `RestTemplate` 基于 Servlet 栈，采用“每个请求一个线程”的阻塞 I/O 模型。随着 AI 业务场景的增加，特别是流式响应 (SSE) 和高并发异步处理的需求日益凸显，`RestTemplate` 的局限性开始显现。

## 决策 (Decision)
我们决定在 `ms-java-biz` 中引入 `spring-boot-starter-webflux`，并使用 `WebClient` 替代 `RestTemplate` 作为全系统的标准化 HTTP 客户端。

### 关键配置
- **WebClientConfig**: 提供 `@LoadBalanced` 的 `WebClient.Builder`。
- **JWT 自动透传**: 通过 `ExchangeFilterFunction` 实现全链路身份凭证的无感传递。
- **契约驱动 (Contract-First)**: 采用 `openapi-generator` 配合 `WebClient` 库，自动生成类型安全的客户端 DTO 与 API 接口。

### 3. 跨服务 SDK 标准化 (ms-java-yaml2code-sdk)
为了解决多服务合约维护困难及 `pom.xml` 膨胀问题，我们决定：
- 创建独立的 **`ms-java-yaml2code-sdk`** 工程。
- 集中存放所有跨服务 OpenAPI YAML 契约。
- 统一生成并分发可重用的 SDK JAR 包，业务工程仅需依赖该 SDK 即可获得最新的 API 定义。

## 理由 (Consequences)

### 优点 (Pros)
1. **现代化与未来兼容性**: `RestTemplate` 目前已进入维护模式，Spring 官方推荐使用 `WebClient`。
2. **支持响应式与流式数据**: `WebClient` 原生支持 `Flux` 和 `Mono`，能够轻松处理 `ms-py-agent` 返回的 SSE (Server-Sent Events) 流，这在 AI 对话场景下是核心需求。
3. **更好的性能**: 基于 Reactor Netty 的非阻塞 I/O 在处理高延迟或高并发请求时具有更高的吞吐量和更低的资源占用。
4. **功能丰富**: 提供更优雅的错误处理机制 (`onStatus`)、超时配置以及更灵活的过滤器链。
5. **代码质量与一致性**: 通过中心化 SDK 保证了多端（Java/Python/Angular）契约的一致性，减少了因手动维护 DTO 导致的类型错误。

### 缺点 (Cons)
1. **学习曲线**: 对于习惯于同步阻塞编程的开发者，理解响应式编程（Reactive Programming）需要一定的学习成本。
2. **调试难度**: 响应式堆栈的调试比传统 Servlet 堆栈更复杂（堆栈信息不直观）。

## 后续行动
- 将现有的 `PythonAgentClientImpl` 迁移至 `WebClient`，并接入标准化异常解析逻辑。
- 建立并完善 `ms-java-yaml2code-sdk` 的自动化构建与发布流程。
- 后续新增的跨服务调用必须优先使用 `WebClient` 且通过 SDK 进行合约定义。
- 逐步替换项目中存量的 `RestTemplate` 调用点。
