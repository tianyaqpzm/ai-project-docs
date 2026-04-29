# ADR 006: Flyway 与 Nacos 启动冲突解决方案

## 背景 (Context)
在微服务启动期间，Nacos 的上下文刷新与 Flyway 的数据库锁机制存在冲突，导致远程连接被重置，产生 `EOFException`。
详见根因分析：[RAG-20260428-连接重置](../incidents/20260428-flyway-nacos-connection-reset.md)

## 可选方案对比 (Alternatives Considered)

| 方案 | 优点 | 缺点 | 结论 |
| :--- | :--- | :--- | :--- |
| **A. 共享数据源模式 (默认)** | 配置简单，连接数利用率高。 | 极易受 Nacos `ContextRefreshEvent` 干扰，导致连接池中途销毁，Flyway 启动崩溃。 | **拒绝** |
| **B. 外部迁移模式 (CI/CD / Init-Container)** | 应用启动零负载，完全解耦。 | 增加 DevOps 部署复杂度，本地开发环境难以模拟，不符合当前快速迭代需求。 | **弃用** |
| **C. 隔离模式 + 会话锁 (咨询锁)** | 解决了 Bean 生命周期冲突。 | 依赖 PG 会话持久性。在网络抖动或驱动 (42.6.x) 下，清理会话状态时仍会触发 EOF。 | **弃用** |
| **D. 隔离模式 + 表级锁 + 驱动补丁 (本项目采用)** | **最稳健**。锁随事务自动释放，不依赖会话；物理隔离不受 Nacos 刷新影响。 | 增加少量配置代码；迁移期间会持有元数据表的排他锁。 | **采纳** |

## 决策 (Decision)

1.  **物理隔离 (Isolated Connection)**
    *   Flyway 配置独立直连参数（`spring.flyway.url/user/password`）。
    *   `bootstrap.yaml` 禁用启动期的配置刷新（`refresh: false`）。

2.  **表级锁强化 (Table-Level Locking)**
    *   设置 `spring.flyway.postgresql.transactional.lock: true`。
    *   *原理*：改用 `LOCK TABLE ... IN EXCLUSIVE MODE`。该锁模式不依赖会话恢复 (restoreOriginalState)，在事务结束时由内核自动回收。

3.  **驱动基线与原子化 (Driver & Atomicity)**
    *   强制升级 `postgresql` 驱动至 `42.7.4` 以支持 PG 16 协议优化。
    *   启用 `execute-in-transaction: true` 保证锁与逻辑的原子边界。

## 影响 (Consequences)
- **正面**: 彻底消除由于 Nacos 刷新或网络抖动导致的启动异常。
- **负面**: 数据库配置需维护两份占位符（Hikari 和 Flyway），需通过环境变量统一。

## 状态 (Status)
Accepted
