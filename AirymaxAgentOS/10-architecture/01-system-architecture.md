# agentrt-liunx（AirymaxOS）架构设计

> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **设计原则**: 微内核设计思想 + agentrt-liunx 工程基线 + Airymax 同源性

## 1. 设计哲学

agentrt-liunx 采用三大设计支柱:

### 1.1 微内核设计思想（参考 seL4 / Zircon / Minix3）

**核心原则**: 最小化特权态代码（Liedtke minimality principle）

| 原则 | 含义 | agentrt-liunx 落地 |
|---|---|---|
| 最小化内核 | 只保留调度、IPC、地址空间、内存管理 | airymaxos-kernel 仅保留必要内核服务 |
| 服务用户态化 | 文件系统、网络、驱动移到用户态 | airymaxos-services 承载用户态系统服务 |
| 消息传递通信 | 服务之间通过 IPC 通信 | 基于 io_uring 的高性能消息传递 |
| capability 安全 | 不可伪造令牌控制资源访问 | airymaxos-security 实现 capability 系统 |
| 隔离与模块化 | 每个服务独立运行，故障隔离 | 每个 Agent 独立地址空间 |

**参考来源**:
- seL4: 形式化验证的微内核，~10-12 kSLOC，capability-based
- Zircon: Google Fuchsia 的微内核，对象导向 + 消息传递
- Minix3: Tanenbaum 的用户态服务设计

### 1.2 agentrt-liunx 工程基线（Linux 6.6 内核基线 / 下一代内核基线）

**核心原则**: 采用 agentrt-liunx 自身的模块设计、技术规格、标准和规范

| 维度 | agentrt-liunx 工程基线 | agentrt-liunx 落地 |
|---|---|---|
| 内核 | Linux 6.6 内核基线 | airymaxos-kernel 基于 Linux 6.6 内核基线 |
| 包管理 | RPM + dnf | airymaxos-system 采用 RPM + dnf |
| 系统服务 | systemd | airymaxos-services 集成 systemd |
| 安全 | SELinux + 国密算法 | airymaxos-security 实现国密支持 |
| 测试 | agentrt-liunx 系统级测试套件 | airymaxos-tests 实现集成测试框架 |
| AI 原生 | 认知循环系统、超节点 OS | airymaxos-cognition + airymaxos-cloudnative |
| 架构支持 | x86, ARM, RISC-V, 鲲鹏, 飞腾 | airymaxos-kernel 多架构支持 |
| 社区治理 | SIG（Special Interest Group）| agentrt-liunx 按子仓划分 SIG |

### 1.3 Airymax 同源性（与 agentrt 同源）

**核心原则**: agentrt 和 agentrt-liunx 共享 Airymax 设计理念

| agentrt 模块 | agentrt-liunx 同源模块 | 同源语义 |
|---|---|---|
| atoms/corekern (MicroCoreRT) | airymaxos-kernel (SCHED_AGENT) | 调度语义一致 |
| atoms/ipc + protocols (AgentsIPC) | airymaxos-services (消息传递) | IPC 协议语义一致 |
| cupolas (Cupolas) | airymaxos-security (capability) | 安全模型一致 |
| heapstore + memoryrovol (MemoryRovol) | airymaxos-memory (记忆持久化) | 记忆模型一致 |
| coreloopthree + frameworks (CoreLoopThree) | airymaxos-cognition (kthread) | 认知模型一致 |
| daemons (12 daemons) | airymaxos-services (systemd 集成) | 服务模型一致 |
| gateway + sdk | airymaxos-cloudnative (K8s+OCI) | 网关语义一致 |
| commons | airymaxos-system (基础库) | 工具语义一致 |

## 2. 整体架构

### 2.1 架构分层图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent 应用层                                  │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐     │
│  │ 科研 Agent   │ 客服 Agent   │ 工业控制     │ 具身智能     │     │
│  └──────┬──────┴──────┬──────┴──────┬──────┴──────┬──────┘     │
│         │             │             │             │             │
│  ┌──────▼─────────────▼─────────────▼─────────────▼──────┐    │
│  │ agentrt SDK (Python/Go/Rust/TypeScript)                │    │
│  │ CognitionClient / SafetyClient / ToolClient / ChatClient│   │
│  └──────────────────────┬──────────────────────────────────┘    │
└─────────────────────────┼───────────────────────────────────────┘
                          │ 同源天然适配
┌─────────────────────────▼───────────────────────────────────────┐
│                 agentrt-liunx 用户态服务层 (airymaxos-services)     │
│  ┌─────────┬──────────┬──────────┬──────────┬─────────────┐   │
│  │ VFS     │ 网络栈   │ 驱动框架  │ 12 daemons│ systemd     │   │
│  │ 用户态  │ 用户态   │ 用户态    │ OS 级守护  │ 集成        │   │
│  └─────────┴──────────┴──────────┴──────────┴─────────────┘   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ airymaxos-cognition (认知运行时)                          │ │
│  │ CoreLoopThree kthread + Wasm runtime + LLM 调度           │ │
│  └───────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ airymaxos-cloudnative (云原生)                             │ │
│  │ K8s + containerd + OCI + CNI + agentctl                   │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────┬───────────────────────────────────────┘
                          │ 系统调用 (syscall)
┌─────────────────────────▼───────────────────────────────────────┐
│              agentrt-liunx 微内核核心 (airymaxos-kernel)             │
│  ┌──────────┬──────────┬──────────┬──────────┬────────────┐   │
│  │ sched_ext│ io_uring │ 内存管理  │ IPC      │ Rust 模块   │   │
│  │ SCHED_   │ 零拷贝   │ 基本能力  │ agent_ipc│ 安全驱动    │   │
│  │ AGENT    │          │          │ syscall  │             │   │
│  └──────────┴──────────┴──────────┴──────────┴────────────┘   │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                 Linux 内核 (Linux 6.6 内核基线)                  │
│  EEVDF 调度器 + XFS 在线 fsck + Btrfs + 网络协议栈 + 驱动          │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 微内核化改造策略

agentrt-liunx 不是从零开发微内核，而是基于 Linux 内核进行**微内核化改造**:

| 传统 Linux（宏内核） | agentrt-liunx（微内核化） |
|---|---|
| VFS 在内核态 | VFS 用户态化（airymaxos-services） |
| 网络栈在内核态 | 网络栈部分用户态化（DPDK/AF_XDP） |
| 驱动在内核态 | 部分驱动用户态化（VFIO/libvfio） |
| 安全模块在内核态 | capability + LSM 用户态化 |
| 调度器在内核态 | sched_ext 允许 eBPF 用户态调度 |

**关键改造**: 利用 agentrt-liunx 内核增强的 sched_ext + eBPF + io_uring 实现微内核化，而不抛弃 Linux 内核的稳定性和硬件支持。

## 3. 与 agentrt 的同源关系

### 3.1 同源性定义

**同源** = agentrt 和 agentrt-liunx 共享 Airymax 设计理念，并共享契约层代码（IRON-9 v2，`include/airymax/` 头文件库），实现层各自独立。

| 维度 | 同源体现 |
|---|---|
| 设计理念 | MicroCoreRT/AgentsIPC/Cupolas/MemoryRovol/CoreLoopThree 的语义一致 |
| 接口语义 | 128B 消息头、capability 模型、记忆卷载语义一致 |
| 能力体系 | agentrt 用户态提供的能力，agentrt-liunx 在 OS 层提供相同语义的能力 |
| 架构契合 | agentrt 的设计假设和 agentrt-liunx 的实现假设一致 |

### 3.2 agentrt 在 agentrt-liunx 上的运行模式

```
agentrt（跨平台用户态运行时，代码不变）
   │
   ├── 在普通 Linux 上: 标准 libc/POSIX，和其他 Linux 应用一样
   ├── 在 macOS 上: 标准 libc/POSIX
   ├── 在 Windows 上: Win32 API
   │
   └── 在 agentrt-liunx 上:
       ├── 作为普通用户态应用运行（标准 libc/POSIX）
       │   └── 天然更稳健 ← 因为同源，设计假设和实现假设一致
       │
       └── 可选使用 OS 原生能力（同源红利）
           例：agentrt 可选调用 SCHED_AGENT 调度类
               因为 SCHED_AGENT 的语义和 MicroCoreRT 一致（同源）
               所以调用是自然的，无适配层
```

### 3.3 同源红利示例

以 IPC 为例:

```
agentrt 的 AgentsIPC 定义 (are_ipc.h):
  - 128 字节定长消息头
  - magic: 0x41524531 ("ARE1")
  - 5 种 payload 协议 (JSON-RPC/MCP/A2A/OpenAI/Custom)

agentrt-liunx 的 IPC 子系统 (airymaxos-kernel + airymaxos-services):
  - 内核原生支持 128 字节消息头（同源）
  - magic 0x41524531 作为内核识别码（同源）
  - 5 种 payload 协议作为内核原生协议（同源）

→ agentrt 在 agentrt-liunx 上运行时，IPC 无需任何转换层，天然契合
```

## 4. 前沿理论应用

### 4.1 微内核理论（seL4 / Zircon）

| 理论 | 来源 | 应用 |
|---|---|---|
| Liedtke minimality principle | seL4 | airymaxos-kernel 最小化 |
| capability-based security | seL4 / Zircon | airymaxos-security |
| 形式化验证 | seL4 | airymaxos-tests |
| 消息传递 IPC | Zircon | airymaxos-services |
| 对象导向资源管理 | Zircon | airymaxos-kernel |

### 4.2 Linux 2026 最新特性

| 特性 | 来源 | 应用 |
|---|---|---|
| EEVDF 调度器 | Linux 6.6 内核基线 | airymaxos-kernel |
| Rust 实验性支持 | Linux 6.6 内核基线 | airymaxos-kernel（安全驱动）|
| sched_ext | agentrt-liunx 内核增强（主线 6.12+） | airymaxos-kernel（SCHED_AGENT）|
| io_uring 异步 I/O 改进 | Linux 6.6 | airymaxos-kernel + services |
| eBPF kfunc + dynamic pointer | Linux 6.6 原生 | airymaxos-security |
| MGLRU（多代 LRU） | Linux 6.6 | airymaxos-memory |
| XFS 在线 fsck | Linux 6.6 内核基线 | airymaxos-services |
| BPF + io_uring 集成 | Linux 6.6 | airymaxos-security |

### 4.3 agentrt-liunx 2026 AI 原生

| 特性 | 来源 | 应用 |
|---|---|---|
| 认知循环系统 | Linux 6.6 内核基线（SP3 增强） | airymaxos-cognition |
| 超节点 OS | Linux 6.6 内核基线（SP3 增强） | airymaxos-cloudnative |
| 超节点沙箱 | Linux 6.15+ 内核 | airymaxos-cognition |
| Token 能效优化 | agentrt-liunx Token 能效框架 2026 | airymaxos-cognition |
| 具身智能 Claw | Linux 6.15+ 内核 | airymaxos-cognition |
| DevStation | agentrt-liunx 工程基线 | airymaxos-system |

## 5. 架构决策记录

| 决策 | 内容 | 日期 |
|---|---|---|
| ADR-001 | agentrt-liunx 基于 Linux 内核，不从零开发微内核 | 2026-07-06 |
| ADR-002 | 采用微内核化改造策略，而非真正的微内核 | 2026-07-06 |
| ADR-003 | 全面采用 agentrt-liunx 工程基线 | 2026-07-06 |
| ADR-004 | agentrt 和 agentrt-liunx 同源且部分代码共享 | 2026-07-06 |
| ADR-005 | 8 子仓按能力域划分 | 2026-07-06 |
| ADR-006 | OS 文档与 RT 文档物理分开 | 2026-07-06 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
