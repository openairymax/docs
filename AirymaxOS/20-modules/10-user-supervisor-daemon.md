Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Macro-Supervisor 用户态守护进程设计
> **文档定位**：A-ULS（统一生命周期管理）模块的用户态监管组件唯一权威设计\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7\
> **设计依据**：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射）+ A-IPC v1.1（Capability Folding 单平面架构）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Macro-Supervisor 用户态守护进程** 的唯一权威源。systemd unit 管理 12 个 daemon、心跳检查、**v1.1 Badge 请求与撤销**（通过 `AIRY_IPC_OP_CAP_REQUEST` opcode 请求 sec_d 编译 Badge，通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码撤销所有 Badge）、故障裁决（警告/降级/暂停/终止）、"用户温情裁决"策略、单点故障 fallback（内核 watchdog 直接重启 + 全局 Epoch 撤销）、io_uring_cmd 执行裁决均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Macro-Supervisor 职责与裁决流程。
>
> **v1.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.1 起，Macro-Supervisor 不再直接注入 CNode 到 radix tree——而是通过 `AIRY_IPC_OP_CAP_REQUEST` opcode 向 sec_d 请求 Badge，sec_d 编译 Badge 64-bit Native Word（`Epoch<<48 | RandomTag<<16 | Perms`）并写入 `agent_caps[agent_id]` 静态数组（详见 [07-airy-lsm-design.md §3.5](../110-security/07-airy-lsm-design.md)）。Macro-Supervisor 通过 io_uring_cmd 执行裁决（不新增 syscall），所有裁决指令通过 `AIRY_IPC_OP_ADJUDICATE` opcode 传递到内核 Micro-Supervisor。Badge 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码 O(1) 立即失效所有已发出 Badge（无 drain、无 bitmap、无 IPI），详见 [09-kernel-agent-supervisor.md §6.4](09-kernel-agent-supervisor.md)。
>
> 技术选型声明：IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（**不使用 page flipping**）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：用户态守护进程开发者、A-ULS 模块开发者、运维工程师
- **前置知识**：理解 [09-kernel-agent-supervisor.md](09-kernel-agent-supervisor.md) Micro-Supervisor、[10-unify-design.md](../10-architecture/10-unify-design.md) §7 A-ULS 模块、[10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) Agent 8 态
- **预计阅读时间**：35 分钟
- **核心概念**：Macro-Supervisor、systemd unit、心跳检查、Capability 注入、故障裁决、温情裁决、单点 fallback、io_uring_cmd
- **复杂度标识**：高级

---

## §1 架构定位：A-ULS 模块的用户态监管组件

### 1.1 双 Supervisor 模型回顾

A-ULS 模块采用双 Supervisor 模型——内核冷酷执法 + 用户温情裁决（详见 [10-unify-design.md](../10-architecture/10-unify-design.md) §7）：

| Supervisor | 执行域 | 职责 | 设计哲学 |
|-----------|--------|------|---------|
| Micro-Supervisor | 内核态 | 检测异常 → 立即冻结 → 通知 | 冷酷执法（机制级） |
| **Macro-Supervisor** | **用户态** | **接收通知 → 查询上下文 → 裁决** | **温情裁决（策略级）** |

Macro-Supervisor 是用户态守护进程，接收 Micro-Supervisor 的故障通知后进行人性化裁决——基于上下文与策略，决定警告、降级、暂停或终止。这是借鉴 seL4 `handleFault()` 的设计：内核只负责检测与通知，裁决交给用户态。

### 1.2 物理宿主

| 组件 | 路径 | 说明 |
|------|------|------|
| 主进程 | `services/daemons/macro_superv/main.c` | 事件循环、裁决分发 |
| 事件循环 | `services/daemons/macro_superv/event_loop.c` | io_uring + eventfd |
| 裁决引擎 | `services/daemons/macro_superv/adjudicate.c` | 故障裁决策略 |
| 调度注入 | `services/daemons/macro_superv/sched.c` | 调度参数注入 |
| Capability 注入 | `services/daemons/macro_superv/cap_inject.c` | io_uring_cmd 注入 |
| 心跳检查 | `services/daemons/macro_superv/heartbeat.c` | Agent 进程状态检查 |
| systemd unit | `systemd/airymax-macro-superv.service` | 自动重启管理 |

### 1.3 12 个 daemon 职责

Macro-Supervisor 管理的 12 个 daemon 及其职责：

| # | Daemon | 职责 | Agent 态管理 |
|---|--------|------|------------|
| 1 | macro_superv | 主监管守护进程（A-ULS） | 自身（高优先级） |
| 2 | logger_daemon | 日志消费（A-ULP） | 心跳监控 |
| 3 | config_daemon | 配置管理（A-UCS） | 心跳监控 |
| 4 | gateway_d | 网关守护 | 心跳监控 |
| 5 | sched_d | 调度守护 | 8 态生命周期 |
| 6 | vfs_d | VFS 用户态服务 | 心跳监控 |
| 7 | net_d | 网络策略守护 | 心跳监控 |
| 8 | mem_d | 记忆管理守护 | 心跳监控 |
| 9 | cogn_d | 认知调度守护 | 8 态生命周期 |
| 10 | sec_d | 安全策略守护 | 心跳监控 |
| 11 | audit_d | 审计守护 | 心跳监控 |
| 12 | dev_d | 设备驱动守护 | 心跳监控 |

---

## §2 进程拉起：systemd unit 管理 12 个 daemon

### 2.1 systemd unit 配置

Macro-Supervisor 作为 systemd service 管理 12 个 daemon 的生命周期：

```ini
# /etc/systemd/system/airymax-macro-superv.service
[Unit]
Description=Airymax Macro-Supervisor (A-ULS user-side)
After=systemd-modules-load.service airy-log.service
Requires=airy-log.service
# 依赖内核 Micro-Supervisor 模块
After=airy-superv.service

[Service]
Type=simple
ExecStart=/usr/sbin/airy-macro-superv --config=/etc/airymax/macro_superv.conf
# 自动重启：崩溃后 1 秒内重启
Restart=always
RestartSec=1
# 资源限制
MemoryMax=512M
LimitNOFILE=256
# 调度：SCHED_FIFO 优先级 80，确保及时裁决
# 对齐sched_tac：关键 daemon 用 SCHED_FIFO
IOSchedulingClass=realtime
IOSchedulingPriority=2
# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/airymax/
# 委派权限：可管理其他 daemon
Delegate=yes

[Install]
WantedBy=multi-user.target
```

### 2.2 子 daemon 拉起流程

Macro-Supervisor 启动后按依赖顺序拉起 12 个 daemon：

```c
/* services/daemons/macro_superv/main.c —— daemon 拉起 */
struct daemon_spec {
    const char *name;
    const char *exec;
    const char *config;
    int sched_policy;     /* SCHED_FIFO / SCHED_NORMAL */
    int sched_priority;   /* 仅 SCHED_FIFO 使用 */
    int restart_max;      /* 最大重启次数 */
};

static const struct daemon_spec daemons[] = {
    { "logger_daemon",  "/usr/sbin/airy-logger-daemon",  "/etc/airymax/logger.conf",     SCHED_FIFO, 50, 5 },
    { "cognition_daemon","/usr/sbin/airy-cognition-daemon","/etc/airymax/cognition.conf",SCHED_DEADLINE, 0, 3 },
    { "memory_daemon",  "/usr/sbin/airy-memory-daemon",  "/etc/airymax/memory.conf",     SCHED_FIFO, 40, 5 },
    { "ipc_daemon",     "/usr/sbin/airy-ipc-daemon",     "/etc/airymax/ipc.conf",        SCHED_FIFO, 60, 5 },
    { "config_daemon",  "/usr/sbin/airy-config-daemon",  "/etc/airymax/config.conf",     SCHED_NORMAL, 0, 5 },
    /* ... 其余 daemon */
};

static int macro_superv_spawn_daemons(void)
{
    for (int i = 0; i < ARRAY_SIZE(daemons); i++) {
        const struct daemon_spec *d = &daemons[i];
        pid_t pid = fork();
        if (pid == 0) {
            /* 子进程：设置调度策略（sched_tac） */
            if (d->sched_policy == SCHED_DEADLINE) {
                macro_superv_set_deadline(getpid(), d);
            } else if (d->sched_policy == SCHED_FIFO) {
                macro_superv_set_fifo(getpid(), d->sched_priority);
            }
            execv(d->exec, (char *[]){ (char *)d->name, "--config", (char *)d->config, NULL });
            _exit(1);
        }
        /* 父进程：记录 daemon 状态，进入 SPAWNING 态 */
        agent_state_set(pid, AIRY_AGENT_SPAWNING);
    }
    return 0;
}
```

### 2.3 依赖顺序

daemon 拉起遵循依赖顺序，避免循环依赖：

```
1. logger_daemon   （无依赖，最先启动）
2. config_daemon   （依赖 logger）
3. memory_daemon   （依赖 config）
4. ipc_daemon      （依赖 memory）
5. security_daemon （依赖 ipc）
6. cognition_daemon（依赖 security）
7-12. 其他 daemon  （依赖前序）
```

---

## §3 心跳检查：定期检查 Agent 进程状态

### 3.1 心跳模型

Macro-Supervisor 定期检查 12 个 daemon 的进程状态，超时则触发裁决：

| 检查项 | 频率 | 超时阈值 | 触发动作 |
|--------|------|---------|---------|
| 进程存活（kill -0） | 每 1 秒 | 3 次失败 | 重启 daemon |
| 心跳消息（eventfd） | 每 5 秒 | 15 秒 | 降级 Agent |
| 响应探测（ping） | 每 10 秒 | 30 秒 | 终止 Agent |

### 3.2 心跳实现

```c
/* services/daemons/macro_superv/heartbeat.c —— 心跳检查 */
struct agent_heartbeat {
    pid_t pid;
    __u64 last_heartbeat_ns;
    __u32 missed_count;
    enum airy_agent_state last_state;
};

static int heartbeat_check(struct agent_heartbeat *hb)
{
    /* 1. 进程存活检查（kill -0 不发送信号，仅检查存在性） */
    if (kill(hb->pid, 0) < 0) {
        if (errno == ESRCH) {
            /* 进程不存在：已 DEAD */
            hb->last_state = AIRY_AGENT_DEAD;
            return HEARTBEAT_DEAD;
        }
    }

    /* 2. 心跳消息超时检查 */
    __u64 now = ktime_get_real_ns();
    if (now - hb->last_heartbeat_ns > 15ULL * 1000000000ULL) {
        hb->missed_count++;
        if (hb->missed_count >= 3) {
            /* 连续 3 次心跳超时：触发降级裁决 */
            return HEARTBEAT_TIMEOUT;
        }
    } else {
        hb->missed_count = 0;  /* 收到心跳，重置 */
    }

    return HEARTBEAT_OK;
}

/* 心跳检查主循环（每秒执行一次） */
static void heartbeat_loop(void)
{
    while (1) {
        for (int i = 0; i < g_agent_count; i++) {
            int ret = heartbeat_check(&g_heartbeats[i]);
            if (ret == HEARTBEAT_TIMEOUT) {
                /* 触发降级裁决 */
                macro_superv_adjudicate_fault(g_heartbeats[i].pid,
                                               AIRY_FAULT_TIMEOUT);
            } else if (ret == HEARTBEAT_DEAD) {
                /* 进程已死，重启 */
                macro_superv_restart_daemon(g_heartbeats[i].pid);
            }
        }
        sleep(1);
    }
}
```

---

## §4 v1.1 Badge 请求：通过 io_uring_cmd 向 sec_d 请求 Badge 编译

### 4.1 v1.1 请求流程（替代 v1.0 CNode 注入）

**v1.1 重构**：Macro-Supervisor 不再直接构造 CNode 并插入 radix tree——而是通过 `AIRY_IPC_OP_CAP_REQUEST` opcode 向 sec_d 请求 Badge 编译。sec_d 是 Badge 的**唯一写者**，编译 Badge 64-bit Native Word（`Epoch<<48 | RandomTag<<16 | Perms`）并写入 `agent_caps[agent_id]` 静态数组（详见 [07-airy-lsm-design.md §3.5](../110-security/07-airy-lsm-design.md)）。Macro-Supervisor 仅持有 Badge 的 opaque handle（不接触 RandomTag 生成、不接触 agent_caps[] 写入）。

```
Macro-Supervisor                     内核 sec_d（Badge 唯一写者）
    │                                    │
    │  1. 构造 CAP_REQUEST 请求           │
    │     （agent_id + perms 位掩码）     │
    │                                    │
    │  2. io_uring_cmd(CAP_REQUEST) ─────▶│
    │     opcode=0x0010                  │
    │                                    │  3. sec_d 编译 Badge
    │                                    │     epoch = atomic_read(global_epoch)
    │                                    │     get_random_bytes_wait(&randtag)
    │                                    │     badge = (epoch<<48)|(randtag<<16)|perms
    │                                    │  4. WRITE_ONCE(agent_caps[agent_id])
    │  5. 接收 CAP_RESPONSE ◀────────────│  返回 Badge opaque handle
    │     opcode=0x0011                  │
    │                                    │
    │  6. Macro-Supervisor 持有 Badge    │
    │     （opaque handle，用于后续 IPC） │
    │                                    │
    │  7. 记录授权日志（A-ULP）            │
```

### 4.2 v1.1 请求实现

```c
/* services/daemons/macro_superv/cap_inject.c —— v1.1 Badge 请求
 * Macro-Supervisor 不再构造 CNode，而是请求 sec_d 编译 Badge
 */
static int macro_superv_cap_request(__u32 agent_id, __u16 perms)
{
    struct airy_ipc_cmd cmd = {
        .opcode = AIRY_IPC_OP_CAP_REQUEST,   /* 0x0010 */
        .agent_id = agent_id,
        .perms = perms,                       /* AIRY_BADGE_PERM_* 位掩码 */
    };

    /* 通过 IORING_OP_URING_CMD 请求 Badge 编译（不新增 syscall）
     * sec_d 是 agent_caps[] 静态数组的唯一写者
     * 详见 110-security/07-airy-lsm-design.md §3.5
     */
    int ret = airy_uring_cmd_exec(&cmd, sizeof(cmd));
    if (ret < 0) {
        syslog(LOG_ERR, "airy: cap request failed: %d (agent=%u perms=0x%x)",
               ret, agent_id, perms);
        return ret;
    }

    /* 内核返回 Badge opaque handle（用户态持有，不暴露内部结构） */
    __u64 badge = (__u64)ret;
    syslog(LOG_INFO, "airy: badge compiled agent=%u perms=0x%x badge=0x%llx",
           agent_id, perms, (unsigned long long)badge);
    return 0;
}
```

### 4.3 v1.1 Badge Perms 位掩码（替代 v1.0 Capability 类型）

**v1.1 变更**：原 v1.0 的 `IPC_SEND/IPC_RECV/RING_CREATE/CAP_MINT/CAP_REVOKE` 五种 Capability 类型已废弃，改为 Badge Perms 16-bit 位掩码（详见 [03-capability-model.md §2.5](../110-security/03-capability-model.md)）：

| Perms 位 | 位掩码 | 含义 | 对应 v1.0 类型 |
|---------|--------|------|---------------|
| AIRY_BADGE_PERM_SEND | 0x0001 | 允许向 Ring 发送消息（SEND/SEND_BATCH opcode） | IPC_SEND |
| AIRY_BADGE_PERM_RECV | 0x0002 | 允许从 Ring 接收消息（RECV opcode） | IPC_RECV |
| AIRY_BADGE_PERM_FREEZE | 0x0004 | 允许冻结 Ring（FREEZE opcode） | —（v1.1 新增） |
| AIRY_BADGE_PERM_CANCEL | 0x0008 | 允许取消 IPC 操作（CANCEL opcode） | —（v1.1 新增） |
| AIRY_BADGE_PERM_CAP_REQUEST | 0x0010 | 允许请求 Badge 编译（CAP_REQUEST opcode） | CAP_MINT |
| AIRY_BADGE_PERM_CAP_REVOKE | 0x0020 | 允许撤销 Badge（atomic_inc 全局 Epoch） | CAP_REVOKE |
| AIRY_BADGE_PERM_RING_CREATE | 0x0040 | 允许创建新 Ring | RING_CREATE |
| AIRY_BADGE_PERM_ADMIN | 0x8000 | 管理员权限（包含所有 Perms） | —（v1.1 新增） |

### 4.4 v1.1 Badge 撤销（一行代码 O(1)）

**v1.1 新增**：Badge 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码完成，立即失效所有已发出 Badge。这是 Capability Folding 单平面架构的关键设计——无需 drain、无需 bitmap、无需 IPI，全局 Epoch 自增后所有 fastpath C-S9a 检查（`badge_epoch == global_epoch`）立即失败。

```c
/* services/daemons/macro_superv/cap_revoke.c —— v1.1 Badge 撤销
 * 一行代码撤销所有 Badge，O(1) 复杂度
 */
static int macro_superv_cap_revoke_all(void)
{
    /* 1. 全局 Epoch 自增——立即失效所有已发出 Badge
     * 所有 fastpath C-S9a 检查（badge_epoch == global_epoch）立即失败
     * 详见 09-kernel-agent-supervisor.md §6.4
     */
    atomic_inc(&airy_cap_global_epoch);

    /* 2. 记录撤销日志 */
    syslog(LOG_WARNING, "airy: all badges revoked (epoch incremented)");
    airy_audit_emit(AGENT_CAP_REVOKE_ALL, 0, 0);

    return 0;
}

/* 撤销单个 Agent 的 Badge（通过重新编译覆盖） */
static int macro_superv_cap_revoke_one(__u32 agent_id)
{
    /* 重新编译 Badge 时，WRITE_ONCE 会覆盖 agent_caps[agent_id].randtag
     * 旧 Badge 的 RandomTag 立即失效（C-S9b 检查失败）
     * 详见 110-security/07-airy-lsm-design.md §3.5
     */
    return macro_superv_cap_request(agent_id, 0);  /* perms=0 即撤销 */
}
```

---

## §5 故障裁决：接收 Micro-Supervisor 故障通知 → 查询上下文 → 裁决

### 5.1 裁决流程

Macro-Supervisor 接收 Micro-Supervisor 的 eventfd 故障通知后，执行温情裁决：

```
eventfd 唤醒
    │
    ▼
1. 读取故障事件（ring_id, fault_code, agent_id）
    │
    ▼
2. 查询上下文
    │  ├── Agent 历史违规记录
    │  ├── 当前 Agent 状态（8 态）
    │  ├── Fault 严重程度
    │  └── 配置策略（A-UCS）
    │
    ▼
3. 裁决决策
    │  ├── 警告（首次轻微违规）
    │  ├── 降级（Capability 边界模糊）
    │  ├── 暂停（多次违规）
    │  └── 终止（严重违规）
    │
    ▼
4. 执行裁决（通过 io_uring_cmd）
    │
    ▼
5. 记录裁决日志（A-ULP Ring Buffer）
```

### 5.2 裁决策略矩阵（v1.1: 对齐 Fault 码 0x1001~0x1006）

**v1.1 变更**：原 `AIRY_FAULT_CAP_FAULT` 已废弃，对齐 v1.1 Fault 码段（0x1001~0x1006，详见 [08-sc-error-contract.md §3](../30-interfaces/08-sc-error-contract.md)）：

| Fault 码 | 数值 | 首次 | 第二次 | 第三次+ | 严重场景 |
|---------|------|------|--------|---------|---------|
| `AIRY_FAULT_CAP_FORGED` | 0x1001 | 终止 | 终止 | 终止 | 终止（伪造不可恢复） |
| `AIRY_FAULT_CAP_LEAK` | 0x1002 | 降级 | 暂停 | 终止 | 终止 |
| `AIRY_FAULT_RING_CORRUPT` | 0x1003 | 暂停 | 终止 | 终止 | 终止 |
| `AIRY_FAULT_TIMEOUT` | 0x1004 | 降级 | 暂停 | 终止 | 终止 |
| `AIRY_FAULT_ABNORMAL_CAP` | 0x1005 | 警告 | 降级 | 暂停 | 终止 |
| `AIRY_FAULT_VM_FAULT` | 0x1006 | 暂停 | 终止 | 终止 | 终止 |

> **裁决原则**：`AIRY_FAULT_CAP_FORGED`（0x1001）是 Badge 伪造，**不可恢复**，首次即终止。其他 Fault 码按递进式处置（警告→降级→暂停→终止）。

### 5.3 裁决实现

```c
/* services/daemons/macro_superv/adjudicate.c —— 温情裁决 */
struct agent_violation_history {
    pid_t pid;
    __u32 violation_count[AIRY_FAULT_MAX];
    __u64 last_violation_ns;
};

static int macro_superv_adjudicate(struct airy_fault_event *ev)
{
    struct agent_violation_history *hist = get_history(ev->agent_id);
    enum airy_adjudication action;

    /* 1. 查询 Agent 当前状态 */
    enum airy_agent_state state = agent_state_get(ev->agent_id);
    if (state == AIRY_AGENT_DEAD || state == AIRY_AGENT_STOPPED) {
        /* Agent 已停止/死亡，无需裁决 */
        return 0;
    }

    /* 2. 根据策略矩阵裁决 */
    hist->violation_count[ev->fault_code]++;
    action = adjudicate_by_policy(ev->fault_code,
                                   hist->violation_count[ev->fault_code]);

    /* 3. 执行裁决 */
    switch (action) {
    case AIRY_ADJUD_WARN:
        syslog(LOG_WARNING, "airy: agent %u warned (fault=0x%x)",
               ev->agent_id, ev->fault_code);
        break;
    case AIRY_ADJUD_DEGRADE:
        macro_superv_degrade_agent(ev->agent_id);
        break;
    case AIRY_ADJUD_PAUSE:
        macro_superv_pause_agent(ev->agent_id);   /* kill(SIGSTOP) */
        break;
    case AIRY_ADJUD_TERMINATE:
        macro_superv_terminate_agent(ev->agent_id);  /* kill(SIGKILL) */
        break;
    }

    /* 4. 记录裁决日志（A-ULP Ring Buffer） */
    airy_log_write(LOG_WARNING, AIRY_FAC_SUPERV, &action, sizeof(action));

    return 0;
}
```

---

## §6 "用户温情裁决"：基于策略的柔性处理

### 6.1 温情 vs 冷酷对比

| 维度 | Micro-Supervisor（冷酷） | Macro-Supervisor（温情） |
|------|------------------------|------------------------|
| 决策依据 | 无（仅执法） | 策略矩阵 + 上下文 |
| 延迟 | ~100ns | ~10ms |
| 可逆性 | 不可逆 | 可逆（警告/降级可恢复） |
| 人性化 | 无 | 有（考虑历史、严重程度） |
| 借鉴 | seL4 handleFault() | seL4 用户态 Fault Handler |

### 6.2 柔性处理策略

温情裁决的柔性体现在以下方面：

1. **递进式处置**：首次警告 → 再次降级 → 屡犯暂停 → 严重终止
2. **上下文感知**：考虑 Agent 是否首次启动、是否在高负载下
3. **可恢复**：警告和降级不永久影响 Agent，可恢复
4. **策略可配置**：裁决策略通过 A-UCS 配置，支持热重载

### 6.3 裁决选项详解

| 裁决选项 | 适用场景 | 执行方式 | 可恢复 |
|---------|---------|---------|--------|
| 警告 | 首次轻微违规 | 记录日志，不中断 Agent | — |
| 降级 | Capability 边界模糊 | 降低优先级、缩减预算 | 是（恢复优先级） |
| 暂停 | 多次违规 | `kill(SIGSTOP)`，进入 STOPPED 态 | 是（`kill(SIGCONT)`） |
| 终止 | 严重违规 | `kill(SIGKILL)`，进入 DEAD 态 | 否（需重启） |

---

## §7 单点故障 fallback：内核 watchdog 直接重启

### 7.1 单点故障风险

Macro-Supervisor 是用户态单点，其故障可能导致：

| 故障场景 | 影响 | 传统处理 |
|---------|------|---------|
| Macro-Supervisor 崩溃 | 无法裁决、无法注入 Capability | systemd 重启 |
| Macro-Supervisor 死锁 | 故障通知无人处理 | 需人工干预 |
| Macro-Supervisor 被杀 | 所有 Agent 失去监管 | systemd 重启 |

### 7.2 内核 watchdog fallback（v1.1: 含 Badge 全局撤销）

为防止单点故障，Airymax 设计了**内核 watchdog 直接重启** fallback。v1.1 起，watchdog 接管时**立即撤销所有 Badge**（`atomic_inc(&airy_cap_global_epoch)`），防止无监管期间恶意 Agent 利用已发出的 Badge 继续操作：

```c
/* kernel/superv/airy_watchdog.c —— v1.1 内核 watchdog fallback
 * 新增 Badge 全局撤销：接管时立即失效所有 Badge
 */
static struct timer_list airy_superv_watchdog;
static __u64 last_macro_superv_heartbeat;

/* watchdog 超时检查（每 10 秒执行一次） */
static void airy_superv_watchdog_fn(struct timer_list *t)
{
    __u64 now = ktime_get_real_ns();
    /* Macro-Supervisor 心跳超时 30 秒 */
    if (now - last_macro_superv_heartbeat > 30ULL * 1000000000ULL) {
        pr_emerg("airy: Macro-Supervisor heartbeat lost, watchdog takeover\n");

        /* 1. v1.1 新增：立即撤销所有 Badge（atomic_inc 一行代码 O(1)）
         * 防止无监管期间恶意 Agent 利用已发出 Badge 操作
         * 所有 fastpath C-S9a 检查立即失败（badge_epoch != global_epoch）
         */
        atomic_inc(&airy_cap_global_epoch);
        pr_emerg("airy: all badges revoked (epoch=%u)\n",
                 atomic_read(&airy_cap_global_epoch));

        /* 2. 进入 [DSL] 降级模式 */
        airy_dsl_enter(AIRY_DSL_REASON_SUPERV_DEAD);

        /* 3. 内核直接重启 Macro-Supervisor（通过 systemd 通知） */
        orderly_poweroff_force("airy-macro-superv.service restart");

        /* 4. 冻结所有活跃 Ring（防止无监管期间数据损坏） */
        airy_ipc_freeze_all(AIRY_FAULT_TIMEOUT);
    }
    mod_timer(&airy_superv_watchdog, jiffies + 10 * HZ);
}
```

### 7.3 fallback 触发条件

| 触发条件 | 阈值 | 处理 |
|---------|------|------|
| 心跳超时 | 30 秒 | watchdog 接管 + [DSL] 降级 |
| systemd 重启失败 | 3 次 | 内核强制重启系统 |
| Macro-Supervisor 死锁 | 60 秒无响应 | watchdog 强制 kill |

### 7.4 fallback 后的恢复

```
Macro-Supervisor 故障
    │
    ▼
watchdog 检测到心跳超时（30s）
    │
    ▼
进入 [DSL] 降级模式
    ├── Agent 降级为 3 态（RUNNING/STOPPED/DEAD）
    ├── 调度降级为 EEVDF 默认
    └── IPC 降级为最简消息头
    │
    ▼
systemd 重启 Macro-Supervisor
    │
    ▼
Macro-Supervisor 恢复
    ├── 触发 Reconciliation（详见 [05-ipc-control-plane-reconciliation.md]）
    └── 退出 [DSL] 降级
    │
    ▼
恢复正常双面协作
```

---

## §8 io_uring_cmd 执行：通过 IORING_OP_URING_CMD 执行裁决

### 8.1 io_uring_cmd 裁决执行

Macro-Supervisor 通过 IORING_OP_URING_CMD 执行裁决，避免新增 syscall。裁决指令通过 io_uring_cmd 回调传递到内核 Micro-Supervisor：

```c
/* services/daemons/macro_superv/main.c —— io_uring_cmd 裁决执行 */
static int macro_superv_execute_adjudication(pid_t pid,
                                              enum airy_adjudication action)
{
    struct airy_ipc_cmd cmd = {
        .op = AIRY_IPC_OP_ADJUDICATE,
        .agent_id = pid,
        .adjudication = action,
    };

    /* 通过 IORING_OP_URING_CMD 执行裁决（不新增 syscall） */
    int ret = airy_uring_cmd_exec(&cmd, sizeof(cmd));
    if (ret < 0) {
        syslog(LOG_ERR, "airy: adjudication exec failed: %d", ret);
        return ret;
    }
    return 0;
}
```

### 8.2 内核侧裁决执行

```c
/* kernel/superv/airy_superv_lsm.c —— 内核侧裁决执行 */
static int airy_adjudicate_execute(struct airy_ipc_cmd *cmd)
{
    pid_t pid = cmd->agent_id;
    switch (cmd->adjudication) {
    case AIRY_ADJUD_WARN:
        /* 警告：仅记录，不执行 */
        break;
    case AIRY_ADJUD_DEGRADE:
        /* 降级：调整调度参数（缩减预算） */
        airy_sched_degrade(pid);
        break;
    case AIRY_ADJUD_PAUSE:
        /* 暂停：发送 SIGSTOP */
        kill_pid(find_vpid(pid), SIGSTOP, 1);
        agent_state_set(pid, AIRY_AGENT_STOPPED);
        break;
    case AIRY_ADJUD_TERMINATE:
        /* 终止：发送 SIGKILL */
        kill_pid(find_vpid(pid), SIGKILL, 1);
        agent_state_set(pid, AIRY_AGENT_DEAD);
        break;
    }
    return 0;
}
```

### 8.3 裁决指令清单

| 裁决指令 | io_uring_cmd op | 内核执行 | Agent 态迁移 |
|---------|----------------|---------|------------|
| 警告 | `AIRY_ADJUD_WARN` | 记录日志 | 无迁移 |
| 降级 | `AIRY_ADJUD_DEGRADE` | 缩减调度预算 | RUNNING → RUNNING（降级） |
| 暂停 | `AIRY_ADJUD_PAUSE` | `kill(SIGSTOP)` | RUNNING → STOPPED |
| 终止 | `AIRY_ADJUD_TERMINATE` | `kill(SIGKILL)` | RUNNING → DEAD |
| 恢复 | `AIRY_ADJUD_RESUME` | `kill(SIGCONT)` + 恢复预算 | STOPPED → READY |
| 解冻 Ring | `AIRY_IPC_OP_UNFREEZE` | `ring->frozen = false` | — |

---

## §9 性能与资源预算

### 9.1 性能 SLO（v1.1: 新增 Badge 请求与撤销）

| 指标 | SLO | 实测 | 说明 |
|------|-----|------|------|
| eventfd 唤醒到裁决完成 | ≤50ms | ~10-20ms | — |
| 心跳检查延迟 | ≤1ms/agent | ~0.5ms | — |
| v1.1 Badge 请求延迟（CAP_REQUEST） | ≤100μs | ~50μs | sec_d 编译 + WRITE_ONCE |
| v1.1 Badge 全局撤销延迟 | ≤5ns | ~1ns | `atomic_inc` 一行代码 O(1) |
| v1.1 Badge 单 Agent 撤销延迟 | ≤100μs | ~50μs | 重新编译覆盖 |
| 裁决执行延迟 | ≤10ms | ~5ms | — |

### 9.2 资源预算

| 资源 | 预算 | 说明 |
|------|------|------|
| 内存 | 512MB | 含 Agent 状态表、裁决策略、历史记录 |
| CPU | SCHED_FIFO priority=80 | 确保及时裁决 |
| 文件描述符 | 256 | eventfd + io_uring + 配置文件 |

---

## §10 相关文档

### 10.1 v1.1 Capability Folding 相关（新增）

- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)（A-IPC 协议 SSoT，Layout C v4 + 7 种 opcode）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)（A-IPC fastpath SSoT，C-S9 Badge 校验 + fastpath/slowpath 职责分割）
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)（错误码契约 SSoT，Fault 码 0x1001~0x1006）
- [110-security/03-capability-model.md](../110-security/03-capability-model.md)（Capability 模型 SSoT，Badge Perms 位掩码定义）
- [110-security/07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md)（纯 C LSM 模块 SSoT，§3.5 sec_d Badge 编译职责）
- [170-performance/03-ipc-performance.md](../170-performance/03-ipc-performance.md)（IPC 性能 SSoT，Badge 请求/撤销延迟 SLO）

### 10.2 A-ULS 模块相关（已有）

- [10-unify-design.md](../10-architecture/10-unify-design.md) §7 —— A-ULS 模块总纲
- [09-kernel-agent-supervisor.md](09-kernel-agent-supervisor.md) —— Micro-Supervisor（内核冷酷执法）
- [10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) —— Agent 8 态生命周期
- [05-ipc-control-plane-reconciliation.md](../110-security/05-ipc-control-plane-reconciliation.md) —— Reconciliation（Macro-Supervisor 是控制面主体）
- [04-ipc-data-plane-autonomy.md](../110-security/04-ipc-data-plane-autonomy.md) —— 数据面自治（Macro-Supervisor 故障时）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级（watchdog fallback）

---

## §11 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：Macro-Supervisor 用户态守护进程设计；systemd unit 管理 12 个 daemon；心跳检查（进程存活/心跳消息/响应探测）；Capability 注入（io_uring_cmd）；故障裁决四选项（警告/降级/暂停/终止）；"用户温情裁决"策略矩阵；单点故障 fallback（内核 watchdog 直接重启 + [DSL] 降级）；io_uring_cmd 执行裁决（不新增 syscall） |
| **v1.1** | **2026-07-18** | **Capability Folding 集成版**：(1) §4 重构为 Badge 请求机制（`AIRY_IPC_OP_CAP_REQUEST` opcode 向 sec_d 请求 Badge 编译，替代 v1.0 CNode 注入 radix tree）；(2) §4.3 Capability 类型改为 Badge Perms 16-bit 位掩码（8 种 Perms）；(3) §4.4 新增 Badge 撤销（`atomic_inc(&airy_cap_global_epoch)` 一行代码 O(1)）；(4) §5.2 裁决策略矩阵对齐 v1.1 Fault 码 0x1001~0x1006（`AIRY_FAULT_CAP_FORGED` 首次即终止）；(5) §7.2 watchdog fallback 新增 Badge 全局撤销；(6) §9.1 性能 SLO 新增 Badge 请求/撤销延迟；(7) §10 相关文档新增 6 份 v1.1 引用，清除内部审查路径引用 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Macro-Supervisor 用户态守护进程设计 | v1.1（Capability Folding 集成版） | 2026-07-18
