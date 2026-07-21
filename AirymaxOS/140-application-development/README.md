Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Agent 应用开发设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）Agent 应用开发工程体系主索引——SDK、API、示例、MemoryRovol API 与认知 API\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[AirymaxOS 总览](../README.md)

---

## 1. 模块概述

`140-application-development/` 是 AirymaxOS 文档体系中**连接系统底座与上层智能体应用的核心工程保障**。它继承 Linux 发行版 30+ 年沉淀的应用开发哲学（系统调用 + 库 ABI + 包管理 + 运行时），并在其上扩展智能体操作系统专属的 Agent 租户模型、Token 预算契约、MemoryRovol 记忆卷载 API、CoreLoopThree 认知 API 等。

Agent 应用是 AirymaxOS 上的运行时租户：通过多语言 SDK 调用系统能力，受 Cupolas 安全穹顶（纯 C LSM）约束，遵循sched_tac 调度语义，通过 A-IPC（AgentsIPC，IORING_OP_URING_CMD 零拷贝）与其他 Agent 或 daemon 通信。

本模块承担五项核心职责：

1. **Agent 生命周期**：定义 Agent 8 态生命周期（INACTIVE → SPAWNING → READY → RUNNING → BLOCKED → STOPPING → STOPPED → DEAD）与系统能力 API。
2. **多语言 SDK 集成**：Python / Rust / Go / TypeScript 四语言 SDK，16 个嵌套客户端（CognitionClient / SafetyClient / ToolClient / ChatClient）。
3. **Token 预算契约**：令牌桶算法 + 三级阈值 + 调度集成 token_factor。
4. **MemoryRovol API**：10 个系统调用（552-561）+ 快照 7 状态生命周期 + 8 步迁移协议。
5. **认知 API**：CoreLoopThree 三阶段 + Thinkdual 模式 + 系统调用编号注册表（SSoT）。

---

## 2. 技术选型声明

agentrt-linux v1.0 在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度做出**不可妥协**的技术选型。本目录所有 SDK、API 与示例设计必须以本声明为基线——Agent 应用开发不得绕过sched_tac 调度、不得使用 page flipping 传输、不得依赖 BPF LSM 安全模型。

| # | 技术维度 | 选定方案 | 明确不采用 | 选定理由 |
|---|---------|---------|-----------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类，通过 `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，**不定义新的调度类宏** | **不使用 sched_ext**（不引入 eBPF 调度器、不依赖 `SCHED_EXT=7`、不使用 SCHED_AGENT 宏） | 保留 Linux 6.6 主线调度器稳定性与可维护性，Agent 通过既有调度类组合获得确定性与优先级，ABI 稳定 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输，registered buffer + mmap 共享页 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | IORING_OP_URING_CMD 在 Linux 6.6 已稳定，避免 page flipping 引发的 DMA 映射复杂性与跨架构兼容问题 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 Linux Security Module（`airy_lsm`），通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | 纯 C LSM 保证安全策略的可审计性与形式化验证可行性，SDK 安全客户端语义确定 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `remap_pfn_range` 映射到用户态地址空间（MemoryRovol 共享内存） | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | 提供跨架构（x86/ARM/RISC-V）一致的内存语义，记忆卷载 API 跨平台行为一致 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | v2 三层模型（升级为 v3，新增 [DSL] 降级生存层） | [DSL] 降级生存层确保 [SC] 损坏时系统仍可降级运行，SDK 多语言类型安全 |

> 详细技术选型声明见上级文档 [AirymaxOS 总览](../README.md) §2 与 [`10-architecture/10-unify-design.md`](../10-architecture/10-unify-design.md) SSoT 声明。

---

## 3. 文档索引

`140-application-development/` 由 8 个文档构成（含本 README）：

```
140-application-development/
├── README.md                       # 本文件 — Agent 应用开发主索引（v1.0）
├── 01-agent-lifecycle.md          # Agent 8 状态生命周期 + 系统调用 API + Capability 撤销（v1.0）
├── 02-sdk-integration.md          # 四语言 SDK 集成设计 + IRON-9 v3 层次（v1.0）
├── 03-agent-orchestration.md      # 多 Agent 协作模型 + DAG 工作流 + TaskFlow 引擎（v1.0）
├── 04-token-budget.md             # Token 预算契约 + 令牌桶 + 三级阈值（v1.0）
├── 05-memory-rovol-api.md         # MemoryRovol API（10 系统调用 552-561 + 快照迁移）（v1.0）
├── 06-agent-deployment.md         # Agent 部署与运行契约（.agentpkg/OCI + 四重沙箱）（v1.0）
└── 07-syscall-registry.md         # Agent 系统调用编号注册表 SSoT（v1.0）
```

### 3.1 各文档定位

| 文档 | 核心问题 | 主要产物 |
|------|---------|---------|
| README.md | 应用开发体系全貌？ | 7 层开发分层 + SDK 总览 + 技术选型声明 |
| 01-agent-lifecycle.md | Agent 怎么活？ | 8 状态生命周期 + 系统调用 API + Capability 撤销算法 + LSM Hook |
| 02-sdk-integration.md | SDK 怎么集成？ | 四语言 SDK + 16 嵌套客户端 + IRON-9 v3 [SC]/[SS] 层次 |
| 03-agent-orchestration.md | 多 Agent 怎么协作？ | DAG 工作流 + TaskFlow 引擎 + 编排模型 |
| 04-token-budget.md | Token 怎么管？ | 令牌桶算法 + 三级阈值 + 调度 token_factor 集成 |
| 05-memory-rovol-api.md | 记忆怎么读写？ | 10 系统调用契约 + 快照 7 状态 + 8 步迁移 + CXL 分层 |
| 06-agent-deployment.md | Agent 怎么部署？ | .agentpkg/OCI/RPM + 9 状态部署状态机 + 四重沙箱 |
| 07-syscall-registry.md | 编号怎么分配？ | 系统调用编号 SSoT 注册表 + 31 编号 + UAPI 模板 |

### 3.2 应用开发分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 系统调用层 | `AIRY_SYS_*` 内核系统调用（512-631） | 最底层能力 |
| L2 | AgentsIPC 协议 | 128B 消息头 + 5 种 payload | Agent 间通信 |
| L3 | C 运行时库 | libc + libagentrt + libcupolas | C 应用基础 |
| L4 | 多语言 SDK | Python / Rust / Go / TypeScript | 4 语言 4 嵌套客户端 |
| L5 | Agent 应用框架 | CognitionClient / SafetyClient / ToolClient / ChatClient | 16 嵌套客户端 |
| **L6** | **Agent 运行时租户** | **AirymaxOS 专属** | **租户隔离 + Token 预算** |
| **L7** | **Agent 编排** | **AirymaxOS 专属** | **多 Agent 协作** |

### 3.3 系统调用编号分布（SSoT 见 07-syscall-registry.md）

```c
/* kernel/include/uapi/agentrt/syscalls.h —— 完整注册表见 07-syscall-registry.md */
#define AIRY_SYS_TASK_SUBMIT        512  /* 提交 Agent 任务 */
#define AIRY_SYS_TASK_REGISTER     516  /* 注册新 Agent */
#define AIRY_SYS_IPC_SEND          532  /* IPC 零拷贝发送（IORING_OP_URING_CMD） */
#define AIRY_SYS_ROVOL_SNAPSHOT    552  /* 记忆快照（MemoryRovol） */
#define AIRY_SYS_SCHED_SET_POLICY  572  /* 设置调度策略（sched_tac） */
#define AIRY_SYS_CAPABILITY_REQUEST 592  /* 申请 capability（纯 C LSM） */
#define AIRY_SYS_CLT_PHASE_NOTIFY  612  /* 认知阶段通知（CoreLoopThree） */
```

---

## 4. Airymax Unify Design 映射

Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）在 Agent 应用开发中的关系如下。应用开发是五模块面向上层 Agent 的 API 出口。

| Unify 模块 | 全称 | 在本目录的体现 | 对应 API/SDK |
|-----------|------|--------------|------------|
| **A-IPC** | Unified Airymax IPC Fabric | **核心**：SDK IPC 接口封装、Agent 间 128B 消息头通信、IORING_OP_URING_CMD 零拷贝 fastpath | `airy_ipc_send()` / AgentsIPC SDK 客户端 |
| **A-UEF** | Unified Error and Fault Framework | **核心**：错误码 API（`AIRY_E*` 负数空间 + `AIRY_FAULT_*` 正数空间）、认知 API（CoreLoopThree 三阶段） | `airy_strerror()` / CognitionClient / `AIRY_SYS_CLT_PHASE_NOTIFY` |
| **A-UCS** | Unified Configuration Subsystem | 配置 API：sysctl/JSON 双向同步、agent.yaml 清单、Token 预算配置热重载 | `airy_config_get()` / agent.yaml Schema |
| **A-ULS** | Unified Lifecycle Supervision Framework | Agent 8 态生命周期 API、Capability 申请/撤销、四重沙箱（cgroup v2 + Landlock + seccomp + capability） | `AIRY_SYS_AGENT_SPAWN` / SafetyClient |
| **A-ULP** | Unified Logging and Printk Subsystem | Agent 日志 API、MemoryRovol 记忆卷载持久化 API（10 系统调用） | `airy_log_emit()` / MemoryRovol API（552-561） |

### 4.1 A-IPC SDK IPC 接口

Agent 应用通过 A-IPC 的 SDK 封装进行零拷贝 IPC，**底层为 IORING_OP_URING_CMD**（不使用 page flipping）：

```c
/* SDK 层封装 —— registered buffer + io_uring 提交 */
struct airy_ipc_msg msg = { .header.opcode = AIRY_IPC_OP_SEND, ... };
airy_ipc_send(ring_fd, &msg, sizeof(msg));  /* 零拷贝，不交换物理页 */
```

### 4.2 A-UEF 错误码 API

A-UEF 的 Error/Fault 双层码空间通过 SDK 暴露给 Agent 应用：

| 码空间 | 值域 | SDK 接口 | 处置 |
|--------|------|---------|------|
| Error（可恢复） | `[-300, -1]` | `airy_strerror(err)` 返回多语言消息 | 调用方处理 |
| Fault（不可恢复） | `[0x1000, 0x1FFF]` | Fault Handler / eventfd 通知 | Micro-Supervisor 接管 |

### 4.3 A-UCS 配置 API

A-UCS 的配置通过 agent.yaml 声明 + sysctl/JSON 热重载对 Agent 暴露：

```yaml
# agent.yaml —— Agent 配置清单（A-UCS 治理入口）
tokenBudget: 1000000
scheduler: SCHED_DEADLINE   # sched_tac 调度类
memoryRovol:
  layers: [L1, L2, L3, L4]
```

### 4.4 同源 agentrt SDK（IRON-9 v3）

Agent 应用 SDK 遵循 **IRON-9 v3 四层模型**与 agentrt SDK 同源：[SC] 共享 10 个头文件（`error.h`/`log_types.h`/`ipc.h` 等），[SS] 语义同源（`airy_cognition_*` 签名一致），[IND] 各自独立实现（agentrt 用户态库 ↔ AirymaxOS 系统调用封装），[DSL] 降级生存块（#ifdef AIRY_SC_FALLBACK，[SC] 损坏时最小可运行子集）。

---

## 5. 相关文档

### 5.1 上级与架构文档
- [AirymaxOS 总览](../README.md) —— 文档体系顶层纲领（v1.0）
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（五模块 SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 四层模型

### 5.2 接口与模块文档
- [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md) —— 系统调用接口设计
- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— AgentsIPC 协议（A-IPC）
- [30-interfaces/03-sdk-api.md](../30-interfaces/03-sdk-api.md) —— SDK 接口设计
- [30-interfaces/05-syscall-semantic-mapping.md](../30-interfaces/05-syscall-semantic-mapping.md) —— [SS] 语义同源映射
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— A-UEF [SC] error.h 契约
- [20-modules/04-memory.md](../20-modules/04-memory.md) —— memory 子仓（MemoryRovol）
- [20-modules/05-cognition.md](../20-modules/05-cognition.md) —— cognition 子仓（CoreLoopThree）

### 5.3 关联模块文档
- [110-security/README.md](../110-security/README.md) —— 安全加固（纯 C LSM + Cupolas + 四重沙箱）
- [170-performance/README.md](../170-performance/README.md) —— 性能工程（Token 能效 + Agent 延迟 SLO）
- [150-cloudnative/README.md](../150-cloudnative/README.md) —— 云原生部署（Agent 容器化）
- [50-engineering-standards/01-coding-standards.md](../50-engineering-standards/01-coding-standards.md) —— 编码规范

### 5.4 参考材料
- Linux 6.6 `Documentation/userspace-api/`（用户态接口文档）
- seL4 `src/object/tcb.c`（TCB 线程生命周期）+ `src/object/cnode.c`（Capability 撤销算法）

---

## 6. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，7/7 文档完成（Agent 生命周期 + SDK + Token + MemoryRovol + 部署 + 注册表） |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）；IORING_OP_URING_CMD（不使用 page flipping）；纯 C LSM（不使用 BPF LSM）；alloc_pages + mmap（不使用 DMA 一致性内存）；IRON-9 v3 四层模型（新增 [DSL] 降级生存层）；新增 Airymax Unify Design 五模块映射（A-IPC SDK IPC 接口 / A-UEF 错误码 API / A-UCS 配置 API） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | Agent 应用开发模块 v1.0 | 共 8 文档 | 维护者：开源极境工程与规范委员会 | Agent 应用是 AirymaxOS 上的运行时租户
