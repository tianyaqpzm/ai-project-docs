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

---

## 补充方案：双令牌身份持久化方案 (2026-05-05 更新)

在完成授权请求的无状态化后，针对登录成功后的身份持久化，系统进一步实施了 **“HttpOnly Cookie + LocalStorage”** 的双令牌方案。

### 1. 核心决策
- **HttpOnly Cookie (jwt_token)**：作为网关层认证的“真身”。由网关签发，设置 `HttpOnly: true` 隔绝 JS 访问，防御 XSS。
- **LocalStorage (jwt_token)**：作为前端 UI 交互的“影子”。由网关通过 URL 参数传回前端，方便 Angular 实时感知登录态及解析用户信息。

### 2. 实现流程 (Implementation Flow)
整个认证与持久化过程如下：

1.  **登录成功 (Gateway)**：用户通过 Casdoor (OAuth2) 登录成功后，网关的 `authenticationSuccessHandler` 被触发。
2.  **签发 JWT (Gateway)**：网关根据用户信息生成加密的 JWT Token。
3.  **写入 Cookie (Gateway)**：网关将 Token 写入 `jwt_token` Cookie，并强制开启 `HttpOnly` 和 `Secure` 属性。
4.  **影子令牌重定向 (Gateway -> Frontend)**：网关将用户重定向回前端，并在 URL 后缀附加 `?token=...` 参数。
5.  **提取与持久化 (Frontend)**：前端 Angular 的 `AuthService` 拦截 URL 参数，将 Token 提取并存入 `localStorage`，随后清理 URL（`replaceState`）以防 Token 泄露在浏览历史中。
6.  **双模识别 (Gateway)**：后续请求进入网关时，`JwtAuthenticationFilter` 会优先检查 Cookie。若 Cookie 缺失（如非浏览器客户端），则回退检查 `Authorization: Bearer` Header。

### 3. 安全与性能考量
- **防 XSS 劫持**：即便前端 `localStorage` 暴露，核心认证依然由无法被 JS 读取的 Cookie 保护。
- **UI 响应性**：前端无需发起网络请求即可从本地 Token 解析出用户姓名、头像，实现了秒级的 UI 状态恢复。
- **拦截优先级**：网关 `JwtAuthenticationFilter` 严格遵循 **“Cookie 优先”** 原则。

### 4. 环境约束
- **跨域支持**：`jwt_token` 必须配置正确的 `.domain` 属性（如 `.122577.xyz`）以支持多级域名下的 SSO 自动登录。
