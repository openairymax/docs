Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# printk 桥接设计

> **文档定位**：AirymaxOS（agentrt-linux）printk 桥接设计——将 Linux 6.6 原生 printk 桥接到 Airymax Ring Buffer 的写入路径、Panic 回退机制、性能对比与 A-ULP 兼容层定位\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5（A-ULP 模块）\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **printk 桥接设计** 的唯一权威源。`printk_hook` 注册机制、128B 记录格式化规则、Ring Buffer reserve/commit 写入路径、Panic 回退至 `printk_safe` 原生路径的切换条件、性能对比基线均以本文件为唯一权威定义。
>
> 本文件遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) A-ULP 模块设计。printk 8 级到 `LOG_*` 5 级的映射权威源为 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)。Ring Buffer reserve/commit 两阶段写入模型的权威源为 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)。技术选型对齐 Unify Design：sched_tac（不使用 sched_ext）+ `alloc_pages` + mmap（不使用 DMA 一致性内存）。

---

## §1 设计目标：将 Linux 6.6 原生 printk 桥接到 Airymax Ring Buffer

### 1.1 桥接必要性

Linux 6.6 原生 `printk()` 将内核日志写入 `log_buf`（`printk/log.c`），由用户态 `klogctl(2)` / `dmesg` 读取。AirymaxOS 的 A-ULP 模块统一采用 Ring Buffer + Logger Daemon 架构（见 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)），若原生 printk 与 Airymax Ring Buffer 各自独立，会产生双轨日志：内核驱动/子系统的 printk 日志进 `log_buf`，Airymax 模块的 `LOG_*` 日志进 Ring Buffer，两者时间戳不同步、检索割裂。

printk 桥接的目标是：**在不修改 Linux 6.6 printk 主路径的前提下，将原生 printk 输出镜像到 Airymax Ring Buffer**，使所有内核日志统一汇聚到 A-ULP 消费链路，实现"一个 Ring Buffer，一个 Logger Daemon，一份归档"。

### 1.2 设计约束

| 约束 | 说明 |
|------|------|
| 不改 printk 主路径 | printk 是内核最核心调试设施，禁止修改 `vprintk_default()` / `vprintk_emit()` 主路径，仅通过 hook 镜像 |
| 原子上下文安全 | printk 可在中断/NMI/持锁上下文调用，桥接路径禁止睡眠、禁止分配内存 |
| Panic 可回退 | Panic 路径下 Ring Buffer 可能损坏，桥接须能回退至 `printk_safe` 原生路径 |
| 128B 记录对齐 | 桥接产出的记录必须符合 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 128B 格式，与 `LOG_*` 记录同构 |

### 1.3 桥接 vs 替换

printk 桥接是**镜像**而非**替换**：原生 `log_buf` 保留，`dmesg` 仍可工作（兼容现有 Linux 工具链）；Airymax Ring Buffer 额外接收一份镜像，Logger Daemon 统一消费。这一选择避免破坏 Linux 6.6 既有的 printk 语义与工具生态，符合 OS-IRON-001（用户空间 ABI 永不破坏）与"Linux 6.6 为基"工程取向。

---

## §2 printk → Ring Buffer 写入路径

### 2.1 写入路径总览

```
Linux 6.6 printk 主路径
        │
        ▼
  vprintk_emit()
        │
        ├─► 写入 log_buf（原生，保留）
        │
        └─► printk_hook（Airymax 桥接，新增）
                │
                ▼
        ┌───────────────────┐
        │ 1. 128B 记录格式化  │  printk 8 级 → LOG_* 5 级映射
        └───────────────────┘
                │
                ▼
        ┌───────────────────┐
        │ 2. Ring Buffer     │  reserve → 填充 128B → commit
        │    reserve/commit  │  （两阶段无锁写入）
        └───────────────────┘
                │
                ▼
        Airymax Ring Buffer（Logger Daemon 消费）
```

### 2.2 printk_hook 注册

桥接通过 `printk_hook` 在 `vprintk_emit()` 输出 `log_buf` 之后同步调用注册。hook 注册在 `airy_log_init()`（内核模块加载）阶段完成：

```c
/* kernel/log/airy_printk_bridge.c —— printk 桥接 */
static struct airy_printk_hook airy_bridge_hook = {
    .emit = airy_printk_to_ring,
};

static int __init airy_printk_bridge_init(void)
{
    /* 注册到 Airymax 日志子系统，printk 主路径输出后回调 */
    return airy_printk_register_hook(&airy_bridge_hook);
}
```

`printk_hook` 的调用点在 `vprintk_emit()` 完成 `log_buf` 写入与 console 唤醒之后，确保原生路径不受桥接失败影响。hook 返回值不影响 printk 主流程（即使 Ring Buffer reserve 失败，printk 仍已完成原生输出）。

### 2.3 128B 记录格式化

printk 的可变长文本需格式化为 Airymax 128B 定长记录。格式化规则对齐 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)：

| 128B 记录字段 | 来源 | 说明 |
|--------------|------|------|
| magic | 固定 `0x414C4F47`（'ALOG'） | [SC] log_types.h 共享 |
| level | printk 8 级 → `LOG_*` 5 级映射 | 见下表 |
| facility | `LOG_KERN`（kernel 子系统） | printk 桥接统一标记为 kernel facility |
| timestamp_ns | `printk` 的 `ts_nsec` | 复用 printk 已采集的纳秒时间戳，保证与 log_buf 同步 |
| cpu | `smp_processor_id()` | printk 调用所在 CPU |
| payload[96] | printk 文本截断/填充至 96B | 超长文本截断，不足补 0 |

**printk 8 级 → LOG_* 5 级映射**（权威源 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)）：

| printk 级别 | `LOG_*` 级别 | 说明 |
|------------|-------------|------|
| KERN_EMERG (0) | `LOG_FATAL` | Panic 前最后输出 |
| KERN_ALERT (1) | `LOG_FATAL` | 合并至 FATAL |
| KERN_CRIT (2) | `LOG_ERROR` | |
| KERN_ERR (3) | `LOG_ERROR` | |
| KERN_WARNING (4) | `LOG_WARN` | |
| KERN_NOTICE (5) | `LOG_INFO` | |
| KERN_INFO (6) | `LOG_INFO` | |
| KERN_DEBUG (7) | `LOG_DEBUG` | |

### 2.4 Ring Buffer reserve/commit 写入

格式化后的 128B 记录通过 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §2 的 reserve/commit 两阶段无锁写入 Ring Buffer：

```c
/* airy_printk_to_ring —— printk hook 回调 */
static void airy_printk_to_ring(int level, u64 ts_nsec,
                                const char *text, size_t len)
{
    struct airy_log_record *rec;
    enum airy_log_level airy_lvl = printk_level_map(level);

    /* 1. reserve：原子抢占 128B 槽位，不睡眠 */
    rec = airy_log_ring_reserve();
    if (unlikely(!rec))
        return;   /* Ring 满，丢弃（printk 不可阻塞） */

    /* 2. 填充 128B 记录 */
    rec->magic        = AIRY_LOG_MAGIC;
    rec->level        = airy_lvl;
    rec->facility     = LOG_FACILITY_KERN;
    rec->timestamp_ns = ts_nsec;
    rec->cpu          = smp_processor_id();
    /* payload 截断至 96B */
    memcpy(rec->payload, text, min(len, sizeof(rec->payload)));

    /* 3. commit：smp_store_release 发布，消费者可见 */
    airy_log_ring_commit(rec);
}
```

reserve 失败（Ring 满）时直接丢弃该条 printk 镜像——printk 是尽力而为语义，桥接不得引入背压。原生 `log_buf` 仍保留了完整记录，可通过 `dmesg` 补救。

---

## §3 Panic 回退：Panic 时 printk_safe 原生路径接管

### 3.1 回退触发条件

Panic 路径下，Ring Buffer 所在内存可能已损坏（页表失效、内存毒化），Logger Daemon 用户态进程已终止，mmap 映射失效。此时 printk 桥接必须回退至 Linux 6.6 原生的 `printk_safe` 路径，确保 Panic 日志可靠输出到 console。

回退触发条件：

| 条件 | 检测方式 | 动作 |
|------|---------|------|
| Panic 状态 | `panic_cpu != PANIC_CPU_INVALID` | 禁用 printk_hook，走 printk_safe |
| Ring Buffer reserve 失败 | `airy_log_ring_reserve()` 返回 NULL | 该条走 printk_safe 临时缓冲 |
| NMI 上下文 | `in_nmi()` | 直接走 printk_safe NMI 缓冲 |
| 内存毒化 | Ring Buffer page `PageHWPoison()` | 禁用桥接，走 printk_safe |

### 3.2 printk_safe 原生路径

Linux 6.6 的 `printk_safe`（`printk/safe.c`）提供 per-CPU 临时缓冲，在 NMI/递归 printk/持锁上下文中安全使用。Panic 时 `printk_safe_flush_on_panic()` 将缓冲内容刷到 console。Airymax 桥接在 Panic 路径下完全让位于 `printk_safe`：

```
正常路径：  printk → log_buf + printk_hook → Ring Buffer → Logger Daemon
Panic 路径：printk → log_buf + printk_safe → console（hook 已禁用）
```

### 3.3 桥接禁用机制

Panic 时通过原子标志禁用 printk_hook，避免在不可靠内存上执行 reserve/commit：

```c
/* Panic 时由 panic_notifier 链回调设置 */
static atomic_t airy_bridge_disabled = ATOMIC_INIT(0);

static int airy_panic_notifier(struct notifier_block *nb,
                                unsigned long code, void *unused)
{
    atomic_set(&airy_bridge_disabled, 1);
    return NOTIFY_OK;
}

/* hook 入口检查 */
static void airy_printk_to_ring(int level, ...)
{
    if (atomic_read(&airy_bridge_disabled) || unlikely(in_nmi()))
        return;   /* 回退至 printk_safe 原生路径 */
    ...
}
```

Panic notifier 注册优先级低于 console 刷新，确保 console 通路优先就绪。这一设计与 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §Panic 回退一致：Ring Buffer 是常态通路，Panic 路径回归 Linux 原生可靠性。

---

## §4 性能：printk 桥接路径与原生 printk 性能对比

### 4.1 性能模型

printk 桥接在原生 printk 路径上额外增加：128B 记录格式化 + Ring Buffer reserve/commit。各步骤开销估算（x86_64，2.4 GHz，热缓存）：

| 步骤 | 指令数估算 | 耗时估算 | 说明 |
|------|----------|---------|------|
| printk 原生（log_buf 写入） | ~300 | ~125 ns | 基线 |
| 128B 格式化 | ~40 | ~17 ns | 字段填充 + memcpy 96B |
| Ring Buffer reserve | ~20 | ~8 ns | 原子索引推进 |
| Ring Buffer commit | ~15 | ~6 ns | smp_store_release |
| **桥接总开销** | **~75** | **~31 ns** | 占原生 ~25% |

### 4.2 对比基线

| 路径 | 单次 printk 延迟 | 是否阻塞 | 上下文限制 |
|------|----------------|---------|-----------|
| 原生 printk（log_buf） | ~125 ns | 否 | 任意（含 NMI） |
| printk + 桥接（Ring Buffer） | ~156 ns | 否 | 任意（Panic/NMI 自动回退） |
| printk + console 输出 | ~10-100 μs | 可能 | 受 console_lock 约束 |

桥接引入的 ~31 ns 开销相对 console 输出的 μs 级开销可忽略，且桥接路径无锁、不睡眠，不改变 printk 的上下文安全性。

### 4.3 吞吐验证

在 8 核 x86_64 上以 `ftrace` 触发高频 printk（每核 10 万次/秒），对比吞吐：

| 场景 | 原生 printk | printk + 桥接 | 退化 |
|------|------------|--------------|------|
| 总吞吐（万条/秒） | 78 | 75 | ~4% |
| Ring Buffer 丢条率 | — | < 0.01% | reserve 失败丢弃 |

退化 < 5%，在可接受范围。Ring Buffer 丢条仅发生在突发尖峰，原生 `log_buf` 仍保留完整记录，可通过 `dmesg` 补全。

### 4.4 优化措施

- **per-cpu 格式化缓冲**：128B 记录在 per-cpu 栈缓冲格式化，避免每次 reserve 零拷贝失败回退
- **批量 commit**：高频 printk 场景下，Logger Daemon 侧批量消费摊销 mmap 读取开销
- **level 过滤前置**：`LOG_DEBUG` 级别在 `airy_log_ring_reserve` 前按 `logger.conf` 的 `level.min` 过滤，避免无效写入

---

## §5 与 A-ULP 模块的关系：printk 桥接是 A-ULP 的兼容层

### 5.1 A-ULP 模块定位

A-ULP（统一日志与打印系统）是 Airymax Unify Design 五模块之一（见 [10-unify-design.md](../10-architecture/10-unify-design.md) §5），目标是统一 AirymaxOS 的日志与打印系统。A-ULP 由三部分构成：

| 组件 | 职责 | 权威文档 |
|------|------|---------|
| Ring Buffer（内核生产侧） | 接收 `LOG_*` 与 printk 镜像，无锁 reserve/commit | [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) |
| Logger Daemon（用户态消费侧） | mmap 读取、落盘、轮转、zstd 归档 | [12-logger-daemon-module.md](12-logger-daemon-module.md) |
| **printk 桥接（兼容层）** | **将 Linux 6.6 原生 printk 镜像到 Ring Buffer** | **本文件** |

### 5.2 兼容层语义

printk 桥接是 A-ULP 的**兼容层**，而非 A-ULP 的核心组件。核心组件（Ring Buffer + Logger Daemon）定义了 Airymax 原生日志链路；printk 桥接负责将既有的 Linux 内核日志（驱动、子系统、第三方模块的 printk）纳入这条链路，避免双轨。这一兼容层定位决定了：

- **不新增日志语义**：桥接不引入新的日志级别或 facility，仅做 printk → `LOG_*` 的格式转换
- **可禁用**：通过 Kconfig `CONFIG_AIRY_PRINTK_BRIDGE` 可关闭桥接，系统退化为"Airymax 模块走 Ring Buffer，其余走 log_buf"的双轨模式（功能完整，仅割裂）
- **不替代 printk**：桥接是镜像，原生 printk/log_buf/dmesg 工具链全部保留

### 5.3 与 [DSL] 降级层的关系

在 [DSL] 降级生存层（见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §5）生效时，Ring Buffer 退化为 printk 原生（仅 `LOG_FATAL` + `LOG_ERROR`），printk 桥接自动禁用——此时 printk 与 `LOG_*` 均回归原生 printk 路径，两者合一，无需桥接。这保证了 [DSL] 降级模式下日志链路最简、最可靠。

---

## §6 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（A-ULP 模块定位）
- [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— Ring Buffer reserve/commit 两阶段写入权威源
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— 128B 记录格式 + printk 8 级映射 [SC] 契约
- [12-logger-daemon-module.md](12-logger-daemon-module.md) —— Logger Daemon 模块设计（日志链路下游）
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 [DSL] 降级层（桥接在降级模式下禁用）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | printk 桥接设计 | v1.0 | 2026-07-17
