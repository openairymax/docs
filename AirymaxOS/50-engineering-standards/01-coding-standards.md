Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）代码规范
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）工程标准规范 / 代码规范\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-06\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：`50-engineering-standards/README.md` §3；`00-architectural-principles.md` 五维正交 24 原则\
> **理论根基**：Linux 6.6 内核基线工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）\
> **适用范围**：agentrt-linux 内核态（C / 内联汇编）与同源用户态组件（C / Rust / Python / TypeScript）\
> **SSoT 声明**：本卷为 agentrt-linux **语义层代码规则**（命名/函数/注释/类型/错误处理/内存/锁）的唯一权威来源（SSoT）。语义规则编号体系为 **OS-STD-CODE-NNN**（4 段前缀，CODE = Coding），覆盖命名、函数、注释、类型、错误处理、内存与锁的全部语义层规则。\
> **合并说明（2026-07-12）**：本卷已合并原 `02-code-format.md`（→ Part II）、`03-code-style.md`（→ Part III）、`60-checkpatch-rule-map.md`（→ Part IV）、`70-kernel-doc-standard.md`（→ Part V）、`80-clang-format-enforcement.md`（→ Part VI）。原文件已物理删除，所有引用须指向本卷对应 Part。

***

# Part I: 代码语义规范

## 0. 文档说明

本文档是 agentrt-linux 工程标准规范第一份子文档，定义**语义层**代码规则。它不是 Linux `coding-style.rst` 的译本，而是基于 Linux 6.6 内核基线沉淀的工程思想，融合 Airymax 五维正交 24 原则后，针对智能体操作系统场景重新表述的工程契约。代码格式（缩进 / 行宽 / clang-format）见本文档 Part II，代码风格（模块化 / 抽象层次）见本文档 Part III。

**权威声明**：本卷是 agentrt-linux **语义层代码规则**（命名/函数/注释/类型/错误处理/内存/锁）的唯一权威来源（SSoT）。`10-coding-style/C_Cpp_coding_style.md Part I` 中的命名约定（§3）、函数定义（§4）、错误处理（§7）、类型（§6）、内存管理（§8）、锁与并发（§9）均引用本卷，不得重新定义。若发现不一致，以本卷为准。

**规则编号**：每条强制规则赋予唯一编号。`OS-KER-xxx` 为内核工程规则（强制，agentrt 不涉及内核态）；`OS-STD-CODE-xxx` 为标准编码规则；`OS-BAN-xxx` 为禁止规则；`OS-ACC-xxx` 为验收标准。多语言对照追求语义等价而非逐字翻译——同一规则在不同语言中以该语言最自然的方式落地。规则编号注册表汇总于 `07-maintainers-and-governance.md`；**模块前缀注册表**的唯一权威来源（SSoT）为 [`10-coding-style/coding_conventions.md Part IV §4.4.3`](10-coding-style/coding_conventions.md#模块前缀清单总表)。

***

## 1. 命名规范

C 是朴素的语言，命名应与之相称。agentrt-linux 命名遵循两条核心准则：**全局可见者必语义完备，局部可见者宜短小精悍**。

### 1.1 全局变量与全局函数

> **OS-STD-CODE-001**： 全局符号必须描述性命名，禁止无意义缩写。

```c
/* 好 */
int count_active_users(void);
struct list_head *airy_sched_ready_queue(void);

/* 坏 */
int cntusr(void);
struct list_head *q(void);
```

### 1.2 局部变量

> **OS-STD-CODE-028**： 局部变量短小切题，循环计数器用 `i`，临时值用 `tmp`，缓冲区用 `buf`。需要长名才能区分说明函数职责过重，应拆分。

```c
int i;                /* 循环计数器 */
char *tmp;            /* 临时指针 */
u8 buf[256];          /* 缓冲区 */
struct airy_task *task;
```

### 1.3 敏感术语禁用

> **OS-BAN-001**： 新增代码禁用 `master/slave`、`blacklist/whitelist`，替换为 `primary/secondary`、`denylist/allowlist`。仅当维护既有 UAPI 或对齐既有硬件 / 协议规范时例外。

```c
/* 好 */                          /* 坏 */
struct airy_primary_node *p;   struct airy_master *m;
bool denylist_match(name);        bool blacklist_match(name);
```

### 1.4 宏与枚举命名

> **OS-STD-CODE-029**： 常量宏全大写；多个相关常量优先用 `enum` 而非散落 `#define`。

```c
#define AIRY_MAX_TASKS     1024
enum airy_task_state {
	AIRY_TASK_PENDING   = 0,
	AIRY_TASK_RUNNING   = 1,
	AIRY_TASK_COMPLETED = 2,
	AIRY_TASK_FAILED    = 3,
};
```

### 1.5 函数返回值与命名约定

> **OS-STD-CODE-030**： 动作式 / 命令式函数返回错误码（0 = 成功，-Exxx = 失败）；谓词式函数返回 `bool`。混合两种约定是难调 Bug 的温床。

```c
int  airy_task_submit(struct airy_task *task);   /* 0 / -EBUSY */
bool airy_task_is_pending(const struct airy_task *task);
```

```rust
// Rust 等价：动作式返回 Result<(), ErrCode>，谓词式返回 bool
fn airy_task_submit(t: &mut Task) -> Result<(), ErrCode>;
fn airy_task_is_pending(t: &Task) -> bool;
```

### 1.6 agentrt-linux 专属前缀

> **OS-STD-CODE-005**： `airy_*` 前缀保留给 agentrt 同源 API；`airy_*` 前缀用于 agentrt-linux 内核 / 发行版专属 API。两者共享 Airymax 同源语义（MicroCoreRT / AgentsIPC / Cupolas / MemoryRovol / CoreLoopThree），但代码归属与 ABI 边界不同，前缀隔离确保无适配层互操作时不冲突。

```c
int airy_ipc_send(u32 ch, const void *msg, size_t len);     /* agentrt 同源 */
int airy_lsm_hook_register(const struct security_hook_list *hooks); /* OS 专属 */
```

**五维正交映射**：E-5 命名语义化、A-1 极简主义、K-2 接口契约化。

***

## 2. 函数规范

### 2.1 函数长度

> **OS-STD-CODE-006**： 函数一屏可读（≤80×24），局部变量 ≤5–10 个。复杂度与长度成反比：200 行的简单 `switch` 调度器可接受，200 行胶水函数不可接受。局部变量超 10 个应拆分。

### 2.2 函数原型元素顺序

> **OS-STD-CODE-007**： 原型元素固定顺序：storage class → storage class attributes → return type → return type attributes → name → parameters → parameter attributes → behavior attributes。

```c
/* 声明：参数属性在参数表后 */
__init void * __must_check
airy_ipc_channel_create(enum airy_ipc_type type, size_t cap,
			  u32 flags) __printf(2, 3) __malloc;

/* 定义：参数属性移到 storage class attributes 之后 */
static __always_inline __init __printf(2, 3) void * __must_check
airy_ipc_channel_create(enum airy_ipc_type type, size_t cap,
			  u32 flags) __malloc { /* ... */ }
```

### 2.3 参数命名

> **OS-STD-CODE-008**： 原型必须含参数名——它是面向读者的廉价文档。

```c
int airy_task_submit(struct airy_task *task);  /* 好 */
int airy_task_submit(struct airy_task *);       /* 坏 */
```

### 2.4 禁止 extern 修饰函数声明

> **OS-BAN-007**： 函数声明禁止 `extern`，使行变长且无额外语义。

### 2.5 EXPORT\_SYMBOL 紧跟函数定义

> **OS-STD-CODE-009**： 导出符号的 `EXPORT_SYMBOL*` 宏必须紧跟右大括号行，中间不留空行。

```c
int airy_perm_granted(const struct cred *c, u32 action) { /* ... */ }
EXPORT_SYMBOL_GPL(airy_perm_granted);
```

**五维正交映射**：K-2 接口契约化、E-7 文档即代码、A-1 极简主义。

***

## 3. 注释规范

### 3.1 注释原则

> **OS-STD-CODE-010**： 注释说明 WHAT 与 WHY，不说明 HOW。好代码自解释 HOW。函数体内尽量避免注释；若需分段注释应拆分为多个函数。函数头说明做什么、为什么这样做。

### 3.2 多行注释风格

> **OS-STD-CODE-011**： 通用代码用通用风格；`net/` 与 `drivers/net/` 用 net 风格。

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

> **OS-STD-CODE-012**： 所有公共 API（`EXPORT_SYMBOL*` 函数、公共结构体、公共宏）必须用 kernel-doc 注释。

```c
/**
 * airy_ipc_send() - 在指定通道上发送消息
 * @channel: 通道 ID，由 airy_ipc_channel_create() 返回
 * @msg:    消息体指针，长度不超过 AIRY_IPC_MSG_BODY_MAX
 * @len:    消息体字节数
 *
 * 阻塞发送，直到对端读取或超时。消息头由 AgentsIPC 128B 协议自动填充。
 *
 * Return: 0 成功；-EAGAIN 通道满；-EMSGSIZE 超长；-ETIMEDOUT 超时。
 */
int airy_ipc_send(u32 channel, const void *msg, size_t len);
```

### 3.4 数据注释

> **OS-STD-CODE-013**： 数据声明每行一个，留出注释空间。

```c
struct airy_task {
	u32    id;            /* 全局唯一任务 ID */
	u32    parent_id;     /* 父任务 ID，根任务为 0 */
	u8     priority;      /* 0=最低，255=最高 */
	u8     state;         /* enum airy_task_state */
	u16    flags;         /* AIRY_TASK_FLAG_* 位掩码 */
};
```

### 3.5 agentrt-linux 多语言注释规范

> **OS-STD-CODE-014**： C 用 kernel-doc；Rust 用 rustdoc；Python 用 Google docstring；TypeScript 用 JSDoc。

```rust
/// 在指定通道上发送消息（阻塞）。
/// # 参数
/// * `channel` - 通道 ID
/// # 返回
/// `Ok(())` 成功；`Err(ErrCode::Again|MsgSize|TimedOut)` 失败。
pub fn airy_ipc_send(channel: u32, msg: &[u8]) -> Result<(), ErrCode>;
```

```python
def airy_ipc_send(channel: int, msg: bytes) -> None:
    """在指定通道上发送消息（阻塞）。
    Args:
        channel: 通道 ID。
        msg:     消息体。
    Raises:
        AgentrtError: EAGAIN / EMSGSIZE / ETIMEDOUT。
    """
```

**五维正交映射**：E-7 文档即代码、K-2 接口契约化、A-3 人文关怀。

***

## 4. 头文件与 import 规范

### 4.1 显式声明原则

> **OS-STD-CODE-015**： 每个 `.c` 必须显式 `#include` 其直接依赖的头文件，不得依赖间接传递包含。传递依赖会在头文件重构时引发雪崩式编译错误。

```c
/* 好 */                       /* 坏：依赖 task.h 间接拉入 slab.h */
#include <linux/slab.h>        #include <agentrt/task.h>
#include <agentrt/ipc.h>        kfree(p);
```

### 4.2 include 顺序

> **OS-STD-CODE-016**： include 顺序固定：系统头 → 架构头 → 本地头 → `#define CREATE_TRACE_POINTS` → trace events 头。

```c
#include <linux/module.h>
#include <linux/slab.h>
#include <asm/barrier.h>
#include "airy_internal.h"
#define CREATE_TRACE_POINTS
#include <trace/events/agentrt.h>
```

### 4.3 禁止自动排序

> **OS-BAN-008**： 禁止 IDE / 工具自动排序 include。`.clang-format` 必须设 `SortIncludes: false`。include 顺序承载语义分组，自动排序会破坏分组并引入隐式依赖。

### 4.4 include guard 规范

> **OS-STD-CODE-017**： 头文件用 `#ifndef` / `#define` / `#endif` 保护，guard 名与文件路径对应。Rust / TS 模块系统天然隔离，无需 guard。

```c
#ifndef _AIRY_IPC_H
#define _AIRY_IPC_H
/* ... */
#endif /* _AIRY_IPC_H */
```

**五维正交映射**：K-2 接口契约化、E-1 安全内生、A-1 极简主义。

***

## 5. 宏使用规范

### 5.1 五类应避免的宏

> **OS-BAN-009**： 禁止以下五类宏：(1) 影响控制流（宏内 `return`）；(2) 依赖魔法命名的局部变量；(3) 作为左值使用的宏参数；(4) 忽略运算优先级的宏常量；(5) 函数式宏中局部变量命名空间冲突。

```c
/* 坏 1 */  #define FOO(x) do { if (blah(x) < 0) return -EINVAL; } while (0)
/* 坏 2 */  #define FOO(val) bar(index, val)
/* 坏 3 */  #define FOO(x) (x)->field
/* 坏 4 */  #define CONSTEXP CONSTANT | 3          /* 应为 (CONSTANT | 3) */
/* 坏 5 */  #define FOO(x) ({ typeof(x) ret; ret = calc(x); (ret); })
```

### 5.2 多语句宏必须用 do-while(0)

> **OS-STD-CODE-018**： 多语句宏必须用 `do { ... } while (0)` 包裹，使其在 `if` 单语句体中行为正确。

```c
#define airy_assert_locked(l) do { lockdep_assert_held(&(l)->lock); } while (0)
```

### 5.3 inline functions 优于宏

> **OS-STD-CODE-019**： 优先用 `static inline` 函数替代函数式宏——有类型检查、不重复求值、不陷阱于优先级。Rust 用 `const fn` 完全替代。

```c
static inline u32 airy_task_priority_max(void) { return AIRY_MAX_PRIO; }
```

**五维正交映射**：E-1 安全内生、A-2 细节关注、K-2 接口契约化。

***

## 6. 类型规范

### 6.1 typedef 严格限制

> **OS-STD-CODE-020**： typedef 仅在以下 5 种例外下使用，其余禁止：(1) 完全不透明对象（如 `pte_t`，仅靠 accessor 访问）；(2) 清晰整数类型（避免 `int` / `long` 歧义，如 `u8/u32`）；(3) sparse 创建的新类型；(4) 与 C99 标准类型等价；(5) 用户空间可见的安全类型（`__u32`）。

```c
struct airy_task *task;     /* 好：结构体直接 struct */
typedef struct airy_task *task_ptr_t;  /* 坏：掩盖类型 */
```

### 6.2 内核原生类型

> **OS-STD-CODE-032**： 内核内部用 `u8/u16/u32/u64` 及有符号变体；用户空间可见的 UAPI 结构体用 `__u8/__u16/__u32/__u64`（2026-07-15 从 OS-STD-CODE-031 让位消解，OS-STD-CODE-031 权威定义为 C_Cpp_coding_style.md §3.2 snake_case 命名规则）。

```c
struct airy_task {        u32 id; u64 deadline_ns; };          /* 内核内部 */
struct airy_ipc_msg_hdr { __u32 magic; __u16 opcode; __u64 trace_id; }; /* UAPI（节选自 SSoT Layout C） */
```

### 6.3 bool 类型使用边界

> **OS-STD-011**： `bool` 仅用于函数返回值与栈变量；对齐敏感的结构体成员、cache line 布局敏感场景不用 `bool`——其大小与对齐随架构变化。

```c
bool airy_task_is_pending(const struct airy_task *t);  /* 好：返回值 */
struct stats { bool fast_path; u32 hits; };                  /* 坏：破坏 cache line */
```

### 6.4 位域与紧凑布局

> **OS-STD-CODE-021**： 多个布尔标志位应合并为 1 位 bitfield 或单个 `u8` flags，避免逐个 `bool`。

```c
struct airy_task {
	u32 id, parent_id;
	u8  priority, state;
	u8  flags;        /* AIRY_TASK_FLAG_* 位掩码 */
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

***

## 7. 错误处理规范（强制范式）

错误处理是 agentrt-linux 工程观的核心。本章所有规则均为强制。

### 7.1 goto 集中出口模式

> **OS-KER-001**： 函数多出口且需公共清理时，必须用 `goto` 跳转到集中出口标签。标签名应描述其行为（如 `out_free_buffer:`），禁止 `err1:` / `err2:` 编号式命名。

```c
int airy_session_open(struct airy_session **out, const char *name)
{
	struct airy_session *s;
	void *buf;
	int ret;

	s = kzalloc(sizeof(*s), GFP_KERNEL);
	if (!s) return -ENOMEM;

	buf = kmalloc(PAGE_SIZE, GFP_KERNEL);
	if (!buf) { ret = -ENOMEM; goto out_free_session; }

	ret = airy_ipc_channel_create(name, &s->chan);
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

> **OS-KER-004**： 多个出口标签必须按资源分配的逆序释放，每个标签只释放其对应资源，防止 one-err bug（某条路径上指针未初始化却被释放）。

```c
/* 坏：foo 可能为 NULL 时仍 kfree(foo->bar) */   /* 好：分级标签 */
err:                                              err_free_bar:
	kfree(foo->bar);                                kfree(foo->bar);
	kfree(foo);                                     err_free_foo:
	return ret;                                      kfree(foo);
                                                   return ret;
```

理想情况下应通过错误注入测试覆盖所有出口路径。

### 7.3 禁止 BUG()/BUG\_ON()/panic()

> **OS-BAN-002**： 禁止新增 `BUG()` / `BUG_ON()` / `VM_BUG_ON()` / `panic()`，改用 `WARN_ON_ONCE()` + 恢复代码。`BUG()` 与 `panic()` 让系统在未释放锁、未恢复状态下崩溃，使事后调试不可能——与 OLK-6.6 `panic()` 治理对齐（OLK-6.6 全仓 `panic()` 调用仅限早期启动 arch 代码与硬件不可恢复故障，子系统代码零 `panic()`）。仅当重大内部损坏且无任何继续运行可能时例外（如早期启动硬件探测失败），且需充分论证并经维护者签字。`WARN*()` 仅用于"不应到达"场景；"可达但不期望"用 `pr_warn_once()`。`BUILD_BUG_ON()` 是编译期断言，鼓励使用。

```c
if (WARN_ON_ONCE(!mutex_is_locked(&p->lock)))   /* 好 */
		return -EINVAL;
BUG_ON(!mutex_is_locked(&p->lock));             /* 坏 */
panic("agentrt: unrecoverable state");          /* 坏：禁止，改 WARN_ON_ONCE + goto err */
```

### 7.4 内存分配惯用法

> **OS-BAN-003**： 传递结构体大小用 `sizeof(*p)`；数组分配用 `kmalloc_array` / `kcalloc`（带溢出检查）；带尾数组用 `struct_size` / `flex_array_size`。禁止在分配器参数中手写乘法（溢出风险）。

```c
struct airy_task *task = kmalloc(sizeof(*task), GFP_KERNEL);   /* 好 */
struct airy_task *arr  = kcalloc(n, sizeof(*arr), GFP_KERNEL);  /* 好：溢出检查 */
struct airy_batch *b = kzalloc(struct_size(b, items, n), GFP_KERNEL); /* 好：柔性数组 */
arr = kmalloc(n * sizeof(*arr), GFP_KERNEL);                      /* 坏：可能溢出 */
```

### 7.5 字符串复制安全

> **OS-BAN-004**： 禁止 `strcpy` / `strncpy` / `strlcpy`。强制 `strscpy`（需 NUL 终止）或 `strscpy_pad`（需 NUL 填充）。`strcpy` 无边界检查；`strncpy` 不保证终止；`strlcpy` 读全源串可能越界。

```c
strscpy(dst->name, src->name, sizeof(dst->name));
```

### 7.6 禁用 %p 默认输出

> **OS-BAN-005**： 禁止新增 `%p` 默认输出——所有 `%p` 自动哈希化，对寻址无用。需要真实指针须在注释与 commit log 充分论证后用 `%px` 并确保权限控制；文本地址用 `%pS` 输出符号名。

```c
pr_info("task %u state %s\n", task->id, state_name);   /* 好 */
pr_info("handler at %pS\n", task->handler);           /* 好 */
pr_info("task at %p\n", task);                          /* 坏：已哈希，无意义 */
```

### 7.7 agentrt-linux 多语言错误处理

> **OS-STD-CODE-022**： C 用 goto 集中出口；Rust 用 `?` + `Result` + RAII；Python 用异常 + 上下文管理器；TypeScript 用 try-finally。

```rust
fn airy_session_open(name: &str) -> Result<Session, ErrCode> {
    let s = Session::new()?;            /* 失败自动 return */
    let buf = Box::<[u8; 4096]>::new_uninit_slice()?;
    let chan = airy_ipc_channel_create(name)?;
    Ok(Session { s, buf, chan })        /* RAII 自动清理 */
}
```

```python
def airy_session_open(name: str) -> Session:
    with Session.alloc() as s, Buffer.alloc(PAGE_SIZE) as buf:
        s.bind_channel(airy_ipc_channel_create(name))
        return s.finalize()            # with 块退出自动释放
```

**五维正交映射**：E-1 安全内生（安全字符串 / 溢出检查）、E-3 资源确定性（goto 集中出口保证释放）、E-6 错误可追溯（分级标签即错误路径文档）、K-2 接口契约化（错误码即契约）。

***

## 8. 并发与 volatile 规范

### 8.1 volatile 几乎从不正确

> **OS-KER-006**： `volatile` 仅在以下 4 种罕见情况下使用，其余禁止：(1) I/O 内存访问器内部（架构相关）；(2) 防止 GCC 删除无可见副作用的内联汇编；(3) `jiffies` 等"每次读取值不同但无需锁"的遗留变量；(4) 一致性内存中可能被 I/O 设备修改的环缓冲指针。`volatile` 抑制优化而非保护并发——锁原语已包含必要的编译器屏障。

```c
volatile int shared_counter;        /* 坏：volatile 不能替代锁 */
spinlock_t lock; int counter;       /* 好：spinlock 保护，屏障由锁提供 */
```

### 8.2 共享数据必须用同步原语保护

> **OS-KER-053**： 共享数据必须通过 spinlock / mutex / memory barrier / RCU 保护。锁保持数据一致性，引用计数管理生命周期，两者不可互替。

```c
struct airy_task_table { spinlock_t lock; struct list_head tasks; };

void airy_task_table_add(struct airy_task_table *t, struct airy_task *task)
{
	spin_lock(&t->lock);
	list_add_tail(&task->link, &t->tasks);
	spin_unlock(&t->lock);
}
```

### 8.3 I/O 内存必须通过 accessor 访问

> **OS-KER-056**： I/O 内存禁止通过裸指针解引用访问，必须通过 `readl` / `writel` / `ioread32` / `iowrite32` 等 accessor。accessor 已封装平台差异与编译器屏障。

```c
writel(AIRY_REG_ENABLE, base + AIRY_REG_CTRL);   /* 好 */
volatile u32 *ctrl = (volatile u32 *)(base + AIRY_REG_CTRL); /* 坏 */
*ctrl = AIRY_REG_ENABLE;
```

### 8.4 忙等用 cpu\_relax()

> **OS-KER-009**： 忙等循环必须调用 `cpu_relax()`，降低功耗并提示超线程孪生核心，同时作为编译器屏障。

```c
while (readl(base + AIRY_REG_STATUS) & AIRY_STATUS_BUSY)
	cpu_relax();
```

### 8.5 Rust Send/Sync 强制要求

> **OS-STD-CODE-023**： Rust 跨线程传递的类型必须实现 `Send`；跨线程共享的类型必须实现 `Sync`。agentrt-linux Rust 同源组件禁止 `unsafe impl Send/Sync`，除非有书面论证与 unsafe audit。

```rust
struct AgentrtChannel { inner: Mutex<ChannelInner> }  /* Mutex<T> 自动提供 Sync */
```

**五维正交映射**：E-1 安全内生、E-3 资源确定性、K-2 接口契约化、A-2 细节关注。

***

## 9. 条件编译与 magic number

### 9.1 尽可能不在 .c 中用 #if/#ifdef

> **OS-STD-CODE-024**： `.c` 中尽量避免 `#if` / `#ifdef`。条件逻辑应在头文件中提供 stub 函数，`.c` 无条件调用。整函数编译裁剪优于函数内片段裁剪。若函数 / 变量在某配置下未使用，用 `__maybe_unused` 标注而非 `#ifdef` 包裹。

```c
/* 头文件：stub */
#ifdef CONFIG_AIRY_AUDIT
static inline void airy_audit_record(u32 e) { /* real */ }
#else
static inline void airy_audit_record(u32 e) { /* no-op */ }
#endif
/* .c：无条件调用 */
void airy_task_submit(struct airy_task *t) {
	airy_audit_record(EVT_SUBMIT);
	__airy_task_enqueue(t);
}
```

### 9.2 用 IS\_ENABLED 转 C 布尔表达式

> **OS-STD-CODE-025**： 在 C 表达式中用 `IS_ENABLED(CONFIG_XXX)` 而非 `#ifdef`，让编译器仍对内部代码做语法 / 类型检查。

```c
if (IS_ENABLED(CONFIG_AIRY_AUDIT)) { airy_audit_enable(); }  /* 好 */
#ifdef CONFIG_AIRY_AUDIT airy_audit_enable(); #endif         /* 坏：内部不检查 */
```

### 9.3 #endif 同行必须注释条件

> **OS-STD-CODE-026**： 非平凡 `#if` / `#ifdef` 块（>数行）的 `#endif` 同行必须注释其条件。

```c
#ifdef CONFIG_AIRY_AUDIT
	/* ... 数十行 ... */
#endif /* CONFIG_AIRY_AUDIT */
```

### 9.4 magic number 必须注册

> **OS-KER-010**： 受保护的数据结构应在结构体开头声明 `magic` 字段，并将该 magic number 注册到 agentrt-linux 维护的 `magic-numbers.md`。magic number 用于运行时检测结构体被覆盖或类型误传，在数组越界覆盖相邻结构的场景中尤其有效。

```c
#define AIRY_SESSION_MAGIC  0x41475331  /* 'AGS1' */
struct airy_session { u32 magic; u32 id; /* ... */ };
static inline bool airy_session_valid(const struct airy_session *s) {
	return s && s->magic == AIRY_SESSION_MAGIC;
}
```

### 9.5 Kconfig 配置文件规范

> **OS-STD-CODE-027**： Kconfig 用 tab 缩进；help 文本额外缩进两空格；危险特性必须在 prompt 字符串中标注 `(DANGEROUS)`。

```
config AIRY_AUDIT
	bool "Agentrt audit subsystem"
	depends on NET
	help
	  Enable the Agentrt audit subsystem that records all permission
	  decisions into an append-only hash chain.

config AIRY_DEVMEM_RW
	bool "Direct /dev/mem write access (DANGEROUS)"
	depends on MMU
```

**五维正交映射**：E-1 安全内生（DANGEROUS 标注）、E-7 文档即代码（Kconfig 即配置文档）、K-2 接口契约化（stub 即契约裁剪）、A-3 人文关怀（#endif 注释尊重后续读者）。

***

## 10. 五维正交原则总映射表

| 章节     | K 内核观    | E 工程观                 | A 设计美学   |
| ------ | -------- | --------------------- | -------- |
| 1 命名   | K-2 接口契约 | E-5 命名语义化             | A-1 极简主义 |
| 2 函数   | K-2 接口契约 | E-7 文档即代码             | A-1 极简主义 |
| 3 注释   | K-2 接口契约 | E-7 文档即代码             | A-3 人文关怀 |
| 4 头文件  | K-2 接口契约 | E-1 安全内生              | A-1 极简主义 |
| 5 宏    | K-2 接口契约 | E-1 安全内生              | A-2 细节关注 |
| 6 类型   | K-2 接口契约 | E-3 资源确定性             | A-2 细节关注 |
| 7 错误处理 | K-2 接口契约 | E-1 / E-3 / E-6 错误可追溯 | —        |
| 8 并发   | K-2 接口契约 | E-1 / E-3 资源确定性       | A-2 细节关注 |
| 9 条件编译 | K-2 接口契约 | E-1 / E-7 文档即代码       | A-3 人文关怀 |

> **同源语义注**： 本章规则在 agentrt 用户态运行时侧有等价表述（agentrt `IRON-1`\~`IRON-10` 同源铁律），两端通过 AgentsIPC 128B 消息头协议实现无适配层互操作——这是 agentrt-linux 与 agentrt "同源且部分代码共享（IRON-9 v2）"关系在代码规范层的体现。

***

## 11. 本章规则编号注册表

| 编号              | 强制度 | 一句话规则                                   |
| --------------- | --- | --------------------------------------- |
| OS-STD-CODE-001 | 标准  | 全局符号描述性命名                                |
| OS-BAN-001      | 禁止  | 敏感术语 master/slave、blacklist/whitelist   |
| OS-STD-CODE-005 | 标准  | `airy_*` 与 `airy_*` 前缀隔离           |
| OS-STD-CODE-006 | 标准  | 函数 ≤80×24，局部变量 ≤5–10                    |
| OS-STD-CODE-007 | 标准  | 函数原型元素固定顺序                              |
| OS-STD-CODE-008 | 标准  | 原型必须含参数名                                |
| OS-BAN-007      | 禁止  | 函数声明禁止 `extern`                         |
| OS-STD-CODE-009 | 标准  | EXPORT\_SYMBOL 紧跟右大括号                   |
| OS-STD-CODE-010 | 标准  | 注释写 WHAT/WHY 不写 HOW                     |
| OS-STD-CODE-011 | 标准  | 多行注释通用风格与 net 风格                        |
| OS-STD-CODE-012 | 标准  | 公共 API 强制 kernel-doc                    |
| OS-STD-CODE-013 | 标准  | 数据声明每行一个                                |
| OS-STD-CODE-014 | 标准  | 多语言注释规范                                 |
| OS-STD-CODE-015 | 标准  | 显式 include，禁止间接传递                       |
| OS-STD-CODE-016 | 标准  | include 顺序固定                            |
| OS-BAN-008      | 禁止  | 禁止自动排序 include                          |
| OS-STD-CODE-017 | 标准  | include guard 命名规范                      |
| OS-BAN-009      | 禁止  | 五类危险宏                                   |
| OS-STD-CODE-018 | 标准  | 多语句宏 do-while(0)                        |
| OS-STD-CODE-019 | 标准  | inline 优于宏                              |
| OS-STD-CODE-020 | 标准  | typedef 仅 5 种例外                         |
| OS-STD-CODE-032 | 标准  | 内部 u8/u32，UAPI \_\_u32                  |
| OS-STD-011      | 标准  | bool 仅返回值与栈变量                           |
| OS-STD-CODE-021 | 标准  | 多布尔值合并 bitfield / flags                 |
| OS-KER-001      | 内核  | goto 集中出口模式                             |
| OS-KER-004      | 内核  | 分级标签按分配逆序释放                             |
| OS-BAN-002      | 禁止  | BUG()/BUG\_ON()，改 WARN\_ON\_ONCE        |
| OS-BAN-003      | 禁止  | sizeof(\*p)/kmalloc\_array/struct\_size |
| OS-BAN-004      | 禁止  | strcpy/strncpy/strlcpy，强制 strscpy       |
| OS-BAN-005      | 禁止  | %p 默认输出                                 |
| OS-STD-CODE-022 | 标准  | 多语言错误处理范式                               |
| OS-KER-006      | 内核  | volatile 仅 4 种例外                        |
| OS-KER-053      | 内核  | 共享数据用同步原语保护                             |
| OS-KER-056      | 内核  | I/O 内存用 accessor                        |
| OS-KER-009      | 内核  | 忙等用 cpu\_relax                          |
| OS-STD-CODE-023 | 标准  | Rust Send/Sync 强制                       |
| OS-STD-CODE-024 | 标准  | .c 中避免 #if，用 stub                       |
| OS-STD-CODE-025 | 标准  | IS\_ENABLED 转 C 布尔                      |
| OS-STD-CODE-026 | 标准  | #endif 同行注释条件                           |
| OS-KER-010      | 内核  | magic number 注册                         |
| OS-STD-CODE-027 | 标准  | Kconfig tab 缩进，DANGEROUS 标注             |
| OS-STD-CODE-028 | 标准  | 局部变量短小精悍                                |
| OS-STD-CODE-029 | 标准  | 宏常量全大写，相关常量优先 enum                      |
| OS-STD-CODE-030 | 标准  | 动作式返回错误码，谓词式返回 bool                     |

合计 44 条规则：13 条 `OS-KER` 内核规则、22 条 `OS-STD` 标准规则、7 条 `OS-BAN` 禁止规则、2 条多语言相关标准。完整编号注册表（含 IRON / ACC / DRV / BUILD 等）汇总于 `07-maintainers-and-governance.md`。

***

## 12. 相关文档

- `50-engineering-standards/README.md` — 工程标准主索引与总纲
- 本文档 Part II — 代码格式（缩进 / 行宽 / clang-format）
- 本文档 Part III — 代码风格（模块化 / 抽象层次 / 防御性）
- `50-engineering-standards/04-engineering-philosophy.md` — 工程思想（双层稳定性 / 策略机制分离）
- `50-engineering-standards/07-maintainers-and-governance.md` — 维护者制度与规则编号注册表
- `docs/AirymaxRT/00-architectural-principles.md` — Airymax 五维正交 24 原则
- `50-engineering-standards/magic-numbers.md` — agentrt-linux magic number 注册表
- `30-interfaces/01-syscall-spec.md` — 系统调用规范（UABI 边界）

***

# Part II: 代码格式规范

## 0. 章节定位

本卷回答"代码长什么样"——纯机械的格式规则。它不讨论"该怎么写代码"（01 代码规范）、"该怎么思考"（03 代码风格）或"工程体系怎么建立"（04 工程思想），只规定视觉层面的可由工具 100% 自动化的规则。agentrt-linux 的格式哲学是：

> **配置即代码（OS-STD-FMT-030）**：每一条格式规则必须有对应的格式化工具配置项；规则若无法被工具强制，等同于不存在。

- **权威声明**：本卷是 agentrt-linux 代码格式规则的**唯一权威来源（SSoT）**。`10-coding-style/C_Cpp_coding_style.md Part I` 和 `10-coding-style/C_Cpp_coding_style.md Part II` 中的格式规则（缩进/行宽/大括号/空格）均引用本卷，不得重新定义。若发现不一致，以本卷为准。
- **上游依赖**：04 工程思想定义"为什么"——双层稳定性、可读性优先、cache 影响是头等大事；本卷定义"长什么样"。
- **下游依赖**：06 工具链与自动化定义"格式检查在哪一层执行"——预提交钩子（Layer 3）+ CI 门禁（Layer 4）必须运行 `make format-check`；本卷定义"被检查的内容"。

本卷格式规则统一赋予 **OS-STD-FMT**（格式子域）编号，内核态专属规则沿用 **OS-KER** 编号（见 §0.2.3），与 07 维护者制度与治理的"规则编号注册表"对齐。

### 0.1 关键术语

| 术语 | 定义 |
|------|------|
| Tab 宽度 | Tab 字符在显示时占用的列数；内核 C 代码统一为 8 |
| K&R 风格 | Kernighan & Ritchie 在《The C Programming Language》中确立的大括号放置风格 |
| 行尾空白 | 行末 Tab 或空格序列；agentrt-linux 全部禁用 |
| 后继行 | 被断行的长行的续行；必须显著短于父行并右移 |
| 配置即代码 | 格式规则与格式化工具配置项一一对应，无人工裁量空间 |

### 0.2 OS-KER → OS-STD-FMT 编号迁移映射

> **本节为编号迁移映射声明**。本卷历史编号 `OS-KER-003~021` 与 `OS-STD-047 / OS-STD-202~211` 已迁移为统一前缀 `OS-STD-FMT-NNN`（FMT 子域）。下表给出权威映射，消除以下三类问题：(1) `OS-KER-005` 双义（Tab 8 字符 vs 字符串禁止断行）；(2) K&R 大括号被拆为 `OS-STD-047 + OS-KER-008` 两个编号；(3) 空格规则被拆为 `OS-KER-010~014` 五个编号。

#### 0.2.1 格式规则迁移映射表

| 当前编号（本卷正文） | 目标编号（OS-STD-FMT） | 规则内容 | 所在章节 | 迁移说明 |
|---------------------|----------------------|----------|---------|---------|
| OS-KER-005（Tab 8 字符） | OS-STD-FMT-001 | Tab 缩进，8 字符宽 | §1.1 | 消除双义：OS-KER-005 在 §1.1 表 Tab、在 §2.2 表字符串禁止断行 |
| OS-KER-004 | OS-STD-FMT-002 | C 代码首选 80 列 | §2.1 | — |
| OS-KER-005（字符串禁止断行） | OS-STD-FMT-008 | 用户可见字符串禁止断行 | §2.2 | 消除双义：拆分为独立 OS-STD-FMT-008 |
| OS-STD-047 | OS-STD-FMT-003 | 非函数语句块开括号在行末（K&R） | §3.1 | 与 OS-KER-008 合并为 K&R 大括号族 |
| OS-KER-008 | OS-STD-FMT-009 | 函数例外：开括号在下一行行首 | §3.2 | 与 OS-STD-047 合并为 K&R 大括号族 |
| OS-KER-009 | OS-STD-FMT-004 | 单语句与多语句的大括号规则 | §3.4 | Linux 6.6 内核基线 立场：单语句可不加（采纳 02 立场） |
| OS-KER-010 / OS-KER-011 / OS-KER-014 | OS-STD-FMT-005 | 空格规则（合并 3 条为 1 条） | §4.1/§4.2/§4.4/§4.5 | 合并：关键字后空格 / sizeof 无空格 / 运算符空格 / 括号内禁空格。注：OS-KER-012 单独迁移至 OS-STD-FMT-006（见下行），OS-KER-013 单独迁移至 OS-STD-CODE-020（typedef 规则，见 Part III §6.1）。2026-07-15 修正：原表"OS-KER-010~014 合并 5 条"错误，实际为 3 条合并 + 2 条单独迁移。 |
| OS-KER-012（指针声明 `*` 贴名字） | OS-STD-FMT-006 | 指针对齐 | §4.3 | 注意：OS-KER-012 在 §4.3 表指针、在 §1.x 不出现，迁移后消除歧义 |
| OS-KER-015 | OS-STD-FMT-010 | case 标签与 switch 对齐 | §5.1 | — |
| OS-KER-016 | OS-STD-FMT-011 | fallthrough 必须显式标注 | §5.2 | — |
| OS-KER-017 | OS-STD-FMT-012 | 函数之间用 1 个空行分隔 | §6.1 | — |
| OS-KER-018 | OS-STD-FMT-013 | 禁止函数内多余空行 | §6.2 | — |
| OS-KER-019 | OS-STD-FMT-004（部分） | 单行多语句禁令 | §7.1 | 与 §3.4 单语句大括号规则协同 |
| OS-KER-020 | OS-STD-FMT-014 | 禁止用逗号逃避大括号 | §7.2 | — |
| OS-KER-021 | OS-STD-FMT-014（扩展） | 禁止单行多赋值 | §7.3 | 与 OS-KER-020 合并为 OS-STD-FMT-014 |
| OS-KER-003 | OS-STD-FMT-007 | 行尾禁止空白 | §1.6 | — |
| OS-KER-006 | OS-STD-FMT-015 | 后继行应大幅短于父行 | §2.4 | 新增 OS-STD-FMT-015 |
| OS-KER-002（02 §1.5 Kconfig Tab） | OS-STD-FMT-016 | Kconfig Tab 缩进（DANGEROUS 标注） | §1.5 | 消解 C9 三重冲突：与 09-ssot-registry.md §3 OS-KER-002（u8/u16 区分）分离 |
| OS-STD-202 ~ OS-STD-205 | OS-STD-FMT-020 ~ OS-STD-FMT-023 | 多语言缩进与行宽规则 | §1.2-§1.4、§2.3 | 多语言段迁移至 OS-STD-FMT 子域 |
| OS-STD-206 | OS-STD-FMT-024（注册表别名） | 连续空行最多保留 1 行 | §6.3 | **§6.3 保留 OS-STD-206 为 C 语言 SSoT 定义**；OS-STD-FMT-024 为注册表别名；100-operations 的 OS-STD-206 已迁移到 OS-OPS-122（消解 C4） |
| OS-STD-207 ~ OS-STD-211 | OS-STD-FMT-025 ~ OS-STD-FMT-029 | 配置即代码工具链维护 | §8.1-§8.5 | 多语言段迁移至 OS-STD-FMT 子域 |

#### 0.2.2 活跃编号迁移至子域前缀（2026-07-14 消解三重冲突）

> **迁移说明**：原裸 `OS-STD-2xx` 编号在 `01-coding-standards.md`、`05-development-process.md`、`70-build-system/02-kconfig-system.md` 三文档中存在三重编号冲突（同一 `OS-STD-201` 指代三条完全不同的规则）。2026-07-14 消解：`05-development-process.md` 的裸编号迁移至 `OS-STD-PROD-NNN`，`70-build-system` 的裸编号迁移至 `OS-KER-030`，本卷的活跃裸编号迁移至对应子域前缀。`OS-STD-202~211` 为已迁移历史编号（见上表），`OS-STD-206` 保留为 C 语言 SSoT 别名，均不再列入下表。

| 原裸编号 | 目标子域编号 | 规则内容 | 所在章节 |
|---------|------------|----------|---------|
| OS-STD-201 | OS-STD-FMT-030 | 配置即代码——每条格式规则必须有对应格式化工具配置项 | Part II §0 |
| OS-STD-212 | OS-STD-CODE-005 | 代码重复 3 处以上应抽出到 lib/ 或 commons/ | §1.3 |
| OS-STD-213 | OS-STD-STY-003 | agentrt-linux 分层职责：热路径 C / 慢路径 Rust | §1.4 |
| OS-STD-214 | OS-STD-CODE-006 | 4 层接口稳定性分级 | §5.4 |
| OS-STD-215 | OS-STD-CODE-007 | 跨语言 FFI 边界显式声明 | §8.3 |
| OS-STD-216 | OS-STD-STY-004 | Agent 工作负载考量——Token 能效与记忆卷载 | §8.4 |
| OS-STD-217 | OS-STD-TOOL-160 | checkpatch.pl --strict | §9.5 |
| OS-STD-218 | OS-STD-TOOL-161 | 多配置构建（defconfig + allnoconfig + allmodconfig） | §9.6 |

#### 0.2.3 迁移状态

- **编号体系**：历史编号 OS-KER-XXX 与目标编号 OS-STD-FMT-NNN 等价有效。
- **编号统一**：代码实现 `.clang-format` / checkpatch 规则映射时统一替换为 OS-STD-FMT-NNN。
- **Linux 6.6 内核基线 立场采纳**：单语句大括号规则采纳 Linux 6.6 内核基线 §3 立场（单语句可不加大括号），即本卷 OS-KER-009 / 迁移后 OS-STD-FMT-004；[09-ssot-registry.md §3](./09-ssot-registry.md) 中 OS-STD-FMT-004（即使单条语句也使用大括号）**不采纳**，本卷以 Linux 6.6 内核基线 立场为准。

#### 0.2.4 迁移后保留的内核态专属 OS-KER 编号

迁移后，`OS-KER` 前缀仅保留**内核态专属规则**（非格式规则）。这些规则的权威定义不在本卷，分布在 01/03/04/07 等卷；本卷仅作交叉引用，不重新定义：

| 保留编号 | 规则 | 权威定义位置 | 本卷引用说明 |
|---------|------|-------------|-------------|
| OS-STD-010 | 内核态禁 float，强制 airy_q16_t | 10-coding-style/C_Cpp_coding_style.md Part II §4 | 2026-07-12 从 OS-KER-007 迁移至 OS-STD-010，与 09-ssot-registry.md §3 SSoT 一致 |
| OS-STD-FMT-016 | Kconfig Tab 缩进（DANGEROUS 标注） | 本卷 §1.5 | 从历史 OS-KER-002（02 §1.5 Kconfig Tab）迁移，消解与 09-ssot-registry.md §3 OS-KER-002（u8/u16 区分）的冲突 |

> **SSoT 对齐声明（2026-07-12 OS-KER-001~008 消解完成）**：历史公开编号 OS-KER-001~008 已全部迁移至 SSoT 编号，OS-KER-001/002/003 复用为 SSoT 内部编号。以下编号的权威定义见 [09-ssot-registry.md §3](./09-ssot-registry.md)，本卷不重新定义：
>
> - `OS-KER-001`：SSoT 内部复用，权威定义为"goto 集中出口模式"（01 §7.1 承载）。历史公开编号 OS-KER-001（Linux 6.6 内核基线）已迁移至 OS-IRON-010。
> - `OS-KER-002`：SSoT 内部复用，权威定义为"禁止 #if 桩函数模式"（补标，由 IRON-1 涵盖）。历史公开编号 OS-KER-002（u8/u16）已迁移至 OS-STD-CODE-032。
> - `OS-KER-003`：SSoT 内部复用，权威定义为"条件编译必须用 IS_ENABLED()"（补标；用法见 OS-STD-CODE-025，01 §9.2）。历史公开编号 OS-KER-003（typedef）已并入 OS-STD-CODE-032（扩展）。
> - 历史 OS-KER-004（BUG）→ OS-BAN-002；OS-KER-005（sizeof）→ OS-BAN-003；OS-KER-006（API unstable）→ OS-IRON-002；OS-KER-007（float）→ OS-STD-010；OS-KER-008（bool）→ OS-STD-011。
> - OS-KER-010/020/021/022/030/031/040/041/050/060 等保留为内核态专属规则，权威定义见 04 工程思想。

---

## 1. 缩进规范

缩进是控制流块边界的视觉表达。agentrt-linux 内核 C 代码严格沿用 Linux 6.6 内核基线 8 字符 Tab——这是 30+ 年沉淀的可读性经验，任何"4 空格更美观"的论调在内核上下文中都是噪声。

### 1.1 C 内核代码：Tab，8 字符宽（OS-STD-FMT-001）

**OS-STD-FMT-001**：kernel 与 security 全部 C 代码使用 Tab 缩进，Tab 宽度 8 字符；禁止用空格缩进。理由有二：

1. **易读**：连续看屏幕 20 小时后，8 字符缩进让块边界更易辨认。
2. **警告嵌套过深**：超过 3 层缩进即提示函数该重构；4 空格会容忍 6 层而不显眼，等同放弃警告。

```c
/* 正确：Tab 缩进，块边界清晰 */
int airy_ipc_recv(struct airy_ipc_msg *msg)
{
	if (msg->hdr.flags & AIRY_IPC_F_NONBLOCK) {
		if (!try_wait_for_packet(msg)) {
			return -EAGAIN;
		}
	}
	return decode_payload(msg);
}
```

```c
/* 错误：4 空格缩进（隐藏了 4 层嵌套的坏味道） */
int airy_ipc_recv(struct airy_ipc_msg *msg)
{
    if (msg->hdr.flags & AIRY_IPC_F_NONBLOCK) {
        if (!try_wait_for_packet(msg)) {
            return -EAGAIN;
        }
    }
    return decode_payload(msg);
}
```

### 1.2 Rust：4 空格（OS-STD-FMT-020）

**OS-STD-FMT-020**：Rust 代码（cognition、cloudnative、内核中的 Rust 模块）使用 4 空格缩进，由 `.rustfmt.toml` 强制。Rust 不沿用 Tab，因为 rustfmt 的默认值即如此，社区惯例已固化。

```rust
// 正确：4 空格
pub async fn submit_task(&self, desc: TaskDesc) -> Result<i32, Error> {
    let req = self.encode_request(desc)?;
    self.transport.send(req).await
}
```

### 1.3 Python：4 空格（PEP 8）（OS-STD-FMT-021）

**OS-STD-FMT-021**：Python 代码（SDK、DevStation 脚本）遵循 PEP 8，4 空格缩进，禁止 Tab。由 `black` + `isort` 强制。

### 1.4 TypeScript：2 空格（OS-STD-FMT-022）

**OS-STD-FMT-022**：TypeScript / JavaScript 代码（cloudnative 前端、SDK TS 包）使用 2 空格缩进，由 `.prettierrc` 强制。前端社区惯例固化，与 Prettier 默认值一致。

### 1.5 Kconfig 与文档例外

**OS-STD-FMT-016**：仅在注释、文档、Kconfig 中允许混用 Tab 与空格；Kconfig 中 `config` 下的字段以 1 个 Tab 缩进，`help` 文本以 Tab + 2 空格缩进。

```kconfig
config AIRYMAXOS_IPC
	bool "agentrt-linux AgentsIPC subsystem"
	depends on NET
	help
	  Enable the agentrt-linux AgentsIPC subsystem (128B fixed header).
	  Required by all agent workloads.
```

### 1.6 行尾禁止空白（OS-STD-FMT-007）

**OS-STD-FMT-007**：任何源代码行尾禁止 Tab 或空格序列。Git 在 `git diff` 中会以红色背景提示；CI 检查 `git diff --check` 任何违规即拒绝合并。

---

## 2. 行长度限制

### 2.1 C 首选 80 列（OS-STD-FMT-002）

**OS-STD-FMT-002**：C 内核代码单行首选 ≤80 列；超过 80 列的语句应拆分为合理块，除非"超过 80 列显著提升可读性且不隐藏信息"。

### 2.2 用户可见字符串禁止断行（OS-STD-FMT-008）

**OS-STD-FMT-008**：用户可见字符串（`printk` / `pr_err` / `dev_warn` 等消息）**禁止断行**。断行会破坏 `grep` 检索——线上故障排查时靠 `dmesg | grep` 定位是基本操作。

```c
/* 正确：字符串保持单行，即使超过 80 列 */
pr_err("cognition: dispatcher refused task %llu due to capability mismatch (have=%#llx, want=%#llx)\n",
       task->id, have_caps, want_caps);

/* 错误：字符串断行破坏 grep */
pr_err("cognition: dispatcher refused task %llu due to "
       "capability mismatch\n", task->id);
```

### 2.3 其他语言 100-120 列（OS-STD-FMT-023）

**OS-STD-FMT-023**：Rust / Python / TypeScript 单行 ≤100 列；TypeScript 在 JSX 模板场景放宽至 ≤120 列。各语言的 `rustfmt` / `black` / `prettier` 配置项与该限制对齐。

### 2.4 后继行应大幅短于父行（OS-STD-FMT-015）

**OS-STD-FMT-015**：被断行的长行的后继行必须显著短于父行并向右对齐；常见做法是后继行与函数开括号对齐。

```c
/* 正确：后继行与开括号对齐 */
int airy_dispatcher_select(struct airy_task_desc *desc,
                                u32 capability_mask,
                                u64 deadline_ns)
{
	...
}
```

---

## 3. 大括号位置（K&R 风格）

K&R 大括号放置是 Linux 6.6 内核基线的工程惯例，agentrt-linux 完全继承。其核心价值是**最小化空行占用**——在 25 行终端屏幕时代确立，至今仍是代码密度与可读性的最佳平衡。

### 3.1 非函数语句块：开括号在行末（OS-STD-FMT-003）

**OS-STD-FMT-003**：所有非函数语句块（`if` / `switch` / `for` / `while` / `do`）开括号置于控制语句行末，闭括号置于下一行行首。

```c
if (x == y) {
	do_a();
	do_b();
}
```

### 3.2 函数例外：开括号在下一行行首（OS-STD-FMT-009）

**OS-STD-FMT-009**：函数定义的开括号独占一行，置于行首。函数与语句块的不一致是 K&R 的有意设计——函数不可嵌套，独占一行让其更显眼。

```c
int airy_ipc_send(struct airy_ipc_msg *msg)
{
	return do_send(msg);
}
```

### 3.3 do-while 与 if-else 例外

闭括号独占一行，**除非**其后紧跟同一语句的延续（`while` 终结 do 循环、`else` 续接 if）：

```c
do {
	body_of_do_loop();
} while (condition);

if (x == y) {
	..
} else if (x > y) {
	...
} else {
	....
}
```

### 3.4 单语句与多语句的大括号规则（OS-STD-FMT-004）

**OS-STD-FMT-004**：单语句不需要大括号；但**任一分支是多语句则两分支都必须加大括号**——对称性是代码可读性的根基。

```c
/* 正确：两分支都为单语句，无需大括号 */
if (condition)
	do_this();
else
	do_that();

/* 错误：if 分支多语句而 else 单语句，不对称 */
if (condition) {
	do_this();
	do_that();
} else
	otherwise();

/* 正确：if 分支多语句时 else 也加大括号 */
if (condition) {
	do_this();
	do_that();
} else {
	otherwise();
}
```

---

## 4. 空格使用

### 4.1 关键字后空格（OS-STD-FMT-005）

**OS-STD-FMT-005**：`if` / `switch` / `case` / `for` / `do` / `while` 等关键字后必须加空格。

### 4.2 例外：`sizeof` / `typeof` / `__attribute__` 不加空格（OS-STD-FMT-005）

**OS-STD-FMT-005**：`sizeof` / `typeof` / `alignof` / `__attribute__` / `defined` 视作函数式关键字，**不加空格**。

```c
/* 正确 */
s = sizeof(struct file);
ret = __builtin_expect(!!err, 0);

/* 错误 */
s = sizeof (struct file);
```

### 4.3 指针声明 `*` 贴名字不贴类型（OS-STD-FMT-006）

**OS-STD-FMT-006**：指针声明中 `*` 紧贴变量名而非类型名。这是 C 语言指针语义本质的体现——指针属于变量，不属于类型。

```c
/* 正确 */
char *airy_banner;
unsigned long long memparse(char *ptr, char **retptr);
char *match_strdup(substring_t *s);

/* 错误：`*` 贴类型 */
char* airy_banner;
```

### 4.4 二元/三元运算符两侧加空格，一元运算符后无空格（OS-STD-FMT-005）

**OS-STD-FMT-005**：`=  +  -  <  >  *  /  %  |  &  ^  <=  >=  ==  !=  ?  :` 两侧加空格；一元 `&  *  +  -  ~  !` 后无空格；前后缀 `++  --` 紧贴操作数；结构成员 `.` / `->` 周围无空格。

```c
/* 正确 */
a = b + c;
*p = ~x & y;
if (!err && (flags & MASK) == VAL)
	r = cond ? x : y;
s->field++;

/* 错误 */
a=b+c;
*p = ~ x&y;
if( !err&&(flags&MASK)==VAL )
```

### 4.5 括号内禁止空格（OS-STD-FMT-005）

**OS-STD-FMT-005**：括号内侧禁止空格，无论是函数参数还是表达式。

```c
/* 正确 */
s = sizeof(struct file);
airy_ipc_send(msg, flags);

/* 错误 */
s = sizeof( struct file );
airy_ipc_send( msg, flags );
```

### 4.6 行尾禁止空白（重申）

`OS-STD-FMT-007` 在第 1 章已声明。此处重申：行尾空白是空格滥用的最坏形态——它既不传递信息，又会在 `git diff --check` 触发噪声。`MaxEmptyLinesToKeep: 1`（见 §9）即为其工具层防线。

---

## 5. switch/case 格式

### 5.1 case 标签与 switch 对齐（OS-STD-FMT-010）

**OS-STD-FMT-010**：`case` 标签与 `switch` 对齐到同一列，**禁止双缩进**——`case` 本身就是 `switch` 的下属，缩进一层即可表达语义。

```c
switch (suffix) {
case 'G':
case 'g':
	mem <<= 30;
	break;
case 'K':
case 'k':
	mem <<= 10;
	fallthrough;
default:
	break;
}
```

### 5.2 fallthrough 必须显式标注（OS-STD-FMT-011）

**OS-STD-FMT-011**：`switch` 中故意 fallthrough 必须 `fallthrough;` 显式标注。`-Wimplicit-fallthrough` 警告会捕捉遗漏。

---

## 6. 空行使用

### 6.1 函数之间用 1 个空行分隔（OS-STD-FMT-012）

**OS-STD-FMT-012**：源文件中函数之间恰好用 1 个空行分隔；`EXPORT_SYMBOL` 紧跟在函数闭括号行之后。

```c
int system_is_up(void)
{
	return system_state == SYSTEM_RUNNING;
}
EXPORT_SYMBOL(system_is_up);

int system_is_down(void)
{
	return system_state != SYSTEM_RUNNING;
}
```

### 6.2 禁止函数内多余空行（OS-STD-FMT-013）

**OS-STD-FMT-013**：函数体内禁止使用空行分隔"逻辑段落"——逻辑段落应当被抽出为独立 helper（详见 03 代码风格 §3）。函数内空行只允许出现在变量声明区与可执行语句区之间。

### 6.3 `MaxEmptyLinesToKeep: 1`（OS-STD-206）

**OS-STD-206**：连续空行最多保留 1 行；`.clang-format` 的 `MaxEmptyLinesToKeep: 1` 配置项强制执行；CI 会对超过 1 行的连续空行拒绝合并。注册表别名为 OS-STD-FMT-024（见 §0.2.1）。

---

## 7. 多语句同行禁令

### 7.1 禁止单行多语句（OS-STD-FMT-004）

**OS-STD-FMT-004**：禁止单行放置多条语句。该禁令源自一个朴素判断：把多语句塞在一行，等于在向读者隐藏什么。

```c
/* 错误：单行多语句，且无大括号 */
if (condition) do_this;

/* 正确 */
if (condition)
	do_this();
```

### 7.2 禁止用逗号逃避大括号（OS-STD-FMT-014）

**OS-STD-FMT-014**：禁止用逗号运算符把多条语句塞进 `if` 单语句体，逃避大括号约束。

```c
/* 错误：逗号逃避大括号 */
if (condition)
	do_this(), do_that();

/* 正确 */
if (condition) {
	do_this();
	do_that();
}
```

### 7.3 禁止单行多赋值（OS-STD-FMT-014）

**OS-STD-FMT-014**：禁止一行内多个赋值；每个赋值独占一行。agentrt-linux 内核代码风格简单——避免技巧性表达。

```c
/* 错误 */
a = 1; b = 2; c = 3;

/* 正确 */
a = 1;
b = 2;
c = 3;
```

---

## 8. 配置即代码（强制）

### 8.1 `.clang-format`（C/C++，OS-STD-FMT-025）

**OS-STD-FMT-025**：kernel / security / services 仓库根目录必须维护 `.clang-format`（基于 Linux 6.6 内核基线 689 行版本 + ForEachMacros 列表 + agentrt-linux 专属宏扩展）。

### 8.2 `.rustfmt.toml`（Rust，OS-STD-FMT-026）

**OS-STD-FMT-026**：cognition / cloudnative / 内核 Rust 模块仓库根目录必须维护 `.rustfmt.toml`，edition 2021，`hard_tabs = false`（4 空格），`max_width = 100`。

### 8.3 `.pyproject.toml`（Python，OS-STD-FMT-027）

**OS-STD-FMT-027**：Python 仓库（SDK / DevStation）必须维护 `.pyproject.toml`，配置 `black` `line-length = 100`、`isort` `profile = "black"`、`mypy` `strict = true`。

### 8.4 `.prettierrc`（TypeScript，OS-STD-FMT-028）

**OS-STD-FMT-028**：TypeScript 仓库必须维护 `.prettierrc`，`printWidth = 120`，`tabWidth = 2`，`singleQuote = true`，`trailingComma = "all"`。

### 8.5 CI 强制 `make format-check`（OS-STD-FMT-029）

**OS-STD-FMT-029**：CI 流水线第 4 层（CI 门禁，详见 06 卷 §4）必须执行 `make format-check`；任何格式违规直接拒绝 PR 合并，无人工豁免通道。修复方式唯一：本地运行 `make format` 自动修复后再提交。

> **SSoT 对齐**：原 80-clang-format-enforcement.md（→ 本文档 Part VI）已合并入本文档 §8/§9。完整 `.clang-format` YAML 文件、Makefile `format-check` / `format-diff` / `format-apply` 目标定义、Git pre-commit 钩子脚本、CI 流水线集成配置、以及与 `checkpatch.pl` 的协作流水线顺序，随代码实现补全。

---

## 9. clang-format 关键配置项

> **SSoT 对齐**：本节是规则到配置项的速查映射表（原 80-clang-format-enforcement.md（→ 本文档 Part VI）§1/§2 已合并于此）。完整 `.clang-format` YAML 文件与每个配置项的 Linux 6.6 内核基线 源码行号标注随代码实现补全。

agentrt-linux `.clang-format` 在 Linux 6.6 内核基线 689 行配置基础上扩展。下表列出关键项及其与第 1-7 章规则的对应关系：

| 配置项 | 值 | 对应规则 | 说明 |
|--------|-----|----------|------|
| `Language` | `Cpp` | — | 主语言 |
| `BasedOnStyle` | `LLVM` | — | 基线 |
| `ColumnLimit` | `80` | OS-STD-FMT-002 | C 行宽首选 80 |
| `IndentWidth` | `8` | OS-STD-FMT-001 | 缩进宽度 |
| `TabWidth` | `8` | OS-STD-FMT-001 | Tab 显示宽度 |
| `UseTab` | `Always` | OS-STD-FMT-001 | 强制 Tab |
| `ContinuationIndentWidth` | `8` | OS-STD-FMT-015 | 后继行缩进 |
| `BreakBeforeBraces` | `Custom` | OS-STD-FMT-003/OS-STD-FMT-009 | K&R 风格 |
| `BraceWrapping.AfterFunction` | `true` | OS-STD-FMT-009 | 函数开括号独占行 |
| `BraceWrapping.AfterControlStatement` | `false` | OS-STD-FMT-003 | 控制语句开括号行末 |
| `BraceWrapping.BeforeElse` | `false` | OS-STD-FMT-003 | else 紧贴闭括号 |
| `PointerAlignment` | `Right` | OS-STD-FMT-006 | `*` 贴名字 |
| `SpaceAfterCStyleCast` | `false` | OS-STD-FMT-005 | 括号内禁空格 |
| `SpaceBeforeParens` | `ControlStatements` | OS-STD-FMT-005 | 关键字后空格、`sizeof` 无 |
| `AlignAfterOpenBracket` | `Align` | OS-STD-FMT-015 | 后继行对齐 |
| `AllowShortIfStatementsOnASingleLine` | `false` | OS-STD-FMT-004 | 禁单行多语句 |
| `AllowShortBlocksOnASingleLine` | `false` | OS-STD-FMT-004 | 同上 |
| `AllowShortLoopsOnASingleLine` | `false` | OS-STD-FMT-004 | 同上 |
| `MaxEmptyLinesToKeep` | `1` | OS-STD-206 | 连续空行最多 1 |
| `BreakStringLiterals` | `false` | OS-STD-FMT-008 | 禁止断字符串 |
| `ForEachMacros` | 560 项 | — | 内核 for_each 宏列表，agentrt-linux 扩展 `airy_for_each_*` |

---

## 10. 五维原则映射

代码格式卷是 Airymax 五维正交 24 原则在视觉层面的最小切片——大多数原则在格式层不直接体现，但下列原则在此卷有具体落地点：

| 原则 | 落地点 | 对应规则 |
|------|--------|----------|
| **A-1 极简主义** | K&R 大括号最小化空行；连续空行最多 1 | OS-STD-FMT-003、OS-STD-FMT-012、OS-STD-206 |
| **A-2 细节关注** | 行尾禁止空白；指针声明贴名；括号内禁空格 | OS-STD-FMT-007、OS-STD-FMT-006、OS-STD-FMT-005 |
| **A-4 完美主义** | 配置即代码——格式规则必须可被工具 100% 强制 | OS-STD-FMT-030、OS-STD-FMT-029 |
| **E-4 跨平台一致性** | 多语言各自规范（C/Rust/Python/TS），无强制跨语言一致 | OS-STD-FMT-020~023 |
| **E-7 文档即代码** | `.clang-format` 等配置文件纳入版本控制，与代码同源审查 | OS-STD-FMT-025~028 |
| **K-2 接口契约化** | 用户可见字符串禁止断行——保证 grep 契约可执行 | OS-STD-FMT-008 |

---

## 11. 相关文档

- [工程标准 README](README.md)：工程标准规范主索引与总纲
- 本文档 Part I：命名、函数、注释、类型、错误处理
- 本文档 Part III：模块化、抽象层次、防御性、性能、工具链使用风格
- [05 开发流程 Part II](05-development-process.md)：7 层验证中格式检查所在的 Layer 3 / Layer 4
- [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md)：五维正交 24 原则完整定义
- Linux 6.6 内核基线 `Documentation/process/coding-style.rst` 与 `.clang-format`（689 行 + 560 ForEachMacros）为同源参考

---

## 12. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-06 | 初始版本（含 §1-§10 全部规则 + clang-format 配置表 + 五维映射） |
| 1.0.1 | 2026-07-12 | OS-KER/OS-STD 格式规则编号统一迁移为 OS-STD-FMT-NNN 子域（29 条）；消解 10 处跨文档编号冲突；ForEachMacros 扩展至包含 agentrt-linux 专属宏 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.

***

# Part III: 代码风格规范

## 0. 章节定位

本卷回答"代码该怎么思考"——非机械的工程风格决策。02 代码格式规定"长什么样"（Tab、80 列、K&R），01 代码规范规定"该怎么写"（命名、函数原型、错误码、类型）。本卷介于两者之间，规定"该怎么思考"：

- **何时抽象，何时不抽象**——反过度抽象原则（§1）
- **机制与策略的边界**——策略外移到用户空间或可插拔模块（§2）
- **函数与数据结构的结构风格**——helper 抽取、引用计数强制（§3、§4）
- **抽象层次的稳定性分级**——内核内部 API 不稳定、用户 ABI 稳定（§5）
- **防御性与性能的工程权衡**——fault injection、cache 影响是头等大事（§6、§7）
- **agentrt-linux 多语言协作的专属风格**——热/冷路径分层、FFI 边界（§8）
- **工具链使用风格**——gcc -W、sparse、lockdep、checkpatch（§9）

- **上游依赖**：04 工程思想定义"为什么"——双层稳定性、策略机制分离是设计哲学；本卷定义"如何落实"。
- **下游依赖**：06 工具链与自动化定义"工具怎么用"——checkpatch、sparse、lockdep 的具体配置；本卷定义"使用风格的判断标准"。

本卷所有强制规则均赋予 **OS-KER**（内核工程）/ **OS-STD**（标准）编号，从 OS-KER-118 接续 02 卷。

### 0.1 关键术语

| 术语 | 定义 |
|------|------|
| 机制 | 内核提供的能力（如调度器、内存分配、IPC 通道） |
| 策略 | 决定如何使用机制的选择（如调度顺序、内存回收阈值、IPC 路由） |
| 热路径 | 每秒被调用数万次以上的代码路径（调度、内存分配、IPC 收发） |
| 冷路径 | 偶发执行或非延迟敏感的代码路径（配置、日志格式化、统计上报） |
| 引用计数 | 多线程可见数据结构强制维护的"使用者数量"，归零方可释放 |
| 内核内部 API | 内核子系统之间调用的函数与数据结构，不保证稳定 |
| 用户空间 ABI | 通过系统调用暴露给用户程序的接口，永久稳定 |

---

## 1. 模块化设计风格

### 1.1 反过度抽象（OS-KER-118）

**OS-KER-118**：禁止"未来扩展"式抽象——若函数参数全为 0 的"未来扩展位"已存在 6 个月以上仍无第二使用者，应删除该抽象。预先抽象会冻结错误的设计——你无法对未来需求做出准确预测，但可以基于当前已知需求做出清晰决策。

```c
/* 错误：预留 5 个 ops 槽位给"未来可能的需求" */
struct airy_dispatcher_ops {
	void (*submit)(struct task *t);
	void (*cancel)(struct task *t);
	void (*future_op_1)(void);  /* 永远不会被实现 */
	void (*future_op_2)(void);
	void (*future_op_3)(void);
};

/* 正确：当前只用到的 ops */
struct airy_dispatcher_ops {
	void (*submit)(struct task *t);
	void (*cancel)(struct task *t);
};
```

### 1.2 反对隐藏硬件访问的多 OS 抽象层（OS-KER-119）

**OS-KER-119**：agentrt-linux 是 Linux 6.6 内核基线专属发行版，不维护"在 Linux / Windows / RTOS 之间切换"的硬件抽象层。若需跨平台一致性，由 agentrt 在用户空间承担（E-4 原则），agentrt-linux 内核态不背跨平台包袱。

### 1.3 代码重复应抽出到库（OS-STD-CODE-005）

**OS-STD-CODE-005**：当代码在 3 处以上重复时，应抽出到 `lib/` 或 `commons/` 子模块；当代码在 2 处重复时，可暂保留，但需在 PR 描述中标注"已知重复，待第三处出现时抽出"。该规则避免"过早抽象"与"放任重复"两极。

### 1.4 agentrt-linux 专属分层职责（OS-STD-STY-003）

**OS-STD-STY-003**：agentrt-linux 按 C-1 双系统协同原则分层选择实现语言：

| 层 | 子仓 | 主语言 | 选择理由 |
|----|------|--------|----------|
| 热路径 | kernel 调度 / 内存 / IPC | C | cache 友好、零运行时开销、与 Linux 6.6 内核基线对齐 |
| 慢路径 | security / memory 元数据 | Rust | 内存安全、防整类 CVE |
| Agent 路径 | cognition / cloudnative | Python / TS | 生态丰富、迭代快、与 agentrt 同源 |

---

## 2. 策略与机制分离

### 2.1 机制留在内核（OS-KER-120）

**OS-KER-120**：内核只提供"能力"——调度器提供"切换 CPU 上下文的能力"、IPC 提供消息传递的能力、内存管理提供分配释放的能力。任何"决策"——按什么顺序调度、消息如何路由、何时回收内存——都外移到用户空间或可插拔模块。

### 2.2 策略外移至用户空间或可插拔模块（OS-KER-121）

**OS-KER-121**：策略必须可在不重新编译内核的情况下替换。agentrt-linux 通过两类机制实现：用户态守护进程（K-3 服务隔离）+ eBPF / 方案 C-Prime 用户态调度器（K-4 可插拔策略）。

```c
/* 错误：策略硬编码进内核 */
static int airy_pick_next_cpu(struct task_struct *p)
{
	/* 硬编码"优先选 CPU 0"——这是策略，不是机制 */
	return 0;
}

/* 正确：内核只提供机制，策略由方案 C-Prime 用户态调度器提供 */
struct airy_sched_ops airy_agent_sched = {
	.enqueue		= agent_sched_enqueue,  /* 用户态调度器注入 */
	.dispatch		= agent_sched_dispatch,
	/* ...其余回调由用户态调度器决定 */
};
```

### 2.3 方案 C-Prime 用户态调度器作为极端范式（OS-KER-122）

**OS-KER-122**：kernel 的 `AIRY_SCHED_AGENT` 用户态调度器策略必须基于方案 C-Prime（SCHED_DEADLINE/SCHED_FIFO/EEVDF）实现——这是策略机制分离的最纯粹形态。用户态调度器在用户态编写、动态加载、热替换，而无需内核重新编译或重启。任何把调度策略硬编码进 `kernel/sched/` 的 PR 直接拒绝。

### 2.4 agentrt-linux 专属：K-4 可插拔策略原则映射

K-4 可插拔策略是 Airymax 五维正交 24 原则在调度、IPC 路由、安全裁决、记忆检索等"算法选择"上的统一约束。所有算法维度必须暴露策略接口，可在运行时替换。详见 [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md) §3.4。

---

## 3. 函数分解风格

### 3.1 大函数应分解为多个 helper（OS-KER-123）

**OS-KER-123**：函数长度一屏可读（≤80×24）；局部变量 ≤5-10 个；超过即应抽取 helper。但"长但简单"的函数（如长 switch-case）可保留——分解的判据是复杂度而非行数。

### 3.2 编译器 in-lining 比手工更高效（OS-KER-124）

**OS-KER-124**：相信编译器。若函数需要被 inline，使用 `static inline` 让编译器决定；编译器的 in-lining 策略（基于调用图、profile-guided 优化）远比人类判断精准。除非有实测证据，否则不写 `__always_inline`。

### 3.3 复杂条件编译不内嵌，抽出为独立 helper（OS-KER-125）

**OS-KER-125**：禁止在 `.c` 文件内用 `#ifdef` 切割函数体；条件编译应抽出到头文件中提供 stub，再由 `.c` 无条件调用。这样编译器仍可对 stub 调用做类型检查，逻辑可读性也得以保留。

```c
/* 错误：条件编译内嵌在函数体内 */
int airy_init(void)
{
#ifdef CONFIG_AIRYMAXOS_IPC
	ipc_init();
#endif
	return do_init();
}

/* 正确：抽出 stub 到头文件 */
/* airy_ipc.h */
#ifdef CONFIG_AIRYMAXOS_IPC
int ipc_init(void);
#else
static inline int ipc_init(void) { return 0; }
#endif

/* airy_init.c */
int airy_init(void)
{
	int ret = ipc_init();   /* 无条件调用 */
	if (ret)
		return ret;
	return do_init();
}
```

### 3.4 多用 `IS_ENABLED(CONFIG_XXX)` 而非 `#ifdef`（OS-KER-126）

**OS-KER-126**：在 `.c` 文件内优先使用 `if (IS_ENABLED(CONFIG_XXX))` 而非 `#ifdef`。编译器会做常量折叠，运行时零开销；同时让 C 编译器仍能对条件块内的代码做语法、类型、符号引用检查。

---

## 4. 数据结构设计风格

### 4.1 引用计数强制（OS-KER-127）

**OS-KER-127**：多线程可见的数据结构必须有引用计数。agentrt-linux 内核无 GC（垃圾回收），任何可被另一线程找到的数据结构都必须维护"使用者数量"——归零方可释放。agentrt-linux 的 E-3 资源确定性原则在此处落地。

```c
struct airy_task_desc {
	struct kref	refcount;   /* 强制：多线程可见 */
	u64		task_id;
	/* ... */
};

static void desc_release(struct kref *ref)
{
	struct airy_task_desc *desc =
		container_of(ref, struct airy_task_desc, refcount);
	kfree(desc);
}

/* 获取：在传递指针前 kref_get；释放：用完后 kref_put */
```

### 4.2 锁不替代引用计数（OS-KER-128）

**OS-KER-128**：锁保证数据结构**一致性**（读写串行化），引用计数保证数据结构**生命周期**（不被释放）。两者职责不同，不可混淆。常见 bug：持锁睡眠时另一线程释放了被引用的数据结构。

```c
/* 错误：锁替代引用计数——持锁睡眠导致 UAF */
mutex_lock(&tbl->lock);
desc = tbl_lookup(id);       /* 拿到 desc，无引用计数 */
schedule();                  /* 睡眠，另一线程从 tbl 删除并 kfree(desc) */
desc->field = 1;             /* UAF！ */
mutex_unlock(&tbl->lock);

/* 正确：先取引用，再睡眠 */
mutex_lock(&tbl->lock);
desc = tbl_lookup(id);
if (desc)
		kref_get(&desc->refcount);   /* 标记我正在用 */
mutex_unlock(&tbl->lock);
schedule();
desc->field = 1;
kref_put(&desc->refcount, desc_release);
```

### 4.3 灵活数组成员（OS-KER-129）

**OS-KER-129**：变长尾随数据使用 C99 灵活数组成员（`[]`），禁止"长度 1 数组"（`field[1]`）的旧 hack。配合 `kmalloc(struct_size(p, field, n), ...)` 自动做溢出检查。

```c
struct airy_ipc_batch {
	u64		count;
	struct airy_ipc_msg	msg[];   /* 灵活数组成员 */
};

p = kmalloc(struct_size(p, msg, n), GFP_KERNEL);
```

### 4.4 位域与紧凑布局（OS-KER-130）

**OS-KER-130**：多个布尔值合并为 `bitfield`；对齐敏感的结构体不用 `bool`，改用 `u8` 或位域。理由：`bool` 在不同架构上的大小与对齐变化，会破坏 cache line 布局。

```c
/* 错误：4 个 bool 占 4 字节且对齐不确定 */
struct flags_bad {
	bool	enabled;
	bool	ready;
	bool	pending;
	bool	in_flight;
};

/* 正确：1 字节位域 */
struct flags_good {
	u8	enabled:1;
	u8	ready:1;
	u8	pending:1;
	u8	in_flight:1;
};
```

---

## 5. 抽象层次

### 5.1 不维护稳定内核内部 API（OS-KER-131）

**OS-KER-131**：agentrt-linux 内核子系统之间的内部 API **不保证稳定**，这是 deliberate design decision（深思熟虑的设计决策）。允许持续重构是降低主线维护成本的核心机制——若每次重构都需兼顾二进制兼容，代码会迅速被历史包袱压垮。配套规则：API 改动者必须同时修复所有受影响代码（"you broke it, you fix it"）。

### 5.2 用户空间 ABI 必须稳定（OS-KER-132）

**OS-KER-132**：用户空间 ABI（系统调用、ioctl、文件系统接口、/proc、/sys、/dev 节点的语义）**永久稳定**。一旦接口导出至用户空间，必须永久支持；任何破坏性变更必须先经过弃用声明 + 6 个月宽限期，详见 04 工程思想 §6.2 的 4 层接口稳定性分级。

### 5.3 抽象只在必要时使用（OS-KER-133）

**OS-KER-133**：抽象是有成本的——一层间接调用、一份额外内存、一份认知负担。只有在"已经存在 3 处以上重复"或"已经能预见第二使用者"时才抽出抽象。空抽象比无抽象更糟——它会让后续维护者误以为存在多处使用者，不敢删除。

### 5.4 agentrt-linux 4 层接口稳定性分级（OS-STD-CODE-006）

**OS-STD-CODE-006**：agentrt-linux 在双层稳定性（内核内部 / 用户 ABI）基础上扩展为 4 层：

| 层 | 接口类型 | 稳定性 | 变更流程 |
|----|---------|--------|---------|
| L1 | Agent 应用 API（SDK + syscall） | 极稳定（类 Linux syscall） | RFC + 6 个月宽限 + 弃用声明 |
| L2 | AgentsIPC 协议（128B 头布局） | 中等稳定（版本化演进） | 季度评审 + 兼容性测试 |
| L3 | 内核子系统 API（内部 API） | 可重构（"you broke it, you fix it"） | 补丁序列修复所有调用点 |
| L4 | 内部实现细节 | 完全自由 | 无约束 |

---

## 6. 防御性编程风格

### 6.1 错误路径必须测试（OS-KER-134）

**OS-KER-134**：所有错误路径必须有对应测试用例。agentrt-linux 通过 fault injection 框架强制执行——`make kmalloc-fail` / `should_fail_alloc_page` 等机制可在运行时注入分配失败，验证错误路径是否真正可达且不破坏状态。任何未测试错误路径等同于未实现。

### 6.2 溢出检查（OS-KER-135）

**OS-KER-135**：所有用户可控长度参数必须经溢出检查后才能用于乘法。强制使用内核提供的安全算术：`struct_size()` / `array_size()` / `size_mul()` / `check_mul_overflow()`。

```c
/* 错误：无溢出检查 */
p = kmalloc(n * sizeof(*p), GFP_KERNEL);   /* n 来自用户态时可触发整数溢出 */

/* 正确：使用 struct_size 自动检查 */
p = kmalloc(struct_size(p, field, n), GFP_KERNEL);

/* 正确：使用 check_mul_overflow 显式检查 */
size_t bytes;
if (check_mul_overflow(n, sizeof(*p), &bytes))
	return -EOVERFLOW;
p = kmalloc(bytes, GFP_KERNEL);
```

### 6.3 不假设指针非空——分级标签释放（OS-KER-136）

**OS-KER-136**：错误路径禁止假设指针非空，应使用**分级标签按分配逆序释放**。每个标签只释放一个资源；标签按"分配顺序"逆序排列。该模式可避免"one err bug"——所有释放路径都可达，无一例外。

```c
/* 错误：单一 err 标签，部分路径下 foo 为 NULL */
err:
	kfree(foo->bar);   /* foo 可能未分配 */
	kfree(foo);
	return ret;

/* 正确：分级标签按分配逆序释放 */
err_free_bar:
	kfree(foo->bar);
err_free_foo:
	kfree(foo);
	return ret;
```

### 6.4 `WARN_ON_ONCE` 而非反复 `WARN`（OS-KER-137）

**OS-KER-137**：检测到不变式破坏时使用 `WARN_ON_ONCE(cond)`，禁止 `WARN_ON(cond)` 或 `WARN()`。`ONCE` 变体保证同一不变式破坏只警告一次——若反复触发，日志会被淹没。配套：禁止 `BUG()` / `BUG_ON()`，因为内核崩溃决策权属于用户而非开发者。

---

## 7. 性能优化风格

### 7.1 反对 inline 滥用（OS-KER-138）

**OS-KER-138**：超过 3 行的函数不应 `inline`；`static` 且仅被一处调用的函数编译器会自动 inline，无需手工标注。理由：`inline` 滥用导致内核体积膨胀，icache footprint 增大，整体性能下降。一个 pagecache miss 引发 5ms 磁盘寻道，远比省一个函数调用代价大。

```c
/* 错误：3 行以上还 inline，污染调用点 cache */
static inline int decode_complex_field(struct msg *m, u64 *out)
{
	u64 raw = get_unaligned(&m->field);
	*out = (raw >> 16) & 0xffffffffffff;
	return (raw & 0xffff) ? 0 : -EINVAL;
}

/* 正确：移除 inline，让编译器决定 */
static int decode_complex_field(struct msg *m, u64 *out)
{
	/* 同上 */
}
```

### 7.2 cache 影响是头等大事——空间即时间（OS-KER-139）

**OS-KER-139**：更大的程序运行更慢。代码与数据的空间局部性是性能首要因素——一个 cache line miss 等同上百个时钟周期。设计数据结构时优先考虑：是否可对齐到 cache line（64 字节）？是否可消除伪共享（不同 CPU 写入字段隔离到不同 cache line）？

```c
/* 错误：伪共享——两个 CPU 频繁写入相邻字段 */
struct counter_bad {
	u64	rx_packets;
	u64	tx_packets;   /* 与 rx_packets 同 cache line，互相 invalidade */
};

/* 正确：隔离到不同 cache line */
struct counter_good {
	struct {
		u64	rx_packets;
		u8	__pad[L1_CACHE_BYTES - sizeof(u64)];
	} ____cacheline_aligned;
	struct {
		u64	tx_packets;
		u8	__pad[L1_CACHE_BYTES - sizeof(u64)];
	} ____cacheline_aligned;
};
```

### 7.3 内存分配惯用法——`sizeof(*p)`（OS-KER-140）

**OS-KER-140**：内存分配强制使用 `sizeof(*p)` 而非 `sizeof(struct xxx)`，配合 `kcalloc` / `kmalloc_array` 处理数组。理由：当指针类型变更时，`sizeof(*p)` 自动跟随，不会留下"忘了改 sizeof"的 bug。

```c
/* 错误：类型变更时易遗漏 */
p = kmalloc(sizeof(struct airy_old_desc), GFP_KERNEL);

/* 正确：跟随指针类型 */
p = kmalloc(sizeof(*p), GFP_KERNEL);
arr = kcalloc(n, sizeof(*arr), GFP_KERNEL);
```

### 7.4 不滥用 printk——分级日志（OS-KER-141）

**OS-KER-141**：禁止单独使用 `printk`，必须用分级宏：设备相关用 `dev_err` / `dev_warn` / `dev_dbg`，设备无关用 `pr_err` / `pr_warn` / `pr_info` / `pr_debug`。`pr_debug` 在未定义 `DEBUG` 或未设置 `CONFIG_DYNAMIC_DEBUG` 时编译为空，零开销——这是热路径调试日志的正确方式。

### 7.5 锁设计必须前置（OS-KER-142）

**OS-KER-142**：锁设计必须在数据结构设计阶段就完成——不是"代码写完再加锁"。每条共享数据必须明确：哪个锁保护？锁的范围？锁内是否可睡眠？是否需引用计数配合？详见 04 工程思想 §3 与 80-testing 的 lockdep 强制检查。

---

## 8. agentrt-linux 专属风格

### 8.1 热路径 vs 冷路径分层（OS-KER-143）

**OS-KER-143**：热路径（kernel 调度、内存分配、IPC 收发）必须用 C 实现，且必须满足严格性能预算——单次调用 ≤1μs（64 核典型负载）。冷路径（配置解析、日志格式化、统计上报）可放宽 inline 限制，使用 Rust 实现以换取内存安全。

### 8.2 Rust 内核模块（OS-KER-144）

**OS-KER-144**：Rust 内核模块仅用于"安全敏感且非热路径"的子系统——典型如安全策略裁决（security LSM hook）、文件系统解析（路径遍历的字符串处理）、配置解析器。热路径禁用 Rust，避免 Rust 运行时与 panic 语义引入不可控延迟。

### 8.3 多语言协作风格——FFI 边界显式声明（OS-STD-CODE-007）

**OS-STD-CODE-007**：跨语言 FFI（Foreign Function Interface）边界必须显式声明——Rust ↔ C 边界标注 `extern "C"`、Python ↔ C 边界标注 `ctypes` 签名、TS ↔ Native 边界走 N-API。禁止隐式类型转换，所有跨边界数据必须经序列化或显式 layout 检查。

### 8.4 Agent 工作负载考量——Token 能效与记忆卷载（OS-STD-STY-004）

**OS-STD-STY-004**：agentrt-linux 设计须考虑 Agent 工作负载特性——Token 能效（每 token 的 CPU/J 预算）、记忆卷载（C-3，L1-L4 分层卷载模式）。详见 [40-dataflows/01-cognition-flow.md](../40-dataflows/01-cognition-flow.md) 与 [40-dataflows/02-memory-flow.md](../40-dataflows/02-memory-flow.md)。Agent 任务在调度器中应作为 `AIRY_SCHED_AGENT` 用户态调度器策略，与普通进程隔离调度，避免被普通负载饿死或反噬。

### 8.5 OLK-6.6 品味哲学——8 项隐性工程智慧（OS-STD-STY-005~012）

> **来源**：Linux 6.6 内核基线（openEuler OLK-6.6 标杆）源码深度提炼，`_review_0.9.0/01-olk66-code-taste-distillation.md`。以下 8 项隐性工程智慧不在 Linux 内核编码风格文档中显式书写，但在 OLK-6.6 源码中反复体现，是 Linus Torvalds 所称"代码品味（code taste）"的核心组成。agentrt-linux（AirymaxOS）内核态 C 代码必须遵循。

#### 8.5.1 信任读者——暴露而非封装（OS-STD-STY-005）

**OS-STD-STY-005**：内核代码应信任读者的工程素养——对于位编码、位掩码、状态标记等底层实现，应直接暴露而非过度封装。注释应解释"为什么这样设计"而非"这是什么"。

**OLK-6.6 源码证据**：`kernel/locking/mutex.c` L60-73——mutex `owner` 字段低 3 位编码了等待者状态（`MUTEX_FLAG_WAITERS`/`MUTEX_FLAG_HANDOFF`/`MUTEX_FLAG_PICKUP`），直接用 `#define` 暴露，注释解释设计理由而非隐藏在抽象层后。

```c
/* 好：信任读者，直接暴露位编码，注释解释设计 */
#define MUTEX_FLAG_WAITERS	0x01
#define MUTEX_FLAG_HANDOFF	0x02
#define MUTEX_FLAG_PICKUP	0x04
#define MUTEX_FLAGS		0x07

/* 坏：过度封装，隐藏底层实现 */
static inline bool mutex_has_waiters(struct mutex *lock)
{
	return (atomic_long_read(&lock->owner) & MUTEX_FLAG_WAITERS);
}
/* ↑ Linux 实际只在需要处直接检查，不为每个位单独封装 getter */
```

#### 8.5.2 短函数——一屏可读的强制约束（OS-STD-STY-006）

**OS-STD-STY-006**：函数应短到一屏可读（≤80 列 × 24 行）。如果一个函数需要滚动才能读完，它就已经过于复杂——应拆分为多个语义内聚的 helper。短函数的真正价值不在于行数限制本身，而在于它强制开发者将复杂逻辑分解为可命名的子操作。

**OLK-6.6 源码证据**：`kernel/locking/mutex.c` L845——大量使用 `static inline` 短函数作为 helper，每个函数只做一件事且命名精确。

```c
/* 好：短函数，语义内聚，命名精确 */
static inline int
__mutex_trylock_slowpath(struct mutex *lock)
{
	/* 清晰的逻辑：尝试获取，失败则返回 */
}

/* 坏：长函数，多职责混杂，需要滚动阅读 */
static int airy_ipc_handle_connection(struct airy_ipc_channel *chan,
				       struct airy_ipc_msg *msg,
				       struct airy_task *task)
{
	/* 200 行胶水逻辑：参数验证 + 权限检查 + 资源分配 + 消息分发
	 * + 错误处理 + 日志记录 + 统计更新 + ... 应拆分为 5-6 个 helper */
}
```

#### 8.5.3 简洁文件头——SPDX 优先（OS-STD-STY-007）

**OS-STD-STY-007**：源文件头必须以 SPDX-License-Identifier 开头（第 1 行），随后是简短的模块描述注释（≤20 行），然后直接进入 `#include`。禁止冗长的版权声明块、禁止作者列表、禁止版本历史——这些信息属于 git log 和 MAINTAINERS 文件。

**OLK-6.6 源码证据**：`kernel/locking/mutex.c` L1-20——SPDX 在第 1 行，4 行模块描述注释（含原始作者署名与设计文档引用），然后直接进入 `#include`。

```c
/* 好：简洁文件头 */
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kernel/locking/mutex.c
 *
 * Mutexes: blocking mutual exclusion locks
 *
 * Started by Ingo Molnar:
 */
#include <linux/mutex.h>
/* ... 直接进入代码 ... */

/* 坏：冗长的版权声明块 */
/*
 * Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
 *
 * [50 行版权声明]
 * [30 行作者列表]
 * [20 行版本历史]
 * [10 行法律声明]
 */
#include <linux/module.h>
```

#### 8.5.4 EXPORT_SYMBOL 紧跟定义——导出即声明（OS-STD-STY-008）

**OS-STD-STY-008**：`EXPORT_SYMBOL*` 宏必须紧跟在函数定义的右大括号之后，不得分离到文件末尾或单独的导出区。导出即声明——读者在看到函数定义时立即知道它是否被导出，无需翻页查找。

**OLK-6.6 源码证据**：`kernel/locking/mutex.c` L46-58——`__mutex_init` 函数定义结束后紧接 `EXPORT_SYMBOL(__mutex_init);`，无空行间隔。

```c
/* 好：EXPORT_SYMBOL 紧跟定义 */
void
__mutex_init(struct mutex *lock, const char *name, struct lock_class_key *key)
{
	atomic_long_set(&lock->owner, 0);
	raw_spin_lock_init(&lock->wait_lock);
	INIT_LIST_HEAD(&lock->wait_list);
	debug_mutex_init(lock, name, key);
}
EXPORT_SYMBOL(__mutex_init);

/* 坏：导出宏分离到文件末尾 */
void __mutex_init(...)
{
	/* ... */
}
/* ... 500 行其他代码 ... */

/* 文件末尾的导出区 */
EXPORT_SYMBOL(__mutex_init);
EXPORT_SYMBOL(mutex_lock);
EXPORT_SYMBOL(mutex_unlock);
```

#### 8.5.5 include 语义分组——禁止字母序自动排序（OS-STD-STY-009）

**OS-STD-STY-009**：`#include` 必须按**语义分组**排列（系统头 → 架构头 → 子系统头 → 本地头 → `#define CREATE_TRACE_POINTS` → trace events 头），**禁止使用工具自动按字母序排序**。字母序排序破坏了 include 的语义关联性——相关头文件应相邻排列以便读者理解依赖关系。`checkpatch.pl` 不检查 include 字母序。

**OLK-6.6 源码证据**：`kernel/locking/mutex.c` L21-35——include 按语义分组排列（`linux/mutex` → `linux/sched/*` → `linux/cgroup` → `linux/export` → `linux/spinlock` → `linux/interrupt` → `linux/debug` → `linux/osq`），非字母序。

```c
/* 好：语义分组——相关头文件相邻 */
#include <linux/mutex.h>
#include <linux/ww_mutex.h>
#include <linux/sched/signal.h>
#include <linux/sched/rt.h>
#include <linux/sched/wake_q.h>
#include <linux/sched/debug.h>
#include <linux/cgroup.h>
#include <linux/export.h>
#include <linux/spinlock.h>
#include <linux/interrupt.h>
#include <linux/debug_locks.h>
#include <linux/osq_lock.h>

#define CREATE_TRACE_POINTS
#include <trace/events/lock.h>

/* 坏：字母序自动排序——破坏语义关联性 */
#include <linux/cgroup.h>
#include <linux/debug_locks.h>
#include <linux/export.h>
#include <linux/interrupt.h>
#include <linux/mutex.h>
#include <linux/osq_lock.h>
#include <linux/sched/debug.h>
#include <linux/sched/rt.h>
#include <linux/sched/signal.h>
#include <linux/sched/wake_q.h>
#include <linux/spinlock.h>
#include <linux/ww_mutex.h>
```

#### 8.5.6 `__` 前缀——内部接口约定（OS-STD-STY-010）

**OS-STD-STY-010**：内核内部 helper 函数使用 `__` 双下划线前缀标注——这是 C 语言在缺乏访问控制的情况下表达"内部接口"的约定。`__foo()` 表示"这是 foo 的内部实现，外部不应直接调用"。外部接口（已 `EXPORT_SYMBOL`）使用无前缀版本作为入口，内部实现使用 `__` 前缀版本。

**OLK-6.6 源码证据**：`kernel/locking/mutex.c` L76-80——`__mutex_owner` 标注为 "Internal helper function; C doesn't allow us to hide it :/ DO NOT USE (outside of mutex code)."

```c
/* 好：__ 前缀标注内部接口 */
static inline struct task_struct *__mutex_owner(struct mutex *lock)
{
	return (struct task_struct *)
		(atomic_long_read(&lock->owner) & ~MUTEX_FLAGS);
}

/* 外部入口函数调用 __ 前缀内部实现 */
int mutex_lock_interruptible(struct mutex *lock)
{
	/* ... */
	owner = __mutex_owner(lock);
	/* ... */
}

/* 坏：无前缀区分，内外接口混淆 */
static inline struct task_struct *mutex_owner(struct mutex *lock)
{
	/* 外部代码可能误调用此"内部"函数 */
}
```

#### 8.5.7 `__read_mostly`——缓存友好的数据布局（OS-STD-STY-011）

**OS-STD-STY-011**：频繁读取但很少修改的全局变量必须使用 `__read_mostly` 注解。`__read_mostly` 将变量放入 `.data..read_mostly` 段，与频繁修改的数据分离，减少 false sharing 导致的缓存行失效。agentrt-linux 的 Agent 注册表、调度器配置等 mostly-read 数据应使用此注解。

**OLK-6.6 源码证据**：`kernel/sched/core.c` L168/178/7786——`sysctl_resched_latency_warn_ms`、`scheduler_running`、`sched_smp_initialized` 等全局变量使用 `__read_mostly` 注解。

```c
/* 好：mostly-read 数据使用 __read_mostly */
__read_mostly int scheduler_running;
__read_mostly bool sched_smp_initialized;
__read_mostly int sysctl_resched_latency_warn_ms = 100;

/* agentrt-linux 应用 */
static __read_mostly struct airy_sched_class
				*airy_sched_classes[MAX_SCHED_CLASSES];
static __read_mostly atomic_t airy_agent_count = ATOMIC_INIT(0);

/* 坏：频繁修改的数据误用 __read_mostly */
__read_mostly atomic_t airy_current_token_usage;  /* 频繁更新，不应使用 */
```

#### 8.5.8 `__randomize_layout`——结构体布局随机化（OS-STD-STY-012）

**OS-STD-STY-012**：安全敏感的结构体（含函数指针表、含权限字段、含内核地址）必须使用 `__randomize_layout` 注解。`__randomize_layout` 在编译时随机化结构体字段顺序，防止攻击者通过预测结构布局实施函数指针劫持或信息泄露。agentrt-linux 的 `airy_task_struct`、`airy_security_ops` 等安全敏感结构体应使用此注解（需 `CONFIG_RANDSTRUCT` 启用）。

**OLK-6.6 源码证据**：`include/linux/sched.h` L658——`struct task_struct` 使用 `__randomize_layout` 注解。

```c
/* 好：安全敏感结构体使用 __randomize_layout */
struct task_struct {
	/* ... 大量字段 ... */
} __randomize_layout;

/* agentrt-linux 应用——安全敏感结构体 */
struct airy_task_struct {
	void (*sched_class)(struct airy_task_struct *);
	u32 capability_mask;
	void *security_blob;
	/* ... */
} __randomize_layout;

struct airy_security_ops {
	int (*inode_permission)(struct inode *, int);
	int (*file_open)(struct file *, const struct cred *);
	/* ... */
} __randomize_layout;

/* 坏：无随机化，攻击者可预测结构布局 */
struct airy_task_struct {
	/* 字段顺序固定，攻击者可通过偏移量定位敏感字段 */
};
```

#### 8.5.9 品味哲学映射表

| 编号 | 隐性智慧 | OLK-6.6 源码证据 | 工程价值 |
|------|---------|-----------------|---------|
| OS-STD-STY-005 | 信任读者，暴露而非封装 | `mutex.c` L60-73 owner 位编码 | 减少封装层，直接暴露底层实现 |
| OS-STD-STY-006 | 短函数，一屏可读 | `mutex.c` L845 短 helper | 强制逻辑分解，提升可测试性 |
| OS-STD-STY-007 | 简洁文件头，SPDX 优先 | `mutex.c` L1-20 | 减少冗余，代码即文档 |
| OS-STD-STY-008 | EXPORT_SYMBOL 紧跟定义 | `mutex.c` L46-58 | 导出即声明，无需翻页查找 |
| OS-STD-STY-009 | include 语义分组，禁止字母序 | `mutex.c` L21-35 | 保留语义关联性，checkpatch 不检查字母序 |
| OS-STD-STY-010 | `__` 前缀标注内部接口 | `mutex.c` L80 `__mutex_owner` | C 无访问控制下的内部接口约定 |
| OS-STD-STY-011 | `__read_mostly` 缓存优化 | `core.c` L168/178/7786 | 减少 false sharing，缓存友好 |
| OS-STD-STY-012 | `__randomize_layout` 安全加固 | `sched.h` L658 `task_struct` | 防止结构布局预测攻击 |

---

## 9. 工具链使用风格

### 9.1 gcc -W（OS-KER-145）

**OS-KER-145**：编译必须开启 `-Wall -Wextra -Werror`；新增警告项必须立即修复或显式 `// NOLINT: <reason>` 标注。零警告是 agentrt-linux 不可妥协的底线。

### 9.2 sparse——类型检查（OS-KER-146）

**OS-KER-146**：内核 C 代码必须通过 `make C=1`（sparse 检查）。sparse 检测 `__user` / `__kernel` / `__iomem` 等 address space 注解违反、endianness 错误、锁注解不一致——这些 gcc 不检查。新增涉及用户空间指针的代码必须含 `__user` 注解。

### 9.3 lockdep——锁依赖检查（OS-KER-147）

**OS-KER-147**：开发与 CI 配置 `CONFIG_LOCKDEP=y` + `CONFIG_PROVE_LOCKING=y`。lockdep 在运行时构建锁依赖图，检测死锁、锁序违反、加锁上下文不一致。任何 lockdep 警告视为 PR 阻断。

### 9.4 fault injection（OS-KER-148）

**OS-KER-148**：错误路径测试必须使用 fault injection 框架——`CONFIG_FAULT_INJECTION=y` + `CONFIG_FAILSLAB` / `CONFIG_FAIL_PAGE_ALLOC` / `CONFIG_FAIL_MAKE_REQUEST`。每个错误路径必须在 CI 中至少运行一次"分配全程失败"的测试用例。

### 9.5 checkpatch.pl（OS-STD-TOOL-160）

**OS-STD-TOOL-160**：每个 PR 提交前必须通过 `scripts/checkpatch.pl --strict`，无 ERROR / WARNING。checkpatch 检查 Linux 6.6 内核基线规则 + agentrt-linux 扩展规则（详见 06 卷 §3.2）。

### 9.6 多配置构建（OS-STD-TOOL-161）

**OS-STD-TOOL-161**：每个 PR 必须通过至少 3 种配置构建：`defconfig`（默认）、`allnoconfig`（最小）、`allmodconfig`（最大）。三者皆绿方可合并——`allnoconfig` 暴露 `#ifdef` 错配、`allmodconfig` 暴露符号冲突、`defconfig` 验证默认路径。

---

## 10. 五维原则映射

代码风格卷是 Airymax 五维正交 24 原则在工程判断层面的核心切片——大多数原则在此卷都有具体落地点：

| 原则 | 落地点 | 对应规则 |
|------|--------|----------|
| **A-1 极简主义** | 反过度抽象；反 inline 滥用；抽象只在必要时使用；信任读者暴露而非封装 | OS-KER-118、OS-KER-133、OS-KER-138、OS-STD-STY-005 |
| **A-2 细节关注** | 分级标签按分配逆序释放；`sizeof(*p)` 跟随类型；短函数；简洁文件头；EXPORT_SYMBOL 紧跟；include 语义分组；`__` 前缀内部接口 | OS-KER-136、OS-KER-140、OS-STD-STY-006~010 |
| **A-4 完美主义** | 零警告编译；多配置构建（allnoconfig + allmodconfig）；短函数一屏可读 | OS-KER-145、OS-STD-TOOL-161、OS-STD-STY-006 |
| **E-3 资源确定性** | 引用计数强制；锁不替代引用计数；`__read_mostly` 缓存友好数据布局 | OS-KER-127、OS-KER-128、OS-STD-STY-011 |
| **E-1 安全内生** | Rust 模块用于安全敏感路径；sparse 类型检查；`__randomize_layout` 结构体布局随机化 | OS-KER-144、OS-KER-146、OS-STD-STY-012 |
| **E-8 可测试性** | 错误路径必须测试；fault injection 强制 | OS-KER-134、OS-KER-148 |
| **K-2 接口契约化** | 4 层接口稳定性分级；用户 ABI 永久稳定 | OS-KER-132、OS-STD-CODE-006 |
| **K-3 服务隔离** | 策略外移至用户态守护进程 | OS-KER-121 |
| **K-4 可插拔策略** | 策略机制分离；方案 C-Prime 用户态调度范式 | OS-KER-120、OS-KER-122 |
| **C-1 双系统协同** | 热路径 C / 慢路径 Rust / Agent Python+TS 分层 | OS-STD-STY-003、OS-KER-143 |
| **C-3 记忆卷载** | Agent 工作负载考量，Token 能效与 L1-L4 卷载 | OS-STD-STY-004 |

---

## 11. 相关文档

- [工程标准 README](README.md)：工程标准规范主索引与总纲
- 本文档 Part I：命名、函数、注释、类型、错误处理
- 本文档 Part II：缩进、行宽、大括号、空格、clang-format
- [04 工程思想](04-engineering-philosophy.md)：双层稳定性哲学、策略机制分离、渐进式
- [05 开发流程 Part II](05-development-process.md)：checkpatch、sparse、lockdep 的具体配置
- [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md)：五维正交 24 原则完整定义
- [20-modules/01-kernel.md](../20-modules/01-kernel.md)：kernel 模块设计
- Linux 6.6 内核基线 `Documentation/process/coding-style.rst` §6~§22 与 `stable-api-nonsense.rst` 为同源参考

---

## 12. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-06 | 初始版本（含 §1-§10 全部规则 + 11 条五维映射 + agentrt-linux 专属风格） |
| 1.0.1 | TBD | 与代码实现同步验证；Agent 工作负载 Token 能效基准确立 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.

***

# Part IV: Checkpatch 规则映射

## 0. 文档说明

### 0.1 目的与范围

本文件研究 Linux 6.6 内核基线 `scripts/checkpatch.pl`（239KB，约 7800 行）中定义的所有补丁检查规则，提取 ≥30 条 ERROR/WARNING 类型规则，建立到 agentrt-linux 规则编号体系的映射。

每条规则包含以下字段：

- **checkpatch 规则名**：`scripts/checkpatch.pl` 中的 `ERROR("RULE_NAME")` / `WARN("RULE_NAME")` / `CHK("RULE_NAME")` 调用标识
- **级别**：ERROR / WARNING / CHECK
- **检查内容**：规则触发条件的中文化描述
- **Linux 6.6 内核基线 源码行号**：`scripts/checkpatch.pl:行号` 格式
- **agentrt-linux 规则编号**：OS-STD-CODE-NNN
- **示例**：触发该规则的代码片段

> **交叉引用**：本 Part 是 checkpatch 规则到 agentrt-linux 规则编号的逐条映射表（规则映射层）。checkpatch 在 7 层验证体系中的框架定位（第 2 层静态分析 + 第 3 层预提交 + 第 4 层 CI 门禁）、三级报告策略、CI 调用方式与退出码处理，详见 [05-development-process.md](05-development-process.md) Part II §5.1（checkpatch.pl）与 §1.2-§1.4（7 层验证体系第 2-4 层）。.clang-format 配置项与格式规则定义，详见 本文档 Part II §8/§9（原 80-clang-format-enforcement.md 已合并入本文档 Part II/Part VI；clang-format 与 checkpatch 的协作流水线顺序随代码实现补全）。

### 0.2 checkpatch 规则分级约定

agentrt-linux 对 checkpatch 规则的分级处理：

| checkpatch 级别 | agentrt-linux 处理策略 |
|----------------|---------------------|
| **ERROR** | CI 强制门禁，0 容忍，违反即拒合并 |
| **WARNING** | CI 强制门禁，0 容忍（升级为 ERROR），违反即拒合并 |
| **CHECK** | Code Review 提示项，不阻塞合并但建议修复 |

### 0.3 规则编号体系

agentrt-linux 编码规则编号采用 `OS-STD-CODE-NNN` 三段式：

- `OS`：归属 agentrt-linux（OS 发行版）
- `STD-CODE`：编码标准 - 代码规则子类
- `NNN`：三位数字序号（001-999）

---

## 1. 字符串与内存安全规则

### 规则 1：STRCPY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `STRCPY` |
| **级别** | WARNING |
| **检查内容** | 检测 `strcpy()` 调用，建议改用 `strscpy`。`strcpy` 无边界检查，可导致线性溢出。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7029-7031` |
| **agentrt-linux 规则编号** | OS-STD-CODE-010 |
| **示例** | `strcpy(dst, src);` → 触发 "Prefer strscpy over strcpy" |

### 规则 2：STRLCPY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `STRLCPY` |
| **级别** | WARNING |
| **检查内容** | 检测 `strlcpy()` 调用，建议改用 `strscpy`。`strlcpy` 多余读源且可能溢出。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7035-7037` |
| **agentrt-linux 规则编号** | OS-STD-CODE-010 |
| **示例** | `strlcpy(dst, src, len);` → 触发 "Prefer strscpy over strlcpy" |

### 规则 3：STRNCPY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `STRNCPY` |
| **级别** | WARNING |
| **检查内容** | 检测 `strncpy()` 调用，建议改用 `strscpy`/`strscpy_pad`/`__nonstring`。`strncpy` 不保证 NUL 终止。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7041-7043` |
| **agentrt-linux 规则编号** | OS-STD-CODE-010 |
| **示例** | `strncpy(dst, src, n);` → 触发 "Prefer strscpy, strscpy_pad, or __nonstring" |

### 规则 4：MEMSET

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MEMSET` |
| **级别** | ERROR / WARNING |
| **检查内容** | 检测 `memset(p, 0, sizeof(*p))` 模式，建议改用 `kzalloc` 或显式零初始化。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6978-6981` |
| **agentrt-linux 规则编号** | OS-STD-CODE-018 |
| **示例** | `p = kmalloc(...); memset(p, 0, sizeof(*p));` → 建议用 `kzalloc` |

### 规则 5：ALLOC_WITH_MULTIPLY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ALLOC_WITH_MULTIPLY` |
| **级别** | WARNING |
| **检查内容** | 检测 `kmalloc(n * sizeof(...))` 模式，建议改用 `kmalloc_array`/`kcalloc`（含溢出检查）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7255` |
| **agentrt-linux 规则编号** | OS-STD-CODE-012 |
| **示例** | `kmalloc(n * sizeof(struct x), GFP)` → 建议用 `kcalloc(n, sizeof(*p), GFP)` |

### 规则 6：ALLOC_SIZEOF_STRUCT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ALLOC_SIZEOF_STRUCT` |
| **级别** | CHECK |
| **检查内容** | 检测 `kmalloc(sizeof(struct xxx), ...)` 模式，建议改用 `sizeof(*p)`。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7229` |
| **agentrt-linux 规则编号** | OS-STD-CODE-012 |
| **示例** | `kmalloc(struct foo, GFP)` → 建议用 `kmalloc(sizeof(*p), GFP)` |

### 规则 7：KREALLOC_ARG_REUSE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `KREALLOC_ARG_REUSE` |
| **级别** | WARNING |
| **检查内容** | 检测 `p = krealloc(p, ...)` 模式，重用参数 p 在失败时会泄漏原 p。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7268` |
| **agentrt-linux 规则编号** | OS-STD-CODE-019 |
| **示例** | `p = krealloc(p, n, GFP);` → 应用临时变量 |

---

## 2. 缩进与空格规则

### 规则 8：CODE_INDENT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `CODE_INDENT` |
| **级别** | ERROR |
| **检查内容** | 检测用空格而非 Tab 进行代码缩进。代码必须用 Tab 缩进。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:3916` |
| **agentrt-linux 规则编号** | OS-STD-FMT-001 |
| **示例** | `    foo();`（4 空格） → 触发 "code indent should use tabs where possible" |

### 规则 9：SPACE_BEFORE_TAB

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPACE_BEFORE_TAB` |
| **级别** | WARNING |
| **检查内容** | 检测 Tab 前有空格的混合缩进（如 `\t  \t`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:3926` |
| **agentrt-linux 规则编号** | OS-STD-FMT-001 |
| **示例** | `  \tfoo` → 触发 "please, no space before tabs" |

### 规则 10：TABSTOP

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TABSTOP` |
| **级别** | WARNING |
| **检查内容** | 检测行内出现 Tab 字符（用于代码对齐位置之外）。Tab 仅用于行首缩进。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:3967` |
| **agentrt-linux 规则编号** | OS-STD-FMT-001 |
| **示例** | `printk("a\tb");` 中 Tab 用于对齐 → 触发警告 |

### 规则 11：TRAILING_WHITESPACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TRAILING_WHITESPACE` |
| **级别** | ERROR |
| **检查内容** | 检测行尾空白字符（空格或 Tab）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:3599` |
| **agentrt-linux 规则编号** | OS-STD-FMT-007 |
| **示例** | `foo();  ` → 触发 "trailing whitespace" |

### 规则 12：SPACING

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPACING` |
| **级别** | ERROR / WARNING |
| **检查内容** | 检测关键字后缺空格、`sizeof` 后多余空格、逗号后缺空格、`(`后多余空格等多种空格规范。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4967,5014,5192-5348,5444-5491,5665-5708`（多处） |
| **agentrt-linux 规则编号** | OS-STD-CODE-021 |
| **示例** | `if(x)` → 触发 "space required after the 'if' keyword" |

### 规则 13：LEADING_SPACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `LEADING_SPACE` |
| **级别** | WARNING |
| **检查内容** | 检测文件首行有空格而非 Tab 缩进。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4161` |
| **agentrt-linux 规则编号** | OS-STD-FMT-001 |
| **示例** | 文件首行 `    #include` → 触发警告 |

---

## 3. 大括号与控制流规则

### 规则 14：OPEN_BRACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `OPEN_BRACE` |
| **级别** | ERROR |
| **检查内容** | 检测大括号位置错误：非函数块（if/for/while/switch）的 `{` 应在行尾，函数的 `{` 应在新行首。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4361,4568,4931,4950,7204` |
| **agentrt-linux 规则编号** | OS-STD-CODE-022 |
| **示例** | `if (x)\n{` → 触发 "open brace '{' following if go on the same line" |

### 规则 15：BRACKET_SPACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `BRACKET_SPACE` |
| **级别** | ERROR |
| **检查内容** | 检测数组声明 `[` 前多余空格（如 `char buf [8]`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:5054` |
| **agentrt-linux 规则编号** | OS-STD-CODE-021 |
| **示例** | `char buf [8]` → 触发 "space prohibited before open square bracket" |

### 规则 16：SWITCH_CASE_INDENT_LEVEL

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SWITCH_CASE_INDENT_LEVEL` |
| **级别** | ERROR |
| **检查内容** | 检测 `switch` 与 `case` 标签未对齐到同一列（应同列而非 case 多缩进一级）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4326` |
| **agentrt-linux 规则编号** | OS-STD-CODE-023 |
| **示例** | `switch` 后 `case` 多缩进一级 → 触发错误 |

### 规则 17：ASSIGN_IN_IF

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ASSIGN_IN_IF` |
| **级别** | ERROR |
| **检查内容** | 检测 `if ((a = b))` 在 if 条件中赋值的反模式。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:5708` |
| **agentrt-linux 规则编号** | OS-STD-CODE-024 |
| **示例** | `if ((ret = foo()))` → 建议拆分为 `ret = foo(); if (ret)` |

### 规则 18：TRAILING_STATEMENTS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TRAILING_STATEMENTS` |
| **级别** | ERROR |
| **检查内容** | 检测 `if/while` 后同行跟语句（如 `if (x) foo();`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:5754,5792,5798,5809` |
| **agentrt-linux 规则编号** | OS-STD-CODE-025 |
| **示例** | `if (x) do_this();` → 触发 "trailing statements should be on next line" |

### 规则 19：DEFAULT_NO_BREAK

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DEFAULT_NO_BREAK` |
| **级别** | WARNING |
| **检查内容** | 检测 `switch` 的 `case` 末尾缺少 `break`/`fallthrough`/`return`/`goto`/`continue`。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7344` |
| **agentrt-linux 规则编号** | OS-STD-CODE-015 |
| **示例** | `case A: foo(); case B:` → 触发 "break/return/goto/continue/fallthrough expected" |

### 规则 20：ELSE_AFTER_BRACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ELSE_AFTER_BRACE` |
| **级别** | ERROR |
| **检查内容** | 检测 `}` 与 `else` 未同行（`}\nelse` 应为 `} else`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:5817` |
| **agentrt-linux 规则编号** | OS-STD-CODE-022 |
| **示例** | `}\nelse` → 触发 "else should follow close brace" |

### 规则 21：WHILE_AFTER_BRACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `WHILE_AFTER_BRACE` |
| **级别** | ERROR |
| **检查内容** | 检测 `do { ... }\n while` 中 `while` 未与 `}` 同行（应为 `} while`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:5843` |
| **agentrt-linux 规则编号** | OS-STD-CODE-022 |
| **示例** | `}\nwhile (x);` → 触发 "while should follow close brace" |

---

## 4. 类型与声明规则

### 规则 22：NEW_TYPEDEFS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `NEW_TYPEDEFS` |
| **级别** | WARNING |
| **检查内容** | 检测新增 `typedef`，禁止为结构体/指针新增 typedef。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4783-4790` |
| **agentrt-linux 规则编号** | OS-STD-CODE-020 |
| **示例** | `typedef struct { ... } foo_t;` → 触发 "do not add new typedefs" |

### 规则 23：POINTER_LOCATION

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `POINTER_LOCATION` |
| **级别** | ERROR |
| **检查内容** | 检测 `*` 紧贴类型而非变量名（如 `char* p` 应为 `char *p`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4808,4835` |
| **agentrt-linux 规则编号** | OS-STD-CODE-026 |
| **示例** | `char* p` → 触发 "\"foo* bar\" should be \"foo *bar\"" |

### 规则 24：GLOBAL_INITIALISERS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `GLOBAL_INITIALISERS` |
| **级别** | ERROR |
| **检查内容** | 检测全局变量使用位置初始化器而非指定初始化器。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4662` |
| **agentrt-linux 规则编号** | OS-STD-CODE-014 |
| **示例** | `int arr[] = {1, 2, 3};` → 建议用 `{ [0] = 1, ... }` |

### 规则 25：INITIALISED_STATIC

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `INITIALISED_STATIC` |
| **级别** | ERROR |
| **检查内容** | 检测 `static` 变量初始化为 0/NULL（BSS 段已默认零初始化，无需显式）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4670` |
| **agentrt-linux 规则编号** | OS-STD-CODE-027 |
| **示例** | `static int x = 0;` → 触发 "do not initialise statics to 0" |

### 规则 26：AVOID_EXTERNS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `AVOID_EXTERNS` |
| **级别** | WARNING / CHECK |
| **检查内容** | 检测函数原型使用 `extern` 关键字（多余且使行变长）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7121,7141,7160,7168` |
| **agentrt-linux 规则编号** | OS-STD-CODE-055 |
| **示例** | `extern int foo(void);` → 触发 "function prototypes should not be extern" |

### 规则 27：FUNCTION_ARGUMENTS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `FUNCTION_ARGUMENTS` |
| **级别** | WARNING |
| **检查内容** | 检测函数原型参数无名称（参数应含描述性名称以助可读性）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7146,7180` |
| **agentrt-linux 规则编号** | OS-STD-CODE-056 |
| **示例** | `int foo(int, char*);` → 建议用 `int foo(int count, char *name)` |

### 规则 28：VOLATILE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `VOLATILE` |
| **级别** | WARNING |
| **检查内容** | 检测 `volatile` 使用，建议改用 `READ_ONCE`/`WRITE_ONCE`/屏障。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6275` |
| **agentrt-linux 规则编号** | OS-STD-CODE-057 |
| **示例** | `volatile int flag;` → 触发 "Use of volatile is usually wrong" |

---

## 5. 宏与预处理规则

### 规则 29：MULTISTATEMENT_MACRO_USE_DO_WHILE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MULTISTATEMENT_MACRO_USE_DO_WHILE` |
| **级别** | ERROR |
| **检查内容** | 检测多语句宏未用 `do { ... } while (0)` 包裹（避免 if 嵌套错乱）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6016,6019,6022` |
| **agentrt-linux 规则编号** | OS-STD-CODE-058 |
| **示例** | `#define FOO(a) a++; b--;` → 建议用 `do { a++; b--; } while (0)` |

### 规则 30：COMPLEX_MACRO

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `COMPLEX_MACRO` |
| **级别** | ERROR |
| **检查内容** | 检测宏内运算未加括号（参数与表达式都需括号保护优先级）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6022` |
| **agentrt-linux 规则编号** | OS-STD-CODE-032 |
| **示例** | `#define X(a) a + 1` → 建议用 `#define X(a) ((a) + 1)` |

### 规则 31：MACRO_WITH_FLOW_CONTROL

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MACRO_WITH_FLOW_CONTROL` |
| **级别** | WARNING |
| **检查内容** | 检测宏内含 `return`/`break`/`goto` 等控制流（破坏调用方控制流预期）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6073` |
| **agentrt-linux 规则编号** | OS-STD-CODE-033 |
| **示例** | `#define FOO(x) if (x < 0) return -1;` → 建议用 inline 函数 |

### 规则 32：TRAILING_SEMICOLON

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TRAILING_SEMICOLON` |
| **级别** | WARNING |
| **检查内容** | 检测宏定义末尾多余分号（导致 `if (cond) MACRO;` 错乱）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6131` |
| **agentrt-linux 规则编号** | OS-STD-CODE-034 |
| **示例** | `#define FOO(x) do_bar(x);` → 建议去掉末尾分号 |

### 规则 33：MACRO_ARG_PRECEDENCE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MACRO_ARG_PRECEDENCE` |
| **级别** | CHECK |
| **检查内容** | 检测宏参数在表达式中未加括号（优先级风险）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6062` |
| **agentrt-linux 规则编号** | OS-STD-CODE-035 |
| **示例** | `#define X(a) a * 2` → 建议用 `(a) * 2` |

---

## 6. 注释与文档规则

### 规则 34：BLOCK_COMMENT_STYLE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `BLOCK_COMMENT_STYLE` |
| **级别** | WARNING |
| **检查内容** | 检测多行块注释样式不符（首行应几乎为空，左侧星号列对齐）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4038,4047,4070` |
| **agentrt-linux 规则编号** | OS-STD-CODE-036 |
| **示例** | `/* line1\n * line2 */` → 建议用 `/*\n * line1\n */` |

### 规则 35：NETWORKING_BLOCK_COMMENT_STYLE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `NETWORKING_BLOCK_COMMENT_STYLE` |
| **级别** | WARNING |
| **检查内容** | `net/` 与 `drivers/net/` 下文件的多行注释首行不应为空（特殊风格）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4028` |
| **agentrt-linux 规则编号** | OS-STD-CODE-059 |
| **示例** | net/ 下 `/*\n * foo */` → 建议用 `/* foo\n */` |

### 规则 36：C99_COMMENTS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `C99_COMMENTS` |
| **级别** | ERROR |
| **检查内容** | 检测 `//` 单行注释（部分场景禁止，建议用 `/* */`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4601` |
| **agentrt-linux 规则编号** | OS-STD-CODE-037 |
| **示例** | UAPI 头文件中 `// comment` → 触发 "do not use C99 // comments" |

---

## 7. 头文件与许可证规则

### 规则 37：MALFORMED_INCLUDE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MALFORMED_INCLUDE` |
| **级别** | ERROR |
| **检查内容** | 检测 `#include` 语法错误（如 `#include <foo.h` 缺 `>`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4590` |
| **agentrt-linux 规则编号** | OS-STD-CODE-038 |
| **示例** | `#include <foo.h` → 触发 "malformed #include" |

### 规则 38：UAPI_INCLUDE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `UAPI_INCLUDE` |
| **级别** | ERROR |
| **检查内容** | 检测内核内部代码包含 `include/uapi/` 头文件（应包含 `include/linux/`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4594` |
| **agentrt-linux 规则编号** | OS-STD-CODE-039 |
| **示例** | 内核代码 `#include <uapi/linux/foo.h>` → 建议用 `<linux/foo.h>` |

### 规则 39：SPDX_LICENSE_TAG

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPDX_LICENSE_TAG` |
| **级别** | WARNING |
| **检查内容** | 检测 SPDX 许可证标识缺失、格式错误或不被 LICENSES/ 支持。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:3776,3782,3787,3803,3812,3823` |
| **agentrt-linux 规则编号** | OS-STD-CODE-040 |
| **示例** | `// SPDX-License-Identifier: GPL-2.0` 在 .c 文件中 → 建议用 `/* */` 注释 |

---

## 8. 杂项与质量规则

### 规则 40：DEPRECATED_API

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DEPRECATED_API` |
| **级别** | ERROR / WARNING |
| **检查内容** | 检测调用 `deprecated_apis` 表中的废弃 API（如 `kmap`/`synchronize_sched`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6456-6458,7424` |
| **agentrt-linux 规则编号** | OS-STD-CODE-041 |
| **示例** | `kmap(page)` → 建议用 `kmap_local_page(page)` |

### 规则 41：DEEP_INDENTATION

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DEEP_INDENTATION` |
| **级别** | WARNING |
| **检查内容** | 检测续行过度缩进（>6 级）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4339` |
| **agentrt-linux 规则编号** | OS-STD-CODE-042 |
| **示例** | 续行缩进 8 级 → 触发警告 |

### 规则 42：SUSPECT_CODE_INDENT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SUSPECT_CODE_INDENT` |
| **级别** | WARNING |
| **检查内容** | 检测 if/else 块内代码缩进与条件不一致（混合 Tab/空格的典型症状）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:4474` |
| **agentrt-linux 规则编号** | OS-STD-FMT-001 |
| **示例** | `if (x)\n        foo();`（空格而非 Tab） → 触发 "suspect code indent" |

### 规则 43：RETURN_PARENTHESES

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `RETURN_PARENTHESES` |
| **级别** | ERROR |
| **检查内容** | 检测 `return (x);` 多余括号。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:5595` |
| **agentrt-linux 规则编号** | OS-STD-CODE-043 |
| **示例** | `return (ret);` → 建议用 `return ret;` |

### 规则 44：USE_NEGATIVE_ERRNO

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `USE_NEGATIVE_ERRNO` |
| **级别** | WARNING |
| **检查内容** | 检测 `return ENOENT;`（应返回负 errno `-ENOENT`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:5663` |
| **agentrt-linux 规则编号** | OS-STD-CODE-044 |
| **示例** | `return ENOENT;` → 建议用 `return -ENOENT;` |

### 规则 45：DATE_TIME

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DATE_TIME` |
| **级别** | ERROR |
| **检查内容** | 检测代码中嵌入 `__DATE__`/`__TIME__`（导致不可重现构建）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7359` |
| **agentrt-linux 规则编号** | OS-STD-CODE-045 |
| **示例** | `pr_info("built at %s %s", __DATE__, __TIME__);` → 触发错误 |

### 规则 46：EXECUTE_PERMISSIONS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `EXECUTE_PERMISSIONS` |
| **级别** | ERROR |
| **检查内容** | 检测源码文件（.c/.h）被设为可执行权限。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:2957` |
| **agentrt-linux 规则编号** | OS-STD-CODE-046 |
| **示例** | `-rwxr-xr-x foo.c` → 触发 "do not set execute permissions for source files" |

### 规则 47：DOS_LINE_ENDINGS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DOS_LINE_ENDINGS` |
| **级别** | ERROR |
| **检查内容** | 检测 CRLF（\r\n）行尾，必须用 LF（\n）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:3592` |
| **agentrt-linux 规则编号** | OS-STD-CODE-047 |
| **示例** | 文件含 `\r\n` → 触发 "DOS line endings" |

### 规则 48：MISSING_EOF_NEWLINE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MISSING_EOF_NEWLINE` |
| **级别** | WARNING |
| **检查内容** | 检测文件末尾无换行符（POSIX 要求文件以换行结尾）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:3893` |
| **agentrt-linux 规则编号** | OS-STD-CODE-048 |
| **示例** | 文件末行无 `\n` → 触发 "Missing a newline at the end of the file" |

### 规则 49：SPLIT_STRING

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPLIT_STRING` |
| **级别** | WARNING |
| **检查内容** | 检测字符串字面量被折断（如 `"foo"\n"bar"`）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6286` |
| **agentrt-linux 规则编号** | OS-STD-FMT-002 |
| **示例** | `pr_info("foo " "bar");` → 触发 "quoted string split across lines" |

### 规则 50：FLEXIBLE_ARRAY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `FLEXIBLE_ARRAY` |
| **级别** | ERROR |
| **检查内容** | 检测结构体末尾使用零长/单元素数组，应改用 C99 柔性数组 `items[]`。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7479` |
| **agentrt-linux 规则编号** | OS-STD-CODE-049 |
| **示例** | `struct x { int n; struct y items[1]; };` → 建议用 `items[]` |

### 规则 51：MEMORY_BARRIER

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MEMORY_BARRIER` |
| **级别** | WARNING |
| **检查内容** | 检测 `smp_mb()`/`smp_rmb()`/`smp_wmb()`，建议用 `smp_store_release`/`smp_load_acquire`。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6695,6706` |
| **agentrt-linux 规则编号** | OS-STD-CODE-050 |
| **示例** | `smp_mb();` → 建议用 `smp_store_release()` |

### 规则 52：INLINE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `INLINE` |
| **级别** | WARNING |
| **检查内容** | 检测 `inline` 滥用（>3 行函数不建议 inline）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6757` |
| **agentrt-linux 规则编号** | OS-STD-CODE-051 |
| **示例** | 大函数加 `inline` → 触发 "inline is usually wrong" |

### 规则 53：LIKELY_MISUSE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `LIKELY_MISUSE` |
| **级别** | WARNING |
| **检查内容** | 检测 `likely()`/`unlikely()` 滥用（不应在普通条件分支上用）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7461,7488` |
| **agentrt-linux 规则编号** | OS-STD-CODE-052 |
| **示例** | `if (likely(x > 0))` → 触发 "likely/unlikely misuse" |

### 规则 54：CONST_STRUCT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `CONST_STRUCT` |
| **级别** | WARNING |
| **检查内容** | 检测 `static struct foo` 应为 `static const struct foo`（只读结构体应 const 化）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:7433` |
| **agentrt-linux 规则编号** | OS-STD-CODE-053 |
| **示例** | `static struct foo default = {...};` → 建议用 `static const struct foo` |

### 规则 55：INLINE_LOCATION

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `INLINE_LOCATION` |
| **级别** | ERROR |
| **检查内容** | 检测 `inline` 函数定义在 .c 文件而非 .h 文件（应放头文件以便共享）。 |
| **Linux 6.6 内核基线 源码行号** | `scripts/checkpatch.pl:6750` |
| **agentrt-linux 规则编号** | OS-STD-CODE-054 |
| **示例** | `.c` 文件中 `static inline int foo(void) {...}` 且被多文件用 → 触发错误 |

---

## 9. 规则汇总与统计

### 9.1 按级别统计

| 级别 | 规则数 | 占比 |
|------|--------|------|
| ERROR | 23 | 41.8% |
| WARNING | 26 | 47.3% |
| CHECK | 6 | 10.9% |
| **总计** | **55** | **100%** |

### 9.2 按主题分类统计

| 主题 | 规则数 | agentrt-linux 规则编号区间 |
|------|--------|--------------------------|
| 字符串与内存安全 | 7 | OS-STD-CODE-010/012/018/019 |
| 缩进与空格 | 6 | OS-STD-FMT-001 / OS-STD-FMT-007 / OS-STD-CODE-021 |
| 大括号与控制流 | 8 | OS-STD-CODE-015/022-025 |
| 类型与声明 | 7 | OS-STD-CODE-014/020/026/027/055/056/057 |
| 宏与预处理 | 5 | OS-STD-CODE-032-035/058 |
| 注释与文档 | 3 | OS-STD-CODE-036/037/059 |
| 头文件与许可证 | 3 | OS-STD-CODE-038-040 |
| 杂项与质量 | 16 | OS-STD-CODE-041-054 |

### 9.3 与 12 条强化规则的交叉引用

下表展示 checkpatch 规则如何支撑 `10-coding-style/C_Cpp_coding_style.md Part II` 的 12 条强化规则：

| 强化规则编号 | 强化规则名 | 对应 checkpatch 规则 |
|------------|----------|---------------------|
| OS-STD-FMT-001 | Tab-8 缩进强制 | CODE_INDENT, SPACE_BEFORE_TAB, TABSTOP, LEADING_SPACE, SUSPECT_CODE_INDENT |
| OS-KER-001 | goto 集中错误处理 | （间接，无直接 checkpatch 规则） |
| OS-STD-011 | bool 仅用于返回值与栈变量 | （间接，无直接 checkpatch 规则） |
| OS-STD-010 | 内核态禁 float | （由 -mno-80387 编译选项强制，无 checkpatch 规则） |
| OS-STD-FMT-002 | 行宽 80 列硬上限 | SPLIT_STRING（部分） |
| OS-STD-CODE-014 | 指定初始化器强制 | GLOBAL_INITIALISERS |
| OS-STD-CODE-015 | fallthrough; 显式穿透 | DEFAULT_NO_BREAK |
| OS-STD-CODE-020 | 禁止结构体 typedef | NEW_TYPEDEFS |
| OS-STD-CODE-004 | 零警告门禁 | （checkpatch 自身 0 ERROR/WARNING 退出码） |
| OS-STD-CODE-010 | strscpy 强制替代 | STRCPY, STRLCPY, STRNCPY |
| OS-BAN-002 | 禁止 BUG()/BUG_ON() | （间接，由 deprecated API 登记） |
| OS-BAN-003 | sizeof(*p) 替代 sizeof(struct) | ALLOC_SIZEOF_STRUCT, ALLOC_WITH_MULTIPLY |

---

## 10. CI 集成与门禁配置

### 10.1 checkpatch CI 调用

agentrt-linux CI 在每次合并请求中按以下方式调用 checkpatch：

```bash
# 基准路径（agentrt-linux 仓库根）
$ ./scripts/checkpatch.pl --no-tree --no-summary --no-tree \
    --strict --codespell --min-conf-desc-length=1 \
    --max-line-length=80 \
    $(git diff origin/main...HEAD --name-only)
```

关键参数说明：

- `--strict`：启用额外检查（CHK 级别提升为 WARNING）
- `--max-line-length=80`：行宽 80 列硬上限（对应 `.clang-format: ColumnLimit: 80`）
- `--no-tree`：不依赖内核源码树
- `--codespell`：启用拼写检查

### 10.2 退出码处理

| 退出码 | 含义 | agentrt-linux 处理 |
|--------|------|------------------|
| 0 | 0 ERROR / 0 WARNING | 通过，允许合并 |
| 1 | ≥1 ERROR | 拒绝合并 |
| 2 | 0 ERROR / ≥1 WARNING | 拒绝合并（WARNING 升级为 ERROR） |

### 10.3 白名单机制

部分场景需豁免 checkpatch 规则，需在 `.checkpatch-ignore` 文件中登记并经维护者批准：

```
# 文件: .checkpatch-ignore
# 格式: <文件路径> <规则名> <豁免理由>

# 例: 第三方贡献的兼容层
drivers/airymax/legacy/compat.c STRCPY 已知遗留代码，迁移中
```

---

## 11. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，覆盖 55 条 checkpatch 规则映射 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | Phase 1 格式迁移遗留消解 + 编号碰撞修复：(1) §1-§8 中 OS-KER-011→OS-STD-FMT-001（5 处缩进规则）、OS-KER-012→OS-STD-FMT-002（SPLIT_STRING）、OS-KER-013→OS-STD-CODE-020（NEW_TYPEDEFS）；(2) TRAILING_WHITESPACE 从 OS-STD-CODE-020 修正为 OS-STD-FMT-007（消解与 typedef 规则的编号碰撞）；(3) AVOID_EXTERNS 028→055、FUNCTION_ARGUMENTS 029→056（消解与 01 语义层 028/029 的编号碰撞）；(4) VOLATILE 030→057、MULTISTATEMENT_MACRO 031→058（消解与 01/C 语义层 030/031 的编号碰撞）；(5) NETWORKING_BLOCK_COMMENT_STYLE 036→059（消解与 BLOCK_COMMENT_STYLE 的重复分配）；(6) §9.3 历史标注清理、§0 子域声明更新（语义层范围 010~031，checkpatch 专属 032~059）、§9.2 汇总表更新 | 工程规范委员会 |

---

> **SPDX-License-Identifier**： AGPL-3.0-or-later OR Apache-2.0

***

# Part V: 内核文档标准

## 0. 文档说明

### 0.1 目的与范围

本规范定义 agentrt-linux 内核态 C 代码的 kernel-doc 注释强制标准，源自由 Linux 6.6 内核基线 `scripts/kernel-doc`（68KB Perl 脚本）所解析的内核文档注释语法。所有导出函数、公共结构体、枚举、typedef 必须有 kernel-doc 注释，CI 通过 `scripts/kernel-doc` 提取并校验。

### 0.2 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）
- 共享原语称"微核心原语"（不称"微内核原语"）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

### 0.3 Linux 6.6 内核基线 源码路径

- `scripts/kernel-doc`（68969 字节，Perl 脚本，第 1-50 行头部说明）
- `scripts/kernel-doc:45-50` —— "Read C language source or header FILEs, extract embedded documentation comments"
- `scripts/kernel-doc:48` —— "The documentation comments are identified by the \"/**\" opening comment mark."
- `scripts/kernel-doc:62-77` —— 类型匹配正则定义（`$type_constant`/`$type_func`/`$type_param` 等）

---

## 1. kernel-doc 注释基础规则

### 1.1 规则 OS-STD-DOC-001：`/** */` 起始标记强制

kernel-doc 注释**必须**以 `/**` 起始（双星号），以 `*/` 结尾。普通注释 `/* */`（单星号）不被 kernel-doc 脚本识别。

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:48`

**正确示例**：

```c
/**
 * airy_ipc_send() - Send a message on an IPC channel
 * @chan: IPC channel to send on
 * @msg: Message to send
 *
 * Context: Process context. Takes and releases chan->lock.
 * Return: 0 on success, negative errno on failure.
 */
int airy_ipc_send(struct airy_ipc_chan *chan,
		     const struct airy_ipc_msg *msg);
```

**错误示例**：

```c
/* WRONG: 单星号，不被 kernel-doc 识别 */
/* airy_ipc_send - Send a message
 * @chan: IPC channel
 */
int airy_ipc_send(struct airy_ipc_chan *chan, ...);
```

### 1.2 规则 OS-STD-DOC-002：短描述（short description）强制

每个 kernel-doc 注释第一行**必须**包含 `<符号名>() - <短描述>` 形式的短描述。短描述应在一行内完成，描述符号的核心用途。

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc` 短描述解析逻辑（`-Wshort-description` 警告）

**正确示例**：

```c
/**
 * airy_sched_pick_next() - Pick the next runnable agent task
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
 * airy_sched_pick_next	/* WRONG: 无短描述，缺 " - " 分隔 */
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

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:66` (`$type_param` 正则 `\@(\w*((\.\w+)|(->\w+))*(\.\.\.)?)`)

**完整模板**：

```c
/**
 * airy_ipc_send_batch() - Send a batch of IPC messages atomically
 * @chan: IPC channel to send on; must be initialized.
 * @msgs: Array of messages to send; length must equal @count.
 * @count: Number of messages in @msgs; must be > 0 and <= 64.
 * @flags: Send flags, bitwise OR of %AIRY_IPC_F_NOWAIT and
 *         %AIRY_IPC_F_SIGNAL.
 *
 * Sends @count messages from @msgs to @chan atomically. If
 * %AIRY_IPC_F_NOWAIT is set, the function returns immediately if the
 * channel buffer is full; otherwise it blocks.
 *
 * The function is idempotent on failure: no partial send occurs.
 *
 * Context: Process context. May sleep if @flags does not include
 *          %AIRY_IPC_F_NOWAIT. Takes and releases @chan->lock.
 * Return:
 * * 0 on success.
 * * -ENOMEM if internal allocation fails.
 * * -EAGAIN if %AIRY_IPC_F_NOWAIT is set and buffer is full.
 * * -E2BIG if @count exceeds the channel maximum.
 */
int airy_ipc_send_batch(struct airy_ipc_chan *chan,
			   const struct airy_ipc_msg *msgs,
			   size_t count, u32 flags);
```

### 2.2 规则 OS-STD-DOC-004：`@arg:` 参数描述强制

所有函数参数（含 `void`/`struct` 指针）**必须**有 `@arg:` 描述行。`...` 可变参数用 `@...:` 描述。参数描述应包含取值范围、单位、是否可空（NULL）、是否被修改。

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:66` (`$type_param`)

**正确示例**：

```c
/**
 * airy_mem_reserve() - Reserve memory for agent task
 * @task: Task to reserve memory for; must not be NULL.
 * @size: Reserve size in bytes; must be > 0 and power of 2.
 * @gfp: GFP flags; %GFP_KERNEL or %GFP_ATOMIC.
 * @...: Variable format arguments for reservation tag, printf-style.
 */
```

### 2.3 规则 OS-STD-DOC-005：`Return:` 返回值强制

所有非 `void` 函数**必须**有 `Return:` 字段，描述成功值与所有错误码（含负 errno）。错误码以 `-EXXX` 形式列出，并解释触发条件。

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:26` (`-Wreturn` 警告开关)

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

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc` `Context:` 解析逻辑

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

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:72` (`$type_struct` 正则 `\&(struct\s*([_\w]+))`)

**完整模板**：

```c
/**
 * struct airy_task - Agent task descriptor (micro-core primitive)
 * @id: Unique task ID; assigned at creation, never recycled.
 * @link: List node for ready queue linkage; owned by scheduler.
 * @state: Task state, see enum airy_task_state.
 * @token_budget: Token budget in Q16.16 fixed-point; see airy_q16_t.
 * @magic: Magic 0x41475453 ('AGTS') for descriptor validation.
 *
 * Describes a single agent task managed by the micro-core scheduler.
 * Shared between agentrt (user-space) and agentrt-linux (kernel)
 * via the [SC] contract layer.
 *
 * Locking: state and token_budget are protected by the owning
 * scheduler's lock.
 */
struct airy_task {
	u64			id;
	struct list_head	link;
	enum airy_task_state	state;
	airy_q16_t		token_budget;
	__u32			magic;		/* 0x41475453 'AGTS' */
};
```

### 3.2 规则 OS-STD-DOC-008：嵌套成员引用

结构体成员若为嵌套结构体，kernel-doc 中用 `@parent.child:` 或 `@parent->child:` 形式描述子成员。

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:66` (`$type_param` 含 `(\.\w+)|(->\w+)`)

**正确示例**：

```c
/**
 * struct airy_ipc_ctx - IPC context
 * @tx.queue: Transmit queue head; protected by @tx.lock.
 * @tx.depth: Maximum pending entries in @tx.queue.
 * @rx.queue: Receive queue head; protected by @rx.lock.
 * @rx.depth: Maximum pending entries in @rx.queue.
 */
struct airy_ipc_ctx {
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

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:71` (`$type_enum` 正则 `\&(enum\s*([_\w]+))`)

**完整模板**：

```c
/**
 * enum airy_task_state - Agent task lifecycle states
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
enum airy_task_state {
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

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:68-69` (`$type_fp_param`/`$type_fp_param2`)

**完整模板**：

```c
/**
 * struct airy_sched_ops - Scheduler operations vector
 * @pick_next(): Select next runnable task; returns NULL if queue empty.
 * @enqueue(): Enqueue a task into the ready queue; called with @lock held.
 * @dequeue(): Dequeue a task from the ready queue; called with @lock held.
 * @yield(): Yield current task; re-queue at tail of its priority band.
 *
 * All callbacks are invoked under the scheduler's @lock. Callbacks
 * must not sleep or block.
 */
struct airy_sched_ops {
	struct airy_task *(*pick_next)(struct airy_sched *sched);
	void (*enqueue)(struct airy_sched *sched,
			struct airy_task *task);
	void (*dequeue)(struct airy_sched *sched,
			struct airy_task *task);
	void (*yield)(struct airy_sched *sched);
};
```

---

## 6. typedef 文档模板

### 6.1 规则 OS-STD-DOC-011：仅允许 typedef 的 kernel-doc

agentrt-linux 禁止结构体 typedef（OS-STD-CODE-020，原 OS-KER-013），但允许 (a)-(e) 类 typedef（如 `airy_q16_t`）。这些允许的 typedef **必须**有 kernel-doc 注释，描述其底层类型与语义。

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:73` (`$type_typedef` 正则)

**完整模板**：

```c
/**
 * typedef airy_q16_t - Q16.16 fixed-point value
 *
 * Signed 32-bit integer representing a fixed-point value: high 16
 * bits are the integer part, low 16 bits are the fractional part.
 * Range: [-32768.0, 32767.99998].
 *
 * Required in kernel code because x86 builds with -mno-80387
 * (no x87 FPU). See arch/x86/Makefile:137.
 *
 * Use airy_q16_mul(), airy_q16_div() helpers for arithmetic;
 * never operate on the raw int32_t.
 */
typedef int32_t airy_q16_t;
```

---

## 7. 交叉引用与高亮

### 7.1 规则 OS-STD-DOC-012：交叉引用标记

kernel-doc 注释中引用其他符号时**必须**使用以下标记：

| 标记 | 含义 | Linux 6.6 内核基线 正则 |
|------|------|------------|
| `%CONST` | 宏常量 | `$type_constant2` (`\%([-_\w]+)`) |
| `func()` | 函数引用 | `$type_func` (`(\w+)\(\)`) |
| `&struct foo` | 结构体引用 | `$type_struct` |
| `&enum foo` | 枚举引用 | `$type_enum` |
| `&typedef foo` | typedef 引用 | `$type_typedef` |
| `&union foo` | 联合体引用 | `$type_union` |
| `&foo.bar` | 成员引用 | `$type_member` |

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc:63-77`（高亮正则定义）

**正确示例**：

```c
/**
 * airy_sched_init() - Initialize a scheduler
 * @sched: Scheduler context; must be zero-initialized.
 *
 * Initializes @sched with default ops from %AIRY_SCHED_DEFAULT_OPS.
 * The ready queue is built via &struct list_head. After init, call
 * airy_sched_pick_next() to dispatch tasks.
 *
 * See also: &enum airy_task_state, &typedef airy_q16_t.
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

**Linux 6.6 内核基线 源码路径**: `scripts/kernel-doc` `DOC:` 段解析逻辑

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

#ifndef _AIRY_IPC_H
#define _AIRY_IPC_H

#include <linux/types.h>

#define AIRY_IPC_MAGIC		0x41524531	/* 'ARE1' */
#define AIRY_IPC_HDR_SZ		128

/**
 * struct airy_ipc_msg_hdr - IPC message header (128 bytes)
 * @magic: Magic 0x41524531 ('ARE1'); validates protocol version.
 * @opcode: Operation code; see enum airy_ipc_opcode.
 * @flags: Message flags, bitwise OR of %AIRY_IPC_F_*.
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
struct airy_ipc_msg_hdr {
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

#endif /* _AIRY_IPC_H */
```

> **SSoT 声明**：本结构体定义以 `include/airymax/ipc.h`（物理宿主见 `50-engineering-standards/120-cross-project-code-sharing.md` §Layout C）为单一数据源。此处的 kernel-doc 示例与 SSoT Layout C 逐字段一致，结构体名为 `struct airy_ipc_msg_hdr`。

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

***

# Part VI: Clang-Format 强制执行

## 0. 文档说明

### 0.1 目的与范围

本文件研究 Linux 6.6 内核基线 `.clang-format`（689 行配置），提取关键格式化选项，制定 agentrt-linux 定制版 `.clang-format`，并定义 CI 中的 `make format-check` 门禁流程。所有 agentrt-linux 内核态 C 文件必须通过 `clang-format` 校验。

> **交叉引用**：本 Part 是 `.clang-format` 的工具配置层（完整 YAML + Linux 6.6 内核基线 源码行号 + CI 门禁流程）。`.clang-format` 在 7 层验证体系中的框架定位（第 3 层预提交 + 第 4 层 CI 门禁）、7 层验证流水线全图、CI/CD 矩阵配置与其他工具链（sparse / Smatch / Coccinelle / KUnit / kselftest）的协同关系，详见 [05-development-process.md](05-development-process.md) Part II §5（格式化与风格检查）与 §1（7 层自动化验证体系）。格式规则定义（OS-STD-FMT-001~029）与规则到配置项的速查映射表，详见 本文档 Part II §1-§7（格式规则）与 §9（clang-format 关键配置项表）。checkpatch 规则到 agentrt-linux 规则编号的逐条映射表，详见 本文档 Part IV §1-§8（55 条规则映射）。

### 0.2 Linux 6.6 内核基线 源码路径

- `.clang-format`（689 行，文件首行 `# SPDX-License-Identifier: GPL-2.0`）
- `.clang-format:2-9` —— 头部说明（要求 clang-format >= 11）
- `.clang-format:55` —— `ColumnLimit: 80`
- `.clang-format:645` —— `IndentWidth: 8`
- `.clang-format:687` —— `TabWidth: 8`
- `.clang-format:688` —— `UseTab: Always`
- `.clang-format:31-46` —— `BraceWrapping` 配置（K&R 风格）
- `.clang-format:48` —— `BreakBeforeBraces: Custom`
- `.clang-format:668` —— `PointerAlignment: Right`
- `.clang-format:71-635` —— `ForEachMacros` 列表（约 565 个宏）

### 0.3 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

---

## 1. 关键配置项详解

> **交叉引用**：本节是每个配置项的 Linux 6.6 内核基线 源码行号与详细理由。格式规则定义（OS-STD-FMT-001~029）与规则到配置项的速查映射表，详见 本文档 Part II §1-§7（格式规则）与 §9（clang-format 关键配置项表）。

### 1.1 缩进配置（OS-STD-FMT-001 强制）

**Linux 6.6 内核基线 源码路径**: `.clang-format:645,687,688`

```yaml
IndentWidth: 8			# 缩进宽度 8 字符
TabWidth: 8			# Tab 显示宽度 8
UseTab: Always			# 始终用 Tab（非空格）
ContinuationIndentWidth: 8	# 续行缩进宽度 8
ConstructorInitializerIndentWidth: 8
```

**理由**：8 字符缩进是 Linux 内核传统，作为"复杂度自然惩罚"——嵌套超过 3 层即提示应重构。`UseTab: Always` 确保 Tab 而非空格，配合 `IndentWidth`/`TabWidth` 一致为 8。

### 1.2 行宽配置（OS-STD-FMT-002 强制）

**Linux 6.6 内核基线 源码路径**: `.clang-format:55`

```yaml
ColumnLimit: 80			# 行宽 80 列硬上限
```

**理由**：80 列限制源于终端显示传统，强制函数拆分并允许分屏 diff。`clang-format` 在格式化时自动折行至 80 列内。

### 1.3 大括号配置（K&R 风格）

**Linux 6.6 内核基线 源码路径**: `.clang-format:31-46,48`

```yaml
BreakBeforeBraces: Custom	# 自定义大括号放置
BraceWrapping:
  AfterClass: false		# class 后不换行
  AfterControlStatement: false	# if/for/while 后不换行
  AfterEnum: false		# enum 后不换行
  AfterFunction: true		# 函数后换行（{ 在新行）
  AfterNamespace: true
  AfterStruct: false		# struct 后不换行
  AfterUnion: false
  BeforeCatch: false		# catch 前不换行
  BeforeElse: false		# else 前不换行
  IndentBraces: false
  SplitEmptyFunction: true
  SplitEmptyRecord: true
```

**理由**：这是 Linux K&R 风格——非函数块（if/for/while/switch）的 `{` 在行尾，函数的 `{` 在新行首。

### 1.4 指针对齐配置

**Linux 6.6 内核基线 源码路径**: `.clang-format:62,668`

```yaml
DerivePointerAlignment: false	# 不从现有代码推导
PointerAlignment: Right		# * 贴变量名（char *p 而非 char* p）
```

**理由**：`*` 贴变量名避免 `char* a, b` 的歧义（b 不是指针）。

### 1.5 空格配置

**Linux 6.6 内核基线 源码路径**: `.clang-format:672-685`

```yaml
SpaceAfterCStyleCast: false			# (cast) 后无空格
SpaceAfterTemplateKeyword: true		# template 后有空格
SpaceBeforeAssignmentOperators: true		# = 前有空格
SpaceBeforeCtorInitializerColon: true
SpaceBeforeInheritanceColon: true
SpaceBeforeParens: ControlStatementsExceptForEachMacros	# if/for 后有空格，foreach 宏除外
SpaceBeforeRangeBasedForLoopColon: true
SpaceInEmptyParentheses: false		# () 内无空格
SpacesBeforeTrailingComments: 1		# 注释前 1 个空格
SpacesInAngles: false			# <int> 内无空格
SpacesInContainerLiterals: false
SpacesInCStyleCastParentheses: false
SpacesInParentheses: false		# ( 内无空格
SpacesInSquareBrackets: false		# [ 内无空格
```

### 1.6 列对齐与折行配置

**Linux 6.6 内核基线 源码路径**: `.clang-format:12-30,47-66,559-666`

```yaml
AlignAfterOpenBracket: Align		# 开括号后对齐续行
AlignConsecutiveAssignments: false	# 不对齐连续赋值
AlignConsecutiveDeclarations: false	# 不对齐连续声明
AlignEscapedNewlines: Left		# 转义换行左对齐
AlignOperands: true			# 操作数对齐
AlignTrailingComments: false		# 不对齐行尾注释
BinPackArguments: true			# 参数可打包多行
BinPackParameters: true
BreakBeforeBinaryOperators: None	# 二元操作符前不换行
BreakBeforeTernaryOperators: false	# 三元操作符前不换行
BreakStringLiterals: false		# 不折断字符串字面量
ReflowComments: false			# 不重排注释
```

### 1.7 Include 排序配置

**Linux 6.6 内核基线 源码路径**: `.clang-format:637-642`

```yaml
IncludeBlocks: Preserve		# 保留现有 include 块结构
IncludeCategories:
  - Regex: '.*'
    Priority: 1
IncludeIsMainRegex: '(Test)?$'
SortIncludes: false			# 不自动排序 include（保留手动顺序）
SortUsingDeclarations: false
```

**理由**：Linux 内核 include 顺序有特定要求（系统头、内核头、子系统头），自动排序会破坏语义，故禁用。

### 1.8 短语句配置

**Linux 6.6 内核基线 源码路径**: `.clang-format:20-24`

```yaml
AllowShortBlocksOnASingleLine: false		# 不允许单行块
AllowShortCaseLabelsOnASingleLine: false	# 不允许单行 case
AllowShortFunctionsOnASingleLine: None		# 不允许单行函数
AllowShortIfStatementsOnASingleLine: false	# 不允许单行 if
AllowShortLoopsOnASingleLine: false		# 不允许单行循环
```

### 1.9 ForEachMacros 配置

**Linux 6.6 内核基线 源码路径**: `.clang-format:71-635`（约 565 个宏）

`ForEachMacros` 列表告知 `clang-format` 哪些宏是 `for_each` 风格的循环宏（如 `list_for_each_entry`），使其在格式化时正确处理。agentrt-linux 在此基础上需追加自定义宏：

```yaml
ForEachMacros:
  # ... 保留 Linux 6.6 内核基线 全部 565 个宏 ...
  - 'airy_for_each_task'		# agentrt-linux 自定义
  - 'airy_for_each_chan'
  - 'airy_for_each_token'
```

---

## 2. agentrt-linux 定制版 .clang-format

### 2.1 完整配置文件

以下是 agentrt-linux 仓库根目录 `.clang-format` 的完整内容（基于 Linux 6.6 内核基线 定制）：

```yaml
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
#
# clang-format configuration for agentrt-linux (AirymaxOS).
# Based on Linux 6.6 .clang-format (Linux 6.6 内核基线) with agent-specific macros.
# Intended for clang-format >= 11.
#
# See: Documentation/process/clang-format.rst (upstream)
---
AccessModifierOffset: -4
AlignAfterOpenBracket: Align
AlignConsecutiveAssignments: false
AlignConsecutiveDeclarations: false
AlignEscapedNewlines: Left
AlignOperands: true
AlignTrailingComments: false
AllowAllParametersOfDeclarationOnNextLine: false
AllowShortBlocksOnASingleLine: false
AllowShortCaseLabelsOnASingleLine: false
AllowShortFunctionsOnASingleLine: None
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false
AlwaysBreakAfterDefinitionReturnType: None
AlwaysBreakAfterReturnType: None
AlwaysBreakBeforeMultilineStrings: false
AlwaysBreakTemplateDeclarations: false
BinPackArguments: true
BinPackParameters: true
BraceWrapping:
  AfterClass: false
  AfterControlStatement: false
  AfterEnum: false
  AfterFunction: true
  AfterNamespace: true
  AfterObjCDeclaration: false
  AfterStruct: false
  AfterUnion: false
  AfterExternBlock: false
  BeforeCatch: false
  BeforeElse: false
  IndentBraces: false
  SplitEmptyFunction: true
  SplitEmptyRecord: true
  SplitEmptyNamespace: true
BreakBeforeBinaryOperators: None
BreakBeforeBraces: Custom
BreakBeforeInheritanceComma: false
BreakBeforeTernaryOperators: false
BreakConstructorInitializersBeforeComma: false
BreakConstructorInitializers: BeforeComma
BreakAfterJavaFieldAnnotations: false
BreakStringLiterals: false
ColumnLimit: 80
CommentPragmas: '^ IWYU pragma:'
CompactNamespaces: false
ConstructorInitializerAllOnOneLineOrOnePerLine: false
ConstructorInitializerIndentWidth: 8
ContinuationIndentWidth: 8
Cpp11BracedListStyle: false
DerivePointerAlignment: false
DisableFormat: false
ExperimentalAutoDetectBinPacking: false
FixNamespaceComments: false
ForEachMacros:
  # ----- Linux 6.6 内核基线 inherited (565 macros, abbreviated here) -----
  - 'list_for_each_entry'
  - 'list_for_each_entry_safe'
  - 'hlist_for_each_entry'
  - 'for_each_cpu'
  - 'for_each_online_cpu'
  - 'xa_for_each'
  # ... (full list inherited from Linux 6.6 内核 .clang-format:71-635) ...
  # ----- agentrt-linux custom -----
  - 'airy_for_each_task'
  - 'airy_for_each_chan'
  - 'airy_for_each_sched'
  - 'airy_for_each_token'
  - 'airy_for_each_mem_range'
IncludeBlocks: Preserve
IncludeCategories:
  - Regex: '.*'
    Priority: 1
IncludeIsMainRegex: '(Test)?$'
IndentCaseLabels: false
IndentGotoLabels: false
IndentPPDirectives: None
IndentWidth: 8
IndentWrappedFunctionNames: false
JavaScriptQuotes: Leave
JavaScriptWrapImports: true
KeepEmptyLinesAtTheStartOfBlocks: false
MacroBlockBegin: ''
MacroBlockEnd: ''
MaxEmptyLinesToKeep: 1
NamespaceIndentation: None
ObjCBinPackProtocolList: Auto
ObjCBlockIndentWidth: 8
ObjCSpaceAfterProperty: true
ObjCSpaceBeforeProtocolList: true
PenaltyBreakAssignment: 10
PenaltyBreakBeforeFirstCallParameter: 30
PenaltyBreakComment: 10
PenaltyBreakFirstLessLess: 0
PenaltyBreakString: 10
PenaltyExcessCharacter: 100
PenaltyReturnTypeOnItsOwnLine: 60
PointerAlignment: Right
ReflowComments: false
SortIncludes: false
SortUsingDeclarations: false
SpaceAfterCStyleCast: false
SpaceAfterTemplateKeyword: true
SpaceBeforeAssignmentOperators: true
SpaceBeforeCtorInitializerColon: true
SpaceBeforeInheritanceColon: true
SpaceBeforeParens: ControlStatementsExceptForEachMacros
SpaceBeforeRangeBasedForLoopColon: true
SpaceInEmptyParentheses: false
SpacesBeforeTrailingComments: 1
SpacesInAngles: false
SpacesInContainerLiterals: false
SpacesInCStyleCastParentheses: false
SpacesInParentheses: false
SpacesInSquareBrackets: false
Standard: Cpp03
TabWidth: 8
UseTab: Always
...
```

### 2.2 与 Linux 6.6 内核基线的差异

| 配置项 | Linux 6.6 内核基线 值 | agentrt-linux 值 | 差异说明 |
|--------|-----------|-----------------|---------|
| `ForEachMacros` | 565 个上游宏 | 565 + 5 个 agent 自定义 | 追加 agentrt 自定义宏 |
| SPDX 标识 | `GPL-2.0` | `AGPL-3.0-or-later OR Apache-2.0` | 许可证变更 |
| 其余所有项 | — | 完全继承 | 无差异 |

---

## 3. CI 门禁：make format-check

### 3.1 规则 OS-STD-FMT-029：format-check 强制门禁（复用 02 §8.5）

agentrt-linux CI 在每次合并请求中**必须**运行 `make format-check`，校验所有 C 文件符合 `.clang-format`。任何格式偏差即视为构建失败。

> **C8 冲突消解说明（2026-07-12）**：本节原使用 `OS-STD-FMT-001` 标注 format-check 门禁，与 07 §10.2.2 / 02 §1.1 中 `OS-STD-FMT-001 = "Tab 8 字符宽"` 冲突。已改为复用 02 §8.5 定义的 `OS-STD-FMT-029`（CI 强制 `make format-check`），消除编号双义。

### 3.2 Makefile 集成

在 agentrt-linux 仓库根 `Makefile` 中添加以下目标：

```makefile
# 文件: Makefile（agentrt-linux 仓库根）

CLANG_FORMAT ?= clang-format
FORMAT_SOURCES := $(shell find . -name '*.c' -o -name '*.h' | \
	grep -vE '^(./vendor/|./third_party/)')

.PHONY: format-check format-diff format-apply

## format-check: 验证所有 C 文件符合 .clang-format
format-check:
	@echo "  CHECK   clang-format"
	@fail=0; \
	for f in $(FORMAT_SOURCES); do \
		if ! $(CLANG_FORMAT) --dry-run -W error $$f > /dev/null 2>&1; then \
			echo "  FAIL    $$f"; \
			fail=1; \
		fi; \
	done; \
	if [ $$fail -ne 0 ]; then \
		echo "Error: clang-format check failed. Run 'make format-apply' to fix."; \
		exit 1; \
	fi
	@echo "  OK      clang-format"

## format-diff: 显示需要格式化的 diff（不修改文件）
format-diff:
	@for f in $(FORMAT_SOURCES); do \
		$(CLANG_FORMAT) $$f | diff -u $$f - || true; \
	done

## format-apply: 应用 clang-format 自动修复
format-apply:
	@echo "  FORMAT  $(FORMAT_SOURCES)"
	@for f in $(FORMAT_SOURCES); do \
		$(CLANG_FORMAT) -i $$f; \
	done
```

### 3.3 Git pre-commit 钩子（推荐）

开发者可在本地 `.git/hooks/pre-commit` 中安装钩子，在提交前自动检查：

```bash
#!/bin/bash
# 文件: .git/hooks/pre-commit
# 检查暂存的 C 文件格式

CLANG_FORMAT=${CLANG_FORMAT:-clang-format}
exit_code=0

for f in $(git diff --cached --name-only --diff-filter=ACM -- '*.c' '*.h'); do
    if ! git show ":$f" | $CLANG_FORMAT --dry-run -W error - 2>/dev/null; then
        echo "clang-format check failed: $f"
        echo "Run 'clang-format -i $f' to fix."
        exit_code=1
    fi
done

exit $exit_code
```

### 3.4 CI 流水线集成

在 CI 配置文件（如 `.github/workflows/ci.yml` 或 `.gitlab-ci.yml`）中添加 format-check 阶段：

```yaml
# 文件: .github/workflows/ci.yml（示例）
stages:
  - lint
  - build
  - test

format-check:
  stage: lint
  image: clang-15:latest
  script:
    - make format-check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

---

## 4. 常见格式问题与修复

### 4.1 问题 1：空格 vs Tab 混用

**触发**：`CODE_INDENT` (checkpatch) + clang-format 偏差

**修复前**：

```c
static int foo(void)
{
    int x = 0;		/* WRONG: 4 空格 */
    return x;
}
```

**修复后**（`clang-format -i` 自动修复）：

```c
static int foo(void)
{
	int x = 0;		/* OK: Tab */
	return x;
}
```

### 4.2 问题 2：大括号位置错误

**修复前**：

```c
if (x)
{
	foo();
}
```

**修复后**：

```c
if (x) {
	foo();
}
```

### 4.3 问题 3：指针 * 位置

**修复前**：

```c
char* p;			/* WRONG: * 贴类型 */
```

**修复后**：

```c
char *p;			/* OK: * 贴变量名 */
```

### 4.4 问题 4：行宽超限

**修复前**：

```c
static int very_long_function_name(struct very_long_struct_name *param1, struct another_long_struct *param2)
```

**修复后**（clang-format 自动折行对齐到开括号）：

```c
static int very_long_function_name(struct very_long_struct_name *param1,
				   struct another_long_struct *param2)
```

### 4.5 问题 5：函数大括号位置

**修复前**：

```c
int foo(void) {
	return 0;
}
```

**修复后**（函数 `{` 在新行）：

```c
int foo(void)
{
	return 0;
}
```

---

## 5. 与 checkpatch 的协作

### 5.1 职责分工

| 工具 | 检查范围 | 优势 |
|------|---------|------|
| `clang-format` | 纯格式（缩进、空格、大括号、对齐） | 可自动修复 |
| `checkpatch.pl` | 格式 + 语义（如废弃 API、typedef 滥用、宏错误） | 语义感知 |

### 5.2 推荐流水线顺序

```
git commit
   ↓
clang-format --dry-run -W error   (格式门禁)
   ↓ pass
checkpatch.pl --strict            (语义门禁)
   ↓ pass
build with -Wall -Werror          (编译门禁)
   ↓ pass
tests                             (测试门禁)
   ↓ pass
merge allowed
```

### 5.3 互不覆盖的场景

以下场景 `clang-format` 无法检测，必须依赖 `checkpatch`：

- `strcpy` 调用（OS-STD-CODE-010）
- `BUG_ON` 使用（OS-BAN-002）
- 新增 `typedef`（OS-STD-CODE-020）
- 缺失 SPDX 标识（OS-STD-CODE-040）
- 隐式 fall-through（OS-STD-CODE-015）

---

## 6. 速查表：关键配置项速查

> **SSoT 注（2026-07-12 全量迁移完成）**：本表与 本文档 Part II §9 内容重叠，但本表附"Linux 6.6 内核基线 行号"列为独特价值（Part II §9 无此列）。格式规则定义的唯一权威来源（SSoT）为 本文档 Part II §1-§8（OS-STD-FMT-001~029）。本表所有历史编号（原 OS-KER 系列格式规则）已于 2026-07-12 全量迁移为 OS-STD-FMT-NNN / OS-STD-CODE-NNN / OS-BAN-NNN 三类，与本文档 Part II SSoT 完全一致。

| 配置项 | 值 | Linux 6.6 内核基线 行号 | 对应规则 |
|--------|-----|------------|---------|
| `ColumnLimit` | 80 | `.clang-format:55` | OS-STD-FMT-002 |
| `IndentWidth` | 8 | `.clang-format:645` | OS-STD-FMT-001 |
| `TabWidth` | 8 | `.clang-format:687` | OS-STD-FMT-001 |
| `UseTab` | Always | `.clang-format:688` | OS-STD-FMT-001 |
| `BreakBeforeBraces` | Custom | `.clang-format:48` | OS-STD-CODE-022 |
| `PointerAlignment` | Right | `.clang-format:668` | OS-STD-CODE-026 |
| `SpaceBeforeParens` | ControlStatementsExceptForEachMacros | `.clang-format:677` | OS-STD-CODE-021 |
| `SortIncludes` | false | `.clang-format:670` | OS-STD-CODE-038 |
| `AllowShortFunctionsOnASingleLine` | None | `.clang-format:22` | OS-STD-CODE-022 |
| `ReflowComments` | false | `.clang-format:669` | OS-STD-CODE-036 |
| `BreakStringLiterals` | false | `.clang-format:54` | OS-STD-FMT-002 |

---

## 7. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，定义 agentrt-linux 定制 .clang-format + CI 门禁 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | Phase 1 格式迁移遗留消解：§1.1/§1.2/§5.3/§6 速查表中原残留 OS-KER-011/012/013/015 全量迁移为 OS-STD-FMT-001/002 + OS-STD-CODE-020 + OS-BAN-002，与本文档 Part II SSoT 完全一致 | 工程规范委员会 |

---

> **SPDX-License-Identifier**： AGPL-3.0-or-later OR Apache-2.0
