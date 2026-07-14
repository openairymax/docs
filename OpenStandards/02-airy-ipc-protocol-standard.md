Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax IPC 协议标准  

> **文档定位**: Airymax 开放标准
> **版本**: 1.0（开放标准草案）
> **最后更新**: 2026-07-09
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 一、范围与目的

本标准定义 AI Agent 之间、Agent 与运行时之间的进程间通信（IPC）开放协议契约，覆盖：

- 128 字节定长消息头二进制布局
- 5 种 payload 类型语义
- 请求-响应模式与发布-订阅模式
- Wireshark dissector 兼容性约定

本标准面向：

- **运行时实现者**：实现兼容的 IPC 传输层（用户态消息队列、内核 io_uring 等）。
- **Agent 作者**：使用标准 IPC 接口收发消息。
- **协议工具开发者**：构建 Wireshark dissector、抓包分析器、协议测试器。
- **跨运行时互通方**：在不同实现间交换 IPC 消息。

本标准仅规定**协议格式与语义**，不规定底层传输（io_uring、共享内存、套接字等均可作为承载）。本标准遵循 IRON-9 v2 [SC] 共享契约层——消息头字节布局在 agentrt 与 agentrt-linux 之间完全共享。

### 1.1 术语

| 术语 | 含义 |
|------|------|
| AgentsIPC | 本标准定义的 Agent 间通信协议 |
| 消息头（Header） | 128 字节定长消息元数据 |
| payload | 消息头之后的数据负载 |
| magic | 4 字节协议识别魔数 |
| trace_id | 跨端链路追踪标识符 |

### 1.2 规范用词

沿用 RFC 2119 用词（MUST / MUST NOT / SHOULD / SHOULD NOT / MAY）。

---

## 二、AgentsIPC 128B 消息头标准

### 2.1 整体布局

消息头为 128 字节定长，采用 `__attribute__((packed))` 无填充紧凑布局（占 2 个 cache line）。布局**永久稳定**（AOS-STD-IPC-001，L1 稳定级）。

```
偏移 0          4    6    8               16              24              32          40          44                                              128
+---------------+----+----+---------------+---------------+---------------+-----------+-----------+--------------------------------------------------+
| magic (4B)    |opcode|flags| trace_id (8B)|timestamp_ns   | src_task (8B) |dst_task   |payload_len| reserved (84B)                                   |
| 0x41524531    | (2B)|(2B)|               | (8B)          |               | (8B)      | (4B)      |                                                  |
| 'ARE1'        |     |    |               |               |               |           |           |                                                  |
+---------------+----+----+---------------+---------------+---------------+-----------+-----------+--------------------------------------------------+
```

### 2.2 C 结构定义

```c
#define AIRY_IPC_HDR_SZ  128u
#define AIRY_IPC_MAGIC         0x41524531u  /* 'A''R''E''1' */

/**
 * struct airy_ipc_msg_hdr - IPC 128 字节定长消息头（SSoT：Layout C）
 *
 * 权威定义见 include/airymax/ipc.h（物理宿主见
 * 50-engineering-standards/120-cross-project-code-sharing.md §Layout C）。
 */
struct airy_ipc_msg_hdr {
    __u32   magic;          /* 偏移 0:  魔数 0x41524531 'ARE1' */
    __u16   opcode;         /* 偏移 4:  SQE/CQE 操作码（SEND/RECV/SEND_BATCH/CANCEL） */
    __u16   flags;          /* 偏移 6:  标志位（NOWAIT/SIGNAL 等） */
    __u64   trace_id;       /* 偏移 8:  链路追踪 ID（OpenTelemetry） */
    __u64   timestamp_ns;   /* 偏移 16: 纳秒时间戳（CLOCK_REALTIME） */
    __u64   src_task;       /* 偏移 24: 源任务 ID（0 表示内核发起） */
    __u64   dst_task;       /* 偏移 32: 目标任务 ID（0 表示广播） */
    __u32   payload_len;    /* 偏移 40: payload 字节数，0 表示无 payload */
    __u8    reserved[84];   /* 偏移 44: 保留字段，填充 0 */
} __attribute__((packed));

_Static_assert(sizeof(struct airy_ipc_msg_hdr) == AIRY_IPC_HDR_SZ,
               "airy_ipc_msg_hdr must be 128 bytes");
```

> **SSoT 声明**：本结构体定义以 `include/airymax/ipc.h`（物理宿主见 `50-engineering-standards/120-cross-project-code-sharing.md` §Layout C）为单一数据源。结构体名 `struct airy_ipc_msg_hdr`，`__attribute__((packed))` 对齐，UAPI 类型 `__u32`/`__u16`/`__u64`/`__u8`，字段顺序 magic/opcode/flags/trace_id/timestamp_ns/src_task/dst_task/payload_len/reserved[84]。本开放标准与 SSoT 逐字节一致。

结构体字节布局**永久稳定**（AOS-STD-IPC-002，L1 稳定级）；新增字段只能写入 `reserved` 区并通过 `flags` 标记启用。

### 2.3 字段语义

| 字段 | 偏移 | 长度 | 类型 | 语义 | 契约约束 |
|------|------|------|------|------|---------|
| magic | 0 | 4 | __u32 LE | 协议识别，**必须**为 `0x41524531` | 永不变更 |
| opcode | 4 | 2 | __u16 LE | SQE/CQE 操作码（SEND/RECV/SEND_BATCH/CANCEL） | 传输层操作码，与 payload 类型分属不同层次 |
| flags | 6 | 2 | __u16 LE | 标志位（NOWAIT/SIGNAL 等） | 未定义位必须为 0 |
| trace_id | 8 | 8 | __u64 LE | OpenTelemetry 链路追踪 ID | 单调递增 + 随机后缀 |
| timestamp_ns | 16 | 8 | __u64 LE | 纳秒时间戳（CLOCK_REALTIME） | 对齐北京时间（UTC+8） |
| src_task | 24 | 8 | __u64 LE | 源任务 ID | 0 表示内核发起，运行时校验真实性 |
| dst_task | 32 | 8 | __u64 LE | 目标任务 ID | 0 表示广播 |
| payload_len | 40 | 4 | __u32 LE | payload 字节数 | 0 ~ 64 MiB |
| reserved | 44 | 84 | __u8[84] | 保留扩展区 | **必须**填充 0 |

所有多字节字段采用**小端序**（AOS-STD-IPC-003，L1 稳定级）。

> **注意**：Layout C 消息头无 `version` 与 `type` 字段。payload 协议类型（REQUEST/RESPONSE/EVENT/STREAM/NOTIFICATION）由 payload 体首字段携带，见第 3 节。

### 2.4 标志位定义

| 标志位 | 值 | 层级 | 语义 | 安全约束 |
|--------|-----|------|------|---------|
| `AIRY_IPC_F_NOWAIT` | 0x0001 | [SC] SSoT | 非阻塞操作（对应 io_uring IOSQE_ASYNC） | 调用方须处理 EAGAIN |
| `AIRY_IPC_F_SIGNAL` | 0x0002 | [SC] SSoT | 完成后投递信号（对应 io_uring IORING_CQE_F_NOTIF） | 信号目标须持有 sig_perm |
| `AIRY_IPC_F_ZEROCOPY` | 0x0004 | [IND] 扩展 | 零拷贝路径 | 发送方必须注册 buffer |
| `AIRY_IPC_F_CAP_CARRY` | 0x0008 | [IND] 扩展 | 携带 capability 令牌 | 内核校验令牌有效性 |
| `AIRY_IPC_F_COMPRESS` | 0x0010 | [IND] 扩展 | payload 已压缩（LZ4） | 接收方自动解压 |
| `AIRY_IPC_F_ENCRYPT` | 0x0020 | [IND] 扩展 | payload 已加密（SM4） | 密钥由 capability 携带 |
| `AIRY_IPC_F_URGENT` | 0x0040 | [IND] 扩展 | 紧急消息，优先投递 | 仅实时调度类可用 |
| `AIRY_IPC_F_NOREPLY` | 0x0080 | [IND] 扩展 | 无需响应（单向通知） | 仅 EVENT/NOTIFICATION 类型可用 |

> **SSoT 对齐说明**：标志位前缀统一为 `AIRY_IPC_F_*`（对齐 SSoT 命名），bit 0-1 由 [SC] SSoT 锁定为 NOWAIT/SIGNAL，扩展标志从 bit 2 开始编号。广播通过 `dst_task = 0` 实现，不需要独立标志位。

未定义的标志位**必须**为 0（AOS-STD-IPC-004，L1 稳定级）；接收方**必须**忽略未知标志位而非报错（前向兼容）。

---

## 三、5 种 payload 类型标准

### 3.1 类型枚举

| type 值 | 名称 | 通信模式 | 可靠性 | 典型用途 |
|---------|------|---------|--------|---------|
| 0x0001 | REQUEST | 请求-响应 | 可靠投递，超时重试 | RPC 调用、服务查询 |
| 0x0002 | RESPONSE | 请求-响应 | 可靠投递 | RPC 响应 |
| 0x0003 | EVENT | 发布-订阅 | 尽力投递，允许丢失 | 状态变更通知 |
| 0x0004 | STREAM | 双向流 | 可靠投递，有序 | LLM token 流、流式数据 |
| 0x0005 | NOTIFICATION | 单向通知 | 尽力投递，无需响应 | 系统广播、心跳 |

枚举值**永久稳定**（AOS-STD-IPC-005，L1 稳定级）；新增类型**必须**追加到 `0x0006` 之后，**不得**修改既有值。

### 3.2 REQUEST payload

```c
typedef struct airy_ipc_request {
    uint64_t request_id;      /* 请求 ID（与 RESPONSE.request_id 配对） */
    uint32_t method_id;       /* 方法 ID（RPC 方法编号） */
    uint32_t timeout_ms;      /* 超时（毫秒），0 表示无超时 */
    uint8_t  params[];        /* 方法参数（柔性数组，最大 1 MiB） */
} airy_ipc_request_t;
```

| 字段 | 约束 |
|------|------|
| `request_id` | 同一 src_task 内**必须**单调递增，不可重复 |
| `method_id` | 由服务端注册表分配，见 01 标准能力声明 |
| `timeout_ms` | 超时后发送方**必须**返回 `AIRY_ETIMEDOUT` |
| `params` | 序列化方式由 method_id 协商（默认 JSON-RPC 2.0） |

### 3.3 RESPONSE payload

```c
typedef struct airy_ipc_response {
    uint64_t request_id;      /* 请求 ID（与 REQUEST 配对） */
    int32_t  status;          /* 状态码（0 成功，<0 AIRY_E* 错误码） */
    uint32_t reserved;        /* 保留字段，填充 0 */
    uint8_t  result[];         /* 结果数据（柔性数组） */
} airy_ipc_response_t;
```

RESPONSE 的 `request_id` **必须**与对应的 REQUEST 配对（AOS-STD-IPC-006，L1 稳定级）；`status` 使用 Airymax 统一错误码体系。

### 3.4 EVENT payload

```c
typedef struct airy_ipc_event {
    uint64_t event_id;        /* 事件 ID（全局唯一） */
    uint32_t topic_id;        /* 主题 ID（订阅时分配） */
    uint32_t priority;        /* 优先级（0-139，对齐调度类） */
    uint8_t  payload[];       /* 事件数据（柔性数组） */
} airy_ipc_event_t;
```

EVENT 为**尽力投递**（AOS-STD-IPC-007），订阅者宕机期间的事件**可以**丢失。

### 3.5 STREAM payload

```c
typedef struct airy_ipc_stream {
    uint64_t stream_id;       /* 流 ID */
    uint32_t seq;             /* 序列号（递增，用于重排序） */
    uint32_t flags;           /* 流标志（FIN/RST/MORE） */
    uint8_t  chunk[];         /* 流数据块（柔性数组） */
} airy_ipc_stream_t;

#define AIRY_IPC_STREAM_FLAG_FIN  0x00000001u  /* 流结束 */
#define AIRY_IPC_STREAM_FLAG_RST  0x00000002u  /* 流重置 */
#define AIRY_IPC_STREAM_FLAG_MORE 0x00000004u  /* 更多数据 */
```

STREAM **必须**保证有序投递（AOS-STD-IPC-008，L1 稳定级）；接收方**必须**按 `seq` 重排序。

### 3.6 NOTIFICATION payload

```c
typedef struct airy_ipc_notification {
    uint64_t notif_id;        /* 通知 ID */
    uint32_t category;       /* 类别（0=系统/1=安全/2=认知/3=运维） */
    uint32_t severity;       /* 严重级别（0=INFO/1=WARN/2=ERROR/3=FATAL） */
    uint8_t  message[];      /* 通知消息（柔性数组） */
} airy_ipc_notification_t;
```

NOTIFICATION **不得**期待响应（AOS-STD-IPC-009，L1 稳定级）。

---

## 四、请求-响应模式标准

### 4.1 模式语义

请求-响应模式（REQUEST + RESPONSE）实现同步 RPC 语义。流程：

1. 发送方生成 `request_id`，发送 REQUEST 消息。
2. 发送方等待 RESPONSE，超时由 `timeout_ms` 控制。
3. 接收方收到 REQUEST 后，根据 `method_id` 调度处理。
4. 接收方返回 RESPONSE，`request_id` 与 REQUEST 配对。
5. 发送方收到 RESPONSE，完成 RPC。

### 4.2 request_id 配对规则

| 规则 | 约束 | 标准条目 |
|------|------|---------|
| 唯一性 | 同一 src_task 内 request_id 单调递增，不可重复 | AOS-STD-IPC-010 |
| 配对 | RESPONSE.request_id **必须**等于 REQUEST.request_id | AOS-STD-IPC-011 |
| 超时 | 超过 timeout_ms 未收到 RESPONSE，发送方返回 ETIMEDOUT | AOS-STD-IPC-012 |
| 重试 | 可靠投递类型可重试，最多 3 次；重试 request_id 不变 | AOS-STD-IPC-013 |
| 丢弃 | 收到未匹配的 RESPONSE **必须**丢弃并记录告警 | AOS-STD-IPC-014 |

### 4.3 防重放保护

REQUEST 类型**必须**启用防重放保护（AOS-STD-IPC-015）：

- 接收方维护 60 秒滑动窗口，记录已处理的 `request_id`。
- 窗口内重复的 `request_id` **必须**拒绝并返回 `AIRY_EBUSY`。
- `timestamp_ns` 超过窗口 60 秒的消息**必须**拒绝。

### 4.4 QoS 保证

| QoS 维度 | 保证内容 | 实现机制 |
|---------|---------|---------|
| 可靠投递 | REQUEST/RESPONSE 保证投递 | 发送方确认 + 超时重传（最多 3 次） |
| 有序投递 | STREAM 保证顺序 | 内核 seq 递增 + 接收方重排序缓冲 |
| 优先级调度 | 高优先级消息优先投递 | 优先级队列 |
| 反压控制 | 接收方缓冲区满时暂停发送 | FLOW_OFF / FLOW_ON 控制消息 |

---

## 五、发布-订阅模式标准

### 5.1 模式语义

发布-订阅模式（EVENT + NOTIFICATION）实现一对多消息分发。流程：

1. 发布方发送 EVENT 消息，`dst_task = 0` 表示广播。
2. 订阅方通过 `topic_id` 订阅感兴趣的事件。
3. 运行时根据订阅表分发 EVENT 到所有订阅方。
4. EVENT 为尽力投递，订阅方宕机期间事件可丢失。

### 5.2 订阅接口

```c
/**
 * airy_ipc_subscribe - 订阅主题
 * @topic_id:    主题 ID
 * @cap:         调用方 capability 令牌（须含 ipc.recv 权限）
 *
 * 返回值:
 *   0:                  成功
 *   -AIRY_EINVAL:   topic_id 非法
 *   -AIRY_EPERM:    权限不足
 */
int airy_ipc_subscribe(uint32_t topic_id, are_cap_t cap);

/**
 * airy_ipc_unsubscribe - 取消订阅
 */
int airy_ipc_unsubscribe(uint32_t topic_id, are_cap_t cap);
```

### 5.3 发布接口

```c
/**
 * airy_ipc_publish - 发布事件
 * @topic_id:    主题 ID
 * @priority:    优先级（0-139）
 * @payload:     事件数据
 * @payload_len: 数据长度
 * @cap:         调用方 capability 令牌（须含 ipc.send 权限）
 *
 * 发布为异步操作，立即返回。运行时按订阅表分发。
 */
int airy_ipc_publish(uint32_t topic_id, uint32_t priority,
                        const uint8_t *payload, size_t payload_len,
                        are_cap_t cap);
```

### 5.4 订阅规则

| 规则 | 约束 | 标准条目 |
|------|------|---------|
| 主题隔离 | 不同 topic_id 的事件互不干扰 | AOS-STD-IPC-016 |
| 权限校验 | 订阅/发布**必须**通过 capability 校验 | AOS-STD-IPC-017 |
| 优先级 | 高优先级事件优先分发 | AOS-STD-IPC-018 |
| 丢失容忍 | 订阅方缓冲区满时**可以**丢弃事件 | AOS-STD-IPC-019 |

---

## 六、标准条目注册表

| 编号 | 名称 | 稳定级 |
|------|------|--------|
| AOS-STD-IPC-001 | 128B 消息头布局 | L1 |
| AOS-STD-IPC-002 | 消息头字节布局稳定 | L1 |
| AOS-STD-IPC-003 | 小端序 | L1 |
| AOS-STD-IPC-004 | 标志位约束 | L1 |
| AOS-STD-IPC-005 | payload 类型枚举 | L1 |
| AOS-STD-IPC-006 | request_id 配对 | L1 |
| AOS-STD-IPC-007 | EVENT 尽力投递 | L1 |
| AOS-STD-IPC-008 | STREAM 有序投递 | L1 |
| AOS-STD-IPC-009 | NOTIFICATION 无响应 | L1 |
| AOS-STD-IPC-010 | request_id 唯一性 | L1 |
| AOS-STD-IPC-011 | request_id 配对约束 | L1 |
| AOS-STD-IPC-012 | 超时返回 ETIMEDOUT | L1 |
| AOS-STD-IPC-013 | 重试上限 3 次 | L2 |
| AOS-STD-IPC-014 | 未匹配 RESPONSE 丢弃 | L2 |
| AOS-STD-IPC-015 | REQUEST 防重放 | L1 |
| AOS-STD-IPC-016 | 主题隔离 | L1 |
| AOS-STD-IPC-017 | 订阅权限校验 | L1 |
| AOS-STD-IPC-018 | 事件优先级分发 | L2 |
| AOS-STD-IPC-019 | EVENT 丢失容忍 | L2 |
| AOS-STD-IPC-020 | magic 值 0x41524531 | L1 |
| AOS-STD-IPC-021 | payload_len 上限 64 MiB | L1 |

---

## 七、Wireshark Dissector 兼容性约定

为支持协议分析工具（Wireshark 等），本标准约定以下 dissector 规则（AOS-STD-IPC-022）：

### 7.1 端口约定

- **TCP/UDP 6060**：AgentsIPC over TCP/UDP（推荐）
- **Unix Domain Socket**：路径 `/var/run/airymax-ipc.sock`
- **io_uring**：内核固定 OP（不暴露端口）

### 7.2 Dissector 伪代码

```c
/* Wireshark dissector 伪代码（Lua 风格） */
function dissect_airymaxispc(buffer, pinfo, tree)
    -- 1. 检查 magic
    local magic = buffer(0, 4):uint()
    if magic ~= 0x41524531 then
        return 0  -- 非 AgentsIPC，交给下一个 dissector
    end

    pinfo.cols.protocol = "AgentsIPC"

    -- 2. 解析消息头
    local subtree = tree:add(buffer(0, 128), "AgentsIPC Header")
    subtree:add(buffer(0, 4),  "Magic: 0x41524531 'ARE1'")
    subtree:add(buffer(4, 2),  "Opcode: " .. opcode_name(buffer(4, 2):uint()))
    subtree:add(buffer(6, 2),  "Flags: 0x" .. string.format("%04x", buffer(6, 2):uint()))
    subtree:add(buffer(8, 8),  "Trace ID: 0x" .. string.format("%016x", buffer(8, 8):uint64():tonumber()))
    subtree:add(buffer(16, 8), "Timestamp: " .. format_ns_time(buffer(16, 8):uint64():tonumber()))
    subtree:add(buffer(24, 8), "Src Task: " .. buffer(24, 8):uint64():tonumber())
    subtree:add(buffer(32, 8), "Dst Task: " .. buffer(32, 8):uint64():tonumber())
    subtree:add(buffer(40, 4), "Payload Length: " .. buffer(40, 4):uint())
    subtree:add(buffer(44, 84), "Reserved")

    -- 3. 解析 payload
    local payload_len = buffer(40, 4):uint()
    if payload_len > 0 then
        local payload_tree = tree:add(buffer(128, payload_len), "Payload")
        -- payload 协议类型由 payload 体首字段携带（Layout C 消息头无 type 字段）
        dissect_payload(buffer, 128, payload_len, payload_tree)
    end

    return 128 + payload_len
end

function opcode_name(o)
    local names = {
        [0x0001] = "SEND",
        [0x0002] = "RECV",
        [0x0003] = "SEND_BATCH",
        [0x0004] = "CANCEL",
    }
    return names[o] or ("UNKNOWN(0x" .. string.format("%04x", o) .. ")")
end

function payload_type_name(t)
    local names = {
        [0x0001] = "REQUEST",
        [0x0002] = "RESPONSE",
        [0x0003] = "EVENT",
        [0x0004] = "STREAM",
        [0x0005] = "NOTIFICATION",
    }
    return names[t] or ("UNKNOWN(0x" .. string.format("%04x", t) .. ")")
end
```

### 7.3 字段显示约定

| 字段 | 显示格式 | 示例 |
|------|---------|------|
| magic | 十六进制 + ASCII | `0x41524531 'ARE1'` |
| opcode | 名称 + 数值 | `SEND (0x0001)` |
| flags | 位展开 | `NOWAIT\|SIGNAL` |
| timestamp_ns | 人类可读时间 | `2026-07-09 14:30:00.123456789` |
| payload 类型（payload 体首字段） | 名称 + 数值 | `REQUEST (0x0001)` |

Dissector 实现**应当**向 Wireshark 官方仓库贡献，作为 AgentsIPC 协议的参考分析工具（AOS-STD-IPC-023）。

---

## 八、安全性要求

### 8.1 进程身份验证

`src_task` **必须**由运行时（内核态或受信用户态）填充，用户态**不得**直接设置（AOS-STD-IPC-024，L1 稳定级）。伪造 `src_task` 的消息**必须**被拒绝。

### 8.2 消息完整性

可选 HMAC-SM3 校验（通过 `AIRY_IPC_F_ENCRYPT` 标志启用）。启用后 payload 末尾追加 32 字节 HMAC-SM3 标签。

### 8.3 资源耗尽防护

| 防护项 | 阈值 | 标准条目 |
|--------|------|---------|
| 单进程消息速率 | 100K msg/s | AOS-STD-IPC-025 |
| 单消息 payload | 64 MiB | AOS-STD-IPC-021 |
| 单进程 ring 注册数 | 16 | AOS-STD-IPC-026 |
| capability 派生深度 | 16 层 | AOS-STD-IPC-027 |
| 默认超时 | 30 s | AOS-STD-IPC-028 |

超过阈值时运行时**必须**返回 `AIRY_EBUSY` 或 `AIRY_ELIMIT`。

---

## 九、错误码

| 错误码 | 数值 | 含义 |
|--------|------|------|
| `AIRY_EOK` | 0 | 成功 |
| `AIRY_EINVAL` | -2 | 参数非法（含 magic 不匹配） |
| `AIRY_EPERM` | -22 | 权限不足 |
| `AIRY_EBUSY` | -16 | 资源忙（含速率超限） |
| `AIRY_ELIMIT` | -7 | 资源上限 |
| `AIRY_ETIMEDOUT` | -1004 | 超时 |
| `AIRY_EMSGSIZE` | -1201 | payload 超过 64 MiB |
| `AIRY_ENOMSG` | -1202 | 无匹配 REQUEST 的 RESPONSE |

---

## 十、兼容性测试要求

| 测试 ID | 测试内容 | 通过条件 |
|---------|---------|---------|
| IPC-T-001 | 消息头字节布局 | 128 字节，字段偏移精确匹配 |
| IPC-T-002 | magic 校验 | 非 0x41524531 的消息被拒绝 |
| IPC-T-003 | 小端序 | x86 与 aarch64 字节布局一致 |
| IPC-T-004 | request_id 配对 | RESPONSE 与 REQUEST 正确配对 |
| IPC-T-005 | 防重放 | 60 秒窗口内重复 request_id 被拒绝 |
| IPC-T-006 | STREAM 有序 | 乱序 chunk 按 seq 重排序 |
| IPC-T-007 | EVENT 丢失容忍 | 订阅方缓冲区满时不崩溃 |
| IPC-T-008 | payload 上限 | 超过 64 MiB 返回 EMSGSIZE |
| IPC-T-009 | src_task 不可伪造 | 用户态设置 src_task 被拒绝 |
| IPC-T-010 | 标志位前向兼容 | 未知标志位不报错 |

---

## 十一、最小示例

### 11.1 发送 REQUEST

```c
struct airy_ipc_msg_hdr hdr = {0};
hdr.magic        = AIRY_IPC_MAGIC;    /* 0x41524531 */
hdr.opcode       = AIRY_IPC_OP_SEND;  /* 传输层操作码 */
hdr.flags        = 0;
hdr.src_task     = my_task_id;
hdr.dst_task     = target_task_id;
hdr.trace_id     = generate_trace_id();
hdr.timestamp_ns = now_ns();

airy_ipc_request_t req = {0};
req.request_id = next_request_id();
req.method_id  = 0x0100;                /* echo 方法 */
req.timeout_ms = 5000;
const char *params = "{\"text\":\"hello\"}";
req.params_len = strlen(params);        /* 概念字段，实际序列化见 method_id 规范 */

uint8_t payload[256];
size_t plen = encode_request(&req, params, payload, sizeof(payload));
hdr.payload_len = (uint32_t)plen;

int rc = airy_ipc_send(hdr, payload, plen);
```

### 11.2 接收 RESPONSE

```c
struct airy_ipc_msg_hdr hdr;
uint8_t payload[4096];
size_t plen = sizeof(payload);
int rc = airy_ipc_recv(&hdr, payload, &plen, 5000 /* timeout ms */);
if (rc == -AIRY_ETIMEDOUT) { /* 超时处理 */ }

airy_ipc_response_t resp;
decode_response(payload, plen, &resp);
assert(resp.request_id == my_request_id);
if (resp.status != 0) { /* 错误处理 */ }
```

---

## 十二、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-07-09 | 初始草案：128B 消息头、5 种 payload 类型、请求-响应/发布-订阅模式、Wireshark dissector 约定 |

---

## 十三、参考文献

- [Airymax 开放标准体系总览](./README.md)
- [Airymax Agent 运行时标准](./01-airy-agent-runtime-standard.md)
- [Airymax 安全模型标准](./03-airy-security-model-standard.md)
- OpenTelemetry：https://opentelemetry.io/
- JSON-RPC 2.0：https://www.jsonrpc.org/specification
- LZ4 压缩：https://github.com/lz4/lz4
- Wireshark 协议开发：https://www.wireshark.org/docs/wsdg_html_chunked/

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
