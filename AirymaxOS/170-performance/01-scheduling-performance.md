Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）调度性能工程设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）调度子系统性能工程详细设计
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-liunx（AirymaxOS）调度性能工程旨在为智能体工作负载提供确定性可预测的 CPU 调度能力。本设计聚焦于三大目标：

1. **Agent 延迟 SLO 100% 保障**：L1 认知延迟 ≤ 100ms、L2 规划延迟 ≤ 1s、L3 执行延迟 ≤ 10s，在 99 分位（P99）下达成
2. **Token 能效最大化**：每瓦功耗处理的 Token 数（Token/Watt）较默认 CFS 调度器提升 ≥ 30%
3. **内核态禁用浮点约束下的高精度 vtime 计算**：采用 Q16.16 定点数表达虚拟时间，与 Linux 6.6 内核 EEVDF 调度器语义对齐

### 1.2 适用范围

- 基于 Linux 6.6 内核基线向前移植的 sched_ext 框架（SCHED_EXT=7）
- SCHED_AGENT 调度类（Agent 调度类，IRON-9 v2 [SS] 语义同源层）
- Agent kthread 与用户态 daemon 双向调度协同
- 与 agentrt（AirymaxAgentRT）的 MicroCoreRT 调度语义保持一致

### 1.3 术语规范

本设计严格遵守 agentrt-liunx 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-liunx（OS 发行版）称为**微内核**（micro-kernel）。禁止使用"微内核原语"描述 agentrt 的能力，正确表述为"微核心原语"。

### 1.4 性能分层与 SLO 指标

| 层级 | 指标 | 阈值 | 测量方式 |
|------|------|------|----------|
| L1 | 调度延迟（enqueue → dispatch） | ≤ 50μs | perf trace + ftrace |
| L2 | 上下文切换开销 | ≤ 2μs（同核）/ ≤ 8μs（跨核） | perf sched |
| L3 | Agent 任务 P99 响应延迟 | ≤ 100ms | user_events trace |
| L4 | Token/Watt 能效 | 较 CFS 提升 ≥ 30% | perf + RAPL 功耗 |
| L5 | 调度器抖动（jitter） | P99 ≤ 200μs | cyclictest + HPET |

---

## 2. sched_ext BPF 调度器架构

### 2.1 sched_ext 在 Linux 6.6 的部署

agentrt-liunx 将 sched_ext 框架从主线 6.12+ 向前移植到 Linux 6.6 内核基线。启用配置如下：

```kconfig
# airymaxos-kernel/.config（节选）
CONFIG_SCHED_CLASS_EXT=y
CONFIG_BPF_SCHED=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_SCHED_AGENT=y
# sched_ext 调度类编号固定为 7（SCHED_EXT=7）
```

### 2.2 SCHED_AGENT 调度类分层

SCHED_AGENT 调度类（Agent 调度类）位于调度层次中的位置：

```
┌────────────────────────────────────────────┐
│  SCHED_FIFO / SCHED_RR（实时类， prio 0-99）│  ← 实时任务（保留）
├────────────────────────────────────────────┤
│  SCHED_AGENT（Agent 调度类，SCHED_EXT=7） │  ← agentrt-liunx 专属
├────────────────────────────────────────────┤
│  SCHED_NORMAL / SCHED_BATCH（CFS + EEVDF）│  ← 默认负载
├────────────────────────────────────────────┤
│  SCHED_IDLE / SCHED_EXT（fallback）        │  ← 空闲与兜底
└────────────────────────────────────────────┘
```

SCHED_AGENT 高于 CFS 但低于实时类，确保 Agent 工作负载优先于普通进程但不抢占实时任务。

### 2.3 struct_ops BPF 钩子集

SCHED_AGENT 通过 BPF struct_ops 暴露下列钩子：

| 钩子 | 触发时机 | 职责 |
|------|----------|------|
| `agent_init` | 调度器加载 | 初始化 BPF map、队列 |
| `agent_exit` | 调度器卸载 | 释放资源、刷新统计 |
| `agent_enqueue` | 任务入队 | 计算 vtime、插入红黑树 |
| `agent_dequeue` | 任务出队 | 清理状态、更新预算 |
| `agent_dispatch` | 选下一个任务 | 按 vtime 最小者派发 |
| `agent_runnable` | 任务可运行 | 唤醒 kthread |
| `agent_stopping` | 任务停止 | 更新 vtime、统计 |
| `agent_quiescent` | 任务静默 | 资源回收 |

---

## 3. vtime 衰减公式设计

### 3.1 Q16.16 定点数表示

Linux 6.6 内核态禁用浮点（kernel_fpu 禁用），因此 vtime 必须以定点数表达。agentrt-liunx 采用 **Q16.16 定点数**（`airymax_q16_t`），整数部分 16 位、小数部分 16 位，定义于 `include/airymax/memory_types.h`（IRON-9 v2 [SC] 共享契约层）：

```c
/* include/airymax/airymax_q16.h —— Q16.16 定点数原语 */
#ifndef AIRYMAX_Q16_H
#define AIRYMAX_Q16_H

#include <linux/types.h>

typedef __s64 airymax_q16_t;       /* Q16.16 定点数 */
typedef __u32 airymax_weight_t;    /* 0-65535 权重 */

#define AIRYMAX_Q16_SHIFT       16
#define AIRYMAX_Q16_ONE         (((airymax_q16_t)1) << AIRYMAX_Q16_SHIFT)
#define AIRYMAX_Q16_HALF        (AIRYMAX_Q16_ONE >> 1)

/* 整数 ↔ Q16.16 */
static inline airymax_q16_t airymax_q16_from_int(__s64 v)
{
	return (airymax_q16_t)v << AIRYMAX_Q16_SHIFT;
}

static inline __s64 airymax_q16_to_int(airymax_q16_t v)
{
	return (__s64)(v >> AIRYMAX_Q16_SHIFT);
}

/* Q16.16 ↔ u32 权重（0-65535 映射到 0.0-1.0） */
static inline airymax_q16_t airymax_q16_from_weight(airymax_weight_t w)
{
	return (airymax_q16_t)w << (AIRYMAX_Q16_SHIFT - 16 + 16);
}

/* 乘法：a * b（其中一个必须是权重，避免溢出） */
static inline airymax_q16_t airymax_q16_mul_w(airymax_q16_t a, airymax_weight_t w)
{
	return (airymax_q16_t)((__s128)a * w >> 16);
}

/* 加法 / 减法 / 比较 */
static inline airymax_q16_t airymax_q16_add(airymax_q16_t a, airymax_q16_t b)
{
	return a + b;
}

static inline airymax_q16_t airymax_q16_sub(airymax_q16_t a, airymax_q16_t b)
{
	return a - b;
}

static inline bool airymax_q16_lt(airymax_q16_t a, airymax_q16_t b)
{
	return a < b;
}

#endif /* AIRYMAX_Q16_H */
```

### 3.2 vtime 衰减公式

借鉴 EEVDF（Earliest Eligible Virtual Deadline First），SCHED_AGENT 的 vtime 衰减公式如下（全部以 Q16.16 表达）：

```
vtime_new = vtime_old + delta_exec * weight / max_weight
```

其中：
- `delta_exec`：本次运行的实际时间（ns），Q16.16 表示
- `weight`：任务权重（0-65535，受 Token 预算、认知阶段影响）
- `max_weight`：基准权重（默认 32768，即 0.5）

衰减公式采用指数加权移动平均（EMA）使 vtime 平滑：

```
vtime_smooth = α * vtime_new + (1 - α) * vtime_old
```

其中 α = 0x4000（Q16.16 表示 0.25）。EMA 计算示例：

```c
/* SCHED_AGENT EMA vtime 平滑 */
static inline airymax_q16_t airymax_sched_agent_ema(airymax_q16_t old_v,
						    airymax_q16_t new_v)
{
	const airymax_q16_t alpha = 0x4000;     /* 0.25 */
	const airymax_q16_t one_minus_alpha = 0xC000; /* 0.75 */

	/* α * new_v + (1-α) * old_v，全部定点运算 */
	airymax_q16_t term1 = (airymax_q16_t)((__s128)new_v * alpha >> 16);
	airymax_q16_t term2 = (airymax_q16_t)((__s128)old_v * one_minus_alpha >> 16);
	return airymax_q16_add(term1, term2);
}
```

### 3.3 Token 预算感知的权重动态调整

Agent 任务的 weight 由三因子动态决定：

```
weight = base_weight * stage_factor * token_factor / 65536
```

| 因子 | 取值范围 | 含义 |
|------|----------|------|
| base_weight | 16384-49152 | 任务基础优先级 |
| stage_factor | 0x8000-0x10000 | 认知阶段因子（Cognition Engine 阶段最高） |
| token_factor | 0x4000-0x10000 | Token 预算因子（预算 < 20% 时升至 0x10000） |

```c
/* token 预算感知权重计算 */
static airymax_weight_t airymax_sched_agent_compute_weight(
	const struct agentrt_task_meta *meta)
{
	airymax_weight_t base = meta->base_weight;
	airymax_weight_t stage = meta->stage_factor;
	airymax_weight_t token;
	airymax_q16_t w;

	if (meta->token_budget < 20) {       /* 预算 < 20%，加速 */
		token = 0x10000;
	} else if (meta->token_budget < 50) { /* 50% 以下中等加速 */
		token = 0xC000;
	} else {
		token = 0x8000;
	}

	/* base * stage * token / 65536 / 65536 */
	w = (airymax_q16_t)((__s128)base * stage >> 16);
	w = (airymax_q16_t)((__s128)w * token >> 16);
	if (w > 0xFFFF)
		w = 0xFFFF;
	return (airymax_weight_t)w;
}
```

---

## 4. AI 工作负载 CPU 亲和性

### 4.1 Agent kthread 与 daemon 的亲和性策略

agentrt-liunx 的 12 个 daemon 与认知层 kthread 共享 CPU 资源。亲和性策略遵循以下规则：

1. **认知 kthread 绑定 NUMA-local CPU**：CoreLoopThree 的 L1/L2/L3 kthread 与对应 MemoryRovol 内存域同 NUMA 节点
2. **LLM 推理 daemon（llm_d）独占物理核**：通过 cpuset.cpus 隔离，避免被抢占
3. **网络 daemon（net_d）绑定 RX/TX 队列所在 CPU**：与 NIC RSS 队列对齐
4. **SMT 兄弟核优先给认知任务**：双思考的 t2 主思考占用物理核，t1 快思考占用 SMT 兄弟核

### 4.2 CPU 拓扑感知的调度映射

```c
/* agentrt-liunx CPU 亲和性映射表 */
struct agentrt_sched_agent_topo {
	int node_id;            /* NUMA 节点 */
	int die_id;            /* CPU die */
	int core_id;           /* 物理核 */
	int smt_sibling;      /* SMT 兄弟核 */
	enum agentrt_sched_agent_class cls;
};

/* 默认映射策略（基于 8 核 2 NUMA 示例） */
static const struct agentrt_sched_agent_topo default_topo[] = {
	{0, 0, 0, 4, AGENTRT_TOPO_COGNITION_T2},   /* CPU0,4 - 认知 t2 */
	{0, 0, 1, 5, AGENTRT_TOPO_COGNITION_T1},   /* CPU1,5 - 认知 t1 */
	{0, 0, 2, 6, AGENTRT_TOPO_LLM},            /* CPU2,6 - llm_d */
	{0, 0, 3, 7, AGENTRT_TOPO_NET},            /* CPU3,7 - net_d */
	{1, 0, 0, 4, AGENTRT_TOPO_MEMORY},         /* NUMA1 CPU0,4 - memory_d */
	{1, 0, 1, 5, AGENTRT_TOPO_SECURITY},        /* NUMA1 CPU1,5 - 安全 */
	{1, 0, 2, 6, AGENTRT_TOPO_IPC},            /* NUMA1 CPU2,6 - IPC */
	{1, 0, 3, 7, AGENTRT_TOPO_GENERIC},        /* NUMA1 CPU3,7 - 通用 */
};

/* 设置 CPU 亲和性 */
static int agentrt_sched_agent_set_affinity(struct task_struct *p,
					    const struct agentrt_sched_agent_topo *t)
{
	struct cpumask mask;
	int ret;

	cpumask_clear(&mask);
	cpumask_set_cpu(t->core_id, &mask);
	cpumask_set_cpu(t->smt_sibling, &mask);

	ret = sched_setaffinity(p->pid, &mask);
	if (ret < 0)
		goto out_err;
	return 0;

out_err:
	pr_warn("agentrt-liunx: set_affinity failed for pid=%d ret=%d\n",
		p->pid, ret);
	return ret;
}
```

### 4.3 cpuset 隔离配置

通过 cgroup v2 的 cpuset 子系统为关键 daemon 设置独占核：

```bash
# /etc/airymaxos/cpuset.conf —— agentrt-liunx cpuset 隔离配置
# 拓扑假设：8 核 / 2 NUMA / SMT=2

# llm_d 独占 CPU2, CPU6（物理核对）
mkdir /sys/fs/cgroup/llm_d
echo "2,6" > /sys/fs/cgroup/llm_d/cpuset.cpus
echo "0" > /sys/fs/cgroup/llm_d/cpuset.mems
echo "exclusive" > /sys/fs/cgroup/llm_d/cpu.isolated

# cognition kthread 绑定 CPU0, CPU4
mkdir /sys/fs/cgroup/cognition
echo "0,4" > /sys/fs/cognition/cpuset.cpus
echo "0" > /sys/fs/cgroup/cognition/cpuset.mems

# 默认任务使用 CPU1, CPU3, CPU5, CPU7
mkdir /sys/fs/cgroup/system
echo "1,3,5,7" > /sys/fs/cgroup/system/cpuset.cpus
```

---

## 5. BPF 调度策略完整示例

### 5.1 SCHED_AGENT BPF 程序

以下是一个完整的 SCHED_AGENT BPF 调度策略实现（BPF C，K&R 风格，Tab=8）：

```c
/* airymaxos-kernel/sched_ext/sched_agent_bpf.c */
#include <scx/common.bpf.h>
#include <airymax/airymax_q16.h>
#include <airymax/memory_types.h>

/* BPF map：任务状态 */
struct {
	__uint(type, BPF_MAP_TYPE_TASK_STORAGE);
	__uint(map_flags, BPF_F_NO_PREALLOC);
	__type(key, int);
	__type(value, struct agentrt_task_state);
} task_state SEC(".task_storage");

/* BPF map：调度统计 */
struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__type(key, u32);
	__type(value, u64);
	__uint(max_entries, 8);
} sched_stats SEC(".maps");

/* 全局 vtime 基线 */
volatile const airymax_q16_t vtime_now = 0;

/* 任务入队 */
s32 BPF_STRUCT_OPS(agent_enqueue, struct task_struct *p, u64 enq_flags)
{
	struct agentrt_task_state *st;
	airymax_q16_t vtime;

	st = bpf_task_storage_get(&task_state, p, 0,
				  BPF_LOCAL_STORAGE_GET_F_CREATE);
	if (!st)
		return -ENOMEM;

	/* 计算 vtime = max(vtime_now, st->vtime) */
	vtime = st->vtime;
	if (airymax_q16_lt(vtime, vtime_now))
		vtime = vtime_now;

	st->vtime = vtime;

	/* 插入 EEVDF 队列（按 vtime 升序） */
	scx_bpf_dispatch_vtime(p, SCX_DSQ_GLOBAL, SCX_SLICE_DFL, vtime, enq_flags);
	return 0;
}

/* 派发下一任务 */
void BPF_STRUCT_OPS(agent_dispatch, s32 cpu, struct task_struct *prev)
{
	/* 从全局 DSQ 取出 vtime 最小者 */
	scx_bpf_consume(SCX_DSQ_GLOBAL);
}

/* 任务运行结束 */
void BPF_STRUCT_OPS(agent_running, struct task_struct *p)
{
	struct agentrt_task_state *st;
	airymax_q16_t delta, weight, new_v;

	st = bpf_task_storage_get(&task_state, p, 0, 0);
	if (!st)
		return;

	/* delta_exec * weight / max_weight */
	delta = airymax_q16_from_int(bpf_get_cpu_usage_ns());
	weight = airymax_sched_agent_compute_weight(&st->meta);
	new_v = airymax_q16_add(st->vtime,
				airymax_q16_mul_w(delta, weight));
	st->vtime = airymax_sched_agent_ema(st->vtime, new_v);
}

/* 注册 struct_ops */
SEC(".struct_ops.link")
struct sched_ext_ops agent_ops = {
	.enqueue     = (void *)agent_enqueue,
	.dispatch    = (void *)agent_dispatch,
	.running     = (void *)agent_running,
	.name        = "agent_sched",
	.timeout_ms  = 5000U,
};

char _license[] SEC("license") = "GPL";
```

### 5.2 用户态调度器加载

```bash
# 加载 SCHED_AGENT BPF 调度器
scx_loader --name agent_sched \
    --bpf /usr/lib/airymaxos/sched_ext/sched_agent_bpf.o \
    --timeout-ms 5000 \
    --log-level info

# 验证调度器状态
cat /sys/kernel/sched_ext/state
# 输出：enabled
#       ops=agent_sched
#       enabled_at=1468923400000000000

# 查看 SCHED_AGENT 任务列表
cat /sys/kernel/sched_ext/agent_tasks
```

---

## 6. 性能基准测试方法

### 6.1 基准测试矩阵

| 基准 | 工具 | 测量维度 | 目标值 |
|------|------|----------|--------|
| 调度延迟 | `perf sched latency` | enqueue→dispatch | P99 ≤ 50μs |
| 抖动 | `cyclictest -p 95 -m -t1` | 最大延迟 | P99 ≤ 200μs |
| 上下文切换 | `perf bench sched messaging` | ops/sec | ≥ 默认 CFS 110% |
| Token 吞吐 | `agentrt-bench token-stream` | Token/s | ≥ CFS 130% |
| 能效 | `perf stat -e power/energy-pkg/` | Token/Watt | ≥ CFS 130% |

### 6.2 自动化基准测试脚本

```bash
#!/bin/bash
# /usr/lib/airymaxos/tests/sched_agent_bench.sh
# SCHED_AGENT 性能基准自动化测试

set -euo pipefail
RESULT_DIR=${1:-/var/tmp/agentrt-bench}
mkdir -p "$RESULT_DIR"

echo "[1/5] 加载 SCHED_AGENT BPF 调度器"
scx_loader --name agent_sched --bpf \
    /usr/lib/airymaxos/sched_ext/sched_agent_bpf.o || exit 1

echo "[2/5] 调度延迟基准（30 秒采样）"
perf sched record -F 99 -a -- sleep 30 2>/dev/null
perf sched latency -i perf.data > "$RESULT_DIR/sched_latency.txt"

echo "[3/5] 抖动测试（cyclictest，60 秒）"
cyclictest -p 95 -m -t1 -i 200 -d 0 -a 0 -D 60m 2>&1 | \
    tee "$RESULT_DIR/cyclictest.txt" | tail -n 5

echo "[4/5] Token 吞吐基准"
agentrt-bench token-stream --duration 60 --concurrency 8 \
    > "$RESULT_DIR/token_throughput.txt"

echo "[5/5] 能效测量（RAPL）"
perf stat -e power/energy-pkg/,power/energy-ram/ \
    agentrt-bench token-stream --duration 60 2>&1 | \
    tee "$RESULT_DIR/energy.txt"

echo "[完成] 结果保存于 $RESULT_DIR"
echo "SCHED_AGENT 性能报告："
awk '/P99/ {print "  "$0}' "$RESULT_DIR/cyclictest.txt"
grep -E 'tokens/sec' "$RESULT_DIR/token_throughput.txt"
```

### 6.3 性能指标采集与上报

通过 user_events 向 agentrt-liunx 可观测性子系统（monit_d）上报指标：

```c
/* 用户态调度指标上报示例 */
#include <linux/user_events.h>

struct sched_agent_metric {
	__u64 ts_ns;
	__u32 cpu;
	__u32 pid;
	__u64 vtime;
	__u32 weight;
	__u32 token_budget;
	__u8  stage;
	__u8  reserved[3];
};

static int emit_sched_metric(const struct sched_agent_metric *m)
{
	struct user_event_data data;
	int ret;

	data.payload = m;
	data.payload_len = sizeof(*m);
	ret = user_event_emit("sched_agent_metric", &data);
	if (ret < 0)
		goto out_err;
	return 0;

out_err:
	pr_warn("agentrt-liunx: emit sched_metric failed ret=%d\n", ret);
	return ret;
}
```

### 6.4 性能指标对照表

| 指标 | CFS 基线 | SCHED_AGENT 目标 | 提升幅度 |
|------|----------|-----------------|----------|
| 调度延迟 P99 | 120μs | 50μs | -58% |
| Agent 任务 P99 响应 | 280ms | 100ms | -64% |
| Token 吞吐（8 并发） | 12k Token/s | 16k Token/s | +33% |
| Token/Watt | 1450 | 1900 | +31% |
| 抖动 P99 | 480μs | 200μs | -58% |

---

## 7. 调度器稳定性与降级策略

### 7.1 调度器心跳与看门狗

SCHED_AGENT 通过 sched_ext 框架内置的心跳看门狗机制，若 BPF 程序在 `timeout_ms`（默认 5000ms）内未能派发任务，框架自动降级到默认 SCX 调度器：

```c
/* 看门狗触发后的降级回调 */
void BPF_STRUCT_OPS(agent_exit, struct scx_exit_info *info)
{
	if (info->kind == SCX_EXIT_UNREG_BPF)
		bpf_printk("agentrt-liunx: SCHED_AGENT BPF error %d\n",
			   info->exit_code);
	else if (info->kind == SCX_EXIT_WATCHDOG)
		bpf_printk("agentrt-liunx: watchdog timeout, fallback to default\n");
}
```

### 7.2 错误码体系对接

调度器错误码纳入 agentrt-liunx 统一错误码体系（`include/airymax/error.h`，IRON-9 v2 [SC] 共享契约层）：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AGENTRT_E_SCHED_TIMEOUT | -210 | 调度超时 |
| AGENTRT_E_SCHED_BPF_LOAD | -211 | BPF 加载失败 |
| AGENTRT_E_SCHED_VTIME_OVF | -212 | vtime 溢出 |
| AGENTRT_E_SCHED_NO_AFFINITY | -213 | 无可用 CPU 亲和性 |
| AGENTRT_E_SCHED_TOKEN_BUDGET | -214 | Token 预算耗尽 |

集中错误处理示例（K&R 风格 + `goto out_free_xxx`）：

```c
int agentrt_sched_agent_init(struct agentrt_sched_config *cfg)
{
	struct bpf_object *obj = NULL;
	struct bpf_link *link = NULL;
	int ret;

	obj = bpf_object__open_file("/usr/lib/airymaxos/sched_ext/sched_agent_bpf.o", NULL);
	if (libbpf_get_error(obj)) {
		ret = AGENTRT_E_SCHED_BPF_LOAD;
		goto out_free_obj;
	}

	ret = bpf_object__load(obj);
	if (ret < 0) {
		ret = AGENTRT_E_SCHED_BPF_LOAD;
		goto out_free_obj;
	}

	link = bpf_map__attach_struct_ops(bpf_object__find_map_by_name(obj, "agent_ops"));
	if (!link) {
		ret = AGENTRT_E_SCHED_BPF_LOAD;
		goto out_free_obj;
	}
	return 0;

out_free_obj:
	if (link)
		bpf_link__destroy(link);
	if (obj)
		bpf_object__close(obj);
	return ret;
}
```

---

## 8. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **S-1 反馈闭环** | vtime EMA + Token 预算反馈形成调度闭环 |
| **K-4 可插拔策略** | BPF struct_ops 允许运行时替换调度策略 |
| **E-2 可观测性** | user_events 全量上报调度指标 |
| **E-3 资源确定性** | CPU 亲和性 + 独占核保证 LLM 调度确定性 |
| **E-8 可测试性** | 自动化基准测试套件覆盖 P99 指标 |
| **A-4 完美主义** | Q16.16 定点数精确表达，禁用浮点 |

---

## 9. IRON-9 v2 同源映射

| 组件 | agentrt-liunx（[SS]） | agentrt（[SS]） | 共享（[SC]） |
|------|------------------------|------------------|--------------|
| 调度语义 | SCHED_AGENT（sched_ext） | MicroCoreRT 用户态调度 | 调度语义 API 签名 |
| vtime 公式 | Q16.16 内核态 | Q16.16 用户态 | `airymax_q16_t` 头文件 |
| 权重因子 | stage + token | stage + token | 权重枚举定义 |
| 错误码 | AGENTRT_E_SCHED_* | AGENTRT_E_SCHED_* | `error.h` 错误码段 |

---

## 10. 相关文档

- `170-performance/README.md`（性能工程主索引）
- `170-performance/02-memory-performance.md`（内存性能设计）
- `170-performance/03-ipc-performance.md`（IPC 性能设计）
- `20-modules/03-scheduling.md`（调度子系统设计）
- `10-architecture/03-microkernel-strategy.md`（微内核化改造策略）
- `50-engineering-standards/01-coding-standards.md`（编码规范）

---

## 11. 参考材料

- Linux 6.6 `kernel/sched/`（EEVDF 调度器）
- Linux 6.6 `tools/sched_ext/`（sched_ext 工具）
- `Documentation/scheduler/sched-ext.rst`
- seL4 微内核调度设计文档
- EEVDF 论文（Earliest Eligible Virtual Deadline First）
- agentrt MicroCoreRT 调度语义规范

---

> **文档结束** | agentrt-liunx（AirymaxOS）调度性能工程设计 | 1.0.1 开发版本
