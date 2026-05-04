# RCA: ms-java-biz 启动时产生生成的安全密码日志

## 问题描述
在 `ms-java-biz` 服务启动时，控制台输出如下日志：
`Using generated security password: e0222719-2068-4baa-af82-ee18ab8041b9`
`This generated password is for development use only. Your security configuration must be updated before running your application in production.`

## 根因分析
该现象是 Spring Security 的默认安全配置机制导致的：
1. **依赖触发**：项目中引入了 `spring-boot-starter-security` 依赖。
2. **自动化配置激活**：Spring Boot 的 `UserDetailsServiceAutoConfiguration` 会在容器中未发现 `UserDetailsService`、`AuthenticationProvider` 或 `AuthenticationManager` Bean 时自动激活。
3. **默认行为**：由于该项目采用 **无状态 (Stateless) 的 JWT 认证架构**，业务逻辑完全由 `JwtAuthenticationFilter` 处理，未显式定义 `UserDetailsService`。Spring Boot 认为需要一个默认用户以防“裸奔”，因此创建了名为 `user` 的内存用户并生成随机密码。

## 解决方案
在 `SecurityConfig.java` 中显式定义一个空的 `UserDetailsService` Bean。这会覆盖 Spring Boot 的默认行为，告诉安全框架我们已经接管了用户来源，从而禁止随机密码的生成。

```java
@Bean
public UserDetailsService userDetailsService() {
    // 返回一个空的内存用户管理器，防止 Spring Boot 自动生成随机密码
    return new InMemoryUserDetailsManager();
}
```

## 经验总结
在构建 **无状态 JWT 认证** 的微服务时，即使不使用基于表单或 Basic 的认证方式，也应显式定义一个 `UserDetailsService`（哪怕是空的），以保持日志整洁并明确应用的安全意图，避免 Spring Boot 启动兜底的随机密码生成逻辑。
