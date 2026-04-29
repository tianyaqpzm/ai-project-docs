# 特性定义：ms-py-agent 领域模型重构与工程健壮性提升

## 修改背景
为了提升 `ms-py-agent` 工程的领域模型纯洁度、类型安全性和整体工程质量，根据 [CODING_STANDARDS.md](file:///Users/pei/projects/ms-py-agent/.agent/rules/CODING_STANDARDS.md) 规范，对该工程进行了深度重构。主要解决领域对象与 ORM 耦合、缺乏类型约束、异常处理过于模糊以及隐藏的运行时 Bug 等问题。

## 核心变更
### 1. 领域层 (Domain Layer) 落地
- **[NEW]** 创建了 `app/domain/models.py`，引入纯粹的 `@dataclass` 领域对象：
    - `ChatMessage`: 消息实体。
    - `ServiceInstance`: Nacos 服务实例。
- 实现了领域对象与 ORM 模型 (`app/core/database.py`) 的彻底解耦。

### 2. 全局类型约束 (Type Hints)
- **100% 覆盖**: 对核心模块进行了全量类型标注。
    - `app/core/config.py`: 对 `Config` 类成员补全类型，并引入 `Optional` 支持。
    - `app/agent/state.py`: 细化 `AgentState` 内部字段类型。
    - `app/core/nacos.py`: 补全方法参数与返回值的类型提示。
    - `app/services/mcp_client.py`: 对所有 MCP 客户端基类与实现类进行强类型约束。

### 3. 异常处理精准化
- 清理了所有模糊的 `except: pass` 或 `except Exception:` 块。
- 在 `dynamic_config.py` 中精准捕获 `ValueError` 和 `TypeError`。
- 在 `nacos.py` 中将 `_get_local_ip` 的异常捕获范围收窄至 `OSError`。
- 增强了 `lifecycle.py` 的日志输出，确保资源释放失败时有明确记录。

### 4. 关键 Bug 修复
- **VectorStore Typo**: 修复了 `app/services/kb/retrieval.py` 中 `self.vectorstore` 拼写错误。
- **Missing Config**: 在 `Config` 类中补全了 `KB_LLM_PROVIDER` 和 `KB_LLM_MODEL` 缺失字段。
- **URL Standard**: 将 MCP 调用路径统一为 `/mcp/messages`，确保与 Java 端控制器对齐。

## 涉及子工程
- **ms-py-agent**: 核心逻辑重构。

## 接口变动
- 无外部 API 变动，仅限内部实现与类型定义优化。

## 验证结论
- **Architecture Tests**: 通过 `tests/test_architecture.py` (4 passed)。
- **Unit Tests**: 通过 `tests/test_mcp_client_unit.py` (4 passed)。
- 验证工具：使用 `uv run pytest` 确保了依赖环境的正确性。
