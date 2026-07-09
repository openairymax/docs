Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）工程标准规范

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）工程标准规范的主索引与总纲
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-07
> **同源映射**: agentrt `docs/ARCHITECTURAL_PRINCIPLES.md`（五维正交 24 原则）+ IRON-9 v2 工程铁律（内部工程标准规范，17 类规则编号体系）
> **理论根基**: Linux 6.6 内核工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）

---

## 1. 工程标准定位

### 1.1 为什么需要 agentrt-liunx 独立的工程标准

agentrt-liunx 作为基于 Linux 内核的智能体操作系统发行版，其工程标准必须同时满足三重诉求：

1. **内核工程严肃性**——继承 Linux 内核 30+ 年沉淀的工程思想（不破坏用户空间 ABI、不维护稳定内部 API、goto 集中错误处理、强制工具链、信任链分层维护者制度），任何脱离此基础的"自创"规范都会引入系统性风险。
2. **智能体场景适配**——传统内核工程规范面向"硬件资源抽象与调度"场景，而 agentrt-liunx 面向"智能体工作负载"场景，需要在内核严肃性之上扩展：多语言栈（C/Rust/Python/TS）、多层接口稳定性分级、Agent 行为契约测试、Token 能效可观测性等专属规范。
3. **Airymax 同源约束**——与 agentrt（AirymaxAgentRT）共享 Airymax 设计理念（MicroCoreRT / AgentsIPC / Cupolas / MemoryRovol / CoreLoopThree），工程标准必须保证同源语义在两端（用户态运行时 + OS 发行版）表述一致、可互操作。

### 1.2 工程标准三大支柱

| 支柱 | 思想来源 | 落地章节 |
|------|---------|---------|
| **Linux 内核工程思想** | Linux 6.6 内核源代码（OLK-6.6）深度研究提炼 | §3-§7 五大主题章节 |
| **Airymax 五维正交 24 原则** | `ARCHITECTURAL_PRINCIPLES.md`（S/K/C/E/A 五维） | §4 工程思想章节 |
| **agentrt 17 类规则编号体系** | `0.1.1工程标准规范手册.md`（IRON/BAN/FOUND 等） | §8 OS 工程规则编号体系 |

### 1.3 与 agentrt 工程规范的关系（同源且部分代码共享，IRON-9 v2）

agentrt-liunx 工程标准与 agentrt 工程标准是**同源且部分代码共享**的关系（IRON-9 v2，2026-07-07 用户决策变更）：

#### 三层共享模型

| 层次 | 共享程度 | 内容 | 组织方式 |
|------|---------|------|---------|
| **共享契约层**（Shared-Contract） | 完全共享代码 | **6 个 [SC] 头文件清单**（`include/airymax/`）：(1) `bpf_struct_ops.h`——struct_ops 状态机 + common_value；(2) `memory_types.h`——MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 持久化接口；(3) `security_types.h`——POSIX capability 38 ID + LSM 钩子 254 ID + Cupolas blob 布局 + capability 派生模型 + Vault backend + 策略裁决 4 值枚举；(4) `cognition_types.h`——CoreLoopThree 阶段枚举 + Thinkdual 模式枚举 + LLM 推理阶段枚举 + 上下文结构 + Token 能效指标 + GPU/NPU 描述符；(5) `sched.h`——SCHED_EXT 调度类编号约束（复用内核 SCHED_EXT=7，禁止 SCHED_AGENT 宏）+ 任务描述符（magic 0x41475453 'AGTS'）+ vtime 类型与衰减公式 + 优先级范围 + AIRYMAX_SLICE_DFL；(6) `ipc.h`——IPC magic（0x41524531 'ARE1'）+ 128B 消息头结构（`agentrt_ipc_msg_hdr_t`）+ SQE/CQE 操作码与标志位。另含 syscall 编号（`AGENTRT_SYS_*`）、capability 令牌格式、错误码（`agentrt_error_t`）、规则编号体系（IRON/BAN/STD/ACC）、五维正交 24 原则 | `include/airymax/` 独立头文件库，两端共同依赖 |
| **语义同源层**（Shared-Semantics） | API 签名相同，实现独立 | 调度语义（MicroCoreRT：agentrt 用户态 vs agentrt-liunx sched_ext）、安全模型（Cupolas：agentrt 用户态 vs agentrt-liunx LSM）、IPC 传输（AgentsIPC：agentrt 消息队列 vs agentrt-liunx io_uring）、记忆模型（MemoryRovol：agentrt heapstore vs agentrt-liunx 内核态） | 各自独立实现，通过契约层保证互操作 |
| **完全独立层**（Independent） | 完全独立 | agentrt-liunx 专属：内核驱动框架、Kbuild、systemd 集成、内核内部 API；agentrt 专属：跨平台用户态运行时、SDK 四语言、CLI/TUI | 各自独立仓库，无依赖关系 |

#### 与旧版 IRON-9 的差异

| 维度 | 旧版 IRON-9（仅语义同源、实现独立） | 新版 IRON-9 v2（同源且部分代码共享） |
|------|-------------------------|-------------------------------|
| 设计哲学 | 共享 | 共享（不变） |
| 规则编号 | 共享骨架 | 共享骨架（不变） |
| 契约层代码 | **不共享**，仅语义同源 | **完全共享**（`include/airymax/` 头文件库） |
| 实现层代码 | 独立 | 独立（不变） |
| 互操作方式 | 通过同源语义无适配层 | **通过共享代码无适配层**（增强） |
| CI 校验 | 无 | 契约层变更双向校验（agentrt + agentrt-liunx CI 同时验证） |

- **同源**：共享 Airymax 五维正交 24 原则作为顶层设计哲学；共享 17 类规则编号体系骨架；共享 E-7 文档即代码、E-6 错误可追溯、E-1 安全内生等核心工程观原则。**共享契约层代码**（`include/airymax/` 头文件库）。
- **独立**：agentrt-liunx 是 OS 发行版，承担内核态严肃性责任，其工程标准必须独立处理内核 ABI 稳定性、内核内部 API 不稳定性、补丁生命周期、维护者层级制度等 agentrt 不涉及的领域；同时 agentrt-liunx 需要为内核态特有的场景（驱动模型、构建系统、可观测性、安全 LSM 等）制定专门规范。
- **互操作**：agentrt 在 agentrt-liunx 上运行时，两端工程标准形成天然互补——agentrt 遵循其用户态运行时规范（如 `agentrt_log_write` 日志），agentrt-liunx 遵循其内核发行版规范（如 `printk` 等级），两端通过**共享契约层代码**（如 `include/airymax/ipc.h` 的 IPC 消息头定义、`include/airymax/sched.h` 的任务描述符定义）实现无适配层互操作。

### 1.4 主流 Linux 发行版标准兼容性声明（2026-07-07 用户决策）

agentrt-liunx（AirymaxOS）的工程思想与实现方式与上游 Linux 保持一致性，以便未来兼容 主流 Linux 发行版标准的发行版工具链。

#### 兼容性层次

| 层次 | 兼容内容 | 兼容方式 | 代码共享 |
|------|----------|----------|----------|
| **协议层兼容** | kthread 创建/停止/驻留协议、IPC 消息传递协议、capability 令牌协议 | 同源协议设计，[IND] 独立实现 | **不共享代码** |
| **接入层兼容** | 加速器接入模式（accel 框架）、GPU 调度器模式（drm_sched）、eBPF 可编程内核 | 参考设计模式，[IND] 独立实现 | **不共享代码** |
| **模块化兼容** | Kconfig 风格、模块组织、Makefile 结构、Kbuild 规范 | 同源工程规范 | **不共享代码** |
| **工具链兼容** | GCC/Clang/Rust 工具链、checkpatch/spatch、RPM/dnf 包管理 | 兼容 主流 Linux 发行版工具链 | 工具链共享（开源生态） |

#### 代码共享边界澄清

- **agentrt ↔ agentrt-liunx**：[SC] 共享契约层完全共享代码（`include/airymax/` 6 个头文件）
- **agentrt-liunx ↔ Linux 6.6 基线**：**仅技术参考，不共享代码**——OLK-6.6 的 kthread.c/accel.c/sched_main.c 等是 agentrt-liunx [IND] 独立层的实现参考
- agentrt-liunx 工程思想与上游 Linux 一致性体现在协议层（创建/停止/驻留协议）和接入层（accel/drm_sched 模式），不体现在代码层

#### 兼容性验证检查项

| 检查项 | 验证方法 | 合格标准 |
|--------|----------|----------|
| kthread 协议一致性 | 核对 kthread_create/stop/park API 签名 | 与 主流 Linux 发行版标准签名一致 |
| 加速器接入一致性 | 核对 drivers/accel/ 模式 | 接入模式一致（[IND] 独立实现） |
| Kconfig 风格一致性 | 核对 CONFIG_ 前缀命名 | 与上游 Linux Kconfig 风格一致 |
| 模块化一致性 | 核对 Makefile/Kbuild 结构 | 与上游 Linux 构建结构一致 |
| 调度类编号不冲突 | 核对 SCHED_AGENT 编号 | 不与上游 Linux 已有调度类冲突 |

> **详见**：内部工程标准规范（闭源）各子系统技术规范

---

## 2. 工程标准框架（5 主题 + 2 治理）

agentrt-liunx 工程标准由 5 个主题章节 + 2 个治理章节 + 1 个合规性检查章节构成，共 8 个子文档（含本 README 共 9 文档）：

```
50-engineering-standards/
├── README.md                         # 本文件 — 主索引与总纲
├── 00-engineering-standards-handbook.md  # 工程标准规范手册（SSoT 索引 + IRON 铁律 + SSoT 注册表）
├── 01-coding-standards.md            # 代码规范（命名/函数/注释/类型/错误处理）
├── 02-code-format.md                  # 代码格式（缩进/行宽/大括号/空格/clang-format）
├── 03-code-style.md                   # 代码风格（模块化/抽象层次/防御性/性能/工具链）
├── 04-engineering-philosophy.md       # 工程思想（双层稳定性/策略机制分离/渐进式/审查优先）
├── 05-development-process.md          # 开发流程（补丁生命周期/维护者层级/补丁格式/审查）
├── 06-toolchain-and-automation.md     # 工具链与自动化（7 层验证/checkpatch/CI/CD）
├── 07-maintainers-and-governance.md  # 维护者制度与治理（MAINTAINERS/层级/成熟度模型/DCO）
├── 08-compliance-checklist.md         # 规范符合性检查机制（STD-DOC/STD-CODE/STD-IRON）
│
├── 10-coding-style/                   # 编码规范子目录（原 Capital_Specifications/coding_standard/）
│   ├── README.md                      # 编码规范总览与导航
│   ├── C_coding_style_standard.md     # C 编码风格规范
│   ├── C_Cpp_secure_coding_standard.md # C/C++ 安全编码规范
│   ├── Rust_coding_style_standard.md  # Rust 编码风格规范
│   └── Rust_secure_coding_standard.md # Rust 安全编码规范
│
├── 20-contracts/                      # 契约规范子目录（原 Capital_Specifications/agentrt_contract/）
│   ├── README.md                      # 契约规范总览与导航
│   ├── ipc_protocol_contract.md       # IPC 协议契约
│   ├── logging_contract.md           # 日志契约
│   └── syscall_api_contract.md       # 系统调用 API 契约
│
├── 30-runtime-interfaces/             # 运行时接口子目录（原 Capital_Specifications/are_standards/）
│   ├── README.md                      # 运行时接口总览与导航
│   ├── L1_runtime_interface.md        # L1 运行时接口规范
│   ├── L2_service_protocol.md         # L2 服务协议规范
│   └── L3_security_governance.md      # L3 安全治理规范
│
├── 40-integration/                    # 集成规范子目录（原 Capital_Specifications/integration_standards/）
│   ├── README.md                      # 集成规范总览与导航
│   └── agentrt_integration.md         # agentrt 集成规范
│
└── 50-project-erp/                    # 项目工程管理子目录（原 Capital_Specifications/project_erp/）
    ├── README.md                      # 项目工程管理总览与导航
    ├── error_code_reference.md        # 错误码参考
    ├── SBOM.md                        # 软件物料清单
    ├── manuals_module_requirements.md # 手册与模块需求
    └── resource_management_table.md   # 资源管理表
```

> **迁移说明**：`10-coding-style/` 至 `50-project-erp/` 五个子目录由原 `Capital_Specifications/` 目录迁移而来（2026-07-08 完成迁移，原 `Capital_Specifications/` 目录已删除）。`TERMINOLOGY.md` 已提升至 `docs/AirymaxAgentOS/` 顶层。

### 2.1 章节定位

| 章节 | 核心问题 | 来源 |
|------|---------|------|
| 01 代码规范 | "代码该怎么写"——命名、函数、注释、类型、错误处理的强制规则 | Linux `coding-style.rst`（1271 行）+ Airymax `coding_standard/`（15 文件）|
| 02 代码格式 | "代码长什么样"——缩进、行宽、大括号、空格的机械规则 + 自动化配置 | Linux `.clang-format`（689 行 + 560 ForEachMacros）+ Airymax 格式规范 |
| 03 代码风格 | "代码该怎么思考"——模块化、抽象层次、防御性、性能、工具链使用风格 | Linux `4.Coding.rst` + `stable-api-nonsense.rst` + Airymax 五维原则 |
| 04 工程思想 | "工程体系怎么建立"——稳定性哲学、策略机制分离、渐进式、审查优先 | Linux `stable-api-nonsense.rst` + Airymax K-1~K-4 内核观 + E-1~E-8 工程观 |
| 05 开发流程 | "代码怎么进入主线"——补丁生命周期、维护者层级、补丁格式、审查响应 | Linux `development-process.rst`（8 章）+ Airymax 17 类规则 |
| 06 工具链与自动化 | "规范怎么被执行"——7 层验证、checkpatch、CI/CD、覆盖率门槛 | Airymax 7 层自动化验证 + Linux `submit-checklist.rst`（24 项） |
| 07 维护者制度与治理 | "项目怎么被治理"——MAINTAINERS、层级、成熟度模型、DCO | Linux `MAINTAINERS`（734KB）+ Airymax IRON-9 v2 同源且部分代码共享 |
| 08 规范符合性检查 | "规范怎么被验证"——17 项检查规则（STD-DOC/STD-CODE/STD-IRON）+ 16 个检查脚本 + CI/CD 集成 | Airymax 工程标准合规性验证机制 |

### 2.2 与 Airymax 五维正交 24 原则的映射

agentrt-liunx 工程标准是 Airymax 五维正交 24 原则在 OS 发行版场景下的具体落地：

| 五维原则 | 在 agentrt-liunx 工程标准中的体现 | 落地章节 |
|---------|----------------------------|---------|
| **S-1 反馈闭环** | 7 层自动化验证的每一层都是反馈闭环；CI 失败即反馈 | 06 工具链与自动化 |
| **S-2 层次分解** | 4 层接口稳定性分级（Agent API / Runtime API / 内核子系统 API / 内部实现） | 04 工程思想 |
| **S-3 总体设计部** | 维护者层级制度（Lieutenant System）+ 总维护者角色 | 07 维护者制度与治理 |
| **S-4 涌现性管理** | 渐进式开发模型 + Merge Window + RC 周期 | 05 开发流程 |
| **K-1 内核极简** | 微内核化改造策略（用户态服务化）+ 内核职责最小化 | 04 工程思想 |
| **K-2 接口契约化** | 双层稳定性哲学（用户 ABI 稳定 / 内部 API 可重构） | 04 工程思想 |
| **K-3 服务隔离** | 驱动用户态化 + 服务进程隔离 + capability 安全 | 03 代码风格 + 110-security |
| **K-4 可插拔策略** | sched_ext BPF 调度器 + LSM 钩子 + 模块签名 | 03 代码风格 |
| **C-1 双系统协同** | 快慢路径分层（热路径 C / 慢路径 Rust）+ 调度类可插拔 | 03 代码风格 |
| **C-2 增量演化** | 补丁序列中点可编译运行 + git bisect 友好 | 05 开发流程 |
| **C-3 记忆卷载** | 多代 LRU（MGLRU）+ 分级内存（CXL/PMEM） | 170-performance |
| **C-4 遗忘机制** | 弃用接口清单（Deprecated Interfaces Registry）+ ABI 弃用流程 | 01 代码规范附录 |
| **E-1 安全内生** | 强制安全编码 + fault injection + LSM 集成 | 01 代码规范 + 110-security |
| **E-2 可观测性** | ftrace + eBPF + perf + 4 层文件系统接口 | 90-observability |
| **E-3 资源确定性** | 引用计数强制 + devm_ 资源管理 + 内存分配惯用法 | 01 代码规范 + 03 代码风格 |
| **E-4 跨平台一致性** | 用户态跨 Linux/macOS/Windows + 内核态 Linux 6.6 基线 | 04 工程思想 |
| **E-5 命名语义化** | 全局函数描述性命名 + 局部变量短小 + 敏感术语禁用 | 01 代码规范 |
| **E-6 错误可追溯** | Fixes:/Closes:/Link: 标签 + Signed-off-by DCO 链 + 12 字符 SHA | 05 开发流程 |
| **E-7 文档即代码** | kernel-doc 强制 + ABI 文档化 + `make htmldocs` 验证 | 01 代码规范 + 06 工具链 |
| **E-8 可测试性** | KUnit + kselftest + fault injection + 覆盖率门槛 | 80-testing |
| **A-1 极简主义** | 反过度抽象 + 反 inline 滥用 + 反宏滥用 | 03 代码风格 |
| **A-2 细节关注** | 行尾禁止空白 + 函数原型元素顺序 + 指针声明贴名 | 02 代码格式 |
| **A-3 人文关怀** | 敏感术语禁用 + 审查礼仪 + 不烧桥管理哲学 | 01 代码规范 + 07 维护者制度 |
| **A-4 完美主义** | 7 层验证 + 24 项提交检查清单 + Reviewed-by Statement | 06 工具链 + 05 开发流程 |

---

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

### 3.2 agentrt-liunx 专属扩展

- **多语言命名规范**：C/Rust/Python/TypeScript 各自的命名风格章节
- **Agent 接口命名**：`agentrt_*` 前缀保留给 agentrt 同源 API，`airymaxos_*` 前缀用于 agentrt-liunx 专属
- **弃用接口清单**：维护 `deprecated-interfaces.md` 动态清单（类 Linux `deprecated.rst`）

---

## 4. 代码格式主题（详见 02-code-format.md）

### 4.1 核心规则摘要

- **缩进**：Tab，8 字符宽（C 内核代码）；其他语言遵循各自规范（Rust 4 空格、Python 4 空格 PEP 8、TypeScript 2 空格）
- **行宽**：首选 80 列（C）；其他语言 100-120 列
- **大括号**：K&R 风格（非函数块开括号行末，函数开括号行首）
- **指针声明**：`*` 贴名字不贴类型（`char *p` 而非 `char* p`）
- **空格**：关键字后加空格（`if (`），括号内禁止空格，二元运算符两侧加空格
- **行尾禁止空白**

### 4.2 配置即代码

- `.clang-format`（C/C++，689 行 + ForEachMacros 列表）
- `.rustfmt.toml`（Rust，edition 2021）
- `.pyproject.toml`（Python，Black + isort）
- `.prettierrc`（TypeScript/JavaScript）
- CI 中强制 `make format-check`

---

## 5. 代码风格主题（详见 03-code-style.md）

### 5.1 核心思想

- **反过度抽象**：抽象只在必要时使用，不预先抽象；函数参数全为 0 的"未来扩展"应删除
- **策略与机制分离**：机制留在内核（提供能力），策略外移至用户空间或可插拔模块（如 sched_ext BPF 调度器）
- **引用计数强制**：内核无 GC，多线程可见数据结构必须有引用计数；锁不替代引用计数
- **反 inline 滥用**：超过 3 行的函数不应 inline；static 且仅用一次的函数编译器会自动 inline
- **cache 影响是头等大事**：空间即时间，更大的程序运行更慢

### 5.2 agentrt-liunx 专属风格

- **热路径 vs 冷路径分层**：热路径（调度、内存、IPC）用 C + 严格性能预算；冷路径（配置、日志格式化）可放宽 inline 限制
- **Rust 内核模块**：仅用于安全敏感且非热路径的子系统（如安全策略、文件系统解析）
- **多语言协作风格**：FFI 边界必须显式声明，禁止隐式类型转换

---

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

### 6.2 agentrt-liunx 4 层接口稳定性分级

agentrt-liunx 在双层稳定性基础上扩展为 4 层：

| 层级 | 接口类型 | 稳定性 | 变更流程 |
|------|---------|--------|---------|
| L1 | Agent 应用 API（SDK + syscall） | 极稳定（类 Linux syscall） | RFC + 6 个月宽限期 + 弃用声明 |
| L2 | 智能体运行时接口（AgentsIPC 协议） | 中等稳定（版本化演进） | 季度评审 + 兼容性测试 |
| L3 | 内核子系统接口（内核内部 API） | 可重构（"you broke it, you fix it"） | 补丁序列修复所有调用点 |
| L4 | 内部实现细节 | 完全自由 | 无约束 |

### 6.3 其他核心工程思想

- **策略与机制分离**（K-4 可插拔策略）
- **渐进式开发与持续集成**（S-4 涌现性管理）
- **稳定性优先：regression 不可接受**（E-6 错误可追溯）
- **审查优先文化**（Reviewer's Statement of Oversight 等价物）
- **可追溯性**（Fixes/Closes/Link/DCO 标签体系）
- **文档即代码**（E-7）
- **不破坏用户空间原则**

---

## 7. 开发流程主题（详见 05-development-process.md）

### 7.1 补丁生命周期

```
Design → Early review → Wider review → Mainline → Stable release → Long-term maintenance
```

每个阶段都有明确的输入、输出、责任人和 SLA。

### 7.2 agentrt-liunx 适配

- **SCM**：git + GitHub PR（替代邮件 + git send-email）
- **审查工具**：GitHub PR review + CODEOWNERS + 必填 PR 模板
- **DCO 验证**：DCO bot 自动验证 Signed-off-by 链条
- **标签体系**：保留 `Fixes:`/`Closes:`/`Link:`/`Reviewed-by:`/`Acked-by:`/`Tested-by:` 等（PR 评论形式）
- **预览集成分支**：`develop` 分支等价 linux-next 树
- **稳定版分支**：`release/0.1.x`、`release/1.0.x` 等价 -stable 树

---

## 8. OS 工程规则编号体系

agentrt-liunx 在 agentrt 17 类规则编号体系基础上，新增 OS 专属编号前缀：

| 编号前缀 | 含义 | 与 agentrt 的关系 |
|---------|------|------------------|
| **OS-IRON** | agentrt-liunx 工程铁律（不可妥协） | 继承 agentrt IRON-1~10 + 新增 OS-IRON-1~N |
| **OS-BAN** | agentrt-liunx 禁止规则 | 继承 agentrt BAN-1~356 + 新增 OS-BAN-1~N |
| **OS-STD** | agentrt-liunx 标准规则 | 继承 agentrt STD + OS 专属扩展 |
| **OS-ACC** | agentrt-liunx 验收标准 | 继承 agentrt ACC-1~149 + OS 专属验收 |
| **OS-KER** | agentrt-liunx 内核工程规则 | 全新（agentrt 不涉及内核态） |
| **OS-DRV** | agentrt-liunx 驱动工程规则 | 全新 |
| **OS-BUILD** | agentrt-liunx 构建工程规则 | 全新 |
| **OS-TEST** | agentrt-liunx 测试工程规则 | 全新 |
| **OS-OBS** | agentrt-liunx 可观测性规则 | 全新 |
| **OS-SEC** | agentrt-liunx 安全工程规则 | 全新 |
| **OS-ABI** | agentrt-liunx ABI 稳定性规则 | 全新 |

具体规则编号在各个子文档中定义，汇总于 `07-maintainers-and-governance.md` 的"规则编号注册表"。

---

## 9. 工程标准实施路线

### 9.1 0.1.1 版本（奠基）

- 完成 50-engineering-standards/ 全部 9 文档（含 08-compliance-checklist.md）
- 完成 130-roadmap/ 全部 7 文档
- 完成其他 P0 模块（60-120）的 README.md 占位
- 不要求实施（实施在 1.0.1 版本）

### 9.2 1.0.1 版本（开发）

- 完成 19 模块全部 ~140 文档
- 工程标准全面实施
- 7 层自动化验证全部就位
- 维护者制度与治理落地

### 9.3 实施优先级

| 优先级 | 模块 | 文档数 | 0.1.1 范围 |
|--------|------|--------|-----------|
| P0 | 50-engineering-standards/ | 8 | 全部完成 |
| P0 | 130-roadmap/ | 7 | 全部完成 |
| P0 | 60-driver-model/ | 7 | README + 01 + 02 |
| P0 | 70-build-system/ | 8 | README + 01 + 02 |
| P0 | 80-testing/ | 10 | README + 01 + 02 |
| P0 | 90-observability/ | 9 | README + 01 + 02 |
| P0 | 100-operations/ | 10 | README + 01 + 02 |
| P0 | 110-security/ | 9 | README + 01 + 02 |
| P0 | 120-development-process/ | 9 | README + 01 + 02 |
| P1 | 140-170/ | 33 | 仅 README |
| P2 | 180-190/ | 12 | 仅 README |

**0.1.1 版本总产出**：~64 文档（工程标准 9 + 根 README 1 + 00-requirements 4 + 10-architecture 6 + 20-modules 9 + 30-interfaces 5 + 40-dataflows 4 + P0 模块 21 + P1 模块 4 + P2 模块 2 ≈ 64）。

---

## 10. 相关文档

### 10.1 同源 Airymax 文档

- `docs/ARCHITECTURAL_PRINCIPLES.md`（五维正交 24 原则）
- `docs/AirymaxAgentOS/50-engineering-standards/10-coding-style/`（编码规范子目录，含 C/Rust 风格与安全编码 4 文档 + README）
- `docs/AirymaxAgentOS/TERMINOLOGY.md`（术语规范，已提升至顶层）
- IRON-9 v2 工程铁律（闭源内部参考，17 类规则编号体系，v28.0）
- 内部工程改进方案（闭源，v4.0）

### 10.2 agentrt-liunx 设计文档

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

---

## 11. 文档版本与维护

- **当前版本**: v1.1（2026-07-07，新增 [SC] sched.h + ipc.h 头文件清单）
- **维护者**: agentrt-liunx 工程标准委员会（待成立，详见 07-maintainers-and-governance.md）
- **变更流程**: 任何工程标准变更必须经过 RFC → 评审 → ACC 验收流程
- **回顾周期**: 季度回顾 + 年度大版本

---

> **文档结束** | 共 8 文档 | 0.1.1 版本 P0 优先完成
