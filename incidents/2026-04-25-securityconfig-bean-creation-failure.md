# SecurityConfig Bean 创建失败 — 启动崩溃

## 故障现象

```
Error creating bean with name 'securityConfig': Injection of autowired dependencies failed
```

Gateway 服务启动即崩溃，无法提供任何服务。

## 影响范围

- **直接影响**: `gateway-service` 完全不可用，所有前端请求 (ms-ng-view)、Agent 请求 (ms-py-agent) 均无法到达后端。
- **间接影响**: 整个系统全挂 — 登录、聊天、知识库等所有功能不可用。

## 根因分析 (RCA)

### 问题定位

`SecurityConfig.java` 第 58 行：

```java
@Value("${spring.security.ignore.urls}")
private String[] ignoreUrls;
```

### 根本原因

`@Value` 注解**无法**将 YAML 列表格式（`-` 前缀的 list）直接绑定到 `String[]`。

YAML 中的配置为：
```yaml
spring:
  security:
    ignore:
      urls:
        - /api/public/**
        - /favicon.ico
        - /actuator/health
        - /login/**
        - /error
        - /oauth2/**
        - /logout
```

Spring `@Value` 对这种嵌套 list 的解析存在歧义，尤其当 Nacos 远程配置合并后，属性 key 格式可能不一致，导致注入失败。

### 矛盾点

项目中其实**已经存在** `IgnoreWhiteProperties` 类通过 `@ConfigurationProperties(prefix = "spring.security.ignore")` 正确绑定了这个列表，`JwtAuthenticationFilter` 也已经在使用它。`SecurityConfig` 是重复绑定了同一份数据，且使用了错误的方式。

## 修复方案

将 `SecurityConfig` 中的 `@Value` 注入替换为构造器注入已有的 `IgnoreWhiteProperties` Bean：

```diff
- @Value("${spring.security.ignore.urls}")
- private String[] ignoreUrls;
+ private final IgnoreWhiteProperties ignoreWhiteProperties;
+
+ public SecurityConfig(IgnoreWhiteProperties ignoreWhiteProperties) {
+     this.ignoreWhiteProperties = ignoreWhiteProperties;
+ }

  @Bean
  public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
+     String[] ignoreUrls = ignoreWhiteProperties.getUrls().toArray(new String[0]);
      http
          .authorizeExchange(
              exchanges -> exchanges
                  .pathMatchers(ignoreUrls)
                  .permitAll().anyExchange().authenticated())
```

## 涉及文件

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| `ms-java-gateway/.../SecurityConfig.java` | MODIFY | 移除 `@Value`，改用构造器注入 `IgnoreWhiteProperties` |

## 经验教训

1. **避免 `@Value` 绑定复杂类型**：对于 YAML list/map 类型的配置，始终使用 `@ConfigurationProperties` 进行类型安全绑定。
2. **消除重复绑定**：同一份配置数据不应在多个地方以不同方式绑定，应复用已有的 Properties Bean。
3. **Nacos 配置合并风险**：远程配置与本地配置合并后，`@Value` 对属性 key 的解析可能与预期不同。
