# Claude Code 最佳实践完整指南

**来源**: [code.claude.com/docs/best-practices](https://code.claude.com/docs/en/best-practices)  
**整理日期**: 2026年4月8日  
**适用对象**: 希望最大化Claude Code效率的开发者

---

## ⚠️ 核心约束

**大多数最佳实践基于一个约束：Claude的上下文窗口填充很快，性能随着填充而下降**

上下文窗口包含：
- 整个对话历史
- Claude读取的每个文件
- 每个命令输出

单个调试会话或代码库探索可能生成并消耗**数万个token**。当上下文窗口接近满时：
- Claude可能开始"忘记"早期指令
- 犯更多错误

**结论：上下文窗口是最需要管理的资源**

---

## 🎯 最高杠杆率做法：给Claude验证方法

**这是你能做的最重要的一件事！**

包含测试、截图或预期输出让Claude能自我检查。没有明确成功标准，它可能产出看起来正确但实际上不起作用的东西——你成了唯一的反馈循环。

### 对比示例

| 场景 | ❌ Before | ✅ After |
|------|----------|----------|
| 提供验证标准 | "implement a function that validates email addresses" | "write a validateEmail function. example test cases: test@example.com is true, invalid is false, test@.com is false. run the tests after implementing" |
| UI变更验证 | "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them" |
| 解决根本原因而非症状 | "the build is failing" | "the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error" |

**验证方式**:
- 测试套件 ✅
- Linter ✅
- Bash命令检查输出 ✅
- **Chrome extension截图对比** ✅

> **投资建议**: 让你的验证坚如磐石！

---

## 🗺️ 探索→规划→编码 三阶段工作流

将研究与实施分离，避免解决错误的问题。

### 推荐四阶段流程

#### Phase 1: Explore（探索）- Plan Mode
```
claude (Plan Mode)
> read /src/auth and understand how we handle sessions and login.
> also look at how we manage environment variables for secrets.
```
- Claude读取文件、回答问题，但不做修改
- `Ctrl+G` 打开计划在文本编辑器中直接编辑

#### Phase 2: Plan（规划）
```
claude (Plan Mode)
> I want to add Google OAuth. What files need to change?
> What's the session flow? Create a plan.
```

#### Phase 3: Implement（实现）- Normal Mode
```
claude (Normal Mode)
> implement the OAuth flow from your plan. write tests for the
> callback handler, run the test suite and fix any failures.
```
切换回Normal Mode让Claude编码，对照计划验证

#### Phase 4: Commit（提交）
```
claude (Normal Mode)
> commit with a descriptive message and open a PR
```

### 何时跳过Plan Mode？

✅ **适合直接做的情况**:
- 任务范围清晰且修复小（如修typo、加log行、重命名变量）

❌ **需要Plan Mode的情况**:
- 不确定方法
- 更改多个文件
- 不熟悉被修改的代码

**判断标准**: 如果能用一句话描述diff，就跳过plan

---

## 📍 提供具体的上下文

指令越精确，需要的修正越少。Claude可以推断意图，但不能读心术。

### 具体策略

| 策略 | ❌ Vague | ✅ Specific |
|------|----------|-------------|
| **限定任务范围** | "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| **指向来源** | "why does ExecutionFactory have such a weird api?" | "look through ExecutionFactory's git history and summarize how its api came to be" |
| **参考现有模式** | "add a calendar widget" | "look at how existing widgets are implemented on the home page to understand the patterns. HotDogWidget.php is a good example. follow the pattern to implement a new calendar widget..." |
| **描述症状** | "fix the login bug" | "users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it" |

**模糊提示的使用场景**: 当你在探索且负担得起纠正成本时。像"what would you improve in this file?"这样的提示可能会暴露你没想到要问的事情

---

## 📎 提供丰富内容

多种方式提供丰富数据给Claude：

1. **@引用文件**: `@filename` - Claude在响应前读取文件
2. **粘贴图片**: 直接复制/粘贴或拖拽图片到提示中
3. **提供URL**: 给出文档和API参考链接（用 `/permissions` 白名单常用域）
4. **管道数据**: `cat error.log | claude` 直接发送文件内容
5. **让Claude自行获取**: 让Claude用Bash命令、MCP工具或读文件拉取所需上下文

---

## ⚙️ 环境配置

几个设置步骤显著提高所有会话的效果。

### 1. 编写有效的CLAUDE.md

运行 `/init` 基于当前项目结构生成初始CLAUDE.md文件，然后随时间优化。

**什么是CLAUDE.md？**
特殊文件，每次对话开始时Claude读取。包含Bash命令、代码风格和工作流规则。提供Claude无法从代码本身推断的持久上下文。

**格式要求**: 无必需格式，但保持简短和人类可读

**示例**:
```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

#### CLAUDE.md 内容清单

| ✅ 应该包含 | ❌ 不应包含 |
|-------------|-----------|
| Claude猜不到的Bash命令 | Claude通过读代码能弄清楚的东西 |
| 与默认不同的代码风格规则 | Claude已知的标准语言约定 |
| 测试指令和首选测试运行器 | 详细的API文档（改为链接文档） |
| Repository etiquette（分支命名、PR惯例） | 经常变化的信息 |
| 特定项目的架构决策 | 长解释或教程 |
| 开发者环境怪癖（必需的环境变量） | 逐文件的代码库描述 |
| 常见的陷阱或不明显的行为 | 显而易见的做法如"写干净的代码" |

**重要原则**:
- 每次加载，只包含广泛应用的内容
- 对于偶尔相关的领域知识或工作流，改用**skills**
- **保持简洁**: 问自己"移除这行会导致Claude犯错吗？"如果不能，删除
- **膨胀的CLAUDE.md会导致Claude忽略你的实际指令！**

**调试技巧**:
- 如果Claude尽管有规则仍一直做你不想要的事 → 文件可能太长了，规则被淹没了
- 如果Claude问你CLAUDE.md中已回答的问题 → 措辞可能模糊
- **像对待代码一样对待CLAUDE.md**: 出问题时审查它，定期修剪，通过观察行为变化测试更改
- 可添加强调（如"IMPORTANT"或"YOU MUST"）提高遵守度
- **检入git**让团队贡献。文件价值随时间复合增长

**导入其他文件**:
```markdown
See @README.md for project overview and @package.json for available npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```

**放置位置**:
| 位置 | 作用域 |
|------|--------|
| `~/.claude/CLAUDE.md` | 所有Claude会话 |
| `./CLAUDE.md` | 项目根目录，检入git分享给团队 |
| `./CLAUDE.local.md` | 个人项目特定笔记，加入.gitignore |
| **父目录** | monorepos有用（root/CLAUDE.md 和 root/foo/CLAUDE.md 自动引入）|
| **子目录** | Claude在使用那些目录中的文件时按需引入 |

### 2. 配置权限模式

默认情况下，Claude Code对可能修改系统的动作请求权限（文件写入、Bash命令、MCP工具等）。安全但繁琐。

#### 三种减少中断的方式

| 方式 | 特点 | 适用场景 |
|------|------|----------|
| **Auto mode** | 分类器模型审查命令，只阻止看起来有风险的（权限升级、未知基础设施、恶意内容驱动） | 信任任务总体方向但不想每步都点击 |
| **Permission allowlists** | 许可你知道是安全的特定工具（如 `npm run lint`, `git commit`） | 明确知道哪些命令安全 |
| **Sandboxing** | 启用OS级隔离，限制文件系统和网络访问 | 在定义边界内让Claude更自由工作 |

快捷键: `Shift+Tab` 循环权限模式

### 3. 使用CLI工具
告诉Claude Code使用CLI工具（gh, aws, gcloud, sentry-cli等）与外部服务交互。

**为什么？CLI工具是与外部服务交互最上下文高效的方式**

示例：
- GitHub → 安装 `gh` CLI（Claude会用它创建issue、打开PR、读评论）
- 学习新CLI: `"Use 'foo-cli-tool --help' to learn about foo tool, then use it to solve A, B, C"`

### 4. 连接MCP服务器
```bash
claude mcp add
```
连接外部工具如Notion、Figma、数据库。可以让Claude从issue tracker实现功能、查询数据库、分析监控数据、集成Figma设计。

### 5. 设置Hooks
用于必须每次发生的零例外动作。与CLAUDE.md指令不同（建议性的），hooks是确定性的，保证执行。

示例：
- "Write a hook that runs eslint after every file edit"
- "Write a hook that blocks writes to the migrations folder"

编辑 `.claude/settings.json` 手动配置，运行 `/hooks` 浏览配置

### 6. 创建Skills
在 `.claude/skills/` 创建SKILL.md文件，给Claude领域知识和可重用工作流。

#### Skill类型1: 领域知识
```markdown
---
name: api-conventions
description: REST API design conventions for our services
---
# API Conventions
- Use kebab-case for URL paths
- Use camelCase for JSON properties
- Always include pagination for list endpoints
- Version APIs in the URL path (/v1/, /v2/)
```
Claude在相关时自动应用，或用 `/skill-name` 直接调用

#### Skill类型2: 可调用工作流
```markdown
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---
Analyze and fix the GitHub issue: $ARGUMENTS.

1. Use `gh issue view` to get the issue details
2. Understand the problem described in the issue
3. Search the codebase for relevant files
4. Implement the necessary changes to fix the issue
5. Write and run tests to verify the fix
6. Ensure code passes linting and type checking
7. Create a descriptive commit message
8. Push and create a PR
```
运行 `/fix-issue 1234` 调用

**`disable-model-invocation: true`**: 用于有副作用的你想手动触发的工作流

### 7. 创建自定义Subagents
在 `.claude/agents/` 中定义专业助手，Claude可以委托隔离任务。

**特点**: Subagent在自己的上下文中运行，有自己允许的工具集。适用于读很多文件或需要专注而不混乱主对话的任务。

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---
You are a senior security engineer. Review code for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure data handling

Provide specific line references and suggested fixes.
```
**显式告诉Claude使用subagent**: "Use a subagent to review this code for security issues."

### 8. 安装Plugins
运行 `/plugin` 浏览marketplace。插件打包skills、hooks、subagents和MCP服务器为单一可安装单元。

**推荐**: 如果使用类型语言，安装代码智能插件以获得精确符号导航和编辑后自动错误检测

---

## 💬 有效沟通

### 询问代码库问题
向Claude提问你会问高级工程师的问题（入职学习有效）：
- How does logging work?
- How do I make a new API endpoint?
- What does `async move { ... }` do on line 134 of foo.rs?
- What edge cases does CustomerOnboardingFlowImpl handle?
- Why does this code call foo() instead of bar() on line 333?

**无需特殊提示技巧：直接问问题即可**

### 让Claude采访你
对于较大功能，先用最小prompt让Claude使用AskUserQuestion工具采访你。

```
I want to build [brief description]. Interview me in detail using the AskUserQuestion tool.
Ask about technical implementation, UI/UX, edge cases, concerns, and tradeoffs.
Don't ask obvious questions, dig into the hard parts I might not have considered.
Keep interviewing until we've covered everything, then write a complete spec to SPEC.md.
```
完成后，开始新会话执行。新会话有干净的上下文完全专注于实现，你有书面规范可参考

---

## 🎮 会话管理

### 尽早且经常纠偏
**最佳结果来自紧密反馈循环**

- **Esc**: 中途停止Claude（保留上下文，可重定向）
- **Esc + Esc** 或 `/rewind`: 打开回退菜单恢复先前对话和代码状态，或从选定消息总结
- **"Undo that"**: 让Claude撤销更改
- **/clear**: 无关任务间重置上下文（长会话无关上下文可能降低性能）

**黄金法则**: 如果在同一会话中同一问题纠正超过**两次**，上下文已被失败方法污染。运行 `/clear` 并用结合所学内容的更具体prompt重新开始。

**干净会话 + 更好prompt 几乎总是优于 有积累纠正的长会话**

### 积极管理上下文

| 技术 | 说明 |
|------|------|
| **频繁使用/clear** | 无关任务间重置上下文窗口 |
| **自动压缩** | 接近上下文限制时触发，保留重要代码和决策同时释放空间 |
| **`/compact <instructions>`** | 更多控制，如 `/compact Focus on the API changes` |
| **选择性压缩** | `Esc + Esc` + 选择消息检查点 + "Summarize from here" - 只压缩选中点之后的消息 |
| **自定义压缩指令** | 在CLAUDE.md中添加"When compacting, always preserve..."确保关键上下文存活 |
| **`/btw`** | 不需要留在上下文中的快速问题，答案出现在可关闭覆盖层中，从不进入对话历史 |

### 使用Subagent进行调研
委托调研用"use subagents to investigate X"。它们在单独上下文中探索，保持主对话干净用于实现。

**优势**: Subagent在独立上下文窗口运行并报告摘要，不混乱主对话

```
Use subagents to investigate how our authentication system handles token refresh, 
and whether we have any existing OAuth utilities I should reuse.
```

也可用于实现后验证：
```
use a subagent to review this code for edge cases
```

### Checkpoint回退
**每个Claude动作创建checkpoint**。你可以恢复对话、代码或两者到任何先前checkpoint。

**特点**:
- 双击Escape或 `/rewind` 打开回退菜单
- 可只恢复对话、只恢复代码、或两者都恢复、或从选定点总结
- **跨会话持久**：关闭终端后仍可回退
- **只跟踪Claude做的更改**，非外部进程（不是git替代品）

**策略**: 不要仔细规划每一步，告诉Claude尝试有风险的东西。如果不行，回退尝试不同方法

### 恢复对话
```bash
claude --continue    # 恢复最近的对话
claude --resume      # 从最近对话中选择
```
用 `/rename` 给会话描述性名称（如 "oauth-migration", "debugging-memory-leak"）

**像对待分支一样对待会话**: 不同工作流可以有分离的持久上下文

---

## 🚀 自动化和扩展

一旦能有效使用一个Claude，用并行会话、非交互模式和fan-out模式倍增产出

### 非交互模式
```bash
# 一次性查询
claude -p "Explain what this project does"

# 结构化输出用于脚本
claude -p "List all API endpoints" --output-format json

# 流式JSON用于实时处理
claude -p "Analyze this log file" --output-format stream-json
```
**用途**: CI管道、pre-commit hooks、任何自动化工作流

### 运行多个Claude会话

三种主要并行方式：

| 方式 | 特点 |
|------|------|
| **Desktop app** | 可视化管理多个本地会话，每个有独立worktree |
| **Web版** | 在Anthropic管理的云基础设施上运行（隔离VM）|
| **Agent teams** | 多会话自动协调（共享任务、消息传递、团队领导）|

**质量导向工作流 - Writer/Reviewer模式**:

| Session A (Writer) | Session B (Reviewer) |
|--------------------|---------------------|
| Implement a rate limiter for our API endpoints | Review the rate limiter implementation in @src/middleware/rateLimiter.ts. Look for edge cases, race conditions, and consistency with our existing middleware patterns. |
| Here's the review feedback: [Session B output]. Address these issues. | |

类似可用于测试：一个Claude写测试，另一个写代码通过测试

### Fan-out跨文件操作
循环任务列表对每个调用 `claude -p`。用 `--allowedTools` 限制批量操作的权限。

**步骤**:

1. **生成任务列表**: 让Claude列出需要迁移的所有文件（如列出2000个Python文件）

2. **编写脚本循环**:
```bash
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

3. **先测试几个然后大规模运行**: 基于前2-3个出错情况优化prompt，然后在全集合上运行

**`--allowedTools`标志**限制Claude能做什么，无人看管时很重要

也可集成到现有数据处理管道：
```bash
claude -p "<your prompt>" --output-format json | your_command
```

### Autonomous Auto Mode（自主自动模式）
```bash
claude --permission-mode auto -p "fix all lint errors"
```
分类器模型在运行前审查命令，阻止权限升级、未知基础设施和恶意内容驱动的行动，同时让日常工作无提示进行。

**非交互模式(-p)注意事项**: 如果分类器反复阻止动作会中止，因为没有用户可回退

---

## ⚠️ 常见失败模式识别

| 失败模式 | 症状 | 解决方案 |
|----------|------|----------|
| **Kitchen sink会话** | 开始一个任务，然后问无关的东西，再回去 | 无关任务间 `/clear` |
| **反复纠正** | 同一问题纠正超过两次 | `/clear` 并写更好的初始prompt（结合所学）|
| **过度指定的CLAUDE.md** | 文件太长，Claude忽略一半 | 无情修剪。删掉Claude已经正确做的事或转为hook |
| **信任-验证差距** | 产出看似合理但没处理边缘情况的实现 | **始终提供验证**（测试、脚本、截图）。不能验证就不发货 |
| **无限探索** | 让Claude"investigate"但未限定范围，读了数百文件填满上下文 | 窄范围调研或使用subagents避免消耗主上下文 |

---

## 🧠 培养直觉

**本指南的模式不是金科玉律**。它们是在一般情况下效果好的起点，但对每种情况不一定最优。

**有时你应该**:
- **让上下文累积**: 因为深入复杂问题且历史有价值
- **跳过规划**: 因为任务是探索性的
- **用模糊提示**: 正确！因为想看Claude如何解释问题再约束它

**关注什么有效**:
- 当Claude产出优秀输出时 → 注意你做了什么：prompt结构、提供的上下文、使用的模式
- 当Claude挣扎时 → 问为什么：上下文太嘈杂？prompt太模糊？任务太大一次完成？

**时间培养的直觉没有指南能捕捉**。你会知道何时具体何时开放式，何时规划何时探索，何时清除上下文何时让它累积。

---

## 📚 相关资源

- [How Claude Code works](https://code.claude.com/docs/how-claude-code-works): agentic loop、tools和上下文管理
- [Extend Claude Code](https://code.claude.com/docs/extend): skills、hooks、MCP、subagents和plugins
- [Common workflows](https://code.claude.com/docs/common-workflows): 调试、测试、PR等的分步配方
- [CLAUDE.md](https://code.claude.com/docs/store-instructions-and-memories): 存储项目惯例和持久上下文
