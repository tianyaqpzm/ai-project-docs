# 事故复盘报告：本地开发环境 OAuth2 认证失败复盘

## 1. 问题描述
在架构重构完成后，本地调试过程中依然出现认证无法通过的现象：
1. **认证报错**：OAuth2 回调时报 `/login?error`。
2. **Token 丢失**：前端成功重定向后无法在 `localStorage` 中找到 Token。
3. **Cookie 写入失败**：浏览器控制台显示认证相关的 Cookie 因 Domain 不匹配被拒绝。

## 2. 根因分析
该阶段的问题主要由环境配置与本地浏览器安全限制导致：

### 2.1 Cookie Domain 域名不匹配
*   **现象**: 网关配置设置了 `app.cookie-domain: 122577.xyz`。
*   **根因**: 本地访问地址为 `localhost`，浏览器出于安全考虑，禁止将 Cookie 写入非后缀匹配的域名，导致 Session 无法建立。Session 的丢失直接导致 OAuth2 的 `state` 校验在回调阶段失效。

### 2.2 客户端-服务端认证反馈闭环缺失
*   **现象**: 前端无法读取 HttpOnly Cookie。
*   **根因**: 网关仅通过 Cookie 模式返回 Token。在本地调试这种 Cookie 极易被策略拦截的环境下，一旦 Cookie 写入失败，前端没有任何其他途径获取认证状态。

### 2.3 配置参数失效
*   **根因**: `client-secret` 由于环境变量未同步，导致在 code 换取 token 阶段被 Casdoor 拒绝（`invalid_client`）。

## 3. 修复措施

### 3.1 增强认证反馈模式 (双通路)
*   **URL 参数兜底**: 修改 `authenticationSuccessHandler`，在成功后除了设置 Cookie，同时将 Token 作为 Query Parameter 拼接到回调 URL 中。
*   **前端适配**: `AuthService` 支持从 URL 参数提取 Token 并持久化。

### 3.2 环境适配优化
*   **动态配置**: 在本地启动脚本或 Nacos 中将 `app.cookie-domain` 设为空，以便开发环境能正常使用 `localhost` 的 Session。
*   **日志可见性**: 在 `SecurityConfig` 中增加详细的 `authenticationFailureHandler` 日志输出，能够第一时间定位 `client_secret` 错误等配置问题。

## 4. 经验总结
*   **防御式重定向**: 在跨域或本地调试场景下，不应透过度依赖 Cookie 传递首个认证状态。
*   **日志可见性**: 对于 OAuth2 这种多系统协作的流程，失败回调的日志采集至关重要（避免“盲调”）。
