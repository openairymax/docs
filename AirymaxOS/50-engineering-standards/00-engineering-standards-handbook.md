Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）工程标准规范手册
> **文档定位**：交付成果 #3——工程标准规范手册。本文档是 agentrt-linux（AirymaxOS，极境智能体操作系统）工程标准的执行依据，汇总 7 主题文档的核心规范条目、SSoT 规则编号注册表、IRON-9 v3 四层共享模型、合规性禁词清单与乔布斯美学审查标准，作为开发的强制执行依据。\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-12\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）+ seL4 微内核设计思想 + 主流 Linux 工程标准规范参考\
> **配套文档**：comprehensive-review-report.md（交付成果 #1）、revision-changelog.md（交付成果 #2）\
> **权威性**：本手册为工程标准的**单一事实源（SSoT）**索引。当各主题文档与本手册冲突时，以本手册为准；当本手册与 SSoT 注册表（[09-ssot-registry.md](./09-ssot-registry.md)）冲突时，以 SSoT 注册表为准。新规则须按 [09-ssot-registry.md §1.4](./09-ssot-registry.md) 流程申请注册。

---

## 目录

- [第 1 章 工程标准体系总览](#第-1-章-工程标准体系总览)
- [第 2 章 IRON 工程铁律（不可妥协）](#第-2-章-iron-工程铁律不可妥协)
- [第 3 章 SSoT 规则编号注册表](#第-3-章-ssot-规则编号注册表)
- [第 4 章 IRON-9 v3 四层共享模型与跨项目代码共享机制](#第-4-章-iron-9-v2-三层共享模型与跨项目代码共享机制)
- [第 5 章 7 层自动化验证体系](#第-5-章-7-层自动化验证体系)
- [第 6 章 合规性禁词清单与命名体系规范](#第-6-章-合规性禁词清单与命名体系规范)
- [第 7 章 微核心与微内核术语规范](#第-7-章-微核心与微内核术语规范)
- [第 8 章 乔布斯美学"无隐藏瑕疵"审查标准](#第-8-章-乔布斯美学无隐藏瑕疵审查标准)
- [第 9 章 Linux 6.6 内核基线 源码验证基线](#第-9-章-olk-66-源码验证基线)
- [第 10 章 开发就绪前置条件检查清单](#第-10-章-开发就绪前置条件检查清单)

---

## 第 1 章 工程标准体系总览

### 1.1 4 主题文档架构

agentrt-linux 工程标准由 4 主题文档构成，本手册为其总索引与执行依据：

| 主题 | 文档 | 职责 |
|------|------|------|
| 01 | [01-coding-standards.md](./01-coding-standards.md) | 代码规范合集：Part I 语义规范 + Part II 格式 + Part III 风格 + Part IV Checkpatch + Part V kernel-doc + Part VI clang-format |
| 04 | [04-engineering-philosophy.md](./04-engineering-philosophy.md) | 工程思想：双层稳定性 / 策略机制分离 / regression 不可接受 |
| 05 | [05-development-process.md](./05-development-process.md) | 开发流程合集：Part I 流程 + Part II 工具链 + Part III 合规检查 + Part IV SPDX 许可证 |
| 07 | [07-maintainers-and-governance.md](./07-maintainers-and-governance.md) | 治理：MAINTAINERS / DCO / 信任链 / §10 注册表镜像 |
| —  | [09-ssot-registry.md](./09-ssot-registry.md) | SSoT 注册表（唯一权威） |
| —  | [90-terminology.md](./90-terminology.md) | 术语表 SSoT |
| —  | [120-cross-project-code-sharing.md](./120-cross-project-code-sharing.md) | 跨项目代码共享（IRON-9 v3 [SC] 共享契约层） |

### 1.2 三大支柱

| 支柱 | 来源 | 在工程标准中的体现 |
|------|------|------------------|
| **Linux 6.6 内核基线工程思想** | Linux 6.6 内核基线 源码深度研究 | 双层稳定性、渐进式开发、regression 不可接受、不破坏用户空间 |
| **Airymax 五维正交 24 原则** | 00-architectural-principles.md（S/K/C/E/A 五维） | 每条工程标准映射到至少一条五维原则 |
| **seL4 微内核设计思想** | Liedtke minimality principle + capability-based security | 内核极简、服务用户态化、消息传递通信 |

### 1.3 工程标准适用范围

- **适用对象**：agentrt-linux 全部 8 子仓（kernel/services/security/memory/cognition/cloudnative/system/tests-linux）
- **适用语言**：内核态 C（主力）、内核模块 Rust（安全敏感子系统）、用户态服务 C/Rust/Python/TypeScript
- **适用阶段**：0.1.1→ 1.0.1→ 后续所有版本
- **不适用**：agentrt 用户态运行时（遵循 agentrt 自身的 IRON-1~10 规则，与本手册 IRON-9 v3 同源但独立）

---

## 第 2 章 IRON 工程铁律（不可妥协）

### 2.1 OS-IRON 编号汇总（15 条铁律）

> **SSoT 声明**：OS-IRON 铁律编号的唯一权威来源为 [09-ssot-registry.md §2](./09-ssot-registry.md)（15 条铁律，含 OS-IRON-015 编号管理元规则，2026-07-15 提升）。本表为该注册表的公开镜像，必须与之一一对应。原表中不属于 IRON 层级的规则已重分类：①"7 层自动化验证强制"→ OS-ACC-002（验收标准）；②"信任链分层不可越级提交"→ OS-STD-GOV-008；③"Reviewed-by 不可替代 Signed-off-by"→ 由 IRON-005（审查优先）+ IRON-007（DCO）共同涵盖；④"组织 6 级成熟度 Level 2"→ OS-STD-GOV-009。

| 编号 | 规则 | 五维映射 | 来源文档 |
|------|------|---------|---------|
| **OS-IRON-001** | 用户空间 ABI 永不破坏——一旦接口导出至用户空间，必须永久支持 | K-2 / E-7 | 04 工程思想 §2.1 |
| **OS-IRON-002** | 内核内部 API 不保证稳定；API 改动者必须同时修复所有受影响代码（"you broke it, you fix it"） | K-1 / C-2 | 04 工程思想 §2.2 |
| **OS-IRON-003** | 策略与机制分离——机制留在内核（提供能力），策略外移至用户空间或可插拔模块 | K-4 | 04 工程思想 §3 |
| **OS-IRON-004** | 渐进式开发，补丁自包含——每个补丁是逻辑上自包含的最小变更单元，中点可编译运行，支持 git bisect | S-4 / C-2 | 04 工程思想 §4 / 05 开发流程 |
| **OS-IRON-005** | 审查优先文化——所有代码进入主线前必须通过至少一名维护者审查并获得 Reviewed-by 标签 | S-3 / S-1 | 04 工程思想 §6 / 07 治理 §6 |
| **OS-IRON-006** | 不破坏用户空间原则——任何补丁不得导致用户空间程序行为改变，除非该行为被明确标记为 bug（regression 零容忍） | E-6 | 04 工程思想 §9 / 05 开发流程 |
| **OS-IRON-007** | DCO 强制——所有补丁必须包含 Signed-off-by 标签，CI 中 DCO bot 自动验证链条完整性 | E-6 | 07 治理 §5 |
| **OS-IRON-008** | 共享契约层 10 个 [SC] 核心头文件（包含 error.h，bpf_struct_ops.h 为补充共享文件非 [SC] 核心）双向 CI 验证——[SC] 头文件变更必须通过 agentrt + AirymaxOS CI 同时通过 | E-6 / E-8 | 120 跨项目代码共享 |
| **OS-IRON-009** | 代码共享边界：仅 agentrt ↔ AirymaxOS——与 主流 Linux 发行版 项目仅技术参考，不共享代码 | — | 120 跨项目代码共享 / 07 治理 §1.4 |
| **OS-IRON-010** | "Linux 6.6 为基、seL4 为鉴"工程取向——以 Linux 6.6 内核基线 工程思想为基线，以 seL4 微内核思想为架构层借鉴 | — | 04 工程思想 §12 |
| **OS-IRON-011** | 双源边界声明——Linux 6.6 内核基线 与 seL4 源码仅为本地参考路径研究对象，关系是"技术参考"非"代码共享" | — | 04 工程思想 §12 |
| **OS-IRON-012** | seL4 借鉴仅限架构层（ES-SEL4-1~5）——不包括编码风格层（4 空格 / camelCase / 重度 typedef） | — | 04 工程思想 §12 |
| **OS-IRON-013** | 8 子仓独立 git 仓库 + submodule 管理——拆分为 8 个独立 leaf 仓，由管理仓通过 submodule 统一管理 | S-2 | 04 工程思想 §13 |
| **OS-IRON-014** | [SC] 共享契约层 10 个 [SC] 核心头文件单一数据源（禁止物理副本）——10 个 [SC] 头文件物理宿主在 kernel/include/airymax/，其他子仓通过 -I 引用（bpf_struct_ops.h 为补充共享文件，非 [SC] 核心） | E-7 | 120 跨项目代码共享 |
| **OS-IRON-015** | 编号管理元规则——OS-KER / OS-STD / OS-OBS / OS-DRV 等所有规则编号一经分配不得复用；废弃规则标记 `DEPRECATED` 但保留编号（原 OS-STD-017，2026-07-15 提升为 OS-IRON-015） | S-1 | 90-observability/02-ebpf-probes.md §14.2 |

### 2.2 IRON 铁律执行机制

| 铁律 | 执行机制 | 违反后果 |
|------|---------|---------|
| OS-IRON-001 | ABI 审查组 + 6 个月宽限期 + 弃用声明 | PR 被拒绝合并 |
| OS-IRON-002 | 同补丁序列修复所有调用点 | NACK，补丁被打回 |
| OS-IRON-003 | 可插拔策略审查（sched_tac / LSM 钩子 / ops 注入） | 策略硬编码入内核被拒绝 |
| OS-IRON-004 | CI 每个补丁中点编译测试，git bisect 可追溯 | CI 失败，阻断合并 |
| OS-IRON-005 | CODEOWNERS bot 自动分配审查人 + Reviewed-by 标签校验 | 缺少 Reviewed-by 的 PR 无法合并 |
| OS-IRON-006 | regression 报告追踪 + RC 周期强制修复或 revert | revert 引入补丁 |
| OS-IRON-007 | DCO bot 自动验证 Signed-off-by 链条完整性 | PR 无法合并 |
| OS-IRON-008 | [SC] 头文件变更触发 agentrt + AirymaxOS 双向 CI | 任一端 CI 失败阻断合并 |
| OS-IRON-009 | 同源语义漂移检测 | 两端语义不一致告警 |
| OS-IRON-010 | 工程取向合规审查（编码风格 / 构建系统 / 内存模型基线检查） | 偏离基线的补丁被拒绝 |
| OS-IRON-011 | 双源边界审查（01Reference/ 仅本地参考路径） | 代码层面的直接引用被拒绝 |
| OS-IRON-012 | seL4 借鉴范围审查（仅 ES-SEL4-1~5 架构层） | 编码风格层借鉴被拒绝 |
| OS-IRON-013 | 8 子仓仓库结构校验 + submodule 一致性检查 | 跨仓直接引用（非 submodule）被拒绝 |
| OS-IRON-014 | `scripts/check-sc-single-source.sh` 检查 [SC] 头文件单一数据源 | 物理副本被检测并拒绝 |
| OS-IRON-015 | SSoT 注册表唯一性校验 + 废弃规则编号占用扫描 + CI 编号冲突检测 | 重复登记或复用废弃编号被 CI 阻断合并 |

---

## 第 3 章 SSoT 规则编号注册表

> **SSoT 声明（2026-07-12 重新设计）**：规则编号的**唯一权威来源**为 [09-ssot-registry.md](./09-ssot-registry.md)。该注册表统一登记 AirymaxOS 全部编号、规则与命名，任何文档不得私自定义规则编号。新规则须按 [09-ssot-registry.md §1.4](./09-ssot-registry.md) 流程申请注册。
>
> **模块前缀 SSoT 交叉引用**：函数与变量命名前缀（`are_`/`airy_`/`airy_core_`/`airy_` 及模块子前缀）的唯一权威来源（SSoT）为 [`10-coding-style/coding_conventions.md Part IV §4.4.3`](./10-coding-style/coding_conventions.md)。

本章不再镜像 09-ssot-registry.md 的完整内容（2026-07-12 SSoT 重新设计：取消三层镜像，消除 ~200 行重复）。请直接查阅 [09-ssot-registry.md](./09-ssot-registry.md) 获取：

- §2 OS-IRON 工程铁律（15 条）
- §3 OS-KER 内核工程规则（001-155 + 211/221）
- §4 OS-STD 标准规则（CODE/FMT/STY/GOV/DOC/CHK 子域）
- §5 OS-BAN 禁止规则（11 条）
- §6 OS-ACC 验收标准
- §7 OS-ABI 接口稳定性
- §8 命名规范
- §9 OS-SEC 安全工程规则
- §10 OS-ARCH 架构工程规则（2026-07-15 新增，登记 OS-ARCH-001~010）
- §11 OS-IFACE 接口工程规则（2026-07-15 新增，登记 OS-IFACE-001~010）
- §12 OS-TEST 测试工程规则（2026-07-15 新增，登记 OS-TEST-001~022）
- §13 agentrt 规则编号体系（17 类 751 条，2026-07-15 升级合并，原 §10 重编号）

### 3.1 编号格式约定（速查）

所有规则编号使用 3 位数字序号：`OS-<前缀>-<子域>-<NNN>`

- **前缀**：IRON（铁律）/ STD（标准）/ KER（内核）/ BAN（禁止）/ ABI（接口稳定性）/ ACC（验收）
- **子域**（可选，仅 STD 前缀使用）：CODE（编码）/ FMT（格式）/ STY（风格）/ GOV（治理）/ DOC（文档）/ TEST/OBS/SEC/DRV/TOOL（文档专属子域）
- **示例**：`OS-KER-001`、`OS-STD-FMT-001`、`OS-STD-CODE-001`、`OS-STD-GOV-005`

> 完整编号格式约定与文档专属子域说明见 [09-ssot-registry.md §1](./09-ssot-registry.md)。

### 3.2 编号申请流程

新规则编号须经过以下流程注册：

1. **RFC**：在 GitHub Discussions 发起 RFC，说明规则编号、规则文本、来源文档、适用范围。
2. **评审**：经至少一名顶级子系统维护者审查，公示 14 天接受异议。
3. **注册**：通过评审后，由总维护者将规则写入 [09-ssot-registry.md](./09-ssot-registry.md) 对应章节。

> 完整编号申请流程见 [09-ssot-registry.md §1.4](./09-ssot-registry.md)。

### 3.3 已废除的本地章节

> **说明**（2026-07-12 SSoT 重新设计）：以下本地章节已废除，统一由 [09-ssot-registry.md](./09-ssot-registry.md) 承载，消除三层镜像重复：
>
> - 原 §3.2 OS-STD 标准规则汇总（CODE/FMT/STY/GOV 子节完整表格）→ 见 09 §4
> - 原 §3.3 OS-KER 内核工程规则 → 见 09 §3
> - 原 §3.4 OS-BAN 禁止规则汇总（11 条）→ 见 09 §5
> - 原 §3.5 OS-FMT 格式规则汇总（已废除）→ 已并入 OS-STD-FMT 子域
> - 原 §3.6 OS-ABI 接口稳定性规则 → 见 09 §6
> - 原 §3.7 OS-ACC 验收标准（5 条）→ 见 09 §7
> - 原 §3.8 SSoT 交叉引用映射表（OS-KER/OS-BAN/OS-ACC 技术点映射）→ 2026-07-12 全量消解完成，映射表退役
> - 原 §3.9 编号申请流程 → 已合并至本节 §3.2

---

## 第 4 章 IRON-9 v3 四层共享模型与跨项目代码共享机制

### 4.1 三层共享模型

agentrt-linux 与 agentrt 之间的代码共享与语义同源遵循 IRON-9 v3 四层模型：

| 层次 | 标注 | 共享程度 | 共享内容 | 共享方式 |
|------|------|---------|---------|---------|
| **共享契约层** | [SC] | 完全共享头文件 | 10 个头文件（见 §4.2） | `include/airymax/` 目录，两端 `#include` 同一头文件 |
| **语义同源层** | [SS] | 语义一致，实现独立 | MicroCoreRT 调度语义 / AgentsIPC 128B 消息头 / Cupolas 安全模型 / CoreLoopThree 三层循环 | 两端独立实现，通过行为契约测试验证语义一致 |
| **完全独立层** | [IND] | 完全独立 | agentrt-linux 专属：内核驱动框架、Kbuild、内核内部 API；agentrt 专属：跨平台用户态运行时、SDK 四语言 | 各自独立仓库，无依赖关系 |

### 4.2 [SC] 共享契约层头文件清单

> 基础 4 个头文件先确立；`sched.h` 和 `ipc.h` 于后续调度子系统与 IPC 子系统修订时新增，现共 10 个 [SC] 头文件。

| 头文件 | 共享内容 | IRON-9 v3 层次 |
|--------|---------|---------------|
| `include/airymax/syscalls.h` | Syscall 编号体系（v1.1: 4 核心 + 20 预留 = 24 槽位） | [SC] |
| `include/airymax/memory_types.h` | MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 持久化接口 | [SC] |
| `include/airymax/security_types.h` | POSIX capability 41 ID 枚举 + LSM 钩子 252 ID 枚举 + Cupolas blob 布局 + capability 派生模型 + Vault backend 抽象 + 策略裁决结果 4 值枚举 | [SC] |
| `include/airymax/cognition_types.h` | CoreLoopThree 阶段枚举 + Thinkdual 模式枚举 + LLM 推理阶段枚举 + CoreLoopThree 上下文结构 + Token 能效指标结构 + GPU/NPU 能力描述符 | [SC] |
| `include/airymax/sched.h` | sched_tac 调度类约束（使用 SCHED_DEADLINE/SCHED_FIFO/EEVDF 原生调度类，禁止定义 SCHED_AGENT 宏）+ 任务描述符（magic 0x41475453 'AGTS'）+ vtime 类型与衰减公式 + 优先级范围 0-139 + AIRY_SLICE_DFL（20ms） | [SC] |
| `include/airymax/ipc.h` | IPC magic（0x41524531 'ARE1'）+ 128B 消息头结构（`struct airy_ipc_msg_hdr`）+ SQE/CQE 操作码与标志位 | [SC] |

### 4.3 代码共享边界

| 边界 | 共享策略 | 说明 |
|------|---------|------|
| agentrt ↔ agentrt-linux | [SC] 共享契约层 10 个头文件 | 唯一的代码共享通道 |
| agentrt-linux ↔ Linux 6.6 基线 | 仅技术参考，不共享代码 | 参考主流 Linux 工程思想与实现方式，不共享代码 |
| agentrt ↔ 第三方项目 | 按许可证条款使用 | agentrt 采用 AGPL-3.0 + Apache-2.0 双开源许可证，第三方项目按相应许可证条款使用 |

### 4.4 IPC magic 值与任务描述符 magic

| 标识 | magic 值 | ASCII | 用途 | 共享层次 |
|------|---------|-------|------|---------|
| IPC 消息头 magic | `0x41524531` | 'ARE1' | agentrt 与 AirymaxOS 共享 IPC 消息头布局 | [SC] |
| 任务描述符 magic | `0x41475453` | 'AGTS' | 任务描述符独立 magic，避免与 IPC 消息混淆 | [IND] |

> **设计原则**：IPC magic 同源但独立——agentrt 和 AirymaxOS 共享 IPC 消息头布局，magic 值同源，无需协议转换层。任务描述符不是 IPC 消息，使用独立 magic 避免混淆。

### 4.5 安全作为横切关注点

安全不是独立数据流，而是横切关注点（cross-cutting concern），贯穿所有 4 大数据流：
- 认知循环数据流（CoreLoopThree）
- 记忆卷载数据流（MemoryRovol）
- IPC 消息数据流（AgentsIPC）
- 调度数据流（sched_tac）

---

## 第 5 章 7 层自动化验证体系

### 5.1 7 层验证流水线

| 层级 | 验证内容 | 工具 | 失败后果 |
|------|---------|------|---------|
| **第 1 层** | 编译期验证 | gcc -W / clang / -Werror | 阻断提交 |
| **第 2 层** | 静态分析 | sparse / Smatch / Coccinelle / clang-analyzer / rust-clippy | 阻断提交 |
| **第 3 层** | 预提交验证 | pre-commit hooks / DCO bot / format-check | 阻断推送 |
| **第 4 层** | CI 门禁 | GitHub Actions / 9 矩阵（3 架构 × 3 配置） | 阻断合并 |
| **第 5 层** | 连接验证 | 符号检查 / ABI 漂移检测 | 阻断合并 |
| **第 6 层** | 协议验证 | AgentsIPC 128B 校验器 / Agent 行为契约测试 | 阻断合并 |
| **第 7 层** | 发布验证 | GPG 签名 / 模块签名 / SHA-256 校验 | 阻断发布 |

### 5.2 CI 矩阵覆盖要求

| 维度 | 覆盖范围 |
|------|---------|
| 架构 | x86_64 / aarch64 / riscv64（+ ppc64 推荐用于交叉编译检查） |
| 配置 | allnoconfig / allmodconfig / defconfig |
| 编译器 | gcc + clang（双编译器交叉验证） |
| 总时长 | ≤ 60 分钟 |

### 5.3 动态分析工具矩阵

| 工具 | 检测内容 | 配置 | 适用场景 |
|------|---------|------|---------|
| KASAN | 地址越界 / use-after-free | `CONFIG_KASAN=y` | 测试环境 |
| KFENCE | 轻量级内存错误 | `CONFIG_KFENCE=y` | 生产环境（采样模式） |
| UBSAN | 未定义行为 | `CONFIG_UBSAN=y` | 测试环境 |
| KCSAN | 数据竞争 | `CONFIG_KCSAN=y` | 并发测试 |
| lockdep | 死锁 / 锁顺序违规 | `CONFIG_LOCKDEP=y` | 全功能启用 |
| kmemleak | 内存泄漏 | `CONFIG_DEBUG_KMEMLEAK=y` | kselftest 后检查 |

### 5.4 覆盖率门槛

| 门槛 | 要求 |
|------|------|
| KCOV | 模糊测试必须启用 KCOV 收集覆盖率 |
| gcov | GCC 覆盖率工具，全局或按模块覆盖率 |
| KUnit | 白盒单元测试，毫秒级运行 |
| kselftest | 用户态系统级测试 |

---

## 第 6 章 合规性禁词清单与命名体系规范

### 6.1 BAN-361 禁止引用的 Linux 7.0 专属特性

以下特性属于 Linux 7.0 专属，禁止在 agentrt-linux 文档、代码、注释中引用为 Linux 6.6 原生能力：

| 禁止引用 | 正确表述 | 说明 |
|---------|---------|------|
| `PREEMPT_LAZY` | `PREEMPT_DYNAMIC` | Linux 6.6 原生抢占模型 |
| `Rust 正式转正` | `Rust 实验性支持` | Linux 6.6 Rust 仍为实验性 |
| `MGLRU 2.0` | `MGLRU 多代 LRU` | Linux 6.6 原生 MGLRU |
| `XFS 自修复` | `XFS 在线修复` | Linux 6.6 原生 XFS 能力 |
| `eBPF 签名验证` | `eBPF kfunc + dynamic pointer` | Linux 6.6 原生 eBPF 能力 |
| `sched_ext（误标为 Linux 6.6 原生能力）` | `sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射）` | 标准 Linux 6.6 主线不包含 sched_ext（6.12 才合入主线）；agentrt-linux 采用sched_tac 替代 sched_ext |

### 6.2 命名体系规范

#### 6.2.1 产品标准名称

| 产品 | 标准全称 | 简称 | 定位 |
|------|---------|------|------|
| agentrt | AirymaxAgentRT 极境智能体运行底座平台工程 | agentrt / AirymaxRT | 用户态组件、AI Agent 运行时平台工程 |
| agentrt-linux | AirymaxOS 极境智能体操作系统 | agentrt-linux / AirymaxOS | 操作系统发行版、智能体操作系统 |

#### 6.2.2 命名规则

- **正式英文名**：全称使用 "AirymaxOS"（无空格），简称使用 "AirymaxOS"
- **正式中文名**：全称使用 "极境智能体操作系统"
- **代码仓库**：仓库名为 "agentrt-linux"（注意：目录名与文档中使用 "agentrt-linux" 作为产品代号）
- **文档表述**：首次出现使用全称 "agentrt-linux（AirymaxOS，极境智能体操作系统）"，后续使用简称 "agentrt-linux" 或 "AirymaxOS"

### 6.3 开源品牌表述清零规范

| 范围 | 规范 | 说明 |
|------|------|------|
| 开源文档（docs/AirymaxOS/） | 禁止出现 开源品牌表述 | 100% 清零，防止抄袭争议 |
| 内部参考文档 | 允许作为技术参考引用 | 内部参考，不公开 |
| 代码注释 | 禁止出现 开源品牌表述 | 防止开源后引发争议 |
| 与 上游 Linux 项目的关系 | 仅技术参考，不共享代码 | 符合 2026-07-07 开源极境工程与规范委员会决策 |

---

## 第 7 章 微核心与微内核术语规范

### 7.1 术语区分原则

| 术语 | 适用对象 | 英文 | 含义 |
|------|---------|------|------|
| **微核心**（micro-core） | agentrt（用户态） | MicroCore | agentrt 的 atoms/corekern（MicroCoreRT）是用户态微核心运行时，不是真正的内核 |
| **微内核**（micro-kernel） | agentrt-linux（OS 发行版） | Microkernel | agentrt-linux 基于 Linux 6.6 进行微内核化改造，将 VFS/网络栈/部分驱动移到用户态 |

### 7.2 MicroCoreRT 术语使用规范

MicroCoreRT 是 IRON-9 v3 [SS] 语义同源层组件，在两端有不同实现：

| 上下文 | 正确术语 | 错误术语 | 说明 |
|--------|---------|---------|------|
| 描述 agentrt 中的 MicroCoreRT | 微核心运行时 | ~~微内核运行时~~ | agentrt 是用户态，不是内核 |
| 描述 agentrt-linux 中的 MicroCoreRT | 微核心运行时 | ~~微内核运行时~~ | agentrt-linux 虽是 OS，但 MicroCoreRT 源自 agentrt，跨两端统一称"微核心运行时" |
| MicroCoreRT 标准名称（跨两端） | 微核心运行时（MicroCoreRT） | ~~微内核运行时~~ | 以 agentrt 上下文为准，因为 MicroCoreRT 源自 agentrt |

> **注**：[90-terminology.md](./90-terminology.md) 已将 MicroCoreRT 标准名称修正为"微核心运行时"。

### 7.3 微内核化改造策略表述规范

| 正确表述 | 错误表述 | 说明 |
|---------|---------|------|
| agentrt-linux 基于 Linux 6.6 进行微内核化改造 | ~~agentrt-linux 是从零开发的微内核~~ | 不是从零开发，是基于 Linux 改造 |
| 微内核化改造策略（sched_tac + seL4 MCS 映射 + eBPF + io_uring） | ~~真正的微内核~~ | 是改造，不是原生微内核 |
| agentrt 的微核心设计思想 | ~~agentrt 的微内核化哲学~~ | agentrt 是用户态，没有"微内核化" |

---

## 第 8 章 乔布斯美学"无隐藏瑕疵"审查标准

### 8.1 审查哲学

> "For you to sleep well at night, the aesthetic, the quality, has to be carried all the way through." — Steve Jobs

agentrt-linux 工程标准贯彻乔布斯美学——不仅用户可见的表面要完美，用户不可见的内部构造（代码结构、编码风格、实现模式、设计思想）也必须达到最高质量标准。**没有"用户看不到所以可以妥协"的隐藏瑕疵。**

### 8.2 6 维度审查清单

| 维度 | 审查项 | 通过标准 | 验证方法 |
|------|--------|---------|---------|
| **代码结构** | 8 子仓边界清晰 | 模块划分遵循微内核设计思想，无跨子仓循环依赖 | 依赖图分析 |
| **编码风格** | 7 主题文档覆盖全部场景 | C 内核态 + Rust 内核模块 + 用户态服务全覆盖 | 文档审查 |
| **实现模式** | 策略与机制分离 | 内核只提供机制，策略外移至用户态/BPF | 架构审查 |
| **设计思想** | 五维正交 24 原则映射 | 每条工程标准映射到至少一条五维原则 | 映射表检查 |
| **规则编号** | SSoT 注册表无冲突 | 所有编号唯一，无多重定义 | SSoT 一致性检查 |
| **文档完整性** | Copyright + 基线声明 | 所有文档有 Copyright + Linux 6.6 基线声明 | Grep 全文档扫描 |

### 8.3 "无隐藏瑕疵"检查清单

| 检查项 | 检查方法 | 通过标准 |
|--------|---------|---------|
| 禁词清零 | `grep -rE "MGLRU 2.0\|PREEMPT_LAZY\|Rust 正式转正\|XFS 自修复\|eBPF 签名验证" docs/` | 0 结果（排除禁词清单本身） |
| 开源品牌清零 | `grep -riE "主流 Linux 发行版" docs/AirymaxOS/` | 0 结果（排除审查报告） |
| 微核心概念一致 | `grep -r "微内核" docs/AirymaxOS/ \| grep -i agentrt` | 0 结果（agentrt 不应使用"微内核"） |
| SSoT 编号一致 | 对照 [09-ssot-registry.md](./09-ssot-registry.md) | 所有编号引用与注册表一致 |
| Copyright 完整 | `grep -rL "Copyright (c) 2025-2026 SPHARX" --include="*.md" docs/AirymaxOS/` | 0 结果 |
| 基线声明完整 | `grep -rL "Linux 6.6" --include="*.md" docs/AirymaxOS/` | 仅顶层 README 可缺失 |

---

## 第 9 章 Linux 6.6 内核基线 源码验证基线

### 9.1 关键源码验证路径

所有技术方案修正均对照 Linux 6.6 内核源码 验证：

| 验证项 | 源码路径 | 验证内容 | 验证结果 |
|--------|---------|---------|---------|
| sched_tac 调度类 | `include/uapi/linux/sched.h:114-123` | SCHED_NORMAL=0 / SCHED_FIFO=1 / SCHED_DEADLINE=6 等原生调度类（SCHED_EXT=7 仅宏定义，sched_ext 框架未实现） | Linux 6.6 内核基线 原生支持（sched_tac 复用） |
| FPU 禁用 | `arch/x86/Makefile:137` | `-mno-80387` | 内核态禁用 float |
| io_uring 仅用户态 | `io_uring/io_uring.c` | 仅 3 个 SYSCALL_DEFINE 入口 | 无内核内部 kthread API |
| kfifo SPSC | `include/linux/kfifo.h` | 无锁环形队列 | kthread 间通信适用 |
| wait_event | `include/linux/wait.h:478` | `wait_event_interruptible` | kthread 等待机制 |

### 9.2 不移植的 上游 Linux 6.6 不含的特性

以下 上游 Linux 6.6 不含的特性不移植到 agentrt-linux：

| 特性 | 不移植原因 | 替代方案 |
|------|---------|---------|
| KABI_RESERVE | 与 IRON-1（禁止新特性）冲突 | — |
| etmem | 用 memory.reclaim + tiering 替代 | `memory.reclaim` + tiering |
| dynamic_pool | 用 memcg 替代 | memcg |
| numa_remote | 用 CXL 替代 | CXL |
| KMSAN | 开销过大 | KASAN + KCSAN 组合 |

---

## 第 10 章 开发就绪前置条件检查清单

### 10.1 M1 开发前置条件

进入开发前，须满足以下全部条件：

| # | 前置条件 | 当前状态 | 说明 |
|---|---------|---------|------|
| 1 | P0 阻断性问题全部修复 | 12/13 已修复 | 工时口径统一延期至 M0 第 1 周，不阻断 |
| 2 | 开源文档 开源品牌清零 | 验证通过 | Grep 验证 0 结果 |
| 3 | MGLRU 2.0 清零 | 验证通过 | 10/10 实际违规已修复 |
| 4 | SSoT 规则编号无冲突 | 验证通过 | OS-KER-001 / OS-STD-001 冲突已消解 |
| 5 | IRON-9 v3 四层模型落地 | 验证通过 | 10 个 [SC] 共享头文件已定义 |
| 6 | 工程标准规范手册产出 | 交付 | 本手册（交付成果 #3） |
| 7 | 高优先级修复（5 项） | M0 阶段 | — |
| 8 | Linux 6.6 内核基线 源码验证 | 验证通过 | 7 项关键路径验证通过 |

### 10.2 M0 阶段任务

M0 阶段（进入 M1 开发前的准备阶段）须完成以下任务：

| 周次 | 任务 | 优先级 |
|------|------|--------|
| 第 1 周 | Copyright 补齐（17 文件） | 高 |
| 第 1 周 | MicroCoreRT 微内核→微核心（90-terminology.md 9 行） | 高 |
| 第 1 周 | agentrt 微内核化→微核心（01-kernel.md 3 行） | 高 |
| 第 1 周 | 核心设计文档扩至 400+ 行 | 高 |
| 第 1 周 | 模块文档补充 Mermaid 图 | 高 |
| 第 1 周 | 工时口径统一 | 高 |
| 第 2 周 | 命名体系修正（AirymaxOS 全称） | 中 |
| 第 2 周 | agentrt 微核心原语命名统一 | 中 |
| 第 2 周 | "不是微内核"表述张力消解 | 中 |
| 第 2 周 | 文档版本控制机制建立 | 中 |
| 第 2 周 | Linux 6.6 基线声明补齐（39 文件） | 中 |
| 第 2 周 | CoreLoopThree kthread 替代方案论述 | 中 |

### 10.3 就绪状态结论

**当前状态**：0.1.1 文档体系修订后综合成熟度 93%，达到"有条件 Go"状态。

**进入开发的前置条件**：M0 第 1 周完成 5 项高优先级修复 + 工时口径统一后，即可正式进入开发。

**质量承诺**：本手册作为后续开发的强制执行依据，所有开发的代码提交、文档变更、工程决策均须符合本手册规定。任何偏离须经 RFC 评审 + 总维护者批准。

---

> **手册维护**： 本手册随 0.1.1 文档体系迭代而更新。每次修订须更新第 3 章 SSoT 镜像、第 6 章禁词清单、第 10 章就绪状态。手册版本号与 0.1.1 文档体系版本号保持一致。
> **权威性声明**： 当各主题文档（01-07）与本手册冲突时，以本手册为准；当本手册与 SSoT 注册表（[09-ssot-registry.md](./09-ssot-registry.md)）冲突时，以 SSoT 注册表为准。新规则须按 [09-ssot-registry.md §1.4](./09-ssot-registry.md) 流程申请注册。
