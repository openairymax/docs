Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 设计文档
> **文档定位**：agentrt-linux（AirymaxOS）的全部设计思想、架构设计、子仓设计草案\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-13\
> **正式全称**：agentrt-linux（极境智能体操作系统，英文名：AirymaxOS）\
> **仓库别名**：agentrt-linux（仓库名）\
> **文档维护**：开源极境工程与规范委员会（OpenAirymax Engineering and Standardization Committee）

***

## 1. agentrt-linux 是什么

**agentrt-linux**（英文名：AirymaxOS，中文：极境智能体操作系统）是基于 Linux 内核的**操作系统发行版**，与 agentrt（AirymaxAgentRT / 极境智能体运行底座平台工程）**同源**。

**同源关系**:

- agentrt = 跨平台用户态运行时（Linux/macOS/Windows）
- agentrt-linux = 基于 Linux 内核的发行版（专为 Agent 工作负载优化）
- 两者共享 Airymax 设计理念（MicroCoreRT / AgentsIPC / Cupolas / MemoryRovol / CoreLoopThree 的语义）
- agentrt 在 agentrt-linux 上运行**天然更稳健和适配**（架构同源，无适配层）

## 2. 设计三大支柱

| 支柱                 | 思想                                                          | 理论根基                                  | 参考来源                                                |
| ------------------ | ----------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------ |
| **微内核设计思想**        | 最小化特权态代码、服务用户态化、消息传递通信、capability 安全                        | seL4（核心参考，ADR-014 唯一来源）               | seL4 源码（`01Reference/seL4-master`）    |
| **Linux 6.6 内核基线** | 基于 Linux 6.6 内核，sched\_ext + eBPF + io\_uring + Rust 微内核化改造 | Linux 内核工程基线（openEuler OLK-6.6 标杆）    | Linux 6.6 内核源码（`01Reference/kernel-OLK-6.6`） |
| **Airymax 同源性**    | 与 agentrt 共享设计理念，天然适配                                       | agentrt atoms/cupolas/coreloopthree 等 | IRON-9 v2 三层模型（\[SC]/\[SS]/\[IND]）落地                   |

## 3. 子仓清单（8 个）

| # | 子仓                                                  | 中文    | 核心职责                                                    | 同源 agentrt                   |
| - | --------------------------------------------------- | ----- | ------------------------------------------------------- | ---------------------------- |
| 1 | [kernel](10-architecture/01-system-architecture.md) | 极境内核  | Linux 内核 + 微内核化改造（sched\_ext + eBPF + io\_uring + Rust） | atoms/corekern (MicroCoreRT) |
| 2 | [services](20-modules/02-services.md)               | 极境服务  | 用户态系统服务（VFS + 网络 + 驱动 + 12 daemons 集成）                  | daemons                      |
| 3 | [security](20-modules/03-security.md)               | 极境安全  | capability 安全 + LSM + 机密计算 + 国密                         | cupolas                      |
| 4 | [memory](20-modules/04-memory.md)                   | 极境记忆  | 记忆持久化 + CXL + PMEM + MGLRU 多代 LRU                       | heapstore + memoryrovol      |
| 5 | [cognition](20-modules/05-cognition.md)             | 极境认知  | CoreLoopThree kthread + Wasm + LLM 调度 + 超节点沙箱           | coreloopthree + frameworks   |
| 6 | [cloudnative](20-modules/06-cloudnative.md)         | 极境云原生 | K8s + containerd + OCI + agentctl + 超节点 OS              | gateway + sdk                |
| 7 | [system](20-modules/07-system.md)                   | 极境系统  | 包管理 + 配置 + shell + 基础库 + DevStation                     | commons                      |
| 8 | [tests-linux](20-modules/08-tests-linux.md)               | 极境测试  | 单元测试 + 集成测试 + 形式化验证 + Soak + 混沌                         | 全模块测试                        |

## 4. 文档体系结构（19 模块）

```
docs/AirymaxOS/
├── README.md                        # 总览（本文件）
├── 00-requirements/                 # 需求分析层（4 文档）
├── 10-architecture/                 # 架构设计层（6 文档）
├── 20-modules/                      # 模块设计层（9 文档，8 子仓设计）
├── 30-interfaces/                   # 接口设计层（6 文档）
├── 40-dataflows/                    # 数据流程设计层（5 文档）
├── 50-engineering-standards/        # 工程标准规范（23 文档，含 5 子目录）
├── 60-driver-model/                 # 驱动模型（README + 01 + 02）
├── 70-build-system/                 # 构建系统（README + 01 + 02）
├── 80-testing/                      # 测试体系（README + 01 + 02）
├── 90-observability/                # 可观测性（README + 01 + 02）
├── 100-operations/                  # 运维体系（README + 01 + 02）
├── 110-security/                    # 安全加固（README + 01 + 02 + 03）
├── 120-development-process/         # 开发流程（README + 01 + 02）
├── 130-roadmap/                     # 开发路线图（7 文档）
├── 140-application-development/     # Agent 应用开发（8 文档）
├── 150-cloudnative/                # 云原生部署（6 文档）
├── 160-compatibility/               # 兼容性（6 文档）
├── 170-performance/                 # 性能工程（7 文档）
├── 180-i18n/                        # 国际化（6 文档）
└── 190-distribution/                # 发行版管理（6 文档）
```

### 4.1 文档分层说明

| 层级        | 模块                       | 文档数            |
| --------- | ------------------------ | -------------- |
| **需求层**   | 00-requirements          | 4 文档           |
| **架构层**   | 10-architecture          | 6 文档           |
| **模块层**   | 20-modules               | 9 文档           |
| **接口层**   | 30-interfaces            | 6 文档           |
| **数据流层**  | 40-dataflows             | 5 文档           |
| **工程标准**  | 50-engineering-standards | 23 文档          |
| **P0 模块** | 60-120（7 模块）             | README + 01-03 |
| **路线图**   | 130-roadmap              | 7 文档           |
| **P1 模块** | 140-170（4 模块）            | 6-8 文档/模块      |
| **P2 模块** | 180-190（2 模块）            | 6 文档/模块        |

### 4.2 模块导航

#### 需求与设计层（00-40）

| 模块                                           | 描述                                      | 文档数 |
| -------------------------------------------- | --------------------------------------- | --- |
| [00-requirements](00-requirements/README.md) | 需求分析（业务需求 + 功能需求 + 非功能需求 + 系统需求）        | 4   |
| [10-architecture](10-architecture/README.md) | 架构设计（系统架构 + 五维原则 + 微内核策略 + ADR + 工程基线）  | 6   |
| [20-modules](20-modules/README.md)           | 模块设计（8 子仓设计 + 模块概览）                     | 9   |
| [30-interfaces](30-interfaces/README.md)     | 接口设计（系统调用 + AgentsIPC + SDK API + 编码标准） | 6   |
| [40-dataflows](40-dataflows/README.md)       | 数据流程（调度流 + IPC 流 + 记忆卷载流 + 认知循环流）       | 5   |

#### 工程标准与实施层（50-130）

| 模块                                                             | 描述                                                              |
| -------------------------------------------------------------- | --------------------------------------------------------------- |
| [50-engineering-standards](50-engineering-standards/README.md) | 工程标准规范（编码 + 错误处理 + 内存 + 并发 + 开发流程 + 工具链 + 治理 + 验收，23 文档含 5 子目录） |
| [60-driver-model](60-driver-model/README.md)                   | 驱动模型（设备模型 + 平台驱动）                                               |
| [70-build-system](70-build-system/README.md)                   | 构建系统（Kbuild + Kconfig）                                          |
| [80-testing](80-testing/README.md)                             | 测试体系（KUnit + kselftest）                                         |
| [90-observability](90-observability/README.md)                 | 可观测性（ftrace + eBPF 探针）                                          |
| [100-operations](100-operations/README.md)                     | 运维体系（部署 + 配置管理）                                                 |
| [110-security](110-security/README.md)                         | 安全加固（LSM 框架 + Landlock 沙箱 + capability 模型）                      |
| [120-development-process](120-development-process/README.md)   | 开发流程（补丁生命周期 + 维护者层级）                                            |
| [130-roadmap](130-roadmap/README.md)                           | 开发路线图（9 Part + M0-M8 里程碑 + 110 项 OS-ACC）                        |

#### 应用与生态层（140-190）

| 模块                                                                   | 描述                                     |
| -------------------------------------------------------------------- | -------------------------------------- |
| [140-application-development](140-application-development/README.md) | Agent 应用开发（4 语言 16 嵌套客户端 + Token 预算契约） |
| [150-cloudnative](150-cloudnative/README.md)                         | 云原生部署（containerd + K8s CRD + 超节点 OS）   |
| [160-compatibility](160-compatibility/README.md)                     | 兼容性（4 层接口稳定性 + KABI + 跨发行版）            |
| [170-performance](170-performance/README.md)                         | 性能工程（Token 能效 + Agent 延迟 SLO）          |
| [180-i18n](180-i18n/README.md)                                       | 国际化（多语言 + 国密合规）                        |
| [190-distribution](190-distribution/README.md)                       | 发行版管理（RPM + ISO + Agent 应用商店）          |

## 5. 仓库地址

所有 agentrt-linux 仓库归属 `openairymax` 组织，托管于 atomgit.com：

| 仓库   | URL                                             |
| ---- | ----------------------------------------------- |
| 管理仓  | `https://atomgit.com/openairymax/agentrt-linux` |
| 子仓 1 | `https://atomgit.com/openairymax/kernel`        |
| 子仓 2 | `https://atomgit.com/openairymax/services`      |
| 子仓 3 | `https://atomgit.com/openairymax/security`      |
| 子仓 4 | `https://atomgit.com/openairymax/memory`        |
| 子仓 5 | `https://atomgit.com/openairymax/cognition`     |
| 子仓 6 | `https://atomgit.com/openairymax/cloudnative`   |
| 子仓 7 | `https://atomgit.com/openairymax/system`        |
| 子仓 8 | `https://atomgit.com/openairymax/tests-linux`   |

## 6. 版本规划

| 版本    | agentrt-linux 范围                           | agentrt 范围                 |
| ----- | ------------------------------------------ | -------------------------- |
| 0.1.1 | 设计文档体系（README + 完整的设计草案 + 工程标准规范 + 工程基线声明） | 全部三大支柱（奠基 + 29 仓拆分 + 生产就绪） |
| 1.0.1 | 内核和 OS 实现                                  | 与 agentrt-linux 协同验证       |

## 7. 前沿理论参考

| 前沿理论                                                    | 应用到子仓                        |
| ------------------------------------------------------- | ---------------------------- |
| seL4 微内核（形式化验证、capability、MCS 2026.6.29 验证完成、消息传递 IPC）  | kernel + security + services |
| LionsOS（seL4 Microkit 生态，2026）                          | kernel + system              |
| sDDF（seL4 设备驱动框架，2026）                                  | kernel + services            |
| Linux 6.6 内核基线（EEVDF + MGLRU + eBPF kfunc + Rust 实验性支持） | kernel                       |
| sched\_ext（eBPF 用户态调度器、sub-scheduler）                   | kernel（Agent 调度策略）           |
| io\_uring（零 syscall 高性能 I/O）                            | kernel + services            |
| eBPF 代码完整性 + 机密计算                                        | security                     |
| MGLRU 多代 LRU（Linux 6.6 原生）                              | memory                       |
| Wasm 3.0（安全沙箱运行时）                                       | cognition                    |
| CXL（内存分层与池化）                                            | memory                       |

> **参考声明**：agentrt-linux 的微内核设计思想**唯一来源为 seL4**（参考 `01Reference/seL4-master`），工程实现标准完全对齐 Linux 6.6 内核基线（参考 `01Reference/kernel-OLK-6.6`，即 openEuler OLK-6.6 内核源代码）。agentrt-linux 不移植 openEuler 特有特性，与 openEuler 的关系仅限于技术参考，不共享代码。

## 8. 与 agentrt 的关系

```
agentrt（AirymaxAgentRT，跨平台用户态运行时）
   │
   └── 运行在 agentrt-linux 上: 天然更稳健和适配（同源）
       │
       ├── agentrt 的 MicroCoreRT 调度 ←→ agentrt-linux 的 Agent 调度策略（语义同源）
       ├── agentrt 的 AgentsIPC 协议 ←→ agentrt-linux 的 IPC 子系统（128B 消息头同源）
       ├── agentrt 的 Cupolas 安全 ←→ agentrt-linux 的 capability + LSM（模型同源）
       ├── agentrt 的 MemoryRovol ←→ agentrt-linux 的 记忆子系统（记忆模型同源）
       └── agentrt 的 CoreLoopThree ←→ agentrt-linux 的 认知 kthread（循环模型同源）
```

**同源红利**: agentrt 的设计假设和 agentrt-linux 的实现假设一致，agentrt 在 agentrt-linux 上运行**无适配层**，天然契合，更稳健。

## 9. 相关文档

- RT 设计文档: 内部参考文档（仅授权人员访问）
- OS 设计文档: `docs/AirymaxOS/`（本目录，开源）
- 工程规范: IRON-9 v2 工程铁律（含同源代码共享）
- 微内核参考: 内部参考文档（仅授权人员访问）

***

> **文档结束** | agentrt-linux 0.1.1 设计文档体系 | 维护者：开源极境工程与规范委员会

