Copyright (c) 2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# OpenAirymax AgentOS — 统一术语表

**最新**: 2026-06-10
**状态**: 维护中
**路径**: `Docs/TERMINOLOGY.md`

**作者**: Liren Wang、Zhixian Zhou — AgentOS 架构委员会

---

## 一、编制说明

本术语表是 OpenAirymax AgentOS 规范体系的**唯一权威术语参考**。

**目的**:
- 统一所有规范文档、代码注释、API 文档中的术语定义
- 当不同文档间存在术语冲突时，**以本表为准**

**使用规则**:
1. 编写文档时必须使用本表的**标准名称**
2. 标注为"禁止使用"的旧称不得出现在任何新的规范文档或代码注释中
3. 新增术语需经架构委员会审核后加入

---

## 二、核心架构术语

### Agent / 智能体

**定义**: 具有认知能力的实体，能够感知环境、进行推理决策并执行行动。Agent 是 AgentOS 的基本执行单元。

**系统内代码**: `agentos_sys_agent_*`

**参见**: Skill（技能）、Contract（契约）

---

### AirymaxSyscall / 系统调用层

**定义**: 用户态程序请求内核服务的标准接口，覆盖 Agent、Task、Skill、Session、Sandbox、Memory、Telemetry 七大功能域。遵循机制与策略分离原则。

**标准名称**: 系统调用层 (AirymaxSyscall)

**系统内代码**: `agentos_sys_*`

**参见**: MicroCoreRT（微核心）、IPC

---

### CognitiveEvolution / 认知进化系统

**定义**: CoreLoopThree 认知层的自适应能力提升系统。通过经验记录→模式提取→策略进化→知识迁移的闭环，实现感知→反应→学习→推理→创造的五级认知层级跃迁。

**系统内代码**: `cog_*` / `agentos_cog_*`

**参见**: Thinkdual（双思考系统）、CoreLoopThree（三层认知循环）、MAC（多智能体协作框架）

---

### Contract / 契约

**定义**: 机器可读的能力描述文件，定义 Agent 或 Skill 的接口、能力、权限等元数据。采用 JSON Schema 格式。

**参见**: Input Schema、Output Schema

---

### CoreKern / 原子核心

**定义**: AgentOS 微核心中不可再分的基础内核实现，承载 IPC、内存管理、任务调度、时间服务、OOM 处理、可观测性等原子机制。是体系并行论中基础体 (Base Body) 的具象实现。

**标准名称**: 原子核心 (CoreKern)

**系统内代码**: `agentos_core_*`

**代码目录**: `AgentOS/agentos/atoms/corekern/`

**参见**: MicroCoreRT（微核心）、IPC、AirymaxSyscall（系统调用层）

---

### CoreLoopThree / 三层认知循环

**定义**: AgentOS 的三层认知-执行-记忆主循环运行时：

| 层级 | 名称 | 职责 |
|------|------|------|
| L1 认知层 | Cognition Engine | 意图解析 → 规划 → 分发 → 协调 → 批判 |
| L2 执行层 | Execution Engine | 任务执行 → 单元调度 → 补偿事务 |
| L3 记忆层 | Memory Engine | 结果写入 → 查询 → 挂载 |

认知层实现双思考系统（Thinkdual）的 t2/t1-f/t1-p 三组件认知架构。

**标准名称**: 三层认知循环 (CoreLoopThree)

**旧称/禁止使用**: "第三循环内核"、"三层循环运行时"、"三层一体架构"、"三层循环核心"

**系统内代码**: `agentos_loop_*`

**代码目录**: `AgentOS/agentos/atoms/coreloopthree/`

**参见**: Thinkdual（双思考系统）、MemoryRovol（记忆卷载）、TaskFlow（任务流引擎）

---

### Cupolas / 安全穹顶

**定义**: AgentOS 的内生安全防护层，采用输入净化→权限仲裁→网络过滤→审计记录四重防护链，包含 Guards、Permission、Sanitizer、Audit、Workbench、Security Vault、Network Security 七个子系统。

**标准名称**: 安全穹顶 (Cupolas)

**系统内代码**: `cupolas_*`

**参见**: Security by Design（安全内生设计）、Sandbox（沙箱）、Service Isolation（服务隔离）

---

### Daemon / 用户态服务层

**定义**: AgentOS 的用户态服务集群。共 10 个独立服务进程（gateway_d、llm_d、tool_d、market_d、sched_d、monit_d、channel_d、observe_d、notify_d、info_d），以 `_d` 后缀命名，通过 IPC 与微内核通信，通过 JSON-RPC 2.0 对外提供服务。

**标准名称**: 用户态服务层 (Daemon)

**旧称/禁止使用**: "守护进程"、"后端服务层"、"daemon进程"
> 严禁使用"守护进程"——该词在 AgentOS 语境中会与操作系统传统守护进程概念混淆。

**代码目录**: `AgentOS/agentos/daemon/`

**参见**: MicroCoreRT（微核心）、IPC、Protocols（协议适配层）

---

### ErrorCodeSystem / 统一错误码体系

**定义**: AgentOS 采用双错误码体系：

| 体系 | 适用场景 | 格式 |
|------|---------|------|
| **C 负整数体系**（首要） | C 内核和 daemon 层 | `AGENTOS_SUCCESS=0`、`AGENTOS_EINVAL=-2` |
| **SDK 十六进制体系**（次要） | SDK 和外部接口 | `0x0000`-`0x7FFF` 分段 |

**系统内代码**: `AGENTOS_E*` / `AGENTOS_ERR_*`

**禁止**: C 内核代码中使用十六进制错误码；SDK 中使用负整数错误码

**参见**: JSON-RPC 2.0

---

### Forgetting Engine / 遗忘机制

**定义**: 基于艾宾浩斯遗忘曲线驱动的记忆衰减与清理机制。遗忘公式: R(t) = e^(-t/τ)，τ 默认 7 天。支持 NONE/EBBINGHAUS/LINEAR/ACCESS_BASED 四种遗忘策略。

**系统内代码**: `agentos_forgetting_*`

**代码目录**: `MemoryRovol/src/forgetting/`

**参见**: MemoryRovol（记忆卷载）、MemorySwap（记忆交换算法）

---

### HeapStore / 数据分区存储

**定义**: AgentOS 的结构化数据分区持久层，支持日志、追踪、会话、Agent、Skill、内存、Token、IPC 等多数据分区的批量写入与熔断保护。

**系统内代码**: `agentos_heapstore_*`

**代码目录**: `AgentOS/agentos/heapstore/`

---

### IPC / 进程间通信

**定义**: 微核心提供的进程间通信机制。支持 Request-Response（同步）、Publish-Subscribe（异步）、Streaming（长连接）三种通信模式。

**标准名称**: 进程间通信 (IPC)

**系统内代码**: `agentos_ipc_*`

**代码目录**: `AgentOS/agentos/atoms/corekern/src/ipc/`

**参见**: MicroCoreRT（微核心）、AirymaxSyscall（系统调用层）、Service Isolation（服务隔离）

---

### JSON-RPC 2.0

**定义**: AgentOS 采用的轻量级远程过程调用协议，支持请求-响应和通知两种模式。所有 Daemon 服务通过 JSON-RPC 2.0 对外暴露方法。

**协议层错误码**: -32700 (Parse error)、-32600 (Invalid Request)、-32601 (Method not found)、-32602 (Invalid params)、-32603 (Internal error)

**参见**: ErrorCodeSystem（统一错误码体系）

---

### MAC / 多智能体协作框架

**定义**: CoreLoopThree 认知层的多智能体协作基础设施，支持独立、协作、共识、委托四种协作模式，以及多数投票、全票通过、加权投票、领导者否决四种共识策略。

**系统内代码**: `mac_*` / `agentos_mac_*`

**旧称/禁止使用**: "Multi-Agent System"（过于泛化）

**参见**: Thinkdual（双思考系统）、CognitiveEvolution（认知进化系统）

---

### Memory Safety Macro System / 安全宏体系

**定义**: AgentOS 的内存操作安全宏体系，覆盖分配、释放、字符串操作和禁止函数，确保内存操作的类型安全和空指针安全。

**核心宏**:
- 分配: `AGENTOS_MALLOC` / `AGENTOS_CALLOC` / `AGENTOS_REALLOC`
- 释放: `AGENTOS_FREE` / `MEMORY_FREE_SAFE`
- 字符串: `AGENTOS_STRDUP` / `AGENTOS_STRNCPY_TERM`
- 拷贝: `AGENTOS_MEMCPY_SAFE`
- 清除: `AGENTOS_SEC_CLEAR`

**禁止函数**: `malloc`、`free`、`printf`、`strcpy`、`strcat`、`sprintf`、`gets` 等（完整列表见 `banned_functions.h`）

**参见**: MEMORY_FREE_SAFE、Security by Design（安全内生设计）

---

### MemorySwap / 记忆交换算法

**定义**: 记忆卷载的存储层级交换机制。借鉴操作系统虚拟内存分页交换，将记忆按访问热度划分为热/温/冷三个存储层级，在不同层级之间自动迁移。

**标准名称**: 记忆交换算法（正式）/ MemorySwap（简称）

**系统内代码**: `agentos_memory_swap_manager_t`

**参见**: MemoryRovol（记忆卷载）、Forgetting Engine（遗忘机制）

---

### MemoryRovol / 记忆卷载

**定义**: AgentOS 的四层渐进式记忆架构，从原始数据到深层模式的完整处理管线：

| 层级 | 名称 | 功能 |
|------|------|------|
| L1 原始卷 | Raw Storage | SHA-256 哈希 + ZSTD 压缩，append-only 持久化 |
| L2 特征层 | Feature Extraction | HNSW 向量检索 + BM25 文本检索 + 混合融合 |
| L3 结构层 | Structure Binding | 知识图谱 + VSA 绑定/解绑算子 |
| L4 模式层 | Pattern Mining | HDBSCAN 聚类 + 0D 持久同调 + 模式挖掘 |

**名称由来**: Memory + Rovol（Roll + Volume 的融合造词），寓意记忆的滚动卷载。

**标准名称**: 记忆卷载 (MemoryRovol)

**旧称/禁止使用**: "记忆漩涡引擎"

**系统内代码**: `agentos_layer*_*` / `agentos_memoryrov_*`

**参见**: CoreLoopThree（三层认知循环）、Forgetting Engine（遗忘机制）、TimeSliceInfer（分时推理框架）、MemorySwap（记忆交换算法）

---

### MEMORY_FREE_SAFE

**定义**: 安全释放宏。释放内存后将指针置 NULL，防止悬垂指针。

**签名**: `MEMORY_FREE_SAFE(ptr_to_ptr)` — 参数为指向待释放指针的指针

**参见**: Memory Safety Macro System（安全宏体系）

---

### MicroCoreRT / 微核心

**定义**: 极简的操作系统内核抽象层，只提供最基本的 IPC、内存管理、任务调度、时间管理、可观测性功能。其他功能作为用户态服务实现。名称取自 Micro + Core + RT（Runtime）。

> 注: 这是"微内核风格"的用户态抽象层，不是真正的操作系统内核（不运行在 ring 0，不管理硬件）。

**标准名称**: 微核心 (MicroCoreRT)

**旧称/禁止使用**: "微内核"、"Microkernel"（不加限定词时易与真正 OS 内核混淆）

**系统内代码**: `agentos_core_*`

**代码目录**: `AgentOS/agentos/atoms/corekern/`（与 CoreKern 同目录）

**参见**: CoreKern（原子核心）、IPC、AirymaxSyscall（系统调用层）、Service Isolation（服务隔离）

---

### Protocols / 协议适配层

**定义**: AgentOS 的多协议适配层，提供统一的协议抽象、路由、转换和扩展框架。支持 9 种协议类型：JSON-RPC 2.0、MCP v1、A2A v0.3、OpenAI、Claude、OpenJiuwen、OpenClaw、ChinaEco、AGNTCY ACP。

**标准名称**: 协议适配层 (Protocols)

**代码目录**: `AgentOS/agentos/protocols/`

**参见**: Daemon（用户态服务层）

---

### Sandbox / 沙箱

**定义**: 隔离的执行环境，限制 Skill 的权限和资源访问。支持进程隔离、容器隔离、WASM 隔离三种模式。

**标准名称**: 沙箱 (Sandbox)

**系统内代码**: `agentos_sys_sandbox_*`、`cupolas_workbench_*`

**参见**: Service Isolation（服务隔离）、Permission（权限）、Cupolas（安全穹顶）

---

### Security by Design / 安全内生设计

**定义**: 将安全机制内嵌于系统架构和设计中的理念，而非事后附加补丁。Cupolas 安全穹顶是该理念的具象实现。

**参见**: Cupolas（安全穹顶）、Sandbox（沙箱）

---

### Service Isolation / 服务隔离

**定义**: 用户态服务之间相互隔离，只能通过内核 IPC 通信。一个服务的崩溃不会影响其他服务。

**参见**: MicroCoreRT（微核心）、IPC

---

### Skill / 技能

**定义**: 可复用的执行单元，为智能体提供文件操作、网络请求、数据分析、浏览器自动化等具体能力。每个 Skill 通过 Contract 定义其接口和能力。

**标准名称**: 技能 (Skill)

**系统内代码**: `agentos_sys_skill_*`

**参见**: Agent（智能体）、Contract（契约）

---

### TaskFlow / 任务流引擎

**定义**: 基于 Pregel BSP（Bulk Synchronous Parallel）图计算模型的任务流引擎，提供超步计算、检查点容错、工作流模式等功能，支撑复杂任务的有向无环图（DAG）编排。

**标准名称**: 任务流引擎 (TaskFlow)

**代码目录**: `AgentOS/agentos/atoms/taskflow/`

**参见**: CoreLoopThree（三层认知循环）

---

### Thinkdual / 双思考系统

**定义**: AgentOS 的核心认知创新，由三组件构成的认知架构：

| 组件 | 角色 | 功能 |
|------|------|------|
| **t2 主思考** | 慢思考 | 深度推理、反思调整、长期规划 |
| **t1-f 快思考** | 快思考-事实 | 快速响应、流式验证、事实核查 |
| **t1-p 专业思考** | 快思考-专业 | 专业领域仲裁、深度质量评估 |

三组件通过 triple_coordinator 协同工作。"双思"指深度思考（t2）与快速思考（t1）两大思维模式的协作。

**标准中文名**: 双思考系统（正式）/ Thinkdual（简称）
**标准英文名**: Thinkdual Cognitive Dual-Thinking System（完整）/ Thinkdual（简化）

**旧称/禁止使用**: "认知双思系统"、"双系统认知模型"、"Thinkdual 认知双思系统"、"Triple Coordinator"（triple_coordinator 是协调器，非系统名称）

**系统内代码**: `tc_*`（思考链）/ `mc_*`（元认知）/ `sc_*`（流式验证）/ `tc3_*`（协调器）

**代码目录**: `AgentOS/agentos/atoms/coreloopthree/src/cognition/`

**参见**: CoreLoopThree（三层认知循环）、CognitiveEvolution（认知进化系统）

---

### TimeSliceInfer / 分时推理框架

**定义**: 记忆卷载的分时推理框架。将单次"全量加载→推理"分解为多个时间片上的"渐进式加载→增量推理→状态传递"流水线。每一时间片处理一个记忆子集，时间片间通过推理状态向量传递上下文连续性。

**标准名称**: 分时推理框架（正式）/ TimeSliceInfer（简称）

**旧称/禁止使用**: "时间切片推理"

**系统内代码**: `ts_inference_session_t`（设计中）

**参见**: MemoryRovol（记忆卷载）、MemorySwap（记忆交换算法）

---

### TraceID / 跟踪标识符

**定义**: 跨组件请求的全局唯一标识符，用于分布式追踪和日志关联。

**参见**: Security by Design（安全内生设计）、JSON-RPC 2.0

---

## 三、标准化名称映射总表

| 标准中文 | 标准英文 | 代码前缀 | 禁止使用的旧称 |
|----------|----------|---------|---------------|
| 智能体 | Agent | `agentos_sys_agent_*` | — |
| 系统调用层 | AirymaxSyscall | `agentos_sys_*` | Syscall Layer |
| 认知进化系统 | CognitiveEvolution | `cog_*` | — |
| 契约 | Contract | — | — |
| 原子核心 | CoreKern | `agentos_core_*` | 原子内核 |
| 三层认知循环 | CoreLoopThree | `agentos_loop_*` | 第三循环内核、三层循环运行时、三层一体架构 |
| 安全穹顶 | Cupolas | `cupolas_*` | Cupolas安全模块 |
| 用户态服务层 | Daemon | `*_d` 进程 | 守护进程、后端服务层 |
| 统一错误码体系 | ErrorCodeSystem | `AGENTOS_E*` | — |
| 反馈闭环 | Feedback Loop | — | — |
| 遗忘机制 | Forgetting Engine | `agentos_forgetting_*` | — |
| 数据分区存储 | HeapStore | `agentos_heapstore_*` | — |
| 进程间通信 | IPC | `agentos_ipc_*` | — |
| 多智能体协作框架 | MAC | `mac_*` | Multi-Agent System |
| 安全宏体系 | Memory Safety Macro System | `AGENTOS_MALLOC` 等 | — |
| 记忆交换算法 | MemorySwap | `agentos_memory_swap_*` | — |
| 记忆卷载 | MemoryRovol | `agentos_layer*_*` | 记忆漩涡引擎 |
| 微核心 | MicroCoreRT | `agentos_core_*` | 微内核、Microkernel |
| 协议适配层 | Protocols | — | — |
| 沙箱 | Sandbox | `agentos_sys_sandbox_*` | — |
| 安全内生设计 | Security by Design | — | — |
| 服务隔离 | Service Isolation | — | — |
| 技能 | Skill | `agentos_sys_skill_*` | — |
| 任务流引擎 | TaskFlow | `agentos_taskflow_*` | — |
| 双思考系统 | Thinkdual | `tc_*` / `mc_*` / `sc_*` | 认知双思系统、双系统认知模型、Thinkdual 认知双思系统 |
| 分时推理框架 | TimeSliceInfer | `ts_*` | 时间切片推理 |
| 跟踪标识符 | TraceID | — | — |

---

## 四、MemoryRovol 子系统术语

### L1 原始卷

| 标准中文 | 标准英文 | 系统内代码 |
|----------|----------|-----------|
| 原始卷 | L1 Raw | `agentos_layer1_raw_*` |
| 元数据索引 | Metadata DB | `agentos_raw_metadata_db_*` |
| 异步存储引擎 | Async Storage Engine | `async_storage_engine` |

### L2 特征层

| 标准中文 | 标准英文 | 系统内代码 |
|----------|----------|-----------|
| 特征层 | L2 Feature | `agentos_layer2_feature_*` |
| HNSW 索引 | HNSW Index | `hnsw_index_t` |
| BM25 索引 | BM25 Index | `agentos_bm25_index_*` |
| 混合检索 | Hybrid Search | `agentos_hybrid_search_*` |
| 嵌入器 | Embedder | `generate_*_embedding` |

### L3 结构层

| 标准中文 | 标准英文 | 系统内代码 |
|----------|----------|-----------|
| 结构层 | L3 Structure | `agentos_layer3_structure_*` |
| 知识图谱 | Knowledge Graph | `agentos_knowledge_graph_*` |
| 绑定算子 | VSA Binder | `agentos_binder_*` |
| 解绑算子 | VSA Unbinder | `agentos_unbinder_*` |
| 关系编码器 | Relation Encoder | `agentos_relation_encoder_*` |

### L4 模式层

| 标准中文 | 标准英文 | 系统内代码 |
|----------|----------|-----------|
| 模式层 | L4 Pattern | `agentos_layer4_pattern_*` |
| 持久同调 | Persistent Homology | `agentos_persistence_*` |
| 模式挖掘器 | Pattern Miner | `miner_discover_patterns` |
| 规则生成器 | Rule Generator | `agentos_rule_generator_*` |

### 检索系统

| 标准中文 | 标准英文 | 系统内代码 |
|----------|----------|-----------|
| 吸引子网络 | Attractor Network | `agentos_attractor_network_*` |
| 检索缓存 | Retrieval Cache | `agentos_retrieval_cache_*` |
| 重排序器 | Reranker | `agentos_reranker_*` |
| 挂载算子 | Mount Operator | `agentos_memoryrov_mount` |

### 遗忘机制

| 标准中文 | 标准英文 | 系统内代码 |
|----------|----------|-----------|
| 遗忘引擎 | Forgetting Engine | `agentos_forgetting_*` |
| 遗忘策略 | Forget Strategy | `agentos_forget_strategy_t` |
| 遗忘裁剪 | Prune | `agentos_forgetting_prune` |
| 记忆归档 | Archive | `archive_memory` |

### 跨层事件

| 标准中文 | 标准英文 | 系统内代码 |
|----------|----------|-----------|
| 跨层事件队列 | Cross-Layer Event Queue | `event_queue_*` |
| 事件类型 | Event Type | `cross_layer_event_type_t` |

---

## 五、命名规范速查

### 函数命名

| 模块 | 格式 | 示例 |
|------|------|------|
| 内核 | `agentos_core_[action]` | `agentos_core_init` |
| 系统调用 | `agentos_sys_[domain]_[action]` | `agentos_sys_task_submit` |
| 认知层 | `agentos_[prefix]_[action]` | `agentos_tc_context_window_append` |
| 安全穹顶 | `cupolas_[subsystem]_[action]` | `cupolas_permission_check` |
| 用户态服务 | `[service]_[action]` | `llm_d_complete` |
| 认知进化 | `cog_[action]` / `agentos_cog_[action]` | `cog_evolution_create` |
| 多智能体 | `mac_[action]` / `agentos_mac_[action]` | `mac_framework_create` |

### 结构体命名

| 规范 | 格式 | 示例 |
|------|------|------|
| 公共结构体 | `agentos_[name]_t` | `agentos_error_t` |
| 子系统结构体 | `[prefix]_[name]_t` | `tc_context_window_t` |
| 配置结构体 | `agentos_[name]_config_t` | `agentos_cognition_config_t` |

### 枚举命名

| 规范 | 格式 | 示例 |
|------|------|------|
| 状态枚举 | `AGENTOS_[NAME]_STATE_*` | `AGENTOS_TASK_STATE_READY` |
| 类型枚举 | `AGENTOS_[NAME]_TYPE_*` | `AGENTOS_PROTOCOL_TYPE_MCP` |

### 文件/目录命名

| 规范 | 规则 |
|------|------|
| 用户态服务目录 | `[name]_d`（如 `llm_d/`） |
| 源文件 | 下划线分隔 + `.c`（如 `thinking_chain.c`） |
| 头文件 | 下划线分隔 + `.h`（如 `cupolas.h`） |
| 测试文件 | `test_[module].c`（如 `test_loop.c`） |

---

## 六、附录

### A. Daemon 服务完整列表（10 个）

| 服务 | 功能 |
|------|------|
| gateway_d | API 网关（MCP/A2A/OpenAI 兼容协议） |
| llm_d | LLM 推理代理（缓存/计费/多 Provider 路由） |
| tool_d | 工具调用代理（注册/执行/验证） |
| market_d | 市场/注册表服务（Agent/Skill 注册安装） |
| sched_d | 任务调度服务 |
| monit_d | 监控服务（指标/追踪/告警） |
| channel_d | 通道服务 |
| observe_d | 观测服务 |
| notify_d | 通知服务 |
| info_d | 信息服务 |

### B. 协议适配层完整列表（9 种）

| 类别 | 协议 | 枚举值 |
|------|------|--------|
| 标准协议 | MCP v1 | `PROTO_MCP` |
| 标准协议 | A2A v0.3 | `PROTO_A2A` |
| 标准协议 | AGNTCY ACP | `PROTO_AGNTCY` |
| 集成适配 | OpenAI | `PROTO_OPENAI` |
| 集成适配 | Claude | `PROTO_CLAUDE` |
| 集成适配 | OpenJiuwen | `PROTO_OPENJIUWEN` |
| 集成适配 | OpenClaw | `PROTO_OPENCLAW` |
| 集成适配 | China Eco | `PROTO_CHINA_ECO` |
| 框架适配 | LangChain / AutoGen | — |

### C. 错误码分段表

| 段 | 范围 | 含义 |
|----|------|------|
| 成功 | 0 | `AGENTOS_SUCCESS` |
| 通用 | -1 ~ -999 | `AGENTOS_ERR_UNKNOWN` 等 |
| Task | -1001 ~ -1099 | 任务相关错误 |
| Memory | -2001 ~ -2099 | 记忆相关错误 |
| Session | -3001 ~ -3099 | 会话相关错误 |
| Telemetry | -4001 ~ -4099 | 遥测相关错误 |
| Internal | -5001 ~ -5099 | 内部通用错误 |
| Skill | -6001 ~ -6099 | 技能相关错误 |
| Agent | -7001 ~ -7099 | Agent 相关错误 |
| Sandbox | -8001 ~ -8099 | 沙箱相关错误 |
| Security | -9001 ~ -9099 | 安全相关错误 |

---

## 参考文献

[1] 体系并行论 — `Basic_Theories/CN_01_体系并行论.md`
[2] 认知层理论 — `Basic_Theories/CN_02_认知层理论.md`
[3] 记忆层理论 — `Basic_Theories/CN_03_记忆层理论.md`
[4] 设计原则 — `Basic_Theories/CN_04_设计原则.md`
[5] 架构设计原则 — `ARCHITECTURAL_PRINCIPLES.md`
[6] MemoryRovol 架构文档 — `Capital_Architecture/memoryrovol.md`
[7] 微核心架构 — `Capital_Architecture/microkernel.md`
[8] 三层认知循环 — `Capital_Architecture/coreloopthree.md`
[9] 系统调用层 — `Capital_Architecture/syscall.md`

---

**维护者**: AgentOS 架构委员会
**联系方式**: architecture@agentos.org(暂)
**版本历史**: v1.0 (2026-06-09) → v2.0 (2026-06-03) → v3.0 (2026-06-10) → **v4.0 (2026-06-10)**
**v4.0 变更**: 全面精简重构 — 移除文件行数统计、代码片段、测试列表、性能基准表等实现细节，回归纯术语定义参考。修正 AirymaxSyscall 拼写。