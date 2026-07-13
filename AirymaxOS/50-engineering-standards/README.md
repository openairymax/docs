Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）工程标准规范

> **文档定位**： agentrt-linux（AirymaxOS，极境智能体操作系统）工程标准规范的主索引与总纲\
> **版本**： 0.1.1\
> **最后更新**： 2026-07-13\
> **同源映射**： agentrt `docs/AirymaxRT/00-architectural-principles.md`（五维正交 24 原则）+ IRON-9 v2 工程铁律（工程标准规范，17 类规则编号体系）\
> **理论根基**： Linux 6.6 内核工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）

> **审查状态**：Wave 2 v2 源码级深读审查完成（Phase A/B/C/D）。本模块（工程标准层 23 文档）已通过：[SC] 6 头文件 Tab 8 缩进验证（对齐 OLK-6.6 §1）、SSoT 物理宿主 `120-cross-project-code-sharing.md` [SC] 代码段 69 行 Tab 缩进 / 0 行 4 空格、09-ssot-registry.md 唯一 SSoT 注册表确认、ES-OLK-1~13 工程思想 + seL4 设计模式对齐（B1/B2 深读）、当前方案 v0.6.0 P0 修复率 92% / 三层评分综合 80/100（B）。Wave 2 v2 Phase C D 类 OLK-6.6 工程标准 8 项差距（C-D01~C-D08，~18h）已识别，移交 1.0.1 M1+。

---

## 1. 工程标准定位

### 1.1 为什么需要 agentrt-linux 独立的工程标准

agentrt-linux 作为基于 Linux 内核的智能体操作系统发行版，其工程标准必须同时满足三重诉求：

1. **内核工程严肃性**——继承 Linux 内核 30+ 年沉淀的工程思想（不破坏用户空间 ABI、不维护稳定内部 API、goto 集中错误处理、强制工具链、信任链分层维护者制度），任何脱离此基础的"自创"规范都会引入系统性风险。
2. **智能体场景适配**——传统内核工程规范面向"硬件资源抽象与调度"场景，而 agentrt-linux 面向"智能体工作负载"场景，需要在内核严肃性之上扩展：多语言栈（C/Rust/Python/TS）、多层接口稳定性分级、Agent 行为契约测试、Token 能效可观测性等专属规范。
3. **Airymax 同源约束**——与 agentrt（AirymaxAgentRT）共享 Airymax 设计理念（MicroCoreRT / AgentsIPC / Cupolas / MemoryRovol / CoreLoopThree），工程标准必须保证同源语义在两端（用户态运行时 + OS 发行版）表述一致、可互操作。

### 1.2 工程标准三大支柱

| 支柱                     | 思想来源                                        | 落地章节           |
| ---------------------- | ------------------------------------------- | -------------- |
| **Linux 内核工程思想**       | Linux 6.6 内核源代码（Linux 6.6 内核基线）深度研究提炼              | §3-§7 章节（01/04/05 三文档）   |
| **Airymax 五维正交 24 原则** | `00-architectural-principles.md`（S/K/C/E/A 五维） | §4 工程思想章节      |
| **agentrt 17 类规则编号体系** | `0.1.1工程标准规范手册.md`（IRON/BAN/FOUND 等）        | §8 OS 工程规则编号体系 |

### 1.3 与 agentrt 工程规范的关系（同源且部分代码共享，IRON-9 v2）

agentrt-linux 工程标准与 agentrt 工程标准是**同源且部分代码共享**的关系（IRON-9 v2，2026-07-07 开源极境工程与规范委员会决策变更）：

#### 三层共享模型

| 层次                          | 共享程度          | 内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | 组织方式                             |
| --------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| **共享契约层**（Shared-Contract）  | 完全共享代码        | **6 个 \[SC] 头文件清单**（`include/airymax/`）：(1) `syscalls.h`——Syscall 编号体系（12 核心 + 12 预留 = 24 槽位）；(2) `memory_types.h`——MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 持久化接口；(3) `security_types.h`——POSIX capability 41 ID + LSM 钩子 252 ID + Cupolas blob 布局 + capability 派生模型 + Vault backend + 策略裁决 4 值枚举；(4) `cognition_types.h`——CoreLoopThree 阶段枚举 + Thinkdual 模式枚举 + LLM 推理阶段枚举 + 上下文结构 + Token 能效指标 + GPU/NPU 描述符；(5) `sched.h`——SCHED\_EXT 调度类编号约束（复用内核 SCHED\_EXT=7，禁止 SCHED\_AGENT 宏）+ 任务描述符（magic 0x41475453 'AGTS'）+ vtime 类型与衰减公式 + 优先级范围 + AIRYMAX\_SLICE\_DFL；(6) `ipc.h`——IPC magic（0x41524531 'ARE1'）+ 128B 消息头结构（`struct airy_ipc_msg_hdr`）+ SQE/CQE 操作码与标志位。另含 syscall 编号（`AIRY_SYS_*`）、capability 令牌格式、错误码（`airy_err_t`）、规则编号体系（IRON/BAN/STD/ACC）、五维正交 24 原则 | `include/airymax/` 独立头文件库，两端共同依赖 |
| **语义同源层**（Shared-Semantics） | API 签名相同，实现独立 | 调度语义（MicroCoreRT：agentrt 用户态 vs agentrt-linux sched\_ext）、安全模型（Cupolas：agentrt 用户态 vs agentrt-linux LSM）、IPC 传输（AgentsIPC：agentrt 消息队列 vs agentrt-linux io\_uring）、记忆模型（MemoryRovol：agentrt heapstore vs agentrt-linux 内核态）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | 各自独立实现，通过契约层保证互操作                |
| **完全独立层**（Independent）      | 完全独立          | agentrt-linux 专属：内核驱动框架、Kbuild、systemd 集成、内核内部 API；agentrt 专属：跨平台用户态运行时、SDK 四语言、CLI/TUI                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | 各自独立仓库，无依赖关系                     |

#### 与旧版 IRON-9 的差异

| 维度    | 旧版 IRON-9（仅语义同源、实现独立） | 新版 IRON-9 v2（同源且部分代码共享）                    |
| ----- | --------------------- | ------------------------------------------ |
| 设计哲学  | 共享                    | 共享（不变）                                     |
| 规则编号  | 共享骨架                  | 共享骨架（不变）                                   |
| 契约层代码 | **不共享**，仅语义同源         | **完全共享**（`include/airymax/` 头文件库）          |
| 实现层代码 | 独立                    | 独立（不变）                                     |
| 互操作方式 | 通过同源语义无适配层            | **通过共享代码无适配层**（增强）                         |
| CI 校验 | 无                     | 契约层变更双向校验（agentrt + agentrt-linux CI 同时验证） |

- **同源**：共享 Airymax 五维正交 24 原则作为顶层设计哲学；共享 17 类规则编号体系骨架；共享 E-7 文档即代码、E-6 错误可追溯、E-1 安全内生等核心工程观原则。**共享契约层代码**（`include/airymax/` 头文件库）。
- **独立**：agentrt-linux 是 OS 发行版，承担内核态严肃性责任，其工程标准必须独立处理内核 ABI 稳定性、内核内部 API 不稳定性、补丁生命周期、维护者层级制度等 agentrt 不涉及的领域；同时 agentrt-linux 需要为内核态特有的场景（驱动模型、构建系统、可观测性、安全 LSM 等）制定专门规范。
- **互操作**：agentrt 在 agentrt-linux 上运行时，两端工程标准形成天然互补——agentrt 遵循其用户态运行时规范（如 `airy_log_write` 日志），agentrt-linux 遵循其内核发行版规范（如 `printk` 等级），两端通过**共享契约层代码**（如 `include/airymax/ipc.h` 的 IPC 消息头定义、`include/airymax/sched.h` 的任务描述符定义）实现无适配层互操作。

### 1.4 主流 Linux 发行版标准兼容性声明

agentrt-linux（AirymaxOS）的工程思想与实现方式与上游 Linux 保持一致性，以便未来兼容 主流 Linux 发行版标准的发行版工具链。

#### 兼容性层次

| 层次        | 兼容内容                                               | 兼容方式               | 代码共享        |
| --------- | -------------------------------------------------- | ------------------ | ----------- |
| **协议层兼容** | kthread 创建/停止/驻留协议、IPC 消息传递协议、capability 令牌协议      | 同源协议设计，\[IND] 独立实现 | **不共享代码**   |
| **接入层兼容** | 加速器接入模式（accel 框架）、GPU 调度器模式（drm\_sched）、eBPF 可编程内核 | 参考设计模式，\[IND] 独立实现 | **不共享代码**   |
| **模块化兼容** | Kconfig 风格、模块组织、Makefile 结构、Kbuild 规范              | 同源工程规范             | **不共享代码**   |
| **工具链兼容** | GCC/Clang/Rust 工具链、checkpatch/spatch、RPM/dnf 包管理   | 兼容 主流 Linux 发行版工具链 | 工具链共享（开源生态） |

#### 代码共享边界澄清

- **agentrt ↔ agentrt-linux**：\[SC] 共享契约层完全共享代码（`include/airymax/` 6 个头文件）
- **agentrt-linux ↔ Linux 6.6 基线**：**仅技术参考，不共享代码**——Linux 6.6 内核基线的 kthread.c/accel.c/sched\_main.c 等是 agentrt-linux \[IND] 独立层的实现参考
- agentrt-linux 工程思想与上游 Linux 一致性体现在协议层（创建/停止/驻留协议）和接入层（accel/drm\_sched 模式），不体现在代码层

#### 兼容性验证检查项

| 检查项           | 验证方法                                | 合格标准                   |
| ------------- | ----------------------------------- | ---------------------- |
| kthread 协议一致性 | 核对 kthread\_create/stop/park API 签名 | 与 主流 Linux 发行版标准签名一致   |
| 加速器接入一致性      | 核对 drivers/accel/ 模式                | 接入模式一致（\[IND] 独立实现）    |
| Kconfig 风格一致性 | 核对 CONFIG\_ 前缀命名                    | 与上游 Linux Kconfig 风格一致 |
| 模块化一致性        | 核对 Makefile/Kbuild 结构               | 与上游 Linux 构建结构一致       |
| 调度类编号不冲突      | 核对 SCHED\_AGENT 编号                  | 不与上游 Linux 已有调度类冲突     |

> **详见**：工程规范委员会发布的各子系统技术规范

***

## 2. 工程标准框架（4 主题 + SSoT + 术语 + 跨项目共享）

agentrt-linux 工程标准由 4 个主题文档（01/04/05/07）+ 1 个 SSoT 注册表（09）+ 1 个术语表（90）+ 1 个跨项目代码共享文档（120）+ 1 个工程标准手册（00）+ 5 个子目录（10-coding-style/20-contracts/30-runtime-interfaces/40-integration/50-project-erp）构成，共 23 个文档（含本 README）：

```
50-engineering-standards/
├── README.md                            # 本文件 — 主索引与总纲
├── 00-engineering-standards-handbook.md # 工程标准规范手册（SSoT 索引 + IRON 铁律）
├── 01-coding-standards.md               # 代码规范合集（6 Parts：语义+格式+风格+Checkpatch+kernel-doc+clang-format）
├── 04-engineering-philosophy.md         # 工程思想（双层稳定性/策略机制分离/渐进式/审查优先）
├── 05-development-process.md            # 开发流程合集（4 Parts：流程+工具链+合规检查+SPDX 许可证）
├── 07-maintainers-and-governance.md     # 维护者制度与治理（MAINTAINERS/层级/成熟度模型/DCO/CODEOWNERS）
├── 09-ssot-registry.md                  # SSoT 全局规则编号注册表（唯一权威，所有编号登记于此）
├── 90-terminology.md                     # 术语表 SSoT
├── 120-cross-project-code-sharing.md    # 跨项目代码共享（IRON-9 v2 [SC] 共享契约层）
│
├── 10-coding-style/                      # 编码规范子目录
│   ├── README.md                         # 编码规范总览与导航
│   ├── C_Cpp_coding_style.md             # C/C++ 编码风格规范（Part I-IV：内核C风格+补充+C++风格+安全编码）
│   ├── Rust_coding_style.md              # Rust 编码风格规范（Part I-II：风格+安全编码）
│   ├── Go_coding_style.md                # Go 编码风格规范（Part I-II：风格+安全编码）
│   ├── scripting_coding_style.md         # 脚本语言规范（Part I-III：Python+JavaScript/TS+Java）
│   └── coding_conventions.md             # 通用编码约定（Part I-V：注释+命名+日志+前缀+安全设计）
│
├── 20-contracts/                         # 契约规范子目录
│   ├── README.md                         # 契约规范总览与导航
│   └── contracts.md                      # 契约规范合集（IPC 协议+日志+系统调用 API 契约）
│
├── 30-runtime-interfaces/                 # 运行时接口子目录
│   ├── README.md                         # 运行时接口总览与导航
│   └── runtime_interfaces.md             # 运行时接口合集（L1 运行时接口+L2 服务协议+L3 安全治理）
│
├── 40-integration/                        # 集成规范子目录
│   ├── README.md                         # 集成规范总览与导航
│   └── integration.md                    # 集成标准合集（agentrt 集成+生态伙伴+配置集成+标准贡献）
│
└── 50-project-erp/                        # 项目工程管理子目录
    ├── README.md                         # 项目工程管理总览与导航
    └── project_erp.md                    # 项目管理规范合集（SBOM+错误码+模块需求+资源管理）
```

> **文件分层说明**（2026-07-13 精简后）：
> - **顶层核心文档（9 文件）**：README + 00 手册 + 01 代码规范 + 04 工程思想 + 05 开发流程 + 07 治理 + 09 SSoT 注册表 + 90 术语表 + 120 跨项目共享
> - **SSoT 注册表**：唯一权威文件为 `09-ssot-registry.md`，登记全部规则编号、废弃文档、术语。2026-07-12 SSoT 重新设计：取消三层镜像，只保留 09-ssot-registry.md 一个 SSoT 文档
> - **子目录（5 目录 = 14 文档）**：10-coding-style（6 文档：README + 5 源文件）/ 20-contracts（2 文档）/ 30-runtime-interfaces（2 文档）/ 40-integration（2 文档）/ 50-project-erp（2 文档）
> - **已合并删除（9 文件，2026-07-13）**：02-code-format→01 Part II / 03-code-style→01 Part III / 60-checkpatch→01 Part IV / 70-kernel-doc→01 Part V / 80-clang-format→01 Part VI / 06-toolchain→05 Part II / 08-compliance→05 Part III / 110-spdx→05 Part IV / 100-deprecated→09 §8.2

> **迁移说明**：`10-coding-style/` 至 `50-project-erp/` 五个子目录由原 `50-specifications/` 目录迁移而来（原 `50-specifications/` 目录已删除）。`TERMINOLOGY.md` 已移入本模块并重命名为 `90-terminology.md`（2026-07-12 SSoT 整合）。

### 2.1 章节定位

| 章节 | 核心问题 | 来源 |
|------|---------|------|
| 00 工程标准手册 | "标准体系怎么索引"——SSoT 索引 + IRON 铁律 + 合规禁词 + 美学审查 | Linux `coding-style.rst` + Airymax IRON 体系 |
| 01 代码规范合集 | "代码该怎么写 + 长什么样 + 怎么思考"——6 Parts 合并：语义规范 + 格式 + 风格 + Checkpatch + kernel-doc + clang-format | Linux `coding-style.rst`（1271 行）+ `.clang-format`（689 行）+ `checkpatch.pl`（7800 行）+ Airymax 15 文件 |
| 04 工程思想 | "工程体系怎么建立"——稳定性哲学、策略机制分离、渐进式、审查优先 | Linux `stable-api-nonsense.rst` + Airymax K-1\~K-4 内核观 + E-1\~E-8 工程观 |
| 05 开发流程合集 | "代码怎么进入主线 + 规范怎么执行 + 怎么验证 + 许可证怎么标注"——4 Parts 合并：流程 + 工具链 + 合规检查 + SPDX 许可证 | Linux `development-process.rst`（8 章）+ `submit-checklist.rst`（24 项）+ `license-rules.rst` + Airymax 17 类规则 |
| 07 维护者制度与治理 | "项目怎么被治理"——MAINTAINERS、层级、成熟度模型、DCO | Linux `MAINTAINERS`（734KB）+ Airymax IRON-9 v2 同源且部分代码共享 |
| 09 SSoT 注册表 | "编号归谁管"——全局唯一规则编号注册表（唯一权威） | Airymax SSoT 体系 |
| 90 术语表 | "术语怎么用"——微核心/微内核/MicroCoreRT 等术语规范 | Airymax 术语体系 |
| 120 跨项目代码共享 | "agentrt ↔ agentrt-linux 怎么共享代码"——IRON-9 v2 [SC] 6 头文件 + 三层共享模型 | Airymax IRON-9 v2 同源且部分代码共享原则 |

### 2.2 与 Airymax 五维正交 24 原则的映射

agentrt-linux 工程标准是 Airymax 五维正交 24 原则在 OS 发行版场景下的具体落地：

| 五维原则           | 在 agentrt-linux 工程标准中的体现                                    | 落地章节                   |
| -------------- | ----------------------------------------------------------- | ---------------------- |
| **S-1 反馈闭环**   | 7 层自动化验证的每一层都是反馈闭环；CI 失败即反馈                                 | 05 开发流程 Part II             |
| **S-2 层次分解**   | 4 层接口稳定性分级（Agent API / Runtime API / 内核子系统 API / 内部实现）      | 04 工程思想                |
| **S-3 总体设计部**  | 维护者层级制度（Lieutenant System）+ 总维护者角色                          | 07 维护者制度与治理            |
| **S-4 涌现性管理**  | 渐进式开发模型 + Merge Window + RC 周期                              | 05 开发流程                |
| **K-1 内核极简**   | 微内核化改造策略（用户态服务化）+ 内核职责最小化                                   | 04 工程思想                |
| **K-2 接口契约化**  | 双层稳定性哲学（用户 ABI 稳定 / 内部 API 可重构）                             | 04 工程思想                |
| **K-3 服务隔离**   | 驱动用户态化 + 服务进程隔离 + capability 安全                             | 01 代码规范 Part III + 110-security |
| **K-4 可插拔策略**  | sched\_ext BPF 调度器 + LSM 钩子 + 模块签名                          | 01 代码规范 Part III                |
| **C-1 双系统协同**  | 快慢路径分层（热路径 C / 慢路径 Rust）+ 调度类可插拔                            | 01 代码规范 Part III                |
| **C-2 增量演化**   | 补丁序列中点可编译运行 + git bisect 友好                                 | 05 开发流程                |
| **C-3 记忆卷载**   | 多代 LRU（MGLRU）+ 分级内存（CXL/PMEM）                               | 170-performance        |
| **C-4 遗忘机制**   | 弃用接口清单（Deprecated Interfaces Registry）+ ABI 弃用流程            | 09-ssot-registry.md §8.2              |
| **E-1 安全内生**   | 强制安全编码 + fault injection + LSM 集成                           | 01 代码规范 + 110-security |
| **E-2 可观测性**   | ftrace + eBPF + perf + 4 层文件系统接口                            | 90-observability       |
| **E-3 资源确定性**  | 引用计数强制 + devm\_ 资源管理 + 内存分配惯用法                              | 01 代码规范 Part I + Part III      |
| **E-4 跨平台一致性** | 用户态跨 Linux/macOS/Windows + 内核态 Linux 6.6 基线                 | 04 工程思想                |
| **E-5 命名语义化**  | 全局函数描述性命名 + 局部变量短小 + 敏感术语禁用                                 | 01 代码规范                |
| **E-6 错误可追溯**  | Fixes\:/Closes\:/Link: 标签 + Signed-off-by DCO 链 + 12 字符 SHA | 05 开发流程                |
| **E-7 文档即代码**  | kernel-doc 强制 + ABI 文档化 + `make htmldocs` 验证                | 01 代码规范 + 06 工具链       |
| **E-8 可测试性**   | KUnit + kselftest + fault injection + 覆盖率门槛                 | 80-testing             |
| **A-1 极简主义**   | 反过度抽象 + 反 inline 滥用 + 反宏滥用                                  | 01 代码规范 Part III                |
| **A-2 细节关注**   | 行尾禁止空白 + 函数原型元素顺序 + 指针声明贴名                                  | 01 代码规范 Part II                |
| **A-3 人文关怀**   | 敏感术语禁用 + 审查礼仪 + 不烧桥管理哲学                                     | 01 代码规范 + 07 维护者制度     |
| **A-4 完美主义**   | 7 层验证 + 24 项提交检查清单 + Reviewed-by Statement                  | 05 开发流程 Part II + Part I       |

### 2.3 工程标准交叉引用矩阵

agentrt-linux 工程标准采用"规则层 + 实现层"双层设计，部分文档之间存在天然的职责分层关系。为避免读者误认为内容重复，下表明确各文档的职责边界：

#### 2.3.1 代码规范内部 Part 架构（01，6 Parts 合并）

> **合并说明（2026-07-13）**：原 `02-code-format.md`、`03-code-style.md`、`60-checkpatch-rule-map.md`、`70-kernel-doc-standard.md`、`80-clang-format-enforcement.md` 已合并入 `01-coding-standards.md` 的 6 个 Part。各 Part 职责分层清晰，规则层（Part I-III）定义"怎么写"，工具层（Part IV-VI）定义"怎么检查"。

| Part | 原 文件 | 职责层 | 内容范围 |
|------|---------|--------|---------|
| Part I | 01-coding-standards.md（原 §1-§6） | 语义层 | 命名/函数/注释/类型/错误处理的强制规则 |
| Part II | 02-code-format.md | 格式层 | 缩进/行宽/大括号/空格 + `.clang-format` 配置即代码 + CI 强制 `make format-check` |
| Part III | 03-code-style.md | 风格层 | 模块化/抽象层次/防御性/性能/工具链使用风格 |
| Part IV | 60-checkpatch-rule-map.md | 工具映射层 | `checkpatch.pl` ≥30 条规则 → OS-STD-CODE-NNN 映射 |
| Part V | 70-kernel-doc-standard.md | 文档标准层 | kernel-doc 注释格式 + `make htmldocs` 验证 + ABI 文档化 |
| Part VI | 80-clang-format-enforcement.md | CI 门禁层 | `.clang-format` 完整 YAML + CI 门禁流程 + 常见问题修复 |

> **交叉引用约定**：Part IV（checkpatch 规则映射）引用 Part I-III 的规则定义；Part VI（clang-format CI 门禁）引用 Part II 的格式规则。Part V（kernel-doc）引用 Part I §5 的注释规范。

#### 2.3.2 开发流程内部 Part 架构（05，4 Parts 合并）

> **合并说明（2026-07-13）**：原 `06-toolchain-and-automation.md`、`08-compliance-checklist.md`、`110-spdx-license-compliance.md` 已合并入 `05-development-process.md` 的 4 个 Part。

| Part | 原 文件 | 职责层 | 内容范围 |
|------|---------|--------|---------|
| Part I | 05-development-process.md（原 §1-§8） | 流程层 | 补丁生命周期/维护者层级/补丁格式/审查响应 |
| Part II | 06-toolchain-and-automation.md | 工具链层 | 7 层自动化验证体系（Layer 1-7）+ CI/CD 矩阵 + 覆盖率门槛 |
| Part III | 08-compliance-checklist.md | 合规检查层 | 17 项检查规则（STD-DOC/STD-CODE/STD-IRON）+ 16 个检查脚本 + CI/CD 集成 |
| Part IV | 110-spdx-license-compliance.md | 许可证层 | SPDX 标识符规范 + `SPDX-License-Identifier` 头部 + 双许可证选择策略 |

> **交叉引用约定**：05 Part II（工具链 Layer 3/4）引用 01 Part IV（checkpatch 规则）与 01 Part II（clang-format 配置）；05 Part III（合规检查）引用 01 全部 Part 的规则定义。

#### 2.3.3 维护者治理统一文档（07，原 90 已合并）

| 文档 | 职责层 | 内容范围 | 变更说明 |
|------|--------|---------|---------|
| **07-maintainers-and-governance.md** | 治理文档 | MAINTAINERS 字段定义 + Lieutenant System + 8 子仓分配 + DCO + Reviewer's Statement + 成熟度模型 + 审查礼仪 + 管理哲学 + 五维映射 | 原 90-maintainers-subsystem-model.md 已于 2026-07-09 合并入 §2.4 |

> **SSoT 对齐**：07 §10 为规则编号注册表的公开镜像，权威来源为 [09-ssot-registry.md](./09-ssot-registry.md)。07 不重新定义规则编号，仅引用 09 的编号。

### 2.4 Wave 2 v2 Phase D 验证结论

Wave 2 v2 Phase D（源码级深读审查的修复阶段）针对工程标准层 23 文档执行了 4 项关键验证与修复，全部通过：

| # | 验证项 | 结论 | 证据 |
|---|--------|------|------|
| D7（C-A07） | 陈旧注释路径修复 | ✅ 通过 | `170-performance/03-ipc-performance.md` 中陈旧注释路径 `ipc_msg_hdr.h` → `ipc.h` 已全部修复（5 处） |
| D9 | 系统性目录结构 SSoT 落地 | ✅ 通过 | 新建 `10-architecture/06-directory-structure.md`（~350 行），含 [SC] 物理隔离 + ES-OLK-1~13 落地 + seL4 设计模式对齐 |
| D10 | [SC] 头文件缩进验证 | ✅ 通过 | `include/uapi/airymax/*.h` 6 头文件 Tab 8 缩进对齐 OLK-6.6 §1 |
| D-SSoT | SSoT 物理宿主验证 | ✅ 通过 | `120-cross-project-code-sharing.md`（[SC] 共享契约层物理宿主）中 [SC] 代码段 69 行 Tab 缩进 / 0 行 4 空格 |

**关键约束确认**：

- [SC] 头文件必须 Tab 8 缩进（OLK-6.6 §1，对齐 `01-kernel.md` 源码品味），已通过 D10 验证
- [SC] 头文件必须最小 typedef（OLK-6.6 §5，仅 5 种例外）
- [SC] 头文件变更必须双向 CI 校验（IRON-9 v2）
- [SC] 头文件物理宿主为 `120-cross-project-code-sharing.md` SSoT
- 23 文档精简完成（2026-07-13），`09-ssot-registry.md` 作为唯一 SSoT 注册表（2026-07-12 SSoT 重新设计，取消三层镜像）

> **代码引用**：
> - [SC] 头文件物理宿主：`file:///home/spharx/SpharxWorks/OpenAirymax/docs/AirymaxOS/50-engineering-standards/120-cross-project-code-sharing.md`
> - 系统性目录结构 SSoT：`file:///home/spharx/SpharxWorks/OpenAirymax/docs/AirymaxOS/10-architecture/06-directory-structure.md`
> - SSoT 注册表：`file:///home/spharx/SpharxWorks/OpenAirymax/docs/AirymaxOS/50-engineering-standards/09-ssot-registry.md`
> - IPC 性能（D7 修复对象）：`file:///home/spharx/SpharxWorks/OpenAirymax/docs/AirymaxOS/170-performance/03-ipc-performance.md`

### 2.5 OLK-6.6 工程标准对齐差距（1.0.1 M1+）

Wave 2 v2 Phase C 统一差距清单中的 D 类（OLK-6.6 工程标准）8 项差距（C-D01~C-D08，预估工时 ~18h），移交至 1.0.1 M1+ 阶段实施。这些差距源于 B1 OLK-6.6 源码深读中识别出的 8 项新发现，需要逐项落地到工程标准规范中：

| # | 差距编号 | 主题 | 源码标杆 | 落地目标 | 预估工时 |
|---|---------|------|---------|---------|---------|
| 1 | C-D01 | Makefile.oever 工程规范 | OLK-6.6 `Makefile.oever`（openEuler 发行版构建扩展） | 70-build-system 工程标准 | ~2h |
| 2 | C-D02 | Kconfig.openeuler 配置规范 | OLK-6.6 `Kconfig.openeuler`（发行版专属 CONFIG 命名空间） | 70-build-system + 50 工程标准 | ~2h |
| 3 | C-D03 | atomic SHA1 完整性校验 | OLK-6.6 atomic SHA1 实现模式 | 110-security + 50 工程标准 | ~3h |
| 4 | C-D04 | missing-syscalls 检查规范 | OLK-6.6 `scripts/missing-syscalls` 工具 | 05 开发流程 Part II 工具链层 | ~2h |
| 5 | C-D05 | .rustfmt.toml Rust 格式化 | OLK-6.6 `.rustfmt.toml`（edition 2021） | 10-coding-style/Rust_coding_style.md | ~2h |
| 6 | C-D06 | EXPORT_TRACEPOINT 符号导出规范 | OLK-6.6 `EXPORT_TRACEPOINT` 宏使用模式 | 01 代码规范 Part III + 10-coding-style | ~2h |
| 7 | C-D07 | checkpatch 100 列行宽规范 | OLK-6.6 `checkpatch.pl` 100 列扩展（vs 上游 80 列） | 01 代码规范 Part II + Part IV | ~2h |
| 8 | C-D08 | lockdep_assert 锁断言规范 | OLK-6.6 `lockdep_assert*` 系列宏（运行时不变式） | 01 代码规范 Part III + 10-coding-style | ~3h |
| | **合计** | | | | **~18h** |

**移交说明**：

- **触发条件**：B1 OLK-6.6 深读识别出 8 项新发现（Makefile.oever / Kconfig.openeuler / atomic SHA1 / missing-syscalls / .rustfmt.toml / EXPORT_TRACEPOINT / checkpatch 100列 / lockdep_assert），均源自 Linux 6.6 内核源码级证据
- **优先级**：D 类（Engineering Standards），不阻塞 0.1.1 文档体系发布，但需在 1.0.1 M1 编码启动前完成落地
- **责任主体**：工程规范委员会（待成立，详见 07-maintainers-and-governance.md）
- **验收标准**：每项差距需在工程标准文档中形成可执行的规则条目（OS-STD-* / OS-KER-* / OS-BAN-* 编号），并配套 CI 检查规则

> **代码引用**：OLK-6.6 内核源码（Linux 6.6 内核源代码，OLK 分支，B1 深读基础）

***

## 3. 代码规范主题（详见 01-coding-standards.md）

### 3.1 核心规则摘要

**命名规范**：

- 全局函数与全局变量必须描述性命名（`count_active_users()` 而非 `cntusr()`）
- 局部变量短小精悍（`i`、`tmp`、`buf`）
- 禁止匈牙利命名法（编译器已知类型）
- 禁止敏感术语：`master/slave` → `primary/secondary`，`blacklist/whitelist` → `denylist/allowlist`
- 宏名常量全大写（`#define CONSTANT 0x12345`），相关常量优先 `enum`

**函数规范**：

- 函数长度一屏可读（≤80x24），局部变量 ≤5-10 个
- 函数原型元素固定顺序：storage class → attributes → return type → name → params → behavior attributes
- 返回值约定：动作式函数返回错误码（0=成功，-Exxx=失败），谓词式函数返回 bool
- 所有 EXPORTed 函数必须遵守返回值约定

**错误处理（强制范式）**：

- `goto out_free_xxx:` 集中出口模式 + **分级标签按分配逆序释放**
- 禁止 `BUG()`/`BUG_ON()`，改用 `WARN()`/`WARN_ON_ONCE()` 并提供恢复代码
- 内存分配用 `sizeof(*p)` 而非 `sizeof(struct xxx)`
- 数组分配用 `kmalloc_array`/`kcalloc`（含溢出检查）
- 禁止 `strcpy`/`strncpy`/`strlcpy`，强制 `strscpy`
- 禁用 `%p` 默认输出（哈希化），新增 `%p` 不被接受

**类型规范**：

- 内核内部 `u8/u16/u32/u64`，用户空间可见结构体用 `__u32` 等
- typedef 严格限制（仅 5 种例外：完全不透明对象、清晰整数类型、sparse 新类型、C99 等价、UAPI 安全类型）
- bool 仅用于函数返回值与栈变量，对齐敏感的结构体不用 bool

### 3.2 agentrt-linux 专属扩展

- **多语言命名规范**：C/Rust/Python/TypeScript 各自的命名风格章节
- **Agent 接口命名**：`airy_*` 前缀保留给 agentrt 同源 API，`airy_*` 前缀用于 agentrt-linux 专属
- **弃用接口清单**：维护 `deprecated-interfaces.md` 动态清单（类 Linux `deprecated.rst`）

***

## 4. 代码格式（详见 01-coding-standards.md Part II）

### 4.1 核心规则摘要

- **缩进**：Tab，8 字符宽（C 内核代码）；其他语言遵循各自规范（Rust 4 空格、Python 4 空格 PEP 8、TypeScript 2 空格）
- **行宽**：首选 80 列（C）；其他语言 100-120 列
- **大括号**：K\&R 风格（非函数块开括号行末，函数开括号行首）
- **指针声明**：`*` 贴名字不贴类型（`char *p` 而非 `char* p`）
- **空格**：关键字后加空格（`if (`），括号内禁止空格，二元运算符两侧加空格
- **行尾禁止空白**

### 4.2 配置即代码

- `.clang-format`（C/C++，689 行 + ForEachMacros 列表）
- `.rustfmt.toml`（Rust，edition 2021）
- `.pyproject.toml`（Python，Black + isort）
- `.prettierrc`（TypeScript/JavaScript）
- CI 中强制 `make format-check`

***

## 5. 代码风格（详见 01-coding-standards.md Part III）

### 5.1 核心思想

- **反过度抽象**：抽象只在必要时使用，不预先抽象；函数参数全为 0 的"未来扩展"应删除
- **策略与机制分离**：机制留在内核（提供能力），策略外移至用户空间或可插拔模块（如 sched\_ext BPF 调度器）
- **引用计数强制**：内核无 GC，多线程可见数据结构必须有引用计数；锁不替代引用计数
- **反 inline 滥用**：超过 3 行的函数不应 inline；static 且仅用一次的函数编译器会自动 inline
- **cache 影响是头等大事**：空间即时间，更大的程序运行更慢

### 5.2 agentrt-linux 专属风格

- **热路径 vs 冷路径分层**：热路径（调度、内存、IPC）用 C + 严格性能预算；冷路径（配置、日志格式化）可放宽 inline 限制
- **Rust 内核模块**：仅用于安全敏感且非热路径的子系统（如安全策略、文件系统解析）
- **多语言协作风格**：FFI 边界必须显式声明，禁止隐式类型转换

***

## 6. 工程思想主题（详见 04-engineering-philosophy.md）

### 6.1 双层稳定性哲学（核心思想）

**用户空间 ABI 永不破坏**：

- 一旦接口导出至用户空间，必须永久支持
- 用户空间接口设计必须深思熟虑、清晰文档、广泛审查
- 所有用户空间接口改动必须经过专门的 ABI 审查流程

**内核内部 API 不保证稳定**：

- 这是 deliberate design decision（深思熟虑的设计决策），允许持续重构
- 配套规则：API 改动者必须同时修复所有受影响代码（"you broke it, you fix it"）
- 这是降低主线维护成本的核心机制

### 6.2 agentrt-linux 4 层接口稳定性分级

agentrt-linux 在双层稳定性基础上扩展为 4 层：

| 层级 | 接口类型                        | 稳定性                             | 变更流程                 |
| -- | --------------------------- | ------------------------------- | -------------------- |
| L1 | Agent 应用 API（SDK + syscall） | 极稳定（类 Linux syscall）            | RFC + 6 个月宽限期 + 弃用声明 |
| L2 | 智能体运行时接口（AgentsIPC 协议）      | 中等稳定（版本化演进）                     | 季度评审 + 兼容性测试         |
| L3 | 内核子系统接口（内核内部 API）           | 可重构（"you broke it, you fix it"） | 补丁序列修复所有调用点          |
| L4 | 内部实现细节                      | 完全自由                            | 无约束                  |

### 6.3 其他核心工程思想

- **策略与机制分离**（K-4 可插拔策略）
- **渐进式开发与持续集成**（S-4 涌现性管理）
- **稳定性优先：regression 不可接受**（E-6 错误可追溯）
- **审查优先文化**（Reviewer's Statement of Oversight 等价物）
- **可追溯性**（Fixes/Closes/Link/DCO 标签体系）
- **文档即代码**（E-7）
- **不破坏用户空间原则**

***

## 7. 开发流程主题（详见 05-development-process.md）

### 7.1 补丁生命周期

```
Design → Early review → Wider review → Mainline → Stable release → Long-term maintenance
```

每个阶段都有明确的输入、输出、责任人和 SLA。

### 7.2 agentrt-linux 适配

- **SCM**：git + GitHub PR（替代邮件 + git send-email）
- **审查工具**：GitHub PR review + CODEOWNERS + 必填 PR 模板
- **DCO 验证**：DCO bot 自动验证 Signed-off-by 链条
- **标签体系**：保留 `Fixes:`/`Closes:`/`Link:`/`Reviewed-by:`/`Acked-by:`/`Tested-by:` 等（PR 评论形式）
- **预览集成分支**：`develop` 分支等价 linux-next 树
- **稳定版分支**：`release/0.1.x`、`release/1.0.x` 等价 -stable 树

***

## 8. OS 工程规则编号体系

agentrt-linux 在 agentrt 17 类规则编号体系基础上，新增 OS 专属编号前缀：

| 编号前缀         | 含义                       | 与 agentrt 的关系                           |
| ------------ | ------------------------ | --------------------------------------- |
| **OS-IRON**  | agentrt-linux 工程铁律（不可妥协） | 继承 agentrt IRON-1\~10 + 新增 OS-IRON-1\~N |
| **OS-BAN**   | agentrt-linux 禁止规则       | 继承 agentrt BAN-1\~356 + 新增 OS-BAN-1\~N  |
| **OS-STD**   | agentrt-linux 标准规则       | 继承 agentrt STD + OS 专属扩展                |
| **OS-ACC**   | agentrt-linux 验收标准       | 继承 agentrt ACC-1\~149 + OS 专属验收         |
| **OS-KER**   | agentrt-linux 内核工程规则     | 全新（agentrt 不涉及内核态）                      |
| **OS-DRV**   | agentrt-linux 驱动工程规则     | 全新                                      |
| **OS-BUILD** | agentrt-linux 构建工程规则     | 全新                                      |
| **OS-TEST**  | agentrt-linux 测试工程规则     | 全新                                      |
| **OS-OBS**   | agentrt-linux 可观测性规则     | 全新                                      |
| **OS-SEC**   | agentrt-linux 安全工程规则     | 全新                                      |
| **OS-ABI**   | agentrt-linux ABI 稳定性规则  | 全新                                      |

具体规则编号在各个子文档中定义，汇总于 `09-ssot-registry.md`（唯一权威 SSoT 注册表）。

***

## 9. 工程标准实施路线

### 9.1 0.1.1 版本（奠基）

- 完成 50-engineering-standards/ 全部 23 文档（含本 README + 顶层 8 文档 + 10-coding-style 6 文档 + 其余 4 子目录各 2 文档，2026-07-13 精简合并后）
- 完成 130-roadmap/ 全部 7 文档
- 完成其他 P0 模块（60-120）的 README.md 占位
- 不要求实施（实施在 1.0.1 版本）

### 9.2 1.0.1 版本（开发）

- 完成 19 模块全部 \~140 文档
- 工程标准全面实施
- 7 层自动化验证全部就位
- 维护者制度与治理落地

### 9.3 实施优先级

| 优先级 | 模块                        | 文档数 | 0.1.1 范围         |
| --- | ------------------------- | --- | ---------------- |
| P0  | 50-engineering-standards/ | 23  | 全部完成             |
| P0  | 130-roadmap/              | 7   | 全部完成             |
| P0  | 60-driver-model/          | 7   | README + 01 + 02 |
| P0  | 70-build-system/          | 8   | README + 01 + 02 |
| P0  | 80-testing/               | 10  | README + 01 + 02 |
| P0  | 90-observability/         | 9   | README + 01 + 02 |
| P0  | 100-operations/           | 10  | README + 01 + 02 |
| P0  | 110-security/             | 9   | README + 01 + 02 |
| P0  | 120-development-process/  | 9   | README + 01 + 02 |
| P1  | 140-170/                  | 33  | 仅 README         |
| P2  | 180-190/                  | 12  | 仅 README         |

**0.1.1 版本总产出**：\~85 文档（工程标准 23 + 根 README 1 + 00-requirements 4 + 10-architecture 6 + 20-modules 9 + 30-interfaces 5 + 40-dataflows 4 + P0 模块 21 + 路线图 7 + P1 模块 4 + P2 模块 2 ≈ 86）。

***

## 10. 相关文档

### 10.1 同源 Airymax 文档

- `docs/AirymaxRT/00-architectural-principles.md`（五维正交 24 原则）
- `docs/AirymaxOS/50-engineering-standards/10-coding-style/`（编码规范子目录，含 C/C++ + Rust + Go + 脚本语言 + 通用编码约定共 6 文档，2026-07-12 精简合并后）
- `docs/AirymaxOS/50-engineering-standards/90-terminology.md`（术语规范 SSoT，2026-07-12 从 `docs/AirymaxOS/TERMINOLOGY.md` 移入并重命名）
- IRON-9 v2 工程铁律（17 类规则编号体系，v28.0）
- agentrt 工程改进方案（v4.0）

### 10.2 agentrt-linux 设计文档

- `00-requirements/`（业务/功能/非功能需求）
- `10-architecture/`（系统架构 + 微内核策略 + 工程哲学）
- `20-modules/`（8 子仓模块设计）
- `30-interfaces/`（系统调用 + IPC + SDK + 编码规范）
- `40-dataflows/`（认知流 + 内存流 + IPC 流 + 调度流）
- `60-driver-model/`（驱动模型设计）
- `70-build-system/`（构建系统设计）
- `80-testing/`（测试体系设计）
- `90-observability/`（可观测性设计）
- `100-operations/`（运维体系设计）
- `110-security/`（安全加固设计）
- `120-development-process/`（开发流程设计）
- `130-roadmap/`（路线图与里程碑）

### 10.3 参考材料

- `Linux 6.6 内核源码`（Linux 6.6 内核源代码，OLK 分支，深度研究基础）
- Linux 内核 `Documentation/process/`（coding-style.rst、submitting-patches.rst、stable-api-nonsense.rst 等）
- Linux 内核 `.clang-format`（689 行 + 560 ForEachMacros）
- Linux 内核 `MAINTAINERS`（734KB，维护者制度范本）

***

## 11. 文档版本与维护

- **当前版本**: v1.4（2026-07-13，Wave 2 v2 Phase D 审查状态声明 + §2.4 Phase D 验证结论 + §2.5 OLK-6.6 工程标准对齐差距 C-D01~C-D08 移交 1.0.1 M1+；修正子目录文档计数（5 目录 = 14 文档））
- **历史版本**: v1.3（2026-07-13，50-engineering-standards 精简合并 53→23 文档：B 组合并顶层 8→2+删除 1 / C 组合并 10-coding-style 17→6 / D 组合并 30+40+50 子目录各 4→2 / 20-contracts 4→2；更新目录树、章节定位表、五维映射、交叉引用矩阵）
- **维护者**: 工程规范委员会（待成立，详见 07-maintainers-and-governance.md）
- **变更流程**: 任何工程标准变更必须经过 RFC → 评审 → ACC 验收流程
- **回顾周期**: 季度回顾 + 年度大版本

***

> **文档结束** | 共 23 文档（含本 README + 顶层 8 文档 + 10-coding-style 6 文档 + 其余 4 子目录各 2 文档）| 0.1.1 版本 P0 优先完成 | 2026-07-13 精简合并

