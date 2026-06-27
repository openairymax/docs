# SUBSYSTEM_llm_d — LLM 服务用户态服务

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/daemon/llm_d/`  
**负责人**: Team B (生态 + SDK)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

llm_d 是 Airymax 的 LLM（大语言模型）服务用户态服务，负责管理多个 LLM 提供商的路由、调用、缓存和成本追踪。它是 coreloopthree 认知引擎与外部 LLM API 之间的代理层。

### 1.1 核心职责

| 职责 | 模块 | 说明 |
|------|------|------|
| **LLM 推理** | `llm_service.h` | 统一的 LLM 调用接口（completion / stream） |
| **智能路由** | `llm_router.h` | 多提供商智能路由（成本/延迟/质量） |
| **提供商管理** | `providers/` | 多 LLM 提供商适配（OpenAI/Claude/本地模型） |
| **响应缓存** | `cache.h` | LLM 响应缓存，减少重复调用 |
| **Token 计数** | `token_counter.h` | Token 消耗精确统计 |
| **成本追踪** | `cost_tracker.h` | 费用计算与预算控制 |
| **流式响应** | `response.h` | 流式输出处理 |
| **服务适配** | `llm_svc_adapter.h` | 对接 coreloopthree 认知引擎 |

### 1.2 不在范围内的职责

- 不负责 LLM 模型训练或推理引擎实现（调用外部 API）
- 不负责 Prompt 模板管理（由 coreloopthree 的 `prompt_loader.h` 负责）
- 不负责 Agent 市场管理（由 market_d 负责）
- 不负责工具执行（由 tool_d 负责）

---

## 2. 输入/输出接口

### 2.1 服务生命周期 (`llm_service.h`)

```c
llm_service_t *llm_service_create(const char *config_path);
void llm_service_destroy(llm_service_t *svc);
```

### 2.2 推理接口

```c
// 同步推理
int llm_service_complete(llm_service_t *svc,
                         const llm_request_config_t *config,
                         llm_response_t **out_response);

// 流式推理
int llm_service_complete_stream(llm_service_t *svc,
                                const llm_request_config_t *config,
                                llm_stream_callback_t callback,
                                void *callback_data,
                                llm_response_t **out_response);

// 释放响应
void llm_response_free(llm_response_t *resp);
```

### 2.3 请求配置

```c
typedef struct {
    const char *model;              // 模型名称
    const llm_message_t *messages;  // 消息数组
    size_t message_count;           // 消息数量
    float temperature;              // 温度参数
    float top_p;                    // Top-P 采样
    int max_tokens;                 // 最大 Token 数
    int stream;                     // 是否流式
    const char **stop;              // 停止词
    size_t stop_count;              // 停止词数量
    double presence_penalty;        // 存在惩罚
    double frequency_penalty;       // 频率惩罚
    void *user_data;                // 用户数据
} llm_request_config_t;
```

### 2.4 响应结构

```c
typedef struct {
    char *id;                       // 响应 ID
    char *model;                    // 使用的模型
    llm_message_t *choices;         // 响应选项
    size_t choice_count;            // 选项数量
    uint64_t created;               // 创建时间戳
    uint32_t prompt_tokens;         // Prompt Token 数
    uint32_t completion_tokens;     // 补全 Token 数
    uint32_t total_tokens;          // 总 Token 数
    char *finish_reason;            // 完成原因
} llm_response_t;
```

### 2.5 统计接口

```c
int llm_service_stats(llm_service_t *svc, char **out_json);
```

### 2.6 流式回调

```c
typedef void (*llm_stream_callback_t)(const char *chunk, void *user_data);
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | IPC 总线、内存管理、错误码 |
| **daemon/common** | 强依赖 | 服务框架（`svc_common.h`） |
| **外部 LLM API** | 外部服务 | OpenAI / Claude / 本地模型 API |
| **cupolas** | 可选依赖 | 输入净化 |

### 3.2 被依赖方

- **coreloopthree**: 通过 `llm_svc_adapter.h` 调用 LLM 推理
- **orchestrator**: 编排器通过 `orchestrator_set_cognition_llm_service()` 注入 LLM 服务

### 3.3 智能路由架构

```
┌──────────────────────────────────────────┐
│              llm_router.h                 │
│    (成本/延迟/质量 三维路由策略)          │
└──────────────────┬───────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌──────────┐
│ OpenAI  │  │ Claude  │  │ 本地模型  │
│ 适配器  │  │ 适配器  │  │  适配器  │
└─────────┘  └─────────┘  └──────────┘
```

---

## 4. 配置参数

### 4.1 服务配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `config_path` | 由调用者指定 | 配置文件路径 |
| `model` | 由调用者指定 | 默认模型名称 |
| `temperature` | 由调用者指定 | 温度参数 |
| `top_p` | 由调用者指定 | Top-P 采样 |
| `max_tokens` | 由调用者指定 | 最大 Token 数 |
| `stream` | 0 | 是否流式输出 |

### 4.2 路由配置

| 参数 | 说明 |
|------|------|
| 成本权重 | 路由时的成本因素权重 |
| 延迟权重 | 路由时的延迟因素权重 |
| 质量权重 | 路由时的质量因素权重 |

### 4.3 缓存配置

| 参数 | 说明 |
|------|------|
| 缓存 TTL | 响应缓存过期时间 |
| 缓存策略 | 精确匹配 / 语义相似度匹配 |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `llm_service_create` | 否 | 创建时单线程 |
| `llm_service_complete` | 是 | 可并发调用 |
| `llm_service_complete_stream` | 是 | 流式调用线程安全 |
| `llm_service_stats` | 是 | 只读操作 |

---

## 6. 连接线

| 连接线 | 源 | 目标 | 说明 |
|--------|-----|------|------|
| C-L06 | orchestrator | llm_d | 编排器注入 LLM 服务到认知引擎 |

---

## 7. 相关文档

- [核心循环三层架构](../coreloopthree.md)
- [通信协议规范](../../Capital_Specifications/agentos_contract/protocol_contract.md)