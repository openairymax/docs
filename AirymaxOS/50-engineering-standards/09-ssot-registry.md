Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# AirymaxOS 全局规则编号注册表（SSoT）

> **文档定位**：AirymaxOS（agentrt-linux）全部编号、规则、命名的**唯一权威来源（Single Source of Truth）**。任何文档不得私自定义规则编号；所有规则编号必须在此注册表中登记后方可使用。
>
> **版本**：0.1.1\
> **最后更新**：2026-07-12\
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
| OS-KER-006 | volatile 仅 4 种例外 | 01 §6.3 |
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

### 4.2 OS-STD-FMT 格式规范

> **权威定义文档**：01-coding-standards.md Part II（29 条，详见 Part II §0.2.1 迁移映射表）

完整 29 条清单见 [01-coding-standards.md Part II §0.2.1](./01-coding-standards.md)。此处不重复，以 01-coding-standards.md Part II 为唯一权威来源。

### 4.3 OS-STD-STY 风格规范

> **权威定义文档**：01-coding-standards.md Part III

| 编号 | 规则 |
|------|------|
| OS-STD-STY-001 | 反过度抽象，不预先抽象 |
| OS-STD-STY-002 | 引用计数强制，锁不替代引用计数 |

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

### 4.5 OS-STD-TOOL 工具链规则编号段声明

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

> **历史编号等价声明**：05-development-process.md Part II 正文中现存的历史编号 `OS-STD-002~158` 与目标 `OS-STD-TOOL-002~158` 等价有效，规则效力以 Part II 正文为准。CI/CD 脚本与 GitHub Actions workflow 实施时统一替换为 `OS-STD-TOOL-NNN`。

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

---

## 附录 A 变更日志

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|---------|------|
| 2026-07-12 | 0.1.1 | 创建本注册表，替代碎片化注册表（00 §3 + 07 §10.3）；完成 OS-KER 001-155 全量消解（086-155 为新分配编号，消除 42 处跨文档冲突） | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | 第二轮 SSoT 整合：创建 3 个族索引文件（01-ssot-code-standards-index.md / 05-ssot-process-toolchain-index.md / 10-coding-style/00-ssot-c-cpp-index.md）；09-license-strategy.md 降级为 stub 重定向至 110-spdx-license-compliance.md；TERMINOLOGY.md 移入本模块并重命名为 90-terminology.md | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B1 其他语言族索引扩展：创建 3 个语言族索引（10-coding-style/00-ssot-rust-index.md / 00-ssot-go-index.md / 00-ssot-python-js-java-index.md）；发现 Rust 文件 OS-STD-030~035 + OS-SEC-001~002,030~037+ 编号登记缺口，留待 B5 审查消解 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B2 横切关注点族索引：创建 3 个横切族索引（10-coding-style/00-ssot-docs-index.md 文档族 / 00-ssot-logging-index.md 日志族 / 00-ssot-security-index.md 安全设计族）；横切族索引与语言族索引形成正交叉，同一 secure_coding 文件同时属于语言族和安全族 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B3 子目录 README 登记到 SSoT：5 个子目录 README（10/20/30/40/50）头部追加 SSoT 依赖声明（编号权威 09 §3 + 族索引导航 §4.5）；09-ssot-registry.md 新增 §4.6 子目录 README SSoT 依赖登记表；20-contracts/README.md 修复父文档路径（../../README.md → ../00-engineering-standards-handbook.md） | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B4 扩展文档 SSoT 依赖声明：6 个扩展文档（60/70/80/100/110/120）头部追加编号权威字段（09 §3）+ SSoT 依赖声明块；80 已有 SSoT 声明，追加 09 §3 引用；110 声明为 SPDX 许可证策略唯一 SSoT；120 声明 IRON-9 v2 三层模型 + IPC magic 权威引用 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B5 第二轮全模块审查：11 维度（D1-D11）全部 PASS；63 文件计数一致；§4.5 族索引 9 条 + §4.6 子目录 README 5 条；发现并修复 2 处格式不一致——(1) 80-clang-format-enforcement.md "SSoT 声明" → "SSoT 依赖声明"（措辞统一）；(2) 10-coding-style/README.md §9 编号冲突（"版本历史"重编为 §11）；Rust 编号缺口已文档化，留待后续消解 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B6 第二轮 V1-V12 验证：12/12 项全部 PASS；V1 编号唯一性（Rust 缺口已文档化）+ V2 SSoT 三层一致 + V3 §4.5 九条/§4.6 五条 + V4 90-terminology 登记 + V5 stub 正确 + V6 0 活跃旧路径 + V7 README 9 族索引引用 + V8 B1-B5 五条 changelog + V9 9/9 族索引文件存在 + V10 §4.6 五条 README + V11 6/6 扩展文档 SSoT 声明 + V12 9/9 族索引格式一致；第二阶段整合工作完成 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | Wave 2 NEW 命名整理：消除数字前缀冲突——根目录 3 组（01/05/09）+ 子目录 7 文件（00）；SSoT 族索引文件统一使用 200 段唯一编号（200-206 + 200-201 根目录）；09-license-strategy.md → 15-license-strategy.md（09-ssot-registry.md 保留 09 前缀体现权威地位）；§4.5 族索引登记表 + §4.6 子目录 README + §8.2 废弃文档登记同步更新 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | **SSoT 重新设计**：取消三层镜像模型（00 §3 + 07 §10 改为引用链接，消除 ~448 行重复）；删除 9 个族索引文件（200/201 根目录 + 200-206 子目录）+ 1 个 stub 文件（15-license-strategy.md 物理删除）；§4.5 族索引登记表整节废除；§4.6 子目录 README 登记更新（移除族索引导航引用）；§8.2 废弃文档登记更新（15-license-strategy.md 标记为已物理删除）；文件数从 63 减至 53。**设计原则**：只保留 09-ssot-registry.md 一个 SSoT 文档，废弃文档全部删除，文档不能越来越多，要精要准确 | SPHARX 工程标准组 |
| 2026-07-13 | 0.1.1 | 10-coding-style 文件合并：16 份源文件合并为 5 份合并文档（C_Cpp_coding_style.md / Rust_coding_style.md / Go_coding_style.md / scripting_coding_style.md / coding_conventions.md），10-coding-style 文件数从 17 减至 6（含 README）；本注册表同步更新：§3 编号区间说明 + §3.2 章节标题与引用（C_coding_style_standard.md → C_Cpp_coding_style.md Part I）+ §4.7 子目录 README 登记表（naming_conventions.md → coding_conventions.md Part IV）+ §8 命名规范权威文档引用（naming_conventions.md §4.4/§4.4.6/§1.10 → coding_conventions.md Part IV §4.4/§4.4.6/§1.10） | SPHARX 工程标准组 |
| 2026-07-13 | 0.1.1 | **顶层文件合并（B 组）**：8 份源文件合并为 2 份合并文档 + 1 份废弃文档删除。`01-coding-standards.md` 合并 5 份源文件（02-code-format→Part II / 03-code-style→Part III / 60-checkpatch-rule-map→Part IV / 70-kernel-doc-standard→Part V / 80-clang-format-enforcement→Part VI），692→3667 行；`05-development-process.md` 合并 3 份源文件（06-toolchain-and-automation→Part II / 08-compliance-checklist→Part III / 110-spdx-license-compliance→Part IV），666→2685 行；`100-deprecated-api-registry.md` 废弃删除（642 行）。本注册表同步更新：§3.6.7 标题（03-code-style→01 Part III）+ §4.2 OS-STD-FMT 权威文档（02→01 Part II）+ §4.3 OS-STD-STY 权威文档（03→01 Part III）+ §4.5 OS-STD-TOOL 权威文档与编号段归属表（06→05 Part II）+ §4.5 历史编号等价声明（06→05 Part II）+ §8.2 废弃文档登记（新增 9 条 + 15-license 重定向至 05 Part IV）。版本规则 IRON-8 验证通过（0 处违规） | SPHARX 工程标准组 |
