# 问题复盘 (Incident Postmortem) - 统一知识库路由前缀并修复 404 异常

## 1. 问题描述 (Incident Description)
在前端调用知识库构建接口以增量同步和重构菜谱向量时，由于端点路由混乱导致 `404 Not Found` 异常：
```
POST /rest/dark/v1/agent/knowledge/build -> 404 Not Found
```

## 2. 根因分析 (Root Cause Analysis - RCA)
1. **老旧路由设计遗留 (Legacy API Inconsistency)**：
   - 知识库模块中，有些接口（如 `/documents/ingest`）属于 Java 后端至 Python 服务的内部通信，使用的是统一的知识库前缀 `/rest/kb/v1/`。
   - 而增量构建接口被分配了非统一的外部前缀 `/rest/dark/v1/agent/knowledge/build`。
   - 当在 [kb.py](file:///Users/pei/projects/ms-py-agent/app/api/routers/kb.py) 内部声明 `@router.post("/rest/dark/v1/agent/knowledge/build", ...)` 时，由于 `kb.router` 已经在入口挂载了 `prefix="/rest/kb/v1"` 统一命名空间前缀，导致最终物理路径变成了污染的 `/rest/kb/v1/rest/dark/v1/agent/knowledge/build`。

2. **API 网关转发映射缺失 (Missing Gateway Routing)**：
   - 原先的 API 网关 ([ms-java-gateway](file:///Users/pei/projects/ms-java-gateway/src/main/resources/application.yml)) 中仅代理转发了 `/rest/dark/v1/agent/**` 路径到 Python 服务，却并没有配置 `/rest/kb/v1/**` 到 `ms-py-agent` 的转发规则（因为原先 Java 和 Python 对该域接口为直连模式）。

---

## 3. 解决方案与实施 (Resolution & Action Items)

### 3.1 统一路由前缀归属 (`ms-py-agent` 规范化)
- 放弃临时多路由的补丁设计，彻底将构建接口划归到统一知识库命名空间下。
- 将增量构建接口的端点装饰器修正为挂载在标准 `router` 上，将路径设为相对路径 `/agent/knowledge/build`：
  ```python
  @router.post("/agent/knowledge/build", summary="触发菜谱知识库增量向量化构建")
  async def build_recipe_knowledge_base(...):
  ```
- 如此一来，该接口在 `ms-py-agent` 中对外暴露出最标准的统一物理路径：`/rest/kb/v1/agent/knowledge/build`。

### 3.2 网关转发链路打通 (`ms-java-gateway` 支持)
- 在网关的 `application.yml` 路由定义中，对 `agent-route` 进行匹配扩展，将 `/rest/kb/v1/**` 也一并代理路由到 `ms-py-agent` 实例上：
  ```yaml
  - id: agent-route
    uri: lb://ms-py-agent
    predicates:
      - Path=/rest/dark/v1/agent/**,/rest/kb/v1/**
  ```

### 3.3 前端统一适配 (`ms-ng-view` 适配)
- 将前端 [url.config.ts](file:///Users/pei/projects/ms-ng-view/src/app/core/infrastructure/constants/url.config.ts) 中的 `BUILD_RECIPE` 端点地址统一调整为符合 `/rest/kb/v1/` 命名规范的新地址：
  ```typescript
  BUILD_RECIPE: '/rest/kb/v1/agent/knowledge/build'
  ```

---

## 4. 验证与总结 (Validation & Lessons)
- **单元测试全量通过**：
  - `ms-py-agent` 的全量单元测试（47/47）顺利通过，未引起任何 regression。
  - 前端 Angular 的 Jest 适配器与 Mock 测试（46/46）全量绿灯通过。
- **系统架构更优雅**：
  - 避免了接口前缀的随意分化。所有关于知识库（Knowledge Base）的原子端点全部收归到统一前缀 `/rest/kb/v1/` 下，API 风格更加规整高内聚。
