# Claude Code Plugins 完整开发指南

**来源**: [code.claude.com/docs/plugins](https://code.claude.com/docs/plugins) | 官方文档  
**整理日期**: 2026年4月8日

---

## 📖 概述

Plugins 让你用 **custom skills（技能）、agents（子代理）、hooks（钩子）和 MCP servers** 来扩展 Claude Code。Plugin 是可打包、可分发、可版本化的扩展单元，可通过 marketplace 分享给团队或社区。

### 核心价值

- **团队共享**: 一次定义 skills/agents/hooks，多项目/多人使用
- **版本控制**: Semantic versioning + 可更新
- **Marketplace 分发**: 通过官方或私有 marketplace 安装
- **命名空间隔离**: Plugin skills 自动加前缀避免冲突

---

## 🆚 Plugin vs 独立配置对比

| 方面 | 独立配置 (`.claude/` 目录) | Plugins |
|------|---------------------------|---------|
| **Skill 名称格式** | `/hello`（短名称）| `/plugin-name:hello`（命名空间前缀）|
| **适用场景** | 个人 workflow、项目定制、快速实验 | 团队共享、社区分发、跨项目复用、marketplace 发布 |
| **可见范围** | 单项目 | 多项目 / 社区 |
| **版本管理** | 无 | ✅ Semantic Versioning |
| **安装方式** | 手动创建文件 | `/plugin install` 或 `--plugin-dir` |

### 选择指南

**用独立配置 (`.claude/`)** 当：
- 为单个项目自定义 Claude Code
- 配置是个人化的不需要共享
- 在打包为 plugin 前实验 skills 或 hooks
- 想要短 skill 名称如 `/hello`

**用 Plugins** 当：
- 想与团队或社区分享功能
- 需要相同的 skills/agents 跨多个项目
- 想要版本控制和易更新
- 通过 marketplace 分发
- 接受命名空间前缀如 `/my-plugin:hello`

> 💡 **推荐工作流**: 先在 `.claude/` 中独立配置快速迭代，准备好分享时再转换为 Plugin。

---

## 🚀 快速开始：创建第一个 Plugin（带 Skill）

### Prerequisites

- Claude Code 已安装且已认证
- 如果看不到 `/plugin` 命令 → 更新 Claude Code 到最新版

### Step 1: 创建插件目录

```bash
mkdir my-first-plugin
```

每个 plugin 是一个包含 manifest 和你的 skills/agents/hooks 的目录。

### Step 2: 创建 Plugin Manifest

Manifest 文件 `.claude-plugin/plugin.json` 定义插件的身份：

```bash
mkdir my-first-plugin/.claude-plugin
```

```json
// my-first-plugin/.claude-plugin/plugin.json
{
  "name": "my-first-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

| 字段 | 用途 |
|------|------|
| `name` | 唯一标识符 + skill 命名空间（skills 前缀为此值）|
| `description` | Plugin manager 中显示的描述 |
| `version` | 语义化版本号 |
| `author` | 可选，用于归属 |

> 还有更多字段如 `homepage`, `repository`, `license` — 见完整 schema 参考。

### Step 3: 添加 Skill

Skills 存放在 `skills/` 目录中。每个 skill 是一个含 `SKILL.md` 文件的文件夹。文件夹名 = skill 名，加上 plugin 的命名空间前缀。

```bash
mkdir -p my-first-plugin/skills/hello
```

```markdown
<!-- my-first-plugin/skills/hello/SKILL.md -->
---
description: Greet the user with a friendly message
disable-model-invocation: true
---

Greet the user warmly and ask how you can help them today.
```

### Step 4: 测试 Plugin

使用 `--plugin-dir` flag 加载本地 plugin 进行开发测试：

```bash
claude --plugin-dir ./my-first-plugin
```

启动后尝试：

```
/my-first-plugin:hello
```

Claude 会以问候回应。运行 `/help` 可以看到 skill 列在 plugin namespace 下。

### Step 5: 添加 Skill 参数

用 `$ARGUMENTS` 占位符捕获用户输入：

```markdown
<!-- 更新 SKILL.md -->
---
description: Greet the user with a personalized message
---
# Hello Skill

Greet the user named "$ARGUMENTS" warmly and ask how you can help them today.
Make the greeting personal and encouraging.
```

运行 `/reload-plugins` 加载更改，然后测试：

```
/my-first-plugin:hello Alex
```

> **为什么需要命名空间？** Plugin skills 总是带命名空间前缀（如 `/my-first-plugin:hello`），防止不同 plugins 有同名 skills 时冲突。要改命名空间前缀，更新 `plugin.json` 中的 `name` 字段。

---

## 📁 Plugin 目录结构总览

```
my-plugin/
├── .claude-plugin/          # ⚠️ 只有 plugin.json 放这里！
│   └── plugin.json           # 插件清单（manifest）
│
├── commands/                 # Skills 作为 Markdown 文件
├── agents/                   # 自定义 agent 定义
├── skills/                   # Agent Skills（SKILL.md 文件）
├── hooks/                    # 事件处理器（hooks.json）
├── .mcp.json                 # MCP 服务器配置
├── .lsp.json                 # LSP 服务器配置（代码智能）
├── bin/                      # 启用时加入 Bash 工具 PATH 的可执行文件
└── settings.json             # 启用时应用的默认设置
```

> ⚠️ **常见错误**: 不要把 `commands/`, `agents/`, `skills/`, 或 `hooks/` 放进 `.claude-plugin/` 目录！只有 `plugin.json` 属于 `.claude-plugin/`。所有其他目录必须在 **plugin root level**。

---

## 🔧 开发更复杂的 Plugins

### 添加 Skills 到 Plugin

Plugin 可包含 Agent Skills 扩展 Claude 能力。Skills 是 model-invoked：Claude 根据任务上下文自动使用它们。

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── code-review/
        └── SKILL.md
```

```markdown
---
name: code-review
description: Reviews code for best practices and potential issues. Use when reviewing code, checking PRs, or analyzing code quality.
---
When reviewing, check for:
1. Code organization and structure
2. Error handling
3. Security concerns
4. Test coverage
```

安装后运行 `/reload-plugins` 加载 Skills。完整 Skill 编写指南见 [Agent Skills](./08-claude-code-skills-complete-guide.md)。

### 添加 LSP Servers

对于 TypeScript、Python、Rust 等常见语言，从官方 marketplace 安装预构建 LSP plugins。只在需要支持未覆盖语言时创建自定义 LSP plugin。

LSP（Language Server Protocol）plugins 给 Claude 提供**实时代码智能**：

```json
// .lsp.json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

用户需在机器上安装语言服务器二进制文件。

### 打包默认 Settings

Plugin 可在根目录包含 `settings.json`，启用时应用默认配置。目前只支持 `agent` key——设置 agent 激活 plugin 的一个自定义 agent 作为主线程：

```json
{
  "agent": "security-reviewer"
}
```

这会激活 plugin `agents/` 目录中的 `security-reviewer` agent 定义，应用其系统提示、工具限制和模型。

Settings from `settings.json` priority 高于 `plugin.json` 中声明的 settings。未知 key 被静默忽略。

### 组织复杂 Plugin

对于有很多组件的 plugin，按功能组织目录结构。

---

## 🧪 本地测试 Plugin

使用 `--plugin-dir` flag 开发期间测试：

```bash
claude --plugin-dir ./my-plugin
```

当 `--plugin-dir` plugin 与已安装的 marketplace plugin 同名时，**本地副本优先**（该 session 内）。这让无需先卸载就能测试已安装 plugin 的更改。

⚠️ Managed settings 强制启用的 marketplace plugins 是唯一例外——不能被覆盖。

做更改后运行 `/reload-plugins` 无重启加载更新（重载 plugins、skills、agents、hooks、MCP servers、LSP servers）。

**测试各组件**：
- 用 `/plugin-name:skill-name` 测试 skills
- 用 `/agents` 检查 agents 是否出现
- 验证 hooks 正常触发

**同时加载多个 plugin**：

```bash
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

---

## 🐛 调试 Plugin 问题

如果 plugin 不按预期工作：

1. **检查结构**: 确认目录在 **plugin root** 而非 `.claude-plugin/` 内部
2. **单独测试组件**: 分别检查每个 command、agent、hook
3. **使用验证和调试工具**: 见 Debugging and development tools CLI commands

---

## 📦 分享和分发 Plugin

当 plugin 准备好分享：

1. **添加文档**: 包含 README.md 说明安装和使用方法
2. **版本化**: 在 plugin.json 使用语义化版本
3. **创建/使用 Marketplace**: 通过 marketplace 分发安装
4. **团队测试**: 广泛分发前让团队成员测试

### 提交到官方 Marketplace

通过应用内提交表单：

- **Claude.ai**: claude.ai/settings/plugins/submit
- **Console**: platform.claude.com/plugins/submit

完整技术规格、调试技术和分发策略见 **Plugins reference**。

---

## 🔄 从独立配置迁移到 Plugin

如果你已在 `.claude/` 目录有 skills 或 hooks，可转换为 plugin 以便分享和分发。

### 迁移步骤

#### Step 1: 创建 Plugin 结构

```bash
mkdir -p my-plugin/.claude-plugin
```

创建 manifest:

```json
{
  "name": "my-plugin",
  "description": "Migrated from standalone configuration",
  "version": "1.0.0"
}
```

#### Step 2: 复制现有文件

```bash
# 复制 commands
cp -r .claude/commands my-plugin/

# 复制 agents（如果有）
cp -r .claude/agents my-plugin/

# 复制 skills（如果有）
cp -r .claude/skills my-plugin/
```

#### Step 3: 迁移 Hooks

如果在 settings 中有 hooks，创建 hooks 目录：

```bash
mkdir my-plugin/hooks
```

```json
// my-plugin/hooks/hooks.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npm run lint:fix" }]
      }
    ]
  }
}
```

Hook 通过 stdin 接收 JSON 输入，用 `jq` 提取字段。

#### Step 4: 测试迁移后的 Plugin

```bash
claude --plugin-dir ./my-plugin
```

测试每个组件：运行 commands、检查 agents 出现、验证 hooks 触发。

### 变化对照表

| 方面 | 独立 (`.claude/`) | Plugin |
|------|-------------------|--------|
| 可见范围 | 仅一个项目 | 可通过 marketplaces 共享 |
| Commands 位置 | `.claude/commands/` | `plugin-name/commands/` |
| Hooks 位置 | settings.json | `hooks/hooks.json` |
| 分享方式 | 手动复制 | `/plugin install` |

迁移完成后可以删除 `.claude/` 中的原始文件避免重复。Plugin 版本加载时优先于原始文件。

---

## 🔗 相关功能

| 功能 | 关系说明 |
|------|----------|
| **[Skills](./08-claude-code-skills-complete-guide.md)** | Skill 开发完整细节（frontmatter、高级模式等）|
| **[Subagents](./09-subagents-complete-guide.md)** | Agent 配置和能力 |
| **[Hooks](../06-claude-code-best-practices.md#环境配置五大件)** | 事件处理和自动化 |
| **[MCP](../05-claude-code-overview.md)** | 外部工具集成 |
| **[Agent Teams](./10-agent-teams-complete-guide.md)** | 多实例并行协调 |

---

*文档来源: [Create plugins - Claude Code Docs](https://code.claude.com/docs/plugins)*
