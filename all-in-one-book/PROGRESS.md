# Loom 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 0 | 序言：Loom 全景地图 | ch00-preface.md | 项目定位 / 架构全景图 / 核心概念词典 / 代码库地图 / 一次典型交互全流程 | ✅ |
| 1 | 数据流全景：一条消息的完整旅程 | ch01-data-flow.md | 用户输入 → CLI → 状态机 → LLM 代理 → 服务器 → LLM 提供商 → 工具执行 → 自动提交 → 响应展示 | ✅ |
| 2 | 大脑：Agent 状态机 | ch02-state-machine.md | 7 个状态 / 6 类事件 / 7 种动作 / 控制反转设计 / 状态转移图 / 错误恢复 | ✅ |
| 3 | 代理层：LLM 代理架构 | ch03-llm-proxy.md | 客户端 ProxyLlmClient / 服务器 LlmService / SSE 流式传输 / Anthropic OAuth 池 / 多提供商路由 | ✅ |
| 4 | 手与脚：工具系统 | ch04-tool-system.md | Tool trait / ToolRegistry / 内置工具（ReadFile, EditFile, Bash, ListFiles, Oracle, WebSearch）/ 安全边界 | ⏳ |
| 5 | 记忆：Thread 与会话持久化 | ch05-thread-system.md | Thread 数据模型 / LocalThreadStore / SyncingThreadStore / 服务器端存储 / FTS5 搜索 | ⏳ |
| 6 | 身份与权限：认证授权体系 | ch06-auth-system.md | OAuth PKCE / Magic Link / Device Code / API Key / Session / ABAC 策略引擎 / 审计日志 | ⏳ |
| 7 | 远程织机：Weaver 远程执行环境 | ch07-weaver-system.md | K8s Pod 编排 / Provisioner / WireGuard 隧道 / eBPF 审计 / SPIFFE 身份 / 生命周期管理 | ⏳ |
| 8 | 可观测性平台 | ch08-observability.md | Analytics（PostHog 风格）/ Crash Reporting（符号化 + 指纹）/ Feature Flags（多变体 + 实验 + 熔断）/ Sessions & Crons 监控 | ⏳ |
| 9 | 界面层：TUI 与 Web 前端 | ch09-ui-layer.md | Ratatui 组件体系 / Svelte 5 Web 前端 / 实时通信（SSE/WebSocket）| ⏳ |
| 10 | 支线系统：版本控制、搜索与集成 | ch10-auxiliary-systems.md | Spool（jj 版本控制）/ Auto-Commit / SCM 托管 / Clips / SCIM / WhatsApp / i18n | ⏳ |
| 11 | 项目演进史 | ch11-evolution.md | 从初始提交到当前架构的演进过程（注：项目仅 2 个 commit，以架构设计意图推演为主）| ⏳ |
| 12 | 端到端追踪：三个关键场景 | ch12-e2e-trace.md | 场景1：首次登录并发起对话 / 场景2：工具调用与自动提交 / 场景3：创建远程 Weaver 并执行代码 | ⏳ |

## 章节规划说明

认知路径设计：

1. **序言**（第 0 章）— 读者首先需要知道"Loom 是什么、长什么样"，建立全局心智模型
2. **数据流全景**（第 1 章）— 用一次完整交互串联所有模块，让读者看到"整体怎么工作"
3. **状态机**（第 2 章）— 理解了全景后，深入最核心的"大脑"——Agent 状态机，这是理解一切的基础
4. **LLM 代理**（第 3 章）— 状态机发出"调用 LLM"的动作后，消息如何到达真正的 AI 提供商
5. **工具系统**（第 4 章）— LLM 返回工具调用后，工具如何被执行
6. **Thread 系统**（第 5 章）— 对话过程中产生的数据如何被持久化
7. **认证授权**（第 6 章）— 上述所有流程的前提：用户是谁、有什么权限
8. **Weaver**（第 7 章）— 从本地执行到远程执行的跨越
9. **可观测性**（第 8 章）— 贯穿全系统的监控与实验基础设施
10. **界面层**（第 9 章）— 用户实际看到和操作的部分
11. **支线系统**（第 10 章）— 补充模块：版本控制、搜索、外部集成
12. **演进史**（第 11 章）— 理解为什么架构是现在这个样子
13. **端到端追踪**（第 12 章）— 用完整场景串联和验证全书知识

## 状态说明
- ✅ 已完成
- 🔄 进行中
- ⏳ 待开始

## 术语约定

### 行业标准术语（直接使用）
- **LLM** (Large Language Model) — 大语言模型
- **SSE** (Server-Sent Events) — 服务器推送事件，HTTP 单向流式协议
- **OAuth 2.0 PKCE** — 授权框架的安全扩展（Proof Key for Code Exchange）
- **ABAC** (Attribute-Based Access Control) — 基于属性的访问控制
- **RBAC** (Role-Based Access Control) — 基于角色的访问控制
- **K8s** — Kubernetes 的简写
- **SPIFFE** — 安全的工作负载身份标准
- **WireGuard** — 现代 VPN 协议
- **eBPF** — Linux 内核可编程追踪框架
- **DERP** — Designated Encrypted Relay Protocol（Tailscale 设计的中继协议）
- **FTS5** — SQLite 全文搜索引擎
- **SCIM** — 跨域身份管理标准
- **gettext** — GNU 国际化框架

### 项目特有术语
- **Weaver（织机）** — Loom 的远程执行环境，实际是一个 K8s Pod。类比：一个按需创建的云端开发容器。
- **Thread（线程/对话线）** — 一次完整的人机对话记录，包含所有消息和工具调用。类比：聊天记录。
- **Spool（线轴）** — Loom 基于 jj 的版本控制系统，使用纺织术语命名。类比：Git，但用不同的概念模型。
  - **Stitch（针脚）** = Commit
  - **Pin（别针）** = Branch/Bookmark
  - **Knot（结）** = 带消息的正式提交
  - **Tangle（缠结）** = 冲突
  - **Rethread（重穿线）** = Rebase
- **Clip（片段）** — 短代码片段分享，类比 GitHub Gist。
- **ProxyLlmClient** — 客户端侧的 LLM 代理客户端，通过 HTTP 转发请求到服务器。
- **LlmService** — 服务器侧的 LLM 服务，管理多个提供商的认证和路由。
- **Provisioner** — Weaver 的编排器，管理 K8s Pod 的创建、监控和清理。

## 下次续写指引
### 从哪里继续
从第 4 章（工具系统）开始写作。第 0-3 章已完成。

### 交接备忘
- 项目仅有 2 个 commit（"Test deployment commit" 和 "Trigger rebuild"），演进史章节需要基于架构设计意图和 spec 文件来推演设计决策过程
- 代码库约 80+ 个 crate，50+ 个 spec 文件，结构完整但许多功能可能尚在开发中
- 核心数据流：CLI → ProxyLlmClient → Server LlmProxy Route → LlmService → AnthropicClient → Claude API → SSE Stream 回传
- 状态机采用控制反转：状态机返回 Action，调用者执行 I/O

### 待验证项
- [ ] AnthropicPool 的具体 round-robin + failover 算法细节
- [ ] TUI storybook 的 visual snapshot testing 实际实现情况
- [ ] Spool 系统与 jj 的具体集成方式（是 CLI wrapper 还是 library binding）
- [ ] WebSocket 升级（phase3）的实际实现状态
