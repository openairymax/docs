Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 纯 C LSM 模块设计
> **文档定位**：Airymax 纯 C LSM（Linux Security Module）模块的唯一权威详细设计，对齐 openEuler 纯 C LSM 模式\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7 + [01-lsm-framework.md](01-lsm-framework.md)\
> **设计依据**：本文件为 SSoT（Single Source of Truth），纯 C LSM 模块职责、fastpath/slowpath 职责分割、sec_d Badge 编译职责、Capability 校验钩子均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **纯 C LSM 模块设计** 的唯一权威源。`DEFINE_LSM(airy)` 纯 C 注册、**v1.1 fastpath C-S9 + slowpath LSM 职责分割**（fastpath 内联 Badge 校验，LSM 钩子仅 slowpath 接管）、**sec_d Badge 编译职责**（sec_d 是 Badge 的唯一写者，编译 Badge = `Epoch<<48 | RandomTag<<16 | Perms`）、io_uring 安全钩子（opcode 白名单校验）、与 seL4 Capability 模型的关系、纯 C 性能优势（无 BPF 间接调用开销）、源码参考（OLK 6.6 security/ 目录）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Airymax LSM 模块结构与 fastpath/slowpath 职责分割。
>
> **v1.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.1 起，纯 C LSM 的 capability 校验路径从"独立前置 `airy_cap_check()` + radix tree 查找"重构为 **fastpath C-S9 内联 Badge 校验**（`airy_cap_badge_ok()`，~10ns，详见 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md)）。纯 C LSM 不再在 `security_uring_cmd` 钩子中独立执行 capability 校验——fastpath C-S9 已在 io_uring 数据面完成 Badge 校验，LSM 钩子仅在 slowpath（异常路径）做策略裁决与冷酷执法。**sec_d 是 Badge 的唯一写者**——sec_d 编译 Badge 时使用全局 Epoch + 随机生成的 32-bit Random Tag + Perms 位掩码，写入 `agent_caps[agent_id]` 静态数组。Badge 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码立即失效所有已发出 Badge（详见 [09-kernel-agent-supervisor.md §6.4](../20-modules/09-kernel-agent-supervisor.md)）。
>
> 技术选型声明：安全采用 **纯 C LSM 模块**（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式：SELinux/AppArmor/Landlock/Tomoyo 全部纯 C）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：安全模块开发者、内核开发者、A-ULS/A-IPC 模块开发者
- **前置知识**：理解 [01-lsm-framework.md](01-lsm-framework.md) LSM 框架、[03-capability-model.md](03-capability-model.md) Capability 模型、[06-io-uring-hardening.md](06-io-uring-hardening.md) io_uring 安全加固、seL4 capability-based security
- **预计阅读时间**：35 分钟
- **核心概念**：纯 C LSM、DEFINE_LSM、Capability 校验钩子、io_uring 安全钩子、seL4 Capability、BPF LSM 对比
- **复杂度标识**：高级

---

## §1 设计目标：对齐 openEuler 纯 C LSM 模式（SELinux/Landlock 纯 C，不用 BPF LSM）

### 1.1 为什么用纯 C LSM

Airymax 选择纯 C LSM 模块（**不使用 BPF LSM**），对齐 openEuler（OLK 6.6）纯 C 模式。原因有四：

| 维度 | 纯 C LSM（Airymax 选择） | BPF LSM | 说明 |
|------|------------------------|---------|------|
| 性能 | 直接函数调用，~ns 级 | BPF 虚拟机间接调用，~10ns+ | 纯 C 无间接开销 |
| 安全 | 编译期校验，源码可见 | BPF 字节码运行时验证 | 纯 C 无注入风险 |
| 可审计 | 源码静态分析 | BPF 字节码需反编译 | 纯 C 更透明 |
| 复杂度 | 简单（C 函数） | 复杂（BPF 程序 + verifier） | 纯 C 更易维护 |

### 1.2 openEuler 纯 C 模式参考

openEuler（OLK 6.6）的 `security/` 目录中，所有主流 LSM 模块均为纯 C 实现：

| LSM 模块 | 实现语言 | 源码位置 | Airymax 借鉴点 |
|---------|---------|---------|---------------|
| SELinux | 纯 C | `security/selinux/` | 策略引擎设计 |
| AppArmor | 纯 C | `security/apparmor/` | 路径-based 权限 |
| Landlock | 纯 C | `security/landlock/` | 用户态可配置沙箱 |
| Tomoyo | 纯 C | `security/tomoyo/` | 学习模式 |
| **Airymax** | **纯 C** | `security/airy/`（新增） | **Capability-based** |

Airymax 的纯 C LSM 模块是 OLK 6.6 `security/` 目录的新增成员，与 SELinux/AppArmor/Landlock/Tomoyo 并列，全部纯 C 实现，不使用 BPF LSM。

### 1.3 设计目标（v1.1: fastpath/slowpath 职责分割）

纯 C LSM 模块的核心设计目标：

1. **Capability 校验**（v1.1 职责分割）：fastpath C-S9 内联 Badge 校验（~10ns）+ slowpath LSM 钩子接管异常（冷酷执法）
2. **io_uring 安全**：opcode 白名单校验（§4）
3. **LSM_ORDER_FIRST**：永远第一个执行，确保 Capability 优先
4. **零 BPF 依赖**：纯 C 实现，不依赖 BPF 虚拟机
5. **sec_d Badge 编译**（v1.1 新增）：sec_d 是 Badge 的唯一写者，编译 Badge = `Epoch<<48 | RandomTag<<16 | Perms`，写入 `agent_caps[agent_id]` 静态数组

---

## §2 DEFINE_LSM(airy)：纯 C LSM 模块注册

### 2.1 LSM 注册机制

Airymax 使用 `DEFINE_LSM` 宏注册纯 C LSM 模块。这是 Linux 6.6 标准的 LSM 注册方式，与 SELinux/AppArmor/Landlock 一致：

```c
/* security/airy/airy_lsm.c —— 纯 C LSM 模块注册 */
#include <linux/lsm_hooks.h>
#include <linux/module.h>
#include "airy_internal.h"

/* Airymax LSM 模块信息 */
DEFINE_LSM(airy) = {
    .name = "airy",
    .order = LSM_ORDER_FIRST,   /* 永远第一个执行 */
    .enabled = &airy_enabled,
    .init = airy_lsm_init,
    .blobs = &air_blob_sizes,   /* LSM blob 大小声明 */
};

/* 模块加载时初始化（v1.1: agent_caps[] 静态数组替代 radix tree） */
static int __init airy_lsm_init(void)
{
    /* 1. 初始化 agent_caps[] 静态数组 + 全局 Epoch（v1.1 替代 radix tree）
     * agent_caps[1024] 是 16KB 内核静态数组，sec_d 唯一写者
     * fastpath C-S9 多读者 READ_ONCE 访问，无锁
     */
    airy_cap_agent_caps_init();
    atomic_set(&airy_cap_global_epoch, 0);

    /* 2. 注册安全钩子（纯 C 函数指针，非 BPF）
     * v1.1: uring_cmd 钩子仅在 slowpath 被 C-S9 失败时调用
     */
    security_add_hooks(airy_hooks, ARRAY_SIZE(airy_hooks), "airy");

    /* 3. 初始化 IPC Ring 冻结机制 */
    airy_ipc_freeze_init();

    pr_info("airy: pure-C LSM loaded (no BPF), order=FIRST, v1.1 Capability Folding\n");
    return 0;
}

/* 模块参数：可通过 sysctl 控制启用/禁用 */
static bool airy_enabled = true;
module_param(airy_enabled, bool, 0644);
MODULE_PARM_DESC(airy_enabled, "Enable/disable Airymax pure-C LSM");
```

### 2.2 LSM_ORDER_FIRST 的意义

`LSM_ORDER_FIRST` 确保 Airymax LSM 永远第一个执行，在任何其他 LSM（SELinux/AppArmor/Landlock）之前：

| 执行顺序 | LSM 模块 | 职责 |
|---------|---------|------|
| 1（第一个） | **Airymax** | **Capability 校验（冷酷执法）** |
| 2 | SELinux | 策略强制 |
| 3 | AppArmor | 路径权限 |
| 4 | Landlock | 用户态沙箱 |

这确保 Capability 校验优先——若 Capability 不通过，后续 LSM 不执行，直接返回 Fault。这是借鉴 seL4 的设计：安全检查在最内层。

### 2.3 LSM blob 声明

Airymax LSM 使用 LSM blob 机制存储每对象的安全数据：

```c
/* security/airy/airy_internal.h —— LSM blob 声明 */
struct airy_task_sec {
    __u32 cap_space_root;     /* Agent 的 CSpace 根 */
    __u32 agent_state;        /* Agent 8 态 */
    __u64 sched_budget_ns;    /* 调度预算（sched_tac） */
};

struct airy_inode_sec {
    __u32 cap_required;       /* 访问此 inode 所需 Capability */
};

/* blob 大小声明 */
struct lsm_blob_sizes air_blob_sizes __lsm_ro_after_init = {
    .lbs_task = sizeof(struct airy_task_sec),
    .lbs_inode = sizeof(struct airy_inode_sec),
};
```

---

## §3 Capability 校验钩子：v1.1 fastpath C-S9 + slowpath LSM（职责分割）

### 3.1 v1.1 架构：fastpath/slowpath 职责分割

**v1.1 重构**：Capability 校验从"独立前置 `airy_cap_check()` + radix tree 查找"重构为 **fastpath C-S9 内联 Badge 校验** + **slowpath LSM 钩子接管**。这是 Capability Folding 单平面架构的核心设计——IPC 不是承载能力校验，IPC 就是能力校验。

| 路径 | 执行位置 | 校验内容 | 延迟 | 触发频率 |
|------|---------|---------|------|---------|
| **fastpath C-S9** | io_uring 数据面内联 | Badge 64-bit Native Word 校验（Epoch + RandomTag + Perms） | ~10ns | 每次 IPC（99%+ 通过） |
| **slowpath LSM** | `security_uring_cmd` 钩子 | 策略裁决 + 冷酷执法（冻结 Ring + 通知 Macro-Supervisor） | ~100ns+ | 仅 C-S9 失败时（<1%） |

### 3.2 fastpath C-S9 内联 Badge 校验（~10ns）

fastpath C-S9 在 io_uring 数据面内联执行 Badge 校验，**不进入 LSM 钩子链**。Badge 是 64-bit Native Word：`(Epoch<<48 | RandomTag<<16 | Perms)`，校验仅需 3 次 READ_ONCE + 位运算 + 比较：

```c
/* kernel/ipc/airy_uring_cmd.c —— fastpath C-S9 Badge 校验（v1.1 内联，~10ns）
 * 详见 30-interfaces/07-ipc-fastpath.md §5.2
 */
static __always_inline int airy_cap_badge_ok(struct task_struct *task,
                                              __u64 capability_badge,
                                              __u16 opcode)
{
    __u32 agent_id = task->pid;
    __u32 global_epoch, randtag, perms;

    /* 1. 三次 READ_ONCE 读取 agent_caps[agent_id]（无锁，多读者安全）
     * agent_caps[] 是 16KB 静态数组，sec_d 唯一写者
     */
    global_epoch = READ_ONCE(airy_cap_global_epoch.counter);
    randtag = READ_ONCE(agent_caps[agent_id].randtag);
    perms = READ_ONCE(agent_caps[agent_id].perms);

    /* 2. Badge 解码（位运算，~1ns） */
    __u64 badge_epoch = (capability_badge >> 48) & 0xFFFF;
    __u64 badge_randtag = (capability_badge >> 16) & 0xFFFFFFFF;
    __u64 badge_perms = capability_badge & 0xFFFF;

    /* 3. C-S9a: Epoch 校验（badge_epoch == global_epoch?） */
    if (unlikely(badge_epoch != global_epoch))
        return -AIRY_ECAP_EPOCH;   /* -79: Badge 已撤销/过期 */

    /* 4. C-S9b: RandomTag 校验（防伪造） */
    if (unlikely(badge_randtag != randtag))
        return -AIRY_ECAP_FORGED;  /* -80: Badge 伪造，触发 Fault 0x1001 */

    /* 5. C-S9c: Perms 校验（badge_perms & required == required?） */
    __u16 required = airy_op_required_perms(opcode);
    if (unlikely((badge_perms & required) != required))
        return -AIRY_ECAP_PERM;    /* -81: 权限位不满足 */

    return 0;   /* Badge 校验通过，放行 */
}
```

### 3.3 slowpath LSM 钩子：security_uring_cmd（仅 C-S9 失败时）

`security_uring_cmd` 钩子在 v1.1 中**仅 slowpath 被 C-S9 失败时调用**，做策略裁决与冷酷执法。这是 Micro-Supervisor 的核心实现，详见 [09-kernel-agent-supervisor.md §2.3](../20-modules/09-kernel-agent-supervisor.md)：

```c
/* security/airy/airy_cap_hooks.c —— v1.1 slowpath LSM 钩子（fastpath C-S9 异常接管）
 * 纯 C 实现，不使用 BPF
 */
static int airy_uring_cmd_check(struct io_uring_cmd *ioucmd,
                                  unsigned int issue_flags)
{
    struct airy_ipc_cmd *cmd = (struct airy_ipc_cmd *)ioucmd->cmd;
    int fastpath_ret;

    /* v1.1: fastpath C-S9 已在 io_uring 数据面完成 Badge 校验
     * LSM 钩子仅在 slowpath（异常路径）被调用
     * 详见 07-ipc-fastpath.md §5.2 C-S9 实现
     */

    /* 1. 调用 fastpath C-S9 Badge 校验（内联函数，~10ns） */
    fastpath_ret = airy_cap_badge_ok(cmd->src_task, cmd->capability_badge,
                                      cmd->opcode);
    if (likely(fastpath_ret == 0))
        return 0;   /* Badge 校验通过，放行 */

    /* 2. slowpath: Badge 校验失败，Micro-Supervisor 接管 */
    switch (fastpath_ret) {
    case -AIRY_ECAP_FORGED:     /* -80: Badge 伪造，触发 Fault */
        return airy_fault_enforce(AIRY_FAULT_CAP_FORGED, cmd);  /* 0x1001 */

    case -AIRY_ECAP_EPOCH:      /* -79: Epoch 不匹配（已撤销或过期） */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */

    case -AIRY_ECAP_PERM:       /* -81: 权限位不满足 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */

    case -AIRY_ECAP_FROZEN:     /* -82: Ring 已冻结 */
        return fastpath_ret;    /* Error，不是 Fault */

    case -AIRY_ECAP_BADGE:      /* -78: Badge 格式无效 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */

    default:
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);  /* 0x1005 */
    }
}
```

### 3.4 Capability 校验流程（v1.1 fastpath C-S9 + slowpath LSM）

```
io_uring_cmd 提交（IORING_OP_URING_CMD）
       │
       ▼
fastpath C-S0: Ring 冻结检查（unlikely(ring->frozen)）
       ├── 冻结 ────────────────▶ 返回 -AIRY_ECAP_FROZEN (-82, Error)
       └── 未冻结
       ▼
fastpath C-S9: Badge 64-bit Native Word 校验（~10ns 内联）
       │
       ├── C-S9a: Epoch 校验（badge_epoch == global_epoch?）
       │   └── 不匹配 ─────────▶ 返回 -AIRY_ECAP_EPOCH (-79)
       │
       ├── C-S9b: RandomTag 校验（防伪造）
       │   └── 不匹配 ─────────▶ 返回 -AIRY_ECAP_FORGED (-80, 触发 Fault 0x1001)
       │
       └── C-S9c: Perms 校验（badge_perms & required == required?）
           └── 不满足 ─────────▶ 返回 -AIRY_ECAP_PERM (-81)
       ▼
fastpath C-S9 通过（99%+ 正常路径）
       │
       ▼
正常路径：C-S1~C-S8, C-S10~C-S12 校验链 + Ring Buffer 写入
       │
       ▼
FAST_SEND 完成（~158ns 总延迟）

─── 异常路径（C-S9 失败，<1%）───
       ▼
slowpath LSM: security_uring_cmd 钩子被调用
       │
       ▼
switch(fastpath_ret):
       ├── -AIRY_ECAP_FORGED   → airy_fault_enforce(AIRY_FAULT_CAP_FORGED)   [0x1001]
       ├── -AIRY_ECAP_EPOCH    → airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP) [0x1005]
       ├── -AIRY_ECAP_PERM     → airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP) [0x1005]
       ├── -AIRY_ECAP_FROZEN   → 返回 Error（不触发 Fault）
       └── -AIRY_ECAP_BADGE    → airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP) [0x1005]
       ▼
冷酷执法：冻结 Ring + eventfd 通知 Macro-Supervisor
```

### 3.5 sec_d Badge 编译职责（v1.1 新增）

**v1.1 新增**：sec_d（security daemon）是 Badge 的**唯一写者**。sec_d 编译 Badge 时使用全局 Epoch + 随机生成的 32-bit Random Tag + Perms 位掩码，写入 `agent_caps[agent_id]` 静态数组。这是 Capability Folding 单平面架构的关键约束——fastpath C-S9 仅做读操作（READ_ONCE），sec_d 独占写操作（WRITE_ONCE）。

```c
/* services/daemons/sec_d/badge_compile.c —— sec_d Badge 编译（v1.1）
 * sec_d 是 agent_caps[] 静态数组的唯一写者
 */
__u64 airy_cap_badge_compile(__u32 agent_id, __u16 perms)
{
    __u32 epoch, randtag;
    __u64 badge;

    /* 1. 读取全局 Epoch（atomic_read，无锁） */
    epoch = atomic_read(&airy_cap_global_epoch);

    /* 2. 生成 32-bit 随机 Random Tag（防伪造）
     * 使用 get_random_bytes_wait() 确保密码学安全
     */
    get_random_bytes_wait(&randtag, sizeof(randtag));

    /* 3. 编译 Badge 64-bit Native Word */
    badge = ((__u64)epoch << 48) | ((__u64)randtag << 16) | (__u64)perms;

    /* 4. WRITE_ONCE 写入 agent_caps[agent_id]（唯一写者） */
    WRITE_ONCE(agent_caps[agent_id].randtag, randtag);
    WRITE_ONCE(agent_caps[agent_id].perms, perms);

    /* 5. 审计日志（通过 A-IPC 上报 Audit 子系统） */
    airy_audit_emit(AGENT_CAP_COMPILE, agent_id, badge);

    return badge;   /* 返回 Badge 给 Agent（用户态持有 opaque handle） */
}
```

**sec_d 写者约束**：

| 约束 | 说明 |
|------|------|
| 唯一写者 | 仅 sec_d 可写 `agent_caps[agent_id]`，其他内核路径只读 |
| WRITE_ONCE | 所有写操作必须使用 `WRITE_ONCE`，确保 fastpath C-S9 的 READ_ONCE 可见性 |
| Random Tag 强制 | 每次 Badge 编译必须生成新的 32-bit Random Tag，禁止复用 |
| Epoch 同步 | Badge 中的 Epoch 必须与 `airy_cap_global_epoch` 一致（atomic_read） |
| Perms 位掩码 | Perms 必须是已定义的 `AIRY_BADGE_PERM_*` 位组合（详见 [03-capability-model.md §2.5](03-capability-model.md)） |

### 3.6 其他 Capability 钩子（非 IPC 路径）

除 `security_uring_cmd` 外，Airymax 还注册以下 Capability 校验钩子（这些钩子不在 IPC fastpath 上，依然走传统 LSM 路径）：

| 钩子 | 触发场景 | 校验内容 | v1.1 变更 |
|------|---------|---------|----------|
| `security_uring_cmd` | io_uring_cmd 提交 | Badge 校验（核心，IPC fastpath） | v1.1: 仅 slowpath，fastpath C-S9 内联 |
| `security_uring_sqe` | SQE 提交 | opcode 白名单 | 无变更 |
| `security_uring_register_buffers` | buffer 注册 | CAP_REGISTER_BUFFER 权限 | 无变更 |
| `security_task_kill` | kill 信号 | CAP_KILL 权限 | 无变更 |
| `security_file_open` | 文件打开 | CAP_FILE_OPEN 权限 | 无变更 |

---

## §4 io_uring 安全钩子：io_uring_cmd opcode 白名单校验

### 4.1 io_uring 安全钩子集

Airymax 纯 C LSM 注册一组 io_uring 安全钩子，实现 opcode 白名单与 buffer 注册校验（详见 [06-io-uring-hardening.md](06-io-uring-hardening.md)）：

```c
/* security/airy/airy_uring_hooks.c —— io_uring 安全钩子 */
/* opcode 白名单 */
static const unsigned int airy_uring_allowed_opcodes[] = {
    IORING_OP_URING_CMD,   /* IPC 语义映射 */
    IORING_OP_NOP,         /* 基础测试 */
};

/* SQE 提交时 opcode 白名单校验 */
static int airy_uring_sqe_check(struct io_kiocb *req, unsigned int opcode)
{
    for (size_t i = 0; i < ARRAY_SIZE(airy_uring_allowed_opcodes); i++) {
        if (opcode == airy_uring_allowed_opcodes[i])
            return 0;  /* 白名单允许 */
    }
    /* 非法 opcode：拒绝 */
    pr_warn_ratelimited("airy: blocked io_uring opcode %u pid=%d\n",
                        opcode, current->pid);
    return -EOPNOTSUPP;
}

/* buffer 注册时 Capability 校验 */
static int airy_uring_register_buffers_check(struct io_ring_ctx *ctx,
                                               void __user *addr,
                                               unsigned long size)
{
    /* 1. Capability 校验 */
    if (airy_cap_check_fastpath(current_cap_id(),
                                 AIRY_CAP_REGISTER_BUFFER) < 0)
        return -AIRY_ECAP_PERM;

    /* 2. 地址与大小校验 */
    if (!access_ok(addr, size) || size > AIRY_URING_MAX_BUFFER_SIZE)
        return -EINVAL;

    return 0;
}
```

### 4.2 钩子注册

```c
/* security/airy/airy_lsm.c —— 钩子注册 */
static struct security_hook_list airy_hooks[] __lsm_ro_after_init = {
    /* Capability 校验钩子 */
    LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_check),
    LSM_HOOK_INIT(task_kill, airy_task_kill_check),
    LSM_HOOK_INIT(file_open, airy_file_open_check),

    /* io_uring 安全钩子 */
    LSM_HOOK_INIT(uring_sqe, airy_uring_sqe_check),
    LSM_HOOK_INIT(uring_register_buffers, airy_uring_register_buffers_check),

    /* 进程安全钩子 */
    LSM_HOOK_INIT(task_alloc, airy_task_alloc),
    LSM_HOOK_INIT(task_free, airy_task_free),
};
```

### 4.3 `__lsm_ro_after_init` 优化

钩子数组使用 `__lsm_ro_after_init` 标记，确保初始化后只读——这利用 Linux 内核的 ro_after_init 机制，将钩子表放入只读内存段，防止运行时篡改：

| 优化 | 效果 | 安全价值 |
|------|------|---------|
| `__lsm_ro_after_init` | 钩子表只读 | 防止内核漏洞利用篡改钩子 |
| `static const` 白名单 | opcode 白名单只读 | 防止运行时注入非法 opcode |
| `rcu_read_lock` | 无锁读取 | 性能 + 并发安全 |

---

## §5 与 seL4 Capability 模型的关系：借鉴 seL4 capability-based security（v1.1: Badge 64-bit Native Word）

### 5.1 seL4 Capability 模型

seL4 微内核采用 capability-based security——每个安全敏感操作需要对应的 Capability 令牌（详见 [03-capability-model.md](03-capability-model.md)）。Airymax 纯 C LSM 借鉴此模型：

| seL4 机制 | Airymax 纯 C LSM 对齐（v1.1） | 说明 |
|-----------|---------------------|------|
| CNode（Capability Node） | `struct airy_cap_entry`（agent_caps[] 数组项） | v1.1: Capability 存储（替代 airy_cnode） |
| CSpace（Capability Space） | `agent_caps[1024]` 静态数组（16KB） | v1.1: 替代 radix tree |
| mint/copy/move/revoke | `airy_cap_badge_compile/revoke`（sec_d 编译 + atomic_inc 撤销） | v1.1: sec_d 唯一写者，撤销 = 1 行 atomic_inc |
| badge（权限掩码） | Badge 64-bit Native Word（`Epoch<<48 \| RandomTag<<16 \| Perms`） | v1.1: 完整 64-bit Badge，含 Epoch + RandomTag + Perms |
| handleFault() | `airy_fault_enforce()` | Fault 处理（冷酷执法） |

### 5.2 借鉴的关键设计

| seL4 设计 | Airymax 实现（v1.1） | 价值 |
|-----------|------------|------|
| Capability 不可伪造 | sec_d 编译 Badge 时生成 32-bit 随机 Random Tag，fastpath C-S9b 校验 | 防伪造（~10ns 校验） |
| 最小权限 | Perms 16-bit 位掩码，每操作校验 required perms | 最小授权 |
| 递归撤销 | `atomic_inc(&airy_cap_global_epoch)` 一行代码立即失效所有已发出 Badge | O(1) 撤销，无 drain/bitmap/IPI |
| 形式化验证 | 参考验证方法（未完全形式化） | 可信基础 |

### 5.3 与 seL4 的差异（v1.1: agent_caps[] 静态数组）

| 维度 | seL4 | Airymax（v1.1） | 适配原因 |
|------|------|---------|---------|
| 执行域 | 微内核（纯内核态） | Linux 6.6（LSM 钩子 slowpath + fastpath 内联） | 复用主线 LSM + io_uring 数据面 |
| Capability 存储 | CSpace（内核对象） | `agent_caps[1024]` 静态数组（16KB） | v1.1: 替代 radix tree，无锁多读者 |
| 校验方式 | 内核内联 | fastpath C-S9 内联（~10ns） + slowpath LSM | v1.1: fastpath/slowpath 职责分割 |
| 撤销机制 | 递归遍历 CSpace（O(n)） | `atomic_inc` 全局 Epoch（O(1)） | v1.1: 1 行代码完成全局撤销 |
| 形式化验证 | 完全形式化 | 未形式化（CBMC 全函数验证 fastpath） | 工程权衡 |

---

## §6 性能：纯 C 无 BPF 间接调用开销（v1.1: fastpath C-S9 ~10ns）

### 6.1 性能对比（v1.1: fastpath C-S9 Badge 校验 ~10ns）

纯 C LSM 相比 BPF LSM 的性能优势在于无间接调用开销。v1.1 起 fastpath C-S9 Badge 校验降至 ~10ns（原 ~160ns），整体 FAST_SEND 降至 ~158ns（原 ~320ns）：

| 维度 | 纯 C LSM（v1.1） | BPF LSM | 差距 |
|------|---------|---------|------|
| 调用方式 | 直接函数调用（fastpath 内联） | BPF 虚拟机间接调用 | — |
| fastpath 延迟 | ~10ns（C-S9 Badge 校验） | ~200ns+ | ~20x |
| 内存开销 | 16KB 静态数组（agent_caps[]） | BPF 程序 + map | ~3x |
| 编译开销 | 编译期 | 运行时 JIT | — |

### 6.2 fastpath C-S9 性能拆解（v1.1）

v1.1 fastpath C-S9 Badge 校验的性能拆解（无 RCU 锁、无 radix tree 查找）：

| 步骤 | 操作 | 耗时 |
|------|------|------|
| 1. READ_ONCE × 3 | 读取 global_epoch + randtag + perms | ~3ns |
| 2. Badge 解码 | 位运算（位移 + 位与） | ~1ns |
| 3. C-S9a Epoch 校验 | 比较运算 | ~1ns |
| 4. C-S9b RandomTag 校验 | 比较运算（unlikely） | ~1ns（预测成功） |
| 5. C-S9c Perms 校验 | 位运算 + 比较 | ~2ns |
| 6. 分支预测 | `unlikely` 优化 | ~0ns（预测成功） |
| **总计** | | **~8-10ns** |

### 6.3 与 IPC fastpath 的关系（v1.1: FAST_SEND ~158ns）

v1.1 fastpath C-S9 是 IPC fastpath 的一部分，总延迟在 SLO 内：

| 组件 | 延迟 | 占比 |
|------|------|------|
| C-S0 Ring 冻结检查 | ~1ns | ~1% |
| C-S1~C-S8 其他校验 | ~30ns | ~19% |
| **C-S9 Badge 校验（v1.1）** | **~10ns** | **~6%** |
| C-S10~C-S12 其他校验（含 CRC32） | ~17ns | ~11% |
| Ring Buffer 写入（reserve + memcpy + commit） | ~100ns | ~63% |
| **FAST_SEND 总计** | **~158ns** | **100%** |

### 6.4 性能优化手段（v1.1）

| 优化 | 效果 | 实现位置 |
|------|------|---------|
| `unlikely` 分支预测 | 正常路径零开销 | fastpath C-S0/C-S9 检查 |
| READ_ONCE 无锁读取 | 无 RCU 锁开销 | fastpath C-S9 Badge 校验 |
| `agent_caps[]` 静态数组 | 无 radix tree 查找开销（~100ns → ~3ns） | fastpath C-S9 |
| `__lsm_ro_after_init` | 只读内存，TLB 友好 | LSM 钩子表 |
| `__always_inline` | fastpath C-S9 强制内联，无函数调用开销 | `airy_cap_badge_ok()` |
| `static_key` 跳过 | 99%+ 正常请求跳过 slowpath | 第一层轻量标记 |

---

## §7 源码参考：OLK 6.6 security/ 目录（SELinux/AppArmor/Landlock/Tomoyo 全部纯 C）

### 7.1 OLK 6.6 security/ 目录结构

Airymax 纯 C LSM 模块的源码结构对齐 OLK 6.6（openEuler Linux Kernel 6.6）的 `security/` 目录：

```
security/
├── airy/                    ← Airymax 新增（纯 C）
│   ├── airy_lsm.c           LSM 注册（DEFINE_LSM）
│   ├── airy_cap_hooks.c     v1.1: slowpath Capability 校验钩子（fastpath C-S9 异常接管）
│   ├── airy_uring_hooks.c   io_uring 安全钩子
│   ├── airy_capability.c    v1.1: agent_caps[] 静态数组 + 全局 Epoch + Badge 撤销（替代 airy_cap_tree.c）
│   ├── airy_internal.h      内部头文件
│   └── Kconfig              配置
├── selinux/                 SELinux（纯 C 参考）
├── apparmor/                AppArmor（纯 C 参考）
├── landlock/                Landlock（纯 C 参考）
├── tomoyo/                  Tomoyo（纯 C 参考）
├── commoncap.c              POSIX capability（纯 C）
└── lsm.c                    LSM 框架
```

### 7.2 借鉴的纯 C 模式（v1.1: agent_caps[] 替代 radix tree）

从 OLK 6.6 `security/` 目录借鉴的纯 C 模式：

| 借鉴源 | 借鉴模式 | Airymax 应用（v1.1） |
|--------|---------|-------------|
| `security/selinux/hooks.c` | `DEFINE_LSM` + `security_add_hooks` | LSM 注册 |
| `security/selinux/ss/policydb.c` | 策略数据库 | v1.1: `agent_caps[1024]` 静态数组（替代 radix tree，16KB 无锁多读者） |
| `security/landlock/setup.c` | `LSM_ORDER_FIRST` | 永远第一个执行 |
| `security/apparmor/path.c` | 路径权限 | 文件访问 Capability |
| `security/tomoyo/common.h` | 学习模式 | Capability 授权（可选） |
| `security/commoncap.c` | POSIX cap 检查 | 与 POSIX capability 共存 |

### 7.3 Kconfig 配置

```kconfig
# security/airy/Kconfig —— Airymax LSM 配置
config SECURITY_AIRY
    bool "Airymax pure-C LSM (Capability-based, no BPF)"
    depends on SECURITY
    depends on IO_URING
    default y
    help
      Airymax pure-C LSM module for Capability-based security.
      Aligns with openEuler pure-C LSM pattern (SELinux/AppArmor/
      Landlock/Tomoyo all pure-C, no BPF LSM).

      This LSM provides:
      - Capability verification in io_uring_cmd callback
      - io_uring opcode whitelist
      - registered buffer security check
      - seL4-style capability-based security

      Say Y here to enable Airymax pure-C LSM.
```

### 7.4 Makefile

```makefile
# security/airy/Makefile（v1.1: airy_capability.o 替代 airy_cap_tree.o）
obj-$(CONFIG_SECURITY_AIRY) := airy_lsm.o airy_cap_hooks.o \
                               airy_uring_hooks.o airy_capability.o
```

---

## §8 与其他安全模块的协作

### 8.1 LSM 共存

Airymax 纯 C LSM 与其他 LSM 共存，执行顺序为 `LSM_ORDER_FIRST`：

| 顺序 | LSM | 执行内容 | Airymax 关系 |
|------|-----|---------|-------------|
| 1 | Airymax | Capability 校验 | 最先执行，冷酷执法 |
| 2 | SELinux | 策略强制 | Airymax 通过后才执行 |
| 3 | AppArmor | 路径权限 | 独立检查 |
| 4 | Landlock | 用户态沙箱 | 独立检查 |

### 8.2 与 POSIX capability 共存

Airymax Capability（seL4 风格）与 POSIX capability 共存（详见 [03-capability-model.md](03-capability-model.md) §1.2）：

| 类型 | 来源 | 粒度 | Airymax LSM 关系 |
|------|------|------|----------------|
| seL4 风格 Capability | Airymax | 令牌级（细粒度） | 核心校验 |
| POSIX capability | Linux | 进程级（41 个 cap） | 兼容共存 |

Airymax LSM 在 `security_capable` 钩子中检查 POSIX capability，在 `security_uring_cmd` 钩子中检查 seL4 风格 Capability，二者互补。

---

## §9 测试与验证（v1.1: 对齐 Badge 错误码）

### 9.1 测试场景

| 测试场景 | 验证内容 | 预期结果 |
|---------|---------|---------|
| 合法 Badge 提交 io_uring_cmd | C-S9 通过 | 正常执行（FAST_SEND ~158ns） |
| 伪造 Badge 提交（RandomTag 不匹配） | C-S9b 失败 | `-AIRY_ECAP_FORGED` (-80) → `AIRY_FAULT_CAP_FORGED` (0x1001) |
| Epoch 不匹配提交（已撤销 Badge） | C-S9a 失败 | `-AIRY_ECAP_EPOCH` (-79) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |
| 权限不足提交（Perms 不满足） | C-S9c 失败 | `-AIRY_ECAP_PERM` (-81) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |
| 已冻结 Ring 提交 | C-S0 失败 | `-AIRY_ECAP_FROZEN` (-82, Error，不触发 Fault） |
| Badge 格式无效提交（CAP_CARRY 但 badge=0） | C-S9 失败 | `-AIRY_ECAP_BADGE` (-78) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |
| 非法 opcode 提交 | 白名单外 | -EOPNOTSUPP |
| 无权限注册 buffer | CAP_REGISTER_BUFFER 缺失 | -AIRY_ECAP_PERM |
| sec_d 编译 Badge 后提交 | C-S9 通过 | 正常执行（Badge 唯一写者路径） |
| `atomic_inc` 撤销后旧 Badge 提交 | C-S9a Epoch 失效 | `-AIRY_ECAP_EPOCH` (-79) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |

### 9.2 性能 SLO（v1.1: fastpath C-S9 ~10ns）

| 指标 | SLO | 实测 | 说明 |
|------|-----|------|------|
| fastpath C-S9 Badge 校验 | ≤20ns | ~10ns | v1.1: 3×READ_ONCE + 位运算 + 比较 |
| FAST_SEND 总延迟 | ≤200ns | ~158ns | v1.1: 含 C-S9 + Ring Buffer 写入 |
| SLOW_SEND（C-S9 失败） | ≤1μs | ~600ns-5.5μs | v1.1: LSM 钩子 + 冷酷执法 |
| opcode 白名单检查 | ≤10ns | ~5ns | 无变更 |
| Badge 撤销（atomic_inc） | ≤5ns | ~1ns | v1.1: 1 行 atomic_inc |
| LSM 注册开销 | 编译期 | 0（运行时） | 无变更 |
| 与 BPF LSM 对比 | 更快 | ~20x faster（fastpath C-S9） | v1.1: ~10ns vs ~200ns+ |

---

## §10 相关文档

- [01-lsm-framework.md](01-lsm-framework.md) —— LSM 框架（上级设计，v1.1: §8.4 fastpath C-S9 与 LSM 钩子职责分割）
- [10-unify-design.md](../10-architecture/10-unify-design.md) §7 —— A-ULS 模块（Micro-Supervisor 基于 LSM）
- [09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) —— Micro-Supervisor（v1.1: §2.3 slowpath LSM 接管 + §6.4 Badge 撤销 atomic_inc）
- [03-capability-model.md](03-capability-model.md) —— Capability 模型（v1.1: Badge 64-bit Native Word，§2.5 Perms 位定义）
- [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) §3 —— opcode 表（v1.1: FREEZE 0x0005）
- [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) §5.2 —— fastpath C-S9 Badge 校验实现（~10ns）
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— Error/Fault 码（v1.1: -78~-82 Badge 码 + 0x1001-0x1006 Fault 码）
- [11-unified-config.md](../20-modules/11-unified-config.md) §7 —— A-IPC Capability Folding 配置项（agent_caps 容量、Epoch 位宽等）
- [06-io-uring-hardening.md](06-io-uring-hardening.md) —— io_uring 安全加固（基于纯 C LSM）
- [02-landlock-sandbox.md](02-landlock-sandbox.md) —— Landlock 沙箱（共存 LSM）
- [03-ipc-performance.md](../170-performance/03-ipc-performance.md) —— A-IPC 性能 SLO（FAST_SEND ~158ns）

---

## §11 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：纯 C LSM 模块设计；对齐 openEuler 纯 C LSM 模式（SELinux/AppArmor/Landlock/Tomoyo 全部纯 C，不使用 BPF LSM）；DEFINE_LSM(airy) 纯 C 注册 + LSM_ORDER_FIRST；Capability 校验钩子（security_uring_cmd 回调）；io_uring 安全钩子（opcode 白名单）；借鉴 seL4 capability-based security；纯 C 性能优势（fastpath ~160ns，无 BPF 间接调用开销）；源码参考 OLK 6.6 security/ 目录 |
| v1.1 | 2026-07-18 | **Capability Folding 集成版**（A-IPC 第一块基石）：① §1.3 设计目标新增 sec_d Badge 编译职责；② §2.1 初始化从 radix tree 改为 agent_caps[] 静态数组 + 全局 Epoch；③ §3 重构——fastpath C-S9 内联 Badge 校验（~10ns） + slowpath LSM 钩子接管（仅 C-S9 失败时），新增 §3.5 sec_d Badge 编译职责（唯一写者 + WRITE_ONCE + Random Tag 强制）；④ §5 seL4 对齐表更新（CSpace→agent_caps[]，radix tree→静态数组，撤销 O(n)→O(1) atomic_inc）；⑤ §6 性能更新（fastpath ~160ns→~10ns，FAST_SEND ~320ns→~158ns）；⑥ §7 源码结构 airy_cap_tree.c→airy_capability.c，Makefile 更新；⑦ §9 测试场景对齐 Badge 错误码（-78~-82, 0x1001-0x1006），性能 SLO 更新；⑧ §10 相关文档清除内部审查路径引用，新增 v1.1 引用 |

---

> **文档结束** | 纯 C LSM 模块设计 | v1.1 | 2026-07-18

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | 纯 C LSM 模块设计 | v1.1 | 2026-07-18
