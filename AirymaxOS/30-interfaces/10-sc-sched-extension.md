Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] sched.h 扩展契约 — 方案 C-Prime
> **文档定位**：USV（统一生命周期监管）模块的调度扩展 [SC] 共享契约，定义 Agent 8 态生命周期与方案 C-Prime 调度接口的唯一权威契约\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7\
> **设计依据**：[15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §1（方案 C-Prime）+ §4.2.4（USV 设计）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **USV 模块调度扩展 [SC] 契约** 的唯一权威源。Agent 8 态生命周期枚举、3 态降级映射、方案 C-Prime 调度接口（`sched_setattr` / `sched_setscheduler`）、seL4 MCS 语义映射（`scBudget` ↔ `sched_runtime`、`scPeriod` ↔ `sched_deadline`）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Agent 状态机与调度参数契约。
>
> 技术选型声明：本文件遵循 **方案 C-Prime**（SCHED_DEADLINE / SCHED_FIFO / EEVDF + seL4 MCS 语义映射，**不使用 sched_ext**）。IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（不使用 page flipping）。安全采用 **纯 C LSM 模块**（不使用 BPF LSM）。日志内存采用 **alloc_pages + mmap**（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：agentrt-linux 内核调度开发者、USV 模块开发者、Macro-Supervisor 开发者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) §7 USV 模块、[04-scheduling-flow.md](../40-dataflows/04-scheduling-flow.md) 调度数据流、Linux 6.6 `sched_setattr(2)` 与 seL4 MCS（Mixed-Criticality Scheduling）
- **预计阅读时间**：25 分钟
- **核心概念**：方案 C-Prime、Agent 8 态、3 态降级、SCHED_DEADLINE、SCHED_FIFO、EEVDF、seL4 MCS、sched_context_t
- **复杂度标识**：高级

---

## §1 契约概述：USV 模块的调度扩展 [SC] 共享契约

### 1.1 契约定位

`kernel/include/airymax/sched.h` 是 USV 模块的 [SC] 共享契约头文件，定义 Agent 生命周期状态机与方案 C-Prime 调度参数的统一契约。该头文件由 agentrt（用户态 SDK）与 agentrt-linux（内核态实现）双端逐字节共享，CI 通过 `sc-dual-ci.yml` 验证两端一致性。

### 1.2 契约边界

本 [SC] 契约仅定义以下三部分内容，**不涉及调度策略实现**：

| 内容 | 类型 | 边界约束 |
|------|------|---------|
| Agent 状态枚举 | `enum airy_agent_state` | 仅定义状态码，不定义状态机迁移逻辑 |
| 调度参数结构 | `struct airy_sched_attr` | 仅定义二进制布局，不定义校验逻辑 |
| seL4 MCS 映射宏 | `AIRY_MCS_MAP_*` | 仅定义字段映射关系，不定义内核实现 |

### 1.3 与方案 C-Prime 的关系

方案 C-Prime 是本 [SC] 契约的调度机制选型依据（详见 [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §1）。方案 C-Prime 的核心决策为：

1. **不使用 sched_ext**：sched_ext 在 Linux 6.6 标准内核不可用，且需要 BPF 调度器，与"纯 C LSM"原则冲突
2. **复用主线调度器**：直接使用 `SCHED_DEADLINE` / `SCHED_FIFO` / `SCHED_NORMAL`(EEVDF) 三类已有调度策略
3. **借鉴 seL4 MCS 语义**：将 seL4 的 `sched_context_t` 预算/周期模型映射到 Linux `sched_attr` 字段，获得 MCS 语义但不修改内核调度器

---

## §2 Agent 8 态生命周期：INACTIVE/SPAWNING/READY/RUNNING/BLOCKED/STOPPING/STOPPED/DEAD

### 2.1 8 态枚举定义

```c
/* kernel/include/airymax/sched.h —— Agent 8 态生命周期枚举 */
#ifndef _AIRYM_SCHED_H
#define _AIRYM_SCHED_H

#include "uapi_compat.h"

/**
 * enum airy_agent_state - Agent 8 态生命周期
 *
 * 方案 C-Prime 核心成果：8 态与 Linux 进程状态天然映射，
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

当 [DSL] 降级生存层激活（`AIRY_SC_FALLBACK` 定义）或 Macro-Supervisor 故障时，USV 将 8 态降级为 3 态，仅保留最小可运行子集：

```c
/* kernel/include/airymax/sched.h —— [DSL] 3 态降级 */
#ifdef AIRY_SC_FALLBACK
/* 降级块：仅保留 3 态，对齐 EEVDF 默认调度 */
#define AIRY_AGENT_RUNNING   3   /* 运行中 */
#define AIRY_AGENT_STOPPED   6   /* 已停止 */
#define AIRY_AGENT_DEAD      7   /* 已终止 */
#warning "AIRY_SC_FALLBACK active: only 3-state lifecycle available"
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

## §4 方案 C-Prime 接口：sched_setattr / sched_setscheduler

### 4.1 调度参数结构

```c
/* kernel/include/airymax/sched.h —— 方案 C-Prime 调度参数 */
/**
 * struct airy_sched_attr - Agent 调度参数（方案 C-Prime）
 *
 * @sched_policy:   调度策略（SCHED_DEADLINE / SCHED_FIFO / SCHED_NORMAL）
 * @sched_runtime:  CPU 预算（纳秒），对齐 seL4 scBudget
 * @sched_deadline: 相对截止时间（纳秒），对齐 seL4 scPeriod
 * @sched_period:   周期（纳秒），对齐 seL4 scPeriod
 * @sched_priority: 静态优先级（仅 SCHED_FIFO 使用，1-99）
 * @sched_flags:    标志位（AIRY_SCHED_FLAG_*）
 *
 * 二进制布局由 [SC] 共享，两端逐字节相同。
 * 调度策略选择逻辑由 Macro-Supervisor [SS] 层实现。
 */
struct airy_sched_attr {
    __u32 sched_policy;
    __u64 sched_runtime;
    __u64 sched_deadline;
    __u64 sched_period;
    __u32 sched_priority;
    __u32 sched_flags;
} __attribute__((packed));

/* 调度策略选择（方案 C-Prime 三层） */
#define AIRY_SCHED_DEADLINE   0   /* 实时 Agent：SCHED_DEADLINE */
#define AIRY_SCHED_FIFO      1   /* 中断/IPC Agent：SCHED_FIFO */
#define AIRY_SCHED_EEVDF     2   /* 普通 Agent：SCHED_NORMAL(EEVDF) */
```

### 4.2 sched_setattr 设置 SCHED_DEADLINE

实时 Agent（如认知推理 Agent）使用 `sched_setattr()` 设置 SCHED_DEADLINE 参数：

```c
/* services/daemons/macro_superv/sched.c —— 设置 SCHED_DEADLINE */
static int airy_agent_set_deadline(pid_t pid, struct airy_sched_attr *attr)
{
    struct sched_attr sa = {
        .size         = sizeof(sa),
        .sched_policy = SCHED_DEADLINE,
        .sched_flags  = attr->sched_flags,
        .sched_runtime  = attr->sched_runtime,   /* CPU 预算 */
        .sched_deadline = attr->sched_deadline,  /* 截止时间 */
        .sched_period   = attr->sched_period,    /* 周期 */
    };
    /* sched_setattr：Linux 6.6 原生接口，零内核修改 */
    return syscall(__NR_sched_setattr, pid, &sa, 0);
}
```

### 4.3 sched_setscheduler 设置 SCHED_FIFO

中断/IPC Agent 使用 `sched_setscheduler()` 设置 SCHED_FIFO：

```c
/* services/daemons/macro_superv/sched.c —— 设置 SCHED_FIFO */
static int airy_agent_set_fifo(pid_t pid, struct airy_sched_attr *attr)
{
    struct sched_param sp = {
        .sched_priority = attr->sched_priority,  /* 1-99 */
    };
    /* sched_setscheduler：Linux 6.6 原生接口 */
    return sched_setscheduler(pid, SCHED_FIFO, &sp);
}
```

### 4.4 三层调度策略选择

| Agent 类型 | 调度策略 | 接口 | 典型参数 |
|-----------|---------|------|---------|
| 实时推理 Agent | SCHED_DEADLINE | `sched_setattr()` | runtime=10ms, deadline=50ms, period=50ms |
| IPC/中断 Agent | SCHED_FIFO | `sched_setscheduler()` | priority=50 |
| 普通 Agent | SCHED_NORMAL(EEVDF) | `sched_setscheduler()` | nice=0 |
| 后台 Daemon | SCHED_IDLE | `sched_setscheduler()` | — |

---

## §5 seL4 MCS 映射：sched_context_t 字段映射

### 5.1 MCS 语义映射

方案 C-Prime 借鉴 seL4 MCS（Mixed-Criticality Scheduling）模型，将 seL4 的 `sched_context_t` 字段映射到 Linux `sched_attr` 字段，**不修改内核调度器**即可获得 MCS 语义：

| seL4 字段 | seL4 类型 | Linux 映射字段 | Linux 类型 | 语义 |
|-----------|----------|--------------|-----------|------|
| `sched_context_t.scBudget` | `time_t` | `sched_attr.sched_runtime` | `__u64` | CPU 预算（纳秒） |
| `sched_context_t.scPeriod` | `time_t` | `sched_attr.sched_period` | `__u64` | 调度周期（纳秒） |
| `sched_context_t.scCore` | `word_t` | `sched_attr.sched_attr_cpu_set` | `cpu_set_t` | CPU 亲和性 |
| `sched_context_t.scRefill` | `word_t` | `sched_attr.sched_deadline` | `__u64` | 截止时间（周期内） |

### 5.2 映射宏定义

```c
/* kernel/include/airymax/sched.h —— seL4 MCS 映射宏 */
/**
 * seL4 MCS 字段映射宏
 * 将 seL4 sched_context_t 语义映射到 Linux sched_attr 字段
 * 映射关系仅用于文档与校验，不引入 seL4 依赖
 */
#define AIRY_MCS_MAP_BUDGET(sc_budget)    (sc_budget)        /* scBudget → sched_runtime */
#define AIRY_MCS_MAP_PERIOD(sc_period)    (sc_period)       /* scPeriod → sched_period */
#define AIRY_MCS_MAP_DEADLINE(sc_refill)  (sc_refill)       /* scRefill → sched_deadline */
#define AIRY_MCS_MAP_CORE(sc_core)        ((unsigned long)(sc_core)) /* scCore → cpu_set */

/* MCS 校验：budget <= deadline <= period（对齐 seL4 MCS 约束） */
#define AIRY_MCS_CHECK(attr) \
    (((attr)->sched_runtime <= (attr)->sched_deadline) && \
     ((attr)->sched_deadline <= (attr)->sched_period))
```

### 5.3 MCS 语义保持

映射后，Linux SCHED_DEADLINE 调度器在内核中保证以下 MCS 语义（由 CFS/DEADLINE 调度器实现，无需修改）：

1. **预算保证**：每个周期内 Agent 至少获得 `sched_runtime` 纳秒 CPU 时间
2. **截止时间保证**：CPU 预算在 `sched_deadline` 纳秒内交付
3. **周期重置**：每 `sched_period` 纳秒预算重置
4. **混合关键性**：高关键性 Agent 可在过载时抢占低关键性 Agent（通过优先级配置）

---

## §6 物理宿主与版本管理

### 6.1 物理宿主

| 项目 | 路径 | 说明 |
|------|------|------|
| [SC] 头文件 | `kernel/include/airymax/sched.h` | 双端逐字节共享 |
| 内核实现 | `kernel/superv/airy_sched.c` | 调度策略实现 |
| 用户态实现 | `services/daemons/macro_superv/sched.c` | 调度参数注入 |

### 6.2 版本号

```c
/* kernel/include/airymax/sched.h —— 版本号 */
#define AIRY_SCHED_VERSION  1   /* 契约版本号，不匹配时拒绝加载 */

/* 加载时校验 */
#define AIRY_SCHED_VERSION_CHECK(ver) \
    ((ver) == AIRY_SCHED_VERSION ? 0 : -AIRY_ECFGVERSION)
```

### 6.3 CI 双端校验

`sc-dual-ci.yml` 对 `sched.h` 执行以下校验：

1. **逐字节对比**：agentrt 与 agentrt-linux 的 `sched.h` 必须逐字节相同
2. **枚举值校验**：8 态枚举值必须在 `[0, 7]` 范围内且不重复
3. **结构体偏移校验**：`struct airy_sched_attr` 字段偏移在两端一致
4. **MCS 映射校验**：`AIRY_MCS_MAP_*` 宏定义两端一致

---

## §7 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §7 —— USV 模块总纲
- [04-scheduling-flow.md](../40-dataflows/04-scheduling-flow.md) —— 调度数据流
- [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) —— Micro-Supervisor 设计
- [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md) —— Macro-Supervisor 设计
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §3 —— [DSL] 3 态降级
- [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §1 —— 方案 C-Prime 设计依据

---

## §8 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Agent 8 态生命周期枚举；3 态降级（RUNNING/STOPPED/DEAD）；方案 C-Prime 接口（sched_setattr/sched_setscheduler）；seL4 MCS 映射（scBudget→sched_runtime, scPeriod→sched_deadline）；物理宿主 kernel/include/airymax/sched.h |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] sched.h 扩展契约 — 方案 C-Prime | v1.0 | 2026-07-17
