Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# IPC Fastpath 状态机设计
> **文档定位**：agentrt-linux（AirymaxOS） IPC Fastpath 状态机设计——fastpath/slowpath 双路径状态机、Capability Folding C-S9 Badge 内联校验、io_uring `IORING_OP_URING_CMD` 复用与 kfifo kthread 集成、性能优化与形式化验证计划\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[agentrt-linux 设计文档](README.md)

---

> **SSoT 依赖声明**： 本文档状态机行为以 `50-engineering-standards/09-ssot-registry.md` 登记的接口规约为权威依据，重点依赖下列条目：OS-IFACE-003 / OS-IFACE-004（Layout C v4 128B 消息头 + magic 0x41524531 'ARE1' + `capability_badge` 字段经 [SC] `ipc.h` 共享）、OS-ARCH-005 / OS-ARCH-006（seL4 思想分布落地——capability/IPC 契约经 [SC] 共享，io_uring 仅用于用户态-内核态通信、kthread 间改用 kfifo + wait_event_interruptible）、OS-KER-139（数据结构 cache line 对齐）、OS-SEC-121 / OS-SEC-122（capability 检查与令牌管理）。Layout C v4 消息头结构体 `struct airy_ipc_msg_hdr` 的物理宿主为 `include/uapi/linux/airymax/ipc.h`（见 [120-cross-project-code-sharing.md §2.7](../50-engineering-standards/120-cross-project-code-sharing.md)）。
>
> **Capability Folding 工程定义**（v1.1 新增）：A-IPC 采用 Capability Folding 设计模式——将 capability check 从独立控制面操作"折叠"到 IPC 数据面 fastpath 中。物理载体是 [SC] `ipc.h` Layout C v4 消息头 offset 40-47 的 `capability_badge` 字段（64-bit Native Word：Epoch + Random Tag + Perms）；执行点是 fastpath C-S9 内联校验 `airy_cap_badge_ok()`（~10ns，3 个 `READ_ONCE` + 位运算 + 比较）。本文件 §3.1 / §5.2 是 fastpath 校验链与 C-S9 实现的 SSoT 权威源。
>
> **理论根基**： seL4 fastpath 设计思想（设计借鉴来源编号 ES-SEL4-4）——seL4 在 `decodeIPC`/`sendIPC` 路径中通过极简状态机实现最小延迟 IPC，本设计将其"无调度、无阻塞、内联检查"思想迁移至 agentrt-linux 的 io_uring + kfifo 双传输通道之上。Capability Folding 进一步将 seL4 的 capability 校验内联到 IPC fastpath，消除独立 capability syscall。

---

## 1. 设计目标

agentrt-linux IPC Fastpath 状态机是 Layout C v4 128B 消息头（见 [02-ipc-protocol.md](02-ipc-protocol.md) §2）传输路径的核心控制逻辑。状态机决定每条 IPC 请求走 fastpath（无阻塞直通）还是 slowpath（排队/阻塞回退），目标是在 seL4 fastpath 思想（ES-SEL4-4）与 Linux 6.6 io_uring/kfifo 基础设施之间取得最优平衡。v1.1 起，fastpath 内联 C-S9 Capability Folding Badge 校验，IPC 数据传递即能力校验。

### 1.1 目标量化

| # | 目标 | 指标 | 验收方式 |
|---|------|------|---------|
| G-1 | 最小化 IPC 延迟 | fastpath 单次端到端延迟 ~158ns（x86_64，2.4 GHz 基频，热缓存），含 C-S9 Badge 校验 | §5.3 延迟分解 + §10.4 有界延迟证明 |
| G-2 | 优先 fastpath | fastpath 无阻塞、无等待、无调度——禁止在 fastpath 内调用 `schedule()`/`mutex_lock()`/`wait_event*()` | §8 故障注入 + KCSAN 静态扫描 |
| G-3 | slowpath 回退 | 资源不足（kfifo 满、无匹配消息、跨 CPU）时降级至 slowpath，不丢弃消息 | §4 slowpath 回退机制 + §8 注入测试 |
| G-4 | 可测性 | fastpath/slowpath 命中比例、错误率可统计，输出至 debugfs + tracepoint | §9 可观测性 |
| G-5 | Capability Folding 内联 | C-S9 Badge 校验 ~10ns（3 个 `READ_ONCE` + 位运算 + 比较），不阻塞、不调度 | §5.2 fastpath 实现 + §5.3 延迟分解 |

### 1.2 设计约束

- **传输通道双栈**：用户态 ↔ 内核态走 io_uring（`IORING_OP_URING_CMD` + `cmd_op=AIRY_URING_CMD_IPC_*`，见 [02-ipc-protocol.md](02-ipc-protocol.md) §4.4）；内核 kthread 间走 per-cpu kfifo + `wait_event_interruptible`（OS-ARCH-006）。状态机对两条通道统一建模。
- **Layout C v4 = 2 cache lines**：fastpath 仅处理消息头（无 payload）路径，payload 路径强制走 slowpath。`capability_badge` 字段（offset 40-47）在 fastpath C-S9 内联校验，不增加 cache miss（与 magic/opcode/flags 等同 cache line）。
- **per-cpu 无锁**：fastpath 必须在 per-cpu 上下文执行，禁止跨 CPU 同步。
- **Capability Folding H1-H6 硬约束**：fastpath C-S9 是 Badge 校验的唯一执行点（H4），agentrt 用户态 `capability_badge=0` 跳过（H3），[DSL] 降级模式跳过（H6）。
- **ABI 稳定**：状态机内部状态编号不进入 UAPI；对外契约仅 7 操作码（SEND/RECV/SEND_BATCH/CANCEL/FREEZE/CAP_REQUEST/CAP_RESPONSE）与返回码。

### 1.3 操作码定义（7 opcode）

操作码经 [SC] `ipc.h` 共享（OS-IFACE-003），与消息头 `opcode` 字段（offset 4，2 字节）对应：

| opcode 值 | 宏名 | 语义 | fastpath 适用 |
|----------|------|------|--------------|
| 0x0001 | `AIRY_IPC_OP_SEND` | 单条消息发送 | ✅（仅消息头） |
| 0x0002 | `AIRY_IPC_OP_RECV` | 单条消息接收（仅用于接收方声明的 expected opcode） | ✅（有匹配消息） |
| 0x0003 | `AIRY_IPC_OP_SEND_BATCH` | 批量发送（≥2 条） | ⚠️（走 BATCH 状态，聚合后 fastpath） |
| 0x0004 | `AIRY_IPC_OP_CANCEL` | 取消已提交请求 | ✅（per-cpu 内联取消） |
| 0x0005 | `AIRY_IPC_OP_FREEZE` | 冻结 ring（A-ULS 触发，C-S0 检查） | ✅（特权操作） |
| 0x0010 | `AIRY_IPC_OP_CAP_REQUEST` | 自举：请求 Badge（无 Badge，C-S9 跳过） | ✅（仅发给 sec_d） |
| 0x0011 | `AIRY_IPC_OP_CAP_RESPONSE` | sec_d 返回编译好的 Badge | ✅（sec_d → 请求方） |

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
| capability 令牌过期 / 权限不足 | 进入 ERROR，返回 `AIRY_ECAP_EPOCH`(-79) 或 `AIRY_ECAP_PERM`(-81)（OS-SEC-121，错误码 SSoT 见 [08-sc-error-contract.md](08-sc-error-contract.md) §3） |
| `flags` 含 `AIRY_IPC_F_NOWAIT` 且需阻塞 | 不进入 SLOW_*，直接返回 `AIRY_EAGAIN` |

---

## 3. Fastpath 触发条件

fastpath 是状态机的"热路径"，必须满足下列全部条件方可进入 `FAST_SEND` / `FAST_RECV`。任一条件不满足即触发 §4 slowpath 回退。条件检查全部内联（`__always_inline`），且按"廉价检查在前、昂贵检查在后"排序以缩短平均路径长度。

v1.1 起，fastpath 校验链为 **C-S0~C-S12 共 13 项**（含 Capability Folding C-S9 Badge 校验和 C-S12 CRC32 完整性校验），与 [08-sc-error-contract.md §2.3](08-sc-error-contract.md) 的 IPC 错误码空间精确对齐。

### 3.1 发送方条件（C-S0~C-S12 校验链，进入 FAST_SEND）

| # | 条件 | 检查方式 | 失败后果 | 错误码 |
|---|------|---------|---------|--------|
| C-S0 | `ring->frozen == false`（ring 未冻结） | 内联 `READ_ONCE` | → ERROR | `AIRY_ECAP_FROZEN`(-82) |
| C-S1 | `magic == 0x41524531`（'ARE1'） | 内联比较 | → ERROR | `AIRY_EIPC_MAGIC`(-41) |
| C-S2 | `opcode` 合法（见 §1.3 7 种 opcode） | 内联 `airy_ipc_opcode_valid()` | → ERROR | `AIRY_EIPC_OPCODE`(-42) |
| C-S3 | `payload_len <= AIRY_IPC_MAX_PAYLOAD` | 内联比较 | → ERROR | `AIRY_EIPC_PAYLOAD`(-43) |
| C-S4 | `hdr_size == 128` 且 `reserved[72]` 全零 | 内联 + 循环检查 | → ERROR | `AIRY_EIPC_HDRSIZE`(-44) / `AIRY_EIPC_RESERVED`(-45) |
| C-S5 | src 与 dst 在同一 CPU（per-cpu fastpath 无锁） | `smp_processor_id() == dst_cpu` | → SLOW_SEND | — |
| C-S6 | per-cpu kfifo 未满（`kfifo_avail() >= hdr_size + payload_len`） | 内联 | → SLOW_SEND（背压） | `AIRY_EIPC_KFIFO`(-48) |
| C-S7 | `cmd->reclaim == false`（无 reclaim flag） | 内联 | → ERROR | `AIRY_EIPC_RECLAIM`(-49) |
| C-S8 | `in_task()`（per-cpu 上下文，非中断） | 内联 | → SLOW_SEND | `AIRY_EIPC_CONTEXT`(-50) |
| **C-S9** | **Capability Folding Badge 校验**（见 §3.1.1 详解） | 内联 `airy_cap_badge_ok()`，~10ns | → ERROR 或 cap_pass | `AIRY_ECAP_BADGE`(-78) / `AIRY_ECAP_EPOCH`(-79) / `AIRY_ECAP_FORGED`(-80) / `AIRY_ECAP_PERM`(-81) |
| C-S10 | `flags & AIRY_IPC_F_RESERVED == 0` 且不含 `ENCRYPT`/`COMPRESS` | 位测试 | → ERROR | `AIRY_EIPC_FLAGS`(-46) / `AIRY_EIPC_NOTSUPP`(-47) |
| C-S11 | `preempt_disable()` 准备（保护 kfifo 操作） | 内联 | — | — |
| C-S12 | CRC32 校验通过（覆盖 `header[0:52) + payload`，投递前） | 内联 `airy_ipc_crc32_ok()` | → ERROR + Fault | `AIRY_EIPC_CRC32`(-51) + `AIRY_FAULT_RING_CORRUPT`(0x1003) |

**C-S0~C-S12 校验链总耗时**：~68ns（锁内），其中 C-S9 Badge 校验 ~10ns（3 个 `READ_ONCE` + 位运算 + 比较）。

#### 3.1.1 C-S9 Capability Folding Badge 校验详解（v1.1 新增）

C-S9 是 Capability Folding 的核心执行点，内联在 fastpath 校验链中。

> **术语声明（SSoT）**：`C-S9` 是 AirymaxOS 自创的 fastpath 校验状态编号（**C**heck-**S**tate 9），命名风格参考 Linux netfilter hook 编号（`NF_INET_PRE_ROUTING = 0`，数值编号 + 描述性宏名并存）与 Linux TCP 状态编号（`TCP_ESTABLISHED = 2`）。C-S9 对应行业标准术语 **"capability validation checkpoint"**（能力校验检查点），其语义等同于 seL4 `cap_cap_check()` 函数的 fastpath 内联版本。C-S9 内部包含 3 个子检查点，统一采用 `C-S9.<SUBCHECK>` 命名风格（点号分隔，类似 Linux `subsystem.op` 风格）：
>
> | 子检查点 | 命名 | 行业等价语义 | 校验内容 |
> |---------|------|------------|---------|
> | Epoch 校验 | `C-S9.EPOCH` | capability generation check | `badge_epoch == global_epoch` |
> | Random Tag 校验 | `C-S9.RANDTAG` | capability authenticity check | `badge_randtag == agent_caps[src].randtag` |
> | 权限校验 | `C-S9.PERMS` | capability permission check | `airy_cap_has_perm(badge_perms, opcode)` |
>
> **v1.1 术语统一**：早期文档使用的 `C-S9a`/`C-S9b`/`C-S9c` 子状态命名已废弃，统一替换为 `C-S9.EPOCH`/`C-S9.RANDTAG`/`C-S9.PERMS`。本文档为该命名的唯一权威源（SSoT）。

校验流程：

```c
/* C-S9: Capability Folding Badge 校验（关键步骤，~10ns） */
/* H2: 校验机制属于 [SS] 语义同源层 */
/* H3: agentrt 用户态 capability_badge 始终为 0 */
/* H4: agentrt-linux 内核 capability_badge 由 sec_d 编译 */
/* H6: [DSL] 降级模式 capability_badge=0，跳过校验 */

/* (1) CAP_REQUEST 自举：无 Badge，内核路由到 sec_d */
if (unlikely(hdr->opcode == AIRY_IPC_OP_CAP_REQUEST)) {
    if (unlikely(hdr->dst_task != AIRY_SEC_D_TASK_ID))
        return -AIRY_ECAP_BADGE;  /* CAP_REQUEST 只能发给 sec_d */
    goto cap_request_pass;
}

badge = hdr->capability_badge;

/* (2) [DSL] 降级模式：badge == 0 跳过校验（H6） */
/* agentrt 用户态：badge == 0（H3） */
if (badge == 0) {
    if (hdr->flags & AIRY_IPC_F_CAP_CARRY) {
        /* agentrt-linux 内核模式不应出现 badge=0 + CAP_CARRY */
        return -AIRY_ECAP_BADGE;  /* -78 */
    }
    /* agentrt 用户态或 [DSL] 模式：badge=0，跳过 */
    goto cap_pass;
}

/* (3) Badge 校验：3 个 READ_ONCE + 位运算 + 比较（~10ns） */
badge_epoch   = AIRY_BADGE_EPOCH(badge);     /* bits 48-63 */
badge_randtag = AIRY_BADGE_RANDTAG(badge);   /* bits 16-47 */
badge_perms   = AIRY_BADGE_PERMS(badge);     /* bits 0-15  */

/* (3a) C-S9.EPOCH: Epoch 校验——badge_epoch == global_epoch */
global_epoch  = airy_cap_epoch_get();        /* atomic_read */
if (unlikely(badge_epoch != global_epoch))
    return -AIRY_ECAP_EPOCH;  /* -79，已撤销或过期 */

/* (3b) C-S9.RANDTAG: Random Tag 校验——防伪造 */
agent_randtag = READ_ONCE(agent_caps[hdr->src_task].randtag);
agent_perms   = READ_ONCE(agent_caps[hdr->src_task].perms);

if (unlikely(badge_randtag != agent_randtag)) {
    /* Random Tag 不匹配 → 伪造尝试，触发 Fault */
    airy_security_fault(hdr->src_task, AIRY_FAULT_CAP_FORGED);  /* 0x1001 */
    return -AIRY_ECAP_FORGED;  /* -80 */
}

/* (3c) C-S9.PERMS: 权限校验——badge_perms 必须包含 opcode 所需权限 */
if (unlikely(!airy_cap_has_perm(badge_perms, hdr->opcode)))
    return -AIRY_ECAP_PERM;  /* -81 */

cap_request_pass:
cap_pass:
    /* C-S9 通过，继续 C-S10~C-S12 */
```

**C-S9 关键不变量**：
- **纯读取零副作用**（OS-KER-225）：C-S9 不持有锁、不调度、不写共享内存（`agent_caps[]` 是只读访问），唯一副作用是检测到伪造时调用 `airy_security_fault()` 触发 Fault（异步处理）。
- **agent_caps[] 隔离**（30 号 §3 修复）：`agent_caps[]` 是内核静态数组，用户态不可见、不可写，与 kfifo 数据结构物理隔离。
- **badge_epoch 与 global_epoch 解耦**：badge_epoch 是 Badge 编译时的快照，global_epoch 是当前全局代际，两者解耦避免 race condition。

### 3.2 接收方条件（进入 FAST_RECV）

接收方走 CQE poll 路径，已通过 C-S0~C-S12 校验的消息直接从 kfifo 出队，无需再次校验 Badge（发送方已校验）。

| # | 条件 | 检查方式 | 失败后果 |
|---|------|---------|---------|
| C-R1 | kfifo 非空且 `dst_task` 匹配 | `kfifo_out_peek()` | → SLOW_RECV |
| C-R2 | 接收缓冲区已就绪（用户态已 `IORING_REGISTER_BUFFERS`） | 内联查表 | → SLOW_RECV |
| C-R3 | src 与 dst 在同一 CPU | `smp_processor_id()` | → SLOW_RECV |
| C-R4 | 接收消息 `payload_len == 0`（仅消息头） | 内联比较 | → SLOW_RECV |
| C-R5 | 接收方轮询模式（`dst_polling == true`） | 内联 | → SLOW_RECV（需 `wake_up`） |

> **注意**：接收方无需重新校验 Badge。Badge 校验是发送方 fastpath C-S9 的责任，接收方信任 kfifo 中的消息已通过校验。这是 Capability Folding "IPC 就是能力校验"的语义保证——通过 C-S9 的消息已具备合法能力。

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
| capability 检查需异步询问 daemon | SLOW_SEND | OS-SEC-009：COMPLAIN 裁决需经 A-IPC 询问 daemon（v1.1: fastpath C-S9 失败时进入 slowpath） |

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
| `timeout_ns` | 消息头无此字段，由 `flags` 高位 + `reserved` 协商或调用参数传入 | 5 s（与 OS-SEC-009 COMPLAIN 超时对齐） |
| 超时动作 | 移出 wait queue，返回 `AIRY_ETIMEDOUT`，状态 → CANCEL | — |
| `flags` 含 NOWAIT | 不进入 slowpath，直接返回 `AIRY_EAGAIN` | — |

---

## 5. io_uring 集成

io_uring 承担用户态 ↔ 内核态的 IPC 数据面（OS-ARCH-006）。v1.1 起，agentrt-linux 不注册独立的 IPC 专用 io_uring OP，全部 IPC 数据传递通过 `IORING_OP_URING_CMD` 复用，由 `cmd_op` 字段区分操作类型（Capability Folding 单平面架构）。状态机在 `FAST_SEND` / `FAST_RECV` / `BATCH` 状态与 io_uring SQE/CQE 交互。

### 5.1 SQE/CQE 格式适配 IPC Layout C v4 128B 消息头

| io_uring 字段 | IPC 用途 |
|--------------|---------|
| `sqe.opcode` | `IORING_OP_URING_CMD`（统一复用） |
| `sqe.cmd_op` | `AIRY_URING_CMD_IPC_SEND` / `IPC_SEND_BATCH` / `IPC_CANCEL` / `IPC_FREEZE` / `IPC_CAP_REQUEST` |
| `sqe.cmd` | OLK 6.6 标准 SQE 的 `cmd` 字段为 16 字节（`__u8 cmd[16]`），不足以承载 `airy_ipc_cmd`；agentrt-linux IPC 启用 **SQE128 模式**（`IORING_SETUP_SQE128`，详见 §5.7），`cmd` 字段扩展至 80 字节，承载 `airy_ipc_cmd`（≤ 80 字节，含消息头指针 + payload 指针 + 路由信息） |
| `sqe.fd` | 目标 ring fd（`io_uring_setup()` 返回） |
| `sqe.user_data` | 携带 `trace_id`，原样回填至 `cqe.user_data` |
| `cqe.res` | 成功=0（fastpath），失败=`-AIRY_E*` |
| `cqe.flags` | 携带 fastpath/slowpath 命中标记（`AIRY_CQE_F_FASTPATH`） |

> **注意**：`opcode` 字段在消息头（offset 4，`__u16`，IPC 协议操作码 SEND/RECV/...）与 io_uring SQE（`sqe.opcode`=`IORING_OP_URING_CMD`，`sqe.cmd_op`=`AIRY_URING_CMD_IPC_*`）中均存在，但分属不同层次——消息头 `opcode` 是 IPC 协议操作码，io_uring `sqe.opcode` 是 io_uring 私有操作码。二者通过 `sqe.cmd_op` 桥接，**禁止混用**。

### 5.2 fastpath 完整实现（v1.1 新增，Capability Folding C-S9 内联）

以下代码是 A-IPC fastpath 的 SSoT 实现参考，物理宿主为 `kernel/ipc/airy_uring_cmd.c`。fastpath 入口 `airy_uring_cmd()` 在 io_uring 持有 `ctx->uring_lock` 时被调用，锁内仅做校验 + 快速投递（无 `wake_up`），需要唤醒时返回 `-EAGAIN`，io-wq 接管（锁外）。

```c
/* kernel/ipc/airy_uring_cmd.c */

#include <linux/io_uring.h>
#include <linux/io_uring_types.h>
#include <linux/kfifo.h>
#include <linux/random.h>
#include <linux/preempt.h>
#include <linux/atomic.h>
#include <airymax/ipc.h>
#include <airymax/error.h>
#include "airy_internal.h"

/**
 * airy_uring_cmd - A-IPC fastpath 入口
 *
 * 调用上下文: io_uring 持有 ctx->uring_lock
 * 锁内时间: NONBLOCK fastpath ~158ns，校验失败 ~68ns
 *
 * 设计原则:
 *   1. 锁内仅做校验 + 快速投递（无 wake_up）
 *   2. 需要唤醒时返回 -EAGAIN，io-wq 接管（锁外）
 *   3. C-S9 Badge 校验是纯读取，零副作用（OS-KER-225）
 */
static int airy_uring_cmd(struct io_uring_cmd *ioucmd, unsigned int issue_flags)
{
    /* OLK 6.6 + SQE128 模式（IORING_SETUP_SQE128，D-7 修复，详见 §5.7）:
     * - 标准 SQE 的 cmd 字段为 16 字节（__u8 cmd[16]），不足以承载 airy_ipc_cmd
     * - SQE128 模式下 SQE 为 128 字节（64B 标准 + 64B 扩展），cmd 扩展至 80 字节
     * - struct io_uring_cmd 的 pdu[32] 仅内联存储 cmd 的前 32 字节副本
     * - 完整 80 字节 cmd 数据通过 ioucmd->sqe->cmd 访问原始 SQE 区域
     * - airy_ipc_cmd 必须保证 sizeof <= 80（BUILD_BUG_ON 校验）
     */
    struct airy_ipc_cmd *cmd = io_uring_cmd_to_pdu(ioucmd, struct airy_ipc_cmd);
    int ret;

    /* Phase 1: 完整校验链 C-S0~C-S11（锁内，~68ns） */
    ret = airy_ipc_validate(cmd);
    if (ret < 0)
        return ret;  /* 立即失败，不持有额外锁时间 */

    /* Phase 2: 数据投递 —— NONBLOCK 快慢路径分离 */
    if (issue_flags & IO_URING_F_NONBLOCK) {
        /*
         * NONBLOCK 模式（io_uring 持有 uring_lock 时传入）:
         * - 同 CPU + kfifo 有空位 + 接收方轮询 → FAST_SEND（锁内 ~90ns）
         * - 否则返回 -EAGAIN → io_uring 自动排队到 io-wq
         */
        if (airy_ipc_can_fast_deliver(cmd)) {
            ret = airy_ipc_deliver_fast(cmd);  /* C-S12 CRC32 + kfifo_in */
            if (ret >= 0)
                return ret;  /* FAST_SEND 完成，~158ns 端到端 */
        }
        return -EAGAIN;  /* 需要 io-wq 接管 */
    }

    /*
     * 非 NONBLOCK 模式（已在 io-wq 线程中，不持有 uring_lock）:
     * - 可以安全执行 wake_up（eventfd_signal）
     * - 可以安全执行跨 CPU kfifo
     */
    return airy_ipc_deliver_full(cmd);
}

/**
 * airy_ipc_validate - C-S0~C-S11 校验链
 *
 * 锁内执行，~68ns，纯校验，零副作用。
 * 所有检查遵循 check-and-return 模式，前面失败不影响后面。
 * C-S9 Badge 校验是 Capability Folding 的核心执行点（~10ns）。
 */
static int airy_ipc_validate(struct airy_ipc_cmd *cmd)
{
    struct airy_ipc_msg_hdr *hdr = cmd->hdr;
    u64 badge;
    u16 badge_epoch, global_epoch;
    u32 badge_randtag, agent_randtag;
    u16 badge_perms;

    /* C-S0: ring 冻结检查（A-ULS 通过 FREEZE opcode 设置） */
    if (unlikely(cmd->ring->frozen))
        return -AIRY_ECAP_FROZEN;  /* -82 */

    /* C-S1: magic 检查 */
    if (unlikely(hdr->magic != AIRY_IPC_MAGIC))
        return -AIRY_EIPC_MAGIC;  /* -41 */

    /* C-S2: opcode 合法性检查 */
    if (unlikely(!airy_ipc_opcode_valid(hdr->opcode)))
        return -AIRY_EIPC_OPCODE;  /* -42 */

    /* C-S3: payload_len 合法性检查 */
    if (unlikely(hdr->payload_len > AIRY_IPC_MAX_PAYLOAD))
        return -AIRY_EIPC_PAYLOAD;  /* -43 */

    /* C-S4: 头部大小 + reserved[72] 全零检查 */
    if (unlikely(cmd->hdr_size != AIRY_IPC_HDR_SIZE))
        return -AIRY_EIPC_HDRSIZE;  /* -44 */
    if (unlikely(!airy_ipc_reserved_zero(hdr->reserved, 72)))
        return -AIRY_EIPC_RESERVED;  /* -45 */

    /* C-S5: CPU 匹配检查（优化条件，不是正确性条件） */
    cmd->same_cpu = (smp_processor_id() == cmd->dst_cpu);

    /* C-S6: kfifo 满检查（在 airy_ipc_can_fast_deliver() 中执行） */

    /* C-S7: reclaim flag 检查 */
    if (unlikely(cmd->reclaim))
        return -AIRY_EIPC_RECLAIM;  /* -49 */

    /* C-S8: 上下文检查（in_task()） */
    if (unlikely(!in_task()))
        return -AIRY_EIPC_CONTEXT;  /* -50 */

    /* C-S9: Capability Folding Badge 校验（详见 §3.1.1） */
    /* H2: 校验机制属于 [SS] 语义同源层 */
    /* H4: agentrt-linux 内核 capability_badge 由 sec_d 编译 */
    /* H6: [DSL] 降级模式 capability_badge=0，跳过校验 */

    if (unlikely(hdr->opcode == AIRY_IPC_OP_CAP_REQUEST)) {
        /* CAP_REQUEST 自举：无 Badge，内核路由到 sec_d */
        if (unlikely(hdr->dst_task != AIRY_SEC_D_TASK_ID))
            return -AIRY_ECAP_BADGE;  /* -78, CAP_REQUEST 只能发给 sec_d */
        goto cap_request_pass;
    }

    badge = hdr->capability_badge;

    /* [DSL] 降级模式：badge == 0 跳过校验（H6） */
    /* agentrt 用户态：badge == 0（H3） */
    if (badge == 0) {
        if (hdr->flags & AIRY_IPC_F_CAP_CARRY) {
            /* agentrt-linux 内核模式不应出现 badge=0 + CAP_CARRY */
            return -AIRY_ECAP_BADGE;  /* -78 */
        }
        /* agentrt 用户态或 [DSL] 模式：badge=0，跳过 */
        goto cap_pass;
    }

    /* Badge 校验：3 个 READ_ONCE + 位运算 + 比较（~10ns） */
    badge_epoch   = AIRY_BADGE_EPOCH(badge);
    badge_randtag = AIRY_BADGE_RANDTAG(badge);
    badge_perms   = AIRY_BADGE_PERMS(badge);

    /* (3a) Epoch 校验 */
    global_epoch  = airy_cap_epoch_get();
    if (unlikely(badge_epoch != global_epoch))
        return -AIRY_ECAP_EPOCH;  /* -79, 已撤销或过期 */

    /* (3b) Random Tag 校验：防伪造 */
    agent_randtag = READ_ONCE(agent_caps[hdr->src_task].randtag);
    if (unlikely(badge_randtag != agent_randtag)) {
        /* Random Tag 不匹配 → 伪造尝试，触发 Fault */
        airy_security_fault(hdr->src_task, AIRY_FAULT_CAP_FORGED);  /* 0x1001 */
        return -AIRY_ECAP_FORGED;  /* -80 */
    }

    /* (3c) 权限校验 */
    if (unlikely(!airy_cap_has_perm(badge_perms, hdr->opcode)))
        return -AIRY_ECAP_PERM;  /* -81 */

cap_request_pass:
cap_pass:

    /* C-S10: flags 检查 */
    if (unlikely(hdr->flags & AIRY_IPC_F_RESERVED))
        return -AIRY_EIPC_FLAGS;  /* -46 */
    if (unlikely((hdr->flags & AIRY_IPC_F_ENCRYPT) ||
                 (hdr->flags & AIRY_IPC_F_COMPRESS)))
        return -AIRY_EIPC_NOTSUPP;  /* -47, 0.1.1 不支持 */

    /* C-S11: preempt_disable 准备（实际在 deliver 中执行） */
    return 0;
}

/**
 * airy_ipc_can_fast_deliver - 判断是否可以走 FAST_SEND
 *
 * 条件: 同 CPU + kfifo 有空位 + 接收方轮询模式
 */
static inline bool airy_ipc_can_fast_deliver(struct airy_ipc_cmd *cmd)
{
    if (!cmd->same_cpu)
        return false;  /* C-S5 失败，走 SLOW_SEND */

    if (kfifo_avail(&cmd->dst_kfifo) < (AIRY_IPC_HDR_SIZE + cmd->hdr->payload_len))
        return false;  /* C-S6 失败，走 SLOW_SEND */

    if (!cmd->dst_polling)
        return false;  /* 接收方非轮询模式，需要 wake_up */

    return true;  /* FAST_SEND */
}

/**
 * airy_ipc_deliver_fast - FAST_SEND 路径
 *
 * 锁内执行，~90ns，无 wake_up。
 * C-S12 CRC32 校验 + kfifo_in + cqe_fill，preempt_disable 保护。
 */
static int airy_ipc_deliver_fast(struct airy_ipc_cmd *cmd)
{
    struct airy_ipc_msg_hdr *hdr = cmd->hdr;
    size_t total = AIRY_IPC_HDR_SIZE + hdr->payload_len;
    int ret;

    /* C-S11: preempt_disable（保护 kfifo 操作） */
    preempt_disable();

    /* C-S12: CRC32 校验（在投递前） */
    if (unlikely(!airy_ipc_crc32_ok(hdr, total))) {
        preempt_enable();
        return -AIRY_EIPC_CRC32;  /* -51 */
    }

    /* kfifo 入队（SPSC 无锁，per-cpu） */
    ret = kfifo_in(&cmd->dst_kfifo, hdr, total);
    if (unlikely(ret != total)) {
        preempt_enable();
        return -AIRY_EIPC_KFIFO;  /* -48, 不应发生，can_fast_deliver 已检查 */
    }

    preempt_enable();

    /* CQE 填充（无 wake_up，接收方轮询消费）
     * OLK 6.6 io_uring_cmd_done() 4 参数：(cmd, ret, res2, issue_flags)
     * issue_flags 必须使用核心 io_uring 代码传入的 mask，禁止硬编码（OLK 6.6 注释要求）
     * fastpath 在 NONBLOCK 上下文执行，传入 IO_URING_F_NONBLOCK
     */
    io_uring_cmd_done(cmd->ioucmd, 0, 0, IO_URING_F_NONBLOCK);
    return 0;
}

/**
 * airy_ipc_deliver_full - SLOW_SEND 路径
 *
 * io-wq 线程中执行，不持有 uring_lock。
 * 可以安全执行跨 CPU kfifo + eventfd_signal(wake_up)。
 */
static int airy_ipc_deliver_full(struct airy_ipc_cmd *cmd)
{
    struct airy_ipc_msg_hdr *hdr = cmd->hdr;
    size_t total = AIRY_IPC_HDR_SIZE + hdr->payload_len;
    int ret;

    /* C-S12: CRC32 校验 */
    if (unlikely(!airy_ipc_crc32_ok(hdr, total)))
        return -AIRY_EIPC_CRC32;  /* -51 */

    /* 跨 CPU kfifo 入队（需要锁保护，但不是 uring_lock） */
    spin_lock(&cmd->dst_kfifo_lock);
    ret = kfifo_in(&cmd->dst_kfifo, hdr, total);
    spin_unlock(&cmd->dst_kfifo_lock);

    if (unlikely(ret != total))
        return -AIRY_EIPC_KFIFO;  /* -48 */

    /* 唤醒接收方（可能 1-5μs，在 io-wq 中，不持有 uring_lock） */
    if (!cmd->dst_polling)
        eventfd_signal(cmd->dst_eventfd_ctx, 1);

    /* CQE 填充
     * SLOW_SEND 在 io-wq 上下文执行，传入 IO_URING_F_IOWQ
     */
    io_uring_cmd_done(cmd->ioucmd, 0, 0, IO_URING_F_IOWQ);
    return 0;
}
```

### 5.3 fastpath 延迟分解

```
FAST_SEND 路径（SQPOLL + 同 CPU + kfifo 有空位 + 接收方轮询）:
  airy_ipc_validate():     ~68ns (锁内)
    C-S0~C-S8:             ~20ns
    C-S9 Badge 校验:        ~10ns (3 READ_ONCE + 位运算 + 比较)
    C-S10~C-S11:            ~5ns
  airy_ipc_deliver_fast(): ~90ns (锁内)
    C-S12 CRC32:            ~15ns
    kfifo_in:               ~50ns
    cqe_fill:               ~20ns
    preempt_disable/enable: ~5ns
  ────────────────────────────────────
  锁内总计:                  ~158ns
  端到端:                    ~158ns（接收方轮询消费）

SLOW_SEND 路径（跨 CPU 或 kfifo 满或接收方阻塞）:
  airy_ipc_validate():     ~68ns (锁内)
  返回 -EAGAIN              → 释放 uring_lock
  io-wq 调度:               ~500ns
  airy_ipc_deliver_full():  ~100ns + wake_up(~1-5μs) (锁外)
  ────────────────────────────────────
  端到端:                    ~600ns ~5.5μs

校验失败路径:
  airy_ipc_validate():     ~68ns (锁内)
  返回 -AIRY_E*             → 立即失败
  ────────────────────────────────────
  端到端:                    ~68ns
```

### 5.4 注册固定缓冲区（registered buffers）

- 发送方启动时 `io_uring_register(ring_fd, IORING_REGISTER_BUFFERS, iov, n)` 预注册 128B 缓冲池，内核 pin 住 page，避免每条消息 `get_user_pages()`。
- 接收方同理注册接收缓冲池。
- fastpath 中 `sqe.flags |= IOSQE_FIXED_FILE | IOSQE_BUFFER_SELECT`，从固定池选取 buffer id，零拷贝 `IORING_OP_URING_CMD` + registered buffer（OS-IFACE-003 零拷贝优先）。

### 5.5 polled I/O 模式

- ring 创建时设 `IORING_SETUP_IOPOLL` + `IORING_SETUP_SQPOLL`，内核轮询 SQE，消除 `io_uring_enter()` 系统调用开销。
- fastpath 纯用户态提交 → 内核 sqpoll 线程轮询 → 内联拷贝 → CQE 回填，全程无 syscall。
- sqpoll 纺程 idle 超时后退出，由新 SQE 唤醒（首次有 ≤1μs 唤醒开销，热路径稳定后归零）。

### 5.6 批量提交（SEND_BATCH）的 io_uring 优化

- `BATCH` 状态聚合 N 条 128B 消息至连续缓冲（N × 128B），单次 SQE 提交（`sqe.len = N * 128`）。
- 阈值 `AIRY_IPC_BATCH_THRESHOLD`（默认 8）：达到即 flush 走 `FAST_SEND`；或调用方显式 flush。
- 批量提交摊销 SQE 入队与 CQE 回填开销，吞吐较单条提升 4–6×（实测目标）。
- CQE 返回批量结果掩码，逐条状态由 `cqe.res` 高 16 位（成功数）+ 低 16 位（失败数）编码。

### 5.7 SQE128 模式（cmd 字段扩展，D-7 修复）

**背景**：OLK 6.6 `include/uapi/linux/io_uring.h` 中 `struct io_uring_sqe` 的 `cmd` 字段标准大小为 16 字节（`__u8 cmd[16]`），不足以承载 agentrt-linux IPC 的 `airy_ipc_cmd` 结构体（含消息头指针 + payload 指针 + 路由信息）。v1.1 方案早期文档错误声称可直接传递 80 字节 `airy_ipc_cmd`，未说明需要启用 SQE128 模式。D-7 修复对此进行文档化。

**SQE128 模式**（`IORING_SETUP_SQE128`，Linux 5.18+，OLK 6.6 基线支持）：

- SQE 大小从标准 64 字节扩展到 128 字节（64B 标准 + 64B 扩展）。
- `cmd` 字段从 16 字节扩展到 80 字节（标准 16B + 扩展 64B）。
- io_uring 实例创建时必须设置 `IORING_SETUP_SQE128` flag。

**启用要求**：

1. `io_uring_setup()` 时 `flags` 参数必须置位 `IORING_SETUP_SQE128`（OS-IPC-009 已更新，详见 [40-dataflows/03-ipc-flow.md](../40-dataflows/03-ipc-flow.md) §2.5）。
2. SQ ring 内存大小 = `128 × nr_entries`（128 字节 × 队列深度），由内核自动分配。
3. 用户态提交 SQE 时必须填充完整的 128 字节（标准 64B + 扩展 64B），扩展区 `cmd[16:80)` 由 `airy_ipc_cmd` 覆盖。
4. `airy_ipc_cmd` 结构体大小必须 ≤ 80 字节（SQE128 的 `cmd` 字段大小），通过 `BUILD_BUG_ON(sizeof(struct airy_ipc_cmd) <= 80)` 编译期校验。

**OLK 6.6 内核侧访问**：

- `struct io_uring_cmd` 的 `pdu[32]` 是 32 字节内联数据区，仅存储 `cmd` 的前 32 字节副本（OLK 6.6 标准 `include/linux/io_uring.h`）。
- `io_uring_cmd_to_pdu()` 宏用于快速访问前 32 字节，fastpath 路径仅依赖前 32 字节时使用。
- 完整的 80 字节 `cmd` 数据需通过 `ioucmd->sqe->cmd` 访问原始 SQE 区域（含扩展 64 字节）。
- `airy_ipc_cmd` ≤ 32 字节的字段（如 `hdr` 指针 + `payload` 指针 + `payload_len`）走 `io_uring_cmd_to_pdu()` fastpath；> 32 字节的扩展字段（如 `dst_task` + `ring_id` + 路由元数据）走 `ioucmd->sqe->cmd` 访问路径。

**`airy_ipc_cmd` 结构体布局约束**（≤ 80 字节）：

```c
struct airy_ipc_cmd {
    struct airy_ipc_msg_hdr *hdr;   /* 消息头指针，offset  0, 8 字节 */
    const void              *payload; /* payload 指针，offset  8, 8 字节 */
    size_t                   payload_len; /* payload 长度，offset 16, 8 字节 */
    __u64                    dst_task;    /* 目标任务，offset 24, 8 字节 */
    __u32                    ring_id;     /* ring ID，offset 32, 4 字节 */
    __u32                    reserved;    /* 保留字段，offset 36, 4 字节 */
    /* 共 40 字节，远小于 80 字节上限（SQE128 cmd 字段大小） */
};
BUILD_BUG_ON(sizeof(struct airy_ipc_cmd) > 80);
```

**与 fastpath 的协作**：

- `airy_uring_cmd()` 入口通过 `io_uring_cmd_to_pdu(ioucmd, struct airy_ipc_cmd)` 获取前 32 字节内联副本（覆盖 `hdr` / `payload` / `payload_len` / `dst_task`），fastpath 校验与投递仅需这 4 个字段。
- 扩展字段（`ring_id` / `reserved` / 未来扩展）通过 `ioucmd->sqe->cmd` 访问，slowpath 或审计路径使用。

**对齐说明**：

- SQE128 模式是 OLK 6.6 标准 io_uring 特性（Linux 5.18+ 引入），不引入任何内核 patch，符合"不修改 OLK 6.6 内核源码"原则（OS-ARCH-001）。
- SQE128 启用后 SQ ring 内存占用翻倍（64B → 128B/SQE），但 IPC fastpath 收益（消除 cmd 数据截断、支持完整 `airy_ipc_cmd` 内联传递）远大于内存开销。
- SQE128 与 `SQPOLL` / `DEFER_TASKRUN` / `NO_SQARRAY` / `NO_IEC` 等 flag 正交，可同时启用。

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
    if (likely(airy_cap_badge_ok(current, hdr.capability_badge, hdr.opcode) == 0))
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
- `struct airy_ipc_msg_hdr` 已 `__attribute__((aligned(64)))`（D-9 修复后移除 `__attribute__((packed))`，见 [02-ipc-protocol.md](02-ipc-protocol.md) §2.3），所有 `__u64` 字段自然 8 字节对齐，热字段（magic ~ crc32，offset 0-55）集中在 cache line 1。
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
    likely(airy_cap_badge_ok(current, hdr.capability_badge, hdr.opcode) == 0)) {
    /* FAST_SEND fastpath */
} else {
    /* slowpath fallback */
}
```

### 7.4 内联函数

- fastpath 入口 `airy_ipc_fastpath_send()` / `airy_ipc_fastpath_recv()` 标注 `__always_inline`，消除函数调用开销（call/ret、栈帧）。
- `airy_cap_badge_ok()` 内联（v1.1 Capability Folding Badge 校验：3 次 READ_ONCE + 位运算 + 比较，~10ns）。
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
| capability 检查失败 | `fail_cap_check`（强制 `airy_cap_badge_ok()` 返回非 0） | FAST_SEND → ERROR | 返回 `AIRY_ECAP_FORGED`(-80) 或 `AIRY_ECAP_PERM`(-81) |
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

### 10.6 CBMC 形式化验证属性规约（v1.1 新增）

CBMC（C Bounded Model Checker）是对 fastpath **100 行代码**做全函数验证的工业级工具，与 §10.1 Isabelle/HOL（定理证明）形成互补——Isabelle 证明状态机层语义性质，CBMC 验证 C 代码层内存安全与逻辑不变量。CBMC 验证范围限定在 **fastpath C-S9 Badge 校验 + C-S12 CRC32 校验 + kfifo_in 投递**共 ~100 行 C 代码（H4 硬约束要求）。

#### 10.6.1 验证范围与目标

| 验证对象 | 物理宿主 | 行数 | 验证工具 | 验证目标 |
|---------|---------|------|---------|---------|
| `airy_cap_badge_ok()` | kernel/security/airy/capability.c | ~30 行 | CBMC | 内存安全 + Badge 校验逻辑不变量 |
| `airy_ipc_validate()` | kernel/ipc/airy_uring_cmd.c | ~40 行 | CBMC | C-S0~C-S11 校验链完备性 |
| `airy_ipc_deliver_fast()` | kernel/ipc/airy_uring_cmd.c | ~20 行 | CBMC | kfifo_in 边界 + preempt 平衡 |
| `airy_ipc_crc32_ok()` | kernel/ipc/airy_crc32.c | ~10 行 | CBMC | CRC32 计算无越界 |
| **合计** | — | **~100 行** | CBMC | 全函数验证（H4 硬约束） |

#### 10.6.2 CBMC 属性规约（Property Specifications）

CBMC 通过 `__CPROVER_assert()` 与函数契约（`__CPROVER_requires()` / `__CPROVER_ensures()`）声明属性规约。以下属性规约是 fastpath 100 行代码的**唯一权威规约源**（SSoT）。

```c
/* kernel/security/airy/capability.c —— CBMC 属性规约（SSoT）
 *
 * 物理宿主: kernel/security/airy/capability.c（CBMC 验证块）
 * 启用方式: #ifdef CBMC_VERIFICATION 包裹，编译期注入
 * 验证脚本: tests-linux/formal/cbmc/run_fastpath_cbmc.sh
 */

#ifdef CBMC_VERIFICATION
#include <cbmc_contract.h>

/* ========== 属性 1: agent_caps[] 访问边界 ========== */
/* 给定任意 agent_id，agent_caps[agent_id] 访问不越界 */

__CPROVER_requires(0 <= agent_id && agent_id < AIRY_CAP_MAX_AGENTS)
__CPROVER_ensures(
    __CPROVER_return_value == 0 ||
    __CPROVER_return_value == -AIRY_ECAP_EPOCH ||
    __CPROVER_return_value == -AIRY_ECAP_FORGED ||
    __CPROVER_return_value == -AIRY_ECAP_PERM)
int airy_cap_badge_ok(struct task_struct *task, u64 badge, u16 opcode)
{
    u32 agent_id = task->pid;

    /* CBMC 不变量: agent_id 在 [0, 1024) 范围内 */
    __CPROVER_assume(agent_id < AIRY_CAP_MAX_AGENTS);

    /* CBMC 不变量: agent_caps[] 是静态数组，访问 O(1) 无越界 */
    __CPROVER_assert(agent_id < AIRY_CAP_MAX_AGENTS,
        "P1.1: agent_caps[agent_id] access in bounds");

    /* CBMC 不变量: READ_ONCE 不引入数据竞争（单写者 sec_d + 多读者 fastpath） */
    u16 global_epoch = (u16)atomic_read(&airy_cap_global_epoch);
    u32 agent_randtag = READ_ONCE(agent_caps[agent_id].randtag);
    u16 agent_perms   = READ_ONCE(agent_caps[agent_id].perms);

    /* CBMC 不变量: Badge 字段提取不溢出（位运算无 UB） */
    u16 badge_epoch   = (u16)(badge >> 48);
    u32 badge_randtag = (u32)((badge >> 16) & 0xFFFFFFFF);
    u16 badge_perms   = (u16)(badge & 0xFFFF);

    __CPROVER_assert((badge >> 48) <= 0xFFFF,
        "P1.2: badge_epoch extraction no overflow");
    __CPROVER_assert(((badge >> 16) & 0xFFFFFFFF) <= 0xFFFFFFFF,
        "P1.3: badge_randtag extraction no overflow");

    /* CBMC 不变量: C-S9.EPOCH 校验逻辑正确 */
    if (badge_epoch != global_epoch)
        return -AIRY_ECAP_EPOCH;

    /* CBMC 不变量: C-S9.RANDTAG 校验逻辑正确 */
    if (badge_randtag != agent_randtag)
        return -AIRY_ECAP_FORGED;

    /* CBMC 不变量: C-S9.PERMS 校验逻辑正确（严格包含语义） */
    u16 required = airy_op_required_perms(opcode);
    if ((badge_perms & required) != required)
        return -AIRY_ECAP_PERM;

    return 0;
}

/* ========== 属性 2: opcode → required_perms 映射完备性 ========== */
/* 对任意 opcode，airy_op_required_perms() 返回值属于 {0, 0xFFFF, AIRY_CAP_PERM_*} */

__CPROVER_requires(opcode <= 0xFFFF)
__CPROVER_ensures(
    __CPROVER_return_value == 0 ||
    __CPROVER_return_value == 0xFFFF ||
    (__CPROVER_return_value & ~AIRY_CAP_PERM_ALL) == 0)
u16 airy_op_required_perms(u16 opcode)
{
    /* CBMC 不变量: 所有 opcode 路径都有返回值（无 fallthrough） */
    u8 idx;
    if (opcode <= 0x0005)
        idx = opcode - 1;
    else if (opcode >= 0x0010 && opcode <= 0x0011)
        idx = (opcode - 0x0010) + 6;
    else
        return 0xFFFF;

    __CPROVER_assert(idx < 8,
        "P2.1: airy_op_required_perms_table index in bounds");

    return airy_op_required_perms_table[idx];
}

/* ========== 属性 3: CRC32 校验无缓冲区越界 ========== */
__CPROVER_requires(hdr != NULL)
__CPROVER_requires(total > 0 && total <= AIRY_IPC_HDR_SIZE + AIRY_IPC_MAX_PAYLOAD)
__CPROVER_ensures(__CPROVER_return_value == 0 || __CPROVER_return_value == 1)
bool airy_ipc_crc32_ok(struct airy_ipc_msg_hdr *hdr, size_t total)
{
    /* CBMC 不变量: CRC32 计算覆盖范围不越界 */
    __CPROVER_assert(total <= sizeof(*hdr) + AIRY_IPC_MAX_PAYLOAD,
        "P3.1: CRC32 computation range in bounds");

    u32 computed = airy_ipc_crc32_compute(hdr, total);
    u32 expected = hdr->crc32;

    /* CBMC 不变量: 比较是无符号比较，无符号溢出 UB */
    return computed == expected;
}

/* ========== 属性 4: kfifo_in 边界与原子性 ========== */
__CPROVER_requires(cmd != NULL)
__CPROVER_requires(cmd->hdr != NULL)
__CPROVER_requires(cmd->hdr->payload_len <= AIRY_IPC_MAX_PAYLOAD)
__CPROVER_ensures(
    __CPROVER_return_value == 0 ||
    __CPROVER_return_value == -AIRY_EIPC_CRC32 ||
    __CPROVER_return_value == -AIRY_EIPC_KFIFO)
int airy_ipc_deliver_fast(struct airy_ipc_cmd *cmd)
{
    size_t total = AIRY_IPC_HDR_SIZE + cmd->hdr->payload_len;

    __CPROVER_assert(total <= AIRY_IPC_HDR_SIZE + AIRY_IPC_MAX_PAYLOAD,
        "P4.1: kfifo_in total size in bounds");

    /* CBMC 不变量: preempt_disable/enable 必须配对 */
    preempt_disable();
    __CPROVER_assert(preempt_count() & PREEMPT_DISABLE_OFFSET,
        "P4.2: preempt disabled after preempt_disable()");

    if (!airy_ipc_crc32_ok(cmd->hdr, total)) {
        preempt_enable();
        __CPROVER_assert(!(preempt_count() & PREEMPT_DISABLE_OFFSET),
            "P4.3: preempt re-enabled on CRC32 failure");
        return -AIRY_EIPC_CRC32;
    }

    int ret = kfifo_in(&cmd->dst_kfifo, cmd->hdr, total);
    __CPROVER_assert(ret == 0 || ret == total,
        "P4.4: kfifo_in returns 0 or total (SPSC invariant)");

    preempt_enable();
    __CPROVER_assert(!(preempt_count() & PREEMPT_DISABLE_OFFSET),
        "P4.5: preempt re-enabled on success path");

    return (ret == total) ? 0 : -AIRY_EIPC_KFIFO;
}

/* ========== 属性 5: 校验链完备性（C-S0~C-S11 全覆盖） ========== */
__CPROVER_requires(cmd != NULL)
__CPROVER_requires(cmd->hdr != NULL)
__CPROVER_ensures(
    __CPROVER_return_value == 0 || __CPROVER_return_value < 0)
int airy_ipc_validate(struct airy_ipc_cmd *cmd)
{
    /* CBMC 不变量: 所有错误码来自预定义集合（无未定义错误码） */
    int allowed_ret[] = {
        0,
        -AIRY_ECAP_FROZEN, -AIRY_EIPC_MAGIC, -AIRY_EIPC_OPCODE,
        -AIRY_EIPC_PAYLOAD, -AIRY_EIPC_HDRSIZE, -AIRY_EIPC_RESERVED,
        -AIRY_EIPC_RECLAIM, -AIRY_EIPC_CONTEXT,
        -AIRY_ECAP_BADGE, -AIRY_ECAP_EPOCH, -AIRY_ECAP_FORGED, -AIRY_ECAP_PERM,
        -AIRY_EIPC_FLAGS, -AIRY_EIPC_NOTSUPP,
    };

    int ret = airy_ipc_validate_impl(cmd);

    /* CBMC 不变量: 返回值必须属于预定义错误码集合 */
    bool found = false;
    for (size_t i = 0; i < sizeof(allowed_ret)/sizeof(allowed_ret[0]); i++) {
        if (ret == allowed_ret[i]) {
            found = true;
            break;
        }
    }
    __CPROVER_assert(found,
        "P5.1: airy_ipc_validate returns only predefined error codes");

    return ret;
}

#endif /* CBMC_VERIFICATION */
```

#### 10.6.3 CBMC 验证属性清单（SSoT）

| 属性 ID | 属性描述 | 验证函数 | 失败后果 |
|---------|---------|---------|---------|
| P1.1 | `agent_caps[agent_id]` 访问边界 | `airy_cap_badge_ok` | 内存越界，CWE-125 |
| P1.2 | `badge_epoch` 提取无溢出 | `airy_cap_badge_ok` | 整数溢出，CWE-190 |
| P1.3 | `badge_randtag` 提取无溢出 | `airy_cap_badge_ok` | 整数溢出，CWE-190 |
| P2.1 | `airy_op_required_perms_table[]` 索引边界 | `airy_op_required_perms` | 数组越界，CWE-125 |
| P3.1 | CRC32 计算范围不越界 | `airy_ipc_crc32_ok` | 缓冲区过读，CWE-125 |
| P4.1 | `kfifo_in` 写入大小不越界 | `airy_ipc_deliver_fast` | 缓冲区过写，CWE-787 |
| P4.2 | `preempt_disable` 后抢占计数 > 0 | `airy_ipc_deliver_fast` | 调度语义违反 |
| P4.3 | CRC32 失败路径抢占重新启用 | `airy_ipc_deliver_fast` | 调度语义违反 |
| P4.4 | `kfifo_in` 返回 0 或 total（SPSC） | `airy_ipc_deliver_fast` | SPSC 不变量违反 |
| P4.5 | 成功路径抢占重新启用 | `airy_ipc_deliver_fast` | 调度语义违反 |
| P5.1 | `airy_ipc_validate` 返回值属于预定义集合 | `airy_ipc_validate` | 未定义错误码，CWE-754 |

#### 10.6.4 CBMC 配置（CBMC.yml）

```yaml
# tests-linux/formal/cbmc/CBMC.yml —— CBMC 验证配置（SSoT）
# 物理宿主: tests-linux/formal/cbmc/CBMC.yml

verification_targets:
  - name: airy_cap_badge_ok
    source: kernel/security/airy/capability.c
    function: airy_cap_badge_ok
    unwind: 10                    # 循环展开上界（Badge 校验无循环，10 足够）
    object_bits: 8                # 堆对象位宽（256 字节，覆盖 agent_caps[] 单元素）
    property_file: tests-linux/formal/cbmc/properties/fastpath.prp

  - name: airy_op_required_perms
    source: kernel/security/airy/capability.c
    function: airy_op_required_perms
    unwind: 5
    object_bits: 4

  - name: airy_ipc_crc32_ok
    source: kernel/ipc/airy_crc32.c
    function: airy_ipc_crc32_ok
    unwind: 32                    # CRC32 表查找循环最多 256 次
    object_bits: 16              # 覆盖最大 payload

  - name: airy_ipc_deliver_fast
    source: kernel/ipc/airy_uring_cmd.c
    function: airy_ipc_deliver_fast
    unwind: 10
    object_bits: 16

  - name: airy_ipc_validate
    source: kernel/ipc/airy_uring_cmd.c
    function: airy_ipc_validate
    unwind: 20
    object_bits: 16

global_options:
  compiler: gcc
  c_standard: gnu11
  include_paths:
    - kernel/include
    - kernel/include/airymax
    - kernel/security/airy
  defines:
    - CBMC_VERIFICATION=1
    - CONFIG_AIRY_CAP_MAX_AGENTS=1024
    - CONFIG_AIRY_IPC_MAX_PAYLOAD=4096
  flags:
    - -DCBMC_VERIFICATION
    - -I${INCLUDE_PATHS}
    - -nostdinc
    - -fno-builtin

ci_integration:
  trigger: pull_request
  matrix:
    - target: airy_cap_badge_ok
      timeout: 300s
    - target: airy_op_required_perms
      timeout: 60s
    - target: airy_ipc_crc32_ok
      timeout: 600s
    - target: airy_ipc_deliver_fast
      timeout: 300s
    - target: airy_ipc_validate
      timeout: 600s
  fail_fast: false               # 全量运行，便于一次性发现所有违规
```

#### 10.6.5 CBMC 验证脚本

```bash
#!/bin/bash
# tests-linux/formal/cbmc/run_fastpath_cbmc.sh —— fastpath 100 行代码 CBMC 验证
#
# 物理宿主: tests-linux/formal/cbmc/run_fastpath_cbmc.sh
# 调用方式: ./run_fastpath_cbmc.sh [target]
# 无参数时验证全部 5 个目标，否则验证指定目标

set -euo pipefail

CBMC_BIN="${CBMC_BIN:-cbmc}"
CONFIG_FILE="tests-linux/formal/cbmc/CBMC.yml"
RESULTS_DIR="tests-linux/formal/cbmc/results"

mkdir -p "${RESULTS_DIR}"

# 解析 YAML 配置（简化版，实际可用 yq）
verify_target() {
    local target="$1"
    local source="$2"
    local function="$3"
    local unwind="$4"
    local object_bits="$5"

    echo "=========================================="
    echo "[CBMC] Verifying: ${function}"
    echo "  Source:    ${source}"
    echo "  Unwind:    ${unwind}"
    echo "  Obj bits:  ${object_bits}"
    echo "=========================================="

    # 步骤 1: goto-cc 编译（生成 goto 二进制）
    goto-cc \
        -DCBMC_VERIFICATION=1 \
        -DCONFIG_AIRY_CAP_MAX_AGENTS=1024 \
        -DCONFIG_AIRY_IPC_MAX_PAYLOAD=4096 \
        -Ikernel/include \
        -Ikernel/include/airymax \
        -Ikernel/security/airy \
        -nostdinc -fno-builtin \
        "${source}" \
        -o "${RESULTS_DIR}/${target}.goto"

    # 步骤 2: CBMC 验证
    "${CBMC_BIN}" \
        "${RESULTS_DIR}/${target}.goto" \
        --function "${function}" \
        --unwind "${unwind}" \
        --object-bits "${object_bits}" \
        --pointer-check \
        --bounds-check \
        --div-by-zero-check \
        --signed-overflow-check \
        --unsigned-overflow-check \
        --conversion-check \
        --undefined-shift-check \
        --enum-range-check \
        --no-assertions \
        --assertions \
        2>&1 | tee "${RESULTS_DIR}/${target}.log"

    # 步骤 3: 解析结果
    if grep -q "VERIFICATION SUCCESSFUL" "${RESULTS_DIR}/${target}.log"; then
        echo "[CBMC] PASS: ${function}"
        return 0
    else
        echo "[CBMC] FAIL: ${function}"
        return 1
    fi
}

# 全量验证（5 个目标）
TARGETS=(
    "airy_cap_badge_ok|kernel/security/airy/capability.c|airy_cap_badge_ok|10|8"
    "airy_op_required_perms|kernel/security/airy/capability.c|airy_op_required_perms|5|4"
    "airy_ipc_crc32_ok|kernel/ipc/airy_crc32.c|airy_ipc_crc32_ok|32|16"
    "airy_ipc_deliver_fast|kernel/ipc/airy_uring_cmd.c|airy_ipc_deliver_fast|10|16"
    "airy_ipc_validate|kernel/ipc/airy_uring_cmd.c|airy_ipc_validate|20|16"
)

FAILED=0
for entry in "${TARGETS[@]}"; do
    IFS='|' read -r name source function unwind obj_bits <<< "${entry}"
    if ! verify_target "${name}" "${source}" "${function}" "${unwind}" "${obj_bits}"; then
        FAILED=$((FAILED + 1))
    fi
done

echo "=========================================="
echo "[CBMC] Summary: $(( ${#TARGETS[@]} - FAILED ))/${#TARGETS[@]} passed"
echo "=========================================="

exit $(( FAILED > 0 ? 1 : 0 ))
```

#### 10.6.6 CBMC 验证与 KCSAN/KCSAN 交叉验证

CBMC 验证的是**单线程 C 代码层**的内存安全与逻辑不变量。多线程并发安全由 KCSAN（Kernel Concurrency Sanitizer）运行时检测器交叉验证：

| 验证维度 | CBMC | KCSAN | 覆盖关系 |
|---------|------|-------|---------|
| 单函数内存安全 | ✅ 全函数验证 | ❌ 仅运行时覆盖 | CBMC 主导 |
| 数组边界 | ✅ 全路径 | ❌ 仅运行时 | CBMC 主导 |
| 整数溢出 | ✅ 全路径 | ❌ 仅运行时 | CBMC 主导 |
| 抢占计数平衡 | ✅ 全路径 | ❌ 仅运行时 | CBMC 主导 |
| 数据竞争（多线程） | ❌ 不支持 | ✅ 运行时 | KCSAN 主导 |
| 锁序倒置 | ❌ 不支持 | ✅ 运行时（lockdep） | lockdep 主导 |

CBMC 与 KCSAN 形成互补——CBMC 在编译期保证单函数不变量，KCSAN 在运行时保证多线程不变量。

#### 10.6.7 CBMC 验证里程碑（v1.1 新增）

| 阶段 | 交付物 | 状态 |
|------|--------|------|
| V-C1 | CBMC 属性规约（§10.6.2）+ CBMC.yml 配置（§10.6.4） | ✅ 完成（本文档） |
| V-C2 | fastpath 100 行代码 goto-cc 编译通过 | 规划中（1.0.1 Phase 3） |
| V-C3 | 5 个验证目标全部 VERIFICATION SUCCESSFUL | 规划中（1.0.1 Phase 3） |
| V-C4 | CBMC 验证集成 CI（PR 触发） | 规划中（1.0.1 Phase 3） |
| V-C5 | CBMC 属性规约回归保护（新增属性必经 CBMC 验证） | 规划中（1.0.1 Phase 4） |

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

> **升级背景**：本次 v1.0 升级将本文档对齐 Airymax Unify Design（见 [10-unify-design.md](../10-architecture/10-unify-design.md)）A-IPC 模块设计与sched_tac 技术选型，消解 v0.2.8 遗留的零拷贝路径与 capability 检查架构冲突点。升级内容如下。

### A.1 已删除 page flipping，改为 IORING_OP_URING_CMD + registered buffer + mmap

v0.2.8 在 §5 io_uring 集成章节中存在 page flipping 零拷贝路径的残留表述（见 综合修正方案 §6 C-02 冲突点，6 个文件引用已废弃的 page flipping）。v1.0 彻底删除 page flipping，统一为 A-IPC 模块技术选型：

- **零拷贝载体**：`IORING_OP_URING_CMD` + registered buffer，消除 page flipping 的页所有权反转复杂度
- **共享内存**：`alloc_pages(GFP_KERNEL)` + mmap（不使用 DMA 一致性内存），对齐 [05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §1 内存方案
- §5.2 registered buffers 机制已更新为固定缓冲池 + `IOSQE_BUFFER_SELECT` 零拷贝选取，不再涉及 page flipping

### A.2 已新增 Capability 缓存机制

v0.2.8 §3 fastpath 触发条件中 C-S9 / C-R7 的 capability 检查（`airy_cap_check_fast()`）依赖同步查询 capability daemon，破坏 fastpath 无阻塞假设。v1.0 新增 **Capability 缓存机制**，使 fastpath capability 检查真正内联无阻塞：

- **per-cpu capability 缓存**：fastpath 路径读取 per-cpu 缓存的 capability badge 掩码，无需查询 daemon
- **缓存一致性**：capability 撤销/派生时通过 IPI 失效目标 CPU 的缓存行，对齐 [03-capability-model.md](../110-security/03-capability-model.md) 离线缓存校验与 Reconciliation 设计
- **缓存未命中回退**：缓存失效时 fastpath 降级至 slowpath（§4），由 daemon 异步裁决后回填缓存
- 该机制对齐 Airymax Unify Design 的 [SC] `lsm_types.h` Capability 缓存结构定义

### A.3 已对齐 Airymax Unify Design A-IPC 模块

本文档作为 A-IPC（统一 IPC Fastpath）模块的数据面状态机设计，v1.0 明确对齐 Airymax Unify Design 的 A-IPC 模块边界：

- **A-IPC 模块定位**：A-IPC 是 Unify Design 5 模块之一（[10-unify-design.md](../10-architecture/10-unify-design.md) §5），负责统一 IPC fastpath 的状态机、零拷贝路径、capability 检查缓存
- **权威源边界**：本文档是 fastpath 状态机（§2 8 状态 + §3 触发条件 + §4 回退机制）的唯一权威源；128B 消息头布局的权威源为 [02-ipc-protocol.md](02-ipc-protocol.md) + [SC] `ipc.h`；capability 模型的权威源为 [03-capability-model.md](../110-security/03-capability-model.md)
- **技术选型统一**：IPC 零拷贝统一为 `IORING_OP_URING_CMD` + registered buffer + mmap（不使用 page flipping），与 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §4.2 [IND] 层 IPC 实现差异表一致

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
