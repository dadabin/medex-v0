# 16. AgentScope 整合腾讯云 Agent 沙箱方案

## 概述

本文档详细说明如何将腾讯云的容器/沙箱服务与 AgentScope v2 的 Workspace 系统整合，实现国产化的 Agent 安全执行环境。

## 一、AgentScope Workspace 架构分析

### WorkspaceBase 接口

AgentScope v2 的 Workspace 系统基于 `WorkspaceBase` 抽象类，定义了 12 个必须实现的方法：

```python
class WorkspaceBase:
    workspace_id: str
    is_alive: bool

    # 生命周期
    async def initialize(self) -> None: ...   # 分配资源、连接 MCP、复制技能
    async def close(self) -> None: ...         # 释放所有资源
    async def reset(self) -> None: ...         # 重置为干净状态（可选覆盖）

    # 上下文管理器
    async def __aenter__(self) -> Self: ...    # 调用 initialize()
    async def __aexit__(self, *exc) -> None: ...  # 调用 close()

    # 信息获取
    async def get_instructions(self) -> str: ...     # 返回 workspace 特定的系统提示词片段

    # 工具与资源发现
    async def list_tools(self) -> list[ToolBase]: ...   # 此 workspace 的内置工具
    async def list_mcps(self) -> list[MCPClient]: ...   # 活跃的 MCP 客户端
    async def list_skills(self) -> list[Skill]: ...     # 可用技能

    # 上下文卸载
    async def offload_context(self, session_id: str, msgs: list[Msg]) -> str: ...
    async def offload_tool_result(self, session_id: str, tool_result: ToolResultBlock) -> str: ...

    # 动态管理
    async def add_mcp(self, mcp_client: MCPClient) -> None: ...
    async def remove_mcp(self, name: str) -> None: ...
    async def add_skill(self, skill_path: str) -> None: ...
    async def remove_skill(self, name: str) -> None: ...
```

### 两种实现模式

| 模式 | 代表 | MCP 运行位置 | list_tools() | 适用场景 |
|------|------|-------------|-------------|----------|
| **直接 MCP** | LocalWorkspace | 宿主机 | 返回内置工具列表 | 本地开发 |
| **Gateway MCP** | DockerWorkspace / E2BWorkspace | 沙箱内部 | 返回 `[]` | 云端沙箱 |

### Gateway 模式架构

```
宿主机 (Host)                         沙箱 (Sandbox)
┌──────────────┐                     ┌─────────────────────────┐
│  Agent       │                     │  MCP Gateway (FastAPI)   │
│    │         │   HTTP (Bearer)     │    ├── MCP Server 1      │
│    ├── GatewayClient ──────────►  │    ├── MCP Server 2      │
│    │         │                     │    └── MCP Server N      │
│    └── GatewayMCPClient           │                          │
│         ├── list_raw_tools()  ──► │  GET /mcps/{name}/tools  │
│         └── __call__(kwargs)  ──► │  POST /mcps/{name}/tools/{tool} │
└──────────────┘                     └─────────────────────────┘
```

### MCP Gateway 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/health` | GET | 健康检查（无需认证） |
| `/mcps` | GET | 列出所有已注册 MCP |
| `/mcps` | POST | 注册新 MCP |
| `/mcps/{name}` | DELETE | 注销并关闭 MCP |
| `/mcps/{name}/tools` | GET | 列出上游工具 Schema |
| `/mcps/{name}/tools/{tool}` | POST | 调用工具 |

### Gateway 配置格式

```json
{
  "token": "bearer-token",
  "servers": [<MCPClient.model_dump()>, ...]
}
```

### Offloader 协议

```python
class Offloader(Protocol):
    async def offload_context(self, session_id: str, msgs: list[Msg]) -> str: ...
    async def offload_tool_result(self, session_id: str, tool_result: ToolResultBlock) -> str: ...
```

`WorkspaceBase` 本身实现了这两个方法，所以每个 Workspace 自动满足 Offloader 协议。

## 二、整合方案设计

### 推荐方案：基于 ECI 的 TencentCloudWorkspace

选择 ECI（弹性容器实例）作为沙箱后端，原因：

1. **免运维** — 不需要管理 K8s 集群
2. **秒级启动** — 适合 Agent 交互场景
3. **按量计费** — 只为实际运行时间付费
4. **API 完善** — 官方 Python SDK 成熟
5. **安全隔离** — 每个容器实例独立隔离
6. **国内部署** — 低延迟，数据合规

### 架构设计

```
┌─────────────────────────────────────────────────────┐
│  AgentScope (宿主机)                                 │
│                                                     │
│  ┌─────────────┐     ┌──────────────────────┐      │
│  │  Agent      │────►│ TencentCloudWorkspace │      │
│  │             │     │   ├── GatewayClient    │      │
│  │             │     │   ├── GatewayMCPClient │      │
│  │             │     │   └── ECI SDK Client   │      │
│  └─────────────┘     └──────────┬─────────────┘      │
│                                 │ HTTP (Bearer)       │
└─────────────────────────────────┼────────────────────┘
                                  │
                    腾讯云 ECI 容器实例
┌─────────────────────────────────┼────────────────────┐
│                                 ▼                    │
│  ┌─────────────────────────────────────────────┐    │
│  │  MCP Gateway (FastAPI, :5600)               │    │
│  │    ├── filesystem MCP Server                │    │
│  │    ├── bash MCP Server                      │    │
│  │    └── ... (自定义 MCP)                      │    │
│  │                                             │    │
│  │  /workspace/                                │    │
│  │    ├── skills/                              │    │
│  │    ├── sessions/                            │    │
│  │    ├── data/                                │    │
│  │    └── .mcp                                 │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

## 三、实现方案

### 3.1 项目结构

```
agentscope_tencent_cloud/
├── __init__.py
├── _tencent_cloud_workspace.py      # 核心 Workspace 实现
├── _eci_client.py                    # ECI API 封装
├── _workspace_manager.py             # Workspace 生命周期管理
└── README.md
```

### 3.2 TencentCloudWorkspace 实现

```python
"""
腾讯云 ECI Workspace — AgentScope v2 自定义 Workspace 实现

参考 E2BWorkspace 实现，使用腾讯云 ECI 作为沙箱后端。
"""

import asyncio
import hashlib
import json
import mimetypes
import os
import posixpath
import shlex
import uuid
from copy import deepcopy
from typing import Any

from agentscope.workspace._base import WorkspaceBase
from agentscope.workspace._gateway_client import GatewayClient, GatewayMCPClient
from agentscope.workspace._utils import (
    _read_gateway_script_bytes,
    _agentscope_version,
    _is_released_install,
    _GATEWAY_BASE_REQUIREMENTS,
)
from agentscope.mcp import MCPClient
from agentscope.message import DataBlock, Base64Source, URLSource
from agentscope.skill import Skill

# 沙箱内部路径常量
SANDBOX_WORKDIR = "/workspace"
SANDBOX_SESSIONS_DIR = f"{SANDBOX_WORKDIR}/sessions"
SANDBOX_DATA_DIR = f"{SANDBOX_WORKDIR}/data"
SANDBOX_SKILLS_DIR = f"{SANDBOX_WORKDIR}/skills"
SANDBOX_MCP_FILE = f"{SANDBOX_WORKDIR}/.mcp"
GATEWAY_HOME = f"{SANDBOX_WORKDIR}/.gateway"
GATEWAY_VENV = f"{GATEWAY_HOME}/venv"
GATEWAY_VENV_PY = f"{GATEWAY_VENV}/bin/python"
GATEWAY_SCRIPT = f"{GATEWAY_HOME}/gateway.py"
GATEWAY_CONFIG = f"{GATEWAY_HOME}/config.json"
GATEWAY_LOG = f"{GATEWAY_HOME}/gateway.log"
DEFAULT_GATEWAY_PORT = 5600

_DEFAULT_INSTRUCTIONS = (
    "你的工作目录是 {workdir}，所有文件操作应在此目录下进行。"
)


class TencentCloudWorkspace(WorkspaceBase):
    """基于腾讯云 ECI 的 AgentScope Workspace。

    在 ECI 容器实例中运行 MCP Gateway，Agent 通过 HTTP 与沙箱交互。
    """

    def __init__(
        self,
        *,
        workspace_id: str | None = None,
        # 腾讯云凭证
        secret_id: str = "",
        secret_key: str = "",
        region: str = "ap-guangzhou",
        # ECI 配置
        container_image: str = "python:3.11-slim",
        cpu: str = "2",
        memory: str = "4Gi",
        # Gateway 配置
        gateway_port: int = DEFAULT_GATEWAY_PORT,
        # 通用配置
        env: dict[str, str] | None = None,
        extra_pip: list[str] | None = None,
        instructions: str = _DEFAULT_INSTRUCTIONS,
        default_mcps: list[MCPClient] | None = None,
        skill_paths: list[str] | None = None,
    ) -> None:
        super().__init__(workspace_id=workspace_id)

        # 腾讯云凭证
        self.secret_id = secret_id
        self.secret_key = secret_key
        self.region = region

        # ECI 配置
        self.container_image = container_image
        self.cpu = cpu
        self.memory = memory

        # Gateway 配置
        self.gateway_port = gateway_port

        # 通用配置
        self.env: dict[str, str] = dict(env or {})
        self.extra_pip: list[str] = list(extra_pip or [])
        self.instructions = instructions
        self.default_mcps: list[MCPClient] = list(default_mcps or [])
        self.skill_paths: list[str] = list(skill_paths or [])

        # 运行时状态
        self._container_group_id: str | None = None
        self._gateway: GatewayClient | None = None
        self._gateway_token: str = ""
        self._mcps: list[MCPClient] = []
        self._gateway_clients: dict[str, GatewayMCPClient] = {}
        self._mcp_lock = asyncio.Lock()
        self._skill_lock = asyncio.Lock()

    # ─── 生命周期 ──────────────────────────────────

    async def initialize(self) -> None:
        """初始化：创建/连接 ECI 容器实例，启动 MCP Gateway。"""
        if self.is_alive:
            return

        # 1. 创建或重连 ECI 容器实例
        await self._create_or_attach_container()

        # 2. 引导安装（首次）
        if not await self._file_exists(GATEWAY_SCRIPT):
            await self._exec(f"mkdir -p {shlex.quote(SANDBOX_WORKDIR)}")
            await self._run_bootstrap()

        # 3. 恢复或初始化 MCP 配置
        self._mcps = await self._restore_or_seed_mcps()

        # 4. 生成 Gateway Token
        self._gateway_token = uuid.uuid4().hex

        # 5. 启动 Gateway
        await self._exec("pkill -f gateway.py || true")
        await self._write_gateway_config()
        await self._start_gateway_process()

        # 6. 连接 Gateway
        gateway_url = await self._get_gateway_url()
        self._gateway = GatewayClient(
            base_url=gateway_url,
            token=self._gateway_token,
            timeout=30.0,
        )
        await self._wait_for_gateway()

        # 7. 获取 Gateway 侧的 MCP Client
        self._gateway_clients = {
            c.name: c for c in await self._gateway.list_mcps()
        }

        # 8. 保存 MCP 配置，植入技能
        await self._save_mcp_file()
        await self._seed_skills()

        self.is_alive = True

    async def close(self) -> None:
        """关闭：释放 Gateway 连接，停止 ECI 容器实例。"""
        if self._gateway is not None:
            try:
                await self._gateway.aclose()
            except Exception:
                pass
            self._gateway = None
        self._gateway_clients.clear()

        if self._container_group_id is not None:
            try:
                await self._delete_container_group()
            except Exception as e:
                import logging
                logging.warning(f"TencentCloudWorkspace: 停止容器失败: {e}")
            self._container_group_id = None

        self.is_alive = False

    async def reset(self) -> None:
        """重置沙箱到干净状态。"""
        async with self._mcp_lock, self._skill_lock:
            for gw_client in list(self._gateway_clients.values()):
                try:
                    await gw_client.close()
                except Exception:
                    pass
            self._gateway_clients.clear()
            self._mcps = []

            paths = [SANDBOX_SESSIONS_DIR, SANDBOX_DATA_DIR, SANDBOX_SKILLS_DIR]
            await self._exec("rm -rf " + " ".join(shlex.quote(p) for p in paths))
            await self._save_mcp_file()

    # ─── 信息获取 ──────────────────────────────────

    async def get_instructions(self) -> str:
        return self.instructions.format(workdir=SANDBOX_WORKDIR)

    # ─── 工具与资源发现 ──────────────────────────────

    async def list_tools(self) -> list:
        """Gateway 模式：所有工具通过 MCP 提供。"""
        return []

    async def list_mcps(self) -> list[MCPClient]:
        return list(self._gateway_clients.values())

    async def list_skills(self) -> list[Skill]:
        import frontmatter as fm

        result = await self._exec(
            f"find {SANDBOX_SKILLS_DIR} -name SKILL.md 2>/dev/null || true"
        )
        if not result.ok():
            return []
        listing = result.stdout.decode(errors="replace").strip()
        if not listing:
            return []

        skills: list[Skill] = []
        for md_path in (line.strip() for line in listing.split("\n")):
            if not md_path:
                continue
            try:
                raw = await self._read(md_path)
                doc = fm.loads(raw.decode("utf-8"))
                name = doc.get("name")
                desc = doc.get("description")
                if not name or not desc:
                    continue
                skills.append(Skill(
                    name=str(name),
                    description=str(desc),
                    dir=posixpath.dirname(md_path),
                    markdown=doc.content or "",
                    updated_at=0.0,
                ))
            except Exception:
                pass
        return skills

    # ─── 上下文卸载 ──────────────────────────────────

    async def offload_context(self, session_id: str, msgs: list) -> str:
        base = f"{SANDBOX_SESSIONS_DIR}/{session_id}"
        path = f"{base}/context.jsonl"

        copied = deepcopy(msgs)
        lines: list[str] = []
        for msg in copied:
            if not isinstance(msg.content, str):
                content = []
                for block in msg.content:
                    if isinstance(block, DataBlock) and isinstance(block.source, Base64Source):
                        block = await self._offload_data_block(block)
                    content.append(block)
                msg.content = content
            lines.append(msg.model_dump_json())

        await self._exec(f"mkdir -p {shlex.quote(base)}")
        existing = b""
        try:
            existing = await self._read(path)
        except FileNotFoundError:
            pass
        await self._write(path, existing + ("\n".join(lines) + "\n").encode("utf-8"))
        return path

    async def offload_tool_result(self, session_id: str, tool_result) -> str:
        base = f"{SANDBOX_SESSIONS_DIR}/{session_id}"
        path = f"{base}/tool_result-{tool_result.id}.txt"

        parts: list[str] = []
        if isinstance(tool_result.output, str):
            parts.append(tool_result.output)
        else:
            from agentscope.message import TextBlock
            for block in tool_result.output:
                if isinstance(block, TextBlock):
                    parts.append(block.text)

        await self._exec(f"mkdir -p {shlex.quote(base)}")
        await self._write(path, "".join(parts).encode("utf-8"))
        return path

    # ─── 动态 MCP 管理 ──────────────────────────────

    async def add_mcp(self, mcp_client: MCPClient) -> None:
        async with self._mcp_lock:
            if mcp_client.name in self._gateway_clients:
                raise ValueError(f"MCP {mcp_client.name!r} 已存在。")
            spec = mcp_client.model_dump(mode="json")
            assert self._gateway is not None
            gw_client = self._gateway.make_client(spec)
            await gw_client.connect()
            self._mcps.append(mcp_client)
            self._gateway_clients[gw_client.name] = gw_client
            await self._save_mcp_file()

    async def remove_mcp(self, name: str) -> None:
        async with self._mcp_lock:
            gw_client = self._gateway_clients.pop(name, None)
            if gw_client is None:
                return
            try:
                await gw_client.close()
            except Exception:
                pass
            self._mcps = [m for m in self._mcps if m.name != name]
            await self._save_mcp_file()

    # ─── 动态技能管理 ──────────────────────────────

    async def add_skill(self, skill_path: str) -> None:
        skill_md = os.path.join(skill_path, "SKILL.md")
        if not os.path.isfile(skill_md):
            raise ValueError(f"无效技能路径: {skill_path!r}")

        async with self._skill_lock:
            await self._exec(f"mkdir -p {SANDBOX_SKILLS_DIR}")
            dir_name = os.path.basename(os.path.abspath(skill_path))

            for root, _dirs, files in os.walk(skill_path):
                for fname in files:
                    local = os.path.join(root, fname)
                    rel = os.path.relpath(local, skill_path)
                    remote = f"{SANDBOX_SKILLS_DIR}/{dir_name}/{rel}"
                    with open(local, "rb") as f:
                        data = f.read()
                    await self._write(remote, data)

    async def remove_skill(self, name: str) -> None:
        skills = await self.list_skills()
        target_dir = None
        for s in skills:
            if s.name == name:
                target_dir = s.dir
                break
        if target_dir is None:
            raise KeyError(f"技能 {name!r} 未找到。")
        await self._exec(f"rm -rf {shlex.quote(target_dir)}")

    # ─── ECI 操作（核心实现）──────────────────────────

    async def _create_or_attach_container(self) -> None:
        """创建 ECI 容器组或重连已有容器。"""
        from tencentcloud.common import credential
        from tencentcloud.eci.v20180508 import eci_client, models

        cred = credential.Credential(self.secret_id, self.secret_key)
        client = eci_client.EciClient(cred, self.region)

        # 查找已有容器组
        existing = await self._find_existing_container_group(client)
        if existing is not None:
            self._container_group_id = existing
            return

        # 创建新容器组
        req = models.CreateContainerGroupRequest()
        req.ContainerGroupName = f"agentscope-{self.workspace_id[:12]}"
        req.RestartPolicy = "Never"

        container = models.Container()
        container.Name = "agent-sandbox"
        container.Image = self.container_image
        container.Cpu = self.cpu
        container.Memory = self.memory
        container.Command = ["sleep", "infinity"]

        # 环境变量
        if self.env:
            container.Envs = [
                models.EnvironmentVar(Name=k, Value=v)
                for k, v in self.env.items()
            ]

        # 端口映射（Gateway）
        container.PortMappings = [
            models.PortMapping(
                ContainerPort=self.gateway_port,
                Protocol="TCP",
            )
        ]

        req.Containers = [container]
        resp = client.CreateContainerGroup(req)
        self._container_group_id = resp.ContainerGroupId

        # 等待容器运行
        await self._wait_until_running(client)

    async def _find_existing_container_group(self, client) -> str | None:
        """通过 metadata 查找已有的容器组。"""
        from tencentcloud.eci.v20180508 import models

        req = models.DescribeContainerGroupsRequest()
        # 通过过滤器查找
        req.Filters = [
            models.Filter(Name="ContainerGroupName", Values=[f"agentscope-{self.workspace_id[:12]}"]),
        ]
        resp = client.DescribeContainerGroups(req)

        if resp.ContainerGroupSet:
            return resp.ContainerGroupSet[0].ContainerGroupId
        return None

    async def _wait_until_running(self, client, timeout: float = 120.0) -> None:
        """等待容器组进入运行状态。"""
        from tencentcloud.eci.v20180508 import models

        deadline = asyncio.get_event_loop().time() + timeout
        delay = 2.0

        while asyncio.get_event_loop().time() < deadline:
            req = models.DescribeContainerGroupsRequest()
            req.ContainerGroupIds = [self._container_group_id]
            resp = client.DescribeContainerGroups(req)

            if resp.ContainerGroupSet:
                status = resp.ContainerGroupSet[0].Status
                if status == "Running":
                    return
                if status in ("Failed", "Succeeded"):
                    raise RuntimeError(f"容器组异常状态: {status}")

            await asyncio.sleep(delay)
            delay = min(delay * 1.5, 10.0)

        raise RuntimeError(f"容器组未在 {timeout}s 内就绪")

    async def _delete_container_group(self) -> None:
        """删除 ECI 容器组。"""
        if not self._container_group_id:
            return

        from tencentcloud.common import credential
        from tencentcloud.eci.v20180508 import eci_client, models

        cred = credential.Credential(self.secret_id, self.secret_key)
        client = eci_client.EciClient(cred, self.region)

        req = models.DeleteContainerGroupRequest()
        req.ContainerGroupId = self._container_group_id
        client.DeleteContainerGroup(req)

    async def _get_gateway_url(self) -> str:
        """获取 Gateway 的访问 URL。"""
        from tencentcloud.common import credential
        from tencentcloud.eci.v20180508 import eci_client, models

        cred = credential.Credential(self.secret_id, self.secret_key)
        client = eci_client.EciClient(cred, self.region)

        req = models.DescribeContainerGroupsRequest()
        req.ContainerGroupIds = [self._container_group_id]
        resp = client.DescribeContainerGroups(req)

        if not resp.ContainerGroupSet:
            raise RuntimeError("容器组不存在")

        # ECI 公网 IP
        cg = resp.ContainerGroupSet[0]
        ip = cg.Eip or cg.InnerIp
        if not ip:
            raise RuntimeError("无法获取容器组 IP")

        return f"http://{ip}:{self.gateway_port}"

    # ─── 沙箱 I/O 操作 ──────────────────────────────

    async def _exec(self, command: str, *, timeout: float | None = None) -> "_ExecResult":
        """在 ECI 容器内执行命令。"""
        from tencentcloud.common import credential
        from tencentcloud.eci.v20180508 import eci_client, models

        cred = credential.Credential(self.secret_id, self.secret_key)
        client = eci_client.EciClient(cred, self.region)

        req = models.ExecuteCommandRequest()
        req.ContainerGroupId = self._container_group_id
        req.ContainerName = "agent-sandbox"
        req.Command = command
        req.WorkingDir = SANDBOX_WORKDIR

        if timeout:
            req.Timeout = int(timeout)

        try:
            resp = client.ExecuteCommand(req)
            return _ExecResult(
                exit_code=0,
                stdout=resp.Output.encode("utf-8") if resp.Output else b"",
                stderr=b"",
            )
        except Exception as e:
            return _ExecResult(
                exit_code=-1,
                stdout=b"",
                stderr=str(e).encode("utf-8"),
            )

    async def _read(self, path: str) -> bytes:
        """从容器中读取文件。"""
        result = await self._exec(f"cat {shlex.quote(path)}")
        if not result.ok():
            raise FileNotFoundError(f"文件不存在: {path}")
        return result.stdout

    async def _write(self, path: str, data: bytes) -> None:
        """向容器写入文件。"""
        import base64
        encoded = base64.b64encode(data).decode("ascii")
        dir_path = posixpath.dirname(path)
        await self._exec(f"mkdir -p {shlex.quote(dir_path)}")
        await self._exec(f"echo '{encoded}' | base64 -d > {shlex.quote(path)}")

    async def _file_exists(self, path: str) -> bool:
        """检查容器内文件是否存在。"""
        result = await self._exec(f"test -f {shlex.quote(path)}")
        return result.ok()

    # ─── Gateway 操作 ──────────────────────────────

    async def _run_bootstrap(self) -> None:
        """引导安装：安装 uv、gateway venv、agentscope。"""
        install_cmd = (
            f"pip install agentscope=={_agentscope_version()} --no-deps"
            if _is_released_install()
            else "pip install -e /tmp/agentscope --no-deps"
        )

        commands = [
            # 安装 uv
            "curl -LsSf https://astral.sh/uv/install.sh | sh",
            "export PATH=$HOME/.local/bin:$PATH",
            # 创建 Gateway venv
            f"mkdir -p {shlex.quote(GATEWAY_HOME)}",
            f"uv venv {shlex.quote(GATEWAY_VENV)}",
            # 安装 Gateway 依赖
            f"{shlex.quote(GATEWAY_VENV_PY)} -m pip install "
            + " ".join(_GATEWAY_BASE_REQUIREMENTS),
            # 安装 agentscope（gateway 需要导入模型定义）
            f"{shlex.quote(GATEWAY_VENV_PY)} -m pip install {install_cmd}",
            # 安装额外依赖
        ]

        if self.extra_pip:
            commands.append(
                f"{shlex.quote(GATEWAY_VENV_PY)} -m pip install "
                + " ".join(self.extra_pip)
            )

        for cmd in commands:
            result = await self._exec(cmd, timeout=600.0)
            if not result.ok():
                raise RuntimeError(
                    f"引导安装失败: {cmd!r}\n"
                    f"stderr: {result.stderr.decode(errors='replace')}"
                )

        # 上传 Gateway 脚本
        await self._write(GATEWAY_SCRIPT, _read_gateway_script_bytes())

    async def _restore_or_seed_mcps(self) -> list[MCPClient]:
        """恢复持久化的 MCP 配置或使用默认配置。"""
        try:
            raw = await self._read(SANDBOX_MCP_FILE)
        except FileNotFoundError:
            return list(self.default_mcps)

        try:
            data = json.loads(raw.decode("utf-8"))
            return [MCPClient.model_validate(m) for m in data]
        except Exception:
            return list(self.default_mcps)

    async def _save_mcp_file(self) -> None:
        """持久化 MCP 配置。"""
        payload = json.dumps(
            [m.model_dump(mode="json") for m in self._mcps],
            indent=2,
            ensure_ascii=False,
        )
        await self._exec(f"mkdir -p {shlex.quote(SANDBOX_WORKDIR)}")
        await self._write(SANDBOX_MCP_FILE, payload.encode("utf-8"))

    async def _write_gateway_config(self) -> None:
        """写入 Gateway 配置文件。"""
        cfg = {
            "token": self._gateway_token,
            "servers": [m.model_dump(mode="json") for m in self._mcps],
        }
        await self._exec(f"mkdir -p {shlex.quote(GATEWAY_HOME)}")
        await self._write(
            GATEWAY_CONFIG,
            json.dumps(cfg, indent=2, ensure_ascii=False).encode("utf-8"),
        )

    async def _start_gateway_process(self) -> None:
        """启动 Gateway 进程。"""
        cmd = (
            f"nohup {shlex.quote(GATEWAY_VENV_PY)} -u "
            f"{shlex.quote(GATEWAY_SCRIPT)} "
            f"--config {shlex.quote(GATEWAY_CONFIG)} "
            f"--port {self.gateway_port} "
            f"> {shlex.quote(GATEWAY_LOG)} 2>&1 &"
        )
        await self._exec(cmd)

    async def _wait_for_gateway(self, timeout: float = 30.0) -> None:
        """等待 Gateway 健康检查通过。"""
        assert self._gateway is not None
        deadline = asyncio.get_event_loop().time() + timeout
        delay = 0.5

        while asyncio.get_event_loop().time() < deadline:
            if await self._gateway.health():
                return
            await asyncio.sleep(delay)
            delay = min(delay * 1.5, 3.0)

        raise RuntimeError(f"Gateway 未在 {timeout}s 内就绪")

    async def _seed_skills(self) -> None:
        """植入默认技能。"""
        if not self.skill_paths:
            return
        result = await self._exec(f"ls -A {shlex.quote(SANDBOX_SKILLS_DIR)} 2>/dev/null || true")
        if result.ok() and result.stdout.strip():
            return
        for path in self.skill_paths:
            try:
                await self.add_skill(path)
            except Exception:
                pass

    async def _offload_data_block(self, block: DataBlock) -> DataBlock:
        """将 base64 数据块卸载到文件。"""
        if not isinstance(block.source, Base64Source):
            return block
        h = hashlib.sha256(block.source.data.encode()).hexdigest()
        ext = mimetypes.guess_extension(block.source.media_type) or ".bin"
        path = f"{SANDBOX_DATA_DIR}/{h}{ext}"
        await self._exec(f"mkdir -p {shlex.quote(SANDBOX_DATA_DIR)}")
        import base64
        await self._write(path, base64.b64decode(block.source.data))
        return DataBlock(
            id=block.id,
            name=block.name,
            source=URLSource(
                url=f"file://{path}",
                media_type=block.source.media_type,
            ),
        )


class _ExecResult:
    """命令执行结果。"""
    def __init__(self, exit_code: int, stdout: bytes, stderr: bytes):
        self.exit_code = exit_code
        self.stdout = stdout
        self.stderr = stderr

    def ok(self) -> bool:
        return self.exit_code == 0
```

### 3.3 WorkspaceManager 实现

```python
"""腾讯云 ECI Workspace 生命周期管理器。"""

import asyncio
import logging
from typing import Optional

from agentscope.app._manager._workspace_manager import WorkspaceManagerBase
from agentscope_tencent_cloud._tencent_cloud_workspace import TencentCloudWorkspace


logger = logging.getLogger(__name__)


class TencentCloudWorkspaceManager(WorkspaceManagerBase):
    """基于腾讯云 ECI 的 Workspace 生命周期管理器。

    支持 TTL 自动过期和后台清理。
    """

    def __init__(
        self,
        *,
        secret_id: str,
        secret_key: str,
        region: str = "ap-guangzhou",
        container_image: str = "python:3.11-slim",
        ttl: float = 3600.0,
        sweep_interval: float = 300.0,
    ):
        self.secret_id = secret_id
        self.secret_key = secret_key
        self.region = region
        self.container_image = container_image
        self.ttl = ttl
        self.sweep_interval = sweep_interval

        self._cache: dict[str, tuple[float, TencentCloudWorkspace]] = {}
        self._lock = asyncio.Lock()
        self._sweep_task: Optional[asyncio.Task] = None

    async def __aenter__(self):
        self._sweep_task = asyncio.create_task(self._sweep_loop())
        return self

    async def __aexit__(self, *exc):
        if self._sweep_task:
            self._sweep_task.cancel()
            self._sweep_task = None
        await self.close_all()

    async def get_workspace(
        self, user_id, agent_id, session_id, workspace_id
    ) -> TencentCloudWorkspace:
        async with self._lock:
            if workspace_id in self._cache:
                ts, ws = self._cache[workspace_id]
                if not ws.is_alive:
                    await ws.initialize()
                self._cache[workspace_id] = (asyncio.get_event_loop().time(), ws)
                return ws
            return await self.create_workspace(user_id, agent_id, session_id)

    async def create_workspace(
        self, user_id, agent_id, session_id
    ) -> TencentCloudWorkspace:
        ws = TencentCloudWorkspace(
            secret_id=self.secret_id,
            secret_key=self.secret_key,
            region=self.region,
            container_image=self.container_image,
        )
        async with ws:
            pass  # initialize via __aenter__
        async with self._lock:
            self._cache[ws.workspace_id] = (asyncio.get_event_loop().time(), ws)
        return ws

    async def close(self, workspace_id: str) -> None:
        async with self._lock:
            entry = self._cache.pop(workspace_id, None)
            if entry:
                _, ws = entry
                await ws.close()

    async def close_all(self) -> None:
        async with self._lock:
            for _, (_, ws) in self._cache.items():
                try:
                    await ws.close()
                except Exception as e:
                    logger.warning(f"关闭 workspace 失败: {e}")
            self._cache.clear()

    async def _sweep_loop(self) -> None:
        """后台清理过期 workspace。"""
        while True:
            await asyncio.sleep(self.sweep_interval)
            now = asyncio.get_event_loop().time()
            async with self._lock:
                expired = [
                    wid for wid, (ts, _) in self._cache.items()
                    if now - ts > self.ttl
                ]
            for wid in expired:
                await self.close(wid)
```

### 3.4 使用方式

```python
import os
import asyncio
from agentscope.agent import Agent
from agentscope.tool import Toolkit
from agentscope.credential import DashScopeCredential
from agentscope.model import DashScopeChatModel
from agentscope.message import UserMsg
from agentscope_tencent_cloud import TencentCloudWorkspace


async def main():
    # 创建腾讯云 Workspace
    workspace = TencentCloudWorkspace(
        secret_id=os.environ["TENCENTCLOUD_SECRET_ID"],
        secret_key=os.environ["TENCENTCLOUD_SECRET_KEY"],
        region="ap-guangzhou",
        container_image="python:3.11-slim",
        cpu="2",
        memory="4Gi",
        extra_pip=["numpy", "pandas"],
        skill_paths=["./skills"],
    )

    # 使用 workspace
    async with workspace:
        toolkit = Toolkit(
            mcps=await workspace.list_mcps(),
            skills_or_loaders=[],  # skills 已在 workspace 中
        )

        agent = Agent(
            name="cloud_agent",
            system_prompt="你是一个运行在腾讯云沙箱中的开发助手。",
            model=DashScopeChatModel(
                credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
                model="qwen-plus",
            ),
            toolkit=toolkit,
            offloader=workspace,
        )

        result = await agent.reply(
            UserMsg(name="user", content="在沙箱中创建一个 hello.py 并运行它")
        )
        print(result.get_text_content())


asyncio.run(main())
```

### 3.5 与 Agent Service 集成

```python
import uvicorn
from agentscope.app import create_app
from agentscope_tencent_cloud import TencentCloudWorkspaceManager

workspace_manager = TencentCloudWorkspaceManager(
    secret_id=os.environ["TENCENTCLOUD_SECRET_ID"],
    secret_key=os.environ["TENCENTCLOUD_SECRET_KEY"],
    region="ap-guangzhou",
    ttl=3600.0,
    sweep_interval=300.0,
)

app = create_app(
    storage=storage,
    workspace_manager=workspace_manager,
)

uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 四、进阶方案：基于 TKE Agent 沙箱

如果需要更高性能和更完整的沙箱能力，可基于 TKE Agent 沙箱实现：

### 区别

| 维度 | ECI 方案 | TKE 方案 |
|------|---------|---------|
| 运维成本 | 零 | 需管理 K8s 集群 |
| 启动速度 | 秒级 | 毫秒级 |
| 沙箱类型 | 通用容器 | 专用沙箱（浏览器/代码/自定义） |
| 扩展性 | 中 | 高 |
| 成本 | 低 | 中 |

### TKE 方案实现要点

1. 创建 TKE 集群并启用超级节点（Serverless）
2. 通过 k8s Python SDK（`kubernetes`）管理 Pod 生命周期
3. 沙箱 Pod 使用 TKE Agent 沙箱镜像
4. Gateway 部署为 Pod 内 sidecar 或独立进程

```python
from kubernetes import client, config

# 加载 kubeconfig
config.load_kube_config()

# 创建沙箱 Pod
v1 = client.CoreV1Api()
pod = client.V1Pod(
    api_version="v1",
    kind="Pod",
    metadata=client.V1ObjectMeta(
        name=f"agent-sandbox-{workspace_id[:12]}",
        labels={"app": "agent-sandbox", "workspace-id": workspace_id},
    ),
    spec=client.V1PodSpec(
        containers=[
            client.V1Container(
                name="agent-sandbox",
                image="python:3.11-slim",
                command=["sleep", "infinity"],
                ports=[client.V1ContainerPort(container_port=5600)],
            )
        ],
        # 使用超级节点（Serverless）
        node_selector={"node.kubernetes.io/instance-type": "super-node"},
    ),
)

v1.create_namespaced_pod(namespace="agent-sandboxes", body=pod)
```

## 五、注意事项

### 1. ECI 公网访问

ECI 容器默认无公网 IP，需要：
- 分配 EIP（弹性公网 IP）用于 Gateway 访问
- 或使用内网 VPC + NAT 网关
- 或使用 API 网关作为代理

### 2. 安全考虑

- Gateway Token 应定期轮换
- ECI 安全组应限制入站来源
- 敏感凭证（腾讯云 SecretKey）应使用密钥管理服务
- 生产环境应启用 VPC 网络隔离

### 3. 性能优化

- 使用镜像缓存加速 ECI 启动
- 预构建包含 Gateway 依赖的自定义镜像
- 长驻场景考虑 TKE 超级节点

### 4. 数据持久化

- ECI 容器删除后数据丢失
- 需要持久化的数据应使用 CBS（云硬盘）挂载
- 或使用 COS（对象存储）备份

## 六、参考链接

| 资源 | 链接 |
|------|------|
| AgentScope GitHub | https://github.com/agentscope-ai/agentscope |
| AgentScope Workspace 源码 | https://github.com/agentscope-ai/agentscope/tree/main/src/agentscope/workspace |
| 腾讯云 ECI 文档 | https://cloud.tencent.com/product/eci |
| 腾讯云 TKE 文档 | https://cloud.tencent.com/document/product/457 |
| 腾讯云 Python SDK | https://github.com/TencentCloud/tencentcloud-sdk-python |
| E2B 沙箱（参考实现） | https://e2b.dev |
