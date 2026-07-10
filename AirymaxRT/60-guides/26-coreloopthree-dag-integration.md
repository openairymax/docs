# CoreLoopThree DAG 工作流集成指南

**版本**: 0.1.1 (W18)
**状态**: 已实现
**路径**: OpenAirymax/Docs/60-guides/26-coreloopthree-dag-integration.md
**版权**: (c) 2026 SPHARX Ltd. All Rights Reserved.

## 概述

CoreLoopThree（三层认知循环）在 0.1.1 版本中集成了 taskflow_advanced DAG 工作流引擎，提供结构化的复杂工作流编排能力。与 `agentrt_loop_submit()`（自然语言单任务提交）不同，DAG API 接受结构化的 `taskflow_workflow_t` 工作流定义，支持条件分支、并行汇聚、循环迭代等 DAG 模式。

### 架构关系

```
                    ┌─────────────────────────────────┐
                    │       CoreLoopThree (loop.c)     │
                    │                                  │
  自然语言任务 ────►│  agentrt_loop_submit()           │
                    │  (reactive/reflective planner)   │
                    │                                  │
  结构化 DAG ─────►│  agentrt_loop_submit_dag()       │
                    │  (taskflow_advanced 引擎)        │
                    │                                  │
                    │  ┌─────────────────────────┐    │
                    │  │  taskflow_engine_t      │    │
                    │  │  (内部持有，create/destroy)│   │
                    │  └─────────────────────────┘    │
                    └─────────────────────────────────┘
```

### 设计要点

- CoreLoopThree 在 `agentrt_loop_create()` 时内部创建 `taskflow_engine_t*`，在 `agentrt_loop_destroy()` 时销毁
- 4 个 DAG API 暴露在 `loop.h` 中，通过 `taskflow_advanced.h` 类型定义工作流
- `taskflow_engine_get_execution()` 返回内部指针（非副本），不可调用 `taskflow_execution_destroy()`

---

## 一、API 参考

### 1.1 agentrt_loop_submit_dag

```c
agentrt_error_t agentrt_loop_submit_dag(agentrt_core_loop_t *loop,
                                         const taskflow_workflow_t *workflow,
                                         const char *input_json,
                                         char **out_execution_id);
```

提交 DAG 工作流任务。内部调用 `taskflow_engine_register_workflow()` 注册工作流定义，然后调用 `taskflow_engine_start()` 启动执行。

| 参数 | 方向 | 说明 |
|------|------|------|
| `loop` | in | 循环句柄（非NULL） |
| `workflow` | in | 工作流定义（非NULL） |
| `input_json` | in | 输入数据（JSON字符串，可为NULL） |
| `out_execution_id` | out | 执行实例ID（调用者负责释放） |

**返回**: `AGENTRT_SUCCESS` 成功，`AGENTRT_EINVAL` 参数错误，`AGENTRT_ENOTINIT` 引擎未初始化

### 1.2 agentrt_loop_dag_register_handler

```c
agentrt_error_t agentrt_loop_dag_register_handler(
    agentrt_core_loop_t *loop, const char *name,
    taskflow_task_handler_t handler, void *user_data);
```

注册 DAG 任务处理器。工作流节点的 `task_handler_name` 与此处的 `name` 匹配以路由执行。

### 1.3 agentrt_loop_dag_wait

```c
agentrt_error_t agentrt_loop_dag_wait(agentrt_core_loop_t *loop,
                                       const char *execution_id,
                                       uint32_t timeout_ms,
                                       char **out_result_json);
```

等待 DAG 工作流执行完成并获取结果。先检查执行状态，已完成（COMPLETED/FAILED/CANCELED）则直接返回结果；仍在执行中则调用 `taskflow_engine_run_to_completion()` 阻塞等待。

**注意**: `timeout_ms` 参数在 0.1.1 版本中未实现（`run_to_completion` 不支持超时），0 表示无限等待。

### 1.4 agentrt_loop_dag_status

```c
agentrt_error_t agentrt_loop_dag_status(agentrt_core_loop_t *loop,
                                         const char *execution_id,
                                         char **out_state,
                                         double *out_progress);
```

获取 DAG 工作流执行状态。`out_state` 返回状态字符串（"pending"/"running"/"completed"/"failed"/"canceled" 等），`out_progress` 返回 0.0~1.0 进度。

---

## 二、taskflow_task_handler_t 回调签名

```c
typedef int (*taskflow_task_handler_t)(taskflow_engine_t *engine,
                                        const char *node_id,
                                        const char *input_json,
                                        char **output_json,
                                        void *user_data);
```

| 参数 | 说明 |
|------|------|
| `engine` | taskflow 引擎句柄 |
| `node_id` | 当前节点ID |
| `input_json` | 输入数据（JSON字符串） |
| `output_json` | 输出数据（调用者负责释放，引擎在 destroy 时释放） |
| `user_data` | 注册时传入的用户数据 |

**返回**: 0 表示成功，非 0 表示失败（引擎将执行状态置为 FAILED）

---

## 三、完整使用示例

### 3.1 基础 DAG 提交

```c
#include "loop.h"
#include "taskflow_advanced.h"
#include "memory_compat.h"

/* 1. 定义任务处理器 */
static int my_handler(taskflow_engine_t *engine, const char *node_id,
                      const char *input_json, char **output_json, void *user_data)
{
    (void)engine; (void)user_data;
    /* 业务逻辑：处理 input_json，生成 output */
    char buf[256];
    snprintf(buf, sizeof(buf), "{\"node\":\"%s\",\"result\":\"processed\"}",
             node_id ? node_id : "");
    *output_json = AGENTRT_STRDUP(buf);
    return 0;  /* 0 = 成功 */
}

/* 2. 创建 loop 并提交 DAG */
void dag_example(void)
{
    /* 创建 CoreLoopThree 实例 */
    agentrt_loop_config_t config;
    AGENTRT_MEMSET(&config, 0, sizeof(config));
    config.loop_config_cognition_threads = 4;
    config.loop_config_execution_threads = 8;
    config.loop_config_memory_threads = 2;
    config.loop_config_max_queued_tasks = 100;

    agentrt_core_loop_t *loop = NULL;
    agentrt_error_t err = agentrt_loop_create(&config, &loop);
    if (err != AGENTRT_SUCCESS || !loop) return;

    /* 注册任务处理器 */
    err = agentrt_loop_dag_register_handler(loop, "process_data", my_handler, NULL);
    if (err != AGENTRT_SUCCESS) { agentrt_loop_destroy(loop); return; }

    /* 构造单节点工作流 */
    taskflow_workflow_t wf;
    AGENTRT_MEMSET(&wf, 0, sizeof(wf));
    AGENTRT_STRNCPY_TERM(wf.id, "wf_example_001", sizeof(wf.id));
    AGENTRT_STRNCPY_TERM(wf.name, "Example Workflow", sizeof(wf.name));
    AGENTRT_STRNCPY_TERM(wf.version, "1.0.0", sizeof(wf.version));

    wf.nodes = (taskflow_node_t *)AGENTRT_CALLOC(1, sizeof(taskflow_node_t));
    wf.node_count = 1;
    AGENTRT_STRNCPY_TERM(wf.nodes[0].id, "node_start", sizeof(wf.nodes[0].id));
    wf.nodes[0].type = TASKFLOW_NODE_TASK;
    wf.nodes[0].state = TASKFLOW_STATE_PENDING;
    wf.nodes[0].task_handler_name = AGENTRT_STRDUP("process_data");
    wf.initial_node_id = AGENTRT_STRDUP("node_start");

    /* 提交 DAG */
    char *exec_id = NULL;
    err = agentrt_loop_submit_dag(loop, &wf, "{\"input\":\"data\"}", &exec_id);
    if (err != AGENTRT_SUCCESS) {
        /* submit 失败，需自行释放 workflow 资源 */
        AGENTRT_FREE(wf.nodes[0].task_handler_name);
        AGENTRT_FREE(wf.nodes);
        AGENTRT_FREE(wf.initial_node_id);
        agentrt_loop_destroy(loop);
        return;
    }

    /* 等待完成并获取结果 */
    char *result = NULL;
    err = agentrt_loop_dag_wait(loop, exec_id, 0, &result);
    if (err == AGENTRT_SUCCESS) {
        printf("DAG result: %s\n", result);
        AGENTRT_FREE(result);
    }

    /* 查询状态 */
    char *state = NULL;
    double progress = 0.0;
    agentrt_loop_dag_status(loop, exec_id, &state, &progress);
    printf("State: %s, Progress: %.0f%%\n", state ? state : "?", progress * 100);
    AGENTRT_FREE(state);

    /* 清理 */
    AGENTRT_FREE(exec_id);
    agentrt_loop_destroy(loop);  /* 内部销毁 taskflow_engine，释放 workflow 资源 */
}
```

---

## 四、所有权规则（重要）

### 4.1 workflow 资源所有权

`taskflow_engine_register_workflow()` 是**浅拷贝**（复制 `taskflow_workflow_t` 结构体值，包括 `nodes`/`edges` 指针），`taskflow_engine_destroy()` 会释放：

- `nodes[j]` 的各字段（`task_handler_name`、`config_json`、`input_transform_json` 等）
- `nodes` 数组本身
- `edges` 数组本身
- `initial_node_id`
- `input_schema_json`、`output_schema_json`

**因此**：

| 场景 | 谁释放 workflow 资源 |
|------|---------------------|
| `submit_dag()` 成功 | `agentrt_loop_destroy()`（通过 engine destroy） |
| `submit_dag()` 失败 | 调用者自行释放 |

### 4.2 输出参数所有权

| 参数 | 释放者 |
|------|--------|
| `out_execution_id` | 调用者 `AGENTRT_FREE()` |
| `out_result_json` | 调用者 `AGENTRT_FREE()` |
| `out_state` | 调用者 `AGENTRT_FREE()` |

### 4.3 handler 输出所有权

handler 的 `output_json` 参数：handler 内部分配（如 `AGENTRT_STRDUP`），引擎在 `taskflow_engine_start()` 中将所有权转移到 `execution.output_json`，最终由 `taskflow_engine_destroy()` 释放。**handler 不应自行释放 output_json**。

---

## 五、注意事项

### 5.1 单节点同步执行

`taskflow_engine_start()` 对单节点工作流（`initial_node_id` 指向的节点）会**同步执行** handler，执行完毕后状态直接置为 COMPLETED/FAILED。此时 `agentrt_loop_dag_wait()` 会检测到已完成状态，直接返回结果，不调用 `run_to_completion()`。

### 5.2 多节点 DAG 的限制

当前 0.1.1 版本中，`taskflow_engine_start()` 只执行 `initial_node` 的 handler，不自动遍历后续节点。多节点 DAG 的完整遍历需要后续版本实现（通过 `taskflow_engine_step()` 或增强 `run_to_completion()`）。

### 5.3 load_workflow_json 桩函数

`taskflow_engine_load_workflow_json()` 是**桩函数**（不解析 JSON，只创建占位工作流 `wf_json_<count>`），违反 IRON-2（禁止桩函数）。W18 集成改用 `taskflow_engine_register_workflow()`（结构体方式）规避此桩函数。桩函数问题单独跟踪。

### 5.4 不使用 assert 执行副作用

Release 构建定义 `NDEBUG`，`assert(expr)` 展开为 `((void)0)`，导致 `expr` 中的函数调用不执行。所有执行副作用的检查必须用显式 `if` + 错误处理，不得依赖 `assert`。

---

## 六、集成测试参考

完整的集成测试位于 `AgentRT/agentrt/atoms/coreloopthree/tests/unit/test_dag_integration.c`，包含 4 个测试用例：

| 测试 | 验证内容 |
|------|---------|
| `test_dag_basic` | 基础 DAG 提交：创建 loop → 注册 handler → 提交 → 等待 → 验证结果 |
| `test_dag_status_query` | 状态查询：提交后查询 status，验证 state="completed" + progress=1.0 |
| `test_dag_param_validation` | 参数验证：NULL 参数返回 AGENTRT_EINVAL |
| `test_dag_handler_output` | handler 输出回传：handler 生成 JSON → wait 返回相同 JSON |

运行测试：

```bash
ctest --test-dir build -R cl3_dag_integration --output-on-failure
```

---

## 七、相关文件

| 文件 | 说明 |
|------|------|
| `agentrt/atoms/coreloopthree/include/loop.h` | DAG API 声明（4 个函数） |
| `agentrt/atoms/coreloopthree/src/loop.c` | DAG API 实现 + engine 生命周期 |
| `agentrt/atoms/taskflow/include/taskflow_advanced.h` | taskflow 类型定义 |
| `agentrt/atoms/taskflow/src/taskflow_advanced.c` | taskflow 引擎实现 |
| `agentrt/atoms/coreloopthree/tests/unit/test_dag_integration.c` | W18.3 集成测试 |
| `agentrt/atoms/coreloopthree/CMakeLists.txt` | 编译配置（链接 agentrt_taskflow） |
| `agentrt/daemon/common/CMakeLists.txt` | svc_common include taskflow 路径 |

---

## 八、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-04 | W18 初始实现：4 个 DAG API + 集成测试 + 文档 |
