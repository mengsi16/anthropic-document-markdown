# Claude Code Agent Teams 完整指南

**来源**: [code.claude.com/docs/agent-teams](https://code.claude.com/docs/agent-teams) | 官方文档  
**整理日期**: 2026年4月8日  
**状态**: 🧪 Experimental（实验性功能，默认禁用）

---

## 📖 概述

Agent Teams 让你**协调多个独立的 Claude Code 实例作为一个团队共同工作**，具有共享任务列表、代理间消息传递和集中管理能力。

> ⚠️ **关键区别于 Subagents**:
> - **Subagent**: 单会话内运行，结果只报告回主代理
> - **Agent Team**: 每个 teammate 是完全独立的 Claude Code 实例，拥有自己的上下文窗口，**可以直接互相通信**

### 适用场景（最强场景）

| 场景 | 为什么适合 |
|------|-----------|
| **研究和审查** | 多个 teammate 可同时调查同一问题的不同方面，然后分享和互相挑战发现 |
| **新模块/功能开发** | 每个 teammate 独立负责不同部分，不会互相干扰 |
| **竞争假设调试** | teammate 并行测试不同理论，更快收敛到答案 |
| **跨层协调** | 前端/后端/测试变更由不同 teammate 各自负责 |

### 不适用场景

- 顺序任务
- 同文件编辑（导致冲突）
- 有很多依赖的工作
- 日常例行任务（单 session 更经济）

> Agent Teams 增加**协调开销**和**显著更高的 token 消耗**（每个 teammate 都是独立的 Claude 实例）。只在队友能**独立操作**时效果最佳。

---

## 🆚 Subagents vs Agent Teams 对比

| 特性 | Subagents | Agent Teams |
|------|-----------|-------------|
| **Context** | 自己的上下文窗口；结果返回给调用者 | 自己的上下文窗口；完全独立 |
| **通信** | 只报告结果回主代理 | Teammate 直接互相发消息 |
| **协调** | 主代理管理所有工作 | 共享任务列表+自我协调 |
| **Token 成本** | 较低：结果摘要返回主上下文 | 较高：每个 teammate 是独立实例 |
| **适用场景** | 只关注结果的聚焦任务 | 需要讨论和协作的复杂工作 |
| **嵌套** | 不能生成其他 subagent | 不能嵌套团队（teammate 不能生成团队）|

**选择原则**: 需要 quick focused workers 报告结果 → Subagents。需要 teammates 分享发现、互相质疑、自我协调 → Agent Teams。

---

## ⚙️ 启用 Agent Teams

默认禁用。通过设置环境变量启用：

```json
// settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

或在 shell 环境中设置：

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

**要求**: Claude Code v2.1.32 或更高版本。检查版本：`claude --version`

---

## 🚀 开始第一个 Agent Team

启用后，用自然语言告诉 Claude 创建团队、描述任务和团队结构：

> I'm designing a CLI tool that helps developers track TODO comments across their codebase. Create an agent team to explore this from different angles: one teammate on UX, one on technical architecture, one playing devil's advocate.

Claude 会：
1. 创建带有**共享任务列表**的团队
2. 为每个视角生成 teammate
3. 让他们探索问题
4. 综合发现
5. 尝试清理团队

Lead 的终端列出所有 teammate 和他们的工作状态。用 **Shift+Down** 循环切换 teammate 并直接发消息。

---

## 🎮 控制你的 Agent Team

### 显示模式选择

| 模式 | 说明 | 要求 |
|------|------|------|
| **In-process**（默认非 tmux）| 所有 teammate 在主终端内运行。Shift+Down 切换 | 任意终端 |
| **Split panes**（tmux/iTerm2）| 每个 teammate 一个 pane，同时看到所有人输出 | tmux 或 iTerm2 |
| **auto**（默认）| 已在 tmux 中则 split panes，否则 in-process | — |

配置全局默认值（`~/.claude.json`）：

```json
{
  "teammateMode": "in-process"
}
```

单次会话强制 in-process：

```bash
claude --teammate-mode in-process
```

**Split-pane 模式安装**：
- **tmux**: 通过系统包管理器安装
- **iTerm2**: 安装 it2 CLI，然后在 Preferences → General → Magic → Enable Python API

### 指定 Teammate 和模型

Claude 根据任务决定 teammate 数量，或你显式指定：

> Create a team with 4 teammates to refactor these modules in parallel. Use Sonnet for each teammate.

### 要求 Plan Approval

对于复杂或有风险的任务，可以让 teammate 先 plan 再实施：

> Spawn an architect teammate to refactor the authentication module. Require plan approval before they make any changes.

流程：
1. Teammate 在**只读 plan mode** 工作
2. 完成计划后向 lead 发送**计划批准请求**
3. Lead 自主审查并**批准**或**拒绝**（附反馈）
4. 被拒绝 → teammate 修订并重新提交
5. 被批准 → teammate 退出 plan mode 开始实施

可以通过 prompt 影响 lead 判断标准："only approve plans that include test coverage"

### 与 Teammate 直接对话

每个 teammate 是**完整的独立 Claude Code session**。

**In-process 模式**：
- Shift+Down 循环切换 teammate
- 输入发送消息
- Enter 查看 teammate session
- Escape 中断其当前 turn
- Ctrl+T 切换任务列表

**Split-pane 模式**：
- 点击进入 teammate pane 直接交互
- 每个 teammate 有完整终端视图

### 分配和认领任务

共享任务列表有三个状态：**pending → in progress → completed**

任务可以有**依赖关系**：未解决的依赖任务不能被认领。

两种分配方式：
- **Lead 分配**：告诉 lead 把哪个任务给哪个 teammate
- **Self-claim（自主认领）**：完成后自动领取下一个未分配、未阻塞的任务

> 认领使用**文件锁**防止竞态条件（多 teammate 同时尝试认领同一任务）

### 关闭 Teammate

优雅结束 teammate session：

> Ask the researcher teammate to shut down

Lead 发送关闭请求。teammate 可**批准退出**或**拒绝并解释原因**。

### 清理团队

> Clean up the team

移除共享团队资源。Lead 清理前会检查是否还有活跃 teammate，所以**先关闭所有 teammate 再清理**。

⚠️ 始终用 lead 来清理。Teammate 不应自己运行 cleanup（团队上下文可能无法正确解析）。

### Hooks 强制质量门

| Hook Event | 触发时机 | exit code 2 效果 |
|------------|----------|------------------|
| `TeammateIdle` | Teammate 即将空闲 | 发送反馈让其继续工作 |
| `TaskCreated` | 任务即将创建 | 阻止创建并发送反馈 |
| `TaskCompleted` | 任务即将标记完成 | 阻止完成并发送反馈 |

---

## 🏗️ 架构深度解析

### 组成部分

| 组件 | 角色 |
|------|------|
| **Team Lead** | 创建团队的 main Claude Code session，生成 teammate，协调工作 |
| **Teammates** | 处理已分配任务的独立 Claude Code 实例 |
| **Task List** | Teammate 认领和完成工作的共享列表 |
| **Mailbox** | Agent 间通信的消息系统 |

### 如何启动

两种方式：

1. **你请求团队**: 给 Claude 适合并行工作的任务，明确要求 agent team
2. **Claude 建议团队**: Claude 判定任务受益于并行工作时主动建议

两种情况下你都保持控制权，Claude 未经你批准不会创建团队。

### 使用 Subagent 定义作 Teammate

生成 teammate 时可以引用**任何 subagent scope**中的 subagent 类型（project/user/plugin/CLI-defined）：

> Spawn a teammate using the security-reviewer agent type to audit the auth module.

Teammate 会：
- 遵守该定义的 **tools allowlist 和 model**
- 该定义的 body **追加**（非替换）到 teammate 系统提示作为额外指令
- 团队协调工具（SendMessage、task management tools）**始终可用**
- ⚠️ `skills` 和 `mcpServers` frontmate 字段**不被应用**（teammate 从 project/user settings 加载）

### 权限

Teammate **启动时继承 lead 的权限设置**。如果 lead 用 `--dangerously-skip-permissions` 运行，所有 teammate 也一样。启动后可改变个别 teammate 模式，但不能在 spawn 时设定 per-teammate 模式。

### Context 与通信

**初始化**: 每个 teammate 加载与常规 session 相同的项目上下文：CLAUDE.md、MCP servers、skills。还收到 lead 的 spawn prompt。**Lead 的对话历史不传递**。

**信息共享机制**：

| 机制 | 说明 |
|------|------|
| **自动消息投递** | Teammate 发送的消息自动送达收件人，lead 无需轮询 |
| **Idle 通知** | Teammate 完成并停止时自动通知 lead |
| **共享任务列表** | 所有 agent 可见任务状态并认领可用工作 |

**Teammate 消息类型**：
- `message`: 发送消息给特定 teammate
- `broadcast**: 同时发给所有 teammate（慎用！成本随团队规模线性增长）

Lead 为每个 teammate 分配名字。任何 teammate 都可以用名字互相发消息。

### Token 消耗

Agent Teams 比**单 session 显著消耗更多 token**。每个 teammate 有自己的上下文窗口，token 使用随活跃 teammate 数量线性增长。对于研究/审查/新功能工作通常值得；日常任务用单 session 更经济。

### 存储

```
~/.claude/teams/{team-name}/config.json   # 团队配置（runtime state）
~/.claude/tasks/{team-name}/              # 任务列表
```

config 包含 members 数组（每 member 的 name, agent ID, agent type）。**不要手动编辑 config 或预先创建**——下次状态更新时会被覆盖。

---

## 📋 使用案例示例

### 案例 1: 并行代码审查

单个审查者倾向于一次关注一种问题类型。拆分审查标准到独立域意味着安全、性能和测试覆盖率同时得到充分关注：

> Create an agent team to review PR #142. Spawn three reviewers:
> - One focused on security implications
> - One checking performance impact
> - One validating test coverage
> Have them each review and report findings.

每个 reviewer 从同一个 PR 工作但应用不同 filter。Lead 在全部完成后**综合发现**。

### 案例 2: 竞争假设调查

根本原因不清楚时，单个 agent 倾向找到一个合理解释就停止。并行调查对抗这种倾向：

> Users report the app exits after one message instead of staying connected.
> Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
> each other to try to disprove each other's theories, like a scientific debate.
> Update the findings doc with whatever consensus emerges.

**辩论结构是关键机制**：顺序调查受锚定效应影响（一旦探索一个理论，后续调查偏向它）。多个独立调查者积极尝试**互相反驳**，存活下来的理论更可能是真正原因。

---

## ✅ 最佳实践

### 1. 给足 Teammate Context

Teammate 自动加载项目上下文（CLAUDE.md, MCP servers, skills），但不继承 lead 的对话历史。在 spawn prompt 中包含任务细节：

> Spawn a security reviewer teammate with the prompt: "Review the authentication module at src/auth/ for security vulnerabilities. Focus on token handling, session management, and input validation. The app uses JWT tokens stored in httpOnly cookies. Report any issues with severity ratings."

### 2. 选择合适的团队规模

没有硬性数量限制，但有实际约束：

| 因素 | 影响 |
|------|------|
| **Token 成本** | 随 teammate 数量**线性增长** |
| **协调开销** | 更多 teammate → 更多通信/协调/潜在冲突 |
| **收益递减** | 超过某点后，额外 teammate 不等比例提速 |

**推荐**: 大多数 workflow 从 **3-5 个 teammate** 开始。每个 teammate 保持 **5-6 个任务**保持高效而不至于过度上下文切换。

> 15 个独立任务？3 个 teammate 是好的起点。只在工作确实受益于同时进行时才扩展。**三个专注的 teammate 通常胜过五个分散的 teammate**。

### 3. 合理划分任务粒度

| 粒度 | 问题 |
|------|------|
| **太小** | 协调开销超过收益 |
| **太大** | Teammate 工作太久没有检查点，增加浪费风险 |
| **刚刚好** | 自包含单元，产出清晰交付物（一个函数、一个测试文件、一份审查）|

如果 lead 创建任务不够细，要求它拆分。

### 4. 等 Teammate 完成

有时 lead 开始自己实现任务而不是等待 teammate。如果注意到这种情况：

> Wait for your teammates to complete their tasks before proceeding

### 5. 从研究和审查开始

如果新手使用 agent teams，从有清晰边界且不需写代码的任务开始：审查 PR、研究库、调查 bug。这些展示**并行探索的价值**而没有**并行实现的协调挑战**。

### 6. 避免文件冲突

两个 teammate 编辑同一文件导致覆盖。确保每个 teammate **拥有不同的文件集合**。

### 7. 监控和引导

定期检查 teammate 进度，重定向不起效的方法，综合发现。让团队**无人值守太久**增加浪费风险。

---

## 🔧 故障排除

| 问题 | 解决方案 |
|------|----------|
| **Teammate 不出现** | In-process 模式按 Shift+Down 循环；检查任务是否复杂到值得建团队；确认 tmux 已安装（`which tmux`）；iTerm2 需 it2 CLI + Python API |
| **权限提示过多** | 在 spawn 前预批常用操作的权限设置 |
| **Teammate 遇错停止** | 用 Shift+Down/in-process 查看输出，然后给额外指令或 spawn 替换 teammate |
| **Lead 过早关闭** | 告诉它继续；也可告诉 lead 等 teammate 完成后再继续 |
| **孤儿 tmux session** | `tmux ls` 然后 `tmux kill-session -t <session-name>` |

---

## ⚠️ 当前限制

| 限制 | 详情 |
|------|------|
| **In-process teammate 无法恢复** | `/resume` 和 `/rewind` 不恢复 in-process teammate |
| **任务状态滞后** | Teammate 有时不标记任务完成，阻塞依赖任务。手动更新或 nudge |
| **关闭较慢** | Teammate 完成当前请求/tool call 后才关闭，可能耗时 |
| **一 session 一团队** | Lead 一次只能管理一个团队。清理后再建新团队 |
| **不支持嵌套团队** | Teammate 不能生成自己的团队或 teammate |
| **Lead 固定** | 创建团队的 session 就是 lead，不可转让领导权 |
| **权限 spawn 时设定** | 所有 teammate 启动时继承 lead 权限模式，之后可单独改 |
| **Split pane 要求 tmux/iTerm2** | 默认 in-process 模式适用于任意终端。VS Code 集成终端、Windows Terminal、Ghostty 不支持 split pane |

> **CLAUDE.md 正常工作**: Teammate 从工作目录读取 CLAUDE.md 文件。用此给所有 teammate 提供项目指导。

---

## 🔗 相关功能

| 功能 | 关系说明 |
|------|----------|
| **[Subagents](./09-subagents-complete-guide.md)** | 轻量级委派：session 内生成 helper agent，无需 inter-agent 协调 |
| **Git Worktrees** | 手动并行多 session 方案（无自动化团队协调）|
| **[Plugins](./11-plugins-complete-guide.md)** | 打包分发 subagents/skills/hooks/MCP 给团队 |

---

*文档来源: [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/agent-teams)*
