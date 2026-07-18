Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [DSL] 降级生存层统一框架
> **文档定位**：IRON-9 v3 第四层 [DSL]（Degraded Survival Layer）的唯一权威设计框架\
> **文档版本**：v1.1（Capability Folding H6 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[Airymax Unify Design 总纲](10-unify-design.md)\
> **设计依据**：Airymax Unify Design §9 [DSL] 降级策略 + A-IPC Capability Folding H6 硬约束

---

## SSoT 声明

> **单一权威源声明**：本文件是 **[DSL] 降级生存层** 的唯一权威源。`#ifdef AIRY_SC_FALLBACK` 降级块机制、Panic 回退路径、最小可运行子集定义、与 IRON-9 v3 四层模型的关系、**Capability Folding H6 降级落地**均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 [DSL] 降级策略。
>
> 本文件遵循 **sched_tac**（降级时回退到 EEVDF 默认调度）、**纯 C LSM**（不依赖 BPF）、**alloc_pages + mmap**（不依赖 DMA 一致性内存）技术选型。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。
>
> **v1.1 Capability Folding H6 集成声明**：[DSL] 降级模式下，IPC 消息头 `capability_badge=0`，fastpath C-S9 跳过 Badge 校验（直接 `goto cap_pass`），IPC 仍可使用但仅保留 POSIX capability 校验（slowpath `airy_cap_check()`）。这是 6 条硬约束 H6 的落地，详见 §4.4。

---

## 文档信息卡

- **目标读者**：agentrt-linux 内核开发者、可靠性工程师、构建系统维护者
- **前置知识**：理解 [10-unify-design.md](10-unify-design.md) Unify Design 总纲、[06-iron9-shared-model.md](06-iron9-shared-model.md) IRON-9 v3 四层模型
- **预计阅读时间**：25 分钟
- **核心概念**：[DSL]、AIRY_SC_FALLBACK、最小可运行子集、Panic 回退、printk_safe
- **复杂度标识**：高级

---

## §1 设计目标：[SC] 头文件损坏/缺失时的最小可运行子集

### 1.1 问题背景

Airymax Unify Design 的 [SC] 共享契约层（物理宿主 `kernel/include/airymax/`，共 10 个头文件）是 agentrt 与 agentrt-linux 双端逐字节共享的权威源。但在以下极端场景下，[SC] 头文件可能不可用或损坏：

| 场景 | 触发原因 | 影响 |
|------|---------|------|
| 头文件版本漂移 | 双端未同步更新导致 `error.h` 不一致 | 编译失败或运行时码值错位 |
| 头文件物理损坏 | 磁盘错误、构建产物污染 | 编译失败 |
| 最小化构建环境 | 嵌入式/恢复模式下仅保留内核最小子集 | [SC] 头文件未部署 |
| CI 双端校验失败 | `sc-dual-ci.yml` 检测到两端不一致 | 必须降级构建以保持可运行 |

传统设计在上述场景下直接编译失败，导致系统无法启动。Airymax 引入 [DSL] 降级生存层，确保在 [SC] 头文件损坏/缺失时，系统仍具备**最小可运行子集**——能启动、能打印日志、能完成基本 IPC、能安全 Panic。

### 1.2 设计目标

[DSL] 降级生存层的核心设计目标有三：

1. **永不 brick**：[SC] 头文件任何形式的损坏/缺失都不能导致系统无法启动
2. **最小可用**：降级后保留 POSIX 兼容的最小功能子集，能完成基本运维操作
3. **显式可见**：降级状态必须通过编译告警 + 运行时日志显式告知，禁止静默降级

### 1.3 与正常路径的关系

[DSL] 不是正常路径的替代，而是**最后防线**。正常运行时 [SC] 头文件完整可用，[DSL] 降级块不生效。仅当编译时定义 `AIRY_SC_FALLBACK` 宏（或构建系统检测到 [SC] 校验失败自动注入）时，[DSL] 降级块才生效。

```
正常路径                         降级路径（[DSL]）
┌──────────────────────┐        ┌──────────────────────┐
│ [SC] error.h 完整    │        │ [SC] error.h 损坏    │
│ 5 个码空间（300 码） │  ──┐   │ 仅 38 个 POSIX 码    │
│ Ring Buffer 日志     │    │   │ printk_safe 原生     │
│ sched_tac 三层    │    └──▶│ EEVDF 默认调度       │
│ IORING_OP_URING_CMD  │        │ 最简 IPC 消息头      │
│ 纯 C LSM 完整校验    │        │ Capability 跳过      │
└──────────────────────┘        └──────────────────────┘
```

---

## §2 #ifdef AIRY_SC_FALLBACK 机制：每个 [SC] 头文件底部的降级块设计

### 2.1 降级块结构

每个 [SC] 头文件底部均包含一个 `#ifdef AIRY_SC_FALLBACK` 降级块。降级块的设计遵循三原则：

1. **自包含**：降级块不依赖任何外部头文件，仅使用标准 C 预处理与编译器内置类型
2. **最小集**：降级块仅定义维持系统启动所需的最小符号集
3. **显式告警**：降级块生效时必须发出 `#warning`，告知开发者当前为降级模式

```c
/* kernel/include/airymax/error.h —— 文件底部 */
#ifdef AIRY_SC_FALLBACK
/*
 * [DSL] 降级生存层 —— AIRY_SC_FALLBACK 激活
 * 仅保留 38 个 POSIX errno 负值 + 1 个配置版本码
 * 禁止在此定义 [SC]/IPC/Capability/[DSL] 扩展码
 */
#ifndef _AIRY_ERROR_FALLBACK_H
#define _AIRY_ERROR_FALLBACK_H

#define AIRY_EOK            0       /* 成功 */
#define AIRY_EPERM          (-1)    /* 权限不足 */
#define AIRY_ENOENT         (-2)    /* 不存在 */
#define AIRY_EINTR          (-4)    /* 被信号中断 */
#define AIRY_EIO            (-5)    /* I/O 错误 */
#define AIRY_ENXIO          (-6)    /* 无此设备 */
#define AIRY_E2BIG          (-7)    /* 参数列表过长 */
#define AIRY_ENOEXEC        (-8)    /* exec 格式错误 */
#define AIRY_EBADF          (-9)    /* 坏文件描述符 */
#define AIRY_ECHILD         (-10)   /* 无子进程 */
#define AIRY_EAGAIN         (-11)   /* 重试 */
#define AIRY_ENOMEM         (-12)   /* 内存不足 */
#define AIRY_EACCES         (-13)   /* 拒绝访问 */
#define AIRY_EFAULT         (-14)   /* 错误地址 */
#define AIRY_EBUSY          (-16)   /* 设备忙 */
#define AIRY_EEXIST         (-17)   /* 已存在 */
#define AIRY_EXDEV          (-18)   /* 跨设备链接 */
#define AIRY_ENODEV         (-19)   /* 无此设备 */
#define AIRY_ENOTDIR        (-20)   /* 非目录 */
#define AIRY_EISDIR         (-21)   /* 是目录 */
#define AIRY_EINVAL         (-22)   /* 参数无效 */
#define AIRY_ENFILE         (-23)   /* 系统文件表满 */
#define AIRY_EMFILE         (-24)   /* 文件描述符表满 */
#define AIRY_ENOTTY         (-25)   /* 非 tty */
#define AIRY_ETXTBSY        (-26)   /* 文本忙 */
#define AIRY_EFBIG          (-27)   /* 文件过大 */
#define AIRY_ENOSPC         (-28)   /* 空间不足 */
#define AIRY_ESPIPE         (-29)   /* 不可 seek */
#define AIRY_EROFS          (-30)   /* 只读文件系统 */
#define AIRY_EMLINK         (-31)   /* 链接过多 */
#define AIRY_EPIPE          (-32)   /* 管道破裂 */
#define AIRY_EDOM           (-33)   /* 域错误 */
#define AIRY_ERANGE         (-34)   /* 范围错误 */
#define AIRY_EDEADLK        (-35)   /* 死锁 */
#define AIRY_ENAMETOOLONG   (-36)   /* 文件名过长 */
#define AIRY_ENOLCK         (-37)   /* 无可用锁 */
#define AIRY_ENOSYS         (-38)   /* 功能未实现 */
#define AIRY_ENOTEMPTY      (-39)   /* 目录非空 */
#define AIRY_ELOOP          (-40)   /* 符号链接循环 */
/* [DSL] 唯一保留的非 POSIX 码：配置版本不匹配 */
#define AIRY_ECFGVERSION    (-101)

/* Fault 码在降级模式下全部禁用，所有故障统一走 Panic 路径 */

#warning "AIRY_SC_FALLBACK active: only 38 POSIX codes + AIRY_ECFGVERSION available, Fault codes disabled"

#endif /* _AIRY_ERROR_FALLBACK_H */
#endif /* AIRY_SC_FALLBACK */
```

### 2.2 各头文件降级块职责

10 个 [SC] 头文件各自的降级块职责如下：

| 头文件 | 降级块职责 | 保留符号 |
|--------|----------|---------|
| `error.h` | 38 个 POSIX 码 + `AIRY_ECFGVERSION` | 39 个错误码 |
| `log_types.h` | 仅 `LOG_FATAL` + `LOG_ERROR` 两级 | 2 个日志级别 |
| `ipc.h` | 最简 128B 消息头（magic + opcode + payload_len + **capability_badge=0**） | 4 个字段（H6 落地） |
| `sched.h` | 仅 `AIRY_TASK_MAGIC` + `MAC_MAX_AGENTS` | 2 个符号 |
| `memory_types.h` | 仅 L1 记忆结构 | 1 个结构 |
| `security_types.h` | 仅 POSIX 41 个 capability + **Badge 访问宏**（H6 落地） | 41 个 ID + 3 个宏 |
| `cognition_types.h` | 仅 `airy_cog_phase` 三阶段枚举 | 1 个枚举 |
| `uapi_compat.h` | 仅 `__KERNEL__` / `__linux__` 两路桥接 | 2 路分支 |
| `syscalls.h` | **v1.1: 仅 4 核心 syscall 编号**（512-515，原 12 编号已废弃） | 4 个编号 |
| `lsm_types.h` | 仅 `DEFINE_LSM(airy)` 最小骨架 | 1 个宏 |

**v1.1 变更说明**：
- `ipc.h` 降级块新增 `capability_badge=0` 字段（H6 落地），保证 Layout C v4 二进制布局兼容
- `security_types.h` 降级块新增 `AIRY_BADGE_EPOCH/RANDTAG/PERMS` 三个访问宏，保证 [DSL] 模式下 Badge 字段提取编译通过（但运行时跳过校验）
- `syscalls.h` 降级块从 12 编号缩减为 4 编号（Capability Folding syscall 12→4 精确映射）

### 2.3 构建系统自动注入

构建系统在以下情况自动注入 `AIRY_SC_FALLBACK`：

1. `sc-dual-ci.yml` 检测到两端 [SC] 头文件不一致
2. `airy_defconfig` 中显式配置 `CONFIG_AIRY_SC_FALLBACK=y`
3. [SC] 头文件物理校验失败（hash 不匹配）

```makefile
# kernel/Kconfig
config AIRY_SC_FALLBACK
    bool "Airymax [SC] Fallback (Degraded Survival Layer)"
    default n
    help
      启用 [DSL] 降级生存层。当 [SC] 共享契约头文件损坏或缺失时，
      系统降级为最小可运行子集。正常情况下应保持 n。
      详见 docs/AirymaxOS/10-architecture/11-degraded-survival-layer.md
```

---

## §3 Panic 回退路径：printk_safe 原生路径

### 3.1 为什么需要 Panic 回退

当系统进入 Panic 状态时，A-ULP 的 Ring Buffer 可能已损坏（内存损坏导致元数据不一致），或 Logger Daemon 已死亡（无法消费 Ring Buffer）。此时若仍尝试写入 Ring Buffer，可能导致递归 Panic 或死锁。[DSL] 规定 Panic 路径必须回退到 Linux 6.6 原生 `printk_safe` 路径。

### 3.2 printk_safe 机制（对齐 Linux 6.6 NMI-safe buffer）

Linux 6.6 的 `printk_safe` 提供一个 NMI-safe 的 per-CPU 临时缓冲区，专为 Panic/NMI/递归打印场景设计。其特性：

- **无锁**：per-CPU 缓冲区，无需锁，NMI 上下文安全
- **无依赖**：不依赖任何内核子系统（调度器、内存分配、RCU）
- **延迟刷新**： Panic 时通过 `printk_safe_flush()` 将缓冲区内容刷入主 log buffer
- **大小有限**：每 CPU 8192 字节，足以容纳 Panic 栈

Airymax 的 Panic 回退路径直接调用 `printk_safe`：

```c
/* kernel/log/airy_log_ring.c —— Panic 回退 */
static void airy_log_panic(const char *fmt, ...)
{
    /* 1. 尝试写入 Ring Buffer（best-effort，失败不阻塞） */
    if (airy_log_ring_try_write(fmt, ...) < 0) {
        /* 2. Ring 不可用，回退到 printk_safe */
        printk_safe_flush_on_panic();
        vprintk(fmt, args);   /* 走 printk_safe NMI-safe 路径 */
    }
    /* 3. 触发 Panic */
    panic("Airymax fatal fault");
}
```

### 3.3 [DSL] 降级模式下的日志路径

在 `AIRY_SC_FALLBACK` 激活时，A-ULP 完全不初始化 Ring Buffer，所有日志直接走 `printk` 原生路径。此时性能从 ~50-100ns 退化为传统 printk 的 ~5-10μs，但保证了可运行性——这正是 [DSL] "永不 brick"原则的体现。

| 模式 | 日志路径 | 单条延迟 | 适用场景 |
|------|---------|---------|---------|
| 正常 | Ring Buffer + Logger Daemon | ~50-100ns | 生产环境 |
| [DSL] 降级 | printk 原生 | ~5-10μs | [SC] 损坏的恢复模式 |
| Panic | printk_safe NMI-safe | ~1-2μs | Panic/NMI 上下文 |

---

## §4 最小可运行子集：仅保留 38 个 POSIX 码、最简 IPC 消息头、EEVDF 默认调度

### 4.1 最小可运行子集定义

[DSL] 降级模式下的最小可运行子集覆盖四个维度：

#### 4.1.1 错误码子集

仅保留 38 个 POSIX errno 负值（`AIRY_EPERM=-1` ~ `AIRY_ELOOP=-40`）+ 1 个配置版本码 `AIRY_ECFGVERSION=-101`。Fault 码全部禁用——降级模式下所有不可恢复故障统一走 Panic，不尝试通过 Fault Handler 优雅处理。

#### 4.1.2 IPC 消息头子集

最简 128B 消息头保留 4 个字段（v1.1: 新增 `capability_badge=0` 字段，H6 落地），其余字段置零：

```c
/* 降级模式 IPC 消息头（v1.1: Layout C v4 兼容，capability_badge=0）*/
struct airy_ipc_msg_hdr_min {
    __u32   magic;              /* offset  0, 'ARE1' 0x41524531 */
    __u16   opcode;             /* offset  4 */
    __u32   payload_len;        /* offset 40 */
    __u64   capability_badge;   /* offset 44, v1.1 新增: 固定为 0（H6 硬约束）*/
    /* 其余 110 字节置零（trace_id/timestamp_ns/src_task/dst_task/crc32/reserved）*/
} __attribute__((packed));
_Static_assert(offsetof(struct airy_ipc_msg_hdr_min, capability_badge) == 44,
               "H1: capability_badge offset must be 44");
```

降级模式下 IPC 仅支持 `AIRY_IPC_OP_SEND` / `AIRY_IPC_OP_RECV` 两个操作，不支持 `FREEZE` / `CAP_REQUEST` 等高级操作。fastpath C-S9 检测到 `capability_badge=0` 时直接 `goto cap_pass` 跳过 Badge 校验（H6 落地）。

#### 4.1.3 调度子集

降级模式下不依赖 `SCHED_DEADLINE` / `SCHED_FIFO` 配置，所有 Agent 统一使用 Linux 6.6 默认的 **EEVDF 调度器**（`SCHED_NORMAL` + nice）。sched_tac 的三层调度在 [SC] 头文件完整时才启用。

#### 4.1.4 日志子集

降级模式下 A-ULP 不初始化 Ring Buffer，日志级别仅保留 `LOG_FATAL` + `LOG_ERROR` 两级，直接走 `printk` 原生路径。

### 4.2 降级模式的能力边界

降级模式下明确**禁用**以下能力：

| 禁用能力 | 原因 | 替代方案 |
|---------|------|---------|
| Fault Handler | Fault 码禁用 | 统一 Panic |
| IPC 队列冻结 | 降级无 FREEZE 操作 | 直接 kill 进程 |
| **Capability Badge 完整校验**（v1.1 H6） | `capability_badge=0` 固定，fastpath C-S9 跳过 | 仅 POSIX capability（slowpath `airy_cap_check()`） |
| Ring Buffer 日志 | 内存不可靠 | printk 原生 |
| Macro-Supervisor 裁决 | io_uring_cmd 不可靠 | 内核 watchdog 直接重启 |
| RCU 热重载 | 配置版本不可靠 | 仅默认配置 |
| **Badge 编译/撤销**（v1.1 H6） | sec_d 不可达，`airy_sys_call` 暂停 | 恢复后批量处理 CAP_REQUEST 队列 |

### 4.3 降级模式的退出

降级模式是临时状态，恢复正常路径的流程：

1. 运维修复 [SC] 头文件（重新部署或 git checkout）
2. `sc-dual-ci.yml` 校验通过
3. 重新编译内核（去掉 `AIRY_SC_FALLBACK`）
4. 重启系统，进入正常路径

### 4.4 [DSL] 模式下 Capability Folding Badge 处理（v1.1 新增——H6 落地）

**H6 硬约束**：[DSL] 降级 `capability_badge=0` 跳过 C-S9。这是 Capability Folding 6 条硬约束的最后一条，确保 [SC] 头文件损坏时 IPC 仍可用。

#### 4.4.1 Badge 字段处理

[DSL] 模式下，IPC 消息头 `capability_badge` 字段固定为 0：

```c
/* [DSL] 模式下 fastpath C-S9 Badge 校验逻辑 */
static inline int airy_cap_badge_ok_dsl(u64 badge, u16 opcode)
{
    /* H6: [DSL] 降级模式，capability_badge=0，直接跳过 Badge 校验 */
    if (badge == 0) {
        /* CAP_CARRY 标志下不允许 badge=0（防绕过，与正常模式一致）*/
        if (unlikely(airy_ipc_hdr_flags_cap_carry()))
            return -AIRY_ECAP_BADGE;  /* -78 */
        goto cap_pass;  /* 跳过 Epoch/RandomTag/Perms 校验 */
    }
    /* [DSL] 模式下 badge 非 0 视为异常（sec_d 不可达，无法编译 Badge）*/
    return -AIRY_ECAP_BADGE;  /* -78 */

cap_pass:
    return 0;
}
```

#### 4.4.2 [DSL] 与正常模式对比

| 维度 | 正常模式 | [DSL] 降级模式 |
|------|---------|--------------|
| `capability_badge` | 64-bit Native Word（Epoch + RandomTag + Perms） | 固定为 0 |
| fastpath C-S9 | 3 个 READ_ONCE + 位运算（~10ns） | 直接 `goto cap_pass`（~1ns） |
| Capability 校验 | Badge 完整校验（Epoch/RandomTag/Perms） | 跳过，仅 POSIX cap（slowpath） |
| sec_d 状态 | 在线，可编译/撤销 Badge | 不可达，暂停服务 |
| `airy_cap_global_epoch` | 撤销时 atomic_inc | 冻结，避免误失效 |
| IPC 可用性 | 完整（7 opcode） | 仅 SEND/RECV（2 opcode） |
| CAP_REQUEST 自举 | 支持（首次 Badge 申请） | 不支持（sec_d 不可达） |

#### 4.4.3 [DSL] 模式下 IPC 流程

```
[DSL] 模式下 Agent 发送 IPC 消息:

1. Agent 填充消息头:
   - magic = 0x41524531 ('ARE1')
   - opcode = AIRY_IPC_OP_SEND (0x0001)
   - capability_badge = 0  (H6: 固定为 0)
   - flags = 0  (不允许 CAP_CARRY)
   - src_task / dst_task / payload_len 正常填充

2. 提交 io_uring SQE（IORING_OP_URING_CMD）

3. 内核 fastpath:
   - C-S0~C-S8: magic/opcode/flags/src_task/dst_task/payload_len 校验
   - C-S9: airy_cap_badge_ok_dsl(badge=0, opcode) → goto cap_pass（H6 跳过）
   - C-S10~C-S12: CRC32 校验（若 [SC] ipc.h 降级块保留 crc32 字段）

4. 投递消息（airy_ipc_deliver_fast）

5. 接收方:
   - 仅校验 POSIX capability（slowpath airy_cap_check）
   - 不执行 Badge 派生（CAP_CARRY 标志禁用）
```

#### 4.4.4 [DSL] 退出时的 Badge 恢复

[DSL] 模式退出（[SC] 头文件修复 + 重启）后，Badge 恢复流程：

1. 系统重启，[SC] 头文件完整
2. sec_d 启动，初始化 `agent_caps[1024]` 静态数组
3. sec_d 重置 `airy_cap_global_epoch = 0`
4. Agent 通过 CAP_REQUEST opcode 向 sec_d 申请新 Badge
5. sec_d 调用 `airy_sys_call + COMPILE_BADGE` 编译 Badge
6. fastpath C-S9 恢复完整校验（Epoch + RandomTag + Perms）

---

## §5 与 IRON-9 v3 的关系：[SC]/[SS]/[IND] + 新增 [DSL] 第四层

### 5.1 IRON-9 演进路径

[DSL] 是 IRON-9 v3 新增的第四层，详见 [06-iron9-shared-model.md](06-iron9-shared-model.md)。IRON-9 演进路径：

| 版本 | 层级数 | 层级构成 | 核心问题 |
|------|--------|---------|---------|
| v1 | 2 层 | 共享 + 独立 | 缺少语义同源层 |
| v2 | 3 层 | [SC] + [SS] + [IND] | 缺少降级生存层 |
| **v3** | **4 层** | **[SC] + [SS] + [IND] + [DSL]** | **新增 [DSL] 第四层** |

### 5.2 [DSL] 在四层模型中的定位

| 层级 | 全称 | 共享程度 | [DSL] 关系 |
|------|------|---------|-----------|
| [SC] | Shared Contract | 物理共享，逐字节相同 | [DSL] 是 [SC] 的降级降级回退 |
| [SS] | Semantic Sibling | 语义同源，签名独立 | [DSL] 不影响 [SS]（[SS] 本就独立） |
| [IND] | Independent | 完全独立 | [DSL] 不影响 [IND] |
| **[DSL]** | **Degraded Survival** | **自包含降级块** | **[SC] 损坏时的最小可运行子集** |

### 5.3 [DSL] 的独立性

[DSL] 降级块虽然在 [SC] 头文件内部（`#ifdef AIRY_SC_FALLBACK`），但其设计是**自包含**的——不依赖 [SC] 的任何其他符号，仅依赖标准 C 预处理与编译器内置类型。这意味着即使 [SC] 头文件主体损坏，只要降级块本身完整（通过 hash 校验），系统仍可降级启动。

为保障降级块本身的完整性，构建系统对每个 [SC] 头文件的 `#ifdef AIRY_SC_FALLBACK ... #endif` 区域单独计算 hash，并存入 `kernel/include/airymax/.fallback_hashes`。CI 校验时不仅校验 [SC] 主体，也校验降级块 hash。

---

## §6 相关文档

- [10-unify-design.md](10-unify-design.md) —— Airymax Unify Design 总纲（[DSL] 是其韧性领域的最后防线，v1.1 §9 [DSL] ipc.h 降级块）
- [06-iron9-shared-model.md](06-iron9-shared-model.md) —— IRON-9 v3 四层模型（[DSL] 是第四层）
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) §5 —— error.h [DSL] 降级块细节
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— log_types.h 契约（降级时仅 2 级）
- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) §2.6 —— ipc.h Layout C v4 契约（capability_badge 字段）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— IPC fastpath C-S9 Badge 校验
- [110-security/03-capability-model.md](../110-security/03-capability-model.md) §12.4 —— Capability Folding [DSL] 降级
- [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §6 —— Panic 回退路径

---

## §7 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：[DSL] 降级生存层统一框架；`#ifdef AIRY_SC_FALLBACK` 机制；Panic 回退到 printk_safe；最小可运行子集（38 POSIX 码 + 最简 IPC + EEVDF 默认调度）；IRON-9 v3 第四层定义 |
| **v1.1** | **2026-07-18** | **Capability Folding H6 集成版**：(1) §2.2 各头文件降级块职责表更新——ipc.h 新增 `capability_badge=0` 字段，security_types.h 新增 Badge 访问宏，syscalls.h 从 12 编号缩减为 4 编号；(2) §4.1.2 IPC 消息头子集更新为 Layout C v4 兼容版（4 个字段含 capability_badge=0）；(3) §4.2 降级模式能力边界表新增 Badge 校验跳过 + Badge 编译/撤销暂停；(4) §4.4 新增 [DSL] 模式下 Capability Folding Badge 处理（H6 落地详细说明，含 Badge 字段处理/正常 vs [DSL] 对比/IPC 流程/Badge 恢复）；(5) 清除所有内部审查路径引用 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [DSL] 降级生存层统一框架 | v1.1 | 2026-07-18
