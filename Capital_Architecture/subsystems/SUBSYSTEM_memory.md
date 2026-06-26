# SUBSYSTEM_memory — 记忆子系统

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/atoms/memory/` + `agentos/atoms/memoryrovol/`  
**负责人**: Team A (核心引擎)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

memory 子系统是 Airymax 的记忆存储与检索基础设施，采用**可拔插提供商架构**，支持内置免费提供商和 MemoryRovol 商业提供商两种实现。

### 1.1 核心职责

| 职责 | 模块 | 说明 |
|------|------|------|
| **提供商抽象** | `memory_provider.h` | 定义可拔插的内存提供商函数指针表 |
| **L1 原始存储** | `write_raw` / `get_raw` / `delete_raw` | 原始字节流的存储与检索 |
| **L2 特征提取** | `query` / `retrieve` | 向量索引与语义检索 |
| **L3 结构绑定** | 关系存储 | 知识图谱与实体关系 |
| **L4 模式识别** | `evolve` | 记忆模式挖掘与进化 |
| **遗忘引擎** | `forget` | 基于重要性的记忆淘汰 |
| **吸引子网络** | 吸引子 | 记忆关联网络 |
| **持久同调** | 持久性 | 拓扑数据分析 |
| **FAISS 加速** | 索引加速 | 向量检索加速 |
| **异步操作** | `async_ops` | 非阻塞记忆操作 |
| **LLM 集成** | `llm_integration` | 与 LLM 的深度集成 |
| **同步机制** | `sync_push` / `sync_pull` | 跨提供商同步 |

### 1.2 四层记忆模型

```
L1: 原始存储 (Raw)       — 字节流存储，原始数据
L2: 特征提取 (Feature)    — 向量索引，语义特征
L3: 结构绑定 (Structure)  — 知识图谱，实体关系
L4: 模式识别 (Pattern)    — 模式挖掘，规律发现
```

### 1.3 不在范围内的职责

- 不负责记忆的业务语义（由 coreloopthree 的记忆引擎负责）
- 不负责持久化存储格式（由 heapstore 负责）
- 不负责记忆的认知决策（由 coreloopthree 的认知引擎负责）

---

## 2. 输入/输出接口

### 2.1 提供商函数指针表 (`memory_provider.h`)

```c
typedef struct agentos_memory_provider {
    const char *name;                    // 提供商名称
    const char *version;                 // 版本号
    agentos_memory_capabilities_t caps;  // 能力标记
    void *impl;                          // 实现私有数据
    struct agentos_memory_provider *sync_target;  // 同步目标

    // 生命周期
    agentos_error_t (*init)(provider, config_path);
    void (*destroy)(provider);

    // L1: 原始存储
    agentos_error_t (*write_raw)(provider, data, len, metadata, out_id);
    agentos_error_t (*get_raw)(provider, record_id, out_data, out_len);
    agentos_error_t (*delete_raw)(provider, record_id);

    // L2: 查询检索
    agentos_error_t (*query)(provider, query_text, limit, out_ids, out_scores, out_count);
    agentos_error_t (*retrieve)(provider, query_text, limit, out_ids, out_scores, out_count);

    // L3/L4: 进化与遗忘
    agentos_error_t (*evolve)(provider, force);
    agentos_error_t (*forget)(provider);

    // 统计与健康
    agentos_error_t (*stats)(provider, out_stats);
    agentos_error_t (*health_check)(provider, out_json);

    // 上下文挂载
    agentos_error_t (*mount)(provider, record_id, context);

    // 增量添加
    agentos_error_t (*add_memory)(provider, content, content_len);

    // 同步
    agentos_error_t (*sync_push)(provider, record_id);
    agentos_error_t (*sync_pull)(provider, filter_json, out_ids, out_count);
    int (*has_active_sync)(provider);
} agentos_memory_provider_t;
```

### 2.2 提供商注册接口

| 函数 | 说明 |
|------|------|
| `agentos_memory_provider_register(provider)` | 注册提供商 |
| `agentos_memory_provider_get_active()` | 获取当前活跃提供商 |
| `agentos_memory_provider_set_active(provider)` | 切换活跃提供商 |
| `agentos_memory_provider_unregister()` | 注销提供商 |
| `agentos_builtin_memory_provider_init(storage_path)` | 初始化内置提供商 |
| `agentos_builtin_provider_create()` | 创建内置提供商实例 |

### 2.3 能力标记

```c
typedef struct agentos_memory_capabilities {
    uint8_t l1_raw;           // L1 原始存储
    uint8_t l2_feature;       // L2 特征提取/向量索引
    uint8_t l3_structure;     // L3 结构绑定/知识图谱
    uint8_t l4_pattern;       // L4 模式识别
    uint8_t forgetting;       // 遗忘引擎
    uint8_t attractor;        // 吸引子网络
    uint8_t persistence;      // 持久同调
    uint8_t faiss;            // FAISS 加速索引
    uint8_t async_ops;        // 异步操作
    uint8_t llm_integration;  // LLM 集成
} agentos_memory_capabilities_t;
```

### 2.4 统计信息

```c
typedef struct agentos_memory_stats {
    uint64_t total_records;      // 总记录数
    uint64_t total_bytes;        // 总字节数
    uint64_t l1_count;           // L1 记录数
    uint64_t l2_indexed;         // L2 索引数
    uint64_t l3_relations;       // L3 关系数
    uint64_t l4_patterns;        // L4 模式数
    double index_utilization;    // 索引利用率
    char provider_name[64];      // 提供商名称
    char provider_version[32];   // 提供商版本
} agentos_memory_stats_t;
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | 错误码 `error.h`、内存管理 |
| **commons** | 强依赖 | 类型定义、平台抽象 |
| **heapstore** | 可选依赖 | 持久化存储后端 |

### 3.2 被依赖方

- **coreloopthree**: 通过记忆引擎调用提供商接口
- **MemoryRovol**: 商业提供商实现此接口

### 3.3 提供商架构

```
┌──────────────────────────────────────────┐
│          coreloopthree 记忆引擎            │
│         (agentos_memory_engine_t)         │
└──────────────────┬───────────────────────┘
                   │ agentos_memory_provider_t*
     ┌─────────────┴─────────────┐
     ▼                           ▼
┌─────────────┐          ┌──────────────┐
│ builtin      │          │ MemoryRovol   │
│ provider     │          │ provider      │
│ (免费)       │          │ (商业)        │
└─────────────┘          └──────────────┘
```

---

## 4. 配置参数

### 4.1 内置提供商配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `storage_path` | 由调用者指定 | 存储路径 |
| `config_path` | NULL (使用默认) | 配置文件路径 |

### 4.2 提供商能力配置

根据 `AGENTOS_MEMORY_BACKEND` 编译选项选择：
- `builtin` — 内置免费提供商
- `memoryrovol` — MemoryRovol 商业提供商

### 4.3 编译选项

| 选项 | 说明 |
|------|------|
| `AGENTOS_MEMORY_BACKEND` | 选择记忆后端 |
| `AGENTOS_WITH_MEMORYROVOL` | 是否启用 MemoryRovol |

---

## 5. 线程安全与并发

| 操作 | 线程安全 | 说明 |
|------|----------|------|
| `provider_register` | 否 | 注册时单线程 |
| `write_raw` / `get_raw` / `delete_raw` | 取决于实现 | 内置提供商线程安全 |
| `query` / `retrieve` | 取决于实现 | 读操作通常线程安全 |
| `evolve` / `forget` | 否 | 写操作，需外部同步 |
| `sync_push` / `sync_pull` | 是 | 同步操作线程安全 |

---

## 6. 相关文档

- [记忆卷载系统](../memoryrovol.md)
- [记忆层理论](../../Basic_Theories/CN_03_记忆层设计.md)
- [MemoryRovol 集成](../../Capital_Specifications/integration_standards/README.md)