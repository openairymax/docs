Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# AirymaxOS 云原生 Agent 部署设计

> **文档定位**: AirymaxOS（agentrt-linux，极境智能体操作系统）云原生工程体系主索引
> **版本**: 0.1.1（占位）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **优先级**: P1（0.1.1 仅创建 README 占位，1.0.1 完成 5 文档）
> **同源映射**: agentrt gateway + Linux 6.6 容器与编排（containerd / K8s / OCI）
> **理论根基**: 云原生计算哲学 + Airymax S-4 涌现性管理 + K-3 服务隔离

---

## 1. 模块定位

AirymaxOS 云原生 Agent 部署体系是智能体操作系统在云原生场景下的核心工程保障。它继承 Linux 发行版与云原生社区 15+ 年沉淀的容器编排哲学（OCI 标准 + containerd + Kubernetes + CNI + CSI），并在其上扩展智能体操作系统专属的 Agent 容器化、超节点 OS、Agent 即 CRD（Custom Resource Definition）等。

AirymaxOS 与生俱来具备云原生亲和力：MicroCoreRT 调度语义与 K8s 调度器对齐，AgentsIPC 协议可作为 K8s CRD 表达，MemoryRovol 记忆卷载可作为 CSI 卷挂载。Agent 应用在 AirymaxOS 上既可以传统进程形态运行，也可以容器化形态运行，两种形态共享同一套 SDK 与契约。

### 1.1 云原生分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 容器运行时 | containerd + OCI runtime | 容器隔离 |
| L2 | 容器网络 | CNI + Calico / Cilium | 容器网络 |
| L3 | 容器存储 | CSI + MemoryRovol 卷 | 容器存储 |
| L4 | 编排平台 | Kubernetes + CRD | 集群编排 |
| **L5** | **Agent 容器化** | **AirymaxOS 专属** | **Agent 镜像规范** |
| **L6** | **超节点 OS** | **AirymaxOS 专属** | **超节点容器编排** |
| **L7** | **Agent CRD** | **AirymaxOS 专属** | **Agent 即 K8s 资源** |

### 1.2 AirymaxOS 扩展

- **Agent 容器镜像规范**：基于 OCI 标准，扩展 Token 预算、记忆卷载声明
- **超节点 OS**：超节点（Supernode）场景下的容器编排，参考 Token 能效框架思想
- **Agent CRD**：Agent 应用作为 K8s Custom Resource，声明式管理
- **MemoryRovol CSI**：记忆卷载作为 CSI 卷，跨容器挂载

---

## 2. 核心云原生机制

### 2.1 containerd 集成

AirymaxOS 内置 containerd 作为容器运行时：

```yaml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri.containerd"]
    default_runtime_name = "runc"
    [plugins."io.containerd.grpc.v1.cri.containerd.runtimes.runc"]
      runtime_type = "io.containerd.runc.v2"
```

### 2.2 Kubernetes CRD

Agent 应用作为 K8s Custom Resource：

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
    scheduler: SCHED_AGENT
```

### 2.3 MemoryRovol CSI

记忆卷载作为 CSI 卷挂载到容器：

```yaml
apiVersion: v1
kind: PersistentVolume
spec:
  csi:
    driver: memoryrovol.airymaxos.dev
    volumeHandle: agent-01-memory
    volumeAttributes:
      layers: "L1,L2,L3,L4"
      capacity: "100Gi"
```

---

## 3. 文档索引

```
150-cloud-native/
├── README.md                       # 本文件
├── 01-container-runtime.md        # containerd 集成
├── 02-agent-image-spec.md         # Agent 容器镜像规范
├── 03-kubernetes-crd.md           # Agent CRD 设计
├── 04-supernode-os.md             # 超节点 OS
└── 05-memoryrovol-csi.md          # MemoryRovol CSI 驱动
```

### 3.1 0.1.1 版本范围

仅完成 README 占位（P1 优先级，0.1.1 不开发具体文档）。

### 3.2 1.0.1 版本范围

完成全部 5 文档，并实施云原生工程标准。

---

## 4. AirymaxOS 专属扩展

### 4.1 Agent 容器镜像规范

基于 OCI 标准，扩展以下标签：

| 标签 | 用途 |
|------|------|
| `airymaxos.agent.token-budget` | Token 预算声明 |
| `airymaxos.agent.memory-rovol` | 记忆卷载声明 |
| `airymaxos.agent.scheduler` | 调度类声明（SCHED_AGENT） |
| `airymaxos.agent.cognition-cycle` | 认知循环类型 |

### 4.2 超节点 OS

超节点（Supernode）场景下的容器编排：
- 多节点 Agent 协同调度
- 跨节点记忆卷载同步
- 超节点级 Token 预算管理

### 4.3 同源 agentrt gateway

agentrt 的 `gateway_d` daemon 与 AirymaxOS 云原生体系同源：
- agentrt 用户态：`gateway_d` 进程作为 Agent 网关
- AirymaxOS 云原生：gateway 作为 K8s Ingress Controller
- 两端通过同源 AgentsIPC 协议保持互操作

### 4.4 IRON-9 同源但独立

云原生体系遵循 IRON-9 原则：
- agentrt 用户态网关（gateway_d）
- AirymaxOS 云原生编排（K8s CRD + containerd）
- 两端独立演进，但通过同源 API 保持互操作

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **S-4 涌现性管理** | Agent 集群涌现行为管理 |
| **K-3 服务隔离** | 容器隔离 + cgroup v2 |
| **K-4 可插拔策略** | CNI / CSI 可插拔 |
| **E-2 可观测性** | 容器监控 + Agent 追踪 |
| **IRON-9 同源但独立** | 与 agentrt gateway 同源 |

---

## 6. 相关文档

- `20-modules/06-cloudnative.md`（cloudnative 子仓设计）
- `100-operations/README.md`（部署运维）
- `90-observability/README.md`（可观测性）
- `140-application-development/README.md`（Agent 应用开发）
- `170-performance/README.md`（性能工程）

---

## 7. 参考材料

- Linux 6.6 容器子系统（cgroup v2 / namespace）
- OCI 标准（Open Container Initiative）
- containerd 项目
- Kubernetes 项目
- CNI / CSI / CRI 规范

---

> **文档结束** | 0.1.1 P1 占位，1.0.1 完成 5 文档
