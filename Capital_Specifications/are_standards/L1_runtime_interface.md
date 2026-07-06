# ARE Standards L1 — 核心运行时接口标准

> **版本**: v0.1.0-draft | **状态**: 草案（v0.1.1 发行前冻结）| **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）
> **标准层级**: L1 | **回滚至**: [ARE Standards 总览](README.md)

---

## 1. 标准范围与目标

### 1.1 范围

本标准规定 Agent Runtime Environment（ARE）的**微核心（microkernel）原语接口**、**运行时生命周期**、**内存模型**以及 **ops 注入机制**。所有接口以 C ABI 形式描述，可直接映射到 C/C++/Rust/Go/Zig 等系统语言。

本标准**不**规定：
- L2 层 IPC Bus 消息头格式（见 [L2_service_protocol.md](L2_service_protocol.md)）；
- L3 层权限裁决、沙箱、错误码体系（见 [L3_security_governance.md](L3_security_governance.md)）；
- 具体调度算法实现（轮询/加权/优先级/ML 等为策略，本标准仅约束机制）。

### 1.2 目标

| 目标 | 描述 |
|------|------|
| **可移植性** | 第三方可在任意 POSIX/Windows 平台上实现本接口集，无源码改动接入 ARE 生态 |
| **机制与策略分离** | 接口仅规定"机制"（如何申请内存、如何 yield），不规定"策略"（分配器实现、调度算法） |
| **零反向依赖** | atoms 层（微核心）不得 `#include` daemon 层头文件；daemon 通过 ops 注入表反向回调 atoms |
| **ABI 稳定性** | 同一 MAJOR 版本内 ABI 兼容；破坏性变更必须递增 MAJOR 并提供迁移说明 |
| **可测试性** | 提供一致性测试套件（CTS），实现者必须通过全部测试方可声明合规 |

### 1.3 关键术语

| 术语 | 定义 |
|------|------|
| **atoms** | 微核心原语集合（IPC/内存/任务/时间/同步），对应 Airymax 的 `agentos/atoms/` 目录 |
| **daemon** | 用户态守护进程，通过 IPC Bus 协作，对应 `agentos/daemon/` |
| **ops 注入** | daemon 在 init 阶段向 atoms 注册回调表（checkpoint/hook/orch），实现反向控制流 |
| **SHM** | 共享内存段（Shared Memory），用于跨进程零拷贝数据交换 |
| **heapstore** | 堆存储抽象，提供持久化数据分区（registry/traces/logs/services） |

---

## 2. 微核心原语接口定义

所有接口以 `are_` 前缀命名，避免与宿主系统符号冲突。参考实现位于 Airymax atoms 层（corekern/syscall/memory/taskflow），其原生符号以 `agentos_` 前缀提供，由适配层桥接至 `are_` 标准 ABI。

### 2.1 错误码基线

所有 L1 接口返回 `are_error_t`（`int32_t`）。成功为 `ARE_OK (0)`，错误为负值。错误码权威定义与分段见 L3 标准《第 5 节 统一错误码体系》；本标准仅引用下列基础码：

| 宏 | 值 | 含义 |
|----|----|------|
| `ARE_OK` | 0 | 成功 |
| `ARE_EINVAL` | -1 | 参数无效 |
| `ARE_ENOMEM` | -2 | 内存不足 |
| `ARE_EBUSY` | -3 | 资源忙碌 |
| `ARE_ENOENT` | -4 | 资源不存在 |
| `ARE_EPERM` | -5 | 权限不足 |
| `ARE_ETIMEDOUT` | -6 | 操作超时 |
| `ARE_ENOTINIT` | -9 | 子系统未初始化 |
| `ARE_ENOTSUP` | -11 | 操作不支持 |
| `ARE_EUNKNOWN` | -99 | 未知错误 |

```c
#include <stdint.h>
typedef int32_t are_error_t;
```

### 2.2 IPC 接口

#### 2.2.1 设计原则

L1 IPC 是**内核级轻量级**接口，仅提供"创建通道—发送—接收—关闭"四元原语。完整的应用层 IPC 消息头（含 magic/version/trace_id）属 L2 范畴，L1 不感知。参考实现：`atoms/corekern/include/ipc.h`。

#### 2.2.2 核心结构

```c
/* 不透明句柄，实现者内部定义 */
typedef struct are_ipc_channel are_ipc_channel_t;

/* 内核级 IPC 消息：零拷贝、40 字节，避免引入 commons 复杂类型 */
typedef struct {
    uint32_t    code;     /* 消息类型码（0x01=数据, 0x02=控制, 0x03=响应） */
    const void *data;     /* 负载指针（零拷贝，调用者持有所有权） */
    size_t      size;     /* 负载字节数 */
    int32_t     fd;       /* 文件描述符（用于传递 socket/SHM 句柄），<0 表示无 */
    uint64_t    msg_id;   /* 请求-响应匹配 ID */
} are_ipc_message_t;     /* 共 40 字节 */
```

#### 2.2.3 接口签名

```c
/* 初始化 IPC 子系统，进程级、幂等。重复调用必须安全返回 ARE_OK。
 * @threadsafe 否  @reentrant 否 */
are_error_t are_ipc_init(void);

/* 发送一条消息到 channel。msg.data 生命周期由调用者管理，实现者
 * 不得在 send 返回后继续持有 data 指针。
 * @threadsafe 否  @reentrant 否 */
are_error_t are_ipc_send(are_ipc_channel_t *channel, const are_ipc_message_t *msg);

/* 阻塞接收一条消息，timeout_ms=0 表示非阻塞，UINT32_MAX 表示永久等待。
 * out_msg 由调用者分配；其 data 字段在零拷贝实现下指向通道内部缓冲，
 * 调用者必须在下一次 recv 前消费完毕。
 * @threadsafe 否  @reentrant 否 */
are_error_t are_ipc_recv(are_ipc_channel_t *channel, uint32_t timeout_ms,
                         are_ipc_message_t *out_msg);

/* 关闭通道并释放内部资源。channel 之后不可再用。
 * @threadsafe 否  @reentrant 否 */
are_error_t are_ipc_close(are_ipc_channel_t *channel);

/* 创建命名通道（服务端）。name 需在进程间可见（如 Unix socket 路径）。 */
are_error_t are_ipc_create_channel(const char *name, are_ipc_channel_t **out);

/* 连接到已存在的命名通道（客户端）。 */
are_error_t are_ipc_connect(const char *name, are_ipc_channel_t **out);

/* 同步 RPC：发送 msg 并阻塞等待响应（timeout_ms）。
 * response_size 输入时为缓冲区容量，输出时为实际响应字节数。
 * @threadsafe 是  @reentrant 否 */
are_error_t are_ipc_call(are_ipc_channel_t *channel, const are_ipc_message_t *msg,
                         void *response, size_t *response_size, uint32_t timeout_ms);

/* 回复消息：唤醒等待 msg_id 的 are_ipc_call 调用者。
 * @threadsafe 是  @reentrant 否 */
are_error_t are_ipc_reply(are_ipc_channel_t *channel, const are_ipc_message_t *msg);
```

#### 2.2.4 一致性要求

- `are_ipc_init` 必须可被重复调用而不泄漏资源；
- `are_ipc_send` 在 `msg->size > ARE_IPC_MAX_MSG_SIZE`（默认 512 KiB）时返回 `ARE_EMSGSIZE`；
- `are_ipc_close(NULL)` 必须安全无操作（NULL-safe）；
- 单一通道上并发 `send` 与 `recv` 的线程安全性**非强制**，但实现者必须在文档中声明。

### 2.3 内存管理接口

#### 2.3.1 基础分配

```c
/* 初始化内存子系统。heap_size=0 表示使用平台默认值。
 * @threadsafe 否  @reentrant 否 */
are_error_t are_mem_init(size_t heap_size);

/* 分配 size 字节并清零。失败返回 NULL。
 * @threadsafe 是  @reentrant 是 */
void *are_mem_alloc(size_t size);

/* 重分配。ptr=NULL 等价于 alloc；new_size=0 等价于 free 并返回 NULL。
 * @threadsafe 是  @reentrant 否 */
void *are_mem_realloc(void *ptr, size_t new_size);

/* 释放。ptr=NULL 必须安全无操作。只能释放由本族函数分配的指针。
 * @threadsafe 是  @reentrant 是 */
void  are_mem_free(void *ptr);

/* 对齐分配。alignment 必须是 2 的幂。 */
void *are_mem_aligned_alloc(size_t size, size_t alignment);
void  are_mem_aligned_free(void *ptr);

/* 内存统计：返回 total/used/peak 字节数。NULL 输出参数被忽略。 */
void  are_mem_stats(size_t *out_total, size_t *out_used, size_t *out_peak);

/* 清理内存子系统，释放所有内部资源。调用后除 init 外所有 API 不可用。 */
void  are_mem_cleanup(void);
```

#### 2.3.2 五级内存压力框架

实现者必须提供**五级压力监控**机制，在内存紧张时触发已注册的回调。该机制对应 Airymax 的 SEC-12 OOM 分级响应框架（参考实现：`atoms/corekern/include/oom_handler.h`）。

| 级别 | 宏 | 使用率 | 默认动作 |
|------|----|--------|----------|
| 0 | `ARE_MEM_PRESSURE_NORMAL` | < 70% | 允许所有分配 |
| 1 | `ARE_MEM_PRESSURE_WARNING` | 70–80% | 记录警告，启动非必要清理 |
| 2 | `ARE_MEM_PRESSURE_DEGRADED` | 80–90% | 主动降级（缩减缓存、降低日志级别） |
| 3 | `ARE_MEM_PRESSURE_CRITICAL` | 90–95% | 拒绝非必要分配，仅完成现有请求 |
| 4 | `ARE_MEM_PRESSURE_FATAL` | > 95% | 拒绝所有分配，立即终止进程 |

```c
typedef enum {
    ARE_MEM_PRESSURE_NORMAL   = 0,
    ARE_MEM_PRESSURE_WARNING  = 1,
    ARE_MEM_PRESSURE_DEGRADED = 2,
    ARE_MEM_PRESSURE_CRITICAL = 3,
    ARE_MEM_PRESSURE_FATAL    = 4,
} are_mem_pressure_level_t;

/* 压力变化回调：当级别达到或超过注册级别时触发。
 * level      — 触发时的当前压力级别
 * user_data  — 注册时透传的上下文 */
typedef void (*are_pressure_callback_t)(are_mem_pressure_level_t level,
                                        void *user_data);

/* 注册回调。每级最多支持 8 个回调，超出返回 ARE_EBUSY。
 * @return ARE_OK / ARE_EINVAL / ARE_EBUSY */
are_error_t are_mem_pressure_register(are_mem_pressure_level_t level,
                                      are_pressure_callback_t cb,
                                      void *user_data);

/* 注销回调。 */
are_error_t are_mem_pressure_unregister(are_mem_pressure_level_t level,
                                        are_pressure_callback_t cb);

/* 查询当前压力级别（线程安全）。 */
are_mem_pressure_level_t are_mem_get_pressure(void);

/* 检查分配是否应被允许（基于当前压力级别）。
 * @return ARE_OK 允许 / 非 0 拒绝（CRITICAL/FATAL） */
are_error_t are_mem_check_allocation(size_t requested_size);
```

**阈值可配置**，默认值如下，实现者必须支持通过配置结构覆盖：

```c
typedef struct {
    double warning_threshold;      /* 默认 0.70 */
    double degraded_threshold;     /* 默认 0.80 */
    double critical_threshold;     /* 默认 0.90 */
    double fatal_threshold;        /* 默认 0.95 */
    uint32_t check_interval_ms;    /* 默认 1000 */
    uint32_t recovery_cooldown_ms; /* 默认 5000 */
    bool     enable_auto_recovery; /* 默认 true */
    size_t   emergency_pool_size;  /* 默认 1 MiB，FATAL 时应急 */
} are_oom_config_t;
```

### 2.4 任务调度接口

L1 任务接口基于系统原生线程（pthreads/Windows threads），不引入用户态调度器。优先级、依赖解析、循环检测等增强能力属 taskflow 模块。

```c
/* 任务优先级常量 */
#define ARE_TASK_PRIORITY_MIN    0
#define ARE_TASK_PRIORITY_LOW   25
#define ARE_TASK_PRIORITY_NORMAL 50
#define ARE_TASK_PRIORITY_HIGH  75
#define ARE_TASK_PRIORITY_MAX  100

typedef enum {
    ARE_TASK_STATE_CREATED,
    ARE_TASK_STATE_READY,
    ARE_TASK_STATE_RUNNING,
    ARE_TASK_STATE_BLOCKED,
    ARE_TASK_STATE_TERMINATED,
} are_task_state_t;

typedef uint64_t are_task_id_t;  /* 唯一任务标识 */

typedef struct {
    const char *name;        /* 线程名（用于调试与观测） */
    int         priority;    /* 0–100 */
    size_t      stack_size;  /* 栈字节，0=平台默认 */
    int         detach_state;/* 0=joinable, 1=detached */
} are_thread_attr_t;

/* 初始化调度子系统 */
are_error_t are_task_init(void);

/* 创建并启动一个任务。entry 接收 user_data 并返回 void*（结果）。
 * attr=NULL 使用默认属性。out_id 输出任务 ID。
 * @return ARE_OK / ARE_ENOMEM / ARE_EINVAL */
are_error_t are_task_create(void *(*entry)(void *), void *user_data,
                            const are_thread_attr_t *attr,
                            are_task_id_t *out_id);

/* 让出当前 CPU 时间片。实现可退化为 sched_yield()。 */
void are_task_yield(void);

/* 等待任务结束并回收资源。detached 任务调用本函数返回 ARE_EINVAL。
 * out_result（可为 NULL）输出 entry 的返回值。 */
are_error_t are_task_join(are_task_id_t tid, void **out_result);

/* 当前任务休眠 ms 毫秒。 */
void are_task_sleep(uint32_t ms);

/* 查询/设置优先级与状态（线程安全）。 */
are_error_t are_task_get_priority(are_task_id_t tid, int *out);
are_error_t are_task_set_priority(are_task_id_t tid, int priority);
are_error_t are_task_get_state(are_task_id_t tid, are_task_state_t *out);

/* 返回当前任务 ID（线程安全、可重入）。 */
are_task_id_t are_task_self(void);

/* 清理调度子系统。 */
void are_task_cleanup(void);
```

#### 2.4.1 依赖解析增强（可选，taskflow）

下列接口为**可选增强**，实现者声明支持时必须通过全部相关测试：

```c
/* 拓扑排序 + 循环检测。返回 ARE_OK 或 ARE_ECYCLE。 */
are_error_t are_task_resolve_dependencies(const uint64_t *dep_from,
                                          const uint64_t *dep_to,
                                          size_t edge_count,
                                          are_dep_result_t *out);

/* 优先级继承：将高优先级阻塞任务的优先级传递给阻塞者。 */
are_error_t are_task_priority_inherit(are_task_id_t blocker, are_task_id_t blocked);

/* 资源预留检查：返回 ARE_OK 或 ARE_ERESOURCE。 */
are_error_t are_task_resource_reserve(size_t est_memory_kb, int est_cpu_cores);
```

### 2.5 时间接口

L1 时间接口严格区分两类时钟：

| 函数 | 时钟 | 用途 | 单调性 |
|------|------|------|--------|
| `are_time_ns` | `CLOCK_REALTIME` | 对齐网络时间、审计时间戳 | 否（受 NTP/手动调整影响） |
| `are_time_monotonic_ns` | `CLOCK_MONOTONIC` | 调度、超时、性能测量 | 是（必须） |

> Windows 平台对应 `GetSystemTimePreciseAsFileTime`（realtime）与 `QueryPerformanceCounter`（monotonic）。macOS 对应 `clock_gettime(CLOCK_REALTIME/...)`。

```c
/* 纳秒级实时时钟（CLOCK_REALTIME）。用于跨节点对齐与审计。
 * 受 NTP 调整影响，禁止用于调度/超时逻辑。 */
uint64_t are_time_ns(void);

/* 纳秒级单调时钟（CLOCK_MONOTONIC）。用于调度、超时、性能测量。
 * 必须单调递增，不受系统时间调整影响。 */
uint64_t are_time_monotonic_ns(void);

/* 毫秒便利函数。 */
uint64_t are_time_ms(void);
uint64_t are_time_monotonic_ms(void);

/* 高精度睡眠。ns<=0 立即返回。 */
are_error_t are_time_nanosleep(uint64_t ns);
are_error_t are_time_sleep_ms(uint32_t ms);

/* 定时器：one_shot=1 单次，=0 周期。 */
typedef struct are_timer are_timer_t;
typedef void (*are_timer_callback_t)(void *user_data);

are_timer_t *are_timer_create(are_timer_callback_t cb, void *user_data);
are_error_t  are_timer_start(are_timer_t *t, uint32_t interval_ms, int one_shot);
are_error_t  are_timer_stop(are_timer_t *t);
void         are_timer_destroy(are_timer_t *t);
```

**一致性要求**：`are_time_monotonic_ns` 在同一进程内**必须单调非递减**。CTS 包含"时钟回退检测"用例：在 1 秒内连续采样 1000 次，若任意后值小于前值则失败。

### 2.6 同步原语接口

下列类型与接口对应 Airymax `commons/platform/include/platform.h`。实现者必须提供：

```c
/* 互斥锁 */
typedef struct are_mutex are_mutex_t;        /* 实现内部封装 pthread_mutex_t / SRWLOCK */
are_error_t are_mutex_init(are_mutex_t *m);
are_error_t are_mutex_lock(are_mutex_t *m);
are_error_t are_mutex_trylock(are_mutex_t *m);  /* 锁定失败返回 ARE_EBUSY */
are_error_t are_mutex_unlock(are_mutex_t *m);
void        are_mutex_destroy(are_mutex_t *m);

/* 读写锁 */
typedef struct are_rwlock are_rwlock_t;      /* pthread_rwlock_t / SRWLOCK */
are_error_t are_rwlock_init(are_rwlock_t *l);
are_error_t are_rwlock_rdlock(are_rwlock_t *l);
are_error_t are_rwlock_wrlock(are_rwlock_t *l);
are_error_t are_rwlock_unlock(are_rwlock_t *l);
void        are_rwlock_destroy(are_rwlock_t *l);

/* 条件变量 */
typedef struct are_condvar are_condvar_t;    /* pthread_cond_t / CONDITION_VARIABLE */
are_error_t are_condvar_init(are_condvar_t *c);
are_error_t are_condvar_wait(are_condvar_t *c, are_mutex_t *m);
are_error_t are_condvar_timedwait(are_condvar_t *c, are_mutex_t *m, uint32_t timeout_ms);
are_error_t are_condvar_signal(are_condvar_t *c);    /* 唤醒一个等待者 */
are_error_t are_condvar_broadcast(are_condvar_t *c); /* 唤醒全部等待者 */
void        are_condvar_destroy(are_condvar_t *c);
```

**一致性要求**：
- `are_mutex_lock` 在已持有该锁的线程上再次调用必须返回 `ARE_EDEADLK`（递归锁实现除外，需显式声明）；
- `are_condvar_wait` 必须原子地释放 `m` 并阻塞，被唤醒时重新获取 `m`；
- 所有同步原语在 `destroy` 后再次使用的行为未定义，实现者应在 debug 构建中触发断言。

---

## 3. 生命周期接口

运行时生命周期由四个阶段组成，必须严格按序调用：

```
are_runtime_init  →  are_runtime_start  →  (运行期)  →  are_runtime_stop  →  are_runtime_finalize
                          ↑                                                ↓
                          └───────── (可重复 start/stop) ────────────────────┘
```

```c
typedef struct are_runtime_config {
    uint32_t    abi_version;        /* 见 § 7 版本兼容性 */
    size_t      heap_size;          /* 0=平台默认 */
    const char *shm_name;          /* 共享内存段名，NULL=不启用 SHM */
    size_t      shm_size;           /* SHM 段字节，0=不启用 */
    const char *data_root;          /* heapstore 数据根路径 */
    are_oom_config_t oom;          /* 内存压力配置，可填 0 使用默认 */
    uint32_t    flags;              /* 见下表 */
} are_runtime_config_t;

/* flags 位定义 */
#define ARE_RUNTIME_FLAG_ENABLE_OOM     0x01  /* 启用五级压力框架 */
#define ARE_RUNTIME_FLAG_ENABLE_SHM     0x02  /* 启用共享内存 */
#define ARE_RUNTIME_FLAG_DEBUG_ASSERTS  0x04  /* debug 构建断言 */
#define ARE_RUNTIME_FLAG_DISABLE_CACHE  0x08  /* 禁用 tcache/slab（用于测试） */

/* 初始化：分配资源、加载配置、调用各子模块 init。
 * cfg=NULL 使用全默认值。
 * @return ARE_OK / ARE_ENOMEM / ARE_EINVAL / ARE_EPLATFORM */
are_error_t are_runtime_init(const are_runtime_config_t *cfg);

/* 启动：开始调度、激活 IPC 监听、启动 OOM 监控线程。
 * 在 init 之后、finalize 之前可多次调用 stop/start。 */
are_error_t are_runtime_start(void);

/* 停止：停止接收新请求，等待在途请求完成（最多 drain_timeout_ms）。
 * 不释放资源，可再次 start。 */
are_error_t are_runtime_stop(uint32_t drain_timeout_ms);

/* 终结：释放全部资源，进程级。调用后除 init 外所有 API 不可用。 */
are_error_t are_runtime_finalize(void);

/* 查询运行时状态（线程安全）。 */
typedef enum {
    ARE_RUNTIME_STATE_UNINIT,
    ARE_RUNTIME_STATE_READY,    /* init 完成，未 start */
    ARE_RUNTIME_STATE_RUNNING,  /* 运行中 */
    ARE_RUNTIME_STATE_DRAINING, /* stop 中 */
    ARE_RUNTIME_STATE_STOPPED,  /* 已 stop */
    ARE_RUNTIME_STATE_FINALIZED,
} are_runtime_state_t;

are_runtime_state_t are_runtime_get_state(void);

/* 获取运行时统计：uptime/任务数/内存使用/IPC 消息计数等。 */
are_error_t are_runtime_get_stats(are_runtime_stats_t *out);
```

**一致性要求**：
- `are_runtime_init` 重复调用必须返回 `ARE_EALREADY`（非崩溃）；
- `are_runtime_start` 在 `RUNNING` 状态下返回 `ARE_EALREADY`；
- `are_runtime_finalize` 在 `RUNNING` 状态下必须先隐式 `stop(UINT32_MAX)` 再终结，但**强烈建议**调用方显式 stop；
- `are_runtime_finalize` 后再次 `init` 必须可工作（支持重启）。

---

## 4. 内存模型：SHM 与 heapstore 统一抽象

L1 抽象两类存储后端，由实现者选择具体存储引擎（参考实现采用 SQLite + memory-mapped files）。

### 4.1 共享内存（SHM）

SHM 用于跨进程零拷贝数据交换（如服务注册表、大块张量传输）。

```c
/* 创建或打开 SHM 段。flags: ARE_SHM_CREATE / ARE_SHM_RDONLY / ARE_SHM_EXCL */
typedef struct are_shm are_shm_t;

are_error_t are_shm_open(const char *name, size_t size, uint32_t flags, are_shm_t **out);
void       *are_shm_map(are_shm_t *s);            /* 返回基地址 */
size_t      are_shm_size(const are_shm_t *s);
are_error_t are_shm_unmap(are_shm_t *s);
are_error_t are_shm_close(are_shm_t *s);

/* 提示：将 [offset, offset+len) 区域标记为将要从/向磁盘读写，
 * 实现可退化为无操作。 */
are_error_t are_shm_advise(are_shm_t *s, size_t offset, size_t len, int advice);
```

### 4.2 heapstore（堆存储）

heapstore 是结构化持久化数据分区抽象，对应 Airymax `agentos/heapstore/`。L1 仅规定**最小接口集**，具体存储引擎（SQLite/LMDB/RocksDB）由实现者选择。

```c
typedef enum {
    ARE_HEAPSTORE_PATH_KERNEL,        /* 内核元数据 */
    ARE_HEAPSTORE_PATH_LOGS,          /* 日志 */
    ARE_HEAPSTORE_PATH_REGISTRY,      /* 服务注册表/配置 */
    ARE_HEAPSTORE_PATH_SERVICES,      /* 各 daemon 业务数据 */
    ARE_HEAPSTORE_PATH_TRACES,        /* 链路追踪 */
    ARE_HEAPSTORE_PATH_KERNEL_IPC,    /* IPC 落盘 */
    ARE_HEAPSTORE_PATH_KERNEL_MEMORY,/* 内存检查点 */
    ARE_HEAPSTORE_PATH_MAX,
} are_heapstore_path_t;

typedef struct are_heapstore are_heapstore_t;

are_error_t are_heapstore_init(const char *root_path, size_t max_log_size_mb,
                               int log_retention_days, are_heapstore_t **out);
are_error_t are_heapstore_path_get(are_heapstore_t *hs, are_heapstore_path_t type,
                                   char *buf, size_t buf_len);
are_error_t are_heapstore_cleanup(are_heapstore_t *hs);
```

### 4.3 内存模型一致性要求

| 要求 | 描述 |
|------|------|
| **统一所有权** | `are_mem_alloc` 返回的指针必须由 `are_mem_free` 释放，禁止 `free()` 或 `delete` |
| **跨子系统零泄漏** | `are_runtime_finalize` 后 `are_mem_check_leaks()` 必须返回 0 |
| **SHM 句柄独立** | SHM 映射区不参与 `are_mem_stats` 统计，避免与堆统计混淆 |
| **heapstore 路径隔离** | 不同 `are_heapstore_path_t` 必须落到不同子目录，禁止跨路径写文件 |

---

## 5. ops 注入机制

### 5.1 设计动机

atoms 层不得反向依赖 daemon 层（零反向依赖原则）。但运行时需要"反向控制流"：当 atoms 发生 checkpoint、OOM、任务完成等事件时，需通知 daemon 执行相应动作（持久化检查点、释放缓存、触发编排）。该矛盾通过 **ops 注入表**解决：daemon 在 `are_runtime_post_init()` 阶段将一组回调函数指针注入 atoms，atoms 在事件点反向调用。

### 5.2 三类 ops 抽象表

```c
/* ==================== Checkpoint ops ==================== */
/* 当 atoms 创建/恢复任务检查点时调用，由 sched_d 注册。 */
typedef struct are_checkpoint_ops {
    uint32_t struct_version;  /* = ARE_CHECKPOINT_OPS_VERSION */
    are_error_t (*save)(const char *task_id, const char *session_id,
                        uint64_t seq, const char *state_json,
                        void *user_data);
    are_error_t (*restore)(const char *task_id, uint64_t seq,
                          char **out_state_json, void *user_data);
    are_error_t (*delete)(const char *task_id, uint64_t seq, void *user_data);
    void       *user_data;
} are_checkpoint_ops_t;

/* ==================== Hook ops ==================== */
/* 任务生命周期钩子，由 hook_d 注册。在事件点同步调用。 */
typedef enum {
    ARE_HOOK_PRE_TASK    = 0,
    ARE_HOOK_POST_TASK   = 1,
    ARE_HOOK_PRE_TOOL    = 2,
    ARE_HOOK_POST_TOOL   = 3,
    ARE_HOOK_ON_ERROR    = 4,
    ARE_HOOK_ON_OOM      = 5,
    ARE_HOOK_TYPE_MAX,
} are_hook_type_t;

typedef are_error_t (*are_hook_fn)(are_hook_type_t type,
                                    const char *task_id,
                                    const char *event_json,
                                    void *user_data);

typedef struct are_hook_ops {
    uint32_t struct_version;
    are_hook_fn hooks[ARE_HOOK_TYPE_MAX];
    void       *user_data;
} are_hook_ops_t;

/* ==================== Orchestrator ops ==================== */
/* 编排回调，由 sched_d / agent_d 注册，用于跨任务流水线协调。 */
typedef enum {
    ARE_ORCH_PHASE_PLAN     = 0,
    ARE_ORCH_PHASE_DISPATCH = 1,
    ARE_ORCH_PHASE_EXECUTE  = 2,
    ARE_ORCH_PHASE_REDUCE   = 3,
    ARE_ORCH_PHASE_COMPENSATE = 4,
} are_orch_phase_t;

typedef are_error_t (*are_orch_fn)(are_orch_phase_t phase,
                                   const char *pipeline_id,
                                   const char *input_json,
                                   char **out_json,
                                   void *user_data);

typedef struct are_orch_ops {
    uint32_t struct_version;
    are_orch_fn execute;
    are_orch_fn compensate;   /* 可为 NULL */
    void       *user_data;
} are_orch_ops_t;
```

### 5.3 注入 API

```c
#define ARE_OPS_ABI_VERSION 1

/* 在 are_runtime_post_init() 钩子内调用，向 atoms 注册 ops。
 * 任何一项可为 NULL，表示该类回调不启用。
 * atoms 在注册时 deep-copy 函数指针与 user_data，调用方可在注册后释放栈帧。
 * @return ARE_OK / ARE_ENOTINIT / ARE_EINVAL */
are_error_t are_ops_set_checkpoint(const are_checkpoint_ops_t *ops);
are_error_t are_ops_set_hook(const are_hook_ops_t *ops);
are_error_t are_ops_set_orch(const are_orch_ops_t *ops);

/* 查询当前已注册的 ops（用于诊断）。返回静态只读指针，不可修改。 */
const are_checkpoint_ops_t *are_ops_get_checkpoint(void);
const are_hook_ops_t       *are_ops_get_hook(void);
const are_orch_ops_t       *are_ops_get_orch(void);
```

### 5.4 注入时机与失败语义

| 时机 | 行为 |
|------|------|
| `are_runtime_init` 之前调用 `are_ops_set_*` | 返回 `ARE_ENOTINIT` |
| `are_runtime_start` 之后调用 `are_ops_set_*` | 返回 `ARE_EBUSY`（运行期不可改） |
| atoms 触发事件但 ops 未注册 | atoms 跳过该回调，记录 `WARN` 级别日志，不视为错误 |
| ops 回调内部返回非 `ARE_OK` | atoms 记录错误并继续（不传播到调用链，避免反向依赖崩溃） |

---

## 6. 适配器注入点：`are_runtime_post_init()` 钩子

### 6.1 钩子签名

```c
/* 在 are_runtime_init 成功之后、are_runtime_start 之前调用。
 * 用于注入 ops、注册 daemon 服务、初始化第三方适配器。
 * 实现：daemon 侧通过 are_runtime_register_post_init 注册多个钩子，
 *       atoms 在 init 末尾按注册顺序依次调用。 */
typedef are_error_t (*are_post_init_fn)(void *user_data);

are_error_t are_runtime_register_post_init(are_post_init_fn fn, void *user_data);
```

### 6.2 典型注入流程

```
are_runtime_init(cfg)
   ├─ are_mem_init / are_task_init / are_ipc_init / are_time_init
   ├─ are_shm_open (若 cfg.flags & ARE_RUNTIME_FLAG_ENABLE_SHM)
   ├─ are_heapstore_init
   └─ 调用所有已注册的 are_post_init_fn  ← daemon 在此注入 ops
        ├─ sched_d.post_init → are_ops_set_checkpoint(...)
        ├─ hook_d.post_init  → are_ops_set_hook(...)
        └─ sched_d/agent_d   → are_ops_set_orch(...)
```

### 6.3 钩子失败处理

任一 `are_post_init_fn` 返回非 `ARE_OK` 时，`are_runtime_init` 立即返回该错误码并回滚已完成的子模块初始化（逆序调用 finalize）。这保证"全有或全无"语义：要么所有 daemon 适配器就绪，要么 runtime 干净失败。

---

## 7. 版本兼容性策略

### 7.1 双重版本号

ARE L1 采用**双重版本号**保证源码与二进制双重兼容：

| 版本 | 形式 | 含义 | 兼容性约束 |
|------|------|------|-----------|
| **Spec 版本** | `v0.1.0-draft` (semver) | 本文档版本 | MAJOR 变更允许接口签名破坏 |
| **ABI 版本** | `ARE_L1_ABI_VERSION` (单整数) | 二进制接口版本 | 同 ABI 版本内字节布局兼容 |

```c
/* 实现者必须在编译产物中导出以下符号供 CTS 检测 */
#define ARE_L1_SPEC_MAJOR 0
#define ARE_L1_SPEC_MINOR 1
#define ARE_L1_SPEC_PATCH 0
#define ARE_L1_ABI_VERSION 1
```

### 7.2 兼容性矩阵

| 变更类型 | Spec 版本 | ABI 版本 | 通知期 |
|---------|-----------|----------|--------|
| 新增接口（向后兼容） | MINOR+1 | 不变 | 提前 1 月 |
| 修改接口签名（破坏） | MAJOR+1 | +1 | 提前 6 月 + 迁移指南 |
| 删除接口（破坏） | MAJOR+1 | +1 | 提前 6 月 + 弃用期 ≥ 1 个 MAJOR |
| 调整结构体字段顺序（破坏） | MAJOR+1 | +1 | 提前 6 月 |
| 调整宏值（如错误码值） | MINOR+1 | 不变 | 提前 1 月（语义宏引用者无感知） |

### 7.3 运行期版本协商

```c
/* 查询实现者的 ABI 版本与 spec 版本。
 * CTS 在测试开始前调用此函数，若 ABI 不匹配则跳过该批用例。 */
typedef struct {
    uint32_t abi_version;
    uint32_t spec_major;
    uint32_t spec_minor;
    uint32_t spec_patch;
    const char *vendor;     /* 实现者标识，如 "airymax" / "third-party-x" */
    const char *build_id;   /* 构建标识（git commit 等） */
} are_version_info_t;

const are_version_info_t *are_version_get(void);
```

实现者返回的 `are_version_info_t` 必须是进程生命周期的静态存储，禁止返回栈地址。

---

## 8. 一致性测试要求

实现者必须通过下列测试用例清单（CTS，Conformance Test Suite）。CTS 参考实现以 Apache-2.0 发布，位于 `are-standards/cts/L1/`。

### 8.1 必测用例清单

| 用例 ID | 模块 | 测试要点 | 通过判据 |
|---------|------|----------|----------|
| L1-CTS-001 | IPC | 重复 `are_ipc_init` 后 `are_ipc_close(NULL)` | 不崩溃，返回 `ARE_OK` |
| L1-CTS-002 | IPC | `are_ipc_send` 超长消息（> 512 KiB） | 返回 `ARE_EMSGSIZE` |
| L1-CTS-003 | IPC | `are_ipc_call` 超时（timeout_ms=100，对端不回复） | 返回 `ARE_ETIMEDOUT` |
| L1-CTS-004 | IPC | `are_ipc_recv` 非阻塞（timeout_ms=0）无消息 | 返回 `ARE_EAGAIN` 或 `ARE_ETIMEDOUT` |
| L1-CTS-005 | Mem | `are_mem_alloc(0)` 行为定义 | 返回 NULL 或 1 字节均可，但不得崩溃 |
| L1-CTS-006 | Mem | `are_mem_free(NULL)` 安全无操作 | 不崩溃 |
| L1-CTS-007 | Mem | `are_mem_realloc(NULL, 1024)` 等价 alloc | 返回非 NULL |
| L1-CTS-008 | Mem | 五级压力回调触发 | 模拟 95% 使用率，FATAL 回调被调用 |
| L1-CTS-009 | Mem | `are_mem_check_allocation` 在 CRITICAL 拒绝非必要分配 | 返回非 0 |
| L1-CTS-010 | Task | `are_task_create` + `are_task_join` 往返 | entry 返回值正确回收 |
| L1-CTS-011 | Task | `are_task_yield` 不阻塞 | 立即返回 |
| L1-CTS-012 | Task | `are_task_join` 在 detached 任务上 | 返回 `ARE_EINVAL` |
| L1-CTS-013 | Time | `are_time_monotonic_ns` 1000 次采样单调非递减 | 通过 |
| L1-CTS-014 | Time | `are_time_nanosleep(100_000_000)` 实际睡眠 | 误差 < 50 ms |
| L1-CTS-015 | Sync | `are_mutex_lock` 同线程二次加锁 | 返回 `ARE_EDEADLK`（或递归锁声明） |
| L1-CTS-016 | Sync | `are_condvar_timedwait` 超时返回 | 100ms 内返回 `ARE_ETIMEDOUT` |
| L1-CTS-017 | Lifecycle | `are_runtime_init` → start → stop → finalize 全流程 | `are_runtime_get_state` 序列正确 |
| L1-CTS-018 | Lifecycle | `are_runtime_init` 重复调用 | 返回 `ARE_EALREADY` |
| L1-CTS-019 | Lifecycle | `are_runtime_finalize` 后 `are_mem_check_leaks` | 返回 0（零泄漏） |
| L1-CTS-020 | ops | 注入 checkpoint ops 后触发 save | save 回调被调用一次 |
| L1-CTS-021 | ops | 未注入 hook ops 时触发 OOM 事件 | 不崩溃，记录 WARN 日志 |
| L1-CTS-022 | ops | `are_ops_set_*` 在 `start` 之后调用 | 返回 `ARE_EBUSY` |
| L1-CTS-023 | SHM | `are_shm_open` + `are_shm_map` + `are_shm_close` | 子进程可见同一 region |
| L1-CTS-024 | heapstore | `are_heapstore_path_get` 不同 type 路径隔离 | 路径前缀不同 |
| L1-CTS-025 | Version | `are_version_get` 返回静态存储 | 指针稳定，多次调用结果一致 |

### 8.2 性能基线（参考性，非阻塞）

| 操作 | 目标延迟 | 备注 |
|------|----------|------|
| `are_ipc_send`（同机，1 KiB 负载） | < 100 μs | Unix socket |
| `are_mem_alloc/free`（4 KiB） | < 1 μs | slab/tcache |
| `are_mutex_lock/unlock`（无竞争） | < 100 ns | |
| `are_time_monotonic_ns` | < 50 ns | |

---

## 9. 参考实现

本标准的参考实现由 Airymax 主仓库提供（Apache-2.0），关键文件位置如下：

| 标准接口 | 参考实现位置 |
|----------|-------------|
| `are_ipc_*` | `agentos/atoms/corekern/include/ipc.h`（`agentos_ipc_*`） |
| `are_mem_alloc/free/realloc` | `agentos/atoms/corekern/include/mem.h`（`agentos_mem_*`） |
| 五级压力框架 | `agentos/atoms/corekern/include/oom_handler.h`（`agentos_oom_*`） |
| `are_task_create/yield/join` | `agentos/atoms/corekern/include/task.h`（`agentos_task_*`） |
| `are_time_ns/monotonic_ns` | `agentos/atoms/corekern/include/agentos_time.h`（`agentos_time_*`） |
| 同步原语 | `agentos/commons/platform/include/platform.h`（`agentos_mutex_*` 等） |
| `are_heapstore_*` | `agentos/heapstore/include/heapstore.h` |
| 任务依赖解析 | `agentos/atoms/taskflow/include/taskflow.h` |
| checkpoint 参考 | `agentos/daemon/common/include/checkpoint.h` |

### 9.1 适配层桥接示例

参考实现提供 `are_l1_adapter.c` 将 `agentos_*` 符号桥接至 `are_*` 标准 ABI：

```c
/* are_l1_adapter.c — 桥接 Airymax 原生 API 至 ARE L1 标准 */
#include "are_l1.h"
#include "ipc.h"
#include "mem.h"
#include "task.h"
#include "agentos_time.h"

are_error_t are_ipc_init(void) {
    return (are_error_t)agentos_ipc_init();
}

void *are_mem_alloc(size_t size) {
    return agentos_mem_alloc(size);
}

uint64_t are_time_monotonic_ns(void) {
    return agentos_time_monotonic_ns();
}
/* ... 其余接口一一映射 ... */
```

第三方实现者若已自有等价微核心，只需实现上述 `are_*` 接口集合并通过 CTS 即可声明合规，无需复用 Airymax 代码。

---

## 10. 参考文档

- [ARE Standards 总览](README.md)
- [L2 服务通信协议](L2_service_protocol.md) — IPC Bus 消息头、JSON-RPC 命名空间
- [L3 安全与治理](L3_security_governance.md) — 错误码权威定义、沙箱、审计

---

<div align="center">

**© 2026 SPHARX Ltd.** | 本文档及参考实现均以 AGPL v3 + Apache 2.0 双许可证发布（SPDX: AGPL-3.0-or-later OR Apache-2.0）。

</div>
