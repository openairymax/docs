# Airymax 命名规范 (P4.4.2)

本文档定义了 Airymax 项目中所有关键组件、文件、函数、类型、常量、错误码及 API 的命名约定，确保整个代码库的一致性与可读性。

---

## 1. 核心组件命名

### 1.1 Airymax

**Airymax**（极境）是产品名。**AgentRT/** 是代码仓库目录路径，为 **Agent** **R**un**t**ime（即"Agent 运行时"）的缩写。Airymax 是一个 OS 级的 AI Agent 运行时平台，为 AI Agent 提供操作系统级别的调度、资源管理和生命周期管理能力。

"RT" 的命名参照了传统操作系统中的"Runtime"概念（如 Windows Runtime / WinRT），强调 Airymax 不是上层框架，而是位于操作系统层面的基础运行时。

### 1.2 agentrt

**agentrt** = **agent** + **os**（全小写，无空格），即 Airymax 内部的"操作系统层"。它是整个 Airymax 平台的核心基础设施层，提供进程管理、内存管理、IPC 通信、文件系统等底层能力。

- 所有 C 源代码文件位于 `agentrt/` 目录下
- 代码中的命名空间前缀统一使用 `agentrt_`（如 `agentrt_error_t`、`agentrt_log_write`）
- **注意**：目录名与代码前缀使用小写，与代码仓库目录 `AgentRT/` 的 PascalCase 形成层次区分（产品名为 Airymax）

### 1.3 Atoms

**Atoms** = 原子（复数），即"原子构建块"。Atoms 是 Airymax 中最小的计算单元，代表不可再分的原子操作。

- 每个 Atom 是一个独立的、无副作用的计算单元
- 多个 Atoms 可以组合成更复杂的 TaskFlow
- 命名灵感来源于函数式编程中的原子操作概念

### 1.4 CoreKern

**CoreKern** = **Core** + **Kern**el，即"原子核心"。它是 Airymax 的中央调度器和分发器（Central Scheduler and Dispatcher）。

- 采用微核心（MicroCoreRT）架构设计
- 负责所有 Cupolas 服务的生命周期管理
- 实现 CoreLoopThree 认知循环的调度
- 目录名为 `corekern/`（小写），代码前缀为 `agentrt_core_*`
- **注**: MicroCoreRT（微核心）是 CoreKern 的架构概念层名称，两者共享同一目录和代码前缀，非独立模块

### 1.5 CoreLoopThree

**CoreLoopThree** = **Core** + **Loop** + **Three**，即"三层认知循环"。它描述了 Airymax 的认知处理流程：

```
Cognition（认知）→ Planning（规划）→ Action（执行）
```

- 第一阶段 **Cognition**：感知环境、理解上下文、提取意图
- 第二阶段 **Planning**：制定策略、分解任务、资源预估
- 第三阶段 **Action**：执行操作、调用工具、产生输出
- 循环往复，形成持续运行的认知回路

### 1.6 Cupolas

**Cupolas** = 穹顶（复数），即"安全穹顶"。命名灵感来源于大教堂建筑顶部的穹顶结构（Cupola），寓意每个 Cupola 是一个独立、自包含、向上拱起的安全服务单元。

- 每个 Cupola 是一个独立的服务进程，通过 IPC Bus 与 CoreKern 通信
- Cupola 之间相互隔离，通过消息传递进行协作
- 典型的 Cupola 包括：llm_d、tool_d、info_d、observe_d 等
- 目录名为 `cupolas/`（小写），代码前缀为 `cupolas_`

### 1.7 TaskFlow

**TaskFlow** = **Task** + **Flow**，即"任务编排流水线"。它表示任务在系统中的流动和处理过程。

- 任务从入口经过 CoreKern 调度，流经各个 Cupola 服务
- 每个阶段对任务进行特定的加工和处理
- 支持串行、并行、条件分支等编排模式
- 目录名为 `taskflow/`（小写），代码前缀为 `taskflow_`

### 1.8 HeapStore

**HeapStore** = **Heap** + **Store**，即"堆式持久化存储"。它结合了"堆"（Heap，动态内存）和"存储"（Store，持久化）两个概念。

- 提供动态分配的持久化存储能力
- 支持内存堆和磁盘存储的透明切换
- 自动管理数据的序列化和反序列化
- 目录名为 `heapstore/`（小写），代码前缀为 `heapstore_`

### 1.9 MemoryRovol

**MemoryRovol** = **Memory** + **Rovol**（**Roll** + **Volume** 的融合造词），即"记忆卷载"。它实现了 Airymax 的 4 层记忆系统：

| 层级 | 名称 | 说明 |
|------|------|------|
| L1 | Raw（原始记忆） | 未经处理的原始输入数据 |
| L2 | Feature（特征记忆） | 从原始数据中提取的关键特征 |
| L3 | Structure（结构记忆） | 特征之间的结构化关系 |
| L4 | Pattern（模式记忆） | 高层次的行为模式和知识 |

- 目录名为 `memoryrovol/`（小写），代码前缀为 `memoryrovol_`

---

## 2. 用户态服务（Daemon）命名

所有 Airymax 的后台用户态服务使用 `*_d` 后缀命名，遵循 Unix 用户态服务命名惯例（`d` = daemon）。

### 2.1 完整 Daemon 列表

| Daemon 名 | 完整名称 | 职责描述 |
|-----------|---------|---------|
| `gateway_d` | Gateway Daemon | 外部请求入口，协议适配，认证鉴权，速率限制 |
| `llm_d` | LLM Daemon | LLM 推理调度，模型路由，Token 管理，流式输出 |
| `tool_d` | Tool Daemon | 工具调用管理，工具注册表，执行沙箱，结果校验 |
| `sched_d` | Scheduler Daemon | 任务调度引擎，优先级队列，资源分配，负载均衡 |
| `info_d` | Information Daemon | 信息聚合服务，知识检索，上下文注入，数据转换 |
| `observe_d` | Observability Daemon | 可观测性服务，Metrics 采集，Tracing，健康检查 |
| `monit_d` | Monitor Daemon | 监控告警服务，阈值检测，告警路由，自愈策略 |
| `market_d` | Market Daemon | 能力市场，计费计量，配额管理 |
| `channel_d` | Channel Daemon | 通道管理服务，长连接管理，流式通道，会话保持 |
| `notify_d` | Notification Daemon | 通知推送服务，消息推送，多渠道分发，投递保证 |

### 2.2 Daemon 命名规则

```
<service_name>_d
```

- 服务名使用小写英文单词，多个单词间用下划线分隔
- 统一以 `_d` 后缀结尾
- 对应的 C 源码文件使用相同的命名（如 `gateway_d.c`）

---

## 3. 文件命名

### 3.1 C 源文件：snake_case

所有 C 源文件（`.c`）和头文件（`.h`）使用 **snake_case** 命名。

```
agentrt/commons/utils/observability/src/logger.c
agentrt/commons/utils/observability/include/logger.h
agentrt/commons/platform/include/platform.h
agentrt/commons/utils/quality/agentrt_quality.h
agentrt/commons/utils/config_unified/include/core_config.h
```

### 3.2 目录名：PascalCase（顶层）/ snake_case（代码层）

- **顶层概念目录** 使用 PascalCase：`Docs/`、`Tests/`、`Scripts/`
- **代码层目录** 使用 snake_case：`agentrt/`、`commons/`、`protocols/`、`cupolas/`、`corekern/`

### 3.3 配置文件：kebab-case

所有配置文件使用 **kebab-case**（小写字母 + 连字符）命名。

```
.env.example
.editorconfig
.gitattributes
```

### 3.4 脚本文件：snake_case / kebab-case

Shell 脚本和 Python 脚本使用 snake_case 或 kebab-case：

```
scripts/ci/pipeline/run-tests.sh
scripts/ci/pipeline/security-scan.sh
scripts/ci/quality/check-quality.sh
scripts/dev/utils/run_all_fixes.sh
```

---

## 4. 函数命名

### 4.1 基本规则：`module_action_object()`

所有函数遵循 `module_action_object()` 的命名规范，使用 snake_case。

```
<模块前缀>_<动作>_<对象>()
```

### 4.2 代码示例

```c
// agentrt 模块 - 日志操作
const char *agentrt_log_set_trace_id(const char *trace_id);
const char *agentrt_log_get_trace_id(void);
void        agentrt_log_write(int level, const char *file, int line, const char *fmt, ...);

// config 模块 - 配置值操作
config_value_t *config_value_create_null(void);
config_value_t *config_value_create_bool(bool value);
config_value_t *config_value_create_int(int32_t value);
config_value_t *config_value_get_type(const config_value_t *value);
void            config_value_destroy(config_value_t *value);

// config 模块 - 配置上下文操作
config_context_t *config_context_create(const char *name);
void              config_context_destroy(config_context_t *ctx);
config_error_t    config_context_set(config_context_t *ctx, const char *key, config_value_t *value);
const config_value_t *config_context_get(const config_context_t *ctx, const char *key);

// heapstore 模块
int heapstore_init(const heapstore_config_t *config);
int heapstore_put(const char *key, const void *data, size_t size);
int heapstore_get(const char *key, void **data, size_t *size);

// CoreKern 模块（代码前缀 agentrt_core_）
int agentrt_core_start(void);
int agentrt_core_stop(void);
int agentrt_core_register_cupola(const char *name, cupola_entry_t entry);
```

### 4.3 常见动作动词

| 动词 | 含义 | 示例 |
|------|------|------|
| `create` | 创建/分配资源 | `config_value_create_null` |
| `destroy` | 销毁/释放资源 | `config_value_destroy` |
| `get` | 获取值（不修改状态） | `config_value_get_type` |
| `set` | 设置值 | `config_context_set` |
| `init` | 初始化 | `heapstore_init` |
| `start` | 启动 | `agentrt_core_start` |
| `stop` | 停止 | `agentrt_core_stop` |
| `write` | 写入 | `agentrt_log_write` |
| `clone` | 克隆/深拷贝 | `config_value_clone` |
| `has` | 检查是否存在 | `config_context_has` |
| `delete` | 删除 | `config_context_delete` |
| `clear` | 清空 | `config_context_clear` |
| `count` | 计数 | `config_context_count` |
| `lock` | 加锁 | `config_context_lock` |
| `unlock` | 解锁 | `config_context_unlock` |

---

## 5. 类型命名

### 5.1 基本规则：`module_type_t`

所有类型定义遵循 `module_type_t` 的命名规范，使用 snake_case，以 `_t` 后缀结尾。

```c
<模块前缀>_<类型名>_t
```

### 5.2 代码示例

```c
// 基础类型
typedef int32_t agentrt_error_t;       // agentrt 错误码类型

// 结构体类型
typedef struct config_value    config_value_t;     // 配置值
typedef struct config_context  config_context_t;   // 配置上下文
typedef struct config_schema   config_schema_t;    // 配置模式

// 枚举类型
typedef enum {
    CONFIG_TYPE_NULL = 0,
    CONFIG_TYPE_BOOL = 1,
    CONFIG_TYPE_INT  = 2,
    // ...
} config_value_type_t;

typedef enum {
    CONFIG_SUCCESS = 0,
    CONFIG_ERROR_INVALID_ARG = 1,
    // ...
} config_error_t;

// heapstore 模块类型
typedef struct heapstore_config heapstore_config_t;
typedef struct heapstore_entry  heapstore_entry_t;

// memoryrovol 模块类型
typedef struct memoryrovol_layer memoryrovol_layer_t;
typedef enum   memoryrovol_level memoryrovol_level_t;
```

### 5.3 枚举值命名

枚举值使用 `MODULE_UPPER_SNAKE_CASE` 格式：

```c
typedef enum {
    AGENTRT_MEM_LAYER1_RAW = 0,       // 注意：不是全大写，而是模块前缀 + 含义
    AGENTRT_MEM_LAYER2_FEATURE = 1,
    AGENTRT_MEM_LAYER3_EPISODIC = 2,
    AGENTRT_MEM_LAYER4_PATTERN = 3,
} agentrt_memory_layer_t;

typedef enum {
    CONFIG_TYPE_NULL = 0,
    CONFIG_TYPE_BOOL = 1,
    CONFIG_TYPE_INT = 2,
    CONFIG_TYPE_INT64 = 3,
    CONFIG_TYPE_DOUBLE = 4,
    CONFIG_TYPE_STRING = 5,
    CONFIG_TYPE_ARRAY = 6,
    CONFIG_TYPE_OBJECT = 7,
    CONFIG_TYPE_BINARY = 8,
} config_value_type_t;
```

---

## 6. 常量与宏命名

### 6.1 宏定义：UPPER_SNAKE_CASE

所有预处理器宏使用 **UPPER_SNAKE_CASE**（全大写 + 下划线分隔）。

```c
// 日志级别宏
#define AGENTRT_LOG_LEVEL_DEBUG 0
#define AGENTRT_LOG_LEVEL_INFO  1
#define AGENTRT_LOG_LEVEL_WARN  2
#define AGENTRT_LOG_LEVEL_ERROR 3
#define AGENTRT_LOG_LEVEL_FATAL 4

// 平台检测宏
#define AGENTRT_PLATFORM_LINUX   1
#define AGENTRT_PLATFORM_MACOS   1
#define AGENTRT_PLATFORM_WINDOWS 1
#define AGENTRT_PLATFORM_NAME    "Linux"
#define AGENTRT_PLATFORM_BITS    64
#define AGENTRT_PLATFORM_POSIX   1

// 质量保证宏
#define AGENTRT_CHECK_NULL(ptr, error_code)       ...
#define AGENTRT_CHECK_NULL_GOTO(ptr, label, code) ...
#define AGENTRT_CHECK_CONDITION(cond, error_code) ...
#define AGENTRT_CHECK_RANGE(value, min, max, err) ...

// 头文件保护宏
#define AGENTRT_TYPES_H
#define AGENTRT_UTILS_LOGGER_H
#define AGENTRT_CORE_CONFIG_H
#define AGENTRT_PLATFORM_H
#define AGENTRT_QUALITY_H
```

### 6.2 模块前缀规则

所有宏必须以模块前缀开头，避免全局命名冲突：

```
AGENTRT_<类别>_<名称>
```

---

## 7. 错误码命名

### 7.1 基本规则：`MODULE_ERR_CATEGORY`

错误码的命名遵循 `MODULE_ERR_CATEGORY` 的格式：

| 模块 | 格式 | 示例 |
|------|------|------|
| agentrt（通用） | `AGENTRT_ERR_<CODE>` | `AGENTRT_ERR_INVALID_PARAM`、`AGENTRT_ERR_OUT_OF_MEMORY` |
| config | `CONFIG_ERROR_<NAME>` | `CONFIG_ERROR_INVALID_ARG`、`CONFIG_ERROR_NOT_FOUND` |

### 7.2 通用错误码（agentrt 层）

```c
#define AGENTRT_OK      (0)   // 成功
#define AGENTRT_ERR_INVALID_PARAM       (-2)  // 参数无效
#define AGENTRT_ERR_OUT_OF_MEMORY       (-4)  // 内存不足
#define AGENTRT_ERR_BUSY        (-17)  // 资源忙碌
#define AGENTRT_ENOENT       (-4)  // 资源不存在
#define AGENTRT_ERR_PERMISSION_DENIED        (-10)  // 权限不足
#define AGENTRT_ETIMEDOUT    (-6)  // 操作超时
#define AGENTRT_EIO          (-7)  // I/O 错误
#define AGENTRT_EEXIST       (-8)  // 资源已存在
#define AGENTRT_ENOTINIT     (-9)  // 引擎未初始化
#define AGENTRT_ECANCELLED   (-10) // 操作已取消
#define AGENTRT_ENOTSUP      (-11) // 操作不支持
#define AGENTRT_EOVERFLOW    (-12) // 溢出错误
#define AGENTRT_EPROTO       (-13) // 协议错误
#define AGENTRT_ENOTCONN     (-14) // 未连接
#define AGENTRT_ECONNRESET   (-15) // 连接重置
#define AGENTRT_ENOSYS       (-16) // 函数未实现
#define AGENTRT_EFAIL        (-17) // 通用失败
#define AGENTRT_ENOTFOUND    (-18) // 资源未找到
#define AGENTRT_EPLATFORM    (-27) // 平台未初始化
#define AGENTRT_EPROTONOSUPPORT (-28) // 协议/命令不支持
#define AGENTRT_ESERVICE     (-29) // 服务不可用
#define AGENTRT_EUNKNOWN     (-99) // 未知错误
```

### 7.3 模块级错误码（config 模块示例）

```c
typedef enum {
    CONFIG_SUCCESS = 0,              // 成功
    CONFIG_ERROR_INVALID_ARG = 1,    // 参数无效
    CONFIG_ERROR_NOT_FOUND = 2,      // 配置项不存在
    CONFIG_ERROR_TYPE_MISMATCH = 3,  // 类型不匹配
    CONFIG_ERROR_OUT_OF_MEMORY = 4,  // 内存不足
    CONFIG_ERROR_IO = 5,             // I/O 错误
    CONFIG_ERROR_PARSE = 6,          // 解析错误
    CONFIG_ERROR_VALIDATION = 7,     // 验证失败
    CONFIG_ERROR_LOCKED = 8,         // 配置被锁定
    CONFIG_ERROR_UNSUPPORTED = 9,    // 不支持的操作
    CONFIG_ERROR_THREAD = 10,        // 线程操作失败
} config_error_t;
```

### 7.4 错误码设计原则

1. **通用层**（agentrt）使用类 POSIX 风格的宏定义（`AGENTRT_E*`），所有错误码为负值，`0` 表示成功
2. **模块层**（如 config）使用枚举类型，正值表示错误类别，`0` 表示成功
3. 错误码语义应与 POSIX errno 保持一致的直觉（如 `EINVAL` = 无效参数）
4. 每个模块的错误码转换为字符串的函数命名为 `module_error_to_string()`

---

## 8. API 版本化

### 8.1 `@since` 标签

所有公开 API 在 Doxygen 注释中使用 `@since vX.Y.Z` 标签标注引入版本，遵循语义化版本（Semantic Versioning）。

### 8.2 代码示例

```c
/**
 * @brief 创建空配置值
 * @return 配置值对象，失败返回 NULL
 * @since v0.1.0
 */
config_value_t *config_value_create_null(void);

/**
 * @brief 创建布尔配置值
 * @param value 布尔值
 * @return 配置值对象，失败返回 NULL
 * @since v0.1.0
 */
config_value_t *config_value_create_bool(bool value);

/**
 * @brief 获取配置上下文中的值
 * @param ctx 配置上下文
 * @param key 键名
 * @return 配置值对象，不存在时返回 NULL
 * @since v0.1.0
 */
const config_value_t *config_context_get(const config_context_t *ctx, const char *key);

/**
 * @brief 设置跨平台线程局部存储变量
 * @param key 变量键名
 * @param value 变量值
 * @return AGENTRT_OK 成功，AGENTRT_ENOMEM 内存不足
 * @since v0.2.0
 */
int agentrt_tls_set(const char *key, void *value);
```

### 8.3 版本号规则

- 主版本号（MAJOR）：不兼容的 API 修改
- 次版本号（MINOR）：向下兼容的功能新增
- 修订号（PATCH）：向下兼容的问题修正

---

## 9. 命名速查表

| 类别 | 规则 | 示例 |
|------|------|------|
| 平台名 | PascalCase | `Airymax` |
| 操作系统层 | 全小写 | `agentrt` |
| 核心组件 | PascalCase | `CoreKern`、`CoreLoopThree`、`TaskFlow`、`HeapStore`、`MemoryRovol` |
| 复数组件 | PascalCase（复数） | `Atoms`、`Cupolas` |
| 用户态服务 | snake_case + `_d` | `gateway_d`、`llm_d`、`tool_d` |
| C 源文件 | snake_case | `logger.c`、`core_config.c` |
| 头文件 | snake_case | `logger.h`、`platform.h` |
| 代码目录 | snake_case | `agentrt/`、`commons/`、`protocols/` |
| 顶层目录 | PascalCase | `Docs/`、`Tests/`、`Scripts/` |
| 配置文件 | kebab-case | `.editorconfig`、`.env.example` |
| 函数 | `module_action_object()` | `heapstore_init`、`config_value_create` |
| 类型 | `module_type_t` | `agentrt_error_t`、`config_value_t` |
| 枚举 | `MODULE_UPPER` | `CONFIG_TYPE_INT`、`AGENTRT_MEM_LAYER1_RAW` |
| 宏 | `UPPER_SNAKE_CASE` | `AGENTRT_LOG_LEVEL_DEBUG`、`AGENTRT_CHECK_NULL` |
| 通用错误码 | `AGENTRT_ERR_*` | `AGENTRT_ERR_INVALID_PARAM`、`AGENTRT_ERR_OUT_OF_MEMORY` |
| 模块错误码 | `MODULE_ERROR_*` | `CONFIG_ERROR_NOT_FOUND` |
| 头文件保护 | `UPPER_SNAKE_CASE` | `AGENTRT_TYPES_H`、`AGENTRT_PLATFORM_H` |
| API 版本 | `@since vX.Y.Z` | `@since v0.1.0` |

---

## 10. 常见错误与反模式

### 10.1 应避免的命名

| 反模式 | 问题 | 正确写法 |
|--------|------|---------|
| `agentrtLogWrite` | 混用 camelCase，C 代码应使用 snake_case | `agentrt_log_write` |
| `ConfigValue` | C 类型不应使用 PascalCase | `config_value_t` |
| `AGENTRT_log_level` | 宏不应混用大小写 | `AGENTRT_LOG_LEVEL` |
| `gatewayDaemon` | Daemon 名应使用 `_d` 后缀 | `gateway_d` |
| `core_kern` | 组件名应使用 PascalCase | `CoreKern` |
| `heap-store.h` | C 头文件应使用 snake_case | `heapstore.h` |

### 10.2 关键原则

1. **一致性优先**：当不确定时，参考已有代码中的命名模式
2. **模块前缀必须**：所有公开符号必须带模块前缀，避免链接冲突
3. **语义明确**：命名应准确表达其用途，避免缩写歧义（除非是广泛接受的缩写如 `init`、`cfg`）
4. **C 惯例为准**：由于 Airymax 核心以 C 语言实现，命名以 C 社区惯例为基准