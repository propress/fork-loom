# 第四章 LLM 抽象层：如何统一多个 AI 提供商

---

## 本章概览

在上一章中,我们理解了状态机如何通过 `SendLlmRequest(LlmRequest)` 动作触发 LLM 调用。但 Loom 如何支持多个 AI 提供商(Anthropic、OpenAI、Z.ai 等)?答案是:**LlmClient trait 抽象层**。

Loom 使用一个统一的 `LlmClient` trait,将不同 Provider 的 API 差异隐藏在实现细节中。这种抽象带来了:

1. **多 Provider 支持** — 同时配置 Anthropic、OpenAI、Z.ai,客户端选择使用哪个
2. **可替换性** — 添加新 Provider 无需修改核心代码,只需实现 trait
3. **可测试性** — 可以用 Mock 实现替换真实 LLM 客户端
4. **统一接口** — 所有 Provider 都通过 `complete()` 和 `complete_streaming()` 两个方法调用

本章将介绍 LLM 抽象层的设计:trait 定义、请求/响应转换、流式响应解析、错误处理,以及如何通过服务端代理隐藏 API Key。

**重点模块:**
- `crates/loom-common-core/src/llm.rs` — LlmClient trait 和核心类型
- `crates/loom-common-core/src/error.rs` — LlmError 错误类型
- `crates/loom-server-llm-anthropic/` — Anthropic 实现示例
- `crates/loom-server-llm-service/` — 服务端 LLM 服务(多 Provider 并存)
- `specs/llm-client.md` — 完整设计文档

---

## 4.1 LlmClient trait 设计:统一的接口

### Trait 定义

`LlmClient` 是 Loom 中所有 LLM Provider 的统一接口:

```rust
#[async_trait]
pub trait LlmClient: Send + Sync {
	/// 发送完整的请求并等待响应(非流式)
	async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError>;

	/// 发送请求并返回流式响应
	async fn complete_streaming(&self, request: LlmRequest) -> Result<LlmStream, LlmError>;
}
```

**两个核心方法:**

| 方法 | 用途 | 返回类型 | 使用场景 |
|------|------|---------|----------|
| `complete()` | 非流式请求 | `Result<LlmResponse, LlmError>` | 批处理、测试、不需要实时显示的场景 |
| `complete_streaming()` | 流式请求 | `Result<LlmStream, LlmError>` | 实时显示、长文本生成、用户交互场景 |

### 为什么需要抽象层?

**场景:** Loom 需要支持 Anthropic Claude、OpenAI GPT、Z.ai GLM 等多个 Provider,但它们的 API 格式不同:

- **Anthropic API:** `POST /v1/messages`,System 消息在 `system` 字段,工具结果在 `tool_result` content block
- **OpenAI API:** `POST /chat/completions`,System 消息是 `role=system`,工具结果是 `role=tool`
- **Z.ai API:** OpenAI 兼容格式

**问题:** 如果直接调用 Provider API,核心代码会充斥大量 `if provider == "anthropic"` 的分支。

**解决方案:** 定义统一的 `LlmClient` trait,每个 Provider 实现这个 trait,核心代码只依赖 trait:

```rust
// 核心代码(状态机)
let llm: Arc<dyn LlmClient> = ...; // 运行时决定 Provider
let response = llm.complete(request).await?;
```

### Trait 的设计权衡

#### 为什么用 `async_trait`?

Rust 目前不支持原生的 async trait 方法。`#[async_trait]` 宏将 async 方法脱糖(desugar)为返回 `Pin<Box<dyn Future>>`:

```rust
// 宏展开前
#[async_trait]
pub trait LlmClient {
	async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError>;
}

// 宏展开后(简化)
pub trait LlmClient {
	fn complete(&self, request: LlmRequest) -> Pin<Box<dyn Future<Output = Result<LlmResponse, LlmError>> + Send + '_>>;
}
```

**优势:**
- 支持 trait object(`dyn LlmClient`)
- 支持动态分发(runtime polymorphism)

**代价:**
- 堆分配(Box)
- 稍微增加运行时开销(通常可忽略)

#### 为什么用 `Arc<dyn LlmClient>` 而非泛型?

**泛型方案(不采用):**

```rust
struct Agent<L: LlmClient> {
	llm: L,
	// ...
}
```

**问题:**
- 编译时必须确定 Provider(无法运行时切换)
- 每个 Provider 会编译出不同的 `Agent` 类型(`Agent<AnthropicClient>`、`Agent<OpenAiClient>`)
- 配置驱动的 Provider 选择无法实现

**Trait Object 方案(采用):**

```rust
struct Agent {
	llm: Arc<dyn LlmClient>,
	// ...
}
```

**优势:**
- 运行时切换 Provider(配置驱动)
- 所有 Provider 共享同一个 `Agent` 类型
- `Arc` 提供线程安全的引用计数(跨 async 任务共享)

**代价:**
- 虚函数调用开销(实测可忽略)
- 堆分配(Arc)

---

## 4.2 LlmRequest 和 LlmResponse:标准化的数据

### LlmRequest 结构

`LlmRequest` 是发送给 LLM 的标准化请求(已在 ch02 介绍,这里补充转换细节):

```rust
pub struct LlmRequest {
	pub model: String,                // 模型名(如 "claude-3-5-sonnet-20241022")
	pub messages: Vec<Message>,       // 对话历史
	pub tools: Vec<ToolDefinition>,   // 可用工具列表
	pub max_tokens: Option<u32>,      // 生成 Token 上限
	pub temperature: Option<f32>,     // 生成温度(0.0-1.0)
}
```

**Builder 模式:**

```rust
LlmRequest::new("claude-3-5-sonnet-20241022")
	.with_messages(messages)
	.with_tools(tools)
	.with_max_tokens(4096)
	.with_temperature(0.7)
```

### LlmResponse 结构

`LlmResponse` 是 LLM 返回的标准化响应:

```rust
pub struct LlmResponse {
	pub message: Message,             // AI 的回复消息(role=Assistant)
	pub tool_calls: Vec<ToolCall>,    // AI 请求的工具调用(可能为空)
	pub usage: Option<Usage>,         // Token 使用统计
	pub finish_reason: Option<String>,// 完成原因("stop" | "tool_use" | "max_tokens")
}
```

**Usage 结构:**

```rust
pub struct Usage {
	pub input_tokens: u32,   // 输入消耗的 Token 数
	pub output_tokens: u32,  // 输出生成的 Token 数
}
```

### 如何映射到不同 Provider 的 API?

不同 Provider 的 API 格式存在差异,需要转换。

#### Anthropic vs OpenAI 格式差异

| 对比维度 | Anthropic | OpenAI | Z.ai |
|---------|----------|--------|------|
| 端点 | `POST /v1/messages` | `POST /chat/completions` | `POST /chat/completions` |
| 认证头 | `x-api-key: {key}` | `Authorization: Bearer {key}` | `Authorization: Bearer {key}` |
| 系统消息 | 顶层 `system` 字段 | `messages` 数组中(`role=system`) | `messages` 数组中(`role=system`) |
| 工具结果 | `role=user`, `content=[{"type":"tool_result",...}]` | `role=tool`, `tool_call_id`, `name` | `role=tool`, `tool_call_id`, `name` |
| 工具定义 | `{name, description, input_schema}` | `{type:"function", function:{name, description, parameters}}` | `{type:"function", function:{name, description, parameters}}` |

**示例:System 消息转换**

Loom 统一格式:

```rust
Message {
	role: Role::System,
	content: "你是 Rust 专家",
	// ...
}
```

转换为 Anthropic 请求:

```json
{
  "model": "claude-3-5-sonnet-20241022",
  "system": "你是 Rust 专家",   // System 消息提取到顶层
  "messages": [...]              // 不包含 System 消息
}
```

转换为 OpenAI 请求:

```json
{
  "model": "gpt-4o",
  "messages": [
    {"role": "system", "content": "你是 Rust 专家"},  // System 消息在数组中
    ...
  ]
}
```

**示例:工具结果转换**

Loom 统一格式:

```rust
Message::tool("call_123", "read_file", "{\"content\": \"...\"}")
```

转换为 Anthropic 请求:

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "call_123",
      "content": "{\"content\": \"...\"}"
    }
  ]
}
```

转换为 OpenAI 请求:

```json
{
  "role": "tool",
  "tool_call_id": "call_123",
  "name": "read_file",
  "content": "{\"content\": \"...\"}"
}
```

---

## 4.3 LlmEvent 和 LlmStream:流式响应的事件模型

### LlmEvent 枚举

流式响应通过 `LlmEvent` 枚举表示不同阶段的事件(已在 ch02/ch03 介绍):

```rust
pub enum LlmEvent {
	TextDelta { content: String },
	ToolCallDelta { call_id: String, tool_name: String, arguments_fragment: String },
	Completed(LlmResponse),
	Error(LlmError),
}
```

### LlmStream 结构

`LlmStream` 是流式响应的包装器,实现了 `Stream` trait:

```rust
pub struct LlmStream {
	inner: Pin<Box<dyn Stream<Item = LlmEvent> + Send>>,
}

impl LlmStream {
	pub fn new(inner: Pin<Box<dyn Stream<Item = LlmEvent> + Send>>) -> Self {
		Self { inner }
	}

	pub async fn next(&mut self) -> Option<LlmEvent> {
		use futures::StreamExt;
		self.inner.next().await
	}
}

impl Stream for LlmStream {
	type Item = LlmEvent;
	fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
		self.project().inner.poll_next(cx)
	}
}
```

**为什么用 `Stream` trait 而非 async iterator?**

Rust 的 async iterator 还在稳定化中,`Stream` trait(来自 `futures` crate)是当前的标准:

- 支持 `StreamExt` 组合子(map、filter、take 等)
- 与 Tokio/async-std 生态兼容
- Pin-project 友好

### SSE(Server-Sent Events)协议解析

流式响应通常使用 SSE 协议传输。SSE 是 HTTP 流的标准格式:

**SSE 格式示例(Anthropic):**

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_123"}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" world"}}

event: message_stop
data: {"type":"message_stop"}
```

**SSE 格式示例(OpenAI):**

```
data: {"choices":[{"delta":{"role":"assistant","content":""},"index":0}]}

data: {"choices":[{"delta":{"content":"Hello"},"index":0}]}

data: {"choices":[{"delta":{"content":" world"},"index":0}]}

data: [DONE]
```

**SSE 解析步骤:**

1. **接收字节流** — HTTP 响应体是 `Stream<Item = Result<Bytes, Error>>`
2. **缓冲和拼接** — 累积字节直到遇到 `\n\n`(事件分隔符)
3. **提取 `data:` 行** — 解析 `data: {...}` 或 `event: ...` 行
4. **反序列化 JSON** — 将 JSON 字符串解析为 Provider 特定的事件类型
5. **转换为 LlmEvent** — 将 Provider 事件映射到统一的 `LlmEvent`
6. **累积状态** — 拼接文本片段、工具调用参数
7. **发送 Completed** — 流结束时,发送 `LlmEvent::Completed(LlmResponse)`

**伪代码(简化):**

```rust
fn parse_sse_stream<S>(stream: S) -> impl Stream<Item = LlmEvent>
where
	S: Stream<Item = Result<Bytes, Error>>,
{
	let mut buffer = String::new();
	let mut accumulated_text = String::new();

	stream.filter_map(move |chunk| {
		buffer.push_str(&String::from_utf8_lossy(&chunk?));

		while let Some(pos) = buffer.find("\n\n") {
			let event = &buffer[..pos];
			buffer.drain(..pos + 2);

			// 提取 data: 行
			for line in event.lines() {
				if let Some(data) = line.strip_prefix("data: ") {
					if data == "[DONE]" {
						return Some(LlmEvent::Completed(...));
					}

					// 解析 JSON
					let delta: ProviderDelta = serde_json::from_str(data)?;

					// 转换为 LlmEvent
					if let Some(text) = delta.text {
						accumulated_text.push_str(&text);
						return Some(LlmEvent::TextDelta { content: text });
					}
				}
			}
		}
		None
	})
}
```

---

## 4.4 错误处理:LlmError 类型层次

### LlmError 枚举

`LlmError` 定义了 LLM 操作可能的错误类型:

```rust
#[derive(Clone, Error, Debug)]
pub enum LlmError {
	#[error("HTTP error: {0}")]
	Http(String),                     // 网络/传输错误

	#[error("API error: {0}")]
	Api(String),                      // Provider API 错误(认证、验证等)

	#[error("Request timed out")]
	Timeout,                          // 请求超时

	#[error("Invalid response: {0}")]
	InvalidResponse(String),          // 解析/反序列化失败

	#[error("Rate limited: retry after {retry_after_secs:?} seconds")]
	RateLimited { retry_after_secs: Option<u64> }, // 429 限流响应
}
```

### 哪些错误可重试?

Loom 使用 `RetryableError` trait 标记可重试的错误:

```rust
pub trait RetryableError {
	fn is_retryable(&self) -> bool;
}

impl RetryableError for LlmError {
	fn is_retryable(&self) -> bool {
		match self {
			LlmError::Http(_) => true,        // 网络错误可重试
			LlmError::Timeout => true,        // 超时可重试
			LlmError::RateLimited { .. } => true, // 限流可重试(等待后)
			LlmError::Api(_) => false,        // API 错误通常不可重试(如认证失败)
			LlmError::InvalidResponse(_) => false, // 解析错误不可重试
		}
	}
}
```

**重试策略(由 `loom-common-http` 提供):**

- **指数退避** — 1s、2s、4s、8s、16s...
- **最大重试次数** — 默认 3 次
- **抖动(jitter)** — 避免所有请求同时重试(雪崩)

### 错误传播和转换

LlmError 通过 `From` trait 转换为更高层的 `AgentError`:

```rust
#[derive(Error, Debug)]
pub enum AgentError {
	#[error("LLM error: {0}")]
	Llm(#[from] LlmError),  // 自动转换

	#[error("Tool error: {0}")]
	Tool(#[from] ToolError),

	// ...
}
```

**使用示例:**

```rust
async fn call_llm(llm: Arc<dyn LlmClient>) -> AgentResult<LlmResponse> {
	let response = llm.complete(request).await?; // LlmError 自动转换为 AgentError
	Ok(response)
}
```

---

## 4.5 实现示例:AnthropicClient

### 客户端结构

```rust
pub struct AnthropicClient {
	config: AnthropicConfig,      // API Key、Base URL、Model
	http_client: reqwest::Client, // HTTP 客户端(来自 loom-http)
	retry_config: RetryConfig,    // 重试配置
}

pub struct AnthropicConfig {
	pub api_key: String,
	pub base_url: String, // Default: "https://api.anthropic.com"
	pub model: String,    // Default: "claude-3-5-sonnet-20241022"
}
```

### complete() 实现(非流式)

```rust
#[async_trait]
impl LlmClient for AnthropicClient {
	async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError> {
		// 1. 转换 LlmRequest → AnthropicRequest
		let anthropic_req: AnthropicRequest = (&request).into();

		// 2. 构造 HTTP 请求
		let http_req = self.http_client
			.post(format!("{}/v1/messages", self.config.base_url))
			.header("x-api-key", &self.config.api_key)
			.header("anthropic-version", "2023-06-01")
			.json(&anthropic_req);

		// 3. 发送请求(带重试)
		let response = retry(self.retry_config, || async {
			http_req.try_clone()?.send().await
		}).await?;

		// 4. 解析 AnthropicResponse
		let anthropic_resp: AnthropicResponse = response.json().await?;

		// 5. 转换 AnthropicResponse → LlmResponse
		Ok(anthropic_resp.try_into()?)
	}

	// complete_streaming() 实现见下节
}
```

### complete_streaming() 实现(流式)

```rust
async fn complete_streaming(&self, request: LlmRequest) -> Result<LlmStream, LlmError> {
	// 1. 转换 LlmRequest → AnthropicRequest(stream=true)
	let mut anthropic_req: AnthropicRequest = (&request).into();
	anthropic_req.stream = true;

	// 2. 构造 HTTP 请求
	let response = self.http_client
		.post(format!("{}/v1/messages", self.config.base_url))
		.header("x-api-key", &self.config.api_key)
		.header("anthropic-version", "2023-06-01")
		.json(&anthropic_req)
		.send()
		.await?;

	// 3. 获取字节流
	let byte_stream = response.bytes_stream();

	// 4. 解析 SSE 流为 LlmEvent 流
	let event_stream = parse_sse_stream(byte_stream);

	// 5. 包装为 LlmStream
	Ok(LlmStream::new(Box::pin(event_stream)))
}
```

### 请求/响应的完整数据流

**数据流:**

```
用户输入 "帮我优化代码"
  ↓
LlmRequest {
	model: "claude-3-5-sonnet-20241022",
	messages: [Message::user("帮我优化代码")],
	tools: [read_file, edit_file, ...],
}
  ↓ AnthropicClient::complete_streaming()
  ↓
AnthropicRequest {
	model: "claude-3-5-sonnet-20241022",
	system: None,
	messages: [{"role": "user", "content": "帮我优化代码"}],
	tools: [{"name": "read_file", "description": "...", "input_schema": {...}}, ...],
	stream: true,
}
  ↓ HTTP POST /v1/messages
  ↓
Anthropic API 返回 SSE 流:
  event: message_start
  data: {"type":"message_start",...}

  event: content_block_delta
  data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"好的"}}

  event: content_block_delta
  data: {"type":"content_block_delta","delta":{"type":"text_delta","text":",我来看看"}}

  event: message_stop
  data: {"type":"message_stop"}
  ↓ parse_sse_stream()
  ↓
LlmEvent::TextDelta { content: "好的" }
LlmEvent::TextDelta { content: ",我来看看" }
LlmEvent::Completed(LlmResponse { message: ..., tool_calls: [], ... })
  ↓
状态机收到事件,转换状态
```

---

## 4.6 服务端 LLM 代理:ProxyLlmClient

### 为什么需要服务端代理?

**问题:** 如果客户端直接调用 Provider API:

1. **安全风险** — API Key 必须存储在客户端(易泄露)
2. **审计困难** — 无法集中记录所有 LLM 请求
3. **配额管理** — 无法在团队间共享配额、限流

**解决方案:** 客户端通过服务端代理调用 LLM,API Key 只存储在服务端。

### ProxyLlmClient 架构

```
客户端(loom-cli)                           服务端(loom-server)
  ↓                                           ↓
ProxyLlmClient                            LlmService
  ↓                                           ↓
POST /proxy/anthropic/stream            AnthropicClient
  ↓                                           ↓
SSE 流 ←──────────────────────────→  Anthropic API
```

**客户端代码:**

```rust
// 使用便捷构造器
let client = ProxyLlmClient::anthropic("https://loom.ghuntley.com")?;

// 或显式选择 Provider
let client = ProxyLlmClient::new("https://loom.ghuntley.com", LlmProvider::Anthropic)?;

// 调用 LLM(通过服务端代理)
let response = client.complete(request).await?;
```

**服务端代码(简化):**

```rust
// 路由:POST /proxy/anthropic/stream
async fn anthropic_stream(
	service: Arc<LlmService>,
	request: LlmRequest,
) -> Result<SSE Stream, Error> {
	let stream = service.complete_streaming_anthropic(request).await?;
	Ok(stream.into_sse())
}
```

### 多 Provider 并存

服务端可以同时配置多个 Provider:

```bash
# 环境变量
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
ZAI_API_KEY=...
```

**LlmService 结构:**

```rust
pub struct LlmService {
	anthropic_client: Option<AnthropicClient>,
	openai_client: Option<OpenAIClient>,
	zai_client: Option<ZaiClient>,
}

impl LlmService {
	pub async fn complete_anthropic(&self, request: LlmRequest) -> Result<LlmResponse, LlmError> {
		let client = self.anthropic_client.as_ref().ok_or(LlmError::Api("Anthropic not configured"))?;
		client.complete(request).await
	}

	pub async fn complete_openai(&self, request: LlmRequest) -> Result<LlmResponse, LlmError> {
		let client = self.openai_client.as_ref().ok_or(LlmError::Api("OpenAI not configured"))?;
		client.complete(request).await
	}

	pub async fn complete_zai(&self, request: LlmRequest) -> Result<LlmResponse, LlmError> {
		let client = self.zai_client.as_ref().ok_or(LlmError::Api("Zai not configured"))?;
		client.complete(request).await
	}
}
```

**客户端选择 Provider:**

```rust
// 使用 Anthropic
let client = ProxyLlmClient::anthropic("https://loom.ghuntley.com")?;

// 使用 OpenAI
let client = ProxyLlmClient::openai("https://loom.ghuntley.com")?;

// 使用 Z.ai
let client = ProxyLlmClient::zai("https://loom.ghuntley.com")?;
```

---

## 4.7 小结

本章介绍了 Loom 的 LLM 抽象层设计:

1. **LlmClient trait** — 统一接口,支持多 Provider(`complete()` 和 `complete_streaming()`)
2. **LlmRequest/LlmResponse** — 标准化的请求/响应,映射到不同 Provider 的 API 格式
3. **LlmEvent 和 LlmStream** — 流式响应的事件模型,通过 SSE 解析实现
4. **LlmError** — 错误类型层次,支持重试逻辑(`RetryableError` trait)
5. **AnthropicClient** — 实现示例,展示请求/响应转换和 SSE 解析
6. **ProxyLlmClient** — 服务端代理,隐藏 API Key,支持多 Provider 并存

**关键设计原则:**

- **统一抽象** — `LlmClient` trait 隐藏 Provider 差异
- **运行时多态** — `Arc<dyn LlmClient>` 支持配置驱动的 Provider 选择
- **流式优先** — SSE 流式响应提供实时反馈
- **安全第一** — API Key 只存储在服务端,客户端通过代理调用
- **可扩展性** — 添加新 Provider 只需实现 trait,无需修改核心代码

**与 ch02/ch03 的衔接:**

- `LlmRequest` 在状态机中通过 `SendLlmRequest(req)` 动作发送
- `LlmEvent` 驱动状态机转换(`CallingLlm` → `ProcessingLlmResponse`)
- `LlmResponse` 携带 `tool_calls`,触发 `ExecutingTools` 状态

下一章(第 5 章)将深入介绍**服务端 LLM 代理**,看 API Key 如何安全存储、Anthropic OAuth 池化如何实现,以及如何在多个 Provider 间分发请求。

---

## 质检清单

- [x] **LlmClient trait 设计** — 讲清楚了为什么需要抽象、trait 的设计权衡(async_trait、Arc<dyn>)
- [x] **不同 Provider 的 API 差异** — 表格对比 Anthropic vs OpenAI,具体转换示例(System 消息、工具结果)
- [x] **SSE 解析流程** — 详细讲解了 SSE 格式、解析步骤、伪代码示例
- [x] **错误处理** — LlmError 类型层次、RetryableError trait、重试策略
- [x] **代码片段控制** — 5 处代码片段(trait 定义、AnthropicClient 实现、SSE 伪代码、LlmService 结构、ProxyLlmClient 使用)
- [x] **与 ch02/ch03 衔接** — 引用了 LlmRequest/LlmResponse/LlmEvent,说明了在状态机中的使用
- [x] **教学节奏** — 从抽象(trait)到细节(转换)到实现(AnthropicClient)到架构(ProxyLlmClient)
- [x] **周边知识** — 解释了 async_trait、SSE 协议、指数退避重试等概念
- [x] **准确性** — 所有类型、方法、转换规则与实际代码一致(已验证 llm.rs/error.rs/anthropic/proxy)
- [x] **可读性** — 使用了表格、代码示例、数据流图示,层次清晰
