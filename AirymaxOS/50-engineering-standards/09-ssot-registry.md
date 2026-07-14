Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# AirymaxOS 全局规则编号注册表（SSoT）

> **文档定位**：AirymaxOS（agentrt-linux）全部编号、规则、命名的**唯一权威来源（Single Source of Truth）**。任何文档不得私自定义规则编号；所有规则编号必须在此注册表中登记后方可使用。
>
> **版本**：0.1.1\
> **最后更新**：2026-07-14\
> **权威性声明**：本注册表是 Airymax 全部规则编号的 SSoT。当任何文档与本注册表冲突时，以本注册表为准。各主题文档可包含规则的详细说明与代码示例，但必须引用本注册表中的编号——**禁止在主题文档中私自定义新编号**。新增规则编号必须通过 RFC 流程在本注册表登记。
>
> **历史**：本注册表于 2026-07-12 创建，替代此前分散在 00-engineering-standards-handbook.md §3 与 07-maintainers-and-governance.md §10 中的碎片化注册表。创建原因：历史演进中各主题文档独立编号累积，导致同一编号在不同文档指代不同规则（42 个 OS-KER 编号存在冲突、OS-ACC-001~005 在不同文件指代完全不同的验收项），必须建立单一权威注册表根除此问题。

---

## 目录

- [第 1 章 注册表使用规则](#第-1-章-注册表使用规则)
- [第 2 章 OS-IRON 工程铁律](#第-2-章-os-iron-工程铁律)
- [第 3 章 OS-KER 内核工程规则](#第-3-章-os-ker-内核工程规则)
- [第 4 章 OS-STD 标准规则](#第-4-章-os-std-标准规则)
- [第 5 章 OS-BAN 禁止规则](#第-5-章-os-ban-禁止规则)
- [第 6 章 OS-ACC 验收标准](#第-6-章-os-acc-验收标准)
- [第 7 章 OS-ABI 接口稳定性](#第-7-章-os-abi-接口稳定性)
- [第 8 章 命名规范](#第-8-章-命名规范)
- [第 9 章 OS-SEC 安全工程规则](#第-9-章-os-sec-安全工程规则)

---

## 第 1 章 注册表使用规则

### 1.1 核心原则

**每个技术点只能有一个权威编号。** 本注册表是全部规则编号的唯一分配者；主题文档是规则详细说明的载体，但编号的分配权仅属于本注册表。

### 1.2 使用规则

| 规则 | 说明 |
|------|------|
| **编号分配权** | 所有 OS-\* 编号的分配权仅属于本注册表。主题文档不得自行创建新编号。 |
| **引用义务** | 主题文档使用规则编号时，必须引用本注册表中已登记的编号。若该编号不存在，须先通过 RFC 流程在本注册表登记。 |
| **详细说明** | 主题文档可包含规则的详细说明、代码示例、checkpatch 映射等，但编号须与本注册表一致。 |
| **冲突解决** | 当任何文档与本注册表冲突时，以本注册表为准。 |
| **语义去重** | 同一技术点在多个文档中出现时，应使用同一个编号（交叉引用），而非各自定义新编号。 |

### 1.3 编号格式

```
OS-<前缀>-<子域>-<NNN>
```

| 前缀 | 子域 | 说明 |
|------|------|------|
| IRON | — | 工程铁律（不可妥协） |
| KER | — | 内核工程规则 |
| STD | CODE | 编码规范 |
| STD | FMT | 格式规范 |
| STD | STY | 风格规范 |
| STD | GOV | 治理规范 |
| STD | TOOL | 工具链与自动化规范（7 层验证体系） |
| STD | PROD | 开发流程规范（补丁生命周期/审查响应） |
| BAN | — | 禁止规则 |
| ACC | — | 验收标准 |
| ABI | — | 接口稳定性 |

### 1.4 新增编号流程

1. 在本注册表对应章节追加新条目（编号、规则简述、权威定义文档）
2. 在主题文档中引用该编号并展开详细说明
3. 通过 PR 审查确认编号唯一性与语义无重复

---

## 第 2 章 OS-IRON 工程铁律

> **权威定义文档**：04-engineering-philosophy.md

| 编号 | 规则 | 权威定义 | 五维映射 |
|------|------|---------|---------|
| OS-IRON-001 | 用户空间 ABI 永不破坏 | 04 §2.1 | K-2 / E-7 |
| OS-IRON-002 | 内核内部 API 不保证稳定；改动者必须修复所有调用点 | 04 §2.2 | K-1 / C-2 |
| OS-IRON-003 | 策略与机制分离 | 04 §3 | K-4 |
| OS-IRON-004 | 渐进式开发，补丁自包含 | 04 §4 / 05 开发流程 | S-4 / C-2 |
| OS-IRON-005 | 审查优先文化 | 04 §6 / 07 §6 | S-3 / S-1 |
| OS-IRON-006 | 不破坏用户空间（regression 零容忍） | 04 §9 / 05 开发流程 | E-6 |
| OS-IRON-007 | DCO 强制 | 07 §5 | E-6 |
| OS-IRON-008 | [SC] 共享契约层双向 CI 验证 | 120 跨项目代码共享 | E-6 / E-8 |
| OS-IRON-009 | 代码共享边界：仅 agentrt ↔ AirymaxOS | 120 / 07 §1.4 | — |
| OS-IRON-010 | "Linux 6.6 为基、seL4 为鉴"工程取向 | 04 §12 | — |
| OS-IRON-011 | 双源边界声明（01Reference/ 仅本地参考） | 04 §12 | — |
| OS-IRON-012 | seL4 借鉴仅限架构层（ES-SEL4-1~5） | 04 §12 | — |
| OS-IRON-013 | 8 子仓独立 git 仓库 + submodule | 04 §13 | S-2 |
| OS-IRON-014 | [SC] 共享契约层 6 头文件单一数据源 | 120 跨项目代码共享 | E-7 |

### 2.1 ES-SEL4 编号范围声明

> **声明目的**：开源文档 `10-architecture/03-microkernel-strategy.md` 在引用 seL4 工程思想时使用了 ES-SEL4-21~43 等扩展编号，但 SSoT 注册表此前的 ES-SEL4 登记仅覆盖 ES-SEL4-1~5（经 OS-IRON-012 引用）。为消除编号范围歧义，特此声明 ES-SEL4 编号的三段作用域。

| 编号段 | 作用域 | 登记状态 | 权威定义 |
|--------|--------|---------|---------|
| ES-SEL4-1~5 | 架构层强制借鉴 | 已注册（OS-IRON-012 引用） | 04-engineering-philosophy.md §12 / 闭源总纲 §12.6 |
| ES-SEL4-6~10 | P2 增强建议 | 已注册（闭源总纲附录 D） | 闭源总纲附录 D |
| ES-SEL4-11~43 | 研究范围编号 | 未注册为工程标准 | `_research_0.2.5/01-sel4-deep-analysis.md`（研究范围，非工程标准范围） |

> **使用约定**：ES-SEL4-11~43 属于 seL4 源码深度研读的研究范围编号（研究文件共提炼 54 条 ES-SEL4 工程思想），非工程标准范围。在开源文档中引用 ES-SEL4-11~43 时，必须标注为"研究范围参考"，不得作为强制工程规则引用。

---

## 第 3 章 OS-KER 内核工程规则

> **编号区间说明**：001-010 由 01-coding-standards.md 权威定义；011-021 由 C_Cpp_coding_style.md Part I 权威定义；022-029 由 02-kconfig-system.md 权威定义；046 由 C_Cpp_coding_style.md Part I 权威定义；053-056 由 01-coding-standards.md 权威定义；060-069 由 04-engineering-philosophy.md 权威定义；071-085 由各主题文档权威定义；086-155 为历史冲突消解后的唯一编号（2026-07-12 全量消解）。（2026-07-13 更新：C_coding_style_standard.md 已合并入 C_Cpp_coding_style.md Part I）

### 3.1 01-coding-standards.md 权威定义（001-010, 053, 056）

| 编号 | 规则 | 章节 |
|------|------|------|
| OS-KER-001 | goto 集中出口模式 | 01 §7.1 |
| OS-KER-002 | 禁止 #if 桩函数模式（由 IRON-1 涵盖） | SSoT 内部 |
| OS-KER-003 | 条件编译必须用 IS_ENABLED() | SSoT 内部 |
| OS-KER-004 | 分级标签按分配逆序释放 | 01 §7.2 |
| OS-KER-005 | [已废弃] 原定义为"C 内核代码 Tab 8 字符缩进"，因格式规则重新组织（内容迁移至 OS-STD-FMT-001 / 闭源 OS-FMT-001）而废弃 | — |
| OS-KER-006 | volatile 仅 4 种例外 | 01 §6.3 |
| OS-KER-007 | [已废弃] 原定义为"内核态禁 float，强制 q16.16 定点"，于 2026-07-12 迁移至 OS-STD-010（与 01-coding-standards.md Part II §0.2.1 迁移映射表一致） | — |
| OS-KER-008 | [已废弃] 原定义为"函数例外：开括号在下一行行首（K&R）"，于 2026-07-12 与 OS-STD-047 合并迁移至 OS-STD-FMT-009（与 01-coding-standards.md Part II §0.2.1 迁移映射表一致） | — |
| OS-KER-009 | 忙等循环必须调用 cpu_relax() | 01 §8.1 |
| OS-KER-010 | 受保护数据结构须声明 magic 字段并注册 | 01 §8.2 |
| OS-KER-053 | 共享数据必须用同步原语保护 | 01 §6.4 |
| OS-KER-056 | I/O 内存必须通过 accessor 访问 | 01 §6.5 |

### 3.2 C_Cpp_coding_style.md Part I 权威定义（011-021, 046, 070）

> **合并说明（2026-07-13）**：原 `C_coding_style_standard.md` 已合并入 `C_Cpp_coding_style.md` Part I。下表章节引用由 `C_coding_style §X` 更新为 `C_Cpp_coding_style Part I §X`。

| 编号 | 规则 | 章节 |
|------|------|------|
| OS-KER-015 | 函数一屏可读（≤80×24），局部变量 ≤5-10 | C_Cpp_coding_style Part I §3 |
| OS-KER-016 | GFP 选型：GFP_KERNEL / GFP_ATOMIC / GFP_NOWAIT | C_Cpp_coding_style Part I §4 |
| OS-KER-017 | 释放内存后指针置 NULL | C_Cpp_coding_style Part I §5 |
| OS-KER-018 | krealloc 用临时变量接收 | C_Cpp_coding_style Part I §6 |
| OS-KER-019 | 锁选型：mutex / spinlock / rwlock / RCU | C_Cpp_coding_style Part I §7 |
| OS-KER-020 | RCU 读写端规则 | C_Cpp_coding_style Part I §8 |
| OS-KER-021 | [SC] 共享契约层代码约束 | C_Cpp_coding_style Part I §9 |
| OS-KER-046 | 锁设计在数据结构设计阶段完成 | C_Cpp_coding_style Part I §10 |
| OS-KER-070 | [SS] 语义同源层 API 签名约束 | C_Cpp_coding_style Part I §11 |

> **注**：历史 OS-KER-011~014（Tab-8 / 行宽 80 / K&R 大括号 / 空格规则）已于 2026-07-12 迁移至 OS-STD-FMT-001~005，不再作为 OS-KER 活跃编号。迁移映射见 01-coding-standards.md Part II §0.2.1。

### 3.3 02-kconfig-system.md 权威定义（022-029）

| 编号 | 规则 | 章节 |
|------|------|------|
| OS-KER-022 | 互斥决策必须用 `choice` | 02-kconfig §2 |
| OS-KER-023 | `select` 目标的依赖须同时满足 | 02-kconfig §3 |
| OS-KER-024 | 可模块化子系统必须用 `tristate` | 02-kconfig §4 |
| OS-KER-025 | `obj-$(CONFIG_*)` 的宏须有 config 声明 | 02-kconfig §5 |
| OS-KER-026 | 新增 Kconfig 须被父 Kconfig source | 02-kconfig §6 |
| OS-KER-027 | Kconfig.airymaxos 仅聚合跨子系统特性 | 02-kconfig §7 |
| OS-KER-028 | 新 choice/menuconfig 须用工具实际验证 | 02-kconfig §8 |
| OS-KER-029 | airymaxos-base.config 变更经评审 | 02-kconfig §9 |
| OS-KER-030 | 内核态配置必须以 Kconfig 为唯一描述手段 | 02-kconfig §1 |

### 3.4 04-engineering-philosophy.md 权威定义（060-069）

| 编号 | 规则 | 章节 |
|------|------|------|
| OS-KER-060 | API 改动者必须修复所有调用点 | 04 §2.2 |
| OS-KER-061 | 内核不内置策略（IRON-003 子规则） | 04 §3.2 |
| OS-KER-062 | 补丁序列中点可编译（IRON-004 子规则） | 04 §4.3 |
| OS-KER-063 | 逻辑变更拆分为可编译补丁序列 | 04 §4.3 |
| OS-KER-064 | 补丁顺序反映逻辑依赖 | 04 §4.3 |
| OS-KER-065 | bug 修复补丁必须附带回归测试 | 04 §5.4 |
| OS-KER-066 | bug 修复复杂度与严重性匹配 | 04 §5.4 |
| OS-KER-067 | bug 修复补丁必须含 Fixes: 标签 | 04 §7.1 |
| OS-KER-068 | 补丁必须含 Signed-off-by DCO 链 | 04 §7.4 |
| OS-KER-069 | 新 /proc 条目必须文档化 | 04 §8.1 |

### 3.5 主题文档权威定义（054-055, 071-085）

| 编号 | 规则 | 权威定义文档 |
|------|------|-------------|
| OS-KER-054 | ruleset 是策略载体，运行时不可变（Landlock） | 110-security/02-landlock-sandbox.md |
| OS-KER-055 | Agent 跟踪必须用 trace_array_create() 独立 instance | 90-observability/01-ftrace-framework.md |
| OS-KER-071 | 系统级测试与 KUnit 契约测试共享同一契约文档 | 80-testing/02-kselftest.md |
| OS-KER-072 | Landlock 网络规则为可选增强，缺失时回退默认拒绝 | 110-security/02-landlock-sandbox.md |
| OS-KER-073 | register_ftrace_function 须设 SAVE_REGS_IF_SUPPORTED | 90-observability/01-ftrace-framework.md |
| OS-KER-074 | L1 白盒单元测试必须用 KUnit | 80-testing/01-kunit-framework.md |
| OS-KER-075 | KUnit 套件命名空间必须正交 | 80-testing/01-kunit-framework.md |
| OS-KER-076 | built-in.a 用 ar cDPrST 确定顺序 | 70-build/01-kbuild-system.md |
| OS-KER-077 | 生成物规则用 if_changed 系列 | 70-build/01-kbuild-system.md |
| OS-KER-078 | scripts_basic/syncconfig 先于 descending | 70-build/01-kbuild-system.md |
| OS-KER-079 | Landlock 三 syscall 是稳定 ABI | 110-security/02-landlock-sandbox.md |
| OS-KER-080 | 每个 Agent 必须施加独立 Landlock 域 | 110-security/02-landlock-sandbox.md |
| OS-KER-081 | no_new_privs 是 Agent 启动前必设标志 | 110-security/02-landlock-sandbox.md |
| OS-KER-082 | Agent tracepoint print fmt 须以 agent_id=0x%04x 起始 | 90-observability/01-ftrace-framework.md |
| OS-KER-083 | kernel 须导出 Agent 路径函数符号至 kallsyms | 90-observability/01-ftrace-framework.md |
| OS-KER-084 | defconfig 须开 CONFIG_KALLSYMS 等 | 90-observability/01-ftrace-framework.md |
| OS-KER-085 | 禁止 KUnit 与 kselftest 混编 | 80-testing/02-kselftest.md |

### 3.6 历史冲突消解编号（086-155，2026-07-12 全量消解）

> 以下编号于 2026-07-12 全量消解时分配，替代各主题文档中私自重用的 001-052 编号。消解原则：每个技术点获得全局唯一编号，杜绝跨文档冲突。

#### 3.6.1 110-security/01-lsm-framework.md（086-089）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-086 | security_hook_heads 由 __ro_after_init 保护 | OS-KER-001 |
| OS-KER-087 | exclusive LSM 互斥语义不可绕过 | OS-KER-002 |
| OS-KER-088 | CONFIG_LSM 顺序编译期固化，运行时不可追加 | OS-KER-003 |
| OS-KER-089 | MicroCoreRT 锁定 Cupolas 钩子白名单 | OS-KER-004 |

#### 3.6.2 70-build/01-kbuild-system.md（090-093）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-090 | 顶层 Kbuild descending 顺序与基线一致 | OS-KER-001 |
| OS-KER-091 | prepare 阶段先于 descending | OS-KER-002 |
| OS-KER-092 | obj-* 用 obj-$(CONFIG_*) 门控 | OS-KER-003 |
| OS-KER-093 | 子系统 Makefile 含 SPDX 标识 | OS-KER-004 |

#### 3.6.3 80-testing/01-kunit-framework.md（094-097）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-094 | 测试函数禁止直接访问 struct kunit 私有字段 | OS-KER-001 |
| OS-KER-095 | 扩展套件以 airy_* 前缀命名 | OS-KER-002 |
| OS-KER-096 | 禁止默认开启 CONFIG_KUNIT_EXAMPLE_TEST | OS-KER-003 |
| OS-KER-097 | 测试文件与被测源码同目录同名加 _test 后缀 | OS-KER-004 |

#### 3.6.4 80-testing/02-kselftest.md（098-100）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-098 | 子目录禁止引入对上游测试集源码的依赖 | OS-KER-010 |
| OS-KER-099 | L2 系统级用例必须用 kselftest 实现 | OS-KER-011 |
| OS-KER-100 | kselftest 子目录命名空间必须正交 | OS-KER-012 |

#### 3.6.5 90-observability/01-ftrace-framework.md（101-105）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-101 | defconfig 必须开启 CONFIG_FUNCTION_TRACER 等 | OS-KER-001 |
| OS-KER-102 | init 阶段完成 tracefs 挂载，失败 panic | OS-KER-002 |
| OS-KER-103 | ring buffer 默认大小 1408 KB/CPU | OS-KER-003 |
| OS-KER-104 | 为 Agent 核心路径打 tracepoint | OS-KER-004 |
| OS-KER-105 | Agent 跟踪代码体积 < 8 KB | OS-KER-010 |

#### 3.6.6 90-observability/02-ebpf-probes.md（106-117）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-106 | defconfig 须开 CONFIG_BPF 等 5 项 | OS-KER-011 |
| OS-KER-107 | 须启用 CONFIG_BPF_JIT_ALWAYS_ON | OS-KER-012 |
| OS-KER-108 | 为 Agent 核心结构导出 BTF | OS-KER-013 |
| OS-KER-109 | 注册 bpf_agent_decision_get 等 kfunc | OS-KER-014 |
| OS-KER-110 | 须开 CONFIG_BPF_VERIFIER_STATE_MARK | OS-KER-015 |
| OS-KER-111 | Agent 核心路径函数支持 kprobe | OS-KER-016 |
| OS-KER-112 | 须支持 bpf(BPF_LINK_CREATE) | OS-KER-017 |
| OS-KER-113 | /sys/kernel/agentrt/token_usage 导出 per-Agent Token | OS-KER-018 |
| OS-KER-114 | 四个函数符号导出至 kallsyms | OS-KER-019 |
| OS-KER-115 | BPF 程序 .o 体积 < 64 KB，指令数 < 1 万 | OS-KER-020 |
| OS-KER-116 | register_bpf_struct_ops 注册 sched_agent_ops | OS-KER-021 |
| OS-KER-117 | SCHED_AGENT fallback 机制 | OS-KER-022 |

#### 3.6.7 50-engineering-standards/01-coding-standards.md Part III（118-148）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-118 | 禁止"未来扩展"式抽象 | OS-KER-022 |
| OS-KER-119 | 不维护跨平台 HAL（E-4 原则） | OS-KER-023 |
| OS-KER-120 | 内核只提供能力，策略外移 | OS-KER-024 |
| OS-KER-121 | 策略可在不重编译内核下替换 | OS-KER-025 |
| OS-KER-122 | SCHED_AGENT 基于 sched_ext | OS-KER-026 |
| OS-KER-123 | 函数长度一屏可读 | OS-KER-027 |
| OS-KER-124 | 相信编译器，不写 __always_inline | OS-KER-028 |
| OS-KER-125 | 禁止 .c 内 #ifdef 切割函数体 | OS-KER-029 |
| OS-KER-126 | IS_ENABLED 优先于 #ifdef | OS-KER-030 |
| OS-KER-127 | 引用计数强制 | OS-KER-031 |
| OS-KER-128 | 锁与引用计数职责不同 | OS-KER-032 |
| OS-KER-129 | C99 灵活数组成员 | OS-KER-033 |
| OS-KER-130 | bool 合并为 bitfield | OS-KER-034 |
| OS-KER-131 | 内核内部 API 不保证稳定 | OS-KER-035 |
| OS-KER-132 | 用户空间 ABI 永久稳定 | OS-KER-036 |
| OS-KER-133 | 抽象有成本 | OS-KER-037 |
| OS-KER-134 | 错误路径必须有测试 | OS-KER-038 |
| OS-KER-135 | 用户可控长度参数须溢出检查 | OS-KER-039 |
| OS-KER-136 | 错误路径分级标签 | OS-KER-040 |
| OS-KER-137 | WARN_ON_ONCE 禁止 WARN_ON | OS-KER-041 |
| OS-KER-138 | >3 行函数不应 inline | OS-KER-042 |
| OS-KER-139 | 数据结构 cache line 对齐 | OS-KER-043 |
| OS-KER-140 | sizeof(*p) 强制 | OS-KER-044 |
| OS-KER-141 | 禁止单独 printk，用分级宏 | OS-KER-045 |
| OS-KER-142 | 锁设计在数据结构阶段完成 | OS-KER-046 |
| OS-KER-143 | 热路径必须用 C 实现 | OS-KER-047 |
| OS-KER-144 | Rust 仅用于安全敏感非热路径 | OS-KER-048 |
| OS-KER-145 | -Wall -Wextra -Werror | OS-KER-049 |
| OS-KER-146 | make C=1 sparse 检查 | OS-KER-050 |
| OS-KER-147 | CONFIG_LOCKDEP 强制 | OS-KER-051 |
| OS-KER-148 | fault injection 框架 | OS-KER-052 |

#### 3.6.8 60-driver-model（149-151）

| 编号 | 规则 | 权威定义文档 | 原编号 |
|------|------|-------------|--------|
| OS-KER-149 | device/driver/bus 模型不得引入用户态 daemon 强依赖 | 60-driver-model/01-device-model.md | OS-KER-030 |
| OS-KER-150 | platform driver remove 回调通知用户态 daemon | 60-driver-model/02-platform-driver.md | OS-KER-040 |
| OS-KER-151 | probe 中 printk 级别限制 | 60-driver-model/02-platform-driver.md | OS-KER-041 |

#### 3.6.9 70-build/02-kconfig-system.md（155）

| 编号 | 规则 | 原编号 |
|------|------|--------|
| OS-KER-155 | config 名字必须以 AIRY_ 或子系统前缀开头 | OS-KER-021 |

### 3.7 120-development-process 独立编号（211, 221）

| 编号 | 规则 | 权威定义文档 |
|------|------|-------------|
| OS-KER-211 | kernel 子仓 ABI 改动须知会 system 子仓维护者 | 120-development-process/02-maintainer-hierarchy.md |
| OS-KER-221 | AgentsIPC/MicroCoreRT 补丁须顶级维护者 Reviewed-by | 120-development-process/02-maintainer-hierarchy.md |

---

## 第 4 章 OS-STD 标准规则

### 4.1 OS-STD-CODE 编码规范

> **权威定义文档**：01-coding-standards.md

| 编号 | 规则 |
|------|------|
| OS-STD-CODE-001 | 全局函数/变量描述性命名，禁止无意义缩写 |
| OS-STD-CODE-002 | goto 集中出口，分级标签按分配逆序释放 |
| OS-STD-CODE-003 | 禁止 strcpy/strncpy/strlcpy，强制 strscpy |
| OS-STD-CODE-004 | 编译期必须 0 警告 0 错误 |
| OS-STD-CODE-005 | 代码重复 3 处以上应抽出到 lib/ 或 commons/ 子模块 |
| OS-STD-CODE-006 | agentrt-linux 4 层接口稳定性分级（内核内部 / 用户 ABI / [SC] / [SS]） |
| OS-STD-CODE-007 | 跨语言 FFI 边界必须显式声明（extern "C" / ctypes / N-API） |

### 4.2 OS-STD-FMT 格式规范

> **权威定义文档**：01-coding-standards.md Part II（29 条，详见 Part II §0.2.1 迁移映射表）

完整 29 条清单见 [01-coding-standards.md Part II §0.2.1](./01-coding-standards.md)。此处不重复，以 01-coding-standards.md Part II 为唯一权威来源。

| 编号 | 规则 |
|------|------|
| OS-STD-FMT-030 | 配置即代码——每条格式规则必须有对应格式化工具配置项 |

### 4.3 OS-STD-STY 风格规范

> **权威定义文档**：01-coding-standards.md Part III §8.1-§8.5（OS-STD-STY-001~004 定义于 §8.1-§8.4；OS-STD-STY-005~012 品味哲学定义于 §8.5，源自 OLK-6.6 源码深度提炼）

| 编号 | 规则 |
|------|------|
| OS-STD-STY-001 | 反过度抽象，不预先抽象 |
| OS-STD-STY-002 | 引用计数强制，锁不替代引用计数 |
| OS-STD-STY-003 | agentrt-linux 分层职责：热路径 C / 慢路径 Rust / Agent Python+TS |
| OS-STD-STY-004 | Agent 工作负载考量——Token 能效与记忆卷载（L1-L4 分层卷载） |
| OS-STD-STY-005 | 信任读者——暴露而非封装（OLK-6.6 `mutex.c` L60-73 owner 位编码） |
| OS-STD-STY-006 | 短函数——一屏可读（≤80×24），强制逻辑分解（OLK-6.6 `mutex.c` L845） |
| OS-STD-STY-007 | 简洁文件头——SPDX 优先，禁止冗长版权声明块（OLK-6.6 `mutex.c` L1-20） |
| OS-STD-STY-008 | EXPORT_SYMBOL 紧跟定义——导出即声明（OLK-6.6 `mutex.c` L46-58） |
| OS-STD-STY-009 | include 语义分组——禁止字母序自动排序，checkpatch 不检查字母序（OLK-6.6 `mutex.c` L21-35） |
| OS-STD-STY-010 | `__` 前缀标注内部接口——C 无访问控制下的约定（OLK-6.6 `mutex.c` L80 `__mutex_owner`） |
| OS-STD-STY-011 | `__read_mostly` 缓存优化——mostly-read 数据分离段（OLK-6.6 `core.c` L168/178/7786） |
| OS-STD-STY-012 | `__randomize_layout` 结构体布局随机化——安全敏感结构体加固（OLK-6.6 `sched.h` L658） |

### 4.4 OS-STD-GOV 治理规范

> **权威定义文档**：07-maintainers-and-governance.md

| 编号 | 规则 |
|------|------|
| OS-STD-GOV-001 | MAINTAINERS 文件为子系统归属单一事实源 |
| OS-STD-GOV-002 | 每子仓须有主维护者与至少 1 名审查者 |
| OS-STD-GOV-003 | 状态字段须如实标注 |
| OS-STD-GOV-004 | Co-developed-by 必须紧跟对应 Signed-off-by |
| OS-STD-GOV-005 | 审查意见必须逐条响应 |
| OS-STD-GOV-006 | 不响应审查是致命错误 |
| OS-STD-GOV-007 | Andrew Morton 规则：未导致改动的审查意见转化为代码注释 |
| OS-STD-GOV-008 | 信任链分层不可越级提交 |
| OS-STD-GOV-009 | 组织 6 级成熟度 Level 2 为底线目标 |

### 4.5 OS-STD-RUST Rust 编码规范

> **权威定义文档**：`10-coding-style/Rust_coding_style.md`（Part I 风格 + Part II 安全编码 + Part III 驱动开发）

| 编号 | 规则 | 所在 Part |
|------|------|-----------|
| OS-STD-030 | 所有权模型——每个资源唯一所有者，move 语义 | Part I §2.1 |
| OS-STD-031 | 借用而非克隆——优先 `&T`/`&mut T` | Part I §2.2 |
| OS-STD-032 | 生命周期显式标注 | Part I §2.3 |
| OS-STD-033 | snake_case 命名约定 | Part I §3.1 |
| OS-STD-034 | `airy_` 前缀隔离 | Part I §3.2 |
| OS-STD-035 | 模块命名与文件名一致 | Part I §3.3 |
| OS-STD-036 | 禁止 panic/unwrap/expect | Part I §5.1 |
| OS-STD-037 | `?` 运算符错误传播 | Part I §5.2 |
| OS-STD-038 | `Option<T>` 代替哨兵值 | Part I §5.3 |
| OS-STD-039 | 使用 kernel crate 安全抽象 | Part I §6.1 |
| OS-STD-040 | KBox/KVec 内存分配 | Part I §6.2 |
| OS-STD-041 | kernel::sync 同步原语 | Part I §6.3 |
| OS-STD-042 | Pin<T> 不可移动保证 | Part I §7.1 |
| OS-STD-043 | pin_init! 宏初始化 | Part I §7.2 |
| OS-STD-044 | extern "C" FFI 边界 | Part I §8.1 |
| OS-STD-045 | FFI 类型映射——`core::ffi`/`kernel::ffi` C 兼容类型 | Part I §8.2 |
| OS-STD-046 | FFI 所有权语义——`#[ownership]` 标注 | Part I §8.3 |
| OS-STD-050 | 驱动注册模型（kernel::module! 宏） | Part III §1.1 |
| OS-STD-051 | 驱动 Trait 约定 | Part III §1.2 |
| OS-STD-052 | 驱动 unsafe 审计标准（SAFETY 注释） | Part III §2.1 |
| OS-STD-053 | 硬件 MMIO 寄存器安全访问 | Part III §2.2 |
| OS-STD-054 | DMA 内存安全（DmaObject） | Part III §2.3 |
| OS-STD-055 | Kbuild Rust 集成 | Part III §3.1 |
| OS-STD-056 | Kconfig 配置（CONFIG_AIRY_*_RUST） | Part III §3.2 |
| OS-STD-057 | KUnit Rust 驱动测试 | Part III §4.1 |

### 4.6 OS-STD-TOOL 工具链规则编号段声明

> **权威定义文档**：`05-development-process.md` Part II（§0.2 声明历史 `OS-STD-101~158` 迁移至 `OS-STD-TOOL-101~158`，本注册表登记编号段归属）

`OS-STD-TOOL-NNN`（TOOL = Toolchain）编号段承载 7 层自动化验证体系的工具链规则（编译期/静态分析/预提交/CI 门禁/连接验证/协议验证/发布验证）。本编号段与 `OS-STD-PROD-NNN`（PROD = Process/Development，权威定义于 `05-development-process.md`）隔离，消除历史上 `OS-STD-101~158` 在两卷中的语义冲突。

**编号段归属**：

| 编号段 | 权威定义文档 | 内容范围 | 7 层验证对应 |
|--------|------------|---------|-------------|
| OS-STD-TOOL-001~099 | 05-development-process.md Part II | 通用工具链规则（编译/格式/检查） | 第 1-3 层 |
| OS-STD-TOOL-101~158 | 05-development-process.md Part II | CI/CD 矩阵 + 覆盖率门槛 + 发布流水线 | 第 4-7 层 |
| OS-STD-PROD-001~099 | 05-development-process.md | 开发流程规则（补丁生命周期/审查响应） | 流程层 |
| OS-STD-PROD-101~158 | 05-development-process.md | 开发流程高级规则（RFC/早期审查/维护者响应） | 流程层 |

**关键规则登记**：

| 编号 | 规则 | 权威定义 |
|------|------|---------|
| OS-STD-TOOL-081 | sparse `make C=2` 检查 + W 分级体系（W=1 本地 / W=2 CI） | 06 §3.1 + §3.1.1 |
| OS-STD-TOOL-081-CI | CI 强制 `make W=2 C=2` 检查新增代码 | 06 §3.1.1 |
| OS-STD-TOOL-081-REL | 发布前 `make W=2 C=2` 全量审计 | 06 §3.1.1 |
| OS-STD-TOOL-121 | 内核子系统行覆盖率 ≥80% | 06 §7.3 |
| OS-STD-TOOL-122 | Agent SDK 行覆盖率 ≥80% | 06 §7.3 |
| OS-STD-TOOL-124 | 关键路径覆盖率 100% | 06 §7.4 |
| OS-STD-TOOL-160 | checkpatch.pl --strict 每次提交前必须通过 | 01-coding-standards §9.5 |
| OS-STD-TOOL-161 | 多配置构建（defconfig + allnoconfig + allmodconfig 三者皆绿） | 01-coding-standards §9.6 |

> **历史编号等价声明**：05-development-process.md Part II 正文中现存的历史编号 `OS-STD-002~158` 与目标 `OS-STD-TOOL-002~158` 等价有效，规则效力以 Part II 正文为准。CI/CD 脚本与 GitHub Actions workflow 实施时统一替换为 `OS-STD-TOOL-NNN`。

### 4.6.1 OS-STD-PROD 开发流程规则登记

> **权威定义文档**：`05-development-process.md`
>
> **迁移说明（2026-07-14）**：原 `05-development-process.md` 中的裸 `OS-STD-2xx` 编号（与 `01-coding-standards.md` 的 `OS-STD-2xx` 冲突）已全部迁移至 `OS-STD-PROD-NNN` 子域编号，消除编号三重冲突。

| 编号 | 规则 | 权威定义 |
|------|------|---------|
| OS-STD-PROD-001 | LTS 版本必须每季度发布一个维护版本 | 05 §7.3 |
| OS-STD-PROD-002 | LTS 维护者离职需提前 6 个月通知 | 05 §7.3 |
| OS-STD-PROD-011 | 从 L1 晋升 L2 需至少 5 个被合并的 PR，且无 regression | 05 §8.3 |
| OS-STD-PROD-012 | 从 L2 晋升 L3 需至少 20 个被合并的 PR + 维护者面试 | 05 §8.3 |
| OS-STD-PROD-021 | 每个子仓必须维护一份 MAINTAINERS.md | 05 §9.3 |
| OS-STD-PROD-031 | 所有 commit 必须用 `git commit -s` 添加 DCO 签名 | 05 §10.1 |
| OS-STD-PROD-032 | 禁止 `git push --force` 到 `main`/`develop`/`release/*` 分支 | 05 §10.1 |
| OS-STD-PROD-033 | 所有 PR 必须通过 GitHub Actions 全部检查才能合并 | 05 §10.2 |
| OS-STD-PROD-034 | PR 状态变更须同步到 GitHub Projects 看板 | 05 §10.3 |

### 4.6 族索引登记（已废除）

> **废除说明**（2026-07-12 SSoT 重新设计）：原 9 个族索引文件（200/201 根目录 + 200-206 子目录）已物理删除。族索引模式被证明是"文档越来越多"的根源之一——9 个文件 ~1,260 行仅是导航链接堆叠，无独立规则定义价值。所有规则编号登记统一由本注册表 §2-§7 承载，不再通过族索引中间层导航。

### 4.7 子目录 README SSoT 依赖登记

> 以下 5 个子目录 README 已在文件头部追加 SSoT 依赖声明，声明其规则编号权威为 09-ssot-registry.md §3。子目录 README 本身不定义新规则编号，仅作为子目录内文件的索引入口。（族索引已废除，2026-07-12 SSoT 重新设计）

| 子目录 README | SSoT 依赖声明内容 | 特定权威引用 | 登记日期 |
|--------------|------------------|-------------|---------|
| `10-coding-style/README.md` | 编号权威 09 §3 | coding_conventions.md Part IV §4.4（模块前缀） | 2026-07-12 |
| `20-contracts/README.md` | 编号权威 09 §3 | contracts.md Part III（日志 SSoT） | 2026-07-12 |
| `30-runtime-interfaces/README.md` | 编号权威 09 §3 | coding_conventions.md Part IV §1.10（`are_*` 前缀） | 2026-07-12 |
| `40-integration/README.md` | 编号权威 09 §3 | IRON-9 v2 三层模型（[SC]/[SS]/[IND]） | 2026-07-12 |
| `50-project-erp/README.md` | 编号权威 09 §3 | IRON-9 v2 工程铁律 | 2026-07-12 |

### 4.8 OS-FMT 闭源总纲代码格式规则

> **权威定义文档**：闭源总纲（`agentrt-linux基本工程标准规范.md`）§3 代码格式
>
> **来源说明**：OS-FMT 为闭源总纲定义的格式子域前缀（FMT = Format），承载 C 内核代码格式规则。本节将其登记到 SSoT 注册表以建立跨开源/闭源的编号一致性。注意：本节 OS-FMT-001~007（闭源总纲 §3）与 §4.2 OS-STD-FMT（开源 `01-coding-standards.md` Part II，29 条）为两套独立编号系列——前者源自闭源总纲，后者源自开源编码标准；二者内容有重叠但编号体系不同，引用时须明确区分。

| 编号 | 规则 | 章节定位 |
|------|------|---------|
| OS-FMT-001 | C 内核代码使用 Tab 缩进，Tab 宽度为 8 字符 | 闭源总纲 §3.1 |
| OS-FMT-002 | C 代码首选 80 列行宽 | 闭源总纲 §3.2 |
| OS-FMT-003 | 使用 K&R 风格大括号（非函数块左括号放行末，函数定义左括号新起一行） | 闭源总纲 §3.3 |
| OS-FMT-004 | 即使只有一条语句，也使用大括号包围 | 闭源总纲 §3.3 |
| OS-FMT-005 | 空格规则（关键字后加空格 / 括号内禁止空格 / 二元运算符两侧加空格 / 一元运算符不加空格 / 指针声明 `*` 贴名字） | 闭源总纲 §3.4 |
| OS-FMT-006 | 行尾禁止空白字符 | 闭源总纲 §3.5 |
| OS-FMT-007 | 所有 C/C++ 代码必须通过 `.clang-format` 配置自动格式化 | 闭源总纲 §3.6 |

> **历史关联**：OS-FMT-001（Tab 8 字符缩进）原属 OS-KER-005，因格式规则重新组织而迁移；OS-KER-005 已标记为已废弃（见 §3.1）。

---

## 第 5 章 OS-BAN 禁止规则

> **权威定义文档**：01-coding-standards.md

| 编号 | 规则 |
|------|------|
| OS-BAN-001 | 禁止敏感术语 master/slave、blacklist/whitelist |
| OS-BAN-002 | 禁止 BUG()/BUG_ON()，改用 WARN_ON_ONCE() |
| OS-BAN-003 | sizeof(*p) / kmalloc_array / struct_size |
| OS-BAN-004 | 禁止 strcpy/strncpy/strlcpy，强制 strscpy |
| OS-BAN-005 | 禁止 %p 默认输出（须哈希化） |
| OS-BAN-006 | 禁止匈牙利命名法 |
| OS-BAN-007 | 禁止函数声明 extern |
| OS-BAN-008 | 禁止自动排序 include |
| OS-BAN-009 | 禁止五类危险宏 |
| OS-BAN-010 | 开源文档禁词 |
| OS-BAN-011 | 禁词违规处罚 |

---

## 第 6 章 OS-ACC 验收标准

> **权威定义文档**：130-roadmap/06-acceptance-criteria.md
>
> **SSoT 对齐说明（v0.2.5 已消解）**：历史 00-handbook §3.7 与 07-governance §10 中定义的 OS-ACC-001~005 与 130-roadmap 中的 OS-ACC 编号存在语义冲突（同一编号指代不同验收项）。现已完成消解：本注册表 §7 确认 130-roadmap/06-acceptance-criteria.md 为 OS-ACC 的唯一权威定义文档；00-handbook §3.7 已改为引用本注册表 §7（见 00-handbook L151）；07-governance §10.6 已改为引用本注册表 §7（见 07-governance L645）。OS-ACC 编号体系单源权威已建立。

---

## 第 7 章 OS-ABI 接口稳定性

> **权威定义文档**：04-engineering-philosophy.md §6.2

| 编号 | 规则 |
|------|------|
| OS-ABI-001 | 4 层接口稳定性分级：L1 永久 / L2 弃用流程 / L3 版本协商 / L4 完全自由 |

---

## 第 8 章 命名规范

> **权威定义文档**：10-coding-style/coding_conventions.md Part IV §4.4

模块前缀（`are_` / `airy_` / `airy_core_` / `airy_` 及模块子前缀）的唯一权威来源为 coding_conventions.md Part IV §4.4。本注册表不重复注册模块前缀；前缀注册、变更与新增须遵循 coding_conventions.md Part IV §4.4.6 的 RFC 流程。（2026-07-13 更新：naming_conventions.md 已合并入 coding_conventions.md Part IV）

### 8.1 术语表登记

| 文件 | 内容范围 | 权威范围 | 迁移说明 |
|------|---------|---------|---------|
| `90-terminology.md` | Airymax 全项目术语表（原 `TERMINOLOGY.md`） | 术语定义 SSoT | 2026-07-12 从 AirymaxOS 根目录移入 50-engineering-standards/ |

### 8.2 废弃文档登记

| 文件 | 原内容 | 当前状态 | 重定向目标 | 废弃日期 |
|------|--------|---------|-----------|---------|
| `15-license-strategy.md` | 许可证策略（200 行） | 已物理删除（2026-07-12 SSoT 重新设计） | `05-development-process.md` Part IV | 2026-07-12 |
| `02-code-format.md` | 代码格式标准（547 行） | 已物理删除（2026-07-13 顶层文件合并） | `01-coding-standards.md` Part II | 2026-07-13 |
| `03-code-style.md` | 代码风格标准（489 行） | 已物理删除（2026-07-13 顶层文件合并） | `01-coding-standards.md` Part III | 2026-07-13 |
| `60-checkpatch-rule-map.md` | checkpatch 规则映射表（788 行） | 已物理删除（2026-07-13 顶层文件合并） | `01-coding-standards.md` Part IV | 2026-07-13 |
| `70-kernel-doc-standard.md` | kernel-doc 注释规范（552 行） | 已物理删除（2026-07-13 顶层文件合并） | `01-coding-standards.md` Part V | 2026-07-13 |
| `80-clang-format-enforcement.md` | .clang-format 配置与 CI 门禁（592 行） | 已物理删除（2026-07-13 顶层文件合并） | `01-coding-standards.md` Part VI | 2026-07-13 |
| `06-toolchain-and-automation.md` | 工具链与自动化（764 行） | 已物理删除（2026-07-13 顶层文件合并） | `05-development-process.md` Part II | 2026-07-13 |
| `08-compliance-checklist.md` | 规范符合性检查（756 行） | 已物理删除（2026-07-13 顶层文件合并） | `05-development-process.md` Part III | 2026-07-13 |
| `110-spdx-license-compliance.md` | SPDX 许可证策略与合规（492 行） | 已物理删除（2026-07-13 顶层文件合并） | `05-development-process.md` Part IV | 2026-07-13 |
| `100-deprecated-api-registry.md` | 废弃 API 动态清单（642 行） | 已物理删除（2026-07-13 废弃文档清理） | （无重定向，内容已废弃） | 2026-07-13 |

### 8.3 文档计数三作用域模型

> **登记目的**：工程文档在不同上下文中使用不同的文档计数口径，历史上有 ~79 / ~85 / ~122 三种数字并存且未在 SSoT 注册表中声明其作用域，导致计数混淆。特此登记三作用域模型以消除歧义。

| 计数 | 作用域 | 上下文 | 说明 |
|------|--------|--------|------|
| ~79 文档 | P0 基线 | "文档体系完成"上下文 | P0 级文档基线，核心工程标准与架构文档集合 |
| ~85 文档 | 工程基线 | "0.1.1 合计/总产出"上下文 | 0.1.1 版本工程产出合计，含 P0 基线 + 扩展工程文档 |
| ~122 文档 | 全量展开 | "全部三大支柱"上下文 | 全量文档展开（全部三大支柱），含研究/分析/审查等辅助文档 |

> **使用约定**：引用文档计数时必须标注所属作用域（P0 基线 / 工程基线 / 全量展开），禁止脱离作用域上下文使用裸数字。

---

## 第 9 章 OS-SEC 安全工程规则

> **权威定义文档**：10-coding-style/C_Cpp_coding_style.md Part III（C/C++ 安全编码）+ 10-coding-style/Rust_coding_style.md §4-§7（Rust 安全编码）+ 110-security/01-lsm-framework.md + 110-security/02-landlock-sandbox.md（LSM/Cupolas 安全规则）
>
> **编号空间分配（2026-07-14 消解三重冲突）**：历史 OS-SEC 编号在三个文档体系中有三套完全不同的定义（LSM 安全规则 / C 安全编码 / Rust 安全编码），导致同一编号指代完全不同的规则。本节消解此三重冲突，按编号段分配子域：
>
> | 编号段 | 子域 | 权威定义文档 | 迁移偏移 |
> |--------|------|-------------|---------|
> | OS-SEC-001~099 | LSM/Cupolas 安全规则 | 110-security/ | 不变 |
> | OS-SEC-100~199 | C/C++ 安全编码规则 | C_Cpp_coding_style.md Part III | +100 |
> | OS-SEC-200~299 | Rust 安全编码规则 | Rust_coding_style.md §4-§7 | +200 |

### 9.1 LSM/Cupolas 安全规则（OS-SEC-001~099）

> **权威定义文档**：110-security/01-lsm-framework.md + 110-security/02-landlock-sandbox.md

| 编号 | 规则 | 权威定义 |
|------|------|---------|
| OS-SEC-001 | LSM 钩子内置每一关键路径，不可外挂补丁替代 | 01-lsm-framework.md |
| OS-SEC-002 | 钩子链表 RCU 不变量需通过形式化检查 | 01-lsm-framework.md |
| OS-SEC-003 | Cupolas 必须在 capability 之后初始化 | 01-lsm-framework.md |
| OS-SEC-004 | 沙箱能力内置内核，Agent 无需特权即可施加 | 02-landlock-sandbox.md |
| OS-SEC-005 | 域的不可逆性需通过形式化检查 | 02-landlock-sandbox.md |
| OS-SEC-006 | 沙箱施加前后必须通过 AgentsIPC 上报事件 | 02-landlock-sandbox.md |
| OS-SEC-007 | Landlock 域不可替代 capability，必须与之共存 | 02-landlock-sandbox.md |
| OS-SEC-008 | Cupolas 钩子回调必须返回 [SC] 4 值枚举之一（ALLOW/DENY/ASK/LOG） | 01-lsm-framework.md |
| OS-SEC-009 | ASK 裁决必须通过 AgentsIPC 询问 daemon，5 秒超时后回退 ALLOW 并记录 LOG | 01-lsm-framework.md |
| OS-SEC-010 | 审计事件必须包含 agent_id 字段，缺失 agent_id 的事件必须丢弃并告警 | 01-lsm-framework.md |
| OS-SEC-011 | Cupolas 7 子系统钩子映射必须与 [SC] 钩子 ID 一一对应，新增子系统必须扩展映射表 | 01-lsm-framework.md |
| OS-SEC-012 | Agent 沙箱必须至少 3 层域叠加（L0 全局 + L1 角色 + L2 实例），禁止少于 3 层 | 02-landlock-sandbox.md |
| OS-SEC-013 | 访问掩码合并必须用交集语义（final_mask = L0 & L1 & L2），禁止并集或最宽优先 | 02-landlock-sandbox.md |
| OS-SEC-014 | landlock_restrict_self 必须在 Agent 进程 fork() 后立即调用，禁止延迟施加 | 02-landlock-sandbox.md |
| OS-SEC-015 | 沙箱创建失败必须终止 Agent 启动，禁止降级运行 | 02-landlock-sandbox.md |

### 9.2 C/C++ 安全编码规则（OS-SEC-100~199）

> **权威定义文档**：C_Cpp_coding_style.md Part III
> **迁移说明（2026-07-14）**：原 OS-SEC-010~027 迁移至 OS-SEC-110~127（+100 偏移），消除与 LSM 安全规则 OS-SEC-001~099 的编号冲突。

| 编号 | 规则 | 权威定义 | 原编号 |
|------|------|---------|--------|
| OS-SEC-110 | 边界检查原则——所有涉及用户可控长度参数的缓冲区操作必须进行边界检查 | C_Cpp Part III §1.1 | OS-SEC-010 |
| OS-SEC-111 | 安全内存复制函数——内核态与用户态之间使用 copy_from_user/copy_to_user | C_Cpp Part III §1.3 | OS-SEC-011 |
| OS-SEC-112 | 整数溢出检测——使用 check_add_overflow/check_mul_overflow 等溢出安全函数 | C_Cpp Part III §2.1 | OS-SEC-012 |
| OS-SEC-113 | 类型转换安全——避免隐式类型转换导致的数据截断 | C_Cpp Part III §2.2 | OS-SEC-013 |
| OS-SEC-114 | 符号问题——避免有符号/无符号混用 | C_Cpp Part III §2.3 | OS-SEC-014 |
| OS-SEC-115 | 禁止用户可控格式化字符串 | C_Cpp Part III §3.1 | OS-SEC-015 |
| OS-SEC-116 | 格式化函数 __printf 注解 | C_Cpp Part III §3.2 | OS-SEC-016 |
| OS-SEC-117 | NULL 检查——外部指针在解引用前必须进行 NULL 检查 | C_Cpp Part III §4.1 | OS-SEC-017 |
| OS-SEC-118 | 悬挂指针防护——释放内存后立即置 NULL，使用 kfree_sensitive | C_Cpp Part III §4.2 | OS-SEC-018 |
| OS-SEC-119 | 数据竞争防护——共享数据必须通过同步原语保护，KCSAN 检测 | C_Cpp Part III §5.1 | OS-SEC-019 |
| OS-SEC-120 | 死锁预防——多锁获取遵循固定锁顺序 | C_Cpp Part III §5.2 | OS-SEC-020 |
| OS-SEC-121 | capability 检查——安全敏感操作必须通过 Cupolas capability 系统检查 | C_Cpp Part III §6.1 | OS-SEC-021 |
| OS-SEC-122 | capability 令牌管理——安全通道传递，禁止日志打印原始值 | C_Cpp Part III §6.2 | OS-SEC-022 |
| OS-SEC-123 | 所有用户态输入必须验证（长度/范围/类型/格式） | C_Cpp Part III §7.1 | OS-SEC-023 |
| OS-SEC-124 | 系统调用参数验证模板——UAPI 结构体字段验证不遗漏 | C_Cpp Part III §7.2 | OS-SEC-024 |
| OS-SEC-125 | copy_to_user 敏感数据——确保不含内核地址/未初始化内存 | C_Cpp Part III §8.1 | OS-SEC-025 |
| OS-SEC-126 | 内核地址保护——禁止向用户态泄露内核地址 | C_Cpp Part III §8.2 | OS-SEC-026 |
| OS-SEC-127 | 敏感数据清零——使用 memzero_explicit/kfree_sensitive | C_Cpp Part III §8.3 | OS-SEC-027 |

### 9.3 Rust 安全编码规则（OS-SEC-200~299）

> **权威定义文档**：Rust_coding_style.md §4-§7
> **迁移说明（2026-07-14）**：原 OS-SEC-001~003 迁移至 OS-SEC-201~203，原 OS-SEC-030~047 迁移至 OS-SEC-230~247（+200 偏移），消除与 LSM 安全规则 OS-SEC-001~099 的编号冲突。

| 编号 | 规则 | 权威定义 | 原编号 |
|------|------|---------|--------|
| OS-SEC-201 | unsafe 代码块最小化——仅包裹真正需要 unsafe 的操作 | Rust §4.1 | OS-SEC-001 |
| OS-SEC-202 | SAFETY 注释文档化——每个 unsafe 块包含 // SAFETY: 注释 | Rust §4.2 | OS-SEC-002 |
| OS-SEC-203 | unsafe 代码审查——至少两名资深维护者审查 | Rust §4.3 | OS-SEC-003 |
| OS-SEC-230 | unsafe 审计五步法 | Rust §1.2 | OS-SEC-030 |
| OS-SEC-231 | unsafe 代码审查清单 | Rust §1.3 | OS-SEC-031 |
| OS-SEC-232 | 借用检查器利用——静态验证并发安全性 | Rust §2.2 | OS-SEC-032 |
| OS-SEC-233 | 智能指针与 RAII——资源生命周期自动管理 | Rust §2.3 | OS-SEC-033 |
| OS-SEC-234 | Send 与 Sync trait——跨线程传递/共享类型约束 | Rust §3.1 | OS-SEC-034 |
| OS-SEC-235 | Mutex 与 RwLock——编译器保证锁释放 | Rust §3.2 | OS-SEC-035 |
| OS-SEC-236 | 原子操作——使用 AtomicU32/AtomicBool 替代锁 | Rust §3.3 | OS-SEC-036 |
| OS-SEC-237 | DMA 缓冲区安全——硬件直接访问不受 MMU 保护 | Rust §4.1 | OS-SEC-037 |
| OS-SEC-238 | MMIO 寄存器访问——通过 accessor 函数，禁止裸指针 | Rust §4.2 | OS-SEC-038 |
| OS-SEC-239 | 裸指针与内核链表——使用 kernel::list 安全封装 | Rust §4.3 | OS-SEC-039 |
| OS-SEC-240 | C 调用约定与 ABI 稳定性——extern "C" 声明 | Rust §5.1 | OS-SEC-040 |
| OS-SEC-241 | 类型布局兼容性——#[repr(C, packed)] 确保与 C 布局兼容 | Rust §5.2 | OS-SEC-041 |
| OS-SEC-242 | 所有权穿越 FFI 边界——明确约定所有权转移 | Rust §5.3 | OS-SEC-042 |
| OS-SEC-243 | cargo audit——CRITICAL 级漏洞阻断合并 | Rust §6.1 | OS-SEC-043 |
| OS-SEC-244 | 依赖审查——新增依赖必须审查内容/许可证/安全 | Rust §6.2 | OS-SEC-044 |
| OS-SEC-245 | 最小依赖原则——能用标准库不引入外部 crate | Rust §6.3 | OS-SEC-045 |
| OS-SEC-246 | Kani 验证器——Rust 形式化验证 | Rust §7.1 | OS-SEC-046 |
| OS-SEC-247 | Creusot 验证器——Rust 规范验证 | Rust §7.2 | OS-SEC-047 |

---

## 附录 A 变更日志

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|---------|------|
| 2026-07-12 | 0.1.1 | 创建本注册表，替代碎片化注册表（00 §3 + 07 §10.3）；完成 OS-KER 001-155 全量消解（086-155 为新分配编号，消除 42 处跨文档冲突） | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | 第二轮 SSoT 整合：创建 3 个族索引文件（01-ssot-code-standards-index.md / 05-ssot-process-toolchain-index.md / 10-coding-style/00-ssot-c-cpp-index.md）；09-license-strategy.md 降级为 stub 重定向至 110-spdx-license-compliance.md；TERMINOLOGY.md 移入本模块并重命名为 90-terminology.md | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B1 其他语言族索引扩展：创建 3 个语言族索引（10-coding-style/00-ssot-rust-index.md / 00-ssot-go-index.md / 00-ssot-python-js-java-index.md）；发现 Rust 文件 OS-STD-030~035 + OS-SEC 编号登记缺口（OS-STD-030~046 已于 2026-07-14 完整登记至 §4.5；OS-SEC 三重冲突已于 2026-07-14 消解，见 §9 OS-SEC 安全工程规则——编号空间分配 001~099 LSM / 100~199 C / 200~299 Rust） | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B2 横切关注点族索引：创建 3 个横切族索引（10-coding-style/00-ssot-docs-index.md 文档族 / 00-ssot-logging-index.md 日志族 / 00-ssot-security-index.md 安全设计族）；横切族索引与语言族索引形成正交叉，同一 secure_coding 文件同时属于语言族和安全族 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B3 子目录 README 登记到 SSoT：5 个子目录 README（10/20/30/40/50）头部追加 SSoT 依赖声明（编号权威 09 §3 + 族索引导航 §4.5）；09-ssot-registry.md 新增 §4.6 子目录 README SSoT 依赖登记表；20-contracts/README.md 修复父文档路径（../../README.md → ../00-engineering-standards-handbook.md） | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B4 扩展文档 SSoT 依赖声明：6 个扩展文档（60/70/80/100/110/120）头部追加编号权威字段（09 §3）+ SSoT 依赖声明块；80 已有 SSoT 声明，追加 09 §3 引用；110 声明为 SPDX 许可证策略唯一 SSoT；120 声明 IRON-9 v2 三层模型 + IPC magic 权威引用 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B5 第二轮全模块审查：11 维度（D1-D11）全部 PASS；63 文件计数一致；§4.5 族索引 9 条 + §4.6 子目录 README 5 条；发现并修复 2 处格式不一致——(1) 80-clang-format-enforcement.md "SSoT 声明" → "SSoT 依赖声明"（措辞统一）；(2) 10-coding-style/README.md §9 编号冲突（"版本历史"重编为 §11）；Rust 编号缺口已文档化，留待后续消解 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B6 第二轮 V1-V12 验证：12/12 项全部 PASS；V1 编号唯一性（Rust 缺口已文档化）+ V2 SSoT 三层一致 + V3 §4.5 九条/§4.6 五条 + V4 90-terminology 登记 + V5 stub 正确 + V6 0 活跃旧路径 + V7 README 9 族索引引用 + V8 B1-B5 五条 changelog + V9 9/9 族索引文件存在 + V10 §4.6 五条 README + V11 6/6 扩展文档 SSoT 声明 + V12 9/9 族索引格式一致；第二阶段整合工作完成 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | Wave 2 NEW 命名整理：消除数字前缀冲突——根目录 3 组（01/05/09）+ 子目录 7 文件（00）；SSoT 族索引文件统一使用 200 段唯一编号（200-206 + 200-201 根目录）；09-license-strategy.md → 15-license-strategy.md（09-ssot-registry.md 保留 09 前缀体现权威地位）；§4.5 族索引登记表 + §4.6 子目录 README + §8.2 废弃文档登记同步更新 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | **SSoT 重新设计**：取消三层镜像模型（00 §3 + 07 §10 改为引用链接，消除 ~448 行重复）；删除 9 个族索引文件（200/201 根目录 + 200-206 子目录）+ 1 个 stub 文件（15-license-strategy.md 物理删除）；§4.5 族索引登记表整节废除；§4.6 子目录 README 登记更新（移除族索引导航引用）；§8.2 废弃文档登记更新（15-license-strategy.md 标记为已物理删除）；文件数从 63 减至 53。**设计原则**：只保留 09-ssot-registry.md 一个 SSoT 文档，废弃文档全部删除，文档不能越来越多，要精要准确 | SPHARX 工程标准组 |
| 2026-07-13 | 0.1.1 | 10-coding-style 文件合并：16 份源文件合并为 5 份合并文档（C_Cpp_coding_style.md / Rust_coding_style.md / Go_coding_style.md / scripting_coding_style.md / coding_conventions.md），10-coding-style 文件数从 17 减至 6（含 README）；本注册表同步更新：§3 编号区间说明 + §3.2 章节标题与引用（C_coding_style_standard.md → C_Cpp_coding_style.md Part I）+ §4.7 子目录 README 登记表（naming_conventions.md → coding_conventions.md Part IV）+ §8 命名规范权威文档引用（naming_conventions.md §4.4/§4.4.6/§1.10 → coding_conventions.md Part IV §4.4/§4.4.6/§1.10） | SPHARX 工程标准组 |
| 2026-07-13 | 0.1.1 | **顶层文件合并（B 组）**：8 份源文件合并为 2 份合并文档 + 1 份废弃文档删除。`01-coding-standards.md` 合并 5 份源文件（02-code-format→Part II / 03-code-style→Part III / 60-checkpatch-rule-map→Part IV / 70-kernel-doc-standard→Part V / 80-clang-format-enforcement→Part VI），692→3667 行；`05-development-process.md` 合并 3 份源文件（06-toolchain-and-automation→Part II / 08-compliance-checklist→Part III / 110-spdx-license-compliance→Part IV），666→2685 行；`100-deprecated-api-registry.md` 废弃删除（642 行）。本注册表同步更新：§3.6.7 标题（03-code-style→01 Part III）+ §4.2 OS-STD-FMT 权威文档（02→01 Part II）+ §4.3 OS-STD-STY 权威文档（03→01 Part III）+ §4.5 OS-STD-TOOL 权威文档与编号段归属表（06→05 Part II）+ §4.5 历史编号等价声明（06→05 Part II）+ §8.2 废弃文档登记（新增 9 条 + 15-license 重定向至 05 Part IV）。版本规则 IRON-8 验证通过（0 处违规） | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v2.0-P1-01 修复：新增 §4.8 OS-FMT 闭源总纲代码格式规则章节，登记 OS-FMT-001~007（Tab 8 缩进 / 80 列行宽 / K&R 大括号 / 单语句大括号 / 空格规则 / 行尾禁止空白 / clang-format 配置）；规则源自闭源总纲 §3.1-3.6 | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v2.0-P1-03 修复：新增 §2.1 ES-SEL4 编号范围声明，澄清三段作用域——ES-SEL4-1~5 架构层强制借鉴（已注册 OS-IRON-012）/ ES-SEL4-6~10 P2 增强建议（已注册闭源总纲附录 D）/ ES-SEL4-11~43 研究范围编号（`_research_0.2.5/01-sel4-deep-analysis.md` 研究范围，非工程标准范围，开源文档引用时须标注"研究范围参考"） | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v1.0-P1-02 修复：OS-KER-005 跳号补齐——标记为已废弃，原定义为"C 内核代码 Tab 8 字符缩进"，因格式规则重新组织（内容迁移至 OS-STD-FMT-001 / 闭源 OS-FMT-001）而废弃 | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v1.0-P2-04 修复：新增 §8.3 文档计数三作用域模型登记——~79 文档（P0 基线，"文档体系完成"上下文）/ ~85 文档（工程基线，"0.1.1 合计/总产出"上下文）/ ~122 文档（全量展开，"全部三大支柱"上下文） | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v1.0-P2-05 修复：附录 A 变更日志补齐本次修复会话（2026-07-14）全部条目——OS-FMT-001~007 注册 / ES-SEL4-11~43 研究范围声明 / OS-KER-005 废弃 / 文档计数三作用域模型注册 | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **P0 修复：消解 OS-SEC 三重编号冲突**——历史 OS-SEC 编号在 110-security/（LSM/Cupolas 安全规则 001~015）、C_Cpp_coding_style.md（C 安全编码 010~027）、Rust_coding_style.md（Rust 安全编码 001~003+030~047）三处有完全不同的定义且未在 SSoT 登记。编号空间重新分配：OS-SEC-001~099 LSM 安全规则（不变）/ OS-SEC-100~199 C 安全编码（+100 偏移）/ OS-SEC-200~299 Rust 安全编码（+200 偏移）；新增 §9 OS-SEC 安全工程规则章节登记全部 54 个编号；同步修改 C_Cpp_coding_style.md 18 个编号（010~027→110~127）+ Rust_coding_style.md 21 个编号（001~003→201~203, 030~047→230~247） | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：补全 Rust OS-STD-045~046 登记**——§4.5 OS-STD-RUST 表新增 OS-STD-045（FFI 类型映射——`core::ffi`/`kernel::ffi` C 兼容类型，Part I §8.2）+ OS-STD-046（FFI 所有权语义——`#[ownership]` 标注，Part I §8.3）；Rust 编码规则 OS-STD-030~046 全部 17 项完整登记，消除 Rust 编号登记缺口 | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：文档化 OLK-6.6 品味哲学 8 项隐性智慧**——01-coding-standards.md 新增 §8.5 章节（OS-STD-STY-005~012）：信任读者/短函数/简洁文件头/EXPORT_SYMBOL 紧跟/include 语义分组/`__` 前缀/`__read_mostly`/`__randomize_layout`；每项含 OLK-6.6 源码证据（`mutex.c`/`core.c`/`sched.h`）+ 好坏代码示例；SSoT §4.3 同步登记 8 个新编号；§10 五维映射表更新 A-1/A-2/A-4/E-1/E-3 映射 | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：消解 ES-SEL4-9/10 状态分类模糊**——闭源总纲 §12.8 引言文本区分"ES-SEL4-7/9/10 已在 0.1.1 阶段提前落地（已落地）"与"ES-SEL4-6/8 作为 1.0.1 阶段增量待落地（待落地）"；表格新增"状态"列明确标注每项落地状态，消除"P2 增强建议在 1.0.1 阶段增量落地"引言与"已落地"正文之间的语义冲突 | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：补齐 OS-KER-007/008 跳号废弃说明**——§3.1 新增 OS-KER-007（原"内核态禁 float，强制 q16.16 定点"，2026-07-12 迁移至 OS-STD-010）+ OS-KER-008（原"函数例外：开括号在下一行行首（K&R）"，2026-07-12 与 OS-STD-047 合并迁移至 OS-STD-FMT-009）废弃说明条目；与 01-coding-standards.md Part II §0.2.1 迁移映射表一致，消除 OS-KER-001~010 区间的跳号未标注问题，SSoT 内部一致性完整 | SPHARX 工程标准组 |
