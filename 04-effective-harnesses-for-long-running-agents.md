# Effective Harnesses for Long-Running Agents

**作者**: Justin Young  
**发布日期**: 2025年11月26日  
**来源**: [Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

---

## 概述

代理在跨多个上下文窗口工作时仍面临挑战。我们从人类工程师那里寻找灵感，创建了更有效的长时间运行代理框架。

随着AI代理变得更强，开发者越来越多地要求它们承担需要工作跨越数小时甚至数天的复杂任务。然而，让代理在多个上下文窗口中取得一致的进展仍然是一个开放问题。

### 核心挑战

长时间运行代理的核心问题是它们必须在离散会话中工作，并且每个新会话开始时对之前发生的事情没有记忆。

**类比**: 想象一个软件项目由轮班工程师人员配备，每个新工程师到达时对前一班发生了什么没有记忆。

由于上下文窗口有限，而且大多数复杂项目无法在单个窗口内完成，代理需要一种方法来弥合编码会话之间的差距。

## 解决方案概览

我们开发了**两重解决方案**使 Claude Agent SDK 能够有效地跨多个上下文窗口工作：

1. **Initializer agent（初始化代理）**: 第一次运行时设置环境
2. **Coding agent（编码代理）**: 被要求在每个会话中取得增量进展，同时为下一个会话留下清晰的制品

**代码示例**: 可在 accompanying quickstart 中找到

## 长时间运行代理问题

### Claude Agent SDK 能力

Claude Agent SDK 是一个强大、通用的代理框架，擅长编码以及其他需要模型使用工具收集上下文、规划和执行的任务。它具有上下文管理能力，如**compaction（压缩）**，使代理能够在不耗尽上下文窗口的情况下继续工作。

**理论上**: 给定这种设置，代理应该能够任意长时间地继续做有用的工作。

### 理论与实践的差距

**然而，压缩是不够的**。开箱即用的即使是最前沿的编码模型如Opus 4.5，如果只给定高级提示（如"build a clone of claude.ai"），在Claude Agent SDK上跨多个上下文窗口循环运行也无法构建生产质量的Web应用。

### Claude的两种失败模式

#### 失败模式1: 一次做完倾向

代理倾向于试图一次做得太多——本质上是尝试一次性完成应用。

**后果**:
- 经常导致模型在实施中间耗尽上下文
- 下一个会话开始时功能半实现且无文档
- 代理不得不猜测发生了什么，花费大量时间尝试重新让基本应用工作
- **即使有压缩也会发生**——压缩并不总是传递完美清晰的指令给下一个代理

#### 失败模式2: 过早宣布胜利

项目后期，后来的代理实例会环顾四周，看到已经取得了进展，然后宣布工作完成。

### 问题分解

这两个失败模式将问题分解为两部分：

1. **设置初始环境**：奠定给定提示所需的所有功能的基础，设置代理逐步逐功能地工作
2. **增量进展**：提示每个代理朝目标取得增量进展，同时在会话结束时保持环境清洁状态

**"清洁状态"意味着**: 适合合并到主分支的代码：没有重大bug、代码有序且有文档、开发者可以轻松开始新功能而不必先清理无关混乱

## 两重解决方案详解

### Part 1: Initializer Agent（初始化代理）

**特殊用途**: 第一个非常代理会话使用专门的提示，要求模型设置**初始环境**：

#### 关键组件

##### 1. Feature List（功能列表）

为了解决代理一次性完成应用或过早考虑项目完成的问题，我们提示初始化代理编写全面的**功能需求文件**，扩展用户的初始提示。

**claude.ai clone 示例**: 超过200个功能，如"a user can open a new chat, type in a query, press enter, and see an AI response"

**结构** (JSON格式):

```json
{
  "category": "functional",
  "description": "New chat button creates a fresh conversation",
  "steps": [
    "Navigate to main interface",
    "Click the 'New Chat' button",
    "Verify a new conversation is created",
    "Check that chat area shows welcome state",
    "Verify conversation appears in sidebar"
  ],
  "passes": false
}
```

**为什么选择JSON而非Markdown**: 实验后发现模型不太可能不当改变或覆盖JSON文件

**管理策略**:
- 我们提示编码代理**只能通过更改passes字段的状态来编辑此文件**
- 使用强烈措辞的指令："It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."
- 所有功能最初标记为"failing"，以便后续编码代理有清晰的全功能外观轮廓

##### 2. init.sh Script（初始化脚本）
编写可以运行开发服务器的初始化脚本

##### 3. claude-progress.txt File（进度文件）
保持代理已完成工作的日志

##### 4. Initial Git Commit（初始Git提交）
显示添加了哪些文件的初始git提交

### Part 2: Coding Agent（编码代理）

**每个后续会话**要求模型：
1. **只做一个功能的增量工作**
2. **留下结构化更新**

#### 为什么增量方法至关重要

给定上述初始环境脚手架，下一个编码代理迭代然后被要求**一次只在一个功能上工作**。这种增量方法对于解决代理一次做太多事情的倾向被证明是**关键的**。

#### 保持环境清洁

一旦工作增量，模型在代码更改后**仍然必须保持环境清洁状态**至关重要。

**实验发现**：引出此行为的最佳方式是要求模型：
- 用描述性提交消息将其进度提交到git
- 在progress file中写进度的摘要

**好处**:
- 这允许模型使用git恢复不良代码更改
- 恢复代码库的工作状态
- 也提高了效率，因为消除了代理猜测发生了什么并花费时间尝试重新让基本应用工作的需要

### 关键洞察

找到一种方式让代理在**从新鲜上下文窗口开始时快速理解工作状态**，这是通过 **claude-progress.txt 文件**配合 git history 完成的。

**灵感来源**: 这些实践的灵感来自于了解有效的软件工程师每天做的事情。

## 环境管理详细指南

### Testing（测试）：最后一个主要失败模式

**观察到的问题**: Claude倾向于在不适当测试的情况下将功能标记为完成

**具体表现**:
- Claude确实会做代码更改
- 甚至会用单元测试或针对开发服务器的curl命令做测试
- 但**无法识别**该功能端到端不起作用

**解决方案**: 明确提示Claude使用浏览器自动化工具**像人类用户那样进行所有测试**

**效果**: 提供这些类型的测试工具显著改善了性能，代理能够识别和修复代码中不明显的bug

**实际证据**: Claude通过Puppeteer MCP服务器拍摄的claude.ai clone测试截图

**仍有限制**:
- Claude视觉能力的限制
- 浏览器自动化工具的限制使得识别每种类型的bug变得困难
- 例如：Claude不能通过Puppeteer MCP看到浏览器原生alert模态框，依赖这些模态框的功能因此往往更有bug

## Getting Up to Speed（快速上手）

有了以上所有内容，每个编码代理都被提示运行一系列步骤来**了解当前情况**，有些相当基本但仍有帮助：

### 标准启动流程（每会话开始时）

```
1. 运行 pwd 查看你正在工作的目录。你只能编辑此目录中的文件。
2. 阅读 git logs 和 progress 文件以了解最近做了什么工作。
3. 阅读 features list 文件并选择尚未完成的最高优先级功能来工作。
```

**节省token**: 这种方法在每次会话中为Claude节省了一些token，因为它不必弄清楚如何测试代码

**也有助于**: 要求初始化代理编写init.sh脚本，在实现新功能之前运行开发服务器并通过基本端到端测试

**claude.ai clone 案例**:
- 代理总是启动本地开发服务器
- 使用Puppeteer MCP开始新聊天、发送消息、接收响应
- 确保Claude能快速识别应用是否处于损坏状态
- 立即修复任何现有bug
- 如果代理改为开始实现新功能，可能会使问题恶化

### 典型会话开始流程示例

```text
[Assistant] I'll start by getting my bearings and understanding the current state of the project.

[Tool Use] <bash - pwd>

[Tool Use] <read - claude-progress.txt>

[Tool Use] <read - feature_list.json>

[Assistant] Let me check the git log to see recent work.
[Tool Use] <bash - git log --oneline -20>

[Assistant] Now let me check if there's an init.sh script to restart the servers.
<Starts the development server>

[Assistant] Excellent! Now let me navigate to the application and verify that some fundamental features are still working.
<Tests basic functionality>

[Assistant] Based on my verification testing, I can see that the fundamental functionality is working well. The core chat features, theme switching, conversation loading, and error handling are all functioning correctly. Now let me review the tests.json file more comprehensively to understand what needs to be implemented next.
<Starts work on a new feature>
```

## Agent Failure Modes and Solutions Summary

| 问题 | Initializer Agent 行为 | Coding Agent 行为 |
|------|------------------------|-------------------|
| **Claude过早宣布整个项目胜利** | 设置feature list文件：基于input spec，设置结构化JSON文件包含end-to-end功能描述列表 | 会话开始时阅读feature list文件。选择单个功能开始工作 |
| **Claude留给环境有bug或无文档进展的状态** | 写入initial git repo和progress notes file | 通过阅读progress notes file和git commit logs开始会话，在开发服务器上运行基本测试以捕获任何未记录的bug。会话结束时写git commit和progress update |
| **Claude过早标记功能为完成** | 设置feature list文件 | 自我验证所有功能。仅在仔细测试后将功能标记为"passing" |
| **Claude必须花时间搞清楚如何运行app** | 编写可以运行development server的init.sh脚本 | 会话开始时读取init.sh |

## 未来工作方向

本研究展示了长时间运行代理框架中一组可能的解决方案，使模型能够跨许多上下文窗口取得增量进展。然而，**仍有开放问题**：

### 主要未解决问题

**单代理 vs. 多代理架构**:
- 目前尚不清楚单一通用编码代理是否在所有上下文中表现最佳
- 或者是否可以通过**多代理架构**实现更好的性能
- 合理推测：专业代理如testing agent、quality assurance agent或code cleanup agent可能在软件开发生命周期的子任务上做得更好

**领域通用性**:
- 本demo针对全栈Web应用开发优化
- 未来方向是将这些发现推广到其他领域
- 可能其中一些或全部课程可以应用于科学研究或金融建模等其他领域所需的长时间agentic任务类型

---

## 致谢

**作者**: Justin Young

**特别感谢**: David Hershey, Prithvi Rajasakeran, Jeremy Hadfield, Naia Bouscal, Michael Tingley, Jesse Mu, Jake Eaton, Marius Buleandara, Maggie Vo, Pedram Navid, Nadine Yasser, Alex Notov

**注**: 本工作反映了使Claude能够安全地进行长距离自主软件工程的Anthropic多个团队的集体努力，特别是code RL & Claude Code团队。

**感兴趣的候选人**: 欢迎在 anthropic.com/careers 申请贡献
