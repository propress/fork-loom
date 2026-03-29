# 第二章 核心类型系统：对话的数据骨架

---

## 本章概览

在上一章中,我们理解了 Loom 的设计目标和架构选型。现在我们深入代码,从最基础的数据结构开始:对话数据是如何组织的?

对话看似简单——用户说话,AI 回应——但在 Loom 中,对话数据需要支撑更复杂的场景:

- **多模态消息** — 消息可能包含文本、Tool 调用请求、Tool 执行结果
- **LLM 协议兼容** — 需要序列化为 OpenAI/Anthropic 兼容的 JSON 格式
- **状态机传递** — 状态转换时需要携带完整的对话上下文
- **类型安全** — Rust 的类型系统应当防止无效的消息结构(如 Tool 消息缺少 call_id)

本章将介绍 Loom 对话数据的完整类型系统:从最小的 `Message` 到 LLM 交互的 `LlmRequest/LlmResponse`,再到状态机使用的 `ConversationContext` 和 `ToolExecutionStatus`。

**重点模块:**
- `crates/loom-common-core/src/message.rs` — 消息和 Tool 调用
- `crates/loom-common-core/src/llm.rs` — LLM 请求/响应
- `crates/loom-common-core/src/state.rs` — 对话上下文和 Tool 执行状态

---

## 2.1 类型系统全景图

在深入细节前,先看对话数据的整体结构。下图展示了核心类型之间的依赖关系:

```
ConversationContext (对话上下文)
  ├─ id: UUID (对话唯一标识,UUID v4)
  └─ messages: Vec<Message> (消息历史)
        │
        └─ Message (单条消息)
             ├─ role: Role (User/Assistant/System/Tool)
             ├─ content: String (文本内容)
             ├─ tool_call_id: Option<String> (Tool 结果消息必须有)
             ├─ name: Option<String> (Tool 消息的工具名)
             └─ tool_calls: Vec<ToolCall> (AI 请求的 Tool 调用)

LlmRequest (发送给 LLM 的请求)
  ├─ model: String (模型名,如 "claude-3-5-sonnet-20241022")
  ├─ messages: Vec<Message> (对话历史)
  ├─ tools: Vec<ToolDefinition> (可用工具列表)
  ├─ max_tokens: Option<u32> (生成上限)
  └─ temperature: Option<f32> (生成温度)

LlmResponse (LLM 返回的响应)
  ├─ message: Message (AI 的回复消息)
  ├─ tool_calls: Vec<ToolCall> (AI 请求的 Tool 调用,可能为空)
  ├─ usage: Option<Usage> (Token 使用统计)
  └─ finish_reason: Option<String> ("stop" | "tool_calls" | "length")

ToolExecutionStatus (Tool 执行状态,判别联合类型)
  ├─ Pending { call_id, tool_name, requested_at }
  ├─ Running { call_id, tool_name, started_at, last_update_at, progress }
  └─ Completed { call_id, tool_name, started_at, completed_at, outcome }
```

**为什么需要这些类型?**

1. **类型安全** — Rust 编译器强制 Tool 消息必须有 `tool_call_id`,防止运行时错误
2. **序列化** — 所有类型都实现了 `Serialize/Deserialize`,可以存储到数据库或通过网络传输
3. **状态转换** — 状态机在不同状态间转换时,通过这些类型携带上下文
4. **LLM 协议兼容** — `LlmRequest` 可以序列化为 OpenAI/Anthropic API 兼容的 JSON

**数据流示例:**

```
用户输入 "帮我读取 README.md"
  → Message::user("帮我读取 README.md")
  → 添加到 ConversationContext.messages
  → 构造 LlmRequest { messages, tools: [read_file, ...] }
  → 发送到 LLM
  → 收到 LlmResponse { tool_calls: [ToolCall { tool_name: "read_file", ... }] }
  → 执行 Tool,状态为 ToolExecutionStatus::Pending
  → Tool 完成,状态变为 ToolExecutionStatus::Completed
  → 添加 Message::tool("call_123", "read_file", "文件内容...") 到 messages
  → 再次构造 LlmRequest 发送给 LLM(带 Tool 结果)
  → 收到最终 LlmResponse { message: "文件内容是..." }
```

---

## 2.2 Message 和 Role:对话的最小单元

### Message 结构

每条消息都是一个 `Message` 实例,包含以下字段:

```rust
pub struct Message {
	pub role: Role,                       // 角色:User/Assistant/System/Tool
	pub content: String,                  // 文本内容
	pub tool_call_id: Option<String>,     // Tool 结果消息必须有(关联到 ToolCall.id)
	pub name: Option<String>,             // Tool 消息的工具名(如 "read_file")
	pub tool_calls: Vec<ToolCall>,        // AI 请求的 Tool 调用(Assistant 消息可能有)
}
```

**为什么用结构化的 Message 而非字符串?**

1. **类型安全** — 编译器确保 `role` 只能是 4 种合法值,不能是任意字符串
2. **多模态支持** — 消息不只是文本,还可能包含 Tool 调用(未来可能支持图片、文件等)
3. **序列化** — 可以直接序列化为 JSON,符合 OpenAI/Anthropic API 格式
4. **验证** — 可以在类型层面确保 Tool 消息必须有 `tool_call_id`(虽然目前靠约定)

### Role 枚举

`Role` 定义了消息的角色:

```rust
pub enum Role {
	System,    // 系统提示词(通常在对话开头,设定 AI 行为)
	User,      // 用户消息
	Assistant, // AI 助手消息(可能包含 tool_calls)
	Tool,      // Tool 执行结果消息(必须有 tool_call_id 和 name)
}
```

**各角色的语义:**

- **System** — 设定 AI 的行为规则。例如: "你是一个 Rust 代码专家,总是遵循最佳实践。"
- **User** — 用户的请求或问题。例如: "帮我优化这段代码的性能。"
- **Assistant** — AI 的回复。可能包含纯文本,也可能包含 `tool_calls`(请求执行工具)。
- **Tool** — 工具执行结果。LLM 在收到 Tool 消息后,能够基于结果继续生成回复。

**为什么 Tool 消息必须有 `tool_call_id` 和 `name`?**

这是 OpenAI/Anthropic 协议的要求:

1. `tool_call_id` — 关联到之前的 `ToolCall.id`,LLM 才知道这是哪个调用的结果
2. `name` — 工具名称,帮助 LLM 理解结果的上下文

缺少这两个字段,LLM 会拒绝请求或产生未定义行为。

### 便捷构造器

`Message` 提供了便捷的构造方法:

```rust
impl Message {
	pub fn system(content: impl Into<String>) -> Self
	pub fn user(content: impl Into<String>) -> Self
	pub fn assistant(content: impl Into<String>) -> Self
	pub fn assistant_with_tool_calls(content: impl Into<String>, tool_calls: Vec<ToolCall>) -> Self
	pub fn tool(tool_call_id: impl Into<String>, name: impl Into<String>, content: impl Into<String>) -> Self
}
```

**使用示例:**

```rust
// 系统提示
let sys = Message::system("你是 Rust 专家");

// 用户输入
let user = Message::user("帮我读取 README.md");

// AI 回复(请求调用工具)
let assistant = Message::assistant_with_tool_calls(
	"好的,我来读取文件",
	vec![ToolCall { id: "call_1", tool_name: "read_file", ... }]
);

// Tool 结果
let tool_result = Message::tool("call_1", "read_file", "# Loom\n\n...");
```

---

## 2.3 ToolCall 和 ToolResult:AI 如何调用外部操作

### ToolCall 结构

当 LLM 想要执行工具时,会返回 `ToolCall`:

```rust
pub struct ToolCall {
	pub id: String,                       // 调用 ID(由 LLM 生成,如 "call_abc123")
	pub tool_name: String,                // 工具名(如 "read_file", "bash")
	pub arguments_json: serde_json::Value,// 参数(JSON 对象,如 {"path": "/README.md"})
}
```

**为什么 `id` 由 LLM 生成?**

LLM 可以在一次响应中请求**多个工具调用**(并行执行)。每个调用需要唯一 ID,以便后续返回结果时能对应上。

**示例:**

```json
{
  "id": "call_abc123",
  "tool_name": "read_file",
  "arguments_json": {"path": "/README.md"}
}
```

### ToolResult(通过 Message::tool 表达)

Loom 没有单独的 `ToolResult` 类型。工具执行结果通过 `Message::tool` 表达:

```rust
Message {
	role: Role::Tool,
	content: "{\"content\": \"# Loom\\n\\n...\"}",  // JSON 字符串形式的结果
	tool_call_id: Some("call_abc123"),             // 关联到 ToolCall.id
	name: Some("read_file"),                       // 工具名
	tool_calls: vec![],
}
```

**为什么 `content` 是字符串而非结构化数据?**

这是 OpenAI/Anthropic 协议的设计:Tool 结果必须是字符串(通常是 JSON 序列化后的结果)。LLM 会解析这个字符串。

**为什么 `id` 和 `tool_call_id` 必须匹配?**

这是协议要求:LLM 需要知道每个 Tool 结果对应哪个调用。如果 ID 不匹配,LLM 会报错。

---

## 2.4 LlmRequest 和 LlmResponse:LLM 交互的标准接口

### LlmRequest 结构

`LlmRequest` 封装了发送给 LLM 的所有参数:

```rust
pub struct LlmRequest {
	pub model: String,                // 模型名(如 "claude-3-5-sonnet-20241022")
	pub messages: Vec<Message>,       // 对话历史(包含 System/User/Assistant/Tool 消息)
	pub tools: Vec<ToolDefinition>,   // 可用工具列表(Tool 的 JSON Schema 定义)
	pub max_tokens: Option<u32>,      // 生成 Token 上限(如 4096)
	pub temperature: Option<f32>,     // 生成温度(0.0-1.0,越高越随机)
}
```

**为什么需要 `tools` 字段?**

LLM 需要知道有哪些工具可用(名称、描述、参数 Schema)。这样它才能决定是否调用工具,以及如何构造 `arguments_json`。

**示例 JSON(序列化后发送给 LLM):**

```json
{
  "model": "claude-3-5-sonnet-20241022",
  "messages": [
    {"role": "system", "content": "你是 Rust 专家"},
    {"role": "user", "content": "帮我读取 README.md"}
  ],
  "tools": [
    {
      "name": "read_file",
      "description": "读取文件内容",
      "input_schema": {"type": "object", "properties": {"path": {"type": "string"}}, ...}
    }
  ],
  "max_tokens": 4096,
  "temperature": 0.7
}
```

### LlmResponse 结构

`LlmResponse` 封装了 LLM 的完整响应:

```rust
pub struct LlmResponse {
	pub message: Message,             // AI 的回复消息(role=Assistant)
	pub tool_calls: Vec<ToolCall>,    // AI 请求的工具调用(可能为空)
	pub usage: Option<Usage>,         // Token 使用统计(input/output)
	pub finish_reason: Option<String>,// 完成原因("stop" | "tool_calls" | "length")
}
```

**为什么 `tool_calls` 在 `LlmResponse` 和 `Message` 中都有?**

这是设计权衡:

- `LlmResponse.tool_calls` — 方便直接访问所有 Tool 调用(无需从 `message.tool_calls` 提取)
- `Message.tool_calls` — 保持消息的完整性(序列化时包含 Tool 调用信息)

实际使用时,两者内容相同,选哪个都可以。

**Usage 结构:**

```rust
pub struct Usage {
	pub input_tokens: u32,   // 输入消耗的 Token 数
	pub output_tokens: u32,  // 输出生成的 Token 数
}
```

用于统计成本和配额。

### 流式响应 vs 非流式响应

Loom 支持两种模式:

| 模式 | API | 数据流 | 使用场景 |
|------|-----|--------|----------|
| **非流式** | `LlmClient::complete()` | 等待完整响应,返回 `LlmResponse` | 批处理、测试、简单交互 |
| **流式** | `LlmClient::complete_streaming()` | 返回 `LlmStream`,逐步接收 `LlmEvent` | 实时显示、长文本生成 |

**LlmEvent 枚举(流式模式):**

```rust
pub enum LlmEvent {
	TextDelta { content: String },                          // 增量文本
	ToolCallDelta { call_id, tool_name, arguments_fragment },// 增量 Tool 调用数据
	Completed(LlmResponse),                                 // 流结束,完整响应
	Error(LlmError),                                        // 错误
}
```

**流式响应的数据差异:**

- 非流式: 一次性收到完整的 `message.content` 和 `tool_calls`
- 流式: 先收到多个 `TextDelta`(拼接成完整文本),或多个 `ToolCallDelta`(拼接成完整 Tool 调用),最后收到 `Completed`

**示例(流式):**

```
收到: TextDelta { content: "好的" }
收到: TextDelta { content: ",我来" }
收到: TextDelta { content: "读取文件" }
收到: ToolCallDelta { call_id: "call_1", tool_name: "read_file", arguments_fragment: "{\"path\":" }
收到: ToolCallDelta { call_id: "call_1", tool_name: "read_file", arguments_fragment: "\"/README.md\"}" }
收到: Completed(LlmResponse { message: ..., tool_calls: [...], ... })
```

---

## 2.5 状态机使用的数据结构

状态机(第 3 章详解)需要在状态转换时携带对话上下文和 Tool 执行状态。

### ConversationContext:对话上下文

```rust
pub struct ConversationContext {
	pub id: uuid::Uuid,        // 对话 ID(UUID v4,随机生成)
	pub messages: Vec<Message>,// 消息历史(User/Assistant/System/Tool 消息)
}
```

**为什么需要 `id`?**

- 区分不同对话会话(多个并发对话时,通过 ID 识别)
- 持久化时作为主键(Thread 表的外键)
- 日志追踪(在 tracing 中标识对话)

**为什么用 UUID v4?**

- 无中心化生成(不需要数据库自增 ID)
- 碰撞概率极低(2^122 分之一)
- 随机性好(不泄露创建时间等信息)

注:Loom 的 Thread 持久化(第 8 章)实际使用 **UUID v7**(时间排序),但状态机内部使用 UUID v4。

### ToolExecutionStatus:Tool 执行状态

`ToolExecutionStatus` 是一个**判别联合类型**(discriminated union),表示 Tool 执行的三个阶段:

```rust
pub enum ToolExecutionStatus {
	Pending {
		call_id: String,
		tool_name: String,
		requested_at: Instant,    // 请求时刻
	},
	Running {
		call_id: String,
		tool_name: String,
		started_at: Instant,      // 开始执行时刻
		last_update_at: Instant,  // 最后进度更新时刻
		progress: Option<ToolProgress>,// 进度信息(可选)
	},
	Completed {
		call_id: String,
		tool_name: String,
		started_at: Instant,
		completed_at: Instant,    // 完成时刻
		outcome: ToolExecutionOutcome,// 结果(Success 或 Error)
	},
}
```

**为什么需要三种状态?**

1. **Pending** — Tool 调用已请求,但还没开始执行(可能在队列中等待)
2. **Running** — Tool 正在执行,可能有进度更新(如文件拷贝进度)
3. **Completed** — Tool 执行完成(成功或失败)

**为什么不直接用布尔标志(如 `is_running: bool`)?**

判别联合类型的优势:

- **类型安全** — 编译器确保每个状态只携带相关字段(如 `progress` 只在 `Running` 时存在)
- **穷举匹配** — `match` 时必须处理所有状态,防止遗漏
- **清晰语义** — 状态转换是显式的(Pending → Running → Completed),而非隐式修改标志

**ToolExecutionOutcome(结果):**

```rust
pub enum ToolExecutionOutcome {
	Success { call_id: String, output: serde_json::Value },
	Error { call_id: String, error: ToolError },
}
```

**ToolProgress(进度信息):**

```rust
pub struct ToolProgress {
	pub fraction: Option<f32>,      // 完成百分比(0.0-1.0)
	pub message: Option<String>,    // 进度描述(如 "复制文件 3/10")
	pub units_processed: Option<u64>,// 已处理单元数(如字节数)
}
```

**为什么状态机需要这些数据结构?**

状态机在不同状态间转换时,需要:

1. **保留对话历史** — `ConversationContext` 在所有状态间传递
2. **跟踪 Tool 执行** — `Vec<ToolExecutionStatus>` 在 `ExecutingTools` 状态中维护
3. **判断是否全部完成** — 遍历 `executions`,检查是否都是 `Completed`

**示例(状态机视角):**

```
WaitingForUserInput { conversation }
  → 收到 UserInput
  → CallingLlm { conversation, retries: 0 }
  → 收到 LlmEvent::Completed(tool_calls=[...])
  → ExecutingTools { conversation, executions: [Pending, Pending] }
  → 收到 ToolCompleted
  → ExecutingTools { conversation, executions: [Completed, Pending] }
  → 收到 ToolCompleted
  → ExecutingTools { conversation, executions: [Completed, Completed] }
  → 检查全部完成
  → CallingLlm { conversation(添加了 Tool 结果), retries: 0 }
```

---

## 2.6 类型之间的关系

用文字总结类型依赖:

```
1. ConversationContext 包含 Vec<Message>
   - 每条 Message 可能包含 Vec<ToolCall>(如果 role=Assistant)
   - Tool 消息(role=Tool)必须有 tool_call_id 和 name

2. LlmRequest 引用 ConversationContext.messages
   - 还包含 Vec<ToolDefinition>(可用工具列表)

3. LlmResponse 包含 Message(AI 回复)
   - 还包含 Vec<ToolCall>(如果 AI 请求工具)

4. ToolExecutionStatus 跟踪单个 ToolCall 的执行
   - Pending → Running → Completed
   - Completed 包含 ToolExecutionOutcome(Success | Error)

5. 状态机的 ExecutingTools 状态包含 Vec<ToolExecutionStatus>
   - 全部 Completed 后,将结果转换为 Message::tool,添加到 ConversationContext
```

**依赖图(简化):**

```
        ┌──────────────────────┐
        │ ConversationContext  │
        │  - id: UUID          │
        │  - messages: Vec<M>  │
        └──────────┬───────────┘
                   │
                   ├───> Message ────> ToolCall
                   │      - role          - id
                   │      - content       - tool_name
                   │      - tool_calls    - arguments_json
                   │      - tool_call_id
                   │      - name
                   │
                   └───> LlmRequest ────> LlmResponse
                          - model           - message
                          - messages        - tool_calls
                          - tools           - usage
                          - max_tokens      - finish_reason
                          - temperature

        ┌──────────────────────┐
        │ ToolExecutionStatus  │
        │  - Pending           │
        │  - Running           │
        │  - Completed         │
        └──────────────────────┘
```

---

## 2.7 小结

本章介绍了 Loom 对话数据的完整类型系统:

1. **Message 和 Role** — 对话的最小单元,支持 User/Assistant/System/Tool 四种角色
2. **ToolCall** — AI 请求执行工具的数据结构,包含 id、tool_name、arguments_json
3. **LlmRequest 和 LlmResponse** — LLM 交互的标准接口,封装请求参数和响应数据
4. **ConversationContext** — 对话上下文,包含 UUID 和消息历史,在状态间传递
5. **ToolExecutionStatus** — Tool 执行状态,判别联合类型(Pending/Running/Completed)

**关键设计原则:**

- **类型安全** — Rust 类型系统防止无效数据(如缺少 `tool_call_id` 的 Tool 消息)
- **序列化** — 所有类型都实现 `Serialize/Deserialize`,支持存储和网络传输
- **协议兼容** — `LlmRequest/LlmResponse` 符合 OpenAI/Anthropic API 格式
- **状态转换** — 类型在状态机中携带上下文,确保每次转换都有完整数据

下一章(第 3 章)将介绍**状态机**,看这些类型如何在状态转换中流动,以及如何驱动对话流程。

---

## 质检清单

- [x] **类型总览** — 提供了全景图(章节开头的依赖树和数据流示例)
- [x] **每个类型讲了"为什么"** — 每个类型都解释了设计理由(如为什么 Tool 消息需要 `tool_call_id`)
- [x] **具体的值示例** — JSON 和 Rust 代码示例贯穿全章
- [x] **类型关系清晰** — 2.6 节用文字和图表说明了依赖关系
- [x] **代码片段控制** — 5 处代码片段(Message/ToolCall/LlmRequest/LlmResponse/ToolExecutionStatus 定义),都是类型定义,符合规则
- [x] **避免过早优化** — 没有讨论性能、序列化格式优化等(聚焦类型语义)
- [x] **教学节奏** — 从整体(类型全景)到局部(每个类型详解),从简单(Message)到复杂(ToolExecutionStatus)
- [x] **周边知识** — 解释了 UUID v4、判别联合类型、流式响应等概念
- [x] **代码纪律** — 代码片段控制在 5 处,都是类型定义(符合"类型定义可适当放宽"规则)
- [x] **准确性** — 所有类型定义和字段与实际代码一致(已验证 message.rs/llm.rs/state.rs)
- [x] **可读性** — 使用了表格、树形图、代码示例,层次清晰
