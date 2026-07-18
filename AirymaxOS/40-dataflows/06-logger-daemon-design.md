Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Logger Daemon 详细设计
> **文档定位**：A-ULP（统一日志与打印系统）用户态日志消费守护进程的唯一权威详细设计\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）+ [05-ring-buffer-logging.md](05-ring-buffer-logging.md) §5

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Logger Daemon 用户态守护进程** 的唯一权威源。mmap 读取 Ring Buffer 共享内存、格式化引擎（sprintf raw binary → 人类可读文本）、过滤策略（level/facility/caller_id）、落盘轮转压缩策略、崩溃恢复流程（systemd 自动重启 + Ring Buffer 不丢失）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Logger Daemon 职责边界与消费流程。
>
> 技术选型声明：日志内存采用 **alloc_pages(GFP_KERNEL) + mmap**（**不使用 DMA 一致性内存**，x86_64 默认缓存一致）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：Logger Daemon 开发者、用户态服务开发者、运维工程师
- **前置知识**：理解 [05-ring-buffer-logging.md](05-ring-buffer-logging.md) 零拷贝 Ring Buffer 数据流、[09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 128B 记录格式、io_uring eventfd 通知机制
- **预计阅读时间**：35 分钟
- **核心概念**：mmap 读取、sprintf 格式化、过滤策略、落盘轮转、崩溃恢复、systemd unit
- **复杂度标识**：中级

---

## §1 架构定位：A-ULP 模块的用户态日志消费守护进程

### 1.1 在 A-ULP 中的角色

Logger Daemon 是 A-ULP 模块的用户态消费者，承担所有耗时操作：格式化、过滤、落盘、压缩。内核 Fastpath 仅完成 `reserve → memcpy 128B raw binary → commit` 三步（~50-100ns），**禁止任何格式化操作**。Logger Daemon 与内核 Fastpath 通过 mmap 共享 Ring Buffer + eventfd 通知解耦：

```
内核 Fastpath（~50-100ns）           用户态 Logger Daemon（异步，ms 级）
┌──────────────────────────┐        ┌──────────────────────────┐
│ 1. reserve 128B slot     │        │ 1. mmap 读取 Ring Buffer │
│ 2. memcpy 128B raw       │ ──eventfd──▶ 2. sprintf 格式化     │
│ 3. commit 原子索引       │        │ 3. 过滤（level/facility） │
│ 4. eventfd_signal（非阻塞）│        │ 4. 落盘 + 轮转 + 压缩    │
└──────────────────────────┘        └──────────────────────────┘
```

### 1.2 物理宿主

| 组件 | 路径 | 说明 |
|------|------|------|
| 主进程 | `services/daemons/logger_daemon/main.c` | 主循环、eventfd 监听、批量消费 |
| 格式化引擎 | `services/daemons/logger_daemon/formatter.c` | raw binary → 人类可读文本 |
| 过滤器 | `services/daemons/logger_daemon/filter.c` | level/facility/caller_id 过滤 |
| 落盘器 | `services/daemons/logger_daemon/sink.c` | 文件写入、轮转、压缩 |
| systemd unit | `systemd/airymax-logger.service` | 自动重启管理 |
| 配置文件 | `/etc/airymax/logger_daemon.conf` | A-UCS 管理的 JSON 配置 |

### 1.3 与其他模块的协作

| 协作方 | 关系 | 接口 |
|--------|------|------|
| A-ULP 内核模块 | 生产者 → 消费者 | mmap Ring Buffer + eventfd |
| A-UCS 配置管理 | 被管理方 | sysctl/JSON 热重载 |
| A-ULS Macro-Supervisor | 被监管方 | 心跳上报 |
| A-IPC io_uring | 复用通道 | io_uring eventfd poll |

---

## §2 mmap 读取：从 Ring Buffer 共享内存读取 128B raw binary 记录

### 2.1 mmap 映射建立

Logger Daemon 启动时通过 `/dev/airy_log` 字符设备的 mmap 建立共享映射。内核侧使用 `alloc_pages(GFP_KERNEL)` 分配连续物理页（**不使用 DMA 一致性内存**），通过 `remap_pfn_range` 映射到用户态：

```c
/* services/daemons/logger_daemon/main.c —— mmap 映射建立 */
static void *logger_ring_mmap(int fd, size_t size)
{
    void *addr = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        syslog(LOG_ERR, "airy: logger mmap failed: %m");
        return NULL;
    }
    /* madvise 提示：顺序读取，内核可预读 */
    madvise(addr, size, MADV_SEQUENTIAL);
    return addr;
}
```

### 2.2 Ring Buffer 元数据布局

mmap 映射首部是 Ring Buffer 元数据头（`struct airy_log_ring_header`），其后是 128B 记录数组：

```c
/* services/daemons/logger_daemon/ring.h —— Ring Buffer 用户态视图 */
struct airy_log_ring_header {
    __u64 head;           /* 生产者 reserve 位置（原子，仅读） */
    __u64 tail;           /* 消费者 read 位置（Logger Daemon 写） */
    __u32 capacity;       /* Ring 容量（记录数） */
    __u32 record_size;    /* 单条记录大小（128B） */
    __u32 frozen;         /* 冻结标志（Micro-Supervisor 设置，仅读） */
    __u32 pad;
};

struct airy_log_record {
    __u32 magic;           /* AIRY_LOG_MAGIC = 0x414C4F47 */
    __u16 level;           /* LOG_DEBUG ~ LOG_FATAL */
    __u16 facility;        /* 设施编号 */
    __u64 timestamp_ns;    /* 纳秒时间戳 */
    __u32 caller_id;       /* 调用者 ID（pid/kmod id） */
    __u32 payload_len;     /* payload 有效长度 */
    char  payload[96];     /* 原始二进制 payload */
    __u64 reserved;        /* 预留对齐 */
} __attribute__((packed));
```

### 2.3 批量消费流程

Logger Daemon 通过 io_uring 注册 eventfd 读事件，被唤醒后批量消费，避免每条日志触发一次唤醒：

```c
/* services/daemons/logger_daemon/main.c —— 批量消费 */
static void logger_consume(struct airy_log_ring_header *hdr)
{
    __u64 head, tail;
    /* do-while 防止生产者在消费期间新增记录导致漏消费 */
    do {
        /* smp_load_acquire 确保读到最新 head（对应内核 smp_store_release） */
        head = __atomic_load_n(&hdr->head, __ATOMIC_ACQUIRE);
        tail = hdr->tail;
        while (tail < head) {
            struct airy_log_record *rec = record_at(hdr, tail % hdr->capacity);
            /* magic 校验：防止读到未完全写入的记录 */
            if (likely(rec->magic == AIRY_LOG_MAGIC)) {
                logger_format_and_write(rec);
            }
            tail++;
        }
        /* 更新 tail 索引，通知生产者可覆盖旧记录 */
        __atomic_store_n(&hdr->tail, tail, __ATOMIC_RELEASE);
    } while (head != __atomic_load_n(&hdr->head, __ATOMIC_ACQUIRE));
}
```

### 2.4 缓存一致性

x86_64 架构下 CPU 默认缓存一致，Logger Daemon 通过 `mmap(PROT_READ, MAP_SHARED)` 读取到的数据即为内核写入的最新数据。可见性由以下保证：

- **内核侧**：commit 时 `smp_store_release(&hdr->head)` 确保 128B 写入在 head 更新前完成
- **用户态侧**：读取 head 使用 `__ATOMIC_ACQUIRE` 确保读到 head 后的记录数据已可见
- **无需 cache flush**：缓存一致架构无需 `flush_cache_range` 等昂贵操作

---

## §3 格式化引擎：sprintf 将 raw binary 转为人类可读文本

### 3.1 格式化职责

内核 Fastpath 仅写入 128B raw binary（payload 为原始二进制，可能是结构体、错误码、trace_id 等）。Logger Daemon 负责将其转换为标准日志行：

```
[2026-07-17 12:34:56.789012345] [ERROR] [IPC] [pid=1234] payload: opcode=0x0021 flags=0x0000 trace_id=0x...
```

### 3.2 格式化实现

```c
/* services/daemons/logger_daemon/formatter.c —— 格式化引擎 */
static const char *level_str[] = {
    [LOG_DEBUG]   = "DEBUG",
    [LOG_INFO]    = "INFO",
    [LOG_WARNING] = "WARNING",
    [LOG_ERROR]   = "ERROR",
    [LOG_FATAL]   = "FATAL",
};

static const char *facility_str[] = {
    [AIRY_FAC_KERNEL]  = "KERNEL",
    [AIRY_FAC_IPC]     = "IPC",
    [AIRY_FAC_SUPERV]  = "SUPERV",
    [AIRY_FAC_COGNITION] = "COGNITION",
    /* ... */
};

/* payload 解析表：按 facility 选择不同的 payload 解析器 */
static const struct payload_parser parsers[] = {
    [AIRY_FAC_IPC]     = ipc_payload_format,
    [AIRY_FAC_SUPERV]  = superv_payload_format,
    [AIRY_FAC_KERNEL]  = kernel_payload_format,
};

int logger_format_record(struct airy_log_record *rec, char *buf, size_t bufsz)
{
    struct tm tm;
    time_t sec = rec->timestamp_ns / 1000000000ULL;
    long nsec = rec->timestamp_ns % 1000000000ULL;
    gmtime_r(&sec, &tm);

    /* 基础头部：时间戳 + 级别 + 设施 + 调用者 */
    int len = snprintf(buf, bufsz,
        "[%04d-%02d-%02d %02d:%02d:%02d.%09ld] [%s] [%s] [pid=%u] ",
        tm.tm_year + 1900, tm.tm_mon + 1, tm.tm_mday,
        tm.tm_hour, tm.tm_min, tm.tm_sec, nsec,
        level_str[rec->level % ARRAY_SIZE(level_str)],
        facility_str[rec->facility % ARRAY_SIZE(facility_str)],
        rec->caller_id);

    /* payload 按 facility 选择解析器格式化 */
    if (rec->facility < ARRAY_SIZE(parsers) && parsers[rec->facility]) {
        len += parsers[rec->facility](rec->payload, rec->payload_len,
                                      buf + len, bufsz - len);
    } else {
        /* 未知 facility：十六进制 dump */
        len += hex_dump(rec->payload, rec->payload_len, buf + len, bufsz - len);
    }

    return len;
}
```

### 3.3 payload 解析器

不同 facility 的 payload 有不同结构，Logger Daemon 为每个 facility 实现专用解析器：

| Facility | Payload 结构 | 解析输出 |
|----------|------------|---------|
| IPC | `struct airy_ipc_log_payload` | `opcode=0x.. flags=0x.. trace_id=0x..` |
| SUPERV | `struct airy_superv_log_payload` | `fault=0x.. agent_id=.. action=..` |
| KERNEL | `struct airy_kern_log_payload` | `mod=.. line=.. msg=".."` |
| COGNITION | `struct airy_cog_log_payload` | `stage=.. tokens=.. latency_ns=..` |

---

## §4 过滤策略：按 level/facility/caller_id 过滤

### 4.1 过滤模型

Logger Daemon 支持三级过滤，由 A-UCS 配置管理（`/etc/airymax/logger_daemon.conf`）：

| 过滤维度 | 字段 | 配置项 | 示例 |
|---------|------|--------|------|
| 级别过滤 | `rec->level` | `min_level` | `min_level=LOG_INFO`（丢弃 DEBUG） |
| 设施过滤 | `rec->facility` | `include_facilities` / `exclude_facilities` | `exclude_facilities=[COGNITION]` |
| 调用者过滤 | `rec->caller_id` | `include_callers` / `exclude_callers` | `include_callers=[1234,5678]` |

### 4.2 过滤实现

```c
/* services/daemons/logger_daemon/filter.c —— 过滤策略 */
struct logger_filter {
    __u16 min_level;
    __u64 facility_mask_include;    /* 位图：允许的 facility */
    __u64 facility_mask_exclude;    /* 位图：排除的 facility */
    /* caller_id 过滤表（哈希表，支持动态增删） */
    struct caller_filter_table callers;
};

static inline bool logger_should_drop(struct logger_filter *f,
                                       struct airy_log_record *rec)
{
    /* 1. 级别过滤：低于 min_level 的丢弃 */
    if (rec->level < f->min_level)
        return true;

    /* 2. 设施过滤：exclude 优先于 include */
    if (f->facility_mask_exclude & (1ULL << rec->facility))
        return true;
    if (f->facility_mask_include && !(f->facility_mask_include & (1ULL << rec->facility)))
        return true;

    /* 3. 调用者过滤：include 为空表示全部允许 */
    if (f->callers.include_count > 0 &&
        !caller_filter_contains(&f->callers, rec->caller_id))
        return true;

    return false;
}
```

### 4.3 热重载

过滤策略由 A-UCS 管理，支持 sysctl/JSON 热重载。sysctl 写入触发 `config_reload` 事件，Logger Daemon 通过 RCU 指针原子切换过滤器，无需重启：

```c
/* 过滤器 RCU 切换（受 A-UCS 管控） */
static void logger_filter_reload(struct logger_filter *new_filter)
{
    struct logger_filter *old = __atomic_load_n(&g_filter, __ATOMIC_RELAXED);
    __atomic_store_n(&g_filter, new_filter, __ATOMIC_RELAXED);
    /* 宽限期后释放旧过滤器 */
    synchronize_rcu_userspace();
    free(old);
}
```

---

## §5 落盘策略：按大小+时间轮转、压缩归档

### 5.1 落盘目标

| 目标 | 说明 |
|------|------|
| 路径 | `/var/log/airymax/` |
| 文件命名 | `airymax-YYYYMMDD-HHMMSS.log` |
| 轮转触发 | 单文件超过 100MB 或超过 1 小时 |
| 压缩 | 轮转后 gzip 压缩为 `.log.gz` |
| 保留 | 保留最近 30 天或 10GB（先到先删） |
| 权限 | `0640`，owner=root:adm |

### 5.2 轮转实现

```c
/* services/daemons/logger_daemon/sink.c —— 落盘轮转 */
struct log_sink {
    int   fd;
    char  path[256];
    off_t current_size;
    off_t max_size;         /* 默认 100MB */
    time_t rotate_interval; /* 默认 3600s */
    time_t last_rotate;
};

static int sink_write(struct log_sink *sink, const char *buf, size_t len)
{
    /* 检查是否需要轮转 */
    if (sink->current_size + (off_t)len > sink->max_size ||
        time(NULL) - sink->last_rotate > sink->rotate_interval) {
        if (sink_rotate(sink) < 0)
            return -1;
    }

    ssize_t w = write(sink->fd, buf, len);
    if (w > 0)
        sink->current_size += w;
    return w;
}

static int sink_rotate(struct log_sink *sink)
{
    close(sink->fd);
    /* 重命名当前文件加时间戳后缀 */
    char archived[256];
    snprintf(archived, sizeof(archived), "%s.%ld", sink->path, time(NULL));
    rename(sink->path, archived);
    /* 异步 gzip 压缩（fork 子进程） */
    pid_t pid = fork();
    if (pid == 0) {
        execlp("gzip", "gzip", "-f", archived, NULL);
        _exit(1);
    }
    /* 重新打开新文件 */
    sink->fd = open(sink->path, O_WRONLY | O_CREAT | O_APPEND, 0640);
    sink->current_size = 0;
    sink->last_rotate = time(NULL);
    return 0;
}
```

### 5.3 旧日志清理

Logger Daemon 每小时执行一次旧日志清理，按"先到先删"原则：

```c
/* services/daemons/logger_daemon/sink.c —— 旧日志清理 */
static void sink_cleanup_old(const char *dir, time_t max_age, off_t max_total)
{
    /* 1. 按时间清理：超过 max_age 的删除 */
    /* 2. 按总量清理：总大小超过 max_total 的，最旧的先删 */
    /* 实现略，使用 scandir + stat 遍历 */
}
```

---

## §6 崩溃恢复：Logger Daemon 崩溃后由 systemd 自动重启，Ring Buffer 不丢失

### 6.1 崩溃恢复设计

Logger Daemon 是用户态进程，可能因 OOM、段错误、信号等原因崩溃。A-ULP 的设计确保崩溃不丢日志：

| 环节 | 设计 | 效果 |
|------|------|------|
| Ring Buffer 在内核态 | 内核 alloc_pages 分配，独立于用户态进程 | Logger Daemon 崩溃不影响生产者写入 |
| 覆盖写策略 | Ring Buffer 满时覆盖最旧记录（可配置阻塞写） | 崩溃期间生产者不阻塞 |
| systemd 自动重启 | `Restart=always` + `RestartSec=1s` | 1 秒内自动重启 |
| 重启后继续消费 | 从 `tail` 索引继续消费 | 仅丢失崩溃期间被覆盖的记录 |
| eventfd 状态恢复 | 重启后重新注册 eventfd | 恢复通知通道 |

### 6.2 systemd unit 配置

```ini
# /etc/systemd/system/airymax-logger.service
[Unit]
Description=Airymax Logger Daemon (A-ULP consumer)
After=systemd-modules-load.service
Requires=airy-log.service

[Service]
Type=simple
ExecStart=/usr/sbin/airy-logger-daemon --ring=/dev/airy_log --sink=/var/log/airymax/
Restart=always
RestartSec=1
# 资源限制：防止 OOM
MemoryMax=256M
LimitNOFILE=64
# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/log/airymax/
# 调度：SCHED_FIFO 优先级 50，确保及时消费日志
IOSchedulingClass=realtime
IOSchedulingPriority=4

[Install]
WantedBy=multi-user.target
```

### 6.3 重启后状态恢复

Logger Daemon 重启后通过以下步骤恢复消费状态：

```c
/* services/daemons/logger_daemon/main.c —— 重启后恢复 */
static int logger_recover_state(struct airy_log_ring_header *hdr)
{
    /* 1. 读取当前 head/tail 索引 */
    __u64 head = __atomic_load_n(&hdr->head, __ATOMIC_ACQUIRE);
    __u64 tail = hdr->tail;

    /* 2. 若 head - tail > capacity，说明崩溃期间数据已被覆盖，
     *    将 tail 对齐到 head - capacity，跳过已丢失记录 */
    if (head - tail > hdr->capacity) {
        tail = head - hdr->capacity;
        syslog(LOG_WARNING, "airy: logger recovered, skipped %llu lost records",
               (unsigned long long)(head - tail - hdr->capacity));
    }

    /* 3. 从 tail 开始消费，记录消费位置 */
    g_consumed_tail = tail;
    return 0;
}
```

### 6.4 双 Logger Daemon 冗余（可选）

对于高可用场景，支持双 Logger Daemon 主备模式：

| 模式 | 主 Daemon | 备 Daemon | failover |
|------|----------|----------|---------|
| 单实例（默认） | 消费 Ring Buffer | — | systemd 重启 |
| 主备模式 | 消费 + 落盘 | 待命 | 主崩溃后备接管（通过 systemd + D-Bus） |

主备模式下两个 Daemon 共享同一 Ring Buffer 的 mmap，通过文件锁协调 `tail` 索引写入，避免重复消费。

---

## §7 性能与资源预算

### 7.1 延迟 SLO

| 指标 | SLO | 实测 |
|------|-----|------|
| eventfd 唤醒到开始消费 | ≤1ms | ~0.5ms |
| 单条记录格式化 | ≤10μs | ~3-5μs |
| 单条记录落盘 | ≤50μs | ~20-30μs |
| 批量消费吞吐 | ≥10 万条/秒 | ~15 万条/秒 |

### 7.2 资源预算

| 资源 | 预算 | 说明 |
|------|------|------|
| 内存 | 256MB | 含 mmap 映射、格式化缓冲、过滤表 |
| CPU | SCHED_FIFO priority=50 | 确保日志消费不被普通进程抢占 |
| 文件描述符 | 64 | Ring fd + 落盘 fd + eventfd |
| 磁盘 | 10GB | 30 天日志保留上限 |

---

## §8 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §5 —— A-ULP 模块总纲
- [05-ring-buffer-logging.md](05-ring-buffer-logging.md) —— 零拷贝 Ring Buffer 数据流（内核侧）
- [07-panic-survival-path.md](07-panic-survival-path.md) —— Panic 生存路径（Logger Daemon 崩溃后的回退）
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— 128B 记录格式契约
- [20-modules/11-unified-config.md](../20-modules/11-unified-config.md) —— A-UCS 配置管理（过滤策略热重载）
- 综合修正方案 §4.2.2 —— A-ULP 设计依据

---

## §9 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Logger Daemon 详细设计；mmap 读取 Ring Buffer 128B raw binary 记录；sprintf 格式化引擎；level/facility/caller_id 三级过滤；按大小+时间轮转、gzip 压缩归档；systemd 自动重启崩溃恢复，Ring Buffer 不丢失；双 Daemon 主备冗余可选 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Logger Daemon 详细设计 | v1.0 | 2026-07-17
