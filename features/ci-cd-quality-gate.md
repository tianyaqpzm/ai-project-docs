# 特性定义：CI/CD 自动化测试门禁 (Quality Gate)

## 业务背景
随着系统复杂度的增加和子工程数量的增多（当前为 4 个），手动执行回归测试已无法满足高效迭代的要求。为了保障 `master` 分支的稳定性，必须建立自动化的持续集成门禁，确保任何代码合并前都经过全量用例验证。

## 技术方案
- **平台**: GitHub Actions
- **策略**:
    - **预检 (Pre-check)**: 在 `pull_request` 阶段触发独立的静态测试流，提供快速反馈。
    - **硬门禁 (Hard Gate)**: 在镜像构建 (`docker-build.yml`) 和部署环境 (`cloudflare-pages.yml`) 中嵌入测试步骤，测试失败则终止流水线。
- **配置实现**:
    - Java (Maven): `mvn -B test`
    - Python (uv): `uv run pytest`
    - Angular (Jest): `npm test -- --watchAll=false`

## 涉及范围
- **ms-java-gateway**:
    - 新增 `.github/workflows/test.yml`
    - 修改 `docker-build.yml`：移除 `-DskipTests`
- **ms-java-biz**:
    - 新增 `.github/workflows/test.yml`
    - 修改 `docker-build.yml`：启用 PR 触发并移除 `-DskipTests`
- **ms-py-agent**:
    - 新增 `.github/workflows/test.yml` (基于 `uv` 优化)
- **ms-ng-view**:
    - 新增 `.github/workflows/test.yml`
    - 修改 `cloudflare-pages.yml`：在 Build 前增加测试步骤

## 验证计划
- [x] 手动触发工作流，确认各工程测试环境配置正确（如 JDK 版本、Node 版本、Python UV 环境）。
- [x] 验证 PR 状态检查列表中出现 "Tests" 或 "test (test)" 任务。
- [x] 验证模拟测试失败时，构建/部署任务被成功拦截。

## 变更记录
- 2026-04-26: 全量配置 4 个子工程的测试门禁，废弃所有 `-DskipTests` 的构建脚本。
