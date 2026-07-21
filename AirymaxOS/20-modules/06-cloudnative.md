Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 云原生设计文档

> **文档定位**：agentrt-linux（AirymaxOS）云原生设计文档（cloudnative，极境云原生）\
> **文档版本**：v1.0.1\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **核心约束**：IRON-9 v3 同源且部分代码共享——与 agentrt 用户态 gateway/sdk 通过 \[SC] 共享契约层 + \[SS] 语义同源层协作，\[IND] K8s/containerd/OCI/CNI 实现独立\
> **子仓编号**：06\
> **子仓代号**：极境云原生（Airymax Cloud Native）\
> **设计基准**：K8s + containerd + OCI + agentctl + OpenTelemetry + DPU/IPU + 超节点 OS\
> **同源 agentrt**：gateway + sdk\
> **横切关注点**：云原生是横切关注点（cross-cutting concern），贯穿调度（CRD 调度器）、IPC（gateway\_d io\_uring 通道）、eBPF（服务网格数据平面）、记忆卷载（容器快照/迁移）4 大数据流

***

## 目录

- [1. 子仓职责](#1-子仓职责)
- [2. 同源关系（IRON-9 v3 四层共享模型）](#2-同源关系iron-9-v3-四层共享模型)
- [3. 目录结构](#3-目录结构)
- [4. 核心特性](#4-核心特性)
- [5. 微内核思想体现](#5-微内核思想体现)
- [6. IRON-9 v3 四层共享模型落地](#6-iron-9-v3-四层共享模型落地)
- [7. agentrt-linux 工程基线](#7-agentrt-linux-工程基线)
- [8. 前沿理论参考](#8-前沿理论参考)
- [9. 与其他子仓的协作](#9-与其他子仓的协作)
- [10. 里程碑（M0-M8）](#10-里程碑m0-m8)
- [11. agentrt 一致性检查](#11-agentrt-一致性检查)
- [12. 相关文档](#12-相关文档)
- [13. 参考](#13-参考)

***

## 1. 子仓职责

`cloudnative` 是 agentrt-linux（AirymaxOS）的云原生基础设施子仓，承担以下核心职责：

1. **Kubernetes 集成 \[IND]**：将 Agent 作为 Kubernetes CRD（Custom Resource Definition）原生集成，控制器 reconcile 期望状态。
2. **containerd shim \[IND]**：提供 containerd shim v2，实现 Agent 容器化运行（Wasm/runc/进程三模式）。
3. **OCI 镜像规范 \[IND]**：遵循 OCI（Open Container Initiative）镜像规范，支持 Wasm 模块作为镜像层。
4. **CNI 插件 \[IND]**：提供基于 eBPF 的 CNI 插件，实现无 sidecar 服务网格数据平面。
5. **agentctl \[SS]**：对标 kubectl 的 Agent 管理命令行工具，与 agentrt sdk 管理接口语义同源。
6. **OpenTelemetry 可观测性 \[IND]**：集成 OpenTelemetry 提供统一 Tracing/Metrics/Logs 可观测性。
7. **DPU/IPU 卸载支持 \[IND]**：支持 NVIDIA BlueField / Intel IPU 硬件卸载。
8. **超节点 OS \[IND]**：基于 agentrt-linux 超节点 OS 实现多 die/多 chip 统一管理与跨 die 迁移。
9. **gateway\_d 网关 \[SS]**：与 agentrt gateway 网关语义同源，升级为 K8s Ingress + OS 级 gateway\_d 守护进程，IPC 消息头 \[SC] 共享。

### 1.1 横切关注点声明

云原生是横切关注点（cross-cutting concern），贯穿 agentrt-linux 全部 4 大数据流：

| 数据流      | 云原生切入点                                | 同源标注   |
| -------- | ------------------------------------- | ------ |
| 调度数据流    | CRD 自定义调度器 + K8s 默认调度                 | \[IND] |
| IPC 数据流  | gateway\_d io\_uring 零拷贝通道 + 128B 消息头 | \[SC]  |
| eBPF 数据流 | CNI 服务网格数据平面 + eBPF 采集器               | \[IND] |
| 记忆卷载数据流  | 容器快照 + 跨节点迁移（与 MemoryRovol 协作）        | \[IND] |

***

## 2. 同源关系（IRON-9 v3 四层共享模型）

依据 IRON-9 v3 决策（ADR-014 确立 seL4 唯一来源，参见 `docs/architecture/adr/ADR-014.md`），agentrt（用户态 gateway + sdk）与 agentrt-linux（cloudnative）通过四层共享模型协作：

| 层次               | 共享程度                               | 云原生子系统内容                                                                                                                                                                                                                                                                                         | 组织方式                                                                                                                                        |
| ---------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **\[SC] 共享契约层**  | 完全共享代码                             | IPC 消息头（magic 0x41524531 'ARE1'）、128B 消息头结构（`struct airy_ipc_msg_hdr`）、SQE/CQE 操作码（gateway\_d io\_uring 通道）；capability 41 ID 枚举（容器沙箱 + CNI 网络策略）；CoreLoopThree 阶段枚举 + Thinkdual 模式枚举（Agent CRD cognition 字段引用）；v1.0.1 Capability Folding 的 `agent_caps[1024]` 静态数组与 64-bit Badge 布局 | `include/uapi/linux/airymax/` 10 个头文件（与 kernel/services/security/cognition/memory 共享），清单见 §6.1 |
| **\[SS] 语义同源层**  | 高层 API 语义同源（概念操作一致），签名因抽象层级不同而独立演进 | gateway 网关语义（agentrt gateway → K8s Ingress + gateway\_d）、SDK 管理接口语义（agentrt sdk → agentctl）、Agent 生命周期管理（agentrt lifecycle → CRD controller reconcile）、资源声明语义（agentrt resource → K8s resource spec）、可观测性 API（agentrt monitoring → OpenTelemetry）、containerd shim 生命周期、服务网格数据平面语义、CNI 网络语义等 10+ 项 | 各自独立实现                                                                                                                                      |
| **\[IND] 完全独立层** | 完全独立                               | K8s CRD 定义与控制器实现、containerd shim v2 实现、OCI 镜像规范实现、CNI 插件实现、OpenTelemetry 集成、DPU/IPU 卸载框架、超节点 OS、K8s 自定义调度器、准入 webhook、Multus 多 CNI                                                                                                                                                               | 各自独立仓库                                                                                                                                      |
| **\[DSL] 降级生存层** | 降级模式下最小可用 | 跨节点 Badge 一致性失效时（gossip 通道断开 ≥ 3s）gateway\_d 退化为单节点 Badge 本地校验 + 100ms 后强制 Epoch 推进；CRD controller 在 etcd 不可达时降级为本地 reconcile cache；containerd shim 在 OCI registry 不可达时降级为本地镜像缓存 + 只读运行 | 各自独立实现，但共享降级触发语义 |

### 2.1 维度对比

| 维度        | agentrt（gateway + sdk）          | agentrt-linux（cloudnative）      | 同源标注   |
| --------- | ------------------------------- | ------------------------------- | ------ |
| 网关语义      | gateway（应用层网关）                  | K8s Ingress + gateway\_d（OS 级）  | \[SS]  |
| IPC 消息头   | `struct airy_ipc_msg_hdr`（128B） | `struct airy_ipc_msg_hdr`（128B） | \[SC]  |
| IPC magic | 0x41524531 'ARE1'               | 0x41524531 'ARE1'               | \[SC]  |
| 管理接口      | sdk（应用层 CLI）                    | agentctl（对标 kubectl）            | \[SS]  |
| 部署形态      | 应用进程部署                          | K8s 原生部署（CRD + Controller）      | \[IND] |
| 容器化       | 进程隔离                            | containerd shim v2（OCI）         | \[IND] |
| 可观测性      | 自研监控                            | OpenTelemetry 标准化               | \[IND] |
| 网络策略      | 应用层 ACL                         | CNI + eBPF 数据平面                 | \[IND] |
| 容器沙箱      | capability 令牌                   | capability 41 ID + 容器沙箱         | \[SC]  |
| 认知循环      | CoreLoopThree 阶段                | CRD cognition 字段引用阶段枚举          | \[SC]  |
| 跨平台       | Linux/macOS/Windows             | Linux 6.6 专属                    | \[IND] |

### 2.2 同源传承要点

- 保留 agentrt gateway 的"网关"语义，升级为 K8s Ingress + gateway\_d（OS 级守护进程）\[SS]。
- 保留 agentrt sdk 的"管理接口"语义，升级为 agentctl（对标 kubectl）\[SS]。
- 保留 agentrt 的 Agent 生命周期管理语义，升级为 CRD controller reconcile \[SS]。
- 保留 agentrt 的资源声明语义，升级为 K8s resource spec \[SS]。
- IPC 消息头 \[SC] 共享，确保 gateway\_d 与 agentrt gateway 通信协议一致。
- 容器沙箱 capability \[SC] 共享，确保容器隔离策略两端一致。
- 认知循环阶段枚举 \[SC] 共享，确保 Agent CRD cognition 字段引用一致。
- 升级为 K8s 原生部署，获得声明式编排 + 自愈 + 弹性伸缩能力 \[IND]。

***

## 3. 目录结构

```
cloudnative/
├── kubernetes/             # K8s 集成（Agent 作为 CRD）[IND]
├── containerd-shim/        # containerd shim（Agent 容器化）[IND]
├── oci/                    # OCI 镜像规范 [IND]
├── cni/                    # CNI 插件（服务网格）[IND]
├── agentctl/              # agentctl（对标 kubectl）[SS]
├── observability/          # OpenTelemetry 可观测性 [IND]
├── dpu-ipu/                # DPU/IPU 卸载支持 [IND]
├── super-node-os/          # 超节点 OS（agentrt-linux 自研）[IND]
└── docs/
```

### 3.1 kubernetes/（K8s 集成）\[IND]

- `crds/`：Agent CRD 定义（Agent、AgentRuntime、AgentPipeline），cognition 字段引用 CoreLoopThree 阶段枚举 \[SC]。
- `controllers/`：控制器（Controller Runtime），reconcile 期望状态 \[SS]。
- `operators/`：Operator 实现 \[IND]。
- `schedulers/`：K8s 自定义调度器（与 `cognition/llm-scheduler` 协作）\[IND]。
- `webhooks/`：准入 webhook \[IND]。

### 3.2 containerd-shim/（containerd shim）\[IND]

- `airymax-shim`：Agent 容器 shim（基于 containerd shim v2 API）\[IND]。
- `runtime`：runtime spec 处理 \[IND]。
- `image-pull`：镜像拉取（与 OCI 协作）\[IND]。
- `snapshot`：快照支持（与 `memory` 协作，MemoryRovol 数据结构 \[SC]）\[IND]。

### 3.3 oci/（OCI 镜像规范）\[IND]

遵循 **OCI（Open Container Initiative）** 规范：

- `image-spec`：镜像规范实现 \[IND]。
- `runtime-spec`：运行时规范实现 \[IND]。
- `distribution-spec`：分发规范实现 \[IND]。
- `agent-image`：Agent 镜像构建工具 \[IND]。

### 3.4 cni/（CNI 插件）\[IND]

- `airymax-cni`：agentrt-linux CNI 插件 \[IND]。
- `service-mesh`：服务网格（基于 eBPF 数据平面）\[IND]。
- `network-policy`：网络策略（与 `security` 协作，capability 41 ID \[SC]）\[IND]。
- `multus`：Multus 多 CNI 支持 \[IND]。

### 3.5 agentctl/（对标 kubectl）\[SS]

agentctl 与 agentrt sdk 管理接口语义同源 \[SS]：

- `cmd/`：命令行入口 \[SS]。
- `api-client`：API 客户端（与 K8s API 协作，gateway\_d 通过 io\_uring IPC 通信 \[SC]）\[SS]。
- `plugins/`：插件机制（kubectl plugin 兼容）\[SS]。
- `completion`：命令补全 \[IND]。

### 3.6 observability/（OpenTelemetry 可观测性）\[IND]

集成 **OpenTelemetry** 标准：

- `tracing`：分布式追踪（trace\_id \[SC] 贯穿）\[IND]。
- `metrics`：指标采集 \[IND]。
- `logs`：日志聚合 \[IND]。
- `exporters`：导出器（Prometheus、Jaeger、Loki）\[IND]。
- `ebpf-collector`：eBPF 可观测性采集器 \[IND]。

### 3.7 dpu-ipu/（DPU/IPU 卸载支持）\[IND]

- `dpu-offload`：DPU 卸载框架（NVIDIA BlueField、Intel IPU）\[IND]。
- `network-offload`：网络卸载（OVS offload）\[IND]。
- `storage-offload`：存储卸载（NVMe-oF offload）\[IND]。
- `security-offload`：安全卸载（IPSec/TLS offload）\[IND]。

### 3.8 super-node-os/（超节点 OS）\[IND]

基于 **agentrt-linux 超节点 OS**：

- `topology`：超节点拓扑管理 \[IND]。
- `scheduling`：超节点调度（NUMA 感知）\[IND]。
- `migration`：跨节点迁移（与 MemoryRovol 协作，MemoryRovol L1-L4 数据结构 \[SC]）\[IND]。
- `snapshot`：超节点快照 \[IND]。

***

## 4. 核心特性

### 4.1 K8s 集成（Agent 作为 CRD）\[IND]

**Agent CRD**（cognition 字段引用 CoreLoopThree 阶段枚举 \[SC]）：

```yaml
apiVersion: airymaxos.io/v1
kind: Agent
metadata:
  name: my-agent
spec:
  runtime: wasm  # wasm | container | process
  image: airymaxos/my-agent:v1.0
  cognition:
    coreLoopThree:
      enabled: true      # 引用 PERCEPTION/THINKING/ACTION 阶段枚举 [SC]
    thinkdual:
      enabled: true      # 引用 SYSTEM1_FAST/SYSTEM2_SLOW 模式枚举 [SC]
  resources:
    gpu: 1
    npu: 0
    memory: 4Gi
  replicas: 3
```

**控制器**：监听 Agent CRD 变化，创建/更新/删除 Agent 实例，reconcile 语义与 agentrt Agent 生命周期管理同源 \[SS]。

### 4.2 containerd shim（Agent 容器化）\[IND]

**shim 架构**：

- 基于 containerd shim v2 API 实现 \[IND]。
- 支持 Wasm runtime（与 `cognition/wasm-runtime` 协作）\[IND]。
- 支持传统容器（runc）\[IND]。
- 支持 Agent 进程模式 \[IND]。

**优势**：

- 与 K8s 生态无缝集成 \[IND]。
- 复用 containerd 镜像分发 \[IND]。
- 支持快照与迁移（与 `memory` 协作）\[IND]。

### 4.3 OCI 镜像规范 \[IND]

遵循 **OCI 规范**：

- image-spec：镜像格式规范 \[IND]。
- runtime-spec：运行时规范 \[IND]。
- distribution-spec：分发规范 \[IND]。

**Agent 镜像**：

- 基于 OCI 镜像格式 \[IND]。
- 支持 Wasm 模块作为镜像层 \[IND]。
- 支持多架构（amd64、arm64）\[IND]。

### 4.4 CNI 插件（服务网格）\[IND]

**airymax-cni**：

- 基于 eBPF 的高性能 CNI 插件 \[IND]。
- 服务网格数据平面（替代 Istio sidecar）\[IND]。
- 网络策略（与 `security` 协作，capability 令牌 \[SC]）\[IND]。

**优势**：

- 无 sidecar，减少资源开销 \[IND]。
- eBPF 数据路径，高性能 \[IND]。
- 声明式网络策略 \[IND]。

### 4.5 agentctl（对标 kubectl）\[SS]

agentctl 与 agentrt sdk 管理接口语义同源 \[SS]：

**命令示例**：

```bash
# 创建 Agent
agentctl apply -f agent.yaml

# 查看 Agent 状态
agentctl get agents

# 查看 Agent 日志（trace_id [SC] 贯穿）
agentctl logs my-agent

# 进入 Agent shell
agentctl exec my-agent -- wasm-shell

# 扩缩容
agentctl scale agent my-agent --replicas=5

# 快照（与 MemoryRovol 协作 [SC]）
agentctl snapshot my-agent --output snapshot.tar
```

### 4.6 OpenTelemetry 可观测性 \[IND]

**统一可观测性**：

- Tracing：分布式追踪（OpenTelemetry Tracing，trace\_id \[SC] 贯穿）\[IND]。
- Metrics：指标采集（OpenTelemetry Metrics）\[IND]。
- Logs：日志聚合（OpenTelemetry Logs）\[IND]。

**eBPF 采集器**：

- 基于 eBPF 的零侵入采集 \[IND]。
- 系统调用追踪 \[IND]。
- 网络包追踪 \[IND]。
- 性能分析 \[IND]。

### 4.7 DPU/IPU 卸载支持 \[IND]

**支持的 DPU/IPU**：

| 厂商     | 产品           | 类型  |
| ------ | ------------ | --- |
| NVIDIA | BlueField-3  | DPU |
| Intel  | IPU E2100    | IPU |
| AMD    | Pensando DPU | DPU |
| ARM    | Neoverse     | NPU |

**卸载场景**：

- 网络卸载：OVS、VxLAN、负载均衡 \[IND]。
- 存储卸载：NVMe-oF、压缩、加密 \[IND]。
- 安全卸载：IPSec、TLS、防火墙 \[IND]。
- 虚拟化卸载：virtio、SR-IOV \[IND]。

### 4.8 超节点 OS（agentrt-linux 自研）\[IND]

基于 **agentrt-linux 超节点 OS**：

- 超节点拓扑：多 die/多 chip 统一管理 \[IND]。
- NUMA 感知调度：优先本地 die 调度 \[IND]。
- 跨 die 迁移：基于 MemoryRovol 跨 die 迁移（MemoryRovol L1-L4 数据结构 \[SC]）\[IND]。
- 超节点快照：整机快照与恢复 \[IND]。

### 4.9 gateway\_d 网关（与 agentrt gateway 同源）\[SS]

gateway\_d 是 agentrt gateway 在 OS 级的升级形态 \[SS]：

- 网关语义同源（agentrt gateway → gateway\_d）\[SS]。
- IPC 通道升级为 io\_uring 零拷贝，消息头 \[SC] 共享 \[SS]。
- 注册为 systemd 守护进程（与 `services` 协作）\[IND]。
- K8s Ingress 集成（南北向流量入口）\[IND]。

**gateway\_d io\_uring IPC 消息头** \[SC]（`include/uapi/linux/airymax/ipc.h`，与 agentrt 共享）：

```c
/* IPC 128B 消息头定义见 [SC] 共享契约层（SSoT），不就地重定义 */
#include <airymax/ipc.h>
/* 结构体名称：struct airy_ipc_msg_hdr（Layout C，物理宿主见
 * 50-engineering-standards/120-cross-project-code-sharing.md §Layout C） */
```

> **SSoT 声明**：本节 IPC 128B 消息头不再就地定义，以 `include/uapi/linux/airymax/ipc.h`（物理宿主见 `50-engineering-standards/120-cross-project-code-sharing.md` §Layout C）为单一数据源。结构体名称为 `struct airy_ipc_msg_hdr`，按 OLK 6.6 工程规范使用 `__aligned(64)` 对齐（**禁止**紧凑打包属性与显式 `__attribute__((aligned(...)))` 形式，参考 OLK 6.6 `struct ethhdr`/`struct iphdr` 手动安排字段自然对齐）与 `__u32`/`__u16`/`__u64`/`__u8` UAPI 类型。UAPI 标准路径 `include/uapi/linux/airymax/`，io\_uring 命令访问通过 `io_uring_cmd_to_pdu(cmd, pdu_type)` 安全宏读取 `pdu[32]` 字段，完成路径调用 `io_uring_cmd_done(cmd, ret, res2, issue_flags)` 4 参数签名，`security_uring_cmd` LSM 钩子为单参数 `struct io_uring_cmd *ioucmd`。`CONFIG_SECURITY_AIRY` 默认为 `n`。

***

## 5. 微内核思想体现

### 5.1 云原生作为独立模块 \[IND]

遵循微内核"模块化"原则：

- 云原生功能作为独立子仓 \[IND]。
- 与内核解耦，可独立演进 \[IND]。
- 通过标准接口（CRD、CNI、OCI）与其他子仓协作 \[IND]。

### 5.2 声明式接口 \[SS]

遵循 K8s 声明式范式：

- Agent 状态声明（CRD）\[IND]。
- 控制器 reconcile 期望状态（与 agentrt Agent 生命周期管理同源 \[SS]）\[SS]。
- 符合"机制与策略分离"原则 \[SS]。

### 5.3 标准化接口 \[IND]

- OCI 镜像规范：标准化镜像格式 \[IND]。
- CNI 网络接口：标准化网络插件 \[IND]。
- OpenTelemetry 可观测性：标准化观测协议 \[IND]。

### 5.4 横切数据流贯穿 \[SC]

云原生贯穿 4 大数据流，通过 \[SC] 共享契约层确保与 agentrt 协议一致：

- IPC 数据流：gateway\_d io\_uring 通道 + 128B 消息头 \[SC]。
- 安全横切：容器沙箱 capability 41 ID \[SC]。
- 认知横切：CRD cognition 字段引用 CoreLoopThree 阶段枚举 \[SC]。

***

## 6. IRON-9 v3 四层共享模型落地

### 6.1 \[SC] 共享契约层——10 个头文件

云原生模块的 \[SC] 层通过 10 个头文件与 agentrt 共享（清单与 kernel/services/security/memory/cognition 完全一致，SSoT 见 `50-engineering-standards/120-cross-project-code-sharing.md`）：

| 头文件                                              | 共享内容                                                                                                    | 云原生使用场景                          |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | -------------------------------- |
| `include/uapi/linux/airymax/error.h`              | 扩展错误码（`AIRY_ESEC_D_THROTTLED = -83`、`AIRY_ECAP_FROZEN = -82`、`AIRY_FAULT_URING_MALFORMED = 0x100A`、`AIRY_FAULT_AUDIT_TAMPER = 0x100B`） | gateway\_d 跨节点 RPC 错误映射 + CRD controller reconcile 错误透传 |
| `include/uapi/linux/airymax/log_types.h`          | trace\_id + 结构化日志类型枚举                                                                                   | OpenTelemetry trace\_id 贯穿 + journald 聚合 |
| `include/uapi/linux/airymax/memory_types.h`        | MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 持久化接口                                                            | 容器快照 + 跨节点迁移（与 memory 协作） |
| `include/uapi/linux/airymax/security_types.h`  | capability 41 ID 枚举 + Cupolas blob 布局 + v1.1 `agent_caps[1024]` 静态数组定义 + 64-bit Badge 布局（`Epoch<<48 \| RandomTag<<16 \| Perms`） | 容器沙箱 capability 隔离 + CNI 网络策略 + 跨节点 Badge 一致性 |
| `include/uapi/linux/airymax/cognition_types.h` | CoreLoopThree 阶段枚举（PERCEPTION/THINKING/ACTION）+ Thinkdual 模式枚举（SYSTEM1\_FAST/SYSTEM2\_SLOW）+ LLM 推理阶段枚举 | Agent CRD cognition 字段 + LLM 调度器 |
| `include/uapi/linux/airymax/sched.h`              | sched\_tac 调度类约束（禁止 SCHED\_AGENT 宏）+ task\_desc（magic 0x41475453 'AGTS'）+ vtime 衰减公式 | K8s 自定义调度器与 sched\_d 协作时的调度类约束 |
| `include/uapi/linux/airymax/ipc.h`                | IPC magic 0x41524531 'ARE1' + 128B `struct airy_ipc_msg_hdr` + SQE/CQE 操作码 + io\_uring ring 配置              | gateway\_d io\_uring 零拷贝 IPC 通道  |
| `include/uapi/linux/airymax/syscalls.h`           | v1.1 Syscall 24 槽位（4 核心 + 20 预留）：`airy_sys_call`(0)/`airy_sys_rovol_ctl`(1)/`airy_sys_sched_ctl`(2)/`airy_sys_clt_notify`(3) | agentctl 与 sec\_d/sched\_d/clt\_notify 协作的 ABI 契约 |
| `include/uapi/linux/airymax/uapi_compat.h`        | UAPI 兼容层宏（`__aligned(64)`、`__u32`/`__u16`/`__u64`/`__u8`）+ SQE128 模式 `cmd[80]` 扩展 | IPC 消息头对齐 + io\_uring SQE128 64B `__aligned(64)` |
| `include/uapi/linux/airymax/lsm_types.h`          | airy\_lsm 钩子 ID 250 枚举 + `LSM_ORDER_MUTABLE` 排序定义 + `uring_cmd` 单参数钩子签名 | CNI 网络策略的纯 C LSM（airy_lsm）联动 + 容器沙箱安全钩子 |

### 6.2 \[SS] 语义同源层——10 项 API 映射

高层 API 语义同源（概念操作一致），签名因抽象层级不同而独立演进。云原生模块的同源 API：

| 序号 | 语义           | agentrt 实现    | agentrt-linux 实现                     |
| -- | ------------ | ------------- | ------------------------------------ |
| 1  | gateway 网关   | 应用层 gateway   | K8s Ingress + gateway\_d             |
| 2  | SDK 管理接口     | sdk CLI       | agentctl（对标 kubectl）                 |
| 3  | Agent 生命周期   | 应用层 lifecycle | CRD controller reconcile             |
| 4  | 资源声明         | resource spec | K8s resource spec                    |
| 5  | 可观测性 API     | 自研监控          | OpenTelemetry 标准化                    |
| 6  | trace\_id 贯穿 | user\_data 字段 | io\_uring user\_data + OpenTelemetry |
| 7  | 容器化生命周期      | 进程隔离          | containerd shim v2                   |
| 8  | 服务网格数据平面     | 应用层 ACL       | eBPF 数据平面                            |
| 9  | CNI 网络语义     | 应用层网络         | CNI 插件                               |
| 10 | 网络策略         | 应用层 ACL       | capability + CNI network policy      |

### 6.3 \[IND] 完全独立层——12 项独立实现

| 序号 | 内容                     | 不共享原因                       |
| -- | ---------------------- | --------------------------- |
| 1  | K8s CRD 定义与控制器         | K8s 原生仅 agentrt-linux       |
| 2  | containerd shim v2 实现  | 容器运行时仅 agentrt-linux        |
| 3  | OCI 镜像规范实现             | 镜像规范仅 agentrt-linux         |
| 4  | CNI 插件实现               | 网络插件仅 agentrt-linux         |
| 5  | OpenTelemetry 集成       | 统一可观测性仅 agentrt-linux       |
| 6  | DPU/IPU 卸载框架           | 硬件卸载仅 agentrt-linux         |
| 7  | 超节点 OS                 | 多 die 管理仅 agentrt-linux     |
| 8  | K8s 自定义调度器             | K8s 调度器仅 agentrt-linux      |
| 9  | 准入 webhook             | K8s webhook 仅 agentrt-linux |
| 10 | Multus 多 CNI           | 多网络仅 agentrt-linux          |
| 11 | eBPF 采集器               | 内核 eBPF 仅 agentrt-linux     |
| 12 | systemd 集成（gateway\_d） | OS 级服务管理仅 agentrt-linux     |

### 6.4 \[DSL] 降级生存层落地

依据 IRON-9 v3 决策不允许从四层共享模型降级，\[DSL] 是云原生场景在跨节点通信故障下的最小可用语义层：

| 降级触发条件 | 降级后行为 | 恢复条件 |
| ------- | ------- | ------- |
| 跨节点 gossip 通道断开 ≥ 3s | gateway\_d 退化为单节点 Badge 本地校验 + 100ms 后强制 Epoch 推进 | gossip 通道恢复 + Epoch 同步成功 |
| etcd 不可达 | CRD controller 降级为本地 reconcile cache + 仅维护期望状态视图 | etcd 恢复后强制全量 reconcile |
| OCI registry 不可达 | containerd shim 降级为本地镜像缓存 + 只读运行 | registry 恢复后异步拉取新版本 |
| K8s API Server 不可达 | agentctl 降级为本地 cache 读写 + 仅显示 `--cached` 标记的视图 | API Server 恢复后强制 list/watch resync |

\[DSL] 层保证云原生控制平面在脑裂/分区下仍能维持 Agent 容器运行（数据面继续服务），但拒绝写入操作以避免 Epoch 倒流。

### 6.5 v1.0.1 Capability Folding 在云原生场景的应用

v1.0.1 Capability Folding（ADR-014 seL4 唯一来源，参见 `docs/architecture/adr/ADR-014.md`）在云原生场景下的核心约束：

- **`agent_caps[1024]` 静态数组（16KB）**：每个节点上由 sec\_d 作为唯一写者，云原生 CRD controller 通过 `airy_sys_call(0)` 提交 capability 编译请求，sec\_d 串行化所有 Badge 编译以保证 Epoch 单调。
- **64-bit Badge 布局**（`Epoch<<48 | RandomTag<<16 | Perms`）：跨节点 IPC 时 gateway\_d 验证对端 Badge 的 Epoch 是否在本节点 gossip 窗口内（100ms 偏差容忍），超出窗口则触发 §6.4 \[DSL] 降级。
- **O(1) 撤销**（`atomic_inc(&airy_cap_global_epoch)`）：CRD 删除事件触发秒级 Epoch 推进，全集群 100ms 内完成 Badge 失效；CNI 网络策略联动纯 C LSM（airy_lsm）`uring_cmd` 单参数钩子（`struct io_uring_cmd *ioucmd`）即时阻断已撤销 capability 的 io\_uring 提交。
- **fastpath C-S9 内联校验（~10ns）+ slowpath LSM 钩子**：gateway\_d 跨节点 RPC 入口先走 fastpath C-S9 内联校验（~10ns），未命中或 Epoch 偏差时退化为 slowpath 走 airy\_lsm（`LSM_ORDER_MUTABLE`，非 `LSM_ORDER_FIRST`）钩子做完整 Cupolas blob 验证。
- **io\_uring SQE128 模式**：cmd 扩展至 80 字节（16→80），通过 `io_uring_cmd_to_pdu(cmd, pdu_type)` 安全宏访问 `pdu[32]` 字段；完成路径调用 `io_uring_cmd_done(cmd, ret, res2, issue_flags)` 4 参数签名。

### 6.6 gateway\_d 跨节点 Badge 一致性职责

gateway\_d 是 12 daemon 之一（完整名单见 §6.7），负责跨节点 IPC，必须维护跨节点 Badge 一致性。SSoT 见 `110-security/03-capability-model.md` §13.6。

| 版本范围 | 跨节点 Badge 一致性策略 |
| ----- | ----------------- |
| 0.1.1 | 单节点部署，无跨节点一致性需求；`agent_caps[1024]` 仅本地生效，gossip 模块以 stub 形式占位 |
| 1.0.1 | gateway\_d 通过 gRPC over QUIC + mTLS + X.509 实现跨节点 IPC；gossip 100ms Epoch 同步；per-node Badge 重编译（本节点收到对端 Epoch 后由 sec\_d 重新编译本地 `agent_caps[]` 子集） |

**1.0.1 跨节点 Badge 一致性流程**：

1. 节点 A 的 sec\_d 编译本地 Badge（`Epoch<<48 | RandomTag<<16 | Perms`），写入 `agent_caps[1024]`。
2. gateway\_d 监听 `agent_caps[]` 变更，通过 gossip 协议广播 `(node_id, epoch, capability_bitmap_hash)` 给集群其他节点。
3. 节点 B 的 gateway\_d 收到 gossip 消息，校验 Epoch 单调性，触发本地 sec\_d 重编译受影响的 Badge 子集（per-node Badge 重编译）。
4. 跨节点 IPC 时，gateway\_d 在 fastpath C-S9 内联校验对端 Badge Epoch 是否在 100ms gossip 窗口内；超出窗口则触发 §6.4 \[DSL] 降级，退化为单节点本地校验。
5. Epoch 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 触发，gossip 通道 100ms 内全集群同步失效。

### 6.7 12 daemon 完整名单与云原生部署

agentrt-linux 12 daemon 完整名单（参考 `02-services.md` §1.2 SSoT）在云原生场景下的部署与协作：

| 序号 | daemon | 云原生部署形态 | 协作关系 |
| -- | ------ | ---------- | ------- |
| 1 | sec\_d | DaemonSet（每节点 1 副本） | 唯一 Badge 编译者，串行化 `agent_caps[1024]` 跨节点一致性 |
| 2 | cogn\_d | Deployment（按 LLM 推理负载水平扩展） | 调用 LLM，CRD cognition 字段引用 CoreLoopThree \[SC] |
| 3 | mem\_d | DaemonSet（每节点 1 副本） | MemoryRovol L1-L4 分级内存 + 容器快照/迁移 |
| 4 | gateway\_d | DaemonSet + Ingress 后端 | 跨节点 IPC + Badge 一致性（见 §6.6） |
| 5 | logger\_d | DaemonSet（每节点 1 副本） | journald + OpenTelemetry Logs 聚合 |
| 6 | macro\_d | Deployment（集群级 1 副本，主备） | 宏调度监督，与 K8s 自定义调度器协作 |
| 7 | audit\_d | DaemonSet（每节点 1 副本） | 审计 `AIRY_FAULT_AUDIT_TAMPER = 0x100B` 上报 |
| 8 | sched\_d | DaemonSet（每节点 1 副本） | 与 K8s 自定义调度器协作，遵守 sched\_tac 约束 \[SC] |
| 9 | dev\_d | DaemonSet（每节点 1 副本） | 设备管理，DPU/IPU 卸载协调 |
| 10 | net\_d | DaemonSet（每节点 1 副本） | 与 CNI 插件协作，eBPF 数据平面 |
| 11 | vfs\_d | DaemonSet（每节点 1 副本） | 容器 rootfs + OCI 镜像层挂载 |
| 12 | config\_d | Deployment（集群级，主备） | 统一配置管理（路径 `/etc/agentrt/`，YAML/TOML），与 §6.7 章节配置 SSoT 一致 |

### 6.8 跨态协作流

```mermaid
sequenceDiagram
    participant USER as 用户/Operator
    participant KCTL as agentctl [SS]
    participant K8S as K8s API Server [IND]
    participant CTRL as CRD Controller [SS]
    participant GW as gateway_d (OS 级) [SS]
    participant IPC as io_uring ring [SC]
    participant AGENT as Agent 容器 [IND]
    participant MEM as MemoryRovol [SC]

    USER->>KCTL: agentctl apply -f agent.yaml
    KCTL->>K8S: 创建 Agent CRD（cognition 引用 CoreLoopThree [SC]）
    K8S->>CTRL: 监听 CRD 变化
    CTRL->>GW: 启动 gateway_d（reconcile [SS]）
    GW->>IPC: io_uring 提交（128B msg_hdr [SC]）
    IPC->>AGENT: 零拷贝传递（registered buffer）
    AGENT->>MEM: 容器快照（MemoryRovol L1-L4 [SC]）
    AGENT-->>GW: 结果返回（trace_id [SC] 贯穿）
    GW-->>KCTL: 响应
```

### 6.9 组件架构图

```mermaid
graph TD
    subgraph SC["[SC] 共享契约层（10 头文件）"]
        IPC_HDR[include/uapi/linux/airymax/ipc.h<br/>magic ARE1 + 128B msg_hdr]
        SEC_HDR[include/uapi/linux/airymax/security_types.h<br/>capability 41 ID + agent_caps + Badge 64-bit]
        COG_HDR[include/uapi/linux/airymax/cognition_types.h<br/>CoreLoopThree 阶段枚举]
        SYS_HDR[include/uapi/linux/airymax/syscalls.h<br/>v1.1 24 槽位 syscall]
        UAPI_HDR[include/uapi/linux/airymax/uapi_compat.h<br/>__aligned(64) + SQE128 cmd[80]]
        LSM_HDR[include/uapi/linux/airymax/lsm_types.h<br/>airy_lsm + uring_cmd 钩子]
    end
    subgraph SS["[SS] 语义同源层"]
        GW_SEM[gateway 网关语义<br/>agentrt gateway → K8s Ingress + gateway_d]
        SDK_SEM[SDK 管理接口<br/>agentrt sdk → agentctl]
        LIFE_SEM[Agent 生命周期<br/>agentrt lifecycle → CRD reconcile]
        RES_SEM[资源声明<br/>agentrt resource → K8s spec]
    end
    subgraph IND["[IND] 独立层"]
        CRD[K8s CRD + Controller]
        SHIM[containerd shim v2]
        OCI[OCI 镜像规范]
        CNI[eBPF CNI + 服务网格]
        OTEL[OpenTelemetry]
        DPU[DPU/IPU 卸载]
        SNO[超节点 OS]
    end
    subgraph DSL["[DSL] 降级生存层"]
        GW_FALL[gateway_d 单节点 Badge 本地校验<br/>100ms 强制 Epoch 推进]
        CRD_FALL[CRD controller 本地 reconcile cache]
        SHIM_FALL[containerd shim 本地镜像缓存只读]
    end
    subgraph CAP["v1.0.1 Capability Folding（云原生应用）"]
        AGENT_CAPS[agent_caps[1024] 静态数组<br/>sec_d 唯一写者]
        BADGE[64-bit Badge<br/>Epoch&lt;&lt;48 | RandomTag&lt;&lt;16 | Perms]
        FASTPATH[fastpath C-S9 内联校验 ~10ns]
        SLOWPATH[slowpath airy_lsm 钩子<br/>LSM_ORDER_MUTABLE]
    end
    IPC_HDR --> GW_SEM
    SEC_HDR --> CNI
    COG_HDR --> CRD
    SYS_HDR --> AGENT_CAPS
    UAPI_HDR --> IPC_HDR
    LSM_HDR --> SLOWPATH
    GW_SEM --> CRD
    SDK_SEM --> CRD
    LIFE_SEM --> CRD
    RES_SEM --> CRD
    CRD --> SHIM
    SHIM --> OCI
    CNI --> OTEL
    DPU --> SNO
    AGENT_CAPS --> BADGE
    BADGE --> FASTPATH
    FASTPATH --> SLOWPATH
    GW_SEM -. 降级 .-> GW_FALL
    CRD -. 降级 .-> CRD_FALL
    SHIM -. 降级 .-> SHIM_FALL
```

***

## 7. agentrt-linux 工程基线

- **agentrt-linux 云原生治理组**：云原生最佳实践。
- **agentrt-linux 超节点 OS**：超节点 OS 设计基线。
- **agentrt-linux K8s 发行版**：K8s 集成基线。
- **agentrt-linux 容器运行时**：containerd 集成基线。

***

## 8. 前沿理论参考

| 理论                 | 来源         | 应用        |
| ------------------ | ---------- | --------- |
| Kubernetes 原生      | CNCF       | K8s 原生集成  |
| 服务网格               | CNCF       | eBPF 服务网格 |
| OpenTelemetry      | CNCF       | 统一可观测性    |
| DPU/IPU            | 厂商         | 硬件卸载      |
| OCI                | OCI 组织     | 镜像规范      |
| containerd shim v2 | containerd | Agent 容器化 |
| CNI                | CNCF       | 网络插件      |
| CRD/Operator       | CNCF       | 声明式编排     |

***

## 9. 与其他子仓的协作

| 协作子仓          | 协作内容                           | 同源标注        |
| ------------- | ------------------------------ | ----------- |
| `kernel`      | 提供 eBPF、io\_uring 等云原生所需内核特性   | \[IND]      |
| `services`    | gateway\_d 提供网关，IPC 消息头共享      | \[SS]/\[SC] |
| `security`    | 提供容器沙箱、网络策略（capability 41 ID）  | \[SC]       |
| `memory`      | 提供容器快照、迁移（MemoryRovol L1-L4）   | \[SC]       |
| `cognition`   | Agent 容器化运行、CRD cognition 字段引用 | \[SC]       |
| `system`      | 提供云原生管理工具                      | \[IND]      |
| `tests-linux` | 云原生测试、混沌工程                     | \[IND]      |

***

## 10. 里程碑（M0-M8）

| 阶段 | 目标                            | 时间      |
| -- | ----------------------------- | ------- |
| M0 | 文档体系完成（本模块设计文档）               | 2026-07 |
| M1 | Agent CRD + 控制器               | 2026 Q3 |
| M2 | containerd shim（容器模式）         | 2026 Q4 |
| M3 | agentctl 命令行工具                | 2027 Q1 |
| M4 | OCI 镜像规范 + Wasm 镜像            | 2027 Q2 |
| M5 | CNI 插件 + 服务网格                 | 2027 Q3 |
| M6 | OpenTelemetry + DPU/IPU + 超节点 | 2027 Q4 |
| M7 | 云原生监控 + 日志                    | 2028 Q1 |
| M8 | 生产就绪 + 多集群                    | 2028 Q2 |

### 10.1 0.1.1 版本范围

仅完成 M0（文档体系完成）+ M1（\[IND] 共享契约层头文件占位）。不含内核/OS 代码实施。

### 10.2 1.0.1 版本范围

完成 M2-M8 全部里程碑，并实施云原生工程标准。

***

## 11. agentrt 一致性检查

| 检查项            | 验证内容                                                              | 结果   |
| -------------- | ----------------------------------------------------------------- | ---- |
| 命名一致性          | 核心表述使用 `agentrt-linux（AirymaxOS）` 全角括号配对                          | PASS |
| 语义同源标注         | gateway/sdk/Agent 生命周期/资源声明等标注 \[SS]                              | PASS |
| IRON-9 v3 四层合规 | \[SC] 10 头文件 + \[SS] 10 API + \[IND] 12 项独立实现 + \[DSL] 4 降级场景 + v1.0.1 Capability Folding 跨节点一致性 + 12 daemon 完整部署 | PASS |
| \[SC] 头文件引用    | ipc.h + security\_types.h + cognition\_types.h + syscalls.h + sched.h + memory\_types.h + error.h + log\_types.h + uapi\_compat.h + lsm\_types.h 共 10 个均在 §6.1 引用 | PASS |
| v1.0.1 Capability Folding | `agent_caps[1024]` + 64-bit Badge 布局 + O(1) 撤销 + fastpath C-S9 + slowpath airy\_lsm（`LSM_ORDER_MUTABLE`）均在 §6.5 引用 | PASS |
| 跨节点 Badge 一致性 | gateway\_d gRPC over QUIC + mTLS + X.509 + gossip 100ms + per-node Badge 重编译 + §6.6 SSoT 引用 | PASS |
| 12 daemon 完整性 | sec\_d/cogn\_d/mem\_d/gateway\_d/logger\_d/macro\_d/audit\_d/sched\_d/dev\_d/net\_d/vfs\_d/config\_d 共 12 个均在 §6.7 列出 | PASS |
| ADR-014 引用 | §2 与 §6.5 引用 ADR-014（seL4 唯一来源） | PASS |
| OLK 6.6 工程规范 | `__aligned(64)` + `io_uring_cmd_to_pdu` + `io_uring_cmd_done` 4 参数 + `security_uring_cmd` 单参数钩子 + SQE128 cmd[80] + `CONFIG_SECURITY_AIRY` default 'n' | PASS |
| 不移植特性声明        | 无 KABI\_RESERVE/BPF\_SCHED/KMSAN/etmem/dynamic\_pool/numa\_remote | PASS |
| 横切关注点声明        | §1.1 声明云原生贯穿 4 大数据流                                               | PASS |
| Mermaid 图      | §6.8 sequenceDiagram + §6.9 graph TD（≥2，含 \[DSL] 降级与 Capability Folding 子图） | PASS |
| 行数范围           | 修复后行数在 700-900 范围内                                                | PASS |
| 禁词检查           | 无'中枢'等禁词                                                          | PASS |

***

## 12. 相关文档

- [01-kernel.md](01-kernel.md)——内核模块（\[SC] sched.h + ipc.h + syscalls.h）
- [02-services.md](02-services.md)——服务模块（\[SC] ipc.h，gateway\_d 同源，12 daemon SSoT）
- [03-security.md](03-security.md)——安全模块（\[SC] security\_types.h + lsm\_types.h，容器沙箱 capability + v1.0.1 Capability Folding）
- [04-memory.md](04-memory.md)——记忆模块（\[SC] memory\_types.h，容器快照/迁移 MemoryRovol L1-L4）
- [05-cognition.md](05-cognition.md)——认知模块（\[SC] cognition\_types.h，CRD cognition 引用）
- [07-system.md](07-system.md)——系统模块（config_d 配置 SSoT `/etc/agentrt/`）
- [50-engineering-standards/README.md](../50-engineering-standards/README.md)——\[SC] 共享契约层 10 头文件清单
- [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md)——\[SC] Layout C 物理宿主
- [110-security/03-capability-model.md](../../110-security/03-capability-model.md)——capability 模型 SSoT（§13.6 跨节点 Badge 一致性）
- [docs/architecture/adr/ADR-012.md](../../architecture/adr/ADR-012.md)——ADR-012 微内核化技术路线
- [docs/architecture/adr/ADR-014.md](../../architecture/adr/ADR-014.md)——ADR-014 seL4 唯一来源

***

## 13. 参考

- Kubernetes 官方文档
- containerd shim v2 文档
- OCI 规范（image-spec、runtime-spec、distribution-spec）
- CNI 规范
- OpenTelemetry 官方文档
- NVIDIA BlueField 文档
- Intel IPU 文档
- agentrt-linux 云原生治理组文档
- agentrt-linux 超节点 OS 文档
- agentrt gateway + sdk 设计文档

