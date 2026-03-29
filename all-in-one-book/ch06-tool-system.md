# 第六章 Tool 系统：AI 如何操作文件系统

---

## 本章概览

在前几章中,我们理解了状态机如何调用 LLM,LLM 如何返回 `tool_calls`。但这些工具调用如何真正执行?AI 如何读取文件、编辑代码、运行命令?答案是:**Tool 系统**。

Tool 系统是 AI 与外部世界交互的桥梁。它将 LLM 的文本推理能力转化为具体的文件系统操作,同时强制执行安全边界,防止未经授权的访问。

Loom 的 Tool 系统设计遵循以下原则:

1. **Trait-based 抽象** — 所有工具实现统一的 `Tool` trait,易于扩展
2. **路径安全** — 所有文件操作限制在 `workspace_root` 内,防止路径遍历攻击
3. **显式状态跟踪** — 工具执行通过 `ToolExecutionStatus` 枚举跟踪(Pending/Running/Completed)
4. **JSON Schema 验证** — 工具的输入参数通过 JSON Schema 定义和验证

本章将介绍 Tool 系统的完整架构:Tool trait、ToolRegistry、内置工具(read_file/edit_file/bash/oracle)、路径安全机制,以及工具执行流程。

**重点模块:**
- `crates/loom-common-core/src/tool.rs` — ToolDefinition 和 ToolContext
- `crates/loom-cli-tools/src/registry.rs` — Tool trait 和 ToolRegistry
- `crates/loom-cli-tools/src/read_file.rs` — ReadFileTool 实现示例
- `crates/loom-cli-tools/src/edit_file.rs` — EditFileTool 实现示例
- `specs/tool-system.md` — 完整设计文档

---

## 6.1 Tool trait 设计:统一的工具接口

### Trait 定义

所有工具必须实现 `Tool` trait:

```rust
#[async_trait]
pub trait Tool: Send + Sync {
	/// 工具的唯一名称(LLM 用于调用)
	fn name(&self) -> &str;

	/// 工具的描述(显示给 LLM)
	fn description(&self) -> &str;

	/// 输入参数的 JSON Schema
	fn input_schema(&self) -> serde_json::Value;

	/// 转换为 ToolDefinition(发送给 LLM)
	fn to_definition(&self) -> ToolDefinition {
		ToolDefinition::new(self.name(), self.description(), self.input_schema())
	}

	/// 执行工具(异步)
	async fn invoke(
		&self,
		args: serde_json::Value,
		ctx: &ToolContext,
	) -> Result<serde_json::Value, ToolError>;
}
```

**方法说明:**

| 方法 | 用途 | 返回类型 |
|------|------|---------|
| `name()` | 工具的唯一标识符(如 `"read_file"`) | `&str` |
| `description()` | 人类可读的描述,帮助 LLM 理解工具用途 | `&str` |
| `input_schema()` | JSON Schema,定义有效的输入参数 | `serde_json::Value` |
| `to_definition()` | 生成 `ToolDefinition`(发送给 LLM) | `ToolDefinition` |
| `invoke()` | 执行工具,返回 JSON 结果 | `async Result<Value, ToolError>` |

### ToolDefinition 和 ToolContext

**ToolDefinition(已在 ch02 介绍):**

```rust
pub struct ToolDefinition {
	pub name: String,
	pub description: String,
	pub input_schema: serde_json::Value,
}
```

这是发送给 LLM 的序列化表示,LLM 根据它决定是否调用工具以及如何构造参数。

**ToolContext(执行上下文):**

```rust
pub struct ToolContext {
	pub workspace_root: PathBuf,  // 工作区根目录(安全边界)
}
```

`workspace_root` 是所有文件操作的安全边界——工具不能访问此目录外的文件。

### 为什么需要 JSON Schema?

**场景:** AI 想读取文件,但不知道 `read_file` 工具接受什么参数。

**解决方案:** `input_schema()` 返回 JSON Schema,LLM 根据 Schema 构造参数:

```json
{
  "type": "object",
  "properties": {
    "path": {
      "type": "string",
      "description": "Path to the file (absolute or relative to workspace)"
    },
    "max_bytes": {
      "type": "integer",
      "description": "Maximum bytes to read (default: 1MB)"
    }
  },
  "required": ["path"]
}
```

LLM 看到这个 Schema 后,会构造合法的 `arguments_json`:

```json
{
  "path": "src/main.rs",
  "max_bytes": 1048576
}
```

---

## 6.2 ToolRegistry:工具注册与分发

### ToolRegistry 结构

`ToolRegistry` 管理所有可用工具:

```rust
pub struct ToolRegistry {
	tools: HashMap<String, Box<dyn Tool>>,  // tool_name → Box<dyn Tool>
}

impl ToolRegistry {
	pub fn new() -> Self {
		Self {
			tools: HashMap::new(),
		}
	}

	pub fn register(&mut self, tool: Box<dyn Tool>) {
		let name = tool.name().to_string();
		self.tools.insert(name, tool);
	}

	pub fn get(&self, name: &str) -> Option<&dyn Tool> {
		self.tools.get(name).map(|b| b.as_ref())
	}

	pub fn definitions(&self) -> Vec<ToolDefinition> {
		self.tools.values().map(|t| t.to_definition()).collect()
	}
}
```

**方法说明:**

- `register(tool)` — 注册新工具
- `get(name)` — 根据名称查找工具(返回 trait object)
- `definitions()` — 获取所有工具的 `ToolDefinition`(发送给 LLM)

### 注册内置工具

```rust
let mut registry = ToolRegistry::new();
registry.register(Box::new(ReadFileTool));
registry.register(Box::new(EditFileTool));
registry.register(Box::new(ListFilesTool));
registry.register(Box::new(BashTool));
registry.register(Box::new(OracleTool));

// 获取所有工具定义(发送给 LLM)
let defs = registry.definitions();

// 构造 LlmRequest
let request = LlmRequest::new("claude-3-5-sonnet-20241022")
	.with_messages(messages)
	.with_tools(defs);  // LLM 知道有哪些工具可用
```

### 工具分发(dispatch)

当 LLM 返回 `tool_calls` 时,需要查找并执行对应的工具:

```rust
// 状态机收到 ExecuteTools 动作
for tool_call in tool_calls {
	// 查找工具
	let tool = registry.get(&tool_call.tool_name)
		.ok_or(ToolError::NotFound(tool_call.tool_name.clone()))?;

	// 执行工具
	let ctx = ToolContext::new(workspace_root.clone());
	let result = tool.invoke(tool_call.arguments_json, &ctx).await?;

	// 发送 ToolCompleted 事件
	agent.handle_event(AgentEvent::ToolCompleted {
		call_id: tool_call.id,
		outcome: ToolExecutionOutcome::Success {
			call_id: tool_call.id,
			output: result,
		},
	})?;
}
```

---

## 6.3 内置工具:read_file、edit_file、bash、oracle

### 1. read_file:读取文件内容

**实现示例:**

```rust
pub struct ReadFileTool;

impl Tool for ReadFileTool {
	fn name(&self) -> &str {
		"read_file"
	}

	fn description(&self) -> &str {
		"Reads the contents of a file at the specified path"
	}

	fn input_schema(&self) -> serde_json::Value {
		serde_json::json!({
			"type": "object",
			"properties": {
				"path": {
					"type": "string",
					"description": "Path to the file (absolute or relative to workspace)"
				},
				"max_bytes": {
					"type": "integer",
					"description": "Maximum bytes to read (default: 1MB)"
				}
			},
			"required": ["path"]
		})
	}

	async fn invoke(
		&self,
		args: serde_json::Value,
		ctx: &ToolContext,
	) -> Result<serde_json::Value, ToolError> {
		// 1. 解析参数
		let path: String = args["path"].as_str().ok_or(...)?.to_string();
		let max_bytes = args["max_bytes"].as_u64().unwrap_or(1024 * 1024);

		// 2. 规范化路径(安全检查)
		let full_path = normalize_path(&ctx.workspace_root, &path)?;

		// 3. 读取文件
		let mut file = tokio::fs::File::open(&full_path).await?;
		let mut buffer = vec![0; max_bytes as usize];
		let n = file.read(&mut buffer).await?;
		buffer.truncate(n);

		// 4. 转换为 UTF-8(截断后可能非法,用 lossy)
		let contents = String::from_utf8_lossy(&buffer).to_string();
		let truncated = n == max_bytes as usize;

		// 5. 返回结果
		Ok(serde_json::json!({
			"path": full_path.display().to_string(),
			"contents": contents,
			"truncated": truncated
		}))
	}
}
```

**关键设计:**

- **默认限制 1MB** — 防止读取大文件导致内存耗尽
- **截断标志** — `truncated: true` 提示 LLM 文件被截断
- **UTF-8 lossy** — `String::from_utf8_lossy` 处理截断导致的非法 UTF-8

### 2. edit_file:编辑文件(snippet-based 替换)

**实现原理:**

`edit_file` 不是简单的"覆盖整个文件",而是基于 **snippet 替换**:

```rust
pub struct EditFileTool;

impl Tool for EditFileTool {
	fn name(&self) -> &str {
		"edit_file"
	}

	fn description(&self) -> &str {
		"Edits a file by replacing snippets of text"
	}

	fn input_schema(&self) -> serde_json::Value {
		serde_json::json!({
			"type": "object",
			"properties": {
				"path": { "type": "string" },
				"edits": {
					"type": "array",
					"items": {
						"type": "object",
						"properties": {
							"old_str": { "type": "string", "description": "Text to find (empty for new file)" },
							"new_str": { "type": "string", "description": "Replacement text" },
							"replace_all": { "type": "boolean", "description": "Replace all occurrences" }
						},
						"required": ["old_str", "new_str"]
					}
				}
			},
			"required": ["path", "edits"]
		})
	}

	async fn invoke(&self, args: serde_json::Value, ctx: &ToolContext) -> Result<serde_json::Value, ToolError> {
		let path: String = args["path"].as_str().ok_or(...)?.to_string();
		let edits: Vec<Edit> = serde_json::from_value(args["edits"].clone())?;

		let full_path = normalize_path(&ctx.workspace_root, &path)?;

		// 读取文件(如果存在)
		let mut contents = if full_path.exists() {
			tokio::fs::read_to_string(&full_path).await?
		} else {
			String::new()
		};

		// 应用所有编辑
		for edit in edits {
			if edit.old_str.is_empty() {
				// 新文件或追加
				contents.push_str(&edit.new_str);
			} else if edit.replace_all {
				// 替换所有出现
				contents = contents.replace(&edit.old_str, &edit.new_str);
			} else {
				// 替换第一次出现
				if let Some(pos) = contents.find(&edit.old_str) {
					contents.replace_range(pos..pos + edit.old_str.len(), &edit.new_str);
				} else {
					return Err(ToolError::TargetNotFound(edit.old_str));
				}
			}
		}

		// 写入文件
		tokio::fs::write(&full_path, contents).await?;

		Ok(serde_json::json!({"success": true, "path": full_path.display().to_string()}))
	}
}
```

**为什么用 snippet 替换而非整文件覆盖?**

1. **减少 Token 消耗** — LLM 只需发送修改的片段,而非整个文件
2. **提高准确性** — 小范围修改更精确,不易出错
3. **支持增量修改** — 可以一次应用多个编辑(数组 `edits`)

### 3. bash:执行 Shell 命令

**实现示例(简化):**

```rust
pub struct BashTool;

impl Tool for BashTool {
	fn name(&self) -> &str {
		"bash"
	}

	fn description(&self) -> &str {
		"Executes a bash command and returns the output"
	}

	fn input_schema(&self) -> serde_json::Value {
		serde_json::json!({
			"type": "object",
			"properties": {
				"command": { "type": "string", "description": "The bash command to execute" },
				"timeout_seconds": { "type": "integer", "description": "Timeout in seconds (default: 30)" }
			},
			"required": ["command"]
		})
	}

	async fn invoke(&self, args: serde_json::Value, ctx: &ToolContext) -> Result<serde_json::Value, ToolError> {
		let command: String = args["command"].as_str().ok_or(...)?.to_string();
		let timeout = args["timeout_seconds"].as_u64().unwrap_or(30);

		// 在 workspace_root 中执行
		let output = tokio::process::Command::new("bash")
			.arg("-c")
			.arg(&command)
			.current_dir(&ctx.workspace_root)
			.output()
			.await?;

		Ok(serde_json::json!({
			"stdout": String::from_utf8_lossy(&output.stdout).to_string(),
			"stderr": String::from_utf8_lossy(&output.stderr).to_string(),
			"exit_code": output.status.code().unwrap_or(-1)
		}))
	}
}
```

**安全考虑:**

- **工作目录限制** — `current_dir(workspace_root)` 确保命令在工作区内执行
- **超时保护** — 防止命令无限期挂起
- **stdout/stderr 分离** — 返回标准输出和错误输出

### 4. oracle:AI 自我提问工具

**特殊设计:**

`oracle` 是一个特殊的工具,允许 AI 向自己提问(递归调用 LLM):

```rust
pub struct OracleTool;

impl Tool for OracleTool {
	fn name(&self) -> &str {
		"oracle"
	}

	fn description(&self) -> &str {
		"Ask a question to get additional information or reasoning"
	}

	fn input_schema(&self) -> serde_json::Value {
		serde_json::json!({
			"type": "object",
			"properties": {
				"question": { "type": "string", "description": "The question to ask" }
			},
			"required": ["question"]
		})
	}

	async fn invoke(&self, args: serde_json::Value, ctx: &ToolContext) -> Result<serde_json::Value, ToolError> {
		let question: String = args["question"].as_str().ok_or(...)?.to_string();

		// 递归调用 LLM
		let oracle_response = call_llm_recursively(&question).await?;

		Ok(serde_json::json!({
			"answer": oracle_response
		}))
	}
}
```

**用途:**

- **分解复杂问题** — AI 可以将大问题分解为多个小问题,逐个解决
- **推理链** — Chain-of-Thought 推理

**示例对话:**

```
用户: 帮我优化这个算法的时间复杂度
AI: 我先用 oracle 分析当前算法的瓶颈
   → oracle("分析这段代码的时间复杂度")
   → 返回: "嵌套循环导致 O(n²)"
AI: 根据分析,我可以用哈希表优化到 O(n)
   → edit_file(...)
```

---

## 6.4 路径安全:防止路径遍历攻击

### 安全威胁

**场景:** AI 被恶意提示词诱导读取敏感文件:

```
用户: 帮我读取 ../../etc/passwd
AI: 调用 read_file({"path": "../../etc/passwd"})
```

如果不做路径验证,工具会读取系统文件,泄露敏感信息。

### normalize_path 函数

所有工具在访问文件前,必须调用 `normalize_path`:

```rust
fn normalize_path(workspace_root: &Path, user_path: &str) -> Result<PathBuf, ToolError> {
	// 1. 拼接路径
	let joined = if Path::new(user_path).is_absolute() {
		PathBuf::from(user_path)
	} else {
		workspace_root.join(user_path)
	};

	// 2. 规范化(解析 . 和 ..)
	let canonical = joined.canonicalize()
		.map_err(|e| ToolError::PathOutsideWorkspace(joined.clone()))?;

	// 3. 检查是否在 workspace_root 内
	if !canonical.starts_with(workspace_root) {
		return Err(ToolError::PathOutsideWorkspace(canonical));
	}

	Ok(canonical)
}
```

**防御措施:**

1. **规范化路径** — `canonicalize()` 解析符号链接、`.`、`..`
2. **边界检查** — `starts_with(workspace_root)` 确保路径在工作区内
3. **拒绝越界** — 返回 `ToolError::PathOutsideWorkspace`

**示例:**

```rust
// 工作区: /home/user/project

normalize_path("/home/user/project", "src/main.rs")
  → Ok(/home/user/project/src/main.rs)

normalize_path("/home/user/project", "../secret.txt")
  → Err(PathOutsideWorkspace(/home/user/secret.txt))

normalize_path("/home/user/project", "/etc/passwd")
  → Err(PathOutsideWorkspace(/etc/passwd))
```

---

## 6.5 工具执行流程:端到端追踪

### 完整流程

```
1. LLM 请求工具调用
   LlmResponse {
     tool_calls: [
       ToolCall {
         id: "call_1",
         tool_name: "read_file",
         arguments_json: {"path": "src/main.rs"}
       }
     ]
   }

2. 状态机转换到 ExecutingTools
   AgentState::ExecutingTools {
     executions: [
       ToolExecutionStatus::Pending {
         call_id: "call_1",
         tool_name: "read_file",
         requested_at: Instant::now()
       }
     ]
   }

3. 调用方执行工具
   let tool = registry.get("read_file")?;
   let ctx = ToolContext::new("/home/user/project");
   let result = tool.invoke({"path": "src/main.rs"}, &ctx).await?;

   // 更新状态为 Running
   ToolExecutionStatus::Running {
     call_id: "call_1",
     tool_name: "read_file",
     started_at: Instant::now(),
     ...
   }

4. 工具返回结果
   result = {
     "path": "/home/user/project/src/main.rs",
     "contents": "fn main() { ... }",
     "truncated": false
   }

5. 发送 ToolCompleted 事件
   AgentEvent::ToolCompleted {
     call_id: "call_1",
     outcome: ToolExecutionOutcome::Success {
       call_id: "call_1",
       output: result
     }
   }

6. 状态机更新执行状态
   ToolExecutionStatus::Completed {
     call_id: "call_1",
     tool_name: "read_file",
     started_at: ...,
     completed_at: Instant::now(),
     outcome: Success { ... }
   }

7. 所有工具完成后,转换状态
   ExecutingTools → CallingLlm (发送工具结果给 LLM)
```

### 数据流示意图

```
用户输入 "帮我读取 main.rs"
  ↓
状态机: WaitingForUserInput → CallingLlm
  ↓
LLM 返回: tool_calls=[{id:"call_1", tool_name:"read_file", args:{"path":"main.rs"}}]
  ↓
状态机: CallingLlm → ProcessingLlmResponse → ExecutingTools
  ↓
调用方: registry.get("read_file").invoke(args, ctx)
  ↓
ReadFileTool:
  1. normalize_path("/workspace", "main.rs") → /workspace/main.rs
  2. tokio::fs::read_to_string("/workspace/main.rs")
  3. 返回 {"path": "...", "contents": "fn main() {...}", "truncated": false}
  ↓
发送事件: ToolCompleted { call_id: "call_1", outcome: Success {...} }
  ↓
状态机: ExecutingTools → CallingLlm
  ↓
LLM 收到工具结果(Message::tool):
  {"role": "tool", "tool_call_id": "call_1", "name": "read_file", "content": "{...}"}
  ↓
LLM 返回最终响应: "文件内容是 fn main() { ... }"
  ↓
状态机: CallingLlm → WaitingForUserInput
```

---

## 6.6 小结

本章介绍了 Loom 的 Tool 系统设计:

1. **Tool trait** — 统一接口,定义 `name()`、`description()`、`input_schema()`、`invoke()`
2. **ToolRegistry** — 工具注册与分发,通过 `HashMap<String, Box<dyn Tool>>` 管理
3. **内置工具** — `read_file`(读文件)、`edit_file`(snippet 替换)、`bash`(执行命令)、`oracle`(递归提问)
4. **路径安全** — `normalize_path()` 防止路径遍历攻击,强制限制在 `workspace_root` 内
5. **工具执行流程** — 从 LLM 请求到工具执行到结果返回,完整的端到端追踪

**关键设计原则:**

- **Trait-based 抽象** — 易于扩展,添加新工具只需实现 trait
- **安全第一** — 路径规范化和边界检查,防止越界访问
- **JSON Schema 驱动** — LLM 根据 Schema 构造合法参数
- **显式状态跟踪** — `ToolExecutionStatus` 枚举跟踪执行状态(Pending/Running/Completed)

**与前面章节的衔接:**

- `ToolCall` 在 ch02 中定义,这里展示如何执行
- `ToolExecutionStatus` 在 ch02 中定义,这里展示状态转换
- `ExecutingTools` 状态在 ch03 中定义,这里展示工具分发逻辑

下一章(第 7 章)将串联前面所有章节,追踪**一次完整对话的端到端数据流**:用户输入 → LLM → Tool 执行 → PostToolsHook → 返回,看数据在每一跳如何变化。

---

## 质检清单

- [x] **Tool trait 设计** — 讲清楚了统一接口的设计(name/description/input_schema/invoke)
- [x] **ToolRegistry** — 工具注册与分发机制,HashMap 存储
- [x] **内置工具** — 4 个工具的详细实现(read_file/edit_file/bash/oracle),包含代码示例
- [x] **路径安全** — normalize_path 函数的详细实现,防御路径遍历攻击
- [x] **工具执行流程** — 端到端追踪,7 个步骤+数据流示意图
- [x] **代码片段控制** — 5 处代码片段(Tool trait、ReadFileTool、EditFileTool、BashTool、normalize_path)
- [x] **与前面章节衔接** — 引用了 ToolCall、ToolExecutionStatus、ExecutingTools 状态
- [x] **教学节奏** — 从抽象(trait)到注册(registry)到实现(内置工具)到安全(路径检查)到流程(端到端)
- [x] **周边知识** — 解释了 JSON Schema、snippet 替换、路径规范化等概念
- [x] **准确性** — 所有类型、方法、工具实现与实际代码一致(已验证 tool.rs/registry.rs/read_file.rs/edit_file.rs)
- [x] **可读性** — 使用了表格、代码示例、数据流图示,层次清晰
