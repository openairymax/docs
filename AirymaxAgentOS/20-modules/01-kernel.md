# AirymaxOS 内核设计文档（airymaxos-kernel，极境内核）

> 子仓编号：01
> 子仓代号：极境内核（Airymax Kernel）
> 文档版本：v1.0（2026-07-06）
> 设计基准：Linux 6.6 内核基线 + 微内核化改造
> 同源 agentrt：atoms/corekern（MicroCoreRT）

---

## 1. 子仓职责

`airymaxos-kernel` 是 AirymaxOS 的内核子仓，承担以下核心职责：

1. **Linux 6.6 内核维护**：基于 Linux 6.6 内核基线，保持与上游社区同步演进。
2. **微内核化改造**：在保留 Linux 6.6 完整能力的前提下，遵循 Liedtke minimality principle，将 VFS、网络栈、设备驱动等子系统逐步用户态化，最小化特权态代码体积。
3. **Agent 感知调度**：通过 `sched_ext`（AirymaxOS 内核增强，主线 6.12+ 引入并持续演进）实现 `SCHED_AGENT` 调度类，允许 eBPF 程序在用户态定义调度策略。
4. **高性能 IPC 基础**：基于 `io_uring`（2026 已成为默认高性能 I/O 路径）构建零 syscall、零拷贝的消息传递基础设施。
5. **Rust 安全驱动**：依托 Linux 6.6 中 Rust 实验性支持（持续演进中），构建安全驱动开发框架。
6. **同源传承**：从 agentrt 的 `atoms/corekern`（MicroCoreRT）继承实时性与微内核化设计思想。

---

## 2. 同源关系

| 维度 | agentrt（atoms/corekern） | AirymaxOS（airymaxos-kernel） |
|------|--------------------------|------------------------------|
| 设计目标 | RT 微内核 + Agent 调度 | Linux 6.6 + 微内核化 + Agent 调度 |
| 调度模型 | MicroCoreRT 实时调度 | SCHED_AGENT（sched_ext + eBPF） |
| IPC | 用户态消息队列 | io_uring 零拷贝 IPC |
| 驱动模型 | 用户态驱动 + capability | Rust 驱动 + VFIO + capability |
| 内存模型 | heapstore + memoryrovol | MemoryRovol + CXL + PMEM |

**同源传承要点**：
- 保留 agentrt 的"微内核化"哲学（最小化特权态代码）。
- 保留 agentrt 的"实时性"目标（通过 `sched_ext` 的 sub-scheduler 在 cgroup 上附加实时策略）。
- 保留 agentrt 的"消息传递"通信范式（升级为 io_uring 零拷贝实现）。

---

## 3. 目录结构

```
airymaxos-kernel/
├── linux/                 # Linux 6.6 内核源码（Linux 6.6 内核基线，git subtree）
├── patches/               # AirymaxOS 内核补丁
│   ├── sched_ext-agent/   # SCHED_AGENT 调度类（eBPF 程序）
│   ├── io_uring-ipc/      # 基于 io_uring 的 IPC 优化
│   ├── rust-drivers/      # Rust 安全驱动
│   └── microkernel/       # 微内核化改造（VFS/网络/驱动部分用户态化）
├── configs/               # 内核配置
│   ├── defconfig          # 默认配置
│   ├── defconfig-agent    # Agent 优化配置
│   └── defconfig-micro     # 微内核化配置
├── docs/                  # 设计文档
└── tests/                 # 内核测试
```

### 3.1 patches/sched_ext-agent

存放 `SCHED_AGENT` 调度类的 eBPF 程序源码。包含：
- `sched_agent.bpf.c`：核心调度器逻辑（基于 `sched_ext` BPF 接口）。
- `sub-schedulers/`：按 cgroup 附加的子调度器（实时型、批处理型、交互型、Agent 认知型）。
- `tools/`：用户态控制器（scxctl 命令行工具）。

### 3.2 patches/io_uring-ipc

基于 io_uring 构建 IPC 通道的内核侧实现：
- `io_uring_ipc.c`：注册固定 OP，支持零拷贝消息传递。
- `ring_register.c`：跨进程 ring 共享注册。
- `zerocopy.c`：基于 `MSG_ZEROCOPY` 与 page flipping 的零拷贝路径。

### 3.3 patches/rust-drivers

Rust 安全驱动框架：
- `rust/kernel/`：扩展的 Rust 内核绑定。
- `samples/rust_drivers/`：示例安全驱动（网卡、块设备、字符设备）。
- `frameworks/`：抽象框架（trait-based 驱动模型）。

### 3.4 patches/microkernel

微内核化改造补丁集：
- `vfs-userns/`：VFS 元数据操作下放至用户态服务。
- `net-userns/`：网络栈用户态化（与 `airymaxos-services/net` 协作）。
- `driver-split/`：设备驱动拆分至用户态（与 `airymaxos-services/drivers` 协作）。
- `capability/`：capability 令牌传递内核接口（与 `airymaxos-security/capability` 协作）。

---

## 4. 核心特性

### 4.1 sched_ext（AirymaxOS 内核增强，主线 6.12+，2026 成熟）

**SCHED_AGENT 调度类**：
- 通过 `sched_ext` 提供的 BPF 调度接口，在用户态实现完整调度器。
- 支持 sub-scheduler 机制：不同 cgroup 可附加不同调度策略。
- Agent 认知型 sub-scheduler：识别 CoreLoopThree 的"思考-决策-执行"三阶段，动态调整时间片。

**调度策略矩阵**：
| Sub-scheduler | 适用 cgroup | 策略 |
|---------------|------------|------|
| `scx_realtime` | system.slice | 实时优先级 |
| `scx_batch` | batch.slice | 批处理，吞吐优先 |
| `scx_interactive` | user.slice | 交互响应优先 |
| `scx_agent` | agent.slice | CoreLoopThree 三阶段感知 |

### 4.2 io_uring（2026 默认高性能 I/O）

- 零 syscall：SQ/CQ ring 共享内存，减少陷入内核次数。
- 零拷贝：`MSG_ZEROCOPY` + registered buffers + page flipping。
- 固定 OP 扩展：注册 `IORING_OP_IPC_SEND` / `IORING_OP_IPC_RECV` 等 IPC 专用 OP。
- 跨进程 ring 共享：通过 `io_uring_register` 注册 ring fd 给其他进程。
- 作为全 OS 消息传递基础（详见 `02-services.md`）。

### 4.3 eBPF（可编程内核）

- 观测：kprobe/uprobe/tracepoint 程序用于系统观测。
- 网络：XDP/TC 程序用于高性能数据路径。
- 安全：LSM BPF 程序用于安全策略（与 `airymaxos-security/ebpf-verify` 协作）。
- 调度：sched_ext 程序用于调度策略。
- CO-RE（Compile Once - Run Everywhere）：跨内核版本可移植。

### 4.4 Rust 实验性支持（Linux 6.6）

- Linux 6.6 中 Rust 持续作为实验性支持语言演进（AirymaxOS 内核增强）。
- 安全驱动开发：通过类型系统消除 UAF、Buffer Overflow、Data Race。
- 与 `airymaxos-kernel/patches/rust-drivers` 配套。

### 4.5 EEVDF 调度器（Linux 6.6）

- 混合架构：`PREEMPT_NONE`（吞吐优先）与 `PREEMPT_FULL`（响应优先）之间新增"懒惰抢占"模式。
- 适用 Agent 工作负载：大部分时间高吞吐，关键路径可被快速抢占。
- 减少 cache 抖动，提升能效。

### 4.6 微内核化改造

遵循 **Liedtke minimality principle**（Jochen Liedtke，1995）：
> 微内核只应包含机制（mechanism），不应包含策略（policy）。

改造路径：
1. **VFS 用户态化**：保留虚拟文件系统层在内核，但具体文件系统（ext4、xfs、tmpfs）实现下放至用户态服务。
2. **网络栈用户态化**：保留 socket 层在内核，协议栈（TCP/IP）下放至用户态服务（DPDK/AF_XDP）。
3. **驱动用户态化**：通过 VFIO/libvfio 将设备驱动下放至用户态进程。
4. **capability 接口**：提供内核 capability 令牌传递接口，与 `airymaxos-security` 协作。

---

## 5. 微内核思想体现

### 5.1 Liedtke minimality principle

内核仅保留以下必要机制：
- 调度器（sched_ext 框架）
- 中断/异常处理
- 地址空间管理（页表、TLB）
- IPC 机制（io_uring 基础）
- capability 令牌验证

### 5.2 机制与策略分离

- **机制在内核**：sched_ext 框架（机制）保留在内核。
- **策略在用户态**：具体调度策略（scx_agent 等）以 eBPF 程序形式运行，可热替换。

### 5.3 最小特权态代码

目标：特权态代码体积控制在 5 万行以内（参考 seL4 的约 1 万行，Zircon 的约 8 万行，传统 Linux 的约 3000 万行）。

---

## 6. AirymaxOS 工程基线

- **AirymaxOS 内核治理组**：内核社区贡献与最佳实践。
- **AirymaxOS 多版本内核策略**：支持 LTS 内核与演进内核并存。
- **AirymaxOS 内存管理**：MGLRU、CXL、THP 等特性贡献（详见 `04-memory.md`）。
- **AirymaxOS 编译工具链**：GCC、Clang、Rust 工具链集成。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| Liedtke minimality principle | seL4 / L4 | 微内核化改造哲学 |
| Capability-based security | seL4 / Zircon | 内核 capability 接口 |
| sched_ext | AirymaxOS 内核增强（主线 6.12+） | SCHED_AGENT 调度类 |
| io_uring zero-copy | Linux 5.x+ | IPC 基础设施 |
| eBPF programmable kernel | Linux 6.x+ | 观测/网络/安全/调度 |
| Rust in kernel | Linux 6.6（实验性） | 安全驱动开发 |
| EEVDF 调度器 | Linux 6.6 | 混合抢占模式 |
| Zircon capability | Google Zircon | capability 令牌设计 |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-services` | 提供 VFS/网络/驱动用户态服务的内核侧接口 |
| `airymaxos-security` | 提供 capability 令牌、LSM hook 的内核接口 |
| `airymaxos-memory` | 提供 MemoryRovol、CXL、MGLRU 的内核实现 |
| `airymaxos-cognition` | 提供 CoreLoopThree kthread、Wasm runtime 的内核支持 |
| `airymaxos-cloudnative` | 提供 containerd shim、CNI 所需的内核特性 |
| `airymaxos-system` | 提供配置工具（sysctl）所需的内核接口 |
| `airymaxos-tests` | 提供内核测试所需的可观测性接口 |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | Linux 6.6 集成 + sched_ext 基础 | 2026 Q3 |
| M2 | io_uring IPC OP 注册 + 零拷贝路径 | 2026 Q4 |
| M3 | Rust 驱动框架 + 示例驱动 | 2027 Q1 |
| M4 | VFS 用户态化（Phase 1） | 2027 Q2 |
| M5 | 网络栈用户态化（Phase 1） | 2027 Q3 |
| M6 | 微内核化改造完成（Phase 2） | 2027 Q4 |

---

## 10. 参考

- Linux 6.6 release notes（Linux 6.6 内核基线）
- Liedtke, J. "On μ-Kernel Construction"（1995）
- seL4 项目文档
- Zircon 内核设计文档
- AirymaxOS 内核治理组文档
- agentrt atoms/corekern 设计文档
