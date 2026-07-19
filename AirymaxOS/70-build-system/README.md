Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）构建系统设计
> **文档定位**：agentrt-linux（AirymaxOS）构建系统工程设计主索引（Kconfig + Kbuild + CMake + RPM spec + airy_defconfig）\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[AirymaxOS 总览](../README.md)\
> **同源映射**：agentrt `cmake/`（伞仓直属 5 模块）+ Linux 6.6 Kbuild 系统\
> **理论根基**：Linux Kbuild 递归构建 + Airymax E-7 文档即代码 + SSoT v2 单一权威源

---

## 1. 模块概述

agentrt-linux 构建系统是连接源代码与可分发产物的核心工程基础设施。它继承 Linux Kbuild 系统 30+ 年沉淀的工程思想（递归构建 + `if_changed` 增量构建 + `obj-$(CONFIG_*)` 配置门控），并在其上扩展智能体操作系统的多语言、多仓、多平台构建需求。本目录覆盖五方面职责：

1. **Kconfig 配置系统**：`lib/Kconfig.airymaxos` 作为通用型配置聚合点，子系统特定配置嵌入各原生 Kconfig；`obj-$(CONFIG_*)` 模式使配置开关直接控制构建。
2. **Kbuild 递归构建**：顶层 `Kbuild` 定义子系统 descending 顺序，`scripts/Kbuild.include` 提供 `if_changed` 增量构建宏，`Makefile.airymaxos` 通过 `filechk` 注入版本号。
3. **CMake 用户态构建**：用户态守护进程、SDK、跨平台用户态运行时使用 CMake（与 agentrt `cmake/` 5 模块工具链一致）。
4. **RPM spec 打包**：内核包 / 服务包 / SDK 包采用 RPM spec + dnf 包管理（对齐 `190-distribution/`）。
5. **airy_defconfig 统一配置**：`airy_defconfig` 是 A-UCS（Unified Configuration Subsystem）模块在构建期的物理承载——所有 Unify Design 五模块的 [SC] 配置项在 defconfig 中固化为不可妥协基线。

### 1.1 核心机制

| 机制 | 职责 | Linux 来源 |
|------|------|------------|
| **Kbuild 递归构建** | 顶层 Makefile descending 到子系统 | `Kbuild` + `Makefile` |
| **if_changed 增量构建** | 仅重建变更的文件 | `scripts/Kbuild.include` |
| **filechk 版本注入** | 生成版本头文件 | `Makefile.airymaxos` |
| **Kconfig 配置门控** | `obj-$(CONFIG_*)` 控制 | `Kconfig` + `lib/Kconfig.airymaxos` |
| **airy_defconfig** | A-UCS 构建期配置基线 | `arch/$(SRCARCH)/configs/airy_defconfig` |
| **模块构建** | 可加载模块 `.ko` | `scripts/Makefile.modpost` |

### 1.2 agentrt-linux 扩展

- **多语言构建**：C（Kbuild）+ Rust（cargo）+ Python（poetry）+ TypeScript（tsc/webpack）
- **多仓集成**：8 子仓 + agentrt 同源仓库的统一构建入口
- **多平台目标**：x86_64 / aarch64 / riscv64 + 跨平台用户态（Linux/macOS/Windows）
- **airy_defconfig 锁定五大技术选型**：sched_ext 关闭、page flipping 关闭、BPF LSM 关闭、DMA 一致性内存关闭、IRON-9 v3 [DSL] 降级生存层开启

---

## 2. 技术选型声明

agentrt-linux v1.0 构建系统在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度遵循 [AirymaxOS 总览](../README.md) §2 的不可妥协基线。构建系统通过 `airy_defconfig` 与 Kconfig 选项在编译期锁定五大选型，任何偏离均会导致构建失败。五个维度的选型在本目录的具体落地如下：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 在本目录的落地 |
|---|---------|---------|----------------|--------------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 `SCHED_EXT=7` 调度类） | `airy_defconfig` 强制 `# CONFIG_SCHED_EXT is not set`；Kconfig 中 sched_ext 依赖项被 disable |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | `airy_defconfig` 强制 `CONFIG_IO_URING=y`；page flipping 相关代码路径不编译 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 `airy_lsm` 通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | `airy_defconfig` 强制 `# CONFIG_BPF_LSM is not set`；`CONFIG_SECURITY_AIRY_LSM=y` 编译纯 C LSM 模块 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | `airy_defconfig` 不启用 `DMA_CMA` 一致性路径；共享内存构建目标绑定 `alloc_pages + mmap` 实现源文件 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | （v2 三层模型升级为 v3 四层模型，新增 [DSL] 降级生存层） | CI 流水线强制校验 `include/uapi/linux/airymax/*.h` 共 10 个 [SC] 头文件逐字节一致；`AIRY_SC_FALLBACK` 宏由构建系统在 [SC] 校验失败时自动注入（[DSL] 第四层） |

### 2.1 IRON-9 v3 四层模型在构建系统的归属

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 落地文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| [SC] 头文件逐字节校验 | ● | — | — | — | [03-ci-cd-pipeline.md](03-ci-cd-pipeline.md) |
| `airy_defconfig` 配置基线 | — | — | ● | — | [02-kconfig-system.md](02-kconfig-system.md) |
| `AIRY_SC_FALLBACK` 注入 | — | — | — | ● | [02-kconfig-system.md](02-kconfig-system.md) |
| Kbuild 递归构建 | — | — | ● | — | [01-kbuild-system.md](01-kbuild-system.md) |
| CI 强制校验流水线 | — | — | ● | — | [03-ci-cd-pipeline.md](03-ci-cd-pipeline.md) |

---

## 3. 文档索引

本目录现有文档（v1.0 范围）：

```
70-build-system/
├── README.md                       # 本文件（v1.0）
├── 01-kbuild-system.md             # Kbuild 递归构建详解 + Makefile.airymaxos 版本注入
├── 02-kconfig-system.md            # Kconfig 配置系统 + airy_defconfig A-UCS 基线
└── 03-ci-cd-pipeline.md            # CI/CD 流水线（[SC] 逐字节校验 + 四层模型校验）
```

| # | 文档 | 版本 | 内容概要 |
|---|------|------|---------|
| — | [README.md](README.md) | v1.0 | 构建系统主索引（本文件） |
| 1 | [01-kbuild-system.md](01-kbuild-system.md) | v1.0 | Kbuild 递归构建、`if_changed` 增量构建、`Makefile.airymaxos` 版本注入、模块构建 |
| 2 | [02-kconfig-system.md](02-kconfig-system.md) | v1.0 | Kconfig 配置门控、`lib/Kconfig.airymaxos` 聚合点、`airy_defconfig` A-UCS 构建期基线（锁定五大技术选型） |
| 3 | [03-ci-cd-pipeline.md](03-ci-cd-pipeline.md) | v1.0 | CI/CD 流水线、`sc-dual-ci.yml` [SC] 逐字节校验、`ssot-validate.yml` 四层模型归属校验 |

### 3.1 后续规划文档（1.0.1 版本）

以下文档在 1.0.1 版本完成，不在 v1.0 范围内：

- `04-makefile-patterns.md`：Makefile 模式与惯用法
- `05-cross-compilation.md`：跨平台编译（x86_64/aarch64/riscv64）
- `06-airymaxos-build.md`：agentrt-linux 多仓多语言构建集成
- `07-cmake-userland.md`：用户态 CMake 构建工具链（对齐 agentrt `cmake/`）
- `08-rpm-spec.md`：RPM spec 打包规范
- `09-build-reproducibility.md`：可重现构建

---

## 4. Airymax Unify Design 映射

本目录与 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的关系详见 [AirymaxOS 总览](../README.md) §5。构建系统主要承载 **A-UCS**（airy_defconfig 是 A-UCS 模块的构建配置），其余四模块通过 Kconfig 选项在编译期锁定。

| Unify 模块 | 关系 | 在本目录的体现 |
|-----------|------|--------------|
| **A-UEF** | 辅助 | A-UEF 的 [SC] `error.h` 头文件由 CI 流水线逐字节校验（`sc-dual-ci.yml`） |
| **A-ULP** | 辅助 | A-ULP 的 [SC] `log_types.h` 头文件由 CI 流水线逐字节校验；128B 日志记录格式在 defconfig 中固化 |
| **A-UCS** | **核心** | `airy_defconfig` 是 A-UCS 模块在构建期的物理承载——所有 [SC] 配置项（IPC 消息头大小、Ring Buffer 大小、魔数）在 defconfig 中固化为不可妥协基线；sysctl/JSON 双向热重载的语义同源配置（[SS]）由 Kconfig 选项控制启用 |
| **A-ULS** | 辅助 | A-ULS 的纯 C LSM 模块（`airy_lsm`）通过 `CONFIG_SECURITY_AIRY_LSM=y` 编译；Agent 8 态生命周期 [SC] `sched.h` 由 CI 校验 |
| **A-IPC** | 辅助 | A-IPC 的 [SC] `ipc.h` 头文件由 CI 流水线逐字节校验；`IORING_OP_URING_CMD` 路径通过 `CONFIG_IO_URING=y` 强制启用 |

### 4.1 A-UCS 权威源引用

- A-UCS 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §6
- A-UCS 模块设计：[../20-modules/11-unified-config.md](../20-modules/11-unified-config.md)
- [SC] 类型桥接：[../50-engineering-standards/11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md)

---

## 5. 相关文档

- [AirymaxOS 总览](../README.md)：v1.0 技术选型声明 + 20 子目录索引
- [架构设计层](../10-architecture/README.md)：Unify Design 总纲 + IRON-9 v3 四层模型
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md)：Airymax Unify Design 总纲（SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（SSoT）
- [20-modules/11-unified-config.md](../20-modules/11-unified-config.md)：A-UCS 统一配置管理体系（airy_defconfig 权威源）
- [20-modules/01-kernel.md](../20-modules/01-kernel.md)：kernel 子仓构建
- [50-engineering-standards/09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)：SSoT v2 单一权威源注册表
- [50-engineering-standards/11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md)：[SC] 头文件类型桥接规则
- [60-driver-model/README.md](../60-driver-model/README.md)：驱动构建（Kbuild 模块构建）
- [80-testing/README.md](../80-testing/README.md)：CI 中的测试集成
- [120-development-process/README.md](../120-development-process/README.md)：CI/CD 流程规范
- [190-distribution/01-rpm-packaging.md](../190-distribution/01-rpm-packaging.md)：RPM 打包规范

---

## 6. 参考材料

- Linux 6.6 `Kbuild`（顶层 descending 构建）
- Linux 6.6 `Makefile`（顶层构建系统）
- Linux 6.6 `Makefile.airymaxos`（版本注入机制）
- Linux 6.6 `scripts/Kbuild.include`（Kbuild 核心宏）
- Linux 6.6 `lib/Kconfig.airymaxos`（配置聚合点）
- Linux 6.6 `arch/*/configs/*_defconfig`（defconfig 格式参考）

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，README + 01 + 02 文档奠基，确立 Kbuild/Kconfig 核心机制 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap / IRON-9 v3 四层模型五大技术选型声明（通过 `airy_defconfig` 编译期锁定）；新增 Airymax Unify Design 映射（A-UCS airy_defconfig 为核心）；文档索引对齐实际目录文件（含 03-ci-cd-pipeline.md） |

---

> **文档结束** | agentrt-linux 构建系统设计 v1.0 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
