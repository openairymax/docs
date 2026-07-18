Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Logger Daemon 模块设计

> **文档定位**：AirymaxOS（agentrt-linux）Logger Daemon 模块级设计——systemd unit 编排、配置文件 schema、日志轮转策略、zstd 压缩归档，作为 A-ULP（统一日志与打印系统）用户态消费侧的模块工程契约\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5（A-ULP 模块）\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）+ §6.2.1 C-07（DMA 一致性内存修正）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Logger Daemon 模块级设计** 的唯一权威源。`airy-logger.service` systemd unit 编排、`/etc/agentrt/logger.conf` 配置 schema、按大小+时间的双维度轮转策略、zstd 压缩归档流程均以本文件为唯一权威定义。
>
> 本文件遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) A-ULP 模块设计与 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) 数据流设计。技术选型对齐 Unify Design：日志内存采用 `alloc_pages(GFP_KERNEL)` + mmap（不使用 DMA 一致性内存），调度采用sched_tac（不使用 sched_ext）。`LOG_*` 5 级日志枚举与 128B 记录格式的权威源为 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)。

---

## §1 systemd unit 设计：airy-logger.service

### 1.1 模块定位

Logger Daemon 是 A-ULP 模块的用户态消费侧守护进程，从内核 Ring Buffer（见 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)）mmap 映射区读取 128B 日志记录，按 `/etc/agentrt/logger.conf` 配置写入持久化文件，并执行轮转与压缩归档。Logger Daemon **不参与日志生产**，仅承担消费、落盘、轮转、归档四项职责，与内核 Ring Buffer 生产者彻底解耦。

### 1.2 unit 文件

Logger Daemon 通过 systemd unit `airy-logger.service` 编排，作为系统早期服务拉起，确保 Panic 回退路径之前已就绪：

```ini
# /usr/lib/systemd/system/airy-logger.service
[Unit]
Description=Airymax Logger Daemon (A-ULP consumer)
Documentation=file:///usr/share/doc/agentrt/20-modules/12-logger-daemon-module.md
After=agentrt-init.service
Before=multi-user.target
ConditionPathExists=/etc/agentrt/logger.conf
DefaultDependencies=no

[Service]
Type=simple
ExecStart=/usr/sbin/airy-loggerd --config /etc/agentrt/logger.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=1s
# OOM 不可杀，日志是故障诊断最后防线
OOMScoreAdjust=-1000
# 调度对齐sched_tac：Logger Daemon 用 SCHED_FIFO 高优先级
# 保证日志消费不背压内核 Ring Buffer
Nice=-10
LimitNOFILE=1024
ProtectSystem=strict
ReadWritePaths=/var/log/agentrt
ProtectHome=yes
PrivateTmp=yes
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

### 1.3 启动顺序与依赖

| 依赖项 | 类型 | 说明 |
|--------|------|------|
| `agentrt-init.service` | `After=` | 等待 capability 初始化守护进程就绪，Logger Daemon 需 capability 访问 Ring Buffer 字符设备 |
| `/etc/agentrt/logger.conf` | `ConditionPathExists=` | 配置缺失则不启动，避免误用默认值落盘 |
| `/dev/airy_log_ring` | 运行时依赖 | Ring Buffer 字符设备节点，由 `airy_log_init()` 创建 |

Logger Daemon 在 `multi-user.target` 之前启动，确保用户态服务拉起时日志链路已通。`OOMScoreAdjust=-1000` 保证 OOM 时日志最后被杀，是故障诊断的最后防线。

---

## §2 配置文件：/etc/agentrt/logger.conf

### 2.1 配置 schema

Logger Daemon 配置采用 TOML 格式（对齐 A-UCS 统一配置管理体系，见 [11-unified-config.md](11-unified-config.md)），schema 如下：

```toml
# /etc/agentrt/logger.conf —— Logger Daemon 配置（A-UCS TOML 语义）

[level]
# 全局最低输出级别，低于此级别的记录丢弃
min = "INFO"            # FATAL | ERROR | WARN | INFO | DEBUG

[facility]
# 按 facility 过滤，未列出的 facility 全部输出
enabled = ["kernel", "agent", "ipc", "sched", "sec", "mem"]

[output]
# 输出路径，支持多 sink
path = "/var/log/agentrt/airy.log"
# Ring Buffer mmap 字符设备
ring_dev = "/dev/airy_log_ring"
# 单文件最大行数缓冲（写入聚合，降低 fsync 频次）
flush_lines = 64

[rotation]
# 按大小轮转：单文件达到 max_size 即轮转
max_size = "100MB"      # 支持 KB/MB/GB 单位
# 按时间轮转：达到 max_age 即轮转
max_age = "24h"          # 支持 m/h/d 单位
# 保留归档份数
keep = 16
# 轮转触发方式：size | time | both（任一条件满足即轮转）
policy = "both"

[compress]
# 归档压缩算法
algo = "zstd"
# zstd 压缩级别 1-19，3 为默认均衡档
level = 3
# 压缩线程数，0 = 自动（=CPU 核数）
threads = 0
```

### 2.2 字段语义

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `level.min` | enum | `INFO` | 5 级日志枚举门槛，权威定义见 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) |
| `facility.enabled` | string[] | 全部 | facility 白名单，对应 `LOG_*` 记录的 facility 字段 |
| `output.path` | path | `/var/log/agentrt/airy.log` | 落盘文件路径 |
| `output.ring_dev` | path | `/dev/airy_log_ring` | Ring Buffer 字符设备 |
| `rotation.max_size` | size | `100MB` | 单文件大小阈值 |
| `rotation.max_age` | duration | `24h` | 单文件时间阈值 |
| `rotation.keep` | int | `16` | 归档保留份数，超出则删除最旧 |
| `rotation.policy` | enum | `both` | 轮转触发策略 |
| `compress.algo` | enum | `zstd` | 压缩算法（当前仅支持 zstd） |
| `compress.level` | int | `3` | zstd 压缩级别 |

配置热加载通过 `SIGHUP`（`ExecReload`）触发，Logger Daemon 重新解析 `logger.conf` 并应用新参数，不中断 Ring Buffer 消费。

---

## §3 日志轮转：按大小 + 时间双维度

### 3.1 轮转策略

Logger Daemon 采用**按大小 + 按时间**双维度轮转（`policy = "both"`），任一条件满足即触发轮转：

| 维度 | 默认阈值 | 触发条件 | 设计目的 |
|------|---------|---------|---------|
| 大小 | 100 MB | 当前文件 `size >= max_size` | 防止单文件过大，便于检索与传输 |
| 时间 | 24 h | 当前文件 `age >= max_age` | 保证日志按天切片，便于按时间定位 |

### 3.2 轮转流程

```
活跃文件 airy.log
      │
      ├─ size >= 100MB 或 age >= 24h
      ▼
[1] 关闭当前 airy.log 文件句柄
[2] 重命名为 airy.log.<timestamp>（归档候选）
[3] 创建新 airy.log，恢复写入
[4] 异步对 airy.log.<timestamp> 执行 zstd 压缩 → airy.log.<timestamp>.zst
[5] 压缩完成后删除未压缩归档
[6] 检查归档总数，若 > keep 则删除最旧 .zst
```

轮转操作在独立工作线程执行，不阻塞 Ring Buffer 消费主线程。步骤 [2] 的重命名采用 `renameat2(RENAME_EXCHANGE)` 原子交换，确保轮转瞬间不丢日志——新文件立即承接写入，旧文件进入归档队列。

### 3.3 时间轮转的边界处理

时间轮转以**文件创建时刻**为基准计时，而非自然日边界。即一个文件若在 10:00 创建，则次日 10:00 触发轮转（默认 24h）。若需对齐自然日（每天 00:00 轮转），可设置 `max_age = "24h"` 且配合 `policy = "time"`，但 Logger Daemon 不强制自然日对齐，避免在日志高峰期（通常午夜附近）集中轮转造成 IO 尖峰。

### 3.4 keep 保留策略

`keep = 16` 表示最多保留 16 份归档（不含当前活跃文件）。归档按时间戳命名，超出 `keep` 时删除时间戳最旧的 `.zst` 文件。磁盘空间硬保护：Logger Daemon 启动时探测 `/var/log/agentrt` 所在分区剩余空间，若 < 5% 则提前触发轮转并降低 `keep` 至实际可用份数，避免磁盘写满导致系统不可用。

---

## §4 压缩归档：zstd 压缩旧日志

### 4.1 为什么选择 zstd

| 算法 | 压缩比 | 压缩速度 | 解压速度 | 选择理由 |
|------|--------|---------|---------|---------|
| gzip | 中 | 慢 | 中 | 压缩比与速度均不占优 |
| lz4 | 低 | 极快 | 极快 | 压缩比偏低，归档场景非热路径 |
| **zstd** | **高** | **快** | **极快** | **压缩比接近 gzip，速度数倍于 gzip，归档场景最优** |

日志归档是冷路径（轮转后的历史文件），对压缩速度敏感度低于压缩比，但 zstd 在两项指标上均优于 gzip，且 Linux 6.6 内核原生集成 zstd（用于 BTRFS/SquashFS 压缩），用户态 `libzstd` 已随发行版提供，无额外依赖。

### 4.2 压缩流程

Logger Daemon 通过 `libzstd` 流式压缩（`ZSTD_compressStream2`）处理归档候选文件，避免一次性加载大文件到内存：

```c
/* airy-loggerd 归档压缩伪代码 */
static int logger_archive_compress(const char *src, const char *dst)
{
    /* ZSTD 流式压缩，level=3，输入缓冲 256KB */
    ZSTD_CStream *cstream = ZSTD_createCStream();
    ZSTD_initCStream(cstream, conf.compress.level);   /* 默认 3 */

    int fd_in  = open(src, O_RDONLY);
    int fd_out = open(dst, O_WRONLY | O_CREAT | O_TRUNC, 0640);

    /* 流式读入-压缩-写出循环 */
    while ((n = read(fd_in, in_buf, BUF_SZ)) > 0) {
        ZSTD_inBuffer in = { in_buf, n, 0 };
        while (in.pos < in.size) {
            ZSTD_outBuffer out = { out_buf, BUF_SZ, 0 };
            ZSTD_compressStream(cstream, &out, &in);
            write(fd_out, out.dst, out.pos);
        }
    }
    /* flush 残余 */
    ZSTD_endStream(cstream, &out);
    ...
}
```

### 4.3 并行压缩与资源约束

`compress.threads = 0` 时自动按 CPU 核数启用多线程压缩（`ZSTDMT`）。但 Logger Daemon 的归档压缩受 `OOMScoreAdjust=-1000` 约束，不得抢占系统内存。压缩缓冲上限固定为 `threads × 512KB`，即使 `threads=16` 也仅占 8 MB，确保归档不挤压 Agent 运行时内存预算。

### 4.4 归档完整性

每份 `.zst` 归档写入完成后，Logger Daemon 计算原文件 SHA-256 并将哈希存入 `.zst.sha256` 旁路文件。归档检索工具 `airy-log-query` 解压时校验哈希，确保归档未被篡改或损坏。Panic 回退路径（见 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §Panic 回退）不依赖归档，仅依赖内核 Ring Buffer 残留与 printk 原生路径。

---

## §5 与 06-logger-daemon-design.md 的关系

### 5.1 文档分层

本文件（`12-logger-daemon-module.md`）是 **模块级设计**，与数据流设计 `06-logger-daemon-design.md`（Logger Daemon 数据流设计）构成 A-ULP 模块的两层文档体系：

| 维度 | 本文件（模块级） | 06-logger-daemon-design.md（数据流级） |
|------|----------------|--------------------------------------|
| 关注点 | systemd 编排、配置 schema、轮转策略、压缩归档 | Ring Buffer 消费数据流、eventfd 通知、mmap 读取顺序、背压处理 |
| 抽象层级 | 工程契约（运维可见） | 数据流（内核-用户态交互细节） |
| 变更频率 | 低（配置 schema 稳定） | 中（数据流随内核 Ring Buffer 演进） |
| 读者 | 运维工程师、发行版打包者 | 内核开发者、Logger Daemon 实现者 |

### 5.2 权威边界

- **本文件权威**：`airy-logger.service` unit 字段、`logger.conf` schema、轮转阈值默认值（100MB / 24h / keep=16）、zstd 压缩参数（level=3）
- **06-logger-daemon-design.md 权威**：Ring Buffer 消费指针推进算法、eventfd 通知节流策略、mmap 读取窗口、Panic 时 Logger Daemon 的降级行为
- **共同引用**：两者均引用 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 作为 128B 记录格式与 `LOG_*` 枚举的权威源，禁止任一文件重新定义记录格式

### 5.3 与 A-ULP 模块的整体关系

Logger Daemon 是 A-ULP 模块（见 [10-unify-design.md](../10-architecture/10-unify-design.md) §5）的用户态消费侧，与内核侧 Ring Buffer 生产者、printk 桥接（见 [13-printk-bridge.md](13-printk-bridge.md)）共同构成完整日志链路。三者关系：

```
内核生产者                    用户态消费者
┌──────────────┐   mmap    ┌────────────────┐
│ Ring Buffer  │ ────────► │ Logger Daemon  │ ── 落盘 + 轮转 + zstd 归档
│ (生产 128B)  │            │ (本文件)        │
└──────────────┘            └────────────────┘
       ▲
       │ printk_hook
┌──────────────┐
│ printk 桥接  │  ← Linux 6.6 原生 printk 桥接
│ (13 号文档)  │
└──────────────┘
```

---

## §6 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（A-ULP 模块定位）
- [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— 零拷贝 Ring Buffer 日志数据流（内核生产侧权威）
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— 128B 记录格式与 `LOG_*` 枚举 [SC] 契约
- [13-printk-bridge.md](13-printk-bridge.md) —— printk 桥接设计（日志链路上游）
- [11-unified-config.md](11-unified-config.md) —— A-UCS 统一配置管理体系（logger.conf TOML 语义）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Logger Daemon 模块设计 | v1.0 | 2026-07-17
