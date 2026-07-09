Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 资源管理表

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/project_erp/resource_management_table.md

## 1. 概述

本文档提供了 Airymax 中所有需要管理的资源及其释放责任的详细说明。资源管理是确保系统稳定性和防止资源泄漏的关键部分，所有开发者必须严格遵循本文档中的资源管理规范。本规范基于 Airymax 架构设计原则 V1.7，特别是 **E-3 资源确定性原则** 和 **E-8 可测试性原则**，确保资源生命周期明确且可测试验证。

## 2. 资源管理原则

- **所有权明确**：每个资源必须有明确的所有权归属
- **释放责任**：资源的创建者或接收者必须负责释放资源
- **生命周期管理**：资源的创建、使用和释放必须有清晰的生命周期管理
- **错误处理**：在资源获取失败时，必须正确处理已分配的资源
- **线程安全**：资源管理操作必须考虑线程安全性

## 3. 资源管理表

### 3.1 核心循环模块 (coreloopthree)

| 资源类型 | 创建函数 | 释放函数 | 释放责任 | 线程安全 | 备注 |
|---------|---------|---------|---------|---------|------|
| `agentos_core_loop_t` | `agentos_loop_create()` | `agentos_loop_destroy()` | 创建者释放 | 否（单线程访问） | 核心循环实例 |
| `agentos_cognition_engine_t` | `agentos_cognition_create()` | `agentos_cognition_destroy()` | 创建者释放 | 否（单线程访问） | 认知引擎实例 |
| `agentos_execution_engine_t` | `agentos_execution_create()` | `agentos_execution_destroy()` | 创建者释放 | 否（单线程访问） | 执行引擎实例 |
| `agentos_memory_engine_t` | `agentos_memory_create()` | `agentos_memory_destroy()` | 创建者释放 | 否（单线程访问） | 记忆引擎实例 |
| `agentos_compensation_t` | `agentos_compensation_create()` | `agentos_compensation_destroy()` | 创建者释放 | 否（单线程访问） | 补偿事务管理器 |
| `char*` (任务ID) | `agentos_loop_submit()` | 调用者释放 | 调用者释放 | 内部互斥锁 | 使用 `MEMORY_FREE_SAFE()` 释放 |
| `char*` (JSON结果) | `agentos_loop_wait()` | 调用者释放 | 调用者释放 | 内部互斥锁 | 使用 `MEMORY_FREE_SAFE()` 释放 |
| `char*` (健康状态JSON) | `agentos_cognition_health_check()` | 调用者释放 | 调用者释放 | 内部互斥锁 | 使用 `MEMORY_FREE_SAFE()` 释放 |
| `char*` (健康状态JSON) | `agentos_execution_health_check()` | 调用者释放 | 调用者释放 | 内部互斥锁 | 使用 `MEMORY_FREE_SAFE()` 释放 |
| `char*` (健康状态JSON) | `agentos_memory_health_check()` | 调用者释放 | 调用者释放 | 内部互斥锁 | 使用 `MEMORY_FREE_SAFE()` 释放 |
| `agentos_memory_result_t` | `agentos_memory_query()` | `agentos_memory_result_free()` | 调用者释放 | 内部互斥锁 | 记忆查询结果 |
| `agentos_memory_record_t` | `agentos_memory_get()` | `agentos_memory_record_free()` | 调用者释放 | 内部互斥锁 | 记忆记录 |
| `char*` (记录ID) | `agentos_memory_write()` | 调用者释放 | 调用者释放 | 内部互斥锁 | 使用 `MEMORY_FREE_SAFE()` 释放 |
| `char**` (补偿动作) | `agentos_compensation_get_human_queue()` | 调用者释放 | 调用者释放 | 否（单线程访问） | 包括数组本身和数组中的每个元素 |

### 3.2 系统调用模块 (syscall)

| 资源类型 | 创建函数 | 释放函数 | 释放责任 | 线程安全 | 备注 |
|---------|---------|---------|---------|---------|------|
| `agentos_syscall_table_t` | `agentos_syscall_table_create()` | `agentos_syscall_table_destroy()` | 创建者释放 | 否（单线程访问） | 系统调用表 |
| `agentos_syscall_entry_t` | `agentos_syscall_entry_create()` | `agentos_syscall_entry_destroy()` | 创建者释放 | 否（单线程访问） | 系统调用条目 |

### 3.3 记忆管理模块 (memoryrovol)

| 资源类型 | 创建函数 | 释放函数 | 释放责任 | 线程安全 | 备注 |
|---------|---------|---------|---------|---------|------|
| `agentos_memoryrovol_t` | `agentos_memoryrovol_create()` | `agentos_memoryrovol_destroy()` | 创建者释放 | 否（单线程访问） | 记忆卷管理实例 |
| `agentos_memoryrovol_segment_t` | `agentos_memoryrovol_segment_create()` | `agentos_memoryrovol_segment_destroy()` | 创建者释放 | 否（单线程访问） | 记忆卷段 |

### 3.4 安全域模块 (cupolas)

| 资源类型 | 创建函数 | 释放函数 | 释放责任 | 线程安全 | 备注 |
|---------|---------|---------|---------|---------|------|
| `agentos_cupolas_t` | `agentos_cupolas_create()` | `agentos_cupolas_destroy()` | 创建者释放 | 内部互斥锁 | 安全穹顶实例 |
| `agentos_cupola_t` | `agentos_cupola_create()` | `agentos_cupola_destroy()` | 创建者释放 | 内部互斥锁 | 单个安全穹顶 |

### 3.5 动态模块 (gateway)

| 资源类型 | 创建函数 | 释放函数 | 释放责任 | 线程安全 | 备注 |
|---------|---------|---------|---------|---------|------|
| `agentos_dynamic_t` | `agentos_dynamic_create()` | `agentos_dynamic_destroy()` | 创建者释放 | 否（单线程访问） | 动态模块实例 |

### 3.6 用户态服务层 (daemon)

| 资源类型 | 创建函数 | 释放函数 | 释放责任 | 线程安全 | 备注 |
|---------|---------|---------|---------|---------|------|
| `agentos_daemon_service_t` | `agentos_daemon_service_create()` | `agentos_daemon_service_destroy()` | 创建者释放 | 内部互斥锁 | 用户态服务实例 |
| `agentos_ipc_bus_t` | `agentos_ipc_bus_create()` | `agentos_ipc_bus_destroy()` | 创建者释放 | 内部互斥锁 | IPC 服务总线 |
| `agentos_thread_pool_t` | `agentos_thread_pool_create()` | `agentos_thread_pool_destroy()` | 创建者释放 | 内部互斥锁 | 线程池 |
| `agentos_orchestrator_t` | `agentos_orchestrator_create()` | `agentos_orchestrator_destroy()` | 创建者释放 | 内部互斥锁 | 编排器实例 |
| `char*` (调度结果) | `agentos_sched_schedule()` | 调用者释放 | 调用者释放 | 原子操作 | 使用 `MEMORY_FREE_SAFE()` 释放 |

### 3.7 公共工具层 (commons)

| 资源类型 | 创建函数 | 释放函数 | 释放责任 | 线程安全 | 备注 |
|---------|---------|---------|---------|---------|------|
| `config_context_t` | `config_context_create()` | `config_context_destroy()` | 创建者释放 | 否（单线程访问） | 配置上下文 |
| `config_source_t` | `config_source_create_file()` | `config_source_destroy()` | 创建者释放 | 否（单线程访问） | 配置源 |
| `config_validator_t` | `config_validator_create()` | `config_validator_destroy()` | 创建者释放 | 否（单线程访问） | 配置验证器 |
| `agentos_logger_t` | `agentos_logger_create()` | `agentos_logger_destroy()` | 创建者释放 | 内部互斥锁 | 日志记录器 |
| `char*` (日志格式化输出) | `agentos_log_format()` | 调用者释放 | 调用者释放 | 否（单线程访问） | 使用 `MEMORY_FREE_SAFE()` 释放 |

## 4. 资源管理最佳实践

### 4.1 内存分配与释放

- 始终使用 `malloc()`/`calloc()`/`realloc()` 分配内存
- 始终使用 `MEMORY_FREE_SAFE(&ptr)` 释放内存（定义于 `memory_compat.h`），该宏释放内存后自动将指针置 NULL，防止悬垂指针和双重释放
- 对于不适用 `MEMORY_FREE_SAFE` 的场景（如非动态分配指针），使用 `free()` 后必须立即将指针置 NULL
- 在分配多个资源时，使用 `goto cleanup` 语句确保在出错时正确释放所有已分配的资源
- 避免内存泄漏，确保每个 `malloc()` 都有对应的 `MEMORY_FREE_SAFE()` 或 `free()`

### 4.2 资源生命周期管理

- 资源的创建和释放应该在同一逻辑层级
- 避免跨模块的资源所有权转移，除非有明确的文档说明
- 使用 goto cleanup 模式管理资源生命周期（C 语言不支持 RAII，通过 `goto cleanup` 统一跳转到函数末尾的清理代码段实现等效的资源确定性释放）
- 在资源不再需要时立即释放，避免资源长时间占用

### 4.3 错误处理中的资源管理

- 在错误处理路径中，确保释放所有已分配的资源
- 使用统一的错误处理模式，确保资源释放的一致性
- 记录资源释放失败的情况，便于调试和问题定位

### 4.4 线程安全的资源管理

- 对于共享资源，使用适当的同步机制（如互斥锁、条件变量等）
- 避免在持有锁的情况下进行可能阻塞的操作
- 确保资源的创建和释放操作是线程安全的
- 考虑使用无锁数据结构和原子操作提高性能

## 5. 资源管理示例

### 5.1 正确的资源管理示例（goto cleanup 模式）

```c
// 使用 goto cleanup 模式管理多个资源
agentos_core_loop_t* loop = NULL;
char* task_id = NULL;
char* result = NULL;

agentos_error_t err = agentos_loop_create(NULL, &loop);
if (err != AGENTOS_OK) {
    return err;
}

err = agentos_loop_submit(loop, "Hello World", 11, &task_id);
if (err != AGENTOS_OK) {
    goto cleanup_loop;
}

size_t result_len = 0;
err = agentos_loop_wait(loop, task_id, 10000, &result, &result_len);

// 释放任务ID（不再需要）
MEMORY_FREE_SAFE(&task_id);

if (err != AGENTOS_OK) {
    goto cleanup_loop;
}

// 使用结果
printf("Task result: %s\n", result);

// 释放结果
MEMORY_FREE_SAFE(&result);

// 释放核心循环
agentos_loop_destroy(loop);
return AGENTOS_OK;

cleanup_loop:
    MEMORY_FREE_SAFE(&task_id);
    MEMORY_FREE_SAFE(&result);
    agentos_loop_destroy(loop);
    return err;
```

### 5.2 错误处理中的资源管理（goto cleanup 模式）

```c
// 使用 goto cleanup 模式分配多个资源
void* resource1 = NULL;
void* resource2 = NULL;
agentos_error_t err = AGENTOS_OK;

resource1 = malloc(sizeof(*resource1));
if (!resource1) {
    err = AGENTOS_ENOMEM;
    goto cleanup;
}

resource2 = malloc(sizeof(*resource2));
if (!resource2) {
    err = AGENTOS_ENOMEM;
    goto cleanup;
}

// 使用资源
// ...

cleanup:
    MEMORY_FREE_SAFE(&resource1);
    MEMORY_FREE_SAFE(&resource2);
    return err;
```

## 6. 资源泄漏检测工具

### 6.1 Valgrind（Linux 开发/测试环境）

Valgrind 是检测内存泄漏和非法内存访问的首选工具，适用于开发与 CI 环境：

```bash
# 运行程序并检测内存泄漏
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes ./your_application

# 仅检测内存泄漏（快速模式）
valgrind --leak-check=summary ./your_application

# 抑制已知的第三方库误报
valgrind --suppressions=agentos.supp ./your_application
```

**关键输出解读**:
- **definitely lost**: 确认泄漏，必须修复
- **indirectly lost**: 间接泄漏，通常随 definitely lost 一起修复
- **possibly lost**: 可能泄漏，需人工确认
- **still reachable**: 程序退出时仍可达，通常可忽略（全局变量等）

### 6.2 AddressSanitizer（ASan，跨平台编译时检测）

AddressSanitizer 提供更快的检测速度，适合集成到日常开发和 CI 流水线：

```bash
# CMake 编译时启用 ASan
cmake -DCMAKE_C_FLAGS="-fsanitize=address -fno-omit-frame-pointer -g" \
      -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address" ..

# 运行程序
./your_application

# 检测特定问题
ASAN_OPTIONS=detect_leaks=1:detect_stack_use_after_return=1 ./your_application
```

**ASan 检测能力**:
- 堆缓冲区溢出 / 栈缓冲区溢出 / 全局缓冲区溢出
- 释放后使用（use-after-free）
- 双重释放（double-free）
- 内存泄漏

### 6.3 推荐使用策略

| 场景 | 推荐工具 | 原因 |
|------|---------|------|
| 日常开发编译 | ASan | 速度快，实时检测 |
| CI/CD 流水线 | ASan + Valgrind | ASan 快速检测常见问题，Valgrind 深度分析 |
| 发布前回归测试 | Valgrind | 全面检测，包括未初始化值使用 |
| 嵌入式/交叉编译 | ASan（若支持） | Valgrind 不支持交叉编译目标 |

> **⚠️ 注意**: ASan 和 Valgrind 不可同时使用。ASan 会与 Valgrind 的内存拦截器冲突。

## 7. 资源管理检查清单

在开发和代码审查过程中，使用以下检查清单确保资源管理的正确性：

- [ ] 每个资源都有明确的创建和释放函数
- [ ] 资源的所有权转移有明确的文档说明
- [ ] 错误处理路径中正确释放所有已分配的资源
- [ ] 资源的使用和释放是线程安全的
- [ ] 避免资源泄漏和双重释放
- [ ] 遵循本文档中的资源管理规范

## 8. 结论

资源管理是Airymax系统稳定性和可靠性的关键组成部分。所有开发者必须严格遵循本文档中的资源管理规范，确保资源的正确创建、使用和释放。通过统一的资源管理策略，可以减少内存泄漏、资源竞争等问题，提高系统的整体质量和可维护性。