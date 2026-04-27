# 问题复盘 (RCA): ms-java-biz 测试环境强依赖生产中间件

## 事件回顾
**日期**: 2026-04-25
**现象**: `KnowledgeControllerTest` 在本地运行时，由于无法连接到 PostgreSQL (`localhost:5432`) 导致 Context 启动失败。测试尝试执行 Flyway 迁移并连接 Nacos。

## 根因分析
1. **测试策略不足**: 未能在测试环境隔离物理数据库。测试用例直接继承了生产的自动配置。
2. **依赖缺失**: `pom.xml` 缺乏内存数据库（如 H2）支持，导致无法在断网/无本地 PG 情况下运行用例。

## 解决方案
1. **引入 H2**: 在 `pom.xml` 添加 `h2` 依赖，作用域设为 `test`。
2. **配置隔离**: 
   - 移除 `spring.profiles.active` 在 `application-test.yml` 中的非法声明。
   - 配置 `spring.datasource.url` 指向 H2 内存模式。
   - 显式禁用 Flyway、Nacos Discovery。

## 经验提炼
- **自动化保证**: 后端工程必须保证在没有任何外部服务（DB, MQ, Discovery）运行的情况下，`mvn test` 仍能 100% 通过。
- **配置优先级**: 务必熟悉 Spring Boot 加载 `application-{profile}.yml` 的优先级规则，避免在子配置文件中定义冲突的全局开关。
