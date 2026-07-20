Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 发行版管理设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）发行版管理工程体系主索引——RPM 打包、SBOM 生成、可重现构建、镜像构建、安装程序与发布流程\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[AirymaxOS 总览](../README.md)

---

## 1. 模块概述

`190-distribution/` 是 AirymaxOS 文档体系中**操作系统作为产品发布的工程保障**。它继承 Linux 发行版 30+ 年沉淀的发行版管理哲学（版本号 + RPM 包 + ISO 镜像 + 仓库管理 + 签名验证），并在其上扩展智能体操作系统专属的 Agent 应用商店、记忆卷载分发、12 daemon RPM 包、多架构镜像、可重现构建、SBOM 生成等。

发行版管理不是「打包」的代名词，而是「版本化、可追溯、可验证」的工程体系。AirymaxOS 发行版管理的核心理念是：**每个版本必须可追溯，每个包必须可验证，每个镜像必须可重现构建**。这要求从源码到发布的全链路工程化，而非手工打包。

本模块承担六项核心职责：

1. **RPM 打包**：RPM 打包规范 + spec 模板 + GPG 签名 + 12 daemon RPM 包。
2. **dnf 仓库设计**：dnf 仓库分层（stable/testing/nightly）+ metadata 生成。
3. **安装程序**：三模式安装 + Btrfs 子卷 + LVM + Secure Boot + TPM + kickstart。
4. **更新机制**：内核热更新 + 原子更新 + Btrfs 快照回滚。
5. **可重现构建**：SOURCE_DATE_EPOCH + 编译器确定性标志 + 全产物可重现构建。
6. **SBOM 生成**：软件物料清单生成 + 供应链安全 + 跨架构验证。

---

## 2. 技术选型声明

agentrt-linux v1.0 在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度做出**不可妥协**的技术选型。本目录所有打包与发布流程必须以本声明为基线——RPM 包不得依赖 sched_ext / page flipping / BPF LSM / DMA 一致性内存。

| # | 技术维度 | 选定方案 | 明确不采用 | 选定理由 |
|---|---------|---------|-----------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类，通过 `nice` / `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，**不定义新的调度类宏** | **不使用 sched_ext**（不引入 eBPF 调度器、不依赖 `SCHED_EXT=7`、不使用 SCHED_AGENT 宏） | 内核 RPM 包基于 Linux 6.6 主线调度器，发行版无需打包 sched_ext 相关模块 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输，registered buffer + mmap 共享页 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | IPC 模块 RPM 包基于 IORING_OP_URING_CMD，无需 page flipping 兼容代码 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 Linux Security Module（`airy_lsm`），通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | 安全模块 RPM 包为纯 C LSM，无需打包 BPF LSM 依赖 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `remap_pfn_range` 映射到用户态地址空间 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | 内存模块 RPM 包基于 alloc_pages + mmap，跨架构一致 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | v2 三层模型（升级为 v3，新增 [DSL] 降级生存层） | [DSL] 降级生存层确保 [SC] 损坏时系统仍可降级运行，RPM 包类型安全 |

> 详细技术选型声明见上级文档 [AirymaxOS 总览](../README.md) §2 与 [`10-architecture/10-unify-design.md`](../10-architecture/10-unify-design.md) SSoT 声明。

---

## 3. 文档索引

`190-distribution/` 由 7 个文档构成（含本 README）：

```
190-distribution/
├── README.md                       # 本文件 — 发行版管理主索引（v1.0）
├── 01-rpm-packaging.md             # RPM 打包规范 + spec 模板 + GPG 签名 + 12 daemon 包（v1.0）
├── 02-dnf-repo-design.md           # dnf 仓库设计（分层 stable/testing/nightly + metadata）（v1.0）
├── 03-installer-design.md          # 安装器设计（三模式 + Btrfs + LVM + Secure Boot + TPM）（v1.0）
├── 04-update-mechanism.md          # 更新机制（内核热更新 + 原子更新 + Btrfs 快照回滚）（v1.0）
├── 05-reproducible-build.md        # 可重现构建（SOURCE_DATE_EPOCH + 全产物可重现 + 跨架构）（v1.0）
└── 06-sbom-generation.md           # SBOM 生成（软件物料清单 + 供应链安全 + 跨架构验证）（v1.0）
```

### 3.1 各文档定位

| 文档 | 核心问题 | 主要产物 |
|------|---------|---------|
| README.md | 发行版管理全貌？ | 6 层分层 + 12 daemon RPM 包 + 技术选型声明 |
| 01-rpm-packaging.md | 包怎么打？ | RPM spec 模板 + GPG 签名 + 12 daemon 包清单 |
| 02-dnf-repo-design.md | 仓库怎么管？ | dnf 仓库分层 + metadata 生成 + 签名 |
| 03-installer-design.md | 系统怎么装？ | 三模式安装 + Btrfs + Secure Boot + kickstart |
| 04-update-mechanism.md | 系统怎么更新？ | 内核热更新 + 原子更新 + Btrfs 快照回滚 |
| 05-reproducible-build.md | 构建怎么重现？ | SOURCE_DATE_EPOCH + 全产物可重现 + 跨架构 |
| 06-sbom-generation.md | 物料清单怎么生成？ | SBOM 生成 + 供应链安全 + 跨架构验证 |

### 3.2 发行版分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 版本号 | 语义化版本 + LTS 策略 | 版本管理 |
| L2 | 包格式 | RPM + dnf | 包管理 |
| L3 | 镜像构建 | ISO + QCOW2 + Docker | 镜像分发 |
| L4 | 仓库管理 | dnf 仓库 + 签名 | 软件仓库 |
| **L5** | **Agent 应用商店** | **AirymaxOS 专属** | **Agent 应用分发** |
| **L6** | **记忆卷载分发** | **AirymaxOS 专属** | **记忆卷载包** |

### 3.3 12 daemon RPM 包

AirymaxOS 发行版将 12 个核心 daemon 打包为独立 RPM 包，对应 8 子仓中的 services 子仓与 A-ULS/A-ULP/A-UCS 模块：

| # | RPM 包名 | 对应模块 | 职责 |
|---|---------|---------|------|
| 1 | `agentrt-macro-superv` | A-ULS | Macro-Supervisor 用户态温情裁决守护进程 |
| 2 | `agentrt-logger` | A-ULP | Logger Daemon 异步日志格式化与落盘 |
| 3 | `agentrt-config` | A-UCS | 统一配置管理体系 + sysctl/JSON 热重载 |
| 4 | `agentrt-gateway` | A-IPC | Agent 网关 + K8s Ingress Controller |
| 5 | `agentrt-sched` | A-ULS/sched_tac | 调度策略注入（SCHED_DEADLINE/SCHED_FIFO/EEVDF） |
| 6 | `agentrt-vfs` | services | 用户态 VFS 服务 |
| 7 | `agentrt-net` | services | 网络服务（CNI 集成） |
| 8 | `agentrt-mem` | memory | 记忆卷载服务（MemoryRovol + CXL） |
| 9 | `agentrt-cogn` | cognition | 认知循环服务（CoreLoopThree kthread） |
| 10 | `agentrt-sec` | security | 纯 C LSM 安全模块 + Capability 系统 |
| 11 | `agentrt-audit` | A-UEF/A-ULP | 审计服务（错误码 + 日志观测） |
| 12 | `agentrt-dev` | system | 开发者工具（DevStation + agentctl） |

### 3.4 多架构镜像

| 架构 | 状态 | 优先级 |
|------|------|--------|
| x86_64 | 主架构 | P0 |
| aarch64 | 主架构 | P0 |
| riscv64 | 次架构 | P1 |
| ppc64le | 实验架构 | P2 |

---

## 4. Airymax Unify Design 映射

Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）在发行版管理中的关系如下。发行版管理是 A-UCS airy_defconfig 打包与 A-ULS 12 daemon systemd 服务打包的主要落地维度。

| Unify 模块 | 全称 | 在本目录的体现 | 对应 RPM 包 |
|-----------|------|--------------|------------|
| **A-UCS** | Unified Configuration Subsystem | **核心**：`airy_defconfig` 打包、配置文件 RPM、sysctl/JSON 配置热重载打包 | `agentrt-config` |
| **A-ULS** | Unified Lifecycle Supervision Framework | **核心**：12 daemon systemd 服务打包、Macro-Supervisor 服务单元、8 态生命周期监管服务化 | `agentrt-macro-superv` / `agentrt-sched` / `agentrt-sec` |
| **A-ULP** | Unified Logging and Printk Subsystem | Logger Daemon systemd 服务打包、128B 日志格式配置打包 | `agentrt-logger` / `agentrt-audit` |
| **A-IPC** | Unified Airymax IPC Fabric | Agent 网关服务打包、IPC Ring Buffer 配置打包 | `agentrt-gateway` |
| **A-UEF** | Unified Error and Fault Framework | 错误码 [SC] 头文件打包（`error.h`）、审计服务打包 | `agentrt-audit` |

### 4.1 A-UCS airy_defconfig 打包

A-UCS 的 `airy_defconfig` 通过 RPM 包分发，配置文件遵循 [SC] 共享契约（二进制布局相关配置项）与 [SS] 语义同源配置（sysctl/JSON 语义映射）：

```bash
# 安装配置包
dnf install agentrt-config

# airy_defconfig 打包内容
/etc/agentrt/airy_defconfig      # [SC] 共享配置（二进制布局）
/etc/agentrt/airy_sysctl.conf    # [SS] sysctl 语义同源配置（YAML/TOML 格式，JSON 仅用于 IPC payload）
/etc/agentrt/airy_config.yaml    # [SS] YAML 语义同源配置
```

### 4.2 A-ULS 12 daemon systemd 服务打包

A-ULS 的 12 个 daemon 通过 systemd 服务单元打包为独立 RPM，每个 daemon 对应一个 `.service` 文件：

```bash
# systemd 服务单元示例
/usr/lib/systemd/system/agentrt-macro-superv.service
/usr/lib/systemd/system/agentrt-logger.service
/usr/lib/systemd/system/agentrt-sched.service
# ... 共 12 个 daemon 服务单元
```

A-ULS 的双 Supervisor 模型在发行版中体现为：内核 Micro-Supervisor（冷酷执法，纯 C LSM 模块随内核 RPM）+ 用户态 Macro-Supervisor（温情裁决，`agentrt-macro-superv` RPM 包）。Macro-Supervisor 单点故障时，内核 watchdog 直接重启。

### 4.3 12 daemon 与sched_tac 调度

12 daemon 的调度策略由sched_tac 承载（**不使用 sched_ext**）：

- `agentrt-sched`：通过 `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略
- `agentrt-cogn`：CoreLoopThree kthread 使用 SCHED_DEADLINE（实时确定性）
- `agentrt-macro-superv`：使用 SCHED_FIFO（高优先级监管）
- 其余 daemon：使用 EEVDF（Linux 6.6 默认）

### 4.4 同源 agentrt 版本管理（IRON-9 v3）

发行版管理遵循 **IRON-9 v3 四层模型**与 agentrt 版本管理同源：[SC] 共享头文件打包（`kernel/include/uapi/linux/airymax/` 10 个头文件随内核 RPM 分发），[SS] 语义同源（版本号策略协同），[IND] 各自独立实现（agentrt 用户态版本 ↔ AirymaxOS 发行版版本），[DSL] 降级生存块（#ifdef AIRY_SC_FALLBACK）。

---

## 5. 相关文档

### 5.1 上级与架构文档
- [AirymaxOS 总览](../README.md) —— 文档体系顶层纲领（v1.0）
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（五模块 SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 四层模型

### 5.2 模块与构建文档
- [20-modules/02-services.md](../20-modules/02-services.md) —— services 子仓（12 daemon）
- [20-modules/11-unified-config.md](../20-modules/11-unified-config.md) —— A-UCS 统一配置（airy_defconfig）
- [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) —— A-ULS Micro-Supervisor
- [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md) —— A-ULS Macro-Supervisor
- [70-build-system/README.md](../70-build-system/README.md) —— 构建系统（Kbuild + Kconfig + RPM spec）

### 5.3 关联模块文档
- [100-operations/README.md](../100-operations/README.md) —— 部署运维
- [120-development-process/README.md](../120-development-process/README.md) —— 开发流程（稳定版维护）
- [160-compatibility/README.md](../160-compatibility/README.md) —— 兼容性管理（ABI + 跨发行版）
- [110-security/README.md](../110-security/README.md) —— 安全加固（Secure Boot + TPM + GPG 签名）
- [50-engineering-standards/07-maintainers-and-governance.md](../50-engineering-standards/07-maintainers-and-governance.md) —— 维护者治理

### 5.4 参考材料
- Linux 6.6 RPM 打包规范
- dnf 包管理器 + ISO 9660 镜像标准
- Reproducible Builds 项目 + SPDX/CycloneDX SBOM 标准
- agentrt 版本管理规范

---

## 6. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，5/5 文档完成（RPM + dnf 仓库 + 安装器 + 更新 + 可重现构建） |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）；IORING_OP_URING_CMD（不使用 page flipping）；纯 C LSM（不使用 BPF LSM）；alloc_pages + mmap（不使用 DMA 一致性内存）；IRON-9 v3 四层模型（新增 [DSL] 降级生存层）；新增 12 daemon RPM 包清单（agentrt-macro-superv / agentrt-logger / agentrt-config / agentrt-gateway / agentrt-sched / agentrt-vfs / agentrt-net / agentrt-mem / agentrt-cogn / agentrt-sec / agentrt-audit / agentrt-dev）；补全 06-sbom-generation.md 文档索引；新增 Airymax Unify Design 五模块映射（A-UCS airy_defconfig 打包 / A-ULS 12 daemon systemd 服务打包） |

---

> **文档结束** | 发行版管理模块 v1.0 | 共 7 文档 | 维护者：开源极境工程与规范委员会 | 每个版本必须可追溯，每个包必须可验证
