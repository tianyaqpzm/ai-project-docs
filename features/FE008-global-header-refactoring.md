# FE008: 全局顶栏组件重构与 SSO 集成

## 特性描述
本特性旨在将原本散落在各个页面的顶栏逻辑进行归并与重构，提取为全局统一的 `MsHeaderComponent`。该组件集成了侧边栏控制、多语言切换、深色模式切换以及基于 Casdoor 的 SSO 账号管理功能。

## 涉及子工程
- **ms-ng-view**: 前端 UI 实现、侧边栏状态管理服务 (`SidebarService`)、国际化资源完善、外部 URL 配置中心化。
- **ms-java-gateway**: 网关安全配置调整，支持 `/logout` 路径的 GET 请求、清除 JWT Cookie 并重定向回前端。

## 业务逻辑
1. **全局状态共享与交互逻辑重构**: 
    - 通过 `SidebarService` 统一管理侧边栏状态。
    - **重大变更**: 将侧边栏的展开/收起控制按钮从全局 `ms-header` 移除，移动至各业务页面（Chat, Knowledge, Prompt）侧边栏的右侧边缘。
    - 采用“悬浮手柄”设计，确保侧边栏在收起状态下依然可通过手柄快速呼出。
2. **多功能应用导航菜单**:
    - 在顶栏左侧新增“应用中心”图标（apps），点击后展示 3 列网格布局的导航菜单。
    - 预设功能模块：智能对话、知识管理、提示词管理、控制台、活动列表、界面定制。
    - 支持点击跳转、平滑过渡动画及国际化标签。
3. **多语言切换**: 集成 `LanguageService`，支持中英文实时切换，并自动同步至全局翻译引擎。
4. **SSO 账号管理**: 接入 Casdoor，实现免密进入个人中心。
5. **统一登出**: 调用网关 `/logout` 接口，清理 JWT Cookie。

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
- **MsHeaderComponent**: 
    - 封装了 `navItems` 静态数据，定义了各个模块的图标、背景色、路径及翻译 Key。
    - 移除了 `showSidebarToggle` 逻辑，转而通过各页面局部控制。
- **侧边栏布局适配**: `Chat`, `Knowledge`, `Prompt` 页面统一采用 `md:relative` + `overflow-visible` 布局，支持侧边栏右侧控制手柄的溢出显示。

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
- **前端布局**: 顶栏更加纯粹，不再干预具体业务页面的侧边栏开合，降低了组件间的耦合。
- **交互体验**: 用户在操作侧边栏时，焦点始终保持在侧边栏区域，交互链路更短、更符合直觉。
- **扩展性**: `navItems` 采用声明式配置，未来新增业务模块只需在 `MsHeaderComponent` 中追加配置项即可。

## 变更记录
- **2026-05-01**: 完成布局 Flex 化重构，解决免责声明不显示的问题，修复组件宿主高度导致的溢出 Bug。
- **2026-05-07**: 
    - 新增全局“应用中心”导航菜单。
    - 重构侧边栏控制逻辑，将 Toggle 按钮从 Header 移至侧边栏边缘手柄（Handle）。
    - 优化菜单打开体验，消除 Hover 时的缩放抖动（“滑动感”）。
    - 统一 Prompt 页面的侧边栏切换行为与 Chat 页面一致。
