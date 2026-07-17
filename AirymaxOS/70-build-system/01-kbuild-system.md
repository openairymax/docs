Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）Kbuild 递归构建系统详解
> **文档定位**：agentrt-linux（AirymaxOS）构建系统第 1 卷——Kbuild 递归构建机制详解。本卷剖析顶层 `Kbuild` descending 机制、`obj-y`/`obj-m`/`obj-n` 三态门控、子系统 Makefile 编写规范、`if_changed` 增量构建原理、`filechk` 版本注入与 `Makefile.airymaxos` 供应商扩展，并给出 agentrt-linux 构建依赖图。\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-06\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：agentrt `cmake/`（伞仓直属 5 模块）+ Linux 6.6 Kbuild 系统（`Kbuild`、`Makefile`、`scripts/Kbuild.include`、`scripts/Makefile.build`、`scripts/Makefile.lib`）\
> **理论根基**：Linux 6.6 内核基线 + Airymax 五维正交 24 原则（S/K/C/E/A 五维）\
> **核心约束**：IRON-9 v2 同源且部分代码共享——agentrt-linux 内核态构建沿用 Kbuild 思想但与上游保持独立演进节奏

---

## 0. 章节定位

本卷是 agentrt-linux 构建系统 8 卷文档中的第 1 卷，回答"内核态代码如何被编译成 vmlinux 与模块"这一问题。它在 README（模块主索引）与 02-kconfig-system.md（配置系统）之间形成构建执行层：

- **上游依赖**：README 定义构建系统的机制总览与设计原则；本卷定义"递归构建如何被驱动"——顶层 descending、三态门控、增量构建。
- **下游依赖**：02 定义"配置如何驱动构建门控"；03-makefile-patterns.md 定义"子系统 Makefile 的具体惯用法"；04-module-building.md 定义"可加载模块的构建链"。

本卷所有强制规则均赋予 **OS-KER** / **OS-BUILD** 编号，与 50-engineering-standards/07 维护者制度的"规则编号注册表"对齐。agentrt-linux 构建系统以 **Linux 6.6 内核基线** 为工程思想来源，融合 Airymax **五维正交 24 原则** 后重新表述为工程契约。

### 0.1 关键术语

| 术语 | 定义 |
|------|------|
| Kbuild | Linux 内核递归构建系统，顶层 `Kbuild` 文件定义 descending 顺序 |
| descending | 顶层 Makefile 递归进入子目录构建的机制 |
| obj-y / obj-m / obj-n | 三态门控：内建 / 模块 / 不构建 |
| if_changed | 仅当依赖或命令行变化时才重建目标的 Kbuild 宏 |
| filechk | 比较生成文件新旧内容、仅在变化时更新的 Kbuild 宏 |
| built-in.a | 子目录内建目标归档，由 `ar` 合并 `.o` 生成 |
| FORCE | 伪目标，强制规则每次求值（配合 if_changed 使用） |
| 五维正交 24 原则 | Airymax 架构设计原则体系（S/K/C/E/A 五维） |

---

## 1. Kbuild 递归构建总览

agentrt-linux 内核态构建继承 **Linux 6.6 内核基线** 沉淀的 Kbuild 工程思想：顶层 `Kbuild` 文件声明子系统编译入口顺序，`make` 据此递归 descending 到每个子目录，每个子目录通过自身的 `Kbuild`/`Makefile` 声明本目录的 `obj-y`/`obj-m`，最终把所有 `.o` 归并为 `built-in.a`，再链接为 `vmlinux`。

### 1.1 构建流水线全景

```mermaid
flowchart TD
    A[顶层 Makefile] --> B[include Kbuild]
    B --> C{obj-y / obj-m 列表}
    C -->|init/| D1[init/built-in.a]
    C -->|kernel/| D2[kernel/built-in.a]
    C -->|mm/| D3[mm/built-in.a]
    C -->|CONFIG_BLOCK? block/| D4[block/built-in.a]
    C -->|drivers/| D5[drivers/built-in.a]
    C -->|obj-m xxx.o| M1[xxx.ko 模块]
    D1 --> E[vmlinux 链接]
    D2 --> E
    D3 --> E
    D4 --> E
    D5 --> E
    M1 --> F[modules_install]
    E --> G[bzImage / vmlinux]
```

agentrt-linux 在此之上扩展了多语言、多仓的协调层（见 06-airymaxos-build.md），但内核态构建内核仍是 Kbuild 递归 descending。

- **OS-BUILD-001**：agentrt-linux 内核态构建必须以顶层 `Kbuild` 为 descending 入口，禁止在子系统 Makefile 中跨层引用顶层目标；违反者禁止合并（对齐 S-2 层次分解）。
- **OS-KER-090**：顶层 `Kbuild` 的 descending 顺序必须与 Linux 6.6 基线一致（init → usr → arch → kernel → certs → mm → fs → ... → virt），任何顺序调整必须经构建系统评审。

### 1.2 顶层 Kbuild 文件结构

Linux 6.6 顶层 `Kbuild` 文件分为两段：先做全局头文件准备与健全性检查（`bounds.h`、`timeconst.h`、`asm-offsets.h`、missing-syscalls、atomic-checks），再进入"普通目录 descending"段：

```makefile
# Kbuild（顶层）——普通目录 descending 段
obj-y            += init/
obj-y            += usr/
obj-y            += arch/$(SRCARCH)/
obj-y            += $(ARCH_CORE)
obj-y            += kernel/
obj-y            += certs/
obj-y            += mm/
obj-y            += fs/
obj-y            += ipc/
obj-y            += security/
obj-y            += crypto/
obj-$(CONFIG_BLOCK)    += block/
obj-$(CONFIG_IO_URING) += io_uring/
obj-$(CONFIG_RUST)     += rust/
obj-y            += drivers/
obj-y            += sound/
obj-$(CONFIG_SAMPLES)  += samples/
obj-$(CONFIG_NET)      += net/
obj-y            += virt/
```

`obj-y += init/` 表示 `init/` 目录无条件内建；`obj-$(CONFIG_BLOCK) += block/` 表示仅当 `CONFIG_BLOCK=y` 时才 descending 进入 `block/`。`$(SRCARCH)` 与 `$(ARCH_CORE)`/`$(ARCH_LIB)`/`$(ARCH_DRIVERS)` 来自架构层扩展，允许不同架构注入自身的核心/库/驱动目录。

- **OS-BUILD-002**：新增子目录必须在顶层 `Kbuild` 中声明 `obj-y`/`obj-$(CONFIG_*)`，未声明的目录不参与构建；目录命名使用小写连字符风格。
- **OS-KER-091**：`prepare` 阶段（bounds/asm-offsets/missing-syscalls/atomic-checks）必须在 descending 之前完成，子系统 Makefile 不得在 `prepare` 阶段之前依赖 `include/generated/` 下的产物。

---

## 2. obj-y / obj-m / obj-n 三态门控

Kbuild 的核心设计是"配置即构建"：每个目标只能是三态之一——内建（`y`）、模块（`m`）、不构建（`n`）。这一映射由 `scripts/Makefile.lib` 在读取子目录 `Kbuild`/`Makefile` 后规范化。

### 2.1 三态语义

| 状态 | 变量 | 产物 | 链接关系 |
|------|------|------|---------|
| 内建 | `obj-y += foo.o` | `foo.o` → `built-in.a` → `vmlinux` | 静态链接进内核镜像 |
| 模块 | `obj-m += foo.o` | `foo.ko` | 独立可加载模块 |
| 不构建 | `obj-n += foo.o`（或省略） | 无 | 不参与编译 |

`scripts/Makefile.lib` 对列表做规范化处理：先从 `obj-y` 中移除已在 `obj-y` 的 `obj-m`（避免重复），再把形如 `foo/` 的目录条目替换为 `foo/built-in.a`（内建）或 `foo/modules.order`（模块），最后给所有条目加上 `$(obj)/` 前缀：

```makefile
# scripts/Makefile.lib（节选）
obj-m := $(filter-out $(obj-y),$(obj-m))
subdir-ym := $(sort $(subdir-y) $(subdir-m) \
            $(patsubst %/,%, $(filter %/, $(obj-y) $(obj-m))))
# foo/ in obj-y  =>  foo/built-in.a
# foo/ in obj-m  =>  foo/modules.order
obj-m := $(patsubst %/,%/modules.order, $(filter %/, $(obj-y)) $(obj-m))
obj-y := $(patsubst %/, %/built-in.a, $(obj-y))
obj-y := $(filter-out %/, $(obj-y))
```

- **OS-KER-092**：子系统 `obj-*` 列表必须使用 `obj-$(CONFIG_*)` 形式引用配置，禁止硬编码 `obj-y`/`obj-m` 作为特性开关（架构无条件内建目录除外）。
- **OS-BUILD-003**：同一目标不得同时出现在 `obj-y` 与 `obj-m`；`scripts/Makefile.lib` 会静默移除重复项，但子系统作者应主动避免以保持清单可读（对齐 A-4 完美主义）。

### 2.2 多部分目标（multi-part objects）

当一个内核模块/内建由多个 `.c` 组合而成，使用 `<name>-objs` 或 `<name>-y` 列出组成部分：

```makefile
# drivers/airymax/Makefile
obj-$(CONFIG_AIRY_IPC) += airymax-ipc.o
airymax-ipc-y := ipc_core.o ipc_dispatch.o ipc_ringbuf.o
airymax-ipc-$(CONFIG_AIRY_IPC_DEBUG) += ipc_debug.o
```

`scripts/Makefile.lib` 的 `real-search` 把 `airymax-ipc-y` 展开为真实的 `ipc_core.o` 等目标，再由 `if_changed_rule` 的 `ld_multi_m` 规则链接为 `airymax-ipc.o`，最终打包为 `airymax-ipc.ko`（模块）或并入 `built-in.a`（内建）。

- **OS-BUILD-004**：多部分目标的 `-objs`/`-y` 后缀列表必须与目标名严格匹配；新增组成部分必须同时更新列表，遗漏会导致链接缺失符号。

---

## 3. 子系统 Makefile 编写规范

agentrt-linux 子系统 Makefile 遵循 Linux 6.6 内核基线的 `Kbuild`/`Makefile` 双文件约定：若目录同时存在 `Kbuild` 与 `Makefile`，`scripts/Kbuild.include` 的 `kbuild-file` 宏优先选 `Kbuild`。agentrt-linux 推荐新子系统统一使用 `Kbuild` 命名以避免歧义。

### 3.1 标准子系统 Makefile 骨架

```makefile
# SPDX-License-Identifier: GPL-2.0
# drivers/airymax/ipc/Kbuild
#
# agentrt-linux（AirymaxOS）AgentsIPC 内核态通道构建规则

obj-$(CONFIG_AIRY_IPC) += airymax-ipc.o

airymax-ipc-y := ipc_core.o \
                 ipc_dispatch.o \
                 ipc_ringbuf.o \
                 ipc.o

airymax-ipc-$(CONFIG_AIRY_IPC_BENCH) += ipc_bench.o

ccflags-y += -I$(srctree)/drivers/airymax/include
subdir-ccflags-y += -DAIRY_IPC_PBUF=128
```

要点：`obj-$(CONFIG_*)` 门控特性；`<name>-y` 列出组成部分；`ccflags-y` 仅作用于本目录，`subdir-ccflags-y` 传播到子目录；`-I` 用 `$(srctree)` 而非相对路径以兼容 out-of-tree 构建。

- **OS-KER-093**：子系统 Makefile 必须以 `SPDX-License-Identifier` 开头，缺失许可证标识的文件禁止合并（对齐 50-engineering-standards/01-coding-standards）。
- **OS-BUILD-005**：`ccflags-y`/`subdir-ccflags-y` 用于本目录及子目录的编译开关；跨子系统的全局开关必须在顶层 `Makefile` 或 `arch/$(SRCARCH)/Makefile` 中设置，禁止在叶子子系统扩散。

### 3.2 递归 descending 的驱动规则

`scripts/Makefile.build` 在读取子目录 `Kbuild` 后，把 `obj-y`/`obj-m` 中的 `foo/` 目录替换为 `foo/built-in.a`（或 `foo/modules.order`），并为这些归档目标建立依赖规则：

```makefile
# scripts/Makefile.build（descending 规则）
$(subdir-builtin): $(obj)/%/built-in.a: $(obj)/% ;
$(subdir-modorder): $(obj)/%/modules.order: $(obj)/% ;

# 子目录归档合并规则
quiet_cmd_ar_builtin = AR      $@
      cmd_ar_builtin = rm -f $@; \
        $(if $(real-prereqs), printf "$(obj)/%s " ...) $(AR) cDPrST $@
$(obj)/built-in.a: $(real-obj-y) FORCE
	$(call if_changed,ar_builtin)
```

`$(subdir-ym)` 被声明为 PHONY，`make` 通过 `$(Q)$(MAKE) $(build)=$@` 二次进入子目录，把构建上下文（`obj`、`need-builtin`、`need-modorder`）通过命令行变量传递下去。这就是"递归 make"——每个子目录是一个独立的 make 调用，但通过 `Kbuild.include` 共享宏定义。

- **OS-KER-076**：子系统 `built-in.a` 由 `ar cDPrST`（无符号表、确定顺序）生成，禁止使用带符号表的 `ar`；确定顺序是可重现构建（E-3 资源确定性）的前提。
- **OS-BUILD-006**：`need-builtin`/`need-modorder` 由父目录在 descending 时按需传递，子系统不得自行假设这两个变量的值，必须以传入值为准。

---

## 4. if_changed 增量构建原理

`if_changed` 是 Kbuild 增量构建的核心宏，agentrt-linux 沿用并要求所有生成物规则通过它保护。其设计哲学与 Airymax **五维正交 24 原则** 中的 A-4（完美主义：仅重建真正需要的）与 E-3（资源确定性：可重现）高度契合。

### 4.1 if_changed 的判定逻辑

`if_changed` 仅在三种情况重建目标：(1) 任一真实先决条件比目标新（`newer-prereqs`）；(2) 命令行本身变化（`cmd-check`，比较 `savedcmd_$@`）；(3) 缺失 `FORCE` 先决条件（`check-FORCE`，警告）。判定通过 `if-changed-cond` 聚合：

```makefile
# scripts/Kbuild.include（核心宏）
newer-prereqs = $(filter-out $(PHONY),$?)
check-FORCE = $(if $(filter FORCE, $^),,$(warning FORCE prerequisite is missing))
if-changed-cond = $(newer-prereqs)$(cmd-check)$(check-FORCE)

if_changed = $(if $(if-changed-cond),$(cmd_and_savecmd),@:)
cmd_and_savecmd = \
    $(cmd); \
    printf '%s\n' 'savedcmd_$@ := $(make-cmd)' > $(dot-target).cmd

if_changed_dep = $(if $(if-changed-cond),$(cmd_and_fixdep),@:)
cmd_and_fixdep = \
    $(cmd); \
    scripts/basic/fixdep $(depfile) $@ '$(make-cmd)' > $(dot-target).cmd; \
    rm -f $(depfile)
```

关键点：`FORCE` 是伪目标，每次都被视为"已更新"，从而让 `if_changed` 的判定逻辑接管（而非 Make 的时间戳判定）。`savedcmd_$@` 写入 `.<target>.cmd` 文件，下次构建时回读比较。`if_changed_dep` 额外调用 `fixdep` 把编译器生成的 `.d` 依赖中的 `CONFIG_*` 符号展开为对 `include/config/*.h` 的依赖，实现"配置变化触发重编"。

### 4.2 .cmd 文件与配置依赖

```mermaid
flowchart LR
    SRC[foo.c] -->|gcc -MMD| DEP[.foo.o.d]
    DEP -->|fixdep| CMD[.foo.o.cmd]
    CONF[auto.conf] --> CMD
    CMD -->|savedcmd 回读| JUDGE{if_changed 判定}
    FORCE[FORCE 伪目标] --> JUDGE
    JUDGE -->|需要重建| BUILD[gcc -c foo.c]
    JUDGE |>|无变化| SKIP[跳过]
```

`fixdep` 把 `#include <linux/foo.h>` 中的配置宏（如 `CONFIG_FOO`）翻译为对 `include/config/FOO`（空文件，由 syncconfig 创建作为时间戳标记）的依赖。这样当用户改了 `CONFIG_FOO`，对应的时间戳文件被 touch，所有引用它的 `.o` 都被标记为需要重编。

- **OS-KER-077**：所有生成物规则（`.o`/`.a`/`.ko`/版本头）必须使用 `if_changed`/`if_changed_dep`/`if_changed_rule` 之一，禁止裸命令；裸命令会绕过增量判定导致全量重建。
- **OS-BUILD-007**：禁止手动删除 `.<target>.cmd` 文件作为"强制重建"手段；正确做法是 `make clean` 或在规则中显式声明 `FORCE`。`.cmd` 文件是构建可重现性的证据链。
- **OS-BUILD-008**：调试 `if_changed` 误重建时使用 `make V=2` 查看原因（PHONY / target missing / newer prereqs / command line change / missing .cmd / not in targets）。

---

## 5. filechk 版本注入与 Makefile.airymaxos

版本号的注入是 Kbuild 中 `filechk` 宏的典型应用场景。agentrt-linux 把上游的供应商版本文件改写为 `Makefile.airymaxos`，定义自身的 LTS/主/次/发布四元版本，并通过 `filechk_version.h` 注入到 `include/generated/uapi/linux/version.h`。

### 5.1 filechk 宏机制

`filechk` 的作用是比较生成文件的新旧内容，仅在内容变化时才覆盖目标（避免无意义的时间戳更新触发下游全量重编）：

```makefile
# scripts/Kbuild.include（filechk 定义）
define filechk
	$(check-FORCE)
	$(Q)set -e; \
	mkdir -p $(dir $@); \
	trap "rm -f $(tmp-target)" EXIT; \
	{ $(filechk_$(1)); } > $(tmp-target); \
	if [ ! -r $@ ] || ! cmp -s $@ $(tmp-target); then \
		$(kecho) '  UPD     $@'; \
		mv -f $(tmp-target) $@; \
	fi
endef
```

工作流：先写到临时文件 `$(tmp-target)`，若目标不存在或内容不同（`cmp -s`）则 `mv` 覆盖，否则保持原文件与时间戳不变。这正是 E-3（资源确定性：最小变更面）的工程体现——版本号没变就不触发下游重编。

### 5.2 Makefile.airymaxos 供应商版本注入

agentrt-linux 用 `Makefile.airymaxos` 取代上游供应商版本文件，定义四元版本号：

```makefile
# SPDX-License-Identifier: GPL-2.0
# Makefile.airymaxos —— agentrt-linux 供应商版本定义
AIRYMAXOS_LTS = 0
AIRYMAXOS_MAJOR = 9999
AIRYMAXOS_MINOR = 0
AIRYMAXOS_RELEASE = ""
```

顶层 `Makefile` 通过 `include $(srctree)/Makefile.airymaxos` 引入，再在 `filechk_version.h` 中注入为 C 宏：

```makefile
# Makefile（节选）
include $(srctree)/Makefile.airymaxos

define filechk_version.h
	if [ $(SUBLEVEL) -gt 255 ]; then
		echo \#define LINUX_VERSION_CODE ...;
	else
		echo \#define LINUX_VERSION_CODE ...;
	fi;
	echo '#define KERNEL_VERSION(a,b,c) (((a) << 16) + ((b) << 8) + ((c) > 255 ? 255 : (c)))';
	echo \#define LINUX_VERSION_MAJOR $(VERSION);
	echo \#define LINUX_VERSION_PATCHLEVEL $(PATCHLEVEL);
	echo \#define LINUX_VERSION_SUBLEVEL $(SUBLEVEL);
	echo \#define AIRYMAXOS_LTS $(AIRYMAXOS_LTS);
	echo \#define AIRYMAXOS_MAJOR $(AIRYMAXOS_MAJOR);
	echo \#define AIRYMAXOS_MINOR $(AIRYMAXOS_MINOR);
	echo '#define AIRYMAXOS_VERSION(a,b) (((a) << 8) + (b))';
	echo \#define AIRYMAXOS_VERSION_CODE $(shell expr $(AIRYMAXOS_MAJOR) \* 256 + $(AIRYMAXOS_MINOR));
	echo \#define AIRYMAXOS_RELEASE \"$(AIRYMAXOS_RELEASE)\"
endef

$(version_h): PATCHLEVEL := $(or $(PATCHLEVEL), 0)
$(version_h): SUBLEVEL := $(or $(SUBLEVEL), 0)
$(version_h): FORCE
	$(call filechk,version.h)
```

**特性开关与版本号解耦**：`CONFIG_*` 控制特性是否编译，`AIRYMAXOS_*` 仅注入版本字符串；二者正交，符合 K-4（可插拔策略）与 A-1（极简主义：单一职责）。

- **OS-BUILD-009**：`Makefile.airymaxos` 的版本四元组（LTS/MAJOR/MINOR/RELEASE）变更必须经发布管理评审，且必须同步更新 `kernel` 子仓的发行说明；版本号变更触发 `version.h` 重生进而全量重编属预期行为。
- **OS-STD-049**：禁止在 `Makefile.airymaxos` 中放置除版本四元组以外的任何变量；特性开关必须走 `Kconfig`，禁止把特性塞进版本文件（对齐 S-1 单一职责）。
- **OS-BUILD-010**：`version.h` 的生成必须经 `filechk`，禁止用裸 `echo >` 覆盖；裸覆盖会破坏增量构建并导致每次 `make` 都全量重编。

---

## 6. 构建依赖图与阶段划分

agentrt-linux 把内核构建分为若干阶段，每个阶段有明确的先决与产物。理解阶段划分是编写正确子系统 Makefile 的前提。

```mermaid
flowchart TD
    subgraph P1[阶段 1：脚本与配置准备]
        S1[scripts_basic] --> S2[syncconfig/auto.conf]
        S2 --> S3[include/config/auto.conf]
    end
    subgraph P2[阶段 2：全局头生成]
        P2a[bounds.h] --> P2b[timeconst.h]
        P2b --> P2c[asm-offsets.h]
        P2c --> P2d[missing-syscalls]
        P2d --> P2e[atomic-checks]
        P2e --> P2f[version.h / utsrelease.h]
    end
    subgraph P3[阶段 3：descending 编译]
        D1[obj-y 子目录] --> D2[每子目录 built-in.a]
        D3[obj-m 目标] --> D4[.ko 模块]
    end
    subgraph P4[阶段 4：链接与打包]
        L1[vmlinux 链接] --> L2[bzImage]
        D4 --> L3[modules.builtin]
    end
    P1 --> P2 --> P3 --> P4
```

`prepare` 阶段（P2）必须先于 descending（P3），因为子系统 `.c` 会 `#include <generated/bounds.h>` 等。`auto.conf`（P1 的 syncconfig 产物）被 `scripts/Makefile.build` 在每个子目录 `-include`，提供 `CONFIG_*` 宏的 make 侧值。

- **OS-KER-078**：`scripts_basic` 与 `syncconfig` 必须先于任何 descending；子系统 Makefile 不得在 `scripts_basic` 完成前被 include（否则 `fixdep` 等工具缺失会导致依赖链断裂）。
- **OS-BUILD-011**：`vmlinux` 链接前所有 `built-in.a` 必须就绪；若发现链接时缺符号，先检查对应子系统是否被 `obj-y` 正确声明，而非直接补 `.o`。
- **OS-BUILD-012**：CI 必须在 `allmodconfig`/`allnoconfig`/`defconfig` 三种配置下验证 descending 完整性，确保 `obj-$(CONFIG_*)` 门控在任意配置下都能闭合（对齐 OS-STD-032 CI 矩阵）。

### 6.1 构建目标生命周期状态机

内核构建从源码树到 vmlinux/bzImage 的完整状态转换，覆盖配置 → 编译 → 链接全链路，含失败恢复与清理路径：

```mermaid
stateDiagram-v2
    [*] --> UNCONFIGURED: 源码树就绪，未执行 make

    UNCONFIGURED --> CONFIGURING: make defconfig / make menuconfig
    CONFIGURING --> CONFIGURED: .config 生成，syncconfig 求解依赖闭包
    CONFIGURING --> CONFIG_FAILED: 配置冲突或 depends on 缺失

    CONFIG_FAILED --> CONFIGURING: 修正冲突后重新 make menuconfig

    CONFIGURED --> BUILDING: make 开始 descending 编译
    BUILDING --> BUILT: 所有 obj-y/m 目标编译链接成功
    BUILDING --> FAILED: 编译错误或链接失败

    BUILT --> UNCONFIGURED: make mrproper 清理（回到初始）
    FAILED --> CONFIGURED: 修正错误后 make 继续
    FAILED --> UNCONFIGURED: make mrproper 完全清理

    note right of CONFIGURED
        .config 已持久化到源码树根
        auto.conf / autoconf.h 已由 syncconfig 生成
        include/config/*.h 时间戳已创建
        可随时 make 重新编译
    end note

    note right of BUILDING
        Kbuild 递归 descending：
        prepare 阶段 → 子目录 obj-y/m → built-in.a → vmlinux
        if_changed 增量判定，仅重建变化目标
    end note

    note right of FAILED
        编译器/链接器返回非零退出码
        .cmd 证据链保留，可 make V=2 排查
        修正源码后 make 从失败点继续
    end note
```

**状态转换条件**：

| 从状态 | 到状态 | 触发条件 | 系统行为 |
|--------|--------|---------|---------|
| — | UNCONFIGURED | 源码树就绪（`git clone` / 解压），未执行 `make` | 无 `.config`，无 `auto.conf`，`built-in.a` 不存在 |
| UNCONFIGURED | CONFIGURING | `make defconfig` / `make menuconfig` / `make olddefconfig` | Kconfig 解析器读取 `Kconfig` 源文件树，求解依赖闭包 |
| CONFIGURING | CONFIGURED | `.config` 生成成功，`syncconfig` 产出 `auto.conf`/`autoconf.h` | `include/config/*.h` 时间戳创建，`fixdep` 依赖链就绪 |
| CONFIGURING | CONFIG_FAILED | `depends on` 无法满足 / `select` 目标依赖缺失 / 语法错误 | Kconfig 报 `unsatisfied dependence` 警告，`.config` 不完整 |
| CONFIG_FAILED | CONFIGURING | 修正配置冲突后重新 `make menuconfig`/`make olddefconfig` | 重新求解依赖闭包 |
| CONFIGURED | BUILDING | `make` / `make all` / `make bzImage` 触发 descending | `scripts_basic` + `syncconfig` → `prepare` → 子目录 `obj-y/m` 编译 |
| BUILDING | BUILT | 所有 `obj-y` 目标编译链接成功，`vmlinux`/`bzImage` 生成 | `built-in.a` 合并完成，`modules.builtin` 记录内建模块 |
| BUILDING | FAILED | 编译器返回非零（语法/类型错误）或链接器缺符号 | `make` 退出码非零，`.<target>.cmd` 证据链保留 |
| BUILT | UNCONFIGURED | `make mrproper`（清理 `.config` + `auto.conf` + 产物） | 回到初始状态，需重新 `make defconfig` |
| FAILED | CONFIGURED | 修正源码错误后 `make`（`.config` 未变，增量继续） | `if_changed` 仅重建修正的 `.o` 及其下游 |
| FAILED | UNCONFIGURED | `make mrproper`（放弃当前配置，完全清理） | 回到初始状态 |

---

## 7. 五维原则映射

本卷 Kbuild 递归构建机制与 Airymax **五维正交 24 原则** 的映射如下：

| 原则 | 在本卷的体现 |
|------|-------------|
| **S-1 单一职责** | `Makefile.airymaxos` 仅放版本四元组，特性开关走 Kconfig；`built-in.a` 仅做归并不链接 |
| **S-2 层次分解** | 顶层 `Kbuild` 定义 descending 顺序，子系统 Makefile 仅管本目录；递归 make 按目录层次组织 |
| **K-4 可插拔策略** | `obj-$(CONFIG_*)` 使特性可插拔；`obj-y`/`obj-m`/`obj-n` 三态门控实现策略与机制分离 |
| **E-3 资源确定性** | `filechk` 内容比较避免无谓时间戳更新；`ar cDPrST` 确定顺序；可重现构建基线 |
| **E-6 错误可追溯** | `.<target>.cmd` 记录命令行证据链；`make V=2` 给出重建原因 |
| **A-1 极简主义** | `if_changed` 仅重建真正需要的；`FORCE` + 判定宏避免裸命令全量重建 |
| **A-4 完美主义** | 增量构建精确到命令行参数变化；`cmd-check` 捕获编译选项漂移 |

agentrt-linux 构建系统以 **Linux 6.6 内核基线** 为工程思想来源，但每一项机制都经过 **五维正交 24 原则** 的映射校验，确保工程决策有原则可循而非照搬上游。

---

## 8. 同源 agentrt 映射

agentrt-linux 构建系统与 agentrt 构建系统遵循 **IRON-9 v2 同源且部分代码共享** 原则。agentrt 是 Airymax 智能体运行时（应用层），其 `cmake/` 目录提供 5 模块的跨模块共享 CMake 工具链；agentrt-linux 是智能体操作系统（OS 层），内核态使用 Kbuild。二者在分层与工具链上独立，在工程思想与原则上映射同源。

| 维度 | agentrt | agentrt-linux 内核态 | 关系 |
|------|---------|------------------|------|
| 构建系统 | CMake（用户态） | Kbuild（内核态） | 同源思想（递归/增量/配置门控），独立工具链 |
| 配置门控 | CMake `-D<option>=ON` | `obj-$(CONFIG_*)` | 语义同源（配置驱动构建），机制独立 |
| 增量构建 | CMake Ninja deps | `if_changed` + `.cmd` | 同源思想，独立实现 |
| 版本注入 | CMake `project(VERSION ...)` | `Makefile.airymaxos` + `filechk` | 同源语义，独立注入路径 |
| 多语言协调 | cargo/poetry/tsc 各自 CMake 集成 | Kbuild（C）+ 外部多语言协调层 | 同源多语言目标，独立集成层 |

**同源红利**：agentrt 在 agentrt-linux 上运行时，构建产物的版本号语义对齐（`AIRYMAXOS_*` 与 agentrt `project(VERSION)` 的 MAJOR/MINOR 同源），便于联调时定位版本漂移；增量构建思想一致，开发者心智模型可复用。

**独立性**：agentrt-linux 内核态构建为 OS 层 Kbuild，agentrt 用户态构建为应用层 CMake，二者通过产物契约（`.ko`/`vmlinux` 与可执行文件）解耦。当 agentrt 构建工具链演进时，agentrt-linux 通过构建评审决定是否同步，避免被动跟随。这一独立性正是 **IRON-9 v2 同源且部分代码共享** 在构建系统层的体现——同源思想，独立演进。

agentrt-linux 构建系统在 **Linux 6.6 内核基线** 上构建，其递归 descending、三态门控、`if_changed`、`filechk` 等机制均直接源自上游沉淀；agentrt-linux 的扩展（`Makefile.airymaxos`、多语言协调层、多仓集成）以 **五维正交 24 原则** 为设计准绳，确保扩展不破坏上游机制的可预测性。这种"上游思想 + 自身原则"的双层结构，是 **IRON-9 v2 同源且部分代码共享** 在工程实现层的落实。

### 8.1 IRON-9 v2 三层共享模型

本节将上节"同源 agentrt 映射"进一步细化为 **IRON-9 v2 三层共享模型**，明确构建系统层在用户态（agentrt）与内核态（agentrt-linux）之间的代码共享边界。三层分别为：**[SC] 共享契约层**（共享头文件 / 数据结构定义）、**[SS] 语义同源层**（设计模式同源但实现独立）、**[IND] 完全独立层**（双方各自独立实现）。该模型由 10 个 [SC] 头文件契约、跨态语义对照表与独立实现清单共同支撑。

#### 8.1.1 三层模型概览表

| 层次 | 共享程度 | 构建系统层内容 |
|------|---------|---------------|
| **[SC] 共享契约层** | 无 | 无 [SC] 层——构建系统为 agentrt-linux 内核态专属，不与 agentrt 用户态共享代码 |
| **[SS] 语义同源层** | 设计模式同源 | Makefile 目标模式、Kbuild 目录递归构建模式、`obj-y/m-$(CONFIG_*)` 条件编译——与 agentrt CMake/Meson 在"声明式构建描述"语义上同源 |
| **[IND] 完全独立层** | 完全独立 | 8 子仓 Kbuild 系统、交叉编译工具链配置、Kbuild Makefile include 机制、`scripts/Makefile.*` 模块化 |

#### 8.1.2 [SC] 共享契约层

**无直接 [SC] 共享头文件**。

构建系统层不属于 IRON-9 v2 的 10 个 [SC] 共享头文件清单（`syscalls.h` / `memory_types.h` / `security_types.h` / `cognition_types.h` / `sched.h` / `ipc.h`）。构建系统是工程基础设施，其产物（`.ko` / `vmlinux` / 可执行文件）通过二进制契约解耦，而源码层无共享头文件依赖。这一约束确保 agentrt 用户态构建工具链演进时不会被动牵连 agentrt-linux Kbuild，反之亦然——构建系统层的演进由各自的 **OS-BUILD 评审** 独立裁决。

#### 8.1.3 [SS] 语义同源层

| 语义维度 | agentrt 用户态（CMake/Meson） | agentrt-linux 内核态（Kbuild） | 同源语义 |
|---------|------------------------------|-------------------------------|----------|
| 目标模式 | `all` / `install` / `test` / `clean` | `all` / `modules` / `install` / `clean` | 声明式构建目标，`make <target>` 触发 |
| 目录递归 | `add_subdirectory()` 递归 | Kbuild `subdir-$(CONFIG_*)` + descending | 按目录层次递归构建 |
| 条件编译 | `if(OPTION)` / `-D<option>=ON` | `obj-y` / `obj-m` / `obj-$(CONFIG_*)` 三态门控 | 配置驱动构建选择 |
| 增量构建 | Ninja deps + `ninja -t deps` | `if_changed` + `.<target>.cmd` | 仅重建真正变化的目标 |
| 版本注入 | `project(VERSION x.y.z)` | `Makefile.airymaxos` + `filechk` | 构建期版本号注入产物 |
| 多语言协调 | cargo / poetry / tsc CMake 集成 | Kbuild（C）+ 外部协调层 | 多语言目标统一编排 |

**语义说明**：agentrt 用户态的 CMake/Meson 构建模式与 agentrt-linux 内核态的 Kbuild 在"声明式构建描述"这一核心语义上同源——二者均通过**声明式描述**（CMakeLists.txt / Kbuild）告知构建系统"做什么"，而非"怎么做"，由底层工具链（Ninja / Make）决定执行细节。这种同源使开发者在两端的心智模型可复用：理解了 Kbuild 的 `obj-$(CONFIG_*)` 三态门控，即理解了 CMake 的 `if(OPTION)` 条件编译语义。但**机制完全独立**——CMake 使用 Ninja 依赖图，Kbuild 使用 `if_changed` 命令行证据链，二者无代码共享。

#### 8.1.4 [IND] 完全独立层

| 独立实现项 | agentrt-linux 内核态 | agentrt 用户态 | 独立原因 |
|-----------|---------------------|---------------|---------|
| 8 子仓 Kbuild 系统 | `Makefile` + 8 子仓 Kbuild 递归 | CMake 单仓或独立多仓 | 内核态必须分仓 Kbuild；用户态无此约束 |
| 交叉编译工具链配置 | `CROSS_COMPILE` / `ARCH` / `LLVM=1` | CMake toolchain file | 内核交叉编译语义特有 |
| Kbuild Makefile include 机制 | `include scripts/Makefile.*` | CMake `include()` / Meson `subdir()` | Kbuild 模块化特有机制 |
| `scripts/Makefile.*` 模块化 | `Makefile.build` / `Makefile.lib` / `Makefile.modpost` | 无对应 | Kbuild 构建阶段切分特有 |
| 版本注入路径 | `Makefile.airymaxos` + `filechk` | `project(VERSION ...)` | 注入路径与产物格式不同 |
| 产物格式 | `vmlinux` / `.ko` / `built-in.a` | ELF 可执行文件 / `.so` | 内核态产物格式特有 |

#### 8.1.5 跨态协作流

```mermaid
graph LR
    subgraph "agentrt 用户态（应用层）"
        A1["CMakeLists.txt<br/>声明式描述"]
        A2["CMake/Ninja<br/>构建引擎"]
        A3["agentrt 可执行文件<br/>+ .so"]
    end
    subgraph "agentrt-linux 内核态（OS 层）"
        K1["Kbuild<br/>声明式描述"]
        K2["if_changed + .cmd<br/>构建引擎"]
        K3["vmlinux + .ko<br/>+ built-in.a"]
    end
    SS["[SS] 语义同源层<br/>声明式构建描述<br/>目标模式 / 递归 / 条件编译"]
    IND["[IND] 完全独立层<br/>工具链 / 产物格式<br/>版本注入路径"]

    A1 -->|同源语义| SS
    K1 -->|同源语义| SS
    SS --> A2
    SS --> K2
    A2 --> A3
    K2 --> K3
    A1 -.->|独立实现| IND
    K1 -.->|独立实现| IND
    A3 -.->|二进制契约解耦| K3

    style SS fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000
    style IND fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    style A1 fill:#e3f2fd,stroke:#1565c0,color:#000
    style A2 fill:#e3f2fd,stroke:#1565c0,color:#000
    style A3 fill:#e3f2fd,stroke:#1565c0,color:#000
    style K1 fill:#fce4ec,stroke:#c62828,color:#000
    style K2 fill:#fce4ec,stroke:#c62828,color:#000
    style K3 fill:#fce4ec,stroke:#c62828,color:#000
```

**协作说明**：agentrt 用户态通过 CMake/Ninja 构建可执行文件与共享库，agentrt-linux 内核态通过 Kbuild + `if_changed` 构建内核镜像与模块。两端在 **[SS] 语义同源层** 共享"声明式构建描述"的设计模式（目标模式 / 目录递归 / 条件编译 / 增量构建），使开发者在两端的心智模型可复用；但在 **[IND] 完全独立层** 各自维护工具链配置、产物格式、版本注入路径。最终通过**二进制契约**解耦：agentrt 用户态产物（可执行文件）运行于 agentrt-linux 内核态产物（vmlinux）之上，二者无源码层依赖，无 [SC] 共享头文件。这正是 **IRON-9 v2 同源且部分代码共享** 在构建系统层的落地——同源思想，独立演进，无共享契约。

### 8.2 8 子仓构建依赖与交叉构建矩阵

agentrt-linux 的 8 子仓（kernel/services/security/memory/cognition/cloudnative/system/tests-linux）在构建层存在严格的依赖闭包：kernel 是基础，上层子仓的构建必须等待下层子仓产物就绪后方可触发 descending，违反依赖顺序将导致符号缺失或 ABI 断裂。

#### 8.2.1 8 子仓构建依赖图

```mermaid
graph TD
    K["kernel 子仓<br/>MicroCoreRT 内核 + vmlinux"]
    S["services 子仓<br/>12 daemons"]
    SE["security 子仓<br/>Cupolas 安全穹顶"]
    M["memory 子仓<br/>MemoryRovol 四层记忆"]
    SY["system 子仓<br/>AgentsIPC 128B 协议"]
    CO["cognition 子仓<br/>CoreLoopThree"]
    CL["cloudnative 子仓<br/>云原生 Agent"]
    T["tests-linux 子仓<br/>KUnit / kselftest"]

    K --> S
    K --> SE
    K --> M
    K --> SY
    S --> SY
    S --> CO
    M --> CO
    S --> CL
    SY --> CL
    K --> T
    S --> T
    SE --> T
    M --> T
    SY --> T
    CO --> T
    CL --> T

    style K fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    style S fill:#e8f5e9,stroke:#2e7d32,color:#000
    style SE fill:#fbe9e7,stroke:#bf360c,color:#000
    style M fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style SY fill:#fce4ec,stroke:#ad1457,color:#000
    style CO fill:#fff8e1,stroke:#f57f17,color:#000
    style CL fill:#e0f7fa,stroke:#006064,color:#000
    style T fill:#e0f2f1,stroke:#00695c,stroke-width:2px,color:#000
```

**依赖分层说明**（`A --> B` 表示 A 是 B 的构建先决）：
- **L0 基础层**：`kernel` 子仓产出 `vmlinux` 与 `built-in.a`，是全部上层子仓的构建先决。
- **L1 内核态扩展层**：`services`/`security`/`memory` 仅依赖 `kernel` 的头文件与 `obj-$(CONFIG_*)` 门控，三者可并行构建。
- **L2 组合层**：`system` 依赖 `kernel` + `services`（AgentsIPC 协议栈消费 services daemon 接口）；`cognition` 依赖 `services` + `memory`（CoreLoopThree 消费记忆四层与 daemon 服务）。二者无相互依赖，可并行构建。
- **L3 云原生层**：`cloudnative` 依赖 `services` + `system`（云原生 Agent 经 AgentsIPC 协议与 services daemon 通信），须等待 L2 的 `system` 就绪。
- **L4 验证层**：`tests-linux` 依赖全部 7 仓的构建产物，最后触发 KUnit/kselftest。

#### 8.2.2 交叉构建配置矩阵

agentrt-linux 的 8 子仓跨越 C（kernel/services/security/memory/system 的内核态）、Rust（cognition 的 CoreLoopThree）、Go（cloudnative 的云原生 Agent）、Python（tests-linux 的测试脚本）四种语言生态，对应四套构建系统。下表给出目标平台 × 编译器 × 构建系统的完整交叉构建矩阵：

| 目标平台 | 编译器 | 构建系统 | 适用子仓 | 关键环境变量 | 产物 |
|---------|--------|---------|---------|-------------|------|
| x86_64 | GCC 13.x | Kbuild | kernel/services/security/memory/system | `ARCH=x86_64 CC=gcc` | `vmlinux` + `.ko` |
| x86_64 | Clang 17.x | Kbuild | kernel/services/security/memory/system | `ARCH=x86_64 LLVM=1 CC=clang` | `vmlinux` + `.ko` |
| x86_64 | GCC 13.x | Meson | services（用户态 daemon） | `meson setup build` | ELF 可执行文件 |
| x86_64 | Rust 1.75 | Cargo | cognition | `cargo build --release` | `.so` / 可执行文件 |
| x86_64 | Go 1.22 | Go Modules | cloudnative | `go build ./...` | 容器镜像 |
| aarch64 | GCC 13.x | Kbuild | kernel/services/security/memory/system | `ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-` | `vmlinux` + `.ko` |
| aarch64 | Clang 17.x | Kbuild | kernel/services/security/memory/system | `ARCH=arm64 LLVM=1 CROSS_COMPILE=aarch64-linux-gnu- CC=clang` | `vmlinux` + `.ko` |
| aarch64 | GCC 13.x | Meson | services（用户态 daemon） | `meson setup build --cross-file aarch64.ini` | ELF 可执行文件 |
| aarch64 | Rust 1.75 | Cargo | cognition | `cargo build --target aarch64-unknown-linux-gnu` | `.so` / 可执行文件 |
| aarch64 | Go 1.22 | Go Modules | cloudnative | `GOARCH=arm64 go build ./...` | 容器镜像 |

**矩阵约束**：`LLVM=1` 与 `CROSS_COMPILE` 在 Clang 构建下需配合 `CROSS_COMPILE` 提供 sysroot 前缀（Clang 本身是交叉编译器，但 `ar`/`ld` 仍需 GNU binutils 前缀）。Meson 与 Cargo 的交叉构建通过各自的 cross-file / target triple 配置，与 Kbuild 的 `CROSS_COMPILE` 机制独立——这正是 §8.1.4 [IND] 完全独立层"交叉编译工具链配置"项的体现。

#### 8.2.3 规则编号

- **OS-BUILD-013**：8 子仓构建必须按依赖分层顺序触发——L0（kernel）先于 L1（services/security/memory），L1 先于 L2（system/cognition），L2 先于 L3（cloudnative），L3 先于 L4（tests-linux）；同层子仓可并行构建。CI 编排脚本必须显式声明分层依赖，违反顺序导致符号缺失的构建失败由编排脚本承担责任（对齐 S-2 层次分解）。
- **OS-BUILD-014**：同一目标平台下，全部子仓的交叉编译工具链版本必须一致——GCC/Clang 主版本号与 binutils 版本号由顶层 `toolchain.lock` 文件锁定，禁止子仓各自声明工具链版本；工具链版本漂移会导致 ABI 不一致（对齐 E-3 资源确定性 + IRON-9 v2 工具链独立演进约束）。
- **OS-BUILD-015**：8 子仓构建产物（`vmlinux`/`.ko`/ELF/`.so`/容器镜像）必须经 `sha256sum` 签名并写入 `build-manifest.json`；`tests-linux` 子仓的 KUnit/kselftest 必须校验 manifest 签名后再运行，禁止对未签名产物执行测试（对齐 E-6 错误可追溯 + E-3 资源确定性）。
- **OS-BUILD-016**：交叉构建配置矩阵的任一组合（平台 × 编译器 × 构建系统）变更必须经构建系统评审，并在 CI 矩阵中同步增减；新增目标平台必须同时提供该平台下全部 4 种构建系统的配置，禁止只覆盖部分子仓（对齐 OS-STD-032 CI 矩阵 + A-4 完美主义）。

---

## 9. OS-KER / OS-BUILD 规则编号汇总

本卷定义的规则编号汇总如下，与 07 卷"规则编号注册表"对齐：

| 编号 | 规则 | 强制级别 |
|------|------|---------|
| OS-KER-090 | 顶层 Kbuild descending 顺序与基线一致 | MUST |
| OS-KER-091 | prepare 阶段先于 descending | MUST |
| OS-KER-092 | obj-* 用 obj-$(CONFIG_*) 门控 | MUST |
| OS-KER-093 | 子系统 Makefile 含 SPDX 标识 | MUST |
| OS-KER-076 | built-in.a 用 ar cDPrST 确定顺序 | MUST |
| OS-KER-077 | 生成物规则用 if_changed 系列 | MUST |
| OS-STD-049 | Makefile.airymaxos 仅放版本四元组 | MUST |
| OS-KER-078 | scripts_basic/syncconfig 先于 descending | MUST |
| OS-BUILD-001 | 顶层 Kbuild 为 descending 唯一入口 | MUST |
| OS-BUILD-002 | 新增子目录在 Kbuild 声明 obj-* | MUST |
| OS-BUILD-003 | 目标不同时在 obj-y 与 obj-m | MUST |
| OS-BUILD-004 | 多部分目标 -y 列表与目标名匹配 | MUST |
| OS-BUILD-005 | 跨子系统开关在顶层/arch Makefile | MUST |
| OS-BUILD-006 | need-builtin/need-modorder 以传入值为准 | MUST |
| OS-BUILD-007 | 禁止手动删 .cmd 强制重建 | MUST |
| OS-BUILD-008 | 用 make V=2 调试误重建 | SHOULD |
| OS-BUILD-009 | Makefile.airymaxos 版本变更经评审 | MUST |
| OS-BUILD-010 | version.h 经 filechk 生成 | MUST |
| OS-BUILD-011 | 链接前 built-in.a 就绪 | MUST |
| OS-BUILD-012 | CI 三配置验证 descending 闭合 | MUST |

---

## 10. 相关文档

### 10.1 同卷文档

- `README.md`（构建系统模块主索引）
- `02-kconfig-system.md`（Kconfig 配置系统——CONFIG_* 宏的来源）
- `03-makefile-patterns.md`（Makefile 模式与惯用法）
- `04-module-building.md`（可加载模块构建链）
- `06-airymaxos-build.md`（agentrt-linux 多仓多语言构建）

### 10.2 上游与跨卷文档

- `50-engineering-standards/06-toolchain-and-automation.md`（CI/CD 与多矩阵构建，OS-STD-032/OS-STD-072）
- `50-engineering-standards/04-engineering-philosophy.md`（IRON-9 v2 同源且部分代码共享原则定义）
- `10-architecture/02-five-dimensional-principles.md`（五维正交 24 原则定义）
- `60-driver-model/README.md`（驱动子系统构建约定）

### 10.3 参考材料

- `Linux 6.6 内核源码 Kbuild`（顶层 descending 入口）
- `Linux 6.6 内核源码 Makefile`（顶层构建系统、版本注入段）
- `Linux 6.6 内核源码 scripts/Kbuild.include`（if_changed / filechk / build 宏定义）
- `Linux 6.6 内核源码 scripts/Makefile.build`（descending 与 built-in.a 规则）
- `Linux 6.6 内核源码 scripts/Makefile.lib`（obj-y/m 规范化）

---

## 11. 文档版本与维护

- **当前版本**: v0.1.1/ v1.0.1（开发中）
- **维护者**: agentrt-linux 构建系统 SIG（待成立，详见 07 卷维护者制度）
- **变更流程**: 本卷变更必须经过 RFC → 评审 → ACC 验收流程；Kbuild 机制层变更需同步评估对 8 子仓构建链的影响。
- **回顾周期**: 随 Linux 6.6 内核基线 LTS 更新季度回顾 + agentrt-linux 大版本年度回顾。
- **0.1.1 范围**: README + 01 + 02（3 文档）。
- **1.0.1 范围**: 完成全部 8 文档并实施构建系统工程标准。

---

## 附录 A: 接口定义

> **附录定位**： 本附录汇集 Kbuild 递归构建系统所需的完整接口契约，供直接参照实现。所有数据结构与构建规则对齐 Linux 6.6 `Kbuild`、`Makefile`、`scripts/Kbuild.include`、`scripts/Makefile.build`、`scripts/Makefile.lib` 及 `include/airymax/build_types.h`（agentrt-linux 内核内部头文件，[IND] 独立层，agentrt 用户态不共享）。Kbuild 本质为 Make 宏与规则体系，故 A.1 以 C 结构体建模构建目标内部表示，A.2/A.3 以 Make 宏签名与变量定义为主。

### A.1 核心数据结构

#### A.1.1 build_target — 构建目标描述

```c
/**
 * struct build_target - Kbuild 单个构建目标的内部表示
 *
 * @name:       目标名（如 "airymax-ipc.o"、"vmlinux"、"built-in.a"）
 * @type:       目标类型（见 enum build_target_type）
 * @deps:       依赖目标链表头（struct build_target_dep 节点）
 * @obj_list:   组成该目标的 .o 文件名数组（对应 <name>-y / <name>-objs 展开）
 * @obj_count:  obj_list 元素数
 * @cmd_str:    当前命令行指纹（if_changed 比对依据，存 .cmd 文件）
 * @flags:      目标标志位（BUILD_FL_FORCE 等对应 FORCE 伪目标）
 *
 * 对齐 Linux 6.6 scripts/Makefile.build 的目标求值模型
 * （agentrt-linux 专属建模，描述 Make 规则在 C 侧的等价表示）
 */
enum build_target_type {
    BUILD_T_BUILTIN = 0,  /* obj-y 内建目标，归入 built-in.a */
    BUILD_T_MODULE  = 1,  /* obj-m 可加载模块，产物为 .ko */
    BUILD_T_ARCHIVE  = 2,  /* built-in.a 归档目标 */
    BUILD_T_IMAGE   = 3,  /* vmlinux / bzImage 镜像目标 */
};

#define BUILD_FL_FORCE  0x0001  /* FORCE 伪目标：每次求值强制重建 */

struct build_target {
    const char              *name;
    enum build_target_type  type;
    struct hlist_head        deps;
    const char            **obj_list;
    int                      obj_count;
    const char              *cmd_str;
    unsigned long            flags;
} __attribute__((aligned(64)));
```

#### A.1.2 kbuild_module — 内核模块构建信息

```c
/**
 * struct kbuild_module - 内核模块（.ko）构建元信息
 *
 * @name:        模块名（不含 .ko 后缀，如 "airymax-ipc"）
 * @objs:        组成该模块的 .o 文件列表（<name>-y / <name>-objs 展开）
 * @obj_count:   objs 元素数
 * @modorder:    在 modules.order 中的序号（决定加载顺序）
 * @srcdir:      模块源码目录（$(obj) 前缀已展开）
 * @kbuild_file: 所属 Kbuild/Makefile 路径
 * @is_multi:    是否为多部分目标（airymax-ipc-y 形式）
 *
 * 对齐 Linux 6.6 scripts/Makefile.lib 的 modname/multi-objs 规范化
 * （agentrt-linux 专属建模）
 */
struct kbuild_module {
    const char        *name;
    const char       **objs;
    int                obj_count;
    int                modorder;
    const char        *srcdir;
    const char        *kbuild_file;
    bool               is_multi;
};
```

#### A.1.3 obj_config — obj-y/m/n 三态配置

```c
/**
 * struct obj_config - 子目录 obj-y/obj-m/obj-n 三态配置快照
 *
 * @builtin:    内建目标列表（obj-y 规范化后，含 foo/ → foo/built-in.a）
 * @modules:    模块目标列表（obj-m 规范化后，含 foo/ → foo/modules.order）
 * @excluded:   不构建目标列表（obj-n，仅文档化，Make 视为空）
 * @builtin_cnt: builtin 元素数
 * @module_cnt: modules 元素数
 * @subdir_ym:  递归 descending 子目录集合（subdir-ym 合并）
 *
 * 对齐 Linux 6.6 scripts/Makefile.lib 的 obj-* 规范化逻辑
 * （规范化规则：obj-m 减去 obj-y 重复项；foo/ 替换为归档/modules.order）
 */
struct obj_config {
    const char       **builtin;
    const char       **modules;
    const char       **excluded;
    int                builtin_cnt;
    int                module_cnt;
    const char       **subdir_ym;
    int                subdir_cnt;
};
```

#### A.1.4 compile_command — 编译命令条目

```c
/**
 * struct compile_command - compile_commands.json 单条编译命令
 *
 * @directory:  编译执行目录（展开后的 $(obj)）
 * @file:       被编译源文件绝对路径
 * @command:    完整编译命令行（含 ccflags-y / subdir-ccflags-y）
 *
 * 对齐 Linux 6.6 scripts/Makefile.build 的 compile_commands.json 生成
 * （由 `make compile_commands.json` 目标输出，供 clangd / IDE 索引）
 */
struct compile_command {
    const char *directory;
    const char *file;
    const char *command;
};
```

### A.2 核心函数签名

#### A.2.1 if_changed — 增量构建核心宏

```makefile
# /**
#  * if_changed - 仅当依赖或命令行变化时才重建目标
#  * @rule: 规则名（如 compile、link、ar_builtin）
#  *
#  * 工作流：检查 FORCE + .cmd 文件中记录的上次命令行；
#  * 若命令行指纹变化或任一依赖比目标新，执行 cmd_$(rule)，
#  * 否则跳过（输出 '  CC     ' 提示时才真正编译）。
#  *
#  * 返回: 0 成功；非零表示命令执行失败（exit code 透传）
#  *
#  * 对齐 Linux 6.6 scripts/Kbuild.include
#  */
define if_changed
	$(check-FORCE)
	$(Q)if test -n "$(any-prereq)$(arg-check)" ; then \
		$(echo-cmd) $(cmd_$(1)); \
		printf '%s\n' 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd; \
	fi
endef
# @since 0.1.1
```

#### A.2.2 filechk — 版本注入与文件检查

```makefile
# /**
#  * filechk - 比较生成文件新旧内容，仅在变化时覆盖目标
#  * @name: filechk_<name> 宏名（如 version.h）
#  *
#  * 工作流：先写临时文件 $(tmp-target)；若目标不存在或
#  * cmp -s 内容不同则 mv 覆盖，否则保持原文件时间戳不变
#  * （避免无意义重编下游目标，体现 E-3 资源确定性）。
#  *
#  * 返回: 0 成功；非零表示 filechk_$(name) 执行失败
#  *
#  * 对齐 Linux 6.6 scripts/Kbuild.include
#  */
define filechk
	$(check-FORCE)
	$(Q)set -e; \
	mkdir -p $(dir $@); \
	trap "rm -f $(tmp-target)" EXIT; \
	{ $(filechk_$(1)); } > $(tmp-target); \
	if [ ! -r $@ ] || ! cmp -s $@ $(tmp-target); then \
		$(kecho) '  UPD     $@'; \
		mv -f $(tmp-target) $@; \
	fi
endef
# @since 0.1.1
```

#### A.2.3 build_target_compile — 构建目标编译

```c
/**
 * build_target_compile - 编译单个构建目标
 * @target:  构建目标描述（含 obj_list 与 cmd_str 指纹）
 * @ctx:     构建上下文（srctree/objtree/arch 等）
 *
 * 依次编译 obj_list 中每个 .c → .o，调用 if_changed 比对命令指纹；
 * 多部分目标在 .o 全部就绪后由 ld_multi_m 链接为 <name>.o。
 *
 * 返回: 0 成功；<0 失败（负 errno）
 *   -ENOMEM: 目标/对象内存分配失败
 *   -EINVAL: 目标类型非法或 obj_list 为空
 *   -EIO:   编译器返回非零退出码
 *
 * @since 1.0.1
 */
int build_target_compile(struct build_target *target,
                         struct build_context *ctx);
```

#### A.2.4 kbuild_recursive — 递归构建入口

```c
/**
 * kbuild_recursive - 递归 descending 到子目录构建
 * @subdir:  子目录名（相对 $(obj)）
 * @ctx:     构建上下文
 *
 * 进入 $(obj)/$(subdir)/，读取其 Kbuild/Makefile（kbuild-file 宏优先
 * 选 Kbuild），求解 obj_config 三态，对 builtin/modules 分别构建，
 * 递归进入 subdir-ym 子目录。
 *
 * 返回: 0 成功；<0 失败（负 errno）
 *   -ENOENT: 子目录不存在或缺 Kbuild/Makefile
 *   -EIO:   子目录构建命令失败
 *
 * 对齐 Linux 6.6 scripts/Makefile.build 的 $(subdir-builtin) 规则
 * @since 1.0.1
 */
int kbuild_recursive(const char *subdir, struct build_context *ctx);
```

### A.3 错误码与宏定义

#### A.3.1 obj-y / obj-m / obj-n 三态门控语义

```makefile
# /**
#  * obj-y / obj-m / obj-n - Kbuild 三态门控变量
#  *
#  * obj-y: 内建目标，归入 built-in.a → vmlinux 静态链接
#  * obj-m: 可加载模块，产物为 .ko，经 modules_install 部署
#  * obj-n: 不构建（通常省略，仅在显式文档化时写出）
#  *
#  * 规范化规则（scripts/Makefile.lib）：
#  *   1. obj-m := $(filter-out $(obj-y),$(obj-m))  -- 移除与 obj-y 重复项
#  *   2. obj-y 中 foo/ → foo/built-in.a
#  *   3. obj-m 中 foo/ → foo/modules.order
#  *
#  * 门控惯用法：obj-$(CONFIG_FOO) += foo.o
#  *   CONFIG_FOO=y → obj-y += foo.o（内建）
#  *   CONFIG_FOO=m → obj-m += foo.o（模块）
#  *   CONFIG_FOO=n → obj- += foo.o（空，不构建）
#  *
#  * 对齐 Linux 6.6 scripts/Makefile.lib
#  */
# 门控语义常量（agentrt-linux 专属建模）
#define OBJ_STATE_BUILTIN  'y'  /* 内建 */
#define OBJ_STATE_MODULE   'm'  /* 模块 */
#define OBJ_STATE_DISABLED 'n'  /* 不构建 */
```

#### A.3.2 KBUILD_* 环境变量

```makefile
# /**
#  * KBUILD_* - Kbuild 环境变量与构建路径控制
#  *
#  * 对齐 Linux 6.6 顶层 Makefile 与 scripts/Kbuild.include
#  */

# KBUILD_SRC: 源码树根（$(srctree) 别名，out-of-tree 构建时指向源码目录）
#   默认: 构建发起目录；外部模块构建时须显式设置
KBUILD_SRC ?= $(CURDIR)

# KBUILD_EXTMOD: 外部模块源码目录（`make M=drivers/external` 等价）
#   设置后 Kbuild 进入外部模块模式，不构建 vmlinux/built-in.a
KBUILD_EXTMOD ?=

# KBUILD_VERBOSE: 决定 Q（quiet）是否抑制命令回显
#   KBUILD_VERBOSE=1 输出完整命令行；=0 仅输出简短提示（CC/LD/AR）
KBUILD_VERBOSE ?= 0

# KBUILD_MODULES: 是否构建模块（`make modules` 时置 1）
KBUILD_MODULES :=

# KBUILD_BUILTIN: 是否构建内建（vmlinux/built-in.a）
KBUILD_BUILTIN :=

# @since 0.1.1
```

#### A.3.3 Makefile.airymaxos 自定义目标

```makefile
# SPDX-License-Identifier: GPL-2.0
# /**
#  * Makefile.airymaxos - agentrt-linux 供应商版本定义
#  *
#  * 仅承载版本四元组（OS-STD-049）；特性开关走 Kconfig，禁止塞入此文件。
#  * 顶层 Makefile 经 `include $(srctree)/Makefile.airymaxos` 引入，
#  * 再由 filechk_version.h 注入为 C 宏（AIRYMAXOS_LTS / _MAJOR / _MINOR）。
#  *
#  * 对齐 Linux 6.6 顶层 Makefile 的 VERSION/PATCHLEVEL/SUBLEVEL 机制
#  * @since 0.1.1
#  */
AIRYMAXOS_LTS     = 0
AIRYMAXOS_MAJOR   = 9999
AIRYMAXOS_MINOR   = 0
AIRYMAXOS_RELEASE = ""

# 注入的 C 宏（由 filechk_version.h 生成到 include/generated/uapi/linux/version.h）
# #define AIRYMAXOS_LTS           <LTS>
# #define AIRYMAXOS_MAJOR         <MAJOR>
# #define AIRYMAXOS_MINOR         <MINOR>
# #define AIRYMAXOS_VERSION(a,b)  (((a) << 8) + (b))
# #define AIRYMAXOS_VERSION_CODE  (MAJOR * 256 + MINOR)
# #define AIRYMAXOS_RELEASE       "<RELEASE>"
```

#### A.3.4 构建错误码

```c
/**
 * Kbuild 构建错误码（agentrt-linux 专属，对齐 Linux 6.6 errno 语义）
 *
 * 构建失败以 Make 退出码形式上抛；以下为 C 侧建模的语义错误码。
 */
#define KBUILD_OK            0          /* 构建成功 */
#define KBUILD_E_NOENT       (-ENOENT)  /* 源文件/Kbuild 缺失 */
#define KBUILD_E_INVAL       (-EINVAL)  /* 目标类型非法或 obj_list 为空 */
#define KBUILD_E_NOMEM       (-ENOMEM)  /* 目标/对象内存分配失败 */
#define KBUILD_E_IO          (-EIO)     /* 编译器/链接器返回非零退出码 */
#define KBUILD_E_DEPEND      (-EDEADLK) /* 循环依赖（descending 死锁） */
#define KBUILD_E_CONFIG      (-ENOTSUP) /* CONFIG_* 未声明但被 obj-$(CONFIG_*) 引用 */
```

---

> **文档结束** | agentrt-linux 构建系统第 1 卷 | Kbuild 递归构建详解 | 共 11 章节 + 规则编号汇总
