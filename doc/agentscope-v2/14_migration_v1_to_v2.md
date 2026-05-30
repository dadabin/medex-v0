# 14. v1 到 v2 迁移指南

## 概述

v2.0 是破坏性版本，无自动迁移路径。本文档帮助你将 v1 代码迁移到 v2。

## 迁移清单

### 1. 替换调用方式

```python
# v1
result = agent(user_msg)

# v2
result = await agent.reply(user_msg)
# 或流式
async for event in agent.reply_stream(user_msg):
    ...
```

### 2. 替换 Hook 为 Middleware

```python
# v1
class MyHook(Hook):
    def on_model_call(self, **kwargs):
        ...

agent.hooks.append(MyHook())

# v2
class MyMiddleware(MiddlewareBase):
    async def on_model_call(self, agent, input_kwargs, next_handler):
        result = await next_handler()
        return result

agent = Agent(..., middlewares=[MyMiddleware()])
```

### 3. 替换状态管理

```python
# v1
state = agent.state_dict()
agent.load_state_dict(state)

# v2
state_json = agent.state.model_dump_json()
restored_state = AgentState.model_validate_json(state_json)
```

### 4. 替换内容块类型

```python
# v1
from agentscope.message import ToolUseBlock, ImageBlock, AudioBlock, VideoBlock

# v2
from agentscope.message import ToolCallBlock, DataBlock
# ToolUseBlock → ToolCallBlock
# ImageBlock/AudioBlock/VideoBlock → DataBlock(media_type="image/png")
```

### 5. 移除 Agent print 调用

```python
# v1
agent.print(...)  # 不再支持

# v2 — Agent 是纯生产者，不直接输出
# 通过事件系统获取输出
```

### 6. 迁移 OpenTelemetry

```python
# v1 — Agent 级 OTEL
# （已移除）

# v2 — 使用 TracingMiddleware
from agentscope.middleware import TracingMiddleware

agent = Agent(..., middlewares=[TracingMiddleware()])
```

### 7. 移除 Trinity 模型包装器

```python
# v1
from agentscope.model import Trinity
model = Trinity(...)

# v2 — 使用 Credential + ChatModel
from agentscope.credential import OpenAICredential
from agentscope.model import OpenAIChatModel

model = OpenAIChatModel(
    credential=OpenAICredential(api_key="..."),
    model="gpt-4.1",
)
```

### 8. 移除 agentscope.init()

```python
# v1
agentscope.init(model_configs="./model_configs.json")

# v2 — 编程式配置，无需 init
agent = Agent(
    name="...",
    model=DashScopeChatModel(
        credential=DashScopeCredential(api_key="..."),
        model="qwen-plus",
    ),
)
```

### 9. 迁移内存模块

```python
# v1
from agentscope.memory import InMemoryMemory
memory = InMemoryMemory()
agent.memory = memory

# v2 — 使用 AgentState + 上下文压缩
from agentscope.state import AgentState
from agentscope.agent import ContextConfig

agent = Agent(
    ...,
    state=AgentState(),
    context_config=ContextConfig(trigger_ratio=0.8, reserve_ratio=0.1),
)
```

### 10. 替换 Pipeline 编排

```python
# v1 — SequentialPipeline
from agentscope.pipelines import SequentialPipeline
pipeline = SequentialPipeline([agent1, agent2, agent3])
result = pipeline(user_msg)

# v2 — 直接消息传递
result1 = await agent1.reply(user_msg)
result2 = await agent2.reply(UserMsg(name="user", content=result1.get_text_content()))
result3 = await agent3.reply(UserMsg(name="user", content=result2.get_text_content()))
```

```python
# v1 — MsgHub（广播式群聊）
from agentscope.pipelines import MsgHub
with MsgHub(participants=[agent1, agent2, agent3]):
    ...

# v2 — 通过消息中枢或 A2A 协议实现
# 或简单的循环消息传递
```

### 11. 迁移 Agent 类型

| v1 Agent | v2 等价 |
|----------|---------|
| `ReActAgent` | `Agent` + `toolkit` |
| `DialogAgent` | `Agent`（无 toolkit） |
| `DictDialogAgent` | `Agent` + 结构化输出 |
| `UserAgent` | 直接使用 `UserMsg` |
| `ProgrammerAgent` | `Agent` + `Bash`/`Write` 工具 |
| `TextToImageAgent` | `Agent` + 图片生成 MCP/工具 |

```python
# v1
from agentscope.agent import ReActAgent
agent = ReActAgent(name="my_agent", ...)

# v2
from agentscope.agent import Agent
agent = Agent(name="my_agent", toolkit=toolkit, ...)
```

### 12. 迁移分布式模式

```python
# v1
from agentscope.rpc import RpcAgentServerLauncher
launcher = RpcAgentServerLauncher(...)
launcher.launch()

# v2 — 使用 Workspace + Agent Service
from agentscope.workspace import DockerWorkspace
from agentscope.app import create_app

workspace = DockerWorkspace(...)
app = create_app(storage=storage, workspace_manager=workspace_manager)
```

### 13. 迁移工具定义

```python
# v1 — ServiceFactory
from agentscope.service import ServiceFactory

@ServiceFactory
def my_service(param: str) -> str:
    ...

# v2 — FunctionTool 或 ToolBase
from agentscope.tool import FunctionTool, ToolBase

# 轻量方式
def my_service(param: str) -> str:
    """服务描述。"""
    ...

toolkit = Toolkit(tools=[FunctionTool(my_service)])

# 或完整方式
class MyTool(ToolBase):
    name = "my_service"
    description = "服务描述。"
    ...
```

## 待迁移功能

以下 v1 功能在 v2 中尚未实现，将在未来版本支持：

- **RAG** — `RAGAgentBase`、`KnowledgeBank`、文档读取器、知识库
- **长期记忆** — ReMe 集成
- **Gradio UI** — `as_studio` 一键转 GUI

## 迁移示例：完整的 v1 → v2 转换

### v1 代码

```python
import agentscope
from agentscope.agent import ReActAgent, DialogAgent, UserAgent
from agentscope.service import ServiceFactory, BingSearchService
from agentscope.pipelines import SequentialPipeline
from agentscope.message import Msg

# 初始化
agentscope.init(model_configs="./model_configs.json")

# 定义服务
search_service = ServiceFactory(BingSearchService)

# 创建 Agent
searcher = ReActAgent(
    name="Searcher",
    sys_prompt="你是一个搜索助手。",
    model_config_name="my_config",
    service_list=[search_service],
)
writer = DialogAgent(
    name="Writer",
    sys_prompt="你是一个写作助手。",
    model_config_name="my_config",
)
user = UserAgent(name="User")

# 创建流水线
pipeline = SequentialPipeline([searcher, writer])

# 运行
result = pipeline(Msg(name="User", content="写一篇关于 AI 的文章"))
```

### v2 代码

```python
import os
import asyncio
from agentscope.agent import Agent
from agentscope.tool import Toolkit, FunctionTool
from agentscope.credential import DashScopeCredential
from agentscope.model import DashScopeChatModel
from agentscope.message import UserMsg

async def main():
    # 创建模型
    model = DashScopeChatModel(
        credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
        model="qwen-plus",
    )

    # 定义搜索工具
    async def web_search(query: str) -> str:
        """搜索网络信息。

        Args:
            query: 搜索查询词。
        """
        # 实际搜索逻辑
        return "搜索结果..."

    # 搜索 Agent
    searcher = Agent(
        name="Searcher",
        system_prompt="你是一个搜索助手，根据用户需求搜索信息。",
        model=model,
        toolkit=Toolkit(tools=[FunctionTool(web_search)]),
    )

    # 写作 Agent
    writer = Agent(
        name="Writer",
        system_prompt="你是一个写作助手，根据搜索结果撰写文章。",
        model=model,
    )

    # 手动编排
    user_msg = UserMsg(name="user", content="写一篇关于 AI 的文章")
    search_result = await searcher.reply(user_msg)
    article = await writer.reply(
        UserMsg(name="user", content=f"请根据以下信息写文章：\n{search_result.get_text_content()}")
    )
    print(article.get_text_content())

asyncio.run(main())
```
