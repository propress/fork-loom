# 第 3 章｜数据流全景：一次请求如何穿过整个系统

前两章我们已经得到两样东西：

- 第 1 章：系统地图（知道有哪些部件）
- 第 2 章：主干调用顺序（知道先后关系）

这一章补上第三个维度：

> **数据在每个节点“长什么样”，经过边界后如何变形。**

只要你把这件事看清，后续读状态机、工具系统、LLM 代理、线程持久化都会快很多。

---

## 3.1 追踪样本与边界定义

本章沿用默认 CLI 主路径（`loom` 无子命令），追踪“一轮输入”在以下边界间移动：

1. **终端输入边界**：stdin 文本进入内存
2. **编排边界**：`Message` / `LlmRequest` 组装
3. **网络边界**：HTTP JSON 请求 + SSE 流式响应
4. **工具边界**：`ToolCall` 到 `ToolExecutionOutcome`
5. **持久化边界**：`Thread` 序列化为本地 JSON，并可异步同步服务器

核心函数锚点（已验证）：

- `/home/runner/work/fork-loom/fork-loom/crates/loom-cli/src/main.rs::run_repl()`
- `/home/runner/work/fork-loom/fork-loom/crates/loom-common-core/src/message.rs`
- `/home/runner/work/fork-loom/fork-loom/crates/loom-common-core/src/llm.rs`
- `/home/runner/work/fork-loom/fork-loom/crates/loom-server-llm-proxy/src/client.rs`
- `/home/runner/work/fork-loom/fork-loom/crates/loom-server-llm-proxy/src/stream.rs`
- `/home/runner/work/fork-loom/fork-loom/crates/loom-common-thread/src/model.rs`
- `/home/runner/work/fork-loom/fork-loom/crates/loom-common-thread/src/store.rs`
- `/home/runner/work/fork-loom/fork-loom/crates/loom-common-thread/src/sync.rs`

---

## 3.2 一张图先看完：端到端数据形态演化

```mermaid
flowchart TD
  A[stdin: String] --> B[Message::user]
  B --> C[messages: Vec<Message>]
  C --> D[LlmRequest {model,messages,tools}]
  D --> E[HTTP POST /proxy/{provider}/stream]
  E --> F[SSE bytes stream]
  F --> G[LlmStreamEvent wire enum]
  G --> H[LlmEvent core enum]
  H --> I[assistant_content + tool_calls]
  I --> J[Message::assistant_with_tool_calls]
  I --> K[for each ToolCall -> execute_tool]
  K --> L[ToolExecutionOutcome]
  L --> M[Message::tool]
  J --> N[thread.conversation.messages(MessageSnapshot)]
  M --> N
  N --> O[Thread::touch + agent_state]
  O --> P[LocalThreadStore save JSON]
  P --> Q[SyncingThreadStore async upsert server]
```

这张图重点不是“调用谁”，而是“**同一轮里有几套数据结构在并行推进**”。

---

## 3.3 输入进入系统：`String` -> `Message`

在 `run_repl()` 中，用户输入先经历最小规整：

1. `reader.read_line(&mut input)`
2. `input.trim()`
3. 空串直接 `continue`
4. `Message::user(input)`

此时数据形态从“原始终端字符串”变为统一消息对象：

```rust
Message {
  role: Role::User,
  content: String,
  tool_call_id: None,
  name: None,
  tool_calls: vec![],
}
```

对应定义位于：
`/home/runner/work/fork-loom/fork-loom/crates/loom-common-core/src/message.rs`

这一步的价值是：后续 LLM、工具、持久化都不再直接处理“裸字符串”，而是处理带角色语义的结构化消息。

---

## 3.4 编排层双写：运行时上下文 vs 可恢复快照

`run_repl()` 在用户消息进入后会做两次写入：

- `messages.push(user_message.clone())`（运行时 prompt 上下文）
- `thread.conversation.messages.push(MessageSnapshot::from(&user_message))`（持久化快照）

这是 Loom 主链路非常关键的“并行视图”设计：

- `Vec<Message>`：偏执行（给下一次 LLM 请求）
- `Vec<MessageSnapshot>`：偏存档（用于恢复、检索、分享、同步）

二者内容高度对应，但类型不同、职责不同。

---

## 3.5 请求出站：`Vec<Message>` -> `LlmRequest` -> HTTP JSON

同一轮里请求对象通过 builder 组装：

- `LlmRequest::new("default")`
- `.with_messages(messages.clone())`
- `.with_tools(tool_definitions.to_vec())`

`LlmRequest` 结构（`llm.rs`）关键字段：

- `model: String`
- `messages: Vec<Message>`
- `tools: Vec<ToolDefinition>`
- `max_tokens: Option<u32>`
- `temperature: Option<f32>`

之后在 `ProxyLlmClient::complete_streaming()` 中被 `.json(&request)` 序列化，通过：

- `POST {base}/proxy/{provider}/stream`

发送到服务端代理。

这一段是**强边界 1：进程内 Rust 类型 -> 进程间 JSON 协议**。

---

## 3.6 入站流：SSE bytes -> Wire Event -> Core Event

响应不是一次性 JSON，而是 SSE 流。链路是：

1. `response.bytes_stream()` 得到字节流
2. `ProxyLlmStream` 逐块缓冲，按 `\n\n` 切分 event block
3. 解析 `event: llm` + `data: {...}`
4. 反序列化为 `LlmStreamEvent`（wire）
5. 转换为 `LlmEvent`（core）

也就是说，流式入站至少过两层类型：

- 传输层枚举：`LlmStreamEvent`
- 核心层枚举：`LlmEvent`

`run_repl()` 消费的是核心层事件：

- `TextDelta`：拼接 assistant 文本
- `ToolCallDelta`：记录增量工具调用片段
- `Completed(response)`：拿到完整 `tool_calls`
- `Error(e)`：错误事件

这是**强边界 2：网络字节流 -> 语义事件流**。

---

## 3.7 助手结果聚合：增量文本与终态响应合流

`run_repl()` 里有两个聚合容器：

- `assistant_content: String`
- `tool_calls: Vec<ToolCall>`

在 `TextDelta` 期间实时拼接内容并输出；在 `Completed(response)` 阶段拿到最终 `response.tool_calls`，并可能用 `response.message.content` 覆盖文本。

然后落成一条稳定消息：

- `Message::assistant_with_tool_calls(&assistant_content, tool_calls.clone())`

并同步写入 `thread.conversation.messages` 的 `MessageSnapshot` 形态（`tool_calls` 转为 `ToolCallSnapshot`）。

所以这里不是“事件即存档”，而是“**事件流 -> 聚合态 -> 消息快照**”。

---

## 3.8 工具回路：`ToolCall` -> `ToolExecutionOutcome` -> `Message::tool`

如果 `tool_calls` 非空，流程按调用逐个推进：

1. `execute_tool(tool_registry, tool_call, tool_ctx).await`
2. 得到 `ToolExecutionOutcome::Success|Error`
3. 统一转成字符串内容（错误会前缀 `Error: ...`）
4. 构造 `Message::tool(tool_call_id, tool_name, tool_result)`
5. 写入运行时 `messages`
6. 写入 `thread.conversation.messages`（role=Tool）

这一步把“副作用执行结果”重新结构化成对话消息，供下一轮 LLM 使用。

本质上，工具回路是一个协议适配：

- 外部动作结果（任意输出）
- -> 转换为可对话建模的 Tool role message

---

## 3.9 轮次收尾：会话状态 + Git 快照 + 持久化

一轮结束时，`run_repl()` 做以下收尾：

- `thread.agent_state = WaitingForUserInput`
- `snapshot_git_state(thread, workspace)`
- `thread.touch()`（更新时间 + version++）
- `thread_store.save(thread).await`

`Thread` 的持久化载体（`model.rs`）包含：

- 会话元信息（id/version/timestamps/workspace/provider/model）
- Git 上下文（branch/remote/commit/dirty/commits）
- `conversation: ConversationSnapshot`
- `agent_state: AgentStateSnapshot`
- 可见性与同步控制（visibility/is_private/is_shared_with_support）

这一步是**强边界 3：内存运行态 -> 可恢复历史状态**。

---

## 3.10 本地保存与异步同步：`save` 的双阶段语义

在默认 CLI 里，`ThreadStore` 实际通常是 `SyncingThreadStore`（包裹 `LocalThreadStore` + `ThreadSyncClient`）。

`SyncingThreadStore::save()` 的语义是：

1. **先本地保存**：`self.local.save(thread).await?`
2. 若 `thread.is_private == true`，直接跳过服务端同步
3. 否则异步 `tokio::spawn` 调 `upsert_thread` 到 server
4. 若配置了 pending queue，失败会入队等待后续重试

这意味着 CLI 主流程的“保存成功”首先保证本地 durability；远端一致性是最终一致（异步补偿）。

而 `save_and_sync()`（用于某些命令如 share）才是阻塞式等待远端结果。

---

## 3.11 一轮数据流的“最小字段账本”

如果你只记最小关键字段，记这组：

1. 用户输入阶段：`Message.role=user`, `Message.content`
2. 请求阶段：`LlmRequest.messages`, `LlmRequest.tools`
3. 响应阶段：`LlmEvent::TextDelta.content`, `Completed.tool_calls`
4. 工具阶段：`ToolCall.{id,tool_name,arguments_json}` + `ToolExecutionOutcome`
5. 存档阶段：`MessageSnapshot` 序列 + `Thread.version/updated_at`

这组字段是定位“为什么上下文丢了/工具没回注/恢复不完整”的第一排检查点。

---

## 3.12 常见误区：把“流”当“最终态”

读这段代码最容易犯的错：

- 看到 `TextDelta` 就以为那是最终 assistant 消息
- 看到某个 tool delta 就以为工具参数已最终确定

实际实现是：

- delta 是增量信号
- 最终语义以 `Completed(response)` + 聚合结果为准
- 持久化写入的是聚合后的稳定快照

这一点如果理解错，后面你看状态机和重试逻辑会一直混乱。

---

## 3.13 小结：本章给后续章节提供什么“骨架”

这一章把“同一轮请求”的数据形态切成了五段：

1. 输入结构化
2. 请求序列化
3. 流式事件语义化
4. 工具结果消息化
5. 会话状态持久化

后续章节就沿这五段深入：

- 第 4 章：为什么状态迁移可控（状态机）
- 第 5 章：工具协议与执行细节
- 第 6 章：代理层与 provider 边界
- 第 7 章：Thread/Store/Sync 的一致性策略

---

### 质检报告

**覆盖完整性**
- [x] 覆盖输入、请求、响应、工具、持久化五个阶段
- [x] 覆盖跨进程边界（HTTP/SSE）与跨存储边界（内存/JSON/同步）

**源码可追溯性**
- [x] 关键结构均可在已标注 crate 中定位
- [x] 关键函数均与默认 CLI 主流程一致

**表达可用性**
- [x] 给出端到端数据形态图
- [x] 给出最小字段账本用于排障
- [x] 给出后续章节依赖关系
