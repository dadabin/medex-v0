# 10. Workspace 与分布式部署

## 概述

Workspace 是 Agent 的执行环境，提供工具、技能、MCP 服务器和上下文卸载能力。三种实现共享同一接口，切换只需修改一行代码。

## 三种 Workspace

| 类型 | 说明 | 安全性 | 适用场景 |
|------|------|--------|----------|
| `LocalWorkspace` | 本地文件系统 | 低 | 开发调试 |
| `DockerWorkspace` | Docker 容器隔离 | 中 | 生产部署 |
| `E2BWorkspace` | E2B 云沙箱 | 高 | 多租户/不可信代码 |

## LocalWorkspace

最简单的 Workspace，直接在本地文件系统操作。

```python
from agentscope.workspace import LocalWorkspace

workspace = LocalWorkspace(workdir="/data/my-workspace")
```

文件结构：
```
/data/my-workspace/
  sessions/{session_id}/
    context.jsonl              # 压缩的消息
    tool_result-{tool_id}.txt  # 截断的工具结果
```

### 作为 Offloader 使用

```python
agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    offloader=workspace,
)
```

## DockerWorkspace

在 Docker 容器中执行，提供隔离的运行环境。

```python
from agentscope.workspace import DockerWorkspace

workspace = DockerWorkspace(
    base_image="python:3.11-slim",
    workdir="/data/docker-workspaces/agent-1",
    node_version="20",
    extra_pip=["numpy", "pandas"],
)
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `base_image` | Docker 基础镜像 |
| `workdir` | 容器内工作目录 |
| `node_version` | Node.js 版本（如需要） |
| `extra_pip` | 额外安装的 Python 包 |

### MCP Gateway

对于 Docker/E2B Workspace，容器内运行一个轻量 FastAPI 进程，通过认证 HTTP 暴露 MCP 服务器。Agent 端使用 `GatewayMCPClient`（`MCPClient` 子类），使用方式与本地 MCP 客户端一致。

## E2BWorkspace

使用 E2B 云沙箱，提供最高级别的隔离。

```python
from agentscope.workspace import E2BWorkspace

workspace = E2BWorkspace(
    template="base",
    api_key="your-e2b-api-key",
    timeout_seconds=300,
)
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `template` | E2B 沙箱模板 |
| `api_key` | E2B API Key |
| `timeout_seconds` | 沙箱超时时间 |

## 切换 Workspace

三种 Workspace 共享 `WorkspaceBase` 接口，Agent 代码无需修改：

```python
# 开发环境
workspace = LocalWorkspace(workdir="/data/dev")

# 切换到 Docker（只需改这一行）
workspace = DockerWorkspace(base_image="python:3.11-slim", workdir="/data/prod")

# 切换到 E2B（只需改这一行）
workspace = E2BWorkspace(template="base", api_key="...")

# Agent 代码不变
agent = Agent(
    name="my_agent",
    system_prompt="...",
    model=model,
    offloader=workspace,
)
```

## 自定义 Workspace

```python
from agentscope.workspace import WorkspaceBase

class MyWorkspace(WorkspaceBase):
    # 实现 WorkspaceBase 接口
    ...
```

## v1 分布式模式对比

### v1：基于 Actor 的分布式

- `RpcAgent` / `RpcDialogAgent` / `RpcUserAgent` — 分布式 Agent 变体
- `RpcAgentServerLauncher` — 一键在远程机器部署 Agent 服务器
- Placeholder 消息 — 允许主进程非阻塞继续
- AgentScope Studio — 统一的分布式应用展示

### v2：Workspace + Agent Service

- **Workspace 系统** — Local/Docker/E2B，一行切换
- **Agent Service** — FastAPI 多租户 HTTP 服务，详见 [13_agent_service.md](13_agent_service.md)
- **A2A 协议** — Agent 间通信标准

v2 的分布式架构更侧重于安全隔离和多租户支持，而非 v1 的进程级分布式。
