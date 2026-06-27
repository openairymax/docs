# SUBSYSTEM_tool_d — 工具服务用户态服务

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/daemon/tool_d/`  
**负责人**: Team B (生态 + SDK)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

tool_d 是 Airymax 的工具服务用户态服务，负责工具的注册、发现、执行和安全管理。它使 Agent 能够调用外部工具（命令行、API、脚本等）来完成复杂任务。

### 1.1 核心职责

| 职责 | 模块 | 说明 |
|------|------|------|
| **工具注册** | `registry.h` | 工具注册/注销/查询 |
| **工具执行** | `executor.h` | 同步/流式执行工具命令 |
| **参数校验** | `validator.h` | JSON Schema 参数校验 |
| **结果缓存** | `cache.h` | 可缓存的工具结果 |
| **安全审批** | `tool_approval.h` | 敏感工具的执行审批流程 |
| **安全守卫** | `safety_guard_bridge.h` | 对接 cupolas 安全守卫 |
| **服务适配** | `tool_svc_adapter.h` | 对接 coreloopthree 执行引擎 |
| **错误处理** | `tool_errors.h` | 工具执行错误分类与处理 |

### 1.2 不在范围内的职责

- 不负责 Agent 市场管理（由 market_d 负责）
- 不负责 LLM 推理（由 llm_d 负责）
- 不负责安全审计存储（由 cupolas 负责）
- 不负责工具开发（工具由生态贡献者提供）

---

## 2. 输入/输出接口

### 2.1 服务生命周期

```c
tool_service_t *tool_service_create(const char *config_path);
void tool_service_destroy(tool_service_t *svc);
```

### 2.2 工具管理

```c
int tool_service_register(tool_service_t *svc, const tool_metadata_t *meta);
int tool_service_unregister(tool_service_t *svc, const char *tool_id);
tool_metadata_t *tool_service_get(tool_service_t *svc, const char *tool_id);
void tool_metadata_free(tool_metadata_t *meta);
char *tool_service_list(tool_service_t *svc);  // 返回 JSON 数组
```

### 2.3 工具执行

```c
// 同步执行
int tool_service_execute(tool_service_t *svc,
                         const tool_execute_request_t *req,
                         tool_result_t **out_result);

// 流式执行
int tool_service_execute_stream(tool_service_t *svc,
                                const tool_execute_request_t *req,
                                tool_stream_callback_t callback,
                                void *callback_data,
                                tool_result_t **out_result);

void tool_result_free(tool_result_t *res);
```

### 2.4 工具元数据

```c
typedef struct {
    char *id;              // 工具唯一标识
    char *name;            // 工具名称
    char *description;     // 描述
    char *executable;      // 可执行文件路径或命令
    tool_param_t *params;  // 参数定义数组
    size_t param_count;    // 参数数量
    int timeout_sec;       // 执行超时（秒）
    int cacheable;         // 结果是否可缓存
    char *permission_rule; // 权限规则标识（与 cupolas 配合）
} tool_metadata_t;
```

### 2.5 执行请求与结果

```c
typedef struct {
    const char *tool_id;      // 工具 ID
    const char *params_json;  // 参数 JSON 字符串
    int stream;               // 是否流式输出
    void *user_data;          // 透传用户数据
} tool_execute_request_t;

typedef struct {
    int success;           // 0=成功, 非0=失败
    char *output;          // 标准输出
    char *error;           // 标准错误
    int exit_code;         // 进程退出码
    uint64_t duration_ms;  // 执行耗时
} tool_result_t;
```

### 2.6 流式回调

```c
typedef void (*tool_stream_callback_t)(const char *chunk, int is_stderr, void *user_data);
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | IPC 总线、内存管理、错误码 |
| **daemon/common** | 强依赖 | 服务框架（`svc_common.h`） |
| **cupolas** | 强依赖 | 安全守卫桥接（`safety_guard_bridge.h`） |

### 3.2 被依赖方

- **coreloopthree**: 通过 `tool_svc_adapter.h` 调用工具执行
- **orchestrator**: 编排器通过 `orchestrator_set_cognition_tool_service()` 注入工具服务

### 3.3 安全集成

```
tool_d 执行前
    │
    ▼
safety_guard_bridge.h ──▶ cupolas 权限检查
    │
    ├── 允许 ──▶ executor.h ──▶ 执行工具
    │
    └── 拒绝 ──▶ 返回权限错误
```

---

## 4. 配置参数

### 4.1 服务配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `config_path` | 由调用者指定 | 配置文件路径 |

### 4.2 工具配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `timeout_sec` | 由注册者指定 | 执行超时时间 |
| `cacheable` | 0 | 是否可缓存结果 |
| `permission_rule` | 由注册者指定 | 权限规则标识 |

### 4.3 缓存配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 缓存 TTL | 由配置决定 | 工具结果缓存过期时间 |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `tool_service_create` | 否 | 创建时单线程 |
| `tool_service_register` | 是 | 注册操作线程安全 |
| `tool_service_execute` | 是 | 并发执行不同工具 |
| `tool_service_execute_stream` | 是 | 流式执行线程安全 |

---

## 6. 连接线

| 连接线 | 源 | 目标 | 说明 |
|--------|-----|------|------|
| C-L06 | orchestrator | tool_d | 编排器注入工具服务到认知引擎 |

---

## 7. 相关文档

- [核心循环三层架构](../coreloopthree.md)
- [安全设计规范](../../Capital_Specifications/coding_standard/Security_design_standard.md)
- [Skill 契约规范](../../Capital_Specifications/agentos_contract/skill_contract.md)