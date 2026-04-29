# 问题复盘：ms-java-gateway Metaspace OOM 故障

## 基本信息
- **日期**: 2026-04-28
- **服务**: ms-java-gateway
- **问题描述**: 服务在处理 OAuth2 回调请求 `/login/oauth2/code/casdoor` 时抛出 `java.lang.OutOfMemoryError: Metaspace` 异常，导致请求中断。

## 故障现象
- 日志显示 `JwtAuthenticationFilter` 放行白名单路径后，紧接着发生 `Metaspace OOM`。
- 异常线程为 `reactor-http-epoll-1`。

## 根因分析 (RCA)
1.  **JVM 参数限制**: `ms-java-gateway` 的 `Dockerfile` 中显式设置了 `-XX:MaxMetaspaceSize=64m`。
2.  **框架负载**: Spring Boot 3.x 结合 Spring Cloud Gateway 和 Spring Security OAuth2 Client 涉及大量的类加载（如过滤器链、序列化工具、动态代理等）。
3.  **触发场景**: OAuth2 登录流程会触发额外的安全相关类加载。64MB 的 Metaspace 空间不足以承载这些元数据，导致 JVM 无法为新加载的类分配空间。

## 解决方案
- **修改 Dockerfile**: 将 `MaxMetaspaceSize` 从 `64m` 提高到 `128m`。
- **优化内存分配**: 同步将 `Xmx` 和 `Xms` 从 `128m` 提高到 `256m`，以确保在高并发场景下的堆空间充足。
- **改用 G1GC**: 弃用 `SerialGC`，改用 `G1GC` 以获得更好的吞吐量和更短的停顿时间。
- **增强诊断**: 添加 `-XX:+HeapDumpOnOutOfMemoryError` 参数。

## 预防措施
- **基准测试**: 在引入重型框架组件（如 OAuth2, RAG 相关库）后，应进行基础的负载测试以观察内存抖动。
- **监控告警**: 后续应在 Prometheus/Grafana 中添加对 Metaspace 使用率的告警。
- **规范统一**: 确保所有 Java 微服务的 JVM 基础配置保持一致（参考 `ms-java-biz` 的配置标准）。

## 经验总结
- 网关虽然定位为“轻量级”流量层，但作为 Spring Boot 应用，其基础类加载开销依然存在，不能过度压低 Metaspace 限制。
- 64MB 是 Spring Boot 3 应用的 Metaspace 危险红线。
