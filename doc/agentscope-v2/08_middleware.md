# 8. 中间件系统

## 概述

中间件在 Agent 生命周期的关键位置拦截和扩展行为，替代 v1 的 Hook 机制。

## 5 个生命周期钩子

| 钩子位置 | 类型 | 触发时机 |
|----------|------|----------|
| `on_reply` | 洋葱模型 | 包裹整个 reply（所有 ReAct 轮次） |
| `on_reasoning` | 洋葱模型 | 包裹一轮 ReAct 的推理步骤 |
| `on_acting` | 洋葱模型 | 包裹单个工具调用执行 |
| `on_model_call` | 洋葱模型 | 包裹 ChatModel API 调用 |
| `on_system_prompt` | 转换器 | 系统提示词组装 |

## 执行顺序

- **洋葱模型**：列表中第一个中间件是最外层
- **转换器**：从左到右链式处理

```
请求 → 中间件1 → 中间件2 → 中间件3 → 核心逻辑
                                            ↓
响应 ← 中间件1 ← 中间件2 ← 中间件3 ← 核心逻辑
```

## 创建中间件

### 基础结构

```python
from agentscope.middleware import MiddlewareBase

class MyMiddleware(MiddlewareBase):
    # 洋葱模型钩子
    async def on_reply(self, agent, input_kwargs, next_handler):
        # 前置逻辑
        result = await next_handler()
        # 后置逻辑
        return result

    async def on_reasoning(self, agent, input_kwargs, next_handler):
        result = await next_handler()
        return result

    async def on_acting(self, agent, input_kwargs, next_handler):
        result = await next_handler()
        return result

    async def on_model_call(self, agent, input_kwargs, next_handler):
        result = await next_handler()
        return result

    # 转换器钩子
    async def on_system_prompt(self, agent, current_prompt):
        return f"{current_prompt}\n\n附加指令..."
```

### 示例：计时中间件

```python
import time
from agentscope.middleware import MiddlewareBase

class TimingMiddleware(MiddlewareBase):
    async def on_model_call(self, agent, input_kwargs, next_handler):
        model_name = input_kwargs["current_model"].model
        start = time.time()
        result = await next_handler()
        elapsed = time.time() - start
        print(f"[计时] {agent.name} -> {model_name}: {elapsed:.2f}s")
        return result
```

### 示例：限流中间件

```python
import asyncio
import time
from agentscope.middleware import MiddlewareBase

class RateLimitMiddleware(MiddlewareBase):
    def __init__(self, min_interval: float = 1.0):
        self._last_call = 0.0
        self._min_interval = min_interval

    async def on_model_call(self, agent, input_kwargs, next_handler):
        now = time.time()
        wait = self._min_interval - (now - self._last_call)
        if wait > 0:
            await asyncio.sleep(wait)
        self._last_call = time.time()
        return await next_handler()
```

### 示例：动态上下文中间件

```python
from datetime import datetime
from agentscope.middleware import MiddlewareBase

class DynamicContextMiddleware(MiddlewareBase):
    def __init__(self, context_fn):
        self._context_fn = context_fn

    async def on_system_prompt(self, agent, current_prompt):
        context = self._context_fn()
        return f"{current_prompt}\n\n## 当前上下文\n{context}"

# 使用
agent = Agent(
    name="context_aware",
    system_prompt="你是一个上下文感知助手。",
    model=model,
    middlewares=[
        DynamicContextMiddleware(lambda: f"当前时间: {datetime.now().isoformat()}"),
    ],
)
```

### 示例：模型降级中间件

```python
from agentscope.middleware import MiddlewareBase

class ModelFallbackMiddleware(MiddlewareBase):
    def __init__(self, fallback_model):
        self._fallback = fallback_model

    async def on_model_call(self, agent, input_kwargs, next_handler):
        try:
            return await next_handler()
        except Exception as e:
            print(f"[降级] 主模型失败: {e}，切换到备用模型")
            return await next_handler(current_model=self._fallback)
```

### 示例：工具执行日志中间件

```python
from agentscope.middleware import MiddlewareBase

class ToolLoggingMiddleware(MiddlewareBase):
    async def on_acting(self, agent, input_kwargs, next_handler):
        tool_name = input_kwargs.get("tool_name", "unknown")
        tool_input = input_kwargs.get("tool_input", {})
        print(f"[工具调用] {tool_name}({tool_input})")

        result = await next_handler()

        print(f"[工具结果] {tool_name}: 成功")
        return result
```

### 示例：回复增强中间件

```python
from agentscope.middleware import MiddlewareBase

class ReplyEnhancementMiddleware(MiddlewareBase):
    async def on_reply(self, agent, input_kwargs, next_handler):
        result = await next_handler()
        # 可以修改最终回复
        return result

    async def on_system_prompt(self, agent, current_prompt):
        return (
            f"{current_prompt}\n\n"
            "## 额外指令\n"
            "- 回复时使用 Markdown 格式\n"
            "- 代码示例附带注释\n"
            "- 回复结尾提供总结"
        )
```

## 内置中间件

### TracingMiddleware（OpenTelemetry）

```python
from agentscope.middleware import TracingMiddleware

agent = Agent(
    name="traced_agent",
    system_prompt="...",
    model=model,
    middlewares=[TracingMiddleware()],
)
```

特性：
- 在 `on_reply`、`on_model_call` 和 `on_acting` 上创建层级化 span
- 未配置 `TracerProvider` 时零开销
- 支持 OpenTelemetry 标准的 trace 和 span

## 中间件组合

可以组合多个中间件，按列表顺序执行：

```python
agent = Agent(
    name="production_agent",
    system_prompt="...",
    model=model,
    toolkit=toolkit,
    middlewares=[
        TracingMiddleware(),           # 最外层：追踪
        TimingMiddleware(),            # 计时
        RateLimitMiddleware(1.0),      # 限流
        ModelFallbackMiddleware(fallback),  # 降级
        ToolLoggingMiddleware(),       # 日志
    ],
)
```

执行顺序（洋葱模型）：
```
Tracing → Timing → RateLimit → Fallback → Logging → 核心逻辑
```
