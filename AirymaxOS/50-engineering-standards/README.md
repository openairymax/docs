Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）工程标准规范
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）工程标准规范的主索引与总纲，含 SSoT v2 注册表、编码风格、运行时接口、契约、集成、ERP、术语、跨项目共享与 [SC] 类型桥接\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **同源映射**：agentrt `docs/AirymaxRT/10-architecture/00-architectural-principles.md`（五维正交 24 原则）+ IRON-9 v3 工程铁律（四层模型，17 类规则编号体系）\
> **理论根基**：Linux 6.6 内核工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）\
> **上级文档**：[agentrt-linux 总览](../README.md)

---

## 1. 模块概述

本文档是 `50-engineering-standards/` 目录的主索引与总纲，定义 agentrt-linux 的工程标准体系：

1. **SSoT v2 注册表**：`09-ssot-registry.md` 作为全局唯一规则编号注册表（唯一权威），登记全部规则编号、废弃文档、术语。SSoT v2 取消三层镜像，只保留一个 SSoT 文档。
2. **[SC] 类型桥接**：`11-sc-header-type-bridging.md` 定义 IRON-9 v3 [SC] 层内部类型桥接规则（通过 uapi_compat.h 实现，非独立层），解决 [SC] 头文件在 C/Rust/用户态/内核态之间的类型映射问题。
3. **跨项目代码共享**：`120-cross-project-code-sharing.md` 定义 IRON-9 v3 四层模型（[SC]/[SS]/[IND]/[DSL]），作为 agentrt ↔ agentrt-linux 同源代码共享的工程基线。
4. **4 主题 + 5 子目录**：代码规范（01）、工程思想（04）、开发流程（05）、维护者治理（07）四大主题 + 10-coding-style / 20-contracts / 30-runtime-interfaces / 40-integration / 50-project-erp 五个子目录。

---

## 2. 技术选型声明

本目录的工程标准以 agentrt-linux v1.0 五大技术选型为基线：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 工程标准影响 |
|---|---------|---------|----------------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 SCHED_AGENT 宏） | 编码规范禁止引用 sched_ext API；K-4 可插拔策略原则以sched_tac 为例 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：io_uring 命令操作码零拷贝传输 | **不使用 page flipping** | 契约规范（`20-contracts/`）的 IPC 协议契约基于 IORING_OP_URING_CMD |
| 3 | **安全钩子** | **纯 C LSM**：纯 C 实现 `airy_lsm`，通过 `security_hook_list` 注册 | **不使用 BPF LSM** | 编码规范禁止引用 BPF LSM 框架；安全编码强制纯 C LSM 可审计性 |
| 4 | **内存分配** | **alloc_pages + mmap**：物理页分配后映射到用户态 | **不使用 DMA 一致性内存** | 编码规范禁止 `dma_alloc_coherent` 在跨架构场景使用；E-3 资源确定性以 alloc_pages + mmap 为基线 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] + [SS] + [IND] + [DSL] | （v2 三层模型升级，新增 [DSL] 降级生存层） | `11-sc-header-type-bridging.md` 定义 [SC] 层内部类型桥接规则（通过 uapi_compat.h）；`120-cross-project-code-sharing.md` 升级为 v3 四层模型；SSoT v2（`09-ssot-registry.md`）登记 [DSL] 相关规则编号 |

---

## 3. 文档索引

本目录由顶层核心文档（9 文件）+ [SC] 类型桥接文档（1 文件）+ 5 个子目录（14 文档）构成，共 24 个文档（含本 README）：

### 3.1 顶层核心文档（10 文件）

| # | 文档 | 核心内容 | 版本 | 状态 |
|---|------|---------|------|------|
| 1 | [00-engineering-standards-handbook.md](00-engineering-standards-handbook.md) | 工程标准规范手册（SSoT 索引 + IRON 铁律 + 合规禁词 + 美学审查） | v1.0 | 维护中 |
| 2 | [01-coding-standards.md](01-coding-standards.md) | 代码规范合集（6 Parts：语义+格式+风格+Checkpatch+kernel-doc+clang-format） | v1.0 | 维护中 |
| 3 | [04-engineering-philosophy.md](04-engineering-philosophy.md) | 工程思想（双层稳定性/策略机制分离/渐进式/审查优先） | v1.0 | 维护中 |
| 4 | [05-development-process.md](05-development-process.md) | 开发流程合集（4 Parts：流程+工具链+合规检查+SPDX 许可证） | v1.0 | 维护中 |
| 5 | [07-maintainers-and-governance.md](07-maintainers-and-governance.md) | 维护者制度与治理（MAINTAINERS/层级/成熟度模型/DCO/CODEOWNERS） | v1.0 | 维护中 |
| 6 | [09-ssot-registry.md](09-ssot-registry.md) | **SSoT v2 全局规则编号注册表**（唯一权威，所有编号登记于此） | v2.0 | 维护中 |
| 7 | [11-sc-header-type-bridging.md](11-sc-header-type-bridging.md) | **[SC] 类型桥接**（[SC] 层内部，通过 uapi_compat.h 实现：C↔Rust FFI + 内核态↔用户态类型映射 + errno 跨语言映射） | v1.0 | 维护中 |
| 8 | [90-terminology.md](90-terminology.md) | 术语表 SSoT | v1.0 | 维护中 |
| 9 | [120-cross-project-code-sharing.md](120-cross-project-code-sharing.md) | 跨项目代码共享（IRON-9 v3 [SC]/[SS]/[IND]/[DSL] 四层模型） | v1.0 | 维护中 |

### 3.2 子目录文档（5 目录 = 14 文档）

| 子目录 | 文档 | 核心内容 | 版本 |
|--------|------|---------|------|
| [10-coding-style/](10-coding-style/README.md) | README.md | 编码规范总览与导航 | v1.0 |
| | C_Cpp_coding_style.md | C/C++ 编码风格规范（Part I-IV：内核 C 风格+补充+C++ 风格+安全编码） | v1.0 |
| | Rust_coding_style.md | Rust 编码风格规范（Part I-II：风格+安全编码） | v1.0 |
| | Go_coding_style.md | Go 编码风格规范（Part I-II：风格+安全编码） | v1.0 |
| | scripting_coding_style.md | 脚本语言规范（Part I-III：Python+JavaScript/TS+Java） | v1.0 |
| | coding_conventions.md | 通用编码约定（Part I-V：注释+命名+日志+前缀+安全设计） | v1.0 |
| [20-contracts/](20-contracts/README.md) | README.md | 契约规范总览与导航 | v1.0 |
| | contracts.md | 契约规范合集（A-IPC IPC 协议+日志+系统调用 API 契约） | v1.0 |
| [30-runtime-interfaces/](30-runtime-interfaces/README.md) | README.md | 运行时接口总览与导航 | v1.0 |
| | runtime_interfaces.md | 运行时接口合集（L1 运行时接口+L2 服务协议+L3 安全治理） | v1.0 |
| [40-integration/](40-integration/README.md) | README.md | 集成规范总览与导航 | v1.0 |
| | integration.md | 集成标准合集（agentrt 集成+生态伙伴+配置集成+标准贡献） | v1.0 |
| [50-project-erp/](50-project-erp/README.md) | README.md | 项目工程管理总览与导航 | v1.0 |
| | project_erp.md | 项目管理规范合集（SBOM+错误码+模块需求+资源管理） | v1.0 |

### 3.3 目录结构

```
50-engineering-standards/
├── README.md                            # 本文件 — 主索引与总纲（v1.0）
├── 00-engineering-standards-handbook.md # 工程标准规范手册（SSoT 索引 + IRON 铁律）
├── 01-coding-standards.md               # 代码规范合集（6 Parts）
├── 04-engineering-philosophy.md         # 工程思想
├── 05-development-process.md            # 开发流程合集（4 Parts）
├── 07-maintainers-and-governance.md     # 维护者制度与治理
├── 09-ssot-registry.md                  # SSoT v2 全局规则编号注册表（唯一权威）
├── 11-sc-header-type-bridging.md        # [SC] 类型桥接（[SC] 层内部，通过 uapi_compat.h 实现）
├── 90-terminology.md                    # 术语表 SSoT
├── 120-cross-project-code-sharing.md    # 跨项目代码共享（IRON-9 v3 四层模型）
│
├── 10-coding-style/                      # 编码规范子目录（6 文档）
├── 20-contracts/                         # 契约规范子目录（2 文档）
├── 30-runtime-interfaces/                # 运行时接口子目录（2 文档）
├── 40-integration/                       # 集成规范子目录（2 文档）
└── 50-project-erp/                       # 项目工程管理子目录（2 文档）
```

---

## 4. Airymax Unify Design 映射

工程标准规范为 Airymax Unify Design 五模块提供工程基线与契约保障：

| Unify 模块 | 工程标准文档 | 核心约束 |
|-----------|------------|---------|
| **A-UEF**（统一错误码与故障定义体系） | `01-coding-standards.md`（热路径 C 编码规范）+ `30-runtime-interfaces/`（运行时接口） | CoreLoopThree kthread 热路径必须遵循 C 内核风格；sched_tac 调度策略可插拔 |
| **A-ULP**（统一日志与打印系统） | `20-contracts/contracts.md`（日志契约）+ `30-runtime-interfaces/`（日志运行时接口） | Ring Buffer 日志契约化；Logger Daemon 接口契约化；Panic 生存路径可审计 |
| **A-UCS**（统一配置管理体系） | `09-ssot-registry.md`（SSoT v2，配置规则编号登记）+ `30-runtime-interfaces/`（配置接口） | sysctl/Kconfig/airy_defconfig 配置规则统一登记于 SSoT v2 |
| **A-ULS**（统一生命周期管理） | `07-maintainers-and-governance.md`（维护者层级）+ `30-runtime-interfaces/`（监管接口） | 监管器生命周期管理契约化；故障重启策略可审计 |
| **A-IPC**（统一进程间通信体系） | `11-sc-header-type-bridging.md`（[SC] 类型桥接，IPC 头文件类型映射）+ `20-contracts/contracts.md`（IPC 契约） | [SC] `ipc.h` 头文件的 C↔Rust 类型桥接规则；128B 消息头契约化；IORING_OP_URING_CMD 命令码契约化 |

### 4.1 SSoT v2 与 Unify Design 的关系

`09-ssot-registry.md`（SSoT v2）作为全局唯一规则编号注册表，登记 Unify Design 五模块相关的全部规则编号：

| Unify 模块 | SSoT v2 登记的规则编号前缀 |
|-----------|----------------------|
| A-UEF | OS-KER（内核调度规则）+ OS-STD（编码标准） |
| A-ULP | OS-STD（日志契约）+ OS-OBS（可观测性规则） |
| A-UCS | OS-BUILD（构建配置规则）+ OS-STD（配置标准） |
| A-ULS | OS-KER（内核监管规则）+ OS-SEC（安全规则） |
| A-IPC | OS-KER（内核 IPC 规则）+ OS-STD（IPC 契约） |

### 4.2 [SC] 类型桥接与 Unify Design 的关系

`11-sc-header-type-bridging.md` 定义 IRON-9 v3 [SC] 层内部类型桥接规则，为 Unify Design 五模块的 [SC] 头文件提供跨语言类型映射：

| [SC] 头文件 | [SC] 类型桥接规则 | 关联 Unify 模块 |
|-----------|----------------|---------------|
| `sched.h` | `struct airy_task_descr` 在 C↔Rust 之间的 FFI 映射 + vtime `__s32` Q16.16 定点跨语言一致 | A-UEF + A-ULS |
| `ipc.h` | `struct airy_ipc_msg_hdr` 128B 消息头在 C↔Rust 之间的内存布局一致 + magic 0x41524531 跨语言校验 | A-IPC |
| `log_types.h` | 日志级别枚举在 C↔Rust 之间的映射 + Ring Buffer 数据结构跨语言一致 | A-ULP |
| `security_types.h` | capability 41 ID + LSM 钩子 250 ID 在 C↔Rust 之间的映射 | A-ULS |
| `cognition_types.h` | CoreLoopThree 阶段枚举 + Thinkdual 模式枚举在 C↔Rust 之间的映射 | A-UEF |
| `syscalls.h` | `AIRY_SYS_*` 编号在 C↔Rust 之间的一致性 + UAPI 兼容性 | A-UCS |

---

## 5. IRON-9 v3 四层模型（跨项目代码共享）

`120-cross-project-code-sharing.md` 定义 IRON-9 v3 四层模型，作为 agentrt ↔ agentrt-linux 同源代码共享的工程基线：

| 层次 | 缩写 | 名称 | 共享程度 | 内容 |
|------|------|------|---------|------|
| 第 1 层 | [SC] | 共享契约层（Shared-Contract） | 完全共享代码 | 10 个头文件（`include/uapi/linux/airymax/`）：`error.h` / `log_types.h` / `memory_types.h` / `security_types.h` / `cognition_types.h` / `sched.h` / `ipc.h` / `syscalls.h` / `uapi_compat.h` / `lsm_types.h` |
| 第 2 层 | [SS] | 语义同源层（Shared-Semantics） | API 签名相同，实现独立 | 调度语义（sched_tac）、安全模型（纯 C LSM）、IPC 传输（IORING_OP_URING_CMD）、记忆模型（alloc_pages + mmap）、认知循环（CoreLoopThree） |
| 第 3 层 | [IND] | 独立实现层（Independent） | 完全独立 | agentrt-linux 专属内核驱动/Kbuild/systemd/纯 C LSM；agentrt 专属跨平台用户态运行时/SDK |
| 第 4 层 | [DSL] | 降级生存层（Degraded Survival Layer） | [SC] 损坏时自包含降级 | `#ifdef AIRY_SC_FALLBACK` 降级生存块，提供 [SC] 损坏时最小可运行子集（详见 `11-degraded-survival-layer.md`） |

### 5.1 与旧版 IRON-9 v3 的差异

| 维度 | IRON-9 v2（三层模型） | IRON-9 v3（四层模型） |
| ----- | --------------------- | ------------------------------------------ |
| 层次数 | 3 层（[SC]/[SS]/[IND]） | 4 层（[SC]/[SS]/[IND]/[DSL]） |
| 降级生存 | 无独立层次，[SC] 损坏即 brick | **新增 [DSL] 降级生存层**，提供 [SC] 损坏时自包含降级生存块（`#ifdef AIRY_SC_FALLBACK`） |
| 多语言支持 | 弱（C 为主，Rust 隐式） | 强（C/Rust/Python/TS 显式类型桥接） |
| CI 校验 | 契约层变更双向校验 | 契约层 + 类型桥接层双向校验（增强） |

---

## 6. OS 工程规则编号体系

agentrt-linux 在 agentrt 17 类规则编号体系基础上，新增 OS 专属编号前缀：

| 编号前缀 | 含义 | 与 agentrt 的关系 |
| -------- | -------- | --------------------------------------- |
| **OS-IRON** | agentrt-linux 工程铁律（不可妥协） | 继承 agentrt IRON-1~10 + 新增 OS-IRON-1~N |
| **OS-BAN** | agentrt-linux 禁止规则 | 继承 agentrt BAN-1~356 + 新增 OS-BAN-1~N |
| **OS-STD** | agentrt-linux 标准规则 | 继承 agentrt STD + OS 专属扩展 |
| **OS-ACC** | agentrt-linux 验收标准 | 继承 agentrt ACC-1~149 + OS 专属验收 |
| **OS-KER** | agentrt-linux 内核工程规则 | 全新（agentrt 不涉及内核态） |
| **OS-DRV** | agentrt-linux 驱动工程规则 | 全新 |
| **OS-BUILD** | agentrt-linux 构建工程规则 | 全新 |
| **OS-TEST** | agentrt-linux 测试工程规则 | 全新 |
| **OS-OBS** | agentrt-linux 可观测性规则 | 全新 |
| **OS-SEC** | agentrt-linux 安全工程规则 | 全新 |
| **OS-ABI** | agentrt-linux ABI 稳定性规则 | 全新 |

具体规则编号在各个子文档中定义，汇总于 `09-ssot-registry.md`（SSoT v2 唯一权威注册表）。

---

## 7. 章节定位

| 章节 | 核心问题 | 来源 |
|------|---------|------|
| 00 工程标准手册 | "标准体系怎么索引"——SSoT 索引 + IRON 铁律 + 合规禁词 + 美学审查 | Linux `coding-style.rst` + Airymax IRON 体系 |
| 01 代码规范合集 | "代码该怎么写 + 长什么样 + 怎么思考"——6 Parts 合并 | Linux `coding-style.rst` + `.clang-format` + `checkpatch.pl` + Airymax 15 文件 |
| 04 工程思想 | "工程体系怎么建立"——稳定性哲学、策略机制分离、渐进式、审查优先 | Linux `stable-api-nonsense.rst` + Airymax K-1~K-4 + E-1~E-8 |
| 05 开发流程合集 | "代码怎么进入主线 + 规范怎么执行 + 怎么验证 + 许可证怎么标注"——4 Parts 合并 | Linux `development-process.rst` + `submit-checklist.rst` + `license-rules.rst` + Airymax 17 类规则 |
| 07 维护者制度与治理 | "项目怎么被治理"——MAINTAINERS、层级、成熟度模型、DCO | Linux `MAINTAINERS` + Airymax IRON-9 v3 同源且部分代码共享 |
| 09 SSoT v2 注册表 | "编号归谁管"——全局唯一规则编号注册表（唯一权威） | Airymax SSoT v2 体系 |
| 11 [SC] 类型桥接 | "[SC] 头文件怎么跨语言映射"——IRON-9 v3 [SC] 层内部类型桥接规则 | Airymax IRON-9 v3 [SC] 层内部类型桥接（通过 uapi_compat.h） |
| 90 术语表 | "术语怎么用"——微核心/微内核/MicroCoreRT/sched_tac 等术语规范 | Airymax 术语体系 |
| 120 跨项目代码共享 | "agentrt ↔ agentrt-linux 怎么共享代码"——IRON-9 v3 [SC] 10 头文件 + [DSL] 降级生存层 + 四层模型 | Airymax IRON-9 v3 四层模型 |

---

## 8. 与 Airymax 五维正交 24 原则的映射

| 五维原则 | 在 agentrt-linux 工程标准中的体现 | 落地章节 |
| -------- | ----------------------------------------------------------- | ---------------------- |
| **S-1 反馈闭环** | 7 层自动化验证的每一层都是反馈闭环；CI 失败即反馈 | 05 开发流程 Part II |
| **S-2 层次分解** | 4 层接口稳定性分级（Agent API / Runtime API / 内核子系统 API / 内部实现） | 04 工程思想 |
| **S-3 总体设计部** | 维护者层级制度（Lieutenant System）+ 总维护者角色 | 07 维护者制度与治理 |
| **S-4 涌现性管理** | 渐进式开发模型 + Merge Window + RC 周期 | 05 开发流程 |
| **K-1 内核极简** | 微内核化改造策略（用户态服务化）+ 内核职责最小化 | 04 工程思想 |
| **K-2 接口契约化** | 双层稳定性哲学（用户 ABI 稳定 / 内部 API 可重构） | 04 工程思想 |
| **K-3 服务隔离** | 驱动用户态化 + 服务进程隔离 + capability 安全 | 01 代码规范 Part III + 110-security |
| **K-4 可插拔策略** | sched_tac 调度策略可插拔 + 纯 C LSM 钩子 + 模块签名 | 01 代码规范 Part III |
| **C-1 双系统协同** | 快慢路径分层（热路径 C / 慢路径 Rust）+ 调度类可插拔 | 01 代码规范 Part III |
| **C-2 增量演化** | 补丁序列中点可编译运行 + git bisect 友好 | 05 开发流程 |
| **C-3 记忆卷载** | 多代 LRU（MGLRU）+ 分级内存（CXL/PMEM，alloc_pages + mmap） | 170-performance |
| **C-4 遗忘机制** | 弃用接口清单（Deprecated Interfaces Registry）+ ABI 弃用流程 | 09-ssot-registry.md §8.2 |
| **E-1 安全内生** | 强制安全编码 + fault injection + 纯 C LSM 集成 | 01 代码规范 + 110-security |
| **E-2 可观测性** | ftrace + eBPF + perf + 4 层文件系统接口 | 90-observability |
| **E-3 资源确定性** | 引用计数强制 + devm_ 资源管理 + alloc_pages + mmap 内存分配惯用法 | 01 代码规范 Part I + Part III |
| **E-4 跨平台一致性** | 用户态跨 Linux/macOS/Windows + 内核态 Linux 6.6 基线 | 04 工程思想 |
| **E-5 命名语义化** | 全局函数描述性命名 + 局部变量短小 + 敏感术语禁用 | 01 代码规范 |
| **E-6 错误可追溯** | Fixes:/Closes:/Link: 标签 + Signed-off-by DCO 链 + 12 字符 SHA | 05 开发流程 |
| **E-7 文档即代码** | kernel-doc 强制 + ABI 文档化 + `make htmldocs` 验证 | 01 代码规范 |
| **E-8 可测试性** | KUnit + kselftest + fault injection + 覆盖率门槛 | 80-testing |
| **A-1 极简主义** | 反过度抽象 + 反 inline 滥用 + 反宏滥用 | 01 代码规范 Part III |
| **A-2 细节关注** | 行尾禁止空白 + 函数原型元素顺序 + 指针声明贴名 | 01 代码规范 Part II |
| **A-3 人文关怀** | 敏感术语禁用 + 审查礼仪 + 不烧桥管理哲学 | 01 代码规范 + 07 维护者制度 |
| **A-4 完美主义** | 7 层验证 + 24 项提交检查清单 + Reviewed-by Statement | 05 开发流程 Part II + Part I |

---

## 9. 相关文档

### 9.1 同源 Airymax 文档

- `docs/AirymaxRT/10-architecture/00-architectural-principles.md`（五维正交 24 原则）
- `docs/AirymaxOS/50-engineering-standards/10-coding-style/`（编码规范子目录，6 文档）
- `docs/AirymaxOS/50-engineering-standards/90-terminology.md`（术语规范 SSoT）
- IRON-9 v3 工程铁律（17 类规则编号体系 + [DSL] 降级生存层）

### 9.2 agentrt-linux 设计文档

- [agentrt-linux 总览](../README.md)：v1.0 设计文档体系总览与技术选型声明
- [架构设计](../10-architecture/README.md)：系统架构 + Unify Design 总纲 + IRON-9 v3 + [DSL] 降级层
- [模块设计](../20-modules/README.md)：8 子仓 + A-ULS/A-ULP/A-UCS + [SC] 10 头文件
- [数据流程设计](../40-dataflows/README.md)：A-UEF/A-IPC/A-ULS/A-ULP 数据流路径
- [驱动模型](../60-driver-model/README.md)：设备生命周期（A-ULS）+ io_uring DMA（A-IPC）
- [构建系统](../70-build-system/README.md)：airy_defconfig（A-UCS）
- [测试体系](../80-testing/README.md)：sched_tac 调度测试 + IPC 零拷贝测试
- [可观测性](../90-observability/README.md)：日志观测（A-ULP）+ 调度器状态观测（A-ULS）

### 9.3 参考材料

- Linux 6.6 内核源码（深度研究基础）
- Linux 内核 `Documentation/process/`（coding-style.rst、submitting-patches.rst、stable-api-nonsense.rst 等）
- Linux 内核 `.clang-format`（689 行 + 560 ForEachMacros）
- Linux 内核 `MAINTAINERS`（734KB，维护者制度范本）

---

## 10. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.3 | 2026-07-13 | 50-engineering-standards 精简合并 53→23 文档 |
| v1.4 | 2026-07-13 | D 类 OLK-6.6 工程标准 8 项差距移交 1.0.1 M1+；修正子目录文档计数 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）、IORING_OP_URING_CMD（不使用 page flipping）、纯 C LSM（不使用 BPF LSM）、alloc_pages + mmap（不使用 DMA 一致性内存）；IRON-9 v2 升级为 v3 四层模型（新增 [DSL] 降级生存层）；新增 `11-sc-header-type-bridging.md`（[SC] 类型桥接文档）；SSoT 升级为 v2（`09-ssot-registry.md`）；`120-cross-project-code-sharing.md` 升级为 IRON-9 v3 四层模型；新增 Airymax Unify Design 五模块工程标准映射（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | 共 24 文档（含本 README + 顶层 9 文档 + [SC] 类型桥接 1 文档 + 5 子目录 14 文档）| v1.0 | 2026-07-17
