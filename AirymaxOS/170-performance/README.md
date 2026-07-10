Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）性能工程设计

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）性能工程体系主索引\
> **版本**：0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**：2026-07-06\
> **优先级**：P1（0.1.1 完成 6 文档，1.0.1 实施验证）\
> **同源映射**：agentrt 性能基线 + Linux 6.6 性能子系统（perf / sched_ext / MGLRU / io_uring）\
> **理论根基**：Linux 内核性能工程 + Airymax S-1 反馈闭环 + A-4 完美主义

---

## 1. 模块定位

agentrt-linux 性能工程体系是系统在高负载场景下稳定运行的工程保障。它继承 Linux 内核 30+ 年沉淀的性能工程哲学（perf 剖析 + sched_ext 可插拔调度 + MGLRU 内存回收 + io_uring 零拷贝 I/O），并在其上扩展智能体操作系统专属的 Token 能效工程、Agent 延迟 SLO、记忆卷载性能等。

性能工程不是「调优」的代名词，而是「可测量、可预测、可保障」的工程体系。agentrt-linux 性能工程的核心理念是：**一切性能指标必须可测量，一切性能回归必须可追溯，一切性能 SLO 必须可保障**。这要求从设计阶段就将性能纳入考量，而非在开发末期才进行「调优」。

### 1.1 性能工程分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | CPU 性能 | perf + sched_ext + EEVDF | CPU 调度 |
| L2 | 内存性能 | MGLRU 多代 LRU + CXL + PMEM | 内存管理 |
| L3 | I/O 性能 | io_uring + zero-copy | I/O 子系统 |
| L4 | 网络性能 | AF_XDP + DPDK | 网络栈 |
| L5 | 锁与并发 | lockdep + KCSAN | 并发正确性 |
| **L6** | **Token 能效** | **agentrt-linux 专属** | **Agent Token 消耗** |
| **L7** | **Agent 延迟 SLO** | **agentrt-linux 专属** | **Agent 响应延迟** |

### 1.2 agentrt-linux 扩展

- **Token 能效工程**：Token / Watt / Token / Latency 三大能效指标
- **Agent 延迟 SLO**：认知延迟 / 规划延迟 / 执行延迟三级 SLO
- **记忆卷载性能**：L1→L2→L3→L4 四层记忆的访问延迟与命中率
- **CoreLoopThree 性能**：三层认知循环的吞吐量与延迟

---

## 2. 核心性能机制

### 2.1 perf 性能剖析

```bash
# CPU 性能剖析
perf record -F 99 -a -g -- sleep 30
perf report

# 软件事件
perf stat -e context-switches,scheduler:events

# 硬件事件
perf stat -e cycles,instructions,cache-misses
```

### 2.2 sched_ext 可插拔调度

```c
// SCHED_AGENT 策略（eBPF 程序）
SEC("struct_ops/agent_enqueue")
int BPF_PROG(agent_enqueue, struct task_struct *p, u64 enq_flags) {
    // Agent 任务入队，按 Token 预算排序
    return 0;
}
```

### 2.3 MGLRU 多代 LRU 内存回收

```bash
# 查看 MGLRU 状态
cat /sys/kernel/mm/lru_gen/enabled

# 多代 LRU 参数
echo "y" > /sys/kernel/mm/lru_gen/enabled
echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms
```

### 2.4 io_uring 零拷贝 I/O

```c
// io_uring 提交队列
struct io_uring ring;
io_uring_queue_init(32, &ring, 0);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, len, 0);
io_uring_submit(&ring);
```

---

## 3. 文档索引

```
170-performance/
├── README.md                       # 本文件
├── 01-scheduling-performance.md   # 调度性能（sched_ext + EEVDF）✅
├── 02-memory-performance.md       # 内存性能（MGLRU 多代 LRU + CXL）✅
├── 03-ipc-performance.md          # IPC 性能（AgentsIPC + io_uring）✅
├── 04-token-efficiency.md         # Token 能效工程 ✅
├── 05-agent-latency-slo.md       # Agent 延迟 SLO ✅
└── 06-benchmark-suite.md          # 基准测试套件 ✅
```

### 3.1 0.1.1 版本范围

完成 README + 01-scheduling-performance.md（sched_ext 可插拔调度 + EEVDF + vtime 衰减）+ 02-memory-performance.md（MGLRU 多代 LRU + CXL 内存分层 + PMEM DAX）+ 03-ipc-performance.md（AgentsIPC 零拷贝 + io_uring 批量提交）+ 04-token-efficiency.md（Token/Watt + Token/Latency + Token/Dollar 三大能效指标 + RAPL 功耗测量 + LLM 推理阶段能效 + 能效回归检测）+ 05-agent-latency-slo.md（L1/L2/L3 三级延迟 SLO + SLO 保障机制 + 违约处理 + 延迟预算管理）+ 06-benchmark-suite.md（4 层分层体系 L1 微基准/L2 子系统/L3 端到端/L4 压力 + 回归检测 + CI/CD 集成）。

### 3.2 1.0.1 版本范围

170 模块 6/6 文档全部完成（100%），1.0.1 阶段实施性能工程标准与生产就绪验证。

---

## 4. agentrt-linux 专属扩展

### 4.1 Token 能效工程

三大能效指标：

| 指标 | 含义 | 测量 |
|------|------|------|
| Token / Watt | 每 Watt 功耗处理的 Token 数 | perf + 功耗监控 |
| Token / Latency | 每单位延迟处理的 Token 数 | ftrace + AgentsIPC trace |
| Token / Dollar | 每美元成本处理的 Token 数 | 成本核算 |

### 4.2 Agent 延迟 SLO

三级延迟 SLO：

| 级别 | 延迟阈值 | 适用场景 |
|------|---------|---------|
| L1 认知延迟 | ≤ 100ms | 感知 → 理解 |
| L2 规划延迟 | ≤ 1s | 理解 → 决策 |
| L3 执行延迟 | ≤ 10s | 决策 → 执行 |

### 4.3 记忆卷载性能

```bash
# 查看记忆卷载性能
cat /sys/kernel/agentrt/memory_rovol/perf

# 输出示例
layer    hit_rate  avg_latency  eviction_rate
L1_raw   0.85      0.1ms        0.05
L2_feat  0.72      1ms          0.10
L3_str   0.91      10ms         0.02
L4_pat   0.95      100ms        0.01
```

### 4.4 CoreLoopThree 性能

三层认知循环的吞吐量与延迟：

| 层级 | 吞吐量 | 延迟 | 责任 |
|------|--------|------|------|
| L1 感知循环 | ≥ 1000 ops/s | ≤ 10ms | 数据采集 |
| L2 认知循环 | ≥ 100 ops/s | ≤ 100ms | 推理决策 |
| L3 行为循环 | ≥ 10 ops/s | ≤ 1s | 行动执行 |

### 4.5 同源 agentrt 性能

agentrt 的性能基线与 agentrt-linux 同源：
- agentrt 用户态：`agentrt_metrics_record()` 性能指标采集
- agentrt-linux 内核态：`perf_event` + ftrace 性能剖析
- 两端通过 user_events 桥接性能数据

### 4.6 IRON-9 v2 同源且部分代码共享

性能工程遵循 IRON-9 原则：
- agentrt 性能基线（用户态运行时性能）
- agentrt-linux 性能工程（OS 级 + Agent 专属性能）
- 两端独立演进，但通过同源指标保持互操作

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **S-1 反馈闭环** | 性能监控 → 调度策略反馈 |
| **A-4 完美主义** | 性能 SLO 100% 保障 |
| **E-2 可观测性** | 全栈性能可观测 |
| **E-8 可测试性** | 性能基准测试套件 |
| **IRON-9 v2 同源且部分代码共享** | 与 agentrt 性能基线同源 |

---

## 6. 相关文档

- `90-observability/README.md`（可观测性）
- `100-operations/README.md`（运维体系）
- `20-modules/04-memory.md`（memory 子仓设计）
- `20-modules/05-cognition.md`（cognition 子仓设计）
- `80-testing/README.md`（性能测试）

---

## 7. 参考材料

- Linux 6.6 `tools/perf/`（perf 工具）
- Linux 6.6 `kernel/sched/`（调度子系统）
- Linux 6.6 `mm/`（内存管理）
- Linux 6.6 `io_uring/`（io_uring 子系统）
- sched_ext 文档（`Documentation/scheduler/sched-ext.rst`）

---

> **文档结束** | 170 模块 6/6 文档全部完成（100%）
