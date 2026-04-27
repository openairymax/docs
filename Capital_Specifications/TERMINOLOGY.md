Copyright (c) 2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# AgentOS 统一术语表

**版本**: Doc V2.0  
**状态**: 正式发布  
**维护者**: AgentOS 架构委员会  
**作者**: LirenWang  
**最后更新**: 2026-04-27  
**理论基础**: 工程两论（工程控制论、系统工程）、五维正交系统（系统观、内核观、认知观、工程观、设计美学）、双系统认知理论、微内核哲学、体系并行论（MCIS）

---

## 编制说明

### 一、术语表目的

本术语表是 AgentOS 规范体系的统一术语参考，旨在：

1. **统一术语使用**: 确保所有规范文档使用一致的术语定义
2. **建立交叉引用**: 提供术语与相关规范章节的快速链接
3. **支持快速检索**: 按字母顺序组织，支持快速查找
4. **促进理解**: 为开发者、审计人员、研究人员提供权威解释

### 二、术语来源

本术语表的术语来源于以下核心文档：

- [架构设计原则](../ARCHITECTURAL_PRINCIPLES.md)
- [认知层理论](../Basic_Theories/CN_02_认知层理论.md)
- [设计原则](../Basic_Theories/CN_04_设计原则.md)
- 项目修复与技术改进大纲 v10.5（Docs-closed 目录）
- 工程规范化标准手册 v10.5（Docs-closed 目录）

### 三、使用方法

1. **文档编写**: 所有规范文档必须使用本术语表的标准名称
2. **术语引用**: 使用 `[标准名称](#标准名称英文)` 格式进行交叉引用
3. **术语更新**: 新术语需通过架构委员会审核后加入
4. **版本管理**: 术语表版本与规范体系版本同步

### 四、术语分类

- **A. 核心概念**: Agent、Skill、Contract 等基础概念
- **B. 架构组件**: 微内核、认知循环运行时、记忆卷载等架构元素
- **C. 技术术语**: JSON-RPC、TraceID、错误码等技术细节
- **D. 工程原则**: 反馈闭环、层次分解、安全内生等设计原则

---

## 术语定义（按字母顺序）

### A

#### Agent (智能体)

**定义**: 具有认知能力的实体，能够感知环境、进行推理决策并执行行动。Agent 是 AgentOS 的基本执行单元。

**标准名称**: 智能体 (Agent)

**来源**: agent_contract.md  
**参见**: Skill（技能）

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

#### Cognitive Loop Runtime / 认知循环运行时（CoreLoopThree）

**定义**: AgentOS 的三层一体认知执行架构，包含认知层（Cognition）、执行层（Execution）、记忆层（Memory）三个引擎。认知层实现双思考功能（t2/t1-f/t1-p）。

**标准名称**: 认知循环运行时（CoreLoopThree）

**旧称/禁止使用**: "第三循环内核"、 "CoreLoopThree"（仅代码目录名）

**代码目录**: `atoms/coreloopthree/`

**来源**: 架构设计原则、CoreLoopThree 具体实现方案  
**参见**: 认知层双思考功能、记忆卷载

#### Cognitive Layer Dual-Thinking / 认知层双思考功能

**定义**: 认知循环运行时认知层的核心功能，由 triple_coordinator 实现 t2（主思考/深度推理）、t1-f（快思考/流式验证）、t1-p（专业思考/仲裁）三组件协同。

**标准名称**: 认知层双思考功能

**旧称/禁止使用**: "双思考系统"（会误导为独立系统）

**代码目录**: `atoms/coreloopthree/src/cognition/triple_coordinator`

**来源**: 架构设计原则、CoreLoopThree 具体方案  
**参见**: 认知循环运行时、triple_coordinator

#### Contract (契约)

**定义**: 机器可读的能力描述文件，定义 Agent 或 Skill 的接口、能力、权限等元数据。

**标准名称**: 契约 (Contract)

**来源**: agent_contract.md、skill_contract.md  
**参见**: Input Schema、Output Schema

#### Cupolas / 安全穹顶

**定义**: AgentOS 的安全防护层，提供虚拟工位（workbench）、权限裁决（permission）、输入净化（sanitizer）、审计追踪（audit）四层安全机制。

**标准名称**: 安全穹顶（Cupolas）

**代码目录**: `cupolas/`

**来源**: 架构设计原则、安全设计指南  
**参见**: Security by Design、Sandbox

### D

#### Daemon / 用户态服务层

**定义**: AgentOS 的用户态服务层，包含 LLM 服务（llm_d）、工具服务（tool_d）、网关服务（gateway_d）、监控服务（monit_d）、调度服务（sched_d）、市场服务（market_d）、通道服务（channel_d）共 7 个服务进程。这些进程以 `_d` 后缀命名，运行在用户态。

**标准名称**: 用户态服务层（daemon）

**旧称/禁止使用**: "后端服务层"、 "守护进程"、 "daemon进程"

**代码目录**: `daemon/`

**来源**: 架构设计原则  
**参见**: 微内核、IPC

### E

#### Error Code (错误码)

**定义**: 标准化的错误标识符，用于跨组件错误传递和处理。

**标准名称**: 错误码 (Error Code)

**完整列表**: error_code_reference.md

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

**定义**: 微内核提供的进程间通信机制，包括消息传递、共享内存、信号量等。

**标准名称**: 进程间通信 (IPC)

**来源**: 架构设计原则  
**参见**: 微内核、Syscall

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

#### Memory Rovol / 记忆卷载

**定义**: AgentOS 的四层记忆架构：L1 原始卷（Raw Storage）、L2 特征层（Feature Extraction）、L3 结构层（Knowledge Graph）、L4 模式层（Pattern Mining）。基于艾宾浩斯遗忘曲线驱动演化与检索。

**标准名称**: 记忆卷载（MemoryRovol）

**旧称/禁止使用**: "记忆漩涡引擎"、 "MemoryRovol"（仅代码目录名）

**代码目录**: `atoms/memoryrovol/`

**来源**: 架构设计原则、MemoryRovol 具体方案  
**参见**: 认知循环运行时

#### Microkernel / 微内核

**定义**: 极简的操作系统内核，只提供最基本的 IPC、内存管理、任务调度、时间管理、可观测性功能，其他功能作为用户态服务实现。

**标准名称**: 微内核（Microkernel）

**代码目录**: `atoms/corekern/`

**来源**: 架构设计原则  
**参见**: IPC、Syscall、Service Isolation

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

### S

#### Sandbox (沙箱)

**定义**: 隔离的执行环境，限制 Skill 的权限和资源访问，防止恶意或错误代码影响系统。

**标准名称**: 沙箱 (Sandbox)

**代码目录**: `atoms/syscall/src/sandbox*.c`

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
**参见**: 微内核、IPC

#### Skill (技能)

**定义**: 可复用的执行单元，为智能体提供具体能力，如文件操作、网络请求、数据分析等。

**标准名称**: 技能 (Skill)

**来源**: skill_contract.md  
**参见**: Agent、Contract

#### Syscall / 系统调用层

**定义**: 用户态程序请求内核服务的标准接口，覆盖 agent/memory/sandbox/skill/session/task/telemetry 等子系统。

**标准名称**: 系统调用层（Syscall）

**代码目录**: `atoms/syscall/`

**来源**: syscall_api_contract.md  
**参见**: 微内核、IPC

### T

#### TaskFlow / 任务流引擎

**定义**: AgentOS 的任务流引擎，提供 Pregel BSP 图计算引擎、工作流模式、上下文处理器、并行调度器等功能，支撑复杂任务编排。

**标准名称**: 任务流引擎（TaskFlow）

**旧称/禁止使用**: "TaskFlow"（仅在英文文档可用）

**代码目录**: `atoms/taskflow/`

**来源**: 架构设计原则  
**参见**: 认知循环运行时

#### TraceID (跟踪标识符)

**定义**: 跨组件请求的全局唯一标识符，用于分布式追踪和日志关联。

**标准名称**: 跟踪标识符 (TraceID)

**来源**: logging_format.md  
**参见**: 安全内生设计

---

## 标准化名称映射表

| 标准名称（中文） | 标准名称（英文） | 代码目录 | 禁止使用的旧称 |
|-----------------|-----------------|---------|--------------|
| 认知循环运行时 | CoreLoopThree | `atoms/coreloopthree/` | 第三循环内核 |
| 记忆卷载 | MemoryRovol | `atoms/memoryrovol/` | 记忆漩涡引擎 |
| 用户态服务层 | daemon | `daemon/` | 守护进程、后端服务层 |
| 认知层双思考功能 | Triple Coordinator | `cognition/triple_coordinator` | 双思考系统 |
| 微内核 | Microkernel | `atoms/corekern/` | — |
| 系统调用层 | Syscall | `atoms/syscall/` | — |
| 任务流引擎 | TaskFlow | `atoms/taskflow/` | — |
| 安全穹顶 | Cupolas | `cupolas/` | Cupolas安全模块 |

---

## 附录

### JSON-RPC 错误码

| 错误码 | 定义 |
|--------|------|
| -32700 | Parse error |
| -32600 | Invalid Request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |

**完整列表**: protocol_contract.md

---

## 参考文献

[1] AgentOS 设计哲学。../Basic_Theories/CN_04_设计原则.md  
[2] AgentOS 认知层理论。../Basic_Theories/CN_02_认知层理论.md  
[3] 记忆层理论。../Basic_Theories/CN_03_记忆层理论.md  
[4] 架构设计原则。../ARCHITECTURAL_PRINCIPLES.md  
[5] 工程规范化标准手册 v10.5（Docs-closed）  
[6] 项目修复与技术改进大纲 v10.5（Docs-closed）

---

**维护者**: AgentOS 架构委员会  
**联系方式**: architecture@agentos.org(暂)  
**贡献指南**: 欢迎通过 Pull Request 提交术语建议或修正
