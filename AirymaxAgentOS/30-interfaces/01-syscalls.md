# 系统调用接口

> **文档定位**: agentrt-liunx（AirymaxOS） 内核系统调用的分类、编号、C 接口、清单、性能约束与错误码
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **父文档**: [接口设计](README.md)

---

## 1. 系统调用分类

agentrt-liunx 在 Linux 6.6 内核基线的标准系统调用之上，新增 Agent 感知专用系统调用，覆盖 8 子仓的能力入口。系统调用分为 6 类：

| # | 分类 | 覆盖子仓 | 代表调用 | 同源 agentrt |
|---|------|---------|---------|--------------|
| 1 | Agent 任务管理 | kernel / cognition | `agentrt_sys_task_submit` | MicroCoreRT 调度 |
| 2 | IPC | kernel / services | `agentrt_sys_ipc_send` | AgentsIPC |
| 3 | 内存 | kernel / memory | `agentrt_sys_rovol_snapshot` | MemoryRovol |
| 4 | 调度 | kernel | `agentrt_sys_sched_set_policy` | MicroCoreRT sub-scheduler |
| 5 | 安全 | kernel / security | `agentrt_sys_capability_request` | Cupolas 权限 |
| 6 | 认知 | kernel / cognition | `agentrt_sys_clt_phase_notify` | CoreLoopThree |

**设计原则**:

- **机制在内核**: 系统调用仅提供机制（提交、发送、快照、授权），策略由用户态定义。
- **io_uring 优先**: 高频路径（IPC、内存迁移）优先走 io_uring 零拷贝通道，系统调用仅用于控制面。
- **capability 守卫**: 所有安全敏感系统调用必须先通过 `agentrt_sys_capability_request` 获取令牌。
- **同源兼容**: IPC 与记忆卷载相关的系统调用语义与 agentrt AgentsIPC / MemoryRovol 保持兼容。

### 1.1 调用路径

agentrt-liunx 系统调用的典型调用路径分为控制面与数据面：

| 路径类型 | 路径 | 延迟量级 | 用途 |
|---------|------|---------|------|
| 控制面（syscall） | 用户态 → syscall → 内核 → 返回 | ~1 μs | 任务提交、capability 申请、策略设置 |
| 数据面（io_uring） | 用户态 → SQE → 内核 → CQE → 用户态 | ~10 μs | IPC 收发、记忆迁移、流式数据 |

控制面用于低频、需同步语义的操作；数据面用于高频、可异步、需零拷贝的操作。二者配合实现"机制在内核、策略在用户态"的微内核目标。

### 1.2 capability 守卫流程

安全敏感系统调用统一走 capability 守卫流程：

```c
/* 1. 申请 capability 令牌 */
int cap = agentrt_sys_capability_request("file.read", "/etc/agentrt/config.yaml");
if (cap < 0) {
    log_write(LOG_ERROR, "capability denied: %d", cap);
    return cap;  /* AGENTRT_EPERM */
}

/* 2. 携带令牌执行受保护操作 */
int ret = agentrt_sys_ipc_send(hdr, payload);
if (ret < 0) {
    log_write(LOG_ERROR, "ipc_send failed: errno=%d", ret);
}

/* 3. 操作完成后撤销令牌（可选） */
agentrt_sys_capability_revoke(cap);
```

capability 令牌由内核生成、不可伪造（seL4 风格），通过 IPC 消息可跨进程传递（详见 [02-ipc-protocol.md](02-ipc-protocol.md) `AGENTRT_IPC_FLAG_CAP_CARRY` 标志）。

---

## 2. 系统调用编号规则

### 2.1 命名前缀

所有 agentrt-liunx 专用系统调用统一使用 `AGENTRT_SYS_` 前缀（C 符号使用小写 `agentrt_sys_` 前缀），与 Linux 标准系统调用区分。

| 前缀 | 用途 | 示例 |
|------|------|------|
| `AGENTRT_SYS_TASK_*` | Agent 任务管理 | `AGENTRT_SYS_TASK_SUBMIT` |
| `AGENTRT_SYS_IPC_*` | IPC 消息传递 | `AGENTRT_SYS_IPC_SEND` / `AGENTRT_SYS_IPC_RECV` |
| `AGENTRT_SYS_ROVOL_*` | 记忆卷载 | `AGENTRT_SYS_ROVOL_SNAPSHOT` / `AGENTRT_SYS_ROVOL_RESTORE` |
| `AGENTRT_SYS_SCHED_*` | 调度策略 | `AGENTRT_SYS_SCHED_SET_POLICY` |
| `AGENTRT_SYS_CAP_*` | capability 权限 | `AGENTRT_SYS_CAPABILITY_REQUEST` |
| `AGENTRT_SYS_CLT_*` | 认知循环 | `AGENTRT_SYS_CLT_PHASE_NOTIFY` |

### 2.2 编号分配

agentrt-liunx 专用系统调用编号从 `512` 起始分配（避开 Linux 6.6 标准 0-511 编号空间），按分类分段：

| 编号段 | 分类 | 数量 |
|--------|------|------|
| 512-531 | Agent 任务管理 | 20 |
| 532-551 | IPC | 20 |
| 552-571 | 内存（MemoryRovol） | 20 |
| 572-591 | 调度 | 20 |
| 592-611 | 安全（capability） | 20 |
| 612-631 | 认知（CoreLoopThree） | 20 |

### 2.3 ABI 稳定性

- 编号在 MAJOR 版本内不可变更。
- 新增调用只能追加到对应分类段末尾，不可复用已废弃编号。
- 废弃调用保留编号但返回 `-AGENTRT_ENOSYS`，并在 Doxygen 注释中标注 `@deprecated`。

---

## 3. C 接口定义

所有系统调用通过 `AGENTRT_API` 宏导出，遵循 4 空格缩进 + snake_case + Doxygen 注释规范（详见 [04-coding-standard.md](04-coding-standard.md)）。头文件位置：`airymaxos-kernel/include/uapi/agentrt_syscalls.h`。

### 3.1 导出宏

```c
/* agentrt_api.h */
#if defined(__GNUC__) && __GNUC__ >= 4
    #define AGENTRT_API __attribute__((visibility("default")))
#else
    #define AGENTRT_API
#endif
```

### 3.2 关键系统调用 C 接口

```c
/**
 * @brief 提交 Agent 任务到内核调度器
 *
 * @param task_desc 任务描述符指针（同源 agentrt AgentsIPC 128B 消息头）
 * @param priority 任务优先级（0-139，SCHED_AGENT 调度类）
 * @return 任务 ID（>0 成功，<0 失败）
 *
 * @since 1.0.1
 * @see SCHED_AGENT 调度类
 */
AGENTRT_API int agentrt_sys_task_submit(const agentrt_task_desc_t *task_desc,
                                       uint32_t priority);

/**
 * @brief 发送 IPC 消息（io_uring 零拷贝）
 *
 * @param hdr 128B 消息头指针
 * @param payload 消息负载指针
 * @return 0 成功，<0 失败
 *
 * @since 1.0.1
 */
AGENTRT_API int agentrt_sys_ipc_send(const agentrt_ipc_msg_hdr_t *hdr,
                                    const void *payload);

/**
 * @brief 申请 capability 权限
 *
 * @param capability 权限名称（如 "file.read"）
 * @param resource 资源路径
 * @return 0 授权，<0 拒绝
 *
 * @since 1.0.1
 */
AGENTRT_API int agentrt_sys_capability_request(const char *capability,
                                              const char *resource);
```

### 3.3 补充 C 接口

```c
/**
 * @brief 创建进程记忆快照（同源 MemoryRovol）
 *
 * @param pid 目标进程 ID
 * @param snapshot_id_out 快照 ID 输出指针
 * @return 0 成功，<0 失败
 *
 * @since 1.0.1
 * @see agentrt_sys_rovol_restore
 */
AGENTRT_API int agentrt_sys_rovol_snapshot(uint32_t pid, uint64_t *snapshot_id_out);

/**
 * @brief 从快照恢复进程记忆
 *
 * @param snapshot_id 快照 ID
 * @param pid 目标进程 ID
 * @return 0 成功，<0 失败
 *
 * @since 1.0.1
 */
AGENTRT_API int agentrt_sys_rovol_restore(uint64_t snapshot_id, uint32_t pid);

/**
 * @brief 设置 sched_ext 调度策略（SCHED_AGENT 调度类）
 *
 * @param cgroup_path cgroup 路径
 * @param policy 策略名称（scx_realtime / scx_batch / scx_interactive / scx_agent）
 * @return 0 成功，<0 失败
 *
 * @since 1.0.1
 */
AGENTRT_API int agentrt_sys_sched_set_policy(const char *cgroup_path,
                                            const char *policy);

/**
 * @brief CoreLoopThree 阶段通知
 *
 * @param task_id Agent 任务 ID
 * @param phase 阶段（0=perception / 1=thinking / 2=action）
 * @return 0 成功，<0 失败
 *
 * @since 1.0.1
 */
AGENTRT_API int agentrt_sys_clt_phase_notify(int task_id, uint32_t phase);

/**
 * @brief 接收 IPC 消息（io_uring 零拷贝）
 *
 * @param hdr 消息头输出指针
 * @param payload_buf 负载缓冲区指针
 * @param payload_cap 负载缓冲区容量
 * @return 0 成功，<0 失败
 *
 * @since 1.0.1
 */
AGENTRT_API int agentrt_sys_ipc_recv(agentrt_ipc_msg_hdr_t *hdr,
                                    void *payload_buf,
                                    uint32_t payload_cap);
```

---

## 4. 关键系统调用清单

下表列出 ~20 个关键系统调用，覆盖 8 子仓能力入口。

| 编号 | 调用名 | 分类 | 覆盖子仓 | 说明 |
|------|--------|------|---------|------|
| 512 | `agentrt_sys_task_submit` | 任务管理 | kernel / cognition | 提交 Agent 任务到 SCHED_AGENT |
| 513 | `agentrt_sys_task_cancel` | 任务管理 | kernel / cognition | 取消 Agent 任务 |
| 514 | `agentrt_sys_task_status` | 任务管理 | kernel / cognition | 查询任务状态 |
| 515 | `agentrt_sys_task_migrate` | 任务管理 | kernel / cloudnative | 跨超节点迁移任务 |
| 532 | `agentrt_sys_ipc_send` | IPC | kernel / services | io_uring 零拷贝发送 |
| 533 | `agentrt_sys_ipc_recv` | IPC | kernel / services | io_uring 零拷贝接收 |
| 534 | `agentrt_sys_ipc_register_ring` | IPC | kernel / services | 注册跨进程 io_uring ring |
| 552 | `agentrt_sys_rovol_snapshot` | 内存 | memory | 进程记忆快照 |
| 553 | `agentrt_sys_rovol_restore` | 内存 | memory | 记忆恢复 |
| 554 | `agentrt_sys_rovol_migrate` | 内存 | memory / cognition | 记忆跨节点迁移 |
| 555 | `agentrt_sys_cxl_tier_set` | 内存 | memory | CXL 内存分层策略 |
| 556 | `agentrt_sys_mglru_config` | 内存 | memory | MGLRU（多代 LRU）配置 |
| 572 | `agentrt_sys_sched_set_policy` | 调度 | kernel | 设置 sched_ext 策略 |
| 573 | `agentrt_sys_sched_get_policy` | 调度 | kernel | 查询当前策略 |
| 592 | `agentrt_sys_capability_request` | 安全 | security | 申请 capability 令牌 |
| 593 | `agentrt_sys_capability_revoke` | 安全 | security | 撤销 capability |
| 594 | `agentrt_sys_lsm_load_policy` | 安全 | security | 加载 agent_lsm 策略 |
| 612 | `agentrt_sys_clt_phase_notify` | 认知 | cognition | CoreLoopThree 阶段通知 |
| 613 | `agentrt_sys_clt_register_kthread` | 认知 | cognition | 注册 CoreLoopThree kthread |
| 614 | `agentrt_sys_wasm_load_module` | 认知 | cognition / security | 加载 Wasm 3.0 模块 |

---

## 5. 系统调用性能约束

系统调用性能约束对齐非功能性需求 NFR-P-001（详见 [00-requirements/03-non-functional-requirements.md](../00-requirements/03-non-functional-requirements.md)）。

### 5.1 NFR-P-001 调度延迟

| 约束 ID | 指标 | 阈值 | 测量方法 |
|---------|------|------|---------|
| NFR-P-001 | Agent 任务调度延迟 | < 100 ms（P99） | `agentrt_sys_task_submit` 到任务首次执行 |
| NFR-P-001a | 系统调用本身开销 | < 1 μs（P99） | strace + perf 测量 |
| NFR-P-001b | io_uring IPC 往返延迟 | < 10 μs（P99） | `agentrt_sys_ipc_send` + `agentrt_sys_ipc_recv` |

### 5.2 调度路径优化

- **SCHED_AGENT 调度类**：通过 sched_ext（agentrt-liunx 内核增强，主线 6.12+）在用户态 eBPF 程序中实现调度策略，避免内核态策略切换开销。
- **EEVDF 调度器**：Linux 6.6 原生 EEVDF 调度器提供混合抢占模式，兼顾吞吐与响应。
- **CoreLoopThree 阶段感知**：`agentrt_sys_clt_phase_notify` 在思考阶段提升优先级，行动阶段恢复正常，减少关键路径抢占。

### 5.3 性能回归保护

- 每次提交运行 `airymaxos-tests/benchmark/sched-latency` 微基准。
- 与基线对比，调度延迟退化 > 5% 自动打回（详见 [20-modules/08-tests.md](../20-modules/08-tests.md) 第 4.6 节）。

### 5.4 性能剖析方法

系统调用性能剖析基于 Linux 6.6 原生可观测性能力，遵循 E-2 可观测性原则：

| 工具 | 用途 | 示例 |
|------|------|------|
| `perf trace` | 系统调用延迟直方图 | `perf trace -e agentrt_sys_* --summary` |
| `bpftrace` | 动态追踪系统调用参数与耗时 | `bpftrace -e 'tracepoint:agentrt:sys_ipc_send { ... }'` |
| `perf stat` | 调度器与 cache 事件计数 | `perf stat -e sched:* agentctl bench ipc` |
| `io_uring-bench` | io_uring IPC 吞吐基准 | `io_uring-bench --zerocopy --msg-size 128` |

剖析结果通过 OpenTelemetry Metrics 导出，与 `airymaxos-cloudnative/observability` 集成，形成持续性能基线。

### 5.5 优先级与延迟预算

agentrt-liunx 为不同 Agent 任务类别定义延迟预算（latency budget），由 SCHED_AGENT 调度类强制：

| 任务类别 | cgroup | 优先级范围 | 延迟预算（P99） | 典型场景 |
|---------|---------|-----------|----------------|---------|
| 实时控制 | `realtime.slice` | 0-49 | < 1 ms | 具身智能运动控制 |
| 交互响应 | `interactive.slice` | 50-99 | < 10 ms | 用户对话补全 |
| Agent 认知 | `agent.slice` | 100-119 | < 100 ms | CoreLoopThree 思考 |
| 批处理推理 | `batch.slice` | 120-139 | < 1 s | LLM 批量推理 |

超出延迟预算的任务由 sub-scheduler 触发 `AGENTRT_ETIMEOUT` 错误码，由 SDK 层按重试策略处理（详见 [03-sdk-api.md](03-sdk-api.md) 第 7 章）。

---

## 6. 错误码定义

错误码对齐 `agentrt_errno.h`，与 agentrt 同源且部分代码共享（IRON-9 v2）。错误码统一使用 `AGENTRT_E*` 前缀，负值返回。

| 错误码 | 值 | 含义 | 触发场景 |
|--------|-----|------|---------|
| `AGENTRT_EOK` | 0 | 成功 | 调用成功 |
| `AGENTRT_EINVAL` | -1 | 无效参数 | 参数为 NULL 或非法值 |
| `AGENTRT_ENOMEM` | -2 | 内存不足 | 内核分配失败 |
| `AGENTRT_ENOSYS` | -3 | 未实现 | 编号未实现或已废弃 |
| `AGENTRT_EPERM` | -4 | 权限不足 | capability 令牌缺失 |
| `AGENTRT_ENOENT` | -5 | 资源不存在 | 任务 ID / 快照 ID 不存在 |
| `AGENTRT_EAGAIN` | -6 | 重试 | io_uring 队列满，需重试 |
| `AGENTRT_EMSGSIZE` | -7 | 消息过大 | payload 超过最大长度 |
| `AGENTRT_EBADF` | -8 | 描述符错误 | ring fd / capability 句柄无效 |
| `AGENTRT_EBUSY` | -9 | 资源繁忙 | 任务正在迁移，无法快照 |
| `AGENTRT_ENOTSUP` | -10 | 不支持 | 硬件不支持（如无 CXL 设备） |
| `AGENTRT_ETIMEOUT` | -11 | 超时 | 调度等待超时 |
| `AGENTRT_ECONFLICT` | -12 | 状态冲突 | 任务状态不允许当前操作 |

### 6.1 错误码使用规范

```c
/* 正确：检查返回值并传递错误码 */
int ret = agentrt_sys_ipc_send(hdr, payload);
if (ret < 0) {
    log_write(LOG_ERROR, "ipc_send failed: errno=%d (%s)",
              ret, agentrt_strerror(ret));
    return ret;
}
```

### 6.2 错误码转换

- 与 Linux 标准 `errno` 的转换通过 `agentrt_errno_to_linux()` 工具函数完成。
- 与 agentrt 应用层错误码的转换通过 `agentrt_errno_to_app()` 工具函数完成。
- 转换表在 `agentrt_errno.h` 中以静态数组定义，便于维护。

### 6.3 错误码稳定性

- `AGENTRT_E*` 错误码值在 MAJOR 版本内不可变更。
- 新增错误码只能追加到末尾，不可复用已废弃值。
- 错误码字符串描述通过 `agentrt_strerror()` 提供，与 agentrt 同源保持描述一致。

---

## 7. 相关文档

- [接口设计](README.md)
- [IPC 协议](02-ipc-protocol.md)
- [SDK API](03-sdk-api.md)
- [编码规范](04-coding-standard.md)
- [内核设计](../20-modules/01-kernel.md)
- [安全设计](../20-modules/03-security.md)
- [非功能性需求](../00-requirements/03-non-functional-requirements.md)（NFR-P-001）

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
