Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# IPC 协议
> **文档定位**：agentrt-linux（AirymaxOS） 进程间通信协议的 Layout C v4 128B 消息头、5 种 payload、io_uring 零拷贝与同源映射、Capability Folding Badge 模型\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[agentrt-linux 设计文档](README.md)

---

## 1. IPC 协议设计原则

agentrt-linux IPC 协议与 agentrt AgentsIPC 同源，保留 128 字节定长消息头设计，并在底层升级为基于 io_uring 的零拷贝实现。v1.1 起，集成 **Capability Folding** 设计模式——将 capability check 从独立控制面操作"折叠"到 IPC 数据面 fastpath 中，由 [SC] `ipc.h` Layout C v4 消息头 offset 40-47 的 `capability_badge` 字段承载 64-bit Native Word Badge（D-9 修复后 8 字节对齐至 offset 40），由 fastpath C-S9 内联校验。设计原则：

1. **定长消息头**: 128 字节定长，自然对齐布局（Layout C v4 SSoT，`__attribute__((aligned(64)))`，D-9 修复移除 `__attribute__((packed))`），128 字节 = 2 cache lines，便于零拷贝 IORING_OP_URING_CMD + registered buffer。参考 OLK 6.6 内核协议头（`struct ethhdr`/`struct iphdr`）手动安排字段对齐的做法，不使用 packed。
2. **同源兼容**: 消息头字段布局与 agentrt AgentsIPC 保持兼容，agentrt 在 agentrt-linux 上运行无需协议转换。
3. **机制与策略分离**: 协议提供消息传递机制，消息语义（RPC / 事件 / 流）由 payload 类型决定。
4. **零拷贝优先**: 高频路径基于 io_uring + registered buffers + IORING_OP_URING_CMD 语义映射，避免数据复制。
5. **可观测性**: 消息头携带 `trace_id`（OpenTelemetry）与 `timestamp_ns`，全链路可追踪。
6. **版本协商**: 版本协商在 L2 服务协议层处理（见 `50-engineering-standards/30-runtime-interfaces/runtime_interfaces.md` Part II），L1 [SC] 基础消息头不含 `version` 字段，通过 `opcode` 区分消息类型、`reserved[72]` 预留扩展空间。
7. **Capability Folding**: `capability_badge` 字段（offset 40-47，D-9 修复后 8 字节对齐）承载 64-bit Native Word Badge（Epoch + Random Tag + Perms），fastpath C-S9 内联校验（~10ns），IPC 数据传递即能力校验，无双平面、无独立 capability syscall。
8. **完整性校验**: `crc32` 字段（offset 52-55）覆盖 `header[0:52) + payload`，C-S12 在投递前校验，防 Ring 数据损坏。

---

## 2. 128 字节定长消息头（Layout C v4）

消息头定义位于 [SC] 共享契约层头文件 `include/uapi/linux/airymax/ipc.h`（SSoT 物理宿主见 [120-cross-project-code-sharing.md §2.7](../50-engineering-standards/120-cross-project-code-sharing.md)），遵循 **Tab 8 缩进**（对齐 OLK-6.6 §1 + SSoT）+ Doxygen 注释规范（详见 [04-coding-standard.md](04-coding-standard.md)）。Layout C v4 相对 v1 新增 `capability_badge`（offset 40，D-9 修复后 8 字节对齐）与 `crc32`（offset 52）字段，`reserved` 收缩至 72 字节。

```c
#ifndef AIRYMAX_IPC_H
#define AIRYMAX_IPC_H

#include <airymax/types.h>  /* __u32, __u64, __u8 */

#ifdef __cplusplus
extern "C" {
#endif

/* ===== Layout C v4 — 128 字节定长消息头 ===== */
/* magic: 0x41524531 'ARE1' ——一经发布即冻结（120 号 §7.5）*/

#define AIRY_IPC_MAGIC		0x41524531u  /* 'ARE1' (Airymax Runtime Engine v1) */
#define AIRY_IPC_HDR_SIZE	128

/* opcode 定义（见 §4） */
#define AIRY_IPC_OP_SEND		0x0001
#define AIRY_IPC_OP_RECV		0x0002  /* 仅用于接收方声明的 expected opcode */
#define AIRY_IPC_OP_SEND_BATCH		0x0003
#define AIRY_IPC_OP_CANCEL		0x0004
#define AIRY_IPC_OP_FREEZE		0x0005  /* A-ULS 触发 ring 冻结 */
#define AIRY_IPC_OP_CAP_REQUEST	0x0010  /* 自举：请求 Badge，无 Badge */
#define AIRY_IPC_OP_CAP_RESPONSE	0x0011  /* sec_d 返回编译好的 Badge */

/* flags 定义 */
#define AIRY_IPC_F_ZEROCOPY		0x0001
#define AIRY_IPC_F_CAP_CARRY		0x0002  /* 携带 Badge（agentrt-linux 内核必置） */
#define AIRY_IPC_F_ENCRYPT		0x0004  /* 保留，0.1.1 不启用 */
#define AIRY_IPC_F_COMPRESS		0x0008  /* 保留，0.1.1 不启用 */
#define AIRY_IPC_F_BATCH_TAIL		0x0010  /* SEND_BATCH 的最后一个 SQE */
#define AIRY_IPC_F_RESERVED		0xFFE0  /* 必须为零 */

/* payload 协议类型（payload 体首字段携带，非消息头字段） */
#define AIRY_IPC_TYPE_REQUEST		0x0001
#define AIRY_IPC_TYPE_RESPONSE		0x0002
#define AIRY_IPC_TYPE_EVENT		0x0003
#define AIRY_IPC_TYPE_STREAM		0x0004
#define AIRY_IPC_TYPE_CONTROL		0x0005

/**
 * struct airy_ipc_msg_hdr - IPC 128 字节定长消息头（SSoT：Layout C v4）
 *
 * 权威定义见 include/uapi/linux/airymax/ipc.h（物理宿主见
 * 50-engineering-standards/120-cross-project-code-sharing.md §2.7）。
 *
 * 对齐设计（D-9 修复，对齐 OLK 6.6 内核协议头规范）:
 *   - 不使用 __attribute__((packed))，参考 OLK 6.6 内核协议头（struct ethhdr /
 *     struct iphdr / struct udphdr）手动安排字段自然对齐的做法
 *   - 所有 __u64 字段对齐到 8 字节边界，__u32 对齐到 4 字节边界，__u16 对齐到 2 字节边界
 *   - 128B = 2 cache lines（x86_64/aarch64 64B/line），struct 整体 __attribute__((aligned(64)))
 *   - 热字段（magic ~ crc32）集中在 cache line 1（offset 0-55），reserved 填充至 cache line 2 尾
 *   - capability_badge 从 v0.1.1 的 offset 44 前移至 offset 40，恢复 8 字节自然对齐
 *
 * Field offsets (Layout C v4, D-9 alignment-fixed):
 *   offset  0: magic              (4B) 'ARE1' 0x41524531
 *   offset  4: opcode             (2B) AIRY_IPC_OP_*
 *   offset  6: flags              (2B) AIRY_IPC_F_*
 *   offset  8: trace_id           (8B) OpenTelemetry trace ID
 *   offset 16: timestamp_ns       (8B) CLOCK_REALTIME ns
 *   offset 24: src_task           (8B) source Agent/daemon task_id
 *   offset 32: dst_task           (8B) destination Agent/daemon task_id
 *   offset 40: capability_badge   (8B) [SS] agentrt-linux uses, agentrt=0
 *   offset 48: payload_len        (4B) payload length (excluding header)
 *   offset 52: crc32              (4B) CRC32 over header[0:52) + payload
 *   offset 56: reserved[72]       (72B) must be zero ([56:64) CL1 tail + [64:128) CL2)
 *
 * CRC32 覆盖范围: header[0:52) + payload（capability_badge 含在内，未变）
 */
struct airy_ipc_msg_hdr {
	__u32	magic;			/* offset  0, 'ARE1' 0x41524531 */
	__u16	opcode;			/* offset  4, see AIRY_IPC_OP_* */
	__u16	flags;			/* offset  6, see AIRY_IPC_F_* */
	__u64	trace_id;		/* offset  8, OpenTelemetry trace ID */
	__u64	timestamp_ns;		/* offset 16, CLOCK_REALTIME nanoseconds */
	__u64	src_task;		/* offset 24, source Agent/daemon task_id */
	__u64	dst_task;		/* offset 32, destination Agent/daemon task_id */
	__u64	capability_badge;	/* offset 40, [SS] agentrt-linux uses, agentrt=0 */
	__u32	payload_len;		/* offset 48, payload length (excluding header) */
	__u32	crc32;			/* offset 52, CRC32 over header[0:52) + payload */
	__u8	reserved[72];		/* offset 56, must be zero (CL1 [56:64) + CL2 [64:128)) */
} __attribute__((aligned(64)));

_Static_assert(sizeof(struct airy_ipc_msg_hdr) == AIRY_IPC_HDR_SIZE,
	"airy_ipc_msg_hdr must be exactly 128 bytes");
_Static_assert(offsetof(struct airy_ipc_msg_hdr, capability_badge) == 40,
	"capability_badge must be at offset 40 (8-byte aligned, D-9 fix)");
_Static_assert(offsetof(struct airy_ipc_msg_hdr, crc32) == 52,
	"crc32 must be at offset 52");

/* ===== Badge 位布局（仅 [SS] 语义同源，定义在内核实现侧）===== */
/* agentrt 用户态：capability_badge 始终为 0（H3） */
/* agentrt-linux 内核：capability_badge 由 sec_d 编译（H4） */
/* [DSL] 降级模式：capability_badge = 0，跳过 C-S9（H6） */

#define AIRY_BADGE_EPOCH_SHIFT		48
#define AIRY_BADGE_EPOCH_MASK		0xFFFF000000000000ULL
#define AIRY_BADGE_RANDTAG_SHIFT	16
#define AIRY_BADGE_RANDTAG_MASK		0x0000FFFFFFFF0000ULL
#define AIRY_BADGE_PERMS_SHIFT		0
#define AIRY_BADGE_PERMS_MASK		0x000000000000FFFFULL

/* Badge 提取宏（仅内核侧使用，[SS] 语义同源层） */
#define AIRY_BADGE_EPOCH(b)	(((b) & AIRY_BADGE_EPOCH_MASK)   >> AIRY_BADGE_EPOCH_SHIFT)
#define AIRY_BADGE_RANDTAG(b)	(((b) & AIRY_BADGE_RANDTAG_MASK) >> AIRY_BADGE_RANDTAG_SHIFT)
#define AIRY_BADGE_PERMS(b)	(((b) & AIRY_BADGE_PERMS_MASK)   >> AIRY_BADGE_PERMS_SHIFT)

/* Badge 编译宏（仅 sec_d 使用） */
#define AIRY_BADGE_COMPILE(epoch, randtag, perms) \
	((((__u64)(epoch)   & 0xFFFF) << AIRY_BADGE_EPOCH_SHIFT)   | \
	 (((__u64)(randtag) & 0xFFFFFFFF) << AIRY_BADGE_RANDTAG_SHIFT) | \
	 (((__u64)(perms)   & 0xFFFF) << AIRY_BADGE_PERMS_SHIFT))

/* Capability 权限位（低 16 位） */
#define AIRY_CAP_PERM_SEND	0x0001
#define AIRY_CAP_PERM_RECV	0x0002
#define AIRY_CAP_PERM_CALL	0x0004  /* airy_sys_call 权限 */
#define AIRY_CAP_PERM_GRANT	0x0008  /* 派生能力（mint/derive） */
#define AIRY_CAP_PERM_REVOKE	0x0010  /* 撤销能力（仅 sec_d） */
#define AIRY_CAP_PERM_FREEZE	0x0020  /* 冻结 ring */
#define AIRY_CAP_PERM_BATCH	0x0040  /* 批量发送 */
#define AIRY_CAP_PERM_RESERVED	0xFF80  /* 必须为零 */

#ifdef __cplusplus
}
#endif

#endif /* AIRYMAX_IPC_H */
```

> **SSoT 声明**：本结构体定义以 `include/uapi/linux/airymax/ipc.h`（物理宿主见 [120-cross-project-code-sharing.md §2.7](../50-engineering-standards/120-cross-project-code-sharing.md)）为单一数据源。结构体名 `struct airy_ipc_msg_hdr`，`__attribute__((aligned(64)))` 对齐（D-9 修复后移除 `__attribute__((packed))`，参考 OLK 6.6 `struct ethhdr`/`struct iphdr` 手动安排字段自然对齐），UAPI 类型 `__u32`/`__u16`/`__u64`/`__u8`，字段顺序 magic/opcode/flags/trace_id/timestamp_ns/src_task/dst_task/capability_badge/payload_len/crc32/reserved[72]（capability_badge 前移至 offset 40 恢复 8 字节自然对齐）。本协议主文档与 SSoT 逐字节一致，**缩进风格对齐 OLK-6.6 §1（Tab 8）**。
>
> **[SC] / [SS] 边界声明（H2）**：本头文件中 `struct airy_ipc_msg_hdr` 数据结构、字段偏移与大小、magic 值、opcode 枚举、flags 位定义、Badge 位布局宏、Capability 权限位、`_Static_assert(sizeof == 128)` 属于 [SC] 共享契约层；`airy_cap_badge_ok()` 内联函数、`agent_caps[]` 静态数组、`airy_cap_global_epoch` atomic_t、Random Tag 生成、C-S9 校验逻辑、sec_d Badge 编译流程属于 [SS] 语义同源层（不在本头文件，定义在内核实现侧，权威源为 [07-ipc-fastpath.md §5.2](07-ipc-fastpath.md) 与 [03-capability-model.md §3](../110-security/03-capability-model.md)）。

### 2.1 字段语义

| 字段 | 偏移 | 长度 | 语义 | 校验位置 |
|------|------|------|------|---------|
| `magic` | 0 | 4 | 魔数 `0x41524531`（'ARE1'），用于协议识别 | C-S1 |
| `opcode` | 4 | 2 | SQE/CQE 操作码（见 `AIRY_IPC_OP_*`，如 SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE） | C-S2 |
| `flags` | 6 | 2 | 标志位（ZEROCOPY/CAP_CARRY/ENCRYPT/COMPRESS/BATCH_TAIL） | C-S10 |
| `trace_id` | 8 | 8 | OpenTelemetry 链路追踪 ID | 不校验（透传） |
| `timestamp_ns` | 16 | 8 | 纳秒时间戳（CLOCK_REALTIME，对齐北京时间） | 不校验（透传） |
| `src_task` | 24 | 8 | 源任务 ID（0 表示内核发起），与 Badge 编译时绑定 | C-S9（与 `agent_caps[]` 对照） |
| `dst_task` | 32 | 8 | 目标任务 ID（0 表示广播），决定 kfifo 路由 | C-S5/C-S6（路由） |
| `capability_badge` | 40 | 8 | [SS] 语义同源：agentrt-linux 内核由 sec_d 编译，agentrt 用户态始终为 0（D-9 修复后 8 字节对齐至 offset 40） | C-S9（Badge 校验） |
| `payload_len` | 48 | 4 | payload 字节数，0 表示无 payload | C-S3/C-S4 |
| `crc32` | 52 | 4 | CRC32 覆盖 `header[0:52) + payload`，发送方计算填入 | C-S12（投递前校验） |
| `reserved` | 56 | 72 | 保留字段，必须为零，未来扩展（[56:64) 为 cache line 1 尾部，[64:128) 为 cache line 2） | C-S4（reserved 检查） |

> **注意**：Layout C v4 消息头无 `version` 与 `type` 字段。`opcode` 是传输层 SQE/CQE 操作码（SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE），与 payload 协议类型（REQUEST/RESPONSE/EVENT/STREAM/CONTROL）不同——后者下沉至 payload 体首字段携带（见第 3 章）。
>
> **v1.1 变更说明**：相对 v0.1.1，Layout C v4 在 offset 40 新增 `capability_badge`（8B）字段承载 Capability Folding Badge；在 offset 52 新增 `crc32`（4B）字段保证消息完整性；`reserved` 字段从 84 字节收缩至 72 字节；新增 opcode `FREEZE`(0x0005)/`CAP_REQUEST`(0x0010)/`CAP_RESPONSE`(0x0011)；新增 flags `AIRY_IPC_F_BATCH_TAIL`(0x0010) 与 `AIRY_IPC_F_RESERVED`(0xFFE0)。原 `AIRY_IPC_F_NOWAIT/SIGNAL` 已废弃（NOWAIT 由 io_uring `IOSQE_ASYNC` 替代，SIGNAL 由 CQE 通知替代）。
>
> **D-9 对齐修复说明（v1.1 同期）**：v0.1.1 草案中 `capability_badge` 原位于 offset 44（紧跟 `payload_len` 之后），但 `__u64` 字段未 8 字节对齐，依赖 `__attribute__((packed))` 强制紧凑布局。v1.1 正式版参考 OLK 6.6 内核协议头（`struct ethhdr`/`struct iphdr`/`struct udphdr`）不使用 packed 的惯例，将 `capability_badge` 前移至 offset 40（紧跟 `dst_task` 之后），恢复 8 字节自然对齐；`payload_len` 顺移至 offset 48；`crc32` 保持 offset 52 不变；CRC32 覆盖范围 `header[0:52) + payload` 不变。struct 整体改用 `__attribute__((aligned(64)))` 确保 2 cache line 对齐。

### 2.2 字节序

- 消息头统一使用小端序（与 x86_64 / aarch64 默认字节序一致）。
- 跨架构通信（x86_64 ↔ aarch64）通过 `htonl/ntohl` 转换，由 IPC 库自动处理。

### 2.3 对齐与 cache 优化

- **不使用 `__attribute__((packed))`**（D-9 修复）：参考 OLK 6.6 内核协议头（`struct ethhdr`/`struct iphdr`/`struct udphdr`）手动安排字段自然对齐的惯例，所有 `__u64` 字段对齐到 8 字节边界，`__u32` 对齐到 4 字节边界，`__u16` 对齐到 2 字节边界。`packed` 会导致非对齐访问（x86 性能惩罚、ARM 对齐故障），且破坏 cache line 对齐。
- **`__attribute__((aligned(64)))`** 确保 struct 整体对齐到 cache line（64B），128 字节定长 = 2 个 cache line，单次预取即可加载完整消息头。
- **热字段集中**：`magic` ~ `crc32`（offset 0-55）集中在 cache line 1（offset 0-63），fastpath 仅需 1 次 cache line 加载；`reserved`（offset 56-127）横跨 cache line 1 尾部与 cache line 2，未来扩展不破坏热路径。
- payload 独立于消息头存储，便于零拷贝 registered buffer + mmap 共享页。

### 2.4 `timestamp_ns` 时钟基准设计权衡（CLOCK_REALTIME vs CLOCK_MONOTONIC）

`timestamp_ns` 字段（offset 16）使用 **CLOCK_REALTIME** 作为时钟基准，对齐北京时间（UTC+8）。本节记录该选择的工程权衡，避免未来误改。

**设计决策**：IPC 消息头 `timestamp_ns` 使用 `CLOCK_REALTIME`，调度数据流 vtime 使用 `__s32` Q16.16 定点（不依赖 wall clock）。

**权衡分析**：

| 维度 | CLOCK_REALTIME（已选） | CLOCK_MONOTONIC（备选） |
|------|----------------------|----------------------|
| 语义 | 挂钟时间（wall clock），可被 NTP/settimeofday 调整 | 单调递增，不受系统时间调整影响 |
| 跨主机对齐 | ✅ 支持（agent 分布式部署场景关键） | ❌ 各主机启动基准不同，无法跨机对齐 |
| 跨时区可读性 | ✅ 配合 UTC+8 偏移可读为北京时间 | ❌ 仅"自启动以来纳秒"，无挂钟含义 |
| 链路追踪对齐 | ✅ OpenTelemetry `trace_id` 时间戳默认 CLOCK_REALTIME，可对齐 | ❌ 无法与外部系统日志对齐 |
| NTP 跳变风险 | ⚠️ NTP 渐变或跳变可导致时间戳非单调 | ✅ 单调保证 |
| 适用场景 | 跨主机分布式追踪、可观测性、审计 | 单机内调度顺序、超时检测 |

**取舍依据**：
1. agentrt-linux 部署形态为分布式 agent 集群，IPC 消息在多机间传递，`timestamp_ns` 必须跨主机对齐以支持全链路追踪。
2. 调度顺序与超时检测由 [SC] `airy_vtime_t`（Q16.16 定点）独立承担，不依赖 `timestamp_ns`，避免时钟跳变影响调度正确性。
3. OpenTelemetry 协议默认 CLOCK_REALTIME，保持兼容性。
4. NTP 跳变风险通过 NTP 渐变（slewing，默认 NTP 守护进程行为）缓解，避免跳跃式调整。

**禁止误改**：`timestamp_ns` 不应改用 `CLOCK_MONOTONIC`，否则将破坏跨主机分布式追踪能力。如需单机单调时间戳，应在 payload 协议中扩展独立字段（不污染 [SC] 消息头）。

### 2.5 `crc32` 覆盖范围

`crc32` 字段（offset 52）覆盖 `header[0:52) + payload`，由发送方计算填入，由 fastpath C-S12 在投递前校验。设计目的：检测 Ring 数据损坏（bit flip / page corruption），触发 `AIRY_FAULT_RING_CORRUPT`(0x1003) Fault。

```
CRC32 覆盖范围: header[0:52) + payload（D-9 修复后字段顺序调整，覆盖范围不变）
  ├─ magic              [0:4)    ✓
  ├─ opcode             [4:6)    ✓
  ├─ flags              [6:8)    ✓
  ├─ trace_id           [8:16)   ✓
  ├─ timestamp_ns       [16:24)  ✓
  ├─ src_task           [24:32)  ✓
  ├─ dst_task           [32:40)  ✓
  ├─ capability_badge   [40:48)  ✓  ← 包含 Badge（D-9 修复后 8 字节对齐至 offset 40）
  ├─ payload_len        [48:52)  ✓
  ├─ crc32              [52:56)  ✗  ← 不覆盖自身（循环依赖）
  └─ reserved[72]       [56:128) ✗  ← 必须为零，无覆盖价值（CL1 [56:64) + CL2 [64:128)）
```

**关键不变量**：
- CRC32 自身不覆盖自身（循环依赖）。
- `reserved[72]` 必须为零，C-S4 检查其全零，无 CRC32 覆盖价值。
- CRC32 包含 `capability_badge`，防 Badge 字段被篡改。

### 2.6 Capability Folding Badge 模型

`capability_badge` 字段（offset 40-47，D-9 修复后 8 字节对齐）是 v1.1 引入的核心字段，承载 64-bit Native Word Badge，是 Capability Folding 设计模式的物理载体。

**工程定义**：Capability Folding 是将 capability check 从独立控制面操作"折叠"到 IPC 数据面 fastpath 中的设计模式。其核心论断是"IPC 不是承载能力校验——IPC 就是能力校验"。

**物理载体**：
- 字段位置：`struct airy_ipc_msg_hdr` offset 40-47，8 字节 `__u64`（D-9 修复后 8 字节自然对齐）
- 位布局：`(Epoch << 48) | (RandomTag << 16) | Perms`
  - `Epoch`（bits 48-63，16 位）：全局代际快照，由 `airy_cap_global_epoch` atomic_t 维护，1 行 `atomic_inc` 即可全局撤销
  - `RandomTag`（bits 16-47，32 位）：随机标签，由 sec_d 在编译 Badge 时通过 `get_random_bytes()` 生成，防伪造
  - `Perms`（bits 0-15，16 位）：权限位，由 `AIRY_CAP_PERM_*` 宏定义（SEND/RECV/CALL/GRANT/REVOKE/FREEZE/BATCH）

**执行点**：fastpath C-S9 内联校验 `airy_cap_badge_ok()`，~10ns，3 个 `READ_ONCE` + 位运算 + 比较（详见 [07-ipc-fastpath.md §5.2](07-ipc-fastpath.md)）。

**6 条硬约束（H1-H6）**：

```
H1: Layout C v4 总长保持 128 字节，magic 保持 0x41524531 'ARE1'
H2: capability_badge 字段进入 [SC] ipc.h，但校验机制属于 [SS] 层
H3: agentrt 用户态 capability_badge 始终为 0，使用传统 cap_t 引用
H4: agentrt-linux 内核 capability_badge 由 sec_d 编译，fastpath C-S9 校验
H5: 纯 C LSM 模块职责不变（策略/撤销/审计/执法），Badge 校验是其 fastpath 内联
H6: [DSL] 降级模式下 capability_badge 字段存在但被忽略（值=0，跳过校验）
```

**Badge 生命周期**：

| 阶段 | 操作 | 执行者 | 物理动作 |
|------|------|--------|---------|
| 编译 | `AIRY_BADGE_COMPILE(epoch, randtag, perms)` | sec_d（唯一写者） | 写入 `agent_caps[src_task]` |
| 携带 | 发送方在 `capability_badge` 字段填入 Badge | 发送方 fastpath | 填入消息头 offset 40 |
| 校验 | C-S9 内联 `airy_cap_badge_ok()` | 接收方 fastpath | 3 个 `READ_ONCE` + 位运算 |
| 撤销 | `airy_cap_epoch_increment()` | sec_d（特权操作） | `atomic_inc(&airy_cap_global_epoch)` |
| 自举 | `AIRY_IPC_OP_CAP_REQUEST` opcode | 任意 Agent | 无 Badge 即可请求 |
| 降级 | `AIRY_SC_FALLBACK` 编译开关 | 构建系统 | `capability_badge=0`，跳过 C-S9 |

**自举机制**（消除 NULL Badge 硬编码）：当 Agent 首次启动需要 Badge 时，发送 `opcode=CAP_REQUEST` 的 IPC 消息（`capability_badge=0`、`flags` 不置 `CAP_CARRY`），fastpath C-S9 检测到此 opcode 直接通过，由 sec_d 编译好 Badge 后通过 `opcode=CAP_RESPONSE` 返回。

**与原 cap_t 模型的关系**：
- agentrt 用户态继续使用 `cap_t` 引用（H3），`capability_badge` 字段始终为 0，不影响其语义。
- agentrt-linux 内核使用 Badge 模型（H4），fastpath C-S9 内联校验，无需独立 capability syscall。
- [DSL] 降级模式下 Badge 字段被忽略（H6），`capability_badge=0`，跳过 C-S9，退化到 `airy_cap_check()` slowpath 兜底路径（基于 `agent_caps[1024]` 静态数组；`AIRY_ECAP_RADIX_MISS`/`STATIC_KEY` 错误码保留编号但 v1.1 不触发）。

**Badge 错误码**（错误码 SSoT 见 [08-sc-error-contract.md §2.4](08-sc-error-contract.md)）：

| 错误码 | 值 | 触发条件 |
|--------|---|---------|
| `AIRY_ECAP_BADGE` | -78 | Badge 格式无效、Random Tag 不匹配、`CAP_CARRY` 置位但 `badge=0` 且 `opcode != CAP_REQUEST` |
| `AIRY_ECAP_EPOCH` | -79 | Badge Epoch 与全局 Epoch 不匹配（已撤销或过期） |
| `AIRY_ECAP_FORGED` | -80 | Badge 伪造尝试（同时触发 `AIRY_FAULT_CAP_FORGED` 0x1001） |
| `AIRY_ECAP_PERM` | -81 | 权限位不满足 opcode 所需 |
| `AIRY_ECAP_FROZEN` | -82 | Ring 已冻结（C-S0 检查，A-ULS 通过 `FREEZE` opcode 设置） |

**详细安全模型**：见 [03-capability-model.md §3](../110-security/03-capability-model.md)。

---

## 3. 5 种 payload 协议

> **字段说明**：Layout C v4 128B 消息头（见第 2 章）无 `type` 字段。下列 5 种 payload 协议类型（REQUEST/RESPONSE/EVENT/STREAM/CONTROL）由 payload 体首字段携带，而非消息头 `opcode`。`opcode` 是传输层 SQE/CQE 操作码（SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE），二者分属不同层次。

payload 协议类型共 5 种：

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
typedef struct airy_ipc_request {
    uint64_t request_id;      /* 请求 ID（与 RESPONSE.request_id 对应） */
    uint32_t method_id;       /* 方法 ID（RPC 方法编号） */
    uint32_t timeout_ms;      /* 超时（毫秒） */
    uint8_t  params[];        /* 方法参数（柔性数组） */
} airy_ipc_request_t;
```

### 3.2 RESPONSE payload

```c
/**
 * @brief RESPONSE payload（请求-响应的响应方）
 */
typedef struct airy_ipc_response {
    uint64_t request_id;      /* 请求 ID（与 REQUEST.request_id 对应） */
    int32_t  status;          /* 状态码（0 成功，<0 AIRY_E* 错误码） */
    uint32_t reserved;        /* 保留字段 */
    uint8_t  result[];        /* 结果数据（柔性数组） */
} airy_ipc_response_t;
```

### 3.3 EVENT payload

```c
/**
 * @brief EVENT payload（事件通知，发布-订阅）
 */
typedef struct airy_ipc_event {
    uint64_t event_id;       /* 事件 ID */
    uint32_t topic_id;       /* 主题 ID（订阅时分配） */
    uint32_t priority;       /* 事件优先级（0-139） */
    uint8_t  payload[];      /* 事件数据（柔性数组） */
} airy_ipc_event_t;
```

### 3.4 STREAM payload

```c
/**
 * @brief STREAM payload（流式数据，双向流）
 */
typedef struct airy_ipc_stream {
    uint64_t stream_id;      /* 流 ID */
    uint32_t seq;             /* 序列号（递增） */
    uint32_t flags;           /* 流标志（FIN / RST / MORE） */
    uint8_t  chunk[];        /* 流数据块（柔性数组） */
} airy_ipc_stream_t;

#define AIRY_IPC_STREAM_FLAG_FIN  0x00000001u
#define AIRY_IPC_STREAM_FLAG_RST  0x00000002u
#define AIRY_IPC_STREAM_FLAG_MORE 0x00000004u
```

### 3.5 CONTROL payload

```c
/**
 * @brief CONTROL payload（控制消息，链路管理）
 */
typedef struct airy_ipc_control {
    uint32_t opcode;         /* 控制操作码 */
    uint32_t arg;            /* 操作参数 */
    uint8_t  data[];         /* 附加数据（柔性数组） */
} airy_ipc_control_t;

/* 控制操作码 */
#define AIRY_IPC_CTRL_HELLO     0x0001  /* 握手 */
#define AIRY_IPC_CTRL_BYE       0x0002  /* 关闭 */
#define AIRY_IPC_CTRL_PING      0x0003  /* 心跳 */
#define AIRY_IPC_CTRL_PONG      0x0004  /* 心跳响应 */
#define AIRY_IPC_CTRL_FLOW_OFF  0x0005  /* 流控：暂停 */
#define AIRY_IPC_CTRL_FLOW_ON   0x0006  /* 流控：恢复 */
```

---

## 4. io_uring 零拷贝实现

agentrt-linux IPC 基于 Linux 6.6 内核基线的 io_uring 子系统实现零拷贝消息传递。

### 4.1 通信原语

参考 seL4 消息传递模型（Endpoint/Notification，ADR-014），提供 4 种通信原语（详见 [20-modules/02-services.md](../20-modules/02-services.md) 第 4.5 节）：

| 原语 | 语义 | 用途 |
|------|------|------|
| Channel | 双向、面向消息 | RPC 调用（REQUEST/RESPONSE） |
| Socket | 双向、面向流 | 流式数据（STREAM） |
| FIFO | 单向、面向消息 | 高吞吐单向通信（EVENT） |
| Eventpair | 事件同步 | 跨进程信号量（CONTROL） |

### 4.2 零拷贝路径

```c
/**
 * @brief io_uring 零拷贝发送示例（Capability Folding 单平面架构）
 *
 * 通过 IORING_OP_URING_CMD 复用，cmd_op=AIRY_URING_CMD_IPC_SEND 区分操作。
 * 发送方注册 page 为 io_uring buffer，接收方通过 registered buffer + mmap 共享页获取引用。
 */
static int airy_ipc_send_zerocopy(int ring_fd,
                                     const struct airy_ipc_msg_hdr *hdr,
                                     const void *payload, size_t payload_len)
{
    struct io_uring_sqe sqe;
    struct iovec iov[2];

    /* 注册 header 与 payload 为 iovec */
    iov[0].iov_base = (void *)hdr;
    iov[0].iov_len = AIRY_IPC_HDR_SIZE;	/* 128 */
    iov[1].iov_base = (void *)payload;
    iov[1].iov_len = payload_len;

    /* 提交 IORING_OP_URING_CMD（cmd_op=AIRY_URING_CMD_IPC_SEND） */
    io_uring_prep_uring_cmd(&sqe, ring_fd,
                            AIRY_URING_CMD_IPC_SEND, 0);
    sqe.addr = (unsigned long)iov;
    sqe.len = 2;
    sqe.flags = IOSQE_FIXED_FILE;

    return io_uring_submit_and_wait(ring_fd, 1);
}
```

### 4.3 跨进程 ring 共享

```c
/**
 * @brief 注册 io_uring ring 给目标进程（跨进程 ring 共享）
 *
 * 通过 io_uring_register 将 ring fd 注册给 dst_task，实现零拷贝通道。
 */
AIRY_API int airy_sys_ipc_register_ring(int ring_fd, __u64 dst_task);
```

### 4.4 io_uring OP 复用（Capability Folding 单平面架构）

agentrt-linux 不注册独立的 IPC 专用 io_uring OP，全部 IPC 数据传递通过 `IORING_OP_URING_CMD` 复用，由 `cmd_op` 字段区分操作类型（详见 [20-modules/01-kernel.md](../20-modules/01-kernel.md) 第 4.2 节）：

| `IORING_OP_URING_CMD` 的 `cmd_op` | 用途 | 对应消息头 opcode |
|----|------|----|
| `AIRY_URING_CMD_IPC_SEND` | 零拷贝发送消息 | `AIRY_IPC_OP_SEND` |
| `AIRY_URING_CMD_IPC_SEND_BATCH` | 批量发送（SQE 链） | `AIRY_IPC_OP_SEND_BATCH` |
| `AIRY_URING_CMD_IPC_CANCEL` | 取消在途消息 | `AIRY_IPC_OP_CANCEL` |
| `AIRY_URING_CMD_IPC_FREEZE` | 冻结 ring（A-ULS 触发） | `AIRY_IPC_OP_FREEZE` |
| `AIRY_URING_CMD_IPC_CAP_REQUEST` | 自举请求 Badge | `AIRY_IPC_OP_CAP_REQUEST` |

接收方通过 poll CQE 获取消息，不使用独立 OP。

### 4.5 零拷贝机制

- **registered buffers**: 发送方注册 page 为 io_uring buffer，内核持有 page 引用。
- **registered buffer**: 接收方通过 `IORING_OP_URING_CMD` 接收 registered buffer 引用，内核通过 IORING_OP_URING_CMD 语义映射传递 registered buffer 引用，不拷贝数据。
- **MSG_ZEROCOPY**: 网络路径使用 `MSG_ZEROCOPY` 减少协议栈拷贝。

### 4.6 Capability Folding：Badge 校验

agentrt-linux 内核态 IPC 通过 `capability_badge` 字段承载 64-bit Native Word Badge（详见 §2.6），由 fastpath C-S9 内联校验，无需独立 capability syscall。**v1.1 起，原 `airy_sys_ipc_send_with_cap()` 已废弃，Badge 模型取代 cap_slot 派生模型。**

**Capability Folding 校验流程**（详见 [07-ipc-fastpath.md §5.2](07-ipc-fastpath.md)）:

1. **发送方**：在 `capability_badge` 字段填入 sec_d 编译的 Badge，`flags` 置位 `AIRY_IPC_F_CAP_CARRY`（agentrt-linux 内核必置，agentrt 用户态不置）。
2. **fastpath C-S9 内联校验**（接收方，~10ns）：
   - 提取 `badge_epoch = AIRY_BADGE_EPOCH(badge)`
   - 提取 `badge_randtag = AIRY_BADGE_RANDTAG(badge)`
   - 提取 `badge_perms = AIRY_BADGE_PERMS(badge)`
   - `READ_ONCE(agent_caps[src_task].randtag)` 比对 → 不匹配返回 `AIRY_ECAP_FORGED`(-80) 同时触发 `AIRY_FAULT_CAP_FORGED`(0x1001)
   - 比对 `badge_epoch == airy_cap_global_epoch` → 不匹配返回 `AIRY_ECAP_EPOCH`(-79)
   - 比对 `badge_perms & required == required` → 不满足返回 `AIRY_ECAP_PERM`(-81)
3. **接收方**：使用通过校验的 Badge 权限执行受保护操作。

**自举场景**：当 `opcode == AIRY_IPC_OP_CAP_REQUEST` 时，C-S9 跳过 Badge 校验（无 Badge 即可请求 Badge），由 sec_d 编译好后通过 `AIRY_IPC_OP_CAP_RESPONSE` 返回。

**降级场景**：当 `AIRY_SC_FALLBACK` 编译开关启用时，C-S9 跳过 Badge 校验，`capability_badge=0`，退化到 `airy_cap_check()` slowpath 兜底路径（基于 `agent_caps[1024]` 静态数组；详见 [11-degraded-survival-layer.md §2.2](../10-architecture/11-degraded-survival-layer.md)）。

**Badge 安全模型**：详见 [03-capability-model.md §3](../110-security/03-capability-model.md)。

### 4.7 端点状态机（seL4 风格，Step 2.4 #2 补齐）

参考 seL4 Endpoint 状态机（ADR-014，源码证据 `include/object/structures.h:36-41`：`EPState_Idle/EPState_Send/EPState_Recv`），agentrt-linux IPC 端点有 3 个状态，由内核维护，用户态不可直接观测。**命名严格对齐 seL4**：状态名描述"端点所处的语义态"而非"线程等待态"，因此不带 `Waiting` 后缀。

| 状态 | 含义 | 触发转换 |
|------|------|----------|
| **Idle** | 无消息等待，端点空闲 | Send→Send；Recv→Recv |
| **Send** | 端点处于发送态：发送方已就绪并阻塞，等待接收方 Recv | Recv→匹配发送方，双方就绪，转 Idle |
| **Recv** | 端点处于接收态：接收方已就绪并阻塞，等待发送方 Send | Send→匹配接收方，双方就绪，转 Idle |

```
状态转换图：
  Idle --Send--> Send --Recv--> Idle（消息传递完成）
  Idle --Recv--> Recv --Send--> Idle（消息传递完成）
```

**Fastpath 优化**（借鉴 seL4）：当端点处于 Idle 且发送方与接收方同时就绪时，走 Fastpath——内核直接切换上下文，不经过 SQE/CQE 队列，延迟 < 1μs。Slowpath（阻塞情况）走 io_uring SQE/CQE 标准路径。

**与 seL4 的差异**：seL4 端点状态机是微内核核心（形式化验证）；agentrt-linux 在 io_uring 之上实现端点状态语义，Fastpath 用用户态调度器直接调度（SCHED_FIFO），Slowpath 用 io_uring CQE 通知。状态机语义在 [SS] 语义同源层与 agentrt 共享，实现独立。

### 4.8 Reply 原子性（seL4 `doReplyTransfer` 临界区串行执行模式）

参考 seL4 Reply 语义（ADR-014，源码证据 `src/object/endpoint.c:115-127` 的 `doReplyTransfer` + `src/kernel/thread.c:131-200` 的线程状态切换）：**seL4 不存在独立的"Reply Capability 原子性"机制**，原子性是"线程状态切换 + 回复消息传递"两步在 `doReplyTransfer` 函数内串行执行的涌现属性。agentrt-linux 借鉴该模式：`AIRY_SYS_REPLY` 在内核临界区内串行完成"写入回复 + 恢复接收方"，要么完全成功（接收方收到回复并恢复运行），要么完全失败（回复方阻塞等待接收方就绪），保证 IPC 调用-回复的语义完整性，不会出现"回复已发送但接收方未恢复"的中间状态。

**seL4 原子性来源（源码级分析）**：
1. seL4 master 模式：`doReplyTransfer()` 内部先 `setThreadState(blocked, ts_idle)` 再 `sendIPC()`，调用方 caller slot 由 `setupCallerCap` 在 Call 阶段绑定到 TCB 内嵌的 `tcbCaller`（`src/object/endpoint.c:115-127` + `src/kernel/thread.c:131-200`），整个调用链在单核上不可抢占，因此天然串行。
2. seL4 MCS 模式：通过 `reply_t` 对象 + `reply_push/reply_remove` 显式管理回复链，仍非显式 "atomic reply" 系统调用。
3. **关键澄清**：seL4 没有 "Reply Capability 原子性" 独立设计，原子性是临界区串行执行的涌现属性。

**agentrt-linux 原子性保证机制**（借鉴 `doReplyTransfer` 模式，实现独立）：
1. Reply 操作在内核态持端点锁（`airy_ep->lock`）期间完成——对齐 seL4 单核不可抢占语义。
2. 内核将回复消息写入接收方地址空间，并恢复接收方上下文——两步在同一临界区内串行完成，对齐 seL4 `doReplyTransfer` 的"setThreadState + sendIPC"两步串行模式。
3. 若接收方未在 Recv 状态（端点未处于接收态），回复方阻塞（端点转 Send 态），等待接收方 Recv。

```c
/* Reply 原子性：借鉴 seL4 doReplyTransfer 临界区串行执行模式 */
static int airy_ipc_reply(struct airy_endpoint *ep,
                          const struct airy_ipc_msg_hdr *reply,
                          const void *payload, size_t len)
{
    spin_lock(&ep->lock);                    /* 进入临界区 */
    if (ep->state != AIRY_EP_RECV) {         /* 端点不在 Recv 态 */
        spin_unlock(&ep->lock);
        return -AIRY_EAGAIN;                 /* 接收方未就绪，回复方阻塞 */
    }
    /* 串行两步：写入回复 + 恢复接收方上下文（对齐 doReplyTransfer 两步模式） */
    airy_ipc_copy_reply(ep->recv_task, reply, payload, len);
    airy_task_wake(ep->recv_task);           /* 恢复接收方 */
    ep->state = AIRY_EP_IDLE;                /* 端点回 Idle */
    spin_unlock(&ep->lock);                  /* 退出临界区 */
    return 0;
}
```

**与 seL4 的差异**：seL4 Reply 原子性通过"TCB 内嵌 caller slot + `doReplyTransfer` 临界区串行 + 形式化验证"三重保证；agentrt-linux 通过"自旋锁 + 临界区串行"保证（非形式化验证），借鉴 `doReplyTransfer` 模式但实现独立（M1 暂不引入 TCB 内嵌 reply slot，1.0.1 M2+ 评估 C-C07）。Reply 原子性语义在 [SS] 语义同源层与 agentrt 共享。

### 4.9 SQE128 模式（cmd 字段扩展，D-7 修复）

**背景**：OLK 6.6 `include/uapi/linux/io_uring.h` 中 `struct io_uring_sqe` 的 `cmd` 字段标准大小为 16 字节（`__u8 cmd[16]`），不足以承载 agentrt-linux IPC 的 `airy_ipc_cmd` 结构体（含消息头指针 + payload 指针 + 路由信息，~40 字节）。v1.1 方案早期文档错误声称可直接传递 80 字节 `airy_ipc_cmd`，未说明需要启用 SQE128 模式。D-7 修复对此进行文档化。

**SQE128 模式**（`IORING_SETUP_SQE128`，Linux 5.18+，OLK 6.6 基线支持）：

| 维度 | 标准 SQE | SQE128 模式 |
|------|---------|------------|
| SQE 总大小 | 64 字节 | 128 字节（64B 标准 + 64B 扩展） |
| `cmd` 字段大小 | 16 字节（`__u8 cmd[16]`） | 80 字节（标准 16B + 扩展 64B） |
| `io_uring_setup()` flags | 不置 `IORING_SETUP_SQE128` | 必须置位 `IORING_SETUP_SQE128` |
| SQ ring 内存 | `64 × nr_entries` | `128 × nr_entries` |
| 适用场景 | `cmd` ≤ 16 字节的轻量命令 | `cmd` > 16 字节的复合命令（如 A-IPC `airy_ipc_cmd`） |

**启用要求**：

1. `io_uring_setup()` 时 `flags` 参数必须置位 `IORING_SETUP_SQE128`（OS-IPC-009 已更新，详见 [40-dataflows/03-ipc-flow.md](../40-dataflows/03-ipc-flow.md) §2.5）。
2. 用户态提交 SQE 时必须填充完整的 128 字节（标准 64B + 扩展 64B），扩展区 `cmd[16:80)` 由 `airy_ipc_cmd` 覆盖。
3. `airy_ipc_cmd` 结构体大小必须 ≤ 80 字节（SQE128 的 `cmd` 字段大小），通过 `BUILD_BUG_ON(sizeof(struct airy_ipc_cmd) <= 80)` 编译期校验。

**`airy_ipc_cmd` 结构体约束**（≤ 80 字节）：

```c
/**
 * struct airy_ipc_cmd - io_uring SQE128 cmd 字段承载的 IPC 命令数据
 *
 * 通过 IORING_OP_URING_CMD + SQE128 模式传递，cmd_op 区分操作类型。
 * 大小必须 <= 80 字节（SQE128 cmd 字段上限）。
 */
struct airy_ipc_cmd {
    struct airy_ipc_msg_hdr *hdr;       /* 消息头指针，offset  0, 8 字节 */
    const void              *payload;   /* payload 指针，offset  8, 8 字节 */
    size_t                   payload_len; /* payload 长度，offset 16, 8 字节 */
    __u64                    dst_task;  /* 目标任务，offset 24, 8 字节 */
    __u32                    ring_id;   /* ring ID，offset 32, 4 字节 */
    __u32                    reserved;  /* 保留字段，offset 36, 4 字节 */
    /* 共 40 字节，远小于 80 字节上限（SQE128 cmd 字段大小） */
};
BUILD_BUG_ON(sizeof(struct airy_ipc_cmd) > 80);
```

**OLK 6.6 内核侧访问**：

- `struct io_uring_cmd` 的 `pdu[32]` 是 32 字节内联数据区，仅存储 `cmd` 的前 32 字节副本（OLK 6.6 标准 `include/linux/io_uring.h`）。
- `io_uring_cmd_to_pdu()` 宏用于快速访问前 32 字节，fastpath 路径仅依赖前 32 字节时使用。
- 完整的 80 字节 `cmd` 数据需通过 `ioucmd->sqe->cmd` 访问原始 SQE 区域（含扩展 64 字节）。
- fastpath 校验与投递详见 [07-ipc-fastpath.md §5.2](07-ipc-fastpath.md) + §5.7。

**对齐说明**：

- SQE128 模式是 OLK 6.6 标准 io_uring 特性（Linux 5.18+ 引入），不引入任何内核 patch，符合"不修改 OLK 6.6 内核源码"原则（OS-ARCH-001）。
- SQE128 启用后 SQ ring 内存占用翻倍（64B → 128B/SQE），但 IPC fastpath 收益（消除 cmd 数据截断、支持完整 `airy_ipc_cmd` 内联传递）远大于内存开销。
- SQE128 与 `SQPOLL` / `DEFER_TASKRUN` / `NO_SQARRAY` / `NO_IEC` 等 flag 正交，可同时启用。

---

## 5. 与 agentrt AgentsIPC 的同源映射

agentrt-linux IPC 与 agentrt AgentsIPC 同源，保留 128B 定长消息头布局兼容。

| 维度 | agentrt AgentsIPC | agentrt-linux IPC | 同源关系 |
|------|------------------|---------------|---------|
| 消息头长度 | 128B 定长 | 128B 定长 | 完全兼容 |
| magic | 'ARE1' | 'ARE1' | 完全兼容（同源 agentrt） |
| src_task / dst_task | 任务 ID | 任务 ID | 完全兼容 |
| trace_id | OpenTelemetry | OpenTelemetry | 完全兼容 |
| timestamp_ns | CLOCK_REALTIME | CLOCK_REALTIME | 完全兼容 |
| 底层传输 | 用户态消息队列 | io_uring 零拷贝 | agentrt-linux 升级 |
| payload 协议 | 自定义 | 5 种（REQUEST/RESPONSE/EVENT/STREAM/CONTROL） | agentrt-linux 扩展 |
| capability | 应用权限模型 | seL4 风格 capability | agentrt-linux 升级 |

**同源红利**: agentrt 在 agentrt-linux 上运行时，IPC 消息头布局完全兼容，无需协议转换层，天然契合。

**独立性**: agentrt-linux IPC 为 OS 级实现（基于 io_uring + 内核固定 OP），agentrt AgentsIPC 为应用级实现（用户态消息队列），二者在分层上独立。遵循 IRON-9"同源且部分代码共享"原则。

---

## 6. IPC 性能约束

IPC 性能约束对齐非功能性需求 NFR-P-002（详见 [00-requirements/03-non-functional-requirements.md](../00-requirements/03-non-functional-requirements.md)）。

### 6.1 NFR-P-002 IPC 吞吐

| 约束 ID | 指标 | 阈值 | 测量方法 |
|---------|------|------|---------|
| NFR-P-002 | IPC 吞吐 | > 100K msg/s（单核） | 128B 消息往返吞吐 |
| NFR-P-002a | IPC 延迟 | < 10 μs（P99） | `IORING_OP_URING_CMD + cmd_op=IPC_SEND` + CQE poll 往返 |
| NFR-P-002b | 大消息吞吐 | > 10 Gbps（1MB 消息） | 零拷贝 IORING_OP_URING_CMD + registered buffer |
| NFR-P-002c | 跨节点 IPC | > 1M msg/s（4 节点） | 超节点 OS 跨节点 IPC |

### 6.2 性能优化手段

- **io_uring 零拷贝**: 消除数据复制，单核可达 100K+ msg/s。
- **cache line 对齐**: 128B 消息头 = 2 cache line，单次预取加载完整消息头。
- **batch 提交**: io_uring 支持批量提交 SQE，减少 syscall 次数。
- **registered buffers**: 预注册 buffer 池，避免每次发送注册开销。
- **CXL 跨节点共享**: 跨节点 IPC 通过 CXL 3.0 内存池化共享 ring，减少网络往返。

### 6.3 性能回归保护

- 每次提交运行 `tests-linux/benchmark/ipc-latency` 与 `ipc-throughput` 微基准。
- 与基线对比，吞吐退化 > 5% 或延迟退化 > 10% 自动打回（详见 [20-modules/08-tests-linux.md](../20-modules/08-tests-linux.md) 第 4.6 节）。
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

## 8. IRON-9 v3 四层共享模型

> **OS-IFACE-003**： IPC 协议遵循 IRON-9 v3 四层共享模型——Layout C v4 128B 消息头结构、magic 0x41524531（'ARE1'）、`capability_badge` 位布局、5 种 payload type 通过 [SC] 共享契约层 `ipc.h` 完全共享；io_uring `cmd_op` 与传输实现各自独立。禁止在 agentrt 与 agentrt-linux 之间引入协议转换层或字节序适配层。Capability Folding 校验机制属于 [SS] 语义同源层，[DSL] 降级模式下 `capability_badge=0` 跳过 C-S9。

### 8.1 四层模型概览

| 层次 | 共享程度 | 本接口涉及内容 |
|------|---------|---------------|
| **[SC] 共享契约层** | 完全共享代码 | `ipc.h`（IPC magic 0x41524531 'ARE1' + `struct airy_ipc_msg_hdr` Layout C v4 128B 定长头含 `capability_badge`/`crc32` + 7 种 opcode + flags 位 + Badge 位布局宏 + Capability 权限位 + 5 种 payload type） |
| **[SS] 语义同源层** | 操作模式同源（注册/匹配/生命周期/Capability Folding 校验等概念一致），函数签名因抽象层级不同而独立 | agentrt AgentsIPC（`atoms/ipc`）↔ agentrt-linux `io-uring-ipc`（services）的 send/recv/register_ring 同源 API；C-S9 Badge 校验逻辑同源 |
| **[IND] 完全独立层** | 完全独立 | agentrt POSIX MQ + mmap 传输 + `cap_t` 引用 ↔ agentrt-linux io_uring + SQPOLL + IORING_OP_URING_CMD + registered buffer 零拷贝传输 + Badge 模型 |
| **[DSL] 降级生存层** | 最小生存子集 | `#ifdef AIRY_SC_FALLBACK` 降级块：`capability_badge=0`、跳过 C-S9、退化到 `airy_cap_check()` slowpath 兜底路径（基于 `agent_caps[1024]` 静态数组；`AIRY_ECAP_RADIX_MISS`/`STATIC_KEY` 保留编号但 v1.1 不触发） |

### 8.2 [SC] 共享契约层——`ipc.h` 在 IPC 协议中的角色

| `ipc.h` 定义项 | 在协议中的角色 | 消费方 |
|---------------|---------------|--------|
| `AIRY_IPC_MAGIC` 0x41524531 'ARE1' | 协议识别魔数，消息头首 4 字节 | agentrt AgentsIPC / agentrt-linux io-uring-ipc |
| `struct airy_ipc_msg_hdr` 128B 定长头（Layout C v4） | 自然对齐消息头结构（D-9 修复后移除 packed，使用 `__attribute__((aligned(64)))`，字段顺序 magic/opcode/flags/trace_id/timestamp_ns/src_task/dst_task/capability_badge/payload_len/crc32/reserved[72]） | send/recv 路径 |
| `AIRY_IPC_OP_*` opcode | SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE 操作码 | io_uring `cmd_op` 路由 |
| `AIRY_IPC_F_*` flags 位 | ZEROCOPY/CAP_CARRY/ENCRYPT/COMPRESS/BATCH_TAIL 标志位 | 消息处理 |
| `AIRY_IPC_TYPE_*` 5 种 payload | REQUEST/RESPONSE/EVENT/STREAM/CONTROL 类型枚举 | payload 解码 |
| `AIRY_BADGE_*` 位布局宏 | Epoch/RandomTag/Perms 提取与编译宏 | Badge 校验与编译 |
| `AIRY_CAP_PERM_*` 权限位 | SEND/RECV/CALL/GRANT/REVOKE/FREEZE/BATCH 权限位 | C-S9 权限校验 |

### 8.3 [SS] 语义同源层——agentrt ↔ agentrt-linux IPC API 映射

| agentrt AgentsIPC（用户态） | agentrt-linux io-uring-ipc（内核态） | 同源签名 | 实现差异 |
|----------------------------|--------------------------------------|---------|---------|
| `airy_ipc_send()` | `airy_sys_ipc_send()` → `IORING_OP_URING_CMD + cmd_op=IPC_SEND` | `(const struct airy_ipc_msg_hdr *, const void *) -> int` | 用户态 POSIX MQ vs 内核 io_uring SQE |
| `airy_ipc_recv()` | `airy_sys_ipc_recv()` → CQE poll | `(struct airy_ipc_msg_hdr *, void *, size_t) -> int` | 用户态 mq_receive vs 内核 io_uring CQE |
| `airy_ipc_register_ring()` | `airy_sys_ipc_register_ring()` | `(int ring_fd, __u64 dst_task) -> int` | 用户态 mmap 共享 vs 内核 io_uring_register |
| `airy_cap_badge_ok()`（v1.1 新增） | fastpath C-S9 内联校验 | `(badge, src_task, required_perms) -> bool` | agentrt 用户态始终返回 true（badge=0，H3）；agentrt-linux 内核完整校验（H4） |

> **v1.1 变更说明**：原 `airy_ipc_send_with_cap()` / `airy_sys_ipc_send_with_cap()` 已废弃。Capability Folding 模式下，Badge 直接嵌入消息头 `capability_badge` 字段，无需独立 cap_slot 参数。agentrt 用户态 `capability_badge=0`（H3），不调用 `airy_cap_badge_ok()`；agentrt-linux 内核 fastpath C-S9 内联校验（H4）。

### 8.4 [IND] 完全独立层

| 独立项 | agentrt 实现 | agentrt-linux 实现 | 独立原因 |
|--------|-------------|-------------------|---------|
| 传输后端 | POSIX MQ + mmap | io_uring + SQPOLL + registered buffers | 内核态零拷贝优势 |
| 零拷贝机制 | 用户态 page 共享（mmap） | IORING_OP_URING_CMD（registered buffer 引用传递） | 内核态性能 |
| 跨进程 ring | 用户态 fd 共享（SCM_RIGHTS） | `io_uring_register` 跨进程注册 | 内核态机制 |
| 字节序处理 | `htonl/ntohl` 库函数 | 内核 `cpu_to_le32/le32_to_cpu` | 工具链差异 |
| Capability 模型 | `cap_t` 引用（应用权限模型） | 64-bit Native Word Badge（Capability Folding） | 内核态 fastpath 内联校验 |

### 8.5 [DSL] 降级生存层

`ipc.h` 在 `#ifdef AIRY_SC_FALLBACK` 降级块中提供最小 IPC 子集（详见 [11-degraded-survival-layer.md §2.2](../10-architecture/11-degraded-survival-layer.md)）：

| 维度 | 正常模式（v1.1） | [DSL] 降级模式 |
|------|------|--------------|
| `capability_badge` 字段 | agentrt-linux 内核由 sec_d 编译，C-S9 校验 | 字段存在但被忽略（值=0，跳过 C-S9） |
| Capability 校验 | Badge + Epoch + RandomTag + Perms（C-S9 内联） | 退化到 `airy_cap_check()` slowpath 兜底（基于 `agent_caps[1024]` 静态数组；`AIRY_ECAP_RADIX_MISS`/`STATIC_KEY` 保留编号但 v1.1 不触发） |
| opcode 集 | 7 种（SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE） | 2 种（SEND/RECV） |
| Fault 码 | 0x1001-0x1003 启用（CAP_FORGED/CAP_LEAK/RING_CORRUPT） | Fault 码禁用，统一 Panic |

### 8.6 跨态协作流

```mermaid
graph TB
    subgraph "agentrt 用户态 AgentsIPC"
        RT_SEND[airy_ipc_send<br/>atoms/ipc<br/>capability_badge=0]
        RT_RECV[airy_ipc_recv<br/>atoms/ipc]
    end

    subgraph "agentrt-linux 内核态 io-uring-ipc"
        OS_SQE[IORING_OP_URING_CMD<br/>cmd_op=IPC_SEND<br/>capability_badge=Badge]
        OS_CQE[CQE poll<br/>cmd_op=IPC_RECV]
        OS_CSFastpath[fastpath C-S9<br/>airy_cap_badge_ok]
    end

    subgraph "[SC] 共享契约层"
        SC[ipc.h<br/>Layout C v4 128B hdr<br/>+ Badge 位布局宏<br/>+ AIRY_CAP_PERM_*]
    end

    subgraph "[DSL] 降级生存层"
        DSL["#ifdef AIRY_SC_FALLBACK<br/>capability_badge=0<br/>跳过 C-S9<br/>airy_cap_check() slowpath 兜底"]
    end

    RT_SEND -.->|"API 同源 [SS]"| OS_SQE
    RT_RECV -.->|"API 同源 [SS]"| OS_CQE
    RT_SEND ==>|"共享代码 [SC]"| SC
    RT_RECV ==>|"共享代码 [SC]"| SC
    OS_SQE ==>|"共享代码 [SC]"| SC
    OS_CQE ==>|"共享代码 [SC]"| SC
    OS_SQE -->|"C-S9 校验"| OS_CSFastpath
    SC -.->|"降级时"| DSL

    style SC fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px
    style RT_SEND fill:#e3f2fd,stroke:#1565c0
    style RT_RECV fill:#e3f2fd,stroke:#1565c0
    style OS_SQE fill:#fff3e0,stroke:#e65100
    style OS_CQE fill:#fff3e0,stroke:#e65100
    style OS_CSFastpath fill:#fce4ec,stroke:#c2185b
    style DSL fill:#fff9c4,stroke:#f57f17,stroke-dasharray: 5 5
```

> **OS-IFACE-004**： IPC 协议的 Layout C v4 128B 消息头布局（`struct airy_ipc_msg_hdr`）与 magic 0x41524531（'ARE1'）在 agentrt 与 agentrt-linux 间完全共享同一份 `ipc.h`——agentrt 在 agentrt-linux 上运行时无需任何协议转换或字节序适配，消息头字段布局二进制兼容。`capability_badge` 字段在 [SC] 层共享定义（H2），但校验机制属于 [SS] 语义同源层——两端对同一字段的消费方式不同（agentrt 用户态始终为 0，agentrt-linux 内核由 sec_d 编译并 C-S9 校验），这属于 [SS] 层范畴，不违反 [SC] 逐字节相同原则。

---

## 9. 跨节点 IPC 与 Badge 跨主机一致性（v1.1 新增）

### 9.1 设计边界声明（SSoT）

> **跨节点 IPC 单一权威源声明**：本节是 agentrt-linux 跨节点 IPC 架构与 Badge 跨主机作用域的唯一权威源。A-IPC（agentrt-linux IPC）的**物理作用域严格限定为单节点**——基于 io_uring + per-cpu kfifo 的单节点内核 IPC 体系（H1-H6 硬约束），Badge 64-bit Native Word 是**单节点内核 artifact**（存储于 `agent_caps[]` 静态数组，per-node 独立）。跨节点 IPC 通过独立的网络协议层（gRPC over QUIC/TCP + mTLS）实现，由 `gateway_d` daemon 在用户态承载，**Badge 不跨主机共享**——每个节点独立维护 `agent_caps[]` 与 `airy_cap_global_epoch`，跨节点 IPC 在每个节点的入站路径上重新进行 Badge 校验。

#### 9.1.1 为什么 Badge 不跨主机共享

| 维度 | 单节点 Badge（v1.1 选择） | 假设的跨主机 Badge（已拒绝） |
|------|--------------------------|---------------------------|
| 物理载体 | `agent_caps[1024]` 内核静态数组（per-node） | 需要分布式共享内存（RDMA/CXL）或分布式 KV |
| 全局 Epoch | `atomic_t`（单节点原子操作，~1ns） | 需要分布式原子操作（Paxos/Raft，~ms 级） |
| Random Tag | `get_random_bytes_wait()`（per-node CRNG） | 需要跨主机唯一性保证（避免碰撞） |
| 校验延迟 | ~10ns（fastpath C-S9 内联） | ~ms 级（网络往返 + 分布式共识） |
| 故障域 | 单节点故障不影响其他节点 | 任一节点故障可能影响全局 Epoch |
| 一致性模型 | 单节点线性一致 | 需要 CAP 取舍（强一致 vs 可用性） |
| 复杂度 | 低（Linux 内核标准原语） | 高（需引入分布式共识协议） |

**核心论断**：跨主机共享 Badge 会将 fastpath C-S9 从 ~10ns 退化至 ~ms 级，违反 H4 硬约束（fastpath C-S9 ~10ns 内联校验）。因此 Badge 必须 per-node 独立，跨节点 IPC 通过网络协议层 + 入站 Badge 重新校验实现。

#### 9.1.2 H1-H6 硬约束对跨节点 IPC 的约束

| 硬约束 | 单节点含义 | 跨节点含义 |
|--------|----------|----------|
| H1：Layout C v4 总长/magic | 128B 消息头 + magic 0x41524531 | 跨节点传输保持 128B 消息头不变（网络层不修改 [SC] 字段） |
| H2：[SC] 字段 + [SS] 校验 | `capability_badge` 字段经 [SC] 共享 | 跨节点传输 `capability_badge` 字段保持二进制兼容 |
| H3：agentrt badge=0 | agentrt 用户态 Badge 始终为 0 | agentrt 跨节点调用走用户态 API，badge=0 |
| H4：sec_d 编译 + fastpath C-S9 | sec_d 单节点编译，fastpath C-S9 单节点校验 | **每节点独立 sec_d + 独立 fastpath C-S9**（不跨节点） |
| H5：LSM 职责不变 | LSM slowpath 接管 | 跨节点入站路径每节点独立 LSM slowpath |
| H6：[DSL] badge=0 跳过 | 降级模式跳过 C-S9 | 跨节点降级模式每节点独立跳过 |

### 9.2 跨节点 IPC 架构总览

```
┌────────────────────────────────────────────────────────────────────┐
│                     Airymax 分布式集群（N 节点）                     │
│                                                                    │
│  ┌──────────────────────┐        ┌──────────────────────┐         │
│  │     Node A           │        │     Node B           │         │
│  │                      │        │                      │         │
│  │  ┌────────────────┐  │        │  ┌────────────────┐  │         │
│  │  │ Agent A1       │  │        │  │ Agent B1       │  │         │
│  │  │ (badge_A1)     │  │        │  │ (badge_B1)     │  │         │
│  │  └────────┬───────┘  │        │  └────────▲───────┘  │         │
│  │           │           │        │           │           │         │
│  │  ┌────────▼───────┐  │        │  ┌────────┴───────┐  │         │
│  │  │ A-IPC fastpath │  │        │  │ A-IPC fastpath │  │         │
│  │  │ C-S9 校验      │  │        │  │ C-S9 校验      │  │         │
│  │  │ (单节点)       │  │        │  │ (单节点)       │  │         │
│  │  └────────┬───────┘  │        │  └────────▲───────┘  │         │
│  │           │           │        │           │           │         │
│  │  ┌────────▼───────┐  │        │  ┌────────┴───────┐  │         │
│  │  │ gateway_d (A)  │◄─┼────────┼─┤ gateway_d (B)  │  │         │
│  │  │ gRPC/QUIC +mTLS│  │ 网络层  │  │ gRPC/QUIC +mTLS│  │         │
│  │  └────────────────┘  │        │  └────────────────┘  │         │
│  │                      │        │                      │         │
│  │  agent_caps[] (A)    │        │  agent_caps[] (B)    │         │
│  │  global_epoch (A)    │        │  global_epoch (B)    │         │
│  │  sec_d (A)           │        │  sec_d (B)           │         │
│  └──────────────────────┘        └──────────────────────┘         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

关键设计:
  1. A-IPC 严格单节点（H1-H6），不跨主机
  2. Badge per-node 独立（agent_caps[]、global_epoch、sec_d 均为 per-node）
  3. 跨节点 IPC 由 gateway_d daemon 承载，使用 gRPC over QUIC/TCP + mTLS
  4. 跨节点消息在入站节点重新进行 Badge 校验（每节点独立 C-S9）
  5. 128B 消息头跨节点保持二进制兼容（[SC] 字段不修改）
```

### 9.3 跨节点 IPC 协议分层

| 层次 | 协议 | 职责 | 物理宿主 |
|------|------|------|---------|
| L7 应用层 | Airymax IPC 语义 | SEND/RECV/SEND_BATCH 等 opcode 语义 | Agent 应用代码 |
| L6 消息层 | Layout C v4 128B 消息头 | [SC] 共享消息头（跨节点保持二进制兼容） | `ipc.h`（[SC] 共享） |
| L5 网关层 | Airymax Gateway Protocol（AGP） | 跨节点路由、Agent Global ID 解析、入站 Badge 重新编译 | `gateway_d` daemon |
| L4 传输层 | gRPC over QUIC / TCP | 可靠传输、流复用、连接管理 | gRPC 库 + QUIC/TCP |
| L3 安全层 | mTLS + 证书认证 | 节点身份认证、传输加密 | TLS 1.3 + X.509 证书 |
| L2 网络层 | IP + UDP/TCP | 跨主机网络路由 | Linux 内核网络栈 |
| L1 物理层 | Ethernet / InfiniBand / CXL | 物理传输介质 | 硬件 |

> **关键约束**：L6 消息层（128B 消息头）在跨节点传输中保持二进制兼容——`gateway_d` 不修改 `magic`、`opcode`、`flags`、`trace_id`、`timestamp_ns`、`src_task`、`dst_task`、`payload_len`、`crc32` 字段，**仅修改 `capability_badge` 字段**（入站时重新编译为目标节点的 Badge）。

### 9.4 Agent 身份模型：Local Agent ID vs Global Agent ID

跨节点 IPC 需要区分单节点内 Agent 身份与跨节点 Agent 身份：

```c
/* kernel/include/uapi/linux/airymax/agent_id.h —— Agent 身份模型（v1.1 新增）
 *
 * 物理宿主: kernel/include/uapi/linux/airymax/agent_id.h
 * 共享层: [SC] 共享契约层（agentrt 与 agentrt-linux 共享）
 */

/* Local Agent ID: 单节点内唯一，用于 agent_caps[] 索引与 fastpath C-S9
 * 范围: [0, 1024)，per-node 独立
 * 物理载体: 消息头 src_task / dst_task 字段（32-bit，[SC] ipc.h）
 * 注意: 同一 Global Agent 在不同节点可能有不同的 Local Agent ID
 */
typedef u32 airy_local_agent_id_t;  /* 范围 [0, 1024) */

/* Global Agent ID: 集群内全局唯一，用于跨节点路由
 * 范围: [0, 2^32-1)，集群内唯一
 * 物理载体: Airymax Gateway Protocol（AGP）消息包装层（不进入 [SC] 消息头）
 * 编码: node_id (16-bit) + local_agent_id (16-bit)
 *   - node_id: 集群内节点编号，由 cluster manager 分配
 *   - local_agent_id: 节点内 Agent 编号，由 sec_d 分配
 */
typedef u32 airy_global_agent_id_t;

#define AIRY_GLOBAL_AGENT_ID(node_id, local_id) \
    (((u32)(node_id) << 16) | ((u32)(local_id) & 0xFFFF))
#define AIRY_GLOBAL_AGENT_NODE(gid)   ((u16)((gid) >> 16))
#define AIRY_GLOBAL_AGENT_LOCAL(gid)  ((u16)((gid) & 0xFFFF))

/* Node ID: 集群内节点编号
 * 范围: [0, 65535)，由 cluster manager 分配
 * 物理载体: gateway_d 配置文件 + 集群注册表
 */
typedef u16 airy_node_id_t;
```

**身份映射表**：每个节点的 `gateway_d` 维护 Global Agent ID → Local Agent ID 映射表：

```c
/* services/daemons/gateway_d/agent_routing.c —— Agent 路由表（v1.1）
 *
 * 物理宿主: services/daemons/gateway_d/agent_routing.c
 * 维护: cluster manager 通过 Raft 协议同步
 */

struct airy_agent_route_entry {
    airy_global_agent_id_t  global_id;     /* 全局 Agent ID */
    airy_node_id_t          node_id;       /* 所在节点 */
    airy_local_agent_id_t   local_id;      /* 节点内 Agent ID */
    struct in_addr          node_addr;     /* 节点 IP 地址 */
    u16                     gateway_port;  /* 节点 gateway_d 端口 */
    u8                      state;         /* ACTIVE/DEPRECATED/MIGRATING */
} __attribute__((packed));

/* 路由表（哈希表，O(1) 查找） */
static struct airy_agent_route_entry *route_table[65536];

/* 查找 Global Agent 所在节点 */
static struct airy_agent_route_entry *
airy_route_lookup(airy_global_agent_id_t global_id)
{
    return route_table[global_id & 0xFFFF];  /* 简化哈希 */
}
```

### 9.5 跨节点 IPC 数据流（详尽流程）

#### 9.5.1 出站路径（Node A → Node B）

```
Agent A1 (Node A) 调用 airy_ipc_send() 发送到 Agent B1 (Node B)
       │
       ▼
[1] Agent A1 构造消息，capability_badge = badge_A1（Node A 的 Badge）
       │
       ▼
[2] Node A A-IPC fastpath C-S9 校验（单节点）：
    - 校验 badge_A1 在 Node A 的 agent_caps[A1] 中有效
    - 校验通过 → 消息投递到 Node A gateway_d 的 Ring
       │
       ▼
[3] Node A gateway_d 接收消息（airy_ipc_recv）：
    - 识别 dst_task = B1 是远端 Agent（查路由表）
    - 提取 Global Agent ID = AIRY_GLOBAL_AGENT_ID(B, B1)
    - 查路由表得到 Node B 的 IP 地址
       │
       ▼
[4] Node A gateway_d 通过 gRPC over QUIC 发送：
    - 包装层（AGP header）携带 Global Agent ID + 128B 消息头
    - mTLS 加密传输
    - 128B 消息头保持二进制兼容（不修改 [SC] 字段）
       │
       ▼
[5] 网络传输（QUIC/TCP + IP 路由）
       │
       ▼
[6] Node B gateway_d 接收（mTLS 解密 + AGP 解析）：
    - 解析 Global Agent ID = AIRY_GLOBAL_AGENT_ID(B, B1)
    - 提取 Local Agent ID = B1
    - 提取原始 128B 消息头
       │
       ▼
[7] Node B gateway_d 重新编译 Badge（关键步骤）：
    - 读取 Node B 的 agent_caps[B1]（若 B1 未注册，向 Node B sec_d 请求 Badge）
    - 将消息头 capability_badge 替换为 badge_B1（Node B 的 Badge）
    - 注意：其他字段（magic/opcode/flags/trace_id/timestamp_ns/src_task/dst_task/payload_len/crc32）不变
       │
       ▼
[8] Node B gateway_d 调用 airy_ipc_send() 投递到 Agent B1：
    - 使用 Node B 的 A-IPC（io_uring + fastpath C-S9）
    - fastpath C-S9 校验 badge_B1 在 Node B 的 agent_caps[B1] 中有效
    - 校验通过 → 消息投递到 Agent B1 的 Ring
       │
       ▼
[9] Agent B1 调用 airy_ipc_recv() 接收消息
```

#### 9.5.2 入站 Badge 重新编译机制（关键设计）

```c
/* services/daemons/gateway_d/inbound_relay.c —— 入站 Badge 重新编译（v1.1 新增）
 *
 * 物理宿主: services/daemons/gateway_d/inbound_relay.c
 * 职责: 跨节点入站消息在目标节点重新编译 Badge，使 fastpath C-S9 能正常校验
 *
 * 设计原则:
 *   1. 128B 消息头 [SC] 字段不修改（magic/opcode/flags/trace_id/timestamp_ns/
 *      src_task/dst_task/payload_len/crc32 保持不变）
 *   2. 仅修改 capability_badge 字段（替换为目标节点的 Badge）
 *   3. 重新计算 crc32（因为 capability_badge 变了）
 *   4. 向目标节点 sec_d 请求 Badge（若本地未缓存）
 */

static int gateway_d_inbound_relay(struct airy_ipc_msg_hdr *hdr,
                                     airy_global_agent_id_t src_global_id)
{
    airy_local_agent_id_t dst_local = AIRY_GLOBAL_AGENT_LOCAL(hdr->dst_task);
    u64 dst_badge;
    int ret;

    /* 1. 查询/请求目标 Agent 的 Badge（若未缓存）
     * 向本节点 sec_d 请求 Badge 编译
     */
    dst_badge = gateway_d_get_or_request_badge(dst_local,
                                                 AIRY_CAP_PERM_RECV);
    if (dst_badge == 0) {
        /* 目标 Agent 未注册或权限不足，丢弃消息 */
        pr_warn_ratelimited("airy_gw: agent %u not registered on this node\n",
                             dst_local);
        return -AIRY_ENOENT;
    }

    /* 2. 替换 capability_badge 字段（仅此字段修改） */
    hdr->capability_badge = dst_badge;

    /* 3. 重新计算 crc32（capability_badge 变更后必须重算） */
    hdr->crc32 = airy_ipc_crc32_compute(hdr,
                                          AIRY_IPC_HDR_SIZE + hdr->payload_len);

    /* 4. 通过本节点 A-IPC 投递到目标 Agent */
    ret = airy_ipc_send(hdr, NULL);  /* payload 在 AGP 包装层中携带 */
    if (ret < 0) {
        pr_warn_ratelimited("airy_gw: inbound relay failed: %d\n", ret);
        return ret;
    }

    /* 5. 审计日志（跨节点 IPC 入站） */
    airy_audit_emit_cross_node(AGENT_CROSS_NODE_INBOUND,
                                 src_global_id,
                                 AIRY_GLOBAL_AGENT_ID(airy_local_node_id(), dst_local),
                                 hdr->trace_id);
    return 0;
}

/**
 * gateway_d_get_or_request_badge - 查询或请求本地 Agent Badge
 *
 * @local_agent_id: 目标节点的 Local Agent ID
 * @required_perms: 所需权限位
 *
 * 返回: Badge 64-bit Native Word；0 表示未注册或权限不足
 *
 * 缓存策略:
 *   - gateway_d 维护本地 Badge 缓存（per-Agent，TTL = 60s）
 *   - 缓存未命中或过期时，向本节点 sec_d 请求 Badge 编译
 *   - sec_d 通过 AIRY_IPC_OP_CAP_REQUEST 接收请求，返回 CAP_RESPONSE
 */
static u64 gateway_d_get_or_request_badge(airy_local_agent_id_t local_agent_id,
                                             u16 required_perms)
{
    struct badge_cache_entry *entry;
    u64 badge;
    ktime_t now = ktime_get_real_ns();

    /* 1. 查询本地缓存 */
    entry = badge_cache_lookup(local_agent_id);
    if (entry && (now - entry->timestamp_ns < BADGE_CACHE_TTL_NS)) {
        /* 缓存命中且未过期 */
        if ((entry->perms & required_perms) == required_perms)
            return entry->badge;
    }

    /* 2. 缓存未命中或过期，向 sec_d 请求 Badge 编译 */
    badge = airy_cap_request_badge(local_agent_id, required_perms);
    if (badge == 0)
        return 0;

    /* 3. 更新缓存 */
    if (entry) {
        entry->badge = badge;
        entry->perms = required_perms;
        entry->timestamp_ns = now;
    } else {
        badge_cache_insert(local_agent_id, badge, required_perms, now);
    }

    return badge;
}
```

### 9.6 跨节点 Badge 一致性模型

#### 9.6.1 Badge 不跨主机一致（核心设计）

**设计原则**：Badge 是 **per-node 独立**的，不跨主机共享。每个节点的 `agent_caps[]`、`airy_cap_global_epoch`、`sec_d` 都是独立的。

| 维度 | Node A | Node B | 一致性保证 |
|------|--------|--------|----------|
| `agent_caps[]` | Node A 独立 | Node B 独立 | 不需要一致（per-node 独立） |
| `airy_cap_global_epoch` | Node A 独立 | Node B 独立 | 不需要一致（per-node 独立） |
| `sec_d` | Node A 独立 | Node B 独立 | 不需要一致（per-node 独立） |
| Agent Global ID | 集群唯一 | 集群唯一 | 通过 cluster manager 保证 |
| 路由表 | Node A 副本 | Node B 副本 | 通过 Raft 协议强一致 |
| mTLS 证书 | 集群 CA 签发 | 集群 CA 签发 | 通过 PKI 保证 |

#### 9.6.2 跨节点身份一致性（mTLS + 证书认证）

跨节点 IPC 的身份认证通过 **mTLS + X.509 证书** 实现，不依赖 Badge：

```c
/* services/daemons/gateway_d/tls_init.c —— mTLS 配置（v1.1 新增）
 *
 * 物理宿主: services/daemons/gateway_d/tls_init.c
 * 职责: 跨节点 IPC 的节点身份认证
 *
 * 认证模型:
 *   1. 集群 CA（Certificate Authority）签发每个节点的证书
 *   2. 证书包含 node_id、节点 IP、公钥
 *   3. mTLS 双向认证：Node A 验证 Node B 证书，Node B 验证 Node A 证书
 *   4. 认证通过后建立 QUIC/TCP 加密通道
 *   5. Badge 仅在节点内部校验，跨节点身份认证由 mTLS 完成
 */

static SSL_CTX *gateway_d_tls_init(const char *cert_path,
                                     const char *key_path,
                                     const char *ca_path)
{
    SSL_CTX *ctx = SSL_CTX_new(TLS_method());

    /* TLS 1.3 强制 */
    SSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION);

    /* 加载本节点证书与私钥 */
    if (SSL_CTX_use_certificate_file(ctx, cert_path, SSL_FILETYPE_PEM) <= 0) {
        pr_err("airy_gw: failed to load cert: %s\n", cert_path);
        goto err;
    }
    if (SSL_CTX_use_PrivateKey_file(ctx, key_path, SSL_FILETYPE_PEM) <= 0) {
        pr_err("airy_gw: failed to load key: %s\n", key_path);
        goto err;
    }

    /* 加载集群 CA 证书，强制客户端认证 */
    SSL_CTX_load_verify_locations(ctx, ca_path, NULL);
    SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT,
                         gateway_d_cert_verify_callback);

    /* 证书校验回调：检查 node_id 与证书 CN 匹配 */
    SSL_CTX_set_verify_depth(ctx, 3);

    return ctx;
err:
    SSL_CTX_free(ctx);
    return NULL;
}

/* 证书校验回调：验证对端证书的 node_id 与预期一致 */
static int gateway_d_cert_verify_callback(int preverify_ok, X509_STORE_CTX *ctx)
{
    X509 *cert = X509_STORE_CTX_get_current_cert(ctx);
    char cn[256];

    if (!preverify_ok)
        return 0;  /* 证书链验证失败 */

    /* 提取 CN（Common Name），格式: "airy-node-{node_id}" */
    X509_NAME_get_text_by_NID(X509_get_subject_name(cert),
                                NID_commonName, cn, sizeof(cn));

    /* 验证 CN 格式与 node_id */
    if (strncmp(cn, "airy-node-", 10) != 0)
        return 0;

    /* 验证 node_id 在集群路由表中注册 */
    airy_node_id_t peer_node_id = (airy_node_id_t)atoi(cn + 10);
    if (!cluster_route_table_contains(peer_node_id)) {
        pr_warn_ratelimited("airy_gw: unknown peer node %u\n", peer_node_id);
        return 0;
    }

    return 1;  /* 认证通过 */
}
```

#### 9.6.3 跨节点 Epoch 撤销传播

当某节点撤销所有 Badge（`atomic_inc(&airy_cap_global_epoch)`），该节点的 Badge 全部失效。但**跨节点 IPC 不受影响**——因为：

1. **跨节点 IPC 的 Badge 在入站时重新编译**（§9.5.2）：Node B 的 `gateway_d` 在入站时为 Agent B1 编译新的 Badge，使用 Node B 当前的 `global_epoch`
2. **Node A 的 Epoch 撤销不影响 Node B**：Node A 的 `atomic_inc` 仅影响 Node A 的 Badge，Node B 的 `global_epoch` 独立
3. **跨节点消息在 Node A 出站时使用 Node A 的 Badge**（已校验），在 Node B 入站时重新编译为 Node B 的 Badge（重新校验）

**唯一需要跨节点同步的场景**：Agent 从 Node A 迁移到 Node B 时，需要通知 Node A 撤销该 Agent 的 Badge（防止旧 Node 上的残留 Badge 被滥用）。这是 **Agent 迁移协议**的职责，由 cluster manager 通过 Raft 共识协调：

```c
/* services/daemons/gateway_d/agent_migration.c —— Agent 迁移协议（v1.1）
 *
 * 物理宿主: services/daemons/gateway_d/agent_migration.c
 * 触发: cluster manager 决策（负载均衡/故障转移）
 *
 * 迁移流程:
 *   1. cluster manager 通过 Raft 决议：Agent X 从 Node A 迁移到 Node B
 *   2. Node A sec_d 撤销 Agent X 的 Badge（atomic_inc global_epoch 或单 Agent 撤销）
 *   3. Node A gateway_d 更新路由表：Agent X 的 node_id 改为 Node B
 *   4. Node B sec_d 为 Agent X 编译新 Badge
 *   5. Node B gateway_d 接收 Agent X 的 IPC 请求
 *
 * 注意: 迁移期间 Agent X 的 IPC 可能有短暂中断（~100ms-1s），
 *       由 cluster manager 通知 Agent X 重连
 */
```

### 9.7 跨节点 IPC 性能约束（更新 NFR-P-002c）

| 约束 ID | 指标 | 阈值 | 测量方法 |
|---------|------|------|---------|
| NFR-P-002c | 跨节点 IPC 吞吐 | > 1M msg/s（4 节点） | gRPC over QUIC，128B 消息 |
| NFR-P-002c-latency | 跨节点 IPC 延迟 | < 500μs（P99，同机房） | gRPC over QUIC + mTLS |
| NFR-P-002c-latency-cxl | 跨节点 IPC 延迟（CXL） | < 100μs（P99，CXL 互联） | CXL 3.0 内存池化 |
| NFR-P-002c-relay | gateway_d 入站 Badge 重新编译延迟 | < 50μs（P99，缓存命中） | gateway_d 内部测量 |

**性能优化手段**：
1. **gRPC over QUIC**：避免 TCP 三次握手 + TLS 二次握手，QUIC 0-RTT 重连
2. **mTLS session resumption**：会话恢复避免重复证书验证
3. **gateway_d Badge 缓存**：TTL 60s，避免每次入站都向 sec_d 请求 Badge
4. **连接池**：每对节点维护持久连接，避免连接建立开销
5. **批量传输**：跨节点 IPC 支持 SEND_BATCH opcode，摊销网络开销

### 9.8 跨节点 IPC 故障场景与处理

| 故障场景 | 处理机制 | 恢复时间 |
|---------|---------|---------|
| gateway_d 崩溃 | systemd Restart=always 自动重启 | ~1s |
| 网络分区（Node A ↔ Node B 不通） | cluster manager 标记 Node B 不可达，路由表更新 | ~5s（Raft 心跳超时） |
| sec_d 崩溃（Node A） | Node A sec_d 重启，已发出 Badge 不影响跨节点 IPC（Node B 的 Badge 独立） | ~2s（§4.5 sec_d 故障恢复） |
| 节点故障（Node A 宕机） | cluster manager 触发 Agent 迁移，Node A 上的 Agent 迁移到其他节点 | ~30s-5min（取决于 Agent 状态） |
| 证书过期 | mTLS 握手失败，gateway_d 拒绝连接，触发证书更新告警 | 人工干预或自动证书轮换 |
| 路由表不一致 | Raft 协议保证最终一致，不一致期间消息可能丢弃（依赖重试） | ~1-5s（Raft 收敛） |

### 9.9 跨节点 IPC 与 A-IPC 的边界（SSoT）

| 维度 | A-IPC（单节点） | 跨节点 IPC（gateway_d） |
|------|---------------|----------------------|
| 物理作用域 | 单节点内核 | 集群内多节点 |
| 传输载体 | io_uring + per-cpu kfifo | gRPC over QUIC/TCP + mTLS |
| Badge 模型 | per-node 独立（agent_caps[]、global_epoch） | per-node 独立（不跨主机共享） |
| fastpath C-S9 | 单节点内联校验，~10ns | 每节点独立 C-S9（入站时重新校验） |
| sec_d 职责 | 单节点 Badge 编译 | per-node 独立编译（不跨节点共识） |
| 身份认证 | Badge + Random Tag（内核校验） | mTLS + X.509 证书（网络层认证） |
| 延迟 SLO | < 1μs（P99） | < 500μs（P99，同机房） |
| 吞吐 SLO | > 100K msg/s（单核） | > 1M msg/s（4 节点集群） |
| 故障域 | 单节点 | 集群（任一节点故障不影响其他节点） |

> **核心设计论断**：A-IPC 与跨节点 IPC 是**两个独立的 IPC 体系**——A-IPC 是单节点内核 IPC（H1-H6 硬约束），跨节点 IPC 是用户态网络协议层（gRPC + mTLS）。两者通过 `gateway_d` daemon 桥接，**Badge 在桥接时重新编译**（出站用源节点 Badge，入站重新编译为目标节点 Badge）。这种设计保证了 A-IPC 的 fastpath C-S9 始终是单节点内联校验（~10ns），跨节点 IPC 的延迟与复杂度由 `gateway_d` 承担，不污染 A-IPC fastpath。

### 9.10 跨节点 IPC 相关文档

- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §7 A-ULS 模块（gateway_d daemon 定位）
- [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md) §2.3 12 daemon 拉起顺序（gateway_d 依赖 ipc_daemon）
- [110-security/03-capability-model.md](../110-security/03-capability-model.md) §2.5 Badge 64-bit Native Word 模型（per-node 独立）
- [110-security/07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) §3.5 sec_d Badge 编译职责（per-node 独立）
- [00-requirements/03-non-functional-requirements.md](../00-requirements/03-non-functional-requirements.md) NFR-P-002c 跨节点 IPC 性能约束

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
