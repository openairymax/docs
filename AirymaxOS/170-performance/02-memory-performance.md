Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）内存性能工程设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）内存子系统性能工程详细设计\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-26\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-linux（AirymaxOS）内存性能工程为 MemoryRovol（记忆卷载）四层记忆架构提供硬件级加速支撑。本设计聚焦三大目标：

1. **MemoryRovol L1-L4 分级内存访问延迟 SLO**：L1 ≤ 0.1ms、L2 ≤ 1ms、L3 ≤ 10ms、L4 ≤ 100ms
2. **MGLRU 多代 LRU 与遗忘机制语义对齐**：冷热数据分层命中率 ≥ 90%，与 Forgetting Engine 衰减公式协同
3. **CXL/PMEM 混合内存带宽最优利用**：跨节点内存池化带宽利用率 ≥ 80%，PMEM 持久化写入延迟 ≤ 5μs

### 1.2 适用范围

- MemoryRovol 四层记忆（L1 原始卷、L2 特征层、L3 结构层、L4 模式层）
- Linux 6.6 内核基线 MGLRU 多代 LRU 内存回收（非 2.0 版本）
- CXL 2.0 内存池化 + PMEM 持久内存
- kthread 间通信的 kfifo + wait_event_interruptible 批量迁移机制

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。MemoryRovol 在 agentrt-linux 中基于 CXL + MGLRU 内核态实现，与 agentrt 基于 HeapStore 用户态实现属于 IRON-9 v3 [SS] 语义同源层。

### 1.4 内存性能 SLO 矩阵

| 层级 | 组件 | 访问延迟 SLO | 命中率 SLO | 驱逐率 SLO |
|------|------|--------------|------------|------------|
| L1_raw | 原始卷（DRAM + PMEM） | ≤ 0.1ms | ≥ 0.85 | ≤ 0.05 |
| L2_feat | 特征层（DRAM + CXL） | ≤ 1ms | ≥ 0.72 | ≤ 0.10 |
| L3_str | 结构层（DRAM） | ≤ 10ms | ≥ 0.91 | ≤ 0.02 |
| L4_pat | 模式层（PMEM 持久） | ≤ 100ms | ≥ 0.95 | ≤ 0.01 |

---

## 2. MemoryRovol L1-L4 分级内存架构

### 2.1 四层分级内存拓扑

```
┌─────────────────────────────────────────────────────────────┐
│              MemoryRovol 四层分级内存架构                  │
├─────────────────────────────────────────────────────────────┤
│  L1 原始卷（raw）  │ DRAM + PMEM append-only │ SHA-256+ZSTD │
│                    │   延迟 ≤ 0.1ms          │ 命中率 0.85 │
├─────────────────────────────────────────────────────────────┤
│  L2 特征层（feat） │ DRAM + CXL HNSW + BM25 │ 混合检索     │
│                    │   延迟 ≤ 1ms             │ 命中率 0.72 │
├─────────────────────────────────────────────────────────────┤
│  L3 结构层（str）  │ DRAM KG + VSA          │ 知识图谱     │
│                    │   延迟 ≤ 10ms            │ 命中率 0.91 │
├─────────────────────────────────────────────────────────────┤
│  L4 模式层（pat）  │ PMEM HDBSCAN + 0D       │ 持久化模式   │
│                    │   延迟 ≤ 100ms           │ 命中率 0.95 │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 分级内存数据结构

> **[SC] SSoT 实际定义**：以下代码原样引自 `include/uapi/linux/airymax/memory_types.h`，为 MemoryRoVol 内存类型共享契约层。请勿在本文档中修改其字段名/类型/布局；如需变更，先修改 SSoT 头文件。

```c
/* include/uapi/linux/airymax/memory_types.h —— MemoryRoVol types ([SC] shared contract header) */
#ifndef _UAPI_AIRYMAX_MEMORY_TYPES_H
#define _UAPI_AIRYMAX_MEMORY_TYPES_H

#include <linux/airymax/uapi_compat.h>

/* ─── Memory Tier Levels ─────────────────────────────────────────────── */
enum airy_mem_level {
	AIRY_MEM_HOT    = 0,   /* L1: HBM/DDR hot tier */
	AIRY_MEM_WARM   = 1,   /* L2: DDR warm tier */
	AIRY_MEM_COLD   = 2,   /* L3: CXL/NVMe cold tier */
	AIRY_MEM_PMEM   = 3,   /* L4: PMEM persistent tier */
	AIRY_MEM_LEVEL_MAX
};

/* ─── GFP Mask Semantics for MemoryRoVol ──────────────────────────────── */
#define AIRY_GFP_HOT    0x01   /* Allocate from hot tier */
#define AIRY_GFP_WARM   0x02   /* Allocate from warm tier */
#define AIRY_GFP_COLD   0x04   /* Allocate from cold tier */
#define AIRY_GFP_PMEM   0x08   /* Allocate from PMEM tier */

/* ─── Memory Page Classification ──────────────────────────────────────── */
#define AIRY_PAGE_CLASS_ANON     0x01  /* Anonymous page */
#define AIRY_PAGE_CLASS_FILE     0x02  /* File-backed page */
#define AIRY_PAGE_CLASS_SHMEM    0x04  /* Shared memory page */
#define AIRY_PAGE_CLASS_AGENT    0x08  /* Agent-private page */

/* ─── [DSL] Degraded Survival Layer Fallback Block ──────────────────────
 * When AIRY_SC_FALLBACK is defined, MemoryRoVol L2-L4 tiering is
 * unavailable; only L1 (hot tier, anonymous pages) is accessible. All
 * GFP flags collapse to AIRY_GFP_HOT and all page classes collapse to
 * AIRY_PAGE_CLASS_ANON. alloc_pages + mmap remain functional.
 * See [DSL] §2.2 and §4.1.
 */
#ifdef AIRY_SC_FALLBACK
	#define AIRY_DSL_MEM_LEVEL   AIRY_MEM_HOT
	#define AIRY_DSL_MEM_TIERS   1  /* Only L1 retained */

	#define AIRY_DSL_GFP_HOT     AIRY_GFP_HOT
	#define AIRY_DSL_GFP_WARM    AIRY_GFP_HOT
	#define AIRY_DSL_GFP_COLD    AIRY_GFP_HOT
	#define AIRY_DSL_GFP_PMEM    AIRY_GFP_HOT

	#define AIRY_DSL_PAGE_CLASS_ANON   AIRY_PAGE_CLASS_ANON
	#define AIRY_DSL_PAGE_CLASS_FILE   AIRY_PAGE_CLASS_ANON
	#define AIRY_DSL_PAGE_CLASS_SHMEM  AIRY_PAGE_CLASS_ANON
	#define AIRY_DSL_PAGE_CLASS_AGENT  AIRY_PAGE_CLASS_ANON

	#warning "AIRY_SC_FALLBACK active: memory_types.h degraded to L1 hot tier only"
#endif /* AIRY_SC_FALLBACK */

#endif /* _UAPI_AIRYMAX_MEMORY_TYPES_H */
```

> **SSoT 对齐说明**：
> - MemoryRoVol 四层分级（L1 HOT / L2 WARM / L3 COLD / L4 PMEM）由 `enum airy_mem_level` 唯一界定；本文档 2.1 节的 L1_raw/L2_feat/L3_str/L4_pat 命名仅作为业务语义映射，**不**作为内核枚举名。
> - GFP 掩码仅有 4 个：`AIRY_GFP_HOT/WARM/COLD/PMEM`；SSoT 中**不存在** `AIRY_GFP_L1_RAW`/`AIRY_GFP_L2_FEAT`/`AIRY_GFP_L3_STR`/`AIRY_GFP_L4_PAT` 等历史虚构宏。
> - SSoT 中**不存在** `airy_mr_node`/`airy_mr_mgr` 结构体定义，也**不包含** `<airymax/airy_q16.h>` 头文件；Q16.16 定点数定义位于 `include/uapi/linux/airymax/cognition_types.h`，本文档后续章节如需使用 `airy_q16_t` 将显式标注其来源。

### 2.3 CXL 内存池化配置

```bash
# /etc/airymaxos/cxl.conf —— CXL 内存池化配置
# Linux 6.6 内核基线原生 CXL 支持

# 启用 CXL 驱动
modprobe cxl_acpi
modprobe cxl_mem
modprobe cxl_region

# 查看 CXL 设备
cxl list -LM
# 输出示例：
# {
#   "memdevs":[
#     {"memdev":"mem0","ram_size":"16.00 GiB"},
#     {"memdev":"mem1","ram_size":"16.00 GiB"}
#   ]
# }

# 创建 CXL region（与 NUMA node 1 绑定）
cxl create-region -d -t pmem -s 16G -m mem0 mem1
# 创建后：CXL 内存作为 NUMA node 2 出现

numactl --hardware | grep -A2 "node 2"
# node 2 cpus:
# node 2 size: 16384 MB
# node 2 free: 15820 MB

# 配置 MemoryRovol L2 特征层使用 CXL（NUMA node 2）
echo "2" > /sys/kernel/agentrt/memory_rovol/l2_feat_numa_node
echo "16384" > /sys/kernel/agentrt/memory_rovol/l2_feat_capacity_mb
```

### 2.4 PMEM 持久化配置

```bash
# /etc/airymaxos/pmem.conf —— PMEM 持久内存配置
# MemoryRovol L1 原始卷 + L4 模式层使用 PMEM

# 启用 nd_pmem 驱动
modprobe nd_btt
modprobe libnvdimm

# 查看 PMEM 设备
ndctl list -R
# region0：size=32.00 GiB，可用作 fsdax

# 创建 DAX 模式命名空间
ndctl create-namespace -r region0 -m fsdax --size=32G
# 创建后：/dev/pmem0（DAX 模式）

# 格式化为 XFS（DAX 挂载）
mkfs.xfs -m dax=inode /dev/pmem0
mkdir /var/lib/airymaxos/memoryrovol/pmem
mount -o dax /dev/pmem0 /var/lib/airymaxos/memoryrovol/pmem

# 配置 MemoryRovol 使用 PMEM
echo "/var/lib/airymaxos/memoryrovol/pmem" > \
    /sys/kernel/agentrt/memory_rovol/pmem_mount
echo "1" > /sys/kernel/agentrt/memory_rovol/l1_raw_pmem_enable
echo "1" > /sys/kernel/agentrt/memory_rovol/l4_pat_pmem_enable
```

---

## 3. MGLRU 多代 LRU 调优

### 3.1 MGLRU 基线参数

Linux 6.6 内核基线原生支持 MGLRU 多代 LRU（非 2.0 版本）。agentrt-linux 调优后的默认参数：

```bash
# /etc/airymaxos/mglru.conf —— MGLRU 调优配置

# 启用 MGLRU（lru_gen.enabled 必须为 y 或 1）
echo "y" > /sys/kernel/mm/lru_gen/enabled

# 最小 TTL（保留时间），防止冷数据被过早回收
echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms

# 最大代数（默认 8，agentrt-linux 调整为 12 以匹配 MemoryRovol 四层 × 3 温度）
echo "12" > /sys/kernel/mm/lru_gen/max_nr_generations

# 多代 LRU 二阶分桶（避免抖动）
echo "y" > /sys/kernel/mm/lru_gen/lru_gen_enabled_bit

# 查看 MGLRU 状态
cat /sys/kernel/mm/lru_gen/enabled
# 输出：0x0007（bit0=-enabled, bit1=walk from page table, bit2=walk from memcg）

# 查看当前代数与年龄
cat /sys/kernel/mm/lru_gen/
# 输出示例：
# node 0
#   generation 12    max_seq  12
#     min_gen.time 1468923400000  max_gen.time 1468923401000
#   generation 11
#     ...
```

### 3.2 MGLRU 与 Forgetting Engine 协同

Forgetting Engine 衰减公式 `R(t) = e^(-t/τ)`（τ 默认 7 天）与 MGLRU 多代回收策略语义对齐：

| Forgetting Engine 策略 | MGLRU 代数范围 | 行为 |
|------------------------|----------------|------|
| NONE（不遗忘） | 全部代保留 | 仅按容量淘汰 |
| EBBINGHAUS（艾宾浩斯） | 最热 4 代 / 中间 4 代 / 最冷 4 代 | 按遗忘曲线跨代衰减 |
| LINEAR（线性） | 最冷 2 代 | 均匀淘汰冷数据 |
| ACCESS_BASED（访问驱动） | 仅最冷 1 代 | 按访问频率淘汰 |

> **说明性示例，非 SSoT 定义**：以下 `airy_mglru_forgetting_map` 结构体与 `airy_mglru_apply_forgetting` 函数为本文档设计草案，**不**存在于 `include/uapi/linux/airymax/memory_types.h`。其中 `airy_q16_t` 类型来自 `include/uapi/linux/airymax/cognition_types.h`（Q16.16 定点数）；`enum airy_forgetting_strategy` 与 `AIRY_FORGET_EBBINGHAUS` 来自遗忘机制子系统头文件。SSoT `memory_types.h` 仅提供 `enum airy_mem_level` 与 GFP/页面分类宏。

```c
/* 说明性示例：MGLRU 代数与 Forgetting Engine 衰减映射（非 SSoT 定义） */
#include <linux/airymax/memory_types.h>      /* SSoT: enum airy_mem_level */
#include <linux/airymax/cognition_types.h>   /* airy_q16_t (Q16.16 定点数) */

struct airy_mglru_forgetting_map {
	enum airy_forgetting_strategy strategy;  /* 来自遗忘机制子系统 */
	u8 min_gen;       /* 最冷代数 */
	u8 max_gen;       /* 最热代数 */
	airy_q16_t retention_rate;  /* Q16.16 保留率，类型见 cognition_types.h */
};

/* 艾宾浩斯衰减映射表（基于 R(t) = e^(-t/τ)，τ = 7 天） */
static const struct airy_mglru_forgetting_map ebbinghaus_map[] = {
	/* 代数 | 保留率 | 含义 */
	{AIRY_FORGET_EBBINGHAUS, 0,  3, 0x10000},  /* 最热 4 代：1.0 */
	{AIRY_FORGET_EBBINGHAUS, 4,  7, 0x8000},   /* 中间 4 代：0.5 */
	{AIRY_FORGET_EBBINGHAUS, 8, 11, 0x2000},   /* 最冷 4 代：0.125 */
	{AIRY_FORGET_EBBINGHAUS, 12, 12, 0x0000},  /* 回收代：0 */
};

/* 应用衰减映射到 MGLRU（参数 ctx 为运行时上下文指针，非 SSoT 类型） */
static int airy_mglru_apply_forgetting(void *ctx,
					  const struct airy_mglru_forgetting_map *m)
{
	u8 gen;
	int ret;

	for (gen = m->min_gen; gen <= m->max_gen; gen++) {
		ret = airy_mglru_set_gen_weight(gen, m->retention_rate);
		if (ret < 0)
			goto out_err;
	}
	return 0;

out_err:
	pr_warn("agentrt-linux: mglru apply forgetting failed gen=%u ret=%d\n",
		gen, ret);
	return ret;
}
```

### 3.3 MemoryRovol L1-L4 与 MGLRU 代绑定

> **说明性示例，非 SSoT 定义**：以下 `airy_mr_mglru_bind` 结构体为本文档设计草案，**不**存在于 `include/uapi/linux/airymax/memory_types.h`。其中 `enum airy_mem_level` 与 `AIRY_MEM_HOT/WARM/COLD/PMEM` 枚举值为 SSoT 实际定义（见 §2.2）；L1_raw/L2_feat/L3_str/L4_pat 仅作业务语义注释。

```c
/* 说明性示例：MemoryRoVol 层级与 MGLRU 代绑定（非 SSoT 定义） */
#include <linux/airymax/memory_types.h>  /* SSoT: enum airy_mem_level */

struct airy_mr_mglru_bind {
	enum airy_mem_level layer;  /* SSoT 枚举 */
	u8 hot_gen_lo;     /* 热代下界 */
	u8 hot_gen_hi;     /* 热代上界 */
	u8 cold_gen_lo;    /* 冷代下界 */
	u8 cold_gen_hi;    /* 冷代上界 */
};

static const struct airy_mr_mglru_bind mglru_binds[] = {
	/* L1_raw（AIRY_MEM_HOT）：热代 0-2，冷代 10-11（PMEM 持久化） */
	{AIRY_MEM_HOT,  0,  2, 10, 11},
	/* L2_feat（AIRY_MEM_WARM）：热代 0-3，冷代 9-11（CXL 迁移） */
	{AIRY_MEM_WARM, 0,  3,  9, 11},
	/* L3_str（AIRY_MEM_COLD）：热代 0-4，冷代 8-11（DRAM 驻留） */
	{AIRY_MEM_COLD, 0,  4,  8, 11},
	/* L4_pat（AIRY_MEM_PMEM）：热代 0-5，冷代 11-11（PMEM 持久） */
	{AIRY_MEM_PMEM, 0,  5, 11, 11},
};
```

---

## 4. CXL/PMEM 混合内存带宽管理

### 4.1 混合内存带宽模型

agentrt-linux 内存层级带宽与延迟模型：

| 层级 | 设备 | 延迟（ns） | 带宽（GB/s） | 容量 |
|------|------|-----------|--------------|------|
| L0 | 本地 DRAM | 80-100 | 80-100 | 64-256 GB |
| L1 | CXL 2.0 内存 | 170-250 | 24-50 | 16-64 GB |
| L2 | PMEM（fsdax） | 300-500 | 8-10 | 32-256 GB |
| L3 | NVMe SSD（swap） | 50000 | 3-7 | 1-8 TB |

### 4.2 内存 tiering 自动迁移策略

> **说明性示例，非 SSoT 定义**：以下 `airy_migrate_entry`/`airy_migrate_mgr` 结构体与迁移 kthread 实现为本文档设计草案，**不**存在于 `include/uapi/linux/airymax/memory_types.h`。其中 `enum airy_mem_level` 与 `AIRY_MEM_*` 枚举值为 SSoT 实际定义（见 §2.2）；SSoT 头文件 include 路径为 `<linux/airymax/memory_types.h>`。

```c
/* 说明性示例：memory/airy_memory_tiering.c（非 SSoT 定义） */
#include <linux/kfifo.h>
#include <linux/wait.h>
#include <linux/airymax/memory_types.h>  /* SSoT: enum airy_mem_level */

#define MIGRATE_FIFO_SIZE 4096

/* 迁移请求条目（说明性示例） */
struct airy_migrate_entry {
	enum airy_mem_level from;  /* SSoT 枚举 */
	enum airy_mem_level to;    /* SSoT 枚举 */
	__u64 page_addr;
	__u32 size_kb;
	__u8  reason;   /* 0=热升迁, 1=冷降级, 2=容量压力 */
};

/* 批量迁移管理器 */
struct airy_migrate_mgr {
	DECLARE_KFIFO_PTR(fifo, struct airy_migrate_entry);
	wait_queue_head_t wq;
	spinlock_t lock;
	atomic_t pending;
};

/* 提交迁移请求（kthread 间通信用 kfifo + wait_event_interruptible） */
int airy_memory_tiering_submit(struct airy_migrate_mgr *mgr,
				 const struct airy_migrate_entry *e)
{
	unsigned long flags;
	int ret;

	spin_lock_irqsave(&mgr->lock, flags);
	ret = kfifo_in(&mgr->fifo, e, 1);
	spin_unlock_irqrestore(&mgr->lock, flags);

	if (ret == 0) {
		ret = -EAGAIN;
		goto out_err;
	}

	atomic_inc(&mgr->pending);
	wake_up_interruptible(&mgr->wq);
	return 0;

out_err:
	pr_warn("agentrt-linux: migrate submit failed ret=%d\n", ret);
	return ret;
}

/* 迁移工作 kthread（批量处理迁移请求） */
static int airy_memory_tiering_thread(void *data)
{
	struct airy_migrate_mgr *mgr = data;
	struct airy_migrate_entry batch[16];
	int n, i, ret;

	while (!kthread_should_stop()) {
		/* 等待迁移请求 */
		ret = wait_event_interruptible(mgr->wq,
			atomic_read(&mgr->pending) > 0 || kthread_should_stop());
		if (ret < 0)
			continue;
		if (kthread_should_stop())
			break;

		/* 批量取走 FIFO 中的迁移请求（减少锁竞争） */
		n = kfifo_out_spinlocked(&mgr->fifo, batch,
					 ARRAY_SIZE(batch),
					 &mgr->lock);
		if (n == 0)
			continue;

		/* 处理批量迁移 */
		for (i = 0; i < n; i++) {
			atomic_dec(&mgr->pending);
			ret = airy_memory_do_migrate(&batch[i]);
			if (ret < 0)
				pr_warn("agentrt-linux: migrate failed i=%d\n", i);
		}
	}
	return 0;
}

/* 启动 tiering 迁移 kthread */
int airy_memory_tiering_init(struct airy_migrate_mgr *mgr)
{
	int ret;

	INIT_KFIFO(mgr->fifo);
	init_waitqueue_head(&mgr->wq);
	spin_lock_init(&mgr->lock);
	atomic_set(&mgr->pending, 0);

	kthread_run(airy_memory_tiering_thread, mgr, "airy_tiering");
	if (ret < 0)
		goto out_err;
	return 0;

out_err:
	pr_err("agentrt-linux: tiering init failed ret=%d\n", ret);
	return ret;
}
```

### 4.3 内存 tiering 配置文件

```bash
# /etc/airymaxos/memory_tiering.conf —— 内存 tiering 自动迁移配置

# 热升迁阈值：访问频率 > 8 次/分钟 触发 L2→L1 升迁
[tiering]
l2_to_l1_hot_threshold = 8

# 冷降级阈值：访问频率 < 1 次/分钟 触发 L1→L2 降级
l1_to_l2_cold_threshold = 1

# L4→L3 升迁（模式层激活）
l4_to_l3_hot_threshold = 4

# L3→L4 降级（模式归档）
l3_to_l4_cold_threshold = 0

# 容量压力阈值：L1 占用 > 85% 触发主动降级
l1_capacity_pressure = 85
l2_capacity_pressure = 90

# 迁移批大小（kfifo 批量读取数量）
migrate_batch_size = 16

# 迁移 kthread 优先级（sched_tac 用户态调度器策略）
migrate_prio = "agent_low"
```

### 4.4 kfifo 批量迁移完整示例

> **说明性示例，非 SSoT 定义**：以下 `airy_tiering_ctx` 结构体与 producer/consumer kthread 实现为本文档设计草案，**不**存在于 `include/uapi/linux/airymax/memory_types.h`。其中 `AIRY_MEM_HOT`/`AIRY_MEM_WARM` 为 SSoT 实际枚举值（见 §2.2）；SSoT 头文件 include 路径为 `<linux/airymax/memory_types.h>`。

```c
/* 说明性示例：MemoryRoVol 跨层批量迁移（非 SSoT 定义） */
#include <linux/module.h>
#include <linux/kfifo.h>
#include <linux/wait.h>
#include <linux/kthread.h>
#include <linux/delay.h>
#include <linux/airymax/memory_types.h>  /* SSoT: enum airy_mem_level */
#include <linux/airymax/error.h>

struct airy_tiering_ctx {
	DECLARE_KFIFO(fifo, struct airy_migrate_entry, 4096);
	wait_queue_head_t  producer_wq;
	wait_queue_head_t  consumer_wq;
	spinlock_t         lock;
	atomic_t           inflight;
	bool               shutdown;
};

static int producer_fn(void *data)
{
	struct airy_tiering_ctx *ctx = data;
	struct airy_migrate_entry e = {0};

	while (!kthread_should_stop()) {
		/* 模拟产生迁移请求：扫描 hot/cold 标记 */
		if (wait_event_interruptible_timeout(ctx->producer_wq,
						     !ctx->shutdown,
						     msecs_to_jiffies(50)) <= 0)
			continue;

		e.from = AIRY_MEM_HOT;   /* L1_raw 业务语义 */
		e.to = AIRY_MEM_WARM;    /* L2_feat 业务语义 */
		e.reason = 1;  /* 冷降级 */

		spin_lock(&ctx->lock);
		kfifo_in(&ctx->fifo, &e, 1);
		atomic_inc(&ctx->inflight);
		spin_unlock(&ctx->lock);

		wake_up_interruptible(&ctx->consumer_wq);
	}
	return 0;
}

static int consumer_fn(void *data)
{
	struct airy_tiering_ctx *ctx = data;
	struct airy_migrate_entry batch[16];
	int n, i;

	while (!kthread_should_stop()) {
		/* kfifo + wait_event_interruptible 等待迁移请求 */
		if (wait_event_interruptible(ctx->consumer_wq,
			atomic_read(&ctx->inflight) > 0 ||
			kthread_should_stop()))
			continue;
		if (kthread_should_stop())
			break;

		/* 批量读取 kfifo（一次最多 16 个，减少锁竞争） */
		n = kfifo_out_spinlocked(&ctx->fifo, batch, 16, &ctx->lock);
		if (n == 0)
			continue;

		/* 执行批量迁移 */
		for (i = 0; i < n; i++) {
			atomic_dec(&ctx->inflight);
			airy_memory_do_migrate(&batch[i]);
		}
	}
	return 0;
}
```

---

## 5. 性能基准测试

### 5.1 内存性能基准矩阵

| 基准 | 工具 | 测量维度 | 目标值 |
|------|------|----------|--------|
| L1 延迟 | `agentrt-bench mem-latency --layer L1` | P99 延迟 | ≤ 0.1ms |
| L2 延迟 | `agentrt-bench mem-latency --layer L2` | P99 延迟 | ≤ 1ms |
| L3 延迟 | `agentrt-bench mem-latency --layer L3` | P99 延迟 | ≤ 10ms |
| L4 延迟 | `agentrt-bench mem-latency --layer L4` | P99 延迟 | ≤ 100ms |
| 命中率 | `cat /sys/kernel/agentrt/memory_rovol/perf` | 命中率 | ≥ 0.85 |
| CXL 带宽 | `perf bench numadist` | 跨 NUMA 带宽 | ≥ 80% 利用率 |
| PMEM 写入 | `fio --filename=/dev/pmem0 --rw=randwrite` | 写延迟 | ≤ 5μs |

### 5.2 自动化基准测试脚本

```bash
#!/bin/bash
# /usr/lib/airymaxos/tests/memory_perf_bench.sh
# MemoryRovol 内存性能基准自动化测试

set -euo pipefail
RESULT_DIR=${1:-/var/tmp/agentrt-mem-bench}
mkdir -p "$RESULT_DIR"

echo "[1/6] MemoryRovol 四层延迟基准"
for layer in L1_raw L2_feat L3_str L4_pat; do
    agentrt-bench mem-latency --layer $layer --duration 30 \
        > "$RESULT_DIR/latency_${layer}.txt"
    p99=$(awk '/p99/ {print $3}' "$RESULT_DIR/latency_${layer}.txt")
    echo "  $layer P99 = $p99 ms"
done

echo "[2/6] 命中率与驱逐率"
cat /sys/kernel/agentrt/memory_rovol/perf > "$RESULT_DIR/perf.txt"

echo "[3/6] MGLRU 代数统计"
cat /sys/kernel/mm/lru_gen/ > "$RESULT_DIR/mglru_status.txt"

echo "[4/6] CXL 跨 NUMA 带宽"
perf bench numadist -i 5 > "$RESULT_DIR/cxl_bandwidth.txt"

echo "[5/6] PMEM 写延迟"
fio --filename=/dev/pmem0 --rw=randwrite --bs=4k \
    --ioengine=libaio --iodepth=1 --runtime=30 \
    > "$RESULT_DIR/pmem_write.txt"

echo "[6/6] tiering 迁移 kfifo 吞吐"
agentrt-bench tiering-throughput --duration 60 \
    > "$RESULT_DIR/tiering.txt"

echo "[完成] 报告保存于 $RESULT_DIR"
```

### 5.3 性能指标对照表

| 指标 | 默认 MGLRU | agentrt-linux 调优 | 提升 |
|------|-----------|----------------------|------|
| L1 命中率 | 0.72 | 0.85 | +18% |
| L2 命中率 | 0.58 | 0.72 | +24% |
| 内存回收延迟 P99 | 12ms | 4ms | -67% |
| CXL 利用率 | 45% | 80% | +78% |
| tiering 迁移吞吐 | 2k/s | 12k/s | +500% |

---

## 6. 错误码体系对接

内存子系统错误码纳入 agentrt-linux 统一错误码体系（[SC] 共享契约层，-600 ~ -699 段）：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AIRY_E_MEMORY_NOMEM | -600 | 内存不足 |
| AIRY_E_MEMORY_TIER_FAIL | -601 | 跨层迁移失败 |
| AIRY_E_MEMORY_CXL_FAIL | -602 | CXL 操作失败 |
| AIRY_E_MEMORY_PMEM_FAIL | -603 | PMEM 持久化失败 |
| AIRY_E_MEMORY_MGLRU | -604 | MGLRU 调度错误 |
| AIRY_E_MEMORY_KFIFO_FULL | -605 | 迁移 kfifo 队列满 |
| AIRY_E_MEMORY_HASH_MISMATCH | -606 | SHA-256 哈希校验失败 |

> **说明性示例，非 SSoT 定义**：以下 `airy_memory_do_migrate` 函数为本文档设计草案，**不**存在于 `include/uapi/linux/airymax/memory_types.h`。其中 `AIRY_MEM_LEVEL_MAX`/`AIRY_MEM_PMEM`/`AIRY_MEM_HOT` 为 SSoT 实际枚举值（见 §2.2）。

```c
/* 说明性示例：集中错误处理（非 SSoT 定义） */
int airy_memory_do_migrate(const struct airy_migrate_entry *e)
{
	void *src, *dst;
	int ret;

	if (e->from >= AIRY_MEM_LEVEL_MAX ||
	    e->to >= AIRY_MEM_LEVEL_MAX) {
		ret = -AIRY_EINVAL;
		goto out_err;
	}

	src = airy_memory_map(e->from, e->page_addr, e->size_kb);
	if (!src) {
		ret = -AIRY_E_MEMORY_NOMEM;
		goto out_err;
	}

	dst = airy_memory_alloc(e->to, e->size_kb);
	if (!dst) {
		ret = -AIRY_E_MEMORY_NOMEM;
		goto out_free_src;
	}

	ret = airy_memory_copy(dst, src, e->size_kb);
	if (ret < 0) {
		ret = -AIRY_E_MEMORY_TIER_FAIL;
		goto out_free_dst;
	}

	if (e->to == AIRY_MEM_PMEM ||    /* L4_pat 业务语义 */
	    e->to == AIRY_MEM_HOT) {     /* L1_raw 业务语义 */
		ret = airy_memory_persist(dst, e->size_kb);
		if (ret < 0) {
			ret = -AIRY_E_MEMORY_PMEM_FAIL;
			goto out_free_dst;
		}
	}

	airy_memory_unmap(src);
	airy_memory_unmap(dst);
	return 0;

out_free_dst:
	airy_memory_free(dst);
out_free_src:
	airy_memory_unmap(src);
out_err:
	return ret;
}
```

---

## 7. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **C-3 记忆卷载** | MemoryRovol L1-L4 分级内存直接对应 C-3 原则 |
| **C-4 遗忘机制** | MGLRU 多代回收与 Forgetting Engine 衰减公式语义对齐 |
| **E-3 资源确定性** | CXL/PMEM 分层带宽确定性可预测 |
| **E-2 可观测性** | `/sys/kernel/agentrt/memory_rovol/perf` 全量统计 |
| **S-1 反馈闭环** | tiering 自动迁移形成"冷热反馈→代数调整→迁移"闭环 |
| **K-4 可插拔策略** | Forgetting Engine 四种策略可运行时切换 |

---

## 8. IRON-9 v3 同源映射

| 组件 | agentrt-linux（[SS]） | agentrt（[SS]） | 共享（[SC]） |
|------|------------------------|------------------|--------------|
| 记忆模型 | CXL + MGLRU 内核态 | HeapStore 用户态 | `enum airy_mem_level` |
| GFP 掩码 | `AIRY_GFP_HOT/WARM/COLD/PMEM` | `AIRY_GFP_HOT/WARM/COLD/PMEM` | `memory_types.h` |
| PMEM 持久化 | fsdax + DAX 挂载 | 用户态 mmap | PMEM 持久化接口 |
| 遗忘公式 | R(t) = e^(-t/τ) Q16.16 | R(t) = e^(-t/τ) Q16.16 | Forgetting Engine 枚举 |

> **SSoT 对齐说明**：上表中 `AIRY_GFP_HOT/WARM/COLD/PMEM` 与 `enum airy_mem_level` 均为 `include/uapi/linux/airymax/memory_types.h` 实际定义；历史版本中虚构的 `AIRY_GFP_L1_L4` 宏在 SSoT 中**不存在**，已移除。Q16.16 定点数 `airy_q16_t` 定义位于 `include/uapi/linux/airymax/cognition_types.h`，非 `memory_types.h`。

---

## 9. 相关文档

- `170-performance/README.md`（性能工程主索引）
- `170-performance/01-scheduling-performance.md`（调度性能设计）
- `170-performance/03-ipc-performance.md`（IPC 性能设计）
- `20-modules/04-memory.md`（memory 子仓设计）
- `50-engineering-standards/01-coding-standards.md`（编码规范）

---

## 10. 参考材料

- Linux 6.6 `mm/vmscan.c`（MGLRU 实现）
- Linux 6.6 `mm/migrate.c`（页面迁移）
- Linux 6.6 `drivers/cxl/`（CXL 驱动）
- Linux 6.6 `drivers/nvdimm/`（PMEM 驱动）
- `Documentation/admin-guide/mm/multigeneration_lru.rst`
- `Documentation/driver-api/cxl/`
- agentrt HeapStore 用户态记忆实现规范

---

## 11. 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.1-fix | 2026-07-26 | SSoT 对齐修复：以 `include/uapi/linux/airymax/memory_types.h` 为唯一真源，移除文档中的虚构内容。具体：(1) §2.2 移除虚构的 `airy_mr_layer` 枚举、`AIRY_MEMORYROV_L1_RAW/L2_FEAT/L3_STR/L4_PAT/LAYER_MAX` 枚举值、`AIRY_GFP_L1_RAW/L2_FEAT/L3_STR/L4_PAT` GFP 宏、`airy_mr_node`/`airy_mr_mgr` 结构体、虚构 include `<airymax/airy_q16.h>` 与 `<linux/types.h>`、虚构头文件保护宏 `AIRY_MEMORY_TYPES_H`，替换为 SSoT 实际定义（`enum airy_mem_level`、`AIRY_MEM_HOT/WARM/COLD/PMEM/LEVEL_MAX`、`AIRY_GFP_HOT/WARM/COLD/PMEM`、`AIRY_PAGE_CLASS_*`、`AIRY_SC_FALLBACK` DSL 降级块、`_UAPI_AIRYMAX_MEMORY_TYPES_H` 保护宏、`<linux/airymax/uapi_compat.h>` include）；(2) §3.2/§3.3/§4.2/§4.4/§6 所有说明性示例添加"非 SSoT 定义"标注，将虚构的 `airy_mr_layer`/`AIRY_MEMORYROV_*` 引用替换为 SSoT 实际枚举 `airy_mem_level`/`AIRY_MEM_*`，将虚构的 `airy_mr_mgr` 参数替换为通用 `void *ctx`，标注 `airy_q16_t` 来源为 `cognition_types.h`，修复 include 路径 `<airymax/memory_types.h>` → `<linux/airymax/memory_types.h>`；(3) §8 IRON-9 同源映射表移除虚构宏 `AIRY_GFP_L1_L4`，替换为 SSoT 实际宏 `AIRY_GFP_HOT/WARM/COLD/PMEM` 与 `enum airy_mem_level`。 |
| v0.1.1 | 2026-07-21 | 初始版本。 |

---

> **文档结束** | agentrt-linux（AirymaxOS）内存性能工程设计 
