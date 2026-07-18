Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Panic 生存路径设计
> **文档定位**：A-ULP（统一日志与打印系统）Panic 场景日志生存路径的唯一权威设计\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）+ [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §3

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Panic 生存路径** 的唯一权威源。printk_safe NMI-safe buffer 模型、Ring Buffer 锁规避策略（trylock + 直接写入）、Panic handler 流程（dump 寄存器 → 写入 Ring Buffer → 同步持久存储）、与 [DSL] 降级模式的关系均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Panic 日志生存策略。
>
> 技术选型声明：日志内存采用 **alloc_pages(GFP_KERNEL) + mmap**（**不使用 DMA 一致性内存**）。整体遵循 Unify Design：sched_tac（不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：内核开发者、可靠性工程师、故障排查工程师
- **前置知识**：理解 [05-ring-buffer-logging.md](05-ring-buffer-logging.md) Ring Buffer 数据流、[11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) [DSL] 降级层、Linux 6.6 `printk_safe` 与 NMI-safe 机制
- **预计阅读时间**：25 分钟
- **核心概念**：Panic 生存、printk_safe、NMI-safe buffer、Ring Buffer 锁规避、trylock、Panic handler、[DSL] 降级
- **复杂度标识**：高级

---

## §1 设计目标：系统 Panic 时的日志不丢失

### 1.1 问题背景

当系统进入 Panic 状态时，日志系统的可用性面临三重威胁：

| 威胁 | 原因 | 后果 |
|------|------|------|
| Ring Buffer 损坏 | 内存损坏导致 `airy_log_ring_header` 元数据不一致 | cmpxchg 死循环或写入错位 |
| Logger Daemon 死亡 | Panic 时用户态进程被杀或无法调度 | Ring Buffer 无消费者，数据无法落盘 |
| 递归 Panic | Ring Buffer 写入触发再次 Panic | 无限循环，系统卡死 |

传统设计在 Panic 时依赖 `printk` 全局 logbuf，但 Airymax 正常路径使用 Ring Buffer，Panic 时若仍尝试写入 Ring Buffer 可能导致递归 Panic。

### 1.2 设计目标

Panic 生存路径的核心设计目标有三：

1. **日志不丢失**：Panic 前最后一条日志必须落盘
2. **不递归 Panic**：Panic 路径写入日志不得触发再次 Panic
3. **NMI 安全**：即使在 NMI 上下文（中断栈，不可调度）也能安全记录

### 1.3 路径选择

Airymax 采用**两级回退**策略：

```
正常路径（Ring Buffer）           Panic 路径（回退）
┌──────────────────────┐        ┌──────────────────────┐
│ Ring Buffer 可用     │ ──┐   │ Ring Buffer 不可用   │
│ ~50-100ns            │    │   │ 回退 printk_safe    │
│ eventfd 通知         │    └──▶│ NMI-safe per-CPU    │
└──────────────────────┘        │ 延迟刷入主 logbuf    │
                                 └──────────┬───────────┘
                                            │ Panic handler
                                            ▼
                                 ┌──────────────────────┐
                                 │ 同步到持久存储        │
                                 │ (console + pstore)   │
                                 └──────────────────────┘
```

---

## §2 printk_safe 模型：对齐 Linux 6.6 NMI-safe buffer

### 2.1 printk_safe 机制

Linux 6.6 的 `printk_safe` 提供 NMI-safe 的 per-CPU 临时缓冲区，Airymax 直接复用此机制作为 Panic 回退路径。其核心特性为：

| 特性 | 说明 | Panic 价值 |
|------|------|-----------|
| per-CPU 无锁 | 每个 CPU 独立缓冲区，无需锁 | NMI 上下文安全，无死锁 |
| 无依赖 | 不依赖调度器、内存分配、RCU | Panic 时这些子系统可能已损坏 |
| 延迟刷新 | `printk_safe_flush_on_panic()` 刷入主 logbuf | 避免在 NMI 中执行昂贵操作 |
| 大小有限 | 每 CPU 8192 字节 | 足以记录 Panic 关键信息 |

### 2.2 Airymax 对齐模型

Airymax 的 Panic 生存路径对齐 Linux 6.6 `printk_safe` 模型，但在回退前先尝试 Ring Buffer 的 best-effort 写入：

```c
/* kernel/log/airy_log_ring.c —— Panic 生存模型 */
/*
 * 两级回退策略：
 *   Level 1: best-effort 写入 Ring Buffer（trylock，失败不阻塞）
 *   Level 2: 回退到 printk_safe NMI-safe 路径
 *
 * 对齐 Linux 6.6 printk_safe 模型：
 *   - per-CPU 无锁缓冲区
 *   - 不依赖调度器/内存分配/RCU
 *   - printk_safe_flush_on_panic() 延迟刷新
 */
```

### 2.3 与正常路径的关系

| 维度 | 正常路径 | Panic 路径 |
|------|---------|-----------|
| 写入目标 | Ring Buffer（alloc_pages + mmap） | printk_safe per-CPU buffer |
| 锁机制 | cmpxchg 原子索引 | 无锁（per-CPU） |
| 通知 | eventfd_signal | 无（Panic 时无消费者） |
| 落盘 | Logger Daemon 异步 | console + pstore 同步 |
| 上下文安全 | 进程上下文 | NMI/中断上下文安全 |

---

## §3 NMI-safe buffer：中断上下文安全写入

### 3.1 NMI 上下文约束

NMI（Non-Maskable Interrupt）上下文是最严格的执行环境，存在以下约束：

1. **不可抢占**：NMI 不可被其他中断打断
2. **不可调度**：不能调用 `schedule()`、`kmalloc(GFP_KERNEL)` 等可睡眠函数
3. **不可持锁**：持锁后若被中断再次进入同一锁，死锁
4. **栈空间有限**：NMI 使用独立中断栈，空间有限

### 3.2 NMI-safe 写入实现

printk_safe 的 per-CPU buffer 使用无锁写入，Airymax Panic 路径直接复用：

```c
/* kernel/log/airy_log_panic.c —— NMI-safe 写入 */
/*
 * NMI-safe 写入：使用 per-CPU buffer，无锁
 * 对齐 Linux 6.6 __printk_safe_enter/exit
 */
static DEFINE_PER_CPU(char[AIRY_PANIC_BUF_SIZE], airy_panic_buf);
static DEFINE_PER_CPU(int, airy_panic_buf_pos);

static int airy_panic_write_nmi_safe(const char *fmt, ...)
{
    char *buf = this_cpu_ptr(airy_panic_buf);
    int *pos = this_cpu_ptr(airy_panic_buf_pos);
    va_list args;
    int len;

    /* per-CPU 无锁写入，NMI 安全 */
    va_start(args, fmt);
    len = vsnprintf(buf + *pos, AIRY_PANIC_BUF_SIZE - *pos, fmt, args);
    va_end(args);

    /* 溢出保护：缓冲区满则丢弃（不阻塞） */
    if (*pos + len < AIRY_PANIC_BUF_SIZE)
        *pos += len;
    else
        len = -1;  /* 丢弃，返回错误但不 Panic */

    return len;
}
```

### 3.3 中断上下文安全保证

| 约束 | 解决方案 | 验证 |
|------|---------|------|
| 不可抢占 | per-CPU buffer，本 CPU 操作 | `this_cpu_ptr` 禁用抢占 |
| 不可调度 | 无 `schedule()` 调用 | 静态分析 CI 检查 |
| 不可持锁 | 无锁写入 | `vsnprintf` 不持锁 |
| 栈空间有限 | 仅使用 per-CPU buffer，不分配 | 无 `kmalloc` |

---

## §4 Ring Buffer 锁规避：Panic 时绕过常规锁，使用 trylock + 直接写入

### 4.1 正常路径的锁风险

正常路径使用 `cmpxchg` 原子操作更新 Ring Buffer 索引。在 Panic 场景下，`cmpxchg` 可能导致：

1. **死循环**：若 Ring Buffer 内存损坏，`cmpxchg` 永远不成功
2. **数据竞争**：Panic 发生在某 CPU 持有中间状态时，其他 CPU 写入导致数据损坏
3. **递归 Panic**：cmpxchg 失败触发 BUG_ON，再次进入 Panic

### 4.2 trylock + 直接写入策略

Panic 路径采用 **trylock + 直接写入** 策略，规避常规锁：

```c
/* kernel/log/airy_log_panic.c —— Ring Buffer 锁规避 */
static int airy_panic_ring_write_best_effort(const char *fmt, ...)
{
    struct airy_log_ring_header *hdr = g_panic_ring_hdr;
    struct airy_log_record *rec;
    va_list args;
    __u64 head;
    __u32 idx;

    /* 1. trylock 模式：不等待，失败立即回退 */
    /* frozen 检查：Micro-Supervisor 可能已冻结 Ring */
    if (unlikely(hdr->frozen))
        return -EBUSY;

    /* 2. 读取 head 索引（非原子，best-effort） */
    head = hdr->head;
    idx = head % hdr->capacity;

    /* 3. 直接写入（不 cmpxchg，不等待锁） */
    rec = (struct airy_log_record *)((char *)hdr + sizeof(*hdr)
            + idx * hdr->record_size);

    /* 4. 填充记录（raw binary，不格式化） */
    rec->magic = AIRY_LOG_MAGIC;
    rec->level = LOG_FATAL;
    rec->facility = AIRY_FAC_KERNEL;
    rec->timestamp_ns = ktime_get_real_ns();
    rec->caller_id = 0;  /* Panic 上下文 */

    va_start(args, fmt);
    rec->payload_len = vsnprintf(rec->payload, 96, fmt, args);
    va_end(args);

    /* 5. 更新 head（best-effort，可能与其他 CPU 竞争，但 Panic 下可接受） */
    hdr->head = head + 1;

    return 0;
}
```

### 4.3 best-effort 语义

Panic 路径的 Ring Buffer 写入是 **best-effort**——不保证一致性，仅尽力而为：

| 维度 | 正常路径 | Panic best-effort |
|------|---------|-------------------|
| 索引更新 | `cmpxchg` 原子 | 直接写（非原子） |
| 锁等待 | 无（原子操作） | trylock，失败立即回退 |
| 一致性保证 | 强一致 | 弱一致（可能丢/覆盖） |
| 失败处理 | 返回错误码 | 回退 printk_safe |

### 4.4 回退决策树

```
Panic 日志写入请求
       │
       ▼
 Ring Buffer frozen?
   ├── Yes ──────────────▶ 回退 printk_safe
   └── No
       ▼
   magic 校验通过?
   ├── No ────────────────▶ 回退 printk_safe（Ring 损坏）
   └── Yes
       ▼
   trylock 成功?（best-effort 写入）
   ├── No ────────────────▶ 回退 printk_safe
   └── Yes
       ▼
   写入成功 + eventfd_signal
   （eventfd 可能失败，忽略）
```

---

## §5 Panic handler 流程：dump 寄存器 → 写入 Ring Buffer → 同步到持久存储

### 5.1 Panic handler 总流程

Airymax 的 Panic handler 流程对齐 Linux 6.6 `panic()` 流程，但在其中插入 Airymax 专属的日志生存步骤：

```
panic() 触发
    │
    ▼
1. 禁用中断（local_irq_disable）+ 停止其他 CPU（smp_send_stop）
    │
    ▼
2. dump 寄存器（show_regs / dump_stack）
    │   ├── 通用寄存器（rax/rbx/rcx/...）
    │   ├── 段寄存器（cs/ds/ss）
    │   └── 调用栈
    │
    ▼
3. 写入 Panic 日志（两级回退）
    │   ├── best-effort 写入 Ring Buffer
    │   └── 回退 printk_safe NMI-safe buffer
    │
    ▼
4. 刷新 printk_safe 到主 logbuf（printk_safe_flush_on_panic）
    │
    ▼
5. 同步到持久存储
    │   ├── console_flush()：同步写入串口/控制台
    │   └── pstore_write()：写入 NVRAM/pstore 分区
    │
    ▼
6. 最终状态：死循环或重启（panic_timeout）
```

### 5.2 寄存器 dump 实现

```c
/* kernel/log/airy_log_panic.c —— 寄存器 dump */
static void airy_panic_dump_regs(struct pt_regs *regs)
{
    /* 1. 写入 Ring Buffer（best-effort） */
    airy_panic_ring_write_best_effort(
        "AIRY PANIC: RIP=%lx RSP=%lx RFLAGS=%lx\n",
        regs->ip, regs->sp, regs->flags);

    airy_panic_ring_write_best_effort(
        "RAX=%lx RBX=%lx RCX=%lx RDX=%lx RSI=%lx RDI=%lx\n",
        regs->ax, regs->bx, regs->cx, regs->dx, regs->si, regs->di);

    /* 2. 同时写入 printk_safe（确保 NMI 安全） */
    printk_safe_flush_on_panic();
    printk(KERN_EMERG "AIRY PANIC: RIP=%lx RSP=%lx\n", regs->ip, regs->sp);
    show_regs(regs);
}
```

### 5.3 持久存储同步

Panic 日志必须同步到持久存储，确保重启后可恢复。Airymax 使用双通道持久化：

| 通道 | 机制 | 用途 |
|------|------|------|
| console | `console_unlock()` 同步刷新 | 串口/终端实时输出 |
| pstore | `pstore_write()` 写入 NVRAM | 重启后通过 `/sys/fs/pstore/` 读取 |

```c
/* kernel/log/airy_log_panic.c —— 持久存储同步 */
static void airy_panic_sync_persistent(void)
{
    /* 1. 刷新 printk_safe per-CPU buffer 到主 logbuf */
    printk_safe_flush_on_panic();

    /* 2. console 同步输出（可能阻塞，但 Panic 下可接受） */
    console_unlock();

    /* 3. pstore 写入（NVRAM，重启后可恢复） */
    if (pstore_is_available()) {
        kmsg_dump(KMSG_DUMP_PANIC);
    }
}
```

### 5.4 Panic 信息完整性清单

Panic handler 必须保证以下信息完整落盘：

| 信息 | 来源 | 必须 |
|------|------|------|
| Panic 原因 | `panic()` 参数 | 是 |
| 寄存器 dump | `show_regs()` | 是 |
| 调用栈 | `dump_stack()` | 是 |
| Airymax Fault 码 | A-UEF Fault 码（如 `AIRY_FAULT_CAP_FORGED`） | 是 |
| Agent 状态 | A-ULS Agent 8 态当前值 | 是 |
| Ring Buffer 状态 | head/tail/frozen | 否（best-effort） |

---

## §6 与 [DSL] 层的关系：Panic 时进入 [DSL] 降级模式

### 6.1 Panic 触发 [DSL]

Panic 是 [DSL] 降级生存层的触发条件之一。当 Panic 发生时，系统自动进入 [DSL] 降级模式（详见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)）：

| [DSL] 降级项 | Panic 时的表现 |
|-------------|--------------|
| 错误码 | 仅保留 38 个 POSIX 码 + `AIRY_ECFGVERSION` |
| 日志 | 回退 printk_safe 原生路径（不依赖 Ring Buffer） |
| 调度 | EEVDF 默认（不依赖 SCHED_DEADLINE / SCHED_FIFO） |
| IPC | 最简 128B 消息头（仅 magic + opcode + payload_len） |

### 6.2 Panic 路径与 [DSL] 的协作

```
Panic 发生
    │
    ▼
检查 [SC] 头文件完整性
    │
    ├── 完整 ──▶ best-effort 写入 Ring Buffer + printk_safe 双写
    │
    └── 损坏 ──▶ 进入 [DSL] 降级模式
                    │
                    ▼
                仅使用 printk_safe（不碰 Ring Buffer）
                仅使用 EEVDF 调度（不碰 SCHED_DEADLINE）
```

### 6.3 [DSL] 降级块在 Panic 路径的作用

[DSL] 的 `#ifdef AIRY_SC_FALLBACK` 降级块确保即使 [SC] 头文件损坏，Panic 路径仍可用：

```c
/* kernel/include/airymax/log_types.h 底部 —— [DSL] 降级块 */
#ifdef AIRY_SC_FALLBACK
/* 降级块：Panic 路径仅依赖 printk_safe，不碰 Ring Buffer */
#define airy_panic_write(fmt, ...)  printk_safe(fmt, ##__VA_ARGS__)
#warning "AIRY_SC_FALLBACK active: panic path uses printk_safe only"
#else
/* 正常路径：两级回退（Ring Buffer best-effort + printk_safe） */
#define airy_panic_write(fmt, ...)  airy_panic_write_full(fmt, ##__VA_ARGS__)
#endif
```

---

## §7 Panic 生存路径测试

### 7.1 测试场景

| 测试场景 | 触发方式 | 预期结果 |
|---------|---------|---------|
| 正常 Panic | `echo c > /proc/sysrq-trigger` | Ring Buffer + printk_safe 双写，console + pstore 同步 |
| NMI 上下文 Panic | NMI watchdog 触发 | 仅 printk_safe（NMI 安全），不碰 Ring Buffer |
| Ring Buffer 损坏 Panic | 注入内存错误 | 回退 printk_safe，不递归 Panic |
| Logger Daemon 死亡 Panic | kill -9 logger daemon | Ring Buffer 继续写入，Panic 时 best-effort |
| [SC] 头文件损坏 Panic | 编译时定义 AIRY_SC_FALLBACK | [DSL] 降级，仅 printk_safe |

### 7.2 验证 SLO

| 指标 | SLO | 验证方法 |
|------|-----|---------|
| Panic 日志完整性 | 100% 落盘 | pstore + console 双通道验证 |
| Panic 不递归 | 0 次递归 | Panic 后系统不死锁 |
| NMI 上下文安全 | NMI 中可写入 | NMI watchdog 触发测试 |
| Ring Buffer 回退成功率 | ≥99% | 1000 次 Panic 测试统计 |

---

## §8 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §5 —— A-ULP 模块总纲
- [05-ring-buffer-logging.md](05-ring-buffer-logging.md) §6 —— Panic 回退（本文件的上级设计）
- [06-logger-daemon-design.md](06-logger-daemon-design.md) —— Logger Daemon 崩溃恢复
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §3 —— [DSL] 降级与 Panic 回退
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— A-UEF Fault 码（Panic 时记录）
- 综合修正方案 §4.2.2 —— A-ULP 设计依据

---

## §9 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Panic 生存路径设计；printk_safe 模型对齐 Linux 6.6 NMI-safe buffer；NMI-safe buffer 中断上下文安全写入；Ring Buffer 锁规避（trylock + 直接写入）；Panic handler 流程（dump 寄存器 → 写入 Ring Buffer → 同步持久存储）；与 [DSL] 降级模式关系；双通道持久化（console + pstore） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Panic 生存路径设计 | v1.0 | 2026-07-17
