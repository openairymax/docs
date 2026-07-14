Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）构建系统设计

> **文档定位**： agentrt-linux（AirymaxOS）构建系统工程设计主索引
> **版本**： 0.1.1
> **最后更新**： 2026-07-13
> **同源映射**： agentrt `cmake/`（伞仓直属 5 模块）+ Linux 6.6 Kbuild 系统
> **理论根基**： Linux Kbuild 递归构建 + Airymax E-7 文档即代码

---

## 1. 模块定位

agentrt-linux 构建系统是连接源代码与可分发产物的核心工程基础设施。它继承 Linux Kbuild 系统 30+ 年沉淀的工程思想（递归构建 + `if_changed` 增量构建 + `obj-$(CONFIG_*)` 配置门控），并在其上扩展智能体操作系统的多语言、多仓、多平台构建需求。

### 1.1 核心机制

| 机制 | 职责 | Linux 来源 |
|------|------|------------|
| **Kbuild 递归构建** | 顶层 Makefile descending 到子系统 | `Kbuild` + `Makefile` |
| **if_changed 增量构建** | 仅重建变更的文件 | `scripts/Kbuild.include` |
| **filechk 版本注入** | 生成版本头文件 | `Makefile.airymaxos` |
| **Kconfig 配置门控** | `obj-$(CONFIG_*)` 控制 | `Kconfig` + `lib/Kconfig.airymaxos` |
| **模块构建** | 可加载模块 `.ko` | `scripts/Makefile.modpost` |

### 1.2 agentrt-linux 扩展

- **多语言构建**：C（Kbuild）+ Rust（cargo）+ Python（poetry）+ TypeScript（tsc/webpack）
- **多仓集成**：8 子仓 + agentrt 同源仓库的统一构建入口
- **多平台目标**：x86_64 / aarch64 / riscv64 + 跨平台用户态（Linux/macOS/Windows）

---

## 2. 核心设计原则

### 2.1 递归构建（Descending Build）

顶层 `Kbuild` 定义子系统编译入口顺序：
```makefile
obj-y += kernel/
obj-y += mm/
obj-y += fs/
obj-$(CONFIG_BLOCK) += block/
obj-$(CONFIG_IO_URING) += io_uring/
obj-$(CONFIG_RUST) += rust/
```

每个子系统目录有自己的 `Kbuild`/`Makefile`，递归 descending 构建。

### 2.2 if_changed 增量构建（A-4 完美主义）

```makefile
# scripts/Kbuild.include
if_changed = $(if $(strip $(any-prereq) $(arg-check)), \
    @set -e; $(echo-cmd) $(cmd_$(1)); \
    printf '%s\n' 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd)
```

仅当依赖或命令行参数变化时才重建，确保最小构建时间。

### 2.3 filechk 版本注入

`Makefile.airymaxos` 仅 5 行：
```makefile
# SPDX-License-Identifier: GPL-2.0
AIRYMAXOS_LTS = 0
AIRYMAXOS_MAJOR = 9999
AIRYMAXOS_MINOR = 0
AIRYMAXOS_RELEASE = ""
```

通过 `filechk_version.h` 优雅注入版本宏，**特性开关与版本号解耦**。

### 2.4 Kconfig 配置门控

`obj-$(CONFIG_XXX)` 模式使配置开关直接控制构建：
```makefile
obj-$(CONFIG_GMEM) += gmem.o
obj-$(CONFIG_SHARE_POOL) += share_pool.o
```

`lib/Kconfig.airymaxos` 作为通用型配置聚合点，子系统特定配置仍嵌入各原生 Kconfig。

### 2.5 模块构建（可加载模块）

```bash
make modules    # 构建所有模块
make modules_install  # 安装到 /lib/modules/$(uname -r)
```

模块使用 `MODULE_LICENSE()`、`MODULE_AUTHOR()`、`MODULE_DESCRIPTION()` 元数据。

---

## 3. 文档索引

```
70-build-system/
├── README.md                       # 本文件
├── 01-kbuild-system.md             # Kbuild 递归构建详解
├── 02-kconfig-system.md            # Kconfig 配置系统
├── 03-makefile-patterns.md         # Makefile 模式与惯用法
├── 04-module-building.md           # 可加载模块构建
├── 05-cross-compilation.md         # 跨平台编译
├── 06-airymaxos-build.md           # agentrt-linux 多仓多语言构建
├── 07-ci-integration.md            # CI/CD 集成
└── 08-build-reproducibility.md     # 可重现构建
```

### 3.1 0.1.1 版本范围

README + 01 + 02 文档（3 文档奠基），确立构建系统设计框架与 Kbuild/Kconfig 核心机制。其余 6 文档（03-makefile-patterns 至 08-build-reproducibility）在 1.0.1 版本完成。

### 3.2 1.0.1 版本范围

完成剩余 6 文档（03-makefile-patterns 至 08-build-reproducibility），实施构建系统工程标准，进行代码级验证。

---

## 4. agentrt-linux 专属扩展

### 4.1 多仓构建集成

agentrt-linux 8 子仓 + agentrt 同源仓库的统一构建入口：
- **伞仓**：`airymaxhub` 提供顶层 `Makefile` 入口
- **管理仓**：`agentrt`/`sdk`/`ecosystem`/`products` 各自独立构建
- **叶子仓**：21 个叶子仓通过 submodule 集成

### 4.2 多语言构建协调

| 语言 | 构建工具 | 产物 |
|------|---------|------|
| C | Kbuild + gcc/clang | `.o` / `.ko` / vmlinux |
| Rust | cargo + rustfmt + clippy | `.rlib` / `.so` |
| Python | poetry + black + mypy | wheel / sdist |
| TypeScript | tsc + webpack + prettier | `.js` / `.d.ts` |

### 4.3 同源 agentrt 构建集成

agentrt 的 `cmake/` 目录（5 模块）在伞仓直属，提供跨模块共享的 CMake 工具链。agentrt-linux 内核态使用 Kbuild，用户态使用 CMake（与 agentrt 一致）。

### 4.4 版本注入机制

agentrt-linux 采用基于 `Makefile.airymaxos` 的供应商版本注入：
```makefile
# Makefile.airymaxos
AIRYMAXOS_MAJOR = 0
AIRYMAXOS_MINOR = 1
AIRYMAXOS_PATCH = 1
AIRYMAXOS_RELEASE = ""
```

通过 `filechk` 生成 `airy_version.h`，特性开关与版本号解耦。

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **A-4 完美主义** | if_changed 增量构建确保最小重建 |
| **E-3 资源确定性** | 可重现构建（reproducible builds） |
| **E-7 文档即代码** | Kconfig 帮助文本即文档 |
| **K-4 可插拔策略** | CONFIG_* 配置门控使特性可插拔 |
| **S-2 层次分解** | 递归构建按目录层次组织 |

---

## 6. 相关文档

- `50-engineering-standards/06-toolchain-and-automation.md`（CI/CD 集成）
- `60-driver-model/README.md`（驱动构建）
- `20-modules/01-kernel.md`（kernel 子仓构建）
- `130-roadmap/02-milestones-and-timeline.md`（构建系统里程碑）

---

## 7. 参考材料

- Linux 6.6 `Kbuild`（顶层 descending 构建）
- Linux 6.6 `Makefile`（顶层构建系统）
- Linux 6.6 `Makefile.airymaxos`（版本注入机制）
- Linux 6.6 `scripts/Kbuild.include`（Kbuild 核心宏）
- Linux 6.6 `lib/Kconfig.airymaxos`（配置聚合点）

---

> **文档结束** | README + 01 + 02
