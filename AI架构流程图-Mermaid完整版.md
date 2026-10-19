# OPS AI 助手架构流程图（Mermaid 完整版）

> 基于现有架构 + 10 个功能增强

---

## 一、整体架构全景图

```mermaid
graph TB
    subgraph 用户层["用户层（增强交互）"]
        U1[多轮对话]
        U2[调试界面]
        U3[反馈收集]
        U4[推荐展示]
    end
    
    subgraph 前端层["前端层（React）"]
        F1[对话历史组件]
        F2[任务计划展示]
        F3[执行追踪可视化]
    end
    
    subgraph API层["后端 API 层"]
        A1[上下文管理API]
        A2[任务规划API]
        A3[追踪查询API]
    end
    
    subgraph AI引擎层["AI 引擎层（核心增强）"]
        E1[上下文管理器<br/>多轮对话]
        E2[任务规划器<br/>工具组合]
        E3[缓存管理器<br/>缓存复用]
        E4[权限检查器<br/>细粒度权限]
        E5[追踪器<br/>可观测性]
        E6[重试管理器<br/>容错降级]
        E7[推荐引擎<br/>主动推荐]
        E8[知识搜索器<br/>知识库]
        E9[批量执行器<br/>批量操作]
        E10[反馈分析器<br/>反馈学习]
    end
    
    subgraph eino层["eino 框架"]
        LLM[大模型 API<br/>OpenAI/Ark/DeepSeek]
    end
    
    subgraph MCP层["MCP 协议层"]
        M1[Cluster<br/>K8s查询]
        M2[PostgreSQL<br/>SQL查询]
        M3[Redis<br/>Redis命令]
        M4[Middleware<br/>连接检查]
    end
    
    subgraph 安全层["安全控制层（增强）"]
        S1[策略检查]
        S2[审批控制]
        S3[审计日志]
        S4[时间窗口控制]
        S5[资源范围限制]
    end
    
    subgraph 工具层["工具执行层（增强）"]
        T1[并行执行]
        T2[重试机制]
        T3[缓存读写]
        T4[批量执行]
    end
    
    subgraph 资源层["资源层"]
        R1[(Kubernetes)]
        R2[(PostgreSQL)]
        R3[(Redis)]
        R4[(中间件)]
    end
    
    U1 --> F1
    U2 --> F3
    U3 --> A3
    U4 --> E7
    
    F1 --> A1
    F2 --> A2
    F3 --> A3
    
    A1 --> E1
    A2 --> E2
    A3 --> E5
    
    E1 --> LLM
    E2 --> E9
    E3 --> T3
    E4 --> S1
    E5 --> F3
    E6 --> T2
    E7 --> U4
    E8 --> LLM
    E9 --> T4
    E10 --> E1
    
    LLM --> M1
    LLM --> M2
    LLM --> M3
    LLM --> M4
    
    M1 --> S1
    M2 --> S1
    M3 --> S1
    M4 --> S1
    
    S1 --> S2
    S2 --> S3
    S4 --> S1
    S5 --> S1
    
    S1 --> T1
    T1 --> T2
    T2 --> T3
    T3 --> T4
    
    T4 --> R1
    T4 --> R2
    T4 --> R3
    T4 --> R4
```

---

## 二、完整数据流图

```mermaid
sequenceDiagram
    participant U as 用户
    participant CTX as 上下文管理器
    participant KB as 知识库搜索器
    participant LLM as 大模型
    participant TP as 任务规划器
    participant PERM as 权限检查器
    participant CACHE as 缓存管理器
    participant TOOL as 工具执行器
    participant TRACE as 追踪器
    participant RES as 资源层
    
    U->>CTX: 提问（多轮对话）
    CTX->>CTX: 提取实体引用
    CTX->>CTX: 解析指代词
    CTX->>CTX: 更新焦点资源
    CTX->>KB: 构建消息列表
    
    KB->>KB: 搜索相关知识
    KB->>KB: 返回 Top 3
    KB->>LLM: 注入到系统提示
    
    LLM->>LLM: 生成回答/工具调用
    LLM-->>TP: 工具调用请求
    
    TP->>TP: 生成执行计划
    TP->>TP: 确定并行策略
    TP->>PERM: 提交工具调用
    
    PERM->>PERM: 检查用户角色
    PERM->>PERM: 检查时间窗口
    PERM->>PERM: 检查资源范围
    PERM-->>CACHE: 允许执行
    
    CACHE->>CACHE: 查询缓存
    alt 缓存命中
        CACHE-->>U: 返回缓存结果
    else 缓存未命中
        CACHE->>TOOL: 继续执行
    end
    
    TOOL->>TOOL: 执行工具调用
    TOOL->>TOOL: 失败时自动重试
    TOOL->>TRACE: 记录执行详情
    TOOL->>RES: 查询资源
    RES-->>TOOL: 返回结果
    TOOL->>CACHE: 写入缓存（只读）
    TOOL-->>TP: 返回结果
    
    TP->>TP: 汇总所有结果
    TP->>LLM: 生成最终回答
    LLM-->>TRACE: 记录性能指标
    TRACE-->>U: 显示回答+执行详情
    
    U->>U: 提交反馈（可选）
```

---

## 三、功能分层图

```mermaid
graph TB
    subgraph P0["基础能力（P0 - 必须做）"]
        P0_1[1. 多轮对话<br/>上下文管理器]
        P0_2[2. 可观测性<br/>追踪器]
        P0_3[3. 容错降级<br/>重试管理器]
    end
    
    subgraph P1["增强能力（P1 - 应该做）"]
        P1_1[4. 工具组合<br/>任务规划器]
        P1_2[5. 缓存复用<br/>缓存管理器]
        P1_3[6. 权限控制<br/>权限检查器]
    end
    
    subgraph P2["智能能力（P2 - 可以做）"]
        P2_1[7. 主动推荐<br/>推荐引擎]
        P2_2[8. 知识库<br/>知识搜索器]
        P2_3[9. 批量操作<br/>批量执行器]
        P2_4[10. 反馈学习<br/>反馈分析器]
    end
    
    P0 --> P1
    P1 --> P2
    
    style P0 fill:#ff6b6b,stroke:#333,stroke-width:3px
    style P1 fill:#ffd93d,stroke:#333,stroke-width:3px
    style P2 fill:#6bcf7f,stroke:#333,stroke-width:3px
```

---

## 四、功能依赖关系图

```mermaid
graph LR
    subgraph 基础能力["基础能力（被依赖）"]
        F1[1. 多轮对话]
        F2[2. 可观测性]
        F3[3. 容错降级]
    end
    
    subgraph 增强能力["增强能力（中间层）"]
        F4[4. 工具组合]
        F5[5. 缓存机制]
        F6[6. 权限控制]
    end
    
    subgraph 智能能力["智能能力（顶层）"]
        F7[7. 主动推荐]
        F8[8. 知识库]
        F9[9. 批量操作]
        F10[10. 反馈学习]
    end
    
    F1 -.->|被依赖| F4
    F1 -.->|被依赖| F9
    F1 -.->|被依赖| F8
    
    F2 -.->|被依赖| F4
    F2 -.->|被依赖| F5
    F2 -.->|被依赖| F7
    F2 -.->|被依赖| F10
    
    F3 -.->|被依赖| F4
    F3 -.->|被依赖| F9
    
    F4 -.->|依赖| F9
    F5 -.->|依赖| F7
    
    style 基础能力 fill:#ff6b6b,stroke:#333,stroke-width:2px
    style 增强能力 fill:#ffd93d,stroke:#333,stroke-width:2px
    style 智能能力 fill:#6bcf7f,stroke:#333,stroke-width:2px
```

---

## 五、实施路线图

```mermaid
gantt
    title AI 助手功能增强实施路线图
    dateFormat  YYYY-MM-DD
    section 第一阶段（P0）
    多轮对话上下文管理     :2026-10-15, 30d
    可观测性和调试能力     :2026-10-15, 30d
    容错和降级能力         :2026-10-15, 30d
    
    section 第二阶段（P1）
    工具组合能力           :2026-11-15, 30d
    缓存和复用机制         :2026-11-15, 30d
    权限细粒度控制         :2026-11-15, 30d
    
    section 第三阶段（P2）
    主动推荐能力           :2026-12-15, 30d
    知识库和FAQ            :2026-12-15, 30d
    批量操作能力           :2026-12-15, 30d
    反馈和学习机制         :2026-12-15, 30d
```

---

## 六、增强前 vs 增强后对比

```mermaid
graph TB
    subgraph 增强前["增强前（现有架构）"]
        direction TB
        OLD1[单轮对话]
        OLD2[单工具调用]
        OLD3[无追踪]
        OLD4[无容错]
        OLD5[无缓存]
        OLD6[简单权限]
        OLD7[被动回答]
        OLD8[通用知识]
    end
    
    subgraph 增强后["增强后（完整架构）"]
        direction TB
        NEW1[多轮对话<br/>上下文记忆]
        NEW2[工具组合<br/>自动分解]
        NEW3[完整追踪<br/>可观测]
        NEW4[自动重试<br/>降级处理]
        NEW5[缓存复用<br/>高性能]
        NEW6[细粒度权限<br/>RBAC]
        NEW7[主动推荐<br/>预警推送]
        NEW8[企业知识<br/>最佳实践]
    end
    
    OLD1 --> NEW1
    OLD2 --> NEW2
    OLD3 --> NEW3
    OLD4 --> NEW4
    OLD5 --> NEW5
    OLD6 --> NEW6
    OLD7 --> NEW7
    OLD8 --> NEW8
    
    style 增强前 fill:#ffcccc,stroke:#333,stroke-width:2px
    style 增强后 fill:#ccffcc,stroke:#333,stroke-width:2px
```

---

## 七、10 个功能详细架构图

### 7.1 多轮对话上下文管理

```mermaid
graph LR
    A[用户提问] --> B[上下文管理器]
    B --> C[提取实体引用]
    C --> D[解析指代词]
    D --> E[更新焦点资源]
    E --> F[构建消息列表]
    F --> G[系统提示+历史+当前]
    G --> H[大模型调用]
    H --> I[返回回答]
    I --> J[更新上下文]
    
    style B fill:#6bcf7f,stroke:#333,stroke-width:2px
```

### 7.2 工具组合能力

```mermaid
graph TB
    A[复杂问题] --> B[任务规划器]
    B --> C[生成执行计划]
    C --> D[步骤1: 工具A]
    C --> E[步骤2: 工具B]
    C --> F[步骤3: 工具C]
    D --> G[并行执行引擎]
    E --> G
    F --> G
    G --> H[结果汇总器]
    H --> I[大模型生成综合分析]
    
    style B fill:#6bcf7f,stroke:#333,stroke-width:2px
    style G fill:#6bcf7f,stroke:#333,stroke-width:2px
```

### 7.3 容错和降级能力

```mermaid
graph TB
    A[工具调用] --> B{成功?}
    B -->|是| C[返回结果]
    B -->|否| D{可重试?}
    D -->|是| E[等待退避时间]
    E --> F{重试次数<3?}
    F -->|是| G[重试调用]
    G --> B
    F -->|否| H{部分失败?}
    D -->|否| H
    H -->|是| I[继续其他工具]
    H -->|否| J[返回错误]
    I --> K[降级处理]
    
    style E fill:#ffd93d,stroke:#333,stroke-width:2px
```

### 7.4 缓存和复用机制

```mermaid
graph TB
    A[工具调用请求] --> B[计算缓存Key]
    B --> C{查询缓存}
    C -->|命中| D[更新命中次数]
    D --> E[返回缓存结果]
    C -->|未命中| F[执行实际查询]
    F --> G{只读操作?}
    G -->|是| H[写入缓存]
    G -->|否| I[不缓存]
    H --> J[返回结果]
    I --> J
    
    style C fill:#6bcf7f,stroke:#333,stroke-width:2px
```

### 7.5 权限细粒度控制

```mermaid
graph TB
    A[工具调用请求] --> B[加载用户权限]
    B --> C{时间窗口检查}
    C -->|非工作时间| D[需要审批]
    C -->|工作时间| E{资源范围检查}
    E -->|超出范围| F[拒绝]
    E -->|在范围内| G{匹配权限规则}
    G -->|显式拒绝| H[拒绝]
    G -->|显式允许| I[允许]
    G -->|需要审批| D
    G -->|未匹配| J[默认拒绝]
    
    style C fill:#ffd93d,stroke:#333,stroke-width:2px
    style E fill:#ffd93d,stroke:#333,stroke-width:2px
```

---

## 八、完整调用链路时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant Frontend as 前端
    participant API as 后端API
    participant CTX as 上下文管理器
    participant KB as 知识搜索器
    participant LLM as 大模型
    participant TP as 任务规划器
    participant PERM as 权限检查器
    participant CACHE as 缓存管理器
    participant TOOL as 工具执行器
    participant TRACE as 追踪器
    participant DB as 数据库/K8s
    
    User->>Frontend: 提问
    Frontend->>API: POST /api/ai-agent/runs
    API->>CTX: 更新上下文
    CTX->>CTX: 提取实体+解析指代词
    CTX->>KB: 搜索知识
    KB-->>CTX: Top 3 知识
    CTX->>LLM: 构建消息+调用
    LLM-->>API: 工具调用请求
    API->>TP: 生成执行计划
    TP->>PERM: 检查权限
    PERM-->>TP: 允许
    TP->>CACHE: 查询缓存
    CACHE-->>TP: 未命中
    TP->>TOOL: 执行工具
    TOOL->>TRACE: 记录步骤
    TOOL->>DB: 查询
    DB-->>TOOL: 结果
    TOOL->>CACHE: 写入缓存
    TOOL-->>TP: 结果
    TP->>LLM: 汇总结果
    LLM-->>API: 最终回答
    API->>TRACE: 记录完成
    API-->>Frontend: 回答+追踪
    Frontend-->>User: 显示结果
    User->>API: 提交反馈（可选）
```

---

## 九、总结

```mermaid
graph TB
    subgraph 核心改进["核心改进"]
        A[现有架构] --> B[+ 10个功能]
        B --> C[智能运维助手]
    end
    
    subgraph 关键数字["关键数字"]
        D[10个<br/>核心功能]
        E[3个<br/>实施阶段]
        F[1-4月<br/>实施周期]
    end
    
    subgraph 最终效果["最终效果"]
        G1[智能对话]
        G2[自动规划]
        G3[主动发现]
        G4[持续学习]
        G5[企业知识]
        G6[高可用]
        G7[高性能]
        G8[安全可控]
    end
    
    核心改进 --> 关键数字
    关键数字 --> 最终效果
    
    style 核心改进 fill:#6bcf7f,stroke:#333,stroke-width:3px
    style 关键数字 fill:#ffd93d,stroke:#333,stroke-width:3px
    style 最终效果 fill:#4dabf7,stroke:#333,stroke-width:3px
```

---

**这份 Mermaid 架构流程图完全基于现有架构，展示了 10 个功能增强的完整设计！**
