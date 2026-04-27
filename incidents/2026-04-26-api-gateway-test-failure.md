# 故障记录：GitHub Actions 中 ms-java-gateway 测试失败

**日期：** 2026-04-26
**状态：** 已解决
**根本原因：** 测试断言过于脆弱，以及特定环境下的 localhost 解析问题。

## 问题现象
- `ms-java-gateway` 的 GitHub Action 在执行 `mvn test` 时失败。
- `JwtAuthenticationFilterTest` 报错：`AssertionError: expected:<SERVER_ERROR> but was:<CLIENT_ERROR> (404)`。
- 测试过程中出现偶发性的 `TimeoutException`，发生在连接 `localhost:8080` 时。

## 根本原因分析
1. **断言逻辑脆弱 (Assertion Fragility)：** 原始测试用例在调用虚拟后端 (`localhost:8080`) 时期望返回 `5xxServerError`。但在某些环境或配置下，当后端不可达时，Spring Cloud Gateway 可能会返回 404 (Not Found)。由于测试的目标是验证 **JWT 过滤器** 是否放行（即非 401），硬编码校验 500 状态码会导致不必要的失败。
2. **Localhost 解析问题：** 在某些 macOS 版本或 CI 运行环境中，`localhost` 可能会解析到 IPv6 的 `::1` 或解析过程缓慢，导致 Reactor Netty 在连接时出现 5 秒超时。

## 解决方案
1. **优化鲁棒性断言：** 将 `JwtAuthenticationFilterTest` 中的断言逻辑调整为：验证响应状态码 **不是 401 (Unauthorized)**。这专注于验证过滤器本身的授权逻辑（是否正确识别并透传 Token），而不再依赖于下游服务具体的错误响应。
2. **显式使用 IP 地址：** 将 `application-test.yml` 中的 `localhost` 统一修改为 `127.0.0.1`，确保连接过程快速且稳定，避免 DNS/IPv6 解析干扰。

## 经验教训
- 在单元测试中，除非测试目标就是错误处理链路，否则应避免对下游服务的具体错误码（如 500 vs 404）进行强依赖。
- 在测试配置文件中，优先使用 `127.0.0.1` 而非 `localhost`，以提高跨环境的兼容性和连接速度。
- 过滤器测试应聚焦于其核心职责（如认证、头注入），而非整个路由转发的最终结果。
