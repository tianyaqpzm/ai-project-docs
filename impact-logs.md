# 影响评估汇总

> 记录每次重大修改对其他工程的潜在影响。

---

## 2026-04-25 | SecurityConfig @Value 绑定方式修改

### 修改概述
将 `api-gateway` 中 `SecurityConfig.java` 的白名单 URL 注入方式从 `@Value("${spring.security.ignore.urls}")` 改为构造器注入 `IgnoreWhiteProperties` Bean。

### 触发原因
`@Value` 无法正确绑定 YAML list 到 `String[]`，导致 Gateway 启动崩溃。

### 影响评估

| 子工程 | 影响程度 | 分析 |
|--------|----------|------|
| **api-gateway** | ✅ 直接修复 | Bean 创建逻辑变更，行为不变 |
| **python-agent** | ⚪ 无影响 | 不涉及，是 Gateway 下游消费者 |
| **ai-langchain4j** | ⚪ 无影响 | 不涉及，是 Gateway 下游消费者 |
| **timekeeper (前端)** | ⚪ 无影响 | 不涉及，是 Gateway 上游调用者 |

### 风险等级: 🟢 低
- 修改范围仅限一个配置文件的注入方式
- 运行时行为（白名单匹配逻辑）完全不变
- `IgnoreWhiteProperties` Bean 已在 `JwtAuthenticationFilter` 中稳定使用

### 验证计划
- [ ] 重新部署 Gateway，确认启动成功
- [ ] 验证白名单路径 `/actuator/health`、`/oauth2/**` 等正常放行
- [ ] 验证需要认证的路径仍返回 401/重定向

---

## 2026-04-25 | TDD 全工程测试架构落地

### 修改概述
为 `api-gateway`, `ai-langchain4j`, `python-agent` 三个后端工程建立了共计 52 个测试用例。引入了 H2 内存数据库、禁用 Nacos/Discovery 的测试配置文件夹，并修复了 Gateway 的 YAML 配置冗余。

### 触发原因
系统缺乏核心功能守护，任何修改都可能导致认证失效或协议冲突。

### 影响评估

| 子工程 | 影响程度 | 分析 |
|--------|----------|------|
| **api-gateway** | 🟡 变更 | `SecurityConfig` 开放了 `/api/test/**` 通道（仅 Test 模式），移除了冗余 YAML。测试环境与生产配置进一步隔离。 |
| **ai-langchain4j** | 🟢 优化 | `pom.xml` 引入 H2。测试不再连接物理 PG/Mongo。 |
| **python-agent** | 🟢 优化 | 测试环境不再跳过安全校验（恢复 JWT 真实验证）。 |
| **timekeeper** | ⚪ 无影响 | 目前仅后端变更。 |

### 风险等级: 🟢 低
- 绝大部分修改位于 `src/test` 目录下。
- `application-test.yml` 的配置变更显著降低了对中间件的强依赖风险。

### 验证计划
- [x] 全量回归 52 个测试用例

---

## 2026-04-26 | Timekeeper 国际化 (i18n) 框架落地

### 修改概述
为 `timekeeper` 引入了 `ngx-translate` 国际化框架，建立了中英双语资源文件，并完成了 Chat 页面、设置对话框及全局 UI 的翻译。引入了 `LanguageService` 管理语言选择状态。

### 触发原因
提升用户体验，支持多语言访问。

### 影响评估

| 子工程 | 影响程度 | 分析 |
|--------|----------|------|
| **api-gateway** | ⚪ 无影响 | 透传认证逻辑不变 |
| **python-agent** | ⚪ 无影响 | 不涉及业务协议变更 |
| **ai-langchain4j** | ⚪ 无影响 | 不涉及 RAG 协议变更 |
| **timekeeper** | 🟡 变更 | 引入了 `@ngx-translate` 依赖。`main.ts` 注入了 `APP_INITIALIZER` 导致启动过程包含语言资源加载。 |

### 风险等级: 🟡 中
- 如果 `i18n/*.json` 文件加载失败，`APP_INITIALIZER` 可能导致应用无法启动（已配置 fallback 为 `zh`）。
- 打包体积略有增加（引入了 translate 核心库）。

### 验证计划
- [ ] 验证语言切换按钮功能正常
- [ ] 确认资源文件 (`.json`) 在生产环境下路径解析正确
- [x] 新增 `LanguageService` 单元测试并通过

---

## 2026-04-26 | 全工程 CI/CD 自动化测试门禁落地

### 修改概述
为全系统 4 个子工程配置了 GitHub Actions 测试自动化。在 `api-gateway`, `ai-langchain4j`, `python-agent`, `timekeeper` 中一致性地引入了 PR 测试流，并拦截了构建过程中的 `skipTests` 行为。

### 触发原因
保障 `master` 分支代码质量，防止 regression 导致生产事故。

### 影响评估

| 子工程 | 影响程度 | 分析 |
|--------|----------|------|
| **api-gateway** | 🟡 变更 | 镜像构建耗时增加（增加了测试环节），但安全性提升。 |
| **ai-langchain4j** | 🟡 变更 | 构建耗时增加。开启了 PR 的 Docker Build 预检。 |
| **python-agent** | 🟢 优化 | 引入 `uv` 缓存，测试执行极快。 |
| **timekeeper** | 🟡 变更 | 部署到 Cloudflare Pages 前强制运行 Jest。 |

### 风险等级: 🟢 低
- 即使 CI 报错，本地开发逻辑不受影响。
- 风险点在于若测试用例本身不稳定 (Flaky Tests)，可能阻碍正常部署。

### 验证计划
- [x] 验证 4 个工程的 `.github/workflows/test.yml` 语法正确
- [x] 验证 Java 工程 `mvn package` 不再默认跳过测试
- [x] 验证 `timekeeper` 部署流包含测试步骤
---

## 2026-04-26 | Timekeeper i18n v17 兼容性修复

### 修改概述
修复了 `timekeeper` 中由于 `@ngx-translate` v17 版本不兼容导致的编译及运行时错误。将 i18n 配置方案从 `NgModule` 桥接模式切换到了纯函数式的 `provideTranslateService` 模式。

### 触发原因
`ngx-translate` v17 更改了 `TranslateHttpLoader` 的构造函数签名，不再支持手动通过参数注入 `HttpClient` 和路径信息，且原有 `toPromise()` 在 RxJS 7 中被弃用。

### 影响评估

| 子工程 | 影响程度 | 分析 |
|--------|----------|------|
| **timekeeper** | ✅ 修复 | 解决了编译报错，统一了现代 Angular 的 Provider 配置模式。 |

### 风险等级: 🟢 低
- 修改仅涉及 `i18n.config.ts` 的 Provider 语法，不涉及业务逻辑。
- `lastValueFrom` 替换 `toPromise()` 保障了异步初始化流程的稳定性。

### 验证计划
- [x] 验证 `ChatComponent` 编译报错消失
- [x] 全量运行 `tests/core` 确保 `LanguageService` 及初始化链路正常
- [x] 验证浏览器环境下多语言资源加载成功（基于 CI/CD 预览环境）
