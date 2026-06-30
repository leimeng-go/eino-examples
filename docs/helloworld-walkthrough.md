# helloworld 示例逐段讲解

> 对应代码：[adk/helloworld/helloworld.go](../adk/helloworld/helloworld.go)

本示例是 ADK 最简单的入门用例，展示了"创建模型 → 创建 Agent → 创建 Runner → 执行对话"的完整闭环。

---

## 1. 导入依赖

```go
import (
    "github.com/cloudwego/eino/adk"          // ADK 核心：Agent、Runner、Message
    "github.com/cloudwego/eino/schema"        // 消息 schema：UserMessage 等
    "github.com/cloudwego/eino-ext/components/model/openai"  // OpenAI 兼容模型实现
)
```

| 包 | 作用 |
|----|------|
| `eino/adk` | 提供 `ChatModelAgent`、`Runner`、`Message` 等核心抽象 |
| `eino/schema` | 提供消息构造便捷函数（`UserMessage`、`SystemMessage` 等） |
| `eino-ext/.../openai` | 提供 OpenAI 协议兼容的 `ChatModel` 实现 |

> `eino-ext` 是 Eino 的扩展库，还支持 ark、ollama、deepseek 等模型，只需替换 import 和 config 即可切换。

---

## 2. 初始化模型

```go
model, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
    APIKey:  os.Getenv("OPENAI_API_KEY"),
    Model:   os.Getenv("OPENAI_MODEL"),
    BaseURL: os.Getenv("OPENAI_BASE_URL"),
    ByAzure: func() bool {
        return os.Getenv("OPENAI_BY_AZURE") == "true"
    }(),
})
```

- `NewChatModel` 返回的对象实现了 eino 的 `model.ChatModel` 接口
- `BaseURL` 让你可以接入任何 OpenAI 协议兼容的服务（Ark、DeepSeek、本地 vLLM 等）
- 环境变量从 `.env` 或系统环境读取，不硬编码到代码中

---

## 3. 创建 ChatModelAgent

```go
agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "hello_agent",
    Description: "A friendly greeting assistant",
    Instruction: "You are a friendly assistant. Please respond to the user in a warm tone.",
    Model:       model,
})
```

### 三个关键字段

| 字段 | 作用 | 对应 LLM 调用中的位置 |
|------|------|----------------------|
| `Instruction` | 系统 prompt，定义 Agent 人格/行为规则 | 拼入对话的第一条 `system` 消息 |
| `Name` | Agent 标识符，多 Agent 场景下用于路由/转交 | 元数据 |
| `Description` | 描述 Agent 擅长什么，supervisor 模式下用它决定何时派发 | 元数据 |

### NewChatModelAgent 内部做了什么

1. 把 `Instruction` 固化为系统消息
2. 把 `Model` 包装成内部可调用的 ChatModel
3. 构建 Agent 的 ReAct 循环（配了 Tools 就会自动循环调用工具）
4. 返回实现了 `adk.Agent` 接口的对象

> 本示例没配 `Tools`，所以 Agent 只做单轮对话，不会进入工具调用循环。

---

## 4. 创建 Runner

```go
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    Agent:           agent,
    EnableStreaming: true,
})
```

- `EnableStreaming: true` → 以事件流方式逐步返回结果（打字机效果）
- `EnableStreaming: false` → 一次性返回完整结果

Runner 是 Agent 的执行引擎，负责驱动 Agent 运行、生成结构化事件、处理中断/恢复。

---

## 5. 构造输入并执行

```go
input := []adk.Message{
    schema.UserMessage("你好介绍一下你自已."),
}

events := runner.Run(ctx, input)
```

- `adk.Message` 是 ADK 对消息的统一抽象
- `schema.UserMessage()` 是便捷构造函数，返回值兼容 `adk.Message`
- `Run()` 返回一个**事件迭代器**，不是阻塞返回单个结果

---

## 6. 消费事件流

```go
events := runner.Run(ctx, input)
for {
    event, ok := events.Next()    // 取下一个事件
    if !ok {
        break                       // 事件流结束
    }

    if event.Err != nil {         // 错误事件
        log.Printf("错误: %v", event.Err)
        break
    }

    if msg, err := event.Output.MessageOutput.GetMessage(); err == nil {
        fmt.Printf("Agent: %s\n", msg.Content)
    }
}
```

- `events.Next()` 逐个取出事件，返回 `(event, ok)`，`ok=false` 表示流结束
- 先检查 `event.Err` 处理错误
- `event.Output.MessageOutput.GetMessage()` 从事件中提取消息内容
- 本示例无 Tools，所以只会产生 `MessageOutput` 类型的事件

---

## 完整执行流程图

```
用户输入 "你好介绍一下你自已."
        │
        ▼
   ┌─────────┐
   │ Runner  │  驱动执行
   └────┬────┘
        │
        ▼
┌───────────────────┐
│ ChatModelAgent    │
│ 1. 拼 SystemMsg   │  ← Instruction
│ 2. 拼 UserMsg     │  ← 用户输入
│ 3. 调用 LLM       │  ← model.Generate()
│ 4. 无工具→直接返回│
└────────┬──────────┘
         │
         ▼
    事件流（events）
   ├── Event 1: MessageOutput（第一个 chunk）
   ├── Event 2: MessageOutput（第二个 chunk）
   ├── ...
   └── Event N: 结束
         │
         ▼
   for 循环消费 → fmt.Printf 逐片打印
```

---

## 与更复杂示例的差异

本示例是最小闭环——单轮、无工具、无中间件。真实场景会在 `ChatModelAgentConfig` 中添加：

```go
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    // ... 基础配置 ...
    Tools:     []adk.Tool{...},           // 工具调用 → 触发 ReAct 循环
    SubAgents: []adk.Agent{...},          // 子 Agent → 支持 transfer 转交
    Middlewares: []adk.Middleware{...},   // 中间件 → 摘要/文件系统等
})
```

加了 `Tools` 后，Agent 会自动进入"调用 LLM → 判断是否调工具 → 执行工具 → 再调 LLM"的循环，Runner 会发出 `ToolCall`、`ToolResult` 等更多类型的事件。

参见：
- 带工具的示例：`adk/agentic/research_assistant/`
- 带子 Agent 的示例：`adk/intro/transfer/`
- 带中间件的示例：`adk/intro/agent_with_summarization/`
