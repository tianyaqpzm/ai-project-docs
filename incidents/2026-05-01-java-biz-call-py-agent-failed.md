# 故障复盘：ms-java-biz 调用 ms-py-agent 失败 (Connection Refused & 401)

## 1. 故障描述
**时间**: 2026-05-01
**现象**: 
- 前端触发“知识入库”流程时，`ms-java-biz` 报错 `Connection refused`，无法连接到 `ms-py-agent`。
- 修复连接问题后，紧接着出现 `401 Unauthorized` 报错，身份校验失败。

## 2. 根因分析 (RCA)

### A. Connection Refused (连接被拒绝)
- **直接原因**: `ms-py-agent` 在 Uvicorn 启动时默认绑定 `127.0.0.1`。
- **根本原因**: 
    - Nacos 注册中心自动检测并注册的是主机的局域网 IP (`192.168.3.56`)。
    - `ms-java-biz` 通过 Nacos 发现服务地址后，尝试跨网卡（从局域网 IP）访问 Python 服务。
    - 由于 Python 服务仅监听回环地址，内核直接拒绝了来自非 localhost 接口的连接。
- **经验教训**: 在容器化或多服务发现环境中，服务端必须绑定 `0.0.0.0` 才能被外部网卡或其它容器访问。

### B. 401 Unauthorized (身份校验失败)
- **直接原因**: `ms-java-biz` 发起的下游请求中缺失 `Authorization` 请求头。
- **根本原因**: 
    - 开发者使用了原生的 `RestTemplate` 且未配置任何拦截器。
    - 在微服务架构中，身份信息（JWT）不会自动从入口服务透传到下游服务，必须显式实现 Token Propagator（令牌传播器）。
- **经验教训**: 涉及跨服务的安全调用时，必须统一配置拦截器（Interceptor/Filter）来实现 Context 传播。

## 3. 解决过程
1.  **修复绑定**: 修改 `ms-py-agent` 的 `config.py`，将 `HOST` 默认值设为 `0.0.0.0`。
2.  **实现透传**:
    - 在 `ms-java-biz` 的 `JwtAuthenticationFilter` 中保留原始 Token。
    - 实现 `JwtTokenInterceptor` 并注册到全局 `RestTemplate` 中，自动注入 `Bearer Auth`。

## 4. 预防措施
1.  **工程模板更新**: 更新 Python 服务的基础配置模板，默认 `HOST` 统一设为 `0.0.0.0`。
2.  **基础组件增强**: 在 `ms-java-biz` 基础库中增加通用 Header 传播拦截器，防止未来新增下游服务时再次出现类似漏传问题。
3.  **可观测性**: 在 `ms-java-biz` 的远程调用失败异常中，增加对请求头状态的诊断日志，以便快速定位是否缺失凭证。
