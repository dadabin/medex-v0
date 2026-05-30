# 7. 权限系统

## 概述

权限系统在工具执行前拦截调用，确保 Agent 的行为在安全边界内。

## 权限决策流程

按优先级顺序：

```
1. Deny 规则 → 匹配的调用立即阻止
2. Ask 规则 → 需要用户确认
3. 内置检查 → 工具特定的运行时分析
4. Allow 规则 → 匹配的调用放行
5. 模式回退 → 根据当前模式决定
6. 最终回退 → ASK（或 DONT_ASK 模式下 DENY）
```

## 权限模式

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `DEFAULT` | 每个操作需要显式规则或用户确认 | 最安全 |
| `ACCEPT_EDITS` | 自动允许工作目录内的文件操作 | 活跃开发 |
| `EXPLORE` | 只读：读取允许，写入和命令拒绝 | 代码探索 |
| `BYPASS` | 允许一切，但 deny/ask 规则仍然执行 | 受信沙箱 |
| `DONT_ASK` | 将 ASK 转换为 DENY | 无人值守执行 |

## 设置权限模式

```python
from agentscope.agent import Agent
from agentscope.state import AgentState
from agentscope.permission import PermissionContext, PermissionMode

agent = Agent(
    name="safe_agent",
    system_prompt="...",
    model=model,
    state=AgentState(
        permission_context=PermissionContext(mode=PermissionMode.DEFAULT),
    ),
)
```

### 场景示例

```python
# 代码探索 — 只允许读操作
agent = Agent(
    name="explorer",
    system_prompt="你是一个代码分析助手。",
    model=model,
    toolkit=Toolkit(tools=[Read(), Glob(), Grep()]),
    state=AgentState(
        permission_context=PermissionContext(mode=PermissionMode.EXPLORE),
    ),
)

# 开发模式 — 允许文件编辑
agent = Agent(
    name="developer",
    system_prompt="你是一个开发助手。",
    model=model,
    toolkit=Toolkit(tools=[Bash(), Read(), Write(), Edit()]),
    state=AgentState(
        permission_context=PermissionContext(mode=PermissionMode.ACCEPT_EDITS),
    ),
)

# 自动化 CI — 无人值守
agent = Agent(
    name="ci_agent",
    system_prompt="你是一个 CI 自动化助手。",
    model=model,
    state=AgentState(
        permission_context=PermissionContext(mode=PermissionMode.DONT_ASK),
    ),
)
```

## 权限规则

### PermissionRule 字段

| 字段 | 说明 |
|------|------|
| `tool_name` | 目标工具名称 |
| `rule_content` | 匹配规则（如 `"npm run:*"`） |
| `behavior` | `PermissionBehavior.ALLOW` / `DENY` / `ASK` |
| `source` | 规则来源标识 |

### 规则配置示例

```python
from agentscope.permission import (
    PermissionContext, PermissionMode, PermissionRule, PermissionBehavior
)

# 无人值守自动化，带显式允许规则
agent = Agent(
    name="ci_agent",
    system_prompt="...",
    model=model,
    state=AgentState(
        permission_context=PermissionContext(
            mode=PermissionMode.DONT_ASK,
            allow_rules={
                "Bash": [
                    PermissionRule(
                        tool_name="Bash",
                        rule_content="npm run:*",
                        behavior=PermissionBehavior.ALLOW,
                        source="project",
                    ),
                    PermissionRule(
                        tool_name="Bash",
                        rule_content="git status",
                        behavior=PermissionBehavior.ALLOW,
                        source="project",
                    ),
                ],
            },
        )
    ),
)
```

### 拒绝规则示例

```python
deny_rules = {
    "Bash": [
        PermissionRule(
            tool_name="Bash",
            rule_content="rm -rf *",
            behavior=PermissionBehavior.DENY,
            source="safety",
        ),
        PermissionRule(
            tool_name="Bash",
            rule_content="sudo *",
            behavior=PermissionBehavior.DENY,
            source="safety",
        ),
    ],
}
```

## 工具级权限检查

自定义工具可以覆盖 `check_permissions` 方法：

```python
from agentscope.tool import ToolBase, ToolChunk
from agentscope.permission import PermissionContext, PermissionDecision, PermissionBehavior
from agentscope.message import TextBlock

class DatabaseTool(ToolBase):
    name = "DatabaseQuery"
    description = "执行数据库查询。"
    input_schema = {
        "type": "object",
        "properties": {
            "sql": {"type": "string", "description": "SQL 查询语句。"},
        },
        "required": ["sql"],
    }
    is_concurrency_safe = True
    is_read_only = False

    async def check_permissions(self, tool_input: dict, context: PermissionContext) -> PermissionDecision:
        sql = tool_input.get("sql", "").upper()
        # DROP/DELETE 操作需要用户确认
        if any(keyword in sql for keyword in ["DROP", "DELETE", "TRUNCATE"]):
            return PermissionDecision(
                behavior=PermissionBehavior.ASK,
                message="此操作会修改或删除数据，需要确认。"
            )
        # SELECT 查询自动允许
        if sql.strip().startswith("SELECT"):
            return PermissionDecision(
                behavior=PermissionBehavior.ALLOW,
                message="只读查询，自动允许。"
            )
        return PermissionDecision(
            behavior=PermissionBehavior.ASK,
            message="未知操作类型，需要确认。"
        )

    async def __call__(self, sql: str) -> ToolChunk:
        result = execute_sql(sql)
        return ToolChunk(content=[TextBlock(text=result)])
```

## 关键安全特性

> **Deny 规则和危险路径检查是 bypass-immune 的** — 即使在 BYPASS 模式下也会执行。

这意味着：
- `PermissionBehavior.DENY` 规则始终生效
- 危险路径检查（如 `/etc/passwd`、`rm -rf /`）始终生效
- BYPASS 模式只绕过 Allow/Ask 规则的默认行为

## 人机协作权限流程

当权限决策为 `ASK` 时：

1. Agent 发出 `RequireUserConfirmEvent` 事件
2. 前端/客户端显示确认对话框
3. 用户选择允许或拒绝
4. 客户端发送 `UserConfirmResultEvent`
5. Agent 根据用户决策继续或中断

```python
# 流式处理中的权限确认
async for event in agent.reply_stream(user_msg):
    if isinstance(event, RequireUserConfirmEvent):
        print(f"需要确认: {event.message}")
        # 用户在 UI 中确认后，Agent 继续执行
```
