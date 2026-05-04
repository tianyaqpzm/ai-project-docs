# 架构决策 (ADR)：LangChain 工具定义选型

## 状态
已决定 (Decided) - 2026-05-03

## 背景
在集成 Model Context Protocol (MCP) 时，需要根据远程服务（如 `ms-java-biz`）返回的 JSON 定义动态生成 LangChain 工具。最初尝试使用 `@tool` 装饰器，但在处理动态名称和特定 LangChain 版本兼容性时遇到障碍。

## 决策：使用 StructuredTool 替代 @tool 处理动态工具

### 1. 核心理由
*   **动态性需求**：MCP 工具的名称和描述在运行时由 Nacos 发现的服务决定，`StructuredTool.from_function` 允许编程式地设置这些元数据，而 `@tool` 装饰器更倾向于静态定义。
*   **避免冲突**：`StructuredTool` 实例是独立的，即使它们共享同一个底层 `wrapper` 执行函数，也可以通过 `name` 属性在 LLM 绑定中进行区分。
*   **版本兼容性**：绕过了某些 LangChain 版本中 `@tool` 装饰器不支持 `name` 关键字参数的 Bug。

### 2. 实施规范
*   对于 **MCP 远程工具**：必须使用 `StructuredTool.from_function`。
*   对于 **静态本地工具**：如果不需要动态修改元数据，允许保留 `@tool` 以简化代码。
*   **异步支持**：必须使用 `coroutine` 参数传入 `async def` 函数，确保符合系统的非阻塞架构要求。

## 影响
*   **代码健壮性**：减少了因装饰器魔法导致的运行时错误。
*   **扩展性**：未来如果引入更多的 MCP 服务，可以统一通过工厂方法批量生产工具实例。
*   **维护性**：工具定义变得显式且易于测试。
