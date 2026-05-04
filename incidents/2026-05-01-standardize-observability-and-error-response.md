# RCA: 全链路可观测性增强与错误响应格式标准化

## 现象描述
2026-05-01，在排查知识库入库流程（`/ingest`）异常时，发现 `ms-java-gateway` 仅记录了请求进入的 Trace 日志，但在后端服务抛出异常或连接失败时，网关侧没有任何后续日志记录（既没有响应状态码，也没有异常堆栈），导致难以判定问题发生在网关转发阶段还是后端业务逻辑阶段。

## 根因分析
1. **日志范围不全**: `TraceIdFilter` 原本仅在请求进入时打印日志，未订阅响应结束事件，因此无法记录 HTTP 状态码和处理耗时。
2. **异常记录粒度不足**: `GatewayErrorHandler` 在捕获网关自身异常时，仅记录了 `error.getMessage()`。
3. **透明转发机制**: 对于后端业务服务返回的 500 错误，网关作为代理默认直接透传，且原有的错误响应格式不符合 `error_code`/`error_msg` 的业务要求。

## 解决方案
1. **全链路标准化**: 在所有子工程服务（`ms-java-biz`, `ms-py-agent`）中实现统一的全局异常处理器。
    - **ms-java-biz**: 更新 `ErrorResponse` DTO 和 `GlobalExceptionHandler`，将 `error_code` 定位为 `String` 类型（如 `TMP_0100`），并通过 `MessageSource` 支持 `error_msg` 的国际化（中英文）。
    - **ms-py-agent**: 引入 `global_exception_handler`，确保 Python 侧的异常响应也包含 `String` 类型的 `error_code`。
2. **网关透传与自愈日志**: 
    - **日志记录**: 增强 `TraceIdFilter` 的响应日志记录，确保网关能监控到后端的异常状态码。
    - **网关自修**: `GatewayErrorHandler` 仅在网关自身异常（如 504 响应超时）时生效，并对齐全链路错误格式，提供异常堆栈以供排查。
3. **职责分离**: 明确“网关仅透传，后端报详情”的原则。后端服务必须返回详尽的业务错误码（`error_code`）和描述（`error_msg`）。
4. **规范对齐**: 将上述日志要求固化到 `ms-java-gateway` 的 `ai-code-ws.md` 编码规范中。

## 预防措施
- 核心 Filter 必须包含请求与响应的双向日志闭环。
- 基础设施层的错误处理器严禁“吞掉”堆栈信息。
