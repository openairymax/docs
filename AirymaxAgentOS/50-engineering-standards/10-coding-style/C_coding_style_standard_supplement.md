Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）C 语言编码规范强化补充

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）内核态 C 语言编码规范的强化补充规则，与 `C_coding_style_standard.md` 配套使用。本文件聚焦 12 条强化规则，每条规则附 OLK-6.6 源码路径，并以"正确示例 / 错误示例"成对呈现。
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档定位与适用范围

### 0.1 与基线文档的关系

本规范是 `10-coding-style/C_coding_style_standard.md`（基线）的强化补充，**不重复**基线中已涵盖的命名约定、函数长度、注释格式等通用规则，仅就以下 12 条"高违规率 / 高风险"强化规则给出可执行的强制约束：

1. Tab-8 缩进强制（OS-KER-011）
2. `goto` 集中错误处理（OS-KER-009）
3. `bool` 仅用于返回值与栈变量（OS-KER-008）
4. 内核态禁 `float`，强制 `airymax_q16_t` Q16.16 定点数（OS-KER-007）
5. 行宽 80 列硬上限（OS-KER-012）
6. 指定初始化器（designated initializers）强制（OS-STD-CODE-014）
7. `fallthrough;` 显式穿透注释（OS-STD-CODE-015）
8. 禁止结构体 `typedef`（OS-KER-013）
9. 零警告门禁（OS-KER-014）
10. `strscpy` 强制替代 `strcpy`/`strncpy`/`strlcpy`（OS-STD-CODE-010）
11. 禁止 `BUG()`/`BUG_ON()`，改用 `WARN()`（OS-KER-015）
12. `sizeof(*p)` 替代 `sizeof(struct xxx)`（OS-STD-CODE-012）

> **术语约束**：agentrt（用户态）= 微核心（micro-core）；agentrt-liunx（OS 发行版）= 微内核（micro-kernel）。本规范描述 agentrt-liunx 时使用"微内核"上下文，描述 agentrt 共享原语时使用"微核心原语"。

### 0.2 OLK-6.6 源码路径标注规范

本规范每条规则均以 `文件名:行号` 格式标注 OLK-6.6 源码出处（基准路径 `/home/spharx/SpharxWorks/01Reference/kernel-OLK-6.6/`），便于审查者复核与对齐。

---

## 1. 规则 OS-KER-011：Tab-8 缩进强制

### 1.1 规则文本

agentrt-liunx 内核态 C 代码**必须**使用 Tab 字符缩进，Tab 宽度固定为 8 个字符。禁止使用空格缩进（Kconfig 例外）。8 字符缩进是"代码复杂度的自然惩罚"：嵌套超过 3 层即提示应重构。

### 1.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst:21` —— "Tabs are 8 characters, and thus indentations are also 8 characters."
- `Documentation/process/coding-style.rst:31-39` —— "if you need more than 3 levels of indentation, you're screwed anyway"
- `.clang-format:645` —— `IndentWidth: 8`
- `.clang-format:687` —— `TabWidth: 8`
- `.clang-format:688` —— `UseTab: Always`

### 1.3 正确示例

```c
static int agentrt_sched_pick_next(struct agentrt_sched *sched)
{
	struct agentrt_task *task;		/* Tab 缩进 */

	spin_lock(&sched->lock);
	task = list_first_entry_or_null(&sched->ready_queue,
				       struct agentrt_task, link);
	if (task)
		list_del_init(&task->link);
	spin_unlock(&sched->lock);

	return task ? task->id : -ENOENT;
}
```

### 1.4 错误示例

```c
static int agentrt_sched_pick_next(struct agentrt_sched *sched)
{
    struct agentrt_task *task;		/* WRONG: 4 空格缩进 */

    spin_lock(&sched->lock);
    task = list_first_entry_or_null(&sched->ready_queue,
                                    struct agentrt_task, link);	/* WRONG: 续行空格对齐 */
    return task ? task->id : -ENOENT;
}
```

---

## 2. 规则 OS-KER-009：goto 集中错误处理

### 2.1 规则文本

函数存在多个出口且需要公共清理工作时，**必须**使用 `goto` 跳转到集中错误处理标签。标签名必须描述其行为（如 `out_free_buffer`），禁止 `err1:`/`err2:` 编号式命名。错误标签必须按资源申请的逆序排列，避免 "one err bugs"（释放未分配资源）。

### 2.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst:526-572` —— §7 "Centralized exiting of functions"
- `Documentation/process/coding-style.rst:574-595` —— "one err bugs" 反模式与拆分标签
- `Documentation/process/coding-style.rst:537` —— "Choose label names which say what the goto does"

### 2.3 正确示例

```c
static int agentrt_ipc_setup_channel(struct agentrt_ipc_ctx *ctx)
{
	int ret;
	void *rx_buf;
	void *tx_buf;

	rx_buf = kmalloc(AIRYMAX_IPC_BUF_SZ, GFP_KERNEL);
	if (!rx_buf)
		return -ENOMEM;

	tx_buf = kmalloc(AIRYMAX_IPC_BUF_SZ, GFP_KERNEL);
	if (!tx_buf) {
		ret = -ENOMEM;
		goto out_free_rx;
	}

	ctx->rx_buf = rx_buf;
	ctx->tx_buf = tx_buf;
	return 0;

out_free_rx:					/* 标签按逆序命名 */
	kfree(rx_buf);
	return ret;
}
```

### 2.4 错误示例

```c
static int agentrt_ipc_setup_channel(struct agentrt_ipc_ctx *ctx)
{
	void *rx_buf = kmalloc(AIRYMAX_IPC_BUF_SZ, GFP_KERNEL);
	void *tx_buf;

	if (!rx_buf)
		return -ENOMEM;
	tx_buf = kmalloc(AIRYMAX_IPC_BUF_SZ, GFP_KERNEL);
	if (!tx_buf)
		return -ENOMEM;		/* WRONG: 未释放 rx_buf，泄漏 */

	ctx->rx_buf = rx_buf;
	ctx->tx_buf = tx_buf;
	return 0;
}
```

---

## 3. 规则 OS-KER-008：bool 仅用于返回值与栈变量

### 3.1 规则文本

`bool` 类型**仅允许**用于：函数返回值、栈局部变量。禁止将 `bool` 用于结构体成员、函数参数（应使用 `flags` 位域）。当缓存行布局或大小敏感时禁止 `bool`（其大小与对齐随架构变化）。多真/假值应合并为位域或 `u8`。

### 3.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst:1021-1049` —— §17 "Using bool"
- `Documentation/process/coding-style.rst:1032-1034` —— "bool function return types and stack variables are always fine"
- `Documentation/process/coding-style.rst:1036-1038` —— "Do not use bool if cache line layout or size of the value matters"

### 3.3 正确示例

```c
static bool agentrt_task_is_runnable(const struct agentrt_task *task)
{
	return task->state == TASK_RUNNABLE;	/* 栈变量 / 返回值 OK */
}

static int agentrt_ipc_send(struct agentrt_ipc_chan *chan, u32 flags)
{
	/* flags 参数承载多个布尔开关，而非多个 bool 参数 */
	if (flags & AGENTRT_IPC_F_NOWAIT)
		...
}
```

### 3.4 错误示例

```c
struct agentrt_task {
	bool is_runnable;		/* WRONG: 结构体成员用 bool，缓存行敏感 */
	bool is_pinned;
	bool has_gpu;
};

static int agentrt_ipc_send(struct agentrt_ipc_chan *chan,
			    bool nowait, bool signal);	/* WRONG: 多 bool 参数 */
```

---

## 4. 规则 OS-KER-007：内核态禁 float，强制 airymax_q16_t

### 4.1 规则文本

agentrt-liunx 内核态代码**禁止**使用 `float`/`double` 浮点类型。x86 编译强制 `-mno-80387` 禁止 x87 指令生成。所有需要小数运算的场景**必须**使用 `airymax_q16_t`（`int32_t`）Q16.16 定点数：高 16 位为整数部分，低 16 位为小数部分。定点运算辅助宏定义于 `include/airymax/cognition_types.h`。

### 4.2 OLK-6.6 源码路径

- `arch/x86/Makefile:137` —— `KBUILD_CFLAGS += -mno-80387`
- `arch/x86/Makefile:138` —— `KBUILD_CFLAGS += $(call cc-option,-mno-fp-ret-in-387)`

### 4.3 正确示例

```c
#include <airymax/cognition_types.h>

/* 计算 token 吞吐率：tokens / elapsed_ms（Q16.16 定点） */
static airymax_q16_t agentrt_token_throughput(u64 tokens, u64 elapsed_ms)
{
	if (elapsed_ms == 0)
		return 0;

	/* AIRYMAX_Q16_ONE = 1 << 16，先乘后除保精度 */
	return (airymax_q16_t)div_u64(tokens << 16, elapsed_ms);
}
```

### 4.4 错误示例

```c
/* WRONG: 内核态使用 double */
static double agentrt_token_throughput(u64 tokens, u64 elapsed_ms)
{
	if (elapsed_ms == 0)
		return 0.0;
	return (double)tokens / (double)elapsed_ms;
}
```

---

## 5. 规则 OS-KER-012：行宽 80 列硬上限

### 5.1 规则文本

源代码单行**不得超过 80 列**。注释单行**不得超过 72 列**（注释缩进占 8 列，留 8 列右边界）。超出时必须按"对齐到开括号"风格折行。**禁止**折断用户可见字符串（如 `printk` 消息），以保证可 grep。续行必须显著短于父行并向右偏移。

### 5.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst:104` —— "The preferred limit on the length of a single line is 80 columns."
- `Documentation/process/coding-style.rst:106-108` —— "Statements longer than 80 columns should be broken into sensible chunks"
- `Documentation/process/coding-style.rst:116-117` —— "never break user-visible strings such as printk messages"
- `.clang-format:55` —— `ColumnLimit: 80`

### 5.3 正确示例

```c
static int agentrt_ipc_send_batch(struct agentrt_ipc_channel *chan,
				  const struct agentrt_ipc_msg *msgs,
				  size_t count, u32 flags)
{
	pr_warn("agentrt-ipc: channel %d send batch failed (count=%zu)\n",
		chan->id, count);			/* 字符串不折断 */
	return 0;
}
```

### 5.4 错误示例

```c
static int agentrt_ipc_send_batch(struct agentrt_ipc_channel *chan, const struct agentrt_ipc_msg *msgs, size_t count, u32 flags)	/* WRONG: >80 列 */
{
	pr_warn("agentrt-ipc: channel %d send batch "
		"failed (count=%zu)\n", chan->id, count);	/* WRONG: 折断用户可见字符串 */
	return 0;
}
```

---

## 6. 规则 OS-STD-CODE-014：指定初始化器强制

### 6.1 规则文本

结构体初始化**必须**使用 C99 指定初始化器（`= { .field = value }`），禁止位置初始化器（`= { value1, value2, ... }`）。指定初始化器对字段顺序变化、新增字段具有天然免疫力，且可读性远高于位置式。

### 6.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst`（贯穿示例，如 §3 switch 示例、§5 初始化示例）
- `scripts/checkpatch.pl:4662` —— `GLOBAL_INITIALISERS` 规则检查
- `scripts/checkpatch.pl:4670` —— `INITIALISED_STATIC` 规则检查

### 6.3 正确示例

```c
static const struct agentrt_ipc_ops airymax_ipc_default_ops = {
	.send		= agentrt_ipc_default_send,
	.recv		= agentrt_ipc_default_recv,
	.poll		= agentrt_ipc_default_poll,
	.release	= agentrt_ipc_default_release,
};

static struct agentrt_sched airymax_sched = {
	.name		= "airymax",
	.priority	= AIRYMAX_SCHED_PRIO,
	.flags		= 0,
};
```

### 6.4 错误示例

```c
/* WRONG: 位置初始化器，字段顺序依赖且不可读 */
static const struct agentrt_ipc_ops airymax_ipc_default_ops = {
	agentrt_ipc_default_send,
	agentrt_ipc_default_recv,
	agentrt_ipc_default_poll,
	agentrt_ipc_default_release,
};
```

---

## 7. 规则 OS-STD-CODE-015：fallthrough; 显式穿透注释

### 7.1 规则文本

`switch` 语句中所有 `case` 分支**必须**以下列之一结尾：`break;`、`fallthrough;`、`continue;`、`goto <label>;`、`return [expr];`。禁止隐式 fall-through。`fallthrough` 是伪关键字宏（展开为 `__attribute__((__fallthrough__))`），用于显式标记有意穿透。

### 7.2 OLK-6.6 源码路径

- `Documentation/process/deprecated.rst:200-227` —— "Implicit switch case fall-through"
- `Documentation/process/deprecated.rst:229-235` —— "All switch/case blocks must end in one of: break; fallthrough; continue; goto; return"
- `Documentation/process/coding-style.rst:59` —— `fallthrough;` 示例
- `scripts/checkpatch.pl:7344` —— `DEFAULT_NO_BREAK` 规则检查

### 7.3 正确示例

```c
switch (task->cog_state) {
case AGENTRT_COG_PERCEPT:
	agentrt_percept_run(task);
	fallthrough;			/* 显式标记有意穿透 */
case AGENTRT_COG_THINK:
	agentrt_think_run(task);
	break;
case AGENTRT_COG_ACT:
	agentrt_act_run(task);
	return 0;
default:
	pr_warn("unknown cog state %d\n", task->cog_state);
	break;
}
```

### 7.4 错误示例

```c
switch (task->cog_state) {
case AGENTRT_COG_PERCEPT:
	agentrt_percept_run(task);
case AGENTRT_COG_THINK:			/* WRONG: 隐式穿透，无 fallthrough; */
	agentrt_think_run(task);
	break;
}
```

---

## 8. 规则 OS-KER-013：禁止结构体 typedef

### 8.1 规则文本

**禁止**为结构体和指针定义新 `typedef`（如 `typedef struct { ... } foo_t;`）。结构体必须以 `struct foo` 形式声明与使用。`typedef` 仅允许用于：(a) 完全不透明对象（如 `pte_t`）；(b) 明确整数类型（如 `u8`/`u16`/`u32`/`u64`）；(c) sparse 类型检查；(d) 与 C99 标准类型相同的特殊情况；(e) 用户空间安全类型。`airymax_q16_t`（= `int32_t`）属 (b) 类，允许。

### 8.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst:359-440` —— §5 "Typedefs"
- `Documentation/process/coding-style.rst:362-378` —— "It's a mistake to use typedef for structures and pointers"
- `Documentation/process/coding-style.rst:436-440` —— "the rule should basically be to NEVER EVER use a typedef"
- `scripts/checkpatch.pl:4783-4790` —— `NEW_TYPEDEFS` 规则检查（"do not add new typedefs"）

### 8.3 正确示例

```c
struct agentrt_task {
	u64			id;
	struct list_head	link;
	enum agentrt_task_state	state;
	airymax_q16_t		token_budget;	/* (b) 类 typedef，允许 */
};

static int agentrt_task_enqueue(struct agentrt_task *task);	/* struct foo 直接使用 */
```

### 8.4 错误示例

```c
typedef struct {				/* WRONG: 结构体 typedef */
	u64			id;
	struct list_head	link;
	enum agentrt_task_state	state;
} agentrt_task_t;

typedef struct agentrt_task *agentrt_task_ptr_t;	/* WRONG: 指针 typedef */

static int agentrt_task_enqueue(agentrt_task_t *task);	/* WRONG: 用 typedef 名 */
```

---

## 9. 规则 OS-KER-014：零警告门禁

### 9.1 规则文本

agentrt-liunx 内核构建**必须**零警告通过。`Makefile` 必须启用 `-Wall -Werror`，`scripts/checkpatch.pl` 必须以 0 ERROR / 0 WARNING 通过。CI 门禁中任何警告即视为构建失败，禁止以注释抑制（`// NOLINT` 等）掩盖警告。`__deprecated` 属性不再产生构建警告，因此废弃接口必须从代码中彻底移除或登记到 `100-deprecated-api-registry.md`。

### 9.2 OLK-6.6 源码路径

- `Documentation/process/deprecated.rst:20-30` —— "__deprecated ... does not produce warnings during builds any more ... the standing goals of the kernel is to build without warnings"
- `scripts/checkpatch.pl:2444-2447` —— `ERROR` 与 `WARNING` 计数逻辑（非零即失败）
- `scripts/checkpatch.pl:2469-2482` —— `ERROR()` / `WARN()` 报告函数
- `Makefile`（顶层）—— `-Wall -Wundef` 等警告开关基线

### 9.3 正确示例

```c
/* 修复未使用变量：要么使用，要么删除，不要用 // NOLINT 掩盖 */
static int agentrt_sched_idle(struct agentrt_sched *sched)
{
	(void)sched;				/* WRONG 思路：应删除该参数 */
	return 0;
}

/* 正确做法：移除未使用参数，或用 __maybe_unused 标注条件编译场景 */
static int __maybe_unused agentrt_sched_debug_dump(struct agentrt_sched *sched)
{
	pr_info("sched: %s tasks=%u\n", sched->name, sched->nr_tasks);
	return 0;
}
```

### 9.4 错误示例

```c
static int agentrt_sched_idle(struct agentrt_sched *sched)
{
	// NOLINT: 未使用参数 sched		/* WRONG: 用注释抑制警告 */
	return 0;
}
```

---

## 10. 规则 OS-STD-CODE-010：strscpy 强制替代 strcpy/strncpy/strlcpy

### 10.1 规则文本

agentrt-liunx 内核态**禁止**新增 `strcpy`、`strncpy`、`strlcpy` 调用。所有以 NUL 结尾的字符串复制**必须**使用 `strscpy`；需要 NUL 填充的使用 `strscpy_pad`；非 NUL 结尾的使用 `strtomem`/`strtomem_pad`。`strscpy` 返回已复制的非 NUL 字节数（截断时返回负 errno），调用方必须据此处理截断。

### 10.2 OLK-6.6 源码路径

- `Documentation/process/deprecated.rst:122-132` —— "strcpy() ... The safe replacement is strscpy()"
- `Documentation/process/deprecated.rst:134-154` —— "strncpy() on NUL-terminated strings ... the replacement is strscpy()"
- `Documentation/process/deprecated.rst:156-163` —— "strlcpy() ... The safe replacement is strscpy()"
- `scripts/checkpatch.pl:7029-7031` —— `WARN("STRCPY")` 规则
- `scripts/checkpatch.pl:7035-7037` —— `WARN("STRLCPY")` 规则
- `scripts/checkpatch.pl:7041-7043` —— `WARN("STRNCPY")` 规则

### 10.3 正确示例

```c
static int agentrt_chan_set_name(struct agentrt_ipc_chan *chan,
				 const char *user_name)
{
	ssize_t copied;

	copied = strscpy(chan->name, user_name, sizeof(chan->name));
	if (copied < 0) {
		pr_warn("agentrt-ipc: name truncated for channel %d\n",
			chan->id);
		return -E2BIG;
	}
	return 0;
}
```

### 10.4 错误示例

```c
static int agentrt_chan_set_name(struct agentrt_ipc_chan *chan,
				 const char *user_name)
{
	strcpy(chan->name, user_name);		/* WRONG: 无边界检查 */
	return 0;
}

static int agentrt_chan_set_name2(struct agentrt_ipc_chan *chan,
				 const char *user_name)
{
	strncpy(chan->name, user_name, sizeof(chan->name));	/* WRONG: 不保证 NUL 终止 */
	strlcpy(chan->name, user_name, sizeof(chan->name));	/* WRONG: 多余读源 */
	return 0;
}
```

---

## 11. 规则 OS-KER-015：禁止 BUG()/BUG_ON()，改用 WARN()

### 11.1 规则文本

agentrt-liunx 内核态**禁止**新增 `BUG()`、`BUG_ON()`、`VM_BUG_ON()`。必须改用 `WARN_ON_ONCE()`（首选）或 `WARN()`，并尽可能提供恢复代码。"我懒得做错误处理"不是使用 `BUG()` 的借口。`WARN*()` 仅用于"预期不可达"场景；"可达但 undesirable"场景使用 `pr_warn()`。`BUILD_BUG_ON()` 是编译期断言，无运行时影响，允许且鼓励使用。

### 11.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst:1189-1249` —— §22 "Do not crash the kernel"
- `Documentation/process/coding-style.rst:1202-1212` —— "Use WARN() rather than BUG() ... Do not add new code that uses any of the BUG() variants"
- `Documentation/process/coding-style.rst:1214-1221` —— "Use WARN_ON_ONCE() rather than WARN() or WARN_ON()"
- `Documentation/process/coding-style.rst:1245-1249` —— "BUILD_BUG_ON() is acceptable and encouraged"
- `Documentation/process/deprecated.rst:32-52` —— "BUG() and BUG_ON() ... Use WARN() and WARN_ON() instead"

### 11.3 正确示例

```c
static int agentrt_ipc_validate_hdr(const struct agentrt_ipc_msg_hdr *hdr)
{
	if (hdr->magic != AIRYMAX_IPC_MAGIC) {
		WARN_ON_ONCE(1);		/* 预期不可达：日志 + 继续 */
		return -EINVAL;
	}
	if (hdr->payload_len > AIRYMAX_IPC_PAYLOAD_MAX) {
		pr_warn("agentrt-ipc: payload %u exceeds max %u\n",
			hdr->payload_len, AIRYMAX_IPC_PAYLOAD_MAX);
		return -E2BIG;
	}
	return 0;
}

/* 编译期断言：允许且鼓励 */
BUILD_BUG_ON(AIRYMAX_IPC_HDR_SZ != 128);
```

### 11.4 错误示例

```c
static int agentrt_ipc_validate_hdr(const struct agentrt_ipc_msg_hdr *hdr)
{
	BUG_ON(hdr->magic != AIRYMAX_IPC_MAGIC);	/* WRONG: 内核崩溃 */
	BUG_ON(hdr->payload_len > AIRYMAX_IPC_PAYLOAD_MAX);	/* WRONG */
	return 0;
}
```

---

## 12. 规则 OS-STD-CODE-012：sizeof(*p) 替代 sizeof(struct xxx)

### 12.1 规则文本

向内存分配器传递结构体大小时**必须**使用 `sizeof(*p)` 形式（p 为目标指针变量），禁止 `sizeof(struct xxx)`。理由：(1) 当指针变量类型变更时，`sizeof(*p)` 自动跟随，避免"改了类型忘改 sizeof"的隐蔽 bug；(2) 可读性更佳。数组分配使用 `kmalloc_array(n, sizeof(*p), ...)` 或 `kcalloc(n, sizeof(*p), ...)`，禁止 `kmalloc(n * sizeof(...), ...)` 形式（无溢出检查）。`void *` 返回值**禁止**强制转换（C 语言自动转换）。

### 12.2 OLK-6.6 源码路径

- `Documentation/process/coding-style.rst:925-933` —— "The preferred form for passing a size of a struct is: p = kmalloc(sizeof(*p), ...);"
- `Documentation/process/coding-style.rst:935-937` —— "Casting the return value which is a void pointer is redundant"
- `Documentation/process/coding-style.rst:939-952` —— "The preferred form for allocating an array is: kmalloc_array(n, sizeof(...), ...); kcalloc(n, sizeof(...), ...);"
- `scripts/checkpatch.pl:7229` —— `ALLOC_SIZEOF_STRUCT` 规则检查
- `scripts/checkpatch.pl:7255` —— `ALLOC_WITH_MULTIPLY` 规则检查（`kmalloc(n * sizeof(...))` 反模式）
- `Documentation/process/deprecated.rst:54-110` —— "open-coded arithmetic in allocator arguments"

### 12.3 正确示例

```c
struct agentrt_task *task;

task = kzalloc(sizeof(*task), GFP_KERNEL);		/* sizeof(*p) 形式 */
if (!task)
		return -ENOMEM;

task->cmds = kcalloc(cmd_count, sizeof(*task->cmds), GFP_KERNEL);	/* 数组 + 溢出检查 */
if (!task->cmds) {
	kfree(task);
	return -ENOMEM;
}
```

### 12.4 错误示例

```c
struct agentrt_task *task;

task = (struct agentrt_task *)kmalloc(sizeof(struct agentrt_task), GFP_KERNEL);	/* WRONG: sizeof(struct) + 多余 cast */
if (!task)
	return -ENOMEM;

task->cmds = kmalloc(cmd_count * sizeof(struct agentrt_cmd), GFP_KERNEL);	/* WRONG: 无溢出检查 */
```

---

## 13. 强化规则速查表

| 编号 | 规则 | OLK-6.6 主源码路径 | 严重度 |
|------|------|-------------------|--------|
| OS-KER-011 | Tab-8 缩进强制 | `coding-style.rst:21` / `.clang-format:645,687,688` | ERROR |
| OS-KER-009 | goto 集中错误处理 | `coding-style.rst:526-595` | ERROR |
| OS-KER-008 | bool 仅用于返回值与栈变量 | `coding-style.rst:1021-1049` | WARN |
| OS-KER-007 | 内核态禁 float，强制 airymax_q16_t | `arch/x86/Makefile:137-138` | ERROR |
| OS-KER-012 | 行宽 80 列硬上限 | `coding-style.rst:104` / `.clang-format:55` | WARN |
| OS-STD-CODE-014 | 指定初始化器强制 | `coding-style.rst`（贯穿） / `checkpatch.pl:4662` | ERROR |
| OS-STD-CODE-015 | fallthrough; 显式穿透 | `deprecated.rst:200-235` / `coding-style.rst:59` | WARN |
| OS-KER-013 | 禁止结构体 typedef | `coding-style.rst:359-440` / `checkpatch.pl:4788` | WARN |
| OS-KER-014 | 零警告门禁 | `deprecated.rst:20-30` / `checkpatch.pl:2444-2482` | ERROR |
| OS-STD-CODE-010 | strscpy 强制替代 | `deprecated.rst:122-163` / `checkpatch.pl:7029-7043` | WARN |
| OS-KER-015 | 禁止 BUG()/BUG_ON() | `coding-style.rst:1189-1249` / `deprecated.rst:32-52` | ERROR |
| OS-STD-CODE-012 | sizeof(*p) 替代 sizeof(struct) | `coding-style.rst:925-952` / `checkpatch.pl:7229,7255` | WARN |

---

## 14. 与基线文档及工具链的衔接

### 14.1 与 `C_coding_style_standard.md` 的关系

本文件是基线的"强化补充"，两者关系如下：

| 维度 | 基线文档 | 本补充文档 |
|------|---------|-----------|
| 覆盖范围 | 通用编码风格（命名、函数、注释、头文件等） | 12 条高违规率强化规则 |
| 规则密度 | 章节式叙述 | 单规则深度（正确 / 错误成对示例） |
| 源码引用 | 章节级 | 行号级（`文件:行号`） |
| 工具链 | 提及 checkpatch | 与 checkpatch 规则编号交叉引用 |

### 14.2 与 checkpatch 规则映射的关系

本文件中每条规则在 `60-checkpatch-rule-map.md` 中均有对应的 checkpatch 规则编号映射（如 OS-STD-CODE-010 对应 `STRCPY`/`STRLCPY`/`STRNCPY`），审查者可双向查阅。

### 14.3 与 .clang-format 强制的关系

规则 OS-KER-011（Tab-8）、OS-KER-012（行宽 80）由 `.clang-format` 自动强制（详见 `80-clang-format-enforcement.md`）。`make format-check` 在 CI 中验证所有 C 文件符合 `.clang-format`，违反即构建失败。

---

## 15. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，覆盖 12 条强化规则 | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
