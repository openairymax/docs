Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）IRON-9 v2 三层共享模型落地规范

> **文档定位**： agentrt-linux（AirymaxOS，极境智能体操作系统）与 agentrt（微核心用户态运行时）之间的 IRON-9 v2 三层代码共享模型落地规范。详细说明 [SC] 共享契约层（6 个头文件）、[SS] 语义同源层、[IND] 完全独立层的实施细节，含双向 CI 校验机制与 magic 设计原理。\
> **版本**： 0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**： 2026-07-09\
> **理论根基**： Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**： AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 目的与范围

本文件是 IRON-9 v2 三层共享模型的工程落地规范，定义 agentrt（用户态微核心）与 agentrt-linux（内核微内核发行版）之间的代码共享边界、契约内容、校验机制。

### 0.2 术语约束（硬性）

- **agentrt**（用户态）= **微核心**（micro-core）。描述 agentrt 时使用"微核心"，其原语称"微核心原语"。**禁止**用"微内核原语"描述 agentrt。
- **agentrt-linux**（OS 发行版）= **微内核**（micro-kernel）。描述 agentrt-linux 时使用"微内核"。
- **禁止**使用特定上游发行版名称，统一用"主流 Linux 发行版"。
- 产品全称"agentrt-linux（AirymaxOS，极境智能体操作系统）"，简称"agentrt-linux"或"AirymaxOS"。

### 0.3 Linux 6.6 基线约束

- **SCHED_EXT=7**（OLK-6.6 内置，`include/uapi/linux/sched.h:121`）。agentrt-linux 复用 SCHED_EXT=7，**禁止**定义 `SCHED_AGENT` 宏（避免与内核调度类编号冲突）。
- 内核态禁 float（`arch/x86/Makefile:137` `-mno-80387`），用 Q16.16 定点数 `airymax_q16_t`（= `int32_t`）。
- kthread 间通信用 `kfifo` + `wait_event_interruptible`。

### 0.4 OLK-6.6 源码路径

- `include/uapi/linux/sched.h:114-123` —— SCHED_NORMAL=0 / SCHED_FIFO=1 / SCHED_EXT=7
- `arch/x86/Makefile:137-138` —— `-mno-80387` 禁浮点
- `scripts/checkpatch.pl:836-851` —— `%deprecated_apis` 表（参考共享 API 治理）

---

## 1. IRON-9 v2 三层模型总览

### 1.1 三层定义

| 层次 | 缩写 | 共享程度 | 内容 | 组织方式 |
|------|------|---------|------|---------|
| **共享契约层** | [SC] | 完全共享代码 | 6 个头文件，定义数据结构、magic、操作码、枚举 | `include/airymax/` 独立头文件库，两端共同依赖 |
| **语义同源层** | [SS] | API 签名相同，实现独立 | 调度语义、安全模型、IPC 传输、记忆模型 | 各自独立实现，通过 [SC] 保证互操作 |
| **完全独立层** | [IND] | 完全独立 | 内核驱动/Kbuild/内核 API（agentrt-linux）；跨平台用户态/SDK 四语言（agentrt） | 各自独立仓库，无依赖 |

### 1.2 与旧版 IRON-9 的差异

| 维度 | 旧版 IRON-9 | IRON-9 v2 |
|------|------------|-----------|
| 契约层代码 | 不共享，仅语义同源 | **完全共享**（6 个头文件） |
| 互操作方式 | 同源语义无适配层 | **共享代码无适配层**（增强） |
| CI 校验 | 无 | 契约层变更双向校验 |

---

## 2. [SC] 共享契约层

### 2.1 6 个 [SC] 头文件清单

[SC] 层包含**统一为 6 个**共享头文件，存放于 `include/airymax/` 目录，两端（agentrt 与 agentrt-linux）共同依赖，**逐字节相同**：

| 序号 | 文件 | 共享内容 | magic 值 | 落地路径 |
|------|------|---------|---------|---------|
| 1 | `bpf_struct_ops.h` | struct_ops 状态机 + common_value | — | `include/airymax/bpf_struct_ops.h` |
| 2 | `memory_types.h` | MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 持久化接口 | — | `include/airymax/memory_types.h` |
| 3 | `security_types.h` | POSIX capability 38 ID + LSM 钩子 254 ID + Cupolas blob 布局 + capability 派生模型 + Vault backend + 策略裁决 4 值枚举 | — | `include/airymax/security_types.h` |
| 4 | `cognition_types.h` | CoreLoopThree 阶段枚举 + Thinkdual 模式枚举 + LLM 推理阶段枚举 + 上下文结构 + Token 能效指标 + GPU/NPU 描述符 | — | `include/airymax/cognition_types.h` |
| 5 | `sched.h` | SCHED_EXT 约束（复用 7）+ 任务描述符（magic 0x41475453 'AGTS'）+ vtime 类型与衰减公式 + 优先级范围 + SLICE_DFL | `0x41475453` ('AGTS') | `include/airymax/sched.h` |
| 6 | `ipc.h` | IPC magic（0x41524531 'ARE1'）+ 128B 消息头结构（`agentrt_ipc_msg_hdr_t`）+ SQE/CQE 操作码与标志位 | `0x41524531` ('ARE1') | `include/airymax/ipc.h` |

**补充内容**（不属于上述 6 个头文件，但两端共享语义）：

- syscall 编号（`AGENTRT_SYS_*`）
- capability 令牌格式
- 错误码（`agentrt_error_t`）
- 规则编号体系（IRON/BAN/STD/ACC）
- 五维正交 24 原则

### 2.2 头文件 1：bpf_struct_ops.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRYMAX_BPF_STRUCT_OPS_H
#define _AIRYMAX_BPF_STRUCT_OPS_H

/**
 * struct airymax_struct_ops_value - shared struct_ops state
 * @state: Operation state machine value.
 * @common: Common value shared between BPF and scheduler.
 *
 * Shared between agentrt (user-space BPF loader) and
 * agentrt-linux (kernel struct_ops framework).
 */
struct airymax_struct_ops_value {
	__u32	state;
	__u64	common;
};

enum airymax_struct_ops_state {
	AIRYMAX_OPS_INIT	= 0,
	AIRYMAX_OPS_REGISTERED	= 1,
	AIRYMAX_OPS_ACTIVE	= 2,
	AIRYMAX_OPS_DRAINING	= 3,
};

#endif /* _AIRYMAX_BPF_STRUCT_OPS_H */
```

### 2.3 头文件 2：memory_types.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRYMAX_MEMORY_TYPES_H
#define _AIRYMAX_MEMORY_TYPES_H

#include <linux/types.h>

/* MemoryRovol L1-L4 hierarchy (shared semantics) */
enum airymax_mem_level {
	AIRYMAX_MEM_L1_HOT	= 0,	/* Hot working set, agent-local */
	AIRYMAX_MEM_L2_WARM	= 1,	/* Warm set, node-local */
	AIRYMAX_MEM_L3_COLD	= 2,	/* Cold set, node-remote */
	AIRYMAX_MEM_L4_PMEM	= 3,	/* Persistent memory */
};

/* GFP mask semantics for memory tier selection */
#define AIRYMAX_GFP_HOT		(__GFP_HIGHMEM | __GFP_MOVABLE)
#define AIRYMAX_GFP_WARM	(__GFP_HIGHMEM)
#define AIRYMAX_GFP_COLD	(0)
#define AIRYMAX_GFP_PMEM	(__GFP_HIGH)

#endif /* _AIRYMAX_MEMORY_TYPES_H */
```

### 2.4 头文件 3：security_types.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRYMAX_SECURITY_TYPES_H
#define _AIRYMAX_SECURITY_TYPES_H

#include <linux/types.h>

/* POSIX capability 38 IDs (subset shown) */
#define AIRYMAX_CAP_AGENT_SPAWN		38	/* Agent spawn capability */
#define AIRYMAX_CAP_GPU_SCHED		39	/* GPU scheduling */
#define AIRYMAX_CAP_NPU_ACCESS		40	/* NPU access */

/* LSM hook IDs (254 total) */
#define AIRYMAX_LSM_HOOK_TASK_CREATE	0
#define AIRYMAX_LSM_HOOK_IPC_SEND	1

/* Cupolas policy verdict (4 values) */
enum airymax_verdict {
	AIRYMAX_VERDICT_ALLOW	= 0,
	AIRYMAX_VERDICT_DENY	= 1,
	AIRYMAX_VERDICT_AUDIT	= 2,
	AIRYMAX_VERDICT_ASK	= 3,
};

#endif /* _AIRYMAX_SECURITY_TYPES_H */
```

### 2.5 头文件 4：cognition_types.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRYMAX_COGNITION_TYPES_H
#define _AIRYMAX_COGNITION_TYPES_H

#include <linux/types.h>

/* Q16.16 fixed-point: required because -mno-80387 (no FPU) */
typedef int32_t airymax_q16_t;
#define AIRYMAX_Q16_ONE		(1 << 16)

/* CoreLoopThree phases */
enum airymax_cog_phase {
	AIRYMAX_COG_PERCEPT	= 0,
	AIRYMAX_COG_THINK	= 1,
	AIRYMAX_COG_ACT		= 2,
};

/* Thinkdual modes */
enum airymax_think_mode {
	AIRYMAX_THINK_FAST	= 0,	/* System-1: fast, intuitive */
	AIRYMAX_THINK_SLOW	= 1,	/* System-2: slow, deliberative */
};

#endif /* _AIRYMAX_COGNITION_TYPES_H */
```

### 2.6 头文件 5：sched.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRYMAX_SCHED_H
#define _AIRYMAX_SCHED_H

#include <linux/types.h>

/* SCHED_EXT=7 (OLK-6.6 内置, include/uapi/linux/sched.h:121)
 * 禁止定义 SCHED_AGENT 宏, 复用 SCHED_EXT 调度类编号。 */

/* Task descriptor magic: 0x41475453 = 'AGTS' (Agent Task) */
#define AIRYMAX_TASK_MAGIC	0x41475453u

/* vtime type: Q16.16 fixed-point (no FPU) */
typedef int32_t airymax_vtime_t;

/* Priority range: 0 (highest) - 139 (lowest), compatible with Linux */
#define AIRYMAX_PRIO_MIN	0
#define AIRYMAX_PRIO_MAX	139

/* Default scheduling slice: 20ms */
#define AIRYMAX_SLICE_DFL_MS	20

/**
 * struct airymax_task_desc - Agent task descriptor ([SC] shared)
 * @magic: Must be AIRYMAX_TASK_MAGIC (0x41475453 'AGTS').
 * @prio: Priority in [AIRYMAX_PRIO_MIN, AIRYMAX_PRIO_MAX].
 * @vtime: Virtual time for fair scheduling (Q16.16).
 *
 * Shared between agentrt (user-space) and agentrt-linux (kernel).
 * The magic field validates descriptor integrity across the boundary.
 */
struct airymax_task_desc {
	__u32		magic;		/* 0x41475453 'AGTS' */
	__u16		prio;		/* 0-139 */
	__u16		_pad;
	airymax_vtime_t	vtime;		/* Q16.16 fixed-point */
};

#endif /* _AIRYMAX_SCHED_H */
```

### 2.7 头文件 6：ipc.h

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRYMAX_IPC_H
#define _AIRYMAX_IPC_H

#include <linux/types.h>

/* IPC magic: 0x41524531 = 'ARE1' (Airymax Runtime Engine v1) */
#define AIRYMAX_IPC_MAGIC	0x41524531u

/* Fixed 128-byte message header */
#define AIRYMAX_IPC_HDR_SZ	128

/* SQE/CQE operation codes */
enum airymax_ipc_op {
	AIRYMAX_IPC_OP_SEND		= 0,
	AIRYMAX_IPC_OP_RECV		= 1,
	AIRYMAX_IPC_OP_SEND_BATCH	= 2,
	AIRYMAX_IPC_OP_CANCEL		= 3,
};

/* Message flags */
#define AIRYMAX_IPC_F_NOWAIT	(1u << 0)
#define AIRYMAX_IPC_F_SIGNAL	(1u << 1)

/**
 * struct airymax_ipc_msg_hdr - IPC message header (128 bytes, [SC] shared)
 * @magic: Must be AIRYMAX_IPC_MAGIC (0x41524531 'ARE1').
 * @opcode: Operation code; see enum airymax_ipc_op.
 *
 * Fixed 128-byte header, layout never changes. Shared between
 * agentrt and agentrt-linux via [SC] contract layer.
 */
struct airymax_ipc_msg_hdr {
	__u32	magic;			/* offset 0, 'ARE1' */
	__u16	opcode;			/* offset 4 */
	__u16	flags;			/* offset 6 */
	__u64	trace_id;		/* offset 8 */
	__u64	timestamp_ns;		/* offset 16 */
	__u64	src_task;		/* offset 24 */
	__u64	dst_task;		/* offset 32 */
	__u32	payload_len;		/* offset 40 */
	__u8	reserved[84];		/* offset 44, 84 bytes */
} __attribute__((packed));

#endif /* _AIRYMAX_IPC_H */
```

---

## 3. magic 设计原理

### 3.1 IPC magic 0x41524531 ('ARE1')

| 字段 | 值 | 说明 |
|------|-----|------|
| 十六进制 | `0x41524531` | 大端表示 |
| ASCII | `'ARE1'` | Airymax Runtime Engine v1 |
| 用途 | IPC 消息头 magic | 验证 IPC 协议版本与消息完整性 |

**设计原理**：

1. **可识别性**：'ARE1' 是人类可读的 ASCII，便于 hexdump 调试时识别 IPC 消息
2. **版本化**：末尾 '1' 表示协议版本 1，未来 breaking change 升级为 'ARE2'
3. **跨端一致**：agentrt 与 agentrt-linux 两端字节相同，无字节序转换（host byte order 内部传输）
4. **不可变更**：magic 一经发布即冻结，变更必须升级版本号

### 3.2 任务描述符 magic 0x41475453 ('AGTS')

| 字段 | 值 | 说明 |
|------|-----|------|
| 十六进制 | `0x41475453` | 大端表示 |
| ASCII | `'AGTS'` | AGent Task State |
| 用途 | 任务描述符 magic | 验证任务描述符完整性 |

**设计原理**：

1. **与 IPC magic 区分**：'AGTS' ≠ 'ARE1'，避免任务描述符与 IPC 消息混淆
2. **场景区分**：IPC magic 用于消息边界验证，任务 magic 用于描述符生命周期验证
3. **[SC] 共享**：两端共享同一 magic，跨端传递任务描述符时无需额外标识

### 3.3 magic 常量锁定

| magic | 十六进制 | ASCII | 场景 | 共享层 |
|-------|---------|-------|------|--------|
| IPC 消息头 magic | `0x41524531` | 'ARE1' | IPC 协议识别 | [SC] |
| 任务描述符 magic | `0x41475453` | 'AGTS' | 任务描述符识别 | [SC] |

> **变更禁令**：magic 值一经发布即冻结，禁止修改。如需协议演进，新增 magic（如 'ARE2'）而非修改现有 magic。

---

## 4. [SS] 语义同源层

### 4.1 [SS] 层定义

[SS] 层中，agentrt 与 agentrt-linux 的 API 签名相同，但实现独立。两端通过 [SC] 契约层保证语义一致，无需适配层。

### 4.2 四大语义同源映射

| 语义域 | agentrt（用户态微核心）实现 | agentrt-linux（内核微内核）实现 | [SC] 契约依据 |
|--------|------------------------|--------------------------|--------------|
| **调度语义**（MicroCoreRT） | 用户态调度器（协程/线程池） | sched_ext BPF 调度器（SCHED_EXT=7） | `sched.h`：任务描述符 + vtime + 优先级 |
| **安全模型**（Cupolas） | 用户态策略引擎（seccomp + capability） | LSM 钩子（254 ID） + capability 38 ID | `security_types.h`：capability + verdict |
| **IPC 传输**（AgentsIPC） | 用户态消息队列（mqueue/io_uring） | 内核 io_uring 驱动 | `ipc.h`：128B 消息头 + magic 'ARE1' |
| **记忆模型**（MemoryRovol） | 用户态 heapstore（malloc + mmap） | 内核态 L1-L4 分层（kmalloc + pmem） | `memory_types.h`：L1-L4 + GFP 掩码 |

### 4.3 行为契约测试

[SS] 层的语义一致性通过**行为契约测试**验证。两端针对相同输入验证输出一致：

```c
/* 行为契约测试示例：IPC 消息头序列化 */
static void test_ipc_hdr_serialize(void)
{
	struct airymax_ipc_msg_hdr hdr = {
		.magic		= AIRYMAX_IPC_MAGIC,	/* 'ARE1' */
		.opcode		= AIRYMAX_IPC_OP_SEND,
		.flags		= AIRYMAX_IPC_F_NOWAIT,
		.trace_id	= 0x1234,
		.payload_len	= 256,
	};

	/* 两端断言：序列化后字节序一致 */
	assert(sizeof(hdr) == 128);
	assert(hdr.magic == 0x41524531u);
}
```

两端共享同一测试套件（`tests/shared/contract/`），CI 中两端分别运行并比对结果。

---

## 5. [IND] 完全独立层

### 5.1 agentrt-linux 专属（[IND]）

| 模块 | 内容 | 理由 |
|------|------|------|
| 内核驱动框架 | `drivers/airymax/accel/`、`gpu/`、`npu/` | 内核态专属，agentrt 不涉及 |
| Kbuild/Kconfig | `Kbuild`、`Kconfig`、`Makefile` | 内核构建系统专属 |
| 内核内部 API | `printk`、`kmalloc`、`spin_lock` | 内核态 API，agentrt 用用户态等价物 |
| systemd 集成 | `systemd/airymax-*.service` | 发行版集成专属 |
| LSM 实现 | `security/airymax/` | 内核安全模块专属 |
| eBPF struct_ops | `kernel/bpf/airymax/` | 内核 BPF 框架专属 |

### 5.2 agentrt 专属（[IND]）

| 模块 | 内容 | 理由 |
|------|------|------|
| 跨平台用户态运行时 | `runtime/`（C/Rust 实现） | 跨平台（Linux/macOS/Windows），agentrt-linux 仅 Linux |
| SDK 四语言 | `sdk/python/`、`sdk/rust/`、`sdk/typescript/`、`sdk/c/` | 多语言绑定，agentrt-linux 仅 C |
| CLI/TUI | `cli/`、`tui/` | 用户交互工具，agentrt-linux 无 |
| 用户态内存管理 | `runtime/heapstore` | 用户态 malloc/mmap，agentrt-linux 用内核 kmalloc |

---

## 6. 双向 CI 校验机制

### 6.1 校验流程

[SC] 层（6 个头文件）的任何变更必须经过**双向 CI 校验**，确保两端均能编译通过：

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

### 6.2 CI 配置（agentrt-linux 侧）

```yaml
# 文件: .github/workflows/sc-contract.yml
sc-contract-bidirectional:
  stage: integration
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - include/airymax/*.h
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

### 6.3 校验失败处理

- **agentrt-linux CI 失败**：直接拒绝合并，开发者修复
- **agentrt CI 失败**：拒绝合并，开发者需同时考虑两端兼容性
- **测试结果不一致**：拒绝合并，需对齐两端实现

### 6.4 [SC] 变更分类

| 变更类型 | 处理方式 |
|---------|---------|
| **新增字段/枚举**（向后兼容） | 双向 CI 校验，两端同步更新 |
| **修改字段语义**（breaking） | 必须升级 magic 版本（如 'ARE1' → 'ARE2'） |
| **删除字段**（breaking） | 禁止直接删除，先标记 `__deprecated` 一个版本，再删除 |
| **纯文档/注释** | 仅本仓 CI 校验，无需 agentrt 镜像 |

---

## 7. kthread 间通信约束

### 7.1 OLK-6.6 基线约束

agentrt-linux 内核态 kthread 间通信**必须**使用 `kfifo` + `wait_event_interruptible`，禁止使用浮点（用 `airymax_q16_t`）：

```c
#include <linux/kfifo.h>
#include <linux/wait.h>

struct airymax_kthread_chan {
	DECLARE_KFIFO(queue, struct airymax_ipc_msg_hdr, 16);
	wait_queue_head_t	wq;
	spinlock_t		lock;
};

static int airymax_kthread_recv(struct airymax_kthread_chan *chan,
				struct airymax_ipc_msg_hdr *out)
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

---

## 8. 速查表：三层模型速查

| 层次 | 共享程度 | 内容 | 落地路径 | CI 校验 |
|------|---------|------|---------|---------|
| [SC] | 完全共享 | 6 个头文件 + magic | `include/airymax/` | 双向 CI |
| [SS] | 语义同源 | 调度/安全/IPC/记忆 | 各自实现 | 行为契约测试 |
| [IND] | 完全独立 | 内核驱动/Kbuild/SDK | 各自仓库 | 各自 CI |

### 8.1 magic 速查

| magic | 十六进制 | ASCII | 场景 | 层 |
|-------|---------|-------|------|-----|
| IPC | `0x41524531` | 'ARE1' | IPC 消息头 | [SC] |
| 任务 | `0x41475453` | 'AGTS' | 任务描述符 | [SC] |

### 8.2 调度编号速查

| 常量 | 值 | OLK-6.6 行号 | 约束 |
|------|-----|------------|------|
| SCHED_NORMAL | 0 | `sched.h:114` | — |
| SCHED_FIFO | 1 | `sched.h:115` | — |
| SCHED_EXT | 7 | `sched.h:121` | **OLK-6.6 内置，禁止定义 SCHED_AGENT 宏** |

---

## 9. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，定义 IRON-9 v2 三层模型 + 6 个 [SC] 头文件 + 双向 CI | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**： AGPL-3.0-or-later OR Apache-2.0
