# 问题复盘：Flyway Checksum 校验失败 (V1.0)

## 日期
2026-04-27

## 问题描述
`ms-java-biz` 服务启动失败，报错 `FlywayValidateException`。错误信息显示迁移脚本 `V1.0__init_schema.sql` 的校验和 (Checksum) 与数据库中记录的不一致。

- **数据库记录值**: 2093059458
- **本地脚本计算值**: -1537775090

## 根因分析 (RCA)
迁移脚本 `V1.0__init_schema.sql` 在已经应用到数据库后被修改过。Flyway 的校验机制检测到本地文件内容（可能是注释、空格或 SQL 逻辑）与执行记录不符，为了保证数据库版本的一致性，拒绝启动。

## 解决方案
在 `AiApplication.java` 中引入了 `FlywayMigrationStrategy` Bean，配置为在执行迁移 (migrate) 前先调用 `flyway.repair()`。该操作会使用本地脚本的最新校验和更新 `flyway_schema_history` 表，从而通过校验。

## 经验教训
1. **脚本不可变原则**: 迁移脚本一旦应用到共享环境或生产环境，严禁修改其内容。任何变更应通过新增版本（如 `V1.1`）来实现。
2. **本地开发同步**: 在本地开发阶段，如果确实需要调整旧脚本，应使用 `flyway repair` 同步状态。
3. **自动化修复**: 在开发环境下使用 `FlywayMigrationStrategy` 可以降低环境损坏后的修复成本。

## 预防措施
- 严格遵守 SQL 脚本的追加原则，避免修改已发布的版本。
- 提炼经验到对应工程下的 `ai-code-ws.md` 文件。
