# Claude Code Plugin Marketplaces 完整指南

**来源**:  
- [Discover and install prebuilt plugins through marketplaces](https://code.claude.com/docs/en/discover-plugins) | 官方文档
- [Create and distribute a plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) | 官方文档
- [Plugins reference](https://code.claude.com/docs/en/plugins-reference) | 官方文档

**整理日期**: 2026年4月10日

---

## 📖 概述

Claude Code 的 **plugin marketplace** 是插件目录与分发机制：
- 对使用者来说，它解决“先加市场，再挑插件安装”的发现与安装问题
- 对发布者来说，它解决“如何把 plugin 从本地 `--plugin-dir` 测试，升级为可安装、可更新、可共享的分发单元”的问题

这正好补上了普通 plugin 文档和 `--plugin-dir` 本地开发之间的缺口。

---

## 🧠 先理解两个概念

### Plugin

Plugin 是实际被安装的扩展单元，通常包含：
- `skills/`
- `agents/`
- `hooks/`
- `.mcp.json`
- `.lsp.json`
- `settings.json`
- `.claude-plugin/plugin.json`

### Marketplace

Marketplace 不是单个插件，而是**插件目录索引**。  
Claude Code 先注册 marketplace，再从其中安装具体插件。

可以把它理解为：

```text
plugin = 单个可安装扩展
marketplace = 一组 plugin 的可发现目录
```

---

## 🔄 工作方式

官方文档把 marketplace 的使用流程拆成两步：

1. **添加 marketplace**
   - 只是把插件目录注册到 Claude Code
   - 此时还没有安装任何插件
2. **安装具体 plugin**
   - 在目录里浏览你需要的插件
   - 按名字安装单个插件

这也是为什么一个 plugin 已经能用 `--plugin-dir` 跑起来，不等于它已经具备 marketplace 安装能力。

---

## 🏪 官方 Marketplace

Claude Code 自带官方 Anthropic marketplace：

- 名称：`claude-plugins-official`
- 可通过 `/plugin` 的 `Discover` 标签页浏览
- 也可以在 [claude.com/plugins](https://claude.com/plugins) 查看插件目录

从官方 marketplace 安装插件的命令格式：

```bash
/plugin install <plugin-name>@claude-plugins-official
```

例如：

```bash
/plugin install github@claude-plugins-official
```

官方 marketplace 由 Anthropic 维护；如果要提交自己的 plugin 到官方 marketplace，官方给出的入口是：

- [claude.ai/settings/plugins/submit](https://claude.ai/settings/plugins/submit)
- [platform.claude.com/plugins/submit](https://platform.claude.com/plugins/submit)

---

## 📦 安装与管理 Marketplace

### 交互界面

运行：

```bash
/plugin
```

然后进入 `Marketplaces` 标签页，可以：

- 查看当前已添加的 marketplaces
- 添加新的 marketplace
- 刷新 marketplace 中的插件列表
- 删除 marketplace

### CLI 命令

列出 marketplaces：

```bash
/plugin marketplace list
```

刷新某个 marketplace：

```bash
/plugin marketplace update marketplace-name
```

移除某个 marketplace：

```bash
/plugin marketplace remove marketplace-name
```

> 移除 marketplace 会同时卸载你从该 marketplace 安装的插件。

---

## 🧩 安装与管理 Plugin

常见命令如下：

安装：

```bash
/plugin install plugin-name@marketplace-name
```

禁用：

```bash
/plugin disable plugin-name@marketplace-name
```

重新启用：

```bash
/plugin enable plugin-name@marketplace-name
```

卸载：

```bash
/plugin uninstall plugin-name@marketplace-name
```

更新到最新版本：

```bash
claude plugin update plugin-name@marketplace-name
```

如果安装、启用或禁用发生在当前会话内，官方建议执行：

```bash
/reload-plugins
```

这样无需重启 Claude Code。

---

## 📍 安装作用域（Scope）

官方文档把 plugin / marketplace 的安装范围分成几种 scope：

- `user`
  - 默认范围
  - 对当前用户的所有项目生效
- `project`
  - 写入仓库中的 `.claude/settings.json`
  - 可与协作者共享
- `local`
  - 仅当前仓库、仅当前用户可见
- `managed`
  - 由管理员通过托管配置安装
  - 普通用户不能修改

例如：

```bash
claude plugin install formatter@your-org --scope project
```

---

## 🔧 自动更新机制

Claude Code 支持在启动时自动：

- 刷新 marketplace 列表
- 更新已安装 plugin 到最新版本

默认行为：

- 官方 Anthropic marketplaces：**默认开启 auto-update**
- 第三方 marketplaces / 本地开发 marketplaces：**默认关闭 auto-update**

如果检测到插件已更新，Claude Code 会提示你执行 `/reload-plugins`。

如果你想：

- 禁用 Claude Code 自身自动更新
- 但保留 plugin 自动更新

官方给出的环境变量组合是：

```bash
export DISABLE_AUTOUPDATER=1
export FORCE_AUTOUPDATE_PLUGINS=1
```

---

## 👥 团队共享 Marketplace

团队管理员可以把 marketplace 信息写进项目的 `.claude/settings.json`，让成员在信任仓库后自动收到 marketplace / plugin 安装提示。

官方给出的关键配置项是：

- `extraKnownMarketplaces`
- `enabledPlugins`

一个典型结构如下：

```json
{
  "extraKnownMarketplaces": {
    "my-team-tools": {
      "source": {
        "source": "github",
        "repo": "your-org/claude-plugins"
      }
    }
  }
}
```

这意味着：

- 团队可以把 marketplace 当作项目基础设施的一部分
- 项目 clone 下来后，Claude Code 能发现团队预期使用的插件来源

---

## 🏗️ 自建 Marketplace 的目标

如果你不想只靠官方 marketplace，而是要让自己的 plugin 被一键安装，官方推荐的路径是：**创建并分发自己的 marketplace**。

这类 marketplace 可用于：

- 团队内部私有插件分发
- 社区插件集合
- 公司统一的 Claude Code 插件仓库

---

## 📁 Marketplace 目录结构

官方文档要求 marketplace 有自己的 manifest，位于：

```text
.claude-plugin/marketplace.json
```

一个典型 marketplace 结构可以整理为：

```text
my-marketplace/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    ├── plugin-a/
    │   ├── .claude-plugin/
    │   │   └── plugin.json
    │   ├── skills/
    │   └── agents/
    └── plugin-b/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
```

要点是：

- marketplace 自己有 `marketplace.json`
- marketplace 里的每个 plugin 仍然是独立的 Claude Code plugin
- plugin 的结构规则与普通 plugin 完全一致

---

## 🌍 Marketplace 来源类型

官方 CLI 支持从多种来源添加 marketplace：

### GitHub 简写

```bash
claude plugin marketplace add acme-corp/claude-plugins
```

### 固定到分支或标签

GitHub 简写可用 `@ref`：

```bash
claude plugin marketplace add acme-corp/claude-plugins@main
```

### 非 GitHub 的 git URL

```bash
claude plugin marketplace add https://gitlab.example.com/team/plugins.git
```

### 直接指向 `marketplace.json` 的远程 URL

```bash
claude plugin marketplace add https://example.com/marketplace.json
```

### 本地目录

```bash
claude plugin marketplace add ./my-marketplace
```

### Monorepo 稀疏检出

官方支持 `--sparse`，适合插件目录嵌在大仓库中：

```bash
claude plugin marketplace add acme-corp/monorepo --sparse .claude-plugin plugins
```

---

## 🧪 Marketplace 验证与测试

官方建议在分享 marketplace 之前先验证：

```bash
claude plugin validate .
```

或在 Claude Code 内：

```bash
/plugin validate .
```

然后按下面流程做冒烟测试：

1. 添加 marketplace

```bash
/plugin marketplace add ./path/to/marketplace
```

2. 安装其中一个测试 plugin

```bash
/plugin install test-plugin@marketplace-name
```

这一步可以快速判断：

- `marketplace.json` 是否存在
- marketplace 是否能被识别
- plugin 元数据是否有效
- plugin 是否能被实际安装

---

## 🛠️ 非交互 CLI 子命令

官方文档专门给出了可脚本化的 `claude plugin marketplace` 子命令：

添加：

```bash
claude plugin marketplace add <source> [options]
```

列出：

```bash
claude plugin marketplace list [options]
```

删除：

```bash
claude plugin marketplace remove <name>
```

更新：

```bash
claude plugin marketplace update [name]
```

其中：

- `add` 支持 `--scope`
- `add` 支持 `--sparse`
- `list` 支持 `--json`
- `remove` 的参数是 marketplace 名称，不是你最初传入的 source

---

## 🔐 安全模型

官方明确强调：

- plugins 和 marketplaces 都是**高信任组件**
- 它们可以以当前用户权限在本机执行任意代码
- 只能安装你信任的来源

组织管理员还可以通过托管配置限制用户允许添加哪些 marketplaces。

这意味着 marketplace 不是“纯索引文件”，它实际上是你对插件来源链路的信任声明。

---

## 🗃️ 安装后的缓存与路径限制

`plugins-reference` 里有一个非常关键、容易忽略的实现细节：

- Claude Code 安装 marketplace plugin 时，会把插件复制到本地缓存目录
- 默认不是“就地运行”源目录
- 每个已安装版本都会成为缓存中的独立目录

官方文档给出的缓存目录是：

```text
~/.claude/plugins/cache
```

这带来两个直接约束：

### 1. 不能依赖插件目录外的相对路径

例如这类路径在安装后会失效：

```text
../shared-utils
```

因为缓存时只复制插件本身，不复制外部目录。

### 2. 旧版本缓存不会立刻硬删

更新或卸载后，旧版本目录会被标记为 orphaned，并在约 7 天后清理。  
这样可以避免并发会话在旧版本仍运行时立刻崩掉。

### 3. 如需引用外部资源，官方建议使用符号链接

也就是说，发布成 marketplace plugin 时，最好保证：

- plugin 是自包含的
- 或者通过符号链接显式处理依赖

---

## ⚠️ 常见问题

### Marketplace 无法加载

优先检查：

- URL 或仓库地址是否可访问
- `.claude-plugin/marketplace.json` 是否存在
- JSON 语法是否合法
- 私有仓库权限是否可用

### Plugin 安装后文件丢失

高概率原因：

- plugin 引用了插件目录外的路径
- 安装到缓存后，这些路径不再存在

### 技能或 agent 不出现

优先检查：

- plugin 根目录结构是否正确
- `skills/`、`agents/` 是否放在 plugin root，而不是误放进 `.claude-plugin/`
- 是否执行了 `/reload-plugins`

---

## 🎯 对发布者最重要的结论

如果你的目标是让一个像 `plan-for-all` 这样的 plugin 不再只能通过 `--plugin-dir` 启动，而是能够被用户“一键安装”，官方路径只有两类：

1. **提交到官方 Anthropic marketplace**
   - 适合公共分发
   - 通过官方提交通道进入生态

2. **自建 marketplace**
   - 适合团队私有分发、社区目录、组织统一安装
   - 需要补齐 `.claude-plugin/marketplace.json` 与 marketplace 目录结构

换句话说：

```text
能当 plugin ≠ 能被 marketplace 安装
```

真正具备 marketplace 安装能力，需要再补一层 **marketplace 分发结构**。

---

## 🔗 与现有文档的关系

这篇文档补的是 `11-plugins-complete-guide.md` 没有展开的部分：

- `11-plugins-complete-guide.md`
  - 重点在“如何做一个 plugin”
- `14-claude-code-plugin-marketplaces-complete-guide.md`
  - 重点在“如何让 plugin 被发现、安装、更新、共享”
- `plugins-reference`
  - 重点在 schema、CLI 和底层技术约束

如果你要解决“market 安装”问题，建议阅读顺序是：

1. `11-plugins-complete-guide.md`
2. 本文
3. `plugins-reference` 中的 manifest / cache / CLI 章节

---

*文档来源: Claude Code 官方文档 `discover-plugins`、`plugin-marketplaces`、`plugins-reference`*
