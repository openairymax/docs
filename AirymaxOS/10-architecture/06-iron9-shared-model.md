Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# IRON-9 v3 共享模型
> **文档定位**：Airymax IRON-9 v3 跨端共享模型的唯一权威定义（AirymaxOS 侧）\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](10-unify-design.md)\
> **设计依据**：综合修正方案 §2.3（IRON-9 模型 SSoT）+ §3.1（P0 级缺失文档 #6）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax IRON-9 v3 跨端共享模型** 的唯一权威源（AirymaxOS 侧）。[SC] / [SS] / [IND] / [DSL] 四层定义、10 个 [SC] 头文件清单、四层模型应用归属均以本文件为唯一权威定义。
>
> IRON-9 v3 相比 v2 新增 **[DSL] 第四层**（降级生存层），确保 [SC] 头文件损坏/缺失时的最小可运行子集。本模型遵循 [10-unify-design.md](10-unify-design.md) 的技术选型（sched_tac + IORING_OP_URING_CMD + 纯 C LSM + alloc_pages/mmap）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：agentrt-linux 架构师、跨端集成工程师、agentrt 核心开发者
- **前置知识**：理解 [10-unify-design.md](10-unify-design.md) Unify Design 总纲、[11-degraded-survival-layer.md](11-degraded-survival-layer.md) [DSL] 降级层
- **预计阅读时间**：30 分钟
- **核心概念**：IRON-9 v3、[SC]、[SS]、[IND]、[DSL]、四层模型
- **复杂度标识**：高级

---

## §1 IRON-9 演进：v1（两层）→ v2（[SC]/[SS]/[IND] 三层）→ v3（新增 [DSL] 第四层）

### 1.1 演进背景

Airymax 体系包含两个核心工程：

| 工程 | 定位 | 运行形态 |
|------|------|---------|
| **agentrt**（AirymaxAgentRT） | AI Agent 运行时平台工程 | 用户态跨平台（Linux/macOS/Windows） |
| **agentrt-linux**（AirymaxOS） | 智能体操作系统 | Linux 6.6 内核态发行版（ALK 6.6） |

两者之间存在大量共享的技术契约——IPC 消息头布局、错误码体系、capability 安全模型、调度参数、任务描述符格式等。IRON-9 跨端共享模型即为管理这些共享契约而建立。

### 1.2 三版演进路径

| 版本 | 层级数 | 层级构成 | 核心问题 | 解决方案 |
|------|--------|---------|---------|---------|
| **v1** | 2 层 | 共享 + 独立 | 缺少语义同源层，跨端 API 语义漂移 | 引入 [SS] 语义同源层 |
| **v2** | 3 层 | [SC] + [SS] + [IND] | 缺少降级生存层，[SC] 损坏即 brick | **新增 [DSL] 第四层** |
| **v3** | 4 层 | **[SC] + [SS] + [IND] + [DSL]** | 完整 | 四层模型 |

### 1.3 v3 的核心改进

IRON-9 v3 相比 v2 的唯一但关键改进是**新增 [DSL]（Degraded Survival Layer，降级生存层）作为第四层**。v2 的三层模型在 [SC] 头文件损坏时直接编译失败，导致系统无法启动；v3 通过 [DSL] 降级块确保 [SC] 损坏时仍具备最小可运行子集。

```
IRON-9 v2（三层）                    IRON-9 v3（四层）
┌────────────────────┐              ┌────────────────────┐
│ [SC] 共享契约层    │              │ [SC] 共享契约层    │
├────────────────────┤              ├────────────────────┤
│ [SS] 语义同源层    │              │ [SS] 语义同源层    │
├────────────────────┤              ├────────────────────┤
│ [IND] 完全独立层   │              │ [IND] 完全独立层   │
└────────────────────┘              ├────────────────────┤
   [SC] 损坏 → brick                │ [DSL] 降级生存层   │ ← 新增
                                    └────────────────────┘
                                       [SC] 损坏 → 降级可运行
```

[DSL] 的详细设计见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md)。

---

## §2 [SC] 共享契约层：物理宿主 kernel/include/airymax/，10 个头文件

### 2.1 [SC] 层定义

**[SC]（Shared Contract，共享契约层）**：agentrt 与 agentrt-linux **物理共享同一份头文件**，逐字节相同，通过 `#include` 引用。

**特征**：

- 物理宿主在 `agentrt-linux/kernel/include/airymax/` 目录
- agentrt 通过 `-I` 编译选项引用同一物理文件
- **禁止任何一端单方面修改**——变更必须双向同步，通过 `sc-dual-ci.yml` 逐字节校验
- 使用内核 UAPI 类型（`__u32`/`__u16`/`__u64`/`__u8`），不使用 `uint32_t` 等 C 标准库类型
- **禁止使用 `float` 类型**——内核态不支持浮点运算

### 2.2 10 个 [SC] 头文件清单

v3 将 [SC] 头文件数量从 v2 的 6 个扩展为 **10 个**，新增 `error.h`、`log_types.h`、`uapi_compat.h`、`lsm_types.h`：

| # | 头文件 | 物理路径 | 共享内容 | 关联模块 | 契约文档 |
|---|--------|---------|---------|---------|---------|
| 1 | `error.h` | `include/airymax/error.h` | `AIRY_E*` 错误码 + `AIRY_FAULT_*` 故障码 + [DSL] 降级块 | A-UEF | [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) |
| 2 | `log_types.h` | `include/airymax/log_types.h` | `AIRY_LOG_MAGIC` + 128B 记录 + 5 级日志枚举 + printk 映射 | A-ULP | [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) |
| 3 | `ipc.h` | `include/airymax/ipc.h` | IPC magic + 128B 消息头 + SQE/CQE 操作码 | A-IPC | [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) |
| 4 | `sched.h` | `include/airymax/sched.h` | Agent 8 态生命周期 + sched_tac 调度参数 + `MAC_MAX_AGENTS` | A-ULS | [10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) |
| 5 | `memory_types.h` | `include/airymax/memory_types.h` | MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 接口 | 记忆 | [120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) |
| 6 | `security_types.h` | `include/airymax/security_types.h` | POSIX capability 41 ID + LSM 钩子 252 ID + Cupolas blob 布局 | 安全 | [03-capability-model.md](../110-security/03-capability-model.md) |
| 7 | `cognition_types.h` | `include/airymax/cognition_types.h` | `airy_q16_t` Q16.16 定点数 + CoreLoopThree 三阶段 + Thinkdual 模式 | 认知 | [120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) |
| 8 | `syscalls.h` | `include/airymax/syscalls.h` | Syscall 编号体系（v1.1: 4 核心 + 20 预留 = 24 槽位） | 全部 | [01-syscalls.md](../30-interfaces/01-syscalls.md) |
| 9 | `uapi_compat.h` | `include/airymax/uapi_compat.h` | 三路类型桥接（`__KERNEL__` / `__linux__` / `#else`） | IRON-9 | [11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md) |
| 10 | `lsm_types.h` | `include/airymax/lsm_types.h` | 纯 C LSM 类型定义 + `DEFINE_LSM(airy)` 骨架 + Capability 缓存结构 | A-ULS/A-IPC | [07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) |

### 2.3 [SC] 头文件的 [DSL] 降级块

每个 [SC] 头文件底部均包含 `#ifdef AIRY_SC_FALLBACK` 降级块，提供 [SC] 损坏时的最小可运行子集。详见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md) §2.2。

### 2.4 v2 → v3 的 [SC] 数量变化

| 时期 | [SC] 数量 | 头文件 | 变化原因 |
|------|----------|--------|---------|
| v2 | 6 个 | memory_types/security_types/cognition_types/sched/ipc/syscalls | v2 基线 |
| **v3** | **10 个** | v2 的 6 个 + **error.h / log_types.h / uapi_compat.h / lsm_types.h** | Unify Design 5 模块需要独立的 [SC] 契约 |

v3 新增 4 个 [SC] 头文件的原因：

- `error.h`：A-UEF 模块需要独立的错误码 [SC] 契约（v2 时错误码散落在 agentrt 仓库，未统一）
- `log_types.h`：A-ULP 模块需要独立的日志类型 [SC] 契约（v2 时 128B 记录格式多处定义）
- `uapi_compat.h`：IRON-9 三路类型桥接需要独立 [SC] 头文件
- `lsm_types.h`：纯 C LSM（不使用 BPF LSM）需要独立的类型 [SC] 契约

---

## §3 [SS] 语义同源层：语义一致，签名独立演进

### 3.1 [SS] 层定义

**[SS]（Semantic Sibling，语义同源层）**：agentrt 与 agentrt-linux 的**高层 API 语义同源**——功能语义一致，但函数签名各自独立演进。

**特征**：

- 不共享同一份源码，但 API 语义保持一致
- agentrt 侧使用 JSON-RPC 风格高层 API
- agentrt-linux 侧使用编号 syscall 风格
- SDK 层签名同源（同一份源码两端编译）
- 详细的语义映射表见 [30-interfaces/05-syscall-semantic-mapping.md](../30-interfaces/05-syscall-semantic-mapping.md)

### 3.2 [SS] 语义同源映射示例

| # | agentrt 高层 API | agentrt-linux 编号 syscall | 语义 |
|---|-----------------|--------------------------|------|
| 1 | `airy_sys_task_submit(input, input_len, timeout_ms, out_output)` | #512 `airy_sys_task_submit(task_desc, priority)` | 提交 Agent 任务 |
| 2 | `airy_sys_task_query(task_id, out_status)` | #513 `airy_sys_task_query(task_id)` | 查询任务状态 |
| 3 | `airy_sys_task_cancel(task_id)` | #514 `airy_sys_task_cancel(task_id)` | 取消任务 |
| 4 | `airy_sys_agent_spawn(config, out_agent_id)` | #516 `airy_sys_agent_spawn(config)` | 创建 Agent |
| 5 | `airy_sys_capability_request(...)` | #544 `airy_sys_capability_request(...)` | 请求 capability |
| 6 | `airy_sys_capability_revoke(...)` | #545 `airy_sys_capability_revoke(...)` | 撤销 capability |

### 3.3 [SS] 的配置同源

A-UCS 模块的配置也属于 [SS] 层——内核侧 Kconfig/sysctl 与用户态 JSON/TOML 语义映射。详见 [10-unify-design.md](10-unify-design.md) §6.1。

---

## §4 [IND] 完全独立层：agentrt POSIX MQ + mmap ↔ agentrt-linux io_uring + IORING_OP_URING_CMD

### 4.1 [IND] 层定义

**[IND]（Independent，完全独立层）**：agentrt 与 agentrt-linux 各自独有，不共享。

**特征**：
- agentrt 独有：跨平台抽象（POSIX/libc）、JSON-RPC 统一入口 `airy_syscall_invoke()`
- agentrt-linux 独有：内核态 syscall 编号（512-631）、LSM 钩子集成、KABI 二进制兼容、Kbuild/Kconfig 构建系统

### 4.2 [IND] 的 IPC 实现差异

两端 IPC 实现完全独立，但通过 [SC] 的 `ipc.h` 共享 128B 消息头布局：

| 维度 | agentrt（[IND]） | agentrt-linux（[IND]） |
|------|-----------------|---------------------|
| IPC 机制 | POSIX MQ + mmap 共享内存 | io_uring + `IORING_OP_URING_CMD` |
| 零拷贝 | mmap 共享页 | registered buffer + mmap |
| 同步原语 | 信号量 | eventfd |
| 调度 | MicroCoreRT 用户态调度 | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF） |
| 安全 | 应用权限模型 | 纯 C LSM + Capability 系统 |

### 4.3 agentrt-linux [IND] 独有内容

| # | 独有内容 | 说明 |
|---|---------|------|
| 1 | 内核态 syscall 编号（512-631） | OS 级系统调用 |
| 2 | 纯 C LSM 钩子集成（252 个） | 对齐 openEuler，不使用 BPF LSM |
| 3 | KABI 二进制兼容 | 内核 ABI 稳定性 |
| 4 | Kbuild/Kconfig 构建系统 | 内核模块构建 |
| 5 | `IORING_OP_URING_CMD` 语义映射 | IPC fastpath 载体（不使用 page flipping） |
| 6 | `alloc_pages + mmap` 共享内存 | 日志/IPC 内存方案（不使用 DMA 一致性内存） |

---

## §5 [DSL] 降级生存层：[SC] 损坏时的最小可运行子集

### 5.1 [DSL] 层定义

**[DSL]（Degraded Survival Layer，降级生存层）**：[SC] 头文件损坏/缺失时的最小可运行子集。每个 [SC] 头文件底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块，提供自包含的最小符号集。

**特征**：

- **自包含**：降级块不依赖任何外部头文件
- **最小集**：仅定义维持系统启动所需的最小符号集
- **显式告警**：降级块生效时发出 `#warning`
- **最后防线**：[DSL] 是韧性领域的最后防线

### 5.2 [DSL] 降级时的最小可运行子集

| 维度 | 正常模式 | [DSL] 降级模式 |
|------|---------|--------------|
| 错误码 | 5 子空间（300 码） | 38 个 POSIX 码 + 1 个配置码 |
| 日志 | Ring Buffer + Logger Daemon | printk 原生（仅 LOG_FATAL + LOG_ERROR） |
| IPC | 完整 128B 消息头 + 3 操作 | 最简 128B 消息头 + 2 操作 |
| 调度 | sched_tac 三层 | EEVDF 默认 |
| 安全 | 纯 C LSM 完整校验 | 仅 POSIX capability |
| Fault | Fault Handler 优雅处理 | 统一 Panic |

详细设计见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md)。

### 5.3 [DSL] 的独立性保障

[DSL] 降级块虽然在 [SC] 头文件内部，但设计为**自包含**——不依赖 [SC] 的任何其他符号。构建系统对每个降级块单独计算 hash（存入 `.fallback_hashes`），即使 [SC] 主体损坏，只要降级块 hash 校验通过，系统仍可降级启动。

---

## §6 四层模型应用：每个技术点的四层归属

### 6.1 技术点四层归属矩阵

下表给出 Airymax Unify Design 5 模块中各技术点的四层归属：

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 权威文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| 错误码（`AIRY_E*`） | ● | — | — | ● | [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) |
| 故障码（`AIRY_FAULT_*`） | ● | — | — | — | [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) |
| 128B 日志记录 | ● | — | — | ● | [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) |
| 5 级日志枚举 | ● | — | — | ● | [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) |
| 128B IPC 消息头 | ● | — | — | ● | [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) |
| IPC 操作码 | ● | — | — | ● | [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) |
| Agent 8 态生命周期 | ● | — | — | — | [10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) |
| sched_tac 调度参数 | ● | — | ● | — | [10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) |
| Capability 41 ID | ● | — | — | ● | [03-capability-model.md](../110-security/03-capability-model.md) |
| 纯 C LSM 类型 | ● | — | ● | — | [07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) |
| syscall 编号 | ● | — | — | ● | [01-syscalls.md](../30-interfaces/01-syscalls.md) |
| syscall 高层 API 语义 | — | ● | — | — | [05-syscall-semantic-mapping.md](../30-interfaces/05-syscall-semantic-mapping.md) |
| io_uring + URING_CMD | — | — | ● | — | [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) |
| alloc_pages + mmap | — | — | ● | — | [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) |
| MemoryRovol L1-L4 | ● | — | — | ● | [120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) |
| 配置 sysctl ↔ JSON | — | ● | — | — | [11-unified-config.md](../20-modules/11-unified-config.md) |
| printk 8 级映射 | ● | — | — | — | [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) |
| 三路类型桥接 | ● | — | — | — | [11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md) |

### 6.2 归属判定规则

技术点的四层归属按以下规则判定：

1. **[SC]**：二进制布局、magic 值、枚举编号、结构体定义 → 物理共享
2. **[SS]**：API 语义、配置语义 → 语义同源，签名独立
3. **[IND]**：实现机制、构建系统、平台特性 → 完全独立
4. **[DSL]**：[SC] 的降级回退 → 自包含降级块

### 6.3 CI 校验

`sc-dual-ci.yml` 对 [SC] 头文件进行逐字节校验，`ssot-validate.yml` 对四层归属进行一致性校验：

- [SC] 头文件数量必须为 10（防止误增/误删）
- 每个 [SC] 头文件必须有对应的 [DSL] 降级块
- [SS] 语义映射必须有对应的语义映射文档
- [IND] 实现不得泄露到 [SC]/[SS] 层

---

## §7 跨端一致性校验

### 7.1 CI 校验机制

| # | 校验项 | 校验方法 | 校验 workflow |
|---|--------|---------|--------------|
| 1 | [SC] 头文件逐字节一致 | `diff` 两端 10 个头文件 | `sc-dual-ci.yml` |
| 2 | [SC] 数量为 10 | `ls include/airymax/*.h \| wc -l == 10` | `ssot-validate.yml` |
| 3 | 每个 [SC] 有 [DSL] 降级块 | `grep -l AIRY_SC_FALLBACK include/airymax/*.h \| wc -l == 10` | `ssot-validate.yml` |
| 4 | IPC magic 一致 | 两端均为 `0x41524531` | `sc-dual-ci.yml` |
| 5 | 错误码值一致 | 两端 `error.h` 逐字节相同 | `sc-dual-ci.yml` |
| 6 | 日志 magic 一致 | 两端均为 `0x414C4F47` | `sc-dual-ci.yml` |
| 7 | 编译器无关性 | `check-uapi-compiler-agnostic.sh` | `sc-dual-ci.yml` |

### 7.2 变更流程

[SC] 共享契约层的变更必须遵循以下流程：

1. **提议**：在对应的契约文档（如 `08-sc-error-contract.md`）提出变更提案
2. **双向评审**：agentrt 和 agentrt-linux 双方维护者评审
3. **同步修改**：修改 `kernel/include/airymax/*.h` 物理头文件
4. **CI 验证**：`sc-dual-ci.yml` 双端逐字节校验通过
5. **文档更新**：更新对应契约文档版本号 + 本文件版本号

---

## §8 相关文档

- [10-unify-design.md](10-unify-design.md) —— Airymax Unify Design 总纲（5 模块）
- [11-degraded-survival-layer.md](11-degraded-survival-layer.md) —— [DSL] 降级生存层详细设计
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— A-UEF [SC] error.h 契约
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— A-ULP [SC] log_types.h 契约
- [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) —— 跨项目代码共享（v2 基线）
- 综合修正方案 §2.3 / §3.1 —— 设计依据

---

## §9 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本（IRON-9 v3）：新增 [DSL] 第四层；[SC] 头文件从 v2 的 6 个扩展为 10 个（新增 error.h / log_types.h / uapi_compat.h / lsm_types.h）；四层模型应用归属矩阵；CI 校验机制 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | IRON-9 v3 共享模型 | v1.0 | 2026-07-17
