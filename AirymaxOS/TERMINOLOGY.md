Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# agentrt-linux 统一术语表

**最新**: 2026-07-07
**状态**: 草案（文档体系完成）
**路径**: OpenAirymax/docs/AirymaxOS/TERMINOLOGY.md

---

## 一、编制说明

本术语表是 agentrt-linux（AirymaxOS）规范体系的**唯一权威术语参考**。

**目的**:
- 统一所有 agentrt-linux（AirymaxOS）规范文档、代码注释、API 文档中的术语定义
- 当不同文档间存在术语冲突时，**以本表为准**
- 确保与 agentrt（AirymaxAgentRT）的术语在 IRON-9 v2 [SC] 共享契约层保持一致

**使用规则**:
1. 编写文档时必须使用本表的**标准名称**
2. 核心表述统一使用 "agentrt-linux（AirymaxOS）" 形式，不得单独出现 "AirymaxOS" 而无前缀
3. 标注为"禁止使用"的旧称不得出现在任何新的规范文档或代码注释中
4. 新增术语需经架构团队审核后加入
5. 标准计算机专业名词（如 IPC、Daemon、Sandbox、Microkernel 等）遵循业界通用定义，本表不重新定义，仅在"标准计算机名词"分区中说明其在 agentrt-linux（AirymaxOS）中的使用上下文
6. 当标准名词与 agentrt-linux 特有概念同名时，以 agentrt-linux 特有定义为准
7. 与 agentrt 术语表中共享的术语，通过 IRON-9 v2 [SC] 层标注保持一致性

---

## 二、标准计算机名词（OS 领域通用术语）

本分区所列术语均为计算机科学领域，特别是**操作系统领域**的业界通用专业名词，遵循其标准定义，agentrt-linux（AirymaxOS）**不重新定义**。仅说明其在 agentrt-linux 项目中的使用上下文，供文档编写与代码注释参考。

### IPC / 进程间通信

**业界定义**: Inter-Process Communication，操作系统提供的机制，允许不同进程之间交换数据与信号。常见实现包括管道、消息队列、共享内存、信号量、套接字、io_uring 等。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）微内核化改造的核心机制之一，基于 io_uring 零拷贝消息传递，承载用户态服务（Daemon）之间、Agent 与内核之间的高性能消息传递。128 字节统一消息头（magic=0x41524531）与 agentrt 的 AgentsIPC 同源（IRON-9 v2 [SS] 语义同源层）。

**系统内代码**: `agentrt_ipc_*`

**参见**: MicroCoreRT（微核心运行时）、AgentsIPC（Agent 进程间通信）、io_uring 异步 I/O

---

### Daemon / 用户态服务进程

**业界定义**: 在后台运行的长生命周期进程，通常独立于终端会话，提供特定服务。Unix 传统命名以 `d` 后缀（如 `syslogd`、`sshd`）。在 systemd 体系下，daemon 作为 systemd 单元管理。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）采用微内核化改造策略，将功能服务化为用户态进程，通过 systemd 集成管理。命名遵循 `*_d` 约定（如 `gateway_d`、`llm_d`、`tool_d`）。12 个 daemon 服务覆盖认知、安全、记忆、调度等全部功能域。

**系统内代码**: `*_d` 进程后缀，systemd unit 文件

**参见**: IPC、Service Isolation（服务隔离）、systemd 集成

---

### Sandbox / 沙箱

**业界定义**: 一种安全机制，将程序运行限制在受控的隔离环境中，防止其对宿主系统造成意外或恶意影响。常见实现包括 seccomp、Landlock、namespace、capability 等。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）系统调用层（syscall）的七大功能域之一，为 Agent 与 Skill 提供隔离执行环境。沙箱机制结合 Linux 6.6 内核基线的 seccomp + Landlock + capability 三道防线，配合 Cupolas 共同构成安全防护体系。

**系统内代码**: `agentrt_sys_sandbox_*`

**参见**: Cupolas（安全穹顶）、Service Isolation（服务隔离）、Security by Design（安全内生设计）

---

### Microkernel / 微内核

**业界定义**: 操作系统的最小核心，只提供最基本的服务（调度、IPC、地址空间管理、基本内存管理）。文件系统、网络栈、设备驱动等服务运行在用户态。代表实现：seL4、Zircon、Minix3、QNX、L4。

> **ADR-014 约束**: agentrt-linux 微内核设计思想**唯一来源为 seL4**，不引入 Zircon / Minix3 / 其他微内核架构。业界定义中列出其他实现仅作术语参考，不代表 agentrt-linux 采纳其设计思想。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）基于 Linux 6.6 内核进行**微内核化改造**（非从零开发），演进目标为真正的微内核——利用 sched_ext + eBPF + io_uring 实现微内核化，将 VFS、网络栈、部分驱动移到用户态，同时保留 Linux 内核的稳定性和硬件兼容性。遵循 Liedtke minimality principle。

**参见**: Liedtke Minimality Principle、seL4、MicroCoreRT（微核心运行时）

---

### Liedtke Minimality Principle

**业界定义**: Jochen Liedtke（L4 微内核创始人）提出的微内核设计原则："A concept is tolerated inside the microkernel only if moving it outside the kernel, i.e., permitting competing implementations, would prevent the implementation of the system's required functionality."

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）的内核极简原则（五维正交24原则 K-1）直接遵循 Liedtke minimality——内核只保留不可再分的原子机制（调度、IPC、内存管理、时间服务），一切可以在用户态实现的功能都必须在用户态实现。

**参见**: K-1 内核极简原则、MicroCoreRT（微核心运行时）、五维正交24原则

---

### capability（操作系统安全）

**业界定义**: 一种访问控制机制，使用不可伪造的令牌（capability）代表对资源的访问权限。进程必须持有指向目标资源的 capability 才能进行访问。capability 可以被委托、复制、限制。代表实现：seL4、Zircon。

> **ADR-014 约束**: agentrt-linux capability 模型**唯一来源为 seL4**，不引入 Zircon handle 模型。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）安全子系统（airymaxos-security）实现 capability 系统（seL4 风格），与 Cupolas 同源。capability 令牌格式定义于 `include/airymax/security_types.h`（IRON-9 v2 [SC] 共享契约层），结合 LSM 钩子（SELinux）实现纵深防御。

**系统内代码**: `agentrt_cap_*`

**参见**: Cupolas（安全穹顶）、LSM（Linux Security Module）、Security by Design（安全内生设计）

---

### io_uring / 异步 I/O

**业界定义**: Linux 内核的高性能异步 I/O 接口，由 Jens Axboe 设计。通过共享内存环形缓冲区（submission queue + completion queue）实现零拷贝、低延迟的异步 I/O 操作，支持 read/write/send/recv/accept/connect 等操作。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）IPC 子系统的核心传输机制，基于 io_uring 实现零拷贝消息传递。内核固定 OP 码用于 Agent IPC 消息收发，与 agentrt 的 AgentsIPC 语义同源（IRON-9 v2 [SS] 语义同源层）。

**参见**: IPC、AgentsIPC（Agent 进程间通信）、SCHED_AGENT

---

### sched_ext / 可扩展调度器

**业界定义**: Linux 内核的可扩展调度器框架（主线 6.12+），允许通过 eBPF 程序在用户态定义调度策略，实现自定义调度类。提供 struct_ops 机制将内核调度器行为暴露给 eBPF 程序。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）通过 agentrt-linux 内核增强策略将 sched_ext 向前移植到 Linux 6.6 内核基线，实现 SCHED_AGENT 策略——专门为 Agent 工作负载优化的 eBPF 用户态调度器。调度语义与 agentrt 的 MicroCoreRT 一致（同源）。

**系统内代码**: `agentrt_sched_agent_*`，eBPF 程序

**参见**: SCHED_AGENT（Agent 调度类）、eBPF、MicroCoreRT（微核心运行时）

---

### eBPF

**业界定义**: Extended Berkeley Packet Filter，Linux 内核的可编程虚拟机。允许在沙箱环境中运行用户提供的程序，安全地扩展内核功能。支持 kfunc（内核函数）、dynamic pointer、struct_ops 等高级特性。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）利用 eBPF 实现三大核心能力：SCHED_AGENT 用户态调度器（sched_ext）、内核可观测性探针（kfunc + dynamic pointer）、安全策略可插拔（BPF + io_uring 集成）。所有 eBPF 程序通过 Cupolas 安全穹顶进行权限校验。

**参见**: sched_ext、SCHED_AGENT、Cupolas（安全穹顶）

---

### MGLRU / 多代 LRU

**业界定义**: Multi-Generation LRU，Linux 内核 6.6 引入的页面回收算法。将内存页面按访问热度分为多代（generation），优先回收最冷代页面，实现冷热数据分层和高效内存回收。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）内核内存管理的核心算法，配合 CXL 内存池化实现 MemoryRovol L1-L4 四层记忆的冷热数据分层。MGLRU 的多代回收策略与记忆卷载的遗忘机制（Forgetting Engine）在语义上一致。

**参见**: MemoryRovol（记忆卷载）、Forgetting Engine（遗忘机制）、CXL 内存池化

---

### Linux Security Module (LSM)

**业界定义**: Linux 内核的安全框架，提供钩子（hook）机制允许安全模块（如 SELinux、AppArmor）在关键操作点进行访问控制。LSM 钩子覆盖文件系统、网络、进程管理、capability 等子系统。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）安全子系统（airymaxos-security）利用 LSM 框架实现 Agent 专属安全策略。LSM 钩子 ID 定义于 `include/airymax/security_types.h`（IRON-9 v2 [SC] 共享契约层），与 capability 系统共同构成纵深防御。

**参见**: capability、Cupolas（安全穹顶）、SELinux

---

### systemd

**业界定义**: Linux 系统的初始化系统和服务管理器，负责系统引导、服务管理、日志记录、设备管理等。提供 unit 文件定义服务依赖、启动顺序和资源限制。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）采用 systemd 管理 12 个用户态守护进程。每个 daemon 通过 systemd unit 文件定义隔离策略（独立地址空间、capability 限制、资源限制）。遵循 K-3 服务隔离原则（五维正交24原则）。

**参见**: Daemon（用户态服务进程）、Service Isolation（服务隔离）

---

### Security by Design / 安全内生设计

**业界定义**: 一种工程理念，将安全作为系统设计的首要原则，在架构层面而非附加层面解决安全问题。与"安全左移"（Shift Left Security）理念一致，强调默认安全（Secure by Default）。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）的核心设计理念之一（五维正交24原则 E-1），贯穿内核（capability + LSM）、服务层（沙箱 + 隔离）、记忆层（审计哈希链）等所有安全相关组件。

**参见**: Cupolas（安全穹顶）、capability、LSM、Service Isolation（服务隔离）

---

### TraceID / 跟踪标识符

**业界定义**: 分布式追踪系统中用于唯一标识一次请求链路的标识符，贯穿请求经过的所有服务节点，支持链路分析与性能诊断。常见于 OpenTelemetry、Jaeger、Zipkin 等可观测性体系。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）可观测性子系统的核心标识，贯穿 Agent 任务执行、IPC 消息传递、Daemon 服务调用的全链路。128 字节 IPC 消息头中固定携带 TraceID 字段，与 agentrt 的 TraceID 体系完全一致（IRON-9 v2 [SC] 共享契约层）。

**参见**: IPC、OpenTelemetry、可观测性

---

### Service Isolation / 服务隔离

**业界定义**: 一种架构模式，将不同服务的运行环境、资源、权限相互隔离，防止单一服务故障或被攻破后影响整体系统。常见实现包括进程隔离、容器隔离、虚拟机隔离等。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）微内核化改造的设计原则之一（五维正交24原则 K-3），各 Daemon 服务运行于独立地址空间，通过 IPC 通信，资源与权限相互隔离。systemd unit 文件定义隔离边界，与 Sandbox、Cupolas 协同构成纵深防御。

**参见**: Daemon（用户态服务进程）、Sandbox（沙箱）、Cupolas（安全穹顶）、systemd

---

### JSON-RPC 2.0

**业界定义**: JSON Remote Procedure Call，一种无状态、轻量级的远程过程调用协议，使用 JSON 作为数据格式。规范定义于 https://www.jsonrpc.org/specification。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）协议适配层支持的标准协议之一，用于 Agent 与外部客户端之间的请求-响应通信。使用强制命名空间格式 `<daemon>.<method>`，错误码体系与 agentrt 统一错误码体系对接。

**参见**: IPC、AgentsIPC（Agent 进程间通信）、ErrorCodeSystem（统一错误码体系）

---

### CXL / Compute Express Link

**业界定义**: 一种开放标准的互连协议，基于 PCIe 物理层，支持 CPU 与加速器、内存扩展器之间的高速通信。CXL 2.0+ 支持内存池化（memory pooling），允许多个主机共享内存资源。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）利用 Linux 6.6 内核基线的 CXL 支持实现 MemoryRovol L1-L4 跨节点内存池化，配合 MGLRU（多代 LRU）实现冷热数据分层。这是 agentrt-linux 作为 OS 发行版相对于 agentrt 用户态运行时的独特优势。

**系统内代码**: `agentrt_cxl_*`

**参见**: MGLRU（多代 LRU）、MemoryRovol（记忆卷载）、PMEM（持久内存）

---

### DPDK / AF_XDP

**业界定义**: Data Plane Development Kit，用户态高性能数据包处理框架。AF_XDP（Address Family eXpress Data Path）是 Linux 内核的原生高性能数据路径，允许用户态程序直接访问网卡接收环。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）网络栈用户态化的核心技术，通过 DPDK/AF_XDP 将网络处理从内核态移到用户态（net_d 守护进程），实现微内核化改造。遵循 K-1 内核极简原则（五维正交24原则）。

**系统内代码**: `agentrt_net_d`（net_d 守护进程）

**参见**: Microkernel（微内核）、K-1 内核极简原则、Daemon（用户态服务进程）

---

### VFIO / libvfio

**业界定义**: Virtual Function I/O，Linux 内核的用户态驱动框架。VFIO 允许用户态程序直接访问设备 MMIO 区域和 DMA，实现高性能用户态驱动。libvfio 是 VFIO 的用户态库。

**agentrt-linux 使用上下文**: agentrt-linux（AirymaxOS）驱动框架用户态化的核心技术，通过 VFIO/libvfio 将部分设备驱动从内核态移到用户态（drv_d 守护进程），实现微内核化改造。遵循 K-1 内核极简原则（五维正交24原则）。

**系统内代码**: `agentrt_drv_d`（drv_d 守护进程）

**参见**: Microkernel（微内核）、K-1 内核极简原则、Daemon（用户态服务进程）

---

## 三、agentrt-linux 特有架构术语

### agentrt-linux（AirymaxOS / 极境智能体操作系统）

**定义**: agentrt-linux 是基于 Linux 6.6 内核基线的智能体操作系统发行版，采用微内核化改造策略（而非从零开发微内核）。代码仓库名 `agentrt-linux`，项目定位为 agentrt（AirymaxAgentRT）的 OS 层最佳载体。与 agentrt 遵循 IRON-9 v2"同源且部分代码共享"原则：[SC] 共享契约层完全共享代码，[SS] 语义同源层 API 签名相同但实现独立，[IND] 完全独立层各自独立。

**标准名称**: agentrt-linux（AirymaxOS）——正式文档中必须使用此完整形式，不得单独出现 "AirymaxOS" 而无 "agentrt-linux" 前缀。

**别名**: 极境智能体操作系统（中文完整名称）

**旧称/禁止使用**: 仅出现 "AirymaxOS" 而无 "agentrt-linux" 前缀的表述

**系统内代码**: `agentrt_*` 前缀（IRON-9 v2 [SC] 共享契约层）

**代码目录**: `airymaxos-kernel/` / `airymaxos-services/` / `airymaxos-cognition/` / `airymaxos-memory/` / `airymaxos-security/` / `airymaxos-cloudnative/` / `airymaxos-system/` / `tests-linux/`（8 子仓）

**参见**: agentrt（AirymaxAgentRT）、IRON-9 v2、MicroCoreRT（微核心运行时）、五维正交24原则

---

### MicroCoreRT / 微核心运行时

**定义**: agentrt-linux（AirymaxOS）内核的极简运行时抽象层，提供最基本的 IPC、内存管理、任务调度、时间管理、可观测性功能。与 agentrt 的 MicroCoreRT 共享设计理念（同源），但在 agentrt-linux 中作为 OS 级实现（内核态 + 用户态服务），而在 agentrt 中作为用户态运行时实现。

**标准名称**: 微核心运行时 (MicroCoreRT)

**与 agentrt 的关系**: 遵循 IRON-9 v2 [SS] 语义同源层——API 签名相同（调度语义、IPC 语义、内存管理语义），实现独立（agentrt-linux 基于 sched_ext + io_uring + MGLRU，agentrt 基于用户态消息队列 + 用户态调度器）。

**系统内代码**: `agentrt_core_*`

**代码目录**: `airymaxos-kernel/`

**参见**: K-1 内核极简原则、SCHED_AGENT（Agent 调度类）、Liedtke Minimality Principle

---

### AgentsIPC / Agent 进程间通信

**定义**: agentrt-linux（AirymaxOS）的 Agent 间高性能通信机制，基于 io_uring 零拷贝消息传递 + 128 字节统一消息头（magic=0x41524531）。与 agentrt 的 AgentsIPC 语义同源（IRON-9 v2 [SS] 语义同源层），但在 agentrt-linux 中基于内核 io_uring 固定 OP 码实现，在 agentrt 中基于用户态消息队列实现。

**标准名称**: Agent 进程间通信 (AgentsIPC)

**系统内代码**: `agentrt_ipc_*`

**参见**: IPC、io_uring、128 字节消息头、SCHED_AGENT

---

### Cupolas / 安全穹顶

**定义**: agentrt-linux（AirymaxOS）的内生安全防护层，实现输入净化 -> 权限仲裁 -> 网络过滤 -> 审计记录四重防护链。在 agentrt-linux 中，Cupolas 结合 capability 系统（seL4 风格）、LSM 钩子（SELinux 集成）、国密算法（SM2/SM3/SM4）和机密计算（TEE/SGX），构成 OS 级纵深防御体系。

**与 agentrt 的关系**: 遵循 IRON-9 v2 [SS] 语义同源层——安全模型一致（capability 令牌格式、策略裁决 4 值枚举共享于 [SC] 层），实现独立（agentrt-linux 基于 LSM + capability 内核态，agentrt 基于用户态安全模块）。

**标准名称**: 安全穹顶 (Cupolas)

**系统内代码**: `cupolas_*`

**代码目录**: `airymaxos-security/`

**参见**: capability、LSM、Security by Design（安全内生设计）、Sandbox（沙箱）

---

### MemoryRovol / 记忆卷载

**定义**: agentrt-linux（AirymaxOS）的四层渐进式记忆架构，从原始数据到深层模式的完整处理管线：L1 原始卷（SHA-256 哈希 + ZSTD 压缩，append-only 持久化）、L2 特征层（HNSW 向量检索 + BM25 文本检索 + 混合融合）、L3 结构层（知识图谱 + VSA 绑定/解绑算子）、L4 模式层（HDBSCAN 聚类 + 0D 持久同调 + 模式挖掘）。

在 agentrt-linux 中，MemoryRovol 利用 Linux 6.6 内核基线的 CXL 内存池化和 MGLRU（多代 LRU）实现冷热数据分层，这是 agentrt-linux 作为 OS 发行版的独特优势。

**名称由来**: Memory + Rovol（Roll + Volume 的融合造词），寓意记忆的滚动卷载。

**标准名称**: 记忆卷载 (MemoryRovol)

**旧称/禁止使用**: "记忆漩涡引擎"

**系统内代码**: `agentrt_memoryrov_*`

**代码目录**: `airymaxos-memory/`

**与 agentrt 的关系**: 遵循 IRON-9 v2 [SS] 语义同源层——记忆模型一致（L1-L4 数据结构、GFP 掩码语义、PMEM 持久化接口共享于 [SC] 层），实现独立（agentrt-linux 基于 CXL + MGLRU 内核态，agentrt 基于 HeapStore 用户态）。

**参见**: CoreLoopThree（认知三阶段循环）、Forgetting Engine（遗忘机制）、MGLRU（多代 LRU）、CXL 内存池化

---

### CoreLoopThree / 认知三阶段循环

**定义**: agentrt-linux（AirymaxOS）的三层认知-执行-记忆主循环运行时：

| 层级 | 名称 | 职责 |
|------|------|------|
| L1 认知层 | Cognition Engine | 意图解析 -> 规划 -> 分发 -> 协调 -> 批判 |
| L2 执行层 | Execution Engine | 任务执行 -> 单元调度 -> 补偿事务 |
| L3 记忆层 | Memory Engine | 结果写入 -> 查询 -> 挂载 |

在 agentrt-linux 中，认知层实现为内核 kthread（而非用户态线程），利用 SCHED_AGENT 策略获得 OS 级调度优先级和确定性延迟。

**标准名称**: 认知三阶段循环 (CoreLoopThree)

**旧称/禁止使用**: "三层循环运行时"、"三层一体架构"、"三层循环核心"

**系统内代码**: `agentrt_loop_*`

**代码目录**: `airymaxos-cognition/`

**与 agentrt 的关系**: 遵循 IRON-9 v2 [SS] 语义同源层——认知模型一致（CoreLoopThree 阶段枚举、Thinkdual 模式枚举、LLM 推理阶段枚举共享于 [SC] 层），实现独立（agentrt-linux 基于内核 kthread + SCHED_AGENT，agentrt 基于用户态线程）。

**参见**: Thinkdual（双思考系统）、MemoryRovol（记忆卷载）、SCHED_AGENT（Agent 调度类）

---

### Thinkdual / 双思考系统

**定义**: agentrt-linux（AirymaxOS）的核心认知创新，由三组件构成的认知架构：

| 组件 | 角色 | 功能 |
|------|------|------|
| **t2 主思考** | 慢思考 | 深度推理、反思调整、长期规划 |
| **t1-f 快思考** | 快思考-事实 | 快速响应、流式验证、事实核查 |
| **t1-p 专业思考** | 快思考-专业 | 专业领域仲裁、深度质量评估 |

三组件通过 triple_coordinator 协同工作。"双思"指深度思考（t2）与快速思考（t1）两大思维模式的协作。

**标准中文名**: 双思考系统（正式）/ Thinkdual（简称）
**标准英文名**: Thinkdual Cognitive Dual-Thinking System（完整）/ Thinkdual（简化）

**旧称/禁止使用**: "认知双思系统"、"双系统认知模型"、"Thinkdual 认知双思系统"、"Triple Coordinator"（triple_coordinator 是协调器，非系统名称）

**系统内代码**: `tc_*`（思考链）/ `mc_*`（元认知）/ `sc_*`（流式验证）/ `tc3_*`（协调器）

**代码目录**: `airymaxos-cognition/`

**参见**: CoreLoopThree（认知三阶段循环）、五维正交24原则 C-1（认知层双思考功能）

---

### SCHED_AGENT / Agent 调度类

**定义**: agentrt-linux（AirymaxOS）专为 Agent 工作负载优化的 eBPF 用户态调度类，基于 sched_ext 框架实现。SCHED_AGENT 提供 Agent 感知的调度策略（基于认知阶段、Token 预算、任务优先级），调度语义与 agentrt 的 MicroCoreRT 一致（同源）。

**调度策略维度**:
- 认知阶段感知：Cognition Engine 阶段 Agent 获得更高优先级
- Token 预算感知：Token 预算即将耗尽的 Agent 获得调度加速
- 任务优先级：关键任务（安全审计、事务提交）获得实时调度保证
- 跨 Agent 协同：多 Agent 协作任务在相邻时间片内调度

**标准名称**: Agent 调度类 (SCHED_AGENT)

**系统内代码**: `agentrt_sched_agent_*`，eBPF 程序

**参见**: sched_ext、eBPF、MicroCoreRT（微核心运行时）、CoreLoopThree（认知三阶段循环）

---

### IRON-9 v2 / 同源且部分代码共享

**定义**: agentrt-linux（AirymaxOS）与 agentrt（AirymaxAgentRT）之间的代码共享与语义同源原则。IRON-9 v2 是 IRON-9 的升级版本（2026-07-07 用户决策变更），从"仅语义同源、实现独立"升级为"同源且部分代码共享"。

**三层共享模型**:

| 层次 | 标注 | 共享程度 | 内容 |
|------|------|---------|------|
| **共享契约层** | [SC] | 完全共享代码 | `include/airymax/` 6 个头文件：`bpf_struct_ops.h`、`memory_types.h`、`security_types.h`、`cognition_types.h`、`sched.h`、`ipc.h`。含 IPC 消息头结构、syscall 编号、capability 令牌格式、错误码、规则编号体系、五维正交24原则 |
| **语义同源层** | [SS] | API 签名相同，实现独立 | 调度语义（MicroCoreRT）、安全模型（Cupolas）、IPC 传输（AgentsIPC）、记忆模型（MemoryRovol）、认知模型（CoreLoopThree） |
| **完全独立层** | [IND] | 完全独立 | agentrt-linux 专属：内核驱动框架、Kbuild、systemd 集成、内核内部 API；agentrt 专属：跨平台用户态运行时、SDK 四语言、CLI/TUI |

**标准名称**: IRON-9 v2（同源且部分代码共享）

**旧称/禁止使用**: "IRON-9"（指旧版仅语义同源版本）

**参见**: [SC]/[SS]/[IND] 三层标注、五维正交24原则、agentrt-linux（AirymaxOS）

---

### [SC] / [SS] / [IND] 三层标注

**定义**: IRON-9 v2 的三层代码共享模型的标准标注符号，用于文档、代码注释和 API 设计中标注各组件/接口的共享层次。

**三层标注说明**:

| 标注 | 全称 | 含义 | 使用场景 |
|------|------|------|----------|
| [SC] | Shared-Contract | 共享契约层——完全共享代码 | `include/airymax/` 头文件、错误码、syscall 编号、IPC 消息头结构 |
| [SS] | Shared-Semantics | 语义同源层——API 签名相同，实现独立 | MicroCoreRT 调度语义、Cupolas 安全模型、AgentsIPC 传输、MemoryRovol 记忆模型 |
| [IND] | Independent | 完全独立层——各自独立 | 内核驱动框架、Kbuild、systemd 集成、跨平台用户态运行时 |

**使用规则**:
1. 所有跨 agentrt 和 agentrt-linux 的 API 文档必须标注所属层次
2. [SC] 层代码变更必须通过 agentrt + agentrt-linux CI 双向校验
3. [SS] 层 API 签名变更需两端架构评审
4. [IND] 层各自独立演进，无需同步

**参见**: IRON-9 v2、agentrt-linux（AirymaxOS）

---

### 五维正交24原则

**定义**: agentrt-linux（AirymaxOS）架构设计的顶层设计哲学，基于体系并行论（Multibody Cybernetic Intelligent System），具体实例化为五维正交系统。每个维度对应一类核心设计原则，共 24 条原则覆盖 agentrt-linux 设计的所有关键方面。

**五维正交24原则概览表**:

| 维度 | 核心问题 | 原则数量 | 核心思想 |
|------|----------|----------|----------|
| 系统观 (System) | agentrt-linux 作为复杂自适应系统，如何维持动态平衡？ | 4 项 (S-1~S-4) | 反馈闭环、层次分解、总体设计部、涌现性管理 |
| 内核观 (Kernel) | 内核应该做什么，不应该做什么？ | 4 项 (K-1~K-4) | 内核极简、接口契约化、服务隔离、可插拔策略 |
| 认知观 (Cognition) | 智能体如何高效、可靠地进行认知决策？ | 4 项 (C-1~C-4) | 认知层双思考功能、增量演化、记忆卷载、遗忘机制 |
| 工程观 (Engineering) | 如何构建可维护、可测试、可演进的工程系统？ | 8 项 (E-1~E-8) | 安全内生、可观测性、资源确定性、跨平台一致性、命名语义化、错误可追溯、文档即代码、可测试性 |
| 设计美学 (Aesthetics) | 如何让系统不仅正确，而且优雅？ | 4 项 (A-1~A-4) | 极简主义、细节关注、人文关怀、完美主义 |

**标准名称**: 五维正交24原则

**参见**: 五维正交性、体系并行论、IRON-9 v2

**关联文档**: `docs/AirymaxOS/10-architecture/02-five-dimensional-principles.md`（完整落地映射）

---

### 五维正交性

**定义**: 五维正交24原则中五个维度之间的数学关系特性。五维之间相互独立且正交：

- **正交性**：任一维度的原则不依赖于其他维度，可以独立演进
- **交织性**：在具体设计决策中，五个维度同时发挥作用
- **完备性**：五维覆盖了架构设计的所有关键方面，从思想到实践，从功能到美学
- **最小完备集**：五维是当前阶段的最佳实例，可演进而非封闭

**标准名称**: 五维正交性

**参见**: 五维正交24原则、体系并行论

---

### 体系并行论

**定义**: agentrt-linux（AirymaxOS）和 agentrt（AirymaxAgentRT）的底层理论根基（Multibody Cybernetic Intelligent System）。体系并行论将智能体系统建模为多个并行运转的体系（基础体 + 认知体 + 执行体 + 记忆体 + 安全体），体系之间通过契约层交互，各自独立演进但共享顶层设计哲学。

**标准名称**: 体系并行论 (Multibody Cybernetic Intelligent System)

**参见**: 五维正交24原则、IRON-9 v2、Thinkdual（双思考系统）

---

### ErrorCodeSystem / 统一错误码体系

**定义**: agentrt-linux（AirymaxOS）采用双错误码体系：

| 体系 | 适用场景 | 格式 |
|------|---------|------|
| **C 负整数体系**（首要） | C 内核和 daemon 层 | `AGENTRT_OK=0`、`AGENTRT_EINVAL=-2` |
| **SDK 十六进制体系**（次要） | SDK 和外部接口 | `0x0000`-`0x7FFF` 分段 |

权威源唯一：`include/airymax/error.h`（IRON-9 v2 [SC] 共享契约层）为唯一定义源。

**系统内代码**: `AGENTRT_E*` / `AGENTRT_ERR_*`

**禁止**: C 内核代码中使用十六进制错误码；SDK 中使用负整数错误码

**错误码分段规范**:

| 段 | 范围 | 含义 |
|----|------|------|
| 成功 | 0 | `AGENTRT_OK` |
| 通用 | -1 ~ -99 | 通用错误 |
| 系统 | -100 ~ -199 | 系统级错误 |
| 内核 | -200 ~ -299 | 内核错误 |
| 服务 | -300 ~ -399 | 服务错误 |
| LLM | -400 ~ -499 | LLM 推理错误 |
| 执行 | -500 ~ -599 | 执行错误 |
| 记忆 | -600 ~ -699 | 记忆错误 |
| 安全 | -700 ~ -799 | 安全错误 |
| 协议 | -800 ~ -899 | 协议错误 |

**参见**: JSON-RPC 2.0、IRON-9 v2

---

### Forgetting Engine / 遗忘机制

**定义**: 基于艾宾浩斯遗忘曲线驱动的记忆衰减与清理机制。遗忘公式: R(t) = e^(-t/τ)，τ 默认 7 天。支持 NONE/EBBINGHAUS/LINEAR/ACCESS_BASED 四种遗忘策略。

在 agentrt-linux 中，遗忘机制与 MGLRU（多代 LRU）的多代页面回收策略在语义上一致——冷数据被遗忘（归档），热数据被保留（活跃代）。

**系统内代码**: `agentrt_forgetting_*`

**代码目录**: `airymaxos-memory/`

**参见**: MemoryRovol（记忆卷载）、MGLRU（多代 LRU）、MemorySwap（记忆交换算法）

---

### MemorySwap / 记忆交换算法

**定义**: 记忆卷载的存储层级交换机制。借鉴操作系统虚拟内存分页交换，将记忆按访问热度划分为热/温/冷三个存储层级，在不同层级之间自动迁移。在 agentrt-linux 中，MemorySwap 直接利用 MGLRU 的多代回收机制 + CXL 内存池化实现硬件级加速。

**标准名称**: 记忆交换算法（正式）/ MemorySwap（简称）

**系统内代码**: `agentrt_memory_swap_manager_t`

**参见**: MemoryRovol（记忆卷载）、Forgetting Engine（遗忘机制）、MGLRU（多代 LRU）

---

### TaskFlow / 任务流引擎

**定义**: 基于 Pregel BSP（Bulk Synchronous Parallel）图计算模型的任务流引擎，提供超步计算、检查点容错、工作流模式等功能，支撑复杂任务的有向无环图（DAG）编排。

**标准名称**: 任务流引擎 (TaskFlow)

**代码目录**: `airymaxos-cognition/`

**参见**: CoreLoopThree（认知三阶段循环）

---

### MAC / 多智能体协作框架

**定义**: CoreLoopThree 认知层的多智能体协作基础设施，支持独立、协作、共识、委托四种协作模式，以及多数投票、全票通过、加权投票、领导者否决四种共识策略。

**标准名称**: 多智能体协作框架 (MAC)

**旧称/禁止使用**: "Multi-Agent System"（过于泛化）

**系统内代码**: `mac_*` / `agentrt_mac_*`

**参见**: Thinkdual（双思考系统）、CoreLoopThree（认知三阶段循环）

---

### TimeSliceInfer / 分时推理框架

**定义**: 记忆卷载的分时推理框架。将单次"全量加载->推理"分解为多个时间片上的"渐进式加载->增量推理->状态传递"流水线。每一时间片处理一个记忆子集，时间片间通过推理状态向量传递上下文连续性。

**标准名称**: 分时推理框架（正式）/ TimeSliceInfer（简称）

**旧称/禁止使用**: "时间切片推理"

**系统内代码**: `ts_inference_session_t`（设计中）

**参见**: MemoryRovol（记忆卷载）、MemorySwap（记忆交换算法）

---

### Memory Safety Macro System / 安全宏体系

**定义**: agentrt-linux（AirymaxOS）的内存操作安全宏体系，覆盖分配、释放、字符串操作和禁止函数，确保内存操作的类型安全和空指针安全。适用于内核态和用户态代码。

**核心宏**:
- 分配: `AGENTRT_MALLOC` / `AGENTRT_CALLOC` / `AGENTRT_REALLOC`
- 释放: `AGENTRT_FREE` / `MEMORY_FREE_SAFE`
- 字符串: `AGENTRT_STRDUP` / `AGENTRT_STRNCPY_TERM`
- 拷贝: `AGENTRT_MEMCPY_SAFE`
- 清除: `AGENTRT_SEC_CLEAR`

**禁止函数**: `malloc`、`free`、`printf`、`strcpy`、`strcat`、`sprintf`、`gets` 等（完整列表见 `banned_functions.h`）

**参见**: Security by Design（安全内生设计）、MEMORY_FREE_SAFE

---

### MEMORY_FREE_SAFE

**定义**: 安全释放宏。释放内存后将指针置 NULL，防止悬垂指针。

**签名**: `MEMORY_FREE_SAFE(ptr_to_ptr)` — 参数为指向待释放指针的指针

**参见**: Memory Safety Macro System（安全宏体系）

---

### Linux 6.6 内核基线

**定义**: agentrt-linux（AirymaxOS）1.0.1 版本锁定的内核基线版本。所有 agentrt-linux 内核态代码、驱动、安全模块均基于 Linux 6.6 内核基线开发。禁止引用 6.7+ 主线特性作为 6.6 原生能力（IRON-10 / BAN-361）。

**基线核心特性**:
- EEVDF 调度器（Linux 6.6 原生）
- Rust 实验性支持（Linux 6.6 原生）
- MGLRU 多代 LRU（Linux 6.6 原生）
- XFS 在线 fsck（Linux 6.6 原生）
- eBPF kfunc + dynamic pointer（Linux 6.6 原生）
- io_uring 异步 I/O 改进（Linux 6.6 原生）
- agentrt-linux 内核增强：sched_ext（主线 6.12+ 向前移植）

**标准名称**: Linux 6.6 内核基线

**禁止**: 引用 6.7+ 主线特性作为 6.6 原生能力

**参见**: 工程基线、IRON-10、BAN-361

---

### 8 子仓架构

**定义**: agentrt-linux（AirymaxOS）的代码组织架构，按能力域划分为 8 个独立子仓库：

| 子仓 | 能力域 | 代码目录 |
|------|--------|----------|
| airymaxos-kernel | 内核核心 | `airymaxos-kernel/` |
| airymaxos-services | 用户态服务 | `airymaxos-services/` |
| airymaxos-cognition | 认知运行时 | `airymaxos-cognition/` |
| airymaxos-memory | 记忆卷载 | `airymaxos-memory/` |
| airymaxos-security | 安全穹顶 | `airymaxos-security/` |
| airymaxos-cloudnative | 云原生 | `airymaxos-cloudnative/` |
| airymaxos-system | 系统编排 | `airymaxos-system/` |
| airymaxos-tests-linux | 测试框架 | `tests-linux/` |

**参见**: agentrt-linux（AirymaxOS）、IRON-9 v2

---

## 四、标准化名称映射总表

| 标准中文 | 标准英文 | 代码前缀 | IRON-9 v2 层次 | 禁止使用的旧称 |
|----------|----------|---------|---------------|---------------|
| agentrt-linux（极境智能体操作系统） | agentrt-linux（AirymaxOS） | `agentrt_*` | [SC]/[SS]/[IND] | 仅出现 "AirymaxOS" 而无前缀 |
| 微核心运行时 | MicroCoreRT | `agentrt_core_*` | [SS] | 微内核（不加限定词时） |
| Agent 进程间通信 | AgentsIPC | `agentrt_ipc_*` | [SS] | — |
| 安全穹顶 | Cupolas | `cupolas_*` | [SS] | Cupolas安全模块 |
| 记忆卷载 | MemoryRovol | `agentrt_memoryrov_*` | [SS] | 记忆漩涡引擎 |
| 认知三阶段循环 | CoreLoopThree | `agentrt_loop_*` | [SS] | 三层循环运行时、三层一体架构 |
| 双思考系统 | Thinkdual | `tc_*` / `mc_*` / `sc_*` | [SS] | 认知双思系统、双系统认知模型 |
| Agent 调度类 | SCHED_AGENT | `agentrt_sched_agent_*` | [SS] | — |
| 同源且部分代码共享 | IRON-9 v2 | — | — | IRON-9（旧版仅语义同源） |
| 共享契约层 | [SC] Shared-Contract | — | [SC] | — |
| 语义同源层 | [SS] Shared-Semantics | — | [SS] | — |
| 完全独立层 | [IND] Independent | — | [IND] | — |
| 五维正交24原则 | Five-Dimensional Orthogonal 24 Principles | — | [SC] | — |
| 体系并行论 | Multibody Cybernetic Intelligent System | — | [SC] | — |
| 统一错误码体系 | ErrorCodeSystem | `AGENTRT_E*` | [SC] | — |
| 遗忘机制 | Forgetting Engine | `agentrt_forgetting_*` | [SS] | — |
| 记忆交换算法 | MemorySwap | `agentrt_memory_swap_*` | [SS] | — |
| 任务流引擎 | TaskFlow | `agentrt_taskflow_*` | [SS] | — |
| 多智能体协作框架 | MAC | `mac_*` | [SS] | Multi-Agent System |
| 分时推理框架 | TimeSliceInfer | `ts_*` | [SS] | 时间切片推理 |
| 安全宏体系 | Memory Safety Macro System | `AGENTRT_MALLOC` 等 | [SC] | — |
| Linux 6.6 内核基线 | Linux 6.6 Kernel Baseline | — | [IND] | — |
| 8 子仓架构 | 8-Repository Architecture | — | [IND] | — |
| 用户态服务进程 | Daemon | `*_d` 后缀 | [IND] | 守护进程、后端服务层 |
| 安全内生设计 | Security by Design | — | [SC] | — |
| 服务隔离 | Service Isolation | — | [IND] | — |
| 跟踪标识符 | TraceID | — | [SC] | — |

---

## 五、命名规范速查

### 函数命名

| 模块 | 格式 | 示例 | IRON-9 v2 层次 |
|------|------|------|---------------|
| 内核 | `agentrt_core_[action]` | `agentrt_core_init` | [SS] |
| 系统调用 | `agentrt_sys_[domain]_[action]` | `agentrt_sys_task_submit` | [SC] |
| 认知层 | `agentrt_[prefix]_[action]` | `agentrt_tc_context_window_append` | [SS] |
| 安全穹顶 | `cupolas_[subsystem]_[action]` | `cupolas_permission_check` | [SS] |
| 用户态服务 | `[service]_[action]` | `llm_d_complete` | [IND] |
| IPC 通信 | `agentrt_ipc_[action]` | `agentrt_ipc_send` | [SS] |
| SCHED_AGENT | `agentrt_sched_agent_[action]` | `agentrt_sched_agent_submit` | [SS] |
| 多智能体 | `mac_[action]` / `agentrt_mac_[action]` | `mac_framework_create` | [SS] |

### 结构体命名

| 规范 | 格式 | 示例 |
|------|------|------|
| 公共结构体 | `agentrt_[name]_t` | `agentrt_error_t` |
| 子系统结构体 | `[prefix]_[name]_t` | `tc_context_window_t` |
| 配置结构体 | `agentrt_[name]_config_t` | `agentrt_cognition_config_t` |
| IPC 消息头 | `agentrt_ipc_msg_hdr_t` | 128 字节统一消息头 |

### 枚举命名

| 规范 | 格式 | 示例 |
|------|------|------|
| 状态枚举 | `AGENTRT_[NAME]_STATE_*` | `AGENTRT_TASK_STATE_READY` |
| 类型枚举 | `AGENTRT_[NAME]_TYPE_*` | `AGENTRT_PROTOCOL_TYPE_MCP` |
| 错误码枚举 | `AGENTRT_E*` | `AGENTRT_EINVAL` |

### 文件/目录命名

| 规范 | 规则 |
|------|------|
| 子仓目录 | `airymaxos-[domain]/`（如 `airymaxos-kernel/`） |
| 用户态服务目录 | `[name]_d`（如 `llm_d/`） |
| 源文件 | 下划线分隔 + `.c`（如 `thinking_chain.c`） |
| 头文件 | 下划线分隔 + `.h`（如 `cupolas.h`） |
| 测试文件 | `test_[module].c`（如 `test_loop.c`） |
| Rust 源文件 | 下划线分隔 + `.rs`（如 `capability.rs`） |
| eBPF 程序 | `[name]_bpf.c`（如 `sched_agent_bpf.c`） |

---

## 六、附录

### A. Daemon 服务完整列表（12 个）

| 服务 | 功能 | systemd unit |
|------|------|-------------|
| gateway_d | API 网关（MCP/A2A/OpenAI 兼容协议） | `gateway_d.service` |
| llm_d | LLM 推理代理（缓存/计费/多 Provider 路由） | `llm_d.service` |
| tool_d | 工具调用代理（注册/执行/验证） | `tool_d.service` |
| market_d | 市场/注册表服务（Agent/Skill 注册安装） | `market_d.service` |
| sched_d | 任务调度服务 | `sched_d.service` |
| monit_d | 监控服务（指标/追踪/告警） | `monit_d.service` |
| channel_d | 通道服务 | `channel_d.service` |
| observe_d | 观测服务 | `observe_d.service` |
| notify_d | 通知服务 | `notify_d.service` |
| info_d | 信息服务 | `info_d.service` |
| hook_d | 钩子服务（事件处理与系统钩子） | `hook_d.service` |
| plugin_d | 插件服务（外部插件管理） | `plugin_d.service` |

### B. agentrt-linux 用户态驱动服务（3 个）

| 服务 | 功能 | 实现技术 |
|------|------|----------|
| vfs_d | 用户态文件系统 | VFS 用户态化 |
| net_d | 用户态网络栈 | DPDK / AF_XDP |
| drv_d | 用户态驱动框架 | VFIO / libvfio |

### C. 协议适配层完整列表（9 种）

| 类别 | 协议 | 枚举值 |
|------|------|--------|
| 标准协议 | MCP v1 | `PROTO_MCP` |
| 标准协议 | A2A v0.3 | `PROTO_A2A` |
| 标准协议 | AGNTCY ACP | `PROTO_AGNTCY` |
| 集成适配 | OpenAI | `PROTO_OPENAI` |
| 集成适配 | Claude | `PROTO_CLAUDE` |
| 集成适配 | OpenJiuwen | `PROTO_OPENJIUWEN` |
| 集成适配 | OpenClaw | `PROTO_OPENCLAW` |
| 集成适配 | China Eco | `PROTO_CHINA_ECO` |
| 框架适配 | LangChain / AutoGen | — |

### D. 五维正交24原则编号参考

| 编号 | 原则名称 | 核心要点 |
|------|----------|----------|
| S-1 | 反馈闭环 | 系统的每一层必须设计完整的"感知-决策-执行-反馈"闭环 |
| S-2 | 层次分解 | 复杂系统必须按层次分解，每层只依赖其直接下层 |
| S-3 | 总体设计部 | 存在一个只做协调、不做执行的全局决策层 |
| S-4 | 涌现性管理 | 通过良好的模块交互设计，使系统整体产生超越部分之和的涌现行为 |
| K-1 | 内核极简 | 内核只保留不可再分的原子机制 |
| K-2 | 接口契约化 | 所有跨模块交互必须通过明确定义的接口进行 |
| K-3 | 服务隔离 | 所有用户态服务作为独立守护进程运行 |
| K-4 | 可插拔策略 | 所有算法选择都设计为可运行时替换的策略接口 |
| C-1 | 认知层双思考功能 | 系统的每个认知环节都设计快慢两条路径 |
| C-2 | 增量演化 | 智能体不应一次性规划全部步骤，而应分阶段规划 |
| C-3 | 记忆卷载 | 记忆从原始经验中逐层提炼出可复用的知识模式 |
| C-4 | 遗忘机制 | 遗忘不是缺陷，而是记忆系统的核心功能 |
| E-1 | 安全内生 | 安全不是附加层，而是内嵌于系统设计的每一个环节 |
| E-2 | 可观测性 | 系统必须提供完整的可观测性 |
| E-3 | 资源确定性 | 每块资源的生命周期必须是确定性的 |
| E-4 | 跨平台一致性 | 核心业务逻辑与平台无关 |
| E-5 | 命名语义化 | 名称即文档 |
| E-6 | 错误可追溯 | 每个错误必须可追溯到其根源 |
| E-7 | 文档即代码 | 文档与代码同步更新 |
| E-8 | 可测试性 | 测试不是事后补充，而是设计的首要考量 |
| A-1 | 简约至上 | 用最少的接口提供最大的价值 |
| A-2 | 极致细节 | 细节决定成败 |
| A-3 | 人文关怀 | 技术服务于人 |
| A-4 | 完美主义 | 不接受"足够好" |

---

## 参考文献

[1] 体系并行论 — `Basic_Theories/CN_01_体系并行.md`
[2] 认知层理论 — `Basic_Theories/CN_02_认知层设计.md`
[3] 记忆层理论 — `Basic_Theories/CN_03_记忆层设计.md`
[4] 设计原则 — `Basic_Theories/CN_04_系统设计原则.md`
[5] 五维正交24原则与 agentrt-linux（AirymaxOS）落地映射 — `docs/AirymaxOS/10-architecture/02-five-dimensional-principles.md`
[6] agentrt-linux 架构设计 — `docs/AirymaxOS/10-architecture/01-system-architecture.md`
[7] 微内核设计思想详解 — `docs/AirymaxOS/10-architecture/03-microkernel-strategy.md`
[8] agentrt-linux（AirymaxOS）工程基线 — `docs/AirymaxOS/10-architecture/04-engineering-baseline.md`
[9] agentrt-linux 工程标准规范 — `docs/AirymaxOS/50-engineering-standards/README.md`
[10] agentrt 统一术语表 — `docs/AirymaxRT/Capital_Specifications/TERMINOLOGY.md`
[11] agentrt 技术规范 — `docs/AirymaxRT/Capital_Specifications/README.md`

---

**最后更新**: 2026-07-07
**维护者**: SPHARX Ltd.

---

(C) 2025-2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*