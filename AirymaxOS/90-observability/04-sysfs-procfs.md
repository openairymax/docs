Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）sysfs/procfs 接口
> **文档定位**：agentrt-linux（AirymaxOS）可观测性体系 L4 层——静态信息导出接口 sysfs/procfs 的工程规范\
> **文档版本**：v1.0.1\
> **最后更新**：2026-07-18\
> **上级文档**：[90-observability README](README.md)\
> **同源映射**：agentrt E-2 可观测性 + Linux 6.6 sysfs/procfs/kobject\
> **理论根基**：Linux 6.6 内核基线 + Airymax 五维正交 24 原则 + sched_tac 调度\
> **核心约束**：sysfs/procfs 仅导出只读状态，写操作必须经 A-UCS 模块 config_d 校验

---

## 目录

- [第 1 章 sysfs/procfs 框架概述](#第-1-章-sysfsprocfs-框架概述)
- [第 2 章 /sys/airy/ 目录结构设计](#第-2-章-sysairy-目录结构设计)
- [第 3 章 /proc/airy/ 目录结构设计](#第-3-章-procairy-目录结构设计)
- [第 4 章 Agent 状态查询接口](#第-4-章-agent-状态查询接口)
- [第 5 章 调度器状态接口](#第-5-章-调度器状态接口)
- [第 6 章 IPC 统计接口](#第-6-章-ipc-统计接口)
- [第 7 章 12 daemon 状态接口](#第-7-章-12-daemon-状态接口)
- [第 8 章 A-ULP 日志统计接口](#第-8-章-ulps-日志统计接口)
- [第 9 章 A-ULS 监管状态接口](#第-9-章-usv-监管状态接口)
- [第 10 章 Airymax Unify Design 映射](#第-10-章-airymax-unify-design-映射)
- [第 11 章 相关文档与版本维护](#第-11-章-相关文档与版本维护)

---

## 第 1 章 sysfs/procfs 框架概述

### 1.1 定位

sysfs 与 procfs 是 Linux 6.6 内核基线提供的标准伪文件系统，分别承载设备/内核子系统状态（sysfs）与进程/系统状态（procfs）的静态导出。agentrt-linux 选择 sysfs/procfs 作为可观测性 L4 层，原因有三：

1. **标准化访问语义**：运维人员与脚本工具天然熟悉 `cat /sys/...` 与 `cat /proc/...` 访问语义，无需引入新的客户端工具，符合"运维零学习成本"原则。
2. **与 kobject/sysctl 集成**：sysfs 通过 `kobject` 与 `attribute` 自动管理生命周期；procfs 通过 `proc_ops` 注册回调，与 A-UCS 模块 config_d 的 sysctl/JSON 热重载天然协同。
3. **零运行时开销**：sysfs/procfs 文件仅在 `read()` 时调用 `seq_show` 回调动态生成内容，不维护常驻缓冲区，对稳态运行无影响，符合 A-ULS 模块 macro_d 的"可观测性不破坏系统稳态"原则。

**OS-OBS-041: sysfs 与 procfs 是 agentrt-linux 可观测性 L4 层的强制基线，所有静态状态导出必须经 sysfs/procfs，不得自创 /sys/kernel/airy 或 /proc/airy_v2 等并行目录。**

**OS-KER-131: kernel 的 defconfig 必须开启 CONFIG_SYSFS、CONFIG_PROC_FS、CONFIG_KOBJECT；agentrt.ko 必须在 module_init 阶段完成 airy_kset 注册。**

### 1.2 框架组成

| 组件 | 实现位置 | 职责 |
|------|----------|------|
| sysfs | `fs/sysfs/` | 设备/子系统状态导出 |
| procfs | `fs/proc/` | 进程/系统状态导出 |
| kobject | `lib/kobject.c` | sysfs 对象生命周期 |
| kset | `lib/kobject.c` | kobject 集合管理 |
| attribute | `include/linux/sysfs.h` | sysfs 文件属性 |
| seq_file | `fs/seq_file.c` | 序列化输出框架 |
| proc_ops | `include/linux/proc_fs.h` | procfs 文件操作 |

**OS-STD-031: 任何 sysfs/procfs 文件的 `show()` 回调必须使用 `seq_file` 框架，不得直接使用 `sprintf` 拼接缓冲区，避免缓冲区溢出与并发问题。**

### 1.3 sysfs vs procfs 职责划分

agentrt-linux 遵循上游 Linux 约定的职责划分：

| 文件系统 | 职责 | agentrt-linux 在此导出的内容 |
|---------|------|----------------------------|
| sysfs | 设备/子系统级状态（全局唯一） | 调度器、IPC、daemons、A-ULP、A-ULS 全局统计 |
| procfs | 进程级状态（per-PID 或 per-Agent） | Agent 实例状态、Token 统计、记忆统计 |

---

## 第 2 章 /sys/airy/ 目录结构设计

### 2.1 顶层结构

```
/sys/airy/
├── sched/                  # 调度器状态
│   ├── stats               # 调度器全局统计
│   ├── class_stats         # 三层调度类统计
│   └── agents/             # 调度器视角的 Agent 列表
│       └── [id]/
│           ├── policy      # 当前调度策略
│           ├── runtime     # 累计运行时间
│           └── deadline    # SCHED_DEADLINE 参数
├── ipc/                    # IPC 子系统状态
│   ├── stats               # IPC 全局统计
│   ├── rings/              # io_uring ring 列表
│   │   └── [id]/
│   │       ├── depth       # ring 深度
│   │       ├── inflight    # 在途请求数
│   │       └── errors      # 错误计数
│   └── capabilities        # IPC capability 总表
├── daemons/                # 12 daemon 状态
│   └── [name]/             # macro_d / logger_d / ...
│       ├── status          # running / stopped / faulted
│       ├── pid             # daemon PID
│       ├── uptime_ms       # 累计运行时间
│       └── restart_count   # 重启次数
├── log/                    # A-ULP 日志子系统状态
│   ├── stats               # A-ULP 全局统计
│   ├── ring_buffer/        # Ring Buffer 状态
│   │   ├── size_kb         # 缓冲大小
│   │   ├── used_kb         # 已用大小
│   │   ├── overrun         # 溢出计数
│   │   └── write_rate      # 写入速率
│   └── logger/             # logger_d 状态
│       ├── consume_rate    # 消费速率
│       ├── flush_latency   # 落盘延迟
│       └── panic_fallback  # Panic 回退状态
├── superv/                 # A-ULS 监管子系统状态
│   ├── stats               # A-ULS 全局统计
│   ├── micro/              # Micro-Supervisor 状态
│   │   ├── enforcements    # 冷酷执法计数
│   │   └── last_action     # 最近一次执法
│   └── macro/              # Macro-Supervisor 状态
│       ├── adjudications   # 温情裁决计数
│       └── budget_alerts   # 预算告警计数
├── mem/                    # 内存子系统状态
│   ├── stats               # 内存全局统计
│   └── pools/              # 内存池状态
│       └── [name]/
│           ├── pages       # 已分配页数
│           └── mappings    # mmap 映射数
├── sec/                    # 安全子系统状态
│   ├── stats               # 安全全局统计
│   └── lsm/                # airy_lsm 状态
│       ├── hook_count      # 注册钩子数
│       ├── cap_checks      # capability 校验计数
│       └── denials         # 拒绝计数
└── version                 # agentrt.ko 版本号
```

### 2.2 kset 注册流程

agentrt.ko 在 `airy_init()` 中注册顶层 kset `airy_kset`：

```c
/* kernel/agentrt/airy_sysfs.c */
struct kset *airy_kset;

static int __init airy_sysfs_init(void)
{
    int ret;
    airy_kset = kset_create_and_add("airy", NULL, kernel_kobj);
    if (!airy_kset)
        return -ENOMEM;
    ret = airy_sched_sysfs_init(airy_kset);
    if (ret) goto err_sched;
    ret = airy_ipc_sysfs_init(airy_kset);
    if (ret) goto err_ipc;
    ret = airy_daemons_sysfs_init(airy_kset);
    if (ret) goto err_daemons;
    ret = airy_log_sysfs_init(airy_kset);
    if (ret) goto err_log;
    ret = airy_superv_sysfs_init(airy_kset);
    if (ret) goto err_superv;
    return 0;
    /* 错误回滚 */
}
```

**OS-KER-132: agentrt.ko 必须在 module_init 阶段完成 `/sys/airy/` 顶层 kset 注册；注册失败视为致命错误，agentrt.ko 加载中止。**

### 2.3 attribute 定义规范

每个 sysfs 文件通过 `kset_attribute` 或 `kobj_attribute` 定义 `show`/`store` 回调：

```c
/* 调度器全局统计 attribute */
static ssize_t sched_stats_show(struct kobject *kobj,
    struct kobj_attribute *attr, char *buf)
{
    struct airy_sched_stats stats;
    airy_sched_get_stats(&stats);
    return scnprintf(buf, PAGE_SIZE,
        "switches_total: %llu\n"
        "switches_dl_to_rt: %llu\n"
        "switches_rt_to_eevdf: %llu\n"
        "switches_eevdf_to_dl: %llu\n"
        "avg_latency_ns: %llu\n"
        "p99_latency_ns: %llu\n",
        stats.switches_total,
        stats.switches_dl_to_rt,
        stats.switches_rt_to_eevdf,
        stats.switches_eevdf_to_dl,
        stats.avg_latency_ns,
        stats.p99_latency_ns);
}

static struct kobj_attribute sched_stats_attr =
    __ATTR(stats, 0444, sched_stats_show, NULL);
```

**OS-STD-032: 所有统计类 sysfs 文件权限必须为 0444（只读），不得允许用户态直接写入；状态变更必须经 A-UCS config_d 的 sysctl/JSON 接口。**

---

## 第 3 章 /proc/airy/ 目录结构设计

### 3.1 顶层结构

```
/proc/airy/
├── agents/                 # Agent 实例列表
│   └── [id]/               # Agent ID
│       ├── status          # Agent 当前状态（8 态之一）
│       ├── cmdline         # Agent 启动命令
│       ├── sched           # 调度信息
│       ├── ipc             # IPC 信息
│       ├── token_stats     # Token 消耗统计
│       ├── memory_stats    # 记忆卷载统计
│       ├── call_graph      # Agent 调用图
│       └── trace           # Agent 行为追踪输出
├── daemons/                # 12 daemon 进程列表
│   └── [name]/
│       ├── status          # daemon 运行状态
│       ├── pid             # daemon PID
│       └── fdinfo/         # daemon 文件描述符信息
├── config/                 # A-UCS 配置快照
│   ├── version             # 配置版本号 AIRY_CONFIG_VERSION
│   ├── loaded              # 已加载配置项数
│   └── pending             # 待加载配置项数
└── fault_log               # A-UEF 错误码日志（最近 1024 条）
```

### 3.2 proc_ops 注册流程

agentrt.ko 在 `airy_init()` 中通过 `proc_mkdir` 创建目录：

```c
/* kernel/agentrt/airy_procfs.c */
static struct proc_dir_entry *airy_proc_dir;

static int __init airy_procfs_init(void)
{
    airy_proc_dir = proc_mkdir("airy", NULL);
    if (!airy_proc_dir)
        return -ENOMEM;
    proc_mkdir("agents", airy_proc_dir);
    proc_mkdir("daemons", airy_proc_dir);
    proc_mkdir("config", airy_proc_dir);
    proc_create("fault_log", 0444, airy_proc_dir, &fault_log_ops);
    return 0;
}
```

**OS-KER-133: `/proc/airy/` 目录及其子目录的创建必须在 agentrt.ko init 阶段完成；任一子目录创建失败视为致命错误。**

---

## 第 4 章 Agent 状态查询接口

### 4.1 /proc/airy/agents/[id]/status

每个 Agent 实例拥有独立子目录 `/proc/airy/agents/[id]/`。status 文件导出 Agent 当前 8 态生命周期：

```bash
$ cat /proc/airy/agents/42/status
agent_id: 42
name: cogn_d_worker
state: COGNITION_RUNNING       # 8 态之一
state_since: 1784328868912     # 进入当前状态的时间戳（ns）
sched_class: SCHED_DEADLINE    # 当前调度类
priority: 99                   # 优先级（SCHED_FIFO/RR）
runtime_ms: 12345              # 累计运行时间
cpu_affinity: 0-3              # CPU 亲和性
parent_id: 1                   # 父 Agent ID
```

Agent 8 态定义：

| 状态 | 含义 |
|------|------|
| `CREATED` | 已创建未启动 |
| `COGNITION_RUNNING` | 认知循环运行中 |
| `PLANNING` | 规划中（DAG 更新） |
| `SCHEDULING` | 调度中 |
| `EXECUTING` | 执行工具调用 |
| `BLOCKED` | 阻塞等待 IPC/IO |
| `SUSPENDED` | 被 macro_d 挂起 |
| `TERMINATED` | 已终止 |

### 4.2 /proc/airy/agents/[id]/sched

调度器视角的 Agent 信息：

```bash
$ cat /proc/airy/agents/42/sched
policy: SCHED_DEADLINE
runtime_ns: 5000000           # SCHED_DEADLINE runtime 参数
deadline_ns: 10000000         # SCHED_DEADLINE deadline 参数
period_ns: 10000000           # SCHED_DEADLINE period 参数
switches: 5678                # 累计切换次数
avg_latency_ns: 8234          # 平均调度延迟
p99_latency_ns: 78123         # P99 调度延迟
max_latency_ns: 156789        # 最大调度延迟
vruntime: 12345678            # EEVDF vruntime（仅 EEVDF 类）
```

### 4.3 /proc/airy/agents/[id]/ipc

Agent 的 IPC 统计：

```bash
$ cat /proc/airy/agents/42/ipc
sends_total: 123456           # 累计发送消息数
sends_fastpath: 123000        # fastpath 发送数
sends_slowpath: 456           # slowpath 发送数
recv_total: 123400            # 累计接收消息数
avg_latency_ns: 1234          # 平均 IPC 延迟
p99_latency_ns: 4567          # P99 IPC 延迟
errors: 0                     # 错误计数
```

### 4.4 动态创建与销毁

Agent 子目录在 Agent 创建时动态生成，销毁时移除：

```c
/* Agent 创建时 */
struct proc_dir_entry *agent_dir;
agent_dir = proc_mkdir("42", agents_dir);
proc_create("status", 0444, agent_dir, &agent_status_ops);
proc_create("sched", 0444, agent_dir, &agent_sched_ops);
proc_create("ipc", 0444, agent_dir, &agent_ipc_ops);

/* Agent 销毁时 */
remove_proc_subtree("42", agents_dir);
```

**OS-OBS-042: Agent 子目录的创建与销毁必须由 sched_d daemon 通过 ioctl 触发，不得由 Agent 自身创建自己的目录，避免 Agent 伪造状态信息。**

---

## 第 5 章 调度器状态接口

### 5.1 /sys/airy/sched/stats

调度器全局统计：

```bash
$ cat /sys/airy/sched/stats
switches_total: 12345678
switches_dl_to_rt: 4321
switches_rt_to_eevdf: 5678
switches_eevdf_to_dl: 2346
switches_within_class: 12333233
avg_latency_ns: 12345
p99_latency_ns: 78123
max_latency_ns: 567890
agents_active: 42
agents_dl: 5                  # SCHED_DEADLINE Agent 数
agents_rt: 12                 # SCHED_FIFO Agent 数
agents_eevdf: 25              # EEVDF Agent 数
```

### 5.2 /sys/airy/sched/class_stats

三层调度类分别统计：

```bash
$ cat /sys/airy/sched/class_stats
class: SCHED_DEADLINE
  agents: 5
  switches_in: 2346
  switches_out: 4321
  avg_runtime_ns: 5000000
  p99_latency_ns: 78000
  deadline_misses: 0
class: SCHED_FIFO
  agents: 12
  switches_in: 4321
  switches_out: 5678
  avg_runtime_ns: 12000000
  p99_latency_ns: 287000
class: EEVDF
  agents: 25
  switches_in: 5678
  switches_out: 2346
  avg_runtime_ns: 45000000
  p99_latency_ns: 1230000
```

**OS-OBS-043: SCHED_DEADLINE 的 `deadline_misses` 必须为 0；非零值视为sched_tac 失效，需触发 macro_d 告警并降级 Agent 至 SCHED_FIFO。**

---

## 第 6 章 IPC 统计接口

### 6.1 /sys/airy/ipc/stats

IPC 子系统全局统计：

```bash
$ cat /sys/airy/ipc/stats
rings_active: 8               # 活跃 io_uring ring 数
sends_total: 1234567890
sends_fastpath: 1230000000    # IORING_OP_URING_CMD 路径
sends_slowpath: 4567890       # 普通 write/read 路径
fastpath_ratio: 99.63         # fastpath 占比
recv_total: 1234500000
avg_latency_ns: 1200
p99_latency_ns: 4500
max_latency_ns: 12300
errors: 0
drops: 0                      # ring buffer 满丢弃
```

### 6.2 /sys/airy/ipc/rings/[id]/

每个 io_uring ring 的详细状态：

```bash
$ cat /sys/airy/ipc/rings/0/depth
256                           # SQ/CQ 深度

$ cat /sys/airy/ipc/rings/0/inflight
12                            # 在途请求数

$ cat /sys/airy/ipc/rings/0/errors
0                             # 错误计数
```

### 6.3 IPC capability 总表

```bash
$ cat /sys/airy/ipc/capabilities
agent_id: 1  caps: IPC_SEND|IPC_RECV|IPC_BROADCAST
agent_id: 42 caps: IPC_SEND|IPC_RECV
agent_id: 100 caps: IPC_RECV
```

**OS-OBS-044: IPC fastpath 占比必须 ≥ 95%；低于 95% 需审查 io_uring SQE 批处理策略是否生效。**

---

## 第 7 章 12 daemon 状态接口

### 7.1 /sys/airy/daemons/[name]/status

每个 daemon 的状态：

```bash
$ cat /sys/airy/daemons/macro_d/status
name: macro_d
state: RUNNING                # RUNNING / STOPPED / FAULTED
pid: 1234
uptime_ms: 123456789
restart_count: 0
cpu_percent: 4.5
memory_kb: 12345
last_heartbeat_ms: 100        # 最近心跳距今（ms）
```

### 7.2 12 daemon 状态总表

```bash
$ ls /sys/airy/daemons/
macro_d/  logger_d/  config_d/  gateway_d/
sched_d/  vfs_d/  net_d/  mem_d/  cogn_d/  sec_d/  audit_d/  dev_d/

$ for d in /sys/airy/daemons/*/status; do
    echo "=== $d ==="; cat $d | head -3
  done
```

### 7.3 daemon 健康检查

A-ULS 模块 macro_d 定期读取所有 daemon 状态：

```bash
# 健康检查脚本示例
for d in macro_d logger_d config_d gateway_d \
         sched_d vfs_d net_d mem_d cogn_d sec_d audit_d dev_d; do
    state=$(cat /sys/airy/daemons/$d/state)
    if [ "$state" != "RUNNING" ]; then
        logger -t airy-superv "daemon $d in state $state"
    fi
    hb=$(cat /sys/airy/daemons/$d/last_heartbeat_ms)
    if [ "$hb" -gt 5000 ]; then
        logger -t airy-superv "daemon $d heartbeat stale: ${hb}ms"
    fi
done
```

**OS-OBS-045: 12 daemon 的 `last_heartbeat_ms` 必须始终 ≤ 5000ms；超限视为 daemon 卡死，触发 macro_d 重启。**

### 7.4 各 daemon 独有统计

| Daemon | 独有统计文件 | 内容 |
|--------|------------|------|
| macro_d | `/enforcements` | 冷酷执法与温情裁决计数 |
| logger_d | `/consume_rate` | 日志消费速率 |
| config_d | `/reload_count` | 配置热重载次数 |
| gateway_d | `/ring_count` | 活跃 io_uring ring 数 |
| sched_d | `/class_switches` | 调度类切换计数 |
| vfs_d | `/mount_count` | 挂载点数 |
| net_d | `/socket_count` | 活跃 socket 数 |
| mem_d | `/pool_count` | 内存池数 |
| cogn_d | `/cognition_cycles` | 认知循环次数 |
| sec_d | `/cap_denials` | capability 拒绝计数 |
| audit_d | `/audit_records` | 审计记录数 |
| dev_d | `/device_events` | 设备事件数 |

---

## 第 8 章 A-ULP 日志统计接口

### 8.1 /sys/airy/log/stats

A-ULP 子系统全局统计：

```bash
$ cat /sys/airy/log/stats
ring_buffer_size_kb: 8192
ring_buffer_used_kb: 4231
ring_buffer_overrun: 0        # 溢出计数
write_rate: 1234567           # 写入速率（rec/s）
write_rate_peak: 2345678      # 峰值写入速率
logger_consume_rate: 1234000  # 消费速率
logger_flush_latency_ns: 1234 # 落盘延迟
panic_fallback_active: 0      # Panic 回退状态（0=正常）
panic_fallback_count: 0       # Panic 回退触发次数
```

### 8.2 Ring Buffer 详细状态

```bash
$ cat /sys/airy/log/ring_buffer/size_kb
8192

$ cat /sys/airy/log/ring_buffer/used_kb
4231

$ cat /sys/airy/log/ring_buffer/overrun
0

$ cat /sys/airy/log/ring_buffer/write_rate
1234567
```

### 8.3 logger_d 状态

```bash
$ cat /sys/airy/log/logger/consume_rate
1234000

$ cat /sys/airy/log/logger/flush_latency
1234

$ cat /sys/airy/log/logger/panic_fallback
inactive
```

### 8.4 日志级别分布

```bash
$ cat /sys/airy/log/level_distribution
EMERG: 0
ALERT: 0
CRIT: 0
ERR: 12
WARNING: 234
NOTICE: 1234
INFO: 1234567
DEBUG: 0                       # 生产环境关闭 DEBUG
```

**OS-OBS-046: Ring Buffer `overrun` 必须为 0；非零值视为日志丢失，需扩容 Ring Buffer 或审查 logger_d 消费速率。**

---

## 第 9 章 A-ULS 监管状态接口

### 9.1 /sys/airy/superv/stats

A-ULS 子系统全局统计：

```bash
$ cat /sys/airy/superv/stats
micro_enforcements: 123        # Micro-Supervisor 冷酷执法次数
macro_adjudications: 45        # Macro-Supervisor 温情裁决次数
budget_alerts: 12              # 预算告警次数
agents_suspended: 2            # 当前挂起 Agent 数
agents_terminated: 1           # 终止 Agent 数
```

### 9.2 Micro-Supervisor 状态

```bash
$ cat /sys/airy/superv/micro/enforcements
123

$ cat /sys/airy/superv/micro/last_action
timestamp: 1784328868912
agent_id: 42
action: SUSPEND
reason: TOKEN_BUDGET_EXCEEDED
```

### 9.3 Macro-Supervisor 状态

```bash
$ cat /sys/airy/superv/macro/adjudications
45

$ cat /sys/airy/superv/macro/budget_alerts
12

$ cat /sys/airy/superv/macro/last_adjudication
timestamp: 1784328868912
agent_id: 42
action: ESCALATE
reason: REPEATED_FAULT
detail: AIRY_FAULT_TIMEOUT x3
```

### 9.4 预算监控

```bash
$ cat /sys/airy/superv/budgets/42
agent_id: 42
token_budget: 1000000
token_used: 987654
token_remaining: 12346
budget_ratio: 98.76            # 已用百分比
alert_threshold: 90.0          # 告警阈值
action_threshold: 100.0        # 执法阈值
state: ALERT                   # NORMAL / ALERT / EXCEEDED
```

**OS-OBS-047: A-ULS 监管状态必须实时反映当前快照；`cat /sys/airy/superv/*` 不得返回过期数据（>1s）。**

---

## 第 10 章 Airymax Unify Design 映射

### 10.1 五模块关系

| Unify 模块 | 关系 | 在 sysfs/procfs 的体现 |
|-----------|------|----------------------|
| **A-UEF** | **核心** | `/proc/airy/fault_log` 导出最近 1024 条 `AIRY_FAULT_*` 错误码 |
| **A-ULP** | **核心** | `/sys/airy/log/*` 导出 A-ULP Ring Buffer 与 logger_d 状态 |
| **A-UCS** | **核心** | `/proc/airy/config/*` 导出 A-UCS 配置版本与加载状态 |
| **A-ULS** | **核心** | `/sys/airy/superv/*` 导出 A-ULS Micro/Macro 监管状态 |
| **A-IPC** | 辅助 | `/sys/airy/ipc/*` 导出 A-IPC IPC 统计 |

### 10.2 sysfs/procfs 与 12 daemon 映射

| Daemon | sysfs 路径 | procfs 路径 |
|--------|-----------|------------|
| macro_d | `/sys/airy/superv/macro/` | `/proc/airy/daemons/macro_d/` |
| logger_d | `/sys/airy/log/logger/` | `/proc/airy/daemons/logger_d/` |
| config_d | `/sys/airy/config/` | `/proc/airy/daemons/config_d/` |
| gateway_d | `/sys/airy/ipc/` | `/proc/airy/daemons/gateway_d/` |
| sched_d | `/sys/airy/sched/` | `/proc/airy/daemons/sched_d/` |
| vfs_d | `/sys/airy/vfs/` | `/proc/airy/daemons/vfs_d/` |
| net_d | `/sys/airy/net/` | `/proc/airy/daemons/net_d/` |
| mem_d | `/sys/airy/mem/` | `/proc/airy/daemons/mem_d/` |
| cogn_d | （无独立 sysfs）| `/proc/airy/daemons/cogn_d/` |
| sec_d | `/sys/airy/sec/` | `/proc/airy/daemons/sec_d/` |
| audit_d | `/sys/airy/audit/` | `/proc/airy/daemons/audit_d/` |
| dev_d | `/sys/airy/dev/` | `/proc/airy/daemons/dev_d/` |

---

## 第 11 章 相关文档与版本维护

### 11.1 相关文档

- [90-observability README](README.md)：可观测性体系主索引
- [01-ftrace-framework.md](01-ftrace-framework.md)：ftrace 框架（L1 层）
- [02-ebpf-probes.md](02-ebpf-probes.md)：eBPF 探针（L2 层）
- [03-perf-analysis.md](03-perf-analysis.md)：perf 性能分析（L3 层）
- [05-debugfs-tracefs.md](05-debugfs-tracefs.md)：debugfs/tracefs 接口
- [07-token-efficiency.md](07-token-efficiency.md)：Token 效率监控
- [09-memory-monitoring.md](09-memory-monitoring.md)：记忆监控

### 11.2 参考材料

- Linux 6.6 `Documentation/filesystems/sysfs.rst`（sysfs 文档）
- Linux 6.6 `Documentation/filesystems/proc.rst`（procfs 文档）
- Linux 6.6 `fs/sysfs/`（sysfs 实现）
- Linux 6.6 `fs/proc/`（procfs 实现）
- Linux 6.6 `include/linux/kobject.h`（kobject API）

### 11.3 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0.1 | 2026-07-18 | 初始版本：/sys/airy/ 与 /proc/airy/ 目录结构设计、Agent 状态查询、调度器/IPC/12 daemon/A-ULP/A-ULS 统计接口、kset/proc_ops 注册规范 |

---

> **文档结束** | agentrt-linux sysfs/procfs 接口 v1.0.1 | 维护者：开源极境工程与规范委员会 | "The interface is the system."
