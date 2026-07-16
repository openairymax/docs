Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 会话管理系统调用 API
> **文档定位**：Airymax 会话管理系统调用 API\
> **最后更新**：2026-06-28\
> **上级文档**：[AirymaxAgentRT 接口文档](README.md)\
> **API 版本**：v1.0.1

---

## 概述

会话管理系统调用提供会话生命周期管理接口：创建、查询、关闭和列出会话，以及查询会话持久化状态。会话是 Agent 与用户交互的上下文容器。所有函数使用 `airy_sys_session_*` 前缀。

### 五维正交原则体现

| 维度 | 原则体现 | 具体实现 |
|------|----------|---------|
| **系统观** | 会话状态机的完整反馈闭环 | 创建→查询→关闭的完整生命周期 |
| **内核观** | 极简的接口契约 | 仅 5 个核心函数 |
| **认知观** | Thinkdual 交互上下文管理 | 会话支持 t1-f/t2 思考路径切换 |
| **工程观** | 安全与可观测性内置 | 统一错误码、会话持久化状态查询 |
| **设计美学** | 简洁的会话操作 | 一致的命名前缀、JSON 元数据支持 |

---

## API 索引

| 函数 | 描述 |
|------|------|
| `airy_sys_session_create()` | 创建新会话 |
| `airy_sys_session_get()` | 获取会话信息 |
| `airy_sys_session_close()` | 关闭会话 |
| `airy_sys_session_list()` | 列出所有会话 |
| `airy_sys_session_get_persist_status()` | 获取会话持久化状态 |

---

## API 详细说明

### `airy_sys_session_create()`

```c
/**
 * @brief 创建新会话
 * @param metadata 会话元数据（JSON 格式，可为 NULL）
 * @param out_session_id 输出参数，返回会话 ID
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_session_id，通过 airy_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AIRY_API airy_err_t airy_sys_session_create(const char *metadata,
                                                        char **out_session_id);
```

**使用示例**：
```c
char *session_id = NULL;
airy_err_t err = airy_sys_session_create(
    "{\"user\": \"alice\", \"type\": \"chat\"}", &session_id);
if (err == AIRY_EOK) {
    printf("Session created: %s\n", session_id);
    airy_sys_free(session_id);
}
```

### `airy_sys_session_get()`

```c
/**
 * @brief 获取会话信息
 * @param session_id 会话 ID
 * @param out_info 输出参数，返回会话信息（JSON 格式）
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_info，通过 airy_sys_free() 释放
 * @threadsafe 是
 * @reentrant 是
 */
AIRY_API airy_err_t airy_sys_session_get(const char *session_id,
                                                     char **out_info);
```

### `airy_sys_session_close()`

```c
/**
 * @brief 关闭会话
 * @param session_id 会话 ID
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @threadsafe 是
 * @reentrant 否
 */
AIRY_API airy_err_t airy_sys_session_close(const char *session_id);
```

### `airy_sys_session_list()`

```c
/**
 * @brief 列出所有会话
 * @param out_sessions 输出参数，返回会话 ID 数组
 * @param out_count 输出参数，返回会话数量
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_sessions，通过 airy_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AIRY_API airy_err_t airy_sys_session_list(char ***out_sessions,
                                                      size_t *out_count);
```

**使用示例**：
```c
char **sessions = NULL;
size_t count = 0;
airy_err_t err = airy_sys_session_list(&sessions, &count);
if (err == AIRY_EOK) {
    for (size_t i = 0; i < count; i++) {
        printf("[%zu] %s\n", i, sessions[i]);
    }
    airy_sys_free(sessions);
}
```

### `airy_sys_session_get_persist_status()`

```c
/**
 * @brief 获取会话持久化状态
 * @param session_id 会话 ID
 * @param out_status 输出参数，返回持久化状态
 * @param out_error 输出参数，返回持久化错误码（可为 NULL）
 * @return airy_err_t (AIRY_EOK=0 成功，负值=错误)
 *
 * @threadsafe 是
 * @reentrant 是
 */
AIRY_API airy_err_t airy_sys_session_get_persist_status(
    const char *session_id,
    session_persist_status_t *out_status,
    airy_err_t *out_error);
```

**持久化状态枚举**：
```c
typedef enum {
    SESSION_PERSIST_UNKNOWN = 0,  // 未知状态
    SESSION_PERSIST_PENDING,      // 等待持久化
    SESSION_PERSIST_SUCCESS,      // 持久化成功
    SESSION_PERSIST_FAILED,       // 持久化失败
    SESSION_PERSIST_DISABLED      // 持久化禁用
} session_persist_status_t;
```

---

## 注意事项

1. **输出所有权**：所有 `out_*` 参数由调用者通过 `airy_sys_free()` 释放。
2. **错误码**：返回值使用 [error.h](../../agentrt/commons/utils/error/include/error.h) 定义的负整数错误码。
3. **系统调用初始化**：使用会话相关 API 前需调用 `airy_syscalls_init()`。
4. **会话 ID 格式**：返回的会话 ID 为字符串类型，由系统内部生成。

---

## 相关文档

- [系统调用总览](../README.md)
- [任务管理系统调用](task.md)
- [记忆管理系统调用](memory.md)
- [Agent 管理系统调用](agent.md)
- [架构设计：CoreLoopThree 认知循环](../../10-architecture/02-coreloopthree.md)
- [错误码体系](../../agentrt/commons/utils/error/include/error.h)

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
