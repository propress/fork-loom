# 第 12 章 配置与 Secret 管理：分层配置、自动检测

> **本章目标**
> 讲解 Loom 的配置系统和 Secret 管理机制,包括 XDG 路径规范、多源配置优先级、`Secret<T>` 类型的 Redact 系统、环境变量和文件加载策略。读者将理解如何安全地管理 API 密钥并避免日志泄露。

---

## 12.1 本章是什么

### 12.1.1 章节定位

**配置系统**和**Secret 管理**是 Loom 中两个密切相关但职责不同的子系统:

- **配置系统** — 管理用户可见的配置项(模型名称、日志级别、工作区路径等)
- **Secret 管理** — 管理敏感值(API 密钥、OAuth token、私钥等),防止意外泄露

**本章解答:**

- **配置文件在哪里?** — XDG Base Directory Specification 的路径规范
- **多个配置源如何合并?** — 分层配置的优先级规则(CLI 参数 > 环境变量 > 工作区配置 > 用户配置 > 系统配置)
- **Secret 如何防止泄露?** — `Secret<T>` 类型的 Redact 机制
- **如何加载 Secret?** — 环境变量 + `*_FILE` 文件挂载(Kubernetes Secret 模式)

### 12.1.2 核心概念

**XDG Base Directory Specification** — freedesktop.org 制定的 Linux 文件位置标准,定义了配置、数据、缓存的存放目录。

**分层配置(Layered Configuration)** — 多个配置源按优先级合并,高优先级覆盖低优先级。

**Secret<T>** — Rust 类型包装器,实现了:

1. **Redacted Debug/Display** — `format!("{:?}", secret)` 输出 `Secret("[REDACTED]")`
2. **Redacted Serialize** — JSON/TOML 序列化为 `"[REDACTED]"`
3. **Explicit Access** — 必须调用 `.expose()` 才能访问内部值
4. **Zeroize on Drop** — 内存在 drop 时清零

**`*_FILE` 文件挂载约定** — 环境变量 `ANTHROPIC_API_KEY_FILE` 指向文件路径,从文件读取 Secret(Kubernetes/Docker Secret 模式)。

---

## 12.2 XDG 路径规范

### 12.2.1 XDG Base Directory Specification

**问题:** 应用的配置文件、数据文件、缓存文件应该放在哪里?

**答案:** 遵循 [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html),确保:

- **用户可控性** — 用户通过环境变量自定义路径
- **跨应用一致性** — 所有遵循 XDG 的应用使用相同的规则
- **清晰的职责分离** — 配置、数据、缓存分别存放

**XDG 环境变量:**

| 环境变量 | 默认值 (Linux) | 默认值 (macOS) | 用途 |
|----------|---------------|---------------|------|
| `XDG_CONFIG_HOME` | `~/.config` | `~/Library/Application Support` | 应用配置文件 |
| `XDG_DATA_HOME` | `~/.local/share` | `~/Library/Application Support` | 应用数据文件 |
| `XDG_CACHE_HOME` | `~/.cache` | `~/Library/Caches` | 应用缓存文件 |
| `XDG_STATE_HOME` | `~/.local/state` | `~/Library/Application Support` | 应用运行时状态 |

**Loom 目录映射:**

```
$XDG_CONFIG_HOME/loom/
  ├── config.toml          # 用户配置
  └── .gitignore           # 忽略敏感文件

$XDG_DATA_HOME/loom/
  ├── threads/             # Thread 持久化 (见 ch08)
  │   ├── T-019b2b97-fddf-7602.json
  │   └── ...
  └── history/             # REPL 历史记录

$XDG_CACHE_HOME/loom/
  ├── providers/           # LLM Provider 缓存
  └── compiled-models/     # 编译后的模型缓存(未实现)

$XDG_STATE_HOME/loom/
  ├── logs/                # 日志文件
  │   ├── loom-2025-01-01.log
  │   └── loom-2025-01-02.log
  └── runtime.lock         # 运行时锁文件
```

**工作区配置:** (项目特定)

```
/path/to/workspace/
  ├── .loom/
  │   ├── config.toml      # 工作区配置(优先级高于用户配置)
  │   └── secrets/         # 工作区 Secret (不应提交到 Git)
  ├── .gitignore           # 应包含 .loom/secrets/
  └── ... (项目文件)
```

### 12.2.2 路径解析算法

**Rust 实现:** (使用 `directories` crate)

```rust
use directories::ProjectDirs;
use std::path::PathBuf;

pub fn get_config_dir() -> Result<PathBuf, std::io::Error> {
    let proj_dirs = ProjectDirs::from("com", "ghuntley", "loom")
        .ok_or_else(|| std::io::Error::new(
            std::io::ErrorKind::NotFound,
            "Cannot determine project directories"
        ))?;

    Ok(proj_dirs.config_dir().to_path_buf())
    // 返回: ~/.config/loom (Linux) 或 ~/Library/Application Support/loom (macOS)
}

pub fn get_data_dir() -> Result<PathBuf, std::io::Error> {
    let proj_dirs = ProjectDirs::from("com", "ghuntley", "loom")
        .ok_or_else(|| std::io::Error::new(
            std::io::ErrorKind::NotFound,
            "Cannot determine project directories"
        ))?;

    Ok(proj_dirs.data_dir().to_path_buf())
}
```

**手动解析 XDG 变量:** (当 `directories` crate 不可用时)

```rust
fn resolve_xdg_path(xdg_var: &str, default_suffix: &str) -> PathBuf {
    env::var(xdg_var)
        .map(PathBuf::from)
        .unwrap_or_else(|_| {
            let home = env::var("HOME").expect("HOME must be set");
            PathBuf::from(home).join(default_suffix)
        })
        .join("loom")
}

// 示例调用
let config_home = resolve_xdg_path("XDG_CONFIG_HOME", ".config");
// 结果: /home/alice/.config/loom (如果 XDG_CONFIG_HOME 未设置)
```

### 12.2.3 自动创建默认配置

**问题:** 用户第一次运行 `loom`,配置文件不存在怎么办?

**答案:** **自动创建默认配置文件**,包含注释和示例值。

**逻辑流程:**

1. 检查 `$XDG_CONFIG_HOME/loom/config.toml` 是否存在
2. 如果不存在:
   - 创建父目录 `~/.config/loom/`
   - 写入默认配置模板
   - 记录日志 `INFO: Created default config at ~/.config/loom/config.toml`
3. 如果存在:
   - 跳过创建(永不覆盖现有配置)

**默认配置模板:** (见 `specs/configuration-system.md:162`)

```toml
#
# Loom Configuration File
# Location: ~/.config/loom/config.toml
#

# =============================================================================
# Global Settings
# =============================================================================

[global]
# The default LLM provider to use when none is specified
default_provider = "anthropic"

# Ordered list of model preferences for automatic fallback
model_preferences = [
    "claude-sonnet-4-20250514",
    "claude-3-5-sonnet-20241022",
    "gpt-4o",
]

# =============================================================================
# Provider Configurations
# =============================================================================

[providers.anthropic]
type = "anthropic"
# API key (prefer environment variable ANTHROPIC_API_KEY)
# api_key = "sk-ant-..."
default_model = "claude-sonnet-4-20250514"
max_tokens = 8192
temperature = 0.7

[providers.openai]
type = "openai"
# API key (prefer environment variable OPENAI_API_KEY)
# api_key = "sk-..."
default_model = "gpt-4o"
max_tokens = 8192

# =============================================================================
# Logging
# =============================================================================

[logging]
level = "info" # trace | debug | info | warn | error
format = "compact" # compact | pretty | json
```

**Rust 实现:**

```rust
use std::fs;
use std::path::Path;

pub fn ensure_default_config(config_path: &Path) -> Result<(), std::io::Error> {
    if config_path.exists() {
        return Ok(()); // 不覆盖现有配置
    }

    // 创建父目录
    if let Some(parent) = config_path.parent() {
        fs::create_dir_all(parent)?;
    }

    // 写入默认配置
    let default_config = include_str!("../templates/default_config.toml");
    fs::write(config_path, default_config)?;

    tracing::info!(path = ?config_path, "Created default configuration file");
    Ok(())
}
```

---

## 12.3 分层配置:优先级和合并

### 12.3.1 配置源和优先级

**6 层配置源:** (见 `specs/configuration-system.md:115`)

| 优先级 | 配置源 | 示例 | 数值 |
|--------|--------|------|------|
| 1 (最高) | **CLI 参数** | `loom --model claude-sonnet-4` | 60 |
| 2 | **环境变量** | `LOOM_DEFAULT_MODEL=gpt-4o` | 50 |
| 3 | **工作区配置** | `.loom/config.toml` | 40 |
| 4 | **用户配置** | `~/.config/loom/config.toml` | 30 |
| 5 | **系统配置** | `/etc/loom/config.toml` | 20 |
| 6 (最低) | **内置默认值** | 硬编码在代码中 | 10 |

**优先级规则:**

1. **标量值**(字符串、数字、布尔) — 高优先级**完全替换**低优先级
2. **表/映射**(TOML table) — **深度合并**,高优先级的键覆盖低优先级
3. **数组**(列表) — 高优先级**完全替换**(不合并)

### 12.3.2 合并示例

**场景:** 多个配置文件定义了部分相同的配置项

**系统配置** (`/etc/loom/config.toml`,优先级 20):

```toml
[global]
default_provider = "ollama" # 系统管理员希望默认用本地模型

[logging]
level = "warn"
format = "json"

[providers.ollama]
type = "ollama"
base_url = "http://localhost:11434"
```

**用户配置** (`~/.config/loom/config.toml`,优先级 30):

```toml
[global]
default_provider = "anthropic" # 用户覆盖为 Anthropic

[providers.anthropic]
type = "anthropic"
default_model = "claude-sonnet-4-20250514"
```

**工作区配置** (`.loom/config.toml`,优先级 40):

```toml
[logging]
level = "debug" # 项目开发时需要详细日志
```

**最终合并结果:**

```toml
[global]
default_provider = "anthropic" # 来自用户配置(优先级 30)

[logging]
level = "debug"   # 来自工作区配置(优先级 40)
format = "json"   # 来自系统配置(优先级 20)

[providers.ollama]
type = "ollama"
base_url = "http://localhost:11434" # 来自系统配置

[providers.anthropic]
type = "anthropic"
default_model = "claude-sonnet-4-20250514" # 来自用户配置
```

**关键观察:**

- `global.default_provider` 被用户配置覆盖(30 > 20)
- `logging.level` 被工作区配置覆盖(40 > 20)
- `logging.format` 保留系统配置(无更高优先级的覆盖)
- `providers.ollama` 和 `providers.anthropic` **共存**(深度合并)

### 12.3.3 环境变量映射

**约定:** 环境变量名遵循 `LOOM_<SECTION>_<KEY>` 格式,下划线分隔嵌套层级。

**映射规则:**

```
LOOM_GLOBAL_DEFAULT_PROVIDER  → global.default_provider
LOOM_LOGGING_LEVEL            → logging.level
LOOM_PROVIDERS_ANTHROPIC_API_KEY → providers.anthropic.api_key
```

**Rust 实现:** (使用 `envy` crate 或手动解析)

```rust
use std::env;

pub fn load_env_config(config: &mut Config) {
    // 简化示例:手动映射
    if let Ok(val) = env::var("LOOM_GLOBAL_DEFAULT_PROVIDER") {
        config.global.default_provider = Some(val);
    }

    if let Ok(val) = env::var("LOOM_LOGGING_LEVEL") {
        config.logging.level = val.parse().unwrap_or_default();
    }

    // 实际实现会使用 envy::prefixed("LOOM_") 自动反序列化
}
```

### 12.3.4 配置结构体定义

**Rust 类型:**

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Config {
    pub global: GlobalConfig,
    pub providers: ProvidersConfig,
    pub logging: LoggingConfig,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct GlobalConfig {
    pub default_provider: Option<String>,
    pub model_preferences: Vec<String>,
    pub workspace_root: Option<String>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct ProvidersConfig {
    #[serde(flatten)]
    pub providers: HashMap<String, ProviderConfig>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct ProviderConfig {
    #[serde(rename = "type")]
    pub provider_type: String, // "anthropic" | "openai" | "ollama"
    pub api_key: Option<SecretString>, // 使用 Secret 包装
    pub base_url: Option<String>,
    pub default_model: Option<String>,
    pub max_tokens: Option<u32>,
    pub temperature: Option<f32>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct LoggingConfig {
    pub level: LogLevel,
    pub format: LogFormat,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum LogLevel {
    Trace,
    Debug,
    Info,
    Warn,
    Error,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum LogFormat {
    Compact,
    Pretty,
    Json,
}
```

---

## 12.4 Secret<T> 类型:防止敏感值泄露

### 12.4.1 Secret<T> 设计

**问题:** API 密钥如何防止意外泄露到日志、错误信息或配置转储中?

**答案:** 使用 **`Secret<T>` 类型包装器**,在编译时强制 Redact。

**核心实现:** (见 `specs/secret-system.md:78`)

```rust
use zeroize::Zeroize;
use serde::{Deserialize, Serialize};

#[derive(Zeroize)]
#[zeroize(drop)] // 内存在 drop 时自动清零
pub struct Secret<T>
where
    T: Zeroize,
{
    inner: T,
}

pub type SecretString = Secret<String>;

pub const REDACTED: &str = "[REDACTED]";
```

**API 方法:**

```rust
impl<T: Zeroize> Secret<T> {
    /// 创建新的 Secret 包装器
    pub fn new(inner: T) -> Self {
        Self { inner }
    }

    /// 显式访问内部值(使访问在代码审查中可见)
    pub fn expose(&self) -> &T {
        &self.inner
    }

    /// 可变访问内部值
    pub fn expose_mut(&mut self) -> &mut T {
        &mut self.inner
    }

    /// 消费 Secret 并返回内部值
    pub fn into_inner(self) -> T
    where
        T: Clone,
    {
        let cloned = self.inner.clone();
        // self 会被 drop,内存被 zeroize
        cloned
    }
}
```

### 12.4.2 Redacted Debug 和 Display

**Debug 实现:**

```rust
impl<T: Zeroize> std::fmt::Debug for Secret<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Secret(\"{}\")", REDACTED)
    }
}
```

**Display 实现:**

```rust
impl<T: Zeroize> std::fmt::Display for Secret<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", REDACTED)
    }
}
```

**效果:**

```rust
use loom_secret::SecretString;

let api_key = SecretString::new("sk-ant-api03-xxx".to_string());

println!("{:?}", api_key); // 输出: Secret("[REDACTED]")
println!("{}", api_key);   // 输出: [REDACTED]
```

### 12.4.3 Redacted Serialize

**Serialize 实现:**

```rust
impl<T: Zeroize> Serialize for Secret<T> {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        serializer.serialize_str(REDACTED)
    }
}
```

**效果:**

```rust
let config = ProviderConfig {
    provider_type: "anthropic".to_string(),
    api_key: Some(SecretString::new("sk-ant-api03-xxx".to_string())),
    default_model: Some("claude-sonnet-4".to_string()),
};

let json = serde_json::to_string_pretty(&config)?;
println!("{}", json);
```

**输出:**

```json
{
  "type": "anthropic",
  "api_key": "[REDACTED]",
  "default_model": "claude-sonnet-4"
}
```

**关键点:** 即使配置被序列化并写入日志或错误消息,API 密钥仍然是 `"[REDACTED]"`。

### 12.4.4 Deserialize 正常加载

**Deserialize 实现:** (使用 serde 的默认实现)

```rust
impl<'de, T> Deserialize<'de> for Secret<T>
where
    T: Deserialize<'de> + Zeroize,
{
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let inner = T::deserialize(deserializer)?;
        Ok(Secret::new(inner))
    }
}
```

**效果:**

```toml
[providers.anthropic]
api_key = "sk-ant-api03-real-key-here"
```

**Rust 加载:**

```rust
let config: ProviderConfig = toml::from_str(toml_str)?;
// config.api_key = Some(SecretString::new("sk-ant-api03-real-key-here"))
```

**关键点:** 从配置文件**正常加载**,但一旦加载到 `Secret<T>`,所有输出都被 Redact。

### 12.4.5 与 tracing 集成

**问题:** 结构化日志如何避免泄露 Secret?

**答案:** `tracing` 通过 `Debug` 和 `Display` trait 格式化字段,`Secret<T>` 的实现自动 Redact。

**示例:**

```rust
use tracing::info;
use loom_secret::SecretString;

let api_key = SecretString::new("sk-ant-api03-xxx".to_string());

// Display 格式化 (%): 输出 "[REDACTED]"
info!(api_key = %api_key, "Configured Anthropic");

// Debug 格式化 (?): 输出 "Secret(\"[REDACTED]\")"
info!(?api_key, "API key loaded");

// 实际密钥永不出现在日志中
```

**日志输出:**

```
2025-01-01T12:00:00.123Z INFO loom_config: Configured Anthropic api_key=[REDACTED]
2025-01-01T12:00:00.124Z INFO loom_config: API key loaded api_key=Secret("[REDACTED]")
```

---

## 12.5 Secret 加载:`*_FILE` 文件挂载约定

### 12.5.1 Kubernetes/Docker Secret 模式

**问题:** 在 Kubernetes/Docker 环境中,Secret 通常通过**文件挂载**而非环境变量提供(避免环境变量在 `ps` 命令中可见)。

**示例:** Kubernetes Secret 挂载:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: loom-secrets
stringData:
  anthropic-api-key: "sk-ant-api03-xxx"
---
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: loom-server
    env:
    - name: ANTHROPIC_API_KEY_FILE
      value: /var/run/secrets/anthropic-api-key
    volumeMounts:
    - name: secrets
      mountPath: /var/run/secrets
  volumes:
  - name: secrets
    secret:
      secretName: loom-secrets
```

**结果:**

- 文件 `/var/run/secrets/anthropic-api-key` 包含内容 `sk-ant-api03-xxx`
- 环境变量 `ANTHROPIC_API_KEY_FILE=/var/run/secrets/anthropic-api-key`

### 12.5.2 `*_FILE` 加载函数

**Rust 实现:** (见 `specs/secret-system.md`,假设在 `loom-common-config/src/secret_env.rs`)

```rust
use loom_secret::SecretString;
use std::env;
use std::fs;

/// 从环境变量或文件加载 Secret
/// 优先级: VAR_FILE > VAR
pub fn load_secret_env(var_name: &str) -> Result<Option<SecretString>, LoadSecretError> {
    let file_var_name = format!("{}_FILE", var_name);

    // 1. 尝试加载 VAR_FILE
    if let Ok(file_path) = env::var(&file_var_name) {
        let content = fs::read_to_string(&file_path)
            .map_err(|e| LoadSecretError::FileRead {
                path: file_path.clone(),
                error: e,
            })?;
        let trimmed = content.trim().to_string(); // 去除换行符
        return Ok(Some(SecretString::new(trimmed)));
    }

    // 2. 回退到 VAR
    if let Ok(value) = env::var(var_name) {
        return Ok(Some(SecretString::new(value)));
    }

    // 3. 未设置
    Ok(None)
}

/// 必须存在的 Secret(否则返回错误)
pub fn require_secret_env(var_name: &str) -> Result<SecretString, LoadSecretError> {
    load_secret_env(var_name)?
        .ok_or_else(|| LoadSecretError::Required {
            var_name: var_name.to_string(),
        })
}

#[derive(Debug, thiserror::Error)]
pub enum LoadSecretError {
    #[error("Failed to read secret file {path}: {error}")]
    FileRead {
        path: String,
        error: std::io::Error,
    },

    #[error("Required secret {var_name} is not set (neither {var_name} nor {var_name}_FILE)")]
    Required { var_name: String },
}
```

**使用示例:**

```rust
use loom_common_config::load_secret_env;

// 尝试加载 ANTHROPIC_API_KEY_FILE 或 ANTHROPIC_API_KEY
let api_key = load_secret_env("ANTHROPIC_API_KEY")?;

if let Some(key) = api_key {
    println!("API key loaded: {}", key); // 输出: API key loaded: [REDACTED]
    // 实际使用: client.with_api_key(key.expose())
}
```

### 12.5.3 集成到配置加载

**配置加载流程:**

1. **读取配置文件** — 从 TOML 文件加载 `ProviderConfig`
2. **检查环境变量** — 如果 `api_key` 为 `None`,尝试从环境变量加载
3. **验证必需字段** — 确保所有 Provider 都有有效的 `api_key`

**Rust 实现:**

```rust
pub fn load_provider_config(name: &str, toml_config: ProviderConfig) -> Result<ProviderConfig, ConfigError> {
    let mut config = toml_config;

    // 如果 TOML 中没有 api_key,从环境变量加载
    if config.api_key.is_none() {
        let env_var_name = format!("{}_API_KEY", name.to_uppercase());
        config.api_key = load_secret_env(&env_var_name)?;
    }

    // 验证必需字段
    if config.api_key.is_none() && config.provider_type != "ollama" {
        return Err(ConfigError::MissingApiKey {
            provider: name.to_string(),
        });
    }

    Ok(config)
}
```

**示例:**

```toml
[providers.anthropic]
type = "anthropic"
# api_key 未在 TOML 中设置
default_model = "claude-sonnet-4"
```

**环境变量:**

```bash
export ANTHROPIC_API_KEY="sk-ant-api03-xxx"
# 或
export ANTHROPIC_API_KEY_FILE="/var/run/secrets/anthropic-api-key"
```

**结果:** `config.api_key` 自动从环境变量加载。

---

## 12.6 配置验证和错误处理

### 12.6.1 配置验证规则

**验证时机:** 配置加载后、使用前

**验证项:**

1. **必需字段存在** — 例如 `default_provider` 必须在 `providers` 中有对应的配置
2. **Provider 配置完整** — 每个 Provider 必须有 `type` 和 `api_key`(除非是 Ollama)
3. **路径有效性** — `workspace_root` 必须是有效的目录
4. **值范围合法** — `temperature` 必须在 `[0.0, 2.0]`,`max_tokens` 必须 > 0

**Rust 实现:**

```rust
impl Config {
    pub fn validate(&self) -> Result<(), ConfigError> {
        // 1. 验证 default_provider 存在
        if let Some(default) = &self.global.default_provider {
            if !self.providers.providers.contains_key(default) {
                return Err(ConfigError::InvalidDefaultProvider {
                    provider: default.clone(),
                    available: self.providers.providers.keys().cloned().collect(),
                });
            }
        }

        // 2. 验证每个 Provider
        for (name, provider) in &self.providers.providers {
            if provider.api_key.is_none() && provider.provider_type != "ollama" {
                return Err(ConfigError::MissingApiKey {
                    provider: name.clone(),
                });
            }

            if let Some(temp) = provider.temperature {
                if !(0.0..=2.0).contains(&temp) {
                    return Err(ConfigError::InvalidTemperature {
                        provider: name.clone(),
                        value: temp,
                    });
                }
            }
        }

        Ok(())
    }
}

#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("Default provider '{provider}' not found. Available: {available:?}")]
    InvalidDefaultProvider {
        provider: String,
        available: Vec<String>,
    },

    #[error("Provider '{provider}' is missing API key")]
    MissingApiKey { provider: String },

    #[error("Provider '{provider}' has invalid temperature {value} (must be in [0.0, 2.0])")]
    InvalidTemperature { provider: String, value: f32 },
}
```

### 12.6.2 错误信息示例

**场景 1:** 默认 Provider 不存在

```bash
$ loom
Error: Default provider 'gemini' not found. Available: ["anthropic", "openai", "ollama"]

Hint: Check your config file at ~/.config/loom/config.toml
```

**场景 2:** API Key 缺失

```bash
$ loom
Error: Provider 'anthropic' is missing API key

Hint: Set ANTHROPIC_API_KEY environment variable or add 'api_key' to config:
  [providers.anthropic]
  api_key = "sk-ant-..."
```

---

## 12.7 本章小结

### 12.7.1 核心知识点回顾

1. **XDG 路径规范** — 配置在 `~/.config/loom/`,数据在 `~/.local/share/loom/`,缓存在 `~/.cache/loom/`
2. **分层配置** — 6 层优先级(CLI 参数 > 环境变量 > 工作区 > 用户 > 系统 > 默认值)
3. **深度合并** — TOML table 深度合并,标量值替换,数组替换
4. **自动创建配置** — 首次运行时自动生成 `config.toml` 模板
5. **Secret<T> 类型** — Redacted Debug/Display/Serialize,Explicit Access via `.expose()`,Zeroize on Drop
6. **`*_FILE` 加载** — 支持 Kubernetes/Docker Secret 挂载模式
7. **配置验证** — 加载后验证必需字段和值范围

### 12.7.2 与其他章节的呼应

| 章节 | 在本章的体现 |
|------|------------|
| ch04 | LLM Provider 的 API Key 使用 `SecretString` 包装 |
| ch08 | Thread 存储路径使用 XDG Data Home |
| ch11 | OAuth token 和私钥使用 `Secret<T>` 管理 |

### 12.7.3 未覆盖的主题

以下主题在本章中仅简要提及,将在后续章节详细讨论:

- **ch11 认证与授权** — OAuth token 的 Secret 管理和刷新
- **ch14 SCM 与 Git 集成** — Git credential helper 集成
- **ch18 部署架构** — 服务端配置的环境变量和 Secret 注入

---

## 12.8 质检清单

在完成本章后,请验证以下内容:

- [x] **XDG 路径** — 是否讲解了 `XDG_CONFIG_HOME`、`XDG_DATA_HOME`、`XDG_CACHE_HOME` 的映射?
- [x] **分层配置** — 是否定义了 6 层优先级和合并规则?
- [x] **自动创建配置** — 是否说明了首次运行时的默认配置生成?
- [x] **Secret<T> 设计** — 是否讲解了 Redacted Debug/Display/Serialize 的实现?
- [x] **`*_FILE` 加载** — 是否说明了 Kubernetes Secret 挂载模式?
- [x] **与 tracing 集成** — 是否展示了 Secret 在日志中自动 Redact?
- [x] **配置验证** — 是否讨论了验证规则和错误信息?

---

**下一章预告:** ch13 将讨论 **TUI 组件体系:Ratatui 视觉快照测试**,展示 Loom CLI 的终端用户界面如何通过组件树、状态管理和快照测试实现可靠的交互体验。
