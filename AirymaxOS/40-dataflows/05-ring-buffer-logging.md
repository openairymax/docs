Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 零拷贝 Ring Buffer 日志数据流
> **文档定位**：A-ULP（统一日志与打印系统）零拷贝 Ring Buffer 日志数据流的唯一权威设计\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）+ §6.2.1 C-07（DMA 一致性内存修正）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax 日志 Ring Buffer 数据流** 的唯一权威源。内存方案（alloc_pages + mmap）、无锁 Ring Buffer 设计（reserve/commit 两阶段）、Fastpath 路径、eventfd 通知、Logger Daemon 分离、Panic 回退路径均以本文件为唯一权威定义。
>
> 技术选型声明：日志内存采用 **`alloc_pages(GFP_KERNEL)` + mmap**（**不使用 DMA 一致性内存**，x86_64 默认缓存一致）。本数据流遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) 的 A-ULP 模块设计与 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 的 128B 记录格式。整体技术选型对齐 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

---

## 文档信息卡

- **目标读者**：agentrt-linux 内核开发者、Logger Daemon 开发者、性能工程师
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) A-ULP 模块、[09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 128B 记录格式
- **预计阅读时间**：30 分钟
- **核心概念**：alloc_pages + mmap、无锁 Ring Buffer、reserve/commit 两阶段、Fastpath、eventfd、Logger Daemon、Panic 回退
- **复杂度标识**：高级

---

## §1 内存方案：alloc_pages(GFP_KERNEL) + mmap

### 1.1 为什么不用 DMA 一致性内存

综合修正方案 §6.2.1 C-07 指出，14 个文件误用 DMA 一致性内存（`dma_alloc_coherent`）用于日志/IPC 共享内存。这是错误的，原因有二：

1. **场景不匹配**：DMA 一致性内存的设计目标是为 DMA 设备（网卡、磁盘控制器）提供与 CPU 缓存一致的内存映射。日志场景的生产者是 CPU（内核态），消费者也是 CPU（用户态 Logger Daemon），**没有 DMA 设备参与**，使用 DMA 一致性内存是语义错配。
2. **性能损耗**：在 x86_64 架构下，CPU 默认缓存一致（cache-coherent），`dma_alloc_coherent` 会额外设置 `PAGE_KERNEL_NOCACHE` 或类似属性，禁用 cache 反而**降低** CPU 访问性能。

### 1.2 正确方案：alloc_pages + mmap

A-ULP 采用 `alloc_pages(GFP_KERNEL)` 分配连续物理页 + `remap_pfn_range` 映射到用户态：

```c
/* kernel/log/airy_log_ring.c —— 内存分配 */
struct airy_log_ring {
    struct page *pages;         /* 底层物理页 */
    void        *kaddr;         /* 内核态虚拟地址 */
    size_t       size;          /* Ring 总大小（页对齐） */
    unsigned int order;         /* alloc_pages order */
};

static int airy_log_ring_alloc(struct airy_log_ring *ring, size_t size)
{
    ring->order = get_order(size);
    ring->size  = PAGE_SIZE << ring->order;
    /* GFP_KERNEL：普通内核分配，可睡眠、可回收 —— 日志场景适用 */
    ring->pages = alloc_pages(GFP_KERNEL | __GFP_ZERO, ring->order);
    if (!ring->pages)
        return -ENOMEM;
    ring->kaddr = page_address(ring->pages);
    return 0;
}

/* mmap 到用户态 Logger Daemon */
static int airy_log_ring_mmap(struct file *f, struct vm_area_struct *vma)
{
    struct airy_log_ring *ring = f->private_data;
    vm_flags_set(vma, VM_SHARED | VM_DONTEXPAND | VM_DONTDUMP);
    return remap_pfn_range(vma, vma->vm_start,
                           page_to_pfn(ring->pages),
                           vma->vm_end - vma->vm_start,
                           vma->vm_page_prot);
}
```

### 1.3 缓存一致性保证

x86_64 架构下，CPU 缓存默认一致（cache-coherent），内核生产者写入 `kaddr` 后，用户态消费者通过 mmap 映射的虚拟地址读取，能看到最新数据。可见性通过以下机制保证：

- **x86 内存模型**：x86 是 TSO（Total Store Order），store 后的 load 在其他 CPU 立即可见（经过 store buffer 刷新）
- **原子索引**：生产者 commit 时使用 `smp_store_release()`，消费者读取索引使用 `smp_load_acquire()`，确保 128B 记录写入在索引更新之前完成
- **无需 cache flush**：因为缓存一致，无需调用 `flush_cache_range()` 等昂贵操作

在非缓存一致架构（如部分 ARM）上，需在 commit 时增加 `flush_dcache_page()`，但 Airymax 当前定位 x86_64 优先，后续扩展时再处理。

---

## §2 无锁 Ring Buffer 设计：reserve/commit 两阶段写入、原子索引

### 2.1 两阶段写入模型

A-ULP 的 Ring Buffer 采用 **reserve/commit 两阶段写入**模型，避免生产者持锁期间被中断/抢占导致死锁：

```
生产者（内核态）                  消费者（Logger Daemon）
┌──────────────────────┐        ┌──────────────────────┐
│ 1. reserve(slot)     │        │ 1. 读取 commit 索引   │
│    → 获得 slot 偏移  │        │ 2. 消费 [read, commit)│
│ 2. memcpy 128B 到 slot│        │ 3. 更新 read 索引     │
│ 3. commit(slot)      │        │ 4. eventfd 通知已消费 │
│    → 更新 commit 索引│        │                      │
└──────────────────────┘        └──────────────────────┘
```

### 2.2 原子索引

Ring Buffer 使用两个原子索引：

- `head`：生产者 reserve 位置（仅生产者写，消费者读）
- `tail`：消费者 read 位置（仅消费者写，生产者读）

```c
struct airy_log_ring_header {
    __u64   head;           /* 生产者 reserve 位置（原子） */
    __u64   tail;           /* 消费者 read 位置（原子） */
    __u32   capacity;       /* Ring 容量（记录数） */
    __u32   record_size;    /* 单条记录大小（128B） */
    __u32   frozen;         /* 冻结标志（Micro-Supervisor 设置） */
    __u32   pad;
};
```

### 2.3 reserve 实现

```c
/* kernel/log/airy_log_ring.c —— reserve */
static struct airy_log_record *airy_log_reserve(struct airy_log_ring *ring)
{
    struct airy_log_ring_header *hdr = ring->kaddr;
    __u64 head, tail;
    __u32 idx;

    /* unlikely 检查：99%+ 不冻结 */
    if (unlikely(hdr->frozen))
        return NULL;   /* 返回 NULL，调用方走 printk_safe 回退 */

    do {
        head = smp_load_acquire(&hdr->head);
        tail = smp_load_acquire(&hdr->tail);
        /* Ring 满：head - tail == capacity */
        if (head - tail >= hdr->capacity) {
            /* 丢弃最旧记录（覆盖写）或返回 NULL（丢弃写） */
            smp_store_release(&hdr->tail, tail + 1);
        }
        idx = head % hdr->capacity;
    } while (cmpxchg(&hdr->head, head, head + 1) != head);

    return (struct airy_log_record *)(ring->kaddr + sizeof(*hdr)
            + idx * hdr->record_size);
}
```

### 2.4 commit 实现

```c
/* kernel/log/airy_log_ring.c —— commit */
static void airy_log_commit(struct airy_log_ring *ring, struct airy_log_record *rec)
{
    struct airy_log_ring_header *hdr = ring->kaddr;
    /* smp_store_release 确保 128B 写入在 head 更新前完成 */
    smp_store_release(&hdr->head, hdr->head);  /* head 已在 reserve 时自增 */
    /* eventfd 通知 Logger Daemon（非阻塞） */
    eventfd_signal(ring->efd, 1);
}
```

---

## §3 Fastpath 路径：仅 memcpy 128B raw binary（~20ns 裸 memcpy）

### 3.1 Fastpath 三步

A-ULP 的 Fastpath 严格三步，禁止任何格式化操作：

```c
/* kernel/log/airy_log_ring.c —— Fastpath */
static inline int airy_log_write(struct airy_log_ring *ring,
                                  __u16 level, __u16 facility,
                                  const void *payload, __u32 payload_len)
{
    struct airy_log_record *rec;
    struct timespec64 ts;

    /* 1. reserve 128B slot（~10ns） */
    rec = airy_log_reserve(ring);
    if (unlikely(!rec))
        return -AIRY_EIPC_RING_FULL;

    /* 2. 填充字段 + memcpy 128B raw binary（~20ns 裸 memcpy） */
    rec->magic       = AIRY_LOG_MAGIC;
    rec->level       = level;
    rec->facility    = facility;
    ktime_get_real_ts64(&ts);
    rec->timestamp_ns = timespec64_to_ns(&ts);
    rec->caller_id   = current->pid;
    rec->payload_len = min(payload_len, 96u);
    memcpy(rec->payload, payload, rec->payload_len);

    /* 3. commit 原子索引 + eventfd（~10ns） */
    airy_log_commit(ring, rec);
    return 0;
}
```

### 3.2 Fastpath 性能拆解

| 步骤 | 操作 | 耗时 |
|------|------|------|
| 1. reserve | `cmpxchg` 原子自增 head | ~10ns |
| 2. memcpy | 128B `memcpy`（raw binary，无格式化） | ~20ns |
| 3. commit | `smp_store_release` + `eventfd_signal` | ~10ns |
| **总计** | | **~40-100ns** |

Fastpath 不进行任何 `sprintf`/`snprintf`/字符串拼接，payload 是原始二进制，由 Logger Daemon 异步解析。

### 3.3 分支预测优化

Fastpath 使用 `likely` / `unlikely` 标注关键分支，确保 99%+ 的正常路径不触发分支预测失败：

- `unlikely(ring->frozen)`：冻结检查，99%+ 不冻结
- `unlikely(!rec)`：reserve 失败检查，99%+ 成功
- `likely(payload_len <= 96)`：payload 长度检查，99%+ 合规

---

## §4 eventfd 通知：Logger Daemon 消费通知

### 4.1 通知机制

生产者 commit 后通过 `eventfd_signal()` 通知 Logger Daemon。`eventfd_signal` 是非阻塞的，仅设置信号计数，不触发上下文切换：

```c
/* kernel/log/airy_log_ring.c —— eventfd 通知 */
static int airy_log_ring_register_eventfd(struct airy_log_ring *ring, int efd)
{
    ring->efd = eventfd_ctx_fdget(efd);
    return 0;
}
```

Logger Daemon 通过 `io_uring` 注册 `eventfd` 读事件，当计数 > 0 时被唤醒，批量消费 Ring Buffer。

### 4.2 批量消费

Logger Daemon 被唤醒后批量消费，避免每条日志触发一次唤醒：

```c
/* services/daemons/logger_daemon/main.c —— 批量消费 */
static void logger_consume(struct airy_log_ring *ring)
{
    struct airy_log_ring_header *hdr = mmap_addr;
    __u64 head, tail;
    do {
        head = smp_load_acquire(&hdr->head);
        tail = hdr->tail;
        while (tail < head) {
            struct airy_log_record *rec = record_at(ring, tail % hdr->capacity);
            logger_format_and_write(rec);   /* sprintf 格式化 + 落盘 */
            tail++;
        }
        smp_store_release(&hdr->tail, tail);
    } while (head != smp_load_acquire(&hdr->head));   /* 防止漏消费 */
}
```

### 4.3 无上下文切换

`eventfd_signal` 仅增加计数，不唤醒进程（进程由 `io_uring` 的 poll 机制异步唤醒）。Fastpath 路径全程无上下文切换，这是 A-ULP 性能优于传统 `printk` 的关键之一。

---

## §5 Logger Daemon 分离：mmap 读取 + sprintf 格式化 + 过滤 + 落盘

### 5.1 分离原则

A-ULP 严格遵循"内核 Fastpath 禁止格式化"原则。所有耗时的操作（格式化、过滤、落盘）全部下沉到用户态 Logger Daemon：

```
内核 Fastpath（~50-100ns）           用户态 Logger Daemon（异步，ms 级）
┌──────────────────────────┐        ┌──────────────────────────┐
│ 1. reserve 128B slot     │        │ 1. mmap 读取 Ring Buffer │
│ 2. memcpy 128B raw       │ ──eventfd──▶ 2. sprintf 格式化     │
│ 3. commit 原子索引       │        │ 3. 过滤（level/facility） │
│ 4. eventfd_signal（非阻塞）│        │ 4. 落盘 + 轮转 + 压缩    │
└──────────────────────────┘        └──────────────────────────┘
```

### 5.2 Logger Daemon 职责

| 职责 | 实现位置 | 说明 |
|------|---------|------|
| mmap 读取 | `services/daemons/logger_daemon/main.c` | 通过 mmap 映射 Ring Buffer，零拷贝读取 |
| sprintf 格式化 | `services/daemons/logger_daemon/formatter.c` | 解析 `airy_log_record`，格式化为人类可读字符串 |
| 过滤 | `services/daemons/logger_daemon/main.c` | 按 level/facility 过滤，受 A-UCS sysctl 控制 |
| 落盘 | `services/daemons/logger_daemon/main.c` | 写入 `/var/log/airymax/`，按大小/时间轮转 |
| 压缩 | `services/daemons/logger_daemon/main.c` | 旧日志 gzip 压缩，节省空间 |

### 5.3 格式化示例

Logger Daemon 的格式化引擎将 128B raw binary 转换为标准日志行：

```
[2026-07-17 12:34:56.789012345] [ERROR] [IPC] [pid=1234] payload: opcode=0x0021 flags=0x0000 trace_id=0x...
```

格式化规则通过 `/etc/airymax/logger_daemon.conf` 配置（受 A-UCS 管理），支持自定义字段顺序与格式。

### 5.4 Logger Daemon 崩溃恢复

Logger Daemon 是用户态进程，可能崩溃。A-ULP 的设计确保 Logger Daemon 崩溃不丢日志：

1. **Ring Buffer 不丢**：Ring Buffer 在内核态，Logger Daemon 崩溃不影响生产者写入
2. **覆盖写策略**：当 Ring Buffer 满，最旧记录被覆盖（可配置为阻塞写，但默认覆盖）
3. **systemd 自动重启**：Logger Daemon 作为 systemd service，崩溃后自动重启
4. **重启后继续消费**：Logger Daemon 重启后从 `tail` 索引继续消费，仅丢失崩溃期间被覆盖的记录

---

## §6 Panic 回退：printk_safe 原生路径

### 6.1 为什么需要 Panic 回退

当系统进入 Panic 状态时，Ring Buffer 可能已损坏（内存损坏导致 `airy_log_ring_header` 元数据不一致），或 Logger Daemon 已死亡（无法消费 Ring Buffer）。此时若仍尝试写入 Ring Buffer，可能导致：

- **递归 Panic**：Ring Buffer 写入触发再次 Panic，无限循环
- **死锁**：`cmpxchg` 在 NMI 上下文不安全
- **数据丢失**：Ring Buffer 损坏后写入的数据无法被消费

### 6.2 回退路径

Panic 时 A-ULP 回退到 Linux 6.6 原生 `printk_safe` 路径（NMI-safe buffer）：

```c
/* kernel/log/airy_log_ring.c —— Panic 回退 */
static void airy_log_panic(const char *fmt, ...)
{
    va_list args;
    va_start(args, fmt);

    /* 1. best-effort：尝试写入 Ring Buffer（失败不阻塞） */
    if (airy_log_ring_try_write(fmt, args) < 0) {
        /* 2. Ring 不可用，回退到 printk_safe NMI-safe 路径 */
        printk_safe_flush_on_panic();
        vprintk(fmt, args);
    }
    va_end(args);

    /* 3. 触发 Panic */
    panic("Airymax fatal fault");
}
```

### 6.3 printk_safe 特性

Linux 6.6 的 `printk_safe` 提供 NMI-safe 的 per-CPU 临时缓冲区：

- **无锁**：per-CPU 缓冲区，无需锁，NMI 上下文安全
- **无依赖**：不依赖调度器、内存分配、RCU
- **延迟刷新**：Panic 时通过 `printk_safe_flush_on_panic()` 刷入主 log buffer
- **大小有限**：每 CPU 8192 字节

### 6.4 Panic 回退的触发条件

| 触发条件 | 处理 |
|---------|------|
| `panic()` 调用 | 回退到 printk_safe |
| NMI 上下文 | 回退到 printk_safe |
| Ring Buffer `frozen=true` | 回退到 printk_safe |
| Ring Buffer magic 校验失败 | 回退到 printk_safe |
| Logger Daemon 心跳超时 | 继续用 Ring Buffer（不回退），但告警 |

---

## §7 性能对比：传统 printk ~5-10μs → Airymax 全路径 ~50-100ns

### 7.1 性能对比表

| 场景 | 传统 printk | Airymax 全路径 | 提升 |
|------|------------|--------------|------|
| 单条日志写入 | ~5-10μs | ~50-100ns | 100-500 倍 |
| 格式化位置 | 内核态（`vsnprintf`） | 用户态（Logger Daemon） | — |
| 锁竞争 | `logbuf_lock`（全局锁） | 无锁（`cmpxchg` 原子索引） | — |
| 上下文切换 | 可能触发控制台刷新 | 无（`eventfd_signal` 非阻塞） | — |
| Panic 安全性 | 依赖 `printk_safe` | 正常用 Ring + Panic 回退 `printk_safe` | — |

### 7.2 性能拆解对比

| 步骤 | 传统 printk | Airymax |
|------|------------|---------|
| 格式化 | `vsnprintf` ~2-5μs | 无（payload raw binary） |
| 锁获取 | `logbuf_lock` ~500ns-1μs | 无（原子索引 ~10ns） |
| 数据拷贝 | `memcpy` 到 logbuf ~100ns | `memcpy` 128B ~20ns |
| 通知 | 控制台刷新（可能阻塞） | `eventfd_signal` ~10ns |
| **总计** | ~5-10μs | ~50-100ns |

### 7.3 吞吐量对比

在 8 核 x86_64 机器上，单核持续写日志的吞吐量：

| 方案 | 单核吞吐 | 8 核总吞吐 |
|------|---------|----------|
| 传统 printk | ~10 万条/秒 | ~30 万条/秒（锁竞争） |
| Airymax Ring Buffer | ~1000 万条/秒 | ~8000 万条/秒（无锁） |

### 7.4 延迟 SLO

A-ULP 的延迟 SLO（受 [170-performance/05-agent-latency-slo.md](../170-performance/05-agent-latency-slo.md) 约束）：

| 指标 | SLO | 实测 |
|------|-----|------|
| Fastpath p99 延迟 | ≤200ns | ~80-120ns |
| Fastpath p999 延迟 | ≤500ns | ~150-200ns |
| Logger Daemon 消费延迟 | ≤10ms | ~1-5ms |

---

## §8 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §5 —— A-ULP 模块总纲
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— 128B 记录格式契约
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §3 —— Panic 回退与 printk_safe
- [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— IPC fastpath（与日志 fastpath 设计同源）
- [170-performance/02-memory-performance.md](../170-performance/02-memory-performance.md) —— 内存性能 SLO
- 综合修正方案 §4.2.2 / §6.2.1 C-07 —— 设计依据

---

## §9 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：零拷贝 Ring Buffer 日志数据流；alloc_pages + mmap 内存方案（不使用 DMA 一致性内存）；无锁 reserve/commit 两阶段写入；Fastpath ~50-100ns；eventfd 通知；Logger Daemon 分离；Panic 回退 printk_safe；性能对比 100-500 倍提升 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | 零拷贝 Ring Buffer 日志数据流 | v1.0 | 2026-07-17
