Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# A-UCS 统一配置管理体系设计
> **文档定位**：A-UCS（Unified Configuration Subsystem）统一配置管理体系模块的唯一权威设计\
> **文档版本**：v1.1（Capability Folding 集成版）\
> **最后更新**：2026-07-18\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §6\
> **设计依据**：本文件为 SSoT（Single Source of Truth），A-UCS 配置模型、热重载机制、Capability Folding A-IPC 配置项均以本文件为唯一权威定义

---

## SSoT 声明

> **单一权威源声明**：本文件是 **A-UCS 统一配置管理体系** 的唯一权威源。[SC] 共享配置（二进制布局相关）、[SS] 语义同源配置（内核 Kconfig/sysctl ↔ 用户态 JSON/TOML 语义映射）、RCU 热重载机制（指针原子切换）、配置版本号（`AIRY_CONFIG_VERSION`）、sysctl ↔ JSON 双向同步、**A-IPC Capability Folding 配置项**（fastpath 阈值、Badge Epoch、agent_caps 容量等）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 A-UCS 配置模型与热重载机制。
>
> **v1.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.1 起，A-UCS 新增 A-IPC Capability Folding 相关配置项——IPC 消息头大小（128B Layout C v4）、magic 值（0x41524531 'ARE1'）、`agent_caps[]` 静态数组容量（1024）、Badge Epoch 位宽（16-bit）、fastpath C-S9 校验超时阈值等。这些配置项是 Capability Folding 单平面架构落地的运行时参数 SSoT。详见 §7 A-IPC Capability Folding 配置项。
>
> 技术选型声明：整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/airymax/`。

---

## 文档信息卡

- **目标读者**：内核配置开发者、用户态配置开发者、运维工程师
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) §6 A-UCS 模块、[06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) IRON-9 v3 [SC]/[SS] 分层、Linux RCU 机制
- **预计阅读时间**：25 分钟
- **核心概念**：[SC] 共享配置、[SS] 语义同源配置、RCU 热重载、配置版本号、双向同步
- **复杂度标识**：中级

---

## §1 架构定位：A-UCS 模块的配置管理

### 1.1 在 Unify Design 中的定位

A-UCS（Unified Configuration Subsystem）是 Airymax Unify Design 五模块之一，负责统一配置管理体系（详见 [10-unify-design.md](../10-architecture/10-unify-design.md) §6）。A-UCS 覆盖"治理"技术领域，是 A-ULS 调度参数、A-IPC Ring 大小、A-ULP 日志级别的统一配置入口。

### 1.2 物理宿主

| 组件 | 路径 | 说明 |
|------|------|------|
| [SC] 头文件 | `kernel/include/airymax/config.h` | 共享配置常量 |
| 内核 sysctl | `kernel/config/airy_sysctl.c` | sysctl 注册与处理 |
| 内核 Kconfig | `kernel/config/Kconfig` | 编译时配置 |
| 用户态配置 | `services/daemons/config_daemon/` | JSON/TOML 解析与同步 |
| 配置文件 | `/etc/airymax/*.conf` | JSON 格式配置 |

### 1.3 配置分类

A-UCS 将配置分为两类，对应 IRON-9 v3 的 [SC] 与 [SS] 两个层级：

| 配置类型 | IRON-9 层级 | 特征 | 物理宿主 |
|---------|-----------|------|---------|
| [SC] 共享配置 | [SC] 共享契约层 | 二进制布局相关，两端逐字节相同 | `kernel/include/airymax/` |
| [SS] 语义同源配置 | [SS] 语义同源层 | 语义一致，签名独立演进 | sysctl + JSON |

---

## §2 [SC] 共享配置：IPC 消息头大小、Ring Buffer 大小、魔数

### 2.1 [SC] 配置范围

[SC] 共享配置**仅限二进制布局相关的配置项**，这些配置直接影响内核与用户态的二进制兼容性，必须在两端逐字节相同：

```c
/* kernel/include/airymax/config.h —— [SC] 共享配置 */
#ifndef _AIRYM_CONFIG_H
#define _AIRYM_CONFIG_H

#include "uapi_compat.h"

/* 配置版本号 */
#define AIRY_CONFIG_VERSION  1

/* IPC 消息头大小（二进制布局，两端必须一致）
 * v1.1: Layout C v4 定长 128B = 2 cache lines，含 capability_badge (offset 44-51)
 * 详见 30-interfaces/02-ipc-protocol.md + 50-engineering-standards/120-cross-project-code-sharing.md §2.7
 */
#define AIRY_IPC_HDR_SIZE   128   /* IPC 消息头固定 128B（Layout C v4） */
#define AIRY_IPC_MSG_MAX_PAYLOAD   (64 * 1024)  /* 最大 payload 64KB */

/* Ring Buffer 布局参数（二进制布局，两端必须一致） */
#define AIRY_LOG_RING_RECORD_SIZE   128   /* 日志记录固定 128B */
#define AIRY_LOG_RING_DEFAULT_CAP   4096  /* 默认 4096 条记录 */
#define AIRY_IPC_RING_DEFAULT_CAP   1024  /* IPC Ring 默认 1024 条 */

/* 魔数（二进制标识，两端必须一致）
 * v1.1: AIRY_IPC_MAGIC 从 0x41495043 'AIPC' 变更为 0x41524531 'ARE1'
 * （AIRY Runtime Epoch），同源 agentrt AgentsIPC，标识 Capability Folding
 * 单平面架构。magic 值一经发布即冻结（37 号 §5.2 H1 硬约束）。
 */
#define AIRY_LOG_MAGIC   0x414C4F47   /* 'ALOG' */
#define AIRY_IPC_MAGIC   0x41524531   /* 'ARE1' — Airymax Runtime Epoch（v1.1 Capability Folding） */

#endif /* _AIRYM_CONFIG_H */
```

### 2.2 [SC] 配置约束

[SC] 共享配置有以下严格约束：

| 约束 | 说明 | CI 校验 |
|------|------|---------|
| 逐字节相同 | agentrt 与 agentrt-linux 两端逐字节一致 | `sc-dual-ci.yml` |
| 仅二进制布局 | 仅影响结构体大小、字段偏移、魔数 | 静态分析 |
| 不可运行时修改 | 编译时确定，不支持热重载 | 无 sysctl |
| 版本号强制 | `AIRY_CONFIG_VERSION` 不匹配拒绝加载 | 启动时校验 |

### 2.3 [SC] 配置项清单

| 配置项 | 值 | 影响 | 热重载 |
|--------|-----|------|--------|
| `AIRY_IPC_HDR_SIZE` | 128B（v1.1: 64B→128B） | IPC 消息头结构大小（Layout C v4，含 capability_badge） | 否 |
| `AIRY_IPC_MSG_MAX_PAYLOAD` | 64KB | IPC 消息最大 payload | 否 |
| `AIRY_LOG_RING_RECORD_SIZE` | 128B | 日志记录大小 | 否 |
| `AIRY_LOG_RING_DEFAULT_CAP` | 4096 | 日志 Ring 默认容量 | 否 |
| `AIRY_IPC_RING_DEFAULT_CAP` | 1024 | IPC Ring 默认容量 | 否 |
| `AIRY_LOG_MAGIC` | 0x414C4F47 | 日志魔数 | 否 |
| `AIRY_IPC_MAGIC` | 0x41524531 'ARE1'（v1.1: 'AIPC'→'ARE1'） | IPC 魔数（Capability Folding 单平面架构标识，发布即冻结） | 否 |

---

## §3 [SS] 语义同源配置：内核 Kconfig/sysctl ↔ 用户态 JSON/TOML 语义映射

### 3.1 [SS] 配置模型

[SS] 语义同源配置是 A-UCS 的核心创新——内核侧 Kconfig/sysctl 与用户态 JSON/TOML 语义映射。语义一致，但签名独立演进：

| 维度 | 内核侧 | 用户态侧 |
|------|--------|---------|
| 配置格式 | Kconfig（编译时）/ sysctl（运行时） | JSON/TOML |
| 类型系统 | C 类型（int/bool/char*） | JSON 类型（number/bool/string） |
| 访问方式 | `sysctl` 读写 | 配置文件读写 |
| 签名 | 内核符号 | JSON key |

### 3.2 语义映射表

内核 sysctl 与用户态 JSON 的语义映射（v1.1: `cap.cache_ttl` 已废弃，替换为 Capability Folding 相关配置）：

| 内核 sysctl | 用户态 JSON | 语义 | 默认值 |
|------------|-----------|------|--------|
| `kernel.airy.sched_rt_runtime_us` | `sched.rt_runtime_us` | RT Agent 运行时间 | 950000 |
| `kernel.airy.log_min_level` | `log.min_level` | 最低日志级别 | LOG_INFO(6) |
| `kernel.airy.ipc_ring_size` | `ipc.ring_size` | IPC Ring 容量 | 1024 |
| `kernel.airy.superv_heartbeat_interval` | `superv.heartbeat_interval` | 心跳间隔 | 1(秒) |
| `kernel.airy.cap_max_agents` | `cap.max_agents` | v1.1: `agent_caps[]` 静态数组容量（H4 硬约束） | 1024 |
| `kernel.airy.cap_epoch_bits` | `cap.epoch_bits` | v1.1: Badge Epoch 位宽（默认 16-bit） | 16 |
| `kernel.airy.cap_randtag_bits` | `cap.randtag_bits` | v1.1: Badge Random Tag 位宽（默认 32-bit） | 32 |
| `kernel.airy.cap_forge_threshold` | `cap.forge_threshold` | v1.1: Badge 伪造检测阈值（连续失败次数，触发警报） | 100 |
| `kernel.airy.ipc_fastpath_budget_ns` | `ipc.fastpath_budget_ns` | v1.1: fastpath C-S9 Badge 校验时间预算（ns） | 10 |

### 3.3 语义同源实现

```c
/* kernel/config/airy_sysctl.c —— [SS] 语义同源 sysctl */
static struct airy_config __rcu *airy_cfg_ptr;

struct airy_config {
    __u32 sched_rt_runtime_us;   /* ↔ JSON sched.rt_runtime_us */
    __u16 log_min_level;          /* ↔ JSON log.min_level */
    __u32 ipc_ring_size;          /* ↔ JSON ipc.ring_size */
    __u32 superv_heartbeat_interval; /* ↔ JSON superv.heartbeat_interval */
    /* v1.1: Capability Folding 配置（替代旧 cap_cache_ttl） */
    __u32 cap_max_agents;         /* ↔ JSON cap.max_agents (agent_caps[] 容量，默认 1024) */
    __u16 cap_epoch_bits;         /* ↔ JSON cap.epoch_bits (Badge Epoch 位宽，默认 16) */
    __u16 cap_randtag_bits;       /* ↔ JSON cap.randtag_bits (Random Tag 位宽，默认 32) */
    __u32 cap_forge_threshold;    /* ↔ JSON cap.forge_threshold (伪造检测阈值，默认 100) */
    __u32 ipc_fastpath_budget_ns; /* ↔ JSON ipc.fastpath_budget_ns (C-S9 时间预算，默认 10) */
};

/* sysctl 表：内核侧配置入口 */
static struct ctl_table airy_sysctl_table[] = {
    {
        .procname = "sched_rt_runtime_us",
        .data = &airy_default_cfg.sched_rt_runtime_us,
        .maxlen = sizeof(__u32),
        .mode = 0644,
        .proc_handler = airy_sysctl_handler,
    },
    {
        .procname = "log_min_level",
        .data = &airy_default_cfg.log_min_level,
        .maxlen = sizeof(__u16),
        .mode = 0644,
        .proc_handler = airy_sysctl_handler,
    },
    /* ... 其他 sysctl 项 */
    { }
};
```

### 3.4 用户态 JSON 配置

```json
// /etc/airymax/airymax.conf —— [SS] 语义同源 JSON
{
    "version": 1,
    "sched": {
        "rt_runtime_us": 950000,
        "deadline_period_us": 50000
    },
    "log": {
        "min_level": 6,
        "ring_capacity": 4096
    },
    "ipc": {
        "ring_size": 1024,
        "max_payload": 65536,
        "fastpath_budget_ns": 10
    },
    "superv": {
        "heartbeat_interval": 1,
        "heartbeat_timeout": 30
    },
    "cap": {
        "max_agents": 1024,
        "epoch_bits": 16,
        "randtag_bits": 32,
        "forge_threshold": 100
    }
}
```

---

## §4 热重载机制：RCU 指针原子切换（sysctl 写入触发 config_reload 事件）

### 4.1 RCU 热重载模型

A-UCS 支持 RCU 指针原子切换的热重载机制。sysctl 写入或 JSON 变更触发 `config_reload` 事件，内核通过 `rcu_assign_pointer()` 原子切换配置指针，读者通过 `rcu_dereference()` 获取最新配置，旧配置在宽限期后释放：

```
sysctl 写入 / JSON 变更
       │
       ▼
触发 config_reload 事件
       │
       ▼
构建新配置结构（struct airy_config）
       │
       ▼
rcu_assign_pointer() 原子切换
       │   ├── 新读者立即看到新配置
       └── 旧读者继续用旧配置（直到宽限期结束）
       │
       ▼
synchronize_rcu() 等待宽限期
       │
       ▼
kfree 旧配置
```

### 4.2 内核侧 RCU 热重载

```c
/* kernel/config/airy_sysctl.c —— RCU 热重载 */
static struct airy_config __rcu *airy_cfg_ptr;

/* sysctl 处理器：写入时触发 RCU 热重载 */
static int airy_sysctl_handler(struct ctl_table *tp, int write,
                                 void *buffer, size_t *lenp, loff_t *ppos)
{
    int ret = proc_dointvec(tp, write, buffer, lenp, ppos);
    if (write) {
        /* 1. 构建新配置 */
        struct airy_config *new_cfg = build_config_from_sysctl();
        struct airy_config *old_cfg;

        /* 2. RCU 原子切换指针 */
        old_cfg = rcu_dereference_protected(airy_cfg_ptr,
                                             lockdep_is_held(&airy_cfg_lock));
        rcu_assign_pointer(airy_cfg_ptr, new_cfg);

        /* 3. 等待宽限期后释放旧配置 */
        synchronize_rcu();
        kfree(old_cfg);

        /* 4. 触发 config_reload 事件（通知用户态） */
        airy_eventfd_signal_config_reload();
    }
    return ret;
}

/* 读者获取配置（RCU 保护） */
static inline const struct airy_config *airy_config_get(void)
{
    return rcu_dereference(airy_cfg_ptr);
}
```

### 4.3 用户态 RCU 等效

用户态使用原子指针等效 RCU 机制实现热重载：

```c
/* services/daemons/config_daemon/main.c —— 用户态 RCU 等效 */
static struct airy_config *g_config;  /* 原子指针 */

static void config_reload_from_json(const char *json_path)
{
    /* 1. 解析 JSON 构建新配置 */
    struct airy_config *new_cfg = parse_json_config(json_path);
    if (!new_cfg)
        return;

    /* 2. 校验版本号 */
    if (new_cfg->version != AIRY_CONFIG_VERSION) {
        syslog(LOG_ERR, "airy: config version mismatch %d != %d",
               new_cfg->version, AIRY_CONFIG_VERSION);
        free(new_cfg);
        return;
    }

    /* 3. 原子切换指针 */
    struct airy_config *old = __atomic_exchange_n(&g_config, new_cfg,
                                                   __ATOMIC_SEQ_CST);

    /* 4. 延迟释放旧配置（宽限期等效） */
    /* 用户态无 synchronize_rcu，使用 epoch-based reclamation */
    schedule_deferred_free(old);
}
```

---

## §5 配置版本号：AIRY_CONFIG_VERSION 字段，不匹配时拒绝加载

### 5.1 版本号机制

A-UCS 引入 `AIRY_CONFIG_VERSION` 字段，配置加载时校验版本号，不匹配则拒绝加载并返回 `AIRY_ECFGVERSION`（-101，[SC] 码空间首个值）：

```c
/* kernel/include/airymax/config.h —— 版本号 */
#define AIRY_CONFIG_VERSION  1

/* 版本校验宏 */
#define AIRY_CONFIG_VERSION_CHECK(ver) \
    ((ver) == AIRY_CONFIG_VERSION ? 0 : -AIRY_ECFGVERSION)
```

### 5.2 版本校验流程

| 校验点 | 校验内容 | 失败处理 |
|--------|---------|---------|
| 内核模块加载 | Kconfig 版本 vs `AIRY_CONFIG_VERSION` | 拒绝加载模块 |
| 用户态启动 | JSON version 字段 vs `AIRY_CONFIG_VERSION` | 拒绝启动 + 告警 |
| sysctl 写入 | 新配置版本号 | 拒绝写入 + 返回 `AIRY_ECFGVERSION` |
| 热重载 | 新配置版本号 | 拒绝热重载 + 告警 |

### 5.3 版本演进策略

| 版本变化 | 兼容性 | 处理 |
|---------|--------|------|
| 补丁版本（v1.0 → v1.1） | 向后兼容 | 自动接受 |
| 次版本（v1 → v2） | 可能不兼容 | 拒绝加载，需手动迁移 |
| 主版本（v1 → v2） | 不兼容 | 拒绝加载，需配置迁移工具 |

---

## §6 双向同步：sysctl ↔ JSON 双向同步

### 6.1 双向同步模型

A-UCS 支持 sysctl 与 JSON 的双向同步——任一端的配置变更同步到另一端：

| 方向 | 触发 | 同步机制 |
|------|------|---------|
| sysctl → JSON | sysctl 写入 | config_daemon 监听 config_reload eventfd，读取 sysctl 写入 JSON |
| JSON → sysctl | JSON 文件变更 | config_daemon inotify 监听 JSON 文件，解析后写 sysctl |

### 6.2 sysctl → JSON 同步

```c
/* services/daemons/config_daemon/main.c —— sysctl → JSON 同步 */
static void sync_sysctl_to_json(void)
{
    struct airy_config cfg;

    /* 1. 读取所有 sysctl 值 */
    cfg.sched_rt_runtime_us = read_sysctl("kernel.airy.sched_rt_runtime_us");
    cfg.log_min_level = read_sysctl("kernel.airy.log_min_level");
    cfg.ipc_ring_size = read_sysctl("kernel.airy.ipc_ring_size");
    /* ... */

    /* 2. 写入 JSON 文件（原子写：先写临时文件再 rename） */
    char tmp[256];
    snprintf(tmp, sizeof(tmp), "/etc/airymax/.airymax.conf.tmp.%d", getpid());
    write_json_config(tmp, &cfg);
    rename(tmp, "/etc/airymax/airymax.conf");

    syslog(LOG_INFO, "airy: sysctl -> json synced");
}
```

### 6.3 JSON → sysctl 同步

```c
/* services/daemons/config_daemon/main.c —— JSON → sysctl 同步 */
static void sync_json_to_sysctl(const char *json_path)
{
    /* 1. 解析 JSON */
    struct airy_config *cfg = parse_json_config(json_path);
    if (!cfg)
        return;

    /* 2. 版本校验 */
    if (cfg->version != AIRY_CONFIG_VERSION) {
        syslog(LOG_ERR, "airy: json version mismatch, skip sync");
        return;
    }

    /* 3. 逐项写入 sysctl */
    write_sysctl("kernel.airy.sched_rt_runtime_us", cfg->sched_rt_runtime_us);
    write_sysctl("kernel.airy.log_min_level", cfg->log_min_level);
    write_sysctl("kernel.airy.ipc_ring_size", cfg->ipc_ring_size);
    /* ... 触发 RCU 热重载 */

    syslog(LOG_INFO, "airy: json -> sysctl synced");
}

/* inotify 监听 JSON 文件变更 */
static void watch_json_file(void)
{
    int inotify_fd = inotify_init();
    inotify_add_watch(inotify_fd, "/etc/airymax/", IN_MODIFY | IN_CLOSE_WRITE);
    /* 监听循环... */
}
```

### 6.4 同步冲突处理

| 冲突场景 | 处理策略 |
|---------|---------|
| 同时修改 sysctl 和 JSON | 最后写入者胜（last-write-wins） |
| 版本不匹配 | 拒绝同步，保留当前值 + 告警 |
| 同步失败 | 重试 3 次，仍失败则告警 |

---

## §7 A-IPC Capability Folding 配置项（v1.1 新增）

### 7.1 配置项总览

A-IPC 采用 Capability Folding 单平面架构（详见 [10-unify-design.md §8](../10-architecture/10-unify-design.md)），其运行时参数由 A-UCS 统一管理。本节是 Capability Folding 配置项的 SSoT。

| 配置项 | 类型 | 默认值 | 范围 | 热重载 | 物理宿主 | 说明 |
|--------|------|--------|------|--------|---------|------|
| `AIRY_IPC_HDR_SIZE` | [SC] | 128 | 128（冻结） | 否 | `config.h` | Layout C v4 定长，H1 硬约束 |
| `AIRY_IPC_MAGIC` | [SC] | 0x41524531 | 'ARE1'（冻结） | 否 | `config.h` | magic 值发布即冻结，H1 硬约束 |
| `cap_max_agents` | [SS] | 1024 | 1-1024 | 是 | sysctl + JSON | `agent_caps[]` 静态数组容量，H4 硬约束 |
| `cap_epoch_bits` | [SS] | 16 | 8/16/32 | 否 | sysctl + JSON | Badge Epoch 位宽（1.0.1 可扩展至 32） |
| `cap_randtag_bits` | [SS] | 32 | 16/32 | 否 | sysctl + JSON | Random Tag 位宽（防伪造强度） |
| `cap_forge_threshold` | [SS] | 100 | 10-10000 | 是 | sysctl + JSON | 连续失败次数阈值，触发警报 |
| `ipc_fastpath_budget_ns` | [SS] | 10 | 5-50 | 是 | sysctl + JSON | C-S9 Badge 校验时间预算（ns） |

### 7.2 [SC] 层冻结配置

[SC] 层的 `AIRY_IPC_HDR_SIZE` 与 `AIRY_IPC_MAGIC` 是 Capability Folding 的物理载体定义，**一经发布即冻结**（37 号 §5.2 H1 硬约束）。任何变更需走 [SC] 共享契约变更流程（详见 [08-sc-error-contract.md §6.3](../30-interfaces/08-sc-error-contract.md)）：

1. 在本文件提出变更提案
2. agentrt 与 agentrt-linux 双方维护者评审
3. 同步修改 `kernel/include/airymax/config.h` 物理头文件
4. `sc-dual-ci.yml` 双端校验通过
5. 更新 `AIRY_CONFIG_VERSION`

### 7.3 [SS] 层 Badge 模型配置

Badge 64-bit Native Word 的位宽配置（详见 [03-capability-model.md §2.5](../110-security/03-capability-model.md)）：

```c
/* kernel/include/airymax/config.h —— Badge 位宽配置（[SS] 语义同源） */

/* Badge 64-bit Native Word 布局（默认配置）:
 *   Epoch (16 bits)     [63:48]  全局代际快照
 *   Random Tag (32 bits) [47:16]  per-Agent 随机标签
 *   Perms (16 bits)      [15:0]   权限位图
 *
 * 1.0.1 可扩展路径: Epoch 升级至 32-bit（解决 u16 溢出风险，37 号 §9.1 R6）
 *   Epoch (32 bits)     [63:32]
 *   Random Tag (24 bits) [31:8]
 *   Perms (8 bits)       [7:0]
 */
#define AIRY_BADGE_EPOCH_BITS_DEFAULT    16
#define AIRY_BADGE_RANDTAG_BITS_DEFAULT  32
#define AIRY_BADGE_PERMS_BITS_DEFAULT    16

/* 编译时校验: Epoch + RandomTag + Perms 必须 ≤ 64 bit */
#define AIRY_BADGE_TOTAL_BITS  \
    (AIRY_BADGE_EPOCH_BITS_DEFAULT + \
     AIRY_BADGE_RANDTAG_BITS_DEFAULT + \
     AIRY_BADGE_PERMS_BITS_DEFAULT)
_Static_assert(AIRY_BADGE_TOTAL_BITS <= 64,
               "Badge total bits must not exceed 64-bit Native Word");
```

### 7.4 [SS] 层 agent_caps[] 容量配置

`agent_caps[1024]` 内核静态数组容量配置（H4 硬约束，详见 [03-capability-model.md §2.4](../110-security/03-capability-model.md)）：

```c
/* kernel/security/airy/capability.c —— agent_caps[] 容量（[SS] 语义同源） */

/* 默认容量 1024，可通过 sysctl kernel.airy.cap_max_agents 调整
 * 限制: 0.1.1 阶段 Agent < 100（37 号 §9.1 R8 风险可控）
 * 1.0.1 可扩展: 升级至动态分配 + radix tree 兜底
 */
#define AIRY_CAP_MAX_AGENTS_DEFAULT  1024

/* 静态数组大小 = 1024 × sizeof(struct airy_cap_slot) = 16KB */
extern struct airy_cap_slot agent_caps[AIRY_CAP_MAX_AGENTS_DEFAULT];
```

### 7.5 [SS] 层 fastpath 时间预算

fastpath C-S9 Badge 校验的时间预算配置（详见 [07-ipc-fastpath.md §5.2](../30-interfaces/07-ipc-fastpath.md)）：

```c
/* kernel/ipc/airy_uring_cmd.c —— fastpath 时间预算（[SS] 语义同源） */

/* C-S9 Badge 校验时间预算（默认 10ns）
 * 超过预算的校验会触发 SLOW_PATH（io-wq 接管）
 * 详见 35 号 §4.3 fastpath 7 态状态机
 */
#define AIRY_IPC_FASTPATH_BUDGET_NS_DEFAULT  10

/* fastpath 入口检查 */
static inline bool airy_ipc_fastpath_within_budget(u64 start_ns)
{
    u64 elapsed = ktime_get_ns() - start_ns;
    return elapsed <= airy_config_get()->ipc_fastpath_budget_ns;
}
```

### 7.6 配置变更对 Capability Folding 的影响

| 配置项 | 变更影响 | 兼容性 |
|--------|---------|--------|
| `AIRY_IPC_HDR_SIZE` | 变更破坏 [SC] 双端兼容 | **不可变更**（冻结） |
| `AIRY_IPC_MAGIC` | 变更破坏 [SC] 双端兼容 | **不可变更**（冻结） |
| `cap_max_agents` | 变更需重启系统（静态数组重新分配） | 重启生效 |
| `cap_epoch_bits` | 变更需重新编译 Badge（所有 Badge 失效） | 重启 + Badge 重编译 |
| `cap_randtag_bits` | 变更需重新编译 Badge（防伪造强度变化） | 重启 + Badge 重编译 |
| `cap_forge_threshold` | 实时生效（热重载） | 热重载 |
| `ipc_fastpath_budget_ns` | 实时生效（热重载） | 热重载 |

---

## §8 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §6 —— A-UCS 模块总纲
- [10-unify-design.md](../10-architecture/10-unify-design.md) §8 —— A-IPC 模块总纲（Capability Folding）
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 [SC]/[SS] 分层
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— [SC] 日志配置
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— `AIRY_ESC_CFGVERSION` 错误码
- [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) —— Layout C v4 消息头定义（[SC] 配置载体）
- [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— fastpath C-S9 Badge 校验（时间预算配置）
- [03-capability-model.md](../110-security/03-capability-model.md) —— Badge 64-bit Native Word 模型（位宽配置）
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) —— [DSL] 降级（配置版本不匹配时）

---

## §9 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：A-UCS 统一配置管理体系设计；[SC] 共享配置（二进制布局相关，逐字节相同）；[SS] 语义同源配置（sysctl ↔ JSON 语义映射）；RCU 热重载机制（指针原子切换）；配置版本号（AIRY_CONFIG_VERSION，不匹配拒绝加载）；sysctl ↔ JSON 双向同步 |
| v1.1 | 2026-07-18 | Capability Folding 集成：修复 `AIRY_IPC_HDR_SIZE` 64→128（Layout C v4）；修复 `AIRY_IPC_MAGIC` 0x41495043→0x41524531 'ARE1'（H1 硬约束，发布即冻结）；废弃旧 `cap.cache_ttl`（radix tree cache 模型）；新增 §7 A-IPC Capability Folding 配置项章节（7 项配置：[SC] 冻结 2 项 + [SS] Badge 模型 3 项 + [SS] fastpath 预算 2 项）；新增 Badge 位宽配置 + `_Static_assert`；清除内部审查路径引用 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | A-UCS 统一配置管理体系设计 | v1.1 | 2026-07-18
