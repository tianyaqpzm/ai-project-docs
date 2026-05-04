# RCA: Flyway 迁移导致生产级启动异常 (EOFException/Connection Reset)

## 问题摘要 (Summary)
*   **故障现象**：`ms-java-biz` 在启动引导阶段，Flyway 执行 `migrate` 或 `repair` 时频繁抛出 `Unable to restore connection`，底层表现为 `java.io.EOFException` 或 `Connection reset`。
*   **影响范围**：服务无法进入 Ready 状态，造成启动循环。

## 关键证据 (Critical Evidence)

### 1. Flyway 会话恢复失败 (Session Restoration Failure)
当 Flyway 尝试清理会话状态时，由于连接被物理切断导致的典型 EOF 异常：
```text
Caused by: org.flywaydb.core.internal.exception.FlywaySqlException: Unable to restore connection to its original state
SQL State  : 08006
Message    : An I/O error occurred while sending to the backend.
...
Caused by: org.flywaydb.database.postgresql.PostgreSQLConnection.doRestoreOriginalState(PostgreSQLConnection.java:48)
Caused by: org.postgresql.util.PSQLException: An I/O error occurred while sending to the backend.
Caused by: java.io.EOFException: null
	at org.postgresql.core.PGStream.receiveChar(PGStream.java:467)
```

### 2. Nacos 异步刷新销毁连接池 (HikariCP Shutdown)
Nacos 在引导阶段触发的 Context Refresh 物理销毁了正在使用的连接池：
```text
2026-04-28T20:51:13.043Z  WARN [ms-java-biz] [Thread-5] c.a.nacos.common.notify.NotifyCenter : [NotifyCenter] Start destroying Publisher
2026-04-28T20:51:13.043Z  WARN [ms-java-biz] [Thread-5] c.a.nacos.common.notify.NotifyCenter : [NotifyCenter] Destruction of the end
```

## 深度根因分析 (Deep Root Cause)

本次事故是三个维度的冲突叠加导致的：

### 1. 物理层：Nacos 刷新与 Flyway 的“资源竞争”
Nacos 在服务启动瞬间会触发异步配置刷新（Context Refresh）。若 Flyway 与业务系统共享 HikariCP 连接池，刷新动作会强制销毁当前池内所有连接。由于 Flyway 迁移属于长事务/长连接操作，连接被物理切断直接导致其无法通过 `SET` 指令恢复会话状态。

### 2. 协议层：Flyway 9.x 与 PostgreSQL 16 的“版本代差”
Spring Boot 3.2 默认集成的 Flyway 9.22.3 尚未对 PostgreSQL 16 进行完整适配。PG 16 在处理旧版客户端发送的 `session_replication_role` 或 `search_path` 重置请求时，由于驱动协议的微小差异，会因安全或状态异常主动断开 Socket 连接。

### 3. 逻辑层：会话级锁的“鲁棒性缺陷”
Flyway 默认在 PostgreSQL 上使用“咨询锁 (Advisory Lock)”，该锁严格绑定到当前数据库 Session。在不稳定的网络或配置刷新环境下，Session 一旦中断，锁无法优雅释放或恢复，导致 `restoreOriginalState` 逻辑进入死循环并最终抛出 I/O 错误。

## 最终解决方案 (Architecture-Level Fix)

### 1. 物理隔离：独立直连 (DataSource Isolation)
**策略**：强制 Flyway 避开业务连接池，使用独立 JDBC 连接。
**配置**：在 `spring.flyway` 下直接显式配置 `url/user/password`，确保迁移操作不受 Nacos 配置刷新导致的连接池销毁影响。

### 2. 版本升级：全面适配 PG 16 (Version Alignment)
**策略**：升级基础设施组件以支持高版本数据库协议。
*   **Flyway**：升级至 `10.15.0+`（Flyway 10 正式支持 PG 16）。
*   **驱动**：升级 PostgreSQL JDBC 驱动至 `42.7.4+`，修复会话状态清理时的鲁棒性。
*   **模块化**：显式引入 `flyway-database-postgresql`（Flyway 10 的新要求）。

### 3. 锁降级：事务级表锁 (Robust Locking)
**策略**：将脆弱的会话级咨询锁切换为事务级表锁。
**配置**：设置 `flyway.postgresql.transactional.lock: true`。表级锁不依赖 Session 状态，对 PgBouncer 代理或不稳定链路具有极强的容错性。

## 无效尝试与误区 (Invalid Attempts & Pitfalls)

### 1. 试图通过 `@DependsOn` 强制 Bean 顺序
*   **做法**：在应用层通过 `@DependsOn("flywayInitializer")` 强制业务 Mapper 等待 Flyway 完成。
*   **结果**：**失败**。
*   **教训**：本次故障的本质是 Nacos 异步刷新导致的 **数据库连接被物理销毁**。代码层面的依赖控制（DI 顺序）无法修复已经被操作系统回收的 Socket 连接。

### 2. 试图禁用 Flyway 事务 (`execute-in-transaction: false`)
*   **做法**：尝试以非事务方式运行迁移，期望避开会话恢复逻辑。
*   **结果**：**失败**。
*   **教训**：PostgreSQL 16 对会话状态极为敏感。无论是否开启事务，旧版驱动在退出时仍会触发重置指令并因不匹配导致 EOF。解决之道在于协议对齐，而非逃避事务。

## 经验总结 (Final Lessons Learned)
*   **物理隔离优于逻辑控制**：数据库迁移工具应始终拥有独立的 DataSource 直连，不与业务连接池共享生命周期，以规避配置中心刷新带来的资源竞争。
*   **基础设施版本对齐**：升级核心中间件（如 PostgreSQL 16）时，必须同步评估迁移工具（Flyway）和驱动的协议兼容性，不能仅依赖框架默认版本。
*   **拒绝流水账式排查**：在遇到 `EOFException` 等底层网络错误时，应优先从物理链路（连接池、代理、协议版本）入手，而非仅在应用代码层面寻找逻辑补丁。

## 补充方案：极致延迟初始化与被动会话管理 (2026-05-01 更新)

在 04-28 方案的基础上，针对部分云端环境（特别是通过 PgBouncer 代理连接的集群）仍然出现的 `Unable to restore connection` 顽疾，我们实施了更为彻底的“双保险”方案。

### 1. 启动时序重构：ApplicationReadyEvent 手动触发
*   **做法**：在 `application.yml` 中设置 `spring.flyway.enabled: false`。
*   **核心逻辑**：编写 `FlywayConfig` 配置类，利用 `@EventListener(ApplicationReadyEvent.class)` 监听应用就绪事件。
*   **收益**：确保 Flyway 在 Nacos 客户端心跳、HikariCP 连接池就绪以及所有环境变量完全注入后的“稳态”下启动。**彻底消除了 Bootstrap 阶段 Nacos 异步刷新导致的物理连接销毁风险。**

### 2. 被动会话管理：绕过 `restoreOriginalState` 陷阱 (关键突破)

#### 1). PgBouncer 事务模式冲突 (PgBouncer Transaction Mode Conflict)
*   **核心矛盾**：数据库通过 PgBouncer 的 `transaction` 模式暴露。在此模式下，代理要求连接在事务结束后立即归还池，且尽量保持无状态。
*   **死锁点**：
    1.  Flyway 默认在迁移前后发送 `SET search_path` 和获取 `Advisory Lock`（咨询锁）。
    2.  PgBouncer 会拦截或标记这些 Session 级指令。
    3.  由于地理跨度带来的网络延迟，当 Flyway 驱动在操作结束尝试 `restoreOriginalState`（还原会话）时，物理连接可能已被代理由于安全或超时策略强制断开。
*   **表现**：驱动在发送还原指令时收到 `EOF` 信号，抛出 `Unable to restore connection`。

#### 2). Nacos 异步刷新竞争 (Nacos Bootstrap Race Condition)
*   **现象**：在 Spring Boot 启动的 `Bootstrap` 阶段，Nacos 客户端会多次刷新配置。
*   **冲突**：若 Flyway 在刷新间隙开始执行，Nacos 导致的 `ApplicationContext` 部分刷新可能会触发 `HikariDataSource` 的销毁与重建，直接中断正在进行的数据库迁移长连接。

*   **做法**：在代码配置中 **移除** `.schemas("public")` 调用，改为在 JDBC URL 中硬编码 `options=-c search_path=public`。
*   **深度原理**：Flyway 的报错源于驱动尝试重置会话状态（如 `search_path`）。
    *   **默认行为**：Flyway API 显式指定 schema -> 驱动发送 `SET search_path` -> 退出时驱动尝试 `RESTORE` -> 连接不稳时触发 EOF。
    *   **优化行为**：URL 静态指定 -> 驱动认为会话状态从未被工具修改过 -> 退出时跳过 `restoreOriginalState` 逻辑 -> **即使连接有抖动也不会抛出状态恢复异常。**

### 3. 适配代理环境：禁用咨询锁 (Proxy-Friendly Locking)
*   **做法**：配置 `Map.of("flyway.postgresql.transactional.lock", "false")`。
*   **核心逻辑**：PgBouncer 在 **Transaction Mode** 下不支持会话级咨询锁。强制 Flyway 使用常规的表级排他锁，既保证了迁移的互斥性，又规避了因代理层不支持会话锁而导致的事务回滚。

## 微服务环境下的 Flyway 治理准则 (Best Practices)
1.  **参数静态化原则**：所有会话级参数（`search_path`, `statement_timeout`）应优先通过 JDBC URL 传递，而非通过 Flyway API 或 SQL 指令动态设置，以最大限度减少驱动的状态追踪负担。
2.  **生命周期解耦**：在分布式配置中心（Nacos/Apollo）环境下，建议关闭 Flyway 自动装配，改用 `ApplicationReadyEvent` 手动触发，确保所有动态配置已“尘埃落定”。
3.  **连接池独立化**：迁移连接不应与业务请求共享同一个连接池，应使用独立的、长超时的直连配置，防止业务高并发干扰迁移过程。

## 关联决策与代码
*   [FlywayConfig.java](../../ms-java-biz/src/main/java/com/dark/aiagent/config/FlywayConfig.java)
*   [application.yaml](../../ms-java-biz/src/main/resources/application.yaml)
*   [ADR-006: Flyway-Nacos 冲突解决方案](../architecture/006-flyway-nacos-conflict-resolution.md)
