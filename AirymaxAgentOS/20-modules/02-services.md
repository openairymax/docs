# agentrt-liunx（AirymaxOS）服务设计文档（airymaxos-services，极境服务）

> 子仓编号：02
> 子仓代号：极境服务（Airymax Services）
> 文档版本：v1.0（2026-07-06）
> 设计基准：用户态系统服务 + 消息传递通信
> 同源 agentrt：daemons（12 daemons）

---

## 1. 子仓职责

`airymaxos-services` 是 agentrt-liunx（AirymaxOS）的用户态系统服务子仓，承担以下核心职责：

1. **用户态 VFS**：文件系统实现下放至用户态服务，内核仅保留虚拟文件系统层。
2. **用户态网络栈**：基于 DPDK/AF_XDP 实现高性能用户态网络协议栈。
3. **用户态驱动框架**：基于 VFIO/libvfio 将设备驱动下放至用户态进程。
4. **12 daemons 集成**：将 agentrt 的 12 个核心守护进程注册为 OS 级守护服务，与 systemd 集成。
5. **消息传递通信**：基于 io_uring 构建零拷贝、零 syscall 的消息传递基础设施。

---

## 2. 同源关系

| 维度 | agentrt（daemons） | agentrt-liunx（airymaxos-services） |
|------|-------------------|--------------------------------|
| 服务数量 | 12 daemons | 12 daemons + VFS/Net/Drivers |
| 通信方式 | 进程间消息队列 | io_uring 零拷贝 IPC |
| 服务管理 | 自研 supervisor | systemd 集成（agentrt-liunx 标准） |
| 部署形态 | 用户态进程 | systemd unit + capability |

**12 daemons 清单**（与 agentrt 完全同源）：
1. `gateway_d`：网关守护进程
2. `llm_d`：LLM 推理守护进程
3. `tool_d`：工具守护进程
4. `sched_d`：调度守护进程
5. `market_d`：市场守护进程
6. `monit_d`：监控守护进程
7. `channel_d`：通道守护进程
8. `info_d`：信息守护进程
9. `notify_d`：通知守护进程
10. `observe_d`：观测守护进程
11. `hook_d`：钩子守护进程
12. `plugin_d`：插件守护进程

---

## 3. 目录结构

```
airymaxos-services/
├── vfs/                   # 用户态 VFS
├── net/                   # 用户态网络栈（DPDK/AF_XDP）
├── drivers/               # 用户态驱动框架（VFIO/libvfio）
├── daemons/               # 12 daemons 集成（agentrt 同源）
│   ├── gateway_d/
│   ├── llm_d/
│   ├── tool_d/
│   ├── sched_d/
│   ├── market_d/
│   ├── monit_d/
│   ├── channel_d/
│   ├── info_d/
│   ├── notify_d/
│   ├── observe_d/
│   ├── hook_d/
│   └── plugin_d/
├── systemd/               # systemd 集成（agentrt-liunx 标准）
├── ipc/                   # 消息传递通信（基于 io_uring）
└── docs/
```

### 3.1 vfs/（用户态 VFS）

参考 **Fuchsia fservices** 架构：
- `vfs-service`：虚拟文件系统服务，处理路径解析、挂载管理。
- `fs-providers/`：具体文件系统实现（ext4、xfs、tmpfs、btrfs 等）作为独立服务。
- `fuchsia-channel`：基于 zxio channel 的文件操作协议。
- `vnode`：虚拟节点抽象，跨服务引用。

### 3.2 net/（用户态网络栈）

- `dpdk/`：基于 DPDK 的高性能用户态网络栈。
- `af_xdp/`：基于 AF_XDP 的零拷贝网络路径。
- `tcp-stack/`：用户态 TCP/IP 协议栈（参考 Fuchsia netstack3）。
- `dns/`：用户态 DNS 解析器。
- `dhcp/`：用户态 DHCP 客户端。

### 3.3 drivers/（用户态驱动框架）

基于 **VFIO/libvfio**：
- `libvfio/`：用户态驱动框架库。
- `pci-drivers/`：PCI 设备用户态驱动。
- `usb-drivers/`：USB 设备用户态驱动。
- `gpu-drivers/`：GPU 用户态驱动（与 `airymaxos-cognition/gpu-npu` 协作）。
- `block-drivers/`：块设备用户态驱动。

### 3.4 daemons/（12 daemons）

每个 daemon 均注册为 systemd 服务（`.service` unit），具备：
- capability 令牌（与 `airymaxos-security/capability` 协作）。
- io_uring IPC 通道（与其他 daemon 通信）。
- 资源限制（cgroup v2）。
- 日志输出（journald 集成）。

### 3.5 systemd/（systemd 集成）

遵循 **agentrt-liunx 基础系统治理组** 标准：
- `units/`：systemd unit 文件目录。
- `targets/`：systemd target 定义（airymaxos.target 等）。
- `generators/`：systemd generator（动态生成 unit）。
- `analyze/`：systemd-analyze 性能分析配置。

### 3.6 ipc/（消息传递通信）

基于 **io_uring** 零拷贝 IPC：
- `io-uring-ipc`：核心 IPC 库。
- `channel`：双向通道抽象（参考 Zircon channel）。
- `socket`：面向连接的 socket 抽象（参考 Zircon socket）。
- `fifo`：单向 FIFO 抽象（参考 Zircon fifo）。
- `eventpair`：事件同步原语（参考 Zircon eventpair）。

---

## 4. 核心特性

### 4.1 VFS 用户态化（参考 Fuchsia fservices）

**架构**：
- 内核保留虚拟文件系统层（VFS layer），仅负责路径解析、inode 缓存。
- 具体文件系统实现（ext4、xfs、tmpfs）作为独立用户态服务运行。
- 文件操作通过 io_uring channel 在 VFS 服务与 FS 服务之间传递。

**优势**：
- 文件系统 bug 不会导致内核 panic。
- 可独立重启文件系统服务。
- 支持热加载新文件系统。

### 4.2 网络栈用户态化（DPDK/AF_XDP）

**架构**：
- DPDK 模式：完全绕过内核，直接在用户态轮询网卡。
- AF_XDP 模式：通过 XDP 程序将数据包重定向至用户态 socket。
- 用户态 TCP/IP 协议栈（参考 Fuchsia netstack3）。

**优势**：
- 高性能：单核可处理 100+ Gbps 流量。
- 可演进：协议栈可独立升级。
- 隔离性：协议栈漏洞不影响内核。

### 4.3 驱动框架用户态化（VFIO/libvfio）

**架构**：
- 通过 VFIO 框架将 PCIe 设备 DMA 直通至用户态进程。
- 用户态驱动通过映射设备 MMIO 寄存器与 DMA 缓冲区操作设备。
- 中断处理通过 eventfd 传递至用户态。

**优势**：
- 驱动 bug 不影响内核。
- 可用 Rust 编写安全驱动（与 `airymaxos-kernel/patches/rust-drivers` 协作）。
- 可独立重启驱动服务。

### 4.4 12 daemons 注册为 OS 级守护服务

将 agentrt 的 12 个核心守护进程注册为 systemd 服务，获得：
- 生命周期管理（启动、停止、重启）。
- 依赖关系管理（After/Requires）。
- 资源限制（cgroup v2）。
- 日志聚合（journald）。
- 崩溃自动重启（Restart=always）。

**systemd unit 示例**（gateway_d）：
```ini
[Unit]
Description=agentrt-liunx Gateway Daemon
After=network.target airymaxos-ipc.target
Requires=airymaxos-ipc.target

[Service]
Type=notify
ExecStart=/usr/lib/airymaxos/daemons/gateway_d
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_DAC_OVERRIDE
Restart=always
RestartSec=1s
Slice=agent.slice

[Install]
WantedBy=airymaxos.target
```

### 4.5 消息传递通信（基于 io_uring 零拷贝）

**通信原语**（参考 Zircon）：
| 原语 | 语义 | 用途 |
|------|------|------|
| Channel | 双向、面向消息 | RPC 调用 |
| Socket | 双向、面向流 | 流式数据传输 |
| FIFO | 单向、面向消息 | 高吞吐单向通信 |
| Eventpair | 事件同步 | 跨进程信号量 |

**零拷贝路径**：
- 发送方注册 page 为 io_uring buffer。
- 接收方通过 `IORING_OP_IPC_RECV` 接收 page 引用。
- 内核仅翻转 page table entry，不拷贝数据。

---

## 5. 微内核思想体现

### 5.1 服务用户态化

遵循 **Minix3** 哲学：
- 所有系统服务（VFS、网络、驱动）均运行在用户态。
- 服务通过消息传递通信。
- 服务崩溃可被监督进程重启。

### 5.2 消息传递通信

遵循 **Zircon message passing** 模型：
- 进程间通信通过内核仲裁的消息传递。
- 消息携带 capability 令牌。
- 零拷贝通过 page flipping 实现。

### 5.3 服务隔离

每个服务运行在独立地址空间：
- 进程级隔离（MMU）。
- capability 限制（最小权限）。
- seccomp 过滤（系统调用白名单）。
- Landlock 沙箱（文件访问控制）。

---

## 6. agentrt-liunx 工程基线

- **agentrt-liunx 基础系统治理组**：systemd、基础系统服务最佳实践。
- **agentrt-liunx 网络子系统**：用户态网络栈基线。
- **agentrt-liunx 驱动框架**：用户态驱动框架基线。
- **agentrt-liunx 服务管理**：systemd unit 规范。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| Zircon message passing | Google Zircon | io_uring IPC 通信原语 |
| Minix3 user-mode services | Minix3 | 服务用户态化哲学 |
| Fuchsia fservices | Google Fuchsia | 用户态 VFS 架构 |
| Fuchsia netstack3 | Google Fuchsia | 用户态网络栈 |
| DPDK | Intel | 用户态网络数据路径 |
| AF_XDP | Linux | 零拷贝网络 |
| VFIO/libvfio | Linux | 用户态驱动框架 |
| io_uring zero-copy | Linux 5.x+ | 零拷贝 IPC |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-kernel` | 提供 VFS/网络/驱动用户态服务所需的内核侧接口 |
| `airymaxos-security` | 为每个服务颁发 capability 令牌 |
| `airymaxos-memory` | 提供 VFS 持久化层、MemoryRovol 卷载 |
| `airymaxos-cognition` | llm_d/sched_d 与认知循环协作 |
| `airymaxos-cloudnative` | gateway_d/sdk 与 K8s 集成 |
| `airymaxos-system` | systemd unit 管理、配置工具 |
| `airymaxos-tests` | 服务集成测试、Soak Test |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | io_uring IPC 库 + 通信原语 | 2026 Q3 |
| M2 | 12 daemons systemd 集成 | 2026 Q4 |
| M3 | 用户态 VFS（Phase 1） | 2027 Q1 |
| M4 | 用户态网络栈（AF_XDP） | 2027 Q2 |
| M5 | 用户态驱动框架（VFIO） | 2027 Q3 |
| M6 | 全部服务用户态化完成 | 2027 Q4 |

---

## 10. 参考

- Fuchsia fservices 设计文档
- Zircon 内核 IPC 设计
- Minix3 用户态服务架构
- DPDK 项目文档
- AF_XDP 教程
- VFIO/libvfio 项目文档
- agentrt-liunx 基础系统治理组文档
- systemd 官方文档
- agentrt daemons 设计文档
