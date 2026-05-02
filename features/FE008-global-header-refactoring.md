# FE008: 全局顶栏组件重构与 SSO 集成

## 特性描述
本特性旨在将原本散落在各个页面的顶栏逻辑进行归并与重构，提取为全局统一的 `MsHeaderComponent`。该组件集成了侧边栏控制、多语言切换、深色模式切换以及基于 Casdoor 的 SSO 账号管理功能。

## 涉及子工程
- **ms-ng-view**: 前端 UI 实现、侧边栏状态管理服务 (`SidebarService`)、国际化资源完善、外部 URL 配置中心化。
- **ms-java-gateway**: 网关安全配置调整，支持 `/logout` 路径的 GET 请求、清除 JWT Cookie 并重定向回前端。

## 业务逻辑
1. **全局状态共享**: 通过 `SidebarService` 统一管理侧边栏的展开/收起状态，支持在顶栏通过按钮控制聊天页面的侧边栏。
2. **多语言切换**: 集成 `LanguageService`，支持中英文实时切换，并自动同步至全局翻译引擎。
3. **SSO 账号管理**: 接入 Casdoor，通过 `/login?redirect_uri=/account` 链路触发 SSO 会话检查，实现免密进入个人中心。
4. **统一登出**: 调用网关 `/logout` 接口，清理网关侧 JWT Cookie 并返回前端落地页。

## 接口与配置变动
### ms-java-gateway
- **白名单调整**: 移除了 `/logout` 在 `ignore.urls` 中的配置，确保其受 Spring Security 保护。
- **SecurityConfig**: 
    - 新增 `.requiresLogout(ServerWebExchangeMatchers.pathMatchers(HttpMethod.GET, "/logout"))`。
    - 配置 `logoutHandler` 清理 `jwt_token` Cookie。
    - 配置 `logoutSuccessHandler` 重定向至前端。

### ms-ng-view
- **environment.ts**: 新增 `VITE_CASDOOR_URL`。
- **URLConfig.ts**: 新增 `EXTERNAL.CASDOOR_ACCOUNT`。
- **MsHeaderComponent**: 全新封装的 Standalone 组件，支持通过 `blackList` 属性控制在特定页面（如落地页）隐藏。

## 使用指南 (Usage Guide)

### 1. 全局 Header 的引入与控制
- **引入**: 组件已在 `AppComponent` 中全局注入。
- **控制**: 通过 `blackList` 属性传入路由数组，可控制在特定页面（如 `/landing`）隐藏顶栏。
  ```html
  <ms-header [blackList]="['/landing']"></ms-header>
  ```

### 2. 子页面布局适配建议
为确保页面能够适配全局顶栏而不产生裁切，子页面（尤其是聊天等全屏应用）应遵循以下规范：
- **宿主高度**: 在组件装饰器中设置 `:host` 为块级且高度 100%。
  ```typescript
  host: { class: 'block h-full w-full' }
  ```
- **根容器**: 使用 `h-full` 而非 `h-screen`，避免高度计算包含顶栏导致底部内容溢出。
- **内边距**: 核心操作区（如输入框）底部建议预留 `pb-6` 或 `pb-8`，以防在移动端或窄屏下被遮挡。

### 3. AI 免责声明规范
- **文案**: 统一使用国际化 Key `CHAT.DISCLAIMER`。
- **布局**: 建议放置在输入区域的最底部，独立于输入框容器，以确保即使输入框内容增多，提示文字依然锚定在视图底部。

## 影响评估
- **前端布局**: 所有页面（除黑名单外）现在统一使用该组件，移除了页面级重复的 Header 实现。
- **登录状态**: 账号中心跳转现在依赖 Casdoor SSO，需确保 Casdoor 域名下的 Cookie 策略正常。
- **路由交互**: 侧边栏按钮仅在 `/chat` 相关路由下显示，避免在其他页面引起逻辑混淆。

## 变更记录
- **2026-05-01**: 完成布局 Flex 化重构，解决免责声明不显示的问题，修复组件宿主高度导致的溢出 Bug。
