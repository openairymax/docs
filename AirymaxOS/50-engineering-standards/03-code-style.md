Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）代码风格标准

> **文档定位**: agentrt-linux（AirymaxOS，极境智能体操作系统）工程标准规范第 3 卷——代码风格。本卷规定模块化设计、策略机制分离、函数分解、数据结构、抽象层次、防御性、性能、工具链使用等"该怎么思考"的工程风格决策。它承接 02 代码格式的机械规则，向 04 工程思想的双重稳定性哲学过渡。
> **版本**: 1.0.1（开发）
> **最后更新**: 2026-07-06
> **同源映射**: `docs/AirymaxRT/ARCHITECTURAL_PRINCIPLES.md`（五维正交 24 原则）+ Linux 6.6 内核基线 `Documentation/process/coding-style.rst` §6~§22 + `stable-api-nonsense.rst`
> **理论根基**: Linux 6.6 内核基线工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）

---

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

本卷所有强制规则均赋予 **OS-KER**（内核工程）/ **OS-STD**（标准）编号，从 OS-KER-022 接续 02 卷。

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

### 1.1 反过度抽象（OS-KER-022）

**OS-KER-022**：禁止"未来扩展"式抽象——若函数参数全为 0 的"未来扩展位"已存在 6 个月以上仍无第二使用者，应删除该抽象。预先抽象会冻结错误的设计——你无法对未来需求做出准确预测，但可以基于当前已知需求做出清晰决策。

```c
/* 错误：预留 5 个 ops 槽位给"未来可能的需求" */
struct airymaxos_dispatcher_ops {
	void (*submit)(struct task *t);
	void (*cancel)(struct task *t);
	void (*future_op_1)(void);  /* 永远不会被实现 */
	void (*future_op_2)(void);
	void (*future_op_3)(void);
};

/* 正确：当前只用到的 ops */
struct airymaxos_dispatcher_ops {
	void (*submit)(struct task *t);
	void (*cancel)(struct task *t);
};
```

### 1.2 反对隐藏硬件访问的多 OS 抽象层（OS-KER-023）

**OS-KER-023**：agentrt-linux 是 Linux 6.6 内核基线专属发行版，不维护"在 Linux / Windows / RTOS 之间切换"的硬件抽象层。若需跨平台一致性，由 agentrt 在用户空间承担（E-4 原则），agentrt-linux 内核态不背跨平台包袱。

### 1.3 代码重复应抽出到库（OS-STD-212）

**OS-STD-212**：当代码在 3 处以上重复时，应抽出到 `lib/` 或 `commons/` 子模块；当代码在 2 处重复时，可暂保留，但需在 PR 描述中标注"已知重复，待第三处出现时抽出"。该规则避免"过早抽象"与"放任重复"两极。

### 1.4 agentrt-linux 专属分层职责（OS-STD-213）

**OS-STD-213**：agentrt-linux 按 C-1 双系统协同原则分层选择实现语言：

| 层 | 子仓 | 主语言 | 选择理由 |
|----|------|--------|----------|
| 热路径 | airymaxos-kernel 调度 / 内存 / IPC | C | cache 友好、零运行时开销、与 Linux 6.6 内核基线对齐 |
| 慢路径 | airymaxos-security / airymaxos-memory 元数据 | Rust | 内存安全、防整类 CVE |
| Agent 路径 | airymaxos-cognition / airymaxos-cloudnative | Python / TS | 生态丰富、迭代快、与 agentrt 同源 |

---

## 2. 策略与机制分离

### 2.1 机制留在内核（OS-KER-024）

**OS-KER-024**：内核只提供"能力"——调度器提供"切换 CPU 上下文的能力"、IPC 提供消息传递的能力、内存管理提供分配释放的能力。任何"决策"——按什么顺序调度、消息如何路由、何时回收内存——都外移到用户空间或可插拔模块。

### 2.2 策略外移至用户空间或可插拔模块（OS-KER-025）

**OS-KER-025**：策略必须可在不重新编译内核的情况下替换。agentrt-linux 通过两类机制实现：用户态守护进程（K-3 服务隔离）+ eBPF / sched_ext 可插拔内核扩展（K-4 可插拔策略）。

```c
/* 错误：策略硬编码进内核 */
static int airymaxos_pick_next_cpu(struct task_struct *p)
{
	/* 硬编码"优先选 CPU 0"——这是策略，不是机制 */
	return 0;
}

/* 正确：内核只提供钩子，策略由 sched_ext BPF 程序提供 */
struct sched_class airymaxos_agent_sched = {
	.pick_next_task		= ext_sched_pick,  /* BPF 程序注入 */
	.put_prev_task		= ext_sched_put,
	/* ...其余回调由 BPF 程序决定 */
};
```

### 2.3 sched_ext BPF 调度器作为极端范式（OS-KER-026）

**OS-KER-026**：airymaxos-kernel 的 `SCHED_AGENT` 调度类必须基于 sched_ext 框架实现——这是策略机制分离的最纯粹形态。BPF 程序在用户态编写、动态加载、热替换，而无需内核重新编译或重启。任何把调度策略硬编码进 `kernel/sched/` 的 PR 直接拒绝。

### 2.4 agentrt-linux 专属：K-4 可插拔策略原则映射

K-4 可插拔策略是 Airymax 五维正交 24 原则在调度、IPC 路由、安全裁决、记忆检索等"算法选择"上的统一约束。所有算法维度必须暴露策略接口，可在运行时替换。详见 [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md) §3.4。

---

## 3. 函数分解风格

### 3.1 大函数应分解为多个 helper（OS-KER-027）

**OS-KER-027**：函数长度一屏可读（≤80×24）；局部变量 ≤5-10 个；超过即应抽取 helper。但"长但简单"的函数（如长 switch-case）可保留——分解的判据是复杂度而非行数。

### 3.2 编译器 in-lining 比手工更高效（OS-KER-028）

**OS-KER-028**：相信编译器。若函数需要被 inline，使用 `static inline` 让编译器决定；编译器的 in-lining 策略（基于调用图、profile-guided 优化）远比人类判断精准。除非有实测证据，否则不写 `__always_inline`。

### 3.3 复杂条件编译不内嵌，抽出为独立 helper（OS-KER-029）

**OS-KER-029**：禁止在 `.c` 文件内用 `#ifdef` 切割函数体；条件编译应抽出到头文件中提供 stub，再由 `.c` 无条件调用。这样编译器仍可对 stub 调用做类型检查，逻辑可读性也得以保留。

```c
/* 错误：条件编译内嵌在函数体内 */
int airymaxos_init(void)
{
#ifdef CONFIG_AIRYMAXOS_IPC
	ipc_init();
#endif
	return do_init();
}

/* 正确：抽出 stub 到头文件 */
/* airymaxos_ipc.h */
#ifdef CONFIG_AIRYMAXOS_IPC
int ipc_init(void);
#else
static inline int ipc_init(void) { return 0; }
#endif

/* airymaxos_init.c */
int airymaxos_init(void)
{
	int ret = ipc_init();   /* 无条件调用 */
	if (ret)
		return ret;
	return do_init();
}
```

### 3.4 多用 `IS_ENABLED(CONFIG_XXX)` 而非 `#ifdef`（OS-KER-030）

**OS-KER-030**：在 `.c` 文件内优先使用 `if (IS_ENABLED(CONFIG_XXX))` 而非 `#ifdef`。编译器会做常量折叠，运行时零开销；同时让 C 编译器仍能对条件块内的代码做语法、类型、符号引用检查。

---

## 4. 数据结构设计风格

### 4.1 引用计数强制（OS-KER-031）

**OS-KER-031**：多线程可见的数据结构必须有引用计数。agentrt-linux 内核无 GC（垃圾回收），任何可被另一线程找到的数据结构都必须维护"使用者数量"——归零方可释放。agentrt-linux 的 E-3 资源确定性原则在此处落地。

```c
struct airymaxos_task_desc {
	struct kref	refcount;   /* 强制：多线程可见 */
	u64		task_id;
	/* ... */
};

static void desc_release(struct kref *ref)
{
	struct airymaxos_task_desc *desc =
		container_of(ref, struct airymaxos_task_desc, refcount);
	kfree(desc);
}

/* 获取：在传递指针前 kref_get；释放：用完后 kref_put */
```

### 4.2 锁不替代引用计数（OS-KER-032）

**OS-KER-032**：锁保证数据结构**一致性**（读写串行化），引用计数保证数据结构**生命周期**（不被释放）。两者职责不同，不可混淆。常见 bug：持锁睡眠时另一线程释放了被引用的数据结构。

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

### 4.3 灵活数组成员（OS-KER-033）

**OS-KER-033**：变长尾随数据使用 C99 灵活数组成员（`[]`），禁止"长度 1 数组"（`field[1]`）的旧 hack。配合 `kmalloc(struct_size(p, field, n), ...)` 自动做溢出检查。

```c
struct airymaxos_ipc_batch {
	u64		count;
	struct agentrt_ipc_msg	msg[];   /* 灵活数组成员 */
};

p = kmalloc(struct_size(p, msg, n), GFP_KERNEL);
```

### 4.4 位域与紧凑布局（OS-KER-034）

**OS-KER-034**：多个布尔值合并为 `bitfield`；对齐敏感的结构体不用 `bool`，改用 `u8` 或位域。理由：`bool` 在不同架构上的大小与对齐变化，会破坏 cache line 布局。

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

### 5.1 不维护稳定内核内部 API（OS-KER-035）

**OS-KER-035**：agentrt-linux 内核子系统之间的内部 API **不保证稳定**，这是 deliberate design decision（深思熟虑的设计决策）。允许持续重构是降低主线维护成本的核心机制——若每次重构都需兼顾二进制兼容，代码会迅速被历史包袱压垮。配套规则：API 改动者必须同时修复所有受影响代码（"you broke it, you fix it"）。

### 5.2 用户空间 ABI 必须稳定（OS-KER-036）

**OS-KER-036**：用户空间 ABI（系统调用、ioctl、文件系统接口、/proc、/sys、/dev 节点的语义）**永久稳定**。一旦接口导出至用户空间，必须永久支持；任何破坏性变更必须先经过弃用声明 + 6 个月宽限期，详见 04 工程思想 §6.2 的 4 层接口稳定性分级。

### 5.3 抽象只在必要时使用（OS-KER-037）

**OS-KER-037**：抽象是有成本的——一层间接调用、一份额外内存、一份认知负担。只有在"已经存在 3 处以上重复"或"已经能预见第二使用者"时才抽出抽象。空抽象比无抽象更糟——它会让后续维护者误以为存在多处使用者，不敢删除。

### 5.4 agentrt-linux 4 层接口稳定性分级（OS-STD-214）

**OS-STD-214**：agentrt-linux 在双层稳定性（内核内部 / 用户 ABI）基础上扩展为 4 层：

| 层 | 接口类型 | 稳定性 | 变更流程 |
|----|---------|--------|---------|
| L1 | Agent 应用 API（SDK + syscall） | 极稳定（类 Linux syscall） | RFC + 6 个月宽限 + 弃用声明 |
| L2 | AgentsIPC 协议（128B 头布局） | 中等稳定（版本化演进） | 季度评审 + 兼容性测试 |
| L3 | 内核子系统 API（内部 API） | 可重构（"you broke it, you fix it"） | 补丁序列修复所有调用点 |
| L4 | 内部实现细节 | 完全自由 | 无约束 |

---

## 6. 防御性编程风格

### 6.1 错误路径必须测试（OS-KER-038）

**OS-KER-038**：所有错误路径必须有对应测试用例。agentrt-linux 通过 fault injection 框架强制执行——`make kmalloc-fail` / `should_fail_alloc_page` 等机制可在运行时注入分配失败，验证错误路径是否真正可达且不破坏状态。任何未测试错误路径等同于未实现。

### 6.2 溢出检查（OS-KER-039）

**OS-KER-039**：所有用户可控长度参数必须经溢出检查后才能用于乘法。强制使用内核提供的安全算术：`struct_size()` / `array_size()` / `size_mul()` / `check_mul_overflow()`。

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

### 6.3 不假设指针非空——分级标签释放（OS-KER-040）

**OS-KER-040**：错误路径禁止假设指针非空，应使用**分级标签按分配逆序释放**。每个标签只释放一个资源；标签按"分配顺序"逆序排列。该模式可避免"one err bug"——所有释放路径都可达，无一例外。

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

### 6.4 `WARN_ON_ONCE` 而非反复 `WARN`（OS-KER-041）

**OS-KER-041**：检测到不变式破坏时使用 `WARN_ON_ONCE(cond)`，禁止 `WARN_ON(cond)` 或 `WARN()`。`ONCE` 变体保证同一不变式破坏只警告一次——若反复触发，日志会被淹没。配套：禁止 `BUG()` / `BUG_ON()`，因为内核崩溃决策权属于用户而非开发者。

---

## 7. 性能优化风格

### 7.1 反对 inline 滥用（OS-KER-042）

**OS-KER-042**：超过 3 行的函数不应 `inline`；`static` 且仅被一处调用的函数编译器会自动 inline，无需手工标注。理由：`inline` 滥用导致内核体积膨胀，icache footprint 增大，整体性能下降。一个 pagecache miss 引发 5ms 磁盘寻道，远比省一个函数调用代价大。

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

### 7.2 cache 影响是头等大事——空间即时间（OS-KER-043）

**OS-KER-043**：更大的程序运行更慢。代码与数据的空间局部性是性能首要因素——一个 cache line miss 等同上百个时钟周期。设计数据结构时优先考虑：是否可对齐到 cache line（64 字节）？是否可消除伪共享（不同 CPU 写入字段隔离到不同 cache line）？

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

### 7.3 内存分配惯用法——`sizeof(*p)`（OS-KER-044）

**OS-KER-044**：内存分配强制使用 `sizeof(*p)` 而非 `sizeof(struct xxx)`，配合 `kcalloc` / `kmalloc_array` 处理数组。理由：当指针类型变更时，`sizeof(*p)` 自动跟随，不会留下"忘了改 sizeof"的 bug。

```c
/* 错误：类型变更时易遗漏 */
p = kmalloc(sizeof(struct airymaxos_old_desc), GFP_KERNEL);

/* 正确：跟随指针类型 */
p = kmalloc(sizeof(*p), GFP_KERNEL);
arr = kcalloc(n, sizeof(*arr), GFP_KERNEL);
```

### 7.4 不滥用 printk——分级日志（OS-KER-045）

**OS-KER-045**：禁止单独使用 `printk`，必须用分级宏：设备相关用 `dev_err` / `dev_warn` / `dev_dbg`，设备无关用 `pr_err` / `pr_warn` / `pr_info` / `pr_debug`。`pr_debug` 在未定义 `DEBUG` 或未设置 `CONFIG_DYNAMIC_DEBUG` 时编译为空，零开销——这是热路径调试日志的正确方式。

### 7.5 锁设计必须前置（OS-KER-046）

**OS-KER-046**：锁设计必须在数据结构设计阶段就完成——不是"代码写完再加锁"。每条共享数据必须明确：哪个锁保护？锁的范围？锁内是否可睡眠？是否需引用计数配合？详见 04 工程思想 §3 与 80-testing 的 lockdep 强制检查。

---

## 8. agentrt-linux 专属风格

### 8.1 热路径 vs 冷路径分层（OS-KER-047）

**OS-KER-047**：热路径（airymaxos-kernel 调度、内存分配、IPC 收发）必须用 C 实现，且必须满足严格性能预算——单次调用 ≤1μs（64 核典型负载）。冷路径（配置解析、日志格式化、统计上报）可放宽 inline 限制，使用 Rust 实现以换取内存安全。

### 8.2 Rust 内核模块（OS-KER-048）

**OS-KER-048**：Rust 内核模块仅用于"安全敏感且非热路径"的子系统——典型如安全策略裁决（airymaxos-security LSM hook）、文件系统解析（路径遍历的字符串处理）、配置解析器。热路径禁用 Rust，避免 Rust 运行时与 panic 语义引入不可控延迟。

### 8.3 多语言协作风格——FFI 边界显式声明（OS-STD-215）

**OS-STD-215**：跨语言 FFI（Foreign Function Interface）边界必须显式声明——Rust ↔ C 边界标注 `extern "C"`、Python ↔ C 边界标注 `ctypes` 签名、TS ↔ Native 边界走 N-API。禁止隐式类型转换，所有跨边界数据必须经序列化或显式 layout 检查。

### 8.4 Agent 工作负载考量——Token 能效与记忆卷载（OS-STD-216）

**OS-STD-216**：agentrt-linux 设计须考虑 Agent 工作负载特性——Token 能效（每 token 的 CPU/J 预算）、记忆卷载（C-3，L1-L4 分层卷载模式）。详见 [40-dataflows/01-cognition-flow.md](../40-dataflows/01-cognition-flow.md) 与 [40-dataflows/02-memory-flow.md](../40-dataflows/02-memory-flow.md)。Agent 任务在调度器中应作为 `SCHED_AGENT` 类，与普通进程隔离调度，避免被普通负载饿死或反噬。

---

## 9. 工具链使用风格

### 9.1 gcc -W（OS-KER-049）

**OS-KER-049**：编译必须开启 `-Wall -Wextra -Werror`；新增警告项必须立即修复或显式 `// NOLINT: <reason>` 标注。零警告是 agentrt-linux 不可妥协的底线。

### 9.2 sparse——类型检查（OS-KER-050）

**OS-KER-050**：内核 C 代码必须通过 `make C=1`（sparse 检查）。sparse 检测 `__user` / `__kernel` / `__iomem` 等 address space 注解违反、endianness 错误、锁注解不一致——这些 gcc 不检查。新增涉及用户空间指针的代码必须含 `__user` 注解。

### 9.3 lockdep——锁依赖检查（OS-KER-051）

**OS-KER-051**：开发与 CI 配置 `CONFIG_LOCKDEP=y` + `CONFIG_PROVE_LOCKING=y`。lockdep 在运行时构建锁依赖图，检测死锁、锁序违反、加锁上下文不一致。任何 lockdep 警告视为 PR 阻断。

### 9.4 fault injection（OS-KER-052）

**OS-KER-052**：错误路径测试必须使用 fault injection 框架——`CONFIG_FAULT_INJECTION=y` + `CONFIG_FAILSLAB` / `CONFIG_FAIL_PAGE_ALLOC` / `CONFIG_FAIL_MAKE_REQUEST`。每个错误路径必须在 CI 中至少运行一次"分配全程失败"的测试用例。

### 9.5 checkpatch.pl（OS-STD-217）

**OS-STD-217**：每个 PR 提交前必须通过 `scripts/checkpatch.pl --strict`，无 ERROR / WARNING。checkpatch 检查 Linux 6.6 内核基线规则 + agentrt-linux 扩展规则（详见 06 卷 §3.2）。

### 9.6 多配置构建（OS-STD-218）

**OS-STD-218**：每个 PR 必须通过至少 3 种配置构建：`defconfig`（默认）、`allnoconfig`（最小）、`allmodconfig`（最大）。三者皆绿方可合并——`allnoconfig` 暴露 `#ifdef` 错配、`allmodconfig` 暴露符号冲突、`defconfig` 验证默认路径。

---

## 10. 五维原则映射

代码风格卷是 Airymax 五维正交 24 原则在工程判断层面的核心切片——大多数原则在此卷都有具体落地点：

| 原则 | 落地点 | 对应规则 |
|------|--------|----------|
| **A-1 极简主义** | 反过度抽象；反 inline 滥用；抽象只在必要时使用 | OS-KER-022、OS-KER-037、OS-KER-042 |
| **A-2 细节关注** | 分级标签按分配逆序释放；`sizeof(*p)` 跟随类型 | OS-KER-040、OS-KER-044 |
| **A-4 完美主义** | 零警告编译；多配置构建（allnoconfig + allmodconfig） | OS-KER-049、OS-STD-218 |
| **E-3 资源确定性** | 引用计数强制；锁不替代引用计数 | OS-KER-031、OS-KER-032 |
| **E-1 安全内生** | Rust 模块用于安全敏感路径；sparse 类型检查 | OS-KER-048、OS-KER-050 |
| **E-8 可测试性** | 错误路径必须测试；fault injection 强制 | OS-KER-038、OS-KER-052 |
| **K-2 接口契约化** | 4 层接口稳定性分级；用户 ABI 永久稳定 | OS-KER-036、OS-STD-214 |
| **K-3 服务隔离** | 策略外移至用户态守护进程 | OS-KER-025 |
| **K-4 可插拔策略** | 策略机制分离；sched_ext BPF 范式 | OS-KER-024、OS-KER-026 |
| **C-1 双系统协同** | 热路径 C / 慢路径 Rust / Agent Python+TS 分层 | OS-STD-213、OS-KER-047 |
| **C-3 记忆卷载** | Agent 工作负载考量，Token 能效与 L1-L4 卷载 | OS-STD-216 |

---

## 11. 相关文档

- [工程标准 README](README.md)：工程标准规范主索引与总纲
- [01 代码规范](01-coding-standards.md)：命名、函数、注释、类型、错误处理
- [02 代码格式](02-code-format.md)：缩进、行宽、大括号、空格、clang-format
- [04 工程思想](04-engineering-philosophy.md)：双层稳定性哲学、策略机制分离、渐进式
- [06 工具链与自动化](06-toolchain-and-automation.md)：checkpatch、sparse、lockdep 的具体配置
- [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md)：五维正交 24 原则完整定义
- [20-modules/01-kernel.md](../20-modules/01-kernel.md)：airymaxos-kernel 模块设计
- Linux 6.6 内核基线 `Documentation/process/coding-style.rst` §6~§22 与 `stable-api-nonsense.rst` 为同源参考

---

## 12. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-06 | 初始版本（含 §1-§10 全部规则 + 11 条五维映射 + agentrt-linux 专属风格） |
| 1.0.1 | TBD | 与代码实现同步验证；Agent 工作负载 Token 能效基准确立 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
