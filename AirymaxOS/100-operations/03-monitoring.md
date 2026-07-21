Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 监控体系（Monitoring System）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · 监控体系设计文档，定义系统全局可观测性架构、监控指标采集、聚合与暴露接口\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

AirymaxOS 作为智能体操作系统，需要在 Linux 6.6 内核基础上为 12 个系统 daemon 与任意数量的 Agent 提供统一、低开销、可扩展的监控能力。监控体系设计目标如下：

1. **全栈可观测**：覆盖内核态（IRON-9 v3 四层模型 [SC]/[SS]/[IND]/[DSL]）、用户态（12 daemon）、Agent 业务态（Token 消耗、记忆层级、IPC 流量）三个维度。
2. **低开销采集**：生产环境下监控开销不超过单 CPU 1.5%，主路径零拷贝，复用 IORING_OP_URING_CMD 异步通道。
3. **统一接入**：所有监控数据通过 A-ULS（Unified Supervision）模块 macro_d 汇聚，所有日志事件通过 A-ULP（Unified Log Processing System）模块 logger_d 采集，避免多通道重复上报。
4. **告警联动**：监控指标超阈值自动触发告警体系（详见 [04-alerting.md](04-alerting.md)），并对接事件响应流程（详见 [05-incident-response.md](05-incident-response.md)）。
5. **可扩展**：新增 Agent 或 daemon 时无需修改采集框架，仅通过 sysfs 接口注册即可纳入监控。

### 1.2 监控架构总览

AirymaxOS 监控体系采用「双通道 + 一聚合」架构：

```
┌────────────────────────────────────────────────────────────────┐
│                    监控数据来源（Sources）                       │
├────────────────────────────────────────────────────────────────┤
│  Kernel (ftrace/tracepoint)  │  sysfs  │  procfs  │  daemon 内埋点  │
└──────────────┬───────────────────────────────────────────────────┘
               │
       ┌───────┴────────┐
       │  数据采集通道   │
       │  (双通道并行)   │
       └───────┬────────┘
               │
   ┌───────────┴────────────┐
   │                        │
   ▼                        ▼
┌─────────────┐      ┌──────────────┐
│ macro_d│      │logger_d │
│  (A-ULS 通道) │      │ (A-ULP 通道)  │
│ 指标实时监管 │      │ 日志事件采集 │
└──────┬──────┘      └──────┬───────┘
       │                    │
       │   ┌────────────────┘
       ▼   ▼
┌─────────────────┐
│  audit_d daemon │
│  数据聚合层      │
│  (Aggregator)   │
└────────┬────────┘
         │
         ▼
┌────────────────────────────┐
│  /sys/airy/monitor/        │
│  systemd journal           │
│  RPC 查询接口              │
└────────────────────────────┘
```

**双通道分工**：

| 通道 | 模块 | 数据类型 | 触发模式 | 延迟 |
|------|------|----------|----------|------|
| 指标通道 | A-ULS / macro_d | 数值型指标（CPU、内存、Token 等） | 周期轮询 + 异常推送 | ≤100ms |
| 日志通道 | A-ULP / logger_d | 文本事件（崩溃、警告、错误） | 事件驱动 | ≤10ms |

**一聚合**：audit_d 作为唯一聚合层，对接收到的原始指标和日志进行去重、降采样、关联分析，输出统一的监控视图。

### 1.3 与技术选型的契合

- **sched_tac（不使用 sched_ext）**：调度延迟监控通过 `sched/sched_switch` tracepoint + `/proc/sched_debug` 直接采集，不依赖 BPF 程序。
- **IORING_OP_URING_CMD**：监控数据上报复用 io_uring 命令通道，避免额外系统调用开销。
- **纯 C LSM**：安全相关监控（进程越权、能力变更）通过 LSM hook 钩子直采，无需 BPF LSM。
- **IRON-9 v3 四层模型**：每层暴露独立的监控接口，详见第 4 节。

---

## 2. 监控对象

### 2.1 12 Daemon 健康状态

| Daemon | 职责 | 关键监控项 | 采集来源 |
|--------|------|-----------|----------|
| macro_d | 统一监管 | 监管的子 daemon 数、重启次数、心跳超时数 | A-ULS 内部计数器 |
| logger_d | 统一日志 | Ring Buffer 占用率、丢弃日志数、写入 QPS | A-ULP 内部计数器 |
| config_d | 统一配置 | 配置加载耗时、热更新次数、校验失败数 | A-UCS 内部计数器 |
| gateway_d | 网关 | 连接数、请求 QPS、错误率、握手延迟 | daemon 埋点 |
| sched_d | 调度 | 调度延迟分位、SCHED_DEADLINE miss 次数 | ftrace tracepoint |
| vfs_d | 文件系统 | IOPS、fsync 延迟、缓存命中率 | /proc/fs/airyvfs/ |
| net_d | 网络 | 收发字节、重传率、连接跟踪表占用 | /proc/net/airy/ |
| mem_d | 内存 | L1-L4 记忆占用、defrag 频次、淘汰量 | /sys/airy/memory/ |
| cogn_d | 认知 | Token 消耗速率、推理队列长度、上下文切换数 | daemon 埋点 |
| sec_d | 安全 | LSM hook 命中数、拒绝事件数、密钥访问次数 | LSM hook 计数器 |
| audit_d | 审计聚合 | 自身入队/出队速率、聚合耗时、存储占用 | audit_d 自监控 |
| dev_d | 设备管理 | 已注册设备数、热插拔事件数、驱动加载耗时 | /sys/airy/devices/ |

每个 daemon 的健康状态以三元组表示：`(liveness, readiness, health_score)`：

- `liveness`：进程是否存活（0/1），由 systemd WatchdogSec 与 macro_d 心跳共同判定。
- `readiness`：是否可对外提供服务（0/1），由 daemon 自行上报，例如 gateway_d 需握手链路就绪。
- `health_score`：综合健康分（0-100），由 CPU 占用、内存占用、错误率加权计算。

### 2.2 Agent 运行状态

Agent 是 AirymaxOS 的一等公民，监控粒度细到单 Agent 级别：

| 监控维度 | 采集字段 | 采集频率 | 用途 |
|----------|----------|----------|------|
| 生命周期 | state（created/running/paused/stopped/migrated）、uptime | 1s | 生命周期运维 |
| 资源占用 | CPU%、RSS、IO bytes | 100ms | 资源调优 |
| 调度属性 | sched_policy、runtime/deadline/period | 1s | sched_d 调优 |
| Token 消耗 | input_tokens、output_tokens、tokens/sec | 实时 | cogn_d 预算管理 |
| 记忆状态 | L1-L4 占用、最近淘汰项 | 1s | mem_d 运维 |
| IPC 流量 | tx_msgs、rx_msgs、avg_latency | 100ms | A-IPC 链路诊断 |
| 错误统计 | error_count、last_error_code | 事件驱动 | A-UEF 错误码追踪 |

### 2.3 系统资源

| 资源 | 监控项 | 采集源 |
|------|--------|--------|
| CPU | per-CPU 利用率、运行队列长度、上下文切换频率 | /proc/stat、/proc/sched_debug |
| 内存 | RSS、Cache、Swap、Slab、MemAvailable | /proc/meminfo |
| 磁盘 | 各挂载点使用率、IOPS、队列深度 | /proc/diskstats |
| 网络 | 各 NIC 收发字节、丢包率、TCP 重传率 | /proc/net/dev、/proc/net/snmp |
| IPC | io_uring 队列占用、SHM 段数、消息队列长度 | /proc/sys/airy/ipc/ |
| 内核 | softirq、hardirq、runnable tasks | /proc/softirqs、/proc/interrupts |

---

## 3. 监控指标体系

### 3.1 指标命名规范

所有指标采用 `<scope>_<object>_<dimension>_<unit>` 四段式命名：

- `scope`：`airy`（系统级）或 `agent`（Agent 级）。
- `object`：监控对象，如 `daemon`、`sched`、`mem`、`ipc`。
- `dimension`：具体维度，如 `cpu`、`rss`、`latency`。
- `unit`：单位后缀，如 `pct`、`bytes`、`ms`、`qps`。

示例：`airy_daemon_cpu_pct`、`agent_42_mem_l3_bytes`、`airy_sched_latency_p99_ms`。

### 3.2 核心指标清单

#### 3.2.1 CPU 指标

| 指标 | 单位 | 采集频率 | 告警阈值（见第 7 节） |
|------|------|----------|---------------------|
| `airy_daemon_cpu_pct` | % | 1s | WARNING: 80%, CRITICAL: 95% |
| `agent_N_cpu_pct` | % | 100ms | WARNING: 90%, CRITICAL: 99% |
| `airy_sched_latency_p99_ms` | ms | 1s | WARNING: 5ms, CRITICAL: 20ms |
| `airy_sched_dl_miss_total` | 次 | 1s | WARNING: 10/min, CRITICAL: 50/min |

#### 3.2.2 内存指标

| 指标 | 单位 | 采集频率 | 告警阈值 |
|------|------|----------|----------|
| `airy_mem_available_pct` | % | 5s | WARNING: 20%, CRITICAL: 10% |
| `airy_daemon_rss_bytes` | bytes | 1s | per-daemon MemoryMax 的 80% |
| `agent_N_mem_l1_bytes` | bytes | 1s | 256MB × 90% |
| `agent_N_mem_l2_bytes` | bytes | 1s | 1GB × 90% |
| `airy_mem_l3_total_bytes` | bytes | 5s | /var/lib 可用空间 < 20% |
| `airy_mem_l4_total_bytes` | bytes | 30s | /var/lib 可用空间 < 20% |

#### 3.2.3 IPC 延迟指标

| 指标 | 单位 | 采集频率 | 告警阈值 |
|------|------|----------|----------|
| `airy_ipc_avg_latency_ms` | ms | 100ms | WARNING: 1ms, CRITICAL: 5ms |
| `airy_ipc_p99_latency_ms` | ms | 100ms | WARNING: 5ms, CRITICAL: 20ms |
| `airy_ipc_ring_depth_pct` | % | 1s | WARNING: 70%, CRITICAL: 90% |
| `airy_ipc_dropped_total` | 次 | 事件驱动 | 任意非零即 WARNING |

#### 3.2.4 调度延迟指标

调度延迟指标来自 IRON-9 v3 模型 [SC]（Scheduling Core）层，通过 `sched:sched_switch`、`sched:sched_wakeup` tracepoint 计算任务从进入 runnable 状态到实际获得 CPU 的时间差。

| 指标 | 单位 | 计算方式 |
|------|------|----------|
| `airy_sched_latency_avg_ms` | ms | 滑动窗口平均 |
| `airy_sched_latency_p50_ms` | ms | 直方图分位 |
| `airy_sched_latency_p99_ms` | ms | 直方图分位 |
| `airy_sched_latency_p999_ms` | ms | 直方图分位 |
| `airy_sched_dl_miss_total` | 次 | SCHED_DEADLINE 期限违例累计 |

#### 3.2.5 Token 消耗指标

由 cogn_d 在每次推理调用前后埋点上报：

| 指标 | 单位 | 说明 |
|------|------|------|
| `agent_N_token_input_total` | token | 输入 token 累计 |
| `agent_N_token_output_total` | token | 输出 token 累计 |
| `agent_N_token_rate_per_sec` | token/s | 实时速率 |
| `agent_N_token_budget_remaining` | token | 剩余预算 |
| `airy_token_total_cost_usd` | USD | 预算成本估算（按模型定价表） |

---

## 4. /sys/airy/monitor/ 接口设计

### 4.1 接口层次结构

```
/sys/airy/monitor/
├── daemons/
│   ├── macro_d/
│   │   ├── liveness          # 0/1
│   │   ├── readiness         # 0/1
│   │   ├── health_score      # 0-100
│   │   ├── cpu_pct           # 单次读触发快照
│   │   ├── rss_bytes
│   │   ├── restart_count
│   │   ├── last_heartbeat_ms # 距离上次心跳毫秒数
│   │   └── stats             # 完整 JSON 快照
│   ├── logger_d/
│   │   └── ...
│   └── ... (12 daemon 各一目录)
├── agents/
│   ├── 1/
│   │   ├── state
│   │   ├── cpu_pct
│   │   ├── rss_bytes
│   │   ├── token_input_total
│   │   ├── token_output_total
│   │   ├── mem_l1_bytes
│   │   ├── mem_l2_bytes
│   │   ├── ipc_tx_msgs
│   │   ├── ipc_rx_msgs
│   │   ├── ipc_avg_latency_ms
│   │   └── stats
│   └── 2/ ...
├── system/
│   ├── cpu/
│   │   ├── utilization
│   │   ├── loadavg
│   │   ├── runnable_tasks
│   │   └── ctxt_switch_rate
│   ├── memory/
│   │   ├── available_pct
│   │   ├── slab_bytes
│   │   └── airy_l3_bytes
│   ├── ipc/
│   │   ├── avg_latency_ms
│   │   ├── p99_latency_ms
│   │   └── ring_depth_pct
│   └── sched/
│       ├── latency_p99_ms
│       └── dl_miss_total
└── aggregate/
    ├── top10_cpu_agents      # 实时 Top10 CPU 占用 Agent
    ├── top10_mem_agents
    ├── alert_count_5m        # 过去 5 分钟告警计数
    └── uptime_total
```

### 4.2 接口语义

- **读语义**：所有数值文件支持单次 `cat` 读取，返回当前快照值。读操作不产生副作用。
- **写语义**：除 `health_score` 由 macro_d 周期更新外，其余文件对用户态只读（0444 权限）。
- **原子性**：所有数值读取使用 `seqlock` 保证读到一致的快照，避免撕裂。
- **JSON stats**：`stats` 文件返回该对象的完整 JSON 快照，便于 devstation（详见 [10-devstation.md](10-devstation.md)）一次性拉取。

### 4.3 接口性能

- 单次读延迟 ≤50μs（命中 cache）。
- 全量 `cat /sys/airy/monitor/**` 在 12 daemon + 100 Agent 规模下 ≤50ms。
- 接口实现使用 `kobj_attribute`，避免实现自定义 VFS 操作，与sched_tac 保持一致。

---

## 5. 监控数据采集

### 5.1 采集层组件

| 组件 | 角色 | 实现位置 | 数据通路 |
|------|------|----------|----------|
| ftrace 采集器 | 内核 tracepoint 抓取 | 内核模块 `airy_mon_ftrace.ko` | `trace_pipe` → macro_d |
| sysfs 采集器 | sysfs 周期轮询 | macro_d 用户态线程 | 读 `/sys/airy/monitor/**` |
| procfs 采集器 | procfs 周期轮询 | macro_d 用户态线程 | 读 `/proc/**` |
| daemon 埋点 | 业务指标上报 | 12 daemon 各自实现 | io_uring CMD → logger_d |
| LSM hook 计数器 | 安全事件计数 | 内核 LSM 钩子 | `/sys/airy/security/` |

### 5.2 ftrace 采集

macro_d 启动后通过 `tracefs` 启用以下 tracepoint：

```
sched:sched_switch
sched:sched_wakeup
sched:sched_wakeup_new
sched:sched_process_exit
sched:sched_process_fork
irq:softirq_entry
irq:softirq_exit
syscalls:sys_enter_io_uring_setup
syscalls:sys_enter_io_uring_enter
```

trace 输出通过 `trace_pipe` 流式读取，避免 ring buffer 溢出。解析后的指标以 `per-CPU` 直方图聚合，避免全局锁竞争。

### 5.3 sysfs 与 procfs 采集

macro_d 内置采集线程池（默认 4 线程），按如下周期轮询：

| 采集目标 | 周期 | 线程 |
|----------|------|------|
| `/sys/airy/monitor/daemons/**` | 1s | thread-0 |
| `/sys/airy/monitor/agents/**` | 1s | thread-1 |
| `/sys/airy/monitor/system/**` | 1s | thread-2 |
| `/proc/stat`、`/proc/meminfo` 等 | 5s | thread-3 |

### 5.4 daemon 埋点上报

12 daemon 通过 A-IPC（Unified IPC）模块提供的 `airy_mon_emit()` 接口上报业务指标：

```c
struct airy_mon_metric {
    char name[64];
    uint64_t value;
    uint8_t  type;   /* 0=counter, 1=gauge, 2=histogram */
    uint64_t ts_ns;
};

int airy_mon_emit(const struct airy_mon_metric *m, size_t n);
```

底层实现复用 io_uring 的 `IORING_OP_URING_CMD`，将指标批量提交到 logger_d 的 ring buffer，避免 syscall 开销。

### 5.5 采集开销控制

- ftrace 启用前过滤：仅开启必要 tracepoint，禁用 `syscalls:sys_enter_*` 全集。
- ring buffer 大小：默认 8MB per CPU，可由 `airy_mon.buf_kb` 内核参数调整。
- 采集线程 SCHED_FIFO 优先级 10，避免被 Agent 抢占。
- 总体采集开销上限：单 CPU 1.5%，超出时自动降低采集频率（自适应降采样）。

---

## 6. 监控数据聚合

### 6.1 audit_d 聚合职责

audit_d daemon 是监控数据的唯一聚合层，承担以下职责：

1. **去重**：相同 `(name, labels)` 的指标在 100ms 窗口内仅保留最新值。
2. **降采样**：原始 100ms 采样数据按 1s/10s/1min 三档降采样，长期存储仅保留 1min 粒度。
3. **关联**：将指标与同一时间窗的日志事件关联，输出"事件-指标"对照视图。
4. **存储**：1min 粒度数据写入 `/var/lib/airy/monitor/`，保留 30 天。
5. **导出**：通过 `/sys/airy/monitor/aggregate/` 与 RPC 接口对外暴露。

### 6.2 聚合数据模型

audit_d 内部使用时序数据模型：

```
TSPoint {
    timestamp: uint64  (ns 精度)
    name:      string
    labels:    map<string, string>
    value:     double
    type:      enum(counter/gauge/histogram)
}
```

直方图类型额外存储 bucket 边界与计数。所有 TSPoint 序列化为紧凑二进制格式（参考 Prometheus remote write），单点平均 28 字节。

### 6.3 聚合流水线

```
[macro_d 指标流]  ─┐
                        ├──> [audit_d ingress] ─> [去重 100ms] ─> [降采样]
[logger_d 事件流] ─┘                                            │
                                                                     ▼
                                                              [关联分析]
                                                                     │
                                                  ┌──────────────────┼──────────────────┐
                                                  ▼                  ▼                  ▼
                                          [短期 1s 内存]      [中期 1min 文件]    [导出 RPC]
                                          保留 1 小时         保留 30 天          实时查询
```

### 6.4 存储与压缩

- **短期存储**：audit_d 进程内存，环形缓冲 1 小时数据，供实时面板使用。
- **中期存储**：`/var/lib/airy/monitor/<YYYYMMDD>/<daemon>.tsdb`，按 daemon 分片。
- **压缩**：每日凌晨 03:00 由 audit_d 触发 zstd 压缩，压缩比约 6:1。
- **清理**：超过 30 天的数据自动删除，由 cron-like 任务（由 macro_d 周期触发 audit_d）执行。

---

## 7. 告警阈值与通知机制

### 7.1 阈值配置

告警阈值通过 A-UCS（Unified Configuration Subsystem）模块 config_d 集中管理，配置文件位于 `/etc/airy/monitor/thresholds.yaml`：

```yaml
thresholds:
  - metric: airy_daemon_cpu_pct
    warning: 80
    critical: 95
    window: 60s
    repeat: 5m
  - metric: airy_mem_available_pct
    warning: 20
    critical: 10
    window: 30s
    repeat: 1m
  - metric: airy_sched_latency_p99_ms
    warning: 5
    critical: 20
    window: 10s
    repeat: 30s
  # ... 其余指标
```

每个阈值项包含：

- `metric`：指标名，支持通配符（如 `agent_*_cpu_pct`）。
- `warning`/`critical`：阈值边界。
- `window`：持续超阈值窗口才触发，避免瞬时抖动误报。
- `repeat`：告警重复间隔，控制告警风暴。

### 7.2 通知机制

macro_d 周期比对采集值与阈值，命中后通过以下通道通知：

| 通道 | 目的 | 延迟 |
|------|------|------|
| `/dev/airy_alert` 字符设备 | 内核态订阅者（如 sec_d） | <1ms |
| `sd_notify(STATUS=...)` | systemd 单元状态更新 | <5ms |
| `logger_d` Ring Buffer | 持久化 + 用户态订阅 | <10ms |
| RPC push | devstation 或远端监管中心 | <50ms |

详细告警流程与抑制机制见 [04-alerting.md](04-alerting.md)。

### 7.3 自适应阈值

对于 Token 消耗、IPC 流量等具有明显峰谷特征的指标，audit_d 支持基于历史 7 天数据的自适应基线：

- 计算 7 天同时段（每小时一个桶）的 p50/p95/p99。
- 实际值超过 p99 × 1.5 时触发 WARNING，超过 p99 × 3 时触发 CRITICAL。
- 自适应阈值每日 04:00 重算一次。

---

## 8. 与其他子系统的协作

### 8.1 与 A-ULP（统一日志）的关系

logger_d 在监控体系中承担"事件通道"角色：

- 所有 daemon 崩溃、重启、错误日志通过 A-ULP 上报。
- macro_d 订阅 logger_d 的实时流，将日志事件转化为监控指标（如 `airy_daemon_error_total`）。
- audit_d 同时消费指标流和日志流，进行关联分析。

### 8.2 与 A-ULS（统一监管）的关系

macro_d 在监控体系中承担"指标通道"角色：

- 周期采集 12 daemon 健康状态，作为 A-ULS 监管决策依据。
- 健康分低于阈值时触发 A-ULS 自动重启策略（详见 [05-incident-response.md](05-incident-response.md)）。
- A-ULS 的重启事件反向写入监控指标 `airy_daemon_restart_total`。

### 8.3 与 110-security（安全运维）的关系

- sec_d 通过 LSM hook 上报的安全事件计入监控指标 `airy_security_violation_total`。
- 严重安全违规直接触发 P0 事件响应流程。
- 监控数据本身受纯 C LSM 保护，仅授权用户（`airy-monitor` 组）可读取。

### 8.4 与 150-cloudnative（云原生）的关系

- Agent 迁移场景下，源节点与目标节点的 macro_d 通过 RPC 交换 Agent 监控快照。
- 迁移完成后指标标签 `host` 自动更新，避免历史数据丢失。
- 多节点场景下 audit_d 支持联邦查询，详见 [150-cloudnative](../150-cloudnative/README.md)。

---

## 9. 性能与扩展性

### 9.1 性能基线

在 8 核 32GB 内存的标准节点上，100 Agent 规模下监控开销：

| 项目 | 开销 | 占比 |
|------|------|------|
| ftrace 采集 | 0.4% CPU | 27% |
| sysfs 轮询 | 0.3% CPU | 20% |
| audit_d 聚合 | 0.5% CPU | 33% |
| logger_d 事件流 | 0.2% CPU | 13% |
| 其他 | 0.1% CPU | 7% |
| **总计** | **1.5% CPU** | 100% |

内存占用：audit_d 进程常驻 80MB，macro_d 采集缓冲 20MB。

### 9.2 扩展性

| Agent 规模 | 采集开销 | 聚合开销 | 总开销 |
|-----------|----------|----------|--------|
| 10 | 0.2% | 0.1% | 0.3% |
| 100 | 0.7% | 0.5% | 1.2% |
| 1000 | 2.5% | 2.0% | 4.5% |
| 10000 | 需启用分布式采集 | 需启用联邦聚合 | 见 150-cloudnative |

1000 Agent 以上规模建议启用：

- 分布式采集：每 100 Agent 分配一个 macro_d 副本。
- 联邦聚合：多节点 audit_d 通过 gRPC 互查。

### 9.3 失败处理

- audit_d 崩溃：macro_d 在 5s 内重启，期间短期缓冲保留在 logger_d Ring Buffer，不丢失数据。
- macro_d 崩溃：systemd 在 10s 内（WatchdogSec）重启，期间 sysfs 数据仍可由用户态直读。
- ftrace ring buffer 溢出：丢弃旧数据并增加 `airy_mon_dropped_total` 计数，触发 WARNING 告警。

---

## 10. 配置参数

监控体系的关键配置参数（通过 A-UCS 管理，默认值如下）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `airy_mon.buf_kb` | 8192 | ftrace ring buffer 单 CPU 大小（KB） |
| `airy_mon.collect_interval_ms` | 1000 | sysfs/procfs 采集间隔 |
| `airy_mon.agent_collect_interval_ms` | 100 | Agent 级采集间隔 |
| `airy_mon.audit_retention_days` | 30 | 中期存储保留天数 |
| `airy_mon.audit_mem_buffer_hours` | 1 | 短期内存缓冲小时数 |
| `airy_mon.adaptive_baseline_enabled` | true | 是否启用自适应阈值 |
| `airy_mon.max_overhead_pct` | 1.5 | 采集开销上限（%） |

参数热更新通过 `config_d` 发送 SIGHUP 至 macro_d 与 audit_d 完成。

---

## 11. 安全考虑

### 11.1 访问控制

- `/sys/airy/monitor/**` 默认 0444，所有用户可读。
- `/sys/airy/monitor/aggregate/top10_*` 包含 Agent 标识，仅 `airy-monitor` 组可读（0640）。
- `/var/lib/airy/monitor/` 目录 0750，属主 `airy:airy-monitor`。
- RPC 查询接口需 mTLS 认证，证书由 sec_d 颁发。

### 11.2 数据脱敏

- Agent 上下文内容不进入监控指标，仅采集 token 数量。
- 日志事件中的敏感字段（如密钥、PII）由 logger_d 在写入前脱敏。
- audit_d 持久化时对 Agent 名称进行哈希处理（可选，默认关闭）。

### 11.3 审计自身

audit_d 自身操作（写入、删除、查询）通过 LSM hook 记录到 `/var/log/airy/audit_audit.log`，防止监控数据被篡改。

---

## 12. 版本演进

### 12.1 v0.1.1 → v1.0.1 变更

- 新增 `airy_token_total_cost_usd` 指标。
- 修复 `airy_sched_latency_p99_ms` 在 SCHED_DEADLINE 任务下计算偏差。
- `/sys/airy/monitor/aggregate/` 新增 `top10_*` 系列。
- audit_d 存储压缩比从 4:1 提升至 6:1（改用 zstd level 6）。

### 12.2 后续规划（v1.1.0）

- 引入 eBPF 辅助采集（不替代 ftrace，仅用于细粒度内核函数追踪）。
- 支持分布式 macro_d 副本（1000+ Agent 场景）。
- 自适应阈值引入机器学习模型（基于 audit_d 历史数据训练）。

---

## 13. 相关文档

- [04-alerting.md](04-alerting.md)：告警体系，定义告警级别、传输、抑制。
- [05-incident-response.md](05-incident-response.md)：事件响应流程。
- [07-systemd-integration.md](07-systemd-integration.md)：systemd 集成与 WatchdogSec。
- [10-devstation.md](10-devstation.md)：开发运维工作站，提供监控可视化。
- [../110-security/README.md](../110-security/README.md)：安全运维，与监控的协作。
- [../150-cloudnative/README.md](../150-cloudnative/README.md)：云原生场景下的分布式监控。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
