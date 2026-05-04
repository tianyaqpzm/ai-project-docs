# Feature: 全链路可观测性增强与标准化错误响应

## 特性描述
为了提升系统的稳定性与排障效率，本项目实现了全链路日志追踪与标准化的异常响应机制。通过在网关层、业务层以及智能体层（Python）统一日志埋点与错误输出格式，确保前端与运维人员能够快速定位全链路中的故障节点。

## 涉及子工程
- **ms-java-gateway**: 负责全链路 Trace-Id 注入、请求/响应双向日志、以及网关层异常保底处理。
- **ms-java-biz**: 负责业务逻辑异常捕获、国际化（i18n）错误消息映射。
- **ms-py-agent**: 负责智能编排层异常捕获，确保异步任务与接口错误的格式一致性。

## 核心能力
### 1. 全链路日志闭环
- **请求进入**: 自动生成 32 位 UUID 作为 `X-Trace-Id` 注入 Header。
- **响应结束**: 记录 `【网关响应完成】` 日志，包含 HTTP 路径、状态码、处理耗时（ms）及 Trace-Id。
- **异常感知**: 在 `GatewayErrorHandler` 中记录 5xx 错误的完整堆栈。

### 2. 标准化错误响应 JSON
所有接口异常（4xx/5xx）必须返回如下格式的 JSON：
```json
{
  "traceId": "0e58051cfdfa44bd8e866680eb338680",
  "status": 500,
  "error_code": "500",
  "error_msg": "内部系统错误，请联系管理员",
  "path": "/rest/kb/v1/documents/ingest",
  "timestamp": "2026-05-01T20:25:49Z"
}
```
- **error_code**: 字符串类型，支持业务自定义编码（如 `DEP_0100`）。
- **error_msg**: 支持国际化，优先返回中/英文业务描述。

## 接口变动
- **API 通用规范**: 所有 REST API 抛出异常时不再返回 Spring 默认的 HTML/WhiteLabel 页面，而是强制返回上述 JSON 结构。

## 验证方法
1. 通过 Postman/Curl 构造非法请求（如错误的参数路径），验证返回 JSON 包含 `error_code` 和 `error_msg`。
2. 切换 `Accept-Language: en-US`，验证 `error_msg` 返回英文。
3. 查看网关日志，确认每条请求均有配套的 `【网关请求】` 和 `【网关响应完成】` 日志。
