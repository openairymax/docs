Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# seL4 风格 Capability 安全模型
> **文档定位**：agentrt-linux（AirymaxOS）Capability 安全模型的完整工程契约，定义 CNode/MDB 数据模型、派生算法、POSIX capability 集成、令牌生命周期、Cupolas blob 布局、策略裁决与 Vault backend 抽象；并落地 A-IPC Capability Folding 单平面架构下的 Badge 64-bit Native Word 模型与 fastpath C-S9 内联校验\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：seL4 `src/object/cnode.c`（CNode 操作）+ `src/object/cnode.c:cteRevoke`（递归撤销）+ Linux 6.6 `security/commoncap.c`（POSIX capability）+ agentrt Cupolas 权限模型\
> **文档性质**：实现方案文档（非设计文档）。本契约在 [01-lsm-framework.md](01-lsm-framework.md) 第 7 章 LSM 与 capability 共存的基础上，补充完整的 capability 数据模型、派生算法、生命周期与接口定义\
> **设计参考**：seL4 `src/object/cnode.c`（CNode mint/mintcopy/move/copy/revoke/delete）+ seL4 `src/kernel/mdb.c`（MDB 派生链）+ 主流 Linux 发行版 Linux 6.6 内核基线 `security/commoncap.c`（POSIX cap 检查）+ `include/linux/cred.h`（credential 结构）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **agentrt-linux Capability 安全模型** 的唯一权威源。CNode/CSpace 数据模型、MDB 派生链、POSIX 41 ID 枚举、令牌 7 状态生命周期、Cupolas blob 四类布局、4 值策略裁决、Vault backend 抽象、**Capability Folding Badge 64-bit Native Word 模型**、**fastpath C-S9 内联校验**、**`agent_caps[1024]` 静态数组** 均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 capability 数据模型与 Badge 校验机制。
>
> **v1.0.1 Capability Folding 集成声明**（A-IPC 第一块基石）：自 v1.0.1 起，agentrt-linux 的 seL4 风格 capability 校验路径从"独立前置 `airy_cap_check()` + per-cpu 缓存"重构为 **fastpath C-S9 内联 Badge 校验**。物理载体是 [SC] `ipc.h` Layout C v4 消息头 offset 40-47 的 `capability_badge` 字段（64-bit Native Word：Epoch 16位 + Random Tag 32位 + Perms 16位）；执行点是 fastpath `airy_cap_badge_ok()`（~10ns，3 个 READ_ONCE + 位运算 + 比较）；存储是 `agent_caps[1024]` 内核静态数组（16KB，sec_d 唯一写者）。6 条硬约束 H1-H6 不可妥协。详见 §13 与 [10-unify-design.md §8](../10-architecture/10-unify-design.md)。

---

## 1. 概述

### 1.1 为什么选择 seL4 风格

agentrt-linux 选择 seL4 风格的 capability 安全模型，而非传统 ACL（访问控制列表）或纯 POSIX capability，原因在于：

| 维度 | ACL | POSIX capability | seL4 capability | agentrt-linux 选择 |
|------|-----|-----------------|-----------------|-------------------|
| 权限传递 | 通过组/用户 | 进程位图 | 不可伪造令牌传递 | seL4（Agent 间传递） |
| 权限撤销 | 困难（需遍历 ACL） | 不支持运行时撤销 | 递归级联撤销 | seL4（Agent 终止时） |
| 权限粒度 | 对象级 | 41 个粗粒度 cap | 对象 + 权限掩码 | seL4 + POSIX 混合 |
| 最小权限 | 难以实现 | 进程级 | 令牌级 | seL4（每个操作独立令牌） |
| 形式化验证 | 无 | 无 | 有（seL4 已验证） | seL4（参考验证方法） |

### 1.2 混合模型设计

agentrt-linux 采用 **seL4 风格 + POSIX 混合** capability 模型：

- **seL4 风格**：用于 Agent 间的细粒度权限传递与撤销（令牌级、不可伪造、递归撤销）
- **POSIX capability**：用于传统 Linux 兼容性（41 个标准 cap，进程级位图）
- **LSM 桥接**：capability 作为 `LSM_ORDER_FIRST` 第一个 LSM，与 Landlock/Cupolas 共存

### 1.3 设计目标

1. **不可伪造**：capability 令牌由内核生成，用户态只持有 opaque handle，无法伪造
2. **最小权限**：每个安全敏感操作需独立申请 capability 令牌，而非进程级全权
3. **递归撤销**：Agent 终止时递归撤销所有派生 capability，无权限残留
4. **IPC 传递**：capability 可通过 IPC 消息跨进程传递（`AIRY_IPC_F_CAP_CARRY`）
5. **审计追踪**：所有 capability 操作（申请/派生/撤销/传递）可审计

### 1.4 在安全体系中的位置

对齐 [README.md](README.md) 第 1.1 节安全体系分层，capability 位于 L3 层：

| 层级 | 类型 | 机制 | 与 capability 的关系 |
|------|------|------|---------------------|
| L1 | LSM 框架 | `security_hook_heads` | capability 注册为 LSM |
| L2 | Landlock | 用户态沙箱 | capability 之后执行 |
| **L3** | **capability** | **seL4 风格 + POSIX** | **LSM_ORDER_FIRST，永远第一** |
| L4 | 模块签名 | eBPF 签名 | 独立于 capability |
| L7 | Cupolas | Agent 行为约束 | 基于 capability 构建 |

---

## 2. Capability 数据模型

### 2.1 CNode（Capability Node）结构

借鉴 seL4 `src/object/cnode.c` 的 CNode 设计，agentrt-linux 的 capability 存储在 CNode 表中：

```c
/**
 * struct airy_cnode - Capability Node（单个 capability 槽位）
 *
 * @cap_type:    capability 类型（AIRY_CAP_TYPE_*）
 * @cap_id:      capability 唯一 ID（内核分配，不可伪造）
 * @badge:       权限掩码（位图，标识具体权限）—— v1.0.1 起为 64-bit Native Word
 *               （Epoch<<48 | RandomTag<<16 | Perms），详见 §2.5
 * @randtag:     per-Agent 随机标签（32 位，sec_d 编译时生成，防伪造）
 * @owner_agent:  持有此 capability 的 Agent ID
 * @parent:      父 capability ID（派生链，MDB）
 * @children:     子 capability 列表头（MDB 派生链）
 * @state:       capability 状态（ACTIVE/REVOKED/EXPIRED）
 * @refcount:    引用计数
 * @expires_ns:  过期时间戳（0 = 永不过期）
 *
 * 借鉴 seL4 cte_t（capability table entry）设计。
 * CNode 表是 Agent 的 capability 空间（CSpace），类似 seL4 CSpace。
 *
 * v1.1 变更：新增 @randtag 字段，作为 Capability Folding Badge 64-bit
 * Native Word 的防伪造组件。fastpath C-S9 校验时通过 READ_ONCE() 读取
 * 此字段与消息头 capability_badge 中的 Random Tag 比对。
 */
struct airy_cnode {
    uint8_t  cap_type;
    uint32_t cap_id;
    uint64_t badge;                /* v1.1: 64-bit Native Word（Epoch|RandTag|Perms）*/
    uint32_t randtag;              /* v1.0.1 新增：per-Agent 随机标签，sec_d 编译生成 */
    uint32_t owner_agent;
    uint32_t parent;
    struct list_head children;      /* MDB 派生链 */
    struct list_head sibling;      /* 兄弟节点 */
    uint8_t  state;
    atomic_t refcount;
    uint64_t expires_ns;
} __attribute__((aligned(64)));
```

### 2.2 Capability 类型枚举

```c
/**
 * enum airy_cap_type - capability 类型
 *
 * 对齐 security_types.h [SC] 共享头文件中的类型定义。
 */
enum airy_cap_type {
    AIRY_CAP_TYPE_ENDPOINT    = 0,   /* IPC 端点 capability */
    AIRY_CAP_TYPE_TASK        = 1,   /* Agent 任务 capability */
    AIRY_CAP_TYPE_MEMORY      = 2,   /* 内存访问 capability */
    AIRY_CAP_TYPE_ROVOL       = 3,   /* 记忆卷载 capability */
    AIRY_CAP_TYPE_SCHED       = 4,   /* 调度 capability */
    AIRY_CAP_TYPE_FILE        = 5,   /* 文件系统 capability */
    AIRY_CAP_TYPE_NETWORK     = 6,   /* 网络访问 capability */
    AIRY_CAP_TYPE_WASM        = 7,   /* Wasm 模块 capability */
    AIRY_CAP_TYPE_CAP_MGMT    = 8,   /* capability 管理 capability（元 capability） */
    AIRY_CAP_TYPE_MAX,
};
```

### 2.3 MDB（Mask Domain Database）派生链

借鉴 seL4 `src/kernel/mdb.c`，capability 的派生关系通过 MDB 维护：

```mermaid
graph TD
    ROOT[Root Capability<br/>cap_id=1, owner=Agent_A<br/>badge=0xFF（全部权限）]

    ROOT --> D1[Derived Capability<br/>cap_id=2, owner=Agent_B<br/>badge=0x0F（受限权限）<br/>parent=1]
    ROOT --> D2[Derived Capability<br/>cap_id=3, owner=Agent_C<br/>badge=0x33（自定义权限）<br/>parent=1]

    D1 --> D3[Derived Capability<br/>cap_id=4, owner=Agent_D<br/>badge=0x03（进一步受限）<br/>parent=2]

    style ROOT fill:#e1f5fe,stroke:#0288d1
    style D1 fill:#c8e6c9,stroke:#2e7d32
    style D2 fill:#c8e6c9,stroke:#2e7d32
    style D3 fill:#fff3e0,stroke:#e65100
```

**MDB 派生链规则**：
1. 每个派生 capability 记录 `parent` 指向源 capability
2. 父 capability 维护 `children` 链表，记录所有直接派生
3. 撤销父 capability 时，递归遍历 `children` 链表撤销所有派生
4. 派生 capability 的 `badge` 必须 ≤ 父 capability 的 `badge`（权限只减不增）

### 2.4 CSpace（Capability Space）

每个 Agent 拥有独立的 CSpace（capability 空间），是 CNode 的集合。**v1.0.1 起，CSpace 的物理存储从 radix tree 重构为 `agent_caps[1024]` 内核静态数组**——这是 Capability Folding 单平面架构的硬约束 H4 落地（详见 §13.2）：

```c
/**
 * struct airy_cspace - Agent 的 Capability Space
 *
 * @agent_id:     所属 Agent ID
 * @slots:        CNode 槽位数组（v1.1: 静态数组，索引 = agent_id）
 * @root_cap:     根 capability ID
 * @total_caps:   当前活跃 capability 数
 * @max_caps:     最大 capability 数（默认 1024）
 *
 * v1.1 变更（Capability Folding 集成）:
 *   - 物理存储从 `struct radix_tree_root slots` 重构为
 *     `agent_caps[1024]` 内核静态数组（16KB，per-Agent 索引）
 *   - 唯一写者: sec_d（security daemon）通过 airy_sys_call + COMPILE_BADGE
 *   - 唯一读者: A-IPC fastpath C-S9 Badge 校验（airy_cap_badge_ok）
 *   - 索引复杂度: O(1)（数组索引 vs radix tree O(log n)）
 *   - 内存占用: 16KB 静态分配（vs radix tree 动态分配）
 *   - 校验延迟: ~10ns（vs radix tree ~50-100ns）
 *
 * 设计依据: Capability Folding H4 硬约束——agentrt-linux 内核 Badge
 * 由 sec_d 编译、fastpath C-S9 校验。静态数组消除动态分配开销与
 * RCU 同步开销，是 fastpath 100 行代码 CBMC 全函数验证的前提。
 *
 * 借鉴 seL4 CSpace 的树状组织结构（逻辑视图），
 * 物理视图采用静态数组实现（工程优化）。
 */
struct airy_cspace {
    uint32_t agent_id;
    struct airy_cnode *slots;      /* v1.1: 指向 agent_caps[agent_id] 静态槽位 */
    uint32_t root_cap;
    atomic_t total_caps;
    uint32_t max_caps;
};

/* v1.0.1 新增: 内核静态数组——Capability Folding H4 物理载体
 * 物理宿主: kernel/security/airy/capability.c
 * 大小: 1024 × sizeof(struct airy_cap_slot) = 16KB
 * 写者: sec_d（通过 airy_sys_call + COMPILE_BADGE 子命令）
 * 读者: fastpath C-S9（airy_cap_badge_ok()）
 * 同步: WRITE_ONCE/READ_ONCE（无锁，sec_d 单写者 + fastpath 多读者）
 */
#define AIRY_CAP_MAX_AGENTS  1024

struct airy_cap_slot {
    uint32_t randtag;              /* per-Agent 随机标签，sec_d 编译时生成 */
    uint16_t perms;                /* 当前有效权限位图（与 Badge.Perms 同源）*/
    uint8_t  state;                /* AIRY_CAP_STATE_* */
    uint8_t  reserved[3];          /* 对齐到 12B */
};

extern struct airy_cap_slot agent_caps[AIRY_CAP_MAX_AGENTS];

/* 全局 Epoch（1 个 atomic_t，撤销时 1 行 atomic_inc 立即失效所有 Badge）*/
extern atomic_t airy_cap_global_epoch;

static inline uint16_t airy_cap_epoch_get(void)
{
    return (uint16_t)atomic_read(&airy_cap_global_epoch);
}
```

### 2.5 Badge 64-bit Native Word 模型（v1.0.1 新增）

**这是 Capability Folding 的物理载体**——fastpath C-S9 校验的就是这 64 位。Badge 的内部结构遵循 Layout C v4（[SC] `ipc.h` 消息头 offset 40-47）：

```
63                    48 47                16 15            0
┌──────────────────────┬─────────────────────┬──────────────┐
│   Epoch (16 bits)    │  Random Tag (32 bits)│ Perms (16 bits)│
└──────────────────────┴─────────────────────┴──────────────┘

Epoch (16 bits):       全局代际快照，撤销时 atomic_inc 立即失效
                       所有已发出 Badge（1 行代码，无 drain、无 bitmap）
Random Tag (32 bits):  per-Agent 随机标签，sec_d 编译时生成，防伪造
                       （agent_caps[agent_id].randtag，READ_ONCE 读取）
Perms (16 bits):       权限位图，7 个 opcode 对应 7 个权限位
                       （SEND/RECV/CALL/GRANT/REVOKE/FREEZE/BATCH）
```

**访问宏定义**（[SC] `security_types.h` 共享）：

```c
/* Badge 字段提取宏——CPU 单条指令位运算，~1ns */
#define AIRY_BADGE_EPOCH(badge)    ((uint16_t)((badge) >> 48))
#define AIRY_BADGE_RANDTAG(badge)  ((uint32_t)((badge) >> 16 & 0xFFFFFFFF))
#define AIRY_BADGE_PERMS(badge)    ((uint16_t)((badge) & 0xFFFF))

/* Badge 编译宏——仅 sec_d 使用 */
#define AIRY_BADGE_COMPILE(epoch, randtag, perms) \
    ((((uint64_t)(epoch)) << 48) | \
     (((uint64_t)(randtag)) << 16) | \
     (((uint64_t)(perms)) & 0xFFFF))

/* Perms 权限位（与 §2.6 opcode 表对齐）*/
#define AIRY_CAP_PERM_SEND    0x0001
#define AIRY_CAP_PERM_RECV    0x0002
#define AIRY_CAP_PERM_CALL    0x0004
#define AIRY_CAP_PERM_GRANT   0x0008
#define AIRY_CAP_PERM_REVOKE  0x0010
#define AIRY_CAP_PERM_FREEZE  0x0020
#define AIRY_CAP_PERM_BATCH   0x0040

/* 权限检查宏——fastpath C-S9 内联使用 */
static inline bool airy_cap_has_perm(uint16_t perms, uint16_t opcode)
{
    return (perms & (1u << (opcode & 0x000F))) != 0;
}
```

**安全性分析**：

| 攻击向量 | 防御机制 | 校验点 |
|---------|---------|--------|
| 伪造 Badge | Random Tag 32 位（per-Agent，sec_d 编译时生成，攻击者不可预测） | C-S9 #2 READ_ONCE 比对 |
| 重放旧 Badge | Epoch 16 位（全局 atomic_inc，撤销立即失效） | C-S9 #1 atomic_read 比对 |
| 越权操作 | Perms 16 位（7 opcode 对应 7 位权限位） | C-S9 #3 位运算检查 |
| Badge 泄露 | sec_d 编译时绑定 Agent ID，跨 Agent 不可复用 | agent_caps[agent_id].randtag 比对 |
| 暴力枚举 | 2^48（Random Tag + Epoch）组合空间，单次校验 ~10ns | 不可行（10^15 ns ≈ 31 年）|

**Epoch 溢出处理**：16 位 Epoch 理论上 65535 次撤销后溢出。实际场景中，agentrt-linux 单节点 Agent 终止-重启频率 < 1 次/秒，溢出周期 > 18 小时。溢出时 sec_d 重新编译所有 Badge（全量刷新），不影响系统可用性。

### 2.6 Badge 编译与校验完整实现（v1.0.1 新增）

本节是 `airy_cap_badge_compile()` / `airy_cap_has_perm()` / `airy_op_required_perms()` 三个函数的**唯一权威实现源**（SSoT）。物理宿主为 `kernel/security/airy/capability.c`。

#### 2.6.1 opcode → required_perms 映射表（SSoT）

`airy_op_required_perms()` 定义了每个 opcode 所需的最小权限位。该映射表是 Capability Folding 的权限契约核心，fastpath C-S9.PERMS 通过此表判断 Badge 的 Perms 位是否满足 opcode 要求。

```c
/* kernel/security/airy/capability.c —— opcode → required_perms 映射表（SSoT）
 *
 * 设计原则：1 opcode → 1 required_perms 位（严格 1:1 映射）
 *   - SEND         → PERM_SEND     (0x0001)
 *   - RECV         → PERM_RECV     (0x0002)
 *   - SEND_BATCH   → PERM_BATCH    (0x0040)  ← 批量发送独立权限
 *   - CANCEL       → PERM_SEND     (0x0001)  ← 取消复用 SEND 权限
 *   - FREEZE       → PERM_FREEZE   (0x0020)  ← 特权操作
 *   - CAP_REQUEST  → 0（自举路径，无需权限）
 *   - CAP_RESPONSE → 0（sec_d 回复，无需权限）
 *
 * 行业对照：类似 Linux capable(opcode) → capable(CAP_SYS_ADMIN) 的映射，
 *           但 AirymaxOS 采用 1:1 精确映射，避免过度授权。
 */

static const uint16_t airy_op_required_perms_table[8] = {
    /* [0] AIRY_IPC_OP_SEND         0x0001 */ AIRY_CAP_PERM_SEND,
    /* [1] AIRY_IPC_OP_RECV         0x0002 */ AIRY_CAP_PERM_RECV,
    /* [2] AIRY_IPC_OP_SEND_BATCH   0x0003 */ AIRY_CAP_PERM_BATCH,
    /* [3] AIRY_IPC_OP_CANCEL       0x0004 */ AIRY_CAP_PERM_SEND,    /* 复用 SEND */
    /* [4] AIRY_IPC_OP_FREEZE       0x0005 */ AIRY_CAP_PERM_FREEZE,
    /* [5] reserved                 0x0006 */ 0xFFFF,                 /* 非法 opcode */
    /* [6] AIRY_IPC_OP_CAP_REQUEST  0x0010 */ 0,                      /* 自举，无需权限 */
    /* [7] AIRY_IPC_OP_CAP_RESPONSE 0x0011 */ 0,                      /* sec_d 回复 */
};

/**
 * airy_op_required_perms - 获取 opcode 所需的最小权限位
 * @opcode: IPC 操作码（AIRY_IPC_OP_*）
 *
 * 返回: 所需权限位掩码；0 表示无需权限（如 CAP_REQUEST）；0xFFFF 表示非法 opcode。
 *
 * 该函数是 C-S9.PERMS 校验的依据，fastpath 内联调用。
 */
static inline uint16_t airy_op_required_perms(uint16_t opcode)
{
    /* opcode 紧凑索引映射（0x0001-0x0005 → 0-4，0x0010-0x0011 → 6-7）*/
    uint8_t idx;

    if (opcode <= 0x0005)
        idx = opcode - 1;              /* SEND/CANCEL/FREEZE 等 → 0-4 */
    else if (opcode >= 0x0010 && opcode <= 0x0011)
        idx = (opcode - 0x0010) + 6;   /* CAP_REQUEST/RESPONSE → 6-7 */
    else
        return 0xFFFF;                 /* 非法 opcode */

    return airy_op_required_perms_table[idx];
}
```

#### 2.6.2 airy_cap_has_perm() 完整实现

```c
/**
 * airy_cap_has_perm - 检查 Badge 的 Perms 位是否满足 opcode 所需权限
 * @perms:  Badge 的 Perms 字段（16-bit）
 * @opcode: IPC 操作码（AIRY_IPC_OP_*）
 *
 * C-S9.PERMS 检查点调用此函数。设计原则：
 *   - 严格 "包含" 语义：(perms & required) == required
 *   - 超集允许：Badge 可携带比所需更多的权限（mint 派生后可能富余）
 *   - 非法 opcode → 拒绝（required = 0xFFFF，任何 perms 都不满足）
 *
 * 返回: true 权限满足；false 权限不足或 opcode 非法。
 *
 * 性能：1 次表查找 + 1 次位运算 + 1 次比较，~2ns（热缓存）。
 */
static inline bool airy_cap_has_perm(uint16_t perms, uint16_t opcode)
{
    uint16_t required = airy_op_required_perms(opcode);

    /* 非法 opcode（required == 0xFFFF）→ 任何 perms 都不满足 */
    if (unlikely(required == 0xFFFF))
        return false;

    /* 自举 opcode（required == 0）→ 总是满足（C-S9 入口已处理 CAP_REQUEST）*/
    if (likely(required == 0))
        return true;

    /* 严格包含语义：perms 必须包含 required 的所有位 */
    return (perms & required) == required;
}
```

#### 2.6.3 airy_cap_badge_compile() 完整实现（sec_d 唯一写者）

`airy_cap_badge_compile()` 是 sec_d 的核心函数，编译 Badge 并写入 `agent_caps[]` 静态数组。**整个系统仅此函数可以写入 `agent_caps[]`**（H4 单写者约束）。

```c
/* kernel/security/airy/capability.c —— sec_d Badge 编译（v1.1 SSoT）
 *
 * 物理宿主: kernel/security/airy/capability.c
 * 调用者:   仅 sec_d（通过 airy_sys_call + COMPILE_BADGE 子命令）
 * 同步:     mutex 串行化（单写者保证，见 §2.6.4 并发串行化机制）
 */

/**
 * airy_cap_badge_compile - 编译 Badge 并写入 agent_caps[]（sec_d 唯一写者）
 * @agent_id: 目标 Agent ID（0 ~ AIRY_CAP_MAX_AGENTS-1）
 * @perms:    权限位掩码（AIRY_CAP_PERM_* 的按位或）
 *
 * 此函数执行以下步骤：
 *   1. 参数校验（agent_id 范围、perms 合法性）
 *   2. 获取全局 mutex（串行化，见 §2.6.4）
 *   3. 读取当前全局 Epoch
 *   4. 生成 32-bit Random Tag（get_random_bytes_wait）
 *   5. 组合 64-bit Badge：epoch<<48 | randtag<<16 | perms
 *   6. WRITE_ONCE 写入 agent_caps[agent_id]（更新 randtag + perms + state）
 *   7. 释放 mutex
 *   8. 返回 Badge opaque handle（供 Macro-Supervisor 持有）
 *
 * 返回: ≥0 成功（Badge opaque handle）；<0 错误码：
 *   -EINVAL: agent_id 越界或 perms 非法
 *   -ENOMEM: get_random_bytes_wait 失败（极罕见）
 *
 * 性能：~2μs（含 get_random_bytes_wait 平均 1.5μs + mutex + WRITE_ONCE）
 */
static DEFINE_MUTEX(airy_cap_compile_mutex);  /* §2.6.4 串行化机制 */

int airy_cap_badge_compile(uint32_t agent_id, uint16_t perms)
{
    uint16_t epoch;
    uint32_t randtag;
    uint64_t badge;
    unsigned long flags;
    int ret = 0;

    /* 1. 参数校验 */
    if (unlikely(agent_id >= AIRY_CAP_MAX_AGENTS))
        return -EINVAL;
    /* perms 仅允许 AIRY_CAP_PERM_* 位（0x0063 = SEND|RECV|CALL|FREEZE|BATCH）*/
    if (unlikely(perms & ~(AIRY_CAP_PERM_SEND | AIRY_CAP_PERM_RECV |
                           AIRY_CAP_PERM_CALL | AIRY_CAP_PERM_GRANT |
                           AIRY_CAP_PERM_REVOKE | AIRY_CAP_PERM_FREEZE |
                           AIRY_CAP_PERM_BATCH)))
        return -EINVAL;

    /* 2. 获取全局 mutex（串行化 Badge 编译，保证 agent_caps[] 写入顺序）*/
    mutex_lock(&airy_cap_compile_mutex);

    /* 3. 读取当前全局 Epoch（atomic_read，~1ns）*/
    epoch = (uint16_t)atomic_read(&airy_cap_global_epoch);

    /* 4. 生成 32-bit Random Tag（防伪造，~1.5μs 平均）
     *    使用 get_random_bytes_wait 确保熵充足（不使用 get_random_bytes_arch）
     *    32-bit 空间 2^32 = 42 亿，per-Agent 独立，碰撞概率 1/2^32（见 §13.2）
     */
    ret = get_random_bytes_wait(&randtag, sizeof(randtag));
    if (unlikely(ret < 0)) {
        mutex_unlock(&airy_cap_compile_mutex);
        return -ENOMEM;
    }

    /* 5. 组合 64-bit Badge */
    badge = AIRY_BADGE_COMPILE(epoch, randtag, perms);

    /* 6. WRITE_ONCE 写入 agent_caps[agent_id]（无锁多读者安全）
     *    更新 randtag + perms + state，原子性由 WRITE_ONCE 保证
     *    读者（fastpath C-S9）使用 READ_ONCE 配对访问
     */
    WRITE_ONCE(agent_caps[agent_id].randtag, randtag);
    WRITE_ONCE(agent_caps[agent_id].perms, perms);
    WRITE_ONCE(agent_caps[agent_id].state, AIRY_CAP_STATE_ACTIVE);

    /* 7. 释放 mutex */
    mutex_unlock(&airy_cap_compile_mutex);

    /* 8. 审计日志（A-ULP）*/
    airy_audit_emit(AGENT_CAP_COMPILE, agent_id, badge);

    /* 9. 返回 Badge opaque handle（供 Macro-Supervisor 持有）*/
    return (int)badge;
}
```

#### 2.6.4 Badge 编译并发串行化机制

**设计背景**：当 10000 个 Agent 并发启动时，可能同时向 sec_d 请求 Badge 编译。若无串行化机制，会导致：
1. `agent_caps[]` 写入竞争（WRITE_ONCE 不能保证多字段写入原子性）
2. Random Tag 生成竞争（`get_random_bytes_wait` 内部有锁，但外部仍需串行化）
3. Epoch 读取与写入之间的 TOCTOU race

**串行化方案**：使用 `DEFINE_MUTEX(airy_cap_compile_mutex)` 全局 mutex，保证 `airy_cap_badge_compile()` 串行执行。

**性能分析**：
- 单次 Badge 编译：~2μs（含 `get_random_bytes_wait` 平均 1.5μs）
- 10000 个并发请求：10000 × 2μs = 20ms（串行化总耗时）
- 单 Agent 感知延迟：平均 10ms（排队等待）

**为什么 20ms 可接受**：
1. **启动场景**：Agent 启动是低频事件（系统启动或故障恢复），非热路径
2. **fastpath 零影响**：Badge 编译在 sec_d slowpath，不影响 fastpath C-S9 ~10ns
3. **批量编译优化**：sec_d 支持 `AIRY_IPC_OP_SEND_BATCH` 批量编译（见 §2.6.5），10000 个请求可合并为单次 mutex 获取 + 批量 `get_random_bytes`（一次填充 40KB 随机池）

**替代方案对比**：

| 方案 | 优点 | 缺点 | AirymaxOS 选择 |
|-----|------|------|---------------|
| 全局 mutex（已选） | 简单、正确性易证明 | 10000 并发需 20ms | ✅ 0.1.1 阶段 |
| per-Agent spinlock | 更细粒度 | 10000 个 spinlock 内存开销 | ❌ 过度设计 |
| 无锁 + per-CPU Random Tag 池 | 最快 | 实现复杂、调试困难 | ⏳ 1.0.1 优化 |
| RCU + 世代标签 | 读多写少最优 | Badge 编译是写操作，RCU 不适用 | ❌ 场景不匹配 |

**结论**：0.1.1 阶段采用全局 mutex 串行化，10000 并发请求 20ms 串行化延迟在可接受范围内（非热路径）。1.0.1 阶段可考虑 per-CPU Random Tag 池优化。

#### 2.6.5 批量编译优化（10000+ Agent 场景）

针对大规模 Agent 启动场景（如 10000+ Agent 同时启动），sec_d 提供 `airy_cap_badge_compile_batch()` 批量编译接口：

```c
/**
 * airy_cap_badge_compile_batch - 批量编译 Badge（10000+ Agent 场景优化）
 * @requests:  编译请求数组（agent_id + perms）
 * @count:     请求数量（≤ AIRY_CAP_MAX_AGENTS）
 * @badges_out: 返回的 Badge opaque handle 数组
 *
 * 优化点：
 *   1. 单次 mutex 获取（避免 10000 次 mutex lock/unlock）
 *   2. 批量 get_random_bytes（一次填充 40KB 随机池，比 10000 次单独调用快 5x）
 *   3. 单次 audit_emit（合并日志）
 *
 * 性能：10000 个 Badge 批量编译 ~5ms（vs 串行 20ms，提速 4x）
 *
 * 返回: 0 成功；<0 错误码
 */
int airy_cap_badge_compile_batch(const struct airy_cap_compile_req *requests,
                                  size_t count, uint64_t *badges_out)
{
    uint32_t *randtags;
    uint16_t epoch;
    int ret = 0;
    size_t i;

    if (unlikely(count == 0 || count > AIRY_CAP_MAX_AGENTS))
        return -EINVAL;

    /* 1. 单次 mutex 获取 */
    mutex_lock(&airy_cap_compile_mutex);
    epoch = (uint16_t)atomic_read(&airy_cap_global_epoch);

    /* 2. 批量生成 Random Tag（一次填充 count×4 字节随机池）*/
    randtags = kmalloc_array(count, sizeof(uint32_t), GFP_KERNEL);
    if (unlikely(!randtags)) {
        mutex_unlock(&airy_cap_compile_mutex);
        return -ENOMEM;
    }
    ret = get_random_bytes_wait(randtags, count * sizeof(uint32_t));
    if (unlikely(ret < 0)) {
        kfree(randtags);
        mutex_unlock(&airy_cap_compile_mutex);
        return -ENOMEM;
    }

    /* 3. 批量编译 + 写入 agent_caps[] */
    for (i = 0; i < count; i++) {
        uint32_t agent_id = requests[i].agent_id;
        uint16_t perms = requests[i].perms;
        uint64_t badge;

        if (unlikely(agent_id >= AIRY_CAP_MAX_AGENTS)) {
            ret = -EINVAL;
            break;
        }

        badge = AIRY_BADGE_COMPILE(epoch, randtags[i], perms);
        WRITE_ONCE(agent_caps[agent_id].randtag, randtags[i]);
        WRITE_ONCE(agent_caps[agent_id].perms, perms);
        WRITE_ONCE(agent_caps[agent_id].state, AIRY_CAP_STATE_ACTIVE);
        badges_out[i] = badge;
    }

    /* 4. 单次审计日志（批量合并）*/
    airy_audit_emit_batch(AGENT_CAP_COMPILE_BATCH, count, badges_out);

    kfree(randtags);
    mutex_unlock(&airy_cap_compile_mutex);
    return ret;
}
```

---

## 3. Capability 派生模型

### 3.1 四种派生操作

对齐 `security_types.h`（[SC] 共享头文件）定义的 capability 派生模型：

| 操作 | 语义 | badge 变化 | 适用场景 |
|------|------|-----------|---------|
| **mint** | 缩减权限派生（不复制，原 cap 保留） | 子 badge = 父 badge & mask | 将全权 cap 缩减为只读 cap |
| **mintcopy** | 复制 + 缩减（生成新 cap，原 cap 保留） | 子 badge = 父 badge & mask | 传递受限 cap 给其他 Agent |
| **derive** | 派生（不缩减，全权继承） | 子 badge = 父 badge | 将 cap 委托给子任务 |
| **revoke** | 递归撤销（撤销所有派生） | 子 cap 全部失效 | Agent 终止时清理 |

### 3.1.x seL4 完整派生操作映射（v1.1 扩展）

agentrt-linux 借鉴 seL4 的 7 种核心 CNode 派生操作（`src/object/cnode.c`），当前 v1.1 实现 4 种，1.0.1 扩展至 7 种：

| seL4 操作 | agentrt-linux 映射 | 状态 | 说明 |
|-----------|-------------------|------|------|
| **Copy** | `airy_cap_copy()` | v1.1 | 复制 capability（新 badge ≤ 旧 badge） |
| **Mint** | `airy_cap_mint()` | v1.1 | 铸造 capability（可附加新 badge 数据） |
| **Move** | `airy_cap_move()` | 1.0.1 | 移动 capability（改变 CNode 位置，badge 不变） |
| **Mutate** | `airy_cap_mutate()` | 1.0.1 | 变异 capability（修改权限位，不重新生成 RandomTag） |
| **Revoke** | `airy_cap_revoke()` | v1.1 | 递归撤销 capability（`atomic_inc(&airy_cap_global_epoch)`） |
| **Delete** | `airy_cap_delete()` | 1.0.1 | 删除单个 capability（不影响派生树） |
| **Rotate** | `airy_cap_rotate()` | v1.1 | 轮换 Badge（仅更新 Epoch，不重新生成 RandomTag） |

**Rotate vs Revoke 区别**：
- `Revoke`：`atomic_inc(&airy_cap_global_epoch)`，全局撤销所有 Agent 的所有 Badge
- `Rotate`：仅更新单个 Agent 的 `agent_caps[agent_id].randtag`，不影响其他 Agent
- 使用场景：Revoke 用于安全事件（Badge 泄漏），Rotate 用于定期轮换（降低 Badge 暴力破解风险）

### 3.2 Mint 操作（缩减权限派生）

```c
/**
 * airy_cap_mint - 缩减权限派生 capability
 * @src_cap:   源 capability ID
 * @mask:      权限掩码（缩减后 badge = src.badge & mask）
 * @new_cap_out: 新 capability ID 输出
 *
 * 返回: 0 成功，-AIRY_EINVAL 源 cap 无效，
 *       -AIRY_EPERM 调用者不拥有源 cap，-AIRY_ENOMEM 分配失败
 *
 * 借鉴 seL4 cnode.c 的 mint 操作。
 * 源 capability 保留不变，生成新的派生 capability。
 */
int airy_cap_mint(uint32_t src_cap, uint64_t mask, uint32_t *new_cap_out);
```

### 3.3 MintCopy 操作（复制 + 缩减）

```c
/**
 * airy_cap_mintcopy - 复制 + 缩减权限
 * @src_cap:    源 capability ID
 * @dest_agent: 目标 Agent ID
 * @mask:       权限掩码
 * @new_cap_out: 新 capability ID 输出
 *
 * 返回: 0 成功，<0 AIRY_E* 错误码
 *
 * 借鉴 seL4 cnode.c 的 mintcopy 操作。
 * 与 mint 的区别：mintcopy 将新 cap 分配到指定目标 Agent 的 CSpace。
 * 用于 Agent 间传递受限 capability。
 */
int airy_cap_mintcopy(uint32_t src_cap, uint32_t dest_agent,
                         uint64_t mask, uint32_t *new_cap_out);
```

### 3.4 Derive 操作（全权派生）

```c
/**
 * airy_cap_derive - 全权派生 capability
 * @src_cap:    源 capability ID
 * @dest_agent: 目标 Agent ID
 * @new_cap_out: 新 capability ID 输出
 *
 * 返回: 0 成功，<0 AIRY_E* 错误码
 *
 * 与 mintcopy 的区别：derive 不缩减权限（badge 全继承）。
 * 用于委托全权给受信任的子任务。
 */
int airy_cap_derive(uint32_t src_cap, uint32_t dest_agent,
                      uint32_t *new_cap_out);
```

### 3.5 Revoke 操作（递归级联撤销）

借鉴 seL4 `cteRevoke()` 的递归级联撤销算法，对齐 [01-agent-lifecycle.md](../140-application-development/01-agent-lifecycle.md) 第 5.2 节：

```c
/**
 * airy_cap_revoke - 递归撤销所有派生 capability
 * @src_cap: 源 capability ID
 *
 * 返回: 0 成功，-AIRY_EINVAL 源 cap 无效
 *
 * 借鉴 seL4 cnode.c:528-550 的 cteRevoke 算法。
 *
 * 递归撤销算法（MDB 链表遍历 + preemptionPoint）:
 *   while (还有派生 capability) {
 *       cap = 获取下一个派生 capability;
 *       if (cap 是 Endpoint 类型) {
 *           取消 badge 上所有待发消息;
 *       }
 *       cap_delete(cap);          // 标记 Zombie 或直接删除
 *       preemption_point();        // 允许调度器干预
 *       if (错误 != SUCCESS) return 错误;
 *   }
 *
 * @since 1.0.1
 */
int airy_cap_revoke(uint32_t src_cap);
```

**preemptionPoint 设计**：借鉴 seL4 的抢占点模式，长时间撤销操作分块可抢占。每撤销 N 个 capability（默认 N=64）插入一次抢占点，允许调度器响应更高优先级任务。

---

## 4. POSIX Capability 集成

### 4.1 41 ID 枚举

对齐 `security_types.h`（[SC] 共享头文件）定义的 POSIX capability 41 ID 枚举。agentrt-linux 在 Linux 6.6 标准 41 个 POSIX capability（编号 0-40，CAP_LAST_CAP=CAP_CHECKPOINT_RESTORE=40，无废弃）基础上，新增 Airymax 专属 capability（编号 41-43）：

```c
/**
 * POSIX capability ID 枚举（41 个 + 3 个 Airymax 专属 = 44）
 * 对齐 Linux 6.6 include/uapi/linux/capability.h
 * 无废弃项，CAP_LAST_CAP = CAP_CHECKPOINT_RESTORE = 40
 */
#define AIRY_CAP_CHOWN              0   /* 修改文件所有者 */
#define AIRY_CAP_DAC_OVERRIDE       1   /* 绕过 DAC 读/写/执行检查 */
#define AIRY_CAP_DAC_READ_SEARCH    2   /* 绕过 DAC 读/搜索 */
#define AIRY_CAP_FOWNER             3   /* 绕过文件权限检查 */
#define AIRY_CAP_FSETID            4   /* 设置 setuid/setgid */
#define AIRY_CAP_KILL               5   /* 绕过发送信号权限检查 */
#define AIRY_CAP_SETGID            6   /* 设置 GID */
#define AIRY_CAP_SETUID            7   /* 设置 UID */
#define AIRY_CAP_SETPCAP           8   /* 设置进程 capability */
#define AIRY_CAP_LINUX_IMMUTABLE    9   /* 设置不可变文件 */
#define AIRY_CAP_NET_BIND_SERVICE  10  /* 绑定 <1024 端口 */
#define AIRY_CAP_NET_BROADCAST     11  /* 广播/监听 */
#define AIRY_CAP_NET_ADMIN         12  /* 网络管理 */
#define AIRY_CAP_NET_RAW           13  /* 原始 socket */
/* ... 完整 41 个枚举对齐 security_types.h（含 CAP_PERFMON=38/CAP_BPF=39/CAP_CHECKPOINT_RESTORE=40）... */
#define AIRY_CAP_LAST_CAP          40  /* 最后一个 Linux 标准 cap（CAP_CHECKPOINT_RESTORE）*/
#define AIRY_CAP_AGENT_SPAWN       41  /* Agent 生成（Airymax 专属，从 41 开始避免与 Linux 0-40 冲突）*/
#define AIRY_CAP_GPU_SCHED         42  /* GPU 调度 */
#define AIRY_CAP_NPU_ACCESS        43  /* NPU 访问 */
#define AIRY_CAP_AGENT_MAX         44  /* agentrt-linux 专属上限（41 Linux + 3 Airymax）*/
```

### 4.2 与传统 POSIX cap 的映射

| 场景 | POSIX capability | seL4 风格 capability | 映射关系 |
|------|-----------------|---------------------|---------|
| 传统 Linux 程序 | `cap_setuid` 进程级位图 | 不适用（POSIX 路径） | 直接使用 POSIX |
| Agent 间传递 | 不适用 | `cap_id` 令牌级传递 | IPC + cap_mintcopy |
| Agent 终止 | 不适用 | `cap_revoke` 递归撤销 | 生命周期清理 |
| 文件访问 | `cap_dac_override` | `cap_id` + badge | 两者共存（LSM 链） |

### 4.3 混合检查流程

**v1.0.1 起，混合检查流程分为两条路径**：

| 路径 | 触发场景 | 执行点 | 延迟 |
|------|---------|--------|------|
| **fastpath C-S9 内联**（v1.0.1 新增） | A-IPC 消息投递（io_uring `IORING_OP_URING_CMD`） | `airy_cap_badge_ok()` | ~10ns |
| **slowpath `airy_cap_check()`**（保留） | 非 IPC 路径（文件访问、内存映射等 LSM 钩子） | `airy_cap_check()` | ~100-200ns |

#### 4.3.1 fastpath C-S9 内联校验（v1.0.1 新增——A-IPC 第一块基石）

fastpath C-S9 是 Capability Folding 的执行点，由 A-IPC fastpath `airy_ipc_validate()` 在消息投递前内联调用。**3 个 READ_ONCE + 位运算 + 比较，零锁、零 RCU、零 kmalloc**：

```c
/**
 * airy_cap_badge_ok - fastpath C-S9 Capability Folding Badge 校验
 * @src_task: 发送方 Agent ID（消息头 src_task 字段）
 * @badge:    消息头 capability_badge 字段（64-bit Native Word）
 * @opcode:   消息头 opcode 字段（用于权限位检查）
 *
 * 返回: 0 允许，<0 AIRY_ECAP_* 错误码
 *   -AIRY_ECAP_BADGE  (-78)  Badge 无效（CAP_REQUEST 路径错或 CAP_CARRY 冲突）
 *   -AIRY_ECAP_EPOCH  (-79)  Epoch 不匹配（Badge 已撤销）
 *   -AIRY_ECAP_FORGED (-80)  Random Tag 不匹配（伪造 Badge，触发 Fault）
 *   -AIRY_ECAP_PERM   (-81)  权限不足
 *
 * 性能: ~10ns（3 个 READ_ONCE + 位运算 + 比较）
 * 上下文: fastpath，持有 io_uring uring_lock，禁止 sleep/block
 * 验证: CBMC 全函数验证（fastpath 100 行代码）
 *
 * 设计依据: 6 条硬约束 H1-H6（详见 §13.2）
 *
 * @since v1.0.1
 */
static inline int airy_cap_badge_ok(u64 src_task, u64 badge, u16 opcode)
{
    u16 badge_epoch;
    u32 badge_randtag, agent_randtag;
    u16 badge_perms, global_epoch;

    /* CAP_REQUEST 自举: 首 Badge 申请走 sec_d，无 Badge 校验 */
    if (unlikely(opcode == AIRY_IPC_OP_CAP_REQUEST)) {
        if (unlikely(src_task != AIRY_SEC_D_TASK_ID))
            return -AIRY_ECAP_BADGE;  /* -78: CAP_REQUEST 仅允许发往 sec_d */
        goto cap_pass;
    }

    /* H3/H6: badge=0 路径（agentrt 用户态 或 [DSL] 降级模式）*/
    if (badge == 0) {
        /* CAP_CARRY 标志下不允许 badge=0（防绕过）*/
        if (unlikely(airy_ipc_hdr_flags_cap_carry()))  /* 内联读取消息头 flags */
            return -AIRY_ECAP_BADGE;  /* -78 */
        goto cap_pass;  /* H3: agentrt 用户态 badge=0 跳过；H6: [DSL] 降级跳过 */
    }

    /* #1: Epoch 校验（1 次 atomic_read，~1ns）*/
    badge_epoch = AIRY_BADGE_EPOCH(badge);
    global_epoch = airy_cap_epoch_get();
    if (unlikely(badge_epoch != global_epoch))
        return -AIRY_ECAP_EPOCH;  /* -79: Badge 已撤销 */

    /* #2: Random Tag 校验（1 次 READ_ONCE，~1ns）——防伪造核心*/
    badge_randtag = AIRY_BADGE_RANDTAG(badge);
    agent_randtag = READ_ONCE(agent_caps[src_task].randtag);
    if (unlikely(badge_randtag != agent_randtag)) {
        /* 伪造 Badge，触发 Fault（不可恢复故障）*/
        airy_security_fault(src_task, AIRY_FAULT_CAP_FORGED);  /* 0x1001 */
        return -AIRY_ECAP_FORGED;  /* -80 */
    }

    /* #3: 权限校验（1 次 READ_ONCE + 位运算，~1ns）*/
    badge_perms = AIRY_BADGE_PERMS(badge);
    if (unlikely(!airy_cap_has_perm(badge_perms, opcode)))
        return -AIRY_ECAP_PERM;  /* -81 */

cap_pass:
    return 0;  /* ~10ns 完成 */
}
```

#### 4.3.2 slowpath `airy_cap_check()`（保留——非 IPC 路径）

```c
/**
 * airy_cap_check - 混合 capability 检查（slowpath，非 IPC 路径）
 * @agent_id: Agent ID
 * @cap_id:   seL4 风格 capability ID（0 = 仅检查 POSIX）
 * @posix_cap: POSIX capability 编号（0 = 仅检查 seL4 风格）
 * @resource: 资源路径（可选，用于审计）
 *
 * 返回: 0 允许，-AIRY_EPERM 拒绝
 *
 * v1.0.1 起此函数仅用于非 IPC 路径（LSM 钩子：inode_permission /
 * file_open / task_create 等）。A-IPC 路径走 fastpath C-S9 内联。
 *
 * 检查流程:
 *   1. 若 cap_id > 0，检查 seL4 风格 capability 令牌
 *   2. 若 posix_cap >= 0，检查 POSIX capability 位图
 *   3. 两者都通过才允许
 */
int airy_cap_check(uint32_t agent_id, uint32_t cap_id,
                     int posix_cap, const char *resource);
```

---

## 5. Capability 令牌生命周期

### 5.1 令牌状态机

对齐 [contracts.md](../50-engineering-standards/20-contracts/contracts.md) 第 6.2 节的 capability 令牌生命周期状态机：

```mermaid
stateDiagram-v2
    [*] --> Created: capability_request()

    Created --> Active: 令牌生成成功（通过策略裁决）
    Created --> Denied: 权限拒绝（AIRY_EPERM）

    Active --> Derived: 通过 mint/mintcopy/derive 派生
    Active --> Used: 执行受保护操作
    Active --> Transferred: 通过 IPC 传递（CAP_CARRY）
    Active --> Revoked: capability_revoke()
    Active --> Expired: 超时自动过期（expires_ns 到达）

    Derived --> Used: 派生令牌执行操作
    Derived --> Revoked: 递归撤销（原令牌撤销时自动撤销）

    Transferred --> Active: 目标 Agent 接收
    Transferred --> Revoked: 传递失败或撤销

    Revoked --> [*]
    Expired --> [*]
    Denied --> [*]
```

### 5.2 状态枚举

```c
enum airy_cap_state {
    AIRY_CAP_STATE_CREATED     = 0,  /* 已创建，待激活 */
    AIRY_CAP_STATE_ACTIVE       = 1,  /* 活跃，可使用 */
    AIRY_CAP_STATE_DERIVED      = 2,  /* 已派生 */
    AIRY_CAP_STATE_USED         = 3,  /* 使用中 */
    AIRY_CAP_STATE_TRANSFERRED  = 4,  /* 传递中 */
    AIRY_CAP_STATE_REVOKED      = 5,  /* 已撤销 */
    AIRY_CAP_STATE_EXPIRED      = 6,  /* 已过期 */
    AIRY_CAP_STATE_DENIED       = 7,  /* 被拒绝 */
};
```

### 5.3 状态转换条件

**v1.0.1 起，syscall 12→4 精确映射落地**——所有 capability 操作统一走 `airy_sys_call`（512）+ 子命令，原 592-600 编号全部废弃：

| 从状态 | 到状态 | 触发条件 | 系统调用（v1.0.1）| 旧编号（v1.0，已废弃） |
|--------|--------|---------|---------------|----------------------|
| — | CREATED | CAP_REQUEST opcode 自举 | A-IPC `AIRY_IPC_OP_CAP_REQUEST`（无 syscall） | 592 airy_sys_capability_request |
| CREATED | ACTIVE | sec_d 编译 Badge 成功 | `airy_sys_call`(512) + `COMPILE_BADGE` 子命令（sec_d 专属） | 内部 |
| CREATED | DENIED | 策略裁决 DENY | 内部 | 内部 |
| ACTIVE | DERIVED | sec_d 重编译 Badge（缩权限） | `airy_sys_call`(512) + `COMPILE_BADGE` 子命令 | 595/596/597 airy_sys_capability_derive/mint/mintcopy |
| ACTIVE | TRANSFERRED | IPC `AIRY_IPC_F_CAP_CARRY` 标志 | A-IPC io_uring `IORING_OP_URING_CMD`（无 syscall） | 600 airy_sys_capability_transfer |
| ACTIVE/DERIVED | REVOKED | `airy_sys_call` + `REVOKE_BADGE`（1 行 atomic_inc） | `airy_sys_call`(512) + `REVOKE_BADGE` 子命令 | 593 airy_sys_capability_revoke |
| ACTIVE | EXPIRED | `expires_ns` 到达 | 内核定时器 | 内核定时器 |

**v1.1 syscall 精简映射**：

| 旧 syscall（v1.0） | 旧编号 | v1.1 处理方式 | v1.1 落地 |
|-------------------|-------|--------------|----------|
| `airy_sys_capability_request` | 592 | 移除 → CAP_REQUEST opcode 自举 | A-IPC `AIRY_IPC_OP_CAP_REQUEST`（无 syscall，无特殊 Badge，无硬编码 dst_task） |
| `airy_sys_capability_revoke` | 593 | 移除 → `airy_sys_call` + `REVOKE_BADGE` | 1 行 `atomic_inc(&airy_cap_global_epoch)`（无 drain、无 bitmap）|
| `airy_sys_lsm_load_policy` | 594 | 移除 → `airy_sys_call` + `LSM_CTL` | sec_d 专属 |
| `airy_sys_capability_derive` | 595 | 移除 → `airy_sys_call` + `COMPILE_BADGE` | sec_d 专属 |
| `airy_sys_capability_mint` | 596 | 移除 → `airy_sys_call` + `COMPILE_BADGE` | sec_d 专属 |
| `airy_sys_capability_mintcopy` | 597 | 移除 → `airy_sys_call` + `COMPILE_BADGE` | sec_d 专属 |
| `airy_sys_capability_inspect` | 598 | 移除 → sysctl 查询 | `kernel.airy.cap_*` sysctl |
| `airy_sys_lsm_audit_query` | 599 | 移除 → sysctl 查询 | `kernel.airy.lsm_audit_*` sysctl |
| `airy_sys_capability_transfer` | 600 | 移除 → IPC + `AIRY_IPC_F_CAP_CARRY` | A-IPC io_uring fastpath |

详细 syscall 12→4 精确映射见 [30-interfaces/01-syscalls.md §2.2](../30-interfaces/01-syscalls.md)。

---

## 6. Cupolas Blob 布局

### 6.1 扁平 Blob + 偏移访问模式

对齐 [01-lsm-framework.md](01-lsm-framework.md) 第 8 章的 Cupolas blob 设计，capability 在四类内核对象上附加 blob：

```c
/**
 * Cupolas blob 尺寸定义
 * 对齐 security_types.h [SC] 共享头文件
 */
struct lsm_blob_sizes cupolas_blob_sizes __ro_after_init = {
    .lbs_cred  = sizeof(struct cupolas_cred_blob),
    .lbs_inode = sizeof(struct cupolas_inode_blob),
    .lbs_file  = sizeof(struct cupolas_file_blob),
    .lbs_task  = sizeof(struct cupolas_task_blob),
};
```

### 6.2 四类 Blob 结构

```c
/**
 * struct cupolas_cred_blob - credential 上的 capability blob
 * @cap_set:   Agent 拥有的 capability 集合（位图）
 * @cspace:   指向 Agent CSpace 的指针
 * @audit_seq: 审计序列号
 */
struct cupolas_cred_blob {
    uint64_t cap_set[2];           /* 128 位位图，支持 128 个 cap */
    struct airy_cspace *cspace;
    uint64_t audit_seq;
};

/**
 * struct cupolas_inode_blob - inode 上的 capability blob
 * @required_cap: 访问此 inode 所需的 capability 类型
 * @policy_hash: 策略哈希（用于快速匹配）
 */
struct cupolas_inode_blob {
    uint8_t  required_cap;
    uint32_t policy_hash;
};

/**
 * struct cupolas_file_blob - file 上的 capability blob
 * @granted_cap: 打开文件时授予的 capability ID
 * @access_mask: 访问掩码（读/写/执行）
 */
struct cupolas_file_blob {
    uint32_t granted_cap;
    uint8_t  access_mask;
};

/**
 * struct cupolas_task_blob - task 上的 capability blob
 * @agent_id:    Agent ID
 * @budget_cap:  Token 预算 capability ID
 * @suspend_flags: 挂起标志
 */
struct cupolas_task_blob {
    uint32_t agent_id;
    uint32_t budget_cap;
    uint32_t suspend_flags;
};
```

### 6.3 Blob 访问内联函数

```c
/* 扁平 blob + 偏移访问内联函数（借鉴 Landlock 模式） */
static inline struct cupolas_cred_blob *cupolas_cred(const struct cred *cred)
{
    return (struct cupolas_cred_blob *)(cred->security + cupolas_blob_sizes.offsets.lbs_cred);
}

static inline struct cupolas_inode_blob *cupolas_inode(const struct inode *inode)
{
    return (struct cupolas_inode_blob *)(inode->i_security + cupolas_blob_sizes.offsets.lbs_inode);
}

static inline struct cupolas_file_blob *cupolas_file(const struct file *file)
{
    return (struct cupolas_file_blob *)(file->f_security + cupolas_blob_sizes.offsets.lbs_file);
}

static inline struct cupolas_task_blob *cupolas_task(const struct task_struct *task)
{
    return (struct cupolas_task_blob *)(task->security + cupolas_blob_sizes.offsets.lbs_task);
}
```

---

## 7. 策略裁决模型

### 7.1 四值裁决枚举

对齐 `security_types.h`（[SC] 共享头文件）定义的策略裁决结果 4 值枚举：

```c
/**
 * enum airy_cap_verdict - capability 策略裁决结果
 *
 * 4 值裁决模型对齐 SELinux 的 allow/deny/audit/deny_audit 语义。
 */
enum airy_cap_verdict {
    AIRY_CAP_ALLOW         = 0,  /* 允许，不记录 */
    AIRY_CAP_DENY          = 1,  /* 拒绝，不记录 */
    AIRY_CAP_AUDIT         = 2,  /* 允许，记录审计日志 */
    AIRY_CAP_DENY_AUDIT    = 3,  /* 拒绝，记录审计日志 */
};
```

### 7.2 裁决流程

```mermaid
graph TD
    A[capability_request 请求] --> B[1. 参数校验]
    B --> C[2. POSIX cap 检查<br/>检查进程 cap 位图]
    C --> D{POSIX cap 满足?}
    D -->|否| E[返回 AIRY_CAP_DENY]
    D -->|是| F[3. seL4 风格 cap 检查<br/>检查 CSpace 中是否有对应 cap]
    F --> G{seL4 cap 存在?}
    G -->|否| H[4. 策略裁决<br/>查询 Cupolas 策略引擎]
    G -->|是| I[5. badge 检查<br/>检查请求的操作在 badge 范围内]
    H --> J{策略裁决}
    I --> K{badge 满足?}
    K -->|否| E
    K -->|是| L[返回 AIRY_CAP_ALLOW]
    J -->|ALLOW| L
    J -->|DENY| E
    J -->|AUDIT| L
    J -->|DENY_AUDIT| E

    style E fill:#ffcdd2,stroke:#c62828
    style L fill:#c8e6c9,stroke:#2e7d32
```

### 7.3 裁决实现

```c
/**
 * airy_cap_adjudicate - 策略裁决
 * @agent_id:  Agent ID
 * @cap_type:  capability 类型
 * @resource:  资源路径
 * @operation: 请求的操作
 *
 * 返回: AIRY_CAP_* 裁决结果
 *
 * 裁决顺序:
 *   1. POSIX cap 检查（快速路径）
 *   2. seL4 风格 cap 检查（令牌检查）
 *   3. Cupolas 策略引擎（慢速路径）
 */
enum airy_cap_verdict airy_cap_adjudicate(
    uint32_t agent_id,
    enum airy_cap_type cap_type,
    const char *resource,
    uint32_t operation);
```

---

## 8. Vault Backend 抽象

### 8.1 后端存储抽象

对齐 `security_types.h`（[SC] 共享头文件）定义的 Vault backend 抽象，capability 的持久化存储通过后端抽象层访问：

```c
/**
 * struct airy_vault_backend - Vault 后端抽象
 *
 * @name:         后端名称
 * @store:        存储 capability
 * @retrieve:      检索 capability
 * @delete:        删除 capability
 * @list:          列出 capability
 * @seal:         封存（加密 + TPM 度量）
 * @unseal:       解封
 *
 * 借鉴 Linux keyring 的 key_type 抽象模式。
 * 支持多种后端：内存/TPM/机密计算。
 */
struct airy_vault_backend {
    const char *name;
    int  (*store)(uint32_t cap_id, const void *data, size_t len);
    int  (*retrieve)(uint32_t cap_id, void *buf, size_t *len);
    int  (*delete)(uint32_t cap_id);
    int  (*list)(uint32_t agent_id, uint32_t *cap_ids, uint32_t *count);
    int  (*seal)(uint32_t cap_id, const uint8_t *kek, size_t kek_len);
    int  (*unseal)(uint32_t cap_id, const uint8_t *kek, size_t kek_len);
};
```

### 8.2 后端实现

| 后端 | 用途 | 安全级别 | 实现状态 |
|------|------|---------|---------|
| `vault_memory` | 内存存储（默认） | 低 | 0.1.1 |
| `vault_tpm` | TPM 2.0 封存 | 高 | 1.0.1 |
| `vault_cvm` | 机密计算（VirtCCA） | 最高 | 2.0+ |

### 8.3 TPM 封存流程

```c
/**
 * airy_vault_seal_tpm - 使用 TPM 封存 capability
 * @cap_id:    capability ID
 * @tpm_handle: TPM 密钥句柄
 *
 * 封存流程:
 *   1. 生成对称密钥（AES-256）
 *   2. 加密 capability 数据
 *   3. 用 TPM 密钥加密对称密钥（封存）
 *   4. 度量 capability 哈希到 TPM PCR
 *   5. 存储加密数据
 *
 * @since 1.0.1
 */
int airy_vault_seal_tpm(uint32_t cap_id, uint32_t tpm_handle);
```

---

## 9. 系统调用集成（v1.0.1：syscall 12→4 精确映射）

### 9.1 v1.1 Capability 系统调用总览

**v1.0.1 起，agentrt-linux 仅保留 4 个核心 syscall**，原 592-600 共 9 个 capability 相关 syscall 全部废弃，统一收敛到 `airy_sys_call`(512) + 子命令。详细映射见 [30-interfaces/01-syscalls.md §2.2](../30-interfaces/01-syscalls.md)。

| v1.1 syscall | 编号 | 子命令 | 功能 | 调用者 |
|--------------|------|--------|------|--------|
| `airy_sys_call` | 512 | `COMPILE_BADGE` | sec_d 编译 Badge（含 mint/mintcopy/derive 等价语义）| 仅 sec_d |
| `airy_sys_call` | 512 | `REVOKE_BADGE` | 撤销 Badge（1 行 `atomic_inc(&airy_cap_global_epoch)`）| 仅 sec_d |
| `airy_sys_call` | 512 | `LSM_CTL` | 加载 LSM 策略 | 仅 sec_d |
| `airy_sys_call` | 512 | `WASM_LOAD` | 加载 Wasm 模块 | 仅 sec_d |
| `airy_sys_rovol_ctl` | 513 | — | MemoryRovol 控制 | rovol_d |
| `airy_sys_sched_ctl` | 514 | — | sched_tac 调度控制 | sched_d |
| `airy_sys_clt_notify` | 515 | — | 客户端通知 | clt_d |

**Capability 申请路径**（CAP_REQUEST opcode 自举，无 syscall）：

```
Agent → io_uring SQE（opcode=CAP_REQUEST, capability_badge=0, dst_task=sec_d）
       → fastpath C-S9（CAP_REQUEST 特例，跳过 Badge 校验，仅允许 dst_task=sec_d）
       → sec_d 接收 → sec_d 调用 airy_sys_call + COMPILE_BADGE
       → 内核编译 Badge（Epoch + Random Tag + Perms）
       → sec_d 通过 IPC 响应返回 Badge 给 Agent
```

### 9.2 核心 C 接口（v1.0.1）

```c
/**
 * airy_sys_call - Capability Invocation 系统调用（v1.1 唯一保留的 capability syscall）
 * @cmd:     子命令（AIRY_SYS_CMD_COMPILE_BADGE / REVOKE_BADGE / LSM_CTL / WASM_LOAD）
 * @arg:     子命令参数（结构体指针，子命令特定）
 * @arg_len: 参数长度
 *
 * 返回: 0 成功，<0 AIRY_E* 错误码
 *   -AIRY_ECAP_BADGE   (-78)  调用者非 sec_d（H4 硬约束）
 *   -AIRY_ECAP_EPOCH   (-79)  Epoch 不匹配
 *   -AIRY_ECAP_FORGED  (-80)  伪造 Badge
 *   -AIRY_ECAP_PERM    (-81)  权限不足
 *   -AIRY_EINVAL       (-22)  参数无效
 *
 * v1.1 变更: 原 592-600 共 9 个 capability syscall 全部收敛到此入口。
 * 仅 sec_d（security daemon）允许调用，其他 Agent 调用返回 -AIRY_ECAP_BADGE。
 *
 * @since v1.0.1
 * @see 30-interfaces/01-syscalls.md §1.4
 */
AIRY_API int airy_sys_call(uint32_t cmd, void *arg, size_t arg_len);

/* 子命令枚举 */
enum airy_sys_cmd {
    AIRY_SYS_CMD_COMPILE_BADGE = 1,   /* sec_d 编译 Badge（含 mint/mintcopy/derive 等价）*/
    AIRY_SYS_CMD_REVOKE_BADGE  = 2,   /* sec_d 撤销 Badge（1 行 atomic_inc）*/
    AIRY_SYS_CMD_LSM_CTL       = 3,   /* sec_d 加载 LSM 策略 */
    AIRY_SYS_CMD_WASM_LOAD     = 4,   /* sec_d 加载 Wasm 模块 */
};

/* COMPILE_BADGE 子命令参数 */
struct airy_cmd_compile_badge {
    uint32_t agent_id;              /* 目标 Agent ID */
    uint16_t perms;                 /* 权限位图（AIRY_CAP_PERM_* 或运算）*/
    uint16_t reserved;
    uint64_t expires_ns;            /* 过期时间戳（0 = 永不过期）*/
    /* 输出: 编译后的 Badge（64-bit Native Word）*/
    uint64_t badge_out;
};

/* REVOKE_BADGE 子命令参数 */
struct airy_cmd_revoke_badge {
    uint32_t agent_id;              /* 目标 Agent ID（0 = 全局撤销，atomic_inc）*/
    uint32_t reserved;
    /* REVOKE_BADGE 执行: atomic_inc(&airy_cap_global_epoch)
     * 效果: 所有已发出 Badge 的 Epoch 立即过期，下次 fastpath C-S9 校验失败
     * 性能: 1 行代码，~1ns，无 drain、无 bitmap、无 IPI
     */
};
```

### 9.3 IPC 传递 capability（v1.0.1：通过 A-IPC fastpath，无 syscall）

v1.0.1 起，capability 传递不再走独立 syscall，而是通过 A-IPC 消息头 `AIRY_IPC_F_CAP_CARRY` 标志在 fastpath 内联完成：

```c
/* Agent A 通过 IPC 传递 capability 给 Agent B（v1.1: 无 syscall）*/

/* 1. Agent A 通过 CAP_REQUEST opcode 向 sec_d 申请 Badge */
struct airy_ipc_msg_hdr req_hdr = {
    .opcode = AIRY_IPC_OP_CAP_REQUEST,    /* 0x0010 */
    .capability_badge = 0,                /* 自举: 无 Badge */
    .dst_task = AIRY_SEC_D_TASK_ID,       /* 目标: sec_d */
    /* ... */
};
io_uring_submit(sq_cap_request);          /* A-IPC fastpath */

/* 2. sec_d 返回编译后的 Badge 给 Agent A */
/* Agent A 获得 Badge（含 perms=SEND|RECV|GRANT）*/

/* 3. Agent A 携带 Badge 发送 IPC 给 Agent B */
struct airy_ipc_msg_hdr carry_hdr = {
    .opcode = AIRY_IPC_OP_SEND,           /* 0x0001 */
    .flags  = AIRY_IPC_F_CAP_CARRY,       /* 携带 Badge 传递 */
    .capability_badge = my_badge,         /* Agent A 的 Badge */
    .src_task = agent_a_id,
    .dst_task = agent_b_id,
    /* ... */
};
io_uring_submit(sq_send);                 /* A-IPC fastpath C-S9 校验 Badge */

/* 4. Agent B 接收消息，从消息头提取 Badge（派生权限）*/
/* fastpath C-S9 校验 Agent A 的 Badge 通过后，Agent B 获得派生 Badge */
```

---

## 10. LSM 集成

### 10.1 LSM_ORDER_FIRST 注册

对齐 [01-lsm-framework.md](01-lsm-framework.md) 第 7 章，capability 作为 `LSM_ORDER_FIRST` 永远第一个执行：

```c
/* security/commoncap.c（借鉴 Linux 6.6 实现）*/
static struct lsm_id capability_lsmid __lsm_ro_after_init = {
    .lsm  = "capability",
    .slot = LSMBLOB_NOT_NEEDED,
};

DEFINE_LSM(capability) = {
    .name       = "capability",
    .order      = LSM_ORDER_FIRST,  /* 永远第一 */
    .blobs      = &cupolas_blob_sizes,
    .init       = capability_init,
    .lsmid      = &capability_lsmid,
};
```

### 10.2 与 Landlock/Cupolas 共存

```mermaid
sequenceDiagram
    participant K as 内核路径
    participant SH as security_X() 入口
    participant CAP as capability (LSM_ORDER_FIRST)
    participant LL as Landlock
    participant CP as Cupolas

    K->>SH: security_file_open()
    SH->>CAP: 调用回调（第一优先级）
    CAP-->>SH: 返回 0（放行）
    SH->>LL: 调用回调
    LL-->>SH: 返回 0（放行）
    SH->>CP: 调用回调（最后）
    CP-->>SH: 返回 -EACCES（拒绝）
    SH-->>K: 短路返回 -EACCES
```

**短路评估**：任一 LSM 钩子返回非零即终止链路，对齐 [01-lsm-framework.md](01-lsm-framework.md) 第 4 章 `call_int_hook` 短路评估模式。capability 作为第一优先级，快速放行无 capability 需求的操作。

### 10.3 LSM 钩子注册

```c
/* capability LSM 钩子注册（借鉴 lsm_hook_defs.h X-Macro 模式）*/
static struct security_hook_list capability_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(capable,           airy_cap_capable),
    LSM_HOOK_INIT(inode_permission,  airy_cap_inode_perm),
    LSM_HOOK_INIT(file_open,         airy_cap_file_open),
    LSM_HOOK_INIT(task_create,       airy_cap_task_create),
    LSM_HOOK_INIT(cred_prepare,      airy_cap_cred_prepare),
    LSM_HOOK_INIT(cred_free,         airy_cap_cred_free),
};
```

---

## 11. Capability 守卫流程（v1.0.1：fastpath C-S9 内联，无独立守卫）

### 11.1 v1.1 守卫模式变更

**v1.0.1 起，A-IPC 路径不再使用独立 capability 守卫**——Badge 校验已"折叠"到 fastpath C-S9 内联。原 v1.0 的 `AIRY_CAP_GUARD` 宏（前置申请 + 操作 + 撤销）仅保留给非 IPC 路径（LSM 钩子）使用。

| 路径 | v1.0 守卫模式 | v1.1 守卫模式 |
|------|--------------|--------------|
| A-IPC 消息投递 | `AIRY_CAP_GUARD` 前置申请 → `airy_sys_ipc_send` → `AIRY_CAP_GUARD_END` 撤销（3 步） | fastpath C-S9 内联 Badge 校验（1 步，~10ns） |
| 文件访问（LSM 钩子） | `AIRY_CAP_GUARD` 前置申请 → 操作 → 撤销 | 保留 v1.0 模式（slowpath `airy_cap_check()`） |
| 内存映射（LSM 钩子） | `AIRY_CAP_GUARD` 前置申请 → 操作 → 撤销 | 保留 v1.0 模式（slowpath `airy_cap_check()`） |

### 11.2 A-IPC 路径的 fastpath C-S9 守卫（v1.0.1 新增）

A-IPC 路径完全消除了独立守卫调用。Badge 校验直接嵌入 fastpath：

```c
/* v1.1: A-IPC fastpath 内联 Badge 校验，无独立守卫 */

/* 发送方: 仅需填充消息头 capability_badge 字段 */
struct airy_ipc_msg_hdr hdr = {
    .magic             = AIRY_IPC_MAGIC,        /* 0x41524531 'ARE1' */
    .opcode            = AIRY_IPC_OP_SEND,      /* 0x0001 */
    .capability_badge  = my_badge,              /* 64-bit Native Word */
    .src_task          = my_agent_id,
    .dst_task          = target_agent_id,
    /* ... */
};

/* 提交到 io_uring——内核 fastpath 自动执行 C-S9 Badge 校验 */
io_uring_submit(ring);  /* fastpath: airy_ipc_validate() → airy_cap_badge_ok() */

/* 内核 fastpath C-S9 流程:
 *   1. C-S0~C-S8: magic/opcode/flags/trace_id/timestamp/src_task/dst_task/payload_len 校验
 *   2. C-S9: airy_cap_badge_ok(src_task, badge, opcode)  ← Capability Folding 执行点
 *      - Epoch 校验（atomic_read，~1ns）
 *      - Random Tag 校验（READ_ONCE，~1ns）
 *      - Perms 校验（位运算，~1ns）
 *   3. C-S10~C-S12: CRC32 完整性校验
 *   4. 投递消息（airy_ipc_deliver_fast）
 *
 * 总延迟: ~158ns（FAST_SEND），含 C-S9 Badge 校验 ~10ns
 */
```

### 11.3 非 IPC 路径的 slowpath 守卫（保留 v1.0 模式）

非 IPC 路径（文件访问、内存映射等 LSM 钩子）仍使用 `AIRY_CAP_GUARD` 宏：

```c
/**
 * capability 守卫宏（v1.1: 仅用于非 IPC 路径的 LSM 钩子）
 * A-IPC 路径走 fastpath C-S9 内联，不使用此宏。
 */
#define AIRY_CAP_GUARD(cap_name, resource) ({            \
    int __cap = airy_cap_check(current_agent_id(),       \
                                 0, (cap_name), (resource)); \
    if (__cap < 0) {                                        \
        log_write(LOG_ERROR, "capability denied: %d", __cap); \
        return -AIRY_EPERM;                              \
    }                                                       \
    __cap;                                                  \
})

#define AIRY_CAP_GUARD_END(cap_slot)  \
    /* v1.1: 非 IPC 路径无需撤销令牌（slowpath 不持有 Badge）*/
```

### 11.4 守卫使用示例（v1.0.1）

```c
/* A-IPC 路径: 无需守卫宏，fastpath C-S9 内联校验 */
int airy_ipc_send(const struct airy_ipc_msg_hdr *hdr,
                   const void *payload)
{
    /* 直接提交 io_uring，fastpath C-S9 自动校验 Badge */
    return io_uring_submit(ring);
}

/* 非 IPC 路径: 使用 AIRY_CAP_GUARD 宏 */
int airy_file_open(const char *path, int flags)
{
    int cap = AIRY_CAP_GUARD(AIRY_CAP_FILE_READ, path);
    int fd = __do_file_open(path, flags);
    AIRY_CAP_GUARD_END(cap);
    return fd;
}
```

### 11.5 IPC 传递 capability（v1.0.1：通过 A-IPC fastpath，无 syscall）

v1.0.1 起，capability 传递完全通过 A-IPC 消息头 `AIRY_IPC_F_CAP_CARRY` 标志，在 fastpath C-S9 校验通过后自动派生：

```c
/* 发送方: 携带 capability_badge */
struct airy_ipc_msg_hdr hdr = {
    .magic             = AIRY_IPC_MAGIC,
    .flags             = AIRY_IPC_F_CAP_CARRY,    /* 携带 Badge 传递 */
    .opcode            = AIRY_IPC_OP_SEND,
    .capability_badge = my_badge,                 /* Agent A 的 Badge */
    .src_task          = agent_a_id,
    .dst_task          = agent_b_id,
    /* ... */
};
io_uring_submit(ring);  /* fastpath C-S9 校验 my_badge */

/* 接收方: 从消息头提取派生 Badge（自动 mintcopy 等价）*/
struct airy_ipc_msg_hdr recv_hdr;
io_uring_wait_cqe(ring, &cqe);
/* recv_hdr.capability_badge 是 Agent A 的 Badge
 * Agent B 获得派生权限（受 Agent A 的 Perms 约束）
 * Agent B 后续使用此 Badge 发送 IPC 时，fastpath C-S9 校验
 * agent_caps[agent_b_id].randtag 与 Badge 中的 Random Tag 比对
 * （注: Agent B 需先通过 CAP_REQUEST 向 sec_d 申请自己的 Badge）
 */
```

---

## 12. IRON-9 v3 同源映射

### 12.1 [SC] 层共享

`security_types.h` 是 agentrt 与 agentrt-linux 的 [SC] 共享头文件，包含：

| 共享内容 | 影响 |
|---------|------|
| POSIX capability 41 ID 枚举 | 编号一致 |
| LSM 钩子 250 ID 枚举 | 钩子编号一致 |
| Cupolas blob 布局（cred/inode/file/task） | 结构体一致 |
| capability 派生模型（mint/mintcopy/derive/revoke） | 算法语义一致 |
| Vault backend 抽象 | 接口一致 |
| 策略裁决 4 值枚举 | 裁决语义一致 |

### 12.2 [SS] 层语义同源

| 维度 | agentrt 用户态 | agentrt-linux OS 层 | 同源语义 |
|------|---------------|-------------------|---------|
| 权限模型 | Cupolas 应用权限 | seL4 风格 + POSIX | 不可伪造、最小权限 |
| 令牌生成 | 用户态 opaque handle | 内核态 cap_id | 不可伪造语义一致 |
| 撤销算法 | 应用级递归清理 | `cteRevoke` 递归撤销 | 递归级联语义一致 |
| IPC 传递 | AgentsIPC CAP_CARRY | io_uring CAP_CARRY | 标志位语义一致 |

### 12.3 [IND] 层独立

agentrt-linux 独有：
- 内核态 CNode 表（**v1.1: `agent_caps[1024]` 静态数组存储**，sec_d 唯一写者）
- 内核态 Badge 64-bit Native Word 模型（v1.0.1 新增，agentrt 用户态 badge=0）
- fastpath C-S9 内联 Badge 校验（v1.0.1 新增，~10ns）
- 全局 Epoch atomic_t（v1.0.1 新增，撤销时 1 行 atomic_inc）
- capability 模块 `LSM_ORDER_FIRST` 注册（OLK 6.6 硬编码，仅用于 capabilities；airy_lsm 改用 `LSM_ORDER_MUTABLE` + `CONFIG_LSM` 列表首位）
- TPM Vault 封存
- 内核审计日志（SHA-256 哈希链）

### 12.4 [DSL] 降级生存层（v1.0.1 新增）

[DSL] 降级模式下，capability 校验退化为 `capability_badge=0` 跳过 C-S9（H6 硬约束）：

| [DSL] 降级行为 | 说明 |
|---------------|------|
| `capability_badge=0` | 所有 IPC 消息头 Badge 字段置零 |
| 跳过 C-S9 | fastpath 不执行 `airy_cap_badge_ok()`，直接 `goto cap_pass` |
| 仅保留 POSIX capability | 41 个标准 cap 位图校验（slowpath `airy_cap_check()`） |
| sec_d 不可达 | Badge 编译/撤销暂停，恢复后批量处理 |
| Epoch 不更新 | `airy_cap_global_epoch` 冻结，避免误失效 |

详细 [DSL] 降级模式见 [11-degraded-survival-layer.md §4.4](../10-architecture/11-degraded-survival-layer.md)。

### 12.5 Capability Folding 集成总览（v1.0.1 新增——A-IPC 第一块基石）

#### 12.5.1 6 条硬约束 H1-H6

| 硬约束 | 内容 | 落地位置 |
|--------|------|---------|
| **H1** | Layout C v4 总长 128B / magic 0x41524531 / `capability_badge` offset 40-47 | [SC] `ipc.h`（§2.5）|
| **H2** | `capability_badge` 字段进 [SC] `ipc.h`，但 Badge 校验机制属 [SS] `airy_cap_badge_ok()` | 本文件 §4.3.1 + [02-ipc-protocol.md §2.6](../30-interfaces/02-ipc-protocol.md) |
| **H3** | agentrt 用户态 `capability_badge=0`（跳过 C-S9） | [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) [IND] 层 |
| **H4** | agentrt-linux 内核 Badge 由 sec_d 编译、fastpath C-S9 校验 | 本文件 §2.4 + §9 |
| **H5** | 纯 C LSM 职责不变（不使用 BPF LSM） | [01-lsm-framework.md](01-lsm-framework.md) |
| **H6** | [DSL] 降级 `capability_badge=0` 跳过 C-S9 | 本文件 §12.4 + [11-degraded-survival-layer.md §4.4](../10-architecture/11-degraded-survival-layer.md) |

#### 12.5.2 数据结构隔离三原则

Capability Folding 的隔离基础是数据结构级隔离（vs seL4 架构级隔离），三个独立内存区域：

```
1. agent_caps[1024]   能力数据（内核静态数组，16KB）
   ├─ 唯一写者: sec_d（通过 airy_sys_call + COMPILE_BADGE）
   ├─ 读者: fastpath C-S9（READ_ONCE）
   └─ 同步: WRITE_ONCE/READ_ONCE（无锁，单写者多读者）

2. kfifo              数据投递缓冲区（per-cpu SPSC 无锁）
   ├─ 写者: fastpath kfifo_in
   └─ 读者: 接收方 CQE

3. io_uring ring      通信原语
   ├─ 写者: 发送方 SQE
   └─ 读者: 接收方 CQE
```

三者内存区域独立，更新路径独立，失败域独立——这是 31 号审查论证的"数据结构隔离"基础，使 fastpath 100 行代码 CBMC 全函数验证可行。

#### 12.5.3 fastpath C-S9 落地路径

```
Agent 发送 IPC 消息（携带 capability_badge）
  │
  ▼
io_uring SQE（IORING_OP_URING_CMD）
  │
  ▼
airy_uring_cmd() ──▶ airy_ipc_validate()
  │
  ▼
C-S0~C-S8: magic/opcode/flags/trace_id/timestamp/src_task/dst_task/payload_len 校验
  │
  ▼
C-S9: airy_cap_badge_ok(src_task, badge, opcode)  ← Capability Folding 执行点
  │  ├─ CAP_REQUEST 自举 → 跳过（仅允许 dst_task=sec_d）
  │  ├─ badge=0 → 跳过（H3 agentrt / H6 [DSL]）
  │  ├─ #1 Epoch 校验（atomic_read，~1ns）
  │  ├─ #2 Random Tag 校验（READ_ONCE，~1ns）
  │  └─ #3 Perms 校验（位运算，~1ns）
  │
  ▼
C-S10~C-S12: CRC32 完整性校验
  │
  ▼
airy_ipc_deliver_fast()（NONBLOCK，~158ns 总延迟）
```

#### 12.5.4 v1.0 → v1.1 对比

| 维度 | v1.0（radix tree + per-cpu cache） | v1.1（Capability Folding） |
|------|-----------------------------------|---------------------------|
| 物理存储 | `struct radix_tree_root slots`（动态分配） | `agent_caps[1024]` 静态数组（16KB） |
| 校验路径 | 独立前置 `airy_cap_check()` + per-cpu cache | fastpath C-S9 内联 `airy_cap_badge_ok()` |
| 校验延迟 | ~50-100ns（radix tree 遍历 + cache 查询） | ~10ns（3 个 READ_ONCE + 位运算） |
| 撤销机制 | 递归遍历 children 链表 + IPI 失效 per-cpu cache | 1 行 `atomic_inc(&airy_cap_global_epoch)` |
| 锁开销 | CSpace 读写锁 + per-cpu cache 锁 | 零锁（单写者 sec_d + 多读者 fastpath） |
| 同步开销 | RCU synchronize + IPI | 零 RCU、零 IPI |
| syscall 数量 | 9 个（592-600） | 1 个（`airy_sys_call` + 子命令） |
| 验证范围 | 全函数 + 数据结构 | fastpath 100 行代码 CBMC 全函数验证 |
| 防伪造 | cap_id 内核分配（不可伪造） | Random Tag 32 位（per-Agent，sec_d 编译） |

---

## 13. 使用示例

### 13.1 Agent 注册时获取初始 capability

```c
/* Agent 注册时自动获得初始 capability */
struct airy_task_config cfg = {
    .capability_flags = AIRY_CAP_COGNITION | AIRY_CAP_IPC | AIRY_CAP_ROVOL,
};
uint32_t agent_id;
airy_sys_task_register(&cfg, &agent_id);

/* Agent 的 CSpace 自动创建，包含初始 capability */
```

### 13.2 Agent 间传递受限 capability

```c
/* Agent A 将 file.read capability 受限传递给 Agent B */

/* 1. Agent A 申请 capability */
int cap = airy_sys_capability_request("file.read", "/data/shared.txt");

/* 2. Agent A 通过 mintcopy 传递受限 capability */
uint32_t new_cap;
airy_sys_capability_mintcopy(cap, agent_b_id, 0x01, &new_cap);
/* 0x01 = 仅读权限（badge 缩减） */

/* 3. Agent A 通过 IPC 传递 */
struct airy_ipc_msg_hdr hdr = {
    .flags  = AIRY_IPC_F_CAP_CARRY,
    .cap_id = new_cap,
};
airy_sys_ipc_send(&hdr, NULL);

/* 4. Agent A 撤销原 capability 时，Agent B 的派生 cap 自动撤销 */
airy_sys_capability_revoke(cap);
```

### 13.3 Agent 终止时递归撤销

```c
/* Agent 终止时清理所有 capability（对齐 01-agent-lifecycle.md 第 5.2 节）*/
void agent_terminate_cleanup(uint32_t agent_id)
{
    struct cupolas_cred_blob *blob;
    struct airy_cspace *cspace;

    /* 1. 获取 Agent 的 CSpace */
    blob = cupolas_cred(current_cred());
    cspace = blob->cspace;

    /* 2. 递归撤销根 capability */
    airy_cap_revoke(cspace->root_cap);

    /* 3. 清理 CSpace */
    airy_cspace_destroy(cspace);

    log_write(LOG_INFO, "agent %u capabilities revoked", agent_id);
}
```

### 13.4 Random Tag 碰撞概率分析（v1.0.1 新增）

**设计问题**：Random Tag 是 32 位随机数，per-Agent 独立。当系统中有 N 个 Agent 时，是否存在两个 Agent 拥有相同 Random Tag 的概率？碰撞会导致什么后果？

#### 13.4.1 生日攻击概率分析

Random Tag 碰撞遵循**生日悖论**（Birthday Paradox）。给定 N 个 Agent，每个 Agent 独立生成 32-bit Random Tag，至少两个 Agent 碰撞的概率为：

```
P(N, 32) ≈ 1 - e^(-N² / (2 × 2^32))
```

| Agent 数量 N | 碰撞概率 P(N, 32) | 等价场景 |
|-------------|------------------|---------|
| 10 | 0.000001% (10⁻⁸) | 极不可能 |
| 100 | 0.0001% (10⁻⁶) | 极不可能 |
| 1000 | 0.01% (10⁻⁴) | 罕见 |
| 5000 | 0.3% | 罕见 |
| 10000 | 1.2% | 可能（10000 Agent 系统需关注）|
| 65536 | 39.4% | 高概率（理论极限，实际不会达到）|

**结论**：0.1.1 阶段（Agent < 100），碰撞概率 < 10⁻⁶，可忽略。1.0.1 阶段（Agent ≤ 10000），碰撞概率 ~1.2%，需考虑检测机制。

#### 13.4.2 碰撞的安全影响分析

**关键问题**：如果 Agent A 和 Agent B 拥有相同的 Random Tag，会发生什么？

**答案：无安全影响**。原因：

1. **Badge 绑定 Agent ID**：Badge 校验时，fastpath C-S9.RANDTAG 读取的是 `agent_caps[hdr->src_task].randtag`，而非全局查找。即使 Agent A 和 Agent B 的 Random Tag 相同，Agent A 的 Badge 也只能通过 `agent_caps[A].randtag` 的校验，无法通过 `agent_caps[B].randtag` 的校验。

2. **src_task 字段防伪**：消息头的 `src_task` 字段由内核填写（[SC] ipc.h Layout C v4 offset 24），用户态无法伪造。Agent A 无法声称自己是 Agent B。

3. **Epoch 全局唯一**：Epoch 是全局 atomic_t，所有 Agent 共享。Epoch 不匹配时，所有 Badge 立即失效。

**碰撞的实际影响**：仅影响审计日志的可读性（两个 Agent 的 Badge 在日志中可能看起来相似），不影响安全性。

#### 13.4.3 Random Tag 碰撞检测机制（可选，1.0.1 阶段）

虽然碰撞无安全影响，但 1.0.1 阶段可添加可选的碰撞检测，提升审计可读性：

```c
/* kernel/security/airy/capability.c —— Random Tag 碰撞检测（可选）
 *
 * 1.0.1 阶段可选实现。0.1.1 阶段不实现（碰撞概率 < 10⁻⁶）。
 *
 * 检测策略：编译 Badge 时遍历 agent_caps[]，检查是否已有相同 Random Tag。
 * 若碰撞，重新生成 Random Tag（最多重试 3 次）。
 *
 * 性能影响：O(N) 遍历，1000 Agent 时 ~10μs（可接受，slowpath）。
 */
static int airy_cap_randtag_collision_check(uint32_t new_randtag)
{
    int i, retries = 0;

retry:
    for (i = 0; i < AIRY_CAP_MAX_AGENTS; i++) {
        if (READ_ONCE(agent_caps[i].state) == AIRY_CAP_STATE_ACTIVE &&
            READ_ONCE(agent_caps[i].randtag) == new_randtag) {
            /* 碰撞！重新生成 */
            if (++retries > 3)
                return -EAGAIN;
            get_random_bytes_wait(&new_randtag, sizeof(new_randtag));
            goto retry;
        }
    }
    return 0;  /* 无碰撞 */
}
```

#### 13.4.4 Random Tag 暴力枚举攻击分析

**攻击场景**：攻击者控制 Agent A，尝试枚举 Random Tag 来伪造 Agent B 的 Badge。

**攻击成本**：
- Random Tag 空间：2^32 = 4,294,967,296（42 亿）
- 单次 fastpath C-S9 校验：~10ns
- 枚举完所有 Random Tag：2^32 × 10ns = 42.9 秒

**防御机制**：
1. **C-S9.RANDTAG 失败触发 Fault**：每次 Random Tag 不匹配都会触发 `AIRY_FAULT_CAP_FORGED`(0x1001)，导致 Agent 被 KILL（见 [07-airy-lsm-design.md §6.2](07-airy-lsm-design.md)）
2. **连续失败计数器**：Agent 连续 3 次 C-S9.RANDTAG 失败后，Micro-Supervisor 冻结其 Ring（`ring->frozen = true`）
3. **审计告警**：每次 C-S9.RANDTAG 失败都记录审计日志，A-ULP 触发告警

**结论**：暴力枚举 Random Tag 在实际中不可行（3 次失败即被冻结）。

#### 13.4.5 Random Tag 熵源保障

**熵源**：`get_random_bytes_wait()` 使用 Linux CRNG（ChaCha20-based），熵充足时返回高质量随机数。

**熵不足场景**：系统启动早期（熵池未充满），`get_random_bytes_wait()` 会阻塞等待熵充足。这确保 sec_d 在熵不足时不会生成弱 Random Tag。

**性能影响**：系统启动早期，sec_d 首次编译 Badge 可能阻塞 ~100ms（等待 CRNG 就绪）。这是可接受的，因为系统启动是非热路径。

### 13.5 NUMA 场景 agent_caps[] 访问延迟分析与优化（v1.0.1 新增）

**设计问题**：`agent_caps[1024]` 是 16KB 内核静态数组（`struct airy_cap_slot` 16B × 1024 项）。在多 NUMA 节点服务器上（典型 2-8 节点），fastpath C-S9 读取 `agent_caps[src_task].randtag` 与 `.perms` 时，若 `src_task` 所属 Agent 运行在远端 NUMA 节点，访问延迟将显著高于本地节点访问。本节从工程角度分析此问题并给出优化策略。

#### 13.5.1 NUMA 访问延迟模型

| 访问场景 | 典型延迟 | 说明 |
|---------|---------|------|
| L1 cache 命中（本地 NUMA） | ~1ns | `agent_caps[src_task]` 已在 L1（热路径） |
| L2 cache 命中（本地 NUMA） | ~4ns | `agent_caps[src_task]` 在 L2（温路径） |
| L3 cache 命中（本地 NUMA） | ~12ns | `agent_caps[src_task]` 在 L3（冷启动） |
| 本地 DRAM 访问（本地 NUMA） | ~60-80ns | cache miss，本地节点 DRAM |
| 远端 DRAM 访问（远端 NUMA） | ~120-150ns | cache miss，跨 NUMA 节点 DRAM（UPI/QPI 互联） |
| 跨 socket 访问（远端 NUMA） | ~200-300ns | 跨 socket，最坏情况 |

**关键观察**：fastpath C-S9 总延迟预算 ~10ns，远端 NUMA 访问 ~120-150ns，**远端 NUMA 访问延迟是 fastpath 预算的 12-15 倍**。若不优化，远端 NUMA Agent 的 IPC fastpath 延迟将从 ~158ns 退化至 ~270-310ns。

#### 13.5.2 agent_caps[] 内存占用与 cache 覆盖分析

```
agent_caps[1024] 内存布局（16KB 静态数组）:

┌──────────────────────────────────────────────────────┐
│ Slot 0    │ Slot 1    │ Slot 2    │ ... │ Slot 1023 │
│ 16B       │ 16B       │ 16B       │     │ 16B       │
│ randtag   │ randtag   │ randtag   │     │ randtag   │
│ perms     │ perms     │ perms     │     │ perms     │
│ state     │ state     │ state     │     │ state     │
│ reserved  │ reserved  │ reserved  │     │ reserved  │
└──────────────────────────────────────────────────────┘

Cache line 覆盖（64B/cache line）:
- 4 个 slot / cache line（64B / 16B = 4）
- 256 cache lines 总计（1024 / 4）
- 16KB 占 L1 cache 的 50%（典型 L1 = 32KB）
- 16KB 占 L2 cache 的 6.25%（典型 L2 = 256KB）
- 16KB 占 L3 cache 的 1.3%（典型 L3 = 1.25MB）
```

**关键结论**：
1. **16KB 完全可放入 L2/L3 cache**——若 fastpath 访问模式稳定，L2 命中率应 > 95%
2. **L1 cache 占用 50%**——`agent_caps[]` 占用一半 L1，可能与 fastpath 其他热数据竞争
3. **工作集远小于 16KB**——实际活跃 Agent 通常 < 100，活跃工作集 ~1.6KB（< 1 cache line）

#### 13.5.3 NUMA 访问模式分析

**访问模式**：fastpath C-S9 按 `src_task`（Agent ID）随机访问 `agent_caps[]`。在 NUMA 系统上：

| 部署场景 | 访问模式 | NUMA 影响 |
|---------|---------|----------|
| 单 NUMA 节点 | 所有 Agent 在 node 0 | 无 NUMA 影响，本地访问 |
| 2 NUMA 节点，Agent 均匀分布 | 50% 本地访问 + 50% 远端访问 | 平均延迟退化 ~60-75ns |
| 4 NUMA 节点，Agent 均匀分布 | 25% 本地 + 75% 远端 | 平均延迟退化 ~90-110ns |
| 8 NUMA 节点（跨 socket） | 12.5% 本地 + 87.5% 远端/跨 socket | 平均延迟退化 ~150-260ns |

**关键风险**：8 NUMA 节点场景下，fastpath 平均延迟可能从 ~158ns 退化至 ~310-420ns，仍低于 1μs 预算但接近上限。

#### 13.5.4 优化策略 1：NUMA 本地分配 agent_caps[]（基础策略）

**策略**：`agent_caps[]` 分配在 sec_d 所在的 NUMA 节点（通常为 node 0），sec_d 是唯一写者，写访问天然本地化。

```c
/* kernel/security/airy/capability.c —— NUMA 本地分配（v1.0.1 新增）
 *
 * agent_caps[] 分配在 sec_d 所在 NUMA 节点（node 0）。
 * sec_d 是唯一写者，写访问本地化。
 * fastpath 读者可能跨 NUMA，但 agent_caps[] 16KB 易被 L3 cache 覆盖。
 */

/* 静态数组使用 __initdata 标记初始化期数据，运行期使用 alloc_pages_node 分配 */
static struct airy_cap_slot *agent_caps __read_mostly;

static int __init airy_cap_agent_caps_init(void)
{
    int node = 0;  /* sec_d 默认运行在 node 0 */
    struct page *page;

    /* 分配 16KB 连续内存（4 个 pages），指定 NUMA 节点 */
    page = alloc_pages_node(node, GFP_KERNEL | __GFP_ZERO | __GFP_NOWARN,
                              get_order(AIRY_CAP_MAX_AGENTS * sizeof(struct airy_cap_slot)));
    if (!page) {
        /* 退化：使用普通 kmalloc（不绑定 NUMA 节点） */
        agent_caps = kzalloc(AIRY_CAP_MAX_AGENTS * sizeof(struct airy_cap_slot),
                              GFP_KERNEL);
        if (!agent_caps)
            return -ENOMEM;
        pr_warn("airy: agent_caps[] fallback to kmalloc (no NUMA pinning)\n");
    } else {
        agent_caps = page_address(page);
        pr_info("airy: agent_caps[] pinned to NUMA node %d (%zu KB)\n",
                 node, AIRY_CAP_MAX_AGENTS * sizeof(struct airy_cap_slot) / 1024);
    }

    atomic_set(&airy_cap_global_epoch, 0);
    return 0;
}
```

**效果**：
- sec_d 写访问：本地 NUMA，~60-80ns（可接受，slowpath）
- fastpath 读者：可能跨 NUMA，~120-150ns（远端）

#### 13.5.5 优化策略 2：per-NUMA-node 只读副本（进阶策略，1.0.1 阶段可选）

**策略**：在每个 NUMA 节点维护 `agent_caps[]` 的只读副本，fastpath 读者访问本地副本，sec_d 写者通过 IPI 同步所有副本。

```c
/* kernel/security/airy/capability.c —— per-NUMA-node 只读副本（1.0.1 阶段可选）
 *
 * 设计:
 *   - 每个 NUMA 节点维护 agent_caps[] 的只读副本
 *   - sec_d 是唯一写者，写入主副本（node 0）后通过 IPI 同步到其他节点
 *   - fastpath 读者访问本节点副本（本地 NUMA 访问）
 *
 * 权衡:
 *   - 优点: fastpath 读者始终本地访问，~60-80ns
 *   - 缺点: 内存占用 ×N（N = NUMA 节点数），IPI 同步开销
 *   - 适用: 8+ NUMA 节点大规模服务器
 *
 * 同步模型:
 *   - sec_d WRITE_ONCE 主副本
 *   - smp_call_function_many() IPI 通知其他节点
 *   - 其他节点在 IPI 处理函数中 memcpy 副本
 *   - fastpath READ_ONCE 本地副本
 *
 * 注意: 0.1.1 阶段不实现，1.0.1 阶段根据实测延迟决定是否启用
 */

#define AIRY_CAP_NUMA_REPLICATION  0  /* 0.1.1: 关闭，1.0.1: 可选 */

#if AIRY_CAP_NUMA_REPLICATION
static struct airy_cap_slot *agent_caps_replica[NR_NODES] __read_mostly;

/* fastpath 读取本地副本 */
static inline u32 airy_cap_read_randtag(u32 agent_id)
{
    int node = numa_node_id();
    return READ_ONCE(agent_caps_replica[node][agent_id].randtag);
}

/* sec_d 写入主副本 + IPI 同步 */
static void airy_cap_write_slot(u32 agent_id, struct airy_cap_slot *new_slot)
{
    int node;

    /* 1. 写入主副本（node 0） */
    WRITE_ONCE(agent_caps_replica[0][agent_id].randtag, new_slot->randtag);
    WRITE_ONCE(agent_caps_replica[0][agent_id].perms, new_slot->perms);
    smp_wmb();

    /* 2. IPI 同步到其他节点 */
    for_each_online_node(node) {
        if (node == 0)
            continue;
        smp_call_function_single(node_to_cpu(node),
                                  airy_cap_replica_sync,
                                  (void *)new_slot, 1);
    }
}
#endif
```

**效果**：
- fastpath 读者：始终本地 NUMA 访问，~60-80ns（消除远端访问）
- sec_d 写者：写入主副本 + IPI 同步，~5-20μs（slowpath，可接受）
- 内存开销：16KB × N（8 节点 = 128KB，可忽略）

#### 13.5.6 优化策略 3：Agent NUMA 亲和性绑定（部署策略）

**策略**：在部署层面，将 Agent 进程绑定到特定 NUMA 节点，使 `src_task` 的 Agent 与 `agent_caps[]` 主副本（node 0）通信模式可预测。

```c
/* services/daemons/macro_d/sched.c —— Agent NUMA 亲和性绑定（v1.0.1）
 *
 * 策略:
 *   - 关键 Agent（sched_d, sec_d, audit_d）绑定到 node 0（与 agent_caps[] 主副本同节点）
 *   - 计算 Agent 按负载均衡分布到各 NUMA 节点
 *   - fastpath C-S9 访问 agent_caps[src_task]，关键 Agent 本地访问
 */

static const struct {
    const char *daemon_name;
    int preferred_node;
} airy_daemon_numa_prefs[] = {
    { "sec_d",          0 },   /* Badge 唯一写者，必须与 agent_caps[] 同节点 */
    { "sched_d",        0 },   /* 调度守护，频繁访问 agent_caps[] */
    { "audit_d",        0 },   /* 审计守护，记录所有 Badge 操作 */
    { "macro_d",   0 },   /* 主监管，与 sec_d 通信频繁 */
    { "logger_d", -1 },   /* 日志守护，无 NUMA 偏好 */
    { "vfs_d",         -1 },   /* VFS 守护，无 NUMA 偏好 */
    { "net_d",         -1 },   /* 网络守护，无 NUMA 偏好 */
    /* 其他 daemon 按负载均衡分配 */
};

static int macro_d_set_numa_affinity(pid_t pid, const char *name)
{
    int node = -1;

    for (size_t i = 0; i < ARRAY_SIZE(airy_daemon_numa_prefs); i++) {
        if (strcmp(name, airy_daemon_numa_prefs[i].daemon_name) == 0) {
            node = airy_daemon_numa_prefs[i].preferred_node;
            break;
        }
    }

    if (node >= 0) {
        /* 绑定到指定 NUMA 节点 */
        struct bitmask *mask = numa_allocate_nodemask();
        numa_bitmask_setbit(mask, node);
        numa_bind(mask);
        numa_free_nodemask(mask);
        pr_info("airy: daemon %s pinned to NUMA node %d\n", name, node);
    }

    return 0;
}
```

**效果**：
- 关键 daemon（sec_d/sched_d/audit_d/macro_d）本地访问 `agent_caps[]`，~60-80ns
- 其他 daemon 跨 NUMA 访问，~120-150ns（可接受，fastpath 预算 1μs）
- 关键路径 IPC（Agent ↔ sec_d）始终本地访问

#### 13.5.7 优化策略 4：fastpath prefetch（编译期优化）

**策略**：在 fastpath C-S9 之前，prefetch `agent_caps[src_task]` 所在 cache line，隐藏 DRAM 访问延迟。

```c
/* kernel/ipc/airy_uring_cmd.c —— fastpath prefetch 优化（v1.0.1 新增）
 *
 * 在 C-S0~C-S8 校验期间 prefetch agent_caps[src_task]，隐藏 DRAM 延迟。
 * prefetch 是非阻塞指令，CPU 继续执行后续校验，cache line 异步加载。
 */

static int airy_ipc_validate(struct airy_ipc_cmd *cmd)
{
    struct airy_ipc_msg_hdr *hdr = cmd->hdr;

    /* C-S0: ring 冻结检查 */
    if (unlikely(cmd->ring->frozen))
        return -AIRY_ECAP_FROZEN;

    /* C-S1: magic 检查 */
    if (unlikely(hdr->magic != AIRY_IPC_MAGIC))
        return -AIRY_EIPC_MAGIC;

    /* === v1.0.1 新增: prefetch agent_caps[src_task]（在 C-S2~C-S8 期间加载） === */
    prefetch(&agent_caps[hdr->src_task]);

    /* C-S2: opcode 合法性检查 */
    if (unlikely(!airy_ipc_opcode_valid(hdr->opcode)))
        return -AIRY_EIPC_OPCODE;

    /* C-S3~C-S8: 其他校验（~20ns 期间，prefetch 异步加载 cache line） */
    /* ... */

    /* C-S9: Badge 校验——此时 agent_caps[src_task] 已被 prefetch 加载 */
    /* 即使远端 NUMA，prefetch 已隐藏大部分延迟 */
    return airy_cap_badge_ok(current, hdr->capability_badge, hdr->opcode);
}
```

**效果**：
- 远端 NUMA 访问延迟从 ~120-150ns 降低至 ~30-50ns（prefetch 隐藏 ~80-100ns）
- 本地 NUMA 访问无变化（~60-80ns，prefetch 无显著收益）
- 总 fastpath 延迟：~158ns（本地） / ~190-210ns（远端，prefetch 后）

#### 13.5.8 NUMA 优化策略汇总

| 策略 | 阶段 | 实现复杂度 | fastpath 远端延迟 | 适用场景 |
|------|------|----------|-----------------|---------|
| 基础策略（NUMA 本地分配） | 0.1.1 | 低 | ~120-150ns | 所有 NUMA 系统 |
| per-NUMA-node 只读副本 | 1.0.1 | 高 | ~60-80ns | 8+ NUMA 节点 |
| Agent NUMA 亲和性绑定 | 0.1.1 | 低 | ~60-80ns（关键 daemon） | 所有 NUMA 系统 |
| fastpath prefetch | 0.1.1 | 极低 | ~30-50ns | 所有 NUMA 系统 |
| **组合（基础 + 亲和性 + prefetch）** | **0.1.1** | **低** | **~30-50ns** | **推荐 0.1.1 落地** |

**0.1.1 落地决策**：采用"基础策略 + Agent NUMA 亲和性 + fastpath prefetch"组合方案。per-NUMA-node 只读副本策略留至 1.0.1 阶段根据实测延迟决定。

#### 13.5.9 NUMA 场景性能 SLO

| 指标 | 单 NUMA | 2 NUMA | 4 NUMA | 8 NUMA（跨 socket） |
|------|---------|--------|--------|-------------------|
| fastpath 本地访问延迟 | ~158ns | ~158ns | ~158ns | ~158ns |
| fastpath 远端访问延迟（无优化） | N/A | ~270ns | ~310ns | ~420ns |
| fastpath 远端访问延迟（优化后） | N/A | ~190ns | ~210ns | ~270ns |
| fastpath p99 延迟 SLO | < 1μs | < 1μs | < 1μs | < 1μs |
| 是否满足 SLO | ✅ | ✅ | ✅ | ✅ |

**结论**：0.1.1 阶段采用组合优化策略后，8 NUMA 跨 socket 场景 fastpath 延迟 ~270ns，远低于 1μs SLO 预算，留有 ~3.7× 余量。

### 13.6 跨节点 Badge 一致性（R5 补强：0.1.1 单节点 / 1.0.1 跨节点）

**设计背景**：多节点部署时，每个节点的 sec_d 独立编译 Badge，但 Badge 中的 `Epoch` 和 `RandomTag` 的跨节点一致性需要明确定义。本节区分 0.1.1（单节点）与 1.0.1（跨节点）两种部署模型，明确各自的 Badge 一致性语义，对齐前序会话"per-node recompilation"决策。

#### 13.6.1 架构假设

| 版本 | 部署模型 | 跨节点支持 | 实现状态 |
|------|---------|-----------|---------|
| **0.1.1** | 单节点 | ❌ 不实现 | sec_d 独占 `agent_caps[1024]`，无跨节点一致性问题 |
| **1.0.1** | 跨节点（通过 `gateway_d`）| ✅ 实现 | per-node Badge + gateway_d 跨节点 IPC + gossip Epoch 同步 |

**`gateway_d` 守护进程**（1.0.1 跨节点 IPC 网关）：基于 gRPC over QUIC + mTLS + X.509 实现节点间安全通信。`gateway_d` 是 Macro-Supervisor 管理的 12 个 daemon 之一（详见 [20-modules/10-user-supervisor-daemon.md §1.3](../20-modules/10-user-supervisor-daemon.md)）。

#### 13.6.2 0.1.1 单节点模型（无跨节点一致性问题）

```
单节点部署（0.1.1）:
  ┌─────────────────────────────────────────┐
  │            单一节点                      │
  │  ┌──────────────────────────────────┐   │
  │  │       agent_caps[1024]            │   │
  │  │  （sec_d 唯一写者，fastpath 读）    │   │
  │  └──────────────────────────────────┘   │
  │  ┌──────────────────────────────────┐   │
  │  │       airy_cap_global_epoch       │   │
  │  │  （atomic_t，单节点内全局）         │   │
  │  └──────────────────────────────────┘   │
  │  ┌──────────────────────────────────┐   │
  │  │       sec_d（唯一 Badge 编译者）   │   │
  │  └──────────────────────────────────┘   │
  └─────────────────────────────────────────┘
```

**0.1.1 单节点一致性保证**：
- sec_d 独占 `agent_caps[1024]`（H4 单写者约束，详见 §2.4）
- `airy_cap_global_epoch` 是单节点 atomic_t，无跨节点同步需求
- 所有 Agent 的 Badge 在同一节点编译、校验、撤销
- **明确声明：0.1.1 不实现跨节点 Badge 一致性**（AirymaxOS 0.1.1 是单节点部署）

#### 13.6.3 1.0.1 跨节点模型（per-node Badge + gateway_d 跨节点 IPC）

```
跨节点部署（1.0.1）:
  节点 A                                节点 B
  ┌─────────────────────┐              ┌─────────────────────┐
  │ agent_caps_A[1024]  │              │ agent_caps_B[1024]  │
  │ sec_d_A（独立编译）  │              │ sec_d_B（独立编译）  │
  │ epoch_A (atomic_t)  │              │ epoch_B (atomic_t)  │
  └─────────────────────┘              └─────────────────────┘
           │                                    │
           │      gateway_d（gRPC/QUIC/mTLS）     │
           └────────────────────────────────────┘
                    │
                    ▼
            gossip 协议同步 Epoch
            （默认 100ms 间隔传播）
```

**1.0.1 跨节点关键设计决策**：

| 维度 | 1.0.1 设计 | 理由 |
|------|-----------|------|
| Badge 有效性范围 | **per-node**：每个 Agent 的 Badge 仅在其注册节点有效 | sec_d 是 per-node 独立编译者，Badge 绑定 per-node agent_caps[] |
| RandomTag | **per-node 独立**：不跨节点共享 | RandomTag 是 per-Agent 随机标签，跨节点无意义（不同节点的 agent_caps[] 独立）|
| Epoch | **跨节点同步**：通过 gateway_d gossip 协议传播 | 全局撤销（`atomic_inc`）需跨节点生效，否则旧节点 Badge 仍可使用 |
| 跨节点 IPC | gateway_d 在源节点验证 Badge 后，为目标节点重新编译等价 Badge | 避免跨节点共享 agent_caps[]（H4 单写者约束 per-node 成立）|

#### 13.6.4 跨节点 IPC 流程（1.0.1）

```
Agent_A（节点 A）→ Agent_B（节点 B）跨节点 IPC:
    │
    ▼
1. Agent_A 在节点 A 持有 Badge_A（epoch_A, randtag_A, perms）
    │
    ▼
2. Agent_A 发送跨节点 IPC 请求到本节点 gateway_d_A
    │
    ▼
3. gateway_d_A 验证 Badge_A（fastpath C-S9，本地 agent_caps_A[]）
    │  ├── Epoch 校验：badge_epoch == epoch_A
    │  ├── RandomTag 校验：badge_randtag == agent_caps_A[A].randtag
    │  └── Perms 校验：满足跨节点 IPC 所需权限
    │
    ▼
4. gateway_d_A 通过 gRPC over QUIC + mTLS 转发请求到 gateway_d_B
    │  └── 携带：源 Agent ID、目标 Agent ID、请求 payload、perms 位掩码
    │     （不携带 Badge_A 本身，避免跨节点泄漏）
    │
    ▼
5. gateway_d_B 接收请求，向本节点 sec_d_B 请求为目标 Agent_B 编译等价 Badge_B
    │  └── sec_d_B 编译 Badge_B（epoch_B, randtag_B_new, perms）
    │     （ perms 与 Badge_A 相同，epoch/randtag 是节点 B 本地的）
    │
    ▼
6. Agent_B 通过 Badge_B 接收跨节点 IPC 消息（fastpath C-S9 本地校验）
```

#### 13.6.5 Epoch 跨节点同步（gossip 协议）

| 属性 | 值 |
|------|-----|
| 同步协议 | gossip（反熵传播） |
| 同步间隔 | 100ms（默认，可通过 `airy.gateway_d.epoch_gossip_interval` 配置）|
| 传播方式 | gateway_d 间两两交换 epoch 值 |
| 冲突解决 | **高 Epoch 覆盖低 Epoch**（单调递增，取 max）|
| 收敛时间 | ~100ms（单跳 gossip，N 节点收敛 ≤ log(N) × 100ms）|
| 一致性保证 | **最终一致**（跨节点 ~100ms 内收敛）|

**Epoch 单调递增不变量**：

```c
/* services/daemons/gateway_d/epoch_sync.c —— Epoch gossip 同步（1.0.1）
 *
 * 不变量: epoch 跨节点单调递增，永不回退。
 * 冲突解决: 收到对端 epoch > 本地 epoch 时，本地 epoch 提升至对端值。
 */

static int airy_epoch_gossip_merge(uint16_t remote_epoch)
{
    uint16_t local_epoch = (uint16_t)atomic_read(&airy_cap_global_epoch);

    /* 高 Epoch 覆盖低 Epoch（单调递增）*/
    if (remote_epoch > local_epoch) {
        atomic_set(&airy_cap_global_epoch, remote_epoch);
        syslog(LOG_INFO, "airy: epoch synced %u → %u (gossip)",
               local_epoch, remote_epoch);
        /* 本节点所有旧 Badge 自动失效（C-S9.EPOCH 检查失败）*/
    }
    /* remote_epoch <= local_epoch: 无操作（本地已是最新或更新）*/
    return 0;
}
```

#### 13.6.6 一致性保证总结

| 一致性层级 | 范围 | 机制 | 收敛时间 |
|-----------|------|------|---------|
| **强一致** | 单节点内 `agent_caps[]` | sec_d 单写者（H4）+ WRITE_ONCE/READ_ONCE | 即时（~10ns fastpath）|
| **强一致** | 单节点内 Epoch | `atomic_t` 单节点原子操作 | 即时（~1ns atomic_inc）|
| **最终一致** | 跨节点 Epoch | gateway_d gossip 协议（100ms 间隔）| ~100ms（单跳）/ ~log(N)×100ms（N 节点）|
| **per-node 独立** | RandomTag | 不跨节点共享 | N/A（设计如此）|
| **per-node 独立** | Badge 有效性 | Badge 仅在注册节点有效 | N/A（设计如此）|

#### 13.6.7 版本实现声明

| 版本 | 跨节点 Badge 一致性 | gateway_d | gossip Epoch 同步 |
|------|-------------------|-----------|------------------|
| **0.1.1** | ❌ 不实现（单节点部署）| daemon 存在但不启用跨节点 IPC | ❌ 不实现 |
| **1.0.1** | ✅ 实现（per-node Badge + gateway_d 跨节点 IPC）| ✅ gRPC over QUIC + mTLS + X.509 | ✅ 100ms 间隔 gossip |

**明确声明**：
- **0.1.1 不实现跨节点 Badge 一致性**。AirymaxOS 0.1.1 是单节点部署，sec_d 独占 `agent_caps[1024]`，无跨节点一致性问题。
- **1.0.1 实现跨节点 Badge 一致性**。通过 `gateway_d` 守护进程支持跨节点 IPC（gRPC over QUIC + mTLS + X.509），采用 per-node Badge + gossip Epoch 同步模型。

---

## 14. 测试策略

### 14.1 单元测试（KUnit）

| 测试项 | 测试方法 | 通过标准 |
|--------|---------|---------|
| CNode 分配 | 创建 CNode 并检查 cap_id | cap_id 唯一且 > 0 |
| Mint 缩减 | mint 后子 badge ≤ 父 badge | badge = 父 & mask |
| MintCopy 跨 Agent | mintcopy 到目标 Agent | 目标 CSpace 包含新 cap |
| Revoke 递归 | revoke 后检查所有派生 cap | 全部 REVOKED |
| 过期检查 | 设置 expires_ns 并等待 | 自动 EXPIRED |
| badge 越权 | 子 cap 请求超出 badge 的操作 | 返回 -EPERM |

### 14.2 集成测试（kselftest）

| 测试场景 | 步骤 | 预期结果 |
|---------|------|---------|
| 完整生命周期 | request→use→revoke | 状态转换正确 |
| IPC 传递 | Agent A 派生→IPC 传递→Agent B 接收 | B 获得派生 cap |
| 递归撤销 | root→derive→derive→revoke root | 全部撤销 |
| LSM 共存 | capability + Landlock + Cupolas | 短路评估正确 |
| POSIX 混合 | seL4 cap + POSIX cap 同时检查 | 两者都通过才允许 |

### 14.3 KUnit 测试示例

```c
/* KUnit 测试：递归撤销 */
static void test_cap_revoke_recursive(struct kunit *test)
{
    uint32_t root_cap, child1, child2, grandchild;

    /* 1. 创建根 capability */
    KUNIT_EXPECT_EQ(test, 0,
        airy_cap_create(AGENT_TYPE_ENDPOINT, 0xFF, &root_cap));

    /* 2. 派生两个子 capability */
    KUNIT_EXPECT_EQ(test, 0,
        airy_cap_mintcopy(root_cap, agent_b, 0x0F, &child1));
    KUNIT_EXPECT_EQ(test, 0,
        airy_cap_mintcopy(root_cap, agent_c, 0x33, &child2));

    /* 3. child1 再派生孙 capability */
    KUNIT_EXPECT_EQ(test, 0,
        airy_cap_mintcopy(child1, agent_d, 0x03, &grandchild));

    /* 4. 撤销根 capability */
    KUNIT_EXPECT_EQ(test, 0, airy_cap_revoke(root_cap));

    /* 5. 验证所有派生 capability 已撤销 */
    KUNIT_EXPECT_EQ(test, get_cap_state(child1), AIRY_CAP_STATE_REVOKED);
    KUNIT_EXPECT_EQ(test, get_cap_state(child2), AIRY_CAP_STATE_REVOKED);
    KUNIT_EXPECT_EQ(test, get_cap_state(grandchild), AIRY_CAP_STATE_REVOKED);
}
```

---

## 15. 已知问题与修复指引

### 15.1 问题 P1-CAP-01: README 文档索引未更新

**问题描述**：[README.md](README.md) 第 3.1 节记录"仅完成 README + 01 + 02"，但 03-capability-model.md 本文档已创建。

**严重程度**：P1（文档同步）

**修复指引**：更新 README.md 第 3.1 节，追加 03-capability-model.md 完成状态。

---

## 16. 相关文档

### 16.1 上游设计文档（不修改）

- [01-lsm-framework.md](01-lsm-framework.md) — LSM 框架详解（第 7 章 LSM 与 capability 共存）
- [02-landlock-sandbox.md](02-landlock-sandbox.md) — Landlock 用户态沙箱
- [50-engineering-standards/20-contracts/contracts.md](../50-engineering-standards/20-contracts/contracts.md) — 系统调用 API 契约（第 6 章 Cupolas 安全）
- [140-application-development/01-agent-lifecycle.md](../140-application-development/01-agent-lifecycle.md) — Agent 生命周期（第 5.2 节 Capability 撤销）

### 16.2 实现方案文档（本系列）

- [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md) — 系统调用接口（v1.0.1 Capability Folding 集成版，syscall 12→4 精确映射）
- [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) — IPC 协议（Layout C v4 + capability_badge 字段）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) — IPC fastpath（C-S9 Badge 校验执行点）
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) — Airymax Unify Design 总纲（v1.1 §8 A-IPC Capability Folding）
- [10-architecture/11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) — [DSL] 降级生存层（H6 落地）

### 16.3 关联模块

- [20-modules/03-security.md](../20-modules/03-security.md) — security 子仓设计
- [90-observability/README.md](../90-observability/README.md) — 审计日志
- [160-compatibility/01-abi-stability.md](../160-compatibility/01-abi-stability.md) — capability ABI 稳定性

---

## 17. 参考材料

### 17.1 seL4 参考

- `src/object/cnode.c` — CNode 操作（mint/mintcopy/move/copy/revoke/delete）
- `src/kernel/mdb.c` — MDB 派生链遍历
- `src/object/cnode.c:528-550` — cteRevoke 递归撤销算法
- `libsel4/include/api/types.h` — capability 类型定义
- `src/object/endpoint.c` — Endpoint capability 与 IPC

### 17.2 主流 Linux 发行版 Linux 6.6 内核基线 参考

- `security/commoncap.c` — POSIX capability 检查
- `include/linux/capability.h` — capability 结构体定义
- `include/uapi/linux/capability.h` — POSIX capability 编号
- `security/lsm_hook_defs.h` — LSM 钩子 X-Macro 定义

### 17.3 Linux 6.6 参考

- `Documentation/security/credentials.rst` — credential 文档
- `include/linux/cred.h` — `struct cred` 定义
- `include/linux/lsm_hooks.h` — LSM 钩子声明
- `security/keys/key.c` — keyring 后端抽象模式

---

## 18. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-09 | 初始版本。定义 seL4 风格 + POSIX 混合 capability 模型：CNode/CSpace 数据结构、MDB 派生链、四种派生操作（mint/mintcopy/derive/revoke）、POSIX 41 ID 枚举、令牌 7 状态生命周期、Cupolas blob 四类布局（cred/inode/file/task）、4 值策略裁决（ALLOW/DENY/AUDIT/DENY_AUDIT）、Vault backend 抽象（memory/TPM/CVM）、系统调用集成（592-600）、LSM_ORDER_FIRST 集成、capability 守卫流程、IPC 传递、KUnit 测试 |
| v1.0 | 2026-07-17 | 离线缓存校验机制 + Reconciliation + MDB 完整性约束（附录 A.1-A.3） |
| **v1.1** | **2026-07-18** | **Capability Folding 集成版（A-IPC 第一块基石）**：(1) §2.4 CSpace 物理存储从 `radix_tree_root` 重构为 `agent_caps[1024]` 静态数组（H4 落地）；(2) §2.5 新增 Badge 64-bit Native Word 模型（Epoch 16位 + Random Tag 32位 + Perms 16位）；(3) §4.3 新增 fastpath C-S9 内联校验 `airy_cap_badge_ok()`（~10ns，3 个 READ_ONCE + 位运算）；(4) §5.3 状态转换表更新为 syscall 12→4 精确映射（原 592-600 全部废弃，统一走 `airy_sys_call` + 子命令）；(5) §9 系统调用集成重写为 v1.1 版本（4 个核心 syscall + 子命令）；(6) §11 守卫流程更新为 fastpath C-S9 内联（A-IPC 路径无独立守卫）；(7) §12.3 [IND] 层独立更新（radix tree → 静态数组）；(8) §12.4 新增 [DSL] 降级生存层（H6 落地）；(9) §12.5 新增 Capability Folding 集成总览（H1-H6 + 数据结构隔离三原则 + fastpath C-S9 落地路径 + v1.0/v1.1 对比）；(10) 清除所有内部审查路径引用 |
| 1.0.1 | 2027-XX-XX | 内核实现完成，TPM Vault 封存落地，形式化验证探索 |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

## 附录：v1.0 升级说明

> **升级背景**：本次 v1.0 升级将本文档对齐 Airymax Unify Design（见 [10-unify-design.md](../10-architecture/10-unify-design.md)）5 模块设计与 IRON-9 v3 四层模型，补充 v0.1.1 缺失的离线缓存、Reconciliation 与 MDB 完整性约束设计。升级内容如下。

### A.1 v1.1 重构：离线缓存校验机制 → fastpath C-S9 内联 Badge 校验

**v1.0.1 起，v1.0 的"per-cpu capability 缓存 + HMAC 签名 + IPI 失效"机制全部废弃**，由 Capability Folding fastpath C-S9 内联 Badge 校验取代。原因如下：

| v1.0 离线缓存机制问题 | v1.0.1 Capability Folding 解决方案 |
|---------------------|--------------------------------|
| per-cpu cache 需 IPI 失效，跨 CPU 同步开销大 | `agent_caps[1024]` 静态数组，READ_ONCE 无锁读取，零 IPI |
| HMAC 签名增加 ~50ns 校验开销 | Badge 64-bit Native Word 校验仅 ~10ns（位运算） |
| 缓存条目与 daemon 持久化视图对账复杂 | sec_d 唯一写者，无对账需求（H4 硬约束） |
| [DSL] 降级时退化为 POSIX cap（功能受限） | [DSL] 降级时 `capability_badge=0` 跳过 C-S9（H6 硬约束，IPC 仍可用） |

**v1.1 离线场景（sec_d 不可达）行为**：

```
正常模式（sec_d 在线）:
  Agent → CAP_REQUEST → sec_d → airy_sys_call + COMPILE_BADGE
       → 内核编译 Badge → 返回 Agent
       → Agent 携带 Badge 发送 IPC → fastpath C-S9 校验（~10ns）

离线模式（sec_d 不可达）:
  Agent 已持有的 Badge 仍有效（直到 Epoch 溢出或显式撤销）
  Agent 使用已有 Badge 发送 IPC → fastpath C-S9 校验（~10ns）
  新 Badge 申请: CAP_REQUEST 进入 sec_d 队列等待恢复
  sec_d 恢复后: 批量处理 CAP_REQUEST 队列

[DSL] 降级模式（[SC] 头文件损坏）:
  capability_badge=0（H6 硬约束）
  fastpath C-S9 跳过 Badge 校验（goto cap_pass）
  仅保留 POSIX capability 位图校验（slowpath airy_cap_check）
```

### A.2 已新增 Reconciliation 设计

v0.1.1 §5 令牌生命周期未定义"capability 在 daemon 离线期间被撤销，恢复后如何对账"的 Reconciliation 流程。v1.0 新增 **Reconciliation 设计**：

- **对账触发**：capability daemon 重启或网络恢复后，发起 Reconciliation 扫描，比对内核 CSpace 与 daemon 持久化视图
- **三阶段对账**：
  1. **Diff 阶段**：枚举内核活跃 capability 集合，与 daemon 持久化记录 diff
  2. **Reconcile 阶段**：对 diff 中"内核有 daemon 无"的孤儿 capability 执行 revoke；对"daemon 有内核无"的失效记录执行清理
  3. **Verify 阶段**：对账后重新计算 MDB 派生链哈希，与 daemon 持久化哈希比对，不一致则告警
- **一致性保证**：Reconciliation 全程持 CSpace 读写锁，对账期间新 capability 操作降级至 slowpath
- 该设计对齐 Airymax Unify Design A-ULS 模块的"内核冷酷执法，用户温情裁决"哲学

### A.3 已新增 MDB（Memory Database）完整性约束

v0.1.1 §2.3 MDB 派生链仅描述 parent/children 指针关系，未定义完整性约束与不变式。v1.0 新增 **MDB 完整性约束**，对齐 seL4 `src/kernel/mdb.c` 的派生链不变式：

- **约束 I1（权限单调递减）**：子 capability 的 badge 必须 ≤ 父 capability 的 badge（权限只减不增），mint/mintcopy 时强制 `child.badge = parent.badge & mask`
- **约束 I2（派生链无环）**：MDB 派生链是树结构，禁止成环；cteInsert 时校验 parent 不在 child 的子树中
- **约束 I3（撤销原子性）**：revoke 递归撤销时，要么全部子 capability 撤销成功，要么回滚至撤销前状态（对齐 §3.5 preemptionPoint 分块撤销的原子语义）
- **约束 I4（refcount 一致性）**：capability 的 refcount 必须 = MDB 子树中引用此 cap 的节点数；revoke 后 refcount 归零方可物理删除
- **约束 I5（哈希链完整性）**：MDB 派生链维护 Merkle 哈希链，每个节点的哈希 = SHA-256(自身 cap_id + badge + 子节点哈希)；reconciliation 时整链哈希校验，检测任何篡改
- **CI 校验**：KUnit 测试覆盖 5 项约束（§14.1），违反任一约束的 revoke/mint 操作必须返回 `AIRY_EINVAL`

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
