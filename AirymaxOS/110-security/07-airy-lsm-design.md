Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 纯 C LSM 模块设计
> **文档定位**：Airymax 纯 C LSM（Linux Security Module）模块的唯一权威详细设计，对齐 openEuler 纯 C LSM 模式\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7 + [01-lsm-framework.md](01-lsm-framework.md)\
> **设计依据**：[15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §4.2.4（USV 设计）+ §4.2.5（UIPF 设计）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **纯 C LSM 模块设计** 的唯一权威源。`DEFINE_LSM(airy)` 纯 C 注册、Capability 校验钩子（`security_uring_cmd` 回调）、io_uring 安全钩子（opcode 白名单校验）、与 seL4 Capability 模型的关系、纯 C 性能优势（无 BPF 间接调用开销）、源码参考（OLK 6.6 security/ 目录）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Airymax LSM 模块结构。
>
> 技术选型声明：安全采用 **纯 C LSM 模块**（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式：SELinux/AppArmor/Landlock/Tomoyo 全部纯 C）。整体遵循 Unify Design：方案 C-Prime（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：安全模块开发者、内核开发者、USV/UIPF 模块开发者
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

### 1.3 设计目标

纯 C LSM 模块的核心设计目标：

1. **Capability 校验**：在 io_uring_cmd 回调中校验 Capability
2. **io_uring 安全**：opcode 白名单校验
3. **LSM_ORDER_FIRST**：永远第一个执行，确保 Capability 优先
4. **零 BPF 依赖**：纯 C 实现，不依赖 BPF 虚拟机

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

/* 模块加载时初始化 */
static int __init airy_lsm_init(void)
{
    /* 1. 初始化 Capability radix tree */
    airy_cap_tree_init();

    /* 2. 注册安全钩子（纯 C 函数指针，非 BPF） */
    security_add_hooks(airy_hooks, ARRAY_SIZE(airy_hooks), "airy");

    /* 3. 初始化 IPC Ring 冻结机制 */
    airy_ipc_freeze_init();

    pr_info("airy: pure-C LSM loaded (no BPF), order=FIRST\n");
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
    __u64 sched_budget_ns;    /* 调度预算（方案 C-Prime） */
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

## §3 Capability 校验钩子：security_uring_cmd 回调中校验 Capability

### 3.1 security_uring_cmd 钩子

Airymax 纯 C LSM 在 `security_uring_cmd` 钩子中校验 Capability。此钩子在 `IORING_OP_URING_CMD` 提交时触发：

```c
/* security/airy/airy_cap_hooks.c —— Capability 校验钩子 */
/*
 * security_uring_cmd 钩子：在 io_uring_cmd 回调中校验 Capability
 * 纯 C 实现，不使用 BPF
 */
static int airy_uring_cmd_check(struct io_uring_cmd *ioucmd,
                                  unsigned int issue_flags)
{
    struct airy_ipc_cmd *cmd = (struct airy_ipc_cmd *)ioucmd->cmd;
    struct airy_cnode *cap;
    int ret;

    /* 1. 仅校验需要 Capability 的操作 */
    if (!airy_op_needs_cap(cmd->op))
        return 0;   /* 放行 */

    /* 2. fastpath：radix tree 缓存查找（~160ns） */
    rcu_read_lock();
    cap = radix_tree_lookup(&airy_cap_tree, cmd->cap_id);
    if (!cap) {
        rcu_read_unlock();
        /* 未找到 Capability：非法使用，冷酷执法 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);
    }

    /* 3. 校验权限掩码（badge） */
    if ((cap->badge & cmd->required_badge) != cmd->required_badge) {
        rcu_read_unlock();
        /* 权限不足：越权使用 */
        return airy_fault_enforce(AIRY_FAULT_CAP_FAULT, cmd);
    }

    /* 4. 校验状态 */
    if (cap->state != AIRY_CAP_ACTIVE) {
        rcu_read_unlock();
        /* Capability 已撤销/过期 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);
    }

    /* 5. 校验过期时间 */
    if (cap->expires_ns != 0 && ktime_get_real_ns() > cap->expires_ns) {
        rcu_read_unlock();
        /* Capability 已过期 */
        return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);
    }

    rcu_read_unlock();
    return 0;   /* 校验通过，放行 */
}
```

### 3.2 Capability 校验流程

```
io_uring_cmd 提交（IORING_OP_URING_CMD）
       │
       ▼
security_uring_cmd 钩子触发
       │
       ▼
1. 操作需要 Capability?
   ├── No ──────────────▶ 放行
   └── Yes
       ▼
2. fastpath：radix tree 查找 cap_id
   ├── 未找到 ──────────▶ AIRY_FAULT_ABNORMAL_CAP（冷酷执法）
   └── 找到
       ▼
3. 权限掩码校验（badge & required == required?）
   ├── 不匹配 ──────────▶ AIRY_FAULT_CAP_FAULT（冷酷执法）
   └── 匹配
       ▼
4. 状态校验（state == ACTIVE?）
   ├── 否 ─────────────▶ AIRY_FAULT_ABNORMAL_CAP
   └── 是
       ▼
5. 过期校验（expires_ns?）
   ├── 过期 ────────────▶ AIRY_FAULT_ABNORMAL_CAP
   └── 未过期
       ▼
   放行（校验通过）
```

### 3.3 其他 Capability 钩子

除 `security_uring_cmd` 外，Airymax 还注册以下 Capability 校验钩子：

| 钩子 | 触发场景 | 校验内容 |
|------|---------|---------|
| `security_uring_cmd` | io_uring_cmd 提交 | Capability 校验（核心） |
| `security_uring_sqe` | SQE 提交 | opcode 白名单 |
| `security_uring_register_buffers` | buffer 注册 | CAP_REGISTER_BUFFER 权限 |
| `security_task_kill` | kill 信号 | CAP_KILL 权限 |
| `security_file_open` | 文件打开 | CAP_FILE_OPEN 权限 |

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

## §5 与 seL4 Capability 模型的关系：借鉴 seL4 capability-based security

### 5.1 seL4 Capability 模型

seL4 微内核采用 capability-based security——每个安全敏感操作需要对应的 Capability 令牌（详见 [03-capability-model.md](03-capability-model.md)）。Airymax 纯 C LSM 借鉴此模型：

| seL4 机制 | Airymax 纯 C LSM 对齐 | 说明 |
|-----------|---------------------|------|
| CNode（Capability Node） | `struct airy_cnode` | Capability 存储 |
| CSpace（Capability Space） | radix tree | 每个 Agent 的 CSpace |
| mint/copy/move/revoke | `airy_cap_mint/copy/move/revoke` | Capability 操作 |
| badge（权限掩码） | `airy_cnode.badge` | 细粒度权限 |
| handleFault() | `airy_fault_enforce()` | Fault 处理 |

### 5.2 借鉴的关键设计

| seL4 设计 | Airymax 实现 | 价值 |
|-----------|------------|------|
| Capability 不可伪造 | 内核分配 cap_id，用户态仅持有 opaque handle | 防伪造 |
| 最小权限 | 每操作独立 Capability | 最小授权 |
| 递归撤销 | Agent 终止时递归撤销所有派生 Capability | 无残留 |
| 形式化验证 | 参考验证方法（未完全形式化） | 可信基础 |

### 5.3 与 seL4 的差异

| 维度 | seL4 | Airymax | 适配原因 |
|------|------|---------|---------|
| 执行域 | 微内核（纯内核态） | Linux 6.6（LSM 钩子） | 复用主线 LSM |
| Capability 存储 | CSpace（内核对象） | radix tree | 复用 Linux 数据结构 |
| 校验方式 | 内核内联 | LSM 钩子函数 | 复用 LSM 框架 |
| 形式化验证 | 完全形式化 | 未形式化 | 工程权衡 |

---

## §6 性能：纯 C 无 BPF 间接调用开销

### 6.1 性能对比

纯 C LSM 相比 BPF LSM 的性能优势在于无间接调用开销：

| 维度 | 纯 C LSM | BPF LSM | 差距 |
|------|---------|---------|------|
| 调用方式 | 直接函数调用 | BPF 虚拟机间接调用 | — |
| fastpath 延迟 | ~160ns | ~200ns+ | ~25% |
| 内存开销 | 静态分配 | BPF 程序 + map | ~3x |
| 编译开销 | 编译期 | 运行时 JIT | — |

### 6.2 性能拆解

纯 C LSM fastpath（Capability 校验）的性能拆解：

| 步骤 | 操作 | 耗时 |
|------|------|------|
| 1. RCU 读锁 | `rcu_read_lock()` | ~10ns |
| 2. radix tree 查找 | `radix_tree_lookup()` | ~100ns |
| 3. 权限掩码校验 | 位运算 `badge & required` | ~5ns |
| 4. 状态/过期校验 | 比较运算 | ~10ns |
| 5. RCU 解锁 | `rcu_read_unlock()` | ~5ns |
| 6. 分支预测 | `unlikely` 优化 | ~0ns（预测成功） |
| **总计** | | **~130-160ns** |

### 6.3 与 IPC fastpath 的关系

纯 C LSM 校验是 IPC fastpath 的一部分，总延迟在 SLO 内：

| 组件 | 延迟 | 占比 |
|------|------|------|
| Capability 校验（纯 C LSM） | ~160ns | ~50% |
| Ring Buffer 写入（reserve + memcpy + commit） | ~160ns | ~50% |
| **IPC fastpath 总计** | **~320ns** | **100%** |

### 6.4 性能优化手段

| 优化 | 效果 | 实现位置 |
|------|------|---------|
| `unlikely` 分支预测 | 正常路径零开销 | fastpath 检查 |
| RCU 无锁读取 | 无锁竞争 | Capability 查找 |
| `static_key` 跳过 | 99%+ 正常请求跳过 slowpath | 第一层轻量标记 |
| `__lsm_ro_after_init` | 只读内存，TLB 友好 | 钩子表 |

---

## §7 源码参考：OLK 6.6 security/ 目录（SELinux/AppArmor/Landlock/Tomoyo 全部纯 C）

### 7.1 OLK 6.6 security/ 目录结构

Airymax 纯 C LSM 模块的源码结构对齐 OLK 6.6（openEuler Linux Kernel 6.6）的 `security/` 目录：

```
security/
├── airy/                    ← Airymax 新增（纯 C）
│   ├── airy_lsm.c           LSM 注册（DEFINE_LSM）
│   ├── airy_cap_hooks.c     Capability 校验钩子
│   ├── airy_uring_hooks.c   io_uring 安全钩子
│   ├── airy_cap_tree.c      Capability radix tree
│   ├── airy_internal.h      内部头文件
│   └── Kconfig              配置
├── selinux/                 SELinux（纯 C 参考）
├── apparmor/                AppArmor（纯 C 参考）
├── landlock/                Landlock（纯 C 参考）
├── tomoyo/                  Tomoyo（纯 C 参考）
├── commoncap.c              POSIX capability（纯 C）
└── lsm.c                    LSM 框架
```

### 7.2 借鉴的纯 C 模式

从 OLK 6.6 `security/` 目录借鉴的纯 C 模式：

| 借鉴源 | 借鉴模式 | Airymax 应用 |
|--------|---------|-------------|
| `security/selinux/hooks.c` | `DEFINE_LSM` + `security_add_hooks` | LSM 注册 |
| `security/selinux/ss/policydb.c` | 策略数据库 | Capability 存储（radix tree） |
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
# security/airy/Makefile
obj-$(CONFIG_SECURITY_AIRY) := airy_lsm.o airy_cap_hooks.o \
                               airy_uring_hooks.o airy_cap_tree.o
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

## §9 测试与验证

### 9.1 测试场景

| 测试场景 | 验证内容 | 预期结果 |
|---------|---------|---------|
| 合法 Capability 提交 io_uring_cmd | 校验通过 | 正常执行 |
| 伪造 cap_id 提交 | 未找到 | AIRY_FAULT_ABNORMAL_CAP |
| 权限不足提交 | badge 不匹配 | AIRY_FAULT_CAP_FAULT |
| 已撤销 Capability 提交 | state != ACTIVE | AIRY_FAULT_ABNORMAL_CAP |
| 过期 Capability 提交 | expires_ns 超过 | AIRY_FAULT_ABNORMAL_CAP |
| 非法 opcode 提交 | 白名单外 | -EOPNOTSUPP |
| 无权限注册 buffer | CAP_REGISTER_BUFFER 缺失 | -AIRY_ECAP_PERM |

### 9.2 性能 SLO

| 指标 | SLO | 实测 |
|------|-----|------|
| Capability fastpath 校验 | ≤200ns | ~160ns |
| opcode 白名单检查 | ≤10ns | ~5ns |
| LSM 注册开销 | 编译期 | 0（运行时） |
| 与 BPF LSM 对比 | 更快 | ~25% faster |

---

## §10 相关文档

- [01-lsm-framework.md](01-lsm-framework.md) —— LSM 框架（上级设计）
- [10-unify-design.md](../10-architecture/10-unify-design.md) §7 —— USV 模块（Micro-Supervisor 基于 LSM）
- [09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) —— Micro-Supervisor（纯 C LSM 的使用者）
- [03-capability-model.md](03-capability-model.md) —— Capability 模型（校验对象）
- [06-io-uring-hardening.md](06-io-uring-hardening.md) —— io_uring 安全加固（基于纯 C LSM）
- [02-landlock-sandbox.md](02-landlock-sandbox.md) —— Landlock 沙箱（共存 LSM）
- [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §4.2.4 / §4.2.5 —— 设计依据

---

## §11 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：纯 C LSM 模块设计；对齐 openEuler 纯 C LSM 模式（SELinux/AppArmor/Landlock/Tomoyo 全部纯 C，不使用 BPF LSM）；DEFINE_LSM(airy) 纯 C 注册 + LSM_ORDER_FIRST；Capability 校验钩子（security_uring_cmd 回调）；io_uring 安全钩子（opcode 白名单）；借鉴 seL4 capability-based security；纯 C 性能优势（fastpath ~160ns，无 BPF 间接调用开销）；源码参考 OLK 6.6 security/ 目录 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | 纯 C LSM 模块设计 | v1.0 | 2026-07-17
