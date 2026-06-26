# SUBSYSTEM_corekern — 微核心核心

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/atoms/corekern/`  
**负责人**: Team A (核心引擎)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

corekern 是 Airymax 的微核心核心，提供最基础的系统运行时能力。它是所有其他子系统的基础依赖，遵循微核心最小化原则（Liedtke 原则）。

### 1.1 核心职责

| 职责 | 说明 |
|------|------|
| **内存管理** (`mem.h`) | 内核级内存分配/释放、内存池管理、对齐分配、泄漏检测 |
| **任务调度** (`task.h`) | 基于系统原生线程的任务调度、优先级管理、依赖解析、循环检测 |
| **进程间通信** (`ipc.h`) | 内核级 IPC 通道创建/连接/收发、同步调用（RPC 模式）、零拷贝消息传递 |
| **时间管理** (`agentos_time.h`) | 系统时间获取、定时器管理 |
| **错误处理** (`error.h`) | 统一错误码体系（`agentos_error_t`） |
| **可观测性** (`observability.h`) | 指标收集、追踪基础设施 |
| **OOM 处理** (`oom_handler.h`) | 内存不足时的优雅降级与恢复 |
| **引用计数** (`refcount.h`) | 原子引用计数，用于资源生命周期管理 |
| **线程缓存** (`tcache.h`) | 每线程缓存，减少锁竞争 |
| **内存竞技场** (`arena.h`) | 区域分配器，批量分配/释放 |

### 1.2 不在范围内的职责

- 不负责 LLM 调用、工具执行、Agent 编排等业务逻辑
- 不负责安全审计、权限控制（由 cupolas 负责）
- 不负责持久化存储（由 heapstore 负责）
- 不负责网络协议处理（由 gateway 负责）

### 1.3 所有权模型

- corekern 通过 `agentos_core_init()` / `agentos_core_shutdown()` 管理全局初始化/清理
- 所有分配的内存由调用者显式释放，或通过 corekern 的泄漏检测机制追踪
- IPC 通道、任务调度器、内存池等资源遵循"创建者负责销毁"原则

---

## 2. 输入/输出接口

### 2.1 统一入口

```c
// agentos.h — 微核心统一入口
int agentos_core_init(void);       // 初始化核心（幂等）
void agentos_core_shutdown(void);  // 关闭核心并清理资源
```

**API 版本**: `1.0.0` (MAJOR.MINOR.PATCH)

### 2.2 内存管理接口 (`mem.h`)

| 函数 | 说明 |
|------|------|
| `agentos_mem_init(heap_size)` | 初始化内存系统 |
| `agentos_mem_alloc(size)` | 分配内存（自动清零） |
| `agentos_mem_alloc_ex(size, file, line)` | 分配内存（带调试信息） |
| `agentos_mem_aligned_alloc(size, alignment)` | 对齐分配 |
| `agentos_mem_free(ptr)` | 释放内存（空指针安全） |
| `agentos_mem_realloc(ptr, new_size)` | 重新分配内存 |
| `agentos_mem_pool_create(block_size, block_count)` | 创建内存池 |
| `agentos_mem_pool_alloc(pool)` | 从池中分配 |
| `agentos_mem_pool_free(pool, ptr)` | 释放回池中 |
| `agentos_mem_pool_destroy(pool)` | 销毁内存池 |
| `agentos_mem_stats(out_total, out_used, out_peak)` | 获取内存统计 |
| `agentos_mem_check_leaks()` | 检查内存泄漏 |
| `agentos_mem_cleanup()` | 清理内存系统 |

### 2.3 任务调度接口 (`task.h`)

| 函数 | 说明 |
|------|------|
| `agentos_task_init()` | 初始化任务调度系统 |
| `agentos_task_self()` | 获取当前任务 ID |
| `agentos_task_sleep(ms)` | 任务休眠 |
| `agentos_task_yield()` | 让出 CPU 时间片 |
| `agentos_task_set_priority(tid, priority)` | 设置优先级 (0-100) |
| `agentos_task_get_priority(tid, out_priority)` | 获取优先级 |
| `agentos_task_get_state(tid, out_state)` | 获取任务状态 |
| `agentos_scheduler_resolve_dependencies(...)` | 拓扑排序 + 循环检测 |
| `agentos_scheduler_priority_inherit(...)` | 优先级继承 |
| `agentos_scheduler_resource_reserve(...)` | 资源预留检查 |
| `agentos_task_cleanup()` | 清理任务调度系统 |

**优先级常量**: `MIN(0)`, `LOW(25)`, `NORMAL(50)`, `HIGH(75)`, `MAX(100)`

### 2.4 IPC 接口 (`ipc.h`)

| 函数 | 说明 |
|------|------|
| `agentos_ipc_init()` | 初始化 IPC 子系统 |
| `agentos_ipc_create_channel(name, cb, userdata, out)` | 创建 IPC 通道 |
| `agentos_ipc_connect(name, out)` | 连接已有通道 |
| `agentos_ipc_close(channel)` | 关闭通道 |
| `agentos_ipc_send(channel, msg)` | 发送消息 |
| `agentos_ipc_recv(channel, timeout_ms, out)` | 接收消息 |
| `agentos_ipc_call(channel, msg, resp, size, timeout)` | 同步 RPC 调用 |
| `agentos_ipc_reply(channel, msg)` | 回复消息 |
| `agentos_ipc_get_fd(channel)` | 获取通道文件描述符 |
| `agentos_ipc_cleanup()` | 清理 IPC 子系统 |

**消息结构**: `agentos_kernel_ipc_message_t` (40 字节，轻量级)

---

## 3. 依赖关系

### 3.1 外部依赖

| 依赖 | 类型 | 说明 |
|------|------|------|
| POSIX 线程 (pthread) | 系统库 | 线程创建、同步原语 |
| 标准 C 库 | 系统库 | 内存分配、时间函数 |
| `commons/platform/` | 内部 | 平台抽象层 (`platform.h`) |
| `commons/include/` | 内部 | 基础类型定义 (`agentos_types.h`) |

### 3.2 被依赖方（谁依赖 corekern）

corekern 是整个系统的**基础层**，几乎所有其他子系统都依赖它：

- **coreloopthree**: 依赖任务调度、内存管理、IPC
- **cupolas**: 依赖内存管理、错误码
- **gateway_d / llm_d / tool_d / market_d**: 依赖 IPC、内存管理、错误码
- **orchestrator**: 依赖任务调度、内存管理
- **checkpoint**: 依赖内存管理、错误码
- **manager** (config_manager / alert_manager): 依赖内存管理、错误码

### 3.3 依赖图

```
                    ┌─────────────────┐
                    │    corekern     │  (微核心基础层)
                    └────────┬────────┘
           ┌─────────┬──────┼──────┬─────────┐
           ▼         ▼      ▼      ▼         ▼
      coreloopthree cupolas daemon gateway heapstore
```

---

## 4. 配置参数

### 4.1 编译时配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `AGENTOS_CORE_API_VERSION_MAJOR` | 1 | API 主版本号 |
| `AGENTOS_CORE_API_VERSION_MINOR` | 0 | API 次版本号 |
| `AGENTOS_CORE_API_VERSION_PATCH` | 0 | API 修订版本号 |
| `AGENTOS_TASK_PRIORITY_MIN` | 0 | 最低任务优先级 |
| `AGENTOS_TASK_PRIORITY_MAX` | 100 | 最高任务优先级 |

### 4.2 运行时配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `heap_size` | 0 (系统默认) | 堆大小限制（`agentos_mem_init` 参数） |
| `thread name` | 由调用者指定 | 线程名称（`agentos_thread_attr_t.name`） |
| `thread priority` | NORMAL(50) | 线程优先级 |
| `thread stack_size` | 0 (系统默认) | 线程栈大小 |

### 4.3 环境变量

| 变量 | 说明 |
|------|------|
| `AGENTOS_TEST_MODE` | 测试模式标志（影响内存追踪行为） |
| `AGENTOS_LOG_LEVEL` | 日志级别（DEBUG/INFO/WARN/ERROR） |

---

## 5. 线程安全与并发

| 类别 | 线程安全 | 说明 |
|------|----------|------|
| `agentos_core_init` | 否 | 不可多线程同时调用 |
| `agentos_mem_alloc` / `agentos_mem_free` | 是 | 可并发调用 |
| `agentos_mem_pool_alloc` / `agentos_mem_pool_free` | 是 | 池操作线程安全 |
| `agentos_task_*` | 部分 | 查询接口线程安全，调度接口不含内部锁 |
| `agentos_ipc_send` / `agentos_ipc_recv` | 否 | 单个通道单线程操作 |
| `agentos_ipc_call` / `agentos_ipc_reply` | 是 | 支持多线程同步调用 |

---

## 6. 错误码

corekern 定义系统级错误码（`error.h`），所有子系统共享：

- `AGENTOS_SUCCESS` (0) — 成功
- `AGENTOS_ERR_UNKNOWN` — 未知错误
- `AGENTOS_ERR_INVALID_PARAM` — 无效参数
- `AGENTOS_ERR_OUT_OF_MEMORY` — 内存不足
- `AGENTOS_ERR_NOT_FOUND` — 未找到
- `AGENTOS_ERR_PERMISSION_DENIED` — 权限拒绝
- `AGENTOS_ERR_TIMEOUT` — 超时
- `AGENTOS_ERR_STATE_ERROR` — 状态错误
- `AGENTOS_ERR_OVERFLOW` — 溢出
- `AGENTOS_ERR_NOT_SUPPORTED` — 不支持
- `AGENTOS_ERR_IO` — I/O 错误
- `AGENTOS_ECYCLE` — 循环依赖

---

## 7. 相关文档

- [微核心架构](../microkernel.md)
- [IPC 通信机制](../ipc.md)
- [系统调用架构](../syscall.md)