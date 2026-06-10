<!-- SPDX-FileCopyrightText: 2026 SPHARX Ltd. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# AgentOS Rust 编码风格标准

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/coding_standard/Rust_coding_style_standard.md
**作者**:
    - Liren Wang
---

## 目录

1. [总则与适用范围](#1-总则与适用范围)
2. [文件组织与模块结构](#2-文件组织与模块结构)
3. [命名规范](#3-命名规范)
4. [类型设计](#4-类型设计)
5. [函数设计](#5-函数设计)
6. [错误处理](#6-错误处理)
7. [并发编程](#7-并发编程)
8. [序列化与反序列化](#8-序列化与反序列化)
9. [文档注释规范](#9-文档注释规范)
10. [测试规范](#10-测试规范)
11. [依赖管理](#11-依赖管理)
12. [AgentOS 模块编码示例](#12-agentos-模块编码示例)

---

## 1. 总则与适用范围

### 1.1 适用范围

本标准适用于 AgentOS 项目中所有 Rust 代码的编写，包括但不限于：

- AgentOS Rust SDK 核心库（`agentos-rs`）
- 基于 SDK 的上层应用与工具
- 与 Go SDK / Python SDK 保持跨语言一致性的接口层

### 1.2 核心原则

本标准遵循 AgentOS 架构的三大核心原则：

1. **五维正交体系**：模块间职责正交、依赖单向、边界清晰。每个模块只属于一个维度（核心循环 / 认知层 / 执行层 / 记忆层 / 系统调用），禁止跨维度耦合。
2. **Thinkdual 认知双思系统**：代码设计必须同时考虑"声明式意图"与"命令式执行"两条路径。API 设计优先表达意图（what），实现层负责执行（how）。
3. **安全穹顶 Cupolas**：所有外部输入必须经过验证，所有错误必须可追踪至错误码体系，禁止静默丢弃错误。

### 1.3 规则等级

| 等级 | 关键词 | 含义 |
|------|--------|------|
| 强制 | **必须** | 违反将导致 PR 拒绝 |
| 强制 | **禁止** | 违反将导致 PR 拒绝 |
| 推荐 | **应当** | 建议遵循，特殊情况可偏离但需注释说明 |
| 允许 | **可以** | 可选做法 |

### 1.4 强制工具链

| 工具 | 用途 | 执行时机 |
|------|------|----------|
| `cargo clippy` | 静态代码分析 | CI 必过 |
| `cargo fmt` | 代码格式化 | pre-commit |
| `cargo deny` | 依赖安全审计 | CI 必过 |
| `cargo audit` | 漏洞扫描 | CI 必过 |
| `cargo test` | 单元测试 | CI 必过 |

---

## 2. 文件组织与模块结构

### 2.1 目录结构

AgentOS Rust SDK **必须**遵循以下目录结构：

```
src/
├── lib.rs              # 入口：模块声明 + pub use 重导出
├── error.rs            # 统一错误体系（AgentOSError + 错误码常量）
├── protocol.rs         # 协议层（ProtocolClient, ProtocolType）
├── syscall.rs          # 系统调用绑定（SyscallBinding trait + 域 Syscall）
├── telemetry.rs        # 遥测（Meter + Tracer）
├── agent.rs            # Agent 入口
├── plugin.rs           # 插件框架（BasePlugin trait + PluginRegistry）
├── client/             # HTTP 客户端
│   ├── mod.rs          # 模块声明 + pub use
│   └── client.rs       # Client + ClientBuilder + APIClient trait
├── types/              # 类型定义
│   ├── mod.rs          # 模块声明 + pub use
│   └── types.rs        # 枚举、领域模型、请求/响应结构
├── utils/              # 工具函数
│   ├── mod.rs          # 模块声明 + pub use
│   └── helpers.rs      # 通用辅助函数
└── modules/            # 业务模块管理器
    ├── mod.rs          # 模块声明 + pub use
    ├── task/           # 任务管理
    ├── memory/         # 记忆管理
    ├── session/        # 会话管理
    └── skill/          # 技能管理
```

### 2.2 `lib.rs` 结构规范

`lib.rs` **必须**按以下顺序组织：

1. 文件头注释（版本、描述、对应 Go SDK 文件）
2. 模块声明（`pub mod`）
3. 版本/元信息常量
4. `pub use` 重导出（按类别分组，使用中文注释标注分组）

```
// ✅ 正确：lib.rs 按类别分组导出
// 客户端层
pub use client::{APIClient, Client};

// 错误类型
pub use error::{AgentOSError, ErrorCode};

// Syscall 绑定
pub use syscall::{
    SyscallBinding, HttpSyscallBinding,
    SyscallNamespace, SyscallRequest, SyscallResponse,
    TaskSyscall, MemorySyscall, SessionSyscall, SkillSyscall, AgentSyscall,
};
```

```
// ❌ 禁止：无分组的扁平导出
pub use client::Client;
pub use error::AgentOSError;
pub use syscall::SyscallBinding;
pub use types::TaskStatus;
```

### 2.3 子目录 `mod.rs` 规范

子目录的 `mod.rs` **必须**声明子模块并通过 `pub use` 重导出公共类型：

```rust
// ✅ 正确：modules/mod.rs
pub mod task;
pub mod memory;
pub mod session;
pub mod skill;

pub use task::TaskManager;
pub use memory::MemoryManager;
pub use session::SessionManager;
pub use skill::SkillManager;
```

**禁止**在 `mod.rs` 中放置业务逻辑代码。

### 2.4 废弃模块处理

当模块迁移后，旧模块**必须**使用 `#[deprecated]` 标注并指向新模块：

```rust
// ✅ 正确
#[deprecated(since = "0.1.0", note = "请使用 modules::task::TaskManager")]
pub mod task;
```

```rust
// ❌ 禁止：直接删除旧模块或无 deprecated 标注
pub mod task; // 无 deprecated 标注
```

### 2.5 文件头注释

每个 `.rs` 文件**必须**包含文件头注释：

```rust
// ✅ 正确
// AgentOS Rust SDK - HTTP 客户端实现
// Version: 0.1.0
// Last updated: 2026-03-24
//
// 提供 HTTP 通信层、APIClient 接口定义和依赖倒转抽象。
// 对应 Go SDK: client/client.go
```

```rust
// ❌ 禁止：无文件头或缺少版本/对应关系
// client implementation
```

**强制执行**：CI 中通过脚本检查每个 `.rs` 文件是否包含 `Version:` 和 `Last updated:` 行。

---

## 3. 命名规范

### 3.1 模块命名

模块名**必须**使用 `snake_case`：

```rust
// ✅ 正确
pub mod client;
pub mod types;
pub mod modules;
pub mod protocol;
```

```rust
// ❌ 禁止
pub mod Client;      // PascalCase
pub mod client_mod;  // 无意义后缀
pub mod httpClient;  // camelCase
```

### 3.2 类型命名

类型名（struct、enum、trait）**必须**使用 `PascalCase`：

```rust
// ✅ 正确
pub struct Client { ... }
pub struct TaskManager { ... }
pub enum AgentOSError { ... }
pub trait SyscallBinding { ... }
pub struct PluginRegistry { ... }
```

```rust
// ❌ 禁止
pub struct client { ... }          // snake_case
pub struct Task_Manager { ... }    // 下划线分隔
pub enum agentos_error { ... }     // snake_case
```

### 3.3 枚举变体命名

枚举变体**必须**使用 `PascalCase`：

```rust
// ✅ 正确
pub enum TaskStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Cancelled,
}

pub enum SyscallNamespace {
    Task,
    Memory,
    Session,
    Skill,
    Agent,
    Telemetry,
}
```

```rust
// ❌ 禁止
pub enum TaskStatus {
    PENDING,       // SCREAMING_SNAKE_CASE
    running,       // snake_case
    IsCompleted,   // 布尔前缀
}
```

### 3.4 函数命名

函数名**必须**使用 `snake_case`：

```rust
// ✅ 正确
pub fn new(endpoint: &str) -> Result<Self, AgentOSError> { ... }
pub fn submit(&self, description: &str) -> Result<Task, AgentOSError> { ... }
pub fn with_code(code: &str, message: &str) -> Self { ... }
pub fn is_terminal(&self) -> bool { ... }
pub fn http_status_to_code(status: u16) -> ErrorCode { ... }
```

```rust
// ❌ 禁止
pub fn New() { ... }              // PascalCase
pub fn SubmitTask() { ... }       // PascalCase
pub fn getTask() { ... }          // camelCase
```

#### 3.4.1 工厂方法命名

错误类型的工厂方法**必须**使用小写领域名称：

```rust
// ✅ 正确
AgentOSError::network(msg)
AgentOSError::http(msg)
AgentOSError::json(msg)
AgentOSError::task(msg)
AgentOSError::timeout(msg)
```

```rust
// ❌ 禁止
AgentOSError::Network(msg)     // PascalCase
AgentOSError::new_network(msg) // new_ 前缀冗余
AgentOSError::from_net(msg)    // 非标准缩写
```

### 3.5 常量命名

常量**必须**使用 `SCREAMING_SNAKE_CASE`：

```rust
// ✅ 正确
pub const VERSION: &str = "0.1.0";
pub const CODE_SUCCESS: &str = "0x0000";
pub const CODE_TASK_FAILED: &str = "0x3001";
pub const MAX_RESPONSE_BODY_SIZE: usize = 10 * 1024 * 1024;
```

```rust
// ❌ 禁止
pub const Version: &str = "0.1.0";          // PascalCase
pub const codeSuccess: &str = "0x0000";     // camelCase
pub const code_task_failed: &str = "0x3001"; // snake_case
```

#### 3.5.1 错误码常量命名

错误码常量**必须**以 `CODE_` 为前缀，并按十六进制域分类：

| 前缀 | 域 | 示例 |
|------|----|------|
| `CODE_` + 通用名 | 0x0xxx 通用 | `CODE_SUCCESS`, `CODE_TIMEOUT`, `CODE_NETWORK_ERROR` |
| `CODE_LOOP_` | 0x1xxx 核心循环 | `CODE_LOOP_CREATE_FAILED` |
| `CODE_` + 认知名 | 0x2xxx 认知层 | `CODE_COGNITION_FAILED`, `CODE_DAG_BUILD_FAILED` |
| `CODE_TASK_` | 0x3xxx 执行层 | `CODE_TASK_FAILED`, `CODE_TASK_TIMEOUT` |
| `CODE_MEMORY_` / `CODE_SESSION_` / `CODE_SKILL_` | 0x4xxx 记忆层 | `CODE_MEMORY_NOT_FOUND` |
| `CODE_` + 系统名 | 0x5xxx 系统调用 | `CODE_TELEMETRY_ERROR`, `CODE_SYSCALL_ERROR` |
| `CODE_` + 安全名 | 0x6xxx 安全域 | `CODE_PERMISSION_DENIED`, `CODE_CORRUPTED_DATA` |

### 3.6 字段命名

结构体字段**必须**使用 `snake_case`。当 JSON 字段名与 Rust 惯例不同时，**必须**使用 `#[serde(rename = "...")]` 映射：

```rust
// ✅ 正确
pub struct Task {
    #[serde(rename = "task_id")]
    pub id: String,
    pub description: String,
    pub status: TaskStatus,
    pub created_at: String,
    pub updated_at: String,
}

pub struct Memory {
    #[serde(rename = "memory_id")]
    pub id: String,
    pub content: String,
    pub layer: MemoryLayer,
}
```

```rust
// ❌ 禁止：Rust 字段使用 camelCase 或直接使用 JSON 字段名
pub struct Task {
    pub taskId: String,       // camelCase
    pub task_id: String,      // 无 rename，但 JSON 字段也是 task_id 时不需 rename
}
```

### 3.7 类型别名

类型别名**必须**使用 `PascalCase`（与类型命名一致）：

```rust
// ✅ 正确
pub type ErrorCode = &'static str;
pub type PluginFactory = Box<dyn Fn() -> Box<dyn BasePlugin> + Send + Sync>;
```

```rust
// ❌ 禁止
pub type errorCode = &'static str;   // camelCase
pub type ERROR_CODE = &'static str;  // SCREAMING_SNAKE_CASE
```

### 3.8 生命周期参数

生命周期参数**必须**使用简短小写名称，**禁止**使用无意义的多字符名：

```rust
// ✅ 正确
fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { ... }
pub fn endpoint(&self) -> &str { ... }
```

```rust
// ❌ 禁止
fn fmt<'lifetime>(&self, f: &mut std::fmt::Formatter<'lifetime>) -> ...  // 过长
```

**强制执行**：`cargo clippy` 默认检查命名规范。

---

## 4. 类型设计

### 4.1 枚举设计

#### 4.1.1 状态枚举

状态枚举**必须**派生 `Debug, Clone, Copy, PartialEq, Eq`，并实现 `Display` 和 `as_str()`：

```rust
// ✅ 正确
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum TaskStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Cancelled,
}

impl TaskStatus {
    pub fn as_str(&self) -> &'static str {
        match self {
            TaskStatus::Pending => "pending",
            TaskStatus::Running => "running",
            TaskStatus::Completed => "completed",
            TaskStatus::Failed => "failed",
            TaskStatus::Cancelled => "cancelled",
        }
    }

    pub fn is_terminal(&self) -> bool {
        matches!(self, TaskStatus::Completed | TaskStatus::Failed | TaskStatus::Cancelled)
    }
}

impl std::fmt::Display for TaskStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.as_str())
    }
}
```

```rust
// ❌ 禁止
pub enum TaskStatus {
    PENDING,   // SCREAMING_SNAKE_CASE
    RUNNING,
}

// 缺少 Display 实现
// 缺少 as_str() 方法
```

#### 4.1.2 命名空间枚举

命名空间枚举**必须**实现 `Display`，输出值与 Syscall 路径一致：

```rust
// ✅ 正确
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum SyscallNamespace {
    #[serde(rename = "task")]
    Task,
    #[serde(rename = "memory")]
    Memory,
    // ...
}

impl std::fmt::Display for SyscallNamespace {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            SyscallNamespace::Task => write!(f, "task"),
            SyscallNamespace::Memory => write!(f, "memory"),
            // ...
        }
    }
}
```

#### 4.1.3 错误枚举

错误枚举**必须**使用 `thiserror::Error` 派生，**禁止**使用 `anyhow`：

```rust
// ✅ 正确
#[derive(Error, Debug, Clone)]
pub enum AgentOSError {
    #[error("[{code}] {message}")]
    WithCode { code: String, message: String },

    #[error("[{}] {}", CODE_NETWORK_ERROR, .0)]
    Network(String),

    #[error("[{}] {}", CODE_TIMEOUT, .0)]
    Timeout(String),
}
```

```rust
// ❌ 禁止
use anyhow::{anyhow, Result};  // 禁止使用 anyhow

pub enum AgentOSError {
    Network(String),  // 缺少 #[error(...)] 属性
}
```

### 4.2 结构体设计

#### 4.2.1 领域模型

领域模型结构体**必须**派生 `Debug, Clone, Serialize, Deserialize`：

```rust
// ✅ 正确
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    #[serde(rename = "task_id")]
    pub id: String,
    pub description: String,
    pub status: TaskStatus,
    #[serde(default)]
    pub priority: i32,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub output: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<String>,
    #[serde(default)]
    pub metadata: HashMap<String, serde_json::Value>,
    pub created_at: String,
    pub updated_at: String,
}
```

**禁止**在领域模型中包含业务方法（如 `save()`、`delete()`），业务方法**必须**放在对应的 Manager 中。

#### 4.2.2 请求/响应结构

请求/响应结构体**必须**区分必填字段与可选字段：

```rust
// ✅ 正确
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct APIResponse {
    pub success: bool,                              // 必填
    pub data: serde_json::Value,                    // 必填
    #[serde(skip_serializing_if = "Option::is_none")]
    pub message: Option<String>,                    // 可选
}
```

```rust
// ❌ 禁止
pub struct APIResponse {
    pub success: bool,
    pub data: Value,
    pub message: String,  // 必填但实际可能为空
}
```

#### 4.2.3 配置结构

配置结构体**必须**实现 `Default`：

```rust
// ✅ 正确
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProtocolConfig {
    pub protocol_type: ProtocolType,
    pub endpoint: String,
    pub timeout: Duration,
    pub retry_count: u32,
    // ...
}

impl Default for ProtocolConfig {
    fn default() -> Self {
        let endpoint = std::env::var("AGENTOS_ENDPOINT")
            .unwrap_or_else(|_| "http://127.0.0.1:18789".to_string());
        Self {
            protocol_type: ProtocolType::JsonRpc,
            endpoint,
            timeout: Duration::from_secs(30),
            retry_count: 3,
            retry_delay: Duration::from_secs(1),
            enable_streaming: false,
            extra_headers: HashMap::new(),
        }
    }
}
```

### 4.3 类型别名

模块内部**可以**定义 `Result` 类型别名以简化签名：

```rust
// ✅ 正确：syscall.rs 内部
type Result<T> = std::result::Result<T, AgentOSError>;
```

**禁止**在 `lib.rs` 或公共 API 中暴露自定义 `Result` 类型别名。

**强制执行**：`cargo clippy` 检查 derive 完整性；CI 检查 `anyhow` 依赖是否被引入。

---

## 5. 函数设计

### 5.1 构造函数

所有主要类型**必须**提供 `new()` 关联函数作为主构造入口：

```rust
// ✅ 正确
impl Client {
    pub fn new(endpoint: &str) -> Result<Self, AgentOSError> {
        Self::builder(endpoint).build()
    }
}

impl TaskManager {
    pub fn new(api: Arc<dyn APIClient>) -> Self {
        TaskManager { api }
    }
}

impl PluginRegistry {
    pub fn new() -> Self {
        Self {
            factories: HashMap::new(),
            instances: HashMap::new(),
            manifests: HashMap::new(),
            states: HashMap::new(),
        }
    }
}
```

```rust
// ❌ 禁止
impl Client {
    pub fn create() -> Self { ... }  // 非标准命名
}
```

### 5.2 Builder 模式

当构造参数超过 3 个时，**必须**使用 Builder 模式。Builder **必须**：

1. 作为独立 `pub struct` 定义
2. 所有字段私有
3. 提供 `new()` 初始化默认值
4. 链式 setter 返回 `Self`
5. `build()` 返回 `Result<T, AgentOSError>`

```rust
// ✅ 正确
pub struct ClientBuilder {
    endpoint: String,
    timeout: Duration,
    max_retries: u32,
    retry_delay: Duration,
    api_key: Option<String>,
    user_agent: String,
    max_connections: usize,
    idle_conn_timeout: Duration,
}

impl ClientBuilder {
    pub fn new(endpoint: &str) -> Self { ... }
    pub fn timeout(mut self, timeout: Duration) -> Self { ... }
    pub fn max_retries(mut self, max_retries: u32) -> Self { ... }
    pub fn api_key(mut self, api_key: &str) -> Self { ... }
    pub fn build(self) -> Result<Client, AgentOSError> { ... }
}
```

```rust
// ❌ 禁止：构造函数参数过多
impl Client {
    pub fn new(endpoint: &str, timeout: Duration, retries: u32,
               api_key: Option<&str>, user_agent: &str) -> Result<Self, AgentOSError> { ... }
}

// ❌ 禁止：Builder::build() 返回 Self 而非 Result
impl ClientBuilder {
    pub fn build(self) -> Client { ... }  // 无法报告验证错误
}
```

### 5.3 函数选项模式（Functional Options）

当函数需要可选配置时，**必须**使用函数选项模式：

```rust
// ✅ 正确
pub type RequestOption = Box<dyn Fn(&mut RequestOptions) + Send + Sync>;

pub fn with_request_timeout(timeout: Duration) -> RequestOption {
    Box::new(move |opts: &mut RequestOptions| {
        opts.timeout = Some(timeout);
    })
}

pub fn with_header(key: String, value: String) -> RequestOption {
    Box::new(move |opts: &mut RequestOptions| {
        opts.headers.insert(key.clone(), value.clone());
    })
}

// 使用
client.get("/api/v1/tasks", Some(vec![
    with_request_timeout(Duration::from_secs(10)),
    with_header("X-Custom".to_string(), "value".to_string()),
])).await?;
```

```rust
// ❌ 禁止：使用 HashMap 传递可选参数
client.get("/api/v1/tasks", Some(hashmap!{
    "timeout" => "10",
    "header_X-Custom" => "value",
})).await?;
```

选项函数**必须**以 `with_` 为前缀命名。

### 5.4 泛型约束

泛型约束**必须**在 `impl` 块而非类型定义处指定（除非类型本身需要约束）：

```rust
// ✅ 正确：SyscallBinding 约束在 impl 处
pub struct TaskSyscall<B: SyscallBinding> {
    binding: B,
}

impl<B: SyscallBinding> TaskSyscall<B> {
    pub fn new(binding: B) -> Self { ... }
    pub fn submit(&self, description: &str) -> Result<SyscallResponse> { ... }
}
```

```rust
// ❌ 禁止：不必要的约束
pub struct TaskSyscall<B: SyscallBinding + Clone + Debug> {  // 过度约束
    binding: B,
}
```

### 5.5 参数验证

公共 API 函数**必须**在入口处验证参数，使用 `AgentOSError::with_code(CODE_*, msg)` 返回错误：

```rust
// ✅ 正确
pub async fn submit(&self, description: &str) -> Result<Task, AgentOSError> {
    if description.is_empty() {
        return Err(AgentOSError::with_code(CODE_MISSING_PARAMETER, "任务描述不能为空"));
    }
    // ...
}

pub async fn get(&self, task_id: &str) -> Result<Task, AgentOSError> {
    if task_id.is_empty() {
        return Err(AgentOSError::with_code(CODE_MISSING_PARAMETER, "任务ID不能为空"));
    }
    // ...
}
```

```rust
// ❌ 禁止：使用 panic 或 unwrap 处理用户输入
pub async fn submit(&self, description: &str) -> Result<Task, AgentOSError> {
    assert!(!description.is_empty());  // 禁止在公共 API 中使用 assert
    // ...
}
```

**强制执行**：`cargo clippy` 检查 `unwrap()` 在非测试代码中的使用。

---

## 6. 错误处理

### 6.1 thiserror 使用规范

**必须**使用 `thiserror::Error` 派生宏定义错误类型，**禁止**使用 `anyhow`：

```rust
// ✅ 正确
use thiserror::Error;

#[derive(Error, Debug, Clone)]
pub enum AgentOSError {
    #[error("[{code}] {message}")]
    WithCode { code: String, message: String },

    #[error("[{}] {}", CODE_NETWORK_ERROR, .0)]
    Network(String),

    #[error("[{}] {}", CODE_PARSE_ERROR, .0)]
    Json(String),
}
```

```rust
// ❌ 禁止
use anyhow::{anyhow, Result, Error};

fn do_something() -> anyhow::Result<()> {
    Err(anyhow!("something failed"))  // 无结构化错误码
}
```

**理由**：安全穹顶 Cupolas 原则要求所有错误可追踪至错误码体系。`anyhow` 无法保证这一点。

### 6.2 Result 类型

所有可能失败的公共函数**必须**返回 `Result<T, AgentOSError>`：

```rust
// ✅ 正确
pub async fn submit(&self, description: &str) -> Result<Task, AgentOSError> { ... }
pub async fn health(&self) -> Result<HealthStatus, AgentOSError> { ... }
pub fn build(self) -> Result<Client, AgentOSError> { ... }
```

```rust
// ❌ 禁止
pub async fn submit(&self, description: &str) -> Task { ... }   // 无错误处理
pub async fn submit(&self, description: &str) -> Option<Task> { ... }  // 丢失错误信息
```

模块内部**可以**定义 `Result` 别名：

```rust
// ✅ 正确：仅限模块内部
type Result<T> = std::result::Result<T, AgentOSError>;
```

### 6.3 错误码体系

错误码**必须**遵循十六进制分类体系，与 Go SDK 保持一致：

```
0x0xxx  通用错误 (General)
0x1xxx  核心循环错误 (CoreLoop)
0x2xxx  认知层错误 (Cognition)
0x3xxx  执行层错误 (Execution)
0x4xxx  记忆层错误 (Memory)
0x5xxx  系统调用错误 (Syscall)
0x6xxx  安全域错误 (Security)
0x7xxx  动态模块错误 (Gateway)
```

错误码**必须**定义为 `pub const CODE_XXX: &str = "0xNNNN";`：

```rust
// ✅ 正确
pub const CODE_SUCCESS: &str = "0x0000";
pub const CODE_NETWORK_ERROR: &str = "0x000A";
pub const CODE_TASK_FAILED: &str = "0x3001";
pub const CODE_MEMORY_NOT_FOUND: &str = "0x4001";
```

```rust
// ❌ 禁止
pub const CODE_SUCCESS: u16 = 0;        // 必须为十六进制字符串
pub const CODE_SUCCESS: &str = "0";     // 必须带 0x 前缀
pub const CODE_SUCCESS: &str = "0X0000"; // 必须小写 0x
```

### 6.4 错误工厂方法

`AgentOSError` **必须**为每个领域提供工厂方法：

```rust
// ✅ 正确
impl AgentOSError {
    pub fn network(message: &str) -> Self { ... }
    pub fn http(message: &str) -> Self { ... }
    pub fn json(message: &str) -> Self { ... }
    pub fn task(message: &str) -> Self { ... }
    pub fn timeout(message: &str) -> Self { ... }
    pub fn not_found(message: &str) -> Self { ... }
    pub fn invalid_parameter(message: &str) -> Self { ... }
    pub fn with_code(code: &str, message: &str) -> Self { ... }
}
```

```rust
// ❌ 禁止：直接构造枚举变体
return Err(AgentOSError::Network("连接失败".to_string()));  // 应使用工厂方法
```

**推荐**使用工厂方法而非直接构造变体，以便未来在工厂方法中添加日志/追踪逻辑。

### 6.5 From 实现

外部错误类型**必须**通过 `From<T> for AgentOSError` 转换，**禁止**在调用处手动映射：

```rust
// ✅ 正确
impl From<reqwest::Error> for AgentOSError {
    fn from(err: reqwest::Error) -> Self {
        if err.is_timeout() {
            AgentOSError::timeout(&err.to_string())
        } else if err.is_connect() {
            AgentOSError::connection_refused(&err.to_string())
        } else if err.is_request() {
            AgentOSError::network(&err.to_string())
        } else if err.is_status() {
            AgentOSError::http(&err.to_string())
        } else if err.is_body() || err.is_decode() {
            AgentOSError::json(&err.to_string())
        } else {
            AgentOSError::Other(err.to_string())
        }
    }
}

impl From<serde_json::Error> for AgentOSError {
    fn from(err: serde_json::Error) -> Self {
        AgentOSError::json(&err.to_string())
    }
}

impl From<std::io::Error> for AgentOSError {
    fn from(err: std::io::Error) -> Self {
        AgentOSError::with_code(CODE_INTERNAL, &err.to_string())
    }
}
```

```rust
// ❌ 禁止：手动映射
let resp = client.send().await.map_err(|e| {
    if e.is_timeout() {
        AgentOSError::Timeout(e.to_string())
    } else {
        AgentOSError::Other(e.to_string())
    }
})?;
```

### 6.6 HTTP 状态码映射

**必须**提供 `http_status_to_code()` 和 `http_status_to_error()` 函数：

```rust
// ✅ 正确
pub fn http_status_to_code(status: u16) -> ErrorCode {
    match status {
        400 => CODE_INVALID_PARAMETER,
        401 => CODE_UNAUTHORIZED,
        403 => CODE_FORBIDDEN,
        404 => CODE_NOT_FOUND,
        408 => CODE_TIMEOUT,
        409 => CODE_CONFLICT,
        422 => CODE_VALIDATION_ERROR,
        429 => CODE_RATE_LIMITED,
        500 | 502 | 503 => CODE_SERVER_ERROR,
        504 => CODE_TIMEOUT,
        _ => CODE_UNKNOWN,
    }
}
```

**强制执行**：CI 检查 `Cargo.toml` 中是否引入 `anyhow`；`cargo clippy` 检查 `unwrap()` 使用。

---

## 7. 并发编程

### 7.1 Arc<Mutex> 模式

共享可变状态**必须**使用 `Arc<Mutex<T>>`：

```rust
// ✅ 正确
pub struct Telemetry {
    service_name: String,
    meter: Arc<Mutex<Meter>>,
    tracer: Arc<Mutex<Tracer>>,
}

impl Telemetry {
    pub fn meter(&self) -> Arc<Mutex<Meter>> {
        Arc::clone(&self.meter)
    }
}
```

```rust
// ❌ 禁止
pub struct Telemetry {
    meter: RefCell<Meter>,  // RefCell 非线程安全
}

pub struct Telemetry {
    meter: Mutex<Meter>,    // 缺少 Arc，无法跨线程共享
}
```

### 7.2 Send + Sync 约束

跨线程传递的 trait **必须**添加 `Send + Sync` 约束：

```rust
// ✅ 正确
#[async_trait::async_trait]
pub trait APIClient: Send + Sync {
    async fn get(&self, path: &str, opts: Option<Vec<RequestOption>>) -> Result<APIResponse, AgentOSError>;
    // ...
}

pub trait BasePlugin: Send + Sync {
    fn plugin_id(&self) -> &str;
    // ...
}

pub type PluginFactory = Box<dyn Fn() -> Box<dyn BasePlugin> + Send + Sync>;
```

```rust
// ❌ 禁止
pub trait APIClient {
    async fn get(&self, path: &str) -> Result<APIResponse, AgentOSError>;
    // 缺少 Send + Sync，无法放入 Arc<dyn APIClient>
}
```

### 7.3 依赖倒转与 Arc<dyn Trait>

Manager 层**必须**通过 `Arc<dyn APIClient>` 持有客户端引用，实现依赖倒转：

```rust
// ✅ 正确
pub struct TaskManager {
    api: Arc<dyn APIClient>,
}

impl TaskManager {
    pub fn new(api: Arc<dyn APIClient>) -> Self {
        TaskManager { api }
    }
}
```

```rust
// ❌ 禁止：直接依赖具体类型
pub struct TaskManager {
    client: Client,  // 耦合具体实现
}
```

### 7.4 Lazy 单例

全局单例**必须**使用 `once_cell::sync::Lazy`：

```rust
// ✅ 正确
use once_cell::sync::Lazy;

static GLOBAL_REGISTRY: Lazy<Mutex<PluginRegistry>> = Lazy::new(|| Mutex::new(PluginRegistry::new()));

pub fn with_plugin_registry<F, R>(f: F) -> R
where
    F: FnOnce(&mut PluginRegistry) -> R,
{
    let mut guard = GLOBAL_REGISTRY.lock().expect("plugin registry mutex poisoned");
    f(&mut guard)
}
```

```rust
// ❌ 禁止
static mut REGISTRY: Option<PluginRegistry> = None;  // unsafe 全局变量

lazy_static! { ... }  // 使用已弃用的 lazy_static crate
```

### 7.5 async-trait

异步 trait **必须**使用 `async-trait` crate：

```rust
// ✅ 正确
#[async_trait::async_trait]
pub trait APIClient: Send + Sync {
    async fn get(&self, path: &str, opts: Option<Vec<RequestOption>>) -> Result<APIResponse, AgentOSError>;
    async fn post(&self, path: &str, body: Option<&Value>, opts: Option<Vec<RequestOption>>) -> Result<APIResponse, AgentOSError>;
}
```

```rust
// ❌ 禁止：手动返回 Pin<Box<dyn Future>>
fn get(&self, path: &str) -> Pin<Box<dyn Future<Output = Result<APIResponse, AgentOSError>> + Send + '_>>;
```

### 7.6 Mutex 中毒处理

`Mutex::lock()` 的中毒**必须**使用 `expect()` 处理，**禁止**使用 `unwrap()` 或忽略：

```rust
// ✅ 正确
let mut guard = GLOBAL_REGISTRY.lock().expect("plugin registry mutex poisoned");
```

```rust
// ❌ 禁止
let guard = GLOBAL_REGISTRY.lock().unwrap();  // 无上下文信息
let guard = GLOBAL_REGISTRY.lock().expect("");  // 空消息
```

**强制执行**：`cargo clippy` 检查 `Send + Sync` 约束；CI 检查 `lazy_static` 依赖。

---

## 8. 序列化与反序列化

### 8.1 Serde 派生

需要序列化的类型**必须**派生 `Serialize, Deserialize`：

```rust
// ✅ 正确
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task { ... }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TaskStatus { ... }
```

```rust
// ❌ 禁止：手动实现 Serialize/Deserialize
impl Serialize for Task {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: Serializer { ... }
}
```

### 8.2 JSON 字段映射

当 Rust 字段名与 JSON 键名不同时，**必须**使用 `#[serde(rename = "...")]`：

```rust
// ✅ 正确
pub struct Task {
    #[serde(rename = "task_id")]
    pub id: String,
}

pub struct Memory {
    #[serde(rename = "memory_id")]
    pub id: String,
}

pub struct MemorySearchResult {
    #[serde(rename = "top_k")]
    pub top_k: i32,
}
```

### 8.3 枚举序列化

枚举序列化**必须**使用 `#[serde(rename_all = "...")]` 统一命名风格：

```rust
// ✅ 正确：统一转为 lowercase
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum TaskStatus {
    Pending,    // → "pending"
    Running,    // → "running"
    Completed,  // → "completed"
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum ProtocolType {
    JsonRpc,    // → "jsonrpc"
    Mcp,        // → "mcp"
    A2a,        // → "a2a"
}
```

```rust
// ❌ 禁止：逐个变体 rename（除非风格不一致）
#[derive(Serialize, Deserialize)]
pub enum TaskStatus {
    #[serde(rename = "pending")]
    Pending,
    #[serde(rename = "running")]
    Running,
    // 应使用 rename_all = "lowercase"
}
```

当枚举变体需要特殊映射时，**可以**逐个使用 `#[serde(rename = "...")]`：

```rust
// ✅ 正确：特殊映射
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum SyscallNamespace {
    #[serde(rename = "task")]
    Task,
    #[serde(rename = "memory")]
    Memory,
    #[serde(rename = "session")]
    Session,
    #[serde(rename = "skill")]
    Skill,
    #[serde(rename = "agent")]
    Agent,
    #[serde(rename = "telemetry")]
    Telemetry,
}
```

### 8.4 默认值处理

可选字段**必须**使用 `Option<T>` + `#[serde(skip_serializing_if = "Option::is_none")]`；集合字段**必须**使用 `#[serde(default)]`：

```rust
// ✅ 正确
pub struct Task {
    #[serde(default)]
    pub priority: i32,                                    // 缺省为 0
    #[serde(skip_serializing_if = "Option::is_none")]
    pub output: Option<String>,                           // 缺省为 null，不序列化
    #[serde(default)]
    pub metadata: HashMap<String, serde_json::Value>,     // 缺省为 {}
}
```

```rust
// ❌ 禁止
pub struct Task {
    pub output: String,           // 必填但实际可能为空
    pub metadata: HashMap<...>,   // 缺少 #[serde(default)]，反序列化时若字段缺失会报错
}
```

### 8.5 跳过序列化

内部字段**必须**使用 `#[serde(skip)]` 防止泄露到 JSON：

```rust
// ✅ 正确
pub struct ProtocolConfig {
    pub protocol_type: ProtocolType,
    pub endpoint: String,
    #[serde(skip)]
    pub extra_headers: HashMap<String, String>,  // 内部使用，不序列化
}
```

**强制执行**：`cargo test` 验证序列化往返一致性。

---

## 9. 文档注释规范

### 9.1 公共 API 文档

所有 `pub` 项（函数、结构体、枚举、trait、常量）**必须**提供 `///` 文档注释：

```rust
// ✅ 正确
/// AgentOSError 是 SDK 所有错误的统一基类
#[derive(Error, Debug, Clone)]
pub enum AgentOSError { ... }

/// 创建新的 AgentOS 客户端
///
/// # 参数
/// - `endpoint`: AgentOS 服务端点地址
///
/// # 返回
/// 返回 Result<Client, AgentOSError>
///
/// # 示例
/// ```rust
/// use agentos_rs::new_client;
///
/// let client = new_client("http://localhost:18789");
/// ```
pub fn new_client(endpoint: &str) -> Result<Client, AgentOSError> { ... }
```

```rust
// ❌ 禁止：无文档的公共 API
pub fn new_client(endpoint: &str) -> Result<Client, AgentOSError> { ... }
```

### 9.2 文档注释结构

函数文档**应当**包含以下部分（按需）：

1. 一行摘要（中文）
2. `# 参数` — 参数说明
3. `# 返回` — 返回值说明
4. `# 错误` — 可能返回的错误
5. `# 示例` — 代码示例

```rust
// ✅ 正确
/// 阻塞等待任务到达终态，支持超时控制
///
/// # 参数
/// - `task_id`: 任务 ID
/// - `timeout`: 超时时间
///
/// # 返回
/// 返回任务结果
///
/// # 错误
/// - 超时返回 `AgentOSError::WithCode { code: CODE_TASK_TIMEOUT, ... }`
pub async fn wait(&self, task_id: &str, timeout: Duration) -> Result<TaskResult, AgentOSError> { ... }
```

### 9.3 文件头注释

每个 `.rs` 文件**必须**包含文件头注释：

```rust
// ✅ 正确
// AgentOS Rust SDK - 统一错误体系
// Version: 0.1.0
// Last updated: 2026-04-05
//
// 定义 SDK 的完整错误类型层级、错误码枚举、哨兵错误和 HTTP 状态码映射。
// 所有异常继承自 AgentOSError，支持错误链追踪。
// 对应 Go SDK: errors.go
```

### 9.4 版本标注

废弃 API **必须**在文档注释中标注版本信息：

```rust
// ✅ 正确
/// 创建新的 AgentOS 客户端
///
/// # 弃用
/// 自 0.2.0 起请使用 `Client::builder(endpoint).build()`
#[deprecated(since = "0.2.0", note = "请使用 Client::builder(endpoint).build()")]
pub fn new_client(endpoint: &str) -> Result<Client, AgentOSError> { ... }
```

### 9.5 中文描述

文档注释**必须**使用中文描述，代码标识符保持英文：

```rust
// ✅ 正确
/// 创建带 API Key 的 AgentOS 客户端
///
/// # 参数
/// - `endpoint`: AgentOS 服务端点地址
/// - `api_key`: API 密钥
pub fn new_with_api_key(endpoint: &str, api_key: &str) -> Result<Client, AgentOSError> { ... }
```

```rust
// ❌ 禁止
/// Create a new AgentOS client with API key
///
/// # Arguments
/// - `endpoint`: The AgentOS service endpoint address
pub fn new_with_api_key(endpoint: &str, api_key: &str) -> Result<Client, AgentOSError> { ... }
```

**强制执行**：`cargo doc` 构建成功；CI 检查 `pub` 项缺少文档注释。

---

## 10. 测试规范

### 10.1 测试模块位置

测试代码**必须**放在同文件的 `#[cfg(test)] mod tests` 中：

```rust
// ✅ 正确
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_version() {
        assert_eq!(VERSION, "0.1.0");
    }
}
```

```rust
// ❌ 禁止：在 src/ 下创建独立测试文件
// src/test_client.rs  ← 禁止
```

集成测试**可以**放在 `tests/` 目录下。

### 10.2 测试函数命名

测试函数**必须**以 `test_` 为前缀，使用描述性名称：

```rust
// ✅ 正确
#[test]
fn test_error_codes_format() { ... }

#[test]
fn test_error_with_code() { ... }

#[test]
fn test_http_status_to_code() { ... }

#[test]
fn test_register_duplicate() { ... }

#[test]
fn test_full_lifecycle() { ... }
```

```rust
// ❌ 禁止
#[test]
fn it_works() { ... }       // 无描述
#[test]
fn test1() { ... }          // 无描述
#[test]
fn TestError() { ... }      // PascalCase
```

### 10.3 断言选择

| 场景 | 使用 | 示例 |
|------|------|------|
| 相等比较 | `assert_eq!` | `assert_eq!(err.code(), CODE_NETWORK_ERROR)` |
| 布尔判断 | `assert!` | `assert!(client.is_ok())` |
| 不相等 | `assert_ne!` | `assert_ne!(id1, id2)` |
| 模式匹配 | `matches!` + `assert!` | `assert!(err.is_network_error())` |
| 错误结果 | `assert!(result.is_err())` | — |

```rust
// ✅ 正确
assert_eq!(CODE_SUCCESS, "0x0000");
assert!(client.is_ok());
assert!(result.is_err());
assert!(TaskStatus::Completed.is_terminal());
assert!(!TaskStatus::Pending.is_terminal());
```

```rust
// ❌ 禁止
assert!(a == b);   // 应使用 assert_eq!
assert_eq!(true, x);  // 应使用 assert!(x)
```

### 10.4 Mock 模式

当需要 Mock 外部依赖时，**必须**通过 trait 实现替代（依赖倒转）：

```rust
// ✅ 正确：通过 APIClient trait Mock
struct MockAPIClient {
    responses: HashMap<String, APIResponse>,
}

#[async_trait::async_trait]
impl APIClient for MockAPIClient {
    async fn get(&self, path: &str, _opts: Option<Vec<RequestOption>>) -> Result<APIResponse, AgentOSError> {
        self.responses.get(path)
            .cloned()
            .ok_or_else(|| AgentOSError::not_found(&format!("mock not found: {}", path)))
    }
    // ...
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_task_manager_submit() {
        let mut mock = MockAPIClient::new();
        mock.add_response("/api/v1/tasks", APIResponse { ... });
        let manager = TaskManager::new(Arc::new(mock));
        // ...
    }
}
```

```rust
// ❌ 禁止：使用 mockall crate 的 auto-mock（增加编译时间）
// ❌ 禁止：直接测试真实 HTTP 端点
```

### 10.5 测试覆盖要求

| 模块 | 最低覆盖率 |
|------|-----------|
| `error.rs` | 90% |
| `types/` | 80% |
| `client/` | 70% |
| `modules/` | 70% |
| `syscall.rs` | 60% |
| `plugin.rs` | 60% |

**强制执行**：`cargo test` 必须通过；CI 使用 `cargo tarpaulin` 检查覆盖率。

---

## 11. 依赖管理

### 11.1 Cargo.toml 规范

`Cargo.toml` **必须**包含完整的 `[package]` 元数据：

```toml
# ✅ 正确
[package]
name = "agentos-rs"
version = "0.1.0"
edition = "2021"
description = "AgentOS Rust SDK - 与 Go SDK 保持一致的模块化结构"
license = "MIT"
homepage = "https://github.com/spharx/agentos"
repository = "https://github.com/spharx/agentos"
authors = ["Spharx AgentOS Team"]
```

```toml
# ❌ 禁止
[package]
name = "agentos-rs"
version = "0.1.0"
edition = "2021"
# 缺少 description, license, repository
```

### 11.2 依赖声明

依赖**必须**按功能分组并添加中文注释：

```toml
# ✅ 正确
[dependencies]
# HTTP 客户端
reqwest = { version = "0.11", features = ["json", "rustls-tls", "blocking", "stream"] }

# 异步运行时
tokio = { version = "1", features = ["full"] }

# 序列化
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# 错误处理
thiserror = "1.0"

# 异步 trait
async-trait = "0.1"

# 时间处理
chrono = { version = "0.4", features = ["serde"] }

# UUID 生成
uuid = { version = "1.0", features = ["v4"] }

# 单例模式
once_cell = "1.19"
```

### 11.3 版本锁定

- 库 crate **禁止**提交 `Cargo.lock`
- 二进制 crate **必须**提交 `Cargo.lock`

### 11.4 特性选择

| 依赖 | 必须特性 | 禁止特性 | 理由 |
|------|----------|----------|------|
| `reqwest` | `rustls-tls` | `native-tls`, `openssl` | 安全穹顶：避免系统 OpenSSL 依赖 |
| `tokio` | `full` | — | SDK 需要完整异步运行时 |
| `serde` | `derive` | — | 必须使用 derive 宏 |
| `uuid` | `v4` | `v1`, `v3`, `v5` | UUID v4 为随机 ID 标准 |
| `chrono` | `serde` | — | 时间字段需要序列化 |

```toml
# ✅ 正确
reqwest = { version = "0.11", features = ["json", "rustls-tls", "blocking", "stream"] }

# ❌ 禁止
reqwest = { version = "0.11", features = ["json", "native-tls"] }  # 禁止 native-tls
```

### 11.5 禁止引入的依赖

| 依赖 | 理由 |
|------|------|
| `anyhow` | 与统一错误体系冲突，违反安全穹顶原则 |
| `openssl` / `native-tls` | 引入系统级 OpenSSL 依赖，违反安全穹顶原则 |
| `lazy_static` | 已弃用，使用 `once_cell::sync::Lazy` 替代 |
| `failure` | 已弃用的错误处理库 |

### 11.6 开发依赖

```toml
[dev-dependencies]
tokio-test = "0.4"
assert_cmd = "2.0"
pretty_assertions = "1.0"
futures = "0.3"
```

**强制执行**：`cargo deny check` 检查禁止依赖；`cargo audit` 检查漏洞。

---

## 12. AgentOS 模块编码示例

### 12.1 Client 模板

```rust
// AgentOS Rust SDK - XXX 客户端实现
// Version: 0.1.0
// Last updated: 2026-06-08
//
// 提供 XXX 功能的 HTTP 通信层。
// 对应 Go SDK: xxx/client.go

use reqwest::Client as ReqwestClient;
use serde_json::Value;
use std::time::Duration;

use crate::error::AgentOSError;
use crate::types::RequestOptions;

/// XXXClient 提供 XXX 领域的 HTTP 通信能力
#[derive(Debug, Clone)]
pub struct XxxClient {
    endpoint: String,
    http_client: ReqwestClient,
    api_key: Option<String>,
    timeout: Duration,
}

impl XxxClient {
    /// 创建新的 XXX 客户端
    ///
    /// # 参数
    /// - `endpoint`: 服务端点地址
    ///
    /// # 返回
    /// 返回 Result<XxxClient, AgentOSError>
    pub fn new(endpoint: &str) -> Result<Self, AgentOSError> {
        Self::builder(endpoint).build()
    }

    /// 创建客户端构建器
    pub fn builder(endpoint: &str) -> XxxClientBuilder {
        XxxClientBuilder::new(endpoint)
    }
}

/// XXX 客户端构建器
pub struct XxxClientBuilder {
    endpoint: String,
    timeout: Duration,
    api_key: Option<String>,
}

impl XxxClientBuilder {
    pub fn new(endpoint: &str) -> Self {
        Self {
            endpoint: endpoint.trim_end_matches('/').to_string(),
            timeout: Duration::from_secs(30),
            api_key: None,
        }
    }

    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }

    pub fn api_key(mut self, api_key: &str) -> Self {
        self.api_key = Some(api_key.to_string());
        self
    }

    /// 构建客户端
    ///
    /// # 错误
    /// 如果端点地址格式无效，返回错误
    pub fn build(self) -> Result<XxxClient, AgentOSError> {
        if !self.endpoint.starts_with("http://") && !self.endpoint.starts_with("https://") {
            return Err(AgentOSError::Config(
                "端点地址必须以 http:// 或 https:// 开头".to_string(),
            ));
        }

        let http_client = ReqwestClient::builder()
            .timeout(self.timeout)
            .use_rustls_tls()
            .build()
            .map_err(|e| AgentOSError::Config(format!("创建 HTTP 客户端失败: {}", e)))?;

        Ok(XxxClient {
            endpoint: self.endpoint,
            http_client,
            api_key: self.api_key,
            timeout: self.timeout,
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_xxx_client_new() {
        let client = XxxClient::new("http://localhost:18789");
        assert!(client.is_ok());
    }

    #[test]
    fn test_xxx_client_invalid_endpoint() {
        let client = XxxClient::new("invalid-endpoint");
        assert!(client.is_err());
    }
}
```

### 12.2 Manager 模板

```rust
// AgentOS Rust SDK - XXX 管理器实现
// Version: 0.1.0
// Last updated: 2026-06-08
//
// 提供 XXX 的完整生命周期管理功能。
// 对应 Go SDK: modules/xxx/manager.go

use std::collections::HashMap;
use std::sync::Arc;

use crate::client::APIClient;
use crate::error::{AgentOSError, CODE_MISSING_PARAMETER, CODE_INVALID_PARAMETER};
use crate::types::{ListOptions, APIResponse};
use crate::utils::{extract_data_map, get_string, build_url};

/// XxxManager 管理 XXX 完整生命周期
pub struct XxxManager {
    api: Arc<dyn APIClient>,
}

impl XxxManager {
    /// 创建新的 XXX 管理器实例
    ///
    /// # 参数
    /// - `api`: API 客户端
    pub fn new(api: Arc<dyn APIClient>) -> Self {
        XxxManager { api }
    }

    /// 创建新的 XXX
    ///
    /// # 参数
    /// - `name`: 名称
    ///
    /// # 返回
    /// 返回创建的对象
    pub async fn create(&self, name: &str) -> Result<(), AgentOSError> {
        if name.is_empty() {
            return Err(AgentOSError::with_code(CODE_MISSING_PARAMETER, "名称不能为空"));
        }

        let body = serde_json::json!({ "name": name });
        self.api.post("/api/v1/xxx", Some(&body), None).await?;
        Ok(())
    }

    /// 获取指定 XXX
    ///
    /// # 参数
    /// - `id`: 对象 ID
    pub async fn get(&self, id: &str) -> Result<(), AgentOSError> {
        if id.is_empty() {
            return Err(AgentOSError::with_code(CODE_MISSING_PARAMETER, "ID不能为空"));
        }

        let path = format!("/api/v1/xxx/{}", id);
        self.api.get(&path, None).await?;
        Ok(())
    }

    /// 列出 XXX，支持分页和过滤
    pub async fn list(&self, opts: Option<&ListOptions>) -> Result<(), AgentOSError> {
        let path = if let Some(options) = opts {
            build_url("/api/v1/xxx", options.to_query_params())
        } else {
            "/api/v1/xxx".to_string()
        };

        self.api.get(&path, None).await?;
        Ok(())
    }

    /// 删除指定 XXX
    pub async fn delete(&self, id: &str) -> Result<(), AgentOSError> {
        if id.is_empty() {
            return Err(AgentOSError::with_code(CODE_MISSING_PARAMETER, "ID不能为空"));
        }

        let path = format!("/api/v1/xxx/{}", id);
        self.api.delete(&path, None).await?;
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    // 使用 MockAPIClient 进行测试
}
```

### 12.3 Syscall 模板

```rust
// AgentOS Rust SDK - XXX Syscall 实现
// Version: 0.1.0
// Last updated: 2026-06-08
//
// 提供 XXX 域的系统调用绑定。
// 对应五维正交体系中的系统调用维度。

use std::collections::HashMap;
use serde_json::Value;

use crate::AgentOSError;
use crate::syscall::{SyscallBinding, SyscallRequest, SyscallResponse, SyscallNamespace};

/// XXX 域系统调用
pub struct XxxSyscall<B: SyscallBinding> {
    binding: B,
}

impl<B: SyscallBinding> XxxSyscall<B> {
    /// 创建新的 XXX 系统调用
    pub fn new(binding: B) -> Self {
        XxxSyscall { binding }
    }

    /// 执行 XXX 操作
    pub fn do_something(&self, param: &str) -> Result<SyscallResponse, AgentOSError> {
        let mut params = HashMap::new();
        params.insert("param".to_string(), Value::String(param.to_string()));
        self.binding.invoke(SyscallRequest {
            namespace: SyscallNamespace::Task,  // 替换为实际命名空间
            operation: "do_something".to_string(),
            params,
        })
    }
}
```

### 12.4 Plugin 模板

```rust
// AgentOS Rust SDK - XXX 插件实现
// Version: 0.1.0
// Last updated: 2026-06-08
//
// 提供 XXX 功能的插件扩展。
// 遵循安全穹顶 Cupolas 原则：所有权限必须声明。

use std::collections::HashMap;

use crate::plugin::{BasePlugin, PluginManifest};

/// XXX 插件实现
pub struct XxxPlugin {
    plugin_id: String,
    // 内部状态
}

impl XxxPlugin {
    /// 创建新的 XXX 插件
    pub fn new() -> Self {
        Self {
            plugin_id: String::new(),
        }
    }
}

impl BasePlugin for XxxPlugin {
    fn plugin_id(&self) -> &str {
        &self.plugin_id
    }

    fn set_plugin_id(&mut self, id: &str) {
        self.plugin_id = id.to_string();
    }

    fn on_load(&mut self, _context: &HashMap<String, serde_json::Value>) -> Result<(), String> {
        // 初始化资源
        Ok(())
    }

    fn on_activate(&mut self, _context: &HashMap<String, serde_json::Value>) -> Result<(), String> {
        // 激活功能
        Ok(())
    }

    fn on_deactivate(&mut self) -> Result<(), String> {
        // 停用功能
        Ok(())
    }

    fn on_unload(&mut self) -> Result<(), String> {
        // 释放资源
        Ok(())
    }

    fn on_error(&mut self, error: &str) {
        eprintln!("[XxxPlugin] error: {}", error);
    }

    fn get_capabilities(&self) -> Vec<String> {
        vec!["xxx".to_string()]
    }
}

/// 注册插件到全局注册表
pub fn register_xxx_plugin() {
    crate::plugin::with_plugin_registry(|registry| {
        let factory: crate::plugin::PluginFactory =
            Box::new(|| Box::new(XxxPlugin::new()));
        let manifest = PluginManifest::new("xxx_plugin", "XXX Plugin");
        let _ = registry.register(factory, Some(manifest));
    });
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::plugin::PluginRegistry;

    #[test]
    fn test_xxx_plugin_lifecycle() {
        let mut registry = PluginRegistry::new();
        let factory: crate::plugin::PluginFactory =
            Box::new(|| Box::new(XxxPlugin::new()));
        let manifest = PluginManifest::new("xxx_plugin", "XXX Plugin");

        let pid = registry.register(factory, Some(manifest)).unwrap();
        assert_eq!(pid, "xxx_plugin");

        let result = registry.load("xxx_plugin");
        assert!(result.is_ok());

        let activated = registry.activate("xxx_plugin");
        assert!(activated);

        let deactivated = registry.deactivate("xxx_plugin");
        assert!(deactivated);
    }
}
```

---

## 附录 A：Clippy 配置建议

在 `Cargo.toml` 或 `.clippy.toml` 中建议配置：

```toml
# .clippy.toml
cognitive-complexity-threshold = 30
```

在 `lib.rs` 或 `main.rs` 顶部建议添加：

```rust
#![warn(clippy::all, clippy::pedantic, clippy::nursery)]
#![allow(clippy::module_name_repetitions)]  // 允许 types::types 模式
#![allow(clippy::must_use_candidate)]       // 允许非 MustUse 返回值
```

## 附录 B：Cargo Deny 配置建议

创建 `deny.toml`：

```toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause", "ISC"]

[bans]
deny = [
    { name = "anyhow" },
    { name = "openssl" },
    { name = "native-tls" },
    { name = "lazy_static" },
    { name = "failure" },
]
```

## 附录 C：错误码速查表

| 域 | 范围 | 示例 |
|----|------|------|
| 通用 | 0x0000–0x0FFF | CODE_SUCCESS=0x0000, CODE_TIMEOUT=0x0004 |
| 核心循环 | 0x1000–0x1FFF | CODE_LOOP_CREATE_FAILED=0x1001 |
| 认知层 | 0x2000–0x2FFF | CODE_COGNITION_FAILED=0x2001 |
| 执行层 | 0x3000–0x3FFF | CODE_TASK_FAILED=0x3001 |
| 记忆层 | 0x4000–0x4FFF | CODE_MEMORY_NOT_FOUND=0x4001 |
| 系统调用 | 0x5000–0x5FFF | CODE_TELEMETRY_ERROR=0x5001 |
| 安全域 | 0x6000–0x6FFF | CODE_PERMISSION_DENIED=0x6001 |
| 动态模块 | 0x7000–0x7FFF | (预留) |

---

> **文档维护者**: Spharx AgentOS Team
> **审核周期**: 每季度审核一次，与 SDK 主版本同步更新
