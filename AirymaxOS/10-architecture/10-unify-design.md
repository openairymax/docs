Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax Unify Design 总纲
> **文档定位**：Airymax Unify Design 五模块统一设计思想的唯一权威总纲\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[架构设计总览](01-system-architecture.md)\
> **设计依据**：[15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §4（任务 4：确定采用 Airymax Unify Design 设计思想）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax Unify Design** 的唯一权威源（Single Source of Truth）。UEF / ULPS / UCF / USV / UIPF 五模块的架构定位、模块间关系、四大技术领域覆盖度、共享内存布局、[DSL] 降级策略均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Unify Design 模块边界与领域归属。
>
> 本文件遵循 **方案 C-Prime**（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 语义映射），IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（不使用 page flipping），安全采用 **纯 C LSM 模块**（不使用 BPF LSM），日志内存采用 **alloc_pages + mmap**（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：agentrt-linux 架构师、内核开发者、用户态守护进程开发者、跨端集成工程师
- **前置知识**：理解 [03-microkernel-strategy.md](03-microkernel-strategy.md) 微内核化策略、[06-iron9-shared-model.md](06-iron9-shared-model.md) IRON-9 v3 四层模型、[04-engineering-baseline.md](04-engineering-baseline.md) 工程基线
- **预计阅读时间**：60 分钟
- **核心概念**：UEF、ULPS、UCF、USV、UIPF、[DSL]、方案 C-Prime、四大技术领域
- **复杂度标识**：高级

---

## §1 设计背景：解决 7 个架构冲突点

### 1.1 冲突点全景

在 [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §6 文档逐一检查中，agentrt-linux 文档体系暴露出 7 个 P0 级架构冲突点。这些冲突并非孤立的技术表述错误，而是源于"缺少一个统一的设计思想来约束所有模块的边界"。Airymax Unify Design 即为消解这 7 个冲突点而提出。

| # | 冲突点 | 冲突表现 | Unify Design 消解方式 |
|---|--------|---------|---------------------|
| C-01 | sched_ext / SCHED_AGENT 表述不一致 | 50 个文件各自定义调度策略，sched_ext 在 6.6 不可用 | USV + 方案 C-Prime 统一三层调度，零内核调度器修改 |
| C-02 | page flipping 表述不一致 | 6 个文件引用已废弃的 page flipping 零拷贝路径 | UIPF 统一 IORING_OP_URING_CMD + registered buffer + mmap |
| C-03 | VFS 用户态化决策矛盾 | 2 个文件对"VFS 是否完全下沉用户态"结论相反 | USV 统一决策：内核保留路径解析，FS 实现用户态化 |
| C-04 | [SC] 头文件数量不一致 | 多处文档对共享头文件数量有 6/10 等不同说法 | UEF/ULPS/UCF 联合约束 [SC] 头文件物理宿主为 `kernel/include/airymax/`，共 10 个 |
| C-05 | 错误码双轨制 | agentrt 全仓库 `AIRY_E*` 与 `AIRY_ERR_*` 并存 | UEF 统一为 `AIRY_E*`（Error 负数空间）+ `AIRY_FAULT_*`（Fault 正数空间） |
| C-06 | 日志双轨制 | `LOG_*` 与 `AIRY_LOG_*` 并存，128B 记录格式多处定义 | ULPS 统一为 `LOG_*` 枚举 + 128B 固定记录格式，单一权威源 |
| C-07 | DMA 一致性内存误用 | 14 个文件误用 DMA 一致性内存用于日志/IPC 共享内存 | ULPS/UIPF 统一 `alloc_pages(GFP_KERNEL)` + mmap，x86_64 默认缓存一致 |

### 1.2 设计哲学

Airymax Unify Design 的设计哲学可概括为三句话：

1. **机制在内核，策略在用户态**——直接借鉴 seL4 微内核思想，内核只保留最小机制（IPC 回调、Ring Buffer、Micro-Supervisor），全部策略下沉到用户态守护进程。
2. **内核冷酷执法，用户温情裁决**——USV 的双 Supervisor 模型，内核侧立即冻结异常 Agent，用户侧根据上下文进行人性化裁决。
3. **每个技术点只允许有一个权威源**——SSoT v2 单一权威源模型，[SC] 共享契约头文件物理宿主唯一，CI 强制逐字节校验。

### 1.3 与方案 C-Prime 的关系

方案 C-Prime 是 Airymax Unify Design 在调度领域的具体落地（详见 [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §1）。Unify Design 不替代方案 C-Prime，而是将方案 C-Prime 纳入 USV 模块的调度策略实现。二者关系为：

- **方案 C-Prime**：调度机制选型（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 映射）
- **Airymax Unify Design**：整体架构思想，USV 模块内部使用方案 C-Prime 作为 Agent 8 态生命周期的调度承载

---

## §2 五模块总览：UEF / ULPS / UCF / USV / UIPF

Airymax Unify Design 由 5 大模块构成，每个模块有明确的架构定位、权威源、物理宿主与技术领域归属。

```
┌─────────────────────────────────────────────────────────────┐
│  Airymax Unify Design 5 模块统一方案                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ UEF         │  │ ULPS        │  │ UCF         │          │
│  │ 统一错误码   │  │ 统一日志    │  │ 统一配置    │          │
│  │ 与故障定义   │  │ 与打印系统  │  │ 管理        │          │
│  │ 领域：观测   │  │ 领域：观测  │  │ 领域：治理  │          │
│  │      + 韧性 │  │             │  │             │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │ USV         │  │ UIPF        │                          │
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
| **UEF** | Unified Error & Fault | 统一错误码与故障定义体系 | [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) | `kernel/include/airymax/error.h` |
| **ULPS** | Unified Log & Print System | 统一日志与打印系统 | [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) + [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) | `kernel/include/airymax/log_types.h` + `kernel/log/` |
| **UCF** | Unified Config Framework | 统一配置管理 | [20-modules/11-unified-config.md](../20-modules/11-unified-config.md) | `kernel/include/airymax/` + `kernel/config/` |
| **USV** | Unified Supervisor | 统一生命周期监管 | [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) + [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md) | `kernel/superv/` + `services/daemons/macro_superv/` |
| **UIPF** | Unified IPC Framework | 统一进程间通信体系 | [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) + [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) | `kernel/include/airymax/ipc.h` + `kernel/ipc/` |

### 2.1 模块间关系

5 模块并非平铺并列，而是存在明确的依赖与协作关系：

- **UEF 是基础**：所有模块的返回值、故障通知均使用 UEF 定义的 Error/Fault 码空间。UEF 是纯头文件模块，无 `.c` 代码。
- **ULPS 是观测主干**：USV 的故障通知、UIPF 的 IPC 异常、UCF 的配置变更均通过 ULPS 的 Ring Buffer 写入观测面。
- **UCF 是治理入口**：USV 的调度参数、UIPF 的 Ring 大小、ULPS 的日志级别均通过 UCF 的 sysctl/JSON 双向同步进行热重载。
- **USV 是韧性核心**：UEF 的 Fault 码触发 USV 的 Micro-Supervisor 冷酷执法；UIPF 的 IPC 队列冻结由 USV 触发。
- **UIPF 是通信骨干**：USV 的 Macro-Supervisor 通过 UIPF 的 `io_uring_cmd` 执行裁决；ULPS 的 eventfd 通知复用 UIPF 的 io_uring 通道。

---

## §3 四大技术领域覆盖度

Airymax Unify Design 的 5 模块覆盖"通信、治理、观测、韧性"四大技术领域，每个领域有核心模块与辅助模块。

| 技术领域 | 核心模块 | 辅助模块 | 覆盖能力 |
|---------|---------|---------|---------|
| **通信** | UIPF | UEF（错误码传递）+ ULPS（日志传递） | 进程间消息传递、零拷贝数据面、错误码跨端传递 |
| **治理** | UCF + USV | UEF（故障分类） | 配置热重载、生命周期裁决、故障分级处置 |
| **观测** | ULPS | UEF（错误码观测）+ USV（心跳观测） | 零拷贝日志、错误码可观测、Agent 状态可观测 |
| **韧性** | USV | UEF（故障定义）+ UIPF（数据面自治）+ [DSL]（降级生存） | 故障检测、冷酷执法、温情裁决、降级生存 |

### 3.1 领域交叉矩阵

```
              通信    治理    观测    韧性
   UEF        ●○      ●○      ●○      ●○
   ULPS       ●○      —       ●●      —
   UCF        —       ●●      —       ●○
   USV        ●○      ●●      ●○      ●●
   UIPF       ●●      —       —       ●○
   [DSL]      —       —       ●○      ●●

   ●● = 核心覆盖    ●○ = 辅助覆盖    — = 不覆盖
```

[DSL]（降级生存层）是 IRON-9 v3 新增的第四层（详见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md)），在 [SC] 头文件损坏/缺失时提供最小可运行子集，是韧性领域的最后防线。

---

## §4 UEF（统一错误码与故障定义体系）

**权威源**：[30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) + `kernel/include/airymax/error.h`

### 4.1 Error vs Fault 分层

UEF 的核心设计是将"可恢复的错误"与"不可恢复的故障"分离到两个不重叠的码空间，避免语义混淆：

| 维度 | Error（错误） | Fault（故障） |
|------|--------------|--------------|
| **码空间** | 负数 `[-300, -1]` | 正数 `[0x1000, 0x1FFF]` |
| **可恢复性** | 可恢复，函数返回值 | 不可恢复，触发 Fault Handler |
| **处置方式** | 调用方处理 | Micro-Supervisor 立即接管 |
| **传递路径** | 函数返回值 / IPC 响应码 | eventfd 通知 / Fault Handler |
| **示例** | `AIRY_EPERM=-1`（权限不足） | `AIRY_FAULT_CAP_FAULT=0x1001`（Capability 故障） |

### 4.2 Error 码空间分配

Error 码空间按来源分层分配，每个分层有明确的值域与语义：

| 分层 | 值域 | 来源 | 数量 |
|------|------|------|------|
| POSIX 码 | `[-1, -40]` | 对齐 Linux errno | 40 |
| IPC 码 | `[-41, -70]` | UIPF 协议层 | 30 |
| Capability 码 | `[-71, -100]` | 安全子系统 | 30 |
| [SC] 码 | `[-101, -200]` | [SC] 共享契约层 | 100 |
| [DSL] 码 | `[-201, -300]` | [DSL] 降级生存层 | 100 |

### 4.3 Fault 码空间分配

Fault 码从 `0x1000` 起步，每个 Fault 对应一个不可恢复的异常场景，由 Micro-Supervisor 直接接管：

| Fault 码 | 值 | 语义 | 触发场景 |
|---------|-----|------|---------|
| `AIRY_FAULT_CAP_FAULT` | `0x1001` | Capability 故障 | 检测到伪造/越权 Capability |
| `AIRY_FAULT_VM_FAULT` | `0x1002` | 虚拟内存故障 | 共享页映射损坏 |
| `AIRY_FAULT_IPC_FAULT` | `0x1003` | IPC 故障 | Ring Buffer 元数据损坏 |
| `AIRY_FAULT_TIMEOUT` | `0x1004` | 超时故障 | Agent 心跳超时且未响应冻结 |
| `AIRY_FAULT_ABNORMAL_CAP` | `0x1005` | 异常 Capability | Capability 树完整性校验失败 |

### 4.4 [SC] error.h 契约

`kernel/include/airymax/error.h` 是 UEF 的 [SC] 共享契约头文件，agentrt 与 agentrt-linux 物理共享同一份文件，逐字节相同。CI 通过 `sc-dual-ci.yml` 验证两端 `error.h` 逐字节一致。详细契约定义见 [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)。

---

## §5 ULPS（统一日志与打印系统）

**权威源**：[30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) + [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)

### 5.1 内存方案：alloc_pages + mmap

ULPS 的日志 Ring Buffer 内存采用 `alloc_pages(GFP_KERNEL)` 分配 + `mmap` 映射到用户态。**明确不使用 DMA 一致性内存**——在 x86_64 架构下，CPU 默认缓存一致（cache-coherent），普通页内存即可满足内核生产者与用户态消费者之间的可见性要求。DMA 一致性内存仅在 DMA 设备直接写内存时才需要，日志场景无此需求。

```c
/* kernel/log/airy_log_ring.c —— 内存分配 */
struct page *pages;
pages = alloc_pages(GFP_KERNEL | __GFP_ZERO, order);
/* mmap 到用户态 Logger Daemon */
vm_flags_set(vma, VM_SHARED | VM_DONTEXPAND);
remap_pfn_range(vma, vma->vm_start, page_to_pfn(pages), size, vma->vm_page_prot);
```

### 5.2 128B 固定记录格式

ULPS 采用 128 字节固定长度日志记录，对齐 cache line，确保单次 memcpy 完成写入。记录格式详见 [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) §2：

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

ULPS 严格遵循"Fastpath 禁止格式化"原则。内核生产者仅执行 `reserve → memcpy 128B raw binary → commit` 三步，不进行任何字符串格式化。格式化、过滤、落盘全部由用户态 Logger Daemon 完成：

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

当系统进入 Panic 状态时，Ring Buffer 可能已损坏或 Logger Daemon 已死亡。ULPS 回退到 Linux 6.6 原生 `printk_safe` 路径（NMI-safe buffer），确保 Panic 日志可靠落盘。详见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md) §3。

---

## §6 UCF（统一配置管理）

**权威源**：[20-modules/11-unified-config.md](../20-modules/11-unified-config.md)

### 6.1 [SC] 共享配置 + [SS] 语义同源配置

UCF 将配置分为两类，对应 IRON-9 v3 的两个层级：

- **[SC] 共享配置**：仅限二进制布局相关的配置项，如 IPC 消息头大小、Ring Buffer 大小、魔数。这些配置在 `kernel/include/airymax/` 头文件中以 `#define` 形式定义，两端逐字节相同。
- **[SS] 语义同源配置**：内核侧 Kconfig/sysctl 与用户态 JSON/TOML 语义映射。语义一致，签名独立演进。例如内核 `kernel.sched_rt_runtime_us` 与用户态 JSON `sched.rt_runtime_us` 语义同源。

### 6.2 RCU 热重载

UCF 支持 RCU 指针原子切换的热重载机制。sysctl 写入或 JSON 变更触发 `config_reload` 事件，内核通过 `rcu_assign_pointer()` 原子切换配置指针，读者通过 `rcu_dereference()` 获取最新配置，旧配置在宽限期后释放。

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

UCF 引入 `AIRY_CONFIG_VERSION` 字段，配置加载时校验版本号，不匹配则拒绝加载并返回 `AIRY_ECFGVERSION`（-101，[SC] 码空间首个值）。

---

## §7 USV（统一生命周期监管）

**权威源**：[20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) + [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md)

### 7.1 Micro-Supervisor（内核冷酷执法）

Micro-Supervisor 是内核态的纯 C LSM 模块（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式），负责检测非法 Capability 并立即执法：

1. **检测**：纯 C LSM 钩子检测到伪造/越权 Capability
2. **冻结**：立即设置 `ring->frozen = true`，IPC fastpath 通过 `unlikely(ring->frozen)` 零开销检查跳过
3. **返回**：返回 `AIRY_FAULT_ABNORMAL_CAP`（0x1005）
4. **通知**：eventfd 通知 Macro-Supervisor

Micro-Supervisor 不做任何"人性化"决策，仅执行冷酷的机制级执法——这是借鉴 seL4 `handleFault()` 的设计：内核只负责检测与通知，裁决交给用户态。

### 7.2 Macro-Supervisor（用户温情裁决）

Macro-Supervisor 是用户态守护进程（`services/daemons/macro_superv/`），接收 Micro-Supervisor 的故障通知后进行人性化裁决：

| 裁决选项 | 适用场景 | 执行方式 |
|---------|---------|---------|
| 警告 | 首次轻微违规 | 记录日志，不中断 Agent |
| 降级 | Capability 边界模糊 | 降低优先级、缩减预算 |
| 暂停 | 多次违规 | `kill(SIGSTOP)`，进入 STOPPED 态 |
| 终止 | 严重违规 | `kill(SIGKILL)`，进入 DEAD 态 |

裁决通过 `io_uring_cmd` 执行，避免新增 syscall。Macro-Supervisor 单点故障时，内核 watchdog 直接重启。

### 7.3 Agent 8 态生命周期

USV 定义 Agent 8 态生命周期，与 Linux 进程状态天然映射（方案 C-Prime 的核心成果之一）：

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

## §8 UIPF（统一进程间通信体系）

**权威源**：[30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) + [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)

### 8.1 IORING_OP_URING_CMD 语义映射

UIPF 采用 `IORING_OP_URING_CMD` 作为 IPC 语义映射载体，**明确不使用 page flipping**。page flipping 在 Linux 6.6 主线已不推荐，且语义上属于"内核翻转页表项"，与微内核化原则冲突。UIPF 改为：

- **registered buffer**：用户态预注册缓冲区，内核通过 `io_uring_cmd` 回调直接引用，零拷贝
- **mmap 共享页**：Ring Buffer 元数据通过 mmap 共享，内核与用户态读写同一物理页

```c
/* kernel/ipc/airy_uring_cmd.c —— io_uring_cmd 回调 */
static int airy_uring_cmd(struct io_uring_cmd *ioucmd, unsigned int issue_flags)
{
    struct airy_ipc_cmd *cmd = (struct airy_ipc_cmd *)ioucmd->cmd;
    switch (cmd->op) {
    case AIRY_IPC_OP_SEND:    return airy_ipc_send(ioucmd, cmd, issue_flags);
    case AIRY_IPC_OP_RECV:    return airy_ipc_recv(ioucmd, cmd, issue_flags);
    case AIRY_IPC_OP_FREEZE:  return airy_ipc_freeze(ioucmd, cmd, issue_flags);
    default:                  return -EOPNOTSUPP;
    }
}
```

### 8.2 Capability 分层校验

UIPF 的 Capability 校验采用分层设计，平衡安全与性能：

- **第一层（fastpath）**：纯 C LSM 轻量标记，通过 `static_key` 跳过——99%+ 的正常请求在此层直接放行，零开销
- **第二层（slowpath）**：内核模块 radix tree 完整查找，仅在第一层标记可疑时触发

分层校验确保 fastpath 性能（~160ns）不受 Capability 完整校验影响。

### 8.3 数据面自治三原则

UIPF 遵循数据面自治三原则，确保控制面故障不影响数据面正常运作：

1. **Ring 生命周期解耦**：Ring Buffer 的生命周期独立于控制面连接，控制面重启不丢失未消费消息
2. **离线缓存校验**：Capability 校验结果缓存在内核 radix tree，控制面离线时仍可基于缓存校验
3. **Reconciliation**：控制面恢复后，数据面将离线期间的状态变更同步给控制面，达成最终一致

### 8.4 IPC 队列冻结

当 Micro-Supervisor 检测到异常时，通过 `ring->frozen = true` 冻结 IPC 队列。fastpath 通过 `unlikely(ring->frozen)` 零开销检查——正常路径不触发冻结检查的分支预测失败，仅异常路径才进入冻结处理。

---

## §9 [DSL] 降级生存层

**权威源**：[11-degraded-survival-layer.md](11-degraded-survival-layer.md)

[DSL] 是 IRON-9 v3 新增的第四层，确保 [SC] 头文件损坏/缺失时系统仍具备最小可运行子集。

### 9.1 #ifdef AIRY_SC_FALLBACK 机制

每个 [SC] 头文件底部均包含 `#ifdef AIRY_SC_FALLBACK` 降级块。当编译时定义 `AIRY_SC_FALLBACK`（或 [SC] 头文件校验失败时由构建系统自动注入），降级块生效，提供最小可用的错误码、日志级别、IPC 消息头定义。

```c
/* kernel/include/airymax/error.h 底部 */
#ifdef AIRY_SC_FALLBACK
/* 降级块：仅保留 38 个 POSIX 码 */
#define AIRY_EPERM      (-1)
#define AIRY_ENOENT     (-2)
/* ... 共 38 个 POSIX errno 负值 */
#define AIRY_ECFGVERSION (-101)   /* [DSL] 唯一保留的非 POSIX 码 */
#warning "AIRY_SC_FALLBACK active: only 38 POSIX codes available"
#endif /* AIRY_SC_FALLBACK */
```

### 9.2 最小可运行子集

[DSL] 降级时的最小可运行子集包括：

- **错误码**：仅保留 38 个 POSIX 码 + 1 个 `AIRY_ECFGVERSION`
- **IPC 消息头**：最简 128B 消息头（仅 magic + opcode + payload_len）
- **调度**：EEVDF 默认调度（不依赖 SCHED_DEADLINE / SCHED_FIFO 配置）
- **日志**：printk_safe 原生路径（不依赖 Ring Buffer）

### 9.3 与 IRON-9 v3 的关系

[DSL] 是 IRON-9 v3 的第四层，详见 [06-iron9-shared-model.md](06-iron9-shared-model.md) §1。IRON-9 演进路径：v1（两层）→ v2（[SC]/[SS]/[IND] 三层）→ v3（新增 [DSL] 第四层）。

---

## §10 SSoT 声明

> **Airymax Unify Design 单一权威源声明**：
>
> 本文件是 **Airymax Unify Design** 的唯一权威源。UEF / ULPS / UCF / USV / UIPF 五模块的架构定位、模块间关系、四大技术领域覆盖度、共享内存布局、[DSL] 降级策略均以本文件为唯一权威定义。
>
> **技术选型权威声明**：
> - 调度：方案 C-Prime（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 语义映射），不使用 sched_ext
> - IPC：IORING_OP_URING_CMD + registered buffer + mmap，不使用 page flipping
> - 安全：纯 C LSM 模块（对齐 openEuler），不使用 BPF LSM
> - 日志内存：alloc_pages(GFP_KERNEL) + mmap，不使用 DMA 一致性内存
> - [SC] 物理宿主：`kernel/include/airymax/`，共 10 个头文件
>
> **冲突消解权威声明**：当其他文档与本文件冲突时，以本文件为准。本文件遵循 [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §4 的设计决策。

---

## §11 相关文档

- [03-microkernel-strategy.md](03-microkernel-strategy.md) —— 微内核化策略（USV 决策依据）
- [06-iron9-shared-model.md](06-iron9-shared-model.md) —— IRON-9 v3 四层模型
- [11-degraded-survival-layer.md](11-degraded-survival-layer.md) —— [DSL] 降级生存层
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— UEF [SC] error.h 契约
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— ULPS [SC] log_types.h 契约
- [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— ULPS 零拷贝 Ring Buffer 数据流
- [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §4 —— 设计依据

---

## §12 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Airymax Unify Design 5 模块（UEF/ULPS/UCF/USV/UIPF）完整设计；消解 7 个架构冲突点；四大技术领域覆盖度；[DSL] 降级生存层；方案 C-Prime + IORING_OP_URING_CMD + 纯 C LSM + alloc_pages/mmap 技术选型固化；SSoT 声明 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Airymax Unify Design 总纲 | v1.0 | 2026-07-17
