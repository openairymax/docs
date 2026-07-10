# CoreLoopThree API - 三层认知循环  

**最新**: 2026-06-09  
**状态**: 维护中  
**路径**: OpenAirymax/Docs/30-api/20-core/01-coreloop-api.md  
---

## 📖 概述

CoreLoopThree 是 Airymax 的核心运行时，实现了**认知-执行-记忆**三层架构：

```
┌─────────────────────────────────────┐
│         用户输入 (User Input)        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│    🧠 认知层 (Cognition Engine)      │
│  ┌─────────┬─────────┬──────────┐   │
│  │ 意图解析 │ 规划生成 │ 元认知   │   │
│  └─────────┴─────────┴──────────┘   │
│  Phase 0-4: 拆解→规划→流式→审计→对齐│
└──────────────┬──────────────────────┘
               │ 任务计划 (Task Plan)
               ▼
┌─────────────────────────────────────┐
│    ⚡ 执行层 (Execution Engine)      │
│  ┌─────────┬─────────┬──────────┐   │
│  │ 工作线程 │ 任务队列 │ 引用计数 │   │
│  └─────────┴─────────┴──────────┘   │
│  并发执行、超时管理、结果收集       │
└──────────────┬──────────────────────┘
               │ 执行结果
               ▼
┌─────────────────────────────────────┐
│    📚 记忆层 (Memory Engine)         │
│  L1:原始卷 → L2:特征 → L3:结构 → L4:模式│
│  写入、检索、演化、遗忘              │
└─────────────────────────────────────┘
```

---

## 🔧 主循环 API

### 数据结构

#### `agentrt_core_loop_t`
三层认知循环的主句柄（不透明指针）。

#### `agentrt_loop_config_t`
```c
typedef struct {
    uint32_t loop_config_cognition_threads;    /* 认知线程数 (默认4) */
    uint32_t loop_config_execution_threads;     /* 执行线程数 (默认8) */
    uint32_t loop_config_memory_threads;        /* 记忆线程数 (默认2) */
    uint32_t loop_config_max_queued_tasks;      /* 最大队列任务数 (默认1000) */
    uint32_t loop_config_stats_interval_ms;     /* 统计间隔 (默认60000ms) */
    uint32_t loop_config_memory_query_limit;    /* 记忆查询限制 (默认5) */
    uint32_t loop_config_task_timeout_ms;       /* 任务超时 (默认30000ms) */
    float    loop_config_memory_importance;     /* 记忆重要性权重 (默认0.7) */
    
    /* 策略配置 */
    void*    loop_config_plan_strategy;         /* 规划策略 (可选) */
    void*    loop_config_coord_strategy;         /* 协调策略 (可选) */
    void*    loop_config_disp_strategy;          /* 分发策略 (可选) */
} agentrt_loop_config_t;
```

### 函数接口

#### `agentrt_loop_create`
创建三层认知循环实例。

```c
AGENTRT_API agentrt_error_t agentrt_loop_create(
    const agentrt_loop_config_t* config,   /* [in] 配置参数，NULL使用默认值 */
    agentrt_core_loop_t** out_loop        /* [out] 输出循环句柄 */
);
```

**返回值**:
- `AGENTRT_OK`: 成功创建
- `AGENTRT_EINVAL`: 参数无效
- `AGENTRT_ENOMEM`: 内存不足

**示例**:
```c
agentrt_core_loop_t* loop = NULL;
agentrt_error_t err = agentrt_loop_create(NULL, &loop);
if (err != AGENTRT_OK) {
    // 处理错误
}
```

---

#### `agentrt_loop_destroy`
销毁循环实例并释放所有资源。

```c
AGENTRT_API void agentrt_loop_destroy(
    agentrt_core_loop_t* loop  /* [in] 循环句柄 */
);
```

**注意**: 如果循环正在运行，会先调用 `agentrt_loop_stop`。

---

#### `agentrt_loop_run`
启动主事件循环（阻塞当前线程）。

```c
AGENTRT_API agentrt_error_t agentrt_loop_run(
    agentrt_core_loop_t* loop  /* [in] 循环句柄 */
);
```

**行为**:
1. 设置 `running = 1`, `stop_requested = 0`
2. 进入主循环，等待任务或停止信号
3. 每50ms检查一次条件变量
4. 收到停止信号后退出

**线程安全**: 此函数应在独立线程中调用，或使用 `agentrt_loop_submit` 提交任务后在另一个线程中等待。

---

#### `agentrt_loop_stop`
请求停止运行中的循环。

```c
AGENTRT_API void agentrt_loop_stop(
    agentrt_core_loop_t* loop  /* [in] 循环句柄 */
);
```

**行为**:
1. 设置 `stop_requested = 1`
2. 广播条件变量唤醒主循环
3. 阻塞等待 `running` 变为0

**阻塞**: 直到循环完全停止才返回。

---

#### `agentrt_loop_submit`
提交用户输入到循环处理。

```c
AGENTRT_API agentrt_error_t agentrt_loop_submit(
    agentrt_core_loop_t* loop,      /* [in] 循环句柄 */
    const char* input,              /* [in] 用户输入文本 */
    size_t input_len,               /* [in] 输入长度 */
    char** out_task_id              /* [out] 任务ID (需调用者释放) */
);
```

**处理流程**:
```
① 记忆查询 → 获取相关上下文
② 构建增强输入 → 拼接记忆 + 原始输入
③ 认知处理 → 5阶段管线生成计划
④ 执行提交 → 将计划节点转为任务并提交
⑤ 信号通知 → 唤醒工作线程
```

**返回值**:
- `AGENTRT_OK`: 任务已提交
- `AGENTRT_EINVAL`: 参数无效或引擎未初始化
- `AGENTRT_ENOMEM`: 内存不足

**示例**:
```c
char* task_id = NULL;
err = agentrt_loop_submit(loop, "你好", 2, &task_id);
if (err == AGENTRT_OK) {
    printf("任务ID: %s\n", task_id);
    // 后续使用 task_id 等待结果
}
// 使用后释放
AGENTRT_FREE(task_id);
```

---

#### `agentrt_loop_wait`
等待指定任务完成并获取结果。

```c
AGENTRT_API agentrt_error_t agentrt_loop_wait(
    agentrt_core_loop_t* loop,      /* [in] 循环句柄 */
    const char* task_id,            /* [in] 任务ID */
    uint32_t timeout_ms,            /* [in] 超时时间(毫秒), 0=无限等待 */
    char** out_result,              /* [out] 结果字符串 (需调用者释放) */
    size_t* out_result_len          /* [out] 结果长度 */
);
```

**返回值**:
- `AGENTRT_OK`: 任务完成，结果可用
- `AGENTRT_ETIMEDOUT`: 超时
- `AGENTRT_EINVAL**: 参数无效
- `AGENTRT_ENOENT`: 任务不存在
- `AGENTRT_ENOMEM`: 内存不足

**副作用**: 成功后会自动将结果写入记忆系统。

**示例**:
```c
char* result = NULL;
size_t result_len = 0;
err = agentrt_loop_wait(loop, task_id, 30000, &result, &result_len);
if (err == AGENTRT_OK && result) {
    printf("响应: %.*s\n", (int)result_len, result);
    AGENTRT_FREE(result);
}
```

---

#### `agentrt_loop_get_engines`
获取三个引擎的句柄（用于高级用法）。

```c
AGENTRT_API void agentrt_loop_get_engines(
    agentrt_core_loop_t* loop,           /* [in] 循环句柄 */
    agentrt_cognition_engine_t** out_cognition,  /* [out] 认知引擎 */
    agentrt_execution_engine_t** out_execution,  /* [out] 执行引擎 */
    agentrt_memory_engine_t** out_memory        /* [out] 记忆引擎 */
);
```

**用途**: 当需要直接操作某个引擎时使用（如自定义策略注入）。

---

## 🧠 认知引擎 API

### 数据结构

#### `agentrt_cognition_engine_t`
认知引擎句柄（不透明指针）。

#### `agentrt_cognition_config_t`
```c
typedef struct {
    uint32_t cognition_default_timeout_ms;   /* 默认超时 (30000ms) */
    uint32_t cognition_max_retries;          /* 最大重试次数 (3) */
    
    /* 回调函数 */
    agentrt_feedback_callback_t feedback_callback;
    void* feedback_user_data;
} agentrt_cognition_config_t;
```

#### `agentrt_task_plan_t`
任务计划（DAG结构）。
```c
typedef struct {
    char* task_plan_id;                    /* 计划唯一ID */
    agentrt_task_node_t** task_plan_nodes; /* 计划节点数组 */
    size_t task_plan_node_count;           /* 节点数量 */
    char** task_plan_entry_points;         /* 入口点列表 */
} agentrt_task_plan_t;
```

#### `agentrt_task_node_t`
计划节点。
```c
typedef struct {
    char* task_node_id;                   /* 节点ID */
    char* task_node_agent_role;           /* 需要的Agent角色 */
    size_t task_node_role_len;            /* 角色长度 */
    void* task_node_input;                /* 输入数据 */
    char** task_node_depends_on;          /* 依赖的节点ID列表 */
    size_t task_node_depends_count;       /* 依赖数量 */
    uint32_t task_node_timeout_ms;        /* 超时时间 */
    uint8_t task_node_priority;           /* 优先级 (0-255) */
} agentrt_task_node_t;
```

### 函数接口

#### `agentrt_cognition_create`
创建认知引擎（基础版本）。

```c
agentrt_error_t agentrt_cognition_create(
    agentrt_plan_strategy_t* plan_strategy,      /* [in] 规划策略 (可选) */
    agentrt_coordinator_strategy_t* coord_strategy, /* [in] 协调策略 (可选) */
    agentrt_dispatching_strategy_t* disp_strategy, /* [in] 分发策略 (可选) */
    agentrt_cognition_engine_t** out_engine       /* [out] 输出引擎 */
);
```

---

#### `agentrt_cognition_create_ex`
创建认知引擎（扩展版本，支持完整配置）。

```c
agentrt_error_t agentrt_cognition_create_ex(
    const agentrt_cognition_config_t* config,    /* [in] 完整配置 (可选) */
    agentrt_plan_strategy_t* plan_strategy,
    agentrt_coordinator_strategy_t* coord_strategy,
    agentrt_dispatching_strategy_t* disp_strategy,
    agentrt_cognition_engine_t** out_engine
);
```

**额外功能**:
- 支持双思考系统（Thinking Chain + Metacognition）
- 可配置元认知阈值和链容量
- 反馈回调机制

---

#### `agentrt_cognition_destroy`
销毁认知引擎。
```c
void agentrt_cognition_destroy(agentrt_cognition_engine_t* engine);
```

---

#### `agentrt_cognition_process`
核心处理函数：将输入转换为任务计划。

```c
agentrt_error_t agentrt_cognition_process(
    agentrt_cognition_engine_t* engine,   /* [in] 引擎句柄 */
    const char* input,                   /* [in] 输入文本 */
    size_t input_len,                    /* [in] 输入长度 */
    agentrt_task_plan_t** out_plan       /* [out] 输出计划 (需调用者释放) */
);
```

**五阶段处理管线**:

| 阶段 | 名称 | 功能 |
|------|------|------|
| **Phase 0** | 指令拆解 (S1) | 使用Thinking Chain分解复杂指令 |
| **Phase 1** | 规划 (S2+S1) | 生成执行计划DAG，支持fallback |
| **Phase 2** | 流式关键循环 | Triple Coordinator协调S2/S1/t1 |
| **Phase 3** | 子任务审计 | S1+专家S1验证每个步骤 |
| **Phase 4** | 目标对齐 | 检测漂移，必要时重新规划 |

**双思考系统集成**:
- **Thinking Chain**: 显式推理链，支持步骤追踪
- **Metacognition**: 元认知评估，自动纠错

**返回值**:
- `AGENTRT_OK`: 计划生成成功
- `AGENTRT_EINVAL`: 参数无效
- `AGENTRT_ENOTSUP`: 无可用规划策略

**示例**:
```c
agentrt_task_plan_t* plan = NULL;
err = agentrt_cognition_process(engine, "帮我写一份报告", strlen("帮我写一份报告"), &plan);

if (err == AGENTRT_OK && plan) {
    printf("计划ID: %s\n", plan->task_plan_id);
    printf("节点数: %zu\n", plan->task_plan_node_count);
    
    // 遍历计划节点
    for (size_t i = 0; i < plan->task_plan_node_count; i++) {
        agentrt_task_node_t* node = plan->task_plan_nodes[i];
        printf("  [%zu] 角色: %s\n", i, node->task_node_agent_role);
    }
    
    // 释放计划
    agentrt_task_plan_free(plan);
}
```

---

#### `agentrt_task_plan_free`
释放任务计划及其所有子资源。

```c
void agentrt_task_plan_free(agentrt_task_plan_t* plan);
```

**释放内容**:
- 所有节点的ID、角色、依赖列表
- 节点数组本身
- 计划ID和入口点列表

---

#### `agentrt_cognition_stats`
获取认知引擎统计信息。

```c
agentrt_error_t agentrt_cognition_stats(
    agentrt_cognition_engine_t* engine,
    char** out_stats,      /* [out] JSON格式统计数据 (需释放) */
    size_t* out_len        /* [out] 数据长度 */
);
```

**输出JSON示例**:
```json
{
    "processed": 42,
    "avg_time_ns": 15200000,
    "dual_think_invocations": 38,
    "dual_think_corrections": 5
}
```

---

#### `agentrt_cognition_health_check`
健康检查（JSON格式）。

```c
agentrt_error_t agentrt_cognition_health_check(
    agentrt_cognition_engine_t* engine,
    char** out_json        /* [out] JSON格式健康状态 (需释放) */
);
```

---

### 高级配置API

#### `agentrt_cognition_set_fallback_plan`
设置备用规划策略。

```c
void agentrt_cognition_set_fallback_plan(
    agentrt_cognition_engine_t* engine,
    agentrt_plan_strategy_t* fallback
);
```

**用途**: 主策略失败时自动切换到备用策略。

---

#### `agentrt_cognition_set_context`
设置认知上下文（如对话历史）。

```c
void agentrt_cognition_set_context(
    agentrt_cognition_engine_t* engine,
    void* context,
    void (*destroy)(void*)  /* 上下文销毁函数 */
);
```

---

#### `agentrt_cognition_set_memory`
关联记忆引擎（用于上下文预填充）。

```c
void agentrt_cognition_set_memory(
    agentrt_cognition_engine_t* engine,
    agentrt_memory_engine_t* memory
);
```

---

## ⚡ 执行引擎 API

### 数据结构

#### `agentrt_execution_engine_t`
执行引擎句柄（不透明指针）。

#### `agentrt_task_t`
任务描述。
```c
typedef struct {
    char* task_id;             /* 任务ID (由引擎分配) */
    char* task_agent_id;       /* 目标Agent/执行单元ID */
    void* task_input;          /* 输入数据 */
    void* task_output;         /* 输出数据 (执行完成后填充) */
    size_t task_input_size;    /* 输入大小 */
    size_t task_output_size;   /* 输出大小 */
    uint64_t task_timeout_ms;  /* 超时时间 */
    uint32_t task_priority;    /* 优先级 */
    agentrt_task_status_t task_status; /* 当前状态 */
} agentrt_task_t;
```

#### `agentrt_task_status_t`
任务状态枚举。
```c
typedef enum {
    TASK_STATUS_PENDING = 0,   /* 等待中 */
    TASK_STATUS_RUNNING = 1,   /* 执行中 */
    TASK_STATUS_SUCCEEDED = 2,/* 成功完成 */
    TASK_STATUS_FAILED = 3,    /* 执行失败 */
    TASK_STATUS_CANCELLED = 4, /* 已取消 */
    TASK_STATUS_TIMEOUT = 5    /* 超时 */
} agentrt_task_status_t;
```

#### `agentrt_execution_unit_t`
执行单元（注册到引擎的工作器）。
```c
typedef struct {
    char* unit_name;                          /* 单元名称 */
    agentrt_error_t (*execution_unit_execute)( /* 执行函数 */
        struct agentrt_execution_unit_s* self,
        const void* input,
        void** output
    );
    void* context;                             /* 用户上下文 */
    void (*destroy)(struct agentrt_execution_unit_s*); /* 销毁函数 */
} agentrt_execution_unit_t;
```

### 函数接口

#### `agentrt_execution_create`
创建执行引擎。

```c
agentrt_error_t agentrt_execution_create(
    uint32_t max_concurrency,          /* [in] 最大并发任务数 */
    agentrt_execution_engine_t** out_engine /* [out] 输出引擎 */
);
```

**内部初始化**:
1. 创建任务队列互斥锁和条件变量
2. 创建运行任务锁
3. 创建哈希表（大小 = max_concurrency * 2）
4. 创建工作线程池（max_concurrency个线程）

**线程模型**:
```
主线程 → 提交任务到队列 → signal(available_cond)
                                    ↓
工作线程池 ← wait(available_cond) ← 从队列取任务
    ↓
执行任务 → 更新状态 → signal(completed_cond)
    ↓
从哈希表移除 → release TCB
```

---

#### `agentrt_execution_destroy`
销毁执行引擎。

```c
void agentrt_execution_destroy(agentrt_execution_engine_t* engine);
```

**清理顺序**:
1. 设置 running = 0
2. broadcast 唤醒所有工作线程
3. join 等待所有线程结束
4. 清理队列中剩余的任务
5. 清理运行中的任务
6. 销毁同步原语和哈希表

---

#### `agentrt_execution_submit`
提交任务到执行引擎。

```c
agentrt_error_t agentrt_execution_submit(
    agentrt_execution_engine_t* engine,  /* [in] 引擎 */
    const agentrt_task_t* task,          /* [in] 任务描述 */
    char** out_task_id                   /* [out] 生成的任务ID (需释放) */
);
```

**处理流程**:
1. 深拷贝任务描述（防止外部修改）
2. 创建TCB（Task Control Block）
3. 生成唯一任务ID（使用 `agentrt_generate_task_id`）
4. 初始化同步原语（mutex, cond）
5. 入队并插入哈希表
6. signal 通知工作线程

**返回的任务ID**: 使用 `AGENTRT_FREE` 释放。

---

#### `agentrt_execution_query`
查询任务状态（非阻塞）。

```c
agentrt_error_t agentrt_execution_query(
    agentrt_execution_engine_t* engine,
    const char* task_id,
    agentrt_task_status_t* out_status  /* [out] 当前状态 */
);
```

**时间复杂度**: O(1) - 哈希表查找

---

#### `agentrt_execution_wait`
等待任务完成（阻塞）。

```c
agentrt_error_t agentrt_execution_wait(
    agentrt_execution_engine_t* engine,
    const char* task_id,
    uint32_t timeout_ms,            /* [in] 超时(ms), 0=无限 */
    agentrt_task_t** out_result    /* [out] 结果副本 (需释放) */
);
```

**行为**:
1. 在哈希表中查找任务
2. 增加引用计数（防止等待期间被释放）
3. 等待 completed_cond 信号
4. 超时检查
5. 深拷贝结果（如果需要）

**结果释放**: 使用 `agentrt_task_free`。

---

#### `agentrt_execution_cancel`
取消等待中的任务。

```c
agentrt_error_t agentrt_execution_cancel(
    agentrt_execution_engine_t* engine,
    const char* task_id
);
```

**只能取消**: PENDING状态的任务（在队列中但未开始执行）。

---

#### `agentrt_execution_get_result`
获取已完成任务的结果（非阻塞）。

```c
agentrt_error_t agentrt_execution_get_result(
    agentrt_execution_engine_t* engine,
    const char* task_id,
    agentrt_task_t** out_result  /* [out] 结果 (需释放) */
);
```

**前置条件**: 任务必须处于 SUCCEEDED 或 FAILED 状态。

---

#### `agentrt_task_free`
释放任务结构体。

```c
void agentrt_task_free(agentrt_task_t* task);
```

---

#### `agentrt_execution_health_check`
健康检查（JSON格式）。

```c
agentrt_error_t agentrt_execution_health_check(
    agentrt_execution_engine_t* engine,
    char** out_json  /* [out] JSON统计信息 */
);
```

**输出示例**:
```json
{
    "status": "healthy",
    "task_queue_length": 5,
    "running_tasks": 3,
    "current_concurrency": 3,
    "max_concurrency": 8,
    "running": 1
}
```

---

## 📚 记忆引擎 API

### 数据结构

#### `agentrt_memory_engine_t`
记忆引擎句柄（不透明指针）。

#### `agentrt_memory_record_t`
记忆记录。
```c
typedef struct {
    char* memory_record_data;        /* 数据内容 */
    size_t memory_record_data_len;    /* 数据长度 */
    char* memory_record_type;         /* 类型标识 */
    float memory_record_importance;   /* 重要性 (0.0-1.0) */
    uint64_t memory_record_timestamp; /* 时间戳 */
    uint32_t memory_record_access_count; /* 访问次数 */
} agentrt_memory_record_t;
```

#### `agentrt_memory_query_t`
查询参数。
```c
typedef struct {
    char* memory_query_text;          /* 查询文本 */
    size_t memory_query_text_len;     /* 文本长度 */
    uint32_t memory_query_limit;      /* 返回数量限制 */
    uint8_t memory_query_include_raw;  /* 是否包含原始数据 */
    float memory_query_min_similarity;/* 最小相似度阈值 */
} agentrt_memory_query_t;
```

#### `agentrt_memory_result_ext_t`
查询结果。
```c
typedef struct {
    agentrt_memory_result_item_t** memory_result_items; /* 结果项数组 */
    size_t memory_result_count;                         /* 结果数量 */
    uint64_t memory_result_query_time_ns;               /* 查询耗时 */
} agentrt_memory_result_ext_t;
```

### 函数接口

#### `agentrt_memory_create`
创建记忆引擎。

```c
agentrt_error_t agentrt_memory_create(
    const agentrt_memory_config_t* config,  /* [in] 配置 (可选, NULL用默认值) */
    agentrt_memory_engine_t** out_memory    /* [out] 输出引擎 */
);
```

---

#### `agentrt_memory_destroy`
销毁记忆引擎。
```c
void agentrt_memory_destroy(agentrt_memory_engine_t* memory);
```

---

#### `agentrt_memory_write`
写入一条记忆记录。

```c
agentrt_error_t agentrt_memory_write(
    agentrt_memory_engine_t* memory,
    const agentrt_memory_record_t* record,  /* [in] 记录数据 */
    char** out_record_id                    /* [out] 生成的记录ID (需释放) */
);
```

**处理流程**:
1. L1: 写入原始数据（异步）
2. L2: 特征提取 + 向量化
3. L3: 实体抽取 + 关系构建
4. 返回记录ID

**示例**:
```c
agentrt_memory_record_t record = {
    .memory_record_data = "用户喜欢Python",
    .memory_record_data_len = strlen("用户喜欢Python"),
    .memory_record_type = "preference",
    .memory_record_importance = 0.8f
};

char* record_id = NULL;
err = agentrt_memory_write(memory, &record, &record_id);
printf("记录ID: %s\n", record_id);
AGENTRT_FREE(record_id);
```

---

#### `agentrt_memory_query`
语义相似性检索。

```c
agentrt_error_t agentrt_memory_query(
    agentrt_memory_engine_t* memory,
    const agentrt_memory_query_t* query,      /* [in] 查询参数 */
    agentrt_memory_result_ext_t** out_result  /* [out] 结果 (需释放) */
);
```

**检索流程**:
1. 向量化查询文本
2. L2: 近似最近邻搜索（HNSW/IVF）
3. 按相似度排序
4. 截断到 limit 数量
5. 可选：包含原始数据

**结果释放**: 使用 `agentrt_memory_result_free`。

---

#### `agentrt_memory_read`
按ID读取记录。

```c
agentrt_error_t agentrt_memory_read(
    agentrt_memory_engine_t* memory,
    const char* record_id,
    agentrt_memory_record_t** out_record  /* [out] 记录 (需释放) */
);
```

---

#### `agentrt_memory_delete`
删除记录。

```c
agentrt_error_t agentrt_memory_delete(
    agentrt_memory_engine_t* memory,
    const char* record_id
);
```

---

#### `agentrt_memory_result_free`
释放查询结果。

```c
void agentrt_memory_result_free(agentrt_memory_result_ext_t* result);
```

---

## 🎯 完整使用示例

### 示例1: 简单问答Agent

```c
#include "agentrt.h"
#include "loop.h"

int main() {
    // 1. 初始化
    agentrt_core_init();
    
    // 2. 创建循环（使用默认配置）
    agentrt_core_loop_t* loop = NULL;
    agentrt_loop_create(NULL, &loop);
    
    // 3. 启动后台线程
    // 注意：实际应用中应该在独立线程中调用 run
    // 这里为了演示，我们直接 submit + wait
    
    // 4. 提交问题
    char* task_id = NULL;
    const char* question = "什么是量子计算？";
    agentrt_loop_submit(loop, question, strlen(question), &task_id);
    
    // 5. 等待回答（30秒超时）
    char* answer = NULL;
    size_t answer_len = 0;
    agentrt_error_t err = agentrt_loop_wait(loop, task_id, 30000, &answer, &answer_len);
    
    if (err == AGENTRT_OK) {
        printf("Q: %s\n", question);
        printf("A: %.*s\n", (int)answer_len, answer);
        AGENTRT_FREE(answer);
    } else {
        printf("错误: %d\n", err);
    }
    
    // 6. 清理
    AGENTRT_FREE(task_id);
    agentrt_loop_destroy(loop);
    agentrt_core_shutdown();
    
    return 0;
}
```

### 示例2: 自定义执行单元

```c
#include "execution.h"
#include "agent_registry.h"

// 自定义执行单元：文本分析器
static agentrt_error_t text_analyzer_execute(
    agentrt_execution_unit_t* unit,
    const void* input,
    void** output) {
    
    const char* text = (const char*)input;
    
    // 简单的分析逻辑：计算词数
    int word_count = 0;
    int in_word = 0;
    for (; *text; text++) {
        if (isspace(*text)) {
            in_word = 0;
        } else if (!in_word) {
            in_word = 1;
            word_count++;
        }
    }
    
    // 格式化结果
    char* result = NULL;
    asprintf(&result, "{\"word_count\":%d}", word_count);
    *output = result;
    
    return AGENTRT_OK;
}

// 注册执行单元
void register_text_analyzer() {
    agentrt_execution_unit_t* unit = AGENTRT_CALLOC(1, sizeof(*unit));
    unit->unit_name = "text_analyzer";
    unit->execution_unit_execute = text_analyzer_execute;
    unit->context = NULL;
    unit->destroy = free;
    
    agentrt_registry_register_unit(unit);
}
```

---

## ⚠️ 重要注意事项

### 线程安全
- ✅ 所有 `_t` 句柄类型都是线程安全的
- ✅ 可以从多个线程同时调用 `submit` 和 `wait`
- ⚠️ 不要在回调函数中调用可能阻塞的API

### 内存管理
- ✅ 使用 `AGENTRT_*` 系列宏进行内存操作
- ✅ 文档中标明"需调用者释放"的返回值必须手动释放
- ❌ 不要混用 `malloc/free` 和 `AGENTRT_MALLOC/AGENTRT_FREE`

### 性能建议
- **高并发场景**: 增加 `execution_threads` 数量（建议 CPU核心数 * 2）
- **内存敏感场景**: 减少 `memory_query_limit`（默认5）
- **低延迟场景**: 减少 `task_timeout_ms` 并设置合理的重试策略

### 错误处理最佳实践
```c
agentrt_error_t err = agentrt_loop_submit(loop, input, len, &task_id);
switch (err) {
    case AGENTRT_OK:
        // 正常处理
        break;
    case AGENTRT_EINVAL:
        fprintf(stderr, "无效参数\n");
        break;
    case AGENTRT_ENOMEM:
        fprintf(stderr, "内存不足\n");
        break;
    default:
        fprintf(stderr, "未知错误: %d\n", err);
}
```

---

## 📊 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v0.0.4 | 2026-04-28 | Foundation Release - 完整API文档 |
| v0.0.3 | 2026-04-25 | 双思考系统集成 |
| v0.0.2 | 2026-04-20 | 三层架构实现 |
| v0.0.1 | 2026-04-15 | 初始版本 |

---

## 🔗 相关文档

- [MemoryRovol 详细API（待编写）](../core/memoryrovol_api.md)
- [用户态服务集成指南](../daemon/gateway_api.md)
- [协议栈适配（待编写）](../protocols/unified_protocol_api.md)
- [快速入门示例](../examples/quickstart.md)
