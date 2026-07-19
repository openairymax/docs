Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）IRON-9 v3 四层共享模型落地规范
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）与 agentrt（微核心用户态运行时）之间的 IRON-9 v3 四层代码共享模型落地规范。详细说明 \[SC] 共享契约层（10 个头文件）、\[SS] 语义同源层、\[IND] 完全独立层、\[DSL] 降级生存层的实施细节，含双向 CI 校验机制与 magic 设计原理。本文件落地 **IRON-9 v3 四层模型（[SC]+[SS]+[IND]+[DSL]）**，权威源为 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)。\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[10-unify-design.md](../10-architecture/10-unify-design.md)（Unify Design 总纲）\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **编号权威**：[09-ssot-registry.md §3](./09-ssot-registry.md)\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **SSoT 依赖声明**：本文件引用 **IRON-9 v3 四层模型（[SC]/[SS]/[IND]/[DSL]）**作为跨项目代码共享的权威框架，权威源为 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)。IPC magic 值（0x41524531 'ARE1'）与任务描述符 magic（0x41475453 'AGTS'）的权威定义登记于 [09-ssot-registry.md §3](./09-ssot-registry.md)。命名前缀隔离（`airy_*`/`airy_*`）引用 [10-coding-style/coding_conventions.md Part IV §4.4.2](./10-coding-style/coding_conventions.md)。[DSL] 降级生存层设计引用 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)。

***

## 0. 文档说明

### 0.1 目的与范围

本文件是 **IRON-9 v3 四层共享模型（[SC]+[SS]+[IND]+[DSL]）**的工程落地规范，定义 agentrt（用户态微核心）与 agentrt-linux（内核微内核发行版）之间的代码共享边界、契约内容、校验机制。四层模型的权威定义见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)，[DSL] 降级生存层的详细设计见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)。

### 0.2 术语约束（硬性）

- **agentrt**（用户态）= **微核心**（micro-core）。描述 agentrt 时使用"微核心"，其原语称"微核心原语"。**禁止**用"微内核原语"描述 agentrt。
- **agentrt-linux**（OS 发行版）= **微内核**（micro-kernel）。描述 agentrt-linux 时使用"微内核"。
- **禁止**使用特定上游发行版名称，统一用"主流 Linux 发行版"。
- 产品全称"agentrt-linux（AirymaxOS，极境智能体操作系统）"，简称"agentrt-linux"或"AirymaxOS"。

### 0.3 Linux 6.6 基线约束

- **sched_tac 调度类约束**：使用 Linux 6.6 内核基线 原生调度类（SCHED\_NORMAL=0 / SCHED\_FIFO=1 / SCHED\_RR=2 / SCHED\_BATCH=3 / SCHED\_IDLE=5 / SCHED\_DEADLINE=6，`include/uapi/linux/sched.h:114-123`）。agentrt-linux 通过 sched_tac 复用这些原生调度类，**禁止**定义 `SCHED_AGENT` 内核调度类宏（避免与内核调度类编号冲突），**禁止**使用 `SCHED_EXT=7`（Linux 6.6 主线不含 sched_ext 框架，仅宏定义存在但实现不可用）。
- 内核态禁 float（`arch/x86/Makefile:137` `-mno-80387`），用 Q16.16 定点数 `airy_q16_t`（= `int32_t`）。
- kthread 间通信用 `kfifo` + `wait_event_interruptible`。

### 0.4 Linux 6.6 内核基线 源码路径

- `include/uapi/linux/sched.h:114-123` —— SCHED\_NORMAL=0 / SCHED\_FIFO=1 / SCHED\_RR=2 / SCHED\_BATCH=3 / SCHED\_IDLE=5 / SCHED\_DEADLINE=6 / SCHED\_EXT=7（仅宏定义，sched_ext 框架未实现，6.6 主线不可用）
- `arch/x86/Makefile:137-138` —— `-mno-80387` 禁浮点
- `scripts/checkpatch.pl:836-851` —— `%deprecated_apis` 表（参考共享 API 治理）

***

## 1. IRON-9 v3 四层模型总览

> **权威源声明**：本节为 IRON-9 v3 四层模型（[SC]+[SS]+[IND]+[DSL]）的工程落地概览，权威定义见 [06-iron9-shared-model.md §1-§5](../10-architecture/06-iron9-shared-model.md)。

### 1.1 四层定义

| 层次          | 缩写     | 共享程度               | 内容                                                        | 组织方式                             |
| ----------- | ------ | ------------------ | --------------------------------------------------------- | -------------------------------- |
| **共享契约层**   | \[SC]  | 完全共享代码             | 10 个头文件，定义数据结构、magic、操作码、枚举                                | `include/uapi/linux/airymax/` 独立 UAPI 头文件库（D-8 对齐 OLK 6.6），两端共同依赖 |
| **语义同源层**   | \[SS]  | 高层 API 语义同源，签名独立演进 | 调度语义、安全模型、IPC 传输、记忆模型                                     | 各自独立实现，通过 \[SC] 保证互操作            |
| **完全独立层**   | \[IND] | 完全独立               | 内核驱动/Kbuild/内核 API（agentrt-linux）；跨平台用户态/SDK 四语言（agentrt） | 各自独立仓库，无依赖                       |
| **降级生存层**   | \[DSL] | 自包含降级块（[SC] 头文件内）   | `#ifdef AIRY_SC_FALLBACK` 降级块；[SC] 损坏时的最小可运行子集（38 POSIX 码 + 最简 IPC + EEVDF 默认调度） | 详见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) |

### 1.2 与旧版 IRON-9 的差异

| 维度    | 旧版 IRON-9 | IRON-9 v2        | IRON-9 v3                    |
| ----- | --------- | ---------------- | ---------------------------- |
| 契约层代码 | 不共享，仅语义同源 | **完全共享**（10 个头文件） | **完全共享**（10 个头文件，v3 沿用）       |
| 互操作方式 | 同源语义无适配层  | **共享代码无适配层**（增强） | **共享代码无适配层**（v3 沿用）           |
| CI 校验 | 无         | 契约层变更双向校验        | 契约层变更双向校验 + magic 双向校验（v3 增强） |
| 降级路径  | 无         | 无                | **[DSL] 降级生存层**（v3 新增第四层）    |

***

## 2. \[SC] 共享契约层

### 2.1 10 个 \[SC] 头文件清单

\[SC] 层包含**10 个**共享头文件，**唯一物理宿主**于 `kernel/include/uapi/linux/airymax/` 目录（其他子仓通过 `-I../kernel/include` 引用，禁止物理副本，OS-IRON-014 落地），两端（agentrt 与 agentrt-linux）共同依赖，**逐字节相同**。下表清单与 [06-iron9-shared-model.md §2.2](../10-architecture/06-iron9-shared-model.md) 完全一致（权威源）：

| 序号 | 文件                  | 共享内容                                                                                                    | magic 值               | 落地路径                                |
| -- | ------------------- | ------------------------------------------------------------------------------------------------------- | --------------------- | ----------------------------------- |
| 1  | `error.h`           | A-UEF 统一错误码（AIRY_E* POSIX 负值 + AIRY_FAULT_* 0x1000+ 故障码）+ [DSL] 降级块                                      | —                     | `include/uapi/linux/airymax/error.h`           |
| 2  | `log_types.h`       | A-ULP 统一日志类型（128B 固定记录格式 + 5 级日志枚举 + printk 8 级映射）                                                     | —                     | `include/uapi/linux/airymax/log_types.h`       |
| 3  | `ipc.h`             | IPC magic（0x41524531 'ARE1'）+ 128B 消息头结构（`struct airy_ipc_msg_hdr`）+ SQE/CQE 操作码与标志位                 | `0x41524531` ('ARE1') | `include/uapi/linux/airymax/ipc.h`             |
| 4  | `sched.h`           | sched_tac 调度约束（SCHED_DEADLINE/SCHED_FIFO/EEVDF）+ 任务描述符（magic 0x41475453 'AGTS'）+ vtime 类型与衰减公式 + 优先级范围 + SLICE\_DFL | `0x41475453` ('AGTS') | `include/uapi/linux/airymax/sched.h`           |
| 5  | `memory_types.h`    | MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 持久化接口                                                          | —                     | `include/uapi/linux/airymax/memory_types.h`    |
| 6  | `security_types.h`  | POSIX capability 41 ID + LSM 钩子 252 ID + Cupolas blob 布局 + capability 派生模型（`airy_capability_t` 结构体 + MDB 派生树）+ **capability 引用类型（`cap_t` = `uint64_t`）**+ Vault backend + 策略裁决 4 值枚举 | —                     | `include/uapi/linux/airymax/security_types.h`  |
| 7  | `cognition_types.h` | `airy_q16_t` Q16.16 定点数 + CoreLoopThree 阶段枚举（`airy_cog_phase`）+ Thinkdual 模式枚举（`airy_think_mode`） | —                     | `include/uapi/linux/airymax/cognition_types.h` |
| 8  | `syscalls.h`        | Syscall 编号体系（v1.1: 4 核心 + 20 预留 = 24 槽位）                                                                     | —                     | `include/uapi/linux/airymax/syscalls.h`        |
| 9  | `uapi_compat.h`     | 三路类型桥接（内核态 `__u32` ↔ 用户态 Linux `uint32_t` ↔ 第三方 `uint32_t` with stdint.h），确保 [SC] 头文件跨平台逐字节相同编译         | —                     | `include/uapi/linux/airymax/uapi_compat.h`    |
| 10 | `lsm_types.h`       | 纯 C LSM 模块类型定义（`struct airy_lsm_blob` + `airy_capability_check()` 回调原型 + Capability 缓存结构）            | —                     | `include/uapi/linux/airymax/lsm_types.h`       |

**补充内容**（不属于上述 10 个 [SC] 核心头文件，但两端共享语义）：

- capability 令牌格式
- 规则编号体系（IRON/BAN/STD/ACC）
- 五维正交 24 原则

### 2.2 补充共享文件 1：bpf\_struct\_ops.h（非 [SC] 核心头文件）

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRY_BPF_STRUCT_OPS_H
#define _AIRY_BPF_STRUCT_OPS_H

#include <airymax/uapi_compat.h>

/**
 * struct airy_struct_ops_value - shared struct_ops state
 * @state: Operation state machine value.
 * @common: Common value shared between BPF and scheduler.
 *
 * Shared between agentrt (user-space BPF loader) and
 * agentrt-linux (kernel struct_ops framework).
 */
struct airy_struct_ops_value {
	__u32	state;
	__u64	common;
};

enum airy_struct_ops_state {
	AIRY_OPS_INIT	= 0,
	AIRY_OPS_REGISTERED	= 1,
	AIRY_OPS_ACTIVE	= 2,
	AIRY_OPS_DRAINING	= 3,
};

#endif /* _AIRY_BPF_STRUCT_OPS_H */
```

> **定位说明**：`bpf_struct_ops.h` 是 agentrt 与 agentrt-linux 之间的**补充共享文件**（SDK 网关状态管理共享结构），两端共享代码但**不属于 10 个 [SC] 核心头文件**。其 `struct airy_struct_ops_value` 和 `enum airy_struct_ops_state` 用于 agentrt 用户态 BPF loader 与 agentrt-linux 内核 struct_ops 框架之间的状态同步。

### 2.2.1 [SC] 核心头文件 9：uapi\_compat.h（三路类型桥接）

> **定位说明**：`uapi_compat.h` 是 [SC] 10 个核心头文件之一（IRON-9 v3 升级），提供三路类型桥接：内核态（`__u32` 等，通过 `<linux/types.h>`）、用户态 Linux（`uint32_t` 等，通过 `<stdint.h>` + typedef）、第三方平台（`uint32_t` with stdint.h）。

所有 10 个 [SC] 核心头文件 + `bpf_struct_ops.h`（补充）均通过 `#include <airymax/uapi_compat.h>` 获取 `__u8`/`__u16`/`__u32`/`__u64`/`__s32`/`__s64` 等 UAPI 类型。该文件在 Linux 内核态直接包含 `<linux/types.h>`，在用户态 Linux 上通过 `<stdint.h>` + typedef 提供等价类型，在 macOS/Windows 上通过 `<stdint.h>` 提供等价类型，确保 [SC] 头文件在 agentrt 用户态跨平台（Linux/macOS/Windows）和 agentrt-linux 内核态均能逐字节相同地编译。

> **设计约束**：[SC] 头文件中**禁止**直接 `#include <linux/types.h>`（macOS/Windows 无此头文件），**必须**通过 `<airymax/uapi_compat.h>` 间接获取 UAPI 类型。

### 2.2.2 [SC] 核心头文件 1：error.h（A-UEF 统一错误码）

> **定位说明**：`error.h` 是 [SC] 10 个核心头文件之一（IRON-9 v3 升级，A-UEF 模块），物理宿主为 `kernel/include/uapi/linux/airymax/error.h`，agentrt 用户态通过 `-I` 引用，禁止物理副本。

**错误码体系**（A-UEF 统一错误码与故障定义体系）：

- `airy_err_t` 类型定义（`typedef int32_t airy_err_t`）
- 成功码：`AIRY_EOK = 0`（与 `AIRY_SUCCESS = 0` 等价，推荐使用 `AIRY_EOK`）
- Error 码（负数，可恢复）：5 子空间
  - `[-1, -40]` POSIX errno 负值（`AIRY_EINVAL=(-22)`、`AIRY_ENOMEM=(-12)` 等）
  - `[-41, -70]` IPC 错误码
  - `[-71, -100]` Capability 错误码
  - `[-101, -200]` [SC] 契约错误码
  - `[-201, -300]` [DSL] 降级错误码
- Fault 码（正数 0x1000+，不可恢复）：`AIRY_FAULT_CAP_FORGED`(0x1001) / `AIRY_FAULT_CAP_LEAK`(0x1002) / `AIRY_FAULT_RING_CORRUPT`(0x1003) / `AIRY_FAULT_TIMEOUT`(0x1004) / `AIRY_FAULT_ABNORMAL_CAP`(0x1005) / `AIRY_FAULT_VM_FAULT`(0x1006)
- [DSL] 降级块：`#ifdef AIRY_SC_FALLBACK` → 仅保留 38 个 POSIX 码

**权威源迁移状态**（2026-07-17 完成）：

- **当前权威源**：`kernel/include/uapi/linux/airymax/error.h`（[SC] 物理宿主，已创建，195 行）
- **历史源**：`agentrt/commons/utils/error/include/error.h` + `agentrt/commons/include/airy_types.h`（迁移完成后废弃）
- **SSoT 登记**：`09-ssot-registry.md §2 OS-IRON-014`（[SC] 10 个头文件单一数据源）
- **CI 校验**：`sc-dual-ci.yml` 逐字节验证两端 error.h 相同

**已废弃方案**：方案 B（runtime_interfaces.md 的 -1/-2/-11 自定义序列）、方案 C（coding_conventions.md 的 -2/-4/-6 旧值）、方案 D（project_erp.md 的位掩码 0x00010000，仅用于 ERP 分类，非错误码方案）均已废弃，禁止在新代码中使用。

任何文档、代码不得另起定义，必须引用此权威源。详见 `180-i18n/03-error-message-i18n.md` §2.2 和 `30-interfaces/08-sc-error-contract.md`。

### 2.3 头文件 1：memory\_types.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRY_MEMORY_TYPES_H
#define _AIRY_MEMORY_TYPES_H

#include <airymax/uapi_compat.h>

/* MemoryRovol L1-L4 hierarchy (shared semantics) */
enum airy_mem_level {
	AIRY_MEM_L1_HOT	= 0,	/* Hot working set, agent-local */
	AIRY_MEM_L2_WARM	= 1,	/* Warm set, node-local */
	AIRY_MEM_L3_COLD	= 2,	/* Cold set, node-remote */
	AIRY_MEM_L4_PMEM	= 3,	/* Persistent memory */
};

/*
 * GFP mask semantics for memory tier selection.
 * Platform-independent semantic bitmask constants (NOT kernel __GFP_* flags).
 * Each side translates AIRY_GFP_* to its own allocation mechanism internally.
 */
#define AIRY_GFP_HOT		(0x01u)	/* Hot: agent-local working set */
#define AIRY_GFP_WARM		(0x02u)	/* Warm: node-local */
#define AIRY_GFP_COLD		(0x04u)	/* Cold: node-remote */
#define AIRY_GFP_PMEM		(0x08u)	/* Persistent memory */

#endif /* _AIRY_MEMORY_TYPES_H */
```

### 2.4 头文件 2：security\_types.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRY_SECURITY_TYPES_H
#define _AIRY_SECURITY_TYPES_H

#include <airymax/uapi_compat.h>

/* POSIX capability 41 IDs (subset shown) */
#define AIRY_CAP_AGENT_SPAWN		41	/* Agent spawn capability (Airymax 专属，从 41 开始避免与 Linux 0-40 冲突) */
#define AIRY_CAP_GPU_SCHED		42	/* GPU scheduling */
#define AIRY_CAP_NPU_ACCESS		43	/* NPU access */

/* LSM hook IDs (252 total) */
#define AIRY_LSM_HOOK_TASK_CREATE	0
#define AIRY_LSM_HOOK_IPC_SEND	1

/* Cupolas policy verdict (4 values) */
enum airy_verdict {
	AIRY_VERDICT_ALLOW	= 0,
	AIRY_VERDICT_DENY	= 1,
	AIRY_VERDICT_AUDIT	= 2,
	AIRY_VERDICT_COMPLAIN	= 3,   /* 拒绝但记录（学习模式，对齐 AppArmor complain 语义） */
};

/* Capability invocation operations (seL4 CNode model) */
enum airy_cap_op {
	AIRY_CAP_OP_COPY	= 0,
	AIRY_CAP_OP_MINT	= 1,
	AIRY_CAP_OP_MOVE	= 2,
	AIRY_CAP_OP_MUTATE	= 3,
	AIRY_CAP_OP_REVOKE	= 4,
	AIRY_CAP_OP_DELETE	= 5,
	AIRY_CAP_OP_ROTATE	= 6,
};

#endif /* _AIRY_SECURITY_TYPES_H */
```

### 2.5 头文件 3：cognition\_types.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRY_COGNITION_TYPES_H
#define _AIRY_COGNITION_TYPES_H

#include <airymax/uapi_compat.h>

/* Q16.16 fixed-point: required because -mno-80387 (no FPU) */
typedef __s32 airy_q16_t;
#define AIRY_Q16_ONE		(1 << 16)

/* CoreLoopThree phases */
enum airy_cog_phase {
	AIRY_COG_PERCEPT	= 0,
	AIRY_COG_THINK	= 1,
	AIRY_COG_ACT		= 2,
};

/* Thinkdual modes */
enum airy_think_mode {
	AIRY_THINK_FAST	= 0,	/* System-1: fast, intuitive */
	AIRY_THINK_SLOW	= 1,	/* System-2: slow, deliberative */
};

#endif /* _AIRY_COGNITION_TYPES_H */
```

### 2.6 头文件 4：sched.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRY_SCHED_H
#define _AIRY_SCHED_H

#include <airymax/uapi_compat.h>

/* sched_tac (Linux 6.6 内核基线 原生调度类: SCHED_NORMAL=0/SCHED_FIFO=1/SCHED_DEADLINE=6)
 * 禁止定义 SCHED_AGENT 内核调度类宏, 禁止使用 SCHED_EXT=7（6.6 主线不含 sched_ext 框架）。 */

/* Task descriptor magic: 0x41475453 = 'AGTS' (Agent Task) */
#define AIRY_TASK_MAGIC	0x41475453u

/* vtime type: Q16.16 fixed-point (no FPU) */
typedef __s32 airy_vtime_t;

/* Priority range: 0 (highest) - 139 (lowest), compatible with Linux */
#define AIRY_PRIO_MIN	0
#define AIRY_PRIO_MAX	139

/* Default scheduling slice: 20ms */
#define AIRY_SLICE_DFL	20

/* Weight range: compatible with Linux EEVDF weight model */
#define AIRY_WEIGHT_MIN	1
#define AIRY_WEIGHT_MAX	10000

/*
 * MAC_MAX_AGENTS: hard limit for concurrent agent scheduling.
 * SSoT value: 1024 (validated by benchmark to 1000 concurrent).
 * All modules MUST reference this constant instead of local MAX_AGENTS.
 */
#define MAC_MAX_AGENTS	1024

/**
 * airy_vtime_decay - Compute vtime after slice consumption
 * @vtime: Current virtual time (Q16.16 fixed-point).
 * @consumed_slice: Slice consumed in nanoseconds.
 * @weight: Task weight in [AIRY_WEIGHT_MIN, AIRY_WEIGHT_MAX].
 *
 * Return: New vtime after decay.
 */
static inline airy_vtime_t
airy_vtime_decay(airy_vtime_t vtime, __u64 consumed_slice, __u32 weight)
{
	return vtime + (airy_vtime_t)(consumed_slice * 100 / weight);
}

/**
 * struct airy_task_desc - Agent task descriptor ([SC] shared)
 * @magic: Must be AIRY_TASK_MAGIC (0x41475453 'AGTS').
 * @prio: Priority in [AIRY_PRIO_MIN, AIRY_PRIO_MAX].
 * @vtime: Virtual time for fair scheduling (Q16.16).
 *
 * Shared between agentrt (user-space) and agentrt-linux (kernel).
 * The magic field validates descriptor integrity across the boundary.
 */
struct airy_task_desc {
	__u32		magic;		/* 0x41475453 'AGTS' */
	__u16		prio;		/* 0-139 */
	__u16		_pad;
	airy_vtime_t	vtime;		/* Q16.16 fixed-point */
};

#endif /* _AIRY_SCHED_H */
```

### 2.7 头文件 5：ipc.h

> **SSoT 权威源声明**：本头文件为 A-IPC `Layout C v4` 消息头与 Badge 位布局的 [SC] 共享契约层物理宿主。Layout C v4 字段语义权威源为 [02-ipc-protocol.md §2](../30-interfaces/02-ipc-protocol.md)；fastpath 校验语义（C-S9 等）属于 [SS] 语义同源层，不在本头文件中定义，权威源为 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md)；Badge 64-bit Native Word 安全模型权威源为 [03-capability-model.md §3](../110-security/03-capability-model.md)。

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
/*
 * kernel/include/uapi/linux/airymax/ipc.h —— [SC] 共享契约层，物理宿主
 * IRON-9 v3 [SC] 层：agentrt 与 agentrt-linux 共享数据结构定义
 * 校验机制属于 [SS] 语义同源层，不在本头文件中定义
 */
#ifndef AIRYMAX_IPC_H
#define AIRYMAX_IPC_H

#include <airymax/types.h>  /* __u32, __u64, __u8 */

#ifdef __cplusplus
extern "C" {
#endif

/* ===== Layout C v4 — 128 字节定长消息头 ===== */
/* magic: 0x41524531 'ARE1' ——一经发布即冻结（本文件 §7.5）*/

#define AIRY_IPC_MAGIC		0x41524531u  /* 'ARE1' (Airymax Runtime Engine v1) */
#define AIRY_IPC_HDR_SIZE	128

/* Ring capacity constants */
#define AIRY_IPC_RING_DEF_ENTRIES	256
#define AIRY_IPC_RING_MAX_ENTRIES	32768

/* opcode 定义（见 02-ipc-protocol.md §4） */
#define AIRY_IPC_OP_SEND		0x0001
#define AIRY_IPC_OP_RECV		0x0002  /* 仅用于接收方声明的 expected opcode */
#define AIRY_IPC_OP_SEND_BATCH		0x0003
#define AIRY_IPC_OP_CANCEL		0x0004
#define AIRY_IPC_OP_FREEZE		0x0005  /* A-ULS 触发 ring 冻结 */
#define AIRY_IPC_OP_CAP_REQUEST	0x0010  /* 自举：请求 Badge，无 Badge */
#define AIRY_IPC_OP_CAP_RESPONSE	0x0011  /* sec_d 返回编译好的 Badge */

/* flags 定义 */
#define AIRY_IPC_F_ZEROCOPY		0x0001
#define AIRY_IPC_F_CAP_CARRY		0x0002  /* 携带 Badge（agentrt-linux 内核必置） */
#define AIRY_IPC_F_ENCRYPT		0x0004  /* 保留，0.1.1 不启用 */
#define AIRY_IPC_F_COMPRESS		0x0008  /* 保留，0.1.1 不启用 */
#define AIRY_IPC_F_BATCH_TAIL		0x0010  /* SEND_BATCH 的最后一个 SQE */
#define AIRY_IPC_F_RESERVED		0xFFE0  /* 必须为零 */

/* SQE flags for io_uring IPC operations */
#define AIRY_IPC_SQE_F_FIXED_BUF	(1u << 0)
#define AIRY_IPC_SQE_F_ASYNC		(1u << 1)
#define AIRY_IPC_SQE_F_BUF_SELECT	(1u << 2)
#define AIRY_IPC_SQE_F_SKIP_CQE	(1u << 3)

/* CQE flags for io_uring IPC operations */
#define AIRY_IPC_CQE_F_BUFFER		(1u << 0)
#define AIRY_IPC_CQE_F_MORE		(1u << 1)
#define AIRY_IPC_CQE_F_NOTIF		(1u << 2)

/**
 * struct airy_ipc_msg_hdr - IPC message header (128 bytes, [SC] shared)
 * @magic:            Must be AIRY_IPC_MAGIC (0x41524531 'ARE1').
 * @opcode:           Operation code; see AIRY_IPC_OP_* above.
 * @flags:            Message flags; see AIRY_IPC_F_* above.
 * @trace_id:         OpenTelemetry trace ID, end-to-end tracing.
 * @timestamp_ns:     CLOCK_REALTIME nanoseconds, sender fills.
 * @src_task:         Source Agent/daemon task_id, bound to Badge at compile.
 * @dst_task:         Destination task_id, decides kfifo routing.
 * @capability_badge: [SS] semantic-aligned: agentrt-linux uses (compiled by
 *                    sec_d), agentrt user-space always 0 (H3).
 * @payload_len:      Payload length, excluding header.
 * @crc32:            CRC32 over header[0:52) + payload.
 * @reserved:         Reserved, must be zero (C-S4 checks).
 *
 * Layout C v4 — 128-byte fixed header. Shared between agentrt and
 * agentrt-linux via [SC] contract layer. magic is frozen post-release.
 *
 * Alignment design (D-9 fix, mirrors OLK 6.6 kernel protocol headers like
 * struct ethhdr/iphdr/udphdr which do NOT use __attribute__((packed))):
 *   - No __attribute__((packed)); all __u64 fields 8-byte aligned naturally.
 *   - capability_badge moved to offset 40 (was 44 in v0.1.1 draft) to
 *     restore 8-byte natural alignment; payload_len shifted to offset 48.
 *   - struct uses __attribute__((aligned(64))) to guarantee 2 cache lines.
 *   - 128B = 2 cache lines (x86_64/aarch64 64B/line).
 */
struct airy_ipc_msg_hdr {
	__u32	magic;			/* offset  0, 'ARE1' 0x41524531 */
	__u16	opcode;			/* offset  4, see AIRY_IPC_OP_* */
	__u16	flags;			/* offset  6, see AIRY_IPC_F_* */
	__u64	trace_id;		/* offset  8, OpenTelemetry trace ID */
	__u64	timestamp_ns;		/* offset 16, CLOCK_REALTIME nanoseconds */
	__u64	src_task;		/* offset 24, source Agent/daemon task_id */
	__u64	dst_task;		/* offset 32, destination Agent/daemon task_id */
	__u64	capability_badge;	/* offset 40, [SS] agentrt-linux uses, agentrt=0 */
	__u32	payload_len;		/* offset 48, payload length (excluding header) */
	__u32	crc32;			/* offset 52, CRC32 over header[0:52) + payload */
	__u8	reserved[72];		/* offset 56, must be zero (CL1 [56:64) + CL2 [64:128)) */
} __attribute__((aligned(64)));

_Static_assert(sizeof(struct airy_ipc_msg_hdr) == AIRY_IPC_HDR_SIZE,
	"airy_ipc_msg_hdr must be exactly 128 bytes");
_Static_assert(offsetof(struct airy_ipc_msg_hdr, capability_badge) == 40,
	"capability_badge must be at offset 40 (8-byte aligned, D-9 fix)");
_Static_assert(offsetof(struct airy_ipc_msg_hdr, crc32) == 52,
	"crc32 must be at offset 52");

/* ===== Badge 位布局（仅 [SS] 语义同源，定义在内核实现侧）===== */
/* agentrt 用户态：capability_badge 始终为 0（H3） */
/* agentrt-linux 内核：capability_badge 由 sec_d 编译（H4） */
/* [DSL] 降级模式：capability_badge = 0，跳过 C-S9（H6） */

#define AIRY_BADGE_EPOCH_SHIFT		48
#define AIRY_BADGE_EPOCH_MASK		0xFFFF000000000000ULL
#define AIRY_BADGE_RANDTAG_SHIFT	16
#define AIRY_BADGE_RANDTAG_MASK		0x0000FFFFFFFF0000ULL
#define AIRY_BADGE_PERMS_SHIFT		0
#define AIRY_BADGE_PERMS_MASK		0x000000000000FFFFULL

/* Badge 提取宏（仅内核侧使用，[SS] 语义同源层） */
#define AIRY_BADGE_EPOCH(b)	(((b) & AIRY_BADGE_EPOCH_MASK)   >> AIRY_BADGE_EPOCH_SHIFT)
#define AIRY_BADGE_RANDTAG(b)	(((b) & AIRY_BADGE_RANDTAG_MASK) >> AIRY_BADGE_RANDTAG_SHIFT)
#define AIRY_BADGE_PERMS(b)	(((b) & AIRY_BADGE_PERMS_MASK)   >> AIRY_BADGE_PERMS_SHIFT)

/* Badge 编译宏（仅 sec_d 使用） */
#define AIRY_BADGE_COMPILE(epoch, randtag, perms) \
	((((__u64)(epoch)   & 0xFFFF) << AIRY_BADGE_EPOCH_SHIFT)   | \
	 (((__u64)(randtag) & 0xFFFFFFFF) << AIRY_BADGE_RANDTAG_SHIFT) | \
	 (((__u64)(perms)   & 0xFFFF) << AIRY_BADGE_PERMS_SHIFT))

/* Capability 权限位（低 16 位） */
#define AIRY_CAP_PERM_SEND	0x0001
#define AIRY_CAP_PERM_RECV	0x0002
#define AIRY_CAP_PERM_CALL	0x0004  /* airy_sys_call 权限 */
#define AIRY_CAP_PERM_GRANT	0x0008  /* 派生能力（mint/derive） */
#define AIRY_CAP_PERM_REVOKE	0x0010  /* 撤销能力（仅 sec_d） */
#define AIRY_CAP_PERM_FREEZE	0x0020  /* 冻结 ring */
#define AIRY_CAP_PERM_BATCH	0x0040  /* 批量发送 */
#define AIRY_CAP_PERM_RESERVED	0xFF80  /* 必须为零 */

#ifdef __cplusplus
}
#endif

#endif /* AIRYMAX_IPC_H */
```

**[SC] 与 [SS] 的精确边界（H2）**：

- **[SC] 共享契约层（本头文件）**：`struct airy_ipc_msg_hdr` 数据结构、字段偏移与大小、magic 值、opcode 枚举、flags 位定义、Badge 位布局宏、Capability 权限位、`_Static_assert(sizeof == 128)`。
- **[SS] 语义同源层（不在本头文件）**：`airy_cap_badge_ok()` 内联函数、`agent_caps[]` 静态数组、`airy_cap_global_epoch` atomic_t、Random Tag 生成、C-S9 校验逻辑、sec_d Badge 编译流程。
- **[IND] 完全独立层**：agentrt 用户态不使用 `capability_badge`（始终为 0），使用传统 `cap_t` 引用；agentrt-linux 内核使用 `capability_badge` + C-S9 内联校验。
- **[DSL] 降级生存层**：`capability_badge` 字段存在但被忽略（值=0），C-S9 跳过，退化到 radix tree 兜底路径。

### 2.8 头文件 6：syscalls.h

> **SSoT 权威源声明**：本头文件为 agentrt-linux syscall 接口的 [SC] 共享契约层物理宿主。Capability Folding 单平面架构将原 12 syscall 精简为 4 syscall（IPC 数据传递全部走 io_uring 数据面），权威源为 [01-syscalls.md §1](../30-interfaces/01-syscalls.md)。

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
/*
 * kernel/include/uapi/linux/airymax/syscalls.h —— [SC] 共享契约层，物理宿主
 * IRON-9 v3 [SC] 层：agentrt 与 agentrt-linux 共享数据结构定义
 */
#ifndef AIRYMAX_SYSCALLS_H
#define AIRYMAX_SYSCALLS_H

#include <airymax/types.h>

#ifdef __cplusplus
extern "C" {
#endif

/*
 * Agent syscall architecture (Capability Folding 单平面架构):
 *   4 core + 20 reserved = 24 slots total
 *
 *   Control plane (4 syscalls, low-frequency management operations):
 *     AIRY_SYS_CALL       — Capability invocation (sec_d compiles Badge)
 *     AIRY_SYS_ROVOL_CTL  — MemoryRovol snapshot/restore/tier
 *     AIRY_SYS_SCHED_CTL  — sched_tac scheduling policy config
 *     AIRY_SYS_CLT_NOTIFY — CoreLoopThree phase + kthread notify
 *
 *   Data plane (io_uring, zero syscall):
 *     IORING_OP_URING_CMD → airy_uring_cmd()
 *       ├─ airy_ipc_validate()   [C-S0~C-S11, 含 C-S9 Badge]
 *       ├─ airy_ipc_deliver_fast()  [NONBLOCK, ~158ns]
 *       └─ airy_ipc_deliver_full()  [io-wq, wake_up]
 *
 *   Original 12 → 4 精确映射（见 01-syscalls.md §1.3）:
 *     airy_sys_send        → 移除，改用 io_uring URING_CMD (cmd_op=SEND)
 *     airy_sys_recv        → 移除，改用 io_uring CQE poll
 *     airy_sys_call        → 保留（仅 sec_d 调用，编译 Badge）
 *     airy_sys_reply       → 移除，改用 io_uring URING_CMD (cmd_op=SEND 反向)
 *     airy_sys_delegate    → 移除，由 sec_d 重编译 Badge (airy_sys_call + COMPILE_BADGE)
 *     airy_sys_mint        → 移除，由 sec_d 编译 Badge (airy_sys_call + COMPILE_BADGE)
 *     airy_sys_revoke      → 移除，由 sec_d 撤销 + atomic_inc (airy_sys_call + REVOKE_BADGE)
 *     airy_sys_call_batch  → 移除，改用 io_uring SQE 链（多个 URING_CMD）
 *     airy_sys_rovol_ctl   → 保留
 *     airy_sys_sched_ctl   → 保留
 *     airy_sys_clt_notify  → 保留
 *     airy_sys_lsm_ctl     → 合并到 airy_sys_call (airy_sys_call + LSM_CTL)
 *
 *   Linux 512-535 槽位注册号（4 个使用，其余 reserved）:
 */
#define AIRY_SYS_CALL		512	/* Capability Invocation (sec_d compiles Badge) */
#define AIRY_SYS_ROVOL_CTL	513	/* MemoryRovol control */
#define AIRY_SYS_SCHED_CTL	514	/* sched_tac scheduling policy control */
#define AIRY_SYS_CLT_NOTIFY	515	/* Client event notification */
/* Reserved slots 516-535 for future expansion */

#ifdef __cplusplus
}
#endif

#endif /* AIRYMAX_SYSCALLS_H */
```

***

## 3. [DSL] 降级生存层

> **权威源声明**：本节为 [DSL] 降级生存层的工程落地概览，权威定义见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)。四层模型权威源为 [06-iron9-shared-model.md §5](../10-architecture/06-iron9-shared-model.md)。

### 3.1 [DSL] 层定义

**[DSL]（Degraded Survival Layer，降级生存层）**是 IRON-9 v3 新增的第四层，定义 [SC] 头文件损坏/缺失时的最小可运行子集。每个 [SC] 头文件底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块，提供自包含的最小符号集，确保 [SC] 损坏时系统仍能启动、打印日志、完成基本 IPC、安全 Panic。

**特征**：

- **自包含**：降级块不依赖任何外部头文件，仅使用标准 C 预处理与编译器内置类型
- **最小集**：仅定义维持系统启动所需的最小符号集
- **显式告警**：降级块生效时发出 `#warning`，告知开发者当前为降级模式
- **最后防线**：[DSL] 是韧性领域的最后防线，正常运行时不生效

### 3.2 [DSL] 降级时的最小可运行子集

| 维度 | 正常模式 | [DSL] 降级模式 |
|------|---------|--------------|
| 错误码 | 5 子空间（300 码） | 38 个 POSIX 码 + 1 个配置码（`AIRY_ECFGVERSION`） |
| 日志 | Ring Buffer + Logger Daemon | printk 原生（仅 `LOG_FATAL` + `LOG_ERROR`） |
| IPC | 完整 128B 消息头 + 3 操作 | 最简 128B 消息头（3 字段）+ 2 操作 |
| 调度 | sched_tac 三层（SCHED_DEADLINE/SCHED_FIFO/EEVDF） | EEVDF 默认调度 |
| 安全 | 纯 C LSM 完整校验 | 仅 POSIX capability |
| Fault | Fault Handler 优雅处理 | 统一 Panic（Fault 码禁用） |

### 3.3 [DSL] 与 [SC] 的关系

[DSL] 降级块的物理宿主仍在 [SC] 的 10 个头文件内部（通过 `#ifdef AIRY_SC_FALLBACK` 条件编译实现），但语义独立——[SC] 定义"正常时的完整契约"，[DSL] 定义"失败时的生存契约"。构建系统对每个降级块单独计算 hash（存入 `.fallback_hashes`），即使 [SC] 主体损坏，只要降级块 hash 校验通过，系统仍可降级启动。

### 3.4 [DSL] 降级块触发条件

构建系统在以下情况自动注入 `AIRY_SC_FALLBACK`：

1. `sc-dual-ci.yml` 检测到两端 [SC] 头文件不一致
2. `airy_defconfig` 中显式配置 `CONFIG_AIRY_SC_FALLBACK=y`
3. [SC] 头文件物理校验失败（hash 不匹配）

详细降级路径设计、Panic 回退路径（`printk_safe`）、各头文件降级块职责见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)。

***

## 4. magic 值设计

> **权威源声明**：本节为 IPC magic 与任务描述符 magic 的工程落地说明，权威定义见 [09-ssot-registry.md §3](./09-ssot-registry.md) 与 [06-iron9-shared-model.md §2.2](../10-architecture/06-iron9-shared-model.md)。

### 4.1 IPC magic 0x41524531 ('ARE1')

| 字段    | 值             | 说明                        |
| ----- | ------------- | ------------------------- |
| 十六进制  | `0x41524531`  | 大端表示                      |
| ASCII | `'ARE1'`      | Airymax Runtime Engine v1 |
| 用途    | IPC 消息头 magic | 验证 IPC 协议版本与消息完整性         |

**SSoT 权威定义**：`#define AIRY_IPC_MAGIC 0x41524531u`（[SC] `include/uapi/linux/airymax/ipc.h`）

**设计原理**：

1. **可识别性**：'ARE1' 是人类可读的 ASCII，便于 hexdump 调试时识别 IPC 消息
2. **版本化**：末尾 '1' 表示协议版本 1，未来 breaking change 升级为 'ARE2'
3. **跨端一致**：agentrt 与 agentrt-linux 两端字节相同，无字节序转换（host byte order 内部传输）
4. **不可变更**：magic 一经发布即冻结，变更必须升级版本号

**P0-05 收敛（0.1.1）**：agentrt 内全部 IPC 传输层 magic 统一引用 `AIRY_IPC_MAGIC`，禁止本地硬编码十六进制值。收敛映射表：

| 层 | 原独立定义 | 收敛后 | 文件 |
| - | -------- | --- | --- |
| [SC] 共享契约 | `AIRY_IPC_MAGIC 0x41524531u` | **SSoT（不变）** | `include/uapi/linux/airymax/ipc.h` |
| L2 corekern IPC | `ARE_IPC_MAGIC 0x41524531u` | `#define ARE_IPC_MAGIC AIRY_IPC_MAGIC` | `atoms/corekern/include/are_ipc.h` |
| 应用层 IPC | `IPC_MAGIC 0x49504300` ❌ | `#define IPC_MAGIC AIRY_IPC_MAGIC` | `commons/utils/ipc/include/ipc_common.h` |
| 服务总线 | `IPC_BUS_MESSAGE_MAGIC 0x49534200` ❌ | `#define IPC_BUS_MESSAGE_MAGIC AIRY_IPC_MAGIC` | `commons/utils/ipc/include/ipc_service_bus.h` |
| airy_types | 注释 `0x414F5350 'AOSP'` ❌ | 注释改为 `AIRY_IPC_MAGIC = 0x41524531 'ARE1'` | `commons/include/airy_types.h` |

> **注意**：`IPC_RPC_MAGIC 0x52504300`（'RPC\0'）不在收敛范围——它是 payload 侧 RPC 子协议标识符（`ipc_rpc_header_t`），不是 IPC 传输层 magic，保留独立值。
>
> **L2 布局声明（[IND] 独立层）**：`ARE_IPC_MAGIC` 与 `AIRY_IPC_MAGIC` 共享同源 magic 值，但 L2 消息头布局 `are_ipc_message_header_t`（128B，14 字段）与 [SC] `struct airy_ipc_msg_hdr`（128B，9 字段）字段布局不同，属 IRON-9 [IND] 层——magic 同源、布局独立。

### 4.2 任务描述符 magic 0x41475453 ('AGTS')

| 字段    | 值            | 说明               |
| ----- | ------------ | ---------------- |
| 十六进制  | `0x41475453` | 大端表示             |
| ASCII | `'AGTS'`     | AGent Task State |
| 用途    | 任务描述符 magic  | 验证任务描述符完整性       |

**设计原理**：

1. **与 IPC magic 区分**：'AGTS' ≠ 'ARE1'，避免任务描述符与 IPC 消息混淆
2. **场景区分**：IPC magic 用于消息边界验证，任务 magic 用于描述符生命周期验证
3. **\[SC] 共享**：两端共享同一 magic，跨端传递任务描述符时无需额外标识

### 4.3 magic 常量锁定

| magic         | 十六进制         | ASCII  | 场景       | 共享层   |
| ------------- | ------------ | ------ | -------- | ----- |
| IPC 消息头 magic | `0x41524531` | 'ARE1' | IPC 协议识别 | \[SC] |
| 任务描述符 magic   | `0x41475453` | 'AGTS' | 任务描述符识别  | \[SC] |

> **变更禁令**：magic 值一经发布即冻结，禁止修改。如需协议演进，新增 magic（如 'ARE2'）而非修改现有 magic。

***

## 5. \[SS] 语义同源层

### 5.1 \[SS] 层定义

\[SS] 层中，agentrt 与 agentrt-linux 的高层 API 语义同源（概念操作一致）；SDK 层签名同源（同一份源码两端编译），其他层签名因抽象层级不同而独立演进。两端通过 \[SC] 契约层保证语义一致，无需适配层。

### 5.2 四大语义同源映射

| 语义域                   | agentrt（用户态微核心）实现             | agentrt-linux（内核微内核）实现            | \[SC] 契约依据                              |
| --------------------- | ----------------------------- | --------------------------------- | --------------------------------------- |
| **调度语义**（MicroCoreRT） | 用户态调度器（协程/线程池）                | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF，不含 sched\_ext）  | `sched.h`：任务描述符 + vtime + 优先级           |
| **安全模型**（Cupolas）     | 用户态策略引擎（seccomp + capability） | 纯 C LSM 钩子（252 ID） + capability 41 ID | `security_types.h`：capability + verdict |
| **IPC 传输**（AgentsIPC） | 用户态消息队列（mqueue/io\_uring）     | 内核 io\_uring 驱动（`IORING_OP_URING_CMD`）                   | `ipc.h`：128B 消息头 + magic 'ARE1'         |
| **记忆模型**（MemoryRovol） | 用户态 heapstore（malloc + mmap）  | 内核态 L1-L4 分层（`alloc_pages` + `mmap` + pmem）      | `memory_types.h`：L1-L4 + GFP 掩码         |

### 5.3 行为契约测试

\[SS] 层的语义一致性通过**行为契约测试**验证。两端针对相同输入验证输出一致：

```c
/* 行为契约测试示例：IPC 消息头序列化（Layout C v4） */
static void test_ipc_hdr_serialize(void)
{
	struct airy_ipc_msg_hdr hdr = {
		.magic		= AIRY_IPC_MAGIC,	/* 'ARE1' 0x41524531 */
		.opcode		= AIRY_IPC_OP_SEND,	/* 0x0001 */
		.flags		= AIRY_IPC_F_CAP_CARRY,	/* 携带 Badge */
		.trace_id	= 0x1234,
		.timestamp_ns	= 0,
		.src_task	= 0x100,
		.dst_task	= 0x200,
		.payload_len	= 256,
		.capability_badge = 0,		/* agentrt 用户态始终为 0 */
		.crc32		= 0,			/* 由发送方计算填入 */
	};

	/* 两端断言：序列化后字节序一致 */
	assert(sizeof(hdr) == AIRY_IPC_HDR_SIZE);	/* 128 */
	assert(hdr.magic == 0x41524531u);
	/* offset 44 处 capability_badge 字段存在且对齐 */
	assert(offsetof(struct airy_ipc_msg_hdr, capability_badge) == 44);
	assert(offsetof(struct airy_ipc_msg_hdr, crc32) == 52);
}
```

两端共享同一测试套件（`tests/shared/contract/`），CI 中两端分别运行并比对结果。

***

## 6. \[IND] 完全独立层

> **权威源声明**：[IND] 层的权威定义见 [06-iron9-shared-model.md §4](../10-architecture/06-iron9-shared-model.md)。本节列出两端 [IND] 层的工程落地细节，含 io_uring + IORING_OP_URING_CMD（不使用 page flipping）、alloc_pages + mmap（不使用 DMA 一致性内存）、纯 C LSM（不使用 BPF LSM）三项关键技术选型。

### 6.1 agentrt-linux 专属（\[IND]）

| 模块               | 内容                                     | 理由                      |
| ---------------- | -------------------------------------- | ----------------------- |
| 内核驱动框架           | `drivers/airymax/accel/`、`gpu/`、`npu/` | 内核态专属，agentrt 不涉及       |
| Kbuild/Kconfig   | `Kbuild`、`Kconfig`、`Makefile`          | 内核构建系统专属                |
| 内核内部 API         | `printk`、`kmalloc`、`spin_lock`         | 内核态 API，agentrt 用用户态等价物 |
| systemd 集成       | `systemd/airymax-*.service`            | 发行版集成专属                 |
| LSM 实现           | 纯 C LSM 模块（`security/airy/`）       | 内核安全模块专属，**不使用 BPF LSM**（纯 C 实现，对齐 openEuler 252 钩子） |
| eBPF struct\_ops | `kernel/bpf/airymax/`                  | 内核 BPF 框架专属             |
| io\_uring 命令路径    | `IORING_OP_URING_CMD`（agentrt-linux 内核驱动） | 内核态专属，IPC fastpath 载体，**不使用 page flipping**（zero-copy via registered buffer + uring_cmd） |
| 物理内存分配           | `alloc_pages` + `mmap`（agentrt-linux 内核） | 内核态专属，日志 Ring Buffer / IPC 共享内存方案，**不使用 DMA 一致性内存**（streaming DMA via page mapping，对齐 Linux 6.6 DMA API） |

### 6.2 agentrt 专属（\[IND]）

| 模块        | 内容                                                   | 理由                                             |
| --------- | ---------------------------------------------------- | ---------------------------------------------- |
| 跨平台用户态运行时 | `runtime/`（C/Rust 实现）                                | 跨平台（Linux/macOS/Windows），agentrt-linux 仅 Linux |
| SDK 四语言   | `sdk/python/`、`sdk/rust/`、`sdk/typescript/`、`sdk/c/` | 多语言绑定，agentrt-linux 仅 C                        |
| CLI/TUI   | `cli/`、`tui/`                                        | 用户交互工具，agentrt-linux 无                         |
| 用户态内存管理   | `runtime/heapstore`                                  | 用户态 malloc/mmap，agentrt-linux 用内核 kmalloc      |

***

## 7. 双向 CI 校验机制

### 7.1 校验流程

\[SC] 层（10 个头文件）的任何变更必须经过**双向 CI 校验**，确保两端均能编译通过：

```
开发者提交 [SC] 变更 MR（agentrt-linux 仓库）
   ↓
步骤 1: agentrt-linux CI 校验
  - 编译 agentrt-linux 内核
  - 运行 [SC] 行为契约测试
  - checkpatch + clang-format + kernel-doc 校验
   ↓ pass
步骤 2: 镜像 PR 到 agentrt 仓库
  - 自动创建 agentrt 仓库的镜像 PR
  - agentrt CI 校验：编译 agentrt 用户态
  - 运行 [SC] 行为契约测试（与 agentrt-linux 同一套测试套件）
   ↓ pass
步骤 3: L1 子系统维护者审查（airymax-shared-contract）
   ↓ approve
步骤 4: L3 总维护者（SPHARX Engineering）最终审批
   ↓ approve
合并至 agentrt-linux main
   ↓
步骤 5: 自动合并至 agentrt main（镜像同步）
```

### 7.2 CI 配置（agentrt-linux 侧）

```yaml
# 文件: .github/workflows/sc-contract.yml
sc-contract-bidirectional:
  stage: integration
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - include/uapi/linux/airymax/*.h
  script:
    # 步骤 1: 本仓编译与测试
    - make build
    - make test-contract

    # 步骤 2: 创建 agentrt 镜像 PR 并等待结果
    - ./scripts/airymax/mirror_sc_pr.sh
    - ./scripts/airymax/wait_mirror_ci.sh

    # 步骤 3: 校验两端测试结果一致
    - ./scripts/airymax/compare_contract_results.sh
```

### 7.3 校验失败处理

- **agentrt-linux CI 失败**：直接拒绝合并，开发者修复
- **agentrt CI 失败**：拒绝合并，开发者需同时考虑两端兼容性
- **测试结果不一致**：拒绝合并，需对齐两端实现

### 7.4 \[SC] 变更分类

| 变更类型                 | 处理方式                               |
| -------------------- | ---------------------------------- |
| **新增字段/枚举**（向后兼容）    | 双向 CI 校验，两端同步更新                    |
| **修改字段语义**（breaking） | 必须升级 magic 版本（如 'ARE1' → 'ARE2'）   |
| **删除字段**（breaking）   | 禁止直接删除，先标记 `__deprecated` 一个版本，再删除 |
| **纯文档/注释**           | 仅本仓 CI 校验，无需 agentrt 镜像            |

### 7.5 magic 值 CI 校验

[SC] 层的 magic 值在双向 CI 中进行校验，确保两端字节一致且不可篡改。magic 值一经发布即冻结，CI 校验任何对 magic 的修改均为合并阻断错误：

| magic         | 十六进制         | ASCII  | 物理宿主 | CI 校验方式 |
| ------------- | ------------ | ------ | -------- | -------- |
| IPC 消息头 magic | `0x41524531` | 'ARE1' | `include/uapi/linux/airymax/ipc.h` | `sc-dual-ci.yml` 逐字节验证 + 行为契约测试（`assert(hdr.magic == 0x41524531u)`） |
| 任务描述符 magic   | `0x41475453` | 'AGTS' | `include/uapi/linux/airymax/sched.h` | `sc-dual-ci.yml` 逐字节验证 + 行为契约测试（`assert(desc.magic == 0x41475453u)`） |

> **变更禁令**：magic 值一经发布即冻结，禁止修改。如需协议演进，新增 magic（如 'ARE2'）而非修改现有 magic。CI 校验对 magic 值的任何修改均为合并阻断错误。

***

## 8. kthread 间通信约束

### 8.1 Linux 6.6 内核基线 基线约束

agentrt-linux 内核态 kthread 间通信**必须**使用 `kfifo` + `wait_event_interruptible`，禁止使用浮点（用 `airy_q16_t`）：

```c
#include <linux/kfifo.h>
#include <linux/wait.h>

struct airy_kthread_chan {
	DECLARE_KFIFO(queue, struct airy_ipc_msg_hdr, 16);
	wait_queue_head_t	wq;
	spinlock_t		lock;
};

static int airy_kthread_recv(struct airy_kthread_chan *chan,
				struct airy_ipc_msg_hdr *out)
{
	int ret;

	/* 阻塞等待消息 */
	ret = wait_event_interruptible(chan->wq,
		!kfifo_is_empty(&chan->queue));
	if (ret)
		return -EINTR;

	spin_lock(&chan->lock);
	ret = kfifo_out(&chan->queue, out, 1);
	spin_unlock(&chan->lock);

	return ret ? 0 : -EAGAIN;
}
```

***

## 9. 速查表：四层模型速查

| 层次     | 共享程度 | 内容              | 落地路径               | CI 校验  |
| ------ | ---- | --------------- | ------------------ | ------ |
| \[SC]  | 完全共享 | 10 个头文件 + magic  | `include/uapi/linux/airymax/` | 双向 CI  |
| \[SS]  | 语义同源 | 调度/安全/IPC/记忆    | 各自实现               | 行为契约测试 |
| \[IND] | 完全独立 | 内核驱动/Kbuild/SDK | 各自仓库               | 各自 CI  |
| \[DSL] | 自包含降级块 | `#ifdef AIRY_SC_FALLBACK` 降级块（38 POSIX 码 + 最简 IPC + EEVDF 默认调度） | [SC] 头文件底部 | 降级块 hash 校验 |

### 9.1 magic 速查

| magic | 十六进制         | ASCII  | 场景      | 层     |
| ----- | ------------ | ------ | ------- | ----- |
| IPC   | `0x41524531` | 'ARE1' | IPC 消息头 | \[SC] |
| 任务    | `0x41475453` | 'AGTS' | 任务描述符   | \[SC] |

### 9.2 调度编号速查

| 常量            | 值 | Linux 6.6 内核基线 行号    | 约束                                 |
| ------------- | - | ------------- | ---------------------------------- |
| SCHED\_NORMAL | 0 | `sched.h:114` | —                                  |
| SCHED\_FIFO   | 1 | `sched.h:115` | —                                  |
| SCHED\_EXT    | 7 | `sched.h:121` | **Linux 6.6 内核基线 内置，禁止定义 SCHED\_AGENT 宏** |

***

## 10. 历史与变更记录

| 日期         | 版本    | 变更摘要                                           | 责任人          |
| ---------- | ----- | ---------------------------------------------- | ------------ |
| 2026-07-09 | 0.1.1 | 初始创建，定义 IRON-9 v3 四层模型 + 10 个 \[SC] 头文件 + 双向 CI | SPHARX 工程标准组 |
| 2026-07-17 | v1.0  | 升级为 IRON-9 v3 四层模型（[SC]+[SS]+[IND]+[DSL]）；新增 §3 [DSL] 降级生存层（引用 11-degraded-survival-layer.md）；[IND] 层补充 io_uring + IORING_OP_URING_CMD、alloc_pages + mmap、纯 C LSM；§2.1 头文件清单与 06-iron9-shared-model.md §2.2 对齐；§7.5 新增 magic 值 CI 校验表（含 AGTS 0x41475453）；上级文档改为 10-unify-design.md | SPHARX 工程标准组 |

***

> **SPDX-License-Identifier**： AGPL-3.0-or-later OR Apache-2.0

