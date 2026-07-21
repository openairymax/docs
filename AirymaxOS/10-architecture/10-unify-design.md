Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax Unify Design 总纲
> **文档定位**：Airymax Unify Design 五模块统一设计思想的唯一权威总纲\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[架构设计总览](01-system-architecture.md)\
> **设计依据**：Airymax Unify Design 设计决策（任务 4）+ Capability Folding 决策（A-IPC 第一块基石重构）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax Unify Design** 的唯一权威源（Single Source of Truth）。A-UEF / A-ULP / A-UCS / A-ULS / A-IPC 五模块的架构定位、模块间关系、四大技术领域覆盖度、共享内存布局、[DSL] 降级策略均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Unify Design 模块边界与领域归属。
>
> 本文件遵循 **sched_tac**（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 语义映射），IPC 采用 **Capability Folding 单平面架构**（io_uring `IORING_OP_URING_CMD` + registered buffer + mmap + fastpath C-S9 Badge 内联校验，不使用 page flipping，无双平面、无独立 capability syscall），安全采用 **纯 C LSM 模块**（不使用 BPF LSM），日志内存采用 **alloc_pages + mmap**（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。
>
> **v1.0.1 Capability Folding 决策声明**（A-IPC 第一块基石）：A-IPC 采用 Capability Folding 设计模式——将 capability check 从独立控制面 syscall"折叠"到 IPC 数据面 fastpath 中。物理载体是 [SC] `ipc.h` Layout C v4 消息头 offset 40-47 的 `capability_badge` 字段（64-bit Native Word：Epoch + Random Tag + Perms）；执行点是 fastpath C-S9 内联校验 `airy_cap_badge_ok()`（~10ns）。6 条硬约束 H1-H6 不可妥协。详见 §8 与 [02-ipc-protocol.md §2.6](../30-interfaces/02-ipc-protocol.md)。

---

## 文档信息卡

- **目标读者**：agentrt-linux 架构师、内核开发者、用户态守护进程开发者、跨端集成工程师
- **前置知识**：理解 [03-microkernel-strategy.md](03-microkernel-strategy.md) 微内核化策略、[06-iron9-shared-model.md](06-iron9-shared-model.md) IRON-9 v3 四层模型、[04-engineering-baseline.md](04-engineering-baseline.md) 工程基线
- **预计阅读时间**：60 分钟
- **核心概念**：A-UEF、A-ULP、A-UCS、A-ULS、A-IPC、[DSL]、sched_tac、四大技术领域
- **复杂度标识**：高级

---

## §1 设计背景：解决 7 个架构冲突点

### 1.1 冲突点全景

在 agentrt-linux v0.1.1 文档体系逐一检查中，暴露出 7 个 P0 级架构冲突点。这些冲突并非孤立的技术表述错误，而是源于"缺少一个统一的设计思想来约束所有模块的边界"。Airymax Unify Design 即为消解这 7 个冲突点而提出。

| # | 冲突点 | 冲突表现 | Unify Design 消解方式 |
|---|--------|---------|---------------------|
| C-01 | sched_ext / SCHED_AGENT 表述不一致 | 50 个文件各自定义调度策略，sched_ext 在 vanilla 6.6 不可用（OLK 6.6 已 backport 但选择不使用） | A-ULS + sched_tac 统一三层调度，零内核调度器修改 |
| C-02 | page flipping 表述不一致 | 6 个文件引用已废弃的 page flipping 零拷贝路径 | A-IPC 统一 IORING_OP_URING_CMD + registered buffer + mmap（v1.1 进一步升级为 Capability Folding 单平面架构） |
| C-03 | VFS 用户态化决策矛盾 | 2 个文件对"VFS 是否完全下沉用户态"结论相反 | A-ULS 统一决策：内核保留路径解析，FS 实现用户态化 |
| C-04 | [SC] 头文件数量不一致 | 多处文档对共享头文件数量有 6/10 等不同说法 | A-UEF/A-ULP/A-UCS 联合约束 [SC] 头文件物理宿主为 `kernel/include/uapi/linux/airymax/`，共 10 个 |
| C-05 | 错误码双轨制 | agentrt 全仓库 `AIRY_E*` 与 `AIRY_ERR_*` 并存 | A-UEF 统一为 `AIRY_E*`（Error 负数空间）+ `AIRY_FAULT_*`（Fault 正数空间） |
| C-06 | 日志双轨制 | `LOG_*` 与 `AIRY_LOG_*` 并存，128B 记录格式多处定义 | A-ULP 统一为 `LOG_*` 枚举 + 128B 固定记录格式，单一权威源 |
| C-07 | DMA 一致性内存误用 | 14 个文件误用 DMA 一致性内存用于日志/IPC 共享内存 | A-ULP/A-IPC 统一 `alloc_pages(GFP_KERNEL)` + mmap，x86_64 默认缓存一致 |
| C-08（v1.0.1 新增） | 控制面/数据面双平面架构死锁 | v1.0 的 12 syscall 中 8 个是 seL4 风格同步阻塞 IPC，与 io_uring 异步数据面形成双平面，导致死锁、不可通约、排序丢失 | A-IPC v1.0.1 Capability Folding 单平面架构：syscall 12→4，IPC 数据传递即能力校验（C-S9 内联 Badge） |

### 1.2 设计哲学

Airymax Unify Design 的设计哲学可概括为四句话：

1. **机制在内核，策略在用户态**——直接借鉴 seL4 微内核思想，内核只保留最小机制（IPC 回调、Ring Buffer、Micro-Supervisor），全部策略下沉到用户态守护进程。
2. **内核冷酷执法，用户温情裁决**——A-ULS 的双 Supervisor 模型，内核侧立即冻结异常 Agent，用户侧根据上下文进行人性化裁决。
3. **每个技术点只允许有一个权威源**——SSoT v2 单一权威源模型，[SC] 共享契约头文件物理宿主唯一，CI 强制逐字节校验。
4. **（v1.0.1 新增）IPC 就是能力校验**——Capability Folding 将 capability check 从独立控制面 syscall"折叠"到 IPC 数据面 fastpath 中。通信 = io_uring 数据面（唯一平面）；授权 = 1 个 Badge；撤销 = 1 条 atomic_inc。这是乔布斯/艾夫"简约就是美"哲学在内核工程上的落地——7 颗"看不见的螺丝"全部拧掉。

### 1.3 与 sched_tac 的关系

sched_tac 是 Airymax Unify Design 在调度领域的具体落地。Unify Design 不替代 sched_tac，而是将 sched_tac 纳入 A-ULS 模块的调度策略实现。二者关系为：

- **sched_tac**：调度机制选型（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 映射）
- **Airymax Unify Design**：整体架构思想，A-ULS 模块内部使用 sched_tac 作为 Agent 8 态生命周期的调度承载

### 1.4 与 Capability Folding 的关系（v1.0.1 新增）

Capability Folding 是 Airymax Unify Design 在通信领域的具体落地，是 A-IPC 模块的最终架构形态。Unify Design 不替代 Capability Folding，而是将 Capability Folding 纳入 A-IPC 模块的 IPC 数据面 + 能力校验统一承载实现。二者关系为：

- **Capability Folding**：IPC 数据面 + 能力校验统一承载（io_uring `IORING_OP_URING_CMD` + fastpath C-S9 Badge 内联校验，syscall 12→4）
- **Airymax Unify Design**：整体架构思想，A-IPC 模块内部使用 Capability Folding 作为 12 daemon 通信网格的承载

> **第一块基石声明**：A-IPC 是 AirymaxOS 的"第一块基石"——后续所有设计（sched_tac / MemoryRovol / sec_d / cogn_d）都建立在 A-IPC 之上。Capability Folding 赋予基石三属性：(1) 不可移动性（6 条硬约束 H1-H6）(2) 承重性（fastpath 100 行代码 + CBMC 全函数验证）(3) 形状确定性（Layout C v4 + opcode 表 + syscall 表）。

---

## §2 五模块总览：A-UEF / A-ULP / A-UCS / A-ULS / A-IPC

Airymax Unify Design 由 5 大模块构成，每个模块有明确的架构定位、权威源、物理宿主与技术领域归属。

```
┌─────────────────────────────────────────────────────────────┐
│  Airymax Unify Design 5 模块统一方案                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ A-UEF         │  │ A-ULP        │  │ A-UCS         │          │
│  │ 统一错误码   │  │ 统一日志    │  │ 统一配置    │          │
│  │ 与故障定义   │  │ 与打印系统  │  │ 管理        │          │
│  │ 领域：观测   │  │ 领域：观测  │  │ 领域：治理  │          │
│  │      + 韧性 │  │             │  │             │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │ A-ULS         │  │ A-IPC        │                          │
│  │ 统一生命周期 │  │ 统一进程间  │                          │
│  │ 监管        │  │ 通信体系    │          │
│  │ 领域：韧性   │  │ 领域：通信  │                          │
│  │      + 治理 │  │      + 韧性 │                          │
│  └─────────────┘  └─────────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| 模块 | 全称 | 架构定位 | 权威源文档 | 物理宿主 |
|------|------|---------|-----------|---------|
| **A-UEF** | Unified Error and Fault Framework | 统一错误码与故障定义体系 | [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) | `kernel/include/uapi/linux/airymax/error.h` |
| **A-ULP** | Unified Logging and Printk Subsystem | 统一日志与打印系统 | [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) + [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) | `kernel/include/uapi/linux/airymax/log_types.h` + `kernel/log/` |
| **A-UCS** | Unified Configuration Subsystem | 统一配置管理体系 | [20-modules/11-unified-config.md](../20-modules/11-unified-config.md) | `kernel/include/uapi/linux/airymax/` + `kernel/config/` |
| **A-ULS** | Unified Lifecycle Supervision Framework | 统一生命周期管理 | [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) + [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md) | `kernel/kernel/superv/` + `services/daemons/macro_d/` |
| **A-IPC** | Unified Airymax IPC Fabric | 统一进程间通信体系 | [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) + [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) | `kernel/include/uapi/linux/airymax/ipc.h` + `kernel/ipc/` |

### 2.1 模块间关系

5 模块并非平铺并列，而是存在明确的依赖与协作关系：

- **A-UEF 是基础**：所有模块的返回值、故障通知均使用 A-UEF 定义的 Error/Fault 码空间。A-UEF 是纯头文件模块，无 `.c` 代码。
- **A-ULP 是观测主干**：A-ULS 的故障通知、A-IPC 的 IPC 异常、A-UCS 的配置变更均通过 A-ULP 的 Ring Buffer 写入观测面。
- **A-UCS 是治理入口**：A-ULS 的调度参数、A-IPC 的 Ring 大小、A-ULP 的日志级别均通过 A-UCS 的 sysctl/JSON 双向同步进行热重载。
- **A-ULS 是韧性核心**：A-UEF 的 Fault 码触发 A-ULS 的 Micro-Supervisor 冷酷执法；A-IPC 的 IPC 队列冻结由 A-ULS 触发。
- **A-IPC 是通信骨干**：A-ULS 的 Macro-Supervisor 通过 A-IPC 的 `io_uring_cmd` 执行裁决；A-ULP 的 eventfd 通知复用 A-IPC 的 io_uring 通道。

---

## §3 四大技术领域覆盖度

Airymax Unify Design 的 5 模块覆盖"通信、治理、观测、韧性"四大技术领域，每个领域有核心模块与辅助模块。

| 技术领域 | 核心模块 | 辅助模块 | 覆盖能力 |
|---------|---------|---------|---------|
| **通信** | A-IPC | A-UEF（错误码传递）+ A-ULP（日志传递） | 进程间消息传递、零拷贝数据面、错误码跨端传递 |
| **治理** | A-UCS + A-ULS | A-UEF（故障分类） | 配置热重载、生命周期裁决、故障分级处置 |
| **观测** | A-ULP | A-UEF（错误码观测）+ A-ULS（心跳观测） | 零拷贝日志、错误码可观测、Agent 状态可观测 |
| **韧性** | A-ULS | A-UEF（故障定义）+ A-IPC（数据面自治）+ [DSL]（降级生存） | 故障检测、冷酷执法、温情裁决、降级生存 |

### 3.1 领域交叉矩阵

```
              通信    治理    观测    韧性
   A-UEF        ●○      ●○      ●○      ●○
   A-ULP       ●○      —       ●●      —
   A-UCS        —       ●●      —       ●○
   A-ULS        ●○      ●●      ●○      ●●
   A-IPC       ●●      —       —       ●○
   [DSL]      —       —       ●○      ●●

   ●● = 核心覆盖    ●○ = 辅助覆盖    — = 不覆盖
```

[DSL]（降级生存层）是 IRON-9 v3 新增的第四层（详见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md)），在 [SC] 头文件损坏/缺失时提供最小可运行子集，是韧性领域的最后防线。

---

## §4 A-UEF（统一错误码与故障定义体系）

**权威源**：[30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) + `kernel/include/uapi/linux/airymax/error.h`

### 4.1 Error vs Fault 分层

A-UEF 的核心设计是将"可恢复的错误"与"不可恢复的故障"分离到两个不重叠的码空间，避免语义混淆：

| 维度 | Error（错误） | Fault（故障） |
|------|--------------|--------------|
| **码空间** | 负数 `[-300, -1]` | 正数 `[0x1000, 0x1FFF]` |
| **可恢复性** | 可恢复，函数返回值 | 不可恢复，触发 Fault Handler |
| **处置方式** | 调用方处理 | Micro-Supervisor 立即接管 |
| **传递路径** | 函数返回值 / IPC 响应码 | eventfd 通知 / Fault Handler |
| **示例** | `AIRY_EPERM=-1`（权限不足） | `AIRY_FAULT_CAP_FORGED=0x1001`（Badge 伪造） |

### 4.2 Error 码空间分配

Error 码空间按来源分层分配，每个分层有明确的值域与语义：

| 分层 | 值域 | 来源 | 数量 |
|------|------|------|------|
| POSIX 码 | `[-1, -40]` | 对齐 Linux errno | 40 |
| IPC 码 | `[-41, -70]` | A-IPC 协议层 | 30 |
| Capability 码 | `[-71, -100]` | 安全子系统 | 30 |
| [SC] 码 | `[-101, -200]` | [SC] 共享契约层 | 100 |
| [DSL] 码 | `[-201, -300]` | [DSL] 降级生存层 | 100 |

### 4.3 Fault 码空间分配

Fault 码从 `0x1000` 起步，每个 Fault 对应一个不可恢复的异常场景，由 Micro-Supervisor 直接接管：

| Fault 码 | 值 | 语义 | 触发场景 |
|---------|-----|------|---------|
| `AIRY_FAULT_CAP_FORGED` | `0x1001` | Badge 伪造 | C-S9 检测到 `badge_randtag` 不匹配（v1.1 从 `AIRY_FAULT_CAP_FAULT` 精确化） |
| `AIRY_FAULT_CAP_LEAK` | `0x1002` | Badge 泄漏 | sec_d 审计检测到跨 Agent 使用 Badge（v1.0.1 新增） |
| `AIRY_FAULT_RING_CORRUPT` | `0x1003` | Ring 数据损坏 | CRC32 校验失败 + Ring 元数据不一致（v1.1 从 `AIRY_FAULT_IPC_FAULT` 精确化） |
| `AIRY_FAULT_TIMEOUT` | `0x1004` | 超时故障 | Agent 心跳超时且未响应冻结 |
| `AIRY_FAULT_ABNORMAL_CAP` | `0x1005` | 异常 Capability | Capability 树完整性校验失败 |
| `AIRY_FAULT_VM_FAULT` | `0x1006` | 虚拟内存故障 | 共享页映射损坏（v1.1 从 `0x1002` 迁移至此） |

### 4.4 [SC] error.h 契约

`kernel/include/uapi/linux/airymax/error.h` 是 A-UEF 的 [SC] 共享契约头文件，agentrt 与 agentrt-linux 物理共享同一份文件，逐字节相同。CI 通过 `sc-dual-ci.yml` 验证两端 `error.h` 逐字节一致。详细契约定义见 [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)。

---

## §5 A-ULP（统一日志与打印系统）

**权威源**：[30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) + [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)

### 5.1 内存方案：alloc_pages + mmap

A-ULP 的日志 Ring Buffer 内存采用 `alloc_pages(GFP_KERNEL)` 分配 + `mmap` 映射到用户态。**明确不使用 DMA 一致性内存**——在 x86_64 架构下，CPU 默认缓存一致（cache-coherent），普通页内存即可满足内核生产者与用户态消费者之间的可见性要求。DMA 一致性内存仅在 DMA 设备直接写内存时才需要，日志场景无此需求。

```c
/* kernel/log/airy_log_ring.c —— 内存分配 */
struct page *pages;
pages = alloc_pages(GFP_KERNEL | __GFP_ZERO, order);
/* mmap 到用户态 Logger Daemon */
vm_flags_set(vma, VM_SHARED | VM_DONTEXPAND);
remap_pfn_range(vma, vma->vm_start, page_to_pfn(pages), size, vma->vm_page_prot);
```

### 5.2 128B 固定记录格式

A-ULP 采用 128 字节固定长度日志记录，对齐 cache line，确保单次 memcpy 完成写入。记录格式详见 [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) §2：

| 字段 | 偏移 | 长度 | 说明 |
|------|------|------|------|
| `magic` | 0 | 4B | `AIRY_LOG_MAGIC = 0x414C4F47`（'ALOG'） |
| `level` | 4 | 2B | 日志级别（LOG_DEBUG ~ LOG_FATAL） |
| `facility` | 6 | 2B | 设施编号（内核模块/daemon 编号） |
| `timestamp_ns` | 8 | 8B | 纳秒时间戳（CLOCK_MONOTONIC） |
| `caller_id` | 16 | 4B | 调用者 ID（pid / kmod id） |
| `payload_len` | 20 | 4B | payload 有效长度 |
| `payload[96]` | 24 | 96B | 原始二进制 payload（不格式化） |
| `reserved` | 120 | 8B | 预留对齐 |

### 5.3 Logger Daemon 分离

A-ULP 严格遵循"Fastpath 禁止格式化"原则。内核生产者仅执行 `reserve → memcpy 128B raw binary → commit` 三步，不进行任何字符串格式化。格式化、过滤、落盘全部由用户态 Logger Daemon 完成：

```
内核 Fastpath（~50-100ns）           用户态 Logger Daemon（异步）
┌──────────────────────────┐        ┌──────────────────────────┐
│ 1. reserve 128B slot     │        │ 1. mmap 读取 Ring Buffer │
│ 2. memcpy 128B raw       │ ──eventfd──▶ 2. sprintf 格式化     │
│ 3. commit 原子索引       │        │ 3. 过滤（level/facility） │
│ 4. eventfd_signal（非阻塞）│        │ 4. 落盘 + 轮转           │
└──────────────────────────┘        └──────────────────────────┘
```

### 5.4 Panic 回退

当系统进入 Panic 状态时，Ring Buffer 可能已损坏或 Logger Daemon 已死亡。A-ULP 回退到 Linux 6.6 原生 `printk_safe` 路径（NMI-safe buffer），确保 Panic 日志可靠落盘。详见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md) §3。

---

## §6 A-UCS（统一配置管理体系）

**权威源**：[20-modules/11-unified-config.md](../20-modules/11-unified-config.md)

### 6.1 [SC] 共享配置 + [SS] 语义同源配置

A-UCS 将配置分为两类，对应 IRON-9 v3 的两个层级：

- **[SC] 共享配置**：仅限二进制布局相关的配置项，如 IPC 消息头大小、Ring Buffer 大小、魔数。这些配置在 `kernel/include/uapi/linux/airymax/` 头文件中以 `#define` 形式定义，两端逐字节相同。
- **[SS] 语义同源配置**：内核侧 Kconfig/sysctl 与用户态 JSON/TOML 语义映射。语义一致，签名独立演进。例如内核 `kernel.sched_rt_runtime_us` 与用户态 JSON `sched.rt_runtime_us` 语义同源。

### 6.2 RCU 热重载

A-UCS 支持 RCU 指针原子切换的热重载机制。sysctl 写入或 JSON 变更触发 `config_reload` 事件，内核通过 `rcu_assign_pointer()` 原子切换配置指针，读者通过 `rcu_dereference()` 获取最新配置，旧配置在宽限期后释放。

```c
/* kernel/config/airy_sysctl.c —— RCU 热重载 */
static struct airy_config __rcu *airy_cfg_ptr;

static int airy_sysctl_handler(struct ctl_table *tp, int write,
                               void *buffer, size_t *lenp, loff_t *ppos)
{
    int ret = proc_dointvec(tp, write, buffer, lenp, ppos);
    if (write) {
        struct airy_config *new_cfg = build_config_from_sysctl();
        struct airy_config *old_cfg;
        old_cfg = rcu_dereference_protected(airy_cfg_ptr, ...);
        rcu_assign_pointer(airy_cfg_ptr, new_cfg);
        synchronize_rcu();
        kfree(old_cfg);
    }
    return ret;
}
```

### 6.3 配置版本号

A-UCS 引入 `AIRY_CONFIG_VERSION` 字段，配置加载时校验版本号，不匹配则拒绝加载并返回 `AIRY_ECFGVERSION`（-101，[SC] 码空间首个值）。

---

## §7 A-ULS（统一生命周期管理）

**权威源**：[20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) + [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md)

### 7.1 Micro-Supervisor（内核冷酷执法）

Micro-Supervisor 是内核态的纯 C LSM 模块（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式），负责检测非法 Capability 并立即执法：

1. **检测**：纯 C LSM 钩子检测到伪造/越权 Capability
2. **冻结**：立即设置 `ring->frozen = true`，IPC fastpath 通过 `unlikely(ring->frozen)` 零开销检查跳过
3. **返回**：返回 `AIRY_FAULT_ABNORMAL_CAP`（0x1005）
4. **通知**：eventfd 通知 Macro-Supervisor

Micro-Supervisor 不做任何"人性化"决策，仅执行冷酷的机制级执法——这是借鉴 seL4 `handleFault()` 的设计：内核只负责检测与通知，裁决交给用户态。

### 7.2 Macro-Supervisor（用户温情裁决）

Macro-Supervisor 是用户态守护进程（`services/daemons/macro_d/`），接收 Micro-Supervisor 的故障通知后进行人性化裁决：

| 裁决选项 | 适用场景 | 执行方式 |
|---------|---------|---------|
| 警告 | 首次轻微违规 | 记录日志，不中断 Agent |
| 降级 | Capability 边界模糊 | 降低优先级、缩减预算 |
| 暂停 | 多次违规 | `kill(SIGSTOP)`，进入 STOPPED 态 |
| 终止 | 严重违规 | `kill(SIGKILL)`，进入 DEAD 态 |

裁决通过 `io_uring_cmd` 执行，避免新增 syscall。Macro-Supervisor 单点故障时，内核 watchdog 直接重启。

### 7.3 Agent 8 态生命周期

A-ULS 定义 Agent 8 态生命周期，与 Linux 进程状态天然映射（sched_tac 的核心成果之一）：

| Agent 态 | Linux 进程状态 | 调度行为 | Macro-Supervisor 操作 |
|---------|--------------|---------|---------------------|
| INACTIVE | 进程不存在 | — | `fork()` 创建 |
| SPAWNING | fork/exec 中 | — | 等待 exec 完成 |
| READY | TASK_RUNNING | 在运行队列等待 | `sched_setattr()` 设置预算/周期 |
| RUNNING | TASK_RUNNING | 正在运行 | 监控 |
| BLOCKED | TASK_INTERRUPTIBLE | 不在运行队列 | 等待唤醒 |
| STOPPING | SIGSTOP 发送中 | 不在运行队列 | `kill(SIGSTOP)` |
| STOPPED | TASK_STOPPED | 不在运行队列 | 裁决后 `kill(SIGCONT)` 或终止 |
| DEAD | EXIT_ZOMBIE | 不在运行队列 | `waitpid()` 回收 |

---

## §8 A-IPC（统一进程间通信体系）

**权威源**：[30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) + [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) + [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md)

> **v1.1 重大变更**：A-IPC 从 v1.0 的"双平面架构（控制面 12 syscall + 数据面 io_uring）"重构为 v1.1 的"Capability Folding 单平面架构（数据面 io_uring 唯一平面 + 控制面 4 syscall 低频管理）"。这是 AirymaxOS 第一块基石的形状确定——后续所有设计（sched_tac / MemoryRovol / sec_d / cogn_d）都建立在此基石之上。

### 8.1 Capability Folding 工程定义

**Capability Folding 是一种将 capability check（能力校验）从独立的控制面操作"折叠"到 IPC 数据面消息传递路径中的设计模式。**

核心论断：**"IPC 不是承载能力校验——IPC 就是能力校验。"**

| 维度 | v1.0 双平面架构 | v1.0.1 Capability Folding 单平面架构 |
|------|---------------|-----------------------------------|
| 控制面 | 12 syscall（8 seL4 IPC 原语 + 4 控制原语），同步阻塞 | 4 syscall（1 Capability Invocation + 3 控制原语），低频管理 |
| 数据面 | io_uring CQE 异步 | io_uring `IORING_OP_URING_CMD` 唯一平面 |
| 能力校验位置 | 控制面 `decode_and_invoke` + 数据面 `cap_check_fast` 双路径 | 数据面 fastpath C-S9 内联（`airy_cap_badge_ok()` ~10ns） |
| 校验与投递 | 两个函数、两个栈帧、两个失败域 | 一个函数、一个栈帧、一个失败域 |
| 隔离基础 | 架构级（seL4 Isabelle/HOL 证明） | 数据结构级（CBMC 全函数验证） |
| 性能 | fastpath ~160ns（不含 cap check）+ slowpath cap check 50-100ns | fastpath ~158ns（含 C-S9 Badge 校验 ~10ns） |

### 8.2 6 条硬约束 H1-H6（不可妥协）

| 约束 | 工程含义 | 违反后果 |
|------|---------|---------|
| H1 | Layout C v4 总长 128 字节，magic 0x41524531 'ARE1'——一经发布即冻结 | 破坏 [SC] 契约，双端不兼容 |
| H2 | `capability_badge` 字段进入 [SC] `ipc.h`，但校验机制属于 [SS] 语义同源层 | [SC]/[SS] 边界模糊，违反 IRON-9 |
| H3 | agentrt 用户态 `capability_badge` 始终为 0，使用传统 cap_t 引用 | agentrt 用户态复杂化 |
| H4 | agentrt-linux 内核 `capability_badge` 由 sec_d 编译，fastpath C-S9 校验 | 安全模型破坏 |
| H5 | 纯 C LSM 模块职责不变（策略/撤销/审计/执法），Badge 校验是其 fastpath 内联 | 违反 SSoT，需重新选型 |
| H6 | [DSL] 降级模式下 `capability_badge` 字段存在但被忽略（值=0，跳过 C-S9 校验） | 降级模式失败，系统不可用 |

### 8.3 Layout C v4 128B 消息头（H1）

A-IPC 的物理载体是 [SC] `ipc.h` Layout C v4 128B 定长消息头（2 cache lines），相对 v1.0 新增 `capability_badge`（offset 40, 8B）与 `crc32`（offset 52, 4B）字段，`reserved` 收缩至 72 字节。

| Offset | 字段 | 大小 | 语义 | 校验位置 |
|--------|------|------|------|---------|
| 0 | magic | 4B | `0x41524531 'ARE1'` | C-S1 |
| 4 | opcode | 2B | 7 opcode（SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE） | C-S2 |
| 6 | flags | 2B | ZEROCOPY/CAP_CARRY/ENCRYPT/COMPRESS/BATCH_TAIL | C-S10 |
| 8 | trace_id | 8B | OpenTelemetry trace ID | 透传 |
| 16 | timestamp_ns | 8B | CLOCK_REALTIME 纳秒 | 透传 |
| 24 | src_task | 8B | 源 task_id，与 Badge 编译时绑定 | C-S9 |
| 32 | dst_task | 8B | 目标 task_id，决定 kfifo 路由 | C-S5/C-S6 |
| 48 | payload_len | 4B | 负载长度 | C-S3/C-S4 |
| 40 | capability_badge | 8B | **Capability Folding 物理载体**：64-bit Native Word（Epoch + Random Tag + Perms），agentrt=0，agentrt-linux 由 sec_d 编译 | C-S9 |
| 52 | crc32 | 4B | CRC32 覆盖 `header[0:52) + payload` | C-S12 |
| 56 | reserved[72] | 72B | 保留，必须为零 | C-S4 |

> **Badge 64-bit Native Word 位布局**：`Epoch<<48 | RandomTag<<16 | Perms`（Epoch 16 位全局代际 + RandomTag 32 位 per-Agent 随机标签 + Perms 16 位权限位图）。详见 [02-ipc-protocol.md §2.6](../30-interfaces/02-ipc-protocol.md)。

### 8.4 单平面架构（H1, H4）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     A-IPC v1.1 单平面架构（Capability Folding）                │
│                                                                              │
│  数据面（io_uring，高频 100K-1M 次/秒）:                                      │
│    IORING_OP_URING_CMD → airy_uring_cmd()                                    │
│      ├─ airy_ipc_validate()    [C-S0~C-S12, 含 C-S9 Badge 内联校验 ~10ns]    │
│      ├─ airy_ipc_deliver_fast() [NONBLOCK, ~158ns, 锁内]                     │
│      └─ airy_ipc_deliver_full() [io-wq 接管, ~600ns-5.5μs, 锁外]             │
│    kfifo (kthread 间, per-cpu SPSC 无锁)                                     │
│    eventfd 通知                                                              │
│                                                                              │
│  控制面（4 个 syscall，低频 < 100 次/秒）:                                     │
│    airy_sys_call        — 仅 sec_d 调用，编译/撤销 Badge（H4）                │
│    airy_sys_rovol_ctl   — MemoryRovol 控制                                   │
│    airy_sys_sched_ctl   — sched_tac 调度策略控制                              │
│    airy_sys_clt_notify  — 客户端事件通知                                      │
│                                                                              │
│  关键不变量:                                                                  │
│    - 控制面不传递 IPC 数据 → 不存在控制面死锁                                 │
│    - 数据面不做管理操作 → fastpath 保持纯线性                                 │
│    - C-S9 在数据面 → 控制面不绕过能力校验（控制面 airy_sys_call 仍走 11 阶段） │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.5 syscall 12 → 4 精确映射

v1.0 的 12 核心 syscall 精简为 v1.1 的 4 核心 syscall。8 个 seL4 风格 IPC 原语全部移除，IPC 数据传递完全由 io_uring 数据面 `IORING_OP_URING_CMD + cmd_op` 承载。完整映射表见 [01-syscalls.md §2.2.1](../30-interfaces/01-syscalls.md)。

| 保留的 4 个 syscall | 注册号 | 职责 |
|---|---|---|
| `airy_sys_call` | 512 | Capability Invocation（sec_d 编译/撤销 Badge + LSM_ctl + Wasm_load） |
| `airy_sys_rovol_ctl` | 513 | MemoryRovol 控制 |
| `airy_sys_sched_ctl` | 514 | sched_tac 调度策略控制 |
| `airy_sys_clt_notify` | 515 | 客户端事件通知 |

### 8.6 fastpath C-S9 Badge 校验（H4, H5）

fastpath 校验链为 **C-S0~C-S12 共 13 项**（详见 [07-ipc-fastpath.md §3.1](../30-interfaces/07-ipc-fastpath.md)），其中 C-S9 是 Capability Folding 的核心执行点：

```c
/* C-S9: Capability Folding Badge 校验（关键步骤，~10ns） */
/* H2: 校验机制属于 [SS] 语义同源层 */
/* H3: agentrt 用户态 capability_badge=0，跳过 Badge 校验 */
/* H4: agentrt-linux 内核 Badge 由 sec_d 编译，C-S9 校验 */
/* H6: [DSL] 降级模式 capability_badge=0，跳过 C-S9 */

/* CAP_REQUEST 自举：Agent 向 sec_d 请求 Badge（无需 Badge） */
if (unlikely(hdr->opcode == AIRY_IPC_OP_CAP_REQUEST)) {
    if (unlikely(hdr->dst_task != AIRY_SEC_D_TASK_ID))
        return -AIRY_ECAP_BADGE;  /* -78 */
    goto cap_request_pass;
}

badge = hdr->capability_badge;
if (badge == 0) {
    if (hdr->flags & AIRY_IPC_F_CAP_CARRY)
        return -AIRY_ECAP_BADGE;  /* -78：CAP_CARRY 置位但 badge=0 矛盾 */
    goto cap_pass;  /* H3/H6: agentrt 用户态或 [DSL] 降级 */
}

/* Badge 三阶段校验：3 个 READ_ONCE + 位运算 + 比较 */
badge_epoch   = AIRY_BADGE_EPOCH(badge);
badge_randtag = AIRY_BADGE_RANDTAG(badge);
badge_perms   = AIRY_BADGE_PERMS(badge);

/* 1. Epoch 校验（1 次 atomic_read）——撤销立即生效 */
global_epoch  = airy_cap_epoch_get();
if (unlikely(badge_epoch != global_epoch))
    return -AIRY_ECAP_EPOCH;  /* -79 */

/* 2. Random Tag 校验（1 次 READ_ONCE）——防伪造 */
agent_randtag = READ_ONCE(agent_caps[hdr->src_task].randtag);
if (unlikely(badge_randtag != agent_randtag)) {
    airy_security_fault(hdr->src_task, AIRY_FAULT_CAP_FORGED);  /* 0x1001 */
    return -AIRY_ECAP_FORGED;  /* -80 */
}

/* 3. 权限校验（1 次 READ_ONCE + 位运算）——opcode 权限位 */
if (unlikely(!airy_cap_has_perm(badge_perms, hdr->opcode)))
    return -AIRY_ECAP_PERM;  /* -81 */
```

> **关键属性**：3 个 `READ_ONCE`、零锁、零 RCU、零 `kmalloc`——纯读取 + 位运算 + 比较。C-S9 不可被绕过（代码审查 + CBMC 静态验证保证）。

### 8.7 数据结构隔离（数据面自治基础）

Capability Folding 的隔离基础是**数据结构级隔离**（非 seL4 架构级隔离），通过三个独立的内存区域实现：

```
1. agent_caps[1024]   能力数据（内核静态数组，16KB）
   ├─ 唯一写者: sec_d（通过 airy_sys_call + COMPILE_BADGE）
   ├─ 读者: fastpath C-S9（READ_ONCE 无锁）
   └─ 失败域: 独立于 IPC 数据投递

2. kfifo              数据投递缓冲区（per-cpu SPSC 无锁）
   ├─ 写者: fastpath kfifo_in
   ├─ 读者: 接收方 CQE
   └─ 失败域: 独立于能力数据

3. io_uring ring      通信原语
   ├─ 写者: 发送方 SQE
   ├─ 读者: 接收方 CQE
   └─ 失败域: 独立于能力数据与 kfifo
```

三者内存区域独立、更新路径独立、失败域独立——这是 v1.1 数据面自治的根本设计。相对 v1.0 的"数据面自治三原则（Ring 生命周期解耦 / 离线缓存校验 / Reconciliation）"，v1.1 升级为"数据结构隔离三原则"：

1. **能力数据隔离**：`agent_caps[]` 由 sec_d 唯一写，fastpath 唯一读，与 IPC 数据路径完全解耦
2. **投递缓冲隔离**：kfifo 由 fastpath 写，接收方读，与能力校验路径完全解耦
3. **通信原语隔离**：io_uring ring 由发送方写 SQE，接收方读 CQE，与能力数据/投递缓冲完全解耦

### 8.8 IPC 队列冻结（A-ULS 触发）

当 Micro-Supervisor 检测到异常时，通过 `airy_sys_call + REVOKE_BADGE` 触发 `atomic_inc(&airy_cap_global_epoch)` 立即撤销所有 Badge（1 行代码，无 drain、无 bitmap、无 IPI），同时通过 `AIRY_IPC_OP_FREEZE` opcode 冻结特定 ring。fastpath 通过 C-S0（`ring->frozen == false`）+ C-S9（`badge_epoch != global_epoch`）双重检查零开销跳过——正常路径不触发冻结检查的分支预测失败，仅异常路径才进入冻结处理。

### 8.9 IRON-9 v3 四层模型在 A-IPC 的落地

| 层次 | A-IPC 落地内容 |
|------|---------------|
| **[SC] 共享契约层** | `ipc.h`：`struct airy_ipc_msg_hdr` Layout C v4 128B 消息头 + 7 opcode + 6 flags + Badge 位布局宏 + Capability 权限位 + `_Static_assert(sizeof == 128)` |
| **[SS] 语义同源层** | `airy_cap_badge_ok()` 内联函数 + `agent_caps[1024]` 静态数组 + `airy_cap_global_epoch` atomic_t + Random Tag 生成 + C-S9 校验逻辑 + sec_d Badge 编译流程 |
| **[IND] 完全独立层** | agentrt 用户态 `capability_badge=0`（H3）↔ agentrt-linux 内核 `capability_badge` 由 sec_d 编译（H4） |
| **[DSL] 降级生存层** | [DSL] 模式下 `capability_badge=0`，fastpath C-S9 跳过 Badge 校验（H6）；IPC 数据面 fastpath 仍可用，控制面 `airy_sys_call` 降级为传统 cap_t 引用模式 |

### 8.10 与 v1.0 的对比总结

| 维度 | v1.0 双平面架构 | v1.0.1 Capability Folding 单平面架构 |
|------|---------------|-----------------------------------|
| syscall 数量 | 12（8 seL4 IPC 原语 + 4 控制原语） | 4（1 Capability Invocation + 3 控制原语） |
| 能力校验路径 | 双路径（控制面 decode_and_invoke + 数据面 cap_check_fast） | 单路径（数据面 fastpath C-S9 内联） |
| 能力数据结构 | radix tree + per-CPU cache + IPI coherency（~300 行） | `agent_caps[1024]` 静态数组 + 1 个 atomic_t（~50 行） |
| 撤销机制 | 5ms drain + bitmap + IPI | 1 行 `atomic_inc` |
| 自举机制 | NULL Badge 硬编码特例 | `AIRY_IPC_OP_CAP_REQUEST` opcode（无硬编码） |
| 完整性校验 | 无 | CRC32 覆盖 `header[0:52) + payload` |
| fastpath 延迟 | ~160ns（不含 cap check）+ 50-100ns cap check = ~210-260ns | ~158ns（含 C-S9 Badge ~10ns） |
| 隔离基础 | 架构级（seL4 Isabelle/HOL 证明） | 数据结构级（CBMC 全函数验证） |
| 失败域 | 两个（控制面/数据面） | 一个（数据面 fastpath） |
| 设计哲学 | 双平面外挂校验 | 单平面内生校验（IPC 就是能力校验） |

---

## §9 [DSL] 降级生存层

**权威源**：[11-degraded-survival-layer.md](11-degraded-survival-layer.md)

[DSL] 是 IRON-9 v3 新增的第四层，确保 [SC] 头文件损坏/缺失时系统仍具备最小可运行子集。

### 9.1 #ifdef AIRY_SC_FALLBACK 机制

每个 [SC] 头文件底部均包含 `#ifdef AIRY_SC_FALLBACK` 降级块。当编译时定义 `AIRY_SC_FALLBACK`（或 [SC] 头文件校验失败时由构建系统自动注入），降级块生效，提供最小可用的错误码、日志级别、IPC 消息头定义。

```c
/* kernel/include/uapi/linux/airymax/error.h 底部 */
#ifdef AIRY_SC_FALLBACK
/* 降级块：仅保留 38 个 POSIX 码 */
#define AIRY_EPERM      (-1)
#define AIRY_ENOENT     (-2)
/* ... 共 38 个 POSIX errno 负值 */
#define AIRY_ECFGVERSION (-101)   /* [DSL] 唯一保留的非 POSIX 码 */
#warning "AIRY_SC_FALLBACK active: only 38 POSIX codes available"
#endif /* AIRY_SC_FALLBACK */
```

### 9.2 [DSL] ipc.h 降级块（v1.0.1 新增，H6）

v1.0.1 Capability Folding 决策为 [SC] `ipc.h` 新增 [DSL] 降级块。当 `AIRY_SC_FALLBACK` 生效时，`capability_badge` 字段存在但被忽略（值=0，跳过 C-S9 校验）：

```c
/* kernel/include/uapi/linux/airymax/ipc.h 底部 */
#ifdef AIRY_SC_FALLBACK
/* [DSL] 降级块：capability_badge 字段存在但被忽略（H6） */
/* fastpath C-S9 检测 capability_badge=0 + AIRY_IPC_F_CAP_CARRY=0 → 跳过 Badge 校验 */
/* 此时 IPC 数据面 fastpath 仍可用，但所有 Agent 均以 cap_t 引用模式运行（无 Badge 防伪造） */
#define AIRY_IPC_OP_CAP_REQUEST  0x0010  /* [DSL] 模式下此 opcode 仍可用，sec_d 直接返回 badge=0 */
#define AIRY_IPC_OP_CAP_RESPONSE 0x0011
#warning "AIRY_SC_FALLBACK active: Capability Folding Badge 校验被跳过（H6 降级模式）"
#endif /* AIRY_SC_FALLBACK */
```

### 9.3 最小可运行子集

[DSL] 降级时的最小可运行子集包括：

- **错误码**：仅保留 38 个 POSIX 码 + 1 个 `AIRY_ECFGVERSION`
- **IPC 消息头**：Layout C v4 128B 消息头（`capability_badge=0`，跳过 C-S9 Badge 校验，H6）
- **调度**：EEVDF 默认调度（不依赖 SCHED_DEADLINE / SCHED_FIFO 配置）
- **日志**：printk_safe 原生路径（不依赖 Ring Buffer）
- **能力校验（v1.0.1 新增）**：传统 cap_t 引用模式（无 Badge 防伪造，依赖 LSM 完整校验）

### 9.4 与 IRON-9 v3 的关系

[DSL] 是 IRON-9 v3 的第四层，详见 [06-iron9-shared-model.md](06-iron9-shared-model.md) §1。IRON-9 演进路径：v1（两层）→ v2（[SC]/[SS]/[IND] 三层）→ v3（新增 [DSL] 第四层）。

### 9.5 [DSL] 错误码空间（v1.0.1 新增）

[DSL] 错误码空间 `[-201, -300]`，详见 [30-interfaces/08-sc-error-contract.md §2.6](../30-interfaces/08-sc-error-contract.md)。

---

## §10 SSoT 声明

> **Airymax Unify Design 单一权威源声明**：
>
> 本文件是 **Airymax Unify Design** 的唯一权威源。A-UEF / A-ULP / A-UCS / A-ULS / A-IPC 五模块的架构定位、模块间关系、四大技术领域覆盖度、共享内存布局、[DSL] 降级策略均以本文件为唯一权威定义。
>
> **技术选型权威声明**：
> - 调度：sched_tac（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 语义映射），不使用 sched_ext
> - IPC（v1.1 升级）：Capability Folding 单平面架构——io_uring `IORING_OP_URING_CMD` + registered buffer + mmap + fastpath C-S9 Badge 内联校验，不使用 page flipping，无双平面、无独立 capability syscall
> - 安全：纯 C LSM 模块（对齐 openEuler），不使用 BPF LSM；Badge 校验是 fastpath 内联（H5）
> - 日志内存：alloc_pages(GFP_KERNEL) + mmap，不使用 DMA 一致性内存
> - [SC] 物理宿主：`kernel/include/uapi/linux/airymax/`，共 10 个头文件
>
> **v1.0.1 Capability Folding 决策权威声明**（A-IPC 第一块基石）：
> - 6 条硬约束 H1-H6 不可妥协（详见 §8.2）
> - Layout C v4 128B 消息头一经发布即冻结（magic 0x41524531 'ARE1'）
> - syscall 12→4 精确映射（详见 §8.5 + [01-syscalls.md §2.2.1](../30-interfaces/01-syscalls.md)）
> - fastpath C-S9 Badge 校验是唯一能力校验路径（~10ns 内联，3 个 READ_ONCE + 位运算）
> - sec_d 是唯一 Badge 编译者（H4），撤销通过 1 行 atomic_inc 立即生效
> - [DSL] 降级模式下 `capability_badge=0`，跳过 C-S9（H6）
>
> **冲突消解权威声明**：当其他文档与本文件冲突时，以本文件为准。

---

## §11 相关文档

- [03-microkernel-strategy.md](03-microkernel-strategy.md) —— 微内核化策略（A-ULS 决策依据）
- [06-iron9-shared-model.md](06-iron9-shared-model.md) —— IRON-9 v3 四层模型
- [11-degraded-survival-layer.md](11-degraded-survival-layer.md) —— [DSL] 降级生存层
- [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md) —— 系统调用接口（v1.0.1 Capability Folding 集成版，syscall 12→4 SSoT）
- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— IPC 协议（Layout C v4 + Capability Folding Badge 模型 SSoT）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— IPC Fastpath 状态机（C-S9 Badge 校验 + 7 态状态机 SSoT）
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— A-UEF [SC] error.h 契约（6 码空间 SSoT）
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— A-ULP [SC] log_types.h 契约
- [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— A-ULP 零拷贝 Ring Buffer 数据流
- [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) —— [SC] 共享契约层物理宿主（§2.7 ipc.h + §2.8 syscalls.h）
- [110-security/03-capability-model.md](../110-security/03-capability-model.md) —— Capability 模型（Badge 64-bit Native Word SSoT）

---

## §12 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Airymax Unify Design 5 模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）完整设计；消解 7 个架构冲突点；四大技术领域覆盖度；[DSL] 降级生存层；sched_tac + IORING_OP_URING_CMD + 纯 C LSM + alloc_pages/mmap 技术选型固化；SSoT 声明 |
| v1.1 | 2026-07-18 | **Capability Folding 集成版**：(1) §1.1 新增 C-08 双平面架构死锁冲突点；(2) §1.2 新增"IPC 就是能力校验"设计哲学；(3) §1.4 新增与 Capability Folding 的关系；(4) §8 A-IPC 全面重写为单平面架构（Capability Folding 工程定义 + 6 条硬约束 H1-H6 + Layout C v4 + syscall 12→4 + C-S9 Badge 校验 + 数据结构隔离三原则 + IRON-9 v3 四层落地 + v1.0/v1.1 对比）；(5) §9 [DSL] 新增 ipc.h 降级块（H6）+ [DSL] 错误码空间；(6) §10 SSoT 声明新增 Capability Folding 决策权威声明；(7) §11 相关文档新增 01-syscalls.md / 07-ipc-fastpath.md / 110-security/03-capability-model.md / 120-cross-project-code-sharing.md 引用；(8) 清除所有内部审查文档路径引用 |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Airymax Unify Design 总纲 | v1.0.1 | 2026-07-21
