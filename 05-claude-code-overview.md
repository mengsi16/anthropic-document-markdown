# Claude Code 官方文档 - 概览与核心概念

**来源**: [code.claude.com/docs](https://code.claude.com/docs/en/overview)  
**整理日期**: 2026年4月8日

---

## 🎯 什么是 Claude Code？

Claude Code 是一个 **agentic coding tool**（代理编码工具），能够：
- 读取你的代码库
- 编辑文件
- 运行命令
- 与开发工具集成

**可用环境**: 终端、IDE、桌面应用、浏览器

---

## 📦 安装方式

### Terminal CLI（推荐）

#### macOS, Linux, WSL:
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

#### Windows PowerShell:
```powershell
irm https://claude.ai/install.ps1 | iex
```

#### Windows CMD:
```bash
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```
⚠️ Windows需要先安装 Git for Windows

#### Homebrew:
```bash
brew install --cask claude-code
```
*注意：Homebrew不会自动更新，需定期运行 `brew upgrade claude-code`*

#### WinGet:
```powershell
winget install Anthropic.ClaudeCode
```
*注意：WinGet也不会自动更新*

### 启动方式
```bash
cd your-project
claude
```
首次使用时会提示登录。

---

## 💻 其他环境

### VS Code 扩展
- 内联diffs、@提及、计划审查、对话历史
- 搜索"Claude Code"扩展安装
- 快捷键: `Cmd+Shift+X` (Mac) / `Ctrl+Shift+X` (Windows/Linux)

### Desktop App
- 独立应用，可在IDE或终端外运行
- 视觉diff审查、并行多会话、定时任务
- 支持: macOS (Intel/Apple Silicon), Windows x64/ARM64

### Web 版本
- 浏览器中运行，无需本地设置
- 可启动长任务后离开，稍后查看结果
- 访问: [claude.ai/code](https://claude.ai/code)

### JetBrains IDEs
- 支持 IntelliJ IDEA, PyCharm, WebStorm 等
- 从 JetBrains Marketplace 安装插件

---

## ✨ 核心功能

### 1. 自动化重复性任务
```bash
# 示例：自动编写测试并修复
claude "write tests for the auth module, run them, and fix any failures"
```
处理：写测试、修复lint错误、解决合并冲突、更新依赖、写发布说明

### 2. 构建功能和修复bug
- 用自然语言描述需求 → Claude规划方案 → 跨文件实现代码 → 验证功能
- Bug修复：粘贴错误信息或描述症状 → 追踪问题根因 → 实施修复

### 3. 创建提交和PR
```bash
# 创建带描述性消息的提交
claude "commit my changes with a descriptive message"
```
支持：暂存更改、写commit message、创建分支、打开PR

CI集成：可自动化代码审查和issue分类（GitHub Actions / GitLab CI/CD）

### 4. MCP工具连接
**Model Context Protocol (MCP)** 是连接AI工具到外部数据源的开放标准：
- 读取Google Drive中的设计文档
- 更新Jira工单
- 从Slack拉取数据
- 使用自定义工具

### 5. 自定义配置

#### CLAUDE.md 文件
项目根目录的markdown文件，每次会话开始时读取：
- 编码标准
- 架构决策
- 首选库
- 审查清单

#### Auto Memory（自动记忆）
Claude工作时自动保存学习内容（构建命令、调试洞察），无需手动编写

#### Custom Commands（自定义命令）
打包可重用的工作流供团队共享：
- `/review-pr` 
- `/deploy-staging`

#### Hooks（钩子）
在Claude Code操作前后运行shell命令：
- 每次文件编辑后自动格式化
- 提交前运行lint

### 6. Agent团队和自定义Agent
- **并行Agent**: 多个Claude Code同时处理不同子任务，主Agent协调
- **Agent SDK**: 构建自定义Agent，完全控制编排、工具访问和权限

### 7. CLI管道和自动化
遵循Unix哲学，可组合使用：
```bash
# 分析最近日志输出
tail -200 app.log | claude -p "Slack me if you see any anomalies"

# CI中自动化翻译
claude -p "translate new strings into French and raise a PR for review"

# 跨文件批量操作
git diff main --name-only | claude -p "review these changed files for security issues"
```

### 8. 定时任务
按计划自动运行Claude：
- **Cloud scheduled tasks**: 在Anthropic管理的基础设施上运行，电脑关机也持续运行
- **Desktop scheduled tasks**: 本地机器运行，直接访问本地文件和工具
- **`/loop`**: 在CLI会话内快速轮询重复prompt

### 9. 跨平台工作
会话不绑定单一界面：
| 需求 | 方案 |
|------|------|
| 手机继续本地会话 | Remote Control |
| 从手机推送任务到桌面 | Message Dispatch |
| Web/iOS启动长任务，终端继续 | `claude --teleport` |
| 终端会话转桌面视觉审查 | `/desktop` |
| Slack路由bug报告→PR | Slack @Claude |

---

## 🔧 所有表面环境的统一引擎

每个界面连接相同的底层Claude Code引擎，因此：
- CLAUDE.md文件跨所有环境通用
- 设置通用
- MCP服务器通用

### 额外集成

| 我想... | 最佳选择 |
|---------|----------|
| 从手机/另一设备继续本地会话 | Remote Control |
| 从Telegram/Discord/iMessage/Webhooks推送事件 | Channels |
| 本地启动，移动端继续 | Web 或 Claude iOS app |
| 定期运行Claude | Cloud scheduled tasks 或 Desktop scheduled tasks |
| 自动化PR审查和issue分类 | GitHub Actions 或 GitLab CI/CD |
| 每个PR自动代码审查 | GitHub Code Review |
| Slack bug报告→PR | Slack |
| 调试实时Web应用 | Chrome |
| 为自己的工作流构建自定义Agent | Agent SDK |

---

## 🚀 后续步骤指南

安装完成后，推荐阅读顺序：

1. **Quickstart**: 第一个真实任务的完整流程（探索代码库→提交修复）
2. **Store instructions and memories**: 使用CLAUDE.md和auto memory持久化指令
3. **Common workflows and best practices**: 充分利用Claude Code的模式
4. **Settings**: 根据工作流定制
5. **Troubleshooting**: 常见问题解决方案

---

**产品详情**: [code.claude.com](https://code.claude.com) - demos, pricing, and product details
