# 4. 消息与事件系统

## 消息（Msg）

消息是 Agent 之间通信和持久化的基本单元。

### Msg 字段

| 字段 | 类型 | 用途 |
|------|------|------|
| `id` | `str` | 唯一标识符 |
| `name` | `str` | 发送者名称 |
| `role` | `"user" \| "assistant" \| "system"` | 发送者角色 |
| `content` | `list[ContentBlock]` | 有序的类型化内容块列表 |
| `metadata` | `dict` | 任意键值对 |
| `created_at` | `str` | ISO 8601 创建时间戳 |
| `finished_at` | `str \| None` | ISO 8601 完成时间戳 |
| `usage` | `Usage` | Token 统计（仅 assistant） |

### 消息工厂函数

```python
from agentscope.message import UserMsg, AssistantMsg, SystemMsg

# 简单文本消息
user_msg = UserMsg(name="user", content="你好")

# 系统消息
system_msg = SystemMsg(name="system", content="你是一个助手")

# 助手消息
assistant_msg = AssistantMsg(name="agent", content="你好！")
```

## 内容块（ContentBlock）

消息的 `content` 字段是由类型化内容块组成的列表。不同角色允许的内容块类型不同。

### 内容块类型

| 块类型 | 用途 | 允许的角色 |
|--------|------|------------|
| `TextBlock` | 纯文本 | 所有 |
| `DataBlock` | 二进制数据（通过 base64 或 URL） | user, assistant |
| `ThinkingBlock` | 链式思考推理 | 仅 assistant |
| `ToolCallBlock` | 工具调用（名称、输入、状态） | 仅 assistant |
| `ToolResultBlock` | 工具执行输出 | 仅 assistant |
| `HintBlock` | 作为用户上下文注入的指令 | 仅 assistant |

### 角色与内容块对应

- **user 消息**：仅 `TextBlock` 和 `DataBlock`
- **system 消息**：仅 `TextBlock`
- **assistant 消息**：所有类型

### 创建内容块

```python
from agentscope.message import TextBlock, DataBlock, Base64Source

# 纯文本
text_block = TextBlock(text="这是一段文本")

# 多模态消息（图片）
user_msg = UserMsg(name="user", content=[
    TextBlock(text="请描述这张图片："),
    DataBlock(source=Base64Source(data="...", media_type="image/png")),
])

# URL 来源
from agentscope.message import UrlSource
user_msg = UserMsg(name="user", content=[
    TextBlock(text="这张图片里有什么？"),
    DataBlock(source=UrlSource(url="https://example.com/image.jpg", media_type="image/jpeg")),
])
```

### 访问消息内容

```python
# 获取文本内容
text = msg.get_text_content()

# 获取特定类型的内容块
tool_calls = msg.get_content_blocks("tool_call")

# 检查是否包含特定类型的内容块
if msg.has_content_blocks("tool_result"):
    print("消息包含工具结果")
```

## 事件（Event）

事件是前端交互和流式输出的基本单元，遵循 **start → delta → end** 模式。

### 事件类型完整目录

#### 生命周期事件

| 事件 | 说明 |
|------|------|
| `ReplyStartEvent` | 回复开始 |
| `ReplyEndEvent` | 回复结束 |
| `ExceedMaxItersEvent` | 超过最大迭代次数 |

#### 文本流式事件

| 事件 | 说明 |
|------|------|
| `TextBlockStartEvent` | 文本块开始 |
| `TextBlockDeltaEvent` | 文本增量（`delta` 字段） |
| `TextBlockEndEvent` | 文本块结束 |

#### 思考流式事件

| 事件 | 说明 |
|------|------|
| `ThinkingBlockStartEvent` | 思考块开始 |
| `ThinkingBlockDeltaEvent` | 思考增量 |
| `ThinkingBlockEndEvent` | 思考块结束 |

#### 数据流式事件

| 事件 | 说明 |
|------|------|
| `DataBlockStartEvent` | 数据块开始 |
| `DataBlockDeltaEvent` | 数据增量 |
| `DataBlockEndEvent` | 数据块结束 |

#### 工具调用流式事件

| 事件 | 说明 |
|------|------|
| `ToolCallStartEvent` | 工具调用开始 |
| `ToolCallDeltaEvent` | 工具调用增量 |
| `ToolCallEndEvent` | 工具调用结束 |

#### 工具结果流式事件

| 事件 | 说明 |
|------|------|
| `ToolResultStartEvent` | 工具结果开始 |
| `ToolResultTextDeltaEvent` | 文本结果增量 |
| `ToolResultDataDeltaEvent` | 数据结果增量 |
| `ToolResultEndEvent` | 工具结果结束（state: SUCCESS/ERROR/INTERRUPTED/DENIED/RUNNING） |

#### 模型调用事件

| 事件 | 说明 |
|------|------|
| `ModelCallStartEvent` | 模型 API 调用开始 |
| `ModelCallEndEvent` | 模型 API 调用结束 |

#### 人机协作事件

| 事件 | 说明 |
|------|------|
| `RequireUserConfirmEvent` | 请求用户确认 |
| `RequireExternalExecutionEvent` | 请求外部执行 |
| `UserConfirmResultEvent` | 用户确认结果 |
| `ExternalExecutionResultEvent` | 外部执行结果 |

### 从事件重建消息

```python
from agentscope.message import AssistantMsg
from agentscope.event import (
    ReplyStartEvent, TextBlockDeltaEvent, ToolCallStartEvent,
    ToolResultEndEvent, ReplyEndEvent
)

msg = None
async for event in agent.reply_stream(UserMsg("user", "修复这个 bug")):
    if isinstance(event, ReplyStartEvent):
        msg = AssistantMsg(name=event.name, content=[], id=event.reply_id)
    elif isinstance(event, TextBlockDeltaEvent):
        print(event.delta, end="", flush=True)
    elif isinstance(event, ToolCallStartEvent):
        print(f"\n[正在调用 {event.tool_call_name}...]")
    elif isinstance(event, ToolResultEndEvent):
        print(f"[工具完成: {event.state}]")
    elif isinstance(event, ReplyEndEvent):
        print("\n[完成]")
    if msg is not None:
        msg.append_event(event)
# msg 现在持有完整回复
```

### 事件流处理示例：带进度的工具调用

```python
async for event in agent.reply_stream(user_msg):
    match event:
        case TextBlockDeltaEvent(delta=text):
            print(text, end="", flush=True)
        case ThinkingBlockDeltaEvent(delta=thought):
            print(f"[思考: {thought}]", end="", flush=True)
        case ToolCallStartEvent(tool_call_name=name):
            print(f"\n🔧 调用工具: {name}")
        case ToolResultEndEvent(state=state):
            print(f"  工具结果: {state}")
        case ReplyEndEvent():
            print("\n✅ 回复完成")
```

## 多 Agent 消息传递

v2 中 Agent 之间通过消息直接传递：

```python
# Agent A 的输出作为 Agent B 的输入
result_a = await agent_a.reply(UserMsg(name="user", content="分析这段代码"))

# 将 A 的回复作为 B 的输入
# 需要将 AssistantMsg 转换为适合 B 的输入格式
result_b = await agent_b.reply(UserMsg(name="user", content=result_a.get_text_content()))
```

### 使用格式化器（MultiAgentFormatter）

多 Agent 场景中，使用 `MultiAgentFormatter` 让模型理解消息来自不同 Agent：

```python
from agentscope.model import OpenAIChatModel, OpenAIMultiAgentFormatter

model = OpenAIChatModel(
    credential=OpenAICredential(api_key=os.environ["OPENAI_API_KEY"]),
    model="gpt-4.1",
    formatter=OpenAIMultiAgentFormatter(),
)
```

`MultiAgentFormatter` 会将连续的同一 Agent 消息分组，并在消息中标注发送者名称。
