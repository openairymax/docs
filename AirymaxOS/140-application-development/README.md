Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）Agent 应用开发设计

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）Agent 应用开发工程体系主索引\
> **版本**：0.1.1\
> **最后更新**：2026-07-13\
> **优先级**：P1（7 文档）\
> **同源映射**：agentrt SDK（Python/Rust/Go/TypeScript 四语言）+ Linux 6.6 用户态开发模型 + seL4 TCB 生命周期\
> **理论根基**：Linux 应用开发哲学 + Airymax K-2 接口契约化 + E-7 文档即代码

> **审查状态**：Wave 2 v2 源码级深读审查完成（Phase A/B/C/D），Agent 应用开发已通过审查（应用开发层 Phase B/C/D 验证：系统调用编号 SSoT 注册表 31 编号 + Agent 8 状态生命周期 + seL4 TCB 生命周期借鉴 + D9 目录结构 SSoT 对齐）

---

## 1. 模块定位

agentrt-linux Agent 应用开发体系是连接系统底座与上层智能体应用的核心工程保障。它继承 Linux 发行版 30+ 年沉淀的应用开发哲学（系统调用 + 库 ABI + 包管理 + 运行时），并在其上扩展智能体操作系统专属的 Agent 租户模型、Token 预算契约、记忆卷载 API、认知循环 SDK 等。

Agent 应用是 agentrt-linux 上的运行时租户：它通过 SDK 调用系统能力，受 Cupolas 安全穹顶约束，遵循 MicroCoreRT 调度语义，通过 AgentsIPC 与其他 Agent 或 daemon 通信。

### 1.1 应用开发分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 系统调用层 | `AIRY_SYS_*` 内核系统调用 | 最底层能力 |
| L2 | AgentsIPC 协议 | 128B 消息头 + 5 种 payload | Agent 间通信 |
| L3 | C 运行时库 | libc + libagentrt + libcupolas | C 应用基础 |
| L4 | 多语言 SDK | Python / Rust / Go / TypeScript | 4 语言 4 嵌套客户端 |
| L5 | Agent 应用框架 | CognitionClient / SafetyClient / ToolClient / ChatClient | 16 嵌套客户端 |
| **L6** | **Agent 运行时租户** | **agentrt-linux 专属** | **租户隔离 + Token 预算** |
| **L7** | **Agent 编排** | **agentrt-linux 专属** | **多 Agent 协作** |

---

## 2. 核心开发机制

### 2.1 系统调用接口

Agent 应用通过 `AIRY_SYS_*` 系统调用访问内核能力。系统调用编号从 512 起始分配（避开 Linux 6.6 标准 0-511 编号空间），按 6 类分段，每类预留 20 个编号：

```c
/* kernel/include/uapi/agentrt/syscalls.h */
/* 完整编号注册表见 07-syscall-registry.md（SSoT） */
#define AIRY_SYS_TASK_SUBMIT        512  /* 提交 Agent 任务 */
#define AIRY_SYS_TASK_REGISTER     516  /* 注册新 Agent */
#define AIRY_SYS_IPC_SEND          532  /* IPC 零拷贝发送 */
#define AIRY_SYS_ROVOL_SNAPSHOT    552  /* 记忆快照 */
#define AIRY_SYS_SCHED_SET_POLICY  572  /* 设置调度策略 */
#define AIRY_SYS_CAPABILITY_REQUEST 592  /* 申请 capability */
#define AIRY_SYS_CLT_PHASE_NOTIFY  612  /* 认知阶段通知 */
```

> **编号 SSoT**：完整的系统调用编号注册表见 [07-syscall-registry.md](07-syscall-registry.md)，涵盖 31 个已分配编号 + 89 个预留编号，是编号分配的唯一权威来源。

### 2.2 AgentsIPC 协议

Agent 间通信基于 128B 定长消息头 + 5 种 payload 类型。

### 2.3 多语言 SDK

四语言 SDK 提供 4 嵌套客户端（共 16 个嵌套客户端）：

| 语言 | 包名 | CognitionClient | SafetyClient | ToolClient | ChatClient |
|------|------|----------------|--------------|------------|------------|
| Python | `agentrt-python` | ✓ | ✓ | ✓ | ✓ |
| Rust | `agentrt-rust` | ✓ | ✓ | ✓ | ✓ |
| Go | `agentrt-go` | ✓ | ✓ | ✓ | ✓ |
| TypeScript | `agentrt-typescript` | ✓ | ✓ | ✓ | ✓ |

---

## 3. 文档索引

```
140-application-development/
├── README.md                       # 本文件
├── 01-agent-lifecycle.md          # Agent 应用生命周期 
├── 02-sdk-integration.md          # 四语言 SDK 集成设计 
├── 03-agent-orchestration.md      # Agent 编排设计 
├── 04-token-budget.md             # Token 预算契约 
├── 05-memory-rovol-api.md         # 记忆卷载 API 
├── 06-agent-deployment.md         # Agent 部署与运行契约 
└── 07-syscall-registry.md         # Agent 系统调用编号注册表 
```

### 3.1 0.1.1 版本范围

完成 README + 01-agent-lifecycle.md（Agent 8 状态生命周期 + 系统调用 API + Capability 撤销算法 + LSM 风格 Hook）+ 02-sdk-integration.md（四语言 SDK 集成设计 + IRON-9 v2 [SC]/[SS] 层次）+ 03-agent-orchestration.md（多 Agent 协作模型 + DAG 工作流 + TaskFlow 引擎）+ 04-token-budget.md（令牌桶算法 + 三级阈值 + 系统调用集成 523/524 + 调度集成 token_factor + SDK 集成 + 错误码统一）+ 05-memory-rovol-api.md（10 个系统调用 552-561 完整契约 + 快照 7 状态生命周期 + 8 步迁移协议 + CXL 分层 + MGLRU 配置 + 艾宾浩斯层级升降级 + 四语言 SDK 集成 + 3 个专用错误码）+ 06-agent-deployment.md（Agent 包格式 .agentpkg/OCI/RPM + agent.yaml 完整清单 Schema + 9 状态部署生命周期状态机 + 四重沙箱 cgroup v2/Landlock/seccomp/capability + 三类健康检查探针 + 三种滚动更新策略 rolling/blue-green/canary + MemoryRovol 快照回滚 + 4 维资源配额 + agentdeploy CLI + Python/Rust SDK 集成）+ 07-syscall-registry.md（系统调用编号 SSoT 注册表 + 31 编号 + UAPI 模板 + 审批流程 + Agent 生命周期 API 映射）。**140 模块 7/7 文档全部完成（100%）**。

### 3.2 1.0.1 版本范围

140 模块 覆盖工程标准实施与 SDK 实现验证，包括：SDK 实现的契约合规性验证、Agent 包构建工具链（agentpkg/agentctl）、部署状态机的集成测试、滚动更新与回滚的端到端验证。

---

## 4. agentrt-linux 专属扩展

### 4.1 Agent 租户模型

| 维度 | 传统应用 | agentrt-linux Agent |
|------|---------|----------------|
| 资源隔离 | 进程级 | cgroup v2 + Landlock + capability |
| 通信 | socket / pipe | AgentsIPC 128B 协议 |
| 调度 | CFS | MicroCoreRT（sched_ext + eBPF） |
| 记忆 | 文件系统 | MemoryRovol 四层卷载 |
| 安全 | 用户权限 | Cupolas 安全穹顶 |

### 4.2 同源 agentrt SDK

| 维度 | agentrt SDK | agentrt-linux SDK |
|------|------------|---------------|
| 实现层 | 用户态库 | OS 级 SDK（封装系统调用） |
| 接口 | `airy_cognition_*` | `airy_cognition_*`（同源语义） |
| 通信 | 用户态消息队列 | AgentsIPC（io_uring 零拷贝） |
| 隔离 | 进程级 | cgroup + Landlock |

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **K-2 接口契约化** | SDK API 契约 + Token 预算契约 |
| **E-7 文档即代码** | SDK 文档与代码同源同审 |
| **E-8 可测试性** | Agent 行为契约测试 |
| **S-1 反馈闭环** | Agent 运行时反馈到调度策略 |
| **IRON-9 v2 同源且部分代码共享** | 与 agentrt SDK 同源 |

---

## 6. 相关文档

- [01-agent-lifecycle.md](01-agent-lifecycle.md) — Agent 8 状态生命周期 + 系统调用 API
- `30-interfaces/03-sdk-api.md` — SDK 接口设计
- `30-interfaces/01-syscalls.md` — 系统调用接口
- `30-interfaces/02-ipc-protocol.md` — AgentsIPC 协议
- `50-engineering-standards/01-coding-standards.md` — 编码规范
- `50-engineering-standards/05-development-process.md` — 开发流程
- `110-security/README.md` — Agent 安全

---

## 7. 参考材料

- Linux 6.6 `Documentation/userspace-api/`（用户态接口文档）
- 主流 Linux 发行版 Linux 6.6 内核基线 `kernel/sched/sched.h`（sched_class 虚表注册模式）
- seL4 `src/object/tcb.c`（TCB 线程生命周期）
- seL4 `src/object/cnode.c`（Capability 撤销算法）
- seL4 `src/fastpath/fastpath.c`（Point of No Return 原子状态转换）

---
> **文档结束** | README + 01
