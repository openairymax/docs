Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# IPC Fastpath 状态机设计
> **文档定位**：agentrt-linux（AirymaxOS） IPC Fastpath 状态机设计——fastpath/slowpath 双路径状态机、io_uring 与 kfifo kthread 集成、性能优化与形式化验证计划\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[agentrt-linux 设计文档](README.md)

---

> **SSoT 依赖声明**： 本文档状态机行为以 `50-engineering-standards/09-ssot-registry.md` 登记的接口规约为权威依据，重点依赖下列条目：OS-IFACE-003 / OS-IFACE-004（128B 消息头 + magic 0x41524531 'ARE1' 经 [SC] `ipc.h` 共享）、OS-ARCH-005 / OS-ARCH-006（seL4 思想分布落地——capability/IPC 契约经 [SC] 共享，io_uring 仅用于用户态-内核态通信、kthread 间改用 kfifo + wait_event_interruptible）、OS-KER-139（数据结构 cache line 对齐）、OS-SEC-121 / OS-SEC-122（capability 检查与令牌管理）。128B 消息头结构体 `struct airy_ipc_msg_hdr` 的物理宿主为 `include/airymax/ipc.h`（见 `50-engineering-standards/120-cross-project-code-sharing.md` §Layout C）。
>
> **理论根基**： seL4 fastpath 设计思想（设计借鉴来源编号 ES-SEL4-4）——seL4 在 `decodeIPC`/`sendIPC` 路径中通过极简状态机实现最小延迟 IPC，本设计将其"无调度、无阻塞、内联检查"思想迁移至 agentrt-linux 的 io_uring + kfifo 双传输通道之上。

---

## 1. 设计目标

agentrt-linux IPC Fastpath 状态机是 128B 消息头（见 [02-ipc-protocol.md](02-ipc-protocol.md) §2）传输路径的核心控制逻辑。状态机决定每条 IPC 请求走 fastpath（无阻塞直通）还是 slowpath（排队/阻塞回退），目标是在 seL4 fastpath 思想（ES-SEL4-4）与 Linux 6.6 io_uring/kfifo 基础设施之间取得最优平衡。

### 1.1 目标量化

| # | 目标 | 指标 | 验收方式 |
|---|------|------|---------|
| G-1 | 最小化 IPC 延迟 | fastpath 单次端到端延迟 < 1μs（x86_64，2.4 GHz 基频，热缓存） | §9.4 延迟直方图 + §10.4 有界延迟证明 |
| G-2 | 优先 fastpath | fastpath 无阻塞、无等待、无调度——禁止在 fastpath 内调用 `schedule()`/`mutex_lock()`/`wait_event*()` | §8 故障注入 + KCSAN 静态扫描 |
| G-3 | slowpath 回退 | 资源不足（kfifo 满、无匹配消息、capability 缺失）时降级至 slowpath，不丢弃消息 | §4 slowpath 回退机制 + §8 注入测试 |
| G-4 | 可测性 | fastpath/slowpath 命中比例、错误率可统计，输出至 debugfs + tracepoint | §9 可观测性 |

### 1.2 设计约束

- **传输通道双栈**：用户态 ↔ 内核态走 io_uring（`IORING_OP_IPC_SEND` / `IORING_OP_IPC_RECV`，见 [02-ipc-protocol.md](02-ipc-protocol.md) §4.4）；内核 kthread 间走 per-cpu kfifo + `wait_event_interruptible`（OS-ARCH-006）。状态机对两条通道统一建模。
- **128B 消息头 = 2 cache lines**：fastpath 仅处理消息头（无 payload）路径，payload 路径强制走 slowpath。
- **per-cpu 无锁**：fastpath 必须在 per-cpu 上下文执行，禁止跨 CPU 同步。
- **ABI 稳定**：状态机内部状态编号不进入 UAPI；对外契约仅 4 操作码（SEND/RECV/SEND_BATCH/CANCEL）与返回码。

### 1.3 操作码定义（4 opcode）

操作码经 [SC] `ipc.h` 共享（OS-IFACE-003），与消息头 `opcode` 字段（offset 4，2 字节）对应：

| opcode 值 | 宏名 | 语义 | fastpath 适用 |
|----------|------|------|--------------|
| 0 | `AIRY_IPC_OP_SEND` | 单条消息发送 | ✅（仅消息头） |
| 1 | `AIRY_IPC_OP_RECV` | 单条消息接收 | ✅（有匹配消息） |
| 2 | `AIRY_IPC_OP_SEND_BATCH` | 批量发送（≥2 条） | ⚠️（走 BATCH 状态，聚合后 fastpath） |
| 3 | `AIRY_IPC_OP_CANCEL` | 取消已提交请求 | ✅（per-cpu 内联取消） |

---

## 2. Fastpath 状态机

### 2.1 状态定义

状态机共 8 状态。状态编号仅用于本文档与 trace 输出，**不进入 UAPI**（内部实现细节，可随版本演进）。

| 状态 | 编号 | 语义 | 是否阻塞 | 是否持锁 | 退出条件 |
|------|------|------|---------|---------|---------|
| `IDLE` | 0 | 空闲，等待 IPC 请求 | 否 | 否 | 收到任意 opcode 事件 |
| `FAST_SEND` | 1 | fastpath 发送——无阻塞直接拷贝 128B 消息头至对端 kfifo / io_uring CQE | 否 | 否（per-cpu） | 拷贝完成 → IDLE；条件不满足 → SLOW_SEND |
| `FAST_RECV` | 2 | fastpath 接收——有匹配的待接收消息且接收缓冲区已就绪，直接拷贝 | 否 | 否（per-cpu） | 拷贝完成 → IDLE；无匹配 → SLOW_RECV |
| `SLOW_SEND` | 3 | slowpath 发送——kfifo 满 / payload 非空 / 跨 CPU，需排队或阻塞 | 是 | 是（等待队列锁） | 唤醒并完成 → IDLE；超时/取消 → CANCEL/ERROR |
| `SLOW_RECV` | 4 | slowpath 接收——无匹配消息，需等待发送方投递 | 是 | 是（等待队列锁） | 收到消息并完成 → IDLE；超时/取消 → CANCEL/ERROR |
| `BATCH` | 5 | 批量发送模式——聚合多条消息后批量提交 io_uring，降低 syscall 摊销 | 否（聚合阶段） | 否 | 聚合满 / flush → FAST_SEND；取消 → CANCEL |
| `CANCEL` | 6 | 取消操作——回收已提交资源，从等待队列移除 | 否 | 短暂持队列锁 | 清理完成 → IDLE；清理失败 → ERROR |
| `ERROR` | 7 | 错误状态——capability 失败 / magic 校验失败 / 资源耗尽不可恢复 | 否 | 否 | 上报错误码后 → IDLE |

### 2.2 状态转换图

下图展示 8 状态间的完整转换关系。`[cond]` 为触发条件，`/act` 为执行动作；`★` 标注 fastpath 直通边（无调度、无阻塞）。

```
                            ┌──────────────────────────────────────────────┐
                            │                                              │
                            ▼                                              │
                       ┌────────┐                                          │
              ┌───────►│  IDLE  │◄─────────────────────────────────────────┤
              │        └───┬────┘                                          │
              │            │                                               │
              │   ┌────────┼────────┬───────────────┬─────────────┐       │
              │   │ SEND   │ RECV   │ SEND_BATCH    │ CANCEL      │       │
              │   │ ≤128B  │ match  │ N≥2           │             │       │
              │   │ cap ok │ buf ok │               │             │       │
              │   │ kfifo↓ │        │               │             │       │
              │   ▼        ▼        ▼               ▼             │       │
              │ ┌─────────────┐ ┌─────────────┐ ┌────────┐  ┌─────────┐    │
              │ │ FAST_SEND ★│ │ FAST_RECV ★ │ │ BATCH  │  │ CANCEL  │    │
              │ └─────┬───────┘ └─────┬───────┘ └───┬────┘  └────┬────┘    │
              │       │ done          │ done        │ flush       │ done    │
              │       │               │             │ N≥thresh    │         │
              │       └──►IDLE        └──►IDLE      ▼             └──►IDLE  │
              │       │ cond fail         no match  FAST_SEND               │
              │       ▼                   ▼                                 │
              │ ┌─────────────┐   ┌─────────────┐                           │
              │ │  SLOW_SEND  │   │  SLOW_RECV  │                           │
              │ │ (queue+blk) │   │ (wait+blk)  │                           │
              │ └─────┬───────┘   └─────┬───────┘                           │
              │       │ wake             │ wake                              │
              │       │ done ►IDLE       │ done ►IDLE                        │
              │       │ timeout          │ timeout                           │
              │       │ cancel           │ cancel                            │
              │       └──►CANCEL         └──►CANCEL                          │
              │                                                            │
              │   magic fail / cap fail / ENOMEM unrecoverable              │
              └──────────────────────────────────────────────────────►┌───────┐
                                                                        │ ERROR │
                                                                        └───┬───┘
                                                                            │ report
                                                                            └──►IDLE
```

**关键边语义说明**：

| 转换边 | 触发条件 | 动作 | fastpath? |
|--------|---------|------|-----------|
| IDLE → FAST_SEND | opcode=SEND 且 §3 全部满足 | 内联拷贝 128B 至对端 | ★ 是 |
| IDLE → FAST_RECV | opcode=RECV 且 §3 全部满足 | 内联拷贝 128B 至用户缓冲 | ★ 是 |
| IDLE → BATCH | opcode=SEND_BATCH | 入聚合缓冲，待 flush | 是（聚合阶段无锁） |
| IDLE → CANCEL | opcode=CANCEL | 从对端队列移除未完成项 | 是 |
| IDLE → SLOW_SEND | opcode=SEND 但 §3 不满足 | 入等待队列，`wait_event_interruptible` | 否 |
| IDLE → SLOW_RECV | opcode=RECV 但 §3 不满足 | 入等待队列，`wait_event_interruptible` | 否 |
| FAST_SEND → IDLE | 拷贝完成 | 唤醒对端 RECV 等待者 | ★ 是 |
| FAST_RECV → IDLE | 拷贝完成 | 释放接收槽 | ★ 是 |
| BATCH → FAST_SEND | 聚合数达阈值或 flush | 批量提交 io_uring SQE | 是 |
| SLOW_SEND → IDLE | 唤醒并完成 | 通知 CQE | 否 |
| SLOW_RECV → IDLE | 收到匹配消息 | 通知 CQE | 否 |
| SLOW_* → CANCEL | 超时或主动取消 | 移出等待队列 | 短暂持锁 |
| CANCEL → IDLE | 清理完成 | 返回 `AIRY_ECANCELED` | 是 |
| 任意 → ERROR | magic / capability / 不可恢复资源失败 | 返回错误码，统计计数 | 否 |

### 2.3 状态转换表

下表覆盖 8 状态 × 4 操作码 × 主要边界条件。`—` 表示该事件在当前状态不合法（保持原状态并返回 `AIRY_EINVAL`），`→` 表示状态转换。`FP` = fastpath 直通，`SP` = slowpath 回退。

#### 2.3.1 当前状态 × 事件 → 新状态 + 动作

| 当前状态 | 事件 | 新状态 | 动作 | 备注 |
|---------|------|--------|------|------|
| `IDLE` | SEND 且 §3 全满足 | `FAST_SEND` | 内联拷贝 128B → 对端 kfifo/CQE；统计 FP | ★ FP |
| `IDLE` | SEND 且 §3 不满足 | `SLOW_SEND` | 入 wait queue；`wait_event_interruptible` | SP |
| `IDLE` | RECV 且有匹配消息 + buf 就绪 | `FAST_RECV` | 内联拷贝 128B → 用户 buf；统计 FP | ★ FP |
| `IDLE` | RECV 且无匹配消息 | `SLOW_RECV` | 入 wait queue；`wait_event_interruptible` | SP |
| `IDLE` | SEND_BATCH（N≥2） | `BATCH` | 入聚合缓冲；待阈值/flush | 聚合阶段无锁 |
| `IDLE` | CANCEL（有可取消项） | `CANCEL` | 从对端队列移除；释放资源 | 内联 |
| `IDLE` | CANCEL（无可取消项） | `IDLE` | 返回 `AIRY_ENOENT` | 空操作 |
| `IDLE` | magic != 0x41524531 | `ERROR` | 返回 `AIRY_EPROTO`；统计 error | — |
| `FAST_SEND` | 拷贝完成 | `IDLE` | 唤醒对端 RECV 等待者；填 CQE | ★ FP 出口 |
| `FAST_SEND` | 拷贝过程中 kfifo 满（竞态） | `SLOW_SEND` | 回滚已拷贝部分；入 wait queue | SP 降级 |
| `FAST_SEND` | CANCEL 请求 | `CANCEL` | 中止拷贝；回滚 | — |
| `FAST_SEND` | RECV / SEND_BATCH 事件 | `—` | 排队至下一轮 IDLE | 单任务单流程 |
| `FAST_RECV` | 拷贝完成 | `IDLE` | 释放接收槽；填 CQE | ★ FP 出口 |
| `FAST_RECV` | 接收 buf 失效（被 munmap） | `ERROR` | 返回 `AIRY_EFAULT` | — |
| `FAST_RECV` | CANCEL 请求 | `CANCEL` | 中止拷贝 | — |
| `SLOW_SEND` | 唤醒（kfifo 有空位） | `IDLE` | 完成拷贝；填 CQE | SP 出口 |
| `SLOW_SEND` | 超时（> `timeout_ns`） | `CANCEL` | 移出 wait queue；返回 `AIRY_ETIMEDOUT` | — |
| `SLOW_SEND` | CANCEL 请求 | `CANCEL` | 移出 wait queue；返回 `AIRY_ECANCELED` | — |
| `SLOW_SEND` | 信号中断（EINTR） | `CANCEL` | 移出 wait queue；返回 `AIRY_EINTR` | — |
| `SLOW_SEND` | SEND / RECV / SEND_BATCH | `—` | 排队，不抢占当前流程 | — |
| `SLOW_RECV` | 收到匹配消息（唤醒） | `IDLE` | 完成拷贝；填 CQE | SP 出口 |
| `SLOW_RECV` | 超时 | `CANCEL` | 返回 `AIRY_ETIMEDOUT` | — |
| `SLOW_RECV` | CANCEL 请求 | `CANCEL` | 返回 `AIRY_ECANCELED` | — |
| `SLOW_RECV` | 信号中断 | `CANCEL` | 返回 `AIRY_EINTR` | — |
| `BATCH` | 聚合数达阈值 / flush | `FAST_SEND` | 批量提交 SQE；统计 FP | ★ FP 出口 |
| `BATCH` | CANCEL 请求 | `CANCEL` | 丢弃未提交聚合项；返回 `AIRY_ECANCELED` | — |
| `BATCH` | 单条聚合项 magic 失败 | `ERROR` | 丢弃该项；继续聚合其余；统计 error | 部分失败 |
| `BATCH` | SEND / RECV 事件 | `—` | 排队 | — |
| `CANCEL` | 清理完成 | `IDLE` | 返回 `AIRY_ECANCELED` / `AIRY_ETIMEDOUT` | — |
| `CANCEL` | 清理失败（资源已被对端取走） | `IDLE` | 返回 `AIRY_EALREADY` | 视为已完成 |
| `CANCEL` | SEND / RECV / SEND_BATCH / CANCEL | `—` | 排队 | — |
| `ERROR` | 错误码上报完成 | `IDLE` | 重置流程上下文 | — |
| `ERROR` | 任意新事件 | `IDLE` | 先回到 IDLE 再处理 | 强制复位 |

#### 2.3.2 边界条件补充

| 边界条件 | 处理 |
|---------|------|
| fastpath 中途被抢占（抢占式内核 CONFIG_PREEMPT） | 拷贝为原子的 `copy_to_user`/`kfifo_in`，恢复后继续，不降级 |
| 同一 CPU 上 fastpath 与 slowpath 并发 | per-cpu 上下文，单 CPU 串行，无并发 |
| 跨 CPU fastpath（src 与 dst 不同 CPU） | 强制降级 SLOW_SEND（跨 CPU 需 IPI 唤醒，不满足无锁条件） |
| payload_len > 0（有 payload） | 强制降级 SLOW_SEND（fastpath 仅处理 128B 消息头） |
| kfifo 水位 > 高水位（`AIRY_IPC_KFIFO_HI_WMARK`） | 降级 SLOW_SEND，触发背压（见 §6.4） |
| capability 令牌过期 / 权限不足 | 进入 ERROR，返回 `AIRY_ECAPABILITY`（OS-SEC-121） |
| `flags` 含 `AIRY_IPC_F_NOWAIT` 且需阻塞 | 不进入 SLOW_*，直接返回 `AIRY_EAGAIN` |

---

## 3. Fastpath 触发条件

fastpath 是状态机的"热路径"，必须满足下列全部条件方可进入 `FAST_SEND` / `FAST_RECV`。任一条件不满足即触发 §4 slowpath 回退。条件检查全部内联（`__always_inline`），且按"廉价检查在前、昂贵检查在后"排序以缩短平均路径长度。

### 3.1 发送方条件（进入 FAST_SEND）

| # | 条件 | 检查方式 | 失败后果 |
|---|------|---------|---------|
| C-S1 | `magic == 0x41524531`（'ARE1'） | 内联比较 | → ERROR（`AIRY_EPROTO`） |
| C-S2 | `opcode == AIRY_IPC_OP_SEND`（=0） | 内联比较 | 走对应 opcode 分支 |
| C-S3 | `payload_len == 0`（仅消息头，无 payload） | 内联比较 | → SLOW_SEND |
| C-S4 | 消息总大小 ≤ 128B（= `AIRY_IPC_HDR_SZ`） | `sizeof(hdr) <= 128` | → SLOW_SEND |
| C-S5 | src 与 dst 在同一 CPU（per-cpu fastpath 无锁） | `smp_processor_id() == dst_cpu` | → SLOW_SEND |
| C-S6 | per-cpu kfifo 未满（`kfifo_is_full()` == false） | 内联 | → SLOW_SEND（背压） |
| C-S7 | 无内存压力（`current->reclaim_flag` 未置 / `__GFP_MEMALLOC` 未触发） | 内联 | → SLOW_SEND |
| C-S8 | 无需锁（per-cpu 上下文，`in_task()` && `!in_interrupt()`） | 内联 | → SLOW_SEND |
| C-S9 | capability 检查通过（fastpath 内联 `airy_cap_check_fast()`） | 内联（OS-SEC-121） | → ERROR（`AIRY_ECAPABILITY`） |
| C-S10 | `flags` 不含 `AIRY_IPC_F_ENCRYPT` / `COMPRESS`（fastpath 不处理加解密） | 位测试 | → SLOW_SEND |

### 3.2 接收方条件（进入 FAST_RECV）

| # | 条件 | 检查方式 | 失败后果 |
|---|------|---------|---------|
| C-R1 | `magic == 0x41524531` | 内联比较 | → ERROR |
| C-R2 | `opcode == AIRY_IPC_OP_RECV`（=1） | 内联比较 | 走对应分支 |
| C-R3 | 有匹配的待接收消息（kfifo 非空且 `dst_task` 匹配） | `kfifo_out_peek()` | → SLOW_RECV |
| C-R4 | 接收缓冲区已就绪（用户态已 `IORING_REGISTER_BUFFERS`） | 内联查表 | → SLOW_RECV |
| C-R5 | src 与 dst 在同一 CPU | `smp_processor_id()` | → SLOW_RECV |
| C-R6 | 接收消息 `payload_len == 0`（仅消息头） | 内联比较 | → SLOW_RECV |
| C-R7 | capability 检查通过（接收权限） | 内联 | → ERROR |
| C-R8 | 无需等待（`flags` 不要求有序等待） | 位测试 | → SLOW_RECV |

### 3.3 资源充足条件（C-S6 / C-R3 共同前提）

| 资源 | 阈值 | 说明 |
|------|------|------|
| per-cpu kfifo 容量 | 默认 4096 项 × 128B = 512 KiB/CPU | `AIRY_IPC_KFIFO_SIZE`，可 `Kconfig` 调 |
| kfifo 低水位 | 25%（1024 项） | 低于此值解除背压 |
| kfifo 高水位 | 75%（3072 项） | 超过触发背压，新 SEND 降级 |
| registered buffers 余量 | ≥1 | io_uring 固定缓冲区池 |
| 系统内存水位 | 高于 `WMARK_HIGH` | 否则降级（避免 fastpath 挤压回收） |

---

## 4. Slowpath 回退机制

### 4.1 触发条件

slowpath 在 §3 任一条件不满足时触发，具体降级路径如下：

| 失败条件 | 降级目标 | 原因 |
|---------|---------|------|
| `payload_len > 0` | SLOW_SEND | payload 需 IORING_OP_URING_CMD，不可内联 |
| 跨 CPU（src ≠ dst CPU） | SLOW_SEND | 需 IPI 唤醒对端，破坏无锁假设 |
| kfifo 满 / 高水位 | SLOW_SEND | 需排队等待空位 |
| 接收无匹配消息 | SLOW_RECV | 需等待发送方投递 |
| `flags` 含 ENCRYPT/COMPRESS | SLOW_SEND | 需调用加密/压缩子系 |
| capability 检查需异步询问 daemon | SLOW_SEND | OS-SEC-009：ASK 裁决需经 AgentsIPC 询问 daemon |

### 4.2 回退动作

1. **入等待队列**：将 `airy_ipc_wait_node` 挂入 per-channel 等待队列（`wait_queue_head_t`），节点携带消息头副本、`trace_id`、超时值。
2. **设置可中断等待状态**：`set_current_state(TASK_INTERRUPTIBLE)`，响应信号（EINTR）。
3. **调度阻塞**：调用 `schedule()`，让出 CPU。fastpath 严禁进入此步。
4. **唤醒后重试**：被唤醒后重新评估 §3 条件——若此时满足则可"升级"回 fastpath 完成（fastpath-on-wake 优化），否则继续 slowpath。

### 4.3 唤醒机制

- **kthread 间**：发送方完成 kfifo 投递后调用 `wake_up_interruptible(&recv_wait)`；接收方 `wait_event_interruptible(timeout, kfifo_has_match())` 返回后处理。
- **用户态-内核态**：io_uring 完成后填 CQE，用户态 poll 线程或 `io_uring_enter()` 返回时感知。
- **批量唤醒**：`wake_up_interruptible_all` 用于广播消息（`dst_task == 0`）。

### 4.4 超时处理

| 字段 | 来源 | 默认 |
|------|------|------|
| `timeout_ns` | 消息头无此字段，由 `flags` 高位 + `reserved` 协商或调用参数传入 | 5 s（与 OS-SEC-009 ASK 超时对齐） |
| 超时动作 | 移出 wait queue，返回 `AIRY_ETIMEDOUT`，状态 → CANCEL | — |
| `flags` 含 NOWAIT | 不进入 slowpath，直接返回 `AIRY_EAGAIN` | — |

---

## 5. io_uring 集成

io_uring 承担用户态 ↔ 内核态的 IPC 数据面（OS-ARCH-006）。状态机在 `FAST_SEND` / `FAST_RECV` / `BATCH` 状态与 io_uring SQE/CQE 交互。

### 5.1 SQE/CQE 格式适配 IPC 128B 消息头

| io_uring 字段 | IPC 用途 |
|--------------|---------|
| `sqe.opcode` | `IORING_OP_IPC_SEND`(0x40) / `IPC_RECV`(0x41) / `IPC_SEND_BATCH`(0x42) / `IPC_CANCEL`(0x43) |
| `sqe.fd` | 目标 channel fd（`airy_ipc_channel_open()` 返回） |
| `sqe.addr` | 128B 消息头用户态地址（或 registered buffer id） |
| `sqe.len` | 固定 128（fastpath），>128（slowpath 含 payload） |
| `sqe.user_data` | 携带 `trace_id`，原样回填至 `cqe.user_data` |
| `cqe.res` | 成功=128（fastpath），失败=`-E*` / `AIRY_E*` |
| `cqe.flags` | 携带 fastpath/slowpath 命中标记（`AIRY_CQE_F_FASTPATH`） |

> **注意**：`opcode` 字段在消息头（offset 4，`__u16`）与 io_uring SQE（`__u8`）中均存在，但分属不同层次——消息头 `opcode` 是 IPC 协议操作码（SEND/RECV/SEND_BATCH/CANCEL），io_uring `sqe.opcode` 是 io_uring 私有操作码（`IORING_OP_IPC_*`）。二者通过固定映射表转换，**禁止混用**。

### 5.2 注册固定缓冲区（registered buffers）

- 发送方启动时 `io_uring_register(ring_fd, IORING_REGISTER_BUFFERS, iov, n)` 预注册 128B 缓冲池，内核 pin 住 page，避免每条消息 `get_user_pages()`。
- 接收方同理注册接收缓冲池。
- fastpath 中 `sqe.flags |= IOSQE_FIXED_FILE | IOSQE_BUFFER_SELECT`，从固定池选取 buffer id，零拷贝 IORING_OP_URING_CMD + registered buffer（OS-IFACE-003 零拷贝优先）。

### 5.3 polled I/O 模式

- ring 创建时设 `IORING_SETUP_IOPOLL` + `IORING_SETUP_SQPOLL`，内核轮询 SQE，消除 `io_uring_enter()` 系统调用开销。
- fastpath 纯用户态提交 → 内核 sqpoll 线程轮询 → 内联拷贝 → CQE 回填，全程无 syscall。
- sqpoll 纺程 idle 超时后退出，由新 SQE 唤醒（首次有 ≤1μs 唤醒开销，热路径稳定后归零）。

### 5.4 批量提交（SEND_BATCH）的 io_uring 优化

- `BATCH` 状态聚合 N 条 128B 消息至连续缓冲（N × 128B），单次 SQE 提交（`sqe.len = N * 128`）。
- 阈值 `AIRY_IPC_BATCH_THRESHOLD`（默认 8）：达到即 flush 走 `FAST_SEND`；或调用方显式 flush。
- 批量提交摊销 SQE 入队与 CQE 回填开销，吞吐较单条提升 4–6×（实测目标）。
- CQE 返回批量结果掩码，逐条状态由 `cqe.res` 高 16 位（成功数）+ 低 16 位（失败数）编码。

---

## 6. kfifo kthread 间通信

内核 kthread 间（如 CoreLoopThree kthread 之间，见 [01-syscalls.md](01-syscalls.md) §1.1 认知原语）走 per-cpu kfifo + `wait_event_interruptible`，不走 io_uring（OS-ARCH-006）。

### 6.1 per-cpu kfifo 设计

```c
struct airy_ipc_kfifo {
    DECLARE_KFIFO_PTR(fifo, struct airy_ipc_msg_hdr) ____cacheline_aligned_in_smp;
    wait_queue_head_t          recv_wq;
    unsigned long              water_mark;   /* 当前水位 */
    struct airy_ipc_stats __percpu *stats;   /* per-cpu 统计 */
} __attribute__((aligned(SMP_CACHE_BYTES)));

static DEFINE_PER_CPU(struct airy_ipc_kfifo, airy_ipc_kfifo);
```

- 每个 CPU 独立 kfifo，容量 `AIRY_IPC_KFIFO_SIZE`（默认 4096 项 × 128B = 512 KiB）。
- `____cacheline_aligned_in_smp` 保证 SMP 下 kfifo 结构体不跨 cache line 共享。
- 同 CPU 内发送 kthread → 接收 kthread 为单生产者单消费者（SPSC）。

### 6.2 无锁单生产者单消费者模型

- kfifo 本身支持 SPSC 无锁：发送方 `kfifo_in()`，接收方 `kfifo_out()`，无需自旋锁。
- 前提：同一 kfifo 仅有一个生产者 kthread + 一个消费者 kthread；跨 kthread 通信需选择"同 CPU 绑定"的 kthread pair，否则降级 slowpath（跨 CPU 走 io_uring 或加锁 kfifo）。
- 状态机 `FAST_SEND` 仅在 `smp_processor_id() == dst_cpu` 时启用（C-S5）。

### 6.3 内存屏障使用

| 屏障 | 位置 | 作用 |
|------|------|------|
| `smp_wmb()` | `kfifo_in()` 之后、`wake_up_interruptible()` 之前 | 保证接收方唤醒时能看到 kfifo 中已写入的数据 |
| `smp_rmb()` | `kfifo_out_peek()` 之前、读 `water_mark` 之后 | 保证读到水位后再读 kfifo 数据，避免读到旧数据 |
| `smp_mb()` | 等待队列 `wait_event_interruptible` 内部已含 | 标准内核原语保证 |

```c
/* 发送方 fastpath */
if (likely(kfifo_in(&kfifo->fifo, &hdr, 1) == 1)) {
    smp_wmb();                          /* 发布写入 */
    wake_up_interruptible(&kfifo->recv_wq);
    this_cpu_inc(stats->fast_send_hit);
    return 128;                         /* → IDLE */
}

/* 接收方 fastpath */
if (likely(kfifo_out_peek(&kfifo->fifo, &hdr, 1) == 1)) {
    smp_rmb();                          /* 读数据前屏障 */
    if (likely(airy_cap_check_fast(hdr.dst_task, cap)))
        kfifo_out(&kfifo->fifo, &hdr, 1);
}
```

### 6.4 水位线与背压机制

| 水位 | 阈值 | 行为 |
|------|------|------|
| 低水位 `LO_WMARK` | 25%（1024 项） | 解除背压：恢复接收 SEND，关闭 `AIRY_IPC_F_BACKPRESSURE` |
| 中水位 `MID_WMARK` | 50%（2048 项） | 正常 |
| 高水位 `HI_WMARK` | 75%（3072 项） | 触发背压：新 SEND 降级 SLOW_SEND，发送方入 wait queue |
| 满水位 `FULL` | 100%（4096 项） | `kfifo_is_full()` 为真，C-S6 失败，强制 SLOW_SEND |

背压通知：达 `HI_WMARK` 时通过 CQE `flags` 置 `AIRY_CQE_F_BACKPRESSURE`，提示用户态降速；达 `FULL` 时返回 `AIRY_EAGAIN`（NOWAIT）或阻塞（默认）。

---

## 7. 性能优化

### 7.1 cache line 对齐

- 128B 消息头 = 2 cache lines（x86_64 64B/line），单次 `prefetch` 即可加载完整头。
- `struct airy_ipc_msg_hdr` 已 `__attribute__((packed))`（见 [02-ipc-protocol.md](02-ipc-protocol.md) §2.3），无填充间隙。
- per-cpu kfifo 结构体 `____cacheline_aligned_in_smp`，避免 false sharing（OS-KER-139）。
- 热数据（`stats`、`water_mark`）单独 cache line 对齐。

### 7.2 per-cpu 热数据分离

- `DEFINE_PER_CPU` 隔离每 CPU 热路径数据，避免跨 CPU 缓存失效。
- `__read_mostly` 标注只读配置（`AIRY_IPC_KFIFO_SIZE`、`AIRY_IPC_BATCH_THRESHOLD`）。
- 统计计数器用 `this_cpu_inc()` / `this_cpu_add()`，避免原子操作开销。

### 7.3 分支预测优化

- fastpath 条件用 `likely()` 包裹（§3 条件大多数满足时）。
- slowpath 回退用 `unlikely()` 包裹。
- 错误检查（magic / capability 失败）用 `unlikely()`。

```c
if (likely(hdr.magic == AIRY_IPC_MAGIC) &&
    likely(hdr.opcode == AIRY_IPC_OP_SEND) &&
    likely(hdr.payload_len == 0) &&
    likely(smp_processor_id() == dst_cpu) &&
    likely(!kfifo_is_full(&kfifo->fifo)) &&
    likely(airy_cap_check_fast(hdr.dst_task, cap))) {
    /* FAST_SEND fastpath */
} else {
    /* slowpath fallback */
}
```

### 7.4 内联函数

- fastpath 入口 `airy_ipc_fastpath_send()` / `airy_ipc_fastpath_recv()` 标注 `__always_inline`，消除函数调用开销（call/ret、栈帧）。
- `airy_cap_check_fast()` 内联（仅做 capability 位掩码比较，不查 daemon）。
- slowpath 函数不内联（体积大，避免 icache 污染，对齐 OS-KER-138：>3 行函数不应 inline）。

### 7.5 prefetch 指令使用

- `prefetch(hdr)` 在解码 opcode 前预取消息头第二 cache line（offset 64–127）。
- `prefetchw(&kfifo->fifo)` 预取 kfifo 写缓冲，减少写入 miss。
- 接收方 `prefetch(recv_buf)` 在 capability 检查期间预取接收缓冲。

---

## 8. 错误注入测试

### 8.1 故障注入框架

启用 `CONFIG_AIRY_FAULT_INJECTION`（agentrt-linux 专属 Kconfig，基于 Linux `failslab`/`fail_function` 扩展），提供下列注入点：

### 8.2 注入点

| 注入点 | 触发方式 | 预期状态转换 | 验证项 |
|--------|---------|-------------|--------|
| 内存分配失败 | `fail_function(airy_ipc_alloc)` | FAST_SEND → ERROR / SLOW_SEND | 返回 `AIRY_ENOMEM` |
| kfifo 满 | `fail_kfifo_full`（强制 `kfifo_is_full()` 返回 true） | IDLE → SLOW_SEND | 触发背压，入 wait queue |
| capability 检查失败 | `fail_cap_check`（强制 `airy_cap_check_fast()` 返回 false） | FAST_SEND → ERROR | 返回 `AIRY_ECAPABILITY`（OS-SEC-121） |
| magic 校验失败 | 注入错误 magic `0xDEADBEEF` | IDLE → ERROR | 返回 `AIRY_EPROTO` |
| 唤醒丢失 | `fail_wake_up`（丢弃 `wake_up_interruptible`） | SLOW_RECV 超时 → CANCEL | 返回 `AIRY_ETIMEDOUT` |
| 跨 CPU 注入 | 强制 `dst_cpu != smp_processor_id()` | IDLE → SLOW_SEND | 走跨 CPU 路径 |

### 8.3 统计与验证

| 指标 | 采集方式 | 验收 |
|------|---------|------|
| fastpath 命中率 | `stats->fast_send_hit / total_send` | 正常负载 > 90% |
| slowpath 触发率 | `stats->slow_send_hit / total_send` | 正常负载 < 10% |
| 错误率 | `stats->error_hit / total` | < 0.01%（排除注入） |
| fastpath 延迟分位 | §9.4 直方图 | p99 < 1μs |

### 8.4 CI 集成

- 每次 PR 在 `tests-linux` 子仓运行错误注入测试套件 `ipc_fastpath_fault_inject`。
- 矩阵：8 状态 × 4 opcode × 6 注入点 = 192 用例，全量需通过。
- 覆盖率门槛：状态机分支覆盖 ≥ 95%（含 slowpath 与 ERROR 边）。

---

## 9. 可观测性

### 9.1 tracepoint 定义

```c
DECLARE_TRACE(airy_ipc_fastpath_enter,
    TP_PROTO(__u32 state, __u16 opcode, __u64 trace_id),
    TP_ARGS(state, opcode, trace_id));

DECLARE_TRACE(airy_ipc_fastpath_exit,
    TP_PROTO(__u32 state, int res, __u64 trace_id, u64 latency_ns),
    TP_ARGS(state, res, trace_id, latency_ns));
```

- `trace_airy_ipc_fastpath_enter`：状态机进入新状态时触发（含目标 state、opcode、trace_id）。
- `trace_airy_ipc_fastpath_exit`：状态机退出至 IDLE 时触发（含最终 state、返回码、trace_id、本次停留延迟）。
- 通过 `tracefs` 启用，遵循 OS-KER-082：print fmt 须以 `trace_id=0x%016llx` 起始。

### 9.2 统计计数器（per-cpu stats）

```c
struct airy_ipc_stats {
    u64 fast_send_hit;
    u64 fast_recv_hit;
    u64 slow_send_hit;
    u64 slow_recv_hit;
    u64 batch_hit;
    u64 cancel_hit;
    u64 error_hit;
    u64 backpressure_hit;
} __attribute__((aligned(SMP_CACHE_BYTES)));
```

- `this_cpu_inc()` 无锁累加，读时 `for_each_possible_cpu` 求和。
- 周期性（每 1 s）由 watchdog kthread 汇总至全局。

### 9.3 debugfs 接口

```
/sys/kernel/debug/airy/ipc/
├── stats              # 文本统计：fastpath/slowpath/error 命中数与比例
├── kfifo_watermark    # per-cpu kfifo 当前水位
├── latency_hist       # fastpath 延迟直方图（见 §9.4）
├── state_dump         # 当前各 CPU 状态机状态快照
└── fault_inject       # 写入触发故障注入（需 CONFIG_AIRY_FAULT_INJECTION）
```

- 读 `stats` 输出示例：
  ```
  cpu  fast_send   fast_recv   slow_send   slow_recv   batch   cancel   error   backpressure
  0    1024567     1023990     11234       11890        560     12       3       45
  1    998123      997001      10987       11023        540     9        1       38
  fastpath ratio: 98.91%
  ```

### 9.4 延迟直方图

- 分桶：`<100ns` / `100–500ns` / `500ns–1μs` / `1–10μs`（slowpath）/ `>10μs`（异常）。
- 采集点：`trace_airy_ipc_fastpath_exit` 的 `latency_ns`。
- 输出至 `latency_hist`，CI 解析验证 p99 落在 `<1μs` 桶。

---

## 10. 形式化验证计划

参考 seL4 形式化验证方法论（ES-SEL4-4），对 fastpath 状态机进行完备性、无死锁、有界延迟证明。验证工作落入 [IND] 层，由 `tests-linux` 子仓独立维护（OS-ARCH-005：形式化验证经 [IND] 各自独立实现）。

### 10.1 验证工具链

- **Isabelle/HOL**：定理证明主框架（与 seL4 l4v 一致）。
- **AutoCorres**：从 C 代码自动提取 Isabelle 模型（针对 fastpath C 实现）。
- **输入**：`airy_ipc_fastpath_send()` / `airy_ipc_fastpath_recv()` C 源码 + 本文状态机规约。
- **输出**：Isabelle 证明脚本 + 证书（`fastpath_thys.thy`）。

### 10.2 fastpath 状态机完备性证明

- **完备性定理**：对任意 (state, opcode, 边界条件) 组合，状态转换表 §2.3 唯一确定下一状态与动作，无未定义行为。
- **证明策略**：枚举 8 状态 × 4 opcode × N 边界条件，对每个组合应用转换规则，验证可达 IDLE 或 ERROR 终态。
- **不变式**：`magic` 字段在非 ERROR 状态恒为 `0x41524531`；`trace_id` 全链路不变。

### 10.3 无死锁证明

- **死锁条件排查**：fastpath 无锁（per-cpu 无锁），不存在循环等待；slowpath 仅持单一 wait queue 锁，且等待顺序固定（按 `trace_id` 排序入队），不存在锁序倒置（对齐 OS-SEC-120：多锁获取遵循固定锁顺序）。
- **证明策略**：构造等待图（wait-for graph），证明无环；证明 fastpath 路径不持任何可睡眠锁。
- **KCSAN 交叉验证**：运行时数据竞争检测器验证无锁假设成立（OS-SEC-119）。

### 10.4 有界延迟证明（≤ 1μs）

- **延迟建模**：fastpath 延迟 = `decode_opcode` + `cap_check` + `kfifo_in` + `wake_up` + `CQE_fill`。
- **指令计数**：在 2.4 GHz 基频下，上述步骤指令数 ≤ 2400 条（= 1μs 预算），每步计数：
  - `decode_opcode`：≤ 5 条（内联比较）
  - `cap_check_fast`：≤ 20 条（位掩码）
  - `kfifo_in`（SPSC 无锁）：≤ 80 条
  - `wake_up_interruptible`（同 CPU 唤醒，目标已 running）：≤ 100 条
  - `CQE_fill`：≤ 40 条
  - 合计 ≤ 245 条 << 2400 条预算，留有 ~10× 余量
- **证明目标**：`∀ fastpath 执行 e, latency(e) ≤ 1μs`（在热缓存、同 CPU、无抢占前提下）。
- **边界排除**：抢占、缓存 miss、跨 CPU 场景不属 fastpath 保证范围（降级 slowpath）。

### 10.5 验证里程碑

| 阶段 | 交付物 | 状态 |
|------|--------|------|
| V1 | 状态机规约 Isabelle 化 + 完备性证明 | 规划中（1.0.1 开发阶段） |
| V2 | AutoCorres 提取 C 模型 + 无死锁证明 | 规划中 |
| V3 | 有界延迟证明 + KCSAN 交叉验证 | 规划中 |
| V4 | 全量证明证书集成 CI | 规划中 |

---

## 11. 相关文档

- [接口设计](README.md)（父文档）
- [02-ipc-protocol.md](02-ipc-protocol.md)（128B 消息头 + 5 payload + io_uring 零拷贝）
- [01-syscalls.md](01-syscalls.md)（`AIRY_SYS_IPC_*` 系统调用 + capability invocation）
- [50-engineering-standards/09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)（SSoT 依赖：OS-IFACE-003/004、OS-ARCH-005/006、OS-KER-139、OS-SEC-121/122）
- [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) §Layout C（`ipc.h` 物理宿主）
- [10-architecture/03-microkernel-strategy.md](../10-architecture/03-microkernel-strategy.md)（seL4 思想分布落地，ES-SEL4-4 借鉴来源）

---

## 附录：v1.0 升级说明

> **升级背景**：本次 v1.0 升级将本文档对齐 Airymax Unify Design（见 [10-unify-design.md](../10-architecture/10-unify-design.md)）UIPF 模块设计与方案 C-Prime 技术选型，消解 v0.2.8 遗留的零拷贝路径与 capability 检查架构冲突点。升级内容如下。

### A.1 已删除 page flipping，改为 IORING_OP_URING_CMD + registered buffer + mmap

v0.2.8 在 §5 io_uring 集成章节中存在 page flipping 零拷贝路径的残留表述（见 [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §6 C-02 冲突点，6 个文件引用已废弃的 page flipping）。v1.0 彻底删除 page flipping，统一为 UIPF 模块技术选型：

- **零拷贝载体**：`IORING_OP_URING_CMD` + registered buffer，消除 page flipping 的页所有权反转复杂度
- **共享内存**：`alloc_pages(GFP_KERNEL)` + mmap（不使用 DMA 一致性内存），对齐 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §1 内存方案
- §5.2 registered buffers 机制已更新为固定缓冲池 + `IOSQE_BUFFER_SELECT` 零拷贝选取，不再涉及 page flipping

### A.2 已新增 Capability 缓存机制

v0.2.8 §3 fastpath 触发条件中 C-S9 / C-R7 的 capability 检查（`airy_cap_check_fast()`）依赖同步查询 capability daemon，破坏 fastpath 无阻塞假设。v1.0 新增 **Capability 缓存机制**，使 fastpath capability 检查真正内联无阻塞：

- **per-cpu capability 缓存**：fastpath 路径读取 per-cpu 缓存的 capability badge 掩码，无需查询 daemon
- **缓存一致性**：capability 撤销/派生时通过 IPI 失效目标 CPU 的缓存行，对齐 [03-capability-model.md](../110-security/03-capability-model.md) 离线缓存校验与 Reconciliation 设计
- **缓存未命中回退**：缓存失效时 fastpath 降级至 slowpath（§4），由 daemon 异步裁决后回填缓存
- 该机制对齐 Airymax Unify Design 的 [SC] `lsm_types.h` Capability 缓存结构定义

### A.3 已对齐 Airymax Unify Design UIPF 模块

本文档作为 UIPF（统一 IPC Fastpath）模块的数据面状态机设计，v1.0 明确对齐 Airymax Unify Design 的 UIPF 模块边界：

- **UIPF 模块定位**：UIPF 是 Unify Design 5 模块之一（[10-unify-design.md](../10-architecture/10-unify-design.md) §5），负责统一 IPC fastpath 的状态机、零拷贝路径、capability 检查缓存
- **权威源边界**：本文档是 fastpath 状态机（§2 8 状态 + §3 触发条件 + §4 回退机制）的唯一权威源；128B 消息头布局的权威源为 [02-ipc-protocol.md](02-ipc-protocol.md) + [SC] `ipc.h`；capability 模型的权威源为 [03-capability-model.md](../110-security/03-capability-model.md)
- **技术选型统一**：IPC 零拷贝统一为 `IORING_OP_URING_CMD` + registered buffer + mmap（不使用 page flipping），与 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §4.2 [IND] 层 IPC 实现差异表一致

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
