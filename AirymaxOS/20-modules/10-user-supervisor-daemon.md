Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Macro-Supervisor 用户态守护进程设计
> **文档定位**：A-ULS（统一生命周期管理）模块的用户态监管组件唯一权威设计\
> **文档版本**：v1.1.1（落后内容修复版，基于 v1.1 Capability Folding）\
> **最后更新**：2026-07-19\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §7\
> **设计依据**：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射）+ A-IPC v1.1（Capability Folding 单平面架构）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Macro-Supervisor 用户态守护进程** 的唯一权威源。systemd unit 管理 12 个 daemon、心跳检查、**v1.1 Badge 请求与撤销**（通过 `AIRY_IPC_OP_CAP_REQUEST` opcode 请求 sec_d 编译 Badge，通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码撤销所有 Badge）、故障裁决（警告/降级/暂停/终止）、"用户温情裁决"策略、单点故障 fallback（内核 watchdog 直接重启 + 全局 Epoch 撤销）、io_uring_cmd 执行裁决均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 Macro-Supervisor 职责与裁决流程。
>
> **v1.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.1 起，Macro-Supervisor 不再直接注入 CNode 到 radix tree——而是通过 `AIRY_IPC_OP_CAP_REQUEST` opcode 向 sec_d 请求 Badge，sec_d 编译 Badge 64-bit Native Word（`Epoch<<48 | RandomTag<<16 | Perms`）并写入 `agent_caps[agent_id]` 静态数组（详见 [07-airy-lsm-design.md §3.5](../110-security/07-airy-lsm-design.md)）。Macro-Supervisor 通过 io_uring_cmd 执行裁决（不新增 syscall），所有裁决指令通过 `AIRY_IPC_OP_ADJUDICATE` opcode 传递到内核 Micro-Supervisor。Badge 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码 O(1) 立即失效所有已发出 Badge（无 drain、无 bitmap、无 IPI），详见 [09-kernel-agent-supervisor.md §6.4](09-kernel-agent-supervisor.md)。
>
> 技术选型声明：IPC 采用 **IORING_OP_URING_CMD + registered buffer + mmap**（**不使用 page flipping**）。整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。
>
> **ADR 引用声明**：本设计遵循 ADR-012（对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造技术路线确立）与 ADR-014（微内核设计思想来源单一化——seL4 为唯一思想来源，禁止混用 Mach/L4/Fuchsia/Zircon 等其他微内核设计模式）。Macro-Supervisor 的"温情裁决 + 用户态守护"机制严格对齐 seL4 用户态 fault handler 模型，不引入其他微内核的等价机制。IRON-9 v3 四层模型（[SC] 共享契约层 / [SS] 语义同源层 / [IND] 完全独立层 / [DSL] 降级生存层）是 Macro-Supervisor 与 Micro-Supervisor 协作的同源映射框架，详见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)。

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
| 主进程 | `services/daemons/macro_d/main.c` | 事件循环、裁决分发 |
| 事件循环 | `services/daemons/macro_d/event_loop.c` | io_uring + eventfd |
| 裁决引擎 | `services/daemons/macro_d/adjudicate.c` | 故障裁决策略 |
| 调度注入 | `services/daemons/macro_d/sched.c` | 调度参数注入 |
| Capability 注入 | `services/daemons/macro_d/cap_inject.c` | io_uring_cmd 注入 |
| 心跳检查 | `services/daemons/macro_d/heartbeat.c` | Agent 进程状态检查 |
| systemd unit | `systemd/agentrt-macro.service` | 自动重启管理 |

### 1.3 12 个 daemon 职责

Macro-Supervisor 管理的 12 个 daemon 及其职责：

| # | Daemon | 职责 | Agent 态管理 |
|---|--------|------|------------|
| 1 | macro_d | 主监管守护进程（A-ULS） | 自身（高优先级） |
| 2 | logger_d | 日志消费（A-ULP） | 心跳监控 |
| 3 | config_d | 配置管理（A-UCS） | 心跳监控 |
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
# /etc/systemd/system/agentrt-macro.service
[Unit]
Description=Airymax Macro-Supervisor (A-ULS user-side)
After=systemd-modules-load.service airy-log.service
Requires=airy-log.service
# 依赖内核 Micro-Supervisor 模块
After=airy-superv.service

[Service]
Type=simple
ExecStart=/usr/sbin/airy-macro-d --config=/etc/agentrt/macro_d.conf
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
ReadWritePaths=/var/lib/airy/
# 委派权限：可管理其他 daemon
Delegate=yes

[Install]
WantedBy=multi-user.target
```

### 2.2 子 daemon 拉起流程

Macro-Supervisor 启动后按依赖顺序拉起 12 个 daemon：

```c
/* services/daemons/macro_d/main.c —— daemon 拉起 */
struct daemon_spec {
    const char *name;
    const char *exec;
    const char *config;
    int sched_policy;     /* SCHED_FIFO / SCHED_NORMAL */
    int sched_priority;   /* 仅 SCHED_FIFO 使用 */
    int restart_max;      /* 最大重启次数 */
};

static const struct daemon_spec daemons[] = {
    { "logger_d",      "/usr/sbin/airy-logger-d",      "/etc/agentrt/logger.conf",     SCHED_FIFO, 50, 5 },
    { "cogn_d",        "/usr/sbin/airy-cogn-d",        "/etc/agentrt/cognition.conf",  SCHED_DEADLINE, 0, 3 },
    { "mem_d",         "/usr/sbin/airy-mem-d",         "/etc/agentrt/memory.conf",     SCHED_FIFO, 40, 5 },
    { "sec_d",         "/usr/sbin/airy-sec-d",         "/etc/agentrt/sec.conf",        SCHED_FIFO, 60, 5 },
    { "config_d", "/usr/sbin/airy-config-d", "/etc/agentrt/config.conf",     SCHED_NORMAL, 0, 5 },
    /* ... 其余 daemon（gateway_d / sched_d / vfs_d / net_d / audit_d / dev_d / macro_d）
     * 注: v1.1 Capability Folding 后 IPC 数据传递完全由 io_uring IORING_OP_URING_CMD 承载，
     * 不存在独立 ipc daemon；Badge 编译由 sec_d 承担，跨节点 IPC 由 gateway_d 承担。
     */
};

static int macro_d_spawn_daemons(void)
{
    for (int i = 0; i < ARRAY_SIZE(daemons); i++) {
        const struct daemon_spec *d = &daemons[i];
        pid_t pid = fork();
        if (pid == 0) {
            /* 子进程：设置调度策略（sched_tac） */
            if (d->sched_policy == SCHED_DEADLINE) {
                macro_d_set_deadline(getpid(), d);
            } else if (d->sched_policy == SCHED_FIFO) {
                macro_d_set_fifo(getpid(), d->sched_priority);
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
1. logger_d        （无依赖，最先启动）
2. config_d   （依赖 logger_d）
3. mem_d           （依赖 config_d）
4. sec_d           （依赖 mem_d；v1.1 替代原独立 ipc daemon——IPC 数据传递由 io_uring 承载，Badge 编译由 sec_d 承担）
5. cogn_d          （依赖 sec_d，认知调度需要 Badge）
6-12. 其他 daemon  （gateway_d / sched_d / vfs_d / net_d / audit_d / dev_d / macro_d，依赖前序）
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
/* services/daemons/macro_d/heartbeat.c —— 心跳检查 */
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
                macro_d_adjudicate_fault(g_heartbeats[i].pid,
                                               AIRY_FAULT_TIMEOUT);
            } else if (ret == HEARTBEAT_DEAD) {
                /* 进程已死，重启 */
                macro_d_restart(g_heartbeats[i].pid);
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
/* services/daemons/macro_d/cap_inject.c —— v1.1 Badge 请求
 * Macro-Supervisor 不再构造 CNode，而是请求 sec_d 编译 Badge
 */
static int macro_d_cap_request(__u32 agent_id, __u16 perms)
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

**v1.1 变更**：原 v1.0 的 `IPC_SEND/IPC_RECV/RING_CREATE/CAP_MINT/CAP_REVOKE` 五种 Capability 类型已废弃，改为 Badge Perms 16-bit 位掩码（详见 [03-capability-model.md §2.5](../110-security/03-capability-model.md) SSoT）。**v1.1.1 修正**：Perms 位命名统一为 `AIRY_CAP_PERM_*`（对齐 SSoT），FREEZE 位编号修正为 bit 5 (0x0020)。

| Perms 位 | 位掩码 | bit | 含义 | 对应 opcode |
|---------|--------|:---:|------|------------|
| `AIRY_CAP_PERM_SEND` | 0x0001 | 0 | 允许向 Ring 发送消息 | SEND (0x0001) / CANCEL (0x0004，复用 SEND) |
| `AIRY_CAP_PERM_RECV` | 0x0002 | 1 | 允许从 Ring 接收消息 | RECV (0x0002) |
| `AIRY_CAP_PERM_CALL` | 0x0004 | 2 | 允许 RPC 调用 | CALL（保留） |
| `AIRY_CAP_PERM_GRANT` | 0x0008 | 3 | 允许派生 Badge（mint/mintcopy 等价） | CAP_CARRY 标志 |
| `AIRY_CAP_PERM_REVOKE` | 0x0010 | 4 | 允许撤销 Badge（`airy_sys_call` + `REVOKE_BADGE`） | sec_d 专属 |
| `AIRY_CAP_PERM_FREEZE` | 0x0020 | 5 | 允许冻结/解冻 Ring（FREEZE / UNFREEZE opcode） | FREEZE (0x0005) / UNFREEZE |
| `AIRY_CAP_PERM_BATCH` | 0x0040 | 6 | 允许批量发送（SEND_BATCH opcode） | SEND_BATCH (0x0003) |
| `AIRY_CAP_PERM_ADMIN` | 0x8000 | 15 | 管理员权限（包含所有 Perms，仅 sec_d / macro_d） | —（v1.1 扩展，非 SSoT） |

> **Perms SSoT 对齐声明**：`AIRY_CAP_PERM_SEND` ~ `AIRY_CAP_PERM_BATCH` 7 个权限位严格对齐 [03-capability-model.md §2.5](../110-security/03-capability-model.md) SSoT。`AIRY_CAP_PERM_ADMIN` (0x8000) 是 Macro-Supervisor 用户态扩展（聚合权限位，非 SSoT 定义）。CAP_REQUEST opcode (0x0010) 是自举路径，**无需权限位**（fastpath C-S9 直接放行，详见 [03-capability-model.md §2.6.1](../110-security/03-capability-model.md)）。

### 4.4 v1.1 Badge 撤销（一行代码 O(1)）

**v1.1 新增**：Badge 撤销通过 `atomic_inc(&airy_cap_global_epoch)` 一行代码完成，立即失效所有已发出 Badge。这是 Capability Folding 单平面架构的关键设计——无需 drain、无需 bitmap、无需 IPI，全局 Epoch 自增后所有 fastpath C-S9.EPOCH 检查（`badge_epoch == global_epoch`）立即失败。

```c
/* services/daemons/macro_d/cap_revoke.c —— v1.1 Badge 撤销
 * 一行代码撤销所有 Badge，O(1) 复杂度
 */
static int macro_d_cap_revoke_all(void)
{
    /* 1. 全局 Epoch 自增——立即失效所有已发出 Badge
     * 所有 fastpath C-S9.EPOCH 检查（badge_epoch == global_epoch）立即失败
     * 详见 09-kernel-agent-supervisor.md §6.4
     */
    atomic_inc(&airy_cap_global_epoch);

    /* 2. 记录撤销日志 */
    syslog(LOG_WARNING, "airy: all badges revoked (epoch incremented)");
    airy_audit_emit(AGENT_CAP_REVOKE_ALL, 0, 0);

    return 0;
}

/* 撤销单个 Agent 的 Badge（通过重新编译覆盖） */
static int macro_d_cap_revoke_one(__u32 agent_id)
{
    /* 重新编译 Badge 时，WRITE_ONCE 会覆盖 agent_caps[agent_id].randtag
     * 旧 Badge 的 RandomTag 立即失效（C-S9.RANDTAG 检查失败）
     * 详见 110-security/07-airy-lsm-design.md §3.5
     */
    return macro_d_cap_request(agent_id, 0);  /* perms=0 即撤销 */
}
```

### 4.5 sec_d 故障恢复流程（v1.1 新增）

**设计背景**：sec_d 是 Badge 的唯一编译者（H4），其崩溃会导致新 Agent 无法获取 Badge、Badge 撤销无法执行。但已发出 Badge 的 fastpath C-S9 校验**不依赖 sec_d**（agent_caps[] 是内核静态数组，sec_d 崩溃后仍可读取），因此 sec_d 崩溃**不影响已有 IPC 通信**，仅影响新 Badge 的编译与撤销。

#### 4.5.1 sec_d 崩溃检测

sec_d 由 systemd 管理（[§2 进程拉起](#)），systemd 通过 `Restart=always` + `WatchdogSec=` 自动检测 sec_d 崩溃并重启。检测机制：

| 检测方式 | 阈值 | 触发动作 |
|---------|------|---------|
| systemd Restart=always | 进程退出立即触发 | 自动重启 sec_d |
| systemd WatchdogSec=5s | 5 秒无心跳 | systemd 强制重启 sec_d |
| 内核 airy_sec_d_health_check | 10 秒无 CAP_RESPONSE | 内核标记 sec_d OFFLINE，触发 [DSL] 降级 |

#### 4.5.2 sec_d 崩溃期间的系统行为（H4 降级）

sec_d 崩溃期间，系统按以下降级模式运行：

| 功能 | 状态 | 说明 |
|-----|------|------|
| 已建立 IPC 通信 | ✅ 正常 | fastpath C-S9 基于 agent_caps[] 校验，不依赖 sec_d |
| 新 Agent 启动 | ❌ 阻塞 | CAP_REQUEST 无 sec_d 响应，返回 `AIRY_EDSL_CAP_MINIMAL`(-206) |
| Badge 撤销 | ⚠️ 延迟 | `atomic_inc` 仍可执行（内核操作），但审计日志缺失 |
| 新 Ring 创建 | ❌ 阻塞 | 需 sec_d 授权 |
| [DSL] 降级模式 | ⏳ 10 秒后触发 | 内核检测 sec_d OFFLINE 后自动进入 [DSL] |

#### 4.5.3 sec_d 重启后的恢复流程

sec_d 重启后执行以下 5 阶段恢复流程：

```
sec_d 崩溃
    │
    ▼
[阶段 1] systemd 自动重启 sec_d（<1s）
    │
    ▼
[阶段 2] sec_d 加载持久化的 Badge 状态（<100ms）
    │  ├── 读取 /var/lib/airy/sec_d_state.json
    │  ├── 恢复 agent_caps[] 的 randtag + perms + state
    │  └── 恢复 airy_cap_global_epoch 值
    │
    ▼
[阶段 3] Epoch 自增 + 全量重新编译（<500ms）
    │  ├── atomic_inc(&airy_cap_global_epoch)  ← 旧 Badge 全部失效
    │  ├── 遍历持久化状态，重新编译所有 ACTIVE Agent 的 Badge
    │  └── 新 Random Tag（旧 Random Tag 不复用）
    │
    ▼
[阶段 4] 通知 Macro-Supervisor 重新分发 Badge
    │  ├── CAP_RESPONSE 广播给所有 ACTIVE Agent
    │  └── Macro-Supervisor 持有新 Badge opaque handle
    │
    ▼
[阶段 5] 退出 [DSL] 降级模式，恢复正常双面协作
```

#### 4.5.4 agent_caps[] 状态持久化设计

**关键设计**：sec_d 在每次 Badge 编译/撤销后，将 agent_caps[] 关键字段持久化到 `/var/lib/airy/sec_d_state.json`，确保 sec_d 重启后可恢复。

```c
/* services/daemons/sec_d/persist.c —— agent_caps[] 状态持久化
 *
 * 物理宿主: services/daemons/sec_d/persist.c
 * 调用者:   sec_d（在 airy_cap_badge_compile / airy_cap_badge_revoke 后调用）
 * 同步:     写入时使用 O_APPEND + fsync，确保崩溃一致性
 *
 * 持久化字段（仅持久化 sec_d 重启后无法重建的信息）:
 *   - agent_caps[i].randtag    ← Random Tag（必须持久化，否则无法恢复）
 *   - agent_caps[i].perms      ← 权限位（必须持久化，否则需重新授权）
 *   - agent_caps[i].state      ← Agent 状态（ACTIVE/REVOKED/EXPIRED）
 *   - airy_cap_global_epoch    ← 全局 Epoch（必须持久化，避免回退）
 *
 * 不持久化字段:
 *   - agent_caps[i].reserved   ← 对齐填充，无需持久化
 */

struct airy_cap_persist_entry {
    uint32_t agent_id;
    uint32_t randtag;
    uint16_t perms;
    uint8_t  state;
    uint8_t  reserved[3];  /* 对齐填充 */
} __aligned(64);  /* v1.1.1: OLK 6.6 工程规范——强制 64 字节对齐，替代原 packed 属性（OLK 6.6 禁用 packed 导致的跨字段读取性能损耗与字段重排问题） */

struct airy_cap_persist_header {
    uint32_t magic;       /* 'ASCP' = Airymax Sec_d Cap Persist */
    uint32_t version;     /* 1 */
    uint16_t global_epoch;
    uint16_t reserved;
    uint32_t entry_count; /* ACTIVE Agent 数量 */
};

/**
 * airy_cap_persist_save - 持久化 agent_caps[] 状态到磁盘
 *
 * 在每次 Badge 编译/撤销后调用。使用 O_TRUNC + O_WRONLY 重写整个文件
 * （文件小，~12KB for 1024 Agents）。
 *
 * 返回: 0 成功；<0 错误码
 */
static int airy_cap_persist_save(void)
{
    struct airy_cap_persist_header hdr = {
        .magic = 0x41534350,  /* 'ASCP' */
        .version = 1,
        .global_epoch = (uint16_t)atomic_read(&airy_cap_global_epoch),
        .entry_count = 0,
    };
    struct airy_cap_persist_entry entries[AIRY_CAP_MAX_AGENTS];
    int fd, ret, i;

    /* 1. 收集所有 ACTIVE Agent 的状态 */
    for (i = 0; i < AIRY_CAP_MAX_AGENTS; i++) {
        if (READ_ONCE(agent_caps[i].state) == AIRY_CAP_STATE_ACTIVE) {
            entries[hdr.entry_count++] = (struct airy_cap_persist_entry){
                .agent_id = i,
                .randtag = READ_ONCE(agent_caps[i].randtag),
                .perms = READ_ONCE(agent_caps[i].perms),
                .state = READ_ONCE(agent_caps[i].state),
            };
        }
    }

    /* 2. 原子写入：先写 .tmp 文件，再 rename（避免崩溃时文件损坏）*/
    fd = open("/var/lib/airy/sec_d_state.json.tmp",
              O_WRONLY | O_CREAT | O_TRUNC, 0600);
    if (fd < 0)
        return -errno;

    ret = write(fd, &hdr, sizeof(hdr));
    if (ret < 0) goto out;
    ret = write(fd, entries, hdr.entry_count * sizeof(entries[0]));
    if (ret < 0) goto out;

    fsync(fd);  /* 确保数据落盘 */
    close(fd);

    /* 3. atomic rename */
    ret = rename("/var/lib/airy/sec_d_state.json.tmp",
                 "/var/lib/airy/sec_d_state.json");
    return ret < 0 ? -errno : 0;

out:
    close(fd);
    return -errno;
}

/**
 * airy_cap_persist_load - 从磁盘加载 agent_caps[] 状态
 *
 * sec_d 重启时调用。加载后执行 Epoch 自增 + 全量重新编译（见 §4.5.3 阶段 3）。
 *
 * 返回: 0 成功；<0 错误码
 */
static int airy_cap_persist_load(void)
{
    struct airy_cap_persist_header hdr;
    struct airy_cap_persist_entry *entries;
    int fd, ret, i;

    fd = open("/var/lib/airy/sec_d_state.json", O_RDONLY);
    if (fd < 0)
        return -errno;

    ret = read(fd, &hdr, sizeof(hdr));
    if (ret < 0 || hdr.magic != 0x41534350) {
        close(fd);
        return -EINVAL;
    }

    entries = calloc(hdr.entry_count, sizeof(*entries));
    if (!entries) {
        close(fd);
        return -ENOMEM;
    }

    ret = read(fd, entries, hdr.entry_count * sizeof(*entries));
    close(fd);
    if (ret < 0) {
        free(entries);
        return -EINVAL;
    }

    /* 1. 恢复全局 Epoch（避免 Epoch 回退）*/
    atomic_set(&airy_cap_global_epoch, hdr.global_epoch);

    /* 2. 恢复 agent_caps[]（注意：阶段 3 会 Epoch 自增 + 重新编译）*/
    for (i = 0; i < hdr.entry_count; i++) {
        WRITE_ONCE(agent_caps[entries[i].agent_id].randtag,
                   entries[i].randtag);
        WRITE_ONCE(agent_caps[entries[i].agent_id].perms,
                   entries[i].perms);
        WRITE_ONCE(agent_caps[entries[i].agent_id].state,
                   entries[i].state);
    }

    free(entries);
    return 0;
}
```

#### 4.5.5 sec_d 重启后的 Reconciliation

sec_d 重启后，除了恢复 agent_caps[] 状态，还需与 Macro-Supervisor 执行 Reconciliation：

| Reconciliation 步骤 | 执行者 | 动作 |
|---------------------|-------|------|
| 1. 加载持久化状态 | sec_d | `airy_cap_persist_load()` |
| 2. Epoch 自增 | sec_d | `atomic_inc(&airy_cap_global_epoch)`（旧 Badge 全失效）|
| 3. 全量重新编译 | sec_d | 遍历 ACTIVE Agent，重新编译 Badge（新 Random Tag）|
| 4. 通知 Macro-Supervisor | sec_d | 广播 `CAP_RESPONSE`（新 Badge opaque handle）|
| 5. Macro-Supervisor 更新 | Macro-Supervisor | 持有新 Badge，分发给 Agent |
| 6. 旧 Badge 失效验证 | fastpath C-S9 | 旧 Badge Epoch 不匹配，返回 `-AIRY_ECAP_EPOCH`(-79) |
| 7. 退出 [DSL] 降级 | 内核 | 标记 sec_d ONLINE，恢复双面协作 |

#### 4.5.6 sec_d 故障恢复的 SLO

| 指标 | SLO | 说明 |
|-----|-----|------|
| sec_d 重启时间 | <1s | systemd Restart=always |
| 状态加载时间 | <100ms | 读取 ~12KB 持久化文件 |
| 全量重新编译时间 | <500ms | 1000 个 Agent × 0.5ms/个 |
| 总恢复时间 | <2s | 从崩溃到恢复正常 |
| 已有 IPC 通信中断 | 0s | fastpath 不依赖 sec_d |
| 新 Badge 编译中断 | <2s | sec_d 恢复后立即可用 |

#### 4.5.7 sec_d 持久化文件的完整性保护

`/var/lib/airy/sec_d_state.json` 是 sec_d 故障恢复的关键文件，需防止篡改：

| 保护机制 | 实现方式 |
|---------|---------|
| 文件权限 | `0600`（仅 root 可读写）|
| 文件属主 | `root:root` |
| 完整性校验 | 文件头部含 CRC32，加载时校验 |
| 原子写入 | `.tmp` 文件 + `rename()`（避免崩溃时损坏）|
| TPM 封存（可选）| 1.0.1 阶段通过 TPM 封存 Epoch + Random Tag，防离线篡改 |

---

## §4.6 sec_d 限流机制（R1 补强：防滥用过载）

**设计背景**：sec_d 是 Badge 的唯一编译者（H4 硬约束），所有 Agent 的 Badge 编译/撤销请求都经由 `AIRY_IPC_OP_CAP_REQUEST` opcode 汇聚到 sec_d。若恶意 Agent 高频发起 Badge 编译请求（如循环重试、批量 Agent 同时启动），可能导致 sec_d 过载，影响其他正常 Agent 的 Badge 服务。本节定义 sec_d 的多层限流机制，保障 sec_d 在滥用场景下的可用性。

**与前序会话决策对齐**：本节落实前序会话"sec_d 处理延迟 ≤50ms"的 SLO 要求（详见 §9.1），通过配额+排队+拒绝的多阶段限流机制实现（注：本节"多阶段"指时序上的限流处理阶段，**与 IRON-9 v3 四层模型 [SC]/[SS]/[IND]/[DSL] 无语义重叠**——IRON-9 v3 四层是 Capability Folding 同源映射框架，详见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)）。

### 4.6.1 限流架构总览

```
Agent CAP_REQUEST 请求
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 第 1 层：per-Agent 配额检查                       │
│   每 Agent ≤ N 次/秒（默认 N=10）                  │
│   超额 → 进入排队                                 │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 第 2 层：全局阈值检查                              │
│   sec_d 总请求率 ≤ 1000 req/s                     │
│   超额 → 进入排队                                 │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 第 3 层：排队调度（FIFO + 优先级）                  │
│   队列容量 256，超出 → 拒绝                       │
│   处理延迟 ≤ 50ms                                 │
└─────────────────────────────────────────────────┘
    │
    ▼
 sec_d 编译 Badge（airy_cap_badge_compile）
```

### 4.6.2 限流参数（sysctl 可配置）

| 参数 | sysctl 路径 | 默认值 | 说明 |
|------|------------|--------|------|
| per-Agent 配额 | `airy.sec_d.badge_compile_quota` | 10 | 每 Agent 每秒最多 Badge 编译请求次数 |
| 全局阈值 | `airy.sec_d.global_rate_limit` | 1000 | sec_d 总请求率上限（req/s） |
| 队列容量 | `airy.sec_d.request_queue_capacity` | 256 | 超出后返回 `-AIRY_ESEC_D_THROTTLED` |
| 处理延迟 SLO | `airy.sec_d.process_latency_slo_ms` | 50 | 单请求处理时间上限（ms） |
| 优先级高水位 | `airy.sec_d.priority_high_watermark` | 8 | 高优先级 Agent 阈值（用于调度） |

### 4.6.3 限流器数据结构与接口

```c
/* services/daemons/sec_d/throttle.c —— sec_d 限流器（R1 补强）
 *
 * 物理宿主: services/daemons/sec_d/throttle.c
 * 调用者:   sec_d 主循环（在 airy_cap_badge_compile 之前调用）
 * 同步:     限流器状态由 sec_d 单线程持有，无需加锁（单写者）
 *
 * 设计原则:
 *   1. 多层防护：per-Agent 配额 + 全局阈值 + 排队调度
 *   2. 优先级调度：关键 Agent（macro_d/sched_d）优先级高
 *   3. 公平性：同优先级 FIFO，避免饥饿
 *   4. 可观测：所有限流事件记录到 A-ULP 审计日志
 */

#define AIRY_SEC_D_DEFAULT_AGENT_QUOTA      10     /* per-Agent 每秒配额 */
#define AIRY_SEC_D_DEFAULT_GLOBAL_LIMIT     1000   /* 全局每秒阈值 */
#define AIRY_SEC_D_DEFAULT_QUEUE_CAPACITY   256    /* 排队队列容量 */
#define AIRY_SEC_D_LATENCY_SLO_MS           50     /* 处理延迟 SLO */
#define AIRY_SEC_D_PRIO_HIGH_WATERMARK      8      /* 高优先级阈值 */

/* Agent 优先级（数值越大优先级越高）*/
enum airy_sec_d_prio {
    AIRY_SEC_D_PRIO_LOW      = 0,  /* 普通 Agent */
    AIRY_SEC_D_PRIO_NORMAL   = 4,  /* 业务 Agent */
    AIRY_SEC_D_PRIO_HIGH     = 8,  /* 关键 daemon（macro_d/sched_d）*/
    AIRY_SEC_D_PRIO_CRITICAL = 16, /* 系统关键（仅 sec_d 自举）*/
};

/* per-Agent 限流状态 */
struct airy_sec_d_agent_quota {
    uint32_t agent_id;
    uint32_t tokens;            /* 当前可用令牌（令牌桶）*/
    uint32_t refill_rate;       /* 每秒令牌补充速率（默认 10）*/
    uint64_t last_refill_ns;    /* 上次令牌补充时间戳 */
    enum airy_sec_d_prio prio;  /* Agent 优先级 */
};

/* 排队请求条目 */
struct airy_sec_d_request_entry {
    uint32_t agent_id;
    uint16_t perms;
    enum airy_sec_d_prio prio;
    uint64_t enqueue_ns;        /* 入队时间戳（用于 SLO 监控）*/
    struct list_head list;      /* FIFO 链表节点 */
};

/* sec_d 限流器主结构 */
struct airy_sec_d_throttle {
    /* per-Agent 配额表（索引 = agent_id）*/
    struct airy_sec_d_agent_quota agent_quotas[AIRY_CAP_MAX_AGENTS];

    /* 全局令牌桶 */
    uint32_t global_tokens;
    uint32_t global_refill_rate;        /* 默认 1000 */
    uint64_t global_last_refill_ns;

    /* 排队队列（按优先级分桶，桶内 FIFO）*/
    struct list_head queue_buckets[AIRY_SEC_D_PRIO_CRITICAL + 1];
    uint32_t queue_depth;               /* 当前队列长度 */
    uint32_t queue_capacity;            /* 默认 256 */

    /* 统计信息 */
    uint64_t total_requests;
    uint64_t total_throttled;
    uint64_t total_rejected;
};

/**
 * airy_sec_d_throttle_check - 检查请求是否通过限流
 * @t:        限流器实例
 * @agent_id: 请求 Agent ID
 * @perms:    请求的权限位
 *
 * 限流决策流程:
 *   1. 令牌桶补充（按时间差补充 tokens）
 *   2. per-Agent 配额检查（agent_quotas[agent_id].tokens > 0）
 *   3. 全局阈值检查（global_tokens > 0）
 *   4. 通过 → 扣减 tokens，返回 0
 *   5. 不通过 → 入队排队，返回 -EAGAIN（异步处理）
 *   6. 队列满 → 返回 -AIRY_ESEC_D_THROTTLED
 *
 * 返回: 0 允许立即处理；-EAGAIN 排队等待；-AIRY_ESEC_D_THROTTLED 拒绝
 */
static int airy_sec_d_throttle_check(struct airy_sec_d_throttle *t,
                                       uint32_t agent_id, uint16_t perms)
{
    struct airy_sec_d_agent_quota *aq;
    uint64_t now = ktime_get_real_ns();

    /* 1. 令牌桶补充（per-Agent + 全局）*/
    airy_sec_d_refill_tokens(t, now);

    /* 2. per-Agent 配额检查 */
    aq = &t->agent_quotas[agent_id];
    if (aq->tokens > 0 && t->global_tokens > 0) {
        /* 配额充足：扣减令牌，允许处理 */
        aq->tokens--;
        t->global_tokens--;
        t->total_requests++;
        return 0;
    }

    /* 3. 配额不足：尝试入队排队 */
    if (t->queue_depth >= t->queue_capacity) {
        /* 队列已满：拒绝 */
        t->total_rejected++;
        syslog(LOG_WARNING,
               "airy: sec_d throttle rejected (agent=%u queue=%u/%u)",
               agent_id, t->queue_depth, t->queue_capacity);
        return -AIRY_ESEC_D_THROTTLED;  /* -83（详见 08-sc-error-contract.md §2.4）*/
    }

    /* 4. 入队等待（FIFO + 优先级）*/
    airy_sec_d_enqueue_request(t, agent_id, perms, aq->prio, now);
    t->total_throttled++;
    return -EAGAIN;
}

/**
 * airy_sec_d_throttle_drain - 从队列取出下一个待处理请求
 * @t: 限流器实例
 *
 * 调度策略：优先级高的桶优先，同桶内 FIFO。
 * 每次调用取出一个请求，由 sec_d 主循环处理。
 *
 * 返回: 请求条目指针；NULL 表示队列空
 */
static struct airy_sec_d_request_entry *
airy_sec_d_throttle_drain(struct airy_sec_d_throttle *t)
{
    /* 从高优先级桶向低优先级桶扫描 */
    for (int prio = AIRY_SEC_D_PRIO_CRITICAL; prio >= 0; prio--) {
        if (!list_empty(&t->queue_buckets[prio])) {
            struct airy_sec_d_request_entry *entry;
            entry = list_first_entry(&t->queue_buckets[prio],
                                       struct airy_sec_d_request_entry, list);
            list_del(&entry->list);
            t->queue_depth--;

            /* SLO 监控：检查排队延迟 */
            uint64_t now = ktime_get_real_ns();
            uint64_t latency_ms = (now - entry->enqueue_ns) / 1000000ULL;
            if (latency_ms > AIRY_SEC_D_LATENCY_SLO_MS) {
                syslog(LOG_WARNING,
                       "airy: sec_d queue latency %llums > SLO %ums",
                       (unsigned long long)latency_ms,
                       AIRY_SEC_D_LATENCY_SLO_MS);
            }
            return entry;
        }
    }
    return NULL;
}
```

### 4.6.4 限流 SLO 与降级策略

| 场景 | SLO | 触发条件 | 处理动作 |
|------|-----|---------|---------|
| 正常请求 | ≤50ms | per-Agent + 全局配额充足 | 立即处理 |
| 排队请求 | ≤50ms（含排队） | 配额不足，队列未满 | FIFO + 优先级调度 |
| 拒绝请求 | 立即返回 | 队列已满（depth ≥ 256） | 返回 `-AIRY_ESEC_D_THROTTLED`(-83) |
| 持续过载 | 触发 [DSL] | sec_d 10 秒无 CAP_RESPONSE | 内核标记 sec_d OFFLINE，进入 [DSL] 降级（详见 §4.5.2）|

### 4.6.5 限流事件审计

所有限流事件记录到 A-ULP 审计日志，便于运维分析：

| 事件类型 | 审计字段 | 触发频率 |
|---------|---------|---------|
| `AIRY_AUDIT_SEC_D_THROTTLED` | agent_id, queue_depth, latency_ms | 排队时 |
| `AIRY_AUDIT_SEC_D_REJECTED` | agent_id, queue_capacity, reason | 拒绝时 |
| `AIRY_AUDIT_SEC_D_SLO_VIOLATION` | agent_id, latency_ms, slo_ms | 排队延迟超 SLO 时 |

> **错误码说明**：`-AIRY_ESEC_D_THROTTLED` 在 [08-sc-error-contract.md §2.4](../30-interfaces/08-sc-error-contract.md) 中定义为 -83（Capability 码空间 `[-71, -100]` 内的下一个可用值；-82 已分配给 `AIRY_ECAP_FROZEN`）。

---

## §4.7 sec_d 故障恢复协议（R7 补强：Snapshot + WAL 两阶段恢复）

**设计背景**：§4.5.4 的 `airy_cap_persist_save()` 采用"全量重写 + rename"的原子写入策略，在单文件场景下可保证崩溃一致性。但在高频 Badge 编译场景下（如 10000 Agent 同时启动），每次 Badge 编译都触发全量重写（~12KB），写放大严重且无法精确恢复到故障前最后状态。本节补充 **Snapshot + WAL 两阶段恢复协议**，作为 §4.5.4 的增强方案，提供更细粒度的崩溃一致性与更快的恢复速度。

**与前序会话决策对齐**：本节落实前序会话"snapshot + WAL"恢复协议设计，补强 §4.5.3 阶段 2 的状态加载语义。

### 4.7.1 Snapshot 机制（全量快照）

| 属性 | 值 |
|------|-----|
| 快照文件路径 | `/var/lib/airy/sec_d.snapshot` |
| 快照间隔 | 5 分钟（可通过 `airy.sec_d.snapshot_interval` sysctl 配置，单位秒）|
| 快照内容 | `agent_caps[1024]` 全量 + `airy_cap_global_epoch` 当前值 |
| 快照大小 | ~16KB（1024 × 12B entry + 16B header）|
| 写入策略 | O_DIRECT + fdatasync（绕过页缓存，避免污染）|
| 原子性 | 写入 `.tmp` 文件 → fdatasync → rename（崩溃时旧快照仍可用）|

### 4.7.2 WAL（Write-Ahead Log）机制

| 属性 | 值 |
|------|-----|
| WAL 文件路径 | `/var/lib/airy/sec_d.wal` |
| 追加策略 | O_APPEND \| O_DIRECT，每次操作追加一条记录 |
| 同步策略 | 写入后立即 `fdatasync(fd)`，确保落盘后才返回 Badge |
| 记录格式 | `{op_type, agent_id, randtag, perms, state, epoch, ts_ns, crc32}` |
| 记录大小 | 32B（紧凑布局）|
| 截断策略 | 每次 snapshot 成功后清空 WAL（`ftruncate(fd, 0)`）|
| 文件大小上限 | 16MB（默认），超出后触发强制 snapshot |

**WAL 记录类型**：

```c
/* services/daemons/sec_d/wal.c —— WAL 记录类型 */
enum airy_sec_d_wal_op {
    AIRY_WAL_OP_COMPILE    = 1,   /* Badge 编译（新增/覆盖 randtag+perms）*/
    AIRY_WAL_OP_REVOKE_ONE = 2,   /* 单 Agent 撤销（state → REVOKED）*/
    AIRY_WAL_OP_REVOKE_ALL = 3,   /* 全局撤销（epoch 自增，所有 Badge 失效）*/
    AIRY_WAL_OP_EXPIRE     = 4,   /* 过期失效（state → EXPIRED）*/
};
```

### 4.7.3 两阶段恢复协议

sec_d 重启后执行两阶段恢复：

```
sec_d 重启
    │
    ▼
[阶段 1] 加载 Snapshot（恢复至 5 分钟前状态）
    │  ├── open("/var/lib/airy/sec_d.snapshot", O_RDONLY | O_DIRECT)
    │  ├── 读取 header（magic + version + epoch + entry_count）
    │  ├── 校验 CRC32（头部 + entries）
    │  ├── 恢复 agent_caps[i] 的 randtag + perms + state
    │  └── 恢复 airy_cap_global_epoch（snapshot 时刻值）
    │  耗时: <100ms（16KB 顺序读）
    │
    ▼
[阶段 2] Replay WAL（恢复至故障前最后状态）
    │  ├── open("/var/lib/airy/sec_d.wal", O_RDONLY)
    │  ├── 顺序读取每条 WAL 记录（32B）
    │  ├── 校验每条记录 CRC32
    │  ├── 按 op_type 重放：
    │  │   ├── COMPILE    → WRITE_ONCE(agent_caps[id].{randtag,perms,state})
    │  │   ├── REVOKE_ONE → WRITE_ONCE(agent_caps[id].state, REVOKED)
    │  │   ├── REVOKE_ALL → atomic_inc(&airy_cap_global_epoch)
    │  │   └── EXPIRE     → WRITE_ONCE(agent_caps[id].state, EXPIRED)
    │  └── 重放至 WAL 末尾（EOF）
    │  耗时: <500ms（最坏 16MB / 32B = 524288 条记录）
    │
    ▼
[阶段 3] Epoch 自增（强制所有已发出 Badge 失效）
    │  └── atomic_inc(&airy_cap_global_epoch)
    │  效果: 所有旧 Badge 的 Epoch 不匹配，C-S9.EPOCH 检查失败
    │  Agent 需重新通过 CAP_REQUEST 申请 Badge
    │
    ▼
[阶段 4] 通知 Macro-Supervisor 重新分发 Badge
    │  └── 广播 CAP_RESPONSE（详见 §4.5.3 阶段 4）
    │
    ▼
[阶段 5] 退出 [DSL] 降级模式，恢复正常服务
```

### 4.7.4 故障检测（systemd watchdog）

| 检测方式 | 阈值 | 配置项 | 触发动作 |
|---------|------|--------|---------|
| systemd `WatchdogSec=` | 3 秒 | `/etc/systemd/system/airy-sec_d.service` | systemd 强制重启 sec_d |
| `sd_notify(WATCHDOG=1)` | 每 1 秒 | sec_d 主循环心跳 | 重置 watchdog 计时器 |
| 内核 `airy_sec_d_health_check` | 10 秒无 CAP_RESPONSE | 内核定时器 | 标记 sec_d OFFLINE，进入 [DSL] |

**systemd unit 配置示例**：

```ini
# /etc/systemd/system/airy-sec_d.service
[Service]
Type=simple
ExecStart=/usr/sbin/airy-sec_d --config=/etc/agentrt/sec_d.conf
Restart=always
RestartSec=1
WatchdogSec=3
# 内存限制（含 snapshot/WAL 缓冲）
MemoryMax=256M
# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/airy/
# 调度：SCHED_FIFO 优先级 85（高于 macro_d 的 80）
IOSchedulingClass=realtime
IOSchedulingPriority=1
```

### 4.7.5 数据一致性保障

| 一致性维度 | 实现方式 |
|-----------|---------|
| 写入原子性 | Snapshot：`.tmp` + rename；WAL：O_APPEND 单条记录原子写入 |
| 落盘持久性 | Snapshot：fdatasync；WAL：每条记录 fdatasync 后返回 |
| 页缓存隔离 | O_DIRECT 绕过页缓存（Snapshot + WAL 均使用）|
| CRC32 校验 | Snapshot 头部 + 每条 WAL 记录均含 CRC32，加载/重放时校验 |
| 崩溃恢复 | 两阶段恢复：Snapshot（基线）+ WAL（增量），最坏丢失 0 条记录 |
| 并发控制 | sec_d 单线程写入，无并发问题（H4 单写者）|

### 4.7.6 恢复协议伪代码

```c
/* services/daemons/sec_d/recovery.c —— 两阶段恢复协议（R7 补强）
 *
 * 物理宿主: services/daemons/sec_d/recovery.c
 * 调用者:   sec_d 启动入口（main 函数初始化阶段）
 * 依赖:     §4.5.4 airy_cap_persist_load（兼容旧格式）
 *
 * 设计:
 *   - 阶段 1: 加载 snapshot（恢复至 5 分钟前状态）
 *   - 阶段 2: replay WAL（恢复至故障前最后状态）
 *   - 阶段 3: Epoch 自增（强制所有旧 Badge 失效）
 *   - 阶段 4-5: 通知 Macro-Supervisor + 退出 [DSL]（见 §4.5.3）
 */

/**
 * airy_sec_d_recover - sec_d 两阶段恢复协议
 *
 * 返回: 0 成功；<0 错误码
 *   -EINVAL: snapshot 损坏（magic/CRC 不匹配）
 *   -EIO: WAL 读取失败
 *   -ENOENT: snapshot/WAL 文件不存在（首次启动）
 */
static int airy_sec_d_recover(void)
{
    int ret;

    /* 阶段 1: 加载 Snapshot */
    ret = airy_sec_d_snapshot_load("/var/lib/airy/sec_d.snapshot");
    if (ret == -ENOENT) {
        syslog(LOG_INFO, "airy: no snapshot found, first boot");
        /* 首次启动：初始化空的 agent_caps[] */
        atomic_set(&airy_cap_global_epoch, 0);
    } else if (ret < 0) {
        syslog(LOG_ERR, "airy: snapshot load failed: %d", ret);
        return ret;
    } else {
        syslog(LOG_INFO, "airy: snapshot loaded (epoch=%u)",
               atomic_read(&airy_cap_global_epoch));
    }

    /* 阶段 2: Replay WAL */
    ret = airy_sec_d_wal_replay("/var/lib/airy/sec_d.wal");
    if (ret < 0) {
        syslog(LOG_ERR, "airy: WAL replay failed: %d", ret);
        return ret;
    }
    syslog(LOG_INFO, "airy: WAL replayed (%u records)", ret);

    /* 阶段 3: Epoch 自增（强制所有已发出 Badge 失效）*/
    atomic_inc(&airy_cap_global_epoch);
    syslog(LOG_WARNING, "airy: epoch incremented to %u (post-recovery)",
           atomic_read(&airy_cap_global_epoch));

    /* 阶段 4: 通知 Macro-Supervisor（详见 §4.5.3 阶段 4）*/
    airy_sec_d_notify_recovery_complete();

    /* 阶段 5: 退出 [DSL] 降级模式（由内核 airy_sec_d_health_check 检测 ONLINE）*/
    return 0;
}

/**
 * airy_sec_d_snapshot_load - 阶段 1: 加载 snapshot
 * @path: snapshot 文件路径
 *
 * 读取 snapshot 文件，恢复 agent_caps[] + global_epoch 至快照时刻状态。
 * 使用 O_DIRECT 绕过页缓存。
 *
 * 返回: 0 成功；-ENOENT 文件不存在；-EINVAL magic/CRC 不匹配
 */
static int airy_sec_d_snapshot_load(const char *path)
{
    struct airy_cap_persist_header hdr;
    struct airy_cap_persist_entry *entries;
    int fd, ret;

    fd = open(path, O_RDONLY | O_DIRECT);
    if (fd < 0)
        return (errno == ENOENT) ? -ENOENT : -errno;

    /* 读取头部（含 CRC32）*/
    ret = read(fd, &hdr, sizeof(hdr));
    if (ret != sizeof(hdr) || hdr.magic != 0x41534350) {
        close(fd);
        return -EINVAL;
    }

    /* 校验头部 CRC32 */
    if (!airy_crc32_verify(&hdr, sizeof(hdr))) {
        close(fd);
        syslog(LOG_ERR, "airy: snapshot header CRC mismatch");
        return -EINVAL;
    }

    /* 读取 entries */
    entries = calloc(hdr.entry_count, sizeof(*entries));
    if (!entries) { close(fd); return -ENOMEM; }
    ret = read(fd, entries, hdr.entry_count * sizeof(*entries));
    close(fd);
    if (ret < 0) { free(entries); return -EIO; }

    /* 恢复 agent_caps[] + global_epoch */
    atomic_set(&airy_cap_global_epoch, hdr.global_epoch);
    for (uint32_t i = 0; i < hdr.entry_count; i++) {
        WRITE_ONCE(agent_caps[entries[i].agent_id].randtag, entries[i].randtag);
        WRITE_ONCE(agent_caps[entries[i].agent_id].perms, entries[i].perms);
        WRITE_ONCE(agent_caps[entries[i].agent_id].state, entries[i].state);
    }

    free(entries);
    return 0;
}

/**
 * airy_sec_d_wal_replay - 阶段 2: replay WAL
 * @path: WAL 文件路径
 *
 * 顺序读取 WAL 文件，按 op_type 重放每条记录。
 * 校验每条记录 CRC32，损坏的记录停止 replay（保留已 replay 的状态）。
 *
 * 返回: ≥0 成功 replay 的记录数；<0 错误码
 */
static int airy_sec_d_wal_replay(const char *path)
{
    struct airy_sec_d_wal_record rec;
    int fd, ret, count = 0;

    fd = open(path, O_RDONLY);
    if (fd < 0)
        return (errno == ENOENT) ? 0 : -errno;  /* 无 WAL = 无增量 */

    while ((ret = read(fd, &rec, sizeof(rec))) == sizeof(rec)) {
        /* 校验记录 CRC32 */
        if (!airy_crc32_verify(&rec, sizeof(rec))) {
            syslog(LOG_WARNING, "airy: WAL record %u CRC mismatch, stop replay", count);
            break;
        }

        /* 按 op_type 重放 */
        switch (rec.op_type) {
        case AIRY_WAL_OP_COMPILE:
            WRITE_ONCE(agent_caps[rec.agent_id].randtag, rec.randtag);
            WRITE_ONCE(agent_caps[rec.agent_id].perms, rec.perms);
            WRITE_ONCE(agent_caps[rec.agent_id].state, AIRY_CAP_STATE_ACTIVE);
            break;
        case AIRY_WAL_OP_REVOKE_ONE:
            WRITE_ONCE(agent_caps[rec.agent_id].state, AIRY_CAP_STATE_REVOKED);
            break;
        case AIRY_WAL_OP_REVOKE_ALL:
            atomic_inc(&airy_cap_global_epoch);
            break;
        case AIRY_WAL_OP_EXPIRE:
            WRITE_ONCE(agent_caps[rec.agent_id].state, AIRY_CAP_STATE_EXPIRED);
            break;
        default:
            syslog(LOG_WARNING, "airy: unknown WAL op %u, skip", rec.op_type);
        }
        count++;
    }

    close(fd);
    return count;
}

/**
 * airy_sec_d_wal_append - 追加一条 WAL 记录（Badge 编译/撤销后调用）
 * @op:       操作类型
 * @agent_id: 目标 Agent ID
 * @randtag:  Random Tag（COMPILE 时有效）
 * @perms:    权限位（COMPILE 时有效）
 *
 * 使用 O_APPEND | O_DIRECT 追加，fdatasync 确保落盘后返回。
 *
 * 返回: 0 成功；<0 错误码
 */
static int airy_sec_d_wal_append(enum airy_sec_d_wal_op op, uint32_t agent_id,
                                   uint32_t randtag, uint16_t perms)
{
    struct airy_sec_d_wal_record rec = {
        .op_type   = op,
        .agent_id  = agent_id,
        .randtag   = randtag,
        .perms     = perms,
        .state     = AIRY_CAP_STATE_ACTIVE,
        .epoch     = (uint16_t)atomic_read(&airy_cap_global_epoch),
        .ts_ns     = ktime_get_real_ns(),
    };
    rec.crc32 = airy_crc32_compute(&rec, sizeof(rec) - sizeof(rec.crc32));

    int fd = open("/var/lib/airy/sec_d.wal", O_WRONLY | O_APPEND | O_DIRECT);
    if (fd < 0)
        return -errno;

    int ret = write(fd, &rec, sizeof(rec));
    if (ret != sizeof(rec)) {
        close(fd);
        return -EIO;
    }

    fdatasync(fd);  /* 确保落盘后才返回 */
    close(fd);
    return 0;
}
```

### 4.7.7 Snapshot + WAL 与 §4.5.4 的关系

| 维度 | §4.5.4（原方案）| §4.7（增强方案）|
|------|----------------|----------------|
| 持久化文件 | `/var/lib/airy/sec_d_state.json` | `/var/lib/airy/sec_d.snapshot` + `/var/lib/airy/sec_d.wal` |
| 写入策略 | 全量重写（O_TRUNC + rename）| Snapshot 周期全量 + WAL 增量追加 |
| 写放大 | 每次操作写 ~12KB | 每次操作写 32B（WAL），5 分钟一次 16KB（Snapshot）|
| 恢复粒度 | 恢复至最后一次持久化时刻 | 恢复至故障前最后一条 WAL 记录 |
| 落盘保证 | fsync | O_DIRECT + fdatasync（绕过页缓存）|
| 适用阶段 | 0.1.1（简单方案）| 0.1.1+（增强方案，推荐）|

**兼容性**：§4.7 的 Snapshot 文件格式与 §4.5.4 的 `sec_d_state.json` 二进制兼容（相同的 magic + header + entries 结构），sec_d 启动时优先尝试加载 `/var/lib/airy/sec_d.snapshot`，若不存在则回退到 `/var/lib/airy/sec_d_state.json`（向后兼容）。

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
/* services/daemons/macro_d/adjudicate.c —— 温情裁决 */
struct agent_violation_history {
    pid_t pid;
    __u32 violation_count[AIRY_FAULT_MAX];
    __u64 last_violation_ns;
};

static int macro_d_adjudicate(struct airy_fault_event *ev)
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
        macro_d_degrade_agent(ev->agent_id);
        break;
    case AIRY_ADJUD_PAUSE:
        macro_d_pause_agent(ev->agent_id);   /* kill(SIGSTOP) */
        break;
    case AIRY_ADJUD_TERMINATE:
        macro_d_terminate_agent(ev->agent_id);  /* kill(SIGKILL) */
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
static __u64 last_macro_d_heartbeat;

/* watchdog 超时检查（每 10 秒执行一次） */
static void airy_superv_watchdog_fn(struct timer_list *t)
{
    __u64 now = ktime_get_real_ns();
    /* Macro-Supervisor 心跳超时 30 秒 */
    if (now - last_macro_d_heartbeat > 30ULL * 1000000000ULL) {
        pr_emerg("airy: Macro-Supervisor heartbeat lost, watchdog takeover\n");

        /* 1. 立即冻结所有活跃 Ring（防止无监管期间数据损坏，最优先） */
        airy_ipc_freeze_all(AIRY_FAULT_TIMEOUT);

        /* 2. v1.1 新增：立即撤销所有 Badge（atomic_inc 一行代码 O(1)）
         * 防止无监管期间恶意 Agent 利用已发出 Badge 操作
         * 所有 fastpath C-S9.EPOCH 检查立即失败（badge_epoch != global_epoch）
         */
        atomic_inc(&airy_cap_global_epoch);
        pr_emerg("airy: all badges revoked (epoch=%u)\n",
                 atomic_read(&airy_cap_global_epoch));

        /* 3. 进入 [DSL] 降级模式 */
        airy_dsl_enter(AIRY_DSL_REASON_SUPERV_DEAD);

        /* 4. 内核直接重启 Macro-Supervisor（通过 systemd 通知）
         * v1.1.1: 原 force 系列 API 已废弃（OLK 6.6 中不存在），
         * 改用标准 API orderly_poweroff(force=true)
         */
        orderly_poweroff(true);
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
/* services/daemons/macro_d/main.c —— io_uring_cmd 裁决执行 */
static int macro_d_execute_adjudication(pid_t pid,
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

> **解冻协议（v1.1.1 统一：与 [09-kernel-agent-supervisor.md §3.5](09-kernel-agent-supervisor.md) 严格一致）**：上表中"解冻 Ring"的简化执行步骤仅作概要示意，完整解冻流程为：① Macro-Supervisor 通过 `AIRY_IPC_OP_UNFREEZE` opcode 向 **sec_d** 请求解冻（**禁止 Macro-Supervisor 直接设置 `ring->frozen = false`**——绕过 sec_d 串行化会破坏 Badge 一致性）；② **sec_d 串行化解冻**（避免并发解冻导致状态不一致，sec_d 内部持锁串行处理）；③ 解冻时执行 `atomic_inc(&airy_cap_global_epoch)`（Epoch 自增使旧 Badge 失效，强制 Agent 重新申请 Badge）；④ 最后通过 `ring->frozen = false` 解冻 Ring。sec_d 解冻全过程由 systemd watchdog 监控（`WatchdogSec=3`，超时触发 sec_d 重启 + 两阶段恢复，详见 §4.7）。

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
| **v1.1.1** | **2026-07-19** | **落后内容修复版**：① §1.3 修复 12 daemon 命名（原 logger/cognition/memory/security daemon 后缀统一为 `_d`，删除原独立 ipc daemon——IPC 由 io_uring 承载、Badge 编译由 sec_d 承担）；② §2.2/§2.3 daemon 拉起代码与依赖顺序对齐 12 daemon 命名；③ §4.3 Perms 表格完全重写——原 BADGE_PERM 前缀统一为 `AIRY_CAP_PERM_*`（对齐 03-capability-model.md SSoT），FREEZE 位编号修正为 bit 5 (0x0020)，移除 SSoT 不存在的 4 项非 SSoT opcode（CANCEL/CAP_REQUEST/CAP_REVOKE/RING_CREATE）；④ §4.5.4 修复原 packed 属性 → `__aligned(64)`（OLK 6.6 工程规范）；⑤ §7.2 watchdog fallback 修复原 force 系列 API → `orderly_poweroff(true)`（OLK 6.6 标准 API）+ 操作顺序调整（先冻结 Ring → 再撤销 Badge → 再降级 → 再重启，防止无监管期间数据损坏最优先）；⑥ §4.6 修复原限流机制表述——改为"多阶段限流机制"避免与 IRON-9 v3 四层模型语义冲突；⑦ §8.3 补充解冻协议完整流程说明（与 09-kernel-agent-supervisor.md §3.5 严格一致：sec_d 串行化 + Epoch 自增 + systemd watchdog 监控）；⑧ SSoT 声明新增 ADR-014 引用 + IRON-9 v3 四层模型引用 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | Macro-Supervisor 用户态守护进程设计 | v1.1.1 | 2026-07-19
