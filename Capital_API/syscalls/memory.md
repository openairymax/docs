Copyright (c) 2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 记忆管理系统调用 API

**最新**: 2026-06-28
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_API/syscalls/memory.md
**API 版本**: v1.0.0
---

## 概述

记忆管理系统调用提供 MemoryRovol 记忆系统的写入、搜索、获取和删除接口。记忆系统采用四层渐进抽象架构（L1→L4），使用 HNSW 索引进行语义检索。所有函数使用 `agentos_sys_memory_*` 前缀。

### 五维正交原则体现

| 维度 | 原则体现 | 具体实现 |
|------|----------|---------|
| **系统观** | 四层记忆系统的层次抽象 | L1 原始卷→L2 特征层→L3 结构层→L4 模式层 |
| **内核观** | 极简的接口契约 | 仅 4 个核心函数 |
| **认知观** | 记忆卷载与检索 | 语义搜索支持，记录 ID 关联 |
| **工程观** | 安全与可观测性内置 | 统一错误码、输入净化（Cupolas） |
| **设计美学** | 简洁的数据操作 | 一致的命名前缀、清晰的资源所有权 |

---

## API 索引

| 函数 | 描述 |
|------|------|
| `agentos_sys_memory_write()` | 写入记忆数据 |
| `agentos_sys_memory_search()` | 语义搜索记忆 |
| `agentos_sys_memory_get()` | 获取记忆记录 |
| `agentos_sys_memory_delete()` | 删除记忆记录 |

---

## API 详细说明

### `agentos_sys_memory_write()`

```c
/**
 * @brief 写入记忆数据
 * @param data 数据指针
 * @param len 数据长度
 * @param metadata 元数据（JSON 格式）
 * @param out_record_id 输出参数，返回记录 ID
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_record_id，通过 agentos_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AGENTOS_API agentos_error_t agentos_sys_memory_write(const void *data,
                                                      size_t len,
                                                      const char *metadata,
                                                      char **out_record_id);
```

**使用示例**：
```c
char *record_id = NULL;
const char *content = "用户反馈：产品体验良好";
agentos_error_t err = agentos_sys_memory_write(
    content, strlen(content),
    "{\"type\": \"feedback\", \"rating\": 5}",
    &record_id);
if (err == AGENTOS_OK) {
    printf("Record ID: %s\n", record_id);
    agentos_sys_free(record_id);
}
```

### `agentos_sys_memory_search()`

```c
/**
 * @brief 语义搜索记忆
 * @param query 查询字符串
 * @param limit 返回数量限制
 * @param out_record_ids 输出参数，返回记录 ID 数组
 * @param out_scores 输出参数，返回相似度分数数组
 * @param out_count 输出参数，返回结果数量
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_record_ids、*out_scores，通过 agentos_sys_free() 释放
 * @threadsafe 是
 * @reentrant 是
 */
AGENTOS_API agentos_error_t agentos_sys_memory_search(const char *query,
                                                       uint32_t limit,
                                                       char ***out_record_ids,
                                                       float **out_scores,
                                                       size_t *out_count);
```

**使用示例**：
```c
char **record_ids = NULL;
float *scores = NULL;
size_t count = 0;
agentos_error_t err = agentos_sys_memory_search(
    "用户体验反馈", 10, &record_ids, &scores, &count);
if (err == AGENTOS_OK) {
    for (size_t i = 0; i < count; i++) {
        printf("[%zu] %s (score: %.2f)\n", i, record_ids[i], scores[i]);
    }
    agentos_sys_free(record_ids);
    agentos_sys_free(scores);
}
```

### `agentos_sys_memory_get()`

```c
/**
 * @brief 获取记忆记录
 * @param record_id 记录 ID
 * @param out_data 输出参数，返回数据指针
 * @param out_len 输出参数，返回数据长度
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_data，通过 agentos_sys_free() 释放
 * @threadsafe 是
 * @reentrant 是
 */
AGENTOS_API agentos_error_t agentos_sys_memory_get(const char *record_id,
                                                    void **out_data,
                                                    size_t *out_len);
```

### `agentos_sys_memory_delete()`

```c
/**
 * @brief 删除记忆记录
 * @param record_id 记录 ID
 * @return agentos_error_t (AGENTOS_OK=0 成功，负值=错误)
 *
 * @threadsafe 是
 * @reentrant 否
 */
AGENTOS_API agentos_error_t agentos_sys_memory_delete(const char *record_id);
```

---

## 注意事项

1. **记忆索引**：语义搜索基于 HNSW 索引（非 FAISS）。详见 [MemoryRovol 架构](../../Capital_Architecture/memoryrovol.md)。
2. **输出所有权**：所有 `out_*` 参数由调用者通过 `agentos_sys_free()` 释放。
3. **错误码**：返回值使用 [error.h](../../AgentRT/agentos/commons/utils/error/include/error.h) 定义的负整数错误码。
4. **系统调用初始化**：使用记忆相关 API 前需调用 `agentos_syscalls_init()`。

---

## 相关文档

- [系统调用总览](../README.md)
- [任务管理系统调用](task.md)
- [会话管理系统调用](session.md)
- [架构设计：MemoryRovol 记忆系统](../../Capital_Architecture/memoryrovol.md)
- [错误码体系](../../AgentRT/agentos/commons/utils/error/include/error.h)

---

© 2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
