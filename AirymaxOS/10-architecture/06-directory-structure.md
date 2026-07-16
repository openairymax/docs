Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 系统性目录结构设计

> **文档定位**：agentrt-linux（AirymaxOS）源码目录结构设计文档\
> **文档版本**：0.2.4（对齐闭源 v0.2.4 生产级修正版）/ 1.0.1 M1（代码落地）\
> **最后更新**：2026-07-15\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **来源**：基于 Linux 6.6 内核基线源码参考 + seL4 源码参考 + 8 子仓架构\
> **铁律依据**：IRON-9 v2 三层模型 + ES-OLK-1\~13 工程思想 + ADR-014 微内核来源单一化 + OS-IRON-013 8 子仓 submodule + OS-IRON-014 \[SC] 共享契约层单一数据源 + OS-IRON-015 编号管理元规则\
> **SSoT 依赖声明**：本文档涉及的所有规则编号（OS-IRON-001\~015、OS-KER-xxx、OS-STD-xxx 等）的**唯一权威来源**为 [`50-engineering-standards/09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md)。本文档不是规则编号 SSoT，仅作为目录结构设计的技术阐述载体。当本文档与 SSoT 注册表冲突时，以 SSoT 注册表为准。

***

## 1. 设计原则

agentrt-linux 源码目录结构遵循六大设计原则，每条原则都有源码级标杆依据：

| # | 原则                                | 来源                                             | 标杆证据                                                                                                                             |
| - | --------------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 1 | ES-OLK-1 关注点分离                    | Linux 6.6 内核基线 `mm/`/`kernel/`/`fs/`/`net/` 分离 | 顶层目录各司其职                                                                                                                         |
| 2 | ES-OLK-2 架构正交                     | Linux 6.6 内核基线 `arch/` 是唯一架构相关根目录              | `arch/x86/`/`arch/arm64/`/`arch/riscv/`                                                                                          |
| 3 | ES-OLK-3/6 UAPI 分离                | Linux 6.6 内核基线 `include/uapi/` 是用户空间 ABI 唯一来源  | `include/uapi/linux/`                                                                                                            |
| 4 | seL4 单编译单元（可选）                    | seL4 `kernel_all.c` 拼接模式                       | `CMakeLists.txt:355-363`                                                                                                         |
| 5 | IRON-9 v2 三层模型                    | \[SC] 共享契约层物理隔离                                | [50-engineering-standards/120-cross-project-code-sharing.md §1.1](../50-engineering-standards/120-cross-project-code-sharing.md) |
| 6 | **OS-IRON-013 8 子仓 submodule 管理** | 拆分为 8 个独立 leaf 仓，由管理仓通过 submodule 统一管理         | [04-engineering-philosophy.md §13](../50-engineering-standards/04-engineering-philosophy.md)                                     |

**关键约束**：

- **OS-IRON-014（\[SC] 单一数据源）**：\[SC] 6 个头文件**唯一物理宿主**于 `kernel/include/airymax/`，其他子仓通过 `-I../kernel/include` 引用，**禁止物理副本**。
- \[SC] 头文件必须 Tab 8 缩进（OS-FMT-001，对齐 Linux 6.6 内核基线 `01-kernel.md` 源码品味）。
- \[SC] 头文件必须最小 typedef（OS-STD-CODE-020，仅 5 种例外）。
- \[SC] 头文件变更必须双向 CI 校验（OS-IRON-008）。
- **生产级价值标准（v0.2.4 核心声明）**：一切以"生产级的 Linux 发行版"为价值标准——kernel/ 子仓采用**模型 A（完整 Linux 6.6 fork）**，含完整 Linux 6.6 源码树（\~60K 文件 1.6GB），Airymax 修改**直接写入上游目录**（如 `mm/airymax_mm.c`），不使用 `patches/` 隔离层。

> **代码组织模型（v0.2.4 裁决）**：AirymaxOS 采用"**直接写入上游源码树**"模型——所有 Airymax 代码直接放置在 Linux 6.6 内核源码树对应子系统中，对齐生产级 Linux 发行版实践（参考 `mm/dynamic_pool.c` 直接写入的工程惯例）。`20-modules/01-kernel.md` 中展示的 `patches/` 目录树为早期设计稿的概念分组，**不作为物理落地路径**。开发者应以下方树为准。

***

## 2. 管理仓 agentrt-linux 目录结构

管理仓（非 leaf 仓）聚合 8 个子模块，自身不含实现代码，仅含治理与文档索引。

**OS-IRON-013**：agentrt-linux 拆分为 8 个独立 git 仓库（leaf 仓），由 agentrt-linux 管理仓通过 `.gitmodules` + submodule 统一管理。8 子仓的目录结构设计基于 Linux 6.6 内核基线（24 顶层目录）+ seL4（`src/{api,kernel,object,fastpath,arch,machine,plat}`）+ Airymax 同源（atoms/corekern 复用）三大设计支柱。

```
agentrt-linux/                    # 管理仓（main 分支）
├── README.md                     # 英文 README（8 子仓矩阵 + 架构层次图）
├── README_zh.md                  # 中文 README
├── LICENSE                       # AGPL-3.0 + Apache-2.0 双许可证
├── NOTICE                        # 版权、商标、第三方声明
├── CONTRIBUTING.md               # 贡献指南
├── .gitmodules                   # 8 子模块定义（feature/official-hubs-01 分支）
├── .gitignore
│
├── kernel/                       # [submodule] kernel（裸名，对齐 .gitmodules）
├── services/                     # [submodule] services（裸名）
├── security/                     # [submodule] security（裸名）
├── memory/                       # [submodule] memory（裸名）
├── cognition/                    # [submodule] cognition（裸名）
├── cloudnative/                  # [submodule] cloudnative（裸名）
├── system/                       # [submodule] system（裸名）
├── tests-linux/                  # [submodule] tests-linux（裸名）
│
├── tools/                        # 跨子仓工具聚合
│   ├── checkpatch-airymax.sh     # 引用 kernel/scripts/checkpatch.pl 的聚合入口
│   ├── build-all.sh              # 跨 8 子仓统一构建脚本
│   └── README.md                 # 工具索引（各子仓 scripts/ 清单）
│
└── docs/                         # 软链接/引用 → umbrella docs/AirymaxOS/
                                  # （设计文档维护在 umbrella 仓库，管理仓仅引用）
```

**说明**：

- 管理仓的 `docs/` 是对 umbrella 仓库 `docs/AirymaxOS/` 的引用（软链接或文档指针），避免设计文档与代码分离维护。
- 管理仓的 `tools/` 是**跨子仓工具聚合入口**，对齐 Linux 6.6 内核基线顶层 `tools/` 实践——各子仓内部工具在自身 `scripts/`，管理仓 `tools/` 仅提供聚合入口与索引（如 `checkpatch-airymax.sh` 调用 `kernel/scripts/checkpatch.pl`），不重复实现。
- agentrt-linux 管理仓本身不创建 feature 分支（`main` only），所有子仓开发在 `feature/official-hubs-01` 分支进行。

### 2.1 8 子仓公共骨架

每个子仓根目录必须包含以下公共文件（双许可证 + 治理 + 元信息）：

```
<submodule>/
├── README.md              # 英文 README
├── README_zh.md           # 中文 README
├── LICENSE                # AGPL-3.0 + Apache-2.0 双许可证全文
├── NOTICE                 # 版权、商标、第三方声明
├── MAINTAINERS            # 维护者制度（R-03 落地，ES-OLK-8）
├── .gitignore
├── .clang-format          # C 代码格式（OS-FMT-001 Tab 8，仅 kernel/security/memory/cognition）
├── .editorconfig          # 编辑器统一配置
└── CONTRIBUTING.md        # 贡献指南（DCO + Signed-off-by + 审查流程）
```

**含 Rust 代码的子仓**（cognition + cloudnative）必须额外包含 `rustfmt.toml`（OS-FMT-001 4 空格）。

### 2.2 8 子仓类型与构建系统

8 子仓按代码形态分为 4 类，每类对应不同构建系统：

| 类型     | 子仓                                                                   | 构建系统                                  | 依据                           |
| ------ | -------------------------------------------------------------------- | ------------------------------------- | ---------------------------- |
| 内核态 C  | kernel / security / memory / cognition（kthread 部分）                   | Kbuild + Kconfig + Makefile（三者并存）     | ES-OLK-7（与 Linux 6.6 内核基线一致） |
| 用户态 C  | services（daemons）                                                    | Meson（推荐，现代用户态构建）                     | ES-SEL4-10 现代构建系统借鉴          |
| 用户态多语言 | cloudnative（Go/Rust/TS） / cognition（Wasm/Rust） / system（Rust/Python） | Cargo + Go Modules + tsc + Meson（按语言） | 各语言生态主流                      |
| 测试     | tests-linux                                                          | CTest + Cargo + Go + Shell            | 多语言测试聚合                      |

***

## 3. kernel/ 子仓目录结构（模型 A 完整 fork）

### 3.1 设计依据与模型 A 声明

**OS-IRON-013 落地**：kernel 子仓是 8 个 leaf 仓之一，通过 git submodule 接入管理仓。

**模型 A（完整 Linux 6.6 fork）**：kernel/ 子仓采用**完整 fork 模型**——含完整 Linux 6.6 源码树（\~60K 文件 1.6GB），Airymax 修改**直接写入上游目录**，不使用 `patches/` 隔离层。

| 维度                 | 模型 A 含义                                                              | 工程实践                                                                                                         |
| ------------------ | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| 子仓内容               | 完整 Linux 6.6 源码树 + Airymax 修改直接写入上游目录                                | kernel/ 含 `fs`/`net`/`block`/`drivers`/`mm`/`security`/`sound`/`certs`/`ipc`/`samples`/`usr`/`virt` 等全部上游子系统 |
| Airymax 专属代码       | `corekern/`（新增子目录）+ `include/airymax/`（\[SC] 头文件）+ `arch/<arch>/` 扩展 | Airymax 修改直接写入对应上游目录（如 `mm/airymax_mm.c`）                                                                    |
| patches/           | **删除**（模型 A 不需要补丁隔离层）                                                | Airymax 修改直接写入上游目录                                                                                           |
| Makefile.airymaxos | **保留**（定义 AIRYMAX\_LTS/MAJOR/MINOR/RELEASE 版本四元组）                    | 对齐 Linux 6.6 内核基线的版本管理惯例                                                                                     |
| 构建自包含性             | `make` 即可构建，无需外部基线源码树                                                | 子仓内自包含全部依赖                                                                                                   |
| 上游同步               | git rebase / merge 上游 release tag                                    | 标准 fork 维护流程                                                                                                 |

### 3.2 kernel/ 顶层目录树

```
kernel/                           # kernel 子仓（feature/official-hubs-01 分支）
├── README.md                     # 英文 README
├── README_zh.md                  # 中文 README
├── LICENSE                       # AGPL-3.0 + Apache-2.0
├── NOTICE
├── MAINTAINERS                   # 维护者制度（R-03）
├── CONTRIBUTING.md
├── .gitignore
├── .clang-format                 # OS-FMT-001 Tab 8
├── .editorconfig
│
├── Kbuild                        # 顶层 Kbuild（递归构建入口）
├── Kconfig                       # 顶层 Kconfig（特性配置入口）
├── Makefile                      # 顶层 Makefile（构建规则）
├── Makefile.airymaxos            # AirymaxOS 版本四元组定义
│                                 # 定义：AIRYMAX_LTS / AIRYMAX_MAJOR / AIRYMAX_MINOR / AIRYMAX_RELEASE
│
├── arch/                         # 架构相关代码（ES-OLK-2 + ES-SEL4-7）
│   ├── x86/                      # x86/x86_64 架构
│   │   ├── Kbuild
│   │   ├── Kconfig
│   │   ├── configs/              # x86 defconfig
│   │   │   └── airymax_x86_64_defconfig
│   │   ├── include/              # x86 架构相关头文件
│   │   ├── kernel/               # x86 内核执行逻辑
│   │   ├── mm/                   # x86 内存管理
│   │   ├── entry/                # x86 系统调用/中断入口
│   │   ├── machine/              # x86 架构相关机器原语
│   │   │   ├── capdl.c           # capability 分布描述（架构特定版）
│   │   │   ├── fpu.c             # 浮点单元
│   │   │   ├── registerset.c     # 寄存器集
│   │   │   └── hardware.c        # 硬件探测
│   │   ├── lib/                  # x86 特定库函数
│   │   └── boot/                 # x86 启动引导
│   ├── arm64/                    # ARM64 架构（结构同 x86/）
│   └── riscv/                    # RISC-V 架构（结构同 x86/）
│
├── include/                      # 公共头文件（ES-OLK-3 / ES-OLK-6）
│   ├── uapi/                     # 用户空间 ABI（永久稳定，OS-IRON-001）
│   │   └── airymax/
│   │       ├── syscall.h         # 系统调用号定义（冻结）
│   │       ├── syscall.xml       # 系统调用契约源定义（R-01）
│   │       ├── ipc.h              # IPC UAPI（128B 消息头用户可见部分）
│   │       ├── sched.h            # 调度 UAPI
│   │       ├── memory.h          # 内存 UAPI
│   │       ├── security.h       # 安全 UAPI
│   │       └── cognition.h      # 认知 UAPI
│   ├── airymax/                  # 内核内部头文件（[SC] 唯一物理宿主，OS-IRON-014）
│   │   ├── bpf_struct_ops.h      # [SC] struct_ops 状态机（唯一物理副本）
│   │   ├── memory_types.h        # [SC] MemoryRovol L1-L4（唯一物理副本）
│   │   ├── security_types.h      # [SC] capability 41 ID + LSM 钩子 252 ID（唯一物理副本）
│   │   ├── cognition_types.h     # [SC] CoreLoopThree + Thinkdual（唯一物理副本）
│   │   ├── sched.h               # [SC] 任务描述符（AGTS）+ vtime（唯一物理副本）
│   │   ├── ipc.h                 # [SC] IPC magic 0x41524531 + 128B 消息头（唯一物理副本）
│   │   ├── structures_32.bf      # 32-bit 位域定义（R-02）
│   │   ├── structures_64.bf      # 64-bit 位域定义（R-02）
│   │   ├── corekern.h            # 微核心原语内部头
│   │   └── airymax_kernel.h      # 总头文件
│   └── asm-generic/              # 通用汇编默认实现（ES-OLK-2）
│
├── kernel/                       # 上游内核核心 [模型 A 完整继承上游]
│                                 # 含 22 个子目录：bpf/cgroup/debug/dma/entry/events/
│                                 # futex/gcov/irq/kcsan/livepatch/locking/module/pgo/
│                                 # power/printk/rcu/sched/time/trace/xsched 等
│                                 # Airymax 增强直接写入对应子目录（如 kernel/sched/airymax_sched_ext.c）
│
├── corekern/                     # Airymax 微核心抽象层 [ES-SEL4-1 极简原则]
│                                 # 定位：Airymax 专属原语组织层，不替代上游 kernel/
│   ├── Kbuild
│   ├── Kconfig
│   ├── api/                      # 系统调用分发
│   │   ├── syscall_dispatch.c
│   │   └── faults.c
│   ├── sched/                    # 调度原语（sched_ext 策略，非 SCHED_AGENT 宏）
│   │   ├── airymax_agent_sched.c
│   │   ├── core_sched.c
│   │   └── ext.c
│   ├── ipc/                      # IPC 原语（io_uring 用户态-内核态 + kfifo kthread 间）
│   │   ├── io_uring_ipc.c
│   │   ├── kfifo_ipc.c
│   │   └── fastpath.c            # IPC fastpath 优化（ES-SEL4-4 借鉴）
│   ├── taskflow/                 # 任务流原语
│   │   ├── graph_engine.c
│   │   └── task_descriptor.c     # 任务描述符（magic 0x41475453）
│   ├── memory/                   # 内存原语接口（实现移入 memory 子仓）
│   ├── time/                     # 时间原语
│   ├── object/                   # 内核对象（cnode/tcb/endpoint/notification）
│   ├── locking/                  # 锁机制与同步原语
│   ├── irq/                      # 中断处理
│   └── bpf/                      # eBPF 可编程内核（kfunc + dynamic pointer + struct_ops）
│       └── struct_ops/
│           └── airymax_sched_ops.c
│
├── init/                         # 内核初始化（main.c + do_initcalls.c + version.c）
├── lib/                          # 通用库函数（string.c / ctype.c / bitmap.c / radix-tree.c）
├── machine/                      # 架构无关通用机器原语层（capdl/fpu/io/registerset/profiler）
│
├── crypto/                       # 加密算法库（含 sm2-sm4/ 国密 + aes/ + sha/）
├── io_uring/                     # 异步 I/O（ES-OLK-1 独立子系统）
│
├── fs/                           # 文件系统层 [模型 A 完整继承上游]
├── net/                          # 网络协议栈 [模型 A 完整继承上游]
├── block/                        # 块设备层 [模型 A 完整继承上游]
├── drivers/                      # 设备驱动 [模型 A 完整继承上游]
│
├── mm/                           # 核心内存管理 [模型 A 完整继承上游]
│                                 # 含 20+ 扁平 .c 文件（backing-dev.c/compaction.c/cma.c 等）+ damon/
│                                 # Airymax 修改直接写入（如 mm/airymax_mm.c）
│                                 # 边界：上游内核态内存管理实现在此；memory 子仓含 Airymax 专属
│                                 # 内核态扩展模块（rovol-kmod）+ 用户态工具，通过 hooks 接入上游 mm/
│
├── security/                     # LSM 框架 [模型 A 完整继承上游]
│                                 # 含 12 扁平 LSM 顶层目录（apparmor/bpf/integrity/keys/landlock/...）
│                                 # Airymax agent_lsm 直接写入 kernel/security/agent_lsm/
│                                 # 边界：security 子仓的 agent_lsm/ 是 Airymax 专属 LSM 策略 + 用户态工具
│
├── sound/                        # 音频子系统 [模型 A 完整继承上游]
├── certs/                        # 证书管理 [模型 A 完整继承上游]
├── ipc/                          # System V IPC [模型 A 完整继承上游]
├── samples/                      # 示例代码 [模型 A 完整继承上游]
├── usr/                          # 用户空间构建 [模型 A 完整继承上游]
├── virt/                         # 虚拟化 [模型 A 完整继承上游]
├── LICENSES/                     # 许可证文本 [模型 A 完整继承上游]
├── rust/                         # Rust 内核支持 [模型 A 完整继承上游]
│
├── Documentation/                # 文档即代码（ES-OLK-13）
│   ├── ABI/                      # ABI 文档（OS-IRON-001）
│   ├── airymax/                  # airymax 专属文档
│   └── process/                  # 开发流程文档
│
├── scripts/                     # 构建与检查脚本（ES-OLK-9）
│   ├── checkpatch.pl             # 编码规范检查
│   ├── syscall_gen.py            # syscall.xml → C 代码生成（R-01）
│   ├── bitfield_gen.py           # structures.bf → C 代码生成（R-02）
│   ├── kbuild_check.py           # Kbuild/Kconfig 规范检查
│   ├── maintainer_check.py       # MAINTAINERS 完整性检查（R-03）
│   └── dts/                      # 设备树源文件
│
├── tools/                       # 内核工具与测试基础设施 [模型 A 完整继承上游]
│   ├── testing/                  # 上游测试基础设施
│   │   ├── kunit/                # KUnit 框架
│   │   ├── selftests/            # kselftest 框架
│   │   └── fault-injection/     # 故障注入框架
│   ├── perf/                     # 性能剖析工具
│   ├── build/                    # 构建工具
│   └── scripts/                  # 内核构建辅助脚本
│
└── tests/                        # Airymax 专属测试用例
    └── airymax/                  # Airymax 专属测试用例（跨子仓集成测试在 tests-linux/integration/）
```

### 3.3 \[SC] 头文件子仓引用方式（禁止物理副本）

**OS-IRON-014 落地**：其他子仓**不得**在自身 `include/` 下创建 \[SC] 头文件的物理副本，**必须**通过构建系统 include path 引用 kernel 子仓的物理宿主：

```makefile
# 各子仓 Makefile / Kbuild 中的引用方式（示例）
# security/Makefile:
ccflags-y += -I$(srctree)/../kernel/include

# memory/Kbuild:
ccflags-y += -I$(srctree)/../kernel/include

# cognition/Kbuild:
ccflags-y += -I$(srctree)/../kernel/include
```

### 3.4 子仓内部头文件命名规范（避免与 \[SC] 混淆）

子仓自身的 `include/airymax/<sub>/` 目录**只存放该子仓内部头文件**（非 \[SC]），命名以 `_internal.h` 后缀区分：

| 子仓        | 内部头文件目录                                | 内容（均为非 \[SC] 内部头）                                              |
| --------- | -------------------------------------- | -------------------------------------------------------------- |
| security  | `security/include/airymax/security/`   | capability\_internal.h / lsm\_internal.h / cupolas\_internal.h |
| memory    | `memory/include/airymax/memory/`       | memoryrovol\_internal.h / cxl\_internal.h / pmem\_internal.h   |
| cognition | `cognition/include/airymax/cognition/` | clt\_internal.h / llm\_internal.h / thinkdual\_internal.h      |

**关键区分**：\[SC] 头文件无 `_internal` 后缀（如 `security_types.h`），子仓内部头文件有 `_internal` 后缀（如 `capability_internal.h`）。任何子仓的 `include/airymax/<sub>/` 下**禁止出现** \[SC] 头文件名。

***

## 4. 其他 7 子仓目录结构概要

> **OS-STD-062**：其余 7 子仓的完整目录结构详见各自模块设计文档（`20-modules/`）。每个子仓必须包含 MAINTAINERS 文件（R-03）和 Documentation/ 目录（ES-OLK-13）。

| 子仓          | 关键目录                                                                | 构建系统                             | seL4 借鉴点                   | 设计要点                                          |
| ----------- | ------------------------------------------------------------------- | -------------------------------- | -------------------------- | --------------------------------------------- |
| services    | userland-{vfs,net,drivers}/ + 12 daemons                            | Meson                            | ES-SEL4-3 服务用户态化           | userland-\* 通过 FUSE/VFIO/AF\_UNIX 接入上游        |
| security    | agent\_lsm/ + common/ + integrity/ + keys/ + capability/ + cupolas/ | Kbuild + Meson                   | ES-SEL4-2 capability 单一模型  | 方案 A 扁平 + 新增 integrity/keys                   |
| memory      | memoryrovol/ + cxl/ + pmem/ + mglru/ + vfs-persist-kern/            | Kbuild                           | Linux 6.6 内核基线 mm/         | 三方边界声明（上游 mm/ + corekern/ + memory 子仓）        |
| cognition   | CoreLoopThree + Thinkdual + LLM + compute-accel/                    | Cargo                            | Airymax 同源                 | compute-accel 边界声明                            |
| cloudnative | k8s-crd + containerd-shim + cni + sdk/                              | Go Modules                       | ES-SEL4-9 libsel4 独立 API 库 | sdk/ 按语言组织，非四级 include                        |
| system      | systemd 适配 + init + commons + devstation/                           | Kbuild + Meson                   | Linux 6.6 内核基线 init/       | devstation/ AI 边界（cognition 前端消费者）            |
| tests-linux | kunit/ + selftests/ + fault-injection/ + formal-verification/       | KUnit + kselftest + Isabelle/HOL | ES-SEL4-5 形式化验证            | unit/→kunit/，对齐 Linux 6.6 内核基线 tools/testing/ |

### 4.1 形式化验证工程标准（tests-linux 子仓）

**OS-TEST-010**：tests-linux 子仓必须包含 `formal-verification/` 目录，采用 seL4 三段式结构（ES-SEL4-5）：

```
formal-verification/
├── spec/        # 抽象规约（内核"应该做什么"的数学描述）
├── proof/       # Isabelle/HOL 证明（C 实现满足抽象规约）
└── capdl/       # capability 分布语言（系统初始 capability 状态描述）
```

形式化验证范围（1.0.1 阶段）：仅覆盖关键路径（调度器核心、IPC fastpath、capability 派生模型），不覆盖全部内核代码。

***

## 5. \[SC] 共享契约层物理隔离

### 5.1 物理宿主（OS-IRON-014 落地）

`kernel/include/airymax/` 目录是 \[SC] 共享契约层的**唯一物理宿主**，6 个核心头文件 + 2 个补充共享文件：

| 头文件                 | 内容                                                                                      | 共享对象                    | 落地路径                                                                                      |
| ------------------- | --------------------------------------------------------------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------- |
| `syscalls.h`        | 12 核心 syscall 编号 + 12 预留槽位                                                              | agentrt + agentrt-linux | `kernel/include/airymax/syscalls.h`                                                       |
| `ipc.h`             | IPC magic 0x41524531 + 128B 消息头（Layout C）                                               | agentrt + agentrt-linux | `kernel/include/airymax/ipc.h`                                                            |
| `sched.h`           | SCHED\_EXT + 任务描述符 magic 0x41475453                                                     | agentrt + agentrt-linux | `kernel/include/airymax/sched.h`                                                          |
| `security_types.h`  | POSIX capability + LSM 钩子 + Cupolas blob                                                | agentrt + agentrt-linux | `kernel/include/airymax/security_types.h`                                                 |
| `memory_types.h`    | MemoryRovol L1-L4 + GFP 掩码                                                              | agentrt + agentrt-linux | `kernel/include/airymax/memory_types.h`                                                   |
| `cognition_types.h` | CoreLoopThree + Thinkdual + LLM 推理                                                      | agentrt + agentrt-linux | `kernel/include/airymax/cognition_types.h`                                                |
| `bpf_struct_ops.h`  | struct\_ops 状态机（补充共享文件 1，非 \[SC] 核心）                                                    | agentrt + agentrt-linux | `kernel/include/airymax/bpf_struct_ops.h`                                                 |
| `error.h`           | 错误码 SSoT（补充共享文件 2，非 \[SC] 核心）——AIRY\_E\* POSIX 负值 + AIRY\_ERR\_\* 扩展码 + airy\_err\_t 类型 | agentrt + agentrt-linux | `kernel/include/airymax/error.h`（规划中，当前权威源 `agentrt/commons/utils/error/include/error.h`） |

> **SSoT 依赖声明**：\[SC] 头文件清单与共享内容的权威定义登记于 [`50-engineering-standards/120-cross-project-code-sharing.md §2`](../50-engineering-standards/120-cross-project-code-sharing.md)，规则编号权威登记于 [`50-engineering-standards/09-ssot-registry.md §2 OS-IRON-014`](../50-engineering-standards/09-ssot-registry.md)。本表为镜像，与上述 SSoT 冲突时以 SSoT 为准。

### 5.2 \[SC] 头文件铁律

1. **逐字节相同**：agentrt 与 agentrt-linux 两端 \[SC] 头文件必须逐字节一致（IRON-9 v2）。
2. **Tab 8 缩进**：\[SC] 头文件必须 Tab 8 缩进（OS-FMT-001，对齐 `.clang-format IndentWidth: 8`）。
3. **最小 typedef**：\[SC] 头文件必须最小 typedef（OS-STD-CODE-020，仅 5 种例外：`airy_vtime_t`/`airy_cap_op`/`airy_struct_ops_state`/`airy_ipc_msg_type`/`airy_err_t`）。
4. **双向 CI 校验**：\[SC] 头文件变更必须通过 agentrt 与 agentrt-linux 双向 CI 校验（OS-IRON-008）。
5. **SSoT 物理宿主**：\[SC] 头文件物理宿主为 `kernel/include/airymax/`，规则编号登记于 [09-ssot-registry.md §2 OS-IRON-014](../50-engineering-standards/09-ssot-registry.md)。
6. **变更流程**：任何 \[SC] 变更必须先更新 SSoT，再双向同步两端代码 + CI 验证。

### 5.3 \[SS] 语义同源层与 \[IND] 独立层

| 层           | 物理位置                                               | 共享方式            | 变更约束                       |
| ----------- | -------------------------------------------------- | --------------- | -------------------------- |
| \[SC] 共享契约层 | `kernel/include/airymax/*.h`                       | 逐字节相同           | 双向 CI（OS-IRON-008）         |
| \[SS] 语义同源层 | `kernel/include/kernel/airy_*.h` + agentrt 对应头     | API 签名独立演进，语义一致 | 文档化语义契约                    |
| \[IND] 独立层  | `kernel/`/`mm/`/`ipc/`/`security/`/`cognition/` 实现 | 完全独立            | 无约束（遵循 Linux 6.6 内核基线代码品味） |

***

## 6. 与 AirymaxOS 8 子仓的映射

agentrt-linux 8 子仓架构（详见 [10-architecture/README.md §2](README.md)）与源码目录的映射：

| AirymaxOS 子仓 | 源码目录                                                                             | 物理范围                       | 说明                           |
| ------------ | -------------------------------------------------------------------------------- | -------------------------- | ---------------------------- |
| kernel       | `kernel/`（含 `arch/` + `include/` + `corekern/` + 完整 Linux 6.6 源码树）               | 内核核心 + 架构相关 + UAPI + 微核心抽象 | 模型 A 完整 fork 落点              |
| services     | `services/`（userland-{vfs,net,drivers}/ + 12 daemons）                            | 服务子系统                      | VFS 用户态化 + 12 daemons        |
| security     | `security/`（agent\_lsm/ + common/ + integrity/ + keys/ + capability/ + cupolas/） | 安全子系统                      | capability + LSM + Cupolas   |
| memory       | `memory/`（memoryrovol/ + cxl/ + pmem/ + mglru/ + vfs-persist-kern/）              | 内存管理                       | MemoryRovol + CXL + MGLRU    |
| cognition    | `cognition/`（CoreLoopThree + Thinkdual + LLM + compute-accel/）                   | 认知子系统                      | CoreLoopThree kthread + Wasm |
| cloudnative  | `cloudnative/`（k8s-crd + containerd-shim + cni + sdk/）                           | 云原生适配                      | K8s + containerd shim        |
| system       | `system/`（systemd 适配 + init + commons + devstation/）                             | 系统集成 + 构建系统                | RPM + dnf + DevStation       |
| tests-linux  | `tests-linux/`（kunit/ + selftests/ + fault-injection/ + formal-verification/）    | 测试框架                       | 单元 + 集成 + 形式化 + Soak         |

### 6.1 子仓间的依赖关系

```
kernel (L2)
  ↑
  ├── services (L3)
  ├── security (L3)
  ├── memory (L3)
  └── cognition (L4)
        ↑
        ├── cloudnative (L5)
        └── system (L6)
              ↑
              └── tests-linux (L7)
```

**层次纪律**（S-2）：

1. 每层只依赖其直接下层的抽象接口，禁止越级访问
2. 同层之间通过 IPC 通信，禁止直接函数调用
3. 层次之间的接口契约通过 `30-interfaces/` 文档定义
4. 任何新增跨层依赖必须通过 ADR 评审

***

## 7. ES-OLK 工程思想映射

### 7.1 ES-OLK-1\~13 落地证据

| ES-OLK    | 工程思想            | 落地目录                                                              | 落地证据                           |
| --------- | --------------- | ----------------------------------------------------------------- | ------------------------------ |
| ES-OLK-1  | 关注点分离           | 8 子仓顶层划分 + `kernel/`/`mm/`/`ipc/`/`security/`/`cognition/` 顶层分离   | 8 子仓 + 5 大子系统目录独立              |
| ES-OLK-2  | 架构正交            | `kernel/arch/{x86,arm64,riscv}/`                                  | 唯一架构相关根目录                      |
| ES-OLK-3  | UAPI 分离         | `kernel/include/uapi/airymax/`                                    | 用户空间 ABI 唯一来源                  |
| ES-OLK-4  | 特性可配置           | `Kconfig` + `scripts/Kconfig`                                     | 顶层配置入口                         |
| ES-OLK-5  | 条件编译规范化         | `#ifdef CONFIG_*` 配置项                                             | 与 Kconfig 配合                   |
| ES-OLK-6  | UAPI 物理隔离       | `kernel/include/uapi/` 物理独立目录                                     | 与 `kernel/include/airymax/` 分离 |
| ES-OLK-7  | 声明式配置 + 命令式构建分离 | `Kconfig`（声明）+ `Makefile`（命令）+ `Makefile.airymaxos`（AirymaxOS 扩展） | 三层分离                           |
| ES-OLK-8  | 维护者制度           | 每子仓根 `MAINTAINERS`                                                | 文件级维护者登记（R-03）                 |
| ES-OLK-9  | 规范可执行           | `kernel/scripts/checkpatch.pl` + `.clang-format`                  | 80 偏好 / 100 硬阈值                |
| ES-OLK-10 | regression 零容忍  | `kernel/tools/testing/` + `tests-linux/`                          | 回归测试                           |
| ES-OLK-11 | ABI 永久稳定        | `kernel/include/uapi/airymax/syscalls.h`                          | syscall 编号 MAJOR 版本内不可变更       |
| ES-OLK-12 | 运行时可加载          | `kernel/scripts/Kbuild` 模块构建                                      | 模块化                            |
| ES-OLK-13 | 文档即代码           | `kernel/Documentation/process/*.rst`                              | 与源码同仓库                         |

### 7.2 Linux 6.6 内核基线工程实践对齐

| Linux 6.6 内核基线实践                             | agentrt-linux 落地                                      | 差距编号  | 优先级 |
| -------------------------------------------- | ----------------------------------------------------- | ----- | --- |
| `Makefile.oever` 版本四元组扩展                     | `kernel/Makefile.airymaxos`                           | C-D01 | P1  |
| `lib/Kconfig.distro` 发行版 Kconfig 扩展          | `kernel/scripts/Kconfig`                              | C-D02 | P1  |
| `arch/*/include/asm/atomic.h` SHA1 校验        | 待引入                                                   | C-D03 | P2  |
| `scripts/missing-syscalls` syscall 完整性检查     | `kernel/scripts/missing-syscalls`                     | C-D04 | P2  |
| `.rustfmt.toml` Rust 格式化                     | `cognition/rustfmt.toml` + `cloudnative/rustfmt.toml` | C-D05 | P2  |
| `EXPORT_TRACEPOINT_SYMBOL_GPL` tracepoint 导出 | 待引入                                                   | C-D06 | P2  |
| `checkpatch.pl max_line_length=100` 100 列硬阈值 | `kernel/scripts/checkpatch.pl`                        | C-D07 | P1  |
| `lockdep_assert_*_held` 锁断言                  | 待引入                                                   | C-D08 | P2  |

***

## 8. seL4 微内核思想映射

### 8.1 seL4 设计模式对齐

| seL4 设计模式                                             | agentrt-linux 落地                                                            | 差距编号  | 优先级   |
| ----------------------------------------------------- | --------------------------------------------------------------------------- | ----- | ----- |
| Liedtke 最小性原则                                         | `kernel/corekern/` 微核心原语                                                    | —     | ✅ 已对齐 |
| Capability 系统（CNode 7 操作 + MDB 派生树）                   | `kernel/corekern/object/cnode.c` + `security/capability/`                   | C-C08 | P2    |
| Endpoint 3 状态机（`EPState_Idle/Send/Recv`）              | `kernel/corekern/object/endpoint.c`                                         | —     | ✅ 已对齐 |
| Reply 原子性（`doReplyTransfer` 临界区串行）                    | `kernel/corekern/object/endpoint.c` `airy_ipc_reply()`                      | C-C07 | P2    |
| Notification（bitwise OR badge + 3 状态）                 | `kernel/corekern/object/notification.c`                                     | C-C01 | P1    |
| Fastpath（POINT OF NO RETURN + 12 项前置检查）               | `kernel/corekern/ipc/fastpath.c`                                            | —     | ✅ 已对齐 |
| Zombie 能力增量清理（`reduceZombie` + preemptionPoint）       | `kernel/corekern/object/`                                                   | C-C05 | P2    |
| 极简错误码（POSIX errno 负值方案）                               | `airy_err_t`（详见 120-cross-project-code-sharing.md §2.1）                     | C-C03 | ✅ 已收敛 |
| 单编译单元（`kernel_all.c` 拼接 + `-ffreestanding -nostdinc`） | 待评估                                                                         | C-C06 | P2    |
| 位域 DSL（`bitfield_gen.py` + Isabelle/HOL 同源）           | `kernel/scripts/bitfield_gen.py`（fork seL4，移除 HOL 部分）                       | C-C02 | P1    |
| syscall XML 化（`syscall.xml` + 生成脚本）                   | `kernel/include/uapi/airymax/syscall.xml` + `kernel/scripts/syscall_gen.py` | C-C04 | P1    |
| TCB 内嵌 reply slot（`tcbCaller`/`tcbReply`）             | `kernel/corekern/object/tcb.c`（待引入）                                         | C-C07 | P2    |

### 8.2 6 项 seL4 设计模式深度借鉴

| seL4 设计模式                                         | 来源                                 | agentrt-linux 落地建议    | 优先级 |
| ------------------------------------------------- | ---------------------------------- | --------------------- | --- |
| prune + 生成头文件分离构建                                 | seL4 `tools/bitfield_gen.py`       | 1.0.1 M2+ 评估          | P2  |
| preemptionPoint 精细化（work-unit 计数器）                | seL4 `src/kernel/thread.c`         | `kernel/corekern/` 吸收 | P1  |
| 三态 IRQ 状态机（IRQInactive/IRQSignal/IRQTimer/IRQIPI） | seL4 `include/object/structures.h` | 1.0.1 M2+ 评估          | P2  |
| 调度 bitmap 二级索引 + 倒序优化                             | seL4 `src/kernel/sched.c`          | 1.0.1 M3+ 评估          | P2  |
| `field_ptr(N)` 类型标记技巧                             | seL4 `include/object/structures.h` | 1.0.1 M2+ 评估          | P2  |
| Haskell error 注释模式                                | seL4 assert 注释                     | 0.1.1 文档层吸收           | P1  |

***

## 9. 文件命名规范

### 9.1 命名约定

| 类型             | 命名规则                    | 示例                                                 |
| -------------- | ----------------------- | -------------------------------------------------- |
| 内核源文件          | `airy_<子系统>_<功能>.c`     | `airy_ipc_msg.c`、`airy_endpoint.c`                 |
| 内核头文件          | `airy_<子系统>_<功能>.h`     | `airy_cspace.h`、`airy_endpoint.h`                  |
| \[SC] UAPI 头文件 | `<语义域>.h`（无 `airy_` 前缀） | `syscalls.h`、`ipc.h`、`sched.h`                     |
| 架构相关文件         | `arch/<arch>/<子系统>/`    | `arch/x86/kernel/airy_syscall.c`                   |
| 工具脚本           | `<工具名>_<类型>.py`/`.pl`   | `bitfield_gen.py`、`checkpatch.pl`                  |
| 构建脚本           | `<构建系统>.<扩展>`           | `Makefile`、`Kconfig`、`Kbuild`、`Makefile.airymaxos` |

### 9.2 函数前缀规范

- **内核函数**：`airy_<子系统>_<功能>()`，例如 `airy_ipc_send()`、`airy_endpoint_recv()`
- **\[SC] 共享函数**：`airy_<语义域>_<功能>()`，例如 `airy_vtime_decay()`（agentrt 与 agentrt-linux 共享）
- **禁止前缀**：`airymaxos_*`（已废弃，详见 project\_memory）、`agentrt_*`（已改名）、`agentos_*`（已废弃）

***

## 10. 与现有文档的关系

### 10.1 文档体系映射

| 文档                                                                                                                          | 目录结构关注点                                             |
| --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| [10-architecture/README.md](README.md)                                                                                      | 8 子仓架构 + 7 层架构模型                                    |
| [10-architecture/01-system-architecture.md](01-system-architecture.md)                                                      | 系统架构总览                                              |
| [10-architecture/03-microkernel-strategy.md](03-microkernel-strategy.md)                                                    | 微内核化改造策略                                            |
| [10-architecture/04-engineering-baseline.md](04-engineering-baseline.md)                                                    | 工程基线（Linux 6.6 内核基线锁定）                              |
| [20-modules/](../20-modules/)                                                                                               | 8 子仓详细模块设计                                          |
| [30-interfaces/](../30-interfaces/)                                                                                         | syscall + IPC + SDK + 编码规范                          |
| [40-dataflows/](../40-dataflows/)                                                                                           | 4 大数据流（认知 + 记忆 + IPC + 调度）                          |
| [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) | \[SC] 共享契约层落地规范                                     |
| [50-engineering-standards/09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)                             | **全局规则编号 SSoT**（OS-IRON-013/014/015 等所有规则编号的唯一权威来源） |

### 10.2 本文档的 SSoT 依赖声明

**本文档不是 SSoT**。本文档涉及的规则编号（如 OS-IRON-013 8 子仓 submodule、OS-IRON-014 \[SC] 单一数据源、OS-IRON-015 编号管理元规则、OS-STD-062 子仓完整目录结构、OS-FMT-001 Tab 8 等）的**唯一权威来源**为 [`50-engineering-standards/09-ssot-registry.md`](../50-engineering-standards/09-ssot-registry.md)。

当本文档与 SSoT 注册表冲突时，以 SSoT 注册表为准。新规则须按 [09-ssot-registry.md §1.4](../50-engineering-standards/09-ssot-registry.md) 流程申请注册。

***

## 11. 实施路线图

### 11.1 0.1.1（文档体系完成）

- [x] 本文档创建（v0.1.1 初始版本）
- [x] v0.2.4 升级：单仓视图 → 8 子仓 submodule（OS-IRON-013 落地）
- [x] v0.2.4 升级：\[SC] 路径修正 `include/uapi/airymax/` → `kernel/include/airymax/`（OS-IRON-014 落地）
- [x] v0.2.4 升级：kernel/ 子仓定位修正（10 个 airy\_\*.c → 模型 A 完整 fork 60K+文件）
- [x] v0.2.4 升级：§10.2 SSoT 声明修正（自称 SSoT → 引用 09-ssot-registry 唯一 SSoT）
- [x] ES-OLK-1\~13 落地证据映射
- [x] seL4 设计模式对齐状态评估

### 11.2 1.0.1 M0（目录骨架创建）

- [ ] 8 子仓 git 仓库初始化（kernel/services/security/memory/cognition/cloudnative/system/tests-linux）
- [ ] 管理仓 .gitmodules 配置 + submodule 接入
- [ ] 每子仓公共骨架（README/LICENSE/NOTICE/MAINTAINERS/.gitignore/CONTRIBUTING.md）
- [ ] kernel/ 子仓完整 Linux 6.6 fork 接入（git submodule 或 git subtree）
- [ ] `kernel/include/airymax/` 6 \[SC] 头文件物理落地
- [ ] `kernel/arch/{x86,arm64,riscv}/` 三架构骨架
- [ ] `kernel/scripts/` 构建系统（`Makefile`/`Kconfig`/`Kbuild`/`Makefile.airymaxos`）
- [ ] `kernel/scripts/checkpatch.pl` + `.clang-format` 落地

### 11.3 1.0.1 M1（代码开发启动）

- [ ] `kernel/corekern/` 微核心原语骨架代码填充
- [ ] 各子仓模块骨架代码填充
- [ ] CI 流水线搭建（含 \[SC] 双向 CI）

### 11.4 1.0.1 M2+（深度借鉴）

- [ ] `kernel/scripts/bitfield_gen.py` fork seL4 + 移除 HOL 部分（C-C02）
- [ ] `kernel/include/uapi/airymax/syscall.xml` + `kernel/scripts/syscall_gen.py`（C-C04）
- [ ] `kernel/corekern/object/notification.c` 重新设计（3 状态 + bitwise OR badge，C-C01）
- [ ] `kernel/corekern/object/` Zombie 增量清理算法（C-C05）
- [ ] TCB 内嵌 reply slot（C-C07）
- [ ] MDB 派生树增强（C-C08）
- [ ] CapRights 4 位掩码（C-C09）

***

## 12. 版本历史

| 版本        | 日期         | 变更                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0.1.1     | 2026-07-13 | 初始版本（单仓视图，\[SC] 路径误用 `include/uapi/airymax/`，kernel/ 仅 10 个 airy\_\*.c）                                                                                                                                                                                                                                                                                                                                                          |
| 0.2.4     | 2026-07-15 | **生产级修正版**：(1) 单仓视图 → 8 子仓 submodule（OS-IRON-013 落地）；(2) \[SC] 路径 `include/uapi/airymax/` → `kernel/include/airymax/`（OS-IRON-014 落地）；(3) kernel/ 子仓 10 个 airy\_\*.c → 模型 A 完整 fork 60K+文件；(4) §10.2 自称 SSoT → 引用 09-ssot-registry 唯一 SSoT；(5) 清洗所有上游发行版名称引用（BAN-INTERNAL-001 落地，统一表述为"Linux 6.6 内核基线"）；(6) 新增 §3 kernel/ 完整目录树 + §3.3 子仓引用方式 + §3.4 子仓内部头文件命名规范 + §4 其他 7 子仓目录结构概要 + §4.1 形式化验证工程标准；(7) §11 实施路线图新增 M0 目录骨架创建阶段 |
| 1.0.1 M1  | 2027-XX-XX | M1 代码开发启动，骨架代码填充                                                                                                                                                                                                                                                                                                                                                                                                                 |
| 1.0.1 M2+ | 2027-XX-XX | seL4 深度借鉴（bitfield DSL / Notification 重设计 / Zombie 清理等）                                                                                                                                                                                                                                                                                                                                                                          |

***

© 2025-2026 SPHARX Ltd. All Rights Reserved.
