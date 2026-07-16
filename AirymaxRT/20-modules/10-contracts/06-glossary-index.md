Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 规范集术语表与快速索引
> **文档定位**：Airymax 规范集术语表与快速索引\
> **最后更新**：2026-06-09

---

## 使用说明

本文档是 Airymax 规范集的**快速索引**,提供常用术语和章节的快速查找。完整术语定义请参见 [统一术语表](../../TERMINOLOGY.md)。

---

## 快速索引

### Agent 契约相关

| 主题 | 章节 | 位置 |
|------|------|------|
| Agent ID 格式 | 01-agent-contract.md - 3.1 节 | [Agent ID](../../TERMINOLOGY.md#agent-id-智能体标识) |
| 能力描述 | 01-agent-contract.md - 第 4 章 | [Capability](../../TERMINOLOGY.md#capability-能力) |
| 成本画像 | 01-agent-contract.md - 3.7 节 | [Cost Profile](../../TERMINOLOGY.md#cost-profile-成本画像) |
| 信任度量 | 01-agent-contract.md - 3.8 节 | [Trust Metrics](../../TERMINOLOGY.md#trust-metrics-信任度量) |
| 模型配置 | 01-agent-contract.md - 2.3.2 节 | [Cognitive Engine](../../TERMINOLOGY.md#cognitive-engine-认知引擎) |
| 权限要求 | 01-agent-contract.md - 3.6 节 | [Permission](../../TERMINOLOGY.md#permission-权限) |
| 验证工具 | 01-agent-contract.md - 第 7 章 | [Validation](../../TERMINOLOGY.md#validation-验证) |

### Skill 契约相关

| 主题 | 章节 | 位置 |
|------|------|------|
| Skill 类型 | 02-skill-contract.md - 3.1 节 | [Skill Type](../../TERMINOLOGY.md#skill-type-技能类型) |
| 依赖声明 | 02-skill-contract.md - 3.3 节 | [Dependency Declaration](../../TERMINOLOGY.md#dependency-declaration-依赖声明) |
| 工具定义 | 02-skill-contract.md - 第 4 章 | [Tool Skill](../../TERMINOLOGY.md#tool-skill-工具技能) |
| 权限配置 | 02-skill-contract.md - 第 5 章 | [Permission](../../TERMINOLOGY.md#permission-权限) |
| 验证工具 | 02-skill-contract.md - 第 7 章 | [Validation](../../TERMINOLOGY.md#validation-验证) |

### 通信协议相关

| 主题 | 章节 | 位置 |
|------|------|------|
| 协议分层 | 03-protocol-contract.md - 第 2 章 | [Protocol Stack](../../TERMINOLOGY.md#protocol-stack-协议栈) |
| HTTP 网关 | 03-protocol-contract.md - 3.1 节 | [Habitat Gateway](../../TERMINOLOGY.md#habitat-gateway-habitat-网关) |
| WebSocket | 03-protocol-contract.md - 3.2 节 | [WebSocket](../../TERMINOLOGY.md#websocket) |
| stdio | 03-protocol-contract.md - 3.3 节 | [Stdio Gateway](../../TERMINOLOGY.md#stdio-gateway-stdio-网关) |
| 认证机制 | 03-protocol-contract.md - 3.4 节 | [Bearer Token](../../TERMINOLOGY.md#bearer-token-持有者令牌) |
| JSON-RPC | 03-protocol-contract.md - 第 4 章 | [JSON-RPC 2.0](../../TERMINOLOGY.md#json-rpc-20) |
| 错误码 | 03-protocol-contract.md - 第 7 章 | [JSON-RPC Error](../../TERMINOLOGY.md#json-rpc-error-json-rpc-错误) |

### 系统调用相关

| 主题 | 章节 | 位置 |
|------|------|------|
| 初始化 | 04-syscall-api-contract.md - 3.1 节 | [Syscall Initialization](../../TERMINOLOGY.md#syscall-initialization-系统调用初始化) |
| 任务管理 | 04-syscall-api-contract.md - 3.2 节 | [Task Management](../../TERMINOLOGY.md#task-management-任务管理) |
| 记忆管理 | 04-syscall-api-contract.md - 3.3 节 | [Memory Management](../../TERMINOLOGY.md#memory-management-记忆管理) |
| 会话管理 | 04-syscall-api-contract.md - 3.4 节 | [Session Management](../../TERMINOLOGY.md#session-management-会话管理) |
| 可观测性 | 04-syscall-api-contract.md - 3.5 节 | [Telemetry](../../TERMINOLOGY.md#telemetry-遥测) |
| 错误处理 | 04-syscall-api-contract.md - 第 4 章 | [Error Handling](../../TERMINOLOGY.md#error-handling-错误处理) |
| 性能考虑 | 04-syscall-api-contract.md - 第 5 章 | [Performance Optimization](../../TERMINOLOGY.md#performance-optimization-性能优化) |

### 日志格式相关

| 主题 | 章节 | 位置 |
|------|------|------|
| 日志字段 | 05-logging-format.md - 第 2 章 | [Structured Logging](../../TERMINOLOGY.md#structured-logging-结构化日志) |
| 日志级别 | 05-logging-format.md - 3.1 节 | [Log Level](../../TERMINOLOGY.md#log-level-日志级别) |
| 存储策略 | 05-logging-format.md - 第 4 章 | [Log Retention](../../TERMINOLOGY.md#log-retention-日志保留) |
| 分布式追踪 | 05-logging-format.md - 第 5 章 | [Distributed Tracing](../../TERMINOLOGY.md#distributed-tracing-分布式追踪) |
| 安全审计 | 05-logging-format.md - 第 6 章 | [Audit Log](../../TERMINOLOGY.md#audit-log-审计日志) |
| 最佳实践 | 05-logging-format.md - 第 7 章 | [Logging Best Practices](../../TERMINOLOGY.md#logging-best-practices-日志最佳实践) |

---

## 常用代码示例索引

### Agent 契约示例

| 示例 | 位置 |
|------|------|
| 产品经理 Agent 完整契约 | [01-agent-contract.md - 附录 B](./01-agent-contract.md) |
| 能力定义示例 | [01-agent-contract.md - 4.1 节](./01-agent-contract.md) |
| 成本画像示例 | [01-agent-contract.md - 3.7 节](./01-agent-contract.md) |

### Skill 契约示例

| 示例 | 位置 |
|------|------|
| GitHub 集成技能完整契约 | [02-skill-contract.md - 附录 B](./02-skill-contract.md) |
| 工具定义示例 | [02-skill-contract.md - 4.1 节](./02-skill-contract.md) |
| 依赖声明示例 | [02-skill-contract.md - 3.3 节](./02-skill-contract.md) |

### 通信协议示例

| 示例 | 位置 |
|------|------|
| HTTP 请求示例 | [03-protocol-contract.md - 8.2 节](./03-protocol-contract.md) |
| WebSocket 流式示例 | [03-protocol-contract.md - 8.3 节](./03-protocol-contract.md) |
| 系统调用示例 | [03-protocol-contract.md - 8.1 节](./03-protocol-contract.md) |

### 系统调用示例

| 示例 | 位置 |
|------|------|
| 任务提交示例 | [04-syscall-api-contract.md - 3.2.1 节](./04-syscall-api-contract.md) |
| 记忆搜索示例 | [04-syscall-api-contract.md - 3.3.2 节](./04-syscall-api-contract.md) |
| 会话创建示例 | [04-syscall-api-contract.md - 3.4.1 节](./04-syscall-api-contract.md) |

### 日志格式示例

| 示例 | 位置 |
|------|------|
| 基本日志格式 | [05-logging-format.md - 2.1 节](./05-logging-format.md) |
| 审计日志格式 | [05-logging-format.md - 6.2 节](./05-logging-format.md) |
| Span 层级示例 | [05-logging-format.md - 5.2 节](./05-logging-format.md) |

### 编码风格与安全编码示例

| 示例 | 位置 |
|------|------|
| Rust 编码风格 | [Rust_coding_style.md](../../50-engineering-standards/10-coding-style/Rust_coding_style.md) |
| Rust 安全编码 | [Rust_coding_style.md](../../50-engineering-standards/10-coding-style/Rust_coding_style.md) |
| Go 编码风格 | [Go_coding_style.md](../../50-engineering-standards/10-coding-style/Go_coding_style.md) |
| Go 安全编码 | [Go_coding_style.md](../../50-engineering-standards/10-coding-style/Go_coding_style.md) |

---

## 最佳实践索引

### Agent 契约最佳实践

| 实践 | 位置 |
|------|------|
| 明确能力边界 | [01-agent-contract.md - 6.1 节](./01-agent-contract.md) |
| 提供清晰示例 | [01-agent-contract.md - 6.2 节](./01-agent-contract.md) |
| 合理预估成本 | [01-agent-contract.md - 6.3 节](./01-agent-contract.md) |
| 定期审计更新 | [01-agent-contract.md - 6.4 节](./01-agent-contract.md) |

### Skill 契约最佳实践

| 实践 | 位置 |
|------|------|
| 单一职责 | [02-skill-contract.md - 7.1 节](./02-skill-contract.md) |
| 精确权限限定 | [02-skill-contract.md - 7.2 节](./02-skill-contract.md) |
| 完整依赖声明 | [02-skill-contract.md - 7.3 节](./02-skill-contract.md) |
| 详细文档注释 | [02-skill-contract.md - 7.4 节](./02-skill-contract.md) |

### 日志最佳实践

| 实践 | 位置 |
|------|------|
| 结构化消息 | [05-logging-format.md - 7.1 节](./05-logging-format.md) |
| 控制日志量 | [05-logging-format.md - 7.2 节](./05-logging-format.md) |
| 使用上下文管理器 | [05-logging-format.md - 7.3 节](./05-logging-format.md) |
| 异步日志 | [05-logging-format.md - 7.4 节](./05-logging-format.md) |

### 安全编码工具宏

| 宏 | 用途 | 定义位置 |
|----|------|---------|
| `MEMORY_FREE_SAFE(pptr)` | 安全释放内存并置指针为 NULL（接受指针的指针） | [C_Cpp_coding_style.md](../../50-engineering-standards/10-coding-style/C_Cpp_coding_style.md) / `memory_compat.h` |
| `AIRY_SECURE_ZERO(ptr, size)` | 安全内存清零（防止编译器优化，跨平台替代 `explicit_bzero`） | [C_Cpp_coding_style.md](../../50-engineering-standards/10-coding-style/C_Cpp_coding_style.md) / `memory_compat.h` |

---

## 错误码速查

### 系统调用错误码

> **SSoT 声明**：错误码权威定义位于 `agentrt/commons/include/airy_types.h`（`airy_err_t` 类型 + `AIRY_E*` POSIX 码）和 `agentrt/commons/utils/error/include/error.h`（`AIRY_ERR_*` 扩展码）。`include/airymax/error.h` 为规划中的 [SC] 共享路径，当前尚未创建。类型 `airy_err_t = int32_t` 定义于 `airy_types.h:41`。C 内核值与 `docs/AirymaxRT/50-engineering-standards/120-cross-project-code-sharing.md` §2.1（13 个 `AIRY_E*` 宏，对齐 POSIX errno 负值）逐字节一致；成功码统一为 `AIRY_EOK = 0`。

| 错误码 | C 内核值 | SDK 十六进制值 | 说明 |
|--------|---------|--------------|------|
| AIRY_EOK | 0 | 0x0000 | 成功 |
| AIRY_EPERM | -1 | 0x0005 | 权限不足（对齐 POSIX EPERM） |
| AIRY_ENOENT | -2 | 0x0004 | 资源不存在（对齐 POSIX ENOENT） |
| AIRY_ENOMEM | -12 | 0x0002 | 内存不足（对齐 POSIX ENOMEM） |
| AIRY_EINVAL | -22 | 0x0003 | 参数无效（对齐 POSIX EINVAL） |
| AIRY_ETIMEDOUT | -110 | 0x0006 | 操作超时（对齐 POSIX ETIMEDOUT） |

> **📝 双错误码体系说明**  
> Airymax 采用双错误码体系：C 内核使用负整数（权威定义于 `agentrt/commons/include/airy_types.h`（`AIRY_E*` POSIX 码）+ `agentrt/commons/utils/error/include/error.h`（`AIRY_ERR_*` 扩展码），[SC] 实际权威源；`include/airymax/error.h` 为规划中的 [SC] 共享路径，当前尚未创建；`airy_err_t` 类型定义于 `airy_types.h:41`），SDK 使用十六进制值。完整映射关系参见 [error_code_reference.md](../../50-engineering-standards/50-project-erp/error_code_reference.md)。

**完整列表**: [04-syscall-api-contract.md - 第 7 章](./04-syscall-api-contract.md) | [error_code_reference.md](../../50-engineering-standards/50-project-erp/error_code_reference.md)

### JSON-RPC 错误码

| 错误码 | 含义 |
|--------|------|
| -32700 | Parse error |
| -32600 | Invalid Request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |

**完整列表**: [03-protocol-contract.md - 4.3 节](./03-protocol-contract.md)

---

## 核心术语速查

以下是 Airymax 最核心的 10 个术语，完整术语表请参见 [TERMINOLOGY.md](../../TERMINOLOGY.md):

1. **Agent (智能体)** - 具有认知能力的实体，通过契约描述自身能力
2. **Skill (技能)** - 可复用的执行单元，为 Agent 提供具体能力
3. **Contract (契约)** - 机器可读的能力描述文件 (JSON 格式)
4. **Dispatcher (调度官)** - 根据任务需求选择最合适 Agent 的组件
5. **安全穹顶 (Cupolas)** - 提供虚拟工位、权限裁决、安全隔离等核心安全机制的内核子系统
6. **系统调用 (Syscall)** - 用户态程序与内核态交互的唯一接口，遵循最小权限原则
7. **追踪标识 (TraceID)** - 分布式追踪中请求链路的唯一标识，支持全链路可观测性
8. **微核心/原子核心 (MicroCoreRT/CoreKern)** - 只提供IPC、内存管理、任务调度、时间服务等原子机制的最小化内核
9. **权限 (Permission)** - 执行特定操作所需的授权，遵循最小权限原则和安全内生设计
10. **可观测性 (Observability)** - 通过日志、指标、追踪、健康检查实现系统行为的全面透明化

---

## 参考文献与理论基础

[1] Airymax 统一术语表。../../TERMINOLOGY.md  
[2] 架构设计原则。../../10-architecture/00-architectural-principles.md  
[3] 《工程控制论》. 科学出版社. 1954  
[4] 《论系统工程》. 湖南科学技术出版社. 1982  
[5] 《思考，快与慢》. 中信出版社. 2012  
[6] 产品设计哲学与用户体验思想. Apple Inc.  

---
**维护者**: Airymax 架构委员会  
**最后更新**: 2026-04-09
