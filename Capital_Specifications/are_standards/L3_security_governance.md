# ARE Standards L3 — 安全与治理标准

> **版本**: v0.1.0-draft | **状态**: 草案（v0.1.1 发行前冻结）| **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）
> **标准层级**: L3 | **回滚至**: [ARE Standards 总览](README.md)

---

## 1. 标准范围与目标

### 1.1 范围

本标准规定 Agent Runtime Environment（ARE）的**安全控制面**与**治理规范**，包括：

- Cupolas 权限引擎接口（基于主体的访问裁决）；
- 五级沙箱隔离规范（none/mem/cpu/fs/net）；
- 系统调用 5 类划分与 4 级保护环；
- 统一错误码体系（权威源唯一性、分段、禁令）；
- 审计日志格式（结构化 JSON）；
- 数据保护与敏感信息脱敏规范；
- 双许可证体系（AGPL v3 + Apache 2.0，SPDX: `AGPL-3.0-or-later OR Apache-2.0`）；
- SBOM 软件物料清单规范；
- 合规检查清单与第三方实现安全审计要求。

本标准**不**规定：
- L1 内核原语实现（见 [L1_runtime_interface.md](L1_runtime_interface.md)）；
- L2 IPC Bus 协议（见 [L2_service_protocol.md](L2_service_protocol.md)）；
- 具体加密算法选择（AES-GCM/ChaCha20-Poly1305 由实现者选择，但必须 AEAD）。

### 1.2 目标

| 目标 | 描述 |
|------|------|
| **零信任** | 默认拒绝（fail-closed），所有访问必须显式授权 |
| **可审计** | 所有安全相关操作必须留痕，审计日志不可篡改 |
| **跨实现一致** | 不同实现者对同一权限请求的裁决结果一致 |
| **错误码统一** | 全 ARE 生态只有一套错误码体系，禁止多套并行 |
| **许可证清晰** | 不同层级的组件适用不同许可证，避免许可证污染 |

### 1.3 关键术语

| 术语 | 定义 |
|------|------|
| **Cupolas** | Airymax 的安全穹顶模块，提供权限裁决、输入净化、审计、虚拟工位 |
| **subject** | 访问主体（Agent / 用户 / daemon），权限裁决的发起方 |
| **resource** | 访问客体（文件、工具、记忆、API 等），权限裁决的目标 |
| **沙箱（sandbox）** | 隔离执行环境，限制 syscall 可见性与资源配额 |
| **保护环（protection ring）** | 4 级特权层级，约束 syscall 调用来源 |

---

## 2. Cupolas 权限引擎接口

### 2.1 裁决模型

Cupolas 采用 **ABAC（Attribute-Based Access Control）** 模型，裁决函数接收四元组：

```
are_permission_check(subject, action, resource, context) → allow / deny / ask
```

- `subject`：主体标识（如 `agent:agent_42`、`daemon:tool_d`、`user:alice`）；
- `action`：动作类型（`read` / `write` / `execute` / `delete` / `admin`）；
- `resource`：资源路径（如 `tool:rm`、`mem:session_123`、`file:/etc/passwd`）；
- `context`：可选上下文（JSON 字符串，包含 IP、时间、风险评分等）。

### 2.2 裁决结果枚举

```c
typedef enum {
    ARE_PERM_ALLOW = 1,   /* 显式允许 */
    ARE_PERM_DENY  = 0,   /* 显式拒绝 */
    ARE_PERM_ASK   = -1,  /* 需用户确认（交互式场景） */
} are_perm_decision_t;
```

**默认拒绝原则**：当无任何规则匹配时，必须返回 `ARE_PERM_DENY`，不得默认放行。

### 2.3 接口签名

```c
typedef struct are_permission_ctx {
    const char *subject;     /* 主体标识，非 NULL */
    const char *action;       /* 动作，非 NULL */
    const char *resource;     /* 资源路径，非 NULL */
    const char *context_json; /* 上下文 JSON，可为 NULL */
} are_permission_ctx_t;

/* 权限裁决。返回 ARE_PERM_ALLOW / ARE_PERM_DENY / ARE_PERM_ASK。
 * 错误时返回负 ARE 错误码（如 ARE_ERR_SEC_PERMISSION）。
 * @threadsafe 是  @reentrant 是
 * @note 裁决结果可缓存，但缓存失效后必须重新裁决 */
are_perm_decision_t are_permission_check(const char *subject,
                                         const char *action,
                                         const char *resource,
                                         const char *context_json);

/* 带结构化上下文的便捷重载。 */
are_perm_decision_t are_permission_check_ctx(const are_permission_ctx_t *ctx);

/* 添加权限规则。priority 越高越优先；同优先级时 DENY 优先于 ALLOW。
 * subject/action/resource 支持通配符 '*' 与 glob（如 'tool:*'）。
 * @return ARE_OK / ARE_EINVAL / ARE_ENOMEM */
are_error_t are_permission_add_rule(const char *subject,
                                    const char *action,
                                    const char *resource,
                                    are_perm_decision_t decision,
                                    int priority);

/* 清空权限缓存（配置变更后调用）。 */
void are_permission_clear_cache(void);

/* 初始化 Cupolas 权限引擎。config_path 为配置文件路径，NULL 使用默认。 */
are_error_t are_cupolas_init(const char *config_path);

/* 清理 Cupolas 模块，释放全部资源。 */
void are_cupolas_cleanup(void);
```

### 2.4 规则匹配优先级

按下列顺序匹配，命中即返回：

1. **显式 DENY**（最高优先级，优先级数最高的 DENY 规则）；
2. **显式 ASK**（次高优先级）；
3. **显式 ALLOW**（按 priority 降序）；
4. **默认 DENY**（无匹配）。

通配符 `*` 匹配任意字符序列；`?` 匹配单个字符。

### 2.5 裁决缓存

为提升性能，裁决结果可缓存。缓存键 = `hash(subject) + hash(action) + hash(resource)`。缓存约束：

- 缓存 TTL 默认 60 秒，可配置；
- `are_permission_add_rule` 后必须自动清空缓存；
- ASK 结果**不缓存**（每次都需用户确认）；
- 缓存命中率应 > 90%（生产环境观测指标）。

### 2.6 参考实现

Airymax 原生实现：`agentos/cupolas/include/cupolas.h`（`cupolas_check_permission`、`cupolas_add_permission_rule`、`cupolas_clear_permission_cache`）。L3 标准 ABI 通过适配层桥接至 `are_permission_*`。

---

## 3. 五级沙箱隔离规范

### 3.1 沙箱级别

| 级别 | 宏 | 隔离内容 | 适用场景 | 性能开销 |
|------|----|----------|----------|----------|
| 0 | `ARE_SANDBOX_NONE` | 无隔离 | 受信内部 daemon（如 cupolas_d 自身） | 无 |
| 1 | `ARE_SANDBOX_MEM` | 内存隔离（地址空间独立、配额限制） | 计算密集型 Agent | 低 |
| 2 | `ARE_SANDBOX_CPU` | CPU 隔离（cgroup 限流、时间片配额） | 长时任务、防死循环 | 低 |
| 3 | `ARE_SANDBOX_FS` | 文件系统隔离（chroot/namespace、路径白名单） | 工具执行、Skill 安装 | 中 |
| 4 | `ARE_SANDBOX_NET` | 网络隔离（仅允许白名单域名/端口） | 不可信代码执行、第三方插件 | 中高 |

级别**累积生效**：`ARE_SANDBOX_NET` 隐含 mem + cpu + fs 全部约束。

### 3.2 沙箱接口

```c
typedef struct are_sandbox are_sandbox_t;

typedef struct {
    int       level;            /* ARE_SANDBOX_* 之一 */
    uint32_t  timeout_ms;       /* 单次执行超时，默认 30000 */
    size_t    mem_quota_bytes; /* 内存配额，0=不限 */
    uint32_t  cpu_quota_pct;   /* CPU 配额百分比，0=不限 */
    char     *fs_root;          /* chroot 根，NULL=不隔离 fs */
    char    **net_whitelist;    /* 域名/端口白名单，NULL=完全禁止网络 */
    int       priority;         /* 0=普通 */
} are_sandbox_config_t;

/* 初始化沙箱管理器（进程级，幂等） */
are_error_t are_sandbox_manager_init(void);
void        are_sandbox_manager_destroy(void);

/* 创建沙箱实例。config=NULL 使用默认（ARE_SANDBOX_FS + 30s 超时）。 */
are_error_t are_sandbox_create(const are_sandbox_config_t *config,
                               const char *owner_id,
                               are_sandbox_t **out);

/* 添加权限规则：在沙箱内允许/拒绝特定 syscall。 */
are_error_t are_sandbox_add_rule(are_sandbox_t *sb,
                                 int syscall_num,
                                 are_perm_decision_t decision,
                                 const char *condition_json);

/* 在沙箱中执行系统调用。
 * 执行流程：状态检查 → permission_check → quota_check → invoke → audit_log
 * @return ARE_OK / ARE_EPERM（沙箱终止）/ ARE_EACCES（权限拒绝）/
 *         ARE_EQUOTA（配额超限）/ ARE_EINVAL */
are_error_t are_sandbox_invoke(are_sandbox_t *sb,
                               int syscall_num,
                               void **args, int argc,
                               void **out_result);

void        are_sandbox_destroy(are_sandbox_t *sb);
```

### 3.3 fail-closed 原则

`are_sandbox_invoke` 在 `sb == NULL` 时**必须**拒绝执行（由调用方检查返回值，调用方未检查时视为失败）。沙箱初始化失败时，所有后续 invoke 必须返回 `ARE_EPERM`，**不得**回退到无沙箱模式。

### 3.4 配额超限处理

| 配额类型 | 超限行为 | 错误码 |
|---------|----------|--------|
| 内存 | 拒绝分配，OOM 触发降级 | `ARE_ERR_SEC_QUOTA` (-706) |
| CPU 时间 | 强制终止任务 | `ARE_ERR_SEC_QUOTA` |
| 磁盘空间 | 拒绝写入 | `ARE_ERR_SEC_QUOTA` |
| 网络连接数 | 拒绝新连接 | `ARE_ERR_SEC_QUOTA` |

### 3.5 参考实现

Airymax 原生实现：`agentos/atoms/syscall/include/agentos_sandbox.h`（`agentos_sandbox_*`）。原生权限枚举 `PERM_ALLOW/PERM_DENY/PERM_ASK` 与本标准 `ARE_PERM_*` 一一对应。

---

## 4. 系统调用 5 类划分与 4 级保护环

### 4.1 5 类系统调用

ARE 系统调用按职责划分为 5 类，每类对应一组 syscall 号段。新增 syscall 必须归入对应类别，不得跨类：

| 类别 | syscall 号段 | 说明 | 示例 syscall |
|------|-------------|------|-------------|
| **任务管理（Task）** | 1–19 | 任务提交、查询、取消、Agent spawn/terminate/invoke | `SYS_TASK_SUBMIT=1`, `SYS_AGENT_SPAWN=15` |
| **记忆管理（Memory）** | 20–39 | 记忆读写、检索、演化 | `SYS_MEMORY_WRITE=5`, `SYS_MEMORY_SEARCH=6` |
| **会话管理（Session）** | 40–59 | 会话创建、查询、关闭、列表 | `SYS_SESSION_CREATE=9` |
| **可观测性（Telemetry）** | 60–79 | 指标查询、链路追踪 | `SYS_TELEMETRY_METRICS=13` |
| **工具与技能（Tool & Skill）** | 80–99 | 工具执行、Skill 安装/卸载/执行 | `SYS_TOOL_EXECUTE=23`, `SYS_SKILL_INSTALL=19` |

> 参考实现：`agentos/atoms/syscall/include/syscalls.h`。原生 `agentos_syscall_num_t` 枚举与本标准一致。

### 4.2 4 级保护环

借鉴硬件保护环概念，ARE 将 syscall 调用来源分为 4 级特权层级。低特权环不得调用仅高特权环可用的 syscall：

| 环 | 宏 | 特权层级 | 调用来源 | 可用 syscall |
|----|----|----------|----------|--------------|
| Ring 0 | `ARE_RING_KERNEL` | 最高 | 微内核自身、cupolas_d 权限裁决 | 全部 syscall（含 admin 类） |
| Ring 1 | `ARE_RING_DAEMON` | 高 | ARE 标准注册的 12 个 daemon | 除 admin 外的全部 syscall |
| Ring 2 | `ARE_RING_AGENT` | 中 | 普通用户态 Agent | 任务、记忆、会话、技能执行（不含 admin） |
| Ring 3 | `ARE_RING_UNTRUSTED` | 最低 | 第三方插件、不可信代码、用户上传脚本 | 仅只读查询与受限工具执行 |

```c
typedef enum {
    ARE_RING_KERNEL    = 0,
    ARE_RING_DAEMON    = 1,
    ARE_RING_AGENT     = 2,
    ARE_RING_UNTRUSTED = 3,
} are_protection_ring_t;
```

### 4.3 环-syscall 授权矩阵

| syscall 类别 | Ring 0 | Ring 1 | Ring 2 | Ring 3 |
|-------------|--------|--------|--------|--------|
| Task（提交/取消） | ✓ | ✓ | ✓（仅自己提交的） | ✗ |
| Task（查询） | ✓ | ✓ | ✓ | ✓（受限） |
| Memory（写） | ✓ | ✓ | ✓（仅自己 session） | ✗ |
| Memory（读/检索） | ✓ | ✓ | ✓ | ✓（白名单资源） |
| Session（创建/关闭） | ✓ | ✓ | ✓ | ✗ |
| Session（查询） | ✓ | ✓ | ✓ | ✓ |
| Telemetry（metrics） | ✓ | ✓ | ✓（脱敏） | ✓（脱敏） |
| Telemetry（traces） | ✓ | ✓ | ✓（仅自己 trace） | ✗ |
| Tool execute | ✓ | ✓ | ✓（经权限裁决） | ✓（沙箱内 + 白名单工具） |
| Skill install/uninstall | ✓ | ✓ | ✗ | ✗ |
| Skill execute | ✓ | ✓ | ✓ | ✓（沙箱内） |

✗ 表示默认拒绝，cupolas 可通过显式规则开放（但 Ring 3 不得获得 Ring 0 专属能力）。

### 4.4 环判定与提权禁止

- 调用方在发起 syscall 时必须声明当前环（通过 IPC header 的 `source` 字段隐式判定）；
- **禁止跨环提权**：Ring 3 不得通过任何手段调用 Ring 0/1 专属 syscall；
- 跨环调用必须经 cupolas_d 裁决，未授权返回 `ARE_ERR_SEC_VIOLATION` (-701)。

---

## 5. 统一错误码体系规范

### 5.1 权威源唯一性

**全 ARE 生态只有一套错误码权威定义**，位于：

```
commons/utils/error/include/error.h
```

实现路径（Airymax 主仓库）：`agentos/commons/utils/error/include/error.h`。

**权威性约束**：
- 该文件是所有错误码的**唯一**定义源（single source of truth）；
- 所有 atoms、daemon、SDK、工具链代码必须 `#include` 此文件获取错误码宏，**禁止**在本地重复定义；
- 第三方实现者必须复制此文件至自有仓库的对应路径，**不得**自创错误码体系。

### 5.2 错误码分段

错误码类型为 `int32_t`（`agentos_error_t` / `are_error_t`），成功为 0，错误为负值。分段如下：

| 段 | 范围 | 类别 | 示例宏 | 值 | 含义 |
|----|------|------|--------|----|------|
| **通用** | -1 ~ -99 | 基础错误（参数、内存、权限、超时等） | `AGENTOS_ERR_INVALID_PARAM` | -2 | 参数无效 |
| | | | `AGENTOS_ERR_OUT_OF_MEMORY` | -4 | 内存不足 |
| | | | `AGENTOS_ERR_PERMISSION_DENIED` | -10 | 权限不足 |
| | | | `AGENTOS_ERR_TIMEOUT` | -8 | 操作超时 |
| | | | `AGENTOS_ERR_NOT_IMPLEMENTED` | -30 | 未实现 |
| **系统** | -100 ~ -199 | 平台、线程、socket、文件 | `AGENTOS_ERR_SYS_NOT_INIT` | -101 | 系统未初始化 |
| | | | `AGENTOS_ERR_SYS_MUTEX` | -105 | 互斥锁错误 |
| | | | `AGENTOS_ERR_SYS_SOCKET` | -109 | socket 错误 |
| **内核** | -200 ~ -299 | IPC、任务、调度、内存 | `AGENTOS_ERR_KERN_IPC` | -201 | 内核 IPC 错误 |
| | | | `AGENTOS_ERR_KERN_TASK` | -202 | 任务错误 |
| **服务** | -300 ~ -399 | 服务状态、依赖、负载均衡 | `AGENTOS_ERR_SVC_NOT_READY` | -301 | 服务未就绪 |
| | | | `AGENTOS_ERR_SVC_BUSY` | -302 | 服务忙（背压） |
| **LLM** | -400 ~ -499 | LLM provider、限流、上下文 | `AGENTOS_ERR_LLM_NO_PROVIDER` | -401 | 无可用 provider |
| | | | `AGENTOS_ERR_LLM_RATE_LIMIT` | -403 | 限流 |
| | | | `AGENTOS_ERR_LLM_CONTEXT_LEN` | -404 | 上下文超长 |
| **执行** | -500 ~ -599 | 工具执行、沙箱、超时 | `AGENTOS_ERR_EXEC_NOT_FOUND` | -501 | 工具未找到 |
| | | | `AGENTOS_ERR_EXEC_SANDBOX` | -505 | 沙箱违规 |
| **记忆** | -600 ~ -699 | 记忆读写、查询、演化 | `AGENTOS_ERR_MEM_WRITE` | -601 | 记忆写入失败 |
| | | | `AGENTOS_ERR_MEM_FULL` | -605 | 记忆存储满 |
| **安全** | -700 ~ -799 | 权限、净化、审计、沙箱 | `AGENTOS_ERR_SEC_VIOLATION` | -701 | 安全违规 |
| | | | `AGENTOS_ERR_SEC_SANITIZE` | -702 | 净化失败 |
| | | | `AGENTOS_ERR_SEC_QUOTA` | -706 | 配额超限 |
| | | | `AGENTOS_ERR_SEC_PATH_TRAV` | -709 | 路径遍历攻击 |
| **协议** | -800 ~ -889 | 编排、协调、补偿 | `AGENTOS_ERR_COORD_PLAN_FAIL` | -801 | 编排计划失败 |
| | | | `AGENTOS_ERR_COORD_RETRY_EXCEED` | -806 | 重试超限 |

> 完整定义见 `commons/utils/error/include/error.h`。L2 错误响应中的 `code` 字段直接使用上述错误码值。

### 5.3 禁令

为防止错误码碎片化，本标准规定以下**禁令**，违反即为不合规：

| # | 禁令 | 说明 |
|---|------|------|
| 1 | **禁止多套并行错误码系统** | 不得在 SDK、daemon、atoms 中定义独立的错误码枚举（如 `sdk_error_t`、`daemon_error_t`），所有代码必须使用 `agentos_error_t` |
| 2 | **禁止本地 `#define` 重定义错误码** | 不得在 `.c`/`.h` 文件中 `#define AGENTOS_EINVAL (-1)` 之类，必须 `#include "error.h"` |
| 3 | **禁止与字面量直接比较** | 调用方应使用语义宏（如 `if (rc == AGENTOS_ERR_TIMEOUT)`），禁止 `if (rc == -8)` |
| 4 | **禁止跨段借用** | LLM 类错误不得占用 -600 段；新增错误码必须归入对应段 |
| 5 | **禁止删除已发布错误码** | 已发布的错误码值必须保留，可标记 `deprecated` 但不得删除或重用 |

### 5.4 新增错误码流程

1. 在 `commons/utils/error/include/error.h` 对应段末尾追加 `#define`；
2. 在 `agentos_error_str()` 函数中添加可读描述；
3. 在 i18n 表中补充多语言描述（至少 en_US / zh_CN）；
4. 更新 L3 标准文档（MINOR+1）；
5. CTS 添加该错误码的触发用例。

### 5.5 POSIX 风格别名（兼容层）

为兼容 POSIX 习惯，`error.h` 提供部分别名（如 `AGENTOS_EINVAL` → `AGENTOS_ERR_INVALID_PARAM` 的等价映射）。这些别名**仅作过渡**，新代码必须使用 `AGENTOS_ERR_*` 形式。

---

## 6. 审计日志格式

### 6.1 结构化 JSON 格式

所有安全相关操作必须记录结构化审计日志，格式为单行 JSON（每行一条，便于日志聚合工具解析）：

```jsonc
{
  "timestamp": "2026-07-04T12:34:56.789Z",
  "timestamp_ns": 1783186496789000000,
  "trace_id": "01HX...",
  "actor": {
    "type": "agent",
    "id": "agent_42",
    "ring": 2
  },
  "action": "tool.execute",
  "resource": "tool:rm",
  "result": "deny",
  "reason": "permission denied by cupolas rule priority=100",
  "sandbox": {
    "level": "fs",
    "instance_id": "sb_abc123"
  },
  "context": {
    "ip": "10.0.0.1",
    "session_id": "sess_xyz",
    "risk_score": 0.42
  }
}
```

### 6.2 必填字段

审计日志**必须**包含下列字段，缺失任一即不合规：

| 字段 | 类型 | 说明 |
|------|------|------|
| `timestamp` | string (ISO 8601) | UTC 时间戳，毫秒精度 |
| `timestamp_ns` | uint64 | 纳秒时间戳（CLOCK_REALTIME），用于精确排序 |
| `trace_id` | string | 跨进程追踪 ID（与 IPC header 一致） |
| `actor` | object | 操作主体，含 `type`/`id`/`ring` |
| `action` | string | 操作类型（如 `tool.execute`、`cupolas.permission_check`） |
| `resource` | string | 资源路径（如 `tool:rm`、`mem:session_123`） |
| `result` | string | `allow` / `deny` / `ask` / `error` |

可选字段：`reason`、`sandbox`、`context`、`duration_ms`、`error_code`。

### 6.3 不可篡改性

- 审计日志写入后**禁止修改或删除**（append-only）；
- 实现者必须提供日志完整性校验（如链式哈希：每条日志包含前一条的 SHA-256）；
- 日志保留期默认 90 天，可配置，但生产环境**不得**低于 30 天；
- 日志存储路径必须在 `heapstore` 的 `ARE_HEAPSTORE_PATH_LOGS` 分区。

### 6.4 审计事件清单

下列事件**必须**记录审计日志：

| 事件 | action 字段 | result |
|------|------------|--------|
| 权限裁决 | `cupolas.permission_check` | allow/deny/ask |
| 沙箱违规 | `sandbox.violation` | deny |
| 配额超限 | `sandbox.quota_exceeded` | deny |
| 系统调用执行 | `<syscall>.invoke` | allow/deny |
| Agent spawn/terminate | `agent.spawn` / `agent.terminate` | allow |
| 工具执行 | `tool.execute` | allow/deny |
| Skill 安装/卸载 | `skill.install` / `skill.uninstall` | allow/deny |
| 配置变更 | `config.update` | allow |
| 沙箱创建/销毁 | `sandbox.create` / `sandbox.destroy` | allow |

---

## 7. 数据保护：敏感信息脱敏规范

### 7.1 必须脱敏的字段

下列字段在日志、审计、错误响应、trace 中**必须**脱敏（mask），不得明文记录：

| 字段类型 | 匹配模式 | 脱敏方式 | 示例 |
|---------|----------|----------|------|
| API key | `*_api_key`, `*_apikey`, `api_key` | 仅保留前 4 + 后 4 字符，中间 `****` | `sk-ab...xy12` → `sk-ab****xy12` |
| Token | `*_token`, `bearer`, `authorization` | 同上 | `Bearer eyJ...xyz` → `Bearer eyJ****xyz` |
| Password | `*_password`, `passwd`, `pwd` | 完全 mask | `secret123` → `********` |
| Secret | `*_secret` | 完全 mask | `my_secret` → `********` |
| 私钥 | `*_private_key`, `private_key` | 完全 mask | `-----BEGIN PRIVATE...` → `********` |
| 数据库连接串 | 含 `://*:*@` | 用户名密码部分 mask | `postgres://user:pass@host` → `postgres://user:****@host` |

### 7.2 脱敏实现接口

```c
/* 对 JSON 字符串中的敏感字段进行脱敏。
 * 输入：原始 JSON 字符串
 * 输出：脱敏后的 JSON 字符串（调用者负责释放）
 * @return ARE_OK / ARE_EINVAL / ARE_ENOMEM */
are_error_t are_redact_sensitive(const char *input_json,
                                 char **out_redacted);

/* 检查字符串是否包含敏感模式（用于自定义日志 handler）。
 * @return true 含敏感字段 / false 安全 */
bool are_contains_sensitive(const char *input);
```

### 7.3 脱敏规则扩展

实现者可通过配置文件扩展敏感字段模式：

```yaml
redaction:
  enabled: true
  patterns:
    - field: "*_api_key"
      strategy: partial      # partial | full
      keep_prefix: 4
      keep_suffix: 4
    - field: "authorization"
      strategy: full
    - field: "custom_secret"
      strategy: full
  # 自定义正则（高级）
  regex:
    - pattern: 'sk-[a-zA-Z0-9]{40,}'
      replacement: 'sk-****'
```

### 7.4 脱敏一致性要求

- **同一密钥在不同日志条目中脱敏结果必须一致**（便于关联分析）；
- 脱敏**不得**改变 JSON 结构（仅替换值，不删字段）；
- 脱敏后的字符串长度应与原长度相近（避免侧信道泄露长度信息）；
- **禁止**在 debug 日志中记录原始密钥（debug 构建也应脱敏）。

---

## 8. 许可证治理

### 8.1 双许可证体系

ARE 生态采用**统一双许可证**体系（SPDX: `AGPL-3.0-or-later OR Apache-2.0`，版权人 SPHARX Ltd.），所有源代码、构建脚本、文档与配套 NOTICE 文件统一适用，避免许可证碎片化与污染：

| 组件类别 | 许可证 | 适用范围 | 传染性 | 链接约束 |
|----------|--------|----------|--------|----------|
| **运行时核心** | AGPL-3.0-or-later OR Apache-2.0 | atoms 层（corekern/syscall/memory/taskflow）、cupolas、IPC Bus 核心、daemon、SDK、文档、Docker、Desktop | 双许可证：接收方可任选其一遵循 | 任选其一，AGPL 路径强传染（网络服务也须开源），Apache 路径不传染 |
| **工具链/SDK/文档** | AGPL-3.0-or-later OR Apache-2.0 | SDK（Python/Rust/Go/TypeScript）、CLI 工具、构建脚本、CTS 测试套件、ARE Standards 文档 | 同上 | 同上 |

> **历史决策**：v0.1.1 之前曾设计"三许可证体系（AGPL v3 + Apache 2.0 + SPHARX-Proprietary）+ CC-BY-SA-4.0"，已于 2026-07-04 由 SPHARX Ltd. 决定统一为 AGPL v3 + Apache 2.0 双许可证，取消 SPHARX-Proprietary 与 CC-BY-SA-4.0 路径。MemoryRovol 作为独立闭源仓库不在此体系内。

### 8.2 许可证兼容性矩阵

| 上游许可证 → 下游 | AGPL-3.0-or-later OR Apache-2.0（双许可证） |
|------------------|---------------------------------------------|
| AGPL-3.0-or-later OR Apache-2.0（双许可证） | ✓ 兼容（同源许可证） |
| Apache 2.0（仅） | ✓ 兼容（Apache 是双许可证子集） |
| AGPL v3（仅） | ✓ 兼容（AGPL 是双许可证子集，下游需选择 AGPL 路径） |
| MIT / BSD-3-Clause | ✓ 兼容（可被双许可证任一路径吸收） |
| GPL v3 | ✓ 兼容（仅 AGPL 路径，与 Apache 2.0 路径不兼容） |
| 专有/闭源 | ✗ 不兼容（除非仅链接 Apache 路径且不修改 Airymax 源代码） |

### 8.3 第三方实现者的许可证选择

| 实现目标 | 推荐许可证 | 约束 |
|----------|-----------|------|
| 实现完整 ARE 运行时（含 atoms 替代） | AGPL-3.0-or-later | 必须 opensource 全部修改 |
| 实现 daemon 兼容层（基于 Airymax atoms） | AGPL-3.0-or-later | 同上 |
| 实现 SDK 客户端 | Apache-2.0 / MIT / BSD | 可闭源 |
| 实现私有 daemon（不链接 AGPL 核心） | 任意 | 必须 IPC Bus 隔离，不得静态/动态链接 AGPL 代码 |

### 8.4 许可证声明要求

- 每个源文件顶部必须包含 SPDX-License-Identifier 标识（统一为 `AGPL-3.0-or-later OR Apache-2.0`）；
- 二进制分发必须附带 `LICENSE` 文件与 `NOTICE` 文件；
- AGPL 组件的衍生作品必须在用户界面显著位置声明（AGPL §5）；
- 不得移除或修改版权声明与许可证头。

```c
// SPDX-FileCopyrightText: 2026 SPHARX Ltd.
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
```

---

## 9. SBOM 软件物料清单规范

### 9.1 SBOM 格式

ARE 发布的每个二进制版本必须附带 SBOM，采用 [SPDX 2.3](https://spdx.dev/specifications/) 或 [CycloneDX 1.5](https://cyclonedx.org/) 格式。推荐 SPDX。

### 9.2 SBOM 必填字段

```jsonc
{
  "spdxVersion": "SPDX-2.3",
  "SPDXID": "SPDXRef-DOCUMENT",
  "name": "airymax-agentrt",
  "version": "0.1.1",
  "creationInfo": {
    "created": "2026-07-04T12:00:00Z",
    "creators": [
      "Organization: SPHARX Ltd.",
      "Tool: airymax-sbom-generator-1.0"
    ]
  },
  "packages": [
    {
      "SPDXID": "SPDXRef-Package-atoms-corekern",
      "name": "agentos-atoms-corekern",
      "versionInfo": "0.1.1",
      "licenseConcluded": "AGPL-3.0-only",
      "licenseDeclared": "AGPL-3.0-only",
      "supplier": "Organization: SPHARX Ltd.",
      "filesAnalyzed": true,
      "checksums": [
        { "algorithm": "SHA256", "value": "..." }
      ]
    },
    {
      "SPDXID": "SPDXRef-Package-sdk",
      "name": "agentos-sdk",
      "versionInfo": "0.1.1",
      "licenseConcluded": "Apache-2.0",
      "supplier": "Organization: SPHARX Ltd."
    }
  ],
  "relationships": [
    {
      "spdxElementId": "SPDXRef-DOCUMENT",
      "relationshipType": "DESCRIBES",
      "relatedSpdxElement": "SPDXRef-Package-atoms-corekern"
    }
  ]
}
```

### 9.3 SBOM 必填内容

每个 SBOM 必须包含：

| 字段 | 说明 |
|------|------|
| 组件名与版本 | 所有直接组件 |
| 许可证 | 每个组件的 SPDX 许可证表达式 |
| 供应链关系 | 组件间依赖关系（DEPENDS_ON） |
| 校验和 | 每个组件的 SHA-256 |
| 供应商 | 组件来源（Organization / Person） |
| 漏洞关联 | 关联 CVE 列表（通过 VEX 或 dependency-track） |

### 9.4 SBOM 生成与发布

- SBOM 在 CI/CD 流水线中自动生成（构建时产物）；
- 与二进制一同发布（同目录 `*.sbom.spdx.json`）；
- 每次发布必须重新生成（不得复用旧 SBOM）；
- SBOM 文件本身以 AGPL v3 + Apache 2.0 双许可证发布。

---

## 10. 合规检查清单

实现者在声明 ARE L3 合规前，必须完成下列检查：

### 10.1 权限引擎合规

- [ ] 实现了 `are_permission_check` 四参数接口
- [ ] 默认拒绝（无匹配规则返回 `ARE_PERM_DENY`）
- [ ] 规则匹配优先级正确（DENY > ASK > ALLOW > 默认 DENY）
- [ ] 通配符 `*` 与 `?` 正确实现
- [ ] 裁决缓存实现且可配置 TTL
- [ ] `are_permission_clear_cache` 在规则变更后调用

### 10.2 沙箱合规

- [ ] 实现 5 级沙箱（none/mem/cpu/fs/net）
- [ ] `ARE_SANDBOX_NET` 隐含 mem+cpu+fs 约束
- [ ] `are_sandbox_invoke(sb=NULL)` fail-closed
- [ ] 配额超限返回 `ARE_ERR_SEC_QUOTA`
- [ ] 沙箱内权限规则按 DENY 优先匹配

### 10.3 错误码合规

- [ ] 使用 `commons/utils/error/include/error.h` 作为唯一权威源
- [ ] 无本地 `#define` 重定义错误码
- [ ] 无独立的 `sdk_error_t` / `daemon_error_t` 体系
- [ ] 错误响应中的 `code` 使用权威错误码值
- [ ] 新增错误码遵循 §5.4 流程

### 10.4 审计日志合规

- [ ] 所有 §6.4 列出的事件均记录审计日志
- [ ] 日志为单行 JSON 格式
- [ ] 必填字段（timestamp/trace_id/actor/action/resource/result）齐全
- [ ] 日志 append-only，不可篡改
- [ ] 保留期 ≥ 30 天

### 10.5 数据保护合规

- [ ] §7.1 列出的敏感字段均脱敏
- [ ] 同一密钥脱敏结果一致
- [ ] debug 构建也脱敏
- [ ] 提供 `are_redact_sensitive` 接口

### 10.6 许可证合规

- [ ] AGPL 组件源文件含 SPDX-License-Identifier
- [ ] 二进制分发附带 LICENSE 与 NOTICE
- [ ] 无许可证污染（AGPL 代码未链接至专有组件）
- [ ] SBOM 已生成并随发布附带

---

## 11. 第三方实现的安全审计要求

### 11.1 审计范围

第三方实现者声明 L3 合规时，必须提交下列安全审计材料：

| 审计项 | 内容 | 频率 |
|--------|------|------|
| 源码审计 | 第三方代码与标准接口的符合性 | 每个发布版本 |
| 渗透测试 | 对 daemon IPC Bus 的攻击测试 | 每个主版本 |
| 依赖扫描 | SBOM 中所有依赖的 CVE 扫描 | 每月 |
| 权限模型测试 | 权限规则的边界用例（提权、绕过） | 每次规则变更 |
| 沙箱逃逸测试 | 5 级沙箱的逃逸尝试 | 每个主版本 |
| 错误码一致性 | 全错误码与权威源一致性校验 | 每次 CI |

### 11.2 审计报告要求

审计报告必须包含：
- 审计范围与方法（黑盒/白盒/灰盒）；
- 发现的安全问题（按 CVSS 3.1 评分分级）；
- 修复建议与修复期限（Critical ≤ 7 天，High ≤ 30 天，Medium ≤ 90 天）；
- 审计员资质（CSP/CISSP/OSCP 等认证）；
- 审计时间窗口与覆盖代码行数。

### 11.3 已知漏洞披露

- 发现安全漏洞时必须通过 `security@spharx.io` 报告（90 天披露窗口）；
- 不得在修复前公开披露；
- 修复后必须在 CHANGELOG 中记录（CVE 编号、影响版本、修复版本）。

### 11.4 L3 一致性测试套件（CTS）

L3 CTS 位于 `are-standards/cts/L3/`，包含下列用例类别：

| 用例类别 | 用例数 | 关键测试 |
|---------|--------|----------|
| 权限裁决 | 24 | 默认拒绝、通配符、优先级、缓存失效 |
| 沙箱隔离 | 18 | 5 级隔离、配额超限、fail-closed |
| 错误码 | 12 | 权威源一致性、分段正确性、禁令检测 |
| 审计日志 | 15 | 必填字段、不可篡改、保留期 |
| 数据脱敏 | 20 | 6 类字段脱敏、一致性、debug 构建 |
| 保护环 | 10 | 跨环调用拒绝、提权禁止 |
| syscall 分类 | 8 | 5 类划分正确性 |

### 11.5 合规声明

通过全部 CTS 用例的第三方实现可在文档与产品中声明：

```
ARE L3 Compliant — Verified by <审计方> on <日期>, CTS version <版本>
```

未通过 CTS 的实现不得使用 "ARE Compliant" 标识。

---

## 12. 参考实现

| 标准接口 | 参考实现位置 |
|----------|-------------|
| 权限引擎 | `agentos/cupolas/include/cupolas.h`（`cupolas_check_permission` 等） |
| 沙箱 | `agentos/atoms/syscall/include/agentos_sandbox.h`（`agentos_sandbox_*`） |
| 系统调用号 | `agentos/atoms/syscall/include/syscalls.h`（`agentos_syscall_num_t`） |
| 错误码权威源 | `agentos/commons/utils/error/include/error.h` |
| 类型定义权威源 | `agentos/commons/include/agentos_types.h`（`agentos_error_t`） |
| 动态策略引擎 | `agentos/cupolas/include/dynamic_policy_engine.h` |
| 零信任集成 | `agentos/cupolas/include/zero_trust_integration.h` |
| SafetyGuard | `agentos/cupolas/include/safety_guard.h` |
| 日志净化 | `agentos/daemon/common/include/log_sanitizer.h` |
| 输入校验 | `agentos/daemon/common/include/input_validator.h` |

### 12.1 适配层桥接示例

```c
/* are_l3_adapter.c — 桥接 Airymax 原生 API 至 ARE L3 标准 ABI */
#include "are_l3.h"
#include "cupolas.h"

are_perm_decision_t are_permission_check(const char *subject,
                                         const char *action,
                                         const char *resource,
                                         const char *context_json) {
    int rc = cupolas_check_permission(subject, action, resource, context_json);
    if (rc > 0)  return ARE_PERM_ALLOW;
    if (rc == 0) return ARE_PERM_DENY;
    if (rc == -1) return ARE_PERM_ASK;
    return ARE_PERM_DENY;  /* 错误时默认拒绝 */
}

are_error_t are_permission_add_rule(const char *subject, const char *action,
                                    const char *resource,
                                    are_perm_decision_t decision, int priority) {
    int allow = (decision == ARE_PERM_ALLOW) ? 1 : 0;
    return (are_error_t)cupolas_add_permission_rule(subject, action, resource,
                                                    allow, priority);
}
```

---

## 13. 参考文档

- [ARE Standards 总览](README.md)
- [L1 核心运行时接口](L1_runtime_interface.md) — IPC 原语、内存、任务
- [L2 服务通信协议](L2_service_protocol.md) — IPC Bus 消息头、JSON-RPC 命名空间
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [SPDX Specification 2.3](https://spdx.dev/specifications/)
- [CycloneDX 1.5](https://cyclonedx.org/docs/1.5/)
- [NIST SP 800-53](https://csrc.nist.gov/projects/cprt/catalog/cp/80053) — 安全控制基线

---

<div align="center">

**© 2026 SPHARX Ltd.** | 本文档及参考实现均以 AGPL v3 + Apache 2.0 双许可证发布（SPDX: AGPL-3.0-or-later OR Apache-2.0）。

</div>
