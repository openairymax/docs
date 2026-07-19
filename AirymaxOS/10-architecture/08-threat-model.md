Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 安全威胁模型与防护设计

> **文档定位**：agentrt-linux（AirymaxOS）安全威胁模型与防护设计，基于 STRIDE 框架识别资产、分析威胁、设计防护措施，作为 1.0.1 开发阶段安全设计的总纲\
> **文档版本**：0.2.8\
> **最后更新**：2026-07-15\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：STRIDE 威胁建模方法论 + seL4 capability 安全模型（ADR-014）+ Linux 6.6 LSM/Landlock/capability\
> **版本基线**：1.x.x 锁定 Linux 6.6 LTS / 2.x.x 锁定 Linux 7.1（ADR-013）\
> **SSoT 依赖声明**：本文档涉及的所有安全规则编号（OS-SEC-001\~299、OS-STD-SEC-010\~011、OS-IRON-001/014 等）的**唯一权威来源**为 [`50-engineering-standards/09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md)。本文档不是规则编号 SSoT，仅作为威胁模型与防护设计的技术阐述载体。当本文档与 SSoT 注册表冲突时，以 SSoT 注册表为准。

***

## 目录

- [1. 威胁模型概述](#1-威胁模型概述)
- [2. 资产识别](#2-资产识别)
- [3. 威胁分析（STRIDE 矩阵）](#3-威胁分析stride-矩阵)
- [4. 防护措施设计](#4-防护措施设计)
- [5. 安全属性保证与不保证](#5-安全属性保证与不保证)
- [6. 安全 CI 门禁](#6-安全-ci-门禁)
- [7. 漏洞响应流程](#7-漏洞响应流程)
- [8. 8 子仓安全责任矩阵](#8-8-子仓安全责任矩阵)

***

## 1. 威胁模型概述

### 1.1 威胁建模方法论

agentrt-linux（AirymaxOS）采用 **STRIDE** 威胁建模方法论作为安全设计的统一框架。STRIDE 将威胁划分为六个正交维度，覆盖智能体操作系统在内核态、用户态、Agent 应用层与网络边界所面临的全部典型威胁类别：

| 维度    | 英文全称                   | 威胁本质 | 在 agentrt-linux 中的核心关切                     |
| ----- | ---------------------- | ---- | ------------------------------------------ |
| **S** | Spoofing               | 身份伪造 | 伪造 Agent 身份、伪造 syscall 来源、伪造 capability 令牌 |
| **T** | Tampering              | 数据篡改 | 篡改 IPC 消息、篡改 \[SC] 共享契约头文件、篡改 capability 表 |
| **R** | Repudiation            | 抵赖否认 | Agent 否认操作行为、审计日志缺失或被擦除                    |
| **I** | Information Disclosure | 信息泄露 | 内核内存泄露、跨租户 Agent 数据泄露、LLM 推理状态泄露           |
| **D** | Denial of Service      | 拒绝服务 | 资源耗尽、IPC 风暴、LLM 推理饥饿、capability 表膨胀        |
| **E** | Elevation of Privilege | 权限提升 | 内核提权、capability 绕过、Landlock 沙箱逃逸           |

STRIDE 与 agentrt-linux 的 capability 安全模型（ADR-014，seL4 风格）形成正交覆盖：capability 解决"谁能做什么"的授权问题，STRIDE 解决"还会出什么问题"的威胁面问题。两者共同构成 1.0.1 开发阶段安全设计的双支柱。

### 1.2 安全边界

agentrt-linux 划定三类安全边界，威胁分析以边界为坐标系展开：

| 边界            | 划分依据                               | 隔离机制                                          | 跨边界通信通道                                                       |
| ------------- | ---------------------------------- | --------------------------------------------- | ------------------------------------------------------------- |
| **内核态 / 用户态** | 特权级（ring 0 / ring 3）               | MMU + CPU 特权级 + syscall 门                     | 系统调用（`airy_sys_call` 等 4 核心 syscall，v1.1）+ io\_uring + eBPF kfunc |
| **Agent 租户间** | Agent 命名空间 + capability 空间（CSpace） | CSpace 隔离 + Landlock 域叠加 + Cupolas 命名空间标签     | IPC 消息（io_uring `IORING_OP_URING_CMD`，需 Badge 授权）       |
| **网络边界**      | 主机网络命名空间 + Cupolas 网络子系统           | 网络命名空间 + netfilter + Cupolas Network Security | 网络协议栈（受 LSM `socket_*` 钩子约束）                                  |

### 1.3 信任边界定义

agentrt-linux 定义三级信任边界（L1/L2/L3），与 [`110-security/README.md`](../110-security/README.md) 第 1.1 节安全体系分层对齐。本文档聚焦威胁建模视角的信任边界，与 L1-L9 安全体系分层在语义上对应但视角不同：

| 信任级别   | 名称         | 信任主体                                                         | 信任依据                                                 | 失效后果              |
| ------ | ---------- | ------------------------------------------------------------ | ---------------------------------------------------- | ----------------- |
| **L1** | 内核信任       | 内核镜像（kernel 子仓）+ 内核模块 + eBPF 程序                              | 编译时签名验证 + Lockdown + 形式化验证（tests-linux）              | 全系统沦陷，所有资产暴露      |
| **L2** | 系统服务信任     | 12 daemons（services 子仓）+ security 子仓 LSM/Cupolas + memory 子仓 | capability 约束 + Landlock 沙箱 + Cupolas 裁决             | 跨租户泄露、单服务沦陷       |
| **L3** | Agent 应用信任 | Agent 进程（cognition 子仓 CoreLoopThree + 用户 Agent）              | capability 令牌 + Landlock 3 层域（L0 全局 + L1 角色 + L2 实例） | 单 Agent 越权、推理数据泄露 |

> **信任递减原则**：L1 → L2 → L3 信任度严格递减。L1 代码量必须最小化（微内核化改造，ADR-012），L2 服务用户态化并受 capability 强制，L3 Agent 必须在 Landlock 沙箱内运行且施加失败即中止启动（OS-STD-SEC-011）。

***

## 2. 资产识别

资产识别是威胁建模的起点。agentrt-linux 识别以下 8 类核心安全资产，每类资产标注其信任级别、价值评估与泄露/篡改后果：

| #  | 资产                                      | 信任级别  | 价值评估 | 关联 SSoT 规则       | 泄露/篡改后果                  |
| -- | --------------------------------------- | ----- | ---- | ---------------- | ------------------------ |
| A1 | **内核内存空间**                              | L1    | 极高   | OS-SEC-125\~127  | 内核地址泄露 → KASLR 绕过 → 内核提权 |
| A2 | **\[SC] 共享契约层头文件（10 个）**                 | L1    | 极高   | OS-IRON-014      | 双端契约不一致 → ABI 破坏 → 内存破坏  |
| A3 | **Capability 表（seL4 风格，41 ID）**         | L1    | 极高   | OS-SEC-001\~099  | capability 伪造/绕过 → 权限提升  |
| A4 | **IPC 消息通道（128B 消息头，magic 0x41524531）** | L2    | 高    | OS-IFACE-003/004 | 消息篡改/伪造 → 跨租户越权通信        |
| A5 | **Agent 任务描述符（magic 0x41475453）**       | L2    | 高    | OS-IFACE-001     | 任务描述符伪造 → 身份冒充           |
| A6 | **LLM 推理状态（CoreLoopThree kthread）**     | L2    | 高    | OS-ARCH-005/006  | 推理状态泄露 → 跨租户认知数据泄露       |
| A7 | **密钥与凭证（vault）**                        | L1/L2 | 极高   | OS-SEC-122       | 密钥泄露 → 全量数据解密、签名伪造       |
| A8 | **网络栈**                                 | L2    | 中高   | OS-SEC-001       | 网络栈劫持 → 流量窃听、C2 通道       |

### 2.1 资产详细说明

**A1 内核内存空间**：包括内核代码段、数据段、堆栈、slab/slub 缓存。内核内存是 L1 信任根的物理载体，任何向用户态泄露内核地址（`copy_to_user` 含内核指针）或未初始化内存的行为均违反 OS-SEC-125/126。防护依赖 `copy_from_user`/`copy_to_user` 边界检查（OS-SEC-111）、`memzero_explicit`/`kfree_sensitive` 敏感数据清零（OS-SEC-127）与 KASLR。

**A2 \[SC] 共享契约层头文件（10 个）**：10 个 [SC] 核心头文件（`error.h`、`log_types.h`、`memory_types.h`、`security_types.h`、`cognition_types.h`、`sched.h`、`ipc.h`、`syscalls.h`、`uapi_compat.h`、`lsm_types.h`）+ 1 个补充共享文件（`bpf_struct_ops.h`），唯一物理宿主于 `kernel/include/uapi/linux/airymax/`。OS-IRON-014 强制其为单一数据源，禁止物理副本。任何篡改将导致 agentrt 用户态与 agentrt-linux 内核态契约不一致，引发 ABI 破坏与内存破坏。

**A3 Capability 表**：seL4 风格 CNode 表 + POSIX 41 ID capability（编号 0-40，`CAP_LAST_CAP=CAP_CHECKPOINT_RESTORE=40`）+ 3 个 Airymax 专属（编号 41-43，`AIRY_CAP_AGENT_SPAWN` 等）。capability 使用 `LSM_ORDER_FIRST`（OLK 6.6 硬编码，仅用于 capabilities），永远在 Landlock/Cupolas 之前执行；airy_lsm 使用 `LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位置于 capability 之后。伪造或绕过 capability 检查直接导致权限提升（OS-SEC-121）。

**A4 IPC 消息通道**：128B 消息头 `struct airy_ipc_msg_hdr`，magic `0x41524531`（'ARE1'），5 种 payload type，4 种操作码（SEND/RECV/SEND\_BATCH/CANCEL）。magic 与 checksum 共同保证消息完整性，`trace_id` 支持全链路追踪。IPC 是 Agent 间通信的唯一合法通道，篡改将导致跨租户越权。

**A5 Agent 任务描述符**：`struct airy_task_desc`，magic `0x41475453`（'AGTS'），sched_tac 调度策略（SCHED_DEADLINE/SCHED_FIFO/EEVDF），优先级 0-139，`MAC_MAX_AGENTS=1024`。描述符伪造将导致 Agent 身份冒充，绕过 capability 授权。

**A6 LLM 推理状态**：CoreLoopThree kthread 维护的三阶段认知循环状态（`cognition_types.h` 三阶段枚举）。推理状态含 Agent 上下文、提示词、中间张量，属高价值认知数据。跨租户泄露违反 Agent 数据隔离原则。

**A7 密钥与凭证（vault）**：Cupolas Vault backend 抽象管理的密钥存储，对接 4 层密钥环（builtin/secondary/machine/platform）+ TPM + 机密计算。capability 令牌经安全通道传递，禁止日志打印原始值（OS-SEC-122）。

**A8 网络栈**：Linux 6.6 网络协议栈 + Cupolas Network Security 子系统。受 LSM `socket_create`/`socket_connect`/`socket_bind` 钩子约束，跨命名空间访问受 Cupolas 裁决。

***

## 3. 威胁分析（STRIDE 矩阵）

### 3.1 S - Spoofing（身份伪造）

| 维度       | 分析                                                                                                                                                                                                                                                   |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **攻击面**  | Agent 身份注册路径、syscall 入口、capability 令牌传递路径、IPC 消息 `src_task` 字段                                                                                                                                                                                       |
| **攻击向量** | ① 伪造 `airy_task_desc` magic `0x41475453` 冒充合法 Agent；② 伪造 syscall 来源（用户态进程冒充系统服务调用特权 syscall）；③ 重放/伪造 capability 令牌（`cap_id`）；④ 伪造 IPC 消息 `src_task` 字段冒充发送方                                                                                          |
| **攻击场景** | 恶意 Agent A 构造伪 magic 的任务描述符注册自身，获取合法 Agent 身份后，通过 IPC 向 Agent B 发送伪造 `src_task` 的消息，诱导 B 执行敏感操作（如释放共享记忆卷载 MemoryRovol）                                                                                                                               |
| **影响评估** | 高。身份冒充破坏 capability 授权前提，可引发越权访问、数据篡改、凭证窃取                                                                                                                                                                                                           |
| **防护措施** | ① 任务描述符 magic 强制校验（`0x41475453`），不匹配即 `-EINVAL`；② capability 令牌由内核生成 opaque handle，用户态无法伪造（seL4 不可伪造性）；③ IPC 消息 `src_task` 字段由内核填充，禁止用户态设置，并通过 `src_task` capability 检查验证来源；④ syscall 入口由 capability 守卫（OS-IFACE-001），无对应 capability 的调用返回 `-EACCES` |

### 3.2 T - Tampering（数据篡改）

| 维度       | 分析                                                                                                                                                                                                                                                                                                               |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **攻击面**  | IPC 消息缓冲区、\[SC] 共享契约头文件物理宿主、capability CNode 表、LLM 推理状态内存、审计哈希链                                                                                                                                                                                                                                                  |
| **攻击向量** | ① 篡改 IPC 消息体（绕过 magic/checksum 校验）；② 篡改 `kernel/include/uapi/linux/airymax/` 下 \[SC] 头文件导致双端契约漂移；③ 通过内存破坏漏洞（UAF/越界写）篡改 CNode `badge` 或 `state` 字段；④ 篡改审计日志擦除痕迹                                                                                                                                                              |
| **攻击场景** | 攻击者利用内核内存破坏漏洞（如 slab 越界写）修改某 Agent 的 CNode `badge` 字段，将只读 capability 提权为读写 capability，进而篡改目标 Agent 的 MemoryRovol 卷载数据                                                                                                                                                                                            |
| **影响评估** | 极高。capability 篡改直接导致权限提升；\[SC] 头文件篡改破坏双端 ABI 一致性                                                                                                                                                                                                                                                                 |
| **防护措施** | ① IPC 消息头 magic `0x41524531` + checksum 完整性校验，校验失败丢弃并告警；② \[SC] 头文件物理宿主单一化（OS-IRON-014），CI 强制双端一致性检查（`fix_sc_header_consistency.py`）；③ CNode 表 `__attribute__((aligned(64)))` 缓存行对齐 + RCU 保护（OS-SEC-002 钩子链表 RCU 不变量）；④ 审计事件采用哈希链结构（Cupolas Audit 子系统），篡改可检测；⑤ 内存破坏防护依赖 OS-SEC-110\~127（边界检查、溢出检测、悬挂指针防护、数据竞争防护） |

### 3.3 R - Repudiation（抵赖否认）

| 维度       | 分析                                                                                                                                                                                                                            |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **攻击面**  | Agent 操作审计日志、capability 操作日志、IPC 消息发送/接收记录、LLM 推理决策记录                                                                                                                                                                         |
| **攻击向量** | ① Agent 执行敏感操作后否认，声称日志被伪造；② 攻击者擦除/篡改审计日志消除痕迹；③ IPC 消息无 `trace_id` 关联，无法追溯操作链路；④ 日志缺失 `agent_id` 字段无法归因                                                                                                                        |
| **攻击场景** | Agent A 通过 IPC 触发 Agent B 释放共享资源，事后 A 否认发送过该消息。若日志缺失 `trace_id` 关联或 `agent_id` 归因，无法举证                                                                                                                                        |
| **影响评估** | 中高。抵赖破坏责任归因，影响多租户环境下的责任界定与计费                                                                                                                                                                                                  |
| **防护措施** | ① 审计事件必须包含 `agent_id` 字段，缺失即丢弃并告警（OS-SEC-010）；② IPC 消息携带 `trace_id` 全链路追踪，发送/接收/处理全程关联；③ Cupolas Audit 审计哈希链（前向链接），单条篡改导致链断裂可检测；④ capability 操作（申请/派生/撤销/传递）全部可审计（capability 设计目标 §1.3 第 5 条）；⑤ 审计日志写入只追加（append-only）存储，禁止覆盖 |

### 3.4 I - Information Disclosure（信息泄露）

| 维度       | 分析                                                                                                                                                                                                                                                              |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **攻击面**  | 内核内存（`copy_to_user` 路径）、跨租户 Agent 内存、LLM 推理状态、capability 令牌、密钥 vault、IPC 消息 payload                                                                                                                                                                             |
| **攻击向量** | ① `copy_to_user` 拷贝含内核地址或未初始化内存（OS-SEC-125/126）；② 跨租户 Agent 通过内存破坏读取他租户内存；③ LLM 推理状态（提示词/中间张量）经共享内存泄露；④ capability 令牌被日志打印（OS-SEC-122）；⑤ IPC payload 跨租户可见                                                                                                      |
| **攻击场景** | 攻击者利用 `copy_to_user` 未清零的内核缓冲区读取相邻内核数据（含 capability 令牌或 KASLR 基址），进而绕过 KASLR 发起内核提权                                                                                                                                                                             |
| **影响评估** | 高。内核地址泄露导致 KASLR 失效；跨租户数据泄露违反隔离契约；密钥泄露导致全量数据暴露                                                                                                                                                                                                                  |
| **防护措施** | ① `copy_to_user` 前确保不含内核地址/未初始化内存（OS-SEC-125）；② 禁止向用户态泄露内核地址（OS-SEC-126）；③ 敏感数据清零 `memzero_explicit`/`kfree_sensitive`（OS-SEC-127）；④ capability 令牌安全通道传递，禁止日志打印原始值（OS-SEC-122）；⑤ Agent 租户间内存隔离（CSpace + Landlock 域 + 命名空间）；⑥ 机密计算（TDX/SEV-SNP）内存加密，宿主机不可见（§4.4） |

### 3.5 D - Denial of Service（拒绝服务）

| 维度       | 分析                                                                                                                                                                                                                                                                                                          |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **攻击面**  | 内核资源（slab/kfifo/等待队列）、IPC 通道、capability 表、LLM 推理调度（CoreLoopThree kthread）、CPU/内存/IO                                                                                                                                                                                                                         |
| **攻击向量** | ① 资源耗尽——恶意 Agent 耗尽 slab/kfifo 导致其他 Agent 无法分配；② IPC 风暴——高频 `AIRY_IPC_OP_SEND` 占满 fastpath 与 slowpath 队列；③ LLM 推理饥饿——恶意 Agent 持续提交推理请求占用 CoreLoopThree kthread；④ capability 表膨胀——派生大量 capability 耗尽 CSpace；⑤ 触发频繁 `cond_resched()` 路径拖慢调度                                                                   |
| **攻击场景** | 恶意 Agent 以线速发送 IPC 消息（SEND\_BATCH 批量发送），占满 kfifo 与等待队列，导致合法 Agent 的 io_uring CQE poll 长期阻塞，引发系统级 DoS                                                                                                                                                                                                          |
| **影响评估** | 中高。单租户 DoS 可降级全系统可用性，但不应影响内核 L1 稳定性                                                                                                                                                                                                                                                                         |
| **防护措施** | ① IPC 每租户速率限制（per-Agent token bucket）；② capability 表大小上限 `MAC_MAX_AGENTS=1024`，CNode 槽位上限强制；③ CoreLoopThree kthread 公平调度（sched\_ext SCHED\_AGENT，按 Agent 配额）；④ 长循环抢占点规则——耗时 >1ms 或迭代 >1000 次必须 `cond_resched()`（OS-KER-228），防止单 Agent 独占 CPU；⑤ Landlock 沙箱限制 Agent 资源访问面；⑥ kfifo 满时 slowpath 排队而非丢消息，配合超时机制 |

### 3.6 E - Elevation of Privilege（权限提升）

| 维度       | 分析                                                                                                                                                                                                                                                                                                                                                                                                                        |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **攻击面**  | 内核提权路径、capability 检查路径、Landlock 域施加路径、沙箱逃逸路径、模块加载路径                                                                                                                                                                                                                                                                                                                                                                       |
| **攻击向量** | ① 内核提权——利用内核漏洞（UAF/越界写）修改 `cred` 结构提升权限；② capability 绕过——绕过 `LSM_ORDER_FIRST` capability 检查或伪造 capability；③ Landlock 沙箱逃逸——延迟施加 `landlock_restrict_self` 或域叠加少于 3 层；④ 模块加载提权——加载未签名内核模块/eBPF 程序；⑤ 固件/硬件路径（不在保证范围，见 §5.2）                                                                                                                                                                                                  |
| **攻击场景** | 攻击者利用内核 UAF 漏洞修改自身 `cred->cap_effective` 位图，获取 `CAP_SYS_ADMIN` 后绕过 capability 与 Landlock 全部检查，逃逸沙箱获得宿主权限                                                                                                                                                                                                                                                                                                                  |
| **影响评估** | 极高。提权是安全防线最终失守点，可导致全系统沦陷                                                                                                                                                                                                                                                                                                                                                                                                  |
| **防护措施** | ① capability 使用 `LSM_ORDER_FIRST`（OLK 6.6 硬编码，仅用于 capabilities）永远第一个执行，不可绕过（OS-SEC-001 LSM 钩子内置每一关键路径）；airy_lsm 使用 `LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位置于 capability 之后；② Landlock `landlock_restrict_self` 必须在 Agent fork() 后立即调用，禁止延迟施加（OS-SEC-014）；③ Agent 沙箱必须至少 3 层域叠加（L0 全局 + L1 角色 + L2 实例），掩码合并用交集语义（OS-SEC-012/013）；④ 沙箱施加失败必须中止 Agent 启动，禁止降级运行（OS-STD-SEC-011）；⑤ 内核模块签名 `CONFIG_MODULE_SIG=y` + eBPF JIT 始终启用；⑥ Lockdown 模式限制 root；⑦ Rust 安全子集减少内存安全漏洞（OS-SEC-201\~247）；⑧ 形式化验证（tests-linux 子仓，Kani/Creusot） |

### 3.7 STRIDE 威胁矩阵汇总

| 威胁   | 主要受影响资产  | 首要防护规则                                 | 防护强度     |
| ---- | -------- | -------------------------------------- | -------- |
| S 伪造 | A3/A4/A5 | OS-IFACE-001、capability 不可伪造           | 强        |
| T 篡改 | A2/A3/A4 | OS-IRON-014、OS-SEC-002、IPC checksum    | 强        |
| R 抵赖 | A6 审计    | OS-SEC-010、trace\_id、哈希链               | 强        |
| I 泄露 | A1/A6/A7 | OS-SEC-122/125\~127、机密计算               | 强（不含侧信道） |
| D 拒绝 | A4/A6    | 速率限制、调度公平、抢占点                          | 中强       |
| E 提权 | A1/A3    | OS-SEC-001、OS-STD-SEC-011、Landlock 3 层 | 强        |

***

## 4. 防护措施设计

### 4.1 Capability-Based Security（OS-SEC-001\~099）

agentrt-linux 采用 **seL4 风格 + POSIX 混合** capability 安全模型，详细工程契约见 [`110-security/03-capability-model.md`](../110-security/03-capability-model.md)。

#### 4.1.1 seL4 风格 capability 模型

| 特性     | 实现方式                                    | 对应威胁          |
| ------ | --------------------------------------- | ------------- |
| 不可伪造   | capability 令牌由内核生成，用户态只持有 opaque handle | S（身份伪造）、E（提权） |
| 最小权限   | 每个安全敏感操作独立申请 capability 令牌              | E（提权）         |
| 递归撤销   | Agent 终止时递归撤销所有派生 capability（MDB 派生链）   | E（权限残留）       |
| IPC 传递 | 通过 `AIRY_IPC_F_CAP_CARRY` 跨进程传递         | 跨租户授权         |
| 审计追踪   | 申请/派生/撤销/传递全可审计                         | R（抵赖）         |

capability 使用 `LSM_ORDER_FIRST`（OLK 6.6 硬编码，仅用于 capabilities）作为第一个 LSM，永远在 Landlock/Cupolas 之前执行；airy_lsm 使用 `LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位置于 capability 之后。Cupolas 必须在 capability 之后初始化（OS-SEC-003），保证授权顺序。

#### 4.1.2 41 ID capability 类型

对齐 `security_types.h`（\[SC] 共享头文件）POSIX capability 41 ID 枚举：

- **POSIX 41 ID**（编号 0-40）：完整对齐 Linux 6.6 标准 41 个 POSIX capability（含 `CAP_PERFMON=38`、`CAP_BPF=39`、`CAP_CHECKPOINT_RESTORE=40`，无废弃）
- **Airymax 专属**（编号 41-43）：`AIRY_CAP_AGENT_SPAWN=41`（Agent 生成）等 3 个专属 capability，从 41 开始避免与 Linux 0-40 冲突
- **总数**：41 Linux + 3 Airymax = 44（`AIRY_CAP_AGENT_MAX=44`）

seL4 风格 capability 类型（`airy_cap_type`）：ENDPOINT/TASK/MEMORY/ROVOL/SCHED/FILE/NETWORK/WASM/CAP\_MGMT 共 9 类，覆盖 IPC 端点、任务、内存、记忆卷载、调度、文件、网络、Wasm 模块、capability 管理。

#### 4.1.3 capability 传递与撤销机制

| 操作       | 语义                   | 安全约束                                                   |
| -------- | -------------------- | ------------------------------------------------------ |
| mint     | 派生新 capability（缩减权限） | 派生权限必须是父权限子集                                           |
| mintcopy | 派生并复制                | 同 mint                                                 |
| derive   | 派生（保留 MDB 链）         | 记录 parent/children 关系                                  |
| revoke   | 递归级联撤销               | 撤销父 capability 时递归撤销所有子 capability，借鉴 seL4 `cteRevoke` |

capability 令牌经安全通道传递，禁止日志打印原始值（OS-SEC-122）。Agent 终止时递归撤销，无权限残留。

#### 4.1.4 capability 空间隔离

每个 Agent 拥有独立 CSpace（capability space），CNode 表 `__attribute__((aligned(64)))` 缓存行对齐。跨 Agent 访问 capability 必须通过 IPC 显式传递（`AIRY_IPC_F_CAP_CARRY`），禁止跨 CSpace 直接引用。

### 4.2 LSM + Landlock 沙箱（OS-STD-SEC-010\~011）

#### 4.2.1 Cupolas LSM 钩子（252 ID）

agentrt-linux Cupolas 作为最后初始化的 LSM 注册到框架，与 capability、Landlock、Yama 等共存（不打 `LSM_FLAG_EXCLUSIVE` 标记）。Cupolas 消费 5 类核心钩子：

| 钩子类别   | 代表钩子                                           | Cupolas 职责                        |
| ------ | ---------------------------------------------- | --------------------------------- |
| inode  | `inode_alloc_security`/`inode_permission`      | 初始化 Agent 命名空间标签、校验跨 Agent 命名空间访问 |
| file   | `file_open`                                    | Landlock 完成后追加 Agent 主体校验         |
| task   | `task_create`                                  | Agent 创建权限裁决                      |
| socket | `socket_create`/`socket_connect`/`socket_bind` | 网络访问控制                            |
| cred   | `cred_prepare`                                 | capability 位图继承校验                 |

钩子回调必须返回 \[SC] 4 值枚举之一（ALLOW/DENY/AUDIT/COMPLAIN，OS-SEC-008）。COMPLAIN 裁决通过 A-IPC 询问 daemon，5 秒超时后回退 ALLOW 并记录 AUDIT（OS-SEC-009）。审计事件必须包含 `agent_id` 字段（OS-SEC-010）。

#### 4.2.2 Landlock 用户态沙箱

Landlock 允许非特权进程定义自己的安全策略，提供文件系统与网络细粒度访问控制。在 agentrt-linux 中用于 Agent 进程级隔离（L3 信任边界）。

#### 4.2.3 沙箱失败必须中止启动（OS-STD-SEC-011）

| 规则             | 内容                                                        | 违反后果                  |
| -------------- | --------------------------------------------------------- | --------------------- |
| OS-STD-SEC-010 | `is_initialized()` 失败时必须返回 `-EOPNOTSUPP`，清晰告知用户态          | 用户态无法感知沙箱状态           |
| OS-STD-SEC-011 | 沙箱施加失败必须中止 Agent 启动，禁止降级运行                                | 沙箱失效后 Agent 裸奔 → 沙箱逃逸 |
| OS-SEC-014     | `landlock_restrict_self` 必须在 Agent 进程 fork() 后立即调用，禁止延迟施加 | 延迟窗口内 Agent 可越权访问     |
| OS-SEC-015     | 沙箱创建失败必须终止 Agent 启动，禁止降级运行                                | 同 OS-STD-SEC-011      |

> **铁律**：沙箱施加是 Agent 启动的前置条件，失败即终止，绝不允许"无沙箱运行"。这是 L3 信任边界的底线。

### 4.3 IPC 安全

IPC 是 Agent 间通信的唯一合法通道，安全设计见 [`30-interfaces/02-ipc-protocol.md`](../30-interfaces/02-ipc-protocol.md) 与 [`30-interfaces/07-ipc-fastpath.md`](../30-interfaces/07-ipc-fastpath.md)。

#### 4.3.1 128B 消息头完整性校验

```c
struct airy_ipc_msg_hdr {
    __u32 magic;          /* 魔数 'ARE1'（0x41524531） */
    __u16 opcode;         /* 操作码 SEND/RECV/SEND_BATCH/CANCEL */
    __u16 flags;          /* 标志位（含 AIRY_IPC_F_CAP_CARRY） */
    __u32 src_task;       /* 发送方任务 ID（内核填充，禁止用户态设置） */
    __u32 dst_task;       /* 接收方任务 ID */
    __u64 trace_id;       /* 全链路追踪 ID */
    __u32 checksum;       /* 校验和 */
    __u32 payload_type;   /* 5 种 payload type */
    /* ... 其余字段填充至 128B ... */
};
```

完整性校验：magic `0x41524531` + checksum 双重校验。magic 不匹配直接丢弃；checksum 校验 payload 完整性，失败丢弃并告警。

#### 4.3.2 trace\_id 全链路追踪

每条 IPC 消息携带 `trace_id`，跨 Agent 传递时保持不变，实现发送/接收/处理全程关联。配合审计哈希链（OS-SEC-010 `agent_id` 归因），形成完整责任链路，对抗 R（抵赖）威胁。

#### 4.3.3 消息来源验证（src\_task capability 检查）

`src_task` 字段由内核填充，禁止用户态设置。接收方校验 `src_task` 对应 Agent 是否持有向本 Agent 发送消息的 capability（IPC 端点 capability `AIRY_CAP_TYPE_ENDPOINT`）。无 capability 的消息返回 `-EACCES`，对抗 S（身份伪造）。

#### 4.3.4 IPC 操作码 4 种

| 操作码 | 名称                       | 语义         | 安全检查                                  |
| --- | ------------------------ | ---------- | ------------------------------------- |
| 0   | `AIRY_IPC_OP_SEND`       | 单条发送       | src/dst capability + magic + checksum |
| 1   | `AIRY_IPC_OP_RECV`       | 接收         | 接收方 capability                        |
| 2   | `AIRY_IPC_OP_SEND_BATCH` | 批量发送（≥2 条） | 每条独立校验 + 速率限制                         |
| 3   | `AIRY_IPC_OP_CANCEL`     | 取消已提交请求    | 原提交方 capability                       |

状态机内部状态编号不进入 UAPI（ABI 稳定），对外契约仅 4 操作码与返回码。

### 4.4 机密计算

#### 4.4.1 TDX/SEV-SNP 集成方案

agentrt-linux 机密计算基于 CVM（Confidential Virtual Machine），支持两类硬件 TEE：

| 方案          | 厂商    | 隔离语义                | 适用场景       |
| ----------- | ----- | ------------------- | ---------- |
| Intel TDX   | Intel | VM 级内存加密 + CPU 状态隔离 | Intel 平台部署 |
| AMD SEV-SNP | AMD   | VM 级内存加密 + 逆向控制     | AMD 平台部署   |

CVM 提供三重隔离：内存加密（虚拟机内存对宿主机不可见）、CPU 隔离（虚拟机 CPU 状态对宿主机不可见）、中断隔离（虚拟机中断对宿主机不可见）。

#### 4.4.2 内存加密

CVM 内存加密保护 LLM 推理状态（A6）与密钥 vault（A7）免受宿主机/同主机其他 VM 窥探。对抗 I（信息泄露）威胁中的"云租户侧信道"路径（注：硬件侧信道本身不在保证范围，见 §5.2）。

#### 4.4.3 远程证明

CVM 启动时生成硬件 attestation report，由远程证明服务验证 VM 完整性（内核镜像哈希、初始内存状态）。证明通过后才允许加载敏感密钥到 vault。对抗 T（篡改）与 S（伪造）在启动阶段的攻击。

### 4.5 国密支持

#### 4.5.1 SM2/SM3/SM4 算法集成

| 算法  | 用途       | 替代      | 集成位置                          |
| --- | -------- | ------- | ----------------------------- |
| SM2 | 椭圆曲线数字签名 | ECDSA   | 模块签名、capability 令牌签名、远程证明     |
| SM3 | 密码哈希     | SHA-256 | 审计哈希链、\[SC] 头文件完整性校验、镜像哈希     |
| SM4 | 分组密码     | AES     | vault 密钥加密、IPC payload 加密（可选） |

国密算法与标准算法并存，通过编译配置选择启用，满足国内合规场景。

#### 4.5.2 密码模块合规

国密密码模块遵循 GM/T 系列标准实现，密钥生成、存储、使用、销毁全生命周期管理。capability 令牌传递使用 SM2 签名 + SM4 加密的安全通道（OS-SEC-122 安全通道传递）。

***

## 5. 安全属性保证与不保证

### 5.1 保证的安全属性

| 属性                   | 内容                                | 依据规则                      | 保证机制                                                          |
| -------------------- | --------------------------------- | ------------------------- | ------------------------------------------------------------- |
| **用户空间 ABI 稳定性**     | 用户空间 ABI 永不破坏                     | OS-IRON-001               | \[SC] 共享契约 + CI 双端一致性检查 + syscall 编号 seL4 syscall.xml 式单一来源管理 |
| **\[SC] 头文件单一数据源**   | 10 个 [SC] 核心头文件单一物理宿主         | OS-IRON-014               | 物理宿主 `kernel/include/uapi/linux/airymax/` + 禁止物理副本 + CI 强制               |
| **Capability 隔离**    | Agent 间 CSpace 隔离，capability 不可伪造 | OS-SEC-001\~099           | seL4 风格 opaque handle + `LSM_ORDER_FIRST` + MDB 派生链递归撤销       |
| **IPC 消息完整性**        | 128B 消息头 magic + checksum 完整性     | OS-IFACE-003/004          | magic `0x41524531` + checksum + src\_task 内核填充                |
| **沙箱不可降级**           | Agent 启动必须沙箱，失败即终止                | OS-STD-SEC-011、OS-SEC-015 | 沙箱施加前置条件 + `landlock_restrict_self` fork 后立即调用                |
| **审计可归因**            | 审计事件含 agent\_id，可追溯               | OS-SEC-010                | agent\_id 强制 + trace\_id 全链路 + 哈希链防篡改                         |
| **capability 令牌不泄露** | 令牌不进日志                            | OS-SEC-122                | 安全通道传递 + 禁止日志打印原始值                                            |

### 5.2 不保证的安全属性

agentrt-linux 明确声明以下安全属性**不在保证范围**，使用者需通过部署环境补充防护：

| 属性                    | 不保证原因                                          | 缓解建议                           |
| --------------------- | ---------------------------------------------- | ------------------------------ |
| **侧信道攻击防护**（时序/功耗/缓存） | Linux 6.6 内核未提供完整侧信道防护，capability/IPC 路径存在时序差异 | 部署于隔离核心、关闭超线程、机密计算屏蔽部分侧信道      |
| **物理攻击防护**            | 物理攻击（冷启动、DMA、硬件探针）超出 OS 软件防护范围                 | TPM + 硬件安全模块 + 物理安保            |
| **固件级攻击防护**           | 固件（UEFI/BMC/微码）漏洞超出 agentrt-linux 信任根          | 固件签名验证 + 固件更新 + 供应链可信          |
| **硬件漏洞防护**            | CPU 硬件漏洞（如推测执行类）依赖微码与编译缓解                      | 启用 Retpoline/KPTI 等软件缓解 + 微码更新 |
| **零日漏洞绝对免疫**          | 内核零日漏洞可能绕过所有防护                                 | 及时安全更新 + 漏洞响应流程（§7）+ 最小权限部署    |

> **边界声明**：agentrt-linux 的安全保证止于 L1 内核信任根的软件边界。硬件、固件、物理层面需部署环境配套防护。

***

## 6. 安全 CI 门禁

agentrt-linux 在 CI 流水线中设置多层安全门禁，所有合并请求必须通过方可合入。安全 CI 门禁与 [`50-engineering-standards/05-development-process.md`](../50-engineering-standards/05-development-process.md) 的工程纪律对齐。

### 6.1 静态分析工具链

| 工具                 | 检查范围                                               | 阻断级别             | 对应规则                  |
| ------------------ | -------------------------------------------------- | ---------------- | --------------------- |
| **checkpatch**     | 安全相关编码风格（capability 检查、`copy_from_user` 使用、格式化字符串） | Error 阻断         | OS-SEC-110\~127       |
| **sparse**         | 静态类型安全分析（`__user` 标注、RCU 标注、锁标注）                   | Error 阻断         | OS-SEC-002、OS-SEC-119 |
| **Coccinelle**     | 安全模式匹配（UAF 模式、双重释放、`copy_to_user` 含内核地址）           | Error 阻断         | OS-SEC-118/125/126    |
| **clang-analyzer** | 安全规则（空指针解引用、内存泄漏、整数溢出）                             | Error 阻断         | OS-SEC-112/117        |
| **KCSAN**          | 数据竞争检测（并发访问未保护共享数据）                                | Warning 阻断（关键路径） | OS-SEC-119            |
| **cargo audit**    | Rust 依赖漏洞扫描                                        | CRITICAL 阻断      | OS-SEC-243            |

### 6.2 Fuzzing 集成（syzkaller 风格）

agentrt-linux 集成 syzkaller 风格的内核 fuzzing：

| Fuzzing 目标      | 覆盖 syscall                   | 关注威胁    |
| --------------- | ---------------------------- | ------- |
| `airy_sys_call` | 4 核心 syscall 全覆盖（v1.1）            | S/T/E   |
| IPC 操作码         | SEND/RECV/SEND\_BATCH/CANCEL | T/D     |
| capability 操作   | mint/mintcopy/derive/revoke  | E       |
| Landlock 域施加    | fork 后 restrict\_self 时序     | E（沙箱逃逸） |

Fuzzing 发现的崩溃自动归档为安全 issue，进入漏洞响应流程（§7）。

### 6.3 形式化验证

| 验证器                    | 验证对象                              | 验证属性                 |
| ---------------------- | --------------------------------- | -------------------- |
| Kani                   | Rust 安全子集                         | 借用检查、无 panic         |
| Creusot                | Rust 规范验证                         | 函数契约满足               |
| seL4 风格验证（tests-linux） | capability MDB 派生链、Landlock 域不可逆性 | OS-SEC-002/005 形式化检查 |

***

## 7. 漏洞响应流程

### 7.1 私密报告渠道

安全漏洞通过私密渠道报告，**禁止公开 issue**：

- **邮箱**：<security@spharx.com>
- **PGP 密钥**：发布于本仓库 [`110-security/`](../110-security/) 目录（管理密钥见 vault）
- **响应时效**：收到报告后 48 小时内确认，5 个工作日内初步评估

### 7.2 CVSS v3.1 分级

采用 CVSS v3.1 标准分级：

| 等级       | CVSS 区间  | agentrt-linux 典型场景      |
| -------- | -------- | ----------------------- |
| Critical | 9.0-10.0 | 内核提权、capability 绕过、沙箱逃逸 |
| High     | 7.0-8.9  | 跨租户数据泄露、IPC 篡改、密钥泄露     |
| Medium   | 4.0-6.9  | DoS、审计缺失、信息泄露（非敏感）      |
| Low      | 0.1-3.9  | 配置缺陷、文档安全错误             |

### 7.3 修复时间线

| 等级       | 修复时限 | 安全公告      |
| -------- | ---- | --------- |
| Critical | 7 天  | 修复即发布安全公告 |
| High     | 30 天 | 修复即发布安全公告 |
| Medium   | 90 天 | 随版本发布附带公告 |
| Low      | 下一版本 | 版本日志记录    |

### 7.4 披露策略

agentrt-linux 采用**协调披露 + 90 天宽限期**策略：

1. **私密修复**：报告者与维护者私密协作修复
2. **修复发布**：修复随版本发布，同步发布安全公告（CVE 编号，如适用）
3. **宽限期**：修复发布后 90 天宽限期，报告者可公开技术细节
4. **例外**：已被野外利用的漏洞立即披露，无宽限期

***

## 8. 8 子仓安全责任矩阵

| 子仓              | 安全责任                                                                                          | 关键 OS-SEC 规则                                                   |
| --------------- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **kernel**      | 内核提权防护、capability 强制（`LSM_ORDER_FIRST`）、syscall 守卫、内核内存保护、KASLR、Lockdown                      | OS-SEC-001、OS-SEC-110\~127、OS-SEC-125\~127、OS-IFACE-001        |
| **security**    | LSM/Cupolas 实现、capability 模型、Landlock 沙箱、国密 SM2/SM3/SM4、机密计算（TDX/SEV-SNP）、审计哈希链、Vault backend | OS-SEC-001\~099、OS-STD-SEC-010\~011、OS-SEC-008\~010、OS-SEC-122 |
| **services**    | 用户态服务隔离、12 daemons 沙箱、VFS 安全、网络栈安全、systemd 集成安全                                               | OS-SEC-004\~007、OS-SEC-012\~015                                |
| **memory**      | 内存隔离、CXL 内存池安全、MGLRU 冷热分层安全、MemoryRovol 卷载隔离、PMEM 安全                                          | OS-SEC-119（数据竞争）、OS-SEC-127（敏感清零）                              |
| **cognition**   | LLM 推理隔离（CoreLoopThree kthread）、Wasm 3.0 沙箱、超节点沙箱、双系统协同安全、推理状态防泄露                             | OS-ARCH-005/006、OS-SEC-201\~247（Rust）                          |
| **cloudnative** | 容器隔离、网络策略（CNI）、K8s 安全、containerd shim 安全、超节点 OS 隔离                                            | OS-SEC-011（Cupolas 钩子映射）                                       |
| **system**      | 配置安全、包签名验证（RPM/dnf）、DevStation 开发环境安全、基础库安全、shell 安全                                          | OS-IRON-001（ABI 稳定）                                            |
| **tests-linux** | 安全测试覆盖、形式化验证（Kani/Creusot）、kselftest 安全用例、Soak 长时安全测试、fuzzing 集成、混沌工程                         | OS-SEC-002/005（形式化）、OS-SEC-243\~247、OS-TEST-020                |

### 8.1 责任边界说明

- **kernel** 是 L1 信任根，承载 capability 强制与 syscall 守卫，是安全防线的最后一道内核态屏障
- **security** 是安全机制的核心实现子仓，LSM/Cupolas/capability/Landlock/国密/机密计算全部落地于此
- **services** 与 **cognition** 是 L2 系统服务信任的主要承载者，受 kernel capability 强制与自身 Landlock 沙箱约束
- **tests-linux** 通过形式化验证与 fuzzing 为安全属性提供工程证据，是"保证的安全属性"可信度的来源
- 8 子仓共同构成纵深防御，任一子仓的安全失守不应导致全系统沦陷（最小权限 + 隔离 + 审计）

***

## 9. 相关文档

- [`10-architecture/README.md`](README.md)——架构设计总览（父文档）
- [`10-architecture/03-microkernel-strategy.md`](03-microkernel-strategy.md)——微内核设计思想（capability/IPC 落地策略）
- [`110-security/README.md`](../110-security/README.md)——安全加固设计主索引
- [`110-security/01-lsm-framework.md`](../110-security/01-lsm-framework.md)——LSM 框架详解
- [`110-security/02-landlock-sandbox.md`](../110-security/02-landlock-sandbox.md)——Landlock 用户态沙箱
- [`110-security/03-capability-model.md`](../110-security/03-capability-model.md)——seL4 风格 capability 模型
- [`30-interfaces/02-ipc-protocol.md`](../30-interfaces/02-ipc-protocol.md)——IPC 协议契约
- [`30-interfaces/07-ipc-fastpath.md`](../30-interfaces/07-ipc-fastpath.md)——IPC fastpath 状态机
- [`50-engineering-standards/09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md)——SSoT 规则编号唯一权威来源（OS-SEC-001\~299）

***

## 10. 参考方法与标准

- **STRIDE** 威胁建模方法论（Microsoft）
- **seL4** 微内核 capability 安全模型（ADR-014 唯一来源）
- **Linux 6.6** LSM 框架 + Landlock + capability + Lockdown + 4 层密钥环
- **CVSS v3.1** 漏洞严重性评分标准
- **GM/T 系列国密标准**（SM2/SM3/SM4）
- **TDX / SEV-SNP** 硬件可信执行环境规范

***

> **文档结束** | agentrt-linux（AirymaxOS）安全威胁模型与防护设计 v0.2.8 | 父文档：[架构设计](README.md) | SSoT：[09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)

