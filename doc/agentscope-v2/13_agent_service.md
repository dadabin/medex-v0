# 13. Agent Service 生产部署

## 概述

Agent Service 是基于 FastAPI 的托管层，将 Agent 转化为多租户、多会话的 HTTP 服务。

## 快速启动

```python
import uvicorn
from agentscope.app import create_app, RedisStorage, LocalWorkspaceManager

storage = RedisStorage(host="localhost", port=6379)
workspace_manager = LocalWorkspaceManager(basedir="/data/workspaces", ttl=3600.0)

app = create_app(
    storage=storage,
    workspace_manager=workspace_manager,
)
uvicorn.run(app, host="0.0.0.0", port=8000)
```

## create_app 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `storage` | `StorageBase`（必填） | 持久化后端（内置 Redis） |
| `workspace_manager` | `WorkspaceManagerBase \| None` | Workspace 生命周期管理 |
| `extra_credentials` | `list[Type[CredentialBase]] \| None` | 额外的凭证类型 |
| `extra_middlewares` | `list[Middleware] \| None` | ASGI 中间件（CORS、认证、协议适配器） |
| `extra_agent_middlewares` | `AgentMiddlewareFactory \| None` | 每 Agent 的中间件工厂 |
| `extra_agent_tools` | `AgentToolFactory \| None` | 每 Agent 的工具工厂 |

## REST API

### 聊天

| 端点 | 方法 | 说明 |
|------|------|------|
| `/chat` | POST | SSE 流式响应，支持回放和多订阅者扇出 |

### 会话管理

| 端点 | 方法 | 说明 |
|------|------|------|
| `/sessions` | GET | 列出所有会话 |
| `/sessions` | POST | 创建会话 |
| `/sessions/{id}` | GET | 获取会话详情 |
| `/sessions/{id}` | PUT | 更新会话 |
| `/sessions/{id}` | DELETE | 删除会话 |
| `/sessions/{id}/messages` | GET | 分页获取消息记录 |

### Agent 管理

| 端点 | 方法 | 说明 |
|------|------|------|
| `/agent` | GET | 列出所有 Agent |
| `/agent` | POST | 创建 Agent |
| `/agent/{id}` | GET | 获取 Agent 详情 |
| `/agent/{id}` | PUT | 更新 Agent |
| `/agent/{id}` | DELETE | 删除 Agent |

### 凭证管理

| 端点 | 方法 | 说明 |
|------|------|------|
| `/credential` | GET | 列出所有凭证 |
| `/credential` | POST | 创建凭证 |
| `/credential/{id}` | DELETE | 删除凭证 |
| `/credential/schemas` | GET | 获取凭证 JSON Schema |

### 模型管理

| 端点 | 方法 | 说明 |
|------|------|------|
| `/model` | GET | 获取 ModelCard 列表（支持 `?provider=<name>` 过滤） |

### 调度管理

| 端点 | 方法 | 说明 |
|------|------|------|
| `/schedule` | GET | 列出所有调度 |
| `/schedule` | POST | 创建调度 |
| `/schedule/{id}` | GET | 获取调度详情 |
| `/schedule/{id}` | PUT | 更新调度 |
| `/schedule/{id}` | DELETE | 删除调度 |

### 后台任务

| 端点 | 方法 | 说明 |
|------|------|------|
| `/background-tasks` | GET | 获取卸载的工具执行状态 |

### Workspace MCP

| 端点 | 方法 | 说明 |
|------|------|------|
| `/workspace/mcp` | GET | 列出 MCP 客户端 |
| `/workspace/mcp` | POST | 添加 MCP 客户端 |
| `/workspace/mcp/{id}` | DELETE | 删除 MCP 客户端 |

### Workspace 技能

| 端点 | 方法 | 说明 |
|------|------|------|
| `/workspace/skill` | GET | 列出技能 |
| `/workspace/skill` | POST | 添加技能 |
| `/workspace/skill/{id}` | DELETE | 删除技能 |

## 存储后端

### Redis（内置）

```python
from agentscope.app import RedisStorage

storage = RedisStorage(host="localhost", port=6379)
```

### 自定义存储后端

```python
from agentscope.app.storage import StorageBase

class PostgresStorage(StorageBase):
    async def __aenter__(self):
        # 打开连接池
        ...

    async def __aexit__(self, *args):
        # 关闭连接池
        ...

    # 实现 CRUD：AgentRecord, SessionRecord, CredentialRecord, ScheduleRecord, Msg
```

## 多租户架构

所有资源按 `user_id` 隔离。默认实现从 `X-User-ID` 头读取（占位符，生产环境必须替换）：

```python
from fastapi import Header, HTTPException, status

async def get_current_user_id(authorization: str = Header(...)) -> str:
    payload = decode_jwt(authorization.removeprefix("Bearer "))
    return payload["sub"]

# 覆盖默认依赖
app.dependency_overrides[default_dependency] = get_current_user_id
```

## 调度系统

支持 Cron 定时执行 Agent：

| 模式 | 说明 |
|------|------|
| **Stateless** | 每次执行使用新会话 |
| **Stateful** | 复用同一会话，累积上下文 |

调度通过 APScheduler 持久化，服务重启后自动恢复。

## Web UI

### 启动后端

```bash
cd agentscope/examples/agent_service
python main.py
```

### 启动前端

```bash
cd agentscope/examples/web_ui
pnpm install
pnpm dev
```

前端提供聊天式界面，内置权限系统的人机协作确认功能。

## TypeScript SDK

### 安装

```bash
pnpm install @agentscope-ai/agentscope
```

### 使用

```typescript
import { AssistantMsg, ReplyStartEvent } from "@agentscope-ai/agentscope/message";

let msg: AssistantMsg | null = null;
for await (const event of stream) {
    if (event.type === "REPLY_START") {
        msg = new AssistantMsg({ name: event.name, content: [], id: event.reply_id });
    } else {
        msg?.appendEvent(event);
    }
}
```

## 协议适配

Agent Service 支持通过中间件转换为外部协议（AG-UI, A2A）：

```python
from agentscope.app import create_app, AGUIProtocolMiddleware
from fastapi import Middleware

app = create_app(
    storage=storage,
    extra_middlewares=[Middleware(AGUIProtocolMiddleware)],
)
```

## 完整生产部署示例

```python
import os
import uvicorn
from fastapi import Header, HTTPException, Middleware
from fastapi.middleware.cors import CORSMiddleware
from agentscope.app import create_app, RedisStorage, LocalWorkspaceManager
from agentscope.middleware import TracingMiddleware

# 存储后端
storage = RedisStorage(host="redis", port=6379, password=os.environ.get("REDIS_PASSWORD"))

# Workspace 管理
workspace_manager = LocalWorkspaceManager(
    basedir="/data/workspaces",
    ttl=3600.0,  # 1 小时过期
)

# 认证
async def get_current_user_id(authorization: str = Header(...)) -> str:
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid token")
    token = authorization.removeprefix("Bearer ")
    payload = verify_jwt(token)
    return payload["sub"]

# 创建应用
app = create_app(
    storage=storage,
    workspace_manager=workspace_manager,
    extra_middlewares=[
        Middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"]),
    ],
    extra_agent_middlewares=lambda: [TracingMiddleware()],
)

# 覆盖认证依赖
# app.dependency_overrides[default_dependency] = get_current_user_id

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000, workers=4)
```
