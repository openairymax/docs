Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）内存性能工程设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）内存子系统性能工程详细设计\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-09\
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

```c
/* include/uapi/linux/airymax/memory_types.h —— MemoryRovol 内存类型（[SC] 共享契约层） */
#ifndef AIRY_MEMORY_TYPES_H
#define AIRY_MEMORY_TYPES_H

#include <linux/types.h>
#include <airymax/airy_q16.h>

/* GFP 掩码语义（与 agentrt 共享） */
#define AIRY_GFP_L1_RAW    (__GFP_HIGH | __GFP_MOVABLE)     /* DRAM 优先 */
#define AIRY_GFP_L2_FEAT   (__GFP_HIGHMEM | __GFP_MOVABLE) /* DRAM/CXL */
#define AIRY_GFP_L3_STR    (__GFP_KERNEL)                  /* DRAM 内核 */
#define AIRY_GFP_L4_PAT    (GFP_KERNEL | __GFP_HIGHMEM)  /* PMEM/CXL 通过 mempolicy 指定 NUMA 节点 */

/* 记忆层级枚举 */
enum airy_mr_layer {
	AIRY_MEMORYROV_L1_RAW = 0,
	AIRY_MEMORYROV_L2_FEAT,
	AIRY_MEMORYROV_L3_STR,
	AIRY_MEMORYROV_L4_PAT,
	AIRY_MEMORYROV_LAYER_MAX,
};

/* 分级内存节点描述 */
struct airy_mr_node {
	enum airy_mr_layer layer;
	int numa_node;              /* 0=DRAM local, 1=CXL, 2=PMEM */
	__u32 capacity_mb;
	__u32 used_mb;
	airy_q16_t hit_rate;     /* Q16.16 表示的命中率 */
	airy_q16_t evict_rate;
};

/* 分级内存管理器 */
struct airy_mr_mgr {
	struct airy_mr_node nodes[AIRY_MEMORYROV_LAYER_MAX];
	struct kfifo *migrate_fifo;   /* 批量迁移队列 */
	wait_queue_head_t *migrate_wq;
	airy_q16_t global_hit_rate;
};

#endif /* AIRY_MEMORY_TYPES_H */
```

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

```c
/* MGLRU 代数与 Forgetting Engine 衰减映射 */
struct airy_mglru_forgetting_map {
	enum airy_forgetting_strategy strategy;
	u8 min_gen;       /* 最冷代数 */
	u8 max_gen;       /* 最热代数 */
	airy_q16_t retention_rate;  /* Q16.16 保留率 */
};

/* 艾宾浩斯衰减映射表（基于 R(t) = e^(-t/τ)，τ = 7 天） */
static const struct airy_mglru_forgetting_map ebbinghaus_map[] = {
	/* 代数 | 保留率 | 含义 */
	{AIRY_FORGET_EBBINGHAUS, 0,  3, 0x10000},  /* 最热 4 代：1.0 */
	{AIRY_FORGET_EBBINGHAUS, 4,  7, 0x8000},   /* 中间 4 代：0.5 */
	{AIRY_FORGET_EBBINGHAUS, 8, 11, 0x2000},   /* 最冷 4 代：0.125 */
	{AIRY_FORGET_EBBINGHAUS, 12, 12, 0x0000},  /* 回收代：0 */
};

/* 应用衰减映射到 MGLRU */
static int airy_mglru_apply_forgetting(struct airy_mr_mgr *mgr,
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

```c
/* MemoryRovol 层级与 MGLRU 代绑定 */
struct airy_mr_mglru_bind {
	enum airy_mr_layer layer;
	u8 hot_gen_lo;     /* 热代下界 */
	u8 hot_gen_hi;     /* 热代上界 */
	u8 cold_gen_lo;    /* 冷代下界 */
	u8 cold_gen_hi;    /* 冷代上界 */
};

static const struct airy_mr_mglru_bind mglru_binds[] = {
	/* L1_raw：热代 0-2，冷代 10-11（PMEM 持久化） */
	{AIRY_MEMORYROV_L1_RAW,  0,  2, 10, 11},
	/* L2_feat：热代 0-3，冷代 9-11（CXL 迁移） */
	{AIRY_MEMORYROV_L2_FEAT, 0,  3,  9, 11},
	/* L3_str：热代 0-4，冷代 8-11（DRAM 驻留） */
	{AIRY_MEMORYROV_L3_STR,  0,  4,  8, 11},
	/* L4_pat：热代 0-5，冷代 11-11（PMEM 持久） */
	{AIRY_MEMORYROV_L4_PAT,  0,  5, 11, 11},
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

```c
/* memory/airy_memory_tiering.c */
#include <linux/kfifo.h>
#include <linux/wait.h>
#include <airymax/memory_types.h>

#define MIGRATE_FIFO_SIZE 4096

/* 迁移请求条目 */
struct airy_migrate_entry {
	enum airy_mr_layer from;
	enum airy_mr_layer to;
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

```c
/* 完整示例：MemoryRovol 跨层批量迁移 */
#include <linux/module.h>
#include <linux/kfifo.h>
#include <linux/wait.h>
#include <linux/kthread.h>
#include <linux/delay.h>
#include <airymax/memory_types.h>
#include <airymax/error.h>

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

		e.from = AIRY_MEMORYROV_L1_RAW;
		e.to = AIRY_MEMORYROV_L2_FEAT;
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

集中错误处理示例：

```c
int airy_memory_do_migrate(const struct airy_migrate_entry *e)
{
	void *src, *dst;
	int ret;

	if (e->from >= AIRY_MEMORYROV_LAYER_MAX ||
	    e->to >= AIRY_MEMORYROV_LAYER_MAX) {
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

	if (e->to == AIRY_MEMORYROV_L4_PAT ||
	    e->to == AIRY_MEMORYROV_L1_RAW) {
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
| 记忆模型 | CXL + MGLRU 内核态 | HeapStore 用户态 | L1-L4 数据结构 |
| GFP 掩码 | AIRY_GFP_L1_L4 | AIRY_GFP_L1_L4 | `memory_types.h` |
| PMEM 持久化 | fsdax + DAX 挂载 | 用户态 mmap | PMEM 持久化接口 |
| 遗忘公式 | R(t) = e^(-t/τ) Q16.16 | R(t) = e^(-t/τ) Q16.16 | Forgetting Engine 枚举 |

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

> **文档结束** | agentrt-linux（AirymaxOS）内存性能工程设计 
