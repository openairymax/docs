Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）IPC 性能工程设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）进程间通信性能工程详细设计\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-09\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-linux（AirymaxOS）IPC 性能工程为 AgentsIPC（Agent 进程间通信）提供低延迟、高吞吐、零拷贝的传输能力。本设计聚焦三大目标：

1. **128B 消息头零拷贝传输延迟 SLO**：P50 ≤ 5μs、P99 ≤ 20μs、P99.9 ≤ 50μs
2. **io_uring SQE/CQE 批量提交吞吐量**：≥ 2M ops/s（单核），≥ 12M ops/s（8 核并发）
3. **kfifo 批量读取内核消息通道吞吐量**：≥ 1.5M msg/s（单 kthread）

### 1.2 适用范围

- AgentsIPC 128 字节统一消息头（magic=0x41524531，IRON-9 v2 [SC] 共享契约层）
- Linux 6.6 内核基线 io_uring 异步 I/O 子系统
- 内核 kthread 间通信用 kfifo + wait_event_interruptible
- Agent kthread 与用户态 daemon 双向消息通道

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。AgentsIPC 在 agentrt-linux 中基于内核 io_uring 固定 OP 码实现，在 agentrt 中基于用户态消息队列实现，两者属于 IRON-9 v2 [SS] 语义同源层。

### 1.4 IPC 性能 SLO 矩阵

| 消息类型 | 载荷大小 | 延迟 P50 | 延迟 P99 | 吞吐量 | 通道 |
|----------|----------|----------|----------|--------|------|
| 控制消息 | 128B（仅头） | ≤ 5μs | ≤ 20μs | ≥ 2M ops/s | io_uring |
| Agent 指令 | 4KB（头+体） | ≤ 15μs | ≤ 60μs | ≥ 800k ops/s | io_uring |
| 记忆查询 | 16KB | ≤ 50μs | ≤ 200μs | ≥ 200k ops/s | io_uring |
| 内核事件 | 128B | ≤ 2μs | ≤ 10μs | ≥ 1.5M msg/s | kfifo |
| 流式 Token | 4KB×N | ≤ 10μs/块 | ≤ 50μs/块 | ≥ 1M ops/s | io_uring |

---

## 2. 128B 消息头零拷贝优化

### 2.1 128B 消息头结构

AgentsIPC 128 字节统一消息头（定义于 `include/airymax/ipc.h`，IRON-9 v2 [SC] 共享契约层）：

```c
/* include/airymax/ipc.h —— 128B IPC 消息头（[SC] 共享契约层，物理宿主见 SSoT §Layout C） */
#ifndef AIRY_IPC_H
#define AIRY_IPC_H

#include <linux/types.h>

#define AIRY_IPC_MAGIC       0x41524531U   /* "ARE1" ASCII */
#define AIRY_IPC_HDR_SIZE    128

/* IPC 128B 消息头定义见 [SC] 共享契约层（SSoT），不就地重定义 */
#include <airymax/ipc.h>
/* 结构体名称：struct airy_ipc_msg_hdr（Layout C，物理宿主见
 * 50-engineering-standards/120-cross-project-code-sharing.md §Layout C） */
/* 注意：原 Layout D（21 字段零拷贝布局）已废弃，统一引用 SSoT Layout C。 */

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

> **SSoT 声明**：本节 IPC 128B 消息头不再就地重定义，以 `include/airymax/ipc.h`（物理宿主见 `50-engineering-standards/120-cross-project-code-sharing.md` §Layout C）为单一数据源。结构体名称为 `struct airy_ipc_msg_hdr`（Layout C）。 原 Layout D（21 字段）已废弃。

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
# AgentsIPC 性能基准自动化测试

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

## 6. 错误码体系对接

IPC 子系统错误码纳入 agentrt-linux 统一错误码体系（[SC] 共享契约层，-800 ~ -899 段）：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AIRY_E_IPC_MAGIC | -800 | 消息头魔数错误 |
| AIRY_E_IPC_CRC | -801 | CRC32 校验失败 |
| AIRY_E_IPC_OVF | -802 | 消息队列溢出 |
| AIRY_E_IPC_REGION | -803 | 内存区域未注册 |
| AIRY_E_IPC_TIMEOUT | -804 | IPC 超时 |
| AIRY_E_IPC_KFIFO_FULL | -805 | kfifo 队列满 |
| AIRY_E_IPC_RING_FULL | -806 | io_uring SQ 已满 |
| AIRY_E_IPC_BATCH_SIZE | -807 | 批量大小非法 |

集中错误处理示例：

```c
int airy_ipc_validate_hdr(const struct airy_ipc_msg_hdr *h)
{
	if (h->magic != AIRY_IPC_MAGIC)
		return -AIRY_EINVAL;

	if (h->payload_len < AIRY_IPC_HDR_SIZE)
		return -AIRY_EINVAL;

	if (h->payload_len > AIRY_IPC_MAX_PAYLOAD)
		return -AIRY_EINVAL;

	return 0;
}

int airy_ipc_send_validate(struct airy_ipc_ring *r,
			      const struct airy_ipc_msg_hdr *h)
{
	int ret;

	ret = airy_ipc_validate_hdr(h);
	if (ret < 0)
		goto out_err;

	ret = airy_ipc_send_batch(r, (struct airy_ipc_msg_hdr **)&h, 1);
	if (ret < 0)
		goto out_err;

	return 0;

out_err:
	pr_warn("agentrt-linux: ipc send validate failed ret=%d\n", ret);
	return ret;
}
```

---

## 7. 安全与隔离考量

### 7.1 零拷贝内存区域权限

零拷贝内存区域必须经 Cupolas 安全穹顶权限仲裁：

```c
/* 零拷贝区域注册前的权限校验 */
int airy_ipc_register_region(int ring_fd,
				const struct airy_ipc_region_desc *desc)
{
	struct cupolas_perm perm;
	int ret;

	/* 1. 权限仲裁（Cupolas，[SS] 语义同源层） */
	perm.op = CUPOLAS_OP_IPC_REGION_REGISTER;
	perm.src_task = current->pid;
	perm.size = desc->size;
	ret = cupolas_permission_check(&perm);
	if (ret != CUPOLAS_PERMIT)
		goto out_deny;

	/* 2. 区域对齐校验（必须 2MB 对齐） */
	if (desc->user_addr & 0x1FFFFF || desc->size & 0x1FFFFF) {
		ret = -AIRY_EINVAL;
		goto out_err;
	}

	/* 3. 大页分配并固定 */
	ret = airy_hugetlb_pin(desc->user_addr, desc->size);
	if (ret < 0)
		goto out_err;

	/* 4. 注册到 io_uring */
	ret = io_uring_register_region(ring_fd, desc);
	return ret;

out_deny:
	pr_warn("agentrt-linux: ipc region register denied by cupolas\n");
	return -AIRY_E_IPC_REGION;
out_err:
	return ret;
}
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

## 9. IRON-9 v2 同源映射

| 组件 | agentrt-linux（[SS]） | agentrt（[SS]） | 共享（[SC]） |
|------|------------------------|------------------|--------------|
| IPC 传输 | 内核 io_uring 固定 OP 码 | 用户态消息队列 | 128B 消息头结构 |
| 消息类型枚举 | AIRY_IPC_MSG_* | AIRY_IPC_MSG_* | `ipc.h` |
| TraceID | 16B UUID | 16B UUID | TraceID 体系 |
| 错误码 | AIRY_E_IPC_* | AIRY_E_IPC_* | `error.h` 错误码段 |

---

## 10. 相关文档

- `170-performance/README.md`（性能工程主索引）
- `170-performance/01-scheduling-performance.md`（调度性能设计）
- `170-performance/02-memory-performance.md`（内存性能设计）
- `20-modules/02-ipc.md`（IPC 子系统设计）
- `10-architecture/03-microkernel-strategy.md`（微内核化改造策略）

---

## 11. 参考材料

- Linux 6.6 `io_uring/`（io_uring 子系统）
- Linux 6.6 `include/uapi/linux/io_uring.h`
- `Documentation/filesystems/dax.rst`（DAX 零拷贝）
- Linux 6.6 `kernel/kfifo.h`（kfifo 实现）
- seL4 微内核 IPC 设计（fastpath 优化）
- agentrt AgentsIPC 用户态实现规范

---

> **文档结束** | agentrt-linux（AirymaxOS）IPC 性能工程设计 
