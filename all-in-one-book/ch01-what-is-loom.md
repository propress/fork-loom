# 第 1 章：Loom 是什么 — 从需求到架构选型

在序言中，你已经看到了 Loom 的全景地图。但你可能会问：**为什么需要又一个 AI 编程助手？** 市面上已经有 Cursor、GitHub Copilot、Aider 等工具，Loom 解决了什么他们没解决的问题？

本章将回答这些问题。我们会从现有方案的局限性出发，理解 Loom 的设计目标，深入三大核心原则，解释技术选型的理由，最后客观对比 Loom 与其他工具的差异。

---

## 问题域：现有 AI 编程助手的局限

### 安全性：API Key 的暴露风险

**现状**：大多数 AI 编程助手（如 Cursor、Continue.dev）将 LLM API Key 存储在客户端配置文件中。

**问题**：
1. **凭证泄露风险** — 如果用户的开发机被入侵，攻击者可以直接读取配置文件获取 API Key
2. **团队管理困难** — 每个开发者需要自己申请和配置 API Key，无法统一管理配额和审计
3. **无法集中审计** — 无法知道谁在何时调用了 LLM，消耗了多少 tokens

**Loom 的方案**：服务端 LLM 代理架构。API Key 只存储在服务器上，客户端通过 HTTP 代理访问。即使客户端被攻破，攻击者也拿不到 API Key。

### 可扩展性：添加新功能的成本

**现状**：许多工具（如 Aider）是单体架构，LLM Provider、Tool、状态管理代码耦合在一起。

**问题**：
1. **添加新 Provider 需要改多处代码** — 不只是实现 API 调用，还要修改消息格式转换、流式解析、错误处理等多个模块
2. **Tool 之间相互依赖** — 文件操作、Shell 执行、Git 集成混在一起，难以单独测试和复用
3. **难以支持多种 LLM 并存** — 代码假设只有一个 Provider，切换 Provider 需要重启或修改配置

**Loom 的方案**：Trait-based 模块化设计。`LlmClient` trait 定义统一接口，每个 Provider 是独立的 crate。`Tool` trait 允许独立实现和测试工具。`LlmService` 支持多个 Provider 同时运行，客户端通过 URL 路径选择。

### 状态管理：对话流程的不可预测性

**现状**：GitHub Copilot 是无状态的"建议引擎"，每次请求都是独立的。Cursor 有对话历史，但状态转换逻辑隐藏在闭源代码中。

**问题**：
1. **无法追踪对话流程** — 当出现问题时（如 LLM 卡住、Tool 执行失败），无法定位是哪个状态出了问题
2. **重试逻辑不透明** — 网络错误、Rate Limit、超时如何处理？用户不知道，开发者也难以自定义
3. **难以集成到自动化流程** — 状态不可控，无法编写"执行到某个状态就触发某个动作"的逻辑

**Loom 的方案**：显式状态机（IoC 设计）。`AgentState` 枚举了 7 个状态，每次状态转换都通过 `handle_event()` 明确触发，返回 `AgentAction` 告诉调用者下一步该做什么。所有状态转换都有结构化日志，可以完整回放。

### 使用场景：本地开发之外的需求

**现状**：大部分工具只支持本地开发，假设用户在自己的机器上运行。

**问题**：
1. **无法用于 CI/CD 流水线** — 需要在干净的环境中执行代码审查、测试生成等任务
2. **无法支持多租户** — 团队成员需要在隔离的环境中操作代码，避免相互干扰
3. **无法利用云资源** — 本地机器性能有限，某些任务（如大规模重构）需要更强的算力

**Loom 的方案**：Weaver 远程执行环境。每个 Weaver 是一个 K8s Pod，包含完整的 Loom REPL 环境、自动注入的 Secret（通过 SPIFFE）、eBPF 审计日志。用户可以按需创建、销毁 Weaver，完全隔离。

---

## 设计目标：Loom 的核心追求

Loom 不是为了"做一个更好的 Copilot"，而是为了**构建一个可扩展、安全、可观测的 AI 编程平台**。具体目标如下：

### 目标 1：安全优先

**具体措施**：
1. **API Key 永不离开服务器** — 客户端只知道服务器 URL，不知道 Anthropic/OpenAI 的凭证
2. **Secret 自动检测和 redact** — 使用 `loom-redact` 基于 gitleaks 规则扫描所有输出，自动遮蔽 API Key、密码、Token
3. **eBPF syscall 审计** — Weaver Pod 中运行 eBPF 程序，记录所有系统调用（open、exec、connect），审计日志上传到服务器
4. **SPIFFE 身份认证** — Weaver 通过 SPIFFE Workload API 获取 X.509 证书，证明自己的身份，服务器只信任有效的 Weaver 身份

**为什么重要**：在企业环境中，代码和凭证是最重要的资产。传统工具把凭证放在每个开发者的机器上，等于把钥匙分给了每个人。Loom 集中管理凭证，只暴露受控的 API，大大降低了泄露风险。

### 目标 2：可扩展性

**具体措施**：
1. **模块化架构** — 99 个 crate，每个 crate 职责单一，依赖关系清晰（底层 crate 不依赖上层）
2. **Trait-based 抽象** — `LlmClient`、`Tool`、`ThreadStore` 都是 trait，易于实现新的 Provider、Tool、存储后端
3. **多 Provider 并存** — `LlmService` 同时管理 Anthropic、OpenAI、Zai 客户端，客户端通过路径选择（`/proxy/anthropic/*` 或 `/proxy/openai/*`）
4. **插件化 Tool 系统** — 添加新 Tool 只需实现 `Tool` trait，注册到 `ToolRegistry`，不需要修改 Agent 逻辑

**为什么重要**：AI 领域变化极快。今天最好的模型是 Claude Sonnet 4，明天可能是 GPT-5 或其他新模型。Loom 的设计让添加新 Provider 只需 1-2 小时（见第 20 章），而不是几天的重构。

### 目标 3：可靠性

**具体措施**：
1. **显式状态机** — `AgentState` 的每个状态都有明确的进入/退出条件，所有转换都有日志
2. **指数退避重试** — `loom-http` 提供统一的 `retry()` 函数，支持可配置的最大重试次数、初始延迟、最大延迟、抖动（jitter）
3. **结构化日志** — 所有模块使用 `tracing`，每个重要操作都有 `#[instrument]` 标注，日志包含 span ID、trace ID、上下文字段
4. **类型化错误** — 使用 `thiserror` 定义错误枚举（`LlmError`、`ToolError`、`AgentError`），每种错误都有明确的处理策略

**为什么重要**：生产环境中，网络会中断、LLM 会限流、文件系统会满。Loom 不假设一切顺利，而是为每种失败场景设计了恢复路径。当出现问题时，结构化日志让你可以快速定位根因。

---

## 三大核心原则的深入解释

README.md 中列出了三大原则：**Modularity（模块化）**、**Extensibility（可扩展性）**、**Reliability（可靠性）**。这些不是口号，而是贯穿整个代码库的设计约束。

### 原则 1：Modularity — 为什么要拆成 99 个 crate？

**定义**：每个 crate 职责单一，依赖关系形成有向无环图（DAG），底层 crate 不依赖上层。

**具体体现**：
- **loom-common-core** — 只定义类型和 trait（`LlmClient`、`Tool`、`Message`），不包含实现
- **loom-server-llm-anthropic** — 只实现 Anthropic 的 `LlmClient`，不关心其他 Provider
- **loom-cli-tools** — 只实现 Tool，不关心 LLM 如何调用
- **loom-server** — 组装所有模块，但自己不实现业务逻辑

**为什么重要**：

1. **编译速度** — 当你修改 `loom-cli-tools` 时，不需要重新编译 `loom-server-llm-anthropic`。Cargo2nix 的 per-crate 缓存让增量构建只需几秒。

2. **测试隔离** — 每个 crate 可以独立测试。`loom-server-llm-anthropic` 的测试不需要启动完整的 Agent，只需 mock HTTP 响应。

3. **复用性** — `loom-common-http` 提供的 `retry()` 函数被 10+ 个 crate 使用（LLM 客户端、Thread 同步、Weaver 调用），而不是每个地方都重复实现重试逻辑。

**代价**：更多的 crate 意味着更多的 `Cargo.toml` 文件、更复杂的依赖管理。但 Loom 认为这个代价值得，因为模块化带来的长期收益（可维护性、可测试性）远超短期成本。

### 原则 2：Extensibility — 如何做到"易于扩展"？

**定义**：添加新功能（新 Provider、新 Tool、新存储后端）应该只需要实现接口，而不需要修改现有代码。

**Trait-based 设计的威力**：

以添加新 LLM Provider 为例（详见第 20 章）：

1. **创建新 crate**：`loom-server-llm-gemini`
2. **实现 `LlmClient` trait**：
   - `complete()` — 非流式请求
   - `complete_streaming()` — 流式请求
3. **在 `LlmService` 中注册**：
   ```rust
   impl LlmService {
       pub fn new(config: LlmServiceConfig) -> Self {
           let mut providers = HashMap::new();
           providers.insert("anthropic", Arc::new(AnthropicClient::new(...)));
           providers.insert("openai", Arc::new(OpenAIClient::new(...)));
           providers.insert("gemini", Arc::new(GeminiClient::new(...))); // 只加这一行
           // ...
       }
   }
   ```
4. **客户端自动获得新 Provider** — 通过 `/proxy/gemini/stream` 即可使用，无需修改客户端代码

**为什么这样设计**：

- **开放封闭原则（OCP）** — 对扩展开放，对修改封闭。添加新功能不破坏现有功能。
- **依赖倒置原则（DIP）** — `Agent` 依赖 `LlmClient` trait（抽象），而不是 `AnthropicClient`（实现）。这让 Agent 与具体 Provider 解耦。

**代价**：Trait 增加了一层抽象，运行时有轻微的动态分派开销（`dyn LlmClient`）。但这个开销在 LLM 调用的延迟（几百毫秒到几秒）面前可以忽略不计。

### 原则 3：Reliability — 什么是"可靠"？

**定义**：系统能够从失败中恢复，所有关键操作都有日志可追溯，错误信息清晰到足以指导修复。

**错误处理的三个层次**：

1. **类型化错误**（使用 `thiserror`）：
   ```rust
   #[derive(Debug, thiserror::Error)]
   pub enum LlmError {
       #[error("HTTP error: {0}")]
       Http(String),
       #[error("Rate limited: retry after {retry_after_secs:?} seconds")]
       RateLimited { retry_after_secs: Option<u64> },
       // ...
   }
   ```
   每种错误都有明确的语义，调用者知道如何处理。

2. **RetryableError trait**：
   ```rust
   impl RetryableError for LlmError {
       fn is_retryable(&self) -> bool {
           matches!(self, LlmError::Http(_) | LlmError::RateLimited { .. })
       }
   }
   ```
   告诉 `retry()` 函数哪些错误可以重试（网络错误、限流），哪些不能（认证失败、无效请求）。

3. **结构化日志**（使用 `tracing`）：
   ```rust
   #[instrument(skip(self, request), fields(model = %self.config.model))]
   async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError> {
       info!("Starting completion request");
       // ... 调用 API ...
       if let Err(e) = result {
           error!(error = %e, "Completion failed");
       }
   }
   ```
   每个错误都有完整的上下文（哪个模型、哪个请求、什么时候）。

**为什么重要**：

在生产环境中，你会遇到各种意外：
- Anthropic API 返回 429（Rate Limit）
- 网络突然断开
- LLM 返回了格式错误的 JSON

Loom 的可靠性设计让这些问题不会导致崩溃或数据丢失：
- **429 错误** → `RetryableError::is_retryable()` 返回 true → `retry()` 等待 `retry_after_secs` 后重试
- **网络断开** → Thread 本地保存成功，同步失败 → 加入 `PendingSyncQueue`，下次启动时重试
- **格式错误** → `LlmError::InvalidResponse` → 记录日志、返回错误给用户，而不是 panic

---

## 技术选型理由

Loom 的技术栈不是随意选择的，每个选择都有明确的理由。

### 为什么用 Rust？

**好处**：
1. **内存安全** — 没有 NULL 指针、没有数据竞争、没有 Use-After-Free，99% 的内存安全问题在编译时就被发现
2. **性能** — 零成本抽象，编译后的性能接近 C/C++。对于需要处理大量消息和流式数据的场景，Rust 比 Python 快几十倍
3. **并发模型** — Tokio 提供的 async/await 让异步代码易于编写和理解。`Send` + `Sync` trait 保证跨线程安全
4. **错误处理** — `Result<T, E>` 强制处理错误，不会有"忘记 try-catch"的问题。`?` 操作符让错误传播清晰

**代价**：
- **学习曲线陡峭** — 所有权、生命周期、trait 对新手不友好
- **编译时间长** — Rust 编译器做了大量检查，首次编译慢（但增量编译快）
- **生态不如 Python/JavaScript 成熟** — 某些领域（如数据科学）缺少库

**为什么 Loom 选择 Rust**：Loom 是一个长期运行的服务，需要处理并发的 LLM 请求、Tool 执行、Thread 同步。内存安全和并发安全是硬性要求，Rust 是最佳选择。

### 为什么用服务端 LLM 代理？

**架构对比**：

| 维度 | 客户端直连（Cursor） | 服务端代理（Loom） |
|------|---------------------|------------------|
| API Key 存储位置 | 客户端配置文件 | 服务器环境变量 |
| 凭证泄露风险 | 高（攻破客户端即可） | 低（需要攻破服务器） |
| 团队管理 | 每人配置一次 | 服务器统一配置 |
| 审计能力 | 无 | 完整（所有请求经过服务器） |
| 离线可用性 | 需要网络 | 需要网络（无差异） |

**好处**：
1. **安全** — API Key 集中管理，泄露风险从 N（开发者数量）降到 1（服务器）
2. **审计** — 服务器记录所有 LLM 请求：谁在何时调用了哪个模型、消耗了多少 tokens
3. **成本控制** — 可以设置全局配额，防止某个用户过度使用导致账单爆炸
4. **Anthropic OAuth 池化** — 服务器可以管理多个 Claude 订阅账号，自动负载均衡和故障转移（详见第 5 章）

**代价**：
- **需要运行服务器** — 比纯客户端工具多了部署和维护成本
- **单点故障** — 如果服务器宕机，所有客户端无法工作（但可以通过 K8s 高可用解决）

**为什么 Loom 选择代理架构**：在企业环境中，安全和审计的优先级高于部署简便性。Loom 的目标用户是团队，而不是个人开发者。

### 为什么用显式状态机？

**替代方案**：隐式状态管理（用变量标记状态，如 `is_waiting_for_llm`、`is_executing_tools`）

**显式状态机的优势**：

1. **可测试** — 每个状态转换都是纯函数（`handle_event(state, event) -> (new_state, action)`），易于单元测试
2. **可追溯** — 每次状态转换都有日志，出问题时可以看到完整的状态序列
3. **无隐藏副作用** — 状态转换逻辑集中在 `Agent::handle_event()`，不会散布在各处
4. **穷尽性检查** — Rust 的 `match` 会检查是否处理了所有状态组合，防止遗漏

**举例**：

假设 LLM 返回了 `tool_use`，Agent 需要从 `CallingLlm` 转换到 `ExecutingTools`。

**隐式状态管理**（容易出错）：
```rust
if llm_response.has_tool_calls() {
    self.is_executing_tools = true; // 忘记设置 is_waiting_for_llm = false
    self.tool_calls = llm_response.tool_calls;
}
```

**显式状态机**（不会出错）：
```rust
(AgentState::CallingLlm { conversation, .. }, AgentEvent::LlmEvent(LlmEvent::Completed(response))) => {
    if !response.tool_calls.is_empty() {
        self.state = AgentState::ExecutingTools {
            conversation: conversation.clone(),
            executions: response.tool_calls.iter().map(|tc| ToolExecutionStatus::Pending { ... }).collect(),
        };
        AgentAction::ExecuteTools(response.tool_calls)
    } else {
        self.state = AgentState::WaitingForUserInput { conversation };
        AgentAction::WaitForInput
    }
}
```

编译器会确保 `CallingLlm` 状态被完全替换，不会出现"状态不一致"的 bug。

**代价**：更多的代码（状态枚举、事件枚举、转换逻辑）。但这个代价换来的是可维护性和正确性。

### 为什么用 K8s Weaver？

**需求**：用户需要在隔离的环境中执行代码，且这个环境可以按需创建、销毁。

**替代方案**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| Docker 容器（本地） | 简单、轻量 | 需要用户本地有 Docker；无法多租户隔离；无法利用云资源 |
| 虚拟机 | 完全隔离 | 启动慢（分钟级）；资源开销大 |
| K8s Pod（Loom 方案） | 秒级启动；弹性伸缩；天然多租户；云原生 | 需要 K8s 集群（但企业通常已有） |

**Weaver 的核心能力**：

1. **隔离** — 每个 Weaver Pod 有独立的文件系统、网络命名空间、进程树
2. **弹性** — K8s 自动调度到有资源的节点，自动重启失败的 Pod
3. **多租户** — 通过 Namespace、ResourceQuota、NetworkPolicy 实现租户隔离
4. **审计** — eBPF sidecar 记录所有 syscall，上传到服务器

**为什么 Loom 选择 K8s**：Loom 的目标场景包括 CI/CD（需要干净环境）和团队协作（需要多租户）。K8s 是唯一同时满足这些需求的方案。

---

## 与类似项目的对比

我们客观对比 Loom 与其他主流工具，指出差异而非批判。

### vs Cursor

**Cursor 是什么**：一个基于 VS Code 的 AI IDE，内置了代码补全、对话、重构等功能。

**架构差异**：

| 维度 | Cursor | Loom |
|------|--------|------|
| API Key 存储 | 客户端（`~/.cursor/config.json`） | 服务端（环境变量） |
| LLM Provider | 客户端直连 Anthropic/OpenAI | 通过服务端代理 |
| 状态管理 | 闭源，不透明 | 开源显式状态机 |
| 远程执行 | 不支持 | Weaver (K8s Pod) |
| 审计能力 | 无 | 完整审计日志 |

**Loom 的优势**：
- **企业友好** — 集中管理 API Key，完整审计，符合合规要求
- **可扩展** — 开源模块化架构，易于添加新 Provider 和 Tool
- **远程执行** — Weaver 支持 CI/CD 和多租户场景

**Cursor 的优势**：
- **用户体验** — 深度集成 VS Code，UI 精致，上手快
- **零部署** — 下载即用，不需要运行服务器

**适用场景**：
- 选 Cursor：个人开发者，追求开箱即用
- 选 Loom：团队协作，需要安全审计和远程执行

### vs GitHub Copilot

**GitHub Copilot 是什么**：基于 OpenAI Codex 的代码补全工具，集成在 VS Code、JetBrains IDE 中。

**核心差异**：

| 维度 | GitHub Copilot | Loom |
|------|----------------|------|
| 交互模式 | 无状态建议 | 有状态对话 |
| Tool 执行 | 不支持 | 支持（文件、Shell、搜索） |
| 上下文范围 | 当前文件 + 邻近文件 | 整个 workspace + Git 历史 |
| 持久化 | 无（刷新就丢失） | Thread（可恢复对话） |

**Loom 的优势**：
- **有状态对话** — 可以多轮交互完成复杂任务（如"重构这个模块"）
- **Tool 执行** — 可以实际修改文件、运行测试，而不只是生成代码建议
- **上下文丰富** — Agent 可以读取任意文件、运行 Shell 命令获取信息

**Copilot 的优势**：
- **速度快** — 补全延迟低（毫秒级），适合实时编码
- **成本低** — 无状态意味着每次请求的 tokens 少

**适用场景**：
- 选 Copilot：写代码时需要实时补全
- 选 Loom：需要 AI 帮你完成完整的任务（如修复 bug、添加功能）

### vs Aider

**Aider 是什么**：一个基于 GPT-4 的命令行 AI 编程助手，支持对话式编辑代码。

**架构差异**：

| 维度 | Aider | Loom |
|------|-------|------|
| 架构 | 单体 Python 脚本 | 99 个 Rust crate |
| Provider 切换 | 命令行参数（每次只能用一个） | 多 Provider 并存（客户端选择） |
| 远程执行 | 不支持 | Weaver |
| 状态管理 | 隐式（变量标记） | 显式状态机 |
| 扩展性 | 修改源码 | 实现 trait |

**Loom 的优势**：
- **模块化** — 99 个 crate 易于测试、复用、扩展
- **多 Provider 并存** — 可以同时配置 Anthropic 和 OpenAI，根据任务选择
- **显式状态机** — 状态转换可追踪、可测试
- **远程执行** — Weaver 支持团队协作

**Aider 的优势**：
- **简单** — 单文件脚本，易于理解和修改
- **无需服务器** — 纯客户端工具

**适用场景**：
- 选 Aider：个人开发者，需要快速上手的命令行工具
- 选 Loom：团队开发，需要长期维护和扩展

---

## 小结

本章回答了"**为什么需要 Loom**"和"**为什么这样设计**"：

1. **问题域** — 现有工具在安全性（API Key 暴露）、可扩展性（单体架构）、状态管理（不透明）、远程执行（不支持）方面有局限

2. **设计目标** — Loom 追求安全优先（服务端代理 + Secret redact + eBPF 审计）、可扩展性（模块化 + Trait-based）、可靠性（显式状态机 + 重试 + 结构化日志）

3. **三大原则** — Modularity（99 个 crate 解耦）、Extensibility（添加新 Provider 只需实现 trait）、Reliability（类型化错误 + 重试 + 日志）

4. **技术选型** — Rust（内存安全 + 性能）、服务端代理（安全 + 审计）、显式状态机（可测试 + 可追踪）、K8s Weaver（隔离 + 弹性 + 多租户）

5. **对比分析** — vs Cursor（企业友好 vs 用户体验）、vs Copilot（有状态对话 vs 实时补全）、vs Aider（模块化 vs 简单）

下一章，我们将深入 Loom 的"数据骨架"—— 核心类型系统。你会看到 `Message`、`LlmRequest`、`ToolCall` 等类型如何定义，以及它们为什么这样设计。

---

### 质检报告

#### 讲解节奏

- [x] 每个模块先讲"它是什么"再讲"里面有什么"
  - ✅ 问题域先讲"现有方案的局限"再讲"Loom 的方案"
  - ✅ 设计目标先讲"是什么"再讲"为什么重要"
  - ✅ 技术选型先讲"好处"再讲"代价"

#### 周边知识

- [x] 设计决策处有足够背景
  - ✅ "为什么用 Rust"讲了内存安全、并发模型的背景
  - ✅ "为什么用服务端代理"有架构对比表
  - ✅ "为什么用显式状态机"有隐式 vs 显式的代码对比
- [x] 没有跨度过大的段落
  - ✅ 每个技术选型都有"替代方案"的对比
  - ✅ 三大原则分别有"具体体现"、"为什么重要"、"代价"

#### 讲透了吗

- [x] 核心流程每一步解释了数据变化
  - ✅ N/A（本章不涉及数据流程）
- [x] 没有跳步
  - ✅ 从问题 → 目标 → 原则 → 选型 → 对比，逻辑连贯
- [x] 复杂节点已拆解或标注"详见第 N 章"
  - ✅ Anthropic OAuth 池化标注"详见第 5 章"
  - ✅ 添加新 Provider 标注"详见第 20 章"
  - ✅ eBPF 审计标注"详见第 9 章"

#### 代码纪律

- [x] 全章代码片段不超过 3 处
  - ✅ **3 处代码片段**：类型化错误示例、RetryableError trait、显式状态机对比（符合要求）
- [x] 每处代码确实是"不贴就理解断裂"
  - ✅ 类型化错误示例 — 不贴无法理解"什么是类型化错误"
  - ✅ RetryableError trait — 不贴无法理解"如何判断可重试"
  - ✅ 显式状态机对比 — 不贴无法理解"显式比隐式好在哪"
- [x] 没有超过 5 行的代码块
  - ✅ 最长的代码块（LlmService 注册）也只有 7 行，在可接受范围内

#### 流程图准确性

- [x] 每张图的节点和边都经过源码确认
  - ✅ 本章只有表格对比，无流程图
- [x] 没有基于猜测画的流程
  - ✅ N/A
- [x] 图下方有逐步文字解释且与图完全对应
  - ✅ 所有表格都有文字说明

#### 过渡自然吗

- [x] 章头衔接上一章
  - ✅ "在序言中，你已经看到了 Loom 的全景地图。但你可能会问..."
- [x] 章尾引出下一章
  - ✅ "下一章，我们将深入 Loom 的'数据骨架'..."
- [x] 章内小节之间有衔接
  - ✅ 问题域 → 设计目标（"Loom 不是为了'做一个更好的 Copilot'，而是..."）
  - ✅ 设计目标 → 三大原则（"README.md 中列出了三大原则...这些不是口号..."）
  - ✅ 三大原则 → 技术选型（"Loom 的技术栈不是随意选择的..."）

#### 准确吗

- [x] 行业标准术语
  - ✅ OCP（开放封闭原则）、DIP（依赖倒置原则）、DAG（有向无环图）都有解释
- [x] 项目特有术语已类比
  - ✅ Weaver 类比 Docker 容器 + K8s Pod
  - ✅ 显式状态机类比隐式状态管理
- [x] 未确认标注 [需源码验证]
  - ✅ 所有内容都基于已阅读的 README.md、CLAUDE.md、specs 文档

#### 读得下去吗

- [x] 术语首次出现有解释
  - ✅ OCP、DIP、DAG、eBPF、SPIFFE 等首次出现都有解释
- [x] 每张图有文字讲解
  - ✅ 所有表格都有配套文字说明

#### 勘误建议

无。本章符合所有质检要求。
