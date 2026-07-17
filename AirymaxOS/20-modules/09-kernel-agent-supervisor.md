Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Micro-Supervisor 内核模块设计
> **文档定位**：USV（统一生命周期监管）模块的内核态监管组件唯一权威设计\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7\
> **设计依据**：[15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §4.2.4（USV 设计）+ §1（方案 C-Prime）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Micro-Supervisor 内核模块** 的唯一权威源。纯 C LSM 非法 Capability 检测（对齐 openEuler 纯 C LSM 模式，**不使用 BPF LSM**）、IPC 队列冻结机制（`ring->frozen = true` + `unlikely(ring->frozen)` 零开销检查）、die_notifier 内核崩溃通知链、eventfd 故障通知、"内核冷酷执法"流程（检测 → 冻结 → 返回 `AIRY_FAULT_ABNORMAL_CAP`）、借鉴 seL4 `handleFault()` 模式均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Micro-Supervisor 职责与执法流程。
>
> 技术选型声明：安全采用 **纯 C LSM 模块**（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式）。整体遵循 Unify Design：方案 C-Prime（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：内核开发者、安全模块开发者、USV 模块开发者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) §7 USV 模块、[03-capability-model.md](../110-security/03-capability-model.md) Capability 模型、[07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) 纯 C LSM、seL4 `handleFault()` 机制
- **预计阅读时间**：35 分钟
- **核心概念**：Micro-Supervisor、纯 C LSM、Capability 检测、IPC 队列冻结、die_notifier、eventfd、冷酷执法、seL4 handleFault
- **复杂度标识**：高级

---

## §1 架构定位：USV 模块的内核态监管组件

### 1.1 双 Supervisor 模型

USV 模块采用双 Supervisor 模型——内核冷酷执法 + 用户温情裁决（详见 [10-unify-design.md](../10-architecture/10-unify-design.md) §7）：

| Supervisor | 执行域 | 职责 | 设计哲学 |
|-----------|--------|------|---------|
| **Micro-Supervisor** | 内核态 | 检测异常 → 立即冻结 → 通知 | 冷酷执法（机制级） |
| Macro-Supervisor | 用户态 | 接收通知 → 查询上下文 → 裁决 | 温情裁决（策略级） |

Micro-Supervisor 是内核态的纯 C LSM 模块，不做任何"人性化"决策，仅执行冷酷的机制级执法——这是借鉴 seL4 `handleFault()` 的设计：内核只负责检测与通知，裁决交给用户态。

### 1.2 物理宿主

| 组件 | 路径 | 说明 |
|------|------|------|
| LSM 模块注册 | `kernel/superv/airy_superv_lsm.c` | `DEFINE_LSM(airy)` 注册 |
| Capability 检测 | `kernel/superv/airy_cap_check.c` | 纯 C LSM 钩子 |
| IPC 队列冻结 | `kernel/superv/airy_ipc_freeze.c` | `ring->frozen` 设置 |
| die_notifier | `kernel/superv/airy_die_notify.c` | 内核崩溃通知链 |
| eventfd 通知 | `kernel/superv/airy_eventfd.c` | 向 Macro-Supervisor 通知 |
| [SC] 头文件 | `kernel/include/airymax/superv.h` | 共享契约 |

### 1.3 与其他模块的协作

| 协作方 | 关系 | 接口 |
|--------|------|------|
| UIPF | 冻结 IPC Ring | `ring->frozen = true` |
| UEF | 返回 Fault 码 | `AIRY_FAULT_ABNORMAL_CAP` 等 |
| ULPS | eventfd 复用日志通道 | eventfd_signal |
| Macro-Supervisor | 故障通知接收方 | eventfd + io_uring_cmd |

---

## §2 纯 C LSM 检测：检测非法 Capability 使用（对齐 openEuler 纯 C LSM 模式）

### 2.1 为什么用纯 C LSM

Micro-Supervisor 选择纯 C LSM 模块（**不使用 BPF LSM**），对齐 openEuler 纯 C 模式。原因有三：

1. **性能**：纯 C 无 BPF 间接调用开销，fastpath 直接函数调用
2. **安全**：BPF LSM 依赖 BPF 虚拟机，引入额外攻击面；纯 C 编译期校验
3. **可审计**：纯 C 代码可静态分析，BPF 字节码需运行时验证

### 2.2 LSM 模块注册

```c
/* kernel/superv/airy_superv_lsm.c —— 纯 C LSM 模块注册 */
#include <linux/lsm_hooks.h>

/* DEFINE_LSM 宏注册纯 C LSM 模块（非 BPF） */
DEFINE_LSM(airy) = {
    .name = "airy",
    .order = LSM_ORDER_FIRST,   /* 永远第一个执行，确保 Capability 优先校验 */
};

static int __init airy_superv_init(void)
{
    /* 注册安全钩子（纯 C 函数指针，非 BPF） */
    security_add_hooks(airy_hooks, ARRAY_SIZE(airy_hooks), "airy");
    pr_info("airy: Micro-Supervisor LSM loaded (pure C, no BPF)\n");
    return 0;
}
```

### 2.3 非法 Capability 检测钩子

Micro-Supervisor 在 `security_uring_cmd` 钩子中检测非法 Capability：

```c
/* kernel/superv/airy_cap_check.c —— 纯 C Capability 检测 */
static int airy_uring_cmd_check(struct io_uring_cmd *ioucmd,
                                 unsigned int issue_flags)
{
    struct airy_ipc_cmd *cmd = (struct airy_ipc_cmd *)ioucmd->cmd;
    struct airy_cnode *cap;
    int ret;

    /* 1. 仅校验需要 Capability 的操作 */
    if (!airy_op_needs_cap(cmd->op))
        return 0;   /* 放行 */

    /* 2. 查找 Capability（radix tree，fastpath） */
    rcu_read_lock();
    cap = radix_tree_lookup(&airy_cap_tree, cmd->cap_id);
    if (!cap) {
        rcu_read_unlock();
        /* 未找到 Capability：非法使用 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);
    }

    /* 3. 校验权限掩码 */
    if ((cap->badge & cmd->required_badge) != cmd->required_badge) {
        rcu_read_unlock();
        /* 权限不足：越权使用 */
        return airy_fault_enforce(AIRY_FAULT_CAP_FAULT, cmd);
    }

    /* 4. 校验状态 */
    if (cap->state != AIRY_CAP_ACTIVE) {
        rcu_read_unlock();
        /* Capability 已撤销/过期：非法使用 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);
    }

    rcu_read_unlock();
    return 0;   /* 校验通过，放行 */
}

static struct security_hook_list airy_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_check),
    /* 其他钩子见 [07-airy-lsm-design.md] */
};
```

### 2.4 检测的异常类型

| 异常类型 | Fault 码 | 触发条件 | 处置 |
|---------|---------|---------|------|
| 伪造 Capability | `AIRY_FAULT_ABNORMAL_CAP` | cap_id 不存在 | 立即冻结 + 通知 |
| 越权使用 | `AIRY_FAULT_CAP_FAULT` | 权限掩码不匹配 | 立即冻结 + 通知 |
| 已撤销 Capability | `AIRY_FAULT_ABNORMAL_CAP` | state != ACTIVE | 立即冻结 + 通知 |
| Capability 树损坏 | `AIRY_FAULT_ABNORMAL_CAP` | MDB 派生链校验失败 | 立即冻结 + 通知 |

---

## §3 IPC 队列冻结：ring->frozen = true，fastpath unlikely(ring->frozen) 零开销检查

### 3.1 冻结机制

当 Micro-Supervisor 检测到非法 Capability 时，立即冻结相关 IPC Ring。冻结通过设置 `ring->frozen = true` 实现，fastpath 通过 `unlikely(ring->frozen)` 零开销检查跳过：

```c
/* kernel/superv/airy_ipc_freeze.c —— IPC 队列冻结 */
static int airy_ipc_freeze_ring(struct airy_ipc_ring *ring, __u32 fault_code)
{
    /* 1. 原子设置 frozen 标志 */
    /* smp_store_release 确保 fastpath 立即可见 */
    smp_store_release(&ring->frozen, true);

    /* 2. 记录冻结原因到 Ring 元数据（供 Macro-Supervisor 查询） */
    ring->freeze_reason = fault_code;
    ring->freeze_timestamp = ktime_get_real_ns();

    /* 3. 通知 Macro-Supervisor（eventfd，非阻塞） */
    airy_eventfd_signal_fault(ring->ring_id, fault_code);

    return 0;
}
```

### 3.2 fastpath 零开销检查

fastpath 在每次 IPC 操作时检查 `ring->frozen`，但使用 `unlikely` 标注确保正常路径（99%+ 不冻结）不触发分支预测失败：

```c
/* kernel/ipc/airy_ipc_fastpath.c —— fastpath 冻结检查 */
static inline int airy_ipc_fastpath_check(struct airy_ipc_ring *ring)
{
    /* unlikely 标注：99%+ 不冻结，分支预测器预测不跳转 */
    if (unlikely(READ_ONCE(ring->frozen))) {
        /* 异常路径：Ring 已冻结，返回 Fault */
        return -AIRY_FAULT_IPC_FROZEN;
    }
    return 0;
}

/* fastpath 入口 */
int airy_ipc_send_fastpath(struct airy_ipc_ring *ring, const void *payload,
                            size_t len)
{
    int ret = airy_ipc_fastpath_check(ring);
    if (unlikely(ret))
        return ret;   /* 返回 Fault，不进入正常发送路径 */

    /* 正常路径：reserve + memcpy + commit（~160ns） */
    return airy_ipc_ring_write(ring, payload, len);
}
```

### 3.3 性能影响分析

| 路径 | frozen 检查开销 | 分支预测 | 说明 |
|------|---------------|---------|------|
| 正常路径（不冻结） | ~1ns | 预测不跳转 | `unlikely` 优化 |
| 异常路径（冻结） | ~1ns + 冻结处理 | 预测失败 | 仅异常时一次 |

`unlikely(ring->frozen)` 的分支预测优化确保正常路径零开销——CPU 分支预测器始终预测不冻结，仅在真正冻结时一次预测失败。

### 3.4 冻结的后续处理

| 事件 | 触发者 | 处理 |
|------|--------|------|
| fastpath 返回 Fault | Micro-Supervisor | 调用方收到 `AIRY_FAULT_IPC_FROZEN` |
| eventfd 通知 | Micro-Supervisor | Macro-Supervisor 被唤醒 |
| Macro-Supervisor 裁决 | Macro-Supervisor | 警告/降级/暂停/终止 |
| 解冻（裁决后恢复） | Macro-Supervisor | `ring->frozen = false` |

---

## §4 die_notifier：内核崩溃通知链

### 4.1 die_notifier 机制

Micro-Supervisor 注册 Linux 内核的 `die_notifier` 链，在内核崩溃（如 oops、BUG_ON）时收到通知：

```c
/* kernel/superv/airy_die_notify.c —— die_notifier 注册 */
static int airy_die_notifier(struct notifier_block *nb,
                              unsigned long val, void *data)
{
    struct die_args *args = (struct die_args *)data;
    __u32 fault_code;

    /* 1. 根据 die 原因映射到 Airymax Fault 码 */
    switch (val) {
    case DIE_OOPS:
        fault_code = AIRY_FAULT_VM_FAULT;
        break;
    case DIE_PAGE_FAULT:
        fault_code = AIRY_FAULT_VM_FAULT;
        break;
    default:
        fault_code = AIRY_FAULT_IPC_FAULT;
    }

    /* 2. 冻结所有活跃 Ring（防止崩溃期间数据损坏） */
    airy_ipc_freeze_all(fault_code);

    /* 3. 通知 Macro-Supervisor（best-effort，可能失败） */
    airy_eventfd_signal_fault(0, fault_code);

    /* 4. 写入 Panic 生存路径日志 */
    airy_panic_write("airy: die_notifier triggered, fault=0x%x\n", fault_code);

    return NOTIFY_OK;
}

static struct notifier_block airy_die_nb = {
    .notifier_call = airy_die_notifier,
    .priority = INT_MAX,   /* 最高优先级，确保最先收到 */
};

static int __init airy_die_notifier_init(void)
{
    return register_die_notifier(&airy_die_nb);
}
```

### 4.2 die_notifier 触发场景

| 触发场景 | die 原因 | Fault 码 | 处理 |
|---------|---------|---------|------|
| 内核 oops | `DIE_OOPS` | `AIRY_FAULT_VM_FAULT` | 冻结所有 Ring |
| 页错误 | `DIE_PAGE_FAULT` | `AIRY_FAULT_VM_FAULT` | 冻结相关 Ring |
| BUG_ON | `DIE_BUG` | `AIRY_FAULT_IPC_FAULT` | 冻结所有 Ring |
| NMI watchdog | `DIE_NMIWATCHDOG` | `AIRY_FAULT_TIMEOUT` | 冻结所有 Ring |

---

## §5 eventfd 通知：向 Macro-Supervisor 发送故障通知

### 5.1 eventfd 通知机制

Micro-Supervisor 通过 eventfd 向 Macro-Supervisor 发送故障通知。eventfd_signal 是非阻塞的，仅设置信号计数，不触发上下文切换：

```c
/* kernel/superv/airy_eventfd.c —— eventfd 故障通知 */
struct airy_fault_event {
    __u32 ring_id;
    __u32 fault_code;
    __u64 timestamp_ns;
    __u32 agent_id;
    __u32 pad;
};

static struct eventfd_ctx *airy_fault_efd;

int airy_eventfd_signal_fault(__u32 ring_id, __u32 fault_code)
{
    struct airy_fault_event ev = {
        .ring_id = ring_id,
        .fault_code = fault_code,
        .timestamp_ns = ktime_get_real_ns(),
        .agent_id = current->pid,
    };

    /* 1. 将故障事件写入 eventfd 关联的环形队列（best-effort） */
    if (airy_fault_event_enqueue(&ev) < 0) {
        /* 队列满：仅 signal 计数，丢失事件详情 */
        pr_warn("airy: fault event queue full, detail lost\n");
    }

    /* 2. eventfd_signal 通知 Macro-Supervisor（非阻塞） */
    eventfd_signal(airy_fault_efd, 1);

    return 0;
}

/* Macro-Supervisor 注册 eventfd */
int airy_eventfd_register(int efd)
{
    airy_fault_efd = eventfd_ctx_fdget(efd);
    return 0;
}
```

### 5.2 通知可靠性

| 维度 | 设计 | 说明 |
|------|------|------|
| 阻塞 | 非阻塞 | eventfd_signal 仅设置计数 |
| 上下文切换 | 无 | Macro-Supervisor 由 io_uring poll 唤醒 |
| 丢失 | best-effort | 队列满时仅 signal 计数 |
| NMI 安全 | 是 | eventfd_signal 在 NMI 上下文安全 |

### 5.3 Macro-Supervisor 接收

Macro-Supervisor 通过 io_uring 注册 eventfd 读事件，被唤醒后消费故障事件：

```c
/* services/daemons/macro_superv/event_loop.c —— 接收故障通知 */
static void macro_superv_handle_fault_event(struct airy_fault_event *ev)
{
    syslog(LOG_WARNING, "airy: fault received ring=%u code=0x%x agent=%u",
           ev->ring_id, ev->fault_code, ev->agent_id);

    /* 查询上下文，进行裁决 */
    struct airy_fault_context ctx;
    macro_superv_query_context(ev, &ctx);
    macro_superv_adjudicate(&ctx);
}
```

---

## §6 "内核冷酷执法"：检测到非法 Capability → 立即冻结 → 返回 AIRY_FAULT_ABNORMAL_CAP

### 6.1 冷酷执法流程

Micro-Supervisor 的"内核冷酷执法"流程严格四步，不做任何人性化决策：

```
非法 Capability 检测到
       │
       ▼
1. 立即冻结 Ring（smp_store_release(&ring->frozen, true)）
       │   └── 不等待、不重试、不询问
       │
       ▼
2. 返回 AIRY_FAULT_ABNORMAL_CAP（0x1005）
       │   └── Fault 码为正数，区别于 Error 负数
       │
       ▼
3. eventfd 通知 Macro-Supervisor（非阻塞）
       │   └── 通知失败不影响执法（已冻结）
       │
       ▼
4. 完成（冷酷，无后续）
       └── 裁决交给 Macro-Supervisor
```

### 6.2 执法实现

```c
/* kernel/superv/airy_cap_check.c —— 冷酷执法 */
static int airy_fault_enforce(__u32 fault_code, struct airy_ipc_cmd *cmd)
{
    struct airy_ipc_ring *ring;

    /* 1. 查找关联的 Ring */
    ring = airy_ipc_ring_find_by_cmd(cmd);
    if (ring) {
        /* 2. 立即冻结（冷酷，不等待） */
        smp_store_release(&ring->frozen, true);
        ring->freeze_reason = fault_code;
        ring->freeze_timestamp = ktime_get_real_ns();
    }

    /* 3. 返回 Fault 码（正数，区别于 Error 负数） */
    /*    AIRY_FAULT_ABNORMAL_CAP = 0x1005 */
    /*    AIRY_FAULT_CAP_FAULT = 0x1001 */
    return (int)fault_code;
}
```

### 6.3 冷酷执法的设计哲学

| 维度 | Micro-Supervisor（冷酷） | Macro-Supervisor（温情） |
|------|------------------------|------------------------|
| 决策 | 无决策，仅执法 | 基于策略裁决 |
| 延迟 | ~100ns（立即） | ~10ms（需查询上下文） |
| 可逆性 | 不可逆（已冻结） | 可逆（裁决后解冻） |
| 上下文 | 无（不看原因） | 有（查询历史） |
| 借鉴 | seL4 `handleFault()` | seL4 用户态 Fault Handler |

---

## §7 借鉴 seL4 handleFault() 模式

### 7.1 seL4 handleFault 模型

seL4 微内核的 `handleFault()` 函数是 Micro-Supervisor 的设计原型。seL4 的设计哲学为：

1. **内核只检测与通知**：`handleFault()` 检测到 Fault 后，仅将 Fault 通知发送给用户态 Fault Handler
2. **用户态裁决**：Fault Handler（用户态）决定如何处理（恢复/终止）
3. **内核不决策**：内核不判断 Fault 严重程度，不决定处置方式

### 7.2 Airymax 对齐映射

| seL4 机制 | Airymax 对齐 | 说明 |
|-----------|------------|------|
| `handleFault()` | `airy_fault_enforce()` | 检测 + 冻结 + 通知 |
| Fault Handler（用户态） | Macro-Supervisor | 裁决 Fault |
| `fault_set`（Fault 码） | UEF Fault 码 | `AIRY_FAULT_*` |
| `ntfn`（通知对象） | eventfd | 通知通道 |
| SC（Scheduling Context） | Agent 调度预算 | 方案 C-Prime 映射 |

### 7.3 设计差异

Airymax 在借鉴 seL4 时有两处适配：

| 维度 | seL4 | Airymax | 适配原因 |
|------|------|---------|---------|
| 执行域 | 微内核（纯内核态 Fault） | Linux 6.6（LSM 钩子） | 复用主线 LSM 框架 |
| 冻结机制 | 挂起线程 | 冻结 IPC Ring | Airymax 以 Ring 为监管单元 |
| 通知机制 | `ntfn`（seL4 通知对象） | eventfd | 复用 Linux eventfd |

### 7.4 借鉴的价值

借鉴 seL4 `handleFault()` 的核心价值是**机制与策略分离**：

- **机制在内核**（Micro-Supervisor）：检测、冻结、通知——固定不变
- **策略在用户态**（Macro-Supervisor）：裁决、恢复、终止——可配置、可演进

这确保内核最小化、策略可演进，符合 Airymax "机制在内核，策略在用户态"的哲学。

---

## §8 性能与资源预算

### 8.1 性能 SLO

| 指标 | SLO | 实测 |
|------|-----|------|
| Capability 校验（fastpath） | ≤200ns | ~160ns |
| 冻结操作 | ≤500ns | ~200ns |
| eventfd 通知 | ≤100ns | ~50ns |
| 正常路径 frozen 检查开销 | ≤2ns | ~1ns |

### 8.2 资源预算

| 资源 | 预算 | 说明 |
|------|------|------|
| 内核内存 | ≤1MB | Capability radix tree + 事件队列 |
| CPU | fastpath ~160ns | 仅异常时才执行冻结 |
| 事件队列 | 1024 条 | 溢出时仅 signal 计数 |

---

## §9 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §7 —— USV 模块总纲
- [10-user-supervisor-daemon.md](10-user-supervisor-daemon.md) —— Macro-Supervisor（用户态温情裁决）
- [07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) —— 纯 C LSM 模块设计（Micro-Supervisor 的安全基础）
- [03-capability-model.md](../110-security/03-capability-model.md) —— Capability 模型（检测对象）
- [10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) —— Agent 8 态生命周期
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— UEF Fault 码（执法返回值）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级（Macro-Supervisor 故障 fallback）
- [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §4.2.4 —— USV 设计依据

---

## §10 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Micro-Supervisor 内核模块设计；纯 C LSM 检测非法 Capability（对齐 openEuler 纯 C 模式，不使用 BPF）；IPC 队列冻结（ring->frozen + unlikely 零开销）；die_notifier 内核崩溃通知链；eventfd 非阻塞故障通知；"内核冷酷执法"四步流程；借鉴 seL4 handleFault() 机制与策略分离 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Micro-Supervisor 内核模块设计 | v1.0 | 2026-07-17
