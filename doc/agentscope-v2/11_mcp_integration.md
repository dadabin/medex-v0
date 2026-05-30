# 11. MCP 集成

## 概述

AgentScope v2 原生支持 MCP（Model Context Protocol），通过 `MCPClient` 连接外部 MCP 服务器，将其工具注册到 Agent 的工具包中。

## 两种连接模式

| 模式 | 传输方式 | 生命周期 | 适用场景 |
|------|----------|----------|----------|
| **Stateful** | STDIO 或 HTTP | 持久连接，维护状态 | 需要状态的工具 |
| **Stateless** | HTTP only | 每次调用临时连接 | 无状态工具 |

## Stateful STDIO 连接

通过标准输入输出启动 MCP 服务器进程：

```python
from agentscope.mcp import MCPClient, StdioMCPConfig

client = MCPClient(
    name="filesystem",
    is_stateful=True,
    mcp_config=StdioMCPConfig(
        command="mcp-server-filesystem",
        args=["--root", "/my/project"],
    ),
)
await client.connect()

# 将 MCP 客户端添加到工具包
toolkit = Toolkit(mcps=[client])
```

## Stateful HTTP 连接

通过 HTTP 连接远程 MCP 服务器：

```python
from agentscope.mcp import MCPClient, HttpMCPConfig

client = MCPClient(
    name="weather",
    is_stateful=True,
    mcp_config=HttpMCPConfig(
        url="https://api.weather.com/mcp",
        headers={"Authorization": "Bearer xxx"},
    ),
)
await client.connect()

toolkit = Toolkit(mcps=[client])
```

## Stateless HTTP 连接

每次调用临时创建连接，无需手动 connect：

```python
from agentscope.mcp import MCPClient, HttpMCPConfig

client = MCPClient(
    name="search",
    is_stateful=False,
    mcp_config=HttpMCPConfig(url="https://api.search.com/mcp"),
)

toolkit = Toolkit(mcps=[client])
# 无需 await client.connect()
```

## 工具过滤

可以选择性启用 MCP 服务器上的特定工具：

```python
client = MCPClient(
    name="search",
    is_stateful=False,
    mcp_config=HttpMCPConfig(url="https://api.search.com/mcp"),
    enable_tools=["web_search", "image_search"],  # 只启用这两个工具
)
```

## 工具命名空间

MCP 工具自动命名为 `mcp__{server_name}__{tool_name}`：

```python
# 服务器名: "filesystem"，工具名: "read_file"
# Agent 调用时使用: "mcp__filesystem__read_file"

# 服务器名: "weather"，工具名: "get_forecast"
# Agent 调用时使用: "mcp__weather__get_forecast"
```

## 只读工具自动允许

MCP 工具如果标注了 `readOnlyHint`，会被自动允许（不需要用户确认）。

## 完整示例：多 MCP 集成

```python
import os
import asyncio
from agentscope.agent import Agent
from agentscope.tool import Toolkit, Bash, Read
from agentscope.mcp import MCPClient, StdioMCPConfig, HttpMCPConfig
from agentscope.credential import OpenAICredential
from agentscope.model import OpenAIChatModel
from agentscope.message import UserMsg

async def main():
    # 文件系统 MCP（STDIO）
    fs_client = MCPClient(
        name="filesystem",
        is_stateful=True,
        mcp_config=StdioMCPConfig(
            command="mcp-server-filesystem",
            args=["--root", "/my/project"],
        ),
    )
    await fs_client.connect()

    # 天气 API MCP（HTTP）
    weather_client = MCPClient(
        name="weather",
        is_stateful=True,
        mcp_config=HttpMCPConfig(
            url="https://api.weather.com/mcp",
            headers={"Authorization": "Bearer xxx"},
        ),
    )
    await weather_client.connect()

    # 搜索 MCP（Stateless HTTP）
    search_client = MCPClient(
        name="search",
        is_stateful=False,
        mcp_config=HttpMCPConfig(url="https://api.search.com/mcp"),
        enable_tools=["web_search"],  # 只启用 web_search
    )

    toolkit = Toolkit(
        tools=[Bash(), Read()],
        mcps=[fs_client, weather_client, search_client],
    )

    agent = Agent(
        name="multi_mcp_agent",
        system_prompt="你是一个多功能助手，可以操作文件、查天气和搜索。",
        model=OpenAIChatModel(
            credential=OpenAICredential(api_key=os.environ["OPENAI_API_KEY"]),
            model="gpt-4.1",
        ),
        toolkit=toolkit,
    )

    result = await agent.reply(
        UserMsg(name="user", content="查看项目结构，然后查一下北京的天气")
    )
    print(result.get_text_content())

asyncio.run(main())
```

## Docker/E2B Workspace 中的 MCP

在 Docker 或 E2B Workspace 中，MCP 服务器运行在容器/沙箱内部。Agent 端通过 `GatewayMCPClient`（`MCPClient` 子类）连接，使用方式与本地 MCP 客户端一致。

```python
from agentscope.workspace import DockerWorkspace
from agentscope.mcp import GatewayMCPClient

workspace = DockerWorkspace(
    base_image="python:3.11-slim",
    workdir="/data/agent-workspace",
)

# GatewayMCPClient 通过容器内的 MCP Gateway 连接
client = GatewayMCPClient(
    name="in_container_mcp",
    # ... 配置
)
```

## MCP 与 Tool Group 结合

```python
from agentscope.tool import Toolkit, ToolGroup

toolkit = Toolkit(
    mcps=[fs_client, db_client],
    tool_groups=[
        ToolGroup(
            name="filesystem",
            description="文件操作工具。",
            instructions="操作前确认目标路径。",
            # MCP 工具也可以加入工具组
        ),
        ToolGroup(
            name="database",
            description="数据库操作工具。",
            instructions="所有修改必须在事务中执行。",
        ),
    ],
)
```
