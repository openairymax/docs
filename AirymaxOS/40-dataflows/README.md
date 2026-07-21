Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）数据流程设计
> **文档定位**：agentrt-linux（AirymaxOS）数据流程设计层的总览与索引，覆盖认知循环、记忆卷载、IPC 消息、调度、日志 Ring Buffer、Logger Daemon、Panic 生存路径共 7 大数据流\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agentrt-linux 总览](../README.md)\
> **核心约束**：IRON-9 v3 同源代码共享——[SC] 共享契约层 10 个头文件落地于 `include/uapi/linux/airymax/`，[SS] 4 大数据流语义同源（认知循环/记忆卷载/IPC/调度），[IND] 各子仓驱动与运行时独立实现，[DSL] 降级生存层提供 [SC] 损坏时最小可运行子集（#ifdef AIRY_SC_FALLBACK）；安全为横切关注点，贯穿全部数据流

---

## 1. 模块概述

本文档是 `40-dataflows/` 目录的总览与索引，聚焦于「数据如何流动」，是「接口设计层（30-interfaces）」的动态补充。本文档体系以 Linux 6.6 为内核基线，围绕 8 子仓之间的数据流动路径，刻画 7 大核心数据流：

1. **认知循环数据流（Cognition Flow，A-UEF）**：用户意图 → 认知层 → 规划 → 执行 → 记忆 → 反馈，对应 System 1 快思考 + System 2 慢思考双系统协同，落地于 `cognition` 子仓（同源 agentrt coreloopthree）。
2. **记忆卷载数据流（Memory Flow）**：L1 原始 → L2 特征 → L3 结构 → L4 模式，四层递进 + 遗忘机制 + CXL/PMEM/MGLRU 多代 LRU（Linux 6.6）硬件协同（alloc_pages + mmap），落地于 `memory` 子仓（同源 agentrt memoryrovol + heapstore）。
3. **IPC 消息流（IPC Flow，A-IPC）**：进程 A → io_uring（IORING_OP_URING_CMD）零拷贝 → 进程 B，128B 定长消息头同源 agentrt AgentsIPC，落地于 `kernel`（同源 agentrt atoms/corekern IPC）。
4. **调度数据流（Scheduling Flow，A-ULS）**：任务提交 → sched_tac 调度策略（SCHED_DEADLINE/SCHED_FIFO/EEVDF）→ 执行，实现可插拔调度策略，落地于 `kernel`。
5. **日志 Ring Buffer 数据流（Ring Buffer Logging，A-ULP）**：内核 trace_printk/printk → Ring Buffer → 用户态消费，结构化 JSON 日志 + ANSI 颜色，落地于 `kernel` + `services`。
6. **Logger Daemon 数据流（Logger Daemon Design，A-ULP）**：Logger Daemon 消费 Ring Buffer → 结构化日志输出 → 多后端持久化，落地于 `services`（同源 agentrt daemons/logger_d）。
7. **Panic 生存数据流（Panic Survival Path，A-ULP + A-ULS）**：内核 Panic → printk-bridge 落盘日志 → Logger Daemon 持久化 → 重启恢复，落地于 `kernel` + `services`。

7 大数据流通过 `trace_id` 贯穿 OpenTelemetry 全链路追踪，满足 NFR-O-002 Tracing 覆盖率与 NFR-O-001 Metrics 完整性。

---

## 2. 技术选型声明

本目录的数据流设计以 agentrt-linux v1.0 五大技术选型为基线：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 数据流影响 |
|---|---------|---------|----------------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 SCHED_AGENT 宏） | `04-scheduling-flow.md` 调度数据流基于sched_tac；**AIRY_SCHED_AGENT 已废弃**：AIRY_SCHED_AGENT 术语已彻底废弃（2026-07-18 用户裁决），全部替换为 sched_tac 调度策略 + stc_* 策略枚举 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：io_uring 命令操作码零拷贝传输 | **不使用 page flipping** | `03-ipc-flow.md`（A-IPC）IPC 消息流基于 IORING_OP_URING_CMD，固定 buffer + registered ring 路径 |
| 3 | **安全钩子** | **纯 C LSM**：纯 C 实现 `airy_lsm`，通过 `security_hook_list` 注册 | **不使用 BPF LSM** | 全部数据流的安全检查点基于纯 C LSM，不依赖 BPF 验证器 |
| 4 | **内存分配** | **alloc_pages + mmap**：物理页分配后映射到用户态 | **不使用 DMA 一致性内存** | `02-memory-flow.md` 记忆卷载数据流基于 alloc_pages + mmap；`05-ring-buffer-logging.md` Ring Buffer 内存基于 alloc_pages |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] + [SS] + [IND] + [DSL] | （v2 三层模型升级，新增 [DSL] 降级生存层） | 4 大数据流语义同源（[SS]）；[SC] 头文件提供契约；[IND] 各子仓独立实现；[DSL] 降级生存块（#ifdef AIRY_SC_FALLBACK） |

### 2.1 AIRY_SCHED_AGENT 术语废弃声明

agentrt-linux v1.0 **彻底废弃** `AIRY_SCHED_AGENT` 术语（2026-07-18 用户最终裁决，遵循 Airymax 工程思想五维分析）：

- ~~`任务提交 → AIRY_SCHED_AGENT → EEVDF → 执行`~~ → **任务提交 → sched_tac 调度策略（SCHED_DEADLINE/SCHED_FIFO/EEVDF）→ 执行**
- ~~`EEVDF + sched_tac + AIRY_SCHED_AGENT + 策略可插拔`~~ → **EEVDF + sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF 原生调度类组合）+ 策略可插拔**
- ~~`AIRY_SCHED_AGENT（sched_tac 用户态调度器）`~~ → **sched_tac 调度策略（基于 SCHED_DEADLINE/SCHED_FIFO/EEVDF 原生调度类，非用户态调度器，非 sched_ext）**
- ~~`AIRY_SCHED_AGENT_NAME "airy_sched_agent"`~~ → **`AIRY_STC_POLICY_NAME "stc_agent"`**

**确认结论**：`AIRY_SCHED_AGENT` 术语已**彻底废弃**。统一的调度术语体系为：
- 框架名：`sched_tac`（SCHED_DEADLINE + SCHED_FIFO + EEVDF + seL4 MCS 映射）
- 缩写：`STC`（策略性调度）
- 策略枚举：`stc_realtime`/`stc_interactive`/`stc_agent`/`stc_batch`
- 策略字符串常量：`AIRY_STC_POLICY_NAME "stc_agent"`

---

## 3. 数据流分类矩阵

下表对 7 大数据流进行横向对比，明确涉及子仓、核心路径与关键非功能约束：

| 数据流 | 涉及子仓 | Unify 模块 | 核心路径 | 关键约束 |
|--------|---------|-----------|---------|---------|
| 认知循环 | cognition + memory + kernel | **A-UEF** | 用户意图 → 认知 → 规划 → 执行 → 记忆 → 反馈 | NFR-P-001 < 100ms |
| 记忆卷载 | memory + kernel | — | L1 原始 → L2 特征 → L3 结构 → L4 模式 | NFR-R-001 数据持久 |
| IPC 消息 | kernel + services | **A-IPC** | 进程 A → io_uring（IORING_OP_URING_CMD）→ 进程 B | NFR-P-002 > 100K msg/s |
| 调度 | kernel + cognition | **A-ULS** | 任务提交 → sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF）→ 执行 | NFR-P-001 < 100ms |
| 日志 Ring Buffer | kernel + services | **A-ULP** | 内核 trace_printk/printk → Ring Buffer → 用户态消费 | NFR-O-001 日志完整性 |
| Logger Daemon | services | **A-ULP** | Logger Daemon 消费 Ring Buffer → 结构化日志 → 多后端持久化 | NFR-O-004 结构化日志 |
| Panic 生存 | kernel + services | **A-ULP + A-ULS** | 内核 Panic → printk-bridge 落盘 → Logger Daemon 持久化 → 重启恢复 | NFR-R-005 故障恢复 |

**横向关系**：

- 7 大数据流并非孤立，而是通过 `trace_id` 与 `task_id` 形成跨流关联。例如认知循环产生的任务会触发调度数据流；调度结果会驱动 IPC 消息流；执行结果沉淀到记忆卷载数据流；全部数据流的日志通过 Ring Buffer + Logger Daemon 统一收集。
- 所有数据流共享同一套可观测性基础设施（Prometheus + OpenTelemetry + 结构化 JSON 日志），满足 NFR-O 系列。
- 所有数据流共享同一套安全控制点（capability + 纯 C LSM + 输入净化），满足 NFR-S 系列。

---

## 4. 数据流设计原则

数据流设计遵循《Airymax 架构设计原则》五维正交 24 原则中的 3 条核心：

### 4.1 S-1 反馈闭环原则

每条数据流必须形成闭环，避免「开环单向流动」导致的状态漂移：

- **认知循环**：执行结果反馈到认知层，触发增量规划器扩展任务 DAG（FR-045）。
- **记忆卷载**：L4 模式层与 L3 结构层的检索结果反馈到 L2 特征层与 L1 原始层，调整特征权重与衰减速率。
- **IPC 消息**：响应 CQE 回流到发送方，确认消息送达；超时未确认触发重试或补偿。
- **调度**：执行 Metrics 反馈到调度策略评估器，影响后续 EEVDF 虚拟截止时间计算。
- **日志 Ring Buffer**：用户态消费速率反馈到内核生产速率控制，避免 Ring Buffer 溢出。
- **Logger Daemon**：持久化失败反馈到日志降级策略，触发 Panic 生存路径。
- **Panic 生存**：重启恢复后反馈到 A-ULS 故障分析，触发根因定位与预防策略。

### 4.2 E-2 可观测性原则

每条数据流必须在关键节点埋点，输出 Metrics、Tracing、Logging 三类信号：

- **trace_id 贯穿**：所有数据流共享同一 `trace_id`，从用户请求入口贯穿到内核执行出口。
- **OpenTelemetry span**：每个数据流的关键步骤创建独立 span，span 之间通过 parent_span_id 形成调用链。
- **结构化日志**：所有日志包含 trace_id / task_id / 模块名 / 函数名 / 行号，控制台支持 ANSI 颜色（INFO=蓝 / WARN=黄 / ERROR=红 / FATAL=品红 / DEBUG=灰）。
- **Metrics 指标**：延迟、吞吐、错误率、队列深度等关键指标以 Prometheus 格式暴露。

### 4.3 E-3 资源确定性原则

每条数据流涉及的所有资源必须有明确的所有者与生命周期管理：

- **资源所有权**：每个 IPC 消息缓冲区、每条记忆记录、每个调度任务、每条日志记录都有唯一所有者。
- **RAII / cleanup**：资源释放通过 RAII（C++）或 `__attribute__((cleanup))`（C）自动化，禁止裸 free。
- **禁止隐式转移**：跨子仓资源传递必须通过显式 capability 授权。
- **资源确定性验证**：通过 ASan + TSan + 泄漏检测验证（NFR-R-006）。

---

## 5. 文档索引

本目录包含 7 个数据流设计文档，覆盖认知循环、记忆卷载、IPC 消息、调度、日志 Ring Buffer、Logger Daemon、Panic 生存路径：

| # | 文档 | 数据流 | Unify 模块 | 核心内容 | 版本 | 状态 |
|---|------|--------|-----------|---------|------|------|
| 1 | [01-cognition-flow.md](01-cognition-flow.md) | 认知循环 | **A-UEF** | System 1/2 双系统 + CoreLoopThree kthread + 增量规划 + 补偿事务 | v1.0 | 维护中 |
| 2 | [02-memory-flow.md](02-memory-flow.md) | 记忆卷载 | — | L1→L4 四层递进 + CXL 池化 + MGLRU 多代 LRU（alloc_pages + mmap）+ 遗忘机制 | v1.0 | 维护中 |
| 3 | [03-ipc-flow.md](03-ipc-flow.md) | IPC 消息 | **A-IPC** | io_uring（IORING_OP_URING_CMD）零拷贝 + 128B 消息头 + 5 种 payload + 跨节点 IPC | v1.0 | 维护中 |
| 4 | [04-scheduling-flow.md](04-scheduling-flow.md) | 调度 | **A-ULS** | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF）+ 策略可插拔（**AIRY_SCHED_AGENT 已废弃**） | v1.0 | 维护中 |
| 5 | [05-ring-buffer-logging.md](05-ring-buffer-logging.md) | 日志 Ring Buffer | **A-ULP** | 内核 trace_printk/printk → Ring Buffer → 用户态消费 + 结构化 JSON 日志 + ANSI 颜色 | v1.0 | 维护中 |
| 6 | [06-logger-daemon-design.md](06-logger-daemon-design.md) | Logger Daemon | **A-ULP** | Logger Daemon 消费 Ring Buffer + 结构化日志输出 + 多后端持久化 | v1.0 | 维护中 |
| 7 | [07-panic-survival-path.md](07-panic-survival-path.md) | Panic 生存 | **A-ULP + A-ULS** | 内核 Panic → printk-bridge 落盘 → Logger Daemon 持久化 → 重启恢复 | v1.0 | 维护中 |

每个文档均包含：

- 文件头（文档定位 / 版本 / 最后更新 / 父文档）
- 数据流概览
- Mermaid 图表（含节点样式定义）
- 数据流详细步骤（表格形式）
- 性能约束与可观测性
- 版权声明

---

## 6. Airymax Unify Design 映射

本目录承载 Airymax Unify Design 四个核心模块的数据流设计：

| Unify 模块 | 数据流文档 | 核心数据流路径 | 关键技术 |
|-----------|-----------|------------|---------|
| **A-UEF**（统一错误码与故障定义体系） | `01-cognition-flow.md` | 用户意图 → 认知 → 规划 → 执行 → 记忆 → 反馈 | CoreLoopThree kthread + System 1/2 双系统 + sched_tac 调度 |
| **A-IPC**（统一进程间通信体系） | `03-ipc-flow.md` | 进程 A → io_uring（IORING_OP_URING_CMD）→ 进程 B | IORING_OP_URING_CMD + 128B 消息头（magic 0x41524531 'ARE1'）+ 5 种 payload |
| **A-ULS**（统一生命周期管理） | `04-scheduling-flow.md` | 任务提交 → sched_tac 调度策略 → 执行 → Metrics 反馈 | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF）+ 调度器状态观测 |
| **A-ULP**（统一日志与打印系统） | `05-ring-buffer-logging.md` + `06-logger-daemon-design.md` + `07-panic-survival-path.md` | 内核日志 → Ring Buffer → Logger Daemon → 持久化；Panic → printk-bridge → 落盘 → 恢复 | Ring Buffer + Logger Daemon + printk-bridge + Panic 生存路径 |

---

## 7. 与 agentrt 数据流的关系

agentrt-linux 数据流设计与 agentrt 数据流保持「同源且部分代码共享（IRON-9 v3）」（参考 ADR-010 + IRON-9 v3）：

| agentrt-linux 数据流 | 同源 agentrt 模块 | 同源语义 | agentrt-linux 增强 | IRON-9 v3 层次 |
|---|---|---|---|---|
| 认知循环 | coreloopthree | 三层认知循环：认知→执行→记忆 | 内核态 kthread + System 1/2 双系统切换 | [SC] cognition_types.h + [SS] 循环语义 + [IND] kthread |
| 记忆卷载 | memoryrovol + heapstore | 四层递进 + 堆存储 | CXL 池化 + MGLRU 多代 LRU（Linux 6.6）+ alloc_pages + mmap | [SC] memory_types.h + [SS] 四层卷载 + [IND] 内核态 |
| IPC 消息 | atoms/corekern IPC | 128B 消息头 + 零拷贝 | io_uring（IORING_OP_URING_CMD）原生 + sched_tac 集成 | [SC] ipc.h + [SS] 128B 消息头 + [IND] io_uring |
| 调度 | atoms/corekern Task | 微核心调度 | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF 原生调度类组合） | [SC] sched.h + [SS] 调度语义 + [IND] 内核态 |
| 日志 Ring Buffer | daemons/logger_d | 日志消费 | 内核 Ring Buffer + trace_printk + alloc_pages | [SC] log_types.h + [SS] 日志语义 + [IND] 内核态 |
| Logger Daemon | daemons/logger_d | 日志守护进程 | 多后端持久化 + Panic 生存落盘 | [SS] 守护进程语义 + [IND] systemd 集成 |
| Panic 生存 | 无（新增） | — | printk-bridge + 落盘 + 恢复 | [IND] 独立实现 |

**同源红利**：

1. **协议同源**：128B IPC 消息头结构与 agentrt AgentsIPC 完全一致，跨系统互通无适配层。
2. **模型同源**：记忆卷载四层递进模型与 agentrt memoryrovol 一致，迁移无语义损失。
3. **循环同源**：CoreLoopThree 三层认知循环结构与 agentrt 一致，行为可对比。
4. **独立演进**：agentrt-linux 在内核态实现（kthread + 纯 C LSM + io_uring IORING_OP_URING_CMD），agentrt 在用户态实现，二者独立演进但保持语义对齐。

---

## 8. 数据流版本与演进

数据流设计随 agentrt-linux 版本迭代演进，遵循「契约稳定 + 实现可变」原则：

| 版本 | 数据流特征 | 关键能力 |
|---|---|---|
| 0.1.1 | 4 大数据流框架定义 + [SC]/[SS]/[IND] 三层标注 | 文档体系建立 |
| v1.0 | 7 大数据流完整定义 + [SC]/[SS]/[IND]/[DSL] 四层标注 + sched_tac 修正 | CoreLoopThree kthread + MemoryRovol + io_uring（IORING_OP_URING_CMD）IPC + sched_tac 调度 + Ring Buffer 日志 + Logger Daemon + Panic 生存 |

**演进约束**：

1. **协议稳定性**：128B IPC 消息头、L1→L4 记忆层模型、sched_tac 调度类约束（使用 SCHED_DEADLINE/SCHED_FIFO/EEVDF 原生调度类，**禁止定义 SCHED_AGENT 宏**）在主版本内保持稳定，跨版本变更需 ADR 评审。
2. **向后兼容**：新版本数据流必须兼容旧版本的消息格式与调度语义，保证滚动升级不影响运行任务。
3. **可观测性兼容**：Metrics 名称与 OpenTelemetry span 属性命名遵循向后兼容原则，新增字段不破坏旧消费者。
4. **安全不变性**：capability 检查点、纯 C LSM hook 位置、审计哈希链结构在所有版本中保持不变（NFR-S 系列）。

---

## 9. 相关文档

- [agentrt-linux 总览](../README.md)：v1.0 设计文档体系总览与技术选型声明
- [接口设计层](../30-interfaces/README.md)：syscall / A-IPC IPC / SDK / 编码规范
- [模块设计层](../20-modules/README.md)：8 子仓 + A-ULS/A-ULP/A-UCS 详细设计
- [架构设计层](../10-architecture/README.md)：系统架构 + Unify Design 总纲 + IRON-9 v3 + [DSL] 降级层
- [工程标准规范](../50-engineering-standards/README.md)：SSoT v2 + [SC] 类型桥接
- [功能需求分析](../00-requirements/02-functional-requirements.md)：FR-001 ~ FR-080
- [非功能需求分析](../00-requirements/03-non-functional-requirements.md)：NFR-P/S/R/C/O 系列
- [Airymax 架构设计原则](../../AirymaxRT/10-architecture/00-architectural-principles.md)：五维正交 24 原则

---

## 10. 版本历史

| 版本 | 日期 | 变更 |
|---|---|---|
| 0.1.1 | 2026-07-06 | 初始版本，定义 4 大数据流分类矩阵与索引 |
| 0.1.1 | 2026-07-13 | D5（vtime `u64` → `__s32` Q16.16 定点）+ D6（64 字节对齐表述澄清为 [SC] 8 字节核心视图 + [IND] 扩展视图） |
| v1.0 | 2026-07-17 | 升级为 v1.0：废弃 AIRY_SCHED_AGENT 术语，统一为 sched_tac 调度策略（SCHED_DEADLINE/SCHED_FIFO/EEVDF）+ stc_* 策略枚举；新增sched_tac 技术选型声明（不使用 sched_ext）、IORING_OP_URING_CMD（不使用 page flipping）、纯 C LSM（不使用 BPF LSM）、alloc_pages + mmap（不使用 DMA 一致性内存）、IRON-9 v3 四层模型（新增 [DSL] 降级生存层）；新增 A-ULP 数据流（`05-ring-buffer-logging.md` + `06-logger-daemon-design.md` + `07-panic-survival-path.md`）；数据流文档数 4 → 7；新增 Unify Design 五模块数据流映射（A-UEF/A-IPC/A-ULS/A-ULP） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | "From data intelligence emerges."
