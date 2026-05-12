# 问题复盘：跨域 (CORS) 与私有网络访问 (PNA) 拦截

## 1. 问题描述
**日期**：2026-05-05
**现象**：前端 `https://122577.xyz`（Cloudflare 托管，公网环境）在调用网关接口 `https://tao-lan.122577.xyz:8443/rest/dark/v1/agent/chat`（解析为 Tailscale 节点，私网环境）时失败。
**错误信息**：`Access to fetch at '...' from origin '...' has been blocked by CORS policy: Permission was denied for this request to access the local address space.`
**网络表现**：Network 面板显示请求状态为 `net::ERR_FAILED 200 (OK)`，但控制台抛出异常，JS 无法读取响应。

## 2. 根因分析 (RCA)
1. **私有网络访问 (PNA) 限制**：Chrome 浏览器为了防止公网页面恶意攻击局域网设备，实施了 PNA 安全策略。当“公网安全上下文”（HTTPS 公网域名）尝试访问“私有网络地址”（如私有 IP、.local 域名或 Tailscale 的 100.x.x.x IP）时，浏览器会执行严格检查。
2. **Chrome 130+ 策略升级**：即使后端响应了 `Access-Control-Allow-Private-Network: true`，对于 Fetch/XHR 类型的子资源请求，Chrome 仍会因为缺乏用户显式授权或请求未声明 `targetAddressSpace` 而拦截。
3. **环境矛盾**：前端托管在 Cloudflare (Public)，后端隐藏在 Tailscale (Private)，两者处于不同的网络空间级别，触发了浏览器的跨网络层级安全拦截。

## 3. 尝试过的方案 (失败记录)

### 方案 A：后端网关级 PNA 响应头支持
- **操作**：在 `ms-java-gateway` 的 `CorsConfiguration` 中设置 `setAllowPrivateNetwork(true)`。
- **结果**：**不足以解决问题**。在最新版 Chrome 中，仅靠该响应头无法直接放行公网到私网的子资源请求。

### 方案 B：CSP `treat-as-public-address` 尝试
- **操作**：在后端网关 Response Header 或前端 `index.html` 的 `<meta>` 标签中注入 `Content-Security-Policy: treat-as-public-address`。
- **结果**：**失败并引发次生灾害**。该指令主要针对顶级 Document 生效。应用于 API 接口时不仅无法绕过 PNA，反而因 CSP 策略冲突导致页面其他正常资源（如 `topics` 接口）被大面积阻断。

## 4. 解决方案汇总

### 方案一：浏览器用户级手动授权（本地开发推荐）
- **操作**：点击 Chrome 地址栏左侧“网站设置”图标 -> **不安全的私有网络请求 (Insecure private network requests)** -> 设置为 **允许 (Allow)**。
- **优点**：即时生效，不需要修改任何代码。

### 方案二：同源反向代理（传统生产方案）
- **思路**：将后端 API 代理到前端同域名路径下（如 `https://122577.xyz/api/...`）。
- **局限**：因前端托管在 Cloudflare Pages (Serverless)，无法直接部署 Nginx 进行 `proxy_pass`，且 Cloudflare 节点无法直接触达 Tailscale 内网。

### 最终方案：Cloudflare Tunnel 内网穿透（混合云生产推荐）
- **思路**：将后端私网环境转换为公网环境，变 `Public -> Private` 为 `Public -> Public`。
- **操作**：
    1. 在后端服务器运行 `cloudflared` 建立加密隧道。
    2. 将内网网关映射到公网域名（例如 `https://api.122577.xyz`）。
    3. 修改前端环境变量，将 API 地址指向该公网域名。
- **部署关键坑点 (Troubleshooting)**：
    - **Docker 容器隔离**：若 `cloudflared` 运行在 Docker 中，`localhost` 指向容器自身。必须使用 Docker 网络中的服务名（如 `https://nginx:8443`）或 `host.docker.internal`。
    - **TLS 证书校验失败**：若后端使用自签证书或域名不匹配（如证书是给 `tao-lan` 的，但容器名叫 `nginx`），`cloudflared` 会报 502。必须在 Cloudflare 后台开启 **No TLS Verify**。
- **优点**：彻底规避 PNA 拦截，且可通过 Cloudflare Zero Trust (Access) 提供身份认证保护，安全性优于直接暴露公网。

## 5. 经验总结
- **网络层级意识**：在涉及 Tailscale、VPN 或内网穿透的架构中，必须警惕浏览器的 PNA 拦截机制。
- **CSP 审慎原则**：CSP 策略影响全局，在不完全确定指令作用域（如 Document vs Subresource）时，严禁在生产环境尝试。
- **架构对齐**：前后端应尽量处于同一网络空间，或通过隧道技术将链路对齐，避免跨网络层级的复杂调用。



*(注：曾尝试使用 `Content-Security-Policy: treat-as-public-address` 方案，但在最新版 Chrome 130+ 中，该指令仅对顶级 Document 生效，对于 `fetch`/`XHR` 类型的 API 请求不但无效，反而会因为 CSP 策略异常导致原本正常的其他请求被大面积阻断。因此该方案已被废弃。)*
