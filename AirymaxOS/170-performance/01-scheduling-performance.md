Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）调度性能工程设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）调度子系统性能工程详细设计\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-09\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-linux（AirymaxOS）调度性能工程旨在为智能体工作负载提供确定性可预测的 CPU 调度能力。本设计聚焦于三大目标：

1. **Agent 延迟 SLO 100% 保障**：L1 认知延迟 ≤ 100ms、L2 规划延迟 ≤ 1s、L3 执行延迟 ≤ 10s，在 99 分位（P99）下达成
2. **Token 能效最大化**：每瓦功耗处理的 Token 数（Token/Watt）较默认 CFS 调度器提升 ≥ 30%
3. **内核态禁用浮点约束下的高精度 vtime 计算**：采用 Q16.16 定点数表达虚拟时间，与 Linux 6.6 内核 EEVDF 调度器语义对齐

### 1.2 适用范围

- 基于 Linux 6.6 标准内核原生 SCHED_DEADLINE/SCHED_FIFO/EEVDF 的方案 C-Prime 用户态调度器（替代 sched_ext，6.6 主线不含 sched_ext）
- AIRY_SCHED_AGENT 策略（用户态调度器策略名，IRON-9 v2 [SS] 语义同源层）
- Agent kthread 与用户态 daemon 双向调度协同
- 与 agentrt（AirymaxAgentRT）的 MicroCoreRT 调度语义保持一致

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。禁止使用"微内核原语"描述 agentrt 的能力，正确表述为"微核心原语"。

### 1.4 性能分层与 SLO 指标

| 层级 | 指标 | 阈值 | 测量方式 |
|------|------|------|----------|
| L1 | 调度延迟（enqueue → dispatch） | ≤ 50μs | perf trace + ftrace |
| L2 | 上下文切换开销 | ≤ 2μs（同核）/ ≤ 8μs（跨核） | perf sched |
| L3 | Agent 任务 P99 响应延迟 | ≤ 100ms | user_events trace |
| L4 | Token/Watt 能效 | 较 CFS 提升 ≥ 30% | perf + RAPL 功耗 |
| L5 | 调度器抖动（jitter） | P99 ≤ 200μs | cyclictest + HPET |

---

## 2. 方案 C-Prime 用户态调度器架构

### 2.1 方案 C-Prime 在 Linux 6.6 的部署

标准 Linux 6.6 主线不包含 sched_ext（6.12 才合入主线）。agentrt-linux 不向前移植 sched_ext，而是采用方案 C-Prime（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射）作为用户态调度器，复用 Linux 6.6 原生调度类。启用配置如下：

```kconfig
# kernel/.config（节选）
CONFIG_DEADLINE_GROUP_SCHED=y
CONFIG_RT_GROUP_SCHED=y
CONFIG_CGROUPS=y
CONFIG_CGROUP_SCHED=y
CONFIG_CGROUP_CPUSET=y
CONFIG_SCHED_AUTOGROUP=y
# 方案 C-Prime：基于原生 SCHED_DEADLINE/SCHED_FIFO/EEVDF + cgroup v2
# 禁止 CONFIG_SCHED_CLASS_EXT / CONFIG_BPF_SCHED（6.6 主线无 sched_ext）
```

### 2.2 AIRY_SCHED_AGENT 策略分层

AIRY_SCHED_AGENT 策略（用户态调度器策略名）通过以下原生调度类分层实现：

```
┌────────────────────────────────────────────┐
│  SCHED_FIFO / SCHED_RR（实时类，prio 0-99）│  ← Agent 关键路径（t2 主思考）
├────────────────────────────────────────────┤
│  SCHED_DEADLINE（DEADLINE 类，CBS 带宽）   │  ← Agent 常量带宽（认知 kthread）
├────────────────────────────────────────────┤
│  SCHED_NORMAL / SCHED_BATCH（CFS + EEVDF）│  ← 默认负载（LLM 推理 daemon）
├────────────────────────────────────────────┤
│  SCHED_IDLE（fallback）                    │  ← 空闲与兜底
└────────────────────────────────────────────┘
```

AIRY_SCHED_AGENT 通过 SCHED_FIFO/SCHED_DEADLINE + cgroup cpuset 隔离实现 Agent 工作负载优先级：关键认知路径走 SCHED_FIFO（高 prio），常量带宽任务走 SCHED_DEADLINE（CBS），LLM 推理等吞吐型任务走 SCHED_NORMAL（EEVDF）+ cpuset 独占核。**禁止**定义 `SCHED_AGENT` 内核调度类宏（避免与内核调度类编号冲突）。

### 2.3 用户态调度器策略钩子

AIRY_SCHED_AGENT 用户态调度器通过 `struct airy_sched_ops` 暴露下列回调（非 BPF struct_ops，纯用户态函数表）：

| 钩子 | 触发时机 | 职责 |
|------|----------|------|
| `agent_init` | 调度器加载 | 初始化 cgroup、cpuset、带宽配额 |
| `agent_exit` | 调度器卸载 | 释放资源、刷新统计 |
| `agent_enqueue` | 任务入队 | 计算 vtime、设置 sched_attr |
| `agent_dequeue` | 任务出队 | 清理状态、更新预算 |
| `agent_dispatch` | 选下一个任务 | 按 vtime 最小者派发到 CPU |
| `agent_runnable` | 任务可运行 | 唤醒 kthread / 设置 SCHED_FIFO |
| `agent_stopping` | 任务停止 | 更新 vtime、统计 |
| `agent_quiescent` | 任务静默 | 资源回收 |

---

## 3. vtime 衰减公式设计

### 3.1 Q16.16 定点数表示

Linux 6.6 内核态禁用浮点（kernel_fpu 禁用），因此 vtime 必须以定点数表达。agentrt-linux 采用 **Q16.16 定点数**（`airy_q16_t`），整数部分 16 位、小数部分 16 位，定义于 `include/airymax/memory_types.h`（IRON-9 v2 [SC] 共享契约层）：

```c
/* include/airymax/airy_q16.h —— Q16.16 定点数原语 */
#ifndef AIRY_Q16_H
#define AIRY_Q16_H

#include <linux/types.h>

typedef __s64 airy_q16_t;       /* Q16.16 定点数 */
typedef __u32 airy_weight_t;    /* 0-65535 权重 */

#define AIRY_Q16_SHIFT       16
#define AIRY_Q16_ONE         (((airy_q16_t)1) << AIRY_Q16_SHIFT)
#define AIRY_Q16_HALF        (AIRY_Q16_ONE >> 1)

/* 整数 ↔ Q16.16 */
static inline airy_q16_t airy_q16_from_int(__s64 v)
{
	return (airy_q16_t)v << AIRY_Q16_SHIFT;
}

static inline __s64 airy_q16_to_int(airy_q16_t v)
{
	return (__s64)(v >> AIRY_Q16_SHIFT);
}

/* Q16.16 ↔ u32 权重（0-65535 映射到 0.0-1.0） */
static inline airy_q16_t airy_q16_from_weight(airy_weight_t w)
{
	return (airy_q16_t)w << (AIRY_Q16_SHIFT - 16 + 16);
}

/* 乘法：a * b（其中一个必须是权重，避免溢出） */
static inline airy_q16_t airy_q16_mul_w(airy_q16_t a, airy_weight_t w)
{
	return (airy_q16_t)((__s128)a * w >> 16);
}

/* 加法 / 减法 / 比较 */
static inline airy_q16_t airy_q16_add(airy_q16_t a, airy_q16_t b)
{
	return a + b;
}

static inline airy_q16_t airy_q16_sub(airy_q16_t a, airy_q16_t b)
{
	return a - b;
}

static inline bool airy_q16_lt(airy_q16_t a, airy_q16_t b)
{
	return a < b;
}

#endif /* AIRY_Q16_H */
```

### 3.2 vtime 衰减公式

借鉴 EEVDF（Earliest Eligible Virtual Deadline First），AIRY_SCHED_AGENT 的 vtime 衰减公式如下（全部以 Q16.16 表达）：

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
/* AIRY_SCHED_AGENT EMA vtime 平滑 */
static inline airy_q16_t airy_sched_agent_ema(airy_q16_t old_v,
						    airy_q16_t new_v)
{
	const airy_q16_t alpha = 0x4000;     /* 0.25 */
	const airy_q16_t one_minus_alpha = 0xC000; /* 0.75 */

	/* α * new_v + (1-α) * old_v，全部定点运算 */
	airy_q16_t term1 = (airy_q16_t)((__s128)new_v * alpha >> 16);
	airy_q16_t term2 = (airy_q16_t)((__s128)old_v * one_minus_alpha >> 16);
	return airy_q16_add(term1, term2);
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
static airy_weight_t airy_sched_agent_compute_weight(
	const struct airy_task_meta *meta)
{
	airy_weight_t base = meta->base_weight;
	airy_weight_t stage = meta->stage_factor;
	airy_weight_t token;
	airy_q16_t w;

	if (meta->token_budget < 20) {       /* 预算 < 20%，加速 */
		token = 0x10000;
	} else if (meta->token_budget < 50) { /* 50% 以下中等加速 */
		token = 0xC000;
	} else {
		token = 0x8000;
	}

	/* base * stage * token / 65536 / 65536 */
	w = (airy_q16_t)((__s128)base * stage >> 16);
	w = (airy_q16_t)((__s128)w * token >> 16);
	if (w > 0xFFFF)
		w = 0xFFFF;
	return (airy_weight_t)w;
}
```

---

## 4. AI 工作负载 CPU 亲和性

### 4.1 Agent kthread 与 daemon 的亲和性策略

agentrt-linux 的 12 个 daemon 与认知层 kthread 共享 CPU 资源。亲和性策略遵循以下规则：

1. **认知 kthread 绑定 NUMA-local CPU**：CoreLoopThree 的 L1/L2/L3 kthread 与对应 MemoryRovol 内存域同 NUMA 节点
2. **LLM 推理 daemon（cogn_d）独占物理核**：通过 cpuset.cpus 隔离，避免被抢占
3. **网络 daemon（net_d）绑定 RX/TX 队列所在 CPU**：与 NIC RSS 队列对齐
4. **SMT 兄弟核优先给认知任务**：双思考的 t2 主思考占用物理核，t1 快思考占用 SMT 兄弟核

### 4.2 CPU 拓扑感知的调度映射

```c
/* agentrt-linux CPU 亲和性映射表 */
struct airy_sched_agent_topo {
	int node_id;            /* NUMA 节点 */
	int die_id;            /* CPU die */
	int core_id;           /* 物理核 */
	int smt_sibling;      /* SMT 兄弟核 */
	enum airy_sched_agent_class cls;
};

/* 默认映射策略（基于 8 核 2 NUMA 示例） */
static const struct airy_sched_agent_topo default_topo[] = {
	{0, 0, 0, 4, AIRY_TOPO_COGNITION_T2},   /* CPU0,4 - 认知 t2 */
	{0, 0, 1, 5, AIRY_TOPO_COGNITION_T1},   /* CPU1,5 - 认知 t1 */
	{0, 0, 2, 6, AIRY_TOPO_LLM},            /* CPU2,6 - cogn_d */
	{0, 0, 3, 7, AIRY_TOPO_NET},            /* CPU3,7 - net_d */
	{1, 0, 0, 4, AIRY_TOPO_MEMORY},         /* NUMA1 CPU0,4 - memory_d */
	{1, 0, 1, 5, AIRY_TOPO_SECURITY},        /* NUMA1 CPU1,5 - 安全 */
	{1, 0, 2, 6, AIRY_TOPO_IPC},            /* NUMA1 CPU2,6 - IPC */
	{1, 0, 3, 7, AIRY_TOPO_GENERIC},        /* NUMA1 CPU3,7 - 通用 */
};

/* 设置 CPU 亲和性 */
static int airy_sched_agent_set_affinity(struct task_struct *p,
					    const struct airy_sched_agent_topo *t)
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
	pr_warn("agentrt-linux: set_affinity failed for pid=%d ret=%d\n",
		p->pid, ret);
	return ret;
}
```

### 4.3 cpuset 隔离配置

通过 cgroup v2 的 cpuset 子系统为关键 daemon 设置独占核：

```bash
# /etc/airymaxos/cpuset.conf —— agentrt-linux cpuset 隔离配置
# 拓扑假设：8 核 / 2 NUMA / SMT=2

# cogn_d 独占 CPU2, CPU6（物理核对）
mkdir /sys/fs/cgroup/cogn_d
echo "2,6" > /sys/fs/cgroup/cogn_d/cpuset.cpus
echo "0" > /sys/fs/cgroup/cogn_d/cpuset.mems
echo "exclusive" > /sys/fs/cgroup/cogn_d/cpu.isolated

# cognition kthread 绑定 CPU0, CPU4
mkdir /sys/fs/cgroup/cognition
echo "0,4" > /sys/fs/cognition/cpuset.cpus
echo "0" > /sys/fs/cgroup/cognition/cpuset.mems

# 默认任务使用 CPU1, CPU3, CPU5, CPU7
mkdir /sys/fs/cgroup/system
echo "1,3,5,7" > /sys/fs/cgroup/system/cpuset.cpus
```

---

## 5. 调度策略完整示例

### 5.1 AIRY_SCHED_AGENT 用户态调度器程序

以下是一个完整的 AIRY_SCHED_AGENT 用户态调度策略实现（C，K&R 风格，Tab=8），基于方案 C-Prime（SCHED_FIFO/SCHED_DEADLINE + cgroup cpuset）：

```c
/* services/sched_agent/sched_agent.c */
#include <sched.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <airymax/airy_q16.h>
#include <airymax/memory_types.h>

/* 任务状态表（用户态维护，替代 BPF task_storage） */
struct airy_task_state {
	pid_t pid;
	airy_q16_t vtime;
	struct airy_task_meta meta;
	int sched_policy;	/* SCHED_FIFO / SCHED_DEADLINE / SCHED_NORMAL */
	int sched_prio;		/* 0-99（FIFO/RR）或 DEADLINE 参数 */
};

#define MAX_AGENTS 1024
static struct airy_task_state task_table[MAX_AGENTS];

/* 全局 vtime 基线 */
static volatile airy_q16_t vtime_now = 0;

/* 任务入队：设置调度策略与优先级 */
int agent_enqueue(struct airy_task_state *st, u64 enq_flags)
{
	struct sched_param sp = { .sched_priority = st->sched_prio };
	airy_q16_t vtime;
	int ret;

	/* 计算 vtime = max(vtime_now, st->vtime) */
	vtime = st->vtime;
	if (airy_q16_lt(vtime, vtime_now))
		vtime = vtime_now;
	st->vtime = vtime;

	/* 按策略设置调度类（方案 C-Prime：SCHED_FIFO/SCHED_DEADLINE/SCHED_NORMAL） */
	ret = sched_setscheduler(st->pid, st->sched_policy, &sp);
	if (ret < 0)
		return -errno;
	return 0;
}

/* 派发下一任务：由内核 SCHED_FIFO/SCHED_DEADLINE/EEVDF 完成实际派发 */
int agent_dispatch(s32 cpu, struct airy_task_state *prev)
{
	/* 用户态仅维护 vtime 与策略映射，
	 * 实际 CPU 派发由内核原生调度类完成 */
	return 0;
}

/* 任务运行结束：更新 vtime（EMA 平滑） */
void agent_running(struct airy_task_state *st)
{
	airy_q16_t delta, weight, new_v;

	/* delta_exec * weight / max_weight */
	delta = airy_q16_from_int(airy_get_cpu_usage_ns(st->pid));
	weight = airy_sched_agent_compute_weight(&st->meta);
	new_v = airy_q16_add(st->vtime,
				airy_q16_mul_w(delta, weight));
	st->vtime = airy_sched_agent_ema(st->vtime, new_v);
}

/* 用户态策略回调表（非 BPF struct_ops，纯用户态函数表） */
struct airy_sched_ops agent_ops = {
	.enqueue     = (void *)agent_enqueue,
	.dispatch    = (void *)agent_dispatch,
	.running     = (void *)agent_running,
	.name        = "agent_sched",
	.timeout_ms  = 5000U,
};
```

### 5.2 用户态调度器加载

```bash
# 启动 AIRY_SCHED_AGENT 用户态调度器 daemon
airy-sched-daemon --name agent_sched \
    --policy /usr/lib/airymaxos/user-sched/sched_agent.so \
    --timeout-ms 5000 \
    --log-level info

# 验证调度器状态（cgroup v2 接口）
cat /sys/fs/cgroup/agentrt/sched.state
# 输出：enabled
#       ops=agent_sched
#       enabled_at=1468923400000000000

# 查看 AIRY_SCHED_AGENT 任务列表
cat /sys/fs/cgroup/agentrt/sched.agent_tasks
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
# AIRY_SCHED_AGENT 性能基准自动化测试

set -euo pipefail
RESULT_DIR=${1:-/var/tmp/agentrt-bench}
mkdir -p "$RESULT_DIR"

echo "[1/5] 启动 AIRY_SCHED_AGENT 用户态调度器"
airy-sched-daemon --name agent_sched --policy \
    /usr/lib/airymaxos/user-sched/sched_agent.so || exit 1

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
echo "AIRY_SCHED_AGENT 性能报告："
awk '/P99/ {print "  "$0}' "$RESULT_DIR/cyclictest.txt"
grep -E 'tokens/sec' "$RESULT_DIR/token_throughput.txt"
```

### 6.3 性能指标采集与上报

通过 user_events 向 agentrt-linux 可观测性子系统（audit_d）上报指标：

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
	pr_warn("agentrt-linux: emit sched_metric failed ret=%d\n", ret);
	return ret;
}
```

### 6.4 性能指标对照表

| 指标 | CFS 基线 | AIRY_SCHED_AGENT 目标 | 提升幅度 |
|------|----------|-----------------|----------|
| 调度延迟 P99 | 120μs | 50μs | -58% |
| Agent 任务 P99 响应 | 280ms | 100ms | -64% |
| Token 吞吐（8 并发） | 12k Token/s | 16k Token/s | +33% |
| Token/Watt | 1450 | 1900 | +31% |
| 抖动 P99 | 480μs | 200μs | -58% |

---

## 7. 调度器稳定性与降级策略

### 7.1 调度器心跳与看门狗

AIRY_SCHED_AGENT 用户态调度器内置心跳看门狗机制，若用户态调度器 daemon 在 `timeout_ms`（默认 5000ms）内未能完成派发，自动降级到 EEVDF（CFS）默认调度类：

```c
/* 看门狗触发后的降级回调（用户态） */
void agent_exit(struct airy_sched_exit_info *info)
{
	if (info->kind == AIRY_SCHED_EXIT_DAEMON_ERROR)
		fprintf(stderr, "agentrt-linux: AIRY_SCHED_AGENT daemon error %d\n",
			info->exit_code);
	else if (info->kind == AIRY_SCHED_EXIT_WATCHDOG)
		fprintf(stderr, "agentrt-linux: watchdog timeout, fallback to EEVDF\n");
}
```

### 7.2 错误码体系对接

调度器错误码纳入 agentrt-linux 统一错误码体系（`include/airymax/error.h`，IRON-9 v2 [SC] 共享契约层）：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AIRY_E_SCHED_TIMEOUT | -210 | 调度超时 |
| AIRY_E_SCHED_DAEMON_LOAD | -211 | 用户态调度器加载失败 |
| AIRY_E_SCHED_VTIME_OVF | -212 | vtime 溢出 |
| AIRY_E_SCHED_NO_AFFINITY | -213 | 无可用 CPU 亲和性 |
| AIRY_E_SCHED_TOKEN_BUDGET | -214 | Token 预算耗尽 |

集中错误处理示例（K&R 风格 + `goto out_free_xxx`）：

```c
int airy_sched_agent_init(struct airy_sched_config *cfg)
{
	struct airy_sched_ops *ops = NULL;
	void *policy_hdl = NULL;
	int ret;

	/* 加载用户态调度策略 .so（替代 BPF object） */
	policy_hdl = dlopen("/usr/lib/airymaxos/user-sched/sched_agent.so",
			    RTLD_NOW | RTLD_GLOBAL);
	if (!policy_hdl) {
		ret = AIRY_E_SCHED_DAEMON_LOAD;
		goto out_free;
	}

	ops = dlsym(policy_hdl, "agent_ops");
	if (!ops) {
		ret = AIRY_E_SCHED_DAEMON_LOAD;
		goto out_free;
	}

	/* 注册用户态策略回调表（替代 BPF struct_ops attach） */
	ret = airy_sched_register_ops(ops);
	if (ret < 0)
		goto out_free;
	return 0;

out_free:
	if (policy_hdl)
		dlclose(policy_hdl);
	return ret;
}
```

---

## 8. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **S-1 反馈闭环** | vtime EMA + Token 预算反馈形成调度闭环 |
| **K-4 可插拔策略** | 方案 C-Prime 允许运行时替换用户态调度策略 |
| **E-2 可观测性** | user_events 全量上报调度指标 |
| **E-3 资源确定性** | CPU 亲和性 + 独占核保证 LLM 调度确定性 |
| **E-8 可测试性** | 自动化基准测试套件覆盖 P99 指标 |
| **A-4 完美主义** | Q16.16 定点数精确表达，禁用浮点 |

---

## 9. IRON-9 v2 同源映射

| 组件 | agentrt-linux（[SS]） | agentrt（[SS]） | 共享（[SC]） |
|------|------------------------|------------------|--------------|
| 调度语义 | AIRY_SCHED_AGENT（方案 C-Prime） | MicroCoreRT 用户态调度 | 调度语义（概念同源，签名独立演进） |
| vtime 公式 | Q16.16 内核态 | Q16.16 用户态 | `airy_q16_t` 头文件 |
| 权重因子 | stage + token | stage + token | 权重枚举定义 |
| 错误码 | AIRY_E_SCHED_* | AIRY_E_SCHED_* | `error.h` 错误码段 |

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
- Linux 6.6 `kernel/sched/deadline.c`（SCHED_DEADLINE 调度类）
- Linux 6.6 `kernel/sched/rt.c`（SCHED_FIFO/SCHED_RR 调度类）
- `Documentation/scheduler/sched-deadline.rst`（SCHED_DEADLINE 文档）
- seL4 微内核调度设计文档
- EEVDF 论文（Earliest Eligible Virtual Deadline First）
- agentrt MicroCoreRT 调度语义规范

---

> **文档结束** | agentrt-linux（AirymaxOS）调度性能工程设计 
