Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 云原生 Agent 部署设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）云原生工程体系主索引——K8s CRD、containerd-shim、超节点 OS 与 A-ULS 统一生命周期\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[AirymaxOS 总览](../README.md)

---

## 1. 模块概述

`150-cloudnative/` 是 AirymaxOS 文档体系中**智能体操作系统在云原生场景下的核心工程保障**。它继承 Linux 发行版与云原生社区 15+ 年沉淀的容器编排哲学（OCI 标准 + containerd + Kubernetes + CNI + CSI），并在其上扩展智能体操作系统专属的 Agent 容器化、超节点 OS、Agent 即 CRD、MemoryRovol CSI 等。

AirymaxOS 与生俱来具备云原生亲和力：sched_tac 调度语义与 K8s 调度器对齐，A-IPC（AgentsIPC 协议）可作为 K8s CRD 表达，MemoryRovol 记忆卷载可作为 CSI 卷挂载。Agent 应用在 AirymaxOS 上既可以传统进程形态运行，也可以容器化形态运行，两种形态共享同一套 SDK 与契约。

本模块承担五项核心职责：

1. **K8s CRD 设计**：Agent 应用作为 Kubernetes Custom Resource，声明式管理（Token 预算 + 记忆卷载 + 调度策略）。
2. **containerd-shim 集成**：containerd shim 承载 Agent 容器生命周期，对接 A-ULS 的 8 态生命周期监管。
3. **CNI 网络策略**：Agent 网络隔离与策略，基于 CNI + Calico/Cilium。
4. **超节点 OS**：超节点（Supernode）场景下的容器编排，NUMA 感知调度 + 跨 die 迁移 + CXL 池化。
5. **MemoryRovol CSI**：记忆卷载作为 CSI 卷，跨容器挂载与跨节点迁移。

---

## 2. 技术选型声明

agentrt-linux v1.0 在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度做出**不可妥协**的技术选型。本目录所有云原生设计必须以本声明为基线——containerd-shim 不得依赖 sched_ext、IPC 传输不得使用 page flipping、安全沙箱不得依赖 BPF LSM。

| # | 技术维度 | 选定方案 | 明确不采用 | 选定理由 |
|---|---------|---------|-----------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类，通过 `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，**不定义新的调度类宏** | **不使用 sched_ext**（不引入 eBPF 调度器、不依赖 `SCHED_EXT=7`、不使用 SCHED_AGENT 宏） | 保留 Linux 6.6 主线调度器稳定性，超节点 NUMA 感知调度基于既有调度类，ABI 稳定 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：containerd-shim 与内核通过 io_uring 命令操作码零拷贝通信，registered buffer + mmap 共享页 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | IORING_OP_URING_CMD 在 Linux 6.6 已稳定，避免 page flipping 引发的 DMA 映射复杂性与跨架构兼容问题 |
| 3 | **安全钩子** | **纯 C LSM**：Agent 容器安全策略由纯 C LSM（`airy_lsm`）通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | 纯 C LSM 保证容器安全策略可审计性与形式化验证可行性 |
| 4 | **内存分配** | **alloc_pages + mmap**：MemoryRovol CSI 共享内存与超节点 CXL 池化基于 `alloc_pages` + `remap_pfn_range` | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | 提供跨架构（x86/ARM/RISC-V）一致的内存语义，CXL 池化跨节点行为一致 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | v2 三层模型（升级为 v3，新增 [DSL] 降级生存层） | [DSL] 降级生存层确保 [SC] 损坏时系统仍可降级运行，CRD/shim 类型安全 |

> 详细技术选型声明见上级文档 [AirymaxOS 总览](../README.md) §2 与 [`10-architecture/10-unify-design.md`](../10-architecture/10-unify-design.md) SSoT 声明。

---

## 3. 文档索引

`150-cloudnative/` 由 6 个文档构成（含本 README）：

```
150-cloudnative/
├── README.md                       # 本文件 — 云原生主索引（v1.0）
├── 01-k8s-crd-design.md           # K8s CRD 设计 + 控制器（v1.0）
├── 02-containerd-shim.md          # containerd shim 集成 + Agent 容器生命周期（v1.0）
├── 03-cni-network-policy.md       # CNI 网络策略 + Agent 网络隔离（v1.0）
├── 04-supernode-os.md             # 超节点 OS（NUMA 感知 + 跨 die 迁移 + CXL 池化）（v1.0）
└── 05-memoryrovol-csi.md          # MemoryRovol CSI 驱动 + 三阶段 gRPC + 跨节点迁移（v1.0）
```

### 3.1 各文档定位

| 文档 | 核心问题 | 主要产物 |
|------|---------|---------|
| README.md | 云原生体系全貌？ | 7 层分层 + A-ULS/A-IPC 映射 + 技术选型声明 |
| 01-k8s-crd-design.md | Agent 怎么声明式管理？ | Agent CRD Schema + 控制器 + 调谐循环 |
| 02-containerd-shim.md | 容器怎么承载 Agent？ | containerd shim + Agent 容器生命周期 + A-ULS 8 态对接 |
| 03-cni-network-policy.md | Agent 网络怎么隔离？ | CNI 插件 + 网络策略 + 多租户网络 |
| 04-supernode-os.md | 超节点怎么编排？ | NUMA 感知调度 + 跨 die 迁移 + CXL 池化 + 整机快照 |
| 05-memoryrovol-csi.md | 记忆怎么跨容器挂载？ | CSI 驱动 + L1-L4 层级挂载 + 跨节点迁移 |

### 3.2 云原生分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 容器运行时 | containerd + OCI runtime | 容器隔离 |
| L2 | 容器网络 | CNI + Calico / Cilium | 容器网络 |
| L3 | 容器存储 | CSI + MemoryRovol 卷 | 容器存储 |
| L4 | 编排平台 | Kubernetes + CRD | 集群编排 |
| **L5** | **Agent 容器化** | **AirymaxOS 专属** | **Agent 镜像规范** |
| **L6** | **超节点 OS** | **AirymaxOS 专属** | **超节点容器编排** |
| **L7** | **Agent CRD** | **AirymaxOS 专属** | **Agent 即 K8s 资源** |

### 3.3 Agent CRD 示例

```yaml
apiVersion: agent.airymaxos.dev/v1
kind: Agent
metadata:
  name: cognition-agent-01
spec:
  image: registry.airymaxos.dev/cognition:v1.0.1
  tokenBudget: 1000000
  memoryRovol:
    enabled: true
    layers: [L1, L2, L3, L4]
  cognition:
    cycle: CoreLoopThree
  scheduler: SCHED_DEADLINE   # sched_tac 调度类（不使用 sched_ext）
```

---

## 4. Airymax Unify Design 映射

Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）在云原生体系中的关系如下。云原生是 A-ULS 超节点生命周期统一监管与 A-IPC containerd-shim IPC 的主要落地场景。

| Unify 模块 | 全称 | 在本目录的体现 | 对应组件 |
|-----------|------|--------------|---------|
| **A-ULS** | Unified Lifecycle Supervision Framework | **核心**：超节点 OS 统一生命周期管理、containerd-shim 对接 Agent 8 态生命周期、超节点级故障冷酷执法与温情裁决 | 超节点 OS + containerd-shim + Macro-Supervisor |
| **A-IPC** | Unified Airymax IPC Fabric | **核心**：containerd-shim 与内核通过 IORING_OP_URING_CMD IPC、Agent 容器间 128B 消息头通信 | containerd-shim IPC fastpath |
| **A-UEF** | Unified Error and Fault Framework | Agent 容器故障码传递、CRD 状态条件（Conditions）映射 A-UEF 错误码 | CRD status.conditions |
| **A-UCS** | Unified Configuration Subsystem | CRD spec 配置热重载、超节点 Token 预算管理、ConfigMap 同步 | K8s CRD spec + ConfigMap |
| **A-ULP** | Unified Logging and Printk Subsystem | Agent 容器日志采集、MemoryRovol CSI 持久化、超节点日志聚合 | CSI 驱动 + 日志 sidecar |

### 4.1 A-ULS 超节点生命周期统一监管

超节点（Supernode）场景下，A-ULS 将单机 Agent 8 态生命周期扩展为**超节点级统一监管**：

- **超节点 Macro-Supervisor**：跨节点 Agent 协同裁决，单点故障时内核 watchdog 直接重启
- **NUMA 感知调度**：sched_tac 调度类 + `sched_setaffinity` 实现 NUMA 感知 Agent 放置
- **跨 die 迁移**：Agent 状态迁移基于 A-ULS 8 态机，迁移过程中 IPC 队列冻结（`ring->frozen`）
- **整机快照恢复**：超节点级整机快照，基于 MemoryRovol 8 步迁移协议

### 4.2 A-IPC containerd-shim IPC

containerd-shim 与内核通过 A-IPC 的 **IORING_OP_URING_CMD** 进行零拷贝 IPC（**不使用 page flipping**）：

```c
/* containerd-shim ↔ kernel IPC —— registered buffer + io_uring_cmd */
struct airy_ipc_cmd cmd = { .op = AIRY_IPC_OP_SEND, ... };
io_uring_cmd_submit(ring_fd, &cmd);  /* 零拷贝，不交换物理页 */
```

Agent 容器间通信复用 A-IPC 的 128B 消息头协议（[SC] `ipc.h` 共享契约），数据面自治三原则确保控制面（shim）故障不影响数据面（Agent 间消息）。

### 4.3 同源 agentrt gateway（IRON-9 v3）

云原生体系遵循 **IRON-9 v3 四层模型**与 agentrt gateway 同源：[SC] 共享 `ipc.h`（128B 消息头），[SS] 语义同源（gateway 网关语义），[DSL] 降级生存，[IND] 各自独立实现（agentrt `gateway_d` 进程 ↔ AirymaxOS K8s Ingress Controller）。

---

## 5. 相关文档

### 5.1 上级与架构文档
- [AirymaxOS 总览](../README.md) —— 文档体系顶层纲领（v1.0）
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（五模块 SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 四层模型

### 5.2 模块与接口文档
- [20-modules/06-cloudnative.md](../20-modules/06-cloudnative.md) —— cloudnative 子仓设计
- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— AgentsIPC 协议（A-IPC）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— IPC fastpath（IORING_OP_URING_CMD）
- [20-modules/04-memory.md](../20-modules/04-memory.md) —— memory 子仓（MemoryRovol）
- [40-dataflows/03-ipc-flow.md](../40-dataflows/03-ipc-flow.md) —— IPC 数据流

### 5.3 关联模块文档
- [100-operations/README.md](../100-operations/README.md) —— 部署运维
- [90-observability/README.md](../90-observability/README.md) —— 可观测性（容器监控）
- [140-application-development/README.md](../140-application-development/README.md) —— Agent 应用开发（SDK + 部署契约）
- [170-performance/README.md](../170-performance/README.md) —— 性能工程（IPC 性能 SLO）
- [110-security/README.md](../110-security/README.md) —— 安全加固（容器沙箱）

### 5.4 参考材料
- Linux 6.6 容器子系统（cgroup v2 / namespace）
- OCI 标准（Open Container Initiative）
- containerd / Kubernetes / CNI / CSI / CRI 规范

---

## 6. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，5/5 文档完成（CRD + shim + CNI + 超节点 + CSI） |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）；IORING_OP_URING_CMD（不使用 page flipping）；纯 C LSM（不使用 BPF LSM）；alloc_pages + mmap（不使用 DMA 一致性内存）；IRON-9 v3 四层模型（新增 [DSL] 降级生存层）；新增 Airymax Unify Design 五模块映射（A-ULS 超节点统一生命周期 / A-IPC containerd-shim IPC） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | 云原生模块 v1.0 | 共 6 文档 | 维护者：开源极境工程与规范委员会 | Agent 应用可容器化形态运行
