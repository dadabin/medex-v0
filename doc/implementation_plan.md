# MeDEX 实现方案

## 1. 项目概述

MeDEX 是一个面向医疗场景的智能体工具，类似 OpenAI Codex 的交互模式。核心概念：**一个项目 = 一位患者的全周期信息管理**。用户创建患者项目后上传资料，Agent 自动规划分析任务、动态调整子任务、查询指南共识、整理文档，最终输出结论性报告。过程中 Agent 可主动请求补充信息或提供选择项，用户交互确认后任务继续推进。

### 核心交互流程

```
用户创建项目 → 上传患者资料 → Agent 分析资料
     → 自动生成分析计划(subtask列表)
     → 逐个执行 subtask
        ├─ 需要指南/共识 → 联网搜索 → 整理为MD文档
        ├─ 发现信息不足 → 对话框中提示用户 → 等待补充 → 继续
        ├─ 发现 subtask 有误/不全 → 自动修改/新增 subtask → 重新执行
        └─ subtask 完成 → 更新状态 → 执行下一个
     → 所有 subtask 完成 → 输出结论性报告
     → 生成文件在对话中以链接形式出现 → 右侧文件面板查看
```

---

## 2. 页面布局

### 2.1 三栏布局总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  顶栏  h=56px                                                               │
│  [≡]  MeDEX · 患者项目                   [当前任务进度]  [⚙]  [avatar]        │
├───────────┬──────────────────────────────────────┬──────────────────────────┤
│           │                                      │                          │
│  左侧栏    │  中间 · 对话区                        │  右侧 · 文件面板          │
│  260px    │  flex-1                               │  320px                   │
│  surface  │  bg=base                              │  surface                 │
│           │                                      │                          │
│ ┌───────┐ │  ┌──────────────────────────────┐   │ ┌──────────────────────┐ │
│ │ + 新建 │ │  │  📋 任务计划                   │   │ │ 📁 项目文件           │ │
│ │  项目  │ │  │  ├─ 1. 分析主诉与病史  ✅      │   │ │                      │ │
│ └───────┘ │  │  ├─ 2. 评估检查结果   ⏳      │   │ │ ├─ 📄 门诊记录.md     │ │
│           │  │  ├─ 3. 查询高血压指南  ○       │   │ │ ├─ 🖼️ CT扫描.png     │ │
│ 🔍 搜索   │  │  └─ 4. 生成诊断报告  ○        │   │ │ ├─ 📄 血常规.pdf      │ │
│           │  │                                │   │ │ ├─ 📄 分析报告.md  ✨│ │
│ ─────── │  │  ───────────────────────────── │   │ │ └─ 📄 用药方案.md  ✨│ │
│ 最近项目  │  │                                │   │ │                      │ │
│           │  │  🤖 MeDEX Agent                │   │ │ ────────────────── │ │
│ ● 张三    │  │  │                             │   │ │ 📄 文件预览          │ │
│   高血压   │  │  我已分析患者资料，生成以下     │   │ │                      │ │
│           │  │  分析计划:                      │   │ │ ┌────────────────┐ │ │
│ ○ 李四    │  │                                │   │ │ │                │ │ │
│   糖尿病   │  │  [📋 查看/编辑任务计划]         │   │ │ │  (文件内容      │ │ │
│           │  │                                │   │ │ │   预览区)       │ │ │
│ ○ 王五    │  │  正在执行第2项: 评估检查结果...  │   │ │ │                │ │ │
│   肺结节   │  │                                │   │ │ └────────────────┘ │ │
│           │  │  🔧 Read → 血常规.pdf ✅        │   │ │                      │ │
│ ─────── │  │  🔧 WebSearch → 高血压用药指南   │   │ │ [下载] [在新标签打开] │ │
│ 归档项目  │  │    ⏳ 搜索中...                  │   │ └──────────────────────┘ │
│           │  │                                │   │                          │
│ ○ 赵六    │  │  ⚠️ 需要补充信息                │   │                          │
│   术后     │  │  请提供以下检查结果:            │   │                          │
│           │  │                                │   │                          │
│           │  │  ┌──────────────────────────┐ │   │                          │
│           │  │  │ ○ 有 · 心电图             │ │   │                          │
│           │  │  │ ○ 有 · 超声心动图          │ │   │                          │
│           │  │  │ ● 无以上检查              │ │   │                          │
│           │  │  │                          │ │   │                          │
│           │  │  │     [确认]                │ │   │                          │
│           │  │  └──────────────────────────┘ │   │                          │
│           │  │                                │   │                          │
│           │  ├──────────────────────────────┤   │                          │
│           │  │  [📎上传] [输入医学问题...]  [▶]│   │                          │
│           │  └──────────────────────────────┘   │                          │
│           │                                      │                          │
└───────────┴──────────────────────────────────────┴──────────────────────────┘
```

### 2.2 左侧栏 · 项目管理

```
┌──────────────────┐
│  [+ 新建项目]      │  ← 点击弹出创建对话框
│                   │
│  🔍 搜索项目...    │
│                   │
│  ─── 最近 ─────── │
│                   │
│  ● 张三           │  ← 活跃项目，绿点
│    高血压 · 2型糖尿病│
│    3个任务 · 5个文件 │
│    10分钟前        │
│                   │
│  ○ 李四           │  ← 非活跃
│    糖尿病          │
│    5个任务 · 8个文件 │
│    2小时前         │
│                   │
│  ─── 归档 ─────── │
│                   │
│  ○ 赵六           │
│    术后随访 · 已完成 │
│    昨天            │
└──────────────────┘
```

新建项目弹窗：
```
┌─────────────────────────────────┐
│  新建患者项目              [✕]   │
│                                 │
│  患者姓名 *  [____________]     │
│  项目描述    [____________]     │
│  标签       [+添加标签]        │
│             高血压 糖尿病       │
│                                 │
│  [取消]          [创建项目]     │
└─────────────────────────────────┘
```

### 2.3 中间 · 对话区

对话区由三部分组成：**任务计划面板**（顶部固定）+ **消息流**（中间滚动）+ **输入区**（底部固定）。

#### 任务计划面板

```
┌──────────────────────────────────────────────────────┐
│  📋 任务计划                            [编辑] [收起]  │
│                                                      │
│  1. ✅ 分析主诉与病史                0.3s  [查看结果]  │
│  2. ⏳ 评估检查结果                  ...    [查看]     │
│  3. 🔍 查询高血压用药指南(2024)      ○      [等待中]   │
│  4. 📝 生成诊断与用药建议报告         ○      [等待中]   │
│                                                      │
│  进度: ████████░░░░░░░░░░░░  1/4 完成                  │
└──────────────────────────────────────────────────────┘
```

状态图标：
- `○` 待执行（`--text-tertiary`）
- `⏳` 执行中（`--color-tool`，脉冲动画）
- `✅` 已完成（`--color-success`）
- `❌` 失败（`--color-error`）
- `🔄` 重试中（`--color-warning`）

#### 消息流中的特殊组件

**Agent 主动提问（选择框）：**
```
┌─ 💬 需要补充信息 ─────────────────────────────────┐
│                                                    │
│  为完成「评估心血管风险」，需要以下信息：            │
│                                                    │
│  该患者是否进行过以下检查？                         │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │  ○ 心电图                                    │  │
│  │  ○ 超声心动图                                │  │
│  │  ○ 冠状动脉CT                                │  │
│  │  ● 以上均无                                  │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  或者直接上传检查报告: [📎 上传文件]                 │
│                                                    │
│                                    [确认并继续]     │
└────────────────────────────────────────────────────┘
```

**Agent 主动提问（文本输入）：**
```
┌─ 💬 需要补充信息 ─────────────────────────────────┐
│                                                    │
│  当前血压控制情况如何？请提供近期的血压读数：        │
│                                                    │
│  收缩压: [____] mmHg                               │
│  舒张压: [____] mmHg                               │
│                                                    │
│                                    [确认并继续]     │
└────────────────────────────────────────────────────┘
```

**文件链接（生成文件出现在对话中）：**
```
分析报告已生成，详见：
  📄 [分析报告_20260530.md](file:analysis_report.md)  ✨新
  📄 [用药方案.md](file:treatment_plan.md)            ✨新
  📄 [指南摘要_高血压2024.md](file:guideline_summary.md) ✨新

点击链接可在右侧文件面板中查看。
```

**Subtask 变更通知：**
```
┌─ 🔄 任务调整 ──────────────────────────────────────┐
│                                                    │
│  在执行过程中发现以下调整：                         │
│                                                    │
│  + 新增: 3a. 评估肾功能 (检查结果显示肌酐偏高)       │
│  ~ 修改: 3. 查询高血压用药指南 → 查询高血压+肾功能   │
│                                                    │
│  更新后计划:                                       │
│  1. ✅ 分析主诉与病史                              │
│  2. ✅ 评估检查结果                                │
│  3a. ⏳ 评估肾功能              ← 新增             │
│  3. ○ 查询高血压+肾功能用药指南  ← 修改             │
│  4. ○ 生成诊断与用药建议报告                        │
└────────────────────────────────────────────────────┘
```

#### 输入区

```
┌──────────────────────────────────────────────────────────┐
│  [📎 上传]  支持: 图片/PDF/文本/Word/DICOM               │
│  ┌──────────────────────────────────────────────┐  [▶]  │
│  │  输入医学问题或补充信息...                     │       │
│  │  Shift+Enter 换行                             │       │
│  └──────────────────────────────────────────────┘       │
│  已上传: 📄 血常规.pdf (1.2MB) ✕   🖼️ CT.png (3.4MB) ✕  │
└──────────────────────────────────────────────────────────┘
```

### 2.4 右侧 · 文件面板

```
┌──────────────────────────┐
│  📁 项目文件      [上传]  │
│                          │
│  ─── 患者资料 ────────── │
│  📄 门诊记录.md     2KB  │
│  🖼️ CT扫描.png    3.4MB │
│  📄 血常规报告.pdf  1.2MB│
│  📄 既往病史.txt    0.5KB│
│                          │
│  ─── 生成文件 ────────── │
│  📄 分析报告.md  ✨ 5KB  │
│  📄 用药方案.md  ✨ 3KB  │
│  📄 指南摘要.md  ✨ 8KB  │
│                          │
│  ─── 文件预览 ────────── │
│                          │
│  📄 分析报告.md          │
│  ┌────────────────────┐ │
│  │ # 患者分析报告      │ │
│  │                    │ │
│  │ ## 基本信息        │ │
│  │ - 姓名: 张三       │ │
│  │ - 年龄: 58岁       │ │
│  │ - 性别: 男         │ │
│  │                    │ │
│  │ ## 诊断分析        │ │
│  │ ...                │ │
│  └────────────────────┘ │
│                          │
│  [下载] [在新标签打开]    │
└──────────────────────────┘
```

- 点击文件名 → 预览区显示内容
- 点击对话中的文件链接 → 右侧面板打开该文件
- ✨ 标识新生成的文件
- 支持预览: Markdown渲染、图片、PDF(iframe)、文本
- 文件列表分两组: 「患者资料」(用户上传) + 「生成文件」(Agent 产出)

---

## 3. 技术选型

| 层面 | 方案 | 说明 |
|------|------|------|
| 后端框架 | FastAPI | AgentScope Agent Service 原生集成 |
| Agent 框架 | AgentScope v2 | 统一 Agent 类 + ReAct 循环 |
| 模型 | DashScope (Qwen) | 国产模型，医疗中文能力强 |
| 前端 | 原生 HTML + CSS + JS | 首期不引入框架，快速验证 |
| 流式通信 | SSE | AgentScope 原生支持 |
| 文件存储 | 本地文件系统 | JSON 元数据 + 文件目录，后续迁移 MySQL |
| 搜索工具 | 自定义 WebSearch 工具 | 联网查询医学指南 |

---

## 4. 系统架构

### 4.1 整体架构

```
┌──────────────────────────────────────────────────────┐
│  浏览器                                               │
│                                                      │
│  ┌──────┐  ┌────────────────┐  ┌──────────────┐     │
│  │ 左侧栏 │  │  中间对话区      │  │ 右侧文件面板  │     │
│  │ 项目   │  │  SSE 流式接收    │  │ 文件列表      │     │
│  │ 列表   │  │  + 交互组件      │  │ 预览区        │     │
│  └──┬───┘  └───────┬────────┘  └──────┬───────┘     │
│     │              │                  │              │
│     └──────────────┼──────────────────┘              │
│                    │ HTTP / SSE                      │
└────────────────────┼─────────────────────────────────┘
                     │
┌────────────────────┼─────────────────────────────────┐
│  FastAPI 后端       │                                 │
│                    ▼                                  │
│  ┌──────────────────────────────────────────────┐   │
│  │  API Router                                   │   │
│  │  POST /api/projects        创建项目            │   │
│  │  GET  /api/projects        项目列表            │   │
│  │  POST /api/projects/{id}/chat  对话(SSE)      │   │
│  │  POST /api/projects/{id}/upload  上传文件      │   │
│  │  GET  /api/projects/{id}/files   文件列表      │   │
│  │  GET  /api/projects/{id}/files/{fid} 文件内容  │   │
│  │  POST /api/projects/{id}/interact  用户交互    │   │
│  │  GET  /api/projects/{id}/tasks    任务列表      │   │
│  └──────────────┬───────────────────────────────┘   │
│                 │                                     │
│  ┌──────────────▼───────────────────────────────┐   │
│  │  MeDEX Agent (AgentScope v2)                  │   │
│  │                                               │   │
│  │  System Prompt: 医学分析助手                    │   │
│  │  Model: Qwen (DashScope)                      │   │
│  │  Toolkit:                                      │   │
│  │    ├── FileManager     读写项目文件             │   │
│  │    ├── SubtaskManager  管理分析计划             │   │
│  │    ├── WebSearch       搜索医学指南             │   │
│  │    ├── UserInteraction 请求用户补充信息         │   │
│  │    └── MedicalDocGen   生成MD文档              │   │
│  │  Middleware:                                   │   │
│  │    └── TracingMiddleware                      │   │
│  └───────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  Local Storage                                │   │
│  │  data/                                        │   │
│  │    projects/                                  │   │
│  │      {project_id}/                            │   │
│  │        meta.json      项目元数据               │   │
│  │        messages.jsonl 对话记录                 │   │
│  │        tasks.json     任务列表                 │   │
│  │        files/         患者资料 + 生成文件       │   │
│  │          uploads/     用户上传                 │   │
│  │          generated/   Agent 产出               │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

### 4.2 Agent 工作流

```
用户消息 → Agent.reply()
  │
  ├── 思考: 分析需求，制定/调整计划
  │     └── 调用 SubtaskManager 生成/修改 subtask 列表
  │
  ├── 执行: 逐个执行 subtask
  │     ├── 读取患者资料 → FileManager.read()
  │     ├── 搜索指南共识 → WebSearch.search()
  │     ├── 生成文档 → MedicalDocGen.generate()
  │     │
  │     ├── [条件] 发现信息不足 → UserInteraction.ask()
  │     │     └── 暂停，等待用户回复 → 收到回复 → 继续
  │     │
  │     └── [条件] 发现 subtask 有误 → SubtaskManager.update()
  │           └── 新增/修改 subtask → 通知前端 → 重新执行
  │
  └── 总结: 输出结论性报告
        └── 生成最终文档 → 链接出现在对话中
```

---

## 5. 数据模型

### 5.1 项目 (Project)

存储路径: `data/projects/{project_id}/meta.json`

```json
{
  "id": "proj_abc123",
  "name": "张三",
  "description": "高血压 · 2型糖尿病",
  "tags": ["高血压", "2型糖尿病"],
  "status": "active",
  "created_at": "2026-05-30T14:30:00Z",
  "updated_at": "2026-05-30T15:00:00Z"
}
```

### 5.2 消息 (Message)

存储路径: `data/projects/{project_id}/messages.jsonl`

每行一条消息，兼容 AgentScope v2 的 Msg 格式：

```json
{"id":"msg_001","name":"user","role":"user","content":[{"type":"text","text":"请分析这位患者的情况"}],"created_at":"2026-05-30T14:30:00Z"}
{"id":"msg_002","name":"MeDEX","role":"assistant","content":[{"type":"text","text":"我已分析患者资料..."},{"type":"tool_call","name":"SubtaskManager","input":"..."},{"type":"tool_result","name":"SubtaskManager","output":"..."}],"created_at":"2026-05-30T14:30:05Z"}
```

### 5.3 任务 (Task)

存储路径: `data/projects/{project_id}/tasks.json`

```json
{
  "plan_id": "plan_001",
  "project_id": "proj_abc123",
  "title": "患者分析计划",
  "status": "in_progress",
  "subtasks": [
    {
      "id": "task_001",
      "index": 1,
      "title": "分析主诉与病史",
      "description": "整理患者主诉、现病史、既往史",
      "status": "completed",
      "result_summary": "患者58岁男性，高血压3年，2型糖尿病2年...",
      "started_at": "2026-05-30T14:30:05Z",
      "completed_at": "2026-05-30T14:30:08Z",
      "output_files": ["generated/病史分析.md"]
    },
    {
      "id": "task_002",
      "index": 2,
      "title": "评估检查结果",
      "description": "分析血常规、肝肾功能、影像学检查",
      "status": "in_progress",
      "result_summary": null,
      "started_at": "2026-05-30T14:30:08Z",
      "completed_at": null,
      "output_files": []
    }
  ],
  "created_at": "2026-05-30T14:30:05Z",
  "updated_at": "2026-05-30T14:30:08Z"
}
```

### 5.4 文件 (File)

文件元数据存储在 `meta.json` 的 `files` 数组中：

```json
{
  "files": [
    {
      "id": "file_001",
      "name": "血常规报告.pdf",
      "type": "upload",
      "mime_type": "application/pdf",
      "size": 1224000,
      "path": "uploads/血常规报告.pdf",
      "uploaded_at": "2026-05-30T14:31:00Z"
    },
    {
      "id": "file_002",
      "name": "分析报告.md",
      "type": "generated",
      "mime_type": "text/markdown",
      "size": 5120,
      "path": "generated/分析报告.md",
      "generated_at": "2026-05-30T14:35:00Z",
      "source_task": "task_001"
    }
  ]
}
```

---

## 6. Agent 工具设计

### 6.1 FileManager

管理项目文件的读写操作。

```python
class FileManager(ToolBase):
    name = "FileManager"
    description = "管理患者项目文件的读写操作。"

    async def list_files(self) -> str:
        """列出项目中的所有文件。"""

    async def read_file(self, file_name: str) -> str:
        """读取指定文件的内容。"""

    async def write_file(self, file_name: str, content: str) -> str:
        """写入/创建文件到项目的 generated 目录。"""

    async def append_file(self, file_name: str, content: str) -> str:
        """向已有文件追加内容。"""
```

### 6.2 SubtaskManager

管理分析计划和子任务，支持动态增删改。

```python
class SubtaskManager(ToolBase):
    name = "SubtaskManager"
    description = "管理医学分析计划，创建、修改、完成子任务。"

    async def create_plan(self, subtasks: list[dict]) -> str:
        """创建分析计划。

        Args:
            subtasks: 子任务列表，每项含 title 和 description。
        """

    async def update_subtask(self, task_id: str, status: str, result_summary: str = "") -> str:
        """更新子任务状态。

        Args:
            task_id: 任务ID。
            status: 新状态 (in_progress/completed/failed)。
            result_summary: 结果摘要。
        """

    async def add_subtask(self, title: str, description: str, after_task_id: str = "") -> str:
        """新增子任务（可在执行过程中动态添加）。

        Args:
            title: 任务标题。
            description: 任务描述。
            after_task_id: 插入到指定任务之后，为空则追加到末尾。
        """

    async def modify_subtask(self, task_id: str, title: str = "", description: str = "") -> str:
        """修改已有子任务。"""

    async def get_plan(self) -> str:
        """获取当前完整计划及各任务状态。"""
```

### 6.3 WebSearch

搜索医学指南和共识。

```python
class WebSearch(ToolBase):
    name = "WebSearch"
    description = "搜索网络上的医学指南、共识和文献资料。"

    async def search(self, query: str, search_type: str = "guideline") -> str:
        """搜索医学资料。

        Args:
            query: 搜索关键词。
            search_type: 搜索类型 (guideline/literature/general)。
        """
```

### 6.4 UserInteraction

Agent 主动请求用户补充信息或做出选择。

```python
class UserInteraction(ToolBase):
    name = "UserInteraction"
    description = "向用户请求补充信息或确认选择，当分析过程中发现信息不足时使用。"
    is_external_tool = True  # 需要等待用户响应

    async def ask_choice(self, question: str, options: list[str]) -> str:
        """向用户提出选择题。

        Args:
            question: 问题描述。
            options: 选项列表。
        """

    async def ask_input(self, question: str, fields: list[dict]) -> str:
        """向用户请求输入信息。

        Args:
            question: 问题描述。
            fields: 需要填写的字段列表，每项含 name、label、type。
        """

    async def ask_upload(self, question: str, accept: str = "") -> str:
        """请求用户上传文件。

        Args:
            question: 上传说明。
            accept: 接受的文件类型，如 ".pdf,.jpg,.png"。
        """

    async def confirm(self, message: str) -> str:
        """请求用户确认。"""
```

### 6.5 MedicalDocGen

生成结构化的医学文档。

```python
class MedicalDocGen(ToolBase):
    name = "MedicalDocGen"
    description = "生成结构化的医学分析文档，自动整理为 Markdown 格式。"

    async def generate(self, title: str, sections: list[dict]) -> str:
        """生成医学文档。

        Args:
            title: 文档标题。
            sections: 文档段落列表，每项含 heading 和 content。
        """

    async def generate_report(self, report_type: str, patient_data: str, analysis_result: str) -> str:
        """生成完整报告。

        Args:
            report_type: 报告类型 (diagnosis/treatment/followup)。
            patient_data: 患者数据摘要。
            analysis_result: 分析结果。
        """
```

---

## 7. Agent System Prompt

```
你是 MeDEX，一位专业的医学分析智能助手。你的职责是根据患者的资料，系统性地进行医学分析，并输出结构化的报告。

## 工作流程

1. **接收资料**: 用户上传患者资料后，先用 FileManager 读取所有文件，全面了解患者情况。

2. **制定计划**: 使用 SubtaskManager 创建分析计划。计划应覆盖：
   - 患者基本信息整理
   - 主诉与病史分析
   - 检查检验结果评估
   - 相关指南/共识查询
   - 诊断分析
   - 治疗方案建议
   - 结论性报告生成

3. **执行计划**: 逐个执行 subtask，每完成一个使用 SubtaskManager.update_subtask 更新状态。

4. **动态调整**: 执行过程中若发现：
   - 信息不足 → 使用 UserInteraction 请求用户补充
   - subtask 不准确 → 使用 SubtaskManager.add_subtask 或 modify_subtask 调整
   - 需要查询指南 → 使用 WebSearch 搜索

5. **生成文档**: 分析结果使用 MedicalDocGen 或 FileManager 写入 MD 文件。

6. **输出结论**: 所有任务完成后，生成结论性总结回复，引用生成的文件链接。

## 重要规则

- 所有医学建议必须基于循证医学，查询指南时注明来源和版本
- 对于不确定的内容，使用 UserInteraction 主动向用户确认
- 生成的文件名使用中文，格式为「{类型}_{日期}.md」
- 在回复中引用文件时使用 [文件名](file:文件名) 格式
- subtask 状态变更时，在对话中简要说明变更原因
- 绝不编造患者数据或检查结果
```

---

## 8. API 设计

### 8.1 项目管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/projects` | 创建项目 |
| `GET` | `/api/projects` | 获取项目列表 |
| `GET` | `/api/projects/{id}` | 获取项目详情 |
| `PUT` | `/api/projects/{id}` | 更新项目 |
| `DELETE` | `/api/projects/{id}` | 删除项目 |

### 8.2 对话

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/projects/{id}/chat` | 发送消息（SSE 流式响应） |
| `GET` | `/api/projects/{id}/messages` | 获取历史消息 |
| `POST` | `/api/projects/{id}/interact` | 用户交互响应（选择/输入/上传） |

### 8.3 文件

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/projects/{id}/upload` | 上传文件 |
| `GET` | `/api/projects/{id}/files` | 获取文件列表 |
| `GET` | `/api/projects/{id}/files/{file_id}` | 获取文件内容/下载 |

### 8.4 任务

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/projects/{id}/tasks` | 获取任务列表 |
| `GET` | `/api/projects/{id}/tasks/{task_id}` | 获取任务详情 |

### 8.5 SSE 事件类型

| 事件 | 数据 | 说明 |
|------|------|------|
| `message_start` | `{reply_id, name}` | Agent 回复开始 |
| `text_delta` | `{delta}` | 文本增量 |
| `thinking_delta` | `{delta}` | 思考增量 |
| `tool_call` | `{tool_name, tool_input}` | 工具调用 |
| `tool_result` | `{tool_name, output, state}` | 工具结果 |
| `task_update` | `{task_id, status, ...}` | 任务状态变更 |
| `task_added` | `{task}` | 新增任务 |
| `task_modified` | `{task_id, changes}` | 修改任务 |
| `ask_user` | `{type, question, options/fields}` | 请求用户交互 |
| `file_created` | `{file_id, name, path}` | 新文件生成 |
| `message_end` | `{usage}` | 回复结束 |

---

## 9. 项目目录结构

```
medex/
├── backend/
│   ├── main.py                     # FastAPI 入口
│   ├── config.py                   # 配置（API Key、存储路径等）
│   ├── router/
│   │   ├── __init__.py
│   │   ├── projects.py             # 项目管理 API
│   │   ├── chat.py                 # 对话 API (SSE)
│   │   ├── files.py                # 文件 API
│   │   └── tasks.py                # 任务 API
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── medex_agent.py          # MeDEX Agent 创建与配置
│   │   └── system_prompt.py        # 系统提示词
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── file_manager.py         # FileManager 工具
│   │   ├── subtask_manager.py      # SubtaskManager 工具
│   │   ├── web_search.py           # WebSearch 工具
│   │   ├── user_interaction.py     # UserInteraction 工具
│   │   └── medical_doc_gen.py      # MedicalDocGen 工具
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── base.py                 # 存储接口（后续 MySQL 实现同一接口）
│   │   └── local_storage.py        # 本地文件存储实现
│   ├── models/
│   │   ├── __init__.py
│   │   ├── project.py              # Project 数据模型
│   │   ├── message.py              # Message 数据模型
│   │   ├── task.py                 # Task 数据模型
│   │   └── file.py                 # File 数据模型
│   └── requirements.txt
│
├── frontend/
│   ├── index.html                  # 主页面
│   ├── css/
│   │   ├── variables.css           # CSS 变量（色彩/字体/间距）
│   │   ├── base.css                # 基础重置与全局样式
│   │   ├── layout.css              # 三栏布局
│   │   ├── sidebar.css             # 左侧栏样式
│   │   ├── chat.css                # 对话区样式
│   │   ├── file-panel.css          # 右侧文件面板样式
│   │   └── components.css          # 组件样式（任务面板/选择框/文件链接等）
│   └── js/
│       ├── app.js                  # 应用入口与状态管理
│       ├── api.js                  # API 调用封装
│       ├── sse.js                  # SSE 流式接收与事件分发
│       ├── sidebar.js              # 左侧栏逻辑
│       ├── chat.js                 # 对话区渲染
│       ├── file-panel.js           # 文件面板逻辑
│       └── components.js           # 组件渲染（任务面板/选择框等）
│
├── data/                           # 运行时数据（gitignore）
│   └── projects/
│       └── {project_id}/
│           ├── meta.json
│           ├── messages.jsonl
│           ├── tasks.json
│           └── files/
│               ├── uploads/
│               └── generated/
│
├── doc/                            # 文档
│   ├── agentscope-v2/
│   ├── ui/
│   └── implementation_plan.md
│
├── CLAUDE.md
└── .gitignore
```

---

## 10. 开发计划

### Phase 1: 基础骨架（3天）

| 任务 | 说明 |
|------|------|
| 项目初始化 | 创建目录结构、requirements.txt、.gitignore |
| 数据模型 | Project / Message / Task / File Pydantic 模型 |
| 本地存储 | LocalStorage 实现 JSON/JSONL 读写 |
| FastAPI 骨架 | 项目 CRUD API |
| 前端骨架 | 三栏布局 HTML + CSS 变量 + 基础样式 |

### Phase 2: Agent 核心（3天）

| 任务 | 说明 |
|------|------|
| MeDEX Agent | 创建 Agent 实例，配置模型和 system prompt |
| FileManager 工具 | 文件读写，关联项目目录 |
| SubtaskManager 工具 | 任务增删改查，JSON 持久化 |
| WebSearch 工具 | 简单搜索（requests + 搜索 API） |
| 对话 API | SSE 流式转发 Agent 事件 |

### Phase 3: 交互闭环（3天）

| 任务 | 说明 |
|------|------|
| UserInteraction 工具 | 外部执行工具，暂停/恢复机制 |
| 前端 SSE 渲染 | 流式文本、工具调用卡片、思考块 |
| 任务面板 | 实时更新 subtask 状态 |
| 选择框/输入框组件 | 渲染 ask_user 事件，提交交互响应 |
| MedicalDocGen 工具 | 生成 MD 文件，注册到项目 |

### Phase 4: 文件面板与收尾（2天）

| 任务 | 说明 |
|------|------|
| 文件上传 API | 支持多格式上传，存储到 uploads/ |
| 文件面板 | 列表 + Markdown 预览 + 图片预览 |
| 文件链接 | 对话中可点击文件链接，右侧面板联动 |
| 权限配置 | EXPLORE 模式为默认 |
| 端到端测试 | 完整流程：创建→上传→分析→交互→报告 |

---

## 11. 存储迁移路径

当前使用本地文件存储，后续迁移到 MySQL 时：

1. `StorageBase` 抽象类定义接口（`get_project`, `list_projects`, `save_message`, ...）
2. `LocalStorage` 实现当前方案（JSON 文件）
3. 后续新增 `MySQLStorage` 实现同一接口
4. 通过配置切换：`storage = LocalStorage()` → `storage = MySQLStorage(connection_string)`

无需修改业务逻辑代码。
