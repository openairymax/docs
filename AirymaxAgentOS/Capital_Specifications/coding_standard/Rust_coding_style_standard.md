Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）Rust 语言编码风格规范

> **文档定位**: agentrt-liunx（AirymaxOS）内核模块 Rust 语言编码风格规范
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-07
> **父文档**: [编码规范总览](README.md)
> **同源参考**: Rust for Linux 社区编码约定 + `rust/kernel/` 目录代码风格
> **理论根基**: Airymax 五维正交 24 原则 + Rust 所有权模型

---

## 1. Rust for Linux 编码约定

### 1.1 定位与适用范围

agentrt-liunx（AirymaxOS）的 Rust 代码仅用于**内核模块**（安全敏感且非热路径的子系统），典型场景包括：
- 安全策略裁决（LSM hook 实现）
- 文件系统解析（路径遍历的字符串处理）
- 配置解析器（Kconfig 派生配置验证）
- 加密算法实现（SM2/SM3/SM4 国密算法）

> **五维正交映射**：C-1 双系统协同——C 用于热路径（调度/内存/IPC），Rust 用于慢路径（安全/配置/解析）；E-1 安全内生——Rust 的内存安全保证消除整类 CVE。

### 1.2 与 Rust for Linux 社区的关系

agentrt-liunx（AirymaxOS）的 Rust 编码风格以 Rust for Linux 社区约定为基线，在此基础上增加：
- `agentrt_` / `airymaxos_` 前缀隔离
- IRON-9 v2 三层模型代码归属标注
- 内核模块专属的 unsafe 审计规范
- 与 C 代码互操作的 FFI 边界规范

### 1.3 工具链要求

- `rustfmt`：格式化，配置对齐 `rust/kernel/.rustfmt.toml`
- `clippy`：lint 检查，禁止 `#[allow(clippy::all)]`
- `rustdoc`：文档生成，所有公共 API 强制 rustdoc
- `Miri`：unsafe 代码 UB 检测，CI 阻断

---

## 2. 所有权与借用规范

### 2.1 所有权模型是 Rust 的核心优势（OS-STD-030）

> **OS-STD-030**：在内核模块 Rust 代码中，必须充分利用 Rust 的所有权模型来保证资源确定性。每个资源（内存、文件句柄、锁）必须有唯一的所有者；所有权的转移通过 `move` 语义表达；共享访问通过不可变引用（`&T`）或智能指针（`Arc<T>`、`Rc<T>`）表达。

```rust
// 好：所有权清晰——Channel 拥有其 msg_queue
pub struct AgentrtChannel {
    name: CString,
    msg_queue: VecDeque<AgentrtIpcMsg>,
    lock: Mutex<()>,
}

// 好：move 语义——创建函数转移所有权给调用者
pub fn agentrt_channel_create(name: &str) -> Result<Arc<AgentrtChannel>> {
    let chan = Arc::new(AgentrtChannel {
        name: CString::new(name)?,
        msg_queue: VecDeque::new(),
        lock: Mutex::new(()),
    });
    // chan 的所有权被 Arc 包装后返回给调用者
    Ok(chan)
}
```

### 2.2 借用而非克隆（OS-STD-031）

> **OS-STD-031**：优先使用借用（`&T`、`&mut T`）而非克隆（`.clone()`）。内核模块对性能和内存敏感，不必要的克隆会增加内存分配和释放开销。仅在需要独立所有权时克隆。

```rust
// 好：借用——零拷贝
fn validate_msg(msg: &AgentrtIpcMsg) -> bool {
    msg.len <= AGENTRT_IPC_MSG_BODY_MAX && !msg.body.is_null()
}

// 坏：不必要的克隆
fn validate_msg(msg: AgentrtIpcMsg) -> bool {
    msg.len <= AGENTRT_IPC_MSG_BODY_MAX
}
```

### 2.3 生命周期标注（OS-STD-032）

> **OS-STD-032**：生命周期参数必须显式标注，除非编译器能正确推断。长生命周期函数应使用有意义的名字（如 `'msg`、`'chan`）而非 `'a`、`'b`，以提升可读性。

```rust
// 好：有意义的生命周期名
fn agentrt_msg_validate<'msg>(
    msg: &'msg AgentrtIpcMsg,
    chan: &AgentrtChannel,
) -> Result<&'msg [u8]> {
    // ...
}

// 可接受：简单场景可用 'a
fn first<'a>(x: &'a [u8], y: &[u8]) -> &'a [u8] {
    x
}
```

---

## 3. 命名约定

### 3.1 snake_case 命名（OS-STD-033）

> **OS-STD-033**：Rust 代码遵循 Rust 社区标准命名约定：
> - 函数名、变量名、方法名：`snake_case`
> - 类型名（struct、enum、trait）、枚举变体：`PascalCase`
> - 常量、静态变量：`SCREAMING_SNAKE_CASE`
> - 宏名：`snake_case!`（声明宏）或 `PascalCase!`（过程宏）

```rust
// 函数名：snake_case
pub fn agentrt_ipc_send(channel: u32, msg: &[u8]) -> Result<()> { ... }

// 类型名：PascalCase
pub struct AgentrtIpcChannel { ... }
pub enum AgentrtTaskState { ... }

// 常量：SCREAMING_SNAKE_CASE
const AGENTRT_MAX_TASKS: usize = 1024;
const AGENTRT_IPC_MSG_HDR_SIZE: usize = 128;
```

### 3.2 agentrt_ / airymaxos_ 前缀隔离（OS-STD-034）

> **OS-STD-034**：Rust 代码同样遵循 `agentrt_` / `airymaxos_` 前缀隔离规范：
> - `agentrt_*` 前缀：同源 API（[SS] 语义同源层），与 agentrt 用户态 API 签名一致
> - `airymaxos_*` 前缀：agentrt-liunx（AirymaxOS）专属 API（[IND] 完全独立层）

```rust
/// [SS] 语义同源层：与 agentrt 用户态 agentrt_ipc_send() 签名一致
pub fn agentrt_ipc_send(channel: u32, msg: &[u8]) -> Result<()> { ... }

/// [IND] 完全独立层：agentrt-liunx（AirymaxOS）内核专属
pub fn airymaxos_lsm_hook_register(hooks: &SecurityHookList) -> Result<()> { ... }
```

### 3.3 模块命名（OS-STD-035）

> **OS-STD-035**：模块名使用 `snake_case`，文件名与模块名一致。模块树结构应与 C 代码的子系统结构对应。

```
rust/kernel/
├── agentrt/
│   ├── ipc.rs           // pub mod ipc;
│   ├── task.rs          // pub mod task;
│   └── capability.rs    // pub mod capability;
├── airymaxos/
│   ├── security.rs      // pub mod security;
│   ├── sched.rs         // pub mod sched;
│   └── memory.rs        // pub mod memory;
└── lib.rs
```

---

## 4. unsafe 代码块规范

### 4.1 最小化原则（OS-SEC-001）

> **OS-SEC-001**：unsafe 代码块必须最小化——仅包裹真正需要 unsafe 的操作，不包裹任何 safe 代码。每个 unsafe 块必须在其上方用注释说明"为什么此处 safe 代码无法完成任务"。

```rust
// 好：unsafe 块最小化，有注释说明
fn read_mmio_register(base: *const u8, offset: usize) -> u32 {
    // SAFETY: base 是 ioremap 返回的合法 MMIO 地址，offset 已验证在范围内
    unsafe {
        readl(base.add(offset) as *const u32)
    }
}

// 坏：unsafe 块过大，包裹了 safe 代码
unsafe {
    let base = ioremap(phys_addr, PAGE_SIZE)?;
    let val = readl(base.add(AGENTRT_REG_CTRL));
    if val & AGENTRT_CTRL_ENABLE != 0 {
        pr_info!("Device enabled\n");
    }
    writel(val | AGENTRT_CTRL_RESET, base.add(AGENTRT_REG_CTRL));
}
```

### 4.2 文档化原则（OS-SEC-002）

> **OS-SEC-002**：每个 unsafe 块必须包含 `// SAFETY:` 注释，解释：
> 1. 为什么此处需要 unsafe（具体的安全条件是什么）
> 2. 调用者如何保证这些安全条件被满足
> 3. 如果不能保证，会发生什么后果

```rust
/// 从共享内存缓冲区读取任务描述符。
///
/// # Safety
///
/// 调用者必须确保：
/// - `ptr` 指向有效的 `AgentrtTaskDesc` 结构体
/// - `ptr` 在读取期间不会被并发修改
/// - `ptr` 的生命周期覆盖此函数的执行
pub unsafe fn agentrt_task_desc_read(ptr: *const AgentrtTaskDesc) -> AgentrtTaskDesc {
    // SAFETY: 调用者已保证 ptr 有效且不会被并发修改
    unsafe { ptr::read_volatile(ptr) }
}
```

### 4.3 审查原则（OS-SEC-003）

> **OS-SEC-003**：所有 unsafe 代码必须经过至少两名资深维护者审查，审查记录必须保存在 PR 中。任何新增 unsafe 代码必须附带对应的测试用例，证明安全条件被满足。

---

## 5. 错误处理

### 5.1 Result<T, E> 优于 panic（OS-STD-036）

> **OS-STD-036**：内核模块 Rust 代码禁止使用 `panic!()` / `unwrap()` / `expect()`，必须使用 `Result<T, E>` 返回错误。内核 panic 等同于系统崩溃，这在 agentrt-liunx（AirymaxOS）中不可接受。

```rust
// 好：返回 Result
fn agentrt_channel_lookup(id: u32) -> Result<Arc<AgentrtChannel>> {
    CHANNELS.read()
        .get(&id)
        .cloned()
        .ok_or(ErrCode::NoEnt)
}

// 坏：unwrap 可能导致内核 panic
fn agentrt_channel_lookup(id: u32) -> Arc<AgentrtChannel> {
    CHANNELS.read().get(&id).unwrap().clone()  // 通道不存在时 panic
}
```

### 5.2 ? 运算符（OS-STD-037）

> **OS-STD-037**：使用 `?` 运算符传播错误，但需确保 `?` 返回的错误类型与函数签名兼容。这是 Rust 版本的内核 goto 集中出口模式——资源通过 RAII 自动释放，错误通过 `?` 向上传播。

```rust
fn agentrt_session_create(name: &str) -> Result<Arc<AgentrtSession>> {
    let session = Arc::new(AgentrtSession::new()?);  // 失败自动 return Err
    let buf = Box::<[u8; 4096]>::new_uninit()?;      // 失败自动释放 session
    let chan = agentrt_channel_create(name)?;          // 失败自动释放 session + buf
    // RAII 保证：无需手动 goto 清理
    Ok(session)
}
```

### 5.3 Option<T> 的使用（OS-STD-038）

> **OS-STD-038**：存在性检查使用 `Option<T>`，不要使用 `-1` 或 `null` 等哨兵值。`Option` 的类型系统保证调用者必须处理 `None` 情况。

```rust
// 好：Option 表达"可能不存在"
fn agentrt_task_find(id: u32) -> Option<Arc<AgentrtTask>> {
    TASKS.read().get(&id).cloned()
}

// 使用时：强制处理 None 情况
if let Some(task) = agentrt_task_find(task_id) {
    task.submit()?;
}
```

---

## 6. 内核抽象使用规范

### 6.1 kernel crate（OS-STD-039）

> **OS-STD-039**：agentrt-liunx（AirymaxOS）内核模块 Rust 代码使用 `kernel` crate 提供的安全抽象访问内核 API。禁止直接使用 `bindings::*` 中的裸函数（除非有对应的安全封装）。

```rust
use kernel::prelude::*;
use kernel::sync::{Mutex, Arc};
use kernel::alloc::{KBox, KVec};
use kernel::str::CString;

// 好：使用 kernel crate 的安全抽象
let buf = KBox::new_uninit_slice(4096)?;
let mut guard = self.lock.lock();

// 坏：直接使用 bindings 裸函数
unsafe { bindings::kmalloc(4096, bindings::GFP_KERNEL) };
```

### 6.2 内存分配（OS-STD-040）

> **OS-STD-040**：内核模块 Rust 代码使用 `kernel::alloc` 模块提供的分配器：`KBox`（内联分配）、`KVec`（动态数组）、`KBox::new_uninit_slice`（未初始化缓冲区）。禁止使用 `std` 的 `Box`/`Vec`——内核不链接标准库。

```rust
use kernel::alloc::{KBox, KVec};

// 内联分配
let task = KBox::new(AgentrtTask::new())?;

// 动态数组
let mut tasks: KVec<AgentrtTask> = KVec::new();
tasks.push(task, GFP_KERNEL)?;

// 未初始化缓冲区（零初始化由调用者负责）
let buf = KBox::new_uninit_slice(4096)?;
```

### 6.3 同步原语（OS-STD-041）

> **OS-STD-041**：内核模块 Rust 代码使用 `kernel::sync` 的同步原语：`Mutex<T>`、`SpinLock<T>`、`Arc<T>`、`CondVar`。这些是对 Linux 内核 `struct mutex`、`spinlock_t`、`kref` 的安全封装。

```rust
use kernel::sync::{Mutex, SpinLock, Arc};

pub struct AgentrtChannel {
    // 进程上下文锁（可睡眠）
    lock: Mutex<ChannelInner>,
    // 中断上下文锁（不可睡眠）
    irq_lock: SpinLock<IrqData>,
}

// Arc 提供了内核引用计数
pub type ChannelRef = Arc<AgentrtChannel>;
```

---

## 7. Pin-Init 模式

### 7.1 Pin 的必要性（OS-STD-042）

> **OS-STD-042**：内核中许多数据结构一旦创建就不能移动（例如，被链表嵌入、被 RCU 保护、包含自引用）。对于这些类型，必须使用 `Pin<T>` 确保其内存地址不变。

```rust
use kernel::pin_init;
use kernel::sync::Mutex;

#[pin_data]
pub struct AgentrtTaskTable {
    #[pin]
    lock: Mutex<TaskTableInner>,
    tasks: KVec<AgentrtTask>,
}

impl kernel::InPlaceInit for AgentrtTaskTable {
    fn init(ptr: *mut Self) -> impl PinInit<Self, kernel::error::Error> {
        pin_init!(AgentrtTaskTable {
            lock <- Mutex::new(TaskTableInner::new()),
            tasks: KVec::new(),
        })
    }
}
```

### 7.2 Pin-Init 宏（OS-STD-043）

> **OS-STD-043**：使用 `pin_init!` 宏进行 pinned 初始化，而非手动构造。`pin_init!` 确保所有字段在 pin 约束下正确初始化，避免 UB。

```rust
// 使用 pin_init! 宏安全初始化
let table = KBox::pin_init(
    pin_init!(AgentrtTaskTable {
        lock <- Mutex::new(TaskTableInner::new()),
        tasks: KVec::new(),
    }),
    GFP_KERNEL,
)?;
```

---

## 8. 与 C 代码的互操作规范（FFI 边界）

### 8.1 extern "C" 声明（OS-STD-044）

> **OS-STD-044**：所有跨 C/Rust 边界的函数必须使用 `extern "C"` 声明，确保 ABI 兼容。Rust 侧的函数签名必须与 C 侧完全一致（类型、顺序、调用约定）。

```rust
// Rust 侧：导出给 C 调用的函数
#[no_mangle]
pub extern "C" fn agentrt_ipc_channel_create_rs(
    name: *const kernel::ffi::c_char,
    out: *mut *mut AgentrtIpcChannel,
) -> i32 {
    // 将 C 参数转换为 Rust 类型
    let name_str = unsafe { CStr::from_ptr(name) };
    let name = name_str.to_str().unwrap_or("unknown");

    match agentrt_ipc_channel_create(name) {
        Ok(chan) => {
            unsafe { *out = Arc::into_raw(chan) as *mut _ };
            0
        }
        Err(e) => e.to_errno(),
    }
}
```

### 8.2 类型映射（OS-STD-045）

> **OS-STD-045**：FFI 边界的类型必须使用 `core::ffi` 或 `kernel::ffi` 中的 C 兼容类型。禁止在 FFI 边界上使用 Rust 特有类型（`&str`、`Vec<T>`、`Option<T>` 等）。

| C 类型 | Rust FFI 类型 | 说明 |
|--------|--------------|------|
| `int` | `core::ffi::c_int` | 32 位有符号整数 |
| `u32` | `u32` | 直接映射，大小对齐 |
| `const char *` | `*const core::ffi::c_char` | 以 NUL 结尾的字符串 |
| `void *` | `*mut core::ffi::c_void` | 不透明指针 |
| `size_t` | `usize` | 大小类型 |
| `struct agentrt_task *` | `*mut AgentrtTask` | 结构体指针 |

### 8.3 所有权语义（OS-STD-046）

> **OS-STD-046**：FFI 边界必须明确所有权的转移方向。Rust 侧的注释应使用 `#[ownership]` 标注来表示所有权语义。

```rust
/// 创建 IPC 通道。
///
/// # Ownership
/// - `name`: 借用，调用者拥有，仅读取
/// - `out`: 转移所有权，被调用者写入，调用者负责释放
///
/// # Safety
/// - `name` 必须是以 NUL 结尾的有效 C 字符串
/// - `out` 必须指向有效的内存区域
#[no_mangle]
pub unsafe extern "C" fn agentrt_ipc_channel_create(
    name: *const c_char,
    out: *mut *mut AgentrtIpcChannel,
) -> c_int {
    // ...
}
```

---

## 9. 代码示例

### 9.1 完整的 Rust 内核模块示例

```rust
// SPDX-License-Identifier: GPL-2.0
//! agentrt-liunx（AirymaxOS）IPC 通道内核模块（Rust 实现）
//!
//! [SS] 语义同源层：API 签名与 agentrt 用户态 AgentrtIpcChannel 一致。
//!
//! Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

use kernel::prelude::*;
use kernel::sync::{Arc, Mutex};
use kernel::alloc::KVec;
use kernel::str::CString;
use kernel::error::code::*;

module! {
    type: AgentrtIpcModule,
    name: "agentrt_ipc",
    author: "SPHARX Ltd.",
    description: "agentrt-liunx（AirymaxOS）IPC Channel Module (Rust)",
    license: "GPL",
}

/// IPC 通道内部状态。
struct ChannelInner {
    name: CString,
    pending_msgs: KVec<AgentrtIpcMsg>,
    max_msgs: usize,
}

/// [SS] 语义同源层：IPC 通道，与 agentrt 用户态语义等价。
pub struct AgentrtIpcChannel {
    inner: Mutex<ChannelInner>,
}

impl AgentrtIpcChannel {
    /// 创建新的 IPC 通道。
    pub fn create(name: &str, max_msgs: usize) -> Result<Arc<Self>> {
        let inner = ChannelInner {
            name: CString::try_from(name)?,
            pending_msgs: KVec::new(),
            max_msgs,
        };
        Ok(Arc::new(Self {
            inner: Mutex::new(inner),
        }))
    }

    /// 发送消息到通道。
    pub fn send(&self, msg: AgentrtIpcMsg) -> Result<()> {
        let mut guard = self.inner.lock();
        if guard.pending_msgs.len() >= guard.max_msgs {
            return Err(EAGAIN);
        }
        guard.pending_msgs.push(msg)?;
        Ok(())
    }

    /// 接收消息（非阻塞）。
    pub fn recv(&self) -> Option<AgentrtIpcMsg> {
        let mut guard = self.inner.lock();
        guard.pending_msgs.pop_front()
    }
}
```

---

## 10. Mermaid 架构图：Rust 内核模块与 C 内核的交互

```mermaid
graph TD
    subgraph "用户态"
        AGENT["agentrt 用户态运行时<br/>Rust / Python / TS"]
    end

    subgraph "内核态"
        subgraph "Rust 内核模块 [IND]"
            R_SEC["airymaxos-security<br/>安全策略裁决"]
            R_FS["airymaxos-fs<br/>文件系统解析"]
            R_CFG["airymaxos-config<br/>配置解析"]
        end

        subgraph "C 内核核心 [IND]"
            C_SCHED["调度器"]
            C_MM["内存管理"]
            C_IPC["io_uring IPC"]
        end

        subgraph "[SC] 共享契约层"
            SC_HDR["include/airymax/<br/>ipc_msg.h / errno.h<br/>types.h / capability.h"]
        end
    end

    AGENT -->|"syscall"| C_IPC
    AGENT -.->|"[SS] 语义同源"| R_SEC
    R_SEC -->|"extern \"C\" FFI"| C_IPC
    R_FS -->|"extern \"C\" FFI"| C_MM
    R_CFG -->|"extern \"C\" FFI"| C_SCHED
    SC_HDR --> R_SEC
    SC_HDR --> C_IPC
    SC_HDR --> AGENT

    style AGENT fill:#16213e,stroke:#0f3460,color:#eee
    style R_SEC fill:#0f3460,stroke:#e94560,color:#eee
    style C_SCHED fill:#1a1a2e,stroke:#e94560,color:#eee
    style SC_HDR fill:#e94560,stroke:#e94560,color:#fff
```

---

## 11. 五维正交原则映射

| 章节 | 核心原则 | 映射 |
|------|---------|------|
| §2 所有权与借用 | E-3 资源确定性、K-2 接口契约化 | 所有权模型保证资源生命周期 |
| §3 命名约定 | E-5 命名语义化、K-2 接口契约化 | `agentrt_` 前缀隔离 |
| §4 unsafe 规范 | E-1 安全内生、A-2 细节关注 | 最小化 + 文档化 + 审查 |
| §5 错误处理 | E-6 错误可追溯、E-3 资源确定性 | `?` + RAII 替代 goto |
| §6 内核抽象 | E-1 安全内生、K-2 接口契约化 | kernel crate 安全封装 |
| §7 Pin-Init | E-3 资源确定性、A-2 细节关注 | 不可移动保证 |
| §8 FFI 互操作 | E-1 安全内生、K-2 接口契约化 | 显式 ABI 边界 |

---

## 12. 相关文档

- [编码规范总览](README.md)：规范体系总索引
- [Rust 安全编码规范](Rust_secure_coding_standard.md)：内核模块安全编码
- [C 编码风格规范](C_coding_style_standard.md)：内核态 C 风格
- [C 安全编码规范](C_Cpp_secure_coding_standard.md)：内核态安全编码
- [工程标准规范 03-代码风格](../../50-engineering-standards/03-code-style.md)：工程风格决策
- [五维正交 24 原则](../../10-architecture/02-five-dimensional-principles.md)
- Rust for Linux 社区：`rust/kernel/` 目录代码风格

---

## 13. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-07 | 初始版本：基于 Rust for Linux 社区约定，融合 agentrt-liunx 专属规范 |
| 1.0.1 | TBD | 首个开发版本：与代码实现同步验证 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.