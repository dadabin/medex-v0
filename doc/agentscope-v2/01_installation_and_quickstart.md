# 1. 安装与快速入门

## 安装

### 基础安装

```bash
pip install agentscope
# 或使用 uv
uv pip install agentscope
```

### 从源码安装

```bash
git clone -b main https://github.com/agentscope-ai/agentscope.git
cd agentscope && uv pip install -e .
```

### 完整安装（含所有额外依赖）

```bash
# Windows
pip install agentscope[full]

# Mac/Linux（方括号需要转义）
pip install agentscope\[full\]
```

## 快速入门

### 最小示例：创建一个 Agent

```python
import os
import asyncio
from agentscope.agent import Agent
from agentscope.tool import Toolkit, Bash, Read, Write, Edit
from agentscope.credential import DashScopeCredential
from agentscope.model import DashScopeChatModel
from agentscope.message import UserMsg

async def main():
    agent = Agent(
        name="Friday",
        system_prompt="你是一个名叫 Friday 的智能助手。",
        model=DashScopeChatModel(
            credential=DashScopeCredential(
                api_key=os.environ["DASHSCOPE_API_KEY"]
            ),
            model="qwen3.6-plus",
        ),
        toolkit=Toolkit(
            tools=[Bash(), Read(), Write(), Edit()],
        ),
    )

    # 同步调用
    result = await agent.reply(UserMsg(name="user", content="你好，请帮我创建一个 hello.py 文件"))
    print(result.get_text_content())

    # 流式调用
    async for event in agent.reply_stream(UserMsg(name="user", content="列出当前目录的文件")):
        if hasattr(event, 'delta'):
            print(event.delta, end="", flush=True)

asyncio.run(main())
```

### 使用 OpenAI 模型

```python
import os
from agentscope.agent import Agent
from agentscope.credential import OpenAICredential
from agentscope.model import OpenAIChatModel
from agentscope.message import UserMsg

agent = Agent(
    name="Assistant",
    system_prompt="You are a helpful assistant.",
    model=OpenAIChatModel(
        credential=OpenAICredential(api_key=os.environ["OPENAI_API_KEY"]),
        model="gpt-4.1",
    ),
)
```

### 无工具的对话 Agent

```python
agent = Agent(
    name="ChatBot",
    system_prompt="你是一个友好的聊天机器人。",
    model=DashScopeChatModel(
        credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
        model="qwen-plus",
    ),
    # 不传入 toolkit 即为纯对话模式
)

result = await agent.reply(UserMsg(name="user", content="今天天气怎么样？"))
```

### 环境变量配置

建议将 API Key 配置在环境变量中：

```bash
# DashScope（通义千问）
export DASHSCOPE_API_KEY="your-api-key"

# OpenAI
export OPENAI_API_KEY="your-api-key"

# Anthropic
export ANTHROPIC_API_KEY="your-api-key"

# DeepSeek
export DEEPSEEK_API_KEY="your-api-key"

# Gemini
export GEMINI_API_KEY="your-api-key"
```

也可以使用 `.env` 文件：

```bash
# .env
DASHSCOPE_API_KEY=sk-xxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxx
```

```python
from dotenv import load_dotenv
load_dotenv()
```

## 运行方式

### 异步运行（推荐）

AgentScope v2 的 Agent 原生支持异步：

```python
import asyncio

async def main():
    result = await agent.reply(UserMsg(name="user", content="Hello"))
    print(result.get_text_content())

asyncio.run(main())
```

### 流式运行

```python
async def stream_example():
    msg = None
    async for event in agent.reply_stream(UserMsg(name="user", content="写一首诗")):
        if isinstance(event, ReplyStartEvent):
            msg = AssistantMsg(name=event.name, content=[], id=event.reply_id)
        elif isinstance(event, TextBlockDeltaEvent):
            print(event.delta, end="", flush=True)
        elif isinstance(event, ReplyEndEvent):
            print("\n[完成]")
        if msg is not None:
            msg.append_event(event)
```
