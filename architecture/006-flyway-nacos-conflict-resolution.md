# ADR 006: Flyway 与 Nacos 启动冲突解决方案

## 背景 (Context)
在微服务启动期间，Nacos 的上下文刷新与 Flyway 的数据库锁机制存在冲突。具体表现为：
1.  **资源竞争**：Nacos 在 `Bootstrap` 阶段的异步配置刷新会触发 `HikariDataSource` 的销毁与重建，导致 Flyway 正在执行的长连接被物理切断。
2.  **会话恢复陷阱**：PostgreSQL 驱动在连接关闭前会尝试还原会话状态（如 `search_path`），在 PgBouncer 或网络不稳时会触发 `EOFException` / `Connection Reset`。
3.  **版本不兼容**：Flyway 9.x 对 PostgreSQL 16 的协议支持不完善。

详见根因分析：[20260428-flyway-nacos-connection-reset](../incidents/20260428-flyway-nacos-connection-reset.md)

## 可选方案对比 (Alternatives Considered)

| 方案 | 优点 | 缺点 | 结论 |
| :--- | :--- | :--- | :--- |
| **A. 共享数据源模式 (默认)** | 配置简单。 | 极易受 Nacos 刷新干扰，连接池中途销毁导致崩溃。 | **拒绝** |
| **B. 外部迁移模式 (CI/CD)** | 完全解耦。 | 增加部署复杂度，本地开发环境不便。 | **弃用** |
| **C. 隔离模式 + 咨询锁** | 解决 Bean 冲突。 | 咨询锁依赖会话持久性，在不稳定链路上清理状态时易崩溃。 | **弃用** |
| **D. 隔离模式 + 表级锁 + 驱动补丁** | **较稳健**。锁随事务自动释放，支持 PG 16。 | **不充分**。无法规避启动瞬间 Nacos 刷新导致的物理连接销毁风险。 | **阶段性采纳 (04-28)** |
| **E. 延迟初始化 + 被动会话管理 (当前采用)** | **最稳健**。彻底避开启动刷新竞争，规避驱动状态恢复陷阱，适配代理环境。 | 需要维护 `FlywayConfig` 配置类。 | **最终采纳 (05-01)** |

## 决策 (Decision)

### 1. 生命周期解耦 (Lifecycle Decoupling)
*   **手动控制**：设置 `spring.flyway.enabled: false`，关闭 Spring Boot 自动装配。
*   **事件触发**：编写 `FlywayConfig` 监听 `ApplicationReadyEvent`，确保在 Nacos 刷新完成且应用处于“稳态”后执行。

### 2. 物理隔离与被动管理 (Isolated & Passive Management)
*   **独立直连**：配置 `spring.flyway.url` 使用独立连接，不与业务 HikariCP 共享，防止刷新波及。
*   **参数静态化**：在 JDBC URL 中硬编码 `options=-c search_path=public`。
    *   *原理*：使驱动认为会话状态从未被修改，从而跳过脆弱的 `restoreOriginalState` 逻辑。

### 3. 适配 PgBouncer 代理 (Proxy Compatibility)
*   **锁模式切换**：通过配置 `flyway.postgresql.transactional.lock: false` 规避 PgBouncer 对咨询锁的不支持，改用兼容性更好的同步机制或依赖独立直连的稳定性。
*   **重试增强**：设置 `connect-retries: 20` 增加对云端网络波动的容错。

### 4. 基础设施对齐 (Infrastructure Alignment)
*   **强制升级**：Flyway 升级至 `11.1.0`，驱动升级至 `42.7.4+` 以全面支持 PostgreSQL 16 协议。

## 影响 (Consequences)
- **正面**: 彻底消除由于 Nacos 刷新、PgBouncer 代理或网络抖动导致的启动循环。
- **负面**: 增加了一点配置代码复杂度，且需维护两组数据库连接参数。

## 状态 (Status)
Accepted

## 关联参考
- [FlywayConfig.java](../../ms-java-biz/src/main/java/com/dark/aiagent/config/FlywayConfig.java)
- [application.yaml](../../ms-java-biz/src/main/resources/application.yaml)
