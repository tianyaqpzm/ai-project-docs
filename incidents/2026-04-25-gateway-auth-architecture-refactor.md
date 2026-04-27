# 事故复盘报告：网关认证重定向死循环及架构纠正方案

## 1. 问题描述
用户在访问应用时遇到无限重定向死循环，具体表现为：
1. **重定向循环**：访问前端后，成功在 Casdoor 登录并带 Token 回调，但前端随即又触发登出并重新跳转登录。
2. **鉴权失效**：手动调用后端接口时，即使 Header 携带了正确的 `Authorization: Bearer <Token>`，网关依然返回 `401 Unauthorized`。
3. **日志告警**：网关后台持续出现 `WARN` 日志：`【未授权访问】Reason: Not Authenticated`。

## 2. 根因分析
该问题由 frontend 和 gateway 的三处架构与配置冲突共同导致：

### 2.1 前端：HashLocationStrategy 引起的 Token 提取失效
Angular 应用使用了 `HashLocationStrategy`。当 Casdoor 回调时，URL 结构可能变为 `/?token=xxx#/chat`。
- 原有的 `AuthService` 仅通过 `window.location.search` 提取参数。
- 在路由初始化过程中，hash 外部的查询参数容易被忽略，导致 `AuthGuard` 无法提取 Token 并存入 `localStorage`，从而认定用户未登录。

### 2.2 网关：CORS & OPTIONS 预检请求被拦截
- 浏览器在发送跨域带自定义 Header 的请求前，会发送 `OPTIONS` 预检请求。
- 网关原有的鉴权 Filter 未能放行 `OPTIONS`，导致浏览器端预检请求 401，直接中断了后续的业务请求。

### 2.3 网关：Spring Security 与 GlobalFilter 的执行顺序冲突（核心原因）
- **生命周期隔离**：Spring Security 的 `WebFilter` 链执行顺序**早于**网关的 `GlobalFilter`。
- **解析位置错误**：原方案将 JWT 解析放在了 `GlobalFilter` 中。
- **结果**：当请求到达 Security 拦截器时（执行 `.authenticated()` 判定），`GlobalFilter` 还没运行，Security 发现上下文（SecurityContext）为空，直接拦截并返回 401。

## 3. 最终架构方案 (Architectural Correction)
为了彻底解决冲突，我们遵循 **“谁负责拦截，谁负责解析”** 的原则进行了重构。

### 3.1 前端：全路径 Token 兼容提取
- 更新 `AuthService` 提取逻辑，同时扫描 `search` 和 `hash` 分段，确保无论 Token 位于 URL 的哪个位置都能被捕获。

### 3.2 网关：鉴权前置到 Security 生命周期
- **Filter 重构**：将 `JwtAuthenticationFilter` 从 `GlobalFilter` 改为 **`WebFilter`**。
- **身份注入**：在 `WebFilter` 中解析 Token，并使用 `UsernamePasswordAuthenticationToken` 将身份注入 `ReactiveSecurityContextHolder`。
- **链路集成**：在 `SecurityConfig` 中通过 `.addFilterAt(..., AUTHENTICATION)` 将该过滤器插入 Security 过滤器链。
- **恢复鉴权策略**：将网关安全策略恢复为 `.anyExchange().authenticated()`，此时请求到达该判定点时，身份已由前置的 WebFilter 填充，能够正常放行。

### 3.3 跨域与预检优化
- **显式放行**：在网关配置中显式放行 `HttpMethod.OPTIONS`。
- **CORS 统一管理**：在 `SecurityConfig` 中通过 `CorsConfigurationSource` 统一配置跨域白名单和 Header 权限。

### 3.4 下游信息透传 (Header Injection)
- **明文透传**：`JwtAuthenticationFilter` 在解析 Token 后，通过 `mutate().header()` 方式，将 `X-User-Id`、`X-User-Name`、`X-User-Avatar` 以明文 Header 形式注入请求。
- **效果**：下游的 Java 业务微服务和 ms-py-agent 无需再处理复杂的 JWT 逻辑，直接通过 Request Header 获取用户信息，实现性能优化与架构解耦。

## 4. 经验总结
1. **拦截原则**：鉴权逻辑必须前置于路由逻辑。在集成 Spring Security 时，自定义认证 Filter 必须实现为 `WebFilter`。
2. **上下文隔离**：深刻理解 `GlobalFilter` 对 Security 上下文是不可见的，必须在 Security 链路内部完成身份注入。
3. **架构解耦**：网关应承担“认证中心”职责，将解析后的明文用户信息（$X-User-*$）下发给业务服务，降低业务复杂度。
 * SecurityConfig：负责 OAuth2 握手、登录成功后签发 JWT Cookie、以及全局安全策略。
 * JwtAuthenticationFilter：负责在后续请求中拦截 Cookie、解析用户信息并透传给 downstream。
