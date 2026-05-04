# 问题复盘：知识库注入/检索 500 错误 (Embedding API)

## 问题描述
**日期**：2026-05-02
**现象**：在 `ms-py-agent` 执行知识库向量化注入（Indexing）或检索（Retrieval）时，频繁触发 `500 Internal Server Error`。
**错误信息**：`Error code: 500 - {'error': {'message': 'input is empty', 'type': 'new_api_error', 'code': 'convert_request_failed'}}`。

## 根因分析 (RCA)
1. **参数不兼容**：`langchain-openai` 底层调用 OpenAI SDK 时，会默认在请求体中强制添加 `"encoding_format": "float"` 字段。
2. **代理服务器限制**：项目使用的 `new-api` 代理服务器（或其后端模型网关）对请求体校验较为严格。当识别到非预期的 `encoding_format` 字段时，其解析逻辑出现异常，导致无法正确读取 `input` 字段，从而误报 `input is empty`。
3. **SSL/证书干扰**：由于开发环境使用自签名证书，标准的 LangChain Embedding 类在异步初始化时，若未显式注入跳过验证的 `httpx.AsyncClient`，容易产生连接错误或被包装成 500 错误。

## 解决方案
**最终方案：手动绕过 (Direct Implementation)**
1. **去框架化调用**：在 `indexing.py` 和 `retrieval.py` 中，不再直接调用 `vector_store.aadd_documents`（该方法会触发框架内部的 Embedding 请求），改为使用 `httpx` 手动构造“纯净”的请求体（仅含 `model` 和 `input`）。
2. **直接向量注入**：通过向量数据库的底层接口 `aadd_embeddings` 注入手动获取的向量，从而绕过所有中间件的参数污染。
3. **强制 SSL 信任**：在手动请求中显式设置 `verify=False`。

## 经验总结
- **代理兼容性陷阱**：在使用非官方 OpenAI 代理（如 New-API, One-API）时，标准 SDK 自动添加的“优化参数”（如 `encoding_format`, `user`）是最大的 500 错误源。
- **调试策略**：当标准库（LangChain/OpenAI SDK）持续报错且参数无法透传修改时，应果断通过 `httpx` 进行原生 API 测试。如果原生测试通过，说明是框架封装问题。
- **架构权衡**：在高度不确定的网络/代理环境下，服务层直接接管核心 API 调用（如 Embedding）比强行适配框架参数更具鲁棒性。
