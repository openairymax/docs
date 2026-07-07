# IPC 协议

> **文档定位**: agentrt-liunx（AirymaxOS） 进程间通信协议的 128B 消息头、5 种 payload、io_uring 零拷贝与同源映射
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **父文档**: [接口设计](README.md)

---

## 1. IPC 协议设计原则

agentrt-liunx IPC 协议与 agentrt AgentsIPC 同源，保留 128 字节定长消息头设计，并在底层升级为基于 io_uring 的零拷贝实现。设计原则：

1. **定长消息头**: 128 字节定长，64 字节对齐，便于 cache line 对齐与零拷贝 page flipping。
2. **同源兼容**: 消息头字段布局与 agentrt AgentsIPC 保持兼容，agentrt 在 agentrt-liunx 上运行无需协议转换。
3. **机制与策略分离**: 协议提供消息传递机制，消息语义（RPC / 事件 / 流）由 payload 类型决定。
4. **零拷贝优先**: 高频路径基于 io_uring + registered buffers + page flipping，避免数据复制。
5. **可观测性**: 消息头携带 `trace_id`（OpenTelemetry）与 `timestamp_ns`，全链路可追踪。
6. **版本协商**: 消息头携带 `version` 字段（当前 0x0100），支持协议演进。
7. **capability 守卫**: 消息可携带 capability 令牌，跨进程传递权限。

---

## 2. 128 字节定长消息头

消息头定义位于头文件 `airymaxos-services/ipc/io-uring-ipc/agentrt_ipc_msg.h`，遵循 4 空格缩进 + Doxygen 注释规范（详见 [04-coding-standard.md](04-coding-standard.md)）。

```c
#ifndef AGENTRT_IPC_MSG_H
#define AGENTRT_IPC_MSG_H

#include <stdint.h>

#define AGENTRT_IPC_MSG_HDR_SIZE 128
#define AGENTRT_IPC_MAGIC 0x41524531u  /* 'A''R''E''1' — 同源 agentrt AgentsIPC */

/* payload 协议类型 */
#define AGENTRT_IPC_TYPE_REQUEST  0x0001
#define AGENTRT_IPC_TYPE_RESPONSE 0x0002
#define AGENTRT_IPC_TYPE_EVENT    0x0003
#define AGENTRT_IPC_TYPE_STREAM   0x0004
#define AGENTRT_IPC_TYPE_CONTROL  0x0005

/* flags 位定义 */
#define AGENTRT_IPC_FLAG_ZEROCOPY  0x00000001u
#define AGENTRT_IPC_FLAG_CAP_CARRY 0x00000002u
#define AGENTRT_IPC_FLAG_COMPRESS  0x00000004u
#define AGENTRT_IPC_FLAG_ENCRYPT   0x00000008u

/**
 * @brief agentrt-liunx IPC 128 字节定长消息头
 *
 * 同源 agentrt AgentsIPC 128B 消息头，字段布局兼容。
 * 64 字节对齐以适配 cache line，便于零拷贝 page flipping。
 */
typedef struct __attribute__((aligned(64))) agentrt_ipc_msg_hdr {
    uint32_t magic;          /* 0x41524531 'ARE1' magic（同源 agentrt） */
    uint16_t version;        /* 协议版本，当前 0x0100 */
    uint16_t type;           /* 消息类型（5 种 payload 协议） */
    uint32_t payload_len;    /* payload 长度（字节） */
    uint32_t flags;          /* 标志位 */
    uint32_t src_pid;        /* 源进程 ID */
    uint32_t dst_pid;        /* 目标进程 ID */
    uint64_t trace_id;       /* 链路追踪 ID（OpenTelemetry） */
    uint64_t timestamp_ns;   /* 纳秒时间戳（CLOCK_REALTIME） */
    uint8_t  reserved[72];   /* 保留字段（对齐到 128B） */
} agentrt_ipc_msg_hdr_t;

static_assert(sizeof(agentrt_ipc_msg_hdr_t) == AGENTRT_IPC_MSG_HDR_SIZE,
              "agentrt_ipc_msg_hdr_t must be 128 bytes");

#endif /* AGENTRT_IPC_MSG_H */
```

### 2.1 字段语义

| 字段 | 偏移 | 长度 | 语义 |
|------|------|------|------|
| `magic` | 0 | 4 | 魔数 `0x41524531`（'ARE1'，同源 agentrt），用于协议识别 |
| `version` | 4 | 2 | 协议版本，当前 `0x0100`（v1.0） |
| `type` | 6 | 2 | payload 协议类型（5 种，见第 3 章） |
| `payload_len` | 8 | 4 | payload 字节数，0 表示无 payload |
| `flags` | 12 | 4 | 标志位（零拷贝 / capability 携带 / 压缩 / 加密） |
| `src_pid` | 16 | 4 | 源进程 ID |
| `dst_pid` | 20 | 4 | 目标进程 ID |
| `trace_id` | 24 | 8 | OpenTelemetry 链路追踪 ID |
| `timestamp_ns` | 32 | 8 | CLOCK_REALTIME 纳秒时间戳（对齐北京时间） |
| `reserved` | 40 | 72 | 保留字段，对齐到 128B，未来扩展 |

### 2.2 字节序

- 消息头统一使用小端序（与 x86_64 / aarch64 默认字节序一致）。
- 跨架构通信（x86_64 ↔ aarch64）通过 `htonl/ntohl` 转换，由 IPC 库自动处理。

### 2.3 对齐与 cache 优化

- `__attribute__((aligned(64)))` 确保 64 字节对齐，与 cache line 对齐。
- 128 字节定长 = 2 个 cache line，单次预取即可加载完整消息头。
- payload 独立于消息头存储，便于零拷贝 page flipping。

---

## 3. 5 种 payload 协议

`type` 字段标识 payload 协议类型，共 5 种：

| type 值 | 协议 | 用途 | 通信模式 |
|---------|------|------|---------|
| 0x0001 | REQUEST | 请求-响应的请求方 | 同步 RPC |
| 0x0002 | RESPONSE | 请求-响应的响应方 | 同步 RPC |
| 0x0003 | EVENT | 事件通知 | 发布-订阅 |
| 0x0004 | STREAM | 流式数据 | 双向流 |
| 0x0005 | CONTROL | 控制消息 | 链路管理 |

### 3.1 REQUEST payload

```c
/**
 * @brief REQUEST payload（请求-响应的请求方）
 */
typedef struct agentrt_ipc_request {
    uint64_t request_id;      /* 请求 ID（与 RESPONSE.request_id 对应） */
    uint32_t method_id;       /* 方法 ID（RPC 方法编号） */
    uint32_t timeout_ms;      /* 超时（毫秒） */
    uint8_t  params[];        /* 方法参数（柔性数组） */
} agentrt_ipc_request_t;
```

### 3.2 RESPONSE payload

```c
/**
 * @brief RESPONSE payload（请求-响应的响应方）
 */
typedef struct agentrt_ipc_response {
    uint64_t request_id;      /* 请求 ID（与 REQUEST.request_id 对应） */
    int32_t  status;          /* 状态码（0 成功，<0 AGENTRT_E* 错误码） */
    uint32_t reserved;        /* 保留字段 */
    uint8_t  result[];        /* 结果数据（柔性数组） */
} agentrt_ipc_response_t;
```

### 3.3 EVENT payload

```c
/**
 * @brief EVENT payload（事件通知，发布-订阅）
 */
typedef struct agentrt_ipc_event {
    uint64_t event_id;       /* 事件 ID */
    uint32_t topic_id;       /* 主题 ID（订阅时分配） */
    uint32_t priority;       /* 事件优先级（0-139） */
    uint8_t  payload[];      /* 事件数据（柔性数组） */
} agentrt_ipc_event_t;
```

### 3.4 STREAM payload

```c
/**
 * @brief STREAM payload（流式数据，双向流）
 */
typedef struct agentrt_ipc_stream {
    uint64_t stream_id;      /* 流 ID */
    uint32_t seq;             /* 序列号（递增） */
    uint32_t flags;           /* 流标志（FIN / RST / MORE） */
    uint8_t  chunk[];        /* 流数据块（柔性数组） */
} agentrt_ipc_stream_t;

#define AGENTRT_IPC_STREAM_FLAG_FIN  0x00000001u
#define AGENTRT_IPC_STREAM_FLAG_RST  0x00000002u
#define AGENTRT_IPC_STREAM_FLAG_MORE 0x00000004u
```

### 3.5 CONTROL payload

```c
/**
 * @brief CONTROL payload（控制消息，链路管理）
 */
typedef struct agentrt_ipc_control {
    uint32_t opcode;         /* 控制操作码 */
    uint32_t arg;            /* 操作参数 */
    uint8_t  data[];         /* 附加数据（柔性数组） */
} agentrt_ipc_control_t;

/* 控制操作码 */
#define AGENTRT_IPC_CTRL_HELLO     0x0001  /* 握手 */
#define AGENTRT_IPC_CTRL_BYE       0x0002  /* 关闭 */
#define AGENTRT_IPC_CTRL_PING      0x0003  /* 心跳 */
#define AGENTRT_IPC_CTRL_PONG      0x0004  /* 心跳响应 */
#define AGENTRT_IPC_CTRL_FLOW_OFF  0x0005  /* 流控：暂停 */
#define AGENTRT_IPC_CTRL_FLOW_ON   0x0006  /* 流控：恢复 */
```

---

## 4. io_uring 零拷贝实现

agentrt-liunx IPC 基于 Linux 6.6 内核基线的 io_uring 子系统实现零拷贝消息传递。

### 4.1 通信原语

参考 Zircon 消息传递模型，提供 4 种通信原语（详见 [20-modules/02-services.md](../20-modules/02-services.md) 第 4.5 节）：

| 原语 | 语义 | 用途 |
|------|------|------|
| Channel | 双向、面向消息 | RPC 调用（REQUEST/RESPONSE） |
| Socket | 双向、面向流 | 流式数据（STREAM） |
| FIFO | 单向、面向消息 | 高吞吐单向通信（EVENT） |
| Eventpair | 事件同步 | 跨进程信号量（CONTROL） |

### 4.2 零拷贝路径

```c
/**
 * @brief io_uring 零拷贝发送示例
 *
 * 发送方注册 page 为 io_uring buffer，接收方通过 page flipping 获取引用。
 */
static int agentrt_ipc_send_zerocopy(int ring_fd,
                                     const agentrt_ipc_msg_hdr_t *hdr,
                                     const void *payload, size_t payload_len)
{
    struct io_uring_sqe sqe;
    struct iovec iov[2];

    /* 注册 header 与 payload 为 iovec */
    iov[0].iov_base = (void *)hdr;
    iov[0].iov_len = AGENTRT_IPC_MSG_HDR_SIZE;
    iov[1].iov_base = (void *)payload;
    iov[1].iov_len = payload_len;

    /* 提交 IORING_OP_IPC_SEND（agentrt-liunx 注册的固定 OP） */
    io_uring_prep_rw(&sqe, IORING_OP_IPC_SEND, ring_fd,
                     iov, 2, 0);
    sqe.flags = IOSQE_FIXED_FILE;

    return io_uring_submit_and_wait(ring_fd, 1);
}
```

### 4.3 跨进程 ring 共享

```c
/**
 * @brief 注册 io_uring ring 给目标进程（跨进程 ring 共享）
 *
 * 通过 io_uring_register 将 ring fd 注册给 dst_pid，实现零拷贝通道。
 */
AGENTRT_API int agentrt_sys_ipc_register_ring(int ring_fd, uint32_t dst_pid);
```

### 4.4 固定 OP 扩展

agentrt-liunx 在 io_uring 注册以下 IPC 专用 OP（详见 [20-modules/01-kernel.md](../20-modules/01-kernel.md) 第 4.2 节）：

| OP | 用途 |
|----|------|
| `IORING_OP_IPC_SEND` | 零拷贝发送消息 |
| `IORING_OP_IPC_RECV` | 零拷贝接收消息 |
| `IORING_OP_IPC_REGISTER_RING` | 注册跨进程 ring |
| `IORING_OP_IPC_DEREGISTER_RING` | 注销跨进程 ring |

### 4.5 零拷贝机制

- **registered buffers**: 发送方注册 page 为 io_uring buffer，内核持有 page 引用。
- **page flipping**: 接收方通过 `IORING_OP_IPC_RECV` 接收 page 引用，内核仅翻转 page table entry，不拷贝数据。
- **MSG_ZEROCOPY**: 网络路径使用 `MSG_ZEROCOPY` 减少协议栈拷贝。

### 4.6 capability 携带与权限校验

当消息头 `flags` 中置位 `AGENTRT_IPC_FLAG_CAP_CARRY` 时，消息可携带 capability 令牌跨进程传递，遵循 seL4 风格的不可伪造语义：

```c
/**
 * @brief 携带 capability 令牌发送 IPC 消息
 *
 * capability 令牌由内核生成（不可伪造），随消息传递至 dst_pid，
 * 接收方在 cap_slot_out 中获得派生 capability（受限权限）。
 *
 * @param hdr        128B 消息头（flags 须置 CAP_CARRY）
 * @param payload    消息负载
 * @param cap_slot   待传递的 capability 句柄
 * @param cap_slot_out 接收方 capability 句柄输出
 * @return 0 成功，<0 失败（AGENTRT_EPERM 表示令牌无效）
 *
 * @since 1.0.1
 */
AGENTRT_API int agentrt_sys_ipc_send_with_cap(const agentrt_ipc_msg_hdr_t *hdr,
                                              const void *payload,
                                              uint32_t cap_slot,
                                              uint32_t *cap_slot_out);
```

**校验流程**:

1. 发送方在 `flags` 中置位 `AGENTRT_IPC_FLAG_CAP_CARRY`。
2. 内核校验 `cap_slot` 是否属于 `src_pid`，若不属于则返回 `AGENTRT_EPERM`。
3. 内核派生受限 capability 至 `dst_pid`，写入 `cap_slot_out`。
4. 接收方使用派生令牌执行受保护操作（如 `file.read`）。

**派生规则**: 派生 capability 的权限不得超过原令牌（最小权限原则），支持递归撤销（详见 [20-modules/03-security.md](../20-modules/03-security.md) 第 4.1 节 capability 系统）。

---

## 5. 与 agentrt AgentsIPC 的同源映射

agentrt-liunx IPC 与 agentrt AgentsIPC 同源，保留 128B 定长消息头布局兼容。

| 维度 | agentrt AgentsIPC | agentrt-liunx IPC | 同源关系 |
|------|------------------|---------------|---------|
| 消息头长度 | 128B 定长 | 128B 定长 | 完全兼容 |
| magic | 'ARE1' | 'ARE1' | 完全兼容（同源 agentrt） |
| version | 0x0100 | 0x0100 | 完全兼容 |
| src_pid / dst_pid | 进程 ID | 进程 ID | 完全兼容 |
| trace_id | OpenTelemetry | OpenTelemetry | 完全兼容 |
| timestamp_ns | CLOCK_REALTIME | CLOCK_REALTIME | 完全兼容 |
| 底层传输 | 用户态消息队列 | io_uring 零拷贝 | agentrt-liunx 升级 |
| payload 协议 | 自定义 | 5 种（REQUEST/RESPONSE/EVENT/STREAM/CONTROL） | agentrt-liunx 扩展 |
| capability | 应用权限模型 | seL4 风格 capability | agentrt-liunx 升级 |

**同源红利**: agentrt 在 agentrt-liunx 上运行时，IPC 消息头布局完全兼容，无需协议转换层，天然契合。

**独立性**: agentrt-liunx IPC 为 OS 级实现（基于 io_uring + 内核固定 OP），agentrt AgentsIPC 为应用级实现（用户态消息队列），二者在分层上独立。遵循 IRON-9"同源且部分代码共享"原则。

---

## 6. IPC 性能约束

IPC 性能约束对齐非功能性需求 NFR-P-002（详见 [00-requirements/03-non-functional-requirements.md](../00-requirements/03-non-functional-requirements.md)）。

### 6.1 NFR-P-002 IPC 吞吐

| 约束 ID | 指标 | 阈值 | 测量方法 |
|---------|------|------|---------|
| NFR-P-002 | IPC 吞吐 | > 100K msg/s（单核） | 128B 消息往返吞吐 |
| NFR-P-002a | IPC 延迟 | < 10 μs（P99） | `agentrt_sys_ipc_send` + `agentrt_sys_ipc_recv` 往返 |
| NFR-P-002b | 大消息吞吐 | > 10 Gbps（1MB 消息） | 零拷贝 page flipping |
| NFR-P-002c | 跨节点 IPC | > 1M msg/s（4 节点） | 超节点 OS 跨节点 IPC |

### 6.2 性能优化手段

- **io_uring 零拷贝**: 消除数据复制，单核可达 100K+ msg/s。
- **cache line 对齐**: 128B 消息头 = 2 cache line，单次预取加载完整消息头。
- **batch 提交**: io_uring 支持批量提交 SQE，减少 syscall 次数。
- **registered buffers**: 预注册 buffer 池，避免每次发送注册开销。
- **CXL 跨节点共享**: 跨节点 IPC 通过 CXL 3.0 内存池化共享 ring，减少网络往返。

### 6.3 性能回归保护

- 每次提交运行 `airymaxos-tests/benchmark/ipc-latency` 与 `ipc-throughput` 微基准。
- 与基线对比，吞吐退化 > 5% 或延迟退化 > 10% 自动打回（详见 [20-modules/08-tests.md](../20-modules/08-tests.md) 第 4.6 节）。
- Soak Test 72 小时持续 IPC 负载，验证无内存泄漏、无性能衰减。

---

## 7. 相关文档

- [接口设计](README.md)
- [系统调用接口](01-syscalls.md)
- [SDK API](03-sdk-api.md)
- [编码规范](04-coding-standard.md)
- [内核设计](../20-modules/01-kernel.md)
- [服务设计](../20-modules/02-services.md)
- [非功能性需求](../00-requirements/03-non-functional-requirements.md)（NFR-P-002）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
