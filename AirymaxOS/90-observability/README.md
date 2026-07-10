Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）可观测性设计

> **文档定位**： agentrt-linux（AirymaxOS）可观测性工程体系主索引
> **版本**： 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**： 2026-07-06
> **同源映射**： agentrt E-2 可观测性原则 + Linux 6.6 ftrace/eBPF/perf
> **理论根基**： Linux 内核可观测性 + Airymax E-2 可观测性 + C-3 记忆卷载

---

## 1. 模块定位

agentrt-linux 可观测性体系是系统运行状态可见性的核心保障。它继承 Linux 内核 30+ 年沉淀的多层可观测性哲学（ftrace + eBPF + perf + 4 层文件系统接口），并在其上扩展智能体操作系统专属的 Token 能效可观测性、Agent 行为追踪、记忆卷载监控等。

### 1.1 可观测性分层

| 层级 | 类型 | 工具 | 用途 |
|------|------|------|------|
| L1 | 内核跟踪 | ftrace（tracefs + ring buffer） | 函数级跟踪 |
| L2 | 可编程探针 | eBPF（kfunc + dynamic pointer） | 动态扩展 |
| L3 | 性能分析 | perf（分析前端） | 性能剖析 |
| L4 | 文件系统接口 | sysfs/procfs/debugfs/tracefs | 静态信息导出 |
| L5 | 用户态桥接 | user_events | 用户态事件上报 |
| **L6** | **Token 能效监控** | **agentrt-linux 专属** | **Agent Token 消耗追踪** |
| **L7** | **Agent 行为追踪** | **agentrt-linux 专属** | **Agent 决策路径审计** |
| **L8** | **记忆卷载监控** | **agentrt-linux 专属** | **L1→L2→L3→L4 记忆演化** |

### 1.2 agentrt-linux 扩展

- **Token 能效仪表盘**：实时监控每个 Agent 的 Token 消耗与能效比
- **Agent 行为追踪**：通过 eBPF 探针追踪 Agent 决策路径（认知→规划→调度→执行）
- **记忆卷载监控**：监控 L1（原始卷）→L2（特征层）→L3（结构层）→L4（模式层）的记忆演化

---

## 2. 核心可观测性机制

### 2.1 ftrace（tracefs + ring buffer）

ftrace 是 Linux 内核官方跟踪系统：
```bash
# 启用函数跟踪
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 查看跟踪结果
cat /sys/kernel/debug/tracing/trace
```

支持函数跟踪、函数图跟踪、事件跟踪、硬件分支跟踪等多种模式。

### 2.2 eBPF（可编程探针）

eBPF 提供可编程的内核扩展能力：
- **kfunc 机制**：内核函数可直接被 BPF 程序调用
- **BTF（BPF Type Format）**：保证类型安全
- **dynamic pointer**：动态指针提供安全的内存访问
- **验证器**：加载时验证确保安全性

```c
// eBPF 程序示例（伪代码）
SEC("kprobe/agentrt_cognition_process")
int trace_cognition(struct pt_regs *ctx) {
    // 追踪 Agent 认知处理
    bpf_trace_printk("cognition process called\\n");
    return 0;
}
```

### 2.3 perf（分析前端）

perf 是性能分析前端工具：
```bash
# CPU 性能剖析
perf record -F 99 -a -g -- sleep 30
perf report

# 软件事件
perf stat -e context-switches,scheduler:events

# 硬件事件
perf stat -e cycles,instructions,cache-misses
```

### 2.4 4 层文件系统接口

| 文件系统 | 挂载点 | 用途 |
|---------|--------|------|
| sysfs | `/sys` | 设备/驱动/内核子系统静态信息 |
| procfs | `/proc` | 进程信息 + 内核统计 |
| debugfs | `/sys/kernel/debug` | 调试信息（仅 root） |
| tracefs | `/sys/kernel/tracing` | ftrace 跟踪控制 |

### 2.5 user_events（用户态桥接）

user_events 允许用户态程序上报事件到 ftrace ring buffer，实现用户态与内核态统一跟踪。

---

## 3. 文档索引

```
90-observability/
├── README.md                       # 本文件
├── 01-ftrace-framework.md           # ftrace 框架详解
├── 02-ebpf-probes.md               # eBPF 可编程探针
├── 03-perf-analysis.md             # perf 性能分析
├── 04-sysfs-procfs.md              # sysfs/procfs 接口
├── 05-debugfs-tracefs.md           # debugfs/tracefs 接口
├── 06-user-events.md               # user_events 用户态桥接
├── 07-token-efficiency.md          # agentrt-linux 专属：Token 能效监控
├── 08-agent-tracing.md             # agentrt-linux 专属：Agent 行为追踪
└── 09-memory-monitoring.md         # agentrt-linux 专属：记忆卷载监控
```

### 3.1 0.1.1 版本范围

0.1.1 完成 README + 01 + 02 文档（3 文档奠基），确立可观测性设计框架与 ftrace/eBPF 核心机制。其余 7 文档（03-perf-analysis 至 09-memory-monitoring）在 1.0.1 版本完成。

### 3.2 1.0.1 版本范围

完成剩余 7 文档（03-perf-analysis 至 09-memory-monitoring），实施可观测性工程标准，进行代码级验证。

---

## 4. agentrt-linux 专属扩展

### 4.1 Token 能效监控

```bash
# 查看每个 Agent 的 Token 消耗
cat /sys/kernel/agentrt/token_usage

# 输出示例
agent_id  token_input  token_output  token_total  efficiency
0x0001    1234         567           1801         0.31
0x0002    2345         678           3023         0.22
```

### 4.2 Agent 行为追踪

通过 eBPF 探针追踪 Agent 决策路径：
- `agentrt_cognition_process` 入口/出口
- `agentrt_planner_dag_update` 节点扩展
- `agentrt_scheduler_dispatch` 调度决策
- `agentrt_execution_unit_run` 执行结果

### 4.3 记忆卷载监控

```bash
# 查看记忆卷载演化
cat /sys/kernel/agentrt/memory_rovol/status

# 输出示例
layer    entries  hit_rate  avg_age  eviction_rate
L1_raw   12345    0.85      60s      0.05
L2_feat  6789     0.72      300s     0.10
L3_str   2345     0.91      1800s    0.02
L4_pat   456      0.95      7200s    0.01
```

### 4.4 同源 agentrt 可观测性

agentrt 的 `commons/` 模块（error/logger/metrics/trace/cost）与 agentrt-linux 可观测性同源：
- agentrt 用户态：`agentrt_log_write()` / `agentrt_metrics_record()`
- agentrt-linux 内核态：`printk` / `trace_printk` / `perf_event`
- 两端通过 user_events 桥接

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **E-2 可观测性** | 多层可观测性体系 |
| **C-3 记忆卷载** | L1→L2→L3→L4 记忆监控 |
| **S-1 反馈闭环** | 监控数据反馈到调度策略 |
| **K-4 可插拔策略** | eBPF 探针可动态加载 |
| **A-4 完美主义** | 全栈可观测性 |

---

## 6. 相关文档

- `50-engineering-standards/06-toolchain-and-automation.md`（CI 中的可观测性）
- `100-operations/README.md`（运维体系）
- `170-performance/README.md`（性能工程）
- `20-modules/01-kernel.md`（kernel 子仓可观测性）

---

## 7. 参考材料

- Linux 6.6 `kernel/trace/`（ftrace 实现）
- Linux 6.6 `kernel/bpf/`（eBPF 实现）
- Linux 6.6 `tools/perf/`（perf 工具）
- Linux 6.6 `Documentation/trace/`（跟踪文档）
- Linux 6.6 `Documentation/dev-tools/`（开发工具）

---

> **文档结束** | 0.1.1 P0 优先完成 README + 01 + 02
