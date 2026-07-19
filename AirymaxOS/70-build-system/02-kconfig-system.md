Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）Kconfig 配置系统详解
> **文档定位**：agentrt-linux（AirymaxOS）构建系统第 2 卷——Kconfig 配置系统详解。本卷剖析 Kconfig 语法（`config`/`menuconfig`/`choice`/`depends on`/`select`）、`CONFIG_*` 宏与 `obj-$(CONFIG_*)` 门控、Kconfig 子目录组织、`Kconfig.airymaxos` 供应商扩展、配置工具（`menuconfig`/`nconfig`/`gconfig`）与 `KCONFIG_ALLCONFIG` 全配置覆盖机制。\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-06\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：agentrt `cmake/`（用户态选项缓存）+ Linux 6.6 Kconfig 系统（`Kconfig`、`lib/Kconfig`、`lib/Kconfig.debug`、`Kconfig.airymaxos`、`scripts/kconfig/`）\
> **理论根基**：Linux 6.6 内核基线 Kconfig 工程 + Airymax 五维正交 24 原则（S/K/C/E/A 五维）\
> **核心约束**：IRON-9 同源且部分代码共享（IRON-9 v3）——agentrt-linux 配置门控沿用 Kconfig 语义但供应商扩展独立维护

---

## 0. 章节定位

本卷是 agentrt-linux 构建系统 8 卷文档中的第 2 卷，回答"配置如何驱动构建"这一问题。它在 01-kbuild-system.md（递归构建）与 03-makefile-patterns.md（Makefile 惯用法）之间形成配置门控层：

- **上游依赖**：01 定义"构建如何被驱动"；本卷定义"构建目标的三态从何而来"——`CONFIG_*` 宏由 Kconfig 求解，再被 `obj-$(CONFIG_*)` 引用决定 `obj-y`/`obj-m`/`obj-n`。
- **下游依赖**：03 定义 Makefile 如何引用 `CONFIG_*`；04 定义模块构建如何被 `CONFIG_*=m` 驱动；06 定义 agentrt-linux 多仓配置如何与内核 Kconfig 协调。

本卷所有强制规则均赋予 **OS-KER** / **OS-STD** / **OS-BUILD** 编号，与 50-engineering-standards/07 维护者制度的"规则编号注册表"对齐。agentrt-linux 配置系统以 **Linux 6.6 内核基线** Kconfig 工程思想为来源，融合 Airymax **五维正交 24 原则** 后重新表述为工程契约。

### 0.1 关键术语

| 术语 | 定义 |
|------|------|
| Kconfig | 配置描述语言与解析器，求解 `CONFIG_*` 宏的三态值（y/m/n） |
| CONFIG_* | 由 Kconfig 求解并写入 `auto.conf`/`.config` 的配置宏 |
| config | Kconfig 语法：声明一个配置选项 |
| menuconfig | Kconfig 语法：声明一个可展开子菜单的配置选项 |
| choice | Kconfig 语法：互斥选项组（如调度策略二选一） |
| depends on | Kconfig 属性：声明选项的前置依赖 |
| select | Kconfig 属性：强制选中另一选项（反向依赖） |
| syncconfig | 把 `.config` 同步为 `auto.conf`/`autoconf.h` 的子命令（旧称 silentoldconfig） |
| KCONFIG_ALLCONFIG | 覆盖构建中"全部启用/禁用"基线的环境变量 |
| 五维正交 24 原则 | Airymax 架构设计原则体系（S/K/C/E/A 五维） |

---

## 1. Kconfig 配置系统总览

agentrt-linux 配置系统继承 **Linux 6.6 内核基线** 的 Kconfig 工程：用一种声明式语言描述配置选项及其依赖，由配置工具（`menuconfig` 等）求解为 `CONFIG_*=y/m/n` 三态宏，写入 `.config` 与 `include/config/auto.conf`，再被 Kbuild 的 `obj-$(CONFIG_*)` 引用驱动构建。

### 1.1 配置求解流水线

```mermaid
flowchart LR
    A[Kconfig 源文件树] -->|解析| B[配置工具 menuconfig]
    C[.config 用户选择] --> B
    B -->|求解依赖闭包| D[.config 最终值]
    D -->|syncconfig| E[auto.conf]
    D -->|syncconfig| F[autoconf.h]
    D -->|syncconfig| G[include/config/*.h 时间戳]
    E -->|Make 引用| H[obj-$(CONFIG_*) 门控]
    F -->|C 源码引用| I[#ifdef CONFIG_XXX]
    G -->|fixdep 依赖| J[配置变化触发重编]
```

关键点：Kconfig 不仅求解宏值，还求解**依赖闭包**——当 `select` 强制选中某选项时，其依赖必须同时满足；当 `depends on` 不满足时选项不可见。`syncconfig` 把 `.config` 转化为 make 侧（`auto.conf`）、C 侧（`autoconf.h`）与依赖时间戳（`include/config/*.h`）三套产物，使配置变化能被 `if_changed_dep`/`fixdep` 捕获。

- **OS-KER-030**：agentrt-linux 内核态配置必须以 Kconfig 为唯一描述手段，禁止用 `Makefile` 的 `ifeq`/环境变量绕过 Kconfig 做特性开关（对齐 K-4 可插拔策略）。
- **OS-BUILD-021**：所有 `CONFIG_*` 宏必须由 Kconfig 求解并写入 `auto.conf`，禁止在 `Makefile` 中手动 `CONFIG_XXX := y`；手动赋值会绕过依赖求解导致配置不一致。

---

## 2. Kconfig 语法

Kconfig 语法在 **Linux 6.6 内核基线** 的 `Documentation/kbuild/kconfig-language.rst` 中定义。本节给出 agentrt-linux 子系统最常用的语法形态。

### 2.1 config：声明配置选项

```
# drivers/airymax/ipc/Kconfig
config AIRY_IPC
	bool "agentrt-linux AgentsIPC kernel channel"
	depends on IO_URING
	default y
	help
	  Enable the agentrt-linux AgentsIPC 128B fixed-length message
	  channel built on io_uring. This is the OS-level transport
	  for agent runtime messages. Say Y unless you have a
	  specific reason to disable it.
```

字段含义：`config <NAME>` 声明选项（名字大写下划线）；`bool`/`tristate`/`int`/`hex`/`string` 声明类型；`prompt`（紧跟类型的字符串）是菜单显示文本；`depends on` 声明前置依赖；`default` 声明默认值；`help` 是帮助文本（即文档，对齐 E-7 文档即代码）。

- **OS-STD-143**（沿用 50 卷）：新增 `CONFIG_*` 选项默认 `off`（`default n` 或省略），仅安全核心可 `default y`；默认开启需经安全评审。
- **OS-STD-144**（沿用 50 卷）：新增 `CONFIG_*` 选项必须有 `help` 文本，说明用途、依赖与风险。
- **OS-KER-155**：`config` 名字必须以 `AIRY_` 或所属子系统前缀开头（如 `AIRY_IPC_`），禁止与上游 `CONFIG_*` 名字冲突（对齐 IRON-9 独立性）。

### 2.2 menuconfig：可展开子菜单

```
menuconfig AIRY_IPC_ADVANCED
	bool "agentrt-linux IPC advanced options"
	depends on AIRY_IPC
	help
	  Advanced tuning for the agentrt-linux AgentsIPC channel.

if AIRY_IPC_ADVANCED

config AIRY_IPC_BENCH
	bool "IPC benchmarking hooks"
	help
	  Insert benchmarking counters into the IPC hot path.

config AIRY_IPC_RINGBUF_ORDER
	int "Ring buffer page order"
	range 0 8
	default 2
	help
	  Page order of the per-CPU ring buffer.

endif # AIRY_IPC_ADVANCED
```

`menuconfig` 与 `config` 的区别在于前者在菜单界面可折叠展开子选项；`if <EXPR> ... endif` 包裹的选项仅在表达式为真时可见。`range` 约束 `int`/`hex` 的取值范围。

- **OS-STD-145**（沿用 50 卷）：相关选项必须用 `menuconfig` + `if` 分组，禁止散落在顶层菜单；分组提升可读性（A-2 细节关注）。

### 2.3 choice：互斥选项组

```
choice
	prompt "agentrt-linux agent scheduler policy"
	default stc_agent
	depends on SCHED_CLASS_EXT

config stc_agent
	bool "AIRY_USER_SCHED (sched_tac)"
	help
	  Use the sched_tac user-space scheduler based on
	  SCHED_DEADLINE/SCHED_FIFO/EEVDF for agent task scheduling.

config AIRY_SCHED_FIFO
	bool "SCHED_FIFO legacy"
	help
	  Use legacy SCHED_FIFO for agent scheduling. Only for
	  debugging fallback.

endchoice
```

`choice` 内的选项互斥，只有一个可为 `y`；`default` 指定默认选中项。agentrt-linux 用 `choice` 表达"二选一/多选一"的调度策略、内存模型等决策。

- **OS-KER-022**：互斥决策（如调度策略、IPC 传输后端）必须用 `choice`，禁止用多个独立 `bool` 模拟互斥；`choice` 在求解时自动保证唯一性（对齐 S-1 单一职责）。

### 2.4 depends on 与 select

`depends on` 是正向依赖：选项可见/可选的前提；`select` 是反向依赖：选中本项则强制选中目标项（即使目标的 `depends on` 未满足也会被强制，需谨慎）。

```
config AIRY_IPC
	tristate
	depends on IO_URING
	select CRC32
	help
	  The IPC channel requires io_uring and CRC32 for header
	  integrity.

config AIRY_IPC_ZERO_COPY
	bool
	depends on AIRY_IPC
	select IO_URING_TASK_WORK
	help
	  Zero-copy fast path requires io_uring task work.
```

`select` 的风险：若被 `select` 的目标有 `depends on` 未满足，会产生"不满足依赖但被强制选中"的不一致配置，Kconfig 会警告（`select` 被视为强约束）。

- **OS-STD-146**（沿用 50 卷）：`select` 仅用于"本选项在功能上隐含另一选项"的强约束场景（如选 IPC 必选 CRC32）；弱关联用 `depends on`；滥用 `select` 制造配置悖论禁止合并。
- **OS-KER-023**：`select` 的目标若有 `depends on`，本选项必须同时 `select` 其全部依赖，或在 `depends on` 中显式声明，避免配置不一致（对齐 A-4 完美主义）。

### 2.5 类型与默认值

| 类型 | 取值 | 典型用途 |
|------|------|---------|
| `bool` | y/n | 二态开关（最常见） |
| `tristate` | y/m/n | 三态（内建/模块/关闭） |
| `int` | 整数 | 数值参数（缓冲区大小、超时） |
| `hex` | 十六进制 | 地址、掩码 |
| `string` | 字符串 | 路径、名称 |

`tristate` 是 Kconfig 区别于普通配置系统的关键：它允许同一选项在 `y`（内建进 vmlinux）与 `m`（可加载模块）间切换，直接映射到 Kbuild 的 `obj-y`/`obj-m`。

- **OS-KER-024**：可能作为可加载模块的子系统必须用 `tristate` 而非 `bool`；用 `bool` 会丧失模块化能力（对齐 K-4 可插拔策略）。

### 2.6 Kconfig 配置项三态选择状态机

Kconfig 配置项从未被引用到最终求解为 `y`/`m`/`n` 的完整状态转换，含 `depends on` 约束、`select` 强制、`choice` 互斥的交互规则：

```mermaid
stateDiagram-v2
    [*] --> UNSET: 配置项未被任何 Kconfig 定义引用

    UNSET --> y: default y 或 user 选择 y（编译进内核）
    UNSET --> m: default m 或 user 选择 m（编译为模块）
    UNSET --> n: 未选或显式 n（不编译）

    y --> n: user 取消选择（无 select 强制时）
    n --> y: user 选择（depends on 条件满足）
    y --> m: user 降级为模块（仅 tristate）
    m --> y: select X 强制为 y（依赖满足）
    m --> n: depends on 父项变为 n
    n --> n_blocked: select X 但 depends on 不满足

    n_blocked --> y: depends on 条件满足后 select 生效
    n_blocked --> n: select 强制移除或配置项删除

    note right of y
        CONFIG_XXX=y 写入 .config
        auto.conf: CONFIG_XXX=y
        autoconf.h: #define CONFIG_XXX 1
        obj-$(CONFIG_XXX) 展开为 obj-y
    end note

    note right of m
        CONFIG_XXX=m 写入 .config
        auto.conf: CONFIG_XXX=m
        autoconf.h: #define CONFIG_XXX 1（C 侧不区分 y/m）
        obj-$(CONFIG_XXX) 展开为 obj-m，产物为 .ko
    end note

    note right of n_blocked
        select 强制但 depends on 未满足
        Kconfig 发出警告（配置不一致风险）
        需修正 select 的依赖声明（OS-KER-023）
    end note

    note right of UNSET
        choice 块内：选中一个自动取消其他
        互斥组内仅一个为 y，其余为 n
        default 指定默认选中项
    end note
```

**状态转换条件**：

| 从状态 | 到状态 | 触发条件 | 系统行为 |
|--------|--------|---------|---------|
| — | UNSET | 配置项未被任何 `Kconfig` `config` 声明引用 | 不参与求解，`.config` 中无对应行 |
| UNSET | y | `default y` 或 user 在 `menuconfig` 选择 `y` | `CONFIG_XXX=y` 写入 `.config`，`obj-y += xxx.o` |
| UNSET | m | `default m` 或 user 选择 `m`（仅 `tristate`） | `CONFIG_XXX=m` 写入 `.config`，`obj-m += xxx.o`，产物 `.ko` |
| UNSET | n | 未选或显式 `n`（`default n` 或省略） | `# CONFIG_XXX is not set` 写入 `.config`，不参与构建 |
| y | n | user 取消选择（无 `select` 强制时） | `obj-y` 移除 `xxx.o`，C 侧 `#ifdef CONFIG_XXX` 失效 |
| n | y | user 选择 `y`（`depends on` 条件满足） | `obj-y += xxx.o`，`autoconf.h` 定义宏 |
| y | m | user 降级为模块（仅 `tristate` 类型） | `obj-y` → `obj-m`，产物从 `built-in.a` 变为 `.ko` |
| m | y | 被 `select X` 强制为 `y`（依赖满足） | `obj-m` → `obj-y`，强制内建 |
| m | n | `depends on` 父项变为 `n` | 模块不可用，`CONFIG_XXX` 不定义 |
| n | n_blocked | `select X` 强制但 `depends on` 不满足 | Kconfig 警告配置不一致，`OS-KER-023` 要求修正 |
| n_blocked | y | `depends on` 条件满足后 `select` 生效 | 强制选中，`obj-y += xxx.o` |
| n_blocked | n | `select` 强制移除或配置项删除 | 恢复为 `n`，不再被强制 |

---

## 3. CONFIG_* 宏与 obj-$(CONFIG_*) 门控

Kconfig 求解的 `CONFIG_*` 宏有两套消费者：Make 侧（`obj-$(CONFIG_*)`）与 C 侧（`#ifdef CONFIG_*`）。

### 3.1 Make 侧门控

`obj-$(CONFIG_*)` 是 Kbuild 的配置门控惯用法。当 `CONFIG_FOO=y` 时展开为 `obj-y += foo.o`（内建）；`CONFIG_FOO=m` 时展开为 `obj-m += foo.o`（模块）；`CONFIG_FOO=n`（或未定义）时展开为 `obj- += foo.o`，被 Make 视为空（不构建）：

```makefile
# drivers/airymax/Makefile
obj-$(CONFIG_AIRY_IPC)        += airymax-ipc.o
obj-$(CONFIG_AIRY_CUPOLAS)    += cupolas/
obj-$(CONFIG_AIRY_MEMPOOL)    += airymax-mempool.o
airymax-mempool-y += mempool_core.o
airymax-mempool-y += mempool_region.o
```

`auto.conf`（由 syncconfig 生成）被 `scripts/Makefile.build` 在每个子目录 `-include`，提供 `CONFIG_*` 的 make 侧值。这就是"配置即构建"的闭合回路：Kconfig 求解 → `auto.conf` → `obj-$(CONFIG_*)` → 三态门控。

- **OS-BUILD-022**：子系统 `obj-*` 必须用 `obj-$(CONFIG_*)` 引用，`CONFIG_*` 名字与 Kconfig 中 `config` 名字严格一致；拼写错误会导致选项永远不构建且无警告。
- **OS-KER-025**：`obj-$(CONFIG_*)` 的 `CONFIG_*` 必须在 Kconfig 中有 `config` 声明；引用未声明宏视为配置缺失，CI 必须报错。

### 3.2 C 侧门控

```c
/* drivers/airymax/ipc/ipc.c */
#include <linux/config.h>  /* 实际由 autoconf.h 提供 */

#ifdef CONFIG_AIRY_IPC_BENCH
static atomic_t ipc_rx_counter;
#endif

#if CONFIG_AIRY_IPC_RINGBUF_ORDER > 4
/* large ring buffer path */
#endif
```

`autoconf.h`（由 syncconfig 生成）把每个 `CONFIG_*=y` 定义为 `#define CONFIG_XXX 1`，`CONFIG_*=m` 同样定义为 `1`（C 侧不区分 y/m，只关心是否启用），`CONFIG_*=n` 不定义。`int`/`hex` 类型则定义为字面值。

- **OS-STD-147**（沿用 50 卷）：C 源码引用 `CONFIG_*` 必须经 `#ifdef`/`#if`，禁止直接 `#include <generated/autoconf.h>` 后裸用宏值；`config.h` 已统一包装。

---

## 4. Kconfig 子目录组织

Linux 6.6 用顶层 `Kconfig` 文件 `source` 各子系统的 `Kconfig`，形成一棵 Kconfig 源文件树：

```
# Kconfig（顶层）
mainmenu "Linux/$(ARCH) $(KERNELVERSION) Kernel Configuration"
source "scripts/Kconfig.include"
source "init/Kconfig"
source "kernel/Kconfig.freezer"
source "fs/Kconfig.binfmt"
source "mm/Kconfig"
source "net/Kconfig"
source "drivers/Kconfig"
source "fs/Kconfig"
source "security/Kconfig"
source "crypto/Kconfig"
source "lib/Kconfig"
source "lib/Kconfig.debug"
source "lib/Kconfig.airymaxos"
source "Documentation/Kconfig"
```

每个子系统目录有自己的 `Kconfig`，被父目录的 `Kconfig` 用 `source "path/Kconfig"` 引入。`drivers/Kconfig` 再 `source` 各 `drivers/*/Kconfig`，形成递归结构。

- **OS-KER-026**：agentrt-linux 新增子系统的 `Kconfig` 必须被父目录 `Kconfig` `source` 引入，未 `source` 的 `Kconfig` 不参与求解（对齐 S-2 层次分解）。
- **OS-BUILD-023**：`source` 路径使用相对源码树根的相对路径（如 `"drivers/airymax/Kconfig"`），禁止绝对路径，以保证 out-of-tree 构建正确。

---

## 5. Kconfig.airymaxos 供应商扩展

agentrt-linux 用 `lib/Kconfig.airymaxos` 取代上游供应商配置聚合文件，作为 agentrt-linux 自身特性的通用配置聚合点。子系统特定配置仍嵌入各原生 `Kconfig`（如 `drivers/airymax/Kconfig`），`Kconfig.airymaxos` 仅聚合跨子系统的供应商级特性。

### 5.1 Kconfig.airymaxos 结构

```
# SPDX-License-Identifier: GPL-2.0
# lib/Kconfig.airymaxos —— agentrt-linux 供应商配置聚合

menu "agentrt-linux vendor extensions"

config AIRYMAXOS_LTS
	int
	default 0
	help
	  agentrt-linux LTS track number. Used by the build to tag
	  release artifacts.

config AIRYMAXOS_KWORKER_NUMA_AFFINITY
	bool "kworker NUMA affinity"
	default n
	depends on NUMA
	help
	  This feature implements adaptive mechanisms so that the
	  workqueue can automatically identify the CPU of the soft
	  interrupt and schedule the workqueue to the corresponding
	  NUMA node.

config AIRYMAXOS_GMEM
	bool "agentrt-linux group memory (GMEM)"
	depends on MMU
	select SHMEM
	help
	  Enable group memory accounting for agent workloads.

endmenu # agentrt-linux vendor extensions
```

### 5.2 与原生 Kconfig 的边界

```mermaid
flowchart TD
    TOP[顶层 Kconfig] --> SRC1[init/Kconfig]
    TOP --> SRC2[mm/Kconfig]
    TOP --> SRC3[drivers/Kconfig]
    TOP --> SRC4[lib/Kconfig]
    TOP --> SRC5[lib/Kconfig.debug]
    TOP --> SRC6[lib/Kconfig.airymaxos]
    SRC6 --> V1[agentrt-linux 跨子系统特性]
    SRC3 --> D1[drivers/airymax/Kconfig]
    D1 --> D2[子系统特定特性]
    V1 -.聚合.-> VEND[供应商级通用配置]
    D2 -.嵌入.-> SUB[子系统原生配置]
```

**边界原则**：`Kconfig.airymaxos` 只放跨子系统的供应商级特性（如 `AIRYMAXOS_LTS`、通用 NUMA 亲和、GMEM 等会影响多子系统的开关）；子系统特定特性（如 IPC 环形缓冲大小、Cupolas 策略细节）放在子系统的 `Kconfig`。这符合 S-1（单一职责）与 S-2（层次分解）。

- **OS-KER-027**：`lib/Kconfig.airymaxos` 仅承载 agentrt-linux 跨子系统的供应商级特性；子系统特定配置嵌入该子系统的 `Kconfig`，禁止把子系统细节塞入聚合文件。
- **OS-BUILD-024**：`Kconfig.airymaxos` 的 `config` 名字必须以 `AIRYMAXOS_` 前缀开头，与子系统前缀（如 `AIRY_IPC_`）区分，便于在 `.config` 中快速定位归属。
- **OS-STD-148**（沿用 50 卷）：`Kconfig.airymaxos` 中的 `DEBUG` 类选项必须同时声明对应的测试钩子，确保调试开关与测试同步（对齐 OS-STD-148）。

---

## 6. 配置工具 menuconfig / nconfig / gconfig

Kconfig 提供多种配置工具，agentrt-linux 沿用 **Linux 6.6 内核基线** 的工具集：

| 工具 | 界面 | 依赖 | 适用场景 |
|------|------|------|---------|
| `menuconfig` | ncurses 终端 | libncurses | 最常用，SSH/无图形环境 |
| `nconfig` | ncurses 终端（现代） | libncurses | menuconfig 的现代替代 |
| `gconfig` | GTK 图形 | libglade/libgtk | 桌面图形环境 |
| `xconfig` | Qt 图形 | libqt | 桌面图形环境 |
| `oldconfig` | 命令行交互 | 无 | 增量更新 `.config` |
| `defconfig` | 无（读默认） | 无 | 从 arch defconfig 生成 |
| `allmodconfig` | 无 | 无 | 全部设为 `m`（最大覆盖） |
| `allnoconfig` | 无 | 无 | 全部设为 `n`（最小内核） |

```bash
make menuconfig       # 交互式终端菜单
make nconfig          # 现代 ncurses 菜单
make gconfig          # GTK 图形界面
make olddefconfig     # 静默应用默认值，解决新增选项
make defconfig        # 生成架构默认配置
make allmodconfig     # 全模块配置（CI 用）
make allnoconfig      # 全关闭配置（CI 用）
```

- **OS-BUILD-025**：开发者修改 `Kconfig` 后必须本地运行 `make olddefconfig` 验证新增选项能被正确求解，禁止直接提交未求解的 `.config` 片段。
- **OS-STD-132**（沿用 50 卷）：CI 矩阵必须覆盖 `allmodconfig`/`allnoconfig`/`defconfig` 三种配置，确保 Kconfig 在极端配置下也能闭合求解（对齐 OS-STD-032 CI 矩阵）。
- **OS-KER-028**：新增 `choice` 或 `menuconfig` 后必须用 `menuconfig`/`nconfig` 实际打开验证菜单结构正确，禁止仅靠 `make olddefconfig` 推断结构（对齐 A-2 细节关注）。

---

## 7. KCONFIG_ALLCONFIG 全配置覆盖

`KCONFIG_ALLCONFIG` 是 Kconfig 的"全配置基线"机制：当运行 `allmodconfig`/`allnoconfig`/`defconfig` 时，Kconfig 先读取 `KCONFIG_ALLCONFIG` 指定的文件作为基线，再应用"全部设为 m/n"策略。这让 CI 能在固定一组"必须启用"的选项之上做最大覆盖测试。

```bash
# 使用 agentrt-linux 全配置基线
make allmodconfig KCONFIG_ALLCONFIG=airymaxos-base.config
make allnoconfig KCONFIG_ALLCONFIG=airymaxos-base.config
```

`airymaxos-base.config` 是一个 `.config` 片段，列出 agentrt-linux 必须启用的选项（如 `CONFIG_AIRY_IPC=y`、`CONFIG_IO_URING=y`）。这样 `allnoconfig` 也会保留这些必选项，避免"最小内核"把 agentrt-linux 核心特性关掉。

- **OS-BUILD-026**：agentrt-linux 必须维护 `airymaxos-base.config` 全配置基线文件，列出 OS 核心必须启用的 `CONFIG_*`；CI 的 `allmodconfig`/`allnoconfig` 必须带 `KCONFIG_ALLCONFIG` 引用此文件。
- **OS-KER-029**：`airymaxos-base.config` 的变更必须经构建系统评审，新增/移除项需说明理由；该文件是"agentrt-linux 最小可用配置"的契约（对齐 S-2 层次分解与 K-4 可插拔策略）。
- **OS-BUILD-027**：CI 必须验证 `make allmodconfig KCONFIG_ALLCONFIG=airymaxos-base.config` 与 `make allnoconfig KCONFIG_ALLCONFIG=airymaxos-base.config` 都能成功求解（无 unsatisfied dependence 警告）。

### 7.1 defconfig 与 allconfig 的关系

`arch/$(SRCARCH)/configs/airy_defconfig` 是架构默认配置，`make defconfig` 读取它生成 `.config`。`KCONFIG_ALLCONFIG` 与 `defconfig` 的区别：`defconfig` 是"用户起点配置"，`KCONFIG_ALLCONFIG` 是"全配置策略的保底基线"。二者配合：CI 用 `KCONFIG_ALLCONFIG` 保证极端配置下核心特性不丢，开发者用 `defconfig` 作为日常起点。

- **OS-BUILD-028**：每个支持架构（x86_64/aarch64/riscv64）必须有 `airy_defconfig`；`defconfig` 与 `airymaxos-base.config` 必须一致——`defconfig` 中 `=y` 的项必须出现在 `airymaxos-base.config`，反之亦然。

---

## 8. 五维原则映射

本卷 Kconfig 配置系统与 Airymax **五维正交 24 原则** 的映射如下：

| 原则 | 在本卷的体现 |
|------|-------------|
| **S-1 单一职责** | `Kconfig.airymaxos` 仅聚合跨子系统特性；子系统细节嵌入子系统 `Kconfig`；配置与构建职责分离 |
| **S-2 层次分解** | 顶层 `Kconfig` `source` 子系统 `Kconfig`，按目录层次组织；`menuconfig`/`if` 分组相关选项 |
| **K-2 契约一致** | `CONFIG_*` 是配置与构建的契约；`obj-$(CONFIG_*)` 与 C 侧 `#ifdef` 双消费者共享同一宏定义 |
| **K-4 可插拔策略** | `tristate`/`bool` 使特性可插拔；`obj-y`/`obj-m`/`obj-n` 三态门控实现策略与机制分离 |
| **E-7 文档即代码** | Kconfig `help` 文本即文档；`menuconfig` 菜单结构即配置文档；`make help` 列出可用目标 |
| **A-1 极简主义** | `select` 仅用于强约束；弱关联用 `depends on`；`choice` 保证互斥唯一性 |
| **A-2 细节关注** | `config` 名字前缀规范；`obj-$(CONFIG_*)` 拼写必须严格一致；`defconfig` 与 `base.config` 一致性 |

agentrt-linux 配置系统以 **Linux 6.6 内核基线** Kconfig 工程为来源，但每一项机制都经过 **五维正交 24 原则** 的映射校验。例如 `tristate` 三态门控同时映射 K-4（可插拔策略）与 S-1（单一职责：一个选项一种状态），`select` 的强约束同时映射 A-1（极简主义：避免配置悖论）与 A-4（完美主义：依赖闭包完备）。这种正交映射确保配置决策有原则可循。

---

## 9. 同源 agentrt 映射

agentrt-linux 配置系统与 agentrt 配置系统遵循 **IRON-9 同源且部分代码共享（IRON-9 v3）** 原则。agentrt 是 Airymax 智能体运行时（应用层），其配置通过 CMake 缓存（`CMakeCache.txt`）与 `-D<option>=ON/OFF` 选项描述；agentrt-linux 是智能体操作系统（OS 层），内核态配置通过 Kconfig 描述。二者在配置语义上同源（配置驱动构建），在描述手段上独立（Kconfig vs CMake）。

| 维度 | agentrt | agentrt-linux 内核态 | 关系 |
|------|---------|------------------|------|
| 配置语言 | CMake `-D` 选项 | Kconfig `config` 语法 | 同源语义（配置开关），独立语言 |
| 三态模型 | ON/OFF（二态） | y/m/n（三态：内建/模块/关闭） | agentrt-linux 扩展（模块化能力） |
| 依赖求解 | CMake `find_dependency` | Kconfig `depends on`/`select` | 同源思想（依赖闭包），独立机制 |
| 配置基线 | CMake preset 文件 | `airymaxos-base.config` + `defconfig` | 同源语义（保底基线），独立格式 |
| 配置工具 | `ccmake`（终端）/`cmake-gui` | `menuconfig`/`nconfig`/`gconfig` | 同源交互形态，独立实现 |
| 供应商扩展 | CMake `option()` 聚合 | `lib/Kconfig.airymaxos` 聚合 | 同源聚合点，独立文件 |

**同源红利**：agentrt 在 agentrt-linux 上运行时，配置语义对齐——agentrt 的 `AIRY_*` 选项与 agentrt-linux 的 `CONFIG_AIRY_*` 在概念上一一对应，便于联调时确认特性开关状态；`ccmake` 与 `menuconfig` 的交互形态相似，开发者心智模型可复用。

**独立性**：agentrt-linux 内核态配置为 OS 层 Kconfig（求解 `CONFIG_*` 三态宏），agentrt 用户态配置为应用层 CMake（求解 CMake 变量），二者通过产物契约解耦。当 agentrt 配置工具演进时，agentrt-linux 通过配置评审决定是否同步，避免被动跟随。这种"同源思想 + 独立实现"的双层结构，正是 **IRON-9 同源且部分代码共享（IRON-9 v3）** 在配置系统层的落实——同源语义，独立工具链。

agentrt-linux 配置系统在 **Linux 6.6 内核基线** 上构建，其 `config`/`menuconfig`/`choice`/`depends on`/`select` 语法、`obj-$(CONFIG_*)` 门控、`syncconfig` 产物链均直接源自上游沉淀；agentrt-linux 的扩展（`lib/Kconfig.airymaxos`、`airymaxos-base.config`、`AIRYMAXOS_`/`AIRY_` 命名前缀）以 **五维正交 24 原则** 为设计准绳，确保扩展不破坏上游 Kconfig 的依赖求解可预测性。这一双层结构与构建系统卷（01）的"上游 Kbuild 思想 + agentrt-linux 供应商扩展"形成对称，共同构成 agentrt-linux 内核态构建的配置与执行双轮。

### 9.1 IRON-9 v3 四层共享模型

本节将上节"同源 agentrt 映射"进一步细化为 **IRON-9 v3 四层共享模型**，明确配置系统层在用户态（agentrt）与内核态（agentrt-linux）之间的代码共享边界。三层分别为：**[SC] 共享契约层**（共享头文件 / 数据结构定义）、**[SS] 语义同源层**（设计模式同源但实现独立）、**[IND] 完全独立层**（双方各自独立实现）。该模型由 10 个 [SC] 头文件契约、跨态语义对照表与独立实现清单共同支撑。

#### 9.1.1 三层模型概览表

| 层次 | 共享程度 | 配置系统层内容 |
|------|---------|---------------|
| **[SC] 共享契约层** | 无 | 无 [SC] 层——配置系统为 agentrt-linux 内核态专属，不与 agentrt 用户态共享代码 |
| **[SS] 语义同源层** | 设计模式同源 | `CONFIG_AIRY_*` 命名约定、`y/m/n` 三态选择逻辑、`depends on`/`select` 关联、`choice` 互斥——与 agentrt `airy_config_t` 在"声明式配置 + 依赖关系"语义上同源 |
| **[IND] 完全独立层** | 完全独立 | Kconfig 解析器（`scripts/kconfig/`）、`defconfig` 基线、`savedefconfig` 精简、`CONFIG_` 宏展开为 cpp 宏 |

#### 9.1.2 [SC] 共享契约层

**无直接 [SC] 共享头文件**。

配置系统层不属于 IRON-9 v3 的 10 个 [SC] 共享头文件清单（`syscalls.h` / `memory_types.h` / `security_types.h` / `cognition_types.h` / `sched.h` / `ipc.h`）。配置系统是编译期/运行期基础设施，其产物（`CONFIG_*` 宏 / CMake 变量）通过宏展开与变量传递解耦，而源码层无共享头文件依赖。这一约束确保 agentrt 用户态配置参数演进时不会被动牵连 agentrt-linux Kconfig，反之亦然——配置系统层的演进由各自的 **OS-KER 配置评审** 独立裁决。

#### 9.1.3 [SS] 语义同源层

| 语义维度 | agentrt 用户态（airy_config_t / CMake） | agentrt-linux 内核态（Kconfig） | 同源语义 |
|---------|-------------------------------------------|-------------------------------|----------|
| 配置命名 | `AIRY_*` 运行时参数键 | `CONFIG_AIRY_*` 编译期宏 | 声明式配置项命名约定 |
| 状态选择 | ON/OFF（二态） | y/m/n（三态：内建/模块/关闭） | 离散状态选择逻辑 |
| 依赖关联 | CMake `find_dependency` / 选项约束 | `depends on` / `select` 强弱关联 | 声明式依赖关系 |
| 互斥决策 | CMake 选项枚举 | `choice` 块互斥唯一 | 互斥选择语义 |
| 配置基线 | CMake preset / 默认值 | `airymaxos-base.config` + `defconfig` | 保底基线配置 |
| 配置工具 | `ccmake` / `cmake-gui` | `menuconfig` / `nconfig` / `gconfig` | 交互式配置 UI |
| 供应商聚合 | CMake `option()` 聚合点 | `lib/Kconfig.airymaxos` 聚合 | 跨特性配置聚合 |

**语义说明**：agentrt 用户态的运行时配置参数（`airy_config_t`）与 agentrt-linux 内核态的 Kconfig 在"声明式配置 + 依赖关系"这一核心语义上同源——二者均通过**声明式描述**（CMake `-D` 选项 / Kconfig `config` 块）定义配置项及其依赖关系，由底层求解器（CMake 变量求解 / Kconfig 依赖闭包求解）决定最终生效值。这种同源使配置项在两端的概念可一一对应：`AIRY_*` 与 `CONFIG_AIRY_*` 在概念上对齐，便于联调时确认特性开关状态。但**机制完全独立**——Kconfig 是编译期求解生成 cpp 宏，airy_config_t 是运行期求解生成结构体字段，二者无代码共享。

#### 9.1.4 [IND] 完全独立层

| 独立实现项 | agentrt-linux 内核态 | agentrt 用户态 | 独立原因 |
|-----------|---------------------|---------------|---------|
| Kconfig 解析器 | `scripts/kconfig/`（kconfig-mconf/gconf） | 无对应（CMake 内置） | Kconfig 语言解析特有 |
| `defconfig` 基线 | `arch/*/configs/*_defconfig` | CMake preset 文件 | 基线格式与机制不同 |
| `savedefconfig` 精简 | `make savedefconfig` 最小化 | 无对应 | Kconfig 精简机制特有 |
| `CONFIG_` 宏展开 | `CONFIG_*` → cpp 宏（`#ifdef`） | CMake 变量 → 编译定义 | 编译期宏 vs 运行期变量 |
| 三态门控 | y/m/n → `obj-y/m/n` | ON/OFF → 二态 | 内核模块化能力特有 |
| 配置产物链 | `.config` → `auto.conf` → `autoconf.h` | `CMakeCache.txt` → 编译命令 | 产物链与触发点不同 |

#### 9.1.5 跨态协作流

```mermaid
graph LR
    subgraph "agentrt 用户态（应用层）"
        A1["airy_config_t<br/>运行时配置参数"]
        A2["CMake 选项求解<br/>ON/OFF 二态"]
        A3["运行期生效<br/>结构体字段"]
    end
    subgraph "agentrt-linux 内核态（OS 层）"
        K1["Kconfig config 块<br/>声明式配置"]
        K2["依赖闭包求解<br/>y/m/n 三态"]
        K3["编译期生效<br/>CONFIG_ cpp 宏"]
    end
    SS["[SS] 语义同源层<br/>声明式配置 + 依赖关系<br/>命名约定 / 状态选择 / 互斥"]
    IND["[IND] 完全独立层<br/>解析器 / 产物链<br/>三态 vs 二态"]

    A1 -->|同源语义| SS
    K1 -->|同源语义| SS
    SS --> A2
    SS --> K2
    A2 --> A3
    K2 --> K3
    A1 -.->|独立实现| IND
    K1 -.->|独立实现| IND
    A3 -.->|独立生效| K3

    style SS fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000
    style IND fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    style A1 fill:#e3f2fd,stroke:#1565c0,color:#000
    style A2 fill:#e3f2fd,stroke:#1565c0,color:#000
    style A3 fill:#e3f2fd,stroke:#1565c0,color:#000
    style K1 fill:#fce4ec,stroke:#c62828,color:#000
    style K2 fill:#fce4ec,stroke:#c62828,color:#000
    style K3 fill:#fce4ec,stroke:#c62828,color:#000
```

**协作说明**：agentrt 用户态通过 `airy_config_t` 描述运行时配置参数，agentrt-linux 内核态通过 Kconfig 描述编译期配置。两端在 **[SS] 语义同源层** 共享"声明式配置 + 依赖关系"的设计模式（命名约定 / 状态选择 / 依赖关联 / 互斥决策），使 `AIRY_*` 与 `CONFIG_AIRY_*` 在概念上对齐，便于联调时确认特性开关状态；但在 **[IND] 完全独立层** 各自维护解析器、产物链、状态模型（三态 vs 二态）。最终两端配置**独立生效**：agentrt-linux 的 `CONFIG_` cpp 宏在编译期决定内核特性，agentrt 的运行期配置参数在运行时决定应用行为，二者无源码层依赖，无 [SC] 共享头文件。这正是 **IRON-9 v3 同源且部分代码共享** 在配置系统层的落地——同源语义，独立工具链，无共享契约。

---

## 10. OS-KER / OS-STD / OS-BUILD 规则编号汇总

本卷定义的规则编号汇总如下，与 07 卷"规则编号注册表"对齐：

| 编号 | 规则 | 强制级别 |
|------|------|---------|
| OS-KER-155 | config 名字用 AIRY_/子系统前缀 | MUST |
| OS-KER-022 | 互斥决策用 choice | MUST |
| OS-KER-023 | select 目标的依赖须同时满足 | MUST |
| OS-KER-024 | 可模块化子系统用 tristate | MUST |
| OS-KER-025 | obj-$(CONFIG_*) 的宏须有 config 声明 | MUST |
| OS-KER-026 | 新增 Kconfig 须被父 Kconfig source | MUST |
| OS-KER-027 | Kconfig.airymaxos 仅聚合跨子系统特性 | MUST |
| OS-KER-028 | 新 choice/menuconfig 须用工具实际验证 | MUST |
| OS-KER-029 | airymaxos-base.config 变更经评审 | MUST |
| OS-KER-030 | 内核态配置以 Kconfig 为唯一手段 | MUST |
| OS-STD-143 | 新 CONFIG 选项默认 off（沿用） | MUST |
| OS-STD-144 | 新 CONFIG 选项有 help 文本（沿用） | MUST |
| OS-STD-145 | 相关选项 menuconfig+if 分组（沿用） | MUST |
| OS-STD-146 | select 仅用于强约束（沿用） | MUST |
| OS-STD-147 | C 侧经 #ifdef 引用 CONFIG（沿用） | MUST |
| OS-STD-148 | DEBUG 选项同时声明测试（沿用） | MUST |
| OS-BUILD-021 | CONFIG_* 须由 Kconfig 求解入 auto.conf | MUST |
| OS-BUILD-022 | obj-$(CONFIG_*) 名字严格一致 | MUST |
| OS-BUILD-023 | source 用相对源码树路径 | MUST |
| OS-BUILD-024 | Kconfig.airymaxos 用 AIRYMAXOS_ 前缀 | MUST |
| OS-BUILD-025 | 改 Kconfig 后本地 olddefconfig 验证 | MUST |
| OS-BUILD-026 | 维护 airymaxos-base.config 基线 | MUST |
| OS-BUILD-027 | CI 验证 allmod/allno+ALLCONFIG 求解 | MUST |
| OS-BUILD-028 | 每架构有 defconfig 且与 base.config 一致 | MUST |

---

## 11. 相关文档

### 11.1 同卷文档

- `README.md`（构建系统模块主索引）
- `01-kbuild-system.md`（Kbuild 递归构建——obj-$(CONFIG_*) 的消费者）
- `03-makefile-patterns.md`（Makefile 引用 CONFIG_* 的惯用法）
- `04-module-building.md`（CONFIG_*=m 驱动的模块构建）
- `06-airymaxos-build.md`（agentrt-linux 多仓配置协调）

### 11.2 上游与跨卷文档

- `50-engineering-standards/06-toolchain-and-automation.md`（OS-STD-032 CI 矩阵、OS-STD-143/144/145/146/147/148 Kconfig 规则来源）
- `50-engineering-standards/04-engineering-philosophy.md`（IRON-9 同源且部分代码共享（IRON-9 v3）原则定义）
- `10-architecture/02-five-dimensional-principles.md`（五维正交 24 原则定义）
- `20-modules/01-kernel.md`（kernel 子仓配置）

### 11.3 参考材料

- `Linux 6.6 内核源码 Kconfig`（顶层 Kconfig 源文件树）
- `Linux 6.6 内核源码 init/Kconfig`（config/menuconfig/choice 语法范例）
- `Linux 6.6 内核源码 lib/Kconfig`（lib 聚合 Kconfig）
- `Linux 6.6 内核源码 lib/Kconfig.debug`（调试选项聚合范例）
- `Linux 6.6 内核源码 scripts/kconfig/`（Kconfig 解析器源码）

---

## 12. 文档版本与维护

- **当前版本**: v0.1.1/ v1.0.1（开发中）
- **维护者**: agentrt-linux 构建系统 SIG（待成立，详见 07 卷维护者制度）
- **变更流程**: 本卷变更必须经过 RFC → 评审 → ACC 验收流程；Kconfig 语法层变更需同步评估对 `airymaxos-base.config` 与各架构 `defconfig` 的影响。
- **回顾周期**: 随 Linux 6.6 内核基线 LTS 更新季度回顾 + agentrt-linux 大版本年度回顾。
- **0.1.1 范围**: README + 01 + 02（3 文档）。
- **1.0.1 范围**: 完成全部 8 文档并实施构建系统工程标准。

---

## 附录 A: 接口定义

> **附录定位**： 本附录汇集 Kconfig 配置系统所需的完整接口契约，供直接参照实现。所有数据结构与函数签名对齐 Linux 6.6 `scripts/kconfig/`（解析器与配置工具）、`init/Kconfig`、`lib/Kconfig.airymaxos` 及 `include/airymax/kconfig_types.h`（agentrt-linux 内核内部头文件，[IND] 独立层，agentrt 用户态不共享）。A.1 以 C 结构体建模 Kconfig 符号/菜单的内部表示，A.2 以解析器入口函数为主，A.3 给出 `CONFIG_AIRY_*` 完整清单与三态语义。

### A.1 核心数据结构

#### A.1.1 kconfig_symbol — Kconfig 符号

```c
/**
 * struct kconfig_symbol - Kconfig 单个配置符号（config 声明）
 *
 * @name:        符号名（如 "AIRY_IPC"、"FUNCTION_TRACER"）
 * @type:        符号类型（S_BOOLEAN / S_TRISTATE / S_INT / S_HEX / S_STRING）
 * @curr:        当前求解值（tristate: yes/mod/no；bool: yes/no）
 * @visible:     菜单可见性（受 depends on / prompt 条件约束）
 * @flags:       标志位（SYMBOL_WRITE / SYMBOL_CHOICE / SYMBOL_AUTO 等）
 * @deps:        依赖链表头（depends on 表达式）
 * @rev_dep:     反向依赖链表头（被 select 的目标）
 * @prompt:      菜单提示文本（config 的 "标题"）
 * @help:        help 文本指针
 * @dir:         所属目录（source 路径）
 *
 * 对齐 Linux 6.6 scripts/kconfig/lkc.h 的 struct symbol
 */
enum symbol_type {
    S_UNKNOWN  = 0,
    S_BOOLEAN   = 1,
    S_TRISTATE  = 2,
    S_INT       = 3,
    S_HEX       = 4,
    S_STRING    = 5,
};

#define SYMBOL_WRITE  0x0001  /* 已写入（求解完成） */
#define SYMBOL_CHOICE 0x0002  /* 属于 choice 互斥组 */
#define SYMBOL_AUTO   0x0004  /* 自动选中（被 select） */

struct kconfig_symbol {
    const char           *name;
    enum symbol_type      type;
    tristate              curr;
    tristate              visible;
    unsigned long         flags;
    struct expr          *deps;
    struct expr          *rev_dep;
    const char           *prompt;
    const char           *help;
    const char           *dir;
};
```

#### A.1.2 kconfig_entry — 配置条目

```c
/**
 * struct kconfig_entry - Kconfig 文件中一条声明（config/menuconfig/...）
 *
 * @kind:       条目类型（ENTRY_CONFIG / ENTRY_MENUCONFIG / ENTRY_CHOICE /
 *              ENTRY_MENU / ENTRY_SOURCE / ENTRY_COMMENT）
 * @sym:        关联符号（config/menuconfig 非空，source/comment 为 NULL）
 * @prompt:     提示文本
 * @deps:       depends on 表达式
 * @children:   子条目链表头（menu 内嵌条目）
 * @next:       同级下一条目
 * @file:       来源 Kconfig 文件路径
 * @lineno:     行号（用于诊断与回溯）
 *
 * 对齐 Linux 6.6 scripts/kconfig/menu.c 的 struct menu
 */
enum entry_kind {
    ENTRY_CONFIG      = 0,
    ENTRY_MENUCONFIG  = 1,
    ENTRY_CHOICE      = 2,
    ENTRY_MENU        = 3,
    ENTRY_SOURCE      = 4,
    ENTRY_COMMENT     = 5,
};

struct kconfig_entry {
    enum entry_kind       kind;
    struct kconfig_symbol *sym;
    const char            *prompt;
    struct expr           *deps;
    struct kconfig_entry  *children;
    struct kconfig_entry  *next;
    const char            *file;
    int                    lineno;
};
```

#### A.1.3 choice_entry — choice 互斥组

```c
/**
 * struct choice_entry - Kconfig choice 互斥选项组
 *
 * @sym:        关联符号（类型为 S_BOOLEAN 或 S_TRISTATE）
 * @entries:    组内 config 条目链表头（互斥候选）
 * @is_bool:    true 为 bool 组（单选 yes/no）；false 为 tristate 组
 * @is_optional: 是否 optional（允许全不选）
 * @selected:   当前选中的候选符号（互斥组内仅一个被选中）
 *
 * 对齐 Linux 6.6 scripts/kconfig 的 choice 语义
 * （典型用法：调度策略二选一、编译器选择）
 */
struct choice_entry {
    struct kconfig_symbol  *sym;
    struct kconfig_entry   *entries;
    bool                     is_bool;
    bool                     is_optional;
    struct kconfig_symbol  *selected;
};
```

#### A.1.4 menu_node — menuconfig 菜单树节点

```c
/**
 * struct menu_node - menuconfig 菜单树节点
 *
 * @entry:      对应的 kconfig_entry（config/menu/choice/comment）
 * @parent:     父节点（根节点为 NULL）
 * @child:      首个子节点
 * @sibling:    下一个兄弟节点
 * @depth:      菜单层级（根为 0，用于缩进渲染）
 * @is_visible: 渲染时是否可见（受 depends on 求值影响）
 *
 * 对齐 Linux 6.6 scripts/kconfig/mconf.c / nconf.c 的菜单树遍历模型
 * （menuconfig/gconfig/nconfig 共享同一棵 menu_node 树）
 */
struct menu_node {
    struct kconfig_entry  *entry;
    struct menu_node      *parent;
    struct menu_node      *child;
    struct menu_node      *sibling;
    int                    depth;
    bool                   is_visible;
};
```

### A.2 核心函数签名

#### A.2.1 kconfig_parse — 解析 Kconfig 文件

```c
/**
 * kconfig_parse - 解析 Kconfig 源文件树为符号表与菜单树
 * @root_file:  顶层 Kconfig 文件路径（如 "Kconfig"）
 * @arch:       目标架构（影响默认值与 defconfig 路径）
 *
 * 递归 source 各子 Kconfig，构建 kconfig_symbol 表与 menu_node 树；
 * 求解 depends on / select 反向依赖；填充 visible 与 prompt。
 *
 * 返回: 0 成功；<0 失败（负 errno）
 *   -ENOENT: Kconfig 文件不存在
 *   -EINVAL: 语法错误（参见 stderr 输出）
 *   -ELOOP:  循环 source 引用
 *
 * 对齐 Linux 6.6 scripts/kconfig/zconf.y（yacc 解析器入口）
 */
int kconfig_parse(const char *root_file, const char *arch);
```

#### A.2.2 config_validate — 配置校验

```c
/**
 * config_validate - 校验 .config 与 Kconfig 符号表的一致性
 * @symbols:  解析得到的符号表
 * @config:   .config 文件路径
 *
 * 检查每个 CONFIG_* 是否在 Kconfig 中有对应 config 声明；
 * 检查 depends on 是否全部满足；报告 unsatisfied dependence 警告。
 *
 * 返回: 0 校验通过；<0 失败
 *   -EINVAL: 存在未声明 CONFIG_* 或依赖不满足
 *
 * 对齐 Linux 6.6 `make olddefconfig` 与 syncconfig 校验逻辑
 */
int config_validate(struct kconfig_symbol *symbols, const char *config);
```

#### A.2.3 menuconfig_main — menuconfig 入口

```c
/**
 * menuconfig_main - menuconfig 交互式配置工具入口
 * @root_file: 顶层 Kconfig 文件路径
 * @config:    输出 .config 文件路径
 *
 * 渲染 menu_node 菜单树，捕获键盘事件（ncurses），
 * 用户切换 y/m/n 后即时重求解依赖闭包，退出时写回 .config。
 *
 * 返回: 0 成功（用户保存退出）；<0 失败
 *
 * 对齐 Linux 6.6 scripts/kconfig/mconf.c
 */
int menuconfig_main(const char *root_file, const char *config);
```

#### A.2.4 conf_write — 写入 .config

```c
/**
 * conf_write - 将求解后的符号表写入 .config 文件
 * @symbols:  符号表
 * @config:   输出 .config 文件路径
 *
 * 输出格式：`# CONFIG_FOO is not set` 或 `CONFIG_FOO=y/m/值`；
 * 保留注释与空行以保持 diff 可读。
 *
 * 返回: 0 成功；<0 失败（-EIO 写入失败）
 *
 * 对齐 Linux 6.6 scripts/kconfig/confdata.c
 */
int conf_write(struct kconfig_symbol *symbols, const char *config);
```

#### A.2.5 conf_read — 读取 .config

```c
/**
 * conf_read - 读取 .config 并回填符号表当前值
 * @symbols:  符号表（输入/输出，curr 字段被覆盖）
 * @config:   .config 文件路径
 *
 * 解析 `CONFIG_*=y/m/值` 与 `# CONFIG_FOO is not set`，
 * 回填到对应 symbol->curr；未在 .config 出现的符号保持默认值。
 *
 * 返回: 0 成功；<0 失败（-ENOENT 文件不存在）
 *
 * 对齐 Linux 6.6 scripts/kconfig/confdata.c
 */
int conf_read(struct kconfig_symbol *symbols, const char *config);
```

### A.3 错误码与宏定义

#### A.3.1 CONFIG_* 三态值

```c
/**
 * Kconfig 三态值（tristate）
 *
 * 对齐 Linux 6.6 scripts/kconfig/expr.h
 */
#define TRISTATE_NO    0   /* n：不启用（# CONFIG_FOO is not set） */
#define TRISTATE_MOD   1   /* m：编译为模块（CONFIG_FOO=m） */
#define TRISTATE_YES   2   /* y：内建启用（CONFIG_FOO=y） */

/* bool 类型仅取 NO / YES；tristate 取 NO / MOD / YES */
```

#### A.3.2 CONFIG_AIRY_* 关键配置项清单

```c
/**
 * CONFIG_AIRY_* - agentrt-linux 子系统级配置项（嵌入子系统 Kconfig）
 *
 * 对齐 drivers/airymax/Kconfig 等；以下为关键配置项清单。
 */

/* —— AgentsIPC 内核态通道 —— */
#define CONFIG_AIRY_IPC                /* tristate: AgentsIPC 内核态通道 */
#define CONFIG_AIRY_IPC_BENCH          /* bool: 通道基准测试桩 */
#define CONFIG_AIRY_IPC_RINGBUF_ORDER  /* int: 环形缓冲阶数（默认 4） */

/* —— Cupolas 安全穹顶 —— */
#define CONFIG_AIRY_CUPOLAS           /* tristate: Cupolas LSM 框架 */
#define CONFIG_AIRY_CUPOLAS_AUDIT     /* bool: Cupolas 审计日志 */

/* —— 内存池 —— */
#define CONFIG_AIRY_MEMPOOL           /* tristate: agentrt 内存池 */

/* —— Agent 运行时 —— */
#define CONFIG_AIRY_AGENT             /* tristate: Agent 调度运行时 */
#define CONFIG_AIRY_COGNITION         /* tristate: Agent 认知子系统 */
#define CONFIG_AIRY_TOKEN_BUDGET      /* bool: Token 预算调控 */

/* —— MicroCoreRT 调度 —— */
#define CONFIG_AIRY_MICROCORE_RT      /* bool: MicroCoreRT 实时调度策略 */
#define CONFIG_AIRY_MICROCORE_RT_PRIO /* int: 默认实时优先级 */

/* —— 可观测性 —— */
#define CONFIG_AIRY_AGENT_TRACING     /* tristate: Agent 行为追踪 */

/* —— 调试 —— */
#define CONFIG_AIRY_DEBUG            /* bool: agentrt-linux 调试总开关 */
#define CONFIG_AIRY_DEBUG_IPC        /* bool: IPC 调试桩 */

/* 以上为子系统前缀（AIRY_*）；供应商级聚合见 CONFIG_AIRYMAXOS_* */
```

#### A.3.3 CONFIG_AIRYMAXOS_* 供应商级聚合

```c
/**
 * CONFIG_AIRYMAXOS_* - 供应商级跨子系统特性（lib/Kconfig.airymaxos）
 *
 * 对齐 lib/Kconfig.airymaxos（OS-KER-027 / OS-BUILD-024）；
 * 命名以 AIRYMAXOS_ 前缀与子系统前缀区分。
 */
#define CONFIG_AIRYMAXOS_LTS                /* int: LTS 轨道号（默认 0） */
#define CONFIG_AIRYMAXOS_KWORKER_NUMA_AFFINITY /* bool: kworker NUMA 亲和（依赖 NUMA） */
#define CONFIG_AIRYMAXOS_GMEM              /* bool: 分组内存记账（依赖 MMU + select SHMEM） */
```

#### A.3.4 KCONFIG_ALLCONFIG 全配置覆盖语义

```makefile
# /**
#  * KCONFIG_ALLCONFIG - 全配置策略保底基线环境变量
#  *
#  * 用于 allmodconfig / allnoconfig / allyesconfig / randconfig：
#  *   Kconfig 先读取此变量指向的 .config 片段作为基线，
#  *   再应用"全部设为 m/y/n"策略。
#  *
#  * agentrt-linux 必须维护 airymaxos-base.config 作为基线（OS-BUILD-026）：
#  *   make allmodconfig KCONFIG_ALLCONFIG=airymaxos-base.config
#  *   make allnoconfig KCONFIG_ALLCONFIG=airymaxos-base.config
#  *
#  * 这样 allnoconfig 也会保留 OS 核心必须启用的 CONFIG_*，
#  * 避免"最小内核"关闭 agentrt-linux 核心特性。
#  *
#  * 对齐 Linux 6.6 scripts/kconfig/conf.c 的 allconfig 处理
#  */
# 语义常量（agentrt-linux 专属建模）
#define KCONFIG_ALLCONFIG_DEFAULT  "airymaxos-base.config"
# KCONFIG_ALLCONFIG=1 时使用 randconfig 生成的种子；=文件路径则读取该文件
```

#### A.3.5 错误码

```c
/**
 * Kconfig 错误码（agentrt-linux 专属，对齐 Linux 6.6 errno 语义）
 */
#define KCONFIG_OK            0          /* 求解成功 */
#define KCONFIG_E_NOENT       (-ENOENT)  /* Kconfig 文件缺失 */
#define KCONFIG_E_INVAL       (-EINVAL)  /* 语法错误或依赖不满足 */
#define KCONFIG_E_LOOP        (-ELOOP)   /* 循环 source 引用 */
#define KCONFIG_E_UNSATISFIED (-ENOTSUP) /* depends on 无法满足 */
#define KCONFIG_E_IO          (-EIO)     /* .config 读写失败 */
#define KCONFIG_E_UNDEF       (-ENOKEY)  /* obj-$(CONFIG_*) 引用未声明符号 */
```

---

> **文档结束** | agentrt-linux 构建系统第 2 卷 | Kconfig 配置系统详解 | 共 12 章节 + 规则编号汇总
