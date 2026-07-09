# Airymax IPC Bus 消息头规范

> **版本**: v0.1.0-draft | **状态**: 草案（推动为开放标准）| **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）

---

## 1. 现状问题（已验证）

### 1.1 应用层 IPC 已规范化（PARTIAL）
- `daemon/common/include/ipc_service_bus.h:67-84` 定义 `ipc_bus_message_header_t`
- 包含 `magic/version/msg_type/protocol/msg_id/correlation_id/source/target/payload_len/flags/timestamp/checksum/reserved[16]`
- 定义 `IPC_BUS_MESSAGE_MAGIC 0x49534200`、`IPC_BUS_MESSAGE_VERSION 1`

### 1.2 内核层 IPC 未规范化
- `atoms/corekern/include/ipc.h:92-98` 的 `agentos_kernel_ipc_message_t` 仅 5 字段（`code/data/size/fd/msg_id`）
- 无 magic/version/trace_id，注释明确"轻量级，40 字节"

### 1.3 未推动为开放标准
- 仅在头文件注释中描述，无 RFC/标准草案文档

## 2. 标准规范

### 2.1 统一 IPC 消息头（ARE-IPC-01）

```c
typedef struct are_ipc_message_header {
    uint32_t magic;           // 0x41524531 ("ARE1")
    uint16_t version;         // 1
    uint16_t msg_type;        // REQUEST/RESPONSE/NOTIFY/ERROR
    uint32_t protocol;        // JSON-RPC/MCP/A2A/OpenAI/Custom
    uint64_t msg_id;          // 消息唯一 ID
    uint64_t trace_id;        // 分布式追踪 ID（贯穿全链路）
    uint64_t correlation_id;  // 请求-响应关联 ID
    char source[32];          // 源 daemon 名
    char target[32];          // 目标 daemon 名
    uint32_t payload_len;     // 负载长度
    uint32_t flags;           // 标志位（压缩/加密/流式）
    uint64_t timestamp_ns;    // 纳秒时间戳（CLOCK_REALTIME）
    uint32_t checksum;        // CRC32 校验
    uint32_t reserved;        // 保留对齐
} are_ipc_message_header_t;  // 共 128 字节
```

### 2.2 开放标准推动路径

| 阶段 | 目标 |
|------|------|
| v0.1.1 | 标准文档发布（本文档） |
| v0.2.0 | 提供独立参考实现 + 一致性测试套件 |
| v0.3.0 | 提交至 IETF/RFC 草案 |
| v1.0.0 | 标准正式发布，第三方可实现兼容运行时 |

### 2.3 兼容性策略
- 内核层 IPC 保留轻量级 `agentos_kernel_ipc_message_t`（性能优先）
- 应用层 IPC 统一采用 `are_ipc_message_header_t`（规范优先）
- 内核↔应用层通过桥接层转换

## 3. 修复任务

详见 [0.1.1详细任务清单.md](../../../DocsClosed/0.1.1详细任务清单.md) P0.22（IPC Bus 规范化，16h）

---

<div align="center">

**© 2025-2026 SPHARX Ltd.**

</div>
