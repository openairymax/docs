Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# agentrt-linux 统一术语表
> **文档定位**：agentrt-linux 统一术语表\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **作者**：开源极境工程与规范委员会（OpenAirymax Engineering and Standardization Committee）

***

## 一、编制说明

本术语表是 agentrt-linux（AirymaxOS）规范体系的**唯一权威术语参考**。

**目的**:

- 统一所有 agentrt-linux（AirymaxOS）规范文档、代码注释、API 文档中的术语定义
- 当不同文档间存在术语冲突时，**以本表为准**
- 确保与 agentrt（AirymaxAgentRT）的术语在 IRON-9 v3 \[SC] 共享契约层保持一致

**使用规则**:

1. 编写文档时必须使用本表的**标准名称**
2. 核心表述统一使用 "agentrt-linux（AirymaxOS）" 形式，不得单独出现 "AirymaxOS" 而无前缀
3. 标注为"禁止使用"的旧称不得出现在任何新的规范文档或代码注释中
4. 新增术语需经架构团队审核后加入
5. 标准计算机专业名词（如 IPC、Daemon、Sandbox、Microkernel 等）遵循业界通用定义，本表不重新定义，仅在"标准计算机名词"分区中说明其在 agentrt-linux（AirymaxOS）中的使用上下文
6. 当标准名词与 agentrt-linux 特有概念同名时，以 agentrt-linux 特有定义为准
7. 与 agentrt 术语表中共享的术语，通过 IRON-9 v3 \[SC] 层标注保持一致性

***

## 二、快速查阅对照表（团队速查）

> **本节定位**：为团队提供一站式快速查阅，涵盖所有层级名称、缩写、枚举值和错误码。详细定义见后续章节。

### 2.1 调度体系术语对照

| 层级 | 标准名称 | 类型 | 说明 |
|------|---------|------|------|
| 框架名 | `sched_tac` | 标识符 | SCHED_DEADLINE + SCHED_FIFO + EEVDF + seL4 MCS 映射 |
| 缩写 | `STC` | 缩写 | 策略性调度（sched + tactical） |
| 策略枚举 | `stc_realtime` | 枚举值 | 优先级 0-49，<1ms，具身智能运动控制 |
| 策略枚举 | `stc_interactive` | 枚举值 | 优先级 50-99，<10ms，用户对话补全 |
| 策略枚举 | `stc_agent` | 枚举值 | 优先级 100-119，<100ms，CoreLoopThree 思考 |
| 策略枚举 | `stc_batch` | 枚举值 | 优先级 120-139，<1s，LLM 批量推理 |
| 字符串常量 | `AIRY_STC_POLICY_NAME "stc_agent"` | 宏 | 用户态策略标识字符串 |
| 系统调用前缀 | `AIRY_SYS_SCHED_*` | 编号段 | 572-591，sched_tac 策略管理 |
| **禁止使用** | ~~`AIRY_SCHED_AGENT`~~ | 废弃 | 2026-07-18 彻底废弃 |
| **禁止使用** | ~~`AIRY_SCHED_AGENT_NAME`~~ | 废弃 | 已改为 `AIRY_STC_POLICY_NAME` |
| **禁止使用** | ~~`sched_ext`~~ | 废弃 | 6.6 主线不含，已由 sched_tac 替代 |
| **禁止使用** | ~~`SCHED_AGENT` 宏~~ | 废弃 | 禁止定义内核调度类编号宏 |
| **禁止使用** | ~~`方案 C-Prime`~~ | 废弃 | 已由 sched_tac 替代 |

### 2.2 安全体系术语对照

| 层级 | 标准名称 | 类型 | 说明 |
|------|---------|------|------|
| 架构概念 | `Cupolas`（安全穹顶） | 概念名 | agentrt-linux 安全体系的架构概念 |
| 内核模块名 | `airy` | LSM 名称 | `DEFINE_LSM(airy)`，`LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位 |
| 源码目录 | `security/airy/` | 路径 | 纯 C LSM 模块源码 |
| 用户态 API 前缀 | `airy_cupolas_*` | 函数前缀 | agentrt 用户态 Cupolas API（如 `airy_cupolas_guard_enter()`） |
| 内核态 API 前缀 | `airy_*` | 函数前缀 | 内核态 LSM 钩子函数（如 `airy_lsm_init()`） |
| 裁决枚举 | `AIRY_VERDICT_ALLOW = 0` | 枚举 | 允许操作 |
| 裁决枚举 | `AIRY_VERDICT_DENY = 1` | 枚举 | 拒绝操作 |
| 裁决枚举 | `AIRY_VERDICT_AUDIT = 2` | 枚举 | 允许但审计 |
| 裁决枚举 | `AIRY_VERDICT_COMPLAIN = 3` | 枚举 | 拒绝但记录（学习模式，对齐 AppArmor complain） |
| blob 结构体 | `airy_cred_security` 等 | 结构体 | LSM blob 安全上下文 |
| **禁止使用** | ~~`DEFINE_LSM(cupolas)`~~ | 废弃 | 已改为 `DEFINE_LSM(airy)` |
| **禁止使用** | ~~`CUPOLAS_VERDICT_*`~~ | 废弃 | 已改为 `AIRY_VERDICT_*` |
| **禁止使用** | ~~`AIRY_VERDICT_ASK`~~ | 废弃 | 已改为 `AIRY_VERDICT_COMPLAIN` |
| **禁止使用** | ~~`BPF LSM`~~ | 废弃 | 采用纯 C LSM，对齐 openEuler |

### 2.3 Unify Design 5 模块术语对照

| 模块 | 全称 | 缩写 | 导出前缀 | [SC] 头文件 |
|------|------|------|---------|-----------|
| 统一错误与故障定义体系 | Unified Error and Fault Framework | A-UEF | `airy_err_*` | `error.h` |
| 统一日志与打印系统 | Unified Logging and Printk Subsystem | A-ULP | `airy_log_*` | `log_types.h` |
| 统一计算与调度体系 | Unified Computing and Scheduling | A-UCS | — | `cognition_types.h` |
| 统一生命周期管理 | Unified Lifecycle Management | A-ULS | — | `sched.h` |
| 统一进程间通信体系 | Unified Inter-Process Communication | A-IPC | `airy_ipc_*` | `ipc.h` |

### 2.4 IRON-9 v3 四层模型术语对照

| 层次 | 标注 | 全称 | 共享程度 |
|------|------|------|---------|
| 共享契约层 | [SC] | Shared Contract | 完全共享代码（10 个头文件） |
| 语义同源层 | [SS] | Semantic Sharing | 高层 API 语义同源，签名独立演进 |
| 完全独立层 | [IND] | Independent | 完全独立实现 |
| 降级生存层 | [DSL] | Degraded Survival Layer | 自包含降级块（`#ifdef AIRY_SC_FALLBACK`） |

### 2.5 [SC] 10 个共享契约头文件

| # | 头文件 | 模块归属 | magic / 标识 |
|---|--------|---------|-------------|
| 1 | `error.h` | A-UEF | 错误码 + 故障码 |
| 2 | `log_types.h` | A-ULP | 日志级别 + 128B 记录 |
| 3 | `ipc.h` | A-IPC | magic 0x41524531 'ARE1' |
| 4 | `sched.h` | A-ULS | magic 0x41475453 'AGTS'（任务描述符） |
| 5 | `memory_types.h` | MemoryRovol | L1-L4 四层记忆 |
| 6 | `security_types.h` | 安全 | capability 41 ID + LSM 钩子 |
| 7 | `cognition_types.h` | A-UCS | CoreLoopThree + Agent 8 态生命周期 |
| 8 | `syscalls.h` | 系统调用 | 24 槽位 |
| 9 | `uapi_compat.h` | 桥接 | 三路类型桥接 |
| 10 | `lsm_types.h` | 安全 | 纯 C LSM 类型 |

### 2.6 调度相关错误码对照

| 错误码 | 值 | 名称 | 说明 |
|--------|-----|------|------|
| -203 | `E_AIRY_STC_LOAD_FAIL` | sched_tac 加载失败 | 用户态调度器加载失败 |
| -204 | `E_AIRY_STC_UNLOAD_FAIL` | sched_tac 卸载失败 | 用户态调度器卸载失败 |
| -205 | `E_AIRY_STC_ATTACH_FAIL` | sched_tac 附加失败 | 调度器附加到 CPU 失败 |
| -206 | `E_AIRY_STC_DETACH_FAIL` | sched_tac 分离失败 | 调度器从 CPU 分离失败 |
| -207 | `E_AIRY_STC_VERIFY_FAIL` | sched_tac 验证失败 | 调度器验证失败 |
| **禁止使用** | ~~`E_AIRY_SCHED_*`~~ | 废弃 | 已改为 `E_AIRY_STC_*` |

### 2.7 IPC 与任务描述符 magic 对照

| magic 值 | ASCII | 用途 | 独立性 |
|---------|-------|------|--------|
| `0x41524531` | 'ARE1' | IPC 消息头 | 独立于任务描述符 |
| `0x41475453` | 'AGTS' | 任务描述符 | 独立于 IPC 消息头 |

### 2.8 内核基线与许可证术语对照

| 术语 | 标准名称 | 说明 |
|------|---------|------|
| 内核基线 | Linux 6.6 LTS | 1.x.x 版本锁定（ADR-013） |
| 未来基线 | Linux 7.1 | 2.x.x 版本规划 |
| 内核 fork 模型 | 模型 A（完整 Linux 6.6 fork） | 直接写入上游源码树 |
| 许可证（内核） | GPL-2.0-only | kernel 子仓 |
| 许可证（开源核心） | AGPL-3.0-or-later OR Apache-2.0 | 双许可证，用户二选一 |
| 许可证（文档） | CC-BY-NC-4.0 | 非商业用途 |
| 许可证（开放标准） | CC-BY-4.0 | 自由使用 |
| 许可证（MemoryRovol） | SPHARX Commercial EULA v1.0 | 闭源商业 |

***

## 三、标准计算机名词（OS 领域通用术语）

本分区所列术语均为计算机科学领域，特别是**操作系统领域**的业界通用专业名词，遵循其标准定义，agentrt-linux（AirymaxOS）**不重新定义**。仅说明其在 agentrt-linux 项目中的使用上下文，供文档编写与代码注释参考。

### IPC / 进程间通信

**业界定义**: Inter-Process Communication，操作系统提供的机制，允许不同进程之间交换数据与信号。常见实现包括管道、消息队列、共享内存、信号量、套接字、io\_uring 等。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）微内核化改造的核心机制之一，基于 io\_uring 零拷贝消息传递（`IORING_OP_URING_CMD` + registered buffer + mmap，**不使用 page flipping**），承载用户态服务（Daemon）之间、Agent 与内核之间的高性能消息传递。128 字节统一消息头（magic=0x41524531）与 agentrt 的 AgentsIPC 同源（IRON-9 v3 \[SS] 语义同源层）。

**系统内代码**: `airy_ipc_*`

**参见**: MicroCoreRT（微核心运行时）、AgentsIPC（Agent 进程间通信）、io\_uring 异步 I/O、A-IPC（统一进程间通信体系）

***

### Daemon / 用户态服务进程

**业界定义**: 在后台运行的长生命周期进程，通常独立于终端会话，提供特定服务。Unix 传统命名以 `d` 后缀（如 `syslogd`、`sshd`）。在 systemd 体系下，daemon 作为 systemd 单元管理。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）采用微内核化改造策略，将功能服务化为用户态进程，通过 systemd 集成管理。命名遵循 `*_d` 约定（如 `gateway_d`、`cogn_d`、`dev_d`、`sched_d`、`audit_d`、`sec_d`、`mem_d`、`vfs_d`、`net_d`）或 `*_daemon`/`macro_superv` 约定（`logger_daemon`、`config_daemon`、`macro_superv`）。12 个 daemon 服务覆盖认知、安全、记忆、调度、监管等全部功能域。

**系统内代码**: `*_d` 进程后缀，systemd unit 文件

**参见**: IPC、Service Isolation（服务隔离）、systemd 集成、Macro-Supervisor（用户温情裁决）

***

### Sandbox / 沙箱

**业界定义**: 一种安全机制，将程序运行限制在受控的隔离环境中，防止其对宿主系统造成意外或恶意影响。常见实现包括 seccomp、Landlock、namespace、capability 等。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）系统调用层（syscall）的七大功能域之一，为 Agent 与 Skill 提供隔离执行环境。沙箱机制结合 Linux 6.6 内核基线的 seccomp + Landlock + capability 三道防线，配合 Cupolas 共同构成安全防护体系。

**系统内代码**: `airy_sys_sandbox_*`

**参见**: Cupolas（安全穹顶）、Service Isolation（服务隔离）、Security by Design（安全内生设计）

***

### Microkernel / 微内核

**业界定义**: 操作系统的最小核心，只提供最基本的服务（调度、IPC、地址空间管理、基本内存管理）。文件系统、网络栈、设备驱动等服务运行在用户态。代表实现：seL4、Zircon、Minix3、QNX、L4。

> **ADR-014 约束**： agentrt-linux 微内核设计思想**唯一来源为 seL4**，不引入 Zircon / Minix3 / 其他微内核架构。业界定义中列出其他实现仅作术语参考，不代表 agentrt-linux 采纳其设计思想。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）基于 Linux 6.6 内核（ALK 6.6）进行**微内核化改造**（非从零开发），演进目标为真正的微内核——利用sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射）+ io\_uring + 纯 C LSM 实现微内核化，将 VFS、网络栈、部分驱动移到用户态，同时保留 Linux 内核的稳定性和硬件兼容性。遵循 Liedtke minimality principle。**零内核调度器修改**（不使用 sched\_ext，6.6 主线不含 sched\_ext）。

**参见**: Liedtke Minimality Principle、seL4、MicroCoreRT（微核心运行时）、sched_tac

***

### Liedtke Minimality Principle

**业界定义**: Jochen Liedtke（L4 微内核创始人）提出的微内核设计原则："A concept is tolerated inside the microkernel only if moving it outside the kernel, i.e., permitting competing implementations, would prevent the implementation of the system's required functionality."

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）的内核极简原则（五维正交24原则 K-1）直接遵循 Liedtke minimality——内核只保留不可再分的原子机制（调度、IPC、内存管理、时间服务），一切可以在用户态实现的功能都必须在用户态实现。

**参见**: K-1 内核极简原则、MicroCoreRT（微核心运行时）、五维正交24原则

***

### capability（操作系统安全）

**业界定义**: 一种访问控制机制，使用不可伪造的令牌（capability）代表对资源的访问权限。进程必须持有指向目标资源的 capability 才能进行访问。capability 可以被委托、复制、限制。代表实现：seL4、Zircon。

> **ADR-014 约束**： agentrt-linux capability 模型**唯一来源为 seL4**，不引入 Zircon handle 模型。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）安全子系统（security）实现 capability 系统（POSIX capability 41 ID + seL4 风格派生模型），与 Cupolas 同源。capability 令牌格式定义于 `include/uapi/linux/airymax/security_types.h`（IRON-9 v3 \[SC] 共享契约层），结合**纯 C LSM 模块**（对齐 openEuler 252 钩子，**不使用 BPF LSM**）实现纵深防御。

**系统内代码**: `airy_cap_*`

**参见**: Cupolas（安全穹顶）、纯 C LSM 模块、LSM（Linux Security Module）、Security by Design（安全内生设计）

***

### io\_uring / 异步 I/O

**业界定义**: Linux 内核的高性能异步 I/O 接口，由 Jens Axboe 设计。通过共享内存环形缓冲区（submission queue + completion queue）实现零拷贝、低延迟的异步 I/O 操作，支持 read/write/send/recv/accept/connect 等操作。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）IPC 子系统的核心传输机制，基于 io\_uring 的 `IORING_OP_URING_CMD` 操作码 + registered buffer + mmap 实现零拷贝消息传递（**不使用 page flipping**）。内核固定 OP 码用于 Agent IPC 消息收发，与 agentrt 的 AgentsIPC 语义同源（IRON-9 v3 \[SS] 语义同源层）。

**参见**: IPC、AgentsIPC（Agent 进程间通信）、IORING_OP_URING_CMD、sched_tac、A-IPC

***

### sched\_ext / 可扩展调度器

**业界定义**: Linux 内核的可扩展调度器框架（主线 6.12+），允许通过 eBPF 程序在用户态定义调度策略，实现自定义调度类。提供 struct\_ops 机制将内核调度器行为暴露给 eBPF 程序。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）**不使用 sched\_ext**——Linux 6.6 内核基线（ALK 6.6）主线不含 sched\_ext，向前移植会破坏内核稳定性。agentrt-linux 改为采用**sched_tac**（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射），实现三层用户态调度架构，**零内核调度器修改**。`stc_agent` 是用户态调度策略枚举（非内核调度类编号），通过原生调度类（SCHED_FIFO/SCHED_DEADLINE/EEVDF）+ cgroup cpuset 隔离实现 Agent 工作负载优先级，调度语义与 agentrt 的 MicroCoreRT 一致（同源）。

**禁止**: 引用 sched\_ext 作为 agentrt-linux 调度方案；定义 `SCHED_AGENT` 内核调度类宏（避免与内核调度类编号冲突）。

**参见**: sched_tac、stc_* 策略枚举、EEVDF、MicroCoreRT（微核心运行时）

***

### eBPF

**业界定义**: Extended Berkeley Packet Filter，Linux 内核的可编程虚拟机。允许在沙箱环境中运行用户提供的程序，安全地扩展内核功能。支持 kfunc（内核函数）、dynamic pointer、struct\_ops 等高级特性。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）利用 eBPF 实现内核可观测性探针（kfunc + dynamic pointer）。**安全策略不使用 BPF LSM**，改用**纯 C LSM 模块**（对齐 openEuler）。调度策略不使用 sched\_ext + eBPF struct\_ops，改用sched_tac（用户态函数表 `struct airy_sched_ops`，非 BPF struct\_ops）。所有 eBPF 程序通过 Cupolas 安全穹顶进行权限校验。

**参见**: sched_tac、stc_* 策略枚举、纯 C LSM 模块、Cupolas（安全穹顶）

***

### MGLRU / 多代 LRU

**业界定义**: Multi-Generation LRU，Linux 内核 6.6 引入的页面回收算法。将内存页面按访问热度分为多代（generation），优先回收最冷代页面，实现冷热数据分层和高效内存回收。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）内核内存管理的核心算法，配合 CXL 内存池化实现 MemoryRovol L1-L4 四层记忆的冷热数据分层。MGLRU 的多代回收策略与记忆卷载的遗忘机制（Forgetting Engine）在语义上一致。

**参见**: MemoryRovol（记忆卷载）、Forgetting Engine（遗忘机制）、CXL 内存池化

***

### Linux Security Module (LSM)

**业界定义**: Linux 内核的安全框架，提供钩子（hook）机制允许安全模块（如 SELinux、AppArmor）在关键操作点进行访问控制。LSM 钩子覆盖文件系统、网络、进程管理、capability 等子系统。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）安全子系统（security）利用 LSM 框架实现 Agent 专属安全策略，采用**纯 C LSM 模块**（`security/airy/`，对齐 openEuler 252 钩子，**不使用 BPF LSM**）。LSM 钩子 ID 定义于 `include/uapi/linux/airymax/security_types.h` 与 `include/uapi/linux/airymax/lsm_types.h`（IRON-9 v3 \[SC] 共享契约层），与 capability 系统共同构成纵深防御。

**参见**: capability、纯 C LSM 模块、Cupolas（安全穹顶）、SELinux

***

### systemd

**业界定义**: Linux 系统的初始化系统和服务管理器，负责系统引导、服务管理、日志记录、设备管理等。提供 unit 文件定义服务依赖、启动顺序和资源限制。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）采用 systemd 管理 12 个用户态守护进程。每个 daemon 通过 systemd unit 文件定义隔离策略（独立地址空间、capability 限制、资源限制）。systemd 服务名采用 `agentrt-*.service` 格式（独立），二进制名与 agentrt 同源。遵循 K-3 服务隔离原则（五维正交24原则）。

**参见**: Daemon（用户态服务进程）、Service Isolation（服务隔离）、Macro-Supervisor（用户温情裁决）

***

### Security by Design / 安全内生设计

**业界定义**: 一种工程理念，将安全作为系统设计的首要原则，在架构层面而非附加层面解决安全问题。与"安全左移"（Shift Left Security）理念一致，强调默认安全（Secure by Default）。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）的核心设计理念之一（五维正交24原则 E-1），贯穿内核（capability + 纯 C LSM）、服务层（沙箱 + 隔离）、记忆层（审计哈希链）等所有安全相关组件。

**参见**: Cupolas（安全穹顶）、capability、纯 C LSM 模块、LSM、Service Isolation（服务隔离）

***

### TraceID / 跟踪标识符

**业界定义**: 分布式追踪系统中用于唯一标识一次请求链路的标识符，贯穿请求经过的所有服务节点，支持链路分析与性能诊断。常见于 OpenTelemetry、Jaeger、Zipkin 等可观测性体系。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）可观测性子系统的核心标识，贯穿 Agent 任务执行、IPC 消息传递、Daemon 服务调用的全链路。128 字节 IPC 消息头中固定携带 TraceID 字段，与 agentrt 的 TraceID 体系完全一致（IRON-9 v3 \[SC] 共享契约层）。

**参见**: IPC、OpenTelemetry、可观测性、A-ULP（统一日志与打印系统）

***

### Service Isolation / 服务隔离

**业界定义**: 一种架构模式，将不同服务的运行环境、资源、权限相互隔离，防止单一服务故障或被攻破后影响整体系统。常见实现包括进程隔离、容器隔离、虚拟机隔离等。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）微内核化改造的设计原则之一（五维正交24原则 K-3），各 Daemon 服务运行于独立地址空间，通过 IPC 通信，资源与权限相互隔离。systemd unit 文件定义隔离边界，与 Sandbox、Cupolas 协同构成纵深防御。

**参见**: Daemon（用户态服务进程）、Sandbox（沙箱）、Cupolas（安全穹顶）、systemd

***

### JSON-RPC 2.0

**业界定义**: JSON Remote Procedure Call，一种无状态、轻量级的远程过程调用协议，使用 JSON 作为数据格式。规范定义于 <https://www.jsonrpc.org/specification。>

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）协议适配层支持的标准协议之一，用于 Agent 与外部客户端之间的请求-响应通信。使用强制命名空间格式 `<daemon>.<method>`，错误码体系与 agentrt 统一错误码体系（A-UEF）对接。

**参见**: IPC、AgentsIPC（Agent 进程间通信）、A-UEF（统一错误码与故障定义体系）

***

### CXL / Compute Express Link

**业界定义**: 一种开放标准的互连协议，基于 PCIe 物理层，支持 CPU 与加速器、内存扩展器之间的高速通信。CXL 2.0+ 支持内存池化（memory pooling），允许多个主机共享内存资源。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）利用 ALK 6.6 内核基线的 CXL 支持实现 MemoryRovol L1-L4 跨节点内存池化，配合 MGLRU（多代 LRU）实现冷热数据分层。这是 agentrt-linux 作为 OS 发行版相对于 agentrt 用户态运行时的独特优势。

**系统内代码**: `airy_cxl_*`

**参见**: MGLRU（多代 LRU）、MemoryRovol（记忆卷载）、PMEM（持久内存）、ALK 6.6

***

### DPDK / AF\_XDP

**业界定义**: Data Plane Development Kit，用户态高性能数据包处理框架。AF\_XDP（Address Family eXpress Data Path）是 Linux 内核的原生高性能数据路径，允许用户态程序直接访问网卡接收环。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）网络栈用户态化的核心技术，通过 DPDK/AF\_XDP 将网络处理从内核态移到用户态（net\_d 守护进程），实现微内核化改造。遵循 K-1 内核极简原则（五维正交24原则）。

**系统内代码**: `airy_net_d`（net\_d 守护进程）

**参见**: Microkernel（微内核）、K-1 内核极简原则、Daemon（用户态服务进程）

***

### VFIO / libvfio

**业界定义**: Virtual Function I/O，Linux 内核的用户态驱动框架。VFIO 允许用户态程序直接访问设备 MMIO 区域和 DMA，实现高性能用户态驱动。libvfio 是 VFIO 的用户态库。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）驱动框架用户态化的核心技术，通过 VFIO/libvfio 将部分设备驱动从内核态移到用户态（dev\_d 守护进程，已合并原 drv\_d 职责），实现微内核化改造。遵循 K-1 内核极简原则（五维正交24原则）。

**系统内代码**: `airy_dev_d`（dev\_d 守护进程，原 drv\_d 已合并）

**参见**: Microkernel（微内核）、K-1 内核极简原则、Daemon（用户态服务进程）

***

## 三、agentrt-linux 特有架构术语

本分区术语为 agentrt-linux（AirymaxOS）特有概念，按 8 大分类组织：内核与调度、安全与 Capability、IPC 与通信、日志与错误码、生命周期与监管、跨端共享模型、工程标准与 CI、部署与运维。

### 3.1 内核与调度

#### agentrt-linux（AirymaxOS / 极境智能体操作系统）

**定义**: agentrt-linux 是基于 Linux 6.6 内核基线（ALK 6.6）的智能体操作系统发行版，采用微内核化改造策略（而非从零开发微内核）。代码仓库名 `agentrt-linux`，项目定位为 agentrt（AirymaxAgentRT）的 OS 层最佳载体。与 agentrt 遵循 IRON-9 v3"同源且部分代码共享"原则：\[SC] 共享契约层完全共享代码，\[SS] 语义同源层高层 API 语义同源（概念操作一致），签名因抽象层级不同而独立演进，\[IND] 完全独立层各自独立，\[DSL] 降级生存层提供 [SC] 损坏时的最小可运行子集。

**标准名称**: agentrt-linux（AirymaxOS）——正式文档中必须使用此完整形式，不得单独出现 "AirymaxOS" 而无 "agentrt-linux" 前缀。

**别名**: 极境智能体操作系统（中文完整名称）

**旧称/禁止使用**: 仅出现 "AirymaxOS" 而无 "agentrt-linux" 前缀的表述

**系统内代码**: `airy_*` 前缀（IRON-9 v3 \[SC] 共享契约层）

**代码目录**: `kernel/` / `services/` / `cognition/` / `memory/` / `security/` / `cloudnative/` / `system/` / `tests-linux/`（8 子仓）

**参见**: agentrt（AirymaxAgentRT）、IRON-9 v3、ALK 6.6、MicroCoreRT（微核心运行时）、五维正交24原则

***

#### ALK 6.6 / Airymax Linux Kernel 6.6

**定义**: ALK 6.6（Airymax Linux Kernel 6.6）是 agentrt-linux（AirymaxOS）1.0 版本锁定的**极境内核**，基于 Linux 6.6 内核基线进行微内核化改造。ALK 6.6 不引入 sched\_ext（6.6 主线不含），不引入 BPF LSM（采用纯 C LSM 对齐 openEuler），不引入 page flipping（采用 IORING_OP_URING_CMD + registered buffer），不引入 DMA 一致性内存用于日志/IPC（采用 alloc_pages + mmap）。内核配置通过 `airy_defconfig` fragment 定义。

**标准名称**: ALK 6.6（Airymax Linux Kernel 6.6，极境内核）

**参见**: Linux 6.6 内核基线、airy_defconfig、sched_tac、纯 C LSM 模块、IORING_OP_URING_CMD

***

#### Linux 6.6 内核基线

**定义**: agentrt-linux（AirymaxOS）1.0 版本锁定的内核基线版本，即 ALK 6.6 的基线。所有 agentrt-linux 内核态代码、驱动、安全模块均基于 Linux 6.6 内核基线开发。禁止引用 6.7+ 主线特性作为 6.6 原生能力（IRON-10 / BAN-361）。

**基线核心特性**:

- EEVDF 调度器（Linux 6.6 原生）
- Rust 实验性支持（Linux 6.6 原生）
- MGLRU 多代 LRU（Linux 6.6 原生）
- XFS 在线 fsck（Linux 6.6 原生）
- eBPF kfunc + dynamic pointer（Linux 6.6 原生）
- io\_uring 异步 I/O 改进（Linux 6.6 原生）
- **不引入** sched\_ext（主线 6.12+ 特性，6.6 不可用）

**标准名称**: Linux 6.6 内核基线（等同 ALK 6.6）

**禁止**: 引用 6.7+ 主线特性作为 6.6 原生能力；引入 sched\_ext 作为调度方案

**参见**: ALK 6.6、airy_defconfig、工程基线、IRON-10、BAN-361

***

#### airy_defconfig

**定义**: ALK 6.6 内核配置 fragment，定义 agentrt-linux（AirymaxOS）的内核编译选项。每个支持架构（x86\_64/aarch64/riscv64）必须有对应的 `airy_defconfig`，位于 `arch/$(SRCARCH)/configs/airy_defconfig`。`make defconfig` 读取它生成 `.config`。`airy_defconfig` 中可显式配置 `CONFIG_AIRY_SC_FALLBACK=y` 启用 [DSL] 降级模式。

**标准名称**: airy_defconfig（ALK 6.6 内核配置 fragment）

**参见**: ALK 6.6、Linux 6.6 内核基线、[DSL] 降级生存层、Kconfig 系统

***

#### sched_tac（STC，策略性调度）

**全称**: sched_tac（sched + tactical，调度 + 策略性）
**简称**: STC
**中文**: 策略性调度

**定义**: agentrt-linux（AirymaxOS）用户态调度器的核心调度机制选型，**替代 sched\_ext**（6.6 主线不含 sched\_ext）。sched_tac 基于 Linux 6.6 原生调度类 SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS（Mixed Criticality Systems）语义映射，实现**三层用户态调度架构**，**零内核调度器修改**。sched_tac 的核心思想是"策略性调度"——通过用户态调度策略表（`struct airy_sched_ops`）动态选择调度类，而非修改内核调度器。

**三层调度架构**:

| 层级 | 调度类 | 适用场景 | 语义映射 |
| ---- | ------ | -------- | -------- |
| L1 关键认知路径 | SCHED_FIFO（高 prio） | 安全审计、事务提交、关键决策 | seL4 SC 最高临界区 |
| L2 常量带宽任务 | SCHED_DEADLINE（CBS） | 实时 Agent 任务、确定性延迟保障 | seL4 SC 保障带宽 |
| L3 吞吐型任务 | SCHED_NORMAL（EEVDF）+ cpuset 独占核 | LLM 推理、批量处理 | seL4 SC 尽力而为 |

**核心约束**:

- **零内核调度器修改**：不新增内核调度类，不修改 CFS/EEVDF 核心代码
- **禁止**定义 `SCHED_AGENT` 内核调度类宏（避免与内核调度类编号冲突）
- stc_* 是用户态策略枚举，通过 `struct airy_sched_ops` 暴露回调（纯用户态函数表，非 BPF struct\_ops）
- [DSL] 降级模式下退化为 EEVDF 默认调度

**标准名称**: sched_tac（STC，策略性调度）—— SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射

**旧称/禁止使用**: sched\_ext + eBPF 调度方案、SCHED_AGENT 内核调度类、`AIRY_SCHED_AGENT`（2026-07-18 彻底废弃）

**参见**: stc_* 策略枚举、ALK 6.6、MicroCoreRT（微核心运行时）、seL4、[DSL] 降级生存层

***

#### stc_* / sched_tac 策略枚举（用户态调度策略）

**定义**: agentrt-linux（AirymaxOS）基于 sched_tac 框架的**用户态调度策略枚举**，专为 Agent 工作负载优化。stc_* 通过原生调度类（SCHED_FIFO/SCHED_DEADLINE/EEVDF）+ cgroup cpuset 隔离实现 Agent 工作负载优先级，调度语义与 agentrt 的 MicroCoreRT 一致（同源，IRON-9 v3 \[SS] 语义同源层）。

**4 种策略枚举**:

| 枚举值 | 优先级范围 | 延迟预算 | 典型场景 |
|--------|-----------|---------|---------|
| `stc_realtime` | 0-49 | < 1 ms | 具身智能运动控制 |
| `stc_interactive` | 50-99 | < 10 ms | 用户对话补全 |
| `stc_agent` | 100-119 | < 100 ms | CoreLoopThree 思考 |
| `stc_batch` | 120-139 | < 1 s | LLM 批量推理 |

**调度策略维度**:

- 认知阶段感知：Cognition Engine 阶段 Agent 获得更高优先级
- Token 预算感知：Token 预算即将耗尽的 Agent 获得调度加速
- 任务优先级：关键任务（安全审计、事务提交）获得实时调度保证
- 跨 Agent 协同：多 Agent 协作任务在相邻时间片内调度

**关键约束**: stc_* 是用户态策略枚举，**不是**内核调度类编号。禁止定义 `SCHED_AGENT` 内核调度类宏。通过 `struct airy_sched_ops` 暴露回调（纯用户态函数表，非 BPF struct\_ops）。策略字符串常量为 `AIRY_STC_POLICY_NAME "stc_agent"`。

**标准名称**: stc_realtime / stc_interactive / stc_agent / stc_batch（sched_tac 策略枚举）

**旧称/禁止使用**: `AIRY_SCHED_AGENT`（2026-07-18 彻底废弃，遵循 Airymax 工程思想五维分析）、`SCHED_AGENT`（易与内核调度类编号混淆）、SCHED_AGENT 内核调度类

**系统内代码**: `stc_*` 策略枚举，`AIRY_STC_POLICY_NAME` 字符串常量，用户态调度器 daemon（`airy-sched-daemon`）

**参见**: sched_tac、MicroCoreRT（微核心运行时）、CoreLoopThree（认知三阶段循环）、ALK 6.6

***

#### MicroCoreRT / 微核心运行时

**定义**: agentrt-linux（AirymaxOS）内核的极简运行时抽象层，提供最基本的 IPC、内存管理、任务调度、时间管理、可观测性功能。与 agentrt 的 MicroCoreRT 共享设计理念（同源），但在 agentrt-linux 中作为 OS 级实现（内核态 + 用户态服务），而在 agentrt 中作为用户态运行时实现。

**标准名称**: 微核心运行时 (MicroCoreRT)

**与 agentrt 的关系**: 遵循 IRON-9 v3 \[SS] 语义同源层——语义同源（调度语义、IPC 语义、内存管理语义概念一致），实现独立（agentrt-linux 基于sched_tac + io\_uring + MGLRU，agentrt 基于用户态消息队列 + 用户态调度器）。

**系统内代码**: `airy_core_*`

**代码目录**: `kernel/`

**参见**: K-1 内核极简原则、sched_tac、stc_* 策略枚举、Liedtke Minimality Principle

***

#### CoreLoopThree / 认知三阶段循环

**定义**: agentrt-linux（AirymaxOS）的三层认知-执行-记忆主循环运行时：

| 层级     | 名称               | 职责                           |
| ------ | ---------------- | ---------------------------- |
| L1 认知层 | Cognition Engine | 意图解析 -> 规划 -> 分发 -> 协调 -> 批判 |
| L2 执行层 | Execution Engine | 任务执行 -> 单元调度 -> 补偿事务         |
| L3 记忆层 | Memory Engine    | 结果写入 -> 查询 -> 挂载             |

在 agentrt-linux 中，认知层实现为内核 kthread（而非用户态线程），利用 `stc_agent` 策略（sched_tac）获得 OS 级调度优先级和确定性延迟。

**标准名称**: 认知三阶段循环 (CoreLoopThree)

**旧称/禁止使用**: "三层循环运行时"、"三层一体架构"、"三层循环核心"

**系统内代码**: `airy_loop_*`

**代码目录**: `cognition/`

**与 agentrt 的关系**: 遵循 IRON-9 v3 \[SS] 语义同源层——认知模型一致（CoreLoopThree 阶段枚举、Thinkdual 模式枚举、LLM 推理阶段枚举共享于 \[SC] 层），实现独立（agentrt-linux 基于内核 kthread + sched_tac 策略，agentrt 基于用户态线程）。

**参见**: Thinkdual（双思考系统）、MemoryRovol（记忆卷载）、sched_tac、stc_* 策略枚举

***

#### Thinkdual / 双思考系统

**定义**: agentrt-linux（AirymaxOS）的核心认知创新，由三组件构成的认知架构：

| 组件            | 角色     | 功能             |
| ------------- | ------ | -------------- |
| **t2 主思考**    | 慢思考    | 深度推理、反思调整、长期规划 |
| **t1-f 快思考**  | 快思考-事实 | 快速响应、流式验证、事实核查 |
| **t1-p 专业思考** | 快思考-专业 | 专业领域仲裁、深度质量评估  |

三组件通过 triple\_coordinator 协同工作。"双思"指深度思考（t2）与快速思考（t1）两大思维模式的协作。

**标准中文名**: 双思考系统（正式）/ Thinkdual（简称）
**标准英文名**: Thinkdual Cognitive Dual-Thinking System（完整）/ Thinkdual（简化）

**旧称/禁止使用**: "认知双思系统"、"双系统认知模型"、"Thinkdual 认知双思系统"、"Triple Coordinator"（triple\_coordinator 是协调器，非系统名称）

**系统内代码**: `tc_*`（思考链）/ `mc_*`（元认知）/ `sc_*`（流式验证）/ `tc3_*`（协调器）

**代码目录**: `cognition/`

**参见**: CoreLoopThree（认知三阶段循环）、五维正交24原则 C-1（认知层双思考功能）

***

#### MAC / 多智能体协作框架

**定义**: CoreLoopThree 认知层的多智能体协作基础设施，支持独立、协作、共识、委托四种协作模式，以及多数投票、全票通过、加权投票、领导者否决四种共识策略。

**标准名称**: 多智能体协作框架 (MAC)

**旧称/禁止使用**: "Multi-Agent System"（过于泛化）

**系统内代码**: `mac_*` / `airy_mac_*`

**参见**: Thinkdual（双思考系统）、CoreLoopThree（认知三阶段循环）

***

#### TaskFlow / 任务流引擎

**定义**: 基于 Pregel BSP（Bulk Synchronous Parallel）图计算模型的任务流引擎，提供超步计算、检查点容错、工作流模式等功能，支撑复杂任务的有向无环图（DAG）编排。

**标准名称**: 任务流引擎 (TaskFlow)

**代码目录**: `cognition/`

**参见**: CoreLoopThree（认知三阶段循环）

***

### 3.2 安全与 Capability

#### Cupolas / 安全穹顶

**定义**: agentrt-linux（AirymaxOS）的内生安全防护层，实现输入净化 -> 权限仲裁 -> 网络过滤 -> 审计记录四重防护链。在 agentrt-linux 中，Cupolas 结合 capability 系统（POSIX 41 ID + seL4 风格派生模型）、**纯 C LSM 模块**（对齐 openEuler 252 钩子，**不使用 BPF LSM**）、国密算法（SM2/SM3/SM4）和机密计算（TEE/SGX），构成 OS 级纵深防御体系。

**与 agentrt 的关系**: 遵循 IRON-9 v3 \[SS] 语义同源层——安全模型一致（capability 令牌格式、策略裁决 4 值枚举共享于 \[SC] 层），实现独立（agentrt-linux 基于纯 C LSM + capability 内核态，agentrt 基于用户态安全模块）。

**标准名称**: 安全穹顶 (Cupolas)

**系统内代码**: `cupolas_*`

**代码目录**: `security/`

**参见**: capability、纯 C LSM 模块、LSM、Security by Design（安全内生设计）、Sandbox（沙箱）

***

#### 纯 C LSM 模块

**定义**: agentrt-linux（AirymaxOS）的安全策略实现方式，采用**纯 C 编写的 LSM（Linux Security Module）模块**（位于 `security/airy/`），**对齐 openEuler 的 252 钩子实现**，**不使用 BPF LSM**。纯 C LSM 模块类型定义于 `include/uapi/linux/airymax/lsm_types.h`（IRON-9 v3 \[SC] 共享契约层），提供 `struct airy_lsm_blob` + `airy_capability_check()` 回调原型 + Capability 缓存结构。

**选型理由**:

- Linux 6.6 内核基线（ALK 6.6）稳定支持纯 C LSM
- 对齐 openEuler 发行版的 LSM 实现方式，便于上游同步
- 纯 C 实现避免 BPF LSM 的运行时开销与安全验证复杂度
- [DSL] 降级模式下退化为仅 POSIX capability 校验

**标准名称**: 纯 C LSM 模块（不使用 BPF LSM）

**旧称/禁止使用**: BPF LSM、eBPF 安全策略

**系统内代码**: `security/airy/`、`DEFINE_LSM(airy)` 骨架

**参见**: Cupolas（安全穹顶）、capability、LSM、Micro-Supervisor（内核冷酷执法）、ALK 6.6

***

#### Memory Safety Macro System / 安全宏体系

**定义**: agentrt-linux（AirymaxOS）的内存操作安全宏体系，覆盖分配、释放、字符串操作和禁止函数，确保内存操作的类型安全和空指针安全。适用于内核态和用户态代码。

**核心宏**:

- 分配: `AIRY_MALLOC` / `AIRY_CALLOC` / `AIRY_REALLOC`
- 释放: `AIRY_FREE` / `MEMORY_FREE_SAFE`
- 字符串: `AIRY_STRDUP` / `AIRY_STRNCPY_TERM`
- 拷贝: `AIRY_MEMCPY_SAFE`
- 清除: `AIRY_SEC_CLEAR`

**禁止函数**: `malloc`、`free`、`printf`、`strcpy`、`strcat`、`sprintf`、`gets` 等（完整列表见 `banned_functions.h`）

**参见**: Security by Design（安全内生设计）、MEMORY\_FREE\_SAFE

***

#### MEMORY\_FREE\_SAFE

**定义**: 安全释放宏。释放内存后将指针置 NULL，防止悬垂指针。

**签名**: `MEMORY_FREE_SAFE(ptr_to_ptr)` — 参数为指向待释放指针的指针

**参见**: Memory Safety Macro System（安全宏体系）

***

### 3.3 IPC 与通信

#### AgentsIPC / Agent 进程间通信

**定义**: agentrt-linux（AirymaxOS）的 Agent 间高性能通信机制，基于 io\_uring 的 `IORING_OP_URING_CMD` 操作码 + registered buffer + mmap 实现零拷贝消息传递（**不使用 page flipping**）+ 128 字节统一消息头（magic=0x41524531）。与 agentrt 的 AgentsIPC 语义同源（IRON-9 v3 \[SS] 语义同源层），但在 agentrt-linux 中基于内核 io\_uring 固定 OP 码实现，在 agentrt 中基于用户态消息队列实现。

**标准名称**: Agent 进程间通信 (AgentsIPC)

**系统内代码**: `airy_ipc_*`

**参见**: IPC、io\_uring、IORING_OP_URING_CMD、A-IPC（统一进程间通信体系）、128 字节消息头

***

#### IORING_OP_URING_CMD

**定义**: agentrt-linux（AirymaxOS）IPC 零拷贝机制的核心 io\_uring 操作码，配合 registered buffer + mmap 实现 Agent 间高性能消息传递。**不使用 page flipping**（page flipping 已废弃）。`IORING_OP_URING_CMD` 复用 io\_uring 的 SQE/CQE 通道，避免新增 syscall，Macro-Supervisor 的裁决命令亦通过此通道下发。

**标准名称**: IORING_OP_URING_CMD（IPC 零拷贝机制）

**旧称/禁止使用**: page flipping、页面翻转零拷贝

**参见**: AgentsIPC、io\_uring、A-IPC（统一进程间通信体系）、Macro-Supervisor

***

#### A-IPC / Unified Airymax IPC Fabric

**定义**: Unified Airymax IPC Fabric（统一进程间通信体系），Airymax Unify Design 5 模块之一。A-IPC 统一 agentrt-linux（AirymaxOS）的进程间通信体系，基于 io\_uring 的 `IORING_OP_URING_CMD` + registered buffer + mmap（**不使用 page flipping**）实现零拷贝消息传递。IPC magic（0x41524531 'ARE1'）+ 128 字节消息头结构定义于 `include/uapi/linux/airymax/ipc.h`（IRON-9 v3 \[SC] 共享契约层，A-IPC 模块）。

**技术领域**: 通信

**导出前缀**: `airy_ipc_*`（IPC 导出函数前缀）

**核心约束**:

- 零拷贝路径统一为 `IORING_OP_URING_CMD` + registered buffer + mmap
- **禁止**使用 page flipping 作为零拷贝路径
- IPC 队列冻结由 A-ULS（Micro-Supervisor）触发，fastpath 通过 `unlikely(ring->frozen)` 零开销检查
- [DSL] 降级模式下退化为最简 128B 消息头（3 字段）+ 2 操作

**标准名称**: A-IPC（Unified Airymax IPC Fabric，统一进程间通信体系）

**参见**: Airymax Unify Design、AgentsIPC、IORING_OP_URING_CMD、A-ULS（统一生命周期管理）

***

### 3.4 日志与错误码

#### A-UEF / Unified Error and Fault Framework

**定义**: Unified Error and Fault Framework（统一错误码与故障定义体系），Airymax Unify Design 5 模块之一。A-UEF 统一 agentrt-linux（AirymaxOS）的错误码与故障码空间，权威源为 `include/uapi/linux/airymax/error.h`（IRON-9 v3 \[SC] 共享契约层，A-UEF 模块）。A-UEF 是纯头文件模块，无 `.c` 代码。

**双码空间**:

| 码空间 | 范围 | 可恢复性 | 处置方式 |
| ------ | ---- | -------- | -------- |
| Error 码（负数） | `[-300, -1]` | 可恢复，函数返回值 | 调用方处理 |
| Fault 码（正数） | `[0x1000, 0x1FFF]` | 不可恢复，触发 Fault Handler | Micro-Supervisor 立即接管 |

**Error 码分段**: 通用 `[-1, -70]` / Capability `[-71, -100]` / [SC] 契约 `[-101, -200]` / [DSL] 降级 `[-201, -300]`

**Fault 码示例**: `AIRY_FAULT_CAP_FORGED=0x1001` / `AIRY_FAULT_CAP_LEAK=0x1002` / `AIRY_FAULT_RING_CORRUPT=0x1003` / `AIRY_FAULT_TIMEOUT=0x1004` / `AIRY_FAULT_ABNORMAL_CAP=0x1005` / `AIRY_FAULT_VM_FAULT=0x1006`

**技术领域**: 观测

**标准名称**: A-UEF（Unified Error and Fault Framework，统一错误码与故障定义体系）

**参见**: Airymax Unify Design、ErrorCodeSystem、A-ULP、Micro-Supervisor、[DSL] 降级生存层

***

#### A-ULP / Unified Logging and Printk Subsystem

**定义**: Unified Logging and Printk Subsystem（统一日志与打印系统），Airymax Unify Design 5 模块之一。A-ULP 统一 agentrt-linux（AirymaxOS）的日志体系，采用 `LOG_*` 枚举（5 级日志）+ 128 字节固定记录格式 + printk 8 级映射，权威源为 `include/uapi/linux/airymax/log_types.h`（IRON-9 v3 \[SC] 共享契约层，A-ULP 模块）。

**技术领域**: 观测

**导出前缀**: `airy_log_*`（日志导出函数前缀）

**核心约束**:

- 日志级别统一为 `LOG_*` 枚举（**禁止** `AIRY_LOG_*` 旧名）
- 128 字节固定记录格式单一权威源
- 共享内存方案采用 `alloc_pages(GFP_KERNEL)` + mmap（**不使用 DMA 一致性内存**，x86\_64 默认缓存一致）
- eventfd 通知复用 A-IPC 的 io\_uring 通道
- [DSL] 降级模式下退化为 printk 原生（仅 `LOG_FATAL` + `LOG_ERROR`）

**标准名称**: A-ULP（Unified Logging and Printk Subsystem，统一日志与打印系统）

**参见**: Airymax Unify Design、A-UEF、alloc_pages + mmap、TraceID、[DSL] 降级生存层

***

#### alloc_pages + mmap

**定义**: agentrt-linux（AirymaxOS）日志与 IPC 共享内存方案，采用 `alloc_pages(GFP_KERNEL)` 分配内核物理页 + `mmap` 映射到用户态地址空间。**不使用 DMA 一致性内存**（dma\_alloc\_coherent）——x86\_64 默认缓存一致，无需 DMA 一致性内存的强一致保证。该方案由 A-ULP（日志）与 A-IPC（IPC）联合约束。

**选型理由**:

- x86\_64 默认缓存一致，DMA 一致性内存非必要
- `alloc_pages` + `mmap` 是 Linux 内核通用内存分配原语，无架构特殊依赖
- 避免 DMA 一致性内存的池化开销与平台差异

**标准名称**: alloc_pages + mmap（日志/IPC 共享内存方案）

**旧称/禁止使用**: DMA 一致性内存、dma\_alloc\_coherent 用于日志/IPC

**参见**: A-ULP、A-IPC、ALK 6.6、MemoryRovol（记忆卷载）

***

#### ErrorCodeSystem / 统一错误码体系

**定义**: agentrt-linux（AirymaxOS）采用双错误码体系（现由 A-UEF 统一管理）：

| 体系                 | 适用场景           | 格式                            |
| ------------------ | -------------- | ----------------------------- |
| **C 负整数体系**（首要）    | C 内核和 daemon 层 | `AIRY_EOK=0`、`AIRY_EINVAL=-1` |
| **SDK 十六进制体系**（次要） | SDK 和外部接口      | `0x0000`-`0x7FFF` 分段          |

权威源唯一：`include/uapi/linux/airymax/error.h`（IRON-9 v3 \[SC] 共享契约层，A-UEF 模块）为唯一定义源。

**系统内代码**: `AIRY_E*`（C 内核首要体系）/ `AIRY_ERROR_*`（SDK 次要体系）

**禁止**: C 内核代码中使用十六进制错误码；SDK 中使用负整数错误码

**错误码分段规范**:

| 段   | 范围           | 含义         |
| --- | ------------ | ---------- |
| 成功  | 0            | `AIRY_EOK` |
| 通用  | -1 \~ -99    | 通用错误       |
| 系统  | -100 \~ -199 | 系统级错误      |
| 内核  | -200 \~ -299 | 内核错误       |
| 服务  | -300 \~ -399 | 服务错误       |
| LLM | -400 \~ -499 | LLM 推理错误   |
| 执行  | -500 \~ -599 | 执行错误       |
| 记忆  | -600 \~ -699 | 记忆错误       |
| 安全  | -700 \~ -799 | 安全错误       |
| 协议  | -800 \~ -899 | 协议错误       |

**参见**: A-UEF、JSON-RPC 2.0、IRON-9 v3

***

### 3.5 生命周期与监管

#### A-ULS / Unified Lifecycle Supervision Framework

**定义**: Unified Lifecycle Supervision Framework（统一生命周期管理），Airymax Unify Design 5 模块之一。A-ULS 统一 agentrt-linux（AirymaxOS）的 Agent 生命周期监管，采用**双 Supervisor 模型**：内核侧 Micro-Supervisor（冷酷执法）+ 用户侧 Macro-Supervisor（温情裁决）。调度策略由sched_tac 承载，Agent 8 态生命周期与 Linux 进程状态天然映射。

**技术领域**: 韧性

**核心约束**:

- 内核只保留最小机制（IPC 回调、Ring Buffer、Micro-Supervisor），策略下沉用户态
- Micro-Supervisor 检测异常后立即冻结 IPC 队列（`ring->frozen = true`）
- Macro-Supervisor 通过 `io_uring_cmd` 执行裁决，避免新增 syscall
- Macro-Supervisor 单点故障时，内核 watchdog 直接重启

**标准名称**: A-ULS（Unified Lifecycle Supervision Framework，统一生命周期管理）

**参见**: Airymax Unify Design、Micro-Supervisor、Macro-Supervisor、sched_tac、A-IPC

***

#### Micro-Supervisor / 内核冷酷执法

**定义**: agentrt-linux（AirymaxOS）的**内核态监管组件**，负责冷酷执法（不做人性化决策）。Micro-Supervisor 是纯 C LSM 模块（**不使用 BPF LSM**，对齐 openEuler），负责检测非法 Capability 并立即执法：

1. **检测**：纯 C LSM 钩子检测到伪造/越权 Capability
2. **冻结**：立即设置 `ring->frozen = true`，IPC fastpath 通过 `unlikely(ring->frozen)` 零开销检查跳过
3. **返回**：返回 `AIRY_FAULT_ABNORMAL_CAP`（0x1005）
4. **通知**：eventfd 通知 Macro-Supervisor

Micro-Supervisor 不做任何"人性化"决策，仅执行冷酷的机制级执法——借鉴 seL4 `handleFault()` 的设计：内核只负责检测与通知，裁决交给用户态。

**标准名称**: Micro-Supervisor（内核态监管组件，冷酷执法）

**参见**: Macro-Supervisor、A-ULS、纯 C LSM 模块、A-IPC、seL4、A-UEF（Fault 码）

***

#### Macro-Supervisor / 用户温情裁决

**定义**: agentrt-linux（AirymaxOS）的**用户态监管守护进程**（`macro_superv`，`services/daemons/macro_superv/`），接收 Micro-Supervisor 的故障通知后进行人性化裁决：

| 裁决选项 | 适用场景 | 执行方式 |
|---------|---------|---------|
| 继续 | 误报或可恢复 | 无操作 |
| 降级 | 资源不足 | 调整调度参数，进入 DEGRADED 态 |
| 暂停 | 多次违规 | `kill(SIGSTOP)`，进入 STOPPED 态 |
| 终止 | 严重违规 | `kill(SIGKILL)`，进入 DEAD 态 |

裁决通过 `io_uring_cmd` 执行，避免新增 syscall。Macro-Supervisor 单点故障时，内核 watchdog 直接重启。Macro-Supervisor 是 12 个 daemon 之一，systemd unit 为 `agentrt-macro-superv.service`，启动顺序 1。

**标准名称**: Macro-Supervisor（用户态监管守护进程，温情裁决）

**系统内代码**: `macro_superv` 二进制

**参见**: Micro-Supervisor、A-ULS、Daemon、IORING_OP_URING_CMD、Agent 8 态生命周期

***

### 3.6 跨端共享模型

#### IRON-9 v3 / 同源且部分代码共享（四层模型）

**定义**: agentrt-linux（AirymaxOS）与 agentrt（AirymaxAgentRT）之间的代码共享与语义同源原则。IRON-9 v3 是 IRON-9 的最新版本（开源极境工程与规范委员会决策变更），从 v2 的"三层模型 [SC]+[SS]+[IND]"升级为"**四层模型 [SC]+[SS]+[IND]+[DSL]**"——新增 [DSL] 降级生存层，确保 [SC] 头文件损坏/缺失时的最小可运行子集。

**四层共享模型**:

| 层次        | 标注     | 共享程度               | 内容                                                                                                                                                                                                                                                 |
| --------- | ------ | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **共享契约层** | \[SC]  | 完全共享代码             | `include/uapi/linux/airymax/` 10 个头文件：`error.h`（A-UEF）、`log_types.h`（A-ULP）、`ipc.h`（A-IPC）、`sched.h`（sched_tac）、`memory_types.h`（MemoryRovol L1-L4）、`security_types.h`（capability + LSM）、`cognition_types.h`（CoreLoopThree 阶段）、`syscalls.h`（24 槽位）、`uapi_compat.h`（三路类型桥接）、`lsm_types.h`（纯 C LSM）。另含错误码、规则编号体系、五维正交24原则 |
| **语义同源层** | \[SS]  | 高层 API 语义同源，签名独立演进 | 调度语义（MicroCoreRT）、安全模型（Cupolas）、IPC 传输（AgentsIPC）、记忆模型（MemoryRovol）、认知模型（CoreLoopThree）                                                                                                                                                            |
| **完全独立层** | \[IND] | 完全独立               | agentrt-linux 专属：内核驱动框架、Kbuild、systemd 集成、内核内部 API、纯 C LSM 实现；agentrt 专属：跨平台用户态运行时、SDK 四语言、CLI/TUI                                                                                                                                                            |
| **降级生存层** | \[DSL] | 自包含降级块（[SC] 头文件内）   | `#ifdef AIRY_SC_FALLBACK` 降级块；[SC] 损坏时的最小可运行子集（38 POSIX 码 + 最简 IPC + EEVDF 默认调度） | 

**v2 → v3 演进**:

| 维度    | IRON-9 v2        | IRON-9 v3                    |
| ----- | ---------------- | ---------------------------- |
| 契约层代码 | **完全共享**（10 个头文件） | **完全共享**（10 个头文件，v3 沿用）       |
| CI 校验 | 契约层变更双向校验        | 契约层变更双向校验 + magic 双向校验（v3 增强） |
| 降级路径  | 无                | **[DSL] 降级生存层**（v3 新增第四层）    |

**标准名称**: IRON-9 v3（同源且部分代码共享，四层模型 [SC]+[SS]+[IND]+[DSL]）

**旧称/禁止使用**: "IRON-9"（指旧版仅语义同源版本）、"IRON-9 v2"（指三层模型旧版，缺失 [DSL]）

**参见**: \[SC]/\[SS]/\[IND]/\[DSL] 四层标注、五维正交24原则、agentrt-linux（AirymaxOS）、Airymax Unify Design

***

#### \[SC] / Shared Contract（共享契约层）

**定义**: IRON-9 v3 四层模型的第一层，**Shared Contract（共享契约层）**。agentrt 与 agentrt-linux **物理共享同一份头文件**，逐字节相同，通过 `#include` 引用。物理宿主在 `agentrt-linux/kernel/include/uapi/linux/airymax/` 目录，共 10 个头文件。

**特征**:

- 物理宿主唯一：`kernel/include/uapi/linux/airymax/`
- agentrt 通过 `-I` 编译选项引用同一物理文件
- **禁止任何一端单方面修改**——变更必须双向同步，通过 `sc-dual-ci.yml` 逐字节校验
- 使用内核 UAPI 类型（`__u32`/`__u16`/`__u64`/`__u8`），不使用 `uint32_t` 等 C 标准库类型
- **禁止使用 `float` 类型**——内核态不支持浮点运算
- 每个头文件底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块（[DSL] 层）

**标准名称**: \[SC]（Shared Contract，共享契约层）

**参见**: IRON-9 v3、\[SS]、\[IND]、\[DSL]

***

#### \[SS] / Semantic Sibling（语义同源层）

**定义**: IRON-9 v3 四层模型的第二层，**Semantic Sibling（语义同源层）**。agentrt 与 agentrt-linux 高层 API 语义同源（概念操作一致），签名因抽象层级不同而独立演进。两端各自独立实现，通过 \[SC] 保证互操作。

**覆盖范围**: MicroCoreRT 调度语义、Cupolas 安全模型、AgentsIPC 传输、MemoryRovol 记忆模型、CoreLoopThree 认知模型。

**标准名称**: \[SS]（Semantic Sibling，语义同源层）

**参见**: IRON-9 v3、\[SC]、\[IND]、\[DSL]

***

#### \[IND] / Independent（完全独立层）

**定义**: IRON-9 v3 四层模型的第三层，**Independent（完全独立层）**。agentrt 与 agentrt-linux 各自独立演进，无依赖，无同步要求。

**覆盖范围**:

- agentrt-linux 专属：内核驱动框架、Kbuild/Kconfig、systemd 集成、内核内部 API（`printk`/`kmalloc`/`spin_lock`）、纯 C LSM 实现、`airy_defconfig`
- agentrt 专属：跨平台用户态运行时、SDK 四语言、CLI/TUI

**标准名称**: \[IND]（Independent，完全独立层）

**参见**: IRON-9 v3、\[SC]、\[SS]、\[DSL]

***

#### \[DSL] / Degraded Survival Layer（降级生存层）

**定义**: IRON-9 v3 四层模型的第四层，**Degraded Survival Layer（降级生存层）**，v3 新增。定义 [SC] 头文件损坏/缺失时的最小可运行子集。每个 [SC] 头文件底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块，提供自包含的最小符号集，确保 [SC] 损坏时系统仍能启动、打印日志、完成基本 IPC、安全 Panic。

**特征**:

- **自包含**：降级块内不依赖任何外部符号，仅依赖标准 POSIX/内核原语
- **最小集**：仅定义维持系统启动所需的最小符号集
- **显式告警**：降级块生效时发出 `#warning`，告知开发者当前为降级模式
- **最后防线**：[DSL] 是韧性领域的最后防线，正常运行时不生效

**[DSL] 降级时的最小可运行子集**:

| 维度 | 正常模式 | [DSL] 降级模式 |
|------|---------|--------------|
| 错误码 | 5 子空间（300 码） | 38 个 POSIX 码 + 1 个配置码（`AIRY_ECFGVERSION`） |
| 日志 | Ring Buffer + Logger Daemon | printk 原生（仅 `LOG_FATAL` + `LOG_ERROR`） |
| IPC | 完整 128B 消息头 + 3 操作 | 最简 128B 消息头（3 字段）+ 2 操作 |
| 调度 | sched_tac 三层（SCHED_DEADLINE/SCHED_FIFO/EEVDF） | EEVDF 默认调度 |
| 安全 | 纯 C LSM 完整校验 | 仅 POSIX capability |
| Fault | Fault Handler 优雅处理 | 统一 Panic（Fault 码禁用） |

**触发条件**: `sc-dual-ci.yml` 检测到两端 [SC] 头文件不一致 / `airy_defconfig` 中显式配置 `CONFIG_AIRY_SC_FALLBACK=y` / [SC] 头文件物理校验失败（hash 不匹配）。

**标准名称**: \[DSL]（Degraded Survival Layer，降级生存层）

**参见**: IRON-9 v3、\[SC]、sched_tac、A-UEF、A-ULP、airy_defconfig

***

#### Airymax Unify Design / 5 模块统一方案

**定义**: Airymax Unify Design 是 agentrt-linux（AirymaxOS）的**整体架构设计思想**，由 5 大模块构成统一方案：A-UEF（统一错误码与故障定义体系）/ A-ULP（统一日志与打印系统）/ A-UCS（统一配置管理体系）/ A-ULS（统一生命周期管理）/ A-IPC（统一进程间通信体系）。Unify Design 消解 7 个 P0 级架构冲突点（sched\_ext / page flipping / VFS 用户态化 / [SC] 头文件数量 / 错误码双轨制 / 日志双轨制 / DMA 一致性内存误用）。

**设计哲学**:

1. **机制在内核，策略在用户态**——直接借鉴 seL4 微内核思想，内核只保留最小机制（IPC 回调、Ring Buffer、Micro-Supervisor），全部策略下沉到用户态守护进程
2. **内核冷酷执法，用户温情裁决**——A-ULS 的双 Supervisor 模型，内核侧立即冻结异常 Agent，用户侧根据上下文进行人性化裁决
3. **每个技术点只允许有一个权威源**——SSoT v2 单一权威源模型，[SC] 共享契约头文件物理宿主唯一，CI 强制逐字节校验

**技术选型**: sched_tac（调度）+ IORING_OP_URING_CMD + registered buffer + mmap（IPC，不使用 page flipping）+ 纯 C LSM 模块（安全，不使用 BPF LSM）+ alloc_pages + mmap（日志/IPC 共享内存，不使用 DMA 一致性内存）。

**标准名称**: Airymax Unify Design（5 模块统一方案：A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）

**参见**: A-UEF、A-ULP、A-UCS、A-ULS、A-IPC、sched_tac、IRON-9 v3、ssot-registry.yaml

***

#### A-UCS / Unified Configuration Subsystem

**定义**: Unified Configuration Subsystem（统一配置管理体系），Airymax Unify Design 5 模块之一。A-UCS 统一 agentrt-linux（AirymaxOS）的配置管理，采用 sysctl/JSON 双向同步进行热重载。A-ULS 的调度参数、A-IPC 的 Ring 大小、A-ULP 的日志级别均通过 A-UCS 进行配置变更。

**技术领域**: 治理

**标准名称**: A-UCS（Unified Configuration Subsystem，统一配置管理体系）

**参见**: Airymax Unify Design、A-ULS、A-IPC、A-ULP、airy_defconfig

***

#### SSoT / 单一权威来源

**定义**: Single Source of Truth，单一权威来源。agentrt-linux（AirymaxOS）规范体系中，每一类技术决策（syscall 编号、IPC magic、任务描述符 magic、错误码、规则编号等）有且仅有一个权威定义源。所有其他文档引用该源时必须标注"SSoT 声明"，不得重新定义或产生冲突值。SSoT v2 引入机器可读注册表 `ssot-registry.yaml`。

**当前 SSoT 清单**:

| SSoT 对象                        | 权威源文件                                                        | 所在章节    |
| ------------------------------ | ------------------------------------------------------------ | ------- |
| 4 核心 syscall 编号（v1.1）               | `50-engineering-standards/120-cross-project-code-sharing.md` | §2.8    |
| IPC magic（0x41524531 'ARE1'）   | `50-engineering-standards/120-cross-project-code-sharing.md` | §3.1    |
| 任务描述符 magic（0x41475453 'AGTS'） | `50-engineering-standards/120-cross-project-code-sharing.md` | §3.2    |
| 错误码（`AIRY_E*`）                 | `include/uapi/linux/airymax/error.h`（\[SC] 共享契约层，A-UEF 模块）                | —       |
| 语义层代码规则                        | `50-engineering-standards/01-coding-standards.md`            | 全卷      |
| 规则编号注册表                        | `50-engineering-standards/09-ssot-registry.md`               | 规则编号注册表 |
| 规则编号机器可读 Schema                | `ssot-registry.yaml`（管理仓根）                                    | —       |
| IRON-9 v3 四层模型                 | `10-architecture/06-iron9-shared-model.md`                   | 全文      |
| Airymax Unify Design            | `10-architecture/10-unify-design.md`                         | 全文      |

**使用规则**:

1. 任何文档定义技术常量时必须标注其 SSoT 源
2. 当文档间存在值冲突时，以 SSoT 源为准
3. SSoT 源变更时必须同步更新所有引用文档
4. 新增 SSoT 对象需经开源极境工程与规范委员会审核后注册

**标准名称**: 单一权威来源 (SSoT)

**参见**: IRON-9 v3、\[SC] 共享契约层、ssot-registry.yaml

***

#### ssot-registry.yaml

**定义**: SSoT v2 机器可读注册表，位于 agentrt-linux 管理仓根（`agentrt-linux/ssot-registry.yaml`）。`ssot-registry.yaml` 是全部已登记规则编号（`OS-*-NNN` 格式）的机器可读 Schema，由 `agentrt-linux/tools/validate-ssot.py` 在 CI 中扫描比对。每 PR + 每 push 触发校验，未登记引用或已登记未引用均为 P0 级违规。

**校验逻辑**:

1. 加载 `ssot-registry.yaml`，构建已登记编号集合 `registered`（含 `status: active` 与 `status: deprecated`）
2. 递归扫描仓库内所有 `.md` 文件，正则提取 `OS-[A-Z]+(-[A-Z]+)?-\d+` 形式的规则 ID，构建 `referenced` 集合
3. 比对：未登记引用（`referenced - registered`）为 P0 级违规；已登记未引用（`registered - referenced`）为 P1 级告警

**标准名称**: ssot-registry.yaml（SSoT v2 机器可读注册表）

**参见**: SSoT、CI 校验、sc-dual-ci.yml

***

#### 五维正交24原则

**定义**: agentrt-linux（AirymaxOS）架构设计的顶层设计哲学，基于体系并行论（Multibody Cybernetic Intelligent System），具体实例化为五维正交系统。每个维度对应一类核心设计原则，共 24 条原则覆盖 agentrt-linux 设计的所有关键方面。

**五维正交24原则概览表**:

| 维度                | 核心问题                              | 原则数量           | 核心思想                                          |
| ----------------- | --------------------------------- | -------------- | --------------------------------------------- |
| 系统观 (System)      | agentrt-linux 作为复杂自适应系统，如何维持动态平衡？ | 4 项 (S-1\~S-4) | 反馈闭环、层次分解、总体设计部、涌现性管理                         |
| 内核观 (Kernel)      | 内核应该做什么，不应该做什么？                   | 4 项 (K-1\~K-4) | 内核极简、接口契约化、服务隔离、可插拔策略                         |
| 认知观 (Cognition)   | 智能体如何高效、可靠地进行认知决策？                | 4 项 (C-1\~C-4) | 认知层双思考功能、增量演化、记忆卷载、遗忘机制                       |
| 工程观 (Engineering) | 如何构建可维护、可测试、可演进的工程系统？             | 8 项 (E-1\~E-8) | 安全内生、可观测性、资源确定性、跨平台一致性、命名语义化、错误可追溯、文档即代码、可测试性 |
| 设计美学 (Aesthetics) | 如何让系统不仅正确，而且优雅？                   | 4 项 (A-1\~A-4) | 极简主义、细节关注、人文关怀、完美主义                           |

**标准名称**: 五维正交24原则

**参见**: 五维正交性、体系并行论、IRON-9 v3、Airymax Unify Design

**关联文档**: `docs/AirymaxOS/10-architecture/02-five-dimensional-principles.md`（完整落地映射）

***

#### 五维正交性

**定义**: 五维正交24原则中五个维度之间的数学关系特性。五维之间相互独立且正交：

- **正交性**：任一维度的原则不依赖于其他维度，可以独立演进
- **交织性**：在具体设计决策中，五个维度同时发挥作用
- **完备性**：五维覆盖了架构设计的所有关键方面，从思想到实践，从功能到美学
- **最小完备集**：五维是当前阶段的最佳实例，可演进而非封闭

**标准名称**: 五维正交性

**参见**: 五维正交24原则、体系并行论

***

#### 体系并行论

**定义**: agentrt-linux（AirymaxOS）和 agentrt（AirymaxAgentRT）的底层理论根基（Multibody Cybernetic Intelligent System）。体系并行论将智能体系统建模为多个并行运转的体系（基础体 + 认知体 + 执行体 + 记忆体 + 安全体），体系之间通过契约层交互，各自独立演进但共享顶层设计哲学。

**标准名称**: 体系并行论 (Multibody Cybernetic Intelligent System)

**参见**: 五维正交24原则、IRON-9 v3、Thinkdual（双思考系统）

***

### 3.7 工程标准与 CI

#### 8 子仓架构

**定义**: agentrt-linux（AirymaxOS）的代码组织架构，按能力域划分为 8 个独立子仓库：

| 子仓                    | 能力域   | 代码目录                     |
| --------------------- | ----- | ------------------------ |
| kernel      | 内核核心  | `kernel/`      |
| services    | 用户态服务 | `services/`    |
| cognition   | 认知运行时 | `cognition/`   |
| memory      | 记忆卷载  | `memory/`      |
| security    | 安全穹顶  | `security/`    |
| cloudnative | 云原生   | `cloudnative/` |
| system      | 系统编排  | `system/`      |
| tests-linux | 测试框架  | `tests-linux/`           |

**参见**: agentrt-linux（AirymaxOS）、IRON-9 v3

***

### 3.8 部署与运维

#### MemoryRovol / 记忆卷载

**定义**: agentrt-linux（AirymaxOS）的四层渐进式记忆架构，从原始数据到深层模式的完整处理管线：L1 原始卷（SHA-256 哈希 + ZSTD 压缩，append-only 持久化）、L2 特征层（HNSW 向量检索 + BM25 文本检索 + 混合融合）、L3 结构层（知识图谱 + VSA 绑定/解绑算子）、L4 模式层（HDBSCAN 聚类 + 0D 持久同调 + 模式挖掘）。

在 agentrt-linux 中，MemoryRovol 利用 ALK 6.6 内核基线的 CXL 内存池化和 MGLRU（多代 LRU）实现冷热数据分层，共享内存方案采用 `alloc_pages` + `mmap`（**不使用 DMA 一致性内存**），这是 agentrt-linux 作为 OS 发行版的独特优势。

**名称由来**: Memory + Rovol（Roll + Volume 的融合造词），寓意记忆的滚动卷载。

**标准名称**: 记忆卷载 (MemoryRovol)

**旧称/禁止使用**: "记忆漩涡引擎"

**系统内代码**: `airy_mr_*`

**代码目录**: `memory/`

**与 agentrt 的关系**: 遵循 IRON-9 v3 \[SS] 语义同源层——记忆模型一致（L1-L4 数据结构、GFP 掩码语义、PMEM 持久化接口共享于 \[SC] 层），实现独立（agentrt-linux 基于 CXL + MGLRU + alloc_pages/mmap 内核态，agentrt 基于 HeapStore 用户态）。

**参见**: CoreLoopThree（认知三阶段循环）、Forgetting Engine（遗忘机制）、MGLRU（多代 LRU）、CXL 内存池化、alloc_pages + mmap

***

#### Forgetting Engine / 遗忘机制

**定义**: 基于艾宾浩斯遗忘曲线驱动的记忆衰减与清理机制。遗忘公式: R(t) = e^(-t/τ)，τ 默认 7 天。支持 NONE/EBBINGHAUS/LINEAR/ACCESS\_BASED 四种遗忘策略。

在 agentrt-linux 中，遗忘机制与 MGLRU（多代 LRU）的多代页面回收策略在语义上一致——冷数据被遗忘（归档），热数据被保留（活跃代）。

**系统内代码**: `airy_forgetting_*`

**代码目录**: `memory/`

**参见**: MemoryRovol（记忆卷载）、MGLRU（多代 LRU）、MemorySwap（记忆交换算法）

***

#### MemorySwap / 记忆交换算法

**定义**: 记忆卷载的存储层级交换机制。借鉴操作系统虚拟内存分页交换，将记忆按访问热度划分为热/温/冷三个存储层级，在不同层级之间自动迁移。在 agentrt-linux 中，MemorySwap 直接利用 MGLRU 的多代回收机制 + CXL 内存池化实现硬件级加速。

**标准名称**: 记忆交换算法（正式）/ MemorySwap（简称）

**系统内代码**: `airy_memory_swap_manager_t`

**参见**: MemoryRovol（记忆卷载）、Forgetting Engine（遗忘机制）、MGLRU（多代 LRU）

***

#### TimeSliceInfer / 分时推理框架

**定义**: 记忆卷载的分时推理框架。将单次"全量加载->推理"分解为多个时间片上的"渐进式加载->增量推理->状态传递"流水线。每一时间片处理一个记忆子集，时间片间通过推理状态向量传递上下文连续性。

**标准名称**: 分时推理框架（正式）/ TimeSliceInfer（简称）

**旧称/禁止使用**: "时间切片推理"

**系统内代码**: `ts_inference_session_t`

**参见**: MemoryRovol（记忆卷载）、MemorySwap（记忆交换算法）

***

## 四、标准化名称映射总表

| 标准中文                     | 标准英文                                      | 代码前缀                       | IRON-9 v3 层次       | 禁止使用的旧称              |
| ------------------------ | ----------------------------------------- | -------------------------- | ------------------ | -------------------- |
| agentrt-linux（极境智能体操作系统） | agentrt-linux（AirymaxOS）                  | `airy_*`                   | \[SC]/\[SS]/\[IND]/\[DSL] | 仅出现 "AirymaxOS" 而无前缀 |
| 极境内核                     | ALK 6.6（Airymax Linux Kernel 6.6）          | —                          | \[IND]             | —                    |
| Linux 6.6 内核基线           | Linux 6.6 Kernel Baseline                 | —                          | \[IND]             | —                    |
| 内核配置 fragment            | airy_defconfig                            | —                          | \[IND]             | —                    |
| sched_tac               | SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS | —                          | \[IND]             | sched_ext + eBPF 调度方案 |
| 用户态调度器策略名                | AIRY\_SCHED\_AGENT                        | `airy_sched_agent_*`       | \[SS]              | SCHED\_AGENT（旧称，易与内核调度类编号混淆） |
| 微核心运行时                   | MicroCoreRT                               | `airy_core_*`              | \[SS]              | 微内核（不加限定词时）          |
| Agent 进程间通信              | AgentsIPC                                 | `airy_ipc_*`               | \[SS]              | —                    |
| IPC 零拷贝机制                | IORING_OP_URING_CMD                       | —                          | \[IND]             | page flipping        |
| 安全穹顶                     | Cupolas                                   | `cupolas_*`                | \[SS]              | Cupolas安全模块          |
| 纯 C LSM 模块               | 纯 C LSM（不使用 BPF LSM）                      | `security/airy/`        | \[IND]             | BPF LSM              |
| 日志/IPC 共享内存方案            | alloc_pages + mmap                        | —                          | \[IND]             | DMA 一致性内存            |
| 统一错误码与故障定义体系             | A-UEF                                       | `AIRY_E*` / `AIRY_FAULT_*` | \[SC]              | —                    |
| 统一日志与打印系统                | A-ULP                                      | `LOG_*`                    | \[SC]              | `AIRY_LOG_*`         |
| 统一配置管理体系                 | A-UCS                                       | —                          | \[SC]              | —                    |
| 统一生命周期管理                 | A-ULS                                       | —                          | \[SC]              | —                    |
| 统一进程间通信体系                | A-IPC                                      | —                          | \[SC]              | —                    |
| 内核态监管组件（冷酷执法）            | Micro-Supervisor                          | —                          | \[IND]             | —                    |
| 用户态监管守护进程（温情裁决）          | Macro-Supervisor                          | `macro_superv`             | \[IND]             | —                    |
| 5 模块统一方案                 | Airymax Unify Design                      | —                          | \[SC]              | —                    |
| 记忆卷载                     | MemoryRovol                               | `airy_mr_*`                | \[SS]              | 记忆漩涡引擎               |
| 认知三阶段循环                  | CoreLoopThree                             | `airy_loop_*`              | \[SS]              | 三层循环运行时、三层一体架构       |
| 双思考系统                    | Thinkdual                                 | `tc_*` / `mc_*` / `sc_*`   | \[SS]              | 认知双思系统、双系统认知模型       |
| 同源且部分代码共享（四层模型）          | IRON-9 v3                                 | —                          | —                  | IRON-9、IRON-9 v2     |
| 共享契约层                    | \[SC] Shared Contract                     | —                          | \[SC]              | Shared-Contract（旧称）  |
| 语义同源层                    | \[SS] Semantic Sibling                    | —                          | \[SS]              | Shared-Semantics（旧称） |
| 完全独立层                    | \[IND] Independent                        | —                          | \[IND]             | —                    |
| 降级生存层                    | \[DSL] Degraded Survival Layer            | —                          | \[DSL]             | —                    |
| 单一权威来源                   | SSoT (Single Source of Truth)             | —                          | —                  | —                    |
| SSoT v2 机器可读注册表          | ssot-registry.yaml                        | —                          | —                  | —                    |
| 五维正交24原则                 | Five-Dimensional Orthogonal 24 Principles | —                          | \[SC]              | —                    |
| 体系并行论                    | Multibody Cybernetic Intelligent System   | —                          | \[SC]              | —                    |
| 统一错误码体系                  | ErrorCodeSystem                           | `AIRY_E*` / `AIRY_ERROR_*` | \[SC]              | —                    |
| 遗忘机制                     | Forgetting Engine                         | `airy_forgetting_*`        | \[SS]              | —                    |
| 记忆交换算法                   | MemorySwap                                | `airy_memory_swap_*`       | \[SS]              | —                    |
| 任务流引擎                    | TaskFlow                                  | `airy_taskflow_*`          | \[SS]              | —                    |
| 多智能体协作框架                 | MAC                                       | `mac_*`                    | \[SS]              | Multi-Agent System   |
| 分时推理框架                   | TimeSliceInfer                            | `ts_*`                     | \[SS]              | 时间切片推理               |
| 安全宏体系                    | Memory Safety Macro System                | `AIRY_MALLOC` 等            | \[SC]              | —                    |
| 8 子仓架构                   | 8-Repository Architecture                 | —                          | \[IND]             | —                    |
| 用户态服务进程                  | Daemon                                    | `*_d` 后缀                   | \[IND]             | 守护进程、后端服务层           |
| 安全内生设计                   | Security by Design                        | —                          | \[SC]              | —                    |
| 服务隔离                     | Service Isolation                         | —                          | \[IND]             | —                    |
| 跟踪标识符                    | TraceID                                   | —                          | \[SC]              | —                    |

***

## 五、命名规范速查

### 函数命名

| 模块               | 格式                                   | 示例                              | IRON-9 v3 层次 |
| ------------ | ------------------------------------ | ------------------------------- | ------------ |
| 内核           | `airy_core_[action]`                 | `airy_init`                     | \[SS]        |
| 系统调用         | `airy_sys_[domain]_[action]`         | `airy_sys_task_submit`          | \[SC]        |
| 认知层          | `airy_[prefix]_[action]`             | `airy_tc_context_window_append` | \[SS]        |
| 安全穹顶         | `cupolas_[subsystem]_[action]`       | `cupolas_permission_check`      | \[SS]        |
| 用户态服务        | `[service]_[action]`                 | `cogn_d_complete`                | \[IND]       |
| IPC 通信       | `airy_ipc_[action]`                  | `airy_ipc_send`                 | \[SS]        |
| AIRY\_SCHED\_AGENT | `airy_sched_agent_[action]`          | `airy_sched_agent_submit`       | \[SS]        |
| 多智能体         | `mac_[action]` / `airy_mac_[action]` | `mac_framework_create`          | \[SS]        |

### 结构体命名

| 规范      | 格式                        | 示例                        |
| ------- | ------------------------- | ------------------------- |
| 公共结构体   | `airy_[name]_t`           | `airy_err_t`              |
| 子系统结构体  | `[prefix]_[name]_t`       | `tc_context_window_t`     |
| 配置结构体   | `airy_[name]_config_t`    | `airy_cognition_config_t` |
| IPC 消息头 | `struct airy_ipc_msg_hdr` | 128 字节统一消息头               |

### 枚举命名

| 规范    | 格式                    | 示例                       |
| ----- | --------------------- | ------------------------ |
| 状态枚举  | `AIRY_[NAME]_STATE_*` | `AIRY_TASK_STATE_READY`  |
| 类型枚举  | `AIRY_[NAME]_TYPE_*`  | `AIRY_PROTOCOL_TYPE_MCP` |
| 错误码枚举 | `AIRY_E*`             | `AIRY_EINVAL`            |
| 故障码枚举 | `AIRY_FAULT_*`        | `AIRY_FAULT_CAP_FORGED`  |

### 文件/目录命名

| 规范       | 规则                                           |
| -------- | -------------------------------------------- |
| 子仓目录     | `[domain]/`（如 `kernel/`、`tests-linux/`） |
| 用户态服务目录  | `[name]_d`（如 `cogn_d/`）                       |
| 源文件      | 下划线分隔 + `.c`（如 `thinking_chain.c`）           |
| 头文件      | 下划线分隔 + `.h`（如 `cupolas.h`）                  |
| 测试文件     | `test_[module].c`（如 `test_loop.c`）           |
| Rust 源文件 | 下划线分隔 + `.rs`（如 `capability.rs`）             |
| 纯 C LSM  | `security/airy/`（如 `airy_lsm.c`）           |
| 内核配置     | `arch/$(SRCARCH)/configs/airy_defconfig`     |

***

## 六、附录

### A. Daemon 服务完整列表（12 个）

> **权威源**：[100-operations/01-deployment.md §8.1](../100-operations/01-deployment.md)。二进制名与 agentrt 同源（`macro_superv`/`logger_daemon`/`config_daemon` 及 `*_d` 后缀守护进程），systemd 服务名采用 `agentrt-*.service` 格式（独立）。

| 二进制 | systemd unit | 职责 | 启动顺序 |
| ---------- | ------------- | ----------------------------- | --- |
| `macro_superv`   | `agentrt-macro-superv.service` | 主监管守护进程（Macro-Supervisor，用户温情裁决） | 1 |
| `gateway_d`      | `agentrt-gateway.service`      | 网关，对外入口（API 网关 + Agent/Skill 注册市场） | 1 |
| `sched_d`        | `agentrt-sched.service`        | 调度守护进程（AIRY\_SCHED\_AGENT 策略承载） | 1 |
| `logger_daemon`  | `agentrt-logger.service`       | 日志消费守护进程（A-ULP 用户态消费端） | 2 |
| `vfs_d`          | `agentrt-vfs.service`          | VFS 用户态服务守护进程 | 2 |
| `net_d`          | `agentrt-net.service`          | 网络策略守护进程（DPDK/AF\_XDP） | 2 |
| `cogn_d`         | `agentrt-cogn.service`         | 认知调度守护进程（LLM 推理代理） | 2 |
| `dev_d`          | `agentrt-dev.service`          | 工具调用代理（注册/执行/验证，含设备驱动） | 3 |
| `config_daemon`  | `agentrt-config.service`       | 配置管理守护进程（A-UCS 用户态消费端） | 3 |
| `mem_d`          | `agentrt-mem.service`          | 记忆管理守护进程（MemoryRovol L1-L4） | 4 |
| `audit_d`        | `agentrt-audit.service`        | 审计守护进程（监控/指标/追踪/告警） | 4 |
| `sec_d`          | `agentrt-sec.service`          | 安全策略守护进程（Cupolas 用户态策略） | 5 |

**旧称/禁止使用**: `llm_d`（→ `cogn_d`）、`tool_d`（→ `dev_d`）、`market_d`（→ `gateway_d`，已合并）、`drv_d`（→ `dev_d`，已合并）

### B. agentrt-linux 用户态驱动服务技术（3 项）

| 服务     | 功能      | 实现技术           |
| ------ | ------- | -------------- |
| vfs\_d | 用户态文件系统 | VFS 用户态化       |
| net\_d | 用户态网络栈  | DPDK / AF\_XDP |
| dev\_d | 用户态工具/驱动框架 | VFIO / libvfio + 工具调用 |

### C. 协议适配层完整列表（9 种）

| 类别   | 协议                  | 枚举值                |
| ---- | ------------------- | ------------------ |
| 标准协议 | MCP v1              | `PROTO_MCP`        |
| 标准协议 | A2A v0.3            | `PROTO_A2A`        |
| 标准协议 | AGNTCY ACP          | `PROTO_AGNTCY`     |
| 集成适配 | OpenAI              | `PROTO_OPENAI`     |
| 集成适配 | Claude              | `PROTO_CLAUDE`     |
| 集成适配 | OpenJiuwen          | `PROTO_OPENJIUWEN` |
| 集成适配 | OpenClaw            | `PROTO_OPENCLAW`   |
| 集成适配 | China Eco           | `PROTO_CHINA_ECO`  |
| 框架适配 | LangChain / AutoGen | —                  |

### D. Airymax Unify Design 5 模块速查

| 模块   | 全称                                       | 技术领域 | [SC] 头文件         | 核心职责                     |
| ---- | ---------------------------------------- | ---- | --------------- | ------------------------ |
| A-UEF  | Unified Error and Fault Framework        | 观测   | `error.h`       | 统一错误码（负数）+ 故障码（正数）+ [DSL] |
| A-ULP | Unified Logging and Printk Subsystem     | 观测   | `log_types.h`   | 统一日志枚举 + 128B 记录 + printk 映射 |
| A-UCS  | Unified Configuration Subsystem          | 治理   | —               | sysctl/JSON 双向同步热重载      |
| A-ULS  | Unified Lifecycle Supervision Framework                       | 韧性   | `sched.h`       | 双 Supervisor 模型 + Agent 8 态生命周期 |
| A-IPC | Unified Airymax IPC Fabric                    | 通信   | `ipc.h`         | IORING_OP_URING_CMD + registered buffer + mmap |

### E. IRON-9 v3 四层模型速查

| 层次        | 标注     | 全称                      | 共享程度               |
| --------- | ------ | ----------------------- | ------------------ |
| 共享契约层     | \[SC]  | Shared Contract         | 完全共享代码（10 个头文件）   |
| 语义同源层     | \[SS]  | Semantic Sibling        | 高层 API 语义同源，签名独立演进 |
| 完全独立层     | \[IND] | Independent              | 完全独立               |
| 降级生存层     | \[DSL] | Degraded Survival Layer  | 自包含降级块（[SC] 头文件内）  |

### F. 五维正交24原则编号参考

| 编号  | 原则名称     | 核心要点                           |
| --- | -------- | ------------------------------ |
| S-1 | 反馈闭环     | 系统的每一层必须设计完整的"感知-决策-执行-反馈"闭环   |
| S-2 | 层次分解     | 复杂系统必须按层次分解，每层只依赖其直接下层         |
| S-3 | 总体设计部    | 存在一个只做协调、不做执行的全局决策层            |
| S-4 | 涌现性管理    | 通过良好的模块交互设计，使系统整体产生超越部分之和的涌现行为 |
| K-1 | 内核极简     | 内核只保留不可再分的原子机制                 |
| K-2 | 接口契约化    | 所有跨模块交互必须通过明确定义的接口进行           |
| K-3 | 服务隔离     | 所有用户态服务作为独立守护进程运行              |
| K-4 | 可插拔策略    | 所有算法选择都设计为可运行时替换的策略接口          |
| C-1 | 认知层双思考功能 | 系统的每个认知环节都设计快慢两条路径             |
| C-2 | 增量演化     | 智能体不应一次性规划全部步骤，而应分阶段规划         |
| C-3 | 记忆卷载     | 记忆从原始经验中逐层提炼出可复用的知识模式          |
| C-4 | 遗忘机制     | 遗忘不是缺陷，而是记忆系统的核心功能             |
| E-1 | 安全内生     | 安全不是附加层，而是内嵌于系统设计的每一个环节        |
| E-2 | 可观测性     | 系统必须提供完整的可观测性                  |
| E-3 | 资源确定性    | 每块资源的生命周期必须是确定性的               |
| E-4 | 跨平台一致性   | 核心业务逻辑与平台无关                    |
| E-5 | 命名语义化    | 名称即文档                          |
| E-6 | 错误可追溯    | 每个错误必须可追溯到其根源                  |
| E-7 | 文档即代码    | 文档与代码同步更新                      |
| E-8 | 可测试性     | 测试不是事后补充，而是设计的首要考量             |
| A-1 | 简约至上     | 用最少的接口提供最大的价值                  |
| A-2 | 极致细节     | 细节决定成败                         |
| A-3 | 人文关怀     | 技术服务于人                         |
| A-4 | 完美主义     | 不接受"足够好"                       |

***

## 参考文献

\[1] 体系并行论 — `docs/AirymaxRT/00-requirements/01-mcis-cn.md`\
\[2] 认知层理论 — `docs/AirymaxRT/00-requirements/02-cognition-design-cn.md`\
\[3] 记忆层理论 — `docs/AirymaxRT/00-requirements/03-memory-design-cn.md`\
\[4] 设计原则 — `docs/AirymaxRT/00-requirements/04-design-principles-cn.md`\
\[5] 五维正交24原则与 agentrt-linux（AirymaxOS）落地映射 — `docs/AirymaxOS/10-architecture/02-five-dimensional-principles.md`\
\[6] agentrt-linux 架构设计 — `docs/AirymaxOS/10-architecture/01-system-architecture.md`\
\[7] 微内核设计思想详解 — `docs/AirymaxOS/10-architecture/03-microkernel-strategy.md`\
\[8] agentrt-linux（AirymaxOS）工程基线 — `docs/AirymaxOS/10-architecture/04-engineering-baseline.md`\
\[9] IRON-9 v3 共享模型 — `docs/AirymaxOS/10-architecture/06-iron9-shared-model.md`\
\[10] Airymax Unify Design 总纲 — `docs/AirymaxOS/10-architecture/10-unify-design.md`\
\[11] [DSL] 降级生存层 — `docs/AirymaxOS/10-architecture/11-degraded-survival-layer.md`\
\[12] agentrt-linux 工程标准规范 — `docs/AirymaxOS/50-engineering-standards/README.md`\
\[13] IRON-9 v3 四层共享模型落地规范 — `docs/AirymaxOS/50-engineering-standards/120-cross-project-code-sharing.md`\
\[14] SSoT 规则编号权威注册表 — `docs/AirymaxOS/50-engineering-standards/09-ssot-registry.md`\
\[15] SSoT v2 机器可读 Schema — `agentrt-linux/ssot-registry.yaml`（管理仓根）\
\[16] 12 daemons 部署映射 — `docs/AirymaxOS/100-operations/01-deployment.md`\
\[17] 调度性能与sched_tac — `docs/AirymaxOS/170-performance/01-scheduling-performance.md`\

***

**最后更新**: 2026-07-17\
**文档版本**: v1.0\
**维护者**: 开源极境工程与规范委员会（OpenAirymax Engineering and Standardization Committee）

***

(C) 2025-2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
