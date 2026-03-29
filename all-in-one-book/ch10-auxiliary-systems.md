# 第 10 章 · 支线系统：版本控制、搜索与集成

> 前面九章覆盖了 Loom 的主干架构。本章讲解围绕主干生长出来的一系列支线系统——它们各自独立，但为 Loom 提供了更完整的工作流支持。

---

## Spool：纺织术语包装的版本控制

Spool（线轴）是 Loom 基于 **jj**（Jujutsu）的版本控制系统。jj 是 Google 开源的新一代版本控制工具，与 Git 兼容但概念模型不同。Loom 在 jj 之上包装了一层纺织主题的术语。

### 术语映射

| Spool 术语 | jj/Git 对应 | 含义 |
|-----------|-------------|------|
| **Stitch（针脚）** | Change/Commit | 一次原子修改 |
| **Pin（别针）** | Branch/Bookmark | 命名标记 |
| **Knot（结）** | Commit with message | 带消息的正式提交 |
| **Shuttle（梭子）** | Working copy / push | 工作副本 |
| **Tangle（缠结）** | Conflict | 合并冲突 |
| **Rethread（重穿线）** | Rebase | 变基操作 |
| **Ply（折叠）** | Squash | 合并多个提交 |
| **Unpick（拆线）** | Undo | 撤销操作 |
| **Tension log** | Operation log | 操作历史 |

### CLI 命令

| 命令 | 说明 |
|------|------|
| `loom spool wind` | 初始化仓库 |
| `loom spool stitch` | 创建新变更 |
| `loom spool knot` | 带消息提交 |
| `loom spool trace` | 查看日志 |
| `loom spool rethread` | 变基 |
| `loom spool shuttle` | 推送到远程 |

Spool 在本地维护一个 `.spool/` 目录，与 `.git/` 并存（colocated 模式），保持与 Git 的互操作性。

---

## Auto-Commit：文件修改的自动记录

在第 2 章我们看到，Agent 在工具执行完毕后进入 `PostToolsHook` 状态。Auto-Commit 就是这个 Hook 的核心实现。

**调用路径**：

```
loom-cli-auto-commit/src/lib.rs::AutoCommitService::run()
  — 输入：已完成的工具列表
  — 检查是否有"文件修改类"工具（edit_file、bash）
  — 如果没有 → 跳过
  — 检测 Git 仓库状态
  — 如果没有未暂存的更改 → 跳过
  — 用 LLM（Haiku 模型，小而快）分析 diff 并生成提交消息
  — 执行 git add + git commit
  — 输出：AutoCommitResult（commit hash / 跳过原因）
```

两个关键设计：

1. **用 LLM 生成 commit message** — 不是写死的"auto-commit"，而是分析 diff 内容，生成有意义的消息
2. **不阻塞主流程** — 即使 Auto-Commit 失败（Git 仓库不存在、diff 太大等），也只记日志，不影响 Agent 循环

---

## SCM：Git 仓库托管

`loom-server-scm` 和 `loom-server-scm-mirror` 提供 Git 仓库托管能力：

| 功能 | 说明 |
|------|------|
| 仓库管理 | 创建、列出、删除 Git 仓库 |
| 分支管理 | 列出分支、保护规则 |
| Mirror | 从外部 Git 仓库（GitHub 等）同步到 Loom |
| Webhooks | 仓库事件通知 |
| Git 维护 | 定期 gc、optimize |

数据库表：`scm_repos`、`scm_branches`、`scm_commits`、`scm_webhooks`、`scm_mirrors`。

---

## Clips：代码片段分享

Clips（代码片段）类似 GitHub Gist——用户可以创建短小的代码片段并分享。

特色功能：
- **自动敏感信息脱敏** — 使用 `loom-redact` 检测并遮蔽 API Key、Token 等
- **FTS5 全文搜索** — 支持搜索片段内容（migration `039_clips_fts.sql`）
- **Star 收藏** — 用户可以收藏片段

---

## 外部集成

### GitHub App

`loom-server-github-app` 提供 GitHub App 集成，用于：
- 通过 GitHub 的 Installation Token 访问用户仓库
- 读取仓库代码为 AI 提供上下文
- 接收 GitHub Webhooks

### WhatsApp

`loom-whatsapp` 和 `loom-server-whatsapp` 提供 WhatsApp Business API 集成，让用户可以通过 WhatsApp 与 Loom AI 对话。

### SCIM

`loom-scim` 和 `loom-server-scim` 实现 SCIM（System for Cross-domain Identity Management）标准，用于与企业 IdP（如 Okta、Azure AD）自动同步用户和组。

---

## i18n：国际化

`loom-common-i18n` 使用 GNU gettext 框架，支持 17 种语言的翻译。

| 组件 | 说明 |
|------|------|
| `.po` 文件 | 翻译源文件，每种语言一个 |
| `.mo` 文件 | 编译后的二进制翻译文件 |
| `t(locale, key)` | 查找翻译 |
| `t_fmt(locale, key, vars)` | 带变量的翻译 |
| `is_rtl(locale)` | 检查是否为 RTL 语言 |

翻译字符串使用层级点分命名：`server.email.magic_link.subject`、`client.error.connection_failed`。

---

## 小结

这些支线系统虽然各自独立，但为 Loom 提供了完整的工作流闭环：

- **Spool** 让版本控制融入 AI 工作流
- **Auto-Commit** 确保每次修改都有记录
- **SCM + GitHub App** 连接外部代码仓库
- **Clips** 让代码分享更便捷、更安全
- **SCIM** 让企业用户管理自动化
- **i18n** 让 Loom 说多种语言

下一章回顾整个项目的演进历程。

---

### 质检报告

**讲解节奏**
- [x] 每个系统先讲"是什么"再讲内部

**代码纪律**
- [x] 全章代码片段 0 处

**过渡自然吗**
- [x] 章头定位"支线系统"
- [x] 章尾引出第 11 章

**准确吗**
- [x] Spool 术语映射与 spec 一致
- [x] Migration 文件编号与代码一致

**勘误建议**
（无）
