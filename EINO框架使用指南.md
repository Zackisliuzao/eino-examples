# Eino 框架使用指南

## 目录
1. [框架概述](#框架概述)
2. [核心概念](#核心概念)
3. [快速开始](#快速开始)
4. [Agent 开发](#agent-开发)
5. [工具集成](#工具集成)
6. [人机协作](#人机协作)
7. [多智能体系统](#多智能体系统)
8. [编排模式](#编排模式)
9. [状态管理](#状态管理)
10. [最佳实践](#最佳实践)

---

## 框架概述

Eino 是 CloudWeGo 团队开发的 AI 应用开发框架，提供了从简单到复杂的全栈式 AI 应用开发能力。框架分为三个层次：

### 架构层次

```
┌─────────────────────────────────────────────┐
│           ADK (应用开发套件)                  │
│  快速构建 Agent，内置最佳实践                 │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│         Compose (编排层)                     │
│  Graph/Chain 流程编排，灵活控制               │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│       Components (组件层)                    │
│  Model/Tool/Prompt 等基础组件                │
└─────────────────────────────────────────────┘
```

### 项目结构

```
eino-examples/
├── adk/                    # ADK 高级示例
│   ├── common/             # 公共工具和模型
│   ├── helloworld/         # 入门示例
│   ├── intro/              # 核心功能介绍
│   ├── human-in-the-loop/  # 人机协作
│   └── multiagent/         # 多智能体系统
├── compose/                # 编排示例
│   ├── chain/              # 链式流程
│   ├── graph/              # 图编排
│   └── workflow/           # 工作流
├── components/             # 组件扩展
└── quickstart/             # 快速开始
```

---

## 核心概念

### 1. Agent（智能体）

Agent 是 Eino 中的核心概念，代表一个能够执行任务、调用工具、与人交互的 AI 实体。

**核心特性：**
- **指令驱动**：通过 Instruction 定义行为
- **工具调用**：集成外部工具和 API
- **状态管理**：维护对话历史和中间状态
- **可观测性**：内置事件流和回调机制

### 2. Runner（运行器）

Runner 负责执行 Agent，管理生命周期和状态。

**核心功能：**
- 流式和非流式输出
- 检查点（Checkpoint）持久化
- 中断恢复机制
- 事件回调

### 3. Tool（工具）

工具是 Agent 与外部世界交互的接口。

**工具类型：**
- **基础工具**：单个函数调用
- **图工具**：封装完整的工作流
- **工具包装器**：添加中间件逻辑（审批、编辑等）

### 4. Compose（编排）

用于构建复杂的工作流和流程。

**编排模式：**
- **Chain**：线性步骤序列
- **Graph**：带分支和循环的有向图
- **Workflow**：状态机驱动的流程

---

## 快速开始

### 环境准备

```bash
# 克隆项目
git clone https://github.com/cloudwego/eino-examples.git
cd eino-examples

# 设置环境变量
export MODEL_TYPE=openai  # 或 ark, ollama
export OPENAI_API_KEY=your_api_key
export OPENAI_BASE_URL=https://api.openai.com/v1
```

### Hello World

**创建简单的对话 Agent：**

```go
package main

import (
    "context"
    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino-examples/adk/common/model"
)

func main() {
    ctx := context.Background()

    // 创建 Agent
    agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "HelloBot",
        Description: "A simple greeting bot",
        Instruction: "You are a helpful assistant. Greet users warmly.",
        Model:       model.NewChatModel(),
    })
    if err != nil {
        panic(err)
    }

    // 创建 Runner
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        Agent:           agent,
        EnableStreaming: true,
    })

    // 运行对话
    err = runner.Run(ctx, "Hello!")
    if err != nil {
        panic(err)
    }
}
```

**运行：**

```bash
go run adk/helloworld/main.go
```

---

## Agent 开发

### 基础 Agent

使用 `ChatModelAgent` 是最快的方式：

```go
agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "MyAgent",
    Description: "Agent description",
    Instruction: `
        You are a helpful assistant.
        Follow these guidelines:
        1. Be concise
        2. Use tools when available
        3. Ask for clarification when needed
    `,
    Model: model.NewChatModel(),
})
```

### 带工具的 Agent

```go
import "github.com/cloudwego/eino/components/tool/utils"

// 定义工具输入结构
type SearchInput struct {
    Query string `json:"query"`
}

// 创建工具
searchTool, err := utils.InferTool(
    "Search",
    "Search the web for information",
    func(ctx context.Context, input SearchInput) (string, error) {
        // 实现搜索逻辑
        return fmt.Sprintf("Results for: %s", input.Query), nil
    },
)

// 创建带工具的 Agent
agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "SearchAgent",
    Instruction: "You help users search for information",
    Model:       model.NewChatModel(),
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{searchTool},
        },
    },
})
```

### 自定义 Agent

实现 `Agent` 接口以完全自定义行为：

```go
type CustomAgent struct {
    // 自定义字段
}

func (a *CustomAgent) Run(
    ctx context.Context,
    input []adk.Message,
    opts ...adk.AgentOption,
) (*adk.AgentEvent, error) {
    // 自定义逻辑
    return &adk.AgentEvent{}, nil
}
```

### Agent 配置选项

**ChatModelAgentConfig 完整参数：**

| 参数 | 类型 | 必需 | 说明 |
|-----|------|------|------|
| Name | string | 是 | Agent 名称 |
| Description | string | 否 | Agent 描述 |
| Instruction | string | 是 | 系统指令 |
| Model | model.ToolCallingChatModel | 是 | 聊天模型 |
| ToolsConfig | ToolsConfig | 否 | 工具配置 |
| ToolCallingPrompt | string | 否 | 工具调用提示 |
| Middlewares | []AgentMiddleware | 否 | 中间件 |
| MaxSteps | int | 否 | 最大执行步数 |

---

## 工具集成

### 基础工具

使用 `InferTool` 从函数自动生成工具：

```go
type CalculatorInput struct {
    Expression string `json:"expression"`
}

calcTool, err := utils.InferTool(
    "Calculator",
    "Evaluate mathematical expressions",
    func(ctx context.Context, input CalculatorInput) (string, error) {
        result, err := eval(input.Expression)
        return fmt.Sprintf("Result: %v", result), err
    },
)
```

### Graph 工具

将整个工作流封装为工具：

```go
// 创建子图
subGraph := compose.NewGraph[Input, Output]()
subGraph.AddLambdaNode("Process", ...)
subGraph.AddEdge(START, "Process")
subGraph.AddEdge("Process", END)

// 编译为工具
graphTool := tool.NewInvokableGraphTool(
    subGraph,
    "MyTool",
    "Description of what this tool does",
    compileOptions...,
)
```

### 工具包装器（中间件）

#### 1. 审批包装器

需要用户批准后执行工具：

```go
import "github.com/cloudwego/eino-examples/adk/common/tool"

approvedTool := &tool.InvokableApprovableTool{
    InvokableTool: originalTool,
}
```

**工作流程：**
1. Agent 调用工具
2. 工具中断，显示参数给用户
3. 用户批准/拒绝
4. 根据反馈执行或返回

#### 2. 编辑包装器

用户可以修改工具参数：

```go
editableTool := &tool.InvokableReviewEditWrapper{
    InvokableTool: originalTool,
}
```

#### 3. JSON 修复包装器

自动修复错误的 JSON 参数：

```go
jsonRepairTool := &tool.JsonRepairToolWrapper{
    InvokableTool: originalTool,
}
```

### 自定义工具包装器

```go
type LoggingWrapper struct {
    tool.InvokableTool
}

func (w *LoggingWrapper) InvokableRun(
    ctx context.Context,
    argumentsInJSON string,
    opts ...tool.Option,
) (string, error) {
    log.Printf("Tool called with: %s", argumentsInJSON)

    result, err := w.InvokableTool.InvokableRun(ctx, argumentsInJSON, opts...)

    log.Printf("Tool returned: %s", result)
    return result, err
}
```

---

## 人机协作

### 1. 审批流程

**场景：** 敏感操作需要人工确认

```go
// 工具定义
transferTool, _ := utils.InferTool(
    "TransferMoney",
    "Transfer money to another account",
    transferMoneyHandler,
)

// 包装为需要审批的工具
approvedTool := &tool.InvokableApprovableTool{
    InvokableTool: transferTool,
}

agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "BankAgent",
    Instruction: "Help users with banking operations",
    Model:       model.NewChatModel(),
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{approvedTool},
        },
    },
})
```

**用户交互：**

```
User: Transfer $1000 to account 123456

Agent: tool 'TransferMoney' interrupted with arguments '{"amount": 1000, "to": "123456"}',
       waiting for your approval, please answer with Y/N

User: Y

Agent: Transfer successful
```

### 2. 审查和编辑

**场景：** 用户可以修改工具参数

```go
editableTool := &tool.InvokableReviewEditWrapper{
    InvokableTool: sensitiveOperationTool,
}
```

**交互流程：**

```
User: Execute operation with parameters

Agent: [Review requested]
       Arguments: {"param": "value"}
       Press Enter to accept, or edit:

User: {"param": "modified_value"}

Agent: Executing with modified parameters...
```

### 3. 多轮反馈

**场景：** 迭代改进

```go
// 使用 FollowUpTool 收集反馈
feedbackTool := follow_up_tool.NewInvokableFollowUpTool(
    originalTool,
    &follow_up_tool.Config{
        CollectPositive:  true,
        CollectNegative:  true,
        MaxRounds:        3,
    },
)
```

**流程：**

```
1. 工具执行
2. 收集用户反馈（正面/负面）
3. 根据反馈调整
4. 重复（最多 MaxRounds 次）
```

### 4. 监督模式

**场景：** 人工监督多智能体系统

```go
supervisorAgent := adk.NewSupervisorAgent(ctx, &adk.SupervisorAgentConfig{
    Name:        "Supervisor",
    Instruction: "Coordinate worker agents",
    Workers:     []adk.Agent{worker1, worker2},
    HumanMode:   true,  // 启用人工监督
})
```

---

## 多智能体系统

### 1. 计划-执行-再规划

```go
plannerAgent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "Planner",
    Instruction: "Create a plan for the user's request",
    Model:       model.NewChatModel(),
})

executorAgent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "Executor",
    Instruction: "Execute the plan steps",
    Model:       model.NewChatModel(),
    Tools:       []tool.BaseTool{/* 工具列表 */},
})

// 使用工作流编排
wf := compose.NewWorkflow("PlanExecuteReplan")
wf.AddAgentNode("Plan", plannerAgent)
wf.AddAgentNode("Execute", executorAgent)
wf.AddEdge(compose.START, "Plan")
wf.AddEdge("Plan", "Execute")
wf.AddEdge("Execute", "Plan", compose.WithEdgeCondition("needs_replanning"))
wf.AddEdge("Execute", compose.END)
```

### 2. 监督模式

```go
supervisor := adk.NewSupervisorAgent(ctx, &adk.SupervisorAgentConfig{
    Name:        "ProjectManager",
    Instruction: `
        You manage a team of specialist agents:
        - Researcher: gathers information
        - Developer: writes code
        - Tester: validates results
        Assign tasks to the appropriate agent.
    `,
    Workers: []adk.Agent{
        researcherAgent,
        developerAgent,
        testerAgent,
    },
})
```

**交互示例：**

```
User: Build a web scraper

Supervisor: I'll assign this to our Developer

Developer: [creates scraper code]

Supervisor: Now let's have the Tester validate it

Tester: [runs tests and reports issues]

Supervisor: Developer, please fix these issues

...
```

### 3. 分层监督

```go
// 第一层：技术负责人
techLead := adk.NewSupervisorAgent(ctx, &adk.SupervisorAgentConfig{
    Name: "TechLead",
    Workers: []adk.Agent{
        frontendAgent,
        backendAgent,
        dbaAgent,
    },
})

// 第二层：项目经理
projectManager := adk.NewSupervisorAgent(ctx, &adk.SupervisorAgentConfig{
    Name: "ProjectManager",
    Workers: []adk.Agent{
        techLead,
        designerAgent,
        qaAgent,
    },
})
```

### 4. 集成项目管理器

实际场景：复杂的项目管理流程

```go
// 来自 adk/multiagent/integration_project_manager/
// 完整的多智能体协作系统
```

---

## 编排模式

### Chain 编排

线性步骤序列：

```go
chain := compose.NewChain[map[string]any, string]()

chain.AppendChatTemplate(prompt.FromMessages(schema.FString,
    schema.SystemMessage("You are a helpful assistant"),
    schema.UserMessage("{{.input}}"),
))

chain.AppendChatModel(model.NewChatModel())

chain.AppendLambda(compose.StraightLambdaFunc(
    func(ctx context.Context, msg *schema.Message) (string, error) {
        return msg.Content, nil
    },
))

compiledChain, err := chain.Compile(ctx)
result, err := compiledChain.Invoke(ctx, map[string]any{"input": "Hello"})
```

### Graph 编排

带分支的有向图：

```go
g := compose.NewGraph[Input, Output]()

// 添加节点
g.AddLambdaNode("parse", parseInput)
g.AddChatModelNode("classify", classifier)
g.AddLambdaNode("route", router)
g.AddLambdaNode("handle_a", handleA)
g.AddLambdaNode("handle_b", handleB)

// 添加边
g.AddEdge(compose.START, "parse")
g.AddEdge("parse", "classify")
g.AddEdge("classify", "route")

// 条件分支
g.AddBranch("route", compose.NewBranch(
    "result",
    map[string]compose.Output{"a": "handle_a", "b": "handle_b"},
))

g.AddEdge("handle_a", compose.END)
g.AddEdge("handle_b", compose.END)

// 编译
graph, err := g.Compile(ctx)
```

### Workflow 编排

状态机驱动的流程：

```go
wf := compose.NewWorkflow("ticket_workflow")

wf.AddChatModelNode("triage", triageAgent)
wf.AddChatModelNode("investigate", investigateAgent)
wf.AddChatModelNode("resolve", resolveAgent)

wf.AddEdge(compose.START, "triage")
wf.AddEdge("triage", "investigate",
    compose.WithEdgeCondition("needs_investigation"))
wf.AddEdge("triage", compose.END,
    compose.WithEdgeCondition("resolved"))
wf.AddEdge("investigate", "resolve")
wf.AddEdge("resolve", compose.END)

runner, err := adk.NewRunner(ctx, adk.RunnerConfig{
    Agent: wf,
})
```

---

## 状态管理

### 检查点（Checkpoint）

持久化 Agent 状态，支持断点续传：

```go
// 内存存储
store := store.NewInMemoryStore()

// 或使用数据库存储
store := store.NewRedisStore(redisClient)

runner := adk.NewRunner(ctx, adk.RunnerConfig{
    Agent:           agent,
    CheckPointStore: store,
})

// 运行时会自动保存检查点
err := runner.Run(ctx, "User message", adk.WithRunID("conversation_1"))

// 从检查点恢复
err = runner.Run(ctx, "Another message", adk.WithRunID("conversation_1"))
```

### 中断和恢复

```go
// 在特定节点前中断
runner, _ := adk.NewGraphRunner(ctx, g,
    compose.WithInterruptBeforeNodes([]string{"critical_step"}),
)

// 运行到中断点
err := runner.Run(ctx, input)

// 用户提供数据后恢复
resumeData := map[string]any{"approval": true}
err = runner.Resume(ctx, resumeData)
```

### 对话历史压缩

自动压缩长对话以节省 token：

```go
import "github.com/cloudwego/eino-examples/adk/intro/agent_with_summarization/summarization"

middleware, _ := summarization.New(ctx, &summarization.Config{
    Model:                  model.NewChatModel(),
    MaxTokensBeforeSummary: 4000,  // 超过 4000 token 时触发
    MaxTokensForRecentMessages: 1000,  // 保留最近 1000 token
})

agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "SummaryAgent",
    Instruction: "You are a helpful assistant",
    Model:       model.NewChatModel(),
    Middlewares: []adk.AgentMiddleware{middleware},
})
```

---

## 最佳实践

### 1. Agent 设计

**DO:**
- 使用清晰的 Instruction
- 为 Agent 选择合适的工具
- 设置合理的 MaxSteps 防止无限循环
- 使用中间件处理横切关注点

**DON'T:**
- 不要让 Agent 做太多事情
- 不要忽视错误处理
- 不要过度依赖流式输出

### 2. 工具开发

**DO:**
- 使用结构化输入（不要用 `any`）
- 提供清晰的工具描述
- 实现适当的错误处理
- 考虑使用包装器增强功能

**DON'T:**
- 不要在工具中硬编码敏感信息
- 不要忽略工具参数验证

### 3. 状态管理

**DO:**
- 生产环境使用持久化检查点存储
- 合理设置中断点
- 定期清理旧的检查点

**DON'T:**
- 不要在内存存储中保存关键数据
- 不要忽视并发问题

### 4. 性能优化

```go
// 使用流式输出提升响应速度
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    Agent:           agent,
    EnableStreaming: true,
})

// 使用并发执行
wf.AddParallelNodes([]string{"task1", "task2", "task3"})

// 启用对话压缩
middleware, _ := summarization.New(ctx, cfg)
```

### 5. 可观测性

```go
// 添加全局回调
callbacks.AppendGlobalHandlers(
    logging.NewLoggerHandler(),
    metrics.NewMetricsHandler(),
)

// 节点特定回调
g.AddChatModelNode("model", cm,
    compose.WithStatePreHandler(myPreHandler),
    compose.WithStatePostHandler(myPostHandler),
)

// 事件流处理
for event := range runner.RunStream(ctx, message) {
    // 处理事件
    fmt.Printf("Event: %+v\n", event)
}
```

---

## 高级主题

### 1. 自定义回调处理器

```go
type MyHandler struct{}

func (h *MyHandler) OnStart(ctx context.Context, info *callbacks.RunInfo) context.Context {
    log.Printf("Starting: %s", info.Name)
    return ctx
}

func (h *MyHandler) OnEnd(ctx context.Context, info *callbacks.RunInfo) context.Context {
    log.Printf("Completed: %s", info.Name)
    return ctx
}

func (h *MyHandler) OnError(ctx context.Context, err error, info *callbacks.RunInfo) context.Context {
    log.Printf("Error in %s: %v", info.Name, err)
    return ctx
}
```

### 2. 多模型支持

```go
func getModel() model.ChatModel {
    modelType := os.Getenv("MODEL_TYPE")

    switch modelType {
    case "ark":
        return model.NewArkChatModel()
    case "ollama":
        return model.NewOllamaChatModel()
    default:
        return model.NewOpenAIChatModel()
    }
}
```

### 3. 测试策略

```go
// 使用 Mock 模型
mockModel := &mock.MockChatModel{
    ResponseFunc: func(ctx context.Context, msgs []*schema.Message) (*schema.Message, error) {
        return schema.AssistantMessage("Mock response"), nil
    },
}

// 使用 Mock 工具
mockTool := &mock.MockTool{
    RunFunc: func(ctx context.Context, args string) (string, error) {
        return "Mock tool result", nil
    },
}
```

---

## 示例项目

### 完整示例：客户服务系统

```go
package main

import (
    "context"
    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/components/tool/utils"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino-examples/adk/common/model"
    tool2 "github.com/cloudwego/eino-examples/adk/common/tool"
)

func main() {
    ctx := context.Background()

    // 定义工具
    queryOrder, _ := utils.InferTool("QueryOrder", "Query order status", queryHandler)
    refundOrder, _ := utils.InferTool("RefundOrder", "Process refund", refundHandler)

    // 退款需要审批
    approvedRefund := &tool2.InvokableApprovableTool{
        InvokableTool: refundOrder,
    }

    // 创建客服 Agent
    agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "CustomerService",
        Instruction: `
            You are a customer service agent.
            Help customers with:
            - Order inquiries (use QueryOrder)
            - Refunds (use RefundOrder, requires approval)
            - General questions
        `,
        Model: model.NewChatModel(),
        ToolsConfig: adk.ToolsConfig{
            ToolsNodeConfig: compose.ToolsNodeConfig{
                Tools: []tool.BaseTool{queryOrder, approvedRefund},
            },
        },
    })

    // 创建带检查点的 Runner
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        Agent:           agent,
        EnableStreaming: true,
        CheckPointStore: store.NewInMemoryStore(),
    })

    // 运行
    runner.Run(ctx, adk.UserMessage("I want a refund for order #12345"))
}
```

---

## 资源链接

- **GitHub**: [cloudwego/eino](https://github.com/cloudwego/eino)
- **示例**: [cloudwego/eino-examples](https://github.com/cloudwego/eino-examples)
- **文档**: [Eino Documentation](https://www.cloudwego.io/docs/eino/)

---

## 总结

Eino 框架提供了完整的 AI 应用开发能力：

1. **ADK 层**：快速构建 Agent，适合大多数场景
2. **Compose 层**：灵活编排复杂流程
3. **Components 层**：可扩展的组件生态

推荐学习路径：
1. 从 `adk/helloworld` 开始
2. 学习 `adk/intro` 中的核心功能
3. 探索 `adk/human-in-the-loop` 了解人机协作
4. 研究 `adk/multiagent` 掌握多智能体系统
5. 深入 `compose` 理解底层编排机制

Happy coding with Eino!
