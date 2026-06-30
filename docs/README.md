# Eino ADK 学习笔记

本目录记录了学习 Eino ADK（Agent Development Kit）过程中整理的原理知识和代码讲解。

## 文档索引

| 文档 | 内容 |
|------|------|
| [adk-overview.md](./adk-overview.md) | ADK 架构总览、核心组件、分层关系、模块列表 |
| [helloworld-walkthrough.md](./helloworld-walkthrough.md) | helloworld 示例逐段讲解与执行流程分析 |
| [agent-principles.md](./agent-principles.md) | AI Agent 原理：ReAct、Tool Calling、记忆管理、多 Agent 编排、Human-in-the-Loop 等 |

## 阅读顺序

1. 先读 [agent-principles.md](./agent-principles.md) 了解 AI Agent 的核心原理
2. 再读 [adk-overview.md](./adk-overview.md) 了解 ADK 的架构和组件设计
3. 最后读 [helloworld-walkthrough.md](./helloworld-walkthrough.md) 结合代码理解 ADK 的实际运行

## 对应示例代码

| 文档 | 示例代码 |
|------|----------|
| helloworld-walkthrough.md | [adk/helloworld/helloworld.go](../adk/helloworld/helloworld.go) |
| adk-overview.md | [adk/](../adk/) 全部示例 |
