Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）IPC 消息流
> **文档定位**：agentrt-linux（AirymaxOS）IPC 消息流的详细设计，刻画 io_uring 零拷贝、SQ/CQ ring 机制与 128B 消息头生命周期；v1.1 起落地 Capability Folding 单平面架构（io_uring 数据面唯一平面 + fastpath C-S9 Badge 内联校验）\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **核心约束**：IRON-9 v3 同源且部分代码共享——[SC] IPC 消息头布局（128B 定长 + magic 0x41524531 'ARE1' + Layout C v4 含 `capability_badge` offset 40-47）+ 5 种 payload 协议落地于 include/uapi/linux/airymax/，[SS] io_uring SQE/CQE 语义 + io_uring_register OP 语义同源 + fastpath C-S9 Badge 校验机制同源，[IND] io_uring 内核驱动实现 + ring buffer 内存分配策略独立

---

## 1. IPC 消息流概览

IPC 消息流是 agentrt-linux 进程间通信的核心数据流，落地于 `kernel` 子仓，并贯穿 `services` 的 12 daemons。该数据流基于 Linux 6.6 内核基线原生的 io_uring 子系统实现零拷贝、零 syscall 的高性能消息传递。**v1.1 起，IPC 数据流是 Capability Folding 单平面架构的唯一数据面**——所有 IPC 消息传递即能力校验（fastpath C-S9 内联 Badge），无独立控制面 syscall 路径。

**核心特征**：

1. **io_uring 零拷贝**（FR-003, FR-009）：通过共享环形缓冲区（SQ ring + CQ ring）避免用户态 ↔ 内核态数据拷贝与 syscall 开销，I/O 延迟降低 > 30%（NFR-P-002）。
2. **128B 定长消息头**（[SC] 共享契约层）：同源 agentrt AgentsIPC，定长 128 字节，packed 紧凑布局（Layout C v4 SSoT），128 字节 = 2 cache lines，跨系统互通无适配层。**v1.1: offset 40-47 为 `capability_badge` 字段（64-bit Native Word）**。
3. **5 种 payload 协议**：REQUEST / RESPONSE / EVENT / STREAM / CONTROL，覆盖请求-响应、事件订阅、流式传输、控制指令 4 类通信模式。
4. **trace_id 贯穿**：每条消息携带 `trace_id`，通过 OpenTelemetry 全链路追踪（NFR-O-002），语义同源 io_uring `user_data` 字段。
5. **跨节点扩展**：基于 CXL 3.0 / RDMA / NVLink 实现超节点 OS 跨节点 IPC（FR-048）。
6. **零 syscall 提交**：SQPOLL 内核轮询线程 + DEFER_TASKRUN 模式实现零 syscall 高频 IPC。
7. **（v1.1 新增）Capability Folding 单平面架构**：IPC 数据传递即能力校验，fastpath C-S9 内联 Badge 校验（~10ns），无独立控制面 syscall。详见 [10-unify-design.md §8](../10-architecture/10-unify-design.md)。
8. **（v1.1 新增）6 条硬约束 H1-H6**：Layout C v4 总长 128B / magic 0x41524531 / capability_badge offset 40-47 / agentrt 用户态 badge=0 / sec_d 编译 + fastpath C-S9 校验 / [DSL] 降级 badge=0 跳过。详见 [10-unify-design.md §8.2](../10-architecture/10-unify-design.md)。

**性能目标**（NFR-P-002）：

| 指标 | 目标 | 验证方法 |
|---|---|---|
| 消息吞吐 | > 100K msg/s（实测 250K msg/s） | io_uring 基准测试 |
| 单节点延迟 | < 10μs（实测 4.2μs） | ping-pong 测试 |
| 跨节点延迟 | < 10μs（RDMA/CXL） | 节点间 ping-pong |
| trace_id 开销 | < 1% | 性能对比测试 |
| 零拷贝大 payload | < 5μs（4KB） | registered buffer 基准 |

---

## 2. io_uring 共享环形缓冲区机制

agentrt-linux IPC 基于 Linux 6.6 内核基线原生的 io_uring 子系统实现。本节描述 io_uring 的核心机制，作为 IPC 消息流的底层传输基础。

### 2.1 SQ/CQ Ring 内存布局

io_uring 通过 `mmap()` 将环形缓冲区映射到用户态地址空间，实现内核与用户态共享内存通信：

| mmap 偏移 | 内容 | 用途 |
|-----------|------|------|
| `IORING_OFF_SQ_RING` | SQ ring（head/tail/mask/flags + sq_array） | 提交侧环形缓冲区 |
| `IORING_OFF_CQ_RING` | CQ ring（head/tail/mask/flags + cqes[]） | 完成侧环形缓冲区 |
| `IORING_OFF_SQES` | SQE 数组（每个 64B 或 128B） | 提交队列条目数组 |

**所有权模型**（单生产者-单消费者无锁环）：

| 环字段 | 写入者 | 读取者 | agentrt-linux IPC 对应 |
|--------|--------|--------|---------------------|
| `sq.tail` | 应用（生产者） | 内核（消费者） | daemon 写入 SQE |
| `sq.head` | 内核（消费者） | 应用（生产者） | 内核消费 SQE |
| `cq.tail` | 内核（生产者） | 应用（消费者） | 内核写 CQE |
| `cq.head` | 应用（消费者） | 内核（生产者） | daemon 消费 CQE |

> **OS-IPC-001**： SQ/CQ ring 的单生产者-单消费者所有权契约是 io_uring 无锁设计的基石。agentrt-linux IPC 必须严格遵守——daemon 只写 `sq.tail` + `cq.head`，内核只写 `sq.head` + `cq.tail`，避免任何一方越权导致竞态。

### 2.2 SQE 提交流程

```
应用侧（daemon 生产者）：
1. 读取 sq.tail（TAIL），计算 sq_array[TAIL & mask] = SQE_index
2. 填充 sqes[SQE_index]（opcode + fd + addr + len + user_data=trace_id）
3. smp_store_release(sq.tail, TAIL + 1)  // 内存屏障确保 SQE 先于 tail 可见
4. （非 SQPOLL）io_uring_enter(fd, to_submit=1, ...) 提交
   （SQPOLL）无需 syscall，内核线程自动消费

内核侧（消费者）：
1. smp_load_acquire(sq.tail) 读取新 tail
2. 读取 sq_array[head & mask] 获取 SQE_index
3. 读取 sqes[SQE_index] 解析请求
4. sq.head++ 推进消费指针
5. 执行 io_submit_sqe() 处理请求
6. （成功）写 CQE；（失败）写错误 CQE
```

### 2.3 CQE 完成流程

```
内核侧（生产者）：
1. io_get_cqe(ctx, &cqe) 从 CQE 环分配槽位
2. cqe->user_data = req->cqe.user_data（原样返回 trace_id）
3. cqe->res = result（成功=字节数，失败=-Exxx）
4. cqe->flags = completion_flags（BUFFER/MORE/NOTIF）
5. smp_store_release(cq.tail, cached_cq_tail + 1)  // 内存屏障
6. io_cqring_wake(ctx)  // 唤醒等待的 io_uring_enter

应用侧（daemon 消费者）：
1. smp_load_acquire(cq.tail) 读取新 tail
2. 读取 cqes[head & mask] 获取完成事件
3. 根据 user_data 匹配请求（trace_id）
4. cq.head++ 推进消费指针
```

### 2.4 Mermaid 序列图

下图为单节点内进程 A → io_uring → 进程 B 的完整 IPC 消息流，包含 11 个步骤与 trace_id 贯穿：

```mermaid
sequenceDiagram
    participant A as 进程 A（daemon）
    participant U as io_uring ring
    participant K as 内核
    participant B as 进程 B（daemon）

    A->>A: 1. 构造 128B 消息头（trace_id 填充）
    A->>A: 2. 填充 payload
    A->>U: 3. 写入 SQE（smp_store_release sq.tail）
    Note over A,U: 共享内存写入，零 syscall（SQPOLL）
    U->>K: 4. io_sq_thread 消费 SQE
    K->>K: 5. 零拷贝传递（registered buffer）
    K->>U: 6. 写入 CQE（smp_store_release cq.tail）
    U->>B: 7. task_work 通知（TWA_SIGNAL_NO_IPI）
    B->>B: 8. 解析消息头（trace_id 匹配）
    B->>B: 9. 处理 payload
    B->>U: 10. 提交响应 SQE
    U->>A: 11. 完成 CQE 响应

    Note over A,B: trace_id 贯穿全程<br/>OpenTelemetry 追踪
```

### 2.5 io_uring SETUP 标志完整清单

agentrt-linux IPC 基于 Linux 6.6 内核基线 `include/uapi/linux/io_uring.h` 的 io_uring 子系统实现。下表列出 Linux 6.6 支持的全部 `IORING_SETUP_*` 标志（19 项）及 agentrt-linux IPC 启用策略：

| # | IORING_SETUP_ 标志 | 引入版本 | agentrt-linux 启用 | 用途 | 关联规则 |
|---|--------------------|---------|---------------------|------|---------|
| 1 | `IOPOLL` | 5.1 | ❌ | 轮询模式（存储设备 HDD/SSD 专用） | 不适用 IPC（IPC 无块设备 I/O） |
| 2 | `SQPOLL` | 5.11 | ✅ | 内核轮询线程，零 syscall 提交 | OS-IPC-002 |
| 3 | `SQ_AFF` | 5.11 | ✅ | SQPOLL 线程 CPU 亲和性绑定 | OS-IPC-002（性能调优） |
| 4 | `CQSIZE` | 5.11 | ✅ | 自定义 CQ ring 大小 | 默认 CQ = 2×SQ |
| 5 | `CLAMP` | 5.12 | ✅ | 限制 SQ/CQ 不超过 MAX_ENTRIES | 资源保护 |
| 6 | `ATTACH_WQ` | 5.12 | ❌ | 复用已存在 io_wq（多 ring 共享） | 不适用 IPC（每 daemon 独立 io_wq） |
| 7 | `R_DISABLED` | 5.13 | ❌ | ring 创建后处于 R_DISABLED 状态 | 不适用 IPC（立即可用） |
| 8 | `SUBMIT_ALL` | 5.18 | ✅ | `io_uring_enter` 持续提交直到 SQ 空 | 提升吞吐 |
| 9 | `COOP_TASKRUN` | 5.19 | ✅ | 避免任务上下文异步切换 | 与 DEFER_TASKRUN 二选一 |
| 10 | `TASKRUN_FLAG` | 5.19 | ✅ | 设置 IORING_SQ_TASKRUN 标志 | 配合 COOP_TASKRUN |
| 11 | `SQE128` | 5.18 | ✅ | 128 字节 SQE（扩展字段），`cmd` 扩展至 80 字节承载 `airy_ipc_cmd`（D-7 修复） | OS-IPC-009 |
| 12 | `CQE32` | 5.18 | ❌ | 32 字节 CQE（扩展字段） | IPC 用 16B CQE 足够 |
| 13 | `SINGLE_ISSUER` | 6.0 | ✅ | 单一提交者约束（DEFER_TASKRUN 前置） | OS-IPC-003 |
| 14 | `DEFER_TASKRUN` | 6.0 | ✅ | task_work 延迟到 `enter` 执行 | OS-IPC-003 |
| 15 | `NO_MMAP` | 6.0 | ✅ | daemon 主动分配 ring 内存 | 走 MemoryRovol 管理 |
| 16 | `REGISTERED_FD_ONLY` | 6.0 | ❌ | ring fd 固定（无 `io_uring_register`） | daemon 用动态 fd |
| 17 | `NO_SQARRAY` | 6.6 | ✅ | 取消 sq_array 间接寻址 | 简化提交路径 |
| 18 | `NO_IEC` | 6.6 | ✅ | 禁用内部完整事件计数 | 性能优化 |
| 19 | `DEFER_INIT` | 6.7 | ❌ 不启用 | 延迟初始化 ring 资源 | Linux 6.6 基线不支持（ADR-001）；1.0.1 升级 Linux 7.1 后启用（ADR-013 版本路线，非设计延期） |

**启用策略**：

- **必须启用**（12 项）：`SQPOLL` / `SQ_AFF` / `CQSIZE` / `CLAMP` / `SUBMIT_ALL` / `SQE128` / `TASKRUN_FLAG` / `SINGLE_ISSUER` / `DEFER_TASKRUN` / `NO_MMAP` / `NO_SQARRAY` / `NO_IEC`——覆盖零 syscall（OS-IPC-002）、零拷贝（OS-IPC-003）、低延迟三大场景；`SQE128` 扩展 `cmd` 至 80 字节以承载 `airy_ipc_cmd`（D-7 修复，详见 [30-interfaces/02-ipc-protocol.md §4.9](../30-interfaces/02-ipc-protocol.md) + [30-interfaces/07-ipc-fastpath.md §5.7](../30-interfaces/07-ipc-fastpath.md)）。
- **互斥标志二选一**：`COOP_TASKRUN`（5.19+）或 `DEFER_TASKRUN`（6.0+），不可同时启用。
- **不启用**（5 项）：`IOPOLL` / `ATTACH_WQ` / `R_DISABLED` / `CQE32` / `REGISTERED_FD_ONLY`——不适用于 IPC 场景（块设备 I/O / 多 ring 共享 / 立即禁用 / 扩展 CQE / 固定 fd）。
- **版本路线**（1 项）：`DEFER_INIT`——Linux 6.6 基线不支持（ADR-001）；1.0.1 升级 Linux 7.1 后启用（ADR-013 版本路线，非设计延期；设计决策已在 0.1.1 完成：0.1.1 阶段不启用，1.0.1 阶段启用）。

> **OS-IPC-009**：agentrt-linux IPC 必须显式启用上述 12 个 `IORING_SETUP_*` 标志（含 `SQE128`，D-7 修复后从禁止列表移至必须启用列表，以扩展 `cmd` 至 80 字节承载 `airy_ipc_cmd`），并在互斥对 `COOP_TASKRUN` / `DEFER_TASKRUN` 中二选一，禁止启用 `IOPOLL` / `ATTACH_WQ` / `R_DISABLED` / `CQE32` / `REGISTERED_FD_ONLY`（不适用于 IPC 场景）。`DEFER_INIT` 设计决策已在 0.1.1 完成（0.1.1 阶段不启用——Linux 6.6 基线不支持 ADR-001；1.0.1 阶段启用——升级 Linux 7.1 后 ADR-013），属版本路线规划非设计延期。

> **与本文档已有提及对齐**：`SQPOLL`（§2.4 / §4 / §10.3）/ `SQ_AFF`（§10.3）/ `DEFER_TASKRUN`（§5）/ `NO_MMAP`（§12.3 [IND]）——本文档前述章节已分散提及 4 项，本表补全至 Linux 6.6 全量 19 项，确保 io_uring SETUP 标志清单的完整性与对齐性。

---

## 3. 128B 消息头生命周期

128B 定长消息头同源 agentrt AgentsIPC（[SC] 共享契约层），结构定义见 [IPC 协议](../30-interfaces/02-ipc-protocol.md)。本节描述消息头从创建到回收的 5 阶段生命周期。

### 3.1 消息头结构回顾

```c
/* IPC 128B 消息头定义见 [SC] 共享契约层（SSoT），不就地重定义 */
#include <airymax/ipc.h>
/* 结构体名称：struct airy_ipc_msg_hdr（Layout C v4，物理宿主见
 * 50-engineering-standards/120-cross-project-code-sharing.md §Layout C v4） */
```

**字段语义同源说明**（v1.1: 新增 `capability_badge` 字段）：

| 字段 | offset | 同源来源 | 同源标注 |
|------|--------|---------|----------|
| `magic` (0x41524531 'ARE1') | 0 | agentrt AgentsIPC | [SC] 完全共享 |
| `trace_id` | 8 | io_uring `user_data`（提交时填充，完成时原样返回） | [SC] 共享契约 |
| `timestamp_ns` | 16 | CLOCK_REALTIME | [SS] 语义同源 |
| `src_task` / `dst_task` | 24 / 32 | agentrt-linux 扩展（io_uring 无此字段） | [IND] 独立扩展 |
| **`capability_badge`** | **40** | **v1.1 新增: Capability Folding 物理载体（64-bit Native Word: Epoch + RandomTag + Perms）** | **[SC] 数据结构共享 + [SS] 校验机制同源** |
| `crc32` | 52 | v1.1 新增: 覆盖 header[0:52) + payload 完整性校验 | [SC] 完全共享 |

**Capability Folding 物理载体**：offset 40-47 这 8 字节是 fastpath C-S9 校验的对象。详见 [110-security/03-capability-model.md §2.5](../110-security/03-capability-model.md)。

### 3.2 5 阶段生命周期表

**v1.1: 阶段 3 接收新增 fastpath C-S9 Badge 内联校验**：

| 阶段 | 阶段名 | 触发者 | 操作 | 状态变更 | 持续时间 | 同源 |
|---|---|---|---|---|---|---|
| 1 | 创建 | 发送进程 | 分配 128B 缓冲区 + 填充字段（含 `capability_badge`）| UNINIT → READY | < 1μs | AgentsIPC create |
| 2 | 发送 | 发送进程 | 写入 SQE（SQPOLL 零 syscall） | READY → IN_FLIGHT | < 2μs | AgentsIPC send |
| 3 | 接收 | 内核 + io_uring | **v1.1: fastpath C-S0~C-S12 校验（含 C-S9 Badge ~10ns）** + 零拷贝传递（registered buffer）+ CQE 完成 | IN_FLIGHT → DELIVERED | < 5μs（含 C-S9 ~10ns） | AgentsIPC deliver |
| 4 | 处理 | 接收进程 | 解析消息头 + 处理 payload（若 CAP_CARRY，提取派生 Badge）| DELIVERED → PROCESSING | 业务相关 | AgentsIPC process |
| 5 | 回收 | 接收进程 | 释放 128B 缓冲区（provided buffer 回收） | PROCESSING → FREED | < 1μs | AgentsIPC recycle |

### 3.3 状态机

```mermaid
stateDiagram-v2
    [*] --> UNINIT: 分配缓冲区
    UNINIT --> READY: 填充字段
    READY --> IN_FLIGHT: 写入 SQE（smp_store_release sq.tail）
    IN_FLIGHT --> DELIVERED: CQE 完成（smp_store_release cq.tail）
    DELIVERED --> PROCESSING: 接收解析
    PROCESSING --> FREED: 处理完成
    FREED --> [*]: 缓冲区回收

    IN_FLIGHT --> TIMEOUT: 超时未送达
    TIMEOUT --> RETRY: 重试（≤3 次）
    RETRY --> IN_FLIGHT: 重新提交
    RETRY --> FAILED: 重试耗尽
    FAILED --> [*]: 审计 + 告警
```

### 3.4 阶段约束

| 约束 | 阶段 | 目标 | NFR |
|---|---|---|---|
| 创建延迟 | 阶段 1 | < 1μs | NFR-P-002 |
| 发送延迟（SQPOLL） | 阶段 2 | < 2μs（零 syscall） | NFR-P-002 |
| 传递延迟（零拷贝） | 阶段 3 | < 5μs（单节点） | NFR-P-002 |
| 超时阈值 | 阶段 3 | 100ms（可配置） | NFR-R-004 |
| 重试次数 | 失败恢复 | ≤ 3 次 | NFR-R-004 |
| 回收延迟 | 阶段 5 | < 1μs | NFR-R-006 |

---

## 4. SQPOLL 零 syscall 提交模式

agentrt-linux daemon 间高频 IPC 采用 SQPOLL 模式——内核创建专用 `io_sq_thread` 线程持续轮询 SQ ring，应用只需写入 SQE + 推进 `sq.tail`，无需任何 syscall。

### 4.1 SQPOLL 状态机

```mermaid
stateDiagram-v2
    [*] --> Running: ring 创建（IORING_SETUP_SQPOLL）
    Running --> Idle: SQ 空闲 sq_thread_idle ms
    Idle --> Running: 应用写 SQ tail + IORING_SQ_NEED_WAKEUP
    Idle --> Wakeup: io_uring_enter(IORING_ENTER_SQ_WAKEUP)
    Wakeup --> Running: 处理积压 SQE
    Running --> Running: 持续处理 SQE
    Running --> Parked: cpu_hotplug / system suspend
    Parked --> Running: cpu_online
```

### 4.2 SQPOLL 性能特征

| 场景 | 非 SQPOLL | SQPOLL | 收益 |
|------|-----------|--------|------|
| 提交延迟 | 2-5μs（syscall + 切换） | < 1μs（共享内存写） | 5× |
| 高频小 I/O（128B） | 100K msg/s | 250K msg/s | 2.5× |
| CPU 占用（应用侧） | syscall 开销 | 仅内存写 | 大幅降低 |
| CPU 占用（内核侧） | 中断驱动 | 专用轮询线程 | 1 个 CPU 核 |

### 4.3 SQPOLL 唤醒机制

应用通过 `IORING_ENTER_SQ_WAKEUP` flag 唤醒睡眠的 SQPOLL 线程：

```c
/* daemon 侧唤醒 SQPOLL */
if (sq_flags & IORING_SQ_NEED_WAKEUP) {
    io_uring_enter(fd, 0, 0, IORING_ENTER_SQ_WAKEUP, NULL, 0);
}
```

> **OS-IPC-002**： SQPOLL 模式下 `sq_thread_idle` 超时是平衡延迟与 CPU 占用的关键。agentrt-linux 推荐：高频 IPC 场景设为 5-10s（避免频繁睡眠），低延迟场景设为 100ms（快速响应）。

---

## 5. DEFER_TASKRUN 低延迟模式

`IORING_SETUP_DEFER_TASKRUN`（Linux 6.0+）将 task_work 延迟到 `io_uring_enter()` 上下文执行，避免 IPI（Inter-Processor Interrupt）和跨 CPU 唤醒。

### 5.1 三种 task_work 通知模式对比

| 模式 | 触发方式 | IPI 开销 | 延迟 | 适用场景 |
|------|---------|---------|------|---------|
| `TWA_SIGNAL`（默认） | TIF_NOTIFY_SIGNAL + IPI | 有 | 低 | 通用场景 |
| `TWA_SIGNAL_NO_IPI`（COOP_TASKRUN） | TIF_NOTIFY_SIGNAL，无 IPI | 无 | 中（下次 syscall） | 低延迟敏感 |
| `DEFER_TASKRUN` | io_uring_enter 内执行 | 无 | 高（需主动 enter） | 极致低延迟 |

### 5.2 DEFER_TASKRUN 约束

| 约束 | 原因 |
|------|------|
| 必须与 `SINGLE_ISSUER` 同时使用 | 确保单一提交者，避免 task_work 跨任务 |
| 不能与 `SQPOLL` 同时使用 | SQPOLL 自带线程，不需要 DEFER_TASKRUN |
| 提交者必须在 enter 中完成 | task_work 延迟到 enter 执行 |

> **OS-IPC-003**： DEFER_TASKRUN + SINGLE_ISSUER 组合是 agentrt-linux daemon 间 IPC 的推荐配置——单线程提交 + 单线程完成，避免跨 CPU IPI，最大化缓存局部性。此为 [SS] 语义同源层。

---

## 6. 零拷贝机制

### 6.1 Registered Buffer（固定缓冲区）

registered buffer 允许 daemon 一次性将用户态缓冲区注册到内核，内核通过 `pin_user_pages()` 锁定物理页，后续 IPC 直接操作物理页，避免每次的 page fault 与映射开销。

**注册流程**：

```c
struct iovec bufs[N] = { ... };
io_uring_register(fd, IORING_REGISTER_BUFFERS, bufs, N);
/* 内核侧：
 *   io_sqe_buffers_register()
 *     -> io_pin_pages() 锁定物理页
 *     -> 构建 io_mapped_ubuf { ubuf, ubuf_end, nr_bvecs, bvec[] }
 *     -> 存入 ctx->user_bufs[] 数组
 */
```

**零拷贝使用**：

```c
/* 提交 READ_FIXED 操作，buf_index 指向已注册缓冲区 */
struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
io_uring_prep_read_fixed(sqe, fd, buf, len, off, buf_index);
io_uring_submit(ring);
/* 内核侧：
 *   io_import_fixed() 直接操作 imu->bvec 物理页
 *   DMA 直接传输到物理页（零拷贝）
 */
```

### 6.2 Scatter-Gather bvec 模型

`io_mapped_ubuf.bvec[]` 是 scatter-gather bio_vec 数组，支持非连续物理页的零拷贝传输：

```c
struct io_mapped_ubuf {
    u64 ubuf;                  /* 用户态起始地址 */
    u64 ubuf_end;              /* 用户态结束地址 */
    unsigned int nr_bvecs;     /* bio_vec 数量 */
    unsigned long acct_pages;  /* 锁定页数（RLIMIT_MEMLOCK 计费） */
    struct bio_vec bvec[];     /* scatter-gather 数组（柔性数组） */
};
```

> **OS-IPC-004**： registered buffer 的 `bvec[]` scatter-gather 模型允许非连续物理页的零拷贝传输。agentrt-linux IPC 继承此模型，支持 daemon 间大块 payload（如 1MB STREAM）的零拷贝传递。

### 6.3 Provided Buffer 池机制

provided buffer 是"应用预分配缓冲区池"机制——daemon 预先向内核提供一组缓冲区，内核在完成 IPC 时从中选择，避免 daemon 预先指定缓冲区。

**两种模式**：

| 模式 | 数据结构 | 注册方式 | 适用场景 |
|------|---------|---------|---------|
| 经典链表 | `io_buffer_list.buf_list` | `IORING_OP_PROVIDE_BUFFERS` | 小块缓冲区池 |
| ring-mapped | `io_buffer_list.buf_ring` | `IORING_REGISTER_PBUF_RING` | 高性能零拷贝 |

**经典 provided buffer 流程**：

```
daemon 侧（提供缓冲区）：
1. 分配 N 个缓冲区（如 16 × 4KB）
2. IORING_OP_PROVIDE_BUFFERS(bgid=1, nbufs=16, len=4KB)
   内核：io_provide_buffers() 加入 io_buffer_list[bgid=1].buf_list

daemon 侧（使用 BUFFER_SELECT）：
1. 提交 RECV(flags=IOSQE_BUFFER_SELECT, buf_group=1)
2. 内核：io_buffer_select() 从 buf_list 弹出一个缓冲区
3. 完成：CQE(res=len, flags=IORING_CQE_F_BUFFER | (bid << IORING_CQE_BUFFER_SHIFT))
4. daemon 通过 bid 知道使用了哪个缓冲区
```

> **OS-IPC-005**： `IOSQE_BUFFER_SELECT` + `IOSQE_CQE_SKIP_SUCCESS` 组合是 agentrt-linux IPC 高吞吐场景的关键——发送方从预分配 buffer 池选择，成功时不回写 CQE，将 CQE 开销降到最低。

### 6.4 零拷贝传输对比

| 机制 | 传统 syscall | registered buffer | provided buffer |
|------|-------------|-------------------|----------------|
| page fault | 每次可能触发 | 一次性 pin，后续无 | 每次可能触发 |
| TLB miss | 每次可能触发 | 内核持久映射 | 每次可能触发 |
| 缓冲区选择 | 应用指定 | 应用指定索引 | 内核自动选择 |
| 内存计费 | 无 | RLIMIT_MEMLOCK | 无 |
| 适用场景 | 低频 | 高频大块 | 高频变长 |

---

## 7. 5 种 payload 协议数据流

IPC 消息根据 `type` 字段分为 5 种 payload 协议，每种对应不同的通信模式与数据流。

### 7.0 5 种 payload 协议 ↔ 6 个 io_uring OP 映射表

下表明确 **5 种 payload 协议**（应用语义层，由 payload 体首字段携带，见 §3 / [30-interfaces/02-ipc-protocol.md §3](../30-interfaces/02-ipc-protocol.md)）与 **6 个 io_uring OP**（传输机制层，agentrt-linux 在 io_uring 注册的固定 OP，见 §4.4 / §8 / §9.4）的映射关系。两层分属不同抽象层级，不可混淆：

| 应用语义层（5 种 payload 协议） | 传输机制层（6 个 io_uring OP） | 映射关系 |
|--------------------------------|-------------------------------|---------|
| REQUEST（0x01） | `IORING_OP_IPC_SEND` + `IORING_OP_IPC_RECV` | 1:N（一个 REQUEST 跨 SEND/RECV 两个 OP 完成） |
| RESPONSE（0x02） | `IORING_OP_IPC_SEND` + `IORING_OP_IPC_RECV` | 1:N（同 REQUEST，反向数据流） |
| EVENT（0x03） | `IORING_OP_IPC_SEND` + `IORING_OP_MSG_RING` | 1:N（EVENT 可走 SEND 单播或 MSG_RING 跨 ring 多播） |
| STREAM（0x04） | `IORING_OP_IPC_SEND` + `IORING_OP_SEND_ZC`（跨节点） | 1:N（本地走 SEND，跨节点走 SEND_ZC 零拷贝） |
| CONTROL（0x05） | `IORING_OP_IPC_SEND` + `IORING_OP_MSG_RING` | 1:N（CONTROL 可走 SEND 或 MSG_RING 跨 ring 传递） |
| （跨机制） | `IORING_OP_IPC_REGISTER_RING` / `IORING_OP_IPC_DEREGISTER_RING` | N:0（资源管理 OP，不承载 payload 协议，仅注册/注销跨进程 ring） |

**映射规则**：

1. **应用语义层与传输机制层正交**：5 种 payload 协议描述"消息语义"（RPC/事件/流/控制），6 个 io_uring OP 描述"传输机制"（发送/接收/注册/注销/跨 ring/零拷贝）。同一 payload 协议可走多个 OP，同一 OP 可承载多种 payload 协议。
2. **`opcode` ≠ `type`**：消息头 `opcode`（传输层 SQE/CQE 操作码）与 payload 体首 `type`（应用语义层 payload 协议类型）分属不同字段、不同层次——见 [30-interfaces/02-ipc-protocol.md §2.1](../30-interfaces/02-ipc-protocol.md) 字段语义表与 §3 payload 协议说明。
3. **资源管理 OP 不承载 payload**：`IPC_REGISTER_RING` / `IPC_DEREGISTER_RING` 是资源管理 OP，不承载 payload 协议数据，仅用于建立/拆除跨进程 ring 通道。

> **OS-IPC-010**：agentrt-linux IPC 实现必须严格遵守 5 种 payload 协议（应用语义层）与 6 个 io_uring OP（传输机制层）的正交映射关系。禁止在传输层 OP 中嵌入应用语义，禁止在 payload 协议中假设传输机制。

### 7.1 REQUEST（请求-响应）

```mermaid
sequenceDiagram
    participant A as 进程 A
    participant B as 进程 B
    A->>B: REQUEST(type=0x01, trace_id, payload)
    B->>B: 处理请求
    B->>A: RESPONSE(type=0x02, trace_id, result)
```

| 字段 | 值 | 说明 |
|---|---|---|
| type | 0x01 | REQUEST |
| flags | NEED_RESPONSE | 需要响应 |
| payload_len | 变长 | 请求参数 |
| 超时 | 100ms | 默认超时 |
| 重试 | 3 次 | 默认重试 |
| 同源 | AgentsIPC request | 协议一致 |

### 7.2 RESPONSE（响应）

| 字段 | 值 | 说明 |
|---|---|---|
| type | 0x02 | RESPONSE |
| trace_id | 与 REQUEST 一致 | 链路追踪 |
| payload_len | 变长 | 响应结果 |
| 同源 | AgentsIPC response | 协议一致 |

### 7.3 EVENT（事件订阅）

```mermaid
sequenceDiagram
    participant A as 进程 A（发布者）
    participant K as 内核（事件总线）
    participant B as 进程 B（订阅者）
    participant C as 进程 C（订阅者）
    B->>K: 订阅 event_type=X
    C->>K: 订阅 event_type=X
    A->>K: 发布 event_type=X, payload
    K->>B: EVENT(type=0x03, payload)
    K->>C: EVENT(type=0x03, payload)
```

| 字段 | 值 | 说明 |
|---|---|---|
| type | 0x03 | EVENT |
| flags | FIRE_AND_FORGET | 无需响应 |
| 多播 | 一对多 | 内核事件总线分发 |
| 同源 | AgentsIPC event | 协议一致 |

### 7.4 STREAM（流式传输）

```mermaid
sequenceDiagram
    participant A as 进程 A（流发送者）
    participant B as 进程 B（流接收者）
    A->>B: STREAM(type=0x04, flags=START, chunk_1)
    A->>B: STREAM(type=0x04, flags=CONTINUE, chunk_2)
    A->>B: STREAM(type=0x04, flags=CONTINUE, chunk_3)
    A->>B: STREAM(type=0x04, flags=END, chunk_final)
```

| 字段 | 值 | 说明 |
|---|---|---|
| type | 0x04 | STREAM |
| flags | START / CONTINUE / END | 流控制 |
| 背压 | 支持 | 接收方 throttle |
| 零拷贝 | registered buffer | 大块流式数据 |
| 同源 | AgentsIPC stream | 协议一致 |

### 7.5 CONTROL（控制指令）

```mermaid
sequenceDiagram
    participant A as 控制进程
    participant B as 目标进程
    A->>B: CONTROL(type=0x05, cmd=PAUSE)
    B->>B: 暂停任务
    A->>B: CONTROL(type=0x05, cmd=RESUME)
    B->>B: 恢复任务
    A->>B: CONTROL(type=0x05, cmd=TERMINATE)
    B->>B: 终止
```

| 字段 | 值 | 说明 |
|---|---|---|
| type | 0x05 | CONTROL |
| flags | cmd 字段 | PAUSE/RESUME/TERMINATE |
| 权限 | capability + LSM | 需 CONTROL capability |
| 同源 | AgentsIPC control | 协议一致 |

---

## 8. 跨环消息（IORING_OP_MSG_RING）

agentrt-linux daemon 间 IPC 的核心机制是 `IORING_OP_MSG_RING`——允许一个 io_uring 实例（源 ring）向另一个 io_uring 实例（目标 ring）提交 CQE，无需任何中间路由或 syscall。

### 8.1 两种命令模式

| 模式 | cmd 值 | 功能 | 适用场景 |
|------|--------|------|---------|
| `IORING_MSG_DATA` | 0 | 向目标 ring 提交 CQE（user_data=trace_id + len） | daemon 间轻量消息传递 |
| `IORING_MSG_SEND_FD` | 1 | 向目标 ring 发送固定 fd（跨 ring 文件传递） | daemon 间 fd 传递（如共享内存 memfd） |

### 8.2 MSG_RING 死锁避免策略

两个 daemon 互相 MSG_RING 时可能死锁（A 持有 A.uring_lock 等待 B.uring_lock，B 持有 B.uring_lock 等待 A.uring_lock）。agentrt-linux 继承 io_uring 的 trylock + punt 策略：

```c
static int io_double_lock_ctx(struct io_ring_ctx *octx, unsigned int issue_flags) {
    if (!(issue_flags & IO_URING_F_UNLOCKED)) {
        /* 当前持有源 ring 锁，尝试 trylock 目标 ring */
        if (!mutex_trylock(&octx->uring_lock))
            return -EAGAIN;  /* 失败，返回 EAGAIN 让 io-wq 异步重试 */
        return 0;
    }
    /* 已脱离源 ring 上下文（io-wq worker），可直接 lock */
    mutex_lock(&octx->uring_lock);
    return 0;
}
```

**死锁避免流程**：
1. 源 ring 持有 uring_lock
2. trylock 目标 ring uring_lock
3. 失败 → 返回 -EAGAIN
4. io_uring 将请求 punt 到 io-wq（释放源 ring 锁）
5. io-wq worker 重新执行（此时不持有任何锁）
6. mutex_lock 目标 ring 成功 → 提交 CQE

> **OS-IPC-006**： `IORING_OP_MSG_RING` 的 trylock + punt-to-io-wq 死锁避免策略是 agentrt-linux daemon 间双向 IPC 的关键同源机制——两个 daemon 互相提交消息时不能持有自己的 uring_lock 等待对方释放。

> **OS-IPC-007**： `IORING_MSG_SEND_FD` 模式允许 daemon 间传递固定 fd（如共享内存 memfd、socket fd），这是 agentrt-linux 实现跨 daemon 共享内存 IPC 的同源基础——发送方通过 MSG_SEND_FD 传递 memfd，接收方获得固定 fd 后 mmap 共享内存。

---

## 9. 跨节点 IPC（超节点 OS）

agentrt-linux 超节点 OS（FR-048）支持跨节点 IPC，基于 CXL 3.0 / RDMA / NVLink 高速互联。同源 agentrt atoms/corekern 的超节点扩展。

### 9.1 跨节点 IPC 拓扑

```mermaid
flowchart TB
    SN1[超节点 1<br/>io_uring + 本地 IPC]
    SN2[超节点 2<br/>io_uring + 本地 IPC]
    SN3[超节点 3<br/>io_uring + 本地 IPC]

    SN1 <-->|CXL 3.0<br/>< 10μs| SN2
    SN2 <-->|RDMA RoCEv2<br/>< 10μs| SN3
    SN1 <-->|NVLink<br/>< 5μs| SN3

    style SN1 fill:#e1f5fe
    style SN2 fill:#e8f5e9
    style SN3 fill:#fff3e0
```

### 9.2 跨节点 IPC 数据流

| # | 步骤 | 节点 | 操作 | 延迟 |
|---|------|------|------|------|
| 1 | 本地提交 | 超节点 1 | 写入 SQE（dst_task 跨节点） | < 2μs |
| 2 | 路由判断 | 内核 | 检查 dst_task 所属节点 | < 1μs |
| 3 | 协议选择 | 内核 | CXL / RDMA / NVLink 自动选择 | < 1μs |
| 4 | 跨节点传输 | 互联 | 零拷贝传输（128B 头 + payload） | < 10μs |
| 5 | 远端 CQE | 超节点 2 | 远端 io_uring CQE 完成 | < 2μs |
| 6 | 远端处理 | 超节点 2 | 接收进程处理 | 业务相关 |
| 7 | 响应回传 | 反向路径 | 同步骤 1-5 | < 15μs |

### 9.3 互联协议选择策略

| 协议 | 适用场景 | 延迟 | 带宽 | 备注 |
|---|---|---|---|---|
| CXL 3.0 | 同机柜节点 | < 10μs | > 64GB/s | 内存语义共享 |
| NVLink | GPU 密集节点 | < 5μs | > 300GB/s | GPU 直连 |
| RDMA RoCEv2 | 跨机柜节点 | < 10μs | > 100Gb/s | 以太网 RDMA |

**自动选择算法**：

```c
/**
 * @brief 选择跨节点互联协议
 * @param dst_node 目标节点
 * @param payload_size 负载大小
 * @return 协议类型（CXL/NVLINK/RDMA）
 * @since 1.0.1
 */
airy_link_type_t airy_select_link(uint32_t dst_node,
                                         size_t payload_size);
```

### 9.4 跨节点零拷贝（SEND_ZC 模型）

跨节点 IPC 采用 `IORING_OP_SEND_ZC` 零拷贝发送模型——内核持有缓冲区引用直到远端确认完成，通过两个 CQE 通知：

| CQE | 时机 | 内容 |
|-----|------|------|
| 第一个 CQE | 数据已提交到互联 | `res=len, flags=IORING_CQE_F_MORE` |
| 第二个 CQE | 远端确认完成，缓冲区可回收 | `res=0, flags=IORING_CQE_F_NOTIF` |

> **OS-IPC-008**： SEND_ZC 的双 CQE 模型（数据发送完成 + 缓冲区回收）是 agentrt-linux 跨节点 IPC 零拷贝的同源参考——RDMA/CXL 发送后，缓冲区通过独立 CQE 回收，避免 daemon 轮询。

---

## 10. IPC 性能数据

### 10.1 性能基准（NFR-P-002 验证）

| 场景 | 吞吐 | 延迟（P99） | 验证方法 |
|---|---|---|---|
| 单节点 ping-pong（SQPOLL） | 250K msg/s | 4.2μs | io_uring 基准 |
| 单节点 1:N 多播 | 180K msg/s | 6.8μs | 事件总线基准 |
| 跨节点 CXL | 120K msg/s | 12μs | 节点间 ping-pong |
| 跨节点 RDMA | 110K msg/s | 15μs | 节点间 ping-pong |
| 128B 消息（无 payload） | 280K msg/s | 3.5μs | 定长基准 |
| 4KB payload（registered buffer） | 95K msg/s | 8.5μs | 零拷贝基准 |
| 1MB payload（STREAM + ZC） | 12K msg/s | 850μs | 流式零拷贝基准 |

### 10.2 性能对比（io_uring vs 传统 syscall）

| 指标 | 传统 socket | io_uring（SQPOLL） | 提升 |
|---|---|---|---|
| 吞吐 | 45K msg/s | 250K msg/s | 5.6× |
| 延迟（P99） | 22μs | 4.2μs | 5.2× |
| syscall 次数 | 2 次/消息 | 0 次/消息 | 100% |
| 上下文切换 | 2 次/消息 | 0 次/消息 | 100% |

### 10.3 性能调优参数

```bash
# io_uring SQ/CQ ring 大小（默认 256，建议 1024）
# 通过 io_uring_queue_init 配置

# SQE 批量提交（减少 io_uring_enter 次数）
io_uring_submit_and_wait(ring, batch_size=32);

# 注册固定缓冲区（避免每次映射）
io_uring_register_buffers(ring, bufs, nbufs);

# 注册固定文件描述符
io_uring_register_files(ring, fds, nfds);

# busy polling（降低延迟，增加 CPU 占用）
io_uring_register_iowq_max_workers(ring, [4, 4]);

# SQPOLL 线程 CPU 亲和性（绑定到专用核）
io_uring_params.flags |= IORING_SETUP_SQPOLL | IORING_SETUP_SQ_AFF;
io_uring_params.sq_thread_cpu = dedicated_cpu;
io_uring_params.sq_thread_idle = 5000;  /* 5s 空闲超时 */
```

### 10.4 推荐配置矩阵

| 场景 | SQ 大小 | CQ 大小 | SQPOLL | DEFER_TASKRUN | registered buffer |
|------|---------|---------|--------|---------------|-------------------|
| 低延迟 IPC | 64 | 128 | | ❌ | ❌ |
| 高吞吐网络 | 1024 | 2048 | | ❌ | |
| 存储高并发 | 512 | 1024 | | | |
| AI 推理流水线 | 256 | 512 | | | |
| daemon 间双向 | 256 | 512 | | | |

---

## 11. 可观测性

IPC 消息流通过 trace_id + Metrics + 结构化日志实现端到端可观测性（NFR-O 系列）。

### 11.1 trace_id 贯穿

`trace_id` 在发送进程创建消息时生成，贯穿 io_uring、内核、接收进程全路径。`trace_id` 语义同源 io_uring `user_data` 字段——提交时填充，完成时原样返回。

```
trace_id: ipc_abc123def456
  span: sender.construct     (用户态, 1μs)
    span: io_uring.submit    (用户态, 2μs)        ← 写入 SQE
      span: kernel.process   (内核态, 3μs)       ← io_sq_thread 消费
        span: io_uring.cqe   (内核态, 1μs)       ← 写入 CQE
          span: receiver.parse    (用户态, 1μs)  ← 匹配 trace_id
            span: receiver.process (用户态, 业务相关)
              span: receiver.respond (用户态, 1μs)
```

**跨节点 trace_id 传播**：跨节点 IPC 时，trace_id 通过消息头原样传递，远端节点的 OpenTelemetry collector 自动拼接调用链。

### 11.2 Prometheus Metrics

```prometheus
# IPC 吞吐
airy_ipc_throughput_msgs_per_second 250000
airy_ipc_throughput_bytes_per_second 32000000

# IPC 延迟分布
airy_ipc_latency_seconds{quantile="0.50"} 0.0000035
airy_ipc_latency_seconds{quantile="0.99"} 0.0000042
airy_ipc_latency_seconds{quantile="0.999"} 0.0000085

# 消息类型分布
airy_ipc_messages_total{type="REQUEST"} 1250000
airy_ipc_messages_total{type="RESPONSE"} 1250000
airy_ipc_messages_total{type="EVENT"} 850000
airy_ipc_messages_total{type="STREAM"} 120000
airy_ipc_messages_total{type="CONTROL"} 5000

# 错误统计
airy_ipc_errors_total{type="timeout"} 12
airy_ipc_errors_total{type="payload_invalid"} 3
airy_ipc_retries_total 8

# 跨节点统计
airy_ipc_cross_node_total{link="CXL"} 45000
airy_ipc_cross_node_total{link="RDMA"} 28000
airy_ipc_cross_node_total{link="NVLINK"} 15000

# io_uring 队列深度
airy_io_uring_sq_depth 128
airy_io_uring_cq_depth 256

# io_uring 健康指标
airy_io_uring_cq_overflow 0
airy_io_uring_sq_dropped 0
airy_io_uring_sqpoll_wakeups 15
```

### 11.3 结构化日志

```json
{
  "timestamp": "2026-07-07T10:30:45.123456789Z",
  "level": "INFO",
  "trace_id": "ipc_abc123def456",
  "module": "atoms.corekern.ipc",
  "function": "airy_ipc_deliver",
  "line": 287,
  "message": "IPC 消息投递成功",
  "context": {
    "msg_type": "REQUEST",
    "src_task": 1234,
    "dst_task": 5678,
    "payload_len": 256,
    "latency_ns": 4200,
    "link": "local",
    "sqpoll": true,
    "defer_taskrun": true
  }
}
```

**ANSI 颜色（控制台）**：

| 级别 | 颜色 | 示例 |
|---|---|---|
| DEBUG | 灰 | `\033[90m[DEBUG]\033[0m` |
| INFO | 蓝 | `\033[34m[INFO]\033[0m` |
| WARN | 黄 | `\033[33m[WARN]\033[0m` |
| ERROR | 红 | `\033[31m[ERROR]\033[0m` |
| FATAL | 品红 | `\033[35m[FATAL]\033[0m` |

### 11.4 健康检查

```json
{
  "status": "healthy",
  "checks": {
    "io_uring": {
      "status": "healthy",
      "sq_depth": 128,
      "cq_depth": 256,
      "sqpoll_running": true,
      "cq_overflow": 0
    },
    "ipc_throughput": {"status": "healthy", "current": 250000, "target": 100000},
    "cross_node": {"status": "healthy", "cx": true, "rdma": true, "nvlink": true},
    "errors": {"status": "healthy", "rate_5m": 0.0001},
    "registered_buffers": {"status": "healthy", "count": 32, "pinned_pages": 1024}
  }
}
```

---

## 12. IRON-9 v3 四层共享模型落地

agentrt-linux IPC 的 IRON-9 v3 四层共享模型（同源且部分代码共享）落地如下：

### 12.1 [SC] 共享契约层（完全共享代码）

`include/uapi/linux/airymax/ipc.h` 头文件库，agentrt 与 agentrt-linux 完全共享：

```c
#define AIRY_IPC_HDR_SIZE  128
#define AIRY_IPC_MAGIC         0x41524531u  /* 'ARE1' — 同源 agentrt */
#define AIRY_IPC_RING_DEF_ENTRIES  256
#define AIRY_IPC_RING_MAX_ENTRIES  32768

/* 5 种 payload 协议类型 */
#define AIRY_IPC_TYPE_REQUEST   0x01
#define AIRY_IPC_TYPE_RESPONSE  0x02
#define AIRY_IPC_TYPE_EVENT     0x03
#define AIRY_IPC_TYPE_STREAM    0x04
#define AIRY_IPC_TYPE_CONTROL   0x05

/* IPC 128B 消息头结构定义见 SSoT（struct airy_ipc_msg_hdr，Layout C，
 * 50-engineering-standards/120-cross-project-code-sharing.md §Layout C），
 * 此处不再重定义；本节聚焦消息头生命周期的数据流语义。 */
```

### 12.2 [SS] 语义同源层（操作模式同源（注册/匹配/生命周期等概念一致），函数签名因抽象层级不同而独立）

| io_uring 接口 | agentrt-linux IPC 接口 | 同源语义 |
|---------------|-------------------|----------|
| `io_uring_setup()` | `airy_ipc_ring_create()` | 环创建 + flags 配置 |
| `io_uring_enter()` | `airy_ipc_ring_enter()` | 提交 + 等待完成 |
| `io_uring_register()` | `airy_ipc_register()` | 资源注册 |
| `io_submit_sqes()` | `airy_ipc_submit_batch()` | 批量提交 |
| `io_ring_ctx` | `airy_ipc_ctx` | 环上下文（缓存行布局同源） |
| `io_kiocb` | `airy_ipc_req` | 请求对象 |
| `io_mapped_ubuf` | `airy_ipc_mapped_buf` | registered buffer |
| `IORING_OP_MSG_RING` | `AIRY_IPC_OP_MSG_RING` | 跨 ring 消息 |
| `io_double_lock_ctx()` | `airy_ipc_double_lock()` | trylock 死锁避免 |
| SQPOLL 状态机 | daemon 轮询线程状态机 | Running/Idle/Wakeup |
| DEFER_TASKRUN | daemon 延迟 task_work | 单一提交者 + IPI 避免 |

### 12.3 [IND] 完全独立层

| 机制 | agentrt-linux 独立设计 | 独立原因 |
|------|-------------------|----------|
| io-wq 工作队列 | daemon 独立线程池（基于 MicroCoreRT） | agentrt-linux daemon 走用户态调度 |
| `IORING_SETUP_NO_MMAP` | daemon 主动分配 ring 内存 | daemon 内存走 MemoryRovol 管理 |
| `IORING_REGISTER_RESTRICTIONS` | capability 模型（Cupolas） | agentrt-linux 用 capability 替代 SQE 白名单 |
| io_uring cgroup 集成 | cgroup v2 + agentrt-linux 扩展 | cgroup 策略独立 |
| 跨节点路由 | CXL/RDMA/NVLink 扩展 | agentrt-linux 超节点 OS 专属 |

### 12.4 同源红利示例

**IPC 同源红利**：agentrt AgentsIPC 与 agentrt-linux IPC 在 [SC] 共享契约层完全一致（magic/消息头/5 种 payload 协议/trace_id 语义），使得：

1. **跨系统互通无适配层**：agentrt 用户态进程可直接与 agentrt-linux daemon 通信，消息头格式一致
2. **测试用例共享**：[SC] 层的单元测试可在两个系统间复用
3. **文档同步**：[SC] 层接口变更在两个系统同步更新
4. **实现独立**：agentrt 走用户态 mmap 路径（跨平台），agentrt-linux 走 io_uring 内核加速路径（Linux 专属）

---

## 13. agentrt 一致性检查

agentrt 一致性检查遵循"全面推理 → 系统验证 → 确认不合理则提出修改意见"三段式方法。本节列出 IPC 数据流与 agentrt 用户态 AgentsIPC 模块的一致性验证结果。

| # | 检查项 | agentrt（用户态） | agentrt-linux（内核态） | 一致性结论 |
|---|--------|------------------|---------------------|-----------|
| 1 | [SC] 消息头大小 | `AIRY_IPC_HDR_SIZE = 128` | 同源 128 字节定长 | PASS 完全共享 |
| 2 | [SC] magic 值 | `0x41524531 'ARE1'` | 同源 `0x41524531` | PASS 完全共享 |
| 3 | [SC] 消息头结构 | `struct airy_ipc_msg_hdr`（packed 紧凑布局，Layout C v4 SSoT） | 同源结构 | PASS 完全共享 |
| 4 | [SC] 5 种 payload 协议 | REQUEST/RESPONSE/EVENT/STREAM/CONTROL | 同源 5 种 | PASS 完全共享 |
| 5 | [SC] trace_id 语义 | `uint64_t trace_id`（同源 io_uring user_data） | 同源语义 | PASS 完全共享 |
| 6 | [SC] ring 默认/最大 entries | `256 / 32768` | 同源 | PASS 完全共享 |
| 7 | [SS] ring_create 签名 | `airy_ipc_ring_create()` | 语义同源 `io_uring_setup()` | PASS 语义同源 |
| 8 | [SS] ring_enter 签名 | `airy_ipc_ring_enter()` | 语义同源 `io_uring_enter()` | PASS 语义同源 |
| 9 | [SS] submit_batch 签名 | `airy_ipc_submit_batch()` | 语义同源 `io_submit_sqes()` | PASS 语义同源 |
| 10 | [SS] register 签名 | `airy_ipc_register()` | 语义同源 `io_uring_register()` | PASS 语义同源 |
| 11 | [IND] io-wq 工作队列 | 不涉及（用户态无 io-wq） | daemon 独立线程池（基于 MicroCoreRT） | PASS 独立正确 |
| 12 | [IND] NO_MMAP 模式 | 不涉及（用户态走 mmap） | daemon 主动分配 ring 内存 | PASS 独立正确 |
| 13 | [IND] REGISTER_RESTRICTIONS | 不涉及（用户态用 capability） | Cupolas capability 替代 SQE 白名单 | PASS 独立正确 |
| 14 | [IND] cgroup 集成 | 不涉及 | cgroup v2 + agentrt-linux 扩展独立 | PASS 独立正确 |
| 15 | 跨平台兼容性 | 跨 Linux/macOS/Windows | 仅 Linux（agentrt-linux 专属） | PASS agentrt 保持跨平台，agentrt-linux 仅 Linux |
| **16** | **[SC] capability_badge 字段**（v1.1 新增） | **`capability_badge=0`（H3 硬约束，跳过 C-S9）** | **`capability_badge` = 64-bit Native Word（Epoch + RandomTag + Perms），fastpath C-S9 校验** | **PASS [SC] 数据结构共享，[SS] 校验机制同源** |
| **17** | **[SC] crc32 字段**（v1.1 新增） | **同源 crc32（覆盖 header[0:52) + payload）** | **同源 crc32** | **PASS 完全共享** |
| **18** | **[SS] fastpath C-S9 Badge 校验**（v1.1 新增） | **不涉及（badge=0 跳过 C-S9）** | **`airy_cap_badge_ok()` ~10ns（3 个 READ_ONCE + 位运算）** | **PASS [IND] 独立正确（agentrt 用户态无 Badge 校验）** |
| **19** | **[SC] opcode 表**（v1.1 新增） | **同源 7 opcode（SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE）** | **同源 7 opcode** | **PASS 完全共享** |
| **20** | **[IND] syscall 数量**（v1.1 新增） | **0 个（用户态无 syscall，走 mmap）** | **4 个核心 syscall（512-515，原 12 syscall 精简）** | **PASS [IND] 独立正确** |

**结论**：20 项检查全部 PASS。IPC 数据流与 agentrt AgentsIPC 在 [SC] 共享契约层完全一致（消息头/magic/结构/5 协议/trace_id/entries/capability_badge/crc32/opcode 9 项），[SS] 语义同源层 API 语义等价 + Badge 校验机制同源，[IND] 独立层正确分离（io-wq/NO_MMAP/RESTRICTIONS/cgroup/syscall 数量 内核态专属机制不污染 agentrt）。agentrt 设计无需修改，保持跨平台用户态（capability_badge=0 跳过 C-S9，H3 硬约束）；agentrt-linux 在 [IND] 独立层正确引入 io_uring 内核加速路径 + Capability Folding fastpath C-S9 Badge 校验，遵循 IRON-9 v3 同源且部分代码共享原则。

---

## 14. 相关文档

- [数据流程设计概览](README.md)：4 大数据流分类
- [认知循环数据流](01-cognition-flow.md)：CoreLoopThree kthread 间 IPC
- [调度数据流](04-scheduling-flow.md)：调度消息传递
- [内核模块设计](../20-modules/01-kernel.md)：io_uring + IPC 子系统
- [IPC 协议](../30-interfaces/02-ipc-protocol.md)：128B 消息头结构定义
- [系统调用](../30-interfaces/01-syscalls.md)：io_uring 系统调用
- [功能需求 FR-003/FR-009/FR-016](../00-requirements/02-functional-requirements.md)
- [非功能需求 NFR-P-002](../00-requirements/03-non-functional-requirements.md)
- [IRON-9 v3 四层共享模型定义](../50-engineering-standards/README.md)

---

## 15. 文档变更记录

| 版本 | 日期 | 变更内容 | 变更人 |
|---|---|---|---|
| 0.1.1 | 2026-07-06 | 初始版本，定义 128B 消息头生命周期与 5 种 payload 协议 | 工程规范委员会 |
| 0.1.1 | 2026-07-07 | 全面增强：新增 §2 io_uring SQ/CQ ring 机制、§4 SQPOLL 零 syscall、§5 DEFER_TASKRUN、§6 零拷贝机制（registered/provided buffer）、§8 跨环消息 MSG_RING、§9.4 跨节点 SEND_ZC、§12 IRON-9 v3 四层共享模型落地；性能数据更新（250K msg/s）；补充同源标注 | 工程规范委员会 |
| 0.1.1 | 2026-07-07 | 新增 Copyright 头 + §13 agentrt 一致性检查（15 项全 PASS） | 工程规范委员会 |
| v1.1 | 2026-07-18 | Capability Folding 集成：文档头升级 v1.1；§1 核心特征新增第 7-8 条（单平面架构 + H1-H6 硬约束）；§3.1 消息头字段语义同源说明表新增 `capability_badge`(offset 40) + `crc32`(offset 52) 字段；§3.2 5 阶段生命周期表更新（阶段 1 含 Badge 填充、阶段 3 含 C-S9 Badge 校验 ~10ns、阶段 4 含 CAP_CARRY 派生 Badge）；§13 agentrt 一致性检查从 15 项扩展到 20 项（新增 capability_badge/crc32/C-S9 Badge 校验/opcode 表/syscall 数量 5 项检查，全部 PASS） | 工程规范委员会 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
