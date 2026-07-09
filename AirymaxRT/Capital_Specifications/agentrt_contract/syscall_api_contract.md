Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 系统调用 API 规范

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/agentrt_contract/syscall_api_contract.md
---

## 编制说明

### 本文档定位

系统调用 API 规范是 Airymax 规范体系的核心组成部分，属于**操作层规范**。本规范定义了用户态程序与内核交互的唯一接口，是 Airymax MicroCoreRT 微核心架构的关键实现机制。

### 与设计哲学的关系

本规范是 Airymax 五维正交设计体系在系统调用层面的具体实现，每个维度都有对应的设计体现：

#### 五维正交体系映射

| 维度 | 设计原则 | 在系统调用中的体现 |
|------|---------|------------------|
| **系统观（S维度）** | S-2 层次分解原则 | 系统调用作为用户态与内核态的明确边界，严格遵循层次分解，用户态程序只能通过系统调用接口访问内核功能 |
| | S-1 反馈闭环原则 | 所有系统调用都内置可观测性数据采集，为系统层面的反馈闭环提供数据支持 |
| **内核观（K维度）** | K-1 内核极简原则 | 系统调用只暴露最核心的内核功能（任务管理、记忆管理、会话管理），其他功能外置为服务 |
| | K-2 接口契约化原则 | 每个系统调用都有完整的契约定义，包括参数方向、所有权语义、线程安全性等 |
| | K-3 服务隔离原则 | 用户态服务层（daemon）必须通过系统调用与内核交互，禁止直接通信 |
| **认知观（C维度）** | C-1 双思考系统协同原则 | 系统调用支持不同的调用策略，允许在快速路径（t1-f）和慢速路径（t2）之间权衡 |
| **工程观（E维度）** | E-1 安全内生原则 | 每个系统调用都内置参数校验、权限检查和审计日志记录 |
| | E-2 可观测性原则 | 所有系统调用自动记录 TraceID、耗时、错误码等可观测数据 |
| | E-3 资源确定性原则 | 系统调用遵循明确的内存管理规则，输入输出参数所有权清晰 |
| | E-6 错误可追溯原则 | 统一的错误码体系和完整的错误上下文信息 |
| | E-8 可测试性原则 | 系统调用接口设计支持单元测试、集成测试和性能测试 |
| **设计美学（A维度）** | A-1 简约至上原则 | 接口数量最小化，每个系统调用只做一件事且做到极致 |
| | A-2 极致细节原则 | 错误消息包含具体错误、数值上下文和修复建议 |
| | A-4 完美主义原则 | 零警告编译，完整的测试覆盖，文档与代码同步更新 |

#### 理论基础融合

系统调用规范是多重理论融合的产物：
- **工程两论**：通过层次分解（系统工程）和反馈机制（控制论）构建稳定的系统调用架构
- **Thinkdual 双思考系统**：支持不同认知需求的调用策略，平衡效率与精度（t1-f 快思考 / t2 慢思考）
- **MicroCoreRT 微核心设计**：最小化内核接口，最大化服务隔离
- **设计美学**：追求接口的简约、对称和自解释性

### 适用范围

本规范适用于以下场景：

1. **应用开发者**: 使用系统调用开发上层应用
2. **服务开发者**: agentos/daemon/ 用户态服务层通过系统调用访问内核功能
3. **内核开发者**: 实现和维护系统调用接口
4. **安全审计人员**: 审查系统调用的安全性和合规性

### 术语定义

本规范使用的术语定义见 [统一术语表](../TERMINOLOGY.md),关键术语包括：

| 术语 | 简要定义 | 来源 |
|------|---------|------|
| 系统调用 (Syscall) | 用户态进入内核的唯一入口 | 本规范 |
| MicroCoreRT 微核心/原子核心 (MicroCoreRT/CoreKern) | 只提供不可再分原子机制的最小化内核：IPC、内存管理、任务调度、时间服务 | [架构设计原则](../../../Capital_Architecture/ARCHITECTURAL_PRINCIPLES.md) |
| TraceID | 分布式追踪的唯一标识 | [日志打印规范](../coding_standard/Log_standard.md) |

---

## 第 1 章 概述

### 1.1 系统调用简介

Airymax 系统调用 (syscall) 是用户态程序与内核交互的唯一接口。它遵循 MicroCoreRT 微核心设计思想，将内核核心功能 (任务管理、记忆管理、会话管理、可观测性) 以稳定、安全的 API 形式暴露给上层应用和服务。

所有系统调用定义于 `agentos/atoms/syscall/include/syscalls.h`，使用 `agentos_sys_*` 统一命名前缀，确保参数校验和权限控制的一致性。

### 1.2 设计目标

本规范旨在实现以下目标：

1. **接口稳定**: 提供长期稳定的 API 接口，支持向后兼容
2. **安全可靠**: 内置参数校验、权限检查、审计日志
3. **性能优良**: 最小化上下文切换开销，优化热路径
4. **易于使用**: 提供清晰的文档和示例，降低使用门槛
5. **可观测完备**: 自动记录 TraceID、耗时、错误码等指标
6. **可测试性优**: 接口设计支持全面的单元测试、集成测试和性能测试，确保代码质量和可靠性

### 1.3 调用约定

#### 1.3.1 返回值类型

系统调用返回值统一为 `agentos_error_t`(整数错误码):

```c
typedef int32_t agentos_error_t;

// 常见错误码（定义于 agentos/commons/utils/error/include/error.h）
#define AGENTOS_OK                     0    // 成功
#define AGENTOS_ERR_UNKNOWN           -1    // 未知错误
#define AGENTOS_ERR_INVALID_PARAM     -2    // 参数无效
#define AGENTOS_ERR_OUT_OF_MEMORY     -4    // 内存不足
#define AGENTOS_ERR_NOT_FOUND         -6    // 资源不存在
#define AGENTOS_ERR_TIMEOUT           -8    // 操作超时
#define AGENTOS_ERR_PERMISSION_DENIED -10   // 权限不足
#define AGENTOS_ERR_BUSY              -17   // 资源忙
#define AGENTOS_ERR_FAIL              -31   // 内部错误
```

完整错误码列表见 [第 7 章 错误码](#第 7 章 错误码)。

#### 1.3.2 同步阻塞模型

系统调用采用同步阻塞模型 (任务提交和等待除外，支持超时):

```c
// 同步调用，等待结果返回
char* result = NULL;
agentos_error_t err = agentos_sys_task_submit(
    "Develop a feature", 17, 30000, &result);
// 调用返回时，任务已完成或超时
```

#### 1.3.3 内存管理规则

- **输入参数**: 由调用者分配，内核不持有引用
- **输出参数**: 由内核分配，调用者负责释放
- **释放方式**: 使用 `agentos_sys_free()` 函数

**示例:**
```c
char** record_ids = NULL;
float* scores = NULL;
size_t count = 0;
agentos_error_t err = agentos_sys_memory_search("query", 5, &record_ids, &scores, &count);
if (err == AGENTOS_OK) {
    for (size_t i = 0; i < count; i++) {
        printf("Record %zu: %s (score: %.2f)\n", i, record_ids[i], scores[i]);
    }
    agentos_sys_free(record_ids);  // 必须使用 agentos_sys_free 释放
    agentos_sys_free(scores);
}
```

---

## 第 2 章 系统调用列表

### 2.1 按功能分类

| 类别 | 系统调用 | 功能描述 | 复杂度 |
| :--- | :--- | :--- | :--- |
| **初始化** | `agentos_syscalls_init` | 初始化系统调用子系统 | O(1) |
| | `agentos_syscalls_cleanup` | 清理系统调用子系统 | O(1) |
| **任务管理** | `agentos_sys_task_submit` | 提交自然语言任务，等待执行完成并返回结果 | O(n) |
| | `agentos_sys_task_query` | 查询任务状态 | O(1) |
| | `agentos_sys_task_wait` | 等待指定任务完成并获取结果 | O(n) |
| | `agentos_sys_task_cancel` | 取消任务 | O(1) |
| **记忆管理** | `agentos_sys_memory_write` | 写入原始记忆 | O(log n) |
| | `agentos_sys_memory_search` | 语义搜索记忆 | O(log n) |
| | `agentos_sys_memory_get` | 获取记忆原始数据 | O(1) |
| | `agentos_sys_memory_delete` | 删除记忆 | O(log n) |
| **会话管理** | `agentos_sys_session_create` | 创建新会话 | O(1) |
| | `agentos_sys_session_get` | 获取会话信息 | O(1) |
| | `agentos_sys_session_close` | 关闭会话 | O(1) |
| | `agentos_sys_session_list` | 列出所有活跃会话 | O(n) |
| | `agentos_sys_session_get_persist_status` | 获取会话持久化状态 | O(1) |
| **Agent 管理** | `agentos_sys_agent_spawn` | 生成新的 Agent 实例 | O(1) |
| | `agentos_sys_agent_terminate` | 终止 Agent 实例 | O(1) |
| | `agentos_sys_agent_invoke` | 调用 Agent 执行任务 | O(n) |
| | `agentos_sys_agent_list` | 列出所有 Agent 实例 | O(n) |
| **Skill 管理** | `agentos_sys_skill_install` | 安装 Skill | O(1) |
| | `agentos_sys_skill_execute` | 执行 Skill | O(n) |
| | `agentos_sys_skill_list` | 列出所有已安装 Skill | O(n) |
| | `agentos_sys_skill_uninstall` | 卸载 Skill | O(1) |
| **可观测性** | `agentos_sys_telemetry_metrics` | 导出系统指标 (JSON) | O(1) |
| | `agentos_sys_telemetry_traces` | 导出追踪数据 (JSON) | O(n) |
| **辅助函数** | `agentos_sys_free` | 释放内核分配的内存 | O(1) |

### 2.2 按优先级分类

**关键路径 (Critical Path):**
- `agentos_sys_task_submit`: 任务提交，核心业务逻辑
- `agentos_sys_memory_search`: 记忆检索，高频调用

**重要路径 (Important Path):**
- `agentos_sys_task_query`: 任务查询
- `agentos_sys_session_create`: 会话创建
- `agentos_sys_memory_write`: 记忆写入

**辅助路径 (Auxiliary Path):**
- `agentos_sys_telemetry_metrics`: 指标导出
- `agentos_sys_session_list`: 会话列表

---

## 第 3 章 详细说明

### 3.1 初始化

#### 3.1.1 `agentos_syscalls_init`

**功能**: 初始化系统调用子系统，为后续系统调用做好准备。

**语法**:
```c
agentos_error_t agentos_syscalls_init(void);
```

**参数**: 无

**返回值**:
- `AGENTOS_OK`: 初始化成功
- `AGENTOS_ERR_FAIL`: 初始化失败

**说明**: 
- 此函数应在内核启动后、任何其他系统调用之前调用一次
- 多次调用是安全的，后续调用将直接返回成功
- 若初始化失败，系统调用功能不可用

**示例**:
```c
// 内核启动流程
int main() {
    // 1. 初始化系统调用子系统
    agentos_error_t err = agentos_syscalls_init();
    if (err != AGENTOS_OK) {
        fprintf(stderr, "Syscall init failed: %d\n", err);
        return 1;
    }

    // 2. 开始处理请求
    // ...

    // 3. 清理
    agentos_syscalls_cleanup();
    return 0;
}
```

#### 3.1.2 `agentos_syscalls_cleanup`

**功能**: 清理系统调用子系统，释放所有内核资源。

**语法**:
```c
void agentos_syscalls_cleanup(void);
```

**参数**: 无

**返回值**: 无

**说明**: 
- 此函数应在内核退出前调用一次
- 调用后所有系统调用功能不可用
- 重复调用不会导致未定义行为

### 3.2 任务管理

#### 3.2.1 `agentos_sys_task_submit`

**功能**: 提交一个自然语言任务，同步等待执行完成并返回结果。任务内部会调用认知层生成计划，并由执行层按 DAG 顺序执行。

**语法**:
```c
agentos_error_t agentos_sys_task_submit(
    const char* input,
    size_t input_len,
    uint32_t timeout_ms,
    char** out_output
);
```

**参数**:
- `input`: 自然语言描述的任务目标
- `input_len`: 输入长度
- `timeout_ms`: 超时 (毫秒),0 表示无限等待
- `out_output`: 输出结果 (JSON 字符串),调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足
- `AGENTOS_ERR_TIMEOUT`: 任务超时
- `AGENTOS_ERR_FAIL`: 内部错误

**性能特征**:
- **时间复杂度**: O(n),n 为任务复杂度
- **空间复杂度**: O(m),m 为输出结果大小
- **典型耗时**: 简单任务 1-5 秒，复杂任务 10-30 秒

**示例**:
```c
char* result = NULL;
agentos_error_t err = agentos_sys_task_submit(
    "开发一个电商应用", 18, 30000, &result);
if (err == AGENTOS_OK) {
    printf("Task result: %s\n", result);
    agentos_sys_free(result);
} else {
    fprintf(stderr, "Task failed: %d\n", err);
}
```

**最佳实践**:

**✅ 推荐做法**
```c
// 1. 始终检查返回值
char* result = NULL;
if (agentos_sys_task_submit(...) == AGENTOS_OK) {
    // 处理结果
    agentos_sys_free(result);
}

// 2. 合理设置超时时间
agentos_sys_task_submit(task, len, 30000, &result);  // 30 秒超时

// 3. 及时释放输出结果
if (result) {
    agentos_sys_free(result);
    result = NULL;
}
```

**❌ 禁止做法**
```c
// 1. 不检查返回值
agentos_sys_task_submit(..., &result);  // 危险！可能失败
printf("Result: %s\n", result);  // result 可能为 NULL

// 2. 不释放内存
char* result = NULL;
agentos_sys_task_submit(..., &result);
// 忘记 agentos_sys_free(result),导致内存泄漏

// 3. 超时时间设置不合理
agentos_sys_task_submit(..., 100, &result);  // 100ms 太短，容易超时
agentos_sys_task_submit(..., 0, &result);    // 无限等待，可能永久阻塞
```

#### 3.2.2 `agentos_sys_task_query`

**功能**: 查询指定任务的状态。

**语法**:
```c
agentos_error_t agentos_sys_task_query(
    const char* task_id,
    int* out_status
);
```

**参数**:
- `task_id`: 任务 ID(由 `agentos_sys_task_submit` 返回的 ID)
- `out_status`: 输出状态值

**状态枚举**:
```c
enum task_status {
    TASK_PENDING = 0,    // 等待执行
    TASK_RUNNING = 1,    // 正在执行
    TASK_SUCCEEDED = 2,  // 执行成功
    TASK_FAILED = 3,     // 执行失败
    TASK_CANCELLED = 4   // 已取消
};
```

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_NOT_FOUND`: 任务不存在
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效

**示例**:
```c
int status;
if (agentos_sys_task_query(task_id, &status) == AGENTOS_OK) {
    switch (status) {
        case TASK_PENDING:
            printf("Task is pending\n");
            break;
        case TASK_RUNNING:
            printf("Task is running\n");
            break;
        case TASK_SUCCEEDED:
            printf("Task succeeded\n");
            break;
        case TASK_FAILED:
            printf("Task failed\n");
            break;
        case TASK_CANCELLED:
            printf("Task cancelled\n");
            break;
    }
}
```

#### 3.2.3 `agentos_sys_task_wait`

**功能**: 等待指定任务完成并获取结果。

**语法**:
```c
agentos_error_t agentos_sys_task_wait(
    const char* task_id,
    uint32_t timeout_ms,
    char** out_result
);
```

**参数**:
- `task_id`: 任务 ID
- `timeout_ms`: 超时 (毫秒)
- `out_result`: 输出结果，调用者需释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_NOT_FOUND`: 任务不存在
- `AGENTOS_ERR_TIMEOUT`: 等待超时
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**与 `task_submit` 的区别**:
- `task_submit`: 提交并立即执行，适用于新任务
- `task_wait`: 等待已提交的任务，适用于异步场景

**示例**:
```c
// 1. 异步提交任务
const char* task_id = submit_task_async("...");

// 2. 做其他事情
do_other_work();

// 3. 等待任务完成
char* result = NULL;
if (agentos_sys_task_wait(task_id, 30000, &result) == AGENTOS_OK) {
    printf("Task completed: %s\n", result);
    agentos_sys_free(result);
}
```

#### 3.2.4 `agentos_sys_task_cancel`

**功能**: 取消任务。

**语法**:
```c
agentos_error_t agentos_sys_task_cancel(const char* task_id);
```

**参数**:
- `task_id`: 任务 ID

**返回值**:
- `AGENTOS_OK`: 取消成功
- `AGENTOS_ERR_NOT_FOUND`: 任务不存在
- `AGENTOS_ERR_PERMISSION_DENIED`: 无权取消该任务

**说明**:
- 只能取消 PENDING 或 RUNNING 状态的任务
- 已完成的任务无法取消
- 取消操作是异步的，不保证立即生效

**示例**:
```c
if (agentos_sys_task_cancel(task_id) == AGENTOS_OK) {
    printf("Task cancelled successfully\n");
} else {
    fprintf(stderr, "Failed to cancel task\n");
}
```

### 3.3 记忆管理

#### 3.3.1 `agentos_sys_memory_write`

**功能**: 写入原始记忆。

**语法**:
```c
agentos_error_t agentos_sys_memory_write(
    const void* data,
    size_t len,
    const char* metadata,
    char** out_record_id
);
```

**参数**:
- `data`: 记忆数据 (二进制或文本)
- `len`: 数据长度
- `metadata`: 元数据 (JSON 字符串，可选，可为 NULL)
- `out_record_id`: 输出记录 ID,调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
const char* data = "{\"type\":\"fact\",\"content\":\"Earth is round\"}";
const char* meta = "{\"source\":\"user\",\"priority\":1}";
char* record_id = NULL;

if (agentos_sys_memory_write(data, strlen(data), meta, &record_id) == AGENTOS_OK) {
    printf("Memory written, record_id: %s\n", record_id);
    agentos_sys_free(record_id);
}
```

#### 3.3.2 `agentos_sys_memory_search`

**功能**: 语义搜索记忆。

**语法**:
```c
agentos_error_t agentos_sys_memory_search(
    const char* query,
    uint32_t limit,
    char*** out_record_ids,
    float** out_scores,
    size_t* out_count
);
```

**参数**:
- `query`: 搜索查询 (自然语言)
- `limit`: 最大返回结果数量
- `out_record_ids`: 输出记录 ID 数组，调用者需使用 `agentos_sys_free()` 释放
- `out_scores`: 输出相关性分数数组，调用者需使用 `agentos_sys_free()` 释放
- `out_count`: 输出结果数量

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足
- `AGENTOS_ERR_NOT_FOUND`: 未找到匹配结果

**示例**:
```c
char** record_ids = NULL;
float* scores = NULL;
size_t count = 0;

if (agentos_sys_memory_search("地球形状", 5, &record_ids, &scores, &count) == AGENTOS_OK) {
    for (size_t i = 0; i < count; i++) {
        printf("Record %s (score: %.2f)\n", record_ids[i], scores[i]);
    }
    agentos_sys_free(record_ids);
    agentos_sys_free(scores);
}
```

#### 3.3.3 `agentos_sys_memory_get`

**功能**: 获取记忆原始数据。

**语法**:
```c
agentos_error_t agentos_sys_memory_get(
    const char* record_id,
    void** out_data,
    size_t* out_len
);
```

**参数**:
- `record_id`: 记录 ID
- `out_data`: 记忆数据，调用者需使用 `agentos_sys_free()` 释放
- `out_len`: 输出数据长度

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_NOT_FOUND`: 记录不存在
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
void* data = NULL;
size_t len = 0;
if (agentos_sys_memory_get(record_id, &data, &len) == AGENTOS_OK) {
    printf("Memory data (%zu bytes)\n", len);
    agentos_sys_free(data);
}
```

#### 3.3.4 `agentos_sys_memory_delete`

**功能**: 删除记忆。

**语法**:
```c
agentos_error_t agentos_sys_memory_delete(const char* record_id);
```

**参数**:
- `record_id`: 记录 ID

**返回值**:
- `AGENTOS_OK`: 删除成功
- `AGENTOS_ERR_NOT_FOUND`: 记录不存在
- `AGENTOS_ERR_PERMISSION_DENIED`: 无权删除

**示例**:
```c
if (agentos_sys_memory_delete(record_id) == AGENTOS_OK) {
    printf("Memory deleted\n");
}
```

### 3.4 会话管理

#### 3.4.1 `agentos_sys_session_create`

**功能**: 创建新会话。

**语法**:
```c
agentos_error_t agentos_sys_session_create(
    const char* metadata,
    char** out_session_id
);
```

**参数**:
- `metadata`: 会话元数据 (JSON 格式)
- `out_session_id`: 会话 ID,调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* session_id = NULL;
const char* metadata = "{\"user\":\"alice\",\"ttl\":3600}";
if (agentos_sys_session_create(metadata, &session_id) == AGENTOS_OK) {
    printf("Session created: %s\n", session_id);
    agentos_sys_free(session_id);
}
```

#### 3.4.2 `agentos_sys_session_get`

**功能**: 获取会话信息。

**语法**:
```c
agentos_error_t agentos_sys_session_get(
    const char* session_id,
    char** out_info
);
```

**参数**:
- `session_id`: 会话 ID
- `out_info`: 会话信息 (JSON 格式),调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_NOT_FOUND`: 会话不存在
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* info = NULL;
if (agentos_sys_session_get(sess_id, &info) == AGENTOS_OK) {
    printf("Session info: %s\n", info);
    agentos_sys_free(info);
}
```

#### 3.4.3 `agentos_sys_session_close`

**功能**: 关闭会话。

**语法**:
```c
agentos_error_t agentos_sys_session_close(const char* session_id);
```

**参数**:
- `session_id`: 会话 ID

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_NOT_FOUND`: 会话不存在

**示例**:
```c
agentos_sys_session_close(sess_id);
```

#### 3.4.4 `agentos_sys_session_list`

**功能**: 列出所有活跃会话。

**语法**:
```c
agentos_error_t agentos_sys_session_list(char*** out_sessions, size_t* out_count);
```

**参数**:
- `out_sessions`: 会话 ID 数组，调用者需使用 `agentos_sys_free()` 释放
- `out_count`: 输出会话数量

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char** sessions = NULL;
size_t count = 0;
if (agentos_sys_session_list(&sessions, &count) == AGENTOS_OK) {
    for (size_t i = 0; i < count; i++) {
        printf("Session: %s\n", sessions[i]);
    }
    agentos_sys_free(sessions);
}
```

#### 3.4.5 `agentos_sys_session_get_persist_status`

**功能**: 获取会话持久化状态。

**语法**:
```c
agentos_error_t agentos_sys_session_get_persist_status(
    const char* session_id,
    session_persist_status_t* out_status,
    agentos_error_t* out_error
);
```

**参数**:
- `session_id`: 会话 ID
- `out_status`: 输出持久化状态
- `out_error`: 输出详细错误信息

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_NOT_FOUND`: 会话不存在

**示例**:
```c
session_persist_status_t status;
agentos_error_t persist_err;
if (agentos_sys_session_get_persist_status(sess_id, &status, &persist_err) == AGENTOS_OK) {
    printf("Persist status: %d\n", status);
}
```

### 3.5 Agent 管理

#### 3.5.1 `agentos_sys_agent_spawn`

**功能**: 生成新的 Agent 实例。

**语法**:
```c
agentos_error_t agentos_sys_agent_spawn(
    const char* agent_spec,
    char** out_agent_id
);
```

**参数**:
- `agent_spec`: Agent 规格说明 (JSON 格式)
- `out_agent_id`: 输出 Agent ID,调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* agent_id = NULL;
const char* spec = "{\"name\":\"assistant\",\"model\":\"gpt-4\"}";
if (agentos_sys_agent_spawn(spec, &agent_id) == AGENTOS_OK) {
    printf("Agent created: %s\n", agent_id);
    agentos_sys_free(agent_id);
}
```

#### 3.5.2 `agentos_sys_agent_terminate`

**功能**: 终止 Agent 实例。

**语法**:
```c
agentos_error_t agentos_sys_agent_terminate(const char* agent_id);
```

**参数**:
- `agent_id`: Agent ID

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_NOT_FOUND`: Agent 不存在
- `AGENTOS_ERR_PERMISSION_DENIED`: 无权终止

**示例**:
```c
if (agentos_sys_agent_terminate(agent_id) == AGENTOS_OK) {
    printf("Agent terminated\n");
}
```

#### 3.5.3 `agentos_sys_agent_invoke`

**功能**: 调用 Agent 执行任务。

**语法**:
```c
agentos_error_t agentos_sys_agent_invoke(
    const char* agent_id,
    const char* input,
    size_t input_len,
    char** out_output
);
```

**参数**:
- `agent_id`: Agent ID
- `input`: 输入数据
- `input_len`: 输入长度
- `out_output`: 输出结果，调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_NOT_FOUND`: Agent 不存在
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* output = NULL;
if (agentos_sys_agent_invoke(agent_id, "Hello", 5, &output) == AGENTOS_OK) {
    printf("Agent response: %s\n", output);
    agentos_sys_free(output);
}
```

#### 3.5.4 `agentos_sys_agent_list`

**功能**: 列出所有 Agent 实例。

**语法**:
```c
agentos_error_t agentos_sys_agent_list(char*** out_agent_ids, size_t* out_count);
```

**参数**:
- `out_agent_ids`: Agent ID 数组，调用者需使用 `agentos_sys_free()` 释放
- `out_count`: 输出 Agent 数量

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char** agent_ids = NULL;
size_t count = 0;
if (agentos_sys_agent_list(&agent_ids, &count) == AGENTOS_OK) {
    for (size_t i = 0; i < count; i++) {
        printf("Agent: %s\n", agent_ids[i]);
    }
    agentos_sys_free(agent_ids);
}
```

### 3.6 Skill 管理

#### 3.6.1 `agentos_sys_skill_install`

**功能**: 安装 Skill。

**语法**:
```c
agentos_error_t agentos_sys_skill_install(
    const char* skill_url,
    char** out_skill_id
);
```

**参数**:
- `skill_url`: Skill 下载地址
- `out_skill_id`: 输出 Skill ID,调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* skill_id = NULL;
if (agentos_sys_skill_install("https://example.com/skills/calc.zip", &skill_id) == AGENTOS_OK) {
    printf("Skill installed: %s\n", skill_id);
    agentos_sys_free(skill_id);
}
```

#### 3.6.2 `agentos_sys_skill_execute`

**功能**: 执行 Skill。

**语法**:
```c
agentos_error_t agentos_sys_skill_execute(
    const char* skill_id,
    const char* input,
    char** out_output
);
```

**参数**:
- `skill_id`: Skill ID
- `input`: 输入数据
- `out_output`: 输出结果，调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_INVALID_PARAM`: 参数无效
- `AGENTOS_ERR_NOT_FOUND`: Skill 不存在
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* output = NULL;
if (agentos_sys_skill_execute(skill_id, "{\"a\":1,\"b\":2}", &output) == AGENTOS_OK) {
    printf("Skill output: %s\n", output);
    agentos_sys_free(output);
}
```

#### 3.6.3 `agentos_sys_skill_list`

**功能**: 列出所有已安装的 Skill。

**语法**:
```c
agentos_error_t agentos_sys_skill_list(char*** out_skills, size_t* out_count);
```

**参数**:
- `out_skills`: Skill ID 数组，调用者需使用 `agentos_sys_free()` 释放
- `out_count`: 输出 Skill 数量

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char** skills = NULL;
size_t count = 0;
if (agentos_sys_skill_list(&skills, &count) == AGENTOS_OK) {
    for (size_t i = 0; i < count; i++) {
        printf("Skill: %s\n", skills[i]);
    }
    agentos_sys_free(skills);
}
```

#### 3.6.4 `agentos_sys_skill_uninstall`

**功能**: 卸载 Skill。

**语法**:
```c
agentos_error_t agentos_sys_skill_uninstall(const char* skill_id);
```

**参数**:
- `skill_id`: Skill ID

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_NOT_FOUND`: Skill 不存在
- `AGENTOS_ERR_PERMISSION_DENIED`: 无权卸载

**示例**:
```c
if (agentos_sys_skill_uninstall(skill_id) == AGENTOS_OK) {
    printf("Skill uninstalled\n");
}
```

### 3.7 可观测性

#### 3.7.1 `agentos_sys_telemetry_metrics`

**功能**: 导出系统指标 (JSON 格式)。

**语法**:
```c
agentos_error_t agentos_sys_telemetry_metrics(char** out_metrics);
```

**参数**:
- `out_metrics`: 指标数据，调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* metrics = NULL;
if (agentos_sys_telemetry_metrics(&metrics) == AGENTOS_OK) {
    printf("System metrics: %s\n", metrics);
    agentos_sys_free(metrics);
}
```

**指标内容**:
```json
{
  "request_total": 12345,
  "request_duration_seconds": {
    "p50": 0.125,
    "p95": 0.456,
    "p99": 0.789
  },
  "llm_tokens_total": 987654,
  "task_completed_total": 5678,
  "agent_active_count": 42
}
```

#### 3.7.2 `agentos_sys_telemetry_traces`

**功能**: 导出追踪数据 (JSON 格式)。

**语法**:
```c
agentos_error_t agentos_sys_telemetry_traces(
    const char* trace_id,
    char** out_spans
);
```

**参数**:
- `trace_id`: 追踪 ID (为 NULL 时返回所有追踪)
- `out_spans`: 追踪数据，调用者需使用 `agentos_sys_free()` 释放

**返回值**:
- `AGENTOS_OK`: 成功
- `AGENTOS_ERR_OUT_OF_MEMORY`: 内存不足

**示例**:
```c
char* spans = NULL;
if (agentos_sys_telemetry_traces("trace-123", &spans) == AGENTOS_OK) {
    printf("Spans: %s\n", spans);
    agentos_sys_free(spans);
}
```

### 3.8 辅助函数

#### 3.8.1 `agentos_sys_free`

**功能**: 释放由系统调用分配的内存。

**语法**:
```c
void agentos_sys_free(void* ptr);
```

**参数**:
- `ptr`: 指向待释放内存的指针

**返回值**: 无

**说明**:
- 所有系统调用输出的动态分配内存，必须使用此函数释放
- 禁止使用标准库 `free()` 释放系统调用分配的内存
- 传入 NULL 是安全的，不会产生任何效果

**示例**:
```c
char* output = NULL;
agentos_sys_task_submit("task", 4, 30000, &output);
// 使用输出...
agentos_sys_free(output);  // 必须使用 agentos_sys_free 释放
```

---

## 第 4 章 错误处理

### 4.1 错误码定义

错误码定义在 [`agentos/commons/utils/error/include/error.h`](../../AgentRT/agentos/commons/utils/error/include/error.h) 中，详见 [第 7 章 错误码](#第 7 章 错误码)。

### 4.2 错误处理策略

#### 4.2.1 可恢复错误

对于可恢复的错误 (如超时、资源暂时耗尽),应重试：

```c
int retries = 3;
agentos_error_t err;
char* result = NULL;

while (retries-- > 0) {
    err = agentos_sys_task_submit(input, len, 30000, &result);
    if (err == AGENTOS_OK || err != AGENTOS_ERR_TIMEOUT) {
        break;  // 成功或非超时错误，退出重试
    }
    sleep(1);  // 等待 1 秒后重试
}

if (err != AGENTOS_OK) {
    fprintf(stderr, "Task failed after retries: %d\n", err);
}
```

#### 4.2.2 不可恢复错误

对于不可恢复的错误 (如参数无效、权限不足),应立即报错：

```c
agentos_error_t err = agentos_sys_task_submit(NULL, 0, 30000, &result);
if (err == AGENTOS_ERR_INVALID_PARAM) {
    fprintf(stderr, "Invalid parameters: %d\n", err);
    exit(1);  // 立即退出
}
```

#### 4.2.3 错误日志记录

所有错误应记录到日志系统：

```c
if (err != AGENTOS_OK) {
    AGENTOS_LOG_ERROR("Syscall failed: error=%d, function=%s", err, __func__);
}
```

---

## 第 5 章 性能考虑

### 5.1 性能基准

**典型性能指标** (参考值):

| 系统调用 | 平均耗时 | P95 耗时 | P99 耗时 |
|---------|---------|---------|---------|
| `task_submit` (简单任务) | 1.2s | 2.5s | 4.0s |
| `task_submit` (复杂任务) | 8.5s | 15.0s | 25.0s |
| `memory_search` | 50ms | 120ms | 200ms |
| `memory_write` | 10ms | 25ms | 40ms |
| `session_create` | 5ms | 10ms | 15ms |

### 5.2 性能优化建议

#### 5.2.1 批量操作

对于频繁调用，应批量处理以减少开销：

```c
// ❌ 低效：逐个写入
for (int i = 0; i < 100; i++) {
    agentos_sys_memory_write(data[i], len[i], NULL, 0);
}

// ✅ 高效：批量写入 (如果支持)
agentos_sys_memory_write_batch(datas, lens, count);
```

#### 5.2.2 缓存结果

对于重复查询，应缓存结果：

```c
// 使用本地缓存
static cache_t* search_cache = NULL;

char* cached_search(const char* query) {
    char* result = cache_get(search_cache, query);
    if (result) {
        return result;  // 命中缓存
    }
    
    // 未命中，调用系统调用
    char** record_ids = NULL;
    float* scores = NULL;
    size_t count = 0;
    agentos_sys_memory_search(query, 5, &record_ids, &scores, &count);
    if (count > 0) {
        result = strdup(record_ids[0]);
        cache_put(search_cache, query, result);
        agentos_sys_free(record_ids);
        agentos_sys_free(scores);
    }
    return result;
}
```

#### 5.2.3 异步处理

对于耗时操作，应使用异步模式：

```c
// 异步提交任务
const char* task_id = submit_task_async("...");

// 继续做其他事情
do_other_work();

// 稍后等待结果
char* result = NULL;
agentos_sys_task_wait(task_id, 30000, &result);
```

---

## 第 6 章 安全考虑

### 6.1 参数校验

所有系统调用内部都会进行严格的参数校验：

```c
// 伪代码示例
agentos_error_t agentos_sys_task_submit(const char* input, size_t input_len, ...) {
    // 1. 空指针检查
    if (!input || input_len == 0) {
        return AGENTOS_ERR_INVALID_PARAM;
    }
    
    // 2. 长度限制检查
    if (input_len > MAX_INPUT_SIZE) {
        return AGENTOS_ERR_INVALID_PARAM;
    }
    
    // 3. 权限检查
    if (!has_permission(TASK_SUBMIT)) {
        return AGENTOS_ERR_PERMISSION_DENIED;
    }
    
    // 4. 审计日志
    audit_log("task_submit", input_len);
    
    // ... 实际处理
}
```

### 6.2 权限控制

敏感操作需要相应权限：

| 系统调用 | 所需权限 |
|---------|---------|
| `task_submit` | `TASK_SUBMIT` |
| `task_cancel` | `TASK_CANCEL` (仅限任务所有者) |
| `memory_delete` | `MEMORY_DELETE` |
| `session_close` | `SESSION_CLOSE` (仅限会话所有者) |

### 6.3 审计日志

所有系统调用自动记录审计日志：

```json
{
  "timestamp": 1701234567.890,
  "level": "audit",
  "syscall": "task_submit",
  "trace_id": "abc123def456",
  "user_id": "alice",
  "input_size": 50,
  "duration_ms": 1234,
  "error_code": 0
}
```

---

## 第 7 章 错误码

> **错误码体系说明**: 本文档中使用的负整数错误码属于 Airymax **首要错误码体系**，适用于 C 内核、daemon 层和 atoms 模块。SDK 和外部接口应使用**次要体系**（十六进制分段错误码），详见 [error_code_reference.md](../project_erp/error_code_reference.md)。禁止在 C 内核代码中使用十六进制错误码，或在 SDK 中使用负整数错误码。

### 7.1 完整错误码列表

| 宏名 | 值 | 说明 | 常见原因 |
|------|-----|------|---------|
| `AGENTOS_OK` | 0 | 成功 | - |
| `AGENTOS_ERR_UNKNOWN` | -1 | 未知错误 | 未分类的内部错误 |
| `AGENTOS_ERR_INVALID_PARAM` | -2 | 无效参数 | 空指针、长度超限、格式错误 |
| `AGENTOS_ERR_OUT_OF_MEMORY` | -4 | 内存不足 | 系统内存耗尽 |
| `AGENTOS_ERR_NOT_FOUND` | -6 | 资源不存在 | ID 错误、资源已删除 |
| `AGENTOS_ERR_TIMEOUT` | -8 | 操作超时 | 超时时间设置过短 |
| `AGENTOS_ERR_PERMISSION_DENIED` | -10 | 权限不足 | 缺少权限 |
| `AGENTOS_ERR_BUSY` | -17 | 资源忙 | 资源被占用 |
| `AGENTOS_ERR_FAIL` | -31 | 内部错误 | 通用内部故障 |

### 7.2 错误码映射

系统调用错误码会映射到 JSON-RPC 错误码 (详见 [通信协议规范](./protocol_contract.md)):

```python
ERROR_MAP = {
    AGENTOS_ERR_INVALID_PARAM: -32602,      # Invalid params
    AGENTOS_ERR_NOT_FOUND: -32601,          # Method not found
    AGENTOS_ERR_PERMISSION_DENIED: -32003,  # Permission denied
    AGENTOS_ERR_OUT_OF_MEMORY: -32000,      # Server error
    AGENTOS_ERR_TIMEOUT: -32000,            # Server error
}
```

---

## 附录 A: 快速索引

### A.1 按功能分类

**初始化**: 3.1 节  
**任务管理**: 3.2 节  
**记忆管理**: 3.3 节  
**会话管理**: 3.4 节  
**Agent 管理**: 3.5 节  
**Skill 管理**: 3.6 节  
**可观测性**: 3.7 节  
**辅助函数**: 3.8 节  

### A.2 常用系统调用速查

| 功能 | 系统调用 | 参数 | 返回值 |
|------|---------|------|--------|
| 提交任务 | `task_submit` | input, timeout | output |
| 查询任务 | `task_query` | task_id | status |
| 搜索记忆 | `memory_search` | query, limit | record_ids, scores |
| 创建会话 | `session_create` | metadata | session_id |
| 生成 Agent | `agent_spawn` | agent_spec | agent_id |
| 安装 Skill | `skill_install` | skill_url | skill_id |
| 导出指标 | `telemetry_metrics` | - | metrics_json |

---

## 附录 B: 与其他规范的引用关系

| 引用规范 | 关系说明 |
|---------|---------|
| [通信协议规范](./protocol_contract.md) | 本规范定义系统调用的 API 接口，protocol_contract.md 定义了传输协议 |
| [架构设计原则](../../../Capital_Architecture/ARCHITECTURAL_PRINCIPLES.md) | 本规范是架构原则在系统调用层面的具体实现 |
| [日志格式规范](./logging_format.md) | 系统调用的日志记录应遵循统一的日志格式 |
| [日志打印规范](../coding_standard/Log_standard.md) | 错误处理和审计日志的记录规范 |
| [统一术语表](../TERMINOLOGY.md) | 本规范使用的术语定义和解释 |

---

## 参考文献

[1] Airymax 设计哲学。../../Basic_Theories/CN_04_系统设计原则.md  
[2] 架构设计原则。../../../Capital_Architecture/ARCHITECTURAL_PRINCIPLES.md  
[3] 统一术语表。../TERMINOLOGY.md  
[4] Microkernel Architecture. https://en.wikipedia.org/wiki/Microkernel  
[5] POSIX System Call Interface. https://pubs.opengroup.org/onlinepubs/9699919799/  

---

## 版本历史

| 版本 | 日期 | 作者 | 变更说明 |
|------|------|------|---------|
| Doc V2.0 | 2026-03-25 | Airymax Team | 根据架构设计原则V1.6进行全面优化，更新理论基础，重构五维正交体系映射，补充可测试性原则应用 |
| Doc V2.0 | 2026-03-24 | Airymax Team | 增加与设计哲学的关系章节，优化表述结构 |
| Doc V2.0 | 2026-03-24 | Airymax Team | 按文档格式规范重新编写 |
| Doc V2.0 | 2026-03-23 | Airymax Team | 基于项目实际架构全面重构 |
| Doc V2.0 | 2026-03-21 | Airymax Team | 基于系统工程理论重构 |
| Doc V2.0 | 2026-02-01 | Airymax Team | 原始系统调用规范 |

---

**最后更新**: 2026-04-09  
**维护者**: Airymax 架构委员会
