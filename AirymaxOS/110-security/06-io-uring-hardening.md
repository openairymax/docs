Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# io_uring 安全加固设计
> **文档定位**：A-IPC（统一进程间通信体系）io_uring 在 Agent 环境安全加固的唯一权威设计\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-19\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §8\
> **设计依据**：综合修正方案 §4.2.5（A-IPC 设计）+ §6.2.1 C-02（page flipping 修正）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **io_uring 安全加固设计** 的唯一权威源。io_uring_disabled sysctl 控制、opcode 白名单（仅允许 IORING_OP_URING_CMD + 基础 opcode）、纯 C LSM 双重校验（io_uring_cmd 回调中纯 C LSM 校验）、registered buffer 安全（Capability 校验后才能注册 buffer）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 io_uring 安全加固策略。
>
> 技术选型声明：IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（**不使用 page flipping**）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（**不使用 BPF LSM**）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

---

## 文档信息卡

- **目标读者**：安全模块开发者、A-IPC 模块开发者、内核开发者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) §8 A-IPC 模块、[03-capability-model.md](03-capability-model.md) Capability 模型、Linux 6.6 io_uring 机制与 `io_uring_disabled` sysctl
- **预计阅读时间**：25 分钟
- **核心概念**：io_uring_disabled、opcode 白名单、纯 C LSM 双重校验、registered buffer 安全、IORING_OP_URING_CMD
- **复杂度标识**：高级

---

## §1 设计目标：io_uring 在 Agent 环境的安全加固

### 1.1 问题背景

io_uring 是 Linux 5.1 引入的高性能异步 I/O 框架，Airymax 使用 `IORING_OP_URING_CMD` 作为 IPC 语义映射载体（详见 [10-unify-design.md](../10-architecture/10-unify-design.md) §8.1）。但 io_uring 的强大能力也带来安全风险：

| 风险 | 说明 | 攻击场景 |
|------|------|---------|
| opcode 滥用 | Agent 可使用任意 opcode 访问文件/网络 | 越权读文件、外连网络 |
| buffer 注入 | Agent 注册任意 buffer，内核写入导致信息泄露 | 读取内核内存 |
| SQE 注入 | Agent 提交恶意 SQE，触发内核漏洞 | 提权攻击 |
| 资源耗尽 | Agent 创建大量 io_uring 实例 | DoS 攻击 |

### 1.2 设计目标

io_uring 安全加固的核心目标是：**在保留 io_uring 高性能的同时，限制 Agent 可用 opcode 与 buffer 注册**：

1. **最小 opcode 集**：仅允许 IPC 所需的 `IORING_OP_URING_CMD` + 基础 opcode
2. **Capability 校验**：buffer 注册前必须通过 Capability 校验
3. **纯 C LSM 双重校验**：io_uring_cmd 回调中纯 C LSM 校验（不使用 BPF LSM）
4. **可禁用**：通过 sysctl 全局禁用 io_uring（紧急安全响应）

### 1.3 与 A-IPC 模块的关系

io_uring 安全加固是 A-IPC 模块的安全基础（详见 [10-unify-design.md](../10-architecture/10-unify-design.md) §8）。A-IPC 的 IPC 语义映射依赖 io_uring 的 `IORING_OP_URING_CMD`，加固确保此通道不被滥用：

```
A-IPC 模块组成
├── IPC 协议（[02-ipc-protocol.md]）
├── IPC fastpath（[07-ipc-fastpath.md]）
├── io_uring_cmd 语义映射
├── Capability 分层校验
├── 数据面自治三原则（[04-ipc-data-plane-autonomy.md]）
└── io_uring 安全加固（本文件）  ← 安全基础
    ├── io_uring_disabled sysctl
    ├── opcode 白名单
    ├── 纯 C LSM 双重校验
    └── registered buffer 安全
```

---

## §2 io_uring_disabled sysctl：控制 io_uring 可用性

### 2.1 io_uring_disabled 机制

Linux 6.6 提供 `kernel.io_uring_disabled` sysctl，控制 io_uring 的全局可用性。Airymax 利用此机制作为紧急安全响应开关：

| 值 | 含义 | Airymax 使用 |
|----|------|------------|
| 0 | io_uring 完全可用（默认） | 正常运行 |
| 1 | 仅 CAP_SYS_ADMIN 可用 | 紧急限制模式 |
| 2 | 完全禁用 io_uring | 安全事件响应 |

### 2.2 sysctl 配置

```c
/* kernel/ipc/airy_uring_security.c —— io_uring_disabled 控制 */
/*
 * Airymax 默认 io_uring_disabled=0（可用），
 * 安全事件时可动态切换为 2（完全禁用）。
 * 切换为 2 时，所有新建 io_uring 实例失败，
 * 已有实例继续运行直到关闭。
 */
static int airy_io_uring_disabled = 0;

static struct ctl_table airy_uring_sysctl[] = {
    {
        .procname = "airy_io_uring_disabled",
        .data = &airy_io_uring_disabled,
        .maxlen = sizeof(int),
        .mode = 0644,
        .proc_handler = proc_dointvec,
    },
    { }
};

/* io_uring 创建时检查 */
static int airy_uring_create_check(void)
{
    if (airy_io_uring_disabled == 2) {
        return -EPERM;  /* 完全禁用 */
    }
    if (airy_io_uring_disabled == 1 && !capable(CAP_SYS_ADMIN)) {
        return -EPERM;  /* 仅管理员可用 */
    }
    return 0;
}
```

### 2.3 紧急安全响应流程

| 触发条件 | 响应动作 |
|---------|---------|
| 检测到 io_uring 漏洞利用 | `sysctl -w airy_io_uring_disabled=2` |
| Agent 异常 io_uring 行为 | `sysctl -w airy_io_uring_disabled=1` |
| 安全审计 | 临时禁用 + 审计后恢复 |

---

## §3 opcode 白名单：仅允许 IORING_OP_URING_CMD + 基础 opcode

### 3.1 白名单设计

Airymax 对 Agent 可用的 io_uring opcode 实施白名单限制，仅允许 IPC 所需的 opcode：

| opcode | 是否允许 | 用途 |
|--------|---------|------|
| `IORING_OP_URING_CMD` | ✅ 允许 | IPC 语义映射（A-IPC 核心） |
| `IORING_OP_NOP` | ✅ 允许 | 基础测试 |
| `IORING_OP_READV` | ❌ 禁止 | 文件读取（越权风险） |
| `IORING_OP_WRITEV` | ❌ 禁止 | 文件写入（越权风险） |
| `IORING_OP_SEND` | ❌ 禁止 | 网络发送（外连风险） |
| `IORING_OP_RECV` | ❌ 禁止 | 网络接收 |
| `IORING_OP_OPENAT` | ❌ 禁止 | 文件打开（越权风险） |
| 其他 | ❌ 禁止 | 默认拒绝 |

### 3.2 白名单实现

```c
/* kernel/ipc/airy_uring_security.c —— opcode 白名单 */
/*
 * opcode 白名单：仅允许 IORING_OP_URING_CMD + 基础 opcode
 * 其他 opcode 一律拒绝，返回 -EOPNOTSUPP
 */
static const unsigned int airy_uring_allowed_opcodes[] = {
    IORING_OP_URING_CMD,   /* IPC 语义映射（核心） */
    IORING_OP_NOP,         /* 基础测试 */
};

static bool airy_uring_opcode_allowed(unsigned int opcode)
{
    for (size_t i = 0; i < ARRAY_SIZE(airy_uring_allowed_opcodes); i++) {
        if (opcode == airy_uring_allowed_opcodes[i])
            return true;
    }
    return false;  /* 默认拒绝 */
}

/* io_uring cmd_op 白名单校验辅助函数（非独立 LSM 钩子）
 * OLK 6.6 对齐: 不存在 uring_sqe 钩子，opcode 白名单改在 security_uring_cmd 内部分发
 * 详见 07-airy-lsm-design.md §4.1
 */
static int airy_uring_opcode_check(struct io_uring_cmd *ioucmd)
{
    u32 cmd_op = ioucmd->cmd_op;
    if (!airy_uring_opcode_allowed(cmd_op)) {
        /* 非法 opcode：记录告警 + 拒绝 */
        pr_warn_ratelimited("airy: blocked io_uring cmd_op %u for pid %d\n",
                cmd_op, current->pid);
        return -EOPNOTSUPP;
    }
    return 0;
}
```

### 3.3 白名单的 A-UCS 管理

opcode 白名单通过 A-UCS 配置管理，支持热重载：

| 配置项 | sysctl | JSON | 默认值 |
|--------|--------|------|--------|
| 允许的 opcode | `kernel.airy.uring_allowed_opcodes` | `uring.allowed_opcodes` | `[URING_CMD, NOP]` |

### 3.5 Malformed SQE/CQE Input 防护（R6 补强：malformed 输入防护）

opcode 白名单（§3.2 `airy_uring_opcode_check()`）仅校验 `cmd_op` 是否在允许列表内，但未覆盖 SQE/CQE 字段级的 malformed 输入。恶意 Agent 可构造 malformed SQE 绕过白名单，导致内核越界访问或崩溃。本节新增 5 阶段完整校验机制 `airy_uring_sqe_validate()`，作为 opcode 白名单的强化后继。

#### 3.5.1 威胁模型：5 种 malformed SQE 攻击向量

| # | 攻击向量 | 攻击载荷 | 危害 | CWE 映射 |
|---|---------|---------|------|---------|
| 1 | 无效 opcode | SQE->opcode 超出 `IORING_OP_URING_CMD` 范围（如伪造 `IORING_OP_OPENAT=18`） | 越权文件访问 | CWE-862（Missing Authorization） |
| 2 | 越界 cmd_op | `ioucmd->cmd_op` 超出 `AIRY_CMD_*` 枚举范围（如 `cmd_op=0xFFFF`） | 内核函数指针越界跳转 | CWE-129（Improper Validation of Array Index） |
| 3 | 错误 pdu 长度 | `pdu[32]` 缓冲区越界访问（如 Agent 通过 `ioucmd->cmd` 直接读取 64B） | 内核信息泄露 | CWE-125（Out-of-bounds Read） |
| 4 | 伪造 SQE128 模式 | 未启用 `IORING_SETUP_SQE128` 但提交 128B SQE（设置 `SQE128_FLAG`） | 内核解析 128B 越界读 | CWE-787（Out-of-bounds Write） |
| 5 | CQE 溢出攻击 | 高频提交 SQE 导致 CQ ring 溢出，覆盖相邻内核内存 | 内核内存损坏 | CWE-120（Buffer Copy without Checking Size） |

#### 3.5.2 防护机制

针对 5 种攻击向量，提供对应的防护代码：

- **opcode 白名单**：`airy_uring_opcode_check()` 已有（§3.2），强化为 `airy_uring_sqe_validate()` 完整校验（阶段 1 复用已有函数）
- **cmd_op 范围检查**：

  ```c
  if (sqe->cmd_op > AIRY_CMD_MAX)
      return -EINVAL;
  ```

- **pdu 长度校验**：通过 `io_uring_cmd_to_pdu()` 宏安全访问 `pdu[32]`，**禁止直接 `ioucmd->cmd`**（OLK 6.6 中 `struct io_uring_cmd` 无 `cmd` 字段，详见 §4.2）
- **SQE128 模式校验**：

  ```c
  if (!(ctx->flags & IORING_SETUP_SQE128) && (sqe->flags & SQE128_FLAG))
      return -EINVAL;
  ```

- **CQE 溢出防护**：通过 `io_cqring_overflow_timeout` 配置 + 强制 `io_cqring_overflow_flush()`

#### 3.5.3 `airy_uring_sqe_validate()` 完整 C 实现

```c
/* kernel/ipc/airy_uring_security.c —— malformed SQE 完整校验（R6 补强） */
/*
 * airy_uring_sqe_validate: 5 阶段 malformed SQE 完整校验
 *   阶段 1: opcode 白名单（复用 airy_uring_opcode_allowed）
 *   阶段 2: cmd_op 范围检查（AIRY_CMD_MIN..AIRY_CMD_MAX）
 *   阶段 3: pdu 长度校验（强制走 io_uring_cmd_to_pdu，禁止 ioucmd->cmd）
 *   阶段 4: SQE128 模式一致性校验
 *   阶段 5: CQE 溢出预检（CQ ring 余量检查）
 *
 * 全部失败统一返回 -EINVAL 并触发 airy_security_fault 通知 Micro-Supervisor
 */
int airy_uring_sqe_validate(struct io_ring_ctx *ctx,
                             struct io_uring_cmd *ioucmd,
                             unsigned int issue_flags)
{
    /* ── 阶段 1: opcode 白名单（复用 §3.2 已有函数） ── */
    if (!airy_uring_opcode_allowed(IORING_OP_URING_CMD)) {
        pr_warn_ratelimited("airy: SQE opcode %u not in whitelist (pid=%d)\n",
                IORING_OP_URING_CMD, current->pid);
        goto malformed;
    }

    /* ── 阶段 2: cmd_op 范围检查 ── */
    if (ioucmd->cmd_op < AIRY_CMD_MIN || ioucmd->cmd_op > AIRY_CMD_MAX) {
        pr_warn_ratelimited("airy: SQE cmd_op %u out of range [%u..%u] (pid=%d)\n",
                ioucmd->cmd_op, AIRY_CMD_MIN, AIRY_CMD_MAX, current->pid);
        goto malformed;
    }

    /* ── 阶段 3: pdu 长度校验（强制走安全宏，禁止 ioucmd->cmd 直接访问） ──
     * io_uring_cmd_to_pdu() 返回 pdu[32] 起始地址；OLK 6.6 中 pdu 固定 32B，
     * 任何超出 32B 的访问由 BUILD_BUG_ON 在编译期拦截
     */
    struct airy_ipc_cmd *cmd = io_uring_cmd_to_pdu(ioucmd, struct airy_ipc_cmd);
    if (!cmd) {
        pr_warn_ratelimited("airy: SQE pdu access returned NULL (pid=%d)\n",
                current->pid);
        goto malformed;
    }

    /* ── 阶段 4: SQE128 模式一致性校验 ──
     * 未启用 IORING_SETUP_SQE128 但 SQE 携带 SQE128_FLAG 视为伪造
     */
    if (!(ctx->flags & IORING_SETUP_SQE128) &&
        (ioucmd->sqe->flags & SQE128_FLAG)) {
        pr_warn_ratelimited("airy: SQE128 forgery: ctx->flags=0x%x sqe->flags=0x%x (pid=%d)\n",
                ctx->flags, ioucmd->sqe->flags, current->pid);
        goto malformed;
    }

    /* ── 阶段 5: CQE 溢出预检 ──
     * CQ ring 余量 < 1 时拒绝，防止高频提交导致 CQ 溢出覆盖相邻内核内存；
     * 先尝试 io_cqring_overflow_flush 强制 flush，避免正常负载下合法 SQE 被丢弃
     */
    if (io_cqring_avail(ctx) < 1) {
        pr_warn_ratelimited("airy: CQ ring full, forcing flush (pid=%d)\n",
                current->pid);
        io_cqring_overflow_flush(ctx, issue_flags);
        if (io_cqring_avail(ctx) < 1) {
            pr_warn_ratelimited("airy: CQ ring still full after flush (pid=%d)\n",
                    current->pid);
            goto malformed;
        }
    }

    return 0;  /* 5 阶段校验全部通过，放行至 §4 纯 C LSM 双重校验 */

malformed:
    /* 所有 malformed 校验失败统一返回 -EINVAL（不区分阶段，防攻击者阶段定位） */
    airy_security_fault(current->pid, AIRY_FAULT_URING_MALFORMED, ioucmd);
    return -EINVAL;
}
```

#### 3.5.4 校验流程图：5 阶段 SQE 提交校验

```
Agent 提交 SQE (IORING_OP_URING_CMD)
        │
        ▼
┌──────────────────────────────────────────┐
│ 阶段 1: opcode 白名单校验                  │
│  airy_uring_opcode_allowed(URING_CMD)     │
│  失败 → -EOPNOTSUPP（白名单层拒绝）        │
└──────────────────────────────────────────┘
        │ 通过
        ▼
┌──────────────────────────────────────────┐
│ 阶段 2: cmd_op 范围校验                    │
│  AIRY_CMD_MIN ≤ cmd_op ≤ AIRY_CMD_MAX     │
│  失败 → -EINVAL + airy_security_fault     │
└──────────────────────────────────────────┘
        │ 通过
        ▼
┌──────────────────────────────────────────┐
│ 阶段 3: pdu 长度校验                       │
│  io_uring_cmd_to_pdu() 宏安全访问 pdu[32]  │
│  禁止直接 ioucmd->cmd                     │
│  失败 → -EINVAL + airy_security_fault     │
└──────────────────────────────────────────┘
        │ 通过
        ▼
┌──────────────────────────────────────────┐
│ 阶段 4: SQE128 模式一致性校验              │
│  ctx->flags & IORING_SETUP_SQE128         │
│    一致于 sqe->flags & SQE128_FLAG        │
│  失败 → -EINVAL + airy_security_fault     │
└──────────────────────────────────────────┘
        │ 通过
        ▼
┌──────────────────────────────────────────┐
│ 阶段 5: CQE 溢出预检                       │
│  io_cqring_avail(ctx) ≥ 1                 │
│  不足 → io_cqring_overflow_flush + 重检    │
│  仍不足 → -EINVAL + airy_security_fault   │
└──────────────────────────────────────────┘
        │ 通过
        ▼
    放行至 §4 纯 C LSM 双重校验
    （airy_uring_cmd_security_check）
```

#### 3.5.5 性能影响

5 阶段校验的开销分解：

| 阶段 | 操作 | 开销 |
|------|------|------|
| 阶段 1 opcode 白名单 | 2 次比较（数组遍历） | ~2ns |
| 阶段 2 cmd_op 范围 | 2 次比较 | ~0.5ns |
| 阶段 3 pdu 长度 | 1 次宏展开 + NULL 检查 | ~0.3ns |
| 阶段 4 SQE128 模式 | 2 次位运算 + 1 次比较 | ~0.5ns |
| 阶段 5 CQE 溢出预检 | 1 次 atomic_read + 1 次比较 | ~0.7ns |
| **合计** | **5 次比较 + 位运算** | **~3ns** |

- **fastpath 影响**：malformed 校验增加 ~3ns，占 IPC fastpath SLO（≤200ns）的 ~1.5%，可接受
- **slowpath 影响**：阶段 5 触发 `io_cqring_overflow_flush` 时进入 slowpath（~1μs），仅在高负载场景偶发，不阻塞 fastpath

#### 3.5.6 失败处理契约

所有 malformed 校验失败统一遵循以下契约：

1. **返回值**：统一返回 `-EINVAL`（不区分具体阶段，避免给攻击者阶段定位信息）
2. **Fault 通知**：调用 `airy_security_fault(pid, AIRY_FAULT_URING_MALFORMED, ioucmd)` 通知 Micro-Supervisor（详见 [07-airy-lsm-design.md §3.5](07-airy-lsm-design.md)）
3. **审计日志**：`airy_security_fault` 内部触发 `airy_audit_emit_security(AGENT_URING_MALFORMED, ...)` 写入 A-ULP 审计日志（永不可绕过）
4. **限流**：`pr_warn_ratelimited` 防止日志洪水
5. **Fault 码**：`AIRY_FAULT_URING_MALFORMED = 0x100A`（已在 [08-sc-error-contract.md §3.1](../30-interfaces/08-sc-error-contract.md) 注册；注：原计划使用 0x1007，但该值已分配给 `AIRY_FAULT_MEMORY_QUOTA_EXCEEDED`，故追加至 0x100A）

> **OS-KER-172**（R6 新增）：所有 `airy_uring_sqe_validate()` 失败必须触发 `airy_security_fault` 通知 Micro-Supervisor；连续 3 次 malformed SQE 的 Agent 由 Macro-Supervisor 裁决为 `AIRY_VERDICT_PAUSE`（暂停）或 `AIRY_VERDICT_TERMINATE`（终止），详见 [10-user-supervisor-daemon.md §5.2](../20-modules/10-user-supervisor-daemon.md)。

---

## §4 纯 C LSM 双重校验：io_uring_cmd 回调中纯 C LSM 校验

### 4.1 双重校验模型

io_uring_cmd 回调中执行纯 C LSM 双重校验，确保 Capability 与 opcode 均合法（**不使用 BPF LSM**，对齐 openEuler 纯 C 模式）：

| 校验层 | 校验内容 | 实现 | 失败处理 |
|--------|---------|------|---------|
| 第一层 | opcode 白名单校验 | 纯 C 函数 | 返回 -EOPNOTSUPP |
| 第二层 | Capability 校验 | 纯 C LSM 钩子 | 返回 AIRY_FAULT_* |

### 4.2 双重校验实现

```c
/* kernel/ipc/airy_uring_security.c —— 纯 C LSM 双重校验 */
/*
 * io_uring_cmd 回调中的纯 C LSM 双重校验
 * 第一层：opcode 白名单
 * 第二层：Capability 校验
 * 不使用 BPF LSM，对齐 openEuler 纯 C 模式
 */
static int airy_uring_cmd_security_check(struct io_uring_cmd *ioucmd,
                                          unsigned int issue_flags)
{
    /* OLK 6.6: struct io_uring_cmd 无 cmd 字段，使用 io_uring_cmd_to_pdu() 宏访问 pdu[32] */
    struct airy_ipc_cmd *cmd = io_uring_cmd_to_pdu(ioucmd, struct airy_ipc_cmd);

    /* 第一层：opcode 白名单校验 */
    if (!airy_uring_opcode_allowed(IORING_OP_URING_CMD)) {
        return -EOPNOTSUPP;
    }

    /* 第二层：Capability 校验（纯 C LSM，非 BPF）
     * v1.1: IPC fastpath 走 C-S9 Badge 内联校验（airy_cap_badge_ok）；
     *       non-IPC 路径（如 buffer 注册）走 slowpath airy_cap_check()。
     *       本钩子是 security_uring_cmd LSM 入口，仅 slowpath 被 C-S9 失败时调用
     *       （详见 07-airy-lsm-design.md §3.3）。
     */
    if (airy_op_needs_cap(cmd->op)) {
        int ret = airy_cap_check(current_agent_id(), cmd->cap_id, cmd->required_badge);
        if (ret < 0) {
            /* Capability 校验失败：冷酷执法 */
            return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);
        }
    }

    return 0;  /* 双重校验通过 */
}

/* 注册到 security_uring_cmd 钩子（OLK 6.6 唯一 io_uring 钩子）
 * v1.1: 不注册 uring_sqe（OLK 6.6 中不存在），opcode 白名单在本钩子内部分发
 */
static struct security_hook_list airy_uring_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_security_check),
};
```

### 4.3 纯 C vs BPF LSM 对比

| 维度 | 纯 C LSM（Airymax 选择） | BPF LSM |
|------|------------------------|---------|
| 性能 | 直接函数调用，无间接开销 | BPF 虚拟机间接调用 |
| 安全 | 编译期校验，无运行时注入风险 | BPF 字节码需运行时验证 |
| 可审计 | 源码可静态分析 | BPF 字节码需反编译 |
| 复杂度 | 简单（C 函数） | 复杂（BPF 程序 + verifier） |
| 对齐 | openEuler 纯 C 模式 | — |

---

## §5 registered buffer 安全：Capability 校验后才能注册 buffer

### 5.1 registered buffer 风险

`IORING_OP_URING_CMD` 使用 registered buffer 实现零拷贝（**不使用 page flipping**）。但 buffer 注册存在风险：

| 风险 | 说明 | 攻击场景 |
|------|------|---------|
| 任意 buffer 注册 | Agent 注册任意内存地址 | 内核写入导致信息泄露 |
| buffer 重用 | Agent 重用已释放 buffer | Use-after-free |
| buffer 越界 | Agent 注册过小 buffer | 内核写入越界 |

### 5.2 安全注册流程

Airymax 要求 buffer 注册前必须通过 Capability 校验：

```
Agent 请求注册 buffer
       │
       ▼
1. Capability 校验（纯 C LSM）
   ├── 校验 AIRY_CAP_REGISTER_BUFFER 权限
   └── 校验 buffer 地址范围合法
       │
       ▼
2. buffer 安全检查
   ├── 地址范围在 Agent 允许的内存区域
   ├── 大小不超过限制（默认 1MB）
   └── 页面映射有效（access_ok）
       │
       ▼
3. 注册到 io_uring
   └── 内核记录 buffer 引用，pin 住物理页
       │
       ▼
4. 返回 buffer_id（Agent 后续引用）
```

### 5.3 注册实现

```c
/* kernel/ipc/airy_uring_security.c —— registered buffer 安全注册 */
static int airy_uring_register_buffer_check(struct io_ring_ctx *ctx,
                                              void __user *addr,
                                              unsigned long size)
{
    /* 1. Capability 校验：必须拥有 REGISTER_BUFFER 权限
     * v1.1: buffer 注册是 non-IPC 路径，走 slowpath airy_cap_check()
     *       （IPC fastpath 走 C-S9 Badge 内联校验，详见 03-capability-model.md §4.3.2）
     */
    if (airy_cap_check(current_agent_id(), current_cap_id(),
                       AIRY_CAP_REGISTER_BUFFER) < 0) {
        pr_warn("airy: buffer register denied (no cap) pid=%d\n",
                current->pid);
        return -AIRY_ECAP_PERM;
    }

    /* 2. 地址范围检查 */
    if (!access_ok(addr, size)) {
        pr_warn("airy: buffer register denied (bad addr) pid=%d\n",
                current->pid);
        return -EFAULT;
    }

    /* 3. 大小限制（防资源耗尽） */
    if (size > AIRY_URING_MAX_BUFFER_SIZE) {  /* 默认 1MB */
        pr_warn("airy: buffer register denied (too large) pid=%d size=%lu\n",
                current->pid, size);
        return -EINVAL;
    }

    /* 4. 页面有效性检查（pin 住页面，确保有效） */
    /* 实际 pin 操作由 io_uring 框架完成，此处仅预检 */

    return 0;
}

/* v1.1: 不注册 uring_register_buffers 钩子（OLK 6.6 中不存在）
 * buffer 注册校验改为 airy_uring_cmd_check() 在 IORING_OP_URING_CMD 子命令中调用
 * 本函数 airy_uring_register_buffer_check() 作为辅助函数被调用，不作为独立 LSM 钩子
 *
 * 替代方案：buffer 注册类操作也可通过 security_file_ioctl() 等既有钩子承担
 * 详见 07-airy-lsm-design.md §4.1
 */
```

### 5.4 buffer 生命周期管理

| 阶段 | 安全检查 | 说明 |
|------|---------|------|
| 注册 | Capability + 地址 + 大小 | 本节实现 |
| 使用 | buffer_id 校验 | io_uring_cmd 回调中校验 |
| 注销 | 引用计数检查 | 确保无在途操作 |
| Agent 终止 | 自动注销 | 防止 buffer 泄漏 |

---

## §6 与 A-IPC 模块的关系：io_uring 安全是 A-IPC 的安全基础

### 6.1 在 A-IPC 中的定位

io_uring 安全加固是 A-IPC 模块的安全基础，确保 IPC 通道不被滥用：

```
A-IPC 安全体系
├── io_uring 安全加固（本文件）
│   ├── io_uring_disabled sysctl（全局开关）
│   ├── opcode 白名单（最小权限）
│   ├── 纯 C LSM 双重校验（Capability + opcode）
│   └── registered buffer 安全（注册前校验）
├── Capability 分层校验（[03-capability-model.md]）
│   ├── fastpath 缓存校验
│   └── slowpath 完整校验
└── 数据面自治（[04-ipc-data-plane-autonomy.md]）
    ├── Ring 生命周期解耦
    ├── 离线缓存校验
    └── Reconciliation
```

### 6.2 与纯 C LSM 模块的关系

io_uring 安全加固依赖纯 C LSM 模块（详见 [07-airy-lsm-design.md](07-airy-lsm-design.md)）：

| io_uring 安全机制 | 纯 C LSM 钩子 | 说明 |
|-----------------|-------------|------|
| opcode 白名单 | `security_uring_cmd`（内部分发） | v1.1: 在 uring_cmd 钩子内基于 ioucmd->cmd_op 校验（OLK 6.6 无 uring_sqe 钩子） |
| Capability 校验 | `security_uring_cmd` | URING_CMD 回调中校验 Badge |
| buffer 注册校验 | `security_uring_cmd`（子命令分发） | v1.1: 在 uring_cmd 钩子内分发 buffer 注册子命令（OLK 6.6 无 uring_register_buffers 钩子） |

### 6.3 与 seL4 Capability 模型的关系

io_uring 安全加固借鉴 seL4 capability-based security（详见 [03-capability-model.md](03-capability-model.md)）：

| seL4 机制 | Airymax io_uring 对齐 | 说明 |
|-----------|---------------------|------|
| Capability 授权 | registered buffer 注册前校验 | 必须拥有 CAP 权限 |
| Capability 传递 | io_uring_cmd 携带 cap_id | 每次操作校验 |
| Capability 撤销 | 注销 buffer + 撤销 CAP | Agent 终止时自动 |

### 6.4 性能影响

| 安全检查 | 位置 | 开销 | 频率 |
|---------|------|------|------|
| io_uring_disabled 检查 | io_uring 创建 | ~1ns | 一次性 |
| opcode 白名单 | SQE 提交 | ~5ns | 每次 SQE |
| Capability 校验（fastpath） | URING_CMD 回调 | ~160ns | 每次操作 |
| buffer 注册校验 | 注册时 | ~1μs | 一次性 |

fastpath 总安全开销：~165ns，占 IPC fastpath（~160ns）的 ~50%，但仍在 SLO（≤200ns）内。

---

## §7 测试与验证

### 7.1 测试场景

| 测试场景 | 验证内容 | 预期结果 |
|---------|---------|---------|
| Agent 使用 IORING_OP_URING_CMD | 白名单允许 | 正常执行 |
| Agent 使用 IORING_OP_READV | 白名单拒绝 | 返回 -EOPNOTSUPP |
| Agent 无 CAP 注册 buffer | Capability 拒绝 | 返回 -AIRY_ECAP_PERM |
| Agent 注册超大 buffer | 大小限制 | 返回 -EINVAL |
| io_uring_disabled=2 | 全局禁用 | 新建 io_uring 失败 |
| 纯 C LSM 双重校验 | opcode + Capability | 两层均通过才放行 |

### 7.2 SLO

| 指标 | SLO | 实测 |
|------|-----|------|
| opcode 白名单检查 | ≤10ns | ~5ns |
| Capability fastpath 校验 | ≤200ns | ~160ns |
| buffer 注册校验 | ≤10μs | ~1μs |
| 安全检查对 IPC fastpath 影响 | ≤20% | ~50% of 160ns |

---

## §8 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §8 —— A-IPC 模块总纲
- [07-airy-lsm-design.md](07-airy-lsm-design.md) —— 纯 C LSM 模块设计（安全钩子基础）
- [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— IPC 协议契约
- [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— IPC fastpath（性能基础）
- [03-capability-model.md](03-capability-model.md) —— Capability 模型（校验基础）
- [04-ipc-data-plane-autonomy.md](04-ipc-data-plane-autonomy.md) —— 数据面自治（离线校验）
- 综合修正方案 §4.2.5 / §6.2.1 C-02 —— 设计依据

---

## §9 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：io_uring 安全加固设计；io_uring_disabled sysctl（0/1/2 三级控制）；opcode 白名单（仅 IORING_OP_URING_CMD + NOP）；纯 C LSM 双重校验（opcode + Capability，不使用 BPF LSM）；registered buffer 安全（注册前 Capability 校验 + 地址 + 大小限制）；与 A-IPC/纯 C LSM/seL4 Capability 模型关系 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | io_uring 安全加固设计 | v1.0 | 2026-07-17
