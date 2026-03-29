# 第 8 章 Thread 持久化：对话如何保存和同步

> **本章目标**
> 讲解 Loom 的 Thread(对话会话)持久化系统,包括 UUID7 ID 设计、本地存储、服务端同步、离线优先策略和版本冲突检测。读者将理解如何通过 `ThreadStore` trait 实现跨设备的对话恢复。

---

## 8.1 本章是什么

### 8.1.1 章节定位

在 ch07 中,我们追踪了一次完整对话的端到端数据流,但**数据流最终消失在内存中** — 一旦 CLI 退出,`ConversationContext` 就丢失了。

**本章的任务是解答:**

- **对话如何持久化?** — 将 `ConversationContext` 和 `AgentState` 序列化为 JSON 并保存到本地
- **对话如何恢复?** — 通过 `loom resume <thread_id>` 恢复之前的对话状态
- **多设备如何同步?** — 通过服务端 API 将本地 Thread 同步到云端,实现跨设备访问
- **版本冲突如何处理?** — 使用乐观锁(版本号)检测并发修改

**与其他章节的关系:**

- **ch02** — Thread 序列化依赖 `Message`、`ToolCall` 等核心类型
- **ch03** — `AgentState` 的快照保存在 Thread 的 `agent_state` 字段
- **ch07** — Thread 在每次推理回合结束(回到 `WaitingForUserInput`)时保存

### 8.1.2 核心概念

**Thread** — Loom 中的"对话会话",类比聊天应用的"对话窗口",但包含:

- 完整的消息历史 (`conversation.messages`)
- Agent 状态快照 (`agent_state`)
- 工作区元数据 (`workspace_root`、`cwd`)
- Git 上下文 (关联的 commit SHA、仓库信息)
- 可见性控制 (`visibility: Organization|Private|Public`)

**UUID7** — 时间排序的 UUID,ID 格式为 `T-019b2b97-fddf-7602-a3e4-1c4a295110c0`,前缀 `T-` 标识 Thread。

**ThreadStore** — 抽象的持久化接口,定义了 `upsert`、`get`、`list`、`delete` 等操作,有两种实现:

1. **本地实现** — 保存到 `~/.local/share/loom/threads/` (XDG 目录)
2. **服务端实现** — SQLite 数据库 + FTS5 全文搜索

**离线优先(Offline-First)** — 总是**先写本地**,同步到服务端是"尽力而为"(best-effort)且非阻塞。

---

## 8.2 Thread 数据模型

### 8.2.1 Thread 结构体定义

**完整的 Thread JSON 示例:**

```json
{
  "id": "T-019b2b97-fddf-7602-a3e4-1c4a295110c0",
  "version": 5,
  "created_at": "2025-01-01T12:00:00Z",
  "updated_at": "2025-01-01T12:05:00Z",
  "last_activity_at": "2025-01-01T12:05:00Z",

  "workspace_root": "/home/alice/projects/my_app",
  "cwd": "/home/alice/projects/my_app",
  "loom_version": "0.4.0",

  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",

  "conversation": {
    "messages": [
      {
        "role": "user",
        "content": "How do I add logging?",
        "created_at": "2025-01-01T12:00:01Z"
      },
      {
        "role": "assistant",
        "content": "You can use the tracing crate...",
        "tool_calls": [],
        "created_at": "2025-01-01T12:00:05Z"
      }
    ]
  },

  "agent_state": {
    "kind": "waiting_for_user_input",
    "retries": 0,
    "last_error": null,
    "pending_tool_calls": []
  },

  "visibility": "organization",
  "is_private": false,
  "is_shared_with_support": false,

  "metadata": {
    "title": "Add logging to my app",
    "tags": ["logging", "tracing"],
    "is_pinned": false,
    "extra": {}
  }
}
```

**Rust 类型定义:** (见 `specs/thread-system.md:156`)

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Thread {
    pub id: ThreadId,
    pub version: u64, // 乐观锁版本号

    pub created_at: String,       // RFC3339 格式
    pub updated_at: String,       // RFC3339 格式
    pub last_activity_at: String, // 最后一次交互时间

    pub workspace_root: Option<String>, // 工作区根目录
    pub cwd: Option<String>,            // 当前工作目录
    pub loom_version: Option<String>,   // CLI 版本号

    pub provider: Option<String>, // "anthropic" | "openai" | "zai"
    pub model: Option<String>,    // 模型名称

    pub visibility: ThreadVisibility,
    pub is_private: bool,              // true 则永不同步
    pub is_shared_with_support: bool,  // 是否已分享给支持团队

    pub conversation: ConversationSnapshot,
    pub agent_state: AgentStateSnapshot,
    pub metadata: ThreadMetadata,
}
```

### 8.2.2 ThreadId 设计:UUID7 + 前缀

**为什么用 UUID7?**

| 需求 | 传统 UUIDv4 | UUID7 |
|------|------------|-------|
| 全局唯一 | ✅ | ✅ |
| 时间排序 | ❌ (随机生成,无法排序) | ✅ (嵌入时间戳,可按创建时间排序) |
| 无需协调 | ✅ | ✅ |
| 索引性能 | ❌ (随机 ID 导致 B-Tree 碎片) | ✅ (时间递增,减少索引碎片) |

**UUID7 结构:** (RFC 9562)

```
019b2b97-fddf-7602-a3e4-1c4a295110c0
└─┬──┘ └─┬┘ └┬┘ └───────┬──────────┘
  │      │   │          │
  ├─ 48-bit 时间戳(毫秒)
  │      │   │          │
  │      ├─ 12-bit 随机数
  │      │   │          │
  │      │   ├─ 6-bit 版本和变体
  │      │   │          │
  │      │   └─ 62-bit 随机数
```

**ThreadId 实现:** (见 `specs/thread-system.md:118`)

```rust
use uuid7::uuid7;

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct ThreadId(pub String);

impl ThreadId {
    /// 创建新的 Thread ID
    pub fn new() -> Self {
        let uuid = uuid7(); // 生成 UUID7
        Self(format!("T-{}", uuid)) // 添加 "T-" 前缀
    }

    /// 解析现有的 Thread ID 字符串
    pub fn parse(s: &str) -> Result<Self, ThreadIdError> {
        if !s.starts_with("T-") {
            return Err(ThreadIdError::InvalidPrefix);
        }
        // 验证 UUID7 部分
        let uuid_part = &s[2..];
        uuid7::Uuid::parse_str(uuid_part)
            .map_err(|_| ThreadIdError::InvalidUuid)?;
        Ok(Self(s.to_string()))
    }
}
```

**前缀 `T-` 的作用:**

1. **区分类型** — 一眼识别是 Thread ID(而非 User ID、Message ID 等)
2. **URL 友好** — 可以安全用在 URL 路径中(例如 `GET /threads/T-019b2b97-...`)
3. **日志可读** — 在日志中搜索 `T-` 立即找到所有 Thread 相关记录

### 8.2.3 ConversationSnapshot vs ConversationContext

**问题:** ch02 中的 `ConversationContext` 是运行时的内存结构,如何转换为持久化的 `ConversationSnapshot`?

**对比:**

| 字段 | ConversationContext (内存) | ConversationSnapshot (持久化) |
|------|---------------------------|------------------------------|
| **ID** | `Uuid` (运行时生成) | `String` (序列化为字符串) |
| **messages** | `Vec<Message>` | `Vec<MessageSnapshot>` (移除 transient 字段) |
| **额外字段** | 可能包含 transient 状态 | 仅保留可序列化的核心字段 |

**MessageSnapshot 定义:** (见 `specs/thread-system.md:188`)

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct MessageSnapshot {
    pub role: MessageRole, // User | Assistant | Tool
    pub content: String,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub tool_call_id: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub tool_name: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub tool_calls: Option<Vec<ToolCallSnapshot>>,
}
```

**转换逻辑:** (假设实现)

```rust
impl From<&Message> for MessageSnapshot {
    fn from(msg: &Message) -> Self {
        Self {
            role: msg.role.into(),
            content: msg.content.clone(),
            tool_call_id: msg.tool_call_id.clone(),
            tool_name: msg.name.clone(),
            tool_calls: if msg.tool_calls.is_empty() {
                None
            } else {
                Some(msg.tool_calls.iter().map(|tc| tc.into()).collect())
            },
        }
    }
}
```

### 8.2.4 AgentStateSnapshot

**问题:** `AgentState` 是一个复杂的枚举(包含 `ConversationContext`、`LlmResponse` 等),如何持久化?

**答案:** 只保存**状态类型** + **必要的上下文**,不保存完整的瞬态数据。

**AgentStateSnapshot 定义:** (见 `specs/thread-system.md:208`)

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct AgentStateSnapshot {
    pub kind: AgentStateKind, // 状态枚举
    pub retries: u32,         // 重试次数
    pub last_error: Option<String>, // 最后的错误信息
    pub pending_tool_calls: Vec<String>, // 待执行的 tool call ID
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum AgentStateKind {
    WaitingForUserInput,
    CallingLlm,
    ProcessingLlmResponse,
    ExecutingTools,
    PostToolsHook,
    Error,
    ShuttingDown,
}
```

**为什么不保存完整状态?**

1. **大部分状态不需要恢复** — 例如 `CallingLlm` 状态下的 `LlmResponse`,恢复时会重新调用 LLM
2. **避免序列化复杂性** — `LlmResponse` 包含大量字段,序列化/反序列化成本高
3. **恢复时总是回到 `WaitingForUserInput`** — 无论保存时是什么状态,恢复后都是"等待用户输入"

**转换逻辑:**

```rust
impl From<&AgentState> for AgentStateSnapshot {
    fn from(state: &AgentState) -> Self {
        let (kind, retries, last_error, pending_tool_calls) = match state {
            AgentState::WaitingForUserInput { .. } => {
                (AgentStateKind::WaitingForUserInput, 0, None, Vec::new())
            }
            AgentState::CallingLlm { retries, .. } => {
                (AgentStateKind::CallingLlm, *retries, None, Vec::new())
            }
            AgentState::ExecutingTools { executions, .. } => {
                let pending: Vec<String> = executions
                    .iter()
                    .filter(|e| !e.is_completed())
                    .map(|e| e.call_id().to_string())
                    .collect();
                (AgentStateKind::ExecutingTools, 0, None, pending)
            }
            AgentState::Error { retries, error, .. } => {
                (AgentStateKind::Error, *retries, Some(error.to_string()), Vec::new())
            }
            // ... 其他状态
        };

        Self {
            kind,
            retries,
            last_error,
            pending_tool_calls,
        }
    }
}
```

---

## 8.3 ThreadStore trait:统一的持久化接口

### 8.3.1 trait 定义

**为什么需要 trait?**

- **多种实现** — 本地文件存储、SQLite 数据库、远程服务端 API
- **可测试性** — 单元测试可以用 in-memory 实现
- **依赖注入** — Agent 不关心 Thread 存储在哪里,只依赖 `ThreadStore` trait

**核心接口:** (见 `crates/loom-server-db/src/thread.rs:25`)

```rust
use async_trait::async_trait;

#[async_trait]
pub trait ThreadStore: Send + Sync {
    /// 插入或更新 Thread(乐观锁版本检查)
    async fn upsert(
        &self,
        thread: &Thread,
        expected_version: Option<u64>,
    ) -> Result<Thread, DbError>;

    /// 根据 ID 获取 Thread
    async fn get(&self, id: &ThreadId) -> Result<Option<Thread>, DbError>;

    /// 列出 Thread(按 last_activity_at 降序)
    async fn list(
        &self,
        workspace: Option<&str>,
        limit: u32,
        offset: u32,
    ) -> Result<Vec<ThreadSummary>, DbError>;

    /// 删除 Thread
    async fn delete(&self, id: &ThreadId) -> Result<bool, DbError>;

    /// 搜索 Thread(全文搜索)
    async fn search(
        &self,
        query: &str,
        workspace: Option<&str>,
        limit: u32,
        offset: u32,
    ) -> Result<Vec<ThreadSearchHit>, DbError>;

    /// 健康检查
    async fn health_check(&self) -> Result<(), DbError>;

    // ... 其他方法(owner 相关、GitHub 集成等)
}
```

**ThreadSummary** — 列表视图的轻量级摘要:

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct ThreadSummary {
    pub id: ThreadId,
    pub title: Option<String>,
    pub workspace_root: Option<String>,
    pub last_activity_at: String, // RFC3339
    pub provider: Option<String>,
    pub model: Option<String>,
    pub tags: Vec<String>,
    pub version: u64,
    pub message_count: u32, // 从 conversation.messages.len() 计算
}
```

### 8.3.2 乐观锁:expected_version

**问题:** 如果两个 CLI 实例同时修改同一个 Thread,如何避免覆盖冲突?

**答案:** 使用**乐观锁**(Optimistic Locking)通过版本号检测冲突。

**工作流程:**

1. **读取 Thread** — 获取当前版本号(例如 `version=5`)
2. **本地修改** — 用户在 CLI 中继续对话,本地 `version` 递增到 `6`
3. **更新时检查** — 调用 `upsert(thread, expected_version=Some(5))`
4. **版本检查:**
   - 如果数据库中的版本仍是 `5` → 更新成功,版本变为 `6`
   - 如果数据库中的版本已是 `6`(另一个客户端先更新了) → 返回 `DbError::VersionMismatch`

**SQL 实现示例:**

```sql
UPDATE threads
SET data = ?, version = version + 1, updated_at = ?
WHERE id = ? AND version = ? -- 乐观锁检查
RETURNING *;
```

**Rust 实现逻辑:**

```rust
async fn upsert(&self, thread: &Thread, expected_version: Option<u64>) -> Result<Thread, DbError> {
    let data_json = serde_json::to_string(thread)?;
    let now = chrono::Utc::now().to_rfc3339();

    let query = if let Some(expected) = expected_version {
        // 乐观锁更新
        sqlx::query(
            "UPDATE threads SET data = ?, version = version + 1, updated_at = ?
             WHERE id = ? AND version = ?"
        )
        .bind(&data_json)
        .bind(&now)
        .bind(&thread.id.0)
        .bind(expected as i64)
    } else {
        // 初次插入(不检查版本)
        sqlx::query(
            "INSERT INTO threads (id, data, version, created_at, updated_at)
             VALUES (?, ?, 1, ?, ?)
             ON CONFLICT(id) DO UPDATE SET data = excluded.data, version = version + 1"
        )
        .bind(&thread.id.0)
        .bind(&data_json)
        .bind(&now)
        .bind(&now)
    };

    let result = query.execute(&self.pool).await?;
    if result.rows_affected() == 0 {
        return Err(DbError::VersionMismatch);
    }

    // 返回更新后的 Thread
    self.get(&thread.id).await?.ok_or(DbError::NotFound)
}
```

### 8.3.3 全文搜索:ThreadSearchHit

**问题:** 用户有 100+ 个 Thread,如何快速找到"上次修复认证 bug 的对话"?

**答案:** 使用 **SQLite FTS5**(全文搜索)索引 Thread 的内容。

**索引字段:**

- `conversation.messages[].content` — 所有消息的文本内容
- `metadata.title` — Thread 标题
- `metadata.tags` — 标签

**SQL 表结构:** (假设)

```sql
CREATE VIRTUAL TABLE threads_fts USING fts5(
    thread_id UNINDEXED,
    content,       -- 所有消息内容的拼接
    title,
    tags,
    tokenize='unicode61'
);
```

**搜索查询:**

```sql
SELECT ts.id, ts.title, ts.last_activity_at, highlight(fts.content, 0, '<mark>', '</mark>') AS snippet
FROM threads_fts AS fts
JOIN threads AS ts ON fts.thread_id = ts.id
WHERE fts MATCH ?  -- FTS5 查询语法
ORDER BY rank      -- FTS5 相关性评分
LIMIT ? OFFSET ?;
```

**Rust 实现:**

```rust
async fn search(
    &self,
    query: &str,
    workspace: Option<&str>,
    limit: u32,
    offset: u32,
) -> Result<Vec<ThreadSearchHit>, DbError> {
    let rows = sqlx::query(
        "SELECT ts.id, ts.data, rank(fts) AS score
         FROM threads_fts AS fts
         JOIN threads AS ts ON fts.thread_id = ts.id
         WHERE fts MATCH ?
         ORDER BY score DESC
         LIMIT ? OFFSET ?"
    )
    .bind(query)
    .bind(limit as i64)
    .bind(offset as i64)
    .fetch_all(&self.pool)
    .await?;

    rows.into_iter()
        .map(|row| {
            let thread: Thread = serde_json::from_str(row.get("data"))?;
            let score: f64 = row.get("score");
            Ok(ThreadSearchHit {
                summary: ThreadSummary::from(&thread),
                score,
            })
        })
        .collect()
}
```

---

## 8.4 本地存储:XDG 目录规范

### 8.4.1 XDG Base Directory Specification

**问题:** Thread 文件应该保存在哪里?

**答案:** 遵循 **XDG Base Directory Specification**,确保跨平台一致性和用户可控性。

**XDG 环境变量:**

| 变量 | 默认值 (Linux) | 默认值 (macOS) | 用途 |
|------|---------------|---------------|------|
| `XDG_DATA_HOME` | `~/.local/share` | `~/Library/Application Support` | 应用数据存储 |
| `XDG_CONFIG_HOME` | `~/.config` | `~/Library/Application Support` | 配置文件 |
| `XDG_CACHE_HOME` | `~/.cache` | `~/Library/Caches` | 缓存文件 |

**Loom Thread 存储路径:**

```
$XDG_DATA_HOME/loom/threads/
  ├── T-019b2b97-fddf-7602-a3e4-1c4a295110c0.json
  ├── T-019b2b98-0001-7000-8000-000000000001.json
  └── ... (每个 Thread 一个 JSON 文件)
```

**Rust 实现:** (使用 `directories` crate)

```rust
use directories::ProjectDirs;
use std::path::PathBuf;

fn get_threads_dir() -> Result<PathBuf, std::io::Error> {
    let proj_dirs = ProjectDirs::from("com", "ghuntley", "loom")
        .ok_or_else(|| std::io::Error::new(
            std::io::ErrorKind::NotFound,
            "Cannot determine project directories"
        ))?;

    let threads_dir = proj_dirs.data_dir().join("threads");
    std::fs::create_dir_all(&threads_dir)?; // 确保目录存在
    Ok(threads_dir)
}
```

**文件命名规则:**

- 文件名 = `{thread_id}.json`
- 例如: `T-019b2b97-fddf-7602-a3e4-1c4a295110c0.json`

### 8.4.2 本地 ThreadStore 实现

**LocalThreadStore 结构:**

```rust
use std::path::PathBuf;
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct LocalThreadStore {
    threads_dir: PathBuf,
    cache: Arc<RwLock<HashMap<ThreadId, Thread>>>, // 内存缓存
}

impl LocalThreadStore {
    pub fn new() -> Result<Self, std::io::Error> {
        let threads_dir = get_threads_dir()?;
        Ok(Self {
            threads_dir,
            cache: Arc::new(RwLock::new(HashMap::new())),
        })
    }

    fn thread_path(&self, id: &ThreadId) -> PathBuf {
        self.threads_dir.join(format!("{}.json", id.0))
    }
}
```

**upsert 实现:**

```rust
#[async_trait]
impl ThreadStore for LocalThreadStore {
    async fn upsert(&self, thread: &Thread, expected_version: Option<u64>) -> Result<Thread, DbError> {
        // 1. 乐观锁检查
        if let Some(expected) = expected_version {
            let existing = self.get(&thread.id).await?;
            if let Some(existing_thread) = existing {
                if existing_thread.version != expected {
                    return Err(DbError::VersionMismatch);
                }
            }
        }

        // 2. 序列化为 JSON
        let json = serde_json::to_string_pretty(thread)?;

        // 3. 原子写入文件(先写 .tmp 再 rename)
        let path = self.thread_path(&thread.id);
        let tmp_path = path.with_extension("tmp");
        tokio::fs::write(&tmp_path, json).await?;
        tokio::fs::rename(&tmp_path, &path).await?;

        // 4. 更新内存缓存
        self.cache.write().await.insert(thread.id.clone(), thread.clone());

        Ok(thread.clone())
    }

    async fn get(&self, id: &ThreadId) -> Result<Option<Thread>, DbError> {
        // 1. 先查内存缓存
        if let Some(thread) = self.cache.read().await.get(id) {
            return Ok(Some(thread.clone()));
        }

        // 2. 读取文件
        let path = self.thread_path(id);
        if !path.exists() {
            return Ok(None);
        }

        let json = tokio::fs::read_to_string(&path).await?;
        let thread: Thread = serde_json::from_str(&json)?;

        // 3. 更新缓存
        self.cache.write().await.insert(id.clone(), thread.clone());

        Ok(Some(thread))
    }

    async fn list(&self, workspace: Option<&str>, limit: u32, offset: u32) -> Result<Vec<ThreadSummary>, DbError> {
        // 1. 读取所有 .json 文件
        let mut entries = tokio::fs::read_dir(&self.threads_dir).await?;
        let mut threads = Vec::new();

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if path.extension().and_then(|s| s.to_str()) != Some("json") {
                continue;
            }

            let json = tokio::fs::read_to_string(&path).await?;
            let thread: Thread = serde_json::from_str(&json)?;

            // 2. 过滤 workspace
            if let Some(ws) = workspace {
                if thread.workspace_root.as_deref() != Some(ws) {
                    continue;
                }
            }

            threads.push(thread);
        }

        // 3. 按 last_activity_at 降序排序
        threads.sort_by(|a, b| b.last_activity_at.cmp(&a.last_activity_at));

        // 4. 分页
        let summaries: Vec<ThreadSummary> = threads
            .into_iter()
            .skip(offset as usize)
            .take(limit as usize)
            .map(|t| ThreadSummary::from(&t))
            .collect();

        Ok(summaries)
    }

    async fn delete(&self, id: &ThreadId) -> Result<bool, DbError> {
        let path = self.thread_path(id);
        if !path.exists() {
            return Ok(false);
        }

        tokio::fs::remove_file(&path).await?;
        self.cache.write().await.remove(id);
        Ok(true)
    }
}
```

---

## 8.5 同步到服务端:离线优先策略

### 8.5.1 离线优先(Offline-First)设计

**核心原则:**

1. **本地总是第一优先级** — 所有操作先写本地,立即返回成功
2. **同步是异步的** — 同步到服务端在后台进行,不阻塞用户
3. **失败不影响本地** — 如果服务端不可达,本地数据仍然可用
4. **最终一致性** — 一旦网络恢复,本地和服务端会最终同步

**优势:**

- **低延迟** — 用户不需要等待网络往返
- **可用性** — 离线时仍然可以使用 Loom
- **可靠性** — 网络故障不会导致数据丢失

**劣势:**

- **冲突处理** — 多设备并发修改可能导致冲突(通过版本号检测)
- **数据滞后** — 服务端数据可能不是最新的

### 8.5.2 同步触发时机

**何时同步?**

1. **推理回合结束** — 每次对话回到 `WaitingForUserInput` 时(见 ch07 § 7.3 [N15])
2. **优雅关闭** — CLI 退出时(`AgentAction::Shutdown`)
3. **手动同步** — 用户执行 `loom sync` 命令

**同步逻辑:** (在 CLI 中)

```rust
async fn sync_thread_to_server(thread: &Thread, client: &LoomClient) -> Result<(), SyncError> {
    // 1. 发送 HTTP PUT 请求到服务端
    let response = client
        .put(&format!("/v1/threads/{}", thread.id.0))
        .header("Content-Type", "application/json")
        .json(thread)
        .send()
        .await?;

    // 2. 处理响应
    match response.status().as_u16() {
        200 | 201 => Ok(()), // 同步成功
        409 => {
            // 版本冲突
            let server_thread: Thread = response.json().await?;
            warn!(
                "Version conflict: local={}, server={}",
                thread.version, server_thread.version
            );
            Err(SyncError::VersionConflict {
                local: thread.version,
                server: server_thread.version,
            })
        }
        _ => Err(SyncError::HttpError(response.status())),
    }
}
```

**非阻塞同步:** (使用 tokio::spawn)

```rust
// 在推理回合结束后
if matches!(agent.state(), AgentState::WaitingForUserInput { .. }) {
    // 1. 保存本地
    thread.version += 1;
    thread.updated_at = now_rfc3339();
    local_store.upsert(&thread, None).await?;

    // 2. 异步同步到服务端(不阻塞)
    let thread_clone = thread.clone();
    let client_clone = Arc::clone(&server_client);
    tokio::spawn(async move {
        if let Err(e) = sync_thread_to_server(&thread_clone, &client_clone).await {
            warn!("Failed to sync thread to server: {}", e);
        }
    });
}
```

### 8.5.3 服务端 Thread API

**HTTP 端点:**

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/v1/threads` | 列出当前用户的 Threads |
| `GET` | `/v1/threads/{id}` | 获取单个 Thread |
| `PUT` | `/v1/threads/{id}` | 创建或更新 Thread(乐观锁) |
| `DELETE` | `/v1/threads/{id}` | 删除 Thread |
| `GET` | `/v1/threads/search?q={query}` | 搜索 Threads |

**PUT 请求示例:**

```http
PUT /v1/threads/T-019b2b97-fddf-7602-a3e4-1c4a295110c0 HTTP/1.1
Host: loom.ghuntley.com
Authorization: Bearer {token}
Content-Type: application/json

{
  "id": "T-019b2b97-fddf-7602-a3e4-1c4a295110c0",
  "version": 6,
  "conversation": { ... },
  ...
}
```

**服务端处理逻辑:** (假设 `crates/loom-server/src/routes/threads.rs`)

```rust
async fn upsert_thread(
    State(state): State<AppState>,
    Path(thread_id): Path<String>,
    Json(thread): Json<Thread>,
) -> Result<Json<Thread>, ApiError> {
    // 1. 解析 Thread ID
    let id = ThreadId::parse(&thread_id)?;

    // 2. 获取当前用户(从 JWT token 或 session)
    let user_id = extract_user_id_from_auth(&state)?;

    // 3. 乐观锁更新
    let updated_thread = state
        .thread_store
        .upsert(&thread, Some(thread.version))
        .await?;

    // 4. 设置 owner(如果是新 Thread)
    state
        .thread_store
        .set_owner_user_id(&id.0, &user_id)
        .await?;

    Ok(Json(updated_thread))
}
```

### 8.5.4 冲突解决策略

**冲突场景:** 用户在设备 A 和设备 B 同时修改同一个 Thread

**检测:**

```
设备 A: 读取 Thread (version=5) → 本地修改 → upsert(version=6)
设备 B: 读取 Thread (version=5) → 本地修改 → upsert(version=6)

服务端: 设备 A 先到达,version 变为 6
        设备 B 到达时,expected_version=5 不匹配,返回 409 Conflict
```

**解决方案:**

1. **Last-Write-Wins(LWW)** — 简单粗暴,最后写入的获胜(不推荐,会丢失数据)
2. **Manual Merge** — 提示用户手动解决冲突(类似 Git merge conflict)
3. **Auto-Merge Messages** — 自动合并消息列表(如果没有重叠)

**Loom 当前策略:** 检测冲突后**警告用户**,但**不自动合并**:

```rust
match sync_thread_to_server(&thread, &client).await {
    Err(SyncError::VersionConflict { local, server }) => {
        eprintln!(
            "⚠️  Thread sync conflict detected!\n\
             Local version: {}, Server version: {}\n\
             Use 'loom pull {}' to fetch the latest version.",
            local, server, thread.id.0
        );
    }
    Err(e) => {
        warn!("Sync failed: {}", e);
    }
    Ok(()) => {
        debug!("Thread synced successfully");
    }
}
```

---

## 8.6 CLI 命令:列出、恢复、搜索

### 8.6.1 `loom list` — 列出本地 Threads

**用法:**

```bash
$ loom list

Recent threads:
  T-019b2b98-0001-7000  Add logging to my app        2025-01-01 12:05  (5 messages)
  T-019b2b97-fddf-7602  Fix authentication bug       2025-01-01 10:30  (12 messages)
  T-019b2b96-0000-1000  Refactor database schema     2024-12-31 18:00  (8 messages)
```

**实现:**

```rust
pub async fn cmd_list(store: &dyn ThreadStore, workspace: Option<&str>) -> Result<(), CliError> {
    let summaries = store.list(workspace, 20, 0).await?; // 默认显示 20 条

    println!("Recent threads:");
    for summary in summaries {
        println!(
            "  {}  {:30}  {}  ({} messages)",
            &summary.id.0[0..20], // 截断 ID
            summary.title.as_deref().unwrap_or("(untitled)"),
            summary.last_activity_at,
            summary.message_count
        );
    }

    Ok(())
}
```

### 8.6.2 `loom resume` — 恢复 Thread

**用法:**

```bash
$ loom resume                                        # 恢复最近的 Thread
$ loom resume T-019b2b97-fddf-7602-a3e4-1c4a295110c0 # 恢复指定 Thread
```

**实现:**

```rust
pub async fn cmd_resume(
    store: &dyn ThreadStore,
    thread_id: Option<String>,
) -> Result<(), CliError> {
    // 1. 获取 Thread
    let thread = if let Some(id_str) = thread_id {
        let id = ThreadId::parse(&id_str)?;
        store.get(&id).await?.ok_or(CliError::ThreadNotFound(id))?
    } else {
        // 恢复最近的 Thread
        let summaries = store.list(None, 1, 0).await?;
        let summary = summaries.first().ok_or(CliError::NoThreadsFound)?;
        store.get(&summary.id).await?.ok_or(CliError::ThreadNotFound(summary.id.clone()))?
    };

    // 2. 从 Thread 恢复 Agent 状态
    let mut agent = Agent::new(config, llm_client, tools);
    agent.restore_from_thread(&thread)?;

    // 3. 进入 REPL
    println!("Resuming thread: {}", thread.metadata.title.as_deref().unwrap_or("(untitled)"));
    println!("{} messages loaded", thread.conversation.messages.len());
    repl_loop(agent, thread, store).await?;

    Ok(())
}
```

**恢复逻辑:**

```rust
impl Agent {
    pub fn restore_from_thread(&mut self, thread: &Thread) -> Result<(), AgentError> {
        // 1. 恢复消息历史
        let messages: Vec<Message> = thread
            .conversation
            .messages
            .iter()
            .map(|ms| Message::from_snapshot(ms))
            .collect();

        // 2. 构造 ConversationContext
        let conversation = ConversationContext {
            id: Uuid::new_v7(),
            messages,
        };

        // 3. 恢复状态(总是回到 WaitingForUserInput)
        self.state = AgentState::WaitingForUserInput { conversation };

        Ok(())
    }
}
```

### 8.6.3 `loom search` — 搜索 Threads

**用法:**

```bash
$ loom search "authentication fix"

Found 2 threads:
  T-019b2b97-fddf-7602  Fix authentication bug  (score: 12.5)
    ... fixed the JWT token validation in auth middleware ...

  T-019b2b96-0000-1000  Add OAuth login         (score: 8.3)
    ... implemented OAuth 2.0 authentication flow ...
```

**实现:**

```rust
pub async fn cmd_search(
    store: &dyn ThreadStore,
    query: &str,
    workspace: Option<&str>,
) -> Result<(), CliError> {
    let hits = store.search(query, workspace, 20, 0).await?;

    println!("Found {} threads:", hits.len());
    for hit in hits {
        println!(
            "  {}  {}  (score: {:.1})",
            &hit.summary.id.0[0..20],
            hit.summary.title.as_deref().unwrap_or("(untitled)"),
            hit.score
        );
        // TODO: 显示匹配片段(snippet)
    }

    Ok(())
}
```

---

## 8.7 本章小结

### 8.7.1 核心知识点回顾

1. **Thread 数据模型** — `Thread` 结构体包含完整的对话历史、Agent 状态快照、工作区元数据
2. **UUID7 ID** — 时间排序的 UUID,前缀 `T-`,支持按创建时间排序和减少索引碎片
3. **ThreadStore trait** — 统一的持久化接口,支持本地文件存储和服务端 SQLite
4. **乐观锁** — 通过 `version` 字段检测并发冲突,`expected_version` 参数实现 CAS(Compare-And-Swap)
5. **XDG 目录规范** — Thread 保存在 `~/.local/share/loom/threads/` (Linux) 或 `~/Library/Application Support/loom/threads/` (macOS)
6. **离线优先** — 总是先写本地,同步到服务端在后台异步进行
7. **全文搜索** — 使用 SQLite FTS5 索引消息内容、标题和标签

### 8.7.2 与其他章节的呼应

| 章节 | 在本章的体现 |
|------|------------|
| ch02 | `MessageSnapshot` 和 `ToolCallSnapshot` 的序列化 |
| ch03 | `AgentStateSnapshot` 保存状态类型和重试次数 |
| ch07 | 推理回合结束(回到 `WaitingForUserInput`)时触发 Thread 保存 |

### 8.7.3 未覆盖的主题

以下主题在本章中仅简要提及,将在后续章节详细讨论:

- **ch11 认证与授权** — Thread 的 `owner_user_id` 和 `visibility` 如何结合 ABAC 策略控制访问
- **ch14 SCM 与 Git 集成** — Thread 的 `git_commits` 字段如何关联代码变更
- **ch18 部署架构** — 服务端的 Thread 数据库如何备份和迁移

---

## 8.8 质检清单

在完成本章后,请验证以下内容:

- [x] **Thread 数据模型** — 是否讲解了 `Thread`、`ThreadId`、`ConversationSnapshot`、`AgentStateSnapshot` 的结构?
- [x] **UUID7 设计** — 是否解释了 UUID7 的优势(时间排序、减少索引碎片)?
- [x] **ThreadStore trait** — 是否定义了 `upsert`、`get`、`list`、`delete`、`search` 方法?
- [x] **乐观锁** — 是否讲解了 `expected_version` 参数和版本冲突检测?
- [x] **本地存储** — 是否说明了 XDG 目录规范和文件命名规则?
- [x] **服务端同步** — 是否讲解了离线优先策略和非阻塞同步?
- [x] **CLI 命令** — 是否介绍了 `loom list`、`loom resume`、`loom search` 的用法?
- [x] **冲突解决** — 是否讨论了版本冲突的检测和处理策略?

---

**下一章预告:** ch09 将讨论 **Weaver:远程执行环境的 K8s 实现**,展示 Tool 如何在 Kubernetes Pod 中执行,包括 Pod 生命周期管理、Secret 注入、SPIFFE 身份认证和 eBPF 审计。
