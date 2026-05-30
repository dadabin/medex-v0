# 12. 技能系统

## 概述

技能（Skill）是基于 Markdown 的指令集，不是工具。Agent 不能直接"调用"技能，而是通过自动注册的 `Skill` 查看器工具阅读指令，然后使用现有工具遵循指令执行。

## 技能文件结构

每个技能是一个目录，包含 `SKILL.md` 文件：

```
skills/
  code-review/
    SKILL.md
  testing/
    SKILL.md
  deployment/
    SKILL.md
```

### SKILL.md 格式

```markdown
---
name: code-review
description: 代码审查技能，自动检查代码质量、安全性和最佳实践。
tags: [code, review, quality]
---

# 代码审查技能

## 概述
本技能指导 Agent 进行系统化的代码审查。

## 审查步骤

1. **读取代码**
   使用 Read 工具读取目标文件。

2. **检查代码质量**
   - 命名是否清晰
   - 函数是否过长
   - 是否有重复代码

3. **检查安全性**
   - SQL 注入风险
   - XSS 漏洞
   - 敏感信息暴露

4. **检查最佳实践**
   - 错误处理是否完善
   - 日志是否充分
   - 测试覆盖是否足够

5. **生成审查报告**
   使用 Markdown 格式输出审查结果。

## 输出格式
...
```

## 加载技能

### 直接路径

```python
from agentscope.tool import Toolkit

toolkit = Toolkit(skills_or_loaders=["/path/to/skills"])
```

### 使用 LocalSkillLoader

支持子目录扫描：

```python
from agentscope.skill import LocalSkillLoader
from agentscope.tool import Toolkit

loader = LocalSkillLoader(
    directory="/path/to/skills",
    scan_subdir=True,
)

toolkit = Toolkit(skills_or_loaders=[loader])
```

### 多路径加载

```python
toolkit = Toolkit(
    skills_or_loaders=[
        "/path/to/core-skills",
        "/path/to/project-skills",
        LocalSkillLoader(directory="/path/to/extra-skills", scan_subdir=True),
    ]
)
```

## 技能与工具的区别

| 特性 | 报能（Skill） | 工具（Tool） |
|------|---------------|-------------|
| 本质 | Markdown 指令文档 | Python 类/函数 |
| 调用方式 | Agent 阅读 SKILL.md 后自行执行 | Agent 直接调用 |
| 侧重点 | 告诉 Agent "怎么做" | 给 Agent "做什么" |
| 灵活性 | 高，Agent 可自行判断 | 低，固定输入输出 |
| 注册方式 | 自动注册 `Skill` 查看器 | 需要显式注册到 Toolkit |

## 技能工作流

```
1. Agent 收到任务
2. Agent 调用 Skill 查看器工具，浏览可用技能列表
3. Agent 选择相关技能，阅读 SKILL.md 内容
4. Agent 按照技能指令，使用已有工具逐步执行
5. Agent 根据执行结果和技能指引生成回复
```

## 完整示例

### 项目结构

```
my_project/
  skills/
    bug-fix/
      SKILL.md
    code-style/
      SKILL.md
  main.py
```

### bug-fix/SKILL.md

```markdown
---
name: bug-fix
description: Bug 修复技能，指导系统化的 bug 定位和修复。
tags: [bug, fix, debug]
---

# Bug 修复技能

## 修复步骤

1. **复现问题**
   - 理解 bug 描述
   - 运行相关代码复现问题
   - 记录错误信息和堆栈

2. **定位原因**
   - 阅读相关代码
   - 添加调试日志
   - 分析数据流

3. **实施修复**
   - 编写修复代码
   - 保持最小改动原则
   - 添加防御性代码

4. **验证修复**
   - 运行原有测试
   - 编写回归测试
   - 确认 bug 已修复

5. **代码审查**
   - 检查修复是否引入新问题
   - 确认代码风格一致
```

### main.py

```python
import os
import asyncio
from agentscope.agent import Agent
from agentscope.tool import Toolkit, Bash, Read, Write, Edit
from agentscope.skill import LocalSkillLoader
from agentscope.credential import DashScopeCredential
from agentscope.model import DashScopeChatModel
from agentscope.message import UserMsg

async def main():
    toolkit = Toolkit(
        tools=[Bash(), Read(), Write(), Edit()],
        skills_or_loaders=[
            LocalSkillLoader(directory="./skills", scan_subdir=True),
        ],
    )

    agent = Agent(
        name="dev_agent",
        system_prompt="你是一个开发助手，擅长修复 bug 和保持代码风格。",
        model=DashScopeChatModel(
            credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
            model="qwen3.6-plus",
        ),
        toolkit=toolkit,
    )

    result = await agent.reply(
        UserMsg(name="user", content="app.py 中的 login 函数在用户名为空时会崩溃，请修复")
    )
    print(result.get_text_content())

asyncio.run(main())
```

## 技能与 Tool Group 结合

可以将技能打包到 Tool Group 中，按需激活：

```python
from agentscope.tool import Toolkit, ToolGroup

toolkit = Toolkit(
    skills_or_loaders=["/path/to/skills"],
    tool_groups=[
        ToolGroup(
            name="code_quality",
            description="代码质量相关工具和技能。",
            instructions="始终遵循最佳实践。",
            tools=[lint_tool, format_tool],
        ),
    ],
)
```
