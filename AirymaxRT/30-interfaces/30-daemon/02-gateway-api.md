# 用户态服务 API 文档
> **文档定位**：用户态服务 API 文档\
> **最后更新**：2026-06-09\
> **上级文档**：[AirymaxAgentRT 接口文档](README.md)

---

## 📋 概述

Airymax 提供10个用户态服务（Daemon），构成完整的Agent运行时基础设施：

```
┌──────────────────────────────────────────────────────────────┐
│                      Airymax Daemon 层                       │
├──────────┬──────────┬──────────┬──────────┬─────────────────┤
│ gateway_d│  llm_d   │ tool_d   │ market_d │    sched_d      │
│  网关服务 │ LLM服务  │工具执行器 │ 市场服务 │    调度器       │
├──────────┼──────────┼──────────┼──────────┼─────────────────┤
│ monit_d  │channel_d │observe_d │notify_d  │    info_d       │
│ 监控服务 │ 通道服务 │ 观测服务 │ 通知服务 │   信息服务      │
└──────────┴──────────┴──────────┴──────────┴─────────────────┘
           ↕            ↕           ↕
     ┌──────────┬──────────┬──────────┐
     │ Protocol │  Core    │ Commons  │
     │   Stack  │  Atoms   │ Utils    │
     └──────────┴──────────┴──────────┘
```

---

## 🌐 网关服务 (gateway_d)

### 功能概述
- **HTTP网关**: RESTful API端点 (端口8080)
- **WebSocket网关**: 实时双向通信 (端口8081)
- **Stdio网关**: 标准输入输出模式
- **协议转换**: MCP/A2A/OpenAI协议适配

### API接口

#### 服务生命周期

```c
#include "gateway_service.h"

// 获取默认配置
void gateway_service_get_default_config(gateway_service_config_t* config);

// 从文件加载配置
airy_err_t gateway_service_load_config(
    gateway_service_config_t* config,
    const char* config_path
);

// 创建服务实例
airy_err_t gateway_service_create(
    gateway_service_t* service,
    const gateway_service_config_t* config
);

// 初始化服务（创建后必须调用）
airy_err_t gateway_service_init(gateway_service_t service);

// 启动服务（开始监听端口）
airy_err_t gateway_service_start(gateway_service_t service);

// 停止服务
airy_err_t gateway_service_stop(gateway_service_t service, bool force);

// 销毁服务
void gateway_service_destroy(gateway_service_t service);
```

#### 状态查询

```c
// 获取当前状态
airy_svc_state_t gateway_service_get_state(gateway_service_t service);
// 返回值: CREATED, READY, RUNNING, STOPPED, ERROR

// 是否运行中
bool gateway_service_is_running(gateway_service_t service);

// 获取统计信息
airy_err_t gateway_service_get_stats(
    gateway_service_t service,
    airy_svc_stats_t* stats
);
// stats包含: request_count, error_count, avg_time_ms, current_concurrent

// 健康检查
airy_err_t gateway_service_healthcheck(gateway_service_t service);
```

### 配置结构

```c
typedef struct {
    // 服务基本信息
    const char* name;              /* 服务名称 */
    const char* version;           /* 版本号 */

    // HTTP配置
    struct {
        gateway_daemon_type_t type;      /* 类型: HTTP */
        const char* host;                 /* 监听地址 */
        uint16_t port;                    /* 端口 */
        bool enabled;                     /* 是否启用 */
        size_t max_request_size;          /* 最大请求大小 */
        uint32_t timeout_ms;              /* 超时时间 */
    } http;

    // WebSocket配置
    struct {
        gateway_daemon_type_t type;
        const char* host;
        uint16_t port;
        bool enabled;
        size_t max_request_size;
        uint32_t timeout_ms;
    } ws;

    // Stdio配置
    struct {
        gateway_daemon_type_t type;
        bool enabled;
        size_t max_request_size;
        uint32_t timeout_ms;
    } stdio;

    // 高级选项
    bool enable_metrics;             /* 启用指标收集 */
    bool enable_tracing;             /* 启用追踪 */
    uint32_t shutdown_timeout_ms;    /* 关闭超时 */
} gateway_service_config_t;
```

### 使用示例

```c
#include "gateway_service.h"
#include "svc_common.h"

int main() {
    // 1. 初始化公共库
    svc_common_init();

    // 2. 创建并配置服务
    gateway_service_t service = NULL;
    gateway_service_config_t config;
    gateway_service_get_default_config(&config);

    // 自定义配置
    config.http.port = 9090;
    config.http.host = "127.0.0.1";
    config.enable_metrics = true;

    // 3. 创建服务
    airy_err_t err = gateway_service_create(&service, &config);
    if (err != AIRY_EOK) {
        SVC_LOG_ERROR("Failed to create gateway: %d", err);
        return -1;
    }

    // 4. 初始化并启动
    err = gateway_service_init(service);
    if (err == AIRY_EOK) {
        err = gateway_service_start(service);
        if (err == AIRY_EOK) {
            SVC_LOG_INFO("Gateway started on port %d", config.http.port);

            // 主循环
            while (gateway_service_is_running(service)) {
                svc_sleep_ms(1000);
                
                // 定期健康检查
                if (config.enable_metrics) {
                    airy_svc_stats_t stats;
                    gateway_service_get_stats(service, &stats);
                    SVC_LOG_INFO("Requests: %llu, Errors: %llu",
                               stats.request_count, stats.error_count);
                }
            }
        }
    }

    // 5. 清理
    gateway_service_destroy(service);
    svc_common_cleanup();
    
    return 0;
}
```

---

## 🤖 LLM服务 (llm_d)

### 功能概述
- LLM模型调用封装（OpenAI兼容API）
- Token计数与成本估算
- 响应缓存机制
- 定价规则管理
- 多模型支持

### API接口

```c
#include "service.h"  // llm_d/service.h

// 创建LLM服务
llm_service_t* llm_service_create(const char* config_path);

// 销毁服务
void llm_service_destroy(llm_service_t* svc);

// 发送请求
airy_err_t llm_service_request(
    llm_service_t* svc,
    const llm_request_config_t* request,
    llm_response_t** response
);

// Token计数
int llm_count_tokens(llm_service_t* svc, 
                      const char* text, 
                      const char* model);

// 缓存操作
airy_err_t llm_cache_put(llm_service_t* svc, 
                                const char* key, 
                                const void* value, 
                                size_t size);
airy_err_t llm_cache_get(llm_service_t* svc, 
                                const char* key, 
                                void** value, 
                                size_t* size);
void llm_cache_clear(llm_service_t* svc);
```

### 请求配置

```c
typedef struct {
    const char* model;                    /* 模型名称 (如 "gpt-4") */
    llm_message_t* messages;              /* 消息数组 */
    size_t message_count;                 /* 消息数量 */
    
    float temperature;                     /* 温度参数 (0.0-2.0) */
    int max_tokens;                       /* 最大生成token数 */
    float top_p;                          /* Top-p采样 */
    
    bool stream;                          /* 是否流式输出 */
    uint32_t timeout_ms;                  /* 超时时间 */
} llm_request_config_t;

typedef struct {
    const char* role;      /* "system"/"user"/"assistant" */
    const char* content;   /* 消息内容 */
} llm_message_t;
```

### 使用示例

```c
#include "service.h"
#include "response.h"

int main() {
    // 1. 创建服务
    llm_service_t* svc = llm_service_create("/etc/agentrt/llm.conf");
    if (!svc) {
        fprintf(stderr, "Failed to create LLM service\n");
        return -1;
    }

    // 2. 构建请求
    llm_message_t messages[] = {
        { .role = "system", .content = "You are a helpful assistant." },
        { .role = "user", .content = "Explain quantum computing." }
    };

    llm_request_config_t request = {
        .model = "gpt-4",
        .messages = messages,
        .message_count = 2,
        .temperature = 0.7,
        .max_tokens = 1000,
        .timeout_ms = 30000
    };

    // 3. 发送请求
    llm_response_t* response = NULL;
    airy_err_t err = llm_service_request(svc, &request, &response);

    if (err == AIRY_EOK && response) {
        printf("Response:\n%s\n", response->content);
        printf("Tokens used: %d\n", response->usage.total_tokens);
        printf("Cost: $%.4f\n", response->cost);
        
        llm_response_free(response);
    } else {
        fprintf(stderr, "Request failed: %d\n", err);
    }

    // 4. 清理
    llm_service_destroy(svc);
    return 0;
}
```

---

## 📡 通道服务 (channel_d)

### 功能概述
- Agent间消息传递通道
- 发布/订阅模式
- 点对点通信
- 线程安全保证

### API接口

```c
#include "channel_service.h"

// 创建通道服务
airy_err_t channel_service_create(channel_service_t* svc);

// 销毁
void channel_service_destroy(channel_service_t svc);

// 创建通道
airy_err_t channel_service_create_channel(
    channel_service_t svc,
    const char* channel_name,
    channel_config_t* config
);

// 发布消息
airy_err_t channel_service_publish(
    channel_service_t svc,
    const char* channel_name,
    const channel_message_t* message
);

// 订阅通道
airy_err_t channel_service_subscribe(
    channel_service_t svc,
    const char* channel_name,
    channel_callback_t callback,
    void* user_data
);

// 取消订阅
airy_err_t channel_service_unsubscribe(
    channel_service_t svc,
    const char* channel_name,
    channel_subscription_t sub
);
```

---

## ⏱️ 调度器 (sched_d)

### 功能概述
- 任务调度策略（FIFO/LIFO/Priority/Weighted）
- 并发控制
- 超时管理
- 资源分配

### API接口

```c
#include "scheduler_service.h"

// 创建调度器
airy_err_t scheduler_service_create(
    scheduler_service_t* svc,
    const scheduler_config_t* config
);

// 提交任务
airy_err_t scheduler_submit(
    scheduler_service_t svc,
    const task_descriptor_t* task,
    task_handle_t* out_handle
);

// 取消任务
airy_err_t scheduler_cancel(
    scheduler_service_t svc,
    task_handle_t handle
);

// 等待完成
airy_err_t scheduler_wait(
    scheduler_service_t svc,
    task_handle_t handle,
    uint32_t timeout_ms,
    task_result_t** result
);

// 获取调度器状态
airy_err_t scheduler_get_stats(
    scheduler_service_t svc,
    scheduler_stats_t* stats
);
```

### 调度策略

```c
typedef enum {
    SCHED_STRATEGY_FIFO,       /* 先进先出 */
    SCHED_STRATEGY_LIFO,       /* 后进先出 */
    SCHED_STRATEGY_PRIORITY,   /* 优先级 */
    SCHED_STRATEGY_WEIGHTED,   /* 加权轮询 */
    SCHED_STRATEGY_FAIR        /* 公平调度 */
} scheduler_strategy_t;
```

---

## 📊 监控服务 (monit_d)

### 功能概述
- 运行时指标收集
- 分布式追踪
- 告警规则引擎
- 健康检查端点

### API接口

```c
#include "monitoring_service.h"

// 创建监控服务
airy_err_t monitoring_service_create(monitoring_service_t* svc);

// 注册指标
airy_err_t monitoring_register_metric(
    monitoring_service_t svc,
    const metric_definition_t* metric
);

// 记录指标值
airy_err_t monitoring_record(
    monitoring_service_t svc,
    const char* metric_name,
    double value,
    tag_list_t* tags
);

// 创建追踪span
trace_span_t* monitoring_start_trace(
    monitoring_service_t svc,
    const char* operation_name,
    trace_context_t* parent_ctx
);

void monitoring_end_trace(trace_span_t* span);

// 配置告警规则
airy_err_t monitoring_set_alert_rule(
    monitoring_service_t svc,
    const alert_rule_t* rule
);

// 导出Prometheus格式指标
char* monitoring_export_prometheus(monitoring_service_t svc);
```

### 指标类型

```c
typedef enum {
    METRIC_TYPE_COUNTER,    /* 计数器 (只增不减) */
    METRIC_TYPE_GAUGE,      /* 仪表盘 (可增可减) */
    METRIC_TYPE_HISTOGRAM,  /* 直方图 */
    METRIC_TYPE_SUMMARY      /* 摘要 */
} metric_type_t;
```

---

## 🏪 市场服务 (market_d)

### 功能概述
- Agent注册中心
- 技能/Skill注册与发现
- 安装器接口
- 能力协商

### API接口

```c
#include "market_service.h"

// 创建市场服务
airy_err_t market_service_create(market_service_t* svc);

// 注册Agent
airy_err_t market_register_agent(
    market_service_t svc,
    const agent_descriptor_t* agent,
    agent_handle_t* out_handle
);

// 注销Agent
airy_err_t market_unregister_agent(
    market_service_t svc,
    agent_handle_t handle
);

// 发现Agent
airy_err_t market_discover_agents(
    market_service_t svc,
    const agent_query_t* query,
    agent_descriptor_t** results,
    size_t* count
);

// 注册技能
airy_err_t market_register_skill(
    market_service_t svc,
    const skill_descriptor_t* skill,
    skill_handle_t* out_handle
);

// 查询技能
airy_err_t market_find_skill(
    market_service_t svc,
    const char* skill_name,
    skill_descriptor_t* out_skill
);

// 安装Agent/Skill
airy_err_t market_install(
    market_service_t svc,
    const char* package_name,
    install_options_t* options
);
```

---

## 🔧 工具执行器 (tool_d)

### 功能概述
- 工具注册与管理
- 安全的沙箱执行
- 输入验证
- 结果缓存

### API接口

```c
#include "tool_service.h"

// 创建工具服务
airy_err_t tool_service_create(tool_service_t* svc);

// 注册工具
airy_err_t tool_register(
    tool_service_t svc,
    const tool_descriptor_t* tool,
    tool_handle_t* out_handle
);

// 注销工具
airy_err_t tool_unregister(tool_service_t svc, tool_handle_t handle);

// 执行工具
airy_err_t tool_execute(
    tool_service_t svc,
    tool_handle_t handle,
    const tool_input_t* input,
    tool_output_t** output
);

// 验证输入
airy_err_t tool_validate_input(
    tool_service_t svc,
    tool_handle_t handle,
    const tool_input_t* input,
    validation_result_t* result
);

// 列出所有已注册工具
airy_err_t tool_list_tools(
    tool_service_t svc,
    tool_descriptor_t** tools,
    size_t* count
);
```

### 工具描述符

```c
typedef struct {
    const char* name;                  /* 工具名称 */
    const char* description;           /* 描述 */
    const char* version;               /* 版本 */
    
    // 输入模式 (JSON Schema)
    const char* input_schema;
    
    // 执行函数
    tool_execute_fn execute;
    
    // 权限要求
    tool_permissions_t permissions;
    
    // 超时设置
    uint32_t timeout_ms;
    
    // 是否需要网络访问
    bool requires_network;
    
    // 用户上下文
    void* user_data;
} tool_descriptor_t;
```

---

## 🔗 用户态服务通用接口

所有用户态服务都遵循统一的接口规范：

### 通用生命周期

```c
typedef enum {
    AIRY_SVC_STATE_NONE = 0,
    AIRY_SVC_STATE_CREATED,
    AIRY_SVC_STATE_READY,      /* initialized but not started */
    AIRY_SVC_STATE_RUNNING,
    AIRY_SVC_STATE_STOPPED,
    AIRY_SVC_STATE_ERROR,
    AIRY_SVC_STATE_DEGRADED     /* running with limited functionality */
} airy_svc_state_t;

typedef struct {
    uint64_t request_count;        /* 总请求数 */
    uint64_t error_count;          /* 错误数 */
    uint64_t current_concurrent;   /* 当前并发数 */
    double   avg_time_ms;          /* 平均响应时间(ms) */
    time_t   uptime_seconds;       /* 运行时间(秒) */
} airy_svc_stats_t;
```

### 统一初始化/清理

```c
// 所有用户态服务都需要调用此函数进行全局初始化
airy_err_t svc_common_init(void);
void svc_common_cleanup(void);

// 日志宏 (线程安全)
SVC_LOG_DEBUG(format, ...);
SVC_LOG_INFO(format, ...);
SVC_LOG_WARN(format, ...);
SVC_LOG_ERROR(format, ...);
```

---

## 🚀 快速启动示例

### 完整的多用户态服务启动

```c
#include <stdio.h>
#include <signal.h>
#include "gateway_service.h"
#include "llm_d/service.h"
#include "scheduler_service.h"
#include "monitoring_service.h"

static volatile int g_running = 1;

void signal_handler(int sig) {
    (void)sig;
    g_running = 0;
}

int main() {
    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);

    // 1. 全局初始化
    svc_common_init();

    // 2. 创建各服务
    gateway_service_t gateway = NULL;
    llm_service_t* llm = NULL;
    scheduler_service_t scheduler = NULL;
    monitoring_service_t monitor = NULL;

    // Gateway
    gateway_service_config_t gw_cfg;
    gateway_service_get_default_config(&gw_cfg);
    gateway_service_create(&gateway, &gw_cfg);
    gateway_service_init(gateway);
    gateway_service_start(gateway);

    // LLM Service
    llm = llm_service_create("/etc/agentrt/llm.conf");

    // Scheduler
    scheduler_config_t sched_cfg = { .strategy = SCHED_STRATEGY_PRIORITY };
    scheduler_service_create(&scheduler, &sched_cfg);

    // Monitor
    monitoring_service_create(&monitor);

    printf("All services started successfully!\n");

    // 3. 主循环
    while (g_running) {
        svc_sleep_ms(1000);
        
        // 定期打印状态
        if (gateway_service_is_running(gateway)) {
            airy_svc_stats_t stats;
            gateway_service_get_stats(gateway, &stats);
            printf("[Gateway] Requests: %llu | Errors: %llu\n",
                   stats.request_count, stats.error_count);
        }
    }

    // 4. 优雅关闭 (反向顺序)
    printf("\nShutting down...\n");

    monitoring_service_destroy(monitor);
    scheduler_service_destroy(scheduler);
    llm_service_destroy(llm);
    gateway_service_stop(gateway, false);
    gateway_service_destroy(gateway);

    svc_common_cleanup();
    printf("All services stopped.\n");

    return 0;
}
```

---

## 📝 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v0.0.4 | 2026-04-28 | Foundation Release - 完整API文档 |
| v0.0.3 | 2026-04-25 | 新增monit_d, market_d |
| v0.0.2 | 2026-04-20 | 核心用户态服务实现 |
| v0.0.1 | 2026-04-15 | 初始版本 |

---

## 🔗 相关文档

- [CoreLoopThree API](../20-core/01-coreloop-api.md)
- [协议适配指南](../70-toolkit/01-protocol-guide.md)
- [Docker 部署指南](../40-docker/README.md)
- [快速入门示例](../50-examples/01-quickstart.md)
