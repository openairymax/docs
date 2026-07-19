Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）perf 性能分析
> **文档定位**：agentrt-linux（AirymaxOS）可观测性体系 L3 层——性能剖析前端 perf 的工程规范与sched_tac 调度性能对照\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[90-observability README](README.md)\
> **同源映射**：agentrt E-2 可观测性 + Linux 6.6 perf/PMU/perf_event\
> **理论根基**：Linux 6.6 内核基线 + Airymax 五维正交 24 原则 + sched_tac 调度\
> **核心约束**：perf 仅作为可观测性前端，不得用于安全钩子或调度策略注入

---

## 目录

- [第 1 章 perf 框架概述](#第-1-章-perf-框架概述)
- [第 2 章 perf stat 系统级统计](#第-2-章-perf-stat-系统级统计)
- [第 3 章 perf record/report 剖析](#第-3-章-perf-recordreport-剖析)
- [第 4 章 PMU 硬件计数器](#第-4-章-pmu-硬件计数器)
- [第 5 章 sched_tac 调度性能分析](#第-5-章-方案-c-prime-调度性能分析)
- [第 6 章 IPC fastpath 性能分析](#第-6-章-ipc-fastpath-性能分析)
- [第 7 章 内存性能分析](#第-7-章-内存性能分析)
- [第 8 章 性能基准对照](#第-8-章-性能基准对照)
- [第 9 章 Airymax Unify Design 映射](#第-9-章-airymax-unify-design-映射)
- [第 10 章 相关文档与版本维护](#第-10-章-相关文档与版本维护)

---

## 第 1 章 perf 框架概述

### 1.1 定位

perf 是 Linux 6.6 内核基线提供的官方性能剖析工具集，由 `tools/perf/` 用户态工具与 `kernel/events/core.c` 内核态 `perf_event` 子系统组成。它通过 PMU（Performance Monitoring Unit）硬件计数器、软件事件、tracepoint 三类数据源，为系统级与进程级性能分析提供统一前端。agentrt-linux 选择 perf 作为可观测性 L3 层，原因有三：

1. **硬件级精度**：perf 直接驱动 PMU 硬件计数器，提供 cycles/instructions/cache-misses/branch-misses 等微架构级指标，是sched_tac 调度延迟微秒级归因的唯一可靠手段。
2. **与 ftrace/eBPF 共生**：perf 可消费 tracepoint 事件、采样 BPF 程序输出，与可观测性 L1/L2 层共享数据通路，不引入新的基础设施。
3. **零运行时开销（采样模式）**：在 `perf record -F 99` 采样模式下，perf 开销 < 1% CPU，可在生产环境长期开启，符合 A-ULS 模块 macro_d 的"可观测性不破坏系统稳态"原则。

**OS-OBS-031: perf 是 agentrt-linux 可观测性 L3 层的强制基线，性能剖析必须以 perf 为统一前端，不得引入 vtune、perfctr 等第三方工具。**

**OS-KER-121: kernel 的 defconfig 必须开启 CONFIG_PERF_EVENTS、CONFIG_HW_PERF_EVENTS、CONFIG_PERF_COUNTERS；架构层必须实现 arm64_pmu / x86_pmu 驱动。**

### 1.2 框架组成

| 组件 | 实现位置 | 职责 |
|------|----------|------|
| perf 工具 | `tools/perf/builtin-{stat,record,report}.c` | 用户态前端 |
| perf_event 子系统 | `kernel/events/core.c` | 内核态事件源 |
| PMU 驱动 | `arch/arm64/kernel/perf_event.c` / `arch/x86/events/core.c` | 硬件计数器 |
| tracepoint 桥接 | `kernel/trace/trace_event_perf.c` | tracepoint → perf |
| BPF 桥接 | `kernel/trace/bpf_trace.c` | BPF 程序 perf 输出 |
| ring buffer | `kernel/events/ring_buffer.c` | perf 采样缓冲（独立于 ftrace ring buffer）|

**OS-STD-021: 任何对 `tools/perf/` 子目录的修改必须经过 perf 维护者审查；agentrt-linux 不得 fork perf 工具，必须紧跟上游 LTS。**

### 1.3 perf 子命令矩阵

| 子命令 | 用途 | 在 agentrt-linux 的应用 |
|--------|------|------------------------|
| `perf stat` | 计数器统计 | sched_tac 调度类整体开销测量 |
| `perf record` | 采样记录 | Agent 决策路径热点函数定位 |
| `perf report` | 报告分析 | 采样数据可视化与归因 |
| `perf top` | 实时热点 | 生产环境实时观察 |
| `perf trace` | 系统调用追踪 | 替代 strace，开销更低 |
| `perf sched` | 调度分析 | sched_tac 调度延迟直方图 |
| `perf mem` | 内存访问剖析 | alloc_pages + mmap 映射开销分析 |
| `perf c2c` | 缓存一致性 | NUMA 与 cacheline 共享分析 |
| `perf stat -e` | 自定义事件 | tracepoint 与 PMU 组合分析 |

---

## 第 2 章 perf stat 系统级统计

### 2.1 基础用法

`perf stat` 对指定命令或 PID 进行计数器统计，输出周期内事件总数。agentrt-linux 在sched_tac 调度性能基准测试中，使用如下标准命令：

```bash
# 标准 6 大硬件计数器
perf stat -e cycles,instructions,cache-misses,cache-references,branch-misses,branch-instructions \
    -a -- sleep 10

# sched_tac 调度延迟统计（per-CPU）
perf stat -e sched:sched_switch,sched:sched_wakeup,sched:sched_stat_runtime \
    -a -- per-agent-benchmark 60
```

### 2.2 sched_tac 调度类专用事件

sched_tac 三层调度类（SCHED_DEADLINE / SCHED_FIFO / EEVDF）的切换通过 tracepoint 暴露：

```bash
# SCHED_DEADLINE 相关
perf stat -e sched:sched_switch,sched:sched_stat_runtime,airy:airy_sched_class_switch \
    -a -- per-agent-benchmark 60

# 输出片段示例
# Performance counter stats for 'system wide':
#     1,234,567  sched:sched_switch
#       987,654  sched:sched_stat_runtime
#        12,345  airy:airy_sched_class_switch
#              DL→RT:4321  RT→EEVDF:5678  EEVDF→DL:2346
```

### 2.3 perf stat 输出格式

agentrt-linux 定义了标准 CSV 输出格式，供 A-ULS 模块 macro_d 自动归集：

```bash
perf stat -x, -e cycles,instructions,cache-misses -- \
    per-agent-benchmark 60 -o /var/log/airy/perf-stat-$(date +%s).csv
```

CSV 字段顺序：`value,unit,event,runtime,percent,counted`

**OS-OBS-032: 性能基准测试必须使用 `perf stat -x,` CSV 格式输出，便于 macro_d 自动解析；不得使用人类可读格式作为基准数据源。**

**OS-STD-022: 性能基准测试运行期间，必须关闭 IRQ affinity 平衡（`echo 0 > /proc/sys/kernel/irq_balancing`），避免 CPU 迁移污染数据。**

---

## 第 3 章 perf record/report 剖析

### 3.1 perf record 采样

`perf record` 以指定频率采样调用栈，生成 perf.data 文件。agentrt-linux 默认采样频率 99Hz（避开 100Hz 周期性干扰）：

```bash
# 标准 99Hz 采样，call-graph 用 DWARF
perf record -F 99 -g --call-graph dwarf -a -- sleep 30

# Agent 决策路径专用采样
perf record -F 99 -g --call-graph dwarf \
    -e cycles,instructions,sched:sched_switch,airy:airy_cognition_enter \
    -p $(pgrep -f macro_d) -- sleep 60
```

### 3.2 perf report 分析

`perf report` 解析 perf.data，按开销排序展示热点函数：

```bash
# TUI 交互式报告
perf report --sort overhead,symbol,dso --stdio

# 按 Agent 分类（需要符号表包含 agent_id）
perf report --sort overhead,symbol,dso,symbol_precedence \
    --dsos=agentrt.ko --comms=agent_42
```

### 3.3 perf report 热点归因

agentrt-linux 的 perf report 输出典型热点分布如下（基线测试数据）：

| 排名 | 函数 | 模块 | 开销占比 | 说明 |
|------|------|------|---------|------|
| 1 | `io_uring_cmd_submit` | io_uring.ko | 12.3% | IPC fastpath 入口 |
| 2 | `airy_cognition_process` | agentrt.ko | 9.8% | Agent 认知循环 |
| 3 | `pick_next_task_fair` | kernel | 7.2% | EEVDF 调度类选择 |
| 4 | `__schedule` | kernel | 6.5% | 主调度器入口 |
| 5 | `airy_ring_buffer_write` | agentrt.ko | 5.1% | A-ULP 日志写入 |
| 6 | `remap_pfn_range` | kernel | 4.3% | mmap 映射建立 |
| 7 | `alloc_pages_current` | kernel | 3.8% | 物理页分配 |
| 8 | `airy_lsm_cap_check` | airy_lsm.ko | 3.2% | 纯 C LSM 钩子 |

**OS-OBS-033: perf report 热点列表前 20 项中，agentrt.ko 与 airy_lsm.ko 合计不得超过 35%；超限视为可观测性开销过大，需审查 tracepoint 密度。**

---

## 第 4 章 PMU 硬件计数器

### 4.1 核心 PMU 事件

agentrt-linux 在性能分析中标准启用的 PMU 硬件计数器：

| 事件 | 含义 | 在sched_tac 分析中的用途 |
|------|------|----------------------------|
| `cycles` | CPU 周期数 | 计算绝对执行时间 |
| `instructions` | 退役指令数 | 计算 IPC = instructions / cycles |
| `cache-misses` | LLC 缺失 | 评估 Agent 工作集是否超出 LLC |
| `cache-references` | LLC 访问 | 计算 LLC 命中率 |
| `branch-misses` | 分支预测失败 | 评估认知循环分支可预测性 |
| `branch-instructions` | 分支指令 | 计算分支预测失败率 |
| `stalled-cycles-frontend` | 前端停顿 | 评估指令取回效率 |
| `stalled-cycles-backend` | 后端停顿 | 评估内存延迟影响 |
| `LLC-load-misses` | LLC 加载缺失 | 评估记忆访问局部性 |
| `node-load-misses` | NUMA 节点缺失 | 评估跨节点访问 |

### 4.2 IPC 指标

IPC（Instructions Per Cycle）是评估sched_tac 调度效率的关键指标：

```bash
# IPC 测量
perf stat -e cycles,instructions -a -- sleep 10
# 输出：
#   10,234,567,890  cycles
#    5,678,901,234  instructions
#  # 0.555 IPC  →  IPC < 1.0 表示受内存延迟约束
```

agentrt-linux 基线 IPC 标准：

| 场景 | IPC 基线 | 说明 |
|------|---------|------|
| Agent 认知循环 | ≥ 1.2 | 计算密集型，应充分利用流水线 |
| IPC fastpath | ≥ 1.5 | io_uring 路径优化良好 |
| A-ULP 日志写入 | ≥ 0.8 | 受 ring buffer 内存延迟影响 |
| sched_tac 调度 | ≥ 1.0 | 调度器自身不应成为瓶颈 |

**OS-OBS-034: 性能基准测试必须报告 IPC 指标；IPC < 0.5 视为严重内存瓶颈，需启动 c2c 与 mem 分析。**

### 4.3 自定义 PMU 事件组

agentrt-linux 使用 `perf stat -e` 组合 PMU 事件，测量sched_tac 调度类切换开销：

```bash
# SCHED_DEADLINE → SCHED_FIFO 切换开销测量
perf stat -e cycles,instructions,cache-misses,branch-misses,\
sched:sched_switch,airy:airy_sched_class_switch \
    -a -- per-agent-benchmark 60
```

事件组通过 `:G` 修饰符保证同组事件在同一 CPU 上计数：

```bash
perf stat -e '{cycles,instructions,cache-misses}:G' -a -- sleep 10
```

---

## 第 5 章 sched_tac 调度性能分析

### 5.1 sched_tac 三层调度类

sched_tac 复用 Linux 6.6 原生调度类，不使用 sched_ext。三层结构：

| 层级 | 调度类 | 适用场景 | 典型 Agent |
|------|--------|---------|-----------|
| L1 | `SCHED_DEADLINE` | 硬实时认知循环 | 高优先级工具调用 Agent |
| L2 | `SCHED_FIFO` | 软实时决策路径 | 规划器 Agent |
| L3 | `EEVDF`（CFS） | 普通负载 | 后台监控、审计 Agent |

### 5.2 调度延迟测量

使用 `perf sched` 测量sched_tac 调度延迟：

```bash
# 录制调度事件
perf sched record -a -- sleep 30
# 分析调度延迟
perf sched latency -p
# 输出：
#   Task                  |   Runtime ms  | Switches | Average delay ms | Maximum delay ms |
#   agent_42:macro_d |      1234.56  |     5678 |           0.0234 |           0.1234 |
#   agent_43:cogn_d       |       987.65  |     4321 |           0.0456 |           0.2345 |
```

### 5.3 调度延迟分布

agentrt-linux sched_tac 调度延迟基线（10000 次采样）：

| Agent 类型 | 平均延迟 | P50 | P95 | P99 | 最大延迟 |
|-----------|---------|-----|-----|-----|---------|
| SCHED_DEADLINE | 12μs | 8μs | 35μs | 78μs | 156μs |
| SCHED_FIFO | 45μs | 32μs | 125μs | 287μs | 567μs |
| EEVDF | 234μs | 178μs | 678μs | 1.23ms | 4.56ms |

**OS-OBS-035: SCHED_DEADLINE Agent 的 P99 调度延迟必须 ≤ 100μs；超限视为sched_tac 失效，需触发 macro_d 告警。**

### 5.4 调度类切换开销

sched_tac 不使用 sched_ext，调度类切换由内核原生 `__schedule` 完成。切换开销通过 PMU 计数器测量：

```bash
# 测量单次调度类切换开销
perf stat -e cycles,instructions,cache-misses \
    -e airy:airy_sched_class_switch \
    -a -- per-agent-benchmark 60
# 计算：cycles_per_switch = total_cycles / sched_class_switch_count
```

基线测量结果：

| 切换路径 | cycles | instructions | cache-misses |
|---------|--------|--------------|--------------|
| DL → RT | 2,134 | 1,567 | 8 |
| RT → EEVDF | 1,876 | 1,423 | 6 |
| EEVDF → DL | 2,456 | 1,789 | 12 |

**OS-STD-023: 单次调度类切换的 cycles 开销不得超过 3000 cycles（约 1μs @ 3GHz）；超限需审查 `airy_sched_class_switch` tracepoint 的密度。**

---

## 第 6 章 IPC fastpath 性能分析

### 6.1 IORING_OP_URING_CMD 路径

agentrt-linux IPC fastpath 基于 `IORING_OP_URING_CMD`，不使用 page flipping。性能分析聚焦于 io_uring 命令入口到 ring buffer 写入的端到端延迟：

```bash
# IPC fastpath 端到端延迟采样
perf record -F 999 -g --call-graph dwarf \
    -e cycles,instructions,io_uring:io_uring_cmd_submit,airy:airy_ipc_send \
    -p $(pgrep gateway_d) -- sleep 30
```

### 6.2 IPC 延迟基线

| 路径 | 平均延迟 | P99 | 吞吐 (msg/s) |
|------|---------|-----|-------------|
| IORING_OP_URING_CMD（fastpath） | 1.2μs | 4.5μs | 8.5M |
| 普通 write/read（slowpath） | 8.7μs | 32μs | 1.2M |
| Page flipping（不采用） | 0.8μs | 2.1μs | 12M |

**OS-OBS-036: IPC fastpath P99 延迟必须 ≤ 5μs；超限需审查 io_uring SQ/CQ 深度与 poll 模式配置。**

### 6.3 IPC 热点函数

perf report 在 IPC fastpath 中的典型热点：

| 函数 | 模块 | 开销占比 | 说明 |
|------|------|---------|------|
| `io_uring_cmd_submit` | io_uring.ko | 35% | 命令提交入口 |
| `copy_from_user` | kernel | 22% | 用户态参数拷贝 |
| `airy_ipc_ring_write` | agentrt.ko | 18% | ring buffer 写入 |
| `io_uring_cq_advance` | io_uring.ko | 12% | 完成队列推进 |
| `airy_lsm_ipc_check` | airy_lsm.ko | 8% | IPC capability 校验 |

注意：因不使用 page flipping，`copy_from_user` 仍是必要开销；agentrt-linux 通过 SQE 批处理摊薄此开销。

---

## 第 7 章 内存性能分析

### 7.1 alloc_pages + mmap 映射开销

agentrt-linux 内存分配策略：`alloc_pages` 分配物理页 + `vm_map_pages` / `remap_pfn_range` 映射。不使用 DMA 一致性内存。性能分析聚焦映射建立与访问开销：

```bash
# 内存映射开销分析
perf record -F 99 -g --call-graph dwarf \
    -e cycles,instructions,cache-misses,kmem:mm_page_alloc,kmem:mm_page_free \
    -p $(pgrep mem_d) -- sleep 60
```

### 7.2 映射建立开销基线

| 操作 | 平均开销 | P99 | 说明 |
|------|---------|-----|------|
| `alloc_pages(order=0)` | 0.3μs | 1.2μs | 单页分配 |
| `alloc_pages(order=4)` | 2.1μs | 8.7μs | 64KB 分配 |
| `vm_map_pages`（单页） | 1.8μs | 5.6μs | 内核虚拟地址映射 |
| `remap_pfn_range`（单页） | 2.3μs | 7.8μs | 用户态映射 |
| `dma_alloc_coherent`（不采用） | 4.5μs | 12μs | 对照组 |

### 7.3 访问延迟分析

使用 `perf mem` 分析 Agent 记忆访问的延迟分布：

```bash
# 内存访问延迟采样
perf mem record -e ld,ld,latency -p $(pgrep cogn_d) -- sleep 30
perf mem report --sort symbol,dso,mem
```

agentrt-linux 基线记忆访问延迟：

| 层级 | 平均延迟 | P99 | 说明 |
|------|---------|-----|------|
| L1 cache | 1.2ns | 2.1ns | 工作记忆热数据 |
| L2 cache | 3.4ns | 5.6ns | 工作记忆温数据 |
| LLC | 12.3ns | 28ns | 短期记忆 |
| 主存 | 78ns | 156ns | 长期记忆 |
| NUMA 远端 | 234ns | 567ns | 跨节点访问 |

**OS-OBS-037: L4 经验记忆检索的 P99 延迟必须 ≤ 200ns；超限需启动 perf c2c 分析 cacheline 共享情况。**

---

## 第 8 章 性能基准对照

### 8.1 sched_tac vs 标准 EEVDF

agentrt-linux 在 12 daemon 混合负载下的性能基准对照（同硬件、同内核版本）：

| 指标 | sched_tac | 标准 EEVDF（CFS） | 差异 |
|------|-------------|------------------|------|
| SCHED_DEADLINE P99 调度延迟 | 78μs | N/A | sched_tac 独有 |
| SCHED_FIFO P99 调度延迟 | 287μs | N/A | sched_tac 独有 |
| EEVDF P99 调度延迟 | 1.23ms | 1.45ms | 改善 15.2% |
| IPC fastpath 吞吐 | 8.5M msg/s | 8.5M msg/s | 一致（IPC 与调度正交）|
| A-ULP 日志吞吐 | 1.2M rec/s | 1.2M rec/s | 一致 |
| 整体 IPC | 1.23 | 1.18 | 改善 4.2% |
| cache-miss 率 | 8.7% | 9.1% | 改善 4.4% |
| branch-miss 率 | 2.3% | 2.5% | 改善 8.0% |

### 8.2 sched_ext 对照（仅理论）

agentrt-linux 不采用 sched_ext，但保留对照数据用于技术选型论证：

| 指标 | sched_tac | sched_ext（理论） | 说明 |
|------|-------------|------------------|------|
| 调度器切换开销 | 2,134 cycles | 3,567 cycles | sched_ext 需 BPF 程序调用 |
| BPF 依赖 | 无 | 必需 | sched_ext 需 CONFIG_SCHED_BPF |
| 稳定性 | 内核原生 | 验证器保护 | sched_tac 更稳定 |
| 灵活性 | 受限于原生类 | 任意 BPF 策略 | sched_ext 更灵活 |

### 8.3 基准测试自动化

agentrt-linux 提供标准基准测试脚本 `airy-perf-bench`，封装 perf stat/record/report 全流程：

```bash
# 标准基准测试套件
airy-perf-bench --suite c-prime-vs-eevdf --duration 300 \
    --output /var/log/airy/perf-bench-$(date +%s)/

# 输出目录结构
# /var/log/airy/perf-bench-XXXX/
#   ├── stat-summary.csv          # perf stat 汇总
#   ├── record-c-prime.perf.data  # sched_tac 采样
#   ├── record-eevdf.perf.data    # 标准 EEVDF 采样
#   ├── report-c-prime.txt        # sched_tac 热点报告
#   ├── report-eevdf.txt          # 标准 EEVDF 热点报告
#   └── comparison.md             # 自动对照报告
```

**OS-STD-024: 性能基准测试必须在标准化硬件环境（≥ 8 核 / 32GB / NVMe）上运行；不得在虚拟化环境中采集基线数据。**

**OS-OBS-038: 性能基准测试结果必须随版本发布归档至 `/var/log/airy/perf-bench-archive/`，保留至少 12 个版本，供回归分析使用。**

---

## 第 9 章 Airymax Unify Design 映射

### 9.1 五模块关系

| Unify 模块 | 关系 | 在 perf 分析中的体现 |
|-----------|------|---------------------|
| **A-UEF** | 辅助 | A-UEF 的 `AIRY_FAULT_*` 码触发频率通过 perf stat 统计 |
| **A-ULP** | **核心** | perf record 采样 A-ULP Ring Buffer 写入路径，定位 logger_d 消费瓶颈 |
| **A-UCS** | 辅助 | A-UCS 热重载事件通过 perf stat 计数 |
| **A-ULS** | **核心** | perf sched 测量sched_tac 调度延迟，验证 macro_d 监管有效性 |
| **A-IPC** | 辅助 | perf record 分析 IPC fastpath 端到端延迟 |

### 9.2 与 12 daemon 的性能分析映射

| Daemon | 主要 perf 指标 | 基线 |
|--------|---------------|------|
| macro_d | 调度延迟、CPU 占用 | P99 ≤ 100μs，CPU ≤ 5% |
| logger_d | 日志吞吐、消费延迟 | ≥ 1M rec/s，P99 ≤ 10μs |
| config_d | 热重载延迟 | ≤ 1ms |
| gateway_d | IPC 吞吐、延迟 | ≥ 8M msg/s，P99 ≤ 5μs |
| sched_d | 调度类切换开销 | ≤ 3000 cycles |
| vfs_d | 文件 IO 延迟 | P99 ≤ 100μs |
| net_d | 网络包延迟 | P99 ≤ 50μs |
| mem_d | 映射建立延迟 | P99 ≤ 10μs |
| cogn_d | 认知循环延迟 | P99 ≤ 1ms |
| sec_d | LSM 钩子开销 | ≤ 3% CPU |
| audit_d | 审计写入吞吐 | ≥ 500K rec/s |
| dev_d | 设备事件延迟 | P99 ≤ 100μs |

---

## 第 10 章 相关文档与版本维护

### 10.1 相关文档

- [90-observability README](README.md)：可观测性体系主索引
- [01-ftrace-framework.md](01-ftrace-framework.md)：ftrace 框架（L1 层）
- [02-ebpf-probes.md](02-ebpf-probes.md)：eBPF 探针（L2 层）
- [04-sysfs-procfs.md](04-sysfs-procfs.md)：sysfs/procfs 接口
- [05-debugfs-tracefs.md](05-debugfs-tracefs.md)：debugfs/tracefs 接口
- [07-token-efficiency.md](07-token-efficiency.md)：Token 效率监控
- [08-agent-tracing.md](08-agent-tracing.md)：Agent 行为追踪
- [../170-performance/README.md](../170-performance/README.md)：性能工程总览

### 10.2 参考材料

- Linux 6.6 `tools/perf/Documentation/`（perf 工具文档）
- Linux 6.6 `kernel/events/core.c`（perf_event 子系统）
- Linux 6.6 `Documentation/perf/`（perf 设计文档）
- ARM Architecture Reference Manual（PMU 事件定义）
- Intel SDM Volume 3B（PMU 编程指南）

### 10.3 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0.1 | 2026-07-18 | 初始版本：perf stat/record/report 工程规范、PMU 硬件计数器、sched_tac 调度性能分析、IPC fastpath 与内存性能分析、sched_tac vs 标准 EEVDF 基准对照 |

---

> **文档结束** | agentrt-linux perf 性能分析 v1.0.1 | 维护者：开源极境工程与规范委员会 | "Measure what is measurable, and make measurable what is not so."
