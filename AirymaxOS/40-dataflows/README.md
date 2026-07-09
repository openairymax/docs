Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）数据流程设计

> **文档定位**: agentrt-linux（AirymaxOS）数据流程设计层的总览与索引
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-07
> **父文档**: [agentrt-linux 总览](../README.md)
> **核心约束**: IRON-9 v2 同源且部分代码共享——[SC] 共享契约层 6 个头文件（bpf_struct_ops.h/memory_types.h/security_types.h/cognition_types.h/sched.h/ipc.h）落地于 include/airymax/，[SS] 4 大数据流语义同源（认知循环/记忆卷载/IPC/调度），[IND] 各子仓驱动与运行时独立实现；安全为横切关注点，贯穿全部 4 大数据流

---

## 1. 数据流设计概览

agentrt-linux 数据流程设计层聚焦于「数据如何流动」，是「接口设计层（30-interfaces）」的动态补充。本文档体系以 Linux 6.6 为内核基线，围绕 8 子仓之间的数据流动路径，刻画 4 大核心数据流：

1. **认知循环数据流（Cognition Flow）**：用户意图 → 认知层 → 规划 → 执行 → 记忆 → 反馈，对应 System 1 快思考 + System 2 慢思考双系统协同，落地于 `airymaxos-cognition` 子仓（同源 agentrt coreloopthree）。
2. **记忆卷载数据流（Memory Flow）**：L1 原始 → L2 特征 → L3 结构 → L4 模式，四层递进 + 遗忘机制 + CXL/PMEM/MGLRU 多代 LRU（Linux 6.6）硬件协同，落地于 `airymaxos-memory` 子仓（同源 agentrt memoryrovol + heapstore）。
3. **IPC 消息流（IPC Flow）**：进程 A → io_uring 零拷贝 → 进程 B，128B 定长消息头同源 agentrt AgentsIPC，落地于 `airymaxos-kernel`（同源 agentrt atoms/corekern IPC）。
4. **调度数据流（Scheduling Flow）**：任务提交 → SCHED_AGENT → EEVDF → 执行，基于 sched_ext（agentrt-linux 内核增强，主线 6.12+）实现可插拔调度策略，落地于 `airymaxos-kernel`。

4 大数据流通过 `trace_id` 贯穿 OpenTelemetry 全链路追踪，满足 NFR-O-002 Tracing 覆盖率与 NFR-O-001 Metrics 完整性。

---

## 2. 数据流分类矩阵

下表对 4 大数据流进行横向对比，明确涉及子仓、核心路径与关键非功能约束：

| 数据流 | 涉及子仓 | 核心路径 | 关键约束 |
|--------|---------|---------|---------|
| 认知循环 | cognition + memory + kernel | 用户意图 → 认知 → 规划 → 执行 → 记忆 → 反馈 | NFR-P-001 < 100ms |
| 记忆卷载 | memory + kernel | L1 原始 → L2 特征 → L3 结构 → L4 模式 | NFR-R-001 数据持久 |
| IPC 消息 | kernel + services | 进程 A → io_uring → 进程 B | NFR-P-002 > 100K msg/s |
| 调度 | kernel + cognition | 任务提交 → SCHED_AGENT → EEVDF → 执行 | NFR-P-001 < 100ms |

**横向关系**：

- 4 大数据流并非孤立，而是通过 `trace_id` 与 `task_id` 形成跨流关联。例如认知循环产生的任务会触发调度数据流；调度结果会驱动 IPC 消息流；执行结果沉淀到记忆卷载数据流。
- 所有数据流共享同一套可观测性基础设施（Prometheus + OpenTelemetry + 结构化 JSON 日志），满足 NFR-O 系列。
- 所有数据流共享同一套安全控制点（capability + LSM + 输入净化），满足 NFR-S 系列。

---

## 3. 数据流设计原则

数据流设计遵循《Airymax 架构设计原则》五维正交 24 原则中的 3 条核心：

### 3.1 S-1 反馈闭环原则

每条数据流必须形成闭环，避免「开环单向流动」导致的状态漂移：

- **认知循环**：执行结果反馈到认知层，触发增量规划器扩展任务 DAG（FR-045）。
- **记忆卷载**：L4 模式层与 L3 结构层的检索结果反馈到 L2 特征层与 L1 原始层，调整特征权重与衰减速率。
- **IPC 消息**：响应 CQE 回流到发送方，确认消息送达；超时未确认触发重试或补偿。
- **调度**：执行 Metrics 反馈到调度策略评估器，影响后续 EEVDF 虚拟截止时间计算。

### 3.2 E-2 可观测性原则

每条数据流必须在关键节点埋点，输出 Metrics、Tracing、Logging 三类信号：

- **trace_id 贯穿**：所有数据流共享同一 `trace_id`，从用户请求入口贯穿到内核执行出口。
- **OpenTelemetry span**：每个数据流的关键步骤创建独立 span，span 之间通过 parent_span_id 形成调用链。
- **结构化日志**：所有日志包含 trace_id / task_id / 模块名 / 函数名 / 行号，控制台支持 ANSI 颜色（INFO=蓝 / WARN=黄 / ERROR=红 / FATAL=品红 / DEBUG=灰）。
- **Metrics 指标**：延迟、吞吐、错误率、队列深度等关键指标以 Prometheus 格式暴露。

### 3.3 E-3 资源确定性原则

每条数据流涉及的所有资源必须有明确的所有者与生命周期管理：

- **资源所有权**：每个 IPC 消息缓冲区、每条记忆记录、每个调度任务都有唯一所有者。
- **RAII / cleanup**：资源释放通过 RAII（C++）或 `__attribute__((cleanup))`（C）自动化，禁止裸 free。
- **禁止隐式转移**：跨子仓资源传递必须通过显式 capability 授权。
- **资源确定性验证**：通过 ASan + TSan + 泄漏检测验证（NFR-R-006）。

---

## 4. 本目录文档索引

| # | 文档 | 数据流 | 核心内容 |
|---|------|--------|---------|
| 1 | [01-cognition-flow.md](01-cognition-flow.md) | 认知循环 | System 1/2 双系统 + CoreLoopThree kthread + 增量规划 + 补偿事务 |
| 2 | [02-memory-flow.md](02-memory-flow.md) | 记忆卷载 | L1→L4 四层递进 + CXL 池化 + MGLRU 多代 LRU + 遗忘机制 |
| 3 | [03-ipc-flow.md](03-ipc-flow.md) | IPC 消息 | io_uring 零拷贝 + 128B 消息头 + 5 种 payload + 跨节点 IPC |
| 4 | [04-scheduling-flow.md](04-scheduling-flow.md) | 调度 | EEVDF + sched_ext + SCHED_AGENT + 策略可插拔 |

每个文档均包含：

- 文件头（文档定位 / 版本 / 最后更新 / 父文档）
- 数据流概览
- Mermaid 图表（含节点样式定义）
- 数据流详细步骤（表格形式）
- 性能约束与可观测性
- 版权声明

---

## 5. 与 agentrt 数据流的关系

agentrt-linux 数据流设计与 agentrt 数据流保持「同源且部分代码共享（IRON-9 v2）」（参考 ADR-010 + IRON-9）：

| agentrt-linux 数据流 | 同源 agentrt 模块 | 同源语义 | agentrt-linux 增强 |
|---|---|---|---|
| 认知循环 | coreloopthree | 三层认知循环：认知→执行→记忆 | 内核态 kthread + System 1/2 双系统切换 |
| 记忆卷载 | memoryrovol + heapstore | 四层递进 + 堆存储 | CXL 池化 + MGLRU 多代 LRU（Linux 6.6）+ 持久同调 |
| IPC 消息 | atoms/corekern IPC | 128B 消息头 + 零拷贝 | io_uring 原生 + sched_ext 集成 |
| 调度 | atoms/corekern Task | 微核心调度 | EEVDF + SCHED_AGENT（sched_ext BPF） |

**同源红利**：

1. **协议同源**：128B IPC 消息头结构与 agentrt AgentsIPC 完全一致，跨系统互通无适配层。
2. **模型同源**：记忆卷载四层递进模型与 agentrt memoryrovol 一致，迁移无语义损失。
3. **循环同源**：CoreLoopThree 三层认知循环结构与 agentrt 一致，行为可对比。
4. **独立演进**：agentrt-linux 在内核态实现（kthread + BPF + io_uring），agentrt 在用户态实现，二者独立演进但保持语义对齐。

**与接口设计层的关系**：本文档体系是 [接口设计层](../30-interfaces/README.md) 的动态补充。接口设计层定义「调用契约（What）」，数据流程设计层定义「流动路径（How）」，二者共同构成完整的接口规约。

**与模块设计层的关系**：本文档体系横向贯穿 [模块设计层](../20-modules/README.md) 的 8 子仓，纵向贯穿内核态 / 用户态边界，是子仓间协作的动态规约。

---

## 6. 数据流版本与演进

数据流设计随 agentrt-linux 版本迭代演进，遵循「契约稳定 + 实现可变」原则：

| 版本 | 数据流特征 | 关键能力 |
|---|---|---|
| 0.1.1（文档体系完成） | 4 大数据流框架定义 + [SC]/[SS]/[IND] 三层标注 | 文档体系建立 |
| 1.0.1（开发） | 4 大数据流完整实现 | CoreLoopThree kthread + MemoryRovol + io_uring IPC + SCHED_AGENT |

**演进约束**：

1. **协议稳定性**：128B IPC 消息头、L1→L4 记忆层模型、SCHED_EXT 调度类编号（复用内核 SCHED_EXT=7，禁止 SCHED_AGENT 宏）在主版本内保持稳定，跨版本变更需 ADR 评审。
2. **向后兼容**：新版本数据流必须兼容旧版本的消息格式与调度语义，保证滚动升级不影响运行任务。
3. **可观测性兼容**：Metrics 名称与 OpenTelemetry span 属性命名遵循向后兼容原则，新增字段不破坏旧消费者。
4. **安全不变性**：capability 检查点、LSM hook 位置、审计哈希链结构在所有版本中保持不变（NFR-S 系列）。

---

## 7. 相关文档

- [agentrt-linux 总览](../README.md)：5 层文档体系与 8 子仓清单
- [接口设计层](../30-interfaces/README.md)：syscall / IPC / SDK / 编码规范
- [模块设计层](../20-modules/README.md)：8 子仓详细设计
- [架构设计层](../10-architecture/README.md)：三大支柱与五维原则
- [功能需求分析](../00-requirements/02-functional-requirements.md)：FR-001 ~ FR-080
- [非功能需求分析](../00-requirements/03-non-functional-requirements.md)：NFR-P/S/R/C/O 系列
- [Airymax 架构设计原则](../../AirymaxRT/00-architectural-principles.md)：五维正交 24 原则

---

## 8. 文档变更记录

| 版本 | 日期 | 变更内容 | 变更人 |
|---|---|---|---|
| 0.1.1 | 2026-07-06 | 初始版本，定义 4 大数据流分类矩阵与索引 | Airymax 架构委员会 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
