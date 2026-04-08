# Anthropic Skills & Agents 官方技术文档集

**整理日期**: 2026年4月8日  
**来源**: 
- [Anthropic Engineering Blog](https://www.anthropic.com/engineering)
- [Claude Code Official Docs](https://code.claude.com/docs)
**文档语言**: 中文（翻译自英文原文）

---

## 📚 文档概览

本文档集包含Anthropic官方发布的关于 **AI代理（Agents）、技能（Skills）和Claude Code工具** 的最新完整技术文档，经过深度阅读、清洗和结构化处理。

### 核心主题

1. ✅ **代理架构与设计模式**
2. ✅ **上下文工程（Context Engineering）**
3. ✅ **评估方法（Evaluation）**
4. ✅ **长时间运行代理框架**
5. ✅ **Claude Code完整指南**
6. ✅ **Skills开发实战手册**
7. ✅ **Anthropic API传输协议**
8. ✅ **Claude Code传输架构**

---

## 📖 文档列表（共13篇）

---

### 🔬 第一部分：Agent理论与方法论（4篇）

#### 01. Building Effective AI Agents (构建有效的AI代理)
- **文件**: `01-building-effective-agents.md` | 大小: 6.5 KB
- **来源**: Anthropic Engineering Blog | **发布日期**: 2024年12月19日
- **作者**: Erik Schluntz, Barry Zhang
- **核心内容**:
  - 代理定义：Workflow vs Agent的本质区别
  - 五种工作流模式：Prompt Chaining / Routing / Parallelization / Orchestrator-Workers / Evaluator-Optimizer
  - 自主代理的三大核心原则：简洁性、透明度、ACI设计
  - 工具设计的ACI（Agent-Computer Interface）最佳实践 + Poka-yoke防错设计
  - 实际应用：客户支持代理、SWE-bench编码代理
- **适用读者**: 所有AI代理开发者，特别是刚开始构建代理的团队
- **关键要点**: 简洁性 > 复杂性；直接用LLM API而非过度抽象框架；花更多时间优化tools而非prompts

---

#### 02. Effective Context Engineering for AI Agents (AI代理的有效上下文工程)
- **文件**: `02-effective-context-engineering-for-ai-agents.md` | 大小: 9.57 KB
- **来源**: Anthropic Engineering Blog | **发布日期**: 2025年9月29日 ⭐ 最新
- **作者**: Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, Jeremy Hadfield 等
- **核心内容**:
  - **Context Engineering vs Prompt Engineering 的范式转变**
  - Context Rot现象：上下文增大→回忆能力下降（所有模型通用）
  - 注意力预算限制：n²成对关系的Transformer架构约束
  - System Prompts / Tools / Examples 的Goldilocks设计原则
  - "Just-in-time"动态上下文检索 vs 传统预检索
  - 长时间范围任务三大技术：
    1. **Compaction（压缩）**：对话摘要+保留关键决策
    2. **Structured Note-taking（结构化笔记）/Agentic Memory**：持久化外部记忆
    3. **Sub-agent Architectures（子代理架构）**：隔离上下文+蒸馏返回
- **适用读者**: 构建复杂多轮交互代理的开发者
- **关键要点**: 找到最大化期望结果可能性的最小高信号token集；上下文是有限资源需精心策划

---

#### 03. Demystifying Evals for AI Agents (揭开AI代理评估的神秘面纱)
- **文件**: `03-demystifying-evals-for-ai-agents.md` | 大小: 23.06 KB ⭐ 最全面
- **来源**: Anthropic Engineering Blog | **发布日期**: 2026年1月9日 ⭐ 最新
- **作者**: Mikaela Grace, Jeremy Hadfield, Rodrigo Olivares, Jiri De Jonghe 等
- **核心内容**:
  - **评估体系完整术语表**：Task / Trial / Grader / Transcript / Outcome / Harness / Suite
  - **三种评分器对比**：
    - Code-based（快速/客观/可重现但缺乏细微差别）
    - Model-based（灵活/可扩展但非确定性需校准）
    - Human（金标准但昂贵慢）
  - **四种Agent类型专门评估方法**：
    - Coding Agents → SWE-bench Verified / Terminal-Bench（确定性评分器为主）
    - Conversational Agents → τ-Bench / τ2-Bench（LLM rubric + 状态检查）
    - Research Agents → BrowseComp（groundedness + coverage checks）
    - Computer Use Agents → WebArena / OSWorld（环境状态验证）
  - **pass@k vs pass^k 指标**：一次成功概率 vs 一致性概率
  - **8步从零到一路线图**：
    1. 尽早开始（20-50个任务即可）
    2. 从手动测试开始
    3. 编写明确任务+参考方案
    4. 构建健壮harness+稳定环境
    5. 思考设计评分器（优先确定性→LLM→Human审慎使用）
    6. **检查transcript！**
    7. 监控eval饱和（100%无改进信号）
    8. 保持eval套件健康长期维护
  - **瑞士奶酪模型**：多方法整合（自动evals + 生产监控 + A/B + 用户反馈 + 人工研究）
- **适用读者**: 需要建立代理质量保证体系的团队、ML工程师
- **关键要点**: 尽早开始、从真实失败学习、**阅读transcript!**、像维护unit tests一样维护evals

---

#### 04. Effective Harnesses for Long-Running Agents (长时间运行代理的有效框架)
- **文件**: `04-effective-harnesses-for-long-running-agents.md` | 大小: 11.19 KB
- **来源**: Anthropic Engineering Blog | **发布日期**: 2025年11月26日
- **作者**: Justin Young
- **核心内容**:
  - **核心挑战**：跨会话记忆缺失（类比轮班工程师无交接）
  - Claude Agent SDK能力：compaction但仍不足
  - **两种失败模式**：一次性做完倾向 + 过早宣布胜利
  - **两重解决方案**：
    1. **Initializer Agent**（首次会话）：设置Feature List（200+功能JSON）+ init.sh + progress file + git commit
    2. **Coding Agent**（后续会话）：增量单功能开发 + git commit + progress update
  - **Feature List最佳实践**：JSON格式（比Markdown更难被误改）+ passes字段状态管理
  - **测试策略**：Puppeteer MCP浏览器自动化端到端验证（显著改善效果）
  - **四种失败模式及对应解决方案对照表**
- **适用读者**: 构建需要跨越多个上下文窗口的长时间任务代理的开发者
- **关键要点**: 模仿人类软件工程实践——增量开发、git提交、进度文档、先测后交付

---

### 💻 第二部分：Claude Code实战指南（3篇）

#### 05. Claude Code 官方文档 - 概览与核心概念
- **文件**: `05-claude-code-overview.md`
- **来源**: [code.claude.com/docs](https://code.claude.com/docs/en/overview) | 官方文档
- **核心内容**:
  - **产品定位**：Agentic coding tool - 读代码库、编辑文件、运行命令、集成开发工具
  - **安装方式**：Terminal CLI (curl/Homebrew/WinGet) / VS Code扩展 / Desktop App / Web版 / JetBrains插件
  - **八大核心功能**：
    1. 自动化重复任务（写测试、修复lint、解决合并冲突等）
    2. 自然语言构建功能和修复bug
    3. Git集成（提交、分支、PR）
    4. MCP工具连接（Google Drive/Jira/Slack等）
    5. 自定义配置（CLAUDE.md/Auto Memory/Custom Commands/Hooks）
    6. Agent团队和自定义Agent（并行+协调）
    7. CLI管道自动化（Unix哲学）
    8. 定时任务（Cloud/Desktop/loop）
  - **跨平台统一引擎**：所有界面共享底层agentic loop
- **适用读者**: 所有想了解或开始使用Claude Code的开发者

---

#### 06. Claude Code 最佳实践完整指南
- **文件**: `06-claude-code-best-practices.md` | 大小: ~15 KB
- **来源**: [code.claude.com/docs/best-practices](https://code.claude.com/docs/best-practices) | 官方文档
- **核心内容**:
  - **最高杠杆率做法**：给Claude验证方法（测试/截图/预期输出）— 这是最重要的单一件事
  - **探索→规划→编码四阶段工作流**（Plan Mode何时用/何时不用的判断标准）
  - **提示具体化策略**：限定范围/指向来源/参考现有模式/描述症状
  - **丰富内容提供方式**：@引用/粘贴图片/URL/管道数据/让Claude自取
  - **环境配置五大件**：
    1. CLAUDE.md编写（✅应包含/❌不应包含清单 + 导入语法）
    2. 权限模式配置（Auto mode/Allowlist/Sandboxing三选一）
    3. CLI工具使用（gh/aws/gcloud最上下文高效）
    4. MCP服务器连接
    5. Hooks设置（确定性行为vs建议性指令）
  - **扩展能力**：Skills/Subagents/Plugins（选择指导）
  - **会话管理黄金法则**：
    - 同一问题纠正超两次 → `/clear` 重来
    - Subagent调研避免上下文膨胀
    - Checkpoint回退（双击Esc）
    - Session resume/fork
  - **自动化扩展**：非交互模式(-p) / 多会话并行 / Fan-out批量操作 / Auto mode自主执行
  - **五种常见失败模式识别表** + 培养直觉建议
- **适用读者**: 希望最大化Claude Code效率的开发者
- **核心原则**: 上下文窗口是最重要资源；尽早且经常纠偏；培养个人直觉

---

#### 07. How Claude Code Works 工作原理深度解析
- **文件**: `07-how-claude-code-works.md`
- **来源**: [code.claude.com/docs/how-claude-code-works](https://code.claude.com/docs/how-claude-code-works) | 官方文档
- **核心内容**:
  - **Agentic Loop三大阶段**：Gather Context → Take Action → Verify Results（混合迭代）
  - **Models**：Sonnet(日常) vs Opus(复杂决策) 可随时切换
  - **Tools五类别**：File operations/Search/Execution/Web/Code intelligence
  - **工具使用实例**："Fix the failing tests"的6步骤循环演示
  - **访问权限范围**：项目文件/终端命令/git状态/CLAUDE.md/auto memory/extensions
  - **执行环境三种**：Local(默认)/Cloud(VM)/Remote Control(浏览器控制本地)
  - **Session管理机制**：
    - JSONL格式本地保存(~/.claude/projects/)
    - 跨分支工作(git worktrees)
    - Resume/Fork（⚠️同session多终端警告）
  - **Context Window详解**：
    - 包含内容：对话历史/文件内容/命令输出/CLAUDE.md/memory/skills/system instructions
    - 自动压缩策略：先清旧输出→必要时总结
    - 主动管理：/clear /compact /btw /skills按需加载/subagent隔离
  - **安全双重机制**：
    - Checkpoint（每编辑前快照，session local，跨会话持久）
    - Permissions四模式：Default/Auto-accept edits/Plan mode/Auto mode(research preview)
  - **七大工作技巧**：Ask for help/对话式/中断引导/具体 upfront/验证依据/探索优先/委托不指挥
- **适用读者**: 深入理解Claude Code内部机制的进阶用户

---

### 🛠️ 第三部分：Skills开发实战（1篇）

#### 08. Claude Code Skills 开发完整指南
- **文件**: `08-claude-code-skills-complete-guide.md` | 大小: ~18 KB ⭐ 实操性强
- **来源**: [code.claude.com/docs/extend-claude-with-skills](https://code.claude.com/docs/extend-claude-with-skills) | 官方文档
- **核心内容**:
  - **什么是Skills**：SKILL.md文件+frontmatter配置+可选支持文件目录
  - **5个Bundled Skills详解**：
    - `/batch` - 并行大规模代码库更改（研究→分解→计划→多agent实现→PR）
    - `/claude-api` - API参考材料加载（8种语言+SDK）
    - `/debug` - Debug logging启用+log分析
    - `/loop` - 定时重复prompt（轮询部署/照看PR）
    - `/simplify` - 代码审查+自动修复（3个并行review agent）
  - **Skill存储位置与作用域**：Enterprise > Personal > Project > Plugin（monorepo嵌套发现）
  - **Frontmatter完整字段参考表**（17个字段的详细说明和示例）
  - **四种字符串替换变量**：$ARGUMENTS / $ARGUMENTS[N] / $N / ${CLAUDE_SESSION_ID} / ${CLAUDE_SKILL_DIR}
  - **调用控制两种模式**：
    - `disable-model-invocation: true` - 仅手动（deploy/commit等有副作用操作）
    - `user-invocable: false` - 仅Claude调用（背景知识如legacy-system-context）
  - **高级模式三部曲**：
    1. **动态上下文注入**：`!<command>` shell预处理器（示例：PR摘要获取gh数据）
    2. **Subagent执行**：`context: fork` + `agent:` 字段（示例：deep-research用Explore agent）
    3. **可视化输出**：捆绑脚本生成交互HTML（示例：codebase-map.py树可视化）
  - **参数传递**：位置访问($0/$1) + 无$ARGUMENTS时的fallback行为
  - **权限限制三层**：完全禁用Skill tool / 允许禁止特定skill(Skill(name)语法) / frontmatter隐藏
  - **分享分发三级**：Project(version control) / Plugin(packaging) / Managed(org-wide deployment)
  - **故障排除**：未触发/触发过频/description截断的解决方案
- **适用读者**: 希望扩展Claude Code能力的技术负责人、团队工具链工程师
- **标准遵循**: [Agent Skills open standard](https://github.com/anthropics/agent-skills)（跨AI工具通用）+ Claude Code扩展特性

---

### 🤖 第四部分：多代理协作实战（3篇）

#### 09. Subagents 完整指南（子代理深度解析）
- **文件**: `09-subagents-complete-guide.md` | 大小: ~20 KB
- **来源**: [code.claude.com/docs/subagents](https://code.claude.com/docs/subagents) | 官方文档
- **核心内容**:
  - Subagent 定义：独立上下文窗口中的专业化 AI 助手（vs Agent Teams 的本质区别）
  - 3个内置 Subagent：Explore(Haiku只读) / Plan(规划模式) / General-purpose(全工具)
  - 5层存储优先级：Managed > CLI > `.claude/agents/` > `~/.claude/agents/` > Plugin
  - 17个 Frontmatter 字段完整参考表
  - 工具控制三模式：白名单(tools) / 黑名单(disallowedTools) / Agent()生成限制
  - 权限模式6种：default/acceptEdits/auto/dontAsk/bypassPermissions/plan
  - MCP 服务器作用域：内联定义(仅subagent可见) + 引用共享
  - 持久化 Memory 三范围：user/project/local + 自动注入 MEMORY.md 前200行
  - Hooks 条件验证：PreToolUse hook 阻止危险操作（如 SQL 只读查询）
  - 调用方式三级：自然语言 → @-mention 保证执行 → --agent 全session
  - 前台 vs 后台运行：阻塞式 vs 并发(Ctrl+B切换)
  - Auto-Compaction：默认95%容量触发压缩
  - 4个完整示例：Code Reviewer / Debugger / Data Scientist / DB Query Validator
- **适用读者**: 想要创建专业化、可复用子代理的开发者

---

#### 10. Agent Teams 完整指南（多实例团队协调）
- **文件**: `10-agent-teams-complete-guide.md` | 大小: ~18 KB
- **来源**: [code.claude.com/docs/agent-teams](https://code.claude.com/docs/agent-teams) | 官方文档
- **状态**: 🧪 Experimental（实验性功能，默认禁用）
- **核心内容**:
  - 与 Subagents 本质区别：Teammate 可直接互相通信（Subagent 只报告回主代理）
  - 启用方式：CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1（要求 v2.1.32+）
  - 两种显示模式：In-process(任意终端) / Split panes(tmux/iTerm2)
  - 架构四大组件：Team Lead + Teammates(独立Claude实例) + Task List(共享) + Mailbox(消息系统)
  - Plan Approval 工作流：Teammate plan → Lead 审批(批准/拒绝+反馈) → 实施
  - 任务管理：三状态(pending/in_progress/completed) + 文件锁防竞态 + 依赖自动解除阻塞
  - 通信机制：message(点对点) / broadcast(全员，慎用成本线性增长)
  - 两大完整使用案例：并行代码审查(安全/性能/测试三reviewer) / 竞争假设调试(5teammate互相反驳)
  - 最佳实践7条 + 当前限制9条 + 故障排除表
- **适用读者**: 需要多个 AI agent 协同完成复杂任务的进阶用户

---

#### 11. Plugins 完整开发指南（打包分发扩展）
- **文件**: `11-plugins-complete-guide.md` | 大小: ~14 KB
- **来源**: [code.claude.com/docs/plugins](https://code.claude.com/docs/plugins) | 官方文档
- **核心内容**:
  - Plugin vs 独立配置(.claude/)完整对比表
  - 完整目录结构：.claude-plugin/plugin.json + commands/agents/skills/hooks/.mcp.json/.lsp.json/bin/settings.json
  - Quickstart 5步：创建目录 → manifest → skill(SKILL.md) → --plugin-dir 测试 → $ARGUMENTS 参数化
  - 组件添加全解：Skills/Agents/Hooks/MCP/LSP/Settings/Bin
  - 本地开发：--plugin-dir + /reload-plugins 热重载 + 多 plugin 同时加载
  - 从 .claude/ 迁移4步骤 + 变化对照表
  - 分发方式：团队分享 / 私有 Marketplace / 官方 Marketplace 提交
- **适用读者**: 想要打包和分发 Claude Code 扩展的团队工具链工程师
- **关键要点**: ⚠️ 只有 plugin.json 放 .claude-plugin/！其他目录必须在 root level

---

---

### 📡 第五部分：传输协议与架构（2篇）

#### 12. Anthropic API 传输协议（Messages API / SSE流式传输）
- **文件**: `12-anthropic-api-transportation-protocol.md` | 大小: ~38 KB
- **来源**: 
  - [Anthropic API Overview](https://platform.claude.com/docs/en/api/overview)
  - [Build with Claude: Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming)
  - [API Errors](https://platform.claude.com/docs/en/api/errors)
- **核心内容**:
  - **HTTP 规范**: REST 端点 `/v1/messages`，请求头（`x-api-key`, `anthropic-version`, `content-type`），认证方式
  - **多平台认证**: API Key 直连 / AWS Bedrock / Google Vertex AI / Azure 三方平台集成
  - **请求体完整 schema**: `model` / `messages` (role+content) / `max_tokens`(必填) / `system` / `temperature` / `top_p` / `top_k` / `stop_sequences` / `stream` / `tools` / `tool_choice` / `thinking`
  - **响应格式**: 完整 JSON 响应 vs SSE 流式响应对比
  - **SSE 流式协议**: 
    - 事件生命周期：`message_start` → `content_block_start/delta/stop` → `message_delta/stop`
    - 8种内容块类型：text / image / tool_use / tool_result / thinking / redacted_thinking / signature / input_json
    - 4种 delta 事件：text_delta / input_json_delta / thinking_delta / signature_delta
    - Mermaid 流程图可视化完整生命周期
  - **错误处理体系**:
    - 认证错误 (401) / 权限错误 (403) / 未找到 (404) / 请求过大 (413)
    - 速率限制 (429)：Token Bucket 算法 + Retry-After 头
    - 服务端错误 (5xx) + API 错误 (521-599)
    - 错误响应 JSON 结构 (`type` + `error` 对象)
  - **SDK 集成**: Python / TypeScript SDK 使用示例
- **适用读者**: 需要直接调用 Anthropic API 的开发者、理解底层传输机制的系统工程师
- **关键要点**: `max_tokens` 是唯一必填字段；SSE 流式通过 `stream:true` 启用；速率限制遵循 Token Bucket 模型

---

#### 13. Claude Code 传输架构深度解析
- **文件**: `13-claude-code-transportation-architecture.md` | 大小: ~52 KB ⭐ 最深入
- **来源**:
  - [How Claude Code Works](https://code.claude.com/docs/how-claude-code-works)
  - [Claude Code Overview](https://code.claude.com/docs/overview)
  - [Model Context Protocol Specification](https://modelcontextprotocol.io/specification)
- **核心内容**:
  - **Agentic Loop 3阶段模型**: Gather Context → Take Action → Verify Results（Mermaid流程图）
  - **统一核心引擎**: Terminal CLI / VS Code / Desktop / Web / JetBrains 共享同一 agentic loop
  - **API 通信层**: 
    - Messages API 调用链路（用户输入 → System Prompt 组装 → API 请求 → SSE 解析 → 输出渲染）
    - 多模型策略：Sonnet(日常) / Opus(复杂决策) 自动切换逻辑
  - **MCP 传输层**:
    - stdio 模式：子进程 stdin/stdout JSON-RPC 通信
    - Streamable HTTP 模式：远程服务器 + SSE 长连接
    - Session 管理：initialize → initialized → 生命周期通知
  - **工具调用协议**:
    - 6大类别：File / Search / Execution / Web / Code Intelligence / Orchestration
    - 6步生命周期：Tool selection → Permission check → Execution → Result parsing → Context update → Next action
    - 4种权限模式：Default / Auto-edit / Plan / Auto
  - **上下文窗口管理**:
    - 6个组成组件：System Prompt / CLAUDE.md / Conversation History / Dynamic Content / Auto Memory / Skills
    - 压缩策略：自动清理旧输出 → 必要时摘要压缩 → Compaction
    - Token 预算分配与优化技巧
  - **多环境部署对比**: Local / Cloud VM / Remote Control 的网络拓扑、数据流向、安全性差异（Mermaid架构图）
  - **安全机制**: Checkpoint 快照系统 + Permissions 四层权限控制 + Sandbox 隔离
  - **扩展系统**: Skills / Subagents / Plugins / Hooks 与传输层的集成点
  - **24个 Mermaid 图表**: 覆盖架构图、时序图、流程图、甘特图等多种可视化形式
- **适用读者**: 需要深入理解 Claude Code 内部通信机制的进阶开发者、构建类似系统的架构师
- **关键要点**: 所有界面共享同一核心引擎；MCP 双传输模式(stdio/HTTP)；上下文窗口是最稀缺资源需精心管理

---

## 🔗 相关资源链接

### 官方文档源
- [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

### Anthropic平台
- [Claude Developer Platform](https://platform.claude.com)
- [Claude Documentation](https://docs.anthropic.com)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io)

### 传输协议与架构源
- [Anthropic API Overview](https://platform.claude.com/docs/en/api/overview)
- [Build with Claude: Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming)

---

## 🎯 推荐阅读路径

### 入门级（刚接触AI代理）
1. **01-Building-Effective-AI-Agents** → 了解基本概念和模式
2. **04-Effective-Harnesses-for-Long-Running-Agents** → 学习实际实现框架

### 进阶级（正在构建代理）
1. **02-Effective-Context-Engineering** → 掌握上下文管理核心技术
2. **03-Demystifying-Evals** → 建立评估体系

### 高高级（优化生产系统）
按需参考所有文档，特别关注：
- **07-How-It-Works**: 深入理解内部机制
- **08-Skills-Complete-Guide**: 开发自定义技能（最强实操）
- **10-Agent-Teams**: 多实例并行协调（实验性功能）
- **11-Plugins**: 打包分发扩展给团队
- **12-API-Transportation-Protocol**: Anthropic Messages API 与 SSE 流式传输底层协议
- **13-Claude-Code-Transportation-Architecture**: Claude Code 完整传输架构与 MCP 协议集成
- 评估路线图（文档03的第7-8步）
- 多代理架构讨论（文档02和04的未来工作部分）

### 底层协议与架构（系统工程师/架构师）
1. **12-Anthropic-API-Transportation-Protocol** → 理解 HTTP API / SSE 流式 / 认证 / 错误处理 / 速率限制
2. **13-Claude-Code-Transportation-Architecture** → 深入 Agentic Loop / MCP 双模式 / 工具调用协议 / 上下文管理 / 安全机制

---

## 💡 核心概念速查表

| 概念 | 定义 | 关键文献 |
|------|------|----------|
| **Agent** | LLM在循环中自主使用工具的系统 | 文档01 |
| **Workflow** | 通过预定义代码路径编排LLM和工具 | 文档01 |
| **Subagent** | 独立上下文窗口中的专业化AI助手，单会话内运行 | 文档09 |
| **Agent Team** | 多个独立Claude Code实例协调工作，可互相通信 | 文档10 |
| **Plugin** | 打包分发skills/agents/hooks/MCP的可安装扩展单元 | 文档11 |
| **Context Engineering** | 管理推理期间最优token集合的策略 | 文档02 |
| **Context Rot** | 上下文窗口增大时模型回忆能力下降 | 文档02 |
| **Compaction** | 压缩对话内容以延续长期任务 | 文档02, 04, 09 |
| **Agentic Memory** | 结构化笔记持久化到上下文外部 | 文档02, 09 |
| **Skill** | SKILL.md文件+frontmatter配置的agent能力扩展 | 文档08, 11 |
| **Eval** | AI系统的自动化测试 | 文档03 |
| **Grader** | 评分代理性能的逻辑 | 文档03 |
| **pass@k** | k次尝试中至少一次成功的概率 | 文档03 |
| **pass^k** | 所有k次试验都成功的概率 | 文档03 |
| **Harness** | 使模型能够充当代理的系统框架 | 文档03, 04 |
| **ACI** | Agent-Computer Interface（代理-计算机接口）| 文档01 |
| **MCP** | Model Context Protocol，工具集成标准（stdio/Streamable HTTP 双模式）| 文档05, 09, 11, 13 |
| **SSE** | Server-Sent Events，服务端推送流式传输协议 | 文档12, 13 |
| **Agentic Loop** | Claude Code 的 Gather→Act→Verify 三阶段循环 | 文档07, 13 |
| **Hook** | 工具使用前后的生命周期事件处理器 | 文档06, 09, 11 |

---

## 📊 技术栈关联

### 与Claude平台的集成
- **Claude Code**: 使用文档04中的harness模式，文档13详解其传输架构
- **MCP (Model Context Protocol)**: 文档01和02中提到的工具集成标准，文档13详解双传输模式
- **Messages API**: 文档12完整覆盖 HTTP 规范 / SSE 流式 / 认证 / 错误处理
- **Agent SDK**: 文档04的核心框架
- **Memory Tool**: 文档02提到的Sonnet 4.5新功能

### 评估基准工具
- SWE-bench Verified (编码)
- Terminal-Bench (终端操作)
- BrowseComp (研究)
- WebArena / OSWorld (计算机使用)
- τ-Bench / τ2-Bench (对话)

---

## ⚠️ 使用说明

### 文档格式
- Markdown格式，支持所有主流Markdown编辑器
- 包含代码示例（YAML配置、JSON结构）
- 表格对比清晰展示不同方案优劣

### 更新计划
本文档基于2025年11月至2026年1月的最新文章。建议关注Anthropic Engineering博客以获取后续更新。

### 反馈与贡献
如发现翻译或理解错误，欢迎提出修正建议。

---

**最后更新**: 2026年4月8日
**版本**: v1.2（新增 Anthropic API 传输协议 + Claude Code 传输架构两篇底层协议文档）
