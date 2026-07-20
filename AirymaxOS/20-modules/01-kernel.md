Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 内核设计文档
> **文档定位**：agentrt-linux 内核设计文档（Airymax Kernel 极境内核）\
> **文档版本**：v1.1 2026-07-18\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **核心约束**：IRON-9 v3 四层共享模型（\[SC] 共享契约层 + \[SS] 语义同源层 + \[IND] 完全独立层 + \[DSL] 降级生存层）——与 agentrt 用户态 atoms/corekern 通过 \[SC] 共享契约层（10 头文件）+ \[SS] 语义同源层 + \[DSL] 降级生存层协作，\[IND] 完全独立层承载内核态 sched_tac（基于 Linux 6.6 原生调度类组合 SCHED_DEADLINE/SCHED_FIFO/EEVDF + cgroup cpuset 隔离 + seL4 MCS 语义映射的策略框架，非用户态调度器，非 sched_ext）/io\_uring/eBPF/Rust 实现。**Syscall 架构**（v1.1 Capability Folding 后）：4 核心 + 20 预留 = 24 槽位（`airy_sys_call` sec_d 专属管理入口 + 3 控制原语，IPC 数据面完全由 io\_uring `IORING_OP_URING_CMD` 承载，fastpath C-S9 内联 Badge 校验）。**命名统一**：所有 \[SC] 共享标识符使用 `AIRY_*` 前缀，C 函数使用 `airy_sys_*` 前缀。\
> **子仓编号**：01\
> **子仓代号**：极境内核（Airymax Kernel）\
> **设计基准**：Linux 内核基线 + 微内核化改造\
> **同源 agentrt**：atoms/corekern（MicroCoreRT）\
> **理论基础**：seL4 微内核工程思想\
> **技术路线**：基于 Linux 内核微内核化改造，非从零开发微内核\
> **横切关注点**：内核是横切关注点（cross-cutting concern），贯穿调度、IPC、eBPF、记忆卷载 4 大数据流，提供机制骨架

***

## 目录

- [1. 子仓职责与设计哲学](#1-子仓职责与设计哲学)
- [2. 同源关系（IRON-9 v3 四层共享模型）](#2-同源关系iron-9-v3-四层共享模型)
- [3. 目录结构](#3-目录结构)
- [4. 内核对象模型（seL4 借鉴）](#4-内核对象模型sel4-借鉴)
- [5. Capability 系统（seL4 借鉴）](#5-capability-系统sel4-借鉴)
- [6. IPC 消息传递（seL4 借鉴 + io\_uring 落地）](#6-ipc-消息传递sel4-借鉴--io_uring-落地)
- [7. 调度与线程模型](#7-调度与线程模型)
- [8. 内存管理（seL4 借鉴 + Linux 对接）](#8-内存管理sel4-借鉴--linux-对接)
- [9. 核心特性（Linux 6.6 原生）](#9-核心特性linux-66-原生)
- [10. IRON-9 v3 四层共享模型落地](#10-iron-9-v3-四层共享模型落地)
- [11. Boot 流程](#11-boot-流程)
- [12. 错误处理与形式化验证预留](#12-错误处理与形式化验证预留)
- [13. agentrt-linux 工程基线与版本规划](#13-agentrt-linux-工程基线与版本规划)
- [14. 与其他子仓的协作](#14-与其他子仓的协作)
- [15. 里程碑（M0-M8）](#15-里程碑m0-m8)
- [16. agentrt 一致性检查](#16-agentrt-一致性检查)
- [17. 相关文档](#17-相关文档)
- [18. 参考文献](#18-参考文献)

***

## 1. 子仓职责与设计哲学

`kernel` 是 agentrt-linux（AirymaxOS）的内核子仓，承担以下核心职责：

1. **Linux 6.6 内核维护 \[IND]**：基于 Linux 6.6 内核基线（1.x.x 版本锁定，ADR-013），保持与上游社区同步演进。
2. **微内核化改造 \[IND]**：在保留 Linux 6.6 完整能力的前提下，遵循 Liedtke minimality principle（ES-SEL4-01, 04），将 VFS、网络栈、设备驱动等子系统逐步用户态化，最小化特权态代码体积。
3. **Agent 感知调度（sched_tac）\[SS]**：采用sched_tac（基于 Linux 6.6 原生调度类组合 SCHED_DEADLINE/SCHED_FIFO/EEVDF + cgroup cpuset 隔离 + seL4 MCS 语义映射的策略框架，非用户态调度器，非 sched_ext）实现 stc_agent 等策略枚举（用户态策略 daemon），允许用户态定义调度策略。任务描述符与优先级语义 \[SC] 与 agentrt 共享。
4. **高性能 IPC 基础（io\_uring）\[SS]**：基于 io\_uring 构建零 syscall、零拷贝的消息传递基础设施。IPC 消息头与操作码语义 \[SC] 与 agentrt 共享。
5. **eBPF 可编程扩展 \[SS]**：提供 struct\_ops 注册机制 + kfunc 扩展 + ringbuf 上报，可观测/网络/安全/调度均可通过 eBPF 编程。struct\_ops 状态机与 common\_value 布局与 agentrt 共享（补充共享文件 `bpf_struct_ops.h`，非 \[SC] 核心头文件）。
6. **Rust 安全驱动 \[IND]**：依托 Linux 6.6 中 Rust 实验性支持（持续演进中），构建安全驱动开发框架。
7. **\[SC] 共享契约层物理宿主 \[IND]**：IRON-9 v3 全部 10 个 \[SC] 共享契约头文件物理宿主在 `kernel/include/uapi/linux/airymax/`，其他子仓通过 `-I` 引用确保单一数据源。
8. **同源传承 \[SS]**：从 agentrt 的 atoms/corekern（MicroCoreRT）继承实时性与微核心设计思想。

### 1.1 Liedtke 极简原则的 agentrt-linux 诠释

**Jochen Liedtke**（L4 微内核创始人）提出微内核设计的核心原则（ES-SEL4-01）：

> "A concept is tolerated inside the microkernel only if moving it outside the kernel, i.e., permitting competing implementations, would prevent the implementation of the system's required functionality."

**翻译**：只有当某个概念移出微内核会导致系统无法实现所需功能时，才容忍它留在微内核内。

**seL4 的工程验证**：seL4 严格遵循此原则，真正的"微内核必需代码"约 **1.44 万行 C**（不含 arch/plat/drivers），其中五大内核对象（TCB + CNode + Endpoint + Notification + Untyped）合计仅约 **4,353 行**。这与 Linux 单体内核约 3,000 万行形成鲜明对比——差距约 2,000 倍。

**agentrt-linux 的诠释**：agentrt-linux 不是从零开发微内核（ADR-012），而是基于 Linux 6.6 内核进行微内核化改造。因此"极简"不意味着将 Linux 缩减到 1.44 万行，而是遵循以下体量控制策略：

| 维度          | seL4 基线   | agentrt-linux 预算      | 控制策略                  |
| ----------- | --------- | --------------------- | --------------------- |
| 内核核心代码      | \~1.44 万行 | 5-10 万行（含 agent 调度原语） | 每新增子系统需论证"无法在用户态安全实现" |
| 微内核化改造补丁    | —         | 控制在 2 万行以内            | VFS / 网络栈 / 驱动用户态化补丁  |
| \[SC] 共享契约层 | —         | 10 个头文件（IRON-9 v3）     | 单一物理宿主，禁止重复定义         |

### 1.2 内核代码体量预算

借鉴 seL4 的体量控制哲学（ES-SEL4-01），agentrt-linux 设定以下代码体量预算：

| 代码区域                         | seL4 行数  | agentrt-linux 预算 | 性质              |
| ---------------------------- | -------- | ---------------- | --------------- |
| 内核核心（调度/IPC/capability/内存原语） | \~14,400 | 5-10 万行          | 真正的"微内核化"必需     |
| 微内核化改造补丁（VFS/网络/驱动用户态化）      | —        | ≤ 2 万行           | 渐进式改造           |
| \[SC] 共享契约层                  | —        | 10 个头文件           | IRON-9 v3 单一数据源 |
| \[SS] 语义同源层                  | —        | 30+ 项高层 API 语义   | 实现独立，语义同源       |
| \[IND] 独立层                   | —        | 15+ 项            | 内核态专属           |

**体量审计机制**：每版本发布前执行代码体量审计，消除"蠕变增胖"（kernel bloat）。新增子系统需向 TSC 提交"无法在用户态安全实现"论证报告。

### 1.3 系统调用数量约束

**seL4 基线**（ES-SEL4-02）：seL4 内核仅 7 个系统调用（非 MCS）或 11 个（MCS）：

| 系统调用             | 用途              |
| ---------------- | --------------- |
| `seL4_Call`      | 同步调用（发送 + 等待回复） |
| `seL4_ReplyRecv` | 回复并接收下一条        |
| `seL4_Send`      | 同步发送（不等待）       |
| `seL4_NBSend`    | 非阻塞发送           |
| `seL4_Recv`      | 等待接收            |
| `seL4_Reply`     | 仅回复             |
| `seL4_Yield`     | 让出 CPU          |

**v1.1 目标**：**4 核心 + 20 预留 = 24 槽位**（Capability Folding 后，见 [01-syscalls.md](../30-interfaces/01-syscalls.md)），SSoT 定义于 `syscalls.h`：

| 编号    | 调用名                   | 分类     | 说明                                                |
| ----- | --------------------- | ------ | ------------------------------------------------- |
| 0     | `airy_sys_call`       | Capability Invocation | sec_d 专属管理入口（Badge 编译/撤销 + LSM_ctl + Wasm_load） |
| 1     | `airy_sys_rovol_ctl`  | 控制原语   | 记忆卷载控制                                            |
| 2     | `airy_sys_sched_ctl`  | 控制原语   | 调度策略配置                                            |
| 3     | `airy_sys_clt_notify` | 控制原语   | CoreLoopThree 通知 + kthread 注册                     |
| 4-23  | 预留                    | —      | 未来扩展                                              |

**数据面 I/O 完全由 io\_uring 处理（零 syscall）**。v1.1 Capability Folding 后，8 个 seL4 风格 IPC 原语（send/recv/nbsend/nbrecv/reply_recv/yield/reply/notify）全部移除，IPC 数据传递完全由 io\_uring `IORING_OP_URING_CMD` 承载。`airy_sys_call` 语义收窄为 sec_d 专属管理入口（Badge 编译/撤销/LSM_ctl/Wasm_load）。详见 [01-syscalls.md](../30-interfaces/01-syscalls.md) §2.2。

**syscall 审计流程**：新增 syscall 需 TSC 评审，遵循"机制在内核、策略在用户态"原则。推荐采用 seL4 的 syscall.xml 单一来源管理（ES-SEL4-21），对应 R-01 增强建议（纳入 1.0.1 M1 阶段）。

### 1.4 Capability Invocation 统一入口

`airy_sys_call` 是统一的 capability invocation 入口，借鉴 seL4 的 `decodeInvocation` 模式。操作类型由 **capability 类型**决定，而非 syscall 编号：

```c
/* Unified capability invocation: seL4 Call style */
int ret = airy_sys_call(cap, &msg);
if (ret < 0) {
    log_write(LOG_ERROR, "cap invocation failed: errno=%d", ret);
    return ret;
}
```

- 任务提交/取消/状态查询 → task capability invocation
- Capability 派生/撤销 → CNode capability invocation
- LSM 安全策略加载 → security capability invocation
- Wasm 模块加载 → module capability invocation
- IPC 同步调用 → endpoint capability invocation

### 1.5 最小完备集原则

借鉴 seL4 的极简主义（ES-SEL4-01），agentrt-linux 内核设计遵循体系并行的“最小完备集"原则：

- **Syscall**：v1.1 后 4 核心 = 1 Capability Invocation（sec_d 专属管理）+ 3 控制原语（rovol/sched/clt），IPC 数据传递零 syscall（io_uring 承载），无冗余
- **IPC opcode**：4 个（SEND/RECV/SEND\_BATCH/CANCEL），无冗余
- **Capability 操作**：7 个（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate），与 seL4 CNode 对齐
- **struct\_ops 状态**：4 个（INIT/REGISTERED/ACTIVE/DRAINING），无冗余
- **错误码**：13 个（-1 到 -12 + 0），无冗余
- **MemoryRovol 层级**：4 层（L1-L4），无冗余

每个功能点必须论证"无法在用户态安全实现"方可纳入内核。目标是 bug 最少——代码越少，bug 越少。

### 1.6 横切关注点声明

内核是横切关注点（cross-cutting concern），机制骨架贯穿 agentrt-linux 全部 4 大数据流：

| 数据流          | 内核切入点                                                                  | 同源标注   |
| ------------ | ---------------------------------------------------------------------- | ------ |
| 调度数据流        | sched_tac 策略框架 + stc_agent 策略枚举 + sub-scheduler                      | \[SS]  |
| IPC 数据流（控制面） | v1.1 Capability Folding 后控制面已折叠：4 核心 syscall（`airy_sys_call` sec_d 专属管理 + 3 控制原语），原 8 个 seL4 风格 IPC 原语已全部移除 | \[SS]  |
| IPC 数据流（数据面） | io\_uring 零拷贝 ring + registered buffer + MSG\_RING                     | \[SS]  |
| eBPF 数据流     | struct\_ops 注册 + kfunc + ringbuf + verifier                            | \[SS]  |
| 记忆卷载数据流      | userfaultfd + MGLRU + CXL bus（与 memory 协作）                   | \[IND] |

***

## 2. 同源关系（IRON-9 v3 四层共享模型）

依据 IRON-9 v3 决策，agentrt（用户态 atoms/corekern）与 agentrt-linux（内核态 kernel）通过四层共享模型协作（\[SC] 共享契约 + \[SS] 同源签名 + \[IND] 独立 + \[DSL] 降级生存）：

| 层次               | 共享程度               | 内核子系统内容                                                                                                                                                                                                                                                                                 | 组织方式                              |
| ---------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| **\[SC] 共享契约层**  | 完全共享代码             | 10 个头文件（详见 §10.1）：error.h / log_types.h / ipc.h / sched.h / memory_types.h / security_types.h / cognition_types.h / syscalls.h / uapi_compat.h / lsm_types.h                                                                                                                                                                                | `kernel/include/uapi/linux/airymax/`（单一物理宿主，每个头文件底部含 \[DSL] 降级块） |
| **\[SS] 语义同源层**  | 高层 API 语义同源，签名独立演进 | sched_tac 25+ 调度回调语义、io\_uring ring 创建/提交/完成/注册、MSG\_RING 跨环消息、SQPOLL 状态机、DEFER\_TASKRUN、eBPF struct\_ops 注册、bpf\_prog 生命周期、bpf\_link 生命周期、bpf\_map\_ops 回调表、ringbuf reserve/submit、kfunc 注册模式 等 30+ 项                                                                                 | 各自独立实现                            |
| **\[IND] 完全独立层** | 完全独立               | 策略守护进程注册、策略守护进程 enable/disable、调度上下文追踪、fallback 回退机制、cgroup 集成、core-sched 集成、debug dump；io-wq 工作队列、NO\_MMAP、REGISTERED\_FD\_ONLY、URING\_CMD；JIT 后端、trampoline 本机码生成、verifier 实现、CFS 钩子（不引入）、cfi\_stubs、KABI\_RESERVE（不采用）；VFS/网络/驱动用户态化改造；Rust 驱动框架 | 各自独立仓库                            |
| **\[DSL] 降级生存层** | \[SC] 损坏时最小可运行子集             | 每个 \[SC] 头文件底部 `#ifdef AIRY_SC_FALLBACK` 降级块（38 POSIX 错误码 + `AIRY_ECFGVERSION`、printk 原生日志、最简 128B IPC、EEVDF 默认调度、仅 POSIX capability、统一 Panic）                                                                                                                                  | 自包含，不依赖 \[SC] 其他符号                  |

### 2.1 维度对比

| 维度             | agentrt（atoms/corekern）         | agentrt-linux（kernel）               | 同源标注   |
| -------------- | ------------------------------- | --------------------------------------------- | ------ |
| 设计目标           | RT 微核心 + Agent 调度               | Linux 6.6 + 微内核化 + Agent 调度                   | \[SS]  |
| 调度模型           | MicroCoreRT 实时调度                | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF 原生调度类 + seL4 MCS 语义映射）               | \[SS]  |
| 任务描述符          | `struct airy_task_desc`（用户态）    | `struct airy_task_desc`（内核态）                  | \[SC]  |
| IPC            | 用户态消息队列                         | io\_uring 零拷贝 IPC（数据面，承载 C-S9 fastpath Badge 内联校验）+ 4 核心 syscall（控制面：`airy_sys_call` sec_d 专属管理 + 3 控制原语） | \[SS]  |
| IPC 消息头        | `struct airy_ipc_msg_hdr`（128B） | `struct airy_ipc_msg_hdr`（128B）               | \[SC]  |
| eBPF           | 用户态策略引擎                         | struct\_ops + kfunc + ringbuf                 | \[SS]  |
| struct\_ops 状态 | `airy_struct_ops_state`         | `airy_struct_ops_state`                       | \[SC]  |
| 安全模型           | Cupolas capability              | seL4 风格 capability + LSM                      | \[SC]  |
| 内存模型           | heapstore + memoryrovol         | MemoryRovol + CXL + PMEM                      | \[SC]  |
| Syscall 编号     | `AIRY_SYS_*` 宏定义                | `syscall_64.tbl` 新增段                          | \[SC]  |
| 驱动模型           | 用户态驱动 + capability              | Rust 驱动 + VFIO + capability                   | \[IND] |
| 跨平台            | Linux/macOS/Windows             | Linux 6.6 专属                                  | \[IND] |

### 2.2 同源传承要点

- 保留 agentrt 的"微核心"哲学（最小化运行时核心）\[SS]。
- 保留 agentrt 的"实时性"目标（通过sched_tac 的 sub-scheduler 在 cgroup 上附加实时策略）\[SS]。
- 保留 agentrt 的"消息传递"通信范式（升级为 io\_uring 零拷贝实现，fastpath C-S9 Badge 内联校验）+ `airy_sys_call` sec_d 专属管理入口 \[SS]。
- 任务描述符与优先级语义 \[SC] 共享，确保两端调度语义一致。
- IPC 消息头与操作码 \[SC] 共享，确保两端通信协议一致。
- struct\_ops 状态机 \[SC] 共享，使 agentrt 用户态可解析 sched_tac 策略守护进程在线状态（复用状态枚举）。
- capability 派生模型 \[SC] 共享（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate），确保两端安全语义一致。
- MemoryRovol L1-L4 数据结构 \[SC] 共享，确保两端记忆卷载语义一致。
- Syscall 编号体系 \[SC] 共享（syscalls.h），确保两端 syscall 语义一致。

***

## 3. 目录结构

```
kernel/                         # kernel 子仓本身是 Linux 6.6 完整 fork（Model A）
│                               # 含完整 Linux 6.6 源码树（~60K 文件 1.6GB，含 fs/ net/ block/ drivers/ mm/
│                               # security/ sound/ certs/ ipc/ samples/ usr/ virt/ 等全部上游子系统）[IND]
│                               # Airymax 修改直接写入对应上游子系统目录（如 mm/airymax_mm.c、kernel/sched/airy_sched.c、
│                               # security/airy/airy_lsm.c），不使用 patches/ 隔离层
│
├── airy_core.c                 # [ES-SEL4-1] 最小内核模块入口（seL4 kernel_all.c 思想）[IND]
│                               # 聚合 corekern/ + superv/ + log/ + ipc/ + config/ 的构建单元
├── corekern/                   # Airymax 微核心抽象层 [IND]（不替代上游 kernel/）
│   ├── sched/                  # sched_tac 策略守护进程源码（用户态策略 daemon，非用户态调度器）[SS]
│   │   ├── airymax_agent_sched.c # stc_agent 策略（sched_tac 落地）[SS]
│   │   ├── core_sched.c
│   │   └── ext.c
│   ├── ipc/                    # 基于 io_uring 的 IPC 优化 [SS]
│   │   ├── io_uring_ipc.c      # 注册固定 OP，支持零拷贝消息传递 [SS]
│   │   ├── kfifo_ipc.c         # kthread 间 kfifo IPC [SS]
│   │   └── fastpath.c          # IPC fastpath 优化（ES-SEL4-4 借鉴）[SS]
│   ├── bpf/struct_ops/         # eBPF struct_ops 扩展 [SS]
│   │   └── airymax_sched_ops.c # struct_ops 注册机制扩展 [SS]
│   ├── object/                 # capability 系统内核侧接口（seL4 借鉴 ES-SEL4-05~09）[SS]
│   │   ├── cte.c               # CTE（Capability Table Entry）管理 [SS]
│   │   ├── cspace.c            # CSpace 树形寻址（guard + radix）[SS]
│   │   ├── mdb.c               # MDB（Memory Disclosure Base）派生树维护 [SS]
│   │   └── cnode_ops.c         # 7 种 CNode 操作（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate）[SS]
│   ├── taskflow/               # 任务流原语 [SS]
│   ├── memory/                 # 内存原语接口（实现移入 memory 子仓）[IND]
│   ├── time/                   # 时间原语 [IND]
│   ├── locking/                # 锁机制与同步原语 [IND]
│   └── irq/                    # 中断处理 [IND]
├── superv/                     # [A-ULS] Micro-Supervisor 内核冷酷执法层（纯 C LSM，不使用 BPF LSM）[IND]
├── log/                        # [A-ULP] 统一日志与打印系统（内核侧）[IND]
├── ipc/                        # [A-IPC] IORING_OP_URING_CMD 回调入口（数据面载体，不使用 page flipping）[IND]
├── config/                     # [A-UCS] 统一配置管理体系（内核侧）[IND]
├── arch/                       # 架构相关代码（ES-OLK-2 + ES-SEL4-7，x86/arm64/riscv）[IND]
├── include/
│   ├── uapi/airymax/           # 用户空间 ABI（永久稳定，OS-IRON-001）[IND]
│   └── airymax/                # [SC] 共享契约层 10 头文件物理宿主（单一数据源，每个底部含 [DSL] 降级块）
│       ├── error.h             # 错误码 + 故障码 [SC] #1
│       ├── log_types.h         # 日志类型契约（128B 记录 + 5 级枚举）[SC] #2
│       ├── ipc.h               # IPC 契约（128B 消息头 + magic 'ARE1'）[SC] #3
│       ├── sched.h             # 调度契约（任务描述符 + vtime + 优先级）[SC] #4
│       ├── memory_types.h      # 记忆卷载契约（与 memory 子仓共享）[SC] #5
│       ├── security_types.h    # 安全契约（与 security 子仓共享）[SC] #6
│       ├── cognition_types.h   # 认知契约（与 cognition 子仓共享）[SC] #7
│       ├── syscalls.h          # Syscall 编号体系 [SC] #8
│       ├── uapi_compat.h       # 三路类型桥接（__KERNEL__/__linux__/#else）[SC] #9
│       └── lsm_types.h         # 纯 C LSM 类型契约（DEFINE_LSM(airy) 骨架）[SC] #10
├── configs/                    # AirymaxOS 内核配置 [IND]
│   ├── airy_defconfig          # 默认配置（含 CONFIG_AIRY_SC_FALLBACK）
│   ├── airy_defconfig.base     # 跨架构共享基线
│   └── README.md
├── Documentation/              # 文档即代码（ES-OLK-13）[IND]
├── scripts/                    # 构建与检查脚本（checkpatch.pl 等，ES-OLK-9）[IND]
└── tests/airymax/              # Airymax 专属测试用例（跨子仓集成测试在 tests-linux/integration/）[IND]
```

> **代码组织模型（Model A）**：kernel/ 子仓本身就是 Linux 6.6 完整 fork，Airymax 修改**直接写入上游子系统目录**（如 `mm/airymax_mm.c`、`kernel/sched/airy_sched.c`、`security/airy/airy_lsm.c`），**不使用 `patches/` 隔离层**。早期设计稿中的 `patches/user-sched-agent/`、`patches/io_uring-ipc/`、`patches/bpf-struct-ops/`、`patches/capability/`、`patches/rust-drivers/`、`patches/microkernel/` 等概念分组已合并到 fork 模型下的对应子系统目录（`corekern/sched/`、`corekern/ipc/`、`corekern/bpf/`、`corekern/object/`、`rust/`、上游 `fs/`+`net/`+`drivers/` 等）。权威源：[07-directory-structure.md §4](../10-architecture/07-directory-structure.md)。

### 3.1 corekern/sched — sched_tac 策略守护进程 \[SS]

存放 sched_tac 策略守护进程源码（用户态策略 daemon，非用户态调度器）。任务描述符与优先级 \[SC] 与 agentrt 共享：

- `airymax_agent_sched.c`：策略守护进程核心逻辑（基于 sched_tac，SCHED_FIFO/SCHED_DEADLINE + cgroup cpuset）\[SS]。
- `core_sched.c`：core-sched 集成 \[SS]。
- `ext.c`：扩展策略点 \[SS]。
- 调度策略枚举：`stc_realtime` / `stc_interactive` / `stc_agent` / `stc_batch`（用户态策略枚举，通过 `struct airy_sched_ops` 暴露回调，非 BPF struct\_ops）\[SS]。

### 3.2 corekern/ipc — io\_uring IPC 内核侧 \[SS]

基于 io\_uring 构建 IPC 通道的内核侧实现。IPC 消息头与操作码 \[SC] 与 agentrt 共享：

- `io_uring_ipc.c`：注册固定 OP，支持零拷贝消息传递（IORING\_OP\_URING\_CMD，不使用 page flipping）\[SS]。
- `kfifo_ipc.c`：kthread 间 kfifo IPC（禁止 io\_uring 用于 kthread 间通信）\[SS]。
- `fastpath.c`：IPC fastpath 优化（ES-SEL4-4 借鉴，POINT OF NO RETURN 模式）\[SS]。

### 3.3 corekern/bpf/struct\_ops — eBPF struct\_ops 扩展 \[SS]

eBPF struct\_ops 扩展，struct\_ops 状态机与 common\_value \[SC] 与 agentrt 共享：

- `airymax_sched_ops.c`：struct\_ops 注册机制扩展 \[SS]。
- 与 `bpf_struct_ops.h`（补充共享文件，非 \[SC] 核心头文件）协作。
- 自定义 kfunc 导出（`bpf_agent_decision_get` 等）与 ringbuf 事件格式（与 agentrt AgentsIPC 128B 消息头同源）由 `corekern/bpf/` 其他文件承载。

### 3.4 corekern/object — capability 内核侧 \[SS]

capability 系统的内核侧接口，借鉴 seL4 capability 模型（ES-SEL4-05\~09）：

- `cte.c`：CTE（Capability Table Entry）管理 \[SS]。
- `cspace.c`：CSpace 树形寻址（guard + radix）\[SS]。
- `mdb.c`：MDB（Memory Disclosure Base）派生树维护 \[SS]。
- `cnode_ops.c`：7 种 CNode 操作（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate）\[SS]。
- 与 `security/capability` 协作，通过 \[SC] `security_types.h` 共享契约。

> **v1.1 Capability Folding 实现注记**：上述 `cte.c` / `cspace.c` / `mdb.c` / `cnode_ops.c` 保留了 seL4 风格 CSpace+MDB 派生树的设计语义（ES-SEL4-05\~09 设计溯源），但 v1.1 Capability Folding 后工程实现已**简化**为：
>
> - **`agent_caps[1024]` 静态数组**（16KB，每槽 16 字节）替代动态 CSpace radix tree + MDB 双向链表，sec_d 为**唯一写者**（串行化写入消除并发同步开销）。
> - **Badge 64-bit 编码**：`Epoch<<48 | RandomTag<<16 | Perms`，派生关系隐式编码在 RandomTag 中，无需 MDB 链表维护。
> - **O(1) 撤销**：`atomic_inc(&airy_cap_global_epoch)` 全局递增 epoch，所有 badge 校验路径通过 epoch 比对自动失效旧 badge。
> - **CNode 7 操作语义保留**（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate），但实现路径通过 sec_d 串行化（不暴露并发 CNode 操作 API）。
> - **fastpath C-S9 内联校验**（~10ns）：在 io\_uring `IORING_OP_URING_CMD` 数据面路径上内联 Badge 校验，避免 slowpath LSM 钩子开销（slowpath 保留作审计/兜底）。
>
> 详见 §5.0 v1.1 Capability Folding 实现概览。SSoT：[03-capability-model.md](../110-security/03-capability-model.md) + [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)。

### 3.5 rust/ — Rust 安全驱动 \[IND]

Rust 安全驱动框架（直接写入上游 `rust/` 子系统目录）：

- `rust/kernel/`：扩展的 Rust 内核绑定。
- `samples/rust_drivers/`：示例安全驱动（网卡、块设备、字符设备）。
- `frameworks/`：抽象框架（trait-based 驱动模型）。

### 3.6 微内核化改造（直接写入上游 fs/ + net/ + drivers/）\[IND]

微内核化改造补丁直接合并到上游子系统目录（不使用 patches/ 隔离层）：

- `fs/`：VFS 元数据操作下放至用户态服务（与 `services/vfs/` 协作）。
- `net/`：网络栈用户态化（与 `services/net/` 协作）。
- `drivers/`：设备驱动拆分至用户态（与 `services/drivers/` 协作）。
- `corekern/memory/` + `mm/`：Untyped-Retype 内存模型内核接口（借鉴 ES-SEL4-03），与上游 `mm/` 协作。

***

## 4. 内核对象模型（seL4 借鉴）

> **设计依据**：seL4 五大内核对象模型（ES-SEL4-01），代码证据 `src/object/{tcb.c(2118), cnode.c(934), endpoint.c(577), notification.c(419), untyped.c(305)}`。

### 4.1 对象类型清单

seL4 内核仅维护 **12 种定长 capability 类型**（ES-SEL4-05），对应 **5 大核心内核对象**：

| seL4 对象                   | 文件                          | 行数    | agentrt-linux 对应               | 借鉴层 |
| ------------------------- | --------------------------- | ----- | ------------------------------ | --- |
| TCB（Thread Control Block） | `src/object/tcb.c`          | 2,118 | Linux `task_struct` + agent 扩展 | 架构层 |
| CNode（Capability Node）    | `src/object/cnode.c`        | 934   | capability 表节点                 | 架构层 |
| Endpoint（IPC 端点）          | `src/object/endpoint.c`     | 577   | io\_uring ring + Endpoint 语义   | 架构层 |
| Notification（异步信号）        | `src/object/notification.c` | 419   | eventfd + signal 语义            | 架构层 |
| Untyped（未类型化内存）           | `src/object/untyped.c`      | 305   | Retype 接口层                     | 架构层 |

**agentrt-linux 扩展对象**（在 Linux `task_struct` 基础上扩展）：

| 扩展对象         | 用途                                                         | 同源标注  |
| ------------ | ---------------------------------------------------------- | ----- |
| AgentTCB     | Agent 专属 TCB 扩展（agent\_id / capability\_set / ipc\_buffer） | \[SS] |
| SchedContext | 调度上下文（参考 seL4 MCS 模式 schedcontext.c:444）                   | \[SS] |
| TaskFlow     | 任务流描述符（与 agentrt taskflow 同源）                              | \[SS] |

### 4.2 TCB 结构设计

**seL4 TCB 布局参考**（`include/object/structures.h:48-75`）：

```
seL4 TCB 物理布局（seL4_TCBBits 对象）：
| cte_t[16]      |  16 个内嵌 CTE 槽位（tcbCTable/tcbVTable/tcbReply/tcbCaller/tcbBuffer 等）
| debug_tcb_t    |  调试信息
| tcb_t          |  核心 TCB 字段
```

**agentrt-linux TCB 扩展设计**（在 Linux `task_struct` 基础上）：

| 字段                   | 类型                  | 说明                     | 同源标注  |
| -------------------- | ------------------- | ---------------------- | ----- |
| `agent_id`           | u64                 | Agent 唯一标识             | \[SS] |
| `capability_set`     | cte\_t\*            | 指向 Agent 的 CSpace 根    | \[SS] |
| `ipc_buffer`         | void\*              | IPC Buffer 用户态地址       | \[SS] |
| `sched_context`      | sched\_context\_t\* | 调度上下文（MCS 模式参考）        | \[SS] |
| `bound_notification` | notification\_cap\* | 绑定的 Notification cap   | \[SS] |
| `badge`              | u64                 | Agent 身份标识（ES-SEL4-09） | \[SC] |

**关键设计决策**：agentrt-linux 不替换 Linux `task_struct`，而是在其上附加 `airy_tcb_ext` 扩展结构，通过 `task_struct->airy_ext` 指针关联。这保留了 Linux 调度器、信号处理、内存管理的完整性，同时注入 seL4 风格的 capability / IPC 语义。

### 4.3 对象类型分派

借鉴 seL4 的 `deriveCap` / `finaliseCap` 分派模式（`src/object/objecttype.c:62-200`）：

| 操作                  | seL4 实现                                             | agentrt-linux 落地  | 借鉴层   |
| ------------------- | --------------------------------------------------- | ----------------- | ----- |
| `deriveCap`         | 派生校验（zombie/irq\_control/reply 不可派生）                | capability 派生校验接口 | \[SS] |
| `finaliseCap`       | 终结化（final cap 触发 `cancelAllIPC`/`cancelAllSignals`） | Agent 终结化清理       | \[SS] |
| `isFinalCapability` | 最终 cap 判定（`src/object/cnode.c:846-874`）             | 最终 capability 判定  | \[SS] |

**对象大小约束**：seL4 每种对象有固定大小（`seL4_TCBBits` 等），agentrt-linux 通过 \[SC] 头文件定义对象大小常量，确保两端一致。

### 4.4 对象生命周期

借鉴 seL4 的对象生命周期管理（ES-SEL4-03 Untyped-Retype 机制）：

```
Create（Retype from Untyped）→ Activate → Suspend → Resume → Delete（Retype back to Untyped）
```

**agentrt-linux 落地**：

- **Create**：通过 `airy_sys_call(obj_cap, &msg)` 从 Untyped 内存 Retype 为具体对象（capability invocation）。
- **Activate**：绑定到 AgentTCB，开始参与调度。
- **Suspend/Resume**：通过sched_tac 的 `agent_runnable`/`agent_stopping` 回调控制。
- **Delete**：回收对象，内存 Retype 回 Untyped。

***

## 5. Capability 系统（seL4 借鉴）

> **设计依据**：seL4 capability 单一安全模型（ES-SEL4-05 至 09），是 seL4 安全模型的精髓。代码证据 `src/object/cnode.c`（934 行）、`src/kernel/cspace.c`（193 行）、`include/object/structures_64.bf`。

### 5.0 v1.1 Capability Folding 实现概览

v1.1 Capability Folding 决策对 seL4 风格 CSpace/MDB/radix tree 进行了工程简化，**保留设计语义、简化实现路径**。以下为 v1.1 实现概览（SSoT：[03-capability-model.md](../110-security/03-capability-model.md) + [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)）：

| 维度 | seL4 原始设计（设计溯源） | v1.1 Capability Folding 工程实现 | 简化理由 |
| --- | --- | --- | --- |
| cap 存储 | CSpace radix tree + CTE + mdb\_node 双向链表 | **`agent_caps[1024]` 静态数组**（16KB，每槽 16 字节） | MAC\_MAX\_AGENTS=1024 上限已知，静态数组消除动态分配 + 并发同步 |
| 写者模型 | 内核态多写者（CNode 操作并发） | **sec_d 唯一写者**（用户态串行化） | 用户态串行化消除内核锁，sec_d 持单写令牌 |
| Badge 编码 | 64-bit badge 字段（endpoint\_cap/notification\_cap） | **`Epoch<<48 \| RandomTag<<16 \| Perms`** | epoch 位段支持 O(1) 撤销，RandomTag 隐式编码派生关系 |
| 派生关系 | MDB（Mapping Database）双向链表维护父子 | **RandomTag 隐式编码**（同源派生共享 RandomTag） | 免链表维护，撤销通过 epoch 全局递增实现 |
| 撤销机制 | `cteRevoke` 递归遍历 MDB 子树 | **`atomic_inc(&airy_cap_global_epoch)`** O(1) | epoch 递增使所有旧 badge 自动失效，校验路径比对 epoch |
| 校验路径 | fastpath `cap_endpoint_cap_get_capEPBadge` + slowpath | **fastpath C-S9 内联校验**（~10ns）+ slowpath LSM 钩子（`airy_lsm`） | fastpath 内联避免函数调用开销，slowpath 保留审计 |
| CNode 操作 | 7 操作（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate）内核态并发 | **7 操作语义保留**，实现路径通过 sec_d 串行化 | 语义对齐 seL4（ES-SEL4-08），实现消除并发 |

**关键数据结构**（`security/airy/`）：

```c
/* v1.1 Capability Folding: 静态数组 + Badge 64-bit 编码 */
#define AIRY_CAP_MAX_AGENTS    1024
#define AIRY_CAP_GLOBAL_EPOCH_INIT  1

struct airy_cap_slot {
    __u64 badge;        /* Epoch<<48 | RandomTag<<16 | Perms */
    __u64 target_id;    /* Agent ID / Endpoint ID / Resource ID */
} __aligned(64);

struct airy_cap_table {
    struct airy_cap_slot slots[AIRY_CAP_MAX_AGENTS];  /* 16KB 静态数组 */
    atomic_t global_epoch;  /* O(1) 撤销：atomic_inc 即失效所有旧 badge */
};

/* 全局唯一实例，sec_d 唯一写者 */
extern struct airy_cap_table airy_cap_global_table;
```

**fastpath C-S9 内联校验路径**（~10ns，详见 §6.7 + [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)）：

```c
/* io_uring IORING_OP_URING_CMD fastpath: C-S9 Badge 内联校验 */
static inline int airy_cap_fastpath_verify(const struct airy_cap_slot *slot,
                                           __u64 expected_epoch)
{
    __u64 badge = smp_load_acquire(&slot->badge);
    __u64 slot_epoch = (badge >> 48) & 0xFFFF;
    if (unlikely(slot_epoch != expected_epoch))
        return -AIRY_ECAP_FROZEN;  /* -82: capability 冻结（epoch 失效） */
    return 0;
}
```

**slowpath LSM 钩子**（`airy_lsm`，CONFIG\_SECURITY\_AIRY，default 'n'）：通过 `LSM_ORDER_MUTABLE` 注册（受 CONFIG\_LSM 控制），在 `security_file_ioctl` / `security_uring_cmd` 等标准 LSM 钩子点叠加 Badge 审计。`LSM_ORDER_FIRST` 仅用于 capabilities（早期启动阶段）。

**与 seL4 设计溯源对照**：seL4 CSpace radix tree、MDB 派生树、CTE 双向链表等概念在 v1.1 保留为**设计溯源参考**（§5.1-§5.4 详述 seL4 原始模型），但工程实现简化为静态数组 + Badge 编码 + epoch 撤销。这一简化符合"对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造"定位（ADR-012 + ADR-014），不引入 seL4 完整 CSpace/MDB 实现复杂度。

### 5.1 CTE 与 CSpace 设计

**seL4 capability 即内存**（ES-SEL4-05）：每个 cap 是定长 word（64 位系统为 128 字节），通过 CTE（Capability Table Entry）存储。

| 维度         | seL4 实现                     | 代码证据                                    | agentrt-linux 落地                                     |
| ---------- | --------------------------- | --------------------------------------- | ---------------------------------------------------- |
| cap 类型     | 12 种定长 cap                  | `include/object/structures_64.bf:7-138` | 与 POSIX capability 41 ID 兼容（\[SC] security\_types.h） |
| CTE 结构     | cap + mdb\_node             | `include/object/structures.h:60-75`     | 内核 CTE 结构 \[IND]                                     |
| TCB 内嵌 CTE | 16 个槽位（TCB\_CNODE\_RADIX=4） | `include/object/structures.h:60-75`     | AgentTCB 内嵌 capability\_set \[SS]                    |

**CNode 树形寻址**（ES-SEL4-06）：CSpace 由 CNode 组成的 radix tree，每个 CNode cap 含三个关键字段：

| 字段                  | 位数     | 说明         |
| ------------------- | ------ | ---------- |
| `capCNodeGuard`     | 64 bit | 本节点匹配的地址前缀 |
| `capCNodeGuardSize` | 6 bit  | guard 位数   |
| `capCNodeRadix`     | 6 bit  | 本节点索引位数    |

**寻址算法**（`src/kernel/cspace.c:126-193` `resolveAddressBits`）：从根 CNode 开始，逐级匹配 guard、提取 radix 位索引下一层。

**agentrt-linux 落地**：支持多层 CSpace + guard，使 Agent 可传递"局部 capability 视图"给子 Agent，实现 capability 命名空间隔离。这是 Agent 沙箱的关键安全原语。

### 5.2 Capability 派生与撤销

**MDB 派生树**（ES-SEL4-07）：seL4 通过 MDB（Mapping Database）维护所有 capability 之间的派生关系。MDB 是双向链表，父 cap 可递归撤销所有派生自它的子孙 cap。

| 操作                  | seL4 实现     | 代码证据                         | agentrt-linux 用途 |
| ------------------- | ----------- | ---------------------------- | ---------------- |
| `cteInsert`         | 维护 MDB 链表   | `src/object/cnode.c:410-443` | 插入新 cap 时建立父子关系  |
| `cteRevoke`         | 递归删除所有子 cap | `src/object/cnode.c:528-550` | "一键撤销"子 Agent 权限 |
| `isMDBParentOf`     | 父子关系判定      | `src/object/cnode.c:775-819` | 沙箱权限边界判定         |
| `isFinalCapability` | 最终 cap 判定   | `src/object/cnode.c:846-874` | 触发对象销毁           |

**agentrt-linux 安全沙箱关键机制**：主 Agent 可通过 `cteRevoke` 一键撤销授予子 Agent 的所有权限——这是 seL4 实现安全沙箱的核心，也是 Agent 间权限传递的安全保障。

**\[SC] 对接**：通过 `include/uapi/linux/airymax/security_types.h` 定义 capability 派生模型（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate），与 agentrt Cupolas 同源。

### 5.3 Badge 机制与 Agent 身份

**seL4 Badge**（ES-SEL4-09）：Endpoint 和 Notification cap 含 64 位 badge 字段。当消息通过该 cap 发送时，badge 自动附加到消息中，接收方通过 badge 区分消息来源。

| 维度                  | seL4 实现                           | 代码证据                                    | agentrt-linux 落地           |
| ------------------- | --------------------------------- | --------------------------------------- | -------------------------- |
| badge 字段位置          | endpoint\_cap + notification\_cap | `include/object/structures_64.bf:24-43` | AgentTCB.badge 字段 \[SC]    |
| fastpath badge 传递   | `cap_endpoint_cap_get_capEPBadge` | `src/fastpath/fastpath.c:185`           | IPC 消息头 src\_task 字段 \[SC] |
| 派生规则                | badge=0 可派生任意，badge≠0 子 cap 相同    | `src/object/cnode.c:798-819`            | Agent 身份派生约束 \[SS]         |
| `cancelBadgedSends` | 按 badge 取消特定发送者                   | `src/object/endpoint.c:489-539`         | 按 Agent 取消挂起消息 \[SS]       |

**Agent 身份识别**：server Agent 能通过 badge 识别"哪个 client Agent 通过共享 Endpoint 发来的请求"，无需维护连接表。这是 seL4 实现多客户端 server 的核心机制。

### 5.4 7 种 CNode 操作

**seL4 CNode 操作原语**（ES-SEL4-08，`src/object/cnode.c:42-313`）：

| 操作         | 语义                    | badge 处理                      | agentrt-linux 用途 |
| ---------- | --------------------- | ----------------------------- | ---------------- |
| **Copy**   | 复制 cap，不修改 badge      | `maskCapRights` + `deriveCap` | Agent 间权限复制      |
| **Mint**   | Copy + 设置 badge/guard | `updateCapData`               | Agent 身份委托       |
| **Move**   | 原子迁移 cap              | 不修改                           | cap 重排           |
| **Mutate** | Move + 修改 guard       | 修改 guard                      | cap 重排 + 局部视图调整  |
| **Revoke** | 递归撤销所有子 cap           | —                             | "一键撤销"子 Agent 权限 |
| **Delete** | 删除单个 cap              | —                             | 单权限回收            |
| **Rotate** | 三槽位原子旋转               | —                             | 无临时槽的 cap 重排     |

**MCS 模式扩展**：seL4 MCS 模式增加 `SaveCaller` 操作（替换 `Rotate`），用于 Reply Cap 管理。

**agentrt-linux API**：通过 `airy_sys_call(cnode_cap, &msg)` 提供这 7 种原语（capability invocation），特别是 **Mint**（带 badge 复制）和 **Revoke**（递归撤销），实现可审计的 Agent 权限流转。

### 5.5 Capability Invocation 统一入口

`airy_sys_call` 是统一的 capability invocation 入口，借鉴 seL4 的 `decodeInvocation` 模式。操作类型由 capability 类型决定（cap-type dispatch），而非独立的 syscall 编号：

```c
/* Capability invocation dispatch (seL4 decodeInvocation style) */
int airy_sys_call(cap_t cap, const struct airy_ipc_msg_hdr *msg)
{
    switch (cap_get_type(cap)) {
    case CAP_TYPE_CNODE:
        return airy_cnode_invoke(cap, msg);  /* CNode 7 operations */
    case CAP_TYPE_TCB:
        return airy_tcb_invoke(cap, msg);    /* Task submit/cancel/status */
    case CAP_TYPE_ENDPOINT:
        return airy_endpoint_invoke(cap, msg); /* IPC rendezvous */
    case CAP_TYPE_SECURITY:
        return airy_security_invoke(cap, msg); /* LSM policy load */
    case CAP_TYPE_MODULE:
        return airy_module_invoke(cap, msg);   /* Wasm module load */
    default:
        return -AIRY_EINVAL;
    }
}
```

这消除了独立 syscall 的需求（task_submit/cancel/status/capability_request/revoke/lsm_ctl/wasm_load），v1.1 Capability Folding 后，airy_sys_call 收窄为 sec_d 专属管理入口（Badge 编译/撤销 + LSM_ctl + Wasm_load），IPC 数据传递由 io_uring fastpath C-S9 内联 Badge 校验承载，syscall 总数从 12 精简为 4。

***

## 6. IPC 消息传递（seL4 借鉴 + io\_uring 落地）

> **设计依据**：seL4 IPC 消息传递机制（ES-SEL4-10 至 15）。代码证据 `src/object/endpoint.c`（577 行）、`src/fastpath/fastpath.c`（899 行）、`src/object/notification.c`（419 行）。

### 6.1 IPC 控制面/数据面分离架构

> **术语澄清**：本节"控制面/数据面分离"指 IPC 通信路径的分离（syscall 控制面 + io_uring 数据面），与 §8 Capability Folding 的 fastpath/slowpath 校验路径分离（fastpath C-S9 内联 + slowpath LSM 钩子）是**两个不同维度的分离**，禁止混淆。

agentrt-linux 的 IPC 采用 **数据面/控制面分离** 架构：

| 平面      | 路径                                | 延迟      | 通信方式               | 用途                                |
| ------- | --------------------------------- | ------- | ------------------ | --------------------------------- |
| **控制面** | 用户态 → syscall → 内核 → 返回           | \~1 μs  | 4 核心 syscall（`airy_sys_call` sec_d 专属管理 + 3 控制原语） | capability 编译/撤销、调度策略、记忆卷载、CoreLoopThree 通知 |
| **数据面** | 用户态 → SQE → io\_uring → CQE → 用户态 | \~10 μs | io\_uring 零拷贝 ring（C-S9 fastpath Badge 内联校验） | IPC 消息收发、记忆迁移、流式数据                  |

**设计原理**：

- 控制面用于低频、需同步语义的操作（capability invocation、调度策略设置、认知阶段通知），走 syscall。
- 数据面用于高频、可异步、需零拷贝的操作（IPC 消息收发、记忆迁移），走 io\_uring。
- 二者配合实现"机制在内核、策略在用户态"的微内核目标。

### 6.2 Endpoint 状态机

**seL4 Endpoint**（ES-SEL4-10）：同步 IPC 通道，仅 16 字节，通过 2 bit state 字段显式编码三种状态：

| 状态             | 值 | 含义     | seL4 代码证据                        |
| -------------- | - | ------ | -------------------------------- |
| `EPState_Idle` | 0 | 无等待者   | `include/object/structures.h:34` |
| `EPState_Send` | 1 | 有发送者等待 | `include/object/structures.h:35` |
| `EPState_Recv` | 2 | 有接收者等待 | `include/object/structures.h:36` |

**状态转换**（`src/object/endpoint.c:27-296`）：

- `sendIPC`：Idle/Send 时阻塞发送者（ThreadState\_BlockedOnSend），Recv 时直接 transfer + dequeue 接收者。
- `receiveIPC`：Idle/Recv 时阻塞（ThreadState\_BlockedOnReceive），Send 时直接 transfer。

**agentrt-linux 落地**：IPC channel 采用显式状态机，而非 Linux pipe 的"缓冲区+阻塞"模型。显式状态使 IPC 路径可形式化推理，且支持 fastpath 优化。io\_uring ring 作为 Endpoint 的物理载体，\[SC] `ipc.h` 定义 128B 消息头。

### 6.3 Message Register 与 IPC Buffer

**seL4 双通道 IPC**（ES-SEL4-11, 12）：优先使用 CPU 寄存器（msgRegisters）传递前 N 个 word，超出部分通过用户态 IPC buffer 传递。

| 维度                   | seL4 实现                                    | 代码证据                              | agentrt-linux 落地          |
| -------------------- | ------------------------------------------ | --------------------------------- | ------------------------- |
| MessageInfo          | length + extraCaps + capsUnwrapped + label | `include/api/syscall.h:38-49`     | \[SC] ipc.h 消息头字段         |
| MR（Message Register） | 物理 CPU 寄存器（aarch64 4 个）                    | `src/arch/arm/64/api/benchmark.c` | io\_uring SQE 内联数据        |
| IPC Buffer           | 用户态可写区域                                    | `include/object/structures.h`     | io\_uring SQ/CQ ring 共享内存 |
| copyMRs              | 跨架构实现                                      | `src/arch/*/object/ipc.c`         | io\_uring 零拷贝路径           |

**\[SC] 对接**：`struct airy_ipc_msg_hdr`（128B）定义在 `include/uapi/linux/airymax/ipc.h`，包含 magic（0x41524531 'ARE1'）、opcode、flags、trace\_id、timestamp\_ns、src\_task、dst\_task、capability\_badge（v1.1 Capability Folding，offset 40，D-9 修复后 8 字节对齐）、payload\_len、crc32（offset 52）、reserved\[72]（offset 56，CL1 [56:64) + CL2 [64:128)）等字段，共 11 个字段，`__attribute__((aligned(64)))` 对齐。与 SSoT [02-ipc-protocol.md §2](../30-interfaces/02-ipc-protocol.md) 逐字节一致。

### 6.4 Fastpath 设计（借鉴 seL4 POINT OF NO RETURN 模式）

**seL4 Fastpath**（ES-SEL4-13，源码证据 `src/fastpath/fastpath.c:168-233`）：IPC 快路径的核心理念是 **POINT OF NO RETURN**——一旦进入 fastpath，就不允许回退到慢路径。seL4 Fastpath 的语义是"批量验证后不可逆提交 + 直接切换线程上下文"，**不是"零拷贝"**。

| 维度                 | seL4 实现                           | 代码证据                                |
| ------------------ | --------------------------------- | ----------------------------------- |
| fastpath 入口        | `lookup_fp` 无递归 do-while          | `include/fastpath/fastpath.h:46-90` |
| fastpath 主体        | 12 项前置检查 + 直接 transfer            | `src/fastpath/fastpath.c:168-233`（POINT OF NO RETURN 在 L168-233 之间）    |
| badge 传递           | `cap_endpoint_cap_get_capEPBadge` | `src/fastpath/fastpath.c:185`       |
| POINT OF NO RETURN | 12 项前置检查全部通过后进入不可逆点，直接切换线程上下文                           | `src/fastpath/fastpath.c:168-233`           |

**Fastpath 触发条件**（4 类）：

1. **call fastpath**：Caller → Callee，无 fault、无额外 caps
2. **reply\_recv fastpath**：Callee 回复并等待下一条
3. **signal fastpath**：Notification 发送
4. **vm\_fault fastpath**：页故障处理

**agentrt-linux 落地（语义分层）**：
- **Fastpath 模式借鉴**（POINT OF NO RETURN）：agentrt-linux IPC 快路径借鉴 seL4 "12 项前置检查 + 不可逆提交"模式——内核在进入临界区后完成全部前置检查（端点状态、capability 有效性、buffer 已注册、无 fault），全部通过后进入不可逆点，直接切换上下文，避免回退开销。
- **零拷贝优化（独立命名）**：io\_uring 的固定 buffer + registered ring 路径是**独立的零拷贝优化**，与 seL4 Fastpath 语义不同——seL4 Fastpath 是"批量验证后不可逆提交 + 线程上下文切换"，io_uring 零拷贝是"buffer 共享避免数据拷贝"。两者同名但不同义，**禁止混用**。

**当满足"固定 buffer + registered ring + 无 fault + 端点 Idle + 双方就绪"条件时，走 IPC Fastpath（POINT OF NO RETURN 模式 + io_uring 零拷贝优化）；否则走慢路径（io_uring enter + fallback）。**

### 6.5 Notification 异步信号

**seL4 Notification**（ES-SEL4-14）：与 Endpoint 不同，Notification 是异步信号机制。

| 维度            | seL4 实现                   | 代码证据                              |
| ------------- | ------------------------- | --------------------------------- |
| 数据传递          | word 传递 + badge 聚合        | `src/object/notification.c:1-100` |
| 状态机           | Idle / Waiting / Active   | `include/object/structures.h`     |
| TCB 绑定        | bound notification        | `src/object/tcb.c`                |
| 与 Endpoint 关系 | 可绑定到 TCB，被 blocked IPC 唤醒 | `src/object/notification.c`       |

**agentrt-linux 落地**：基于 eventfd + signalfd 实现 Notification 语义。bound notification 对应 AgentTCB 的 `bound_notification` 字段（§4.2）。

### 6.6 IPC 数据面与控制面分离

**控制面（v1.1 4 核心 syscall）**：

| 编号 | syscall               | seL4 对应          | 说明                       |
| -- | --------------------- | ---------------- | ------------------------ |
| 0  | `airy_sys_call`       | `seL4_Call`      | 统一 capability invocation（sec_d Badge 编译/撤销 + LSM_ctl + Wasm_load） |
| 1  | `airy_sys_rovol_ctl`  | —                | 记忆卷载控制（agent 领域最小需求） |
| 2  | `airy_sys_sched_ctl`  | —                | 调度策略配置（agent 领域最小需求） |
| 3  | `airy_sys_clt_notify` | `seL4_Notification` | CoreLoopThree 通知 + kthread 注册 |

> **v1.1 Capability Folding 说明**：原 8 个 seL4 风格 IPC 原语 syscall（`airy_sys_send`/`recv`/`nbsend`/`nbrecv`/`reply_recv`/`yield`/`reply`/`notify`）在 v1.1 全部移除，IPC 数据传递完全由 io\_uring `IORING_OP_URING_CMD` 承载（见下方数据面表）。能力校验折叠到数据面 fastpath C-S9 内联 Badge 校验，`airy_sys_call` 语义收窄为 sec_d 专属管理入口。seL4 借鉴映射保留作设计溯源参考。

**数据面（io\_uring 零拷贝）**：

| io\_uring 机制              | seL4 对应           | 同源标注  |
| ------------------------- | ----------------- | ----- |
| SQ/CQ ring 共享内存           | IPC Buffer        | \[SS] |
| SQE 内联数据                  | Message Register  | \[SS] |
| MSG\_RING 跨 ring 消息       | Endpoint send     | \[SS] |
| registered buffer         | 固定 cap 映射         | \[SS] |
| zero-copy (IORING_OP_URING_CMD) | fastpath transfer | \[SS] |

**IPC magic 0x41524531 'ARE1'** \[SC] 与 agentrt 共享，确保两端通信协议一致。

### 6.7 v1.1 Capability Folding 后的 fastpath C-S9 Badge 校验路径

v1.1 Capability Folding 将 capability 校验从控制面 syscall 折叠到数据面 io\_uring fastpath，**消除每条 IPC 消息的 syscall 开销**。SSoT：[07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)。

**校验路径概览**：

```
用户态提交 SQE（IORING_OP_URING_CMD，cmd 扩展至 80 字节，SQE128 模式）
    ↓
io_uring 内核侧 issue 路径
    ↓
[fastpath C-S9] 内联 Badge 校验（~10ns）
    ├─ C-S9.EPOCH：slot_epoch == expected_epoch？（不匹配返回 -AIRY_ECAP_FROZEN=-82）
    ├─ C-S9.RANDTAG：RandomTag 派生关系校验
    └─ C-S9.PERMS：Perms 位段权限校验（send/recv/cap_request 等）
    ↓
校验通过 → io_uring_cmd_done(cmd, ret, res2, issue_flags) 4 参数完成
校验失败 → 返回 -82 或 -83（AIRY_ESEC_D_THROTTLED sec_d 限流拒绝）
    ↓
[slowpath] airy_lsm 钩子（LSM_ORDER_MUTABLE，CONFIG_LSM 控制）
    └─ security_uring_cmd(struct io_uring_cmd *ioucmd) 单参数钩子叠加审计
```

**C-S9 子状态**（仅在 [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) 声明，本文档仅引用）：

| 子状态 | 校验内容 | 失败返回码 |
| --- | --- | --- |
| C-S9.EPOCH | `slot_epoch == expected_epoch`（O(1) 撤销生效判定） | `-AIRY_ECAP_FROZEN`（-82） |
| C-S9.RANDTAG | RandomTag 派生关系合法性 | `-AIRY_ECAP_FROZEN`（-82） |
| C-S9.PERMS | Perms 位段权限匹配（opcode vs cap 权限） | `-AIRY_EPERM`（-4） |

**sec_d 串行化 Badge 编译**：sec_d 通过 `airy_sys_call`（编号 0）独占 Badge 编译/撤销，内部采用**令牌桶限流**（避免恶意 Agent 暴力耗尽 epoch 空间），**50ms SLO**（Badge 编译请求 50ms 内完成）。sec_d 限流拒绝时返回 `AIRY_ESEC_D_THROTTLED = -83`。

**fastpath 与 slowpath 分工**：

| 路径 | 延迟 | 触发条件 | 责任 |
| --- | --- | --- | --- |
| fastpath C-S9 | ~10ns | io\_uring `IORING_OP_URING_CMD` 数据面 | Badge epoch/randtag/perms 内联校验 |
| slowpath LSM | ~1μs | `security_uring_cmd` 钩子（airy_lsm） | 审计日志、策略裁决（ALLOW/DENY/AUDIT/COMPLAIN） |

**io_uring 工程规范**（OLK 6.6）：

- SQE128 模式：`IORING_SETUP_SQE128`，cmd 扩展至 80 字节（16→80）。
- PDU 访问：`io_uring_cmd_to_pdu(cmd, pdu_type)` 安全宏访问 `pdu[32]` 字段。
- 完成接口：`io_uring_cmd_done(cmd, ret, res2, issue_flags)` 4 参数。
- uring\_cmd LSM 钩子：单参数 `struct io_uring_cmd *ioucmd`。

**故障码**（`AIRY_FAULT_*` 前缀）：

| 故障码 | 值 | 含义 |
| --- | --- | --- |
| `AIRY_FAULT_URING_MALFORMED` | `0x100A` | io\_uring SQE/cqe 格式错误 |
| `AIRY_FAULT_AUDIT_TAMPER` | `0x100B` | 审计哈希链被篡改（audit_d 检测） |

***

## 7. 调度与线程模型

> **设计依据**：seL4 调度器设计（ES-SEL4-04 零策略内核）+ Linux 6.6 sched_tac（策略框架，非用户态调度器，非 sched_ext）。代码证据 `src/kernel/thread.c`（752 行）、`src/object/tcb.c:32-50`（checkPrio）、`src/object/schedcontext.c`（444 行）。

### 7.1 调度器设计

**seL4 调度器**（ES-SEL4-04）：只实现 priority + round-robin + domain 三个机制，具体优先级数值、时间片长度由用户态通过 capability invocation 设置。`checkPrio` 仅校验 `prio > mcp`，不决定优先级数值。

**agentrt-linux 落地**：Agent 调度策略（优先级、QoS、deadline）作为用户态 policy server 实现，内核仅提供 `set_priority`/`yield_to`/`bind_sched_context` 等原语。这与 seL4 的"机制在内核、策略在用户态"原则一致。

### 7.2 任务描述符（\[SC] 共享）

任务描述符 `struct airy_task_desc` 定义于 SSoT `sched.h`，两端共享：

| 字段      | 类型             | 说明                                            |
| ------- | -------------- | --------------------------------------------- |
| `magic` | `__u32`        | `AIRY_TASK_MAGIC`（0x41475453 'AGTS'）          |
| `prio`  | `__u16`        | 优先级 \[AIRY\_PRIO\_MIN=0, AIRY\_PRIO\_MAX=139] |
| `_pad`  | `__u16`        | 填充对齐                                          |
| `vtime` | `airy_vtime_t` | 虚拟时间（Q16.16 定点数，`int32_t`）                    |

**vtime 衰减公式**（SSoT `sched.h`）：

```c
static inline airy_vtime_t
airy_vtime_decay(airy_vtime_t vtime, u64 consumed_slice, u32 weight)
{
    return vtime + (airy_vtime_t)(consumed_slice * 100 / weight);
}
```

**关键常量**：

| 常量                  | 值     | 说明           |
| ------------------- | ----- | ------------ |
| `AIRY_PRIO_MIN`     | 0     | 最高优先级        |
| `AIRY_PRIO_MAX`     | 139   | 最低优先级        |
| `AIRY_WEIGHT_MIN`   | 1     | 最小权重         |
| `AIRY_WEIGHT_MAX`   | 10000 | 最大权重         |
| `AIRY_SLICE_DFL` | 20    | 默认时间片（ms）    |
| `AIRY_CAP_MAX_AGENTS`    | 1024  | 并发 Agent 硬上限 |

**注意**：vtime 使用 Q16.16 定点数（`int32_t`），因内核态禁止浮点运算（`-mno-80387`）。agentrt 用户态可使用 `float`，但跨 \[SC] 边界统一使用 Q16.16。

### 7.3 调度上下文（MCS 模式参考）

**seL4 MCS 模式**（`src/object/schedcontext.c` 444 行 + `src/object/schedcontrol.c` 186 行）：

| 维度           | seL4 MCS 实现   | agentrt-linux 落地              |
| ------------ | ------------- | ----------------------------- |
| SchedContext | 常量带宽管理        | sched_tac sub-scheduler 带宽配额 |
| refill 机制    | refill + 带宽管理 | cgroup cpu.max/cpu.weight     |
| 常量带宽 vs 突发   | 参数化           | stc_agent 策略枚举可配             |

**Agent 调度数量上限**：MAC\_MAX\_AGENTS 1024，基准测试验证到 1000 个并发。

### 7.4 线程状态机

**seL4 ThreadState**（`src/kernel/thread.c:38-112`）：

| 状态                                  | 含义          | agentrt-linux 对应                            |
| ----------------------------------- | ----------- | ------------------------------------------- |
| `ThreadState_Running`               | 运行中         | Linux TASK\_RUNNING                         |
| `ThreadState_Restart`               | 重启          | Linux TASK\_INTERRUPTIBLE → RUNNING         |
| `ThreadState_BlockedOnReceive`      | 阻塞等待 IPC 接收 | Linux TASK\_INTERRUPTIBLE（io\_uring CQE 等待） |
| `ThreadState_BlockedOnSend`         | 阻塞等待 IPC 发送 | Linux TASK\_INTERRUPTIBLE（io\_uring SQE 提交） |
| `ThreadState_BlockedOnReply`        | 阻塞等待回复      | Linux TASK\_INTERRUPTIBLE                   |
| `ThreadState_BlockedOnNotification` | 阻塞等待通知      | Linux TASK\_INTERRUPTIBLE（eventfd 等待）       |
| `ThreadState_Inactive`              | 未激活         | Linux TASK\_DEAD（退出）                        |

***

## 8. 内存管理（seL4 借鉴 + Linux 对接）

> **设计依据**：seL4 Untyped-Retype 内存模型（ES-SEL4-03）+ Linux 6.6 内存管理。代码证据 `src/object/untyped.c`（305 行）。

### 8.1 Untyped-Retype 内存模型

**seL4 Untyped**（ES-SEL4-03）：物理内存通过 Untyped cap 管理，Retype 操作将 Untyped 变为具体内核对象。

| 操作                        | seL4 实现                         | agentrt-linux 落地         |
| ------------------------- | ------------------------------- | ------------------------ |
| `decodeUntypedInvocation` | 解析 Retype 参数（type + size\_bits） | capability invocation 分发 |
| `invokeUntyped_Retype`    | 将 Untyped 变为具体对象                | 内核对象分配                   |
| `createNewObjects`        | 创建对象 + 初始化 CTE                  | CTE 初始化                  |

**agentrt-linux 落地**：通过 `airy_sys_call(untyped_cap, &msg)` 实现 Retype 操作（capability invocation），而非独立的 `ioctl`。

### 8.2 MemoryRovol 对接（\[SC] 共享）

MemoryRovol L1-L4 层级定义于 SSoT `memory_types.h`：

| 层级     | 枚举                 | 说明           | GFP 掩码          |
| ------ | ------------------ | ------------ | --------------- |
| L1 热数据 | `AIRY_MEM_L1_HOT`  | 当前 Agent 工作集 | `AIRY_GFP_HOT`  |
| L2 温数据 | `AIRY_MEM_L2_WARM` | 节点本地         | `AIRY_GFP_WARM` |
| L3 冷数据 | `AIRY_MEM_L3_COLD` | 节点远端         | `AIRY_GFP_COLD` |
| L4 持久化 | `AIRY_MEM_L4_PMEM` | PMEM 持久内存    | `AIRY_GFP_PMEM` |

通过 `airy_sys_rovol_ctl` 进行快照/恢复/迁移/分层操作，与 `memory` 子仓协作。

***

## 9. 核心特性（Linux 6.6 原生）

agentrt-linux 基于 Linux 6.6 内核基线，充分利用以下原生特性：

### 9.1 sched_tac（策略框架，非用户态调度器）

- **sched_tac**：基于 Linux 6.6 标准内核原生 SCHED\_DEADLINE/SCHED\_FIFO/EEVDF + cgroup cpuset 隔离 + seL4 MCS 语义映射的策略框架（非用户态调度器，非 sched\_ext）。> **注**：OLK 6.6 已 backport sched_ext（`kernel/sched/ext.c`，215KB），但选择不使用 sched_ext 的理由是：① sched_ext 依赖 BPF_SYSCALL && BPF_JIT && DEBUG_INFO_BTF，与 H5 纯 C LSM 原则冲突；② x86 默认禁用 sched_ext（KABI 考量）；③ BPF verifier 语义不确定性影响形式化验证可行性；④ BPF struct_ops 模型与纯 C 体系不一致。vanilla Linux 6.6 主线不含 sched_ext（6.12 才合入），但 OLK 6.6 已 backport 此特性。零内核调度器修改，不新增内核调度类，不修改 CFS/EEVDF 核心代码。
- **stc_\* 策略枚举**：用户态调度策略枚举（`stc_realtime`/`stc_interactive`/`stc_agent`/`stc_batch`），通过 `struct airy_sched_ops` 暴露回调（纯用户态函数表，非 BPF struct\_ops）。`AIRY_SCHED_AGENT` 已彻底废弃（2026-07-18 用户裁决），全部替换为 sched_tac 策略框架 + stc_\* 策略枚举（字符串常量 `AIRY_STC_POLICY_NAME "stc_agent"`）。**禁止**定义 `SCHED_AGENT` 内核调度类宏（已确立禁用，避免与内核调度类编号冲突）。
- 通过 `airy_sys_sched_ctl` 控制策略配置。

### 9.2 io\_uring

- 零 syscall、零拷贝 IPC 数据面。
- 固定 buffer + registered ring + IORING_OP_URING_CMD。
- SQE/CQE 标志位定义于 SSoT `ipc.h`（`AIRY_IPC_SQE_F_*` / `AIRY_IPC_CQE_F_*`）。
- Ring 常量：默认 256 entries，最大 32768 entries。

### 9.3 eBPF

- struct\_ops 状态机 \[SC] 共享：`airy_struct_ops_state`（INIT/REGISTERED/ACTIVE/DRAINING）。
- kfunc 扩展 + ringbuf 上报。
- struct\_ops 状态机与 common\_value 布局 \[SC] 与 agentrt 共享。

### 9.4 Rust 支持

- 依托 Linux 6.6 中 Rust 实验性支持（持续演进中），构建安全驱动开发框架。

### 9.5 约束声明

- **内核态禁 float**（`arch/x86/Makefile:137` `-mno-80387`），用 Q16.16 定点数 `airy_q16_t`（=`int32_t`）。
- **kthread 间通信**用 `kfifo` + `wait_event_interruptible`，禁止 io\_uring 用于 kthread 间通信。
- **sched_tac**：基于 Linux 6.6 原生调度类组合（SCHED\_FIFO/SCHED\_DEADLINE/EEVDF）的策略框架，非用户态调度器，非 sched_ext，禁止定义 SCHED\_AGENT 内核调度类宏。

***

## 10. IRON-9 v3 四层共享模型落地

本节详细列出 agentrt-linux 内核中 IRON-9 v3 四层共享模型的具体落地细节。所有 \[SC] 定义以 SSoT（`120-cross-project-code-sharing.md`）为唯一权威来源。

### 10.1 \[SC] 共享契约层——10 个头文件

#### 10.1.1 sched.h — 调度契约

| 共享内容          | 符号                      | 值/类型            | 说明                                                                      |
| ------------- | ----------------------- | --------------- | ----------------------------------------------------------------------- |
| 任务描述符 magic   | `AIRY_TASK_MAGIC`       | `0x41475453u`   | 'AGTS'（Agent Task State）                                                |
| 任务描述符结构       | `struct airy_task_desc` | 4 字段            | magic(\_\_u32) + prio(\_\_u16) + \_pad(\_\_u16) + vtime(airy\_vtime\_t) |
| vtime 类型      | `airy_vtime_t`          | `int32_t`       | Q16.16 定点数                                                              |
| vtime 衰减函数    | `airy_vtime_decay()`    | inline          | vtime + (consumed\_slice \* 100 / weight)                               |
| 优先级范围         | `AIRY_PRIO_MIN/MAX`     | 0 / 139         | 兼容 Linux 优先级                                                            |
| 权重范围          | `AIRY_WEIGHT_MIN/MAX`   | 1 / 10000       | 兼容sched_tac 权重模型                                                      |
| 默认时间片         | `AIRY_SLICE_DFL`     | 20              | 20ms                                                                    |
| sched_tac 约束 | —                       | 标准 6.6 原生 | 禁止定义 SCHED\_AGENT 内核调度类宏                                                     |

#### 10.1.2 ipc.h — IPC 契约

| 共享内容      | 符号                                                   | 值/类型              | 说明                                                                                                                                                                                     |
| --------- | ---------------------------------------------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IPC magic | `AIRY_IPC_MAGIC`                                     | `0x41524531u`     | 'ARE1'（Airymax Runtime Engine v1）                                                                                                                                                      |
| 消息头大小     | `AIRY_IPC_HDR_SIZE`                                    | 128               | 128 字节                                                                                                                                                                                 |
| 消息头结构     | `struct airy_ipc_msg_hdr`                            | 11 字段              | magic(\_\_u32) + opcode(\_\_u16) + flags(\_\_u16) + trace\_id(\_\_u64) + timestamp\_ns(\_\_u64) + src\_task(\_\_u64) + dst\_task(\_\_u64) + capability\_badge(\_\_u64, offset 40, v1.1 Capability Folding) + payload\_len(\_\_u32) + crc32(\_\_u32, offset 52) + reserved[72](__u8, offset 56) |
| 消息标志      | `AIRY_IPC_F_ZEROCOPY/CAP_CARRY/BATCH_TAIL`           | (1u<<0)/(1u<<1)/(1u<<4) | 零拷贝 / 携带 Badge / 批量尾（v1.1 废弃 NOWAIT/SIGNAL，由 io_uring `IOSQE_ASYNC` 与 CQE 通知替代）                                                                                                  |
| IPC 操作码   | `AIRY_IPC_OP_SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE` | 0x0001~0x0011 | 7 个操作码（v1.1 Capability Folding）                                                                                                                                                 |
| SQE 标志    | `AIRY_IPC_SQE_F_FIXED_BUF/ASYNC/BUF_SELECT/SKIP_CQE` | (1u<<0)\~(1u<<3)  | io\_uring SQE 标志                                                                                                                                                                       |
| CQE 标志    | `AIRY_IPC_CQE_F_BUFFER/MORE/NOTIF`                   | (1u<<0)\~(1u<<2)  | io\_uring CQE 标志                                                                                                                                                                       |
| Ring 容量   | `AIRY_IPC_RING_DEF/MAX_ENTRIES`                      | 256 / 32768       | 默认/最大 ring 条目数                                                                                                                                                                         |

#### 10.1.3 补充共享文件：bpf\_struct\_ops.h — eBPF struct\_ops 状态同步（非 \[SC] 核心头文件）

| 共享内容           | 符号                             | 值/类型 | 说明                                            |
| -------------- | ------------------------------ | ---- | --------------------------------------------- |
| struct\_ops 值  | `struct airy_struct_ops_value` | 2 字段 | state(\_\_u32) + common(\_\_u64)              |
| struct\_ops 状态 | `airy_struct_ops_state`        | 4 枚举 | INIT=0 / REGISTERED=1 / ACTIVE=2 / DRAINING=3 |

#### 10.1.4 memory\_types.h — 记忆卷载契约

| 共享内容   | 符号                            | 值/类型 | 说明                                               |
| ------ | ----------------------------- | ---- | ------------------------------------------------ |
| 记忆层级   | `airy_mem_level`              | 4 枚举 | L1\_HOT=0 / L2\_WARM=1 / L3\_COLD=2 / L4\_PMEM=3 |
| GFP 掩码 | `AIRY_GFP_HOT/WARM/COLD/PMEM` | —    | 对应 L1-L4 分配策略                                    |

#### 10.1.5 security\_types.h — 安全契约

| 共享内容          | 符号                                          | 值/类型     | 说明                                                                   |
| ------------- | ------------------------------------------- | -------- | -------------------------------------------------------------------- |
| Capability ID | `AIRY_CAP_AGENT_SPAWN/GPU_SCHED/NPU_ACCESS` | 41/42/43 | Airymax 专属（从 41 开始，避免与 Linux 0-40 冲突）                                |
| LSM 钩子        | `AIRY_LSM_HOOK_TASK_CREATE/IPC_SEND`        | 0/1      | 250 个 LSM 钩子 ID                                                      |
| 策略裁决          | `airy_verdict`                              | 4 枚举     | ALLOW=0 / DENY=1 / AUDIT=2 / COMPLAIN=3                             |
| Capability 操作 | `airy_cap_op`                               | 7 枚举     | COPY=0 / MINT=1 / MOVE=2 / MUTATE=3 / REVOKE=4 / DELETE=5 / ROTATE=6 |

#### 10.1.6 cognition\_types.h — 认知契约

| 共享内容             | 符号                | 值/类型      | 说明                                 |
| ---------------- | ----------------- | --------- | ---------------------------------- |
| CoreLoopThree 阶段 | `airy_cog_phase`  | 3 枚举      | PERCEPT=0 / THINK=1 / ACT=2        |
| Thinkdual 模式     | `airy_think_mode` | 2 枚举      | FAST=0（System-1）/ SLOW=1（System-2） |
| Q16.16 定点类型      | `airy_q16_t`      | `int32_t` | 内核态禁浮点替代                           |

#### 10.1.7 syscalls.h — Syscall 编号体系（新增）

| 共享内容       | 符号                                                       | 值                     | 说明                         |
| ---------- | -------------------------------------------------------- | --------------------- | -------------------------- |
| Syscall 架构 | —                                                        | v1.1: 4 核心 + 20 预留 = 24 槽位 | 1 Capability Invocation + 3 控制原语（IPC 数据面零 syscall） |
| 编号 0       | `AIRY_SYS_CALL`                                          | 0                     | Capability Invocation（sec_d 专属管理入口） |
| 编号 1-3     | `AIRY_SYS_ROVOL_CTL/SCHED_CTL/CLT_NOTIFY`                | 1-3                   | 3 个控制原语（记忆卷载/调度/CoreLoopThree） |
| 编号 4-23    | 预留                                                       | 4-23                  | 未来扩展                       |

### 10.2 \[SS] 语义同源层——30+ 项

| 语义域        | agentrt 实现                   | agentrt-linux 实现                    | \[SC] 契约依据                              |
| ---------- | ---------------------------- | ----------------------------------- | --------------------------------------- |
| 调度语义       | MicroCoreRT 用户态调度器           | sched_tac 策略框架（SCHED\_FIFO/SCHED\_DEADLINE，非用户态调度器）    | `sched.h`：任务描述符 + vtime + 优先级           |
| 安全模型       | Cupolas 用户态策略引擎              | LSM 钩子（250 ID）+ capability 41 ID    | `security_types.h`：capability + verdict |
| IPC 传输     | 用户态消息队列（mqueue/io\_uring）    | 内核 io\_uring 驱动（数据面，零 syscall）+ 4 核心 syscall（控制面，v1.1 Capability Folding 后） | `ipc.h`：128B 消息头 + magic 'ARE1'         |
| 记忆模型       | 用户态 heapstore（malloc + mmap） | 内核态 L1-L4 分层（kmalloc + pmem）        | `memory_types.h`：L1-L4 + GFP 掩码         |
| 认知模型       | 用户态 CoreLoopThree 引擎         | 内核 kthread 认知通知                     | `cognition_types.h`：三阶段枚举               |
| Syscall 编号 | 用户态 `syscalls.h` 宏定义         | 内核 `syscall_64.tbl` 表项              | `syscalls.h`：24 槽位编号                    |

### 10.3 \[IND] 完全独立层——15+ 项

| 独立项            | agentrt 实现          | agentrt-linux 实现   | 独立原因        |
| -------------- | ------------------- | ------------------ | ----------- |
| 调度类            | 用户态协程/线程池           | sched_tac 策略守护进程注册 | 跨平台 vs 内核专属 |
| io\_uring 工作队列 | 无                   | io-wq 内核工作队列       | 内核专属        |
| eBPF JIT       | 无                   | x86/ARM64 JIT 后端   | 内核专属        |
| eBPF verifier  | 无                   | 内核 BPF verifier    | 内核专属        |
| KABI           | 无                   | 内核 ABI 稳定性         | 内核专属        |
| VFS/网络/驱动      | 用户态服务               | 内核态（逐步用户态化）        | 架构差异        |
| Rust 驱动        | 无                   | 内核 Rust 驱动框架       | 内核专属        |
| 跨平台            | Linux/macOS/Windows | Linux 6.6 专属       | 平台差异        |

***

## 11. Boot 流程

agentrt-linux 的 Boot 流程基于 Linux 6.6 标准启动流程，增加以下阶段：

1. **Linux 内核初始化**：标准 Linux 6.6 启动流程（setup\_arch → sched\_init → mm\_init → rest\_init）。
2. **agentrt-linux 子系统初始化**：在 `initcall` 阶段初始化 io\_uring IPC、eBPF struct\_ops、capability 系统、sched_tac 策略守护进程。
3. **Agent 服务启动**：用户态 init 进程启动 Agent 服务管理器（services），注册 sched_tac 策略守护进程。
4. **Capability 引导**：根 Agent 获得初始 CSpace，开始 capability 分发。
5. **进入 Agent 调度**：sched_tac 策略守护进程接管 Agent 任务调度。

***

## 12. 错误处理与形式化验证预留

### 12.1 错误码体系

错误码统一使用 `AIRY_E*` 前缀，SSoT 定义于 `include/uapi/linux/airymax/error.h`：

| 错误码              | 值   | 含义    |
| ---------------- | --- | ----- |
| `AIRY_EOK`       | 0   | 成功    |
| `AIRY_EINVAL`    | -1  | 无效参数  |
| `AIRY_ENOMEM`    | -2  | 内存不足  |
| `AIRY_ENOSYS`    | -3  | 未实现   |
| `AIRY_EPERM`     | -4  | 权限不足  |
| `AIRY_ENOENT`    | -5  | 资源不存在 |
| `AIRY_EAGAIN`    | -6  | 重试    |
| `AIRY_EMSGSIZE`  | -7  | 消息过大  |
| `AIRY_EBADF`     | -8  | 描述符错误 |
| `AIRY_EBUSY`     | -9  | 资源繁忙  |
| `AIRY_ENOTSUP`   | -10 | 不支持   |
| `AIRY_ETIMEDOUT`  | -11 | 超时    |
| `AIRY_ECONFLICT` | -12 | 状态冲突  |

**使用规范**：遵循 主流 Linux 发行版 工程规范，使用 goto 风格错误处理、`WARN_ON_ONCE()` 而非 `BUG()`。

### 12.2 形式化验证预留

借鉴 seL4 的形式化验证（ES-SEL4-15\~20），agentrt-linux 预留以下验证接口：

- **syscall 契约**：借鉴 seL4 syscall.xml（ES-SEL4-21），R-01 增强建议（纳入 1.0.1 M1）。
- **Capability 安全属性**：access control / confinement / integrity 形式化验证预留。
- **IPC 协议正确性**：消息头 magic 验证 + 端点状态机验证。

***

## 13. agentrt-linux 工程基线与版本规划

### 13.1 内核基线

- **1.x.x 版本阶段**：基于 Linux 6.6 内核（ADR-013）。
- **2.x.x 版本阶段**：升级至 Linux 7.1 内核。

### 13.2 工程标准

- **代码风格**：完全遵循 主流 Linux 发行版 工程规范——Tab=8 缩进、K\&R 大括号、snake\_case 命名、CAPITALIZED 宏/枚举、`__u32/__u16/__u64` 用于 UAPI、`u32/u16/u64` 用于内核内部、kernel-doc 注释、goto 错误处理。
- **设计思想**：参考 seL4 微内核设计思想——capability 类型分发、端点 IPC 三态模型、最小错误码、直接 IPC 缓冲区拷贝。
- **命名统一**：所有 \[SC] 共享标识符使用 `AIRY_*` 前缀，\[IND] 内部标识符使用 `airy_*` 前缀。

### 13.3 版本规划

- **0.1.1**：文档体系完成（当前版本）。
- **1.0.1**：开发版本（直接过渡，无 0.1.2/0.2.0/0.3.0/1.0.0）。

***

## 14. 与其他子仓的协作

### 14.1 子仓级协作

| 子仓                      | 协作方式                                         | 内核切入点                         |
| ----------------------- | -------------------------------------------- | ----------------------------- |
| `services`    | 用户态服务通过 IPC 与内核通信                            | io\_uring IPC ring + Endpoint |
| `security`    | 共享 \[SC] `security_types.h` + LSM 钩子         | capability 系统 + Cupolas       |
| `memory`      | 共享 \[SC] `memory_types.h` + MemoryRovol      | `airy_sys_rovol_ctl`          |
| `cognition`   | 共享 \[SC] `cognition_types.h` + CoreLoopThree | `airy_sys_clt_notify`         |
| `cloudnative` | 可观测性集成（OpenTelemetry）                        | eBPF ringbuf + kfunc          |
| `system`      | 系统配置 + 启动管理                                  | sysfs + procfs 接口             |
| `tests-linux` | 内核测试 + 基准测试                                  | `tests-linux/benchmark/`      |

### 14.2 12 daemon 内核切入点（v1.1 Capability Folding 后）

AirymaxOS 用户态 **12 daemon** 在内核侧的切入点（daemon 命名后缀统一为 `_d`，**无例外**，v2.0 决策 C1；12 daemon 完整名单以 [10-user-supervisor-daemon.md §1.3](10-user-supervisor-daemon.md) 为 SSoT）：

| Daemon | 职责 | 内核切入点 | syscall / 通道 |
| --- | --- | --- | --- |
| `sec_d` | capability 编译/撤销 + Badge 生命周期 + LSM_ctl + Wasm_load | `security/airy/airy_lsm.c` + `agent_caps[1024]` 静态数组写者 | `airy_sys_call`（编号 0）+ airy_lsm LSM 钩子 |
| `cogn_d` | 认知循环调度（CoreLoopThree：PERCEPT/THINK/ACT） | kthread 注册 + CoreLoopThree 阶段通知 | `airy_sys_clt_notify`（编号 3）|
| `mem_d` | 记忆卷载管理（MemoryRovol L1-L4 快照/恢复/迁移） | userfaultfd + MGLRU + CXL bus | `airy_sys_rovol_ctl`（编号 1）|
| `sched_d` | sched_tac 策略守护（stc_realtime/interactive/agent/batch） | sched_tac 策略框架 + cgroup cpuset | `airy_sys_sched_ctl`（编号 2）|
| `logger_d` | 统一日志（128B 记录 + 5 级枚举） | printk bridge + char dev `/dev/airy_log` | char dev + \[SC] `log_types.h` |
| `audit_d` | 审计哈希链（`AIRY_FAULT_AUDIT_TAMPER=0x100B` 检测） | eBPF ringbuf 上报审计事件 | eBPF ringbuf + kfunc |
| `gateway_d` | 跨节点 IPC（分布式 Agent 通信） | io_uring ring + 网络栈 | io_uring + gRPC/QUIC |
| `macro_d` | 宏观监管（系统级看门狗 + daemon 重启） | systemd watchdog + panic 回调 | systemd watchdog |
| `vfs_d` | VFS 用户态化（元数据操作下放） | io_uring + 上游 `fs/` 改造 | io_uring + VFIO |
| `net_d` | 网络栈用户态化（协议处理下放） | io_uring + 上游 `net/` 改造 | io_uring + VFIO |
| `dev_d` | 设备驱动用户态化 | io_uring + 上游 `drivers/` 改造 | io_uring + VFIO |
| `config_d` | 统一配置管理（运行时参数热更新） | sysfs + procfs 接口 | sysfs + procfs |

**daemon 与 syscall 编号映射**（v1.1 Capability Folding 后）：

- 编号 0 `airy_sys_call` → sec_d 独占（Badge 编译/撤销 + LSM_ctl + Wasm_load）
- 编号 1 `airy_sys_rovol_ctl` → mem_d 独占（记忆卷载控制）
- 编号 2 `airy_sys_sched_ctl` → sched_d 独占（调度策略配置）
- 编号 3 `airy_sys_clt_notify` → cogn_d 独占（CoreLoopThree 通知 + kthread 注册）
- 编号 4-23 预留（未来 daemon 扩展）

**说明**：
- logger_d / audit_d / gateway_d / macro_d / vfs_d / net_d / dev_d / config_d 共 8 个 daemon **不直接占用 syscall 槽位**，通过 io_uring 数据面 / char dev / eBPF ringbuf / sysfs 等通道与内核协作。
- sec_d 是 v1.1 Capability Folding 的**核心枢纽**：唯一 `agent_caps[1024]` 写者，所有 Badge 编译/撤销请求通过 `airy_sys_call` 串行化处理（令牌桶限流 + 50ms SLO）。

***

## 15. 里程碑（M0-M8）

| 里程碑 | 内容                              | 依赖子仓                        | 时间        |
| --- | ------------------------------- | --------------------------- | --------- |
| M0  | 文档体系完成（本模块设计文档）                 | —                           | 2026-07   |
| M1  | 内核核心（调度 + IPC + capability）     | kernel                      | 第 1-2 周   |
| M2  | io\_uring IPC 数据面 + 控制面 syscall | kernel + services           | 第 3-4 周   |
| M3  | sched\_tac 策略守护进程（基于原生调度类组合） | kernel                      | 第 5-6 周   |
| M4  | 安全子系统（capability + LSM）         | kernel + security           | 第 7-8 周   |
| M5  | 记忆卷载 + 认知通知                     | kernel + memory + cognition | 第 9-10 周  |
| M6  | 微内核化改造 + Rust 驱动                | kernel + services           | 第 11-12 周 |
| M7  | 性能优化 + 基准测试                     | kernel                      | 第 13-14 周 |
| M8  | 生产就绪 + 稳定性验证                    | kernel + services           | 第 15-16 周 |

### 15.1 0.1.1 版本范围

仅完成 M0（文档体系完成）+ M1（[SC] 共享契约层头文件占位）。不含内核/OS 代码实施。

### 15.2 1.0.1 版本范围

完成 M2-M8 全部里程碑，并实施内核工程标准。

***

## 16. agentrt 一致性检查

### 16.1 命名前缀一致性

| 维度           | agentrt      | agentrt-linux（本文档） | 一致性 |
| ------------ | ------------ | ------------------ | --- |
| \[SC] 共享宏    | `AIRY_*`     | `AIRY_*`           | 一致  |
| \[SC] 共享结构体  | `airy_*`     | `airy_*`           | 一致  |
| \[SC] 共享枚举   | `airy_*`     | `airy_*`           | 一致  |
| syscall C 符号 | `airy_sys_*` | `airy_sys_*`       | 一致  |
| 错误码          | `AIRY_E*`    | `AIRY_E*`          | 一致  |
| API 导出宏      | `AIRY_API`   | `AIRY_API`         | 一致  |

### 16.2 结构体一致性

| 结构体                     | agentrt                                         | agentrt-linux                                   | 一致性 |
| ----------------------- | ----------------------------------------------- | ----------------------------------------------- | --- |
| `airy_task_desc`        | 4 字段（magic/prio/\_pad/vtime）                    | 4 字段（magic/prio/\_pad/vtime）                    | 一致  |
| `airy_ipc_msg_hdr`      | 128B，Layout C v4（magic/opcode/flags/trace\_id/timestamp\_ns/src\_task/dst\_task/capability\_badge/payload\_len/crc32/reserved，11 字段，capability\_badge offset 40） | 128B，Layout C v4（同左，capability\_badge offset 40） | 一致（H1 硬约束） |
| `airy_struct_ops_value` | state + common                                  | state + common                                  | 一致  |
| `airy_struct_ops_state` | INIT/REGISTERED/ACTIVE/DRAINING                 | INIT/REGISTERED/ACTIVE/DRAINING                 | 一致  |

### 16.3 常量一致性

| 常量                    | agentrt                      | agentrt-linux                | 一致性 |
| --------------------- | ---------------------------- | ---------------------------- | --- |
| `AIRY_IPC_MAGIC`      | 0x41524531                   | 0x41524531                   | 一致  |
| `AIRY_TASK_MAGIC`     | 0x41475453                   | 0x41475453                   | 一致  |
| `AIRY_IPC_HDR_SIZE`     | 128                          | 128                          | 一致  |
| `AIRY_SLICE_DFL`   | 20                           | 20                           | 一致  |
| `AIRY_PRIO_MIN/MAX`   | 0/139                        | 0/139                        | 一致  |
| `AIRY_WEIGHT_MIN/MAX` | 1/10000                      | 1/10000                      | 一致  |
| `AIRY_IPC_OP_*`       | SEND/RECV/SEND\_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE (0x0001~0x0011) | 同左（v1.1 Capability Folding 7 opcode） | 一致  |
| `AIRY_SYS_*`          | 0-3（4 核心，v1.1）              | 0-3（4 核心，v1.1）              | 一致  |

### 16.4 Capability 操作一致性

| 操作     | agentrt | agentrt-linux | 一致性 |
| ------ | ------- | ------------- | --- |
| Copy   | <br />  | <br />        | 一致  |
| Mint   | <br />  | <br />        | 一致  |
| Move   | <br />  | <br />        | 一致  |
| Mutate | <br />  | <br />        | 一致  |
| Revoke | <br />  | <br />        | 一致  |
| Delete | <br />  | <br />        | 一致  |
| Rotate | <br />  | <br />        | 一致  |

***

## 17. 相关文档

- [01-syscalls.md（系统调用接口）](../30-interfaces/01-syscalls.md)
- [02-ipc-protocol.md（IPC 协议）](../30-interfaces/02-ipc-protocol.md)
- [120-cross-project-code-sharing.md（SSoT）](../50-engineering-standards/120-cross-project-code-sharing.md)
- [04-scheduling-flow.md（调度数据流）](../40-dataflows/04-scheduling-flow.md)
- [01-cognition-flow.md（认知数据流）](../40-dataflows/01-cognition-flow.md)
- [03-ipc-performance.md（IPC 性能）](../170-performance/03-ipc-performance.md)
- [02-services.md（服务子仓）](02-services.md)
- [03-security.md（安全子仓）](03-security.md)
- [04-memory.md（记忆子仓）](04-memory.md)
- [05-cognition.md（认知子仓）](05-cognition.md)
- [ADR-012（微内核化改造技术路线确认）](../10-architecture/05-adrs.md#adr-012)
- [ADR-013（Linux 6.6 基线决策）](../10-architecture/05-adrs.md#adr-013)
- [ADR-014（微内核设计思想来源单一化——仅 seL4）](../10-architecture/05-adrs.md#adr-014)

***

## 18. 参考文献

| 编号     | 文献                                            | 说明                     |
| ------ | --------------------------------------------- | ---------------------- |
| REF-01 | seL4 Manual                                   | seL4 微内核官方手册           |
| REF-02 | seL4 Whitepaper (Klein et al., 2009)          | seL4 形式化验证论文           |
| REF-03 | Liedtke, J. (1995) "On μ-Kernel Construction" | 微内核极简原则原始论文            |
| REF-04 | Linux 6.6 Documentation                       | Linux 内核文档             |
| REF-05 | Linux SCHED\_DEADLINE/SCHED\_FIFO 文档        | Linux 6.6 实时调度策略文档 |
| REF-06 | io\_uring Documentation                       | Linux io\_uring 接口文档   |
| REF-07 | Linux 6.6 内核基线 Source                         | 主流 Linux 发行版 LTS 内核源码  |

***

© 2025-2026 SPHARX Ltd. All Rights Reserved.
