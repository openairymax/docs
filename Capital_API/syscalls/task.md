Copyright (c) 2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 任务管理系统调用 API

**最新**: 2026-06-28
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_API/syscalls/task.md
**API 版本**: v1.0.0
---

## 概述

任务管理系统调用提供任务提交、查询、等待和取消的接口。所有函数使用 `agentos_sys_task_*` 前缀，任务通过字符串 ID 标识。

### 五维正交原则体现

| 维度 | 原则体现 | 具体实现 |
|------|----------|---------|
| **系统观** | 任务状态反馈闭环 | 提交→查询→等待→取消的完整生命周期 |
| **内核观** | 极简的接口契约 | 仅 4 个核心函数 |
| **认知观** | Thinkdual 任务路径 | 任务通过 CoreLoopThree 认知循环处理 |
| **工程观** | 安全与可观测性内置 | 统一错误码、结构化日志 |
| **设计美学** | 简洁的状态管理 | 统一的命名前缀、清晰的资源所有权 |

---

## API 索引

| 函数 | 描述 |
|------|------|
| `agentos_sys_task_submit()` | 提交任务到内核 |
| `agentos_sys_task_query()` | 查询任务状态 |
| `agentos_sys_task_wait()` | 等待任务完成 |
| `agentos_sys_task_cancel()` | 取消任务 |

---

## API 详细说明

### `agentos_sys_task_submit()`

```c
/**
 * @brief 提交任务到内核
 * @param input 输入数据（JSON 格式）
 * @param input_len 输入长度
 * @param timeout_ms 超时时间（毫秒）
 * @param out_output 输出参数，返回任务结果
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_output，通过 agentos_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AGENTOS_API agentos_error_t agentos_sys_task_submit(const char *input,
                                                     size_t input_len,
                                                     uint32_t timeout_ms,
                                                     char **out_output);
```

**使用示例**：
```c
char *result = NULL;
agentos_error_t err = agentos_sys_task_submit(
    "{\"task\": \"analyze data\"}", 27, 30000, &result);
if (err == AGENTOS_OK) {
    printf("Result: %s\n", result);
    agentos_sys_free(result);
}
```

### `agentos_sys_task_query()`

```c
/**
 * @brief 查询任务状态
 * @param task_id 任务 ID（字符串）
 * @param out_status 输出参数，返回任务状态
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @ownership 无堆内存分配
 * @threadsafe 是
 * @reentrant 是
 */
AGENTOS_API agentos_error_t agentos_sys_task_query(const char *task_id,
                                                    int *out_status);
```

### `agentos_sys_task_wait()`

```c
/**
 * @brief 等待任务完成
 * @param task_id 任务 ID（字符串）
 * @param timeout_ms 等待超时时间（毫秒）
 * @param out_result 输出参数，返回任务结果
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_result，通过 agentos_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AGENTOS_API agentos_error_t agentos_sys_task_wait(const char *task_id,
                                                   uint32_t timeout_ms,
                                                   char **out_result);
```

### `agentos_sys_task_cancel()`

```c
/**
 * @brief 取消任务
 * @param task_id 任务 ID（字符串）
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @threadsafe 是
 * @reentrant 否
 */
AGENTOS_API agentos_error_t agentos_sys_task_cancel(const char *task_id);
```

---

## 注意事项

1. **任务 ID 类型**：所有任务 ID 为 `const char *` 字符串类型（非 `uint64_t`）。
2. **输出所有权**：所有 `out_*` 参数由调用者通过 `agentos_sys_free()` 释放。
3. **错误码**：返回值使用 [error.h](../../AgentRT/agentos/commons/utils/error/include/error.h) 定义的负整数错误码。
4. **系统调用初始化**：使用任务相关 API 前需调用 `agentos_syscalls_init()`。

---

## 相关文档

- [系统调用总览](../README.md)
- [记忆管理系统调用](memory.md)
- [会话管理系统调用](session.md)
- [错误码体系](../../AgentRT/agentos/commons/utils/error/include/error.h)
- [架构设计：CoreLoopThree 认知循环](../../Capital_Architecture/coreloopthree.md)

---

© 2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
