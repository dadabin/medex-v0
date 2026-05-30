# 3. Agent 配置与使用

## 统一的 Agent 类

v2 中只有一个 `Agent` 类，通过配置实现不同行为模式，无需子类化。

## 基本配置

```python
from agentscope.agent import Agent
from agentscope.tool import Toolkit, Bash, Read, Write, Edit
from agentscope.credential import DashScopeCredential
from agentscope.model import DashScopeChatModel

agent = Agent(
    name="Friday",
    system_prompt="你是一个名叫 Friday 的智能助手，擅长编程和系统管理。",
    model=DashScopeChatModel(
        credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
        model="qwen3.6-plus",
    ),
    toolkit=Toolkit(tools=[Bash(), Read(), Write(), Edit()]),
)
```

## 完整参数列表

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | `str` | 必填 | Agent 标识名 |
| `system_prompt` | `str` | 必填 | 基础系统提示词 |
| `model` | `ChatModelBase` | 必填 | 用于推理的 LLM |
| `toolkit` | `Toolkit \| None` | `None` | 管理工具、MCP 客户端、技能和工具组 |
| `state` | `AgentState \| None` | 自动创建 | 持有上下文、权限上下文、会话状态 |
| `offloader` | `Offloader \| None` | `None` | 卸载压缩的上下文和工具结果 |
| `middlewares` | `list[MiddlewareBase] \| None` | `None` | reply、reasoning、acting、model call 和 system prompt 阶段的钩子 |
| `model_config` | `ModelConfig` | 默认值 | 重试次数和备用模型 |
| `context_config` | `ContextConfig` | 默认值 | 压缩阈值和工具结果限制 |
| `react_config` | `ReActConfig` | 默认值 | 最大迭代次数和拒绝处理 |

## 调用方式

### reply() — 同步获取完整结果

```python
from agentscope.message import UserMsg

result = await agent.reply(UserMsg(name="user", content="帮我查看当前目录"))
print(result.get_text_content())
```

### reply_stream() — 流式获取事件

```python
from agentscope.message import AssistantMsg
from agentscope.event import ReplyStartEvent, TextBlockDeltaEvent, ReplyEndEvent

msg = None
async for event in agent.reply_stream(UserMsg(name="user", content="写一个排序算法")):
    if isinstance(event, ReplyStartEvent):
        msg = AssistantMsg(name=event.name, content=[], id=event.reply_id)
    elif isinstance(event, TextBlockDeltaEvent):
        print(event.delta, end="", flush=True)
    elif isinstance(event, ReplyEndEvent):
        print("\n[完成]")
    if msg is not None:
        msg.append_event(event)
# msg 现在持有完整的回复
```

## 三种使用模式

### 1. 纯对话模式（无工具）

```python
agent = Agent(
    name="ChatBot",
    system_prompt="你是一个友好的聊天机器人。",
    model=DashScopeChatModel(
        credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
        model="qwen-plus",
    ),
    # 不传 toolkit
)

result = await agent.reply(UserMsg(name="user", content="你好"))
```

### 2. ReAct 模式（有工具）

```python
agent = Agent(
    name="DevAssistant",
    system_prompt="你是一个开发助手。",
    model=DashScopeChatModel(
        credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
        model="qwen3.6-plus",
    ),
    toolkit=Toolkit(tools=[Bash(), Read(), Write(), Edit()]),
)

result = await agent.reply(UserMsg(name="user", content="创建一个 Flask 应用"))
# Agent 会自动进行多轮推理和工具调用
```

### 3. 结构化输出模式

```python
from pydantic import BaseModel

class WeatherInfo(BaseModel):
    city: str
    temperature: float
    unit: str

response = await model.generate_structured_output(
    messages=[UserMsg(name="user", content="上海今天天气如何？")],
    structured_model=WeatherInfo,
)
print(response.content)  # 验证后的 dict，匹配 WeatherInfo 结构
```

## ReActConfig 配置

```python
from agentscope.agent import Agent, ReActConfig

agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    toolkit=toolkit,
    react_config=ReActConfig(
        max_iterations=20,     # 最大推理-执行迭代次数
        # 拒绝处理配置...
    ),
)
```

## ModelConfig 配置

```python
from agentscope.agent import Agent, ModelConfig

agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    model_config=ModelConfig(
        max_retries=3,    # API 调用失败时的最大重试次数
        # 备用模型配置...
    ),
)
```

## 多轮对话

Agent 内部维护对话上下文，多次调用 `reply()` 会自动累积历史：

```python
# 第一轮
result1 = await agent.reply(UserMsg(name="user", content="创建文件 test.py"))

# 第二轮（Agent 记得之前的上下文）
result2 = await agent.reply(UserMsg(name="user", content="在文件中添加一个 hello 函数"))

# 第三轮
result3 = await agent.reply(UserMsg(name="user", content="运行这个文件"))
```

## 切换模型提供商

只需更换 credential 和 model 对：

```python
# DashScope（通义千问）
from agentscope.credential import DashScopeCredential
from agentscope.model import DashScopeChatModel
model = DashScopeChatModel(
    credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
    model="qwen-plus",
)

# OpenAI
from agentscope.credential import OpenAICredential
from agentscope.model import OpenAIChatModel
model = OpenAIChatModel(
    credential=OpenAICredential(api_key=os.environ["OPENAI_API_KEY"]),
    model="gpt-4.1",
)

# Anthropic
from agentscope.credential import AnthropicCredential
from agentscope.model import AnthropicChatModel
model = AnthropicChatModel(
    credential=AnthropicCredential(api_key=os.environ["ANTHROPIC_API_KEY"]),
    model="claude-sonnet-4-6",
)

# DeepSeek
from agentscope.credential import DeepSeekCredential
from agentscope.model import DeepSeekChatModel
model = DeepSeekChatModel(
    credential=DeepSeekCredential(api_key=os.environ["DEEPSEEK_API_KEY"]),
    model="deepseek-chat",
)

# Gemini
from agentscope.credential import GeminiCredential
from agentscope.model import GeminiChatModel
model = GeminiChatModel(
    credential=GeminiCredential(api_key=os.environ["GEMINI_API_KEY"]),
    model="gemini-2.0-flash",
)

# Ollama（本地，无需 credential）
from agentscope.model import OllamaChatModel
model = OllamaChatModel(model="qwen3:8b")
```

## 自定义 Agent（通过中间件）

推荐使用中间件扩展 Agent 行为，而非子类化：

```python
from agentscope.middleware import MiddlewareBase

class DynamicContextMiddleware(MiddlewareBase):
    def __init__(self, context_fn):
        self._context_fn = context_fn

    async def on_system_prompt(self, agent, current_prompt):
        context = self._context_fn()
        return f"{current_prompt}\n\n## 当前上下文\n{context}"

agent = Agent(
    name="context_aware_agent",
    system_prompt="你是一个上下文感知助手。",
    model=model,
    middlewares=[DynamicContextMiddleware(lambda: f"当前时间: {datetime.now().isoformat()}")],
)
```
