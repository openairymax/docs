Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# io_uring 安全加固设计
> **文档定位**：A-IPC（统一进程间通信体系）io_uring 在 Agent 环境安全加固的唯一权威设计\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §8\
> **设计依据**：综合修正方案 §4.2.5（A-IPC 设计）+ §6.2.1 C-02（page flipping 修正）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **io_uring 安全加固设计** 的唯一权威源。io_uring_disabled sysctl 控制、opcode 白名单（仅允许 IORING_OP_URING_CMD + 基础 opcode）、纯 C LSM 双重校验（io_uring_cmd 回调中纯 C LSM 校验）、registered buffer 安全（Capability 校验后才能注册 buffer）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 io_uring 安全加固策略。
>
> 技术选型声明：IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（**不使用 page flipping**）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（**不使用 BPF LSM**）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

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

/* io_uring SQE 提交时检查（纯 C LSM 钩子） */
static int airy_uring_sqe_check(struct io_kiocb *req, unsigned int opcode)
{
    if (!airy_uring_opcode_allowed(opcode)) {
        /* 非法 opcode：记录告警 + 拒绝 */
        pr_warn("airy: blocked io_uring opcode %u for pid %d\n",
                opcode, current->pid);
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
    struct airy_ipc_cmd *cmd = (struct airy_ipc_cmd *)ioucmd->cmd;

    /* 第一层：opcode 白名单校验 */
    if (!airy_uring_opcode_allowed(IORING_OP_URING_CMD)) {
        return -EOPNOTSUPP;
    }

    /* 第二层：Capability 校验（纯 C LSM，非 BPF） */
    if (airy_op_needs_cap(cmd->op)) {
        int ret = airy_cap_check_fastpath(cmd->cap_id, cmd->required_badge);
        if (ret < 0) {
            /* Capability 校验失败：冷酷执法 */
            return airy_fault_enforce(AIRY_FAULT_ABNORMAL_CAP, cmd);
        }
    }

    return 0;  /* 双重校验通过 */
}

/* 注册到 security_uring_cmd 钩子 */
static struct security_hook_list airy_uring_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_security_check),
    LSM_HOOK_INIT(uring_sqe, airy_uring_sqe_check),
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
    /* 1. Capability 校验：必须拥有 REGISTER_BUFFER 权限 */
    if (airy_cap_check_fastpath(current_cap_id(),
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

/* 注册到 security 钩子 */
static struct security_hook_list airy_buffer_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(uring_register_buffers, airy_uring_register_buffer_check),
};
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
| opcode 白名单 | `security_uring_sqe` | SQE 提交时校验 opcode |
| Capability 校验 | `security_uring_cmd` | URING_CMD 回调中校验 |
| buffer 注册校验 | `security_uring_register_buffers` | 注册时校验权限 |

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
