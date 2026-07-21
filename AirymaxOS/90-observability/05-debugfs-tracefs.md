Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）debugfs/tracefs 接口
> **文档定位**：agentrt-linux（AirymaxOS）可观测性体系 L4 层——调试与追踪接口 debugfs/tracefs 的工程规范\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[90-observability README](README.md)\
> **同源映射**：agentrt E-2 可观测性 + Linux 6.6 debugfs/tracefs/ftrace\
> **理论根基**：Linux 6.6 内核基线 + Airymax 五维正交 24 原则 + sched_tac 调度\
> **核心约束**：debugfs 仅在调试构建中挂载；tracefs 通过 events 子系统集成静态 tracepoint

---

## 目录

- [第 1 章 debugfs/tracefs 框架概述](#第-1-章-debugfstracefs-框架概述)
- [第 2 章 /sys/kernel/debug/airy/ 目录设计](#第-2-章-syskerneldebugairy-目录设计)
- [第 3 章 Agent 详细调试信息](#第-3-章-agent-详细调试信息)
- [第 4 章 调度器调试接口](#第-4-章-调度器调试接口)
- [第 5 章 IPC 调试接口](#第-5-章-ipc-调试接口)
- [第 6 章 tracefs 集成](#第-6-章-tracefs-集成)
- [第 7 章 agentrt-linux 事件追踪](#第-7-章-agentrt-linux-事件追踪)
- [第 8 章 与 ftrace 框架的关系](#第-8-章-与-ftrace-框架的关系)
- [第 9 章 Airymax Unify Design 映射](#第-9-章-airymax-unify-design-映射)
- [第 10 章 相关文档与版本维护](#第-10-章-相关文档与版本维护)

---

## 第 1 章 debugfs/tracefs 框架概述

### 1.1 定位

debugfs 与 tracefs 是 Linux 6.6 内核基线提供的调试与追踪专用伪文件系统。debugfs 用于内核调试信息的临时导出（无稳定 ABI 保证），tracefs 用于 ftrace 与 tracepoint 事件的配置与输出。agentrt-linux 选择 debugfs/tracefs 作为可观测性 L4 层的调试补充，原因有三：

1. **调试场景专用**：debugfs 文件无 ABI 稳定性约束，可在不同版本间自由调整格式，适合调试期频繁迭代；与 sysfs 的"稳定 ABI"职责严格分离。
2. **与 ftrace 原生集成**：tracefs 是 ftrace 的标准接口，所有 tracepoint 事件通过 `events/` 子目录暴露，无需自创追踪文件系统。
3. **生产可关闭**：debugfs 默认不挂载，仅在调试构建或 `airy_debug=1` 启动参数下挂载，符合 A-ULS 模块 macro_d 的"生产环境最小调试面"原则。

**OS-OBS-051: debugfs 与 tracefs 是 agentrt-linux 可观测性 L4 层调试接口的强制基线，调试信息必须经 debugfs 导出，不得在生产 sysfs 中混入调试字段。**

**OS-KER-141: kernel 的 defconfig 必须开启 CONFIG_DEBUG_FS、CONFIG_TRACING、CONFIG_EVENT_TRACING；agentrt.ko 必须在 module_init 阶段完成 debugfs 子目录注册（仅当 debugfs 已挂载时）。**

### 1.2 框架组成

| 组件 | 实现位置 | 职责 |
|------|----------|------|
| debugfs | `fs/debugfs/inode.c` | 调试文件系统 |
| tracefs | `fs/tracefs/inode.c` | 追踪文件系统 |
| ftrace 核心 | `kernel/trace/trace.c` | 追踪引擎 |
| trace_events | `kernel/trace/trace_events.c` | 事件子系统 |
| trace_export | `kernel/trace/trace_export.c` | 事件导出 |
| DECLARE_TRACE | `include/trace/define_trace.h` | tracepoint 宏 |

**OS-STD-041: 任何 debugfs 文件必须在 `airy_debugfs_exit()` 中正确注销；不得依赖 module 卸载自动清理，避免悬挂指针。**

### 1.3 debugfs vs tracefs vs sysfs 职责划分

| 文件系统 | 职责 | ABI 稳定性 | 生产环境 |
|---------|------|-----------|---------|
| sysfs | 稳定状态导出 | 稳定 | 必须挂载 |
| procfs | 进程状态导出 | 稳定 | 必须挂载 |
| debugfs | 调试信息导出 | 不稳定 | 默认不挂载 |
| tracefs | 追踪事件配置与输出 | 稳定（事件格式） | 必须挂载 |

---

## 第 2 章 /sys/kernel/debug/airy/ 目录设计

### 2.1 顶层结构

```
/sys/kernel/debug/airy/
├── agents/                  # Agent 详细调试信息
│   └── [id]/
│       ├── state_machine    # 状态机当前状态与历史
│       ├── decision_log     # 最近 100 条决策记录
│       ├── memory_layout    # 记忆卷载内存布局
│       └── ipc_trace        # 最近 100 条 IPC 交互
├── sched/                   # 调度器调试
│   ├── runqueue_dump        # 当前运行队列快照
│   ├── class_internals      # 三层调度类内部状态
│   ├── deadline_tree        # SCHED_DEADLINE 红黑树
│   └── eevdf_vruntime       # EEVDF vruntime 分布
├── ipc/                     # IPC 调试
│   ├── ring_dump            # io_uring ring 内容快照
│   ├── sqe_history          # 最近 100 条 SQE
│   ├── cqe_history          # 最近 100 条 CQE
│   └── cap_table            # capability 表完整快照
├── log/                     # A-ULP 调试
│   ├── ring_buffer_dump     # Ring Buffer 原始内容
│   ├── logger_state         # logger_d 内部状态
│   └── panic_fallback_log   # Panic 回退日志
├── superv/                  # A-ULS 调试
│   ├── micro_decisions      # Micro-Supervisor 决策历史
│   ├── macro_decisions      # Macro-Supervisor 决策历史
│   └── budget_snapshots     # 预算快照
├── mem/                     # 内存调试
│   ├── page_alloc_log       # 最近 1000 次 alloc_pages
│   ├── mmap_log             # 最近 100 次 mmap
│   └── pool_layout          # 内存池布局
├── sec/                     # 安全调试
│   ├── lsm_hook_log         # 最近 1000 次 LSM 钩子调用
│   ├── cap_check_log        # 最近 1000 次 capability 校验
│   └── denial_details       # 拒绝详情
└── internals/               # agentrt.ko 内部状态
    ├── kobject_tree         # kobject 树
    ├── refcount_audit       # 引用计数审计
    └── leak_detector        # 内存泄漏检测器
```

### 2.2 debugfs 注册流程

agentrt.ko 在 `airy_init()` 中检查 debugfs 是否挂载，若挂载则注册子目录：

```c
/* kernel/agentrt/airy_debugfs.c */
static struct dentry *airy_debugfs_root;

static int __init airy_debugfs_init(void)
{
    struct dentry *root;
    /* 仅当 debugfs 已挂载时注册 */
    if (!dbg_fs_root)
        return 0;  /* 非错误，生产环境 debugfs 不挂载 */
    root = debugfs_create_dir("airy", NULL);
    if (IS_ERR(root))
        return PTR_ERR(root);
    airy_debugfs_root = root;
    airy_agents_debugfs_init(root);
    airy_sched_debugfs_init(root);
    airy_ipc_debugfs_init(root);
    airy_log_debugfs_init(root);
    airy_superv_debugfs_init(root);
    airy_mem_debugfs_init(root);
    airy_sec_debugfs_init(root);
    airy_internals_debugfs_init(root);
    return 0;
}

static void __exit airy_debugfs_exit(void)
{
    debugfs_remove_recursive(airy_debugfs_root);
    airy_debugfs_root = NULL;
}
```

**OS-KER-142: agentrt.ko 必须在 `airy_debugfs_init()` 中检查 `dbg_fs_root` 是否为 NULL；若 debugfs 未挂载，跳过注册而非报错，保证生产构建可正常加载。**

### 2.3 debugfs 文件创建规范

debugfs 文件通过 `debugfs_create_*` 系列 API 创建：

```c
/* 创建只读 0444 文件 */
debugfs_create_u64("switches_total", 0444, sched_dir,
                   &airy_sched_stats.switches_total);

/* 创建 seq_file 文件（复杂格式） */
debugfs_create_file("runqueue_dump", 0444, sched_dir, NULL,
                    &runqueue_dump_fops);

/* 创建 blob（二进制数据） */
debugfs_create_blob("ring_buffer_dump", 0444, log_dir, &ring_blob);
```

**OS-STD-042: 所有 debugfs 文件权限必须为 0444（只读）；不得在 debugfs 中提供写接口，状态变更必须经 A-UCS config_d 的标准接口。**

---

## 第 3 章 Agent 详细调试信息

### 3.1 /sys/kernel/debug/airy/agents/[id]/state_machine

Agent 状态机的完整历史：

```bash
$ cat /sys/kernel/debug/airy/agents/42/state_machine
agent_id: 42
current_state: COGNITION_RUNNING
state_history (last 20):
  [1784328868912] CREATED → COGNITION_RUNNING
  [1784328958912] COGNITION_RUNNING → PLANNING
  [1784328968912] PLANNING → SCHEDULING
  [1784328978912] SCHEDULING → EXECUTING
  [1784328988912] EXECUTING → BLOCKED
  [1784328998912] BLOCKED → COGNITION_RUNNING
  ...
transitions_total: 5678
avg_state_duration_ms: 12.3
```

### 3.2 /sys/kernel/debug/airy/agents/[id]/decision_log

最近 100 条决策记录：

```bash
$ cat /sys/kernel/debug/airy/agents/42/decision_log | head -5
[1784328868912] cognition: input="user_query_42" confidence=0.87
[1784328868922] planner: action=TOOL_CALL tool="search" args="..."
[1784328868932] scheduler: target=agent_43 priority=99
[1784328868942] execution: tool="search" latency=10234ns result=OK
[1784328868952] cognition: output="search_result_summary" tokens=234
```

### 3.3 /sys/kernel/debug/airy/agents/[id]/memory_layout

Agent 记忆卷载内存布局：

```bash
$ cat /sys/kernel/debug/airy/agents/42/memory_layout
L1 working memory:
  pages: 4
  addr_range: 0xffff800000010000 - 0xffff800000014000
  hit_rate: 95.3%
L2 short-term memory:
  pages: 16
  addr_range: 0xffff800000020000 - 0xffff800000030000
  hit_rate: 78.2%
L3 long-term memory:
  pages: 256
  addr_range: 0xffff800100000000 - 0xffff800101000000
  hit_rate: 45.6%
L4 experience memory:
  pages: 1024
  addr_range: 0xffff800200000000 - 0xffff800204000000
  hit_rate: 32.1%
```

### 3.4 /sys/kernel/debug/airy/agents/[id]/ipc_trace

最近 100 条 IPC 交互：

```bash
$ cat /sys/kernel/debug/airy/agents/42/ipc_trace | head -3
[1784328868912] SEND → agent_43 op=SEARCH fastpath=1 latency=1234ns
[1784328868922] RECV ← agent_43 op=SEARCH_RESULT fastpath=1 latency=2345ns
[1784328868932] SEND → broadcast op=NOTIFY fastpath=1 latency=3456ns
```

**OS-OBS-052: Agent 调试信息仅记录最近 100 条，避免 debugfs 文件无限增长；历史数据需通过 A-ULP Ring Buffer 持久化。**

---

## 第 4 章 调度器调试接口

### 4.1 /sys/kernel/debug/airy/sched/runqueue_dump

当前运行队列快照：

```bash
$ cat /sys/kernel/debug/airy/sched/runqueue_dump
CPU 0:
  DL runqueue (3 agents):
    agent_42  runtime=5000ns deadline=10000ns
    agent_43  runtime=5000ns deadline=10000ns
    agent_44  runtime=3000ns deadline=8000ns
  RT runqueue (5 agents):
    agent_50  prio=99
    agent_51  prio=98
    ...
  EEVDF runqueue (12 agents):
    agent_100 vruntime=12345678
    agent_101 vruntime=12345679
    ...
CPU 1:
  ...
```

### 4.2 /sys/kernel/debug/airy/sched/class_internals

三层调度类内部状态：

```bash
$ cat /sys/kernel/debug/airy/sched/class_internals
SCHED_DEADLINE:
  total_bw: 0.65            # 总带宽占用（≤ 1.0 = 100%）
  max_bw: 0.90              # 最大允许带宽
  pushable_tasks: 3         # 可推送任务数
  pulled_tasks: 0           # 已拉取任务数
SCHED_FIFO:
  rt_prio_levels: 99        # 优先级层级
  active_levels: 5          # 活跃层级数
  rt_throttled: 0           # 是否被节流
  rt_runtime_us: 950000     # RT 运行时配额（每秒）
  rt_period_us: 1000000     # RT 周期
EEVDF:
  total_agents: 25
  min_vruntime: 12345678
  lag_sum: 0.0001           # 累积延迟（应接近 0）
  ideal_weight_sum: 25.0
```

### 4.3 /sys/kernel/debug/airy/sched/deadline_tree

SCHED_DEADLINE 红黑树结构：

```bash
$ cat /sys/kernel/debug/airy/sched/deadline_tree
Root: agent_42 (deadline=10000ns)
  Left: agent_43 (deadline=10000ns)
    Left: NULL
    Right: agent_44 (deadline=8000ns)
  Right: agent_45 (deadline=12000ns)
    Left: NULL
    Right: NULL
Total nodes: 5
Leftmost: agent_44 (earliest deadline)
```

### 4.4 /sys/kernel/debug/airy/sched/eevdf_vruntime

EEVDF vruntime 分布直方图：

```bash
$ cat /sys/kernel/debug/airy/sched/eevdf_vruntime
vruntime distribution (25 agents):
  [12345600 - 12345699]: 2 agents
  [12345700 - 12345799]: 5 agents
  [12345800 - 12345899]: 8 agents
  [12345900 - 12345999]: 7 agents
  [12346000 - 12346099]: 3 agents
min_vruntime: 12345600
max_vruntime: 12346050
spread: 450                   # vruntime 差值（应较小）
```

**OS-OBS-053: EEVDF vruntime spread 必须 ≤ 1000；超限需审查调度器是否公平，避免某些 Agent 长期饥饿。**

---

## 第 5 章 IPC 调试接口

### 5.1 /sys/kernel/debug/airy/ipc/ring_dump

io_uring ring 内容快照：

```bash
$ cat /sys/kernel/debug/airy/ipc/ring_dump
Ring 0 (gateway_d):
  SQ ring:
    head: 100  tail: 112  depth: 256
    inflight: 12
    dropped: 0
  CQ ring:
    head: 88  tail: 100  depth: 256
    overflow: 0
  SQE ring (last 5):
    [100] op=IORING_OP_URING_CMD user_data=0x1234 agent_id=42
    [101] op=IORING_OP_URING_CMD user_data=0x1235 agent_id=43
    [102] op=IORING_OP_URING_CMD user_data=0x1236 agent_id=42
    [103] op=IORING_OP_URING_CMD user_data=0x1237 agent_id=44
    [104] op=IORING_OP_URING_CMD user_data=0x1238 agent_id=42
```

### 5.2 /sys/kernel/debug/airy/ipc/sqe_history

最近 100 条 SQE 历史：

```bash
$ cat /sys/kernel/debug/airy/ipc/sqe_history | tail -5
[1784328868912] ring=0 sqe_idx=100 op=IORING_OP_URING_CMD \
    agent_id=42 target=agent_43 cmd=SEARCH submit_ns=1234
[1784328868922] ring=0 sqe_idx=101 op=IORING_OP_URING_CMD \
    agent_id=43 target=agent_42 cmd=SEARCH_RESULT submit_ns=2345
...
```

### 5.3 /sys/kernel/debug/airy/ipc/cqe_history

最近 100 条 CQE 历史：

```bash
$ cat /sys/kernel/debug/airy/ipc/cqe_history | tail -5
[1784328868915] ring=0 cqe_idx=88 user_data=0x1234 res=0 \
    complete_ns=1234 latency=2345ns
...
```

### 5.4 /sys/kernel/debug/airy/ipc/cap_table

capability 表完整快照：

```bash
$ cat /sys/kernel/debug/airy/ipc/cap_table
agent_id=1   caps=IPC_SEND|IPC_RECV|IPC_BROADCAST|IPC_ADMIN
agent_id=42  caps=IPC_SEND|IPC_RECV
agent_id=43  caps=IPC_SEND|IPC_RECV
agent_id=100 caps=IPC_RECV
denial_log (last 10):
  [1784328868912] agent_id=100 denied IPC_SEND (no capability)
  ...
```

**OS-OBS-054: IPC ring_dump 必须实时反映当前状态；`inflight` 与 `dropped` 字段为 0 时分别表示无在途请求与无丢弃。**

---

## 第 6 章 tracefs 集成

### 6.1 tracefs 挂载

tracefs 标准挂载点为 `/sys/kernel/tracing`，agentrt-linux 在 init 阶段自动挂载：

```bash
# 标准挂载
tracefs /sys/kernel/tracing tracefs defaults 0 0
# 兼容挂载（debugfs 下）
/sys/kernel/debug/tracing -> /sys/kernel/tracing
```

### 6.2 events 子目录结构

tracefs 的 `events/` 子目录按子系统组织 tracepoint 事件：

```
/sys/kernel/tracing/events/
├── airy/                    # agentrt-linux 自定义事件
│   ├── sched_switch/        # Agent 调度切换事件
│   │   ├── format           # 事件字段格式
│   │   ├── enable           # 启用/禁用
│   │   ├── filter           # 过滤器
│   │   └── trigger          # 触发器
│   ├── ipc_send/            # IPC 发送事件
│   ├── ipc_recv/            # IPC 接收事件
│   ├── agent_state_change/  # Agent 状态转换事件
│   ├── cognition_enter/     # 认知循环进入事件
│   ├── cognition_exit/      # 认知循环退出事件
│   ├── memory_access/       # 记忆访问事件
│   ├── token_consume/       # Token 消耗事件
│   └── superv_enforce/      # 监管执法事件
├── sched/                   # Linux 原生调度事件
│   ├── sched_switch/
│   ├── sched_wakeup/
│   └── sched_stat_runtime/
├── io_uring/                # io_uring 事件
│   └── io_uring_cmd_submit/
├── kmem/                    # 内存分配事件
│   ├── mm_page_alloc/
│   └── mm_page_free/
└── ...
```

### 6.3 事件启用与禁用

通过 `enable` 文件控制单个事件或整个子系统：

```bash
# 启用单个事件
echo 1 > /sys/kernel/tracing/events/airy/sched_switch/enable

# 启用整个 airy 子系统
echo 1 > /sys/kernel/tracing/events/airy/enable

# 查看启用状态
cat /sys/kernel/tracing/events/airy/sched_switch/enable
# 输出: 1

# 禁用
echo 0 > /sys/kernel/tracing/events/airy/sched_switch/enable
```

### 6.4 事件过滤器

通过 `filter` 文件设置事件过滤器：

```bash
# 仅记录 agent_id == 42 的事件
echo 'agent_id == 42' > /sys/kernel/tracing/events/airy/sched_switch/filter

# 组合条件
echo 'agent_id == 42 && prev_state == COGNITION_RUNNING' \
    > /sys/kernel/tracing/events/airy/sched_switch/filter

# 清除过滤器
echo 0 > /sys/kernel/tracing/events/airy/sched_switch/filter
```

### 6.5 事件格式

每个事件的 `format` 文件描述字段布局：

```bash
$ cat /sys/kernel/tracing/events/airy/sched_switch/format
name: sched_switch
ID: 1234
format:
    field:unsigned short common_type;         offset:0;  size:2;  signed:0;
    field:unsigned char  common_flags;        offset:2;  size:1;  signed:0;
    field:unsigned char  common_preempt_count;offset:3;  size:1;  signed:0;
    field:int            common_pid;          offset:4;  size:4;  signed:1;

    field:u32            agent_id;            offset:8;  size:4;  signed:0;
    field:u32            prev_state;          offset:12; size:4;  signed:0;
    field:u32            next_state;          offset:16; size:4;  signed:0;
    field:u32            sched_class;         offset:20; size:4;  signed:0;
    field:u64            latency_ns;          offset:24; size:8;  signed:0;

print fmt: "agent=%u prev=%u next=%u class=%u latency=%lluns",
    REC->agent_id, REC->prev_state, REC->next_state,
    REC->sched_class, REC->latency_ns
```

**OS-KER-143: agentrt.ko 必须通过 `TRACE_EVENT` 宏定义所有 airy 子系统事件；事件格式必须稳定，不得在 patch 版本间变更字段顺序。**

---

## 第 7 章 agentrt-linux 事件追踪

### 7.1 airy_sched_switch 事件

Agent 调度切换事件，记录sched_tac 三层调度类切换：

```c
/* include/trace/events/airy.h */
TRACE_EVENT(airy_sched_switch,
    TP_PROTO(u32 agent_id, u32 prev_state, u32 next_state,
             u32 sched_class, u64 latency_ns),
    TP_ARGS(agent_id, prev_state, next_state, sched_class, latency_ns),
    TP_STRUCT__entry(
        __field(u32, agent_id)
        __field(u32, prev_state)
        __field(u32, next_state)
        __field(u32, sched_class)
        __field(u64, latency_ns)
    ),
    TP_fast_assign(
        __entry->agent_id = agent_id;
        __entry->prev_state = prev_state;
        __entry->next_state = next_state;
        __entry->sched_class = sched_class;
        __entry->latency_ns = latency_ns;
    ),
    TP_printk("agent=%u prev=%u next=%u class=%u latency=%lluns",
        __entry->agent_id, __entry->prev_state, __entry->next_state,
        __entry->sched_class, __entry->latency_ns)
);
```

### 7.2 airy_ipc_send 事件

IPC 发送事件，记录 IORING_OP_URING_CMD fastpath：

```c
TRACE_EVENT(airy_ipc_send,
    TP_PROTO(u32 src_agent, u32 dst_agent, u32 op, u8 fastpath, u64 latency_ns),
    TP_ARGS(src_agent, dst_agent, op, fastpath, latency_ns),
    TP_STRUCT__entry(
        __field(u32, src_agent)
        __field(u32, dst_agent)
        __field(u32, op)
        __field(u8,  fastpath)
        __field(u64, latency_ns)
    ),
    TP_fast_assign(
        __entry->src_agent = src_agent;
        __entry->dst_agent = dst_agent;
        __entry->op = op;
        __entry->fastpath = fastpath;
        __entry->latency_ns = latency_ns;
    ),
    TP_printk("src=%u dst=%u op=%u fastpath=%u latency=%lluns",
        __entry->src_agent, __entry->dst_agent, __entry->op,
        __entry->fastpath, __entry->latency_ns)
);
```

### 7.3 airy_agent_state_change 事件

Agent 8 态生命周期迁移事件：

```c
TRACE_EVENT(airy_agent_state_change,
    TP_PROTO(u32 agent_id, u32 prev_state, u32 next_state, u32 reason),
    TP_ARGS(agent_id, prev_state, next_state, reason),
    TP_STRUCT__entry(
        __field(u32, agent_id)
        __field(u32, prev_state)
        __field(u32, next_state)
        __field(u32, reason)
    ),
    TP_fast_assign(
        __entry->agent_id = agent_id;
        __entry->prev_state = prev_state;
        __entry->next_state = next_state;
        __entry->reason = reason;
    ),
    TP_printk("agent=%u %u→%u reason=%u",
        __entry->agent_id, __entry->prev_state,
        __entry->next_state, __entry->reason)
);
```

### 7.4 其他关键事件

| 事件名 | 触发点 | 关键字段 |
|--------|--------|---------|
| `airy_cognition_enter` | 认知循环进入 | agent_id, input_tokens |
| `airy_cognition_exit` | 认知循环退出 | agent_id, output_tokens, latency_ns |
| `airy_memory_access` | 记忆访问 | agent_id, level(L1-L4), hit, latency_ns |
| `airy_token_consume` | Token 消耗 | agent_id, input_tokens, output_tokens |
| `airy_superv_enforce` | 监管执法 | agent_id, action, reason |
| `airy_ring_buffer_write` | A-ULP Ring Buffer 写入 | level, size, overrun |

### 7.5 事件追踪输出

启用事件后，通过 `trace` 文件查看输出：

```bash
$ echo 1 > /sys/kernel/tracing/events/airy/enable
$ cat /sys/kernel/tracing/trace | tail -10
# tracer: nop
#
# entries-in-buffer/entries-written: 12345/12345   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||    TIMESTAMP  FUNCTION
#              | |        |   ||||       |         |
  cogn_d-1234  [000] d... 1784328868.912345: airy_cognition_enter: agent=42 input_tokens=234
  sched_d-1235 [001] d... 1784328868.912346: airy_sched_switch: agent=42 prev=COGNITION_RUNNING next=PLANNING class=DL latency=1234ns
  gateway_d-1236 [002] d... 1784328868.912347: airy_ipc_send: src=42 dst=43 op=SEARCH fastpath=1 latency=1234ns
  cogn_d-1234  [000] d... 1784328868.912348: airy_cognition_exit: agent=42 output_tokens=456 latency=1500000ns
```

---

## 第 8 章 与 ftrace 框架的关系

### 8.1 tracefs 作为 ftrace 接口

tracefs 是 ftrace 的标准文件系统接口，agentrt-linux 不创建独立的追踪文件系统，完全复用 tracefs。详细 ftrace 框架规范见 [01-ftrace-framework.md](01-ftrace-framework.md)。

### 8.2 tracefs 关键控制文件

| 文件 | 用途 |
|------|------|
| `current_tracer` | 设置当前 tracer（function/function_graph/event/nop） |
| `available_tracers` | 列出可用 tracer |
| `tracing_on` | 启停 ring buffer 写入 |
| `trace` | 人类可读跟踪输出 |
| `trace_pipe` | 流式跟踪输出（消费型） |
| `set_event` | 批量启用事件 |
| `events/` | 事件子系统目录 |
| `set_ftrace_filter` | 函数过滤白名单 |
| `buffer_size_kb` | 每 CPU 缓冲大小 |
| `trace_clock` | 时间戳时钟源 |

### 8.3 tracefs 与 perf 协同

tracefs 事件可通过 perf 消费：

```bash
# 通过 perf stat 计数 airy 事件
perf stat -e airy:airy_sched_switch,airy:airy_ipc_send -a -- sleep 10

# 通过 perf record 采样 airy 事件
perf record -e airy:airy_sched_switch -a -- sleep 30
perf report
```

详细 perf 集成规范见 [03-perf-analysis.md](03-perf-analysis.md)。

### 8.4 tracefs 与 eBPF 协同

eBPF 程序可挂载到 tracepoint 事件：

```bash
# 将 BPF 程序挂载到 airy_sched_switch tracepoint
bpftool prog load airy_trace.o /sys/fs/bpf/airy_trace \
    type tracepoint attach airy_sched_switch
```

详细 eBPF 集成规范见 [02-ebpf-probes.md](02-ebpf-probes.md)。

**OS-OBS-055: tracefs 事件必须同时支持 ftrace 直接读取、perf stat/record 计数、eBPF 挂载三种消费方式；任一方式失效视为事件定义不合规。**

---

## 第 9 章 Airymax Unify Design 映射

### 9.1 五模块关系

| Unify 模块 | 关系 | 在 debugfs/tracefs 的体现 |
|-----------|------|--------------------------|
| **A-UEF** | **核心** | `/sys/kernel/debug/airy/superv/micro_decisions` 记录冷酷执法历史；tracefs `airy_superv_enforce` 事件 |
| **A-ULP** | **核心** | `/sys/kernel/debug/airy/log/` 提供 Ring Buffer 原始 dump；tracefs `airy_ring_buffer_write` 事件 |
| **A-UCS** | 辅助 | debugfs 不导出 A-UCS 状态（已由 procfs 覆盖） |
| **A-ULS** | **核心** | `/sys/kernel/debug/airy/superv/` 提供 Micro/Macro 决策历史；tracefs `airy_superv_enforce` 事件 |
| **A-IPC** | **核心** | `/sys/kernel/debug/airy/ipc/` 提供 ring dump 与 SQE/CQE 历史；tracefs `airy_ipc_send/recv` 事件 |

### 9.2 与 12 daemon 的 debugfs 映射

| Daemon | debugfs 路径 | 主要调试内容 |
|--------|------------|------------|
| macro_d | `/sys/kernel/debug/airy/superv/macro_*` | 温情裁决历史 |
| logger_d | `/sys/kernel/debug/airy/log/logger_state` | 消费速率与 flush 延迟 |
| config_d | （无独立 debugfs） | 已由 procfs 覆盖 |
| gateway_d | `/sys/kernel/debug/airy/ipc/ring_dump` | io_uring ring 快照 |
| sched_d | `/sys/kernel/debug/airy/sched/` | 运行队列与调度类内部状态 |
| vfs_d | `/sys/kernel/debug/airy/vfs/` | 挂载点与 inode 缓存 |
| net_d | `/sys/kernel/debug/airy/net/` | socket 表与路由缓存 |
| mem_d | `/sys/kernel/debug/airy/mem/` | 页分配日志与池布局 |
| cogn_d | `/sys/kernel/debug/airy/agents/[id]/decision_log` | Agent 决策记录 |
| sec_d | `/sys/kernel/debug/airy/sec/` | LSM 钩子与 capability 校验日志 |
| audit_d | `/sys/kernel/debug/airy/audit/` | 审计记录缓冲 |
| dev_d | `/sys/kernel/debug/airy/dev/` | 设备事件缓冲 |

---

## 第 10 章 相关文档与版本维护

### 10.1 相关文档

- [90-observability README](README.md)：可观测性体系主索引
- [01-ftrace-framework.md](01-ftrace-framework.md)：ftrace 框架详解（L1 层）
- [02-ebpf-probes.md](02-ebpf-probes.md)：eBPF 探针（L2 层）
- [03-perf-analysis.md](03-perf-analysis.md)：perf 性能分析（L3 层）
- [04-sysfs-procfs.md](04-sysfs-procfs.md)：sysfs/procfs 接口（L4 层生产）
- [06-user-events.md](06-user-events.md)：user_events 接口
- [08-agent-tracing.md](08-agent-tracing.md)：Agent 行为追踪

### 10.2 参考材料

- Linux 6.6 `Documentation/filesystems/debugfs.rst`（debugfs 文档）
- Linux 6.6 `Documentation/trace/ftrace.rst`（ftrace 文档）
- Linux 6.6 `Documentation/trace/events.rst`（tracepoint 事件文档）
- Linux 6.6 `fs/debugfs/`（debugfs 实现）
- Linux 6.6 `fs/tracefs/`（tracefs 实现）
- Linux 6.6 `include/trace/events/`（tracepoint 事件示例）

### 10.3 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0.1 | 2026-07-18 | 初始版本：/sys/kernel/debug/airy/ 目录设计、Agent/调度器/IPC 调试接口、tracefs events 子系统集成、airy_sched_switch/airy_ipc_send/airy_agent_state_change 事件定义、与 ftrace 框架关系 |

---

> **文档结束** | agentrt-linux debugfs/tracefs 接口 v1.0.1 | 维护者：开源极境工程与规范委员会 | "Debugging is twice as hard as writing the code in the first place."
