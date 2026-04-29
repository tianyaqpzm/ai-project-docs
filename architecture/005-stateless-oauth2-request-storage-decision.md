# ADR 005: 采用基于 Cookie 的无状态 OAuth2 授权请求存储方案

## 状态
已提议 (Proposed)

## 背景
在 `ms-java-gateway` 的重构过程中，系统从简单的单体逻辑转向了更规范的分布式网关架构。重构后，OAuth2 登录流程中频繁出现 `authorization_request_not_found` 错误。经排查，该错误源于 Spring Security 默认使用内存 Session (`WebSession`) 存储授权请求状态。在容器化部署、多实例运行或代理环境（Nginx/HTTPS）下，Session 极易丢失或无法跨实例共享，导致回调校验失败。

## 决策
我们将 `ms-java-gateway` 的 OAuth2 授权请求存储机制从“基于服务器 Session”切换为“基于客户端 Cookie”。具体实现为自定义 `CookieOAuth2RequestRepository`。

同时，为了保持架构一致性，原本存储在 Session 中的 `CUSTOM_REDIRECT_URI`（由 `RedirectSaveFilter` 捕获）也将迁移至 Cookie 存储。

## 方案对比

| 维度 | 原方案 (基于 WebSession) | 新方案 (基于 Http Cookie) |
| :--- | :--- | :--- |
| **状态管理** | **有状态**：依赖服务器内存存储。 | **无状态**：状态随请求在客户端传递。 |
| **扩展性** | **差**：多实例部署必须依赖 Redis 共享 Session 或开启会话粘滞。 | **优**：天生支持横向扩展，请求可被任意实例处理。 |
| **健壮性** | **弱**：易受 Session 过期、清理机制或 Cookie 属性（SameSite）变动影响。 | **强**：只要浏览器遵循标准发送 Cookie，流程即可闭环。 |
| **复杂性** | **低**：Spring Security 默认行为，无需代码。 | **中**：需要自定义 Repository 并处理序列化。 |
| **安全性** | **高**：数据留在服务端。 | **中**：数据在客户端，需设置 HttpOnly、Secure 并在登录后清理。 |

## 影响
1. **ms-java-gateway**: 
   - 新增 `CookieOAuth2RequestRepository` 工具类。
   - 修改 `SecurityConfig` 注入该存储库。
   - 修改 `RedirectSaveFilter` 改用 Cookie 记录重定向地址。
2. **运维/部署**: 
   - 不再强依赖 Redis 即可实现网关的高可用部署。
   - 解决了由于 Nginx 代理导致的 Session 丢失这一顽疾。

## 关键约束与经验总结 (Technical Constraints & Lessons Learned)

### 1. JVM 内存红线 (Metaspace OOM)
在实施此方案后，由于 OAuth2 Client 流程涉及大量的类加载（包括序列化工具、动态代理及复杂的安全过滤器链），`ms-java-gateway` 的 **Metaspace** 占用显著增加。
- **根因分析**: 初始配置的 `-XX:MaxMetaspaceSize=64m` 在触发 OAuth2 回调流程时会因空间不足导致 `java.lang.OutOfMemoryError: Metaspace`。
- **强制约束**: 凡是集成 OAuth2 安全能力的微服务，其 Metaspace 限制严禁低于 **128MB**。推荐配置：`-Xmx256m -Xms256m -XX:MaxMetaspaceSize=128m`。

### 2. 测试用例适配 (Stateless Testing)
由“有状态”转为“无状态”后，原有的单元测试（如 `RedirectSaveFilterTest`）中基于 `WebSession` 的断言将失效。
- **规范**: 必须同步更新测试代码，将 Session 属性校验改为 **Cookie** 校验，以确保测试逻辑与生产架构一致。

## 结论
虽然 Cookie 存储增加了少量的代码复杂性，但它为微服务网关提供了必要的无状态特性，彻底解决了由于 Session 不一致导致的登录链路中断问题，符合“零业务逻辑、高可用基础设施”的网关设计原则。同时，必须配合合理的 JVM 内存参数设置，以保障基础设施层的稳定性。
