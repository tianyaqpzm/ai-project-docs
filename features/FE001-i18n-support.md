# 特性定义：国际化 (i18n) 支持

## 业务背景
为了支持全球化运营，ms-ng-view 需要提供多语言界面。首期目标是支持中文（简体）和英文，并允许用户在界面上实时切换语言，且该偏好需在本地持久化。

## 技术方案
- **核心库**: `@ngx-translate/core` & `@ngx-translate/http-loader`
- **语言文件**: 存储于 `public/i18n/*.json` (e.g., `zh.json`, `en.json`)
- **状态管理**: `LanguageService` (基于 Angular Signals)
- **持久化**: `localStorage.getItem('lang')`
- **初始化逻辑**: `APP_INITIALIZER` 确保在应用启动前加载语言资源。

## 涉及范围
- `AppComponent`: 全局顶栏增加语言切换按钮。
- `ChatComponent`: 聊天界面所有硬编码字符串（欢迎词、工具栏、侧边栏等）全局替换为 i18n key。
- `SettingsDialog`: 偏好设置对话框完全国际化。

## 验证计划

### 1. 单元测试用例 (Unit Tests)
测试对象：`LanguageService`
代码位置：`tests/core/services/language.service.spec.ts`

| 用例 ID | 测试目标 | 预期结果 |
|---------|----------|----------|
| **TL-01** | 初始加载 | 如果 `localStorage` 为空，默认语言应为 `zh`。 |
| **TL-02** | 语言切换 | 调用 `toggleLanguage()` 时，语言应在 `zh` 和 `en` 之间切换，并更新 `localStorage`。 |
| **TL-03** | 手动设置 | 调用 `setLanguage('en')` 应正确更新信号状态和本地存储。 |

### 2. E2E 验证用例 (Manual/Automated Verification)
验证环境：`http://localhost:3000`

#### 2.1 界面语言实时切换
- **操作**: 
    1. 进入 Chat 页面。
    2. 点击右上角语言切换按钮（当前显示 `EN`）。
- **预期结果**:
    1. 按钮文字变为 `中`。
    2. 侧边栏 "Recent" 变为 "最近记录"。
    3. 欢迎词 "Hello, Traveler" 变为 "你好，旅行者"。
    4. 输入框占位符 "Ask anything..." 变为 "问我任何问题..."。

#### 2.2 刷新页面持久化
- **操作**: 
    1. 切换语言为 English (按钮显示 `中`)。
    2. 刷新浏览器页面。
- **预期结果**:
    1. 页面重新加载后，界面依然保持为英文。
    2. 按钮依然显示 `中`。

#### 2.3 设置对话框国际化
- **操作**: 
    1. 切换语言为中文。
    2. 点击头像菜单 -> "个人偏好设置"。
- **预期结果**:
    1. 对话框标题显示 "个人偏好设置"。
    2. 选项显示 "深色模式"、"回车键发送消息" 等。
    3. 切换为英文后，对话框对应内容自动更新为 English。

#### 2.4 极端情况与容错
- **操作**: 
    1. 清除浏览器的 `LocalStorage` 数据。
    2. 模拟网络异常导致 `en.json` 加载失败。
- **预期结果**:
    1. 应用能够正常启动，回退到默认语言 `zh`。
    2. 界面显示翻译 Key 或 fallback 文本，而非应用崩溃。

## 变更记录
- 2026-04-26: 初始化国际化框架，完成 Chat 页及设置页翻译逻辑。
- 2026-04-26: 新增 i18n 验证用例并合并文档，删除冗余的 `i18n-verification-cases.md`。
