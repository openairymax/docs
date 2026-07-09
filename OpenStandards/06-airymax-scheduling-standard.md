Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax 调度标准

> **文档定位**: Airymax 开放标准
> **版本**: 1.0（开放标准草案）
> **最后更新**: 2026-07-09
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 一、范围与目的

本标准定义 AI Agent 调度的开放契约，覆盖：

- SCHED_EXT 调度类标准
- 任务描述符格式标准（magic `0x41475453` 'AGTS'）
- vtime 衰减公式标准（Q16.16 定点数）
- Agent 优先级范围标准
- 1024 并发上限标准（MAC_MAX_AGENTS）

本标准面向：

- **运行时实现者**：实现兼容的 Agent 调度器（用户态调度器、内核 sched_ext BPF 等）。
- **Agent 作者**：理解优先级与预算对调度的影响。
- **调度策略开发者**：编写可插拔的 BPF 调度策略。
- **性能优化方**：基于 vtime 衰减公式调优公平性。

本标准遵循 IRON-9 v2 [SC] 共享契约层——任务描述符字节布局、vtime 公式、优先级范围在 agentrt 与 agentrt-liunx 之间完全共享。

### 1.1 术语

| 术语 | 含义 |
|------|------|
| SCHED_EXT | Linux 可扩展调度器框架（主线 6.12+，agentrt-liunx 向前移植到 6.6） |
| SCHED_AGENT | Airymax 定义的 Agent 调度类，基于 sched_ext |
| 任务描述符 | Agent 任务的二进制描述结构（magic 'AGTS'） |
| vtime | 虚拟时间，用于公平调度的相对时间度量 |
| Q16.16 | 16 位整数 + 16 位小数的定点数格式 |
| MAC_MAX_AGENTS | 单节点最大并发 Agent 数（1024） |

### 1.2 规范用词

沿用 RFC 2119 用词。

### 1.3 设计原则

本标准遵循五维正交原则：

| 原则 | 在调度中的体现 |
|------|---------------|
| K-1 内核极简 | 调度机制在内核，调度策略可外移到用户态 BPF |
| K-4 可插拔策略 | 调度策略通过 struct_ops 注入，运行时可替换 |
| E-3 资源确定性 | 每个任务有明确的优先级与预算 |
| C-2 增量演化 | Agent 不一次性规划全部步骤，分阶段调度 |

---

## 二、SCHED_EXT 调度类标准

### 2.1 调度类枚举

| 调度类 | 数值 | 名称 | 适用场景 |
|--------|------|------|---------|
| SCHED_NORMAL | 0 | 普通进程 | 默认 Linux 进程 |
| SCHED_FIFO | 1 | 实时 FIFO | 实时任务 |
| SCHED_RR | 2 | 实时轮转 | 实时任务 |
| SCHED_BATCH | 3 | 批处理 | 批量任务 |
| SCHED_IDLE | 5 | 空闲 | 低优先级后台 |
| SCHED_DEADLINE | 6 | 截止时间 | 实时保证 |
| SCHED_EXT | 7 | 可扩展 | agentrt-liunx 1.0.1 内置（基于 Linux 6.6 内核基线向前移植） |
| SCHED_AGENT | 8 | Agent 调度类 | Airymax 专属（基于 SCHED_EXT） |

`SCHED_EXT=7` **永久稳定**（AOS-STD-SCHED-001，L1 稳定级，对齐 Linux 主线）。

### 2.2 SCHED_AGENT 调度类

SCHED_AGENT 是 Airymax 基于 SCHED_EXT 实现的 Agent 专属调度类，**必须**满足：

- **必须**通过 sched_ext struct_ops 机制注册（AOS-STD-SCHED-002）。
- **必须**支持 4 个调度策略维度（AOS-STD-SCHED-003）：
  1. 认知阶段感知：CoreLoopThree 阶段 Agent 获得更高优先级。
  2. Token 预算感知：预算即将耗尽的 Agent 获得调度加速。
  3. 任务优先级：关键任务（安全审计、事务提交）获得实时保证。
  4. 跨 Agent 协同：多 Agent 协作任务在相邻时间片内调度。
- **应当**支持用户态 BPF 策略注入（AOS-STD-SCHED-004）。

### 2.3 调度策略优先级维度

| 维度 | 权重 | 计算依据 |
|------|------|---------|
| 认知阶段 | 0.4 | PERCEPTION/THINKING 阶段优先于 ACTION |
| Token 预算 | 0.3 | 剩余预算 < 20% 时优先调度 |
| 任务优先级 | 0.2 | 0-49 实时 / 50-99 交互 / 100-139 批处理 |
| 跨 Agent 协同 | 0.1 | 协作组内 Agent 相邻调度 |

最终调度优先级 = 加权和（AOS-STD-SCHED-005）。

---

## 三、任务描述符格式标准

### 3.1 整体布局

任务描述符为 128 字节定长结构，magic `0x41475453`（'AGTS'）。**注意**：任务描述符**不是** IPC 消息，使用独立 magic 避免混淆。

布局**永久稳定**（AOS-STD-SCHED-006，L1 稳定级）：

```
偏移 0        4       6      8          12         16         20        24         32
+-----------+--------+------+----------+----------+----------+----------+----------+
| magic 4B  | ver 2B | rsvd | priority | agent_id | task_id  | vtime 8B | deadline |
| 'AGTS'    |        |  2B  | 4B       | 8B       | 8B       | Q16.16?  | 8B       |
+-----------+--------+------+----------+----------+----------+----------+----------+

偏移 40                    48                       56                       64
+--------------------------+--------------------------+--------------------------+
| token_budget 8B          | token_consumed 8B        | started_ns 8B           |
+--------------------------+--------------------------+--------------------------+

偏移 72                    80                       88                       96
+--------------------------+--------------------------+--------------------------+
| cog_phase 2B + rsvd 6B  | think_mode 2B + rsvd 6B | cap_id 4B + rsvd 4B      |
+--------------------------+--------------------------+--------------------------+

偏移 104                                                                          128
+----------------------------------------------------------------------------------+
| reserved 24B                                                                     |
+----------------------------------------------------------------------------------+
```

### 3.2 C 结构定义

```c
#define AGENTRT_TASK_MAGIC       0x41475453u  /* 'A''G''T''S' */
#define AGENTRT_TASK_VERSION     0x0100u
#define AGENTRT_TASK_DESC_SIZE   128u

typedef struct __attribute__((aligned(64))) agentrt_task_desc {
    uint32_t magic;            /* 偏移 0:  魔数 0x41475453 'AGTS' */
    uint16_t version;          /* 偏移 4:  版本 0x0100 */
    uint16_t reserved1;        /* 偏移 6:  保留 */
    uint32_t priority;         /* 偏移 8:  优先级（0-139） */
    uint64_t agent_id;         /* 偏移 12: Agent ID */
    uint64_t task_id;          /* 偏移 20: 任务 ID */
    uint64_t vtime;            /* 偏移 28: 虚拟时间（Q16.16 定点） */
    uint64_t deadline_ns;      /* 偏移 36: 截止时间（0=无） */
    uint64_t token_budget;     /* 偏移 44: Token 预算 */
    uint64_t token_consumed;   /* 偏移 52: 已消耗 Token */
    uint64_t started_ns;       /* 偏移 60: 启动时间 */
    uint16_t cog_phase;        /* 偏移 68: 认知阶段（对齐 05 标准） */
    uint16_t reserved2;
    uint32_t reserved3;
    uint16_t think_mode;       /* 偏移 76: 思考模式（对齐 05 标准） */
    uint16_t reserved4;
    uint32_t reserved5;
    uint32_t cap_id;            /* 偏移 84: capability ID */
    uint32_t reserved6;
    uint8_t  reserved[24];     /* 偏移 92: 保留扩展 */
} agentrt_task_desc_t;

_Static_assert(sizeof(agentrt_task_desc_t) == AGENTRT_TASK_DESC_SIZE,
               "agentrt_task_desc_t must be 128 bytes");
```

> 注：上述偏移为概念示意，实际字段顺序由结构体定义决定，编译后通过 `_Static_assert` 保证总长度 128 字节。本标准约束**总长度 128 字节 + magic 值 + 字段语义**，**不**约束具体偏移（允许实现微调字段顺序以优化对齐）。

### 3.3 字段语义

| 字段 | 约束 | 标准条目 |
|------|------|---------|
| `magic` | **必须**为 `0x41475453` | AOS-STD-SCHED-007 |
| `version` | 当前 `0x0100` | AOS-STD-SCHED-008 |
| `priority` | 0-139，对齐 SCHED_AGENT | AOS-STD-SCHED-009 |
| `agent_id` | 全局唯一 | AOS-STD-SCHED-010 |
| `task_id` | 同 Agent 内唯一 | AOS-STD-SCHED-011 |
| `vtime` | Q16.16 定点数 | AOS-STD-SCHED-012 |
| `deadline_ns` | 0 表示无截止 | AOS-STD-SCHED-013 |
| `token_budget` | 硬上限 | AOS-STD-SCHED-014 |
| `cog_phase` | 对齐 05 标准枚举 | AOS-STD-SCHED-015 |
| `think_mode` | 对齐 05 标准枚举 | AOS-STD-SCHED-016 |
| `cap_id` | 任务关联的 capability | AOS-STD-SCHED-017 |

---

## 四、vtime 衰减公式标准

### 4.1 Q16.16 定点数格式

vtime 使用 Q16.16 定点数：16 位整数 + 16 位小数，存储在 `uint64_t` 中（高 48 位整数 + 低 16 位小数，预留扩展）。

```c
typedef uint64_t agentrt_q16_t;

#define AGENTRT_Q16_ONE        (1ULL << 16)   /* 1.0 的 Q16.16 表示 = 65536 */
#define AGENTRT_Q16_SHIFT      16

/* 转换：整数 -> Q16.16 */
static inline agentrt_q16_t agentrt_q16_from_int(uint64_t i) {
    return i << AGENTRT_Q16_SHIFT;
}

/* 转换：Q16.16 -> 整数（截断小数） */
static inline uint64_t agentrt_q16_to_int(agentrt_q16_t q) {
    return q >> AGENTRT_Q16_SHIFT;
}
```

Q16.16 格式**永久稳定**（AOS-STD-SCHED-018，L1 稳定级）——内核态禁止使用 float/double（对齐工程标准 OS-KER-007），所有调度计算**必须**使用 Q16.16 定点数。

### 4.2 vtime 衰减公式

vtime（虚拟时间）用于公平调度。每次任务运行后，vtime 按以下公式衰减（AOS-STD-SCHED-019，L1 稳定级）：

```
vtime_new = vtime_old + (actual_runtime * weight) >> 16

其中：
  weight = Q16.16(1.0) / priority_weight[priority]
  priority_weight[priority] = 1024 / (1.25 ^ (priority / 8))
```

### 4.3 优先级权重表

| 优先级范围 | weight（近似） | 说明 |
|-----------|----------------|------|
| 0-7 | 1.0 (Q16=65536) | 实时，权重最高 |
| 8-15 | 0.8 (Q16=52428) | 实时 |
| 50-57 | 0.4 (Q16=26214) | 交互 |
| 100-107 | 0.2 (Q16=13107) | 普通 |
| 132-139 | 0.1 (Q16=6553) | 批处理，权重最低 |

权重表**应当**可由策略注入覆盖（AOS-STD-SCHED-020）。

### 4.4 vtime 比较规则

调度器选择 vtime 最小的任务运行（AOS-STD-SCHED-021）。vtime 相同时：

1. 优先级数值小的优先（数值小 = 高优先级）。
2. deadline 早的优先。
3. task_id 小的优先（FIFO 兜底）。

---

## 五、Agent 优先级范围标准

### 5.1 优先级范围

Agent 优先级为 0-139，对齐 Linux nice 值与 SCHED_AGENT（AOS-STD-SCHED-022，L1 稳定级）：

| 范围 | 类别 | 延迟预算 | 调度策略 | 典型场景 |
|------|------|---------|---------|---------|
| 0-49 | 实时控制 | < 1 ms | scx_realtime | 具身智能运动控制 |
| 50-99 | 交互响应 | < 10 ms | scx_interactive | 用户对话 |
| 100-119 | Agent 认知 | < 100 ms | scx_agent | CoreLoopThree 阶段 |
| 120-139 | 批处理 | < 1 s | scx_batch | 日志、记忆迁移 |

### 5.2 优先级设置规则

| 规则 | 标准条目 |
|------|---------|
| 创建时**必须**指定优先级 | AOS-STD-SCHED-023 |
| 优先级 0-49 需要 `CAP_AGENT_SCHED` capability | AOS-STD-SCHED-024 |
| 运行时**可以**调整优先级（在范围内） | AOS-STD-SCHED-025 |
| 调整后 vtime 不重置 | AOS-STD-SCHED-026 |

---

## 六、1024 并发上限标准

### 6.1 MAC_MAX_AGENTS

单节点最大并发 Agent 数**必须**为 1024（AOS-STD-SCHED-027，L1 稳定级）：

```c
#define MAC_MAX_AGENTS  1024u
```

### 6.2 上限约束

| 约束 | 标准条目 |
|------|---------|
| 单节点并发 Agent 数**不得**超过 1024 | AOS-STD-SCHED-028 |
| 超过时 `agentrt_agent_schedule` **必须**返回 `AGENTRT_EBUSY` | AOS-STD-SCHED-029 |
| 1024 个 Agent 共享调度器数据结构，**应当**使用无锁或细粒度锁 | AOS-STD-SCHED-030 |
| 调度器就绪队列**应当**按优先级分桶（140 个桶） | AOS-STD-SCHED-031 |

### 6.3 上限合理性

1024 的选择基于：

- L1 cache 友好（任务描述符 128B × 1024 = 128 KB，约 2 个 cache way）。
- 内存可控（每 Agent 默认 L1 工作记忆 256 KB × 1024 = 256 MB）。
- 调度复杂度 O(140)（优先级桶选择）+ O(log N)（vtime 堆）。
- 实测单节点 1024 并发可满足 99% 智能体场景。

---

## 七、BPF 调度策略骨架

### 7.1 struct_ops 注册

```c
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include "airymax/sched_agent.bpf.h"

char LICENSE[] SEC("license") = "GPL";

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, u32);
    __type(value, struct agentrt_task_desc);
    __uint(max_entries, MAC_MAX_AGENTS);  /* 1024 */
} task_map SEC(".maps");

/* 7.2 入队：选择下一个运行的任务 */
s32 BPF_STRUCT_OPS(agentrt_enqueue, struct task_struct *p, u64 enq_flags)
{
    u32 pid = p->pid;
    struct agentrt_task_desc *desc = bpf_map_lookup_elem(&task_map, &pid);
    if (!desc || desc->magic != AGENTRT_TASK_MAGIC) {
        /* 非 Airymax 任务，交给默认调度器 */
        return -ENOTSUP;
    }

    /* 更新 vtime */
    u64 runtime = p->se.sum_exec_runtime - desc->started_ns;
    u64 weight  = agentrt_q16_weight(desc->priority);  /* Q16.16 权重 */
    desc->vtime += (runtime * weight) >> AGENTRT_Q16_SHIFT;

    /* 入就绪队列（按 vtime 排序的 BPF rbtree） */
    return agentrt_bpf_enqueue_vtime(p, desc->vtime, enq_flags);
}

/* 7.3 出队：选择 vtime 最小的任务 */
struct task_struct *BPF_STRUCT_OPS(agentrt_pick_next, struct task_struct *prev)
{
    return agentrt_bpf_pick_min_vtime();
}

/* 7.4 调度类注册 */
struct sched_ext_ops agentrt_sched_ops = {
    .enqueue    = (void *)agentrt_enqueue,
    .pick_next  = (void *)agentrt_pick_next,
    .name       = "agentrt_sched",
};
SEC(".struct_ops.link")
struct sched_ext_ops *agentrt_sched_ops_link = &agentrt_sched_ops;
```

### 7.2 Q16.16 权重计算（BPF 内联）

```c
static __always_inline u64 agentrt_q16_weight(u32 priority)
{
    /* 简化权重表：实时(0-49)=1.0, 交互(50-99)=0.4, 认知(100-119)=0.2, 批(120-139)=0.1 */
    if (priority < 50)  return 65536ULL;   /* 1.0 */
    if (priority < 100) return 26214ULL;  /* 0.4 */
    if (priority < 120) return 13107ULL;  /* 0.2 */
    return 6553ULL;                       /* 0.1 */
}
```

---

## 八、调度接口

### 8.1 提交任务

```c
/**
 * agentrt_sched_submit - 提交任务到调度器
 * @desc: 任务描述符（已填充）
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   描述符非法（magic/version 不匹配）
 *   -AGENTRT_EBUSY:    并发数达到 MAC_MAX_AGENTS
 *   -AGENTRT_EPERM:    优先级超权限
 */
int agentrt_sched_submit(agentrt_task_desc_t *desc);
```

### 8.2 撤销任务

```c
/**
 * agentrt_sched_cancel - 撤销任务
 * @task_id: 任务 ID
 */
int agentrt_sched_cancel(uint64_t task_id);
```

### 8.3 查询状态

```c
/**
 * agentrt_sched_query - 查询任务调度状态
 * @task_id:  任务 ID
 * @state_out: 输出：当前状态（vtime、剩余预算等）
 */
int agentrt_sched_query(uint64_t task_id, agentrt_task_desc_t *state_out);
```

---

## 九、标准条目注册表

| 编号 | 名称 | 稳定级 |
|------|------|--------|
| AOS-STD-SCHED-001 | SCHED_EXT=7 调度类枚举 | L1 |
| AOS-STD-SCHED-002 | struct_ops 注册机制 | L1 |
| AOS-STD-SCHED-003 | 4 调度策略维度 | L1 |
| AOS-STD-SCHED-004 | 用户态 BPF 策略注入 | L2 |
| AOS-STD-SCHED-005 | 加权和优先级 | L2 |
| AOS-STD-SCHED-006 | 任务描述符 128 字节 | L1 |
| AOS-STD-SCHED-007 | 任务描述符 magic 0x41475453 | L1 |
| AOS-STD-SCHED-008 | 任务描述符版本 | L1 |
| AOS-STD-SCHED-009 | priority 0-139 | L1 |
| AOS-STD-SCHED-010 | agent_id 全局唯一 | L1 |
| AOS-STD-SCHED-011 | task_id Agent 内唯一 | L1 |
| AOS-STD-SCHED-012 | vtime Q16.16 | L1 |
| AOS-STD-SCHED-013 | deadline_ns | L1 |
| AOS-STD-SCHED-014 | token_budget 硬上限 | L1 |
| AOS-STD-SCHED-015 | cog_phase 对齐 05 标准 | L1 |
| AOS-STD-SCHED-016 | think_mode 对齐 05 标准 | L1 |
| AOS-STD-SCHED-017 | cap_id | L1 |
| AOS-STD-SCHED-018 | Q16.16 格式 | L1 |
| AOS-STD-SCHED-019 | vtime 衰减公式 | L1 |
| AOS-STD-SCHED-020 | 权重表可注入 | L2 |
| AOS-STD-SCHED-021 | vtime 最小优先 | L1 |
| AOS-STD-SCHED-022 | 优先级 0-139 范围 | L1 |
| AOS-STD-SCHED-023 | 创建时指定优先级 | L1 |
| AOS-STD-SCHED-024 | 实时优先级需 CAP | L1 |
| AOS-STD-SCHED-025 | 优先级运行时可调 | L2 |
| AOS-STD-SCHED-026 | 调整不重置 vtime | L2 |
| AOS-STD-SCHED-027 | MAC_MAX_AGENTS=1024 | L1 |
| AOS-STD-SCHED-028 | 并发上限 1024 | L1 |
| AOS-STD-SCHED-029 | 超限返回 EBUSY | L1 |
| AOS-STD-SCHED-030 | 无锁/细粒度锁 | L2 |
| AOS-STD-SCHED-031 | 140 优先级桶 | L2 |

---

## 十、错误码

| 错误码 | 数值 | 含义 |
|--------|------|------|
| `AGENTRT_OK` | 0 | 成功 |
| `AGENTRT_EINVAL` | -2 | 描述符非法（magic/version 不匹配） |
| `AGENTRT_EBUSY` | -1601 | 并发数达到上限 |
| `AGENTRT_EPERM` | -1302 | 优先级超权限 |
| `AGENTRT_ESCHED` | -1602 | 调度器未初始化 |
| `AGENTRT_ENOENT` | -2 | 任务不存在 |

---

## 十一、兼容性测试要求

| 测试 ID | 测试内容 | 通过条件 |
|---------|---------|---------|
| SCHED-T-001 | 任务描述符布局 | 128 字节，magic 匹配 |
| SCHED-T-002 | SCHED_EXT=7 | 调度类枚举值精确匹配 |
| SCHED-T-003 | 优先级范围 | 0-139 全范围可设置 |
| SCHED-T-004 | 优先级超权限 | 0-49 无 CAP 返回 EPERM |
| SCHED-T-005 | vtime Q16.16 | 定点数转换正确 |
| SCHED-T-006 | vtime 衰减 | 高优先级任务 vtime 增长慢 |
| SCHED-T-007 | vtime 最小优先 | vtime 最小任务被选中 |
| SCHED-T-008 | 1024 并发上限 | 第 1025 个返回 EBUSY |
| SCHED-T-009 | cog_phase 对齐 | 枚举值与 05 标准一致 |
| SCHED-T-010 | BPF 策略注入 | 策略可运行时替换 |

---

## 十二、最小示例

### 12.1 任务描述符构造

```c
agentrt_task_desc_t desc = {0};
desc.magic         = AGENTRT_TASK_MAGIC;   /* 0x41475453 */
desc.version       = AGENTRT_TASK_VERSION;  /* 0x0100 */
desc.priority      = 100;                   /* Agent 认知类 */
desc.agent_id      = 42;
desc.task_id       = next_task_id();
desc.vtime         = agentrt_q16_from_int(0); /* 初始 vtime = 0 */
desc.deadline_ns   = 0;                     /* 无截止 */
desc.token_budget  = 4096;
desc.token_consumed = 0;
desc.started_ns    = now_ns();
desc.cog_phase     = AGENTRT_COG_PHASE_PERCEPTION;  /* 对齐 05 标准 */
desc.think_mode    = AGENTRT_THINK_SYSTEM2_SLOW;
desc.cap_id        = 1001;

int rc = agentrt_sched_submit(&desc);
if (rc == -AGENTRT_EBUSY) {
    /* 并发已满 */
}
```

### 12.2 Q16.16 vtime 计算

```c
/* 任务运行了 5000 ns，优先级 100（weight=0.2=13107） */
uint64_t runtime = 5000;
uint64_t weight  = 13107;  /* Q16.16(0.2) */
agentrt_q16_t delta = (runtime * weight) >> AGENTRT_Q16_SHIFT;
/* delta = (5000 * 13107) >> 16 = 65535000 >> 16 = 1000 (Q16.16) */
/* 实际 vtime 增长 = 1000 / 65536 ≈ 0.0153 单位 */

desc.vtime += delta;
```

### 12.3 BPF 调度器加载

```bash
# 加载 SCHED_AGENT BPF 调度器
sudo bpftool struct_ops register agentrt_sched.bpf.o

# 验证调度类已切换
cat /sys/kernel/sched_ext/ops
# 输出: agentrt_sched
```

---

## 十三、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-07-09 | 初始草案：SCHED_EXT 调度类、任务描述符 'AGTS'、vtime Q16.16 衰减、优先级 0-139、1024 并发上限 |

---

## 十四、参考文献

- [Airymax 开放标准体系总览](./README.md)
- [Airymax Agent 运行时标准](./01-airymax-agent-runtime-standard.md)
- [Airymax 认知循环标准](./05-airymax-cognition-loop-standard.md)
- [Airymax 安全模型标准](./03-airymax-security-model-standard.md)
- Linux sched_ext 文档：https://docs.kernel.org/scheduler/sched-ext.html
- BPF struct_ops：https://docs.kernel.org/bpf/bpf_implementation.html
- EEVDF 调度器：https://lwn.net/Articles/925371/

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
