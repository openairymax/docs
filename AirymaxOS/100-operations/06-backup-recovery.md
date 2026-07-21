Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 备份恢复（Backup & Recovery）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · 备份恢复设计文档，定义备份对象、策略、存储、恢复流程与灾难恢复方案\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

备份恢复体系是 AirymaxOS 运维分册的关键保障子模块，确保在硬件故障、软件缺陷、人为误操作或安全事件后能够快速恢复业务。设计目标如下：

1. **RPO ≤1 小时**：单小时内的数据丢失风险可控，通过每小时增量备份实现。
2. **RTO ≤15 分钟**：单节点故障后 15 分钟内恢复核心服务，通过自动化恢复流程实现。
3. **完整性保证**：所有备份均经过 SHA256 校验与数字签名，防止篡改与损坏。
4. **业务无感**：备份过程对 Agent 与 daemon 透明，不中断服务。
5. **可演练**：备份恢复流程支持定期演练，确保真实故障时可用。

### 1.2 与其他子系统的关系

| 子系统 | 关系 |
|--------|------|
| mem_d | 提供 L3/L4 记忆数据源，执行记忆恢复 |
| config_d | 提供配置文件备份与恢复入口 |
| macro_d | 协调恢复流程，重启 Agent |
| audit_d | 备份自身聚合数据，记录备份恢复事件 |
| sec_d | 提供备份加密与签名，验证恢复完整性 |
| [DSL] 降级模式 | 灾难恢复时启用，保证最小可运行子集 |
| 事件响应 | 严重事件触发备份恢复，详见 [05-incident-response.md](05-incident-response.md) |

### 1.3 与技术选型的契合

- **IORING_OP_URING_CMD**：备份读写复用 io_uring 异步通道，避免阻塞主路径。
- **纯 C LSM**：备份数据访问受 LSM 保护，仅 `airy-backup` 组可读。
- **IRON-9 v3 [DSL] 层**：灾难恢复时启用降级模式，逐步恢复服务。
- **alloc_pages + mmap**：备份大文件读取使用 mmap，减少内存拷贝。

---

## 2. 备份对象

### 2.1 备份对象清单

AirymaxOS 备份对象分为五大类：

| 类别 | 对象 | 路径 | 优先级 |
|------|------|------|--------|
| Agent 状态 | Agent 元数据、运行状态、调度参数 | `/var/lib/airy/agents/<id>/` | P0 |
| 记忆数据 | MemoryRovol L3/L4 记忆 | `/var/lib/airy/memory/l3/`、`/var/lib/airy/memory/l4/` | P0 |
| 配置文件 | 系统配置、daemon 配置、阈值 | `/etc/airy/` | P0 |
| 监控数据 | audit_d 聚合数据、事件日志 | `/var/lib/airy/monitor/`、`/var/lib/airy/incidents/` | P1 |
| 安全数据 | sec_d 密钥库、审计日志 | `/etc/airy/keys/`、`/var/log/airy/` | P0 |

### 2.2 不备份的对象

以下对象不纳入备份范围：

- L1 工作记忆：每 Agent 256MB，变化频繁，且可通过 L2 重建。
- L2 短期记忆：每 Agent 1GB，变化频繁，可通过 L3 重建。
- 临时文件：`/tmp/`、`/var/tmp/`。
- 内核状态：进程列表、内存映射、文件描述符等运行时状态。
- 日志 Ring Buffer：logger_d 内存中的缓冲，已通过 journal 持久化。

### 2.3 Agent 状态备份

每个 Agent 的状态文件位于 `/var/lib/airy/agents/<id>/`：

```
/var/lib/airy/agents/<id>/
├── meta.json              # Agent 元数据（ID、名称、版本、所有者）
├── state.json             # 当前状态（running/paused/stopped）
├── sched.params           # 调度参数（policy、runtime、deadline、period）
├── budget.json            # Token 预算与消耗记录
├── capabilities.json      # Agent 能力集与权限
├── relations.json         # Agent 间关系（依赖、订阅）
└── checkpoint/            # 最近一次检查点
    ├── memory.snapshot    # L1/L2 内存快照（用于热迁移）
    ├── ipc.state          # IPC 队列状态
    └── cogn.context       # cogn_d 上下文
```

备份时将整个目录打包为 `agent_<id>_<timestamp>.tar.zst`。

### 2.4 记忆数据备份

MemoryRovol L3/L4 记忆是 Agent 长期知识的核心载体：

| 层级 | 路径 | 单 Agent 大小 | 备份方式 |
|------|------|---------------|----------|
| L3 | `/var/lib/airy/memory/l3/<agent_id>/` | 100MB-1GB | 增量备份（每小时） |
| L4 | `/var/lib/airy/memory/l4/<agent_id>/` | 10MB-100MB | 全量备份（每日） |

L3 采用增量备份，记录自上次备份后变更的"记忆块"（默认 4KB 块大小）。L4 由于深度压缩且变更较少，采用每日全量备份。

### 2.5 配置文件备份

`/etc/airy/` 下的配置文件：

```
/etc/airy/
├── airy.conf                  # 主配置
├── daemons/                   # 各 daemon 配置
│   ├── macro_d.conf
│   ├── logger_d.conf
│   └── ...
├── monitor/                   # 监控阈值
│   └── thresholds.yaml
├── alert/                     # 告警配置
│   ├── channels.conf
│   └── suppression.yaml
├── supervision/               # A-ULS 监管策略
│   └── restart_policy.yaml
├── incident/                  # 事件响应 Playbook
│   └── playbooks/
└── keys/                      # 密钥（单独加密备份）
    ├── ca.pem
    └── audit_d.pem
```

配置文件采用全量备份（每次打包整个 `/etc/airy/`），密钥目录单独加密备份。

---

## 3. 备份策略

### 3.1 备份层级

AirymaxOS 采用"增量 + 全量"两级备份策略：

| 类型 | 频率 | 内容 | 保留期 | 大小估算（100 Agent） |
|------|------|------|--------|-----------------------|
| 增量备份 | 每小时 | 自上次备份后变更的数据 | 7 天 | ~500MB |
| 全量备份 | 每日凌晨 01:00 | 全部备份对象 | 30 天 | ~5GB |
| 周备份 | 每周日凌晨 02:00 | 全量 + 历史归档 | 12 周 | ~5GB |
| 月备份 | 每月 1 日凌晨 03:00 | 全量 + 月度归档 | 12 月 | ~5GB |

### 3.2 增量备份

增量备份由 audit_d 协调，每小时整点触发：

1. **触发**：macro_d 通过 cron-like 任务每小时向 audit_d 发送备份指令。
2. **快照**：mem_d 对 L3 记忆创建只读快照（基于文件系统 snapshot 或 copy-on-write）。
3. **比对**：audit_d 比对当前快照与上次备份的差异，生成变更块列表。
4. **打包**：将变更块打包为 `incremental_<YYYYMMDDHH>.tar.zst`。
5. **校验**：计算 SHA256 并签名，写入 manifest。
6. **存储**：保存到 `/var/lib/airy/backup/incremental/`。

增量备份耗时目标：<2 分钟（100 Agent 规模）。

### 3.3 全量备份

全量备份在每日凌晨 01:00 执行，由 macro_d 协调：

1. **触发**：macro_d 在 01:00 向所有 daemon 发送"备份准备"通知。
2. **静默**：logger_d 暂停日志轮转，mem_d 暂停 L1/L2 淘汰，避免数据漂移。
3. **快照**：各 daemon 创建一致性快照。
4. **打包**：audit_d 将所有快照打包为 `full_<YYYYMMDD>.tar.zst`。
5. **校验**：计算 SHA256 并签名。
6. **存储**：保存到 `/var/lib/airy/backup/full/`。
7. **恢复**：各 daemon 恢复正常操作。

全量备份耗时目标：<10 分钟（100 Agent 规模），期间业务不受影响（仅短暂静默）。

### 3.4 备份存储结构

```
/var/lib/airy/backup/
├── full/
│   ├── full_20260718.tar.zst          # 全量备份文件
│   ├── full_20260718.sha256           # SHA256 校验
│   ├── full_20260718.sig              # 数字签名
│   └── full_20260718.manifest.json    # 备份清单
├── incremental/
│   ├── incremental_2026071800.tar.zst
│   ├── incremental_2026071800.sha256
│   ├── incremental_2026071800.sig
│   └── incremental_2026071800.manifest.json
├── weekly/
│   └── weekly_2026W29.tar.zst
├── monthly/
│   └── monthly_202607.tar.zst
├── manifest.json                       # 全局备份索引
└── .backup_lock                        # 备份锁（避免并发）
```

### 3.5 备份清单格式

每个备份附带 manifest.json，记录备份内容与元数据：

```json
{
  "backup_id": "FULL-20260718-001",
  "type": "full",
  "created_at": "2026-07-18T01:00:00Z",
  "completed_at": "2026-07-18T01:08:32Z",
  "size_bytes": 5368709120,
  "sha256": "a3f5e8d7...",
  "signature": "MEUCIQDx...",
  "signed_by": "audit_d",
  "agent_count": 100,
  "contents": {
    "agents": [1, 2, 3, "..."],
    "memory_l3_size_bytes": 3221225472,
    "memory_l4_size_bytes": 536870912,
    "config_files": 42,
    "monitor_data_days": 30,
    "security_keys": 8
  },
  "previous_backup_id": "FULL-20260717-001",
  "schema_version": "1.0.1"
}
```

---

## 4. 备份存储

### 4.1 本地存储

默认备份存储在本地 `/var/lib/airy/backup/`，要求：

- 独立文件系统（建议 LVM 卷），避免与业务数据竞争空间。
- 最小容量：全量备份 ×4 + 增量备份 ×168 = 5GB × 4 + 500MB × 168 = 104GB（100 Agent 规模）。
- 推荐容量：200GB（留出周备份与月备份空间）。
- 文件系统：ext4 或 xfs，启用 journal。
- 挂载选项：`noatime,nodiratime` 减少写入。

### 4.2 远程存储

生产环境必须配置远程备份存储，避免单点故障：

| 后端 | 配置 | 用途 |
|------|------|------|
| NFS | `airy_backup.remote.nfs_path` | 简单共享存储 |
| S3 兼容 | `airy_backup.remote.s3_endpoint` | MinIO、AWS S3、Ceph RGW |
| SCP/RSYNC | `airy_backup.remote.ssh_target` | 跨节点复制 |

远程存储写入由 audit_d 在本地备份完成后异步执行，失败时重试 3 次，仍失败则触发 WARNING 告警。

### 4.3 备份加密

- 本地备份默认不加密（依赖文件系统权限保护）。
- 远程备份强制加密，使用 AES-256-CBC，密钥由 sec_d 托管。
- 密钥本身不存储在备份中，需通过单独的 KMS 或离线介质保管。

### 4.4 备份清理

audit_d 每日凌晨 04:00 执行备份清理：

- 增量备份：保留最近 7 天（168 份）。
- 全量备份：保留最近 30 天。
- 周备份：保留最近 12 周。
- 月备份：保留最近 12 月。
- 清理时生成清理日志，记录删除的备份 ID 与原因。

---

## 5. 备份完整性校验

### 5.1 SHA256 校验

每个备份文件生成时计算 SHA256：

```bash
sha256sum full_20260718.tar.zst > full_20260718.sha256
```

恢复前必须先校验：

```bash
sha256sum -c full_20260718.sha256
```

校验失败则该备份不可用，audit_d 触发 CRITICAL 告警。

### 5.2 数字签名

每个备份由 audit_d 使用自身私钥签名（私钥由 sec_d 颁发），签名独立存储为 `.sig` 文件：

```bash
openssl dgst -sha256 -sign /etc/airy/keys/audit_d.key full_20260718.tar.zst > full_20260718.sig
```

恢复前验证签名：

```bash
openssl dgst -sha256 -verify /etc/airy/keys/audit_d.pub -signature full_20260718.sig full_20260718.tar.zst
```

签名验证失败表示备份被篡改，禁止恢复，触发 CRITICAL 告警。

### 5.3 完整性校验流程

恢复前的完整校验流程：

1. **存在性**：检查备份文件、SHA256、签名、manifest 是否齐全。
2. **SHA256**：重新计算并比对。
3. **签名**：验证数字签名。
4. **manifest**：解析 manifest，校验内容清单。
5. **测试解压**：可选执行 `tar -tzf` 测试压缩包完整性。
6. **校验日志**：所有校验结果写入 `/var/log/airy/backup_verify.log`。

任一环节失败则恢复中止，触发 CRITICAL 告警。

### 5.4 定期校验

audit_d 每周日凌晨 05:00 执行全量备份校验：

- 对所有保留的全量备份执行 SHA256 + 签名校验。
- 抽取 10% 的增量备份执行校验。
- 校验结果写入 `/var/lib/airy/backup/integrity_report.json`。
- 发现损坏立即触发 CRITICAL 告警，并尝试从远程存储恢复。

---

## 6. 恢复流程

### 6.1 恢复流程总览

```
[触发恢复] ──> [选择备份] ──> [完整性校验] ──> [DSL 模式启动]
                                                  │
                                                  ▼
                          ┌────────────────────────────────────┐
                          │  分阶段恢复                         │
                          ├────────────────────────────────────┤
                          │ 1. config_d 读取配置备份        │
                          │ 2. sec_d 恢复密钥库                  │
                          │ 3. mem_d 恢复 L4 记忆（深度压缩）    │
                          │ 4. mem_d 恢复 L3 记忆（增量合并）    │
                          │ 5. macro_d 重启 Agent           │
                          │ 6. audit_d 恢复监控数据              │
                          └────────────────────────────────────┘
                                                  │
                                                  ▼
                                          [恢复验证]
                                                  │
                                                  ▼
                                          [DSL 退出]
```

### 6.2 恢复触发

恢复可由以下方式触发：

- **自动触发**：P0 事件响应流程判定需要恢复（如文件系统损坏）。
- **人工触发**：运维人员通过 `airyctl recover` 命令。
- **演练触发**：定期演练（每月一次），由 macro_d 自动执行。

```bash
# 列出可用备份
airyctl backup list

# 从全量备份恢复
airyctl recover --backup-id FULL-20260718-001

# 从全量 + 增量恢复到指定时间点
airyctl recover --point-in-time "2026-07-18T14:00:00Z"

# 仅恢复特定 Agent
airyctl recover --agent-id 42 --backup-id FULL-20260718-001
```

### 6.3 config_d 读取备份

恢复的第一阶段是恢复配置：

1. config_d 接收恢复指令，读取备份中的 `/etc/airy/` 内容。
2. 校验配置文件完整性（SHA256 与备份 manifest 一致）。
3. 备份当前（损坏的）配置到 `/etc/airy.corrupt.<timestamp>/`。
4. 替换为备份中的配置。
5. 触发 SIGHUP 通知所有 daemon 重新加载配置。
6. config_d 上报恢复完成。

### 6.4 mem_d 恢复记忆

记忆恢复分两个子阶段：

#### 6.4.1 L4 恢复

1. mem_d 读取备份中的 `memory_l4.tar.zst`。
2. 解压到 `/var/lib/airy/memory/l4/`。
3. 校验每个 Agent 的 L4 目录 SHA256。
4. 加载 L4 索引到内存。

#### 6.4.2 L3 恢复

1. mem_d 读取全量备份中的 L3 数据。
2. 按顺序应用增量备份，恢复到目标时间点。
3. 每应用一个增量备份，校验块 SHA256。
4. 加载 L3 索引到内存。
5. 触发 L1/L2 重建（按需，从 L3 提取热数据）。

### 6.5 macro_d 重启 Agent

记忆恢复完成后，macro_d 逐步重启 Agent：

1. 读取备份中的 Agent 元数据与状态。
2. 按优先级排序（管理 Agent > 业务 Agent）。
3. 逐个重启 Agent，每次重启间隔 5 秒（避免雪崩）。
4. 每个 Agent 重启后等待 readiness=1，超时（60s）则跳过。
5. 全部 Agent 重启完成后，触发恢复验证。

### 6.6 恢复验证

恢复验证由 audit_d 协调：

1. **健康检查**：所有 daemon 的 health_score ≥80。
2. **Agent 检查**：所有 Agent 状态为 running 或 paused（stopped 视为正常）。
3. **记忆完整性**：mem_d 抽样验证 L3/L4 数据可读。
4. **IPC 测试**：发起测试 IPC，验证延迟正常。
5. **监控恢复**：audit_d 重新加载监控数据，确认指标正常。
6. 验证通过后，macro_d 退出 DSL 模式。

### 6.7 恢复失败处理

- 任一阶段失败：暂停恢复，触发 CRITICAL 告警。
- 失败时不回滚（避免数据进一步损坏），保留当前状态供人工介入。
- 运维人员可通过 `airyctl recover --resume` 继续恢复，或 `--rollback` 回滚到失败前状态。

---

## 7. 灾难恢复

### 7.1 灾难场景

灾难恢复针对以下严重场景：

| 场景 | 影响 | 恢复策略 |
|------|------|----------|
| 系统盘损坏 | 操作系统不可启动 | 从安装介质重装 + 从备份恢复 |
| /var/lib 损坏 | Agent 与记忆数据丢失 | 从远程备份恢复 |
| 全节点故障 | 节点不可用 | 在备用节点恢复（与 150-cloudnative 联动） |
| 数据中心故障 | 多节点不可用 | 异地灾备恢复 |
| 勒索软件 | 数据被加密 | 从离线备份恢复 |

### 7.2 [DSL] 降级模式启动

灾难恢复时，系统以 [DSL] 降级模式启动：

1. **最小启动**：仅 macro_d / logger_d / config_d / sec_d / audit_d / mem_d 启动。
2. **配置加载**：config_d 加载降级模式专用配置 `airy-dsl.conf`。
3. **只读挂载**：/var/lib 以只读模式挂载，避免进一步损坏。
4. **备份挂载**：将备份存储挂载到 `/mnt/backup/`。
5. **恢复准备**：audit_d 验证备份完整性，准备恢复。

### 7.3 最小可运行子集

降级模式下，最小可运行子集包括：

| 组件 | 状态 | 用途 |
|------|------|------|
| macro_d | 运行 | 协调恢复 |
| logger_d | 运行（WARN 级） | 记录恢复日志 |
| config_d | 运行 | 加载配置 |
| sec_d | 运行（最高警戒） | 保护恢复过程 |
| audit_d | 运行 | 恢复监控数据 |
| mem_d | 运行 | 恢复记忆 |
| gateway_d | 仅管理网络 | 恢复运维访问 |
| 其他 daemon | 停止 | 待恢复后启动 |

### 7.4 逐步恢复

降级模式下按以下顺序逐步恢复：

1. **配置恢复**：config_d 恢复 `/etc/airy/`。
2. **密钥恢复**：sec_d 恢复密钥库。
3. **记忆恢复**：mem_d 恢复 L4 → L3。
4. **核心 daemon 启动**：sched_d、vfs_d、net_d 启动。
5. **管理 Agent 启动**：ID=1 的管理 Agent 启动。
6. **业务 Agent 启动**：按优先级逐步启动。
7. **辅助 daemon 启动**：cogn_d、dev_d 启动。
8. **退出降级模式**：macro_d 验证全部就绪，退出 DSL。

每一步完成后等待 30 秒稳定期，再进入下一步。完整恢复耗时目标 ≤15 分钟。

### 7.5 异地灾备

对于关键业务场景，建议配置异地灾备：

- **备份复制**：每日全量备份自动复制到异地（通过 S3 或 RSYNC）。
- **备用节点**：异地维护一台热备节点，每小时同步配置与监控数据。
- **故障切换**：主节点故障时，DNS 切换到备用节点，触发完整恢复流程。
- **演练**：每季度执行一次异地灾备演练。

---

## 8. 备份恢复演练

### 8.1 演练计划

| 类型 | 频率 | 范围 | 参与者 |
|------|------|------|--------|
| 部分恢复演练 | 每周 | 单 Agent 恢复 | 自动 |
| 全量恢复演练 | 每月 | 全系统恢复（演练环境） | 运维 |
| 灾难恢复演练 | 每季度 | 异地灾备切换 | 运维 + 开发 |
| 真实故障复盘 | 事件后 | 针对性演练 | 全员 |

### 8.2 演练流程

1. **准备**：在演练环境准备备份，确保不影响生产。
2. **执行**：按恢复流程执行，记录每步耗时。
3. **验证**：验证恢复后的系统功能完整性。
4. **评估**：评估 RPO 与 RTO 是否达标。
5. **改进**：发现的问题录入改进任务。

### 8.3 演练报告

演练完成后生成报告，归档到 `/var/lib/airy/backup/drills/`：

- 演练时间、参与者、范围。
- 各阶段耗时（备份读取、校验、恢复、验证）。
- 实际 RPO 与 RTO。
- 发现的问题与改进措施。

---

## 9. 配置参数

备份恢复体系的关键配置参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `airy_backup.full_schedule` | `01:00` | 全量备份时间 |
| `airy_backup.incremental_interval_min` | 60 | 增量备份间隔（分钟） |
| `airy_backup.retention_full_d` | 30 | 全量保留（天） |
| `airy_backup.retention_incremental_d` | 7 | 增量保留（天） |
| `airy_backup.retention_weekly_w` | 12 | 周备份保留（周） |
| `airy_backup.retention_monthly_m` | 12 | 月备份保留（月） |
| `airy_backup.remote.enabled` | false | 是否启用远程备份 |
| `airy_backup.remote.type` | s3 | 远程后端类型 |
| `airy_backup.remote.endpoint` | "" | 远程端点 |
| `airy_backup.remote.encrypt` | true | 远程备份加密 |
| `airy_backup.verify_weekly` | true | 每周完整性校验 |
| `airy_backup.drill_monthly` | true | 每月演练 |
| `airy_backup.max_recovery_min` | 15 | RTO 目标（分钟） |

---

## 10. 安全考虑

### 10.1 备份访问控制

- 本地备份目录 `/var/lib/airy/backup/` 权限 0750，属主 `airy:airy-backup`。
- 仅 audit_d 与 `airyctl backup` 命令可读写。
- 远程备份访问凭证由 sec_d 托管，audit_d 启动时获取。

### 10.2 备份加密

- 远程备份强制 AES-256-CBC 加密。
- 加密密钥独立存储，不出现在备份中。
- 密钥轮换：每 90 天由 sec_d 触发，旧密钥保留用于历史备份解密。

### 10.3 恢复授权

- 恢复操作需 `airy-backup` 组成员身份。
- 全量恢复需 `--confirm` 显式确认。
- 恢复操作完整记录到 `/var/log/airy/recovery.log`，含操作者、时间、备份 ID。

### 10.4 防篡改

- 备份文件 append-only（通过 chattr +a）。
- SHA256 与签名双重保护，任何篡改可被检测。
- sec_d 监控备份目录，非 audit_d 进程的修改触发 CRITICAL 告警。

---

## 11. 与其他子系统的协作

### 11.1 与事件响应

P0 事件可能触发备份恢复，详见 [05-incident-response.md](05-incident-response.md) 第 4 节。

### 11.2 与记忆运维

L3/L4 备份恢复与记忆运维紧密相关，详见 [09-memory-operations.md](09-memory-operations.md)。

### 11.3 与 systemd 集成

恢复流程中 daemon 的启停通过 systemd 单元控制，详见 [07-systemd-integration.md](07-systemd-integration.md)。

### 11.4 与 150-cloudnative

跨节点恢复与异地灾备由 150-cloudnative 模块支持，详见 [../150-cloudnative/README.md](../150-cloudnative/README.md)。

---

## 12. 版本演进

### 12.1 v0.1.1 → v1.0.1 变更

- 新增"分阶段恢复"能力，避免恢复期雪崩。
- 备份压缩比从 3:1 提升至 5:1（改用 zstd level 12）。
- 增量备份块大小从 8KB 调整为 4KB，降低 RPO。
- 新增每月演练自动化支持。

### 12.2 后续规划（v1.1.0）

- 支持备份去重（跨 Agent 的相同记忆块共享存储）。
- 引入持续数据保护（CDP），RPO ≤1 分钟。
- 备份云原生化（Kubernetes Operator 管理）。

---

## 13. 相关文档

- [03-monitoring.md](03-monitoring.md)：监控体系，备份恢复过程监控。
- [05-incident-response.md](05-incident-response.md)：事件响应，触发备份恢复。
- [07-systemd-integration.md](07-systemd-integration.md)：systemd 集成，daemon 启停。
- [09-memory-operations.md](09-memory-operations.md)：记忆运维，L3/L4 备份恢复。
- [10-devstation.md](10-devstation.md)：开发运维工作站，备份恢复可视化。
- [../150-cloudnative/README.md](../150-cloudnative/README.md)：云原生，跨节点恢复。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
