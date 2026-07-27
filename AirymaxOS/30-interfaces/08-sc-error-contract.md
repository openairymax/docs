Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] error.h 二进制契约
> **文档定位**：A-UEF（统一错误码与故障定义体系）模块的 [SC] 共享契约权威定义\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §4\
> **设计依据**：本契约为 SSoT（Single Source of Truth），错误码值域、命名风格、[DSL] 降级块均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax 错误码与故障码体系** 的唯一权威源。Error 码空间分配、Fault 码空间分配、`AIRY_E*` 命名风格、`AIRY_FAULT_*` 命名风格、[DSL] 降级块均以本文件为唯一权威定义。物理宿主为 `kernel/include/uapi/linux/airymax/error.h`，agentrt 与 agentrt-linux 物理共享同一份头文件，逐字节相同。
>
> **Capability Folding 工程定义**（v1.0.1 新增）：A-IPC 采用 Capability Folding 设计模式——将 capability check（能力校验）从独立的控制面操作"折叠"到 IPC 数据面消息传递 fastpath 中。其物理载体是 [SC] `ipc.h` Layout C v4 消息头 offset 40-47 的 `capability_badge` 字段（64-bit Native Word：Epoch + Random Tag + Perms）；其执行点是 fastpath C-S9 内联校验（`airy_cap_badge_ok()`，~10ns，3 个 READ_ONCE + 位运算 + 比较）。本契约新增的 `-78~-82` Capability 错误码与 `0x1001-0x1003` Fault 码是 Capability Folding 的错误语义 SSoT。
>
> 废弃风格声明：`AIRY_ERR_*`（详细前缀）已废弃，全部迁移为 `AIRY_E*`；`EAIRY_*`（errno 风格前缀）已废弃。本契约遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) 的技术选型（sched_tac + 纯 C LSM 不使用 BPF LSM + IORING_OP_URING_CMD + registered buffer + mmap 不使用 page flipping + alloc_pages + mmap 不使用 DMA 一致性内存）。

---

## 文档信息卡

- **目标读者**：agentrt-linux 内核开发者、agentrt 用户态开发者、CI 维护者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) A-UEF 模块定位、[06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) [SC] 层概念、[02-ipc-protocol.md](02-ipc-protocol.md) Layout C v4 消息头定义
- **预计阅读时间**：30 分钟
- **核心概念**：Error（负数可恢复）、Fault（正数 0x1000+ 不可恢复）、[SC] 契约、CI 逐字节校验、Capability Folding Badge 校验
- **复杂度标识**：中级

---

## §1 契约概述：A-UEF 模块的 [SC] 共享契约

A-UEF（Unified Error and Fault Framework，统一错误码与故障定义体系）是 Airymax Unify Design 的基础模块，其 [SC] 共享契约头文件 `error.h` 是整个体系最底层的数据契约。

### 1.1 契约范围

本契约定义以下内容，agentrt 与 agentrt-linux 双端必须逐字节一致：

1. `airy_err_t` 类型定义（错误码类型）
2. `AIRY_E*` 错误码宏（Error 码，负数空间）
3. `AIRY_FAULT_*` 故障码宏（Fault 码，正数 0x1000+ 空间）
4. `AIRY_LOG_MAGIC` 等相关魔数（仅与错误码语义绑定的魔数）
5. `#ifdef AIRY_SC_FALLBACK` 降级块

### 1.2 物理宿主

| 属性 | 值 |
|------|-----|
| 物理路径 | `kernel/include/uapi/linux/airymax/error.h` |
| 共享层级 | [SC] 共享契约层（IRON-9 v3 第一层） |
| 共享方式 | 物理共享，逐字节相同 |
| CI 校验 | `sc-dual-ci.yml` 逐字节校验 |
| agentrt 引用方式 | `-I` 编译选项 + `#include <airymax/error.h>` |

### 1.3 类型约束

遵循 [SC] 共享契约层通用约束：

- 使用内核 UAPI 类型 `int32_t` / `__s32`（不使用 `int`）
- 禁止使用 `float` 类型
- 禁止依赖任何非 UAPI 头文件
- 禁用 `__attribute__((packed))`，改用 `__aligned(64)` 或自然对齐（错误码宏不涉及结构体，无需对齐属性）

### 1.4 Capability Folding 错误码的 [SC]/[SS] 边界

基于 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) IRON-9 v3 四层模型，Capability Folding 相关错误码的层级归属：

| 层级 | 内容 | 物理宿主 |
|------|------|---------|
| [SC] 共享契约层 | 错误码**定义**（`-78~-82`、`0x1001-0x1003`） | 本文件 + `error.h` |
| [SS] 语义同源层 | 错误码**触发逻辑**（C-S9 Badge 校验、Fault 触发条件） | [07-ipc-fastpath.md](07-ipc-fastpath.md) + [03-capability-model.md](../110-security/03-capability-model.md) |
| [IND] 完全独立层 | agentrt 用户态：`capability_badge` 始终为 0，不触发 `-78~-82` | [01-syscalls.md](01-syscalls.md) §1.4 |
| [DSL] 降级生存层 | `capability_badge=0`，跳过 C-S9，使用 `-76/-77` 兜底 | [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2.2 |

**关键不变量**：[SC] 层共享的是错误码**数据定义**（值与名称），不是**触发语义**。两端对同一错误码的触发条件可能不同（agentrt-linux 内核由 C-S9 触发，agentrt 用户态不触发），这属于 [SS] 语义同源层范畴，不违反 [SC] 逐字节相同原则。

---

## §2 Error 码空间分配

Error 码占据负数空间 `[-300, -1]`，按来源分 **10 个子空间**（对齐 `error.h` SSoT，P0-I2 修复：原 5 子空间划分遗漏了 Config/A-ULS/MemoryRoVol/Cognition/Log/Object/Syscall 7 个子空间），每个子空间有明确的值域、来源与语义。

### 2.1 Error 码空间总表（10 子空间，对齐 SSoT error.h）

| 子空间 | 值域 | 来源 | 已定义数 | 命名风格 |
|--------|------|------|---------|---------|
| POSIX 码 | `[-1, -40]` | 对齐 Linux errno | 16 | `AIRY_E*`（对齐 `E*` errno 名） |
| IPC 码 | `[-41, -70]` | A-IPC 协议层（Capability Folding fastpath C-S0~C-S12） | 13 | `AIRY_EIPC_*` |
| Capability 码 | `[-71, -100]` | 安全子系统（含 Capability Folding Badge 校验） | 13 | `AIRY_ECAP_*` / `AIRY_ESEC_*` |
| Config 码 | `[-101, -120]` | A-UCS 配置管理 | 5 | `AIRY_ECFG*` |
| A-ULS 码 | `[-121, -140]` | A-ULS 调度/生命周期 | 10 | `AIRY_ESCHED_*` / `AIRY_ELIFECYCLE_*` |
| MemoryRoVol 码 | `[-141, -160]` | MemoryRoVol 内存子系统 | 8 | `AIRY_EMEM_*` |
| Cognition 码 | `[-161, -180]` | A-UCS 认知子系统 | 6 | `AIRY_ECOG_*` |
| Log 码 | `[-181, -200]` | A-ULP 日志子系统 | 6 | `AIRY_ELOG_*` |
| Object 码 | `[-201, -220]` | Airymax Object 系统 | 4 | `AIRY_EOBJ_*` |
| Syscall 码 | `[-221, -240]` | Airymax syscall 表面 | 4 | `AIRY_ESYS_*` |
| 预留 | `[-241, -300]` | 未来扩展 | 0 | — |

> **⚠️ P0-I2 修复说明**（v1.0.1-fix）：原文档声明 5 子空间（POSIX/IPC/Capability/[SC]/[DSL]），与 `error.h` SSoT 实际 10 子空间严重不符。修复后对齐 SSoT 实际定义：移除虚构的 `[SC]` 与 `[DSL]` 错误码子空间（这两个概念属于 [DSL] 降级块机制，不是独立错误码子空间），新增 7 个实际存在的子空间（Config/A-ULS/MemoryRoVol/Cognition/Log/Object/Syscall）。

### 2.2 POSIX 码 `[-1, -40]`（对齐 SSoT error.h L28-43）

对齐 Linux 6.6 标准 errno，SSoT `error.h` 实际定义 16 个 POSIX 兼容码（非连续编号，对齐 Linux errno 数值）：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L28-43 —— POSIX 错误码 */
#define AIRY_EOK            0       /* 成功（非负数，特殊） */
#define AIRY_EACCES         (-1)    /* Operation not permitted */
#define AIRY_EEXIST         (-2)    /* File exists */
#define AIRY_EFAULT         (-3)    /* Bad address */
#define AIRY_EINTR          (-4)    /* Interrupted system call */
#define AIRY_EINVAL         (-5)    /* Invalid argument */
#define AIRY_EIO            (-6)    /* I/O error */
#define AIRY_EISDIR         (-7)    /* Is a directory */
#define AIRY_ENOENT         (-8)    /* No such file or directory */
#define AIRY_ENOMEM         (-9)    /* Out of memory */
#define AIRY_ENOSPC         (-10)   /* No space left on device */
#define AIRY_ENOTSUP        (-11)   /* Operation not supported */
#define AIRY_EPERM          (-12)   /* Operation not permitted (POSIX) */
#define AIRY_ERANGE         (-13)   /* Result too large */
#define AIRY_EBUSY          (-16)   /* Device or resource busy */
#define AIRY_ECANCELED      (-19)   /* Operation canceled */
#define AIRY_EAGAIN         (-35)   /* Try again */
```

> **⚠️ P0-I1 修复说明**（v1.0.1-fix）：原文档列举的错误码值（`AIRY_EPERM=-1 / AIRY_ENOENT=-2 / AIRY_EINVAL=-22` 等）与 SSoT `error.h` 严重不符。SSoT 实际定义：`AIRY_EACCES=-1 / AIRY_EEXIST=-2 / AIRY_EFAULT=-3 / AIRY_EINTR=-4 / AIRY_EINVAL=-5 / AIRY_EIO=-6` 等。修复后所有错误码值严格对齐 SSoT `error.h` L28-43 行的源码定义。

### 2.3 IPC 码 `[-41, -70]`

A-IPC 协议层专用错误码，覆盖 IPC 协议、Ring Buffer、io_uring_cmd、fastpath C-S0~C-S12 校验链等场景。v1.0.1 起，IPC 码空间与 [07-ipc-fastpath.md](07-ipc-fastpath.md) 的 C-S 检查链精确对齐：

```c
/* ===== IPC 码空间 [-41, -70] ===== */
/* 与 fastpath C-S0~C-S12 检查链精确对齐（见 07-ipc-fastpath.md §5.2） */
#define AIRY_EIPC_MAGIC        (-41)   /* C-S1:  magic 校验失败 */
#define AIRY_EIPC_OPCODE       (-42)   /* C-S2:  未知 opcode */
#define AIRY_EIPC_PAYLOAD      (-43)   /* C-S3:  payload_len 非法 */
#define AIRY_EIPC_HDRSIZE      (-44)   /* C-S4:  头部大小不等于 128 */
#define AIRY_EIPC_RESERVED     (-45)   /* C-S4:  reserved[72] 非全零 */
#define AIRY_EIPC_FLAGS        (-46)   /* C-S10: flags 非法（保留位非零） */
#define AIRY_EIPC_NOTSUPP      (-47)   /* C-S10: 不支持的 opcode/flag（如 ENCRYPT/COMPRESS） */
#define AIRY_EIPC_KFIFO        (-48)   /* C-S6:  kfifo 入队失败 */
#define AIRY_EIPC_RECLAIM      (-49)   /* C-S7:  reclaim flag 置位 */
#define AIRY_EIPC_CONTEXT      (-50)   /* C-S8:  上下文检查失败（!in_task） */
#define AIRY_EIPC_CRC32        (-51)   /* C-S12: CRC32 校验失败（覆盖 header[0:52) + payload） */
#define AIRY_EIPC_TIMEOUT      (-52)   /* SLOW_SEND 超时 */
/* [-53, -70] 预留 */
```

**v1.0.1 变更说明**：v1.0.1 之前的 IPC 码（`RING_FULL`/`RING_EMPTY`/`REGISTERED`/`URING_CMD` 等）已废弃，替换为与 C-S 检查链精确对齐的命名。由于 0.1.1 阶段 0 行内核代码，此替换无向后兼容负担。

### 2.4 Capability 码 `[-71, -100]`

安全子系统专用错误码，覆盖 Capability 校验、纯 C LSM 检查、Capability Folding Badge 校验等场景。v1.0.1 新增 `-78~-82` 用于 Capability Folding Badge 校验：

```c
/* ===== Capability 码空间 [-71, -100] ===== */

/* 原有 Capability 校验码（v1.0.1 保留） */
#define AIRY_ECAP_MISSING       (-71)   /* Capability 缺失 */
#define AIRY_ECAP_REVOKED       (-72)   /* Capability 已撤销 */
#define AIRY_ECAP_EXPIRED       (-73)   /* Capability 已过期 */
#define AIRY_ECAP_MISMATCH      (-74)   /* Capability 不匹配 */
#define AIRY_ECAP_LSM_DENIED    (-75)   /* 纯 C LSM 拒绝 */

/* [DSL] 降级模式兜底码（v1.0.1 保留，仅 AIRY_SC_FALLBACK 模式触发） */
#define AIRY_ECAP_RADIX_MISS    (-76)   /* [DSL] radix tree 查找失败 */
#define AIRY_ECAP_STATIC_KEY    (-77)   /* [DSL] static_key 禁用 */

/* Capability Folding Badge 校验码（v1.0.1 新增） */
/* 触发位置: fastpath C-S9 内联 airy_cap_badge_ok()，见 07-ipc-fastpath.md §5.2 */
#define AIRY_ECAP_BADGE         (-78)   /* Badge 格式无效、Random Tag 不匹配、CAP_CARRY 但 badge=0 */
#define AIRY_ECAP_EPOCH         (-79)   /* Badge Epoch 与全局 Epoch 不匹配（已撤销或过期） */
#define AIRY_ECAP_FORGED        (-80)   /* 检测到 Badge 伪造尝试（同时触发 AIRY_FAULT_CAP_FORGED） */
#define AIRY_ECAP_PERM          (-81)   /* Badge 权限位不满足 opcode 所需权限 */
#define AIRY_ECAP_FROZEN        (-82)   /* Capability badge 已冻结（badge 撤销，A-ULS 控制，非 C-S0） */

/* sec_d 限流拒绝码（R1 补强：sec_d 滥用防护，非 C-S9 fastpath 触发） */
/* 触发位置: sec_d 限流器 airy_sec_d_throttle_check()，见 10-user-supervisor-daemon.md §4.6 */
/* 注: 值 -83 为 Capability 码空间 [-71, -100] 内下一个可用值；-82 已分配给 AIRY_ECAP_FROZEN */
#define AIRY_ESEC_D_THROTTLED   (-83)   /* sec_d 限流拒绝（队列已满，Agent 请求被拒绝）*/

/* [-84, -100] 预留 */
```

**Capability Folding Badge 错误码语义详解**：

| 错误码 | 值 | 触发条件（C-S9 内） | 发送方处理 | 是否触发 Fault |
|--------|---|---------|-----------|:---:|
| `AIRY_ECAP_BADGE` | -78 | Badge 格式无效、Random Tag 不匹配（非伪造）、`CAP_CARRY` 置位但 `badge=0` 且 `opcode != CAP_REQUEST` | 检查 Badge 是否正确编译，重新向 sec_d 请求 | 否 |
| `AIRY_ECAP_EPOCH` | -79 | `badge_epoch != global_epoch`（Badge 已撤销或过期） | 重新向 sec_d 请求 Badge（`CAP_REQUEST` opcode） | 否 |
| `AIRY_ECAP_FORGED` | -80 | `badge_randtag != agent_caps[src_task].randtag`（Badge 伪造尝试） | 触发 Fault，sec_d 审计，可能终止 Agent | **是**（`AIRY_FAULT_CAP_FORGED` 0x1001） |
| `AIRY_ECAP_PERM` | -81 | `badge_perms & required != required`（权限位不满足 opcode 所需） | 重新申请更高权限 Badge | 否 |
| `AIRY_ECAP_FROZEN` | -82 | badge 撤销（A-ULS 控制，非 C-S0） | 等待解冻或联系管理员 | 否 |

**sec_d 限流拒绝码**（R1 补强新增，非 C-S9 fastpath 触发）：

`AIRY_ESEC_D_THROTTLED`(-83) 不是 C-S9 fastpath Badge 校验错误，而是 sec_d 限流器在请求队列满时返回的拒绝码。触发位置是 `airy_sec_d_throttle_check()`（详见 [10-user-supervisor-daemon.md §4.6](../20-modules/10-user-supervisor-daemon.md)）。Agent 收到此错误码后应执行指数退避重试，连续拒绝 3 次后降级至 [DSL] 模式。该错误码与 `-78~-82`（C-S9 Badge 校验码）在码空间上相邻（同属 Capability 码空间 `[-71, -100]`），但触发语义不同：`-78~-82` 由 fastpath C-S9 内联触发，`-83` 由 sec_d slowpath 限流器触发。

**[DSL] 降级模式下的兜底语义**：

当 `AIRY_SC_FALLBACK` 编译开关启用时，C-S9 跳过 Badge 校验（`capability_badge=0`），退化到 `airy_cap_check()` slowpath 兜底路径（基于 `agent_caps[1024]` 静态数组）。此时使用 `-76`（`AIRY_ECAP_RADIX_MISS`）和 `-77`（`AIRY_ECAP_STATIC_KEY`）作为兜底错误码（保留编号但 v1.0.1 不触发实际 radix tree 查找），`-78~-82` 不触发。详见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2.2。

### 2.5 Config 码 `[-101, -120]`（对齐 SSoT error.h L84-93）

A-UCS 配置管理专用错误码，覆盖配置版本/Schema/Base64/JSON/IO 等场景。`AIRY_ECFGVERSION` 是 [DSL] 降级块中唯一保留的非 POSIX 码（详见 §5.1）：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L84-93 —— Config 错误码 */
#define AIRY_ECFGVERSION      (-101)   /* Configuration version mismatch */
#define AIRY_ECFGSCHEMA       (-102)   /* Configuration schema invalid */
#define AIRY_ECFGBASE64       (-103)   /* Base64 decode failure */
#define AIRY_ECFGJSON         (-104)   /* JSON parse failure */
#define AIRY_ECFGIO           (-105)   /* Config I/O error */
/* [-106, -120] 预留 */
```

### 2.6 A-ULS 码 `[-121, -140]`（对齐 SSoT error.h L95-108）

A-ULS（Unified Lifecycle Supervision）调度与生命周期错误码，覆盖 sched_tac 策略/预算/截止时间/周期/优先级/权重与 agent 生命周期状态迁移：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L95-108 —— A-ULS 错误码 */
#define AIRY_ESCHED_POLICY    (-121)   /* Invalid scheduling policy */
#define AIRY_ESCHED_BUDGET    (-122)   /* Runtime budget exceeded */
#define AIRY_ESCHED_DEADLINE  (-123)   /* Deadline missed */
#define AIRY_ESCHED_PERIOD    (-124)   /* Invalid period */
#define AIRY_ESCHED_PRIO      (-125)   /* Invalid priority */
#define AIRY_ESCHED_WEIGHT    (-126)   /* Invalid EEVDF weight */
#define AIRY_ELIFECYCLE_STATE (-127)   /* Invalid agent lifecycle state */
#define AIRY_ELIFECYCLE_TRANS (-128)   /* Illegal state transition */
#define AIRY_ELIFECYCLE_AGENT (-129)   /* Agent not found */
#define AIRY_ELIFECYCLE_ZOMBIE (-130)  /* Agent in zombie state */
/* [-131, -140] 预留 */
```

### 2.7 MemoryRoVol 码 `[-141, -160]`（对齐 SSoT error.h L110-120）

MemoryRovol 内存子系统错误码，覆盖 tier 分配/GFP 标志/PMEM/CXL/page 分类/mmap/alloc/OOM：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L110-120 —— MemoryRoVol 错误码 */
#define AIRY_EMEM_TIER        (-141)   /* Invalid memory tier */
#define AIRY_EMEM_GFP         (-142)   /* Invalid GFP flags */
#define AIRY_EMEM_PMEM        (-143)   /* PMEM operation failed */
#define AIRY_EMEM_CXL         (-144)   /* CXL operation failed */
#define AIRY_EMEM_PAGE_CLASS  (-145)   /* Invalid page classification */
#define AIRY_EMEM_MMAP        (-146)   /* mmap failed */
#define AIRY_EMEM_ALLOC       (-147)   /* alloc_pages failed */
#define AIRY_EMEM_OOM         (-148)   /* Out of memory (agent-scoped) */
/* [-149, -160] 预留 */
```

### 2.8 Cognition 码 `[-161, -180]`（对齐 SSoT error.h L122-130）

A-UCS 认知子系统错误码，覆盖 CoreLoopThree 阶段/Think 模式/Q16.16/超时/迭代上限/置信度：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L122-130 —— Cognition 错误码 */
#define AIRY_ECOG_PHASE       (-161)   /* Invalid cognition phase */
#define AIRY_ECOG_MODE        (-162)   /* Invalid think mode */
#define AIRY_ECOG_Q16         (-163)   /* Q16.16 overflow/underflow */
#define AIRY_ECOG_TIMEOUT     (-164)   /* Cognition loop timeout */
#define AIRY_ECOG_ITERATIONS  (-165)   /* Max think iterations exceeded */
#define AIRY_ECOG_CONFIDENCE  (-166)   /* Confidence threshold not met */
/* [-167, -180] 预留 */
```

### 2.9 Log 码 `[-181, -200]`（对齐 SSoT error.h L132-140）

A-ULP 日志子系统错误码，覆盖 Ring Buffer 写入/满/级别/设施/持久化/magic 校验：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L132-140 —— Log 错误码 */
#define AIRY_ELOG_RING        (-181)   /* Ring Buffer write failed */
#define AIRY_ELOG_FULL        (-182)   /* Ring Buffer full */
#define AIRY_ELOG_LEVEL       (-183)   /* Invalid log level */
#define AIRY_ELOG_FACILITY    (-184)   /* Invalid facility code */
#define AIRY_ELOG_PERSIST     (-185)   /* Log persistence failed */
#define AIRY_ELOG_MAGIC       (-186)   /* Log record magic mismatch */
/* [-187, -200] 预留 */
```

### 2.10 Object 码 `[-201, -220]`（对齐 SSoT error.h L142-148）

Airymax Object 系统错误码，覆盖对象句柄/引用计数/类型匹配/对象销毁：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L142-148 —— Object 错误码 */
#define AIRY_EOBJ_HANDLE      (-201)   /* Invalid object handle */
#define AIRY_EOBJ_REFCOUNT    (-202)   /* Reference count overflow/underflow */
#define AIRY_EOBJ_TYPE        (-203)   /* Object type mismatch */
#define AIRY_EOBJ_GONE        (-204)   /* Object already destroyed */
/* [-205, -220] 预留 */
```

### 2.11 Syscall 码 `[-221, -240]`（对齐 SSoT error.h L150-156）

Airymax syscall 表面错误码，覆盖编号/参数/DSL 禁用/ABI 不匹配：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L150-156 —— Syscall 错误码 */
#define AIRY_ESYS_NUMBER      (-221)   /* Invalid syscall number */
#define AIRY_ESYS_ARGS        (-222)   /* Invalid syscall arguments */
#define AIRY_ESYS_DISABLED    (-223)   /* Syscall disabled in [DSL] mode */
#define AIRY_ESYS_ABI         (-224)   /* ABI mismatch */
/* [-225, -240] 预留 */
```

### 2.12 预留码 `[-241, -300]`

预留用于未来 Airymax 子系统扩展。分配前必须更新本契约文档与 SSoT `error.h`。

---

## §3 Fault 码空间分配

Fault 码占据正数空间 `[0x1000, 0x1FFF]`，从 `0x1000` 起步以避免与 errno 正值（如 `EPERM=1`）冲突。每个 Fault 对应一个不可恢复的异常场景，由 Micro-Supervisor 直接接管。

### 3.1 Fault 码定义（对齐 SSoT error.h L163-169，6 个 Fault 码）

> **⚠️ P0-I3 修复说明**（v1.0.1-fix）：原文档列出 11 个 Fault 码（`0x1001-0x100B`，含虚构的 `AIRY_FAULT_MEMORY_QUOTA_EXCEEDED=0x1007`、`AIRY_FAULT_TOKEN_BUDGET_EXCEEDED=0x1008`、`AIRY_FAULT_DMA=0x1009`、`AIRY_FAULT_URING_MALFORMED=0x100A`、`AIRY_FAULT_AUDIT_TAMPER=0x100B`），与 SSoT `error.h` L163-169 实际定义的 6 个 Fault 码严重不符。修复后严格对齐 SSoT：仅保留 `0x1001-0x1006` 共 6 个 Fault 码。原虚构的 5 个 Fault 码（`0x1007-0x100B`）标注为"v1.1+ 计划扩展，当前 SSoT 未定义"。

SSoT `error.h` 实际定义 6 个 Fault 码（`0x1001-0x1006`），覆盖 Badge 伪造/泄漏、Ring 损坏、Agent 心跳超时、Capability 异常、VM 页错误：

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L163-169 —— Fault 码（正数 0x1000+，不可恢复） */
#define AIRY_FAULT_CAP_FORGED        0x1001  /* Badge forgery (security breach) */
#define AIRY_FAULT_CAP_LEAK          0x1002  /* Capability leak detected */
#define AIRY_FAULT_RING_CORRUPT      0x1003  /* IPC ring corruption */
#define AIRY_FAULT_TIMEOUT           0x1004  /* Agent heartbeat timeout */
#define AIRY_FAULT_ABNORMAL_CAP      0x1005  /* Abnormal capability usage */
#define AIRY_FAULT_VM_FAULT          0x1006  /* VM page fault in Agent */
```

**Fault 码值域**：`[0x1000, 0x1FFF]`，从 `0x1000` 起步以避免与 errno 正值冲突。`0x1000` 本身保留作为"未指定 Fault"哨兵值，实际定义从 `0x1001` 起。`0x1007-0x1FFF` 当前未定义，预留给 v1.1+ 子系统扩展（如 io_uring hardened 路径、审计哈希链、资源配额等场景），新增必须更新 SSoT `error.h` 与本契约文档。

**v1.1+ 计划扩展（当前 SSoT 未定义，标注为计划项）**：

| Fault 码（计划） | 计划值 | 计划用途 | 当前状态 |
|------------------|--------|---------|---------|
| `AIRY_FAULT_MEMORY_QUOTA_EXCEEDED` | 0x1007 | 记忆配额溢出（mem_d 检测） | v1.1+ 计划，SSoT 未定义 |
| `AIRY_FAULT_TOKEN_BUDGET_EXCEEDED` | 0x1008 | Token 预算溢出（cogn_d 检测） | v1.1+ 计划，SSoT 未定义 |
| `AIRY_FAULT_DMA` | 0x1009 | Agent 设备 DMA 故障 | v1.1+ 计划，SSoT 未定义 |
| `AIRY_FAULT_URING_MALFORMED` | 0x100A | malformed SQE/CQE 输入 | v1.1+ 计划，SSoT 未定义 |
| `AIRY_FAULT_AUDIT_TAMPER` | 0x100B | 审计哈希链断裂 | v1.1+ 计划，SSoT 未定义 |

### 3.2 Fault 码触发与处置（对齐 SSoT 6 个 Fault 码）

| Fault 码 | 值 | 触发场景 | Micro-Supervisor 处置 | 后续路径 |
|---------|---|---------|---------------------|---------|
| `AIRY_FAULT_CAP_FORGED` | 0x1001 | C-S9 检测到 `badge_randtag != agent_caps[src_task].randtag`（Badge 伪造） | 立即冻结 IPC 队列 + eventfd 通知 + sec_d 审计 | Macro-Supervisor 裁决，可能 SIGKILL |
| `AIRY_FAULT_CAP_LEAK` | 0x1002 | sec_d 审计检测到 Badge 跨 Agent 使用（Agent A 使用 Agent B 的 Badge） | 标记涉事 Agent 为可疑 + 通知 Macro-Supervisor | Macro-Supervisor 裁决 |
| `AIRY_FAULT_RING_CORRUPT` | 0x1003 | CRC32 校验失败 + Ring 元数据不一致 | 立即冻结对应 Ring | 重启 Ring（保留未损坏消息） |
| `AIRY_FAULT_TIMEOUT` | 0x1004 | Agent 心跳超时且未响应冻结 | 强制 STOPPED 态 | Macro-Supervisor 裁决 |
| `AIRY_FAULT_ABNORMAL_CAP` | 0x1005 | Capability 树完整性校验失败 | 立即终止 Agent（SIGKILL） | 进入 DEAD 态 |
| `AIRY_FAULT_VM_FAULT` | 0x1006 | 共享页映射损坏（MemoryRovol 检测） | 立即标记 Ring 不可用 | 回退到 printk_safe |

> **⚠️ P0-I3 修复说明（续）**：原文档 §3.2 触发与处置表列出 11 行，含 5 行虚构 Fault 码（`AIRY_FAULT_MEMORY_QUOTA_EXCEEDED`/`TOKEN_BUDGET_EXCEEDED`/`DMA`/`URING_MALFORMED`/`AUDIT_TAMPER`）。修复后表格仅保留 SSoT 实际定义的 6 个 Fault 码。虚构 Fault 码的触发场景已移至 §3.1 "v1.1+ 计划扩展" 表中并标注"SSoT 未定义"。

### 3.3 Error 与 Fault 的关系

Error 与 Fault 在码空间上严格不重叠（Error 负数，Fault 正数 0x1000+），调用方可通过符号判断区分：

```c
static inline bool airy_is_fault(airy_err_t err) {
    return err >= 0x1000 && err <= 0x1FFF;
}
static inline bool airy_is_error(airy_err_t err) {
    return err < 0;
}
```

函数正常返回 `AIRY_EOK`（0），可恢复错误返回负数 Error，不可恢复故障返回正数 Fault 并触发 Fault Handler。

**Capability Folding 的 Error/Fault 协同**：C-S9 检测到 Badge 伪造时，**同时**返回 `-AIRY_ECAP_FORGED`（Error，发送方可恢复）和触发 `AIRY_FAULT_CAP_FORGED`（Fault，Micro-Supervisor 不可恢复处置）。发送方收到 Error 后可决定重试或退出，Micro-Supervisor 收到 Fault 后独立裁决。

---

## §4 CI 逐字节校验：sc-dual-ci.yml 验证两端 error.h 相同

### 4.1 校验机制

`sc-dual-ci.yml` 是 [SC] 双端一致性 CI 工作流，对 `error.h` 进行逐字节校验：

```yaml
# .github/workflows/sc-dual-ci.yml
name: [SC] Dual CI
on: [push, pull_request]
jobs:
  sc-error-h-validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: spharx/agentrt-linux
          path: agentrt-linux
      - uses: actions/checkout@v4
        with:
          repository: spharx/agentrt
          path: agentrt
      - name: Verify error.h byte-for-byte identical
        run: |
          diff agentrt-linux/kernel/include/uapi/linux/airymax/error.h \
               agentrt/commons/include/uapi/linux/airymax/error.h
          if [ $? -ne 0 ]; then
            echo "::error::error.h dual-end mismatch detected"
            exit 1
          fi
      - name: Verify [DSL] fallback block hash
        run: |
          scripts/airymax/check-uapi-compiler-agnostic.sh \
            agentrt-linux/kernel/include/uapi/linux/airymax/error.h
```

### 4.2 校验失败的处理

当 `sc-dual-ci.yml` 校验失败时：

1. CI 自动设置 `AIRY_SC_FALLBACK` 并重新构建（验证降级路径可编译）
2. CI 阻止合并，要求双端维护者同步评审
3. 修复后重新触发 CI，校验通过方可合并

### 4.3 编译器无关性校验

除逐字节校验外，`error.h` 还需通过编译器无关性校验（`check-uapi-compiler-agnostic.sh`），确保在 GCC / Clang / 不同版本下布局一致。校验项包括：

- 无 `sizeof` 依赖的结构体
- 无编译器扩展（`__attribute__` 仅限 `packed`）
- 无 `enum` 大小假设（错误码全部用 `#define`，不用 `enum`）

---

## §5 [DSL] 降级块：#ifdef AIRY_SC_FALLBACK → 仅保留 38 个 POSIX 码

### 5.1 降级块设计（对齐 SSoT error.h L175-219）

`error.h` 底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块，详细设计见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2。降级块生效时（P0-I3 修复：对齐 SSoT 实际定义，移除虚构的 `AIRY_ECFGVERSION` 别名声明）：

- **保留**：38 个 POSIX errno 负值（`AIRY_EACCES=-1` ~ `AIRY_EAGAIN=-35`，对齐 SSoT `error.h` L28-43 的 16 个核心 POSIX 码）
- **新增**：38 个 `AIRY_DSL_*` 宏映射（将 38 个未在 SSoT 主块定义的 POSIX 码映射到 7 个核心码 `AIRY_EINVAL`/`AIRY_EEXIST`/`AIRY_EBUSY`/`AIRY_ENOMEM`/`AIRY_ECANCELED`/`AIRY_EAGAIN`/`AIRY_ENOTSUP`）
- **禁用**：IPC 码、Capability 码、Config 码、A-ULS 码、MemoryRoVol 码、Cognition 码、Log 码、Object 码、Syscall 码（全部 9 个非 POSIX 子空间）
- **禁用**：全部 Fault 码（降级模式下故障统一走 Panic）

> **⚠️ P0-I3 修复说明（续）**：原文档 §5.1 声明"保留 1 个配置版本码 `AIRY_ECFGVERSION=-101`（降级块内别名，对应 `AIRY_ESC_CFGVERSION`）"，与 SSoT `error.h` L175-219 实际降级块不符。SSoT 降级块**未定义** `AIRY_ECFGVERSION` 别名（Config 码 `AIRY_ECFGVERSION` 在 L89 主块中定义，降级块不重复定义）。`AIRY_ESC_CFGVERSION` 也是虚构宏，SSoT 不存在。修复后移除该虚构声明。

### 5.2 降级块代码骨架（对齐 SSoT error.h L175-219）

```c
/* SSoT: kernel/include/uapi/linux/airymax/error.h L175-219 —— [DSL] 降级块 */
#ifdef AIRY_SC_FALLBACK
	/*
	 * When [SC] headers are unavailable (boot/rescue mode),
	 * fallback to 5 core POSIX codes mapped from 38 POSIX codes.
	 */
	#define AIRY_DSL_E2BIG        AIRY_EINVAL
	#define AIRY_DSL_ECHILD       AIRY_EINVAL
	#define AIRY_DSL_EDEADLK      AIRY_EBUSY
	#define AIRY_DSL_EDOM         AIRY_EINVAL
	#define AIRY_DSL_EEXIST_S     AIRY_EEXIST
	#define AIRY_DSL_EFBIG        AIRY_EINVAL
	#define AIRY_DSL_EILSEQ       AIRY_EINVAL
	#define AIRY_DSL_EINPROGRESS  AIRY_EAGAIN
	#define AIRY_DSL_EISCONN      AIRY_EBUSY
	#define AIRY_DSL_ELOOP        AIRY_EINVAL
	#define AIRY_DSL_EMFILE       AIRY_ENOMEM
	#define AIRY_DSL_EMLINK       AIRY_ENOMEM
	#define AIRY_DSL_ENAMETOOLONG AIRY_EINVAL
	#define AIRY_DSL_ENFILE       AIRY_ENOMEM
	#define AIRY_DSL_ENODEV       AIRY_EINVAL
	#define AIRY_DSL_ENOEXEC      AIRY_EINVAL
	#define AIRY_DSL_ENOLCK       AIRY_ENOMEM
	#define AIRY_DSL_ENOMSG       AIRY_ECANCELED
	#define AIRY_DSL_ENOTBLK      AIRY_EINVAL
	#define AIRY_DSL_ENOTCONN     AIRY_ECANCELED
	#define AIRY_DSL_ENOTDIR      AIRY_EINVAL
	#define AIRY_DSL_ENOTEMPTY    AIRY_EBUSY
	#define AIRY_DSL_ENOTSOCK     AIRY_EINVAL
	#define AIRY_DSL_ENXIO        AIRY_EINVAL
	#define AIRY_DSL_EOPNOTSUPP   AIRY_ENOTSUP
	#define AIRY_DSL_EOVERFLOW    AIRY_EINVAL
	#define AIRY_DSL_EPIPE        AIRY_ECANCELED
	#define AIRY_DSL_EPROTO       AIRY_EINVAL
	#define AIRY_DSL_EROFS        AIRY_EBUSY
	#define AIRY_DSL_ESPIPE       AIRY_EINVAL
	#define AIRY_DSL_ESRCH        AIRY_EINVAL
	#define AIRY_DSL_ETIMEDOUT    AIRY_ECANCELED
	#define AIRY_DSL_ETXTBSY      AIRY_EBUSY
	#define AIRY_DSL_EWOULDBLOCK  AIRY_EAGAIN
	#define AIRY_DSL_EXDEV        AIRY_EINVAL
	#define AIRY_DSL_ENODATA      AIRY_ECANCELED
	#define AIRY_DSL_ENOSR        AIRY_ENOMEM
	#define AIRY_DSL_ESTALE       AIRY_ECANCELED
#endif /* AIRY_SC_FALLBACK */
```

> **⚠️ P0-I3 修复说明（续）**：原文档 §5.2 降级块骨架虚构了一个 `_AIRY_ERROR_FALLBACK_H` 内层 include guard 与 `AIRY_ECFGVERSION` 别名定义，与 SSoT `error.h` L175-219 实际降级块不符。SSoT 实际降级块仅定义 38 个 `AIRY_DSL_*` 宏映射（映射到 `AIRY_EINVAL`/`AIRY_EEXIST`/`AIRY_EBUSY`/`AIRY_ENOMEM`/`AIRY_ECANCELED`/`AIRY_EAGAIN`/`AIRY_ENOTSUP` 7 个核心码），无内层 include guard，无 `AIRY_ECFGVERSION` 别名。修复后严格对齐 SSoT。

### 5.3 [DSL] 模式下的 Capability Folding 降级语义

当 `AIRY_SC_FALLBACK` 启用时，A-IPC fastpath C-S9 跳过 Badge 校验，退化到 `airy_cap_check()` slowpath 兜底路径（基于 `agent_caps[1024]` 静态数组）。此时：

- `capability_badge` 字段必须为 `0`（H6 硬约束）
- C-S9 跳过 `airy_cap_badge_ok()`，调用 `airy_cap_check()` 兜底（slowpath POSIX capability 校验，详见 [03-capability-model.md §4.3.2](../110-security/03-capability-model.md)）
- 错误码使用 `-76`（`AIRY_ECAP_RADIX_MISS`）和 `-77`（`AIRY_ECAP_STATIC_KEY`）
- `-78~-82`（Badge 校验码）不触发
- `0x1001-0x1003`（Capability Folding Fault 码）不触发（降级模式 Fault 统一走 Panic）

**[DSL] 模式限制**：
- 仅支持 `SEND`/`RECV` opcode（不支持 `SEND_BATCH`/`CANCEL`/`FREEZE`/`CAP_REQUEST`/`CAP_RESPONSE`）
- 性能退化（radix tree 查找 ~50-100ns vs Badge 校验 ~10ns）
- 安全性退化（无 Random Tag 防伪造，依赖传统 capability 模型）

### 5.4 降级块的独立性

降级块设计为**自包含**——不依赖 `error.h` 主体的任何符号。即使 `error.h` 主体损坏，只要降级块本身完整（通过 `.fallback_hashes` 校验），系统仍可降级启动。

---

## §6 物理宿主：kernel/include/uapi/linux/airymax/error.h

### 6.1 文件位置

| 属性 | 值 |
|------|-----|
| 内核侧路径 | `agentrt-linux/kernel/include/uapi/linux/airymax/error.h` |
| agentrt 侧路径 | `agentrt/commons/include/uapi/linux/airymax/error.h`（符号链接或 git submodule 引用内核侧物理文件） |
| 实际物理文件 | 仅一份，位于 `agentrt-linux/kernel/include/uapi/linux/airymax/error.h` |
| 文件编码 | UTF-8 without BOM |
| 换行符 | LF（Unix） |

### 6.2 文件结构（对齐 SSoT error.h 实际结构）

`error.h` 文件结构自上而下（P0-I2 修复：原描述遗漏 7 个子空间）：

1. 版权头 + 文件说明注释（L1-14）
2. 头文件守卫 `#ifndef _UAPI_AIRYMAX_ERROR_H`（L16-17）
3. `#include <linux/airymax/uapi_compat.h>`（L19）
4. `airy_err_t` 类型定义（L22）
5. `AIRY_EOK` 成功码（L25）
6. POSIX 码 `[-1, -40]`（L27-43，16 个定义）
7. IPC 码 `[-41, -70]`（L45-62，13 个定义，含 `AIRY_EIPC_FROZEN` 独立值 -53）
8. Capability 码 `[-71, -100]`（L64-82，13 个定义）
9. Config 码 `[-101, -120]`（L84-93，5 个定义）
10. A-ULS 码 `[-121, -140]`（L95-108，10 个定义）
11. MemoryRoVol 码 `[-141, -160]`（L110-120，8 个定义）
12. Cognition 码 `[-161, -180]`（L122-130，6 个定义）
13. Log 码 `[-181, -200]`（L132-140，6 个定义）
14. Object 码 `[-201, -220]`（L142-148，4 个定义）
15. Syscall 码 `[-221, -240]`（L150-156，4 个定义）
16. 预留码 `[-241, -300]`（L158-161）
17. Fault 码 `[0x1000, 0x1FFF]`（L163-169，6 个定义）
18. 辅助宏 `AIRY_ERR_OK` / `AIRY_ERR_FAIL`（L171-173）
19. `#ifdef AIRY_SC_FALLBACK` 降级块（L175-219，38 个 `AIRY_DSL_*` 映射）
20. 头文件守卫结束 `#endif`（L221）

### 6.3 变更流程

`error.h` 的任何变更必须遵循 [SC] 共享契约变更流程：

1. 在本文件（`08-sc-error-contract.md`）提出变更提案
2. agentrt 与 agentrt-linux 双方维护者评审
3. 同步修改 `kernel/include/uapi/linux/airymax/error.h` 物理头文件
4. `sc-dual-ci.yml` 双端校验通过
5. 更新本契约文档版本号

---

## §7 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §4 —— A-UEF 模块总纲
- [10-unify-design.md](../10-architecture/10-unify-design.md) §8 —— A-IPC 模块总纲（Capability Folding）
- [10-unify-design.md](../10-architecture/10-unify-design.md) §10 —— SSoT 技术选型权威声明
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §2 —— [SC] 共享契约层
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2 —— [DSL] 降级块机制
- [02-ipc-protocol.md](02-ipc-protocol.md) —— Layout C v4 消息头定义（`capability_badge` offset 40-47）
- [07-ipc-fastpath.md](07-ipc-fastpath.md) —— fastpath C-S0~C-S12 检查链（C-S9 Badge 校验触发 -78~-82）
- [01-syscalls.md](01-syscalls.md) —— syscall 12→4 映射（`airy_sys_call` 编译 Badge）
- [03-capability-model.md](../110-security/03-capability-model.md) —— Capability 模型（Badge 编译/撤销/校验）
- [120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) §2.7 —— [SC] `ipc.h` Layout C v4 定义

---

## §8 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：A-UEF [SC] error.h 二进制契约；Error 码 5 子空间分配（POSIX/IPC/Capability/[SC]/[DSL]）；Fault 码 0x1000+ 分配；CI 逐字节校验；[DSL] 降级块；物理宿主 `kernel/include/uapi/linux/airymax/error.h` |
| v1.1 | 2026-07-18 | Capability Folding 同步：IPC 码空间 -41~-52 重定义（与 fastpath C-S 检查链对齐）；新增 Capability 码 -78~-82（Badge 校验：BADGE/EPOCH/FORGED/PERM/FROZEN）；Fault 码 0x1001-0x1003 重定义（CAP_FORGED/CAP_LEAK/RING_CORRUPT）；`AIRY_FAULT_VM_FAULT` 迁移至 0x1006；新增 [DSL] 码 -207（CAP_BADGE_SKIP）；新增 §1.4 Capability Folding 错误码的 [SC]/[SS] 边界；新增 §5.3 [DSL] 模式 Badge 降级语义；新增 §3.3 Error/Fault 协同说明 |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] error.h 二进制契约 | v1.0.1 | 2026-07-21
