# SUBSYSTEM_orchestrator — 流程编排器

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/daemon/common/` (orchestrator.h)  
**负责人**: Team A (核心引擎)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

orchestrator 是 Airymax 的流程编排引擎，负责协调多 Agent / 多 Skill 的复杂任务流程。它支持串行/并行/条件/循环/自适应等多种编排策略，并集成思考链追踪和元认知能力。

### 1.1 核心职责

| 职责 | 说明 |
|------|------|
| **任务编排** | Phase 0-4 串行执行管线（分解→规划→生成→批判→验证→审计→对齐） |
| **多策略调度** | 串行/并行/条件/循环/自适应 5 种编排策略 |
| **子任务分发** | 将任务分解为子任务并分发到不同 Agent/Skill |
| **结果聚合** | 收集子任务结果并聚合为最终输出 |
| **超时与熔断** | 内置超时控制与熔断器集成 |
| **思考链追踪** | 思考链路追踪与审计 |
| **元认知** | 认知过程的自省与优化 |
| **批判循环** | 三重协调批判（置信度校准、流式批判、自动纠正） |
| **进度回调** | 执行进度实时通知 |
| **LLM/Tool 服务注入** | `orchestrator_set_cognition_llm_service()` / `orchestrator_set_cognition_tool_service()` |

### 1.2 编排阶段（Phase 0-6）

| 阶段 | 枚举值 | 说明 |
|------|--------|------|
| Decomposition | `ORCH_PHASE_DECOMPOSITION` | 任务分解 |
| Planning | `ORCH_PHASE_PLANNING` | 计划生成 |
| Generation | `ORCH_PHASE_GENERATION` | 内容生成 |
| Critique | `ORCH_PHASE_CRITIQUE` | 批判性评估 |
| Verification | `ORCH_PHASE_VERIFICATION` | 结果验证 |
| Audit | `ORCH_PHASE_AUDIT` | 审计追踪 |
| Alignment | `ORCH_PHASE_ALIGNMENT` | 对齐校准 |

### 1.3 不在范围内的职责

- 不负责 Agent 运行时执行（由 coreloopthree 负责）
- 不负责 LLM 推理（由 llm_d 负责）
- 不负责工具执行（由 tool_d 负责）
- 不负责持久化记忆（由 memory 子系统负责）

---

## 2. 输入/输出接口

### 2.1 生命周期

```c
orchestrator_t *orchestrator_create(const orch_config_t *config);
void orchestrator_destroy(orchestrator_t *orch);
void orchestrator_global_cleanup(void);  // 销毁全局 mutex
```

### 2.2 管线操作

```c
orch_pipeline_t *orchestrator_pipeline_create(orchestrator_t *orch, const char *name);
void orchestrator_pipeline_destroy(orch_pipeline_t *pipeline);
int orchestrator_pipeline_add_step(orch_pipeline_t *pipeline,
                                   const orch_pipeline_step_t *step);
```

### 2.3 执行

```c
// 编排执行
int orchestrator_execute(orchestrator_t *orch, const char *input,
                         orch_result_t **out_results, size_t *out_count);

// 管线执行
int orchestrator_execute_pipeline(orchestrator_t *orch, orch_pipeline_t *pipeline,
                                  const char *input, orch_result_t **out_results,
                                  size_t *out_count);

// 单阶段执行
int orchestrator_execute_phase(orchestrator_t *orch, orch_phase_t phase,
                               const char *input, orch_result_t **out_result);
```

### 2.4 LLM/Tool 服务注入 (C-L06 连接线)

```c
void orchestrator_set_cognition_llm_service(orchestrator_t *orch, llm_service_t *llm_svc);
void orchestrator_set_cognition_tool_service(orchestrator_t *orch, tool_service_t *tool_svc);
```

### 2.5 查询与控制

```c
orch_task_status_t orchestrator_get_task_status(orchestrator_t *orch, const char *task_id);
orch_result_t *orchestrator_get_result(orchestrator_t *orch, const char *task_id);
uint32_t orchestrator_active_count(orchestrator_t *orch);
int orchestrator_cancel(orchestrator_t *orch, const char *task_id);
int orchestrator_cancel_all(orchestrator_t *orch);
void orchestrator_result_free(orch_result_t *result);
```

### 2.6 进度回调

```c
void orchestrator_set_progress_callback(orchestrator_t *orch,
                                        orch_progress_cb_t callback, void *user_data);
```

### 2.7 配置

```c
typedef struct {
    uint32_t max_subtasks;                  // 最大子任务数（默认 16）
    uint32_t timeout_ms;                    // 超时时间（默认 60000ms）
    uint32_t max_retries;                   // 最大重试次数（默认 3）
    uint32_t retry_delay_ms;                // 重试延迟（默认 100ms）
    orch_strategy_t default_strategy;       // 默认策略（串行）
    bool enable_thinking_chain;             // 启用思考链（默认 true）
    bool enable_metacognition;              // 启用元认知（默认 true）
    bool enable_circuit_breaker;            // 启用熔断器（默认 true）
    bool enable_critique_loop;              // 启用批判循环（默认 true）
    uint32_t critique_max_rounds;           // 最大批判轮次（默认 3）
    float critique_acceptance_threshold;    // 批判接受阈值（默认 0.7）
    float critique_auto_correct_threshold;  // 自动纠正阈值（默认 0.5）
} orch_config_t;
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | 内存管理、错误码 |
| **llm_d** | 服务依赖 | LLM 推理（通过注入） |
| **tool_d** | 服务依赖 | 工具执行（通过注入） |
| **coreloopthree** | 可选依赖 | 认知引擎引用 |

### 3.2 被依赖方

- **gateway_d**: 通过 API 调用编排器执行任务
- **CLI**: 通过 gateway 提交编排任务

### 3.3 编排策略

```
┌──────────────────────────────────────────────────────┐
│                  orchestrator                        │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐       │
│  │ Serial   │  │ Parallel │  │ Conditional  │       │
│  └──────────┘  └──────────┘  └──────────────┘       │
│  ┌──────────┐  ┌──────────┐                         │
│  │ Loop     │  │ Adaptive │                         │
│  └──────────┘  └──────────┘                         │
│                                                      │
│  Phase 0 ──▶ Phase 1 ──▶ Phase 2 ──▶ Phase 3       │
│       ──▶ Phase 4 ──▶ Phase 5 ──▶ Phase 6           │
└──────────────────────────────────────────────────────┘
```

---

## 4. 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_subtasks` | 16 | 最大子任务数 |
| `timeout_ms` | 60000 | 超时时间（毫秒） |
| `max_retries` | 3 | 最大重试次数 |
| `retry_delay_ms` | 100 | 重试延迟（毫秒） |
| `default_strategy` | SEQUENTIAL | 默认编排策略 |
| `enable_thinking_chain` | true | 启用思考链追踪 |
| `enable_metacognition` | true | 启用元认知 |
| `enable_circuit_breaker` | true | 启用熔断器 |
| `enable_critique_loop` | true | 启用批判循环 |
| `critique_max_rounds` | 3 | 最大批判轮次 |
| `critique_acceptance_threshold` | 0.7 | 批判接受阈值 |
| `critique_auto_correct_threshold` | 0.5 | 自动纠正阈值 |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `orchestrator_create` | 否 | 创建时单线程 |
| `orchestrator_execute` | 是 | 并发执行不同编排 |
| `orchestrator_get_task_status` | 是 | 只读查询 |
| `orchestrator_cancel` | 是 | 取消操作线程安全 |
| `orchestrator_global_cleanup` | 否 | 进程退出时调用 |

---

## 6. 连接线

| 连接线 | 源 | 目标 | 说明 |
|--------|-----|------|------|
| C-L06 | orchestrator | llm_d / tool_d | 编排器注入服务到认知引擎 |

---

## 7. 相关文档

- [核心循环三层架构](../coreloopthree.md)
- [Agent 契约规范](../../Capital_Specifications/agentos_contract/agent_contract.md)
- [API 总览](../../Capital_API/README.md)