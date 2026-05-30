# 2. 核心概念

AgentScope v2 围绕以下核心抽象构建：

## Agent（智能体）

Agent 是中心抽象，是一个无状态的推理-行动循环引擎，统一了模型、工具、权限、人机协作、上下文管理、中间件、状态管理和事件。

**关键特性：**
- v2 中只有一个统一的 `Agent` 类，不再有 v1 的多种 Agent 子类
- 内置 ReAct（Reasoning + Acting）循环
- 通过配置而非子类化来定制行为

```python
from agentscope.agent import Agent

agent = Agent(
    name="my_agent",
    system_prompt="你是一个有用的助手。",
    model=model,
    toolkit=toolkit,        # 可选
    state=agent_state,      # 可选
    middlewares=[...],       # 可选
)
```

## Message（消息）

消息是 Agent 之间通信和持久化的基本单元。一条完整的对话轮次存储在上下文中并在 Agent 之间交换。

```python
from agentscope.message import UserMsg, AssistantMsg, SystemMsg

user_msg = UserMsg(name="user", content="你好")
assistant_msg = AssistantMsg(name="agent", content="你好！有什么可以帮助你的？")
system_msg = SystemMsg(name="system", content="你是一个助手")
```

消息的内容由类型化的内容块（ContentBlock）组成，详见 [04_message_and_event.md](04_message_and_event.md)。

## Event（事件）

事件是前端交互和流式输出的基本单元。携带增量进度信息，如文本 token、工具调用片段、权限请求等。

单次 `reply` 调用产生的所有事件累积为恰好一条 assistant `Msg`。

```python
async for event in agent.reply_stream(user_msg):
    # 处理流式事件
    pass
```

## Toolkit（工具包）

Toolkit 是注册工具、MCP 客户端和技能的容器，将其 JSON Schema 暴露给模型，并分发每个工具调用。

```python
from agentscope.tool import Toolkit, Bash, Read, Write, Edit

toolkit = Toolkit(tools=[Bash(), Read(), Write(), Edit()])
```

## Workspace（工作空间）

Agent 的执行环境，提供工具、技能、MCP 服务器和上下文卸载能力。三种实现：

| 类型 | 说明 |
|------|------|
| `LocalWorkspace` | 本地文件系统 |
| `DockerWorkspace` | Docker 容器隔离 |
| `E2BWorkspace` | E2B 云沙箱 |

```python
from agentscope.workspace import LocalWorkspace

workspace = LocalWorkspace(workdir="/data/my-workspace")
```

## Middleware（中间件）

在 Agent 生命周期的关键位置拦截和扩展行为。5 个生命周期钩子：

| 钩子 | 类型 | 触发时机 |
|------|------|----------|
| `on_reply` | 洋葱模型 | 包裹整个 reply（所有 ReAct 轮次） |
| `on_reasoning` | 洋葱模型 | 包裹一轮 ReAct 的推理步骤 |
| `on_acting` | 洋葱模型 | 包裹单个工具调用执行 |
| `on_model_call` | 洋葱模型 | 包裹 ChatModel API 调用 |
| `on_system_prompt` | 转换器 | 系统提示词组装 |

## AgentState（Agent 状态）

Pydantic 模型，持有对话上下文、压缩摘要、权限规则、工具状态和回复位置。可序列化为 JSON 存储到任意后端。

```python
from agentscope.state import AgentState
from agentscope.permission import PermissionContext, PermissionMode

state = AgentState(
    permission_context=PermissionContext(mode=PermissionMode.DEFAULT),
)
```

## 核心流程：ReAct 循环

```
用户消息 → Agent.reply()
  │
  ├── [on_reply 中间件]
  │     │
  │     ├── 推理步骤 [on_reasoning]
  │     │     └── 模型调用 [on_model_call]
  │     │           └── 返回：文本 / 工具调用
  │     │
  │     ├── 如果是工具调用 → 执行步骤 [on_acting]
  │     │     └── 权限检查 → 执行工具 → 返回结果
  │     │
  │     └── 重复推理-执行直到模型返回纯文本或达到最大迭代次数
  │
  └── 返回 AssistantMsg
```

## v1 到 v2 的核心概念映射

| v1 概念 | v2 概念 | 说明 |
|----------|---------|------|
| DialogAgent | Agent（无 toolkit） | 纯对话 |
| ReActAgent | Agent（有 toolkit） | 推理+工具 |
| DictDialogAgent | Agent + 结构化输出 | Pydantic 模型 |
| UserAgent | 直接用 UserMsg | 无需特殊 Agent |
| InMemoryMemory | AgentState + 上下文压缩 | 废弃内存模块 |
| Hook | Middleware | 更灵活的生命周期拦截 |
| Pipeline | 消息中枢 + 直接通信 | 去中心化编排 |
| RpcAgent | Workspace + Agent Service | 新的分布式方案 |
| ServiceFactory | Toolkit + ToolBase | 统一的工具注册 |
