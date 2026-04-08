# Claude Code Skills 开发完整指南

**来源**: [code.claude.com/docs/extend-claude-with-skills](https://code.claude.com/docs/en/extend-claude-with-skills)  
**整理日期**: 2026年4月8日  
**适用对象**: 希望扩展Claude Code能力的开发者、团队技术负责人

---

## 🎯 什么是Skills？

Skills（技能）是扩展Claude能力的方式。创建包含指令的 `SKILL.md` 文件，Claude将其添加到工具包。

### 核心特性

- **自动加载**: Claude在相关时自动使用skill
- **手动调用**: 用 `/skill-name` 直接调用
- **遵循开放标准**: Agent Skills open standard（跨多个AI工具工作）
- **Claude Code扩展**: 添加调用控制、subagent执行、动态上下文注入等额外功能

### 与Commands的关系
Custom commands已合并到skills：
- `.claude/commands/deploy.md` 和 `.claude/skills/deploy/SKILL.md` 都创建 `/deploy` 且工作方式相同
- 现有 `.claude/commands/` 文件继续工作
- **Skills添加可选功能**: 目录存放支持文件、frontmatter控制调用方式、Claude自动加载

---

## 📦 Bundled Skills（内置Skills）

这些随Claude Code一起提供，**每个会话可用**。与执行固定逻辑的built-in commands不同，bundled skills是**基于prompt的**：给Claude详细playbook让它用工具编排工作。

| Skill | 用途 | 示例 |
|-------|------|------|
| `/batch <instruction>` | 并行编排大规模代码库更改。研究代码库→分解为5-30个独立单位→展示计划→批准后在隔离git worktree中为每个单位生成一个后台agent实现→运行测试→开PR | `/batch migrate src/ from Solid to React` |
| `/claude-api` | 为项目语言加载Claude API参考材料（Python, TypeScript, Java, Go, Ruby, C#, PHP, cURL）+ Agent SDK参考（Python和TypeScript）。覆盖tool use, streaming, batches, structured outputs等 | 代码导入anthropic或@anthropic-ai/sdk时自动激活 |
| `/debug [description]` | 启用当前会话debug logging并通过读session debug log排查问题。默认关闭除非以 `claude --debug` 启动 | `/debug 登录失败后token未刷新` |
| `/loop [interval] <prompt>` | 在间隔内重复运行prompt。适用于轮询部署、照看PR、周期性重新运行其他skill | `/loop 5m check if the deploy finished` |
| `/simplify [focus]` | 审查最近更改文件的代码重用、质量和效率问题然后修复。并行spawn三个review agent聚合发现并应用修复 | `/simplify focus on memory efficiency` |

---

## 🚀 快速开始：创建你的第一个Skill

### 示例：创建代码解释Skill

这个skill教Claude用视觉图解和类比解释代码。

#### Step 1: 创建skill目录

```bash
mkdir -p ~/.claude/skills/explain-code
```

**Personal skills** 对所有项目可用。

#### Step 2: 编写SKILL.md

每个skill需要两部分：

1. **YAML frontmatter**（在 `---` 标记之间）：告诉Claude何时使用skill
2. **Markdown内容**：调用时Claude遵循的指令

**文件**: `~/.claude/skills/explain-code/SKILL.md`

```markdown
---
name: explain-code
description: Explains code with visual diagrams and analogies. Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"
---

When explaining code, always include:

1. **Start with an analogy**: Compare the code to something from everyday life
2. **Draw a diagram**: Use ASCII art to show the flow, structure, or relationships
3. **Walk through the code**: Explain step-by-step what happens
4. **Highlight a gotcha**: What's a common mistake or misconception?

Keep explanations conversational. For complex concepts, use multiple analogies.
```

**关键字段说明**:
- `name`: 变成 `/slash-command`
- `description`: 帮助Claude决定何时自动加载

#### Step 3: 测试Skill

**方式1 - 让Claude自动调用**:
```
How does this code work?
```

**方式2 - 直接调用**:
```
/explain-code src/auth/login.ts
```

两种方式都应让Claude在解释中包含类比和ASCII图解

---

## 📍 Skill存储位置与作用域

| 位置 | 路径 | 适用范围 | 企业管理 |
|------|------|----------|---------|
| **Enterprise** | managed settings | 组织所有用户 | ✅ 见managed settings |
| **Personal** | `~/.claude/skills/<skill-name>/SKILL.md` | 你的所有项目 | - |
| **Project** | `.claude/skills/<skill-name>/SKILL.md` | 仅此项目 | - |
| **Plugin** | `<plugin>/skills/<skill-name>/SKILL.md` | Plugin启用处 | - |

### 优先级规则
当skills在不同层级同名时：**enterprise > personal > project**

Plugin skills使用 `plugin-name:skill-name` 命名空间，不会与其他层级冲突

**注意**: 如果skill和command同名，**skill优先**

### 自动发现嵌套目录

编辑子目录文件时，Claude Code自动发现嵌套的 `.claude/skills/` 目录中的skills：
- 编辑 `packages/frontend/` 中文件 → 也查找 `packages/frontend/.claude/skills/`
- 支持 **monorepo setup**（各包有自己的skills）

### 额外目录支持

`--add-dir` 标志通常只授予文件访问权而非配置发现，但 **skills是例外**：
- 添加目录内的 `.claude/skills/` 自动加载
- 支持 live change detection（会话中编辑skills无需重启）
- 其他.claude配置（subagents, commands, output styles）不从额外目录加载

---

## 📁 Skill目录结构

```text
my-skill/
├── SKILL.md           # 主指令（必需）
├── template.md        # Claude填写的模板
├── examples/
│   └── sample.md      # 显示预期格式的示例输出
└── scripts/
    └── validate.sh    # Claude可执行的脚本
```

**SKILL.md 包含主指令且必需**。其他文件可选，让你构建更强大的skills。

**引用支持文件**: 在SKILL.md中引用它们让Claude知道内容和何时加载

**建议**: 保持SKILL.md在500行以下。将详细参考资料移到单独文件

---

## ⚙️ 配置Skills

通过 **YAML frontmatter** 和 **markdown内容**配置

### Types of Skill Content（Skill内容类型）

#### 1. Reference Content（参考内容）
添加Claude应用于当前工作的知识。约定、模式、风格指南、领域知识。
**特点**: 内容inline运行，Claude可结合对话上下文使用

```markdown
---
name: api-conventions
description: API design patterns for this codebase
---

When writing API endpoints:
- Use RESTful naming conventions
- Return consistent error formats
- Include request validation
```

#### 2. Task Content（任务内容）
给Claude特定动作的分步指令，如部署、提交、代码生成。
**特点**: 通常你想直接用 `/skill-name` 调用而非让Claude决定何时运行
**建议**: 添加 `disable-model-invocation: true` 防止Claude自动触发

```markdown
---
name: deploy
description: Deploy the application to production
context: fork
disable-model-invocation: true
---

Deploy the application:
1. Run the test suite
2. Build the application
3. Push to the deployment target
```

### Frontmatter完整参考

所有字段可选，只有description推荐填写：

| 字段 | 必需 | 说明 | 示例值 |
|------|------|------|--------|
| **name** | 否 | 显示名称。省略时用目录名。仅小写字母数字和连字符（最多64字符） | `my-skill` |
| **description** | 推荐 | skill做什么和何时使用。Claude用它决定何时应用。超过250字符截断 | `Deploy the application to staging` |
| **argument-hint** | 否 | autocomplete时显示的参数提示 | `[issue-number]` 或 `[filename] [format]` |
| **disable-model-invocation** | 否 | 设true防止Claude自动加载。用于你想手动触发的工作流 | `true` |
| **user-invocable** | 否 | 设false从/菜单隐藏。用于背景知识用户不应直接调用的 | `false` |
| **allowed-tools** | 否 | skill激活时Claude无需许可使用的工具。空格分隔字符串或YAML列表 | `Read Grep Glob Bash(git *)` |
| **model** | 否 | skill激活时使用的模型 | `opus` 或 `sonnet` |
| **effort** | 否 | skill激活时的努力级别。覆盖session级别。选项: low, medium, high, max (Opus 4.6 only) | `high` |
| **context** | 否 | 设 `fork` 在forked subagent上下文中运行 | `fork` |
| **agent** | 否 | 当context: fork时使用的subagent类型 | `Explore`, `Plan`, 或自定义subagent名 |
| **hooks** | 否 | 此skill生命周期范围的hooks | 见Hooks配置格式 |
| **paths** | 否 | 限制skill激活的glob模式。逗号分隔字符串或YAML列表 | `"src/**/*.ts"` |
| **shell** | 否 | 此skill中!`command`和```!块的shell。bash(默认)或powershell | `powershell` |

---

## 🔧 字符串替换变量

Skills支持动态值字符串替换：

| 变量 | 说明 | 示例 |
|------|------|------|
| `$ARGUMENTS` | 调用时传递的所有参数。如果不存在则追加 `ARGUMENTS: <value>` | `/fix-issue 123` → `$ARGUMENTS`=123 |
| `$ARGUMENTS[N]` 或 `$N` | 按位置访问特定参数（0-based索引） | `$ARGUMENTS[0]` 或 `$0` = 第一个参数 |
| `${CLAUDE_SESSION_ID}` | 当前session ID。用于日志、创建session特定文件、关联输出与会话 | `logs/${CLAUDE_SESSION_ID}.log` |
| `${CLAUDE_SKILL_DIR}` | 包含skill的SKILL.md文件的目录路径。用于引用打包脚本或文件 | 引用scripts/下的辅助脚本 |

**示例 - Session Logger**:

```markdown
---
name: session-logger
description: Log activity for this session
---

Log the following to logs/${CLAUDE_SESSION_ID}.log:

$ARGUMENTS
```

---

## 📎 添加支持文件

Skills可在目录中包含多个文件，保持SKILL.md聚焦核心而让Claude按需访问详细资料。

**推荐结构**:
```
my-skill/
├── SKILL.md (必需 - 概述和导航)
├── reference.md (详细API docs - 按需加载)
├── examples.md (用法示例 - 按需加载)
└── scripts/
    └── helper.py (实用脚本 - 执行而非加载)
```

**在SKILL.md中引用**:
```markdown
## Additional resources

- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

---

## 🎛️ 控制谁可以调用Skill

### 默认行为
你和Claude都可以调用任何skill

### 两字段控制

| 字段 | 你可调用 | Claude可调用 | 何时加载到上下文 |
|------|---------|-------------|----------------|
| *(默认)* | ✅ Yes | ✅ Yes | Description始终在context，调用时加载完整skill |
| `disable-model-invocation: true` | ✅ Yes | ❌ No | **Description不在context**，你调用时才加载 |
| `user-invocable: false` | ❌ No | ✅ Yes | Description始终在context，调用时加载完整skill |

### 使用场景

#### disable-model-invocation: true
用于有副作用或你想控制时机的工作流：
- `/commit` 
- `/deploy`
- `/send-slack-message`

> 不想让Claude因为代码看起来准备好了就决定部署！

**示例 - Deploy Skill**:
```markdown
---
name: deploy
description: Deploy the application to production
disable-model-incovation: true
---

Deploy $ARGUMENTS to production:

1. Run the test suite
2. Build the application
3. Push to the deployment target
4. Verify the deployment succeeded
```

#### user-invocable: false
用于不是用户有意义操作的背景知识：
- legacy-system-context skill解释旧系统如何工作
- Claude应该在相关时知道，但 `/legacy-system-context` 不是用户想触发的操作

---

## 🔒 限制工具访问

用 `allowed-tools` 字段限制skill激活时Claude能用的工具

**示例 - 只读模式**:
```markdown
---
name: safe-reader
description: Read files without making changes
allowed-tools: Read Grep Glob
---
```

Claude只能探索文件但不能修改

---

## 💬 传递参数给Skills

### 基本用法
通过 `$ARGUMENTS` 占位符访问参数

**示例 - Fix GitHub Issue**:
```markdown
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.

1. Read the issue description
2. Understand the requirements
3. Implement the fix
4. Write tests
5. Create a commit
```
运行 `/fix-issue 123` → Claude收到 "Fix GitHub issue 123..."

### 按位置访问参数
用 `$ARGUMENTS[N]` 或简写 `$N`（0-based索引）

**示例 - 迁移组件**:
```markdown
---
name: migrate-component
description: Migrate a component from one framework to another
---

Migrate the $0 component from $1 to $2.
Preserve all existing behavior and tests.
```
运行 `/migrate-component SearchBar React Vue`:
- `$0` = SearchBar
- `$1` = React  
- `$2` = Vue

**无$ARGUMENTS时的行为**: 如果skill不包含$ARGUMENTS但带参数调用，Claude Code会在末尾追加 `ARGUMENTS: <your input>`

---

## 🔄 高级模式

### 动态上下文注入

**语法**: `!<command>` 在skill内容发送给Claude前运行shell命令

**特点**: 
- 每个命令立即执行（在Claude看到任何东西之前）
- 输出替换占位符
- Claude收到实际数据，非命令本身

**这是预处理，非Claude执行的东西。Claude只看到最终结果**

**示例 - PR摘要Skill**:
```markdown
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

**多行命令**: 用 ```! 而非行内形式：
```markdown
## Environment
```!
node --version
npm --version
git status --short
```
```

**安全控制**: 设置 `"disableSkillShellExecution": true` 可禁用来自user/project/plugin/additional-directory sources的shell命令替换。Bundled和managed skills不受影响。Managed settings中最有用（用户无法覆盖）

**启用扩展思维**: 在skill内容任何地方包含单词"ultrathink"

### Subagent中运行Skills

添加 `context: fork` 到frontmatter让skill在隔离环境中运行

**特点**:
- skill内容成为驱动subagent的prompt
- 无对话历史访问
- **只有显式任务的skill才有意义**
- 如果skill只有指南如"use these API conventions"而无任务 → subagent收到指南但无可操作prompt → 返回无意义输出

#### 两种配合方向

| 方法 | System prompt | Task | Also loads |
|------|---------------|------|------------|
| **Skill with context: fork** | 从agent type (Explore, Plan, etc.) | SKILL.md内容 | CLAUDE.md |
| **Subagent with skills field** | Subagent's markdown body | Claude的委托消息 | Preloaded skills + CLAUDE.md |

**用context: fork**: 在skill中写任务 + 选agent type执行  
**反向（定义自定义subagent用skills作参考材料）**: 见Subagents文档

### 示例：Research Skill using Explore Agent

```markdown
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

**执行流程**:
1. 创建新的隔离上下文
2. Subagent接收skill内容作为prompt ("Research $ARGUMENTS thoroughly...")
3. agent字段确定执行环境（model, tools, permissions）
4. 结果总结并返回主对话

**agent字段选项**: 内置agents (Explore, Plan, general-purpose) 或 `.claude/agents/` 中的任何自定义subagent。省略时使用general-purpose

---

## 🚫 限制Claude的Skill访问

### 三种控制方法

#### 1. 完全禁用Skills
在 `/permissions` 中拒绝Skill tool：
```
# Add to deny rules:
Skill
```

#### 2. 允许/禁止特定skills
用permission rules：
```
# Allow only specific skills
Skill(commit)
Skill(review-pr *)

# Deny specific skills
Skill(deploy *)
```

**权限语法**: 
- `Skill(name)` - 精确匹配
- `Skill(name *)` - 带任意参数的前缀匹配

#### 3. 隐藏个别skill
在其frontmatter中添加 `disable-model-invocation: true`
- 这完全从Claude上下文中移除skill

**重要**: `user-invocable` 字段只控制菜单可见性，不控制Skill tool访问。用 `disable-model-invocation: true` 阻止编程式调用

⚠️ **Built-in commands** 如 `/compact` 和 `/init` 不能通过Skill tool获得

---

## 📤 分享Skills

| 方式 | 适用场景 | 实现 |
|------|---------|------|
| **Project skills** | 团队共享 | 将 `.claude/skills/` 提交到版本控制 |
| **Plugins** | 打包分发 | 在plugin中创建 `skills/` 目录 |
| **Managed** | 组织级部署 | 通过managed settings部署组织范围 |

---

## 🎨 高级案例：可视化输出

Skills可以捆绑和运行任何语言脚本，赋予Claude超越单个prompt的能力。强大模式之一：**生成视觉输出**——交互HTML文件在浏览器中打开用于探索数据、调试或创建报告。

### 案例：Codebase Visualizer（代码库可视化器）

**效果**: 可交互树视图——展开折叠目录、一目了然看文件大小、颜色识别文件类型

#### 创建Skill结构

```bash
mkdir -p ~/.claude/skills/codebase-visualizer/scripts
```

#### SKILL.md

```markdown
---
name: codebase-visualizer
description: Generate an interactive collapsible tree visualization of your codebase. Use when exploring a new repo, understanding project structure, or identifying large files.
allowed-tools: Bash(python *)
---

# Codebase Visualizer

Generate an interactive HTML tree view that shows your project's file structure with collapsible directories.

## Usage

Run the visualization script from your project root:

```bash
python ~/.claude/skills/codebase-visualizer/scripts/visualize.py .
```

This creates `codebase-map.html` in the current directory and opens it in your default browser.

## What the visualization shows

- **Collapsible directories**: Click folders to expand/collapse
- **File sizes**: Displayed next to each file
- **Colors**: Different colors for different file types
- **Directory totals**: Shows aggregate size of each folder
```

**测试**: 在任何项目中打开Claude Code问"Visualize this codebase." Claude运行脚本、生成codebase-map.html、在浏览器打开

**此模式适用于任何视觉输出**:
- 依赖图
- 测试覆盖率报告
- API文档
- 数据库schema可视化

**原理**: 打包脚本做繁重工作，Claude处理编排

---

## 🔍 故障排除

### Skill未触发

检查清单：
1. ✅ description包含用户自然会说到的关键词
2. ✅ skill出现在"What skills are available?"
3. ✅ 尝试重新表述请求以更匹配description
4. ✅ 如果skill是user-invocable则用 `/skill-name` 直接调用

### Skill触发过于频繁

解决方案：
1. 使description更具体
2. 如果只想手动调用则添加 `disable-model-invocation: true`

### Skill描述被截断

**原因**: Skill descriptions被加载到context让Claude知道有什么可用。所有skill name始终包含，但如果有很多skills，descriptions会被缩短以适应字符预算

**预算规则**:
- 动态缩放为上下文窗口的 **1%**
- 回退值: **8,000字符**
- 每个条目无论预算如何上限 **250字符**

**解决方案**:
1. 设置环境变量 `SLASH_COMMAND_TOOL_CHAR_BUDGET` 提高限制
2. 从源头精简description：**前置关键用例**（最重要的信息放前面）

---

## 📚 相关资源

- **[Subagents](https://code.claude.com/docs/subagents)**: 委托任务给专业代理
- **[Plugins](https://code.claude.com/docs/plugins)**: 打包分发skills和其他扩展
- **[Hooks](https://code.claude.com/docs/hooks)**: 围绕工具事件自动化工作流
- **[Memory](https://code.claude.com/docs/memory)**: 管理CLAUDE.md文件持久化上下文
- **[Built-in commands](https://code.claude.com/docs/built-in-commands)**: 内置/命令参考
- **[Permissions](https://code.claude.com/docs/permissions)**: 控制工具和skill访问

---

**标准遵循**: Claude Code skills遵循 [Agent Skills open standard](https://github.com/anthropics/agent-skills)，该标准跨多个AI工具工作。Claude Code用调用控制、subagent执行和动态上下文注入扩展了标准
