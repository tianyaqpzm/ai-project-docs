# ms-java-gateway 架构重构：零业务逻辑纯粹流量层

## 日期
2026-04-28

## 涉及子工程
- ms-java-gateway（主要变更）
- ms-java-biz（下游影响：需自行解析 JWT payload）
- ms-py-agent（下游影响：需自行解析 JWT payload）

## 问题描述

网关层 `ms-java-gateway` 存在领域逻辑泄漏和架构缺陷：

1. **JwtAuthenticationFilter 越权解析业务字段**：解析 `name`/`username`/`picture`/`avatar` 等 Claims 并注入 `X-User-Name`、`X-User-Avatar` 头，包含 fallback 逻辑（如 `name` → `username`），违反网关零业务逻辑原则
2. **缺少全局链路追踪**：作为流量第一跳未生成 `X-Trace-Id`
3. **SecurityConfig 职责混杂**：内联 logFilter，存在 `HttpStatus` 强转风险
4. **缺少标准化错误响应**：后端异常堆栈可能暴露给前端
5. **包结构扁平**：所有类堆在 `filter` 包下

## 根因分析

- JwtFilter 中的 Claims 字段解析属于"防腐层 (ACL) 适配"逻辑（如 `name` → `username` 的 fallback），应由下游业务服务处理
- 网关不应感知用户实体的结构，否则当 Casdoor 修改 claim 名称时，网关代码需同步修改
- `sub` 是 OIDC 标准字段，提取 `X-User-Id` 属于鉴权基础设施职责，不算业务逻辑

## 解决方案

### 包结构重组
```
filter/ (旧) → config/ + filter/ + handler/ (新)
```

### JwtFilter 瘦身
- 仅注入 `X-User-Id`（标准 OIDC `sub`）
- 透传原始 JWT（`Authorization: Bearer <token>`）
- 下游自行解析 `name`/`avatar` 等业务字段

### 新增组件
| 组件 | 职责 |
|------|------|
| TraceIdFilter (Order -200) | 全局链路追踪，生成/复用 X-Trace-Id |
| GatewayErrorHandler | 统一错误响应，含 traceId，清洗异常消息 |
| ResilienceConfig | 容错配置（连接超时 3s，响应超时 30s） |

## 经验总结

1. **`AbstractErrorWebExceptionHandler` 必须注入 `ServerCodecConfigurer`**：否则启动时抛出 `Property 'messageWriters' is required`，这是 WebFlux 自定义错误处理器的必备步骤
2. **Filter 白名单测试断言不要绑定具体状态码**：应使用 `!= 401` 而非 `== 404`，因为 actuator 等端点启用后会返回 200
3. **SecurityConfig 中的 OAuth2 成功处理器属于基础设施职责**：JWT 签发是 OAuth2 回调流程的一部分，保留在网关中是合理的
4. **TraceIdFilter 的 Order 必须小于 JwtFilter**：确保即使认证失败的请求也有 Trace-ID

## 下游影响

| 头字段 | 变更前 | 变更后 |
|--------|--------|--------|
| X-User-Id | ✅ 存在 | ✅ 保留 |
| X-User-Name | ✅ 存在 | ❌ 移除 |
| X-User-Avatar | ✅ 存在 | ❌ 移除 |
| Authorization | ❌ 不透传 | ✅ Bearer token 透传 |
| X-Trace-Id | ❌ 不存在 | ✅ 新增 |
