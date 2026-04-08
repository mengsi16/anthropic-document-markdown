# Claude Code Subagents 完整指南

**来源**: [code.claude.com/docs/subagents](https://code.claude.com/docs/en/subagents) | 官方文档  
**整理日期**: 2026年4月8日

---

## 📖 概述

Subagents（子代理）是 **Claude Code 中运行在独立上下文窗口中的专业化 AI 助手**。每个 subagent 拥有自定义的系统提示、特定工具访问权限和独立权限。当 Claude 遇到匹配 subagent 描述的任务时，会将任务委派给该 subagent，subagent 独立工作并返回结果。

> ⚠️ **重要区分**: Subagent 在单个会话内运行，结果只返回给主代理。如果需要多个 agent **并行工作且互相通信**，请使用 [Agent Teams](./10-agent-teams-complete-guide.md)。

### Subagents 的核心价值

| 价值 | 说明 |
|------|------|
| **保留上下文** | 将探索和实现隔离出主对话，避免主对话膨胀 |
| **强制约束** | 限制 subagent 可使用的工具 |
| **跨项目复用** | 通过 user-level subagents 在多项目间共享配置 |
| **行为专业化** | 为特定领域提供专注的系统提示 |
| **控制成本** | 将任务路由到更快速、更便宜的模型（如 Haiku） |

---

## 🔧 内置 Subagents

Claude Code 包含三个内置 subagent，Claude 会根据情况自动使用：

### 1. Explore（探索者）

| 属性 | 值 |
|------|-----|
| **模型** | Haiku（快速、低延迟）|
| **工具** | 只读工具（拒绝 Write 和 Edit）|
| **用途** | 文件发现、代码搜索、代码库探索 |

Claude 在需要搜索或理解代码库而不做更改时委派给 Explore。保持探索结果不出现在主对话中。

调用时可指定 **thoroughness level（彻底程度级别）**：
- `quick` — 快速定向查找
- `medium` — 平衡的探索
- `very thorough` — 全面分析

### 2. Plan（规划者）

| 属性 | 值 |
|------|-----|
| **模型** | 继承主对话 |
| **工具** | 只读工具 |
| **用途** | 规划模式下的代码库研究 |

在 **Plan Mode** 中，当 Claude 需要了解代码库时会将研究委派给 Plan subagent。防止无限嵌套（subagent 不能生成其他 subagent），同时仍收集必要的上下文。

### 3. General-purpose（通用）

| 属性 | 值 |
|------|-----|
| **模型** | 继承主对话 |
| **工具** | 所有工具 |
| **用途** | 复杂研究、多步操作、代码修改 |

当任务需要同时探索和修改、复杂推理或多步依赖操作时委派。

### 其他内置 Helper Agents

| Agent | 模型 | 用途 |
|-------|------|------|
| statusline-setup | Sonnet | 运行 `/statusline` 配置状态栏 |
| Claude Code Guide | Haiku | 回答关于 Claude Code 功能的问题 |

---

## 🚀 快速开始：创建第一个 Subagent

通过 `/agents` 命令交互式创建：

```
步骤1: 运行 /agents 打开 subagent 界面
步骤2: 选择 Create new agent → 选择 Personal（保存到 ~/.claude/agents/）
步骤3: 选择 Generate with Claude，描述你的 subagent
步骤4: 选择可用工具
步骤5: 选择模型
步骤6: 选择显示颜色
步骤7: 配置 memory 范围
步骤8: 保存并试用
```

示例描述 prompt：

```
A code improvement agent that scans files and suggests improvements 
for readability, performance, and best practices. It should explain 
each issue, show the current code, and provide an improved version.
```

---

## 📁 Subagent 存储位置与优先级

Subagent 是带 YAML frontmatter 的 Markdown 文件，存储在不同位置决定其作用域。**同名 subagent 以高优先级为准**。

| 位置 | 作用域 | 优先级 | 创建方式 |
|------|--------|--------|----------|
| Managed settings（管理员部署）| 组织范围 | 1（最高）| 通过 managed settings 部署 |
| `--agents` CLI 标志 | 当前会话 | 2 | 启动时传 JSON 参数 |
| `.claude/agents/` | 当前项目 | 3 | 手动创建或交互式 |
| `~/.claude/agents/` | 所有项目 | 4 | 交互式或手动 |
| Plugin 的 `agents/` 目录 | 已安装插件 | 5（最低）| 通过插件安装 |

### 项目 vs 用户 Subagent

- **`.claude/agents/`**: 理想的项目专属 subagent，可提交版本控制供团队协作改进
- **`~/.claude/agents/`**: 个人 subagent，在所有项目中可用
- **CLI 定义**: 仅存在于当前会话，不保存到磁盘，适合快速测试或自动化脚本

### CLI 定义示例

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on code quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  },
  "debugger": {
    "description": "Debugging specialist for errors and test failures.",
    "prompt": "You are an expert debugger. Analyze errors, identify root causes, and provide fixes."
  }
}'
```

`--agents` 标志接受与文件型 subagent 相同的 frontmatter 字段。

---

## ✍️ 编写 Subagent 文件

基本结构：

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

- **frontmatter**: 定义元数据和配置
- **body**: 成为驱动 subagent 行为的系统提示

> Subagent 只接收此系统提示（加上基本环境信息如工作目录），**不接收完整的 Claude Code 系统提示**。
>
> Subagent 在会话启动时加载。如果手动添加文件，需重启会话或用 `/agents` 立即加载。

---

## 📋 Frontmatter 字段完整参考

以下字段可用于 YAML frontmatter。只有 `name` 和 `description` 是必需的。

| 字段 | 必需 | 说明 | 示例值 |
|------|------|------|--------|
| `name` | ✅ | 唯一标识符，小写字母+连字符 | `code-reviewer` |
| `description` | ✅ | Claude 何时应委派给此 subagent | `"Reviews code for quality..."` |
| `tools` | ❌ | subagent 可用的工具（白名单）。省略则继承全部 | `Read, Glob, Grep, Bash` |
| `disallowedTools` | ❌ | 要拒绝的工具（黑名单） | `Write, Edit` |
| `model` | ❌ | 使用的模型：`sonnet` / `opus` / `haiku` / 完整模型ID / `inherit`（默认） | `sonnet`, `haiku`, `inherit` |
| `permissionMode` | ❌ | 权限模式 | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | ❌ | 最大 agentic turns 数后停止 | `20` |
| `skills` | ❌ | 启动时加载到 subagent 上下文的 skills | `- api-conventions` |
| `mcpServers` | ❌ | 此 subagent 可用的 MCP 服务器 | 见下方详细说明 |
| `hooks` | ❌ | 作用于此 subagent 的生命周期 hooks | 见下方详细说明 |
| `memory` | ❌ | 持久化记忆范围：`user` / `project` / `local` | `user` |
| `background` | ❌ | 设为 `true` 始终在后台运行 | `true` |
| `effort` | ❌ | 活跃时的 effort level。覆盖 session 级别设置 | `low`, `medium`, `high`, `max` (仅 Opus 4.6) |
| `isolation` | ❌ | 设为 `worktree` 在临时 git worktree 中运行 | `worktree` |
| `color` | ❌ | 任务列表和 transcript 中的显示颜色 | `red`, `blue`, `green` 等 |
| `initialPrompt` | ❌ | 作为主 session agent 运行时的首个用户 turn（自动提交） | `"Start by analyzing..."` |

### 模型解析顺序

当 Claude 调用 subagent 时，按以下顺序解析模型：

```
1. CLAUDE_CODE_SUBAGENT_MODEL 环境变量（如果设置了）
2. 每次 invocation 时传递的 model 参数
3. subagent 定义的 model frontmatter
4. 主对话的模型
```

---

## 🛡️ 控制 Subagent 能力

### 工具访问控制

**白名单模式** (`tools`):

```yaml
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
---
# subagent 不能编辑文件或使用 MCP 工具
```

**黑名单模式** (`disallowedTools`):

```yaml
---
name: no-writes
description: Inherits every tool except file writes
disallowedTools: Write, Edit
---
# 保留 Bash, MCP 工具和其他一切
```

> 如果同时设置两者：先应用 `disallowedTools`，再对剩余池解析 `tools`。两个列表中都出现的工具会被移除。

### 限制可生成的 Subagent 类型

当一个 agent 作为主线程运行（`claude --agent`）时，可以使用 `Agent(tool)` 语法限制它可生成的 subagent 类型：

```yaml
---
name: coordinator
description: Coordinates work across specialized agents
tools: Agent(worker, researcher), Read, Bash
---
# 只有 worker 和 researcher 可以被生成
# 尝试生成其他类型会导致失败
```

要允许生成任何 subagent 而不加限制：

```yaml
tools: Agent, Read, Bash
```

> 如果完全省略 `Agent`，该 agent 无法生成任何 subagent。
> 此限制仅适用于作为主线程运行的 agent（`claude --agent`）。Subagent 本身不能生成其他 subagent。

### MCP 服务器作用域

```yaml
---
name: browser-tester
description: Tests features in a real browser using Playwright
mcpServers:
  # 内联定义：仅限此 subagent
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  # 按名称引用：复用已配置的服务器
  - github
---
```

内联定义使用与 `.mcp.json` 相同的 schema。要将 MCP 服务器完全排除在主对话之外以节省上下文，在此处内联定义而非在 `.mcp.json` 中。

### 权限模式

| 模式 | 行为 |
|------|------|
| `default` | 标准权限检查 + 提示确认 |
| `acceptEdits` | 自动接受文件编辑（保护目录除外）|
| `auto` | Auto mode：后台分类器审查命令和保护目录写入 |
| `dontAsk` | 自动拒绝权限提示（显式允许的工具仍正常工作）|
| `bypassPermissions` | 跳过权限提示 |
| `plan` | Plan mode（只读探索）|

> ⚠️ **bypassPermissions 注意**: 跳过权限提示允许无审批执行操作。但写入 `.git`、`.claude`、`.vscode`、`.idea`、`.husky` 目录仍会提示确认（`.claude/commands`、`.claude/agents`、`.claude/skills` 除外）。
>
> 如果父级使用 `bypassPermissions`，则优先级最高，无法被覆盖。如果父级使用 `auto mode`，subagent 继承 auto mode 且其 `permissionMode` 会被忽略。

### Preload Skills 到 Subagent

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
---
Implement API endpoints. Follow the conventions and patterns from the preloaded skills.
```

skill 的**完整内容**被注入到 subagent 上下文中，而不仅仅是使其可用。**Subagent 不从父对话继承 skills；必须明确列出**。

### 持久化 Memory

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: user
---
You are a code reviewer. As you review, update your agent memory with
patterns, conventions, and recurring issues you discover.
```

Memory 范围选项：

| 范围 | 存储位置 | 适用场景 |
|------|----------|----------|
| `user` | `~/.claude/agent-memory/<agent-name>/` | 跨所有项目记住学习内容 |
| `project` | `.claude/agent-memory/<agent-name>/` | 项目特有知识，可通过版本控制共享（推荐默认）|
| `local` | `.claude/agent-memory-local/<agent-name>/` | 项目特有但不应提交版本控制 |

启用 memory 后：
- 系统提示包含读写 memory 目录的指令
- 自动注入 MEMORY.md 前200行或25KB的内容
- 自动启用 Read、Write、Edit 工具以便管理 memory 文件

**Memory 最佳实践**：
1. 让 subagent 在开始工作前查阅 memory："Review this PR, and check your memory for patterns you've seen before."
2. 完成后更新 memory："Now that you're done, save what you learned to your memory."
3. 在 subagent 的 markdown 文件中直接包含 memory 维护指令

### Hooks（条件规则）

使用 PreToolUse hook 在工具执行前验证操作：

```yaml
---
name: db-reader
description: Execute read-only database queries
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---
```

Hook 输入验证脚本（`validate-readonly-query.sh`）:

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# 阻止 SQL 写操作（不区分大小写）
if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE)\b' > /dev/null; then
  echo "Blocked: Only SELECT queries are allowed" >&2
  exit 2
fi

exit 0
```

退出码 `2` = 阻止操作并将错误消息反馈给 Claude。

### 禁止特定 Subagent

在 settings 中使用 deny 数组阻止 Claude 使用特定 subagent：

```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom-agent)"]
  }
}
```

也支持 CLI 标志：

```bash
claude --disallowedTools "Agent(Explore)"
```

---

## 💼 使用 Subagents

### 自动委派

Claude 根据 task description、subagent 配置中的 description 字段和当前上下文**自动决定何时委派**。在 description 中加入 "use proactively" 等短语鼓励主动委派。

### 显式调用 Subagent

三种模式，从一次性到全 session：

#### 1. 自然语言（无特殊语法）

```
Use the test-runner subagent to fix failing tests
Have the code-reviewer subagent look at my recent changes
```
→ Claude 决定是否委派

#### 2. @-mention（保证执行一次）

```
@"code-reviewer (agent)" look at the auth changes
```
→ 确保指定 subagent 执行。输入 `@` 后从 typeahead 中选择。

#### 3. Session-wide（整个 session 使用该 subagent 的系统提示/工具/模型）

```bash
claude --agent code-reviewer
```

subagent 的系统提示**完全替换**默认 Claude Code 系统提示（与 `--system-prompt` 相同方式）。CLAUDE.md 和 project memory 仍通过正常消息流加载。

也可以在 `.claude/settings.json` 中设为项目默认：

```json
{
  "agent": "code-reviewer"
}
```

CLI 标志覆盖 setting（如果两者都存在）。

### 前台 vs 后台运行

| 模式 | 特点 |
|------|------|
| **前台（Foreground）** | 阻塞主对话直到完成。权限提示和澄清问题传递给你 |
| **后台（Background）** | 并发运行，你继续工作。启动前预批所需权限。需要澄清问题时工具调用会失败但 subagent 继续运行 |

控制方式：
- 让 Claude "run this in the background"
- 按 **Ctrl+B** 将正在运行的任务转到后台

禁用所有后台功能：设置环境变量 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`

---

## 📐 常见使用模式

### 模式 1：隔离高容量操作

最有效的用途之一——隔离产生大量输出的操作（运行测试、获取文档、处理日志文件）：

> Use a subagent to run the test suite and report only the failing tests with their error messages

### 模式 2：并行调研

对独立调查任务，生成多个 subagent 同时工作：

> Research the authentication, database, and API modules in parallel using separate subagents

每个 subagent 独立探索各自区域，然后 Claude 综合发现。**最适合各调研路径互不依赖的情况**。

### 模式 3：链式 Subagent

对于多步工作流，让 Claude 按序使用 subagent：

> Use the code-reviewer subagent to find performance issues, then use the optimizer subagent to fix them

### 何时用主对话 vs Subagent

| 使用主对话 | 使用 Subagent |
|------------|---------------|
| 需要频繁来回迭代 | 任务产生大量不需要的主上下文输出 |
| 多阶段共享大量上下文（规划→实现→测试）| 想强制特定工具限制或权限 |
| 快速、针对性修改 | 工作自包含并可返回摘要 |
| 延迟敏感（subagent 从零开始可能耗时）| — |

考虑 **Skills** 当你想要可在主对话上下文中运行的**可复用 prompts/workflows**，而非隔离的 subagent 上下文。对于关于当前对话中已有内容的快速问题，用 `/btw` 替代 subagent。

> ⚠️ **Subagent 不能嵌套**：Subagent 不能生成其他 subagent。如果工作流需要嵌套委派，用 Skills 或从主对话链式调用 subagent。

---

## 🔄 管理 Subagent Context

### Resume Subagent

每次 subagent 调用创建新实例（全新 context）。要继续已有 subagent 的工作而非重新开始，要求 Claude resume 它。Resumed subagent 保留完整对话历史。

Resume 机制：当 subagent 完成后，Claude 收到其 agent ID，使用 `SendMessage` tool（`to` 字段 = agent ID）来 resume。

> **SendMessage tool 仅在 Agent Teams 启用时可用**（`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）。

### Auto-Compaction

Subagent 支持自动压缩，与主对话相同逻辑。默认在约 **95% 容量**时触发。要提前触发，设置：

```bash
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50
```

压缩事件记录在 subagent transcript 文件中：

```json
{
  "type": "system",
  "subtype": "compact_boundary",
  "compactMetadata": {
    "trigger": "auto",
    "preTokens": 167189
  }
}
```

---

## 📝 示例 Subagents

### 示例 1: Code Reviewer（只读审查）

```yaml
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---
You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage
- Performance considerations addressed

Provide feedback organized by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)

Include specific examples of how to fix issues.
```

### 示例 2: Debugger（诊断+修复）

```yaml
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
---
You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works

Debugging process:
- Analyze error messages and logs
- Check recent code changes
- Form and test hypotheses
- Add strategic debug logging
- Inspect variable states

For each issue, provide:
- Root cause explanation
- Evidence supporting the diagnosis
- Specific code fix
- Testing approach
- Prevention recommendations

Focus on fixing the underlying issue, not the symptoms.
```

### 示例 3: Data Scientist（领域专用）

```yaml
---
name: data-scientist
description: Data analysis expert for SQL queries, BigQuery operations, and data insights. Use proactively for data analysis tasks and queries.
tools: Bash, Read, Write
model: sonnet
---
You are a data scientist specializing in SQL and BigQuery analysis.

When invoked:
1. Understand the data analysis requirement
2. Write efficient SQL queries
3. Use BigQuery CLI tools (bq) when appropriate
4. Analyze and summarize results
5. Present findings clearly

Key practices:
- Write optimized SQL queries with proper filters
- Use appropriate aggregations and joins
- Include comments explaining complex logic
- Format results for readability
- Provide data-driven recommendations

Always ensure queries are efficient and cost-effective.
```

### 示例 4: Database Query Validator（Hooks 高级用法）

```yaml
---
name: db-reader
description: Execute read-only database queries. Use when analyzing data or generating reports.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---
You are a database analyst with read-only access. Execute SELECT queries to answer questions about the data.

When asked to analyze data:
1. Identify which tables contain the relevant data
2. Write efficient SELECT queries with appropriate filters
3. Present results clearly with context

You cannot modify data. If asked to INSERT, UPDATE, DELETE, or modify schema, explain that you only have read access.
```

---

## 🔗 相关功能

| 功能 | 关系说明 |
|------|----------|
| **[Plugins](./11-plugins-complete-guide.md)** | 通过 plugins 分发 subagents 给团队/社区 |
| **[Agent Teams](./10-agent-teams-complete-guide.md)** | 多个独立 Claude Code 实例协调工作（跨会话通信）|
| **MCP Servers** | 给 subagent 提供外部工具和数据访问 |
| **Skills** | 可复用的 prompt/workflow（运行在主对话上下文）|
| **Hooks** | subagent 生命周期事件处理 |

---

*文档来源: [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/subagents)*
