Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 纯 C LSM 模块设计
> **文档定位**：Airymax 纯 C LSM（Linux Security Module）模块的唯一权威详细设计，对齐 openEuler 纯 C LSM 模式\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7 + [01-lsm-framework.md](01-lsm-framework.md)\
> **设计依据**：本文件为 SSoT（Single Source of Truth），纯 C LSM 模块职责、fastpath/slowpath 职责分割、sec_d Badge 编译职责、Capability 校验钩子均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **纯 C LSM 模块设计** 的唯一权威源。`DEFINE_LSM(airy)` 纯 C 注册、**v1.1 fastpath C-S9 + slowpath LSM 职责分割**（fastpath 内联 Badge 校验，LSM 钩子仅 slowpath 接管）、**sec_d Badge 编译职责**（sec_d 是 Badge 的唯一写者，编译 Badge = `Epoch<<48 | RandomTag<<16 | Perms`）、io_uring 安全钩子（opcode 白名单校验）、与 seL4 Capability 模型的关系、纯 C 性能优势（无 BPF 间接调用开销）、源码参考（OLK 6.6 security/ 目录）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Airymax LSM 模块结构与 fastpath/slowpath 职责分割。
>
> **v1.0.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.0.1 起，纯 C LSM 的 capability 校验路径从"独立前置 `airy_cap_check()` + radix tree 查找"重构为 **fastpath C-S9 内联 Badge 校验**（`airy_cap_badge_ok()`，~10ns，详见 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md)）。纯 C LSM 不再在 `security_uring_cmd` 钩子中独立执行 capability 校验——fastpath C-S9 已在 io_uring 数据面完成 Badge 校验，LSM 钩子仅在 slowpath（异常路径）做策略裁决与冷酷执法。**sec_d 是 Badge 的唯一写者**——sec_d 编译 Badge 时使用全局 Epoch + 随机生成的 32-bit Random Tag + Perms 位掩码，写入 `agent_caps[agent_id]` 静态数组。Badge 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码立即失效所有已发出 Badge（详见 [09-kernel-agent-supervisor.md §6.4](../20-modules/09-kernel-agent-supervisor.md)）。
>
> 技术选型声明：安全采用 **纯 C LSM 模块**（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式：SELinux/AppArmor/Landlock/Tomoyo 全部纯 C）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

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

> **Landlock 纯 C 事实验证**（基于 OLK 6.6 源码）：`security/landlock/` 目录全部为 `.c/.h` 文件（`setup.c` / `syscalls.c` / `object.c` / `ruleset.c` / `cred.c` / `ptrace.c` / `fs.c`），完全不依赖 BPF。Landlock 使用专用 syscall（`landlock_create_ruleset` / `landlock_add_rule` / `landlock_restrict_self`）而非 BPF 程序。注：Landlock 早期设计草案（2018-2020）曾考虑基于 BPF，但最终实现（Linux 5.13+）为纯 C，常见的外部误解源于此早期草案。

### 1.3 设计目标（v1.1: fastpath/slowpath 职责分割）

纯 C LSM 模块的核心设计目标：

1. **Capability 校验**（v1.1 职责分割）：fastpath C-S9 内联 Badge 校验（~10ns）+ slowpath LSM 钩子接管异常（冷酷执法）
2. **io_uring 安全**：opcode 白名单校验（§4）
3. **LSM_ORDER_MUTABLE + CONFIG_LSM 优先位置**（v1.1 修正）：通过 `CONFIG_LSM` 默认值将 `airy` 置于 `capability` 之后、其他 LSM 之前，实现"capability 优先"语义。**不使用 `LSM_ORDER_FIRST`**——OLK 6.6 `include/linux/lsm_hooks.h:113` 注释明确 `LSM_ORDER_FIRST` "This is only for capabilities"
4. **零 BPF 依赖**：纯 C 实现，不依赖 BPF 虚拟机
5. **sec_d Badge 编译**（v1.0.1 新增）：sec_d 是 Badge 的唯一写者，编译 Badge = `Epoch<<48 | RandomTag<<16 | Perms`，写入 `agent_caps[agent_id]` 静态数组

---

## §2 DEFINE_LSM(airy)：纯 C LSM 模块注册

### 2.1 LSM 注册机制

Airymax 使用 `DEFINE_LSM` 宏注册纯 C LSM 模块。这是 Linux 6.6 标准的 LSM 注册方式，与 SELinux/AppArmor/Landlock 一致：

```c
/* security/airy/airy_lsm.c —— 纯 C LSM 模块注册（v1.1，OLK 6.6 对齐） */
#include <linux/lsm_hooks.h>
#include <linux/module.h>
#include "airy_internal.h"

/* Airymax LSM 模块信息
 *
 * OLK 6.6 对齐说明（include/linux/lsm_hooks.h:112-116）:
 *   enum lsm_order {
 *       LSM_ORDER_FIRST = -1,   /* This is only for capabilities. */
 *       LSM_ORDER_MUTABLE = 0,
 *       LSM_ORDER_LAST = 1,    /* This is only for integrity. */
 *   };
 *
 * v1.1 修正: airy 不使用 LSM_ORDER_FIRST（OLK 6.6 注释明确仅用于 capabilities），
 *           改用 LSM_ORDER_MUTABLE（默认值），通过 CONFIG_LSM 默认值将 airy 置于
 *           capability 之后、其他 LSM 之前，实现"capability 优先"语义。
 *
 *           capability 优先的工程实现不依赖 LSM_ORDER_FIRST，而是通过：
 *             1. fastpath C-S9 内联 Badge 校验（~10ns，在 io_uring 数据面完成）
 *             2. security_uring_cmd 钩子内部分发 opcode 白名单 + buffer 注册校验
 *             3. CONFIG_LSM 默认值: "landlock,lockdown,yama,loadpin,safesetid,airy,selinux,..."
 *                （airy 在 capability/integrity 之后、其他 LSM 之前）
 */
DEFINE_LSM(airy) = {
    .name = "airy",
    /* v1.1: 不设置 .order 或设置为 LSM_ORDER_MUTABLE（默认值）
     * OLK 6.6 注释: LSM_ORDER_FIRST 仅用于 capabilities
     */
    .order = LSM_ORDER_MUTABLE,
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

    pr_info("airy: pure-C LSM loaded (no BPF), order=MUTABLE, v1.0.1 Capability Folding\n");
    return 0;
}

/* 模块参数：可通过 sysctl 控制启用/禁用 */
static bool airy_enabled = true;
module_param(airy_enabled, bool, 0644);
MODULE_PARM_DESC(airy_enabled, "Enable/disable Airymax pure-C LSM");
```

### 2.2 LSM_ORDER_MUTABLE + CONFIG_LSM 优先位置（v1.1 修正）

**v1.1 修正**：airy 不使用 `LSM_ORDER_FIRST`（OLK 6.6 注释明确"仅用于 capabilities"），改用 `LSM_ORDER_MUTABLE`（默认值），通过 `CONFIG_LSM` 默认值将 `airy` 置于 `capability` 之后、其他 LSM 之前。

**OLK 6.6 `CONFIG_LSM` 默认值修改**（`security/Kconfig:266-272`）：

```kconfig
config LSM
    string "Ordered list of enabled LSMs"
    # v1.1: 将 airy 添加到默认值，位于 safesetid 之后、selinux 之前
    default "landlock,lockdown,yama,loadpin,safesetid,airy,smack,selinux,tomoyo,apparmor,bpf" if DEFAULT_SECURITY_SMACK
    default "landlock,lockdown,yama,loadpin,safesetid,airy,apparmor,selinux,smack,tomoyo,bpf" if DEFAULT_SECURITY_APPARMOR
    default "landlock,lockdown,yama,loadpin,safesetid,airy,tomoyo,bpf" if DEFAULT_SECURITY_TOMOYO
    default "landlock,lockdown,yama,loadpin,safesetid,airy,bpf" if DEFAULT_SECURITY_DAC
    default "landlock,lockdown,yama,loadpin,safesetid,airy,selinux,smack,tomoyo,apparmor,bpf"
```

**执行顺序**（OLK 6.6 `security.c` 实际加载逻辑）：

| 执行顺序 | LSM 模块 | order | 来源 |
|---------|---------|-------|------|
| 0（强制首位） | capability | `LSM_ORDER_FIRST` | OLK 6.6 硬编码（仅 capabilities） |
| 1（CONFIG_LSM 列表首位） | **airy** | `LSM_ORDER_MUTABLE` | v1.1: `CONFIG_LSM` 默认值将 airy 置于列表首位 |
| 2 | landlock | `LSM_ORDER_MUTABLE` | `CONFIG_LSM` 列表 |
| 3 | lockdown/yama/loadpin/safesetid | `LSM_ORDER_MUTABLE` | `CONFIG_LSM` 列表 |
| 4 | selinux/apparmor/smack/tomoyo | `LSM_ORDER_MUTABLE` | `CONFIG_LSM` 列表 |
| 5（强制末尾） | integrity | `LSM_ORDER_LAST` | OLK 6.6 硬编码（仅 integrity） |

> **关键说明**：OLK 6.6 `security.c` 中 `LSM_ORDER_FIRST` 和 `LSM_ORDER_LAST` 是硬编码强制位置，**不可被其他 LSM 使用**。`LSM_ORDER_MUTABLE` 的 LSM 按 `CONFIG_LSM` 列表顺序加载。因此 airy 通过 `CONFIG_LSM` 默认值置于 `capability` 之后、其他 LSM 之前，即可实现"capability 优先"语义，无需滥用 `LSM_ORDER_FIRST`。

**capability 优先的工程实现**（不依赖 `LSM_ORDER_FIRST`）：

| 层次 | 机制 | 延迟 | 说明 |
|------|------|------|------|
| L1 fastpath | C-S9 内联 Badge 校验（`airy_cap_badge_ok()`） | ~10ns | 在 io_uring 数据面完成，**不进入 LSM 钩子链**，天然优先 |
| L2 slowpath | `security_uring_cmd` 钩子（airy 在 CONFIG_LSM 首位） | ~100ns+ | 仅 C-S9 失败时调用，airy 在 SELinux/AppArmor 之前执行 |
| L3 冷酷执法 | `airy_fault_enforce()` | ~1-5μs | Badge 伪造立即 SIGKILL，对齐 seL4 `handleFault()` |

这确保 Capability 校验优先——若 fastpath C-S9 不通过，slowpath airy 钩子直接返回 Fault，后续 LSM（SELinux/AppArmor/Landlock）不执行。这是借鉴 seL4 的设计：安全检查在最内层。

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

    /* 3. C-S9.EPOCH: Epoch 校验（badge_epoch == global_epoch?） */
    if (unlikely(badge_epoch != global_epoch))
        return -AIRY_ECAP_EPOCH;   /* -79: Badge 已撤销/过期 */

    /* 4. C-S9.RANDTAG: RandomTag 校验（防伪造） */
    if (unlikely(badge_randtag != randtag))
        return -AIRY_ECAP_FORGED;  /* -80: Badge 伪造，触发 Fault 0x1001 */

    /* 5. C-S9.PERMS: Perms 校验（badge_perms & required == required?） */
    __u16 required = airy_op_required_perms(opcode);
    if (unlikely((badge_perms & required) != required))
        return -AIRY_ECAP_PERM;    /* -81: 权限位不满足 */

    return 0;   /* Badge 校验通过，放行 */
}
```

### 3.3 slowpath LSM 钩子：security_uring_cmd（仅 C-S9 失败时）

`security_uring_cmd` 钩子在 v1.1 中**仅 slowpath 被 C-S9 失败时调用**，做策略裁决与冷酷执法。这是 Micro-Supervisor 的核心实现，详见 [09-kernel-agent-supervisor.md §2.3](../20-modules/09-kernel-agent-supervisor.md)。

#### 3.3.1 钩子调用模型（v1.0.1）

```
io_uring 提交 IORING_OP_URING_CMD
       │
       ▼
io_uring_cmd_submit()（io_uring 核心，fs/io_uring.c）
       │
       ▼
security_uring_cmd() LSM 钩子链（security/security.c）
       │
       ▼  ← Linux LSM 标准调用点
airy_uring_cmd_check()（security/airy/airy_cap_hooks.c）
       │
       ├─【99%+ 正常路径】C-S9 Badge 校验通过（~10ns 内联）
       │   └─ 返回 0 → io_uring 继续 issue → airy_uring_cmd()（kernel/ipc/airy_uring_cmd.c）
       │
       └─【<1% 异常路径】C-S9 校验失败
           │
           ▼  ← slowpath 接管
       airy_fault_enforce()（冷酷执法）
           ├─ 冻结 Ring（WRITE_ONCE(ring->frozen, true)）
           ├─ 触发 Fault（airy_security_fault → Micro-Supervisor）
           ├─ eventfd_signal 通知 Macro-Supervisor
           └─ 审计日志（A-ULP）
```

#### 3.3.2 security_uring_cmd 钩子完整实现（SSoT）

```c
/* security/airy/airy_cap_hooks.c —— v1.1 security_uring_cmd 完整实现
 *
 * 物理宿主: kernel/security/airy/airy_cap_hooks.c
 * 注册: LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_check)
 * 调用上下文: io_uring 提交路径，持有 ctx->uring_lock
 * 调用频率: 每次 IORING_OP_URING_CMD 提交（99%+ fastpath 通过）
 *
 * 设计原则（v1.0.1 Capability Folding）:
 *   1. fastpath C-S9 内联在 io_uring 数据面（详见 07-ipc-fastpath.md §5.2）
 *      —— 实际由 airy_uring_cmd() → airy_ipc_validate() 完成 C-S9
 *   2. 本钩子是 Linux LSM 标准调用点，在 airy_uring_cmd() 之前执行
 *   3. 本钩子承担"二次 C-S9"职责——确保 Badge 校验在 LSM 链中也有执行点
 *      （v1.1: 对齐 LSM_ORDER_MUTABLE + CONFIG_LSM 首位设计，airy 在 SELinux/AppArmor 之前执行）
 *   4. fastpath 内联 + LSM 钩子二次校验，共 ~20ns（C-S9 内联 ~10ns × 2）
 *      99%+ 路径在 fastpath 内联即通过，LSM 钩子二次校验仅做一致性验证
 *
 * 性能预算:
 *   - fastpath 内联 C-S9（airy_uring_cmd 中）: ~10ns
 *   - LSM 钩子二次 C-S9: ~10ns（缓存命中后 ~5ns）
 *   - slowpath Fault 处理: ~1-5μs（<1% 路径）
 */

#include <linux/io_uring.h>
#include <linux/io_uring_types.h>
#include <linux/lsm_hooks.h>
#include <linux/eventfd.h>
#include <linux/audit.h>
#include <linux/atomic.h>
#include <asm/unaligned.h>
#include <airymax/ipc.h>
#include <airymax/error.h>
#include <airymax/security.h>
#include "airy_internal.h"

/* Ring 冻结状态（per-ring，A-ULS 通过 FREEZE opcode 设置） */
struct airy_ring_state {
    bool                     frozen;        /* C-S0 检查 */
    atomic_t                 fault_count;   /* 该 Ring 累计 Fault 数 */
    struct eventfd_ctx      *notify_efd;    /* Macro-Supervisor 通知 eventfd */
    struct airy_ipc_cmd     *last_fault_cmd;/* 最近一次 Fault 上下文（诊断用） */
} ____cacheline_aligned_in_smp;

/**
 * airy_uring_cmd_check - security_uring_cmd LSM 钩子完整实现
 *
 * @ioucmd:      io_uring 命令结构（OLK 6.6 单参数；issue_flags 从 ioucmd->flags 获取）
 *
 * 返回:
 *   0          - Badge 校验通过，io_uring 继续 issue
 *   -AIRY_ECAP_*  - Badge 校验失败（已触发 Fault，冷酷执法完成）
 *   -AIRY_ECAP_FROZEN - Ring 已冻结（不触发 Fault，纯 Error）
 *
 * 设计权衡:
 *   - fastpath 内联 C-S9 在 airy_uring_cmd() 中完成（~10ns）
 *   - 本钩子做"二次 C-S9"（缓存命中 ~5ns），确保 LSM 链中 capability 优先
 *   - 若二次校验与 fastpath 结果不一致（极小概率竞态），以本钩子结果为准
 *     —— 这是 H4（agentrt-linux 内核 Badge 由 sec_d 编译）的最终裁决点
 *
 * OLK 6.6 对齐:
 *   - security_uring_cmd 钩子签名为单参数（include/linux/lsm_hook_defs.h:422）
 *   - LSM_HOOK(int, 0, uring_cmd, struct io_uring_cmd *ioucmd)
 */
static int airy_uring_cmd_check(struct io_uring_cmd *ioucmd)
{
    /* OLK 6.6: struct io_uring_cmd 无 cmd 字段，使用 io_uring_cmd_to_pdu() 宏访问 pdu[32]
     * uring_cmd LSM 钩子在 OLK 6.6 中是单参数（issue_flags 从 ioucmd->flags 获取）
     */
    struct airy_ipc_cmd *cmd = io_uring_cmd_to_pdu(ioucmd, struct airy_ipc_cmd);
    struct airy_ring_state *ring = cmd->ring_state;
    int fastpath_ret;
    u16 opcode;
    u32 src_task;
    u64 badge;

    /* ========== Phase 1: 快速参数校验（~2ns） ========== */

    /* C-S0: Ring 冻结检查（A-ULS 通过 FREEZE opcode 设置） */
    if (unlikely(READ_ONCE(ring->frozen))) {
        airy_audit_emit_security(AGENT_RING_FROZEN_BLOCK,
                                  cmd->src_task, 0, 0);
        return -AIRY_ECAP_FROZEN;  /* -82, Error 不触发 Fault */
    }

    opcode    = cmd->hdr->opcode;
    src_task  = cmd->hdr->src_task;
    badge     = cmd->hdr->capability_badge;

    /* ========== Phase 2: CAP_REQUEST 自举路径（~5ns） ========== */
    /* H4: CAP_REQUEST 无需 Badge，内核路由到 sec_d */
    if (unlikely(opcode == AIRY_IPC_OP_CAP_REQUEST)) {
        if (unlikely(cmd->hdr->dst_task != AIRY_SEC_D_TASK_ID)) {
            /* CAP_REQUEST 只能发给 sec_d，否则视为伪造 */
            return airy_fault_enforce(AIRY_FAULT_CAP_FORGED, cmd,
                                        __func__, __LINE__);
        }
        return 0;  /* CAP_REQUEST 自举路径放行 */
    }

    /* ========== Phase 3: [DSL] 降级模式 / agentrt 用户态（~3ns） ========== */
    /* H3: agentrt 用户态 capability_badge 始终为 0 */
    /* H6: [DSL] 降级模式 capability_badge=0，跳过校验 */
    if (badge == 0) {
        if (unlikely(cmd->hdr->flags & AIRY_IPC_F_CAP_CARRY)) {
            /* 内核模式不应出现 badge=0 + CAP_CARRY */
            return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd,
                                        __func__, __LINE__);
        }
        return 0;  /* agentrt 用户态或 [DSL] 模式放行 */
    }

    /* ========== Phase 4: fastpath C-S9 Badge 二次校验（~5-10ns） ========== */
    /*
     * fastpath 内联 C-S9 已在 airy_uring_cmd() → airy_ipc_validate() 完成，
     * 此处做二次校验确保 LSM 链中 capability 优先（v1.1: LSM_ORDER_MUTABLE + CONFIG_LSM 首位设计）。
     * 二次校验缓存命中后 ~5ns（agent_caps[src_task] 已在 L1）。
     */
    fastpath_ret = airy_cap_badge_ok(current, badge, opcode);
    if (likely(fastpath_ret == 0))
        return 0;   /* 99%+ 正常路径，Badge 校验通过 */

    /* ========== Phase 5: slowpath 冷酷执法（~1-5μs，<1% 路径） ========== */
    /*
     * Badge 校验失败，根据错误码执行不同 Fault 策略。
     * 详见 08-sc-error-contract.md §2.3 IPC 错误码空间。
     */
    return airy_handle_badge_failure(fastpath_ret, cmd, ring);
}

/**
 * airy_handle_badge_failure - Badge 校验失败的 slowpath 处理
 *
 * @ret:    fastpath C-S9 返回的错误码（-78/-79/-80/-81/-82）
 * @cmd:    IPC 命令上下文（含 src_task、opcode、capability_badge）
 * @ring:   目标 Ring 状态
 *
 * 返回: 错误码（与 @ret 一致，便于 io_uring 直接回填 CQE）
 *
 * 处理策略（对齐 Fault 0x1001/0x1003/0x1005）:
 *   -AIRY_ECAP_FORGED (-80)  → Fault 0x1001（capability 伪造，立即 KILL）
 *   -AIRY_ECAP_EPOCH  (-79)  → Fault 0x1005（capability 异常，冻结 Ring）
 *   -AIRY_ECAP_PERM   (-81)  → Fault 0x1005（权限不足，冻结 Ring）
 *   -AIRY_ECAP_BADGE  (-78)  → Fault 0x1005（Badge 格式无效，冻结 Ring）
 *   -AIRY_ECAP_FROZEN (-82)  → 无 Fault（Ring 已冻结，纯 Error）
 */
static noinline int airy_handle_badge_failure(int ret,
                                                struct airy_ipc_cmd *cmd,
                                                struct airy_ring_state *ring)
{
    u32 fault_code;
    bool kill_agent = false;

    switch (ret) {
    case -AIRY_ECAP_FORGED:     /* -80: Badge 伪造，最严重 */
        fault_code = AIRY_FAULT_CAP_FORGED;       /* 0x1001 */
        kill_agent = true;                         /* 立即 KILL Agent */
        break;

    case -AIRY_ECAP_EPOCH:      /* -79: Epoch 不匹配（已撤销或过期） */
        fault_code = AIRY_FAULT_ABNORMAL_CAP;     /* 0x1005 */
        break;

    case -AIRY_ECAP_PERM:       /* -81: 权限位不满足 */
        fault_code = AIRY_FAULT_ABNORMAL_CAP;     /* 0x1005 */
        break;

    case -AIRY_ECAP_BADGE:      /* -78: Badge 格式无效 */
        fault_code = AIRY_FAULT_ABNORMAL_CAP;     /* 0x1005 */
        break;

    case -AIRY_ECAP_FROZEN:     /* -82: Ring 已冻结 */
        /* 不触发 Fault，纯 Error */
        return ret;

    default:
        fault_code = AIRY_FAULT_ABNORMAL_CAP;     /* 0x1005 */
        break;
    }

    /* 1. 触发 Fault（异步冷酷执法，Micro-Supervisor 接管） */
    airy_security_fault(cmd->hdr->src_task, fault_code, cmd);

    /* 2. 冻结 Ring（同步，防止后续 IPC 继续攻击） */
    WRITE_ONCE(ring->frozen, true);
    atomic_inc(&ring->fault_count);
    ring->last_fault_cmd = cmd;

    /* 3. 通知 Macro-Supervisor（eventfd_signal，异步温情裁决） */
    if (ring->notify_efd)
        eventfd_signal(ring->notify_efd, 1);

    /* 4. 审计日志（A-ULP，永不可绕过） */
    airy_audit_emit_security(AGENT_CAP_FAULT,
                              cmd->hdr->src_task,
                              cmd->hdr->capability_badge,
                              fault_code);

    /* 5. 严重 Fault（FORGED）立即 KILL Agent */
    if (kill_agent) {
        /* 发送 SIGKILL 给 Agent 进程，对齐 seL4 handleFault() 语义 */
        send_sig(SIGKILL, find_task_by_vpid(cmd->hdr->src_task), 1);
        pr_warn_ratelimited("airy: agent %u killed (badge forged)\n",
                             cmd->hdr->src_task);
    }

    return ret;
}

/**
 * airy_security_fault - 触发 Fault 通知 Micro-Supervisor
 *
 * @agent_id:   故障 Agent ID
 * @fault_code: Fault 码（0x1001/0x1003/0x1005）
 * @cmd:        故障上下文（含 opcode、capability_badge，用于诊断）
 *
 * 异步通知内核 Micro-Supervisor kthread 处理 Fault。
 * 详见 09-kernel-agent-supervisor.md §2.3。
 */
void airy_security_fault(u32 agent_id, u32 fault_code,
                          struct airy_ipc_cmd *cmd)
{
    struct airy_fault_event event = {
        .agent_id    = agent_id,
        .fault_code  = fault_code,
        .opcode      = cmd ? cmd->hdr->opcode : 0,
        .badge       = cmd ? cmd->hdr->capability_badge : 0,
        .timestamp_ns = ktime_get_real_ns(),
    };

    /* 投递到 Micro-Supervisor kfifo（非阻塞，溢出则计数 + 丢弃） */
    if (kfifo_in(&airy_fault_kfifo, &event, 1) != 1) {
        atomic_inc(&airy_fault_kfifo_dropped);
        pr_warn_ratelimited("airy: fault kfifo overflow, dropped\n");
    }

    /* 唤醒 Micro-Supervisor kthread */
    wake_up_interruptible(&airy_fault_waitq);
}

/**
 * airy_fault_enforce - 冷酷执法入口（early-fault 简化路径）
 *
 * @fault_code: Fault 码（AIRY_FAULT_CAP_FORGED / AIRY_FAULT_ABNORMAL_CAP）
 * @cmd:        IPC 命令上下文
 * @func:       调用函数名（审计用）
 * @line:       调用行号（审计用）
 *
 * 用于 Phase 2/3 等早期路径的快速 Fault 处理（无需 Ring 状态查询）。
 * 完整 Ring 冻结逻辑由 airy_handle_badge_failure() 负责。
 *
 * 返回: -AIRY_ECAP_FORGED 或 -AIRY_ECAP_BADGE（与 fault_code 对应）
 */
static noinline int airy_fault_enforce(u32 fault_code,
                                         struct airy_ipc_cmd *cmd,
                                         const char *func, int line)
{
    bool kill_agent = (fault_code == AIRY_FAULT_CAP_FORGED);

    /* 1. 触发 Fault */
    airy_security_fault(cmd->hdr->src_task, fault_code, cmd);

    /* 2. 审计日志（含调用位置，便于追溯） */
    airy_audit_emit_security(AGENT_CAP_FAULT_EARLY,
                              cmd->hdr->src_task,
                              cmd->hdr->capability_badge,
                              fault_code | ((u32)line << 16));

    /* 3. 严重 Fault（FORGED）立即 KILL Agent */
    if (kill_agent) {
        send_sig(SIGKILL, find_task_by_vpid(cmd->hdr->src_task), 1);
        pr_warn_ratelimited("airy: agent %u killed (early fault %s:%d)\n",
                             cmd->hdr->src_task, func, line);
    }

    return (fault_code == AIRY_FAULT_CAP_FORGED) ?
           -AIRY_ECAP_FORGED : -AIRY_ECAP_BADGE;
}
```

#### 3.3.3 钩子注册与 LSM_ORDER_MUTABLE（v1.1 修正）

```c
/* security/airy/airy_lsm.c —— 钩子注册（v1.1，OLK 6.6 对齐）
 *
 * OLK 6.6 io_uring LSM 钩子全集（include/linux/lsm_hook_defs.h:420-422）：
 *   LSM_HOOK(int, 0, uring_override_creds, const struct cred *new)
 *   LSM_HOOK(int, 0, uring_sqpoll, void)
 *   LSM_HOOK(int, 0, uring_cmd, struct io_uring_cmd *ioucmd)  ← 单参数
 *
 * OLK 6.6 中不存在 uring_sqe / uring_register_buffers 钩子（v1.0 文档误注册，v1.1 已删除）。
 * opcode 白名单 + buffer 注册校验改为：
 *   - opcode 白名单：在 airy_uring_cmd_check() 内部分发（基于 ioucmd->cmd_op）
 *   - buffer 注册校验：在 io_uring_register() 路径中通过 security_file_ioctl() 等既有钩子承担，
 *     或通过 airy_uring_cmd_check() 在 IORING_OP_URING_CMD 子命令中校验
 *
 * OLK 6.6 LSM_ORDER_FIRST 仅用于 capabilities（include/linux/lsm_hooks.h:113 注释）：
 *   LSM_ORDER_FIRST = -1,   /* This is only for capabilities. */
 * airy 使用 LSM_ORDER_MUTABLE（默认值），通过 CONFIG_LSM 默认值置于 capability 之后。
 */

static struct security_hook_list airy_hooks[] __lsm_ro_after_init = {
    /* Capability 校验钩子 —— v1.1: LSM_ORDER_MUTABLE + CONFIG_LSM 首位（替代 LSM_ORDER_FIRST） */
    LSM_HOOK_INIT(uring_cmd,         airy_uring_cmd_check),

    /* 进程安全钩子（Agent 状态追踪） */
    LSM_HOOK_INIT(task_alloc,        airy_task_alloc),
    LSM_HOOK_INIT(task_free,         airy_task_free),
    LSM_HOOK_INIT(task_kill,         airy_task_kill_check),

    /* 文件安全钩子（CAP_FILE_OPEN 校验） */
    LSM_HOOK_INIT(file_open,         airy_file_open_check),
};

DEFINE_LSM(airy) = {
    .name    = "airy",
    .order   = LSM_ORDER_MUTABLE,   /* v1.1: 不使用 LSM_ORDER_FIRST（OLK 6.6 仅用于 capabilities） */
    .enabled = &airy_enabled,
    .init    = airy_lsm_init,
    .blobs   = &air_blob_sizes,
};
```

#### 3.3.4 关键设计权衡（v1.0.1）

| 设计点 | 选择 | 理由 |
|--------|------|------|
| C-S9 执行位置 | fastpath 内联（airy_uring_cmd）+ LSM 钩子二次校验 | 双重校验确保 LSM 链中 capability 优先；fastpath 内联消除独立 syscall |
| LSM 钩子触发频率 | 每次 IORING_OP_URING_CMD 提交（99%+ 通过） | Linux LSM 标准行为，无法选择性跳过；通过缓存命中降低开销至 ~5ns |
| FORGED 处理策略 | 立即 SIGKILL | 对齐 seL4 handleFault() 语义；伪造 Badge 是最严重攻击 |
| EPOCH/PERM/BADGE 处理策略 | 冻结 Ring + 通知 Macro-Supervisor | 温情裁决：可能是 Epoch 撤销未及时同步，或权限配置错误 |
| FROZEN 处理策略 | 纯 Error，不触发 Fault | Ring 已冻结是预期状态，非攻击 |
| Ring 冻结机制 | WRITE_ONCE(ring->frozen, true) | 原子写入，后续 C-S0 检查立即失败，阻止后续攻击 |
| Macro-Supervisor 通知 | eventfd_signal（非阻塞） | 避免在 io_uring 持锁路径中阻塞 |
| 审计日志 | airy_audit_emit_security（永不可绕过） | A-ULP 审计是合规要求，Fault 处理必经审计 |

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
       ├── C-S9.EPOCH: Epoch 校验（badge_epoch == global_epoch?）
       │   └── 不匹配 ─────────▶ 返回 -AIRY_ECAP_EPOCH (-79)
       │
       ├── C-S9.RANDTAG: RandomTag 校验（防伪造）
       │   └── 不匹配 ─────────▶ 返回 -AIRY_ECAP_FORGED (-80, 触发 Fault 0x1001)
       │
       └── C-S9.PERMS: Perms 校验（badge_perms & required == required?）
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

### 3.5 sec_d Badge 编译职责（v1.0.1 新增）

**v1.0.1 新增**：sec_d（security daemon）是 Badge 的**唯一写者**。sec_d 编译 Badge 时使用全局 Epoch + 随机生成的 32-bit Random Tag + Perms 位掩码，写入 `agent_caps[agent_id]` 静态数组。这是 Capability Folding 单平面架构的关键约束——fastpath C-S9 仅做读操作（READ_ONCE），sec_d 独占写操作（WRITE_ONCE）。

```c
/* services/daemons/sec_d/badge_compile.c —— sec_d Badge 编译（v1.0.1）
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
| `security_uring_cmd` | io_uring_cmd 提交 | Badge 校验（核心，IPC fastpath）+ opcode 白名单分发 | v1.1: 仅 slowpath，fastpath C-S9 内联；opcode 白名单改在本钩子内部分发 |
| `security_task_kill` | kill 信号 | CAP_KILL 权限 | 无变更 |
| `security_file_open` | 文件打开 | CAP_FILE_OPEN 权限 | 无变更 |
| `security_file_ioctl` | 文件 ioctl | buffer 注册类操作校验（替代虚构的 uring_register_buffers） | v1.1: 替代 v1.0 误注册的 uring_register_buffers |

> **OLK 6.6 对齐说明**：v1.0 文档曾注册 `security_uring_sqe` 和 `security_uring_register_buffers` 两个虚构 LSM 钩子，但 OLK 6.6 `include/linux/lsm_hook_defs.h` 中**仅定义 3 个 io_uring 钩子**（`uring_override_creds`/`uring_sqpoll`/`uring_cmd`），不存在上述两个钩子。v1.1 已删除虚构钩子，opcode 白名单改在 `security_uring_cmd` 内部基于 `ioucmd->cmd_op` 分发，buffer 注册校验改由 `security_file_ioctl` 等既有钩子承担。

---

## §4 io_uring 安全钩子：io_uring_cmd opcode 白名单校验

### 4.1 io_uring 安全钩子集

Airymax 纯 C LSM 在 `security_uring_cmd` 钩子内部分发 opcode 白名单与 buffer 注册类子命令校验（详见 [06-io-uring-hardening.md](06-io-uring-hardening.md)）：

```c
/* security/airy/airy_uring_hooks.c —— io_uring 安全辅助函数（非独立 LSM 钩子）
 *
 * OLK 6.6 对齐: 不注册 uring_sqe / uring_register_buffers 钩子（OLK 6.6 中不存在）
 * opcode 白名单与 buffer 注册校验改为 airy_uring_cmd_check() 内部分发的辅助函数
 */
/* opcode 白名单 */
static const unsigned int airy_uring_allowed_opcodes[] = {
    IORING_OP_URING_CMD,   /* IPC 语义映射 */
    IORING_OP_NOP,         /* 基础测试 */
};

/* opcode 白名单校验辅助函数（由 airy_uring_cmd_check() 内部调用，非独立 LSM 钩子）
 * 在 security_uring_cmd 钩子内基于 ioucmd->cmd_op 分发
 */
static int airy_uring_opcode_check(struct io_uring_cmd *ioucmd)
{
    u32 cmd_op = ioucmd->cmd_op;
    for (size_t i = 0; i < ARRAY_SIZE(airy_uring_allowed_opcodes); i++) {
        if (cmd_op == airy_uring_allowed_opcodes[i])
            return 0;  /* 白名单允许 */
    }
    /* 非法 opcode：拒绝 */
    pr_warn_ratelimited("airy: blocked io_uring cmd_op %u pid=%d\n",
                        cmd_op, current->pid);
    return -EOPNOTSUPP;
}

/* buffer 注册类子命令校验（由 airy_uring_cmd_check() 在 IORING_OP_URING_CMD 子命令中调用）
 * 替代 v1.0 误注册的 uring_register_buffers LSM 钩子
 */
static int airy_uring_buffer_register_check(struct io_uring_cmd *ioucmd)
{
    struct airy_ipc_cmd *cmd = io_uring_cmd_to_pdu(ioucmd, struct airy_ipc_cmd);
    void __user *addr = (void __user *)cmd->addr;
    unsigned long size = cmd->size;

    /* 1. Capability 校验
     * v1.1: buffer 注册是 non-IPC 路径，走 slowpath airy_cap_check()
     *       （IPC fastpath 走 C-S9 Badge 内联校验，详见 03-capability-model.md §4.3.2）
     */
    if (airy_cap_check(current_agent_id(), current_cap_id(),
                       AIRY_CAP_REGISTER_BUFFER) < 0)
        return -AIRY_ECAP_PERM;

    /* 2. 地址与大小校验 */
    if (!access_ok(addr, size) || size > AIRY_URING_MAX_BUFFER_SIZE)
        return -EINVAL;

    return 0;
}
```

> **设计说明**：`airy_uring_opcode_check()` 和 `airy_uring_buffer_register_check()` 是 `airy_uring_cmd_check()` 内部分发的辅助函数，**不是独立 LSM 钩子**。这样设计避免注册 OLK 6.6 中不存在的虚构钩子，同时保留 opcode 白名单与 buffer 注册校验的能力。

### 4.2 钩子注册

```c
/* security/airy/airy_lsm.c —— 钩子注册（v1.1，OLK 6.6 对齐）
 * OLK 6.6 io_uring LSM 钩子仅 3 个：uring_override_creds / uring_sqpoll / uring_cmd
 * 不注册 uring_sqe / uring_register_buffers（OLK 6.6 中不存在，v1.0 误注册已删除）
 */
static struct security_hook_list airy_hooks[] __lsm_ro_after_init = {
    /* io_uring 安全钩子（OLK 6.6 唯一 io_uring 钩子，opcode 白名单在内部分发） */
    LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_check),

    /* 进程安全钩子 */
    LSM_HOOK_INIT(task_kill, airy_task_kill_check),
    LSM_HOOK_INIT(task_alloc, airy_task_alloc),
    LSM_HOOK_INIT(task_free, airy_task_free),

    /* 文件安全钩子（CAP_FILE_OPEN + buffer 注册类 ioctl 校验） */
    LSM_HOOK_INIT(file_open, airy_file_open_check),
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

| seL4 机制 | Airymax 纯 C LSM 对齐（v1.0.1） | 说明 |
|-----------|---------------------|------|
| CNode（Capability Node） | `struct airy_cap_slot`（agent_caps[] 数组项） | v1.1: 物理存储替代 airy_cnode（逻辑概念保留，详见 03-capability-model.md §2.1） |
| CSpace（Capability Space） | `agent_caps[1024]` 静态数组（16KB） | v1.1: 替代 radix tree |
| mint/copy/move/revoke | `airy_cap_badge_compile/revoke`（sec_d 编译 + atomic_inc 撤销） | v1.1: sec_d 唯一写者，撤销 = 1 行 atomic_inc |
| badge（权限掩码） | Badge 64-bit Native Word（`Epoch<<48 \| RandomTag<<16 \| Perms`） | v1.1: 完整 64-bit Badge，含 Epoch + RandomTag + Perms |
| handleFault() | `airy_fault_enforce()` | Fault 处理（冷酷执法） |

### 5.2 借鉴的关键设计

| seL4 设计 | Airymax 实现（v1.0.1） | 价值 |
|-----------|------------|------|
| Capability 不可伪造 | sec_d 编译 Badge 时生成 32-bit 随机 Random Tag，fastpath C-S9.RANDTAG 校验 | 防伪造（~10ns 校验） |
| 最小权限 | Perms 16-bit 位掩码，每操作校验 required perms | 最小授权 |
| 递归撤销 | `atomic_inc(&airy_cap_global_epoch)` 一行代码立即失效所有已发出 Badge | O(1) 撤销，无 drain/bitmap/IPI |
| 形式化验证 | 参考验证方法（未完全形式化） | 可信基础 |

### 5.3 与 seL4 的差异（v1.1: agent_caps[] 静态数组）

| 维度 | seL4 | Airymax（v1.0.1） | 适配原因 |
|------|------|---------|---------|
| 执行域 | 微内核（纯内核态） | Linux 6.6（LSM 钩子 slowpath + fastpath 内联） | 复用主线 LSM + io_uring 数据面 |
| Capability 存储 | CSpace（内核对象） | `agent_caps[1024]` 静态数组（16KB） | v1.1: 替代 radix tree，无锁多读者 |
| 校验方式 | 内核内联 | fastpath C-S9 内联（~10ns） + slowpath LSM | v1.1: fastpath/slowpath 职责分割 |
| 撤销机制 | 递归遍历 CSpace（O(n)） | `atomic_inc` 全局 Epoch（O(1)） | v1.1: 1 行代码完成全局撤销 |
| 形式化验证 | 完全形式化 | 未形式化（CBMC 全函数验证 fastpath） | 工程权衡 |

---

## §6 性能：纯 C 无 BPF 间接调用开销（v1.1: fastpath C-S9 ~10ns）

### 6.1 性能对比（v1.1: fastpath C-S9 Badge 校验 ~10ns）

纯 C LSM 相比 BPF LSM 的性能优势在于无间接调用开销。v1.0.1 起 fastpath C-S9 Badge 校验降至 ~10ns（原 ~160ns），整体 FAST_SEND 降至 ~158ns（原 ~320ns）：

| 维度 | 纯 C LSM（v1.0.1） | BPF LSM | 差距 |
|------|---------|---------|------|
| 调用方式 | 直接函数调用（fastpath 内联） | BPF 虚拟机间接调用 | — |
| fastpath 延迟 | ~10ns（C-S9 Badge 校验） | ~200ns+ | ~20x |
| 内存开销 | 16KB 静态数组（agent_caps[]） | BPF 程序 + map | ~3x |
| 编译开销 | 编译期 | 运行时 JIT | — |

### 6.2 fastpath C-S9 性能拆解（v1.0.1）

v1.1 fastpath C-S9 Badge 校验的性能拆解（无 RCU 锁、无 radix tree 查找）：

| 步骤 | 操作 | 耗时 |
|------|------|------|
| 1. READ_ONCE × 3 | 读取 global_epoch + randtag + perms | ~3ns |
| 2. Badge 解码 | 位运算（位移 + 位与） | ~1ns |
| 3. C-S9.EPOCH Epoch 校验 | 比较运算 | ~1ns |
| 4. C-S9.RANDTAG RandomTag 校验 | 比较运算（unlikely） | ~1ns（预测成功） |
| 5. C-S9.PERMS Perms 校验 | 位运算 + 比较 | ~2ns |
| 6. 分支预测 | `unlikely` 优化 | ~0ns（预测成功） |
| **总计** | | **~8-10ns** |

### 6.3 与 IPC fastpath 的关系（v1.1: FAST_SEND ~158ns）

v1.1 fastpath C-S9 是 IPC fastpath 的一部分，总延迟在 SLO 内：

| 组件 | 延迟 | 占比 |
|------|------|------|
| C-S0 Ring 冻结检查 | ~1ns | ~1% |
| C-S1~C-S8 其他校验 | ~30ns | ~19% |
| **C-S9 Badge 校验（v1.0.1）** | **~10ns** | **~6%** |
| C-S10~C-S12 其他校验（含 CRC32） | ~17ns | ~11% |
| Ring Buffer 写入（reserve + memcpy + commit） | ~100ns | ~63% |
| **FAST_SEND 总计** | **~158ns** | **100%** |

### 6.4 性能优化手段（v1.0.1）

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

| 借鉴源 | 借鉴模式 | Airymax 应用（v1.0.1） |
|--------|---------|-------------|
| `security/selinux/hooks.c` | `DEFINE_LSM` + `security_add_hooks` | LSM 注册 |
| `security/selinux/ss/policydb.c` | 策略数据库 | v1.1: `agent_caps[1024]` 静态数组（替代 radix tree，16KB 无锁多读者） |
| `security/landlock/setup.c` | `DEFINE_LSM` + 默认 `LSM_ORDER_MUTABLE` | v1.1: airy 同样使用 `LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位 |
| `security/apparmor/path.c` | 路径权限 | 文件访问 Capability |
| `security/tomoyo/common.h` | 学习模式 | Capability 授权（可选） |
| `security/commoncap.c` | POSIX cap 检查 | 与 POSIX capability 共存 |

### 7.3 Kconfig 配置

```kconfig
# security/airy/Kconfig —— Airymax LSM 配置
# OLK 6.6 Kconfig 规范：新增 SECURITY_* 配置项 default 应为 n，用户需显式启用
config SECURITY_AIRY
    bool "Airymax pure-C LSM (Capability-based, no BPF)"
    depends on SECURITY
    depends on IO_URING
    default n
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

> **D-11 OLK 6.6 Makefile 风格对齐说明**：OLK 6.6 Makefile 风格使用 `+=` 追加目录和对象，禁止 `:=` 直接赋值。多文件模块必须通过复合目标 `<name>-y +=` 逐个追加 `.o`，禁止在 `obj-$(CONFIG_*)` 行直接列出多个 `.o`。

```makefile
# security/airy/Makefile（v1.1: airy_capability.o 替代 airy_cap_tree.o；D-11 对齐 OLK 6.6 += 风格）
obj-$(CONFIG_SECURITY_AIRY) += airy.o
airy-y += airy_lsm.o
airy-y += airy_cap_hooks.o
airy-y += airy_uring_hooks.o
airy-y += airy_capability.o
```

---

## §8 与其他安全模块的协作

### 8.1 LSM 共存

Airymax 纯 C LSM 与其他 LSM 共存，v1.1 执行顺序为 `LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位（capability 强制首位由 OLK 6.6 硬编码）：

| 顺序 | LSM | order | 执行内容 | Airymax 关系 |
|------|-----|-------|---------|-------------|
| 0 | capability | `LSM_ORDER_FIRST` | POSIX capability 检查 | OLK 6.6 硬编码首位 |
| 1 | **Airymax** | `LSM_ORDER_MUTABLE` | Badge 校验（C-S9 fastpath + LSM slowpath） | v1.1: `CONFIG_LSM` 默认值置于 capability 之后、其他 LSM 之前 |
| 2 | SELinux | `LSM_ORDER_MUTABLE` | 策略强制 | Airymax 通过后才执行 |
| 3 | AppArmor | `LSM_ORDER_MUTABLE` | 路径权限 | 独立检查 |
| 4 | Landlock | `LSM_ORDER_MUTABLE` | 用户态沙箱 | 独立检查 |
| 5 | integrity | `LSM_ORDER_LAST` | 完整性度量 | OLK 6.6 硬编码末尾 |

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
| 伪造 Badge 提交（RandomTag 不匹配） | C-S9.RANDTAG 失败 | `-AIRY_ECAP_FORGED` (-80) → `AIRY_FAULT_CAP_FORGED` (0x1001) |
| Epoch 不匹配提交（已撤销 Badge） | C-S9.EPOCH 失败 | `-AIRY_ECAP_EPOCH` (-79) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |
| 权限不足提交（Perms 不满足） | C-S9.PERMS 失败 | `-AIRY_ECAP_PERM` (-81) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |
| 已冻结 Ring 提交 | C-S0 失败 | `-AIRY_ECAP_FROZEN` (-82, Error，不触发 Fault） |
| Badge 格式无效提交（CAP_CARRY 但 badge=0） | C-S9 失败 | `-AIRY_ECAP_BADGE` (-78) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |
| 非法 opcode 提交 | 白名单外 | -EOPNOTSUPP |
| 无权限注册 buffer | CAP_REGISTER_BUFFER 缺失 | -AIRY_ECAP_PERM |
| sec_d 编译 Badge 后提交 | C-S9 通过 | 正常执行（Badge 唯一写者路径） |
| `atomic_inc` 撤销后旧 Badge 提交 | C-S9.EPOCH Epoch 失效 | `-AIRY_ECAP_EPOCH` (-79) → `AIRY_FAULT_ABNORMAL_CAP` (0x1005) |

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
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

## §12 侧信道防护（R8 补强新增）

> **章节定位**：本节为 R8 隐私/侧信道防护补强，覆盖 fastpath C-S9 Badge 校验路径与时序、cache、推测执行相关的侧信道攻击面。本节不修改 §1-§11 任何既有设计，仅在 fastpath C-S9 (`airy_cap_badge_ok()`) 与 sec_d Badge 编译（§3.5）之上补强侧信道缓解。所有防护机制的性能开销均纳入 §6 性能 SLO 框架。

### 12.1 威胁模型

AirymaxOS 在多 Agent 共享内核的场景下，面临以下 4 类侧信道攻击。攻击目标均为推断 fastpath C-S9 校验结果或 `agent_caps[]` 内容，从而绕过 Badge 校验或窃取其他 Agent 的 RandomTag。

| # | 攻击类型 | 攻击面 | 攻击者位置 | 攻击目标 |
|---|---------|--------|-----------|---------|
| 1 | **时序攻击（Timing Attack）** | fastpath C-S9 校验通过路径（~10ns）与失败路径（~10ns + Fault 处理 ~1-5μs）的响应时间差 | 同主机恶意 Agent，通过 IPC 响应延迟探测 | 推断 C-S9 校验结果（Epoch/RandomTag/Perms 哪一项失败），缩小暴力枚举空间 |
| 2 | **Cache 侧信道** | `agent_caps[1024]` 16KB 静态数组的 cache line 占用模式 | 同 CPU 核心或共享 LLC 的恶意 Agent，通过 Prime+Probe / Flush+Reload 推断 | 推断 fastpath C-S9 读取 `agent_caps[agent_id]` 的访问模式（哪个 Agent 槽位被读） |
| 3 | **Spectre V2（分支预测）** | fastpath C-S9 的 `unlikely()` 分支预测错误路径上的推测执行 | 同主机恶意 Agent，通过 BTB 投毒诱导错误预测 | 通过推测执行读取 `agent_caps[]` 内容（RandomTag/Perms），绕过校验直接获得机密 |
| 4 | **Meltdown（流氓数据加载）** | 推测执行读取内核态 `agent_caps[]` 静态数组 | 用户态恶意 Agent（无 KPTI 时） | 绕过 kernel/user 地址空间隔离，直接读取 `agent_caps[]` 中的 RandomTag |

**攻击者能力假设**：
- 攻击者控制一个合法 Agent（持有自身 Badge），可发起任意 IPC 请求。
- 攻击者可测量 IPC 响应时间（精度 ≥10ns，通过 `ktime_get_real_ns()` 或 TSC）。
- 攻击者无法直接读取 `agent_caps[]`（KPTI + 内核地址随机化保护）。
- 攻击者可在同 CPU 核心运行（共享 L1/L2 cache）或同 LLC（共享 L3）。

### 12.2 防护机制

#### 12.2.1 时序攻击防护

fastpath C-S9 Badge 校验必须保证**校验通过与失败的响应时间一致**，防止攻击者通过 IPC 响应延迟推断失败原因（缩小枚举空间）。

| 防护点 | 实现 | 性能开销 |
|--------|------|---------|
| **constant-time 比较** | fastpath C-S9 中 Epoch、RandomTag、Perms 三个比较均使用 `airy_consttime_eq()`，内部基于 XOR + OR 累积，**禁止使用 `==`/`!=` 短路求值** | ~1ns |
| **失败路径 dummy 延迟** | 校验失败路径在返回前增加 `airy_delay_ns(10)` dummy 延迟（busy-wait on TSC），使失败响应时间 ≈ 成功响应时间（~10ns） | 仅失败路径 ~10ns |
| **禁止早期 return 优化** | 编译器优化屏障（`barrier()` / `asm volatile("" ::: "memory")`）阻止编译器重排或消除 dummy 延迟；fastpath C-S9 三段校验（Epoch/RandomTag/Perms）**必须全部执行**，禁止基于前一段结果的早期 return | 0（编译期） |

```c
/* security/airy/airy_consttime.h —— constant-time 比较（R8 补强） */
static __always_inline bool airy_consttime_eq_u32(u32 a, u32 b)
{
    u32 diff = a ^ b;          /* 异或：相等则 0 */
    u32 accum = diff;         /* 累积所有位 */
    accum |= diff >> 16;
    accum |= diff << 16;
    /* 累积结果为 0 当且仅当 a == b；禁止短路返回 */
    asm volatile("" ::: "memory");   /* 编译器屏障 */
    return accum == 0;
}

/* fastpath C-S9 内 airy_cap_badge_ok() 改用 airy_consttime_eq_u32 替换 == 比较
 *（替换 §3.2 现有源码中 3 处 == 比较：Epoch / RandomTag / Perms）
 * 详见 §3.2 fastpath C-S9 内联 Badge 校验源码
 */
```

**关键约束**：
- 失败路径的 Fault 处理（`airy_fault_enforce()`，~1-5μs）本身不进入 fastpath，对发送方不可见——发送方仅看到 CQE 中的错误码（-78~-82），CQE 回填时间与成功路径一致。
- dummy 延迟仅在 fastpath 失败路径启用（成功路径零开销）。
- 明确禁止：基于校验结果的早期 return 优化（如 `if (epoch_fail) return -EPOCH;` 后跳过 RandomTag/Perms 校验）。三段校验必须全执行。

#### 12.2.2 Cache 侧信道防护

`agent_caps[1024]` 静态数组的 cache 访问模式可能泄露 Agent 槽位访问信息（哪些 Agent 活跃）。防护通过 cache line 对齐与可选的页隔离实现。

| 防护点 | 实现 | 性能开销 |
|--------|------|---------|
| **cache line 对齐** | `agent_caps[]` 数组项按 64B cache line 对齐（`__cacheline_aligned`），每个 Agent 槽位独占 cache line，避免多 Agent 共享 cache line 引发 false sharing 与侧信道泄露 | 0（对齐无运行时开销） |
| **READ_ONCE + smp_mb** | fastpath C-S9 读取 `agent_caps[agent_id]` 使用 `READ_ONCE()`（防止编译器合并/重排）+ `smp_mb()`（内存屏障，防止推测执行越过读取点） | ~1ns |
| **可选：per-Agent 独立页（4KB）** | 高安全模式（`airy.security_paranoid=1`）下，每个 Agent 分配独立 `agent_caps_slot` 页（4KB），通过 page 隔离避免 cache 冲突；通过 `/proc/airy/agent_caps_layout` 查询布局 | 仅 paranoid 模式启用，page 分配 ~4KB/Agent |

```c
/* security/airy/airy_capability.c —— agent_caps[] cache line 对齐（R8 补强） */
struct airy_cap_slot {
    __u32 randtag;       /* 32-bit Random Tag */
    __u16 perms;         /* 16-bit Perms */
    __u16 _pad;          /* 填充至 8B */
    __u64 _reserved[7];  /* 填充至 64B（cache line 对齐） */
} ____cacheline_aligned_in_smp;

/* agent_caps[1024] 静态数组，每项 64B（独占 cache line） */
static struct airy_cap_slot agent_caps[AIRY_CAP_MAX_AGENTS]
    __cacheline_aligned __ro_after_init;
```

#### 12.2.3 Spectre/Meltdown 缓解

依赖 OLK 6.6 默认缓解策略（`mitigations=auto`），Airymax 不自定义 retpoline 优化。

| 防护点 | 实现 | 性能开销 |
|--------|------|---------|
| **OLK 6.6 默认缓解** | 启用 `mitigations=auto`：IBRS（Indirect Branch Restricted Speculation）、STIBP（Single Thread Indirect Branch Predictors）、IBPB（Indirect Branch Predictor Barrier）、KPTI（Kernel Page Table Isolation） | 由 OLK 6.6 控制（~3-5% 系统级开销） |
| **fastpath lfence 序列化（仅 paranoid 模式）** | fastpath C-S9 校验前增加 `lfence` 序列化指令，阻止推测执行越过校验点；仅在 `airy.security_paranoid=1` 时启用 | ~5ns（仅 paranoid 模式） |
| **不实现自定义 retpoline** | 0.1.1 阶段不实现 retpoline 自定义优化，完全依赖 OLK 6.6 默认 retpoline（`mitigations=auto` 已包含） | 0 |

```c
/* security/airy/airy_cap_hooks.c —— fastpath C-S9 lfence 序列化（R8 补强，仅 paranoid 模式） */
static __always_inline int airy_cap_badge_ok_paranoid_wrapper(struct task_struct *task,
                                                                __u64 capability_badge,
                                                                __u16 opcode)
{
    if (unlikely(airy_security_paranoid)) {
        /* paranoid 模式：lfence 序列化，阻止推测执行越过校验点 */
        asm volatile("lfence" ::: "memory");
    }
    return airy_cap_badge_ok(task, capability_badge, opcode);
}
```

#### 12.2.4 RandomTag 暴力枚举防护

32-bit RandomTag 提供 2^32 ≈ 4.29 × 10^9 的枚举空间，配合限流与冻结策略可阻止单 Agent 暴力枚举。

| 评估维度 | 数值 |
|---------|------|
| RandomTag 空间 | 2^32 = 4,294,967,296 |
| 单 Agent 枚举速率（假设） | 100 次/秒（IPC fastpath 限速） |
| 完整枚举所需时间 | 2^32 / 100 / 86400 ≈ **497 天** |
| 100 次失败触发限流 | `airy.cap_bruteforce_threshold=100`（100 次失败 → 限流 + 告警） |
| 1000 次连续失败冻结 | 冻结 Agent + 通知 Macro-Supervisor（冷酷执法） |

| 触发条件 | 自动响应 |
|---------|---------|
| 单 Agent 1 分钟内 Badge 校验失败 ≥100 次（`airy.cap_bruteforce_threshold=100`） | sec_d 限流该 Agent 的 CAP_REQUEST 频率（指数退避，最低 1 次/10s） + WARNING 告警 |
| 单 Agent 累计 Badge 校验失败 ≥1000 次（连续） | 触发 `AIRY_FAULT_CAP_FORGED`（0x1001）冷酷执法：冻结 Agent + 通知 Macro-Supervisor + 审计日志 |

**指标对接**：Badge 校验失败次数通过 `airy_ipc_badge_fail_total` 指标导出（详见 [100-operations/04-alerting.md §13 自动化 Remediation Playbook](../100-operations/04-alerting.md)），超 100/min 触发自动冻结。

### 12.3 性能影响

侧信道防护机制对 fastpath C-S9 性能的影响（纳入 §6 性能 SLO 框架）：

| 防护机制 | 默认模式开销 | paranoid 模式开销 | 备注 |
|---------|-------------|------------------|------|
| constant-time 比较（`airy_consttime_eq_u32`） | ~1ns | ~1ns | 替换 3 处 `==` 比较 |
| cache line 对齐（`__cacheline_aligned`） | 0 | 0 | 编译期对齐 |
| READ_ONCE + smp_mb | ~1ns | ~1ns | §3.2 现有源码已使用 READ_ONCE，smp_mb 新增 |
| lfence 序列化 | 0（未启用） | ~5ns | 仅 `airy.security_paranoid=1` |
| per-Agent 独立页（4KB） | 0（未启用） | ~0（运行时）/ ~4KB/Agent（内存） | 仅 paranoid 模式 |
| **默认模式总开销** | **< 2ns** | — | 在 §6.2 fastpath C-S9 ~10ns 基础上增加 < 2ns，总计 ~12ns |
| **paranoid 模式总开销** | — | **~7ns** | 在 §6.2 fastpath C-S9 ~10ns 基础上增加 ~7ns，总计 ~17ns，仍 < 20ns SLO（§9.2） |

**结论**：
- 默认模式（`mitigations=auto`，无 paranoid）：fastpath C-S9 总延迟 ~12ns，仍满足 §9.2 SLO（≤20ns）。
- paranoid 模式（`airy.security_paranoid=1`）：fastpath C-S9 总延迟 ~17ns，仍满足 §9.2 SLO（≤20ns），但 FAST_SEND 总延迟从 ~158ns 增至 ~163ns，仍在 §6.3 FAST_SEND ≤200ns SLO 内。
- paranoid 模式仅在对侧信道威胁有明确防御需求的场景启用（如多租户高安全部署、合规审计要求），默认关闭。

---

> **文档结束** | 纯 C LSM 模块设计 | v1.1 | 2026-07-18

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | 纯 C LSM 模块设计 | v1.0.1 | 2026-07-21
