Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] sched.h 扩展契约 — sched_tac
> **文档定位**：A-ULS（统一生命周期管理）模块的调度扩展 [SC] 共享契约，定义 Agent 8 态生命周期与sched_tac 调度接口的唯一权威契约\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-26\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7\
> **设计依据**：综合修正方案 §1（sched_tac）+ §4.2.4（A-ULS 设计）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **A-ULS 模块调度扩展 [SC] 契约** 的唯一权威源。Agent 8 态生命周期枚举、3 态降级映射、sched_tac 调度接口（`sched_setattr` / `sched_setscheduler`）、seL4 MCS 语义映射（`scBudget` ↔ `sched_runtime`、`scPeriod` ↔ `sched_deadline`）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Agent 状态机与调度参数契约。
>
> 技术选型声明：本文件遵循 **sched_tac**（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 语义映射，**不使用 sched_ext**）。IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（不使用 page flipping）。安全采用 **纯 C LSM 模块**（不使用 BPF LSM）。日志内存采用 **alloc_pages + mmap**（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

---

## 文档信息卡

- **目标读者**：agentrt-linux 内核调度开发者、A-ULS 模块开发者、Macro-Supervisor 开发者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) §7 A-ULS 模块、[04-scheduling-flow.md](../40-dataflows/04-scheduling-flow.md) 调度数据流、Linux 6.6 `sched_setattr(2)` 与 seL4 MCS（Mixed-Criticality Scheduling）
- **预计阅读时间**：25 分钟
- **核心概念**：sched_tac、Agent 8 态、3 态降级、SCHED_DEADLINE、SCHED_FIFO、EEVDF、seL4 MCS、sched_context_t
- **复杂度标识**：高级

---

## §1 契约概述：A-ULS 模块的调度扩展 [SC] 共享契约

### 1.1 契约定位

`kernel/include/uapi/linux/airymax/sched.h` 是 A-ULS 模块的 [SC] 共享契约头文件，定义 Agent 生命周期状态机与sched_tac 调度参数的统一契约。该头文件由 agentrt（用户态 SDK）与 agentrt-linux（内核态实现）双端逐字节共享，CI 通过 `sc-dual-ci.yml` 验证两端一致性。

### 1.2 契约边界

本 [SC] 契约仅定义以下三部分内容，**不涉及调度策略实现**：

| 内容 | 类型 | 边界约束 |
|------|------|---------|
| Agent 状态枚举 | `enum airy_agent_state` | 仅定义状态码，不定义状态机迁移逻辑 |
| 调度参数结构 | `struct airy_sched_attr` | 仅定义二进制布局，不定义校验逻辑 |
| seL4 MCS 映射宏 | `AIRY_MCS_MAP_*` | 仅定义字段映射关系，不定义内核实现 |

### 1.3 与sched_tac 的关系

sched_tac 是本 [SC] 契约的调度机制选型依据（详见 综合修正方案 §1）。sched_tac 的核心决策为：

1. **不使用 sched_ext**：OLK 6.6 虽 backport 了 sched_ext（`kernel/sched/ext.c` 215KB，`SCHED_EXT = 7`，`CONFIG_SCHED_CLASS_EXT`），但 sched_ext 依赖 `BPF_SYSCALL && BPF_JIT && DEBUG_INFO_BTF`（`kernel/Kconfig.preempt:138`），与 H5 纯 C LSM 原则冲突；x86 默认禁用 sched_ext（`include/linux/sched.h:1663-1667` 注释明确"x86 disables SCX by default to avoid KABI changes"）；BPF verifier 在调度路径的语义不确定性影响形式化验证可行性；sched_ext 的 BPF struct_ops 模型与 agentrt-linux 纯 C 体系不一致。注：vanilla Linux 6.6 主线不含 sched_ext（6.12 才合入），但 OLK 6.6 已 backport 此特性。
2. **复用主线调度器**：直接使用 `SCHED_DEADLINE` / `SCHED_FIFO` / `SCHED_NORMAL`(EEVDF) 三类已有调度策略
3. **借鉴 seL4 MCS 语义**：将 seL4 的 `sched_context_t` 预算/周期模型映射到 Linux `sched_attr` 字段，获得 MCS 语义但不修改内核调度器

---

## §2 Agent 8 态生命周期：INACTIVE/SPAWNING/READY/RUNNING/BLOCKED/STOPPING/STOPPED/DEAD

### 2.1 8 态枚举定义

```c
/* kernel/include/uapi/linux/airymax/sched.h —— Agent 8 态生命周期枚举 */
#ifndef _UAPI_AIRYMAX_SCHED_H
#define _UAPI_AIRYMAX_SCHED_H

#include <linux/airymax/uapi_compat.h>
#include <linux/airymax/lsm_types.h>

/**
 * enum airy_agent_state - Agent 8 态生命周期
 *
 * sched_tac 核心成果：8 态与 Linux 进程状态天然映射，
 * 无需新增内核调度器状态，仅复用 SCHED_DEADLINE/SCHED_FIFO/EEVDF。
 *
 * 状态迁移由 Macro-Supervisor 驱动，Micro-Supervisor 仅在
 * 检测到异常时触发 RUNNING -> STOPPING 的强制迁移。
 */
enum airy_agent_state {
    AIRY_AGENT_INACTIVE = 0,   /* 进程不存在，等待 fork */
    AIRY_AGENT_SPAWNING = 1,   /* fork/exec 中，未就绪 */
    AIRY_AGENT_READY    = 2,   /* TASK_RUNNING，在运行队列等待 */
    AIRY_AGENT_RUNNING  = 3,   /* TASK_RUNNING，正在 CPU 执行 */
    AIRY_AGENT_BLOCKED  = 4,   /* TASK_INTERRUPTIBLE，等待 IPC/IO */
    AIRY_AGENT_STOPPING = 5,   /* SIGSTOP 发送中，正在冻结 IPC */
    AIRY_AGENT_STOPPED  = 6,   /* TASK_STOPPED，已冻结，待裁决 */
    AIRY_AGENT_DEAD     = 7,   /* EXIT_ZOMBIE，等待 waitpid 回收 */
    AIRY_AGENT_STATE_MAX
};
```

### 2.2 状态迁移图

```
                  fork()                 sched_setattr()
  INACTIVE ──────────────▶ SPAWNING ──────────────▶ READY
                                                    │
                                              挑选执行│
                                                    ▼
                       ┌──────── RUN 调度 ────── RUNNING
                       │                          │
                  IPC/IO 等待                    异常
                       │                          │ Micro-Supervisor
                       ▼                          ▼ 冻结
                    BLOCKED ──── 唤醒 ──▶     STOPPING
                                                    │ SIGSTOP 生效
                                                    ▼
                  ┌───── 裁决终止 ────────────── STOPPED
                  │                                │
                  │                          裁决恢复│ SIGCONT
                  │                                ▼
                  │                              READY
                  ▼
                 DEAD ──── waitpid() ──▶      INACTIVE
```

### 2.3 与 Linux 进程状态的映射

| Agent 态 | Linux 进程状态 | task_struct->state | 调度策略 | 触发者 |
|---------|--------------|-------------------|---------|--------|
| INACTIVE | 进程不存在 | — | — | Macro-Supervisor `fork()` |
| SPAWNING | fork/exec 中 | TASK_RUNNING(过渡) | SCHED_NORMAL | exec 完成 → READY |
| READY | TASK_RUNNING | `TASK_RUNNING` | SCHED_DEADLINE | Macro-Supervisor `sched_setattr()` |
| RUNNING | TASK_RUNNING | `TASK_RUNNING` | SCHED_DEADLINE | 调度器挑选 |
| BLOCKED | TASK_INTERRUPTIBLE | `TASK_INTERRUPTIBLE` | — | IPC/IO 等待 |
| STOPPING | SIGSTOP 发送中 | 过渡态 | — | Micro-Supervisor `kill(SIGSTOP)` |
| STOPPED | TASK_STOPPED | `TASK_STOPPED` | — | SIGSTOP 生效 |
| DEAD | EXIT_ZOMBIE | `EXIT_ZOMBIE` | — | `exit()` / `kill(SIGKILL)` |

---

## §3 3 态降级：RUNNING/STOPPED/DEAD

### 3.1 降级场景

当 [DSL] 降级生存层激活（`AIRY_SC_FALLBACK` 定义）或 Macro-Supervisor 故障时，A-ULS 将 8 态降级为 3 态，仅保留最小可运行子集：

```c
/* kernel/include/uapi/linux/airymax/sched.h —— [DSL] 降级块 */
#ifdef AIRY_SC_FALLBACK
    /* All sched_tac policies collapse to EEVDF default. */
    #define AIRY_DSL_SCHED_POLICY_DEADLINE   AIRY_SCHED_POLICY_EEVDF
    #define AIRY_DSL_SCHED_POLICY_FIFO       AIRY_SCHED_POLICY_EEVDF
    #define AIRY_DSL_SCHED_POLICY_EEVDF      AIRY_SCHED_POLICY_EEVDF
    #define AIRY_DSL_SCHED_POLICY_BESTEFFORT AIRY_SCHED_POLICY_EEVDF
    #define AIRY_DSL_SCHED_POLICIES          1  /* Only EEVDF retained */

    /* vtime decay collapses to identity (no weighted decay in fallback). */
    #define AIRY_DSL_VTIME_DECAY(vtime, weight)  (vtime)

    #warning "AIRY_SC_FALLBACK active: sched.h degraded to EEVDF default only, sched_tac three-tier unavailable"
#endif /* AIRY_SC_FALLBACK */
```

### 3.2 3 态 vs 8 态对比

| 维度 | 8 态（正常） | 3 态（降级） |
|------|------------|------------|
| 状态数 | 8 | 3 |
| 调度策略 | SCHED_DEADLINE/SCHED_FIFO/EEVDF | 仅 EEVDF 默认 |
| 调度参数 | `sched_runtime`/`sched_deadline`/`sched_period` | 系统默认 |
| 状态迁移 | Macro-Supervisor 驱动 | 内核原生 fork/exit |
| MCS 语义 | seL4 scBudget/scPeriod 映射 | 不映射 |

### 3.3 降级触发条件

| 触发条件 | 处理 |
|---------|------|
| `AIRY_SC_FALLBACK` 编译宏定义 | 3 态降级，EEVDF 默认 |
| Macro-Supervisor 心跳超时 | 3 态降级，内核 watchdog 接管 |
| `sched.h` 版本校验失败 | 3 态降级，告警 |

---

## §4 sched_tac 接口：sched_setattr / sched_setscheduler

### 4.1 任务描述符与调度策略标识符

```c
/* kernel/include/uapi/linux/airymax/sched.h —— 任务描述符 + 调度策略 */
#define AIRY_TASK_MAGIC         0x41475453u /* 'AGTS' */

/* ─── Task Priority Range ────────────────────────────────────────────── */
#define AIRY_PRIO_MIN           0
#define AIRY_PRIO_MAX           139

/* ─── Default Scheduling Parameters ──────────────────────────────────── */
#define AIRY_SLICE_DFL          20      /* Default timeslice (ms) */
#define AIRY_WEIGHT_MIN         1
#define AIRY_WEIGHT_MAX         10000

/* ─── vtime: Q16.16 fixed-point for user-space vtime approximation ───── */
typedef __s32 airy_vtime_t;
#define AIRY_VTIME_ONE          (1 << 16)  /* 1.0 in Q16.16 */

static inline airy_vtime_t airy_vtime_decay(airy_vtime_t vtime, __u32 weight)
{
    /*
     * User-space vtime approximation (NOT the kernel EEVDF internal
     * algorithm). The kernel's EEVDF uses vruntime += delta_exec *
     * NICE_0_LOAD / load_weight with actual execution time delta_exec;
     * this UAPI helper uses the default slice constant AIRY_SLICE_DFL
     * for precomputed table consumers that need a static estimate.
     */
    return vtime + (AIRY_SLICE_DFL * AIRY_VTIME_ONE) /
           (weight ? weight : 1);
}

/* ─── Task Descriptor（64 字节，自然 8 字节对齐） ───────────────────── */
/*
 * Field ordering: 64-bit fields are grouped after the header word to
 * guarantee natural 8-byte alignment without padding. 32-bit fields
 * occupy the tail. Total size = 64 bytes (verified by _Static_assert).
 */
struct airy_task_desc {
    __u32       magic;          /* offset 0:  AIRY_TASK_MAGIC */
    __u16       prio;           /* offset 4:  priority [0,139] */
    __u16       _pad;           /* offset 6:  alignment padding */
    __u64       runtime_ns;     /* offset 8:  runtime budget (ns) */
    __u64       deadline_ns;    /* offset 16: deadline (ns) */
    __u64       period_ns;      /* offset 24: period (ns) */
    airy_vtime_t vtime;         /* offset 32: virtual time Q16.16 */
    __u32       agent_id;       /* offset 36: agent identifier [0,1023] */
    __u32       sched_policy;   /* offset 40: SCHED_DEADLINE/FIFO/OTHER */
    __u32       weight;         /* offset 44: EEVDF weight */
    __u32       state;          /* offset 48: agent lifecycle state */
    __u8        reserved[12];   /* offset 52: reserved */
} __attribute__((aligned(64)));

_Static_assert(sizeof(struct airy_task_desc) == 64,
               "airy_task_desc must be exactly 64 bytes");

/* ─── sched_tac Policy Identifiers（对齐 SSoT sched.h L97-100） ─────── */
#define AIRY_SCHED_POLICY_DEADLINE   1   /* 实时 Agent：SCHED_DEADLINE */
#define AIRY_SCHED_POLICY_FIFO       2   /* 中断/IPC Agent：SCHED_FIFO */
#define AIRY_SCHED_POLICY_EEVDF      3   /* 普通 Agent：SCHED_NORMAL(EEVDF) */
#define AIRY_SCHED_POLICY_BESTEFFORT 4   /* 后台 Agent：SCHED_IDLE/BESTEFFORT */
```

> **SSoT 对齐说明（v3.5 修复 P0-I4/I5）**：
>
> 1. **`struct airy_sched_attr` 已废弃**——SSoT `sched.h` 仅定义 `struct airy_task_desc`（64 字节），不存在的 `airy_sched_attr` 已从本契约移除。sched_tac 调度参数通过 `struct airy_task_desc` 的 `runtime_ns`/`deadline_ns`/`period_ns`/`prio`/`sched_policy` 字段承载，由 Macro-Supervisor 转换为 Linux `struct sched_attr` 后通过 `sched_setattr()` 注入。
> 2. **策略标识符前缀统一为 `AIRY_SCHED_POLICY_*`**（非 `AIRY_SCHED_*`），值域 1-4（非 0-2），对齐 SSoT `sched.h:97-100`。

### 4.2 sched_setattr 设置 SCHED_DEADLINE

实时 Agent（如认知推理 Agent）使用 `sched_setattr()` 设置 SCHED_DEADLINE 参数。Macro-Supervisor 从 `struct airy_task_desc` 提取 `runtime_ns`/`deadline_ns`/`period_ns` 字段，转换为 Linux `struct sched_attr` 后注入：

```c
/* services/daemons/macro_d/sched.c —— 设置 SCHED_DEADLINE */
static int airy_agent_set_deadline(pid_t pid, const struct airy_task_desc *td)
{
    struct sched_attr sa = {
        .size           = sizeof(sa),
        .sched_policy   = SCHED_DEADLINE,
        .sched_flags    = 0,
        .sched_runtime  = td->runtime_ns,    /* CPU 预算（对齐 seL4 scBudget） */
        .sched_deadline = td->deadline_ns,   /* 截止时间（对齐 seL4 scRefill） */
        .sched_period   = td->period_ns,     /* 周期（对齐 seL4 scPeriod） */
    };
    /* sched_setattr：Linux 6.6 原生接口，零内核修改 */
    return syscall(__NR_sched_setattr, pid, &sa, 0);
}
```

### 4.3 sched_setscheduler 设置 SCHED_FIFO

中断/IPC Agent 使用 `sched_setscheduler()` 设置 SCHED_FIFO。Macro-Supervisor 从 `struct airy_task_desc.prio` 提取静态优先级（1-99）：

```c
/* services/daemons/macro_d/sched.c —— 设置 SCHED_FIFO */
static int airy_agent_set_fifo(pid_t pid, const struct airy_task_desc *td)
{
    struct sched_param sp = {
        .sched_priority = td->prio,  /* 1-99（对齐 SCHED_FIFO 优先级区间） */
    };
    /* sched_setscheduler：Linux 6.6 原生接口 */
    return sched_setscheduler(pid, SCHED_FIFO, &sp);
}
```

### 4.4 三层调度策略选择

| Agent 类型 | 策略标识符（SSoT） | Linux 调度策略 | 接口 | 典型参数 |
|-----------|------------------|---------------|------|---------|
| 实时推理 Agent | `AIRY_SCHED_POLICY_DEADLINE` (=1) | SCHED_DEADLINE | `sched_setattr()` | runtime=10ms, deadline=50ms, period=50ms |
| IPC/中断 Agent | `AIRY_SCHED_POLICY_FIFO` (=2) | SCHED_FIFO | `sched_setscheduler()` | priority=50 |
| 普通 Agent | `AIRY_SCHED_POLICY_EEVDF` (=3) | SCHED_NORMAL(EEVDF) | `sched_setscheduler()` | nice=0 |
| 后台 Daemon | `AIRY_SCHED_POLICY_BESTEFFORT` (=4) | SCHED_IDLE | `sched_setscheduler()` | — |

---

## §5 seL4 MCS 映射：sched_context_t 字段映射

### 5.1 MCS 语义映射

sched_tac 借鉴 seL4 MCS（Mixed-Criticality Scheduling）模型，将 seL4 的 `sched_context_t` 字段映射到 Linux `sched_attr` 字段，**不修改内核调度器**即可获得 MCS 语义：

| seL4 字段 | seL4 类型 | Linux 映射字段 | Linux 类型 | 语义 |
|-----------|----------|--------------|-----------|------|
| `sched_context_t.scBudget` | `time_t` | `sched_attr.sched_runtime` | `__u64` | CPU 预算（纳秒） |
| `sched_context_t.scPeriod` | `time_t` | `sched_attr.sched_period` | `__u64` | 调度周期（纳秒） |
| `sched_context_t.scCore` | `word_t` | `sched_attr.sched_attr_cpu_set` | `cpu_set_t` | CPU 亲和性 |
| `sched_context_t.scRefill` | `word_t` | `sched_attr.sched_deadline` | `__u64` | 截止时间（周期内） |

### 5.2 映射宏定义

> **SSoT 对齐说明（v3.5 修复）**：以下 `AIRY_MCS_MAP_*` 与 `AIRY_MCS_CHECK` 宏为**文档说明性示例**，**不属于 SSoT `sched.h` 物理头文件定义**。SSoT `sched.h` 不包含 seL4 MCS 映射宏——映射关系通过 `struct airy_task_desc` 字段名（`runtime_ns`/`deadline_ns`/`period_ns`）与 Linux `struct sched_attr` 字段名（`sched_runtime`/`sched_deadline`/`sched_period`）的命名同构性体现，由 Macro-Supervisor 在用户态完成转换（见 §4.2/§4.3）。

```c
/* 文档说明性示例（非 SSoT 定义，不写入 sched.h 物理头文件） */
/**
 * seL4 MCS 字段映射宏（文档级）
 * 将 seL4 sched_context_t 语义映射到 struct airy_task_desc 字段
 * 映射关系仅用于文档与校验，不引入 seL4 依赖，不写入 SSoT 头文件
 */
#define AIRY_MCS_MAP_BUDGET(sc_budget)    (sc_budget)        /* scBudget → runtime_ns */
#define AIRY_MCS_MAP_PERIOD(sc_period)    (sc_period)        /* scPeriod → period_ns */
#define AIRY_MCS_MAP_DEADLINE(sc_refill)  (sc_refill)        /* scRefill → deadline_ns */
#define AIRY_MCS_MAP_CORE(sc_core)        ((unsigned long)(sc_core)) /* scCore → cpu_set */

/* MCS 校验（文档级）：budget <= deadline <= period（对齐 seL4 MCS 约束） */
#define AIRY_MCS_CHECK(td) \
    (((td)->runtime_ns <= (td)->deadline_ns) && \
     ((td)->deadline_ns <= (td)->period_ns))
```

**seL4 MCS Refill 循环缓冲**（`src/object/schedcontext.c`）：
seL4 MCS 使用 sporadic server + refill 循环缓冲（`refill_t` 结构 + `refillNew()`/`refillUpdate()`），比 Linux SCHED_DEADLINE 的 CBS（Constant Bandwidth Server）算法更精确。agentrt-linux 在 Linux CBS 之上模拟 MCS 语义：
- `sched_attr.sched_runtime` → `scBudget`（CPU 预算）
- `sched_attr.sched_period` → `scPeriod`（调度周期）
- `sched_attr.sched_deadline` → `scRefill`（截止时间）
- CBS 的 runtime replenishment 机制模拟 MCS refill 循环缓冲

**差异说明**：CBS 使用全局 replenishment timer（`sched_dl_entity.dl_timer`），MCS 使用 per-context refill 循环缓冲。在 1.0.1 中可考虑基于 Linux 6.6 `sched_attr` 的 deadline 字段实现更精确的 MCS 模拟。

### 5.3 MCS 语义保持

映射后，Linux SCHED_DEADLINE 调度器在内核中保证以下 MCS 语义（由 CFS/DEADLINE 调度器实现，无需修改）：

1. **预算保证**：每个周期内 Agent 至少获得 `sched_runtime` 纳秒 CPU 时间
2. **截止时间保证**：CPU 预算在 `sched_deadline` 纳秒内交付
3. **周期重置**：每 `sched_period` 纳秒预算重置
4. **混合关键性**：高关键性 Agent 可在过载时抢占低关键性 Agent（通过优先级配置）

---

### OLK QoS 调度借鉴

OLK 6.6 提供了一系列 QoS 调度增强（`kernel/sched/grid/`），agentrt-linux 的 sched_tac 可借鉴：

| OLK QoS 特性 | 借鉴点 | agentrt-linux 应用 |
|-------------|--------|-------------------|
| `QOS_SCHED_SMART_GRID` | 8 级 affinity domain（`AD_LEVEL_MAX=8`），自动 cpumask 优化 | Agent NUMA 感知调度 |
| `QOS_SCHED_SMT_EXPELLER` | SMT 兄弟核在线任务驱逐离线任务 | Agent 优先级抢占 |
| `QOS_SCHED_MULTILEVEL` | 扩展 QoS level 到 [-2,2] | Agent 多级 QoS 分类 |
| `SCHED_SOFT_QUOTA` | CFS 带宽灵活使用（idle 路径解 throttle） | Agent 弹性调度 |
| `SCHED_SOFT_DOMAIN` | CPU 软域调度（缓存局部性优化） | Agent 缓存亲和性 |

---

## §6 物理宿主与版本管理

### 6.1 物理宿主

| 项目 | 路径 | 说明 |
|------|------|------|
| [SC] 头文件 | `kernel/include/uapi/linux/airymax/sched.h` | 双端逐字节共享 |
| 内核实现 | `kernel/kernel/superv/airy_sched.c` | 调度策略实现 |
| 用户态实现 | `services/daemons/macro_d/sched.c` | 调度参数注入 |

### 6.2 版本号

> **SSoT 对齐说明（v3.5 修复）**：SSoT `sched.h` **不定义** `AIRY_SCHED_VERSION` 与 `AIRY_SCHED_VERSION_CHECK` 宏。版本一致性通过 [SC] 共享契约层双端 CI（`sc-dual-ci.yml`）逐字节校验保证，无需在头文件内嵌版本号宏。旧文档中引用的 `AIRY_SCHED_VERSION`/`AIRY_SCHED_VERSION_CHECK` 为虚构定义，已移除。

```c
/* SSoT sched.h 不含版本号宏 —— 一致性由 sc-dual-ci.yml 逐字节校验保证 */
/* 旧文档引用的 AIRY_SCHED_VERSION / AIRY_SCHED_VERSION_CHECK 已废弃 */
```

### 6.3 CI 双端校验

`sc-dual-ci.yml` 对 `sched.h` 执行以下校验：

1. **逐字节对比**：agentrt 与 agentrt-linux 的 `sched.h` 必须逐字节相同
2. **枚举值校验**：8 态枚举值必须在 `[0, 7]` 范围内且不重复（含 `AIRY_AGENT_STATE_MAX` 上界哨兵）
3. **结构体偏移校验**：`struct airy_task_desc`（64 字节）字段偏移在两端一致，`_Static_assert` 保证 `sizeof == 64`
4. **策略标识符校验**：`AIRY_SCHED_POLICY_DEADLINE=1` / `_FIFO=2` / `_EEVDF=3` / `_BESTEFFORT=4` 双端一致
5. **[DSL] 块校验**：`AIRY_DSL_SCHED_POLICY_*` 与 `AIRY_DSL_VTIME_DECAY` 宏双端一致

---

## §7 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §7 —— A-ULS 模块总纲
- [04-scheduling-flow.md](../40-dataflows/04-scheduling-flow.md) —— 调度数据流
- [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) —— Micro-Supervisor 设计
- [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md) —— Macro-Supervisor 设计
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §3 —— [DSL] 3 态降级
- 综合修正方案 §1 —— sched_tac 设计依据

---

## §8 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Agent 8 态生命周期枚举；3 态降级（RUNNING/STOPPED/DEAD）；sched_tac 接口（sched_setattr/sched_setscheduler）；seL4 MCS 映射（scBudget→sched_runtime, scPeriod→sched_deadline）；物理宿主 kernel/include/uapi/linux/airymax/sched.h |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-7 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |
| v1.0.1 | 2026-07-26 | **v3.5 审查修复 P0-I4/I5**：(1) 移除虚构 `struct airy_sched_attr`，对齐 SSoT `struct airy_task_desc`（64 字节，11 字段，含 magic/prio/runtime_ns/deadline_ns/period_ns/vtime/agent_id/sched_policy/weight/state/reserved）；(2) 调度策略标识符从 `AIRY_SCHED_*`（值 0-2）修正为 `AIRY_SCHED_POLICY_*`（值 1-4，对齐 sched.h L97-100）；(3) include guard `_AIRYM_SCHED_H` → `_UAPI_AIRYMAX_SCHED_H`；(4) 8 态枚举补充 `AIRY_AGENT_STATE_MAX`；(5) [DSL] 块对齐 SSoT（`AIRY_DSL_SCHED_POLICY_*` + `AIRY_DSL_VTIME_DECAY`）；(6) 补充 `AIRY_TASK_MAGIC`/`AIRY_PRIO_MIN/MAX`/`AIRY_SLICE_DFL`/`AIRY_WEIGHT_MIN/MAX`/`airy_vtime_t`/`AIRY_VTIME_ONE`/`airy_vtime_decay()`；(7) `AIRY_MCS_MAP_*`/`AIRY_MCS_CHECK` 标注为文档说明性示例（非 SSoT 定义）；(8) 移除虚构 `AIRY_SCHED_VERSION`/`AIRY_SCHED_VERSION_CHECK` |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] sched.h 扩展契约 — sched_tac | v1.0.1 | 2026-07-26
