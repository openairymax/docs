Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）代码规范

> **文档定位**: agentrt-linux（AirymaxOS，极境智能体操作系统）工程标准规范 / 代码规范
> **版本**: 0.1.1 ｜ **最后更新**: 2026-07-06
> **同源映射**: `50-engineering-standards/README.md` §3；`ARCHITECTURAL_PRINCIPLES.md` 五维正交 24 原则
> **理论根基**: Linux 6.6 内核基线工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）
> **适用范围**: agentrt-linux 内核态（C / 内联汇编）与同源用户态组件（C / Rust / Python / TypeScript）

---

## 0. 文档说明

本文档是 agentrt-linux 工程标准规范第一份子文档，定义**语义层**代码规则。它不是 Linux `coding-style.rst` 的译本，而是基于 Linux 6.6 内核基线沉淀的工程思想，融合 Airymax 五维正交 24 原则后，针对智能体操作系统场景重新表述的工程契约。代码格式（缩进 / 行宽 / clang-format）见 `02-code-format.md`，代码风格（模块化 / 抽象层次）见 `03-code-style.md`。

**权威声明**：本卷是 agentrt-linux **语义层代码规则**（命名/函数/注释/类型/错误处理/内存/锁）的唯一权威来源（SSoT）。`10-coding-style/C_coding_style_standard.md` 中的命名约定（§3）、函数定义（§4）、错误处理（§7）、类型（§6）、内存管理（§8）、锁与并发（§9）均引用本卷，不得重新定义。若发现不一致，以本卷为准。

**规则编号**：每条强制规则赋予唯一编号。`OS-KER-xxx` 为内核工程规则（强制，agentrt 不涉及内核态）；`OS-STD-xxx` 为标准规则；`OS-BAN-xxx` 为禁止规则；`OS-ACC-xxx` 为验收标准。多语言对照追求语义等价而非逐字翻译——同一规则在不同语言中以该语言最自然的方式落地。注册表汇总于 `07-maintainers-and-governance.md`。

---

## 1. 命名规范

C 是朴素的语言，命名应与之相称。agentrt-linux 命名遵循两条核心准则：**全局可见者必语义完备，局部可见者宜短小精悍**。

### 1.1 全局变量与全局函数

> **OS-STD-CODE-001**: 全局符号必须描述性命名，禁止无意义缩写。

```c
/* 好 */
int count_active_users(void);
struct list_head *agentrt_sched_ready_queue(void);

/* 坏 */
int cntusr(void);
struct list_head *q(void);
```

### 1.2 局部变量

> **OS-STD-002**: 局部变量短小切题，循环计数器用 `i`，临时值用 `tmp`，缓冲区用 `buf`。需要长名才能区分说明函数职责过重，应拆分。

```c
int i;                /* 循环计数器 */
char *tmp;            /* 临时指针 */
u8 buf[256];          /* 缓冲区 */
struct agentrt_task *task;
```

### 1.3 敏感术语禁用

> **OS-BAN-001**: 新增代码禁用 `master/slave`、`blacklist/whitelist`，替换为 `primary/secondary`、`denylist/allowlist`。仅当维护既有 UAPI 或对齐既有硬件 / 协议规范时例外。

```c
/* 好 */                          /* 坏 */
struct agentrt_primary_node *p;   struct agentrt_master *m;
bool denylist_match(name);        bool blacklist_match(name);
```

### 1.4 宏与枚举命名

> **OS-STD-003**: 常量宏全大写；多个相关常量优先用 `enum` 而非散落 `#define`。

```c
#define AGENTRT_MAX_TASKS     1024
enum agentrt_task_state {
	AGENTRT_TASK_PENDING   = 0,
	AGENTRT_TASK_RUNNING   = 1,
	AGENTRT_TASK_COMPLETED = 2,
	AGENTRT_TASK_FAILED    = 3,
};
```

### 1.5 函数返回值与命名约定

> **OS-STD-004**: 动作式 / 命令式函数返回错误码（0 = 成功，-Exxx = 失败）；谓词式函数返回 `bool`。混合两种约定是难调 Bug 的温床。

```c
int  agentrt_task_submit(struct agentrt_task *task);   /* 0 / -EBUSY */
bool agentrt_task_is_pending(const struct agentrt_task *task);
```
```rust
// Rust 等价：动作式返回 Result<(), ErrCode>，谓词式返回 bool
fn agentrt_task_submit(t: &mut Task) -> Result<(), ErrCode>;
fn agentrt_task_is_pending(t: &Task) -> bool;
```

### 1.6 agentrt-linux 专属前缀

> **OS-STD-005**: `agentrt_*` 前缀保留给 agentrt 同源 API；`airymaxos_*` 前缀用于 agentrt-linux 内核 / 发行版专属 API。两者共享 Airymax 同源语义（MicroCoreRT / AgentsIPC / Cupolas / MemoryRovol / CoreLoopThree），但代码归属与 ABI 边界不同，前缀隔离确保无适配层互操作时不冲突。

```c
int agentrt_ipc_send(u32 ch, const void *msg, size_t len);     /* agentrt 同源 */
int airymaxos_lsm_hook_register(const struct security_hook_list *hooks); /* OS 专属 */
```

**五维正交映射**：E-5 命名语义化、A-1 极简主义、K-2 接口契约化。

---

## 2. 函数规范

### 2.1 函数长度

> **OS-STD-006**: 函数一屏可读（≤80×24），局部变量 ≤5–10 个。复杂度与长度成反比：200 行的简单 `switch` 调度器可接受，200 行胶水函数不可接受。局部变量超 10 个应拆分。

### 2.2 函数原型元素顺序

> **OS-STD-007**: 原型元素固定顺序：storage class → storage class attributes → return type → return type attributes → name → parameters → parameter attributes → behavior attributes。

```c
/* 声明：参数属性在参数表后 */
__init void * __must_check
agentrt_ipc_channel_create(enum agentrt_ipc_type type, size_t cap,
			  u32 flags) __printf(2, 3) __malloc;

/* 定义：参数属性移到 storage class attributes 之后 */
static __always_inline __init __printf(2, 3) void * __must_check
agentrt_ipc_channel_create(enum agentrt_ipc_type type, size_t cap,
			  u32 flags) __malloc { /* ... */ }
```

### 2.3 参数命名

> **OS-STD-008**: 原型必须含参数名——它是面向读者的廉价文档。

```c
int agentrt_task_submit(struct agentrt_task *task);  /* 好 */
int agentrt_task_submit(struct agentrt_task *);       /* 坏 */
```

### 2.4 禁止 extern 修饰函数声明

> **OS-BAN-002**: 函数声明禁止 `extern`，使行变长且无额外语义。

### 2.5 EXPORT_SYMBOL 紧跟函数定义

> **OS-STD-009**: 导出符号的 `EXPORT_SYMBOL*` 宏必须紧跟右大括号行，中间不留空行。

```c
int airymaxos_perm_granted(const struct cred *c, u32 action) { /* ... */ }
EXPORT_SYMBOL_GPL(airymaxos_perm_granted);
```

**五维正交映射**：K-2 接口契约化、E-7 文档即代码、A-1 极简主义。

---

## 3. 注释规范

### 3.1 注释原则

> **OS-STD-010**: 注释说明 WHAT 与 WHY，不说明 HOW。好代码自解释 HOW。函数体内尽量避免注释；若需分段注释应拆分为多个函数。函数头说明做什么、为什么这样做。

### 3.2 多行注释风格

> **OS-STD-011**: 通用代码用通用风格；`net/` 与 `drivers/net/` 用 net 风格。

```c
/*
 * 通用风格：左列星号，首尾近空白行。
 */
/* net/ 与 drivers/net/ 风格：首行无近空白行。
 *
 * 与上游一致便于回填。
 */
```

### 3.3 kernel-doc 格式

> **OS-STD-012**: 所有公共 API（`EXPORT_SYMBOL*` 函数、公共结构体、公共宏）必须用 kernel-doc 注释。

```c
/**
 * agentrt_ipc_send() - 在指定通道上发送消息
 * @channel: 通道 ID，由 agentrt_ipc_channel_create() 返回
 * @msg:    消息体指针，长度不超过 AGENTRT_IPC_MSG_BODY_MAX
 * @len:    消息体字节数
 *
 * 阻塞发送，直到对端读取或超时。消息头由 AgentsIPC 128B 协议自动填充。
 *
 * Return: 0 成功；-EAGAIN 通道满；-EMSGSIZE 超长；-ETIMEDOUT 超时。
 */
int agentrt_ipc_send(u32 channel, const void *msg, size_t len);
```

### 3.4 数据注释

> **OS-STD-013**: 数据声明每行一个，留出注释空间。

```c
struct agentrt_task {
	u32    id;            /* 全局唯一任务 ID */
	u32    parent_id;     /* 父任务 ID，根任务为 0 */
	u8     priority;      /* 0=最低，255=最高 */
	u8     state;         /* enum agentrt_task_state */
	u16    flags;         /* AGENTRT_TASK_FLAG_* 位掩码 */
};
```

### 3.5 agentrt-linux 多语言注释规范

> **OS-STD-014**: C 用 kernel-doc；Rust 用 rustdoc；Python 用 Google docstring；TypeScript 用 JSDoc。

```rust
/// 在指定通道上发送消息（阻塞）。
/// # 参数
/// * `channel` - 通道 ID
/// # 返回
/// `Ok(())` 成功；`Err(ErrCode::Again|MsgSize|TimedOut)` 失败。
pub fn agentrt_ipc_send(channel: u32, msg: &[u8]) -> Result<(), ErrCode>;
```
```python
def agentrt_ipc_send(channel: int, msg: bytes) -> None:
    """在指定通道上发送消息（阻塞）。
    Args:
        channel: 通道 ID。
        msg:     消息体。
    Raises:
        AgentrtError: EAGAIN / EMSGSIZE / ETIMEDOUT。
    """
```

**五维正交映射**：E-7 文档即代码、K-2 接口契约化、A-3 人文关怀。

---

## 4. 头文件与 import 规范

### 4.1 显式声明原则

> **OS-STD-015**: 每个 `.c` 必须显式 `#include` 其直接依赖的头文件，不得依赖间接传递包含。传递依赖会在头文件重构时引发雪崩式编译错误。

```c
/* 好 */                       /* 坏：依赖 task.h 间接拉入 slab.h */
#include <linux/slab.h>        #include <agentrt/task.h>
#include <agentrt/ipc.h>        kfree(p);
```

### 4.2 include 顺序

> **OS-STD-016**: include 顺序固定：系统头 → 架构头 → 本地头 → `#define CREATE_TRACE_POINTS` → trace events 头。

```c
#include <linux/module.h>
#include <linux/slab.h>
#include <asm/barrier.h>
#include "agentrt_internal.h"
#define CREATE_TRACE_POINTS
#include <trace/events/agentrt.h>
```

### 4.3 禁止自动排序

> **OS-BAN-003**: 禁止 IDE / 工具自动排序 include。`.clang-format` 必须设 `SortIncludes: false`。include 顺序承载语义分组，自动排序会破坏分组并引入隐式依赖。

### 4.4 include guard 规范

> **OS-STD-017**: 头文件用 `#ifndef` / `#define` / `#endif` 保护，guard 名与文件路径对应。Rust / TS 模块系统天然隔离，无需 guard。

```c
#ifndef _AGENTRT_IPC_H
#define _AGENTRT_IPC_H
/* ... */
#endif /* _AGENTRT_IPC_H */
```

**五维正交映射**：K-2 接口契约化、E-1 安全内生、A-1 极简主义。

---

## 5. 宏使用规范

### 5.1 五类应避免的宏

> **OS-BAN-004**: 禁止以下五类宏：(1) 影响控制流（宏内 `return`）；(2) 依赖魔法命名的局部变量；(3) 作为左值使用的宏参数；(4) 忽略运算优先级的宏常量；(5) 函数式宏中局部变量命名空间冲突。

```c
/* 坏 1 */  #define FOO(x) do { if (blah(x) < 0) return -EINVAL; } while (0)
/* 坏 2 */  #define FOO(val) bar(index, val)
/* 坏 3 */  #define FOO(x) (x)->field
/* 坏 4 */  #define CONSTEXP CONSTANT | 3          /* 应为 (CONSTANT | 3) */
/* 坏 5 */  #define FOO(x) ({ typeof(x) ret; ret = calc(x); (ret); })
```

### 5.2 多语句宏必须用 do-while(0)

> **OS-STD-018**: 多语句宏必须用 `do { ... } while (0)` 包裹，使其在 `if` 单语句体中行为正确。

```c
#define agentrt_assert_locked(l) do { lockdep_assert_held(&(l)->lock); } while (0)
```

### 5.3 inline functions 优于宏

> **OS-STD-019**: 优先用 `static inline` 函数替代函数式宏——有类型检查、不重复求值、不陷阱于优先级。Rust 用 `const fn` 完全替代。

```c
static inline u32 agentrt_task_priority_max(void) { return AGENTRT_MAX_PRIO; }
```

**五维正交映射**：E-1 安全内生、A-2 细节关注、K-2 接口契约化。

---

## 6. 类型规范

### 6.1 typedef 严格限制

> **OS-STD-020**: typedef 仅在以下 5 种例外下使用，其余禁止：(1) 完全不透明对象（如 `pte_t`，仅靠 accessor 访问）；(2) 清晰整数类型（避免 `int` / `long` 歧义，如 `u8/u32`）；(3) sparse 创建的新类型；(4) 与 C99 标准类型等价；(5) 用户空间可见的安全类型（`__u32`）。

```c
struct agentrt_task *task;     /* 好：结构体直接 struct */
typedef struct agentrt_task *task_ptr_t;  /* 坏：掩盖类型 */
```

### 6.2 内核原生类型

> **OS-KER-002**: 内核内部用 `u8/u16/u32/u64` 及有符号变体；用户空间可见的 UAPI 结构体用 `__u8/__u16/__u32/__u64`。

```c
struct agentrt_task {        u32 id; u64 deadline_ns; };          /* 内核内部 */
struct agentrt_ipc_msg_hdr { __u32 magic; __u32 chan; __u64 seq; }; /* UAPI */
```

### 6.3 bool 类型使用边界

> **OS-KER-002**: `bool` 仅用于函数返回值与栈变量；对齐敏感的结构体成员、cache line 布局敏感场景不用 `bool`——其大小与对齐随架构变化。

```c
bool agentrt_task_is_pending(const struct agentrt_task *t);  /* 好：返回值 */
struct stats { bool fast_path; u32 hits; };                  /* 坏：破坏 cache line */
```

### 6.4 位域与紧凑布局

> **OS-STD-021**: 多个布尔标志位应合并为 1 位 bitfield 或单个 `u8` flags，避免逐个 `bool`。

```c
struct agentrt_task {
	u32 id, parent_id;
	u8  priority, state;
	u8  flags;        /* AGENTRT_TASK_FLAG_* 位掩码 */
	u8  reserved;
	u64 deadline_ns;
} __attribute__((aligned(64)));
```
```rust
// Rust 等价：bitflags! 宏
bitflags::bitflags! { pub struct TaskFlags: u8 {
    const CANCELABLE = 1 << 0; const RETRYABLE = 1 << 1; const AUDITED = 1 << 2;
}}
```

**五维正交映射**：E-3 资源确定性、K-2 接口契约化、A-2 细节关注。

---

## 7. 错误处理规范（强制范式）

错误处理是 agentrt-linux 工程观的核心。本章所有规则均为强制。

### 7.1 goto 集中出口模式

> **OS-KER-003**: 函数多出口且需公共清理时，必须用 `goto` 跳转到集中出口标签。标签名应描述其行为（如 `out_free_buffer:`），禁止 `err1:` / `err2:` 编号式命名。

```c
int agentrt_session_open(struct agentrt_session **out, const char *name)
{
	struct agentrt_session *s;
	void *buf;
	int ret;

	s = kzalloc(sizeof(*s), GFP_KERNEL);
	if (!s) return -ENOMEM;

	buf = kmalloc(PAGE_SIZE, GFP_KERNEL);
	if (!buf) { ret = -ENOMEM; goto out_free_session; }

	ret = agentrt_ipc_channel_create(name, &s->chan);
	if (ret) goto out_free_buf;

	*out = s;
	return 0;

out_free_buf:
	kfree(buf);
out_free_session:
	kfree(s);
	return ret;
}
```

### 7.2 分级标签按分配逆序释放

> **OS-KER-004**: 多个出口标签必须按资源分配的逆序释放，每个标签只释放其对应资源，防止 one-err bug（某条路径上指针未初始化却被释放）。

```c
/* 坏：foo 可能为 NULL 时仍 kfree(foo->bar) */   /* 好：分级标签 */
err:                                              err_free_bar:
	kfree(foo->bar);                                kfree(foo->bar);
	kfree(foo);                                     err_free_foo:
	return ret;                                      kfree(foo);
                                                   return ret;
```

理想情况下应通过错误注入测试覆盖所有出口路径。

### 7.3 禁止 BUG()/BUG_ON()

> **OS-BAN-005**: 禁止新增 `BUG()` / `BUG_ON()` / `VM_BUG_ON()`，改用 `WARN_ON_ONCE()` + 恢复代码。`BUG()` 让系统在未释放锁、未恢复状态下崩溃，使事后调试不可能。仅当重大内部损坏且无任何继续运行可能时例外，且需充分论证。`WARN*()` 仅用于"不应到达"场景；"可达但不期望"用 `pr_warn_once()`。`BUILD_BUG_ON()` 是编译期断言，鼓励使用。

```c
if (WARN_ON_ONCE(!mutex_is_locked(&p->lock)))   /* 好 */
		return -EINVAL;
BUG_ON(!mutex_is_locked(&p->lock));             /* 坏 */
```

### 7.4 内存分配惯用法

> **OS-KER-005**: 传递结构体大小用 `sizeof(*p)`；数组分配用 `kmalloc_array` / `kcalloc`（带溢出检查）；带尾数组用 `struct_size` / `flex_array_size`。禁止在分配器参数中手写乘法（溢出风险）。

```c
struct agentrt_task *task = kmalloc(sizeof(*task), GFP_KERNEL);   /* 好 */
struct agentrt_task *arr  = kcalloc(n, sizeof(*arr), GFP_KERNEL);  /* 好：溢出检查 */
struct agentrt_batch *b = kzalloc(struct_size(b, items, n), GFP_KERNEL); /* 好：柔性数组 */
arr = kmalloc(n * sizeof(*arr), GFP_KERNEL);                      /* 坏：可能溢出 */
```

### 7.5 字符串复制安全

> **OS-BAN-006**: 禁止 `strcpy` / `strncpy` / `strlcpy`。强制 `strscpy`（需 NUL 终止）或 `strscpy_pad`（需 NUL 填充）。`strcpy` 无边界检查；`strncpy` 不保证终止；`strlcpy` 读全源串可能越界。

```c
strscpy(dst->name, src->name, sizeof(dst->name));
```

### 7.6 禁用 %p 默认输出

> **OS-BAN-007**: 禁止新增 `%p` 默认输出——所有 `%p` 自动哈希化，对寻址无用。需要真实指针须在注释与 commit log 充分论证后用 `%px` 并确保权限控制；文本地址用 `%pS` 输出符号名。

```c
pr_info("task %u state %s\n", task->id, state_name);   /* 好 */
pr_info("handler at %pS\n", task->handler);           /* 好 */
pr_info("task at %p\n", task);                          /* 坏：已哈希，无意义 */
```

### 7.7 agentrt-linux 多语言错误处理

> **OS-STD-022**: C 用 goto 集中出口；Rust 用 `?` + `Result` + RAII；Python 用异常 + 上下文管理器；TypeScript 用 try-finally。

```rust
fn agentrt_session_open(name: &str) -> Result<Session, ErrCode> {
    let s = Session::new()?;            /* 失败自动 return */
    let buf = Box::<[u8; 4096]>::new_uninit_slice()?;
    let chan = agentrt_ipc_channel_create(name)?;
    Ok(Session { s, buf, chan })        /* RAII 自动清理 */
}
```
```python
def agentrt_session_open(name: str) -> Session:
    with Session.alloc() as s, Buffer.alloc(PAGE_SIZE) as buf:
        s.bind_channel(agentrt_ipc_channel_create(name))
        return s.finalize()            # with 块退出自动释放
```

**五维正交映射**：E-1 安全内生（安全字符串 / 溢出检查）、E-3 资源确定性（goto 集中出口保证释放）、E-6 错误可追溯（分级标签即错误路径文档）、K-2 接口契约化（错误码即契约）。

---

## 8. 并发与 volatile 规范

### 8.1 volatile 几乎从不正确

> **OS-KER-006**: `volatile` 仅在以下 4 种罕见情况下使用，其余禁止：(1) I/O 内存访问器内部（架构相关）；(2) 防止 GCC 删除无可见副作用的内联汇编；(3) `jiffies` 等"每次读取值不同但无需锁"的遗留变量；(4) 一致性内存中可能被 I/O 设备修改的环缓冲指针。`volatile` 抑制优化而非保护并发——锁原语已包含必要的编译器屏障。

```c
volatile int shared_counter;        /* 坏：volatile 不能替代锁 */
spinlock_t lock; int counter;       /* 好：spinlock 保护，屏障由锁提供 */
```

### 8.2 共享数据必须用同步原语保护

> **OS-KER-053**: 共享数据必须通过 spinlock / mutex / memory barrier / RCU 保护。锁保持数据一致性，引用计数管理生命周期，两者不可互替。

```c
struct agentrt_task_table { spinlock_t lock; struct list_head tasks; };

void agentrt_task_table_add(struct agentrt_task_table *t, struct agentrt_task *task)
{
	spin_lock(&t->lock);
	list_add_tail(&task->link, &t->tasks);
	spin_unlock(&t->lock);
}
```

### 8.3 I/O 内存必须通过 accessor 访问

> **OS-KER-008**: I/O 内存禁止通过裸指针解引用访问，必须通过 `readl` / `writel` / `ioread32` / `iowrite32` 等 accessor。accessor 已封装平台差异与编译器屏障。

```c
writel(AGENTRT_REG_ENABLE, base + AGENTRT_REG_CTRL);   /* 好 */
volatile u32 *ctrl = (volatile u32 *)(base + AGENTRT_REG_CTRL); /* 坏 */
*ctrl = AGENTRT_REG_ENABLE;
```

### 8.4 忙等用 cpu_relax()

> **OS-KER-009**: 忙等循环必须调用 `cpu_relax()`，降低功耗并提示超线程孪生核心，同时作为编译器屏障。

```c
while (readl(base + AGENTRT_REG_STATUS) & AGENTRT_STATUS_BUSY)
	cpu_relax();
```

### 8.5 Rust Send/Sync 强制要求

> **OS-STD-023**: Rust 跨线程传递的类型必须实现 `Send`；跨线程共享的类型必须实现 `Sync`。agentrt-linux Rust 同源组件禁止 `unsafe impl Send/Sync`，除非有书面论证与 unsafe audit。

```rust
struct AgentrtChannel { inner: Mutex<ChannelInner> }  /* Mutex<T> 自动提供 Sync */
```

**五维正交映射**：E-1 安全内生、E-3 资源确定性、K-2 接口契约化、A-2 细节关注。

---

## 9. 条件编译与 magic number

### 9.1 尽可能不在 .c 中用 #if/#ifdef

> **OS-STD-024**: `.c` 中尽量避免 `#if` / `#ifdef`。条件逻辑应在头文件中提供 stub 函数，`.c` 无条件调用。整函数编译裁剪优于函数内片段裁剪。若函数 / 变量在某配置下未使用，用 `__maybe_unused` 标注而非 `#ifdef` 包裹。

```c
/* 头文件：stub */
#ifdef CONFIG_AGENTRT_AUDIT
static inline void agentrt_audit_record(u32 e) { /* real */ }
#else
static inline void agentrt_audit_record(u32 e) { /* no-op */ }
#endif
/* .c：无条件调用 */
void agentrt_task_submit(struct agentrt_task *t) {
	agentrt_audit_record(EVT_SUBMIT);
	__agentrt_task_enqueue(t);
}
```

### 9.2 用 IS_ENABLED 转 C 布尔表达式

> **OS-STD-025**: 在 C 表达式中用 `IS_ENABLED(CONFIG_XXX)` 而非 `#ifdef`，让编译器仍对内部代码做语法 / 类型检查。

```c
if (IS_ENABLED(CONFIG_AGENTRT_AUDIT)) { agentrt_audit_enable(); }  /* 好 */
#ifdef CONFIG_AGENTRT_AUDIT agentrt_audit_enable(); #endif         /* 坏：内部不检查 */
```

### 9.3 #endif 同行必须注释条件

> **OS-STD-026**: 非平凡 `#if` / `#ifdef` 块（>数行）的 `#endif` 同行必须注释其条件。

```c
#ifdef CONFIG_AGENTRT_AUDIT
	/* ... 数十行 ... */
#endif /* CONFIG_AGENTRT_AUDIT */
```

### 9.4 magic number 必须注册

> **OS-KER-010**: 受保护的数据结构应在结构体开头声明 `magic` 字段，并将该 magic number 注册到 agentrt-linux 维护的 `magic-numbers.md`。magic number 用于运行时检测结构体被覆盖或类型误传，在数组越界覆盖相邻结构的场景中尤其有效。

```c
#define AGENTRT_SESSION_MAGIC  0x41475331  /* 'AGS1' */
struct agentrt_session { u32 magic; u32 id; /* ... */ };
static inline bool agentrt_session_valid(const struct agentrt_session *s) {
	return s && s->magic == AGENTRT_SESSION_MAGIC;
}
```

### 9.5 Kconfig 配置文件规范

> **OS-STD-027**: Kconfig 用 tab 缩进；help 文本额外缩进两空格；危险特性必须在 prompt 字符串中标注 `(DANGEROUS)`。

```
config AGENTRT_AUDIT
	bool "Agentrt audit subsystem"
	depends on NET
	help
	  Enable the Agentrt audit subsystem that records all permission
	  decisions into an append-only hash chain.

config AGENTRT_DEVMEM_RW
	bool "Direct /dev/mem write access (DANGEROUS)"
	depends on MMU
```

**五维正交映射**：E-1 安全内生（DANGEROUS 标注）、E-7 文档即代码（Kconfig 即配置文档）、K-2 接口契约化（stub 即契约裁剪）、A-3 人文关怀（#endif 注释尊重后续读者）。

---

## 10. 五维正交原则总映射表

| 章节 | K 内核观 | E 工程观 | A 设计美学 |
|------|---------|---------|-----------|
| 1 命名 | K-2 接口契约 | E-5 命名语义化 | A-1 极简主义 |
| 2 函数 | K-2 接口契约 | E-7 文档即代码 | A-1 极简主义 |
| 3 注释 | K-2 接口契约 | E-7 文档即代码 | A-3 人文关怀 |
| 4 头文件 | K-2 接口契约 | E-1 安全内生 | A-1 极简主义 |
| 5 宏 | K-2 接口契约 | E-1 安全内生 | A-2 细节关注 |
| 6 类型 | K-2 接口契约 | E-3 资源确定性 | A-2 细节关注 |
| 7 错误处理 | K-2 接口契约 | E-1 / E-3 / E-6 错误可追溯 | — |
| 8 并发 | K-2 接口契约 | E-1 / E-3 资源确定性 | A-2 细节关注 |
| 9 条件编译 | K-2 接口契约 | E-1 / E-7 文档即代码 | A-3 人文关怀 |

> **同源语义注**: 本章规则在 agentrt 用户态运行时侧有等价表述（agentrt `IRON-1`~`IRON-10` 同源铁律），两端通过 AgentsIPC 128B 消息头协议实现无适配层互操作——这是 agentrt-linux 与 agentrt "同源且部分代码共享（IRON-9 v2）"关系在代码规范层的体现。

---

## 11. 本章规则编号注册表

| 编号 | 强制度 | 一句话规则 |
|------|--------|-----------|
| OS-STD-CODE-001 | 标准 | 全局符号描述性命名 |
| OS-STD-002 | 标准 | 局部变量短小精悍 |
| OS-BAN-001 | 禁止 | 敏感术语 master/slave、blacklist/whitelist |
| OS-STD-003 | 标准 | 宏常量全大写，相关常量优先 enum |
| OS-STD-004 | 标准 | 动作式返回错误码，谓词式返回 bool |
| OS-STD-005 | 标准 | `agentrt_*` 与 `airymaxos_*` 前缀隔离 |
| OS-STD-006 | 标准 | 函数 ≤80×24，局部变量 ≤5–10 |
| OS-STD-007 | 标准 | 函数原型元素固定顺序 |
| OS-STD-008 | 标准 | 原型必须含参数名 |
| OS-BAN-002 | 禁止 | 函数声明禁止 `extern` |
| OS-STD-009 | 标准 | EXPORT_SYMBOL 紧跟右大括号 |
| OS-STD-010 | 标准 | 注释写 WHAT/WHY 不写 HOW |
| OS-STD-011 | 标准 | 多行注释通用风格与 net 风格 |
| OS-STD-012 | 标准 | 公共 API 强制 kernel-doc |
| OS-STD-013 | 标准 | 数据声明每行一个 |
| OS-STD-014 | 标准 | 多语言注释规范 |
| OS-STD-015 | 标准 | 显式 include，禁止间接传递 |
| OS-STD-016 | 标准 | include 顺序固定 |
| OS-BAN-003 | 禁止 | 禁止自动排序 include |
| OS-STD-017 | 标准 | include guard 命名规范 |
| OS-BAN-004 | 禁止 | 五类危险宏 |
| OS-STD-018 | 标准 | 多语句宏 do-while(0) |
| OS-STD-019 | 标准 | inline 优于宏 |
| OS-STD-020 | 标准 | typedef 仅 5 种例外 |
| OS-KER-002 | 内核 | 内部 u8/u32，UAPI __u32 |
| OS-KER-008 | 内核 | bool 仅返回值与栈变量 |
| OS-STD-021 | 标准 | 多布尔值合并 bitfield / flags |
| OS-KER-003 | 内核 | goto 集中出口模式 |
| OS-KER-004 | 内核 | 分级标签按分配逆序释放 |
| OS-BAN-005 | 禁止 | BUG()/BUG_ON()，改 WARN_ON_ONCE |
| OS-KER-005 | 内核 | sizeof(*p)/kmalloc_array/struct_size |
| OS-BAN-006 | 禁止 | strcpy/strncpy/strlcpy，强制 strscpy |
| OS-BAN-007 | 禁止 | %p 默认输出 |
| OS-STD-022 | 标准 | 多语言错误处理范式 |
| OS-KER-006 | 内核 | volatile 仅 4 种例外 |
| OS-KER-053 | 内核 | 共享数据用同步原语保护 |
| OS-KER-008 | 内核 | I/O 内存用 accessor |
| OS-KER-009 | 内核 | 忙等用 cpu_relax |
| OS-STD-023 | 标准 | Rust Send/Sync 强制 |
| OS-STD-024 | 标准 | .c 中避免 #if，用 stub |
| OS-STD-025 | 标准 | IS_ENABLED 转 C 布尔 |
| OS-STD-026 | 标准 | #endif 同行注释条件 |
| OS-KER-010 | 内核 | magic number 注册 |
| OS-STD-027 | 标准 | Kconfig tab 缩进，DANGEROUS 标注 |

合计 44 条规则：13 条 `OS-KER` 内核规则、22 条 `OS-STD` 标准规则、7 条 `OS-BAN` 禁止规则、2 条多语言相关标准。完整编号注册表（含 IRON / ACC / DRV / BUILD 等）汇总于 `07-maintainers-and-governance.md`。

---

## 12. 相关文档

- `50-engineering-standards/README.md` — 工程标准主索引与总纲
- `50-engineering-standards/02-code-format.md` — 代码格式（缩进 / 行宽 / clang-format）
- `50-engineering-standards/03-code-style.md` — 代码风格（模块化 / 抽象层次 / 防御性）
- `50-engineering-standards/04-engineering-philosophy.md` — 工程思想（双层稳定性 / 策略机制分离）
- `50-engineering-standards/07-maintainers-and-governance.md` — 维护者制度与规则编号注册表
- `docs/AirymaxRT/ARCHITECTURAL_PRINCIPLES.md` — Airymax 五维正交 24 原则
- `50-engineering-standards/magic-numbers.md` — agentrt-linux magic number 注册表（计划中）
- `30-interfaces/01-syscall-spec.md` — 系统调用规范（UABI 边界）
