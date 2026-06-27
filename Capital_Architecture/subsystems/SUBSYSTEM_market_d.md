# SUBSYSTEM_market_d — 市场服务用户态服务

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/daemon/market_d/`  
**负责人**: Team B (生态 + SDK)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

market_d 是 Airymax 的市场服务用户态服务，负责 Agent 和 Skill 的注册、发现、安装和管理。它类似于应用商店，为 Airymax 生态系统提供组件分发能力。

### 1.1 核心职责

| 职责 | 模块 | 说明 |
|------|------|------|
| **Agent 注册** | `agent_registry_core.h` | Agent 注册/搜索/安装/卸载 |
| **Skill 注册** | `market_service.h` | Skill 注册/搜索/安装/卸载 |
| **远程注册中心** | 同步机制 | 与远程注册中心同步 |
| **版本管理** | 更新检查 | 检查更新、版本比对 |
| **缓存管理** | 缓存 | 注册信息缓存与过期管理 |
| **服务适配** | `market_svc_adapter.h` | 对接外部服务调用 |

### 1.2 Agent 类型

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| 助手型 | `AGENT_TYPE_ASSISTANT` | 通用助手 Agent |
| 专家型 | `AGENT_TYPE_EXPERT` | 领域专家 Agent |
| 专业型 | `AGENT_TYPE_SPECIALIZED` | 特定任务 Agent |
| 自定义 | `AGENT_TYPE_CUSTOM` | 用户自定义 Agent |

### 1.3 Skill 类型

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| 工具型 | `SKILL_TYPE_TOOL` | 工具类 Skill |
| 知识型 | `SKILL_TYPE_KNOWLEDGE` | 知识库 Skill |
| 集成型 | `SKILL_TYPE_INTEGRATION` | 第三方集成 Skill |
| 自定义 | `SKILL_TYPE_CUSTOM` | 自定义 Skill |

### 1.4 不在范围内的职责

- 不负责 Agent 运行时执行（由 coreloopthree 负责）
- 不负责工具执行（由 tool_d 负责）
- 不负责 LLM 推理（由 llm_d 负责）
- 不负责 CLI 命令行工具（CLI 通过 gateway 调用）

---

## 2. 输入/输出接口

### 2.1 服务生命周期

```c
int market_service_create(const market_config_t *config, market_service_t **service);
int market_service_destroy(market_service_t *service);
```

### 2.2 Agent 管理

```c
int market_service_register_agent(market_service_t *service, const agent_info_t *info);
int market_service_search_agents(market_service_t *service, const search_params_t *params,
                                 agent_info_t ***agents, size_t *count);
int market_service_install_agent(market_service_t *service, const install_request_t *req,
                                 install_result_t **result);
int market_service_uninstall_agent(market_service_t *service, const char *agent_id);
int market_service_get_installed_agents(market_service_t *service, agent_info_t ***agents,
                                        size_t *count);
```

### 2.3 Skill 管理

```c
int market_service_register_skill(market_service_t *service, const skill_info_t *info);
int market_service_search_skills(market_service_t *service, const search_params_t *params,
                                 skill_info_t ***skills, size_t *count);
int market_service_install_skill(market_service_t *service, const install_request_t *req,
                                 install_result_t **result);
int market_service_uninstall_skill(market_service_t *service, const char *skill_id);
int market_service_get_installed_skills(market_service_t *service, skill_info_t ***skills,
                                        size_t *count);
```

### 2.4 更新与同步

```c
int market_service_check_update(market_service_t *service, const char *id,
                                bool *has_update, char **latest_version);
int market_service_reload_config(market_service_t *service, const market_config_t *config);
int market_service_sync_registry(market_service_t *service);
```

### 2.5 数据模型

**Agent 信息**:
```c
typedef struct {
    char *agent_id;          // Agent ID
    char *name;              // Agent 名称
    char *version;           // 版本
    char *description;       // 描述
    agent_type_t type;       // Agent 类型
    agent_status_t status;   // Agent 状态
    char *author;            // 作者
    char *repository;        // 仓库地址
    char *dependencies;      // 依赖项
    float rating;            // 评分
    uint32_t download_count; // 下载次数
    uint64_t last_updated;   // 最后更新时间
} agent_info_t;
```

**搜索参数**:
```c
typedef struct {
    char *query;             // 搜索关键词
    agent_type_t agent_type; // Agent 类型过滤
    skill_type_t skill_type; // Skill 类型过滤
    bool only_installed;     // 仅显示已安装
    bool sort_by_rating;     // 按评分排序
    bool sort_by_download;   // 按下载量排序
    size_t limit;            // 结果数量限制
    size_t offset;           // 结果偏移量
} search_params_t;
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | IPC 总线、内存管理、错误码 |
| **daemon/common** | 强依赖 | 服务框架（`svc_common.h`） |
| **远程注册中心** | 外部服务 | Agent/Skill 注册中心 API |
| **heapstore** | 可选依赖 | 本地缓存持久化 |

### 3.2 被依赖方

- **CLI**: 通过 gateway 调用市场搜索/安装命令
- **coreloopthree**: Agent 发现与能力查询
- **orchestrator**: 编排时需要查询可用 Agent

### 3.3 市场架构

```
┌──────────────────────────────────────────┐
│              market_d                     │
│  ┌────────────┐  ┌────────────────────┐  │
│  │ Agent 注册  │  │  Skill 注册        │  │
│  └────────────┘  └────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │         远程注册中心同步            │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  远程注册中心    │
│  (Agent/Skill)  │
└─────────────────┘
```

---

## 4. 配置参数

### 4.1 市场配置

```c
typedef struct {
    char *registry_url;           // 注册中心 URL
    char *storage_path;           // 存储路径
    uint32_t sync_interval_ms;    // 同步间隔（毫秒）
    uint32_t cache_ttl_ms;        // 缓存过期时间（毫秒）
    bool enable_remote_registry;  // 是否启用远程注册中心
    bool enable_auto_update;      // 是否启用自动更新
} market_config_t;
```

### 4.2 安装请求

```c
typedef struct {
    char *id;            // Agent 或 Skill ID
    char *version;       // 版本（为空表示最新版本）
    bool force_update;   // 是否强制更新
    char *install_path;  // 安装路径（可选）
} install_request_t;
```

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `market_service_create` | 否 | 创建时单线程 |
| `market_service_register_*` | 是 | 注册操作线程安全 |
| `market_service_search_*` | 是 | 只读操作线程安全 |
| `market_service_install_*` | 部分 | 安装操作互斥 |
| `market_service_sync_registry` | 否 | 同步操作需串行 |

---

## 6. 相关文档

- [Agent 契约规范](../../Capital_Specifications/agentos_contract/agent_contract.md)
- [Skill 契约规范](../../Capital_Specifications/agentos_contract/skill_contract.md)
- [OpenLab 子系统](SUBSYSTEM_openlab.md)