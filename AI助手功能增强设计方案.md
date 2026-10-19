# OPS AI 助手功能增强设计方案

## 目录

- [1. 多轮对话上下文管理](#1-多轮对话上下文管理)
- [2. 工具组合能力](#2-工具组合能力)
- [3. 可观测性和调试能力](#3-可观测性和调试能力)
- [4. 容错和降级能力](#4-容错和降级能力)
- [5. 缓存和复用机制](#5-缓存和复用机制)
- [6. 权限细粒度控制](#6-权限细粒度控制)
- [7. 主动推荐能力](#7-主动推荐能力)
- [8. 知识库和 FAQ](#8-知识库和-faq)
- [9. 批量操作能力](#9-批量操作能力)
- [10. 反馈和学习机制](#10-反馈和学习机制)
- [11. 实施路线图](#11-实施路线图)

---

## 1. 多轮对话上下文管理

### 1.1 问题描述

当前每次对话都是独立的，AI 无法理解上下文引用。

**示例**：
```
用户：查看 web-01 的 Pod 状态
AI：Pod 运行中，CPU 45%
用户：那内存呢？  ← AI 不知道问的是 web-01
AI：请指定要查看哪个资源的内存  ← 错误回答
```

### 1.2 设计方案

#### 数据模型

```go
// internal/models/ai_conversation_context.go
type ConversationContext struct {
    ID              string                 `json:"id"`
    ConversationID  string                 `json:"conversation_id"`
    UserID          string                 `json:"user_id"`
    
    // 上下文窗口（最近 N 轮对话）
    Messages        []Message              `json:"messages"`
    
    // 实体引用（自动提取的资源名称）
    EntityRefs      map[string]EntityRef   `json:"entity_refs"`
    
    // 当前焦点资源
    FocusResource   *ResourceRef           `json:"focus_resource"`
    
    // 创建和更新时间
    CreatedAt       time.Time              `json:"created_at"`
    UpdatedAt       time.Time              `json:"updated_at"`
}

type Message struct {
    Role        string    `json:"role"`        // user / assistant / tool
    Content     string    `json:"content"`
    ToolCalls   []ToolCall `json:"tool_calls,omitempty"`
    Timestamp   time.Time `json:"timestamp"`
}

type EntityRef struct {
    Type         string    `json:"type"`         // pod / deployment / database
    Name         string    `json:"name"`         // web-01
    Namespace    string    `json:"namespace"`
    LastMention  time.Time `json:"last_mention"`
    MentionCount int       `json:"mention_count"`
}

type ResourceRef struct {
    Type       string `json:"type"`
    Name       string `json:"name"`
    Namespace  string `json:"namespace"`
}
```

#### 上下文管理器

```go
// internal/service/aiagent/context_manager.go
type ContextManager struct {
    repo           repo.IConversationContextRepo
    maxWindowSize  int           // 最多保留 10 轮对话
    entityExpiry   time.Duration // 实体引用过期时间（5 分钟）
}

func (m *ContextManager) UpdateContext(
    ctx context.Context,
    conversationID string,
    userMessage string,
) (*ConversationContext, error) {
    // 1. 加载现有上下文
    context, err := m.repo.GetByConversationID(ctx, conversationID)
    if err != nil {
        context = &ConversationContext{
            ConversationID: conversationID,
            Messages:       []Message{},
            EntityRefs:     make(map[string]EntityRef),
        }
    }
    
    // 2. 提取实体引用
    entities := m.extractEntities(userMessage)
    m.updateEntityRefs(context, entities)
    
    // 3. 更新焦点资源
    if len(entities) > 0 {
        context.FocusResource = &ResourceRef{
            Type:      entities[0].Type,
            Name:      entities[0].Name,
            Namespace: entities[0].Namespace,
        }
    }
    
    // 4. 添加消息到窗口
    context.Messages = append(context.Messages, Message{
        Role:      "user",
        Content:   userMessage,
        Timestamp: time.Now(),
    })
    
    // 5. 保持窗口大小
    if len(context.Messages) > m.maxWindowSize*2 {
        context.Messages = context.Messages[len(context.Messages)-m.maxWindowSize*2:]
    }
    
    // 6. 清理过期实体
    m.cleanupExpiredEntities(context)
    
    // 7. 保存上下文
    return m.repo.Save(ctx, context)
}

// 提取实体引用（使用正则或 NLP）
func (m *ContextManager) extractEntities(message string) []EntityRef {
    entities := []EntityRef{}
    
    // 正则匹配 Pod 名称
    podPattern := regexp.MustCompile(`\b([a-z0-9-]+-[a-z0-9]+-[a-z0-9]+)\b`)
    if matches := podPattern.FindStringSubmatch(message); len(matches) > 0 {
        entities = append(entities, EntityRef{
            Type:        "pod",
            Name:        matches[1],
            LastMention: time.Now(),
        })
    }
    
    // 匹配 namespace
    nsPattern := regexp.MustCompile(`(?:namespace|ns)\s+([a-z0-9-]+)`)
    if matches := nsPattern.FindStringSubmatch(message); len(matches) > 0 {
        entities = append(entities, EntityRef{
            Type:        "namespace",
            Name:        matches[1],
            LastMention: time.Now(),
        })
    }
    
    return entities
}

// 解析指代词（这、那、它、他等）
func (m *ContextManager) resolvePronouns(
    message string,
    context *ConversationContext,
) string {
    pronouns := []string{"这", "那", "它", "他", "她", "这个", "那个"}
    
    for _, pronoun := range pronouns {
        if strings.Contains(message, pronoun) && context.FocusResource != nil {
            // 替换指代词为具体资源名称
            message = strings.Replace(message, pronoun, 
                context.FocusResource.Name, 1)
        }
    }
    
    return message
}
```

#### 集成到 AI 引擎

```go
// internal/service/aiagent/run_service.go
func (s *Service) CreateRun(
    ctx context.Context,
    actor Actor,
    conversationID string,
    req *request.AIAgentRunCreateRequest,
) (*response.AIAgentRunResponse, error) {
    // 1. 更新上下文
    context, err := s.contextManager.UpdateContext(ctx, conversationID, req.Message)
    if err != nil {
        return nil, err
    }
    
    // 2. 解析指代词
    resolvedMessage := s.contextManager.resolvePronouns(req.Message, context)
    
    // 3. 构建消息列表（包含历史）
    messages := s.buildMessagesWithContext(context, resolvedMessage)
    
    // 4. 调用大模型（带上下文）
    response, err := s.einoRuntime.Send(ctx, messages, s.tools)
    
    // 5. 更新上下文（添加 AI 回答）
    context.Messages = append(context.Messages, Message{
        Role:      "assistant",
        Content:   response.Content,
        Timestamp: time.Now(),
    })
    s.contextManager.Save(ctx, context)
    
    return response, nil
}

// 构建带上下文的消息列表
func (s *Service) buildMessagesWithContext(
    context *ConversationContext,
    currentMessage string,
) []core.Message {
    messages := []core.Message{}
    
    // 添加系统提示
    messages = append(messages, core.Message{
        Role:    "system",
        Content: s.systemPrompt,
    })
    
    // 添加历史消息（最近 5 轮）
    for _, msg := range context.Messages[len(context.Messages)-10:] {
        messages = append(messages, core.Message{
            Role:    msg.Role,
            Content: msg.Content,
        })
    }
    
    // 添加当前消息
    messages = append(messages, core.Message{
        Role:    "user",
        Content: currentMessage,
    })
    
    return messages
}
```

### 1.3 数据库迁移

```go
// migrations/xxx_add_conversation_context.go
func init() {
    registerMigration("add_conversation_context", func(db *gorm.DB) error {
        return db.AutoMigrate(&models.ConversationContext{})
    })
}
```

### 1.4 前端改造

```typescript
// web/src/pages/ai-agent/index.tsx
const [conversationId, setConversationId] = useState<string>("")

const sendMessage = async (message: string) => {
  // 传递 conversation_id
  const response = await aiCreateRunRequest({
    conversation_id: conversationId,
    message: message,
  })
  
  // 更新 UI
  addMessage(response)
}
```

---

## 2. 工具组合能力

### 2.1 问题描述

当前每次只能调用 1 个工具，复杂问题需要用户手动分步提问。

**示例**：
```
用户：分析 web-01 为什么响应慢

AI 应该自动执行：
1. 查看 Pod 状态
2. 查看资源使用率
3. 查看最近日志
4. 查看慢查询
5. 汇总分析
```

### 2.2 设计方案

#### 任务规划器

```go
// internal/ai/planner/task_planner.go
type TaskPlanner struct {
    llmClient  *eino.LLMClient
    toolRegistry *ToolRegistry
}

type TaskPlan struct {
    ID          string        `json:"id"`
    Goal        string        `json:"goal"`
    Steps       []TaskStep    `json:"steps"`
    Status      string        `json:"status"` // pending / running / completed
    Results     []StepResult  `json:"results"`
}

type TaskStep struct {
    ID          string    `json:"id"`
    ToolName    string    `json:"tool_name"`
    Arguments   any       `json:"arguments"`
    DependsOn   []string  `json:"depends_on"` // 依赖的步骤 ID
    Status      string    `json:"status"`
    Result      any       `json:"result"`
}

// 生成任务计划
func (p *TaskPlanner) Plan(
    ctx context.Context,
    goal string,
    availableTools []tool.Tool,
) (*TaskPlan, error) {
    // 构建提示词
    prompt := fmt.Sprintf(`
你是一个运维任务规划器。用户的目标是：%s

可用的工具：
%s

请制定一个执行计划，包括：
1. 需要调用哪些工具
2. 每个工具的参数
3. 步骤之间的依赖关系
4. 哪些步骤可以并行执行

以 JSON 格式返回：
{
  "steps": [
    {
      "id": "step_1",
      "tool_name": "cluster",
      "arguments": {"operation": "get", "name": "web-01"},
      "depends_on": []
    },
    {
      "id": "step_2",
      "tool_name": "cluster",
      "arguments": {"operation": "logs", "name": "web-01"},
      "depends_on": []
    },
    {
      "id": "step_3",
      "tool_name": "postgresql",
      "arguments": {"sql": "SELECT * FROM slow_queries LIMIT 10"},
      "depends_on": ["step_1"]
    }
  ]
}
`, goal, p.formatTools(availableTools))
    
    // 调用大模型生成计划
    response, err := p.llmClient.Generate(ctx, prompt)
    if err != nil {
        return nil, err
    }
    
    // 解析计划
    var plan TaskPlan
    if err := json.Unmarshal([]byte(response), &plan); err != nil {
        return nil, err
    }
    
    plan.ID = uuid.New().String()
    plan.Goal = goal
    plan.Status = "pending"
    
    return &plan, nil
}

// 执行任务计划
func (p *TaskPlanner) Execute(
    ctx context.Context,
    plan *TaskPlan,
) error {
    plan.Status = "running"
    
    // 构建执行图
    graph := p.buildExecutionGraph(plan)
    
    // 拓扑排序，确定执行顺序
    order := graph.TopologicalSort()
    
    // 并行执行无依赖的步骤
    for _, level := range order {
        var wg sync.WaitGroup
        results := make(chan StepResult, len(level))
        
        for _, stepID := range level {
            wg.Add(1)
            go func(id string) {
                defer wg.Done()
                step := plan.GetStep(id)
                result := p.executeStep(ctx, step, plan)
                results <- result
            }(stepID)
        }
        
        wg.Wait()
        close(results)
        
        // 收集结果
        for result := range results {
            plan.Results = append(plan.Results, result)
        }
    }
    
    plan.Status = "completed"
    return nil
}

// 执行单个步骤
func (p *TaskPlanner) executeStep(
    ctx context.Context,
    step *TaskStep,
    plan *TaskPlan,
) StepResult {
    // 解析依赖
    for _, depID := range step.DependsOn {
        depResult := plan.GetResult(depID)
        step.Arguments = p.injectDependency(step.Arguments, depResult)
    }
    
    // 调用工具
    tool := p.toolRegistry.Get(step.ToolName)
    result, err := tool.Execute(ctx, step.Arguments)
    
    return StepResult{
        StepID: step.ID,
        Result: result,
        Error:  err,
    }
}
```

#### 集成到 AI 引擎

```go
// internal/service/aiagent/run_service.go
func (s *Service) CreateRun(
    ctx context.Context,
    actor Actor,
    conversationID string,
    req *request.AIAgentRunCreateRequest,
) (*response.AIAgentRunResponse, error) {
    // 1. 判断是否需要任务规划
    if s.needsPlanning(req.Message) {
        return s.executeWithPlanning(ctx, actor, req)
    }
    
    // 2. 单工具调用（原有逻辑）
    return s.executeSingleTool(ctx, actor, req)
}

// 判断是否需要任务规划
func (s *Service) needsPlanning(message string) bool {
    // 关键词匹配
    keywords := []string{"分析", "为什么", "排查", "诊断", "综合"}
    for _, keyword := range keywords {
        if strings.Contains(message, keyword) {
            return true
        }
    }
    return false
}

// 使用任务规划执行
func (s *Service) executeWithPlanning(
    ctx context.Context,
    actor Actor,
    req *request.AIAgentRunCreateRequest,
) (*response.AIAgentRunResponse, error) {
    // 1. 生成任务计划
    plan, err := s.planner.Plan(ctx, req.Message, s.tools)
    if err != nil {
        return nil, err
    }
    
    // 2. 执行计划
    if err := s.planner.Execute(ctx, plan); err != nil {
        return nil, err
    }
    
    // 3. 汇总结果
    summary := s.summarizeResults(plan)
    
    // 4. 生成最终回答
    response, err := s.einoRuntime.Generate(ctx, summary)
    
    return &response, nil
}
```

### 2.3 前端展示

```typescript
// web/src/components/ai/TaskPlanView.tsx
function TaskPlanView({ plan }: { plan: TaskPlan }) {
  return (
    <div className="task-plan">
      <h3>执行计划</h3>
      <div className="steps">
        {plan.steps.map((step, index) => (
          <div key={step.id} className={`step ${step.status}`}>
            <div className="step-header">
              <span className="step-number">{index + 1}</span>
              <span className="tool-name">{step.tool_name}</span>
              <span className="status">{step.status}</span>
            </div>
            <div className="step-args">
              <pre>{JSON.stringify(step.arguments, null, 2)}</pre>
            </div>
            {step.result && (
              <div className="step-result">
                <pre>{JSON.stringify(step.result, null, 2)}</pre>
              </div>
            )}
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## 3. 可观测性和调试能力

### 3.1 问题描述

当前 AI 助手是黑盒运行，出问题无法排查。

### 3.2 设计方案

#### 执行追踪器

```go
// internal/ai/trace/tracer.go
type Tracer struct {
    repo repo.ITraceRepo
}

type Trace struct {
    ID              string         `json:"id"`
    RunID           string         `json:"run_id"`
    UserID          string         `json:"user_id"`
    ConversationID  string         `json:"conversation_id"`
    
    // 输入
    UserMessage     string         `json:"user_message"`
    
    // 执行过程
    Steps           []TraceStep    `json:"steps"`
    
    // 输出
    FinalAnswer     string         `json:"final_answer"`
    
    // 性能指标
    TotalDuration   time.Duration  `json:"total_duration"`
    ToolCallCount   int            `json:"tool_call_count"`
    TokenUsage      TokenUsage     `json:"token_usage"`
    
    // 状态
    Status          string         `json:"status"` // success / failed
    ErrorMessage    string         `json:"error_message,omitempty"`
    
    CreatedAt       time.Time      `json:"created_at"`
}

type TraceStep struct {
    ID          string        `json:"id"`
    Type        string        `json:"type"` // llm_call / tool_call / policy_check
    Name        string        `json:"name"`
    Input       any           `json:"input"`
    Output      any           `json:"output"`
    Duration    time.Duration `json:"duration"`
    Status      string        `json:"status"`
    Error       string        `json:"error,omitempty"`
    Timestamp   time.Time     `json:"timestamp"`
}

type TokenUsage struct {
    PromptTokens     int `json:"prompt_tokens"`
    CompletionTokens int `json:"completion_tokens"`
    TotalTokens      int `json:"total_tokens"`
}

// 开始追踪
func (t *Tracer) StartTrace(
    ctx context.Context,
    runID string,
    userMessage string,
) *Trace {
    return &Trace{
        ID:             uuid.New().String(),
        RunID:          runID,
        UserMessage:    userMessage,
        Steps:          []TraceStep{},
        Status:         "running",
        CreatedAt:      time.Now(),
    }
}

// 记录步骤
func (t *Tracer) RecordStep(trace *Trace, step TraceStep) {
    trace.Steps = append(trace.Steps, step)
}

// 结束追踪
func (t *Tracer) FinishTrace(
    ctx context.Context,
    trace *Trace,
    finalAnswer string,
    err error,
) error {
    trace.FinalAnswer = finalAnswer
    trace.TotalDuration = time.Since(trace.CreatedAt)
    trace.ToolCallCount = len(trace.Steps)
    
    if err != nil {
        trace.Status = "failed"
        trace.ErrorMessage = err.Error()
    } else {
        trace.Status = "success"
    }
    
    return t.repo.Save(ctx, trace)
}
```

#### 集成到 AI 引擎

```go
// internal/service/aiagent/run_service.go
func (s *Service) CreateRun(
    ctx context.Context,
    actor Actor,
    conversationID string,
    req *request.AIAgentRunCreateRequest,
) (*response.AIAgentRunResponse, error) {
    // 1. 开始追踪
    trace := s.tracer.StartTrace(ctx, runID, req.Message)
    
    // 2. 执行 AI 流程（记录每一步）
    response, err := s.executeWithTracing(ctx, trace, req)
    
    // 3. 结束追踪
    s.tracer.FinishTrace(ctx, trace, response.Content, err)
    
    return response, err
}

func (s *Service) executeWithTracing(
    ctx context.Context,
    trace *Trace,
    req *request.AIAgentRunCreateRequest,
) (*response.AIAgentRunResponse, error) {
    // 记录 LLM 调用
    start := time.Now()
    llmResponse, err := s.einoRuntime.Send(ctx, messages, tools)
    s.tracer.RecordStep(trace, TraceStep{
        Type:     "llm_call",
        Name:     "generate_response",
        Input:    messages,
        Output:   llmResponse,
        Duration: time.Since(start),
        Status:   statusFromError(err),
    })
    
    // 记录工具调用
    for _, toolCall := range llmResponse.ToolCalls {
        start := time.Now()
        result, err := s.executeTool(ctx, toolCall)
        s.tracer.RecordStep(trace, TraceStep{
            Type:     "tool_call",
            Name:     toolCall.Name,
            Input:    toolCall.Arguments,
            Output:   result,
            Duration: time.Since(start),
            Status:   statusFromError(err),
        })
    }
    
    return response, nil
}
```

#### 调试界面

```typescript
// web/src/pages/ai-agent/TraceViewer.tsx
function TraceViewer({ traceId }: { traceId: string }) {
  const { data: trace } = useTrace(traceId)
  
  return (
    <div className="trace-viewer">
      <div className="trace-header">
        <h2>执行追踪</h2>
        <div className="metrics">
          <span>总耗时: {trace.total_duration}ms</span>
          <span>工具调用: {trace.tool_call_count} 次</span>
          <span>Token 使用: {trace.token_usage.total_tokens}</span>
        </div>
      </div>
      
      <div className="trace-steps">
        {trace.steps.map((step, index) => (
          <div key={step.id} className={`step ${step.status}`}>
            <div className="step-header">
              <span className="step-number">{index + 1}</span>
              <span className="step-type">{step.type}</span>
              <span className="step-name">{step.name}</span>
              <span className="step-duration">{step.duration}ms</span>
            </div>
            
            <details>
              <summary>输入</summary>
              <pre>{JSON.stringify(step.input, null, 2)}</pre>
            </details>
            
            <details>
              <summary>输出</summary>
              <pre>{JSON.stringify(step.output, null, 2)}</pre>
            </details>
            
            {step.error && (
              <div className="step-error">
                <strong>错误:</strong> {step.error}
              </div>
            )}
          </div>
        ))}
      </div>
      
      <div className="trace-result">
        <h3>最终回答</h3>
        <p>{trace.final_answer}</p>
      </div>
    </div>
  )
}
```

---

## 4. 容错和降级能力

### 4.1 问题描述

当前工具调用失败会导致整个 AI 流程失败。

### 4.2 设计方案

#### 重试机制

```go
// internal/ai/retry/retry.go
type RetryPolicy struct {
    MaxAttempts     int           `json:"max_attempts"`
    InitialDelay    time.Duration `json:"initial_delay"`
    MaxDelay        time.Duration `json:"max_delay"`
    BackoffFactor   float64       `json:"backoff_factor"`
    RetryableErrors []string      `json:"retryable_errors"`
}

func (p *RetryPolicy) Execute(
    ctx context.Context,
    fn func() (any, error),
) (any, error) {
    var lastErr error
    delay := p.InitialDelay
    
    for attempt := 1; attempt <= p.MaxAttempts; attempt++ {
        result, err := fn()
        if err == nil {
            return result, nil
        }
        
        lastErr = err
        
        // 检查是否可重试
        if !p.isRetryable(err) {
            return nil, err
        }
        
        // 指数退避
        if attempt < p.MaxAttempts {
            time.Sleep(delay)
            delay = time.Duration(float64(delay) * p.BackoffFactor)
            if delay > p.MaxDelay {
                delay = p.MaxDelay
            }
        }
    }
    
    return nil, fmt.Errorf("max retries exceeded: %w", lastErr)
}

func (p *RetryPolicy) isRetryable(err error) bool {
    errMsg := err.Error()
    for _, retryable := range p.RetryableErrors {
        if strings.Contains(errMsg, retryable) {
            return true
        }
    }
    return false
}
```

#### 降级策略

```go
// internal/ai/fallback/fallback.go
type FallbackStrategy struct {
    // 部分失败时，是否继续
    ContinueOnPartialFailure bool `json:"continue_on_partial_failure"`
    
    // 最大允许失败的工具数量
    MaxFailedTools int `json:"max_failed_tools"`
}

func (s *FallbackStrategy) ExecuteWithFallback(
    ctx context.Context,
    tools []ToolCall,
) (*AggregatedResult, error) {
    results := []ToolResult{}
    errors := []error{}
    
    // 并行执行所有工具
    var wg sync.WaitGroup
    for _, tool := range tools {
        wg.Add(1)
        go func(t ToolCall) {
            defer wg.Done()
            result, err := s.executeToolWithRetry(ctx, t)
            if err != nil {
                errors = append(errors, err)
            } else {
                results = append(results, result)
            }
        }(tool)
    }
    wg.Wait()
    
    // 检查失败率
    if len(errors) > s.MaxFailedTools {
        return nil, fmt.Errorf("too many tools failed: %d", len(errors))
    }
    
    // 部分成功，继续处理
    return &AggregatedResult{
        Results:      results,
        Errors:       errors,
        PartialSuccess: len(errors) > 0,
    }, nil
}
```

---

## 5. 缓存和复用机制

### 5.1 问题描述

相同问题每次都重新查询，浪费资源。

### 5.2 设计方案

#### 缓存管理器

```go
// internal/ai/cache/cache.go
type CacheManager struct {
    cache    *redis.Client
    ttl      time.Duration  // 缓存过期时间（5 分钟）
    maxItems int            // 最多缓存 1000 个查询
}

type CacheKey struct {
    ToolName    string `json:"tool_name"`
    Arguments   string `json:"arguments"` // JSON 序列化
}

type CacheEntry struct {
    Key       CacheKey    `json:"key"`
    Result    any         `json:"result"`
    CachedAt  time.Time   `json:"cached_at"`
    HitCount  int         `json:"hit_count"`
}

// 查询缓存
func (m *CacheManager) Get(
    ctx context.Context,
    toolName string,
    arguments any,
) (any, bool) {
    key := m.buildKey(toolName, arguments)
    
    entry, err := m.cache.Get(ctx, key).Result()
    if err != nil {
        return nil, false
    }
    
    var cacheEntry CacheEntry
    if err := json.Unmarshal([]byte(entry), &cacheEntry); err != nil {
        return nil, false
    }
    
    // 更新命中次数
    cacheEntry.HitCount++
    m.cache.Set(ctx, key, cacheEntry, m.ttl)
    
    return cacheEntry.Result, true
}

// 写入缓存
func (m *CacheManager) Set(
    ctx context.Context,
    toolName string,
    arguments any,
    result any,
) error {
    key := m.buildKey(toolName, arguments)
    
    entry := CacheEntry{
        Key:      key,
        Result:   result,
        CachedAt: time.Now(),
        HitCount: 0,
    }
    
    data, _ := json.Marshal(entry)
    return m.cache.Set(ctx, key, data, m.ttl).Err()
}

// 构建缓存 key
func (m *CacheManager) buildKey(toolName string, arguments any) string {
    argsJSON, _ := json.Marshal(arguments)
    return fmt.Sprintf("ai:cache:%s:%s", toolName, string(argsJSON))
}
```

#### 集成到工具执行

```go
// internal/service/aiagent/run_service.go
func (s *Service) executeTool(
    ctx context.Context,
    toolCall ToolCall,
) (any, error) {
    // 1. 查询缓存
    if cached, ok := s.cache.Get(ctx, toolCall.Name, toolCall.Arguments); ok {
        return cached, nil
    }
    
    // 2. 执行工具
    result, err := s.toolExecutor.Execute(ctx, toolCall)
    if err != nil {
        return nil, err
    }
    
    // 3. 写入缓存（只缓存只读操作）
    if s.isReadOnly(toolCall.Name) {
        s.cache.Set(ctx, toolCall.Name, toolCall.Arguments, result)
    }
    
    return result, nil
}

// 判断是否只读操作
func (s *Service) isReadOnly(toolName string) bool {
    readOnlyTools := []string{"cluster", "postgresql", "redis", "middleware"}
    return slices.Contains(readOnlyTools, toolName)
}
```

---

## 6. 权限细粒度控制

### 6.1 问题描述

当前只有"允许"和"需要审批"两种权限，不够灵活。

### 6.2 设计方案

#### RBAC 权限模型

```go
// internal/models/ai_permission.go
type AIPermission struct {
    ID          string    `json:"id"`
    RoleID      string    `json:"role_id"`
    UserID      string    `json:"user_id"`
    
    // 权限规则
    Rules       []PermissionRule `json:"rules"`
    
    // 时间窗口
    TimeWindows []TimeWindow     `json:"time_windows"`
    
    // 资源范围
    ResourceScopes []ResourceScope `json:"resource_scopes"`
    
    CreatedAt   time.Time `json:"created_at"`
}

type PermissionRule struct {
    ToolName    string   `json:"tool_name"`     // cluster / postgresql / *
    Operations  []string `json:"operations"`    // list / get / logs / *
    Effect      string   `json:"effect"`        // allow / deny / require_approval
    Priority    int      `json:"priority"`      // 优先级（数字越大优先级越高）
}

type TimeWindow struct {
    StartHour  int      `json:"start_hour"`   // 9
    EndHour    int      `json:"end_hour"`     // 18
    Weekdays   []int    `json:"weekdays"`     // [1,2,3,4,5] 周一到周五
    AutoApprove  bool   `json:"auto_approve"` // 工作时间自动审批
}

type ResourceScope struct {
    Type       string   `json:"type"`       // pod / deployment / database
    Pattern    string   `json:"pattern"`    // web-* / db-*
    Namespaces []string `json:"namespaces"` // [default, production]
}
```

#### 权限检查器

```go
// internal/ai/permission/checker.go
type PermissionChecker struct {
    repo repo.IAIPermissionRepo
}

func (c *PermissionChecker) Check(
    ctx context.Context,
    userID string,
    toolCall ToolCall,
) (*CheckResult, error) {
    // 1. 加载用户权限
    permissions, err := c.repo.GetByUserID(ctx, userID)
    if err != nil {
        return nil, err
    }
    
    // 2. 检查时间窗口
    if !c.isWithinTimeWindow(permissions.TimeWindows) {
        return &CheckResult{
            Effect:   "require_approval",
            Reason:   "非工作时间，需要审批",
        }, nil
    }
    
    // 3. 检查资源范围
    if !c.isWithinResourceScope(permissions.ResourceScopes, toolCall) {
        return &CheckResult{
            Effect:   "deny",
            Reason:   "无权操作该资源",
        }, nil
    }
    
    // 4. 匹配权限规则
    for _, rule := range c.sortByPriority(permissions.Rules) {
        if c.matchesRule(rule, toolCall) {
            return &CheckResult{
                Effect:   rule.Effect,
                Reason:   fmt.Sprintf("匹配规则: %s", rule.ToolName),
            }, nil
        }
    }
    
    // 5. 默认拒绝
    return &CheckResult{
        Effect:   "deny",
        Reason:   "未找到匹配的权限规则",
    }, nil
}

// 检查时间窗口
func (c *PermissionChecker) isWithinTimeWindow(
    windows []TimeWindow,
) bool {
    now := time.Now()
    currentHour := now.Hour()
    currentWeekday := int(now.Weday())
    
    for _, window := range windows {
        if currentHour >= window.StartHour && 
           currentHour < window.EndHour &&
           slices.Contains(window.Weekdays, currentWeekday) {
            return true
        }
    }
    return false
}

// 检查资源范围
func (c *PermissionChecker) isWithinResourceScope(
    scopes []ResourceScope,
    toolCall ToolCall,
) bool {
    for _, scope := range scopes {
        if c.matchesScope(scope, toolCall) {
            return true
        }
    }
    return false
}
```

---

## 7. 主动推荐能力

### 7.1 问题描述

AI 只能被动回答问题，不能主动发现问题。

### 7.2 设计方案

#### 健康检查器

```go
// internal/ai/proactive/health_checker.go
type HealthChecker struct {
    scheduler  *cron.Cron
    tools      []tool.Tool
    alertRepo  repo.IAlertRepo
}

// 定时检查（每 5 分钟）
func (c *HealthChecker) Start() {
    c.scheduler.AddFunc("@every 5m", func() {
        c.checkAllClusters()
    })
}

func (c *HealthChecker) checkAllClusters() {
    ctx := context.Background()
    
    // 1. 检查 Pod 重启
    c.checkPodRestarts(ctx)
    
    // 2. 检查资源使用率
    c.checkResourceUsage(ctx)
    
    // 3. 检查数据库连接数
    c.checkDatabaseConnections(ctx)
    
    // 4. 检查 Redis 内存
    c.checkRedisMemory(ctx)
}

func (c *HealthChecker) checkPodRestarts(ctx context.Context) {
    // 调用 Cluster 工具查询所有 Pod
    result, _ := c.tools["cluster"].Execute(ctx, ClusterInput{
        Operation: "list",
        Kind:      "Pod",
    })
    
    // 分析重启次数
    for _, pod := range result.Pods {
        if pod.RestartCount > 3 {
            // 发送告警
            c.alertRepo.Save(ctx, &Alert{
                Type:      "pod_restart",
                Resource:  pod.Name,
                Message:   fmt.Sprintf("Pod %s 最近 1 小时重启 %d 次", pod.Name, pod.RestartCount),
                Severity:  "warning",
            })
        }
    }
}
```

#### 推荐引擎

```go
// internal/ai/proactive/recommender.go
type Recommender struct {
    alertRepo  repo.IAlertRepo
    historyRepo repo.IRunHistoryRepo
}

func (r *Recommender) GenerateRecommendations(
    ctx context.Context,
    userID string,
) []Recommendation {
    recommendations := []Recommendation{}
    
    // 1. 基于告警生成推荐
    alerts := r.alertRepo.GetActive(ctx)
    for _, alert := range alerts {
        recommendations = append(recommendations, Recommendation{
            Type:        "alert",
            Title:       alert.Message,
            Description: r.generateAlertDescription(alert),
            Actions:     r.generateAlertActions(alert),
        })
    }
    
    // 2. 基于历史问题生成推荐
    history := r.historyRepo.GetRecent(ctx, userID, 10)
    patterns := r.analyzePatterns(history)
    for _, pattern := range patterns {
        recommendations = append(recommendations, Recommendation{
            Type:        "pattern",
            Title:       fmt.Sprintf("发现重复问题: %s", pattern.Description),
            Description: "您最近多次查询了类似问题，建议...",
            Actions:     []string{"创建自动化脚本", "设置定时检查"},
        })
    }
    
    return recommendations
}
```

---

## 8. 知识库和 FAQ

### 8.1 问题描述

AI 缺乏企业内部知识和历史经验。

### 8.2 设计方案

#### 知识库模型

```go
// internal/models/ai_knowledge.go
type KnowledgeArticle struct {
    ID          string    `json:"id"`
    Title       string    `json:"title"`
    Content     string    `json:"content"`
    Tags        []string  `json:"tags"`
    Category    string    `json:"category"` // troubleshooting / best_practice / faq
    ViewCount   int       `json:"view_count"`
    HelpfulCount int      `json:"helpful_count"`
    CreatedAt   time.Time `json:"created_at"`
}

type KnowledgeSearchResult struct {
    Article  *KnowledgeArticle `json:"article"`
    Score    float64           `json:"score"` // 相关度分数
}
```

#### 知识库搜索

```go
// internal/ai/knowledge/searcher.go
type KnowledgeSearcher struct {
    db *gorm.DB
}

func (s *KnowledgeSearcher) Search(
    ctx context.Context,
    query string,
    topK int,
) ([]KnowledgeSearchResult, error) {
    // 1. 关键词搜索
    var articles []KnowledgeArticle
    s.db.Where("title LIKE ? OR content LIKE ? OR tags LIKE ?",
        "%"+query+"%", "%"+query+"%", "%"+query+"%").
        Limit(topK * 2).
        Find(&articles)
    
    // 2. 计算相关度分数
    results := []KnowledgeSearchResult{}
    for _, article := range articles {
        score := s.calculateRelevance(query, article)
        results = append(results, KnowledgeSearchResult{
            Article: &article,
            Score:   score,
        })
    }
    
    // 3. 按分数排序
    sort.Slice(results, func(i, j int) bool {
        return results[i].Score > results[j].Score
    })
    
    // 4. 返回 top K
    return results[:topK], nil
}
```

#### 集成到 AI 引擎

```go
// internal/service/aiagent/run_service.go
func (s *Service) CreateRun(
    ctx context.Context,
    actor Actor,
    conversationID string,
    req *request.AIAgentRunCreateRequest,
) (*response.AIAgentRunResponse, error) {
    // 1. 搜索知识库
    knowledge, _ := s.knowledgeSearcher.Search(ctx, req.Message, 3)
    
    // 2. 将知识注入到系统提示
    systemPrompt := s.buildSystemPromptWithKnowledge(knowledge)
    
    // 3. 调用大模型（带知识）
    response, err := s.einoRuntime.Send(ctx, messages, tools, systemPrompt)
    
    return response, err
}

func (s *Service) buildSystemPromptWithKnowledge(
    knowledge []KnowledgeSearchResult,
) string {
    prompt := s.baseSystemPrompt + "\n\n相关知识：\n"
    for i, k := range knowledge {
        prompt += fmt.Sprintf("\n[%d] %s\n%s\n", 
            i+1, k.Article.Title, k.Article.Content)
    }
    return prompt
}
```

---

## 9. 批量操作能力

### 9.1 问题描述

当前一次只能操作一个资源。

### 9.2 设计方案

#### 批量执行器

```go
// internal/ai/batch/executor.go
type BatchExecutor struct {
    maxConcurrency int  // 最大并发数（10）
}

type BatchOperation struct {
    ID          string       `json:"id"`
    ToolName    string       `json:"tool_name"`
    Targets     []Target     `json:"targets"`
    Strategy    string       `json:"strategy"` // parallel / sequential
    Status      string       `json:"status"`
    Results     []BatchResult `json:"results"`
}

type Target struct {
    Name       string `json:"name"`
    Namespace  string `json:"namespace"`
    Arguments  any    `json:"arguments"`
}

type BatchResult struct {
    Target  Target  `json:"target"`
    Result  any     `json:"result"`
    Error   error   `json:"error"`
}

func (e *BatchExecutor) Execute(
    ctx context.Context,
    operation *BatchOperation,
) error {
    operation.Status = "running"
    
    if operation.Strategy == "parallel" {
        // 并行执行
        sem := make(chan struct{}, e.maxConcurrency)
        var wg sync.WaitGroup
        
        for _, target := range operation.Targets {
            wg.Add(1)
            sem <- struct{}{} // 获取信号量
            
            go func(t Target) {
                defer wg.Done()
                defer func() { <-sem }() // 释放信号量
                
                result, err := e.executeTarget(ctx, operation.ToolName, t)
                operation.Results = append(operation.Results, BatchResult{
                    Target: t,
                    Result: result,
                    Error:  err,
                })
            }(target)
        }
        
        wg.Wait()
    } else {
        // 顺序执行
        for _, target := range operation.Targets {
            result, err := e.executeTarget(ctx, operation.ToolName, target)
            operation.Results = append(operation.Results, BatchResult{
                Target: target,
                Result: result,
                Error:  err,
            })
        }
    }
    
    operation.Status = "completed"
    return nil
}
```

---

## 10. 反馈和学习机制

### 10.1 问题描述

AI 不会从用户反馈中学习。

### 10.2 设计方案

#### 反馈模型

```go
// internal/models/ai_feedback.go
type AIFeedback struct {
    ID              string    `json:"id"`
    RunID           string    `json:"run_id"`
    UserID          string    `json:"user_id"`
    
    // 评分（1-5 星）
    Rating          int       `json:"rating"`
    
    // 反馈类型
    FeedbackType    string    `json:"feedback_type"` // helpful / incorrect / incomplete
    
    // 用户评论
    Comment         string    `json:"comment"`
    
    // 期望的答案
    ExpectedAnswer  string    `json:"expected_answer"`
    
    CreatedAt       time.Time `json:"created_at"`
}
```

#### 反馈收集

```typescript
// web/src/components/ai/FeedbackForm.tsx
function FeedbackForm({ runId }: { runId: string }) {
  const [rating, setRating] = useState(5)
  const [comment, setComment] = useState("")
  
  const submitFeedback = async () => {
    await aiSubmitFeedbackRequest({
      run_id: runId,
      rating: rating,
      comment: comment,
    })
    
    Message.success("感谢您的反馈！")
  }
  
  return (
    <div className="feedback-form">
      <h3>这个回答有帮助吗？</h3>
      
      <div className="rating">
        {[1, 2, 3, 4, 5].map((star) => (
          <button
            key={star}
            onClick={() => setRating(star)}
            className={star <= rating ? "active" : ""}
          >
            ★
          </button>
        ))}
      </div>
      
      <textarea
        placeholder="请描述您的反馈..."
        value={comment}
        onChange={(e) => setComment(e.target.value)}
      />
      
      <button onClick={submitFeedback}>提交反馈</button>
    </div>
  )
}
```

---

## 11. 实施路线图

### 第一阶段（1-2 个月）：基础能力

- ✅ 多轮对话上下文管理
- ✅ 可观测性和调试能力
- ✅ 容错和降级能力

### 第二阶段（2-3 个月）：增强能力

- 工具组合能力
- 缓存和复用机制
- 权限细粒度控制

### 第三阶段（3-4 个月）：智能能力

- 主动推荐能力
- 知识库和 FAQ
- 批量操作能力
- 反馈和学习机制

### 预期效果

完成所有功能后，AI 助手将具备：

1. **智能对话** - 理解上下文，支持追问
2. **自动规划** - 复杂问题自动分解执行
3. **主动发现** - 自动发现问题并推荐解决方案
4. **持续学习** - 从用户反馈中不断改进
5. **企业知识** - 融合企业内部知识和最佳实践

---

## 总结

这份设计方案涵盖了 AI 助手的 **10 个核心能力增强**，从基础的上下文管理到高级的智能推荐，每个功能都有详细的技术实现方案。

**建议按路线图分阶段实施**，优先解决最痛的问题（多轮对话、可观测性），再逐步增强智能能力。
