Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# IPC 控制面 Reconciliation 设计
> **文档定位**：A-IPC（统一进程间通信体系）控制面恢复后 Reconciliation 流程的唯一权威详细设计\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §8\
> **设计依据**：综合修正方案 §4.2.5（A-IPC 设计）+ [04-ipc-data-plane-autonomy.md](04-ipc-data-plane-autonomy.md) 原则三 + v1.1 Capability Folding 决策

---

## SSoT 声明

> **单一权威源声明**：本文件是 **IPC 控制面 Reconciliation** 的唯一权威源。Reconciliation 流程（控制面重启 → 扫描数据面状态 → 差异检测 → 状态修复）、差异检测算法、状态修复策略（补发/丢弃/重建）、超时处理与 [DSL] 降级触发均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Reconciliation 流程。
>
> 技术选型声明：IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（**不使用 page flipping**）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

---

## 文档信息卡

- **目标读者**：A-IPC 模块开发者、可靠性工程师、Macro-Supervisor 开发者
- **前置知识**：理解 [04-ipc-data-plane-autonomy.md](04-ipc-data-plane-autonomy.md) 数据面自治三原则、[03-capability-model.md](03-capability-model.md) Capability 模型、[07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) IPC fastpath
- **预计阅读时间**：25 分钟
- **核心概念**：Reconciliation、差异检测、状态修复、最终一致性、超时降级
- **复杂度标识**：高级

---

## §1 设计目标：控制面恢复后数据面状态一致性

### 1.1 问题背景

[04-ipc-data-plane-autonomy.md](04-ipc-data-plane-autonomy.md) 原则三定义了 Reconciliation 的目标：控制面恢复后，数据面状态与控制面达成最终一致。但原则三仅定义了原则，未定义具体流程。本文件补充完整的 Reconciliation 设计。

当控制面（Macro-Supervisor）崩溃后，数据面基于自治三原则继续运行，期间可能产生以下状态分歧：

| 分歧类型 | 数据面状态 | 控制面记录 | 差异 |
|---------|----------|-----------|------|
| 新增 Ring | 控制面离线期间新建 Ring | 控制面无记录 | 数据面有，控制面无 |
| 销毁 Ring | 数据面 refcount 归零已释放 | 控制面仍记录存在 | 控制面有，数据面无 |
| Capability 授权 | 暂存请求待处理 | 控制面无授权记录 | 数据面有暂存 |
| Capability 撤销 | 数据面标记 stale | 控制面未撤销 | 控制面认为有效 |
| 暂存消息 | 数据面离线队列 | 控制面无记录 | 数据面有未投递 |

### 1.2 设计目标

Reconciliation 的核心设计目标：

1. **最终一致**：控制面恢复后 30 秒内达成一致
2. **不丢消息**：暂存消息在 Reconciliation 后补发
3. **不越权**：离线期间未授权的 Capability 不被追认
4. **不泄漏**：Reconciliation 后核对 Ring 列表，无僵尸 Ring

### 1.3 一致性模型

Reconciliation 采用 **最终一致性（eventual consistency）**，而非强一致：

| 维度 | 强一致 | 最终一致（Airymax 选择） |
|------|--------|----------------------|
| 一致时间 | 立即 | ≤30s |
| 可用性 | 低（恢复期间阻塞） | 高（恢复期间自治） |
| 分区容忍 | 低 | 高（容忍控制面离线） |
| 适用场景 | 金融交易 | IPC 消息传递 |

---

## §2 Reconciliation 流程：控制面重启 → 扫描数据面状态 → 差异检测 → 状态修复

### 2.1 总流程

```
控制面崩溃
    │
    ▼
数据面自治运行（原则一+二）
    │  └── 记录 reconciliation log
    │
    ▼
控制面重启（systemd 自动重启）
    │
    ▼
1. 扫描数据面状态（io_uring_cmd RECONCILE_SCAN）
    │  ├── 扫描所有存活 Ring（内核 Ring 表遍历）
    │  ├── 扫描 Capability 状态（v1.1 `agent_caps[1024]` 静态数组遍历）
    │  ├── 扫描离线队列（暂存消息）
    │  └── 读取 reconciliation log
    │
    ▼
2. 差异检测（对比控制面记录 vs 数据面实际）
    │  ├── Ring 列表差异
    │  ├── Capability 状态差异
    │  └── 暂存消息差异
    │
    ▼
3. 状态修复
    │  ├── 补发缺失消息
    │  ├── 丢弃过期消息
    │  └── 重建 Capability 缓存
    │
    ▼
4. 确认完成（标记 control_plane_verified=true）
    │
    ▼
恢复正常双面协作
```

### 2.2 控制面重启检测

控制面重启后，首先检测是否需要 Reconciliation：

```c
/* services/daemons/macro_superv/reconcile.c —— 重启检测 */
static bool macro_superv_need_reconcile(void)
{
    /* 1. 读取持久化的控制面状态（上次退出时的快照） */
    struct airy_cp_state *snapshot = load_cp_state("/var/lib/airymax/cp_state");

    /* 2. 检查数据面是否有 reconciliation log（控制面离线期间产生） */
    __u64 log_entries = airy_ipc_reconcile_log_count();
    if (log_entries > 0) {
        syslog(LOG_INFO, "airy: reconciliation needed, %llu offline events",
               (unsigned long long)log_entries);
        return true;
    }

    /* 3. 检查控制面版本号是否变化（版本升级后需全量 Reconciliation） */
    if (snapshot && snapshot->version != AIRY_CP_VERSION) {
        syslog(LOG_INFO, "airy: version mismatch, full reconciliation");
        return true;
    }

    return false;
}
```

### 2.3 扫描数据面状态

控制面通过 io_uring_cmd 向内核请求扫描数据面状态：

```c
/* services/daemons/macro_superv/reconcile.c —— 扫描数据面 */
static int reconcile_scan(struct airy_reconcile_state *state)
{
    struct airy_ipc_cmd cmd = {
        .op = AIRY_IPC_OP_RECONCILE_SCAN,
        .flags = AIRY_RECONCILE_F_FULL,
    };

    /* 通过 IORING_OP_URING_CMD 请求内核扫描 */
    int ret = airy_uring_cmd_exec(&cmd, sizeof(cmd));
    if (ret < 0)
        return ret;

    /* 内核将扫描结果写入 registered buffer */
    memcpy(state, g_registered_buf, sizeof(*state));
    return 0;
}
```

---

## §3 差异检测算法：对比控制面记录与数据面实际状态

### 3.1 差异检测模型

差异检测对比控制面持久化记录与数据面实际状态，输出三类差异：

| 差异类型 | 数据面 | 控制面 | 修复动作 |
|---------|--------|--------|---------|
| 数据面新增 | 有 | 无 | 控制面补录 |
| 数据面缺失 | 无 | 有 | 控制面删除记录 |
| 状态冲突 | 状态 A | 状态 B | 按优先级裁决 |

### 3.2 Ring 差异检测

```c
/* services/daemons/macro_superv/reconcile.c —— Ring 差异检测 */
struct ring_diff {
    __u32 ring_id;
    __u8  diff_type;   /* NEW / MISSING / CONFLICT */
    __u8  data_state;  /* 数据面状态 */
    __u8  cp_state;    /* 控制面状态 */
};

static int reconcile_detect_ring_diff(
    struct airy_ring_list *data_rings,    /* 数据面扫描结果 */
    struct airy_ring_list *cp_rings,      /* 控制面持久化记录 */
    struct ring_diff *diffs, size_t *diff_count)
{
    /* 1. 遍历数据面 Ring，找出控制面无记录的（新增） */
    for_each_ring(data_rings, ring) {
        if (!ring_list_contains(cp_rings, ring->ring_id)) {
            diffs[(*diff_count)++] = (struct ring_diff){
                .ring_id = ring->ring_id,
                .diff_type = AIRY_DIFF_NEW,
                .data_state = ring->state,
            };
        }
    }

    /* 2. 遍历控制面记录，找出数据面无对应（缺失/已释放） */
    for_each_ring(cp_rings, ring) {
        if (!ring_list_contains(data_rings, ring->ring_id)) {
            diffs[(*diff_count)++] = (struct ring_diff){
                .ring_id = ring->ring_id,
                .diff_type = AIRY_DIFF_MISSING,
                .cp_state = ring->state,
            };
        }
    }

    /* 3. 双方都有但状态不同（冲突） */
    for_each_ring(data_rings, ring) {
        struct airy_ring *cp_ring = ring_list_find(cp_rings, ring->ring_id);
        if (cp_ring && cp_ring->state != ring->state) {
            diffs[(*diff_count)++] = (struct ring_diff){
                .ring_id = ring->ring_id,
                .diff_type = AIRY_DIFF_CONFLICT,
                .data_state = ring->state,
                .cp_state = cp_ring->state,
            };
        }
    }
    return 0;
}
```

### 3.3 Capability 差异检测

Capability 差异检测对比控制面授权记录与数据面缓存：

```c
/* services/daemons/macro_superv/reconcile.c —— Capability 差异检测 */
static int reconcile_detect_cap_diff(
    struct airy_cap_cache_list *data_caps,  /* 数据面缓存扫描 */
    struct airy_cap_record_list *cp_caps,   /* 控制面授权记录 */
    struct cap_diff *diffs, size_t *diff_count)
{
    /* 数据面有缓存但 control_plane_verified=false 的 → 需重新校验 */
    for_each_cap(data_caps, cap) {
        if (!cap->control_plane_verified) {
            /* 离线期间基于缓存，需控制面确认 */
            diffs[(*diff_count)++] = (struct cap_diff){
                .cap_id = cap->cap_id,
                .diff_type = AIRY_DIFF_NEED_VERIFY,
            };
        }
    }

    /* 控制面记录已撤销但数据面仍 ACTIVE → 需同步撤销 */
    for_each_cap(cp_caps, cap) {
        if (cap->state == AIRY_CAP_REVOKED) {
            struct airy_cap_cache_entry *data_cap = cap_cache_find(cap->cap_id);
            if (data_cap && data_cap->state != AIRY_CAP_REVOKED) {
                diffs[(*diff_count)++] = (struct cap_diff){
                    .cap_id = cap->cap_id,
                    .diff_type = AIRY_DIFF_NEED_REVOKE,
                };
            }
        }
    }
    return 0;
}
```

### 3.4 暂存消息差异

控制面离线期间的暂存消息（新 Capability 请求等）需检测并处理：

| 暂存类型 | 处理 |
|---------|------|
| 新 Capability 请求 | 控制面重新评估，决定授权/拒绝 |
| 路由变更请求 | 控制面确认后应用 |
| Ring 销毁请求 | 控制面确认后执行 |

---

## §4 状态修复策略：补发缺失消息、丢弃过期消息、重建 Capability 缓存

### 4.1 修复策略矩阵

| 差异类型 | 修复策略 | 原因 |
|---------|---------|------|
| 数据面新增 Ring | 控制面补录 | Ring 已存在，控制面需知悉 |
| 数据面缺失 Ring | 控制面删除记录 | Ring 已释放，记录失效 |
| 状态冲突 | 数据面优先 | 数据面是实际状态 |
| 暂存消息（未过期） | 补发 | 消息不丢失 |
| 暂存消息（已过期） | 丢弃 | 超时失效 |
| Capability 缓存 stale | 重建缓存 | 重新校验 |

### 4.2 补发缺失消息

```c
/* services/daemons/macro_superv/reconcile.c —— 补发暂存消息 */
static int reconcile_redeliver_pending(struct airy_msg_queue *pending)
{
    for_each_msg(pending, msg) {
        /* 检查是否过期（TTL 机制） */
        if (msg->timestamp_ns + AIRY_MSG_TTL < ktime_get_real_ns()) {
            /* 过期：丢弃，记录告警 */
            syslog(LOG_WARNING, "airy: drop expired msg (age=%lluns)",
                   (unsigned long long)(ktime_get_real_ns() - msg->timestamp_ns));
            airy_msg_drop(msg);
            continue;
        }

        /* 未过期：补发到目标 Ring */
        int ret = airy_ipc_send(msg->ring_id, msg->payload, msg->payload_len);
        if (ret < 0) {
            /* 补发失败（Ring 已销毁）：丢弃 */
            syslog(LOG_WARNING, "airy: redeliver failed (ring=%u): %d",
                   msg->ring_id, ret);
        }
    }
    return 0;
}
```

### 4.3 丢弃过期消息

过期消息的判定标准：

| 消息类型 | TTL | 过期处理 |
|---------|-----|---------|
| 普通 IPC 消息 | 30 秒 | 丢弃 + 告警 |
| Capability 请求 | 60 秒 | 丢弃 + 拒绝 |
| 控制指令 | 10 秒 | 丢弃 + 告警 |

### 4.4 重建 Capability 缓存

```c
/* services/daemons/macro_superv/reconcile.c —— 重建 Capability 缓存 */
static int reconcile_rebuild_cap_cache(struct cap_diff *diffs, size_t count)
{
    for (size_t i = 0; i < count; i++) {
        struct cap_diff *d = &diffs[i];
        switch (d->diff_type) {
        case AIRY_DIFF_NEED_VERIFY:
            /* 控制面重新校验，更新缓存 */
            if (macro_superv_verify_cap(d->cap_id)) {
                airy_cap_cache_update_verified(d->cap_id, true);
            } else {
                /* 校验失败：撤销 */
                airy_cap_cache_update_state(d->cap_id, AIRY_CAP_REVOKED);
            }
            break;
        case AIRY_DIFF_NEED_REVOKE:
            /* 同步撤销：控制面已撤销，数据面需同步 */
            airy_cap_cache_update_state(d->cap_id, AIRY_CAP_REVOKED);
            break;
        default:
            break;
        }
    }
    return 0;
}
```

---

## §5 超时处理：Reconciliation 超时触发 [DSL] 降级

### 5.1 超时定义

Reconciliation 必须在 30 秒内完成。超时定义为：

| 阶段 | 超时阈值 | 超时处理 |
|------|---------|---------|
| 扫描阶段 | 5 秒 | 重试一次，仍超时则降级 |
| 差异检测 | 10 秒 | 降级 |
| 状态修复 | 15 秒 | 降级 |

### 5.2 超时降级流程

当 Reconciliation 超时，触发 [DSL] 降级（详见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)）：

```c
/* services/daemons/macro_superv/reconcile.c —— 超时降级 */
static void reconcile_timeout_handler(int sig, siginfo_t *info, void *ctx)
{
    syslog(LOG_ERR, "airy: reconciliation timeout, entering [DSL] degraded mode");

    /* 1. 进入 [DSL] 降级模式 */
    airy_dsl_enter(AIRY_DSL_REASON_RECONCILE_TIMEOUT);

    /* 2. 通知所有 Agent：控制面降级 */
    airy_ipc_broadcast_dsl_notification();

    /* 3. 数据面继续自治运行（基于现有缓存） */
    /*    新 Capability 请求直接拒绝，返回 AIRY_EDSL_CAP_MINIMAL（-206，见 08-sc-error-contract.md §2.6） */

    /* 4. 每 60 秒重试 Reconciliation */
    schedule_retry(60);
}
```

### 5.3 [DSL] 降级模式下的行为

Reconciliation 超时进入 [DSL] 降级后，IPC 行为调整：

| 行为 | 正常模式 | [DSL] 降级模式 |
|------|---------|--------------|
| 消息传递 | 基于完整 Capability | 仅基于缓存（容忍 stale） |
| 新 Ring 创建 | 控制面授权 | 拒绝（返回 `AIRY_EDSL_CAP_MINIMAL`，-206） |
| 新 Capability | 控制面授权 | 拒绝 |
| 消息格式 | 完整 IPC 消息头 | 最简 128B 消息头 |

### 5.4 恢复条件

[DSL] 降级模式下，Reconciliation 重试成功的恢复条件：

| 条件 | 阈值 | 说明 |
|------|------|------|
| Reconciliation 重试成功 | 1 次 | 差异全部修复 |
| 数据面缓存刷新 | 100% | 所有缓存标记 verified |
| Agent 确认 | 全部确认 | 所有 Agent 收到恢复通知 |

恢复后退出 [DSL] 降级，恢复正常双面协作。

---

## §6 测试与验证

### 6.1 测试场景

| 测试场景 | 验证内容 | 预期结果 |
|---------|---------|---------|
| 控制面崩溃 30 秒后恢复 | 完整 Reconciliation | 30 秒内一致，消息不丢 |
| 控制面崩溃 5 分钟后恢复 | 长时间自治 | 暂存消息部分过期丢弃 |
| Reconciliation 超时 | [DSL] 降级 | 降级后继续自治，60 秒重试 |
| Reconciliation 中 Ring 新增 | 并发处理 | 新 Ring 纳入下一次 Reconciliation |
| Reconciliation 中 Capability 撤销 | 并发处理 | 撤销操作即时生效 |

### 6.2 SLO

| 指标 | SLO | 说明 |
|------|-----|------|
| Reconciliation 完成时间 | ≤30s | 最终一致 |
| 消息补发成功率 | ≥99.9% | 未过期消息全部补发 |
| 缓存重建完整度 | 100% | 所有 stale 缓存重新校验 |
| 超时降级触发率 | ≤0.1% | 正常情况不降级 |

---

## §7 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §8 —— A-IPC 模块总纲
- [04-ipc-data-plane-autonomy.md](04-ipc-data-plane-autonomy.md) —— 数据面自治三原则（原则三定义 Reconciliation 目标）
- [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— IPC 协议契约
- [03-capability-model.md](03-capability-model.md) —— Capability 模型（缓存基础）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级（超时触发）
- [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md) —— Macro-Supervisor（控制面主体）
- 综合修正方案 §4.2.5 —— A-IPC 设计依据

---

## §8 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：IPC 控制面 Reconciliation 详细设计；完整流程（控制面重启 → 扫描数据面 → 差异检测 → 状态修复）；差异检测算法（Ring/Capability/暂存消息三类差异）；状态修复策略（补发缺失/丢弃过期/重建缓存）；超时处理触发 [DSL] 降级；最终一致性模型（≤30s） |
| v1.1 | 2026-07-18 | **Capability Folding 集成版**——① §2.3 流程图扫描动作从 "radix tree 遍历" 更新为 "Ring 表遍历 + `agent_caps[1024]` 静态数组遍历"；② §5.2 错误码 `AIRY_ECAP_OFFLINE` 替换为 `AIRY_EDSL_CAP_MINIMAL`（-206，[DSL] 兜底码）；③ §5.3 表格同步更新；④ 全文术语对齐 v1.1（Capability Folding 单平面架构） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | IPC 控制面 Reconciliation 设计 | v1.1 | 2026-07-18
