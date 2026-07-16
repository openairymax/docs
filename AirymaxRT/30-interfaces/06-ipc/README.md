# Airymax IPC Bus 消息头规范
> **文档定位**：Airymax IPC Bus 消息头规范\
> **文档版本**：v0.1.0-draft | **状态**: 草案（推动为开放标准）| **日期**: 2026-07-04\
> **版权归属人**：SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）

---

## 1. 现状问题（已验证）

### 1.1 应用层 IPC 已规范化（PARTIAL）
- `daemon/common/include/ipc_service_bus.h:67-84` 定义 `ipc_bus_message_header_t`
- 包含 `magic/version/msg_type/protocol/msg_id/correlation_id/source/target/payload_len/flags/timestamp/checksum/reserved[16]`
- 定义 `IPC_BUS_MESSAGE_MAGIC 0x49534200`、`IPC_BUS_MESSAGE_VERSION 1`

### 1.2 内核层 IPC 未规范化
- `atoms/corekern/include/ipc.h:92-98` 的 `airy_kernel_ipc_message_t` 仅 5 字段（`code/data/size/fd/msg_id`）
- 无 magic/version/trace_id，注释明确"轻量级，40 字节"

### 1.3 未推动为开放标准
- 仅在头文件注释中描述，无 RFC/标准草案文档

## 2. 标准规范

### 2.1 统一 IPC 消息头（ARE-IPC-01）

> **SSoT 声明**：IPC 消息头结构体权威定义位于 `include/airymax/ipc.h`（[SC] 共享契约层，agentrt 与 agentrt-linux 共享同一物理头文件，逐字节相同）。本节代码必须与 `docs/AirymaxRT/50-engineering-standards/120-cross-project-code-sharing.md` §2.7（`struct airy_ipc_msg_hdr`，128 字节，8 字段，`__attribute__((packed))`，`_Static_assert(sizeof(...) == 128)`）以及源码 `agentrt/commons/include/airymax/ipc.h:67-80` 逐字节一致。
>
> **修正说明**：本节早期草案使用 `are_ipc_message_header_t`（typedef）+ `uint32_t/uint16_t/uint64_t` + 13 字段 + 无 `__attribute__((packed))` + 无 `_Static_assert`，与 [SC] SSoT 三重违规（命名/类型/布局）。已统一为 SSoT 版本 `struct airy_ipc_msg_hdr`（无 typedef）+ `__u32/__u16/__u64` UAPI 类型 + 8 字段 + `__attribute__((packed))` + `_Static_assert`。开放标准接口（`are_ipc_*` 函数族）仍可由第三方实现，但消息头结构体必须与 [SC] SSoT 逐字节一致。

```c
/* include/airymax/ipc.h —— [SC] 共享契约层，agentrt 与 agentrt-linux 逐字节共享 */
#include <airymax/uapi_compat.h>  /* __u32/__u16/__u64/__u8 跨平台 UAPI 类型 */

#define AIRY_IPC_MAGIC    0x41524531u   /* 'ARE1' —— IPC 消息头 magic */
#define AIRY_IPC_HDR_SZ   128           /* 固定 128 字节消息头 */

/**
 * struct airy_ipc_msg_hdr - IPC message header (128 bytes, [SC] shared)
 * @magic: Must be AIRY_IPC_MAGIC (0x41524531 'ARE1').
 * @opcode: Operation code; see enum airy_ipc_op.
 * @flags: Message flags (AIRY_IPC_F_*).
 * @trace_id: Distributed trace identifier for cross-daemon tracing.
 * @timestamp_ns: Timestamp in nanoseconds (CLOCK_REALTIME).
 * @src_task: Source task identifier.
 * @dst_task: Destination task identifier.
 * @payload_len: Payload length in bytes (excludes this 128B header).
 * @reserved: 84 bytes padding, must be zero.
 *
 * Fixed 128-byte header, layout never changes. Shared between
 * agentrt (user-space) and agentrt-linux (kernel) via [SC] contract layer.
 * Recipients MUST validate magic before processing.
 */
struct airy_ipc_msg_hdr {
    __u32   magic;          /* offset 0,  'ARE1' (0x41524531) */
    __u16   opcode;         /* offset 4,  操作码（见 enum airy_ipc_op） */
    __u16   flags;          /* offset 6,  消息标志（AIRY_IPC_F_*） */
    __u64   trace_id;       /* offset 8,  分布式追踪 ID（贯穿全链路） */
    __u64   timestamp_ns;   /* offset 16, 纳秒时间戳（CLOCK_REALTIME） */
    __u64   src_task;       /* offset 24, 源任务 ID（整型 task ID，非字符串） */
    __u64   dst_task;       /* offset 32, 目标任务 ID（整型 task ID，非字符串） */
    __u32   payload_len;    /* offset 40, 负载长度（不含本 128B 头） */
    __u8    reserved[84];   /* offset 44, 84 字节保留，必须为零 */
} __attribute__((packed));

_Static_assert(sizeof(struct airy_ipc_msg_hdr) == 128,
    "IPC message header must be exactly 128 bytes");
```

**与早期草案的对应关系**（仅供迁移参考）：

| 早期草案字段（已废弃） | SSoT 字段（`struct airy_ipc_msg_hdr`） | 说明 |
|---------------------|--------------------------------------|------|
| `uint32_t magic` | `__u32 magic` | 类型改为 UAPI `__u32` |
| `uint16_t version` | — | 移除（version 由 `opcode`/`flags` 承载） |
| `uint16_t msg_type` | `__u16 opcode` | 重命名为 `opcode`，对齐 `enum airy_ipc_op` |
| `uint32_t protocol` | — | 移除（protocol 由上层协议层处理，不在 128B 头中） |
| `uint64_t msg_id` | — | 移除（msg_id 由 `trace_id` + `src_task` 推导） |
| `uint64_t trace_id` | `__u64 trace_id` | 保留，类型改为 UAPI `__u64` |
| `uint64_t correlation_id` | — | 移除（correlation 由 `trace_id` 复用） |
| `char source[32]` | `__u64 src_task` | 改为整型 task ID（字符串名易变且占空间） |
| `char target[32]` | `__u64 dst_task` | 改为整型 task ID |
| `uint32_t payload_len` | `__u32 payload_len` | 保留，类型改为 UAPI `__u32` |
| `uint32_t flags` | `__u16 flags` | 类型改为 UAPI `__u16`（16 位足够） |
| `uint64_t timestamp_ns` | `__u64 timestamp_ns` | 保留，类型改为 UAPI `__u64` |
| `uint32_t checksum` | — | 移除（CRC32 由传输层处理，不在固定头中） |
| `uint32_t reserved` | `__u8 reserved[84]` | 扩展为 84 字节保留区，凑齐 128 字节 |

### 2.2 开放标准推动路径

| 阶段 | 目标 |
|------|------|
| v0.1.1 | 标准文档发布（本文档） |
| v1.0.1 | 标准正式发布，第三方可实现兼容运行时（IRON-8：0.1.1→1.0.1 直接过渡） |

### 2.3 兼容性策略
- 内核层 IPC 保留轻量级 `airy_kernel_ipc_message_t`（性能优先）
- 应用层 IPC 统一采用 [SC] SSoT `struct airy_ipc_msg_hdr`（128 字节固定头，规范优先）
- 内核↔应用层通过桥接层转换

## 3. 修复任务

详见 [0.1.1详细任务清单](../../../docs-closed/agentrt/_design_0.1.1/03-detailed-task-list.md) P0.22（IPC Bus 规范化，16h）

---

<div align="center">

**© 2025-2026 SPHARX Ltd.**

</div>
