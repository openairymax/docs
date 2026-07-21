Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 性能工程设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）性能工程体系主索引——调度性能、IPC 性能、内存性能、Token 效率与 SLO 设定\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[AirymaxOS 总览](../README.md)

---

## 1. 模块概述

`170-performance/` 是 AirymaxOS 文档体系中**系统在高负载场景下稳定运行的工程保障**。它继承 Linux 内核 30+ 年沉淀的性能工程哲学（perf 剖析 + sched_tac 调度 + MGLRU 内存回收 + io_uring 零拷贝 I/O），并在其上扩展智能体操作系统专属的 Token 能效工程、Agent 延迟 SLO、记忆卷载性能等。

性能工程不是「调优」的代名词，而是「可测量、可预测、可保障」的工程体系。AirymaxOS 性能工程的核心理念是：**一切性能指标必须可测量，一切性能回归必须可追溯，一切性能 SLO 必须可保障**。这要求从设计阶段就将性能纳入考量，而非在开发末期才进行「调优」。

本模块承担五项核心职责：

1. **调度性能**：sched_tac 三层调度（SCHED_DEADLINE / SCHED_FIFO / EEVDF）性能 SLO 设定与剖析。
2. **IPC 性能**：A-IPC（IORING_OP_URING_CMD）零拷贝 fastpath 性能与吞吐。
3. **内存性能**：MGLRU 多代 LRU + CXL 内存分层 + PMEM DAX + 记忆卷载性能。
4. **Token 效率**：Token / Watt + Token / Latency + Token / Dollar 三大能效指标。
5. **Agent 延迟 SLO**：L1/L2/L3 三级延迟 SLO + SLO 保障机制 + 延迟预算管理。

---

## 2. 技术选型声明

agentrt-linux v1.0 在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度做出**不可妥协**的技术选型。本目录所有性能 SLO 与基准测试必须以本声明为基线——性能指标均基于sched_tac 原生调度类，不针对 sched_ext / page flipping / BPF LSM / DMA 一致性内存设定任何性能基线。

| # | 技术维度 | 选定方案 | 明确不采用 | 选定理由 |
|---|---------|---------|-----------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类，通过 `nice` / `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，**不定义新的调度类宏** | **不使用 sched_ext**（不引入 eBPF 调度器、不依赖 `SCHED_EXT=7`、不使用 SCHED_AGENT 宏） | 保留 Linux 6.6 主线调度器稳定性，性能 SLO 基于既有调度类可预测、可保障 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输，registered buffer + mmap 共享页 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | IORING_OP_URING_CMD 在 Linux 6.6 已稳定，fastpath 性能可预测 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 Linux Security Module（`airy_lsm`），通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | 纯 C LSM fastpath 通过 `static_key` 跳过，零开销可预测 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `remap_pfn_range` 映射到用户态地址空间 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | alloc_pages + mmap 跨架构内存语义一致，性能可跨架构复现 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | v2 三层模型（升级为 v3，新增 [DSL] 降级生存层） | [DSL] 降级生存层确保 [SC] 损坏时系统仍可降级运行，性能基准跨语言一致 |

### 2.1 sched_tac 三层调度性能 SLO（本模块专属）

> sched_tac 调度性能 SLO 是 AirymaxOS 性能工程的不可妥协基线，所有性能回归测试必须校验下表指标：

| # | 性能指标 | SLO 阈值 | 调度类 / 路径 | 说明 |
|---|---------|---------|--------------|------|
| 1 | **SCHED_DEADLINE 调度延迟** | **≤ 150ns** | `SCHED_DEADLINE`（sched_tac 第一层） | 实时确定性 Agent，Deadline 调度类入队延迟 |
| 2 | **SCHED_FIFO 调度延迟** | **≤ 100ns** | `SCHED_FIFO`（sched_tac 第二层） | 高优先级 Agent，FIFO 调度类入队延迟 |
| 3 | **EEVDF 调度延迟** | **≤ 100ns** | `EEVDF`（sched_tac 第三层，Linux 6.6 默认） | 普通 Agent，EEVDF 调度类入队延迟 |
| 4 | **IPC fastpath 延迟** | **≤ 50ns** | A-IPC（IORING_OP_URING_CMD） | Agent 间零拷贝消息延迟 |
| 5 | **日志 Ring Buffer 写入延迟** | **≤ 100ns** | A-ULP（alloc_pages + mmap） | 内核 fastpath 128B 记录 reserve + memcpy + commit |
| 6 | **Capability 校验延迟** | **≤ 30ns** | 纯 C LSM（`static_key` fastpath） | 第一层轻量标记，99%+ 请求零开销放行 |
| 7 | **IPC 吞吐量** | **≥ 10M msg/s** | A-IPC（IORING_OP_URING_CMD + registered buffer） | Agent 间消息吞吐量 |

---

## 3. 文档索引

`170-performance/` 由 7 个文档构成（含本 README）：

```
170-performance/
├── README.md                       # 本文件 — 性能工程主索引（v1.0）
├── 01-scheduling-performance.md   # 调度性能（sched_tac + EEVDF + vtime 衰减）（v1.0）
├── 02-memory-performance.md       # 内存性能（MGLRU 多代 LRU + CXL + PMEM DAX）（v1.0）
├── 03-ipc-performance.md          # IPC 性能（AgentsIPC + io_uring 批量提交）（v1.0）
├── 04-token-efficiency.md         # Token 能效工程（Token/Watt + Token/Latency + Token/Dollar）（v1.0）
├── 05-agent-latency-slo.md       # Agent 延迟 SLO（L1/L2/L3 三级 + 延迟预算）（v1.0）
└── 06-benchmark-suite.md          # 基准测试套件（4 层分层 + 回归检测 + CI/CD）（v1.0）
```

### 3.1 各文档定位

| 文档 | 核心问题 | 主要产物 |
|------|---------|---------|
| README.md | 性能工程全貌？ | 7 层分层 + sched_tac SLO + 技术选型声明 |
| 01-scheduling-performance.md | 调度性能怎么保障？ | sched_tac 三层 SLO + EEVDF + vtime 衰减 |
| 02-memory-performance.md | 内存性能怎么保障？ | MGLRU + CXL 分层 + PMEM DAX + 记忆卷载性能 |
| 03-ipc-performance.md | IPC 性能怎么保障？ | AgentsIPC 零拷贝 + io_uring 批量提交 + ≤50ns SLO |
| 04-token-efficiency.md | Token 能效怎么测？ | 三大能效指标 + RAPL 功耗测量 + 能效回归检测 |
| 05-agent-latency-slo.md | Agent 延迟怎么保障？ | L1/L2/L3 三级 SLO + 违约处理 + 延迟预算 |
| 06-benchmark-suite.md | 基准测试怎么做？ | 4 层分层体系（L1 微基准/L2 子系统/L3 端到端/L4 压力）+ CI 集成 |

### 3.2 性能工程分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | CPU 性能 | perf + sched_tac + EEVDF | CPU 调度 |
| L2 | 内存性能 | MGLRU 多代 LRU + CXL + PMEM | 内存管理 |
| L3 | I/O 性能 | io_uring + zero-copy | I/O 子系统 |
| L4 | 网络性能 | AF_XDP + DPDK | 网络栈 |
| L5 | 锁与并发 | lockdep + KCSAN | 并发正确性 |
| **L6** | **Token 能效** | **AirymaxOS 专属** | **Agent Token 消耗** |
| **L7** | **Agent 延迟 SLO** | **AirymaxOS 专属** | **Agent 响应延迟** |

### 3.3 Agent 延迟 SLO（应用层）

| 级别 | 延迟阈值 | 适用场景 |
|------|---------|---------|
| L1 认知延迟 | ≤ 100ms | 感知 → 理解 |
| L2 规划延迟 | ≤ 1s | 理解 → 决策 |
| L3 执行延迟 | ≤ 10s | 决策 → 执行 |

---

## 4. Airymax Unify Design 映射

Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）在性能工程中的关系如下。性能工程是 A-ULS sched_tac 调度性能 SLO、A-IPC IPC fastpath 性能、A-ULP 日志 Ring Buffer 性能的主要度量维度。

| Unify 模块 | 全称 | 在本目录的体现 | 性能 SLO |
|-----------|------|--------------|---------|
| **A-ULS** | Unified Lifecycle Supervision Framework | **核心**：sched_tac 三层调度性能 SLO（SCHED_DEADLINE ≤150ns / SCHED_FIFO ≤100ns / EEVDF ≤100ns）、Agent 8 态生命周期调度开销 | 三层调度延迟 SLO |
| **A-IPC** | Unified Airymax IPC Fabric | **核心**：IPC fastpath 性能（IORING_OP_URING_CMD ≤50ns）、IPC 吞吐量（≥10M msg/s） | IPC ≤50ns / 吞吐 ≥10M msg/s |
| **A-ULP** | Unified Logging and Printk Subsystem | **核心**：日志 Ring Buffer 性能（alloc_pages + mmap ≤100ns）、Logger Daemon 异步格式化吞吐 | 日志 ≤100ns |
| **A-UEF** | Unified Error and Fault Framework | 错误码查询性能、CoreLoopThree 认知循环吞吐量与延迟 | 认知循环吞吐 |
| **A-UCS** | Unified Configuration Subsystem | 配置 RCU 热重载性能、sysctl 写入延迟 | 配置切换无锁 |

### 4.1 A-ULS sched_tac 调度性能 SLO

A-ULS 的 Agent 8 态生命周期由sched_tac 三层调度承载，性能 SLO 如下：

```c
/* sched_tac 三层调度 —— 性能 SLO 校验 */
SCHED_DEADLINE  → ≤ 150ns  /* 第一层：实时确定性 Agent */
SCHED_FIFO      → ≤ 100ns  /* 第二层：高优先级 Agent */
EEVDF           → ≤ 100ns  /* 第三层：普通 Agent（Linux 6.6 默认） */
```

sched_tac 复用 Linux 6.6 原生调度类（**不使用 sched_ext**），通过 `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，性能可预测、可保障。Capability 校验由纯 C LSM 的 `static_key` fastpath 承载（**不使用 BPF LSM**），延迟 ≤30ns。

### 4.2 A-IPC IPC fastpath 性能

A-IPC 的 IPC fastpath 基于 **IORING_OP_URING_CMD**（**不使用 page flipping**），性能 SLO：

- **延迟**：≤ 50ns（registered buffer + mmap 共享页零拷贝）
- **吞吐**：≥ 10M msg/s（io_uring 批量提交 + 128B 定长消息头）
- **冻结检查**：`unlikely(ring->frozen)` 零开销，正常路径无分支预测失败

### 4.3 A-ULP 日志 Ring Buffer 性能

A-ULP 的日志 Ring Buffer 基于 **alloc_pages + mmap**（**不使用 DMA 一致性内存**），性能 SLO：

- **fastpath 延迟**：≤ 100ns（reserve 128B slot + memcpy 128B raw + commit 原子索引）
- **格式化异步**：Logger Daemon 异步 sprintf，不阻塞 fastpath
- **Panic 回退**：printk_safe 原生路径，确保 Panic 日志可靠落盘

### 4.4 同源 agentrt 性能基线（IRON-9 v3）

性能工程遵循 **IRON-9 v3 四层模型**与 agentrt 性能基线同源：[SC] 共享性能相关头文件，[SS] 语义同源（性能指标语义一致），[IND] 各自独立实现（agentrt `airy_metrics_record()` 用户态采集 ↔ AirymaxOS `perf_event` + ftrace 内核剖析），[DSL] 降级生存块（#ifdef AIRY_SC_FALLBACK），通过 user_events 桥接性能数据。

---

## 5. 相关文档

### 5.1 上级与架构文档
- [AirymaxOS 总览](../README.md) —— 文档体系顶层纲领（v1.0）
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（五模块 SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 四层模型

### 5.2 模块与数据流文档
- [20-modules/04-memory.md](../20-modules/04-memory.md) —— memory 子仓（MGLRU + CXL）
- [20-modules/05-cognition.md](../20-modules/05-cognition.md) —— cognition 子仓（CoreLoopThree）
- [40-dataflows/03-ipc-flow.md](../40-dataflows/03-ipc-flow.md) —— IPC 数据流（A-IPC）
- [40-dataflows/04-scheduling-flow.md](../40-dataflows/04-scheduling-flow.md) —— 调度数据流（sched_tac）
- [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— Ring Buffer 日志（A-ULP）

### 5.3 关联模块文档
- [90-observability/README.md](../90-observability/README.md) —— 可观测性（perf + ftrace + eBPF 探针）
- [100-operations/README.md](../100-operations/README.md) —— 运维体系
- [80-testing/README.md](../80-testing/README.md) —— 测试体系（性能基准测试）
- [140-application-development/README.md](../140-application-development/README.md) —— Agent 应用开发（Token 预算）
- [130-roadmap/README.md](../130-roadmap/README.md) —— 路线图（M7 性能 SLO 验收）

### 5.4 参考材料
- Linux 6.6 `tools/perf/`（perf 工具）
- Linux 6.6 `kernel/sched/`（调度子系统：`deadline.c` / `rt.c` / `fair.c`）
- Linux 6.6 `mm/`（内存管理）+ `io_uring/`（io_uring 子系统）
- sched_tac 调度基础（Linux 6.6 调度类 + seL4 MCS 时间隔离）

---

## 6. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，6/6 文档完成（调度 + 内存 + IPC + Token + SLO + 基准测试） |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）；IORING_OP_URING_CMD（不使用 page flipping）；纯 C LSM（不使用 BPF LSM）；alloc_pages + mmap（不使用 DMA 一致性内存）；IRON-9 v3 四层模型（新增 [DSL] 降级生存层）；新增sched_tac 三层调度性能 SLO（SCHED_DEADLINE ≤150ns / SCHED_FIFO ≤100ns / EEVDF ≤100ns / IPC ≤50ns / 日志 ≤100ns / Capability ≤30ns / 吞吐 ≥10M msg/s）；新增 Airymax Unify Design 五模块映射（A-ULS 调度性能 / A-IPC IPC fastpath / A-ULP 日志 Ring Buffer） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | 性能工程模块 v1.0 | 共 7 文档 | 维护者：开源极境工程与规范委员会 | 一切性能 SLO 必须可保障
