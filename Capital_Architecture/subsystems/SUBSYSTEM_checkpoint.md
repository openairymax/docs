# SUBSYSTEM_checkpoint — 任务检查点

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/daemon/common/` (checkpoint.h)  
**负责人**: Team C (基础设施)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

checkpoint 子系统是 Airymax 的任务持久化与恢复机制，确保长任务在进程崩溃或重启后可从最近检查点恢复，避免重复计算和状态丢失。

### 1.1 核心职责

| 职责 | 说明 |
|------|------|
| **检查点创建** | 为任务创建包含完整状态的检查点 |
| **检查点保存** | 持久化检查点到存储 |
| **检查点恢复** | 从检查点恢复任务状态并继续执行 |
| **检查点验证** | 校验检查点数据完整性（checksum） |
| **检查点列表** | 列出指定任务的所有检查点 |
| **自动检查点** | 定时自动保存检查点（通过 Hook） |
| **检查点清理** | 按时间或数量清理过期检查点 |
| **快照** | 创建/恢复任务完整快照 |
| **统计** | 检查点操作统计（成功率、大小等） |

### 1.2 检查点状态

| 状态 | 枚举值 | 说明 |
|------|--------|------|
| Pending | `CHECKPOINT_STATE_PENDING` | 待保存 |
| Completed | `CHECKPOINT_STATE_COMPLETED` | 已保存 |
| Failed | `CHECKPOINT_STATE_FAILED` | 保存失败 |
| Invalid | `CHECKPOINT_STATE_INVALID` | 无效/损坏 |

### 1.3 不在范围内的职责

- 不负责任务执行逻辑（由 coreloopthree 负责）
- 不负责持久化存储实现（由 heapstore 负责）
- 不负责数据备份策略（由运维层负责）

---

## 2. 输入/输出接口

### 2.1 检查点管理

```c
// 创建检查点
agentos_error_t agentos_checkpoint_create(
    const char *task_id, const char *session_id,
    uint64_t sequence_num, const char *state_json,
    char **completed_nodes, size_t completed_count,
    char **pending_nodes, size_t pending_count,
    agentos_task_checkpoint_t **out_checkpoint);

// 保存/恢复/删除
agentos_error_t agentos_checkpoint_save(agentos_task_checkpoint_t *checkpoint);
agentos_error_t agentos_checkpoint_restore(const char *task_id, uint64_t sequence_num,
                                           agentos_task_checkpoint_t **out_checkpoint);
agentos_error_t agentos_checkpoint_delete(const char *task_id, uint64_t sequence_num);

// 列表
agentos_error_t agentos_checkpoint_list(const char *task_id,
                                        agentos_task_checkpoint_t ***out_checkpoints,
                                        size_t *out_count);

// 验证与销毁
agentos_error_t agentos_checkpoint_verify(const agentos_task_checkpoint_t *checkpoint,
                                          bool *is_valid);
agentos_error_t agentos_checkpoint_destroy(agentos_task_checkpoint_t *checkpoint);
```

### 2.2 生命周期

```c
agentos_error_t agentos_checkpoint_init(const char *storage_path);
agentos_error_t agentos_checkpoint_shutdown(void);
```

### 2.3 清理与统计

```c
agentos_error_t agentos_checkpoint_cleanup(uint64_t max_age_seconds, size_t max_count);
agentos_error_t agentos_checkpoint_get_stats(agentos_checkpoint_stats_t *out_stats);
```

### 2.4 自动检查点

```c
agentos_error_t agentos_checkpoint_set_auto_hook(
    agentos_checkpoint_hook_fn hook, void *user_data, uint64_t interval_ms);
agentos_error_t agentos_checkpoint_trigger_auto(const char *task_id);
```

### 2.5 快照

```c
agentos_error_t agentos_snapshot_create(const char *task_id, const char *snapshot_path);
agentos_error_t agentos_snapshot_restore(const char *snapshot_path, char **task_id);
```

### 2.6 数据结构

```c
typedef struct agentos_task_checkpoint {
    char task_id[128];                    // 任务 ID
    char session_id[128];                 // 会话 ID
    uint64_t sequence_num;                // 序列号
    uint64_t timestamp;                   // 时间戳
    char *state_json;                     // 状态 JSON
    size_t state_size;                    // 状态大小
    char **completed_nodes;               // 已完成节点
    size_t completed_count;               // 已完成数量
    char **pending_nodes;                 // 待处理节点
    size_t pending_count;                 // 待处理数量
    agentos_checkpoint_state_t state;     // 检查点状态
    uint32_t checksum;                    // 校验和
    char metadata[512];                   // 元数据
} agentos_task_checkpoint_t;

typedef struct agentos_checkpoint_stats {
    uint64_t total_checkpoints;           // 总检查点数
    uint64_t successful_checkpoints;      // 成功检查点数
    uint64_t failed_checkpoints;          // 失败检查点数
    uint64_t total_restore_ops;           // 总恢复操作数
    uint64_t last_checkpoint_time;        // 最后检查点时间
    uint64_t avg_checkpoint_size;         // 平均检查点大小
} agentos_checkpoint_stats_t;
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | 内存管理、错误码 |
| **heapstore** | 可选依赖 | 持久化存储后端 |

### 3.2 被依赖方

- **coreloopthree**: 通过 `agentos_loop_submit_persistent()` / `agentos_loop_restore_task()` 使用检查点
- **orchestrator**: 编排长任务时使用检查点恢复

### 3.3 检查点生命周期

```
agentos_checkpoint_create()
    │
    ▼
agentos_checkpoint_save()  ──▶ 存储
    │
    ▼
agentos_checkpoint_verify()  ──▶ 校验 checksum
    │
    ▼
agentos_checkpoint_restore()  ──▶ 恢复任务状态
    │
    ▼
agentos_checkpoint_delete() 或 agentos_checkpoint_cleanup()
```

---

## 4. 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `storage_path` | 由调用者指定 | 检查点存储路径 |
| `interval_ms` | 由调用者指定 | 自动检查点间隔 |
| `max_age_seconds` | 由调用者指定 | 清理时最大保留时间 |
| `max_count` | 由调用者指定 | 清理时最大保留数量 |

### 4.1 循环配置中的检查点参数

来自 `agentos_loop_config_t`:
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `checkpoint_enabled` | 0 | 是否启用检查点 |
| `checkpoint_path` | 空 | 检查点存储路径 |
| `checkpoint_interval_ms` | 30000 | 自动保存间隔 |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `agentos_checkpoint_init` | 否 | 初始化时单线程 |
| `agentos_checkpoint_create` | 是 | 创建操作线程安全 |
| `agentos_checkpoint_save` | 是 | 保存操作线程安全 |
| `agentos_checkpoint_restore` | 是 | 恢复操作线程安全 |
| `agentos_checkpoint_verify` | 是 | 只读验证 |
| `agentos_checkpoint_cleanup` | 否 | 清理操作需串行 |

---

## 6. 相关文档

- [核心循环三层架构](../coreloopthree.md)
- [备份恢复指南](../../Capital_Guides/backup-recovery.md)
- [资源管理表](../../Capital_Specifications/project_erp/resource_management_table.md)