Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# IPC 数据面自治三原则
> **文档定位**：A-IPC（统一进程间通信体系）数据面自治三大设计原则的唯一权威定义\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §8\
> **设计依据**：综合修正方案 §4.2.5（A-IPC 设计）+ v1.0.1 Capability Folding 决策

---

## SSoT 声明

> **单一权威源声明**：本文件是 **IPC 数据面自治三原则** 的唯一权威源。Ring 生命周期解耦原则、离线缓存校验原则、Reconciliation 原则的定义、触发条件、与 A-IPC 模块的关系均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义数据面自治策略。
>
> 技术选型声明：IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（**不使用 page flipping**）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

---

## 文档信息卡

- **目标读者**：A-IPC 模块开发者、可靠性工程师、Agent 开发者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) §8 A-IPC 模块、[02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) IPC 协议、[07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) IPC fastpath
- **预计阅读时间**：25 分钟
- **核心概念**：数据面自治、Ring 生命周期解耦、离线缓存校验、Reconciliation、控制面/数据面分离
- **复杂度标识**：高级

---

## §1 设计目标：控制面故障时数据面继续运行

### 1.1 问题背景

传统 IPC 系统的控制面与数据面紧耦合：控制面进程（如名称服务、Capability 授权服务）宕机时，数据面（消息传递路径）随之不可用。这导致一个进程的崩溃引发连锁故障，违反"故障隔离"原则。

Airymax 将 IPC 分为控制面与数据面：

| 维度 | 控制面 | 数据面 |
|------|--------|--------|
| 职责 | Capability 授权、Ring 创建/销毁、路由表维护 | 消息发送/接收（fastpath） |
| 实现 | Macro-Supervisor 用户态守护进程 | 内核 Ring Buffer + io_uring_cmd |
| 故障影响 | 授权中断，无新 Ring 创建 | 消息传递应继续可用 |
| 故障频率 | 较高（用户态进程） | 极低（内核态） |

### 1.2 设计目标

数据面自治三原则的核心目标是：**控制面故障时，数据面继续运行**。具体为：

1. **不丢消息**：控制面崩溃期间，已建立的 IPC 通道消息不丢失
2. **不停止传递**：控制面崩溃不影响已注册 Ring 的消息收发
3. **可恢复一致**：控制面恢复后，数据面状态与控制面最终一致

### 1.3 与 seL4 微内核的关系

数据面自治借鉴 seL4 微内核的设计哲学：内核只保留最小机制（Ring Buffer、Capability 缓存），策略（路由、授权）下沉用户态。seL4 的 Capability 系统天然支持离线校验——Capability 一旦 mint 出来，其校验不依赖授权者在线。Airymax 将此特性引入 Linux 6.6 框架。

---

## §2 原则一：Ring 生命周期解耦 — IPC Ring Buffer 独立于控制面进程生命周期

### 2.1 原则定义

**Ring 生命周期解耦原则**：IPC Ring Buffer 的生命周期独立于控制面进程。控制面进程（Macro-Supervisor）崩溃/重启不影响已创建的 Ring Buffer 及其消息传递能力。

### 2.2 生命周期对比

| 维度 | 传统紧耦合设计 | Airymax 解耦设计 |
|------|-------------|-----------------|
| Ring 创建者 | 控制面进程 | 控制面进程（通过 io_uring_cmd 请求内核创建） |
| Ring 物理宿主 | 控制面进程地址空间 | 内核态（alloc_pages 分配） |
| 控制面崩溃影响 | Ring 随之销毁 | Ring 继续存活 |
| 消息传递 | 中断 | 继续 |
| 恢复后 | 重建所有 Ring | 无需重建（仍存在） |

### 2.3 实现机制

Ring Buffer 由内核 `alloc_pages(GFP_KERNEL)` 分配，物理宿主在内核态，mmap 映射到用户态。控制面进程仅持有 Ring 的 handle（opaque 引用），不持有 Ring 内存：

```c
/* kernel/ipc/airy_ipc_ring.c —— Ring 生命周期与控制面解耦 */
struct airy_ipc_ring {
    struct page *pages;          /* 内核态物理页，alloc_pages 分配 */
    void        *kaddr;          /* 内核态虚拟地址 */
    size_t       size;
    __u32        ring_id;        /* Ring 唯一 ID（内核分配） */
    __u32        owner_agent;    /* 创建者 Agent ID（仅记录，不持有引用） */
    struct rcu_head rcu;         /* RCU 延迟释放 */
    bool         frozen;         /* 冻结标志（Micro-Supervisor 设置） */
    atomic_t     refcount;       /* 引用计数（独立于控制面进程） */
};

/*
 * Ring 释放条件：refcount 归零，而非控制面进程退出。
 * 控制面崩溃时，refcount 仍 > 0（数据面仍引用），Ring 不释放。
 */
static void airy_ipc_ring_release(struct airy_ipc_ring *ring)
{
    if (atomic_dec_and_test(&ring->refcount)) {
        /* RCU 延迟释放，确保所有读者退出 */
        call_rcu(&ring->rcu, airy_ipc_ring_free);
    }
}
```

### 2.4 数据面引用独立

数据面通过 io_uring_cmd 的 registered buffer 引用 Ring，引用计数独立于控制面：

```
控制面进程                    数据面进程            内核 Ring
    │                            │                  │
    │── io_uring_cmd(CREATE) ──▶│                  │
    │                            │  refcount=1      │
    │                            │                  │
    │── io_uring_cmd(REGISTER) ──────────────▶     │ refcount=2
    │                            │                  │
    │  崩溃！                     │                  │
    │  ✗                          │  refcount 归 1   │
    │                            │── SEND/RECV ──▶ │ 继续可用
    │                            │                  │
    │  重启恢复                   │                  │
    │── RECONCILE ──────────────▶│  refcount 归 2   │
```

---

## §3 原则二：离线缓存校验 — Capability 缓存离线校验，不依赖在线服务

> **v1.1 重构说明**（Capability Folding 集成版）：原则二的**目标不变**（控制面离线时数据面仍可校验），但**实现机制升级**——
> - v1.0：`radix tree` 缓存查找 + TTL 过期 + 两层校验（fastpath 查缓存 / slowpath 查控制面）
> - v1.0.1：`agent_caps[1024]` 静态数组 + 64-bit Badge 内联校验（fastpath C-S9 ~10ns，无 radix tree 查找、无 RCU 锁）
>
> v1.0.1 Capability Folding 通过 Badge 模型天然满足离线校验要求：Badge 由 sec_d 在控制面在线时编译并写入 `agent_caps[src_task]`，控制面离线后 fastpath C-S9 仍可基于 `agent_caps[]` 内联校验（数组无锁多读者）。v1.0 的 `radix tree` 缓存、TTL 过期、`AIRY_ECAP_RADIX_MISS` 兜底码仅在 [DSL] 降级模式下作为 fallback 路径保留（H6）。v1.1 实现详情见 [03-capability-model.md §13.2](03-capability-model.md) 与 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md)。
>
> 下文 §3.1~§3.5 保留 v1.0 原始设计描述（radix tree 缓存模型），作为原则二的设计语义参考；实际实现以 v1.0.1 Capability Folding 为准。

### 3.1 原则定义

**离线缓存校验原则**：Capability 校验结果缓存在内核 radix tree，控制面离线时数据面仍可基于缓存校验，不依赖在线的 Capability 授权服务。

### 3.2 两层校验模型

A-IPC 的 Capability 校验采用分层设计，fastpath 仅查缓存，不依赖控制面：

| 层级 | 校验方式 | 控制面依赖 | 延迟 |
|------|---------|-----------|------|
| 第一层（fastpath） | 纯 C LSM 轻量标记 + radix tree 缓存查找 | 不依赖 | ~160ns |
| 第二层（slowpath） | 控制面完整校验（仅缓存未命中时） | 依赖 | ~10μs |

### 3.3 缓存设计

```c
/* kernel/ipc/airy_ipc_cap.c —— Capability 离线缓存 */
/*
 * Capability 校验结果缓存在内核 radix tree。
 * 控制面在线时定期刷新缓存（TTL 机制）。
 * 控制面离线时，缓存仍可服务 fastpath 校验。
 */
static RADIX_TREE(airy_cap_cache, GFP_ATOMIC);

struct airy_cap_cache_entry {
    __u32 cap_id;           /* Capability ID */
    __u64 badge;            /* 权限掩码 */
    __u32 owner_agent;
    __u8  state;            /* ACTIVE / REVOKED / EXPIRED */
    __u64 expires_ns;       /* TTL 过期时间（0=永不过期） */
    bool control_plane_verified;  /* 控制面是否已校验 */
};

/* fastpath 校验：仅查缓存，不依赖控制面 */
static int airy_cap_check_fastpath(__u32 cap_id, __u64 required_badge)
{
    struct airy_cap_cache_entry *entry;
    rcu_read_lock();
    entry = radix_tree_lookup(&airy_cap_cache, cap_id);
    if (!entry) {
        rcu_read_unlock();
        return -AIRY_ECAP_RADIX_MISS;  /* 缓存未命中，走 slowpath（[DSL] 兜底码，见 08-sc-error-contract.md §2.4） */
    }
    /* 缓存命中：检查权限掩码与状态 */
    if (entry->state != AIRY_CAP_ACTIVE) {
        rcu_read_unlock();
        return -AIRY_ECAP_REVOKED;
    }
    if ((entry->badge & required_badge) != required_badge) {
        rcu_read_unlock();
        return -AIRY_ECAP_PERM;
    }
    rcu_read_unlock();
    return 0;  /* 校验通过 */
}
```

### 3.4 缓存失效策略

| 策略 | 触发条件 | 行为 |
|------|---------|------|
| TTL 过期 | `expires_ns` 到达 | 标记 stale，fastpath 降级到 slowpath |
| 主动撤销 | 控制面 REVOKE 操作 | 立即标记 REVOKED，fastpath 拒绝 |
| 控制面离线 | 控制面心跳超时 | 缓存继续服务，但标记 `control_plane_verified=false` |
| 控制面恢复 | Reconciliation | 刷新缓存，重新标记 verified |

### 3.5 离线校验的容忍度

控制面离线期间，缓存继续服务，但存在以下容忍度约束：

| 场景 | 容忍度 | 处理 |
|------|--------|------|
| 缓存命中且未过期 | 完全容忍 | fastpath 直接通过 |
| 缓存命中但 stale | 部分容忍 | fastpath 通过但记录告警日志 |
| 缓存未命中 | 不容忍 | fastpath 返回 `AIRY_EDSL_CAP_MINIMAL`（-206），消息暂存离线队列（见 [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) §2.6） |

---

## §4 原则三：Reconciliation — 控制面恢复后，数据面状态同步

### 4.1 原则定义

**Reconciliation 原则**：控制面恢复后，数据面将离线期间的状态变更同步给控制面，达成最终一致（eventual consistency）。

### 4.2 Reconciliation 流程

```
控制面崩溃
    │
    ▼
数据面自治运行（Ring 解耦 + 缓存校验）
    │  ├── 消息继续传递（基于缓存）
    │  ├── 新 Capability 请求暂存离线队列
    │  └── 状态变更记录到 reconciliation log
    │
    ▼
控制面重启恢复
    │
    ▼
触发 Reconciliation
    │  ├── 扫描数据面实际状态（Ring 列表、缓存内容、离线队列）
    │  ├── 与控制面记录对比
    │  ├── 差异检测（新增/删除/修改的 Ring、Capability）
    │  └── 状态修复（补发缺失、丢弃过期、重建缓存）
    │
    ▼
最终一致，恢复正常双面协作
```

### 4.3 Reconciliation log

数据面在控制面离线期间记录状态变更到 reconciliation log：

```c
/* kernel/ipc/airy_ipc_reconcile.c —— Reconciliation log */
enum airy_reconcile_op {
    AIRY_RECON_NEW_RING,        /* 新建 Ring（控制面离线时） */
    AIRY_RECON_DESTROY_RING,    /* 销毁 Ring */
    AIRY_RECON_CAP_GRANT,       /* Capability 授权（暂存） */
    AIRY_RECON_CAP_REVOKE,      /* Capability 撤销 */
    AIRY_RECON_MSG_PENDING,     /* 暂存消息 */
};

struct airy_reconcile_entry {
    enum airy_reconcile_op op;
    __u32 ring_id;
    __u32 agent_id;
    __u64 timestamp_ns;
    char  data[64];             /* 操作相关数据 */
};
```

### 4.4 最终一致性保证

Reconciliation 不要求强一致，而是**最终一致**：

| 维度 | 保证 | 说明 |
|------|------|------|
| 消息不丢失 | 是 | 暂存消息在 Reconciliation 后补发 |
| Capability 不越权 | 是 | 离线期间仅基于缓存，无新增授权 |
| Ring 不泄漏 | 是 | Reconciliation 后核对 Ring 列表 |
| 一致时间 | ≤30s | 控制面恢复后 30 秒内达成一致 |

---

## §5 与 A-IPC 模块的关系：数据面自治是 A-IPC 的核心韧性设计

### 5.1 在 A-IPC 中的定位

数据面自治三原则是 A-IPC 模块的核心韧性设计，对应 [10-unify-design.md](../10-architecture/10-unify-design.md) §3 的"韧性"技术领域：

```
A-IPC 模块组成
├── IPC 协议（[02-ipc-protocol.md]）
├── IPC fastpath（[07-ipc-fastpath.md]）
├── io_uring_cmd 语义映射
├── Capability 分层校验
└── 数据面自治三原则（本文件）  ← 韧性核心
    ├── 原则一：Ring 生命周期解耦
    ├── 原则二：离线缓存校验
    └── 原则三：Reconciliation
```

### 5.2 三原则的协作

三原则并非独立，而是层层递进的韧性保障：

| 原则 | 解决的问题 | 不满足时的退化 |
|------|----------|-------------|
| Ring 生命周期解耦 | 控制面崩溃导致 Ring 消失 | 消息传递中断 |
| 离线缓存校验 | 控制面崩溃导致校验不可用 | 所有 Capability 校验失败 |
| Reconciliation | 控制面恢复后状态不一致 | 永久状态分裂 |

三原则缺一不可：仅有解耦无缓存校验，消息无法传递；有缓存校验无 Reconciliation，恢复后状态分裂。

### 5.3 与 A-ULS 模块的协作

数据面自治与 A-ULS 模块的 Macro-Supervisor 协作：

| A-ULS 组件 | 与数据面自治的关系 |
|---------|-----------------|
| Micro-Supervisor | 检测异常时冻结 Ring（`ring->frozen=true`），触发降级 |
| Macro-Supervisor | 控制面主体，崩溃后由数据面自治保证消息不丢 |
| Agent 8 态 | 控制面崩溃对应 Macro-Supervisor 进入 BLOCKED 态 |

### 5.4 与 [DSL] 降级层的关系

当 Reconciliation 超时（控制面恢复后 30 秒内未达成一致），触发 [DSL] 降级（详见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)）：

| [DSL] 降级项 | Reconciliation 超时时的表现 |
|-------------|---------------------------|
| IPC | 仅使用最简 128B 消息头，不依赖完整 Capability |
| 错误码 | 返回 `AIRY_EDSL_CAP_MINIMAL`（-206，[DSL] 码空间，见 [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) §2.6） |
| 调度 | EEVDF 默认（不依赖 SCHED_DEADLINE） |

---

## §6 测试与验证

### 6.1 测试场景

| 测试场景 | 验证原则 | 预期结果 |
|---------|---------|---------|
| kill -9 Macro-Supervisor 后发消息 | 原则一+二 | 消息正常传递，基于缓存 |
| 控制面离线 60 秒后恢复 | 原则三 | Reconciliation 在 30 秒内完成 |
| Reconciliation 超时 | [DSL] 降级 | 进入降级模式，告警 |
| Ring 创建后控制面崩溃 | 原则一 | Ring 继续存活，消息不丢 |

### 6.2 SLO

| 指标 | SLO | 说明 |
|------|-----|------|
| 控制面崩溃期间消息传递成功率 | ≥99.9% | 基于缓存 |
| Reconciliation 完成时间 | ≤30s | 最终一致 |
| 缓存命中率（正常） | ≥99% | fastpath |
| 缓存命中率（离线） | ≥95% | 容忍 stale |

---

## §7 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §8 —— A-IPC 模块总纲
- [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— IPC 协议契约
- [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— IPC fastpath 设计
- [05-ipc-control-plane-reconciliation.md](05-ipc-control-plane-reconciliation.md) —— Reconciliation 详细设计
- [03-capability-model.md](03-capability-model.md) —— Capability 模型（缓存基础）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级（Reconciliation 超时）
- 综合修正方案 §4.2.5 —— A-IPC 设计依据

---

## §8 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：IPC 数据面自治三原则；原则一 Ring 生命周期解耦（内核 alloc_pages + 引用计数独立）；原则二离线缓存校验（radix tree + TTL + 两层校验）；原则三 Reconciliation（最终一致 + reconciliation log）；与 A-IPC/A-ULS/[DSL] 模块协作关系 |
| v1.1 | 2026-07-18 | **Capability Folding 集成版**——① §3 原则二添加 v1.1 重构说明：实现机制从 v1.0 `radix tree` 缓存升级为 v1.1 `agent_caps[1024]` 静态数组 + 64-bit Badge 内联校验（fastpath C-S9 ~10ns），原则目标不变；② §3.5 错误码 `AIRY_ECAP_OFFLINE` 替换为 `AIRY_EDSL_CAP_MINIMAL`（-206，[DSL] 兜底码）；③ §3.3 `AIRY_ECAP_NOT_FOUND` 替换为 `AIRY_ECAP_RADIX_MISS`（-76，[DSL] 兜底码）；④ 全文术语对齐 v1.1（Capability Folding 单平面架构） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | IPC 数据面自治三原则 | v1.0.1 | 2026-07-21
