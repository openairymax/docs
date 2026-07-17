Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax IRON-9 v2 跨端共享模型
> **文档定位**：Airymax IRON-9 v2 跨端共享模型\
> **文档版本**：0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**：2026-07-10\
> **上级文档**：[AirymaxAgentRT 文档中心](README.md)\
> **权威来源**：AirymaxOS [`50-engineering-standards/120-cross-project-code-sharing.md`](../../AirymaxOS/50-engineering-standards/120-cross-project-code-sharing.md)（SSoT）

---

## 文档信息卡

- **目标读者**: agentrt 核心开发者、agentrt-linux 内核开发者、SDK 开发者、跨端集成工程师
- **前置知识**: 理解 agentrt 微核心架构（MicroCoreRT），了解 agentrt-linux（AirymaxOS）整体方案
- **预计阅读时间**: 30 分钟
- **核心概念**: IRON-9 v2、[SC] 共享契约层、[SS] 语义同源层、[IND] 完全独立层、SSoT
- **复杂度标识**: 高级

---

## 1. 引言

### 1.1 背景

Airymax 体系包含两个核心工程：

| 工程 | 定位 | 运行形态 |
|------|------|---------|
| **agentrt**（AirymaxAgentRT） | AI Agent 运行时平台工程 | 用户态跨平台（Linux/macOS/Windows） |
| **agentrt-linux**（AirymaxOS） | 智能体操作系统 | Linux 内核态发行版 |

两者之间存在大量共享的技术契约——IPC 消息头布局、错误码体系、capability 安全模型、调度参数、任务描述符格式等。如果这些技术契约在两端各自独立定义，将导致跨端通信断裂、编译行为不可预测、安全模型冲突等严重问题。

**IRON-9 v2 跨端共享模型**就是为解决这一问题而建立的工程标准，确保每个技术点在整个 Airymax 体系中**只有一个规范标准（SSoT）**。

### 1.2 设计目标

1. **SSoT 原则**：每个技术契约在整个体系中只有唯一定义，其他文档引用此定义
2. **跨端一致**：agentrt（用户态）与 agentrt-linux（内核态）共享同一套核心数据结构和 magic 值
3. **演进可控**：共享契约的变更必须双向同步，禁止单方面修改
4. **分类明确**：将跨端共享内容分为三层，明确每层的共享程度和变更约束

---

## 2. IRON-9 v2 三层共享模型

IRON-9 v2 将 agentrt 与 agentrt-linux 之间的共享内容分为三个层级：

### 2.1 [SC] 共享契约层（Shared Contract）

**定义**：agentrt 与 agentrt-linux **物理共享同一份头文件**，逐字节相同，通过 `#include` 引用。

**特征**：
- 物理宿主在 `agentrt-linux/kernel/include/airymax/` 目录
- agentrt 通过 `-I` 编译选项引用同一物理文件
- **禁止任何一端单方面修改**——变更必须双向同步
- 使用内核 UAPI 类型（`__u32`/`__u16`/`__u64`/`__u8`），不使用 `uint32_t` 等 C 标准库类型
- **禁止使用 `float` 类型**——内核态不支持浮点运算，需改用定点数（Q16.16）或下沉用户态

**10 个 [SC] 共享头文件**（SSoT 权威清单，对齐 `docs/AirymaxRT/50-engineering-standards/120-cross-project-code-sharing.md` §2.1，两端逐字节相同）：

| # | 头文件 | 物理路径 | 共享内容 |
|---|--------|---------|---------|
| 1 | `memory_types.h` | `include/airymax/memory_types.h` | MemoryRovol L1-L4 数据结构 + GFP 掩码语义 + PMEM 持久化接口 |
| 2 | `security_types.h` | `include/airymax/security_types.h` | POSIX capability 41 ID 枚举 + LSM 钩子 252 ID 枚举 + Cupolas blob 布局 + capability 派生模型 + Vault backend 抽象 + 策略裁决 4 值枚举 |
| 3 | `cognition_types.h` | `include/airymax/cognition_types.h` | `airy_q16_t` Q16.16 定点数 + CoreLoopThree 阶段枚举（`airy_cog_phase`）+ Thinkdual 模式枚举（`airy_think_mode`） |
| 4 | `sched.h` | `include/airymax/sched.h` | SCHED_EXT 调度类编号 + 任务描述符 + vtime 类型与衰减公式 + 优先级范围 + AIRY_SLICE_DFL |
| 5 | `ipc.h` | `include/airymax/ipc.h` | IPC magic + 128B 消息头结构 + SQE/CQE 操作码与标志位 |
| 6 | `syscalls.h` | `include/airymax/syscalls.h` | Syscall 编号体系（12 核心 + 12 预留 = 24 槽位） |

> **补充共享文件**（非 [SC] 核心头文件）：`bpf_struct_ops.h` 位于 `include/airymax/bpf_struct_ops.h`，含 struct_ops 状态机 + common_value，用于 agentrt 用户态 BPF loader 与 agentrt-linux 内核 struct_ops 框架之间的状态同步。两端共享代码但**不属于上述 10 个 [SC] 核心头文件**（详见 `120-cross-project-code-sharing.md` §2.2）。

### 2.2 [SS] 语义同源层（Semantic Sibling）

**定义**：agentrt 与 agentrt-linux 的**高层 API 语义同源**——功能语义一致，但函数签名各自独立演进。

**特征**：
- 不共享同一份源码，但 API 语义保持一致
- agentrt 侧使用 JSON-RPC 风格高层 API（如 `airy_sys_task_submit(input, input_len, timeout_ms, out_output)`）
- agentrt-linux 侧使用编号 syscall 风格（如 `#512 airy_sys_task_submit(task_desc, priority)`）
- SDK 层签名确实同源（同一份源码两端编译）
- 详细的语义映射表见 AirymaxOS [`30-interfaces/05-syscall-semantic-mapping.md`](../../AirymaxOS/30-interfaces/05-syscall-semantic-mapping.md)

**[SS] 语义同源映射示例**：

| # | agentrt 高层 API | agentrt-linux 编号 syscall | 语义 |
|---|-----------------|--------------------------|------|
| 1 | `airy_sys_task_submit(input, input_len, timeout_ms, out_output)` | #512 `airy_sys_task_submit(task_desc, priority)` | 提交 Agent 任务 |
| 2 | `airy_sys_task_query(task_id, out_status)` | #513 `airy_sys_task_query(task_id)` | 查询任务状态 |
| 3 | `airy_sys_task_cancel(task_id)` | #514 `airy_sys_task_cancel(task_id)` | 取消任务 |
| 4 | `airy_sys_agent_spawn(config, out_agent_id)` | #516 `airy_sys_agent_spawn(config)` | 创建 Agent |
| 5 | `airy_sys_capability_request(...)` | #544 `airy_sys_capability_request(...)` | 请求 capability |
| 6 | `airy_sys_capability_revoke(...)` | #545 `airy_sys_capability_revoke(...)` | 撤销 capability |

### 2.3 [IND] 完全独立层（Independent）

**定义**：agentrt 与 agentrt-linux 各自独有，不共享。

**特征**：
- agentrt 独有：跨平台抽象（POSIX/libc）、JSON-RPC 统一入口 `airy_syscall_invoke()`、ops 注入机制等
- agentrt-linux 独有：内核态 syscall 编号（512-631）、LSM 钩子集成、KABI 二进制兼容、Kbuild/Kconfig 构建系统等

---

## 3. [SC] 共享契约层核心定义

### 3.1 IPC magic 值

```c
/* include/airymax/ipc.h */
#define AIRY_IPC_MAGIC    0x41524531u   /* 'ARE1' —— IPC 消息头 magic */
```

- **值**：`0x41524531`（ASCII 字符 'A''R''E''1' 的大端表示）
- **用途**：标识 AgentsIPC 协议消息，接收方必须校验 magic，不匹配则丢弃
- **SSoT**：`include/airymax/ipc.h`，两端共享同一物理头文件

### 3.2 IPC 128B 消息头

**SSoT 定义**（`include/airymax/ipc.h`，物理宿主见 AirymaxOS `120-cross-project-code-sharing.md`）：

```c
/* include/airymax/ipc.h —— [SC] 共享契约层，逐字节共享 */
struct airy_ipc_msg_hdr {
    __u32   magic;          /* offset 0, 'ARE1' (0x41524531) */
    __u16   opcode;         /* offset 4 */
    __u16   flags;          /* offset 6 */
    __u64   trace_id;       /* offset 8 */
    __u64   timestamp_ns;   /* offset 16 */
    __u64   src_task;       /* offset 24 */
    __u64   dst_task;       /* offset 32 */
    __u32   payload_len;    /* offset 40 */
    __u8    reserved[84];   /* offset 44 */
} __attribute__((packed));
```

**关键约束**：
1. 结构体名称为 `airy_ipc_msg_hdr`（**不是** `airymaxos_ipc_msg_hdr` 或 `are_ipc_msg_header_t`）
2. 使用 `__attribute__((packed))` 对齐（**不是** `aligned(64)` 或 `aligned(128)`）
3. 使用内核 UAPI 类型 `__u32`/`__u16`/`__u64`/`__u8`（**不是** `uint32_t`/`uint16_t` 等）
4. 字段顺序：magic / opcode / flags / trace_id / timestamp_ns / src_task / dst_task / payload_len / reserved[84]
5. 总大小：128 字节
6. **禁止在 agentrt 文档中重新定义此结构体**——应通过 `#include <airymax/ipc.h>` 引用

> **SSoT 声明**：本文档中 IPC 128B 消息头的定义以 AirymaxOS [`50-engineering-standards/120-cross-project-code-sharing.md`](../../AirymaxOS/50-engineering-standards/120-cross-project-code-sharing.md) §Layout C 为单一数据源。任何 agentrt 文档中的 IPC 消息头定义必须与此 SSoT 逐字节一致，或直接 `#include <airymax/ipc.h>` 引用。

### 3.3 任务描述符 magic

```c
/* include/airymax/sched.h */
#define AIRY_TASK_MAGIC   0x41475453u   /* 'AGTS' —— 任务描述符 magic */
```

- **值**：`0x41475453`（ASCII 字符 'A''G''T''S' 的大端表示）
- **用途**：标识 Agent 任务描述符，校验任务描述符完整性
- **注意**：任务描述符**不是** IPC 消息，使用独立 magic 避免混淆
- **SSoT**：`include/airymax/sched.h`，两端共享同一物理头文件

> **agentrt 侧约束**：agentrt 在提交任务时必须设置任务描述符的 `magic` 字段为 `AIRY_TASK_MAGIC`（`0x41475453`）。agentrt SDK 应在 `struct airy_task_desc` 初始化时自动填充此值。

### 3.4 SCHED_EXT 调度类编号

```c
/* include/airymax/sched.h */
/* 复用内核 SCHED_EXT=7 调度类编号，禁止定义 SCHED_AGENT 宏 */
/* SCHED_AGENT 仅作为 sched_ext BPF 策略名称（字符串），非调度类编号 */
```

- **调度类编号**：复用 Linux 内核 `SCHED_EXT=7`
- **禁止**：定义 `SCHED_AGENT` 宏作为调度类编号
- **允许**：`SCHED_AGENT` 仅作为 sched_ext BPF 策略名称（字符串）使用

### 3.5 MAC_MAX_AGENTS

```c
/* include/airymax/sched.h */
#define MAC_MAX_AGENTS    1024   /* 智能体调度数量硬上限 */
```

- **值**：1024
- **基准测试验证**：到 1000 个并发

### 3.6 优先级范围

```c
/* include/airymax/sched.h */
/* 优先级范围 0-139（0 最高，139 最低） */
/* 0-99：实时优先级；100-139：普通优先级 */
```

### 3.7 POSIX capability

```c
/* include/airymax/security_types.h */
/* Linux 6.6 标准 41 个 POSIX capability（编号 0-40，无废弃） */
/* AirymaxOS 自定义 capability 从 41 开始，避免与 Linux 0-40 冲突 */
#define AIRY_CAP_AGENT_SPAWN    41   /* Agent 生成（Airymax 专属） */
#define AIRY_CAP_GPU_SCHED      42   /* GPU 调度（Airymax 专属） */
#define AIRY_CAP_NPU_ACCESS     43   /* NPU 访问（Airymax 专属） */
```

- **Linux 标准 capability**：41 个（0-40），包含 `CAP_PERFMON=38`、`CAP_BPF=39`、`CAP_CHECKPOINT_RESTORE=40`
- **Airymax 专属 capability**：从 41 开始，不占用 Linux 已有编号
- **LSM 钩子数量**：252 个（OLK-6.6 `include/linux/lsm_hook_defs.h` 实际 `LSM_HOOK(` 定义数）

### 3.8 错误码 SSoT

**统一风格**：`AIRY_E*` 前缀 + 负 errno 值

```c
/* agentrt/commons/include/airy_types.h —— airy_err_t 类型 + AIRY_E* POSIX 码（权威源） */
/* agentrt/commons/utils/error/include/error.h —— AIRY_ERR_* 扩展码 + 错误链/i18n 接口 */
typedef int32_t airy_err_t;        /* 错误码类型（int32_t），定义于 airy_types.h:41 */

#define AIRY_EOK        0       /* 成功 */
#define AIRY_EPERM      (-1)    /* 权限不足（对齐 POSIX EPERM） */
#define AIRY_ENOENT     (-2)    /* 不存在（对齐 POSIX ENOENT） */
#define AIRY_ENOMEM     (-12)   /* 内存不足（对齐 POSIX ENOMEM） */
#define AIRY_EINVAL     (-22)   /* 参数无效（对齐 POSIX EINVAL） */
#define AIRY_ETIMEDOUT  (-110)  /* 操作超时（对齐 POSIX ETIMEDOUT） */
/* ...共 30 个 AIRY_E* 宏，airy_err_t 类型定义于 airy_types.h:41 */
```

**禁止的风格**：
- ~~`EAIRY_*`~~（errno 风格前缀）—— 已废弃，统一使用 `AIRY_E*`
- ~~`AIRY_ERR_*`~~（详细前缀）—— 已废弃，统一使用 `AIRY_E*`
- ~~正值错误码~~ —— 统一使用负值

> **SSoT 声明**：错误码的权威定义位于 `agentrt/commons/include/airy_types.h`（`airy_err_t` 类型 + `AIRY_E*` POSIX 码）和 `agentrt/commons/utils/error/include/error.h`（`AIRY_ERR_*` 扩展码）。`airymax/error.h` 为规划中的 [SC] 共享路径，当前尚未创建。所有文档中的错误码描述必须与此 SSoT 一致（方案 A：POSIX errno 负值）。

### 3.9 MemoryRovol L1-L4

```c
/* include/airymax/memory_types.h */
/* MemoryRovol 四级记忆数据结构 + GFP 掩码语义 + PMEM 持久化接口 */
/* 详见 AirymaxOS 120-cross-project-code-sharing.md §3.2 */
```

### 3.10 CoreLoopThree 三阶段

```c
/* include/airymax/cognition_types.h */
/* airy_q16_t: Q16.16 定点数（__s32），内核无 FPU 时使用 */
/* CoreLoopThree 阶段枚举（airy_cog_phase）：PERCEPT / THINK / ACT */
/* Thinkdual 模式枚举（airy_think_mode）：FAST / SLOW */
/* 详见 AirymaxOS 120-cross-project-code-sharing.md §3.4 */
```

### 3.11 BPF struct_ops

```c
/* include/airymax/bpf_struct_ops.h */
/* struct_ops 状态机 + common_value */
/* 详见 AirymaxOS 120-cross-project-code-sharing.md §3.1 */
```

### 3.12 函数前缀三层定义

Airymax 体系遵循**函数前缀与共享层级对齐**原则——函数前缀直接反映其所属的 IRON-9 v2 层级，开发者可凭前缀立即判断符号的共享范围与稳定性边界。

| # | 前缀 | 所属层级 | 用途 | 示例 |
|---|------|---------|------|------|
| 1 | `airy_*` | [SC] 共享契约层 + 应用层 | [SC] 共享头文件中的结构体/宏/枚举 + agentrt/agentrt-linux 应用层函数 | `airy_ipc_msg_hdr`、`airy_sys_task_submit()`、`AIRY_EOK` |
| 2 | `are_*` | 标准 ABI 层 | 对外发布的稳定 ABI 接口（SDK 公开符号） | `are_task_create()`、`are_memory_alloc()` |
| 3 | ~~`agentrt_*`~~ | — | **不使用** | 避免与仓库名 `agentrt` 混淆，统一改用 `airy_*` 或 `are_*` |
| 4 | ~~`agentos_*`~~ | — | **已废弃** | 0.1.1 改名前遗留前缀，已全部替换为 `airy_*` |
| 5 | ~~`airymaxos_*`~~ | — | **已废弃** | AirymaxOS 子仓函数前缀早期方案，已统一改用 `airy_*` |

**关键约束**：

1. **[SC] 共享契约层与应用层共用 `airy_*` 前缀**：[SC] 头文件中的结构体/宏/枚举（如 `airy_ipc_msg_hdr`、`AIRY_EOK`）与应用层函数（如 `airy_sys_task_submit()`）均使用 `airy_` 前缀，体现"共享契约即应用基础"的设计理念。
2. **标准 ABI 层使用 `are_*` 前缀**：对外发布的稳定 ABI 接口（SDK 公开符号）使用 `are_` 前缀（**ARE** = AiRtymax Engine/ABI），与 `airy_*` 区分以明确"对外稳定接口"的边界。
3. **禁止使用 `agentrt_*` 前缀**：仓库名 `agentrt` 是工程组织名称，不应作为函数前缀，避免"仓库名 = 前缀名"的混淆。如需引用仓库本身，使用全称 `agentrt`。
4. **`agentos_*` 与 `airymaxos_*` 已废弃**：0.1.1 完成改名后，这两个前缀不再使用。所有遗留引用已替换为 `airy_*`。**禁止**生成兼容别名头（如 `agentos_compat_aliases.h`），**禁止**双轨制。

> **SSoT 声明**：函数前缀命名规范的权威定义见 AirymaxOS [`50-engineering-standards/10-coding-style/coding_conventions.md` Part IV §4.4.2](../../AirymaxOS/50-engineering-standards/10-coding-style/coding_conventions.md)。agentrt 文档中的前缀描述必须与此 SSoT 一致。

---

## 4. agentrt 侧落地约束

### 4.1 编码约束

agentrt 开发者在编写涉及 [SC] 共享契约层的代码时，必须遵守以下约束：

| # | 约束 | 说明 |
|---|------|------|
| 1 | **使用 `#include <airymax/*.h>`** | 禁止在 agentrt 文档中重新定义 [SC] 头文件中的结构体、宏、枚举 |
| 2 | **使用 UAPI 类型** | 使用 `__u32`/`__u16`/`__u64`/`__u8`，不使用 `uint32_t` 等 C 标准库类型 |
| 3 | **不使用 `float`** | [SC] 头文件中禁止 `float` 类型，需改用定点数（Q16.16）或下沉用户态 |
| 4 | **packed 对齐** | IPC 消息头使用 `__attribute__((packed))`，不使用 `aligned(64)` 或 `aligned(128)` |
| 5 | **结构体命名** | 使用 `airy_` 前缀（如 `airy_ipc_msg_hdr`），不使用 `airymaxos_` 或 `are_` 前缀 |
| 6 | **magic 值一致** | IPC magic = `0x41524531`，任务描述符 magic = `0x41475453` |

### 4.2 文档约束

agentrt 文档在描述 [SC] 共享契约层内容时：

1. **引用而非重定义**：应写"详见 `include/airymax/ipc.h`"或"详见 AirymaxOS `120-cross-project-code-sharing.md`"
2. **标注 [SC]**：在涉及共享契约层内容时标注 `[SC]`
3. **不创建副本定义**：禁止在 agentrt 文档中创建 IPC 消息头、错误码、capability 枚举等 [SC] 内容的独立定义

### 4.3 agentrt 独有内容 [IND]

agentrt 作为用户态跨平台运行时，有以下独有内容（不与 agentrt-linux 共享）：

| # | 独有内容 | 说明 |
|---|---------|------|
| 1 | JSON-RPC 统一入口 `airy_syscall_invoke()` | agentrt 高层 API 风格，非编号 syscall |
| 2 | 跨平台抽象层（POSIX/libc） | 支持 Linux/macOS/Windows |
| 3 | ops 注入机制 | Linux 平台可选的内核加速路径接入 |
| 4 | JSON-RPC 2.0 方法签名 | `sched.set_priority` 等命名空间方法 |
| 5 | 跨平台 IPC 实现 | 基于共享内存 + 信号量（非 io_uring） |

---

## 5. 跨端一致性校验

### 5.1 CI 校验机制

两端 CI 应包含以下跨端一致性校验：

| # | 校验项 | 校验方法 |
|---|--------|---------|
| 1 | IPC magic 值一致 | Grep 两端代码中的 `0x41524531`，确保值一致 |
| 2 | 任务描述符 magic 一致 | Grep 两端代码中的 `0x41475453`，确保值一致 |
| 3 | MAC_MAX_AGENTS 一致 | 两端均为 1024 |
| 4 | SCHED_EXT 编号一致 | 两端均复用 `SCHED_EXT=7`，禁止 `SCHED_AGENT` 宏 |
| 5 | capability 枚举一致 | 两端 `security_types.h` 为同一物理文件 |
| 6 | 错误码值一致 | 两端 `error.h` 为同一物理文件 |
| 7 | IPC 消息头布局一致 | 两端 `ipc.h` 为同一物理文件 |

### 5.2 变更流程

[SC] 共享契约层的变更必须遵循以下流程：

1. **提议**：在 AirymaxOS `120-cross-project-code-sharing.md` 中提出变更提案
2. **双向评审**：agentrt 和 agentrt-linux 双方维护者评审
3. **同步修改**：修改 `include/airymax/*.h` 物理头文件
4. **CI 验证**：两端 CI 全量通过
5. **文档更新**：更新两端相关文档

---

## 6. 与 AirymaxOS 的引用关系

| 主题 | SSoT 归属 | 权威文档路径 | agentrt 文档引用方式 |
|------|----------|-------------|-------------------|
| IRON-9 v2 共享模型 | AirymaxOS | `50-engineering-standards/120-cross-project-code-sharing.md` | 本文档（06-iron9-shared-model.md）为 agentrt 侧说明 |
| IPC 128B 消息头 | AirymaxOS | `120-cross-project-code-sharing.md` §Layout C | `#include <airymax/ipc.h>` |
| 错误码体系 | AirymaxOS | `agentrt/commons/include/airy_types.h` + `agentrt/commons/utils/error/include/error.h`（[SC] 实际权威源；`airymax/error.h` 为规划路径，尚未创建） | `#include "airy_types.h"` + `#include "error.h"` |
| capability 安全模型 | AirymaxOS | `110-security/03-capability-model.md` + `include/airymax/security_types.h` | 引用 AirymaxOS 文档 |
| 调度参数 | AirymaxOS | `include/airymax/sched.h`（[SC] 头文件） | `#include <airymax/sched.h>` |
| 系统调用语义映射 | AirymaxOS | `30-interfaces/05-syscall-semantic-mapping.md` | 引用 AirymaxOS 文档 |
| C 编码风格 | AirymaxOS | `50-engineering-standards/10-coding-style/C_coding_style_standard.md` | agentrt `50-engineering-standards/10-coding-style/C_coding_style_standard.md` SSoT 引用 |
| C/C++ 安全编码 | AirymaxOS | `50-engineering-standards/10-coding-style/C_Cpp_secure_coding_standard.md` | agentrt `50-engineering-standards/10-coding-style/C_Cpp_secure_coding_standard.md` SSoT 引用 |
| Rust 编码风格 | AirymaxOS | `50-engineering-standards/10-coding-style/Rust_coding_style_standard.md` | agentrt `50-engineering-standards/10-coding-style/Rust_coding_style_standard.md` SSoT 引用 |
| 日志规范 | AirymaxOS | `50-engineering-standards/10-coding-style/` 日志部分 | agentrt `50-engineering-standards/10-coding-style/log_standard.md` SSoT 引用 |

---

## 7. 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| 0.1.1 | 2026-07-10 | 初始草案：IRON-9 v2 三层共享模型说明、10 个 [SC] 头文件清单、核心 magic 值定义、agentrt 侧落地约束、跨端一致性校验机制 |

---

> **文档结束** | Airymax IRON-9 v2 跨端共享模型 | 2026-07-10 | SPHARX Ltd
