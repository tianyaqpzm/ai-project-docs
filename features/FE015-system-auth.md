# 特性定义：跨服务系统鉴权 (System-to-System Authentication)

> **状态**: ⏳ 待办 (Planned)

## 1. 背景与目标
当前系统已集成 Casdoor 处理用户鉴权。用户登录后，网关会签发一个内部 JWT 并通过 Cookie/Header 传递。
但在系统直接调用场景（如 `ms-py-agent` 调用 `ms-java-biz` 的 MCP 接口）下，如果没有用户触发，`ms-py-agent` 无法获取有效的令牌。

本特性旨在为微服务提供一套**系统级令牌 (System Token)** 机制，确保服务间调用的安全性与可追溯性。

## 2. 核心架构设计

### 2.1 方案选择：OAuth2 Client Credentials Grant
由于系统已对接 Casdoor，采用标准的 OAuth2 客户端模式（M2M）最为规范。

- **颁发者**: Casdoor
- **使用者**: `ms-py-agent` (或其他下游服务)
- **验证者**: `ms-java-biz` (业务后端)

### 2.2 逻辑流程
1. **配置阶段**: 在 Casdoor 为 `ms-py-agent` 创建专属 Client，获取 `clientId` 和 `clientSecret`。
2. **获取令牌**: `ms-py-agent` 通过 `client_id` 和 `client_secret` 向 Casdoor 申请 Token。
3. **请求携带**: `ms-py-agent` 调用 `ms-java-biz` 时，在 Header 中携带 `Authorization: Bearer <System_Token>`。
4. **验证鉴权**: `ms-java-biz` 识别出这是 Casdoor 签发的令牌，通过 Casdoor 公钥或内审接口进行验证，并赋予 `ROLE_SYSTEM` 权限。

## 3. 详细设计

### 3.1 ms-py-agent (Python 端)
- **配置项**: 增加 `CASDOOR_CLIENT_ID`, `CASDOOR_CLIENT_SECRET`, `CASDOOR_TOKEN_URL`。
- **TokenManager**: 实现一个单例类，负责维护系统 Token 的生命周期（定时刷新）。
- **拦截器/客户端**: 在发起 HTTP 请求时，自动注入系统 Token。

### 3.2 ms-java-biz (Java 端)
- **JWT 兼容性升级**: `JwtAuthenticationFilter` 需要同时支持：
    - **内部 HS256 令牌**: 验证 `jwtSecret` (用于用户转发)。
    - **Casdoor RS256 令牌**: 验证 Casdoor 公钥 (用于系统调用)。
- **权限控制**: 系统令牌解析后的身份应具有 `SYSTEM` 标识，以便在某些敏感接口上做权限隔离。

## 4. 实施计划 (Implementation Plan)

### 第一阶段：基础设施配置 (Casdoor)
1. 在 Casdoor 控制台创建一个新 Application: `ms-py-agent-internal`。
2. 设置 Grant Types 包含 `client_credentials`。
3. 记录 `Client ID` 和 `Client Secret`。

### 第二阶段：Python Agent 改造
1. 实现 `SystemTokenProvider`，封装获取 Token 的逻辑。
2. 修改 MCP 调用逻辑，当没有用户 Token 时， fallback 到系统 Token。

### 第三阶段：Java 业务端兼容
1. 修改 `JwtAuthenticationFilter`，引入 `JwCk` 或直接通过 Casdoor SDK 验证第三方 Token。
2. 更新 `SecurityConfig` 白名单或权限规则。

## 5. 影响评估
- **安全性**: 解决了内部接口裸奔的问题。
- **性能**: Python 端需要缓存 Token，避免频繁请求 Casdoor。
- **维护性**: 需要维护 Casdoor 中的 Client 信息。
