# 问题复盘 (Incident Postmortem) - relation "knowledge_topic" does not exist

## 1. 问题描述 (Incident Description)
在 `ms-java-biz` 启动时，数据库迁移（Flyway）失败，抛出异常如下：
```
ERROR: relation "knowledge_topic" does not exist
Location: db/migration/V1.12__update_recipe_topic_template_name.sql
```
这导致微服务初始化中断，无法对外提供服务。

## 2. 根因分析 (Root Cause Analysis - RCA)
1. **表名变更历史不一致**：
   - 在迁移脚本 `V1.1__knowledge_schema.sql` 中，数据库创建了 `knowledge_topic` 表。
   - 在随后版本的 `V1.4__rename_historical_tables.sql` 中，根据数据库命名规范（统一加上 `ms_` 前缀且使用单数形式），该表已被重命名为 `ms_knowledge_topic`。
   - 在迁移脚本 `V1.12__update_recipe_topic_template_name.sql` 中，开发者编写了更新该表的 SQL 语句，但错误地引用了已经不存在的旧表名 `knowledge_topic`，从而导致迁移抛出 "relation does not exist" 异常。

2. **开发/测试环境的迁移历史污染**：
   - 第一次执行失败后，Flyway 在 `flyway_schema_history` 表中留下了标志为失败（`success = false`）的 `V1.12` 记录。
   - 即使直接修复 SQL 文件内容重新启动，Flyway 默认也会报错并拒绝迁移，因为存在失败的迁移历史，需要手动或自动执行 `repair()` 操作以清理该记录。

## 3. 解决方案与实施 (Resolution & Action Items)
1. **修复 SQL 脚本表名引用**：
   - 修改 [V1.12__update_recipe_topic_template_name.sql](file:///Users/pei/projects/ms-java-biz/src/main/resources/db/migration/V1.12__update_recipe_topic_template_name.sql)，将表名 `knowledge_topic` 更正为 `ms_knowledge_topic`。

2. **升级 Flyway 初始化安全性 (自动修复机制)**：
   - 修改 [FlywayConfig.java](file:///Users/pei/projects/ms-java-biz/src/main/java/com/dark/aiagent/config/FlywayConfig.java)，在 `onApplicationReady` 执行 `migrate()` 之前，先调用 `flyway.repair()`。
   - 该机制能够自动检测并清理开发与测试环境中因网络波动、脚本语法错误等遗留的已失败 Flyway 记录，并重新同步迁移文件校验和，显著提升本地与 CI 开发环境下的鲁棒性。

## 4. 经验总结 (Key Takeaways)
- **严格遵循数据库命名契约**：所有涉及已重命名表的后期 SQL 脚本（如 V1.4 之后），必须强制使用正确的带有 `ms_` 前缀的新表名，确保脚本的时间线一致性。
- **自动修复优于手动干预**：在非生产环境的后台启动策略中引入 `flyway.repair()` 能够彻底规避由于修改待迁移文件导致的 checksum 校验异常与残留脏记录，保障团队开发体验和流水线稳定性。
