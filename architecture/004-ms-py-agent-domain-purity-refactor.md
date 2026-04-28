# 架构决策记录 (ADR) 004: ms-py-agent 领域层纯洁性与工程规范重构

## 状态
已通过 (Accepted)

## 背景
在 `ms-py-agent` 的初始实现中，存在以下架构性缺陷：
1. **领域与基础设施耦合**: 业务模型直接依赖于 SQLAlchemy 的 ORM 定义，导致业务逻辑与数据库结构强绑定。
2. **类型系统缺失**: 核心逻辑缺乏 Type Hints，增加了运行时 `AttributeError` 的风险。
3. **容错机制薄弱**: 存在大量的模糊异常捕获（如 `except: pass`），掩盖了配置转换、网络连接等关键环节的失败。
4. **规范不统一**: 配置项字段缺失，且 MCP 协议调用路径与 Java 后端未完全对齐。

## 决策
为了对齐全工程的“四层整洁架构”标准，决定对 `ms-py-agent` 执行以下架构优化：

### 1. 建立纯净领域层 (Domain Purity)
- 强制引入 `app/domain/models.py`，使用 `@dataclass` 定义业务实体。
- **原则**: 领域模型严禁继承任何 ORM 基类，严禁包含数据库特定的逻辑（如 `__tablename__`）。
- **映射**: 基础设施层负责将 ORM 模型转换为领域对象。

### 2. 全量类型化 (Full Typing)
- 在核心模块（Config, State, Service, Core）实现 100% 的 Type Hints 覆盖。
- 使用 `Optional`, `Dict`, `Any` 等泛型提供精确的契约定义，降低维护成本。

### 3. 异常处理标准化 (Standardized Error Handling)
- 废除所有静默失败代码。
- 引入精准捕获机制（如针对 Nacos 转换的 `ValueError`/`TypeError`，针对网络操作的 `httpx.HTTPError`）。
- 关键链路失败必须记录带有上下文的 `logger.error`。

### 4. 配置与协议标准化
- 在 `Config` 类中显式声明所有配置项（包括 `KB_LLM_*` 等）。
- 统一 MCP 通信路径为 `/mcp/messages`，确保跨语言服务调用的标准一致性。

## 后果
### 正面影响
- **代码健壮性**: 修复了 `retrieval.py` 中的拼写错误，补全了配置缺失，系统启动与运行更加稳定。
- **易读性与维护性**: 强类型系统提供了更好的 IDE 补全和静态检查能力。
- **架构一致性**: `ms-py-agent` 的结构现在与 `ms-java-biz` 和 `ms-ng-view` 的整洁架构标准完全对齐。

### 负面影响
- **代码量略增**: 引入了额外的领域模型定义与类型标注。
- **开发门槛**: 要求开发者严格遵守类型约束与异常处理规范。

## 涉及文件
- [NEW] `app/domain/models.py`
- [MODIFY] `app/core/config.py`, `app/core/dynamic_config.py`, `app/core/nacos.py`, `app/core/lifecycle.py`
- [MODIFY] `app/services/mcp_client.py`, `app/services/kb/retrieval.py`, `app/services/kb/generation.py`
- [MODIFY] `app/agent/state.py`
