Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 接口设计

> **文档定位**： agentrt-linux（AirymaxOS） 接口设计层的总览与索引\
> **版本**： 0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**： 2026-07-06\
> **父文档**： [agentrt-linux 总览](../README.md)

---

## 1. 接口设计概览

agentrt-linux 接口设计层定义 8 子仓之间、内核与用户态之间、OS 与应用之间的全部接口契约。所有接口遵循"机制在内核、策略在用户态"的微内核原则，并与 agentrt 同源模块保持语义兼容。

接口分为 4 类：

| # | 接口类别 | 通信形态 | 同源 agentrt | 设计目标 |
|---|---------|---------|--------------|---------|
| 1 | 系统调用（syscall） | 用户态 → 内核态 | MicroCoreRT 调度 + AgentsIPC | Agent 任务管理、IPC、内存、调度、安全、认知的内核入口 |
| 2 | IPC 协议 | 进程 ↔ 进程 | AgentsIPC 128B 消息头 | io_uring 零拷贝消息传递 |
| 3 | SDK API | 应用 → OS | sdk 管理接口 | 4 语言统一封装的客户端 |
| 4 | 编码规范 | 代码级约定 | commons 公共工具 | 命名、风格、日志、Doxygen、安全编码 |

---

## 2. 接口分类矩阵

下表给出 4 类接口的详细分类、覆盖子仓与关键契约。

| 接口类别 | 子分类 | 覆盖子仓 | 关键契约 | 文档 |
|---------|--------|---------|---------|------|
| 系统调用 | agent 任务管理 | kernel / cognition | `AGENTRT_SYS_TASK_*` 调度类 SCHED_AGENT | [01-syscalls.md](01-syscalls.md) |
| 系统调用 | IPC | kernel / services | `AGENTRT_SYS_IPC_*` io_uring 零拷贝 | 01-syscalls.md |
| 系统调用 | 内存 | kernel / memory | `AGENTRT_SYS_ROVOL_*` 记忆卷载 | 01-syscalls.md |
| 系统调用 | 调度 | kernel | `AGENTRT_SYS_SCHED_*` sched_ext 策略 | 01-syscalls.md |
| 系统调用 | 安全 | kernel / security | `AGENTRT_SYS_CAP_*` capability 令牌 | 01-syscalls.md |
| 系统调用 | 认知 | kernel / cognition | `AGENTRT_SYS_CLT_*` CoreLoopThree kthread | 01-syscalls.md |
| IPC 协议 | 消息头 | 全部子仓 | 128B 定长 + 5 种 payload | [02-ipc-protocol.md](02-ipc-protocol.md) |
| IPC 协议 | 通信原语 | services | Channel / Socket / FIFO / Eventpair | 02-ipc-protocol.md |
| IPC 协议 | 零拷贝 | kernel / services | io_uring + page flipping | 02-ipc-protocol.md |
| SDK API | Python | 全部子仓 | `agentrt.AirymaxClient` 4 嵌套客户端 | [03-sdk-api.md](03-sdk-api.md) |
| SDK API | Rust | 全部子仓 | `agentrt::AirymaxClient` 4 嵌套客户端 | 03-sdk-api.md |
| SDK API | Go | 全部子仓 | `github.com/openairymax/agentrt-go` | 03-sdk-api.md |
| SDK API | TypeScript | 全部子仓 | `@openairymax/agentrt` | 03-sdk-api.md |
| 编码规范 | 命名 | 全部子仓 | `agentrt_` / `<service>_d` / `module_action_object()` | [04-coding-standard.md](04-coding-standard.md) |
| 编码规范 | C 风格 | kernel / security / memory | 4 空格 + snake_case + 1TBS + Doxygen | 04-coding-standard.md |
| 编码规范 | Rust 风格 | cognition / cloudnative | rustfmt + clippy + snake_case 函数 | 04-coding-standard.md |
| 编码规范 | 日志 | 全部子仓 | ANSI 颜色 + log_write() + CLOCK_REALTIME | 04-coding-standard.md |

### 2.1 接口覆盖范围

接口覆盖范围按子仓维度展开如下，每个子仓对外暴露的接口均落入上述 4 类之中：

| 子仓 | 系统调用 | IPC 协议 | SDK API | 编码规范 |
|------|---------|---------|---------|---------|
| kernel | `AGENTRT_SYS_TASK_*` / `IPC_*` / `ROVOL_*` / `SCHED_*` / `CAP_*` / `CLT_*` | 128B 消息头内核侧 | 不直接暴露 | C 风格 + Doxygen |
| services | 通过系统调用 + IPC | 128B 消息头用户态 + 4 通信原语 | daemon 客户端 | C 风格 + 日志 |
| security | `AGENTRT_SYS_CAP_*` / `LSM_*` | capability 携带 | SafetyClient | C 风格 + 安全编码 |
| memory | `AGENTRT_SYS_ROVOL_*` / `CXL_*` / `MGLRU_*` | MemoryRovol 迁移消息 | MemoryClient（嵌套） | C 风格 + 内存安全 |
| cognition | `AGENTRT_SYS_CLT_*` / `WASM_*` | CoreLoopThree 阶段通知 | CognitionClient + ChatClient | Rust 风格 |
| cloudnative | 不直接暴露 | gRPC + CRD | agentctl + SDK | Go 风格 |
| system | sysctl / procfs | 不直接暴露 | DevStation CLI | 多语言 |
| tests-linux | 不直接暴露 | 测试协议 | 测试框架 API | 各语言规范 |

---

## 3. 接口设计原则

接口设计严格遵循五维正交 24 原则中的 K-2 与 E-7（详见 [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md)）：

### 3.1 K-2 接口契约化

- **显式契约**: 所有接口以 C 头文件 + Doxygen 注释形式给出显式契约，包括参数语义、返回值、错误码、并发约束。
- **ABI 稳定**: 系统调用编号、IPC 消息头布局、capability 令牌格式在 MAJOR 版本内保持 ABI 稳定，破坏性变更必须升级 MAJOR 版本。
- **版本协商**: IPC 消息头携带 `version` 字段（当前 0x0100），支持协议版本协商。
- **错误码对齐**: 全部接口错误码对齐 `agentrt_errno.h`，与 agentrt 同源且部分代码共享（IRON-9 v2）。
- **并发约束显式**: 所有接口在 Doxygen 注释中显式声明线程安全性与可重入性（thread-safe / reentrant / async-signal-safe）。

### 3.2 E-7 文档即代码

- **头文件即文档**: C 头文件（`agentrt_syscalls.h`、`agentrt_ipc_msg.h`）即接口文档，Doxygen 注释直接生成 API 文档。
- **示例即规约**: SDK 文档中的 Python/Rust/Go/TypeScript 代码示例即为接口使用规约，可直接作为冒烟测试用例。
- **生成校验**: 通过 Doxygen 生成 HTML 文档 + 通过 SDK 示例生成冒烟测试，确保文档与代码同步。
- **接口评审**: 所有接口变更必须经过接口评审（API Review），检查契约一致性、命名规范、错误码规范。

### 3.3 接口设计补充原则

- **机制与策略分离**: 系统调用提供机制（如 `agentrt_sys_task_submit` 提交任务），调度策略由用户态 eBPF 程序定义。
- **最小权限**: 默认拒绝，通过 capability 显式授权（`agentrt_sys_capability_request`）。
- **零拷贝优先**: IPC 与内存接口优先使用 io_uring 零拷贝路径，减少数据复制。
- **同源兼容**: 与 agentrt 同源的接口保持 API 语义兼容，仅升级底层实现。

---

## 4. 本目录文档索引

| # | 文档 | 核心内容 | 行数级别 |
|---|------|---------|---------|
| 1 | [01-syscalls.md](01-syscalls.md) | 系统调用分类 + 编号规则 + C 接口 + 关键调用清单 + 性能约束 + 错误码 | 350-450 |
| 2 | [02-ipc-protocol.md](02-ipc-protocol.md) | 128B 消息头 + 5 种 payload + io_uring 零拷贝 + 同源映射 + 性能约束 | 350-450 |
| 3 | [03-sdk-api.md](03-sdk-api.md) | 4 语言矩阵 + 4 嵌套客户端 + Python/Rust/Go/TS 示例 + 错误处理 | 450-550 |
| 4 | [04-coding-standard.md](04-coding-standard.md) | 15 规范引用 + 命名规范 + C/Rust 风格 + 日志 + Doxygen + 安全编码 + 对照 | 350-450 |

---

## 5. 与 agentrt 接口的关系

agentrt-linux 接口与 agentrt 接口遵循 IRON-9 v2"同源且部分代码共享"原则：

| 维度 | agentrt 接口 | agentrt-linux 接口 | 关系 |
|------|-------------|---------------|------|
| 系统调用 | 无（用户态运行时） | `AGENTRT_SYS_*` 内核系统调用 | agentrt-linux 新增（OS 级） |
| IPC 消息头 | AgentsIPC 128B 定长 | 同源 128B 定长（字段对齐） | 语义同源，布局兼容 |
| IPC payload | 自定义协议 | 5 种 payload（REQUEST/RESPONSE/EVENT/STREAM/CONTROL） | agentrt-linux 扩展 |
| SDK | sdk（应用层） | agentctl + 4 语言 SDK | 同源语义，OS 级封装 |
| capability | 应用权限模型 | seL4 风格 capability 系统 | 同源语义，OS 级升级 |
| 调度 | MicroCoreRT 用户态调度 | SCHED_AGENT（sched_ext + eBPF） | 同源语义，内核态实现 |
| 记忆 | MemoryRovol 用户态 | MemoryRovol 内核态 | 同源语义，内核态升级 |
| 认知循环 | CoreLoopThree 用户态 | CoreLoopThree kthread | 同源语义，内核态加速 |

**同源红利**: agentrt 在 agentrt-linux 上运行时，IPC 消息头布局兼容、调度语义一致、capability 模型一致，无需适配层即可天然契合。具体表现：

- **IPC 兼容**: agentrt 发出的 128B 消息头可直接被 agentrt-linux io_uring 通道接收，无需协议转换。
- **调度兼容**: agentrt 的任务优先级语义（0-139）与 SCHED_AGENT 策略对齐，调度行为一致。
- **记忆兼容**: agentrt 的 MemoryRovol 用户态 API 与 agentrt-linux 内核态实现语义对齐，可平滑迁移。

**独立性**: agentrt-linux 接口为 OS 级接口（系统调用、内核 IPC、OS 级 SDK），与 agentrt 的应用级接口在分层上独立，不构成依赖耦合。当 agentrt 演进时，agentrt-linux 通过接口契约评审决定是否同步，避免被动跟随。独立性体现在：

- **分层独立**: agentrt-linux 接口位于 OS 层（系统调用 + 内核 IPC），agentrt 接口位于应用层（SDK + 用户态 IPC），二者通过契约解耦。
- **演进独立**: agentrt-linux 接口随 Linux 6.6 内核基线演进节奏，agentrt 接口随应用层需求演进，互不阻塞。
- **测试独立**: agentrt-linux 接口由 `airymaxos-tests-linux` 子仓验证（形式化 + Soak + 混沌），agentrt 接口由 agentrt 自有测试覆盖。

### 5.1 接口演进与版本协商

agentrt-linux 接口采用语义化版本 + ABI 兼容承诺的演进策略：

- **MAJOR 版本**: 不兼容的接口变更，需重新评审全部契约，触发 K-2 契约重签。
- **MINOR 版本**: 向后兼容的新增接口（新系统调用编号、新 IPC payload 类型），不影响既有调用方。
- **PATCH 版本**: 缺陷修复与文档完善，不改变契约。
- **IPC 协商**: 接收方读取消息头 `version` 字段，若版本高于本端支持则返回 `AGENTRT_ENOTSUP`，由发送方降级重试。
- **废弃流程**: 废弃接口保留 2 个 MAJOR 版本，标注 `@deprecated` 并提供迁移指引，期满后移除。

接口变更统一在 `airymaxos-kernel/include/uapi/` 头文件中体现，并通过接口评审（API Review）把关，确保与 8 子仓设计文档（[20-modules/README.md](../20-modules/README.md)）一致。

---

## 6. 相关文档

- [agentrt-linux 总览](../README.md)
- [模块设计](../20-modules/README.md)
- [架构设计](../10-architecture/01-system-architecture.md)
- [需求分析](../00-requirements/03-non-functional-requirements.md)（NFR-P-001 调度延迟 / NFR-P-002 IPC 吞吐）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
