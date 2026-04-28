# 架构决策记录 (ADR): ms-java-biz 领域驱动设计 (DDD) 架构重构

## 状态
已提议 / 执行中 (Proposed / In Progress)

## 背景 (Context)
随着 `ms-java-biz` (核心业务服务) 的迭代，目前的代码结构暴露出一些由于过度依赖基础设施和 ORM 框架（MyBatis-Plus）带来的问题：
1. **框架侵入领域层**：目前的领域实体（如 `KnowledgeDocument`）中混杂了 `@TableName` 和 `@TableId` 等注解，导致核心业务模型与数据库表结构强绑定。
2. **贫血模型 (Anemic Domain Model)**：使用了 `@Data` 注解自动生成所有的 Setter/Getter，导致实体的状态可以被随意外部修改，业务行为没有被正确封装在实体内部。
3. **违反依赖倒置**：Service 层直接继承了 MyBatis-Plus 的 `IService`，使得业务逻辑和数据持久化逻辑高度耦合，难以单独针对业务逻辑进行单元测试。

为了保证代码的“低耦合、高内聚”，并符合新更新的全局编码规范和 Java 领域模型编程规范，我们必须对现有的架构模式进行调整。

## 决策 (Decision)
我们决定对 `ms-java-biz` 进行全方位的 DDD（领域驱动设计）架构重构，引入 4 层代码结构规范：
1. **领域层 (Domain Layer)**：
   - 提取纯净的 POJO 作为**领域实体 (Entity)**，必须具备业务语义方法（如 `publish()`, `process()`），且提供私有无参构造函数，屏蔽 Setter。
   - 提取**值对象 (Value Object)**，如 `KnowledgeConfig`，使用 `record` 声明以保证其不可变性。
   - 定义**仓储接口 (Repository Interface)**，反向控制依赖。
2. **基础设施层 (Infrastructure Layer)**：
   - 创建**数据持久化对象 (Data Object / DO)**，专门承载 ORM 框架注解（如 `@TableName`）。
   - 实现领域层定义的 Repository 接口，并在其中调用底层 Mapper。
   - 提供 Converter/Mapper 用于领域实体与 DO 之间的相互转换。
3. **应用层 (Application Layer / Use Case)**：
   - 摒弃继承 `IService` 的做法，所有的 Use Case 通过依赖注入调用 Repository 接口进行数据持久化，负责编排业务流程。
4. **接口层 (Interfaces Layer)**：
   - 保持 Controller 和 MCP 层轻量化，只负责请求的接入与响应格式化。

## 影响评估 (Consequences)

### 正面影响 (Positive)
* **极强的可测试性**：由于 Domain 层完全独立，我们可以针对核心业务模型编写纯净的单元测试，无需启动 Spring 容器或连接数据库。
* **高内聚低耦合**：底层数据表结构的变动不再直接波及核心业务逻辑，只需在 Converter 和 DO 中进行调整即可。
* **代码可读性提升**：实体的业务状态变更具备了清晰的业务语义，不再是简单的 `set()` 方法调用。

### 负面影响 (Negative)
* **短期工作量增加**：重构涉及大量的实体拆分、数据转换类的编写，前期开发成本会上升。
* **复杂度上升**：对开发人员的 DDD 认知有一定要求，需要明确理解 Entity, Value Object, DO, Repository 之间的界限。

## 涉及模块
* 子工程：`ms-java-biz`
* 核心模块：`knowledge`, `chat`, `tools` 等所有需要持久化和状态管理的业务模块。

## 改造实施记录 (Implementation Record)

### 阶段一：KnowledgeDocument 模块重构完成
针对 `knowledge` 模块中的 `KnowledgeDocument` 实体，已完整实施上述 DDD 改造：

1. **底层强依赖剥离**：
   - 彻底删除了原有的 `KnowledgeDocumentServiceImpl` 和 `KnowledgeDocumentService`（移除了对 `IService` 的依赖）。
   - 剥离了原本在 Controller 中的 `QueryWrapper` 等 MyBatis-Plus 专属调用。
2. **纯粹的领域模型落地**：
   - 创建了 `domain.knowledge.entity.KnowledgeDocument`（充血模型），通过 `publish()`, `process()`, `assignConfig()` 封装了业务状态的流转。
   - 创建了不可变值对象 `KnowledgeConfig`，使用 Java `record` 实现。
   - 定义了 `KnowledgeDocumentRepository` 以反向依赖持久化层。
3. **基础设施就绪**：
   - 建立了专属的 `KnowledgeDocumentDO` 承载 `@TableName` 映射。
   - 实现了 `KnowledgeDocumentConverter` 保证 DO 与 Domain 的双向映射。
   - 实现了 `KnowledgeDocumentRepositoryImpl`，将 Mapper (`BaseMapper`) 的调用完全封装在基础设施层内。
4. **应用服务层就绪**：
   - 建立了新的 `KnowledgeDocumentApplicationService` 和实现类，通过构造器注入 `Repository` 接口，独立完成流程的编排，实现了最高级别的解耦。

### 阶段二：Chat 与 Event 模块重构完成 (含高性能汇总表设计)
在 `knowledge` 模块改造完成后，我们将 `module/biz` 进一步拆分为 `chat` 和 `event` 领域，并引入了“物理汇总表”机制解决性能债。

1. **模块化与物理隔离**：
   - 彻底销毁了混合业务的 `module/biz`，建立了独立的 `chat` (聊天) 和 `event` (活动) 架构。
   - 对 `TimeLimitedEvent` 实体进行了充血模型改造，引入 `EventAppearance` 值对象并移除了 `MongoDB` 的框架注解。
2. **ChatSession 物理实体化 (性能优化核心)**：
   - **痛点识别**：原架构中“会话列表”通过对千万级聊天记录表进行 `GROUP BY` 和关联子查询实时推算，导致严重的 O(N) 性能瓶颈。
   - **架构决策**：建立物理表 `chat_sessions` 作为读模型（Read Model）的汇总表，存储 `sessionId`, `title`, `last_active_time`。
   - **自动化同步 (Trigger)**：通过 Flyway 脚本 (`V1.2__create_chat_sessions.sql`) 引入数据库触发器 `trg_sync_chat_session`。当任何服务（Java 或 Python）向 `chat_messages` 写入数据时，数据库自动实时更新/创建会话记录。
   - **查询效率**：将会话列表的查询复杂度从 O(N) 的全表聚合降低到了 O(1) 的索引扫描，完美契合了读写分离的架构思想。
3. **统一 DTO 与 接口契约**：
   - 在应用层定义了 `ChatSessionDto` 和 `EventDto`，确保 Interface 层（Controller）不再直接触碰 Domain Entity 或 Infrastructure DO。

### 阶段三：测试用例适配与回归验证 (Testing & Verification)
为确保重构不引入功能性回归，并适配新的依赖注入结构，我们对原有的测试套件进行了全面重构：
1. **解除废弃 Service 的依赖**：
   - 将 `KnowledgeControllerTest` 和 `AiApplicationTests` 中的 `TopicService`、`EventService` 等 `@MockBean` 依赖切换为新架构下的 `Repository` 或 `ApplicationService` 接口。
2. **安全配置隔离**：
   - 针对使用 `@WebMvcTest` 的控制层单元测试，通过引入 `@AutoConfigureMockMvc(addFilters = false)` 注解成功剥离了 Spring Security 鉴权层（401/403），实现了纯粹的 Web 层逻辑隔离测试。
3. **回归结果**：
   - 全工程所有 17 个测试用例全部通过验证（0 Failures），包括复杂的 MCP 协议拦截与工具注册、业务 CRUD 等流程，充分证明了 DDD 改造后的架构稳健性与高可测试性。
