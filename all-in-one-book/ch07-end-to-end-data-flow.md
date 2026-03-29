# 第 7 章 数据流全景：一次完整对话的端到端追踪

> **本章目标**
> 通过追踪一次完整的用户对话(从输入到最终响应),串联前面所有章节的知识点,展示数据在各个模块间如何流动和转换。这是一个"全景章节",会大量引用 ch02-ch06 的细节。

---

## 7.1 本章是什么

### 7.1.1 章节定位

前面的章节分别讲解了 Loom 的核心组件:

- **ch02** — 核心类型系统 (`Message`、`Role`、`ToolCall`、`LlmRequest/Response`)
- **ch03** — Agent 状态机 (`AgentState`、`AgentEvent`、`AgentAction`)
- **ch04** — LLM 抽象层 (`LlmClient` trait、`LlmEvent`、SSE 解析)
- **ch06** — Tool 系统 (`Tool` trait、`ToolRegistry`、路径安全)

这些组件就像一个个"零件",而**本章的任务是展示它们如何组装成一个完整的"对话引擎"**。

我们会用一个具体例子(用户要求"编辑文件并提交 Git")来追踪数据在整个系统中的流动:

```
用户输入
  ↓
Agent 状态机 (WaitingForUserInput → CallingLlm)
  ↓
LlmClient (ProxyLlmClient → Anthropic API)
  ↓
LLM 响应 (tool_use: edit_file)
  ↓
Tool 执行 (ToolRegistry → EditFileTool)
  ↓
PostToolsHook (检测到 edit_file,触发 auto-commit)
  ↓
再次调用 LLM (带 Tool 结果)
  ↓
最终响应返回给用户
```

### 7.1.2 为什么需要这一章

**认知规律:** 理解复杂系统需要"先局部后全局"。前面章节建立了局部概念,现在需要**全局视角**串联起来。

**工程价值:** 追踪数据流可以帮助我们:

1. **理解数据转换** — 每一跳数据的形态变化(从字符串到 `Message` 到 `LlmRequest` 到 SSE 流)
2. **定位问题边界** — 某个字段丢失了?某个转换出错了?全局视角更容易定位
3. **验证设计一致性** — 状态机的 `ConversationContext` 是否贯穿始终?

---

## 7.2 示例场景定义

为了让数据流可追踪,我们定义一个具体场景:

### 7.2.1 用户目标

用户在 Loom CLI 中输入:

```
请在 src/main.rs 的第 10 行后面添加一行注释 "// Initialize logger",然后提交 Git
```

### 7.2.2 预期行为

Loom 应该:

1. **理解意图** — 用户需要编辑文件 + 提交 Git
2. **调用 Tool** — 使用 `edit_file` 工具修改 `src/main.rs`
3. **自动 Git 提交** — PostToolsHook 检测到 `edit_file`,自动 `git commit`
4. **返回确认** — 告诉用户"已添加注释并提交到 Git"

### 7.2.3 涉及的组件

- **Agent 状态机** (ch03) — 编排整个流程
- **核心类型** (ch02) — `Message`、`ToolCall` 等数据结构
- **LLM 抽象层** (ch04) — 与 Anthropic 通信
- **Tool 系统** (ch06) — `edit_file` 和 `bash` 工具
- **PostToolsHook** (ch03) — 自动 Git 提交逻辑

---

## 7.3 数据流追踪:15 个关键节点

我们将这次对话分解为 **15 个关键节点**,每个节点标注:

- **节点编号和名称** — 例如 `[N1] 用户输入`
- **数据形态** — 该节点的数据类型和示例值
- **触发的状态转换** — Agent 状态机的变化
- **涉及的代码** — 具体的文件和函数

---

### [N1] 用户在 CLI 输入文本

**数据形态:**

```rust
// 原始字符串
let user_input: &str = "请在 src/main.rs 的第 10 行后面添加一行注释 \"// Initialize logger\",然后提交 Git";
```

**发生位置:** `crates/loom-cli/src/tui/input.rs` (假设 TUI 输入组件)

**下一步:** CLI 将字符串包装为 `Message` 并发送 `AgentEvent::UserInput`

---

### [N2] CLI 构造 Message 并触发事件

**数据形态:**

```rust
use loom_common_core::message::{Message, Role};

let msg = Message {
    role: Role::User,
    content: "请在 src/main.rs 的第 10 行后面添加一行注释 \"// Initialize logger\",然后提交 Git".to_string(),
    tool_call_id: None,
    name: None,
    tool_calls: Vec::new(),
};

let event = AgentEvent::UserInput(msg);
```

**涉及的类型定义:** 见 ch02 § 2.2.1 `Message` 结构体

**下一步:** 调用 `agent.handle_event(event)`

---

### [N3] Agent 状态机处理 UserInput 事件

**当前状态:** `AgentState::WaitingForUserInput { conversation }`

**事件:** `AgentEvent::UserInput(msg)`

**处理逻辑:** (见 `crates/loom-common-core/src/agent.rs:167`)

```rust
(AgentState::WaitingForUserInput { conversation }, AgentEvent::UserInput(msg)) => {
    conversation.messages.push(msg); // 将用户消息追加到对话历史
    let request = LlmRequest {
        model: self.config.model_name.clone(),
        messages: conversation.messages.clone(),
        tools: self.tools.clone(),
        max_tokens: Some(self.config.max_tokens),
        temperature: self.config.temperature,
    };
    let new_conversation = conversation.clone();
    self.state = AgentState::CallingLlm {
        conversation: new_conversation,
        retries: 0,
    };
    AgentAction::SendLlmRequest(request)
}
```

**状态转换:**

```
WaitingForUserInput → CallingLlm
```

**返回的 Action:**

```rust
AgentAction::SendLlmRequest(LlmRequest {
    model: "claude-sonnet-4.5",
    messages: [
        Message { role: User, content: "请在 src/main.rs ..." }
    ],
    tools: [
        ToolDefinition { name: "read_file", ... },
        ToolDefinition { name: "edit_file", ... },
        ToolDefinition { name: "bash", ... },
    ],
    max_tokens: Some(8192),
    temperature: 0.7,
})
```

**关键数据变化:**

- `conversation.messages` 从空数组变为包含 1 条用户消息
- `AgentState` 从 `WaitingForUserInput` 变为 `CallingLlm`
- 生成了 `LlmRequest` 携带完整的消息历史和可用工具列表

**下一步:** CLI 收到 `AgentAction::SendLlmRequest`,需要调用 LLM

---

### [N4] CLI 调用 ProxyLlmClient.complete_streaming()

**代码路径:** `crates/loom-llm-proxy/src/lib.rs`

**输入数据:** 上一步的 `LlmRequest`

**关键逻辑:**

```rust
pub async fn complete_streaming(&self, request: LlmRequest) -> Result<LlmStream, LlmError> {
    // 1. 将 LlmRequest 序列化为 JSON
    let body = serde_json::to_vec(&request)?;

    // 2. 发送 HTTP POST 到服务端代理
    let response = self.client
        .post(&format!("{}/llm/v1/messages", self.base_url))
        .header("Authorization", format!("Bearer {}", self.token.expose()))
        .header("Content-Type", "application/json")
        .body(body)
        .send()
        .await?;

    // 3. 返回 SSE 流
    Ok(LlmStream::from_sse_stream(response.bytes_stream()))
}
```

**HTTP 请求示例:**

```http
POST /llm/v1/messages HTTP/1.1
Host: loom.ghuntley.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Content-Type: application/json

{
  "model": "claude-sonnet-4.5",
  "messages": [
    {
      "role": "user",
      "content": "请在 src/main.rs 的第 10 行后面添加一行注释 \"// Initialize logger\",然后提交 Git"
    }
  ],
  "tools": [
    {
      "name": "edit_file",
      "description": "Edit a file by replacing old_text with new_text",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": { "type": "string" },
          "old_text": { "type": "string" },
          "new_text": { "type": "string" }
        },
        "required": ["file_path", "old_text", "new_text"]
      }
    }
  ],
  "max_tokens": 8192,
  "temperature": 0.7
}
```

**下一步:** 服务端 `loom-server` 收到请求

---

### [N5] 服务端 LlmService 转发到 Anthropic

**代码路径:** `crates/loom-server/src/llm_service.rs` (假设)

**关键逻辑:**

```rust
// 1. 解析请求体得到 LlmRequest
let llm_request: LlmRequest = serde_json::from_slice(&body)?;

// 2. 根据 Provider 选择对应的 LlmClient
let client: Arc<dyn LlmClient> = self.get_provider("anthropic")?;

// 3. 调用 Anthropic 的 LlmClient
let stream = client.complete_streaming(llm_request).await?;

// 4. 转发 SSE 流给客户端
Ok(Response::sse(stream))
```

**Anthropic API 请求示例:** (见 ch04 § 4.5.1)

```http
POST /v1/messages HTTP/1.1
Host: api.anthropic.com
x-api-key: sk-ant-api03-xxx
anthropic-version: 2023-06-01
Content-Type: application/json

{
  "model": "claude-sonnet-4.5-20250929",
  "messages": [
    { "role": "user", "content": "请在 src/main.rs ..." }
  ],
  "tools": [
    { "name": "edit_file", ... }
  ],
  "max_tokens": 8192,
  "stream": true
}
```

**下一步:** Anthropic 开始返回 SSE 流

---

### [N6] Anthropic 返回 SSE 流 (TextDelta)

**SSE 数据示例:** (分多个 chunk 返回)

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01ABC","type":"message","role":"assistant"}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"好的"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"，我来"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"帮你编辑"}}
```

**解析为 LlmEvent:** (见 ch04 § 4.4.1)

```rust
LlmEvent::TextDelta { content: "好的" }
LlmEvent::TextDelta { content: "，我来" }
LlmEvent::TextDelta { content: "帮你编辑" }
```

**服务端行为:** 直接转发 SSE 流给客户端(不做解析)

**客户端行为:** `ProxyLlmClient` 的 `LlmStream` 解析 SSE 并生成 `LlmEvent`

---

### [N7] Agent 收到 LlmEvent::TextDelta

**事件:**

```rust
AgentEvent::LlmEvent(LlmEvent::TextDelta { content: "好的，我来帮你编辑" })
```

**当前状态:** `AgentState::CallingLlm { conversation, retries: 0 }`

**处理逻辑:** (见 `agent.rs:186`)

```rust
(AgentState::CallingLlm { .. }, AgentEvent::LlmEvent(LlmEvent::TextDelta { content })) => {
    AgentAction::DisplayMessage(content)
}
```

**状态:** 不变(仍在 `CallingLlm`)

**返回的 Action:**

```rust
AgentAction::DisplayMessage("好的,我来帮你编辑")
```

**CLI 行为:** 在 TUI 中逐字显示文本

---

### [N8] Anthropic 返回 tool_use 块

**SSE 数据示例:**

```
event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_01XYZ","name":"edit_file"}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"file_path\":\""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"src/main.rs\",\"old_text\":\""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"fn main() {\\n\",\"new_text\":\"fn main() {\\n    // Initialize logger\\n\""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}
```

**解析为 LlmEvent:** (见 ch04 § 4.4.2)

```rust
LlmEvent::ToolCallDelta {
    call_id: "toolu_01XYZ",
    tool_name: "edit_file",
    arguments_fragment: "{\"file_path\":\"src/main.rs\",\"old_text\":\"fn main() {\\n\",\"new_text\":\"fn main() {\\n    // Initialize logger\\n\"}"
}
```

**Agent 处理:** (见 `agent.rs:191`)

```rust
(AgentState::CallingLlm { .. }, AgentEvent::LlmEvent(LlmEvent::ToolCallDelta { .. })) => {
    AgentAction::WaitForInput // 不显示中间片段,等待完整的 Completed
}
```

---

### [N9] SSE 流结束,收到 LlmEvent::Completed

**最后的 SSE 数据:**

```
event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use","stop_sequence":null}}

event: message_stop
data: {"type":"message_stop"}
```

**解析为 LlmEvent:**

```rust
LlmEvent::Completed(LlmResponse {
    message: Message {
        role: Role::Assistant,
        content: "好的,我来帮你编辑文件".to_string(),
        tool_calls: vec![
            ToolCall {
                id: "toolu_01XYZ".to_string(),
                name: "edit_file".to_string(),
                arguments: json!({
                    "file_path": "src/main.rs",
                    "old_text": "fn main() {\n",
                    "new_text": "fn main() {\n    // Initialize logger\n"
                }),
            }
        ],
        tool_call_id: None,
        name: None,
    },
    stop_reason: Some("tool_use".to_string()),
})
```

**Agent 状态转换:** (见 `agent.rs:196`)

```rust
(AgentState::CallingLlm { conversation, .. }, AgentEvent::LlmEvent(LlmEvent::Completed(response))) => {
    let mut conv = conversation.clone();
    conv.messages.push(response.message.clone()); // 追加 Assistant 消息到对话历史
    self.state = AgentState::ProcessingLlmResponse {
        conversation: conv,
        response,
    };
    self.process_llm_response() // 内部方法,处理 tool_calls
}
```

**`process_llm_response()` 逻辑:** (假设实现)

```rust
fn process_llm_response(&mut self) -> AgentAction {
    match &self.state {
        AgentState::ProcessingLlmResponse { conversation, response } => {
            if response.message.tool_calls.is_empty() {
                // 没有 tool_calls,直接返回给用户
                self.state = AgentState::WaitingForUserInput {
                    conversation: conversation.clone(),
                };
                AgentAction::WaitForInput
            } else {
                // 有 tool_calls,转到 ExecutingTools 状态
                let executions: Vec<ToolExecutionStatus> = response.message.tool_calls
                    .iter()
                    .map(|tc| ToolExecutionStatus::Pending {
                        call_id: tc.id.clone(),
                        tool_name: tc.name.clone(),
                    })
                    .collect();
                self.state = AgentState::ExecutingTools {
                    conversation: conversation.clone(),
                    executions,
                };
                AgentAction::ExecuteTools(response.message.tool_calls.clone())
            }
        }
        _ => unreachable!(),
    }
}
```

**状态转换:**

```
CallingLlm → ProcessingLlmResponse → ExecutingTools
```

**返回的 Action:**

```rust
AgentAction::ExecuteTools(vec![
    ToolCall {
        id: "toolu_01XYZ",
        name: "edit_file",
        arguments: json!({
            "file_path": "src/main.rs",
            "old_text": "fn main() {\n",
            "new_text": "fn main() {\n    // Initialize logger\n"
        }),
    }
])
```

**关键数据变化:**

- `conversation.messages` 增加了 Assistant 的消息(含 `tool_calls`)
- `AgentState` 从 `CallingLlm` → `ProcessingLlmResponse` → `ExecutingTools`
- `executions` 初始化为 `[Pending { call_id: "toolu_01XYZ", tool_name: "edit_file" }]`

**下一步:** CLI 收到 `AgentAction::ExecuteTools`,需要执行工具

---

### [N10] CLI 执行 edit_file 工具

**代码路径:** CLI 调用 `ToolRegistry::invoke()`

**输入数据:**

```rust
let tool_call = ToolCall {
    id: "toolu_01XYZ",
    name: "edit_file",
    arguments: json!({
        "file_path": "src/main.rs",
        "old_text": "fn main() {\n",
        "new_text": "fn main() {\n    // Initialize logger\n"
    }),
};
```

**ToolRegistry 逻辑:** (见 ch06 § 6.3)

```rust
pub async fn invoke(&self, tool_call: &ToolCall, ctx: &ToolContext) -> Result<serde_json::Value, ToolError> {
    let tool = self.tools.get(&tool_call.name).ok_or_else(|| ToolError::NotFound(tool_call.name.clone()))?;
    tool.invoke(tool_call.arguments.clone(), ctx).await
}
```

**EditFileTool::invoke() 逻辑:** (见 ch06 § 6.4.2)

```rust
async fn invoke(&self, args: serde_json::Value, ctx: &ToolContext) -> Result<serde_json::Value, ToolError> {
    // 1. 解析参数
    let file_path: String = args["file_path"].as_str().ok_or(...)?.to_string();
    let old_text: String = args["old_text"].as_str().ok_or(...)?.to_string();
    let new_text: String = args["new_text"].as_str().ok_or(...)?.to_string();

    // 2. 路径安全检查
    let safe_path = normalize_path(&ctx.workspace_root, &file_path)?;

    // 3. 读取文件
    let content = tokio::fs::read_to_string(&safe_path).await?;

    // 4. 替换文本
    if !content.contains(&old_text) {
        return Err(ToolError::OldTextNotFound(old_text));
    }
    let new_content = content.replacen(&old_text, &new_text, 1);

    // 5. 写回文件
    tokio::fs::write(&safe_path, new_content).await?;

    // 6. 返回成功
    Ok(json!({
        "success": true,
        "file_path": file_path,
        "message": "File edited successfully"
    }))
}
```

**执行结果:**

```rust
ToolExecutionOutcome::Success {
    call_id: "toolu_01XYZ".to_string(),
    output: json!({
        "success": true,
        "file_path": "src/main.rs",
        "message": "File edited successfully"
    }),
}
```

**文件变化:** `src/main.rs` 被修改:

```diff
  fn main() {
+     // Initialize logger
      println!("Hello, world!");
  }
```

**下一步:** CLI 发送 `AgentEvent::ToolCompleted`

---

### [N11] Agent 收到 ToolCompleted 事件

**事件:**

```rust
AgentEvent::ToolCompleted {
    call_id: "toolu_01XYZ",
    outcome: ToolExecutionOutcome::Success {
        call_id: "toolu_01XYZ",
        output: json!({
            "success": true,
            "file_path": "src/main.rs",
            "message": "File edited successfully"
        }),
    },
}
```

**当前状态:**

```rust
AgentState::ExecutingTools {
    conversation: ConversationContext { ... },
    executions: vec![
        ToolExecutionStatus::Pending {
            call_id: "toolu_01XYZ",
            tool_name: "edit_file",
        }
    ],
}
```

**处理逻辑:** (见 `agent.rs:288`)

```rust
(AgentState::ExecutingTools { conversation, executions }, AgentEvent::ToolCompleted { call_id, outcome }) => {
    // 1. 更新 executions 中对应的状态
    for exec in executions.iter_mut() {
        if exec.call_id() == call_id {
            let tool_name = exec.tool_name().to_string();
            *exec = ToolExecutionStatus::Completed {
                call_id: call_id.clone(),
                tool_name,
                started_at: Instant::now(),
                completed_at: Instant::now(),
                outcome: outcome.clone(),
            };
        }
    }

    // 2. 检查是否所有工具都执行完毕
    let all_complete = executions.iter().all(|e| e.is_completed());
    if all_complete {
        // 3. 构造 Tool 结果消息
        let mut tool_messages = Vec::new();
        for exec in executions.iter() {
            if let ToolExecutionStatus::Completed { call_id, outcome, .. } = exec {
                let content = match outcome {
                    ToolExecutionOutcome::Success { output, .. } => {
                        serde_json::to_string(output).unwrap_or_else(|_| "{}".to_string())
                    }
                    ToolExecutionOutcome::Error { error, .. } => format!("Error: {error}"),
                };
                tool_messages.push(Message {
                    role: Role::Tool,
                    content,
                    tool_call_id: Some(call_id.clone()),
                    name: None,
                    tool_calls: Vec::new(),
                });
            }
        }

        // 4. 追加 Tool 结果到对话历史
        let mut conv = conversation.clone();
        conv.messages.extend(tool_messages);

        // 5. 构造下一次 LLM 请求
        let request = LlmRequest {
            model: self.config.model_name.clone(),
            messages: conv.messages.clone(),
            tools: self.tools.clone(),
            max_tokens: Some(self.config.max_tokens),
            temperature: self.config.temperature,
        };

        // 6. 检查是否有 mutating tools (edit_file/bash)
        if has_mutating_tools(executions) {
            let completed_tools = extract_completed_tools(executions);
            self.state = AgentState::PostToolsHook {
                conversation: conv,
                pending_llm_request: request,
                completed_tools: completed_tools.clone(),
            };
            AgentAction::RunPostToolsHook { completed_tools }
        } else {
            self.state = AgentState::CallingLlm {
                conversation: conv,
                retries: 0,
            };
            AgentAction::SendLlmRequest(request)
        }
    } else {
        AgentAction::WaitForInput // 还有工具未完成
    }
}
```

**关键判断:** `has_mutating_tools(executions)` 返回 `true`,因为 `edit_file` 在 `MUTATING_TOOLS` 列表中

**状态转换:**

```
ExecutingTools → PostToolsHook
```

**返回的 Action:**

```rust
AgentAction::RunPostToolsHook {
    completed_tools: vec![
        CompletedToolInfo {
            tool_name: "edit_file".to_string(),
            succeeded: true,
        }
    ],
}
```

**关键数据变化:**

- `conversation.messages` 增加了 `Role::Tool` 消息:
  ```rust
  Message {
      role: Role::Tool,
      content: r#"{"success":true,"file_path":"src/main.rs","message":"File edited successfully"}"#,
      tool_call_id: Some("toolu_01XYZ"),
      name: None,
      tool_calls: Vec::new(),
  }
  ```
- `AgentState` 变为 `PostToolsHook`,携带 `pending_llm_request`(等待 hook 完成后发送)

**下一步:** CLI 收到 `AgentAction::RunPostToolsHook`,执行 post-tools hook

---

### [N12] CLI 执行 PostToolsHook (auto-commit)

**Hook 逻辑:** (假设在 `crates/loom-cli/src/hooks/auto_commit.rs`)

```rust
pub async fn run_auto_commit(completed_tools: &[CompletedToolInfo], workspace: &Path) -> Result<bool, HookError> {
    // 1. 检查是否有成功的 mutating tools
    let has_mutations = completed_tools.iter().any(|t| {
        (t.tool_name == "edit_file" || t.tool_name == "bash") && t.succeeded
    });

    if !has_mutations {
        return Ok(false); // 没有需要提交的变更
    }

    // 2. 检查 Git 仓库状态
    let status_output = Command::new("git")
        .args(["status", "--porcelain"])
        .current_dir(workspace)
        .output()
        .await?;

    if status_output.stdout.is_empty() {
        return Ok(false); // 没有变更
    }

    // 3. Git add 所有变更
    Command::new("git")
        .args(["add", "."])
        .current_dir(workspace)
        .output()
        .await?;

    // 4. Git commit
    let commit_msg = "Auto-commit: edit_file changes";
    Command::new("git")
        .args(["commit", "-m", commit_msg])
        .current_dir(workspace)
        .output()
        .await?;

    Ok(true) // 已执行 auto-commit
}
```

**执行结果:**

```bash
$ git status --porcelain
 M src/main.rs

$ git add .
$ git commit -m "Auto-commit: edit_file changes"
[main abc1234] Auto-commit: edit_file changes
 1 file changed, 1 insertion(+)
```

**返回值:** `action_taken = true`

**下一步:** CLI 发送 `AgentEvent::PostToolsHookCompleted { action_taken: true }`

---

### [N13] Agent 收到 PostToolsHookCompleted 事件

**事件:**

```rust
AgentEvent::PostToolsHookCompleted { action_taken: true }
```

**当前状态:**

```rust
AgentState::PostToolsHook {
    conversation: ConversationContext { ... },
    pending_llm_request: LlmRequest { ... },
    completed_tools: vec![...],
}
```

**处理逻辑:** (见 `agent.rs:377`)

```rust
(AgentState::PostToolsHook { conversation, pending_llm_request, .. }, AgentEvent::PostToolsHookCompleted { action_taken }) => {
    debug!(action_taken = action_taken, "post-tools hook completed");
    let conv = conversation.clone();
    let request = pending_llm_request.clone();
    self.state = AgentState::CallingLlm {
        conversation: conv,
        retries: 0,
    };
    AgentAction::SendLlmRequest(request)
}
```

**状态转换:**

```
PostToolsHook → CallingLlm
```

**返回的 Action:**

```rust
AgentAction::SendLlmRequest(LlmRequest {
    model: "claude-sonnet-4.5",
    messages: [
        Message { role: User, content: "请在 src/main.rs ..." },
        Message { role: Assistant, content: "好的,我来帮你编辑文件", tool_calls: [...] },
        Message { role: Tool, content: r#"{"success":true,...}"#, tool_call_id: Some("toolu_01XYZ") },
    ],
    tools: [...],
    max_tokens: Some(8192),
    temperature: 0.7,
})
```

**关键数据变化:**

- `conversation.messages` 现在包含 **3 条消息** (User → Assistant → Tool)
- 再次进入 `CallingLlm` 状态,准备获取 LLM 的最终响应

**下一步:** CLI 再次调用 `ProxyLlmClient.complete_streaming()`,发送带 Tool 结果的请求给 LLM

---

### [N14] LLM 返回最终响应 (无 tool_calls)

**Anthropic API 请求:** (包含 3 条消息)

```json
{
  "model": "claude-sonnet-4.5-20250929",
  "messages": [
    { "role": "user", "content": "请在 src/main.rs ..." },
    {
      "role": "assistant",
      "content": [
        { "type": "text", "text": "好的,我来帮你编辑文件" },
        {
          "type": "tool_use",
          "id": "toolu_01XYZ",
          "name": "edit_file",
          "input": {
            "file_path": "src/main.rs",
            "old_text": "fn main() {\n",
            "new_text": "fn main() {\n    // Initialize logger\n"
          }
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01XYZ",
          "content": "{\"success\":true,\"file_path\":\"src/main.rs\",\"message\":\"File edited successfully\"}"
        }
      ]
    }
  ],
  "stream": true
}
```

**Anthropic SSE 响应:**

```
event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"已成功"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"在 src/main.rs 的第 10 行后添加了注释"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":",并自动提交到 Git"}}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"}}

event: message_stop
data: {"type":"message_stop"}
```

**解析为 LlmEvent:**

```rust
LlmEvent::TextDelta { content: "已成功" }
LlmEvent::TextDelta { content: "在 src/main.rs 的第 10 行后添加了注释" }
LlmEvent::TextDelta { content: ",并自动提交到 Git" }
LlmEvent::Completed(LlmResponse {
    message: Message {
        role: Role::Assistant,
        content: "已成功在 src/main.rs 的第 10 行后添加了注释,并自动提交到 Git",
        tool_calls: Vec::new(), // 没有 tool_calls
        tool_call_id: None,
        name: None,
    },
    stop_reason: Some("end_turn"),
})
```

**Agent 处理:**

1. **TextDelta 事件:** 逐条显示文本给用户
2. **Completed 事件:**
   ```rust
   (AgentState::CallingLlm { conversation, .. }, AgentEvent::LlmEvent(LlmEvent::Completed(response))) => {
       let mut conv = conversation.clone();
       conv.messages.push(response.message.clone());
       self.state = AgentState::ProcessingLlmResponse { conversation: conv, response };
       self.process_llm_response()
   }
   ```
3. **process_llm_response():**
   ```rust
   // response.message.tool_calls.is_empty() == true
   self.state = AgentState::WaitingForUserInput { conversation };
   AgentAction::WaitForInput
   ```

**状态转换:**

```
CallingLlm → ProcessingLlmResponse → WaitingForUserInput
```

**最终状态:**

```rust
AgentState::WaitingForUserInput {
    conversation: ConversationContext {
        id: Uuid::new_v7(),
        messages: [
            Message { role: User, content: "请在 src/main.rs ..." },
            Message { role: Assistant, content: "好的...", tool_calls: [...] },
            Message { role: Tool, content: r#"{"success":true,...}"#, tool_call_id: Some("toolu_01XYZ") },
            Message { role: Assistant, content: "已成功在 src/main.rs 的第 10 行后添加了注释,并自动提交到 Git", tool_calls: [] },
        ],
    },
}
```

**关键数据变化:**

- `conversation.messages` 现在包含 **4 条消息** (完整的对话历史)
- `AgentState` 回到 `WaitingForUserInput`,等待下一次用户输入
- CLI 显示最终响应:"已成功在 src/main.rs 的第 10 行后添加了注释,并自动提交到 Git"

---

### [N15] 对话完成,等待下一次输入

**最终状态:** Agent 回到 `WaitingForUserInput`,对话历史完整保存

**用户看到的结果:**

```
You: 请在 src/main.rs 的第 10 行后面添加一行注释 "// Initialize logger",然后提交 Git

Loom: 好的,我来帮你编辑文件
[Tool: edit_file]
已成功在 src/main.rs 的第 10 行后添加了注释,并自动提交到 Git
```

**文件系统状态:**

- `src/main.rs` 已修改
- Git 仓库有新的 commit: `abc1234 Auto-commit: edit_file changes`

**对话历史持久化:** (见 ch08,暂不展开)

---

## 7.4 数据流总览:Mermaid 序列图

```mermaid
sequenceDiagram
    participant User as 用户 (CLI TUI)
    participant Agent as Agent 状态机
    participant LlmClient as ProxyLlmClient
    participant Server as loom-server
    participant Anthropic as Anthropic API
    participant ToolReg as ToolRegistry
    participant Hook as PostToolsHook

    User->>Agent: [N1-N2] UserInput("请编辑 src/main.rs...")
    Agent->>Agent: [N3] WaitingForUserInput → CallingLlm
    Agent-->>User: AgentAction::SendLlmRequest

    User->>LlmClient: [N4] complete_streaming(LlmRequest)
    LlmClient->>Server: POST /llm/v1/messages
    Server->>Anthropic: [N5] POST /v1/messages (stream=true)

    Anthropic-->>Server: [N6] SSE: TextDelta("好的,我来...")
    Server-->>LlmClient: SSE 转发
    LlmClient-->>User: LlmEvent::TextDelta
    User->>Agent: [N7] LlmEvent::TextDelta
    Agent-->>User: AgentAction::DisplayMessage("好的...")

    Anthropic-->>Server: [N8] SSE: ToolCallDelta(edit_file)
    Server-->>LlmClient: SSE 转发
    LlmClient-->>User: LlmEvent::ToolCallDelta
    User->>Agent: LlmEvent::ToolCallDelta
    Agent-->>User: AgentAction::WaitForInput

    Anthropic-->>Server: [N9] SSE: Completed(tool_calls)
    Server-->>LlmClient: SSE 转发
    LlmClient-->>User: LlmEvent::Completed
    User->>Agent: LlmEvent::Completed
    Agent->>Agent: CallingLlm → ProcessingLlmResponse → ExecutingTools
    Agent-->>User: AgentAction::ExecuteTools([edit_file])

    User->>ToolReg: [N10] invoke(edit_file)
    ToolReg->>ToolReg: normalize_path() 路径安全检查
    ToolReg->>ToolReg: 读取 src/main.rs
    ToolReg->>ToolReg: 替换文本
    ToolReg->>ToolReg: 写回文件
    ToolReg-->>User: ToolExecutionOutcome::Success

    User->>Agent: [N11] ToolCompleted(success)
    Agent->>Agent: ExecutingTools → PostToolsHook
    Agent-->>User: AgentAction::RunPostToolsHook

    User->>Hook: [N12] run_auto_commit()
    Hook->>Hook: git add .
    Hook->>Hook: git commit
    Hook-->>User: action_taken=true

    User->>Agent: [N13] PostToolsHookCompleted(true)
    Agent->>Agent: PostToolsHook → CallingLlm
    Agent-->>User: AgentAction::SendLlmRequest (带 Tool 结果)

    User->>LlmClient: complete_streaming(LlmRequest + Tool 结果)
    LlmClient->>Server: POST /llm/v1/messages
    Server->>Anthropic: POST /v1/messages (3 条消息)

    Anthropic-->>Server: [N14] SSE: TextDelta("已成功...")
    Server-->>LlmClient: SSE 转发
    LlmClient-->>User: LlmEvent::TextDelta
    User->>Agent: LlmEvent::TextDelta
    Agent-->>User: AgentAction::DisplayMessage("已成功...")

    Anthropic-->>Server: SSE: Completed(no tool_calls)
    Server-->>LlmClient: SSE 转发
    LlmClient-->>User: LlmEvent::Completed
    User->>Agent: LlmEvent::Completed
    Agent->>Agent: CallingLlm → ProcessingLlmResponse → WaitingForUserInput
    Agent-->>User: [N15] AgentAction::WaitForInput
```

---

## 7.5 关键数据结构的生命周期

### 7.5.1 ConversationContext 的演进

`ConversationContext` 在整个流程中贯穿始终,但 `messages` 字段逐步增长:

| 节点 | messages 长度 | 内容 |
|------|--------------|------|
| [N1-N2] | 0 → 1 | 添加 User 消息 |
| [N3] | 1 | 状态机克隆 `conversation` |
| [N9] | 1 → 2 | 添加 Assistant 消息(含 tool_calls) |
| [N11] | 2 → 3 | 添加 Tool 结果消息 |
| [N14] | 3 → 4 | 添加最终 Assistant 消息(无 tool_calls) |

**关键设计:** `ConversationContext` 在每次状态转换时被 **克隆**,确保状态不可变性(见 ch03 § 3.5.3)

### 7.5.2 AgentState 的完整路径

```
WaitingForUserInput [N2]
  ↓ UserInput
CallingLlm(retries=0) [N3]
  ↓ LlmEvent::Completed
ProcessingLlmResponse [N9]
  ↓ (has tool_calls)
ExecutingTools [N9]
  ↓ ToolCompleted (all done, mutating=true)
PostToolsHook [N11]
  ↓ PostToolsHookCompleted
CallingLlm(retries=0) [N13]
  ↓ LlmEvent::Completed
ProcessingLlmResponse [N14]
  ↓ (no tool_calls)
WaitingForUserInput [N14]
```

**关键观察:**

- `CallingLlm` 出现了 **2 次**(第一次获取 tool_calls,第二次获取最终响应)
- `PostToolsHook` 只在检测到 mutating tools 时出现

### 7.5.3 LlmRequest 的两次构造

**第一次 LlmRequest** (在 [N3]):

```rust
LlmRequest {
    model: "claude-sonnet-4.5",
    messages: [
        Message { role: User, content: "请编辑..." }
    ],
    tools: [ToolDefinition { name: "edit_file", ... }],
    max_tokens: Some(8192),
    temperature: 0.7,
}
```

**第二次 LlmRequest** (在 [N13]):

```rust
LlmRequest {
    model: "claude-sonnet-4.5",
    messages: [
        Message { role: User, content: "请编辑..." },
        Message { role: Assistant, content: "好的...", tool_calls: [...] },
        Message { role: Tool, content: "{\"success\":true,...}", tool_call_id: Some("toolu_01XYZ") },
    ],
    tools: [ToolDefinition { name: "edit_file", ... }],
    max_tokens: Some(8192),
    temperature: 0.7,
}
```

**关键差异:** 第二次请求包含了 Tool 的执行结果,LLM 根据这个结果生成最终响应

---

## 7.6 错误处理路径 (简要说明)

如果在上述流程中发生错误,数据流会如何变化?

### 7.6.1 LLM 调用失败 (网络超时)

**发生位置:** [N4-N5] 调用 Anthropic API 时超时

**错误事件:**

```rust
AgentEvent::LlmEvent(LlmEvent::Error(LlmError::Timeout))
```

**状态转换:** (见 `agent.rs:215`)

```
CallingLlm → Error (origin=Llm, retries=1)
```

**Action:**

```rust
AgentAction::WaitForInput // 等待 RetryTimeoutFired 事件
```

**重试逻辑:**

- CLI 设置定时器(例如 5 秒)
- 定时器触发后发送 `AgentEvent::RetryTimeoutFired`
- Agent 重新进入 `CallingLlm` 状态,重试请求

**最大重试次数:** `max_retries = 3` (配置在 `AgentConfig` 中)

### 7.6.2 Tool 执行失败 (文件不存在)

**发生位置:** [N10] `edit_file` 执行时文件路径错误

**错误结果:**

```rust
ToolExecutionOutcome::Error {
    call_id: "toolu_01XYZ",
    error: ToolError::Io(std::io::Error::new(ErrorKind::NotFound, "src/main.rs")),
}
```

**状态转换:**

```
ExecutingTools (仍在该状态,但 executions 中的状态变为 Completed(Error))
```

**Tool 消息:**

```rust
Message {
    role: Role::Tool,
    content: "Error: file not found: src/main.rs",
    tool_call_id: Some("toolu_01XYZ"),
    name: None,
    tool_calls: Vec::new(),
}
```

**LLM 行为:** 收到错误消息后,LLM 可能:

1. 重新调用 tool(例如先调用 `read_file` 检查文件是否存在)
2. 向用户解释错误并请求帮助

---

## 7.7 性能与优化考虑

### 7.7.1 SSE 流式响应的价值

**对比:** 非流式 vs 流式

| 指标 | 非流式 (Completed 一次性返回) | 流式 (TextDelta 增量返回) |
|------|------------------------------|--------------------------|
| **TTFB** (Time To First Byte) | ~2-5 秒 | ~200-500 毫秒 |
| **用户感知延迟** | 高(等待完整响应) | 低(立即看到输出) |
| **网络开销** | 相同 | 相同 |
| **实现复杂度** | 低 | 高(需要 SSE 解析) |

**结论:** 流式响应显著改善用户体验,值得额外的实现复杂度

### 7.7.2 对话历史的内存开销

**问题:** `ConversationContext.messages` 在每次状态转换时被克隆,可能导致内存浪费

**当前实现:**

```rust
let new_conversation = conversation.clone(); // O(n) 复制
```

**优化方案:** (未实现,仅讨论)

1. **Arc 共享:** 使用 `Arc<Vec<Message>>` + 写时复制
2. **持久化数据结构:** 使用 `im::Vector` (immutable vector)

**权衡:** 当前对话历史通常 < 100 条消息,克隆开销可接受(< 1ms)

### 7.7.3 工具并行执行 (未实现)

**当前实现:** `ExecutingTools` 状态携带 `Vec<ToolExecutionStatus>`,但工具是 **串行执行**

**潜在优化:** 如果多个 tool_calls 之间无依赖,可以并行执行

**示例:**

```rust
// LLM 返回 2 个独立的 tool_calls
tool_calls: [
    ToolCall { name: "read_file", arguments: { "file_path": "a.txt" } },
    ToolCall { name: "read_file", arguments: { "file_path": "b.txt" } },
]
```

**并行执行:**

```rust
let results = tokio::join!(
    tool_registry.invoke(&tool_calls[0], &ctx),
    tool_registry.invoke(&tool_calls[1], &ctx),
);
```

**收益:** 延迟从 `T1 + T2` 降低到 `max(T1, T2)`

---

## 7.8 本章小结

### 7.8.1 数据流的关键特征

1. **单向流动** — 数据从用户输入到 LLM 到 Tool,再回到 LLM,最后返回用户,没有"循环依赖"
2. **状态驱动** — 每一步数据转换都伴随 `AgentState` 的显式转换
3. **不可变性** — `ConversationContext` 在每次转换时被克隆,确保历史不被意外修改
4. **类型安全** — 从 `String` 到 `Message` 到 `LlmRequest` 到 SSE 流,每一跳都有明确的类型

### 7.8.2 与前面章节的呼应

| 章节 | 在本章的体现 |
|------|------------|
| ch02 | `Message`、`ToolCall`、`LlmRequest/Response` 的实际使用 |
| ch03 | 状态机的完整转换路径 (`WaitingForUserInput` → ... → `WaitingForUserInput`) |
| ch04 | `ProxyLlmClient` 的 SSE 解析,`LlmEvent` 的流式生成 |
| ch06 | `ToolRegistry::invoke()` 和 `EditFileTool::invoke()` 的执行细节 |

### 7.8.3 未覆盖的主题

以下主题在本章中仅简要提及,将在后续章节详细讨论:

- **ch08 Thread 持久化** — 对话历史如何保存到 SQLite 并同步到服务端
- **ch09 Weaver 远程执行** — 工具在 K8s Pod 中执行时的数据流变化
- **ch10 可观测性** — 如何通过 tracing 追踪整个数据流

---

## 7.9 质检清单

在完成本章后,请验证以下内容:

- [x] **完整性** — 从用户输入到最终响应的 15 个节点是否全部覆盖?
- [x] **准确性** — 每个节点的数据示例是否与实际代码一致?
- [x] **引用正确** — 对 ch02-ch06 的引用是否准确(章节号和小节号)?
- [x] **序列图** — Mermaid 图是否正确反映了节点间的交互顺序?
- [x] **错误处理** — 是否讨论了至少 2 种错误场景?
- [x] **性能考虑** — 是否分析了流式响应和对话历史克隆的性能影响?
- [x] **未来扩展** — 是否标注了"未实现"的优化方向(如工具并行执行)?

---

**下一章预告:** ch08 将讨论 **Thread 持久化**,展示 `ConversationContext` 如何通过 UUID7 ID、本地 SQLite 存储和服务端同步实现"离线优先"的对话历史管理。
