Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Agent 运维（Agent Operations）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · Agent 运维设计文档，定义 Agent 生命周期管理、健康检查、资源调优、迁移、升级与日志收集\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

Agent 是 AirymaxOS 的一等公民，承载智能体业务逻辑。Agent 运维体系设计目标如下：

1. **全生命周期管理**：覆盖创建、启动、监控、调优、迁移、升级、终止七个阶段。
2. **零停机运维**：热升级、灰度发布、在线迁移等操作不中断 Agent 服务。
3. **细粒度资源控制**：每个 Agent 独立的 CPU、内存、Token 预算，避免相互影响。
4. **自动化优先**：健康检查、资源调优、故障恢复等尽量自动化，减少人工介入。
5. **可观测性**：Agent 运行状态、资源消耗、Token 使用等全程可观测。

### 1.2 Agent 模型回顾

AirymaxOS 中的 Agent 具有以下特征：

- **唯一 ID**：每个 Agent 拥有全局唯一 ID（自增整数）。
- **独立地址空间**：Agent 运行在独立进程，进程间通过 A-IPC（IPC）通信。
- **多层记忆**：每个 Agent 拥有 L1-L4 四层 MemoryRovol 记忆（详见 [09-memory-operations.md](09-memory-operations.md)）。
- **调度属性**：Agent 默认使用 SCHED_DEADLINE 调度，保证实时性。
- **能力集**：Agent 拥有显式能力集（capability set），遵循最小权限原则。

### 1.3 与其他子系统的关系

| 子系统 | 在 Agent 运维中的角色 |
|--------|-----------------------|
| macro_d (A-ULS) | Agent 生命周期监管、健康检查、自动重启 |
| sched_d | Agent 调度参数调整、资源配额 |
| cogn_d | Agent Token 消耗监控、推理预算 |
| mem_d | Agent 记忆层级管理 |
| logger_d (A-ULP) | Agent 日志收集与持久化 |
| config_d (A-UCS) | Agent 配置管理 |
| sec_d | Agent 安全隔离与权限管控 |
| audit_d | Agent 运维事件聚合 |
| 150-cloudnative | Agent 跨节点迁移 |

### 1.4 与技术选型的契合

- **sched_tac（不使用 sched_ext）**：Agent 调度通过 `SCHED_DEADLINE` 系统调用直接配置，不依赖 BPF 调度器。
- **IORING_OP_URING_CMD**：Agent 间 IPC 与日志上报复用 io_uring。
- **纯 C LSM**：Agent 权限通过 LSM hook 强制，避免越权访问。
- **IRON-9 v3 [SC] 层**：Agent 调度由 [SC]（Scheduling Core）层负责，提供 deadline 保证。

---

## 2. Agent 生命周期运维

### 2.1 生命周期状态机

Agent 经历以下状态：

```
                  [创建]
                     │
                     ▼
                 ┌────────┐
       ┌────────>│ CREATED│
       │         └────┬───┘
       │              │ start
       │              ▼
       │         ┌────────┐
       │         │RUNNING │◄───────┐
       │         └────┬───┘        │
       │              │ pause      │ resume
       │              ▼            │
       │         ┌────────┐        │
       │         │ PAUSED ├────────┘
       │         └────┬───┘
       │              │ stop
       │              ▼
       │         ┌────────┐
       │         │STOPPED │
       │         └────┬───┘
       │              │ restart
       │              ▼
       └──────────────┘
                      │ terminate
                      ▼
                 ┌────────┐
                 │TERMINATED│
                 └────────┘
```

状态说明：

| 状态 | 含义 | 资源占用 | IPC 可达 |
|------|------|----------|----------|
| CREATED | 已创建未启动 | 仅元数据 | 否 |
| RUNNING | 正在运行 | 全部资源 | 是 |
| PAUSED | 已暂停 | 内存保留，CPU 释放 | 否 |
| STOPPED | 已停止 | 内存释放，状态持久化 | 否 |
| TERMINATED | 已终止 | 全部释放 | 否 |

### 2.2 创建阶段

Agent 创建流程：

1. **请求**：用户/其他 Agent 通过 `airyctl agent create` 或 RPC 发起创建请求。
2. **校验**：macro_d 校验请求参数（名称、模板、预算等）。
3. **分配 ID**：macro_d 分配全局唯一 Agent ID。
4. **初始化元数据**：写入 `/var/lib/airy/agents/<id>/meta.json`。
5. **加载模板**：config_d 加载 Agent 模板配置。
6. **分配资源**：sched_d 分配调度参数，mem_d 分配记忆空间。
7. **安全配置**：sec_d 应用能力集与访问控制。
8. **就绪**：Agent 进入 CREATED 状态，等待启动。

创建命令示例：

```bash
airyctl agent create \
    --name "customer-support-bot" \
    --template "llm-chat-v2" \
    --token-budget 1000000 \
    --memory-l1 256MB \
    --memory-l2 1GB \
    --sched-runtime 10ms \
    --sched-deadline 30ms \
    --sched-period 100ms \
    --capabilities "ipc.send,ipc.recv,memory.l3.read"
```

### 2.3 启动阶段

Agent 启动流程：

1. **请求**：`airyctl agent start <id>` 或自动启动（如开机自启）。
2. **fork 进程**：macro_d 通过 systemd-run 或直接 fork 创建 Agent 进程。
3. **加载记忆**：mem_d 加载 L3/L4 索引到内存，初始化 L1/L2。
4. **建立 IPC**：A-IPC 为 Agent 创建专用 io_uring 队列。
5. **注册调度**：sched_d 将 Agent 进程加入 SCHED_DEADLINE 调度。
6. **就绪上报**：Agent 通过 sd_notify 上报 READY=1。
7. **状态切换**：macro_d 将状态改为 RUNNING。
8. **监控接入**：audit_d 开始采集该 Agent 的监控指标。

### 2.4 监控阶段

详见第 3 节"Agent 健康检查"。

### 2.5 调优阶段

详见第 4 节"Agent 资源调优"。

### 2.6 迁移阶段

详见第 6 节"Agent 迁移"。

### 2.7 终止阶段

Agent 终止流程：

1. **请求**：`airyctl agent terminate <id>`。
2. **优雅停止**：macro_d 向 Agent 发送 SIGTERM，等待 30s。
3. **强制停止**：超时后发送 SIGKILL。
4. **状态持久化**：mem_d 将 L1/L2 数据写入 L3（如配置允许）。
5. **资源释放**：sched_d 移除调度，mem_d 释放记忆，A-IPC 关闭 IPC。
6. **元数据归档**：meta.json 移动到 `/var/lib/airy/agents.archive/`。
7. **审计记录**：audit_d 记录终止事件，保留 90 天。

终止是不可逆操作。如需保留 Agent 以备重启，使用 `stop` 而非 `terminate`。

---

## 3. Agent 健康检查

### 3.1 心跳检测

macro_d 对每个 RUNNING 状态的 Agent 执行定期心跳检测：

- **心跳频率**：默认 2 秒一次（由 `airy_agent.heartbeat_interval_s` 配置）。
- **超时阈值**：默认 6 秒未收到心跳（3 次心跳周期，由 `airy_agent.heartbeat_timeout_s` 配置）。
- **心跳内容**：Agent 通过 io_uring CMD 发送 `AgentHeartbeat` 消息，包含：
  - `agent_id`：Agent ID。
  - `ts_ns`：发送时间戳。
  - `state`：Agent 自评状态（healthy/degraded/critical）。
  - `last_error`：最近错误码（A-UEF 错误码，0 表示无错误）。
  - `pending_tasks`：待处理任务数。

### 3.2 健康分计算

macro_d 综合以下因素计算 Agent 健康分（0-100）：

| 因素 | 权重 | 计算方式 |
|------|------|----------|
| 心跳响应 | 30% | 心跳超时次数 / 30 次扣 100 分 |
| CPU 占用 | 15% | 超过配额 90% 扣分 |
| 内存占用 | 15% | L1+L2 超过限制 90% 扣分 |
| IPC 延迟 | 15% | p99 > 5ms 扣分 |
| 错误率 | 15% | 1 分钟错误数 > 10 扣分 |
| Token 消耗速率 | 10% | 接近预算上限扣分 |

健康分等级：

- 80-100：HEALTHY（健康）
- 50-79：DEGRADED（降级）
- 1-49：CRITICAL（严重）
- 0：DEAD（死亡）

### 3.3 健康检查动作

| 健康分 | 自动动作 |
|--------|----------|
| 80-100 | 无 |
| 50-79 | 触发 WARNING 告警，macro_d 评估是否需要调优 |
| 30-49 | 触发 WARNING 告警，sched_d 降低 Agent 优先级 |
| 10-29 | 触发 CRITICAL 告警，macro_d 暂停 Agent |
| <10 | 触发 CRITICAL 告警，macro_d 重启 Agent |
| 0（连续 3 次心跳超时） | 触发 P0 事件，重启或迁移 Agent |

### 3.4 深度健康检查

除了心跳检测，macro_d 周期执行深度健康检查（默认 30 秒一次）：

- **进程存在性**：通过 `/proc/<pid>/` 验证进程存活。
- **文件描述符**：检查 fd 使用是否异常（如泄漏）。
- **内存映射**：检查 RSS 是否异常增长。
- **IPC 队列**：检查 io_uring 队列是否堆积。
- **磁盘 IO**：检查 Agent 是否有异常 IO 模式。

深度检查结果计入健康分，并触发相应的运维动作。

---

## 4. Agent 资源调优

### 4.1 调度参数调优

sched_d 根据 Agent 运行特征动态调整 `SCHED_DEADLINE` 参数：

| 参数 | 含义 | 调优策略 |
|------|------|----------|
| `runtime` | 单周期内 CPU 时间 | 根据 CPU 利用率动态调整 |
| `deadline` | 任务完成期限 | 根据 IPC 延迟要求调整 |
| `period` | 调度周期 | 根据任务频率调整 |

调优示例：

- Agent CPU 利用率持续 >90%，且 deadline miss 率 >5%：增加 `runtime`（如 10ms → 15ms）。
- Agent CPU 利用率持续 <30%：减少 `runtime`（如 10ms → 5ms），释放 CPU 给其他 Agent。
- Agent IPC 延迟 p99 > 5ms：缩短 `deadline`（如 30ms → 20ms），提高优先级。

### 4.2 内存配额调优

mem_d 根据 Agent 记忆使用情况动态调整 L1/L2 配额：

- L1 持续溢出（LRU 淘汰 >10/min）：将 L1 上限从 256MB 提升至 384MB（如系统内存允许）。
- L1 利用率持续 <50%：将 L1 上限降低至 128MB，节省内存。
- L2 同理，范围 512MB-2GB。

调优动作记录到 audit_d，供运维审计。

### 4.3 Token 预算管理

cogn_d 负责监控 Agent Token 消耗，详见第 5 节。

### 4.4 调优策略配置

每个 Agent 可配置独立的调优策略：

```yaml
# /etc/airy/agents/<id>/tuning.yaml
sched:
  auto_tune: true
  min_runtime_ms: 5
  max_runtime_ms: 50
  target_cpu_pct: 70
  target_dl_miss_rate: 0.01

memory:
  auto_tune: true
  l1_min_mb: 128
  l1_max_mb: 512
  l2_min_mb: 512
  l2_max_mb: 2048
  tune_interval_s: 60

token:
  auto_throttle: true
  warning_pct: 80
  critical_pct: 95
  throttle_action: pause  # 或 degrade
```

### 4.5 手动调优

运维人员可手动调整 Agent 资源：

```bash
# 调整调度参数
airyctl agent tune <id> --sched-runtime 15ms --sched-deadline 25ms

# 调整内存配额
airyctl agent tune <id> --memory-l1 384MB --memory-l2 1536MB

# 调整 Token 预算
airyctl agent tune <id> --token-budget 2000000

# 重置为自动调优
airyctl agent tune <id> --auto
```

---

## 5. Token 预算管理

### 5.1 Token 预算模型

每个 Agent 拥有 Token 预算，由 cogn_d 监控：

| 字段 | 说明 |
|------|------|
| `total_budget` | 总预算（创建时设定） |
| `consumed` | 已消耗 Token 数 |
| `remaining` | 剩余预算 |
| `rate_per_sec` | 实时消耗速率 |
| `reset_period` | 预算重置周期（如 daily/monthly） |
| `last_reset_at` | 上次重置时间 |

### 5.2 Token 消耗监控

cogn_d 在每次 Agent 推理调用前后埋点：

- 调用前：记录 `input_tokens`。
- 调用后：记录 `output_tokens`。
- 累计到 `consumed`，更新 `remaining`。
- 实时计算 `rate_per_sec`（滑动窗口 10s）。

监控指标：

- `agent_N_token_input_total`
- `agent_N_token_output_total`
- `agent_N_token_rate_per_sec`
- `agent_N_token_budget_remaining`
- `agent_N_token_budget_pct`

### 5.3 预算告警

| 阈值 | 动作 |
|------|------|
| remaining < 20% | WARNING 告警，通知运维 |
| remaining < 5% | CRITICAL 告警，触发 `throttle_action` |
| remaining = 0 | 暂停 Agent（或降级到 fallback 模型） |

### 5.4 预算控制策略

`throttle_action` 配置选项：

- `pause`：暂停 Agent，等待预算重置或人工介入。
- `degrade`：切换到更便宜的 fallback 模型（如 GPT-4 → GPT-3.5）。
- `notify`：仅通知，不自动处理（人工决策）。
- `reject`：拒绝新的推理请求，保留现有对话。

### 5.5 预算重置

预算按 `reset_period` 自动重置：

- `daily`：每日 00:00 UTC 重置。
- `monthly`：每月 1 日 00:00 UTC 重置。
- `manual`：仅人工重置（`airyctl agent token reset <id>`）。

重置时 `consumed` 清零，`remaining` 恢复为 `total_budget`，触发 INFO 告警。

---

## 6. Agent 迁移

### 6.1 迁移场景

Agent 迁移适用于以下场景：

| 场景 | 触发方式 | 迁移类型 |
|------|----------|----------|
| 节点故障 | 自动（P0 事件） | 冷迁移 |
| 负载均衡 | 自动（sched_d 决策） | 热迁移 |
| 计划维护 | 人工 | 热迁移 |
| 资源不足 | 自动（mem_d 决策） | 热迁移 |
| 灾难恢复 | 自动（DSL 模式） | 冷迁移 |

### 6.2 热迁移流程

热迁移不中断 Agent 服务，流程：

1. **预迁移**：macro_d 在目标节点预创建 Agent（CREATED 状态）。
2. **状态同步**：
   - L1/L2 记忆通过 RPC 实时同步到目标节点。
   - 调度参数、IPC 状态、Token 预算同步。
3. **切换准备**：
   - 源节点 Agent 进入"dual-write"模式（IPC 消息同时发往源和目标）。
   - 目标节点 Agent 进入"warm-up"模式（接收消息但不响应）。
4. **切换**：
   - 源节点暂停 Agent（PAUSED）。
   - 目标节点 Agent 进入 RUNNING 状态。
   - 通知所有 IPC 对端更新 Agent 地址。
5. **清理**：
   - 源节点 Agent 进入 STOPPED 状态。
   - 释放源节点资源（保留元数据 24 小时）。

热迁移耗时目标：<2 秒（100MB L1+L2 记忆规模）。

### 6.3 冷迁移流程

冷迁移用于源节点不可用场景：

1. **检测**：macro_d 检测到源节点故障（心跳超时）。
2. **选择目标**：根据资源可用性选择目标节点。
3. **恢复状态**：从最近备份恢复 Agent 元数据与 L3/L4 记忆。
4. **重建 L1/L2**：mem_d 从 L3 提取热数据重建 L1/L2。
5. **启动**：在目标节点启动 Agent。
6. **通知**：通知所有 IPC 对端更新地址。

冷迁移 RTO 目标：<60 秒（依赖备份恢复速度）。

### 6.4 与 150-cloudnative 的协作

Agent 跨节点迁移由 150-cloudnative 模块支持：

- **调度决策**：150-cloudnative 的调度器决定迁移目标节点。
- **网络配置**：迁移后自动更新 Service Mesh 路由。
- **状态同步**：复用 150-cloudnative 的数据同步通道。
- **故障转移**：节点故障时自动触发迁移。

详见 [../150-cloudnative/README.md](../150-cloudnative/README.md)。

### 6.5 迁移命令

```bash
# 手动迁移
airyctl agent migrate <id> --target-node node-2

# 查询迁移历史
airyctl agent migrate history <id>

# 回滚迁移（仅热迁移支持）
airyctl agent migrate rollback <id>
```

---

## 7. Agent 版本升级

### 7.1 升级类型

| 类型 | 停机时间 | 适用场景 |
|------|----------|----------|
| 热升级 | 0 | 代码兼容的配置/模型更新 |
| 滚动升级 | 单 Agent 重启时间 | 多实例 Agent 集群升级 |
| 灰度发布 | 0 | 新版本小流量验证 |
| 强制升级 | 全部 | 紧急安全补丁 |

### 7.2 热升级

热升级不重启 Agent 进程，仅更新运行时配置：

1. **准备**：config_d 加载新版本配置到 `/etc/airy/agents/<id>.new/`。
2. **校验**：校验配置完整性与兼容性。
3. **原子切换**：通过 `renameat2(ATOMIC_REPLACE)` 原子替换配置目录。
4. **通知**：macro_d 发送 SIGHUP 给 Agent。
5. **加载**：Agent 重新加载配置，上报 READY=1。
6. **验证**：健康分稳定 60s 后视为升级成功。

适用场景：模型版本切换、阈值调整、能力集变更。

### 7.3 滚动升级

多实例 Agent 集群的滚动升级：

1. **批次划分**：将 Agent 集群分为 N 批（默认 10%）。
2. **逐批升级**：
   - 暂停该批次 Agent（PAUSED）。
   - 更新配置与代码。
   - 重启 Agent。
   - 健康分稳定 60s 后进入下一批。
3. **回滚**：任一批次失败，回滚已升级批次。

```bash
airyctl agent upgrade --group "customer-bots" --version v2.1.0 --batch-size 10%
```

### 7.4 灰度发布

新版本小流量验证：

1. **创建灰度 Agent**：基于新版本创建少量 Agent（如 5%）。
2. **流量分流**：gateway_d 按比例将流量分到灰度 Agent。
3. **监控对比**：audit_d 对比灰度与稳定版本的关键指标（错误率、延迟、Token 消耗）。
4. **决策**：
   - 指标正常：逐步扩大灰度比例（5% → 25% → 50% → 100%）。
   - 指标异常：回滚灰度 Agent。

### 7.5 强制升级

紧急安全补丁的强制升级：

1. sec_d 发布安全告警，标记受影响 Agent。
2. macro_d 立即暂停所有受影响 Agent。
3. 应用补丁并重启。
4. 验证后恢复。

强制升级不保证业务连续性，仅在紧急安全事件时使用。

### 7.6 升级回滚

所有升级操作支持回滚：

```bash
# 查看升级历史
airyctl agent upgrade history <id>

# 回滚到上一版本
airyctl agent upgrade rollback <id>

# 回滚到指定版本
airyctl agent upgrade rollback <id> --version v2.0.5
```

回滚保留最近 3 个版本的配置与代码，存储在 `/var/lib/airy/agents/<id>/versions/`。

---

## 8. Agent 日志收集

### 8.1 日志收集架构

Agent 日志通过 A-ULP 模块 logger_d 统一收集：

```
[Agent 进程] ──── io_uring CMD ────> [logger_d Ring Buffer]
                                              │
                                  ┌───────────┼───────────┐
                                  ▼           ▼           ▼
                            [实时流]    [journal]    [持久化文件]
                                  │           │           │
                                  ▼           ▼           ▼
                            [audit_d]   [systemd]   [/var/log/airy/]
                            [聚合]      [journal]   [agents/<id>/]
```

### 8.2 日志级别

Agent 日志级别对齐 A-ULP 5 级：

| 级别 | 含义 | 持久化 | 告警 |
|------|------|--------|------|
| FATAL | 致命错误，Agent 不可恢复 | 永久 | CRITICAL |
| ERROR | 错误，影响业务 | 90 天 | CRITICAL |
| WARN | 警告，潜在问题 | 30 天 | WARNING |
| INFO | 信息性日志 | 7 天 | 无 |
| DEBUG | 调试日志 | 1 天（仅本地） | 无 |

### 8.3 日志结构

Agent 日志采用结构化格式：

```json
{
  "ts_ns": 1784328858000000000,
  "agent_id": 42,
  "level": "INFO",
  "module": "chat_engine",
  "message": "User query received",
  "request_id": "req-abc123",
  "labels": {
    "user_id": "u-789",
    "session_id": "s-def456"
  }
}
```

### 8.4 日志上报接口

Agent 通过 A-IPC 提供的 `airy_log_emit()` 接口上报日志：

```c
struct airy_log_entry {
    uint64_t ts_ns;
    uint32_t agent_id;
    uint8_t  level;        /* 0=FATAL, 1=ERROR, 2=WARN, 3=INFO, 4=DEBUG */
    char     module[32];
    char     message[512];
    uint32_t labels_off;
    uint32_t labels_len;
};

int airy_log_emit(const struct airy_log_entry *e, size_t n);
```

底层复用 io_uring CMD，批量提交到 logger_d 的 Ring Buffer。

### 8.5 日志查询

```bash
# 查询特定 Agent 的最近日志
airyctl agent log <id> --tail 100

# 按级别过滤
airyctl agent log <id> --level ERROR --since 1h

# 按关键词搜索
airyctl agent log <id> --grep "timeout"

# 实时流
airyctl agent log <id> --follow
```

devstation 提供日志查看器，详见 [10-devstation.md](10-devstation.md)。

### 8.6 日志持久化

- **短期**：logger_d Ring Buffer（内存，1 小时）。
- **中期**：`/var/log/airy/agents/<id>/<YYYYMMDD>.log.zst`（按天分文件，zstd 压缩）。
- **长期**：超 30 天的日志归档到 `/var/lib/airy/log.archive/`，保留 90 天。

---

## 9. Agent 运维命令汇总

### 9.1 生命周期命令

```bash
airyctl agent create [options]              # 创建 Agent
airyctl agent start <id>                    # 启动 Agent
airyctl agent pause <id>                    # 暂停 Agent
airyctl agent resume <id>                   # 恢复 Agent
airyctl agent stop <id>                     # 停止 Agent（保留状态）
airyctl agent terminate <id>                # 终止 Agent（释放资源）
airyctl agent restart <id>                  # 重启 Agent
airyctl agent status <id>                   # 查看状态
airyctl agent list                          # 列出所有 Agent
```

### 9.2 调优命令

```bash
airyctl agent tune <id> [options]           # 调整资源
airyctl agent tune <id> --auto              # 重置为自动调优
airyctl agent token <id> budget <amount>    # 调整 Token 预算
airyctl agent token <id> reset              # 重置 Token 消耗
airyctl agent token <id> status             # 查看 Token 状态
```

### 9.3 迁移与升级命令

```bash
airyctl agent migrate <id> --target-node <node>   # 迁移 Agent
airyctl agent migrate history <id>                # 迁移历史
airyctl agent migrate rollback <id>               # 回滚迁移
airyctl agent upgrade --group <g> --version <v>   # 升级 Agent
airyctl agent upgrade rollback <id>               # 回滚升级
```

### 9.4 日志命令

```bash
airyctl agent log <id> [options]            # 查询日志
airyctl agent log <id> --follow             # 实时流
airyctl agent log archive <id>              # 归档日志
```

---

## 10. 配置参数

Agent 运维的关键配置参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `airy_agent.heartbeat_interval_s` | 2 | 心跳间隔（秒） |
| `airy_agent.heartbeat_timeout_s` | 6 | 心跳超时（秒） |
| `airy_agent.deep_check_interval_s` | 30 | 深度检查间隔（秒） |
| `airy_agent.auto_restart_burst` | 3 | 5 分钟内重启次数上限 |
| `airy_agent.auto_migrate_on_failure` | true | 故障时自动迁移 |
| `airy_agent.default_l1_mb` | 256 | 默认 L1 大小（MB） |
| `airy_agent.default_l2_mb` | 1024 | 默认 L2 大小（MB） |
| `airy_agent.default_token_budget` | 1000000 | 默认 Token 预算 |
| `airy_agent.default_sched_runtime_ms` | 10 | 默认调度 runtime（ms） |
| `airy_agent.default_sched_deadline_ms` | 30 | 默认调度 deadline（ms） |
| `airy_agent.default_sched_period_ms` | 100 | 默认调度 period（ms） |
| `airy_agent.log_retention_d` | 30 | 日志保留（天） |
| `airy_agent.version_rollback_keep` | 3 | 回滚版本保留数 |

---

## 11. 安全考虑

### 11.1 Agent 权限

- 每个 Agent 拥有显式能力集，遵循最小权限原则。
- 能力集通过纯 C LSM 强制，Agent 无法越权。
- Agent 间 IPC 受 A-IPC 控制，仅允许声明的关系通信。

### 11.2 Agent 隔离

- Agent 运行在独立进程，地址空间隔离。
- Agent 文件系统访问受 `ProtectSystem=` 限制。
- Agent 网络访问受 sec_d 网络策略限制。

### 11.3 Agent 日志安全

- Agent 日志中的敏感字段由 logger_d 脱敏。
- Agent 日志查询需 `airy-log` 组权限。
- Agent 日志不可篡改（append-only）。

### 11.4 Agent 升级安全

- 升级包需 sec_d 签名验证。
- 升级前自动备份当前版本。
- 强制升级仅由 sec_d 触发。

---

## 12. 与其他子系统的协作

### 12.1 与监控体系

Agent 监控指标详见 [03-monitoring.md](03-monitoring.md) 第 2.2 节。

### 12.2 与告警体系

Agent 告警源详见 [04-alerting.md](04-alerting.md) 第 3 节。

### 12.3 与事件响应

Agent 异常事件响应详见 [05-incident-response.md](05-incident-response.md) 第 3.3 节。

### 12.4 与备份恢复

Agent 状态备份详见 [06-backup-recovery.md](06-backup-recovery.md) 第 2.3 节。

### 12.5 与记忆运维

Agent L1-L4 记忆运维详见 [09-memory-operations.md](09-memory-operations.md)。

### 12.6 与 150-cloudnative

Agent 跨节点迁移详见 [../150-cloudnative/README.md](../150-cloudnative/README.md)。

---

## 13. 版本演进

### 13.1 v0.1.1 → v1.0.1 变更

- 新增 Agent 灰度发布能力。
- 健康分计算引入 IPC 延迟因子。
- 热迁移支持 dual-write 模式，RTO <2s。
- Token 预算支持 fallback 模型自动切换。

### 13.2 后续规划（v1.1.0）

- Agent 自愈能力（基于历史模式识别异常）。
- Agent 资源调优引入机器学习模型。
- 支持 Agent 模板市场（社区共享）。

---

## 14. 相关文档

- [03-monitoring.md](03-monitoring.md)：监控体系，Agent 监控指标。
- [04-alerting.md](04-alerting.md)：告警体系，Agent 告警。
- [05-incident-response.md](05-incident-response.md)：事件响应，Agent 异常处置。
- [06-backup-recovery.md](06-backup-recovery.md)：备份恢复，Agent 状态备份。
- [07-systemd-integration.md](07-systemd-integration.md)：systemd 集成，Agent 进程管理。
- [09-memory-operations.md](09-memory-operations.md)：记忆运维，Agent L1-L4 记忆。
- [10-devstation.md](10-devstation.md)：开发运维工作站，Agent 管理可视化。
- [../150-cloudnative/README.md](../150-cloudnative/README.md)：云原生，Agent 跨节点迁移。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
