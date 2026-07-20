Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 记忆运维（Memory Operations）

> **文档定位**：AirymaxOS v1.0.1 运维分册 · 记忆运维设计文档，定义 MemoryRovol L1-L4 四层记忆层级的运维策略、碎片整理、迁移与安全擦除\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[100-operations README](README.md)

---

## 1. 概述

### 1.1 设计目标

MemoryRovol 是 AirymaxOS 的多层记忆系统，为每个 Agent 提供 L1-L4 四层记忆能力。记忆运维体系设计目标如下：

1. **层级透明**：Agent 通过统一接口访问记忆，无需感知层级差异。
2. **自动流转**：记忆在层级间自动流转（L1→L2→L3→L4），无需人工干预。
3. **持久化保证**：L3/L4 持久化存储，进程重启不丢失。
4. **运维可控**：碎片整理、迁移、擦除等运维操作可执行、可观测。
5. **安全合规**：敏感记忆支持安全擦除，满足数据合规要求。

### 1.2 MemoryRovol 四层模型回顾

| 层级 | 名称 | 位置 | 容量（单 Agent） | 访问延迟 | 持久化 |
|------|------|------|------------------|----------|--------|
| L1 | 工作记忆 | 进程内存 | 256MB | <1μs | 否 |
| L2 | 短期记忆 | 进程内存 + 共享内存 | 1GB | <10μs | 否 |
| L3 | 长期记忆 | /var/lib/airy/memory/l3/ | 1-10GB | <1ms | 是 |
| L4 | 经验记忆 | /var/lib/airy/memory/l4/ | 100MB-1GB | <10ms | 是（深度压缩） |

层级间数据流向：

```
[Agent 写入] ──> L1 ──(LRU 淘汰)──> L2 ──(压缩归档)──> L3 ──(深度压缩)──> L4
                                                                  │
[Agent 读取] <── L1 <──(按需加载)<── L2 <──(按需加载)<── L3 <──(按需加载)<── L4
```

### 1.3 与其他子系统的关系

| 子系统 | 在记忆运维中的角色 |
|--------|---------------------|
| mem_d | 记忆运维主导模块，执行所有记忆操作 |
| macro_d | 协调记忆迁移、监控记忆健康 |
| sched_d | 为记忆整理任务分配 CPU 资源 |
| vfs_d | 提供 L3/L4 文件系统支持 |
| sec_d | 记忆访问授权、安全擦除 |
| audit_d | 记忆运维事件记录 |
| config_d | 记忆配置管理 |
| 备份恢复 | L3/L4 备份与恢复 |

### 1.4 与技术选型的契合

- **alloc_pages + mmap**：L1/L2 使用 alloc_pages 分配物理内存，mmap 映射到 Agent 地址空间，避免 DMA 一致性内存开销。
- **IORING_OP_URING_CMD**：记忆读写复用 io_uring 异步通道，避免阻塞。
- **纯 C LSM**：记忆文件访问受 LSM hook 保护，仅授权 Agent 可读。
- **IRON-9 v3 [IND] 层**：记忆索引由 [IND]（Index Layer）层管理，支持快速检索。

---

## 2. L1 工作记忆运维

### 2.1 L1 特征

L1 是 Agent 的"工作记忆"，存放当前正在处理的数据：

- **存储位置**：Agent 进程内存（通过 alloc_pages 分配）。
- **默认容量**：每 Agent 256MB。
- **访问延迟**：<1μs（直接内存访问）。
- **持久化**：否（进程退出即丢失）。
- **数据类型**：当前对话上下文、临时变量、活跃任务状态。

### 2.2 L1 容量管理

mem_d 监控每个 Agent 的 L1 使用情况：

- **采样频率**：1 秒一次。
- **告警阈值**：使用率 >90% 持续 30s 触发 WARNING。
- **自动调整**：使用率持续 >95% 时，mem_d 评估是否扩大 L1 上限（受系统总内存约束）。

### 2.3 L1 LRU 淘汰

当 L1 使用率达到上限（默认 256MB），mem_d 触发 LRU（Least Recently Used）淘汰：

1. **扫描**：mem_d 扫描 L1 中的记忆项，按最后访问时间排序。
2. **选择**：选取最久未访问的 10% 记忆项。
3. **迁移**：将选中项迁移到 L2（压缩后存储）。
4. **更新索引**：[IND] 层更新索引，记录项的新位置。
5. **释放**：从 L1 释放对应内存。

淘汰触发频率监控：

- **正常**：每分钟 <10 次淘汰事件。
- **WARNING**：每分钟 10-100 次淘汰事件。
- **CRITICAL**：每分钟 >100 次淘汰事件，建议扩大 L1 容量或优化 Agent 访问模式。

### 2.4 L1 监控指标

| 指标 | 说明 |
|------|------|
| `agent_N_mem_l1_bytes` | L1 当前占用 |
| `agent_N_mem_l1_max_bytes` | L1 容量上限 |
| `agent_N_mem_l1_evict_total` | L1 淘汰次数累计 |
| `agent_N_mem_l1_evict_rate_per_min` | L1 淘汰速率 |
| `agent_N_mem_l1_hit_rate_pct` | L1 命中率（访问 L1 / 总访问） |

---

## 3. L2 短期记忆运维

### 3.1 L2 特征

L2 是 Agent 的"短期记忆"，存放近期使用但不在当前处理范围的数据：

- **存储位置**：进程内存 + 共享内存（多 Agent 可共享只读部分）。
- **默认容量**：每 Agent 1GB。
- **访问延迟**：<10μs（可能涉及共享内存访问）。
- **持久化**：否。
- **数据类型**：近期对话历史、临时计算结果、缓存数据。

### 3.2 L2 容量管理

- **采样频率**：5 秒一次。
- **告警阈值**：使用率 >90% 持续 60s 触发 WARNING。
- **自动调整**：使用率持续 >95% 时，扩大 L2 上限（受系统总内存约束，上限 2GB）。

### 3.3 L2 压缩归档

当 L2 使用率达到上限，mem_d 触发压缩归档到 L3：

1. **选择**：选取最久未访问的 20% 记忆项。
2. **压缩**：使用 zstd 压缩（level 6），典型压缩比 3:1。
3. **写入 L3**：将压缩后的记忆块写入 `/var/lib/airy/memory/l3/<agent_id>/`。
4. **更新索引**：[IND] 层更新索引，记录 L3 位置。
5. **释放 L2**：从 L2 释放对应内存。

### 3.4 L2 共享内存优化

L2 的部分只读数据（如通用知识库缓存）可由多个 Agent 共享：

- **共享区域**：`/dev/shm/airy/l2_shared/`，由 mem_d 管理。
- **共享条件**：记忆项标记为 `shareable=true` 且内容相同。
- **去重**：mem_d 通过 SHA256 检测相同内容，自动去重。
- **引用计数**：共享区域维护引用计数，所有引用者释放后才回收。

共享内存可显著降低多 Agent 场景下的总内存占用（实测节省 30-50%）。

### 3.5 L2 监控指标

| 指标 | 说明 |
|------|------|
| `agent_N_mem_l2_bytes` | L2 当前占用 |
| `agent_N_mem_l2_max_bytes` | L2 容量上限 |
| `agent_N_mem_l2_archive_total` | L2 归档到 L3 次数 |
| `agent_N_mem_l2_archive_bytes` | L2 归档字节数 |
| `agent_N_mem_l2_shared_bytes` | L2 共享内存占用 |
| `agent_N_mem_l2_hit_rate_pct` | L2 命中率 |

---

## 4. L3 长期记忆运维

### 4.1 L3 特征

L3 是 Agent 的"长期记忆"，存放持久化的历史数据：

- **存储位置**：`/var/lib/airy/memory/l3/<agent_id>/`。
- **默认容量**：每 Agent 1-10GB（按需扩展）。
- **访问延迟**：<1ms（磁盘 IO）。
- **持久化**：是。
- **数据类型**：长期对话历史、知识库、训练数据。

### 4.2 L3 文件结构

```
/var/lib/airy/memory/l3/<agent_id>/
├── index.airyidx          # 索引文件（[IND] 层管理）
├── data/
│   ├── 000001.airyblk     # 数据块（4MB each）
│   ├── 000002.airyblk
│   ├── ...
│   └── 000NNN.airyblk
├── meta.json              # 元数据（创建时间、容量、版本）
└── checksum.sha256        # 完整性校验
```

每个数据块（`.airyblk`）大小 4MB，包含多个压缩后的记忆项。块大小选择平衡了 IO 效率与碎片问题。

### 4.3 L3 增量备份

L3 采用增量备份（每小时）：

- **变更追踪**：mem_d 维护"脏块列表"，记录自上次备份后修改的数据块。
- **备份内容**：仅备份脏块，生成 `incremental_<timestamp>.tar.zst`。
- **备份触发**：每小时整点由 audit_d 协调。
- **保留期**：7 天（详见 [06-backup-recovery.md](06-backup-recovery.md)）。

### 4.4 L3 索引优化

[IND] 层维护 L3 索引，支持快速检索：

- **索引结构**：B+ tree，key 为记忆项 ID，value 为块位置 + 偏移。
- **索引缓存**：索引热部分缓存在内存（每 Agent 默认 16MB）。
- **索引重建**：若索引损坏，mem_d 可通过扫描数据块重建（耗时较长）。
- **索引压缩**：索引文件每周压缩一次，减少磁盘占用。

### 4.5 L3 监控指标

| 指标 | 说明 |
|------|------|
| `agent_N_mem_l3_bytes` | L3 当前占用 |
| `agent_N_mem_l3_blocks` | L3 数据块数 |
| `agent_N_mem_l3_index_size_bytes` | L3 索引大小 |
| `agent_N_mem_l3_read_ops_per_sec` | L3 读 IOPS |
| `agent_N_mem_l3_write_ops_per_sec` | L3 写 IOPS |
| `agent_N_mem_l3_hit_rate_pct` | L3 命中率 |
| `airy_mem_l3_total_bytes` | 全系统 L3 总占用 |

---

## 5. L4 经验记忆运维

### 5.1 L4 特征

L4 是 Agent 的"经验记忆"，存放深度压缩的长期经验：

- **存储位置**：`/var/lib/airy/memory/l4/<agent_id>/`。
- **默认容量**：每 Agent 100MB-1GB。
- **访问延迟**：<10ms（深度压缩需解压）。
- **持久化**：是（深度压缩）。
- **数据类型**：训练经验、长期模式、稀有知识。

### 5.2 L4 深度压缩

L4 采用深度压缩算法：

- **算法**：zstd level 19（极高压缩比）。
- **压缩比**：典型 10:1（相比原始数据）。
- **压缩时机**：L3 数据归档到 L4 时压缩。
- **解压缓存**：L4 解压结果缓存到 L3（按需），避免重复解压。

### 5.3 L4 全量备份

L4 由于变更较少且压缩比高，采用每日全量备份：

- **备份内容**：整个 `/var/lib/airy/memory/l4/<agent_id>/` 目录。
- **备份触发**：每日凌晨 01:00 由 audit_d 协调。
- **保留期**：30 天（详见 [06-backup-recovery.md](06-backup-recovery.md)）。

### 5.4 L4 归档策略

L4 数据根据访问频率进一步归档：

- **热数据**：最近 30 天访问过的 L4 项，保留在原位置。
- **温数据**：30-180 天未访问的 L4 项，迁移到 `/var/lib/airy/memory/l4.archive/<agent_id>/`。
- **冷数据**：180 天未访问的 L4 项，可导出到外部存储（如 S3 Glacier）。

归档由 mem_d 每周扫描一次，自动执行。

### 5.5 L4 监控指标

| 指标 | 说明 |
|------|------|
| `agent_N_mem_l4_bytes` | L4 当前占用 |
| `agent_N_mem_l4_compressed_bytes` | L4 压缩后占用 |
| `agent_N_mem_l4_compression_ratio` | L4 压缩比 |
| `agent_N_mem_l4_decompress_total` | L4 解压次数 |
| `agent_N_mem_l4_archive_total` | L4 归档到冷存储次数 |
| `airy_mem_l4_total_bytes` | 全系统 L4 总占用 |

---

## 6. 记忆碎片整理

### 6.1 碎片产生原因

随着记忆的写入、删除、迁移，L3/L4 文件可能产生碎片：

- **外部碎片**：数据块间出现小间隙，无法利用。
- **内部碎片**：数据块内部空间未充分利用（如删除部分项后）。
- **索引碎片**：索引文件膨胀，查询效率下降。

### 6.2 defrag 触发条件

mem_d 定期评估碎片率，满足以下任一条件触发 defrag：

- L3 碎片率 >30%（碎片空间 / 总空间）。
- L3 数据块数 >10000 且平均利用率 <70%。
- L3 索引大小 >数据大小的 10%。
- 距离上次 defrag >7 天。

### 6.3 defrag 流程

1. **冻结写入**：mem_d 暂停目标 Agent 的 L3 写入（读不受影响）。
2. **扫描**：扫描所有数据块，统计有效记忆项。
3. **重排**：将有效记忆项紧凑排列到新数据块。
4. **更新索引**：[IND] 层重建索引，记录新位置。
5. **释放**：删除旧数据块，释放磁盘空间。
6. **解冻写入**：恢复 L3 写入。

defrag 在后台线程执行，不影响 Agent 正常运行（仅短暂写入延迟）。

### 6.4 defrag 调度

为避免影响业务，defrag 调度策略：

- **优先级**：SCHED_IDLE（仅在 CPU 空闲时执行）。
- **时间窗口**：默认仅在 02:00-05:00 执行（业务低峰期）。
- **并发限制**：同时只对 1 个 Agent 执行 defrag。
- **超时**：单次 defrag 最长 30 分钟，超时则暂停下次继续。

### 6.5 defrag 监控

| 指标 | 说明 |
|------|------|
| `airy_mem_defrag_running` | 当前是否在执行 defrag（0/1） |
| `airy_mem_defrag_agent_id` | 当前 defrag 的 Agent ID |
| `airy_mem_defrag_progress_pct` | defrag 进度 |
| `airy_mem_defrag_freed_bytes` | defrag 释放的空间 |
| `airy_mem_defrag_duration_s` | defrag 耗时 |
| `airy_mem_fragmentation_pct` | 当前碎片率 |

---

## 7. 记忆迁移

### 7.1 记忆迁移场景

记忆迁移用于以下场景：

| 场景 | 触发方式 | 迁移类型 |
|------|----------|----------|
| Agent 跨节点迁移 | 自动（详见 [08-agent-operations.md](08-agent-operations.md)） | 全量迁移 |
| Agent 合并 | 人工 | 选择性迁移 |
| Agent 克隆 | 人工 | 快照迁移 |
| 记忆共享 | 人工 | 部分迁移 |

### 7.2 跨 Agent 记忆转移协议

跨 Agent 记忆转移遵循以下协议：

```
[源 Agent] ──── 1. 请求迁移 ────> [mem_d]
                                      │
                                      ▼
                              2. 校验权限（sec_d）
                                      │
                                      ▼
                              3. 选择记忆项
                                      │
                                      ▼
                              4. 复制记忆数据
                                      │
                                      ▼
                              5. 更新目标 Agent 索引
                                      │
                                      ▼
                              6. 通知源/目标 Agent
                                      │
                                      ▼
[目标 Agent] <─── 7. 迁移完成通知 ─── [mem_d]
```

### 7.3 迁移权限

记忆迁移受 sec_d 严格管控：

- **同用户 Agent**：默认允许，仅需源 Agent 同意。
- **跨用户 Agent**：需源与目标 Agent 都同意，且 sec_d 审批。
- **敏感记忆**：标记为 `sensitive=true` 的记忆项不可迁移，仅可在原 Agent 内访问。

### 7.4 迁移命令

```bash
# 迁移特定记忆项
airyctl memory transfer \
    --from-agent 42 \
    --to-agent 43 \
    --items "item-1,item-2,item-3"

# 迁移整个层级
airyctl memory transfer \
    --from-agent 42 \
    --to-agent 43 \
    --level L3

# 克隆 Agent 记忆（不删除源）
airyctl memory clone \
    --from-agent 42 \
    --to-agent 43 \
    --level L3,L4
```

### 7.5 迁移监控

| 指标 | 说明 |
|------|------|
| `airy_mem_migrate_running` | 当前是否在迁移 |
| `airy_mem_migrate_bytes` | 迁移字节数 |
| `airy_mem_migrate_items` | 迁移记忆项数 |
| `airy_mem_migrate_duration_s` | 迁移耗时 |

---

## 8. 记忆安全擦除

### 8.1 擦除场景

记忆安全擦除用于以下场景：

- Agent 终止时，按配置擦除敏感记忆。
- 合规审计要求删除特定数据。
- 用户行使"被遗忘权"（GDPR）。
- 安全事件后清除被污染的记忆。

### 8.2 擦除级别

AirymaxOS 支持三级擦除：

| 级别 | 方式 | 耗时 | 安全性 |
|------|------|------|--------|
| L1 | 快速擦除（unlink） | <1s | 低（仅删除索引） |
| L2 | 覆写擦除（zero fill） | 中 | 中（数据被覆盖） |
| L3 | 安全擦除（DoD 5220.22-M） | 长 | 高（多次覆写） |
| L4 | 加密擦除（密钥销毁） | <1s | 高（数据不可解密） |

### 8.3 安全擦除流程（L3）

L3 安全擦除遵循 DoD 5220.22-M 标准：

1. **选择**：选定要擦除的记忆项或整个 Agent。
2. **覆写 1**：用 0x00 覆写所有数据块。
3. **覆写 2**：用 0xFF 覆写所有数据块。
4. **覆写 3**：用随机数据覆写所有数据块。
5. **校验**：验证所有字节已覆写。
6. **unlink**：删除数据块文件。
7. **索引清除**：[IND] 层删除对应索引项。
8. **审计**：sec_d 记录擦除事件，包含擦除者、时间、范围。

### 8.4 加密擦除流程（L4）

L4 由于深度压缩且数据量大，采用加密擦除：

1. **加密存储**：L4 数据在写入时已用 AES-256-XTS 加密，密钥由 sec_d 托管。
2. **擦除**：销毁加密密钥（sec_d 安全删除密钥）。
3. **结果**：L4 数据虽物理存在，但无法解密，等效于擦除。
4. **可选**：擦除密钥后再删除数据文件，双重保障。

加密擦除的优势是耗时极短（<1s），适合大规模数据擦除。

### 8.5 擦除配置

每个 Agent 可配置擦除策略：

```yaml
# /etc/airy/agents/<id>/erase_policy.yaml
on_terminate:
  l1: quick              # 快速擦除
  l2: overwrite          # 覆写擦除
  l3: secure             # 安全擦除
  l4: crypto             # 加密擦除

on_request:
  default_level: secure  # 用户请求时默认安全擦除
  audit: true            # 擦除必须审计

sensitive_items:
  - pattern: "*password*"
    level: secure
  - pattern: "*credit_card*"
    level: secure
```

### 8.6 擦除命令

```bash
# 擦除特定记忆项
airyctl memory erase --agent-id 42 --items "item-1,item-2" --level secure

# 擦除整个 Agent 记忆
airyctl memory erase --agent-id 42 --all --level secure

# 加密擦除 L4
airyctl memory erase --agent-id 42 --level L4 --method crypto
```

所有擦除操作记录到 `/var/log/airy/memory_erase.log`，永久保留。

---

## 9. 记忆运维命令汇总

### 9.1 查询命令

```bash
airyctl memory status <agent_id>            # 查看记忆状态
airyctl memory stats <agent_id>             # 详细统计
airyctl memory list <agent_id> --level L3   # 列出记忆项
airyctl memory search <agent_id> "keyword"  # 搜索记忆
```

### 9.2 整理命令

```bash
airyctl memory defrag <agent_id>            # 触发 defrag
airyctl memory defrag --all                 # 全 Agent defrag
airyctl memory defrag status                # 查看 defrag 状态
```

### 9.3 迁移命令

```bash
airyctl memory transfer [options]           # 迁移记忆
airyctl memory clone [options]              # 克隆记忆
airyctl memory share [options]              # 共享记忆
```

### 9.4 擦除命令

```bash
airyctl memory erase [options]              # 擦除记忆
airyctl memory erase status                 # 查看擦除状态
airyctl memory erase audit                  # 查看擦除审计
```

---

## 10. 配置参数

记忆运维的关键配置参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `airy_memory.l1_default_mb` | 256 | 默认 L1 大小（MB） |
| `airy_memory.l1_max_mb` | 512 | L1 最大上限（MB） |
| `airy_memory.l2_default_mb` | 1024 | 默认 L2 大小（MB） |
| `airy_memory.l2_max_mb` | 2048 | L2 最大上限（MB） |
| `airy_memory.l3_block_kb` | 4096 | L3 数据块大小（KB） |
| `airy_memory.l3_index_cache_mb` | 16 | L3 索引缓存（MB） |
| `airy_memory.l4_compress_level` | 19 | L4 压缩级别 |
| `airy_memory.defrag_threshold_pct` | 30 | defrag 碎片率阈值 |
| `airy_memory.defrag_window` | "02:00-05:00" | defrag 时间窗口 |
| `airy_memory.defrag_max_duration_min` | 30 | 单次 defrag 最长（分钟） |
| `airy_memory.erase_default_level` | secure | 默认擦除级别 |
| `airy_memory.erase_audit` | true | 擦除审计 |
| `airy_memory.shared_l2_enabled` | true | 启用 L2 共享内存 |

---

## 11. 安全考虑

### 11.1 记忆访问控制

- 每个 Agent 仅能访问自己的记忆（L1-L4）。
- 跨 Agent 访问需显式授权（通过 `airyctl memory share` 或迁移协议）。
- 记忆文件权限 0600，属主 `airy:airy`。
- 纯 C LSM 强制访问控制，绕过权限即触发安全告警。

### 11.2 记忆加密

- L4 数据强制加密（AES-256-XTS）。
- L3 数据可选加密（默认关闭，性能考虑）。
- 加密密钥由 sec_d 托管，独立存储。
- 密钥轮换：每 90 天一次，不影响数据可用性。

### 11.3 记忆审计

- 所有记忆访问（读/写/删除）记录到 audit_d。
- 敏感记忆项（标记 `sensitive=true`）的访问触发额外审计。
- 审计日志保留 90 天，安全擦除日志永久保留。

### 11.4 记忆备份安全

- L3/L4 备份远程传输时强制加密。
- 备份文件 SHA256 + 签名校验（详见 [06-backup-recovery.md](06-backup-recovery.md)）。
- 备份恢复需 sec_d 授权。

---

## 12. 与其他子系统的协作

### 12.1 与监控体系

记忆监控指标详见 [03-monitoring.md](03-monitoring.md) 第 3.2.2 节。

### 12.2 与备份恢复

L3/L4 备份恢复详见 [06-backup-recovery.md](06-backup-recovery.md) 第 2.4 节。

### 12.3 与 Agent 运维

Agent 记忆相关运维详见 [08-agent-operations.md](08-agent-operations.md) 第 4.2 节。

### 12.4 与 110-security

记忆安全擦除与审计详见 [../110-security/README.md](../110-security/README.md)。

---

## 13. 版本演进

### 13.1 v0.1.1 → v1.0.1 变更

- L2 共享内存优化，多 Agent 场景节省 30-50% 内存。
- L4 加密擦除支持，大规模擦除 <1s。
- defrag 引入 SCHED_IDLE 调度，业务无感。
- 记忆迁移协议增加敏感项保护。

### 13.2 后续规划（v1.1.0）

- L3 数据块支持在线扩容（无需 defrag）。
- 引入向量索引（FAISS 集成），支持语义检索。
- L4 支持差分压缩，进一步降低存储。

---

## 14. 相关文档

- [03-monitoring.md](03-monitoring.md)：监控体系，记忆监控指标。
- [05-incident-response.md](05-incident-response.md)：事件响应，记忆相关事件。
- [06-backup-recovery.md](06-backup-recovery.md)：备份恢复，L3/L4 备份。
- [08-agent-operations.md](08-agent-operations.md)：Agent 运维，记忆与 Agent 关系。
- [10-devstation.md](10-devstation.md)：开发运维工作站，记忆可视化。
- [../110-security/README.md](../110-security/README.md)：安全运维，记忆安全擦除。

---

*本文档版权归 SPHARX Ltd. 所有，未经授权不得转载。AirymaxOS 是 SPHARX 旗下的智能体操作系统产品。*
