# 事故复盘：Java 后端 500 异常被网关 403 掩盖

## 1. 现象描述
**日期**：2026-05-01
**症状**：当 `ms-java-biz` 后端发生 SQL 语法错误（500）时，客户端收到的响应码却是 403 Forbidden。网关日志显示 Token 已验证通过。

## 2. 根因分析（RCA）

### 直接原因
后端服务在发生 500 错误后，由于没有自定义的全局异常处理器，Spring Boot 将请求内部转发到了 `/error` 路径。由于 `/error` 路径不在安全白名单内，触发了 Spring Security 的拦截，最终返回了 403。

### 根本原因
1. **安全配置缺失**：`ms-java-biz` 的安全白名单配置（`ignoreUrls`）未包含 Spring Boot 默认的错误转发路径 `/error`。
2. **异常处理机制不完善**：项目初期未建立 `@RestControllerAdvice` 全局异常处理机制，导致异常处理依赖容器默认的 `/error` 转发逻辑。在无状态（Stateless）认证架构下，内部转发可能会因丢失上下文或重新触发过滤器链而导致授权失败。

## 3. 解决过程
1. **即时修复**：在 `application.yaml` 中将 `/error` 加入 `app.security.ignore.urls` 白名单。
2. **架构加固**：引入 `GlobalExceptionHandler`，通过 `@RestControllerAdvice` 捕获所有异常并直接返回标准化 JSON。这使得异常在 Filter 链之后的 Controller 层就被拦截处理并返回，不再触发 `/error` 转发流程，彻底规避此类安全拦截导致的错误掩盖。

## 4. 经验总结与后续行动
- **规范要求**：所有微服务必须在 `SecurityConfig` 中显式放行 `/error` 或提供完备的全局异常处理。
- **最佳实践**：优先使用 `@RestControllerAdvice` 进行异常捕获，避免依赖 Servlet 容器的默认错误处理机制。
- **监控加固**：在网关层加强对 403 异常的监控，识别是否由后端错误掩盖引起。
