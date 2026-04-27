# 问题复盘：2026-04-28 - Nacos 连接因 DNS 解析延迟而失败

## 问题描述
Python Agent 在启动过程中连接 Nacos 报错：
`[do-sync-req] tao-lan.122577.xyz:18848 connection error:[Errno -3] Temporary failure in name resolution`
随后导致：
`[get-access-token] exception All server are not available occur`

虽然 Nacos 服务本身正常，但 Python SDK 的默认超时时间（3s）可能过短，且缺乏针对 DNS 临时故障或网络抖动（如 Tailscale 域名解析延迟）的重试机制。

## 根因分析
1. **DNS 解析延迟**：使用 Tailscale 域名（`tao-lan.122577.xyz`）时，DNS 解析可能存在瞬间延迟或环境兼容性问题，导致 `socket.getaddrinfo` 返回 `Errno -3`。
2. **SDK 超时过短**：`nacos-sdk-python` 默认的 `DEFAULTS["TIMEOUT"]` 仅为 3 秒。在涉及 Auth 登录请求时，包含 DNS 解析 + 握手 + 业务处理的总时长容易超过 3 秒。
3. **缺乏重试机制**：原始 `nacos_manager.connect()` 只尝试一次，一旦 DNS 解析失败就直接回退到本地环境，导致服务注册失败。

## 解决方案
1. **动态调整 SDK 超时**：通过 Monkeypatch 方式将 `nacos.client.DEFAULTS["TIMEOUT"]` 提升至 10s（可通过 `NACOS_TIMEOUT` 环境变量配置）。
2. **引入退避重试逻辑**：
   - 在 `NacosManager.connect()` 中增加重试循环（默认 5 次）。
   - 使用指数退避策略（Exponential Backoff），在 DNS 临时不可用时等待解析就绪。
3. **配置化管理**：
   - 在 `Config` 类中新增 `NACOS_TIMEOUT` 和 `NACOS_RETRIES` 配置项。

## 验证结果
- 手动测试显示，即使在模拟网络波动的情况下，重试机制也能让 SDK 在第二次或第三次尝试时成功建立连接（只要 DNS 解析最终恢复）。
- 解决了由于 "Temporary failure in name resolution" 导致的启动即失败问题。

## 经验教训
- **基础设施依赖的健壮性**：对于 Nacos 这种核心基础设施，连接逻辑必须具备重试能力，不能假设网络环境百分之百可靠。
- **SDK 默认值审计**：使用第三方 SDK 时，应注意其默认超时设置，特别是当服务部署在跨网络（如 VPN/Tailscale）环境下时。
