# SUBSYSTEM_gateway_d — 网关用户态服务

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/daemon/gateway_d/`  
**负责人**: Team C (基础设施)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

gateway_d 是 Airymax 的网关用户态服务，负责管理 HTTP/WebSocket/Stdio 三种协议的网关实例，是外部客户端与 Airymax 核心之间的统一入口。

### 1.1 核心职责

| 职责 | 模块 | 说明 |
|------|------|------|
| **HTTP 网关** | `http_gateway.h` | RESTful API 网关，提供 JSON 请求/响应 |
| **WebSocket 网关** | `ws_gateway.h` | 双向实时通信网关，支持流式响应 |
| **Stdio 网关** | `stdio_gateway.h` | 标准输入/输出网关，用于 CLI 工具集成 |
| **协议路由** | `gateway_protocol_router.h` | 多协议请求路由分发 |
| **MCP 服务** | `gateway_mcp_server.h` | Model Context Protocol 服务端 |
| **A2A 处理** | `gateway_a2a_handler.h` | Agent-to-Agent 协议处理 |
| **OpenAI 兼容** | `gateway_openai_compat.h` | OpenAI API 兼容层 |
| **服务生命周期** | `gateway_service.h` | 创建/初始化/启动/停止/销毁 |
| **请求转发** | `gateway_forward.h` | 请求转发到后端 daemon |
| **速率限制** | `gateway_rate_limiter.h` | 请求速率限制与流量控制 |
| **JSON-RPC** | `jsonrpc.h` | JSON-RPC 2.0 协议实现 |
| **系统调用路由** | `syscall_router.h` | 系统调用 API 路由 |

### 1.2 通信架构

```
CLI (Rust) ──HTTP──▶ gateway (C) ──内部路由──▶ gateway_d ──IPC Bus──▶ 各 daemon
```

| CLI 命令 | HTTP 路由 | 目标 |
|----------|----------|------|
| `agentrt run` | `POST /api/v1/agent/run` | CoreLoopThree |
| `agentrt config reload` | `POST /api/v1/config/reload` | config_manager |
| `agentrt llm list` | `GET /api/v1/llm/providers` | llm_d |
| `agentrt status` | `GET /api/v1/health` | monit_d |

### 1.3 不在范围内的职责

- 不负责 LLM 推理（由 llm_d 负责）
- 不负责工具执行（由 tool_d 负责）
- 不负责 Agent 市场管理（由 market_d 负责）
- 不负责底层协议转换（由 `agentos/gateway/` 协议转换层负责）

---

## 2. 输入/输出接口

### 2.1 服务生命周期 (`gateway_service.h`)

```c
// 创建/销毁
agentos_error_t gateway_service_create(gateway_service_t *service,
                                       const gateway_service_config_t *config);
void gateway_service_destroy(gateway_service_t service);

// 生命周期
agentos_error_t gateway_service_init(gateway_service_t service);
agentos_error_t gateway_service_start(gateway_service_t service);
agentos_error_t gateway_service_stop(gateway_service_t service, bool force);

// 状态查询
agentos_svc_state_t gateway_service_get_state(gateway_service_t service);
bool gateway_service_is_running(gateway_service_t service);
agentos_error_t gateway_service_get_stats(gateway_service_t service,
                                          agentos_svc_stats_t *stats);
agentos_error_t gateway_service_healthcheck(gateway_service_t service);

// 配置管理
agentos_error_t gateway_service_load_config(gateway_service_config_t *config,
                                            const char *config_path);
void gateway_service_get_default_config(gateway_service_config_t *config);
```

### 2.2 网关类型

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| HTTP | `GATEWAY_DAEMON_TYPE_HTTP` | RESTful API 网关 |
| WebSocket | `GATEWAY_DAEMON_TYPE_WS` | 双向实时通信 |
| Stdio | `GATEWAY_DAEMON_TYPE_STDIO` | 标准输入/输出 |

### 2.3 网关配置

```c
typedef struct {
    gateway_daemon_type_t type;   // 网关类型
    const char *host;             // 监听地址
    uint16_t port;                // 监听端口
    bool enabled;                 // 是否启用
    size_t max_request_size;      // 最大请求大小
    uint32_t timeout_ms;          // 超时时间
} gateway_daemon_config_t;

typedef struct {
    const char *name;             // 服务名称
    const char *version;          // 服务版本
    gateway_daemon_config_t http; // HTTP 网关配置
    gateway_daemon_config_t ws;   // WebSocket 网关配置
    gateway_daemon_config_t stdio;// Stdio 网关配置
    bool enable_metrics;          // 启用指标收集
    bool enable_tracing;          // 启用追踪
    uint32_t shutdown_timeout_ms; // 关闭超时
} gateway_service_config_t;
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | IPC 总线、内存管理、错误码 |
| **agentos/gateway/** | 强依赖 | 底层协议转换层 |
| **agentos/protocols/** | 强依赖 | 协议适配器（MCP/A2A/OpenAI） |
| **agentos/atoms/syscall/** | 强依赖 | 系统调用接口 |
| **daemon/common** | 强依赖 | 服务框架（`svc_common.h`） |
| **cupolas** | 可选依赖 | 输入净化、网络安全过滤 |

### 3.2 被依赖方

- **CLI/TUI**: 所有外部命令通过 HTTP API 调用
- **外部客户端**: 第三方集成通过网关访问
- **其他 daemon**: 通过 IPC Bus 接收路由请求

### 3.3 架构层级

```
gateway_d/          ← 用户态服务层（服务管理）
    ↓
agentos/gateway/   ← 协议转换层（HTTP/WS/Stdio 实现）
    ↓
agentos/atoms/syscall/ ← 系统调用层
```

---

## 4. 配置参数

### 4.1 服务配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `name` | 由调用者指定 | 服务名称 |
| `version` | 由调用者指定 | 服务版本 |
| `http.host` | 由调用者指定 | HTTP 监听地址 |
| `http.port` | 由调用者指定 | HTTP 监听端口 |
| `http.enabled` | true | 是否启用 HTTP |
| `http.max_request_size` | 由调用者指定 | 最大请求大小 |
| `http.timeout_ms` | 由调用者指定 | 请求超时 |
| `ws.enabled` | true | 是否启用 WebSocket |
| `stdio.enabled` | false | 是否启用 Stdio |
| `enable_metrics` | true | 启用指标收集 |
| `enable_tracing` | true | 启用追踪 |
| `shutdown_timeout_ms` | 由调用者指定 | 关闭超时 |

### 4.2 环境变量

| 变量 | 说明 |
|------|------|
| `TEST_GATEWAY_HTTP_PORT` | 测试环境 HTTP 端口（默认 8080） |
| `AGENTOS_TEST_MODE` | 测试模式标志 |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `gateway_service_create` | 否 | 创建时单线程 |
| `gateway_service_start` | 否 | 启动时单线程 |
| `gateway_service_stop` | 是 | 支持强制停止 |
| `gateway_service_get_state` | 是 | 只读查询 |
| `gateway_service_healthcheck` | 是 | 健康检查线程安全 |

---

## 6. 相关文档

- [通信协议规范](../../Capital_Specifications/agentos_contract/protocol_contract.md)
- [系统调用 API 规范](../../Capital_Specifications/agentos_contract/syscall_api_contract.md)
- [IPC 通信机制](../ipc.md)