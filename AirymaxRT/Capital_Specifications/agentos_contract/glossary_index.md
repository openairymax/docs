Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 规范集术语表与快速索引

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/agentos_contract/glossary_index.md
---

## 使用说明

本文档是 Airymax 规范集的**快速索引**,提供常用术语和章节的快速查找。完整术语定义请参见 [统一术语表](../TERMINOLOGY.md)。

---

## 快速索引

### Agent 契约相关

| 主题 | 章节 | 位置 |
|------|------|------|
| Agent ID 格式 | agent_contract.md - 3.1 节 | [Agent ID](../TERMINOLOGY.md#agent-id-智能体标识) |
| 能力描述 | agent_contract.md - 第 4 章 | [Capability](../TERMINOLOGY.md#capability-能力) |
| 成本画像 | agent_contract.md - 3.7 节 | [Cost Profile](../TERMINOLOGY.md#cost-profile-成本画像) |
| 信任度量 | agent_contract.md - 3.8 节 | [Trust Metrics](../TERMINOLOGY.md#trust-metrics-信任度量) |
| 模型配置 | agent_contract.md - 2.3.2 节 | [Cognitive Engine](../TERMINOLOGY.md#cognitive-engine-认知引擎) |
| 权限要求 | agent_contract.md - 3.6 节 | [Permission](../TERMINOLOGY.md#permission-权限) |
| 验证工具 | agent_contract.md - 第 7 章 | [Validation](../TERMINOLOGY.md#validation-验证) |

### Skill 契约相关

| 主题 | 章节 | 位置 |
|------|------|------|
| Skill 类型 | skill_contract.md - 3.1 节 | [Skill Type](../TERMINOLOGY.md#skill-type-技能类型) |
| 依赖声明 | skill_contract.md - 3.3 节 | [Dependency Declaration](../TERMINOLOGY.md#dependency-declaration-依赖声明) |
| 工具定义 | skill_contract.md - 第 4 章 | [Tool Skill](../TERMINOLOGY.md#tool-skill-工具技能) |
| 权限配置 | skill_contract.md - 第 5 章 | [Permission](../TERMINOLOGY.md#permission-权限) |
| 验证工具 | skill_contract.md - 第 7 章 | [Validation](../TERMINOLOGY.md#validation-验证) |

### 通信协议相关

| 主题 | 章节 | 位置 |
|------|------|------|
| 协议分层 | protocol_contract.md - 第 2 章 | [Protocol Stack](../TERMINOLOGY.md#protocol-stack-协议栈) |
| HTTP 网关 | protocol_contract.md - 3.1 节 | [Habitat Gateway](../TERMINOLOGY.md#habitat-gateway-habitat-网关) |
| WebSocket | protocol_contract.md - 3.2 节 | [WebSocket](../TERMINOLOGY.md#websocket) |
| stdio | protocol_contract.md - 3.3 节 | [Stdio Gateway](../TERMINOLOGY.md#stdio-gateway-stdio-网关) |
| 认证机制 | protocol_contract.md - 3.4 节 | [Bearer Token](../TERMINOLOGY.md#bearer-token-持有者令牌) |
| JSON-RPC | protocol_contract.md - 第 4 章 | [JSON-RPC 2.0](../TERMINOLOGY.md#json-rpc-20) |
| 错误码 | protocol_contract.md - 第 7 章 | [JSON-RPC Error](../TERMINOLOGY.md#json-rpc-error-json-rpc-错误) |

### 系统调用相关

| 主题 | 章节 | 位置 |
|------|------|------|
| 初始化 | syscall_api_contract.md - 3.1 节 | [Syscall Initialization](../TERMINOLOGY.md#syscall-initialization-系统调用初始化) |
| 任务管理 | syscall_api_contract.md - 3.2 节 | [Task Management](../TERMINOLOGY.md#task-management-任务管理) |
| 记忆管理 | syscall_api_contract.md - 3.3 节 | [Memory Management](../TERMINOLOGY.md#memory-management-记忆管理) |
| 会话管理 | syscall_api_contract.md - 3.4 节 | [Session Management](../TERMINOLOGY.md#session-management-会话管理) |
| 可观测性 | syscall_api_contract.md - 3.5 节 | [Telemetry](../TERMINOLOGY.md#telemetry-遥测) |
| 错误处理 | syscall_api_contract.md - 第 4 章 | [Error Handling](../TERMINOLOGY.md#error-handling-错误处理) |
| 性能考虑 | syscall_api_contract.md - 第 5 章 | [Performance Optimization](../TERMINOLOGY.md#performance-optimization-性能优化) |

### 日志格式相关

| 主题 | 章节 | 位置 |
|------|------|------|
| 日志字段 | logging_format.md - 第 2 章 | [Structured Logging](../TERMINOLOGY.md#structured-logging-结构化日志) |
| 日志级别 | logging_format.md - 3.1 节 | [Log Level](../TERMINOLOGY.md#log-level-日志级别) |
| 存储策略 | logging_format.md - 第 4 章 | [Log Retention](../TERMINOLOGY.md#log-retention-日志保留) |
| 分布式追踪 | logging_format.md - 第 5 章 | [Distributed Tracing](../TERMINOLOGY.md#distributed-tracing-分布式追踪) |
| 安全审计 | logging_format.md - 第 6 章 | [Audit Log](../TERMINOLOGY.md#audit-log-审计日志) |
| 最佳实践 | logging_format.md - 第 7 章 | [Logging Best Practices](../TERMINOLOGY.md#logging-best-practices-日志最佳实践) |

---

## 常用代码示例索引

### Agent 契约示例

| 示例 | 位置 |
|------|------|
| 产品经理 Agent 完整契约 | [agent_contract.md - 附录 B](./agent_contract.md) |
| 能力定义示例 | [agent_contract.md - 4.1 节](./agent_contract.md) |
| 成本画像示例 | [agent_contract.md - 3.7 节](./agent_contract.md) |

### Skill 契约示例

| 示例 | 位置 |
|------|------|
| GitHub 集成技能完整契约 | [skill_contract.md - 附录 B](./skill_contract.md) |
| 工具定义示例 | [skill_contract.md - 4.1 节](./skill_contract.md) |
| 依赖声明示例 | [skill_contract.md - 3.3 节](./skill_contract.md) |

### 通信协议示例

| 示例 | 位置 |
|------|------|
| HTTP 请求示例 | [protocol_contract.md - 8.2 节](./protocol_contract.md) |
| WebSocket 流式示例 | [protocol_contract.md - 8.3 节](./protocol_contract.md) |
| 系统调用示例 | [protocol_contract.md - 8.1 节](./protocol_contract.md) |

### 系统调用示例

| 示例 | 位置 |
|------|------|
| 任务提交示例 | [syscall_api_contract.md - 3.2.1 节](./syscall_api_contract.md) |
| 记忆搜索示例 | [syscall_api_contract.md - 3.3.2 节](./syscall_api_contract.md) |
| 会话创建示例 | [syscall_api_contract.md - 3.4.1 节](./syscall_api_contract.md) |

### 日志格式示例

| 示例 | 位置 |
|------|------|
| 基本日志格式 | [logging_format.md - 2.1 节](./logging_format.md) |
| 审计日志格式 | [logging_format.md - 6.2 节](./logging_format.md) |
| Span 层级示例 | [logging_format.md - 5.2 节](./logging_format.md) |

### 编码风格与安全编码示例

| 示例 | 位置 |
|------|------|
| Rust 编码风格 | [Rust_coding_style_standard.md](../coding_standard/Rust_coding_style_standard.md) |
| Rust 安全编码 | [Rust_secure_coding_standard.md](../coding_standard/Rust_secure_coding_standard.md) |
| Go 编码风格 | [Go_coding_style_standard.md](../coding_standard/Go_coding_style_standard.md) |
| Go 安全编码 | [Go_secure_coding_standard.md](../coding_standard/Go_secure_coding_standard.md) |

---

## 最佳实践索引

### Agent 契约最佳实践

| 实践 | 位置 |
|------|------|
| 明确能力边界 | [agent_contract.md - 6.1 节](./agent_contract.md) |
| 提供清晰示例 | [agent_contract.md - 6.2 节](./agent_contract.md) |
| 合理预估成本 | [agent_contract.md - 6.3 节](./agent_contract.md) |
| 定期审计更新 | [agent_contract.md - 6.4 节](./agent_contract.md) |

### Skill 契约最佳实践

| 实践 | 位置 |
|------|------|
| 单一职责 | [skill_contract.md - 7.1 节](./skill_contract.md) |
| 精确权限限定 | [skill_contract.md - 7.2 节](./skill_contract.md) |
| 完整依赖声明 | [skill_contract.md - 7.3 节](./skill_contract.md) |
| 详细文档注释 | [skill_contract.md - 7.4 节](./skill_contract.md) |

### 日志最佳实践

| 实践 | 位置 |
|------|------|
| 结构化消息 | [logging_format.md - 7.1 节](./logging_format.md) |
| 控制日志量 | [logging_format.md - 7.2 节](./logging_format.md) |
| 使用上下文管理器 | [logging_format.md - 7.3 节](./logging_format.md) |
| 异步日志 | [logging_format.md - 7.4 节](./logging_format.md) |

### 安全编码工具宏

| 宏 | 用途 | 定义位置 |
|----|------|---------|
| `MEMORY_FREE_SAFE(pptr)` | 安全释放内存并置指针为 NULL（接受指针的指针） | [C_coding_style_standard.md](../coding_standard/C_coding_style_standard.md) / `memory_compat.h` |
| `AGENTOS_SECURE_ZERO(ptr, size)` | 安全内存清零（防止编译器优化，跨平台替代 `explicit_bzero`） | [C_Cpp_secure_coding_standard.md](../coding_standard/C_Cpp_secure_coding_standard.md) / `memory_compat.h` |

---

## 错误码速查

### 系统调用错误码

| 错误码 | C 内核值 | SDK 十六进制值 | 说明 |
|--------|---------|--------------|------|
| AGENTOS_SUCCESS | 0 | 0x0000 | 成功 |
| AGENTOS_ERR_INVALID_PARAM | -2 | 0x0003 | 参数无效 |
| AGENTOS_ERR_OUT_OF_MEMORY | -4 | 0x0002 | 内存不足 |
| AGENTOS_ERR_NOT_FOUND | -6 | 0x0004 | 资源不存在 |
| AGENTOS_ERR_PERMISSION_DENIED | -10 | 0x0005 | 权限不足 |
| AGENTOS_ERR_TIMEOUT | -8 | 0x0006 | 操作超时 |

> **📝 双错误码体系说明**  
> Airymax 采用双错误码体系：C 内核使用负整数（定义于 `error.h`），SDK 使用十六进制值。完整映射关系参见 [error_code_reference.md](../project_erp/error_code_reference.md)。

**完整列表**: [syscall_api_contract.md - 第 7 章](./syscall_api_contract.md) | [error_code_reference.md](../project_erp/error_code_reference.md)

### JSON-RPC 错误码

| 错误码 | 含义 |
|--------|------|
| -32700 | Parse error |
| -32600 | Invalid Request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |

**完整列表**: [protocol_contract.md - 4.3 节](./protocol_contract.md)

---

## 核心术语速查

以下是 Airymax 最核心的 10 个术语，完整术语表请参见 [TERMINOLOGY.md](../TERMINOLOGY.md):

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

[1] Airymax 统一术语表。../TERMINOLOGY.md  
[2] 架构设计原则。../../ARCHITECTURAL_PRINCIPLES.md  
[3] 《工程控制论》. 科学出版社. 1954  
[4] 《论系统工程》. 湖南科学技术出版社. 1982  
[5] 《思考，快与慢》. 中信出版社. 2012  
[6] 产品设计哲学与用户体验思想. Apple Inc.  

---
**维护者**: Airymax 架构委员会  
**最后更新**: 2026-04-09
