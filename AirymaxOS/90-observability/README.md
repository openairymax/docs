Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）可观测性设计
> **文档定位**：agentrt-linux（AirymaxOS）可观测性工程体系主索引（eBPF 探针 + tracepoint + user_events + A-ULP 日志观测）\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[AirymaxOS 总览](../README.md)\
> **同源映射**：agentrt E-2 可观测性原则 + Linux 6.6 ftrace/eBPF/perf\
> **理论根基**：Linux 内核可观测性 + Airymax E-2 可观测性 + C-3 记忆卷载

---

## 1. 模块概述

agentrt-linux 可观测性体系是系统运行状态可见性的核心保障。它继承 Linux 内核 30+ 年沉淀的多层可观测性哲学（ftrace + eBPF + perf + 4 层文件系统接口），并在其上扩展智能体操作系统专属的 A-ULP 日志观测、调度器状态观测、Token 能效可观测性、Agent 行为追踪、记忆卷载监控等。本目录覆盖四方面职责：

1. **eBPF 探针**：通过 kfunc + BTF + dynamic pointer 提供可编程的内核扩展能力，追踪 Agent 决策路径（认知→规划→调度→执行）。**注意：eBPF 仅用于可观测性探针，不用于安全钩子**（安全钩子由纯 C LSM 承载）。
2. **tracepoint**：通过 `DECLARE_TRACE` / `trace_*()` 宏暴露内核静态追踪点，覆盖sched_tac 调度类切换、IPC fastpath 入口、Ring Buffer 写入等关键路径。
3. **user_events**：用户态守护进程通过 user_events 将事件上报到 ftrace ring buffer，实现用户态与内核态统一跟踪。
4. **A-ULP 日志观测**：观测 A-ULP（Unified Logging and Printk Subsystem）的 Ring Buffer 写入路径、Logger Daemon 消费速率、Panic 回退到 `printk_safe` 的状态。这是 A-ULP 模块在可观测性侧的镜像。

### 1.1 可观测性分层

| 层级 | 类型 | 工具 | 用途 |
|------|------|------|------|
| L1 | 内核跟踪 | ftrace（tracefs + ring buffer） | 函数级跟踪 |
| L2 | 可编程探针 | eBPF（kfunc + dynamic pointer） | 动态扩展（仅观测，非安全） |
| L3 | 性能分析 | perf（分析前端） | 性能剖析 |
| L4 | 文件系统接口 | sysfs/procfs/debugfs/tracefs | 静态信息导出 |
| L5 | 用户态桥接 | user_events | 用户态事件上报 |
| **L6** | **A-ULP 日志观测** | **agentrt-linux 专属** | **Ring Buffer + Logger Daemon 观测** |
| **L7** | **调度器状态观测** | **agentrt-linux 专属** | **sched_tac 调度类状态 + Agent 8 态** |
| **L8** | **Token 能效监控** | **agentrt-linux 专属** | **Agent Token 消耗追踪** |
| **L9** | **Agent 行为追踪** | **agentrt-linux 专属** | **Agent 决策路径审计** |
| **L10** | **记忆卷载监控** | **agentrt-linux 专属** | **L1→L2→L3→L4 记忆演化** |

### 1.2 agentrt-linux 专属扩展

- **A-ULP 日志观测仪表盘**：实时监控 Ring Buffer 写入速率、Logger Daemon 消费延迟、eventfd 通知频率、Panic 回退触发
- **调度器状态观测**：通过 tracepoint 暴露sched_tac 三层调度类（`SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF`）的切换事件、Agent 8 态生命周期迁移
- **Token 能效仪表盘**：实时监控每个 Agent 的 Token 消耗与能效比
- **Agent 行为追踪**：通过 eBPF 探针追踪 Agent 决策路径（认知→规划→调度→执行）
- **记忆卷载监控**：监控 L1（原始卷）→L2（特征层）→L3（结构层）→L4（模式层）的记忆演化

---

## 2. 技术选型声明

agentrt-linux v1.0 可观测性体系在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度遵循 [AirymaxOS 总览](../README.md) §2 的不可妥协基线。可观测性体系是五大选型的**运行时见证者**——通过 tracepoint 与 eBPF 探针持续观测五大选型的运行时行为。五个维度的选型在本目录的具体落地如下：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 在本目录的落地 |
|---|---------|---------|----------------|--------------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 `SCHED_EXT=7` 调度类） | 调度器状态观测 tracepoint 暴露sched_tac 三层调度类切换事件；不观测 sched_ext（因未启用） |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | IPC fastpath tracepoint 观测 `io_uring_cmd` 入口/出口、Ring Buffer 写入延迟；不观测 page flipping 路径 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 `airy_lsm` 通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | **eBPF 探针仅用于可观测性，不用于安全钩子**；安全钩子的 capability 校验命中/未命中通过 tracepoint 观测 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | A-ULP Ring Buffer 内存路径观测（`alloc_pages` 分配 + `remap_pfn_range` 映射）；不观测 DMA 一致性内存路径 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | （v2 三层模型升级为 v3 四层模型，新增 [DSL] 降级生存层） | A-ULP [SC] `log_types.h` 的 128B 记录格式通过 user_events 桥接到用户态；观测数据格式遵循 [SC] 契约 |

### 2.1 IRON-9 v3 四层模型在可观测性的归属

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 落地文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| A-ULP 128B 日志记录观测 | ● | — | — | — | [01-ftrace-framework.md](01-ftrace-framework.md) |
| sched_tac 调度类观测 | — | — | ● | — | [01-ftrace-framework.md](01-ftrace-framework.md) |
| eBPF Agent 行为追踪 | — | — | ● | — | [02-ebpf-probes.md](02-ebpf-probes.md) |
| Panic 回退观测（[DSL]） | — | — | — | ● | [01-ftrace-framework.md](01-ftrace-framework.md) |
| user_events 用户态桥接 | — | ● | — | — | [02-ebpf-probes.md](02-ebpf-probes.md) |

---

## 3. 文档索引

本目录现有文档（v1.0 范围）：

```
90-observability/
├── README.md                       # 本文件（v1.0）
├── 01-ftrace-framework.md          # ftrace 框架 + tracepoint + A-ULP 日志观测 + 调度器状态观测
└── 02-ebpf-probes.md               # eBPF 可编程探针（仅观测）+ user_events + Agent 行为追踪
```

| # | 文档 | 版本 | 内容概要 |
|---|------|------|---------|
| — | [README.md](README.md) | v1.0 | 可观测性主索引（本文件） |
| 1 | [01-ftrace-framework.md](01-ftrace-framework.md) | v1.0 | ftrace 框架、tracepoint 静态追踪点、A-ULP Ring Buffer 日志观测、sched_tac 调度器状态观测、Panic 回退（[DSL]）观测 |
| 2 | [02-ebpf-probes.md](02-ebpf-probes.md) | v1.0 | eBPF 可编程探针（仅观测，非安全）、kfunc + BTF + dynamic pointer、user_events 用户态桥接、Agent 行为追踪、Token 能效监控 |

### 3.1 后续规划文档（1.0.1 版本）

以下文档在 1.0.1 版本完成，不在 v1.0 范围内：

- `03-perf-analysis.md`：perf 性能分析
- `04-sysfs-procfs.md`：sysfs/procfs 接口
- `05-debugfs-tracefs.md`：debugfs/tracefs 接口
- `06-user-events.md`：user_events 用户态桥接详解
- `07-token-efficiency.md`：agentrt-linux 专属 Token 能效监控
- `08-agent-tracing.md`：agentrt-linux 专属 Agent 行为追踪
- `09-memory-monitoring.md`：agentrt-linux 专属记忆卷载监控

---

## 4. Airymax Unify Design 映射

本目录与 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的关系详见 [AirymaxOS 总览](../README.md) §5。可观测性体系主要承载 **A-ULP**（日志观测）与 **A-ULS**（调度器状态观测），其余三模块为辅助关系。

| Unify 模块 | 关系 | 在本目录的体现 |
|-----------|------|--------------|
| **A-UEF** | 辅助 | A-UEF 的 Fault 码触发（`AIRY_FAULT_*`）通过 tracepoint 观测；错误码分布统计 |
| **A-ULP** | **核心** | A-ULP 日志观测——Ring Buffer 写入速率、Logger Daemon 消费延迟、eventfd 通知频率、Panic 回退到 `printk_safe` 的状态；128B 固定记录格式的写入观测 |
| **A-UCS** | 辅助 | A-UCS 的 sysctl/JSON 热重载事件通过 user_events 上报；配置版本号 `AIRY_CONFIG_VERSION` 校验结果观测 |
| **A-ULS** | **核心** | 调度器状态观测——sched_tac 三层调度类切换事件、Agent 8 态生命周期迁移、Micro-Supervisor 冷酷执法事件、Macro-Supervisor 温情裁决事件 |
| **A-IPC** | 辅助 | A-IPC 的 IPC fastpath 入口/出口 tracepoint、Ring 生命周期解耦事件、离线缓存校验命中率、Reconciliation 同步事件 |

### 4.1 Unify Design 权威源引用

- A-ULP 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §5
- A-ULS 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §7
- A-ULP Ring Buffer 数据流：[../40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)
- Logger Daemon 设计：[../40-dataflows/06-logger-daemon-design.md](../40-dataflows/06-logger-daemon-design.md)
- Panic 生存路径：[../40-dataflows/07-panic-survival-path.md](../40-dataflows/07-panic-survival-path.md)

---

## 5. 相关文档

- [AirymaxOS 总览](../README.md)：v1.0 技术选型声明 + 20 子目录索引
- [架构设计层](../10-architecture/README.md)：Unify Design 总纲 + IRON-9 v3 四层模型
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md)：Airymax Unify Design 总纲（SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（SSoT）
- [10-architecture/11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)：[DSL] 降级生存层（Panic 回退观测依据）
- [20-modules/01-kernel.md](../20-modules/01-kernel.md)：kernel 子仓可观测性
- [20-modules/12-logger-daemon-module.md](../20-modules/12-logger-daemon-module.md)：Logger Daemon 模块
- [20-modules/13-printk-bridge.md](../20-modules/13-printk-bridge.md)：printk-bridge 模块
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)：A-ULP [SC] log_types.h 契约
- [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)：A-ULP 零拷贝 Ring Buffer 数据流
- [40-dataflows/07-panic-survival-path.md](../40-dataflows/07-panic-survival-path.md)：Panic 生存路径
- [100-operations/README.md](../100-operations/README.md)：运维体系（监控告警）
- [170-performance/README.md](../170-performance/README.md)：性能工程（性能剖析）

---

## 6. 参考材料

- Linux 6.6 `kernel/trace/`（ftrace 实现）
- Linux 6.6 `kernel/bpf/`（eBPF 实现，仅观测用途）
- Linux 6.6 `tools/perf/`（perf 工具）
- Linux 6.6 `Documentation/trace/`（跟踪文档）
- Linux 6.6 `Documentation/dev-tools/`（开发工具）
- Linux 6.6 `include/trace/events/`（tracepoint 事件定义参考）

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，README + 01 + 02 文档奠基，确立 ftrace/eBPF 核心机制 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap / IRON-9 v3 四层模型五大技术选型声明（eBPF 仅观测非安全钩子）；新增 Airymax Unify Design 映射（A-ULP 日志观测 + A-ULS 调度器状态观测为核心）；文档索引对齐实际目录文件 |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | agentrt-linux 可观测性设计 v1.0 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
