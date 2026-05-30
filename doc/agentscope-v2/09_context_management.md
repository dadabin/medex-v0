# 9. 上下文管理

## 概述

v2 废弃了 v1 的 `InMemoryMemory` 模块，采用三机制上下文管理系统：

1. **上下文压缩（Context Compression）** — 自动摘要旧消息
2. **工具结果截断（Tool Result Truncation）** — 限制过大的工具输出
3. **上下文卸载（Context Offloading）** — 将丢弃的内容写入外部存储

## 上下文压缩

当 token 使用量接近限制时，自动摘要较旧的消息。

### 配置

```python
from agentscope.agent import Agent, ContextConfig

agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    context_config=ContextConfig(
        trigger_ratio=0.8,       # 在上下文大小的 80% 时触发压缩
        reserve_ratio=0.1,       # 保留 10% 作为最近消息
        tool_result_limit=3000,  # 每个工具结果的最大 token 数
    ),
)
```

### 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `trigger_ratio` | 0.8 | 压缩触发阈值（占上下文大小的比例） |
| `reserve_ratio` | 0.1 | 保留的最近消息比例 |
| `tool_result_limit` | 3000 | 单个工具结果的最大 token 数 |

### 手动压缩

```python
# 触发手动压缩
await agent.compress_context()

# 使用自定义配置覆盖
from agentscope.agent import ContextConfig

await agent.compress_context(
    context_config=ContextConfig(
        trigger_ratio=0.5,    # 更积极地压缩
        reserve_ratio=0.1,
    ),
)
```

### 压缩流程

```
1. 计算 token 数（系统提示词 + 摘要 + 上下文 + 工具 schema）
2. 如果总量超过 trigger_ratio × context_size，激活压缩
3. 分割消息：
   - 较旧消息标记为压缩
   - 最近消息保留不变（工具调用/结果对保持在一起）
4. 模型生成结构化摘要，包含：
   - task_overview（任务概述）
   - current_state（当前状态）
   - important_discoveries（重要发现）
   - next_steps（下一步）
   - context_to_preserve（需保留的上下文）
5. 摘要替换被压缩的消息
```

### 压缩示例

```python
# 长对话中的自动压缩
agent = Agent(
    name="long_chat_agent",
    system_prompt="你是一个代码开发助手。",
    model=DashScopeChatModel(
        credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
        model="qwen-plus",
    ),
    toolkit=Toolkit(tools=[Bash(), Read(), Write(), Edit()]),
    context_config=ContextConfig(
        trigger_ratio=0.7,     # 较早触发压缩
        reserve_ratio=0.15,    # 保留更多最近消息
        tool_result_limit=2000,
    ),
)

# 多轮对话，上下文会自动压缩
for i in range(50):
    result = await agent.reply(UserMsg(name="user", content=f"第 {i+1} 个任务: ..."))
```

## 工具结果截断

限制过大的工具输出，防止上下文溢出。

```python
# 通过 ContextConfig 设置截断限制
agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    context_config=ContextConfig(
        tool_result_limit=3000,  # 超过 3000 token 的结果会被截断
    ),
)
```

如果配置了 offloader，截断的内容会被持久化到存储中，并附带引用路径。

## 上下文卸载

将压缩/截断的内容写入外部存储，以便需要时检索。

### Offloader 协议

```python
class S3Offloader:
    async def offload_context(self, session_id: str, msgs: list[Msg], **kwargs) -> str:
        """持久化压缩的消息，返回引用路径。"""
        ...

    async def offload_tool_result(self, session_id: str, tool_result: ToolResultBlock, **kwargs) -> str:
        """持久化截断的工具结果，返回引用路径。"""
        ...
```

### LocalWorkspace 作为 Offloader

LocalWorkspace 内置 offloader 功能：

```
{workdir}/
  sessions/{session_id}/
    context.jsonl              # 压缩的消息
    tool_result-{tool_id}.txt  # 截断的工具结果
```

```python
from agentscope.workspace import LocalWorkspace

workspace = LocalWorkspace(workdir="/data/my-workspace")

agent = Agent(
    name="offloaded_agent",
    system_prompt="...",
    model=model,
    offloader=workspace,  # 使用 workspace 作为 offloader
    context_config=ContextConfig(
        trigger_ratio=0.8,
        tool_result_limit=3000,
    ),
)
```

## AgentState

`AgentState` 替代 v1 的 memory，是 Pydantic 模型，可序列化为 JSON。

### 创建

```python
from agentscope.state import AgentState
from agentscope.permission import PermissionContext, PermissionMode

# 默认状态
state = AgentState()

# 带权限配置
state = AgentState(
    permission_context=PermissionContext(mode=PermissionMode.DEFAULT),
)
```

### 使用

```python
agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    state=state,  # 传入自定义状态
)
```

### 持久化

AgentState 可序列化为 JSON，存储到任意后端：

```python
import json

# 序列化
state_json = state.model_dump_json()

# 反序列化
restored_state = AgentState.model_validate_json(state_json)
```

## 上下文溢出处理

如果系统提示词本身就超过压缩阈值，会抛出 `RuntimeError`：

```python
try:
    result = await agent.reply(user_msg)
except RuntimeError as e:
    if "exceeds" in str(e):
        print("上下文溢出！请减少系统提示词长度或增大模型上下文窗口。")
```

## v1 对比

| v1 | v2 | 说明 |
|----|----|------|
| `InMemoryMemory` | `AgentState` + 上下文压缩 | 废弃内存模块 |
| 手动管理记忆 | 自动压缩 + 截断 | 更智能的上下文管理 |
| 数据库持久化 | JSON 序列化 + Offloader | 更灵活的存储方案 |
| `state_dict` / `load_state_dict` | `AgentState.model_dump_json()` | Pydantic 原生序列化 |
