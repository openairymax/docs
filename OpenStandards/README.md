Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.  

# Airymax 开放标准体系总览  

> **文档定位**: Airymax 开放标准  
> **版本**: 1.0（开放标准草案）  
> **最后更新**: 2026-07-09  
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0  

***

## 一、本体系是什么

Airymax 开放标准体系（Airymax Open Standards，下称"本体系"）是 SPHARX Ltd. 发布的一组**公开的、可被任何第三方自由实现**的技术标准，覆盖 AI Agent 运行时、Agent 间通信、Agent 安全模型、Agent 记忆管理、Agent 认知循环、Agent 调度六大领域。

本体系的核心定位是**生态繁荣**：通过对外公开稳定、完整、可验证的接口契约，使第三方 Agent、工具链、运行时实现、操作系统发行版能够遵循同一套语义与 Agent 生态互联互通，共同推动更亲和、更高水准的 Agent 技术标准与工程规范，促进开源生态繁荣。

本体系遵循"开放标准 + 双许可证"的治理模式：

- **AGPL-3.0-or-later**：保障网络服务场景下的代码回流，维护生态健康。
- **Apache-2.0**：允许商业闭源实现遵循本标准，鼓励最广泛采纳。

任何组织或个人均可基于本体系实现兼容运行时，并独立分发。兼容性由本体系第 7 节定义的兼容性测试套件裁定。

***

## 二、与 Airymax 工程项目的关系

Airymax 项目包含两个核心产品：

| 产品                | 标准名称                              | 定位                                          |
| ----------------- | --------------------------------- | ------------------------------------------- |
| **agentrt**       | AirymaxAgentRT 极境智能体运行底座平台工程      | 用户态 AI Agent 运行时平台工程，定位为**微核心**（micro-core） |
| **agentrt-linux** | agentrt-linux（AirymaxOS）极境智能体操作系统 | 智能体操作系统发行版，定位为**微内核**（micro-kernel）         |

术语约束：

- **agentrt = 微核心**：agentrt 的 MicroCoreRT 是用户态微核心运行时，**不得**用"微内核"描述 agentrt。
- **agentrt-linux = 微内核**：agentrt-linux 基于 Linux 6.6 内核基线进行微内核化改造（sched\_ext + eBPF + io\_uring），将 VFS、网络栈、部分驱动移到用户态。
- 本体系标准**不依赖任何特定上游发行版**，文档中**不出现**任何具体上游发行版名称。

### 2.1 开放标准 vs. 工程标准

| 维度       | Airymax 开放标准（本体系）        | Airymax 工程标准                                    |
| -------- | ------------------------ | ----------------------------------------------- |
| **可见性**  | 公开、可被第三方实现               | 内部实现规范                                          |
| **关注点**  | 接口契约、语义、兼容性              | 编码规范、代码格式、开发流程、治理                               |
| **变更频率** | 极慢，遵循语义化版本               | 随工程演进持续迭代                                       |
| **合规判定** | 兼容性测试套件                  | CI 流水线、CODEOWNERS、DCO                           |
| **路径**   | `docs/AirymaxStandards/` | `docs/AirymaxAgentOS/50-engineering-standards/` |

简言之：**开放标准是公开的生态接口，工程标准是内部实现规范**。一个第三方实现只要符合开放标准即可纳入 Airymax 生态；其内部如何编码、如何组织代码，由该实现自行决定。

### 2.2 同源与共享层次

本体系标准来自 agentrt 与 agentrt-linux 的共同抽象，遵循 IRON-9 v2 三层模型：

| 层次    | 标注     | 共享程度          | 在开放标准中的体现                     |
| ----- | ------ | ------------- | ----------------------------- |
| 共享契约层 | \[SC]  | 完全共享代码        | 数据结构、错误码、魔术数、枚举值——本体系标准的"硬契约" |
| 语义同源层 | \[SS]  | API 签名相同、实现独立 | 行为契约——本体系标准的"语义契约"            |
| 完全独立层 | \[IND] | 完全独立          | 不进入本体系标准，各实现自由演进              |

本体系仅对外公开 \[SC] 与 \[SS] 两层；\[IND] 不属于开放标准范畴。

***

## 三、标准文档清单

本体系由 1 个总索引（本文件）+ 6 个主题标准文档构成：

| 编号  | 文档                                                                               | 主题        | 主要契约                                           |
| --- | -------------------------------------------------------------------------------- | --------- | ---------------------------------------------- |
| 总索引 | [README.md](./README.md)                                                         | 总览与导航     | 本体系定位、文档清单、治理                                  |
| 01  | [01-airymax-agent-runtime-standard.md](./01-airymax-agent-runtime-standard.md)   | Agent 运行时 | 生命周期、元数据、能力声明、Token 预算契约                       |
| 02  | [02-airymax-ipc-protocol-standard.md](./02-airymax-ipc-protocol-standard.md)     | IPC 协议    | 128B 消息头、5 种 payload 类型、请求-响应/发布-订阅            |
| 03  | [03-airymax-security-model-standard.md](./03-airymax-security-model-standard.md) | 安全模型      | Cupolas capability 38 ID + 派生、LSM 254 ID、4 值裁决 |
| 04  | [04-airymax-memory-rovol-standard.md](./04-airymax-memory-rovol-standard.md)     | 记忆卷载      | L1-L4 分级、GFP 掩码、PMEM 接口、遗忘机制                   |
| 05  | [05-airymax-cognition-loop-standard.md](./05-airymax-cognition-loop-standard.md) | 认知循环      | CoreLoopThree 三阶段、Thinkdual 双模式、LLM 推理阶段       |
| 06  | [06-airymax-scheduling-standard.md](./06-airymax-scheduling-standard.md)         | 调度        | SCHED\_EXT、任务描述符、vtime Q16.16、1024 并发          |

***

## 四、标准编号体系

本体系采用 `AOS-STD-<域>-<NNN>` 三段式编号：

- **AOS**：Airymax Open Standards 前缀
- **域**：RT（运行时）/ IPC / SEC / MEM / COG / SCHED
- **NNN**：3 位序号

示例：`AOS-STD-RT-001`（Agent 生命周期标准第 1 条）。

每条标准的编号一旦分配，**永不变更**；新增标准只能追加到对应域的末尾；废弃标准标记 `DEPRECATED` 但编号不回收。完整的编号注册表见各主题文档的"标准条目"章节。

***

## 五、稳定性承诺

### 5.1 接口稳定性分级

| 级别          | 含义                      | 变更策略                      |
| ----------- | ----------------------- | ------------------------- |
| **L1 永久稳定** | Agent 公共 API、IPC 消息头魔术数 | 永不变更；新增字段只能追加到 reserved 区 |
| **L2 弃用流程** | 运行时接口、能力令牌格式            | 至少 2 个版本弃用期，期间新旧并存        |
| **L3 版本协商** | 内部 API、payload 扩展字段     | 通过 version 字段协商           |
| **L4 完全自由** | 私有实现细节                  | 不属于本体系标准                  |

### 5.2 语义化版本

本体系整体遵循语义化版本 `MAJOR.MINOR.PATCH`：

- **MAJOR**：破坏性变更（L1 稳定接口变更需要 MAJOR 升级，原则上永不出新 MAJOR）
- **MINOR**：向后兼容的新增（新枚举值、新 payload 类型、新 GFP 掩码位）
- **PATCH**：勘误与澄清

当前版本：**1.0（开放标准草案）**。

***

## 六、兼容性判定

### 6.1 兼容性级别

| 级别                     | 判定条件                         | 标识                       |
| ---------------------- | ---------------------------- | ------------------------ |
| **Airymax Certified**  | 通过全套兼容性测试套件 + 通过 SPHARX 认证审计 | 可使用 Airymax Certified 商标 |
| **Airymax Compatible** | 通过全套兼容性测试套件                  | 可声明"兼容 Airymax 开放标准 1.0" |
| **Airymax Inspired**   | 部分遵循标准，未通过全部测试               | 不得使用上述标识                 |

### 6.2 兼容性测试套件

兼容性测试套件包含以下维度（详细测试用例见各主题文档附录）：

| 维度    | 测试内容                   | 必须通过 |
| ----- | ---------------------- | ---- |
| 二进制兼容 | IPC 消息头字节布局、任务描述符字节布局  | 是    |
| 语义兼容  | 生命周期状态机、能力派生约束、记忆层级语义  | 是    |
| 行为兼容  | 请求-响应配对、发布-订阅顺序、调度公平性  | 是    |
| 错误码兼容 | 错误码范围与语义               | 是    |
| 安全兼容  | capability 派生权限不增、撤销递归 | 是    |

测试套件代码以 Apache-2.0 许可证发布，鼓励社区贡献新用例。

***

## 七、治理流程

### 7.1 标准演进流程

任何对本体系标准的修改须遵循以下流程：

1. **RFC**：在 Airymax 标准仓库发起 RFC，说明变更动机、影响范围、向后兼容方案。
2. **公示**：公示期不少于 30 天，接受社区异议。
3. **评审**：经 Airymax 标准委员会评审，至少 1 名标准维护者审查通过。
4. **注册**：通过评审后写入对应主题文档，更新编号注册表，提升版本号。
5. **发布**：发布新版本标准，附带变更说明与迁移指南。

### 7.2 标准委员会

标准委员会由 SPHARX Ltd. 组织，包含：

- 标准总维护者（最终裁决权）
- 各主题域维护者（运行时、IPC、安全、记忆、认知、调度）
- 社区代表（每届任期 1 年，可连任）

委员会每季度召开 1 次例会，处理 RFC、异议、弃用提案。会议纪要公开发布。

### 7.3 弃用流程

弃用提案须包含：

- 弃用对象（编号 + 名称）
- 弃用理由
- 替代方案
- 弃用时间表（至少 2 个 MINOR 版本）

弃用对象在弃用期内保留功能但标记 `DEPRECATED`；弃用期满后从下一个 MAJOR 版本移除。

***

## 八、许可与引用

### 8.1 双许可证

本体系所有文档与配套测试套件遵循双许可证：

- **AGPL-3.0-or-later**：保障网络服务场景下的代码回流。
- **Apache-2.0**：允许商业闭源实现遵循本标准。

第三方可任选其一作为引用与实现本标准的法律基础。

### 8.2 引用规范

引用本体系标准时须注明：

- 标准名称
- 版本号
- 文档路径或永久链接

示例：

> 本实现遵循 Airymax Agent 运行时标准 1.0（AOS-STD-RT-001 \~ AOS-STD-RT-012），见 <https://docs.airymax.org/AirymaxStandards/01-airymax-agent-runtime-standard.md。>

### 8.3 商标

"Airymax Certified"、"Airymax Compatible"、"Airymax Inspired" 为 SPHARX Ltd. 商标。未经认证不得使用 "Certified" 标识；"Compatible" 与 "Inspired" 标识在通过对应兼容性测试后可自行声明。

***

## 九、术语速查

| 术语        | 标准名称                      | 含义                                 |
| --------- | ------------------------- | ---------------------------------- |
| 微核心       | MicroCore                 | agentrt 的 MicroCoreRT 是用户态微核心运行时   |
| 微内核       | Microkernel               | agentrt-linux 基于 Linux 6.6 的微内核化改造 |
| Agent 运行时 | Airymax Agent Runtime     | 见 01 标准                            |
| Agent 间通信 | AgentsIPC                 | 见 02 标准                            |
| 安全穹顶      | Cupolas                   | 见 03 标准                            |
| 记忆卷载      | MemoryRovol               | 见 04 标准                            |
| 认知三阶段循环   | CoreLoopThree             | 见 05 标准                            |
| 双思考系统     | Thinkdual                 | 见 05 标准                            |
| Agent 调度类 | SCHED\_AGENT / SCHED\_EXT | 见 06 标准                            |

***

## 十、参考文献

- agentrt-linux（AirymaxOS）统一术语表：`docs/AirymaxAgentOS/10-terminology.md`
- agentrt-linux 工程标准规范手册：`docs/AirymaxAgentOS/50-engineering-standards/00-engineering-standards-handbook.md`
- 五维正交 24 原则：`docs/AirymaxAgentOS/10-architecture/02-five-dimensional-principles.md`
- IPC 协议契约：`docs/AirymaxAgentOS/50-engineering-standards/20-contracts/ipc_protocol_contract.md`
- L1 核心运行时接口规范：`docs/AirymaxAgentOS/50-engineering-standards/30-runtime-interfaces/L1_runtime_interface.md`
- L3 安全与治理规范：`docs/AirymaxAgentOS/50-engineering-standards/30-runtime-interfaces/L3_security_governance.md`

***

## 十一、版本历史

| 版本  | 日期         | 变更                          |
| --- | ---------- | --------------------------- |
| 1.0 | 2026-07-09 | 初始草案：6 主题标准 + 总索引，奠定开放标准体系基础 |

***

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
