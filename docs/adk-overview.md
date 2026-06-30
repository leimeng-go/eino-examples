# Eino ADK 架构总览

## 什么是 ADK

ADK（Agent Development Kit）是 Eino 框架之上的应用层 SDK，用于构建和编排 AI Agent 应用。它将底层 Graph/Chain 能力封装为"Agent 即节点"的开发体验，让开发者专注于编排逻辑而非底层图编排细节。

### 分层关系

```
┌──────────────────────────────────────────────────┐
│              你的应用代码                          │
├──────────────────────────────────────────────────┤
│  eino/adk         ← ADK：Agent、Runner、Session    │
│  (上层应用框架)      Middleware、Prebuilt 编排模式  │
├──────────────────────────────────────────────────┤
│  eino             ← 底层框架：ChatModel 接口        │
│  (底层框架)         Graph/Chain 编排能力            │
├──────────────────────────────────────────────────┤
│  eino-ext         ← 扩展库：LLM 厂商的具体实现      │
│  (扩展组件)         (openai/ark/ollama 等)         │
└──────────────────────────────────────────────────┘
```

- `eino` 提供核心接口定义（如 `model.ChatModel` 接口）和底层编排能力（Graph/Chain/Workflow）
- `eino-ext` 提供各 LLM 厂商的具体实现（openai、ark、ollama 等），返回的对象实现了 eino 的接口
- `eino/adk` 在上层封装出 Agent、Runner 等应用级概念

---

## 核心组件

### 1. ChatModelAgent

基于 LLM 的对话型 Agent，是 ADK 中最常用的 Agent 类型。

```go
agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "hello_agent",            // Agent 标识符
    Description: "A friendly assistant",   // 描述（supervisor 模式下用于路由）
    Instruction: "You are a friendly...",  // 系统 prompt（转成 system message）
    Model:       model,                    // LLM 后端（实现 model.ChatModel 接口）
    Tools:       []adk.Tool{...},         // 工具列表（可选，配了后自动进入 ReAct 循环）
    SubAgents:   []adk.Agent{...},        // 子 Agent（可选，用于 transfer 转交）
    Middlewares: []adk.Middleware{...},   // 中间件（可选，如摘要/文件系统）
})
```

**内部工作机制：**
1. 把 `Instruction` 固化为系统消息
2. 把 `Model` 包装为内部可调用的 ChatModel
3. 构建 Agent 的 ReAct 循环（如果配了 Tools 就会自动循环调用工具）
4. 返回实现了 `adk.Agent` 接口的对象

### 2. Runner

Agent 的执行引擎，负责驱动 Agent 运行并生成事件流。

```go
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    Agent:           agent,
    EnableStreaming: true,   // true=流式事件，false=一次性返回
})
```

**Runner 的职责：**
- 驱动 Agent 的循环（调用 LLM → 判断是否调工具 → 再调 LLM → ...）
- 生成结构化事件（Event）
- 处理中断/恢复（human-in-the-loop 场景）

### 3. Event（事件）

Runner 以事件流方式返回结果。主要事件类型：

| 事件类型 | 说明 |
|----------|------|
| `MessageOutput` | 模型回复消息（流式分片） |
| `ToolCall` | Agent 决定调用工具 |
| `ToolResult` | 工具执行完成，返回结果 |
| `Error` | 执行出错 |

事件结构示意：

```
Event
├── Type      // 事件类型
├── Output
│   ├── MessageOutput  // 消息输出
│   └── ToolOutput     // 工具调用结果
├── Err        // 错误信息
└── ...        // 其他元数据（AgentName, State 等）
```

### 4. Message（消息）

ADK 对消息的统一抽象，比 `schema.Message` 多了 ADK 特有的元信息。

```go
input := []adk.Message{
    schema.UserMessage("你好"),       // 便捷构造函数
    schema.SystemMessage("你是助手"),   // 返回值兼容 adk.Message
}
```

### 5. Session（会话）

用于跨 Agent 传递数据与状态。

```
Agent A ──写入 Session──→ Agent B 读取
```

通过 Session，多个 Agent 可以共享中间结果、用户偏好等上下文信息，而无需显式传递。对应示例：`adk/intro/session/`。

---

## 预置编排模式（Prebuilt）

ADK 在 `adk/prebuilt` 包中提供了几种开箱即用的多 Agent 编排模式。

### Supervisor（主管模式）

```go
import "github.com/cloudwego/eino/adk/prebuilt/supervisor"
```

一个中心 Supervisor Agent 接收请求，根据子 Agent 的 `Description` 决定派发给谁：

```
         ┌────────────┐
         │ Supervisor │  ← 接收请求，决定派给谁
         └─────┬──────┘
      ┌────────┼────────┐
      ▼        ▼        ▼
  ┌──────┐ ┌──────┐ ┌──────┐
  │搜索  │ │代码  │ │写作  │
  │Agent │ │Agent │ │Agent │
  └──────┘ └──────┘ └──────┘
```

示例：`adk/multiagent/supervisor/`、`adk/multiagent/layered-supervisor/`

### Plan-Execute-Replan（计划-执行-重规划）

```go
import "github.com/cloudwego/eino/adk/prebuilt/planexecute"
```

```
1. Planner  → 拆解任务为步骤列表
2. Executor → 逐步执行
3. Replanner → 根据结果决定是否修改计划
4. 循环直到完成
```

示例：`adk/multiagent/plan-execute-replan/`

### Deep Agents

```go
import "github.com/cloudwego/eino/adk/prebuilt/deep"
```

结合文件系统状态管理的深度 Agent 模式。示例：`adk/multiagent/deep/`

---

## Middleware 机制

ADK 提供中间件机制，可在 Agent 执行流程中插入处理逻辑。

| 中间件 | 包路径 | 功能 |
|--------|--------|------|
| Filesystem | `adk/middlewares/filesystem` | 文件系统状态持久化 |
| Summarization | `adk/middlewares/summarization` | 长对话自动摘要 |
| Skill | `adk/middlewares/skill` | 从文件系统加载 Agent 技能 |
| Dynamic Tool | `adk/middlewares/dynamictool/toolsearch` | 动态检索并注入工具 |

使用方式：
```go
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Middlewares: []adk.Middleware{
        summarization.New(...),
        fsmiddleware.New(...),
    },
})
```

---

## 示例目录速查

| 目录 | 说明 |
|------|------|
| `adk/helloworld` | 最简单的单轮对话 Agent |
| `adk/intro/chatmodel` | ChatModelAgent + Interrupt |
| `adk/intro/custom` | 自定义 Agent 实现 |
| `adk/intro/workflow` | Loop / Parallel / Sequential Agent |
| `adk/intro/session` | Session 跨 Agent 传数据 |
| `adk/intro/transfer` | Agent 间任务转交 |
| `adk/intro/agent_with_summarization` | 对话摘要中间件 |
| `adk/intro/http-sse-service` | HTTP SSE 服务暴露 |
| `adk/agentic` | AgenticModel：研究助手、token 截断重试 |
| `adk/agent/ralph-loop` | 自主迭代循环 + 文件系统状态 |
| `adk/human-in-the-loop` | 8 种人在回路模式 |
| `adk/multiagent` | Supervisor / Plan-Execute / Deep / Excel Agent |
| `adk/middlewares` | Skill 中间件、动态工具检索 |
| `adk/common/tool/graphtool` | Graph/Chain/Workflow 封装为 Agent 工具 |
| `adk/cancel/graceful-exit` | 安全取消和恢复嵌套 Agent |
