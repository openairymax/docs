<!-- SPDX-FileCopyrightText: 2026 SPHARX Ltd. -->
<!-- SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 -->

# Airymax Rust 安全编码标准

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/50-specifications/30-coding-standard/14-rust-secure-coding.md
---

## 目录

1. [总则与安全架构映射](#1-总则与安全架构映射)
2. [unsafe 代码使用规范](#2-unsafe-代码使用规范)
3. [依赖安全](#3-依赖安全)
4. [密码学 API 使用规范](#4-密码学-api-使用规范)
5. [输入验证](#5-输入验证)
6. [错误信息安全](#6-错误信息安全)
7. [并发安全](#7-并发安全)
8. [内存安全](#8-内存安全)
9. [网络安全](#9-网络安全)
10. [审计与可观测性](#10-审计与可观测性)

---

## 1. 总则与安全架构映射

### 1.1 Cupolas 四层防护在 Rust SDK 中的实现映射

Airymax 的安全架构基于 **Cupolas 四层纵深防御模型**，每一层在 Rust SDK 中均有对应的代码实现与约束：

| 防护层 | Cupolas 定义 | Rust SDK 实现 | 关键代码位置 |
|--------|-------------|--------------|-------------|
| **D1: 沙箱隔离** | Process/Container/WASM | `PluginManager` 状态机 + `sandbox_enabled` 标志 | `plugin.rs` — `PluginState` 枚举管控插件生命周期 |
| **D2: 权限裁决** | RBAC + YAML 规则引擎 | `SyscallBinding` trait + 权限规则 YAML | `syscall.rs` — 所有系统调用经由 `SyscallBinding::invoke` |
| **D3: 输入净化** | regex→type→length→encoding | `validate_endpoint()` + `build_url()` URL 编码 + serde 类型约束 | `helpers.rs` — `urlencoding::encode` + `#[serde(rename)]` |
| **D4: 审计追踪** | append-only + SHA-256 哈希链 | `Telemetry` + `Meter`/`Tracer` + 请求 ID 追踪 | `telemetry.rs` — `Arc<Mutex<Meter/Tracer>>` |

### 1.2 五维正交安全视角

本标准从五个正交维度审视 Rust 安全编码：

- **系统观**：Rust SDK 作为 Airymax 生态的客户端组件，必须在不可信网络环境中运行，所有外部交互均视为敌对信道。
- **内核观**：Rust 的所有权模型与借用检查器是编译期安全保证的核心，不可通过 `unsafe` 绕过而不提供安全论证。
- **认知观**：Airymax 的认知层（Cognition Layer, 0x2xxx 错误码）处理意图解析与 DAG 构建，认知输入必须经过 D3 输入净化管道。
- **工程观**：CI/CD 管道中必须集成 `cargo audit`、`cargo deny`、`clippy` 等安全检查工具，安全左移。
- **设计美学**：安全机制应如 Rust 类型系统般"零成本抽象"——正确使用时无运行时开销，错误使用时编译期拒绝。

### 1.3 安全编码总则

1. **编译期安全优于运行期检查**：利用 Rust 类型系统将安全约束编码为类型。
2. **最小权限原则**：所有代码默认无权限，显式授权方可执行。
3. **纵深防御**：不依赖单一安全层，每层独立防护。
4. **安全失败**：失败时进入最安全状态（deny-by-default）。
5. **不可绕过**：安全检查不可通过公共 API 绕过。

---

## 2. unsafe 代码使用规范

### 2.1 核心原则

`unsafe` 代码是 Rust 安全保证的"逃生舱口"，使用时必须满足以下条件：

1. **必须有安全论证注释**：每块 `unsafe` 代码上方必须以 `// SAFETY:` 前缀注释说明为何该操作是安全的。
2. **必须有安全边界文档**：包含 `unsafe` 的模块必须在模块级文档中声明安全边界。
3. **禁止在公共 API 中暴露 unsafe**：公共 API 必须封装 `unsafe` 调用，对外提供安全接口。

### 2.2 安全论证注释规范

❌ **错误**：无安全论证

```rust
unsafe {
    let ptr = raw.as_ptr();
    *ptr.offset(4) = 42;
}
```

✅ **正确**：完整的安全论证

```rust
// SAFETY: `raw` 是从合法的 Vec<u8> 转换而来，长度为 8。
// offset(4) 在边界内 (4 < 8)。当前线程独占该内存区域，
// 不存在数据竞争。写入值 42 为合法 u8。
unsafe {
    let ptr = raw.as_ptr();
    *ptr.offset(4) = 42;
}
```

### 2.3 安全边界文档

❌ **错误**：模块无安全边界声明

```rust
/// FFI 绑定模块
pub mod ffi_bindings {
    // ... unsafe 代码 ...
}
```

✅ **正确**：模块级安全边界文档

```rust
//! # 安全边界
//!
//! 本模块包含与 C 库 `libagentrt_sandbox` 的 FFI 绑定。
//!
//! ## 不变量
//! - 所有传入指针必须非空且对齐
//! - 调用者负责确保 `SandboxHandle` 的生命周期不超过 `Sandbox`
//!
//! ## 安全封装
//! - `Sandbox::new()` 封装了 `unsafe fn agentrt_sandbox_create()`
//! - 公共 API 中无 `unsafe fn`
pub mod ffi_bindings {
    // ...
}
```

### 2.4 公共 API 封装

❌ **错误**：公共 API 暴露 unsafe

```rust
pub unsafe fn raw_syscall(request: SyscallRequest) -> Result<SyscallResponse> {
    // 直接暴露 unsafe 给调用者
}
```

✅ **正确**：封装 unsafe，公共 API 安全

```rust
/// 安全的系统调用接口
pub fn invoke_syscall(request: SyscallRequest) -> Result<SyscallResponse> {
    // SAFETY: 请求参数已通过 validate_syscall_request() 验证，
    // 指针参数由 Rust 分配器管理，生命周期受函数作用域约束。
    unsafe { raw_syscall_inner(request) }
}
```

### 2.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| `unsafe` 块缺少 `// SAFETY:` 注释 | `cargo geiger` + 自定义 lint | 每次 CI |
| 公共 `unsafe fn` | `clippy::unsafe_removed_from_name` + 代码审查 | 每次 MR |
| `unsafe` 代码覆盖率 | `cargo geiger --update-readme` | 每周报告 |

---

## 3. 依赖安全

### 3.1 cargo audit 强制执行

所有 CI 管道必须执行 `cargo audit`，发现任何已知漏洞的依赖必须立即升级或替换。

❌ **错误**：CI 中跳过安全审计

```yaml
# .github/workflows/ci.yml
- run: cargo build
# 缺少 cargo audit
```

✅ **正确**：CI 中强制安全审计

```yaml
# .github/workflows/ci.yml
- run: cargo install cargo-audit
- run: cargo audit
- run: cargo build
```

### 3.2 cargo deny 配置

Airymax Rust SDK 使用 `rustls-tls` 而非 `openssl`，此约束通过 `cargo deny` 强制执行。

❌ **错误**：允许 openssl 依赖

```toml
# Cargo.toml
reqwest = { version = "0.11", features = ["json", "default-tls"] }
```

✅ **正确**：强制使用 rustls（与 Airymax SDK 一致）

```toml
# Cargo.toml
reqwest = { version = "0.11", features = ["json", "rustls-tls", "blocking", "stream"] }
```

**deny.toml 配置**：

```toml
# deny.toml — Airymax Rust SDK 依赖安全策略
[advisories]
db-path = "~/.cargo/advisory-db"
vulnerability = "deny"
unmaintained = "warn"

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause", "ISC", "Zlib"]
unlicensed = "deny"

[bans]
# 禁止 openssl 依赖，强制使用 rustls
[[bans.deny]]
name = "openssl"
wrappers = ["openssl-sys"]
reason = "Airymax 安全策略要求使用 rustls-tls，禁止 openssl 依赖链"

[[bans.deny]]
name = "native-tls"
reason = "native-tls 在 Linux 上链接 openssl，违反 rustls-only 策略"

# 禁止重复依赖（例外列表除外）
multiple-versions = "warn"
```

### 3.3 供应链安全

❌ **错误**：忽略 Cargo.lock

```gitignore
# .gitignore
Cargo.lock
```

✅ **正确**：Cargo.lock 必须提交

```gitignore
# .gitignore — 不忽略 Cargo.lock
# Cargo.lock 必须提交到版本控制以确保可复现构建
```

**规则**：
- 库项目（library）：`Cargo.lock` 必须提交，确保 CI/CD 构建可复现。
- 二进制项目（binary）：`Cargo.lock` 必须提交，部署版本必须与审计版本一致。
- 禁止使用 `cargo update` 而不经审查地更新依赖。

### 3.4 许可证合规

❌ **错误**：未检查许可证兼容性

```bash
# 直接添加依赖而不检查许可证
cargo add some-crate
```

✅ **正确**：CI 中强制许可证检查

```yaml
# CI pipeline
- run: cargo install cargo-deny
- run: cargo deny check licenses
- run: cargo deny check bans
- run: cargo deny check advisories
```

### 3.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 已知漏洞 | `cargo audit` | 每次 CI + 每日定时 |
| 禁止依赖 | `cargo deny check bans` | 每次 CI |
| 许可证合规 | `cargo deny check licenses` | 每次 CI |
| 依赖版本锁定 | `Cargo.lock` diff 检查 | 每次 MR |

---

## 4. 密码学 API 使用规范

### 4.1 禁止自定义加密算法

❌ **错误**：自行实现加密

```rust
fn xor_encrypt(data: &[u8], key: &[u8]) -> Vec<u8> {
    data.iter().zip(key.iter().cycle()).map(|(d, k)| d ^ k).collect()
}
```

✅ **正确**：使用经过审计的密码学库

```rust
use ring::aead::{seal_in_place, open_in_place, Aad, Nonce, AES_256_GCM};

fn encrypt_data(key: &[u8; 32], nonce: &[u8; 12], data: &mut [u8], tag: &mut [u8; 16]) {
    let nonce = Nonce::assume_unique_for_key(*nonce);
    seal_in_place(&AES_256_GCM, key, nonce, Aad::empty(), data, tag)
        .expect("encryption failed");
}
```

### 4.2 推荐库

| 用途 | 推荐库 | 禁止库 |
|------|--------|--------|
| TLS | `rustls` | `openssl`, `native-tls` |
| 对称加密 | `ring` | 自实现 |
| 哈希 | `ring::digest` | 自实现 |
| HMAC | `ring::hmac` | 自实现 |
| 随机数 | `rand::rngs::OsRng` | `rand::thread_rng()` 用于安全场景 |

### 4.3 密钥管理

❌ **错误**：硬编码密钥

```rust
const API_KEY: &str = "sk-prod-abc123def456";
const DB_PASSWORD: &str = "super_secret_password";

fn create_client() -> Client {
    Client::new_with_api_key("http://localhost:8080", API_KEY).unwrap()
}
```

✅ **正确**：从安全来源获取密钥

```rust
use std::env;

fn create_client() -> Result<Client, AgentRTError> {
    let endpoint = env::var("AGENTRT_ENDPOINT")
        .unwrap_or_else(|_| "http://127.0.0.1:8080".to_string());
    let api_key = env::var("AGENTRT_API_KEY")
        .map_err(|_| AgentRTError::Config("AGENTRT_API_KEY 环境变量未设置".to_string()))?;
    Client::new_with_api_key(&endpoint, &api_key)
}
```

密钥管理策略（与 `policy.yaml` 中 `secrets` 配置一致）：

- **开发环境**：环境变量 (`provider: "env"`)
- **生产环境**：HashiCorp Vault / AWS KMS / Azure KeyVault
- **密钥轮换**：90 天（`key_rotation_days: 90`）
- **静态加密**：AES-256-GCM（`encryption: "aes-256-gcm"`）

### 4.4 随机数生成

❌ **错误**：安全场景使用非密码学安全随机数

```rust
// 当前 SDK client.rs 中的 generate_request_id() 使用 thread_rng()
// 这对于请求 ID 生成是可接受的，但不可用于安全令牌
fn generate_token() -> String {
    let random: u32 = rand::thread_rng().gen_range(0..999999);
    format!("token-{:06}", random)
}
```

✅ **正确**：安全场景使用密码学安全随机数

```rust
use rand::rngs::OsRng;
use rand::RngCore;

/// 生成密码学安全的令牌
fn generate_secure_token() -> String {
    let mut buf = [0u8; 32];
    OsRng.fill_bytes(&mut buf);
    hex::encode(buf)
}

/// 生成 UUID v4（内部使用密码学安全随机数）
fn generate_id() -> String {
    Uuid::new_v4().to_string()  // uuid crate 使用 OsRng
}
```

**注意**：当前 SDK 中 `generate_request_id()` 使用 `rand::thread_rng()` 生成请求 ID，这对于非安全标识符是可接受的。但任何涉及认证、授权、令牌生成的场景必须使用 `OsRng`。

### 4.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 硬编码密钥 | `gitleaks` + 自定义正则扫描 | 每次 CI |
| 自定义加密 | 代码审查 | 每次 MR |
| openssl 依赖 | `cargo deny check bans` | 每次 CI |
| 密钥泄露 | `gitleaks --pre-commit` | 每次提交 |

---

## 5. 输入验证

### 5.1 核心原则

所有来自外部的输入必须经过验证，遵循 Cupolas D3 输入净化管道：**regex → type → length → encoding**。

### 5.2 端点验证

❌ **错误**：不验证端点格式

```rust
fn connect(endpoint: &str) -> Client {
    Client::new(endpoint).unwrap() // 可能接受任意字符串
}
```

✅ **正确**：验证端点格式（与 SDK `validate_endpoint` 一致）

```rust
pub fn validate_endpoint(endpoint: &str) -> bool {
    endpoint.starts_with("http://") || endpoint.starts_with("https://")
}

// ClientBuilder::build() 中的验证
pub fn build(self) -> Result<Client, AgentRTError> {
    if !self.endpoint.starts_with("http://") && !self.endpoint.starts_with("https://") {
        return Err(AgentRTError::Config(
            "端点地址必须以 http:// 或 https:// 开头".to_string()
        ));
    }
    // ...
}
```

### 5.3 serde 反序列化安全

❌ **错误**：使用 `deserialize_any` 绕过类型检查

```rust
#[derive(Deserialize)]
struct UserInput {
    #[serde(deserialize_with = "deserialize_any")]
    data: Value,  // 接受任意类型，丧失类型安全
}
```

✅ **正确**：使用强类型反序列化（与 SDK 一致）

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SyscallRequest {
    pub namespace: SyscallNamespace,  // 枚举类型，仅接受已知值
    pub operation: String,
    pub params: HashMap<String, Value>,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum SyscallNamespace {
    #[serde(rename = "task")]
    Task,
    #[serde(rename = "memory")]
    Memory,
    // ... 仅允许已知命名空间
}
```

❌ **错误**：反序列化无大小限制

```rust
let data: serde_json::Value = serde_json::from_reader(reader)?;
// 攻击者可发送数 GB 的 JSON 导致 OOM
```

✅ **正确**：限制反序列化输入大小

```rust
const MAX_JSON_PAYLOAD: usize = 10 * 1024 * 1024; // 10MB，与 MAX_RESPONSE_BODY_SIZE 一致

let mut buffer = Vec::with_capacity(MAX_JSON_PAYLOAD);
reader.take(MAX_JSON_PAYLOAD as u64).read_to_end(&mut buffer)?;
let data: serde_json::Value = serde_json::from_slice(&buffer)?;
```

### 5.4 字符串处理安全

❌ **错误**：直接拼接用户输入到 URL

```rust
let url = format!("{}{}", self.endpoint, user_path);
// user_path = "../../../etc/passwd" → 路径遍历
```

✅ **正确**：URL 编码用户输入（与 SDK `build_url` 一致）

```rust
pub fn build_url(path: &str, params: HashMap<String, String>) -> String {
    if params.is_empty() {
        return path.to_string();
    }
    let query_string: Vec<String> = params
        .iter()
        .map(|(k, v)| format!("{}={}", urlencoding::encode(k), urlencoding::encode(v)))
        .collect();
    format!("{}?{}", path, query_string.join("&"))
}
```

### 5.5 整数溢出检查

❌ **错误**：未检查整数溢出

```rust
fn calculate_total(count: i32, unit_size: i32) -> i32 {
    count * unit_size  // 可能溢出
}
```

✅ **正确**：使用 checked/saturating 运算

```rust
fn calculate_total(count: i32, unit_size: i32) -> Option<i32> {
    count.checked_mul(unit_size)
}

// 或使用 saturating 运算（适用于不应失败的场景）
fn increment_counter(current: u64, delta: u64) -> u64 {
    current.saturating_add(delta)
}
```

### 5.6 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 未验证的外部输入 | `clippy::unwrap_used` + 代码审查 | 每次 MR |
| `deserialize_any` 使用 | 代码审查 + 自定义 lint | 每次 MR |
| 整数溢出 | `clippy::integer_arithmetic` | 每次 CI |
| 路径遍历 | `clippy::path_ends_with` + 审查 | 每次 MR |

---

## 6. 错误信息安全

### 6.1 核心原则

错误消息不得泄露内部实现细节（堆栈跟踪、文件路径、数据库结构、密钥片段等）。Airymax 错误体系基于 `AgentRTError` 枚举，采用错误码 + 通用消息的模式。

### 6.2 AgentRTError 的安全格式化

SDK 中的 `AgentRTError` 采用 `[错误码] 消息` 格式，错误码使用十六进制分类体系：

```
0x0xxx  通用错误
0x1xxx  核心循环错误
0x2xxx  认知层错误
0x3xxx  执行层错误
0x4xxx  记忆层错误
0x5xxx  系统调用错误
0x6xxx  安全域错误
```

❌ **错误**：在错误消息中泄露内部信息

```rust
return Err(AgentRTError::http(
    &format!("Database connection failed: postgres://admin:password@db.internal:5432/agentrt - {}", err)
));
```

✅ **正确**：通用错误消息，详细信息仅写入日志

```rust
log::error!("Database connection failed: {}", err); // 内部日志
return Err(AgentRTError::http("服务暂时不可用")); // 对外消息
```

### 6.3 From 实现中的信息安全

当前 SDK 的 `From<reqwest::Error>` 实现直接将底层错误消息传递到 `AgentRTError`：

```rust
impl From<reqwest::Error> for AgentRTError {
    fn from(err: reqwest::Error) -> Self {
        if err.is_timeout() {
            AgentRTError::timeout(&err.to_string())  // 可能泄露内部 URL
        }
        // ...
    }
}
```

**改进建议**：在生产环境中过滤敏感信息：

❌ **错误**：直接暴露底层错误

```rust
AgentRTError::network(&err.to_string())
// err.to_string() 可能包含 "http://admin:secret@internal-host:8080/..."
```

✅ **正确**：过滤敏感信息

```rust
impl From<reqwest::Error> for AgentRTError {
    fn from(err: reqwest::Error) -> Self {
        if err.is_timeout() {
            AgentRTError::timeout("请求超时")  // 不暴露内部 URL
        } else if err.is_connect() {
            AgentRTError::connection_refused("连接被拒绝")  // 不暴露主机信息
        } else if err.is_request() {
            AgentRTError::network("网络错误")  // 通用消息
        } else {
            AgentRTError::Other("内部错误")  // 最小信息
        }
    }
}
```

### 6.4 日志脱敏

❌ **错误**：日志中记录敏感信息

```rust
log::info!("API key used: {}", api_key);
log::debug!("Request headers: {:?}", headers); // 可能包含 Authorization
```

✅ **正确**：脱敏后记录

```rust
fn mask_api_key(key: &str) -> String {
    if key.len() > 8 {
        format!("{}****{}", &key[..4], &key[key.len()-4..])
    } else {
        "****".to_string()
    }
}

log::info!("API key used: {}", mask_api_key(&api_key));
log::debug!("Request ID: {}", request_id); // 仅记录非敏感标识
```

### 6.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 错误消息泄露 | 代码审查 + `grep -r "password\|secret\|token" src/` | 每次 MR |
| 日志敏感信息 | `gitleaks` + 自定义正则 | 每次 CI |
| unwrap 暴露堆栈 | `clippy::unwrap_used` | 每次 CI |

---

## 7. 并发安全

### 7.1 Arc\<Mutex\<T\>\> 使用规范

Airymax SDK 使用 `Arc<Mutex<T>>` 实现线程安全的共享状态（如 `Telemetry`、`PluginRegistry`、`ProtocolStats`）。

❌ **错误**：长时间持有锁

```rust
let mut registry = get_plugin_registry();
let plugin = registry.load("my-plugin")?;
// 持有锁期间执行耗时操作
tokio::time::sleep(Duration::from_secs(5)).await;
let result = plugin.execute();
```

✅ **正确**：最小化锁持有时间（与 SDK `with_plugin_registry` 模式一致）

```rust
// SDK 中的安全模式：通过闭包限制锁的作用域
pub fn with_plugin_registry<F, R>(f: F) -> R
where
    F: FnOnce(&mut PluginRegistry) -> R,
{
    let mut guard = GLOBAL_REGISTRY.lock().expect("plugin registry mutex poisoned");
    f(&mut guard)
}

// 使用示例：快速操作后释放锁
let plugin_id = with_plugin_registry(|reg| {
    reg.register(factory, manifest)
});
```

### 7.2 禁止在持锁时调用外部代码

❌ **错误**：持锁时进行网络调用

```rust
let mut stats = self.stats.lock().unwrap();
stats.requests_sent += 1;
// 持锁时进行 HTTP 请求 — 可能导致死锁或长时间阻塞
let response = self.http_client.get(&url).send().await;
```

✅ **正确**：先获取数据，释放锁后再调用外部代码

```rust
// 先在锁内完成状态更新
{
    let mut stats = self.stats.lock().unwrap();
    stats.requests_sent += 1;
} // 锁在此处释放

// 释放锁后再进行网络调用
let response = self.http_client.get(&url).send().await?;
```

### 7.3 Send + Sync 约束

Airymax SDK 的所有公共接口均要求 `Send + Sync`：

```rust
// APIClient trait — 所有异步客户端必须可跨线程传递
#[async_trait::async_trait]
pub trait APIClient: Send + Sync {
    async fn get(&self, path: &str, opts: Option<Vec<RequestOption>>) -> Result<APIResponse, AgentRTError>;
    // ...
}

// BasePlugin trait — 插件必须可跨线程使用
pub trait BasePlugin: Send + Sync {
    fn plugin_id(&self) -> &str;
    // ...
}

// PluginFactory — 工厂函数必须可跨线程调用
pub type PluginFactory = Box<dyn Fn() -> Box<dyn BasePlugin> + Send + Sync>;
```

❌ **错误**：公共类型缺少 Send + Sync

```rust
pub trait MyHandler {
    fn handle(&self, request: Request) -> Response;
    // 缺少 Send + Sync，无法在异步运行时中使用
}
```

✅ **正确**：公共 trait 添加 Send + Sync

```rust
pub trait MyHandler: Send + Sync {
    fn handle(&self, request: Request) -> Response;
}
```

### 7.4 死锁预防

Airymax SDK 中的锁层级规则：

1. **全局锁**（`GLOBAL_REGISTRY`）> **模块锁**（`Telemetry.meter`/`Telemetry.tracer`）> **实例锁**（`ProtocolStats`）
2. **禁止反向获取**：持有低层级锁时不得请求高层级锁。
3. **禁止同时持有多个同层级锁**。

❌ **错误**：同时获取两个同层级锁

```rust
let meter = self.meter.lock().unwrap();
let tracer = self.tracer.lock().unwrap();
// 如果另一个线程以相反顺序获取，将导致死锁
```

✅ **正确**：依次获取，立即释放

```rust
{
    let mut meter = self.meter.lock().unwrap();
    meter.record("request_count", 1.0, None);
} // meter 锁释放

{
    let mut tracer = self.tracer.lock().unwrap();
    let span = tracer.start_span("operation");
} // tracer 锁释放
```

### 7.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 缺少 Send + Sync | 编译器 + `clippy` | 每次编译 |
| 持锁时间过长 | 代码审查 + `parking_lot` 超时检测 | 每次 MR |
| 死锁风险 | 代码审查 + `cargo lockbud` | 每次 MR |
| Mutex poison 处理 | `clippy::unwrap_used` | 每次 CI |

---

## 8. 内存安全

### 8.1 Rust 所有权模型与 Airymax 所有权语义的映射

Airymax 的领域模型与 Rust 所有权系统天然契合：

| Airymax 概念 | Rust 所有权映射 | 安全保证 |
|-------------|----------------|---------|
| `Task` 的唯一归属 | `Task` 按值传递，单一所有者 | 不可被多个 Manager 同时修改 |
| `Session` 的共享上下文 | `Arc<Mutex<HashMap<String, Value>>>` | 并发安全的状态共享 |
| `Plugin` 的生命周期 | `PluginState` 状态机 + `Drop` trait | 资源确定性释放 |
| `Client` 的连接池 | `ReqwestClient` 内部 `Arc` 引用计数 | 连接复用且线程安全 |

### 8.2 禁止 mem::forget 在生产代码中

❌ **错误**：使用 `mem::forget` 绕过 Drop

```rust
use std::mem::forget;

fn leak_connection(client: Client) {
    forget(client); // 连接永远不会被关闭，资源泄漏
}
```

✅ **正确**：依赖 Rust 的确定性析构

```rust
fn use_connection(client: Client) {
    // client 在作用域结束时自动 Drop
    let _ = client.health().await;
} // client.Drop() 自动关闭连接
```

### 8.3 禁止手动内存管理

❌ **错误**：手动 alloc/dealloc

```rust
use std::alloc::{alloc, dealloc, Layout};

let layout = Layout::from_size_align(1024, 8).unwrap();
let ptr = unsafe { alloc(layout) };
// ... 使用后忘记 dealloc 或 double free
unsafe { dealloc(ptr, layout); }
```

✅ **正确**：使用 Rust 标准容器和智能指针

```rust
let buffer = Vec::with_capacity(1024);  // 自动管理内存
let shared = Arc::new(Mutex::new(state));  // 引用计数自动释放
```

### 8.4 FFI 边界安全

当与 C 库交互时（如 `libagentrt_sandbox`），必须遵守以下规则：

❌ **错误**：未验证 FFI 返回值

```rust
extern "C" {
    fn agentrt_sandbox_create() -> *mut SandboxHandle;
}

let handle = unsafe { agentrt_sandbox_create() };
// handle 可能为空指针，直接使用将导致 UB
```

✅ **正确**：验证 FFI 返回值并提供安全封装

```rust
extern "C" {
    fn agentrt_sandbox_create() -> *mut SandboxHandle;
    fn agentrt_sandbox_destroy(handle: *mut SandboxHandle);
}

pub struct Sandbox {
    handle: NonNull<SandboxHandle>,
}

impl Sandbox {
    pub fn new() -> Result<Self, AgentRTError> {
        // SAFETY: agentrt_sandbox_create 是线程安全的 C 函数，
        // 返回值可能为 NULL（创建失败），必须检查。
        let handle = unsafe { agentrt_sandbox_create() };
        NonNull::new(handle)
            .map(|h| Sandbox { handle: h })
            .ok_or_else(|| AgentRTError::with_code(CODE_PERMISSION_DENIED, "沙箱创建失败"))
    }
}

impl Drop for Sandbox {
    fn drop(&mut self) {
        // SAFETY: handle 由 new() 验证为非空，且 Drop 仅执行一次。
        unsafe { agentrt_sandbox_destroy(self.handle.as_ptr()) }
    }
}
```

### 8.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| `mem::forget` 使用 | `grep -r "mem::forget" src/` | 每次 MR |
| 手动 alloc/dealloc | `grep -r "std::alloc" src/` | 每次 MR |
| FFI 空指针检查 | 代码审查 + `clippy::not_unsafe_ptr_arg_deref` | 每次 MR |
| 未实现 Drop 的资源类型 | 代码审查 | 每次 MR |

---

## 9. 网络安全

### 9.1 TLS 配置（rustls-tls 强制）

Airymax SDK 的 HTTP 客户端强制使用 `rustls-tls`，禁止 `openssl`/`native-tls`。

❌ **错误**：使用默认 TLS（可能链接 openssl）

```rust
let client = reqwest::Client::new(); // 默认可能使用 native-tls
```

✅ **正确**：显式指定 rustls-tls（与 SDK Cargo.toml 一致）

```rust
// Cargo.toml
// reqwest = { version = "0.11", features = ["json", "rustls-tls", "blocking", "stream"] }

let client = ReqwestClient::builder()
    .use_rustls_tls()  // 显式使用 rustls
    .timeout(Duration::from_secs(30))
    .build()?;
```

### 9.2 请求超时和重试安全

SDK 的 `Client` 实现了带指数退避和抖动的重试机制：

❌ **错误**：无超时、无限重试

```rust
let response = client.get(url).send().await?; // 无超时
loop {
    let response = client.get(url).send().await?; // 无限重试
}
```

✅ **正确**：有限超时 + 指数退避 + 抖动（与 SDK 一致）

```rust
// SDK 中的实现
pub struct Client {
    timeout: Duration,        // 默认 30s
    max_retries: u32,         // 默认 3 次
    retry_delay: Duration,    // 默认 1s
}

fn calculate_backoff(&self, attempt: u32) -> Duration {
    let base_ms = self.retry_delay.as_millis() as f64;
    let backoff_ms = base_ms * 2_f64.powi(attempt as i32 - 1);
    // 添加随机抖动（0-50%），防止惊群效应
    let jitter = rand::thread_rng().gen_range(0.0..0.5) * backoff_ms;
    Duration::from_millis((backoff_ms + jitter) as u64)
}

fn should_retry(&self, status: StatusCode) -> bool {
    status.is_server_error() || status == StatusCode::TOO_MANY_REQUESTS
    // 注意：4xx 客户端错误不重试
}
```

### 9.3 响应体大小限制

SDK 定义了 10MB 的响应体大小限制：

```rust
/// 响应体最大允许大小（10MB）
const MAX_RESPONSE_BODY_SIZE: usize = 10 * 1024 * 1024;
```

❌ **错误**：无大小限制地读取响应

```rust
let body = response.text().await?; // 攻击者可发送无限大的响应
```

✅ **正确**：限制响应体大小

```rust
let response_text = response.text().await
    .map_err(|e| AgentRTError::parse_error(&format!("读取响应失败: {}", e)))?;

if response_text.len() > MAX_RESPONSE_BODY_SIZE {
    return Err(AgentRTError::invalid_response("响应体超过大小限制"));
}
```

### 9.4 URL 构建安全（防注入）

❌ **错误**：直接拼接用户输入到 URL

```rust
let url = format!("{}/api/v1/tasks/{}", self.endpoint, task_id);
// task_id = "../../config" → 路径遍历
```

✅ **正确**：验证和编码 URL 组件

```rust
pub fn build_url(path: &str, params: HashMap<String, String>) -> String {
    if params.is_empty() {
        return path.to_string();
    }
    let query_string: Vec<String> = params
        .iter()
        .map(|(k, v)| format!("{}={}", urlencoding::encode(k), urlencoding::encode(v)))
        .collect();
    format!("{}?{}", path, query_string.join("&"))
}

// 路径参数应验证格式
fn task_url(endpoint: &str, task_id: &str) -> Result<String, AgentRTError> {
    if !task_id.starts_with("task-") || task_id.contains('/') || task_id.contains("..") {
        return Err(AgentRTError::invalid_parameter("无效的任务 ID 格式"));
    }
    Ok(format!("{}/api/v1/tasks/{}", endpoint, task_id))
}
```

### 9.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 非 rustls TLS | `cargo deny check bans` | 每次 CI |
| 缺少超时 | `clippy::slow_vector_initialization` + 代码审查 | 每次 MR |
| 响应体无大小限制 | 代码审查 | 每次 MR |
| URL 注入 | 代码审查 + 模糊测试 | 每次 MR + 每月 |

---

## 10. 审计与可观测性

### 10.1 安全事件日志

Airymax 的安全事件日志必须包含以下字段（与 `policy.yaml` 中 `audit` 配置一致）：

```rust
struct SecurityEvent {
    timestamp: String,        // ISO 8601
    event_type: String,       // "permission_check", "auth_failure", "sandbox_violation"
    agent_id: String,         // 执行操作的 Agent
    action: String,           // 请求的操作
    resource: String,         // 目标资源
    effect: String,           // "allow", "deny", "allow_with_sandbox"
    audit_level: String,      // "low", "medium", "high", "critical"
    request_id: String,       // 关联请求 ID
}
```

❌ **错误**：安全事件无结构化日志

```rust
println!("Permission denied for agent {}", agent_id);
```

✅ **正确**：结构化安全事件日志

```rust
log::warn!(
    target: "agentrt::security",
    "permission_denied|agent={}|action={}|resource={}|level={}|request_id={}",
    agent_id, action, resource, audit_level, request_id
);
```

### 10.2 审计追踪

SDK 的 `Telemetry` 模块提供了审计追踪的基础设施：

```rust
// Telemetry 使用 Arc<Mutex<T>> 确保线程安全
pub struct Telemetry {
    service_name: String,
    meter: Arc<Mutex<Meter>>,   // 指标收集
    tracer: Arc<Mutex<Tracer>>, // 追踪收集
}
```

**审计追踪要求**（与 Cupolas D4 一致）：

1. **Append-Only**：审计记录只能追加，不可修改或删除。
2. **SHA-256 哈希链**：每条审计记录包含前一条记录的哈希值。
3. **完整性验证**：定期验证哈希链完整性。

❌ **错误**：可变审计日志

```rust
let mut audit_log: Vec<SecurityEvent> = Vec::new();
audit_log[0] = modified_event; // 审计记录被篡改
```

✅ **正确**：Append-Only 审计日志

```rust
use ring::digest::{digest, SHA256};

struct AuditEntry {
    sequence: u64,
    event: SecurityEvent,
    prev_hash: [u8; 32],
    current_hash: [u8; 32],
}

struct AuditLog {
    entries: Vec<AuditEntry>,  // 仅追加，不提供修改接口
}

impl AuditLog {
    fn append(&mut self, event: SecurityEvent) {
        let prev_hash = self.entries.last()
            .map(|e| e.current_hash)
            .unwrap_or([0u8; 32]);

        let sequence = self.entries.len() as u64 + 1;
        let payload = format!("{}:{:?}:{:?}",
            sequence, event, prev_hash);

        let current_hash = digest(&SHA256, payload.as_bytes());
        let mut hash = [0u8; 32];
        hash.copy_from_slice(current_hash.as_ref());

        self.entries.push(AuditEntry {
            sequence,
            event,
            prev_hash,
            current_hash: hash,
        });
    }
}
```

### 10.3 指标收集

SDK 的 `Meter` 和 `Tracer` 提供了安全相关的指标收集能力：

```rust
// 安全指标示例
meter.record("permission.denied", 1.0, Some({
    let mut tags = HashMap::new();
    tags.insert("agent".to_string(), agent_id);
    tags.insert("action".to_string(), action);
    tags
}));

meter.record("sandbox.violation", 1.0, Some({
    let mut tags = HashMap::new();
    tags.insert("plugin".to_string(), plugin_id);
    tags
}));
```

**关键安全指标**（与 `policy.yaml` 中 `alerts` 配置对应）：

| 指标名 | 含义 | 告警阈值 |
|--------|------|---------|
| `permission.denied` | 权限拒绝次数 | > 10 次/分钟 → warning |
| `suspicious_pattern_detected` | 可疑模式检测 | 任何触发 → critical |
| `brute_force_attempt` | 暴力破解尝试 | 任何触发 → critical → quarantine |
| `sandbox.violation` | 沙箱违规 | 任何触发 → critical |
| `auth.failure` | 认证失败 | > 5 次/分钟 → warning |

### 10.4 请求追踪

SDK 为每个 HTTP 请求生成唯一 ID 并通过 `X-Request-ID` 头传递：

```rust
fn generate_request_id() -> String {
    let timestamp = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map(|d| d.as_micros())
        .unwrap_or(0);
    let random: u32 = rand::thread_rng().gen_range(0..999999);
    format!("req-{}-{:06}", timestamp, random)
}

builder = builder.header("X-Request-ID", &request_id);
```

此请求 ID 贯穿整个调用链，用于关联日志、指标和审计记录。

### 10.5 执行机制

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 审计日志完整性 | 哈希链验证脚本 | 每日 |
| 安全指标覆盖率 | Prometheus + Grafana 看板 | 实时 |
| 日志格式合规 | 结构化日志 lint | 每次 CI |
| 告警规则有效性 | 混沌工程测试 | 每月 |

---

## 附录 A：快速参考检查清单

### 代码审查安全检查清单

- [ ] 所有 `unsafe` 块是否有 `// SAFETY:` 注释？
- [ ] 公共 API 是否暴露了 `unsafe` 函数？
- [ ] 依赖是否通过 `cargo audit` 和 `cargo deny`？
- [ ] 是否存在硬编码密钥或凭证？
- [ ] 外部输入是否经过验证（类型、长度、编码）？
- [ ] 错误消息是否泄露内部信息？
- [ ] `Arc<Mutex<T>>` 是否在持锁时调用了外部代码？
- [ ] 公共 trait 是否添加了 `Send + Sync`？
- [ ] 是否使用了 `mem::forget` 或手动内存管理？
- [ ] TLS 是否使用 `rustls-tls`？
- [ ] HTTP 请求是否设置了超时？
- [ ] 响应体是否有大小限制？
- [ ] 安全事件是否记录到审计日志？

### CI 安全管道

```yaml
# 推荐的 CI 安全检查步骤
security-check:
  - cargo audit
  - cargo deny check bans
  - cargo deny check licenses
  - cargo deny check advisories
  - cargo clippy -- -W clippy::unwrap_used -W clippy::integer_arithmetic
  - cargo geiger
  - gitleaks detect
```

---

## 附录 B：Airymax 错误码安全映射

| 错误码 | 名称 | 安全关联 |
|--------|------|---------|
| `0x000D` | `CODE_UNAUTHORIZED` | 认证失败 — 触发审计 |
| `0x000E` | `CODE_FORBIDDEN` | 授权失败 — 触发审计 |
| `0x000F` | `CODE_RATE_LIMITED` | 速率限制 — 可能指示攻击 |
| `0x6001` | `CODE_PERMISSION_DENIED` | 权限拒绝 — D2 层事件 |
| `0x6002` | `CODE_CORRUPTED_DATA` | 数据损坏 — D4 层完整性 |

---

*本文档由 Cupolas 安全团队维护。如有疑问，请联系 `cupolas-team@example.com`。*

*SPDX-FileCopyrightText: 2026 SPHARX Ltd. SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0*
