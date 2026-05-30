# 15. 腾讯云 Agent 沙箱调研

## 概述

腾讯云目前没有独立的 "Agent 沙箱" 产品，但提供多个可组合的服务来构建 Agent 安全执行环境。本调研梳理了腾讯云与 Agent 沙箱相关的所有产品和服务，以及它们的适用场景。

## 一、TKE Agent 沙箱服务（推荐方案）

### 产品定位

腾讯云容器服务 TKE 在 2025 年明确推出了 **Agent 沙箱服务**，这是腾讯云最接近独立 "Agent 沙箱" 的产品。

### 核心特性

| 特性 | 说明 |
|------|------|
| **安全隔离** | 独立受控环境，Agent 在沙箱中执行不影响宿主 |
| **极致启动** | 毫秒级实例启动，适合 Agent 交互场景 |
| **丰富种类** | 浏览器沙箱、代码沙箱、可扩展自定义沙箱 |
| **多种接入** | 兼容主流开源社区沙箱接口和协议 |

### 沙箱类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **浏览器沙箱** | 隔离的浏览器环境，Agent 可浏览网页、操作页面 | Web Agent、信息采集 |
| **代码沙箱** | 隔离的代码执行环境，支持多语言运行 | 代码生成与执行、数据分析 |
| **自定义沙箱** | 可扩展的沙箱框架，支持自定义工具和 MCP | 通用 Agent 场景 |

### 技术架构

TKE Agent 沙箱基于 Kubernetes 原生架构，通过超级节点（Serverless 模式）实现：

```
Agent 请求 → TKE 超级节点 → Pod 沙箱实例
                                ├── 代码执行环境
                                ├── 浏览器实例（可选）
                                └── MCP Gateway（可选）
```

### 节点模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **超级节点** | Serverless，免运维，按需弹性 | Agent 沙箱（推荐） |
| **原生节点** | 完整 OS，深度定制 | 需要内核级操作 |
| **注册节点** | 纳管自有资源 | 混合云场景 |

### API 接口

TKE 完全兼容原生 Kubernetes API：

```bash
# 创建沙箱 Pod
kubectl apply -f agent-sandbox-pod.yaml

# 或通过 k8s API
POST /api/v1/namespaces/{namespace}/pods
```

TKE 还提供腾讯云 API（`tencentcloudapi.com`）用于集群管理。

### 定价

- **超级节点（Serverless）**：按量计费，按 Pod 运行时间和资源规格计费
- **原生节点**：包年包月或按量计费（CVM 价格）
- **沙箱服务**：具体定价需咨询腾讯云商务

### Python SDK

```bash
pip install tencentcloud-sdk-python-tke
```

```python
from tencentcloud.common import credential
from tencentcloud.tke.v20180525 import tke_client, models

cred = credential.Credential("SecretId", "SecretKey")
client = tke_client.TkeClient(cred, "ap-guangzhou")

# 创建集群
req = models.CreateClusterRequest()
# ... 配置参数
resp = client.CreateCluster(req)
```

## 二、ECI 弹性容器实例

### 产品定位

ECI 是腾讯云的 Serverless 容器运行服务，无需购买和管理服务器即可运行容器。适合作为轻量级 Agent 代码执行沙箱。

### 核心特性

| 特性 | 说明 |
|------|------|
| **免运维** | 无需管理服务器，只需指定容器镜像 |
| **秒级启动** | 容器实例秒级启动 |
| **按量计费** | 按容器运行时长和资源规格计费 |
| **安全隔离** | 每个容器实例独立隔离 |
| **弹性伸缩** | 自动扩缩容 |

### 适用场景

- Agent 代码执行沙箱
- 临时性数据处理
- 离线任务执行
- CI/CD 流水线

### API 接口

ECI 提供完整的 API 接口，核心接口包括：

| 接口 | 说明 |
|------|------|
| `CreateContainerGroup` | 创建容器组 |
| `DeleteContainerGroup` | 删除容器组 |
| `DescribeContainerGroups` | 查询容器组列表 |
| `RestartContainerGroup` | 重启容器组 |
| `ExecuteCommand` | 在容器内执行命令 |
| `CreateImageCache` | 创建镜像缓存 |

### Python SDK

```bash
pip install tencentcloud-sdk-python-eci
```

```python
from tencentcloud.common import credential
from tencentcloud.eci.v20180508 import eci_client, models

cred = credential.Credential("SecretId", "SecretKey")
client = eci_client.EciClient(cred, "ap-guangzhou")

# 创建容器组
req = models.CreateContainerGroupRequest()
req.FromParams(
    ContainerGroupName="agent-sandbox",
    Containers=[{
        "Name": "agent-runner",
        "Image": "python:3.11-slim",
        "Cpu": "2",
        "Memory": "4Gi",
    }],
)
resp = client.CreateContainerGroup(req)
```

### 定价

- 按量计费：CPU 0.0033 元/核/分钟，内存 0.0014 元/GiB/分钟
- 镜像缓存：0.0004 元/GiB/小时
- 价格因地域略有差异

### ECI vs E2B 对比

| 维度 | ECI | E2B |
|------|-----|-----|
| 部署方式 | 腾讯云托管 | E2B 云端 |
| 网络延迟 | 国内低延迟 | 海外节点为主 |
| 数据合规 | 数据存国内 | 数据存海外 |
| SDK 成熟度 | 官方 SDK 完善 | 专用沙箱 SDK |
| 定价 | 按资源+时长 | 按 vCPU/小时 |
| 适用场景 | 国内企业 | 全球化项目 |

## 三、SCF 云函数

### 产品定位

SCF 是腾讯云的无服务器函数执行服务，适合短时、无状态的 Agent 工具执行。

### 核心特性

| 特性 | 说明 |
|------|------|
| **零运维** | 完全无需管理基础设施 |
| **事件触发** | 支持 API 网关、定时、COS 等多种触发器 |
| **毫秒级启动** | 函数冷启动毫秒级 |
| **按调用计费** | 只为实际执行时间付费 |

### 限制

| 限制项 | 值 |
|--------|-----|
| 执行超时 | 最大 900 秒（15 分钟） |
| 内存 | 最大 3 GB |
| 临时磁盘 | 最大 512 MB |
| 部署包 | 最大 250 MB（含依赖） |

### 适用场景

- Agent 工具调用（API 调用、数据处理）
- 短时代码执行
- Webhook 处理

### Python SDK

```bash
pip install tencentcloud-sdk-python-scf
```

```python
from tencentcloud.common import credential
from tencentcloud.scf.v20180416 import scf_client, models

cred = credential.Credential("SecretId", "SecretKey")
client = scf_client.ScfClient(cred, "ap-guangzhou")

# 创建函数
req = models.CreateFunctionRequest()
# ...
resp = client.CreateFunction(req)
```

### 定价

- 免费额度：每月 100 万次调用，40 万 GBs 资源使用
- 超出后：0.0133 元/万次调用，0.00001667 元/GBs

## 四、ADP 智能体开发平台

### 产品定位

腾讯云 ADP（Agent Development Platform）是原生的 Agent 开发平台，支持 LLM+RAG、Workflow、Multi-Agent 三种模式。

### 核心能力

| 能力 | 说明 |
|------|------|
| **多框架支持** | LLM+RAG、Workflow、Multi-Agent |
| **MCP 协议** | 基于 MCP 生态，提供企业级扩展能力 |
| **插件生态** | 插件广场，可扩展 Agent 能力 |
| **多模型管理** | 支持主流大模型及自建模型 |
| **工作流** | 可视化工作流编排 |
| **多端部署** | 官网、APP、小程序、企微等 |

### API 接口

ADP 提供以下对话端接口：

| 接口类型 | 说明 |
|----------|------|
| HTTP SSE | 服务端推送流式响应 |
| WebSocket | 实时双向通信 |
| 图片/文件对话 | 实时文档解析 |
| 离线文档上传 | 文档离线处理 |

### 开源集成方案

ADP-Chat-Client 是官方开源的对话端集成方案：

- GitHub: https://github.com/TencentCloudADP/adp-chat-client
- 提供 Docker 一键部署
- 支持 GitHub OAuth / Microsoft Entra ID / 自定义 OAuth
- 支持 URL 跳转登录（最简对接）

### 聊天 API

```python
# POST Chat Message (SSE 流式)
import requests

response = requests.post(
    "https://your-domain.com/api/chat/message",
    json={
        "Query": "用户消息",
        "ConversationId": "会话ID（可选）",
        "ApplicationId": "应用ID",
        "SearchNetwork": True,
        "CustomVariables": {}
    },
    headers={"Authorization": "Bearer <token>"},
    stream=True  # SSE 流式
)

for line in response.iter_lines():
    if line:
        print(line.decode())
```

### 局限性

- **无独立沙箱**：ADP 本身不提供代码执行沙箱
- **MCP 支持有限**：文档中 MCP 详细说明较少
- **定制性**：更适合使用 ADP 自身框架，不太适合嵌入 AgentScope

## 五、方案对比与选型建议

### 综合对比

| 维度 | TKE Agent 沙箱 | ECI | SCF | ADP |
|------|----------------|-----|-----|-----|
| **沙箱完整性** | 高（专用沙箱） | 中（容器隔离） | 低（函数级） | 无 |
| **启动速度** | 毫秒级 | 秒级 | 毫秒级 | N/A |
| **执行时长** | 无限制 | 无限制 | 最大 15 分钟 | N/A |
| **定制能力** | 高 | 中 | 低 | 中 |
| **国内部署** | 是 | 是 | 是 | 是 |
| **数据合规** | 国内存储 | 国内存储 | 国内存储 | 国内存储 |
| **成本** | 中 | 低 | 极低 | 中 |
| **与 AgentScope 集成** | 推荐方案 | 可行 | 工具级 | 不推荐 |

### 选型建议

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| **Agent 代码执行沙箱** | TKE Agent 沙箱 | 专用沙箱，功能最完整 |
| **轻量代码执行** | ECI 弹性容器 | 成本低，免运维 |
| **Agent 工具调用** | SCF 云函数 | 极低成本，按调用计费 |
| **国产化 Agent 平台** | ADP | 原生 Agent 框架 |
| **AgentScope 全功能集成** | TKE Agent 沙箱 | 兼容开源接口和协议 |

## 六、腾讯云 Python SDK 统一安装

```bash
# 核心依赖
pip install tencentcloud-sdk-python-common

# 按需安装
pip install tencentcloud-sdk-python-tke    # TKE
pip install tencentcloud-sdk-python-eci    # ECI
pip install tencentcloud-sdk-python-scf    # SCF

# 或安装全部
pip install tencentcloud-sdk-python
```

### 统一认证

```python
from tencentcloud.common import credential

# 所有腾讯云 SDK 共用同一凭证
cred = credential.Credential(
    secretId="your-secret-id",
    secretKey="your-secret-key",
)
```

### 环境变量配置

```bash
export TENCENTCLOUD_SECRET_ID="your-secret-id"
export TENCENTCLOUD_SECRET_KEY="your-secret-key"
export TENCENTCLOUD_REGION="ap-guangzhou"
```

## 七、参考链接

| 产品 | 文档链接 |
|------|----------|
| TKE 容器服务 | https://cloud.tencent.com/document/product/457 |
| ECI 弹性容器实例 | https://cloud.tencent.com/product/eci |
| SCF 云函数 | https://cloud.tencent.com/document/product/583 |
| ADP 智能体开发平台 | https://cloud.tencent.com/document/product/1759 |
| ADP-Chat-Client | https://github.com/TencentCloudADP/adp-chat-client |
| 腾讯云 SDK | https://github.com/TencentCloud/tencentcloud-sdk-python |
