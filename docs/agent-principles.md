# AI Agent 原理知识

本文档系统讲解 AI Agent 的核心原理，并结合 Eino ADK 说明这些原理在代码中的实现方式。

---

## 一、什么是 AI Agent

### 传统 LLM 调用（单轮）

```
用户提问 -> LLM -> 一次性回答
```

LLM 只能基于自身训练知识回答，不能执行操作、不能获取实时信息、不能多步推理。

### AI Agent（多步循环）

```
用户提问 -> LLM 思考 -> 决定调用工具 -> 执行工具 -> 拿到结果 -> LLM 再思考 -> ... -> 最终回答
```

Agent = **LLM 大脑** + **工具手脚** + **记忆** + **循环执行引擎**

核心区别：Agent 能**自主决策"下一步做什么"**，并在循环中不断逼近目标。

---

## 二、ReAct 模式（Agent 的核心范式）

ReAct = **Reasoning + Acting**，是目前主流 Agent 的基础范式。

### 执行过程示例

```
Thought:     用户想查上海明天天气，我需要调用天气工具
Action:      调用 get_weather(city="上海", date="tomorrow")
Observation: 工具返回 {"temp": 28, "weather": "晴"}
Thought:     拿到结果了，28度晴天不需要带伞
Final Answer: 上海明天28度晴天，不需要带伞
```

### 循环结构

```
        +-------------------------+
        |                         |
        v                         |
   +---------+                    |
   | Thought |<-- LLM 推理        |
   +----+----+                    |
        |                         |
        v                         |
   +---------+                    |
   | Action  |<-- 调用工具        |
   +----+----+                    |
        |                         |
        v                         |
   +-----------+  否: 还需要继续  |
   |Observation+------------------+
   +-----+-----+
         | 是: 已得出结论
         v
   +----------+
   |  Answer  |
   +----------+
```

### 在 ADK 中的实现

`ChatModelAgent` 内部就是 ReAct 循环。helloworld 示例没配 Tools，所以只走了一轮就结束（LLM 直接回答）。配了 Tools 后，Agent 会自动进入多轮循环。

```go
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Tools: []adk.Tool{...},  // 配了工具后自动进入 ReAct 循环
})
```

---

## 三、Tool Calling（工具调用）原理

### 1. 工具是怎么"告诉"LLM 的

LLM 本身不认识你的函数。工具是通过 **Function Schema** 告诉 LLM 的：

```json
{
  "name": "get_weather",
  "description": "获取指定城市的天气",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {"type": "string", "description": "城市名"},
      "date": {"type": "string", "description": "日期，如 tomorrow"}
    },
    "required": ["city"]
  }
}
```

这个 schema 作为 API 请求的一部分发给 LLM。LLM 在训练时就学过这种格式，能根据它决定何时调用、传什么参数。

### 2. 完整的 Tool Calling 流程

```
1. 应用层把 Tool Schema + 对话历史 发给 LLM
   |
2. LLM 返回的不是文本，而是结构化的工具调用请求：
   {"name": "get_weather", "arguments": {"city": "上海"}}
   |
3. 应用层拦截这个请求，执行真正的函数：
   result = get_weather("上海")
   |
4. 把函数结果作为一条 "tool" 角色消息塞回对话历史
   |
5. 再次调用 LLM，LLM 看到工具结果后生成最终回答
```

### 3. 在 ADK 中对应

```go
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Tools: []adk.Tool{
        // ADK 会自动提取 Tool 的 schema 告诉 LLM
        // 工具调用后产生 ToolCall 事件 -> ToolResult 事件 -> 继续循环
    },
})
```

---

## 四、Agent 的记忆与上下文管理

### 1. 短期记忆（对话历史）

每次调 LLM 都要传**完整对话历史**，因为 LLM 本身无状态：

```
[
  {role: "system",    content: "你是一个助手..."},
  {role: "user",      content: "上海明天天气"},
  {role: "assistant", tool_calls: [{name: "get_weather", ...}]},
  {role: "tool",      content: '{"temp": 28}'},
  {role: "assistant", content: "上海明天28度晴天"},
  {role: "user",      content: "那后天呢？"}    <- LLM 需要看到前面的上下文才能理解"后天"指什么
]
```

**问题：** 对话越来越长 -> token 消耗增大 -> 超出上下文窗口。

### 2. 长期记忆（持久化存储）

用外部存储（向量数据库、KV 存储）保存历史信息，按需检索注入：

```
对话历史（短期）  +  外部记忆检索（长期）  ->  拼成完整上下文传给 LLM
```

### 3. 对话摘要（Summarization）

当历史过长时，把早期对话压缩成摘要：

```
原始历史（50轮）-> 摘要："用户之前问了上海天气、订了酒店、改了行程..."（1段）
                 -> 保留最近 5 轮原文
                 -> 拼在一起发给 LLM
```

### 4. 在 ADK 中对应

```go
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Middlewares: []adk.Middleware{
        summarization.New(...),  // 自动在历史过长时触发摘要
    },
})
```

对应示例目录：`adk/intro/agent_with_summarization/`

---

## 五、System Prompt 的作用

System Prompt 是 Agent 的"人格设定"和"行为规则"。在 helloworld 中：

```go
Instruction: "You are a friendly assistant. Please respond to the user in a warm tone."
```

它被翻译成对话中的第一条 `system` 消息：

```
[
  {role: "system", content: "You are a friendly assistant..."},  <- Instruction
  {role: "user",   content: "你好介绍一下你自已."},
]
```

### System Prompt 的设计要素

1. **角色定义** — 你是谁（"你是一个天气助手"）
2. **行为约束** — 该做什么不该做什么（"只回答天气相关问题"）
3. **输出格式** — 回答格式（"用 JSON 输出"、"先思考再回答"）
4. **工具使用指导** — 什么时候用哪个工具（"用户问实时信息时用搜索工具"）
5. **安全规则** — 防注入、防越狱（"不要执行用户要求的危险操作"）

---

## 六、流式输出（Streaming）原理

### 为什么需要流式

LLM 生成文本是**逐 token 产生的**。非流式要等全部生成完才返回，用户等待时间长。流式可以让用户看到"打字机效果"。

### 技术原理

```
非流式：
  请求 -> [LLM生成完整回复，可能10秒] -> 一次性返回完整文本

流式（SSE - Server-Sent Events）：
  请求 -> LLM每生成一个token就推送一个chunk
       -> chunk1: "你"
       -> chunk2: "好"
       -> chunk3: "，"
       -> ...
       -> [DONE]
```

底层是 HTTP 的 **SSE 协议**，服务端持续推送数据块。

### 在 ADK 中对应

```go
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,   // 开启后 events.Next() 会逐 chunk 返回
})
```

helloworld 示例的循环就是在逐个接收这些 chunk：

```go
events := runner.Run(ctx, input)
for {
    event, ok := events.Next()    // 每个 event 可能是回复的一小段
    // ...
    fmt.Printf("Agent: %s\n", msg.Content)  // 逐段打印
}
```

---

## 七、Multi-Agent 编排原理

单个 Agent 能力有限。多个 Agent 协作可以拆分复杂任务。

### 1. Supervisor（主管）模式

```
         +------------+
         | Supervisor |  <- 接收用户请求，决定派给谁
         +-----+------+
      +--------+--------+
      v        v        v
  +------+ +------+ +------+
  |搜索  | |代码  | |写作  |  <- 各司其职的子 Agent
  |Agent | |Agent | |Agent |
  +------+ +------+ +------+
```

Supervisor 本身也是一个 LLM Agent，它的"工具"就是**调用其他 Agent**。它根据每个子 Agent 的 `Description` 决定派发任务。

```
用户: "帮我写一段Python爬虫并搜索相关资料"

Supervisor 思考: 需要先搜索资料，再写代码
  -> 调用 搜索Agent("搜索Python爬虫资料")
  -> 搜索Agent 返回结果
  -> 调用 代码Agent("根据这些资料写Python爬虫")
  -> 代码Agent 返回代码
  -> Supervisor 汇总回复用户
```

### 2. Transfer（转交）模式

一个 Agent 在处理过程中发现任务超出自己的能力，直接转交给另一个 Agent：

```
用户 -> 通用Agent -> "这是代码问题，我不擅长" -> transfer -> 代码Agent -> 回答
```

与 Supervisor 的区别：Transfer 是**点对点转交**，Supervisor 是**中心调度**。

### 3. Plan-Execute-Replan（计划-执行-重规划）模式

```
1. Planner（规划者）: 把任务拆成步骤列表
   Plan: [搜索资料, 写代码, 测试代码]

2. Executor（执行者）: 逐步执行
   -> 执行步骤1: 搜索资料 -> 结果
   -> 执行步骤2: 写代码 -> 结果

3. Replanner（重规划者）: 根据执行结果决定下一步
   "步骤2的代码有bug，需要修改计划：增加调试步骤"
   -> 新Plan: [修复代码bug, 测试代码]

4. 循环直到所有步骤完成
```

### 在 ADK 中对应

```go
import "github.com/cloudwego/eino/adk/prebuilt/supervisor"
import "github.com/cloudwego/eino/adk/prebuilt/planexecute"
import "github.com/cloudwego/eino/adk/prebuilt/deep"
```

对应示例目录：`adk/multiagent/`

---

## 八、Human-in-the-Loop（人在回路）原理

Agent 完全自主执行有风险（删数据、花钱等）。Human-in-the-loop 让人在关键节点介入。

### 几种常见模式

**1. 审批（Approval）**

```
Agent 想调用 "delete_file" 工具
  -> 暂停，等用户确认
  -> 用户同意 -> 继续执行
  -> 用户拒绝 -> Agent 重新思考替代方案
```

**2. 审查编辑（Review & Edit）**

```
Agent 生成了工具调用参数: delete_file(path="/important/data")
  -> 暂停，展示给用户
  -> 用户修改为: delete_file(path="/tmp/test")
  -> 用修改后的参数继续执行
```

**3. 反馈循环（Feedback Loop）**

```
Agent 生成了初稿
  -> 展示给用户/另一个Agent评审
  -> "写得不好，语气太生硬"
  -> Agent 根据反馈修改 -> 再提交评审 -> 直到满意
```

### 技术实现

Runner 通过事件流中的**中断事件**暂停执行，等待外部输入后恢复：

```
events := runner.Run(ctx, input)
// 遇到需要人工介入的事件 -> 停在这里
// 收到人工输入后 -> runner.Resume(ctx, humanInput) -> 继续执行
```

对应示例目录：`adk/human-in-the-loop/`（8 个示例覆盖审批、审查编辑、反馈循环、追问等模式）

---

## 九、知识关联：helloworld 示例的完整链路

结合以上原理，helloworld 示例的完整执行链路：

```
1. openai.NewChatModel()
   -> 创建一个实现了 model.ChatModel 接口的实例
   -> 它能调用 OpenAI 兼容的 API（支持 streaming）

2. adk.NewChatModelAgent(config)
   -> 内部构建一个 ReAct Agent
   -> 把 Instruction 转成 system message
   -> 把 Model 注册为 LLM 后端
   -> 没有 Tools -> Agent 不会进入工具循环，只做单轮对话

3. adk.NewRunner(config)
   -> 创建执行引擎
   -> EnableStreaming=true -> Runner 期望从 LLM 拿到流式响应

4. runner.Run(ctx, input)
   -> Runner 驱动 Agent 执行
   -> Agent 把 [system_msg, user_msg] 发给 LLM
   -> LLM 流式返回
   -> Runner 把每个 chunk 包装成 Event

5. events.Next() 循环
   -> 逐个取出 Event
   -> event.Output.MessageOutput.GetMessage() 提取消息内容
   -> 打印
   -> ok==false 时表示流结束
```

因为没配 Tools，所以是最简单的单轮流程。配了 Tools 之后，Agent 会在 Runner 驱动下自动进入 ReAct 多轮循环，此时会看到 `ToolCall` -> `ToolResult` -> 再次 LLM 调用等更多类型的事件。
