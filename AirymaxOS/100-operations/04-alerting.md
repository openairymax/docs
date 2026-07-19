Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 告警体系（Alerting System）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · 告警体系设计文档，定义告警级别、来源、传输路径、抑制策略与恢复确认机制\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

告警体系是 AirymaxOS 运维分册的核心子模块，向上承接监控体系（详见 [03-monitoring.md](03-monitoring.md)）的阈值判定结果，向下驱动事件响应流程（详见 [05-incident-response.md](05-incident-response.md)）。设计目标如下：

1. **低延迟**：从异常发生到告警送达订阅方的端到端延迟 <100ms（CRITICAL 级）/ <500ms（WARNING 级）。
2. **不丢不重**：告警传输基于 A-ULP 模块 logger_d 的 Ring Buffer，保证至少一次（at-least-once）语义。
3. **避免告警风暴**：通过抑制、聚合、分组三重机制，确保单次根因事件最多产生 1 条主告警 + N 条关联告警摘要。
4. **可恢复**：所有告警必须有明确的恢复条件与恢复确认机制，避免"幽灵告警"。
5. **多通道分发**：同时支持内核态（`/dev/airy_alert`）、系统态（systemd notify）、用户态（RPC）订阅。

### 1.2 与监控体系的关系

告警体系与监控体系的边界：

| 维度 | 监控体系 | 告警体系 |
|------|----------|----------|
| 数据 | 原始指标 + 日志事件 | 阈值命中后的告警事件 |
| 触发 | 周期采集 + 事件驱动 | 阈值判定 + 模式匹配 |
| 频率 | 高（100ms-1s） | 低（仅异常时） |
| 主要模块 | macro_d + audit_d | macro_d + audit_d + logger_d |
| 文档 | [03-monitoring.md](03-monitoring.md) | 本文 |

监控体系负责"发现"，告警体系负责"通知与跟踪"。

### 1.3 与技术选型的契合

- **IORING_OP_URING_CMD**：告警提交通道复用 io_uring，避免高并发场景下的 syscall 阻塞。
- **纯 C LSM**：安全告警由 LSM hook 直接触发，无需经过用户态审计中转。
- **IRON-9 v3 四层模型**：[DSL]（Degraded Service Layer）降级触发告警，由 sched_d 上报。
- **systemd 集成**：所有 daemon 告警通过 `sd_notify` 同步 systemd 单元状态。

---

## 2. 告警级别

### 2.1 三级告警体系

AirymaxOS 告警体系采用 CRITICAL / WARNING / INFO 三级，与 A-ULP 5 级日志体系的映射关系如下：

| 告警级别 | A-ULP 日志级别 | 含义 | 响应 SLA | 升级目标 |
|----------|--------------|------|----------|----------|
| CRITICAL | ERROR / FATAL | 系统功能受损，需立即人工介入 | 5 分钟 | P0 事件响应 |
| WARNING | WARN | 潜在风险，需关注但不需立即介入 | 30 分钟 | P1 事件响应 |
| INFO | INFO | 信息性通知，无需人工介入 | 无 | 仅记录 |

A-ULP 的 DEBUG 级日志不产生告警。A-ULP 的 TRACE 级日志仅在 `airy_alert.trace_enabled=true` 时产生 INFO 级告警，默认关闭。

### 2.2 级别判定规则

告警级别由 macro_d 根据以下规则综合判定：

1. **阈值规则**：指标超过 `critical` 阈值 → CRITICAL；超过 `warning` 阈值 → WARNING。
2. **持续窗口**：阈值命中后须持续 `window` 时间，否则降一级（CRITICAL → WARNING，WARNING → INFO）。
3. **影响范围**：影响 ≥3 个 Agent 或 ≥1 个核心 daemon（macro_d/logger_d/config_d/sched_d/sec_d）的告警强制升级为 CRITICAL。
4. **重复频次**：同源告警在 `repeat` 窗口内重复 ≥5 次，强制升级一级。

### 2.3 告警级别示例

| 场景 | 默认级别 | 触发条件 |
|------|----------|----------|
| macro_d 进程崩溃 | CRITICAL | systemd WatchdogSec 超时 |
| logger_d Ring Buffer 满 | WARNING | 占用率 >90% 持续 30s |
| Agent 健康分 <30 | WARNING | health_score <30 持续 60s |
| Agent 健康分 <10 | CRITICAL | health_score <10 持续 10s |
| sched_d SCHED_DEADLINE miss >50/min | CRITICAL | dl_miss 计数 1 分钟内 >50 |
| net_d 重传率 >5% | WARNING | 重传率 1 分钟均值 >5% |
| net_d 链路中断 | CRITICAL | 主 NIC 状态 down |
| sec_d LSM hook 拒绝 >10/s | CRITICAL | 拒绝速率 1 秒内 >10 |
| mem_d L1 溢出 LRU 淘汰 | INFO | 单次淘汰事件 |
| mem_d L1 持续溢出 >10/min | WARNING | 淘汰速率 1 分钟内 >10 |
| cogn_d Token 预算耗尽 | WARNING | remaining=0 |
| config_d 配置校验失败 | CRITICAL | 热更新校验失败 |
| audit_d 存储空间 <10% | CRITICAL | /var/lib 可用空间 <10% |

---

## 3. 告警源

### 3.1 告警源清单

AirymaxOS 告警来自以下源：

| 告警源 | 类型 | 主要告警场景 | 上报方式 |
|--------|------|--------------|----------|
| macro_d | 进程监管 | daemon 崩溃、重启、心跳超时、健康分异常 | 直接调用 `airy_alert_emit()` |
| sched_d | 调度监管 | SCHED_DEADLINE miss、调度延迟超阈值、CPU 饥饿 | io_uring CMD → logger_d |
| net_d | 网络监管 | 链路中断、重传率高、连接跟踪表满 | io_uring CMD → logger_d |
| mem_d | 内存监管 | L1-L4 溢出、defrag 失败、OOM 临近 | io_uring CMD → logger_d |
| sec_d | 安全监管 | LSM hook 命中、密钥访问异常、能力越权 | LSM hook 直采 → `/dev/airy_alert` |
| cogn_d | 认知监管 | Token 预算耗尽、推理队列堆积、模型不可用 | io_uring CMD → logger_d |
| vfs_d | 文件系统监管 | fsync 延迟超阈值、inode 耗尽、只读挂载 | io_uring CMD → logger_d |
| gateway_d | 网关监管 | 握手失败率、请求超时、证书过期 | io_uring CMD → logger_d |
| config_d | 配置监管 | 校验失败、热更新冲突、版本回退 | 直接调用 `airy_alert_emit()` |
| audit_d | 聚合监管 | 自身存储满、聚合延迟、关联分析异常 | 直接调用 `airy_alert_emit()` |
| 内核 LSM hook | 安全钩子 | 进程越权、文件非法访问、网络异常连接 | `/dev/airy_alert` 字符设备 |
| systemd | init 系统 | 单元失败、WatchdogSec 超时、依赖循环 | `sd_notify` → macro_d |

### 3.2 告警源详细说明

#### 3.2.1 macro_d（进程监管）

macro_d 作为 A-ULS 模块的核心，负责 12 daemon 的生命周期监管。其告警场景：

- **进程崩溃**：检测到子 daemon 进程退出（通过 `SIGCHLD` 或 systemd 通知），若退出码非 0 或在 5 分钟内重启 ≥3 次，触发 CRITICAL 告警。
- **心跳超时**：每个 daemon 周期上报心跳（默认 2s 一次），超过 `heartbeat_timeout`（默认 6s）未收到，触发 CRITICAL 告警。
- **健康分异常**：daemon 的 `health_score` <30 持续 60s 触发 WARNING，<10 持续 10s 触发 CRITICAL。
- **重启风暴**：单 daemon 在 5 分钟内重启 ≥5 次，触发 CRITICAL 告警，并暂停自动重启（避免雪崩）。

#### 3.2.2 sched_d（调度监管）

sched_d 基于 ftrace tracepoint 监控调度行为：

- **SCHED_DEADLINE miss**：deadline 任务未在期限内完成，按 1 分钟窗口计数，>50 触发 CRITICAL。
- **调度延迟超阈值**：`airy_sched_latency_p99_ms > 20` 持续 10s 触发 CRITICAL。
- **CPU 饥饿**：单 CPU 运行队列长度 >32 持续 30s 触发 WARNING，>64 持续 30s 触发 CRITICAL。
- **IRON-9 [SC] 层异常**：调度核心层内部状态机异常，直接触发 CRITICAL。

#### 3.2.3 net_d（网络监管）

net_d 通过 `/proc/net/` 与 daemon 内埋点采集：

- **链路中断**：主 NIC 状态变为 down，立即触发 CRITICAL。
- **重传率高**：1 分钟平均重传率 >5% 触发 WARNING，>20% 触发 CRITICAL。
- **连接跟踪表满**：conntrack 表占用 >80% 触发 WARNING，>95% 触发 CRITICAL。
- **DNS 解析失败**：连续 3 次解析失败触发 WARNING。

#### 3.2.4 sec_d（安全监管）

sec_d 通过纯 C LSM hook 采集安全事件，是唯一可绕过用户态直接产生告警的源：

- **LSM hook 命中**：进程越权、文件非法访问等，按 1 秒窗口计数，>10 触发 CRITICAL。
- **密钥访问异常**：未授权进程访问 `/etc/airy/keys/` 立即触发 CRITICAL。
- **能力变更**：进程 capability 集合异常扩展触发 WARNING。
- **审计日志篡改**：检测到 `/var/log/airy/` 下文件被非 audit_d 进程修改，触发 CRITICAL。

---

## 4. 告警传输

### 4.1 传输流水线

告警从产生到送达订阅方的完整流水线：

```
[告警源]                                                       [订阅方]
   │                                                              ▲
   │ 1. airy_alert_emit() 或 /dev/airy_alert 写入                  │
   ▼                                                              │
┌─────────────────────────────────────────────────────────────────┤
│  logger_d Ring Buffer（统一入口）                          │
│  - 容量：16MB，溢出时丢弃最旧 INFO，保留 CRITICAL/WARNING        │
└──────────────────┬──────────────────────────────────────────────┘
                   │ 2. 流式读取
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  audit_d 聚合层                                                  │
│  - 去重（同源同类型 100ms 窗口）                                 │
│  - 抑制（见第 6 节）                                             │
│  - 分组（关联告警合并）                                          │
│  - 路由（按订阅分发）                                            │
└────┬──────────┬──────────────┬──────────────┬───────────────────┘
     │          │              │              │
     ▼          ▼              ▼              ▼
┌────────┐ ┌─────────┐  ┌────────────┐  ┌──────────────┐
│systemd │ │/dev/airy│  │journal     │  │RPC push      │
│notify  │ │_alert   │  │持久化      │  │(devstation等)│
└────────┘ └─────────┘  └────────────┘  └──────────────┘
```

### 4.2 logger_d Ring Buffer

Ring Buffer 是告警传输的统一入口，所有告警源必须经过此通道。设计要点：

- **容量**：默认 16MB，可由 `airy_alert.ring_buf_mb` 配置。
- **背压**：当占用率 >90% 时，logger_d 拒绝写入新的 INFO 级告警，返回 `-EAGAIN`；CRITICAL/WARNING 强制写入，丢弃最旧的 INFO。
- **溢出处理**：CRITICAL 告警永不丢弃；WARNING 在极端情况下（buffer 满 99%）才丢弃最旧的 WARNING；丢弃时增加 `airy_alert_dropped_total` 计数。
- **多读**：支持多个消费者同时读取，每个消费者维护独立的 cursor。

Ring Buffer 单条记录格式：

```c
struct airy_alert_record {
    uint64_t ts_ns;          /* 纳秒时间戳 */
    uint8_t  severity;       /* 0=INFO, 1=WARNING, 2=CRITICAL */
    uint8_t  source_id;      /* 告警源 ID（macro_d=1, sched_d=2, ...） */
    uint16_t alert_code;     /* 告警码（A-UEF 错误码空间） */
    uint32_t agent_id;       /* 关联 Agent ID，0 表示系统级 */
    char     message[256];   /* 人可读消息 */
    uint32_t labels_off;     /* 标签区偏移 */
    uint32_t labels_len;     /* 标签区长度 */
    /* followed by labels in "k=v\0k=v\0" format */
};
```

### 4.3 audit_d 聚合与路由

audit_d 从 Ring Buffer 流式读取告警，经过以下处理阶段：

1. **解析**：将二进制记录解析为内部 `AlertEvent` 结构。
2. **去重**：相同 `(source_id, alert_code, agent_id, labels)` 的告警在 100ms 窗口内仅保留最新一条。
3. **抑制**：根据抑制规则（见 6.1）过滤告警。
4. **分组**：将同一根因的多个告警合并为一个"告警组"，组内保留主告警 + 关联告警摘要。
5. **路由**：根据订阅配置分发到各通道。

### 4.4 systemd notify 通道

每个 daemon 的 systemd 单元配置 `Type=notify`，通过 `sd_notify` 向 systemd 发送状态：

- CRITICAL 告警：`STATUS=CRITICAL: <message>` + `ERRNO=<alert_code>`，systemd 单元状态显示为 `failed`。
- WARNING 告警：`STATUS=WARNING: <message>`，systemd 单元状态保持 `active`。
- INFO 告警：不通过 systemd notify 发送，仅记录到 journal。

systemd 单元状态变更会触发 `OnFailure=` 钩子，可配置级联动作（如重启依赖单元）。

### 4.5 /dev/airy_alert 字符设备

`/dev/airy_alert` 是 audit_d 创建的字符设备，用于内核态与用户态订阅者接收告警：

- **读取语义**：`read()` 阻塞直到有告警可读；`O_NONBLOCK` 模式下无告警时返回 `-EAGAIN`。
- **多路复用**：支持 `poll()`/`epoll`，便于集成到事件循环。
- **过滤**：`ioctl(DEV_AIRY_ALERT_SET_FILTER, ...)` 可按 severity/source_id 过滤。
- **权限**：0660，属主 `root:airy-alert`。

主要订阅者：

- sec_d：实时接收安全告警，触发隔离策略。
- macro_d：接收跨 daemon 告警，触发 A-ULS 自动重启。
- devstation：实时面板展示（详见 [10-devstation.md](10-devstation.md)）。

### 4.6 RPC 推送通道

audit_d 内置 gRPC 服务端，支持远端订阅者通过 `SubscribeAlerts` 流式 RPC 接收告警：

```protobuf
service AlertService {
    rpc SubscribeAlerts(AlertFilter) returns (stream AlertEvent);
    rpc AckAlert(AckRequest) returns (AckResponse);
    rpc QueryHistory(HistoryQuery) returns (stream AlertEvent);
}

message AlertEvent {
    uint64 ts_ns = 1;
    Severity severity = 2;
    uint32 source_id = 3;
    uint32 alert_code = 4;
    uint32 agent_id = 5;
    string message = 6;
    map<string, string> labels = 7;
    string alert_id = 8;  // 全局唯一 ID，用于 ack
}
```

RPC 通道默认启用 mTLS，证书由 sec_d 颁发，订阅者需提供客户端证书。

---

## 5. 告警通知渠道

### 5.1 渠道矩阵

| 渠道 | 默认级别 | 用途 | 延迟 | 配置 |
|------|----------|------|------|------|
| systemd notify | CRITICAL/WARNING | systemd 单元状态 | <5ms | 自动 |
| /dev/airy_alert | 全部 | 内核态 + 用户态订阅 | <1ms | `/etc/airy/alert/channels.conf` |
| systemd journal | 全部 | 持久化归档 | <10ms | 自动 |
| RPC push | 全部 | devstation / 远端监管 | <50ms | mTLS 证书 |
| Email | CRITICAL | 人工通知 | <60s | SMTP 配置（可选） |
| Webhook | CRITICAL/WARNING | 外部系统集成 | <1s | URL 配置（可选） |
| SMS | CRITICAL | 紧急人工通知 | <30s | 网关配置（可选） |

Email/Webhook/SMS 默认关闭，需在 `/etc/airy/alert/channels.conf` 显式启用。

### 5.2 渠道配置示例

```ini
# /etc/airy/alert/channels.conf

[systemd_notify]
enabled = true
min_severity = WARNING

[airy_alert_dev]
enabled = true
min_severity = INFO
path = /dev/airy_alert

[journal]
enabled = true
min_severity = INFO
facility = LOCAL4

[rpc_push]
enabled = true
min_severity = INFO
listen = 0.0.0.0:7463
mtls_ca = /etc/airy/keys/ca.pem
mtls_cert = /etc/airy/keys/audit_d.pem
mtls_key = /etc/airy/keys/audit_d.key

[email]
enabled = false
min_severity = CRITICAL
smtp_host = smtp.spharx.com
smtp_port = 587
from = alert@airy-os.local
to = ops@spharx.com

[webhook]
enabled = false
min_severity = WARNING
url = https://hooks.internal/alert
timeout_ms = 1000

[sms]
enabled = false
min_severity = CRITICAL
gateway = http://sms-gw.local/send
recipients = +8613800000001,+8613800000002
```

### 5.3 渠道故障处理

- **journal 写入失败**：audit_d 重试 3 次，仍失败则降级写入 `/var/log/airy/alert_fallback.log`，并触发 WARNING 告警（自身告警，不通过 journal 通道）。
- **RPC push 失败**：audit_d 缓冲到本地队列（最多 1000 条），订阅者重连后批量补发。
- **Email/SMS/Webhook 失败**：重试 3 次（指数退避），仍失败则记录到 journal，不影响主告警流程。

---

## 6. 告警抑制与聚合

### 6.1 告警抑制

告警抑制（Suppression）用于避免告警风暴，规则定义在 `/etc/airy/alert/suppression.yaml`：

```yaml
suppressions:
  - name: sched_d_dl_miss_storm
    condition:
      source_id: 2          # sched_d
      alert_code: 2003      # DL miss
    suppress_window: 5m
    max_emit: 1
    note: 5 分钟内最多发 1 条 DL miss 告警

  - name: net_d_retrans_storm
    condition:
      source_id: 5          # net_d
      severity: WARNING
    suppress_window: 10m
    max_emit: 3

  - name: maintenance_window
    condition:
      labels.maintenance: "true"
    suppress_window: 1h
    max_emit: 0             # 完全抑制
```

抑制规则字段：

- `condition`：匹配条件，支持 source_id、alert_code、severity、labels 字段匹配。
- `suppress_window`：抑制窗口，从首次匹配开始计时。
- `max_emit`：窗口内最多发送次数，0 表示完全抑制。
- 抑制期间被丢弃的告警计入 `airy_alert_suppressed_total`，可在 audit_d 日志中查询。

### 6.2 告警聚合

告警聚合（Aggregation）将同一根因的多个告警合并为告警组。聚合规则：

1. **同源同码**：相同 source_id 与 alert_code 的告警，在 100ms 窗口内合并为一条，message 后追加 `(×N)` 计数。
2. **同 Agent 关联**：同一 Agent 在 5s 内触发的多个不同告警，合并为"Agent 告警组"，主告警为 severity 最高者。
3. **根因推断**：audit_d 内置根因规则库，例如：

| 现象告警 | 推断根因 | 聚合规则 |
|----------|----------|----------|
| 多个 daemon 心跳超时 | macro_d 自身异常 | 合并为 1 条主告警（macro_d CRITICAL） |
| net_d 链路中断 + gateway_d 握手失败 | 网络链路中断 | 合并为 1 条主告警（net_d CRITICAL） |
| mem_d OOM 临近 + 多个 Agent 健康分下降 | 系统内存不足 | 合并为 1 条主告警（mem_d CRITICAL） |
| sched_d 调度延迟超阈值 + 多个 Agent CPU 高 | CPU 饱和 | 合并为 1 条主告警（sched_d CRITICAL） |

聚合后的告警组通过 RPC 推送时携带 `correlated_alerts` 字段，订阅方可展开查看明细。

### 6.3 告警风暴检测

audit_d 实时监控告警速率，当满足以下任一条件时进入"风暴模式"：

- 1 秒内新增告警 >50 条。
- 1 分钟内新增告警 >1000 条。
- 同一 alert_code 1 分钟内触发 >20 次。

风暴模式下：

- WARNING/INFO 告警自动应用 5 分钟抑制窗口（max_emit=1）。
- CRITICAL 告警保留，但 message 截断为 64 字节。
- 风暴状态写入 `/sys/airy/monitor/aggregate/alert_storm_active`，供 devstation 展示。
- 风暴结束后（告警速率回落至阈值以下 5 分钟）退出风暴模式，发送 1 条 INFO 告警"风暴结束摘要"。

---

## 7. 告警恢复确认机制

### 7.1 告警生命周期

每条告警经历以下状态：

```
[告警触发] ──> FIRING ──> [恢复条件满足] ──> PENDING_RESOLVE ──> [确认] ──> RESOLVED
                  │                                              │
                  └─────────[持续超阈值 3 个窗口]──────────────┘
                                                              │
                                                              ▼
                                                         ESCALATED
```

状态说明：

| 状态 | 含义 | 自动/人工 |
|------|------|-----------|
| FIRING | 告警已触发，异常持续中 | 自动 |
| PENDING_RESOLVE | 异常已恢复，等待确认 | 自动 |
| RESOLVED | 已确认恢复 | 人工 ack 或自动确认 |
| ESCALATED | 持续未恢复，升级处理 | 自动 |

### 7.2 恢复条件

每条告警的恢复条件由告警源定义，常见模式：

- **阈值类告警**：指标连续 3 个采集窗口（默认 3 分钟）低于 warning 阈值。
- **进程类告警**：daemon 重启后健康分恢复至 ≥80 持续 60s。
- **网络类告警**：链路状态恢复 up 持续 30s，重传率回落至 <1%。
- **安全类告警**：需人工干预（如关闭违规进程），不支持自动恢复。

### 7.3 自动恢复确认

满足恢复条件后，audit_d 自动将告警状态从 FIRING 切换到 PENDING_RESOLVE，并在以下情况自动确认（状态变为 RESOLVED）：

- INFO/WARNING 级告警：PENDING_RESOLVE 状态持续 5 分钟无复发，自动确认。
- CRITICAL 级告警：PENDING_RESOLVE 状态持续 15 分钟无复发，自动确认。
- 自动确认后发送 1 条 INFO 级"恢复通知"到所有原订阅渠道。

### 7.4 人工恢复确认

对于不支持自动恢复或运维要求人工确认的场景，订阅方可通过 RPC 主动 ack：

```bash
# 通过 devstation CLI
airyctl alert ack <alert_id> --comment "Root cause: NIC firmware bug, upgraded to v2.3"

# 通过 RPC
AlertService.AckAlert({
    alert_id: "<alert_id>",
    operator: "alice",
    comment: "..."
})
```

人工 ack 后状态变为 RESOLVED，告警记录中包含 `resolved_by` 与 `resolution_comment` 字段，用于复盘。

### 7.5 升级机制

CRITICAL 告警在 FIRING 状态持续 30 分钟未恢复，自动升级为 ESCALATED：

- audit_d 发送"升级通知"到所有渠道（包括 Email/SMS，即使默认关闭）。
- 升级通知触发 P0 事件响应流程（详见 [05-incident-response.md](05-incident-response.md)）。
- ESCALATED 告警每 30 分钟重复通知一次，直至恢复。

---

## 8. 告警归档与查询

### 8.1 归档策略

| 级别 | journal 保留 | audit_d 文件归档 | 长期归档 |
|------|--------------|------------------|----------|
| CRITICAL | 90 天 | 90 天 | 1 年（压缩归档） |
| WARNING | 30 天 | 30 天 | 6 个月（压缩归档） |
| INFO | 7 天 | 7 天 | 不长期归档 |

归档文件位于 `/var/lib/airy/alerts/<YYYYMM>/<DD>.alert.zst`，每日凌晨 02:00 由 audit_d 压缩归档。

### 8.2 查询接口

audit_d 提供 `AlertService.QueryHistory` RPC 接口，支持按时间范围、级别、来源、Agent ID 查询历史告警：

```bash
# 查询过去 1 小时所有 CRITICAL 告警
airyctl alert query --severity CRITICAL --since 1h

# 查询特定 Agent 的告警历史
airyctl alert query --agent-id 42 --since 24h

# 查询特定告警码的历史
airyctl alert query --code 2003 --since 7d
```

查询接口同样支持 devstation 的"告警历史"面板。

### 8.3 告警统计

audit_d 每日生成告警统计报告，写入 `/var/lib/airy/alerts/daily/<YYYYMMDD>.summary`：

- 按级别统计：CRITICAL/WARNING/INFO 各多少条。
- 按来源统计：各 source_id 触发次数。
- 按 Agent 统计：触发告警最多的 Top10 Agent。
- MTTR（平均恢复时间）：按级别统计。
- 抑制率：被抑制告警占总告警比例。

---

## 9. 配置参数

告警体系的关键配置参数（通过 A-UCS 管理）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `airy_alert.ring_buf_mb` | 16 | logger_d 告警 Ring Buffer 大小（MB） |
| `airy_alert.heartbeat_timeout_s` | 6 | daemon 心跳超时阈值（秒） |
| `airy_alert.window_dedup_ms` | 100 | 去重窗口（毫秒） |
| `airy_alert.storm_threshold_1s` | 50 | 1 秒告警风暴阈值 |
| `airy_alert.storm_threshold_1m` | 1000 | 1 分钟告警风暴阈值 |
| `airy_alert.auto_resolve_warn_min` | 5 | WARNING 自动恢复确认等待（分钟） |
| `airy_alert.auto_resolve_crit_min` | 15 | CRITICAL 自动恢复确认等待（分钟） |
| `airy_alert.escalate_min` | 30 | CRITICAL 升级等待（分钟） |
| `airy_alert.retention_critical_d` | 90 | CRITICAL 归档保留（天） |
| `airy_alert.retention_warning_d` | 30 | WARNING 归档保留（天） |
| `airy_alert.retention_info_d` | 7 | INFO 归档保留（天） |

参数热更新通过 config_d 发送 SIGHUP 至 audit_d 完成。

---

## 10. 安全考虑

### 10.1 告警防伪造

- logger_d Ring Buffer 仅接受 root 或 `airy-alert` 组进程写入。
- 写入时附带发送方 PID 与 UID，audit_d 校验来源合法性。
- sec_d 监控 Ring Buffer 写入方，非授权进程立即触发 CRITICAL 告警。

### 10.2 告警内容脱敏

- 告警 message 中不应包含密钥、token、PII 等敏感信息。
- logger_d 内置脱敏规则（正则匹配），命中后替换为 `***`。
- sec_d 周期审计历史告警，发现敏感信息泄露时触发 WARNING 告警。

### 10.3 订阅方认证

- `/dev/airy_alert` 设备权限 0660，仅 `airy-alert` 组可读。
- RPC 推送强制 mTLS，客户端证书由 sec_d 颁发，支持吊销。
- Email/SMS/Webhook 渠道配置支持加密存储（AES-256），密钥由 sec_d 托管。

### 10.4 审计自身

audit_d 处理告警的过程本身被审计：

- 所有 ack 操作记录到 `/var/log/airy/alert_ack.log`，含操作者、时间、comment。
- 抑制规则变更记录到 journal，由 config_d 上报。
- 告警丢弃事件（Ring Buffer 溢出）触发独立的"告警系统健康"CRITICAL 告警。

---

## 11. 与其他子系统的协作

### 11.1 与监控体系

告警体系完全依赖监控体系提供的阈值判定结果。监控体系变更（如新增指标）需同步更新告警阈值配置，详见 [03-monitoring.md](03-monitoring.md) 第 7 节。

### 11.2 与事件响应

CRITICAL 告警自动触发 P0 事件响应流程，WARNING 告警触发 P1 事件响应流程，详见 [05-incident-response.md](05-incident-response.md)。

### 11.3 与 A-ULS（统一监管）

- macro_d 既是告警源，也是告警订阅方（接收跨 daemon 告警触发自动重启）。
- A-ULS 重启事件产生 INFO 告警，记录到 audit_d 用于复盘。

### 11.4 与 A-ULP（统一日志）

- 所有告警同时写入 A-ULP 日志流，与 ERROR/WARN/INFO 级别对齐。
- logger_d 的 Ring Buffer 复用为告警传输通道，避免重复基础设施。

### 11.5 与 110-security（安全运维）

- sec_d 是唯一可绕过用户态直接产生告警的源（通过 LSM hook）。
- 安全告警优先级最高，CRITICAL 级安全告警直接触发 sec_d 隔离策略。
- 详见 [../110-security/README.md](../110-security/README.md)。

---

## 12. 版本演进

### 12.1 v1.0.0 → v1.0.1 变更

- 新增告警风暴检测与自动抑制。
- RPC 推送增加 mTLS 强制要求（v1.0.0 可选）。
- 告警归档压缩比从 5:1 提升至 7:1（改用 zstd level 9）。
- 修复 PENDING_RESOLVE 状态下告警重复触发的 bug。

### 12.2 后续规划（v1.1.0）

- 引入基于机器学习的异常检测，补充阈值告警。
- 支持告警分组规则的自定义（用户可配置根因推断规则）。
- 多节点场景下的联邦告警（与 150-cloudnative 联动）。

---

## 13. 自动化 Remediation Playbook（R2 补强新增）

> **章节定位**：本节为 R2 metrics/observability 自动化 remediation 补强，定义"指标超阈值 → 自动触发 remediation 动作"的文档化 playbook。本节不修改 §1-§12 任何既有告警/抑制/聚合机制，仅在告警传输链路（§4）之上补强"自动响应动作"。所有 remediation 动作均通过 systemd timer + `airy-remediation.service` 执行，由 Prometheus alertmanager 触发。
>
> **与既有机制的关系**：
> - 与 §6 告警抑制的关系：remediation 触发后，相关告警进入 5 分钟静默期（§13.4），避免雪崩。
> - 与 §7 告警恢复确认的关系：remediation 动作不自动 ack 告警，告警仍按 §7 生命周期管理。
> - 与 [05-incident-response.md](05-incident-response.md) 的关系：本节定义**自动**响应，事件响应文档定义**人工**响应；自动响应失败或升级条件满足时，自动转入人工响应流程。

### 13.1 Playbook 表格

下表定义 10 项关键指标超阈值时的自动响应动作。每行包含：指标名、阈值、检测窗口、自动响应、升级条件。

| # | 指标 | 阈值 | 检测窗口 | 自动响应 | 升级条件 |
|---|------|------|---------|---------|---------|
| 1 | `airy_ipc_badge_fail_total`<br>（Badge 校验失败次数，fastpath C-S9 检测，详见 [../110-security/07-airy-lsm-design.md §3.2](../110-security/07-airy-lsm-design.md)） | > 100/min | 60s 滚动窗口 | ① 冻结涉事 Agent（`airyctl agent freeze <agent_id>`）<br>② 通知 Macro-Supervisor（eventfd_signal）<br>③ 审计日志（A-ULP `AGENT_CAP_FAULT`） | 单 Agent 1 小时内累计冻结 ≥3 次 → CRITICAL 告警 + 转人工响应（P0） |
| 2 | `airy_sec_d_request_latency_p99`<br>（sec_d 请求 P99 延迟） | > 100ms | 30s 滚动窗口 | ① sec_d 启用限流降级（`airy_sec_d_throttle_check` 自动扩大拒绝阈值）<br>② WARNING 告警<br>③ 转发至 [DSL] 降级模式（如已持续 5min） | 持续超阈值 ≥10min → CRITICAL 告警 + 转人工响应（P1） |
| 3 | `airy_sec_d_throttle_total`<br>（sec_d 限流次数，触发 `-83` AIRY_ESEC_D_THROTTLED） | > 1000/min | 60s 滚动窗口 | ① 全局 CRITICAL 告警<br>② 自动扩大 sec_d 配额 2x（`airyctl sec_d quota --scale 2x`，上限 8x） | 配额已达 8x 上限仍持续 → 转人工响应（P0）+ 考虑 sec_d 横向扩容 |
| 4 | `airy_uring_malformed_total`<br>（malformed SQE/CQE 次数，触发 `AIRY_FAULT_URING_MALFORMED` 0x100A） | > 10/min | 60s 滚动窗口 | ① 冻结涉事 Agent（malformed SQE 来源）<br>② 审计日志（A-ULP `AGENT_URING_MALFORMED`）<br>③ WARNING 告警 | 单 Agent 1 小时内累计 ≥3 次 → CRITICAL 告警 + 终止 Agent（SIGKILL） |
| 5 | `airy_cap_epoch_rollover_total`<br>（Epoch 溢出次数，16-bit Epoch 回绕） | > 0 | 单次事件 | ① CRITICAL 告警（不应发生，触发即异常）<br>② 暂停 Badge 编译（sec_d 进入只读模式）<br>③ 通知 Macro-Supervisor | 任何单次触发 → 转人工响应（P0）+ 安全团队介入 |
| 6 | `airy_audit_hash_break_total`<br>（审计哈希链断裂次数，触发 `AIRY_FAULT_AUDIT_TAMPER` 0x100B） | > 0 | 单次事件 | ① 紧急 CRITICAL 告警（多通道分发：Email + SMS）<br>② 停止审计写入（`airyctl audit pause`）<br>③ 安全团队介入通知 | 任何单次触发 → 立即转人工响应（P0）+ 密钥轮换 + 审计卷重建 |
| 7 | `airy_ring_freeze_total`<br>（Ring 冻结次数，C-S0 检查） | > 5/min | 60s 滚动窗口 | ① WARNING 告警<br>② 重启涉事 Agent（`airyctl agent restart <agent_id>`，保留未损坏消息） | 单 Agent 1 小时内重启 ≥3 次 → CRITICAL 告警 + 终止 Agent |
| 8 | `airy_fault_enforce_total`<br>（冷酷执法次数，`airy_fault_enforce()` 调用计数） | > 50/min | 60s 滚动窗口 | ① 紧急 CRITICAL 告警<br>② 终止涉事 Agent（SIGKILL，对齐 seL4 handleFault）<br>③ 通知 Macro-Supervisor | 单 Agent 触发冷酷执法 ≥1 次（FORGED）→ 立即 SIGKILL + 转人工响应（P0） |
| 9 | `airy_sec_d_wal_fsync_fail_total`<br>（sec_d WAL fsync 失败次数） | > 0 | 单次事件 | ① CRITICAL 告警<br>② 切换 sec_d 至只读模式（`airyctl sec_d mode --readonly`，停止 WAL 写入）<br>③ 触发 sec_d 两阶段恢复协议 | 持续 ≥5min 未恢复 → 转人工响应（P0）+ sec_d 进入 [DSL] 降级 |
| 10 | `airy_audit_chain_signature_fail_total`<br>（审计链签名失败次数，`airy_audit_chain_sign` 失败） | > 0 | 单次事件 | ① CRITICAL 告警<br>② 触发密钥轮换（`airyctl audit key-rotate`，由 sec_d 托管）<br>③ 暂停审计签名（保留哈希链，仅停止签名） | 持续 ≥3 次签名失败 → 紧急 CRITICAL + 转人工响应（P0）+ 安全团队介入 |

**指标采集来源**：
- 所有指标由 macro_d + audit_d 通过 Prometheus exporter 暴露（`/metrics` 端点，详见 [03-monitoring.md](03-monitoring.md)）。
- 指标命名遵循 Prometheus 约定（`airy_<子系统>_<事件>_total` / `airy_<子系统>_<指标>_p99`）。
- 指标标签：`agent_id`、`ring_id`、`severity` 等，便于按 Agent 维度聚合。

### 13.2 执行机制

Remediation 动作通过以下组件协同执行：

```
[Prometheus]                [Alertmanager]               [airy-remediation.service]
   │                            │                              │
   │ 1. scrape /metrics          │                              │
   ▼                            │                              │
[macro_d + audit_d]        │                              │
   │                            │                              │
   │ 2. 指标超阈值              │                              │
   ▼                            │                              │
[Prometheus alerting rules]     │                              │
   │                            │                              │
   │ 3. 触发 alert              │                              │
   ▼                            │                              │
[Alertmanager routing]          │                              │
   │                            │                              │
   │ 4. webhook 触发            │                              │
   ▼                            │                              │
[airy-remediation.service]─────┘                              │
   │                                                           │
   │ 5. systemd timer 调度（默认 10s 间隔）                    │
   ▼                                                           │
[remediation playbook 执行]                                    │
   ├─ airyctl agent freeze <agent_id>                          │
   ├─ airyctl agent restart <agent_id>                         │
   ├─ airyctl sec_d quota --scale 2x                           │
   ├─ airyctl sec_d mode --readonly                            │
   ├─ airyctl audit pause / audit key-rotate                    │
   └─ ...（详见 §13.1 各行"自动响应"列）                       │
```

**组件职责**：

| 组件 | 职责 | 配置位置 |
|------|------|---------|
| **Prometheus** | 采集 `/metrics`，执行 alerting rules（阈值判定） | `/etc/airy/observability/prometheus.yml` |
| **Alertmanager** | 路由告警，触发 webhook 至 `airy-remediation.service` | `/etc/airy/alert/alertmanager.yml` |
| **systemd timer `airy-remediation.timer`** | 周期调度（默认 10s），检查待执行 remediation 队列 | `/etc/systemd/system/airy-remediation.timer` |
| **`airy-remediation.service`** | 执行 remediation 动作，调用 `airyctl` CLI | `/etc/systemd/system/airy-remediation.service` |
| **macro_d** | 接收 remediation 完成回调，更新 Agent 状态 | 内置 |

**alertmanager webhook 配置示例**：

```yaml
# /etc/airy/alert/alertmanager.yml（R2 补强新增 remediation webhook）
route:
  receiver: 'remediation-webhook'
  group_by: ['alertname', 'agent_id']
  group_wait: 5s          # 5 秒静默期，避免告警风暴
  group_interval: 5m     # 同组告警 5 分钟间隔（对齐 §13.4 静默期）
  repeat_interval: 5m    # 重复告警 5 分钟间隔

receivers:
  - name: 'remediation-webhook'
    webhook_configs:
      - url: 'http://127.0.0.1:7464/remediation'   # airy-remediation.service 监听端口
        send_resolved: true
        max_alerts: 100
```

**systemd 单元配置示例**：

```ini
# /etc/systemd/system/airy-remediation.timer
[Unit]
Description=AirymaxOS Remediation Timer (R2)

[Timer]
OnBootSec=30s
OnUnitActiveSec=10s
AccuracySec=1s

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/airy-remediation.service
[Unit]
Description=AirymaxOS Remediation Service (R2)
After=network.target macro-superv.service
Requires=macro-superv.service

[Service]
Type=oneshot
ExecStart=/usr/lib/airy/airy-remediation --run-once
User=root
Group=airy-remediation
```

### 13.3 回滚机制

每个 remediation 动作必须有对应回滚命令，防止误判或自动化故障导致 Agent/daemon 异常终止。回滚操作记录到 `/var/log/airy/remediation_rollback.log`，含操作者（`airy-remediation` 或人工）、时间、原 remediation ID。

| Remediation 动作 | 回滚命令 | 回滚前置条件 |
|------------------|---------|-------------|
| ① 冻结 Agent（`airyctl agent freeze <id>`） | `airyctl agent unfreeze <id>` | 人工确认 Agent 无攻击行为 |
| ② 重启 Agent（`airyctl agent restart <id>`） | 无（重启本身不可回滚，但可恢复 Agent 状态从快照） | 需有 Agent 状态快照（详见 [06-backup-recovery.md](06-backup-recovery.md)） |
| ③ 终止 Agent（SIGKILL） | 无（终止不可回滚） | 仅在冷酷执法后由人工重新启动 Agent |
| ④ 扩大 sec_d 配额 2x（`airyctl sec_d quota --scale 2x`） | `airyctl sec_d quota --scale 0.5x`（缩回 1x） | 配额缩回前需确认 sec_d 负载已回落 |
| ⑤ 切换 sec_d 只读模式（`airyctl sec_d mode --readonly`） | `airyctl sec_d mode --readwrite` | 需确认 WAL fsync 故障已修复 |
| ⑥ 暂停审计写入（`airyctl audit pause`） | `airyctl audit resume` | 需确认审计哈希链已重建 |
| ⑦ 密钥轮换（`airyctl audit key-rotate`） | 无（密钥轮换不可回滚，旧密钥作废） | 旧密钥作废后不可恢复，需提前备份 |
| ⑧ sec_d 限流降级 | `airyctl sec_d throttle --disable`（关闭限流，恢复默认阈值） | 需确认 P99 延迟已回落至 < 100ms |

**回滚操作审计**：

```bash
# 查询历史 remediation 与回滚记录
airyctl remediation history --since 24h
airyctl remediation rollback <remediation_id> --comment "False positive: NIC firmware bug"

# remediation 历史存储于 /var/lib/airy/remediation/<YYYYMMDD>.log
# 含 remediation_id、指标、阈值、触发时间、执行命令、回滚状态
```

### 13.4 静默期（防雪崩）

为防止指标持续超阈值导致 remediation 动作雪崩触发，每个指标的 remediation 动作在触发后进入 **5 分钟静默期**，期间同指标的同 Agent 维度不再触发新 remediation（仅记录告警）。

**静默期规则**：

| 维度 | 静默期 | 说明 |
|------|--------|------|
| 同指标 + 同 Agent | 5 分钟 | 同一 Agent 同一指标在 5 分钟内只触发一次 remediation |
| 同指标 + 不同 Agent | 不静默 | 不同 Agent 独立计数，互不影响 |
| 不同指标 + 同 Agent | 不静默 | 不同指标可并行触发 remediation |
| 升级条件触发 | 立即生效 | 升级条件满足时（如 §13.1 第 1 行"1 小时内累计冻结 ≥3 次"），静默期不生效 |

**静默期状态查询**：

```bash
# 查询当前静默中的 remediation
airyctl remediation silence list

# 输出示例：
# METRIC                              AGENT_ID  SILENCE_START          REMAINING
# airy_ipc_badge_fail_total           42        2026-07-19 10:23:15   4m32s
# airy_sec_d_wal_fsync_fail_total     0(system) 2026-07-19 10:20:01   1m18s

# 手动解除静默（需人工授权）
airyctl remediation silence break --metric airy_ipc_badge_fail_total --agent-id 42
```

**静默期实现**：
- 静默状态存储于 `/run/airy/remediation_silence.json`（tmpfs，重启清空）。
- `airy-remediation.service` 每次执行前检查静默表，命中静默期则跳过该指标该 Agent 的 remediation（仅记录告警到 journal）。
- 静默期结束（5 分钟后）自动恢复，下次指标超阈值时重新触发 remediation。

**与 §6 告警抑制的关系**：
- §6 抑制的是**告警通知**（避免告警风暴），§13.4 静默的是 **remediation 动作**（避免动作雪崩）。
- 二者独立工作：告警可能被抑制但 remediation 仍执行；remediation 静默但告警仍通知。

---

## 14. 相关文档

- [03-monitoring.md](03-monitoring.md)：监控体系，告警的输入源。
- [05-incident-response.md](05-incident-response.md)：事件响应，告警的下游消费者。
- [07-systemd-integration.md](07-systemd-integration.md)：systemd 集成与 sd_notify。
- [10-devstation.md](10-devstation.md)：开发运维工作站，告警可视化与 ack。
- [../110-security/README.md](../110-security/README.md)：安全运维，安全告警处理。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
