Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 开发运维工作站（DevStation）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · 开发运维工作站设计文档，定义 devstation 的功能、界面、集成与安装方式\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

devstation 是 AirymaxOS 专属的开发运维工具，为开发与运维人员提供统一的可视化操作界面。设计目标如下：

1. **一站式**：集成监控、日志、性能分析、Agent 管理、事件响应五大功能。
2. **实时性**：监控与日志面板支持实时刷新（<1s 延迟）。
3. **低开销**：devstation 运行开销 <50MB 内存，<1% CPU。
4. **多形态**：TUI（终端界面）为主，Web UI（浏览器界面）为辅。
5. **可扩展**：支持插件机制，第三方可扩展功能。
6. **远程访问**：支持通过 SSH 隧道或 mTLS RPC 远程连接到 AirymaxOS 节点。

### 1.2 devstation 定位

devstation 是 agentrt-linux（AirymaxOS）的开发运维工作站，与 12 daemon 的关系：

- **消费者**：消费 macro_d、logger_d、audit_d 提供的监控与日志数据。
- **操作者**：通过 airyctl 命令操作 Agent、记忆、配置等。
- **不参与**：devstation 不参与系统运行时（不承载任何 daemon 角色），可随时启停而不影响 AirymaxOS。

### 1.3 与其他子系统的关系

| 子系统 | 在 devstation 中的角色 |
|--------|------------------------|
| macro_d | 提供 daemon 与 Agent 状态数据 |
| logger_d | 提供实时日志流与历史日志 |
| audit_d | 提供监控指标、告警、事件数据 |
| sched_d | 提供调度性能数据，接收调优指令 |
| mem_d | 提供记忆层级数据，接收记忆操作 |
| config_d | 提供配置查看与修改接口 |
| sec_d | 提供 mTLS 证书与权限校验 |
| cogn_d | 提供 Token 预算与消耗数据 |

### 1.4 与技术选型的契合

- **IORING_OP_URING_CMD**：devstation 通过 io_uring 订阅 logger_d 实时日志流，避免轮询。
- **纯 C LSM**：devstation 的所有操作受 LSM 控制，无特权操作。
- **IRON-9 v3 [IND] 层**：记忆检索复用 [IND] 层索引，支持快速搜索。
- **A-ULS 模块**：devstation 通过 macro_d RPC 接口操作 Agent，与 A-ULS 联动。

---

## 2. 功能总览

### 2.1 功能矩阵

| 功能模块 | TUI | Web UI | 说明 |
|----------|-----|--------|------|
| 实时监控面板 | ✓ | ✓ | 12 daemon 状态、Agent 列表、系统资源 |
| 日志查看器 | ✓ | ✓ | Ring Buffer 实时流、历史检索 |
| 性能分析器 | ✓ | ✓ | perf 集成、ftrace 集成、火焰图 |
| Agent 管理器 | ✓ | ✓ | 创建/启动/停止/迁移/升级 Agent |
| 告警中心 | ✓ | ✓ | 实时告警、历史告警、ack |
| 事件中心 | ✓ | ✓ | 事件列表、详情、响应、复盘 |
| 记忆管理器 | ✓ | ✓ | L1-L4 状态、defrag、迁移、擦除 |
| 备份恢复 | ✓ | ✓ | 备份列表、恢复触发、演练 |
| 配置管理器 | ✓ | ✓ | 配置查看、修改、热更新 |
| Playbook 编辑器 | ✓ | ✓ | 事件响应 Playbook 编辑与测试 |
| 故障注入面板 | ✓ | ✓ | 故障注入测试 |

### 2.2 功能优先级

| 优先级 | 功能 | 实现版本 |
|--------|------|----------|
| P0 | 实时监控面板、日志查看器、Agent 管理器 | v1.0.0 |
| P0 | 告警中心、事件中心 | v1.0.0 |
| P1 | 性能分析器、记忆管理器 | v1.0.1 |
| P1 | 备份恢复、配置管理器 | v1.0.1 |
| P2 | Playbook 编辑器、故障注入面板 | v1.1.0 |
| P2 | Web UI | v1.1.0 |

---

## 3. 界面设计

### 3.1 TUI 界面

TUI 基于 ncurses 实现，支持终端环境下的全功能操作。

#### 3.1.1 主界面布局

```
┌─────────────────────────────────────────────────────────────────────────┐
│ AirymaxOS DevStation v1.0.1          Node: node-1   User: alice         │
├─────────────────────────────────────────────────────────────────────────┤
│ [1]Monitor [2]Logs [3]Perf [4]Agents [5]Alerts [6]Incidents [7]Memory   │
│ [8]Backup [9]Config [0]Playbook [F1]Help [F10]Quit                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─ Daemons ──────────────┐  ┌─ Agents (top 10 CPU) ──────────────────┐ │
│  │ macro_d  ●  3.2%  │  │ ID  Name              CPU%   Mem       │ │
│  │ logger_d ●  1.8%  │  │ 42  customer-bot      45.2   256MB     │ │
│  │ config_d ●  0.0%  │  │ 43  support-bot       38.1   512MB     │ │
│  │ gateway_d     ●  5.5%  │  │ 44  research-bot      22.7   1.2GB     │ │
│  │ sched_d       ●  0.8%  │  │ ...                                              │ │
│  │ vfs_d         ●  2.1%  │  └────────────────────────────────────────┘ │
│  │ net_d         ●  3.0%  │  ┌─ System Resources ─────────────────────┐ │
│  │ mem_d         ●  4.2%  │  │ CPU: 62%   Mem: 18.4/32GB   Disk: 45% │ │
│  │ cogn_d        ●  8.5%  │  │ IPC p99: 2.3ms  Sched p99: 1.8ms      │ │
│  │ sec_d         ●  1.2%  │  └────────────────────────────────────────┘ │
│  │ audit_d       ●  2.0%  │  ┌─ Recent Alerts ────────────────────────┐ │
│  │ dev_d         ●  0.5%  │  │ CRIT  cogn_d OOM risk (5min ago)       │ │
│  └────────────────────────┘  │ WARN  agent_43 health 65 (12min ago)   │ │
│                              └────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│ Status: Connected to macro_d   Refresh: 1s   Local time: 14:23:11  │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 交互方式

- **数字键 1-9, 0**：切换主功能面板。
- **方向键**：在列表中导航。
- **Enter**：查看详情。
- **空格**：刷新当前面板。
- **a**：在告警面板 ack 告警。
- **r**：在 Agent 面板重启 Agent。
- **/**：搜索/过滤。
- **?**：上下文帮助。
- **F1**：全局帮助。
- **F10** / **q**：退出。

#### 3.1.3 配色方案

- 绿色：正常状态（health_score ≥80，daemon alive）。
- 黄色：警告状态（health_score 50-79，WARNING 告警）。
- 红色：严重状态（health_score <50，CRITICAL 告警）。
- 蓝色：信息提示。
- 灰色：禁用/暂停状态。

### 3.2 Web UI 界面

Web UI 为可选组件，基于内嵌 HTTP 服务器实现。

#### 3.2.1 技术栈

- 后端：C 语言内嵌 HTTP 服务器（libmicrohttpd）。
- 前端：原生 HTML + CSS + JavaScript（无外部依赖，支持离线）。
- 通信：WebSocket 实时推送 + REST API 查询。
- 认证：Basic Auth + mTLS（可选）。

#### 3.2.2 部署方式

```bash
# 启动 Web UI
airyctl devstation web --port 8443 --tls-cert /etc/airy/keys/devstation.pem

# 通过浏览器访问
https://node-1:8443/
```

Web UI 适合远程访问与多人协作场景，TUI 适合终端操作与轻量场景。

---

## 4. 实时监控面板

### 4.1 功能概述

实时监控面板展示 AirymaxOS 全系统状态，数据来源：

- **daemon 状态**：通过 RPC 从 macro_d 获取。
- **Agent 列表**：通过 RPC 从 macro_d 获取。
- **系统资源**：读取 `/sys/airy/monitor/system/**`。
- **告警**：订阅 `/dev/airy_alert` 实时流。

### 4.2 子面板

#### 4.2.1 Daemons 子面板

展示 12 daemon 的实时状态：

| 字段 | 说明 | 数据源 |
|------|------|--------|
| 名称 | daemon 名称 | macro_d |
| 状态 | ●(alive)/○(dead) | macro_d |
| CPU% | CPU 占用率 | `/sys/airy/monitor/daemons/<name>/cpu_pct` |
| 内存 | RSS 占用 | `/sys/airy/monitor/daemons/<name>/rss_bytes` |
| 健康分 | 0-100 | `/sys/airy/monitor/daemons/<name>/health_score` |
| 重启次数 | 累计重启 | `/sys/airy/monitor/daemons/<name>/restart_count` |
| 心跳 | 距上次心跳 ms | `/sys/airy/monitor/daemons/<name>/last_heartbeat_ms` |

支持点击 daemon 查看详情（调度参数、依赖关系、最近日志）。

#### 4.2.2 Agents 子面板

展示 Agent 列表，支持排序与过滤：

- **排序**：按 CPU、内存、Token 消耗、健康分、ID 排序。
- **过滤**：按状态（running/paused/stopped）、所有者、模板过滤。
- **搜索**：按名称或 ID 搜索。
- **Top N**：默认显示 Top 10，可切换到全量。

每个 Agent 显示：ID、名称、状态、CPU%、内存、Token 速率、健康分。

#### 4.2.3 System Resources 子面板

展示系统级资源：

- **CPU**：总体利用率、per-CPU 利用率、运行队列长度。
- **内存**：RSS、Cache、Swap、Slab、L3/L4 占用。
- **磁盘**：各挂载点使用率、IOPS、队列深度。
- **网络**：各 NIC 收发速率、重传率。
- **IPC**：平均延迟、p99 延迟、Ring Buffer 占用率。
- **调度**：调度延迟 p50/p99/p999、DL miss 计数。

#### 4.2.4 Recent Alerts 子面板

展示最近 10 条告警，按时间倒序：

- 级别（CRITICAL/WARNING/INFO）。
- 来源（daemon 名或 Agent ID）。
- 消息摘要。
- 时间（相对时间，如 "5min ago"）。

点击告警跳转到告警中心。

### 4.3 刷新策略

- **默认刷新**：1 秒一次。
- **手动刷新**：按空格立即刷新。
- **暂停刷新**：按 `p` 暂停，便于查看快照。
- **自适应刷新**：终端窗口失焦时降为 5 秒一次（节省 CPU）。

---

## 5. 日志查看器

### 5.1 功能概述

日志查看器展示来自 logger_d 的实时日志流与历史日志：

- **实时流**：订阅 logger_d Ring Buffer，<100ms 延迟展示新日志。
- **历史检索**：查询持久化日志（`/var/log/airy/`），支持时间范围、级别、关键词过滤。

### 5.2 实时流界面

```
┌─ Logs (Real-time) ──────────────────────────────────────────────────────┐
│ Filter: level>=WARN | Press '/' to filter, 'f' to follow                 │
├─────────────────────────────────────────────────────────────────────────┤
│ 14:23:11.123  CRIT  macro_d  daemon gateway_d heartbeat timeout     │
│ 14:23:11.234  WARN  gateway_d     connection refused from 10.0.0.5       │
│ 14:23:12.001  INFO  agent_42       User query received (req-abc123)      │
│ 14:23:12.456  WARN  sched_d        DL miss: agent_43 deadline exceeded   │
│ 14:23:13.789  ERROR agent_43       Inference failed: model timeout       │
│ 14:23:14.012  INFO  cogn_d         fallback to gpt-3.5 for agent_43      │
│ ...                                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│ Showing 6 of 1234 entries | Follow: ON | Buffer: 45%                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 过滤与搜索

- **级别过滤**：`level>=WARN` 显示 WARN 及以上。
- **来源过滤**：`source=macro_d` 仅显示 macro_d 日志。
- **Agent 过滤**：`agent_id=42` 仅显示 Agent 42 的日志。
- **关键词搜索**：`grep "timeout"` 显示包含 "timeout" 的日志。
- **时间范围**：`since=1h` 仅显示最近 1 小时。
- **正则搜索**：`regex="error.*timeout"` 支持正则表达式。

### 5.4 历史检索

历史检索基于持久化日志文件：

```bash
# 在 devstation 中按 '/' 进入搜索模式
> since=2026-07-18T14:00:00Z until=2026-07-18T15:00:00Z level=ERROR grep="OOM"
```

检索结果按时间排序，支持上下翻页与详情查看。

### 5.5 日志详情

点击单条日志查看详情：

```
┌─ Log Detail ────────────────────────────────────────────────────────────┐
│ Timestamp: 2026-07-18T14:23:11.123456789Z                               │
│ Level:     CRITICAL                                                     │
│ Source:    macro_d (PID 1234)                                      │
│ Agent:     N/A                                                          │
│ Module:    heartbeat_monitor                                            │
│ Message:   daemon gateway_d heartbeat timeout                           │
│ Labels:                                                                  │
│   daemon: gateway_d                                                     │
│   timeout_ms: 8000                                                      │
│   last_heartbeat: 2026-07-18T14:23:03.000Z                              │
│ Related Alerts: ALT-20260718-001                                        │
│ Related Incident: INC-20260718-001                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 性能分析器

### 6.1 功能概述

性能分析器集成 perf 与 ftrace，提供系统级与 Agent 级性能分析能力。

### 6.2 perf 集成

devstation 封装 perf 命令，提供友好界面：

- **CPU profile**：`perf record -a -g` 采集 CPU 热点。
- **火焰图**：将 perf 数据转换为火焰图（SVG）。
- **缓存命中**：`perf stat -e cache-misses,cache-references`。
- **上下文切换**：`perf stat -e context-switches`。

```
┌─ Perf Profile (agent_42, 30s) ──────────────────────────────────────────┐
│                                                                          │
│   ▼ 100%  agent_42::main                                                │
│     ▼ 78%   inference_engine::forward                                   │
│       ▼ 45%    transformer::attention                                   │
│         ▼ 30%      matmul                                               │
│         ▼ 15%      softmax                                              │
│       ▼ 20%    transformer::ffn                                         │
│       ▼ 13%    tokenizer::encode                                        │
│     ▼ 15%   memory::l1_fetch                                            │
│     ▼ 7%    ipc::send                                                   │
│                                                                          │
│ [F5] Refresh  [E] Export SVG  [D] Drill down                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 ftrace 集成

devstation 封装 ftrace，提供 tracepoint 可视化：

- **调度 trace**：`sched:sched_switch`、`sched:sched_wakeup`。
- **IPC trace**：`syscalls:sys_enter_io_uring_enter`。
- **内存 trace**：`kmem:kmalloc`、`kmem:kfree`。
- **自定义 trace**：用户选择 tracepoint 组合。

```
┌─ Ftrace (sched, 10s) ───────────────────────────────────────────────────┐
│ Time         CPU  Task              State      Duration                 │
│ 14:23:11.001  3   agent_42          RUNNING -> BLOCKED    1.2ms         │
│ 14:23:11.002  3   sched_d           BLOCKED -> RUNNING    0.8ms         │
│ 14:23:11.003  3   agent_43          SWAPPING -> RUNNING   5.1ms         │
│ ...                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.4 性能基线对比

devstation 支持保存性能基线，便于对比：

```bash
# 保存当前性能基线
airyctl devstation perf baseline save --name "before-tuning"

# 执行调优后，对比
airyctl devstation perf baseline compare --name "before-tuning"
```

对比结果以表格展示，突出差异显著的指标。

---

## 7. Agent 管理器

### 7.1 功能概述

Agent 管理器提供 Agent 全生命周期可视化操作，详见 [08-agent-operations.md](08-agent-operations.md)。

### 7.2 界面

```
┌─ Agents ────────────────────────────────────────────────────────────────┐
│ [N]ew  [S]tart  [P]ause  [R]estart  [M]igrate  [U]pgrade  [T]erminate   │
├────┬──────────────────┬──────────┬───────┬────────┬────────┬───────────┤
│ ID │ Name             │ State    │ CPU%  │ Mem    │ Health  │ Token/min │
├────┼──────────────────┼──────────┼───────┼────────┼────────┼───────────┤
│ 42 │ customer-bot     │ RUNNING  │ 45.2  │ 256MB  │ 92      │ 1200      │
│ 43 │ support-bot      │ RUNNING  │ 38.1  │ 512MB  │ 65 ⚠    │ 800       │
│ 44 │ research-bot     │ PAUSED   │  0.0  │ 1.2GB  │ 88      │ 0         │
│ 45 │ data-analyzer    │ RUNNING  │ 22.7  │ 768MB  │ 95      │ 450       │
│ ...                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.3 操作

- **创建 Agent**：按 `N`，填写名称、模板、预算等参数。
- **启动/暂停/重启**：选中 Agent 后按对应键。
- **迁移**：按 `M`，选择目标节点，确认迁移。
- **升级**：按 `U`，选择目标版本，选择升级类型（热/滚动/灰度）。
- **终止**：按 `T`，需二次确认。

所有操作通过 RPC 调用 macro_d，由 macro_d 实际执行。

### 7.4 Agent 详情

点击 Agent 查看详情：

- 基本信息：ID、名称、模板、所有者、创建时间、运行时长。
- 调度信息：policy、runtime、deadline、period、最近 DL miss 次数。
- 资源使用：CPU 曲线、内存曲线、Token 消耗曲线。
- 记忆状态：L1-L4 占用、命中率、淘汰率。
- IPC 状态：发送/接收消息数、平均延迟、p99 延迟。
- 最近告警：与该 Agent 相关的告警。
- 最近日志：该 Agent 的最近日志。

---

## 8. 告警中心

### 8.1 功能概述

告警中心展示实时告警与历史告警，支持 ack 操作，详见 [04-alerting.md](04-alerting.md)。

### 8.2 实时告警面板

```
┌─ Alerts (Real-time) ────────────────────────────────────────────────────┐
│ [A]ck  [F]ilter  [H]istory                                              │
├──────────┬────────┬─────────────────┬──────────┬────────────────────────┤
│ Severity │ Source │ Code            │ Time     │ Message                │
├──────────┼────────┼─────────────────┼──────────┼────────────────────────┤
│ CRIT     │ mem_d  │ 3001            │ 5min ago │ OOM risk: available<5% │
│ WARN     │ sched  │ 2003            │ 12min    │ DL miss >10/min        │
│ CRIT     │ sec_d  │ 5001            │ 18min    │ Unauthorized key access│
│ ...                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.3 告警 ack

按 `A` ack 选中告警，弹出确认框：

```
┌─ Acknowledge Alert ─────────────────────────────────────────────────────┐
│ Alert ID: ALT-20260718-001                                              │
│ Severity: CRITICAL                                                      │
│ Message:  mem_d OOM risk: available<5%                                  │
│                                                                          │
│ Comment: [Manual intervention: freed L2 cache_____________________]      │
│                                                                          │
│ [Enter] Confirm  [Esc] Cancel                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.4 历史告警查询

支持按时间范围、级别、来源、告警码查询历史告警。

---

## 9. 事件中心

### 9.1 功能概述

事件中心展示事件列表与详情，支持响应操作，详见 [05-incident-response.md](05-incident-response.md)。

### 9.2 事件列表

```
┌─ Incidents ─────────────────────────────────────────────────────────────┐
│ [N]ew  [R]espond  [V]Resolve  [P]ostmortem                              │
├────────────┬──────┬────────────┬──────────┬──────────┬─────────────────┤
│ ID         │ Sev  │ Category   │ Status   │ Time     │ Title           │
├────────────┼──────┼────────────┼──────────┼──────────┼─────────────────┤
│ INC-...001 │ P0   │ system     │ RESOLVED │ 2h ago   │ macro crash loop│
│ INC-...002 │ P1   │ agent      │ RESPOND  │ 30min    │ agent_43 OOM    │
│ INC-...003 │ P2   │ network    │ TRIAGED  │ 5min     │ NIC retrans high│
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 事件详情

点击事件查看详情：

- 基本信息：ID、级别、类别、状态、时间。
- 时间线：检测、分类、响应、恢复各阶段时间戳与动作。
- 关联告警：列出触发该事件的所有告警。
- 影响范围：受影响的 Agent 与 daemon。
- 响应动作：已执行的自动/人工响应动作。
- 复盘报告：如已完成复盘，显示报告链接。

### 9.4 事件响应

按 `R` 触发响应动作：

```
┌─ Respond to Incident INC-...002 ────────────────────────────────────────┐
│ Available actions:                                                       │
│   [1] agent_restart (agent_43)                                          │
│   [2] agent_migrate (agent_43 → node-2)                                 │
│   [3] mem_evict_l1l2 (agent_43)                                         │
│   [4] custom action                                                     │
│                                                                          │
│ Selection: [_]                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 记忆管理器

### 10.1 功能概述

记忆管理器展示 L1-L4 记忆状态，支持 defrag、迁移、擦除操作，详见 [09-memory-operations.md](09-memory-operations.md)。

### 10.2 记忆总览

```
┌─ Memory Overview ───────────────────────────────────────────────────────┐
│ Agent ID: [42]  Name: customer-bot                                       │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer  │ Used       │ Capacity    │ Hit Rate │ Evict/min │ Status        │
├────────┼────────────┼─────────────┼──────────┼───────────┼───────────────┤
│ L1     │ 234MB      │ 256MB  91%  │ 87%      │ 12        │ ⚠ High usage  │
│ L2     │ 680MB      │ 1024MB 66%  │ 72%      │ 3         │ OK            │
│ L3     │ 4.2GB      │ 10GB   42%  │ 45%      │ -         │ OK            │
│ L4     │ 180MB      │ 1GB    18%  │ 8%       │ -         │ OK            │
├─────────────────────────────────────────────────────────────────────────┤
│ [D]efrag  [M]igrate  [E]rase  [S]earch  [T]ransfer                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.3 记忆操作

- **defrag**：按 `D`，触发 L3 碎片整理。
- **migrate**：按 `M`，将记忆迁移到其他 Agent。
- **erase**：按 `E`，安全擦除记忆（需二次确认）。
- **search**：按 `S`，搜索记忆内容。
- **transfer**：按 `T`，转移记忆到其他 Agent。

---

## 11. 备份恢复

### 11.1 功能概述

备份恢复面板展示备份列表，支持恢复触发与演练，详见 [06-backup-recovery.md](06-backup-recovery.md)。

### 11.2 备份列表

```
┌─ Backups ───────────────────────────────────────────────────────────────┐
│ [L]ist  [R]ecover  [V]erify  [D]rill                                    │
├──────────────┬────────────┬─────────┬──────────┬──────────┬─────────────┤
│ Backup ID    │ Type       │ Size    │ Created  │ Verified  │ Status      │
├──────────────┼────────────┼─────────┼──────────┼──────────┼─────────────┤
│ FULL-...001  │ full       │ 5.2GB   │ 01:00    │ 02:00     │ ✓ OK        │
│ INCR-...012  │ incremental│ 480MB   │ 12:00    │ -         │ -           │
│ INCR-...013  │ incremental│ 520MB   │ 13:00    │ -         │ -           │
│ ...                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.3 恢复操作

选中备份后按 `R`：

```
┌─ Recover from Backup ───────────────────────────────────────────────────┐
│ Backup ID: FULL-20260718-001                                            │
│ Created:  2026-07-18 01:00:00                                           │
│ Size:     5.2GB                                                         │
│ Verified: ✓                                                             │
│                                                                          │
│ Options:                                                                 │
│   [x] Verify integrity before recovery                                  │
│   [x] Enter DSL mode during recovery                                    │
│   [ ] Restore only specific agents                                      │
│                                                                          │
│ [Enter] Confirm  [Esc] Cancel                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 12. 配置管理器

### 12.1 功能概述

配置管理器展示 AirymaxOS 配置，支持在线查看与热更新。

### 12.2 配置树

```
┌─ Configuration ─────────────────────────────────────────────────────────┐
│ /etc/airy/                                                               │
├── airy.conf                                                              │
├── daemons/                                                               │
│   ├── macro_d.conf                                                  │
│   ├── logger_d.conf                                                 │
│   └── ...                                                                │
├── monitor/                                                               │
│   └── thresholds.yaml                                                    │
├── alert/                                                                 │
│   ├── channels.conf                                                      │
│   └── suppression.yaml                                                   │
└── ...                                                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 12.3 配置编辑

选中配置文件后按 `E` 编辑（调用 `$EDITOR`）。保存后提示：

```
┌─ Apply Configuration ───────────────────────────────────────────────────┐
│ File: /etc/airy/monitor/thresholds.yaml                                 │
│ Changes: 3 lines modified                                               │
│                                                                          │
│ [V]alidate  [A]pply (hot reload)  [S]chedule (maintenance window)       │
│ [Esc] Cancel                                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

热更新通过 config_d 触发 SIGHUP 至相关 daemon。

---

## 13. Playbook 编辑器

### 13.1 功能概述

Playbook 编辑器用于编辑事件响应 Playbook，详见 [05-incident-response.md](05-incident-response.md) 第 7.2 节。

### 13.2 编辑界面

```
┌─ Playbook Editor: oom_recovery ─────────────────────────────────────────┐
│ name: oom_recovery                                                       │
│ trigger:                                                                 │
│   severity: P0                                                           │
│   category: resource                                                     │
│   condition: "airy_mem_available_pct < 5"                                │
│ steps:                                                                   │
│   - action: mem_evict_l1l2                                               │
│     target: all_agents                                                   │
│     on_failure: continue                                                 │
│   - action: sched_adjust_dl                                              │
│     target: low_priority_agents                                          │
│     params:                                                              │
│       runtime_ratio: 0.5                                                 │
│     on_failure: continue                                                 │
│   - action: agent_pause                                                  │
│     target: lowest_priority_agent                                        │
│     on_failure: escalate                                                 │
│                                                                          │
│ [V]alidate  [T]est  [S]ave  [D]eploy                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

支持语法校验（`V`）、模拟测试（`T`）、保存（`S`）、部署（`D`）。

---

## 14. 故障注入面板

### 14.1 功能概述

故障注入面板支持注入故障用于测试，详见 [05-incident-response.md](05-incident-response.md) 第 10.3 节。

### 14.2 故障类型

- **进程崩溃**：模拟 daemon 或 Agent 崩溃。
- **OOM**：模拟内存耗尽。
- **网络中断**：模拟 NIC down。
- **磁盘故障**：模拟磁盘只读。
- **IPC 阻塞**：模拟 IPC 队列堆积。
- **Token 耗尽**：模拟 Token 预算用尽。

### 14.3 注入界面

```
┌─ Fault Injection ───────────────────────────────────────────────────────┐
│ Target: [macro_d ▼]                                                │
│ Action: [crash          ▼]                                              │
│ Duration: [permanent    ▼]                                              │
│                                                                          │
│ Warning: This will cause service disruption.                            │
│                                                                          │
│ [Enter] Inject  [Esc] Cancel                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

所有故障注入操作记录到 audit_d，便于追溯。

---

## 15. 安装与部署

### 15.1 RPM 包安装

devstation 通过 RPM 包安装：

```bash
# 安装
sudo dnf install agentrt-devstation

# 升级
sudo dnf upgrade agentrt-devstation

# 卸载
sudo dnf remove agentrt-devstation
```

RPM 包内容：

```
/usr/bin/airyctl                 # 命令行工具
/usr/bin/devstation              # TUI 入口
/usr/lib/airy/devstation/        # 资源文件
/etc/airy/devstation.conf        # 默认配置
/usr/share/man/man1/devstation.1 # man 手册
```

### 15.2 编译安装

支持从源码编译：

```bash
git clone https://github.com/spharx/agentrt-linux
cd agentrt-linux/tools/devstation
make
sudo make install
```

依赖：ncurses、libmicrohttpd、openssl、liburing。

### 15.3 配置

devstation 配置文件 `/etc/airy/devstation.conf`：

```ini
[connection]
# 默认连接的本地节点
node = localhost
# RPC 端口
rpc_port = 7463
# mTLS 证书
mtls_cert = /etc/airy/keys/devstation.pem
mtls_key = /etc/airy/keys/devstation.key
mtls_ca = /etc/airy/keys/ca.pem

[tui]
# 默认刷新间隔（秒）
refresh_interval = 1
# 配色方案（dark/light）
color_scheme = dark
# 默认面板
default_panel = monitor

[web]
# Web UI 是否启用
enabled = false
# 监听端口
port = 8443
# TLS 证书
tls_cert = /etc/airy/keys/devstation.pem
tls_key = /etc/airy/keys/devstation.key

[permissions]
# 允许的操作（view/operate/admin）
level = view
```

### 15.4 启动

```bash
# 启动 TUI
devstation

# 连接到远程节点
devstation --node node-2 --rpc-port 7463

# 启动 Web UI
devstation --web --port 8443

# 查看版本
devstation --version
```

---

## 16. 权限模型

### 16.1 操作级别

devstation 支持三级操作权限：

| 级别 | 权限 |
|------|------|
| view | 仅查看，不可修改 |
| operate | 查看 + Agent 操作（启停/迁移）+ 告警 ack |
| admin | 全部权限，包括配置修改、故障注入、记忆擦除 |

### 16.2 权限控制

- 权限通过 sec_d 颁发的 mTLS 证书中的 `O`（Organization）字段标识。
- `O=airy-view`：view 级别。
- `O=airy-operate`：operate 级别。
- `O=airy-admin`：admin 级别。
- devstation 启动时校验证书，仅允许证书范围内的操作。

### 16.3 审计

所有 devstation 操作记录到 audit_d：

- 操作者（证书 CN）。
- 操作时间。
- 操作类型（view/operate/admin）。
- 操作内容（命令与参数）。
- 操作结果（成功/失败）。

审计日志保留 90 天，admin 操作永久保留。

---

## 17. 性能考虑

### 17.1 devstation 资源占用

| 模式 | 内存 | CPU | 网络 |
|------|------|-----|------|
| TUI（连接本地） | 30MB | <1% | 0 |
| TUI（连接远程） | 35MB | <1% | <100KB/s |
| Web UI | 80MB | <2% | <1MB/s |

### 17.2 对系统影响

- devstation 通过 RPC 读取数据，不直接访问 daemon 内部状态。
- RPC 频率：默认 1s 一次，可配置降低。
- 大规模 Agent（>1000）场景下，建议使用 Web UI + 分页查询，避免 TUI 一次性加载。

---

## 18. 配置参数

devstation 关键配置参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `devstation.refresh_interval_s` | 1 | TUI 刷新间隔（秒） |
| `devstation.color_scheme` | dark | 配色方案 |
| `devstation.default_panel` | monitor | 默认面板 |
| `devstation.web_enabled` | false | Web UI 启用 |
| `devstation.web_port` | 8443 | Web UI 端口 |
| `devstation.permission_level` | view | 默认权限级别 |
| `devstation.log_buffer_lines` | 1000 | 日志缓冲行数 |
| `devstation.alert_buffer_size` | 100 | 告警缓冲大小 |

---

## 19. 安全考虑

### 19.1 认证

- TUI 本地连接：通过 Unix socket，无需认证。
- TUI 远程连接：强制 mTLS。
- Web UI：Basic Auth + mTLS（可选）。

### 19.2 数据保护

- devstation 不在本地存储任何 AirymaxOS 数据。
- 远程连接的日志、告警等仅在内存中暂存，退出即清除。
- 配置文件中的证书路径受 sec_d 保护。

### 19.3 操作审计

- 所有 operate/admin 级别操作记录到 audit_d。
- admin 操作（如故障注入、记忆擦除）需二次确认。
- 审计日志不可篡改，sec_d 监控异常修改。

---

## 20. 与其他子系统的协作

### 20.1 与监控体系

devstation 是监控体系的主要消费者，详见 [03-monitoring.md](03-monitoring.md)。

### 20.2 与告警体系

devstation 提供告警可视化与 ack，详见 [04-alerting.md](04-alerting.md)。

### 20.3 与事件响应

devstation 提供事件响应操作界面，详见 [05-incident-response.md](05-incident-response.md)。

### 20.4 与备份恢复

devstation 提供备份恢复界面，详见 [06-backup-recovery.md](06-backup-recovery.md)。

### 20.5 与 Agent 运维

devstation 提供 Agent 管理界面，详见 [08-agent-operations.md](08-agent-operations.md)。

### 20.6 与记忆运维

devstation 提供记忆管理界面，详见 [09-memory-operations.md](09-memory-operations.md)。

---

## 21. 版本演进

### 21.1 v1.0.0 → v1.0.1 变更

- 新增性能分析器（perf + ftrace 集成）。
- 新增记忆管理器面板。
- 新增备份恢复面板。
- 新增配置管理器面板。
- TUI 性能优化，大规模 Agent 场景下渲染延迟 <50ms。

### 21.2 后续规划（v1.1.0）

- Web UI 完整实现。
- Playbook 编辑器（可视化拖拽）。
- 故障注入面板。
- 多节点统一视图（联邦查询）。
- 插件机制（第三方扩展）。

---

## 22. 相关文档

- [03-monitoring.md](03-monitoring.md)：监控体系，devstation 数据源。
- [04-alerting.md](04-alerting.md)：告警体系，devstation 告警展示。
- [05-incident-response.md](05-incident-response.md)：事件响应，devstation 事件操作。
- [06-backup-recovery.md](06-backup-recovery.md)：备份恢复，devstation 备份管理。
- [07-systemd-integration.md](07-systemd-integration.md)：systemd 集成，devstation 单元状态。
- [08-agent-operations.md](08-agent-operations.md)：Agent 运维，devstation Agent 管理。
- [09-memory-operations.md](09-memory-operations.md)：记忆运维，devstation 记忆管理。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
