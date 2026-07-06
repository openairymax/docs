# AirymaxOS 云原生设计文档（airymaxos-cloudnative，极境云原生）

> 子仓编号：06
> 子仓代号：极境云原生（Airymax Cloud Native）
> 文档版本：v1.0（2026-07-06）
> 设计基准：K8s + containerd + OCI + agentctl
> 同源 agentrt：gateway + sdk

---

## 1. 子仓职责

`airymaxos-cloudnative` 是 AirymaxOS 的云原生基础设施子仓，承担以下核心职责：

1. **Kubernetes 集成**：将 Agent 作为 Kubernetes CRD（Custom Resource Definition）原生集成。
2. **containerd shim**：提供 containerd shim，实现 Agent 容器化运行。
3. **OCI 镜像规范**：遵循 OCI（Open Container Initiative）镜像规范。
4. **CNI 插件**：提供 CNI 插件，实现服务网格。
5. **agentctl**：对标 kubectl 的 Agent 管理命令行工具。
6. **OpenTelemetry 可观测性**：集成 OpenTelemetry 提供统一可观测性。
7. **DPU/IPU 卸载支持**：支持 DPU/IPU 硬件卸载。
8. **超节点 OS**：基于 AirymaxOS 超节点 OS 实现云原生超节点。

---

## 2. 同源关系

| 维度 | agentrt（gateway + sdk） | AirymaxOS（airymaxos-cloudnative） |
|------|------------------------|----------------------------------|
| 网关 | gateway（应用层） | K8s Ingress + gateway_d（OS 级） |
| SDK | sdk（应用层） | agentctl（对标 kubectl） |
| 部署 | 应用部署 | K8s 原生部署（CRD） |
| 容器化 | 进程隔离 | containerd shim（OCI） |
| 可观测性 | 自研监控 | OpenTelemetry 标准化 |

**同源传承要点**：
- 保留 agentrt gateway 的"网关"语义（升级为 K8s Ingress + gateway_d）。
- 保留 sdk 的"管理接口"语义（升级为 agentctl）。
- 升级为 K8s 原生部署，获得声明式编排能力。

---

## 3. 目录结构

```
airymaxos-cloudnative/
├── kubernetes/             # K8s 集成（Agent 作为 CRD）
├── containerd-shim/        # containerd shim（Agent 容器化）
├── oci/                    # OCI 镜像规范
├── cni/                    # CNI 插件（服务网格）
├── agentctl/              # agentctl（对标 kubectl）
├── observability/          # OpenTelemetry 可观测性
├── dpu-ipu/                # DPU/IPU 卸载支持
├── super-node-os/          # 超节点 OS（AirymaxOS 自研）
└── docs/
```

### 3.1 kubernetes/（K8s 集成）

- `crds/`：Agent CRD 定义（Agent、AgentRuntime、AgentPipeline）。
- `controllers/`：控制器（Controller Runtime）。
- `operators/`：Operator 实现。
- `schedulers/`：K8s 自定义调度器（与 `airymaxos-cognition/llm-scheduler` 协作）。
- `webhooks/`：准入 webhook。

### 3.2 containerd-shim/（containerd shim）

- `airymax-shim`：Agent 容器 shim（基于 containerd shim v2 API）。
- `runtime`：runtime spec 处理。
- `image-pull`：镜像拉取（与 OCI 协作）。
- `snapshot`：快照支持（与 `airymaxos-memory` 协作）。

### 3.3 oci/（OCI 镜像规范）

遵循 **OCI（Open Container Initiative）** 规范：
- `image-spec`：镜像规范实现。
- `runtime-spec`：运行时规范实现。
- `distribution-spec`：分发规范实现。
- `agent-image`：Agent 镜像构建工具。

### 3.4 cni/（CNI 插件）

- `airymax-cni`：AirymaxOS CNI 插件。
- `service-mesh`：服务网格（基于 eBPF）。
- `network-policy`：网络策略（与 `airymaxos-security` 协作）。
- `multus`：Multus 多 CNI 支持。

### 3.5 agentctl/（对标 kubectl）

- `cmd/`：命令行入口。
- `api-client`：API 客户端（与 K8s API 协作）。
- `plugins/`：插件机制（kubectl plugin 兼容）。
- `completion`：命令补全。

### 3.6 observability/（OpenTelemetry 可观测性）

集成 **OpenTelemetry** 标准：
- `tracing`：分布式追踪。
- `metrics`：指标采集。
- `logs`：日志聚合。
- `exporters`：导出器（Prometheus、Jaeger、Loki）。
- `ebpf-collector`：eBPF 可观测性采集器。

### 3.7 dpu-ipu/（DPU/IPU 卸载支持）

- `dpu-offload`：DPU 卸载框架（NVIDIA BlueField、Intel IPU）。
- `network-offload`：网络卸载（OVS offload）。
- `storage-offload`：存储卸载（NVMe-oF offload）。
- `security-offload`：安全卸载（IPSec/TLS offload）。

### 3.8 super-node-os/（超节点 OS）

基于 **AirymaxOS 超节点 OS**：
- `topology`：超节点拓扑管理。
- `scheduling`：超节点调度（NUMA 感知）。
- `migration`：跨节点迁移（与 MemoryRovol 协作）。
- `snapshot`：超节点快照。

---

## 4. 核心特性

### 4.1 K8s 集成（Agent 作为 CRD）

**Agent CRD**：
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
      enabled: true
    thinkdual:
      enabled: true
  resources:
    gpu: 1
    npu: 0
    memory: 4Gi
  replicas: 3
```

**控制器**：监听 Agent CRD 变化，创建/更新/删除 Agent 实例。

### 4.2 containerd shim（Agent 容器化）

**shim 架构**：
- 基于 containerd shim v2 API 实现。
- 支持 Wasm runtime（与 `airymaxos-cognition/wasm-runtime` 协作）。
- 支持传统容器（runc）。
- 支持 Agent 进程模式。

**优势**：
- 与 K8s 生态无缝集成。
- 复用 containerd 镜像分发。
- 支持快照与迁移（与 `airymaxos-memory` 协作）。

### 4.3 OCI 镜像规范

遵循 **OCI 规范**：
- image-spec：镜像格式规范。
- runtime-spec：运行时规范。
- distribution-spec：分发规范。

**Agent 镜像**：
- 基于 OCI 镜像格式。
- 支持 Wasm 模块作为镜像层。
- 支持多架构（amd64、arm64）。

### 4.4 CNI 插件（服务网格）

**airymax-cni**：
- 基于 eBPF 的高性能 CNI 插件。
- 服务网格数据平面（替代 Istio sidecar）。
- 网络策略（与 `airymaxos-security` 协作）。

**优势**：
- 无 sidecar，减少资源开销。
- eBPF 数据路径，高性能。
- 声明式网络策略。

### 4.5 agentctl（对标 kubectl）

**命令示例**：
```bash
# 创建 Agent
agentctl apply -f agent.yaml

# 查看 Agent 状态
agentctl get agents

# 查看 Agent 日志
agentctl logs my-agent

# 进入 Agent shell
agentctl exec my-agent -- wasm-shell

# 扩缩容
agentctl scale agent my-agent --replicas=5

# 快照
agentctl snapshot my-agent --output snapshot.tar
```

### 4.6 OpenTelemetry 可观测性

**统一可观测性**：
- Tracing：分布式追踪（OpenTelemetry Tracing）。
- Metrics：指标采集（OpenTelemetry Metrics）。
- Logs：日志聚合（OpenTelemetry Logs）。

**eBPF 采集器**：
- 基于 eBPF 的零侵入采集。
- 系统调用追踪。
- 网络包追踪。
- 性能分析。

### 4.7 DPU/IPU 卸载支持

**支持的 DPU/IPU**：
| 厂商 | 产品 | 类型 |
|------|------|------|
| NVIDIA | BlueField-3 | DPU |
| Intel | IPU E2100 | IPU |
| AMD | Pensando DPU | DPU |
| ARM | Neoverse | NPU |

**卸载场景**：
- 网络卸载：OVS、VxLAN、负载均衡。
- 存储卸载：NVMe-oF、压缩、加密。
- 安全卸载：IPSec、TLS、防火墙。
- 虚拟化卸载：virtio、SR-IOV。

### 4.8 超节点 OS（AirymaxOS 自研）

基于 **AirymaxOS 超节点 OS**：
- 超节点拓扑：多 die/多 chip 统一管理。
- NUMA 感知调度：优先本地 die 调度。
- 跨 die 迁移：基于 MemoryRovol 跨 die 迁移。
- 超节点快照：整机快照与恢复。

---

## 5. 微内核思想体现

### 5.1 云原生作为独立模块

遵循微内核"模块化"原则：
- 云原生功能作为独立子仓。
- 与内核解耦，可独立演进。
- 通过标准接口（CRD、CNI、OCI）与其他子仓协作。

### 5.2 声明式接口

遵循 K8s 声明式范式：
- Agent 状态声明（CRD）。
- 控制器 reconcile 期望状态。
- 符合"机制与策略分离"原则。

### 5.3 标准化接口

- OCI 镜像规范：标准化镜像格式。
- CNI 网络接口：标准化网络插件。
- OpenTelemetry 可观测性：标准化观测协议。

---

## 6. AirymaxOS 工程基线

- **AirymaxOS 云原生治理组**：云原生最佳实践。
- **AirymaxOS 超节点 OS**：超节点 OS 设计基线。
- **AirymaxOS K8s 发行版**：K8s 集成基线。
- **AirymaxOS 容器运行时**：containerd 集成基线。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| Kubernetes 原生 | CNCF | K8s 原生集成 |
| 服务网格 | CNCF | eBPF 服务网格 |
| OpenTelemetry | CNCF | 统一可观测性 |
| DPU/IPU | 厂商 | 硬件卸载 |
| OCI | OCI 组织 | 镜像规范 |
| containerd shim v2 | containerd | Agent 容器化 |
| CNI | CNCF | 网络插件 |
| CRD/Operator | CNCF | 声明式编排 |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-kernel` | 提供 eBPF、io_uring 等云原生所需内核特性 |
| `airymaxos-services` | gateway_d 提供网关，sdk 提供 SDK |
| `airymaxos-security` | 提供容器沙箱、网络策略 |
| `airymaxos-memory` | 提供容器快照、迁移 |
| `airymaxos-cognition` | Agent 容器化运行、超节点沙箱 |
| `airymaxos-system` | 提供云原生管理工具 |
| `airymaxos-tests` | 云原生测试、混沌工程 |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | Agent CRD + 控制器 | 2026 Q3 |
| M2 | containerd shim（容器模式） | 2026 Q4 |
| M3 | agentctl 命令行工具 | 2027 Q1 |
| M4 | OCI 镜像规范 + Wasm 镜像 | 2027 Q2 |
| M5 | CNI 插件 + 服务网格 | 2027 Q3 |
| M6 | OpenTelemetry + DPU/IPU + 超节点 | 2027 Q4 |

---

## 10. 参考

- Kubernetes 官方文档
- containerd shim v2 文档
- OCI 规范（image-spec、runtime-spec、distribution-spec）
- CNI 规范
- OpenTelemetry 官方文档
- NVIDIA BlueField 文档
- Intel IPU 文档
- AirymaxOS 云原生治理组文档
- AirymaxOS 超节点 OS 文档
- agentrt gateway + sdk 设计文档
