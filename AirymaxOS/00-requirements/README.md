Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 需求分析
> **文档定位**：agentrt-linux（AirymaxOS）需求分析体系的总入口与纲领性文档，定义需求分层模型、需求来源、需求追溯关系，并向下展开为业务需求、功能需求、非功能需求三个子文档。\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **正式全称**：agentrt-linux（AirymaxOS，极境智能体操作系统，简称 AirymaxOS）\
> **仓库别名**：agentrt-linux（仓库名）\
> **上级文档**：[agentrt-linux 总览](../README.md)

---

## 1. 模块概述

本文档（`00-requirements/README.md`）是 agentrt-linux 需求分析体系的**顶层纲领**，承担三项核心职责：

1. **统一需求语言**：为 agentrt-linux 8 个子仓（kernel / services / security / memory / cognition / cloudnative / system / tests-linux）提供一致的需求分类、编号与追溯框架。
2. **定义需求分层**：将 agentrt-linux 的全部需求自顶向下分解为「业务需求 → 功能需求 → 非功能需求」三层，确保从用户场景到工程指标的可追溯性。
3. **链接下游设计**：作为需求层与架构设计层、模块设计层、接口设计层、数据流程设计层之间的桥梁，建立"需求 → 设计 → 实现 → 验证"的完整闭环。

需求层覆盖 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的需求维度：业务需求刻画五模块面向的用户场景，功能需求定义五模块的具体能力，非功能需求约束五模块的质量属性。

---

## 2. 技术选型声明

本目录的需求条目以 agentrt-linux v1.0 五大技术选型为基线，所有需求均不得偏离以下选型：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 对需求的影响 |
|---|---------|---------|----------------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 SCHED_AGENT 宏） | FR-001 内核调度需求以sched_tac 为准，NFR-P-001 调度延迟约束基于 SCHED_DEADLINE 保障 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：io_uring 命令操作码零拷贝传输 | **不使用 page flipping** | FR-IPC 系列需求以 IORING_OP_URING_CMD 为传输基线，NFR-P-002 吞吐约束基于固定 buffer + registered ring |
| 3 | **安全钩子** | **纯 C LSM**：纯 C 实现 `airy_lsm`，通过 `security_hook_list` 注册 | **不使用 BPF LSM** | NFR-S 系列安全需求以纯 C LSM 可审计性为约束，禁止依赖 BPF 验证器语义 |
| 4 | **内存分配** | **alloc_pages + mmap**：物理页分配后映射到用户态 | **不使用 DMA 一致性内存** | FR-memory 系列需求以 alloc_pages + mmap 跨架构一致语义为基线 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] + [SS] + [IND] + [DSL] | （v2 三层模型升级，新增 [DSL] 降级生存层） | 跨项目互操作需求以 IRON-9 v3 [DSL] 降级生存层为保证 |

---

## 3. 文档索引

本目录包含 3 个需求子文档（业务/功能/非功能），每个文档对应一层需求：

| # | 文档 | 关注维度 | 编号前缀 | 核心问题 | 版本 | 状态 |
|---|------|---------|---------|---------|------|------|
| 1 | [01-business-requirements.md](01-business-requirements.md) | 用户与生态维度 | BR- | agentrt-linux 为谁解决什么问题？ | v1.0 | 维护中 |
| 2 | [02-functional-requirements.md](02-functional-requirements.md) | 能力与行为维度 | FR- | agentrt-linux 提供哪些具体能力？ | v1.0 | 维护中 |
| 3 | [03-non-functional-requirements.md](03-non-functional-requirements.md) | 质量属性维度 | NFR- | agentrt-linux 的能力要做到多好？ | v1.0 | 维护中 |

---

## 4. 需求分层模型

agentrt-linux 采用经典的**三层需求分层模型**，借鉴 IEEE 830 与 ISO/IEC 25010 的需求工程思想，并结合 Airymax 的「五维正交 24 原则」中的「S-2 层次分解原则」与「E-8 可测试性原则」进行适配。

### 4.1 三层需求定义

```
+=====================================================================+
|                  第一层：业务需求（Business Requirements）            |
|   来源：Agent 工作负载 / AI 原生 / 云原生 / Linux 企业级生态           |
|   关注：agentrt-linux 为谁解决什么问题，带来什么价值                       |
|   编号：BR-001 ~ BR-0XX                                              |
|   验证：用户场景验收测试（UAT）                                       |
+=====================================================================+
                              | 业务需求推导出
                              v
+=====================================================================+
|                  第二层：功能需求（Functional Requirements）           |
|   来源：8 子仓职责 / agentrt 同源模块 / Unify Design 五模块              |
|   关注：agentrt-linux 提供哪些具体功能能力（输入 → 处理 → 输出）          |
|   编号：FR-001 ~ FR-0XX                                              |
|   验证：单元测试 + 集成测试 + 契约测试                                |
+=====================================================================+
                              | 功能需求受约束于
                              v
+=====================================================================+
|              第三层：非功能需求（Non-Functional Requirements）         |
|   来源：性能 / 安全 / 可靠性 / 兼容性 / 可观测性                      |
|   关注：功能需求的质量属性约束（延迟、吞吐、安全、可靠性）            |
|   编号：NFR-P/S/R/C/O-001 ~ NFR-XXX                                  |
|   验证：性能基准测试 + Soak Test + 形式化验证 + 渗透测试              |
+=====================================================================+
```

### 4.2 分层模型的核心特性

| 特性 | 说明 | 对应设计原则 |
| -------- | ------------------------------------ | --------- |
| **可追溯性** | 每条功能需求可上溯到至少一条业务需求，每条非功能需求约束至少一条功能需求 | E-6 错误可追溯 |
| **正交性** | 三层需求相互独立，业务变更不影响功能编号，质量约束独立于业务逻辑 | 五维正交体系 |
| **可测试性** | 每条需求必须有明确的验收标准与验证方法 | E-8 可测试性 |
| **可演进性** | 需求条目通过版本号管理，变更需通过 ADR 评审 | K-2 接口契约化 |
| **可观测性** | 需求的执行状态通过 Metrics 持续监控 | E-2 可观测性 |

---

## 5. 需求来源

agentrt-linux 的需求来源遵循**四源汇聚**原则：

### 5.1 来源一：agentrt 同源（设计同源性）

| 同源维度 | agentrt 侧 | agentrt-linux 侧 | 同源红利 |
| ------ | --------- | --------------- | --------- |
| 调度语义 | MicroCoreRT 调度器 | sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF） | 架构同源，无适配层 |
| IPC 协议 | AgentsIPC（128B 消息头） | A-IPC（io_uring IORING_OP_URING_CMD 零拷贝） | 协议同源，低延迟 |
| 安全模型 | Cupolas 安全穹顶 | capability + 纯 C LSM（seL4 风格） | 模型同源，安全内生 |
| 记忆模型 | MemoryRovol 四层卷载 | 记忆子系统（L1~L4，alloc_pages + mmap） | 记忆同源，知识复用 |
| 认知循环 | CoreLoopThree 三层循环 | A-UEF（认知 kthread） | 循环同源，反馈闭环 |

### 5.2 来源二：Linux 企业级生态标准规范

agentrt-linux 全面参考 Linux 企业级生态标准，对齐其模块设计与技术规格。内核基线锁定 Linux 6.6（1.x.x 系列），2.x.x 升级至 Linux 7.1（ADR-013）。

### 5.3 来源三：微内核设计思想

**微内核设计思想唯一来源：seL4**（ADR-014，2026-07-09 确立）。agentrt-linux 的微内核核心思想仅来源于 seL4，不引入 Zircon / Minix3 / 其他微内核架构。

### 5.4 来源四：Agent 工作负载

agentrt-linux 作为通用 AI Agent 运行底座，面向四类核心工作负载模式：长序列知识密集型、高并发交互型、实时控制型、多模态感知型。

---

## 6. Airymax Unify Design 映射

需求层覆盖 Airymax Unify Design 五模块的需求维度：

| Unify 模块 | 业务需求（BR） | 功能需求（FR） | 非功能需求（NFR） |
|-----------|--------------|--------------|----------------|
| **A-UEF**（统一错误码与故障定义体系） | BR-Agent 工作负载需求、BR-AI 原生认知循环 | FR-认知循环 CoreLoopThree、FR-调度策略、FR-Wasm 沙箱 | NFR-P-001 调度延迟、NFR-P-005 认知吞吐 |
| **A-ULP**（统一日志与打印系统） | BR-可观测性需求、BR-Panic 生存需求 | FR-Ring Buffer 日志、FR-Logger Daemon、FR-printk-bridge、FR-Panic 生存路径 | NFR-R-001 数据持久、NFR-O-001 日志完整性 |
| **A-UCS**（统一配置管理体系） | BR-配置管理需求 | FR-统一配置管理体系、FR-sysctl、FR-Kconfig、FR-airy_defconfig | NFR-C-002 配置兼容性 |
| **A-ULS**（统一生命周期管理） | BR-可靠性需求、BR-故障恢复需求 | FR-内核 Agent 监管、FR-用户态监管守护进程、FR-设备生命周期、FR-调度器状态监管 | NFR-R-003 故障隔离、NFR-R-005 熔断 |
| **A-IPC**（统一进程间通信体系） | BR-高并发交互需求 | FR-io_uring 零拷贝 IPC、FR-128B 消息头、FR-设备 DMA、FR-IORING_OP_URING_CMD | NFR-P-002 IPC 吞吐 > 100K msg/s |

---

## 7. 需求编号规则

agentrt-linux 需求采用统一的「前缀-序号」编号规则：

| 需求层次 | 编号前缀 | 编号示例 | 数量预估 |
| -------- | ------ | ------------------- | ------ |
| 业务需求 | BR- | BR-001、BR-002 | 约 20 条 |
| 功能需求 | FR- | FR-001、FR-052 | 约 80 条 |
| 非功能需求-性能 | NFR-P- | NFR-P-001、NFR-P-005 | 约 10 条 |
| 非功能需求-安全 | NFR-S- | NFR-S-001、NFR-S-008 | 约 15 条 |
| 非功能需求-可靠性 | NFR-R- | NFR-R-001、NFR-R-006 | 约 10 条 |
| 非功能需求-兼容性 | NFR-C- | NFR-C-001、NFR-C-006 | 约 8 条 |
| 非功能需求-可观测性 | NFR-O- | NFR-O-001、NFR-O-005 | 约 8 条 |

---

## 8. 需求验证体系

每条需求必须有明确的验证方法，对应设计原则「E-8 可测试性原则」：

| 需求层次 | 验证方法 | 验证工具 | 责任子仓 |
| -------- | -------- | -------- | -------- |
| 业务需求 | 用户场景验收测试（UAT） | Python + pytest | tests-linux |
| 功能需求 | 单元测试 + 集成测试 + 契约测试 | CUnit + CMock + 自定义框架 | tests-linux |
| 非功能需求-性能 | 性能基准测试 | Locust + k6 + perf | tests-linux |
| 非功能需求-安全 | 渗透测试 + 形式化验证 | seL4 风格验证 + 静态分析 | tests-linux + security |
| 非功能需求-可靠性 | Soak Test + 混沌工程 | Chaos Mesh + 系统级测试套件 | tests-linux |
| 非功能需求-兼容性 | 兼容性测试矩阵 | 集成测试框架 | tests-linux + system |
| 非功能需求-可观测性 | 可观测性覆盖检查 | Prometheus + OpenTelemetry | tests-linux |

---

## 9. 相关文档

### 9.1 需求分析子文档

- [业务需求分析](01-business-requirements.md)：Agent 工作负载、AI 原生、云原生、Linux 企业级生态对齐
- [功能需求分析](02-functional-requirements.md)：8 子仓功能矩阵、Unify Design 五模块能力清单
- [非功能需求分析](03-non-functional-requirements.md)：性能、安全、可靠性、兼容性、可观测性

### 9.2 上游参考文档

- [agentrt-linux 总览](../README.md)：agentrt-linux v1.0 整体设计与技术选型声明
- [Airymax 架构设计原则](../../AirymaxRT/10-architecture/00-architectural-principles.md)：五维正交 24 原则

### 9.3 下游设计文档

- [架构设计](../10-architecture/README.md)：基于需求推导的架构骨架（含 Unify Design 总纲）
- [模块设计](../20-modules/README.md)：8 子仓 + A-ULS/A-ULP/A-UCS 模块详细设计
- [接口设计](../30-interfaces/README.md)：系统调用与 SDK 接口契约
- [数据流程设计](../40-dataflows/README.md)：A-UEF/A-IPC/A-ULS/A-ULP 数据流路径

---

## 10. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-06 | 初始版本，建立需求分层模型与追溯框架 |
| 0.1.1 | 2026-07-13 | 补充版本基线锁定战略决策说明 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）、IORING_OP_URING_CMD（不使用 page flipping）、纯 C LSM（不使用 BPF LSM）、alloc_pages + mmap（不使用 DMA 一致性内存）、IRON-9 v3 四层模型；新增 Airymax Unify Design 五模块需求映射（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | agentrt-linux v1.0 需求分析 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
