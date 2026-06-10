Copyright (c) 2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# OpenAirymax AgentOS - 统一术语表

**最新**: 2026-06-09
**状态**: 维护中
**路径**: https://atomgit.com/openairymax/OpenAirymax/blob/main/Docs/TERMINOLOGY.md
**作者**:
    - Liren Wang
    - Zhixian Zhou

---

## 编制说明

### 一、术语表目的

本术语表是 OpenAirymax AgentOS 规范体系的统一术语参考，旨在：

1. **统一术语使用**: 确保所有规范文档使用一致的术语定义
2. **建立交叉引用**: 提供术语与相关规范章节的快速链接
3. **支持快速检索**: 按字母顺序组织，支持快速查找
4. **促进理解**: 为开发者、审计人员、研究人员提供权威解释

### 二、术语来源

本术语表的术语来源于以下核心文档：

- [架构设计原则](ARCHITECTURAL_PRINCIPLES.md)
- [体系并行论](Basic_Theories/CN_01_体系并行论.md)
- [认知层理论](Basic_Theories/CN_02_认知层理论.md)
- [记忆层理论](Basic_Theories/CN_03_记忆层理论.md)
- [设计原则](Basic_Theories/CN_04_设计原则.md)
- 项目源码（AgentOS/、MemoryRovol/、Desktop/）

### 三、使用方法

1. **文档编写**: 所有规范文档必须使用本术语表的标准名称
2. **术语引用**: 使用 `[标准名称](#标准名称英文)` 格式进行交叉引用
3. **术语更新**: 新术语需通过架构委员会审核后加入
4. **版本管理**: 术语表版本与规范体系版本同步

### 四、术语分类

- **A. 核心概念**: Agent、Skill、Contract 等基础概念
- **B. 架构组件**: 微核心、三层认知循环、记忆卷载等架构元素
- **C. 技术术语**: JSON-RPC、TraceID、错误码等技术细节
- **D. 工程原则**: 反馈闭环、层次分解、安全内生等设计原则
- **E. 算法术语**: 持久同调、HNSW、BM25、Hopfield 等算法名称

---

## 术语定义（按字母顺序）

### A

#### Agent (智能体)

**定义**: 具有认知能力的实体，能够感知环境、进行推理决策并执行行动。Agent 是 AgentOS 的基本执行单元。

**标准名称**: 智能体 (Agent)

**来源**: agent_contract.md
**参见**: Skill（技能）

#### AirymaxSyscall / 系统调用层

**定义**: 用户态程序请求内核服务的标准接口，覆盖 agent/memory/sandbox/skill/session/task/telemetry 等子系统。遵循机制与策略分离原则，通过精简的系统调用表（约 50 个调用）提供最基础的内核服务访问。

**标准名称**: 系统调用层 (AirymaxSyscall)

**系统内代码**: `agentos_sys_*` 函数族（如 `agentos_sys_task_submit`、`agentos_sys_memory_search`）

**代码目录**: `AgentOS/agentos/atoms/syscall/`

**来源**: syscall_api_contract.md
**参见**: 微核心、IPC

#### Authentication (认证)

**定义**: 验证通信参与方身份的过程，确保消息来源可信。

**标准名称**: 认证 (Authentication)

**来源**: protocol_contract.md
**参见**: Authorization（授权）

#### Authorization (授权)

**定义**: 确定已认证实体是否有权限执行特定操作的过程。

**标准名称**: 授权 (Authorization)

**来源**: protocol_contract.md
**参见**: Authentication（认证）、Permission（权限）

### C

#### Contract (契约)

**定义**: 机器可读的能力描述文件，定义 Agent 或 Skill 的接口、能力、权限等元数据。

**标准名称**: 契约 (Contract)

**来源**: agent_contract.md、skill_contract.md
**参见**: Input Schema、Output Schema

#### CoreKern / 原子核心

**定义**: AgentOS 微核心中不可再分的基础内核实现，承载 IPC、内存管理、任务调度、时间服务、可观测性等原子机制。是体系并行论中基础体 (Base Body) 的具象实现。

**标准名称**: 原子核心 (CoreKern)

**代码目录**: `AgentOS/agentos/atoms/corekern/`

**来源**: 架构设计原则
**参见**: 微核心、IPC

#### CoreLoopThree / 三层认知循环

**定义**: AgentOS 的三层认知循环，包含认知层（Cognition）、执行层（Execution）、记忆层（Memory）三个引擎。认知层实现双思考系统（Thinkdual）的 t2/t1-f/t1-p 三组件认知。

**标准名称**: 三层认知循环 (CoreLoopThree)

**旧称/禁止使用**: "第三循环内核"、"三层循环运行时"、"三层一体架构"、"三层循环核心"

**系统内代码**: `agentos_loop_*` 函数族

**代码目录**: `AgentOS/agentos/atoms/coreloopthree/`

**来源**: 架构设计原则、CoreLoopThree 具体实现方案
**参见**: 双思考系统（Thinkdual）、记忆卷载

#### Cupolas / 安全穹顶

**定义**: AgentOS 的安全防护层，提供虚拟工位（workbench）、权限裁决（permission）、输入净化（sanitizer）、审计追踪（audit）四层安全机制。

**标准名称**: 安全穹顶 (Cupolas)

**系统内代码**: `cupolas_*` 函数族

**代码目录**: `AgentOS/agentos/cupolas/`

**来源**: 架构设计原则、安全设计指南
**参见**: Security by Design、Sandbox

### D

#### Daemon / 用户态服务层

**定义**: AgentOS 的用户态服务层，包含网关服务（gateway_d）、LLM 服务（llm_d）、市场服务（market_d）、调度服务（sched_d）、监控服务（monit_d）、工具服务（tool_d）、通道服务（channel_d）、观测服务（observe_d）、通知服务（notify_d）、信息服务（info_d）共 10 个服务进程。这些进程以 `_d` 后缀命名，运行在用户态。

**标准名称**: 用户态服务层 (daemon)

**旧称/禁止使用**: "后端服务层"、"守护进程"、"daemon进程"

**强制说明**: 在所有文档中必须使用"用户态服务层（daemon）"，**严禁使用"守护进程"**。"守护进程"一词在 AgentOS 语境中具有误导性，会与操作系统传统守护进程概念混淆。所有规范、文档、代码注释中均不得出现"守护进程"。

**代码目录**: `AgentOS/agentos/daemon/`

**来源**: 架构设计原则
**参见**: 微核心、IPC

### E

#### Error Code System / 统一错误码体系

**定义**: AgentOS 采用双错误码体系：C 语言负整数体系（首要，定义于 `error.h`，AGENTOS_SUCCESS=0, AGENTOS_EINVAL=-2 等）和 SDK 十六进制体系（次要，定义于 error_code_reference.md，0x0000-0x7FFF 分段）。C 内核和 daemon 层必须使用负整数体系；SDK 和外部接口可使用十六进制体系。

**标准名称**: 统一错误码体系 (ErrorCodeSystem)

**系统内代码**: `AGENTOS_E*` 宏族（如 `AGENTOS_SUCCESS`、`AGENTOS_EINVAL`、`AGENTOS_ENOMEM`）

**禁止**: 在 C 内核代码中使用十六进制错误码，或在 SDK 中使用负整数错误码

**来源**: error_code_reference.md、error.h
**参见**: Error Code（错误码）

### F

#### Feedback Loop (反馈闭环)

**定义**: 系统通过输出结果调整后续行为的控制机制，包括实时反馈、轮次内反馈、跨轮次反馈三个层次。

**标准名称**: 反馈闭环 (Feedback Loop)

**来源**: 架构设计原则
**参见**: Engineering Cybernetics、System Engineering

### I

#### Input Schema (输入模式)

**定义**: 定义 Skill 或 Agent 输入数据结构的 JSON Schema。

**标准名称**: 输入模式 (Input Schema)

**来源**: skill_contract.md
**参见**: Output Schema、Contract

#### IPC (进程间通信)

**定义**: 微核心提供的进程间通信机制，包括消息传递、共享内存、信号量等。

**标准名称**: 进程间通信 (IPC)

**来源**: 架构设计原则
**参见**: 微核心、AirymaxSyscall

### J

#### JWT (JSON Web Token)

**定义**: 用于认证和授权的紧凑、URL安全的令牌格式。

**标准名称**: JSON Web 令牌 (JWT)

**来源**: protocol_contract.md
**参见**: Authentication、Authorization

#### JSON-RPC 2.0

**定义**: AgentOS 采用的轻量级远程过程调用协议，支持请求-响应和通知两种模式。

**标准名称**: JSON-RPC 2.0

**来源**: protocol_contract.md
**参见**: Error Code、TraceID

### M

#### MemorySwap / MemorySwap 算法

**定义**: 记忆卷载的存储层级交换机制，借鉴操作系统虚拟内存的分页交换，将记忆按访问热度划分为热/温/冷三个存储层级，在不同层级之间自动迁移记忆数据。热记忆保持全状态驻留，冷记忆仅保留最小索引信息，需要时通过 swap-in 恢复到热层。

**标准名称**: 记忆交换算法（正式）/ MemorySwap（简称）

**系统内代码**: `agentos_memory_swap_manager_t`、`swap_score`、`swap_count`

**代码目录**: `MemoryRovol/docs/memory_swap_design.md`

**来源**: MemoryRovol 设计文档
**参见**: 记忆卷载、遗忘机制

#### MemoryRovol / 记忆卷载

**定义**: AgentOS 的四层记忆架构：L1 原始卷（Raw Storage）、L2 特征层（Feature Extraction）、L3 结构层（Knowledge Graph）、L4 模式层（Pattern Mining）。基于艾宾浩斯遗忘曲线驱动演化与检索。名称取自 Memory + Rovol（Roll + Volume 的融合造词），寓意记忆的滚动卷载。

**标准名称**: 记忆卷载 (MemoryRovol)

**旧称/禁止使用**: "记忆漩涡引擎"

**系统内代码**: `agentos_memoryrov_*` 函数族、`agentos_memoryrov_handle_t`

**代码目录**: `MemoryRovol/`（独立子项目）、`AgentOS/agentos/atoms/memoryrovol/`（集成路径）

**来源**: 架构设计原则、MemoryRovol 具体方案
**参见**: 三层认知循环、遗忘机制、TimeSliceInfer

#### MEMORY_FREE_SAFE

**定义**: AgentOS 安全释放宏，定义于 `agentos_memory.h`，接受指向指针的指针（ptr-to-ptr），在释放内存后将指针置 NULL，防止悬垂指针（dangling pointer）。是安全宏体系的核心宏之一。

**标准名称**: MEMORY_FREE_SAFE

**签名**: `MEMORY_FREE_SAFE(ptr_to_ptr)` — 参数为指向待释放指针的指针

**代码目录**: `AgentOS/agentos/atoms/corekern/include/agentos_memory.h`

**来源**: agentos_memory.h、安全编码规范
**参见**: 安全宏体系

#### Memory Safety Macro System / 安全宏体系

**定义**: AgentOS 的完整内存安全宏体系，覆盖内存分配、释放、字符串操作和禁止函数，确保内存操作的类型安全和空指针安全。

**标准名称**: 安全宏体系 (Memory Safety Macro System)

**包含以下宏族**:

1. **AGENTOS_* 内存操作宏**（定义于 `memory_compat.h`）:
   - `AGENTOS_MALLOC` / `AGENTOS_FREE` / `AGENTOS_CALLOC` / `AGENTOS_REALLOC` / `AGENTOS_STRDUP` / `AGENTOS_STRNDUP`

2. **安全操作宏**（定义于 `memory_compat.h`）:
   - `AGENTOS_STRNCPY_TERM` — 安全 strncpy 并保证终止符
   - `AGENTOS_MEMCPY_SAFE` — 安全 memcpy 带边界检查
   - `SAFE_MALLOC_ARRAY` — 安全数组分配带溢出检查

3. **安全释放宏**（定义于 `agentos_memory.h`）:
   - `MEMORY_FREE_SAFE` — 释放内存并置指针为 NULL（接受 ptr-to-ptr）

4. **禁止函数列表**（定义于 `banned_functions.h`）:
   - `malloc` / `free` / `calloc` / `realloc` / `strdup`
   - `printf` / `fprintf` / `strcpy` / `strcat` / `sprintf` / `gets`

**强制要求**: 所有 AgentOS 代码必须使用安全宏体系，严禁直接调用禁止函数列表中的函数。

**代码目录**: `AgentOS/agentos/atoms/corekern/include/`

**来源**: 安全编码规范、memory_compat.h、agentos_memory.h、banned_functions.h
**参见**: MEMORY_FREE_SAFE、Security by Design

#### MicroCoreRT / 微核心

**定义**: 极简的操作系统内核，只提供最基本的 IPC、内存管理、任务调度、时间管理、可观测性功能，其他功能作为用户态服务实现。名称取自 Micro + Core + RT（Runtime），强调其作为运行时基础的核心地位。采用第四代微核心架构设计（L4 MicroCoreRT）。

**标准名称**: 微核心 (MicroCoreRT)

**旧称/禁止使用**: "微内核"、"Microkernel"

**系统内代码**: `agentos_core_*` 函数族

**代码目录**: `AgentOS/agentos/atoms/corekern/`

**来源**: 架构设计原则
**参见**: 原子核心、IPC、AirymaxSyscall、Service Isolation

### O

#### Output Schema (输出模式)

**定义**: 定义 Skill 或 Agent 输出数据结构的 JSON Schema。

**标准名称**: 输出模式 (Output Schema)

**来源**: skill_contract.md
**参见**: Input Schema、Contract

### P

#### Permission (权限)

**定义**: 执行特定操作所需的授权，如文件访问、网络通信、系统调用等。

**标准名称**: 权限 (Permission)

**来源**: skill_contract.md
**参见**: Sandbox、Service Isolation

#### Protocols / 协议适配层

**定义**: AgentOS 的多协议适配层，支持 MCP v1、A2A v0.3、AGNTCY ACP 等标准协议，以及 OpenAI、Claude、OpenJiuwen、OpenClaw 等集成适配，和 LangChain、AutoGen 等框架适配。

**标准名称**: 协议适配层 (Protocols)

**代码目录**: `AgentOS/agentos/protocols/`

**来源**: 架构设计原则
**参见**: 用户态服务层

### S

#### Sandbox (沙箱)

**定义**: 隔离的执行环境，限制 Skill 的权限和资源访问，防止恶意或错误代码影响系统。

**标准名称**: 沙箱 (Sandbox)

**系统内代码**: `agentos_sys_skill_*` 函数族

**代码目录**: `AgentOS/agentos/atoms/syscall/src/sandbox*.c`

**来源**: skill_contract.md
**参见**: Service Isolation、Permission

#### Schema Version (契约模式版本)

**定义**: 契约 JSON Schema 的版本标识符，用于兼容性检查和版本迁移。

**标准名称**: 契约模式版本 (Schema Version)

**来源**: agent_contract.md
**参见**: Contract、Backward Compatibility

#### Security by Design (安全内生设计)

**定义**: 将安全机制内嵌于系统架构和设计中的理念，而非事后附加。

**标准名称**: 安全内生设计 (Security by Design)

**来源**: 架构设计原则
**参见**: 安全穹顶、Sandbox

#### Service Isolation (服务隔离)

**定义**: 用户态服务之间相互隔离，只能通过内核 IPC 通信的安全机制。

**标准名称**: 服务隔离 (Service Isolation)

**来源**: 架构设计原则
**参见**: 微核心、IPC

#### Skill (技能)

**定义**: 可复用的执行单元，为智能体提供具体能力，如文件操作、网络请求、数据分析等。

**标准名称**: 技能 (Skill)

**来源**: skill_contract.md
**参见**: Agent、Contract

### T

#### TaskFlow / 任务流引擎

**定义**: AgentOS 的任务流引擎，提供 Pregel BSP 图计算引擎、工作流模式、上下文处理器、并行调度器等功能，支撑复杂任务编排。

**标准名称**: 任务流引擎 (TaskFlow)

**代码目录**: `AgentOS/agentos/atoms/taskflow/`

**来源**: 架构设计原则
**参见**: 三层认知循环

#### Thinkdual / 双思考系统

**定义**: AgentOS 的核心认知创新，由 t2 主思考、t1-f 快思考、t1-p 专业思考三组件构成的认知架构。t2 负责深度推理与反思调整，t1-f 负责快速响应与流式验证，t1-p 负责专业领域仲裁。三组件通过 triple_coordinator 协同工作，实现效率与精度的动态平衡。"双思"指深度思考（t2）与快速思考（t1）两大思维模式，而 t1-f 与 t1-p 是 t1 模式下的两个子组件。

**标准中文名**: 双思考系统（正式）/ Thinkdual（简称）

**标准英文名**: Thinkdual Cognitive Dual-Thinking System（完整）/ Thinkdual（简化）

**旧称/禁止使用**: "认知双思系统"、"双系统认知模型"、"Thinkdual 认知双思系统"、"Triple Coordinator"（triple_coordinator 是协调器而非系统名称）

**系统内代码**: `agentos_llm_dual_think`、`agentos_dual_think_config_t`

**代码目录**: `AgentOS/agentos/atoms/coreloopthree/src/cognition/`

**来源**: 架构设计原则、CoreLoopThree 具体方案
**参见**: 三层认知循环、triple_coordinator

#### TimeSliceInfer / 分时推理框架

**定义**: 记忆卷载的分时推理框架，将单次"全量加载→推理"的原子操作分解为多个时间片上的"渐进式加载→增量推理→状态传递"流水线。每一时间片处理一个记忆子集，时间片之间通过推理状态向量传递上下文连续性。

**标准名称**: 分时推理框架（正式）/ TimeSliceInfer（简称）

**旧称/禁止使用**: "时间切片推理"

**系统内代码**: `ts_inference_session_t`（设计中）

**代码目录**: `MemoryRovol/docs/time_slicing_design.md`

**来源**: MemoryRovol 设计文档
**参见**: 记忆卷载、MemorySwap

#### TraceID (跟踪标识符)

**定义**: 跨组件请求的全局唯一标识符，用于分布式追踪和日志关联。

**标准名称**: 跟踪标识符 (TraceID)

**来源**: logging_format.md
**参见**: 安全内生设计

---

## 标准化名称映射表

| 标准中文 | 标准英文 | 系统内代码 | 代码目录 | 禁止使用的旧称 |
|----------|----------|-----------|---------|---------------|
| 三层认知循环 | CoreLoopThree | `agentos_loop_*` | `AgentOS/agentos/atoms/coreloopthree/` | 第三循环内核、三层循环运行时、三层一体架构 |
| 记忆卷载 | MemoryRovol | `agentos_memoryrov_*` | `MemoryRovol/` | 记忆漩涡引擎 |
| 分时推理框架 | TimeSliceInfer | `ts_inference_session_t` | `MemoryRovol/` | 时间切片推理 |
| 记忆交换算法 | MemorySwap | `agentos_memory_swap_manager_t` | `MemoryRovol/` | — |
| 双思考系统 | Thinkdual | `agentos_llm_dual_think` | `AgentOS/agentos/atoms/coreloopthree/src/cognition/` | 认知双思系统、双系统认知模型、Thinkdual 认知双思系统、Triple Coordinator |
| 统一错误码体系 | ErrorCodeSystem | `AGENTOS_E*` | `AgentOS/agentos/atoms/corekern/include/error.h` | — |
| 微核心 | MicroCoreRT | `agentos_core_*` | `AgentOS/agentos/atoms/corekern/` | 微内核、Microkernel |
| 原子核心 | CoreKern | — | `AgentOS/agentos/atoms/corekern/` | 原子内核 |
| 系统调用层 | AirymaxSyscall | `agentos_sys_*` | `AgentOS/agentos/atoms/syscall/` | Syscall Layer |
| 任务流引擎 | TaskFlow | — | `AgentOS/agentos/atoms/taskflow/` | — |
| 安全穹顶 | Cupolas | `cupolas_*` | `AgentOS/agentos/cupolas/` | Cupolas安全模块 |
| 安全宏体系 | Memory Safety Macro System | `AGENTOS_MALLOC` 等 | `AgentOS/agentos/atoms/corekern/include/` | — |
| 用户态服务层 | daemon | `*_d` 进程 | `AgentOS/agentos/daemon/` | 守护进程、后端服务层 |
| 协议适配层 | Protocols | — | `AgentOS/agentos/protocols/` | — |
| 遗忘机制 | Forgetting Engine | `agentos_forgetting_*` | `MemoryRovol/src/forgetting/` | — |
| 检索系统 | Retrieval System | `agentos_retrieval_*` | `MemoryRovol/src/retrieval/` | — |

---

## MemoryRovol 子系统术语

### L1 原始卷 (Raw Storage)

| 标准中文 | 标准英文 | 系统内代码 | 说明 |
|----------|----------|-----------|------|
| 原始卷 | L1 Raw | `agentos_layer1_raw_*` | 第一层：原始记忆数据持久化存储 |
| 元数据索引 | Metadata DB | `agentos_raw_metadata_db_*` | SQLite 元数据索引 |
| 异步存储引擎 | Async Storage Engine | `async_storage_engine` | 批量异步写入、错误恢复 |

### L2 特征层 (Feature Extraction)

| 标准中文 | 标准英文 | 系统内代码 | 说明 |
|----------|----------|-----------|------|
| 特征层 | L2 Feature | `agentos_layer2_feature_*` | 第二层：向量化特征提取与索引 |
| HNSW 索引 | HNSW Index | `hnsw_index_t` | 分层导航小世界图近邻索引 |
| BM25 索引 | BM25 Index | `agentos_bm25_index_*` | Okapi BM25 文本检索索引 |
| 混合检索 | Hybrid Search | `agentos_hybrid_search_*` | 向量+BM25 混合检索融合 |
| 自适应融合 | Adaptive Fusion | `agentos_hybrid_search_adaptive` | 归一化+动态权重+双路融合 |
| 嵌入器 | Embedder | `generate_*_embedding` | OpenAI/DeepSeek/ONNX 嵌入生成 |

### L3 结构层 (Knowledge Graph)

| 标准中文 | 标准英文 | 系统内代码 | 说明 |
|----------|----------|-----------|------|
| 结构层 | L3 Structure | `agentos_layer3_structure_*` | 第三层：知识图谱与关系编码 |
| 知识图谱 | Knowledge Graph | `agentos_knowledge_graph_*` | 实体-关系图谱存储与查询 |
| 绑定算子 | VSA Binder | `agentos_binder_*` | 向量符号架构绑定操作（Q矩阵） |
| 解绑算子 | VSA Unbinder | `agentos_unbinder_*` | 向量符号架构解绑操作 |
| 序列编码器 | Sequence Encoder | `agentos_sequence_encoder_*` | Fourier 位置编码 |
| 关系编码器 | Relation Encoder | `agentos_relation_encoder_*` | 三元组(S+P+O)关系编码 |
| 多跳邻域查询 | Neighborhood Query | `agentos_knowledge_graph_neighborhood` | BFS N跳邻域+累积置信度 |
| 置信度传播 | Confidence Propagation | `agentos_knowledge_graph_propagate_confidence` | Label Propagation 置信度传播 |

### L4 模式层 (Pattern Mining)

| 标准中文 | 标准英文 | 系统内代码 | 说明 |
|----------|----------|-----------|------|
| 模式层 | L4 Pattern | `agentos_layer4_pattern_*` | 第四层：持久模式挖掘与规则生成 |
| 持久同调 | Persistent Homology | `agentos_persistence_*` | 0D 持久同调特征计算 |
| 模式挖掘器 | Pattern Miner | `miner_discover_patterns` | 基于聚类和持久同调的模式发现 |
| 聚类引擎 | Clustering Engine | `agentos_clustering_*` | HDBSCAN/DBSCAN 密度聚类 |
| 规则生成器 | Rule Generator | `agentos_rule_generator_*` | LLM 辅助规则生成 |
| 模式匹配器 | Pattern Matcher | `agentos_pattern_matcher_*` | 向量相似度模式匹配 |
| 特征质量评分 | Feature Quality Score | `agentos_persistence_score_features` | Q=persistence/noise*log2(1+death) |
| 特征显著性排序 | Feature Ranking | `agentos_persistence_rank_features` | 按质量分数降序+阈值过滤 |

### 检索系统 (Retrieval)

| 标准中文 | 标准英文 | 系统内代码 | 说明 |
|----------|----------|-----------|------|
| 吸引子网络 | Attractor Network | `agentos_attractor_network_*` | 现代 Hopfield 网络检索 |
| 能量函数 | Energy Function | `agentos_energy_hopfield` | Hopfield 网络能量计算 |
| 检索缓存 | Retrieval Cache | `agentos_retrieval_cache_*` | 增强型 LRU 缓存（TTL+统计） |
| 重排序器 | Reranker | `agentos_reranker_*` | 交叉编码器/BM25 降级重排序 |
| RRF 融合 | Reciprocal Rank Fusion | `agentos_reranker_rrf` | 倒数排名融合排序 |
| 挂载算子 | Mount Operator | `agentos_memoryrov_mount` | 按上下文窗口限制加载工作记忆 |

### 遗忘机制 (Forgetting)

| 标准中文 | 标准英文 | 系统内代码 | 说明 |
|----------|----------|-----------|------|
| 遗忘引擎 | Forgetting Engine | `agentos_forgetting_*` | 艾宾浩斯遗忘曲线驱动 |
| 遗忘策略 | Forget Strategy | `agentos_forget_strategy_t` | NONE/EBBINGHAUS/LINEAR/ACCESS_BASED |
| 遗忘裁剪 | Prune | `agentos_forgetting_prune` | 低权重记忆裁剪 |
| 记忆归档 | Archive | `archive_memory` | 低权重记忆移至冷存储 |
| 记忆复活 | Revive | `revive_memory` | 从归档恢复记忆 |

### 跨层事件 (Cross-Layer Event)

| 标准中文 | 标准英文 | 系统内代码 | 说明 |
|----------|----------|-----------|------|
| 跨层事件队列 | Cross-Layer Event Queue | `event_queue_*` | 发布-订阅模式四层间通信 |
| 事件类型 | Event Type | `cross_layer_event_type_t` | RECORD_APPENDED/EMBEDDING_CREATED 等 |

---

## 附录

### A. JSON-RPC 错误码

| 错误码 | 定义 |
|--------|------|
| -32700 | Parse error |
| -32600 | Invalid Request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |

**完整列表**: protocol_contract.md

### B. AgentOS 用户态服务层 (daemon) 完整列表

| 服务 | 目录 | 说明 |
|------|------|------|
| gateway_d | `daemon/gateway_d/` | 网关服务（MCP/A2A/OpenAI 兼容协议） |
| llm_d | `daemon/llm_d/` | LLM 服务（缓存/Token 计数/成本追踪） |
| market_d | `daemon/market_d/` | 市场服务（Agent/Skill 注册与安装） |
| sched_d | `daemon/sched_d/` | 调度服务（优先级/轮询/加权/ML 策略） |
| monit_d | `daemon/monit_d/` | 监控服务（指标/追踪/告警/日志） |
| tool_d | `daemon/tool_d/` | 工具服务（注册/执行/验证/缓存） |
| channel_d | `daemon/channel_d/` | 通道服务 |
| observe_d | `daemon/observe_d/` | 观测服务 |
| notify_d | `daemon/notify_d/` | 通知服务 |
| info_d | `daemon/info_d/` | 信息服务 |

### C. 协议适配层 (Protocols) 完整列表

| 类别 | 协议 | 目录 |
|------|------|------|
| 标准协议 | MCP v1 | `protocols/standards/mcp/` |
| 标准协议 | A2A v0.3 | `protocols/standards/a2a/` |
| 标准协议 | AGNTCY ACP | `protocols/standards/agntcy/` |
| 集成适配 | OpenAI | `protocols/integrations/openai/` |
| 集成适配 | Claude | `protocols/integrations/claude/` |
| 集成适配 | OpenJiuwen | `protocols/integrations/openjiuwen/` |
| 集成适配 | OpenClaw | `protocols/integrations/openclaw/` |
| 集成适配 | China Eco | `protocols/integrations/china_eco/` |
| 框架适配 | LangChain | `protocols/frameworks/langchain/` |
| 框架适配 | AutoGen | `protocols/frameworks/autogen/` |

---

## 参考文献

[1] AgentOS 设计哲学。Basic_Theories/CN_04_设计原则.md
[2] AgentOS 体系并行论。Basic_Theories/CN_01_体系并行论.md
[3] AgentOS 认知层理论。Basic_Theories/CN_02_认知层理论.md
[4] 记忆层理论。Basic_Theories/CN_03_记忆层理论.md
[5] 架构设计原则。ARCHITECTURAL_PRINCIPLES.md
[6] MemoryRovol 架构文档。Capital_Architecture/memoryrovol.md

---

**维护者**: AgentOS 架构委员会
**联系方式**: architecture@agentos.org(暂)
**贡献指南**: 欢迎通过 Pull Request 提交术语建议或修正
