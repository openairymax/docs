Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax 安全模型标准

> **文档定位**: Airymax 开放标准
> **版本**: 1.0（开放标准草案）
> **最后更新**: 2026-07-09
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 一、范围与目的

本标准定义 AI Agent 安全模型的开放契约，覆盖：

- Cupolas capability 安全模型（38 ID 枚举 + 派生模型：mint/mintcopy/derive/revoke）
- LSM 钩子标准（254 ID 枚举）
- 策略裁决结果 4 值标准（ALLOW/DENY/AUDIT/ENFORCE）
- Agent 权限最小化原则

本标准面向：

- **运行时实现者**：实现 capability 系统、LSM 钩子、策略裁决引擎。
- **Agent 作者**：编写符合最小权限原则的 Agent。
- **安全审计员**：审计 Agent 的能力授予与撤销轨迹。
- **合规集成方**：将企业安全策略接入 Airymax 生态。

本标准遵循 IRON-9 v2 [SC] 共享契约层——capability 令牌格式、LSM 钩子 ID、裁决枚举在 agentrt 与 agentrt-liunx 之间完全共享。

### 1.1 术语

| 术语 | 含义 |
|------|------|
| Cupolas | Airymax 安全穹顶，capability 安全引擎 |
| capability | 不可伪造的资源访问令牌 |
| mint | 从父能力派生子能力，权限不增 |
| mintcopy | 复制能力，权限相等 |
| derive | 派生子集能力 |
| revoke | 撤销能力，递归撤销派生链 |
| 裁决 4 值 | ALLOW / DENY / AUDIT / ENFORCE |

### 1.2 规范用词

沿用 RFC 2119 用词。

---

## 二、Cupolas Capability 安全模型

### 2.1 模型概述

Cupolas 实现基于 capability 的安全模型（参考 seL4）。核心原则：

1. **能力是唯一访问方式**：所有资源访问**必须**通过 capability 令牌。
2. **能力不可伪造**：令牌由运行时生成，用户态无法伪造。
3. **能力派生权限不增**：子能力权限**必须** <= 父能力权限。
4. **能力可撤销**：撤销**必须**递归影响所有派生能力。
5. **最小权限原则**：每个 Agent **必须**只拥有完成功能所需的最小能力。

### 2.2 Capability 令牌格式

令牌格式**永久稳定**（AOS-STD-SEC-001，L1 稳定级）：

```c
/**
 * are_cap_t - 能力令牌类型
 *
 * 能力令牌是资源的不可伪造引用。所有资源访问必须通过能力令牌。
 */
typedef struct {
    uint32_t id;           /* 偏移 0:  能力唯一标识符 */
    uint32_t type;         /* 偏移 4:  能力类型（文件、IPC、内存、网络等） */
    uint64_t permissions;  /* 偏移 8:  权限位掩码（64 位） */
    uint32_t owner;        /* 偏移 16: 能力所有者（Agent ID） */
    uint32_t parent;       /* 偏移 20: 父能力 ID（0 表示根能力） */
    uint64_t issue_time;   /* 偏移 24: 发行时间（纳秒时间戳） */
    uint64_t expire_time;  /* 偏移 32: 过期时间（0 表示永不过期） */
    uint8_t  reserved[24]; /* 偏移 40: 保留字段，填充 0 */
} are_cap_t;  /* 总长度 64 字节 */

_Static_assert(sizeof(are_cap_t) == 64, "are_cap_t must be 64 bytes");
```

| 字段 | 约束 |
|------|------|
| `id` | 运行时全局唯一，单调递增 |
| `type` | 取自能力域枚举，见 01 标准第 4.2 节 |
| `permissions` | 64 位位掩码，每位对应一个动作 |
| `owner` | 创建者 Agent ID，**不可变更** |
| `parent` | 父能力 ID，0 表示根能力（由运行时直接授予） |
| `issue_time` | 发行纳秒时间戳 |
| `expire_time` | 0 表示永不过期；非 0 表示过期时间 |

### 2.3 POSIX capability 38 ID 枚举

本标准在 Linux 标准 38 个 capability 基础上扩展为 41 ID 枚举（前 38 个与 Linux 一致，39-41 为 Airymax 专属）：

| ID | 名称 | 描述 | 风险等级 |
|----|------|------|----------|
| 0 | `CAP_CHOWN` | 修改文件所有者 | 高 |
| 1 | `CAP_DAC_OVERRIDE` | 绕过文件权限检查 | 高 |
| 2 | `CAP_DAC_READ_SEARCH` | 绕过文件读/搜索权限 | 中 |
| 3 | `CAP_FOWNER` | 绕过文件所有者检查 | 高 |
| 4 | `CAP_FSETID` | 设置文件 SUID/SGID | 高 |
| 5 | `CAP_KILL` | 向任意进程发送信号 | 中 |
| 6 | `CAP_SETGID` | 修改进程 GID | 高 |
| 7 | `CAP_SETUID` | 修改进程 UID | 高 |
| 8 | `CAP_SETPCAP` | 修改进程能力集 | 极高 |
| 9 | `CAP_NET_BIND_SERVICE` | 绑定特权端口 | 中 |
| 10 | `CAP_NET_BROADCAST` | 网络广播 | 低 |
| 11 | `CAP_NET_ADMIN` | 网络管理 | 高 |
| 12 | `CAP_NET_RAW` | 原始套接字 | 高 |
| 13 | `CAP_IPC_LOCK` | 锁定内存 | 中 |
| 14 | `CAP_IPC_OWNER` | 绕过 IPC 所有权检查 | 高 |
| 15 | `CAP_SYS_MODULE` | 加载内核模块 | 极高 |
| 16 | `CAP_SYS_RAWIO` | 原始 I/O 操作 | 极高 |
| 17 | `CAP_SYS_CHROOT` | 使用 chroot | 高 |
| 18 | `CAP_SYS_PTRACE` | 跟踪任意进程 | 极高 |
| 19 | `CAP_SYS_PACCT` | 进程记账 | 中 |
| 20 | `CAP_SYS_ADMIN` | 系统管理（万能） | 极高 |
| 21 | `CAP_SYS_BOOT` | 重启系统 | 高 |
| 22 | `CAP_SYS_NICE` | 修改进程优先级 | 中 |
| 23 | `CAP_SYS_RESOURCE` | 修改资源限制 | 中 |
| 24 | `CAP_SYS_TIME` | 修改系统时间 | 中 |
| 25 | `CAP_SYS_TTY_CONFIG` | TTY 配置 | 中 |
| 26 | `CAP_MKNOD` | 创建设备节点 | 高 |
| 27 | `CAP_LEASE` | 文件租约 | 低 |
| 28 | `CAP_AUDIT_WRITE` | 写入审计日志 | 中 |
| 29 | `CAP_AUDIT_CONTROL` | 审计控制 | 极高 |
| 30 | `CAP_SETFCAP` | 设置文件能力 | 高 |
| 31 | `CAP_MAC_OVERRIDE` | 绕过 MAC 策略 | 极高 |
| 32 | `CAP_MAC_ADMIN` | MAC 配置 | 极高 |
| 33 | `CAP_SYSLOG` | 读取内核日志 | 中 |
| 34 | `CAP_WAKE_ALARM` | 触发系统唤醒 | 低 |
| 35 | `CAP_BLOCK_SUSPEND` | 阻止系统挂起 | 低 |
| 36 | `CAP_AUDIT_READ` | 读取审计日志 | 中 |
| 37 | `CAP_PERFMON` | 性能监控 | 中 |
| 38 | `CAP_AGENT_ADMIN` | Agent 管理（Airymax 专属） | 极高 |
| 39 | `CAP_AGENT_SCHED` | Agent 调度（Airymax 专属） | 高 |
| 40 | `CAP_AGENT_SANDBOX` | Agent 沙箱管理（Airymax 专属） | 高 |

枚举值**永久稳定**（AOS-STD-SEC-002，L1 稳定级）；新增 capability **必须**追加到 ID 41 之后。

### 2.4 风险等级与默认策略

| 风险等级 | 默认裁决 | 授予要求 |
|---------|---------|---------|
| 极高 | DENY | 标准委员会特批 + 审计强制开启 |
| 高 | DENY | 显式声明 + 沙箱隔离 |
| 中 | AUDIT | 显式声明 |
| 低 | ALLOW | 默认允许 |

运行时**必须**对极高风险 capability 强制 DENY，除非通过显式审计流程批准（AOS-STD-SEC-003）。

---

## 三、Capability 派生模型

### 3.1 四种派生操作

| 操作 | 符号 | 语义 | 约束 |
|------|------|------|------|
| **mint** | `mint(parent) -> child` | 从父能力创建子能力，子权限 ≤ 父权限 | **不得**扩大权限 |
| **mintcopy** | `mintcopy(src) -> dst` | 复制能力，dst 权限 = src 权限 | **不得**扩大权限 |
| **derive** | `derive(parent, subset) -> child` | 从父能力派生子集能力，只保留 subset 权限 | subset **必须**是父能力子集 |
| **revoke** | `revoke(cap)` | 撤销能力，递归撤销所有派生能力 | **不可逆**操作 |

### 3.2 派生接口

```c
/**
 * agentrt_cap_mint - 创建子能力（权限缩小或相等）
 * @parent:        父能力令牌
 * @permissions:   子能力权限位掩码（必须是 parent 的子集）
 * @ttl_seconds:   生存时间（秒，0 表示永不过期）
 * @child_out:     输出：子能力令牌
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   参数无效
 *   -AGENTRT_EPERM:    请求的权限超集于父能力
 *   -AGENTRT_ENOMEM:   内存不足
 *
 * 五维正交原则 K-2（接口契约化）：权限超集检查是硬性约束，不可绕过。
 */
int agentrt_cap_mint(are_cap_t parent, uint64_t permissions,
                     uint64_t ttl_seconds, are_cap_t *child_out);

/**
 * agentrt_cap_mintcopy - 复制能力（权限完全相等）
 */
int agentrt_cap_mintcopy(are_cap_t src, are_cap_t *dst_out);

/**
 * agentrt_cap_derive - 派生子集能力
 * @parent:  父能力令牌
 * @subset:  需要的权限子集（位掩码）
 * @child_out: 输出：派生能力令牌
 *
 * 常见用途：从文件读写能力派生出只读能力。
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   参数无效
 *   -AGENTRT_EPERM:    subset 不是 parent 的子集
 */
int agentrt_cap_derive(are_cap_t parent, uint64_t subset,
                       are_cap_t *child_out);

/**
 * agentrt_cap_revoke - 撤销能力
 * @cap: 要撤销的能力令牌
 *
 * 撤销操作递归撤销所有子能力，是不可逆操作。
 * 所有持有该能力或其子能力的 Agent 将立即失去对应权限。
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   参数无效
 *   -AGENTRT_EPERM:    权限不足（不是能力所有者）
 */
int agentrt_cap_revoke(are_cap_t cap);
```

接口签名**永久稳定**（AOS-STD-SEC-004 ~ AOS-STD-SEC-007，L1 稳定级）。

### 3.3 派生约束

| 约束 | 标准条目 | 违反后果 |
|------|---------|---------|
| mint 子权限**必须**是父权限子集 | AOS-STD-SEC-008 | 返回 EPERM |
| mintcopy 权限**必须**完全相等 | AOS-STD-SEC-009 | 返回 EPERM |
| derive subset **必须**是父权限子集 | AOS-STD-SEC-010 | 返回 EPERM |
| revoke **必须**递归撤销派生链 | AOS-STD-SEC-011 | 派生令牌立即失效 |
| 派生链最大深度 16 层 | AOS-STD-SEC-012 | 超过返回 ELIMIT |
| 令牌可设置 TTL，超时自动失效 | AOS-STD-SEC-013 | 过期返回 ECAP_EXPIRED |
| 令牌可标记单次使用，使用后自动撤销 | AOS-STD-SEC-014 | 重复使用返回 EBUSY |

### 3.4 派生流程伪代码

```
function cap_mint(parent, requested_perms, ttl):
    # 1. 权限超集检查（硬性约束）
    if (requested_perms & ~parent.permissions) != 0:
        return -EPERM  # 不得扩大权限

    # 2. 创建子令牌
    child = new are_cap_t
    child.id           = next_cap_id()
    child.type         = parent.type
    child.permissions  = requested_perms  # 子权限（已校验为子集）
    child.owner        = parent.owner
    child.parent       = parent.id        # 记录派生链
    child.issue_time   = now_ns()
    child.expire_time  = (ttl == 0) ? 0 : (now_ns() + ttl * 1e9)

    # 3. 注册到派生树（用于递归撤销）
    cap_tree_insert(parent.id, child.id)

    # 4. 写审计日志
    audit_log(AUDIT_CAP_MINT, parent, child, requested_perms)

    return child


function cap_revoke(cap):
    # 1. 检查调用方是否为所有者
    if caller != cap.owner:
        return -EPERM

    # 2. 递归收集所有派生令牌
    descendants = cap_tree_collect_descendants(cap.id)

    # 3. 逐个撤销（深度优先）
    for child in descendants (DFS order):
        invalidate(child.id)
        audit_log(AUDIT_CAP_REVOKE, cap, child)

    # 4. 撤销自身
    invalidate(cap.id)
    audit_log(AUDIT_CAP_REVOKE, cap, cap)

    return 0
```

---

## 四、LSM 钩子标准

### 4.1 钩子 ID 枚举（254 ID）

本标准定义 254 个 LSM 钩子 ID，覆盖 9 个类别：

| 类别 | 钩子数量 | ID 范围 | 描述 |
|------|---------|---------|------|
| 进程管理 | 42 | 0-41 | 进程创建、销毁、信号发送 |
| 文件系统 | 58 | 42-99 | 文件打开、读写、属性修改 |
| IPC | 35 | 100-134 | IPC 消息发送、接收、端点管理 |
| 网络 | 41 | 135-175 | 网络连接、数据包过滤 |
| 能力管理 | 22 | 176-197 | 能力创建、派生、撤销、使用 |
| 沙箱管理 | 18 | 198-215 | 沙箱创建、进入、退出、升级 |
| 审计管理 | 15 | 216-230 | 审计日志写入、查询、配置 |
| 认知管理 | 12 | 231-242 | 认知任务创建、调度、执行 |
| 内存管理 | 11 | 243-253 | 内存分配、映射、防护 |
| **总计** | **254** | **0-253** | |

ID 范围**永久稳定**（AOS-STD-SEC-015，L1 稳定级）；新增钩子**必须**追加到 254 之后，**不得**修改既有 ID。

### 4.2 钩子类别命名

```c
typedef enum {
    AGENTRT_LSM_PROC    = 0,   /* 进程管理：0-41 */
    AGENTRT_LSM_FILE    = 1,   /* 文件系统：42-99 */
    AGENTRT_LSM_IPC     = 2,   /* IPC：100-134 */
    AGENTRT_LSM_NET     = 3,   /* 网络：135-175 */
    AGENTRT_LSM_CAP     = 4,   /* 能力管理：176-197 */
    AGENTRT_LSM_SANDBOX = 5,   /* 沙箱管理：198-215 */
    AGENTRT_LSM_AUDIT   = 6,   /* 审计管理：216-230 */
    AGENTRT_LSM_COG     = 7,   /* 认知管理：231-242 */
    AGENTRT_LSM_MEM     = 8,   /* 内存管理：243-253 */
} agentrt_lsm_category_t;
```

### 4.3 钩子注册接口

```c
/**
 * agentrt_lsm_register - 注册安全模块
 * @ops:  安全操作集（钩子函数指针表）
 * @name: 模块名称（最多 64 字符）
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   参数无效
 *   -AGENTRT_EBUSY:    同名模块已注册
 *   -AGENTRT_EPERM:    权限不足（需要 CAP_MAC_ADMIN）
 */
int agentrt_lsm_register(const struct agentrt_security_ops *ops,
                         const char *name);
```

### 4.4 Agent 专属 LSM 钩子

```c
struct agentrt_agent_security_ops {
    /* 能力管理钩子（ID 176-197） */
    int (*cap_mint_permission)(are_cap_t parent, uint64_t requested_perms);
    int (*cap_derive_permission)(are_cap_t parent, uint64_t subset);
    int (*cap_revoke_permission)(are_cap_t target);

    /* 沙箱钩子（ID 198-215） */
    int (*sandbox_create_permission)(uint32_t level);
    int (*sandbox_enter_permission)(are_cap_t agent, are_cap_t sandbox);
    int (*sandbox_escape_detected)(are_cap_t sandbox, const char *details);

    /* Agent 通信钩子 */
    int (*agent_ipc_permission)(are_cap_t src, are_cap_t dst,
                                const char *src_ns, const char *dst_ns);
    int (*agent_network_egress)(are_cap_t agent, uint32_t dst_ip,
                                uint16_t dst_port);

    /* 认知安全钩子 */
    int (*cog_model_permission)(are_cap_t agent, const char *model_id);
    int (*cog_prompt_filter)(are_cap_t agent, const char *prompt, size_t len);
    int (*cog_response_filter)(are_cap_t agent, const char *response, size_t len);
};
```

---

## 五、策略裁决结果 4 值标准

### 5.1 裁决枚举

安全策略裁决返回 4 种结果之一（AOS-STD-SEC-016，L1 稳定级）：

```c
typedef enum {
    AGENTRT_DECISION_ALLOW   = 0,  /* 允许操作 */
    AGENTRT_DECISION_DENY    = 1,  /* 拒绝操作 */
    AGENTRT_DECISION_AUDIT   = 2,  /* 允许但记录审计 */
    AGENTRT_DECISION_ENFORCE = 3,  /* 强制执行（如强制沙箱升级） */
} agentrt_decision_t;
```

### 5.2 裁决语义

| 裁决 | 行为 | 审计 | 典型场景 |
|------|------|------|---------|
| ALLOW | 操作继续 | 否 | 低风险操作 |
| DENY | 操作被拒绝，返回 EPERM | 是（必记） | 高风险未授权操作 |
| AUDIT | 操作继续，强制记录审计 | 是（必记） | 中风险但合规操作 |
| ENFORCE | 操作继续但触发强制动作 | 是（必记） | 强制沙箱升级、强制加密 |

### 5.3 裁决优先级

多个安全模块对同一操作给出不同裁决时，按以下优先级取最终裁决（AOS-STD-SEC-017）：

```
DENY > ENFORCE > AUDIT > ALLOW
```

即：任一模块 DENY 则最终 DENY；无 DENY 但有 ENFORCE 则最终 ENFORCE；以此类推。运行时**必须**按此优先级合成最终裁决（AOS-STD-SEC-017，L1 稳定级）。

---

## 六、Agent 权限最小化原则

### 6.1 最小权限要求

每个 Agent 在创建时**必须**通过能力声明（见 01 标准第 4 节）声明其所需的最小能力集合。运行时**必须**：

1. **必须**仅授予声明的能力，**不得**授予未声明能力（AOS-STD-SEC-018）。
2. **必须**根据 `constraints` 将约束写入 capability 令牌（AOS-STD-SEC-019）。
3. **应当**对高风险 capability 启用强制审计（AOS-STD-SEC-020）。
4. **应当**对极高风险 capability 强制沙箱隔离（AOS-STD-SEC-021）。

### 6.2 默认拒绝原则

未在能力声明中列出的资源访问，运行时**必须**默认 DENY（AOS-STD-SEC-022，L1 稳定级）。Agent 需要访问未声明资源时，**必须**重新注册并更新能力声明。

### 6.3 能力授予审计

每次能力授予、派生、撤销**必须**写入审计日志（AOS-STD-SEC-023，L1 稳定级）：

```c
typedef struct __attribute__((packed)) {
    uint64_t timestamp_sec;     /* 时间戳（秒） */
    uint64_t timestamp_nsec;    /* 时间戳（纳秒） */
    uint8_t  trace_id[16];     /* 关联的 trace_id */
    uint32_t event_id;         /* 事件唯一 ID */
    uint32_t event_type;       /* AUDIT_CAP_MINT / REVOKE / USE */
    uint32_t agent_id;         /* 触发事件的 Agent ID */
    uint32_t subject_cap;      /* 主体能力 ID */
    uint32_t object_cap;       /* 客体能力 ID */
    uint32_t operation;        /* 操作类型 */
    int32_t  result;           /* 操作结果（0 成功，负值错误） */
    uint8_t  severity;         /* 严重级别 (0-7) */
    char     description[128]; /* 事件描述 */
} agentrt_audit_entry_t;
```

审计日志**不可修改、不可删除**（仅追加，AOS-STD-SEC-024，L1 稳定级）；保留至少 90 天。

---

## 七、五级沙箱模型（与 03 主题相关引用）

本标准引用五级沙箱模型（详细定义见运行时工程标准），仅约束其与安全模型的接口：

| 级别 | 名称 | 隔离方式 | 安全约束 |
|------|------|---------|---------|
| 0 | 无隔离 | 共享进程空间 | 仅用于完全可信的内部 Agent |
| 1 | 进程隔离 | 独立进程 + namespace | Linux capability 限制 |
| 2 | 容器隔离 | 容器 + seccomp + capability | 默认级别 |
| 3 | 虚拟机隔离 | microVM | 不可信外部 Agent |
| 4 | 硬件隔离 | TEE / 独立物理机 | 敏感数据处理 |

沙箱级别**只能升级不能降级**（AOS-STD-SEC-025，单向操作）。

---

## 八、标准条目注册表

| 编号 | 名称 | 稳定级 |
|------|------|--------|
| AOS-STD-SEC-001 | capability 令牌格式 64 字节 | L1 |
| AOS-STD-SEC-002 | POSIX capability 38+3 ID 枚举 | L1 |
| AOS-STD-SEC-003 | 极高风险默认 DENY | L1 |
| AOS-STD-SEC-004 | cap_mint 接口 | L1 |
| AOS-STD-SEC-005 | cap_mintcopy 接口 | L1 |
| AOS-STD-SEC-006 | cap_derive 接口 | L1 |
| AOS-STD-SEC-007 | cap_revoke 接口 | L1 |
| AOS-STD-SEC-008 | mint 子集约束 | L1 |
| AOS-STD-SEC-009 | mintcopy 等权约束 | L1 |
| AOS-STD-SEC-010 | derive 子集约束 | L1 |
| AOS-STD-SEC-011 | revoke 递归撤销 | L1 |
| AOS-STD-SEC-012 | 派生链深度 16 | L2 |
| AOS-STD-SEC-013 | TTL 过期 | L1 |
| AOS-STD-SEC-014 | 单次使用 | L2 |
| AOS-STD-SEC-015 | LSM 钩子 254 ID | L1 |
| AOS-STD-SEC-016 | 裁决 4 值枚举 | L1 |
| AOS-STD-SEC-017 | 裁决优先级 DENY>ENFORCE>AUDIT>ALLOW | L1 |
| AOS-STD-SEC-018 | 最小权限授予 | L1 |
| AOS-STD-SEC-019 | constraints 写入令牌 | L1 |
| AOS-STD-SEC-020 | 高风险强制审计 | L2 |
| AOS-STD-SEC-021 | 极高风险强制沙箱 | L2 |
| AOS-STD-SEC-022 | 默认拒绝 | L1 |
| AOS-STD-SEC-023 | 能力操作审计 | L1 |
| AOS-STD-SEC-024 | 审计日志仅追加 | L1 |
| AOS-STD-SEC-025 | 沙箱单向升级 | L1 |

---

## 九、统一错误码体系

本标准使用 Airymax 统一错误码体系（AOS-STD-SEC-026，L1 稳定级）：

| 段 | 范围 | 含义 | 示例 |
|----|------|------|------|
| 成功 | 0 | `AGENTRT_OK` | `0` |
| 通用 | -1 ~ -99 | 通用错误 | `AGENTRT_EINVAL=-2` |
| 系统 | -100 ~ -199 | 系统级 | `AGENTRT_ETIMEOUT=-1004` |
| 内存 | -1100 ~ -1199 | 内存错误 | `AGENTRT_ENOMEM=-1100` |
| IPC | -1200 ~ -1299 | IPC 错误 | `AGENTRT_EMSGSIZE=-1201` |
| 安全 | -1300 ~ -1399 | 安全错误 | `AGENTRT_EPERM=-1302` |
| 沙箱 | -1400 ~ -1499 | 沙箱错误 | `AGENTRT_SANDBOX_FULL=-1401` |
| 能力 | -1500 ~ -1599 | 能力错误 | `AGENTRT_CAP_EXPIRED=-1501` |
| 调度 | -1600 ~ -1699 | 调度错误 | `AGENTRT_SCHED_EXHAUSTED=-1601` |
| 认知 | -1700 ~ -1799 | 认知错误 | `AGENTRT_COG_TIMEOUT=-1701` |
| 审计 | -1800 ~ -1899 | 审计错误 | `AGENTRT_AUDIT_DISK_FULL=-1801` |

错误码一旦定义**永不变更**（AOS-STD-SEC-027，L1 稳定级）；新错误码只能追加，**不得**删除或重新编号。

---

## 十、兼容性测试要求

| 测试 ID | 测试内容 | 通过条件 |
|---------|---------|---------|
| SEC-T-001 | 令牌字节布局 | 64 字节，字段偏移精确匹配 |
| SEC-T-002 | mint 权限超集 | 请求超集返回 EPERM |
| SEC-T-003 | mintcopy 等权 | 复制后权限完全相等 |
| SEC-T-004 | derive 子集 | subset 非子集返回 EPERM |
| SEC-T-005 | revoke 递归 | 撤销父令牌后所有子令牌失效 |
| SEC-T-006 | 派生深度限制 | 第 17 层派生返回 ELIMIT |
| SEC-T-007 | TTL 过期 | 过期令牌返回 ECAP_EXPIRED |
| SEC-T-008 | 裁决优先级 | DENY 压倒其他裁决 |
| SEC-T-009 | 默认拒绝 | 未声明能力访问被拒绝 |
| SEC-T-010 | 审计完整性 | 审计日志不可修改删除 |
| SEC-T-011 | LSM 钩子 ID | 254 个 ID 范围精确匹配 |

---

## 十一、最小示例

### 11.1 Capability 令牌格式与派生

```c
/* 父能力：文件读写 /var/cache/weather/ */
are_cap_t parent_cap = {
    .id          = 1001,
    .type        = AGENT_CAP_DOMAIN_FILE,
    .permissions = 0x03,   /* read(0x01) | write(0x02) */
    .owner       = 42,     /* Agent ID */
    .parent      = 0,      /* 根能力 */
    .issue_time  = 1780000000000000000ULL,
    .expire_time = 0,      /* 永不过期 */
};

/* 派生子能力：只读，TTL 3600 秒 */
are_cap_t child_cap;
int rc = agentrt_cap_derive(parent_cap,
                             /* subset */ 0x01,  /* read only */
                             &child_cap);
if (rc == -AGENTRT_EPERM) {
    /* subset 不是 parent 的子集，被拒绝 */
}

/* 撤销父能力，子能力自动失效 */
agentrt_cap_revoke(parent_cap);
/* 此后 child_cap 立即不可用 */
```

### 11.2 策略裁决合成

```c
int final_decision = AGENTRT_DECISION_ALLOW;

/* 安全模块 A：AUDIT */
int dec_a = module_a_check_cap_mint(parent, perms);
final_decision = merge_decision(final_decision, dec_a);

/* 安全模块 B：ENFORCE */
int dec_b = module_b_check_cap_mint(parent, perms);
final_decision = merge_decision(final_decision, dec_b);

/* 安全模块 C：DENY */
int dec_c = module_c_check_cap_mint(parent, perms);
final_decision = merge_decision(final_decision, dec_c);

/* 最终：DENY（优先级最高） */
assert(final_decision == AGENTRT_DECISION_DENY);

int merge_decision(int current, int incoming) {
    /* 优先级：DENY > ENFORCE > AUDIT > ALLOW */
    if (incoming == AGENTRT_DECISION_DENY)    return AGENTRT_DECISION_DENY;
    if (incoming == AGENTRT_DECISION_ENFORCE && current != AGENTRT_DECISION_DENY)
        return AGENTRT_DECISION_ENFORCE;
    if (incoming == AGENTRT_DECISION_AUDIT && current == AGENTRT_DECISION_ALLOW)
        return AGENTRT_DECISION_AUDIT;
    return current;
}
```

---

## 十二、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-07-09 | 初始草案：capability 38+3 ID 枚举、派生模型 mint/mintcopy/derive/revoke、LSM 254 ID、裁决 4 值、最小权限原则 |

---

## 十三、参考文献

- [Airymax 开放标准体系总览](./README.md)
- [Airymax Agent 运行时标准](./01-airymax-agent-runtime-standard.md)
- [Airymax IPC 协议标准](./02-airymax-ipc-protocol-standard.md)
- seL4 capability 安全模型：https://sel4.systems/
- Linux capabilities(7)：https://man7.org/linux/man-pages/man7/capabilities.7.html
- Linux LSM 框架：https://docs.kernel.org/admin-guide/LSM/index.html

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
