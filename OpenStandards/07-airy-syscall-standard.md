Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax 系统调用标准

> **文档定位**: Airymax 开放标准
> **版本**: 1.0（开放标准草案）
> **最后更新**: 2026-07-11
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 一、范围与目的

本标准定义 AI Agent 系统调用的开放契约，覆盖：

- 双平面架构（控制面 syscall + 数据面 io_uring）
- Syscall 编号体系（12 核心 + 12 预留 = 24 槽位）
- Capability Invocation 统一入口（`airy_sys_call`）
- 8 IPC 原语 + 3 控制原语 + 1 通知原语
- 错误码标准（`AIRY_E*` 前缀）
- 性能约束（syscall < 1 μs，io_uring IPC < 10 μs）

本标准面向：

- **运行时实现者**：实现兼容的 Agent 系统调用接口（用户态运行时、操作系统发行版）。
- **Agent 作者**：理解系统调用语义与错误处理。
- **工具链开发者**：构建系统调用追踪、性能剖析工具。
- **集成方**：将第三方 Agent 平台接入 Airymax 生态。

本标准遵循 IRON-9 v2 [SC] 共享契约层——syscall 编号、签名、错误码在 agentrt 与 agentrt-linux 之间完全共享。数据面 I/O 由 io_uring 处理，不占用 syscall 槽位。

### 1.1 术语

| 术语 | 含义 |
|------|------|
| 系统调用（Syscall） | 用户态向内核请求服务的同步入口 |
| 控制面 | 低频、需同步语义的操作路径（capability invocation、策略设置） |
| 数据面 | 高频、可异步、需零拷贝的操作路径（IPC 收发、记忆迁移） |
| Capability Invocation | 通过 capability 令牌进行操作分发的统一调用模型（seL4 风格） |
| io_uring | Linux 异步 I/O 框架，通过共享内存 SQ/CQ ring 实现零 syscall 数据面 |
| CNode | Capability Node，capability 存储与操作的容器 |
| Endpoint | IPC 端点，消息传递的 capability 目标 |

### 1.2 规范用词

沿用 RFC 2119 用词（必须 / 不得 / 应当 / 不应当 / 可以）。

### 1.3 设计原则

本标准遵循以下设计原则：

| 原则 | 在系统调用中的体现 |
|------|-------------------|
| seL4 最小完备 | 12 核心 syscall = seL4 8 IPC 原语 + 3 控制原语 + 1 通知原语，无冗余 |
| 机制在内核，策略在用户态 | syscall 仅提供机制，策略由用户态 eBPF struct_ops 定义 |
| Capability 统一入口 | `airy_sys_call` 替代 task_submit/cancel/lsm_ctl/wasm_load 等独立 syscall |
| 数据面零 syscall | 高频 I/O 走 io_uring 共享内存 ring，不陷入内核 |
| ABI 永久稳定 | syscall 编号在 MAJOR 版本内不可变更 |

---

## 二、双平面架构

### 2.1 控制面（12 核心 syscall）

控制面用于低频、需同步语义的操作。任何兼容实现**必须**提供以下 12 个核心系统调用（AOS-STD-SYS-001）：

| 编号 | 分类 | Syscall | 借鉴来源 |
|------|------|---------|---------|
| 0 | IPC 原语 | `airy_sys_call` | seL4 Call（统一 capability invocation） |
| 1 | IPC 原语 | `airy_sys_send` | seL4 Send |
| 2 | IPC 原语 | `airy_sys_recv` | seL4 Recv |
| 3 | IPC 原语 | `airy_sys_nbsend` | seL4 NBSend |
| 4 | IPC 原语 | `airy_sys_nbrecv` | seL4 NBRecv |
| 5 | IPC 原语 | `airy_sys_reply_recv` | seL4 ReplyRecv |
| 6 | IPC 原语 | `airy_sys_yield` | seL4 Yield |
| 7 | 控制原语 | `airy_sys_rovol_ctl` | Agent 领域最小需求 |
| 8 | 控制原语 | `airy_sys_sched_ctl` | Agent 领域最小需求 |
| 9 | 控制原语 | `airy_sys_clt_notify` | Agent 领域最小需求 |
| 10 | IPC 原语 | `airy_sys_reply` | seL4 Reply（独立回复） |
| 11 | 通知原语 | `airy_sys_notify` | seL4 Notification |

> **说明**：LsmCtl（安全策略加载）与 WasmLoad（模块加载）已归入 `airy_sys_call` 统一 capability invocation 入口——通过 security capability 和 module capability 进行类型分发，无需独立 syscall。`airy_sys_reply` 完齐 seL4 8 个 activity syscall（Reply 独立回复，无需等待下一条消息）。

### 2.2 数据面（io_uring，零 syscall）

| 路径 | 延迟量级 | 用途 |
|------|---------|------|
| 用户态 → SQE → 内核 → CQE → 用户态 | < 10 μs（P99） | IPC 收发、记忆迁移、流式数据 |

数据面**不得**占用 syscall 槽位（AOS-STD-SYS-002）。高频路径**必须**走 io_uring 零拷贝通道（共享内存 SQ/CQ ring）。

### 2.3 调用路径选择规则

| 场景 | 平面 | 选择依据 |
|------|------|---------|
| Capability invocation（task/cap/lsm/wasm） | 控制面 | 需同步语义、低频 |
| 调度策略设置 | 控制面 | 需同步语义、低频 |
| 认知阶段通知 | 控制面 | 需同步语义、低频 |
| IPC 消息收发（高频） | 数据面 | 可异步、需零拷贝 |
| 记忆迁移（大块） | 数据面 | 可异步、需零拷贝 |
| 流式数据处理 | 数据面 | 可异步、需零拷贝 |

---

## 三、Syscall 编号体系

### 3.1 编号分配

Agent 专用系统调用采用 **12 核心 + 12 预留 = 24 槽位** 方案（AOS-STD-SYS-003）：

| 编号 | 符号 | 分类 | 说明 |
|------|------|------|------|
| 0 | `AIRY_SYS_CALL` | IPC 原语 | 统一 capability invocation（含 LSM/Wasm 加载） |
| 1 | `AIRY_SYS_SEND` | IPC 原语 | 阻塞同步发送 |
| 2 | `AIRY_SYS_RECV` | IPC 原语 | 阻塞同步接收 |
| 3 | `AIRY_SYS_NBSEND` | IPC 原语 | 非阻塞发送 |
| 4 | `AIRY_SYS_NBRECV` | IPC 原语 | 非阻塞接收 |
| 5 | `AIRY_SYS_REPLY_RECV` | IPC 原语 | 回复并等待下一条 |
| 6 | `AIRY_SYS_YIELD` | IPC 原语 | 让出 CPU |
| 7 | `AIRY_SYS_ROVOL_CTL` | 控制原语 | 记忆卷载控制（snapshot/restore/migrate/tier） |
| 8 | `AIRY_SYS_SCHED_CTL` | 控制原语 | 调度策略配置（set/get） |
| 9 | `AIRY_SYS_CLT_NOTIFY` | 控制原语 | CoreLoopThree 阶段通知 + kthread 注册 |
| 10 | `AIRY_SYS_REPLY` | IPC 原语 | 独立回复（不等待下一条） |
| 11 | `AIRY_SYS_NOTIFY` | 通知原语 | 异步通知信号 |
| 12-23 | 预留 | — | 未来扩展 |

### 3.2 编号稳定性

- 编号在 MAJOR 版本内**不可变更**（AOS-STD-SYS-004，L1 稳定级）。
- 新增调用**只能**追加到预留段末尾（12-23），**不得**复用已废弃编号（AOS-STD-SYS-005）。
- 废弃调用保留编号但返回 `-AIRY_ENOSYS`，并在 kernel-doc 注释中标注 `@deprecated`（AOS-STD-SYS-006）。

### 3.3 命名前缀

所有 Agent 专用系统调用 C 符号使用小写 `airy_sys_` 前缀，编号宏使用 `AIRY_SYS_` 前缀（AOS-STD-SYS-007）。

---

## 四、Capability Invocation 统一入口

### 4.1 设计模型

`airy_sys_call` 是统一的 capability invocation 入口，借鉴 seL4 的 `decodeInvocation` 模式（AOS-STD-SYS-008）：

- 操作类型（task_submit / capability_request / mint / revoke / lsm_policy_load / wasm_load 等）由 **capability 类型**决定，而非 syscall 编号。
- capability 通过 CNode 操作（Copy / Mint / Move / Revoke）在进程间传递，无需独立的 `capability_request` syscall。
- 安全策略加载通过 `airy_sys_call(security_cap, &msg)` 完成。
- Wasm 模块加载通过 `airy_sys_call(module_cap, &msg)` 完成。

### 4.2 Capability 类型分发

| Capability 类型 | 操作 | 消息格式 |
|----------------|------|---------|
| `CAP_TASK` | submit / cancel / status / query | IPC 消息（128B header + payload） |
| `CAP_ENDPOINT` | send / recv / nbsend / nbrecv / reply | IPC 消息 |
| `CAP_CNODE` | copy / mint / move / revoke / delete / rotate | CNode 操作消息 |
| `CAP_SECURITY` | policy_load / policy_unload / audit | 安全策略消息 |
| `CAP_MODULE` | wasm_load / wasm_unload / wasm_instantiate | 模块加载消息 |
| `CAP_NOTIFICATION` | signal / wait / poll | 通知消息 |

### 4.3 调用示例

```c
/* Unified capability invocation: seL4 Call style */
int ret = airy_sys_call(cap, &msg);
if (ret < 0) {
    log_write(LOG_ERROR, "cap invocation failed: errno=%d", ret);
    return ret;
}
```

---

## 五、C 接口标准

### 5.1 导出宏

所有系统调用通过 `AIRY_API` 宏导出，遵循 Linux 内核编码规范（Tab=8, snake_case, kernel-doc 注释）。

```c
#if defined(__GNUC__) && __GNUC__ >= 4
    #define AIRY_API __attribute__((visibility("default")))
#else
    #define AIRY_API
#endif
```

### 5.2 IPC 原语（8 个）

#### 5.2.1 airy_sys_call — 统一 capability invocation

```c
/**
 * airy_sys_call - Unified capability invocation (seL4 Call)
 * @cap:  Capability to invoke.
 * @msg:  IPC message (128B header + payload).
 *
 * Single entry point for all capability-based operations.
 * The capability type determines operation dispatch.
 *
 * Operations: task submit/cancel/status, capability mint/revoke/derive,
 * IPC rendezvous, LSM policy load, Wasm module load, notification delivery.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_call(cap_t cap,
                                 const struct airy_ipc_msg_hdr *msg);
```
（AOS-STD-SYS-009）

#### 5.2.2 airy_sys_send — 阻塞同步发送

```c
/**
 * airy_sys_send - Blocking synchronous send
 * @cap:  Endpoint capability.
 * @msg:  IPC message to send.
 *
 * Blocks until the message is delivered to a receiver (rendezvous).
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_send(cap_t cap,
                                 const struct airy_ipc_msg_hdr *msg);
```
（AOS-STD-SYS-010）

#### 5.2.3 airy_sys_recv — 阻塞同步接收

```c
/**
 * airy_sys_recv - Blocking synchronous receive
 * @cap:  Endpoint capability.
 * @msg:  IPC message buffer for received data.
 *
 * Blocks until a sender delivers a message.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_recv(cap_t cap,
                                 struct airy_ipc_msg_hdr *msg);
```
（AOS-STD-SYS-011）

#### 5.2.4 airy_sys_nbsend — 非阻塞发送

```c
/**
 * airy_sys_nbsend - Non-blocking send
 * @cap:  Endpoint capability.
 * @msg:  IPC message to send.
 *
 * Returns immediately; if no receiver is waiting, returns -AIRY_EAGAIN.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_nbsend(cap_t cap,
                                   const struct airy_ipc_msg_hdr *msg);
```
（AOS-STD-SYS-012）

#### 5.2.5 airy_sys_nbrecv — 非阻塞接收

```c
/**
 * airy_sys_nbrecv - Non-blocking receive
 * @cap:  Endpoint capability.
 * @msg:  IPC message buffer for received data.
 *
 * Returns immediately; if no sender is waiting, returns -AIRY_EAGAIN.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_nbrecv(cap_t cap,
                                   struct airy_ipc_msg_hdr *msg);
```
（AOS-STD-SYS-013）

#### 5.2.6 airy_sys_reply_recv — 回复并接收下一条

```c
/**
 * airy_sys_reply_recv - Reply to current message and wait for next
 * @cap:    Reply capability.
 * @reply:  Reply message to send.
 * @recv:   Buffer for next received message.
 *
 * Combined reply + receive in one syscall (seL4 ReplyRecv).
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_reply_recv(cap_t cap,
                                       const struct airy_ipc_msg_hdr *reply,
                                       struct airy_ipc_msg_hdr *recv);
```
（AOS-STD-SYS-014）

#### 5.2.7 airy_sys_yield — 让出 CPU

```c
/**
 * airy_sys_yield - Yield CPU
 *
 * Voluntarily yields the CPU to the scheduler.
 *
 * Return: 0 on success.
 */
AIRY_API int airy_sys_yield(void);
```
（AOS-STD-SYS-015）

#### 5.2.8 airy_sys_reply — 独立回复

```c
/**
 * airy_sys_reply - Reply to current message without waiting (seL4 Reply)
 * @reply:  Reply capability.
 * @msg:    Reply message to send.
 *
 * Sends a reply without blocking for a new message.
 * Used when the server has no more work to accept immediately.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_reply(cap_t reply,
                                  const struct airy_ipc_msg_hdr *msg);
```
（AOS-STD-SYS-016）

### 5.3 控制原语（3 个）

#### 5.3.1 airy_sys_rovol_ctl — 记忆卷载控制

```c
/**
 * airy_sys_rovol_ctl - Memory snapshot/restore/migrate/tier control
 * @op:   Operation (0=snapshot, 1=restore, 2=migrate, 3=tier_set).
 * @pid:  Target process ID.
 * @arg:  Operation-specific argument.
 *
 * Unified memory control interface covering MemoryRovol L1-L4.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_rovol_ctl(uint32_t op, uint32_t pid,
                                      uint64_t arg);
```
（AOS-STD-SYS-017）

#### 5.3.2 airy_sys_sched_ctl — 调度策略配置

```c
/**
 * airy_sys_sched_ctl - Scheduling policy configuration
 * @op:          Operation (0=set, 1=get).
 * @cgroup_path:  Target cgroup path.
 * @policy:      Policy name (scx_realtime / scx_batch / scx_interactive / scx_agent).
 *
 * Unified scheduling control via sched_ext (SCHED_EXT=7).
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_sched_ctl(uint32_t op,
                                      const char *cgroup_path,
                                      const char *policy);
```
（AOS-STD-SYS-018）

#### 5.3.3 airy_sys_clt_notify — CoreLoopThree 阶段通知

```c
/**
 * airy_sys_clt_notify - CoreLoopThree phase notification + kthread control
 * @task_id:  Agent task ID (0 for kthread register).
 * @phase:    Phase (0=perception, 1=thinking, 2=action) or kthread op.
 *
 * Notifies the kernel of CoreLoopThree phase transitions and
 * manages kthread registration for the cognition data flow.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_clt_notify(int task_id, uint32_t phase);
```
（AOS-STD-SYS-019）

### 5.4 通知原语（1 个）

```c
/**
 * airy_sys_notify - Async notification signal (seL4 Notification)
 * @cap:  Notification capability.
 *
 * Signals a notification object. Wakes any waiting agents.
 * Binary semaphore semantics: signal sets the notification,
 * wait (via airy_sys_call with notification cap) clears it atomically.
 *
 * Return: 0 on success, negative errno on failure.
 */
AIRY_API int airy_sys_notify(cap_t cap);
```
（AOS-STD-SYS-020）

---

## 六、错误码标准

### 6.1 错误码定义

错误码统一使用 `AIRY_E*` 前缀，负值返回（AOS-STD-SYS-021）。错误码对齐 `airy_errno.h`，与 agentrt 同源且部分代码共享（IRON-9 v2 [SC]）。

| 错误码 | 值 | 含义 | 触发场景 |
|--------|-----|------|---------|
| `AIRY_EOK` | 0 | 成功 | 调用成功 |
| `AIRY_EINVAL` | -1 | 无效参数 | 参数为 NULL 或非法值 |
| `AIRY_ENOMEM` | -2 | 内存不足 | 内核分配失败 |
| `AIRY_ENOSYS` | -3 | 未实现 | 编号未实现或已废弃 |
| `AIRY_EPERM` | -4 | 权限不足 | capability 令牌缺失 |
| `AIRY_ENOENT` | -5 | 资源不存在 | 任务 ID / 快照 ID 不存在 |
| `AIRY_EAGAIN` | -6 | 重试 | io_uring 队列满，需重试 |
| `AIRY_EMSGSIZE` | -7 | 消息过大 | payload 超过最大长度 |
| `AIRY_EBADF` | -8 | 描述符错误 | ring fd / capability 句柄无效 |
| `AIRY_EBUSY` | -9 | 资源繁忙 | 任务正在迁移，无法快照 |
| `AIRY_ENOTSUP` | -10 | 不支持 | 硬件不支持（如无 CXL 设备） |
| `AIRY_ETIMEDOUT` | -11 | 超时 | 调度等待超时 |
| `AIRY_ECONFLICT` | -12 | 状态冲突 | 任务状态不允许当前操作 |

### 6.2 错误码使用规范

```c
/* 正确：检查返回值并传递错误码 */
int ret = airy_sys_call(cap, &msg);
if (ret < 0) {
    log_write(LOG_ERROR, "call failed: errno=%d (%s)",
              ret, airy_strerror(ret));
    return ret;
}
```

### 6.3 错误码稳定性

- `AIRY_E*` 错误码值在 MAJOR 版本内**不可变更**（AOS-STD-SYS-022，L1 稳定级）。
- 新增错误码**只能**追加到末尾，**不得**复用已废弃值（AOS-STD-SYS-023）。
- 错误码字符串描述通过 `airy_strerror()` 提供，与 agentrt 同源保持一致（AOS-STD-SYS-024）。

### 6.4 错误码转换

- 与 Linux 标准 `errno` 的转换通过 `airy_errno_to_linux()` 工具函数完成。
- 与 agentrt 应用层错误码的转换通过 `airy_errno_to_app()` 工具函数完成。
- 转换表在 `airy_errno.h` 中以静态数组定义。

---

## 七、性能约束

### 7.1 延迟预算

| 约束 ID | 指标 | 阈值 | 测量方法 |
|---------|------|------|---------|
| AOS-STD-SYS-025 | 系统调用开销 | < 1 μs（P99） | strace + perf 测量 |
| AOS-STD-SYS-026 | io_uring IPC 往返延迟 | < 10 μs（P99） | SQE 提交到 CQE 完成 |
| AOS-STD-SYS-027 | Agent 任务调度延迟 | < 100 ms（P99） | capability invocation 到任务首次执行 |

### 7.2 优先级与延迟预算

| 任务类别 | cgroup | 优先级范围 | 延迟预算（P99） | 典型场景 |
|---------|---------|-----------|----------------|---------|
| 实时控制 | `realtime.slice` | 0-49 | < 1 ms | 具身智能运动控制 |
| 交互响应 | `interactive.slice` | 50-99 | < 10 ms | 用户对话补全 |
| Agent 认知 | `agent.slice` | 100-119 | < 100 ms | CoreLoopThree 思考 |
| 批处理推理 | `batch.slice` | 120-139 | < 1 s | LLM 批量推理 |

超出延迟预算的任务由 sub-scheduler 触发 `AIRY_ETIMEDOUT` 错误码（AOS-STD-SYS-028）。

### 7.3 性能回归保护

- 每次提交**必须**运行调度延迟微基准（AOS-STD-SYS-029）。
- 与基线对比，调度延迟退化 > 5% **必须**自动打回（AOS-STD-SYS-030）。

### 7.4 性能剖析方法

| 工具 | 用途 | 示例 |
|------|------|------|
| `perf trace` | 系统调用延迟直方图 | `perf trace -e airy_sys_* --summary` |
| `bpftrace` | 动态追踪系统调用参数与耗时 | `bpftrace -e 'tracepoint:airymax:sys_call { ... }'` |
| `perf stat` | 调度器与 cache 事件计数 | `perf stat -e sched:* airyctl bench ipc` |
| `io_uring-bench` | io_uring IPC 吞吐基准 | `io_uring-bench --zerocopy --msg-size 128` |

---

## 八、标准条目注册表

| 编号 | 名称 | 稳定级 |
|------|------|--------|
| AOS-STD-SYS-001 | 12 核心 syscall 完整集 | L1 |
| AOS-STD-SYS-002 | 数据面零 syscall | L1 |
| AOS-STD-SYS-003 | 24 槽位编号方案 | L1 |
| AOS-STD-SYS-004 | 编号 MAJOR 版本内不可变更 | L1 |
| AOS-STD-SYS-005 | 新增调用只能追加到预留段 | L1 |
| AOS-STD-SYS-006 | 废弃调用返回 ENOSYS | L1 |
| AOS-STD-SYS-007 | airy_sys_ / AIRY_SYS_ 命名前缀 | L1 |
| AOS-STD-SYS-008 | capability invocation 统一入口 | L1 |
| AOS-STD-SYS-009 | airy_sys_call 签名 | L1 |
| AOS-STD-SYS-010 | airy_sys_send 签名 | L1 |
| AOS-STD-SYS-011 | airy_sys_recv 签名 | L1 |
| AOS-STD-SYS-012 | airy_sys_nbsend 签名 | L1 |
| AOS-STD-SYS-013 | airy_sys_nbrecv 签名 | L1 |
| AOS-STD-SYS-014 | airy_sys_reply_recv 签名 | L1 |
| AOS-STD-SYS-015 | airy_sys_yield 签名 | L1 |
| AOS-STD-SYS-016 | airy_sys_reply 签名 | L1 |
| AOS-STD-SYS-017 | airy_sys_rovol_ctl 签名 | L1 |
| AOS-STD-SYS-018 | airy_sys_sched_ctl 签名 | L1 |
| AOS-STD-SYS-019 | airy_sys_clt_notify 签名 | L1 |
| AOS-STD-SYS-020 | airy_sys_notify 签名 | L1 |
| AOS-STD-SYS-021 | AIRY_E* 错误码前缀 | L1 |
| AOS-STD-SYS-022 | 错误码值 MAJOR 版本内不可变更 | L1 |
| AOS-STD-SYS-023 | 新增错误码只能追加 | L1 |
| AOS-STD-SYS-024 | airy_strerror() 描述一致性 | L1 |
| AOS-STD-SYS-025 | syscall 开销 < 1 μs | L2 |
| AOS-STD-SYS-026 | io_uring IPC < 10 μs | L2 |
| AOS-STD-SYS-027 | Agent 调度延迟 < 100 ms | L2 |
| AOS-STD-SYS-028 | 超延迟预算返回 ETIMEOUT | L1 |
| AOS-STD-SYS-029 | 调度延迟微基准 | L2 |
| AOS-STD-SYS-030 | 延迟退化 > 5% 自动打回 | L2 |

---

## 九、兼容性测试要求

| 测试 ID | 测试内容 | 通过条件 |
|---------|---------|---------|
| SYS-T-001 | 12 核心 syscall 编号 | 编号 0-11 精确匹配 SSoT |
| SYS-T-002 | airy_sys_call 签名 | 参数类型与返回值精确匹配 |
| SYS-T-003 | IPC 8 原语完整性 | Call/Send/Recv/NBSend/NBRecv/ReplyRecv/Yield/Reply 全部可调用 |
| SYS-T-004 | 控制原语完整性 | RovolCtl/SchedCtl/CltNotify 全部可调用 |
| SYS-T-005 | 通知原语完整性 | Notify 可调用且异步唤醒等待者 |
| SYS-T-006 | 非阻塞语义 | NBSend/NBRecv 无接收者/发送者时返回 EAGAIN |
| SYS-T-007 | ReplyRecv 原子性 | 回复与接收在单次 syscall 内完成 |
| SYS-T-008 | Reply 独立性 | Reply 不阻塞等待下一条消息 |
| SYS-T-009 | Capability 类型分发 | 不同 cap 类型产生不同操作 |
| SYS-T-010 | 错误码范围 | 0 成功，-1 至 -12 对应定义错误码 |
| SYS-T-011 | 废弃编号返回 ENOSYS | 预留段（12-23）未实现编号返回 ENOSYS |
| SYS-T-012 | io_uring 数据面 | 高频 IPC 走 io_uring 不占用 syscall |
| SYS-T-013 | 性能约束 | syscall 开销 < 1 μs（P99） |
| SYS-T-014 | 调度延迟 | Agent 任务调度延迟 < 100 ms（P99） |

---

## 十、最小示例

### 10.1 Capability Invocation

```c
#include <airy/ipc.h>
#include <airy/syscalls.h>

/* 通过 airy_sys_call 提交任务 */
struct airy_ipc_msg_hdr msg = {0};
msg.magic   = AIRY_IPC_MAGIC;   /* 0x41524531 'ARE1' */
msg.opcode  = AIRY_OP_TASK_SUBMIT;
msg.cap_id  = task_cap;
/* ... 填充 payload ... */

int ret = airy_sys_call(task_cap, &msg);
if (ret == -AIRY_EPERM) {
    /* capability 权限不足 */
} else if (ret == -AIRY_EAGAIN) {
    /* 需重试 */
}
```

### 10.2 IPC 发送与接收

```c
/* 阻塞发送 */
int ret = airy_sys_send(endpoint_cap, &msg);
if (ret < 0) {
    return ret;  /* 错误码传递 */
}

/* 非阻塞接收 */
ret = airy_sys_nbrecv(endpoint_cap, &recv_buf);
if (ret == -AIRY_EAGAIN) {
    /* 无消息，稍后重试 */
}
```

### 10.3 Reply + Recv 组合

```c
/* 服务端：回复当前消息并等待下一条 */
int ret = airy_sys_reply_recv(reply_cap, &reply_msg, &next_msg);
if (ret < 0) {
    break;  /* 错误退出循环 */
}
/* 处理 next_msg ... */
```

### 10.4 记忆卷载控制

```c
/* 快照 Agent 进程记忆 */
int ret = airy_sys_rovol_ctl(
    0,              /* op: snapshot */
    agent_pid,      /* 目标进程 */
    snapshot_id     /* 快照 ID */
);

/* 恢复 Agent 进程记忆 */
ret = airy_sys_rovol_ctl(
    1,              /* op: restore */
    agent_pid,
    snapshot_id
);
```

### 10.5 CoreLoopThree 阶段通知

```c
/* 通知内核进入思考阶段 */
airy_sys_clt_notify(task_id, AIRY_COG_PHASE_THINKING);

/* 思考完成后通知进入行动阶段 */
airy_sys_clt_notify(task_id, AIRY_COG_PHASE_ACTION);
```

---

## 十一、IRON-9 v2 三层共享模型

### 11.1 三层模型概览

| 层次 | 共享程度 | 本接口涉及内容 |
|------|---------|---------------|
| **[SC] 共享契约层** | 完全共享代码 | `syscalls.h`（24 槽位编号）+ `ipc.h`（IPC magic + 128B 消息头）+ `sched.h`（任务描述符 magic + SCHED_EXT=7）+ `security_types.h`（capability 41 ID + cap_op）+ `memory_types.h`（MemoryRovol L1-L4）+ `cognition_types.h`（三阶段枚举） |
| **[SS] 语义同源层** | API 签名同源，实现独立 | agentrt 用户态 `syscalls.h`（libc syscall 包装）↔ agentrt-linux 内核 `airy_syscalls.h`（SYSCALL_DEFINE*）12 核心同源 |
| **[IND] 完全独立层** | 完全独立 | agentrt 跨平台 syscall 封装（Linux/macOS/Windows）↔ agentrt-linux 内核 syscall 表注册 |

### 11.2 跨态编号一致性

syscall 编号 0-11 段在 agentrt 用户态与 agentrt-linux 内核态**必须**保持二进制一致（AOS-STD-SYS-031）。同一编号在两侧语义完全相同，**禁止**在用户态引入编号重映射（AOS-STD-SYS-032）。

---

## 十二、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-07-11 | 初始草案：12 核心 syscall（8 IPC + 3 控制 + 1 通知）、capability invocation 统一入口、24 槽位编号、AIRY_E* 错误码、双平面架构 |

---

## 十三、参考文献

- [Airymax 开放标准体系总览](./README.md)
- [Airymax IPC 协议标准](./02-airy-ipc-protocol-standard.md)
- [Airymax 调度标准](./06-airy-scheduling-standard.md)
- [Airymax 安全模型标准](./03-airy-security-model-standard.md)
- SSoT syscall 编号：`docs/AirymaxOS/50-engineering-standards/120-cross-project-code-sharing.md` §2.8
- seL4 syscall 模型：https://docs.sel4.systems/projects/sel4/manual/
- Linux io_uring：https://docs.kernel.org/filesystems/io_uring.html
- Linux sched_ext：https://docs.kernel.org/scheduler/sched-ext.html

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
