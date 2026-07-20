Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# systemd 集成（systemd Integration）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · systemd 集成设计文档，定义 12 daemon 的 systemd 单元设计、依赖关系、资源限制与 target 聚合\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

AirymaxOS 12 个系统 daemon 统一由 systemd 管理生命周期。systemd 集成设计目标如下：

1. **统一生命周期**：所有 daemon 通过 systemd 单元启动、停止、重启，避免多套生命周期管理工具。
2. **依赖明确**：daemon 间依赖通过 systemd 原生 `Requires=`/`After=` 表达，避免隐式依赖。
3. **健康监控**：通过 `Type=notify` + `WatchdogSec=` 实现 daemon 活性检测。
4. **资源隔离**：通过 `MemoryMax=`/`CPUQuota=`/`TasksMax=` 限制每个 daemon 资源。
5. **故障恢复**：通过 `Restart=`/`RestartSec=`/`StartLimitBurst=` 控制故障恢复策略。
6. **统一启动**：通过 `airy-os.target` 聚合所有 daemon，支持一键启动 / 停止整个 AirymaxOS。

### 1.2 与 A-ULS 模块的关系

systemd 与 A-ULS（Unified Supervision）模块 macro_d 分工：

| 职责 | systemd | macro_d |
|------|---------|--------------|
| 进程启动/停止 | 主导 | 协助（通过 systemctl） |
| 进程崩溃检测 | WatchdogSec | 心跳超时 |
| 自动重启 | Restart=always | 业务级重启策略 |
| 健康分计算 | 不支持 | 支持（0-100 分） |
| 依赖管理 | 原生依赖 | 业务依赖（Agent → daemon） |
| 资源限制 | cgroup v2 | 不参与 |

systemd 提供基础的进程级监管，macro_d 提供业务级监管，两者互为补充。macro_d 自身也是 systemd 单元，由 systemd 保证其活性。

### 1.3 与技术选型的契合

- **sched_tac（不使用 sched_ext）**：通过 systemd 的 `IOSchedulingClass=`/`IOSchedulingPriority=` 控制 IO 调度，不依赖 BPF。
- **纯 C LSM**：systemd 单元的 `ProtectSystem=`/`ProtectHome=`/`PrivateDevices=` 等安全选项与 LSM 协同。
- **IRON-9 v3 [DSL] 层**：降级模式通过切换 systemd target 实现（`airy-dsl.target`）。
- **io_uring**：systemd 252+ 支持 `LimitNOFILE=` 等 io_uring 所需资源限制。

---

## 2. systemd 单元总览

### 2.1 单元清单

AirymaxOS 共有 14 个 systemd 单元（12 daemon + 1 target + 1 降级 target）：

| 单元名 | 类型 | 功能 | 启动顺序 |
|--------|------|------|----------|
| `airy-os.target` | target | 聚合所有 daemon | 最后 |
| `airy-dsl.target` | target | 降级模式聚合 | 仅 DSL 时 |
| `macro_d.service` | service | 统一监管 | 1 |
| `logger_d.service` | service | 统一日志 | 2 |
| `config_d.service` | service | 统一配置 | 2 |
| `gateway_d.service` | service | 网关 | 3 |
| `sched_d.service` | service | 调度 | 3 |
| `vfs_d.service` | service | 文件系统 | 3 |
| `net_d.service` | service | 网络 | 3 |
| `mem_d.service` | service | 内存 | 3 |
| `cogn_d.service` | service | 认知 | 4 |
| `sec_d.service` | service | 安全 | 3 |
| `audit_d.service` | service | 审计聚合 | 4 |
| `dev_d.service` | service | 设备管理 | 4 |

### 2.2 单元文件位置

- 系统级单元：`/etc/systemd/system/`（运维可修改覆盖）
- 包级单元：`/usr/lib/systemd/system/`（RPM 包安装时提供）
- drop-in 配置：`/etc/systemd/system/<unit>.d/`（局部覆盖，不修改原单元文件）

推荐通过 drop-in 方式进行本地化配置，避免 RPM 升级时覆盖。

---

## 3. macro_d.service

### 3.1 单元文件

```ini
# /usr/lib/systemd/system/macro_d.service
[Unit]
Description=AirymaxOS Unified Supervision Daemon (A-ULS)
Documentation=man:macro_d(8)
After=systemd-journald.service
Before=logger_d.service config_d.service
Requires=systemd-journald.service
PartOf=airy-os.target

[Service]
Type=notify
NotifyAccess=main
ExecStart=/usr/sbin/macro_d --config=/etc/airy/daemons/macro_d.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=1s
StartLimitBurst=3
StartLimitIntervalSec=300
WatchdogSec=10
TimeoutStartSec=30
TimeoutStopSec=15
KillMode=mixed
KillSignal=SIGTERM
SendSIGKILL=yes

# 用户与权限
User=airy
Group=airy
SupplementaryGroups=airy-monitor airy-alert airy-backup

# 资源限制
MemoryMax=512M
MemoryHigh=400M
CPUQuota=200%
TasksMax=512
LimitNOFILE=65536
LimitNPROC=512

# 安全加固
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
NoNewPrivileges=yes
CapabilityBoundingSet=CAP_SYS_PTRACE CAP_KILL CAP_SETUID CAP_SETGID
AmbientCapabilities=CAP_SYS_PTRACE CAP_KILL

# IO
IOSchedulingClass=realtime
IOSchedulingPriority=4

# 日志
SyslogIdentifier=macro_d
LogLevelMax=info

[Install]
WantedBy=airy-os.target
```

### 3.2 关键参数说明

- **Type=notify**：macro_d 启动后通过 `sd_notify("READY=1")` 通知 systemd 就绪，systemd 在 `TimeoutStartSec=30` 内未收到通知则视为启动失败。
- **Restart=always**：进程退出后总是重启（无论退出码）。
- **RestartSec=1s**：重启间隔 1 秒，避免雪崩。
- **StartLimitBurst=3 / StartLimitIntervalSec=300**：5 分钟内最多重启 3 次，超过则进入 failed 状态，由 audit_d 触发 P0 事件。
- **WatchdogSec=10**：macro_d 每 5 秒发送 `sd_notify("WATCHDOG=1")`，超过 10 秒未收到则 systemd 强制重启。
- **KillMode=mixed**：主进程收到 SIGTERM，子进程收到 SIGKILL，确保干净退出。
- **MemoryMax=512M**：硬限制，超过则 OOM kill。
- **MemoryHigh=400M**：软限制，超过则触发回收。
- **CPUQuota=200%**：最多使用 2 个 CPU。
- **TasksMax=512**：进程+线程数上限。
- **ProtectSystem=strict**：文件系统只读（除 `/var/lib/airy/` 等显式可写目录）。
- **CapabilityBoundingSet**：限制 capability 集，仅保留监管所需的 `CAP_SYS_PTRACE`/`CAP_KILL`。

### 3.3 sd_notify 集成

macro_d 在以下事件中调用 sd_notify：

| 事件 | 通知内容 | systemd 行为 |
|------|----------|--------------|
| 启动就绪 | `READY=1` | 单元状态 → active |
| 周期心跳 | `WATCHDOG=1` | 重置 watchdog 计时器 |
| 状态更新 | `STATUS=monitoring 12 daemons, 100 agents` | 显示在 systemctl status |
| 告警 | `STATUS=CRITICAL: daemon X crashed` | 单元状态保持 active |
| 严重错误 | `ERRNO=<code>` + `STOPPING=1` | 单元状态 → failed |
| 降级模式 | `STATUS=DSL mode active` | 单元状态保持 active |

---

## 4. logger_d.service

### 4.1 单元文件

```ini
# /usr/lib/systemd/system/logger_d.service
[Unit]
Description=AirymaxOS Unified Log Processing System (A-ULP)
Documentation=man:logger_d(8)
After=macro_d.service
Requires=macro_d.service
PartOf=airy-os.target

[Service]
Type=notify
NotifyAccess=main
ExecStart=/usr/sbin/logger_d --config=/etc/airy/daemons/logger_d.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=1s
StartLimitBurst=5
StartLimitIntervalSec=300
WatchdogSec=15
TimeoutStartSec=20
TimeoutStopSec=10
KillMode=control-group

User=airy
Group=airy
SupplementaryGroups=airy-alert airy-log

MemoryMax=1G
MemoryHigh=800M
CPUQuota=100%
TasksMax=256
LimitNOFILE=131072

ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/log/airy /var/lib/airy/log
NoNewPrivileges=yes
CapabilityBoundingSet=
AmbientCapabilities=

SyslogIdentifier=logger_d

[Install]
WantedBy=airy-os.target
```

### 4.2 关键差异

- **WatchdogSec=15**：日志处理可能涉及大量 IO，watchdog 周期放宽到 15 秒。
- **MemoryMax=1G**：Ring Buffer 占用较大，内存上限高于 macro_d。
- **LimitNOFILE=131072**：支持大量日志文件并发写入。
- **ReadWritePaths**：仅 `/var/log/airy` 与 `/var/lib/airy/log` 可写。
- **CapabilityBoundingSet=**（空）：logger_d 不需要任何 capability，最小权限原则。

---

## 5. config_d.service

### 5.1 单元文件

```ini
# /usr/lib/systemd/system/config_d.service
[Unit]
Description=AirymaxOS Unified Configuration Subsystem (A-UCS)
Documentation=man:config_d(8)
After=macro_d.service
Requires=macro_d.service
Before=gateway_d.service sched_d.service vfs_d.service net_d.service mem_d.service cogn_d.service sec_d.service audit_d.service dev_d.service
PartOf=airy-os.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/config_d --load --config=/etc/airy/airy.conf
ExecReload=/usr/sbin/config_d --reload
ExecStop=/bin/true

User=airy
Group=airy
SupplementaryGroups=airy-config

MemoryMax=256M
CPUQuota=50%
TasksMax=128

ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/etc/airy /var/lib/airy/config
NoNewPrivileges=yes

SyslogIdentifier=config_d

[Install]
WantedBy=airy-os.target
```

### 5.2 关键差异

- **Type=oneshot**：config_d 是一次性任务，加载配置后退出。
- **RemainAfterExit=yes**：退出后单元仍视为 active，便于其他 daemon 声明依赖。
- **Before=**：所有业务 daemon 在 config_d 之后启动，确保配置已加载。
- **ExecReload**：热更新通过 `systemctl reload config_d` 触发。
- **MemoryMax=256M**：配置加载内存占用小。

---

## 6. 业务 daemon 单元

### 6.1 通用模板

gateway_d / sched_d / vfs_d / net_d / mem_d / cogn_d / sec_d / audit_d / dev_d 共用通用模板，仅参数不同：

```ini
# /usr/lib/systemd/system/<daemon>.service
[Unit]
Description=AirymaxOS <功能> Daemon (<模块>)
Documentation=man:<daemon>(8)
After=macro_d.service logger_d.service config_d.service
Requires=macro_d.service logger_d.service config_d.service
PartOf=airy-os.target

[Service]
Type=notify
NotifyAccess=main
ExecStart=/usr/sbin/<daemon> --config=/etc/airy/daemons/<daemon>.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=2s
StartLimitBurst=3
StartLimitIntervalSec=300
WatchdogSec=<watchdog>
TimeoutStartSec=30
TimeoutStopSec=15
KillMode=mixed

User=airy
Group=airy
SupplementaryGroups=<groups>

MemoryMax=<mem>
MemoryHigh=<mem_high>
CPUQuota=<cpu>
TasksMax=<tasks>
LimitNOFILE=<nofile>

ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=<paths>
NoNewPrivileges=yes
CapabilityBoundingSet=<caps>
AmbientCapabilities=<ambient>

SyslogIdentifier=<daemon>

[Install]
WantedBy=airy-os.target
```

### 6.2 各 daemon 参数差异

| Daemon | WatchdogSec | MemoryMax | CPUQuota | TasksMax | SupplementaryGroups | CapabilityBoundingSet |
|--------|-------------|-----------|----------|----------|----------------------|----------------------|
| gateway_d | 10s | 1G | 200% | 1024 | airy-net | CAP_NET_BIND_SERVICE CAP_NET_RAW |
| sched_d | 5s | 256M | 100% | 128 | airy-sched | CAP_SYS_NICE CAP_SYS_RESOURCE |
| vfs_d | 15s | 1G | 100% | 512 | airy-vfs | CAP_DAC_OVERRIDE |
| net_d | 10s | 1G | 200% | 1024 | airy-net | CAP_NET_ADMIN CAP_NET_RAW |
| mem_d | 10s | 2G | 100% | 256 | airy-memory | (无) |
| cogn_d | 30s | 4G | 400% | 256 | airy-cogn | (无) |
| sec_d | 5s | 512M | 100% | 256 | airy-security airy-alert | CAP_MAC_ADMIN CAP_SYSLOG CAP_SETUID |
| audit_d | 15s | 1G | 200% | 512 | airy-monitor airy-backup | (无) |
| dev_d | 30s | 256M | 50% | 128 | airy-dev | CAP_SYS_ADMIN CAP_MKNOD |

### 6.3 ReadWritePaths 配置

| Daemon | ReadWritePaths |
|--------|----------------|
| gateway_d | `/var/lib/airy/gateway` |
| sched_d | `/var/lib/airy/sched` |
| vfs_d | `/var/lib/airy/vfs` `/var/lib/airy/agents` |
| net_d | `/var/lib/airy/net` |
| mem_d | `/var/lib/airy/memory` `/var/lib/airy/agents` |
| cogn_d | `/var/lib/airy/cogn` |
| sec_d | `/etc/airy/keys` `/var/log/airy` `/var/lib/airy/security` |
| audit_d | `/var/lib/airy/monitor` `/var/lib/airy/incidents` `/var/lib/airy/backup` `/var/lib/airy/alerts` |
| dev_d | `/var/lib/airy/devices` `/dev` |

### 6.4 特殊 daemon 说明

#### 6.4.1 sec_d.service

sec_d 因安全职责特殊，需额外配置：

```ini
# sec_d 额外配置（drop-in）
[Service]
# sec_d 不受 ProtectSystem 限制，需访问全系统
ProtectSystem=no
# sec_d 需要加载 LSM 模块
ExecStartPre=/sbin/modprobe airy_lsm
# sec_d 优先级最高
Nice=-10
IOSchedulingClass=realtime
IOSchedulingPriority=1
```

#### 6.4.2 audit_d.service

audit_d 是聚合层，需较高资源：

```ini
# audit_d 额外配置（drop-in）
[Service]
# audit_d 处理大量数据，IO 优先级高
IOSchedulingClass=best-effort
IOSchedulingPriority=2
# 启动时恢复监控数据
ExecStartPost=/usr/sbin/audit_d --restore
```

---

## 7. 依赖关系图

### 7.1 启动依赖图

```
                    systemd-journald.service
                              │
                              ▼
                       macro_d.service
                       │           │
              ┌────────┘           └────────┐
              ▼                              ▼
       logger_d.service        config_d.service
              │                              │
              └──────────────┬───────────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
  sec_d.service        sched_d.service         net_d.service
       │                     │                     │
       ▼                     ▼                     ▼
  vfs_d.service         mem_d.service         gateway_d.service
                             │
                             ▼
                       cogn_d.service
                             │
                             ▼
                       audit_d.service
                             │
                             ▼
                        dev_d.service
                             │
                             ▼
                      airy-os.target
```

### 7.2 依赖矩阵

| 单元 | Requires | After | Before |
|------|----------|-------|--------|
| macro_d | systemd-journald | systemd-journald | logger_d, config_d |
| logger_d | macro_d | macro_d | (业务 daemon) |
| config_d | macro_d | macro_d | (业务 daemon) |
| sec_d | macro_d, logger_d, config_d | macro_d, logger_d, config_d | vfs_d, mem_d |
| sched_d | macro_d, logger_d, config_d | macro_d, logger_d, config_d | mem_d, cogn_d |
| net_d | macro_d, logger_d, config_d | macro_d, logger_d, config_d | gateway_d |
| vfs_d | sec_d | sec_d | mem_d |
| mem_d | sec_d, sched_d | sec_d, sched_d | cogn_d |
| gateway_d | net_d | net_d | (无) |
| cogn_d | mem_d | mem_d | audit_d |
| audit_d | cogn_d | cogn_d | dev_d |
| dev_d | audit_d | audit_d | (无) |

### 7.3 PartOf 关系

所有 daemon 单元均声明 `PartOf=airy-os.target`，停止 target 时所有 daemon 一并停止：

```bash
systemctl stop airy-os.target   # 停止所有 AirymaxOS daemon
systemctl start airy-os.target  # 启动所有 AirymaxOS daemon
systemctl restart airy-os.target  # 重启所有 AirymaxOS daemon
```

---

## 8. systemd target

### 8.1 airy-os.target

```ini
# /usr/lib/systemd/system/airy-os.target
[Unit]
Description=AirymaxOS Complete Stack
Documentation=https://docs.airy-os.local/100-operations/07-systemd-integration
Requires=macro_d.service logger_d.service config_d.service gateway_d.service sched_d.service vfs_d.service net_d.service mem_d.service cogn_d.service sec_d.service audit_d.service dev_d.service
After=network-online.target
AllowIsolate=yes

[Install]
WantedBy=multi-user.target
```

启用方式：

```bash
systemctl enable airy-os.target
systemctl start airy-os.target
```

### 8.2 airy-dsl.target

降级模式专用 target，仅启动最小可运行子集：

```ini
# /usr/lib/systemd/system/airy-dsl.target
[Unit]
Description=AirymaxOS Degraded Service Layer (DSL)
Requires=macro_d.service logger_d.service config_d.service sec_d.service audit_d.service mem_d.service
Conflicts=airy-os.target
AllowIsolate=yes

[Install]
WantedBy=multi-user.target
```

切换到降级模式：

```bash
systemctl isolate airy-dsl.target
```

退出降级模式：

```bash
systemctl isolate airy-os.target
```

### 8.3 target 切换语义

- `airy-os.target` ↔ `airy-dsl.target` 互斥（`Conflicts=`）。
- 切换时 systemd 自动停止不在新 target 中的单元，启动新 target 中的单元。
- macro_d 监听 target 切换事件，触发相应的业务级降级/恢复逻辑。

---

## 9. 资源限制

### 9.1 资源限制总览

| Daemon | MemoryMax | MemoryHigh | CPUQuota | TasksMax | IO 优先级 |
|--------|-----------|------------|----------|----------|-----------|
| macro_d | 512M | 400M | 200% | 512 | realtime:4 |
| logger_d | 1G | 800M | 100% | 256 | best-effort:2 |
| config_d | 256M | 200M | 50% | 128 | best-effort:4 |
| gateway_d | 1G | 800M | 200% | 1024 | best-effort:3 |
| sched_d | 256M | 200M | 100% | 128 | realtime:3 |
| vfs_d | 1G | 800M | 100% | 512 | best-effort:2 |
| net_d | 1G | 800M | 200% | 1024 | best-effort:2 |
| mem_d | 2G | 1.6G | 100% | 256 | best-effort:3 |
| cogn_d | 4G | 3.2G | 400% | 256 | best-effort:3 |
| sec_d | 512M | 400M | 100% | 256 | realtime:1 |
| audit_d | 1G | 800M | 200% | 512 | best-effort:2 |
| dev_d | 256M | 200M | 50% | 128 | best-effort:4 |
| **总计** | **13G** | **10.4G** | **2050%** | **4864** | - |

### 9.2 资源限制策略

- **MemoryMax**：硬限制，超过则 OOM kill（由内核触发）。
- **MemoryHigh**：软限制，超过则触发内存回收（slow down）。
- **CPUQuota**：CPU 时间配额，`100%` = 1 CPU。
- **TasksMax**：进程+线程数上限，防止 fork 炸弹。
- **IO 优先级**：realtime 优先级用于关键 daemon（macro_d/sched_d/sec_d），best-effort 用于其他。

### 9.3 资源限制调整

运维可通过 drop-in 调整资源限制，无需修改原单元文件：

```bash
# 调整 cogn_d 内存上限到 8G
mkdir -p /etc/systemd/system/cogn_d.service.d
cat > /etc/systemd/system/cogn_d.service.d/memory.conf << 'EOF'
[Service]
MemoryMax=8G
MemoryHigh=6G
EOF

systemctl daemon-reload
systemctl restart cogn_d.service
```

### 9.4 OOM 处理

当 daemon 触发 OOM：

1. systemd 默认行为：杀死超出 MemoryMax 的进程。
2. AirymaxOS 配置 `OOMPolicy=stop`：单元内任何进程 OOM 则停止整个单元。
3. macro_d 监听 OOM 事件（通过 `sd_notify("ERRNO=...")` 或 journald），触发 A-ULS 重启策略。
4. 5 分钟内 OOM ≥3 次则触发 P0 事件，进入 DSL 模式。

---

## 10. 故障恢复

### 10.1 重启策略矩阵

| Daemon | Restart | RestartSec | StartLimitBurst | StartLimitIntervalSec | 失败后行为 |
|--------|---------|------------|-----------------|----------------------|-----------|
| macro_d | always | 1s | 3 | 300s | DSL 模式 |
| logger_d | always | 1s | 5 | 300s | DSL 模式 |
| config_d | on-failure | 5s | 3 | 300s | DSL 模式 |
| gateway_d | always | 2s | 3 | 300s | 通知运维 |
| sched_d | always | 1s | 2 | 300s | kernel panic 兜底 |
| vfs_d | always | 2s | 3 | 300s | DSL 模式 |
| net_d | always | 2s | 3 | 300s | 通知运维 |
| mem_d | always | 2s | 3 | 300s | DSL 模式 |
| cogn_d | always | 5s | 5 | 300s | 通知运维 |
| sec_d | always | 1s | 2 | 300s | DSL 模式 |
| audit_d | always | 2s | 3 | 300s | 降级日志 |
| dev_d | on-failure | 5s | 3 | 300s | 通知运维 |

### 10.2 OnFailure 级联

通过 `OnFailure=` 实现级联通知：

```ini
# 各 daemon 单元中的 OnFailure 配置
[Unit]
OnFailure=airy-incident@%n.service
```

`airy-incident@.service` 是模板单元，接收失败单元名作为参数，触发 audit_d 创建事件：

```ini
# /usr/lib/systemd/system/airy-incident@.service
[Unit]
Description=Handle failure of %i
After=%i

[Service]
Type=oneshot
ExecStart=/usr/sbin/airyctl incident create --source systemd --unit %i --status failed
User=airy
Group=airy
```

### 10.3 故障注入测试

AirymaxOS 提供 `airyctl fault inject` 命令用于故障注入测试：

```bash
# 模拟 macro_d 崩溃
airyctl fault inject --target macro_d --action crash

# 模拟 daemon OOM
airyctl fault inject --target cogn_d --action oom --size 5G

# 模拟网络中断
airyctl fault inject --target net_d --action network-down
```

详见 [10-devstation.md](10-devstation.md) 的"故障注入面板"。

---

## 11. 日志集成

### 11.1 journald 集成

所有 daemon 的日志默认写入 journald，由 systemd-journald 统一管理：

- `SyslogIdentifier=<daemon>`：日志标识符。
- `LogLevelMax=info`：日志级别上限（DEBUG 日志不写入 journal）。
- `StandardOutput=journal`：stdout 重定向到 journal。
- `StandardError=journal`：stderr 重定向到 journal。

### 11.2 日志轮转

journald 配置 `/etc/systemd/journald.conf`：

```ini
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=10G
SystemKeepFree=5G
MaxFileSec=30day
ForwardToSyslog=no
```

- 总磁盘占用上限 10G。
- 保留 30 天。
- 启用压缩。

### 11.3 与 logger_d 的关系

logger_d 接收所有 daemon 的结构化日志（通过 io_uring），journald 接收非结构化的 stdout/stderr。两者分工：

| 维度 | logger_d | journald |
|------|---------------|----------|
| 数据类型 | 结构化日志事件 | 进程 stdout/stderr |
| 用途 | 业务级日志分析 | 系统级日志归档 |
| 查询接口 | RPC + devstation | journalctl |
| 保留期 | 7-90 天（按级别） | 30 天 |

---

## 12. 配置参数

systemd 集成的关键配置参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `airy_systemd.watchdog_default_s` | 10 | 默认 watchdog 超时（秒） |
| `airy_systemd.restart_default_s` | 2 | 默认重启间隔（秒） |
| `airy_systemd.start_limit_burst` | 3 | 默认启动限制次数 |
| `airy_systemd.start_limit_interval_s` | 300 | 启动限制窗口（秒） |
| `airy_systemd.journal_max_use` | 10G | journal 磁盘上限 |
| `airy_systemd.journal_keep_free` | 5G | journal 保留可用空间 |
| `airy_systemd.oom_policy` | stop | OOM 策略 |
| `airy_systemd.auto_isolate_dsl` | true | 故障时自动切换到 DSL |

---

## 13. 安全考虑

### 13.1 最小权限

每个 daemon 仅授予必要的 capability 与文件访问权限：

- 仅 sec_d / dev_d / net_d / sched_d 拥有 capability。
- 其余 daemon 的 `CapabilityBoundingSet=` 为空。
- `NoNewPrivileges=yes` 防止 privilege escalation。
- `ProtectSystem=strict` 限制文件系统访问。

### 13.2 资源限制作为安全屏障

- `TasksMax` 防 fork 炸弹。
- `MemoryMax` 防内存耗尽攻击。
- `LimitNOFILE` 防文件描述符耗尽。

### 13.3 单元文件保护

- `/usr/lib/systemd/system/airy-*.service` 文件权限 0644，属主 `root:root`。
- 运维修改通过 drop-in（`/etc/systemd/system/<unit>.d/`），不修改原文件。
- sec_d 监控单元文件目录，非授权修改触发 CRITICAL 告警。

---

## 14. 与其他子系统的协作

### 14.1 与监控体系

systemd 的 `WatchdogSec` 与监控体系的心跳检测互补，详见 [03-monitoring.md](03-monitoring.md) 第 2.1 节。

### 14.2 与告警体系

systemd 单元失败通过 `OnFailure=` 触发告警，详见 [04-alerting.md](04-alerting.md) 第 3.1 节。

### 14.3 与事件响应

systemd 的重启策略与事件响应的自动重启互补，详见 [05-incident-response.md](05-incident-response.md) 第 4.1 节。

### 14.4 与备份恢复

恢复流程通过 systemd 单元启停 daemon，详见 [06-backup-recovery.md](06-backup-recovery.md) 第 6 节。

---

## 15. 版本演进

### 15.1 v0.1.1 → v1.0.1 变更

- 新增 `airy-dsl.target`，支持降级模式快速切换。
- 所有 daemon 增加 `MemoryHigh` 软限制，提前触发回收。
- sec_d 单元改为 `ProtectSystem=no`，满足全系统监控需求。
- 新增 `OnFailure=airy-incident@%n.service` 级联机制。

### 15.2 后续规划（v1.1.0）

- 支持 systemd portable services（便携式单元），便于跨节点部署。
- 引入 systemd-cryptsetup 加密 /var/lib/airy/。
- 集成 systemd-homed 管理运维用户。

---

## 16. 相关文档

- [03-monitoring.md](03-monitoring.md)：监控体系，watchdog 与心跳互补。
- [04-alerting.md](04-alerting.md)：告警体系，OnFailure 级联。
- [05-incident-response.md](05-incident-response.md)：事件响应，重启策略。
- [06-backup-recovery.md](06-backup-recovery.md)：备份恢复，daemon 启停。
- [10-devstation.md](10-devstation.md)：开发运维工作站，systemctl 可视化。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
