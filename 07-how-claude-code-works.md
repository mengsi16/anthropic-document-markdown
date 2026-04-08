# Claude Code 工作原理深度解析

**来源**: [code.claude.com/docs/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works)  
**整理日期**: 2026年4月8日

---

## 🔄 Agentic Loop（代理循环）

当你给Claude一个任务时，它经历三个阶段：**gather context（收集上下文）、take action（采取行动）、verify results（验证结果）**。

这三个阶段混合在一起。Claude全程使用工具，无论是搜索文件理解代码、编辑做更改还是运行测试检查工作。循环适应你所要求的：

| 任务类型 | 可能的阶段组合 |
|----------|---------------|
| 关于代码库的问题 | 可能只需要上下文收集 |
| Bug修复 | 反复循环所有三个阶段 |
| 重构 | 可能涉及广泛验证 |

**Claude根据上一步学到的内容决定每步需要什么**，链式执行几十个动作并在过程中纠偏调整

### 你也是循环的一部分

你可以随时中断来引导Claude朝不同方向前进、提供额外上下文或要求尝试不同方法。
- Claude自主工作但对你的输入保持响应

### 两大组件

Agentic loop由两个组件驱动：
1. **Models（模型）**: 推理
2. **Tools（工具）**: 行动

**Claude Code作为agentic harness围绕Claude**: 它提供工具、上下文管理和执行环境，将语言模型转变为有能力编程的代理

---

## 🤖 Models（模型）

Claude Code使用Claude模型理解代码和推理任务：
- 能读取任何语言代码
- 理解组件如何连接
- 弄清完成目标需要做什么更改
- 对于复杂任务分解为步骤、执行并根据所学调整

### 可用模型

| 模型 | 特点 | 适用场景 |
|------|------|----------|
| **Sonnet** | 处理大多数编码任务良好 | 日常编码 |
| **Opus** | 更强推理能力 | 复杂架构决策 |

**切换方式**: 会话中用 `/model`，或启动时 `claude --model <name>`

**注**: 当本文说"Claude chooses"或"Claude decides"时，是模型在做推理

---

## 🛠️ Tools（工具）

**工具使Claude Code具有agent能力**。没有工具，Claude只能用文本响应。有了工具，Claude能行动。

### 五大内置工具类别

| 类别 | Claude能做什么 |
|------|---------------|
| **File operations** | 读文件、编辑代码、创建新文件、重命名和重组 |
| **Search** | 按模式查找文件、正则搜索内容、探索代码库 |
| **Execution** | 运行shell命令、启动服务器、运行测试、使用git |
| **Web** | 搜索网络、获取文档、查询错误信息 |
| **Code intelligence** | 编辑后看到类型错误和警告、跳转到定义、查找引用（需要插件）|

### 工具使用示例："Fix the failing tests"

当你说"修复失败的测试"时，Claude可能：

1. **运行测试套件**看看什么失败了
2. **读取错误输出**
3. **搜索相关源文件**
4. **读这些文件**理解代码
5. **编辑文件**解决问题
6. **再次运行测试**验证

**每个工具使用给Claude新信息通知下一步决策。这就是agentic loop的实际运作。**

### 扩展基础能力
内置工具是基础层。你可以用以下方式扩展：
- **Skills**: 扩展Claude的知识
- **MCP**: 连接外部服务
- **Hooks**: 自动化工作流
- **Subagents**: 卸载任务

这些扩展形成核心agentic loop之上的层

---

## 👁️ Claude能访问什么

当你在目录中运行 `claude` 时，Claude Code获得访问：

| 资源 | 说明 |
|------|------|
| **Your project** | 目录和子目录中的文件，以及经许可的其他地方文件 |
| **Your terminal** | 你能运行的任何命令：构建工具、git、包管理器、系统实用程序、脚本 |
| **Your git state** | 当前分支、未提交更改、最近提交历史 |
| **Your CLAUDE.md** | 存储项目特定指令、惯例和上下文的markdown文件 |
| **Auto memory** | Claude工作时自动保存的学习（项目模式、你的偏好）|
| **Extensions** | MCP服务器、skills、subagents、Claude in Chrome浏览器交互 |

**因为Claude看到整个项目，它可以跨项目工作**

示例：说"fix the authentication bug"时，Claude会：
1. 搜索相关文件
2. 读多个文件理解上下文
3. 对它们进行协调编辑
4. 运行测试验证修复
5. 如请求则提交更改

**这不同于只看到当前文件的inline代码助手**

---

## 🖥️ Environments and Interfaces

### Execution Environments（执行环境）

| 环境 | 代码运行位置 | 使用场景 |
|------|-------------|----------|
| **Local（本地）** | 你的机器 | 默认。完全访问文件、工具和环境 |
| **Cloud（云端）** | Anthropic管理的VM | 卸载任务，处理本地没有的仓库 |
| **Remote Control（远程控制）** | 你的机器，从浏览器控制 | 使用Web UI同时保持一切本地 |

### Interfaces（接口）

可通过以下方式访问Claude Code：
- 终端
- Desktop app
- IDE扩展（VS Code, JetBrains）
- claude.ai/code
- Remote Control
- Slack
- CI/CD管道

**接口决定你如何看待和与Claude交互，但底层agentic loop相同**

---

## 📝 Session Management（会话管理）

### 数据保存位置
Claude Code工作时将对话本地保存为**plaintext JSONL文件**，位于 `~/.claude/projects/`：
- 每条消息、工具使用和结果都被写入
- 支持rewinding、resuming和forking sessions
- 在Claude做代码更改前，也会快照受影响文件以便需要时还原

### Sessions独立
- 每个新会话从新鲜上下文窗口开始
- 没有先前会话的对话历史
- Claude可用auto memory跨会话持久化学习
- 你可以在CLAUDE.md中添加自己的持久指令

### Work Across Branches（跨分支工作）
- 每个Claude Code对话是绑定当前目录的session
- Resume时只看到来自该目录的sessions
- Claude看到当前分支的文件
- 切换分支时Claude看到新分支的文件，但对话历史保持不变
- Claude记得讨论内容即使在切换后

**并行Claude sessions**: 用 `git worktrees`（为个别分支创建独立目录）

### Resume or Fork Sessions（恢复或分叉会话）

```bash
claude --continue              # 用相同session ID继续
claude --resume                # 从最近会话中选择
claude --continue --fork-session  # 分叉出新session ID
```

**Fork特点**:
- 保留到那点的对话历史
- 原始session保持不变
- Forked sessions不继承session-scoped permissions

⚠️ **同session多终端**: 如果在多个终端resume同一个session，两者写入同一session文件。消息交错。**对于并行工作用 `--fork-session`**

---

## 📊 The Context Window（上下文窗口）

Claude的上下文窗口持有：
- 对话历史
- 文件内容
- 命令输出
- CLAUDE.md
- Auto memory
- Loaded skills
- System instructions

**工作时上下文填充。Claude自动压缩，但对话早期的指令可能丢失**

**解决方案**: 将持久规则放在CLAUDE.md中，运行 `/context` 查看什么占用空间

### When Context Fills Up（上下文填满时）

Claude Code接近限制时自动管理上下文：

1. **首先清除较旧的工具输出**
2. **必要时总结对话**
3. **保留你的请求和关键代码片段**
4. **可能丢失早期详细指令**

**主动管理**:
- 将持久规则放在CLAUDE.md而非依赖对话历史
- 用"Compact Instructions"部分到CLAUDE.md或运行 `/compact <focus>`
- 运行 `/context` 查看空间使用
- MCP工具定义默认延迟按需加载（tool search），只有工具名占上下文直到Claude使用特定工具
- 运行 `/mcp` 检查每服务器成本

### Manage Context with Skills and Subagents

除了压缩外，用其他特性控制什么加载到上下文：

| 特性 | 上下文影响 |
|------|-----------|
| **Skills** | **按需加载**。Claude在session start看到skill描述，但完整内容只在skill被使用时加载。手动调用的skill设 `disable-model-invocation: true` 保持描述不出上下文直到你需要它们 |
| **Subagents** | **全新上下文**。完全独立于主对话。它们的工作不膨胀你的上下文。完成后返回摘要。这种隔离是subagent帮助长会话的原因 |

详见 [context costs](https://code.claude.com/docs/context-costs) 了解每个特性的成本，[reduce token usage](https://code.claude.com/docs/reduce-token-usage) 获取管理上下文的技巧

---

## 🛡️ 安全机制：Checkpoints和Permissions

### Undo Changes with Checkpoints（用Checkpoint撤销更改）

**每个文件编辑都是可逆的**

- Claude编辑任何文件前，**快照当前内容**
- 出错时：双击Esc回退到先前状态，或让Claudeundo
- Checkpoint是**session local的**，与git分离
- **只覆盖文件更改**
- ⚠️ 影响远程系统（数据库、API、部署）的动作无法checkpoint → 所以Claude在这些命令前会询问

### Control What Claude Can Do（控制Claude能做什么）

**Shift+Tab** 循环权限模式：

| 模式 | 行为 |
|------|------|
| **Default（默认）** | Claude在文件编辑和shell命令前询问 |
| **Auto-accept edits** | Claude编辑文件不询问，仍询问命令 |
| **Plan mode（计划模式）** | Claude只用只读工具，创建你可批准的计划 |
| **Auto mode（自动模式）** | Claude用后台安全评估所有action（目前研究预览）|

**Allowlist特定命令**: 在 `.claude/settings.json` 中设置，这样Claude不每次都问
- 适用于受信任命令如 `npm test` 或 `git status`
- Settings可从组织级策略范围缩小到个人偏好

详见 [Permissions](https://code.claude.com/docs/permissions)

---

## 💡 有效工作的关键技巧

### 1. Ask Claude Code for Help
Claude Code可以教你如何使用它。问"How do I set up hooks?"或"what's the best way to structure my CLAUDE.md?"

**内置引导命令**:
- `/init`: 引导创建项目的CLAUDE.md
- `/agents`: 帮助配置自定义subagents
- `/doctor`: 诊断安装常见问题

### 2. It's a Conversation（这是对话）
Claude Code是对话式的。不需要完美的prompts：
```
Fix the login bug
[Claude investigates, tries something]
That's not quite right. The issue is in the session handling.
[Claude adjusts approach]
```
**第一次尝试不对时不重新开始，而是迭代**

### 3. Interrupt and Steer（中断和引导）
你可以随时中断Claude。如果走错了方向，只需输入纠正按Enter。Claude会停止正在做的事情并根据输入调整方法

### 4. Be Specific Upfront（一开始就具体）
初始prompt越精确需要的修正越少。引用具体文件、提到约束、指向示例模式

**好的具体prompt**:
```
The checkout flow is broken for users with expired cards.
Check src/payments/ for the issue, especially token refresh.
Write a failing test first, then fix it.
```
**模糊prompt可行，但会花更多时间引导。上面这样的具体prompt通常首次成功**

### 5. Give Claude Something to Verify Against（给Claude验证依据）
Claude能自我检查时表现更好。包括测试案例、粘贴预期UI截图、或定义想要输出的样子

```
Implement validateEmail. Test cases: 'test@example.com' → true,
'invalid' → false, 'test@.com' → false. Run the tests after.
```

**视觉工作**: 粘贴设计截图并让Claude对比实现

### 6. Explore Before Implementing（实现前探索）
对于复杂问题，分离研究和编码。**用plan mode（Shift+Tab两次）先分析代码库**:
```
Read src/auth/ and understand how we handle sessions.
Then create a plan for adding OAuth support.
```
**审查计划、通过对话优化、然后让Claude实现。这种两阶段方法比直接跳跃到代码产生更好结果**

### 7. Delegate, Don't Dictate（委托而非指挥）
想象委托给一个有能力的同事。给上下文和方向，然后信任Claude搞清楚细节：

```
The checkout flow is broken for users with expired cards.
The relevant code is in src/payments/. Can you investigate and fix it?
```
**你不需要指定读哪些文件或运行哪些命令。Claude自己搞定**

---

## 🔗 相关资源

### 下一步学习路径

1. **Extend with Features** - 添加Skills、MCP连接和自定义命令
2. **Common Workflows** - 典型任务的分步指导
3. **Store Instructions and Memories** - CLAUDE.md和持久上下文

### 详细主题
- [Tools available to Claude](https://code.claude.com/docs/tools) - 完整工具列表
- [Context window details](https://code.claude.com/docs/explore-context-window) - 交互式演示
- [Checkpointing](https://code.claude.com/docs/checkpointing) - 回退机制详解
- [Permissions](https://code.claude.com/docs/permissions) - 权限配置
- [Sessions](https://code.claude.com/docs/sessions) - 会话管理细节
