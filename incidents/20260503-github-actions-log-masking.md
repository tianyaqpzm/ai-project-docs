# 故障复盘：GitHub Actions 日志掩码导致的认证误判

**背景**: 在 `ms-java-biz` 项目中引入二方库 `ms-java-yaml2code-sdk` 时，需要从 GitHub Packages 私有仓库拉取依赖。

**日期**: 2026-05-03
**问题描述**: 在 `ms-java-biz` 的 CI/CD 构建过程中，日志显示用户名和关键配置被替换为 `***`。
具体报错如下：
```text
Error: Failed to read artifact descriptor for com.dark:ms-java-yaml2code-sdk:jar:0.0.1-SNAPSHOT
Error: Caused by: Could not transfer artifact com.dark:ms-java-yaml2code-sdk:pom:0.0.1-SNAPSHOT from/to github (https://maven.pkg.github.com/***/ms-java-yaml2code-sdk): status code: 401, reason phrase: Unauthorized
```
这导致开发者误认为变量未注入或配置为空，从而反复调整 `setup-java` 逻辑，实际认证失败（401）的根因被视觉干扰掩盖。

## 根因分析 (RCA)
1. **日志掩码机制 (Log Masking)**: 
   - GitHub Actions 的日志清洗器是全局匹配的。如果 Secret 值（如 `tianyaqpzm`）出现在任何日志中（包括 Maven 请求的 URL），都会被替换为 `***`，导致开发者产生“变量丢失”的视觉误判。
2. **`setup-java` 动作机制**:
   - `actions/setup-java@v4` 的 `server-username` 和 `server-password` 配置项接收的**不是真实值**，而是**环境变量的名称**。
   - 动作会在生成的 `settings.xml` 中填入 `${env.VARIABLE_NAME}`。如果后续 `mvn` 步骤没有通过 `env:` 注入这些变量，Maven 将无法获取凭证，从而触发 401。

## 解决方案与最佳实践
- **变量映射**: 在 `setup-java` 中定义变量名（如 `MAVEN_PASSWORD`），并在 `mvn` 步骤中显式注入真实 Secret 值。
- **验证方式**: 调试 CI 流程时，以长度检查（`echo ${#VAR}`）代替内容检查，避开脱敏干扰。
- **显式配置**: 如果追求极致透明和可控，建议采用 Shell 手动生成 `~/.m2/settings.xml`，不使用 `setup-java` 的内建配置功能。
 
 ## 延伸经验：组织迁移下的 CI 镜像命名规范
 在将个人仓库迁移至组织（如 `4fork`）后，GitHub Actions 的环境变量会发生变化，需注意以下镜像命名冲突：
 
 1. **Docker Hub 命名空间**: 
    - Docker Hub 的命名空间（Namespace）通常与docker hub的个人用户名（如 `tianyaqpzm`）绑定。
    - 迁移至 GitHub 组织后，Docker Hub 并不会自动同步。如果继续推送到 Docker Hub，`metadata-action` 中的 `images` 字段仍需硬编码为docker hub的个人用户名（如 `tianyaqpzm/ms-java-biz`），不能误用组织名。
 
 2. **GHCR 与 `github.repository_owner`**:
    - `ghcr.io` 镜像通常建议使用 `ghcr.io/${{ github.repository_owner }}/ms-java-biz`。
    - 迁移后，`${{ github.repository_owner }}` 会自动变为组织名（如 `4fork`）。
    - **避坑点**：如果组织名与 Docker Hub 用户名不一致，会导致两个镜像仓库的路径结构不统一。在 `metadata-action` 中应显式列出所有镜像路径，避免使用单一变量覆盖所有 Registry。
