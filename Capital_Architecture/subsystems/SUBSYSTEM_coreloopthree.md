# SUBSYSTEM_coreloopthree — 三层核心运行时

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/atoms/coreloopthree/`  
**负责人**: Team A (核心引擎)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

coreloopthree 是 Airymax 的智能体认知-执行-记忆三层核心运行时，是 Agent 智能行为的主要执行引擎。它实现感知→决策→执行的完整认知循环。

### 1.1 核心职责

| 职责 | 模块 | 说明 |
|------|------|------|
| **认知引擎** | `cognition.h` | 意图识别、任务规划、策略调度、结果批判 |
| **执行引擎** | `execution.h` | 任务执行、工具调度、步骤追踪 |
| **记忆引擎** | `memory.h` | 记忆写入/查询/进化/挂载，支持四层记忆模型 |
| **循环编排** | `loop.h` | 三层认知循环的主循环调度、任务提交/等待/恢复 |
| **认知进化** | `cognitive_evolution.h` | 记忆模式挖掘、策略自适应优化 |
| **多 Agent 协作** | `multi_agent_collaboration.h` | 多 Agent 任务分发、结果聚合、通信协调 |
| **补偿机制** | `compensation.h` | 任务失败时的补偿策略与回滚 |
| **浏览器交互** | `browser.h` | Web 内容抓取与交互能力 |
| **Agent 注册** | `agent_registry.h` | Agent 的注册、发现与生命周期管理 |
| **配置加载** | `config_loader.h` / `yaml_loader.h` | 配置文件解析与加载 |
| **Prompt 加载** | `prompt_loader.h` | 提示词模板加载与管理 |

### 1.2 认知管线（Phase 0-4）

coreloopthree 负责执行认知管线 Phase 0-4，Phase 5-6 由 orchestrator 编排层负责。

```
Phase 0: Decomposition  → 任务分解        [coreloopthree]
Phase 1: Planning       → 计划生成        [coreloopthree]
Phase 2: Generation     → 内容生成（调用 LLM）[coreloopthree]
Phase 3: Critique       → 批判性评估（三重协调）[coreloopthree]
Phase 4: Verification   → 结果验证        [coreloopthree]
────────────────────────────────────────────────
Phase 5: Audit          → 审计追踪        [orchestrator]
Phase 6: Alignment      → 对齐校准        [orchestrator]
```

### 1.3 不在范围内的职责

- 不负责 LLM 提供商管理（由 llm_d 负责）
- 不负责工具注册与执行（由 tool_d 负责）
- 不负责安全审计（由 cupolas 负责）
- 不负责 HTTP/WS 网关（由 gateway_d 负责）

---

## 2. 输入/输出接口

### 2.1 主循环接口 (`loop.h`)

```c
// 创建/销毁循环
agentos_error_t agentos_loop_create(const agentos_loop_config_t *config,
                                    agentos_core_loop_t **out_loop);
void agentos_loop_destroy(agentos_core_loop_t *loop);

// 运行控制
agentos_error_t agentos_loop_run(agentos_core_loop_t *loop);    // 阻塞运行
void agentos_loop_stop(agentos_core_loop_t *loop);              // 线程安全停止

// 任务提交（自然语言输入）
agentos_error_t agentos_loop_submit(agentos_core_loop_t *loop,
                                    const char *input, size_t input_len,
                                    char **out_task_id);
agentos_error_t agentos_loop_wait(agentos_core_loop_t *loop,
                                  const char *task_id, uint32_t timeout_ms,
                                  char **out_result, size_t *out_result_len);

// Checkpoint 支持
agentos_error_t agentos_loop_submit_persistent(loop, input, input_len,
                                               session_id, out_task_id);
agentos_error_t agentos_loop_restore_task(loop, task_id, out_restored_task_id);
agentos_error_t agentos_loop_list_checkpoints(loop, out_task_ids, out_count);

// 引擎访问
void agentos_loop_get_engines(loop, out_cognition, out_execution, out_memory);
```

### 2.2 记忆引擎接口 (`memory.h`)

| 函数 | 说明 |
|------|------|
| `agentos_memory_create(config_path, out_engine)` | 创建记忆引擎 |
| `agentos_memory_destroy(engine)` | 销毁记忆引擎 |
| `agentos_memory_write(engine, record, out_record_id)` | 写入记忆（线程安全） |
| `agentos_memory_query(engine, query, out_result)` | 查询记忆（线程安全） |
| `agentos_memory_get(engine, record_id, include_raw, out_record)` | 按 ID 获取 |
| `agentos_memory_mount(engine, record_id, context)` | 挂载到上下文 |
| `agentos_memory_evolve(engine, force)` | 触发记忆进化（模式挖掘） |
| `agentos_memory_health_check(engine, out_json)` | 健康检查 |

**记忆类型**: `EPISODIC`（情景）, `SEMANTIC`（语义）, `PROCEDURAL`（程序）, `WORKING`（工作）

### 2.3 适配器接口（外部服务桥接）

| 适配器 | 说明 |
|--------|------|
| `llm_svc_adapter.h` | 对接 llm_d 守护进程 |
| `tool_svc_adapter.h` | 对接 tool_d 守护进程 |
| `orch_adapter.h` | 对接 orchestrator 编排器 |
| `checkpoint_adapter.h` | 对接 checkpoint 检查点 |
| `memoryrovol_bridge.h` | 对接 MemoryRovol 商业提供商 |
| `prompt_loader.h` | Prompt 模板加载 |

### 2.4 循环配置

```c
typedef struct agentos_loop_config {
    uint32_t cognition_threads;       // 认知层线程数
    uint32_t execution_threads;       // 行动层线程数
    uint32_t memory_threads;          // 记忆层线程数
    uint32_t max_queued_tasks;        // 最大排队任务数
    uint32_t stats_interval_ms;       // 统计输出间隔
    size_t   memory_query_limit;      // 记忆检索上限（默认5）
    uint32_t task_timeout_ms;         // 任务超时（默认30000ms）
    float    memory_importance;       // 记忆重要性权重（默认0.7）
    agentos_plan_strategy_t      *plan_strategy;      // 规划策略
    agentos_coordinator_strategy_t *coord_strategy;   // 协同策略
    agentos_dispatching_strategy_t *disp_strategy;    // 调度策略
    uint32_t checkpoint_enabled;      // Checkpoint 启用标志
    char     checkpoint_path[256];    // Checkpoint 存储路径
    uint32_t checkpoint_interval_ms;  // Checkpoint 保存间隔
} agentos_loop_config_t;
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | 内存管理、任务调度、IPC、错误码 |
| **commons** | 强依赖 | 平台抽象、类型定义、同步原语 |
| **llm_d** | 服务依赖 | LLM 推理服务（通过适配器） |
| **tool_d** | 服务依赖 | 工具执行服务（通过适配器） |
| **memory provider** | 可拔插 | 记忆存储后端（内置/MemoryRovol） |
| **checkpoint** | 可选依赖 | 任务持久化与恢复 |
| **orchestrator** | 可选依赖 | 流程编排 |

### 3.2 被依赖方

- **gateway_d**: 通过 IPC 提交任务到 coreloopthree
- **orchestrator**: 编排器持有认知引擎引用
- **CLI/TUI**: 通过 gateway 间接调用

### 3.3 认知引擎内部结构

```
coreloopthree/
├── cognition/          # 认知引擎
│   ├── foundation/     # 语义单元、元认知、思维链
│   ├── intent/         # 意图分类、实体提取
│   ├── planner/        # 规划策略、LLM 客户端
│   ├── dispatcher/     # 调度策略
│   ├── coordinator/    # 协调策略
│   ├── critique/       # 置信度校准、三重协调、流式批判
│   ├── context/        # 上下文处理器
│   └── dispatch/       # 并行分发、委托
├── execution/          # 执行引擎
└── memory/             # 记忆引擎
```

---

## 4. 配置参数

### 4.1 循环配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `cognition_threads` | 由调用者设置 | 认知层线程数 |
| `execution_threads` | 由调用者设置 | 行动层线程数 |
| `memory_threads` | 由调用者设置 | 记忆层线程数 |
| `max_queued_tasks` | 由调用者设置 | 最大排队任务数 |
| `stats_interval_ms` | 0 (不输出) | 统计输出间隔 |
| `memory_query_limit` | 5 | 记忆检索上限 |
| `task_timeout_ms` | 30000 | 任务执行超时 |
| `memory_importance` | 0.7 | 记忆重要性权重 |
| `checkpoint_enabled` | 0 | Checkpoint 启用 |
| `checkpoint_interval_ms` | 30000 | Checkpoint 保存间隔 |

### 4.2 API 版本

| 常量 | 值 |
|------|-----|
| `LOOP_API_VERSION_MAJOR` | 1 |
| `LOOP_API_VERSION_MINOR` | 0 |
| `LOOP_API_VERSION_PATCH` | 0 |
| `MEMORY_API_VERSION_MAJOR` | 1 |
| `MEMORY_API_VERSION_MINOR` | 0 |
| `MEMORY_API_VERSION_PATCH` | 0 |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `agentos_loop_create` | 否 | 创建时单线程 |
| `agentos_loop_run` | 否 | 阻塞运行 |
| `agentos_loop_stop` | 是 | 使用条件变量+互斥锁 |
| `agentos_loop_submit` | 是 | 队列+互斥锁保护 |
| `agentos_loop_wait` | 是 | 条件变量+互斥锁 |
| `agentos_memory_write` | 是 | 互斥锁保护 |
| `agentos_memory_query` | 是 | 互斥锁保护 |
| `agentos_memory_mount` | 是 | 互斥锁保护 |

---

## 6. 相关文档

- [核心循环三层架构](../coreloopthree.md)
- [记忆卷载系统](../memoryrovol.md)
- [认知层理论](../../Basic_Theories/CN_02_认知层设计.md)
- [记忆层理论](../../Basic_Theories/CN_03_记忆层设计.md)