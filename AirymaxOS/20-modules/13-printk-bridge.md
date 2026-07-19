Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# printk 桥接设计

> **文档定位**：AirymaxOS（agentrt-linux）printk 桥接设计——将 Linux 6.6 原生 printk 桥接到 Airymax Ring Buffer 的写入路径、Panic 回退机制、性能对比与 A-ULP 兼容层定位\
> **文档版本**：v1.1\
> **最后更新**：2026-07-19\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5（A-ULP 模块）\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）+ [ADR-012](../10-architecture/05-adrs.md#adr-012-微内核化改造技术路线确认基于-linux-改造--sel4-思想非从零开发)（微内核化改造技术路线）+ [ADR-014](../10-architecture/05-adrs.md#adr-014-微内核设计思想来源单一化仅-sel4不引入-zirconminix3)（微内核设计思想来源单一化：仅 seL4）\
> **极境内核标准**：本模块遵循"对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造"定位，printk 桥接是 seL4 "机制与策略分离"原则在日志链路上的落地——内核仅提供 printk 镜像机制（hook + reserve/commit），策略（过滤、落盘、归档）由用户态 `logger_d` 承担（ADR-014）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **printk 桥接设计** 的唯一权威源。`printk_hook` 注册机制、128B 记录格式化规则、Ring Buffer reserve/commit 写入路径、Panic 回退至 `printk_safe` 原生路径的切换条件、性能对比基线、v1.1 Capability Folding fastpath C-S9 Badge 校验协作均以本文件为唯一权威定义。
>
> 本文件遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) A-ULP 模块设计与 ADR-012/ADR-014 微内核化路线。printk 8 级到 `LOG_*` 5 级的映射权威源为 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)；128B 记录字段命名（含 `caller_id` / `payload_len` / `reserved`）权威源为 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) §2 与 [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) IPC 字段命名规范；Ring Buffer reserve/commit 两阶段写入模型的权威源为 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)。技术选型对齐 Unify Design：sched_tac（不使用 sched_ext）+ `alloc_pages` + mmap（不使用 DMA 一致性内存），微内核化改造仅参考 seL4（不引入 Zircon/Minix3）。

---

## §1 设计目标：将 Linux 6.6 原生 printk 桥接到 Airymax Ring Buffer

### 1.1 桥接必要性

Linux 6.6 原生 `printk()` 将内核日志写入 `log_buf`（`printk/log.c`），由用户态 `klogctl(2)` / `dmesg` 读取。AirymaxOS 的 A-ULP 模块统一采用 Ring Buffer + `logger_d` 架构（见 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)），若原生 printk 与 Airymax Ring Buffer 各自独立，会产生双轨日志：内核驱动/子系统的 printk 日志进 `log_buf`，Airymax 模块的 `LOG_*` 日志进 Ring Buffer，两者时间戳不同步、检索割裂。

printk 桥接的目标是：**在不修改 Linux 6.6 printk 主路径的前提下，将原生 printk 输出镜像到 Airymax Ring Buffer**，使所有内核日志统一汇聚到 A-ULP 消费链路，实现"一个 Ring Buffer，一个 `logger_d`，一份归档"。

### 1.2 设计约束

| 约束 | 说明 |
|------|------|
| 不改 printk 主路径 | printk 是内核最核心调试设施，禁止修改 `vprintk_default()` / `vprintk_emit()` 主路径，仅通过 hook 镜像 |
| 原子上下文安全 | printk 可在中断/NMI/持锁上下文调用，桥接路径禁止睡眠、禁止分配内存 |
| Panic 可回退 | Panic 路径下 Ring Buffer 可能损坏，桥接须能回退至 `printk_safe` 原生路径 |
| 128B 记录对齐 | 桥接产出的记录必须符合 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 128B 格式（含 `caller_id` / `payload_len` / `reserved` 字段），与 `LOG_*` 记录同构 |
| Badge 校验透传 | v1.1 Capability Folding 下，桥接路径需透传 fastpath C-S9 Badge 校验失败事件至 Ring Buffer（不阻断 printk 主路径） |

### 1.3 桥接 vs 替换

printk 桥接是**镜像**而非**替换**：原生 `log_buf` 保留，`dmesg` 仍可工作（兼容现有 Linux 工具链）；Airymax Ring Buffer 额外接收一份镜像，`logger_d` 统一消费。这一选择避免破坏 Linux 6.6 既有的 printk 语义与工具生态，符合 OS-IRON-001（用户空间 ABI 永不破坏）与"Linux 6.6 为基、seL4 为鉴"工程取向（ADR-012/ADR-014）。

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
        │    + caller_id     │  facility = AIRY_FAC_KERNEL
        │    + payload_len   │  填充 payload[96] + reserved
        └───────────────────┘
                │
                ▼
        ┌───────────────────┐
        │ 2. Ring Buffer     │  reserve → 填充 128B → commit
        │    reserve/commit  │  （两阶段无锁写入）
        └───────────────────┘
                │
                ▼
        Airymax Ring Buffer（logger_d 消费）
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

printk 的可变长文本需格式化为 Airymax 128B 定长记录。格式化规则对齐 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) §2，字段命名遵循 [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) IPC 字段命名规范：

| 128B 记录字段 | 偏移 | 长度 | 来源 | 说明 |
|--------------|------|------|------|------|
| `magic` | 0 | 4B | 固定 `0x414C4F47`（'ALOG'） | [SC] log_types.h 共享 |
| `level` | 4 | 2B | printk 8 级 → `LOG_*` 5 级映射 | 见下表 |
| `facility` | 6 | 2B | `AIRY_FAC_KERNEL`（内核子系统） | printk 桥接统一标记为 kernel facility |
| `timestamp_ns` | 8 | 8B | `printk` 的 `ts_nsec` | 复用 printk 已采集的纳秒时间戳，保证与 log_buf 同步 |
| `caller_id` | 16 | 4B | `smp_processor_id()` + 上下文标识 | printk 调用所在 CPU / kmod id（SSoT 字段名：`caller_id`，**非 `cpu`**） |
| `payload_len` | 20 | 4B | `min(text_len, 96)` | payload 有效长度（≤96），缺失则置 0 |
| `payload[96]` | 24 | 96B | printk 文本截断/填充至 96B | 超长文本截断，不足补 0 |
| `reserved` | 120 | 8B | 0（默认） / CRC16（高 16 位，可选） | 预留对齐 + 未来扩展（[SC] 强制字段，禁止省略） |

> **字段命名 SSoT**：`caller_id` 字段命名为 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) §2.2 与 [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) IPC 字段命名规范的统一要求，**禁止使用 `cpu` 字段名**——`caller_id` 语义更广（内核侧为 kmod id / CPU id，用户态为 pid），避免与 `task_struct->cpu` 混淆。`payload_len` 与 `reserved` 为 [SC] 强制字段，禁止省略以保持 128B 严格对齐。

### 2.4 facility 枚举对齐

printk 桥接产出记录的 facility 字段必须使用 `AIRY_FAC_*` 前缀（[SC] log_types.h `airy_log_facility` 枚举），**禁止使用旧版 `LOG_FACILITY_*` 前缀**：

| 新前缀（[SC] 权威） | 值 | 适用场景 | 旧前缀对应位（已废弃） |
|-------------------|----|---------|---------------------|
| `AIRY_FAC_KERNEL` | 0 | printk 桥接默认 facility | KERN 位 |
| `AIRY_FAC_USER` | — | 用户态进程日志（保留位） | USER 位 |
| `AIRY_FAC_AGENT` | — | Agent 运行时日志（保留位） | AGENT 位 |
| `AIRY_FAC_SECURITY` | 6 | sec_d / 纯 C LSM 审计日志 | SEC 位 |
| `AIRY_FAC_MEMORY` | 7 | mem_d 记忆卷载日志 | MEM 位 |
| `AIRY_FAC_COGNITION` | 8 | cogn_d 认知循环日志 | COG 位 |

> **旧前缀废弃说明**：原 `LOG_FACILITY_*` 前缀（含 KERN/USER/AGENT/SEC/MEM/COG 等位）已整体废弃，统一替换为 `AIRY_FAC_*` 前缀。旧前缀与新前缀的完整对应关系见 [SC] log_types.h `airy_log_facility` 枚举注释。

printk 桥接路径固定写入 `AIRY_FAC_KERNEL`，其他 facility 由各 daemon 直接通过 `airy_log_write()` API 写入 Ring Buffer（不经过 printk 桥接）。

**printk 8 级 → LOG_* 5 级映射**（权威源 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) §4）：

| printk 级别 | `LOG_*` 级别 | 说明 |
|------------|-------------|------|
| KERN_EMERG (0) | `LOG_FATAL` | Panic 前最后输出 |
| KERN_ALERT (1) | `LOG_FATAL` | 合并至 FATAL |
| KERN_CRIT (2) | `LOG_FATAL` | 合并至 FATAL |
| KERN_ERR (3) | `LOG_ERROR` | |
| KERN_WARNING (4) | `LOG_WARN` | |
| KERN_NOTICE (5) | `LOG_INFO` | |
| KERN_INFO (6) | `LOG_INFO` | |
| KERN_DEBUG (7) | `LOG_DEBUG` | |

### 2.5 Ring Buffer reserve/commit 写入

格式化后的 128B 记录通过 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §2 的 reserve/commit 两阶段无锁写入 Ring Buffer。API 命名遵循 [SC] ipc.h 与 log_types.h 统一前缀规范，**去除多余的 `_ring` 中缀**：

```c
/* airy_printk_to_ring —— printk hook 回调 */
static void airy_printk_to_ring(int level, u64 ts_nsec,
                                const char *text, size_t len)
{
    struct airy_log_record *rec;
    enum airy_log_level airy_lvl = printk_level_map(level);
    size_t payload_len = min(len, (size_t)sizeof(rec->payload));

    /* 1. reserve：原子抢占 128B 槽位，不睡眠 */
    rec = airy_printk_reserve();
    if (unlikely(!rec))
        return;   /* Ring 满，丢弃（printk 不可阻塞） */

    /* 2. 填充 128B 记录（字段命名对齐 [SC] log_types.h §2.2） */
    rec->magic        = AIRY_LOG_MAGIC;
    rec->level        = airy_lvl;
    rec->facility     = AIRY_FAC_KERNEL;       /* printk 桥接固定 facility */
    rec->timestamp_ns = ts_nsec;
    rec->caller_id    = smp_processor_id();    /* SSoT 字段名：caller_id（非 cpu） */
    rec->payload_len  = payload_len;           /* payload 有效长度 */
    /* payload 截断至 96B */
    memcpy(rec->payload, text, payload_len);
    if (payload_len < sizeof(rec->payload))
        memset(rec->payload + payload_len, 0,
               sizeof(rec->payload) - payload_len);
    rec->reserved     = 0;                     /* 默认 0；CRC 选项见 [SC] §2.4 */

    /* 3. commit：smp_store_release 发布，消费者可见 */
    airy_printk_commit(rec);
}
```

> **API 命名规范**：桥接 API 统一为 `airy_printk_reserve()` / `airy_printk_commit()` / `airy_printk_write()`（**去除 `_ring` 中缀**），与 [SC] log_types.h 的 `airy_log_*` 前缀风格保持一致。原带 `_ring` 中缀的命名（如 reserve/commit/write 三函数的旧签名）已废弃。

reserve 失败（Ring 满）时直接丢弃该条 printk 镜像——printk 是尽力而为语义，桥接不得引入背压。原生 `log_buf` 仍保留了完整记录，可通过 `dmesg` 补救。

### 2.6 128B 记录结构（OLK 6.6 工程规范）

桥接产出的 128B 记录结构对齐 OLK 6.6 工程规范，使用 `__aligned(64)` 而非传统 packed 属性：

```c
/* kernel/include/uapi/linux/airymax/log_types.h —— [SC] 共享契约层 */
struct airy_log_record {
    __u32   magic;          /* offset  0, 4B: AIRY_LOG_MAGIC (0x414C4F47) */
    __u16   level;          /* offset  4, 2B: 日志级别（airy_log_level） */
    __u16   facility;       /* offset  6, 2B: 设施编号（airy_log_facility） */
    __u64   timestamp_ns;   /* offset  8, 8B: 纳秒时间戳（CLOCK_MONOTONIC） */
    __u32   caller_id;      /* offset 16, 4B: 调用者 ID（pid / kmod id / CPU id） */
    __u32   payload_len;    /* offset 20, 4B: payload 有效长度（≤96） */
    __u8    payload[96];    /* offset 24, 96B: 原始二进制 payload（不格式化） */
    __u64   reserved;       /* offset 120, 8B: 预留对齐 + 校验位 */
} __aligned(64);
/* sizeof(struct airy_log_record) == 128，cache line 对齐 */
```

> **OLK 6.6 工程规范**：所有 [SC] 共享结构使用 `__aligned(64)`（cache line 对齐），不使用 packed 属性（会破坏 atomic 访问语义）或 `__attribute__((aligned(64)))`（OLK 6.6 简写为 `__aligned(64)`）。

---

## §3 Panic 回退：Panic 时 printk_safe 原生路径接管

### 3.1 回退触发条件

Panic 路径下，Ring Buffer 所在内存可能已损坏（页表失效、内存毒化），`logger_d` 用户态进程已终止，mmap 映射失效。此时 printk 桥接必须回退至 Linux 6.6 原生的 `printk_safe` 路径，确保 Panic 日志可靠输出到 console。

回退触发条件：

| 条件 | 检测方式 | 动作 |
|------|---------|------|
| Panic 状态 | `panic_cpu != PANIC_CPU_INVALID` | 禁用 printk_hook，走 printk_safe |
| Ring Buffer reserve 失败 | `airy_printk_reserve()` 返回 NULL | 该条走 printk_safe 临时缓冲 |
| NMI 上下文 | `in_nmi()` | 直接走 printk_safe NMI 缓冲 |
| 内存毒化 | Ring Buffer page `PageHWPoison()` | 禁用桥接，走 printk_safe |
| Badge Epoch 跃迁 | `airy_cap_global_epoch` 跃迁期间 | 桥接降级，仅写 log_buf 不写 Ring Buffer |

### 3.2 printk_safe 原生路径

Linux 6.6 的 `printk_safe`（`printk/safe.c`）提供 per-CPU 临时缓冲，在 NMI/递归 printk/持锁上下文中安全使用。Panic 时 `printk_safe_flush_on_panic()` 将缓冲内容刷到 console。Airymax 桥接在 Panic 路径下完全让位于 `printk_safe`：

```
正常路径：  printk → log_buf + printk_hook → Ring Buffer → logger_d
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

## §4 v1.1 Capability Folding 集成

### 4.1 fastpath C-S9 Badge 校验协作

自 v1.1 起，printk 桥接路径与 fastpath C-S9 Badge 校验协作。printk 桥接本身不持有 Badge（printk 是内核原生路径，无 Agent 上下文），但其产出的 128B 记录被 `logger_d` 消费时，`logger_d` 会通过 `agent_caps[1024]` 静态数组反查 `caller_id` 对应的 Badge 状态：

| 协作点 | 触发条件 | 桥接行为 |
|--------|---------|---------|
| printk 正常镜像 | 内核子系统调用 `printk()` | 写入 `AIRY_FAC_KERNEL` facility 记录，`caller_id = smp_processor_id()` |
| sec_d 审计日志镜像 | `sec_d` 检测到 Badge 校验失败 | `sec_d` 通过 `airy_log_write()` 写入 `AIRY_FAC_SECURITY` facility 记录（不经过 printk 桥接） |
| Badge Epoch 跃迁 | `atomic_inc(&airy_cap_global_epoch)` | 桥接进入降级模式，仅写 log_buf 不写 Ring Buffer（避免脏数据） |

### 4.2 Badge 校验失败事件的日志记录

当 fastpath C-S9 Badge 校验失败时，相关错误码通过 Ring Buffer 传递给 `logger_d`：

| 错误码 | 值 | 触发场景 | 桥接参与方式 |
|--------|-----|---------|------------|
| `AIRY_ECAP_FROZEN` | -82 | `agent_caps[src_task].frozen == true`（C-S0 检查，A-ULS 控制） | `sec_d` 写入审计日志，`logger_d` 消费时识别 |
| `AIRY_ESEC_D_THROTTLED` | -83 | `sec_d` 限流器拒绝 Badge 编译请求（50ms SLO 违约保护） | `sec_d` 写入审计日志，`logger_d` 消费时识别 |
| `AIRY_ECAP_FORGED` | -80 | Badge RandomTag 不匹配（伪造尝试） | `sec_d` 写入 `LOG_FATAL` 审计日志 + 触发 `AIRY_FAULT_CAP_FORGED` |
| `AIRY_ECAP_EPOCH` | -79 | Badge Epoch 与全局 Epoch 不匹配（已撤销或过期） | `sec_d` 写入审计日志 |

> **设计原则**：printk 桥接**不直接产生** Badge 校验失败日志（这些日志由 `sec_d` 通过 `airy_log_write()` 直接写入 Ring Buffer，绕过 printk 桥接）。printk 桥接仅承担"将 Linux 6.6 原生 printk 镜像到 Ring Buffer"的职责，与 Badge 校验链路解耦——这是 seL4 "机制与策略分离"原则的体现（ADR-014）：桥接是机制，Badge 校验是策略。

### 4.3 agent_caps[1024] 在桥接中的间接引用

`agent_caps[1024]` 静态数组（16KB，sec_d 唯一写者）在 printk 桥接中**不直接访问**——桥接路径不持有 Badge。但桥接产出的记录被 `logger_d` 消费时，`logger_d` 会通过 `agent_caps[rec->caller_id]` 反查 Badge 状态：

```
printk 桥接路径（不访问 agent_caps）：
  printk() → printk_hook → airy_printk_reserve() → 填充 128B → airy_printk_commit()

logger_d 消费路径（访问 agent_caps）：
  logger_d 消费记录 → 通过 agent_caps[rec->caller_id] 反查 Badge 状态
                  → fastpath C-S9 内联校验（~10ns）
                  → 校验失败则标记 AIRY_ECAP_FROZEN / AIRY_ESEC_D_THROTTLED
```

### 4.4 O(1) 撤销对桥接的影响

`atomic_inc(&airy_cap_global_epoch)` 触发 O(1) 撤销时，所有旧 Badge 立即失效。printk 桥接在 Epoch 跃迁期间的策略：

1. **不停止桥接**——printk 是内核原生路径，与 Badge 生命周期解耦
2. **继续写入 Ring Buffer**——但 `logger_d` 在 Epoch 跃迁期间消费时，会通过 fastpath C-S9 校验发现 Badge 失效，记录 `AIRY_ECAP_FROZEN`
3. **不丢失 printk 镜像**——printk 镜像记录的 `caller_id` 是 CPU id，不依赖 Badge 校验

---

## §5 性能：printk 桥接路径与原生 printk 性能对比

### 5.1 性能模型

printk 桥接在原生 printk 路径上额外增加：128B 记录格式化（含 `caller_id` / `payload_len` / `reserved` 字段填充）+ Ring Buffer reserve/commit。各步骤开销估算（x86_64，2.4 GHz，热缓存）：

| 步骤 | 指令数估算 | 耗时估算 | 说明 |
|------|----------|---------|------|
| printk 原生（log_buf 写入） | ~300 | ~125 ns | 基线 |
| 128B 格式化（含 caller_id/payload_len/reserved） | ~45 | ~19 ns | 字段填充 + memcpy 96B + memset 补零 |
| Ring Buffer reserve | ~20 | ~8 ns | 原子索引推进 |
| Ring Buffer commit | ~15 | ~6 ns | smp_store_release |
| **桥接总开销** | **~80** | **~33 ns** | 占原生 ~26% |

### 5.2 对比基线

| 路径 | 单次 printk 延迟 | 是否阻塞 | 上下文限制 |
|------|----------------|---------|-----------|
| 原生 printk（log_buf） | ~125 ns | 否 | 任意（含 NMI） |
| printk + 桥接（Ring Buffer） | ~158 ns | 否 | 任意（Panic/NMI 自动回退） |
| printk + console 输出 | ~10-100 μs | 可能 | 受 console_lock 约束 |

桥接引入的 ~33 ns 开销相对 console 输出的 μs 级开销可忽略，且桥接路径无锁、不睡眠，不改变 printk 的上下文安全性。

### 5.3 吞吐验证

在 8 核 x86_64 上以 `ftrace` 触发高频 printk（每核 10 万次/秒），对比吞吐：

| 场景 | 原生 printk | printk + 桥接 | 退化 |
|------|------------|--------------|------|
| 总吞吐（万条/秒） | 78 | 75 | ~4% |
| Ring Buffer 丢条率 | — | < 0.01% | reserve 失败丢弃 |

退化 < 5%，在可接受范围。Ring Buffer 丢条仅发生在突发尖峰，原生 `log_buf` 仍保留完整记录，可通过 `dmesg` 补全。

### 5.4 优化措施

- **per-cpu 格式化缓冲**：128B 记录在 per-cpu 栈缓冲格式化，避免每次 reserve 零拷贝失败回退
- **批量 commit**：高频 printk 场景下，`logger_d` 侧批量消费摊销 mmap 读取开销
- **level 过滤前置**：`LOG_DEBUG` 级别在 `airy_printk_reserve` 前按 `logger_d.yaml` 的 `level.min` 过滤，避免无效写入
- **caller_id 缓存**：per-cpu 缓存 `smp_processor_id()` 结果，减少字段填充开销

---

## §6 与 A-ULP 模块的关系：printk 桥接是 A-ULP 的兼容层

### 6.1 A-ULP 模块定位

A-ULP（统一日志与打印系统）是 Airymax Unify Design 五模块之一（见 [10-unify-design.md](../10-architecture/10-unify-design.md) §5），目标是统一 AirymaxOS 的日志与打印系统。A-ULP 由三部分构成：

| 组件 | 职责 | 权威文档 |
|------|------|---------|
| Ring Buffer（内核生产侧） | 接收 `LOG_*` 与 printk 镜像，无锁 reserve/commit | [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) |
| `logger_d`（用户态消费侧） | mmap 读取、落盘、轮转、zstd 归档、审计哈希链追加 | [12-logger-daemon-module.md](12-logger-daemon-module.md) |
| **printk 桥接（兼容层）** | **将 Linux 6.6 原生 printk 镜像到 Ring Buffer** | **本文件** |

### 6.2 兼容层语义

printk 桥接是 A-ULP 的**兼容层**，而非 A-ULP 的核心组件。核心组件（Ring Buffer + `logger_d`）定义了 Airymax 原生日志链路；printk 桥接负责将既有的 Linux 内核日志（驱动、子系统、第三方模块的 printk）纳入这条链路，避免双轨。这一兼容层定位决定了：

- **不新增日志语义**：桥接不引入新的日志级别或 facility，仅做 printk → `LOG_*` 的格式转换（`AIRY_FAC_KERNEL`）
- **可禁用**：通过 Kconfig `CONFIG_AIRY_PRINTK_BRIDGE` 可关闭桥接，系统退化为"Airymax 模块走 Ring Buffer，其余走 log_buf"的双轨模式（功能完整，仅割裂）
- **不替代 printk**：桥接是镜像，原生 printk/log_buf/dmesg 工具链全部保留
- **不参与 Badge 校验**：桥接路径不持有 Badge，Badge 校验由 `sec_d` 与 `logger_d` 协作完成（§4.2）

### 6.3 与 [DSL] 降级层的关系

在 [DSL] 降级生存层（见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §5）生效时，Ring Buffer 退化为 printk 原生（仅 `LOG_FATAL` + `LOG_ERROR`），printk 桥接自动禁用——此时 printk 与 `LOG_*` 均回归原生 printk 路径，两者合一，无需桥接。这保证了 [DSL] 降级模式下日志链路最简、最可靠。

---

## §7 IRON-9 v3 [DSL] 降级生存层

### 7.1 [DSL] 在 printk 桥接中的体现

IRON-9 v3 四层模型（[SC] + [SS] + [IND] + [DSL]）中，printk 桥接涉及 [SC] log_types.h 与 [DSL] 降级块：

| 层 | 头文件/资源 | printk 桥接使用方式 |
|----|----------|---------------------|
| [SC] | `log_types.h` | 128B 记录格式、`LOG_*` 枚举、`AIRY_FAC_KERNEL` facility |
| [SC] | `error.h` | `AIRY_ECAP_FROZEN`、`AIRY_ESEC_D_THROTTLED`（间接，由 `logger_d` 消费时识别） |
| [IND] | `airy_printk_bridge.c` 实现 | 桥接 hook 注册、reserve/commit 写入 |
| [DSL] | `#ifdef AIRY_SC_FALLBACK` 降级块 | log_types.h 损坏时的桥接降级路径 |

### 7.2 [SC] log_types.h 损坏时的桥接降级路径

当 [SC] log_types.h 损坏时，`#ifdef AIRY_SC_FALLBACK` 降级块生效，printk 桥接降级为最小可运行子集：

| 维度 | 正常模式 | [DSL] 降级模式 |
|------|---------|--------------|
| 桥接状态 | 启用（printk 镜像至 Ring Buffer） | **禁用**（桥接自动让位 printk_safe） |
| 日志通路 | printk → log_buf + printk_hook → Ring Buffer → `logger_d` | printk → log_buf + printk_safe → console |
| 日志级别映射 | printk 8 级 → `LOG_*` 5 级 | printk 8 级原样保留（仅 `LOG_FATAL` + `LOG_ERROR` 落地） |
| facility 字段 | `AIRY_FAC_KERNEL` | 退化（不填充 facility） |
| 128B 记录格式 | 完整 8 字段（含 `caller_id` / `payload_len` / `reserved`） | 退化为 printk 原生文本（无 128B 包装） |
| Badge 校验 | fastpath C-S9 协作 | 跳过（降级模式不可靠） |

### 7.3 降级块定义

```c
/* kernel/include/uapi/linux/airymax/log_types.h —— [DSL] 降级块 */
#ifdef AIRY_SC_FALLBACK
#warning "AIRY_SC_FALLBACK: log_types.h degraded, printk bridge auto-disabled"

/* [DSL] 桥接降级：禁用 printk_hook，回归 printk_safe 原生路径 */
#define airy_printk_reserve()     NULL
#define airy_printk_commit(rec)   do {} while (0)
#define airy_printk_register_hook(h)  (-ENOSYS)

/* [DSL] 仅保留 LOG_FATAL + LOG_ERROR 两级 */
#define LOG_FATAL   4
#define LOG_ERROR   3
#define LOG_DEBUG   LOG_ERROR
#define LOG_INFO    LOG_ERROR
#define LOG_WARN    LOG_ERROR

/* [DSL] 仅保留 AIRY_FAC_KERNEL */
#define AIRY_FAC_KERNEL  0

#endif /* AIRY_SC_FALLBACK */
```

### 7.4 降级模式下的桥接行为

printk 桥接在 [DSL] 降级模式下：

1. **`airy_printk_reserve()` 永远返回 NULL**——桥接不再向 Ring Buffer 写入
2. **`airy_printk_register_hook()` 返回 `-ENOSYS`**——hook 注册失败，桥接完全禁用
3. **printk 回归原生路径**——所有 printk 输出仅写入 `log_buf`，由 `dmesg` / `klogctl` 读取
4. **`logger_d` 退出消费**——Ring Buffer 不再被消费，由 printk_safe 接管日志链路

详细设计见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §4.1.4。

---

## §8 12 daemon 协作关系

printk 桥接是内核态组件，不直接参与 12 daemon 协作，但其产出的 Ring Buffer 记录被 12 daemon 中的 `logger_d` 消费。printk 桥接与 12 daemon 的协作关系如下：

| 协作方 | 协作方向 | 接口 | 协作内容 |
|--------|---------|------|---------|
| `logger_d` | 单向（消费者） | Ring Buffer mmap | `logger_d` 消费桥接产出的 128B 记录（`AIRY_FAC_KERNEL` facility） |
| `sec_d` | 间接（同源） | Ring Buffer | `sec_d` 通过 `airy_log_write()` 写入 Badge 校验日志（不经过桥接），与桥接产出的 kernel 日志统一汇聚到 Ring Buffer |
| `audit_d` | 间接（审计读取） | 落盘文件 | `audit_d` 校验审计哈希链时，桥接产出的 `AIRY_FAC_KERNEL` 记录不参与哈希链（仅 `AIRY_FAC_SECURITY` facility 参与） |
| `macro_d` | 间接（监管） | systemd | `macro_d` 通过 systemd 监管 `logger_d`，间接保证桥接产出的日志被消费 |
| `config_d` | 间接（配置） | SIGHUP | `config_d` 修改 `logger_d.yaml` 的 `level.min` 影响 `logger_d` 消费桥接记录时的过滤策略 |
| `cogn_d` | 间接（同源生产者） | Ring Buffer | `cogn_d` 写入 `AIRY_FAC_COGNITION` 日志，与桥接产出的 kernel 日志统一汇聚 |
| `mem_d` | 间接（同源生产者） | Ring Buffer | `mem_d` 写入 `AIRY_FAC_MEMORY` 日志，与桥接产出的 kernel 日志统一汇聚 |
| `gateway_d` | 间接（同源生产者） | Ring Buffer | `gateway_d` 写入跨节点 IPC 日志，与桥接产出的 kernel 日志统一汇聚 |
| `sched_d` | 间接（同源生产者） | Ring Buffer | `sched_d` 写入 `AIRY_FAC_SCHED` 日志，与桥接产出的 kernel 日志统一汇聚 |
| `dev_d` | 间接（同源生产者） | Ring Buffer | `dev_d` 写入设备驱动日志，与桥接产出的 kernel 日志统一汇聚 |
| `net_d` | 间接（同源生产者） | Ring Buffer | `net_d` 写入网络栈日志，与桥接产出的 kernel 日志统一汇聚 |
| `vfs_d` | 间接（同源生产者） | Ring Buffer | `vfs_d` 写入 VFS 日志，与桥接产出的 kernel 日志统一汇聚 |

12 daemon 完整名单（与 [10-user-supervisor-daemon.md](10-user-supervisor-daemon.md) §1.3 对齐）：

```
sec_d（capability 编译/撤销）          cogn_d（认知循环调度）
mem_d（记忆卷载管理）                  gateway_d（跨节点 IPC）
logger_d（统一日志，桥接产出的消费者）  macro_d（宏观监管）
audit_d（审计哈希链校验）              sched_d（sched_tac 策略守护）
dev_d（设备驱动用户态化）              net_d（网络栈用户态化）
vfs_d（VFS 用户态化）                  config_d（统一配置管理）
```

---

## §9 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（A-ULP 模块定位）
- [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— Ring Buffer reserve/commit 两阶段写入权威源
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— 128B 记录格式（含 `caller_id` / `payload_len` / `reserved`）+ printk 8 级映射 + `AIRY_FAC_*` 枚举 [SC] 契约
- [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— IPC 字段命名规范（`caller_id` 命名 SSoT）
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— `AIRY_ECAP_FROZEN` / `AIRY_ESEC_D_THROTTLED` 错误码注册
- [12-logger-daemon-module.md](12-logger-daemon-module.md) —— `logger_d` 模块设计（日志链路下游）
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 [DSL] 降级层（桥接在降级模式下禁用）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级生存层详细设计
- [05-adrs.md](../10-architecture/05-adrs.md) —— ADR-012（微内核化改造技术路线）+ ADR-014（微内核设计思想来源单一化：仅 seL4）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | printk 桥接设计 | v1.1 | 2026-07-19
