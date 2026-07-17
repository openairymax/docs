Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 已知限制与设计边界

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）已知限制集中登记文档——参照 seL4 `CAVEATS.md` 工程实践，集中文档化所有已知设计限制、平台验证覆盖、形式化验证边界、兼容性降级策略与已延期设计项。各子模块文档中已有的"已知问题"章节仅作为子模块内部记录，**本文档为跨子仓的集中索引与权威登记**。\
> **文档版本**：0.1.1（2026-07-16）\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **保密级别**：开源公开\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **参照基准**：seL4 `CAVEATS.md`（258 行，2014, General Dynamics C4 Systems, GPL-2.0-only）

***

## 目录

- [1. 实现正确性与形式化验证边界](#1-实现正确性与形式化验证边界)
- [2. 平台与架构验证覆盖](#2-平台与架构验证覆盖)
- [3. 实时性限制](#3-实时性限制)
- [4. SMP 对称多处理器支持](#4-smp-对称多处理器支持)
- [5. 发行版与硬件兼容性限制](#5-发行版与硬件兼容性限制)
- [6. POSIX 兼容性限制](#6-posix-兼容性限制)
- [7. 安全模型限制](#7-安全模型限制)
- [8. 已知文档问题登记](#8-已知文档问题登记)
- [9. 已延期到 1.0.1 的设计项](#9-已延期到-101-的设计项)
- [10. 子仓已知限制索引](#10-子仓已知限制索引)

***

## 1. 实现正确性与形式化验证边界

### 1.1 形式化验证路径

agentrt-linux 1.0.1 阶段的形式化验证路径如下：

| 验证路径             | 工具                               | 覆盖范围                          | 阶段     | 状态              |
| ---------------- | -------------------------------- | ----------------------------- | ------ | --------------- |
| IPC Fastpath 不变量 | Isabelle/HOL (l4v)               | `airy_ipc_fastpath()` 关键路径    | 1.0.1+ | 设计中             |
| Rust 子仓          | Rust Kani                        | cognition/cloudnative Rust 代码 | 1.0.1+ | 设计中             |
| 编译期断言            | `BUILD_BUG_ON` / `static_assert` | 全量内核 C 代码                     | 0.1.1  | 已落地（OS-KER-229） |
| 运行时断言            | `WARN_ON_ONCE` + 恢复代码            | 关键路径                          | 1.0.1+ | 设计中             |

### 1.2 不使用的验证路径

**不引入 C parser 形式化验证路径**：seL4 的 `unverified_compile_assert` 宏是为 C parser 兼容性而存在的工程妥协（参见 [C\_Cpp\_coding\_style.md §6.5](../50-engineering-standards/10-coding-style/C_Cpp_coding_style.md) OS-KER-229 反向声明）。agentrt-linux 的形式化验证路径不经过 C parser，因此不引入此宏。

### 1.3 验证不覆盖的范围

参照 seL4 `CAVEATS.md` 的"Implementation Correctness"章节声明，agentrt-linux 的形式化验证**不覆盖**以下内容：

- 机器码生成（依赖 GCC/Clang 编译器正确性）
- 链接器行为
- 启动代码（boot / decompressor）
- 缓存与 TLB 管理
- 设备树解析
- 固件（UEFI / U-Boot）

**这意味着**：即使 IPC Fastpath 形式化验证完成，也不能保证二进制层面完全无缺陷。生产部署应结合运行时不变量检查与回归测试。

***

## 2. 平台与架构验证覆盖

### 2.1 目标架构

| 架构              | 内核态支持 | 用户态支持 | 1.0.1 验证状态   |
| --------------- | ----- | ----- | ------------ |
| x86\_64         | ✅     | ✅     | 编译验证 + KUnit |
| arm64 (AArch64) | ✅     | ✅     | 编译验证         |
| riscv64         | ✅     | ✅     | 编译验证（设计级）    |

### 2.2 不支持的架构

- 32 位架构（x86 / ARMv7）—— 不在 1.0.1 目标范围
- MIPS / PowerPC —— 不在 agentrt-linux 目标范围

### 2.3 CI 编译验证矩阵

参见 [70-build-system/03-ci-cd-pipeline.md](../70-build-system/03-ci-cd-pipeline.md) 的 Kbuild 矩阵：

| 配置           | x86\_64     | arm64       | riscv64     |
| ------------ | ----------- | ----------- | ----------- |
| defconfig    | ✅           | ✅           | ✅           |
| allnoconfig  | ✅           | ✅           | ✅           |
| allmodconfig | ✅           | ✅           | ✅           |
| 编译器          | gcc + clang | gcc + clang | gcc + clang |

***

## 3. 实时性限制

参照 seL4 `CAVEATS.md` 的"Real Time"章节声明，agentrt-linux 的实时性限制如下：

### 3.1 不可抢占操作

agentrt-linux 内核存在少量不可抢占操作（与 Linux 6.6 基线一致）：

- RCU 宽限期同步（`synchronize_rcu()`）
- 大页拆分与合并
- 内存压缩（`__alloc_pages_slowpath()`）
- 部分 IPC 批量提交路径（`airy_ipc_send_batch()`）

**降级策略**：低延迟场景应避免在热路径中触发上述操作，可通过 `airy_sched_set_latency_class()` 设置 `AIRY_SCHED_AGENT` 调度类并启用 `AIRY_LATENCY_LOW` 标志。

### 3.2 sched\_ext 不可用约束与方案 C-Prime 替代

**约束声明**：标准 Linux 6.6 主线**不包含** sched\_ext（SCHED\_EXT BPF 调度器框架于 Linux 6.12 才合入主线）。agentrt-linux 锁定 Linux 6.6 内核基线（ADR-013），因此**不可使用** sched\_ext 及其 `CONFIG_SCHED_CLASS_EXT`/`CONFIG_BPF_SCHED` 配置，亦无法依赖 `struct sched_ext_ops`、`scx_bpf_*`、`SCX_DSQ_*` 等 sched\_ext BPF API。

**方案 C-Prime 替代**：agentrt-linux 采用**方案 C-Prime**（SCHED\_DEADLINE/SCHED\_FIFO/EEVDF + seL4 MCS 映射）作为用户态调度器，替代 sched\_ext：

- **策略名**：`AIRY_SCHED_AGENT`（用户态调度器策略名，非内核调度类编号）；**禁止**定义 `SCHED_AGENT` 内核调度类宏（避免与内核调度类编号冲突）。
- **基于原生策略**：复用 Linux 6.6 内核原生 SCHED\_DEADLINE/SCHED\_FIFO/EEVDF + cgroup v2 cpuset，无需向前移植 sched\_ext。
- **MCS 映射**：seL4 MCS（SchedContext / refill 机制）映射到 SCHED\_DEADLINE 带宽配额 + cgroup `cpu.max`/`cpu.weight`，实现等价的常量带宽管理。
- **性能优势**：方案 C-Prime 走标准 6.6 内核调度路径，避免 sched\_ext BPF 验证器开销与 kfunc 上下文切换，P99 调度延迟与抖动均优于 sched\_ext。
- **降级**：用户态调度器异常或 watchdog 超时时，自动回退至 EEVDF（CFS）默认调度，确保系统完整性。

### 3.3 WCET 配置

agentrt-linux 内核默认 WCET 配置值仅作为参考默认值，生产部署时需根据具体使用场景重新评估：

- 静态系统（已知代码集）：可设置低 WCET 值
- 动态系统（含不可信代码）：需设置较高 WCET 值

***

## 4. SMP 对称多处理器支持

参照 seL4 `CAVEATS.md` 的"SMP"章节声明，agentrt-linux 的 SMP 支持状态如下：

### 4.1 SMP 支持状态

| 配置                | 支持状态    | 验证状态              |
| ----------------- | ------- | ----------------- |
| 单核（UP）            | ✅ 完全支持  | KUnit + kselftest |
| SMP（≤ 8 核）        | ✅ 支持    | 编译验证 + 烟雾测试       |
| SMP（> 8 核 / NUMA） | ⚠️ 部分支持 | 仅编译验证             |
| 非对称多处理（ASMP）      | ❌ 不支持   | —                 |

### 4.2 SMP + capability 模型

capability 模型在 SMP 配置下不支持强非干扰性（strong intransitive non-interference）——多核之间的时序通道无法在内核层完全消除。对强隔离场景，建议：

1. 使用 CPU 隔离（`isolcpus=` 启动参数）
2. 关闭超线程（`nosmt`）
3. 使用单核配置 + 多机部署

### 4.3 SMP + MCS

agentrt-linux 不引入 MCS（Mixed Criticality Systems）配置——此为 seL4 特有机制，agentrt-linux 通过方案 C-Prime 的 `AIRY_SCHED_AGENT` + token budget 实现等价能力（seL4 MCS 语义映射到 SCHED\_DEADLINE 带宽配额，详见 [§3.2](#32-sched_ext-不可用约束与方案-c-prime-替代)）。

***

## 5. 发行版与硬件兼容性限制

来源：[160-compatibility/05-cross-distro.md §13.1](../160-compatibility/05-cross-distro.md)

### 5.1 已知硬件/特性限制

| 限制             | 影响范围                  | 降级策略            |
| -------------- | --------------------- | --------------- |
| sched\_ext 不可用 | Linux 6.6 主线（6.12 才合入） | 采用方案 C-Prime（SCHED\_DEADLINE/SCHED\_FIFO/EEVDF）替代 |
| io\_uring 不支持  | RHEL 9（旧版内核）          | 降级至 unix socket |
| BPF 不可用        | 部分嵌入式发行版              | 降级至用户态过滤        |
| CXL 不可用        | 非 CXL 硬件              | 降级至本地内存         |
| PMEM 不可用       | 非 PMEM 硬件             | 降级至 DRAM        |

### 5.2 降级通知机制

当特性不可用时，内核通过 IPC 事件通知 Agent 应用：

```c
void airy_notify_fallback(const char *feature, const char *reason)
{
        log_write(LOG_WARN, "feature fallback: %s reason=%s",
                  feature, reason);
        airy_ipc_event_t event = {
                .type = AIRY_EVENT_FEATURE_FALLBACK,
                .feature = feature,
                .reason = reason,
        };
        airy_ipc_send_event(current_agent_id, &event);
}
```

详见 [160-compatibility/05-cross-distro.md §13.2](../160-compatibility/05-cross-distro.md)。

***

## 6. POSIX 兼容性限制

来源：[160-compatibility/02-posix-compat.md §7.2](../160-compatibility/02-posix-compat.md)

### 6.1 已知不兼容项

| 项目               | 原因                  | 影响     |
| ---------------- | ------------------- | ------ |
| 部分 `ptrace()` 功能 | Cupolas 安全策略过滤      | 仅影响调试器 |
| `kexec_load()`   | 仅 root + Cupolas 限制 | 不影响应用  |
| 部分 `ioctl()`     | Cupolas 按需放行        | 按策略配置  |

### 6.2 兼容性测试矩阵

| 测试套件                               | 标准           | 通过率目标 |
| ---------------------------------- | ------------ | ----- |
| PCTS（POSIX Conformance Test Suite） | POSIX.1-2017 | 95%+  |
| LSB（Linux Standard Base）           | LSB 5.0      | 95%+  |
| LTP（Linux Test Project）            | syscall 兼容性  | 98%+  |
| glibc test suite                   | glibc 兼容     | 99%+  |

***

## 7. 安全模型限制

### 7.1 capability + LSM 融合模型

agentrt-linux 同时使用 capability（seL4 借鉴）和 LSM（Linux 6.6 原生）两层安全模型：

- **capability 层**：细粒度权限控制（agent spawn / GPU sched / NPU access）
- **LSM 层**：系统级安全策略（hooks 252 ID，参见 [110-security/01-lsm-framework.md](../110-security/01-lsm-framework.md)）
- **融合点**：capability 操作触发 LSM 钩子

**已知限制**：

1. **时序通道**：内核不提供时序通道保证，capability 检查的执行时间可能泄露权限信息
2. **Rowhammer**：内核不提供针对 Rowhammer 等硬件攻击的特定防护，但提供原语允许用户态系统配置内存映射以降低风险
3. **IOMMU 中断重映射**：1.0.1 阶段不提供 IOMMU 中断重映射支持，因此设备不能安全直通给不可信虚拟机

### 7.2 Cupolas 安全策略限制

参见 [110-security/03-capability-model.md §15](../110-security/03-capability-model.md)：

- capability 撤销操作在 SMP 配置下存在 TOCTOU 窗口
- 部分 `ptrace()` 功能被 Cupolas 过滤，影响调试器可用性

### 7.3 地址空间重用安全（VSpace Reuse）

参照 seL4 CAVEATS.md "Re-using Address Spaces" 章节登记。

**已知限制**：

1. **capability 撤销与 VSpace 重用的关系**：VSpace 在重用前必须完成所有 frame capability 的撤销（`airy_cap_delete()`），否则旧 capability 可能保留对已释放物理页的访问权限。SMP 配置下 capability 撤销存在 TOCTOU 窗口（参见 §7.2），重用前须确保所有 CPU 上的 capability cache 已刷新。
2. **Agent 容器迁移时的 VSpace 清理**：Agent 容器从节点 A 迁移到节点 B 时，节点 A 上的 VSpace 必须完整清理（撤销所有 frame cap + flush TLB + 释放页表），否则迁移后的 Agent 可能在节点 A 上保留幽灵访问权限。
3. **地址空间复用的 cache 维护**：VSpace 重用后须执行 `airy_cache_flush_vspace()` 确保旧映射不残留在 cache 中，避免新映射读到旧数据。ARM 平台需额外注意 I-cache 一致性。
4. **unmap 操作的异步语义**：`airy_vspace_unmap()` 在 SMP 配置下是异步的——返回后 TLB shootdown 可能尚未完成，需配合 `airy_tlb_sync()` 等待完成。1.0.1 阶段暂不支持异步 unmap 的批量优化。

### 7.4 IOMMU 与设备直通限制

参照 seL4 CAVEATS.md "Intel VT-d (IOMMU)" 章节登记。

**已知限制**：

1. **IOMMU 中断重映射**（§7.1 第 3 条扩展）：1.0.1 阶段不提供 IOMMU 中断重映射支持，因此设备不能安全直通给不可信虚拟机或不可信 Agent。设备直通仅允许在以下场景：
   - 设备已通过 Cupolas LSM 安全策略审查
   - 接收方为受信任的特权 Agent（`CAP_SYS_RAWIO` 持有者）
2. **MSI/MSI-X Remapping**：1.0.1 阶段不提供 MSI/MSI-X Remapping，直通设备的 MSI 中断可能被恶意设备滥用进行 DMA 攻击。临时缓解措施：禁止不可信设备直通 + 启用 IOMMU DMA 隔离（`airy_iommu_isolate_mode = strict`）。
3. **IOMMU 与 capability 的交互**：1.0.1 阶段 IOMMU 不纳入 capability 体系（不提供 per-agent IOMMU context cap），所有设备共享一个 IOMMU domain。1.0.1+ 计划引入 per-agent IOMMU context 实现细粒度设备隔离。
4. **ARM SMMU 支持**：1.0.1 阶段仅支持 x86\_64 的 VT-d，ARM SMMU v3 支持延期至 1.0.1+。

***

## 8. 已知文档问题登记

以下问题已在各子模块文档中登记，此处作为跨子仓索引：

### 8.1 P0 级问题（ABI 稳定性基础）

| 编号        | 问题                                            | 来源                                                                                                                | 状态  |
| --------- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | --- |
| P0-SYS-01 | README syscall 编号冲突（1001-1004 vs 512-631 编号段） | [140-application-development/07-syscall-registry.md §11.1](../140-application-development/07-syscall-registry.md) | 待修复 |

### 8.2 P1 级问题（文档同步/错误码一致性）

| 编号        | 问题                                      | 来源                                                                                                                | 状态                |
| --------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ----------------- |
| P1-BUD-01 | SDK 错误码与系统调用错误码不一致（-401/-701 vs -15/-4） | [140-application-development/04-token-budget.md §16.1](../140-application-development/04-token-budget.md)         | 待修复               |
| P1-SYS-02 | 生命周期文档交叉引用错误                            | [140-application-development/07-syscall-registry.md §11.2](../140-application-development/07-syscall-registry.md) | ✅ 已修复（2026-07-09） |
| P1-CAP-01 | README 文档索引未更新                          | [110-security/03-capability-model.md §15.1](../110-security/03-capability-model.md)                               | 待修复               |

### 8.3 维护者声明

参见 [50-engineering-standards/07-maintainers-and-governance.md §6.2](../50-engineering-standards/07-maintainers-and-governance.md)：

> **已无已知严重问题**——0.1.1 阶段所有 P0-CRITICAL 问题已在前序审查中修复。

***

## 9. 已延期到 1.0.1 的设计项

以下设计项已明确延期到 1.0.1 阶段实现，**不属于 0.1.1 奠基版本范围**：

### 9.1 工程标准遗留项

| 编号   | 遗留项                        | 说明                  |
| ---- | -------------------------- | ------------------- |
| —    | `tests` 简写（49 文件 / 177 处）  | 需统一为 `tests-linux`  |
| —    | OS-STD-2xx 运维域残留           | 需重新编号或迁移            |
| —    | OS-KER-001\~008 SSoT 内部不一致 | 需修复注册表              |
| R-04 | seL4 借鉴深化                  | ES-SEL4-6\~10 P2 增强 |
| —    | Rust 编号登记缺口                | OS-RUST 编号前缀待建立     |
| —    | Rust 驱动规范（§8.3）            | 待 1.0.1 Rust 引入后补充  |

### 9.2 形式化验证遗留项

| 编号 | 遗留项                            | 说明                       |
| -- | ------------------------------ | ------------------------ |
| —  | IPC Fastpath Isabelle/HOL 完整证明 | 1.0.1+ 阶段实施              |
| —  | Rust Kani 验证框架                 | cognition/cloudnative 子仓 |
| —  | capability + LSM 融合模型完整论证      | 1.0.1 M1 阶段（\~10h）       |

### 9.3 服务用户态化风险评估

ES-SEL4-3（服务用户态化）的落地风险待 1.0.1 阶段评估：

| 服务   | 用户态化风险           | 建议          |
| ---- | ---------------- | ----------- |
| VFS  | 中（框架保留内核态，FS 实现可用户态化）    | 保留 VFS 框架在内核，具体 FS 通过 FUSE 模型用户态化 |
| 网络栈  | 高（性能影响 + 协议兼容性）  | 保留内核态       |
| 驱动   | 中（设备覆盖范围 + 性能影响） | 1.0.1 仅非热路径 |
| 日志服务 | 低                | 1.0.1 可用户态化 |
| 监控服务 | 低                | 1.0.1 可用户态化 |

***

## 10. 子仓已知限制索引

### 10.1 索引表

| 子仓          | 已知限制来源                   | 详见                     |
| ----------- | ------------------------ | ---------------------- |
| kernel      | IPC Fastpath 形式化验证未完成    | [§1](#1-实现正确性与形式化验证边界) |
| services    | daemon 用户态化风险待评估         | [§9.3](#93-服务用户态化风险评估) |
| security    | capability + LSM 融合模型待论证 | [§7](#7-安全模型限制)        |
| memory      | CXL/PMEM 降级策略            | [§5](#5-发行版与硬件兼容性限制)   |
| cognition   | Rust Kani 验证框架未建立        | [§9.2](#92-形式化验证遗留项)   |
| cloudnative | Rust Kani 验证框架未建立        | [§9.2](#92-形式化验证遗留项)   |
| system      | Go 子仓无形式化验证              | —                      |
| tests-linux | kselftest 覆盖率待提升         | [§4.1](#41-smp-支持状态)   |

### 10.2 项目风险登记

参见 [50-engineering-standards/50-project-erp/project\_erp.md §10.2](../50-engineering-standards/50-project-erp/project_erp.md) 的"已知风险登记"章节。

***

## 11. 相关文档

- [seL4 CAVEATS.md](https://github.com/seL4/seL4/blob/master/CAVEATS.md) —— 参照基准（258 行，GPL-2.0-only）
- [10-architecture/08-threat-model.md](08-threat-model.md) —— 安全威胁模型
- [50-engineering-standards/10-coding-style/C\_Cpp\_coding\_style.md §6.5](../50-engineering-standards/10-coding-style/C_Cpp_coding_style.md) —— OS-KER-229 编译期断言
- [50-engineering-standards/04-engineering-philosophy.md](../50-engineering-standards/04-engineering-philosophy.md) —— OS-IRON-012 seL4 借鉴边界
- [160-compatibility/05-cross-distro.md §13](../160-compatibility/05-cross-distro.md) —— 发行版兼容性降级策略
- [160-compatibility/02-posix-compat.md §7](../160-compatibility/02-posix-compat.md) —— POSIX 兼容性
- [110-security/03-capability-model.md §15](../110-security/03-capability-model.md) —— capability 模型已知问题

***

## 12. 版本历史

| 版本    | 日期         | 变更                |
| ----- | ---------- | ----------------- |
| 0.1.1 | 2026-07-16 | 初始创建，集中登记 9 类已知限制 |

***

> **文档结束** | agentrt-linux（AirymaxOS）已知限制与设计边界
> **参照基准**： seL4 `CAVEATS.md`（258 行，2014, General Dynamics C4 Systems）
> **维护原则**： 本文档为跨子仓集中索引；各子模块内部"已知问题"章节仍作为子模块内部记录保留，本文档不替代子模块记录，仅作为跨子仓权威登记。

