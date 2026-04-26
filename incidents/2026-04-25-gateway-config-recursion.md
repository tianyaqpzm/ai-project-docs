# 问题复盘 (RCA): api-gateway 测试环境 YAML 冗余导致启动失败

## 事件回顾
**日期**: 2026-04-25
**现象**: 执行 `mvn test` 时，所有测试用例报错 `IllegalStateException: Failed to load ApplicationContext`。由于 SecurityConfig Bean 创建失败导致上下文崩溃。

## 根因分析
1. **配置错误**: 在 `src/test/resources/application-test.yml` 中，出现了重复的 `spring.security` 根键。
2. **Spring Boot 加载逻辑**:
   ```yaml
   spring:
     security:  # 第一个出现点
       login-url: ...
     ...
     security:  # 第二个出现点，覆盖了之前的配置
       ignore:
         urls: ...
   ```
   YAML 解析器在遇到重复键时，虽然部分实现（如 SnakeYAML）会合并或覆盖，但在 Spring Boot 的特定版本/配置下，导致了 `IgnoreWhiteProperties` 无法被正确注入，进而导致 `SecurityConfig` 构造函数异常。

## 解决方案
合并所有 `spring.security` 下的配置项到唯一的 `security` 键下，确保属性绑定的原子性。

## 经验提炼
- **IDE 告警**: 忽略了 YAML 文件的重复键告警。后续需确保测试环境配置文件的简洁，避免大段粘贴导致冗余。
- **配置隔离**: 生产与测试配置的 diff 需保持清晰，避免在测试配置中保留过多生产环境特有的复杂注释。
