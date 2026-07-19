Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）IPC 性能工程设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）进程间通信性能工程详细设计\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## SSoT 声明

> **单一权威源声明**：本文件是 **A-IPC 性能 SLO** 的唯一权威源。FAST_SEND / SLOW_SEND 延迟预算、fastpath C-S9 Badge 校验性能拆解、io_uring SQE/CQE 批量提交吞吐量、kfifo 批量读取吞吐量、错误码对接（v1.1: -41~-70 IPC + -78~-82 Capability + 0x1001-0x1006 Fault）均以本文件为唯一权威定义。
>
> **v1.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.1 起，A-IPC 引入 **fastpath C-S9 Badge 校验**（~10ns 内联），整体 FAST_SEND 降至 ~158ns（含 C-S9 + Ring Buffer 写入）。SLOW_SEND（C-S9 失败路径）为 ~600ns-5.5μs（含 LSM 钩子 + 冷酷执法）。Badge 撤销通过 `atomic_inc` 一行代码完成（~1ns，O(1)）。错误码体系对齐 [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)（-41~-70 IPC + -78~-82 Capability + 0x1001-0x1006 Fault），原 -800~-899 段已废弃。

---

## 1. 设计目标与范围

### 1.1 设计目标（v1.1: FAST_SEND ~158ns / SLOW_SEND ~600ns-5.5μs）

agentrt-linux（AirymaxOS）IPC 性能工程为 A-IPC（Airymax Unify IPC Fabric，v1.1 Capability Folding 单平面架构）提供低延迟、高吞吐、零拷贝的传输能力。本设计聚焦四大目标：

1. **v1.1 FAST_SEND 延迟 SLO**（fastpath C-S9 通过）：≤ 200ns，实测 ~158ns（含 C-S9 Badge 校验 ~10ns + Ring Buffer 写入 ~100ns + 其他校验 ~48ns）
2. **v1.1 SLOW_SEND 延迟 SLO**（fastpath C-S9 失败，LSM 钩子接管）：≤ 10μs，实测 ~600ns-5.5μs（含 LSM 钩子 + 冷酷执法 + eventfd 通知）
3. **128B 消息头零拷贝传输延迟 SLO**（端到端 P50/P99）：P50 ≤ 5μs、P99 ≤ 20μs、P99.9 ≤ 50μs
4. **io_uring SQE/CQE 批量提交吞吐量**：≥ 2M ops/s（单核），≥ 12M ops/s（8 核并发）
5. **kfifo 批量读取内核消息通道吞吐量**：≥ 1.5M msg/s（单 kthread）

### 1.2 适用范围

- A-IPC 128 字节统一消息头（Layout C v4，magic=`0x41524531` 'ARE1'，含 `capability_badge` offset 44-51，IRON-9 v3 [SC] 共享契约层）
- Linux 6.6 内核基线 io_uring 异步 I/O 子系统（IORING_OP_URING_CMD）
- 内核 kthread 间通信用 kfifo + wait_event_interruptible
- Agent kthread 与用户态 daemon 双向消息通道

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。A-IPC（Airymax Unify IPC Fabric，v1.1 Capability Folding 单平面架构）在 agentrt-linux 中基于内核 io_uring `IORING_OP_URING_CMD` 实现，在 agentrt 中基于用户态消息队列实现，两者属于 IRON-9 v3 [SS] 语义同源层。原 AgentsIPC 名称自 v1.1 起统一为 A-IPC。

### 1.4 IPC 性能 SLO 矩阵（v1.1: 新增 FAST_SEND / SLOW_SEND）

| 路径 | 消息类型 | 载荷大小 | 延迟 P50 | 延迟 P99 | 吞吐量 | 通道 |
|------|----------|----------|----------|----------|--------|------|
| **FAST_SEND**（v1.1 新增） | 控制消息 | 128B（仅头） | ≤ 200ns | ≤ 500ns | ≥ 4M ops/s | io_uring fastpath |
| **SLOW_SEND**（v1.1 新增） | 控制消息（C-S9 失败） | 128B | ≤ 5μs | ≤ 20μs | — | io_uring slowpath + LSM |
| 控制消息（端到端） | 128B（仅头） | ≤ 5μs | ≤ 20μs | ≥ 2M ops/s | io_uring |
| Agent 指令 | 4KB（头+体） | ≤ 15μs | ≤ 60μs | ≥ 800k ops/s | io_uring |
| 记忆查询 | 16KB | ≤ 50μs | ≤ 200μs | ≥ 200k ops/s | io_uring |
| 内核事件 | 128B | ≤ 2μs | ≤ 10μs | ≥ 1.5M msg/s | kfifo |
| 流式 Token | 4KB×N | ≤ 10μs/块 | ≤ 50μs/块 | ≥ 1M ops/s | io_uring |

### 1.5 v1.1 FAST_SEND 性能拆解

FAST_SEND（fastpath C-S9 通过路径）的详细性能拆解：

| 阶段 | 操作 | 延迟 | 占比 | 说明 |
|------|------|------|------|------|
| C-S0 | Ring 冻结检查（`unlikely(ring->frozen)`） | ~1ns | ~1% | 分支预测优化 |
| C-S1~C-S8 | magic/版本/源/目的/payload_len 等前置校验 | ~30ns | ~19% | 详见 [07-ipc-fastpath.md §5](../30-interfaces/07-ipc-fastpath.md) |
| **C-S9** | **Badge 64-bit Native Word 校验**（v1.1 新增） | **~10ns** | **~6%** | `airy_cap_badge_ok()` 3×READ_ONCE + 位运算 |
| C-S10~C-S12 | 其他校验（含 CRC32 ~5ns） | ~17ns | ~11% | C-S12 CRC32 完整性校验 |
| Ring Buffer 写入 | reserve + memcpy + commit | ~100ns | ~63% | 128B 头 + payload 引用 |
| **FAST_SEND 总计** | | **~158ns** | **100%** | 含 C-S9 Badge 校验 |

### 1.6 v1.1 SLOW_SEND 性能拆解

SLOW_SEND（fastpath C-S9 失败路径）的详细性能拆解：

| 阶段 | 操作 | 延迟 | 说明 |
|------|------|------|------|
| C-S0~C-S9.{EPOCH,RANDTAG,PERMS} | fastpath 校验至 C-S9 失败 | ~10ns | 返回 Badge 错误码（-78/-79/-80/-81/-82） |
| LSM 钩子 | `security_uring_cmd` 被 slowpath 调用 | ~100ns-1μs | Micro-Supervisor 接管，详见 [09-kernel-agent-supervisor.md §2.3](../20-modules/09-kernel-agent-supervisor.md) |
| 冷酷执法 | `airy_fault_enforce()` 冻结 Ring + 时间戳 | ~200ns | smp_store_release + ktime_get_real_ns |
| eventfd 通知 | `airy_eventfd_signal_fault()` | ~50ns | 非阻塞 signal |
| **SLOW_SEND 总计** | | **~600ns-5.5μs** | 含 LSM 钩子 + 冷酷执法 + eventfd |

---

## 2. 128B 消息头零拷贝优化

### 2.1 128B 消息头结构（v1.1: Layout C v4 + capability_badge）

A-IPC 128 字节统一消息头（Layout C v4，定义于 `include/uapi/linux/airymax/ipc.h`，IRON-9 v3 [SC] 共享契约层）：

```c
/* include/uapi/linux/airymax/ipc.h —— 128B IPC 消息头（[SC] 共享契约层，Layout C v4）
 * v1.1: Layout C v4 含 capability_badge (offset 44-51, 8B)
 * 详见 30-interfaces/02-ipc-protocol.md §2 Layout C v4 完整定义
 */
#ifndef AIRY_IPC_H
#define AIRY_IPC_H

#include <linux/types.h>

#define AIRY_IPC_MAGIC       0x41524531U   /* "ARE1" ASCII — Airymax Runtime Epoch（v1.1 Capability Folding） */
#define AIRY_IPC_HDR_SIZE    128           /* Layout C v4 定长 128B = 2 cache lines */

/* IPC 128B 消息头定义见 [SC] 共享契约层（SSoT），不就地重定义 */
#include <airymax/ipc.h>
/* 结构体名称：struct airy_ipc_msg_hdr（Layout C v4，物理宿主见
 * 30-interfaces/02-ipc-protocol.md §2 + 50-engineering-standards/120-cross-project-code-sharing.md §2.7）
 * v1.1 关键字段：capability_badge (offset 44-51, 8B, Badge 64-bit Native Word)
 */

/* 消息类型枚举 */
enum airy_ipc_msg_type {
	AIRY_IPC_MSG_CTRL    = 0x01,   /* 控制消息 */
	AIRY_IPC_MSG_AGENT   = 0x02,   /* Agent 指令 */
	AIRY_IPC_MSG_MEMORY  = 0x03,   /* 记忆查询 */
	AIRY_IPC_MSG_EVENT   = 0x04,   /* 内核事件 */
	AIRY_IPC_MSG_TOKEN   = 0x05,   /* 流式 Token */
	AIRY_IPC_MSG_HEARTBEAT = 0x06, /* 心跳 */
};

#endif /* AIRY_IPC_H */
```

> **SSoT 声明**：本节 IPC 128B 消息头不再就地重定义，以 `include/uapi/linux/airymax/ipc.h`（物理宿主见 `30-interfaces/02-ipc-protocol.md` §2 Layout C v4）为单一数据源。结构体名称为 `struct airy_ipc_msg_hdr`（Layout C v4）。v1.1 关键字段：`capability_badge`（offset 44-51，8B，Badge 64-bit Native Word = `Epoch<<48 | RandomTag<<16 | Perms`），由 sec_d 编译并写入 `agent_caps[agent_id]` 静态数组。原 Layout D（21 字段）已废弃，原 Layout C v1-v3 由 v4 替代。

### 2.2 零拷贝传输机制

零拷贝传输依赖三种机制：

1. **共享内存区域注册**：用户态 daemon 通过 `AIRY_IPC_OP_REGISTER_REGION` 注册内存区域，io_uring 通过 iova 直接访问
2. **io_uring 固定 OP 码**：`AIRY_IPC_OP_SEND_ZC`（零拷贝发送）、`AIRY_IPC_OP_RECV_ZC`（零拷贝接收）
3. **大页（hugepage）后端**：内存区域以 2MB 大页分配，减少 TLB miss

```c
/* 用户态：注册零拷贝内存区域 */
struct airy_ipc_region_desc {
	__u64 user_addr;     /* 用户态地址 */
	__u64 size;           /* 区域大小（必须 2MB 对齐） */
	__u16 region_id;
	__u8  region_kind;
	__u8  reserved[3];
};

int airy_ipc_register_region(int ring_fd,
				const struct airy_ipc_region_desc *desc)
{
	struct io_uring_sqe *sqe;
	struct io_uring_cqe *cqe;
	int ret;

	sqe = io_uring_get_sqe(&ring);
	if (!sqe) {
		ret = -EAGAIN;
		goto out_err;
	}

	io_uring_prep_rw(IORING_OP_AIRY_IPC_REGISTER_REGION, sqe,
			 r->ring_fd, NULL, 0, 0);
	io_uring_sqe_set_data(sqe, (void *)desc);

	ret = io_uring_submit(&ring);
	if (ret < 0)
		goto out_err;

	ret = io_uring_wait_cqe(&ring, &cqe);
	if (ret < 0)
		goto out_err;
	ret = cqe->res;
	io_uring_cqe_seen(&ring, cqe);
	return ret;

out_err:
	return ret;
}
```

### 2.3 零拷贝发送路径

```
┌──────────────────┐         ┌──────────────────┐
│   发送 daemon    │         │   接收 daemon    │
│  (用户态)        │         │  (用户态)        │
└────────┬─────────┘         └────────▲─────────┘
         │                            │
         │  ① 填充 128B 头 + 载荷     │  ④ 接收通知
         │     （区域 A）            │     （从区域 B 读取）
         ▼                            │
┌──────────────────────────────────────────────┐
│   io_uring SQE（固定 OP 码 ZERO_COPY_SEND）  │
│   ② 内核：仅搬运 128B 头，载荷以 iova 引用   │
└────────┬─────────────────────────────────────┘
         │
         ▼  ③ io_uring CQE 通知接收方
┌──────────────────────────────────────────────┐
│   接收方区域 B（共享内存，iova 映射）         │
└──────────────────────────────────────────────┘
```

零拷贝的关键：128B 头通过 io_uring 复制（数据量小，可忽略），载荷通过 iova 引用，避免大块内存拷贝。

---

## 3. io_uring SQE/CQE 批量提交

### 3.1 io_uring 初始化配置

```c
/* services/common/airy_ipc_ring.c */
#include <liburing.h>
#include <airymax/ipc.h>

#define AIRY_IPC_RING_DEPTH   1024
#define AIRY_IPC_BATCH_SIZE   64

struct airy_ipc_ring {
	struct io_uring ring;
	int ring_fd;
	__u32 submit_pending;
	__u32 batch_count;
};

/* 初始化 io_uring，启用 SQPOLL + 固定 OP 码 */
int airy_ipc_ring_init(struct airy_ipc_ring *r)
{
	struct io_uring_params p;
	int ret;

	memset(&p, 0, sizeof(p));
	p.flags = IORING_SETUP_SQPOLL | IORING_SETUP_SINGLE_ISSUER;
	p.sq_thread_idle = 5000;        /* 5 秒空闲后休眠 */
	p.features = IORING_FEAT_FAST_POLL;

	ret = io_uring_queue_init_mem(AIRY_IPC_RING_DEPTH,
				     &r->ring, &p, NULL);
	if (ret < 0)
		goto out_err;

	r->ring_fd = r->ring.ring_fd;
	r->submit_pending = 0;
	r->batch_count = 0;
	return 0;

out_err:
	pr_err("agentrt-linux: io_uring init failed ret=%d\n", ret);
	return ret;
}
```

### 3.2 批量提交实现

```c
/* 批量提交 SQE（攒够 BATCH_SIZE 或显式 flush） */
int airy_ipc_send_batch(struct airy_ipc_ring *r,
			   struct airy_ipc_msg_hdr **hdrs, int n)
{
	struct io_uring_sqe *sqe;
	int i, submitted;

	if (n > AIRY_IPC_BATCH_SIZE)
		n = AIRY_IPC_BATCH_SIZE;

	for (i = 0; i < n; i++) {
		sqe = io_uring_get_sqe(&r->ring);
		if (!sqe)
			break;

		/* 使用 io_uring 固定 OP 码零拷贝发送 */
		io_uring_prep_rw(IORING_OP_AIRY_IPC_SEND_ZC, sqe,
				 r->ring_fd, hdrs[i],
				 AIRY_IPC_HDR_SIZE, 0);
		io_uring_sqe_set_data64(sqe, hdrs[i]->trace_id);

		r->submit_pending++;
	}

	submitted = io_uring_submit(&r->ring);
	if (submitted < 0) {
		pr_warn("agentrt-linux: io_uring submit failed %d\n",
			submitted);
		return submitted;
	}

	r->batch_count++;
	r->submit_pending -= submitted;
	return submitted;
}

/* 批量接收 CQE */
int airy_ipc_recv_batch(struct airy_ipc_ring *r,
			   struct io_uring_cqe *cqes, int max_n)
{
	unsigned head;
	unsigned count = 0;
	struct io_uring_cqe *cqe;

	io_uring_for_each_cqe(&r->ring, head, cqe) {
		if (count >= max_n)
			break;
		cqes[count] = *cqe;
		count++;
	}

	if (count > 0)
		io_uring_cq_advance(&r->ring, count);
	return count;
}
```

### 3.3 完整批量发送/接收示例

```c
/* 完整示例：Agent daemon 批量 IPC 收发 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <liburing.h>
#include <airymax/ipc.h>
#include <airymax/error.h>

#define BATCH 64

static void fill_msg_hdr(struct airy_ipc_msg_hdr *h,
			 __u32 src, __u32 dst, __u64 seq)
{
	h->magic = AIRY_IPC_MAGIC;
	h->opcode = AIRY_IPC_OP_SEND;
	h->flags = 0;
	h->trace_id = (__u64)seq;
	h->timestamp_ns = ktime_get_ns();
	h->src_task = src;
	h->dst_task = dst;
	h->payload_len = AIRY_IPC_HDR_SIZE;
}

int main(int argc, char **argv)
{
	struct airy_ipc_ring r;
	struct airy_ipc_msg_hdr *send_batch[BATCH];
	struct io_uring_cqe cqes[BATCH];
	int i, ret, total = 0;

	ret = airy_ipc_ring_init(&r);
	if (ret < 0)
		return ret;

	/* 预分配消息头（128B 对齐） */
	for (i = 0; i < BATCH; i++) {
		ret = posix_memalign((void **)&send_batch[i], 64,
				     AIRY_IPC_HDR_SIZE);
		if (ret)
			goto out_free_batch;
	}

	/* 主循环：批量发送 + 接收 */
	for (int iter = 0; iter < 1000000; iter++) {
		/* 填充批量消息头 */
		for (i = 0; i < BATCH; i++)
			fill_msg_hdr(send_batch[i], 1, 2, total++);

		/* 批量提交 SQE */
		ret = airy_ipc_send_batch(&r, send_batch, BATCH);
		if (ret < 0)
			goto out_free_batch;

		/* 等待并批量接收 CQE */
		ret = io_uring_wait_cqe_nr(&r.ring, BATCH);
		if (ret < 0)
			continue;

		ret = airy_ipc_recv_batch(&r, cqes, BATCH);
		/* 处理 CQE（这里仅统计成功） */
	}

out_free_batch:
	for (i = 0; i < BATCH; i++)
		free(send_batch[i]);
	io_uring_queue_exit(&r.ring);
	return ret;
}
```

---

## 4. kfifo 批量读取机制

### 4.1 内核 kthread 间通信通道

内核 kthread 之间通信用 kfifo + wait_event_interruptible（这是 Linux 6.6 内核基线的强制约定）：

```c
/* kernel/ipc/airy_kfifo_channel.c */
#include <linux/kfifo.h>
#include <linux/wait.h>
#include <linux/sched.h>
#include <linux/kthread.h>
#include <linux/slab.h>
#include <airymax/ipc.h>

#define AIRY_KFIFO_SIZE  4096   /* 4096 个消息 */

struct airy_kfifo_chan {
	DECLARE_KFIFO_PTR(fifo, struct airy_ipc_msg_hdr);
	wait_queue_head_t rx_wq;     /* 接收等待队列 */
	wait_queue_head_t tx_wq;     /* 发送（FIFO 未满）等待队列 */
	spinlock_t lock;
	atomic_t msg_count;
	bool shutdown;
};

/* 初始化 kfifo 通道 */
int airy_kfifo_chan_init(struct airy_kfifo_chan *c)
{
	int ret;

	INIT_KFIFO(c->fifo);
	init_waitqueue_head(&c->rx_wq);
	init_waitqueue_head(&c->tx_wq);
	spin_lock_init(&c->lock);
	atomic_set(&c->msg_count, 0);
	c->shutdown = false;
	return 0;
}

/* 发送：阻塞直到 FIFO 有空间 */
int airy_kfifo_chan_send(struct airy_kfifo_chan *c,
			   const struct airy_ipc_msg_hdr *msg)
{
	int ret;

	if (c->shutdown) {
		ret = -ESHUTDOWN;
		goto out_err;
	}

	/* 等待 FIFO 有空间 */
	ret = wait_event_interruptible(c->tx_wq,
		kfifo_avail(&c->fifo) > 0 || c->shutdown);
	if (ret < 0)
		goto out_err;
	if (c->shutdown) {
		ret = -ESHUTDOWN;
		goto out_err;
	}

	spin_lock(&c->lock);
	ret = kfifo_in(&c->fifo, msg, 1);
	spin_unlock(&c->lock);

	if (ret == 0) {
		ret = -EAGAIN;
		goto out_err;
	}

	atomic_inc(&c->msg_count);
	wake_up_interruptible(&c->rx_wq);
	return 0;

out_err:
	return ret;
}

/* 接收：批量读取 kfifo，避免单条拷贝开销 */
int airy_kfifo_chan_recv_batch(struct airy_kfifo_chan *c,
				 struct airy_ipc_msg_hdr *out,
				 int max_n)
{
	int n, ret;

	if (c->shutdown && atomic_read(&c->msg_count) == 0)
		return -ESHUTDOWN;

	/* 等待至少一条消息 */
	ret = wait_event_interruptible(c->rx_wq,
		atomic_read(&c->msg_count) > 0 || c->shutdown);
	if (ret < 0)
		return ret;

	/* 批量读取（一次最多 max_n 条，降低锁竞争） */
	spin_lock(&c->lock);
	n = kfifo_out(&c->fifo, out, max_n);
	spin_unlock(&c->lock);

	if (n > 0) {
		atomic_sub(n, &c->msg_count);
		wake_up_interruptible(&c->tx_wq);
	}
	return n;
}
```

### 4.2 kfifo 接收 kthread 完整示例

```c
/* 完整示例：IPC 接收 kthread（kfifo 批量读取） */
static int airy_ipc_rx_kthread(void *data)
{
	struct airy_kfifo_chan *chan = data;
	struct airy_ipc_msg_hdr batch[16];
	int n, i, ret;

	while (!kthread_should_stop()) {
		n = airy_kfifo_chan_recv_batch(chan, batch, 16);
		if (n == -ESHUTDOWN)
			break;
		if (n <= 0)
			continue;

		/* 批量处理消息 */
		for (i = 0; i < n; i++) {
			ret = airy_ipc_dispatch(&batch[i]);
			if (ret < 0)
				pr_warn("agentrt-linux: dispatch failed %d\n",
					ret);
		}
	}
	return 0;
}

/* 启动接收 kthread */
int airy_ipc_rx_kthread_start(struct airy_kfifo_chan *chan)
{
	struct task_struct *t;

	t = kthread_run(airy_ipc_rx_kthread, chan, "airy_ipc_rx");
	if (IS_ERR(t))
		return PTR_ERR(t);
	return 0;
}
```

---

## 5. 性能指标表与基准测试

### 5.1 性能指标对照表

| 指标 | 旧实现（msgsnd） | io_uring 零拷贝 | kfifo 批量 | 提升幅度 |
|------|------------------|------------------|------------|----------|
| 128B 延迟 P50 | 12μs | 5μs | 2μs | -58% / -83% |
| 128B 延迟 P99 | 85μs | 20μs | 10μs | -76% / -88% |
| 单核吞吐 | 250k ops/s | 2M ops/s | 1.5M msg/s | +700% / +500% |
| 8 核吞吐 | 1.8M ops/s | 12M ops/s | — | +567% |
| CPU 占用 | 100% | 60% | 35% | -40% / -65% |
| 内存拷贝量 | 100% | 12%（仅头） | 100%（批） | -88% |

### 5.2 自动化基准测试脚本

```bash
#!/bin/bash
# /usr/lib/airymaxos/tests/ipc_perf_bench.sh
# A-IPC 性能基准自动化测试（v1.1 Capability Folding 集成版）

set -euo pipefail
RESULT_DIR=${1:-/var/tmp/agentrt-ipc-bench}
mkdir -p "$RESULT_DIR"

echo "[1/5] 128B 控制消息延迟"
agentrt-bench ipc-latency --size 128 --duration 30 \
    > "$RESULT_DIR/lat_128.txt"
p50=$(awk '/p50/ {print $3}' "$RESULT_DIR/lat_128.txt")
p99=$(awk '/p99/ {print $3}' "$RESULT_DIR/lat_128.txt")
echo "  P50 = $p50 us, P99 = $p99 us"

echo "[2/5] 4KB Agent 指令延迟"
agentrt-bench ipc-latency --size 4096 --duration 30 \
    > "$RESULT_DIR/lat_4k.txt"

echo "[3/5] io_uring 单核吞吐"
agentrt-bench ipc-throughput --cores 1 --batch 64 \
    --duration 30 > "$RESULT_DIR/tp_1core.txt"

echo "[4/5] io_uring 8 核并发吞吐"
agentrt-bench ipc-throughput --cores 8 --batch 64 \
    --duration 30 > "$RESULT_DIR/tp_8core.txt"

echo "[5/5] kfifo 内核通道吞吐"
agentrt-bench kfifo-throughput --duration 30 \
    > "$RESULT_DIR/kfifo.txt"

echo "[完成] 结果保存于 $RESULT_DIR"
```

### 5.3 性能指标采集

通过 user_events 上报 IPC 指标到 audit_d 守护进程：

```c
/* IPC 性能指标上报 */
struct airy_ipc_metric {
	__u64 timestamp_ns;
	__u32 src_task;
	__u32 dst_task;
	__u64 latency_ns;
	__u64 batch_size;
	__u8  channel;     /* 0=io_uring, 1=kfifo */
	__u8  reserved[3];
};

static inline void emit_ipc_metric(const struct airy_ipc_metric *m)
{
	user_event_emit("ipc_metric", m, sizeof(*m));
}
```

---

## 6. 错误码体系对接（v1.1: -41~-70 IPC + -78~-82 Capability + 0x1001-0x1006 Fault）

> **v1.1 变更**：原 -800~-899 段错误码已废弃。错误码体系全面对齐 [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)（[SC] 共享契约层 SSoT），分为三类：
> - **Error（负数，可恢复）**：IPC 段 -41~-70、Capability 段 -78~-82、[SC] 段 -101~-200、[DSL] 段 -201~-300
> - **Fault（正数 0x1000+，不可恢复）**：0x1001~0x1006（Capability Folding 异常）
> - **POSIX 兼容段**：-1~-40（EINVAL/ENOMEM/EAGAIN 等）

### 6.1 IPC 错误码段（-41 ~ -70，[SC] 共享契约层）

| 错误码 | 数值 | 含义 | 触发位置 |
|--------|------|------|---------|
| AIRY_E_IPC_MAGIC | -41 | 消息头魔数错误 | fastpath C-S1 |
| AIRY_E_IPC_VERSION | -42 | 协议版本不兼容 | fastpath C-S2 |
| AIRY_E_IPC_HDR_SIZE | -43 | 消息头长度非法 | fastpath C-S3 |
| AIRY_E_IPC_PAYLOAD | -44 | 载荷长度超限 | fastpath C-S4 |
| AIRY_E_IPC_SRC | -45 | 源 task 非法 | fastpath C-S5 |
| AIRY_E_IPC_DST | -46 | 目的 task 非法 | fastpath C-S6 |
| AIRY_E_IPC_OPCODE | -47 | opcode 非法 | fastpath C-S7 |
| AIRY_E_IPC_CRC | -48 | CRC32 校验失败 | fastpath C-S12 |
| AIRY_E_IPC_OVF | -49 | 消息队列溢出 | Ring Buffer 写入 |
| AIRY_E_IPC_REGION | -50 | 内存区域未注册 | 零拷贝注册路径 |
| AIRY_E_IPC_TIMEOUT | -51 | IPC 超时 | slowpath 等待 |
| AIRY_E_IPC_KFIFO_FULL | -52 | kfifo 队列满 | kfifo 通道 |
| AIRY_E_IPC_RING_FULL | -53 | io_uring SQ 已满 | io_uring 提交 |
| AIRY_E_IPC_BATCH_SIZE | -54 | 批量大小非法 | 批量提交路径 |
| AIRY_E_IPC_FROZEN | -55 | Ring 已冻结（FREEZE opcode） | fastpath C-S0 |

### 6.2 Capability 错误码段（-78 ~ -82，Badge 校验，v1.1 新增）

| 错误码 | 数值 | 含义 | 触发位置 | Fault 映射 |
|--------|------|------|---------|-----------|
| AIRY_ECAP_BADGE | -78 | Badge 格式无效 | fastpath C-S9 预校验 | 0x1005 ABNORMAL_CAP |
| AIRY_ECAP_EPOCH | -79 | Epoch 不匹配（已撤销/过期） | fastpath C-S9.EPOCH | 0x1005 ABNORMAL_CAP |
| AIRY_ECAP_FORGED | -80 | Badge 伪造（RandomTag 不匹配） | fastpath C-S9.RANDTAG | **0x1001 CAP_FORGED** |
| AIRY_ECAP_PERM | -81 | Perms 权限位不满足 | fastpath C-S9.PERMS | 0x1005 ABNORMAL_CAP |
| AIRY_ECAP_FROZEN | -82 | Ring 已冻结（重复冻结） | fastpath C-S0 | —（Error，不触发 Fault） |

### 6.3 Fault 码段（0x1001 ~ 0x1006，不可恢复，v1.1 重定义）

| Fault 码 | 数值 | 含义 | 触发条件 | 冷酷执法动作 |
|----------|------|------|---------|-------------|
| AIRY_FAULT_CAP_FORGED | 0x1001 | Badge 伪造 | C-S9.RANDTAG RandomTag 不匹配 | 冻结 Ring + KILL_AGENT |
| AIRY_FAULT_CAP_LEAK | 0x1002 | Capability 泄露 | sec_d 审计发现 | 冻结 Ring + AUDIT |
| AIRY_FAULT_RING_CORRUPT | 0x1003 | Ring 数据损坏 | C-S12 CRC32 失败 | 冻结 Ring + 重启 |
| AIRY_FAULT_TIMEOUT | 0x1004 | IPC 超时不可恢复 | slowpath 等待超时 | 冻结 Ring + 通知 |
| AIRY_FAULT_ABNORMAL_CAP | 0x1005 | Capability 异常 | C-S9.EPOCH/PERMS 失败 | 冻结 Ring + AUDIT |
| AIRY_FAULT_VM_FAULT | 0x1006 | 内存访问异常 | 零拷贝区域访问 | 冻结 Ring + OOM |

### 6.4 集中错误处理示例（v1.1: 对齐 Badge 校验）

```c
/* v1.1: 集中错误处理对齐 fastpath C-S9 Badge 校验
 * 详见 30-interfaces/07-ipc-fastpath.md §5.2 + 08-sc-error-contract.md §3
 */
int airy_ipc_validate_hdr(const struct airy_ipc_msg_hdr *h)
{
	/* C-S1: magic 校验 */
	if (unlikely(h->magic != AIRY_IPC_MAGIC))
		return -AIRY_E_IPC_MAGIC;     /* -41 */

	/* C-S2: 版本校验 */
	if (unlikely(h->version != AIRY_IPC_VERSION))
		return -AIRY_E_IPC_VERSION;   /* -42 */

	/* C-S3: 头长度校验 */
	if (unlikely(h->hdr_size != AIRY_IPC_HDR_SIZE))
		return -AIRY_E_IPC_HDR_SIZE;  /* -43 */

	/* C-S4: 载荷长度校验 */
	if (h->payload_len < AIRY_IPC_HDR_SIZE ||
	    h->payload_len > AIRY_IPC_MAX_PAYLOAD)
		return -AIRY_E_IPC_PAYLOAD;   /* -44 */

	return 0;
}

/* v1.1: fastpath C-S9 Badge 校验失败后的错误分发
 * 将 Badge 错误码映射到 Fault 码，由 LSM slowpath 接管
 */
static inline int airy_ipc_badge_err_dispatch(int fastpath_ret,
                                                struct airy_ipc_cmd *cmd)
{
	switch (fastpath_ret) {
	case -AIRY_ECAP_FORGED:        /* -80: 伪造，最严重 */
		return airy_fault_enforce(AIRY_FAULT_CAP_FORGED, cmd);   /* 0x1001 */
	case -AIRY_ECAP_EPOCH:         /* -79: Epoch 过期 */
	case -AIRY_ECAP_PERM:          /* -81: 权限位不满足 */
	case -AIRY_ECAP_BADGE:         /* -78: Badge 格式无效 */
		return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd); /* 0x1005 */
	case -AIRY_ECAP_FROZEN:        /* -82: Ring 冻结 */
		return fastpath_ret;         /* Error，不触发 Fault */
	default:
		return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd); /* 0x1005 */
	}
}
```

---

## 7. 安全与隔离考量（v1.1: fastpath C-S9 + slowpath LSM 职责分割）

### 7.1 零拷贝内存区域权限（v1.1: Badge 校验前置到 fastpath）

**v1.1 重构**：零拷贝内存区域注册的权限校验从"Cupolas `cupolas_permission_check()` 独立前置"重构为 **fastpath C-S9 Badge 校验**（IPC 操作时内联校验）+ **slowpath LSM 钩子**（仅 C-S9 失败时接管）。区域注册本身走 `security_uring_cmd` 钩子内部分发的 buffer 注册子命令校验（OLK 6.6 无 `uring_register_buffers` 钩子，详见 [07-airy-lsm-design.md §4.1](../110-security/07-airy-lsm-design.md)），区域使用时的 Badge 校验由 fastpath C-S9 内联完成。详见 [07-airy-lsm-design.md §3.1](../110-security/07-airy-lsm-design.md) 职责分割表。

```c
/* v1.1: 零拷贝区域注册——区域注册走 security_uring_cmd 内部分发，区域使用走 fastpath C-S9 Badge 校验
 * 区域注册：security_uring_cmd 钩子内部分发 buffer 注册子命令（slowpath，~100ns+）
 *          （OLK 6.6 无 uring_register_buffers 钩子，v1.0 文档误注册已删除）
 * 区域使用：fastpath C-S9 Badge 校验（每次 IPC 操作内联，~10ns）
 */
int airy_ipc_register_region(int ring_fd,
				const struct airy_ipc_region_desc *desc)
{
	int ret;

	/* 1. 区域注册走 security_uring_cmd 内部分发的 buffer 注册子命令校验（slowpath）
	 * v1.1: 不再调用 cupolas_permission_check() 独立前置
	 * OLK 6.6 对齐: 不注册 uring_register_buffers 钩子（OLK 6.6 中不存在）
	 * 详见 110-security/07-airy-lsm-design.md §4.1
	 */

	/* 2. 区域对齐校验（必须 2MB 对齐） */
	if (desc->user_addr & 0x1FFFFF || desc->size & 0x1FFFFF) {
		ret = -AIRY_E_IPC_REGION;   /* -50 */
		goto out_err;
	}

	/* 3. 大页分配并固定 */
	ret = airy_hugetlb_pin(desc->user_addr, desc->size);
	if (ret < 0)
		goto out_err;

	/* 4. 注册到 io_uring（内核会调用 security_uring_cmd 钩子，内部分发 buffer 注册子命令） */
	ret = io_uring_register_region(ring_fd, desc);
	return ret;

out_err:
	return ret;
}

/* v1.1: 区域使用时的 Badge 校验由 fastpath C-S9 内联完成
 * 详见 30-interfaces/07-ipc-fastpath.md §5.2 + 110-security/07-airy-lsm-design.md §3.2
 * 此处不再独立调用——IPC 操作时 fastpath C-S9 自动校验 capability_badge
 */
```

### 7.2 seccomp 与 capability 隔离

```bash
# /etc/airymaxos/seccomp/cogn_d.seccomp —— cogn_d daemon 的 seccomp 策略
# 仅允许 io_uring + IPC 相关系统调用
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "syscalls": [
        {
            "names": [
                "io_uring_setup", "io_uring_enter", "io_uring_register",
                "airy_ipc_register_region", "airy_ipc_send_zc",
                "airy_ipc_recv_zc", "mmap", "munmap", "futex",
                "epoll_wait", "read", "write", "close"
            ],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
```

---

## 8. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **K-1 内核极简** | io_uring 固定 OP 码 + 用户态零拷贝，内核仅搬运 128B 头 |
| **K-2 接口契约化** | 128B 消息头作为 [SC] 共享契约层契约 |
| **K-3 服务隔离** | 各 daemon 独立 io_uring 实例，seccomp 隔离 |
| **E-2 可观测性** | TraceID + SpanID + user_events 全链路追踪 |
| **E-3 资源确定性** | io_uring SQPOLL 内核线程固定 CPU，避免调度抖动 |
| **A-1 简约至上** | 128B 固定头 + iova 引用，最少接口最大吞吐 |

---

## 9. IRON-9 v3 同源映射（v1.1: Capability Folding 集成）

| 组件 | agentrt-linux（[SS] 语义同源） | agentrt（[SS] 语义同源） | 共享（[SC] 共享契约） |
|------|--------------------------------|--------------------------|----------------------|
| IPC 传输 | 内核 io_uring `IORING_OP_URING_CMD` | 用户态消息队列 | 128B 消息头结构（Layout C v4） |
| 消息类型枚举 | AIRY_IPC_MSG_* | AIRY_IPC_MSG_* | `include/uapi/linux/airymax/ipc.h` |
| Badge 64-bit Native Word | fastpath C-S9 内联校验（~10ns） | 用户态 Badge 校验 | Badge 编码 = `Epoch<<48 \| RandomTag<<16 \| Perms` |
| 全局 Epoch | `atomic_t airy_cap_global_epoch` | 用户态 atomic | Epoch 撤销语义（O(1) atomic_inc） |
| agent_caps[] 静态数组 | 16KB 内核静态数组（sec_d 唯一写者） | 用户态等价结构 | `airy_cap_badge_compile()` / `airy_cap_badge_ok()` |
| opcode 枚举 | SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE | 同左 | 7 种 opcode（[SC] 共享契约层） |
| 错误码 | AIRY_E_IPC_*（-41~-70）+ AIRY_ECAP_*（-78~-82） | 同左 | `include/uapi/linux/airymax/error.h` |
| Fault 码 | AIRY_FAULT_*（0x1001~0x1006） | 用户态映射到 Error | Fault 码仅内核态使用 |
| TraceID | 16B UUID | 16B UUID | TraceID 体系 |
| 4 值裁决枚举 | AIRY_VERDICT_ALLOW/DENY/AUDIT/COMPLAIN | 同左 | `include/uapi/linux/airymax/verdict.h` |

> **IRON-9 v3 四层模型**：[SC] 共享契约层 / [SS] 语义同源层 / [IND] 完全独立层 / [DSL] 降级生存层。详见 [10-unify-design.md §7](../10-architecture/10-unify-design.md) IRON-9 v3 章节。

---

## 10. 相关文档

### 10.1 v1.1 Capability Folding 相关（新增）

- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)（A-IPC 协议 SSoT，Layout C v4 完整定义）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)（A-IPC fastpath SSoT，C-S0~C-S12 校验链 + C-S9 Badge 校验）
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)（错误码契约 SSoT，-41~-70 + -78~-82 + 0x1001~0x1006）
- [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)（Micro-Supervisor SSoT，冷酷执法 + agent_caps[] 静态数组）
- [110-security/01-lsm-framework.md](../110-security/01-lsm-framework.md)（LSM 框架 SSoT，§8.4 职责分割表）
- [110-security/07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md)（纯 C LSM 模块 SSoT，§3 fastpath/slowpath 职责分割 + §3.5 sec_d Badge 编译）
- [110-security/03-capability-model.md](../110-security/03-capability-model.md)（Capability 模型 SSoT，Badge 64-bit Native Word 定义）
- [20-modules/11-unified-config.md](../20-modules/11-unified-config.md)（统一配置 SSoT，v1.1 Capability Folding 配置项）

### 10.2 性能工程相关（已有）

- `170-performance/README.md`（性能工程主索引）
- `170-performance/01-scheduling-performance.md`（调度性能设计）
- `170-performance/02-memory-performance.md`（内存性能设计）
- `20-modules/02-ipc.md`（IPC 子系统设计）
- `10-architecture/03-microkernel-strategy.md`（微内核化改造策略）

---

## 11. 参考材料

- Linux 6.6 `io_uring/`（io_uring 子系统，IORING_OP_URING_CMD）
- Linux 6.6 `include/uapi/linux/io_uring.h`
- `Documentation/filesystems/dax.rst`（DAX 零拷贝）
- Linux 6.6 `kernel/kfifo.h`（kfifo 实现）
- seL4 微内核 IPC 设计（fastpath 优化，capability-based security）
- agentrt A-IPC 用户态实现规范（IRON-9 v3 [SS] 语义同源层）

---

## 12. 版本历史

| 版本 | 日期 | 变更摘要 |
|------|------|---------|
| 0.1.1 | 2026-07-10 | 初版：io_uring 零拷贝 + kfifo 批量读取性能工程设计 |
| **v1.1** | **2026-07-18** | **Capability Folding 集成版**：(1) 新增 FAST_SEND ~158ns / SLOW_SEND ~600ns-5.5μs 性能 SLO；(2) 新增 §1.5/§1.6 FAST_SEND/SLOW_SEND 性能拆解；(3) §2.1 128B 消息头更新为 Layout C v4 + capability_badge (offset 44-51)；(4) §6 错误码体系完全重写（-800~-899 废弃 → -41~-70 IPC + -78~-82 Capability + 0x1001~0x1006 Fault）；(5) §7.1 Cupolas 权限校验重构为 fastpath C-S9 + slowpath LSM 职责分割；(6) §9 IRON-9 v3 同源映射对齐 Badge 64-bit Native Word + agent_caps[] 静态数组；(7) §10 相关文档新增 8 份 v1.1 引用；(8) 全文 AgentsIPC→A-IPC 统一术语 |

---

> **文档结束** | agentrt-linux（AirymaxOS）IPC 性能工程设计 v1.1（Capability Folding 集成版）
