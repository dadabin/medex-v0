# 5. 工具与工具包

## 三个工具概念

| 概念 | 说明 |
|------|------|
| **Tool** | 满足 `ToolBase` 接口的任意类 |
| **Toolkit** | 注册工具、MCP 客户端和技能的容器 |
| **Tool Group** | 可按名称激活/停用的工具束 |

## 内置工具

| 工具 | 说明 | 只读 |
|------|------|------|
| `Bash` | 执行 shell 命令 | 否 |
| `Read` | 读取文件内容（带行号） | 是 |
| `Write` | 创建或覆盖文件 | 否 |
| `Edit` | 文件内精确字符串替换 | 否 |
| `Glob` | 按 glob 模式查找文件 | 是 |
| `Grep` | 使用 ripgrep 搜索文件内容 | 是 |
| `TaskCreate` | 创建结构化任务 | 否 |
| `TaskGet` | 按 ID 获取任务详情 | 是 |
| `TaskList` | 列出所有任务和状态 | 是 |
| `TaskUpdate` | 更新任务状态或元数据 | 否 |

### 使用内置工具

```python
from agentscope.tool import Toolkit, Bash, Read, Write, Edit, Glob, Grep

toolkit = Toolkit(
    tools=[
        Bash(),
        Read(),
        Write(),
        Edit(),
        Glob(),
        Grep(),
    ]
)
```

## 自定义工具（ToolBase 子类）

### 完整示例

```python
from agentscope.tool import ToolBase, ToolChunk
from agentscope.permission import PermissionContext, PermissionDecision, PermissionBehavior
from agentscope.message import TextBlock

class WebSearch(ToolBase):
    name = "WebSearch"
    description = "在网络上搜索信息。"
    input_schema = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索查询词。",
            },
        },
        "required": ["query"],
    }
    is_concurrency_safe = True
    is_read_only = True

    async def check_permissions(self, tool_input: dict, context: PermissionContext) -> PermissionDecision:
        return PermissionDecision(
            behavior=PermissionBehavior.ALLOW,
            message="只读操作，自动允许。"
        )

    async def __call__(self, query: str) -> ToolChunk:
        # 实际搜索逻辑
        results = await do_search(query)
        return ToolChunk(content=[TextBlock(text=results)])
```

### ToolBase 关键属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | `str` | 工具名称（模型调用时使用） |
| `description` | `str` | 工具描述（模型理解工具用途） |
| `input_schema` | `dict` | JSON Schema 格式的输入定义 |
| `is_concurrency_safe` | `bool` | 是否并发安全 |
| `is_read_only` | `bool` | 是否只读（影响权限默认行为） |
| `is_external_tool` | `bool` | 是否外部执行工具（默认 False） |

### ToolBase 关键方法

| 方法 | 说明 |
|------|------|
| `check_permissions(tool_input, context)` | 权限检查，返回 PermissionDecision |
| `__call__(**kwargs)` | 工具执行逻辑，返回 ToolChunk |

## FunctionTool 适配器（轻量级）

将普通 Python 函数快速包装为工具：

```python
from agentscope.tool import FunctionTool, Toolkit

def get_weather(city: str, unit: str = "celsius") -> str:
    """获取指定城市的当前天气。

    Args:
        city: 城市名称。
        unit: 温度单位，"celsius" 或 "fahrenheit"。
    """
    return f"{city}的天气是22度{unit[0].upper()}"

def calculate(expression: str) -> str:
    """计算数学表达式的结果。

    Args:
        expression: 数学表达式字符串。
    """
    try:
        result = eval(expression)  # 注意：生产环境应使用安全的解析器
        return f"结果: {result}"
    except Exception as e:
        return f"计算错误: {e}"

toolkit = Toolkit(tools=[
    FunctionTool(get_weather),
    FunctionTool(calculate),
])
```

**注意：** FunctionTool 包装的函数默认权限行为为 `ASK`（需要用户确认）。

### FunctionTool 的 docstring 约定

- 函数的 docstring 会被解析为工具的 `description`
- `Args:` 部分的参数说明会被解析为输入 schema 的字段描述
- 类型注解会被映射为 JSON Schema 类型

## 外部执行工具

将执行委托给 Agent 运行时之外的环境。设置 `is_external_tool = True`，无需实现 `__call__`。

```python
class HumanApproval(ToolBase):
    name = "HumanApproval"
    description = "请求人类审批敏感操作。"
    input_schema = {
        "type": "object",
        "properties": {
            "action": {
                "type": "string",
                "description": "需要审批的操作描述。",
            },
        },
        "required": ["action"],
    }
    is_concurrency_safe = True
    is_read_only = False
    is_external_tool = True  # 关键：标记为外部执行

    async def check_permissions(self, tool_input: dict, context: PermissionContext) -> PermissionDecision:
        return PermissionDecision(
            behavior=PermissionBehavior.ALLOW,
            message="外部调度允许。"
        )
    # 无需 __call__ 方法
```

执行流程：
1. 模型调用该工具
2. Agent 发出 `RequireExternalExecutionEvent` 事件
3. 暂停等待
4. 外部系统处理完毕后发送 `ExternalExecutionResultEvent`
5. Agent 继续执行

## Tool Group（工具组）

将工具分区为命名束，运行时通过 `reset_tools` 元工具切换：

```python
from agentscope.tool import Toolkit, ToolGroup, Bash, Read, Write, Edit

toolkit = Toolkit(
    tools=[Bash(), Read(), Write(), Edit()],
    tool_groups=[
        ToolGroup(
            name="database",
            description="数据库操作工具。",
            instructions="所有修改操作必须在事务中执行。",
            tools=[db_query_tool, db_migrate_tool],
        ),
        ToolGroup(
            name="deployment",
            description="服务部署工具。",
            instructions="部署前确认目标环境。",
            tools=[deploy_tool, rollback_tool],
        ),
    ],
)
```

**重要：** `"basic"` 组始终激活。每次 `reset_tools` 调用代表所有组的**最终状态**，而非增量变更。未显式设置为 True 的组将变为不活跃。

## Toolkit 完整配置

```python
from agentscope.tool import Toolkit, Bash, Read, Write, Edit
from agentscope.mcp import MCPClient

toolkit = Toolkit(
    # 内置工具
    tools=[Bash(), Read(), Write(), Edit()],

    # MCP 客户端
    mcps=[mcp_client],

    # 技能加载路径
    skills_or_loaders=["/path/to/skills"],

    # 工具组
    tool_groups=[
        ToolGroup(name="db", description="数据库", instructions="...", tools=[...]),
    ],
)

agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    toolkit=toolkit,
)
```
