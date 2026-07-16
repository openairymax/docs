Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 任务管理系统调用 API
> **文档定位**：Airymax 任务管理系统调用 API\
> **最后更新**：2026-06-28\
> **上级文档**：[AirymaxAgentRT 接口文档](README.md)\
> **API 版本**：v1.0.1

---

## 概述

任务管理系统调用提供任务提交、查询、等待和取消的接口。所有函数使用 `airy_sys_task_*` 前缀，任务通过字符串 ID 标识。

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
| `airy_sys_task_submit()` | 提交任务到内核 |
| `airy_sys_task_query()` | 查询任务状态 |
| `airy_sys_task_wait()` | 等待任务完成 |
| `airy_sys_task_cancel()` | 取消任务 |

---

## API 详细说明

### `airy_sys_task_submit()`

```c
/**
 * @brief 提交任务到内核
 * @param input 输入数据（JSON 格式）
 * @param input_len 输入长度
 * @param timeout_ms 超时时间（毫秒）
 * @param out_output 输出参数，返回任务结果
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_output，通过 airy_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AIRY_API airy_err_t airy_sys_task_submit(const char *input,
                                                     size_t input_len,
                                                     uint32_t timeout_ms,
                                                     char **out_output);
```

**使用示例**：
```c
char *result = NULL;
airy_err_t err = airy_sys_task_submit(
    "{\"task\": \"analyze data\"}", 27, 30000, &result);
if (err == AIRY_EOK) {
    printf("Result: %s\n", result);
    airy_sys_free(result);
}
```

### `airy_sys_task_query()`

```c
/**
 * @brief 查询任务状态
 * @param task_id 任务 ID（字符串）
 * @param out_status 输出参数，返回任务状态
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @ownership 无堆内存分配
 * @threadsafe 是
 * @reentrant 是
 */
AIRY_API airy_err_t airy_sys_task_query(const char *task_id,
                                                    int *out_status);
```

### `airy_sys_task_wait()`

```c
/**
 * @brief 等待任务完成
 * @param task_id 任务 ID（字符串）
 * @param timeout_ms 等待超时时间（毫秒）
 * @param out_result 输出参数，返回任务结果
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_result，通过 airy_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AIRY_API airy_err_t airy_sys_task_wait(const char *task_id,
                                                   uint32_t timeout_ms,
                                                   char **out_result);
```

### `airy_sys_task_cancel()`

```c
/**
 * @brief 取消任务
 * @param task_id 任务 ID（字符串）
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @threadsafe 是
 * @reentrant 否
 */
AIRY_API airy_err_t airy_sys_task_cancel(const char *task_id);
```

---

## 注意事项

1. **任务 ID 类型**：所有任务 ID 为 `const char *` 字符串类型（非 `uint64_t`）。
2. **输出所有权**：所有 `out_*` 参数由调用者通过 `airy_sys_free()` 释放。
3. **错误码**：返回值使用 [error.h](../../agentrt/commons/utils/error/include/error.h) 定义的负整数错误码。
4. **系统调用初始化**：使用任务相关 API 前需调用 `airy_syscalls_init()`。

---

## 相关文档

- [系统调用总览](../README.md)
- [记忆管理系统调用](memory.md)
- [会话管理系统调用](session.md)
- [错误码体系](../../agentrt/commons/utils/error/include/error.h)
- [架构设计：CoreLoopThree 认知循环](../../10-architecture/02-coreloopthree.md)

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
