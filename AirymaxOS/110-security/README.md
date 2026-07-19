Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）安全加固设计
> **文档定位**：agentrt-linux（AirymaxOS）安全工程体系主索引（Capability 模型 + 纯 C LSM + IPC 数据面自治 + 控制面 Reconciliation + io_uring 加固）\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-19\
> **上级文档**：[AirymaxOS 总览](../README.md)\
> **同源映射**：agentrt Cupolas（安全穹顶）+ Linux 6.6 LSM/Landlock/capability\
> **理论根基**：Linux 内核安全机制 + Airymax E-1 安全内生 + K-3 服务隔离 + seL4 capability 模型

---

## 1. 模块概述

agentrt-linux 安全加固体系是系统可信运行的核心保障。它继承 Linux 内核 30+ 年沉淀的多层安全哲学（LSM 框架 + Landlock 用户态沙箱 + capability + 模块签名 + Lockdown + 4 层密钥环），并在其上扩展智能体操作系统专属的纯 C `airy_lsm`、IPC 数据面自治、控制面 Reconciliation、io_uring 加固等。本目录覆盖五方面职责：

1. **Capability 模型**：seL4 风格 capability + POSIX caps，41 个 capability ID，4 种派生操作（mint/mintcopy/derive/revoke），令牌 7 状态生命周期，Cupolas blob 四类布局。
2. **纯 C LSM**：以纯 C 实现的 `airy_lsm` 模块通过 `security_hook_list` 注册（**不使用 BPF LSM，对齐 openEuler 纯 C 模式**），252 个 LSM 钩子 ID，`DEFINE_LSM(airy)` 骨架，`LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位共存机制（v1.1：不滥用 `LSM_ORDER_FIRST`，OLK 6.6 注释明确仅用于 capabilities）。
3. **IPC 数据面自治**：A-IPC 数据面自治三原则——Ring 生命周期解耦、离线缓存校验、控制面故障不影响数据面正常运作。
4. **控制面 Reconciliation**：控制面恢复后，数据面将离线期间的状态变更同步给控制面，达成最终一致。
5. **io_uring 加固**：`IORING_OP_URING_CMD` 命令操作码白名单、registered buffer 完整性校验、SQE/CQE 校验、Ring 冻结机制（`ring->frozen`）。

### 1.1 安全体系分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | LSM 框架 | `security_hook_heads` + `lsm_blob_sizes` + 排序 | 内核安全钩子 |
| L2 | 用户态沙箱 | Landlock（非特权进程自限制） | 进程级隔离 |
| L3 | capability | seL4 风格 capability + POSIX caps | 权限细粒度 |
| L4 | 模块签名 | 模块签名 + Lockdown | 代码完整性 |
| L5 | 密钥环 | builtin/secondary/machine/platform 4 层 | 密钥管理 |
| **L6** | **纯 C `airy_lsm`** | **agentrt-linux 专属（不使用 BPF LSM）** | **Agent 行为约束** |
| **L7** | **IPC 数据面自治** | **agentrt-linux 专属** | **Ring 生命周期解耦** |
| **L8** | **控制面 Reconciliation** | **agentrt-linux 专属** | **最终一致性同步** |
| **L9** | **io_uring 加固** | **agentrt-linux 专属** | **命令白名单 + Ring 冻结** |

### 1.2 agentrt-linux 扩展

- **纯 C `airy_lsm`**：不使用 BPF LSM（对齐 openEuler 纯 C 模式），252 个 LSM 钩子 ID 通过 `security_hook_list` 注册
- **IPC 数据面自治**：Ring 生命周期解耦、离线缓存校验、Reconciliation 三原则
- **io_uring 加固**：`IORING_OP_URING_CMD` 命令白名单、registered buffer 完整性校验、Ring 冻结机制
- **Cupolas 安全穹顶**：从 agentrt 同源的 7 大子系统（Guards/Permission/Sanitizer/Audit/Workbench/Vault/Network）

---

## 2. 技术选型声明

agentrt-linux v1.0 安全加固体系在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度遵循 [AirymaxOS 总览](../README.md) §2 的不可妥协基线。安全体系是五大选型中**纯 C LSM**选型的权威承载者，并以 capability 模型与 io_uring 加固贯穿其余四维度。五个维度的选型在本目录的具体落地如下：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 在本目录的落地 |
|---|---------|---------|----------------|--------------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 `SCHED_EXT=7` 调度类） | 安全审计线程（`audit_d`）通过 `SCHED_FIFO` 注入优先级，确保审计不丢失；调度策略变更通过 capability 校验 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | io_uring 加固——命令操作码白名单、registered buffer 完整性校验、Ring 冻结机制（`ring->frozen`） |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 `airy_lsm` 通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | **本目录的核心选型**——纯 C `airy_lsm` 模块（对齐 openEuler 纯 C 模式），252 个 LSM 钩子 ID，`DEFINE_LSM(airy)` 骨架，`LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位共存（v1.1：不滥用 `LSM_ORDER_FIRST`，OLK 6.6 注释明确仅用于 capabilities）；v1.1 Capability Folding——fastpath C-S9 内联 Badge 校验（~10ns，3 个 `READ_ONCE` + 位运算）+ LSM slowpath 接管（仅 C-S9 失败时触发） |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | v1.1 `agent_caps[1024]` 静态数组（16KB，无锁多读者）替代 v1.0 radix tree；IPC Ring Buffer 共享页通过 `alloc_pages + mmap`（不使用 DMA 一致性内存） |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | （v2 三层模型升级为 v3 四层模型，新增 [DSL] 降级生存层） | [SC] `security_types.h`（POSIX capability 41 ID + LSM 钩子 252 ID + Cupolas blob 布局）+ [SC] `lsm_types.h`（纯 C LSM 类型定义 + `DEFINE_LSM(airy)` 骨架 + Capability 缓存结构）双端逐字节一致 |

### 2.1 纯 C LSM 权威声明（不使用 BPF LSM，对齐 openEuler）

> **SSoT 声明**：agentrt-linux v1.0 的安全钩子**唯一采用纯 C LSM 模块**（`airy_lsm`），通过 `security_hook_list` 注册，**明确不使用 BPF LSM**。本选型对齐 openEuler 纯 C LSM 模式，保证安全策略的可审计性与形式化验证可行性，避免 BPF 验证器在安全关键路径的语义不确定性。`airy_defconfig` 强制 `# CONFIG_BPF_LSM is not set`；`CONFIG_SECURITY_AIRY_LSM=y` 编译纯 C LSM 模块。

### 2.2 IRON-9 v3 四层模型在安全体系的归属

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 落地文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| POSIX capability 41 ID | ● | — | — | ● | [03-capability-model.md](03-capability-model.md) |
| LSM 钩子 252 ID | ● | — | — | — | [01-lsm-framework.md](01-lsm-framework.md) |
| 纯 C LSM 类型定义 | ● | — | ● | — | [07-airy-lsm-design.md](07-airy-lsm-design.md) |
| Cupolas blob 布局 | ● | — | — | — | [03-capability-model.md](03-capability-model.md) |
| IPC 数据面自治三原则 | — | — | ● | — | [04-ipc-data-plane-autonomy.md](04-ipc-data-plane-autonomy.md) |
| 控制面 Reconciliation | — | — | ● | — | [05-ipc-control-plane-reconciliation.md](05-ipc-control-plane-reconciliation.md) |
| io_uring 命令白名单 | — | — | ● | — | [06-io-uring-hardening.md](06-io-uring-hardening.md) |
| Ring 冻结机制 | — | — | ● | — | [06-io-uring-hardening.md](06-io-uring-hardening.md) |
| Landlock 沙箱 | — | — | ● | — | [02-landlock-sandbox.md](02-landlock-sandbox.md) |
| 安全模型语义（agentrt 同源） | — | ● | — | — | [03-capability-model.md](03-capability-model.md) |

---

## 3. 文档索引

本目录现有文档（v1.0 范围，含新建 04-07）：

```
110-security/
├── README.md                                   # 本文件（v1.0）
├── 01-lsm-framework.md                         # LSM 框架详解（security_hook_heads + 排序）
├── 02-landlock-sandbox.md                      # Landlock 用户态沙箱
├── 03-capability-model.md                      # seL4 风格 capability + POSIX 41 ID + Cupolas blob
├── 04-ipc-data-plane-autonomy.md               # 新建：IPC 数据面自治三原则
├── 05-ipc-control-plane-reconciliation.md      # 新建：控制面 Reconciliation 最终一致性
├── 06-io-uring-hardening.md                    # 新建：io_uring 加固（命令白名单 + Ring 冻结）
└── 07-airy-lsm-design.md                       # 新建：纯 C airy_lsm 设计（不使用 BPF LSM）
```

| # | 文档 | 版本 | 内容概要 |
|---|------|------|---------|
| — | [README.md](README.md) | v1.0 | 安全加固主索引（本文件） |
| 1 | [01-lsm-framework.md](01-lsm-framework.md) | v1.0 | LSM 框架核心（`security_hook_heads` + `lsm_blob_sizes` + 排序机制）、252 个 LSM 钩子 ID |
| 2 | [02-landlock-sandbox.md](02-landlock-sandbox.md) | v1.0 | Landlock 用户态沙箱（非特权进程自限制）、文件系统/网络细粒度访问控制 |
| 3 | [03-capability-model.md](03-capability-model.md) | v1.0 | seL4 风格 capability、POSIX 41 ID、4 种派生操作（mint/mintcopy/derive/revoke）、令牌 7 状态、Cupolas blob 四类布局、4 值策略裁决、Vault backend、syscall 592-600 集成、capability `LSM_ORDER_FIRST`（OLK 6.6 仅用于 capabilities）+ airy `LSM_ORDER_MUTABLE` + `CONFIG_LSM` 首位共存 |
| 4 | [04-ipc-data-plane-autonomy.md](04-ipc-data-plane-autonomy.md) | v1.0 | **新建**：IPC 数据面自治三原则——Ring 生命周期解耦、离线缓存校验、控制面故障不影响数据面 |
| 5 | [05-ipc-control-plane-reconciliation.md](05-ipc-control-plane-reconciliation.md) | v1.0 | **新建**：控制面 Reconciliation——控制面恢复后数据面状态变更同步、最终一致性达成 |
| 6 | [06-io-uring-hardening.md](06-io-uring-hardening.md) | v1.0 | **新建**：io_uring 加固——`IORING_OP_URING_CMD` 命令白名单、registered buffer 完整性校验、SQE/CQE 校验、Ring 冻结机制（`ring->frozen`） |
| 7 | [07-airy-lsm-design.md](07-airy-lsm-design.md) | v1.0 | **新建**：纯 C `airy_lsm` 设计——`DEFINE_LSM(airy)` 骨架、`security_hook_list` 注册、capability 分层校验（fastpath + slowpath）、**不使用 BPF LSM（对齐 openEuler）** |

### 3.1 后续规划文档（1.0.1 版本）

以下文档在 1.0.1 版本完成，不在 v1.0 范围内：

- `08-module-signing.md`：模块签名验证
- `09-lockdown.md`：内核 Lockdown 模式
- `10-keyrings.md`：4 层密钥环
- `11-cupolas-dome.md`：agentrt-linux 专属 Cupolas 安全穹顶
- `12-confidential-computing.md`：机密计算（VirtCCA）
- `13-cryptography-compliance.md`：国密算法合规

---

## 4. Airymax Unify Design 映射

本目录与 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的关系详见 [AirymaxOS 总览](../README.md) §5。安全加固体系主要承载 **A-ULS**（Capability 系统 + 纯 C LSM）与 **A-IPC**（IPC 安全 + io_uring 加固），其余三模块为辅助关系。

| Unify 模块 | 关系 | 在本目录的体现 |
|-----------|------|--------------|
| **A-UEF** | 辅助 | 安全事件错误码统一使用 A-UEF 的 `AIRY_E*`（如 `AIRY_EPERM=-1`）与 `AIRY_FAULT_*`（如 `AIRY_FAULT_CAP_FORGED=0x1001` / `AIRY_FAULT_ABNORMAL_CAP=0x1005`）码空间 |
| **A-ULP** | 辅助 | 安全审计日志通过 A-ULP Ring Buffer 写入（128B 固定记录）；`audit_d` daemon 消费审计事件 |
| **A-UCS** | 辅助 | 安全策略参数（capability 缓存超时、Ring 冻结阈值）通过 A-UCS 的 sysctl/JSON 双向热重载 |
| **A-ULS** | **核心** | Capability 系统 + 纯 C LSM——`airy_lsm` 的 Micro-Supervisor 冷酷执法（检测伪造/越权 capability → 立即冻结 Ring → 返回 `AIRY_FAULT_ABNORMAL_CAP` → eventfd 通知 Macro-Supervisor）；Agent 8 态生命周期中的权限就绪校验 |
| **A-IPC** | **核心** | IPC 安全 + io_uring 加固——v1.1 Capability Folding（fastpath C-S9 内联 Badge 校验 ~10ns + LSM slowpath 接管仅失败时触发）、IPC 数据面自治三原则、控制面 Reconciliation、io_uring 命令白名单与 Ring 冻结机制 |

### 4.1 Unify Design 权威源引用

- A-ULS 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §7
- A-IPC 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §8
- Micro-Supervisor 设计：[../20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)
- IPC 协议契约：[../30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)
- IPC fastpath：[../30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)

---

## 5. 相关文档

- [AirymaxOS 总览](../README.md)：v1.0 技术选型声明 + 20 子目录索引
- [架构设计层](../10-architecture/README.md)：Unify Design 总纲 + IRON-9 v3 四层模型
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md)：Airymax Unify Design 总纲（SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（SSoT）
- [10-architecture/08-threat-model.md](../10-architecture/08-threat-model.md)：威胁模型
- [20-modules/03-security.md](../20-modules/03-security.md)：security 子仓设计
- [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)：Micro-Supervisor（内核冷酷执法）
- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)：IPC 协议（128B 消息头 + capability 校验）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)：IPC fastpath（capability 分层校验）
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)：A-UEF [SC] error.h 契约（Fault 码空间）
- [50-engineering-standards/11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md)：[SC] 头文件类型桥接规则
- [60-driver-model/README.md](../60-driver-model/README.md)：驱动模型（设备 capability 校验）
- [80-testing/README.md](../80-testing/README.md)：安全测试（纯 C LSM + capability 测试）
- [90-observability/README.md](../90-observability/README.md)：安全审计（capability 操作审计）

---

## 6. 参考材料

- Linux 6.6 `security/security.c`（LSM 框架核心）
- Linux 6.6 `security/landlock/`（Landlock 实现）
- Linux 6.6 `security/keys/`（密钥环实现）
- Linux 6.6 `Documentation/admin-guide/lockdown.rst`（Lockdown）
- Linux 6.6 `include/linux/lsm_hooks.h`（LSM 钩子声明）
- seL4 项目（capability 安全模型参考，MCS / capDL）
- openEuler 纯 C LSM 模式（技术参考，不共享代码）

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，README + 01 + 02 + 03 文档奠基（含 capability 数据模型、CNode/MDB 派生链、POSIX 41 ID、令牌 7 状态、Cupolas blob、4 值策略裁决、Vault backend、syscall 592-600、`LSM_ORDER_FIRST` 共存） |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap / IRON-9 v3 四层模型五大技术选型声明（**纯 C LSM 不使用 BPF LSM 对齐 openEuler 为核心**）；新增 Airymax Unify Design 映射（A-ULS Capability + 纯 C LSM、A-IPC IPC 安全 + io_uring 加固为核心）；新增 4 个文档（04-ipc-data-plane-autonomy / 05-ipc-control-plane-reconciliation / 06-io-uring-hardening / 07-airy-lsm-design）；文档索引对齐实际目录文件 |

---

> **文档结束** | agentrt-linux 安全加固设计 v1.0 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
