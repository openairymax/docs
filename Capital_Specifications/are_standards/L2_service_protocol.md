# ARE Standards L2 — 服务通信协议标准

> **版本**: v0.1.0-draft | **状态**: 草案（v0.1.1 发行前冻结）| **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）
> **标准层级**: L2 | **回滚至**: [ARE Standards 总览](README.md)

---

## 1. 标准范围与目标

### 1.1 范围

本标准规定 Agent Runtime Environment（ARE）用户态守护进程（daemon）之间的**应用层通信协议**，包括：

- IPC Bus 统一消息头规范（128 字节定长结构体）；
- 消息类型与协议字段定义；
- JSON-RPC 2.0 方法命名空间规范；
- 12 个 ARE daemon 的方法命名空间清单；
- 服务发现多后端适配器接口；
- 跨进程 trace_id 贯穿规范；
- 背压控制与错误响应格式；
- 第三方实现兼容性指南。

本标准**不**规定：
- L1 内核级 IPC 原语（见 [L1_runtime_interface.md](L1_runtime_interface.md) §2.2）；
- L3 权限裁决、沙箱、错误码权威定义（见 [L3_security_governance.md](L3_security_governance.md)）；
- 具体传输介质（Unix socket / TCP / SHM，由实现者选择）。

### 1.2 目标

| 目标 | 描述 |
|------|------|
| **跨实现互操作** | 第三方实现的 daemon 可与 Airymax 原生 daemon 在同一 IPC Bus 上互通 |
| **协议自描述** | 消息头携带 magic/version/protocol，接收方可独立判别格式 |
| **可观测性** | trace_id 跨进程贯穿，全链路追踪零遗漏 |
| **弹性** | 内置背压、重试、熔断机制，过载时优雅降级而非雪崩 |
| **可演进** | 字段保留区允许向后兼容扩展，不破坏既有实现 |

### 1.3 与 L1 的关系

| 维度 | L1（内核级） | L2（应用级） |
|------|-------------|-------------|
| 消息结构 | `are_ipc_message_t`（40 字节，5 字段） | `are_ipc_message_header_t`（128 字节，14 字段） |
| 设计目标 | 极致性能、零依赖 | 自描述、可观测、跨进程 |
| 桥接 | L2 header 序列化后作为 L1 message 的 payload 传输 |
| 参考实现 | `atoms/corekern/include/ipc.h` | `daemon/common/include/ipc_service_bus.h` |

L2 消息在 L1 通道上传输时，整个 `are_ipc_message_header_t` + payload 作为一个 L1 `are_ipc_message_t.data` 传入；接收方按 magic 字段判别协议版本。

---

## 2. IPC Bus 统一消息头规范

### 2.1 结构体定义

```c
#include <stdint.h>

/* ==================== 常量 ==================== */
#define ARE_IPC_MAGIC          0x41524531u  /* "ARE1"，big-endian: 'A','R','E','1' */
#define ARE_IPC_VERSION        1
#define ARE_IPC_HEADER_SIZE    128           /* 定长，禁止变更（影响所有 ABI） */
#define ARE_IPC_MAX_SOURCE_LEN 32
#define ARE_IPC_MAX_TARGET_LEN 32
#define ARE_IPC_MAX_PAYLOAD    (512u * 1024u)  /* 512 KiB */

/* ==================== 消息类型 ==================== */
typedef enum {
    ARE_MSG_REQUEST   = 0,  /* 请求-响应模式的请求 */
    ARE_MSG_RESPONSE  = 1,  /* 请求-响应模式的响应 */
    ARE_MSG_NOTIFY    = 2,  /* 单向通知（无响应） */
    ARE_MSG_ERROR     = 3,  /* 错误响应（替代 RESPONSE 时表示失败） */
} are_msg_type_t;

/* ==================== 协议字段 ==================== */
typedef enum {
    ARE_PROTO_JSON_RPC = 0,  /* JSON-RPC 2.0，默认且强制支持 */
    ARE_PROTO_MCP      = 1,  /* Model Context Protocol */
    ARE_PROTO_A2A      = 2,  /* Agent-to-Agent Protocol */
    ARE_PROTO_OPENAI   = 3,  /* OpenAI 兼容 API */
    ARE_PROTO_CUSTOM   = 4,  /* 实现者自定义（payload 自描述） */
} are_proto_t;

/* ==================== flags 位定义 ==================== */
#define ARE_FLAG_COMPRESSED  0x0001u  /* payload 经 zlib 压缩 */
#define ARE_FLAG_ENCRYPTED   0x0002u  /* payload 经 TLS/AES-GCM 加密 */
#define ARE_FLAG_STREAMING   0x0004u  /* 流式消息分片（需重组） */
#define ARE_FLAG_DROPPABLE   0x0008u  /* 背压时可丢弃（日志/指标类） */
#define ARE_FLAG_IDEMPOTENT  0x0010u  /* 幂等请求，可安全重试 */
#define ARE_FLAG_TRACE_SAMPLED 0x0020u /* 该 trace 已被采样 */

/* ==================== 128 字节消息头 ==================== */
typedef struct are_ipc_message_header {
    uint32_t magic;            /* offset  0: 0x41524531 ("ARE1") */
    uint16_t version;          /* offset  4: 协议版本，当前 = 1 */
    uint16_t msg_type;         /* offset  6: are_msg_type_t */
    uint32_t protocol;         /* offset  8: are_proto_t */
    uint64_t msg_id;           /* offset 12: 消息唯一 ID（请求方分配） */
    uint64_t trace_id;         /* offset 20: 分布式追踪 ID，贯穿全链路 */
    uint64_t correlation_id;   /* offset 28: 请求-响应关联 ID */
    char     source[32];       /* offset 36: 源 daemon 名（NULL-terminated） */
    char     target[32];       /* offset 68: 目标 daemon 名 */
    uint32_t payload_len;      /* offset 100: 负载字节数（≤ ARE_IPC_MAX_PAYLOAD） */
    uint32_t flags;            /* offset 104: are_flag_t 位域 */
    uint64_t timestamp_ns;    /* offset 108: 纳秒时间戳（CLOCK_REALTIME） */
    uint32_t checksum;         /* offset 116: CRC32(header[0:116] + payload) */
    uint32_t reserved;         /* offset 120: 保留，必须为 0（未来扩展） */
    /* 总长度 = 124 字节，结构体内含 4 字节对齐填充至 128 */
} are_ipc_message_header_t;  /* _Static_assert(sizeof == 128) */

/* ==================== 完整消息 ==================== */
typedef struct {
    are_ipc_message_header_t header;
    void    *payload;          /* 调用者拥有所有权，长度 = header.payload_len */
    size_t   payload_size;     /* 实际分配容量，≥ payload_len */
} are_ipc_message_t;
```

> **字节序**：所有多字节整数采用**主机字节序**（native endian）。跨网络传输时由传输层（TLS/TCP）或显式序列化处理。CTS 包含大端/小端两套用例。

### 2.2 字段语义

| 字段 | 必填 | 语义 |
|------|------|------|
| `magic` | 是 | 接收方必须先校验 magic，不匹配则丢弃消息并记录 `WARN` |
| `version` | 是 | 接收方仅处理支持的主版本；不匹配返回 `ARE_EPROTO` |
| `msg_type` | 是 | REQUEST/RESPONSE/NOTIFY/ERROR 四选一 |
| `protocol` | 是 | 指示 payload 的序列化协议，决定反序列化器选择 |
| `msg_id` | 是 | 请求方分配的唯一 ID，RESPONSE 必须 echo 回 REQUEST 的 msg_id |
| `trace_id` | 是 | 0 表示未启用追踪；非 0 时全链路透传（见 §6） |
| `correlation_id` | 否 | 用于异步请求-响应匹配，可与 msg_id 相同 |
| `source` | 是 | 发送方 daemon 名（如 "llm_d"），便于审计与路由 |
| `target` | 是 | 目标 daemon 名（如 "tool_d"）；广播时填 "*" |
| `payload_len` | 是 | 0 表示无 payload；> ARE_IPC_MAX_PAYLOAD 拒绝 |
| `flags` | 是 | 位域，未定义位必须为 0 |
| `timestamp_ns` | 是 | 发送时刻的 CLOCK_REALTIME，用于审计与超时计算 |
| `checksum` | 是 | CRC32，覆盖 header[0:116] 与 payload；校验失败丢弃并 `WARN` |
| `reserved` | 是 | 必须 0；未来版本可定义新字段 |

### 2.3 一致性要求

- `sizeof(are_ipc_message_header_t)` 必须等于 128（CTS 用 `_Static_assert` 验证）；
- `checksum` 计算时，`checksum` 字段本身以 0 参与计算；
- 接收方在 magic/version/checksum 任一校验失败时**必须丢弃消息**，不得回复 ERROR（避免错误风暴）；
- `source` 与 `target` 必须以 `\0` 结尾，长度 ≤ 31 字节（保留 1 字节终结符）。

---

## 3. 消息类型

### 3.1 REQUEST / RESPONSE

请求-响应模式，客户端发送 REQUEST，服务端必须返回 RESPONSE 或 ERROR。

```jsonc
// REQUEST payload（JSON-RPC 2.0）
{
  "jsonrpc": "2.0",
  "method": "llm.complete",
  "params": {"model": "gpt-4", "messages": [...]},
  "id": 42
}

// RESPONSE payload
{
  "jsonrpc": "2.0",
  "result": {"content": "...", "tokens": 128},
  "id": 42
}

// ERROR payload（msg_type=ARE_MSG_ERROR）
{
  "jsonrpc": "2.0",
  "error": {"code": -403, "message": "rate limit exceeded"},
  "id": 42
}
```

**RESPONSE 的 header 约束**：
- `msg_type` = `ARE_MSG_RESPONSE` 或 `ARE_MSG_ERROR`；
- `msg_id` 必须 echo REQUEST 的 `msg_id`；
- `source` = 原 `target`，`target` = 原 `source`；
- `trace_id` 必须 echo REQUEST 的 `trace_id`。

### 3.2 NOTIFY

单向通知，无响应。用于心跳、事件广播、指标推送。

```jsonc
{
  "jsonrpc": "2.0",
  "method": "monit.heartbeat",
  "params": {"cpu": 0.42, "mem_mb": 1024}
  // 无 "id" 字段
}
```

**NOTIFY 的 header 约束**：
- `msg_type` = `ARE_MSG_NOTIFY`；
- 接收方不得返回任何 RESPONSE；
- `flags` 可含 `ARE_FLAG_DROPPABLE`，背压时可丢弃。

### 3.3 ERROR

错误响应。`msg_type=ARE_MSG_ERROR` 时，payload 必须是标准 JSON-RPC error 对象（见 §8）。

### 3.4 消息类型矩阵

| 场景 | 客户端发送 | 服务端响应 | 是否需 ack |
|------|-----------|-----------|-----------|
| 同步 RPC | REQUEST | RESPONSE/ERROR | 是（隐式） |
| 异步 RPC | REQUEST（flags 含 IDEMPOTENT） | RESPONSE（任意时刻） | 是 |
| 单向通知 | NOTIFY | 无 | 否 |
| 事件广播 | NOTIFY（target="*"） | 无 | 否 |
| 心跳 | NOTIFY（method="monit.heartbeat"） | 无 | 否 |

---

## 4. 协议字段

### 4.1 JSON-RPC 2.0（默认，强制支持）

所有实现者必须支持 `ARE_PROTO_JSON_RPC`。payload 为标准 [JSON-RPC 2.0](https://www.jsonrpc.org/specification) 报文，UTF-8 编码。

约束：
- `method` 必须遵循 §5 命名空间规范；
- `id` 必须是整数、字符串或 null；NOTIFY 不含 `id`；
- `params` 可为对象（named params）或数组（positional params）；
- 错误码使用 ARE 错误码体系（见 §8 与 L3 标准）。

### 4.2 MCP（Model Context Protocol）

`ARE_PROTO_MCP` 用于与 MCP 兼容客户端通信。payload 为 MCP 消息（JSON-RPC 2.0 子集 + MCP 扩展字段）。gateway_d 负责在 MCP 与 ARE 内部 JSON-RPC 之间转换。

### 4.3 A2A（Agent-to-Agent）

`ARE_PROTO_A2A` 用于 Agent 间通信，遵循 [A2A Protocol](https://a2a-protocol.org/) 规范。a2a_d daemon 负责协议适配。

### 4.4 OpenAI 兼容

`ARE_PROTO_OPENAI` 用于 OpenAI API 兼容接入。gateway_d 将 OpenAI 风格请求转换为内部 `llm.complete` 调用。

### 4.5 Custom

`ARE_PROTO_CUSTOM` 允许实现者自定义 payload 格式。使用时 `flags` 必须额外设置 `ARE_FLAG_COMPRESSED` 或在 payload 头部携带自描述元数据。CTS 不覆盖 Custom 协议，但要求 magic/version/checksum 仍校验。

---

## 5. JSON-RPC 2.0 方法命名空间规范

### 5.1 强制规则

**所有 JSON-RPC `method` 字段必须采用 `<daemon>.<method>` 两段式命名空间格式**，其中：

- `<daemon>` 是 §6 列出的 12 个 daemon 名之一（小写，不含 `_d` 后缀的简写形式）；
- `<method>` 是该 daemon 内的具体方法名，使用 `snake_case`；
- **禁止**使用扁平方法名（如 `complete`、`execute`、`health_check`）；
- **禁止**使用三段及以上命名（如 `llm.d.complete`）；如需分组，用 `<method>` 中的下划线（如 `tool.execute_sandboxed`）。

### 5.2 正确与错误示例

| 正确 | 错误 | 原因 |
|------|------|------|
| `llm.complete` | `complete` | 缺少命名空间 |
| `tool.execute` | `tools.execute` | daemon 名错（应为 `tool`，不是 `tools`） |
| `sched.health_check` | `sched.d.health_check` | 命名空间不应含 `_d` |
| `monit.alert_resolve` | `monit.alert.resolve` | 三段命名，应合并为 `alert_resolve` |
| `cupolas.permission_check` | `permission.check` | 缺少 daemon 前缀 |

### 5.3 方法名约定

- 一律小写 ASCII，下划线分词；
- 动词在前、名词在后（`tool.execute`、`mem.search`、`agent.spawn`）；
- 查询类方法以 `get_`/`list_`/`health_check` 开头；
- 变更类方法以 `create_`/`update_`/`delete_`/`register_`/`unregister_` 开头。

### 5.4 版本演进

方法一经发布即冻结语义。如需破坏性变更：
1. 新增 `<method>_v2`（如 `llm.complete_v2`）；
2. 旧方法标记 `deprecated`，在 spec 中声明弃用周期（≥ 6 个月）；
3. 弃用期满后旧方法可返回 `ARE_ENOTSUP`，但不得删除方法名占用。

---

## 6. 12 个 daemon 的方法命名空间清单

下表为 ARE 标准注册的 12 个 daemon 及其方法命名空间。每个 daemon 名对应 `source`/`target` 字段的合法取值（不含 `_d` 后缀的简写形式用于 `method` 命名空间，完整 daemon 进程名含 `_d` 后缀用于 `source`/`target`）。

| # | daemon 进程名 | 命名空间 | 职责 | 标准方法示例 |
|---|--------------|---------|------|-------------|
| 1 | `llm_d` | `llm.*` | 大语言模型调用、Token 计数、成本追踪、响应缓存 | `llm.complete`, `llm.count_tokens`, `llm.list_models`, `llm.health_check` |
| 2 | `tool_d` | `tool.*` | 工具注册/发现、沙箱执行、参数校验、结果缓存 | `tool.execute`, `tool.execute_sandboxed`, `tool.register`, `tool.list`, `tool.health_check` |
| 3 | `sched_d` | `sched.*` | 任务分发、4 种调度策略（轮询/加权/优先级/ML）、检查点 | `sched.submit`, `sched.query`, `sched.cancel`, `sched.checkpoint_save`, `sched.health_check` |
| 4 | `mem_d` | `mem.*` | 长期记忆写入/检索/演化、向量索引 | `mem.write`, `mem.search`, `mem.get`, `mem.delete`, `mem.evolve`, `mem.health_check` |
| 5 | `agent_d` | `agent.*` | Agent 实例生命周期、Agent 调用 | `agent.spawn`, `agent.terminate`, `agent.invoke`, `agent.list`, `agent.health_check` |
| 6 | `a2a_d` | `a2a.*` | Agent-to-Agent 协议适配 | `a2a.send`, `a2a.receive`, `a2a.register_agent`, `a2a.health_check` |
| 7 | `mcp_d` | `mcp.*` | MCP 协议适配、工具资源暴露 | `mcp.handle_request`, `mcp.list_resources`, `mcp.health_check` |
| 8 | `hook_d` | `hook.*` | 生命周期钩子注册与触发 | `hook.register`, `hook.unregister`, `hook.trigger`, `hook.health_check` |
| 9 | `plugin_d` | `plugin.*` | 插件加载/卸载/依赖管理 | `plugin.install`, `plugin.uninstall`, `plugin.list`, `plugin.health_check` |
| 10 | `cupolas_d` | `cupolas.*` | 权限裁决、输入净化、审计 | `cupolas.permission_check`, `cupolas.sanitize_input`, `cupolas.audit_query`, `cupolas.health_check` |
| 11 | `monit_d` | `monit.*` | 指标采集、健康检查、告警 | `monit.heartbeat`, `monit.metrics`, `monit.alert_raise`, `monit.alert_resolve`, `monit.health_check` |
| 12 | `market_d` | `market.*` | Agent/Skill/Tool/Template 资源管理 | `market.install`, `market.publish`, `market.search`, `market.health_check` |

### 6.1 标准方法语义约束

下列方法在所有 daemon 上的语义必须一致：

| 方法 | 语义 | 返回 |
|------|------|------|
| `<ns>.health_check` | 健康检查，无副作用 | `{"status": "ok"/"degraded"/"down", "version": "..."}` |
| `<ns>.get_stats` | 查询统计 | daemon 自定义统计对象 |
| `<ns>.shutdown` | 优雅停止（仅 monit_d 可调用其他 daemon） | `{"drained": true}` |

### 6.2 命名空间扩展

第三方实现者如需注册新 daemon，须向 ARE 标准委员会申请命名空间（避免冲突）。未注册前可使用 `x_<vendor>.*` 前缀（如 `x_acme.cache_get`），CTS 不覆盖此类方法但保留互操作性。

---

## 7. 服务发现多后端适配器接口

### 7.1 设计目标

Airymax 原生服务发现基于共享内存（SHM），单机性能优秀但无法跨节点。L2 标准定义**统一适配器接口**，允许实现者接入任意后端（shm/dns-sd/consul/etcd/k8s），实现跨节点服务发现。

### 7.2 五种后端

| 后端 | name 标识 | 适用场景 | 一致性 |
|------|----------|----------|--------|
| 共享内存 | `shm` | 单机高性能，Airymax 默认 | 强一致 |
| DNS-SD/mDNS | `dns-sd` | 零配置局域网 | 最终一致 |
| HashiCorp Consul | `consul` | 云原生、跨节点 | 强一致（Raft） |
| CNCF etcd | `etcd` | K8s 原生、跨节点 | 强一致（Raft） |
| Kubernetes API | `k8s` | K8s 集群内原生服务 | 强一致（etcd 后端） |

### 7.3 统一适配器接口

```c
/* ==================== 服务实例条目 ==================== */
typedef struct {
    char     service_name[64];    /* 如 "llm_d" */
    char     instance_id[64];     /* 实例唯一标识 */
    char     endpoint[256];      /* 如 "unix:/var/run/llm_d.sock" 或 "tcp://10.0.0.1:8080" */
    uint16_t port;                /* 0 表示端点已编码在 endpoint 中 */
    uint32_t weight;              /* 负载均衡权重，0=默认 */
    bool     healthy;             /* 健康状态 */
    uint64_t last_heartbeat_ns;  /* 最后心跳时间 */
    uint32_t pid;                 /* 进程 ID，0=N/A */
} are_svc_entry_t;

/* ==================== 适配器接口 ==================== */
typedef struct are_svc_discovery_adapter {
    const char *name;            /* "shm"/"dns-sd"/"consul"/"etcd"/"k8s" */

    /* 注册服务实例。返回 ARE_OK / ARE_EEXIST / ARE_EINVAL。 */
    are_error_t (*register_service)(const char *service_name,
                                    const char *endpoint,
                                    uint16_t port);

    /* 发现服务：返回健康实例列表。
     * entries 由调用者分配，*count 输入时为容量、输出时为实际数量。
     * 若实际数量 > 容量，返回 ARE_EOVERFLOW 且 *count 为实际数量。 */
    are_error_t (*discover_service)(const char *service_name,
                                    are_svc_entry_t *entries,
                                    size_t *count);

    /* 注销服务实例。 */
    are_error_t (*unregister_service)(const char *service_name,
                                      const char *instance_id);

    /* 列出所有已注册服务（管理用途）。 */
    are_error_t (*list_services)(are_svc_entry_t *entries, size_t *count);

    /* 心跳：更新 last_heartbeat_ns，防止被自动过期。
     * 不支持心跳的后端（dns-sd）可返回 ARE_ENOTSUP。 */
    are_error_t (*heartbeat)(const char *service_name, const char *instance_id);

    /* 适配器私有上下文，由适配器 init 时填充。 */
    void       *context;
} are_svc_discovery_adapter_t;
```

### 7.4 后端选择与配置

```yaml
service_discovery:
  backend: shm              # 默认，单机高性能
  # backend: dns-sd         # 局域网零配置
  # backend: consul          # 云原生
  # backend: etcd            # K8s 原生
  # backend: k8s              # K8s API

  shm:
    name: /agentos_service_registry
    size: 1048576            # 1 MiB
    heartbeat_interval_ms: 10000
    expire_timeout_ms: 30000

  dns-sd:
    service_type: _are-daemon._tcp
    domain: local

  consul:
    address: http://consul:8500
    token: ${CONSUL_ACL_TOKEN}    # ACL token，环境变量注入
    health_check_path: /health

  etcd:
    endpoints: [http://etcd-0:2379, http://etcd-1:2379]
    prefix: /are/services/
    lease_ttl_s: 30

  k8s:
    namespace: agentrt
    service_label: are-daemon
    incluster: true            # 使用 ServiceAccount token
```

### 7.5 适配器一致性要求

| 要求 | 描述 |
|------|------|
| **超时统一** | `discover_service` 必须在 100 ms 内返回（本地缓存或后端查询），超时返回 `ARE_ETIMEDOUT` |
| **健康传播** | 适配器必须传播 `healthy=false` 给调用方；调用方据此剔除实例 |
| **零依赖切换** | 切换后端只需改配置文件，无需改代码；上层 daemon 不感知后端实现 |
| **失败回退** | 后端不可达时，适配器应返回 `ARE_EUNAVAILABLE`，不得崩溃 |

### 7.6 参考实现

Airymax 原生实现：`agentos/daemon/common/include/service_discovery.h`（shm 后端）。第三方实现其他后端时需实现 `are_svc_discovery_adapter_t` 全部函数指针（除 `heartbeat` 可返回 `ARE_ENOTSUP`）。

---

## 8. 错误响应格式标准化

### 8.1 JSON-RPC 2.0 error 对象

所有 `ARE_MSG_ERROR` 的 payload 必须是合法的 JSON-RPC 2.0 error 对象：

```jsonc
{
  "jsonrpc": "2.0",
  "error": {
    "code": -403,                    /* ARE 错误码，见 L3 §5 */
    "message": "rate limit exceeded", /* 人类可读描述（英文） */
    "data": {                        /* 可选，附加诊断信息 */
      "retry_after_ms": 5000,
      "trace_id": "01HX...",
      "details": "..."
    }
  },
  "id": 42                            /* 必须 echo REQUEST 的 id */
}
```

### 8.2 错误码分段（与 L3 一致）

| 段 | 范围 | 含义 | 示例 |
|----|------|------|------|
| 通用 | -1 ~ -99 | 参数、内存、权限等基础错误 | `ARE_EINVAL=-1` |
| 系统 | -100 ~ -199 | 平台、线程、socket | `ARE_ERR_SYS_MUTEX=-105` |
| 内核 | -200 ~ -299 | IPC、任务、调度 | `ARE_ERR_KERN_IPC=-201` |
| 服务 | -300 ~ -399 | 服务状态、依赖、负载均衡 | `ARE_ERR_SVC_BUSY=-302` |
| LLM | -400 ~ -499 | LLM provider、限流、上下文 | `ARE_ERR_LLM_RATE_LIMIT=-403` |
| 执行 | -500 ~ -599 | 工具执行、沙箱、超时 | `ARE_ERR_EXEC_SANDBOX=-505` |
| 记忆 | -600 ~ -699 | 记忆读写、查询、演化 | `ARE_ERR_MEM_FULL=-605` |
| 安全 | -700 ~ -799 | 权限、净化、审计 | `ARE_ERR_SEC_VIOLATION=-701` |
| 协议 | -800 ~ -889 | 编排、协调、补偿 | `ARE_ERR_COORD_PLAN_FAIL=-801` |

> 完整权威定义见 L3 标准《第 5 节 统一错误码体系》及参考实现 `commons/utils/error/include/error.h`。

### 8.3 错误响应示例

**权限不足**（method=`tool.execute`）：

```jsonc
{
  "jsonrpc": "2.0",
  "error": {
    "code": -701,
    "message": "permission denied: tool 'rm' not allowed for agent 'agent_42'",
    "data": {"subject": "agent_42", "action": "execute", "resource": "tool:rm"}
  },
  "id": 42
}
```

**服务繁忙**（背压触发，建议重试）：

```jsonc
{
  "jsonrpc": "2.0",
  "error": {
    "code": -302,
    "message": "service busy, retry later",
    "data": {"retry_after_ms": 2000, "backpressure_level": "DROP"}
  },
  "id": 42
}
```

**LLM 限流**：

```jsonc
{
  "jsonrpc": "2.0",
  "error": {
    "code": -403,
    "message": "LLM provider rate limit exceeded",
    "data": {"provider": "openai", "reset_at": "2026-07-04T12:00:00Z"}
  },
  "id": 42
}
```

---

## 9. trace_id 跨进程贯穿规范

### 9.1 trace_id 格式

`trace_id` 是 64 位无符号整数，在 `are_ipc_message_header_t.trace_id` 字段中传递。生成规则：

- 高 32 位：进程启动时的随机种子（每进程唯一）；
- 低 32 位：进程内单调递增计数器。

或采用 ULID 风格的 128 位字符串编码（在 `data.trace_id` 字段中补充），但 `header.trace_id` 必须填充 64 位哈希以保持定长。

### 9.2 传播规则

| 场景 | trace_id 处理 |
|------|--------------|
| 客户端发起请求 | 生成新 trace_id，填入 REQUEST.header.trace_id |
| 服务端接收 REQUEST | 从 header 读取 trace_id，注入到本地日志上下文（thread-local） |
| 服务端返回 RESPONSE | echo REQUEST 的 trace_id 到 RESPONSE.header |
| 服务端发起下游调用 | 复用当前 trace_id（不得生成新值） |
| NOTIFY 消息 | 客户端生成新 trace_id（除非属于现有链路） |
| 错误响应 | echo REQUEST 的 trace_id |
| 跨进程边界（gateway_d 转发） | 必须保留 trace_id；若外部协议无对应字段，存入 `data.trace_id` |

### 9.3 采样

`flags & ARE_FLAG_TRACE_SAMPLED` 表示该 trace 已被采样，observe_d 据此决定是否采集完整 span。未采样的 trace 仅记录必要的日志（INFO 级），减少开销。

### 9.4 一致性要求

- 同一请求链路上所有消息的 `trace_id` 必须**完全一致**；
- CTS 用例：客户端发 REQUEST → 服务端 A 调用服务端 B → B 返回 → A 返回客户端，全链路 4 个消息的 trace_id 相等；
- trace_id=0 表示"未启用追踪"，允许但不推荐（生产环境必须启用）。

---

## 10. 背压控制

### 10.1 三级背压策略

当 daemon 队列深度上升时，按下列阈值触发分级响应（参考实现：`daemon/common/include/ipc_backpressure.h`）：

| 级别 | 阈值 | 行为 |
|------|------|------|
| `ARE_BP_NORMAL` | < 60% | 正常处理 |
| `ARE_BP_SLOW` | 60–80% | 生产者降速：客户端收到 `ARE_ERR_SVC_BUSY` + `retry_after_ms`，建议指数退避 |
| `ARE_BP_DROP` | 80–90% | 丢弃 `ARE_FLAG_DROPPABLE` 消息（日志/指标/心跳类），不回复 ERROR |
| `ARE_BP_REJECT` | > 90% | 拒绝新连接，已建立连接的 REQUEST 返回 `ARE_ERR_SVC_BUSY`，触发告警 |
| 恢复 | < 60% | 恢复正常速率，记录恢复事件 |

### 10.2 退避策略

客户端收到 `ARE_ERR_SVC_BUSY` 时，**必须**遵循退避策略，不得立即重试：

```c
/* 客户端退避算法（伪代码） */
uint32_t backoff_ms = 100;          /* 初始 100ms */
for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
    are_error_t rc = are_ipc_call(...);
    if (rc == ARE_OK) break;
    if (rc == ARE_ERR_SVC_BUSY && (flags & ARE_FLAG_IDEMPOTENT)) {
        /* 指数退避 + 抖动 */
        uint32_t jitter = rand() % 50;
        are_time_sleep_ms(backoff_ms + jitter);
        backoff_ms = MIN(backoff_ms * 2, 5000);  /* 上限 5s */
        continue;
    }
    break;  /* 非幂等或非繁忙错误，不重试 */
}
```

约束：
- **仅幂等请求**（`flags & ARE_FLAG_IDEMPOTENT`）允许重试，非幂等请求重试可能导致重复执行；
- `data.retry_after_ms` 优先于客户端本地退避计算；
- 最大重试次数默认 3 次（与 `IPC_BUS_MAX_RETRIES` 一致）；
- 重试时**必须保留**原 `msg_id` 与 `trace_id`。

### 10.3 背压控制器接口

```c
typedef enum {
    ARE_BP_NORMAL  = 0,
    ARE_BP_SLOW    = 1,
    ARE_BP_DROP    = 2,
    ARE_BP_REJECT  = 3,
} are_bp_level_t;

typedef struct {
    size_t   queue_capacity;          /* 队列容量（消息数） */
    uint32_t slow_threshold_pct;     /* 默认 60 */
    uint32_t drop_threshold_pct;     /* 默认 80 */
    uint32_t reject_threshold_pct;  /* 默认 90 */
    uint32_t recover_threshold_pct; /* 默认 60 */
    uint32_t sample_interval_ms;     /* 默认 5000 */
} are_bp_config_t;

typedef struct are_bp_controller are_bp_controller_t;

are_bp_controller_t *are_bp_create(const are_bp_config_t *cfg);
void                 are_bp_destroy(are_bp_controller_t *ctrl);
are_bp_level_t       are_bp_update(are_bp_controller_t *ctrl, size_t current_depth);
bool                 are_bp_should_send(are_bp_controller_t *ctrl, bool is_droppable);
bool                 are_bp_should_accept_connection(are_bp_controller_t *ctrl);
```

---

## 11. 兼容性策略与版本演进

### 11.1 版本号

L2 标准采用与 L1 一致的双重版本号（spec semver + ABI 整数）：

```c
#define ARE_L2_SPEC_MAJOR 0
#define ARE_L2_SPEC_MINOR 1
#define ARE_L2_SPEC_PATCH 0
#define ARE_L2_ABI_VERSION 1
```

### 11.2 字段保留区演进

`reserved` 字段（4 字节）允许未来扩展：

- v1：必须为 0；
- v2：若需新增字段（如 `tenant_id`），从 `reserved` 切出，并递增 `version`；
- v1 实现者收到 v2 消息时，必须忽略未知 `reserved` 内容（前向兼容）。

### 11.3 协议字段演进

新增 `are_proto_t` 枚举值是**向后兼容**变更（不递增 ABI）；修改现有枚举值含义是**破坏性**变更（递增 ABI + MAJOR）。

### 11.4 命名空间演进

新增 daemon 命名空间（如 `cache.*`）是**向后兼容**变更；重命名现有命名空间是**破坏性**变更，必须提供 6 个月双名共存期。

---

## 12. 第三方实现指南

### 12.1 实现合规 daemon 的步骤

1. **选择传输介质**：Unix socket（推荐，单机）、TCP（跨机）、SHM（高性能数据通道）。
2. **实现消息编解码**：按 §2 定义序列化/反序列化 128 字节 header + payload。
3. **实现服务发现**：选择 §7 五种后端之一，实现 `are_svc_discovery_adapter_t`。
4. **注册命名空间**：向 ARE 标准委员会申请 `<ns>`（或使用 `x_<vendor>.*` 临时前缀）。
5. **实现标准方法**：至少实现 `<ns>.health_check`、`<ns>.get_stats`、`<ns>.shutdown`。
6. **接入 IPC Bus**：启动时调用 `register_service`，停止时调用 `unregister_service`。
7. **通过 CTS**：运行 L2 一致性测试套件，全部用例通过。

### 12.2 最小合规 daemon 骨架

```c
/* myd.c — 第三方最小合规 daemon 示例 */
#include "are_l2.h"

static are_svc_discovery_adapter_t *sd;

static are_error_t handle_health_check(const are_ipc_message_t *req,
                                       are_ipc_message_t *resp) {
    const char *body = "{\"status\":\"ok\",\"version\":\"myd-0.1\"}";
    return are_ipc_build_response(resp, req, body, strlen(body));
}

int main(void) {
    /* 1. 初始化服务发现 */
    sd = are_sd_create("shm", NULL);  /* 使用 SHM 后端 */

    /* 2. 注册服务 */
    sd->register_service("myd", "unix:/var/run/myd.sock", 0);

    /* 3. 创建 IPC 通道 */
    are_ipc_channel_t *ch;
    are_ipc_create_channel("/var/run/myd.sock", &ch);

    /* 4. 注册方法分派表 */
    are_method_dispatch_t dispatch[] = {
        {"myd.health_check", handle_health_check},
        /* ... */
    };

    /* 5. 事件循环 */
    while (running) {
        are_ipc_message_t msg;
        if (are_ipc_recv(ch, 1000, &msg) == ARE_OK) {
            are_dispatch_method(dispatch, &msg);
        }
    }

    /* 6. 清理 */
    sd->unregister_service("myd", "instance-1");
    are_ipc_close(ch);
    are_sd_destroy(sd);
    return 0;
}
```

### 12.3 一致性测试用例（L2 CTS 摘要）

| 用例 ID | 测试要点 | 通过判据 |
|---------|----------|----------|
| L2-CTS-001 | 发送 magic 不匹配的消息 | 接收方丢弃并记录 WARN |
| L2-CTS-002 | 发送 checksum 错误的消息 | 接收方丢弃，不回复 ERROR |
| L2-CTS-003 | REQUEST → RESPONSE 链路 | RESPONSE.msg_id == REQUEST.msg_id |
| L2-CTS-004 | trace_id 跨 4 个 daemon 传播 | 全部消息 trace_id 相等 |
| L2-CTS-005 | 方法名 `complete`（无命名空间） | 返回 `ARE_ERR_PARSE_ERROR` |
| L2-CTS-006 | 方法名 `tools.execute`（错命名空间） | 返回 method-not-found |
| L2-CTS-007 | 队列 85% 时发 droppable NOTIFY | 丢弃，不回复 |
| L2-CTS-008 | 队列 92% 时发 REQUEST | 返回 `ARE_ERR_SVC_BUSY` + retry_after_ms |
| L2-CTS-009 | 切换 sd 后端 shm→consul | 上层 daemon 无感知，行为一致 |
| L2-CTS-010 | ERROR 响应格式校验 | payload 含合法 JSON-RPC error 对象 |
| L2-CTS-011 | `header.sizeof == 128` | _Static_assert 通过 |
| L2-CTS-012 | NOTIFY 无 RESPONSE | 客户端不收到任何回复 |

---

## 13. 参考实现

| 标准接口 | 参考实现位置 |
|----------|-------------|
| `are_ipc_message_header_t` | `agentos/daemon/common/include/ipc_service_bus.h`（`ipc_bus_message_header_t`） |
| 消息类型枚举 | `agentos/daemon/common/include/ipc_service_bus.h`（`ipc_bus_msg_type_t`） |
| 协议字段枚举 | `agentos/daemon/common/include/ipc_service_bus.h`（`ipc_bus_proto_t`） |
| 服务发现 shm 后端 | `agentos/daemon/common/include/service_discovery.h`（`sd_*`） |
| 背压控制器 | `agentos/daemon/common/include/ipc_backpressure.h`（`ipc_bp_*`） |
| JSON-RPC 辅助 | `agentos/daemon/common/include/jsonrpc_helpers.h` |
| 方法分派器 | `agentos/daemon/common/include/method_dispatcher.h` |

### 13.1 与原生实现的差异

Airymax 原生 `ipc_bus_message_header_t` 字段顺序与本标准略有不同（原生 `reserved[16]`，本标准 `reserved` 4 字节 + 对齐）。本标准冻结字段布局为 v1，原生实现需通过适配层桥接或迁移至本标准 ABI。

---

## 14. 参考文档

- [ARE Standards 总览](README.md)
- [L1 核心运行时接口](L1_runtime_interface.md) — IPC 原语、内存、任务
- [L3 安全与治理](L3_security_governance.md) — 错误码权威定义、权限、审计
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [MCP Specification](https://modelcontextprotocol.io/)
- [A2A Protocol](https://a2a-protocol.org/)

---

<div align="center">

**© 2026 SPHARX Ltd.** | 本文档及参考实现均以 AGPL v3 + Apache 2.0 双许可证发布（SPDX: AGPL-3.0-or-later OR Apache-2.0）。

</div>
