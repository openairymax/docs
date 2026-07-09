Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）Agent 应用开发设计

> **文档定位**: agentrt-linux（AirymaxOS，极境智能体操作系统）Agent 应用开发工程体系主索引
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **优先级**: P1（0.1.1 仅创建 README 占位，1.0.1 完成 6 文档）
> **同源映射**: agentrt SDK（Python/Rust/Go/TypeScript 四语言）+ Linux 6.6 用户态开发模型
> **理论根基**: Linux 应用开发哲学 + Airymax K-2 接口契约化 + E-7 文档即代码

---

## 1. 模块定位

agentrt-linux Agent 应用开发体系是连接系统底座与上层智能体应用的核心工程保障。它继承 Linux 发行版 30+ 年沉淀的应用开发哲学（系统调用 + 库 ABI + 包管理 + 运行时），并在其上扩展智能体操作系统专属的 Agent 租户模型、Token 预算契约、记忆卷载 API、认知循环 SDK 等。

Agent 应用是 agentrt-linux 上的运行时租户：它通过 SDK 调用系统能力，受 Cupolas 安全穹顶约束，遵循 MicroCoreRT 调度语义，通过 AgentsIPC 与其他 Agent 或 daemon 通信。Agent 应用与 agentrt-linux 的关系不是「应用程序 vs 操作系统」的二元对立，而是「租户 vs 平台」的契约关系——平台承诺稳定 ABI，租户承诺遵守资源契约。

### 1.1 应用开发分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 系统调用层 | `AGENTRT_SYS_*` 内核系统调用 | 最底层能力 |
| L2 | AgentsIPC 协议 | 128B 消息头 + 5 种 payload | Agent 间通信 |
| L3 | C 运行时库 | libc + libagentrt + libcupolas | C 应用基础 |
| L4 | 多语言 SDK | Python / Rust / Go / TypeScript | 4 语言 4 嵌套客户端 |
| L5 | Agent 应用框架 | CognitionClient / SafetyClient / ToolClient / ChatClient | 16 嵌套客户端 |
| **L6** | **Agent 运行时租户** | **agentrt-linux 专属** | **租户隔离 + Token 预算** |
| **L7** | **Agent 编排** | **agentrt-linux 专属** | **多 Agent 协作** |

### 1.2 agentrt-linux 扩展

- **租户隔离**：每个 Agent 进程是独立租户，受 cgroup v2 + Landlock + capability 三重隔离
- **Token 预算契约**：Agent 必须声明 Token 预算，超出后由 MicroCoreRT 调度降级
- **记忆卷载 API**：Agent 通过 SDK 调用 L1→L2→L3→L4 四层记忆演化
- **认知循环 SDK**：CoreLoopThree 三层认知循环通过 SDK 暴露给 Agent

---

## 2. 核心开发机制

### 2.1 系统调用接口

Agent 应用通过 `AGENTRT_SYS_*` 系统调用访问内核能力：

```c
// airymaxos-kernel/include/uapi/agentrt/syscall.h
#define AGENTRT_SYS_COGNITION_PROCESS  1001
#define AGENTRT_SYS_MEMORY_ROVOL_GET   1002
#define AGENTRT_SYS_TOKEN_BUDGET_QUERY 1003
#define AGENTRT_SYS_AGENT_REGISTER     1004
```

所有系统调用遵循 IRON-9 v2 同源且部分代码共享原则：与 agentrt 的应用级 API 同源语义，但 agentrt-linux 为 OS 级实现。

### 2.2 AgentsIPC 协议

Agent 间通信基于 128B 定长消息头 + 5 种 payload 类型：

```c
struct agentrt_ipc_header {
    uint32_t magic;        // 0x41524531 ('ARE1'，同源 agentrt AgentsIPC)
    uint16_t version;      // 协议版本
    uint16_t type;         // REQUEST/RESPONSE/EVENT/STREAM/CONTROL
    uint32_t src_agent_id; // 发送方
    uint32_t dst_agent_id; // 接收方
    uint64_t payload_len;  // payload 长度
    uint64_t trace_id;     // 追踪 ID
    uint8_t  reserved[80]; // 预留
};  // 共 128 字节
```

### 2.3 多语言 SDK

四语言 SDK 提供 4 嵌套客户端（共 16 个嵌套客户端）：

| 语言 | 包名 | CognitionClient | SafetyClient | ToolClient | ChatClient |
|------|------|----------------|--------------|------------|------------|
| Python | `agentrt-python` | ✓ | ✓ | ✓ | ✓ |
| Rust | `agentrt-rust` | ✓ | ✓ | ✓ | ✓ |
| Go | `agentrt-go` | ✓ | ✓ | ✓ | ✓ |
| TypeScript | `agentrt-typescript` | ✓ | ✓ | ✓ | ✓ |

### 2.4 Token 预算契约

```python
from agentrt import CognitionClient

client = CognitionClient(token_budget=10000)
result = client.process(prompt="hello")
# 超出预算时 SDK 抛出 TokenBudgetExceededError
```

---

## 3. 文档索引

```
140-application-development/
├── README.md                       # 本文件
├── 01-agent-lifecycle.md          # Agent 应用生命周期
├── 02-sdk-usage.md                # 四语言 SDK 使用指南
├── 03-token-budget.md             # Token 预算契约
├── 04-memory-rovol-api.md         # 记忆卷载 API
├── 05-multi-agent-orchestration.md # 多 Agent 编排
└── 06-agent-deployment.md         # Agent 部署与运行
```

### 3.1 0.1.1 版本范围

仅完成 README 占位（P1 优先级，0.1.1 不开发具体文档）。

### 3.2 1.0.1 版本范围

完成全部 6 文档，并实施 Agent 应用开发工程标准。

---

## 4. agentrt-linux 专属扩展

### 4.1 Agent 租户模型

Agent 应用是 agentrt-linux 上的运行时租户：

| 维度 | 传统应用 | agentrt-linux Agent |
|------|---------|----------------|
| 资源隔离 | 进程级 | cgroup v2 + Landlock + capability |
| 通信 | socket / pipe | AgentsIPC 128B 协议 |
| 调度 | CFS | MicroCoreRT（sched_ext + eBPF） |
| 记忆 | 文件系统 | MemoryRovol 四层卷载 |
| 安全 | 用户权限 | Cupolas 安全穹顶 |

### 4.2 同源 agentrt SDK

agentrt 的四语言 SDK 与 agentrt-linux SDK 同源：

| 维度 | agentrt SDK | agentrt-linux SDK |
|------|------------|---------------|
| 实现层 | 用户态库 | OS 级 SDK（封装系统调用） |
| 接口 | `agentrt_cognition_*` | `agentrt_cognition_*`（同源语义） |
| 通信 | 用户态消息队列 | AgentsIPC（io_uring 零拷贝） |
| 隔离 | 进程级 | cgroup + Landlock |

agentrt 在 agentrt-linux 上运行时，SDK 调用天然适配 agentrt-linux 内核能力，无需适配层。

### 4.3 IRON-9 v2 同源且部分代码共享

应用开发遵循 IRON-9 原则：
- agentrt 应用开发规范（用户态运行时 SDK）
- agentrt-linux 应用开发规范（OS 级 SDK + 系统调用 + 租户隔离）
- 两端独立演进，但通过同源 SDK API 保持互操作

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

- `30-interfaces/03-sdk-api.md`（SDK 接口设计）
- `30-interfaces/01-syscalls.md`（系统调用接口）
- `30-interfaces/02-ipc-protocol.md`（AgentsIPC 协议）
- `50-engineering-standards/01-coding-standards.md`（编码规范）
- `50-engineering-standards/05-development-process.md`（开发流程）
- `100-operations/README.md`（Agent 运维）
- `110-security/README.md`（Agent 安全）

---

## 7. 参考材料

- Linux 6.6 `Documentation/userspace-api/`（用户态接口文档）
- Linux 6.6 `Documentation/dev-tools/`（开发工具）
- agentrt SDK 四语言实现（Python/Rust/Go/TypeScript）
- seL4 项目（capability 安全模型参考，ADR-014）
- LionsOS（seL4 Microkit 用户态应用框架参考）

---

> **文档结束** | 0.1.1 P1 占位，1.0.1 完成 6 文档
