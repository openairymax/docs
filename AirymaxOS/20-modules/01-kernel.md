Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 内核设计文档（Airymax Kernel 极境内核）

> **子仓编号**：01\
> **子仓代号**：极境内核（Airymax Kernel）\
> **文档版本**：v3.0 2026-07-11\
> **设计基准**：Linux 内核基线 + 微内核化改造\
> **同源 agentrt**：atoms/corekern（MicroCoreRT）\
> **理论基础**：seL4 微内核工程思想\
> **技术路线**：基于 Linux 内核微内核化改造，非从零开发微内核\
> **核心约束**：IRON-9 v2 同源且部分代码共享——与 agentrt 用户态 atoms/corekern 通过 \[SC] 共享契约层（6 头文件）+ \[SS] 语义同源层协作，\[IND] 内核态 sched\_ext/io\_uring/eBPF/Rust 实现独立。**Syscall 架构**：12 核心 + 12 预留 = 24 槽位（Capability Invocation 统一入口 + IPC 数据面/控制面分离）。**命名统一**：所有 \[SC] 共享标识符使用 `AIRY_*` 前缀，C 函数使用 `airy_sys_*` 前缀。\
> **横切关注点**：内核是横切关注点（cross-cutting concern），贯穿调度、IPC、eBPF、记忆卷载 4 大数据流，提供机制骨架

***

## 目录

- [1. 子仓职责与设计哲学](#1-子仓职责与设计哲学)
- [2. 同源关系（IRON-9 v2 三层共享模型）](#2-同源关系iron-9-v2-三层共享模型)
- [3. 目录结构](#3-目录结构)
- [4. 内核对象模型（seL4 借鉴）](#4-内核对象模型sel4-借鉴)
- [5. Capability 系统（seL4 借鉴）](#5-capability-系统sel4-借鉴)
- [6. IPC 消息传递（seL4 借鉴 + io\_uring 落地）](#6-ipc-消息传递sel4-借鉴--io_uring-落地)
- [7. 调度与线程模型](#7-调度与线程模型)
- [8. 内存管理（seL4 借鉴 + Linux 对接）](#8-内存管理sel4-借鉴--linux-对接)
- [9. 核心特性（Linux 6.6 原生）](#9-核心特性linux-66-原生)
- [10. IRON-9 v2 三层共享模型落地](#10-iron-9-v2-三层共享模型落地)
- [11. Boot 流程](#11-boot-流程)
- [12. 错误处理与形式化验证预留](#12-错误处理与形式化验证预留)
- [13. agentrt-linux 工程基线与版本规划](#13-agentrt-linux-工程基线与版本规划)
- [14. 与其他子仓的协作](#14-与其他子仓的协作)
- [15. 里程碑（M1-M6）](#15-里程碑m1-m6)
- [16. agentrt 一致性检查](#16-agentrt-一致性检查)
- [17. 相关文档](#17-相关文档)
- [18. 参考文献](#18-参考文献)

***

## 1. 子仓职责与设计哲学

`kernel` 是 agentrt-linux（AirymaxOS）的内核子仓，承担以下核心职责：

1. **Linux 6.6 内核维护 \[IND]**：基于 Linux 6.6 内核基线（1.x.x 版本锁定，ADR-013），保持与上游社区同步演进。
2. **微内核化改造 \[IND]**：在保留 Linux 6.6 完整能力的前提下，遵循 Liedtke minimality principle（ES-SEL4-01, 04），将 VFS、网络栈、设备驱动等子系统逐步用户态化，最小化特权态代码体积。
3. **Agent 感知调度（sched\_ext）\[SS]**：通过 sched\_ext（agentrt-linux 内核增强，主线 6.12+ 引入并持续演进）实现 SCHED\_AGENT 策略，允许 eBPF 程序在用户态定义调度策略。任务描述符与优先级语义 \[SC] 与 agentrt 共享。
4. **高性能 IPC 基础（io\_uring）\[SS]**：基于 io\_uring 构建零 syscall、零拷贝的消息传递基础设施。IPC 消息头与操作码语义 \[SC] 与 agentrt 共享。
5. **eBPF 可编程扩展 \[SS]**：提供 struct\_ops 注册机制 + kfunc 扩展 + ringbuf 上报，可观测/网络/安全/调度均可通过 eBPF 编程。struct\_ops 状态机与 common\_value 布局与 agentrt 共享（补充共享文件 `bpf_struct_ops.h`，非 \[SC] 核心头文件）。
6. **Rust 安全驱动 \[IND]**：依托 Linux 6.6 中 Rust 实验性支持（持续演进中），构建安全驱动开发框架。
7. **\[SC] 共享契约层物理宿主 \[IND]**：IRON-9 v2 全部 6 个 \[SC] 共享契约头文件物理宿主在 `kernel/include/airymax/`，其他子仓通过 `-I` 引用确保单一数据源。
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
| \[SC] 共享契约层 | —         | 6 个头文件（IRON-9 v2）     | 单一物理宿主，禁止重复定义         |

### 1.2 内核代码体量预算

借鉴 seL4 的体量控制哲学（ES-SEL4-01），agentrt-linux 设定以下代码体量预算：

| 代码区域                         | seL4 行数  | agentrt-linux 预算 | 性质              |
| ---------------------------- | -------- | ---------------- | --------------- |
| 内核核心（调度/IPC/capability/内存原语） | \~14,400 | 5-10 万行          | 真正的"微内核化"必需     |
| 微内核化改造补丁（VFS/网络/驱动用户态化）      | —        | ≤ 2 万行           | 渐进式改造           |
| \[SC] 共享契约层                  | —        | 6 个头文件           | IRON-9 v2 单一数据源 |
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

**agentrt-linux 目标**：**12 核心 + 12 预留 = 24 槽位**（见 [01-syscalls.md](../30-interfaces/01-syscalls.md)），SSoT 定义于 `syscalls.h`：

| 编号    | 调用名                   | 分类     | 说明                                                |
| ----- | --------------------- | ------ | ------------------------------------------------- |
| 0     | `airy_sys_call`       | IPC 原语 | 统一 capability invocation（seL4 Call，含 LSM/Wasm 加载） |
| 1     | `airy_sys_send`       | IPC 原语 | 阻塞同步发送                                            |
| 2     | `airy_sys_recv`       | IPC 原语 | 阻塞同步接收                                            |
| 3     | `airy_sys_nbsend`     | IPC 原语 | 非阻塞发送                                             |
| 4     | `airy_sys_nbrecv`     | IPC 原语 | 非阻塞接收                                             |
| 5     | `airy_sys_reply_recv` | IPC 原语 | 回复并等待下一条                                          |
| 6     | `airy_sys_yield`      | IPC 原语 | 让出 CPU                                            |
| 7     | `airy_sys_rovol_ctl`  | 控制原语   | 记忆卷载控制                                            |
| 8     | `airy_sys_sched_ctl`  | 控制原语   | 调度策略配置                                            |
| 9     | `airy_sys_clt_notify` | 控制原语   | CoreLoopThree 通知 + kthread 注册                     |
| 10    | `airy_sys_reply`      | IPC 原语 | 独立回复（seL4 Reply，不等待下一条）                           |
| 11    | `airy_sys_notify`     | 通知原语   | 异步通知信号（seL4 Notification）                         |
| 12-23 | 预留                    | —      | 未来扩展                                              |

**数据面 I/O 完全由 io\_uring 处理（零 syscall）**。LsmCtl 与 WasmLoad 通过 capability invocation 归入 `airy_sys_call`。`airy_sys_reply` 完齐 seL4 master 模式 8 个 activity syscall（不引入 MCS 模式，详见 `30-interfaces/01-syscalls.md` §2.2）。`airy_sys_notify` 提供 agent 间异步事件通知（seL4 Notification 二进制信号量语义）。

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

- **Syscall**：12 核心 = seL4 8 IPC 原语 + agent 领域 3 控制原语 + 1 通知原语，无冗余
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
| 调度数据流        | sched\_ext BPF 调度类 + SCHED\_AGENT + sub-scheduler                      | \[SS]  |
| IPC 数据流（控制面） | 8 个 IPC 原语 syscall（Call/Send/Recv/NBSend/NBRecv/ReplyRecv/Yield/Reply） | \[SS]  |
| IPC 数据流（数据面） | io\_uring 零拷贝 ring + registered buffer + MSG\_RING                     | \[SS]  |
| eBPF 数据流     | struct\_ops 注册 + kfunc + ringbuf + verifier                            | \[SS]  |
| 记忆卷载数据流      | userfaultfd + MGLRU + CXL bus（与 memory 协作）                   | \[IND] |

***

## 2. 同源关系（IRON-9 v2 三层共享模型）

依据 IRON-9 v2 决策，agentrt（用户态 atoms/corekern）与 agentrt-linux（内核态 kernel）通过三层共享模型协作：

| 层次               | 共享程度               | 内核子系统内容                                                                                                                                                                                                                                                                                 | 组织方式                              |
| ---------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| **\[SC] 共享契约层**  | 完全共享代码             | 6 个头文件（详见 §10.1）：syscalls.h / ipc.h / sched.h / security\_types.h / memory\_types.h / cognition\_types.h                                                                                                                                                                                | `kernel/include/airymax/`（单一物理宿主） |
| **\[SS] 语义同源层**  | 高层 API 语义同源，签名独立演进 | sched\_ext 25+ BPF 回调、io\_uring ring 创建/提交/完成/注册、MSG\_RING 跨环消息、SQPOLL 状态机、DEFER\_TASKRUN、eBPF struct\_ops 注册、bpf\_prog 生命周期、bpf\_link 生命周期、bpf\_map\_ops 回调表、ringbuf reserve/submit、kfunc 注册模式 等 30+ 项                                                                                 | 各自独立实现                            |
| **\[IND] 完全独立层** | 完全独立               | ext\_sched\_class 注册、scx\_ops\_enable/disable、kf\_mask 上下文追踪、fallback\_dsq 回退、cgroup 集成、core-sched 集成、debug dump；io-wq 工作队列、NO\_MMAP、REGISTERED\_FD\_ONLY、URING\_CMD；JIT 后端、trampoline 本机码生成、verifier 实现、BPF\_SCHED CFS 钩子（不移植）、cfi\_stubs、KABI\_RESERVE（不采用）；VFS/网络/驱动用户态化改造；Rust 驱动框架 | 各自独立仓库                            |

### 2.1 维度对比

| 维度             | agentrt（atoms/corekern）         | agentrt-linux（kernel）               | 同源标注   |
| -------------- | ------------------------------- | --------------------------------------------- | ------ |
| 设计目标           | RT 微核心 + Agent 调度               | Linux 6.6 + 微内核化 + Agent 调度                   | \[SS]  |
| 调度模型           | MicroCoreRT 实时调度                | SCHED\_AGENT（sched\_ext + eBPF）               | \[SS]  |
| 任务描述符          | `struct airy_task_desc`（用户态）    | `struct airy_task_desc`（内核态）                  | \[SC]  |
| IPC            | 用户态消息队列                         | io\_uring 零拷贝 IPC（数据面）+ 7 IPC 原语 syscall（控制面） | \[SS]  |
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
- 保留 agentrt 的"实时性"目标（通过 sched\_ext 的 sub-scheduler 在 cgroup 上附加实时策略）\[SS]。
- 保留 agentrt 的"消息传递"通信范式（升级为 io\_uring 零拷贝实现 + 8 IPC 原语控制面）\[SS]。
- 任务描述符与优先级语义 \[SC] 共享，确保两端调度语义一致。
- IPC 消息头与操作码 \[SC] 共享，确保两端通信协议一致。
- struct\_ops 状态机 \[SC] 共享，使 agentrt 用户态可解析 BPF 调度器在线状态。
- capability 派生模型 \[SC] 共享（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate），确保两端安全语义一致。
- MemoryRovol L1-L4 数据结构 \[SC] 共享，确保两端记忆卷载语义一致。
- Syscall 编号体系 \[SC] 共享（syscalls.h），确保两端 syscall 语义一致。

***

## 3. 目录结构

```
kernel/
├── linux/                 # Linux 6.6 内核源码（Linux 6.6 内核基线，git subtree）[IND]
├── patches/               # agentrt-linux 内核补丁
│   ├── sched_ext-agent/   # SCHED_AGENT 策略（eBPF 程序）[SS]
│   ├── io_uring-ipc/      # 基于 io_uring 的 IPC 优化 [SS]
│   ├── bpf-struct-ops/    # eBPF struct_ops 扩展 [SS]
│   ├── capability/        # capability 系统内核接口（seL4 借鉴）[SS]
│   ├── rust-drivers/      # Rust 安全驱动 [IND]
│   └── microkernel/       # 微内核化改造（VFS/网络/驱动部分用户态化）[IND]
├── include/
│   └── airymax/           # [SC] 共享契约层 6 头文件物理宿主（单一数据源）
│       ├── syscalls.h         # Syscall 编号体系 [SC]
│       ├── ipc.h              # IPC 契约 [SC]
│       ├── sched.h            # 调度契约 [SC]
│       ├── memory_types.h     # 记忆卷载契约（与 memory 子仓共享）[SC]
│       ├── security_types.h   # 安全契约（与 security 子仓共享）[SC]
│       └── cognition_types.h  # 认知契约（与 cognition 子仓共享）[SC]
├── configs/               # 内核配置 [IND]
│   ├── defconfig          # 默认配置
│   ├── defconfig-agent    # Agent 优化配置
│   └── defconfig-micro    # 微内核化配置
├── docs/                  # 设计文档
└── tests/                 # 内核测试
```

### 3.1 patches/sched\_ext-agent \[SS]

存放 SCHED\_AGENT 策略的 eBPF 程序源码。任务描述符与优先级 \[SC] 与 agentrt 共享：

- `sched_agent.bpf.c`：核心调度器逻辑（基于 sched\_ext BPF 接口）\[SS]。
- `sub-schedulers/`：按 cgroup 附加的子调度器（实时型、批处理型、交互型、Agent 认知型）\[SS]。
- `tools/`：用户态控制器（airymaxctl 命令行工具）\[IND]。

### 3.2 patches/io\_uring-ipc \[SS]

基于 io\_uring 构建 IPC 通道的内核侧实现。IPC 消息头与操作码 \[SC] 与 agentrt 共享：

- `io_uring_ipc.c`：注册固定 OP，支持零拷贝消息传递 \[SS]。
- `ring_register.c`：跨进程 ring 共享注册 \[SS]。
- `zerocopy.c`：基于 MSG\_ZEROCOPY 与 page flipping 的零拷贝路径 \[SS]。

### 3.3 patches/bpf-struct-ops \[SS]

eBPF struct\_ops 扩展，struct\_ops 状态机与 common\_value \[SC] 与 agentrt 共享：

- `agent_struct_ops.c`：struct\_ops 注册机制扩展 \[SS]。
- `agent_kfunc.c`：自定义 kfunc 导出（`bpf_agent_decision_get` 等）\[IND]。
- `agent_ringbuf.c`：ringbuf 事件格式与 agentrt AgentsIPC 128B 消息头同源 \[SC]。

### 3.4 patches/capability \[SS]

capability 系统的内核侧接口，借鉴 seL4 capability 模型（ES-SEL4-05\~09）：

- `cte.c`：CTE（Capability Table Entry）管理 \[SS]。
- `cspace.c`：CSpace 树形寻址（guard + radix）\[SS]。
- `mdb.c`：MDB（Memory Disclosure Base）派生树维护 \[SS]。
- `cnode_ops.c`：7 种 CNode 操作（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate）\[SS]。
- 与 `security/capability` 协作，通过 \[SC] `security_types.h` 共享契约。

### 3.5 patches/rust-drivers \[IND]

Rust 安全驱动框架：

- `rust/kernel/`：扩展的 Rust 内核绑定。
- `samples/rust_drivers/`：示例安全驱动（网卡、块设备、字符设备）。
- `frameworks/`：抽象框架（trait-based 驱动模型）。

### 3.6 patches/microkernel \[IND]

微内核化改造补丁集：

- `vfs-userns/`：VFS 元数据操作下放至用户态服务。
- `net-userns/`：网络栈用户态化（与 `services/net` 协作）。
- `driver-split/`：设备驱动拆分至用户态（与 `services/drivers` 协作）。
- `untyped-retype/`：Untyped-Retype 内存模型内核接口（借鉴 ES-SEL4-03）。

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
- **Suspend/Resume**：通过 sched\_ext 的 `agent_runnable`/`agent_stopping` 回调控制。
- **Delete**：回收对象，内存 Retype 回 Untyped。

***

## 5. Capability 系统（seL4 借鉴）

> **设计依据**：seL4 capability 单一安全模型（ES-SEL4-05 至 09），是 seL4 安全模型的精髓。代码证据 `src/object/cnode.c`（934 行）、`src/kernel/cspace.c`（193 行）、`include/object/structures_64.bf`。

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

**\[SC] 对接**：通过 `include/airymax/security_types.h` 定义 capability 派生模型（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate），与 agentrt Cupolas 同源。

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

这消除了独立 syscall 的需求（task\_submit/cancel/status/capability\_request/revoke/lsm\_ctl/wasm\_load），将 syscall 数量从 12 缩减到 10。

***

## 6. IPC 消息传递（seL4 借鉴 + io\_uring 落地）

> **设计依据**：seL4 IPC 消息传递机制（ES-SEL4-10 至 15）。代码证据 `src/object/endpoint.c`（577 行）、`src/fastpath/fastpath.c`（899 行）、`src/object/notification.c`（419 行）。

### 6.1 IPC 双平面架构

agentrt-linux 的 IPC 采用 **数据面/控制面分离** 架构：

| 平面      | 路径                                | 延迟      | 通信方式               | 用途                                |
| ------- | --------------------------------- | ------- | ------------------ | --------------------------------- |
| **控制面** | 用户态 → syscall → 内核 → 返回           | \~1 μs  | 7 个 IPC 原语 syscall | capability invocation、同步 IPC、策略设置 |
| **数据面** | 用户态 → SQE → io\_uring → CQE → 用户态 | \~10 μs | io\_uring 零拷贝 ring | 高频消息收发、记忆迁移、流式数据                  |

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

**\[SC] 对接**：`struct airy_ipc_msg_hdr`（128B）定义在 `include/airymax/ipc.h`，包含 magic（0x41524531 'ARE1'）、opcode、flags、trace\_id、timestamp\_ns、src\_task、dst\_task、payload\_len、reserved\[84] 等字段。与 SSoT 逐字节一致。

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

**控制面（7 个 IPC 原语 syscall）**：

| 编号 | syscall               | seL4 对应          | 说明                       |
| -- | --------------------- | ---------------- | ------------------------ |
| 0  | `airy_sys_call`       | `seL4_Call`      | 统一 capability invocation |
| 1  | `airy_sys_send`       | `seL4_Send`      | 阻塞同步发送                   |
| 2  | `airy_sys_recv`       | `seL4_Recv`      | 阻塞同步接收                   |
| 3  | `airy_sys_nbsend`     | `seL4_NBSend`    | 非阻塞发送                    |
| 4  | `airy_sys_nbrecv`     | `seL4_NBRecv`    | 非阻塞接收                    |
| 5  | `airy_sys_reply_recv` | `seL4_ReplyRecv` | 回复并接收下一条                 |
| 6  | `airy_sys_yield`      | `seL4_Yield`     | 让出 CPU                   |

**数据面（io\_uring 零拷贝）**：

| io\_uring 机制              | seL4 对应           | 同源标注  |
| ------------------------- | ----------------- | ----- |
| SQ/CQ ring 共享内存           | IPC Buffer        | \[SS] |
| SQE 内联数据                  | Message Register  | \[SS] |
| MSG\_RING 跨 ring 消息       | Endpoint send     | \[SS] |
| registered buffer         | 固定 cap 映射         | \[SS] |
| zero-copy (page flipping) | fastpath transfer | \[SS] |

**IPC magic 0x41524531 'ARE1'** \[SC] 与 agentrt 共享，确保两端通信协议一致。

***

## 7. 调度与线程模型

> **设计依据**：seL4 调度器设计（ES-SEL4-04 零策略内核）+ Linux 6.6 sched\_ext。代码证据 `src/kernel/thread.c`（752 行）、`src/object/tcb.c:32-50`（checkPrio）、`src/object/schedcontext.c`（444 行）。

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
| `AIRY_SLICE_DFL_MS` | 20    | 默认时间片（ms）    |
| `MAC_MAX_AGENTS`    | 1024  | 并发 Agent 硬上限 |

**注意**：vtime 使用 Q16.16 定点数（`int32_t`），因内核态禁止浮点运算（`-mno-80387`）。agentrt 用户态可使用 `float`，但跨 \[SC] 边界统一使用 Q16.16。

### 7.3 调度上下文（MCS 模式参考）

**seL4 MCS 模式**（`src/object/schedcontext.c` 444 行 + `src/object/schedcontrol.c` 186 行）：

| 维度           | seL4 MCS 实现   | agentrt-linux 落地              |
| ------------ | ------------- | ----------------------------- |
| SchedContext | 常量带宽管理        | sched\_ext sub-scheduler 带宽配额 |
| refill 机制    | refill + 带宽管理 | cgroup cpu.max/cpu.weight     |
| 常量带宽 vs 突发   | 参数化           | SCHED\_AGENT 策略可配             |

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

### 9.1 sched\_ext（SCHED\_EXT=7）

- **SCHED\_EXT=7**（Linux 6.6 内核基线 内置，`include/uapi/linux/sched.h:121`）：agentrt-linux 复用 SCHED\_EXT=7，**禁止**定义 `SCHED_AGENT` 宏（避免与内核调度类编号冲突）。SCHED\_AGENT 仅作为 sched\_ext 策略名称。
- SCHED\_AGENT 策略通过 eBPF struct\_ops 加载，实现 Agent 感知调度。
- 通过 `airy_sys_sched_ctl` 控制策略配置。

### 9.2 io\_uring

- 零 syscall、零拷贝 IPC 数据面。
- 固定 buffer + registered ring + page flipping。
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
- **SCHED\_EXT=7** 复用，禁止定义 SCHED\_AGENT 宏。

***

## 10. IRON-9 v2 三层共享模型落地

本节详细列出 agentrt-linux 内核中 IRON-9 v2 三层共享模型的具体落地细节。所有 \[SC] 定义以 SSoT（`120-cross-project-code-sharing.md`）为唯一权威来源。

### 10.1 \[SC] 共享契约层——6 个头文件

#### 10.1.1 sched.h — 调度契约

| 共享内容          | 符号                      | 值/类型            | 说明                                                                      |
| ------------- | ----------------------- | --------------- | ----------------------------------------------------------------------- |
| 任务描述符 magic   | `AIRY_TASK_MAGIC`       | `0x41475453u`   | 'AGTS'（Agent Task State）                                                |
| 任务描述符结构       | `struct airy_task_desc` | 4 字段            | magic(\_\_u32) + prio(\_\_u16) + \_pad(\_\_u16) + vtime(airy\_vtime\_t) |
| vtime 类型      | `airy_vtime_t`          | `int32_t`       | Q16.16 定点数                                                              |
| vtime 衰减函数    | `airy_vtime_decay()`    | inline          | vtime + (consumed\_slice \* 100 / weight)                               |
| 优先级范围         | `AIRY_PRIO_MIN/MAX`     | 0 / 139         | 兼容 Linux 优先级                                                            |
| 权重范围          | `AIRY_WEIGHT_MIN/MAX`   | 1 / 10000       | 兼容 sched\_ext 权重模型                                                      |
| 默认时间片         | `AIRY_SLICE_DFL_MS`     | 20              | 20ms                                                                    |
| SCHED\_EXT 约束 | —                       | 复用 SCHED\_EXT=7 | 禁止定义 SCHED\_AGENT 宏                                                     |

#### 10.1.2 ipc.h — IPC 契约

| 共享内容      | 符号                                                   | 值/类型              | 说明                                                                                                                                                                                     |
| --------- | ---------------------------------------------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IPC magic | `AIRY_IPC_MAGIC`                                     | `0x41524531u`     | 'ARE1'（Airymax Runtime Engine v1）                                                                                                                                                      |
| 消息头大小     | `AIRY_IPC_HDR_SZ`                                    | 128               | 128 字节                                                                                                                                                                                 |
| 消息头结构     | `struct airy_ipc_msg_hdr`                            | 8 字段              | magic(\_\_u32) + opcode(\_\_u16) + flags(\_\_u16) + trace\_id(\_\_u64) + timestamp\_ns(\_\_u64) + src\_task(\_\_u64) + dst\_task(\_\_u64) + payload\_len(\_\_u32) + reserved[84](__u8) |
| 消息标志      | `AIRY_IPC_F_NOWAIT/SIGNAL`                           | (1u<<0) / (1u<<1) | 非阻塞 / 信号                                                                                                                                                                               |
| IPC 操作码   | `AIRY_IPC_OP_SEND/RECV/SEND_BATCH/CANCEL`            | 0/1/2/3           | 4 个操作码                                                                                                                                                                                 |
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
| LSM 钩子        | `AIRY_LSM_HOOK_TASK_CREATE/IPC_SEND`        | 0/1      | 252 个 LSM 钩子 ID                                                      |
| 策略裁决          | `airy_verdict`                              | 4 枚举     | ALLOW=0 / DENY=1 / AUDIT=2 / ASK=3                                   |
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
| Syscall 架构 | —                                                        | 12 核心 + 12 预留 = 24 槽位 | 8 IPC 原语 + 3 控制原语 + 1 通知原语 |
| 编号 0-6     | `AIRY_SYS_CALL/SEND/RECV/NBSEND/NBRECV/REPLY_RECV/YIELD` | 0-6                   | 7 个 IPC 原语                 |
| 编号 7-9     | `AIRY_SYS_ROVOL_CTL/SCHED_CTL/CLT_NOTIFY`                | 7-9                   | 3 个控制原语                    |
| 编号 10-11   | `AIRY_SYS_REPLY/NOTIFY`                                  | 10-11                 | 1 个 IPC 原语 + 1 个通知原语       |
| 编号 12-23   | 预留                                                       | 12-23                 | 未来扩展                       |

### 10.2 \[SS] 语义同源层——30+ 项

| 语义域        | agentrt 实现                   | agentrt-linux 实现                    | \[SC] 契约依据                              |
| ---------- | ---------------------------- | ----------------------------------- | --------------------------------------- |
| 调度语义       | MicroCoreRT 用户态调度器           | sched\_ext BPF 调度器（SCHED\_EXT=7）    | `sched.h`：任务描述符 + vtime + 优先级           |
| 安全模型       | Cupolas 用户态策略引擎              | LSM 钩子（252 ID）+ capability 41 ID    | `security_types.h`：capability + verdict |
| IPC 传输     | 用户态消息队列（mqueue/io\_uring）    | 内核 io\_uring 驱动（数据面）+ 8 IPC 原语（控制面） | `ipc.h`：128B 消息头 + magic 'ARE1'         |
| 记忆模型       | 用户态 heapstore（malloc + mmap） | 内核态 L1-L4 分层（kmalloc + pmem）        | `memory_types.h`：L1-L4 + GFP 掩码         |
| 认知模型       | 用户态 CoreLoopThree 引擎         | 内核 kthread 认知通知                     | `cognition_types.h`：三阶段枚举               |
| Syscall 编号 | 用户态 `syscalls.h` 宏定义         | 内核 `syscall_64.tbl` 表项              | `syscalls.h`：24 槽位编号                    |

### 10.3 \[IND] 完全独立层——15+ 项

| 独立项            | agentrt 实现          | agentrt-linux 实现   | 独立原因        |
| -------------- | ------------------- | ------------------ | ----------- |
| 调度类            | 用户态协程/线程池           | sched\_ext 内核调度类注册 | 跨平台 vs 内核专属 |
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
2. **agentrt-linux 子系统初始化**：在 `initcall` 阶段初始化 io\_uring IPC、eBPF struct\_ops、capability 系统、sched\_ext 调度类。
3. **Agent 服务启动**：用户态 init 进程启动 Agent 服务管理器（services），注册 SCHED\_AGENT BPF 程序。
4. **Capability 引导**：根 Agent 获得初始 CSpace，开始 capability 分发。
5. **进入 Agent 调度**：SCHED\_AGENT 策略接管 Agent 任务调度。

***

## 12. 错误处理与形式化验证预留

### 12.1 错误码体系

错误码统一使用 `AIRY_E*` 前缀，SSoT 定义于 `airy_errno.h`：

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
| `AIRY_ETIMEOUT`  | -11 | 超时    |
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

| 子仓                      | 协作方式                                         | 内核切入点                         |
| ----------------------- | -------------------------------------------- | ----------------------------- |
| `services`    | 用户态服务通过 IPC 与内核通信                            | io\_uring IPC ring + Endpoint |
| `security`    | 共享 \[SC] `security_types.h` + LSM 钩子         | capability 系统 + Cupolas       |
| `memory`      | 共享 \[SC] `memory_types.h` + MemoryRovol      | `airy_sys_rovol_ctl`          |
| `cognition`   | 共享 \[SC] `cognition_types.h` + CoreLoopThree | `airy_sys_clt_notify`         |
| `cloudnative` | 可观测性集成（OpenTelemetry）                        | eBPF ringbuf + kfunc          |
| `system`      | 系统配置 + 启动管理                                  | sysfs + procfs 接口             |
| `tests-linux` | 内核测试 + 基准测试                                  | `tests-linux/benchmark/`      |

***

## 15. 里程碑（M1-M6）

| 里程碑 | 内容                              | 依赖子仓                        | 时间        |
| --- | ------------------------------- | --------------------------- | --------- |
| M1  | 内核核心（调度 + IPC + capability）     | kernel                      | 第 1-2 周   |
| M2  | io\_uring IPC 数据面 + 控制面 syscall | kernel + services           | 第 3-4 周   |
| M3  | eBPF struct\_ops + SCHED\_AGENT | kernel                      | 第 5-6 周   |
| M4  | 安全子系统（capability + LSM）         | kernel + security           | 第 7-8 周   |
| M5  | 记忆卷载 + 认知通知                     | kernel + memory + cognition | 第 9-10 周  |
| M6  | 微内核化改造 + Rust 驱动                | kernel + services           | 第 11-12 周 |

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
| `airy_ipc_msg_hdr`      | 128B，Layout C（opcode/flags/src\_task/dst\_task） | 128B，Layout C（opcode/flags/src\_task/dst\_task） | 一致  |
| `airy_struct_ops_value` | state + common                                  | state + common                                  | 一致  |
| `airy_struct_ops_state` | INIT/REGISTERED/ACTIVE/DRAINING                 | INIT/REGISTERED/ACTIVE/DRAINING                 | 一致  |

### 16.3 常量一致性

| 常量                    | agentrt                      | agentrt-linux                | 一致性 |
| --------------------- | ---------------------------- | ---------------------------- | --- |
| `AIRY_IPC_MAGIC`      | 0x41524531                   | 0x41524531                   | 一致  |
| `AIRY_TASK_MAGIC`     | 0x41475453                   | 0x41475453                   | 一致  |
| `AIRY_IPC_HDR_SZ`     | 128                          | 128                          | 一致  |
| `AIRY_SLICE_DFL_MS`   | 20                           | 20                           | 一致  |
| `AIRY_PRIO_MIN/MAX`   | 0/139                        | 0/139                        | 一致  |
| `AIRY_WEIGHT_MIN/MAX` | 1/10000                      | 1/10000                      | 一致  |
| `AIRY_IPC_OP_*`       | SEND/RECV/SEND\_BATCH/CANCEL | SEND/RECV/SEND\_BATCH/CANCEL | 一致  |
| `AIRY_SYS_*`          | 0-11（12 核心）                  | 0-11（12 核心）                  | 一致  |

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
- [ADR-012（微内核化改造决策）](../50-engineering-standards/adrs/012-microkernel.md)
- [ADR-013（Linux 6.6 基线决策）](../50-engineering-standards/adrs/013-linux-6.6-baseline.md)

***

## 18. 参考文献

| 编号     | 文献                                            | 说明                     |
| ------ | --------------------------------------------- | ---------------------- |
| REF-01 | seL4 Manual                                   | seL4 微内核官方手册           |
| REF-02 | seL4 Whitepaper (Klein et al., 2009)          | seL4 形式化验证论文           |
| REF-03 | Liedtke, J. (1995) "On μ-Kernel Construction" | 微内核极简原则原始论文            |
| REF-04 | Linux 6.6 Documentation                       | Linux 内核文档             |
| REF-05 | sched\_ext Documentation                      | Linux sched\_ext 调度类文档 |
| REF-06 | io\_uring Documentation                       | Linux io\_uring 接口文档   |
| REF-07 | Linux 6.6 内核基线 Source                         | 主流 Linux 发行版 LTS 内核源码  |

***

© 2025-2026 SPHARX Ltd. All Rights Reserved.
