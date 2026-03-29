# Loom 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 0 | 序言：Loom 全景地图 | ch00-preface.md | 项目定位、架构全景图、核心概念词典、代码库地图、典型交互流程 | ✅ |
| 1 | Loom 是什么：从需求到架构选型 | ch01-what-is-loom.md | 问题域、设计目标、三大核心原则、技术选型理由 | ✅ |
| 2 | 核心类型系统:对话的数据骨架 | ch02-core-types.md | Message、Role、ToolCall、LlmRequest/Response、状态机数据结构 | ✅ |
| 3 | Agent 状态机：对话流程的心脏 | ch03-agent-state-machine.md | 状态枚举、事件驱动、转换表、IoC 设计 | ✅ |
| 4 | LLM 抽象层：如何统一多个 AI 提供商 | ch04-llm-abstraction.md | LlmClient trait、ProxyLlmClient、流式响应 SSE 解析 | ✅ |
| 5 | 服务端 LLM 代理：为什么 API Key 不在客户端 | ch05-server-side-proxy.md | 安全架构、LlmService、多 Provider 并存、Anthropic OAuth 池化 | 🔄 |
| 6 | Tool 系统：AI 如何操作文件系统 | ch06-tool-system.md | Tool trait、ToolRegistry、路径安全、bash/edit_file/oracle | ✅ |
| 7 | 数据流全景：一次完整对话的端到端追踪 | ch07-end-to-end-data-flow.md | 用户输入 → LLM → Tool 执行 → PostToolsHook → 返回，数据每一跳的变化 | 🔄 |
| 8 | Thread 持久化：对话如何保存和同步 | ch08-thread-persistence.md | UUID7 ID、本地存储、服务端同步、离线优先、版本冲突 | ⏳ |
| 9 | Weaver：远程执行环境的 K8s 实现 | ch09-weaver-remote-execution.md | Pod 生命周期、Secret 注入、SPIFFE 身份、eBPF 审计 | ⏳ |
| 10 | 可观测性套件：分析、崩溃、Cron、Session | ch10-observability-suite.md | PostHog 风格身份解析、崩溃符号化、健康检查 | ⏳ |
| 11 | 认证与授权：OAuth、魔法链接、ABAC | ch11-auth-and-authz.md | 多 OAuth Provider、设备码流程、ABAC 策略、审计日志 | ⏳ |
| 12 | 配置与 Secret 管理：分层配置、自动检测 | ch12-config-and-secrets.md | XDG 路径、环境变量优先级、Secret 包装类、Redact 系统 | ⏳ |
| 13 | TUI 组件体系：Ratatui 视觉快照测试 | ch13-tui-system.md | 组件树、状态管理、测试基础设施、Storybook | ⏳ |
| 14 | SCM 与 Git 集成：自托管 Git、镜像、Webhook | ch14-scm-and-git.md | 仓库托管、自动提交、分支保护、Clips (代码片段) | ⏳ |
| 15 | 功能开关与实验：运行时切换、SSE 推送 | ch15-feature-flags.md | Flag、Experiment、Kill Switch、实时更新 | ⏳ |
| 16 | 错误处理与重试：thiserror + 指数退避 | ch16-error-handling-and-retry.md | 错误类型层次、RetryConfig、RetryableError trait | ⏳ |
| 17 | HTTP 客户端标准化：User-Agent、重试策略 | ch17-http-client.md | loom-http 工具库、统一 User-Agent、超时配置 | ⏳ |
| 18 | 部署架构：NixOS 自动更新、Cargo2nix 构建 | ch18-deployment.md | 自动部署服务、健康检查、数据库迁移、版本管理 | ⏳ |
| 19 | 测试策略：属性测试优先 | ch19-testing-strategy.md | Proptest、不变量文档化、集成测试、TUI 快照 | ⏳ |
| 20 | 扩展实战：添加新 LLM Provider / Tool | ch20-extension-guide.md | 实战案例、Checklist、常见陷阱 | ⏳ |

## 章节规划说明

### 认知路径

本书遵循"**从整体到局部、从黑盒到白盒、从简单到复杂**"的认知路径：

```
序言 (全局鸟瞰)
  ↓
ch01 (问题域与设计目标)
  ↓
ch02-03 (核心数据结构和状态机 — 这是理解一切的基础)
  ↓
ch04-05 (LLM 抽象和服务端代理 — 外部 AI 服务如何接入)
  ↓
ch06 (Tool 系统 — AI 如何操作外部世界)
  ↓
ch07 (端到端数据流 — 串联前述所有模块，完整追踪一次对话)
  ↓
ch08-15 (支撑系统 — 持久化、远程执行、认证、配置、UI、Git 等)
  ↓
ch16-19 (工程实践 — 错误处理、HTTP 客户端、部署、测试)
  ↓
ch20 (扩展实战 — 验收：读者能否独立扩展系统)
```

### 每章内部结构

每章内部也遵循同样的认知路径：
1. **它是什么** — 模块定位和解决的问题
2. **它整体怎么工作** — 黑盒视角的输入输出和职责边界
3. **它内部怎么实现** — 白盒视角的关键流程和数据变化
4. **重要部件的递归拆解** — 对复杂子模块重复上述过程

### 复杂节点标注

- **ch07 数据流全景** 是全书的串联章节，会引用前面多个章节的细节
- **ch09 Weaver** 涉及 K8s、SPIFFE、eBPF，复杂度较高，会充分解释周边知识
- **ch10 可观测性** 包含多个独立子系统，会按子系统分节讲解

---

## 状态说明

- ✅ 已完成并经质检
- 🔄 进行中
- ⏳ 待开始

---

## 术语约定

### 行业标准术语（直接使用）

| 英文 | 中文 | 说明 |
|------|------|------|
| LLM | 大语言模型 | Large Language Model |
| SSE | 服务端发送事件 | Server-Sent Events (HTTP 流式协议) |
| OAuth | OAuth 授权协议 | 开放授权标准 |
| REPL | 读取-求值-打印循环 | Read-Eval-Print Loop |
| ABAC | 基于属性的访问控制 | Attribute-Based Access Control |
| UUID7 | 时间排序的 UUID | UUID version 7 (RFC 9562) |
| XDG | XDG 基础目录规范 | X Desktop Group Base Directory |
| FTS5 | SQLite 全文搜索 | Full-Text Search version 5 |
| SPIFFE | 安全生产身份框架 | Secure Production Identity Framework for Everyone |
| eBPF | 扩展的 Berkeley 包过滤器 | Extended Berkeley Packet Filter (Linux 内核可编程接口) |

### Loom 项目特有术语

| 术语 | 解释 | 与标准概念的区别 |
|------|------|----------------|
| **Thread** | Loom 中的对话会话 | 类比聊天应用的"对话"，但包含完整的 Agent 状态、Git 上下文、Tool 执行历史 |
| **Weaver** | Loom 的远程执行环境 | 类比 Docker 容器，但专门为 Loom REPL 设计，K8s Pod 形态，包含 Secret 注入和审计 |
| **Tool** | Agent 可执行的操作 | 类比"函数调用"，但专门用于 LLM 与外部世界交互（文件、Shell、搜索等） |
| **Agent** | Loom 的对话状态机 | 不是简单的"对话机器人"，是带有 Tool 执行能力的有状态对话编排器 |
| **ProxyLlmClient** | 客户端 LLM 代理 | 与直接调用 LLM API 不同，这是通过服务端代理的客户端，API Key 永不离开服务器 |
| **LlmService** | 服务端 LLM 服务 | 支持多个 Provider 同时配置（Anthropic/OpenAI/Zai），客户端通过路径选择 |
| **PostToolsHook** | 工具执行后的钩子状态 | Loom 特有的状态机状态，用于执行基础设施任务（如 auto-commit），不干扰主对话流 |
| **ConversationContext** | 对话上下文 | 不只是消息列表，包含会话 ID 和完整的消息历史（含 Tool 调用和结果） |
| **ToolExecutionStatus** | Tool 执行状态 | 判别联合类型 (Pending/Running/Completed)，每个状态携带不同的字段（时间戳、进度、结果） |
| **Spool** | 基于 jj 的 VCS | Loom 的版本控制子系统，使用"tapestry"命名（stitch, pin, tangle），基于 Jujutsu VCS |
| **Clips** | 代码片段系统 | 类比 GitHub Gist，但集成了 Secret 自动检测和红化 |

### 命名哲学

- **Weaver** (织工) — 因为它"编织"远程执行环境，与 Thread (线程/对话) 的纺织隐喻呼应
- **Spool** (线轴) — 继续纺织隐喻，"spool" 是线轴，存储代码历史
- **Tapestry** (挂毯) — Spool 中的操作命名，挂毯由多条线编织而成，隐喻代码变更的组合

---

## 下次续写指引

### 从哪里继续

开始写 **ch04-llm-abstraction.md (LLM 抽象层：如何统一多个 AI 提供商)**

### 交接备忘

#### 第4章必须包含的内容

1. **LlmClient trait 设计** — 统一的 LLM 接口
   - `complete()` 和 `complete_streaming()` 两个核心方法
   - 为什么需要抽象层(支持多 Provider、可替换、可测试)
   - trait 的设计权衡(async trait、泛型 vs trait object)

2. **LlmRequest 和 LlmResponse** — 标准化的请求/响应
   - 如何映射到不同 Provider 的 API(Anthropic vs OpenAI 格式差异)
   - 流式响应 vs 非流式响应的实现差异
   - SSE(Server-Sent Events)协议解析

3. **LlmEvent 和 LlmStream** — 流式响应的事件模型
   - TextDelta、ToolCallDelta、Completed、Error 四种事件
   - 如何将 SSE 流解析为 LlmEvent
   - 为什么用 Stream trait 而非 async iterator

4. **错误处理** — LlmError 类型层次
   - Http、Api、RateLimit、Timeout 等错误类型
   - 哪些错误可重试(RetryableError trait)
   - 错误传播和转换(From trait)

5. **实现示例** — AnthropicLlmClient(或 OpenAiLlmClient)
   - 如何将 LlmRequest 转换为 Anthropic API 请求
   - 如何解析 Anthropic SSE 响应为 LlmEvent
   - 请求/响应的完整数据流

#### 写作时注意

- 本章的目标是"**建立抽象层理解**"— 让读者理解如何通过 trait 统一多个 Provider
- 重点讲"为什么需要抽象"和"如何设计抽象"(不只是 API 细节)
- 用具体例子说明不同 Provider 的差异(Anthropic vs OpenAI 的 API 格式)
- SSE 解析是难点,需要详细讲解(如何处理 `data: ` 前缀、事件拼接等)
- 与 ch02/ch03 衔接(LlmRequest/LlmResponse 在状态机中的使用)

#### 质检重点

- [ ] LlmClient trait 设计是否讲清楚了(为什么这样抽象)
- [ ] 不同 Provider 的 API 差异是否有具体例子
- [ ] SSE 解析流程是否清晰(有示例数据和解析步骤)
- [ ] 错误处理是否完整(错误类型、重试逻辑)
- [ ] 代码片段是否不超过 5 处(trait 定义、请求/响应转换、SSE 解析示例)
- [ ] 是否与 ch02/ch03 衔接(引用 LlmRequest/LlmResponse/LlmEvent)

### 待验证项

- 查看 `specs/llm-client.md` 获取 LLM 抽象层设计文档
- 查看 `crates/loom-common-core/src/llm.rs` 获取 LlmClient trait 和 LlmEvent 定义
- 查看 `crates/loom-llm-anthropic/` 或 `crates/loom-llm-openai/` 获取具体实现示例
- 查看 SSE 解析相关代码(可能在 anthropic/openai crate 中)
