Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 兼容性设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）兼容性工程体系主索引——ABI 稳定性、POSIX 兼容、上游跟踪与跨发行版兼容\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[AirymaxOS 总览](../README.md)

---

## 1. 模块概述

`160-compatibility/` 是 AirymaxOS 文档体系中**系统长期演进的工程保障**。它继承 Linux 内核 30+ 年沉淀的兼容性哲学（用户空间 ABI 永不破坏 + 内核内部 API 不保证稳定 + KABI 内核符号兼容性），并在其上扩展智能体操作系统专属的 AgentsIPC 协议兼容性、SDK 跨版本兼容性、Agent 应用迁移兼容性等。

兼容性是操作系统的生命线。AirymaxOS 作为智能体操作系统发行版，必须承诺：在 0.1.1 上编写的 Agent 应用，到 1.0.1、3.0 LTS、5.0 LTS 上仍能正常运行；在 1.0.1 上开发的 Agent 应用，能通过兼容层在 0.1.1 上降级运行。这种双向兼容承诺是 IRON-9 v3 四层模型（[SC]/[SS]/[IND]/[DSL]）的延伸——agentrt 应用级接口与 AirymaxOS OS 级接口同源且部分代码共享演进。

本模块承担五项核心职责：

1. **ABI 稳定性**：用户空间 ABI 永不破坏（OS-IRON-001）+ KABI 内核符号白名单管理。
2. **POSIX 兼容**：POSIX 兼容性 + sched_tac 调度约束下的应用兼容。
3. **上游跟踪**：Linux 6.6 上游跟踪策略 + backport 规范。
4. **IPC 版本协商**：AgentsIPC 协议版本演进 + 运行时协商 + 降级策略。
5. **跨发行版兼容**：与主流 Linux 发行版（Ubuntu / Debian / Fedora / RHEL）的二进制兼容性。

---

## 2. 技术选型声明

agentrt-linux v1.0 在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度做出**不可妥协**的技术选型。本目录所有兼容性设计必须以本声明为基线——兼容性承诺不得引入 sched_ext / page flipping / BPF LSM / DMA 一致性内存的兼容支持，这些被弃用方案不在兼容矩阵内。

| # | 技术维度 | 选定方案 | 明确不采用 | 选定理由 |
|---|---------|---------|-----------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类，通过 `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，**不定义新的调度类宏** | **不使用 sched_ext**（不引入 eBPF 调度器、不依赖 `SCHED_EXT=7`、不使用 SCHED_AGENT 宏） | sched_ext 实验性语义漂移，不在兼容矩阵内；sched_tac 基于 Linux 6.6 主线调度类，ABI 永久稳定 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输，registered buffer + mmap 共享页 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | page flipping 已废弃，不提供向后兼容；IORING_OP_URING_CMD 在 Linux 6.6 已稳定 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 Linux Security Module（`airy_lsm`），通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | BPF LSM 验证器语义不确定，不纳入兼容承诺；纯 C LSM 行为可形式化验证 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `remap_pfn_range` 映射到用户态地址空间 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | DMA 一致性内存跨架构行为不一致，不纳入兼容承诺；alloc_pages + mmap 跨架构语义一致 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | v2 三层模型（升级为 v3，新增 [DSL] 降级生存层） | [DSL] 降级生存层确保 [SC] 损坏时兼容性，兼容层类型安全 |

### 2.1 内核基线兼容声明（本模块专属）

> **标准 Linux 6.6 为唯一原生基线**：agentrt-linux v1.0 以 Linux 6.6 LTS 为唯一原生开发与兼容基线，sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap 全部基于 Linux 6.6 主线能力。
>
> **其余内核版本仅作 DKMS 兼容目标**：对 Linux 6.1 / 6.12 等其他版本，仅通过 DKMS（Dynamic Kernel Module Support）提供有限兼容，不承诺完整功能与性能 SLO。sched_ext / page flipping / BPF LSM / DMA 一致性内存等被弃用方案**不在任何版本的兼容目标内**。

---

## 3. 文档索引

`160-compatibility/` 由 6 个文档构成（含本 README）：

```
160-compatibility/
├── README.md                       # 本文件 — 兼容性主索引（v1.0）
├── 01-abi-stability.md             # 用户空间 ABI 稳定性 + KABI 保留域（v1.0）
├── 02-posix-compat.md              # POSIX 兼容性 + sched_tac 调度约束（v1.0）
├── 03-upstream-tracking.md         # 上游跟踪策略 + backport 规范（v1.0）
├── 04-ipc-versioning.md            # AgentsIPC 版本协商 + 操作码版本化（v1.0）
└── 05-cross-distro.md             # 跨发行版兼容性 + DKMS 兼容目标（v1.0）
```

### 3.1 各文档定位

| 文档 | 核心问题 | 主要产物 |
|------|---------|---------|
| README.md | 兼容性体系全貌？ | 6 层分层 + 4 层接口稳定性分级 + 技术选型声明 |
| 01-abi-stability.md | ABI 怎么保证永不破坏？ | UAPI 稳定性 + KABI 保留域 + OS-IRON-001 |
| 02-posix-compat.md | POSIX 怎么兼容？ | POSIX 兼容性 + sched_tac 调度约束 |
| 03-upstream-tracking.md | 上游怎么跟踪？ | Linux 6.6 上游跟踪 + backport 规范 |
| 04-ipc-versioning.md | IPC 协议怎么演进？ | 协议版本演进 + 运行时协商 + 降级策略 |
| 05-cross-distro.md | 跨发行版怎么兼容？ | 发行版支持矩阵 + glibc 兼容 + DKMS 兼容目标 |

### 3.2 兼容性分层

| 层级 | 类型 | 范围 | 承诺 |
|------|------|------|------|
| L1 | 用户空间 ABI | 系统调用 + ioctl + procfs + sysfs | 永不破坏（OS-IRON-001） |
| L2 | 内核 KABI | EXPORT_SYMBOL 导出符号 | 主版本内兼容 |
| L3 | AgentsIPC 协议 | 128B 消息头 + 5 种 payload | 语义版本化演进 |
| L4 | SDK API | Python / Rust / Go / TS | 语义版本号 |
| **L5** | **Agent 应用迁移** | **AirymaxOS 专属** | **0.1.1 ↔ 1.0.1 双向** |
| **L6** | **跨发行版兼容** | **AirymaxOS 专属** | **与主流 Linux 发行版** |

### 3.3 4 层接口稳定性分级

| 层级 | 接口类型 | 稳定性 | 变更流程 |
|------|---------|--------|---------|
| L1 | Agent 应用 API | 极稳定 | RFC + ABI 审查 + 6 个月宽限期 |
| L2 | AgentsIPC 协议 | 中等稳定 | 季度评审 + 兼容性测试 |
| L3 | 内核子系统 API | 可重构 | 补丁序列修复所有调用点 |
| L4 | 内部实现 | 完全自由 | 无约束 |

### 3.4 跨发行版兼容矩阵

| 发行版 | 二进制兼容 | glibc 版本 | 说明 |
|--------|-----------|-----------|------|
| Ubuntu 22.04+ | ✓ | 2.35+ | 原生支持 |
| Debian 12+ | ✓ | 2.36+ | 原生支持 |
| Fedora 36+ | ✓ | 2.35+ | 原生支持 |
| RHEL 9+ | ✓ | 2.34+ | 原生支持 |
| 其他内核版本 | △（DKMS） | — | 仅 DKMS 有限兼容，不承诺完整 SLO |

---

## 4. Airymax Unify Design 映射

Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）在兼容性体系中的关系如下。兼容性是 IRON-9 v3 [SC] 共享契约 ABI 稳定性与 [IND] 独有兼容层的主要落地维度。

| Unify 模块 | 全称 | 在本目录的体现 | IRON-9 v3 层次 |
|-----------|------|--------------|---------------|
| **A-UEF** | Unified Error and Fault Framework | [SC] `error.h` 错误码 ABI 稳定性（码空间值域永不复用）、SDK 错误码跨版本兼容 | **[SC]** 共享契约 ABI 稳定性 |
| **A-IPC** | Unified Airymax IPC Fabric | AgentsIPC 128B 消息头协议版本协商、操作码版本化、运行时降级 | **[SC]** 共享契约 ABI 稳定性 |
| **A-ULP** | Unified Logging and Printk Subsystem | [SC] `log_types.h` 128B 日志记录格式稳定性、日志级别枚举兼容 | **[SC]** 共享契约 ABI 稳定性 |
| **A-UCS** | Unified Configuration Subsystem | 配置 sysctl/JSON 语义同源兼容、`AIRY_CONFIG_VERSION` 版本校验 | **[SS]** 语义同源兼容 |
| **A-ULS** | Unified Lifecycle Supervision Framework | Agent 8 态生命周期 ABI 稳定性、[SC] `sched.h` 调度参数契约稳定 | **[SC]** 共享契约 ABI 稳定性 |

### 4.1 IRON-9 v3 [SC] 共享契约 ABI 稳定性

兼容性体系的核心是 IRON-9 v3 **[SC] 共享契约层**的 ABI 稳定性承诺。10 个 [SC] 头文件（`error.h`/`log_types.h`/`ipc.h`/`sched.h`/`memory_types.h`/`security_types.h`/`cognition_types.h`/`syscalls.h`/`uapi_compat.h`/`lsm_types.h`）的物理宿主为 `kernel/include/uapi/linux/airymax/`，两端逐字节相同，通过 `sc-dual-ci.yml` 双端校验：

- **错误码值永不复用**：`AIRY_E*` 一旦分配，值域永不回收（[-300, -1] 五子空间）
- **128B 消息头布局固定**：IPC/日志 128B 记录布局一经发布永不改变
- **syscall 编号永不复用**：512-631 段编号一旦分配，永不回收

### 4.2 IRON-9 v3 [IND] 独有兼容层

agentrt 与 agentrt-linux 的兼容性差异落在 **[IND] 独立实现层**：

| 维度 | agentrt（[IND]） | agentrt-linux（[IND]） |
|------|-----------------|---------------------|
| ABI 机制 | 用户态库 ABI（`airy_*` API） | OS 级 ABI（`AIRY_SYS_*` 系统调用 + KABI） |
| IPC 兼容 | POSIX MQ + mmap 跨平台抽象 | io_uring + IORING_OP_URING_CMD |
| 调度兼容 | MicroCoreRT 用户态调度 | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF） |
| 安全兼容 | 应用权限模型 | 纯 C LSM + Capability 系统 |

两端通过 [SC] 共享契约保持互操作，通过 [DSL] 降级生存保证多语言类型一致，[IND] 各自独立演进兼容性策略。

### 4.3 [DSL] 降级兼容

IRON-9 v3 的 [DSL] 降级生存层为兼容性提供最后防线：当 [SC] 头文件损坏/版本不匹配时，`#ifdef AIRY_SC_FALLBACK` 降级块提供最小可运行子集（38 个 POSIX 码 + 最简 128B 消息头 + EEVDF 默认调度），确保系统降级可运行而非 brick。

---

## 5. 相关文档

### 5.1 上级与架构文档
- [AirymaxOS 总览](../README.md) —— 文档体系顶层纲领（v1.0）
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（五模块 SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 四层模型（[SC]/[SS]/[IND]/[DSL]）

### 5.2 接口与工程标准文档
- [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md) —— 系统调用接口（ABI 稳定性）
- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— AgentsIPC 协议（版本协商）
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— A-UEF [SC] error.h 契约
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— A-ULP [SC] log_types.h 契约
- [50-engineering-standards/01-coding-standards.md](../50-engineering-standards/01-coding-standards.md) —— ABI 编码规范
- [50-engineering-standards/11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md) —— [SC] 层内部类型桥接（通过 uapi_compat.h）

### 5.3 关联模块文档
- [120-development-process/README.md](../120-development-process/README.md) —— 开发流程（稳定版维护）
- [190-distribution/README.md](../190-distribution/README.md) —— 发行版管理（版本号 + RPM）
- [140-application-development/07-syscall-registry.md](../140-application-development/07-syscall-registry.md) —— 系统调用编号 SSoT

### 5.4 参考材料
- Linux 6.6 `Documentation/ABI/`（ABI 文档）
- Linux 6.6 KABI 子系统
- 语义化版本规范（SemVer）
- agentrt ABI 稳定性规范

---

## 6. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，5/5 文档完成（ABI + POSIX + 上游 + IPC 版本 + 跨发行版） |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）；IORING_OP_URING_CMD（不使用 page flipping）；纯 C LSM（不使用 BPF LSM）；alloc_pages + mmap（不使用 DMA 一致性内存）；IRON-9 v3 四层模型（新增 [DSL] 降级生存层）；新增内核基线兼容声明（标准 Linux 6.6 为唯一原生基线，其余版本仅作 DKMS 兼容目标）；新增 Airymax Unify Design 五模块映射（[SC] 共享契约 ABI 稳定性 / [IND] 独有兼容层） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | 兼容性模块 v1.0 | 共 6 文档 | 维护者：开源极境工程与规范委员会 | 兼容性是操作系统的生命线
