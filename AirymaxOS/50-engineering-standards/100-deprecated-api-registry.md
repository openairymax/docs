Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）废弃 API 动态清单

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）内核态废弃 API 动态清单。基于 OLK-6.6 `Documentation/process/deprecated.rst` 与 `scripts/checkpatch.pl` 中的 `deprecated_apis` 表，登记所有禁止新增的 API 及其替代方案与迁移示例。
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 目的与范围

本文件登记 agentrt-liunx 内核态所有禁止新增的废弃 API，每条含：废弃 API → 替代 API → OLK-6.6 源码路径 → 迁移示例。checkpatch 在 CI 中根据此清单与新代码扫描，禁止新增废弃 API 调用。

### 0.2 OLK-6.6 源码路径

- `Documentation/process/deprecated.rst`（16575 字节，废弃接口说明文档）
- `scripts/checkpatch.pl:836-851` —— `%deprecated_apis` 哈希表（11 条 RCU/kmap 废弃映射）
- `scripts/checkpatch.pl:854-859` —— `$deprecated_apis_search` 搜索模式构建
- `scripts/checkpatch.pl:6456-6458` —— `ERROR("DEPRECATED_API")` 单参 `k[v]free_rcu()` 检查
- `scripts/checkpatch.pl:7029-7043` —— `STRCPY`/`STRLCPY`/`STRNCPY` 检查
- `scripts/checkpatch.pl:7424` —— `WARN("DEPRECATED_API")` 通用废弃检查

### 0.3 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-liunx（OS 发行版）= 微内核（micro-kernel）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

---

## 1. 字符串处理 API

### 1.1 strcpy() → strscpy()

| 字段 | 内容 |
|------|------|
| **废弃 API** | `strcpy()` |
| **替代 API** | `strscpy()`（需 NUL 终止）/ `strscpy_pad()`（需 NUL 填充）/ `strtomem()`（非 NUL 终止） |
| **OLK-6.6 源码路径** | `deprecated.rst:122-132` / `checkpatch.pl:7029-7031` |
| **废弃理由** | `strcpy` 无边界检查，可导致线性堆溢出。`CONFIG_FORTIFY_SOURCE=y` 仅减风险不根除。 |
| **checkpatch 规则** | `WARN("STRCPY")` |

**迁移示例**：

```c
/* 废弃：strcpy 无边界检查 */
strcpy(chan->name, user_name);

/* 替代：strscpy 返回已复制字节数，截断返回负 errno */
ssize_t copied = strscpy(chan->name, user_name, sizeof(chan->name));
if (copied < 0)
	return -E2BIG;
```

### 1.2 strncpy() → strscpy() / strscpy_pad()

| 字段 | 内容 |
|------|------|
| **废弃 API** | `strncpy()` |
| **替代 API** | `strscpy()` / `strscpy_pad()` / `strtomem()` / `strtomem_pad()` |
| **OLK-6.6 源码路径** | `deprecated.rst:134-154` / `checkpatch.pl:7041-7043` |
| **废弃理由** | `strncpy` 不保证 NUL 终止，且源短时多余 NUL 填充浪费性能。 |
| **checkpatch 规则** | `WARN("STRNCPY")` |

**迁移示例**：

```c
/* 废弃：strncpy 不保证 NUL 终止 */
strncpy(buf, src, sizeof(buf));

/* 替代：需 NUL 终止用 strscpy */
strscpy(buf, src, sizeof(buf));

/* 替代：需 NUL 填充用 strscpy_pad */
strscpy_pad(buf, src, sizeof(buf));

/* 替代：非 NUL 终止用 strtomem（需 __nonstring 标注） */
strtomem(buf, src);
```

### 1.3 strlcpy() → strscpy()

| 字段 | 内容 |
|------|------|
| **废弃 API** | `strlcpy()` |
| **替代 API** | `strscpy()` |
| **OLK-6.6 源码路径** | `deprecated.rst:156-163` / `checkpatch.pl:7035-7037` |
| **废弃理由** | `strlcpy` 为返回 strlen() 会读整个源缓冲，可能溢出读。 |
| **checkpatch 规则** | `WARN("STRLCPY")` |

**迁移示例**：

```c
/* 废弃：strlcpy 多余读源 */
strlcpy(dst, src, sizeof(dst));

/* 替代：strscpy 仅读至 dst 大小 */
strscpy(dst, src, sizeof(dst));
```

### 1.4 simple_strtol() / simple_strtoul() → kstrtol() / kstrtoul()

| 字段 | 内容 |
|------|------|
| **废弃 API** | `simple_strtol()` / `simple_strtoll()` / `simple_strtoul()` / `simple_strtoull()` |
| **替代 API** | `kstrtol()` / `kstrtoll()` / `kstrtoul()` / `kstrtoull()` |
| **OLK-6.6 源码路径** | `deprecated.rst:112-120` |
| **废弃理由** | `simple_strtol` 系列显式忽略溢出，可能产生意外结果。`kstrtol` 系列要求 NUL 或换行终止。 |
| **checkpatch 规则** | `WARN("SSCANF_TO_KSTRTO")`（间接） |

**迁移示例**：

```c
/* 废弃：simple_strtol 忽略溢出 */
long val = simple_strtol(buf, &end, 10);

/* 替代：kstrtol 检查溢出与终止符 */
long val;
int ret = kstrtol(buf, 10, &val);
if (ret)
	return ret;
```

---

## 2. 内存分配 API

### 2.1 open-coded arithmetic in allocator → kmalloc_array / kcalloc / struct_size

| 字段 | 内容 |
|------|------|
| **废弃 API** | `kmalloc(n * size, ...)` / `kzalloc(sizeof(*h) + count * sizeof(*h->item), ...)` |
| **替代 API** | `kmalloc_array(n, size, ...)` / `kcalloc(n, size, ...)` / `struct_size(p, member, n)` / `flex_array_size()` |
| **OLK-6.6 源码路径** | `deprecated.rst:54-110` / `checkpatch.pl:7255` |
| **废弃理由** | 乘法在分配器参数中可能溢出，导致分配过小并线性堆溢出。 |
| **checkpatch 规则** | `WARN("ALLOC_WITH_MULTIPLY")` |

**迁移示例**：

```c
/* 废弃：乘法可能溢出 */
buf = kmalloc(count * size, GFP_KERNEL);

/* 替代：kmalloc_array 内置溢出检查 */
buf = kmalloc_array(count, size, GFP_KERNEL);

/* 替代：需零初始化用 kcalloc */
buf = kcalloc(count, size, GFP_KERNEL);

/* 废弃：结构体 + 尾随数组 */
h = kzalloc(sizeof(*h) + count * sizeof(*h->item), GFP_KERNEL);

/* 替代：struct_size + 柔性数组成员 */
struct h {
	size_t count;
	struct item items[];	/* C99 柔性数组成员，非 [0] 或 [1] */
};
h = kzalloc(struct_size(h, items, count), GFP_KERNEL);
```

### 2.2 sizeof(struct xxx) → sizeof(*p)

| 字段 | 内容 |
|------|------|
| **废弃 API** | `kmalloc(sizeof(struct xxx), ...)` |
| **替代 API** | `kmalloc(sizeof(*p), ...)` |
| **OLK-6.6 源码路径** | `coding-style.rst:925-933` / `checkpatch.pl:7229` |
| **废弃理由** | 指针类型变更时 `sizeof(struct)` 不会跟随，产生隐蔽 bug。 |
| **checkpatch 规则** | `CHK("ALLOC_SIZEOF_STRUCT")` |

**迁移示例**：

```c
/* 废弃：类型变更时不跟随 */
struct agentrt_task *task = kmalloc(sizeof(struct agentrt_task), GFP_KERNEL);

/* 替代：sizeof(*p) 自动跟随类型 */
struct agentrt_task *task = kmalloc(sizeof(*task), GFP_KERNEL);
```

### 2.3 memset(p, 0, sizeof(*p)) → kzalloc

| 字段 | 内容 |
|------|------|
| **废弃 API** | `p = kmalloc(...); memset(p, 0, sizeof(*p));` |
| **替代 API** | `p = kzalloc(...)` |
| **OLK-6.6 源码路径** | `checkpatch.pl:6978-6981` |
| **废弃理由** | `kzalloc` 一步完成分配 + 清零，避免 `memset` 的双重遍历。 |
| **checkpatch 规则** | `WARN("MEMSET")` |

**迁移示例**：

```c
/* 废弃：分配后 memset 清零 */
task = kmalloc(sizeof(*task), GFP_KERNEL);
if (!task)
	return -ENOMEM;
memset(task, 0, sizeof(*task));

/* 替代：kzalloc 一步完成 */
task = kzalloc(sizeof(*task), GFP_KERNEL);
if (!task)
	return -ENOMEM;
```

### 2.4 kmap / kmap_atomic → kmap_local_page

| 字段 | 内容 |
|------|------|
| **废弃 API** | `kmap()` / `kmap_atomic()` / `kunmap()` / `kunmap_atomic()` |
| **替代 API** | `kmap_local_page()` / `kunmap_local()` |
| **OLK-6.6 源码路径** | `checkpatch.pl:847-850` |
| **废弃理由** | `kmap` 系列在 32 位上有全局锁竞争，`kmap_local_page` 无此问题。 |
| **checkpatch 规则** | `ERROR("DEPRECATED_API")` |

**迁移示例**：

```c
/* 废弃：kmap 全局锁竞争 */
void *addr = kmap(page);
memcpy(addr, src, len);
kunmap(page);

/* 替代：kmap_local_page 无全局锁 */
void *addr = kmap_local_page(page);
memcpy(addr, src, len);
kunmap_local(addr);
```

### 2.5 krealloc 重用原指针 → 临时变量

| 字段 | 内容 |
|------|------|
| **废弃 API** | `p = krealloc(p, new_size, GFP_KERNEL);` |
| **替代 API** | 用临时变量保存 krealloc 返回值，失败时不丢失原 p |
| **OLK-6.6 源码路径** | `checkpatch.pl:7268` |
| **废弃理由** | krealloc 失败返回 NULL 但不释放原 p，重用原 p 变量会丢失原指针导致泄漏。 |
| **checkpatch 规则** | `WARN("KREALLOC_ARG_REUSE")` |

**迁移示例**：

```c
/* 废弃：失败时丢失原 buf */
buf = krealloc(buf, new_size, GFP_KERNEL);
if (!buf)
	return -ENOMEM;	/* 原 buf 泄漏！ */

/* 替代：用临时变量 */
void *new_buf = krealloc(buf, new_size, GFP_KERNEL);
if (!new_buf) {
	kfree(buf);		/* 显式释放原 buf */
	return -ENOMEM;
}
buf = new_buf;
```

---

## 3. 控制流与断言 API

### 3.1 BUG() / BUG_ON() → WARN() / WARN_ON_ONCE()

| 字段 | 内容 |
|------|------|
| **废弃 API** | `BUG()` / `BUG_ON()` / `VM_BUG_ON()` |
| **替代 API** | `WARN()` / `WARN_ON()` / `WARN_ON_ONCE()`（首选） |
| **OLK-6.6 源码路径** | `deprecated.rst:32-52` / `coding-style.rst:1189-1249` |
| **废弃理由** | `BUG()` 系列导致内核崩溃，无法调试且破坏系统。`WARN` 系列仅日志不崩溃。 |
| **checkpatch 规则** | （间接，通过 deprecated API 登记） |

**迁移示例**：

```c
/* 废弃：BUG_ON 导致内核崩溃 */
BUG_ON(!ptr);
BUG_ON(x > MAX);

/* 替代：WARN_ON_ONCE 仅日志 + 提供恢复 */
if (WARN_ON_ONCE(!ptr))
	return -EINVAL;
if (WARN_ON_ONCE(x > MAX))
		x = MAX;	/* 恢复 */
```

### 3.2 单参 k[v]free_rcu() → 双参或 k[v]free_rcu_mightsleep()

| 字段 | 内容 |
|------|------|
| **废弃 API** | `kfree_rcu(ptr)` / `kvfree_rcu(ptr)`（单参数） |
| **替代 API** | `kfree_rcu(ptr, rcu_head)` / `kvfree_rcu(ptr, rcu_head)` 或 `kfree_rcu_mightsleep()` |
| **OLK-6.6 源码路径** | `checkpatch.pl:6454-6458` |
| **废弃理由** | 单参数 API 已废弃，应显式传递 rcu_head 或用 mightsleep 变体。 |
| **checkpatch 规则** | `ERROR("DEPRECATED_API")` |

**迁移示例**：

```c
/* 废弃：单参数 kfree_rcu */
kfree_rcu(obj);

/* 替代：双参数（obj 必须含 rcu_head 成员） */
kfree_rcu(obj, rcu_head);

/* 替代：可睡眠上下文用 mightsleep */
kfree_rcu_mightsleep(obj);
```

### 3.3 panic() → 仅限启动期

| 字段 | 内容 |
|------|------|
| **废弃 API** | `panic()`（运行期） |
| **替代 API** | `WARN()` + 恢复代码；仅启动期 OOM 等无可恢复场景可用 `panic()` |
| **OLK-6.6 源码路径** | `coding-style.rst:1195-1201` |
| **废弃理由** | 运行期 panic 剥夺用户决策权，崩溃决策应属于用户。 |
| **checkpatch 规则** | （审查提示，非自动检查） |

---

## 4. 类型与数组 API

### 4.1 零长 / 单元素数组 → C99 柔性数组成员

| 字段 | 内容 |
|------|------|
| **废弃 API** | `struct foo { ...; struct bar items[0]; };` / `items[1]` |
| **替代 API** | `struct foo { ...; struct bar items[]; };`（C99 柔性数组成员） |
| **OLK-6.6 源码路径** | `deprecated.rst:237-348` / `checkpatch.pl:7479` |
| **废弃理由** | 零长/单元素数组 sizeof 计算错误，编译器无法检测非末尾使用。柔性数组成员 sizeof 不完整类型，可检测误用。 |
| **checkpatch 规则** | `ERROR("FLEXIBLE_ARRAY")` |

**迁移示例**：

```c
/* 废弃：零长数组 */
struct msg_queue {
	size_t count;
	struct msg entries[0];	/* WRONG */
};

/* 废弃：单元素数组 */
struct msg_queue {
	size_t count;
	struct msg entries[1];	/* WRONG，sizeof 多算一个 */
};

/* 替代：C99 柔性数组成员 */
struct msg_queue {
	size_t count;
	struct msg entries[];	/* OK */
};

/* 分配：用 struct_size 计算 */
q = kzalloc(struct_size(q, entries, count), GFP_KERNEL);
```

### 4.2 union 中零长数组 → DECLARE_FLEX_ARRAY

| 字段 | 内容 |
|------|------|
| **废弃 API** | union 中 `items[0]` |
| **替代 API** | `DECLARE_FLEX_ARRAY(type, name)` |
| **OLK-6.6 源码路径** | `deprecated.rst:350-373` |
| **废弃理由** | union 中的零长数组违反 C99，需用 `DECLARE_FLEX_ARRAY` 辅助宏。 |

**迁移示例**：

```c
/* 废弃：union 中零长数组 */
struct payload {
	union {
		struct type1 a[0];
		struct type2 b[0];
	};
};

/* 替代：DECLARE_FLEX_ARRAY */
struct payload {
	union {
		DECLARE_FLEX_ARRAY(struct type1, a);
		DECLARE_FLEX_ARRAY(struct type2, b);
	};
};
```

### 4.3 typedef 结构体 → struct 直接使用

| 字段 | 内容 |
|------|------|
| **废弃 API** | `typedef struct { ... } foo_t;` |
| **替代 API** | `struct foo { ... };` 直接用 `struct foo` |
| **OLK-6.6 源码路径** | `coding-style.rst:359-440` / `checkpatch.pl:4788` |
| **废弃理由** | typedef 结构体隐藏类型，降低可读性。 |
| **checkpatch 规则** | `WARN("NEW_TYPEDEFS")` |

**迁移示例**：

```c
/* 废弃：结构体 typedef */
typedef struct {
	u64 id;
} agentrt_task_t;

/* 替代：struct 直接使用 */
struct agentrt_task {
	u64 id;
};
```

---

## 5. 并发与同步 API

### 5.1 volatile → READ_ONCE / WRITE_ONCE

| 字段 | 内容 |
|------|------|
| **废弃 API** | `volatile int flag;` |
| **替代 API** | `int flag;` + `READ_ONCE(flag)` / `WRITE_ONCE(flag, val)` |
| **OLK-6.6 源码路径** | `checkpatch.pl:6275` |
| **废弃理由** | `volatile` 不提供内存屏障，且阻止编译器优化。`READ_ONCE`/`WRITE_ONCE` 显式且正确。 |
| **checkpatch 规则** | `WARN("VOLATILE")` |

**迁移示例**：

```c
/* 废弃：volatile */
volatile int ready = 0;
while (!ready)
	cpu_relax();

/* 替代：READ_ONCE / WRITE_ONCE */
int ready = 0;
while (!READ_ONCE(ready))
	cpu_relax();
WRITE_ONCE(ready, 1);
```

### 5.2 smp_mb / smp_rmb / smp_wmb → smp_store_release / smp_load_acquire

| 字段 | 内容 |
|------|------|
| **废弃 API** | `smp_mb()` / `smp_rmb()` / `smp_wmb()` |
| **替代 API** | `smp_store_release()` / `smp_load_acquire()` |
| **OLK-6.6 源码路径** | `checkpatch.pl:6695,6706` |
| **废弃理由** | 全屏障开销大，release/acquire 语义更精确且性能更佳。 |
| **checkpatch 规则** | `WARN("MEMORY_BARRIER")` |

**迁移示例**：

```c
/* 废弃：全屏障 */
data = compute();
smp_wmb();
ready = 1;

/* 替代：release 语义 */
data = compute();
smp_store_release(&ready, 1);
```

---

## 6. RCU API

### 6.1 synchronize_rcu_bh / call_rcu_bh → synchronize_rcu / call_rcu

| 字段 | 内容 |
|------|------|
| **废弃 API** | `synchronize_rcu_bh()` / `synchronize_rcu_bh_expedited()` / `call_rcu_bh()` / `rcu_barrier_bh()` |
| **替代 API** | `synchronize_rcu()` / `synchronize_rcu_expedited()` / `call_rcu()` / `rcu_barrier()` |
| **OLK-6.6 源码路径** | `checkpatch.pl:837-840` |
| **废弃理由** | bh 变体已合并入普通 RCU API。 |
| **checkpatch 规则** | `ERROR("DEPRECATED_API")` |

**迁移示例**：

```c
/* 废弃：bh 变体 */
synchronize_rcu_bh();
call_rcu_bh(&obj->rcu, obj_free);

/* 替代：普通 RCU API */
synchronize_rcu();
call_rcu(&obj->rcu, obj_free);
```

### 6.2 synchronize_sched / call_rcu_sched → synchronize_rcu / call_rcu

| 字段 | 内容 |
|------|------|
| **废弃 API** | `synchronize_sched()` / `synchronize_sched_expedited()` / `call_rcu_sched()` / `rcu_barrier_sched()` / `get_state_synchronize_sched()` / `cond_synchronize_sched()` |
| **替代 API** | `synchronize_rcu()` / `synchronize_rcu_expedited()` / `call_rcu()` / `rcu_barrier()` / `get_state_synchronize_rcu()` / `cond_synchronize_rcu()` |
| **OLK-6.6 源码路径** | `checkpatch.pl:841-846` |
| **废弃理由** | sched 变体已合并入普通 RCU API。 |
| **checkpatch 规则** | `ERROR("DEPRECATED_API")` |

---

## 7. 打印与格式化 API

### 7.1 sprintf / scnprintf → scnprintf

| 字段 | 内容 |
|------|------|
| **废弃 API** | `sprintf()` |
| **替代 API** | `scnprintf()`（含边界检查）/ `sysfs_emit()`（sysfs） |
| **OLK-6.6 源码路径** | `checkpatch.pl:7469`（sysfs_emit 检查） |
| **废弃理由** | `sprintf` 无边界检查，可溢出。 |
| **checkpatch 规则** | `WARN("SYSFS_EMIT")`（sysfs 场景） |

**迁移示例**：

```c
/* 废弃：sprintf 无边界检查 */
sprintf(buf, "id=%llu", task->id);

/* 替代：scnprintf 含边界检查 */
scnprintf(buf, sizeof(buf), "id=%llu", task->id);

/* 替代：sysfs 场景用 sysfs_emit */
static ssize_t id_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "%llu\n", task->id);
}
```

### 7.2 printk 无 KERN_LEVEL → pr_* / dev_*

| 字段 | 内容 |
|------|------|
| **废弃 API** | `printk("msg")`（无 KERN_LEVEL） |
| **替代 API** | `pr_info()` / `pr_warn()` / `pr_err()` / `dev_err()` 等 |
| **OLK-6.6 源码路径** | `checkpatch.pl:4870` |
| **废弃理由** | 无 KERN_LEVEL 的 printk 无法被动态调试控制。 |
| **checkpatch 规则** | `WARN("PRINTK_WITHOUT_KERN_LEVEL")` / `WARN("PREFER_PR_LEVEL")` / `WARN("PREFER_DEV_LEVEL")` |

---

## 8. 杂项 API

### 8.1 udelay > 10us → usleep_range / msleep

| 字段 | 内容 |
|------|------|
| **废弃 API** | `udelay(usec)`（usec > 10） |
| **替代 API** | `usleep_range(min, max)` / `msleep(msec)` |
| **OLK-6.6 源码路径** | `checkpatch.pl:6616,6620,6628` |
| **废弃理由** | `udelay` 忙等，长延迟应让出 CPU。 |
| **checkpatch 规则** | `CHK("USLEEP_RANGE")` / `WARN("LONG_UDELAY")` / `WARN("MSLEEP")` |

### 8.2 naked sscanf → kstrto*

| 字段 | 内容 |
|------|------|
| **废弃 API** | `sscanf(buf, "%d", &val)` |
| **替代 API** | `kstrtoint()` / `kstrtoull()` 等 |
| **OLK-6.6 源码路径** | `checkpatch.pl:7112` |
| **废弃理由** | `sscanf` 无溢出检查，`kstrto*` 系列更安全。 |
| **checkpatch 规则** | `WARN("SSCANF_TO_KSTRTO")` |

### 8.3 __deprecated 属性 → 彻底移除或登记

| 字段 | 内容 |
|------|------|
| **废弃 API** | `__deprecated` 属性标注 |
| **替代 API** | 彻底移除接口，或登记到本清单 |
| **OLK-6.6 源码路径** | `deprecated.rst:20-30` |
| **废弃理由** | `__deprecated` 不再产生构建警告，标注无意义。 |

---

## 9. 废弃 API 速查表

| 废弃 API | 替代 API | OLK-6.6 主源码 | checkpatch 规则 |
|---------|---------|---------------|----------------|
| `strcpy` | `strscpy` | `deprecated.rst:122` | `STRCPY` |
| `strncpy` | `strscpy`/`strscpy_pad` | `deprecated.rst:134` | `STRNCPY` |
| `strlcpy` | `strscpy` | `deprecated.rst:156` | `STRLCPY` |
| `simple_strtol` | `kstrtol` | `deprecated.rst:112` | — |
| `kmalloc(n*size)` | `kmalloc_array`/`kcalloc` | `deprecated.rst:54` | `ALLOC_WITH_MULTIPLY` |
| `sizeof(struct)` | `sizeof(*p)` | `coding-style.rst:925` | `ALLOC_SIZEOF_STRUCT` |
| `kmalloc+memset` | `kzalloc` | `checkpatch.pl:6978` | `MEMSET` |
| `kmap`/`kmap_atomic` | `kmap_local_page` | `checkpatch.pl:847` | `DEPRECATED_API` |
| `krealloc(p,...)` | 临时变量 | `checkpatch.pl:7268` | `KREALLOC_ARG_REUSE` |
| `BUG`/`BUG_ON` | `WARN`/`WARN_ON_ONCE` | `deprecated.rst:32` | — |
| `kfree_rcu(ptr)` | 双参/mightsleep | `checkpatch.pl:6456` | `DEPRECATED_API` |
| `panic`（运行期） | `WARN`+恢复 | `coding-style.rst:1195` | — |
| `items[0]`/`items[1]` | `items[]` | `deprecated.rst:237` | `FLEXIBLE_ARRAY` |
| `union{items[0]}` | `DECLARE_FLEX_ARRAY` | `deprecated.rst:350` | `FLEXIBLE_ARRAY` |
| `typedef struct` | `struct` 直接用 | `coding-style.rst:359` | `NEW_TYPEDEFS` |
| `volatile` | `READ_ONCE`/`WRITE_ONCE` | `checkpatch.pl:6275` | `VOLATILE` |
| `smp_mb`/`smp_rmb`/`smp_wmb` | `smp_store_release`/`smp_load_acquire` | `checkpatch.pl:6695` | `MEMORY_BARRIER` |
| `synchronize_rcu_bh` | `synchronize_rcu` | `checkpatch.pl:837` | `DEPRECATED_API` |
| `synchronize_sched` | `synchronize_rcu` | `checkpatch.pl:841` | `DEPRECATED_API` |
| `sprintf` | `scnprintf`/`sysfs_emit` | `checkpatch.pl:7469` | `SYSFS_EMIT` |
| `printk` 无 level | `pr_*`/`dev_*` | `checkpatch.pl:4870` | `PRINTK_WITHOUT_KERN_LEVEL` |
| `udelay > 10us` | `usleep_range`/`msleep` | `checkpatch.pl:6616` | `USLEEP_RANGE` |
| `sscanf` | `kstrto*` | `checkpatch.pl:7112` | `SSCANF_TO_KSTRTO` |
| `__deprecated` | 移除或登记 | `deprecated.rst:20` | — |

---

## 10. 动态更新与 CI 集成

### 10.1 清单动态更新流程

本清单随 OLK-6.6 升级与 agentrt-liunx 演进动态更新：

1. OLK-6.6 `deprecated.rst` 或 `checkpatch.pl` 新增废弃 API → 触发清单更新 MR
2. agentrt-liunx 内部废弃某 API → 经 L3 总维护者批准后登记
3. 每次更新在"历史与变更记录"追加条目

### 10.2 CI 集成

CI 在每次合并请求中运行 checkpatch，根据本清单与 `scripts/checkpatch.pl:836-851` 的 `%deprecated_apis` 表扫描新代码：

```bash
$ ./scripts/checkpatch.pl --strict --no-tree \
    $(git diff origin/main...HEAD --name-only -- '*.c' '*.h')
```

任何 `DEPRECATED_API` ERROR 或 `STRCPY`/`STRLCPY`/`STRNCPY` WARNING 即拒绝合并。

---

## 11. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，覆盖 24 条废弃 API 与替代方案 | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
