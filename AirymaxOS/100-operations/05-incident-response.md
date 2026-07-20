Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 事件响应（Incident Response）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · 事件响应设计文档，定义事件分类、响应流程、自动化策略与复盘机制\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

事件响应体系是 AirymaxOS 运维分册的关键子模块，承接告警体系（详见 [04-alerting.md](04-alerting.md)）的输出，对系统异常进行结构化处置。设计目标如下：

1. **快速响应**：P0 事件从检测到自动响应 ≤10 秒，P1 事件 ≤60 秒。
2. **分级处置**：按事件严重程度（P0/P1/P2/P3）采取不同响应策略，避免过度干预。
3. **自动化优先**：能自动恢复的不依赖人工，自动降级、自动重启、自动隔离为首选策略。
4. **可追溯**：所有事件响应动作完整记录，支持事后复盘。
5. **安全优先**：安全类事件由 sec_d 主导处置，遵循"先隔离后恢复"原则。

### 1.2 与告警体系的关系

告警体系负责"通知"，事件响应负责"处置"。两者关系：

| 维度 | 告警体系 | 事件响应 |
|------|----------|----------|
| 输入 | 监控阈值命中 | CRITICAL/WARNING 告警 |
| 输出 | 告警事件 | 响应动作（重启、降级、隔离） |
| 主导模块 | audit_d + logger_d | macro_d + sec_d + sched_d |
| 触发模式 | 事件驱动 | 告警订阅 + 自动响应 |
| 文档 | [04-alerting.md](04-alerting.md) | 本文 |

CRITICAL 告警自动触发 P0 事件响应流程，WARNING 告警触发 P1 事件响应流程。

### 1.3 与技术选型的契合

- **sched_tac（不使用 sched_ext）**：降级运行通过调整 `SCHED_DEADLINE` 参数与 cgroup 限制实现，不依赖 sched_ext 调度器。
- **纯 C LSM**：安全隔离通过 LSM hook 实现，对违规进程施加 capability 剥夺与文件系统隔离。
- **IRON-9 v3 [DSL] 层**：降级服务层（Degraded Service Layer）提供最小可运行子集，事件触发时自动启用。
- **A-ULS 模块**：macro_d 提供统一的 daemon 重启能力，所有自动重启动作通过 A-ULS 执行。

---

## 2. 事件响应流程

### 2.1 五阶段流程

所有事件响应遵循「检测 → 分类 → 响应 → 恢复 → 复盘」五阶段流程：

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ 1. 检测  │ ─> │ 2. 分类  │ ─> │ 3. 响应  │ ─> │ 4. 恢复  │ ─> │ 5. 复盘  │
│ Detect  │    │ Triage  │    │Respond  │    │ Recover │    │ Postmortem│
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
     ▲                                                              │
     └────────────────────── 反馈循环 ──────────────────────────────┘
```

#### 2.1.1 检测（Detect）

事件检测来源：

- **监控体系**：指标超阈值自动产生 CRITICAL/WARNING 告警（详见 [03-monitoring.md](03-monitoring.md)）。
- **告警体系**：告警升级（ESCALATED 状态）自动转为 P0 事件。
- **systemd**：单元失败、WatchdogSec 超时通过 `OnFailure=` 触发 macro_d。
- **内核**：kernel panic 通过 kdump 捕获，oops 通过 `panic_on_oops` 控制。
- **人工上报**：运维人员通过 `airyctl incident create` 创建事件。

检测延迟目标：

| 来源 | 延迟目标 |
|------|----------|
| 监控阈值命中 | <100ms |
| systemd 单元失败 | <1s |
| kernel panic | <5s（kdump 完成后） |
| 人工上报 | 实时 |

#### 2.1.2 分类（Triage）

事件分类由 audit_d 自动完成，规则如下：

1. **严重程度判定**：根据告警级别、影响范围、持续时长综合判定 P0/P1/P2/P3（详见第 3 节）。
2. **类别归属**：将事件归入「系统」「Agent」「安全」「网络」「资源」五大类。
3. **根因初判**：根据根因规则库（详见 4.2.3）推断可能的根因。
4. **响应策略选择**：根据严重程度与类别选择响应策略（详见第 4 节）。

分类完成后，事件写入 `/var/lib/airy/incidents/<incident_id>.json`，并通过 RPC 通知订阅方。

#### 2.1.3 响应（Respond）

响应阶段执行具体处置动作：

- **自动响应**：macro_d / sec_d / sched_d 根据 incident 类别自动执行。
- **人工介入**：P0 事件自动响应后仍需人工确认；P1 事件自动响应失败时升级为人工。
- **响应记录**：所有响应动作（命令、参数、执行者、结果）记录到事件日志。

#### 2.1.4 恢复（Recover）

恢复阶段确认事件影响消除：

- **自动恢复**：响应策略执行成功后，audit_d 监控相关指标 5-15 分钟，确认回归正常。
- **人工确认**：运维人员通过 `airyctl incident resolve <id>` 确认恢复。
- **回归测试**：恢复后可选执行回归测试（由 devstation 触发）。

#### 2.1.5 复盘（Postmortem）

复盘在事件恢复后 24 小时内进行：

- 自动生成事件时间线（含告警、响应、恢复全过程）。
- 运维人员填写复盘报告（详见第 8 节模板）。
- 复盘报告归档到 `/var/lib/airy/incidents/postmortem/`。

### 2.2 事件状态机

事件经历以下状态：

```
[创建] ──> DETECTED ──> TRIAGED ──> RESPONDING ──> RECOVERING ──> RESOLVED
                                       │                │            │
                                       ▼                ▼            ▼
                                  [自动响应失败]   [恢复失败]   [复盘]
                                       │                │            │
                                       ▼                ▼            ▼
                                  ESCALATED       REOPENED      POSTMORTEM
                                       │                │
                                       └──────[人工介入]┘
```

状态说明：

| 状态 | 含义 |
|------|------|
| DETECTED | 事件已检测，待分类 |
| TRIAGED | 已分类，待响应 |
| RESPONDING | 响应策略执行中 |
| RECOVERING | 响应完成，恢复验证中 |
| RESOLVED | 已恢复，待复盘 |
| POSTMORTEM | 复盘完成，事件归档 |
| ESCALATED | 自动响应失败，需人工介入 |
| REOPENED | 恢复失败，事件重新打开 |

---

## 3. 事件分类

### 3.1 严重程度分级

AirymaxOS 采用 P0/P1/P2/P3 四级严重程度：

| 级别 | 含义 | 响应 SLA | 自动化程度 | 升级目标 |
|------|------|----------|------------|----------|
| P0 | 系统级故障，核心功能不可用 | 10 秒内自动响应 | 全自动 + 人工确认 | 立即通知运维 on-call |
| P1 | 重要功能受损，影响部分 Agent | 60 秒内自动响应 | 全自动 | 30 分钟内人工确认 |
| P2 | 次要功能异常，影响有限 | 5 分钟内响应 | 半自动 | 工作日内人工处理 |
| P3 | 信息性事件，无需立即处理 | 无 SLA | 仅记录 | 周报汇总 |

### 3.2 P0 事件

P0 事件代表系统级故障，包括但不限于：

#### 3.2.1 kernel panic

- **触发条件**：内核 panic_on_oops 触发，或 kdump 捕获到 panic。
- **响应策略**：
  1. kdump 自动捕获崩溃转储到 `/var/crash/`。
  2. 系统自动重启（通过 `kernel.panic` 参数控制，默认 10 秒后重启）。
  3. 重启后 macro_d 检测到非正常重启，自动创建 P0 事件。
  4. 加载 [DSL] 降级模式启动（详见 4.3）。
  5. 通知运维 on-call（Email + SMS）。

#### 3.2.2 daemon 崩溃

- **触发条件**：核心 daemon（macro_d/logger_d/config_d/sched_d/sec_d）进程崩溃。
- **响应策略**：
  1. systemd 自动重启（Restart=always）。
  2. macro_d 监管检测到重启事件，评估重启次数：
     - 5 分钟内重启 <3 次：INFO 事件，自动恢复。
     - 5 分钟内重启 ≥3 次：P0 事件，暂停自动重启，进入 [DSL] 降级模式。
  3. 通知运维 on-call。

#### 3.2.3 安全违规

- **触发条件**：sec_d 通过 LSM hook 检测到严重安全违规（如未授权访问密钥、审计日志篡改）。
- **响应策略**：
  1. sec_d 立即隔离违规进程（capability 剥夺 + cgroup 冻结）。
  2. 触发 P0 事件，通知运维 on-call 与安全团队。
  3. sec_d 启动安全审计模式，全量记录违规进程的所有操作。
  4. 涉及 Agent 时，冻结相关 Agent 并迁移其负载。
  5. 不自动恢复，必须人工确认处置。

#### 3.2.4 其他 P0 场景

- macro_d 自身崩溃：systemd 重启，依赖 audit_d 检测并创建事件。
- /var/lib 文件系统只读：触发 vfs_d CRITICAL 告警，进入 [DSL] 降级模式。
- 审计存储满：audit_d 无法写入，触发 P0 事件，暂停非关键日志。

### 3.3 P1 事件

P1 事件代表重要功能受损，包括但不限于：

#### 3.3.1 Agent 异常

- **触发条件**：Agent 健康分 <30 持续 60s，或 Agent 进程异常退出。
- **响应策略**：
  1. macro_d 自动重启 Agent（保留 Agent ID 与状态）。
  2. 重启 3 次失败：迁移 Agent 到备用节点（与 150-cloudnative 联动）。
  3. 迁移失败：标记 Agent 为"故障"，通知运维。

#### 3.3.2 IPC 超时

- **触发条件**：`airy_ipc_p99_latency_ms > 20` 持续 10s，或 io_uring 队列溢出。
- **响应策略**：
  1. audit_d 关联分析，定位阻塞的 daemon 或 Agent。
  2. macro_d 重启阻塞方（若是 daemon）或暂停 Agent（若是 Agent）。
  3. 通知运维，提供 IPC 诊断报告。

#### 3.3.3 内存不足

- **触发条件**：`airy_mem_available_pct < 10`，或 mem_d 触发 OOM 预警。
- **响应策略**：
  1. mem_d 触发 L1/L2 记忆主动淘汰，释放内存。
  2. sched_d 降低低优先级 Agent 的资源配额。
  3. 仍不足：macro_d 暂停最低优先级 Agent。
  4. 极端情况：触发 [DSL] 降级模式，仅保留核心 daemon。

#### 3.3.4 其他 P1 场景

- 调度延迟超阈值（`airy_sched_latency_p99_ms > 20` 持续 10s）。
- 网络重传率 >20% 持续 1 分钟。
- Token 预算耗尽且无 fallback 模型。
- 配置热更新失败。

### 3.4 P2/P3 事件

- **P2**：单个 Agent 健康分下降、非核心 daemon 重启、磁盘空间警告等。响应策略以通知为主，由运维在工作日内处理。
- **P3**：INFO 级告警、例行维护通知、性能波动等。仅记录到事件日志，周报汇总。

---

## 4. 响应策略

### 4.1 自动重启（A-ULS）

A-ULS 模块 macro_d 提供统一的自动重启能力，是事件响应的首选策略。

#### 4.1.1 重启策略配置

每个 daemon 与 Agent 的重启策略通过 A-UCS 配置：

```yaml
# /etc/airy/supervision/restart_policy.yaml
daemons:
  macro_d:
    restart: always
    max_restarts_5m: 3
    backoff: exponential  # 1s, 2s, 4s, 8s, 16s
    max_backoff: 30s
    on_max_restarts: dsl_mode  # 触发 DSL 降级模式
  logger_d:
    restart: always
    max_restarts_5m: 5
    backoff: linear
    on_max_restarts: dsl_mode
  sched_d:
    restart: always
    max_restarts_5m: 2
    on_max_restarts: kernel_panic  # sched_d 不可用时视为系统不可用
  # ... 其余 daemon

agents:
  default:
    restart: on-failure
    max_restarts_5m: 3
    backoff: exponential
    on_max_restarts: migrate  # 迁移到备用节点
```

#### 4.1.2 重启流程

1. **检测**：macro_d 通过 SIGCHLD 或心跳超时检测到进程异常。
2. **评估**：查询最近 5 分钟重启次数，若超过 `max_restarts_5m` 则执行 `on_max_restarts` 策略。
3. **退避**：按 `backoff` 策略等待（避免雪崩）。
4. **重启**：通过 `systemctl restart <unit>` 或内部 RPC 触发重启。
5. **验证**：等待 daemon 就绪（readiness=1），超时则视为重启失败。
6. **记录**：重启事件写入 `/var/lib/airy/incidents/restart_log.jsonl`。

#### 4.1.3 重启失败的级联

- daemon 重启失败 → 触发 P0 事件 → 进入 [DSL] 降级模式。
- Agent 重启失败 → 触发 P1 事件 → 尝试迁移到备用节点。
- 迁移失败 → 标记 Agent 故障，通知运维。

### 4.2 降级运行（[DSL]）

IRON-9 v3 模型的 [DSL]（Degraded Service Layer）层提供最小可运行子集，在系统严重故障时保证核心功能。

#### 4.2.1 降级模式触发条件

- kernel panic 后重启。
- macro_d 5 分钟内重启 ≥3 次。
- 核心 daemon（sched_d/sec_d/audit_d）连续崩溃。
- /var/lib 存储不可用。
- 系统内存 <5% 且无法通过淘汰释放。

#### 4.2.2 降级模式行为

降级模式下系统行为：

| 组件 | 降级行为 |
|------|----------|
| Agent | 暂停所有非关键 Agent，仅保留 ID=1 的管理 Agent |
| sched_d | 切换为 SCHED_FIFO 简化调度，禁用 SCHED_DEADLINE |
| cogn_d | 禁用复杂推理，仅响应 ping/health 请求 |
| mem_d | 冻结 L1/L2 记忆淘汰，仅保留 L3/L4 只读访问 |
| gateway_d | 拒绝新连接，保留现有管理连接 |
| vfs_d | 切换为只读模式，避免数据损坏 |
| net_d | 仅保留管理网络，禁用业务网络 |
| sec_d | 提升至最高警戒级别，全量审计 |
| audit_d | 仅记录 CRITICAL 事件，丢弃 INFO |
| dev_d | 禁用非必要设备 |
| logger_d | 降低日志级别至 WARN |
| config_d | 加载降级模式专用配置 `airy-dsl.conf` |

#### 4.2.3 降级模式退出

降级模式退出需满足：

1. 所有核心 daemon 健康分 ≥80 持续 5 分钟。
2. 系统资源（CPU/内存/存储）恢复正常阈值。
3. 运维人员通过 `airyctl dsl exit` 显式确认。

退出流程：

1. macro_d 逐步恢复非关键 Agent（按优先级）。
2. 各 daemon 切换回正常配置。
3. 触发 INFO 事件"降级模式退出"，记录降级持续时长。

### 4.3 安全隔离（纯 C LSM）

安全事件由 sec_d 主导处置，采用"先隔离后恢复"原则，通过纯 C LSM 实现细粒度隔离。

#### 4.3.1 隔离能力

纯 C LSM 提供以下隔离原语：

- **进程冻结**：通过 cgroup freezer 暂停违规进程。
- **capability 剥夺**：移除进程的 capabilities 集合（如 CAP_SYS_ADMIN）。
- **文件系统隔离**：将进程的文件访问重定向到沙箱目录。
- **网络隔离**：将进程移入独立 network namespace。
- **IPC 隔离**：禁止进程通过 io_uring 与其他进程通信。

#### 4.3.2 隔离流程

1. sec_d 检测到安全违规（LSM hook 触发）。
2. 评估违规严重程度：
   - 轻微（如文件访问越权）：capability 剥夺 + 警告。
   - 严重（如密钥泄露）：进程冻结 + 完整隔离。
   - 紧急（如审计篡改）：进程终止 + 全量审计。
3. 执行隔离动作，记录到 `/var/log/airy/security_action.log`。
4. 创建 P0 安全事件，通知安全团队。
5. 涉及 Agent 时，迁移 Agent 负载到新进程（保留 Agent 状态）。

#### 4.3.3 隔离解除

隔离解除必须人工确认：

```bash
airyctl security release <process_id> --reason "Verified false positive"
```

解除记录写入审计日志，包含操作者、原因、时间。

### 4.4 响应策略矩阵

| 事件类别 | P0 | P1 | P2 | P3 |
|----------|----|----|----|----|
| 系统故障 | DSL 降级 + on-call | 自动重启 | 通知 | 记录 |
| Agent 异常 | 隔离 + 迁移 | 自动重启 + 迁移 | 通知 | 记录 |
| 安全违规 | 完整隔离 + on-call | capability 剥夺 | 警告 | 记录 |
| 网络异常 | DSL 降级 | 重启 net_d | 通知 | 记录 |
| 资源不足 | DSL 降级 | 主动淘汰 + 暂停 Agent | 通知 | 记录 |
| IPC 超时 | 重启阻塞方 | 重启阻塞方 | 通知 | 记录 |
| 调度异常 | DSL 降级 | 调整 SCHED_DEADLINE | 通知 | 记录 |

---

## 5. 事件日志与保留策略

### 5.1 事件日志结构

每条事件日志采用 JSON 格式，存储于 `/var/lib/airy/incidents/<incident_id>.json`：

```json
{
  "incident_id": "INC-20260718-001",
  "severity": "P0",
  "category": "system",
  "title": "macro_d crash loop",
  "status": "POSTMORTEM",
  "detected_at": "2026-07-18T14:23:11.123456789Z",
  "triaged_at": "2026-07-18T14:23:11.234567890Z",
  "responded_at": "2026-07-18T14:23:15.000000000Z",
  "recovered_at": "2026-07-18T14:35:22.000000000Z",
  "resolved_at": "2026-07-18T15:02:00.000000000Z",
  "root_cause": "config_d pushed invalid SCHED_DEADLINE parameters",
  "timeline": [
    {
      "ts": "2026-07-18T14:23:11.123Z",
      "event": "detected",
      "actor": "audit_d",
      "detail": "macro_d heartbeat timeout (8s)"
    },
    {
      "ts": "2026-07-18T14:23:15.000Z",
      "event": "responded",
      "actor": "macro_d",
      "detail": "DSL mode activated"
    },
    {
      "ts": "2026-07-18T14:35:22.000Z",
      "event": "recovered",
      "actor": "ops_alice",
      "detail": "config rollback, macro_d health restored"
    }
  ],
  "alerts": ["ALT-20260718-001", "ALT-20260718-002"],
  "affected_agents": [42, 43, 44],
  "affected_daemons": ["macro_d"],
  "response_actions": [
    {
      "action": "dsl_activate",
      "actor": "macro_d",
      "result": "success"
    },
    {
      "action": "config_rollback",
      "actor": "ops_alice",
      "result": "success"
    }
  ],
  "postmortem": {
    "report_id": "PM-20260718-001",
    "completed_at": "2026-07-19T10:00:00Z",
    "completed_by": "ops_alice"
  }
}
```

### 5.2 保留策略

事件日志按 90 天滚动保留：

| 数据类型 | 保留期 | 存储位置 | 压缩 |
|----------|--------|----------|------|
| 活跃事件日志 | 永久（直至归档） | `/var/lib/airy/incidents/` | 无 |
| 已归档事件 | 90 天 | `/var/lib/airy/incidents/archive/` | zstd |
| 复盘报告 | 90 天 | `/var/lib/airy/incidents/postmortem/` | zstd |
| 事件相关告警 | 90 天 | `/var/lib/airy/alerts/` | zstd |
| 响应动作日志 | 90 天 | `/var/lib/airy/incidents/actions.jsonl` | zstd |
| 安全事件日志 | 1 年 | `/var/lib/airy/security/incidents/` | zstd |

超过保留期的事件由 audit_d 在每日凌晨 03:30 自动清理，清理前生成摘要写入 `/var/lib/airy/incidents/summary/<YYYYMM>.summary`。

### 5.3 日志完整性保护

- 事件日志文件权限 0640，属主 `airy:airy-incident`。
- 写入采用 append-only 模式（通过 chattr +a）。
- sec_d 监控事件日志目录，任何非 audit_d 进程的修改触发 CRITICAL 告警。
- 每日凌晨生成 SHA256 校验和，存储于 `/var/lib/airy/incidents/checksums.sha256`。

---

## 6. 与 110-security 的关系

### 6.1 安全事件由 sec_d 主导

安全类事件（如 LSM hook 触发的违规、密钥泄露、审计篡改）由 sec_d 主导处置，遵循以下原则：

1. **先隔离后恢复**：安全事件优先隔离违规方，再考虑业务恢复。
2. **不自动恢复**：所有安全事件必须人工确认才能关闭。
3. **全量审计**：处置过程本身被审计，防止处置动作被滥用。
4. **通知安全团队**：P0 安全事件同时通知运维 on-call 与安全团队。

### 6.2 安全事件分类

| 类别 | 主导模块 | 响应策略 |
|------|----------|----------|
| 进程越权 | sec_d | capability 剥夺 + 进程冻结 |
| 文件非法访问 | sec_d | 文件系统隔离 + 警告 |
| 密钥泄露 | sec_d | 进程终止 + 密钥轮换 |
| 审计篡改 | sec_d | 进程终止 + 全量审计 |
| 网络异常连接 | sec_d + net_d | 网络隔离 + 连接断开 |
| 恶意 Agent | sec_d + macro_d | Agent 终止 + 状态冻结 |

### 6.3 与 sec_d 的协作流程

```
[LSM hook 触发] ──> [sec_d 评估] ──> [隔离动作] ──> [P0 安全事件]
                          │                                  │
                          ▼                                  ▼
                  [sec_d 全量审计]                  [通知安全团队 + on-call]
                                                            │
                                                            ▼
                                                  [人工调查与处置]
                                                            │
                                                            ▼
                                                  [sec_d 解除隔离 / 终止]
                                                            │
                                                            ▼
                                                  [事件归档 + 复盘]
```

详见 [../110-security/README.md](../110-security/README.md)。

---

## 7. 自动化响应能力

### 7.1 自动响应动作清单

macro_d 支持的自动响应动作：

| 动作 | 触发场景 | 执行模块 |
|------|----------|----------|
| `daemon_restart` | daemon 崩溃 | macro_d + systemd |
| `agent_restart` | Agent 异常退出 | macro_d |
| `agent_pause` | Agent 资源超限 | macro_d + sched_d |
| `agent_migrate` | 节点故障 | macro_d + 150-cloudnative |
| `dsl_activate` | P0 系统故障 | macro_d + 全 daemon |
| `config_rollback` | 配置热更新失败 | config_d |
| `mem_evict_l1l2` | 内存不足 | mem_d |
| `sched_adjust_dl` | 调度延迟超阈值 | sched_d |
| `net_isolate` | 网络异常 | net_d |
| `sec_freeze_process` | 安全违规 | sec_d + LSM |
| `sec_strip_caps` | 轻微安全违规 | sec_d + LSM |
| `audit_drop_info` | 审计存储紧张 | audit_d |

### 7.2 自动响应编排

复杂场景下需要编排多个动作，audit_d 内置编排引擎：

```yaml
# /etc/airy/incident/playbooks/oom_recovery.yaml
name: oom_recovery
trigger:
  severity: P0
  category: resource
  condition: "airy_mem_available_pct < 5"
steps:
  - action: mem_evict_l1l2
    target: all_agents
    on_failure: continue
  - action: sched_adjust_dl
    target: low_priority_agents
    params:
      runtime_ratio: 0.5
    on_failure: continue
  - action: agent_pause
    target: lowest_priority_agent
    on_failure: escalate
  - action: dsl_activate
    condition: "airy_mem_available_pct < 3"
    on_failure: escalate
```

编排引擎支持条件分支、失败处理、超时控制，详见 [10-devstation.md](10-devstation.md) 的"Playbook 编辑器"。

### 7.3 人工介入接口

自动响应失败或 P0 事件必须人工介入时，运维人员通过以下接口操作：

```bash
# 查看事件详情
airyctl incident show INC-20260718-001

# 手动触发响应动作
airyctl incident respond INC-20260718-001 --action agent_migrate --agent-id 42

# 确认恢复
airyctl incident resolve INC-20260718-001 --comment "Root cause fixed"

# 重新打开事件
airyctl incident reopen INC-20260718-001 --reason "Issue recurred"

# 触发降级模式
airyctl dsl activate --reason "Manual activation for maintenance"

# 退出降级模式
airyctl dsl exit --confirm
```

---

## 8. 事件复盘报告模板

### 8.1 模板结构

事件复盘报告采用标准化模板，存储于 `/var/lib/airy/incidents/postmortem/PM-<incident_id>.md`：

```markdown
# 事件复盘报告：<事件标题>

## 元信息
- 事件 ID：<INC-YYYYMMDD-NNN>
- 严重程度：<P0/P1/P2>
- 类别：<system/agent/security/network/resource>
- 检测时间：<YYYY-MM-DDTHH:MM:SSZ>
- 恢复时间：<YYYY-MM-DDTHH:MM:SSZ>
- 持续时长：<HH:MM:SS>
- 报告人：<姓名>
- 报告时间：<YYYY-MM-DDTHH:MM:SSZ>

## 1. 事件摘要
<一段话描述事件经过，包括影响范围与持续时长>

## 2. 影响分析
- 受影响 Agent：<数量 / Agent ID 列表>
- 受影响 daemon：<列表>
- 业务影响：<描述>
- 数据影响：<是否有数据丢失/损坏>

## 3. 时间线
| 时间 | 事件 | 来源 |
|------|------|------|
| T+0s  | 检测到异常 | audit_d |
| T+2s  | 自动响应启动 | macro_d |
| T+10s | DSL 模式激活 | macro_d |
| ...   | ...           | ...   |
| T+12m | 恢复确认 | ops_alice |

## 4. 根因分析
<详细分析事件根因，包括触发条件、传导路径、放大因素>

### 4.1 触发原因
<直接触发事件的原因>

### 4.2 根本原因
<深层原因，可能是设计缺陷、配置错误、外部依赖等>

### 4.3 5 Whys 分析
1. 为什么发生？→ <回答>
2. 为什么<回答1>？→ <回答>
3. ...

## 5. 响应评估
- 检测是否及时？<是/否，说明>
- 分类是否准确？<是/否，说明>
- 自动响应是否有效？<是/否，说明>
- 人工介入是否及时？<是/否，说明>
- 恢复是否完整？<是/否，说明>

## 6. 改进措施
| 编号 | 措施 | 负责人 | 截止日期 | 状态 |
|------|------|--------|----------|------|
| 1    | <措施> | <姓名> | <日期>   | <待办/进行中/完成> |
| 2    | ...   | ...    | ...      | ...   |

## 7. 经验教训
- <经验1>
- <经验2>

## 8. 附录
- 告警记录：ALT-YYYYMMDD-NNN
- 监控快照：<链接>
- 日志归档：<路径>
```

### 8.2 复盘流程

1. 事件 RESOLVED 后 24 小时内，运维 on-call 启动复盘。
2. 自动生成时间线（从事件日志提取）。
3. 召集复盘会议（P0 事件需邀请系统负责人）。
4. 填写复盘报告，归档到 `/var/lib/airy/incidents/postmortem/`。
5. 改进措施录入 devstation 任务看板，跟踪至完成。

### 8.3 复盘报告统计

audit_d 每月生成复盘报告统计：

- 当月事件总数与分级分布。
- 平均 MTTR（Mean Time To Recover）。
- 改进措施完成率。
- 重复事件分析（相同根因的事件）。

---

## 9. 配置参数

事件响应体系的关键配置参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `airy_incident.auto_respond_p0` | true | P0 事件是否自动响应 |
| `airy_incident.auto_respond_p1` | true | P1 事件是否自动响应 |
| `airy_incident.p0_respond_timeout_s` | 10 | P0 响应超时（秒） |
| `airy_incident.p1_respond_timeout_s` | 60 | P1 响应超时（秒） |
| `airy_incident.recovery_monitor_min` | 5 | 恢复监控时长（分钟） |
| `airy_incident.retention_days` | 90 | 事件日志保留（天） |
| `airy_incident.security_retention_days` | 365 | 安全事件保留（天） |
| `airy_incident.postmortem_deadline_h` | 24 | 复盘报告截止（小时） |
| `airy_incident.dsl_auto_exit` | false | 降级模式是否自动退出 |

---

## 10. 安全考虑

### 10.1 响应动作授权

- 自动响应动作仅限 macro_d / sec_d / sched_d 等核心 daemon 执行。
- 人工响应动作需通过 `airyctl` 工具，需 `airy-incident` 组成员身份。
- 危险动作（`dsl_activate`/`sec_freeze_process`）需 `--confirm` 显式确认。

### 10.2 事件日志防篡改

- 事件日志 append-only，禁止修改与删除（活跃期内）。
- 归档后日志只读，仅 audit_d 在清理时（超 90 天）可删除。
- sec_d 监控事件日志目录，违规修改触发 CRITICAL 告警。

### 10.3 敏感信息

- 事件日志中不应包含密钥、token 等敏感信息。
- audit_d 写入前自动脱敏（与告警体系共用规则）。
- 复盘报告中的日志摘录需人工核对，避免敏感信息泄露。

---

## 11. 与其他子系统的协作

### 11.1 与监控体系

监控体系是事件检测的主要来源，详见 [03-monitoring.md](03-monitoring.md)。

### 11.2 与告警体系

告警体系是事件响应的输入，CRITICAL/WARNING 告警触发对应级别事件，详见 [04-alerting.md](04-alerting.md)。

### 11.3 与备份恢复

严重事件可能导致数据损坏，需触发备份恢复流程，详见 [06-backup-recovery.md](06-backup-recovery.md)。

### 11.4 与 systemd 集成

systemd 提供单元级故障检测与重启能力，详见 [07-systemd-integration.md](07-systemd-integration.md)。

### 11.5 与 110-security

安全事件由 sec_d 主导处置，详见 [../110-security/README.md](../110-security/README.md)。

### 11.6 与 150-cloudnative

Agent 迁移与跨节点故障转移由 150-cloudnative 模块支持，详见 [../150-cloudnative/README.md](../150-cloudnative/README.md)。

---

## 12. 版本演进

### 12.1 v0.1.1 → v1.0.1 变更

- 新增"响应编排引擎"，支持 Playbook 定义复杂响应流程。
- DSL 降级模式新增"分阶段退出"能力，避免恢复期雪崩。
- 事件日志保留期从 60 天延长至 90 天。
- 复盘报告模板增加"5 Whys 分析"小节。

### 12.2 后续规划（v1.1.0）

- 引入基于机器学习的根因推断，辅助运维快速定位。
- 支持多节点联邦事件响应（与 150-cloudnative 联动）。
- Playbook 库开源，社区共享响应最佳实践。

---

## 13. 相关文档

- [03-monitoring.md](03-monitoring.md)：监控体系，事件检测来源。
- [04-alerting.md](04-alerting.md)：告警体系，事件响应输入。
- [06-backup-recovery.md](06-backup-recovery.md)：备份恢复，严重事件后的数据恢复。
- [07-systemd-integration.md](07-systemd-integration.md)：systemd 集成，单元级故障检测。
- [10-devstation.md](10-devstation.md)：开发运维工作站，事件可视化与人工操作。
- [../110-security/README.md](../110-security/README.md)：安全运维，安全事件处置。
- [../150-cloudnative/README.md](../150-cloudnative/README.md)：云原生，跨节点故障转移。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
