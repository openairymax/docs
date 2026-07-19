Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] error.h 二进制契约
> **文档定位**：A-UEF（统一错误码与故障定义体系）模块的 [SC] 共享契约权威定义\
> **文档版本**：v1.1\
> **最后更新**：2026-07-18\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §4\
> **设计依据**：本契约为 SSoT（Single Source of Truth），错误码值域、命名风格、[DSL] 降级块均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax 错误码与故障码体系** 的唯一权威源。Error 码空间分配、Fault 码空间分配、`AIRY_E*` 命名风格、`AIRY_FAULT_*` 命名风格、[DSL] 降级块均以本文件为唯一权威定义。物理宿主为 `kernel/include/uapi/linux/airymax/error.h`，agentrt 与 agentrt-linux 物理共享同一份头文件，逐字节相同。
>
> **Capability Folding 工程定义**（v1.1 新增）：A-IPC 采用 Capability Folding 设计模式——将 capability check（能力校验）从独立的控制面操作"折叠"到 IPC 数据面消息传递 fastpath 中。其物理载体是 [SC] `ipc.h` Layout C v4 消息头 offset 40-47 的 `capability_badge` 字段（64-bit Native Word：Epoch + Random Tag + Perms）；其执行点是 fastpath C-S9 内联校验（`airy_cap_badge_ok()`，~10ns，3 个 READ_ONCE + 位运算 + 比较）。本契约新增的 `-78~-82` Capability 错误码与 `0x1001-0x1003` Fault 码是 Capability Folding 的错误语义 SSoT。
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

Error 码占据负数空间 `[-300, -1]`，按来源分 5 个子空间，每个子空间有明确的值域、来源与语义。

### 2.1 Error 码空间总表

| 子空间 | 值域 | 来源 | 数量 | 命名风格 |
|--------|------|------|------|---------|
| POSIX 码 | `[-1, -40]` | 对齐 Linux errno | 40 | `AIRY_E*`（对齐 `E*` errno 名） |
| IPC 码 | `[-41, -70]` | A-IPC 协议层（Capability Folding fastpath C-S0~C-S12） | 30 | `AIRY_EIPC_*` |
| Capability 码 | `[-71, -100]` | 安全子系统（含 Capability Folding Badge 校验） | 30 | `AIRY_ECAP_*` |
| [SC] 码 | `[-101, -200]` | [SC] 共享契约层 | 100 | `AIRY_ESC_*` |
| [DSL] 码 | `[-201, -300]` | [DSL] 降级生存层 | 100 | `AIRY_EDSL_*` |

### 2.2 POSIX 码 `[-1, -40]`

对齐 Linux 6.6 标准 errno，前 40 个为 POSIX 兼容码。完整清单见 `kernel/include/uapi/linux/airymax/error.h`，部分关键码：

```c
#define AIRY_EOK            0       /* 成功（非负数，特殊） */
#define AIRY_EPERM          (-1)    /* 权限不足 */
#define AIRY_ENOENT         (-2)    /* 不存在 */
#define AIRY_EINTR          (-4)    /* 被信号中断 */
#define AIRY_EIO            (-5)    /* I/O 错误 */
#define AIRY_ENOMEM         (-12)   /* 内存不足 */
#define AIRY_EACCES         (-13)   /* 拒绝访问 */
#define AIRY_EFAULT         (-14)   /* 错误地址 */
#define AIRY_EBUSY          (-16)   /* 设备忙 */
#define AIRY_EEXIST         (-17)   /* 已存在 */
#define AIRY_EINVAL         (-22)   /* 参数无效 */
#define AIRY_ENOSPC         (-28)   /* 空间不足 */
#define AIRY_EPIPE          (-32)   /* 管道破裂 */
#define AIRY_ENOSYS         (-38)   /* 功能未实现 */
#define AIRY_ENOTEMPTY      (-39)   /* 目录非空 */
#define AIRY_ELOOP          (-40)   /* 符号链接循环 */
```

### 2.3 IPC 码 `[-41, -70]`

A-IPC 协议层专用错误码，覆盖 IPC 协议、Ring Buffer、io_uring_cmd、fastpath C-S0~C-S12 校验链等场景。v1.1 起，IPC 码空间与 [07-ipc-fastpath.md](07-ipc-fastpath.md) 的 C-S 检查链精确对齐：

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

**v1.1 变更说明**：v1.0 的 IPC 码（`RING_FULL`/`RING_EMPTY`/`REGISTERED`/`URING_CMD` 等）已废弃，替换为与 C-S 检查链精确对齐的命名。由于 0.1.1 阶段 0 行内核代码，此替换无向后兼容负担。

### 2.4 Capability 码 `[-71, -100]`

安全子系统专用错误码，覆盖 Capability 校验、纯 C LSM 检查、Capability Folding Badge 校验等场景。v1.1 新增 `-78~-82` 用于 Capability Folding Badge 校验：

```c
/* ===== Capability 码空间 [-71, -100] ===== */

/* 原有 Capability 校验码（v1.0 保留） */
#define AIRY_ECAP_MISSING       (-71)   /* Capability 缺失 */
#define AIRY_ECAP_REVOKED       (-72)   /* Capability 已撤销 */
#define AIRY_ECAP_EXPIRED       (-73)   /* Capability 已过期 */
#define AIRY_ECAP_MISMATCH      (-74)   /* Capability 不匹配 */
#define AIRY_ECAP_LSM_DENIED    (-75)   /* 纯 C LSM 拒绝 */

/* [DSL] 降级模式兜底码（v1.0 保留，仅 AIRY_SC_FALLBACK 模式触发） */
#define AIRY_ECAP_RADIX_MISS    (-76)   /* [DSL] radix tree 查找失败 */
#define AIRY_ECAP_STATIC_KEY    (-77)   /* [DSL] static_key 禁用 */

/* Capability Folding Badge 校验码（v1.1 新增） */
/* 触发位置: fastpath C-S9 内联 airy_cap_badge_ok()，见 07-ipc-fastpath.md §5.2 */
#define AIRY_ECAP_BADGE         (-78)   /* Badge 格式无效、Random Tag 不匹配、CAP_CARRY 但 badge=0 */
#define AIRY_ECAP_EPOCH         (-79)   /* Badge Epoch 与全局 Epoch 不匹配（已撤销或过期） */
#define AIRY_ECAP_FORGED        (-80)   /* 检测到 Badge 伪造尝试（同时触发 AIRY_FAULT_CAP_FORGED） */
#define AIRY_ECAP_PERM          (-81)   /* Badge 权限位不满足 opcode 所需权限 */
#define AIRY_ECAP_FROZEN        (-82)   /* Ring 已冻结（C-S0 检查，A-ULS 控制） */

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
| `AIRY_ECAP_FROZEN` | -82 | `ring->frozen == true`（C-S0 检查，A-ULS 通过 `FREEZE` opcode 设置） | 等待解冻或联系管理员 | 否 |

**sec_d 限流拒绝码**（R1 补强新增，非 C-S9 fastpath 触发）：

`AIRY_ESEC_D_THROTTLED`(-83) 不是 C-S9 fastpath Badge 校验错误，而是 sec_d 限流器在请求队列满时返回的拒绝码。触发位置是 `airy_sec_d_throttle_check()`（详见 [10-user-supervisor-daemon.md §4.6](../20-modules/10-user-supervisor-daemon.md)）。Agent 收到此错误码后应执行指数退避重试，连续拒绝 3 次后降级至 [DSL] 模式。该错误码与 `-78~-82`（C-S9 Badge 校验码）在码空间上相邻（同属 Capability 码空间 `[-71, -100]`），但触发语义不同：`-78~-82` 由 fastpath C-S9 内联触发，`-83` 由 sec_d slowpath 限流器触发。

**[DSL] 降级模式下的兜底语义**：

当 `AIRY_SC_FALLBACK` 编译开关启用时，C-S9 跳过 Badge 校验（`capability_badge=0`），退化到 radix tree 兜底路径。此时使用 `-76`（`AIRY_ECAP_RADIX_MISS`）和 `-77`（`AIRY_ECAP_STATIC_KEY`）作为兜底错误码，`-78~-82` 不触发。详见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2.2。

### 2.5 [SC] 码 `[-101, -200]`

[SC] 共享契约层专用错误码，覆盖配置版本、头文件校验等场景：

```c
#define AIRY_ESC_CFGVERSION     (-101)  /* 配置版本不匹配（[DSL] 降级块内别名 AIRY_ECFGVERSION） */
#define AIRY_ESC_HASH_MISMATCH  (-102)  /* [SC] 头文件 hash 不匹配 */
#define AIRY_ESC_FALLBACK       (-103)  /* 已降级到 [DSL] 模式 */
#define AIRY_ESC_DUAL_CI        (-104)  /* 双端 CI 校验失败 */
#define AIRY_ESC_UAPI_TYPE      (-105)  /* UAPI 类型违规 */
#define AIRY_ESC_FLOAT_FORBIDDEN (-106) /* 禁止使用 float */
#define AIRY_ESC_PACKED_REQUIRED (-107) /* 缺少 __packed */
/* [-108, -200] 预留 */
```

### 2.6 [DSL] 码 `[-201, -300]`

[DSL] 降级生存层专用错误码，覆盖降级模式下的特殊场景：

```c
#define AIRY_EDSL_ACTIVE            (-201)  /* [DSL] 降级模式已激活 */
#define AIRY_EDSL_FAULT_DISABLED    (-202)  /* Fault 码已禁用（降级模式统一走 Panic） */
#define AIRY_EDSL_RING_UNAVAILABLE  (-203)  /* Ring Buffer 不可用 */
#define AIRY_EDSL_PRINTK_FALLBACK   (-204)  /* 已回退到 printk 原生 */
#define AIRY_EDSL_EEVDF_ONLY        (-205)  /* 仅 EEVDF 调度可用（sched_tac 降级） */
#define AIRY_EDSL_CAP_MINIMAL       (-206)  /* 仅 POSIX capability 可用（Badge 机制降级） */
#define AIRY_EDSL_CAP_BADGE_SKIP    (-207)  /* C-S9 跳过 Badge 校验（v1.1 新增） */
/* [-208, -300] 预留 */
```

---

## §3 Fault 码空间分配

Fault 码占据正数空间 `[0x1000, 0x1FFF]`，从 `0x1000` 起步以避免与 errno 正值（如 `EPERM=1`）冲突。每个 Fault 对应一个不可恢复的异常场景，由 Micro-Supervisor 直接接管。

### 3.1 Fault 码定义

v1.1 起，Fault 码空间按模块重新组织。`0x1001-0x1003` 为 Capability Folding A-IPC 专用，`0x1004-0x1005` 为 A-ULS 专用，`0x1006` 为 MemoryRovol 专用：

```c
/* kernel/include/uapi/linux/airymax/error.h —— Fault 码（正数 0x1000+，不可恢复） */

/* ===== A-IPC Capability Folding Fault 码（v1.1 新增） ===== */
#define AIRY_FAULT_CAP_FORGED    0x1001   /* Badge 伪造尝试（C-S9 检测，触发 -80 + 此 Fault） */
#define AIRY_FAULT_CAP_LEAK      0x1002   /* Badge 泄漏（跨 Agent 使用，sec_d 审计检测） */
#define AIRY_FAULT_RING_CORRUPT  0x1003   /* Ring 数据损坏（CRC32 校验失败 + 元数据不一致） */

/* ===== A-ULS 生命周期 Fault 码 ===== */
#define AIRY_FAULT_TIMEOUT       0x1004   /* Agent 心跳超时且未响应冻结 */
#define AIRY_FAULT_ABNORMAL_CAP  0x1005   /* Capability 树完整性校验失败 */

/* ===== MemoryRovol 内存 Fault 码 ===== */
#define AIRY_FAULT_VM_FAULT      0x1006   /* 共享页映射损坏（v1.1 从 0x1002 迁移至此） */

/* ===== 资源配额与驱动 Fault 码（v1.1 新增预留） ===== */
#define AIRY_FAULT_MEMORY_QUOTA_EXCEEDED  0x1007   /* 记忆配额溢出（mem_d 检测，macro_d 强制降级/终止） */
#define AIRY_FAULT_TOKEN_BUDGET_EXCEEDED  0x1008   /* Token 预算溢出（cogn_d 检测，macro_d 强制降级/终止） */
#define AIRY_FAULT_DMA                    0x1009   /* Agent 设备 DMA 故障（driver model 上报，A-ULS 升级处理） */

/* ===== io_uring / 审计 Fault 码（R9 补强注册） ===== */
/* 注册说明：R9 任务原文指定 AIRY_FAULT_URING_MALFORMED=0x1007、AIRY_FAULT_AUDIT_TAMPER=0x1008，
 *          但 0x1007/0x1008 已分配给 AIRY_FAULT_MEMORY_QUOTA_EXCEEDED/TOKEN_BUDGET_EXCEEDED。
 *          依"保持现有 Fault 码编号不变"硬约束，本注册追加至表末，分配下一个可用值 0x100A/0x100B。
 *          物理宿主：kernel/include/uapi/linux/airymax/error.h（与现有 Fault 码同文件）。
 */
#define AIRY_FAULT_URING_MALFORMED        0x100A   /* malformed SQE/CQE 输入（io_uring hardened 路径检测，
                                                   * 触发 airy_security_fault 通知 Micro-Supervisor） */
#define AIRY_FAULT_AUDIT_TAMPER           0x100B   /* 审计哈希链断裂（airy_audit_chain_verify 检测，
                                                   * 触发紧急告警 + 安全团队介入） */

/* 预留 0x100C ~ 0x1FFF */
```

**v1.1 变更说明**：
- `0x1001`：`AIRY_FAULT_CAP_FAULT`（通用 Capability 故障）→ `AIRY_FAULT_CAP_FORGED`（精确到 Badge 伪造）
- `0x1002`：`AIRY_FAULT_VM_FAULT`（MemoryRovol）→ `AIRY_FAULT_CAP_LEAK`（A-IPC Badge 泄漏）；`AIRY_FAULT_VM_FAULT` 迁移至 `0x1006`
- `0x1003`：`AIRY_FAULT_IPC_FAULT`（通用 IPC 故障）→ `AIRY_FAULT_RING_CORRUPT`（精确到 Ring 数据损坏）
- `0x1004`/`0x1005`：保持不变（A-ULS 专用）

### 3.2 Fault 码触发与处置

| Fault 码 | 触发场景 | Micro-Supervisor 处置 | 后续路径 |
|---------|---------|---------------------|---------|
| `AIRY_FAULT_CAP_FORGED` | C-S9 检测到 `badge_randtag != agent_caps[src_task].randtag`（Badge 伪造） | 立即冻结 IPC 队列 + eventfd 通知 + sec_d 审计 | Macro-Supervisor 裁决，可能 SIGKILL |
| `AIRY_FAULT_CAP_LEAK` | sec_d 审计检测到 Badge 跨 Agent 使用（Agent A 使用 Agent B 的 Badge） | 标记涉事 Agent 为可疑 + 通知 Macro-Supervisor | Macro-Supervisor 裁决 |
| `AIRY_FAULT_RING_CORRUPT` | CRC32 校验失败 + Ring 元数据不一致 | 立即冻结对应 Ring | 重启 Ring（保留未损坏消息） |
| `AIRY_FAULT_TIMEOUT` | Agent 心跳超时且未响应冻结 | 强制 STOPPED 态 | Macro-Supervisor 裁决 |
| `AIRY_FAULT_ABNORMAL_CAP` | Capability 树完整性校验失败 | 立即终止 Agent（SIGKILL） | 进入 DEAD 态 |
| `AIRY_FAULT_VM_FAULT` | 共享页映射损坏（MemoryRovol 检测） | 立即标记 Ring 不可用 | 回退到 printk_safe |
| `AIRY_FAULT_MEMORY_QUOTA_EXCEEDED` | 记忆配额溢出（mem_d 检测） | macro_d 强制降级/终止 | 释放记忆卷载或终止 Agent |
| `AIRY_FAULT_TOKEN_BUDGET_EXCEEDED` | Token 预算溢出（cogn_d 检测） | macro_d 强制降级/终止 | 降级 LLM 推理或终止 Agent |
| `AIRY_FAULT_DMA` | Agent 设备 DMA 故障（driver model 上报） | A-ULS 升级处理 | 释放 devm 资源或终止 Agent |
| `AIRY_FAULT_URING_MALFORMED` | malformed SQE/CQE 输入（io_uring hardened 路径检测，opcode/flags/payload_len 越界） | 触发 `airy_security_fault(agent_id, 0x100A, cmd)` 通知 Micro-Supervisor + 冻结 Ring | Micro-Supervisor 裁决（重启 Ring 或终止 Agent） |
| `AIRY_FAULT_AUDIT_TAMPER` | 审计哈希链断裂（`airy_audit_chain_verify` 检测到 `prev_hash` 不匹配或签名校验失败） | 紧急 CRITICAL 告警 + 停止审计写入 + 安全团队介入 | 安全团队人工介入，密钥轮换 + 审计卷重建 |

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

### 5.1 降级块设计

`error.h` 底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块，详细设计见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2。降级块生效时：

- **保留**：38 个 POSIX errno 负值（`AIRY_EPERM=-1` ~ `AIRY_ELOOP=-40`）
- **保留**：1 个配置版本码 `AIRY_ECFGVERSION=-101`（降级块内别名，对应 `AIRY_ESC_CFGVERSION`）
- **禁用**：IPC 码、Capability 码、[SC] 扩展码、[DSL] 码
- **禁用**：全部 Fault 码（降级模式下故障统一走 Panic）

### 5.2 降级块代码骨架

```c
/* kernel/include/uapi/linux/airymax/error.h —— 文件底部 */
#ifdef AIRY_SC_FALLBACK
#ifndef _AIRY_ERROR_FALLBACK_H
#define _AIRY_ERROR_FALLBACK_H
/* 仅保留 38 个 POSIX 码 + AIRY_ECFGVERSION，详见 11-degraded-survival-layer.md §2.1 */
#define AIRY_EPERM          (-1)
/* ... */
#define AIRY_ELOOP          (-40)
#define AIRY_ECFGVERSION    (-101)   /* [DSL] 唯一保留的非 POSIX 码 */
#warning "AIRY_SC_FALLBACK active: only 38 POSIX codes + AIRY_ECFGVERSION available"
#endif /* _AIRY_ERROR_FALLBACK_H */
#endif /* AIRY_SC_FALLBACK */
```

### 5.3 [DSL] 模式下的 Capability Folding 降级语义

当 `AIRY_SC_FALLBACK` 启用时，A-IPC fastpath C-S9 跳过 Badge 校验，退化到 radix tree 兜底路径。此时：

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

### 6.2 文件结构

`error.h` 文件结构自上而下：

1. 版权头 + 文件说明注释
2. 头文件守卫 `#ifndef _AIRY_ERROR_H`
3. `airy_err_t` 类型定义
4. POSIX 码 `[-1, -40]`
5. IPC 码 `[-41, -70]`（v1.1：C-S 检查链对齐）
6. Capability 码 `[-71, -100]`（v1.1：含 Badge 校验码 -78~-82）
7. [SC] 码 `[-101, -200]`
8. [DSL] 码 `[-201, -300]`
9. Fault 码 `[0x1000, 0x1FFF]`（v1.1：按模块组织）
10. 内联辅助函数 `airy_is_fault()` / `airy_is_error()`
11. `#ifdef AIRY_SC_FALLBACK` 降级块
12. 头文件守卫结束 `#endif`

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

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] error.h 二进制契约 | v1.1 | 2026-07-18
