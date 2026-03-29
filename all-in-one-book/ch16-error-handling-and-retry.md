# 第 16 章 错误处理与重试：thiserror + 指数退避

> **本章目标**
> 讲解 Loom 的错误处理架构,包括 `thiserror` 的错误类型层次、`RetryConfig` 的指数退避算法、`RetryableError` trait 的可重试性判断。读者将理解如何设计健壮的错误传播机制和自动重试策略。

---

## 16.1 本章是什么

### 16.1.1 章节定位

**错误处理**和**重试策略**是构建可靠系统的基石:

- **错误处理** — 如何定义、传播和日志化错误
- **重试策略** — 哪些错误可重试?如何避免"雪崩"(过度重试导致系统崩溃)?

**本章解答:**

- **错误类型如何组织?** — `AgentError` → `LlmError`/`ToolError` 的层次结构
- **如何自动转换错误?** — `From` trait 和 `#[from]` 属性
- **哪些错误可重试?** — `RetryableError` trait 的判断逻辑
- **如何避免重试风暴?** — 指数退避 + Jitter 的延迟算法

### 16.1.2 核心概念

**thiserror** — Rust 错误处理库,通过派生宏自动生成 `Display` 和 `From` 实现,简化错误类型定义。

**错误层次** — `AgentError`(顶层) → `LlmError`/`ToolError`(领域特定) → `std::io::Error`(底层)。

**RetryConfig** — 重试配置结构体,定义 `max_attempts`、`base_delay`、`backoff_factor` 等参数。

**指数退避(Exponential Backoff)** — 重试延迟按指数增长:`delay = base_delay × backoff_factor^attempt`。

**Jitter** — 在延迟时间上加入随机性,避免多个客户端同时重试(惊群效应)。

---

## 16.2 错误类型层次

### 16.2.1 AgentError:顶层错误

**定义:** (见 `specs/error-handling.md:26`)

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AgentError {
    #[error("LLM error: {0}")]
    Llm(#[from] LlmError),

    #[error("Tool error: {0}")]
    Tool(#[from] ToolError),

    #[error("Invalid state: {0}")]
    InvalidState(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Operation timed out: {0}")]
    Timeout(String),

    #[error("Internal error: {0}")]
    Internal(String),
}

pub type AgentResult<T> = Result<T, AgentError>;
```

**关键特性:**

- **自动 Display** — `#[error("...")]` 生成 `Display` 实现
- **自动 From** — `#[from]` 生成 `From<LlmError> for AgentError` 等转换
- **类型别名** — `AgentResult<T>` 简化函数签名

**错误变体说明:**

| 变体 | 用途 | 示例 |
|------|------|------|
| `Llm` | LLM API 错误 | 网络超时、API 限流、无效响应 |
| `Tool` | Tool 执行错误 | 文件不存在、路径越界、序列化失败 |
| `InvalidState` | 状态机非法转换 | 在 `WaitingForUserInput` 时收到 `ToolCompleted` |
| `Io` | 文件/网络 I/O 错误 | `std::io::Error` 的包装 |
| `Timeout` | 操作超时 | LLM 请求超时、Tool 执行超时 |
| `Internal` | 内部错误(Bug) | 不应发生的错误,表明代码逻辑缺陷 |

### 16.2.2 LlmError:LLM 专属错误

**定义:** (见 `specs/error-handling.md:63`)

```rust
#[derive(Clone, Error, Debug)]
pub enum LlmError {
    #[error("HTTP error: {0}")]
    Http(String),

    #[error("API error: {0}")]
    Api(String),

    #[error("Request timed out")]
    Timeout,

    #[error("Invalid response: {0}")]
    InvalidResponse(String),

    #[error("Rate limited: retry after {retry_after_secs:?} seconds")]
    RateLimited { retry_after_secs: Option<u64> },
}
```

**可重试性:**

| 变体 | 可重试? | 原因 |
|------|---------|------|
| `Http` | ✅ 是 | 网络临时故障 |
| `Api` | ⚠️ 取决于消息 | 某些 API 错误可重试,需检查具体消息 |
| `Timeout` | ✅ 是 | 请求超时,可能下次成功 |
| `InvalidResponse` | ❌ 否 | 响应格式错误,重试无意义 |
| `RateLimited` | ✅ 是 | 限流,等待后可重试 |

### 16.2.3 ToolError:Tool 专属错误

**定义:** (见 `specs/error-handling.md:95`)

```rust
#[derive(Clone, Error, Debug)]
pub enum ToolError {
    #[error("Tool not found: {0}")]
    NotFound(String),

    #[error("Invalid arguments: {0}")]
    InvalidArguments(String),

    #[error("IO error: {0}")]
    Io(String), // 注意: 字符串而非 std::io::Error (为了满足 Clone)

    #[error("Tool execution timed out")]
    Timeout,

    #[error("Path outside workspace: {0}")]
    PathOutsideWorkspace(PathBuf),

    #[error("File not found: {0}")]
    FileNotFound(PathBuf),

    #[error("Serialization error: {0}")]
    Serialization(String),
}
```

**设计要点:**

1. **Clone 约束** — `ToolError` 实现 `Clone`,便于在状态机中传递
2. **字符串化 I/O 错误** — `std::io::Error` 不实现 `Clone`,转换为 `String`

---

## 16.3 错误传播机制

### 16.3.1 From trait 自动转换

**问题:** 如何让底层错误自动转换为顶层错误?

**答案:** 使用 `thiserror` 的 `#[from]` 属性自动生成 `From` 实现。

**示例:**

```rust
#[derive(Error, Debug)]
pub enum AgentError {
    #[error("Tool error: {0}")]
    Tool(#[from] ToolError), // 自动生成 From<ToolError> for AgentError
}
```

**生成的代码:**

```rust
impl From<ToolError> for AgentError {
    fn from(e: ToolError) -> Self {
        AgentError::Tool(e)
    }
}
```

**使用效果:**

```rust
fn read_file(path: &Path) -> Result<String, ToolError> {
    let content = std::fs::read_to_string(path)?; // std::io::Error → ToolError::Io
    Ok(content)
}

fn execute_tool(name: &str) -> AgentResult<String> {
    if name == "read_file" {
        return read_file(Path::new("/tmp/file.txt"))?; // ToolError → AgentError::Tool
    }
    Ok("".to_string())
}
```

### 16.3.2 错误传播路径

**数据流:** (从底层到顶层)

```
std::io::Error (文件读取失败)
  ↓ From<std::io::Error> for ToolError
ToolError::Io("No such file or directory")
  ↓ From<ToolError> for AgentError
AgentError::Tool(ToolError::Io(...))
  ↓ Agent::handle_event()
AgentAction::DisplayError("Tool error: IO error: No such file or directory")
```

**代码示例:**

```rust
// 1. 底层 I/O 操作
let content = std::fs::read_to_string(path) // 返回 Result<String, std::io::Error>
    .map_err(|e| ToolError::Io(e.to_string()))?; // 转换为 ToolError

// 2. Tool Registry 调用
let result = tool.invoke(args, ctx).await?; // 返回 Result<Value, ToolError>

// 3. Agent 状态机处理
match outcome {
    ToolExecutionOutcome::Error { error, .. } => {
        // error 已经是 ToolError,会自动转换为 AgentError
        return Err(AgentError::Tool(error));
    }
}
```

---

## 16.4 RetryConfig:重试配置

### 16.4.1 配置结构

**定义:** (见 `specs/retry-strategy.md:20`)

```rust
use std::time::Duration;
use http::StatusCode;

pub struct RetryConfig {
    pub max_attempts: u32,       // 最大尝试次数
    pub base_delay: Duration,    // 初始延迟
    pub max_delay: Duration,     // 最大延迟(上限)
    pub backoff_factor: f64,     // 指数增长因子
    pub jitter: bool,            // 是否添加抖动
    pub retryable_statuses: Vec<StatusCode>, // 可重试的 HTTP 状态码
}
```

**默认值:**

```rust
impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_attempts: 3,
            base_delay: Duration::from_millis(200),
            max_delay: Duration::from_secs(5),
            backoff_factor: 2.0,
            jitter: true,
            retryable_statuses: vec![
                StatusCode::TOO_MANY_REQUESTS,   // 429
                StatusCode::REQUEST_TIMEOUT,     // 408
                StatusCode::BAD_GATEWAY,         // 502
                StatusCode::SERVICE_UNAVAILABLE, // 503
                StatusCode::GATEWAY_TIMEOUT,     // 504
            ],
        }
    }
}
```

### 16.4.2 指数退避算法

**公式:**

```
delay = base_delay × (backoff_factor ^ attempt)
```

**计算步骤:**

1. **指数延迟** = `200ms × 2^attempt`
2. **应用上限** = `min(指数延迟, max_delay)`
3. **应用 Jitter** (如果启用) = `上限延迟 × (0.5 + random(0..1))`

**示例序列:** (默认配置,无 Jitter)

| 尝试次数 | 计算 | 延迟 |
|----------|------|------|
| 0 (初次) | — | 0ms (立即执行) |
| 1 | `200ms × 2^0` | 200ms |
| 2 | `200ms × 2^1` | 400ms |
| 3 | `200ms × 2^2` | 800ms |
| 4 | `200ms × 2^3` | 1600ms |
| 5 | `200ms × 2^4` | 3200ms |
| 6 | `200ms × 2^5 = 6400ms` | **5000ms** (达到上限) |

**Rust 实现:**

```rust
impl RetryConfig {
    pub fn calculate_delay(&self, attempt: u32) -> Duration {
        // 1. 计算指数延迟
        let exponential_delay = self.base_delay.as_secs_f64()
            * self.backoff_factor.powi(attempt as i32);

        // 2. 应用上限
        let capped_delay = exponential_delay.min(self.max_delay.as_secs_f64());

        // 3. 应用 Jitter
        let final_delay = if self.jitter {
            use rand::Rng;
            let mut rng = rand::thread_rng();
            let jitter_factor = 0.5 + rng.gen::<f64>(); // [0.5, 1.5]
            capped_delay * jitter_factor
        } else {
            capped_delay
        };

        Duration::from_secs_f64(final_delay)
    }
}
```

### 16.4.3 Jitter 的作用

**问题:** 多个客户端同时重试会导致"惊群效应"(thundering herd),服务端瞬间收到大量请求。

**答案:** 加入随机性,让重试时间分散在 `[delay × 0.5, delay × 1.5]` 范围内。

**效果对比:**

| 客户端 | 无 Jitter (所有客户端同时重试) | 有 Jitter (分散重试) |
|--------|-------------------------------|---------------------|
| A | 200ms | 150ms (随机) |
| B | 200ms | 220ms (随机) |
| C | 200ms | 180ms (随机) |
| D | 200ms | 240ms (随机) |

**结论:** Jitter 将重试时间从"集中爆发"变为"平缓分布",减轻服务端压力。

---

## 16.5 RetryableError trait

### 16.5.1 trait 定义

**问题:** 如何判断一个错误是否可重试?

**答案:** 定义 `RetryableError` trait,让错误类型实现 `is_retryable()` 方法。

**定义:** (见 `specs/retry-strategy.md:75`)

```rust
pub trait RetryableError {
    fn is_retryable(&self) -> bool;
}
```

### 16.5.2 reqwest::Error 实现

**实现:** (见 `specs/retry-strategy.md:84`)

```rust
impl RetryableError for reqwest::Error {
    fn is_retryable(&self) -> bool {
        // 1. 网络错误总是可重试
        if self.is_timeout() || self.is_connect() {
            return true;
        }

        // 2. 特定 HTTP 状态码可重试
        if let Some(status) = self.status() {
            let retryable_statuses = [
                StatusCode::TOO_MANY_REQUESTS,        // 429
                StatusCode::REQUEST_TIMEOUT,          // 408
                StatusCode::INTERNAL_SERVER_ERROR,    // 500
                StatusCode::BAD_GATEWAY,              // 502
                StatusCode::SERVICE_UNAVAILABLE,      // 503
                StatusCode::GATEWAY_TIMEOUT,          // 504
            ];
            return retryable_statuses.contains(&status);
        }

        false // 其他错误不可重试
    }
}
```

### 16.5.3 可重试 vs 不可重试

**可重试的条件:**

| 类型 | 示例 | 原因 |
|------|------|------|
| **网络超时** | `reqwest::Error::is_timeout()` | 可能是临时网络拥塞 |
| **连接失败** | `reqwest::Error::is_connect()` | 服务端可能临时不可达 |
| **429 Too Many Requests** | 限流 | 等待后配额恢复 |
| **500/502/503/504** | 服务端错误 | 临时故障,可能自动恢复 |

**不可重试的条件:**

| 类型 | 示例 | 原因 |
|------|------|------|
| **400 Bad Request** | 请求格式错误 | 重试不会改变结果 |
| **401 Unauthorized** | 认证失败 | 需要用户修复凭据 |
| **403 Forbidden** | 权限不足 | 重试无意义 |
| **404 Not Found** | 资源不存在 | 重试不会使资源出现 |

---

## 16.6 Agent 状态机的重试逻辑

### 16.6.1 LLM 错误的自动重试

**状态转换:** (见 ch03 § 3.4 状态转换表)

```
CallingLlm + LlmEvent::Error (retries < max) → Error (origin=Llm)
  ↓ 等待 RetryTimeoutFired
Error (origin=Llm) + RetryTimeoutFired → CallingLlm (重试)
```

**代码片段:** (见 `specs/error-handling.md:198`)

```rust
// Agent::handle_event() 中的处理
(
    AgentState::CallingLlm { conversation, retries },
    AgentEvent::LlmEvent(LlmEvent::Error(e)),
) => {
    let new_retries = *retries + 1;
    if new_retries < self.config.max_retries {
        // 进入 Error 状态,等待重试
        self.state = AgentState::Error {
            conversation: conversation.clone(),
            error: AgentError::Llm(e),
            retries: new_retries,
            origin: ErrorOrigin::Llm,
        };
        AgentAction::WaitForInput // CLI 会设置定时器触发 RetryTimeoutFired
    } else {
        // 超过最大重试次数,放弃
        self.state = AgentState::WaitingForUserInput {
            conversation: conversation.clone(),
        };
        AgentAction::DisplayError(AgentError::Llm(e).to_string())
    }
}
```

**延迟计算:**

```rust
// CLI 端(假设实现)
if let AgentAction::WaitForInput = action {
    if let AgentState::Error { retries, origin: ErrorOrigin::Llm, .. } = agent.state() {
        let delay = retry_config.calculate_delay(retries);
        tokio::time::sleep(delay).await;
        agent.handle_event(AgentEvent::RetryTimeoutFired)?;
    }
}
```

### 16.6.2 Tool 错误的处理

**策略:** Tool 错误**不自动重试**,因为 Tool 错误通常是确定性的(文件不存在、路径错误等)。

**处理流程:**

```rust
// Tool 执行失败
ToolExecutionOutcome::Error { call_id, error } => {
    // 将错误包装为 Tool 消息返回给 LLM
    let message = Message {
        role: Role::Tool,
        content: format!("Error: {}", error),
        tool_call_id: Some(call_id),
        ..Default::default()
    };
    // LLM 根据错误信息决定如何处理(可能重新调用工具或向用户请求帮助)
}
```

---

## 16.7 HTTP 客户端的重试集成

### 16.7.1 loom-http::builder() with Retry

**问题:** 如何在 HTTP 客户端中集成重试逻辑?

**答案:** `loom-http::builder()` 返回预配置的 `reqwest::ClientBuilder`,支持自定义重试中间件。

**示例:**

```rust
use loom_http::{builder, RetryConfig};

let retry_config = RetryConfig::default();
let client = builder()
    .timeout(Duration::from_secs(30))
    .build()?;

// 发送请求,手动实现重试
let mut attempt = 0;
loop {
    let response = client.get("https://api.anthropic.com/v1/messages").send().await;

    match response {
        Ok(resp) if resp.status().is_success() => break,
        Err(e) if e.is_retryable() && attempt < retry_config.max_attempts => {
            let delay = retry_config.calculate_delay(attempt);
            tokio::time::sleep(delay).await;
            attempt += 1;
            continue;
        }
        Err(e) => return Err(e.into()),
    }
}
```

**注意:** Loom 当前未使用 `reqwest-middleware` 等自动重试库,重试逻辑由调用方实现(如 `ProxyLlmClient`)。

---

## 16.8 本章小结

### 16.8.1 核心知识点回顾

1. **thiserror** — 通过 `#[error]` 和 `#[from]` 简化错误类型定义和转换
2. **错误层次** — `AgentError` → `LlmError`/`ToolError` → `std::io::Error`
3. **RetryConfig** — 定义 `max_attempts`、`base_delay`、`backoff_factor`、`jitter`
4. **指数退避** — `delay = base_delay × backoff_factor^attempt`,加 Jitter 避免惊群
5. **RetryableError trait** — 判断错误是否可重试
6. **Agent 重试** — LLM 错误自动重试,Tool 错误不重试(返回给 LLM 处理)

### 16.8.2 与其他章节的呼应

| 章节 | 在本章的体现 |
|------|------------|
| ch03 | Agent 状态机的 `Error` 状态和 `RetryTimeoutFired` 事件 |
| ch04 | `LlmError` 的定义和 HTTP 重试策略 |
| ch06 | `ToolError` 的路径安全错误(`PathOutsideWorkspace`) |

### 16.8.3 未覆盖的主题

以下主题在本章中仅简要提及,将在后续章节详细讨论:

- **ch17 HTTP 客户端标准化** — `loom-http::builder()` 的完整配置
- **ch19 测试策略** — 如何测试重试逻辑(模拟失败、验证延迟)

---

## 16.9 质检清单

在完成本章后,请验证以下内容:

- [x] **错误类型层次** — 是否讲解了 `AgentError`、`LlmError`、`ToolError` 的定义?
- [x] **thiserror 用法** — 是否说明了 `#[error]` 和 `#[from]` 的作用?
- [x] **RetryConfig** — 是否定义了所有配置字段和默认值?
- [x] **指数退避算法** — 是否给出了计算公式和示例序列?
- [x] **Jitter 的作用** — 是否解释了为什么需要随机性?
- [x] **RetryableError trait** — 是否讲解了 `is_retryable()` 的实现?
- [x] **Agent 重试逻辑** — 是否说明了 LLM 错误的自动重试流程?

---

**下一章预告:** ch17 将讨论 **HTTP 客户端标准化:User-Agent、重试策略**,展示 `loom-http` 工具库如何统一 HTTP 客户端的配置、User-Agent 格式和超时策略。
