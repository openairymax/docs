Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Micro-Supervisor 内核模块设计
> **文档定位**：A-ULS（统一生命周期管理）模块的内核态监管组件唯一权威设计\
> **文档版本**：v1.1.1（落后内容修复版，基于 v1.1 Capability Folding）\
> **最后更新**：2026-07-19\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7\
> **设计依据**：本文件为 SSoT（Single Source of Truth），Micro-Supervisor 内核模块职责、执法流程、FREEZE opcode、Badge 撤销解耦 atomic_inc 机制均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Micro-Supervisor 内核模块** 的唯一权威源。纯 C LSM 非法 Capability 检测（对齐 openEuler 纯 C LSM 模式，**不使用 BPF LSM**）、IPC 队列冻结机制（`ring->frozen = true` + `unlikely(ring->frozen)` 零开销检查）、die_notifier 内核崩溃通知链、eventfd 故障通知、"内核冷酷执法"流程（检测 → 冻结 → 返回 `AIRY_FAULT_ABNORMAL_CAP`）、借鉴 seL4 `handleFault()` 模式、**FREEZE opcode（0x0005）ring 冻结语义**、**Badge 撤销解耦 atomic_inc 机制**（1 行 `atomic_inc` 立即失效所有已发出 Badge，无 drain、无 bitmap）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Micro-Supervisor 职责与执法流程。
>
> **v1.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.1 起，Micro-Supervisor 的 capability 校验路径从"独立前置 `airy_cap_check()` + radix tree 查找"重构为 **fastpath C-S9 内联 Badge 校验**（`airy_cap_badge_ok()`，~10ns）。Micro-Supervisor 不再在 `security_uring_cmd` 钩子中独立执行 capability 校验——fastpath C-S9 已在 io_uring 数据面完成 Badge 校验，LSM 钩子仅在 slowpath（异常路径）做策略裁决。Badge 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码立即失效所有已发出 Badge，与 IPC fastpath 完全解耦（无 drain、无 bitmap、无 IPI）。详见 §3.5 FREEZE opcode 与 §6.4 Badge 撤销解耦。
>
> 技术选型声明：安全采用 **纯 C LSM 模块**（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。
>
> **ADR 引用声明**：本设计遵循 ADR-012（对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造技术路线确立）与 ADR-014（微内核设计思想来源单一化——seL4 为唯一思想来源，禁止混用 Mach/L4/Fuchsia/Zircon 等其他微内核设计模式）。Micro-Supervisor 的"冷酷执法 + 用户态裁决"机制级/策略级分离严格对齐 seL4 `handleFault()` 模型，不引入其他微内核的等价机制。IRON-9 v3 四层模型（[SC] 共享契约层 / [SS] 语义同源层 / [IND] 完全独立层 / [DSL] 降级生存层）是 Micro-Supervisor 与 Macro-Supervisor 协作的同源映射框架，详见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)。

---

## 文档信息卡

- **目标读者**：内核开发者、安全模块开发者、A-ULS 模块开发者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) §7 A-ULS 模块、[03-capability-model.md](../110-security/03-capability-model.md) Capability 模型、[07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) 纯 C LSM、seL4 `handleFault()` 机制
- **预计阅读时间**：35 分钟
- **核心概念**：Micro-Supervisor、纯 C LSM、Capability 检测、IPC 队列冻结、die_notifier、eventfd、冷酷执法、seL4 handleFault
- **复杂度标识**：高级

---

## §1 架构定位：A-ULS 模块的内核态监管组件

### 1.1 双 Supervisor 模型

A-ULS 模块采用双 Supervisor 模型——内核冷酷执法 + 用户温情裁决（详见 [10-unify-design.md](../10-architecture/10-unify-design.md) §7）：

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
| [SC] 头文件 | `kernel/include/uapi/linux/airymax/superv.h` | 共享契约 |

### 1.3 与其他模块的协作

| 协作方 | 关系 | 接口 |
|--------|------|------|
| A-IPC | 冻结 IPC Ring | `ring->frozen = true` |
| A-UEF | 返回 Fault 码 | `AIRY_FAULT_ABNORMAL_CAP` 等 |
| A-ULP | eventfd 复用日志通道 | eventfd_signal |
| Macro-Supervisor | 故障通知接收方 | eventfd + io_uring_cmd |
| sched_tac（sched_d） | Agent 8 态生命周期联动 + 调度预算冻结 | `airy_sys_sched_ctl`（编号 2） |

### 1.4 与 12 daemon 的协作关系（v1.1 新增）

Micro-Supervisor 作为内核态监管组件，与 Macro-Supervisor 管理的 12 个 daemon 通过 fastpath C-S9 Badge 校验 + slowpath LSM 钩子协作。12 daemon 完整名单（详见 [10-user-supervisor-daemon.md §1.3](10-user-supervisor-daemon.md)）：

| # | Daemon | 与 Micro-Supervisor 的协作 | Badge 校验路径 |
|---|--------|---------------------------|---------------|
| 1 | sec_d | Badge 唯一编译者（H4），`agent_caps[1024]` 唯一写者；sec_d 故障触发 [DSL] 降级 | 自举路径（CAP_REQUEST，badge=0 跳过 C-S9） |
| 2 | cogn_d | 认知调度守护，8 态生命周期管理；cogn_d 违规触发 Ring 冻结 | fastpath C-S9 |
| 3 | mem_d | 记忆管理守护；mem_d IPC 异常触发冷酷执法 | fastpath C-S9 |
| 4 | gateway_d | 跨节点 IPC 网关（1.0.1）；gateway_d Badge 校验在源节点完成 | fastpath C-S9（per-node） |
| 5 | logger_d | 日志消费守护（A-ULP）；logger_d 故障不影响 Badge 校验 | fastpath C-S9 |
| 6 | macro_superv | 故障通知接收方 + 裁决执行者；通过 `AIRY_IPC_OP_ADJUDICATE` 下发裁决 | fastpath C-S9（持 FREEZE 权限位） |
| 7 | audit_d | 审计守护；所有 Badge 编译/撤销/冻结事件写入 audit_d 审计链 | fastpath C-S9 |
| 8 | sched_d | sched_tac 策略守护进程；通过 `airy_sys_sched_ctl`（编号 2）控制调度参数 | fastpath C-S9 |
| 9 | dev_d | 设备驱动守护；dev_d IPC 异常触发 Ring 冻结 | fastpath C-S9 |
| 10 | net_d | 网络策略守护；net_d IPC 异常触发 Ring 冻结 | fastpath C-S9 |
| 11 | vfs_d | VFS 用户态服务；vfs_d IPC 异常触发 Ring 冻结 | fastpath C-S9 |
| 12 | config_daemon | 配置管理守护（A-UCS）；config_daemon 故障不影响 Badge 校验 | fastpath C-S9 |

> **daemon 命名约定**：daemon 命名后缀统一为 `_d`（sec_d/cogn_d/mem_d/gateway_d/logger_d/audit_d/sched_d/dev_d/net_d/vfs_d），例外为 `macro_superv`（主监管守护进程，沿用 Unify Design 命名）和 `config_daemon`（配置管理，沿用 `_daemon` 后缀）。12 daemon 完整名单以 [10-user-supervisor-daemon.md §1.3](10-user-supervisor-daemon.md) 为 SSoT。

### 1.5 与 sched_tac 的协作（v1.1 新增）

**sched_tac 策略守护进程**是 sched_d daemon（详见 [10-user-supervisor-daemon.md §1.3](10-user-supervisor-daemon.md)），通过 `airy_sys_sched_ctl`（syscall 编号 2）与 Micro-Supervisor 协作。协作关系如下：

| 协作维度 | sched_d 职责 | Micro-Supervisor 职责 | 接口 |
|---------|-------------|----------------------|------|
| Agent 8 态生命周期 | 通过 `airy_sys_sched_ctl` 配置 Agent 调度参数（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射） | Agent 态迁移时通知 sched_d 调整调度预算（如 STOPPED → READY 时恢复预算） | `airy_sys_sched_ctl` + eventfd |
| 调度预算冻结 | 收到 Micro-Supervisor 通知后冻结违规 Agent 的调度预算 | 检测到 Badge 异常 → 冻结 Ring + 通知 sched_d | eventfd + `airy_sys_sched_ctl` |
| [DSL] 降级 | 进入 [DSL] 降级模式时，调度退化为 EEVDF 默认 | 通知 sched_d 进入 [DSL] 降级模式 | eventfd |
| Badge 校验 | sched_d 作为 Agent 持有 Badge，fastpath C-S9 校验 | fastpath C-S9 校验 sched_d 的 Badge | io_uring `IORING_OP_URING_CMD` |

**Agent 8 态生命周期与 sched_tac 集成**（对齐 [10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md)）：

```
Agent 8 态:  CREATED → READY → RUNNING → STOPPED → DEAD
                                       ↘ PAUSED ↗
                                       ↘ DEGRADED ↗
                                       ↘ TERMINATED → DEAD

Micro-Supervisor 在态迁移时通知 sched_d:
  - RUNNING → STOPPED: kill(SIGSTOP)，sched_d 冻结调度预算
  - STOPPED → READY:   kill(SIGCONT)，sched_d 恢复调度预算
  - RUNNING → DEGRADED: sched_d 缩减调度预算（不停止进程）
  - * → TERMINATED:    sched_d 释放调度预算 + Micro-Supervisor 撤销 Badge
  - * → DEAD:          sched_d 释放调度预算 + Micro-Supervisor 撤销 Badge + 冻结 Ring
```

**`airy_sys_sched_ctl`（编号 2）子命令**：

| 子命令 | 功能 | 调用者 |
|--------|------|--------|
| `SCHED_SET_POLICY` | 设置 Agent 调度策略（SCHED_DEADLINE/SCHED_FIFO/EEVDF） | 仅 sched_d |
| `SCHED_SET_BUDGET` | 设置 Agent 调度预算（runtime/deadline/period） | 仅 sched_d |
| `SCHED_FREEZE` | 冻结 Agent 调度预算（Micro-Supervisor 通知） | 仅 sched_d |
| `SCHED_THAW` | 恢复 Agent 调度预算（Micro-Supervisor 通知） | 仅 sched_d |

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
    .order = LSM_ORDER_MUTABLE,   /* v1.1: LSM_ORDER_MUTABLE + CONFIG_LSM 首位（替代原 v1.0 误用的 capabilities-only 顺序常量，OLK 6.6 注释明确后者仅用于 capabilities） */
};

static int __init airy_superv_init(void)
{
    /* 注册安全钩子（纯 C 函数指针，非 BPF） */
    security_add_hooks(airy_hooks, ARRAY_SIZE(airy_hooks), "airy");
    pr_info("airy: Micro-Supervisor LSM loaded (pure C, no BPF)\n");
    return 0;
}
```

### 2.3 非法 Capability 检测钩子（v1.1: fastpath C-S9 + slowpath LSM 严格分工，禁止重复校验）

**v1.1 变更**：Micro-Supervisor 不再在 `security_uring_cmd` 钩子中独立执行 capability 校验。fastpath C-S9 已在 io_uring 数据面完成 Badge 校验（`airy_cap_badge_ok()`，~10ns，详见 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md)）。LSM 钩子仅在 slowpath（异常路径）做策略裁决——当 fastpath C-S9 返回 `-AIRY_ECAP_FORGED` (-80) 等 Badge 错误码时，Micro-Supervisor 接管并执行冷酷执法。

**fastpath/slowpath 严格分工（禁止重复校验）**：

| 路径 | 执行点 | 调用 `airy_cap_badge_ok()` | 触发条件 | 延迟 |
|------|--------|:---:|---------|------|
| **fastpath C-S9** | io_uring 数据面内联（`airy_ipc_validate()`） | ✓（唯一调用点） | 每次 IPC 消息投递 | ~10ns |
| **slowpath LSM** | `security_uring_cmd` 钩子（`airy_uring_cmd_check`） | ✗（**禁止重复调用**） | 仅 fastpath C-S9 返回 Badge 错误码时 | ~100ns |

> **关键约束**：LSM 钩子 `airy_uring_cmd_check` **不得再次调用 `airy_cap_badge_ok()`**——fastpath C-S9 已在 io_uring 数据面完成 Badge 校验，LSM 钩子仅读取 fastpath 返回的错误码（通过 `cmd->fastpath_ret` 字段传递）并执行冷酷执法。重复校验会导致：① fastpath ~10ns + slowpath ~100ns 双重开销；② Epoch 双读破坏单快照语义；③ 违反 Capability Folding H4 单写者 + 单校验点不变量。

```c
/* kernel/superv/airy_cap_check.c —— v1.1: slowpath LSM 钩子（fastpath C-S9 异常接管）
 * OLK 6.6 对齐: security_uring_cmd 钩子单参数（issue_flags 从 ioucmd->flags 获取）
 *
 * v1.1 fastpath/slowpath 严格分工（禁止重复校验）:
 *   - fastpath C-S9: io_uring 数据面内联 Badge 校验（airy_cap_badge_ok，~10ns）
 *     正常路径（Badge 通过）不进入 LSM 钩子，零开销
 *   - slowpath LSM: 仅在 fastpath C-S9 返回 Badge 错误码时被调用
 *     读取 cmd->fastpath_ret 字段执行冷酷执法，不再次调用 airy_cap_badge_ok()
 */
static int airy_uring_cmd_check(struct io_uring_cmd *ioucmd)
{
    /* OLK 6.6: struct io_uring_cmd 无 cmd 字段，使用 io_uring_cmd_to_pdu() 宏访问 pdu[32] */
    struct airy_ipc_cmd *cmd = io_uring_cmd_to_pdu(ioucmd, struct airy_ipc_cmd);
    int fastpath_ret;

    /* v1.1: fastpath C-S9 已在 io_uring 数据面完成 Badge 校验
     * LSM 钩子仅在 slowpath（异常路径）被调用
     * 详见 07-ipc-fastpath.md §5.2 C-S9 实现
     *
     * 禁止重复校验: LSM 钩子不调用 airy_cap_badge_ok()
     * 仅根据 fastpath 返回的错误码执行冷酷执法
     */

    /* 1. 读取 fastpath C-S9 的返回码（通过 cmd->fastpath_ret 字段传递）
     * fastpath 在 io_uring 数据面执行 airy_cap_badge_ok() 后
     * 将返回码写入 cmd->fastpath_ret，LSM 钩子 READ_ONCE 读取
     */
    fastpath_ret = READ_ONCE(cmd->fastpath_ret);

    /* 2. fastpath 校验通过（fastpath_ret == 0）: 直接放行
     * 正常路径不进入 slowpath 任何处置
     */
    if (likely(fastpath_ret == 0))
        return 0;

    /* 3. slowpath: Badge 校验失败，Micro-Supervisor 接管（不重复调用 airy_cap_badge_ok） */
    switch (fastpath_ret) {
    case -AIRY_ECAP_FORGED:     /* -80: Badge 伪造，触发 Fault */
        return airy_fault_enforce(AIRY_FAULT_CAP_FORGED, cmd);  /* 0x1001 */

    case -AIRY_ECAP_EPOCH:      /* -79: Epoch 不匹配（已撤销或过期） */
        /* Epoch 失效不一定是攻击，可能是 Agent 持有过期 Badge
         * 触发 Ring 冻结 + 通知 Macro-Supervisor 重新申请 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */

    case -AIRY_ECAP_PERM:       /* -81: 权限位不满足 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */

    case -AIRY_ECAP_FROZEN:     /* -82: Ring 已冻结 */
        /* Ring 冻结是 Error，不是 Fault，直接返回 */
        return fastpath_ret;

    case -AIRY_ECAP_BADGE:      /* -78: Badge 格式无效 / CAP_CARRY 但 badge=0 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */

    case -AIRY_ESEC_D_THROTTLED: /* -83: sec_d 限流拒绝（Badge 编译请求被限流） */
        /* sec_d 限流是 Error，不是 Fault，直接返回让调用方重试 */
        return fastpath_ret;

    default:
        /* 未知 Badge 错误，保守冻结 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */
    }
}

static struct security_hook_list airy_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_check),
    /* 其他钩子见 [07-airy-lsm-design.md] */
};
```

### 2.4 检测的异常类型（v1.1: 对齐 Capability Folding Badge 错误码）

| 异常类型 | Error 码 | Fault 码 | 触发条件（fastpath C-S9） | 处置 |
|---------|---------|---------|-------------------------|------|
| Badge 伪造 | `AIRY_ECAP_FORGED` (-80) | `AIRY_FAULT_CAP_FORGED` (0x1001) | `badge_randtag != agent_caps[src_task].randtag` | 立即冻结 + 通知 + sec_d 审计 |
| Badge 泄漏 | — | `AIRY_FAULT_CAP_LEAK` (0x1002) | sec_d 审计检测到跨 Agent 使用 Badge | 标记可疑 + 通知 Macro-Supervisor |
| Ring 数据损坏 | `AIRY_EIPC_CRC32` (-51) | `AIRY_FAULT_RING_CORRUPT` (0x1003) | CRC32 校验失败 + 元数据不一致 | 立即冻结 Ring |
| Epoch 不匹配 | `AIRY_ECAP_EPOCH` (-79) | `AIRY_FAULT_ABNORMAL_CAP` (0x1005) | `badge_epoch != global_epoch`（已撤销/过期） | 立即冻结 + 通知重新申请 |
| 权限不足 | `AIRY_ECAP_PERM` (-81) | `AIRY_FAULT_ABNORMAL_CAP` (0x1005) | `badge_perms & required != required` | 立即冻结 + 通知 |
| Badge 格式无效 | `AIRY_ECAP_BADGE` (-78) | `AIRY_FAULT_ABNORMAL_CAP` (0x1005) | Badge 格式无效 / CAP_CARRY 但 badge=0 | 立即冻结 + 通知 |
| Ring 已冻结 | `AIRY_ECAP_FROZEN` (-82) | —（Error，不触发 Fault） | `ring->frozen == true`（C-S0 检查） | 直接返回 Error，等待解冻 |
| Capability 槽位损坏 | — | `AIRY_FAULT_ABNORMAL_CAP` (0x1005) | `agent_caps[agent_id].state` 异常或 Badge 64-bit 编码不一致（Epoch/RandomTag/Perms 三段校验失败） | 立即终止 Agent（SIGKILL） |
| io_uring SQE 畸变（**上游检测**） | — | `AIRY_FAULT_URING_MALFORMED` (0x100A) | `airy_uring_sqe_validate()` 5 阶段校验失败（opcode/flags/payload_len 越界或 CRC32 不匹配，详见 [06-io-uring-hardening.md §3.5.3](../110-security/06-io-uring-hardening.md)） | `airy_security_fault()` 通知 Micro-Supervisor + 冻结 Ring（**在 fastpath C-S9 之前执行，不进入 §2.3 LSM 钩子**） |
| sec_d 限流拒绝（**上游检测**） | `AIRY_ESEC_D_THROTTLED` (-83) | —（Error，不触发 Fault） | sec_d 令牌桶限流命中（per-Agent 配额/全局阈值/排队超限，详见 [10-user-supervisor-daemon.md §4.6](10-user-supervisor-daemon.md)） | 直接返回 Error 让调用方退避重试（**在 Badge 编译路径触发，不进入 fastpath C-S9**） |

> **错误码 SSoT**：所有 Error/Fault 码定义以 [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) 为唯一权威源。Ring 冻结是 Error（-82），不是 Fault——Error 可恢复（等待解冻），Fault 不可恢复（Micro-Supervisor 接管）。`AIRY_FAULT_URING_MALFORMED` (0x100A) 与 `AIRY_ESEC_D_THROTTLED` (-83) 是上游检测路径的扩展错误码，**不进入 §2.3 LSM 钩子的 slowpath 处置**（前者在 `airy_uring_sqe_validate()` 拦截，后者在 sec_d Badge 编译路径拦截），列出以完整覆盖 Micro-Supervisor 可能涉及的全部异常类型。

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

### 3.2 fastpath 零开销检查（v1.1: C-S0 检查 + Error 码 -82）

fastpath 在每次 IPC 操作时检查 `ring->frozen`（C-S0 检查，详见 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md)），但使用 `unlikely` 标注确保正常路径（99%+ 不冻结）不触发分支预测失败：

```c
/* kernel/ipc/airy_uring_cmd.c —— fastpath C-S0 冻结检查（v1.1） */
static inline int airy_ipc_fastpath_check(struct airy_ipc_ring *ring)
{
    /* C-S0: unlikely 标注——99%+ 不冻结，分支预测器预测不跳转 */
    if (unlikely(READ_ONCE(ring->frozen))) {
        /* 异常路径：Ring 已冻结，返回 Error（不是 Fault）
         * AIRY_ECAP_FROZEN = -82，详见 08-sc-error-contract.md §2.4
         */
        return -AIRY_ECAP_FROZEN;   /* -82 */
    }
    return 0;
}

/* fastpath 入口 */
int airy_ipc_send_fastpath(struct airy_ipc_ring *ring, const void *payload,
                            size_t len)
{
    int ret = airy_ipc_fastpath_check(ring);   /* C-S0 */
    if (unlikely(ret))
        return ret;   /* 返回 Error -82，不进入正常发送路径 */

    /* 正常路径：C-S1~C-S12 校验链（含 C-S9 Badge 校验 ~10ns）+ reserve + memcpy + commit
     * FAST_SEND 总延迟 ~158ns，详见 170-performance/03-ipc-performance.md
     */
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
| fastpath 返回 Error | Micro-Supervisor（C-S0 检查） | 调用方收到 `AIRY_ECAP_FROZEN`（-82，Error 码，可恢复） |
| eventfd 通知 | Micro-Supervisor（冷酷执法） | Macro-Supervisor 被唤醒 |
| Macro-Supervisor 裁决 | Macro-Supervisor | 警告/降级/暂停/终止 |
| 解冻（裁决后恢复） | Macro-Supervisor | `ring->frozen = false` |

> **v1.1 区分 Error 与 Fault**：Ring 冻结是 Error（-82，可恢复），不是 Fault（不可恢复）。fastpath C-S0 返回 Error 让调用方决定重试或退出；Micro-Supervisor 的"冷酷执法"（Fault）仅在 Badge 伪造等不可恢复异常时触发。

### 3.5 FREEZE opcode（0x0005）ring 冻结语义（v1.1 新增）

**v1.1 新增**：A-IPC Capability Folding 引入 `FREEZE` opcode（0x0005），允许 Micro-Supervisor 或 Macro-Supervisor 通过 IPC 消息主动冻结目标 Ring，无需独立 syscall。这是 Capability Folding 单平面架构的体现——Ring 冻结也通过 io_uring 数据面完成，不引入额外控制面 syscall。

**FREEZE opcode 定义**（详见 [02-ipc-protocol.md §3](../30-interfaces/02-ipc-protocol.md) opcode 表）：

| opcode | 值 | 名称 | 用途 | 是否需要 Badge |
|--------|---|------|------|:---:|
| FREEZE | 0x0005 | 冻结 ring | `ring->frozen = true` | ✓（需 FREEZE 权限位） |

**FREEZE 权限位**：Badge Perms 字段的 bit 5（`AIRY_CAP_PERM_FREEZE` = 0x0020，详见 [03-capability-model.md §2.5](../110-security/03-capability-model.md)）。仅 sec_d 和 Macro-Supervisor 持有此权限位。Perms 16-bit 位掩码对齐 SSoT：`SEND=0x0001` / `RECV=0x0002` / `CALL=0x0004` / `GRANT=0x0008` / `REVOKE=0x0010` / `FREEZE=0x0020` / `BATCH=0x0040`。

**FREEZE 执行流程**：

```c
/* kernel/ipc/airy_uring_cmd.c —— FREEZE opcode 处理（v1.1） */
static int airy_ipc_handle_freeze(struct airy_ipc_cmd *cmd)
{
    struct airy_ipc_ring *ring;

    /* 1. C-S9 Badge 校验已通过（FREEZE 权限位已检查） */
    /* 2. 查找目标 Ring */
    ring = airy_ipc_ring_find(cmd->dst_task);
    if (!ring)
        return -AIRY_EIPC_CONTEXT;  /* -50 */

    /* 3. 原子设置 frozen 标志 */
    smp_store_release(&ring->frozen, true);
    ring->freeze_reason = AIRY_FAULT_ABNORMAL_CAP;  /* 0x1005 */
    ring->freeze_timestamp = ktime_get_real_ns();

    /* 4. eventfd 通知 Macro-Supervisor（非阻塞） */
    airy_eventfd_signal_fault(ring->ring_id, AIRY_FAULT_ABNORMAL_CAP);

    return 0;
}
```

**FREEZE 与冷酷执法的关系**：
- **冷酷执法**（§6）：fastpath C-S9 检测到 Badge 异常 → Micro-Supervisor 主动 FREEZE
- **策略冻结**：Macro-Supervisor 通过发送 FREEZE opcode 主动冻结 Ring（如维护、升级）
- **解冻协议**（v1.1 统一：与 [10-user-supervisor-daemon.md §8.3](10-user-supervisor-daemon.md) 严格一致）：Macro-Supervisor 通过 `AIRY_IPC_OP_UNFREEZE` opcode 向 sec_d 请求解冻，**sec_d 串行化解冻**（避免并发解冻导致状态不一致），解冻时执行 `atomic_inc(&airy_cap_global_epoch)`（Epoch 自增使旧 Badge 失效，强制 Agent 重新申请 Badge），最后通过 `ring->frozen = false` 解冻 Ring。sec_d 解冻全过程由 systemd watchdog 监控（`WatchdogSec=3`，超时触发 sec_d 重启 + 两阶段恢复，详见 [10-user-supervisor-daemon.md §4.7](10-user-supervisor-daemon.md)）。**禁止 Macro-Supervisor 直接设置 `ring->frozen = false`**（绕过 sec_d 串行化会破坏 Badge 一致性）。

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

    /* 1. 根据 die 原因映射到 Airymax Fault 码（v1.1: 对齐 08-sc-error-contract.md §3.1） */
    switch (val) {
    case DIE_OOPS:
        fault_code = AIRY_FAULT_VM_FAULT;       /* 0x1006 */
        break;
    case DIE_PAGE_FAULT:
        fault_code = AIRY_FAULT_VM_FAULT;       /* 0x1006 */
        break;
    default:
        fault_code = AIRY_FAULT_RING_CORRUPT;   /* 0x1003（v1.1: 原 AIRY_FAULT_IPC_FAULT 已废弃） */
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
| 内核 oops | `DIE_OOPS` | `AIRY_FAULT_VM_FAULT` (0x1006) | 冻结所有 Ring |
| 页错误 | `DIE_PAGE_FAULT` | `AIRY_FAULT_VM_FAULT` (0x1006) | 冻结相关 Ring |
| BUG_ON | `DIE_BUG` | `AIRY_FAULT_RING_CORRUPT` (0x1003) | 冻结所有 Ring |
| NMI watchdog | `DIE_NMIWATCHDOG` | `AIRY_FAULT_TIMEOUT` (0x1004) | 冻结所有 Ring |

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
/* kernel/superv/airy_cap_check.c —— 冷酷执法（v1.1） */
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

    /* 3. 返回 Fault 码（正数，区别于 Error 负数）
     * v1.1: 对齐 08-sc-error-contract.md §3.1
     *   AIRY_FAULT_CAP_FORGED   = 0x1001  Badge 伪造
     *   AIRY_FAULT_CAP_LEAK     = 0x1002  Badge 泄漏
     *   AIRY_FAULT_RING_CORRUPT = 0x1003  Ring 数据损坏
     *   AIRY_FAULT_TIMEOUT      = 0x1004  心跳超时
     *   AIRY_FAULT_ABNORMAL_CAP = 0x1005  Capability 树损坏
     *   AIRY_FAULT_VM_FAULT     = 0x1006  共享页映射损坏
     */
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

### 6.4 Badge 撤销解耦 atomic_inc（v1.1 新增）

**v1.1 新增**：Capability Folding 的 Badge 撤销通过 **1 行 `atomic_inc`** 立即失效所有已发出 Badge，与 IPC fastpath 完全解耦。这是 Capability Folding 单平面架构的核心设计——撤销操作不阻塞 fastpath，不 drain，不维护 bitmap，不发送 IPI。

**撤销机制**（详见 [03-capability-model.md §2.4](../110-security/03-capability-model.md)）：

```c
/* kernel/security/airy/capability.c —— Badge 撤销（v1.1） */

/* 全局 Epoch（1 个 atomic_t）
 * 撤销时 atomic_inc 立即失效所有已发出 Badge
 * Badge 中的 Epoch 字段（16-bit）与全局 Epoch 比对，不匹配即失效
 */
atomic_t airy_cap_global_epoch = ATOMIC_INIT(0);

/**
 * airy_cap_badge_revoke - 撤销指定 Agent 的所有 Badge
 * @agent_id: 待撤销的 Agent ID
 *
 * v1.1: 1 行 atomic_inc 完成全局撤销
 *
 * 工作原理:
 *   1. atomic_inc(&airy_cap_global_epoch) 使全局 Epoch +1
 *   2. 所有已发出 Badge 的 Epoch 字段（编译时快照）与新的全局 Epoch 不匹配
 *   3. fastpath C-S9 检测到 Epoch 不匹配，返回 -AIRY_ECAP_EPOCH (-79)
 *   4. Agent 必须重新向 sec_d 请求 Badge（CAP_REQUEST opcode）
 *
 * 性能特征:
 *   - 撤销延迟: ~1ns（1 行 atomic_inc）
 *   - fastpath 影响: 0（fastpath 仅多一次 atomic_read 比较，已在 C-S9 中）
 *   - drain 阻塞: 无
 *   - bitmap 维护: 无
 *   - IPI 广播: 无
 *
 * 对比 seL4:
 *   seL4 撤销需遍历 CSpace + 递归 revoke（O(n) 复杂度）
 *   Airymax 撤销 = 1 行 atomic_inc（O(1) 复杂度）
 *   代价: Agent 需重新申请 Badge（CAP_REQUEST 自举）
 */
void airy_cap_badge_revoke(uint32_t agent_id)
{
    /* 1 行代码完成全局 Badge 撤销 */
    atomic_inc(&airy_cap_global_epoch);

    /* 可选: 更新 agent_caps[agent_id].randtag 防止旧 Badge 复用
     * sec_d 重新编译 Badge 时会生成新的 Random Tag
     */
    WRITE_ONCE(agent_caps[agent_id].randtag,
               airy_cap_generate_randtag());

    /* 通知 Macro-Supervisor（非阻塞） */
    airy_eventfd_signal_fault(agent_id, AIRY_FAULT_ABNORMAL_CAP);
}
```

**撤销与 fastpath 的解耦**：

| 维度 | 传统 radix tree 撤销 | v1.1 atomic_inc 撤销 |
|------|---------------------|---------------------|
| 复杂度 | O(n) 遍历 CSpace | O(1) 1 行 atomic_inc |
| 阻塞 | drain + mdelay(5) | 无阻塞 |
| fastpath 影响 | IPI 缓存一致性开销 | 0（fastpath 仅多一次 atomic_read） |
| bitmap 维护 | 需维护撤销 bitmap | 无 bitmap |
| Agent 恢复 | 自动恢复 | 需重新申请 Badge（CAP_REQUEST） |

**设计哲学**：这是乔布斯/艾夫"简约就是美"在内核工程的落地——将复杂的撤销操作简化为 1 行 `atomic_inc`，代价转嫁给 Agent 重申请 Badge（CAP_REQUEST 自举）。这正是 37 号 §1.2 所说的"IPC 不是承载能力校验——IPC 就是能力校验"的逆向命题："撤销不是独立操作——撤销就是让 fastpath 自动拒绝"。

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
| `fault_set`（Fault 码） | A-UEF Fault 码 | `AIRY_FAULT_*` |
| `ntfn`（通知对象） | eventfd | 通知通道 |
| SC（Scheduling Context） | Agent 调度预算 | sched_tac 映射 |

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

### 8.1 性能 SLO（v1.1: 对齐 Capability Folding fastpath C-S9）

| 指标 | SLO | 实测 | 说明 |
|------|-----|------|------|
| C-S9 Badge 校验（fastpath 内联） | ≤20ns | ~10ns | v1.1: `airy_cap_badge_ok()` 3×READ_ONCE + 位运算 + 比较，详见 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md) |
| Badge 撤销（atomic_inc） | ≤5ns | ~1ns | v1.1: 1 行 `atomic_inc(&airy_cap_global_epoch)`，详见 §6.4 |
| FREEZE opcode 处理 | ≤500ns | ~200ns | v1.1: §3.5，含 Ring 查找 + smp_store_release + eventfd |
| 冻结操作（冷酷执法） | ≤500ns | ~200ns | §6.2，含 Ring 查找 + frozen 设置 + 时间戳 |
| eventfd 通知 | ≤100ns | ~50ns | §5.1，非阻塞 signal |
| 正常路径 frozen 检查开销（C-S0） | ≤2ns | ~1ns | §3.2，`unlikely(ring->frozen)` 分支预测优化 |
| FAST_SEND 总延迟（含 C-S9） | ≤200ns | ~158ns | 详见 [03-ipc-performance.md](../170-performance/03-ipc-performance.md) |

### 8.2 资源预算（v1.1: agent_caps[] 静态数组替代 radix tree）

| 资源 | 预算 | 实测 | 说明 |
|------|------|------|------|
| 内核静态内存 | ≤32KB | 16KB | v1.1: `agent_caps[1024]` 静态数组（1024 × 16B），sec_d 唯一写者，fastpath 多读者 READ_ONCE |
| 全局 Epoch | 4B | 4B | v1.1: 1 个 `atomic_t`，撤销 = 1 行 `atomic_inc`，详见 §6.4 |
| 事件队列 | 1024 条 | 1024 条 | eventfd 关联环形队列，溢出时仅 signal 计数 |
| CPU（fastpath） | ≤20ns | ~10ns | v1.1: 仅 C-S9 Badge 校验，正常路径不进 slowpath |
| CPU（slowpath） | 仅异常 | 仅异常 | v1.1: LSM 钩子仅在 C-S9 失败时被调用（§2.3） |

---

## §9 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §7 —— A-ULS 模块总纲
- [10-user-supervisor-daemon.md](10-user-supervisor-daemon.md) —— Macro-Supervisor（用户态温情裁决）
- [07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) —— 纯 C LSM 模块设计（Micro-Supervisor 的安全基础）
- [03-capability-model.md](../110-security/03-capability-model.md) —— Capability 模型（检测对象，v1.1 Badge 64-bit Native Word）
- [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) §3 —— opcode 表（v1.1 新增 FREEZE 0x0005）
- [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) §5.2 —— fastpath C-S9 Badge 校验实现（~10ns）
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— Error/Fault 码（v1.1: -78~-82 Badge 码 + 0x1001-0x1006 Fault 码）
- [11-unified-config.md](11-unified-config.md) §7 —— A-IPC Capability Folding 配置项（agent_caps 容量、Epoch 位宽等）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级（Macro-Supervisor 故障 fallback，v1.1 Badge=0 跳过 C-S9）
- [03-ipc-performance.md](../170-performance/03-ipc-performance.md) —— A-IPC 性能 SLO（FAST_SEND ~158ns / SLOW_SEND ~600ns-5.5μs）

---

## §10 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Micro-Supervisor 内核模块设计；纯 C LSM 检测非法 Capability（对齐 openEuler 纯 C 模式，不使用 BPF）；IPC 队列冻结（ring->frozen + unlikely 零开销）；die_notifier 内核崩溃通知链；eventfd 非阻塞故障通知；"内核冷酷执法"四步流程；借鉴 seL4 handleFault() 机制与策略分离 |
| v1.1 | 2026-07-18 | **Capability Folding 集成版**（A-IPC 第一块基石）：① §2.3 capability 校验路径重构——从独立前置 `airy_cap_check()` + radix tree 查找改为 fastpath C-S9 内联 Badge 校验（`airy_cap_badge_ok()`，~10ns），LSM 钩子仅在 slowpath（异常路径）做策略裁决；② §2.4 异常类型表对齐 Badge 错误码（-78~-82, 0x1001-0x1006）；③ §3.2 fastpath C-S0 返回 `AIRY_ECAP_FROZEN` (-82) 明确 Error vs Fault 区分；④ §3.5 新增 FREEZE opcode (0x0005) ring 冻结语义；⑤ §4.1/§4.2 Fault 码更新（`AIRY_FAULT_IPC_FAULT`→`AIRY_FAULT_RING_CORRUPT` 0x1003）；⑥ §6.2 `AIRY_FAULT_CAP_FAULT`→`AIRY_FAULT_CAP_FORGED` (0x1001) + 完整 Fault 码参考；⑦ §6.4 新增 Badge 撤销解耦 `atomic_inc` 机制（1 行代码 O(1) 撤销，无 drain/bitmap/IPI）；⑧ §8.1/§8.2 性能 SLO + 资源预算对齐 agent_caps[] 静态数组（16KB）+ atomic_t Epoch；⑨ §9 相关文档清除内部审查路径引用 |
| v1.1.1 | 2026-07-19 | **落后内容修复版**：① §2.3 修复 LSM 钩子重复校验——禁止 slowpath LSM 再次调用 `airy_cap_badge_ok()`，改为读取 `cmd->fastpath_ret` 执行冷酷执法（fastpath/slowpath 严格分工）；② §2.4 修复 MDB 残留——"MDB 派生链校验失败" 替换为 v1.1 `agent_caps[1024]` 静态数组 + Badge 64-bit 编码三段校验；③ §3.5 修复 FREEZE 权限位 bit 编号——`AIRY_BADGE_PERM_FREEZE` bit 4 → `AIRY_CAP_PERM_FREEZE` bit 5 (0x0020)，对齐 03-capability-model.md SSoT；④ §3.5 统一解冻协议——sec_d 串行化解冻 + Epoch 自增 + systemd watchdog（与 10-user-supervisor-daemon.md 严格一致）；⑤ §1.4 新增 12 daemon 完整协作关系；⑥ §1.5 新增 sched_tac 协作章节（sched_d + `airy_sys_sched_ctl` 编号 2 + Agent 8 态生命周期集成）；⑦ SSoT 声明新增 ADR-014 引用 + IRON-9 v3 四层模型引用；⑧ §2.3 新增 `AIRY_ESEC_D_THROTTLED` (-83) sec_d 限流错误码处理 + §2.4 异常类型表扩展两行上游检测路径（`AIRY_FAULT_URING_MALFORMED` 0x100A io_uring SQE 畸变 + `AIRY_ESEC_D_THROTTLED` -83 sec_d 限流），完整覆盖 Micro-Supervisor 可能涉及的异常类型 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Micro-Supervisor 内核模块设计 | v1.1.1 | 2026-07-19
