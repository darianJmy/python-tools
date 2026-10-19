# OPS AI 助手架构图

## 1. 整体架构流程

```mermaid
graph TB
    subgraph 用户层
        A[用户提问<br/>例如: 查看 web-01 的 Pod 状态]
    end
    
    subgraph 前端层
        B[React 前端<br/>AI 助手界面]
    end
    
    subgraph 后端API层
        C[REST API<br/>/api/ai-agent/runs]
    end
    
    subgraph AI引擎层
        D[eino 框架<br/>AI Agent 运行时]
        E[大模型 API<br/>OpenAI/Ark/DeepSeek]
    end
    
    subgraph MCP协议层
        F[MCP Server<br/>工具注册中心]
    end
    
    subgraph 安全控制层
        G[策略检查<br/>Policy Check]
        H[审批控制<br/>Approval]
        I[审计日志<br/>Audit Log]
    end
    
    subgraph 工具执行层
        J[Cluster 工具<br/>K8s 查询]
        K[PostgreSQL 工具<br/>SQL 查询]
        L[Redis 工具<br/>Redis 命令]
        M[Middleware 工具<br/>连接检查]
    end
    
    subgraph 资源层
        N[(Kubernetes)]
        O[(PostgreSQL)]
        P[(Redis)]
        Q[(中间件)]
    end
    
    A --> B
    B --> C
    C --> D
    D -->|调用大模型| E
    E -->|返回工具调用请求| D
    D -->|转发工具调用| F
    F -->|检查策略| G
    G -->|需要审批?| H
    H -->|用户确认| I
    I -->|执行工具| J
    I -->|执行工具| K
    I -->|执行工具| L
    I -->|执行工具| M
    J --> N
    K --> O
    L --> P
    M --> Q
    N -->|返回结果| J
    O -->|返回结果| K
    P -->|返回结果| L
    Q -->|返回结果| M
    J -->|结果| F
    K -->|结果| F
    L -->|结果| F
    M -->|结果| F
    F -->|结果| D
    D -->|生成回答| E
    E -->|返回回答| D
    D -->|回答| C
    C -->|回答| B
    B -->|显示回答| A
```

## 2. 工具调用详细流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant F as 前端
    participant API as 后端API
    participant E as eino引擎
    participant LLM as 大模型
    participant MCP as MCP Server
    participant GW as Gateway
    participant Tool as 工具执行器
    participant DB as 数据库/K8s
    
    U->>F: 提问 "查看 users 表的前10条记录"
    F->>API: POST /api/ai-agent/runs
    API->>E: 创建 AI Run
    E->>LLM: 发送问题 + 可用工具列表
    Note over LLM: 大模型分析：<br/>需要调用 PostgreSQL 工具
    LLM-->>E: 返回工具调用请求<br/>Tool: postgresql<br/>Args: {sql: "SELECT * FROM users LIMIT 10"}
    E->>MCP: 转发工具调用
    MCP->>GW: 检查策略
    GW->>GW: 解析 SQL<br/>检查是否只读
    GW->>GW: 策略匹配<br/>SELECT = 允许
    GW-->>MCP: 允许执行
    MCP->>Tool: 执行 PostgreSQL 查询
    Tool->>DB: SELECT * FROM users LIMIT 10
    DB-->>Tool: 返回 10 条记录
    Tool-->>MCP: 返回查询结果
    MCP-->>E: 返回工具结果
    E->>LLM: 发送工具结果
    Note over LLM: 大模型根据结果<br/>生成自然语言回答
    LLM-->>E: 返回回答 "users 表的前10条记录如下..."
    E-->>API: 返回回答
    API-->>F: 返回回答
    F-->>U: 显示回答
```

## 3. 安全控制流程

```mermaid
graph LR
    A[工具调用请求] --> B{策略检查}
    B -->|只读操作| C[自动允许]
    B -->|变更操作| D{需要审批?}
    D -->|是| E[弹出审批对话框]
    E --> F{用户决策}
    F -->|允许| G[执行操作]
    F -->|拒绝| H[返回拒绝]
    D -->|否| G
    C --> G
    G --> I[记录审计日志]
    I --> J[返回结果]
    H --> J
```

## 4. 工具注册机制

```mermaid
graph TB
    subgraph MCP Server 启动
        A[mcpops/server.go] --> B[注册工具]
        B --> C[ToolCluster<br/>K8s 查询]
        B --> D[ToolPostgreSQL<br/>SQL 查询]
        B --> E[ToolRedis<br/>Redis 命令]
        B --> F[ToolMiddleware<br/>连接检查]
    end
    
    subgraph 工具定义
        C --> G[名称: cluster<br/>参数: operation, kind, namespace...]
        D --> H[名称: postgresql<br/>参数: connection_id, sql]
        E --> I[名称: redis<br/>参数: connection_id, command...]
        F --> J[名称: middleware<br/>参数: operation, connection_id]
    end
    
    subgraph 工具执行
        G --> K[clusterai.Arguments]
        H --> L[postgresqlai.Arguments]
        I --> M[redisai.Arguments]
        J --> N[middlewareai.Arguments]
    end
```

## 5. 对比：业界主流 AI Agent 架构

### 方案 A：MCP 协议（你的平台采用）

```
用户 → AI 引擎 → MCP Server → 工具执行器 → 资源
         ↑                    ↓
         └──── 结果返回 ──────┘
```

**优点**：
- ✅ 标准化工具调用协议
- ✅ 工具与 AI 引擎解耦
- ✅ 易于扩展新工具
- ✅ 安全性好（集中控制）

**缺点**：
- ❌ 需要自己实现 MCP Server
- ❌ 工具调用链路较长

**代表**：Anthropic MCP、你的平台

### 方案 B：Function Calling（OpenAI 原生）

```
用户 → AI 引擎 → 大模型 → Function Call → 工具执行器 → 资源
         ↑                                    ↓
         └──────── 结果返回 ──────────────────┘
```

**优点**：
- ✅ 大模型原生支持
- ✅ 实现简单
- ✅ 调用链路短

**缺点**：
- ❌ 工具定义受限于大模型 API
- ❌ 不同大模型 API 不兼容
- ❌ 安全性控制分散

**代表**：OpenAI、Azure OpenAI

### 方案 C：LangChain Agent

```
用户 → LangChain Agent → 工具链 → 工具执行器 → 资源
              ↑                          ↓
              └────── 结果返回 ──────────┘
```

**优点**：
- ✅ 生态丰富
- ✅ 支持多种工具
- ✅ 社区活跃

**缺点**：
- ❌ 框架较重
- ❌ 学习成本高
- ❌ 性能开销大

**代表**：LangChain、LlamaIndex

### 方案 D：自定义 Agent（你的平台如果不用 MCP）

```
用户 → 自定义 Agent → 工具注册表 → 工具执行器 → 资源
            ↑                          ↓
            └────── 结果返回 ──────────┘
```

**优点**：
- ✅ 完全可控
- ✅ 性能最优
- ✅ 灵活度高

**缺点**：
- ❌ 需要自己实现所有逻辑
- ❌ 维护成本高
- ❌ 难以扩展

## 6. 你的平台架构优势

```mermaid
graph TB
    subgraph 你的平台
        A[eino 框架] --> B[MCP Server]
        B --> C[Gateway 安全层]
        C --> D[工具执行器]
    end
    
    subgraph 优势
        E[标准化工具协议<br/>MCP]
        F[集中安全控制<br/>策略+审批+审计]
        G[易于扩展<br/>添加工具只需 4 步]
        H[框架解耦<br/>可更换 AI 引擎]
    end
    
    A -.-> E
    B -.-> E
    C -.-> F
    D -.-> G
    A -.-> H
```

## 7. 添加新工具的步骤

```mermaid
graph LR
    A[1. 定义工具<br/>mcpops/contracts.go] --> B[2. 注册工具<br/>mcpops/server.go]
    B --> C[3. 实现执行逻辑<br/>service/aiagent/mcp_gateway.go]
    C --> D[4. 创建执行器<br/>ai/tool/新工具/]
    D --> E[完成!]
    
    style A fill:#e1f5ff
    style B fill:#e1f5ff
    style C fill:#e1f5ff
    style D fill:#e1f5ff
    style E fill:#c8e6c9
```

## 8. 实际使用场景

### 场景 1：查询 K8s 资源
```
用户: "查看 web-01 的 Pod 状态"
  ↓
AI: 调用 ToolCluster
  ↓
执行: kubectl get pod web-01
  ↓
返回: Pod 运行中，CPU 使用率 45%
```

### 场景 2：查询数据库
```
用户: "统计每个部门的员工数量"
  ↓
AI: 调用 ToolPostgreSQL
  ↓
执行: SELECT department, COUNT(*) FROM employees GROUP BY department
  ↓
返回: 技术部 50 人，销售部 30 人，市场部 20 人
```

### 场景 3：查看 Redis
```
用户: "查看 session:* 的 key 有多少个"
  ↓
AI: 调用 ToolRedis
  ↓
执行: SCAN 0 MATCH session:* COUNT 1000
  ↓
返回: 共找到 1,234 个 session key
```

## 9. 总结

你的平台采用的 **MCP + eino** 架构是**业界先进的 AI Agent 实现方案**：

1. **标准化工具协议** - MCP 是 Anthropic 推出的开放标准
2. **集中安全控制** - 策略、审批、审计一体化
3. **易于扩展** - 添加工具只需 4 步
4. **框架解耦** - 可以更换 AI 引擎而不影响工具层

**这个架构适合企业级运维场景**，既保证了 AI 的灵活性，又确保了操作的安全性。
