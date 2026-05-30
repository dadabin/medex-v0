# 6. 模型配置

## 配置层级

v2 采用两层配置：**Credential（凭证）** 在顶层，模型族在其下。

## 内置凭证类型

| 凭证 | 模型类 | 说明 |
|------|--------|------|
| `DashScopeCredential` | `DashScopeChatModel` | 通义千问系列 |
| `OpenAICredential` | `OpenAIChatModel` / `OpenAIResponseModel` | GPT 系列 |
| `AnthropicCredential` | `AnthropicChatModel` | Claude 系列 |
| `DeepSeekCredential` | `DeepSeekChatModel` | DeepSeek 系列 |
| `GeminiCredential` | `GeminiChatModel` | Gemini 系列 |
| `MoonshotCredential` | `MoonshotChatModel` | Kimi 系列 |
| `XAIredential` | `XAIChatModel` | Grok 系列 |
| `OllamaCredential` | `OllamaChatModel` | 本地模型（可选凭证） |

## 创建凭证

```python
import os
from agentscope.credential import (
    DashScopeCredential,
    OpenAICredential,
    AnthropicCredential,
    DeepSeekCredential,
    GeminiCredential,
)

# DashScope
cred = DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"])

# OpenAI
cred = OpenAICredential(api_key=os.environ["OPENAI_API_KEY"])

# Anthropic
cred = AnthropicCredential(api_key=os.environ["ANTHROPIC_API_KEY"])

# DeepSeek
cred = DeepSeekCredential(api_key=os.environ["DEEPSEEK_API_KEY"])

# Gemini
cred = GeminiCredential(api_key=os.environ["GEMINI_API_KEY"])
```

## 创建模型

### 基本用法

```python
from agentscope.model import DashScopeChatModel

model = DashScopeChatModel(
    credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
    model="qwen-plus",
    stream=True,
)
```

### 带参数

```python
model = DashScopeChatModel(
    credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
    model="qwen-plus",
    stream=False,
    parameters=DashScopeChatModel.Parameters(
        parallel_tool_calls=False,
    ),
)
```

### 推理/思考模式

```python
model = DashScopeChatModel(
    credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
    model="qwen3-235b-a22b-thinking-2507",
    parameters=DashScopeChatModel.Parameters(
        thinking_enable=True,
        thinking_budget=2048,
    ),
)
```

### OpenAI Responses API（o3/o4-mini）

```python
from agentscope.model import OpenAIResponseModel

model = OpenAIResponseModel(
    credential=OpenAICredential(api_key=os.environ["OPENAI_API_KEY"]),
    model="o3",
)
```

## 内置模型提供商详情

| 提供商 | 模型类 | 特性 |
|--------|--------|------|
| OpenAI | `OpenAIChatModel` | Chat Completions API；vLLM 兼容 |
| OpenAI Responses | `OpenAIResponseModel` | Responses API；o3/o4-mini 推理 |
| Anthropic | `AnthropicChatModel` | Extended thinking，prompt caching |
| DashScope | `DashScopeChatModel` | Qwen 模型；多模态 |
| DeepSeek | `DeepSeekChatModel` | OpenAI 兼容；prompt cache |
| Gemini | `GeminiChatModel` | 多模态 |
| Moonshot | `MoonshotChatModel` | Kimi 模型 |
| xAI | `XAIChatModel` | Grok 模型；原生推理 |
| Ollama | `OllamaChatModel` | 本地运行；凭证可选 |

## 结构化输出

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

## 格式化器（Formatter）

两种内置格式化器：

| 格式化器 | 说明 |
|----------|------|
| `ChatFormatter`（默认） | 单 Agent 对话；1:1 Msg 到 API 消息映射 |
| `MultiAgentFormatter` | 多 Agent 场景；连续同一 Agent 消息分组，标注发送者 |

```python
from agentscope.model import OpenAIChatModel, OpenAIMultiAgentFormatter

model = OpenAIChatModel(
    credential=OpenAICredential(api_key=os.environ["OPENAI_API_KEY"]),
    model="gpt-4.1",
    formatter=OpenAIMultiAgentFormatter(),
)
```

## 自定义模型提供商

```python
from agentscope.credential import CredentialBase
from agentscope.model import ChatModelBase
from typing import Literal
from pydantic import BaseModel, Field

# 1. 定义凭证
class MyProviderCredential(CredentialBase):
    type: Literal["my_provider_credential"] = "my_provider_credential"
    api_key: str
    base_url: str = "https://api.myprovider.com/v1"

    @classmethod
    def get_chat_model_class(cls):
        return MyProviderChatModel

# 2. 定义参数模型
class MyProviderParameters(BaseModel):
    max_tokens: int | None = Field(default=None, gt=0)
    temperature: float | None = Field(default=None, ge=0, le=2)

# 3. 定义模型类
class MyProviderChatModel(ChatModelBase):
    type: Literal["my_provider_chat"] = "my_provider_chat"

    async def _call_api(self, model_name, messages, tools=None, tool_choice=None, **kwargs):
        formatted_messages = await self.formatter.format(messages)
        # 调用提供商 API
        # ...
```

## ModelCard YAML

声明式模型描述，驱动前端选择器：

```yaml
name: claude-sonnet-4-6
label: Claude Sonnet 4.6
status: active
input_types:
  - text/plain
  - image/jpeg
  - image/png
output_types:
  - text/plain
  - application/x-thinking
context_size: 1000000
output_size: 65536
parameter_overrides:
  max_tokens: {"maximum": 65536}
```

## v1 对比（仅作参考）

v1 使用 JSON 配置 + `agentscope.init()`：

```python
# v1 风格（已废弃）
# model_configs.json
[
    {"config_name": "my_config", "model_type": "openai_chat", "model_name": "gpt-4", "api_key": "..."}
]

agentscope.init(model_configs="./model_configs.json")
```

v2 改为编程式配置，无需 `agentscope.init()`。
