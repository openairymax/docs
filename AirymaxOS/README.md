Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 设计文档体系总览

> **文档定位**：agentrt-linux（AirymaxOS 极境智能体操作系统）整体方案文档体系的总入口与纲领\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **正式全称**：agentrt-linux（极境智能体操作系统，英文名：AirymaxOS）\
> **仓库别名**：agentrt-linux（仓库名）\
> **文档维护**：开源极境工程与规范委员会（OpenAirymax Engineering and Standardization Committee）

***

## 1. 模块概述

本文档是 `docs/AirymaxOS/` 目录的**顶层纲领**，承担四项核心职责：

1. **整体方案总览**：定义 agentrt-linux（AirymaxOS）作为基于 Linux 内核的智能体操作系统发行版的整体设计思想、架构支柱与子仓清单。
2. **技术选型声明**：明确声明 v1.x.x 阶段的五大核心技术选型（sched\_tac、IORING\_OP\_URING\_CMD、纯 C LSM、alloc\_pages + mmap、IRON-9 v3 四层模型），作为全文档体系的不可妥协基线。
3. **文档索引**：列出 `docs/AirymaxOS/` 下全部子目录及其文档数量，提供导航入口。
4. **Unify Design 映射**：阐述 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）在文档体系中的分布。

agentrt-linux 与 agentrt（AgentRT 极境智能体运行底座平台工程）同源：agentrt 是跨平台用户态运行时（Linux/macOS/Windows），agentrt-linux 是基于 Linux 6.6 内核基线的发行版（专为 Agent 工作负载优化），两者共享 Airymax 工程设计思想，agentrt 在 agentrt-linux 上运行天然更稳健和适配，架构同源，无适配层。

> **项目定位（极境内核标准）**：AirymaxOS（agentrt-linux）是**对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造的内核**——基于 Linux 6.6 内核基线，通过 seL4 微内核工程思想（Liedtke minimality principle、capability-based security、消息传递 IPC、服务用户态化）进行系统化改造，落地 v1.1 Capability Folding 单平面架构（~10ns fastpath Badge 校验）+ 12 daemon 用户态服务化 + 纯 C LSM + sched_tac 调度优化。

***

## 2. 技术选型声明

agentrt-linux v1.0 在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度做出**不可妥协**的技术选型。全文档体系的所有设计文档必须以本声明为基线，任何偏离均需通过 ADR 评审。

| # | 技术维度        | 选定方案                                                                                                                                                       | 明确不采用的方案                                                                  | 选定理由                                                                                      | 落地文档                                                                                         |
| - | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 1 | **内核调度**    | **sched\_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类，通过 `nice` / `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，**不定义新的调度类宏** | **不使用 sched\_ext**（不引入 eBPF 调度器、不依赖 `SCHED_EXT=7` 调度类、不使用 SCHED\_AGENT 宏） | 保留 Linux 6.6 主线调度器的稳定性与可维护性，避免 sched\_ext 实验性语义漂移；sched\_tac 通过既有调度类组合实现 Agent 感知，ABI 稳定  | `10-architecture/` + `20-modules/01-kernel.md` + `40-dataflows/04-scheduling-flow.md`        |
| 2 | **IPC 零拷贝** | **IORING\_OP\_URING\_CMD**：通过 io\_uring 命令操作码实现内核↔用户态零拷贝传输，固定 buffer + registered ring 路径                                                                  | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性）                                  | IORING\_OP\_URING\_CMD 在 Linux 6.6 已稳定，避免 page flipping 引发的 DMA 映射复杂性与跨架构兼容问题             | `20-modules/01-kernel.md` + `40-dataflows/03-ipc-flow.md`                                    |
| 3 | **安全钩子**    | **纯 C LSM**：以纯 C 实现的 Linux Security Module 注入策略钩子（`airy_lsm`），通过 `security_hook_list` 注册                                                                   | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子）                         | 纯 C LSM 保证安全策略的可审计性与形式化验证可行性，避免 BPF 验证器在安全关键路径的语义不确定性                                     | `20-modules/03-security.md` + `110-security/`                                                |
| 4 | **内存分配**    | **alloc\_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射到用户态地址空间                                                              | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存）                    | alloc\_pages + mmap 提供跨架构（x86/ARM/RISC-V）一致的内存语义，避免 DMA 一致性内存在不支持硬件一致性的平台上的兼容性问题          | `20-modules/04-memory.md` + `60-driver-model/`                                               |
| 5 | **同源代码共享**  | **IRON-9 v3 四层模型**：\[SC] 共享契约层 + \[SS] 语义同源层 + \[IND] 完全独立层 + \[DSL] 降级生存层                                                                                 | （v2 三层模型升级为 v3 四层模型，新增 \[DSL] 降级生存层）                                      | \[DSL] 降级生存层确保 \[SC] 头文件损坏/缺失时系统仍具备最小可运行子集，每个 \[SC] 头文件底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块 | `10-architecture/06-iron9-shared-model.md` + `10-architecture/11-degraded-survival-layer.md` |

### 2.1 IRON-9 v3 四层模型

| 层次    | 缩写     | 名称                             | 共享程度          | 内容                                                                                                                                                                                         | 组织方式                                       |
| ----- | ------ | ------------------------------ | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------ |
| 第 1 层 | \[SC]  | 共享契约层（Shared-Contract）         | 完全共享代码        | 10 个头文件（`include/uapi/linux/airymax/`）：`error.h` / `log_types.h` / `memory_types.h` / `security_types.h` / `cognition_types.h` / `sched.h` / `ipc.h` / `syscalls.h` / `uapi_compat.h` / `lsm_types.h` | `include/uapi/linux/airymax/` 独立头文件库，两端共同依赖           |
| 第 2 层 | \[SS]  | 语义同源层（Shared-Semantics）        | API 签名相同，实现独立 | 调度语义、安全模型、IPC 传输、记忆模型、认知循环                                                                                                                                                                 | 各自独立实现，通过 \[SC] 保证互操作                      |
| 第 3 层 | \[IND] | 完全独立层（Independent）             | 完全独立实现        | io\_uring fastpath、eBPF kfunc、Kbuild/Kconfig、内核模块构建                                                                                                                                        | 各自独立实现，通过 \[SC] 保证互操作                      |
| 第 4 层 | \[DSL] | 降级生存层（Degraded Survival Layer） | 自包含降级块        | `#ifdef AIRY_SC_FALLBACK` 降级块、最小可运行子集（38 POSIX 错误码 + printk LOG\_FATAL/ERROR + 最简 IPC）                                                                                                     | 每个 \[SC] 头文件底部的降级块，`.fallback_hashes` 独立校验 |

***

## 3. 子仓清单（8 个）

| # | 子仓                                                  | 中文    | 核心职责                                                                                  | 同源 agentrt                   |
| - | --------------------------------------------------- | ----- | ------------------------------------------------------------------------------------- | ---------------------------- |
| 1 | [kernel](10-architecture/01-system-architecture.md) | 极境内核  | Linux 6.6 内核基线 + sched\_tac + io\_uring（IORING\_OP\_URING\_CMD）+ 纯 C LSM + Rust 实验性支持 | atoms/corekern (MicroCoreRT) |
| 2 | [services](20-modules/02-services.md)               | 极境服务  | 用户态系统服务（VFS + 网络 + 驱动 + 12 daemons 集成）                                                | daemons                      |
| 3 | [security](20-modules/03-security.md)               | 极境安全  | capability 安全 + 纯 C LSM + 机密计算 + 国密                                                   | cupolas                      |
| 4 | [memory](20-modules/04-memory.md)                   | 极境记忆  | 记忆持久化 + CXL + PMEM + MGLRU 多代 LRU（alloc\_pages + mmap）                                | heapstore + memoryrovol      |
| 5 | [cognition](20-modules/05-cognition.md)             | 极境认知  | CoreLoopThree kthread + Wasm + LLM 调度 + 超节点沙箱                                         | coreloopthree + frameworks   |
| 6 | [cloudnative](20-modules/06-cloudnative.md)         | 极境云原生 | K8s + containerd + OCI + agentctl + 超节点 OS                                            | gateway + sdk                |
| 7 | [system](20-modules/07-system.md)                   | 极境系统  | 包管理 + 配置 + shell + 基础库 + DevStation                                                   | commons                      |
| 8 | [tests-linux](20-modules/08-tests-linux.md)         | 极境测试  | 单元测试 + 集成测试 + 形式化验证 + Soak + 混沌                                                       | 全模块测试                        |

***

## 4. 文档索引

`docs/AirymaxOS/` 下共 20 个子目录，覆盖需求、架构、模块、接口、数据流、工程标准、驱动、构建、测试、可观测性、运维、安全、开发流程、路线图、应用开发、云原生、兼容性、性能、国际化与发行版管理全维度。

```
docs/AirymaxOS/
├── README.md                            # 本文件（总览，v1.0）
├── 00-requirements/                     # 需求分析层（README + 3 文档）
├── 10-architecture/                     # 架构设计层（README + 11 文档）
├── 20-modules/                          # 模块设计层（README + 13 文档）
├── 30-interfaces/                       # 接口设计层（README + 文档）
├── 40-dataflows/                        # 数据流程设计层（README + 7 文档）
├── 50-engineering-standards/            # 工程标准规范（README + 顶层 8 文档 + 5 子目录 14 文档）
├── 60-driver-model/                     # 驱动模型（README + 2 文档）
├── 70-build-system/                     # 构建系统（README + 3 文档）
├── 80-testing/                          # 测试体系（README + 2 文档）
├── 90-observability/                    # 可观测性（README + 2 文档）
├── 100-operations/                      # 运维体系（README + 文档）
├── 110-security/                        # 安全加固（README + 文档）
├── 120-development-process/             # 开发流程（README + 文档）
├── 130-roadmap/                         # 开发路线图（README + 文档）
├── 140-application-development/         # Agent 应用开发（README + 文档）
├── 150-cloudnative/                     # 云原生部署（README + 文档）
├── 160-compatibility/                   # 兼容性（README + 文档）
├── 170-performance/                     # 性能工程（README + 文档）
├── 180-i18n/                            # 国际化（README + 文档）
└── 190-distribution/                    # 发行版管理（README + 文档）
```

### 4.1 文档分层说明

| 层级        | 模块                       | 文档数                         |
| --------- | ------------------------ | --------------------------- |
| **需求层**   | 00-requirements          | README + 3                  |
| **架构层**   | 10-architecture          | README + 11                 |
| **模块层**   | 20-modules               | README + 13                 |
| **接口层**   | 30-interfaces            | README + 文档                 |
| **数据流层**  | 40-dataflows             | README + 7                  |
| **工程标准**  | 50-engineering-standards | README + 8 顶层 + 5 子目录 14 文档 |
| **P0 模块** | 60-120（7 模块）             | README + 01-03              |
| **路线图**   | 130-roadmap              | README + 文档                 |
| **P1 模块** | 140-170（4 模块）            | README + 文档                 |
| **P2 模块** | 180-190（2 模块）            | README + 文档                 |

### 4.2 模块导航

#### 需求与设计层（00-40）

| 模块                                           | 描述                                                                                | 文档数 |
| -------------------------------------------- | --------------------------------------------------------------------------------- | --- |
| [00-requirements](00-requirements/README.md) | 需求分析（业务需求 + 功能需求 + 非功能需求）                                                         | 4   |
| [10-architecture](10-architecture/README.md) | 架构设计（系统架构 + Unify Design 总纲 + IRON-9 v3 + \[DSL] 降级层 + 五维原则 + 微内核策略 + ADR + 工程基线） | 12  |
| [20-modules](20-modules/README.md)           | 模块设计（8 子仓 + A-ULS/A-ULP/A-UCS + printk-bridge，\[SC] 10 头文件）                       | 14  |
| [30-interfaces](30-interfaces/README.md)     | 接口设计（系统调用 + AgentsIPC + SDK API + 编码标准）                                           | 文档  |
| [40-dataflows](40-dataflows/README.md)       | 数据流程（认知 + 记忆 + IPC + 调度 + Ring Buffer + Logger Daemon + Panic 生存）                 | 8   |

#### 工程标准与实施层（50-130）

| 模块                                                             | 描述                                                                       |
| -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| [50-engineering-standards](50-engineering-standards/README.md) | 工程标准规范（SSoT v2 + 编码风格 + 运行时接口 + 契约 + 集成 + ERP + 术语 + 跨项目共享 + \[SC] 类型桥接） |
| [60-driver-model](60-driver-model/README.md)                   | 驱动模型（device/driver/bus 三元组 + Agent 虚拟设备）                                 |
| [70-build-system](70-build-system/README.md)                   | 构建系统（Kbuild + Kconfig + CMake + RPM spec + CI/CD）                        |
| [80-testing](80-testing/README.md)                             | 测试体系（KUnit + kselftest + 集成测试 + sched\_tac 调度测试）                         |
| [90-observability](90-observability/README.md)                 | 可观测性（eBPF 探针 + tracepoint + user\_events + 日志观测 + 调度器状态观测）               |
| [100-operations](100-operations/README.md)                     | 运维体系（部署 + 配置管理）                                                          |
| [110-security](110-security/README.md)                         | 安全加固（纯 C LSM + Landlock 沙箱 + capability 模型）                              |
| [120-development-process](120-development-process/README.md)   | 开发流程（补丁生命周期 + 维护者层级）                                                     |
| [130-roadmap](130-roadmap/README.md)                           | 开发路线图（里程碑 + OS-ACC）                                                      |

#### 应用与生态层（140-190）

| 模块                                                                   | 描述                                   |
| -------------------------------------------------------------------- | ------------------------------------ |
| [140-application-development](140-application-development/README.md) | Agent 应用开发（多语言 SDK + Token 预算契约）     |
| [150-cloudnative](150-cloudnative/README.md)                         | 云原生部署（containerd + K8s CRD + 超节点 OS） |
| [160-compatibility](160-compatibility/README.md)                     | 兼容性（接口稳定性分级 + KABI + 跨发行版）           |
| [170-performance](170-performance/README.md)                         | 性能工程（Token 能效 + Agent 延迟 SLO）        |
| [180-i18n](180-i18n/README.md)                                       | 国际化（多语言 + 国密合规）                      |
| [190-distribution](190-distribution/README.md)                       | 发行版管理（RPM + ISO + Agent 应用商店）        |

***

## 5. Airymax Unify Design 映射

Airymax Unify Design 是 agentrt-linux v1.0 的**统一设计总纲**，定义五个横切模块，贯穿 agentrt-linux 的内核态与用户态、设计层与实现层。五模块在 `docs/AirymaxOS/` 各目录中的分布如下：

| Unify 模块  | 全称                                                | 核心职责                                                                               | 主要分布目录                                                                                                                                                                                                                   |
| --------- | ------------------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **A-UEF** | Unified Error and Fault Framework（统一错误码与故障定义体系）   | 错误码（`AIRY_E*`）+ 故障码（`AIRY_FAULT_*`）双轨制、UEF 错误码体系、`error.h` \[SC] 共享契约、错误码与故障码的统一管理 | `10-architecture/10-unify-design.md`（总纲）、`30-interfaces/08-sc-error-contract.md`、`40-dataflows/07-panic-survival-path.md`（Panic 时错误码记录）                                                                                  |
| **A-ULP** | Unified Logging and Printk Subsystem（统一日志与打印系统）   | Ring Buffer 日志、Logger Daemon、printk-bridge、Panic 生存路径                              | `20-modules/12-logger-daemon-module.md`、`20-modules/13-printk-bridge.md`、`40-dataflows/05-ring-buffer-logging.md`、`40-dataflows/06-logger-daemon-design.md`、`40-dataflows/07-panic-survival-path.md`、`90-observability/` |
| **A-UCS** | Unified Configuration Subsystem（统一配置管理体系）         | 统一配置管理体系、sysctl、Kconfig、airy\_defconfig                                            | `20-modules/11-unified-config.md`、`70-build-system/`（airy\_defconfig）                                                                                                                                                    |
| **A-ULS** | Unified Lifecycle Supervision Framework（统一生命周期管理） | 内核 Agent 监管、用户态监管守护进程、设备生命周期、调度器状态监管                                               | `20-modules/09-kernel-agent-supervisor.md`、`20-modules/10-user-supervisor-daemon.md`、`40-dataflows/04-scheduling-flow.md`、`60-driver-model/`（设备生命周期）、`90-observability/`（调度器状态观测）                                        |
| **A-IPC** | Unified Airymax IPC Fabric（统一进程间通信体系）             | io\_uring 零拷贝 IPC、128B 消息头、设备 DMA、IORING\_OP\_URING\_CMD                           | `40-dataflows/03-ipc-flow.md`、`20-modules/01-kernel.md`、`60-driver-model/`（io\_uring 设备 DMA）、`80-testing/`（IPC 零拷贝测试）                                                                                                    |

### 5.1 Unify Design 总纲

Unify Design 的总纲文档为 [`10-architecture/10-unify-design.md`](10-architecture/10-unify-design.md)，定义五模块的统一接口契约、跨模块协作关系与 IRON-9 v3 四层模型映射。

### 5.2 五模块在 20 个目录中的分布矩阵

| 目录                       | A-UEF           | A-ULP       | A-UCS              | A-ULS     | A-IPC            |
| ------------------------ | --------------- | ----------- | ------------------ | --------- | ---------------- |
| 00-requirements          | ✓               | ✓           | ✓                  | ✓         | ✓                |
| 10-architecture          | ✓（总纲）           | ✓           | ✓                  | ✓         | ✓                |
| 20-modules               | ✓（05-cognition） | ✓（12/13）    | ✓（11）              | ✓（09/10）  | ✓（01）            |
| 30-interfaces            | ✓               | ✓           | ✓                  | ✓         | ✓                |
| 40-dataflows             | ✓（01/04）        | ✓（05/06/07） | —                  | ✓（04）     | ✓（03）            |
| 50-engineering-standards | —               | ✓（日志契约）     | ✓（SSoT）            | —         | ✓（\[SC] 契约）      |
| 60-driver-model          | —               | —           | —                  | ✓（设备生命周期） | ✓（io\_uring DMA） |
| 70-build-system          | —               | —           | ✓（airy\_defconfig） | —         | —                |
| 80-testing               | ✓（调度测试）         | —           | —                  | —         | ✓（IPC 零拷贝测试）     |
| 90-observability         | —               | ✓（日志观测）     | —                  | ✓（调度器状态）  | —                |
| 100-operations           | —               | ✓           | ✓                  | ✓         | —                |
| 110-security             | —               | —           | —                  | ✓         | ✓                |
| 120-development-process  | —               | —           | —                  | —         | —                |
| 130-roadmap              | ✓               | ✓           | ✓                  | ✓         | ✓                |
| 140-190                  | ✓               | ✓           | ✓                  | ✓         | ✓                |

***

## 6. 仓库地址

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

***

## 7. 版本规划

| 版本    | agentrt-linux 范围                                                | agentrt 范围           |
| ----- | --------------------------------------------------------------- | -------------------- |
| v1.0  | 设计文档体系升级（sched\_tac + Airymax Unify Design + IRON-9 v3 + 新文档索引） | 与 agentrt-linux 协同验证 |
| 1.0.1 | 内核和 OS 实现                                                       | 与 agentrt-linux 协同验证 |

***

## 8. 前沿理论参考

| 前沿理论                                                    | 应用到子仓                        |
| ------------------------------------------------------- | ---------------------------- |
| seL4 微内核（形式化验证、capability、MCS、消息传递 IPC）                 | kernel + security + services |
| LionsOS（seL4 Microkit 生态）                               | kernel + system              |
| sDDF（seL4 设备驱动框架）                                       | kernel + services            |
| Linux 6.6 内核基线（EEVDF + MGLRU + eBPF kfunc + Rust 实验性支持） | kernel                       |
| sched\_tac（SCHED\_DEADLINE/SCHED\_FIFO/EEVDF 原生调度类组合）   | kernel（Agent 调度策略）           |
| io\_uring（IORING\_OP\_URING\_CMD 零拷贝）                   | kernel + services            |
| 纯 C LSM（security\_hook\_list 注册）                        | security                     |
| alloc\_pages + mmap（跨架构一致内存语义）                          | memory + driver-model        |
| MGLRU 多代 LRU（Linux 6.6 原生）                              | memory                       |
| Wasm 3.0（安全沙箱运行时）                                       | cognition                    |
| CXL（内存分层与池化）                                            | memory                       |

> **参考声明**：agentrt-linux 的微内核设计思想**唯一来源为 seL4**，工程实现标准完全对齐 Linux 6.6 内核基线。agentrt-linux 不移植 openEuler 特有特性，与 openEuler 的关系仅限于技术参考，不共享代码。

***

## 9. 与 agentrt 的关系

```
agentrt（AirymaxAgentRT，跨平台用户态运行时）
   │
   └── 运行在 agentrt-linux 上: 天然更稳健和适配（同源）
       │
       ├── agentrt 的 MicroCoreRT 调度 ←→ agentrt-linux 的sched_tac 调度策略（语义同源）
       ├── agentrt 的 AgentsIPC 协议 ←→ agentrt-linux 的 A-IPC（io_uring IORING_OP_URING_CMD，128B 消息头同源）
       ├── agentrt 的 Cupolas 安全 ←→ agentrt-linux 的 capability + 纯 C LSM（模型同源）
       ├── agentrt 的 MemoryRovol ←→ agentrt-linux 的 记忆子系统（alloc_pages + mmap，记忆模型同源）
       └── agentrt 的 CoreLoopThree ←→ agentrt-linux 的 A-UEF（认知 kthread，循环模型同源）
```

**同源红利**：agentrt 的设计假设和 agentrt-linux 的实现假设一致，agentrt 在 agentrt-linux 上运行**无适配层**，天然契合，更稳健。两端通过 IRON-9 v3 四层模型（\[SC]/\[SS]/\[IND]/\[DSL]）共享契约层代码与降级生存块。

***

## 10. 相关文档

- [需求分析层](00-requirements/README.md)：业务/功能/非功能需求
- [架构设计层](10-architecture/README.md)：系统架构 + Unify Design 总纲 + IRON-9 v3 + \[DSL] 降级层
- [模块设计层](20-modules/README.md)：8 子仓 + A-ULS/A-ULP/A-UCS
- [数据流程设计层](40-dataflows/README.md)：认知 + 记忆 + IPC + 调度 + 日志 + Panic 生存
- [工程标准规范](50-engineering-standards/README.md)：SSoT v2 + \[SC] 类型桥接
- [Airymax 架构设计原则](../AirymaxRT/10-architecture/00-architectural-principles.md)：五维正交 24 原则
- IRON-9 v3 工程铁律（含同源代码共享 + 类型桥接）

***

## 11. 版本历史

| 版本    | 日期         | 变更                                                                                                                                                                                                                                                         |
| ----- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0.1.1 | 2026-07-06 | 初始版本，建立 19 模块文档体系                                                                                                                                                                                                                                          |
| 0.1.1 | 2026-07-13 | 新增 07-directory-structure.md，\[SC] 头文件 Tab 8 缩进验证                                                                                                                                                                                                          |
| v1.0  | 2026-07-17 | 升级为 v1.0：新增sched\_tac 技术选型声明（不使用 sched\_ext）、IORING\_OP\_URING\_CMD（不使用 page flipping）、纯 C LSM（不使用 BPF LSM）、alloc\_pages + mmap（不使用 DMA 一致性内存）、IRON-9 v3 四层模型（新增 \[DSL] 降级生存层）；新增 Airymax Unify Design 五模块映射（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）；文档索引更新为 20 子目录 |

***

> **文档结束** | agentrt-linux v1.0 设计文档体系 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."

