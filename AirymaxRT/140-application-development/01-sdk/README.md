# Airymax SDK 标准规范

> **版本**: v0.1.0-draft | **状态**: 草案 | **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证）

---

## 1. 现状问题（已验证）

### 1.1 SDK 未对齐 OpenAI API 风格（CONFIRMED）
- 四个 SDK（Rust/Python/Go/TypeScript）全部采用 Manager 模式封装自定义 RESTful 路径
- 仅暴露 `TaskManager`/`MemoryManager`/`SessionManager`/`SkillManager`
- 无 OpenAI 风格的嵌套资源 API（`/v1/chat/completions`、`/v1/responses`、`/v1/embeddings`）

### 1.2 SDK 缺失 Cognition/Safety/Tool 显式覆盖（CONFIRMED）
- 0/4 SDK 覆盖 Cognition/Safety/Tool 三类显式客户端
- 用户无法通过 SDK 直接调用认知引擎、安全审批、工具执行

## 2. 标准规范

### 2.1 双层 API 设计

**Manager 层（保留）**: 现有 `TaskManager`/`MemoryManager`/`SessionManager`/`SkillManager` 保持兼容

**嵌套资源 API 层（新增）**: 对齐 OpenAI 风格

| 资源 | 路径 | 方法 | 描述 |
|------|------|------|------|
| Cognition | `/v1/cognition/process` | POST | 触发认知循环 |
| Cognition | `/v1/cognition/tasks/{id}` | GET | 查询认知任务状态 |
| Safety | `/v1/safety/check` | POST | 提交安全审批 |
| Safety | `/v1/safety/approvals/{id}` | GET | 查询审批结果 |
| Tool | `/v1/tools/execute` | POST | 执行工具 |
| Tool | `/v1/tools/{name}` | GET | 查询工具元数据 |
| Chat | `/v1/chat/completions` | POST | OpenAI 兼容入口 |
| Embeddings | `/v1/embeddings` | POST | OpenAI 兼容入口 |

### 2.2 显式客户端覆盖

每个 SDK 必须提供以下显式客户端类：

```
CognitionClient
  - process(input) -> TaskHandle
  - get_task(id) -> Task
  - stream_events(id) -> Stream

SafetyClient
  - check(event) -> Approval
  - get_approval(id) -> Approval

ToolClient
  - execute(name, args) -> Result
  - list() -> [Tool]
  - get(name) -> Tool

ChatClient  // OpenAI 兼容
  - completions(req) -> Resp
  - embeddings(req) -> Resp
```

### 2.3 跨语言一致性

四语言 SDK 必须提供一致的 API 表面：
- Rust: `agentrt::cognition::CognitionClient`
- Python: `agentrt.cognition.CognitionClient`
- Go: `agentrt.CognitionClient`
- TypeScript: `agentrt.cognition.CognitionClient`

## 3. 修复任务

详见 [0.1.1详细任务清单](../../../docs-closed/agentrt/_design_0.1.1/03-detailed-task-list.md) P0.21（SDK 标准化，32h）

---

<div align="center">

**© 2025-2026 SPHARX Ltd.**

</div>
