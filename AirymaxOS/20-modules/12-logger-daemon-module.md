Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Logger Daemon 模块设计

> **文档定位**：AirymaxOS（agentrt-linux）`logger_d` 模块级设计——systemd unit 编排、配置文件 schema、日志轮转策略、zstd 压缩归档、审计哈希链完整性保护，作为 A-ULP（统一日志与打印系统）用户态消费侧的模块工程契约\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5（A-ULP 模块）\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）+ §6.2.1 C-07（DMA 一致性内存修正）+ [ADR-012](../10-architecture/05-adrs.md#adr-012-微内核化改造技术路线确认基于-linux-改造--sel4-思想非从零开发)（微内核化改造技术路线）+ [ADR-014](../10-architecture/05-adrs.md#adr-014-微内核设计思想来源单一化仅-sel4不引入-zirconminix3)（微内核设计思想来源单一化：仅 seL4）\
> **极境内核标准**：本模块遵循"对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造"定位，capability 单一安全模型、消息传递 IPC、服务用户态化均源自 seL4 工程思想（ADR-014）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **`logger_d` 模块级设计** 的唯一权威源。`airy-logger.service` systemd unit 编排、`/etc/agentrt/logger_d.yaml` 配置 schema、按大小+时间的双维度轮转策略、zstd 压缩归档流程、与 `sec_d`/`audit_d` 的审计哈希链协作契约均以本文件为唯一权威定义。
>
> 本文件遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) A-ULP 模块设计与 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) 数据流设计。技术选型对齐 Unify Design 与 ADR-012/ADR-014：日志内存采用 `alloc_pages(GFP_KERNEL)` + mmap（不使用 DMA 一致性内存），调度采用 sched_tac（不使用 sched_ext），微内核化改造仅参考 seL4（不引入 Zircon/Minix3）。`LOG_*` 5 级日志枚举与 128B 记录格式的权威源为 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)；审计哈希链完整性保护（SHA3-256 + Ed25519 + TPM 2.0）的数据流细节权威源为 [06-logger-daemon-design.md](../40-dataflows/06-logger-daemon-design.md) §5.5。

---

## §0 极境内核标准与 ADR 对齐

### 0.1 极境内核定位

`logger_d` 是 agentrt-linux（AirymaxOS）极境内核的用户态日志消费守护进程。极境内核定位为"对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造的内核"：

- **基座**：Linux 6.6 内核（ADR-012）
- **思想来源**：seL4 微内核工程思想（ADR-014）——Liedtke 极简原则、capability 单一安全模型、消息传递 IPC、服务用户态化
- **服务用户态化**：`logger_d` 即 seL4 "服务用户态化"思想的落地——日志格式化、过滤、落盘、轮转、压缩等耗时操作均下沉到用户态守护进程，内核 fastpath 仅保留 `reserve → memcpy 128B raw binary → commit` 三步（~50-100ns），严禁任何格式化操作

### 0.2 v1.0.1 Capability Folding 集成

自 v1.0.1 起，`logger_d` 集成 Capability Folding 单平面架构（详见 §7）：

- `agent_caps[1024]` 静态数组（16KB，sec_d 唯一写者）为 `logger_d` 提供 O(1) Badge 校验入口
- fastpath C-S9 内联校验（~10ns）拦截伪造/过期 Badge 的日志写入尝试
- Badge 64-bit 布局 `Epoch<<48 | RandomTag<<16 | Perms` 在 `logger_d` 侧通过 `agent_caps[src_task]` 反查验证
- 校验失败时记录 `AIRY_ECAP_FROZEN = -82`、`AIRY_ESEC_D_THROTTLED = -83` 错误码至 Ring Buffer

---

## §1 systemd unit 设计：airy-logger.service

### 1.1 模块定位

`logger_d` 是 A-ULP 模块的用户态消费侧守护进程，从内核 Ring Buffer（见 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)）mmap 映射区读取 128B 日志记录，按 `/etc/agentrt/logger_d.yaml` 配置写入持久化文件，并执行轮转与压缩归档。`logger_d` **不参与日志生产**，仅承担消费、落盘、轮转、归档、审计哈希链追加五项职责，与内核 Ring Buffer 生产者彻底解耦。

### 1.2 unit 文件

`logger_d` 通过 systemd unit `airy-logger.service` 编排，作为系统早期服务拉起，确保 Panic 回退路径之前已就绪：

```ini
# /usr/lib/systemd/system/airy-logger.service
[Unit]
Description=Airymax logger_d (A-ULP consumer)
Documentation=file:///usr/share/doc/agentrt/20-modules/12-logger-daemon-module.md
After=agentrt-init.service
Before=multi-user.target
ConditionPathExists=/etc/agentrt/logger_d.yaml
DefaultDependencies=no

[Service]
Type=simple
ExecStart=/usr/sbin/airy-logger-d --config /etc/agentrt/logger_d.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=1s
# OOM 不可杀，日志是故障诊断最后防线
OOMScoreAdjust=-1000
# 调度对齐 sched_tac：logger_d 用 SCHED_FIFO 高优先级
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
| `agentrt-init.service` | `After=` | 等待 capability 初始化守护进程就绪，`logger_d` 需 capability 访问 Ring Buffer 字符设备 |
| `/etc/agentrt/logger_d.yaml` | `ConditionPathExists=` | 配置缺失则不启动，避免误用默认值落盘 |
| `/dev/airy_log` | 运行时依赖 | Ring Buffer 字符设备节点，由 `airy_log_init()` 创建（命名空间统一为 agentrt） |

`logger_d` 在 `multi-user.target` 之前启动，确保用户态服务拉起时日志链路已通。`OOMScoreAdjust=-1000` 保证 OOM 时日志最后被杀，是故障诊断的最后防线。

---

## §2 配置文件：/etc/agentrt/logger_d.yaml

### 2.1 配置 schema

`logger_d` 配置采用 YAML 格式（对齐 A-UCS 统一配置管理体系，见 [11-unified-config.md](11-unified-config.md)；命名空间统一为 agentrt），schema 如下：

```yaml
# /etc/agentrt/logger_d.yaml —— logger_d 配置（A-UCS YAML 语义）

level:
  # 全局最低输出级别，低于此级别的记录丢弃
  min: INFO            # FATAL | ERROR | WARN | INFO | DEBUG

facility:
  # 按 facility 过滤，未列出的 facility 全部输出
  enabled: [kernel, agent, ipc, sched, sec, mem]

output:
  # 输出路径，支持多 sink
  path: /var/log/agentrt/airy.log
  # Ring Buffer mmap 字符设备（agentrt 命名空间）
  ring_dev: /dev/airy_log
  # 单文件最大行数缓冲（写入聚合，降低 fsync 频次）
  flush_lines: 64

rotation:
  # 按大小轮转：单文件达到 max_size 即轮转
  max_size: 100MB      # 支持 KB/MB/GB 单位
  # 按时间轮转：达到 max_age 即轮转
  max_age: 24h          # 支持 m/h/d 单位
  # 保留归档份数
  keep: 16
  # 轮转触发方式：size | time | both（任一条件满足即轮转）
  policy: both

compress:
  # 归档压缩算法
  algo: zstd
  # zstd 压缩级别 1-19，3 为默认均衡档
  level: 3
  # 压缩线程数，0 = 自动（=CPU 核数）
  threads: 0

audit_chain:
  # 审计哈希链完整性保护（详见 §6，权威源 40-dataflows/06-logger-daemon-design.md §5.5）
  enabled: true
  # 哈希算法：SHA3-256
  hash_algo: sha3_256
  # 签名算法：Ed25519
  sign_algo: ed25519
  # 链尾签名触发：每 N 条记录签名一次
  sign_interval: 1000
  # 链尾签名触发：每 T 秒签名一次（先到者触发）
  sign_timeout: 60
  # TPM 2.0 密钥密封（v1.0.1+ 启用，私钥永不出 TPM）
  tpm_seal: true
```

### 2.2 字段语义

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `level.min` | enum | `INFO` | 5 级日志枚举门槛，权威定义见 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) |
| `facility.enabled` | string[] | 全部 | facility 白名单，对应 `LOG_*` 记录的 facility 字段（取值 `AIRY_FAC_*`） |
| `output.path` | path | `/var/log/agentrt/airy.log` | 落盘文件路径（agentrt 命名空间） |
| `output.ring_dev` | path | `/dev/airy_log` | Ring Buffer 字符设备（agentrt 命名空间） |
| `rotation.max_size` | size | `100MB` | 单文件大小阈值 |
| `rotation.max_age` | duration | `24h` | 单文件时间阈值 |
| `rotation.keep` | int | `16` | 归档保留份数，超出则删除最旧 |
| `rotation.policy` | enum | `both` | 轮转触发策略 |
| `compress.algo` | enum | `zstd` | 压缩算法（当前仅支持 zstd） |
| `compress.level` | int | `3` | zstd 压缩级别 |
| `audit_chain.enabled` | bool | `true` | 审计哈希链完整性保护开关 |
| `audit_chain.sign_interval` | int | `1000` | 链尾签名记录数阈值 |
| `audit_chain.tpm_seal` | bool | `true` | TPM 2.0 密钥密封开关 |

配置热加载通过 `SIGHUP`（`ExecReload`）触发，`logger_d` 重新解析 `logger_d.yaml` 并应用新参数，不中断 Ring Buffer 消费。

---

## §3 日志轮转：按大小 + 时间双维度

### 3.1 轮转策略

`logger_d` 采用**按大小 + 按时间**双维度轮转（`policy = "both"`），任一条件满足即触发轮转：

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

时间轮转以**文件创建时刻**为基准计时，而非自然日边界。即一个文件若在 10:00 创建，则次日 10:00 触发轮转（默认 24h）。若需对齐自然日（每天 00:00 轮转），可设置 `max_age = "24h"` 且配合 `policy = "time"`，但 `logger_d` 不强制自然日对齐，避免在日志高峰期（通常午夜附近）集中轮转造成 IO 尖峰。

### 3.4 keep 保留策略

`keep = 16` 表示最多保留 16 份归档（不含当前活跃文件）。归档按时间戳命名，超出 `keep` 时删除时间戳最旧的 `.zst` 文件。磁盘空间硬保护：`logger_d` 启动时探测 `/var/log/agentrt` 所在分区剩余空间，若 < 5% 则提前触发轮转并降低 `keep` 至实际可用份数，避免磁盘写满导致系统不可用。

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

`logger_d` 通过 `libzstd` 流式压缩（`ZSTD_compressStream2`）处理归档候选文件，避免一次性加载大文件到内存：

```c
/* airy-logger-d 归档压缩伪代码 */
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

`compress.threads = 0` 时自动按 CPU 核数启用多线程压缩（`ZSTDMT`）。但 `logger_d` 的归档压缩受 `OOMScoreAdjust=-1000` 约束，不得抢占系统内存。压缩缓冲上限固定为 `threads × 512KB`，即使 `threads=16` 也仅占 8 MB，确保归档不挤压 Agent 运行时内存预算。

### 4.4 归档完整性

每份 `.zst` 归档写入完成后，`logger_d` 计算原文件 SHA-256 并将哈希存入 `.zst.sha256` 旁路文件。归档检索工具 `airy-log-query` 解压时校验哈希，确保归档未被篡改或损坏。Panic 回退路径（见 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §Panic 回退）不依赖归档，仅依赖内核 Ring Buffer 残留与 printk 原生路径。

---

## §5 与 06-logger-daemon-design.md 的关系

### 5.1 文档分层

本文件（`12-logger-daemon-module.md`）是 **模块级设计**，与数据流设计 `06-logger-daemon-design.md`（`logger_d` 数据流设计）构成 A-ULP 模块的两层文档体系：

| 维度 | 本文件（模块级） | 06-logger-daemon-design.md（数据流级） |
|------|----------------|--------------------------------------|
| 关注点 | systemd 编排、配置 schema、轮转策略、压缩归档、审计哈希链协作契约 | Ring Buffer 消费数据流、eventfd 通知、mmap 读取顺序、背压处理、哈希链 C 实现 |
| 抽象层级 | 工程契约（运维可见） | 数据流（内核-用户态交互细节） |
| 变更频率 | 低（配置 schema 稳定） | 中（数据流随内核 Ring Buffer 演进） |
| 读者 | 运维工程师、发行版打包者 | 内核开发者、`logger_d` 实现者 |

### 5.2 权威边界

- **本文件权威**：`airy-logger.service` unit 字段、`logger_d.yaml` schema、轮转阈值默认值（100MB / 24h / keep=16）、zstd 压缩参数（level=3）、`logger_d` 与 `sec_d`/`audit_d` 的协作契约
- **06-logger-daemon-design.md 权威**：Ring Buffer 消费指针推进算法、eventfd 通知节流策略、mmap 读取窗口、Panic 时 `logger_d` 的降级行为、审计哈希链 C 数据结构与 `airy_audit_chain_*` API（§5.5）
- **共同引用**：两者均引用 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 作为 128B 记录格式与 `LOG_*` 枚举的权威源，禁止任一文件重新定义记录格式

### 5.3 与 A-ULP 模块的整体关系

`logger_d` 是 A-ULP 模块（见 [10-unify-design.md](../10-architecture/10-unify-design.md) §5）的用户态消费侧，与内核侧 Ring Buffer 生产者、printk 桥接（见 [13-printk-bridge.md](13-printk-bridge.md)）共同构成完整日志链路。三者关系：

```
内核生产者                    用户态消费者
┌──────────────┐   mmap    ┌────────────────┐
│ Ring Buffer  │ ────────► │  logger_d      │ ── 落盘 + 轮转 + zstd 归档
│ (生产 128B)  │            │ (本文件)        │ ── 审计哈希链追加（§6）
└──────────────┘            └────────────────┘
       ▲
       │ printk_hook
┌──────────────┐
│ printk 桥接  │  ← Linux 6.6 原生 printk 桥接
│ (13 号文档)  │
└──────────────┘
```

---

## §6 审计哈希链完整性保护

### 6.1 设计目标

A-ULP 审计日志是合规与安全分析的关键证据，但落盘后的日志文件可能被攻击者篡改（如本地 root 攻击、磁盘离线篡改）。本节定义 SHA3-256 哈希链 + Ed25519 数字签名 + TPM 2.0 度量机制，确保审计日志的完整性与可追溯性。

> **数据流权威源**：本节为模块级契约，哈希链 C 数据结构、`airy_audit_chain_*` API、密钥管理细节的权威源为 [06-logger-daemon-design.md](../40-dataflows/06-logger-daemon-design.md) §5.5。本文件不重新定义哈希链算法实现，仅定义 `logger_d` 与 `sec_d`/`audit_d` 的协作契约与配置项。

### 6.2 三道完整性保护

| 层 | 机制 | 触发频率 | 责任方 |
|----|------|---------|--------|
| 1 | **SHA3-256 哈希链** | 每条审计记录 | `logger_d` 在落盘前追加 `prev_hash` |
| 2 | **Ed25519 数字签名** | 每 N=1000 条或 T=60s（先到者触发） | `sec_d` 持有私钥并签名链尾 |
| 3 | **TPM 2.0 度量** | 启动时度量日志哈希链根 | `sec_d` 在启动时将 `genesis_hash` 度量至 TPM PCG |

哈希链结构：

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Record[0]      │    │  Record[1]      │    │  Record[2]      │
│  genesis_hash   │───▶│  prev_hash =    │───▶│  prev_hash =    │───▶ ...
│                 │    │   SHA3-256(R[0])│    │   SHA3-256(R[1])│
│  record_data    │    │  record_data    │    │  record_data    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                                           │
        │           每 N=1000 条或 T=60s            │
        ▼                                           ▼
┌─────────────────────────────────────────────────────────────┐
│  chain_signature = Ed25519_sign(sec_d_private_key,          │
│                                  last_record_hash)          │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 与 sec_d 的协作（Badge 校验日志 + 链尾签名）

`logger_d` 与 `sec_d` 在审计哈希链上双向协作：

| 协作方向 | 触发场景 | 接口 | 数据流 |
|---------|---------|------|--------|
| `sec_d` → `logger_d` | fastpath C-S9 Badge 校验失败（伪造/过期/冻结） | Ring Buffer 写入 `AIRY_FAC_SECURITY` facility 记录 | `logger_d` 消费时识别为审计记录，触发哈希链追加 |
| `logger_d` → `sec_d` | 链尾达到 N=1000 条或 T=60s 阈值 | `airy_audit_chain_sign()` IPC 请求 | `sec_d` 使用 Ed25519 私钥签名链尾哈希，返回 `chain_signature` |
| `sec_d` → `logger_d` | `sec_d` 收到 SIGTERM | 即时签名链尾 | `logger_d` 在退出前完成最后一段链尾签名 |

### 6.4 与 audit_d 的协作（审计读取方校验）

`audit_d` 是审计日志的独立读取方（与 `logger_d` 职责分离，符合 seL4 机制与策略分离原则）：

| 协作方向 | 触发场景 | 接口 | 数据流 |
|---------|---------|------|--------|
| `audit_d` → `logger_d` 落盘文件 | 合规审计、安全事件调查 | 读取 `/var/log/agentrt/audit_chain.log` + `audit_chain.sig` | `audit_d` 调用 `airy_audit_chain_verify()` 重新计算哈希链 |
| `audit_d` → `sec_d` | 验证签名 | 加载 `/var/lib/airy/audit.pub` 公钥 | `Ed25519_verify()` 校验链尾签名 |

### 6.5 篡改检测与故障码

任何篡改（修改记录、删除记录、插入记录、重排顺序）都会导致哈希链断裂，触发故障码：

| 篡改类型 | 检测方式 | 故障码 |
|---------|---------|--------|
| 修改记录 | `prev_hash` 不匹配（链断裂在篡改处） | `AIRY_FAULT_AUDIT_TAMPER = 0x100B` |
| 删除记录 | 后续记录的 `prev_hash` 链断 | `AIRY_FAULT_AUDIT_TAMPER = 0x100B` |
| 插入记录 | 插入处 `prev_hash` 链断 | `AIRY_FAULT_AUDIT_TAMPER = 0x100B` |
| 重排顺序 | `prev_hash` 链断 | `AIRY_FAULT_AUDIT_TAMPER = 0x100B` |
| 签名失败 | `Ed25519_verify()` 返回 false | `AIRY_FAULT_AUDIT_TAMPER = 0x100B` |

> **OS-SEC-130**（R4）：所有 `AIRY_FAC_SECURITY` facility 的审计日志记录必须经过哈希链完整性保护；审计日志读取方必须重新计算哈希链验证完整性，链断裂即视为日志被篡改，触发 `AIRY_FAULT_AUDIT_TAMPER = 0x100B`（已在 [08-sc-error-contract.md §3.1](../30-interfaces/08-sc-error-contract.md) 注册）。

### 6.6 落盘记录结构（OLK 6.6 工程规范）

落盘的审计记录结构对齐 OLK 6.6 工程规范，使用 `__aligned(64)` 而非传统 packed 属性：

```c
/* services/daemons/logger_d/audit_chain.h —— 审计哈希链落盘记录 */
struct airy_audit_record {
    struct airy_log_record log;             /* 128B：A-ULP 标准 128B 日志记录 */
    uint8_t prev_hash[AIRY_SHA3_256_LEN];   /* 32B：前一条记录的 SHA3-256 哈希 */
} __aligned(64);
/* sizeof(struct airy_audit_record) == 192（128 + 32 + 对齐填充） */

/* 链尾签名记录（独立文件 audit_chain.sig） */
struct airy_audit_chain_signature {
    uint64_t sequence;                            /* 签名序号（从 0 递增） */
    uint64_t record_count;                        /* 本次签名覆盖的记录数 */
    uint8_t  last_record_hash[AIRY_SHA3_256_LEN]; /* 链尾记录的哈希 */
    uint8_t  signature[AIRY_ED25519_SIG_LEN];     /* Ed25519 签名 */
    int64_t  timestamp_ns;                         /* 签名时间戳（CLOCK_MONOTONIC） */
} __aligned(64);
```

> **OLK 6.6 工程规范**：所有 A-ULP 落盘结构使用 `__aligned(64)`（cache line 对齐），不使用 packed 属性（会破坏 atomic 访问语义）或 `__attribute__((aligned(64)))`（OLK 6.6 简写为 `__aligned(64)`）。

---

## §7 v1.0.1 Capability Folding 集成

### 7.1 单平面架构落地

自 v1.0.1 起，`logger_d` 集成 Capability Folding 单平面架构（[SC] syscalls.h 4 核心 + 20 预留 = 24 槽位）。`logger_d` 不直接持有 capability，但其消费的每条 Ring Buffer 记录均可能携带 Badge 信息，需通过 `agent_caps[1024]` 静态数组反查校验。

### 7.2 agent_caps[1024] 在 logger_d 中的引用

`agent_caps[1024]` 静态数组（16KB，sec_d 唯一写者）在 `logger_d` 侧只读访问：

| 字段 | 用途 | logger_d 访问方式 |
|------|------|------------------|
| `agent_caps[src_task].epoch` | 校验 Badge Epoch 是否匹配全局 Epoch | fastpath C-S9 内联校验（~10ns） |
| `agent_caps[src_task].randtag` | 校验 Badge RandomTag 是否匹配 | fastpath C-S9 内联校验 |
| `agent_caps[src_task].perms` | 校验 Badge 权限位是否满足日志写入所需权限 | fastpath C-S9 内联校验 |
| `agent_caps[src_task].frozen` | 检测 Ring 是否冻结 | C-S0 检查（O(1)） |

### 7.3 Badge 64-bit 布局校验

Badge 64-bit 布局 `Epoch<<48 | RandomTag<<16 | Perms` 在 `logger_d` 侧通过 fastpath C-S9 内联校验：

```c
/* logger_d 侧 Badge 校验日志记录（伪代码） */
static void logger_validate_badge(struct airy_log_record *rec, u64 badge)
{
    u16 badge_epoch    = (badge >> 48) & 0xFFFF;
    u32 badge_randtag  = (badge >> 16) & 0xFFFFFFFF;
    u16 badge_perms    = badge & 0xFFFF;
    u32 src_task       = rec->caller_id;

    /* fastpath C-S9 内联校验（~10ns） */
    if (unlikely(badge_epoch != agent_caps[src_task].epoch)) {
        /* Epoch 不匹配：Badge 已被 O(1) 撤销（atomic_inc(&airy_cap_global_epoch)） */
        rec->level = LOG_ERROR;
        rec->facility = AIRY_FAC_SECURITY;
        return;  /* 记录 Badge 过期事件，但不丢弃日志 */
    }
    if (unlikely(badge_randtag != agent_caps[src_task].randtag)) {
        /* RandomTag 不匹配：Badge 伪造尝试 */
        rec->level = LOG_FATAL;
        rec->facility = AIRY_FAC_SECURITY;
        /* 触发 AIRY_FAULT_CAP_FORGED，由 sec_d 审计 */
        return;
    }
    /* 校验通过：记录正常日志 */
}
```

### 7.4 错误码记录

Capability Folding 校验失败时，`logger_d` 将错误码记录至 Ring Buffer：

| 错误码 | 值 | 触发场景 | logger_d 处理 |
|--------|-----|---------|--------------|
| `AIRY_ECAP_FROZEN` | -82 | `agent_caps[src_task].frozen == true`（C-S0 检查） | 记录 `LOG_WARN` 级别，标记 Ring 冻结事件 |
| `AIRY_ESEC_D_THROTTLED` | -83 | `sec_d` 限流器拒绝 Badge 编译请求 | 记录 `LOG_WARN` 级别，标记 sec_d 限流事件 |
| `AIRY_ECAP_FORGED` | -80 | Badge RandomTag 不匹配（伪造尝试） | 记录 `LOG_FATAL` 级别，触发 `AIRY_FAULT_CAP_FORGED` |

### 7.5 O(1) 撤销机制

`atomic_inc(&airy_cap_global_epoch)` 触发全局 Epoch 跃迁后，所有旧 Badge 在 `logger_d` 侧 fastpath C-S9 校验时立即失效（Epoch 不匹配），实现 O(1) 撤销。`logger_d` 在 Epoch 跃迁期间记录的日志会标记 `AIRY_ECAP_FROZEN`，但**不丢弃日志**——日志是故障诊断最后防线，即使 Badge 失效也要保留记录用于事后审计。

---

## §8 IRON-9 v3 [DSL] 降级生存层

### 8.1 [DSL] 在 logger_d 中的体现

IRON-9 v3 四层模型（[SC] + [SS] + [IND] + [DSL]）中，`logger_d` 涉及 [SC] log_types.h 与 [DSL] 降级块：

| 层 | 头文件/资源 | logger_d 使用方式 |
|----|----------|-----------------|
| [SC] | `log_types.h` | 128B 记录格式、`LOG_*` 枚举、`AIRY_FAC_*` facility |
| [SC] | `error.h` | `AIRY_ECAP_FROZEN`、`AIRY_ESEC_D_THROTTLED`、`AIRY_FAULT_AUDIT_TAMPER` |
| [SS] | 配置语义 | `logger_d.yaml` 与 sysctl 语义同源 |
| [IND] | systemd unit、zstd 压缩、轮转策略 | `logger_d` 实现细节 |
| [DSL] | `#ifdef AIRY_SC_FALLBACK` 降级块 | log_types.h 损坏时的最小可运行子集 |

### 8.2 [SC] log_types.h 损坏时的降级路径

当 [SC] log_types.h 损坏（如磁盘错误导致头文件字节翻转）时，`#ifdef AIRY_SC_FALLBACK` 降级块生效，`logger_d` 降级为最小可运行子集：

| 维度 | 正常模式 | [DSL] 降级模式 |
|------|---------|--------------|
| 日志级别 | 5 级（LOG_DEBUG~LOG_FATAL） | 仅 LOG_FATAL + LOG_ERROR 两级 |
| 日志通路 | Ring Buffer + `logger_d` 消费 + 落盘 | printk 原生路径（绕过 Ring Buffer） |
| facility 过滤 | `AIRY_FAC_*` 完整枚举 | 仅 `AIRY_FAC_KERNEL` |
| 审计哈希链 | SHA3-256 + Ed25519 + TPM 2.0 | 关闭（降级模式下不可靠） |
| Capability Folding | fastpath C-S9 Badge 校验 | 跳过（仅 POSIX capability） |
| 配置加载 | `/etc/agentrt/logger_d.yaml` | 内置默认值（`#warning` 告警） |

### 8.3 降级块定义

```c
/* kernel/include/uapi/linux/airymax/log_types.h —— [DSL] 降级块 */
#ifdef AIRY_SC_FALLBACK
#warning "AIRY_SC_FALLBACK: log_types.h degraded, only LOG_FATAL+LOG_ERROR available"

/* [DSL] 最小可运行子集：仅保留 LOG_FATAL + LOG_ERROR */
#define LOG_FATAL   4
#define LOG_ERROR   3
/* LOG_DEBUG / LOG_INFO / LOG_WARN 预处理为空操作，直接走 printk 原生路径 */
#define LOG_DEBUG   LOG_ERROR
#define LOG_INFO    LOG_ERROR
#define LOG_WARN    LOG_ERROR

/* [DSL] 仅保留 AIRY_FAC_KERNEL */
#define AIRY_FAC_KERNEL  0

/* [DSL] 128B 记录降级为 printk 原生日志（绕过 Ring Buffer） */
#define airy_log_record  printk_degraded_entry

#endif /* AIRY_SC_FALLBACK */
```

### 8.4 降级模式下的 logger_d 行为

`logger_d` 在 [DSL] 降级模式下：

1. **不再消费 Ring Buffer**——Ring Buffer 可能已损坏，`logger_d` 退出 mmap 读取
2. **降级为 printk 转发器**——将自身产生的告警通过 `syslog()` 转发至 printk 原生路径
3. **审计哈希链关闭**——降级模式下签名密钥不可信，哈希链计算停止
4. **systemd 不重启 `logger_d`**——降级模式下 `logger_d` 主动退出，由 printk 原生路径接管

详细设计见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §4.1.4。

---

## §9 12 daemon 协作关系

`logger_d` 是 agentrt-linux 12 daemon 之一，与其他 11 daemon 的协作关系如下：

| 协作方 | 协作方向 | 接口 | 协作内容 |
|--------|---------|------|---------|
| `sec_d` | 双向 | Ring Buffer + IPC | `sec_d` 写入 Badge 校验日志（`AIRY_FAC_SECURITY` facility）；`logger_d` 请求链尾签名（Ed25519） |
| `audit_d` | 单向（读取） | 落盘文件 | `audit_d` 读取 `/var/log/agentrt/audit_chain.log` + `audit_chain.sig`，调用 `airy_audit_chain_verify()` 校验完整性 |
| `macro_d` | 单向（被监管） | 心跳上报 | `logger_d` 上报心跳至 `macro_d`，崩溃后由 `macro_d` 通过 systemd 重启 |
| `config_d` | 单向（被管理） | SIGHUP 热重载 | `config_d` 修改 `/etc/agentrt/logger_d.yaml` 后通过 SIGHUP 触发 `logger_d` 热重载 |
| `cogn_d` | 单向（生产者） | Ring Buffer | `cogn_d` 写入认知循环日志（`AIRY_FAC_COGNITION` facility），`logger_d` 消费 |
| `mem_d` | 单向（生产者） | Ring Buffer | `mem_d` 写入记忆卷载日志（`AIRY_FAC_MEMORY` facility），`logger_d` 消费 |
| `gateway_d` | 单向（生产者） | Ring Buffer | `gateway_d` 写入跨节点 IPC 日志，`logger_d` 消费 |
| `sched_d` | 单向（生产者） | Ring Buffer | `sched_d` 写入调度策略日志（`AIRY_FAC_SCHED` facility），`logger_d` 消费 |
| `dev_d` | 单向（生产者） | Ring Buffer | `dev_d` 写入设备驱动用户态化日志，`logger_d` 消费 |
| `net_d` | 单向（生产者） | Ring Buffer | `net_d` 写入网络栈用户态化日志，`logger_d` 消费 |
| `vfs_d` | 单向（生产者） | Ring Buffer | `vfs_d` 写入 VFS 用户态化日志，`logger_d` 消费 |

12 daemon 完整名单（与 [10-user-supervisor-daemon.md](10-user-supervisor-daemon.md) §1.3 对齐）：

```
sec_d（capability 编译/撤销）          cogn_d（认知循环调度）
mem_d（记忆卷载管理）                  gateway_d（跨节点 IPC）
logger_d（统一日志，本文件）            macro_d（宏观监管）
audit_d（审计哈希链校验）              sched_d（sched_tac 策略守护）
dev_d（设备驱动用户态化）              net_d（网络栈用户态化）
vfs_d（VFS 用户态化）                  config_d（统一配置管理）
```

---

## §10 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（A-ULP 模块定位）
- [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— 零拷贝 Ring Buffer 日志数据流（内核生产侧权威）
- [06-logger-daemon-design.md](../40-dataflows/06-logger-daemon-design.md) —— `logger_d` 数据流设计（§5.5 审计哈希链 C 实现权威源）
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— 128B 记录格式与 `LOG_*`/`AIRY_FAC_*` 枚举 [SC] 契约
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— `AIRY_ECAP_FROZEN`/`AIRY_ESEC_D_THROTTLED`/`AIRY_FAULT_AUDIT_TAMPER` 错误码注册
- [13-printk-bridge.md](13-printk-bridge.md) —— printk 桥接设计（日志链路上游）
- [11-unified-config.md](11-unified-config.md) —— A-UCS 统一配置管理体系（`logger_d.yaml` YAML 语义）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级生存层（log_types.h 降级块）
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 四层模型
- [05-adrs.md](../10-architecture/05-adrs.md) —— ADR-012（微内核化改造技术路线）+ ADR-014（微内核设计思想来源单一化：仅 seL4）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | `logger_d` 模块设计 | v1.0.1 | 2026-07-21
