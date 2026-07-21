Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）驱动模型设计
> **文档定位**：agentrt-linux（AirymaxOS）驱动子系统工程设计主索引（设备驱动管理 + VFIO 直通 + DMA 安全规范）\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[AirymaxOS 总览](../README.md)\
> **同源映射**：agentrt `daemons`（用户态服务）+ Linux 6.6 `drivers/base/`\
> **理论根基**：Linux device/driver/bus 三元组解耦 + Airymax K-3 服务隔离

---

## 1. 模块概述

agentrt-linux 驱动模型是连接内核能力与硬件/虚拟设备的核心抽象层。它继承 Linux 内核 30+ 年沉淀的 device/driver/bus 三元组解耦哲学，并在其上扩展智能体工作负载所需的"软驱动"概念——Agent 行为契约作为一类虚拟设备参与驱动模型。本目录覆盖三方面职责：

1. **设备驱动管理**：device/driver/bus/class 四元组抽象，platform 总线、misc 框架、`module_*_driver` 宏样板消除、`devm_` 资源托管扩展（含 Token 预算、记忆卷载、IPC 通道等智能体资源）。
2. **VFIO 直通**：将物理设备安全直通给用户态守护进程（dev_d 等），驱动模型负责设备生命周期与 IOMMU 隔离边界。
3. **DMA 安全规范**：所有 DMA 共享内存严格采用 `alloc_pages + mmap` 方案（不使用 DMA 一致性内存），通过 IOMMU 强制隔离，对齐sched_tac 调度约束下的零拷贝语义。

### 1.1 核心抽象

| 抽象 | 职责 | Linux 来源 |
|------|------|------------|
| **device** | 设备实例（硬件或虚拟） | `include/linux/device.h` |
| **driver** | 驱动逻辑（绑定到 device） | `include/linux/device/driver.h` |
| **bus** | 总线类型（匹配 device 与 driver） | `include/linux/device/bus.h` |
| **class** | 设备功能分类（电源/输入/网络等） | `include/linux/device/class.h` |

### 1.2 agentrt-linux 扩展

- **Agent 虚拟驱动**：将 Agent 行为契约作为虚拟设备，通过 `agent_driver_register()` 注册到 `agent_bus_type`
- **智能体资源托管**：`devm_` 资源管理扩展至 Token 预算、记忆卷载、IPC 通道等智能体资源
- **io_uring 设备 DMA**：通过 `IORING_OP_URING_CMD` 暴露设备 DMA 能力到用户态（A-IPC 数据面载体）

---

## 2. 技术选型声明

agentrt-linux v1.0 驱动模型在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度遵循 [AirymaxOS 总览](../README.md) §2 的不可妥协基线。驱动模型作为 A-IPC 设备 DMA 与 A-ULS 设备生命周期的承载者，五个维度的选型在本目录的具体落地如下：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 在本目录的落地 |
|---|---------|---------|----------------|--------------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 `SCHED_EXT=7` 调度类） | 驱动 IRQ 线程与 kworker 通过 `SCHED_FIFO` 注入优先级，设备中断响应在sched_tac 调度类下确定性可预测 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码暴露设备 DMA 能力 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | 设备 DMA 通过 `io_uring_cmd` 回调与用户态零拷贝交互，Ring Buffer 元数据 mmap 共享 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 `airy_lsm` 通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | 驱动 probe/remove 路径的 capability 校验通过纯 C LSM 钩子注入，对齐 openEuler |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | 设备 DMA 缓冲区、VFIO 直通共享页均通过 `alloc_pages(GFP_KERNEL) + mmap` 分配，跨架构（x86/ARM/RISC-V）一致 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | （v2 三层模型升级为 v3 四层模型，新增 [DSL] 降级生存层） | 驱动 UAPI 类型通过 `include/uapi/linux/airymax/uapi_compat.h` [SC] 头文件三路桥接；`memory_types.h` [SC] 头文件承载 DMA 缓冲区布局契约 |

### 2.1 IRON-9 v3 四层模型在驱动模型的归属

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 落地文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| DMA 缓冲区布局 | ● | — | — | ● | [01-device-model.md](01-device-model.md) |
| io_uring 设备 DMA 命令 | — | — | ● | — | [02-platform-driver.md](02-platform-driver.md) |
| 设备 UAPI 类型桥接 | ● | — | — | — | [../50-engineering-standards/11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md) |
| 设备生命周期语义 | — | ● | — | — | [../20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) |

---

## 3. 文档索引

本目录现有文档（v1.0 范围）：

```
60-driver-model/
├── README.md                       # 本文件（v1.0）
├── 01-device-model.md              # device/driver/bus 三元组详解 + DMA 安全规范
└── 02-platform-driver.md           # platform 总线、SoC 设备与 io_uring 设备 DMA
```

| # | 文档 | 版本 | 内容概要 |
|---|------|------|---------|
| — | [README.md](README.md) | v1.0 | 驱动模型主索引（本文件） |
| 1 | [01-device-model.md](01-device-model.md) | v1.0 | device/driver/bus 三元组解耦、`devm_` 资源托管、DMA 安全规范（`alloc_pages + mmap`） |
| 2 | [02-platform-driver.md](02-platform-driver.md) | v1.0 | platform 总线、DT/ACPI 双源回退、Agent 虚拟设备、io_uring 设备 DMA（`IORING_OP_URING_CMD`） |

### 3.1 后续规划文档（1.0.1 版本）

以下文档在 1.0.1 版本完成，不在 v1.0 范围内：

- `03-devm-resource.md`：`devm_` 资源管理与生命周期（含智能体资源托管扩展）
- `04-misc-framework.md`：misc 框架与轻量级字符设备
- `05-agent-driver.md`：agentrt-linux 专属 Agent 虚拟设备驱动
- `06-vfio-passthrough.md`：VFIO 直通与 IOMMU 隔离
- `07-driver-testing.md`：驱动测试方法（KUnit + kselftest）

---

## 4. Airymax Unify Design 映射

本目录与 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的关系详见 [AirymaxOS 总览](../README.md) §5。驱动模型主要承载 **A-IPC**（设备 DMA 数据面）与 **A-ULS**（设备生命周期监管），其余三模块为辅助关系。

| Unify 模块 | 关系 | 在本目录的体现 |
|-----------|------|--------------|
| **A-UEF** | 辅助 | 设备 DMA 失败、probe 异常等错误码统一使用 A-UEF 的 `AIRY_E*` / `AIRY_FAULT_*` 码空间（[SC] `error.h`） |
| **A-ULP** | 辅助 | 驱动 probe/remove 路径的日志通过 A-ULP Ring Buffer 写入（128B 固定记录） |
| **A-UCS** | 辅助 | 驱动运行时参数（如 DMA 缓冲区大小、IRQ 亲和性）通过 A-UCS 的 sysctl/JSON 双向热重载 |
| **A-ULS** | **核心** | 设备生命周期监管——probe 失败、remove 超时、DMA 异常由 A-ULS 的 Micro-Supervisor（内核冷酷执法）+ Macro-Supervisor（用户温情裁决）双层接管；驱动作为 Agent 8 态生命周期中 SPAWNING→READY 的资源就绪承载者 |
| **A-IPC** | **核心** | io_uring 设备 DMA 传输——通过 `IORING_OP_URING_CMD` 暴露设备 DMA 能力到用户态，是 A-IPC 数据面自治三原则（Ring 生命周期解耦 / 离线缓存校验 / Reconciliation）的物理载体 |

### 4.1 Unify Design 权威源引用

- A-IPC 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §8
- A-ULS 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §7
- IPC 协议契约：[../30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)
- IPC fastpath：[../30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)

---

## 5. 相关文档

- [AirymaxOS 总览](../README.md)：v1.0 技术选型声明 + 20 子目录索引
- [架构设计层](../10-architecture/README.md)：Unify Design 总纲 + IRON-9 v3 四层模型
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md)：Airymax Unify Design 总纲（SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（SSoT）
- [20-modules/01-kernel.md](../20-modules/01-kernel.md)：kernel 子仓设计（io_uring + IORING_OP_URING_CMD）
- [20-modules/02-services.md](../20-modules/02-services.md)：services 子仓设计（dev_d 设备守护进程）
- [20-modules/04-memory.md](../20-modules/04-memory.md)：memory 子仓设计（alloc_pages + mmap 同源）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)：IPC fastpath（设备 DMA 通过 io_uring_cmd）
- [50-engineering-standards/11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md)：[SC] 头文件类型桥接规则
- [70-build-system/README.md](../70-build-system/README.md)：驱动构建系统
- [80-testing/README.md](../80-testing/README.md)：驱动测试方法
- [110-security/README.md](../110-security/README.md)：纯 C LSM + 设备 capability 校验

---

## 6. 参考材料

- Linux 6.6 `drivers/base/platform.c`（platform 总线实现）
- Linux 6.6 `include/linux/platform_device.h`（`module_platform_driver` 宏）
- Linux 6.6 `Documentation/driver-api/`（驱动 API 文档）
- Linux 6.6 `drivers/vfio/`（VFIO 框架参考）
- seL4 sDDF（seL4 设备驱动框架，设计思想参考）

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，README + 01 + 02 文档奠基，确立 device/driver/bus 三元组核心机制 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap / IRON-9 v3 四层模型五大技术选型声明；新增 Airymax Unify Design 映射（A-IPC 设备 DMA + A-ULS 设备生命周期）；文档索引对齐实际目录文件 |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | agentrt-linux 驱动模型设计 v1.0 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
