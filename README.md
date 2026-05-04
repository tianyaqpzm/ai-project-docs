

## 整体架构

整体架构设计
我们将系统分为四层：接入层、应用层、数据层、基础设施层。

```Mermaid
graph TD
    %% ==========================================
    %% 外部层
    %% ==========================================
    User(("用户 / 前端"))
    LLM_Provider["外部大模型 API\n(OpenAI / DeepSeek)"]
    
    %% ==========================================
    %% 网关层 (Spring Cloud Gateway)
    %% ==========================================
    subgraph "接入层 (DMZ)"
        Gateway["ms-java-gateway\n(Spring Cloud Gateway + WebFlux)"]
        style Gateway fill:#f3e5f5,stroke:#4a148c
    end

    %% ==========================================
    %% 服务治理
    %% ==========================================
    subgraph "基础设施 (Infra)"
        Nacos["Nacos Cluster\n(注册 & 配置中心)"]
        style Nacos fill:#e3f2fd,stroke:#0d47a1
    end

    %% ==========================================
    %% 内部服务层
    %% ==========================================
    subgraph "应用层 (Microservices)"
        %% Java 服务
        JavaApp["Java 业务服务\n(Spring Boot + LangChain4j)\n职责：RAG, 业务工具, MCP Host"]
        %% ms-java-biz
        style JavaApp fill:#e8f5e9,stroke:#1b5e20
        
        %% ms-py-agent
        PythonApp["ms-py-agent 服务\n(FastAPI + LangGraph)\n职责：复杂推理, 编排"]
        style PythonApp fill:#fff3e0,stroke:#e65100
        
        %% 互通
        JavaApp <==>|"内部调用 (HTTP/RPC)"| PythonApp
        JavaApp -.-> Nacos
        PythonApp -.-> Nacos
    end

    %% ==========================================
    %% 数据层
    %% ==========================================
    subgraph "数据层 (Persistence)"
        PgVector[("PostgreSQL (PgVector)\n长期记忆")]
        Mongo[("MongoDB\n短期记忆 (Chat Memory)")]
    end

    %% ==========================================
    %% 连线
    %% ==========================================
    User ==>|"1. HTTPS/WebSocket"| Gateway
    Gateway ==>|"2. 路由分发"| JavaApp
    Gateway ==>|"2. 路由分发"| PythonApp
    
    JavaApp --> PgVector
    JavaApp --> Mongo
    
    %% AI Proxy 模式
    JavaApp -.->|"3. 请求 LLM"| Gateway
    PythonApp -.->|"3. 请求 LLM"| Gateway
    Gateway -.->|"4. 鉴权/计费/审计"| LLM_Provider
```
