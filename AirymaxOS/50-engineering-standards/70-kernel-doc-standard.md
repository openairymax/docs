Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）kernel-doc 注释规范

> **文档定位**： agentrt-linux（AirymaxOS，极境智能体操作系统）内核态 C 代码的 kernel-doc 注释强制规范。定义 `/** */` 注释格式、`@arg:`/`Return:`/`Context:` 等字段模板，以及 `struct`/`enum`/`function`/`typedef` 等结构的文档模板。\
> **版本**： 0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**： 2026-07-09\
> **理论根基**： Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**： AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 目的与范围

本规范定义 agentrt-linux 内核态 C 代码的 kernel-doc 注释强制标准，源自由 OLK-6.6 `scripts/kernel-doc`（68KB Perl 脚本）所解析的内核文档注释语法。所有导出函数、公共结构体、枚举、typedef 必须有 kernel-doc 注释，CI 通过 `scripts/kernel-doc` 提取并校验。

### 0.2 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）
- 共享原语称"微核心原语"（不称"微内核原语"）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

### 0.3 OLK-6.6 源码路径

- `scripts/kernel-doc`（68969 字节，Perl 脚本，第 1-50 行头部说明）
- `scripts/kernel-doc:45-50` —— "Read C language source or header FILEs, extract embedded documentation comments"
- `scripts/kernel-doc:48` —— "The documentation comments are identified by the \"/**\" opening comment mark."
- `scripts/kernel-doc:62-77` —— 类型匹配正则定义（`$type_constant`/`$type_func`/`$type_param` 等）

---

## 1. kernel-doc 注释基础规则

### 1.1 规则 OS-STD-DOC-001：`/** */` 起始标记强制

kernel-doc 注释**必须**以 `/**` 起始（双星号），以 `*/` 结尾。普通注释 `/* */`（单星号）不被 kernel-doc 脚本识别。

**OLK-6.6 源码路径**: `scripts/kernel-doc:48`

**正确示例**：

```c
/**
 * agentrt_ipc_send() - Send a message on an IPC channel
 * @chan: IPC channel to send on
 * @msg: Message to send
 *
 * Context: Process context. Takes and releases chan->lock.
 * Return: 0 on success, negative errno on failure.
 */
int agentrt_ipc_send(struct agentrt_ipc_chan *chan,
		     const struct agentrt_ipc_msg *msg);
```

**错误示例**：

```c
/* WRONG: 单星号，不被 kernel-doc 识别 */
/* agentrt_ipc_send - Send a message
 * @chan: IPC channel
 */
int agentrt_ipc_send(struct agentrt_ipc_chan *chan, ...);
```

### 1.2 规则 OS-STD-DOC-002：短描述（short description）强制

每个 kernel-doc 注释第一行**必须**包含 `<符号名>() - <短描述>` 形式的短描述。短描述应在一行内完成，描述符号的核心用途。

**OLK-6.6 源码路径**: `scripts/kernel-doc` 短描述解析逻辑（`-Wshort-description` 警告）

**正确示例**：

```c
/**
 * agentrt_sched_pick_next() - Pick the next runnable agent task
 * @sched: Scheduler context
 *
 * Select the highest-priority runnable task from the ready queue.
 *
 * Return: Pointer to the picked task, or NULL if no task is runnable.
 */
```

**错误示例**：

```c
/**
 * agentrt_sched_pick_next	/* WRONG: 无短描述，缺 " - " 分隔 */
 * @sched: Scheduler context
 */
```

---

## 2. 函数文档模板

### 2.1 规则 OS-STD-DOC-003：函数 kernel-doc 字段顺序

函数 kernel-doc 注释**必须**按以下字段顺序编写：

1. 短描述（`symbol() - short desc`）
2. `@arg:` 参数描述（每个参数一行，含 `.x`/`->x` 成员引用）
3. 空行分隔
4. 长描述（可选，描述算法/约束）
5. `Context:` 上下文约束（进程上下文/原子上下文/可睡眠/持锁要求）
6. `Return:` 返回值描述（含错误码语义）

**OLK-6.6 源码路径**: `scripts/kernel-doc:66` (`$type_param` 正则 `\@(\w*((\.\w+)|(->\w+))*(\.\.\.)?)`)

**完整模板**：

```c
/**
 * agentrt_ipc_send_batch() - Send a batch of IPC messages atomically
 * @chan: IPC channel to send on; must be initialized.
 * @msgs: Array of messages to send; length must equal @count.
 * @count: Number of messages in @msgs; must be > 0 and <= 64.
 * @flags: Send flags, bitwise OR of %AGENTRT_IPC_F_NOWAIT and
 *         %AGENTRT_IPC_F_SIGNAL.
 *
 * Sends @count messages from @msgs to @chan atomically. If
 * %AGENTRT_IPC_F_NOWAIT is set, the function returns immediately if the
 * channel buffer is full; otherwise it blocks.
 *
 * The function is idempotent on failure: no partial send occurs.
 *
 * Context: Process context. May sleep if @flags does not include
 *          %AGENTRT_IPC_F_NOWAIT. Takes and releases @chan->lock.
 * Return:
 * * 0 on success.
 * * -ENOMEM if internal allocation fails.
 * * -EAGAIN if %AGENTRT_IPC_F_NOWAIT is set and buffer is full.
 * * -E2BIG if @count exceeds the channel maximum.
 */
int agentrt_ipc_send_batch(struct agentrt_ipc_chan *chan,
			   const struct agentrt_ipc_msg *msgs,
			   size_t count, u32 flags);
```

### 2.2 规则 OS-STD-DOC-004：`@arg:` 参数描述强制

所有函数参数（含 `void`/`struct` 指针）**必须**有 `@arg:` 描述行。`...` 可变参数用 `@...:` 描述。参数描述应包含取值范围、单位、是否可空（NULL）、是否被修改。

**OLK-6.6 源码路径**: `scripts/kernel-doc:66` (`$type_param`)

**正确示例**：

```c
/**
 * agentrt_mem_reserve() - Reserve memory for agent task
 * @task: Task to reserve memory for; must not be NULL.
 * @size: Reserve size in bytes; must be > 0 and power of 2.
 * @gfp: GFP flags; %GFP_KERNEL or %GFP_ATOMIC.
 * @...: Variable format arguments for reservation tag, printf-style.
 */
```

### 2.3 规则 OS-STD-DOC-005：`Return:` 返回值强制

所有非 `void` 函数**必须**有 `Return:` 字段，描述成功值与所有错误码（含负 errno）。错误码以 `-EXXX` 形式列出，并解释触发条件。

**OLK-6.6 源码路径**: `scripts/kernel-doc:26` (`-Wreturn` 警告开关)

**正确示例**：

```c
/**
 * Context: Process context. Takes @lock.
 * Return:
 * * 0 - Success.
 * * -EINVAL - Invalid @task state.
 * * -ENOMEM - Memory allocation failed.
 * * -EBUSY  - @task already has a reservation.
 */
```

### 2.4 规则 OS-STD-DOC-006：`Context:` 上下文约束强制

所有可能涉及并发安全的函数**必须**有 `Context:` 字段，说明：(1) 进程上下文 vs 原子上下文；(2) 是否可睡眠；(3) 持锁要求（持何种锁、释放何种锁）。

**OLK-6.6 源码路径**: `scripts/kernel-doc` `Context:` 解析逻辑

**正确示例**：

```c
 * Context: Atomic context. Must be called with @chan->lock held.
 * Context: Process context. May sleep. Takes and releases @ctx->mutex.
```

---

## 3. 结构体文档模板

### 3.1 规则 OS-STD-DOC-007：struct kernel-doc 字段顺序

结构体 kernel-doc 注释**必须**按以下顺序：

1. 短描述（`struct foo - short desc`）
2. 长描述（可选）
3. 每个成员的 `@member:` 描述行

**OLK-6.6 源码路径**: `scripts/kernel-doc:72` (`$type_struct` 正则 `\&(struct\s*([_\w]+))`)

**完整模板**：

```c
/**
 * struct agentrt_task - Agent task descriptor (micro-core primitive)
 * @id: Unique task ID; assigned at creation, never recycled.
 * @link: List node for ready queue linkage; owned by scheduler.
 * @state: Task state, see enum agentrt_task_state.
 * @token_budget: Token budget in Q16.16 fixed-point; see airymax_q16_t.
 * @magic: Magic 0x41475453 ('AGTS') for descriptor validation.
 *
 * Describes a single agent task managed by the micro-core scheduler.
 * Shared between agentrt (user-space) and agentrt-linux (kernel)
 * via the [SC] contract layer.
 *
 * Locking: state and token_budget are protected by the owning
 * scheduler's lock.
 */
struct agentrt_task {
	u64			id;
	struct list_head	link;
	enum agentrt_task_state	state;
	airymax_q16_t		token_budget;
	__u32			magic;		/* 0x41475453 'AGTS' */
};
```

### 3.2 规则 OS-STD-DOC-008：嵌套成员引用

结构体成员若为嵌套结构体，kernel-doc 中用 `@parent.child:` 或 `@parent->child:` 形式描述子成员。

**OLK-6.6 源码路径**: `scripts/kernel-doc:66` (`$type_param` 含 `(\.\w+)|(->\w+)`)

**正确示例**：

```c
/**
 * struct agentrt_ipc_ctx - IPC context
 * @tx.queue: Transmit queue head; protected by @tx.lock.
 * @tx.depth: Maximum pending entries in @tx.queue.
 * @rx.queue: Receive queue head; protected by @rx.lock.
 * @rx.depth: Maximum pending entries in @rx.queue.
 */
struct agentrt_ipc_ctx {
	struct {
		struct list_head	queue;
		size_t			depth;
		spinlock_t		lock;
	} tx, rx;
};
```

---

## 4. 枚举文档模板

### 4.1 规则 OS-STD-DOC-009：enum kernel-doc 字段顺序

枚举 kernel-doc 注释**必须**按以下顺序：

1. 短描述（`enum foo - short desc`）
2. 长描述（可选）
3. 每个枚举值的 `@value:` 描述行

**OLK-6.6 源码路径**: `scripts/kernel-doc:71` (`$type_enum` 正则 `\&(enum\s*([_\w]+))`)

**完整模板**：

```c
/**
 * enum agentrt_task_state - Agent task lifecycle states
 * @TASK_NEW: Task created but not yet enqueued.
 * @TASK_RUNNABLE: Task is in the ready queue, eligible for scheduling.
 * @TASK_RUNNING: Task is currently executing on a CPU.
 * @TASK_BLOCKED: Task is waiting on an IPC message or I/O.
 * @TASK_DEAD: Task has exited; descriptor pending reclamation.
 *
 * States transition in a unidirectional cycle:
 * NEW -> RUNNABLE -> RUNNING -> BLOCKED -> RUNNABLE -> ... -> DEAD.
 * The DEAD state is terminal.
 */
enum agentrt_task_state {
	TASK_NEW,
	TASK_RUNNABLE,
	TASK_RUNNING,
	TASK_BLOCKED,
	TASK_DEAD,
};
```

---

## 5. 函数指针与回调文档模板

### 5.1 规则 OS-STD-DOC-010：函数指针参数描述

结构体中函数指针成员的 kernel-doc **必须**用 `@member():` 形式（带括号）描述，并说明回调契约（何时调用、参数语义、返回值约定）。

**OLK-6.6 源码路径**: `scripts/kernel-doc:68-69` (`$type_fp_param`/`$type_fp_param2`)

**完整模板**：

```c
/**
 * struct agentrt_sched_ops - Scheduler operations vector
 * @pick_next(): Select next runnable task; returns NULL if queue empty.
 * @enqueue(): Enqueue a task into the ready queue; called with @lock held.
 * @dequeue(): Dequeue a task from the ready queue; called with @lock held.
 * @yield(): Yield current task; re-queue at tail of its priority band.
 *
 * All callbacks are invoked under the scheduler's @lock. Callbacks
 * must not sleep or block.
 */
struct agentrt_sched_ops {
	struct agentrt_task *(*pick_next)(struct agentrt_sched *sched);
	void (*enqueue)(struct agentrt_sched *sched,
			struct agentrt_task *task);
	void (*dequeue)(struct agentrt_sched *sched,
			struct agentrt_task *task);
	void (*yield)(struct agentrt_sched *sched);
};
```

---

## 6. typedef 文档模板

### 6.1 规则 OS-STD-DOC-011：仅允许 typedef 的 kernel-doc

agentrt-linux 禁止结构体 typedef（OS-KER-013），但允许 (a)-(e) 类 typedef（如 `airymax_q16_t`）。这些允许的 typedef **必须**有 kernel-doc 注释，描述其底层类型与语义。

**OLK-6.6 源码路径**: `scripts/kernel-doc:73` (`$type_typedef` 正则)

**完整模板**：

```c
/**
 * typedef airymax_q16_t - Q16.16 fixed-point value
 *
 * Signed 32-bit integer representing a fixed-point value: high 16
 * bits are the integer part, low 16 bits are the fractional part.
 * Range: [-32768.0, 32767.99998].
 *
 * Required in kernel code because x86 builds with -mno-80387
 * (no x87 FPU). See arch/x86/Makefile:137.
 *
 * Use airymax_q16_mul(), airymax_q16_div() helpers for arithmetic;
 * never operate on the raw int32_t.
 */
typedef int32_t airymax_q16_t;
```

---

## 7. 交叉引用与高亮

### 7.1 规则 OS-STD-DOC-012：交叉引用标记

kernel-doc 注释中引用其他符号时**必须**使用以下标记：

| 标记 | 含义 | OLK-6.6 正则 |
|------|------|------------|
| `%CONST` | 宏常量 | `$type_constant2` (`\%([-_\w]+)`) |
| `func()` | 函数引用 | `$type_func` (`(\w+)\(\)`) |
| `&struct foo` | 结构体引用 | `$type_struct` |
| `&enum foo` | 枚举引用 | `$type_enum` |
| `&typedef foo` | typedef 引用 | `$type_typedef` |
| `&union foo` | 联合体引用 | `$type_union` |
| `&foo.bar` | 成员引用 | `$type_member` |

**OLK-6.6 源码路径**: `scripts/kernel-doc:63-77`（高亮正则定义）

**正确示例**：

```c
/**
 * agentrt_sched_init() - Initialize a scheduler
 * @sched: Scheduler context; must be zero-initialized.
 *
 * Initializes @sched with default ops from %AGENTRT_SCHED_DEFAULT_OPS.
 * The ready queue is built via &struct list_head. After init, call
 * agentrt_sched_pick_next() to dispatch tasks.
 *
 * See also: &enum agentrt_task_state, &typedef airymax_q16_t.
 *
 * Context: Process context. May sleep.
 * Return: 0 on success, -EINVAL if @sched is already initialized.
 */
```

---

## 8. 文件级文档

### 8.1 规则 OS-STD-DOC-013：文件级 kernel-doc 注释

每个 `.c` 文件**必须**在文件顶部（许可证标识后、`#include` 前）包含文件级 kernel-doc 注释，格式为：

```c
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
/**
 * DOC: Agent IPC channel implementation
 *
 * This file implements the agentrt-linux IPC channel, which is the
 * kernel-side counterpart of the agentrt user-space IPC. The protocol
 * is defined in include/airymax/ipc.h ([SC] contract layer).
 *
 * The IPC message header magic 0x41524531 ('ARE1') is shared with
 * agentrt; see 120-cross-project-code-sharing.md.
 */
```

**OLK-6.6 源码路径**: `scripts/kernel-doc` `DOC:` 段解析逻辑

---

## 9. CI 校验与门禁

### 9.1 规则 OS-STD-DOC-014：kernel-doc CI 校验

agentrt-linux CI 必须在每次合并请求中调用 `scripts/kernel-doc` 校验所有新增/修改的 `.c`/`.h` 文件：

```bash
# 检查所有导出符号是否有 kernel-doc
$ ./scripts/kernel-doc -Wall -Wreturn -Wshort-description \
    -Wcontents-before-sections \
    -export $(git diff --name-only origin/main...HEAD -- '*.c' '*.h')

# 检查所有内部符号（含 static）
$ ./scripts/kernel-doc -Wall -internal \
    $(git diff --name-only origin/main...HEAD -- '*.c' '*.h')
```

### 9.2 校验级别

| 校验开关 | 含义 | agentrt-linux 处理 |
|---------|------|------------------|
| `-Wall` | 启用所有警告 | 强制 |
| `-Wreturn` | 检查 `Return:` 缺失 | 强制 |
| `-Wshort-description` | 检查短描述缺失 | 强制 |
| `-Wcontents-before-sections` | 检查章节前内容 | 强制 |
| `-Werror` | 警告升级为错误 | 强制 |

### 9.3 必须有 kernel-doc 的符号清单

| 符号类型 | 是否强制 kernel-doc | 例外 |
|---------|------------------|------|
| `EXPORT_SYMBOL` 导出的函数 | 是 | 无 |
| `EXPORT_SYMBOL_GPL` 导出的函数 | 是 | 无 |
| 公共结构体（在 .h 中声明） | 是 | 内部辅助结构体可豁免 |
| 公共枚举 | 是 | 内部辅助枚举可豁免 |
| 公共 typedef | 是 | 无 |
| `static` 内部函数 | 否（建议） | 算法复杂时建议 |
| 内联汇编 | 否 | 但应有行内注释解释指令 |

---

## 10. 完整示例：[SC] 头文件 kernel-doc 范例

以下为 `include/airymax/ipc.h`（[SC] 共享契约层头文件）的 kernel-doc 注释范例：

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
/**
 * DOC: Airymax IPC shared contract ([SC] layer)
 *
 * This header is shared verbatim between agentrt (user-space micro-core)
 * and agentrt-linux (kernel micro-kernel). Changes here trigger
 * bidirectional CI validation; see 120-cross-project-code-sharing.md.
 *
 * The IPC magic 0x41524531 ('ARE1') is constant and must never change.
 */

#ifndef _AIRYMAX_IPC_H
#define _AIRYMAX_IPC_H

#include <linux/types.h>

#define AIRYMAX_IPC_MAGIC		0x41524531	/* 'ARE1' */
#define AIRYMAX_IPC_HDR_SZ		128

/**
 * struct agentrt_ipc_msg_hdr - IPC message header (128 bytes)
 * @magic: Magic 0x41524531 ('ARE1'); validates protocol version.
 * @opcode: Operation code; see enum agentrt_ipc_opcode.
 * @flags: Message flags, bitwise OR of %AGENTRT_IPC_F_*.
 * @trace_id: Trace identifier; propagates across hops for debugging.
 * @timestamp_ns: Sender timestamp in nanoseconds (CLOCK_MONOTONIC).
 * @src_task: Source task ID; 0 for kernel-originated messages.
 * @dst_task: Destination task ID; 0 for broadcast.
 * @payload_len: Payload length in bytes; 0 if no payload.
 * @reserved: Reserved for future use; must be zero.
 *
 * Fixed 128-byte header shared between agentrt and agentrt-linux.
 * Layout is part of the [SC] contract layer and must not change
 * without coordinated update on both sides.
 *
 * All multi-byte fields are in host byte order within each side;
 * cross-endian transport is not supported.
 */
struct agentrt_ipc_msg_hdr {
	__u32	magic;			/* offset 0, 4 bytes */
	__u16	opcode;			/* offset 4, 2 bytes */
	__u16	flags;			/* offset 6, 2 bytes */
	__u64	trace_id;		/* offset 8, 8 bytes */
	__u64	timestamp_ns;		/* offset 16, 8 bytes */
	__u64	src_task;		/* offset 24, 8 bytes */
	__u64	dst_task;		/* offset 32, 8 bytes */
	__u32	payload_len;		/* offset 40, 4 bytes */
	__u8	reserved[84];		/* offset 44, 84 bytes */
} __attribute__((packed));

#endif /* _AIRYMAX_IPC_H */
```

---

## 11. 速查表：kernel-doc 字段速查

| 字段 | 用于 | 模板 | 强制性 |
|------|------|------|--------|
| 短描述 | 所有符号 | `symbol() - short desc` | 强制 |
| `@arg:` | 函数/struct/enum | `@name: description` | 强制 |
| `@...:` | 可变参数函数 | `@...: description` | 条件强制 |
| `Context:` | 函数 | `Context: Process/Atomic context. Locking.` | 条件强制 |
| `Return:` | 非 void 函数 | `Return: 0 on success, -EXXX on ...` | 强制 |
| `DOC:` | 文件级 | `DOC: File title` | 强制 |
| `%CONST` | 常量引用 | `%MACRO_NAME` | 建议 |
| `func()` | 函数引用 | `func_name()` | 建议 |
| `&struct` | 类型引用 | `&struct foo` | 建议 |

---

## 12. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，覆盖 14 条 kernel-doc 规则 | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**： AGPL-3.0-or-later OR Apache-2.0
