# AgentScope v2 开发手册

## 概述

AgentScope 是阿里巴巴 ModelScope 团队开源的生产级 AI Agent 框架，旨在简化从单 Agent 工具到多 Agent 协作系统的构建、部署和服务化。

## 设计哲学

v2 的核心理念：**充分利用模型本身的推理和工具使用能力**，而非通过严格的 prompt 和固化的编排流程来约束它们。

## 三大支柱

| 支柱 | 说明 |
|------|------|
| **Simple** | 5 分钟构建 Agent，内置 ReAct 循环、工具、技能、人机协作、上下文管理、规划、实时语音、评估和模型微调 |
| **Extensible** | 工具/记忆/可观测性的生态集成，原生支持 MCP 和 A2A 协议，消息中枢实现灵活的多 Agent 编排 |
| **Production-ready** | 本地部署、云上 Serverless 或 K8s 部署，内置 OpenTelemetry 支持 |

## 系统要求

- Python 3.11+
- Apache 2.0 许可证

## v2 相比 v1 的重大变化

> v2.0 是**破坏性版本**，无自动迁移路径。

### 架构变化一览

| 领域 | v1 | v2 |
|------|----|----|
| Agent | 多种类型（ReActAgent, DialogAgent, DictDialogAgent 等） | 统一的 `Agent` 类，内部实现 ReAct 循环 |
| 调用方式 | `__call__` | `reply()` 和 `reply_stream()` |
| 事件系统 | 无 | 完整的事件流系统，支持流式输出、人机协作和可观测性 |
| 消息格式 | `Msg` 为 Python dict | `Msg` 为 Pydantic 模型，含类型化内容块 |
| 记忆 | `InMemoryMemory` 类 | 废弃，由 `AgentState` 和上下文压缩/卸载替代 |
| 流水线 | SequentialPipeline, ForkJoinPipeline 等 | 移除，通过消息中枢和 A2A 协议编排 |
| 分布式 | 基于 Actor 的 RpcAgent | Workspace 系统（Local/Docker/E2B）+ Agent Service |
| 权限系统 | 无 | 完整的权限系统（ALLOW/DENY/ASK） |
| 中间件 | Hook 机制 | 完整中间件系统，5 个生命周期钩子 |
| 工具 | ServiceFactory | Toolkit + ToolBase + FunctionTool + MCPTool + ToolGroup |
| 模型 | Trinity 包装器 | Credential + ChatModel 分层，9+ 提供商 |
| RAG | RAGAgentBase, KnowledgeBank | 尚未移植，未来版本支持 |
| 配置 | JSON 文件 + `agentscope.init()` | 编程式配置，无 `agentscope.init()` |

### API 重命名

| v1 | v2 | 说明 |
|----|----|----|
| `ToolUseBlock` | `ToolCallBlock` | 增加 `state` 和 `suggested_rules` 字段 |
| `__call__` | `reply` / `reply_stream` | 产出事件流 |
| `ImageBlock`/`AudioBlock`/`VideoBlock` | `DataBlock` | 统一为 `media_type` 字段 |

### 移除的功能

- Agent 级 OpenTelemetry（迁移至 `TracingMiddleware`）
- `state_dict` / `load_state_dict`（由 `AgentState` 替代）
- Agent 的 `print` 接口
- Memory 模块
- `Trinity` 模型包装器
- 多种内置 Agent 类型
- v1 流水线类

### 新增模块

- 事件系统 — 前端集成和人机协作支持
- 权限系统 — 工具执行的规则和模式控制
- 中间件系统 — 替代 hooks，含 `TracingMiddleware`
- 技能模块 — 基于 Markdown 的指令集
- Workspace 模块 — 统一的执行环境抽象
- Agent Service — FastAPI 多租户 HTTP 服务

## 手册目录

| 章节 | 文件 | 内容 |
|------|------|------|
| 1 | [01_installation_and_quickstart.md](01_installation_and_quickstart.md) | 安装与快速入门 |
| 2 | [02_core_concepts.md](02_core_concepts.md) | 核心概念 |
| 3 | [03_agent.md](03_agent.md) | Agent 配置与使用 |
| 4 | [04_message_and_event.md](04_message_and_event.md) | 消息与事件系统 |
| 5 | [05_tool_and_toolkit.md](05_tool_and_toolkit.md) | 工具与工具包 |
| 6 | [06_model_configuration.md](06_model_configuration.md) | 模型配置 |
| 7 | [07_permission_system.md](07_permission_system.md) | 权限系统 |
| 8 | [08_middleware.md](08_middleware.md) | 中间件系统 |
| 9 | [09_context_management.md](09_context_management.md) | 上下文管理 |
| 10 | [10_workspace.md](10_workspace.md) | Workspace 与分布式部署 |
| 11 | [11_mcp_integration.md](11_mcp_integration.md) | MCP 集成 |
| 12 | [12_skills.md](12_skills.md) | 技能系统 |
| 13 | [13_agent_service.md](13_agent_service.md) | Agent Service 生产部署 |
| 14 | [14_migration_v1_to_v2.md](14_migration_v1_to_v2.md) | v1 到 v2 迁移指南 |
| 15 | [15_tencent_cloud_agent_sandbox.md](15_tencent_cloud_agent_sandbox.md) | 腾讯云 Agent 沙箱调研 |
| 16 | [16_agentscope_tencent_cloud_integration.md](16_agentscope_tencent_cloud_integration.md) | AgentScope 整合腾讯云沙箱方案 |
