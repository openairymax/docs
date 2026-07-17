Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 接口设计
> **文档定位**：agentrt-linux（AirymaxOS） 接口设计层的总览与索引\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[agentrt-linux 总览](../README.md)\
> **设计依据**：[15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §3.1（P0 级缺失文档 #3/#4）+ §6.2.1（30-interfaces 一致性检查）

---

## 1. 接口设计概览

agentrt-linux 接口设计层定义 8 子仓之间、内核与用户态之间、OS 与应用之间的全部接口契约。所有接口遵循"机制在内核、策略在用户态"的微内核原则，并与 agentrt 同源模块保持语义兼容。

接口分为 5 类（v1.0 新增 [SC] 共享契约类别，对齐 Airymax Unify Design 5 模块与 IRON-9 v3 四层模型）：

| # | 接口类别 | 通信形态 | 同源 agentrt | 设计目标 |
|---|---------|---------|--------------|---------|
| 1 | 系统调用（syscall） | 用户态 → 内核态 | MicroCoreRT 调度 + AgentsIPC | Agent 任务管理、IPC、内存、调度、安全、认知的内核入口 |
| 2 | IPC 协议 | 进程 ↔ 进程 | AgentsIPC 128B 消息头 | io_uring + IORING_OP_URING_CMD 零拷贝消息传递（不使用 page flipping） |
| 3 | SDK API | 应用 → OS | sdk 管理接口 | 4 语言统一封装的客户端 |
| 4 | 编码规范 | 代码级约定 | commons 公共工具 | 命名、风格、日志、Doxygen、安全编码 |
| 5 | **[SC] 共享契约** | 内核态 ↔ 用户态（跨端） | 物理共享头文件 | UEF/ULPS/UIPF/USV 的二进制布局契约，物理宿主 `kernel/include/airymax/` |

---

## 2. 接口分类矩阵

下表给出 5 类接口的详细分类、覆盖子仓与关键契约（v1.0 新增 [SC] 共享契约类）。

| 接口类别 | 子分类 | 覆盖子仓 | 关键契约 | 文档 |
|---------|--------|---------|---------|------|
| 系统调用 | agent 任务管理 | kernel / cognition | `AIRY_SYS_TASK_*` 用户态调度器策略 | [01-syscalls.md](01-syscalls.md) |
| 系统调用 | IPC | kernel / services | `AIRY_SYS_IPC_*` io_uring 零拷贝 | 01-syscalls.md |
| 系统调用 | 内存 | kernel / memory | `AIRY_SYS_ROVOL_*` 记忆卷载 | 01-syscalls.md |
| 系统调用 | 调度 | kernel | `AIRY_SYS_SCHED_*` 方案 C-Prime 调度策略 | 01-syscalls.md |
| 系统调用 | 安全 | kernel / security | `AIRY_SYS_CAP_*` capability 令牌 | 01-syscalls.md |
| 系统调用 | 认知 | kernel / cognition | `AIRY_SYS_CLT_*` CoreLoopThree kthread | 01-syscalls.md |
| IPC 协议 | 消息头 | 全部子仓 | 128B 定长 + 5 种 payload | [02-ipc-protocol.md](02-ipc-protocol.md) |
| IPC 协议 | 通信原语 | services | Channel / Socket / FIFO / Eventpair | 02-ipc-protocol.md |
| IPC 协议 | 零拷贝 | kernel / services | io_uring + IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping） | 02-ipc-protocol.md |
| SDK API | Python | 全部子仓 | `agentrt.AirymaxClient` 4 嵌套客户端 | [03-sdk-api.md](03-sdk-api.md) |
| SDK API | Rust | 全部子仓 | `agentrt::AirymaxClient` 4 嵌套客户端 | 03-sdk-api.md |
| SDK API | Go | 全部子仓 | `github.com/openairymax/agentrt-go` | 03-sdk-api.md |
| SDK API | TypeScript | 全部子仓 | `@openairymax/agentrt` | 03-sdk-api.md |
| 编码规范 | 命名 | 全部子仓 | `airy_` / `<service>_d` / `module_action_object()` | [04-coding-standard.md](04-coding-standard.md) |
| 编码规范 | C 风格 | kernel / security / memory | 4 空格 + snake_case + 1TBS + Doxygen | 04-coding-standard.md |
| 编码规范 | Rust 风格 | cognition / cloudnative | rustfmt + clippy + snake_case 函数 | 04-coding-standard.md |
| 编码规范 | 日志 | 全部子仓 | ANSI 颜色 + log_write() + CLOCK_REALTIME | 04-coding-standard.md |
| **[SC] 共享契约** | 错误码 | kernel / 全部 | `AIRY_E*` Error（负数）+ `AIRY_FAULT_*` Fault（0x1000+）+ [DSL] 降级块 | [08-sc-error-contract.md](08-sc-error-contract.md) |
| **[SC] 共享契约** | 日志类型 | kernel / services | 128B 固定记录 + 5 级日志枚举 + printk 8 级映射 | [09-sc-log-types-contract.md](09-sc-log-types-contract.md) |

### 2.1 接口覆盖范围

接口覆盖范围按子仓维度展开如下，每个子仓对外暴露的接口均落入上述 5 类之中：

| 子仓 | 系统调用 | IPC 协议 | SDK API | 编码规范 | [SC] 共享契约 |
|------|---------|---------|---------|---------|--------------|
| kernel | `AIRY_SYS_TASK_*` / `IPC_*` / `ROVOL_*` / `SCHED_*` / `CAP_*` / `CLT_*` | 128B 消息头内核侧 | 不直接暴露 | C 风格 + Doxygen | 物理宿主 `kernel/include/airymax/`（10 个头文件） |
| services | 通过系统调用 + IPC | 128B 消息头用户态 + 4 通信原语 | daemon 客户端 | C 风格 + 日志 | `#include <airymax/*.h>` 引用 |
| security | `AIRY_SYS_CAP_*` / `LSM_*` | capability 携带 | SafetyClient | C 风格 + 安全编码 | `lsm_types.h` / `security_types.h` |
| memory | `AIRY_SYS_ROVOL_*` / `CXL_*` / `MGLRU_*` | MemoryRovol 迁移消息 | MemoryClient（嵌套） | C 风格 + 内存安全 | `memory_types.h` |
| cognition | `AIRY_SYS_CLT_*` / `WASM_*` | CoreLoopThree 阶段通知 | CognitionClient + ChatClient | Rust 风格 | `cognition_types.h` |
| cloudnative | 不直接暴露 | gRPC + CRD | agentctl + SDK | Go 风格 | 不直接涉及 |
| system | sysctl / procfs | 不直接暴露 | DevStation CLI | 多语言 | 不直接涉及 |
| tests-linux | 不直接暴露 | 测试协议 | 测试框架 API | 各语言规范 | 验证 [SC] 双端一致 |

---

## 3. 接口设计原则

接口设计严格遵循五维正交 24 原则中的 K-2 与 E-7（详见 [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md)）：

### 3.1 K-2 接口契约化

- **显式契约**: 所有接口以 C 头文件 + Doxygen 注释形式给出显式契约，包括参数语义、返回值、错误码、并发约束。
- **ABI 稳定**: 系统调用编号、IPC 消息头布局、capability 令牌格式在 MAJOR 版本内保持 ABI 稳定，破坏性变更必须升级 MAJOR 版本。
- **版本协商**: IPC 消息头携带 `version` 字段（当前 0x0100），支持协议版本协商。
- **错误码对齐**: 全部接口错误码对齐 `include/airymax/error.h`（[SC] SSoT），与 agentrt 同源且部分代码共享（IRON-9 v2）。
- **并发约束显式**: 所有接口在 Doxygen 注释中显式声明线程安全性与可重入性（thread-safe / reentrant / async-signal-safe）。

### 3.2 E-7 文档即代码

- **头文件即文档**: C 头文件（`airy_syscalls.h`、`airy_ipc_msg.h`）即接口文档，Doxygen 注释直接生成 API 文档。
- **示例即规约**: SDK 文档中的 Python/Rust/Go/TypeScript 代码示例即为接口使用规约，可直接作为冒烟测试用例。
- **生成校验**: 通过 Doxygen 生成 HTML 文档 + 通过 SDK 示例生成冒烟测试，确保文档与代码同步。
- **接口评审**: 所有接口变更必须经过接口评审（API Review），检查契约一致性、命名规范、错误码规范。

### 3.3 接口设计补充原则

- **机制与策略分离**: 系统调用提供机制（如 `airy_sys_task_submit` 提交任务），调度策略由用户态 Macro-Supervisor 通过方案 C-Prime（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射）定义，不使用 sched_ext。
- **最小权限**: 默认拒绝，通过 capability 显式授权（`airy_sys_capability_request`），纯 C LSM 模块强制校验（不使用 BPF LSM）。
- **零拷贝优先**: IPC 与内存接口优先使用 io_uring + IORING_OP_URING_CMD + registered buffer + mmap 零拷贝路径（不使用 page flipping），日志内存使用 alloc_pages + mmap（不使用 DMA 一致性内存）。
- **同源兼容**: 与 agentrt 同源的接口保持 API 语义兼容，仅升级底层实现。

---

## 4. 本目录文档索引

| # | 文档 | 核心内容 | 行数级别 |
|---|------|---------|---------|
| 1 | [01-syscalls.md](01-syscalls.md) | 系统调用分类 + 编号规则 + C 接口 + 关键调用清单 + 性能约束 + 错误码 | 350-450 |
| 2 | [02-ipc-protocol.md](02-ipc-protocol.md) | 128B 消息头 + 5 种 payload + io_uring + IORING_OP_URING_CMD 零拷贝 + 同源映射 + 性能约束 | 350-450 |
| 3 | [03-sdk-api.md](03-sdk-api.md) | 4 语言矩阵 + 4 嵌套客户端 + Python/Rust/Go/TS 示例 + 错误处理 | 450-550 |
| 4 | [04-coding-standard.md](04-coding-standard.md) | 15 规范引用 + 命名规范 + C/Rust 风格 + 日志 + Doxygen + 安全编码 + 对照 | 350-450 |
| 5 | [05-syscall-semantic-mapping.md](05-syscall-semantic-mapping.md) | P0-03 方案 D 分层 API 设计 + [SS] 语义映射表 + [IND] 独有调用 + SDK 层签名同源 + 调用路径选择 | 400-500 |
| 6 | [06-codegen-pipeline.md](06-codegen-pipeline.md) | 代码生成流水线 + 多语言 SDK 自动生成 + 契约驱动开发 | 300-400 |
| 7 | [07-ipc-fastpath.md](07-ipc-fastpath.md) | IPC fastpath 设计 + IORING_OP_URING_CMD + registered buffer + mmap + Capability 分层校验 + 数据面自治三原则 | 400-500 |
| 8 | [08-sc-error-contract.md](08-sc-error-contract.md) | **[SC] error.h 二进制契约** + Error（负数可恢复）vs Fault（正数 0x1000+ 不可恢复）分层 + 5 子空间分配 + CI 逐字节校验 + [DSL] 降级块 | 300-400 |
| 9 | [09-sc-log-types-contract.md](09-sc-log-types-contract.md) | **[SC] log_types.h 二进制契约** + 128B 固定记录格式 + 5 级日志枚举 + printk 8 级映射 | 250-350 |

> **[SC] 共享契约说明**：文档 #8、#9 是 Airymax Unify Design 五模块（UEF/ULPS）的 [SC] 共享契约权威定义，物理宿主为 `kernel/include/airymax/error.h` 与 `kernel/include/airymax/log_types.h`，由 `sc-dual-ci.yml` 进行双端逐字节校验。详见 [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) 与 [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)。

## 5. 与 agentrt 接口的关系

agentrt-linux 接口与 agentrt 接口遵循 IRON-9 v3"四层共享模型"原则（[SC] 共享契约 + [SS] 语义同源 + [IND] 完全独立 + [DSL] 降级生存，详见 [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)）：

| 维度 | agentrt 接口 | agentrt-linux 接口 | 关系 | IRON-9 v3 层级 |
|------|-------------|---------------|------|---------------|
| 系统调用 | 无（用户态运行时） | `AIRY_SYS_*` 内核系统调用 | agentrt-linux 新增（OS 级） | [IND] |
| IPC 消息头 | AgentsIPC 128B 定长 | 同源 128B 定长（字段对齐） | 语义同源，布局兼容 | [SC]（`ipc.h`） |
| IPC payload | 自定义协议 | 5 种 payload（REQUEST/RESPONSE/EVENT/STREAM/CONTROL） | agentrt-linux 扩展 | [IND] |
| SDK | sdk（应用层） | agentctl + 4 语言 SDK | 同源语义，OS 级封装 | [SS] |
| capability | 应用权限模型 | seL4 风格 capability 系统 + 纯 C LSM | 同源语义，OS 级升级 | [SC]（`security_types.h`）+ [IND] |
| 调度 | MicroCoreRT 用户态调度 | 方案 C-Prime（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射） | 同源语义，内核态实现 | [SC]（`sched.h`）+ [IND] |
| 错误码 | `AIRY_E*` 用户态 | `AIRY_E*` + `AIRY_FAULT_*` 内核态 | 语义同源，内核态扩展 Fault | [SC]（`error.h`） |
| 日志 | `LOG_*` 用户态 | `LOG_*` + 128B Ring Buffer 内核态 | 语义同源，内核态扩展 Ring | [SC]（`log_types.h`） |
| 记忆 | MemoryRovol 用户态 | MemoryRovol 内核态 | 同源语义，内核态升级 | [SC]（`memory_types.h`） |
| 认知循环 | CoreLoopThree 用户态 | CoreLoopThree kthread | 同源语义，内核态加速 | [SC]（`cognition_types.h`） |

**同源红利**: agentrt 在 agentrt-linux 上运行时，IPC 消息头布局兼容、调度语义一致、capability 模型一致，无需适配层即可天然契合。具体表现：

- **IPC 兼容**: agentrt 发出的 128B 消息头可直接被 agentrt-linux io_uring + IORING_OP_URING_CMD 通道接收，无需协议转换（不使用 page flipping）。
- **调度兼容**: agentrt 的任务优先级语义（0-139）与方案 C-Prime（SCHED_DEADLINE/SCHED_FIFO/EEVDF）对齐，调度行为一致（不使用 sched_ext）。
- **记忆兼容**: agentrt 的 MemoryRovol 用户态 API 与 agentrt-linux 内核态实现语义对齐，可平滑迁移。

**独立性**: agentrt-linux 接口为 OS 级接口（系统调用、内核 IPC、OS 级 SDK），与 agentrt 的应用级接口在分层上独立，不构成依赖耦合。当 agentrt 演进时，agentrt-linux 通过接口契约评审决定是否同步，避免被动跟随。独立性体现在：

- **分层独立**: agentrt-linux 接口位于 OS 层（系统调用 + 内核 IPC），agentrt 接口位于应用层（SDK + 用户态 IPC），二者通过契约解耦。
- **演进独立**: agentrt-linux 接口随 Linux 6.6 内核基线演进节奏，agentrt 接口随应用层需求演进，互不阻塞。
- **测试独立**: agentrt-linux 接口由 `tests-linux` 子仓验证（形式化 + Soak + 混沌），agentrt 接口由 agentrt 自有测试覆盖。

### 5.1 接口演进与版本协商

agentrt-linux 接口采用语义化版本 + ABI 兼容承诺的演进策略：

- **MAJOR 版本**: 不兼容的接口变更，需重新评审全部契约，触发 K-2 契约重签。
- **MINOR 版本**: 向后兼容的新增接口（新系统调用编号、新 IPC payload 类型），不影响既有调用方。
- **PATCH 版本**: 缺陷修复与文档完善，不改变契约。
- **IPC 协商**: 接收方读取消息头 `version` 字段，若版本高于本端支持则返回 `AIRY_ENOTSUP`，由发送方降级重试。
- **废弃流程**: 废弃接口保留 2 个 MAJOR 版本，标注 `@deprecated` 并提供迁移指引，期满后移除。

接口变更统一在 `kernel/include/uapi/` 头文件中体现，并通过接口评审（API Review）把关，确保与 8 子仓设计文档（[20-modules/README.md](../20-modules/README.md)）一致。

---

## 6. 相关文档

- [agentrt-linux 总览](../README.md)
- [模块设计](../20-modules/README.md)
- [架构设计](../10-architecture/01-system-architecture.md)
- [Airymax Unify Design 总纲](../10-architecture/10-unify-design.md)
- [IRON-9 v3 共享模型](../10-architecture/06-iron9-shared-model.md)
- [[DSL] 降级生存层](../10-architecture/11-degraded-survival-layer.md)
- [零拷贝 Ring Buffer 日志数据流](../40-dataflows/05-ring-buffer-logging.md)
- [需求分析](../00-requirements/03-non-functional-requirements.md)（NFR-P-001 调度延迟 / NFR-P-002 IPC 吞吐）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
