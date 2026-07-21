Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Go 编码风格规范
> **文档定位**：Go 语言编码风格及安全编码规范合集（含 Go 风格、Go 安全编码）\
> **文档版本**：0.1.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agentrt-linux（AirymaxOS）工程标准规范](README.md)

---

## Part I: Airymax Go 编码风格规范

### 目录

1. [总则与适用范围](#1-总则与适用范围)
2. [包组织与目录结构](#2-包组织与目录结构)
3. [命名规范](#3-命名规范)
4. [类型设计](#4-类型设计)
5. [函数设计](#5-函数设计)
6. [错误处理](#6-错误处理)
7. [并发编程](#7-并发编程)
8. [测试规范](#8-测试规范)
9. [文档注释规范](#9-文档注释规范)
10. [依赖管理](#10-依赖管理)
11. [Airymax 模块编码示例](#11-agentrt-模块编码示例)

---

### 1. 总则与适用范围

#### 1.1 目的

本规范定义 Airymax Go SDK 的编码风格标准，确保代码库在命名、结构、错误处理、并发模式和测试策略上保持高度一致性。所有规则均源自项目实际代码中的既定模式，而非理论推导。

#### 1.2 适用范围

- **必须**适用于 `agentrt/toolkit/go/` 下所有 `.go` 文件
- **必须**适用于新增包、新增类型和新增公共 API
- **应当**适用于项目内其他 Go 代码（工具脚本、代码生成器等可适当放宽）

#### 1.3 规范等级

| 关键词 | 含义 |
|--------|------|
| **必须** | 强制遵守，违反将导致 CI 拒绝合并 |
| **禁止** | 强制禁止，违反将导致 CI 拒绝合并 |
| **应当** | 推荐遵守，特殊场景可偏离但须注释说明 |
| **可以** | 可选建议 |

#### 1.4 强制执行机制

| 工具 | 用途 |
|------|------|
| `go vet` | 静态分析，检测常见错误 |
| `staticcheck` | 高级静态分析 |
| `golangci-lint` | 聚合 linter，CI 必须通过 |
| `gofmt` / `goimports` | 格式化与导入排序 |
| CI Pipeline | 合并前全量检查 |

---

### 2. 包组织与目录结构

#### 2.1 顶层包结构

Airymax Go SDK 采用扁平化子包结构，每个子包对应一个独立职责域：

```
agentrt/toolkit/go/agentrt/
├── agentrt.go          # 包常量 (Version, Author, License) + 全局 Logger
├── config.go           # Config 结构体 + Functional Options + 环境变量构造
├── errors.go           # AgentRTError + ErrorCode + 哨兵错误 + HTTP 映射
├── protocol.go         # ProtocolClient + ProtocolType 枚举
├── client/             # APIClient 接口 + Client 实现 + Mock
├── types/              # 枚举、领域模型、RequestOptions
├── utils/              # 类型安全 map 提取、URL 构建、ID 生成
├── modules/            # BaseManager[T] + 业务子模块
│   ├── base_manager.go # 泛型基类 + ResourceConverter[T]
│   ├── modules.go      # 子包入口声明
│   ├── task/           # 任务管理
│   ├── memory/         # 记忆管理
│   ├── session/        # 会话管理
│   └── skill/          # 技能管理
├── plugin/             # BasePlugin 接口 + PluginRegistry + PluginManager
├── telemetry/          # Meter + Tracer
└── syscall/            # SyscallBinding 接口 + 便捷类型
```

#### 2.2 包划分原则

**必须**按职责单一原则划分包。一个包只做一件事。

❌ **禁止**将不相关的功能混入同一包：

```go
// agentrt/helpers.go — 禁止：混合了 HTTP、JSON、时间处理
package agentrt

func DoHTTPRequest(...) {}
func ParseJSON(...) {}
func FormatTime(...) {}
```

**必须**按职责拆分到独立子包：

```go
// client/client.go — HTTP 客户端职责
package client

// utils/helpers.go — 通用数据提取职责
package utils

// telemetry/telemetry.go — 可观测性职责
package telemetry
```

#### 2.3 包内文件组织

**必须**按类型/功能将代码拆分到不同文件，而非全部塞入单文件：

| 文件 | 内容 |
|------|------|
| `errors.go` | 错误类型、错误码常量、哨兵错误、工厂函数 |
| `config.go` | 配置结构体、Option 函数、环境变量构造 |
| `protocol.go` | 协议客户端、协议类型枚举 |
| `types.go` | 枚举、领域模型、请求/响应结构 |

**禁止**将超过 500 行的代码放在单个文件中（测试文件除外）。

#### 2.4 包名规范

**必须**使用全小写、无下划线、无驼峰的简短名词：

```go
package agentrt    // 正确
package client     // 正确
package types      // 正确
package modules    // 正确
package task       // 正确
package telemetry  // 正确
package syscall    // 正确
```

❌ **禁止**以下包名风格：

```go
package agent_os   // 禁止：下划线
package agentRt    // 禁止：驼峰
package airy_sdk // 禁止：下划线+过长
package util       // 禁止：过于笼统（应为 utils）
```

---

### 3. 命名规范

#### 3.1 导出类型：PascalCase

**必须**使用 PascalCase 命名所有导出类型，名词短语优先：

```go
type Client struct{}           // 
type TaskManager struct{}      // 
type AgentRTError struct{}     // 
type Config struct{}           // 
type ProtocolType int          // 
type ResourceConverter[T any] interface {} // 
```

❌ **禁止**：

```go
type client struct{}           // 禁止：导出类型必须大写开头
type Task_Manager struct{}     // 禁止：下划线
type TASKMANAGER struct{}      // 禁止：全大写
```

#### 3.2 导出函数：PascalCase

**必须**使用 PascalCase 命名所有导出函数：

```go
func NewClient(...) (*Client, error)       // 
func NewConfig(...) *Config                // 
func NewTaskManager(...) *TaskManager      // 
func IsNetworkError(err error) bool        // 
func IsErrorCode(err error, code ErrorCode) bool // 
func HTTPStatusToError(...) *AgentRTError  // 
```

#### 3.3 未导出函数：camelCase

**必须**使用 camelCase 命名所有未导出函数：

```go
func parseTaskFromMap(data map[string]interface{}) *types.Task  // 
func buildQueryString(params map[string]string) string          // 
func calculateBackoff(base time.Duration, attempt int) time.Duration // 
func shouldRetry(statusCode int) bool                           // 
func generateRequestID() string                                // 
```

❌ **禁止**：

```go
func ParseTaskFromMap(...)   // 禁止：包内辅助函数不应导出
func parse_task_from_map(...) // 禁止：下划线
```

#### 3.4 构造函数：New 前缀

**必须**使用 `New` 前缀命名所有构造函数：

```go
func NewClient(opts ...ConfigOption) (*Client, error)        // 
func NewConfig(opts ...ConfigOption) *Config                 // 
func NewTaskManager(api APIClient) *TaskManager              // 
func NewError(code ErrorCode, msg string, cause error) *AgentRTError // 
func NewPluginRegistry() *PluginRegistry                     // 
func NewMeter() *Meter                                       // 
```

当存在多种构造方式时，**必须**使用描述性后缀：

```go
func NewClient(opts ...ConfigOption) (*Client, error)           // 默认构造
func NewClientWithConfig(config *Config) (*Client, error)       // 显式配置构造
func NewConfigFromEnv() (*Config, error)                        // 环境变量构造
func NewProtocolConfigFromEnv() *ProtocolConfig                 // 
func NewHTTPSyscallBinding(apiClient APIClient) *HTTPSyscallBinding // 
```

❌ **禁止**：

```go
func CreateClient(...)    // 禁止：非 New 前缀
func MakeConfig(...)      // 禁止：非 New 前缀
func ClientNew(...)       // 禁止：后置 New
```

#### 3.5 Option 函数：With 前缀

**必须**使用 `With` 前缀命名所有 Functional Option 函数：

```go
func WithEndpoint(endpoint string) ConfigOption        // 
func WithTimeout(timeout time.Duration) ConfigOption   // 
func WithAPIKey(apiKey string) ConfigOption            // 
func WithMaxRetries(maxRetries int) ConfigOption       // 
func WithDebug(debug bool) ConfigOption                // 
func WithRequestTimeout(timeout time.Duration) RequestOption // 
func WithHeader(key, value string) RequestOption       // 
func WithQueryParam(key, value string) RequestOption   // 
func WithSandboxDisabled() func(*PluginManager)        // 
func WithPluginDirectories(dirs []string) func(*PluginManager) // 
```

❌ **禁止**：

```go
func SetEndpoint(endpoint string) ConfigOption   // 禁止：非 With 前缀
func EndpointOpt(endpoint string) ConfigOption   // 禁止：非 With 前缀
func With_endpoint(endpoint string) ConfigOption // 禁止：下划线
```

#### 3.6 判断函数：Is 前缀

**必须**使用 `Is` 前缀命名所有返回 `bool` 的判断函数：

```go
func IsNetworkError(err error) bool      // 
func IsServerError(err error) bool       // 
func IsErrorCode(err error, code ErrorCode) bool // 
func (s TaskStatus) IsTerminal() bool    // 
func (l MemoryLayer) IsValid() bool      // 
```

❌ **禁止**：

```go
func CheckNetworkError(err error) bool   // 禁止：非 Is 前缀
func NetworkError(err error) bool        // 禁止：缺少动词
func Is_network_error(err error) bool    // 禁止：下划线
```

#### 3.7 常量：PascalCase + 语义分组前缀

##### 3.7.1 包级常量

**必须**使用 PascalCase 命名导出常量：

```go
const (
    Version = "0.1.0"     // 
    Author  = "SpharxWorks" // 
    License = "MIT"       // 
)
```

##### 3.7.2 错误码常量

**必须**使用 `Code` 前缀 + PascalCase 语义名，类型为 `ErrorCode`（底层 `string`），值采用十六进制分类体系：

```go
const (
    CodeSuccess          ErrorCode = "0x0000"  // 通用
    CodeUnknown          ErrorCode = "0x0001"  // 
    CodeInvalidParameter ErrorCode = "0x0002"  // 
    CodeNotFound         ErrorCode = "0x0005"  // 

    CodeLoopCreateFailed ErrorCode = "0x1001"  // 核心循环
    CodeCognitionFailed  ErrorCode = "0x2001"  // 认知层
    CodeTaskFailed       ErrorCode = "0x3001"  // 执行层
    CodeMemoryNotFound   ErrorCode = "0x4001"  // 记忆层
    CodeTelemetryError   ErrorCode = "0x5001"  // 系统调用
    CodePermissionDenied ErrorCode = "0x6001"  // 安全域
)
```

十六进制分类体系（依据 `ErrorCodeReference.md`）：

| 区间 | 领域 |
|------|------|
| `0x0xxx` | 通用错误 (General) |
| `0x1xxx` | 核心循环错误 (CoreLoop) |
| `0x2xxx` | 认知层错误 (Cognition) |
| `0x3xxx` | 执行层错误 (Execution) |
| `0x4xxx` | 记忆层错误 (Memory) |
| `0x5xxx` | 系统调用错误 (Syscall) |
| `0x6xxx` | 安全域错误 (Security) |
| `0x7xxx` | 动态模块错误 (Gateway) |

❌ **禁止**：

```go
const SUCCESS = "0x0000"           // 禁止：全大写 + 非 ErrorCode 类型
const code_success ErrorCode = "0x0000" // 禁止：下划线 + 小写
const CodeSuccess int = 0          // 禁止：非 string 类型
```

##### 3.7.3 枚举常量

**必须**使用类型名作为前缀 + PascalCase 值名：

```go
type TaskStatus string

const (
    TaskStatusPending   TaskStatus = "pending"   // 
    TaskStatusRunning   TaskStatus = "running"   // 
    TaskStatusCompleted TaskStatus = "completed" // 
    TaskStatusFailed    TaskStatus = "failed"    // 
    TaskStatusCancelled TaskStatus = "cancelled" // 
)

type ProtocolType int

const (
    ProtocolJSONRPC ProtocolType = iota // 
    ProtocolMCP                          // 
    ProtocolA2A                          // 
    ProtocolOpenAI                       // 
    ProtocolAutoDetect                   // 
)
```

❌ **禁止**：

```go
const (
    STATUS_PENDING   TaskStatus = "pending"  // 禁止：全大写
    TaskStatus_PENDING TaskStatus = "pending" // 禁止：下划线分隔
    Pending TaskStatus = "pending"            // 禁止：缺少类型前缀
)
```

#### 3.8 哨兵错误变量：Err 前缀

**必须**使用 `Err` 前缀命名所有哨兵错误变量，且**必须**使用 `var` 声明：

```go
var (
    ErrNotFound         = NewError(CodeNotFound, "资源未找到", nil)         // 
    ErrTimeout          = NewError(CodeTimeout, "操作超时", nil)            // 
    ErrInvalidConfig    = NewError(CodeInvalidConfig, "配置无效", nil)      // 
    ErrNetworkError     = NewError(CodeNetworkError, "网络错误", nil)       // 
    ErrTaskFailed       = NewError(CodeTaskFailed, "任务执行失败", nil)     // 
    ErrMemoryNotFound   = NewError(CodeMemoryNotFound, "记忆未找到", nil)   // 
    ErrPermissionDenied = NewError(CodePermissionDenied, "权限不足", nil)   // 
)
```

❌ **禁止**：

```go
var NotFoundErr = NewError(...)   // 禁止：后置 Err
var ERR_NOT_FOUND = NewError(...) // 禁止：全大写
var notFoundErr = NewError(...)   // 禁止：小写（哨兵错误必须导出）
```

#### 3.9 接口命名：语义名

**必须**使用语义名（而非 `I` 前缀或 `Interface` 后缀）命名接口：

```go
type APIClient interface {}            // 语义名
type BasePlugin interface {}           // 语义名
type SyscallBinding interface {}       // 语义名
type ResourceConverter[T any] interface {} // 泛型接口
type PluginFactory func() BasePlugin   // 函数类型别名
```

❌ **禁止**：

```go
type IAPIClient interface {}       // 禁止：I 前缀
type APIClientInterface interface {} // 禁止：Interface 后缀
type ClientAPI interface {}        // 禁止：词序颠倒（应为 APIClient）
```

---

### 4. 类型设计

#### 4.1 结构体

##### 4.1.1 字段对齐与分组

**必须**按逻辑分组排列结构体字段，相关字段放在一起：

```go
type Config struct {
    // 连接配置
    Endpoint        string
    Timeout         time.Duration
    MaxRetries      int
    RetryDelay      time.Duration
    MaxConnections  int
    IdleConnTimeout time.Duration

    // 认证配置
    APIKey    string
    UserAgent string

    // 调试配置
    Debug    bool
    LogLevel string
}
```

##### 4.1.2 JSON 标签

**必须**为所有序列化结构体添加 `json` 标签，标签名使用 snake_case：

```go
type Task struct {
    ID          string                 `json:"task_id"`
    Description string                 `json:"description"`
    Status      TaskStatus             `json:"status"`
    Priority    int                    `json:"priority"`
    Output      string                 `json:"output"`
    Error       string                 `json:"error"`
    Metadata    map[string]interface{} `json:"metadata"`
    CreatedAt   time.Time              `json:"created_at"`
    UpdatedAt   time.Time              `json:"updated_at"`
}
```

❌ **禁止**省略 json 标签或使用 camelCase 标签名：

```go
type Task struct {
    ID string `json:"taskID"` // 禁止：camelCase JSON 标签
    Description string         // 禁止：缺少 json 标签
}
```

##### 4.1.3 不可变性方法

当方法返回内部状态的副本时，**必须**返回克隆值而非指针：

```go
func (c *Client) GetConfig() *agentrt.Config {
    return c.config.Clone() // 返回克隆，防止外部修改
}
```

❌ **禁止**直接返回内部可变状态的引用：

```go
func (c *Client) GetConfig() *agentrt.Config {
    return c.config // 禁止：外部可修改内部状态
}
```

##### 4.1.4 String() 方法

**应当**为核心业务类型实现 `String()` 方法：

```go
func (c *Config) String() string {
    return fmt.Sprintf("Config[endpoint=%s, timeout=%v, retries=%d]",
        c.Endpoint, c.Timeout, c.MaxRetries)
}

func (p ProtocolType) String() string {
    switch p {
    case ProtocolJSONRPC:
        return "jsonrpc"
    case ProtocolMCP:
        return "mcp"
    default:
        return "auto"
    }
}
```

#### 4.2 枚举类型

**必须**使用 `type` 定义枚举底层类型，**禁止**使用裸 `int`/`string`：

```go
// 正确：基于 string 的枚举
type TaskStatus string

const (
    TaskStatusPending   TaskStatus = "pending"
    TaskStatusRunning   TaskStatus = "running"
    TaskStatusCompleted TaskStatus = "completed"
)

// 正确：基于 int 的枚举（iota）
type ProtocolType int

const (
    ProtocolJSONRPC ProtocolType = iota
    ProtocolMCP
)
```

**应当**为枚举类型提供 `String()` 方法和判断方法：

```go
func (s TaskStatus) String() string { return string(s) }

func (s TaskStatus) IsTerminal() bool {
    return s == TaskStatusCompleted || s == TaskStatusFailed || s == TaskStatusCancelled
}

func (l MemoryLayer) IsValid() bool {
    switch l {
    case MemoryLayerL1, MemoryLayerL2, MemoryLayerL3, MemoryLayerL4:
        return true
    }
    return false
}
```

#### 4.3 接口

##### 4.3.1 接口定义

**必须**在消费侧定义接口，接口应当小而精：

```go
type APIClient interface {
    Get(ctx context.Context, path string, opts ...RequestOption) (*APIResponse, error)
    Post(ctx context.Context, path string, body interface{}, opts ...RequestOption) (*APIResponse, error)
    Put(ctx context.Context, path string, body interface{}, opts ...RequestOption) (*APIResponse, error)
    Delete(ctx context.Context, path string, opts ...RequestOption) (*APIResponse, error)
}
```

##### 4.3.2 接口满足性编译期检查

**必须**在实现类型所在文件中添加编译期接口满足性检查：

```go
var _ APIClient = (*Client)(nil) // 编译期验证 Client 实现 APIClient
```

❌ **禁止**省略此检查：

```go
// 禁止：缺少编译期接口满足性检查
type Client struct { ... }
```

#### 4.4 泛型

##### 4.4.1 泛型基类

**必须**使用泛型实现资源管理器的通用基类，避免代码重复：

```go
type ResourceConverter[T any] interface {
    Convert(data map[string]interface{}) (*T, error)
}

type BaseManager[T any] struct {
    api          client.APIClient
    resourceType string
    converter    ResourceConverter[T]
}

func NewBaseManager[T any](api client.APIClient, resourceType string,
    converter ResourceConverter[T]) *BaseManager[T] {
    return &BaseManager[T]{
        api:          api,
        resourceType: resourceType,
        converter:    converter,
    }
}
```

##### 4.4.2 泛型工具函数

**可以**使用泛型约束编写类型安全的工具函数：

```go
func ValidateNonEmptySlice[T any](value []T, paramName string) error {
    if len(value) == 0 {
        return fmt.Errorf("%s不能为空", paramName)
    }
    return nil
}
```

---

### 5. 函数设计

#### 5.1 构造函数模式

**必须**遵循以下构造函数模式：

1. 默认构造函数使用 `NewXxx(opts ...Option)` 签名
2. 显式配置构造函数使用 `NewXxxWithConfig(config)` 签名
3. 环境变量构造函数使用 `NewXxxFromEnv()` 签名
4. 内部共享初始化逻辑使用未导出的 `newXxxWithConfig()` 函数

```go
// 公共默认构造
func NewClient(opts ...agentrt.ConfigOption) (*Client, error) {
    config := agentrt.NewConfig(opts...)
    if err := config.Validate(); err != nil {
        return nil, err
    }
    return newClientWithConfig(config)
}

// 公共显式配置构造
func NewClientWithConfig(config *agentrt.Config) (*Client, error) {
    if config == nil {
        config = agentrt.DefaultConfig()
    }
    if err := config.Validate(); err != nil {
        return nil, err
    }
    return newClientWithConfig(config)
}

// 内部共享初始化
func newClientWithConfig(config *agentrt.Config) (*Client, error) {
    return &Client{
        config: config,
        httpClient: &http.Client{
            Timeout: config.Timeout,
            Transport: &http.Transport{
                MaxIdleConns:       config.MaxConnections,
                IdleConnTimeout:    config.IdleConnTimeout,
                DisableCompression: false,
            },
        },
    }, nil
}
```

#### 5.2 Functional Options 模式

**必须**使用 Functional Options 模式配置可变参数：

```go
// 1. 定义 Option 类型
type ConfigOption func(*Config)

// 2. 提供默认值函数
func DefaultConfig() *Config {
    return &Config{
        Endpoint:   "http://127.0.0.1:8080",
        Timeout:    30 * time.Second,
        MaxRetries: 3,
    }
}

// 3. 实现 With 前缀的 Option 函数
func WithEndpoint(endpoint string) ConfigOption {
    return func(c *Config) {
        if endpoint != "" { // 防御性校验：零值不覆盖默认
            c.Endpoint = endpoint
        }
    }
}

func WithTimeout(timeout time.Duration) ConfigOption {
    return func(c *Config) {
        if timeout > 0 { // 防御性校验
            c.Timeout = timeout
        }
    }
}

// 4. 构造函数应用 Options
func NewConfig(opts ...ConfigOption) *Config {
    cfg := DefaultConfig()
    for _, opt := range opts {
        opt(cfg)
    }
    return cfg
}
```

**必须**在 Option 函数内部进行零值防御，避免零值覆盖默认值：

```go
func WithTimeout(timeout time.Duration) ConfigOption {
    return func(c *Config) {
        if timeout > 0 { // 零值不覆盖
            c.Timeout = timeout
        }
    }
}
```

❌ **禁止**零值直接覆盖：

```go
func WithTimeout(timeout time.Duration) ConfigOption {
    return func(c *Config) {
        c.Timeout = timeout // 禁止：timeout=0 会覆盖默认的 30s
    }
}
```

#### 5.3 方法接收者

##### 5.3.1 接收者命名

**必须**使用类型首字母的小写作为接收者名：

```go
func (c *Client) Get(...) (...)      // Client → c
func (tm *TaskManager) Submit(...)   // TaskManager → tm
func (bm *BaseManager[T]) ExecuteGet(...) // BaseManager → bm
func (r *PluginRegistry) Register(...)    // PluginRegistry → r
func (m *Meter) Record(...)               // Meter → m
func (t *Tracer) StartSpan(...)           // Tracer → t
```

❌ **禁止**：

```go
func (this *Client) Get(...)    // 禁止：this/self
func (self *Client) Get(...)    // 禁止：this/self
func (client *Client) Get(...)  // 禁止：与类型名相同
```

##### 5.3.2 值接收者 vs 指针接收者

**必须**遵循以下规则：

- 结构体方法**必须**使用指针接收者（除非方法不需要修改且结构体极小）
- 枚举类型（基于 string/int）的 `String()` / `Is` 方法**可以**使用值接收者
- 接口方法**应当**使用指针接收者

```go
// 结构体 → 指针接收者
func (c *Client) Get(...) (*APIResponse, error)

// 枚举 → 值接收者
func (s TaskStatus) String() string { return string(s) }
func (s TaskStatus) IsTerminal() bool { ... }

// 枚举 → 值接收者（ProtocolType 底层为 int）
func (p ProtocolType) String() string { ... }
```

##### 5.3.3 接收者一致性

**禁止**同一类型的某些方法用值接收者、另一些用指针接收者。**必须**全部统一为指针接收者（枚举类型除外）。

#### 5.4 Context 传播

**必须**在所有涉及 I/O、网络、阻塞等待的公共方法中接受 `context.Context` 作为第一个参数：

```go
func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error) // 
func (tm *TaskManager) Wait(ctx context.Context, taskID string, timeout time.Duration) (*types.TaskResult, error) // 
func (c *ProtocolClient) SendRequest(ctx context.Context, method string, params map[string]interface{}) ([]byte, error) // 
```

❌ **禁止**在 I/O 方法中省略 context：

```go
func (tm *TaskManager) Submit(description string) (*types.Task, error) // 禁止：缺少 ctx
```

#### 5.5 可变参数与 Option 链

**必须**使用 `opts ...Option` 可变参数模式替代配置结构体参数：

```go
// 正确：Functional Options
func NewClient(opts ...agentrt.ConfigOption) (*Client, error)

// 正确：Request Options
func (c *Client) Get(ctx context.Context, path string, opts ...types.RequestOption) (*types.APIResponse, error)
```

❌ **禁止**使用大量 bool/int 参数：

```go
func NewClient(endpoint string, timeout time.Duration, retries int, debug bool, apiKey string) // 禁止
```

---

### 6. 错误处理

#### 6.1 AgentRTError 统一错误类型

**必须**使用 `AgentRTError` 作为所有 SDK 错误的统一基类型：

```go
type AgentRTError struct {
    Code    ErrorCode
    Message string
    Cause   error
}
```

**必须**实现 `error`、`Unwrap()`、`Is()` 三个方法：

```go
func (e *AgentRTError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *AgentRTError) Unwrap() error {
    return e.Cause
}

func (e *AgentRTError) Is(target error) bool {
    t, ok := target.(*AgentRTError)
    if !ok {
        return false
    }
    return e.Code == t.Code
}
```

#### 6.2 错误工厂函数

**必须**使用以下三个工厂函数创建错误，**禁止**直接构造 `AgentRTError` 结构体：

```go
// 无原因错误
err := agentrt.NewError(agentrt.CodeNotFound, "资源未找到", nil)          // 

// 格式化消息
err := agentrt.NewErrorf(agentrt.CodeTaskTimeout, "任务 %s 超时", taskID) // 

// 包装已有错误
err := agentrt.WrapError(agentrt.CodeNetworkError, "网络异常", cause)     // 
```

❌ **禁止**：

```go
// 禁止：直接构造结构体
err := &agentrt.AgentRTError{Code: agentrt.CodeNotFound, Message: "未找到"}

// 禁止：使用标准 errors.New
err := errors.New("something failed")

// 禁止：使用 fmt.Errorf
err := fmt.Errorf("something failed: %w", cause)
```

**例外**：在非 SDK 核心包（如 `utils` 内部辅助函数）中，**可以**使用 `fmt.Errorf` 返回简单错误，但该错误最终**必须**在上层被 `WrapError` 包装。

#### 6.3 哨兵错误

**必须**为每种错误码定义对应的哨兵错误变量，支持 `errors.Is` 语义匹配：

```go
var (
    ErrNotFound       = NewError(CodeNotFound, "资源未找到", nil)
    ErrTimeout        = NewError(CodeTimeout, "操作超时", nil)
    ErrInvalidConfig  = NewError(CodeInvalidConfig, "配置无效", nil)
    ErrNetworkError   = NewError(CodeNetworkError, "网络错误", nil)
)
```

使用方式：

```go
if errors.Is(err, agentrt.ErrNotFound) {
    // 处理未找到
}
```

#### 6.4 错误分类查询

**必须**使用分类查询函数判断错误类别，**禁止**直接比较错误码字符串：

```go
// 正确：使用分类查询
if agentrt.IsNetworkError(err) {
    // 处理网络错误
}
if agentrt.IsServerError(err) {
    // 处理服务端错误
}
if agentrt.IsErrorCode(err, agentrt.CodeNotFound) {
    // 处理特定错误码
}
```

❌ **禁止**：

```go
// 禁止：直接比较错误码字符串
if err != nil && strings.Contains(err.Error(), "0x0005") { ... }

// 禁止：类型断言后手动比较
if agentErr, ok := err.(*AgentRTError); ok && agentErr.Code == "0x0005" { ... }
```

#### 6.5 HTTP 状态码映射

**必须**使用 `HTTPStatusToError()` 将 HTTP 状态码转换为 SDK 错误，**禁止**直接返回 HTTP 错误：

```go
// 正确
if resp.StatusCode >= 400 {
    lastErr = agentrt.HTTPStatusToError(resp.StatusCode, string(respBody))
}
```

❌ **禁止**：

```go
// 禁止：直接返回 HTTP 状态码
return nil, fmt.Errorf("HTTP %d", resp.StatusCode)
```

#### 6.6 错误消息语言

**必须**在哨兵错误和面向用户的错误消息中使用中文描述：

```go
ErrNotFound       = NewError(CodeNotFound, "资源未找到", nil)         // 
ErrTaskFailed     = NewError(CodeTaskFailed, "任务执行失败", nil)      // 
ErrSessionExpired = NewError(CodeSessionExpired, "会话已过期", nil)    // 
```

Mock 和测试辅助代码中的错误消息**可以**使用英文：

```go
return nil, agentrt.NewError(agentrt.CodeNotSupported, "Mock GET handler not configured", nil) // 
```

---

### 7. 并发编程

#### 7.1 sync.RWMutex

**必须**在包含可变共享状态的结构体中使用 `sync.RWMutex` 保护并发访问：

```go
type PluginRegistry struct {
    mu        sync.RWMutex       // 互斥锁
    factories map[string]PluginFactory
    instances map[string]BasePlugin
    manifests map[string]PluginManifest
    states    map[string]PluginState
}
```

**必须**遵循读写分离的锁策略：

```go
// 读操作使用 RLock
func (r *PluginRegistry) Get(pluginID string) (BasePlugin, bool) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    instance, exists := r.instances[pluginID]
    return instance, exists
}

// 写操作使用 Lock
func (r *PluginRegistry) Register(factory PluginFactory, manifest *PluginManifest) (string, error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    // ...
}
```

❌ **禁止**读写操作都使用写锁：

```go
func (r *PluginRegistry) Get(pluginID string) (BasePlugin, bool) {
    r.mu.Lock()           // 禁止：读操作不应使用写锁
    defer r.mu.Unlock()
    // ...
}
```

#### 7.2 sync.Once

**必须**使用 `sync.Once` 实现全局单例的延迟初始化：

```go
var globalRegistry *PluginRegistry
var globalRegistryOnce sync.Once

func GetPluginRegistry() *PluginRegistry {
    globalRegistryOnce.Do(func() {
        globalRegistry = NewPluginRegistry()
    })
    return globalRegistry
}
```

❌ **禁止**使用非线程安全的懒加载：

```go
var globalRegistry *PluginRegistry

func GetPluginRegistry() *PluginRegistry {
    if globalRegistry == nil {     // 禁止：竞态条件
        globalRegistry = NewPluginRegistry()
    }
    return globalRegistry
}
```

#### 7.3 Context 传播与取消

**必须**在所有阻塞操作中检查 `ctx.Done()`：

```go
func (tm *TaskManager) Wait(ctx context.Context, taskID string, timeout time.Duration) (*types.TaskResult, error) {
    for {
        // ... 业务逻辑 ...

        select {
        case <-ctx.Done():
            return nil, ctx.Err()           // 响应上下文取消
        case <-time.After(500 * time.Millisecond):
        }
    }
}
```

**必须**在重试循环中检查上下文取消：

```go
for attempt := 0; attempt <= c.config.MaxRetries; attempt++ {
    if attempt > 0 {
        select {
        case <-ctx.Done():
            return nil, agentrt.WrapError(agentrt.CodeTimeout, "请求在重试等待中被取消", ctx.Err()) // 
        case <-time.After(delay):
        }
    }
    // ...
}
```

#### 7.4 Channel 与 Goroutine 安全

**必须**使用缓冲 channel 防止 goroutine 泄漏：

```go
resultCh := make(chan *types.TaskResult, len(taskIDs))  // 缓冲大小 = goroutine 数
errCh := make(chan error, len(taskIDs))
```

**必须**使用 `sync.WaitGroup` 等待所有 goroutine 完成：

```go
var wg sync.WaitGroup
for _, id := range taskIDs {
    wg.Add(1)
    go func(taskID string) {
        defer wg.Done()
        // ...
    }(id)
}

done := make(chan struct{})
go func() {
    wg.Wait()
    close(done)
}()
```

**必须**在 goroutine 中捕获循环变量（通过参数传递）：

```go
for i, id := range taskIDs {
    wg.Add(1)
    go func(idx int, taskID string) { // 通过参数捕获
        defer wg.Done()
        // ...
    }(i, id)
}
```

❌ **禁止**直接在 goroutine 中引用循环变量：

```go
for i, id := range taskIDs {
    go func() {
        // 禁止：i 和 id 可能已被修改
        result, err := tm.Wait(ctx, id, timeout)
    }()
}
```

#### 7.5 defer 与锁释放

**必须**在获取锁后立即使用 `defer` 释放锁：

```go
func (r *PluginRegistry) Register(...) (string, error) {
    r.mu.Lock()
    defer r.mu.Unlock() // 立即 defer
    // ...
}
```

❌ **禁止**在函数末尾手动释放锁：

```go
func (r *PluginRegistry) Register(...) (string, error) {
    r.mu.Lock()
    // ... 多个 return 路径 ...
    r.mu.Unlock() // 禁止：容易遗漏
}
```

**例外**：当锁的粒度需要精确控制时（如 `PluginRegistry.Load` 中先加锁再解锁再重新加锁的模式），**可以**手动管理锁，但**必须**在代码注释中说明原因。

---

### 8. 测试规范

#### 8.1 表驱动测试

**必须**使用表驱动测试（Table-Driven Tests）覆盖多场景：

```go
func TestIsNetworkError(t *testing.T) {
    tests := []struct {
        code  ErrorCode
        isNet bool
    }{
        {CodeNetworkError, true},
        {CodeTimeout, true},
        {CodeConnectionRefused, true},
        {CodeNotFound, false},
        {CodeServerError, false},
    }
    for _, tt := range tests {
        err := NewError(tt.code, "test", nil)
        if IsNetworkError(err) != tt.isNet {
            t.Errorf("IsNetworkError(%s) = %v, want %v", tt.code, IsNetworkError(err), tt.isNet)
        }
    }
}
```

❌ **禁止**为每个场景写独立测试函数（当逻辑相同时）：

```go
func TestIsNetworkError_NetworkError(t *testing.T) { ... }
func TestIsNetworkError_Timeout(t *testing.T) { ... }
func TestIsNetworkError_NotFound(t *testing.T) { ... }
// 禁止：应合并为表驱动测试
```

#### 8.2 Mock 模式

**必须**使用函数字段 Mock 模式替代接口嵌入：

```go
type MockAPIClient struct {
    GetFn  func(ctx context.Context, path string, opts ...types.RequestOption) (*types.APIResponse, error)
    PostFn func(ctx context.Context, path string, body interface{}, opts ...types.RequestOption) (*types.APIResponse, error)
    PutFn  func(ctx context.Context, path string, body interface{}, opts ...types.RequestOption) (*types.APIResponse, error)
    DelFn  func(ctx context.Context, path string, opts ...types.RequestOption) (*types.APIResponse, error)
}

func (m *MockAPIClient) Get(ctx context.Context, path string, opts ...types.RequestOption) (*types.APIResponse, error) {
    if m.GetFn != nil {
        return m.GetFn(ctx, path, opts...)
    }
    return nil, agentrt.NewError(agentrt.CodeNotSupported, "Mock GET handler not configured", nil)
}
```

使用方式：

```go
mock := &client.MockAPIClient{
    PostFn: func(ctx context.Context, path string, body interface{}, opts ...types.RequestOption) (*types.APIResponse, error) {
        return &types.APIResponse{Success: true, Data: map[string]interface{}{"task_id": "t1"}}, nil
    },
}
tm := NewTaskManager(mock)
```

❌ **禁止**在生产代码中引用 Mock 类型。Mock **必须**仅存在于测试文件中。

#### 8.3 测试函数命名

**必须**使用 `Test<类型>_<方法>_<场景>` 格式命名测试函数：

```go
func TestTaskManager_Submit_Success(t *testing.T)          // 
func TestTaskManager_Submit_Empty(t *testing.T)            // 
func TestTaskManager_Submit_APIError(t *testing.T)         // 
func TestTaskManager_Wait_Timeout(t *testing.T)            // 
func TestTaskManager_Wait_ContextCancel(t *testing.T)      // 
func TestTaskManager_BatchSubmit_PartialFailure(t *testing.T) // 
```

❌ **禁止**：

```go
func TestSubmit(t *testing.T)        // 禁止：缺少类型和场景
func test_task_submit(t *testing.T)  // 禁止：下划线+小写
func TestTaskManagerSubmitSuccess(t *testing.T) // 禁止：缺少分隔符
```

#### 8.4 环境变量清理

**必须**在测试中设置环境变量后使用 `defer` 清理：

```go
func TestNewConfigFromEnv(t *testing.T) {
    os.Setenv("AIRY_ENDPOINT", "http://env:9999")
    os.Setenv("AIRY_TIMEOUT", "45s")
    defer func() {
        os.Unsetenv("AIRY_ENDPOINT")
        os.Unsetenv("AIRY_TIMEOUT")
    }()

    cfg, err := NewConfigFromEnv()
    // ...
}
```

❌ **禁止**不清理环境变量：

```go
func TestNewConfigFromEnv(t *testing.T) {
    os.Setenv("AIRY_ENDPOINT", "http://env:9999")
    // 禁止：缺少 defer 清理，会污染其他测试
}
```

#### 8.5 基准测试

**应当**为核心操作编写基准测试，使用 `Benchmark` 前缀：

```go
func BenchmarkTaskSubmit(b *testing.B) {
    client := newMockClient()
    tm := NewTaskManager(client)
    ctx := context.Background()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = tm.Submit(ctx, "benchmark task")
    }
}

func BenchmarkConcurrentTaskSubmits(b *testing.B) {
    // ...
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            _, _ = tm.Submit(ctx, "concurrent task")
        }
    })
}
```

#### 8.6 测试文件位置

**必须**将测试文件放在与被测代码相同的包中（白盒测试）：

```
modules/task/manager.go
modules/task/manager_test.go    // 同包白盒测试
modules/task/benchmark_test.go  // 基准测试
```

---

### 9. 文档注释规范

#### 9.1 文件头注释

**必须**在每个源文件头部添加文件头注释，包含以下信息：

```go
// Airymax Go SDK - 统一错误体系
// Version: 0.1.0
// Last updated: 2026-04-05
//
// 定义 SDK 的完整错误类型层级、错误码枚举、哨兵错误和 HTTP 状态码映射。
// 所有异常继承自 AgentRTError，支持 errors.Is/As 链式追踪。
// 对应 Python SDK: exceptions.py
```

**应当**包含以下要素：
- 模块中文名
- 版本号
- 最后更新日期
- 功能概述（中文描述）
- 对应的 Python SDK 文件（如有）

#### 9.2 SPDX 头

**应当**在需要明确版权的文件头部添加 SPDX 标识：

```go
// SPDX-FileCopyrightText: 2026 SPHARX Ltd.
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
```

#### 9.3 公共 API 注释

**必须**为所有导出类型、函数、常量、变量添加 godoc 注释：

```go
// ErrorCode 表示 Airymax SDK 的错误码类型
// Since: 0.1.0
type ErrorCode string

// AgentRTError 是 SDK 所有错误的统一基类
// Since: 0.1.0
type AgentRTError struct {
    Code    ErrorCode
    Message string
    Cause   error
}

// NewError 创建指定错误码的新错误
// Since: 0.1.0
func NewError(code ErrorCode, message string, cause error) *AgentRTError {
    return &AgentRTError{Code: code, Message: message, Cause: cause}
}

// IsNetworkError 判断是否为网络相关错误
// Since: 0.1.0
func IsNetworkError(err error) bool { ... }
```

#### 9.4 版本标注

**必须**为所有公共 API 添加 `Since: x.y.z` 版本标注：

```go
// NewClient 创建新的 API 客户端
// Since: 0.1.0
func NewClient(opts ...agentrt.ConfigOption) (*Client, error)
```

#### 9.5 中文描述

**必须**在注释中使用中文描述功能和语义，代码标识符保持英文：

```go
// TaskManager 管理任务完整生命周期
type TaskManager struct { ... }

// Submit 提交新的执行任务
func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error)

// Wait 阻塞等待任务到达终态，支持超时控制和上下文取消
func (tm *TaskManager) Wait(ctx context.Context, taskID string, timeout time.Duration) (*types.TaskResult, error)
```

#### 9.6 分隔注释

**应当**使用分隔注释组织文件内的逻辑分区：

```go
// ============================================================
// 错误码常量
// 基于 ErrorCodeReference.md 规范，采用十六进制分类体系：
//   0x0xxx 通用错误 (General)
//   0x1xxx 核心循环错误 (CoreLoop)
//   ...
// ============================================================

// ============================================================
// 哨兵错误 (Err 前缀, 支持 errors.Is)
// ============================================================

// ============================================================
// 内部辅助函数
// ============================================================
```

#### 9.7 Mock 警告注释

**必须**在 Mock 类型上添加警告注释：

```go
// MockAPIClient 是 APIClient 接口的 Mock 实现，支持通过函数字段自定义行为
// 仅用于测试，不应在生产代码中引用
type MockAPIClient struct { ... }
```

---

### 10. 依赖管理

#### 10.1 零外部依赖原则

**必须**保持零外部依赖。`go.mod` 中除模块声明外不得包含任何 `require` 条目：

```go
// go.mod
module github.com/spharx/agentrt/toolkit/go

go 1.22
// 无 require — 正确
```

❌ **禁止**引入任何第三方依赖：

```go
// go.mod
module github.com/spharx/agentrt/toolkit/go

go 1.22

require (
    github.com/some/third-party v1.2.3  // 禁止
    go.uber.org/zap v1.27.0              // 禁止
)
```

#### 10.2 标准库优先

**必须**仅使用 Go 标准库。当前项目使用的标准库包包括：

| 标准库包 | 用途 |
|----------|------|
| `net/http` | HTTP 客户端与服务 |
| `encoding/json` | JSON 序列化/反序列化 |
| `context` | 上下文传播与取消 |
| `crypto/rand` | 密码学安全随机数 |
| `sync` | 互斥锁、Once、WaitGroup |
| `time` | 时间操作与定时器 |
| `fmt` | 格式化 I/O |
| `errors` | 错误创建与链式追踪 |
| `os` | 环境变量、标准错误输出 |
| `log` | 简单日志 |
| `strconv` | 字符串与数值转换 |
| `strings` | 字符串操作 |
| `bytes` | 字节切片操作 |
| `io` | I/O 接口与工具 |
| `math` | 数学函数 |
| `math/big` | 大整数运算 |
| `net/url` | URL 解析与构建 |
| `encoding/hex` | 十六进制编解码 |

#### 10.3 go.mod 规范

**必须**满足以下条件：

1. 模块路径**必须**为 `github.com/spharx/agentrt/toolkit/go`
2. Go 版本**必须**为 `1.22` 或更高
3. **禁止**包含任何 `require`、`replace`、`exclude` 指令（除非有架构委员会批准）

#### 10.4 内部包引用

**必须**使用完整模块路径引用内部包：

```go
import (
    "github.com/spharx/agentrt/toolkit/go/agentrt"
    "github.com/spharx/agentrt/toolkit/go/agentrt/client"
    "github.com/spharx/agentrt/toolkit/go/agentrt/types"
    "github.com/spharx/agentrt/toolkit/go/agentrt/utils"
)
```

---

### 11. Airymax 模块编码示例

#### 11.1 Manager 模块模板

以下模板展示如何创建一个新的资源 Manager，遵循 `BaseManager[T]` 泛型基类模式：

```go
// Airymax Go SDK - XXX 管理模块
// Version: 0.1.0
// Last updated: 2026-06-08
//
// 提供 XXX 资源的创建、查询、更新、删除等生命周期管理功能。
// 对应 Python SDK: modules/xxx/__init__.py

package xxx

import (
    "context"
    "fmt"
    "time"

    "github.com/spharx/agentrt/toolkit/go/agentrt"
    "github.com/spharx/agentrt/toolkit/go/agentrt/client"
    "github.com/spharx/agentrt/toolkit/go/agentrt/types"
    "github.com/spharx/agentrt/toolkit/go/agentrt/utils"
)

// XxxManager 管理 XXX 资源完整生命周期
type XxxManager struct {
    api client.APIClient
}

// NewXxxManager 创建新的 XXX 管理器实例
func NewXxxManager(api client.APIClient) *XxxManager {
    return &XxxManager{api: api}
}

// Create 创建新的 XXX 资源
func (xm *XxxManager) Create(ctx context.Context, name string, metadata map[string]interface{}) (*types.Xxx, error) {
    if err := utils.ValidateRequiredString(name, "资源名称"); err != nil {
        return nil, agentrt.NewError(agentrt.CodeMissingParameter, err.Error(), nil)
    }

    body := map[string]interface{}{"name": name}
    if metadata != nil {
        body["metadata"] = metadata
    }

    resp, err := xm.api.Post(ctx, "/api/v1/xxxs", body)
    if err != nil {
        return nil, err
    }

    data, err := utils.ValidateAndExtractData(resp, "XXX 创建响应格式异常")
    if err != nil {
        return nil, agentrt.NewError(agentrt.CodeInvalidResponse, err.Error(), nil)
    }

    return parseXxxFromMap(data), nil
}

// Get 获取指定 XXX 资源的详细信息
func (xm *XxxManager) Get(ctx context.Context, id string) (*types.Xxx, error) {
    if err := utils.ValidateRequiredString(id, "资源ID"); err != nil {
        return nil, agentrt.NewError(agentrt.CodeMissingParameter, err.Error(), nil)
    }

    resp, err := xm.api.Get(ctx, fmt.Sprintf("/api/v1/xxxs/%s", id))
    if err != nil {
        return nil, err
    }

    data, err := utils.ValidateAndExtractData(resp, "XXX 详情响应格式异常")
    if err != nil {
        return nil, agentrt.NewError(agentrt.CodeInvalidResponse, err.Error(), nil)
    }

    return parseXxxFromMap(data), nil
}

// List 列出 XXX 资源，支持分页和过滤
func (xm *XxxManager) List(ctx context.Context, opts *types.ListOptions) ([]types.Xxx, error) {
    path := "/api/v1/xxxs"
    if opts != nil {
        path = utils.BuildURL(path, opts.ToQueryParams())
    }

    resp, err := xm.api.Get(ctx, path)
    if err != nil {
        return nil, err
    }

    return parseXxxList(resp)
}

// Delete 删除指定 XXX 资源
func (xm *XxxManager) Delete(ctx context.Context, id string) error {
    if err := utils.ValidateRequiredString(id, "资源ID"); err != nil {
        return agentrt.NewError(agentrt.CodeMissingParameter, err.Error(), nil)
    }
    _, err := xm.api.Delete(ctx, fmt.Sprintf("/api/v1/xxxs/%s", id))
    return err
}

// parseXxxFromMap 从 map 解析 Xxx 结构
func parseXxxFromMap(data map[string]interface{}) *types.Xxx {
    return &types.Xxx{
        ID:        utils.GetString(data, "xxx_id"),
        Name:      utils.GetString(data, "name"),
        Status:    types.XxxStatus(utils.GetString(data, "status")),
        Metadata:  utils.GetMap(data, "metadata"),
        CreatedAt: utils.ParseTimeFromMap(data, "created_at"),
    }
}

// parseXxxList 从 APIResponse 解析 Xxx 列表
func parseXxxList(resp *types.APIResponse) ([]types.Xxx, error) {
    data, err := utils.ValidateAndExtractData(resp, "XXX 列表响应格式异常")
    if err != nil {
        return nil, agentrt.NewError(agentrt.CodeInvalidResponse, err.Error(), nil)
    }

    items := utils.GetInterfaceSlice(data, "items")
    result := make([]types.Xxx, 0, len(items))
    for _, item := range items {
        if m, ok := item.(map[string]interface{}); ok {
            result = append(result, *parseXxxFromMap(m))
        }
    }
    return result, nil
}
```

#### 11.2 Client 模块模板

以下模板展示如何扩展 API 客户端：

```go
package client

import (
    "context"

    "github.com/spharx/agentrt/toolkit/go/agentrt"
    "github.com/spharx/agentrt/toolkit/go/agentrt/types"
)

// XxxClient 定义 XXX 资源的客户端接口
type XxxClient interface {
    CreateXxx(ctx context.Context, name string, opts ...types.RequestOption) (*types.Xxx, error)
    GetXxx(ctx context.Context, id string, opts ...types.RequestOption) (*types.Xxx, error)
    DeleteXxx(ctx context.Context, id string, opts ...types.RequestOption) error
}

// 确保 XxxClientImpl 实现 XxxClient 接口
var _ XxxClient = (*XxxClientImpl)(nil)

// XxxClientImpl 是 XxxClient 的默认实现
type XxxClientImpl struct {
    api APIClient
}

// NewXxxClient 创建新的 XXX 客户端
func NewXxxClient(api APIClient) *XxxClientImpl {
    return &XxxClientImpl{api: api}
}

// CreateXxx 创建 XXX 资源
func (c *XxxClientImpl) CreateXxx(ctx context.Context, name string, opts ...types.RequestOption) (*types.Xxx, error) {
    resp, err := c.api.Post(ctx, "/api/v1/xxxs", map[string]interface{}{"name": name}, opts...)
    if err != nil {
        return nil, err
    }
    // 解析响应 ...
    return nil, nil
}

// GetXxx 获取 XXX 资源
func (c *XxxClientImpl) GetXxx(ctx context.Context, id string, opts ...types.RequestOption) (*types.Xxx, error) {
    resp, err := c.api.Get(ctx, "/api/v1/xxxs/"+id, opts...)
    if err != nil {
        return nil, err
    }
    // 解析响应 ...
    return nil, nil
}

// DeleteXxx 删除 XXX 资源
func (c *XxxClientImpl) DeleteXxx(ctx context.Context, id string, opts ...types.RequestOption) error {
    _, err := c.api.Delete(ctx, "/api/v1/xxxs/"+id, opts...)
    return err
}
```

#### 11.3 Plugin 模块模板

以下模板展示如何实现自定义插件：

```go
package myplugin

import (
    "github.com/spharx/agentrt/toolkit/go/agentrt/plugin"
)

// MyPlugin 自定义插件实现
type MyPlugin struct {
    plugin.BasePluginImpl // 嵌入默认实现
}

// GetPluginID 返回插件唯一标识
func (p *MyPlugin) GetPluginID() string {
    return "com.spharx.my-plugin"
}

// OnLoad 插件加载时调用
func (p *MyPlugin) OnLoad(context map[string]interface{}) error {
    // 初始化逻辑
    return nil
}

// OnActivate 插件激活时调用
func (p *MyPlugin) OnActivate(context map[string]interface{}) error {
    // 激活逻辑
    return nil
}

// OnDeactivate 插件停用时调用
func (p *MyPlugin) OnDeactivate() error {
    // 清理逻辑
    return nil
}

// OnUnload 插件卸载时调用
func (p *MyPlugin) OnUnload() error {
    // 释放资源
    return nil
}

// OnError 插件错误回调
func (p *MyPlugin) OnError(err error) {
    // 错误处理/日志
}

// GetCapabilities 返回插件能力列表
func (p *MyPlugin) GetCapabilities() []string {
    return []string{"data-processing", "event-handling"}
}

// 注册示例
func ExampleRegister() {
    registry := plugin.NewPluginRegistry()
    manifest := &plugin.PluginManifest{
        PluginID:     "com.spharx.my-plugin",
        Name:         "My Plugin",
        Version:      "0.1.1",
        Description:  "示例插件",
        Capabilities: []string{"data-processing"},
    }
    id, err := registry.Register(func() plugin.BasePlugin { return &MyPlugin{} }, manifest)
    if err != nil {
        panic(err)
    }
    _ = id
}
```

#### 11.4 Syscall 便捷类型模板

以下模板展示如何添加新的 Syscall 便捷类型：

```go
// NewXxxSyscall 创建新的 XXX 系统调用便捷类型
type XxxSyscall struct {
    binding SyscallBinding
}

func NewXxxSyscall(binding SyscallBinding) *XxxSyscall {
    return &XxxSyscall{binding: binding}
}

func (x *XxxSyscall) SomeOperation(ctx context.Context, param string) (*SyscallResponse, error) {
    return x.binding.Invoke(ctx, &SyscallRequest{
        Namespace: NamespaceXxx,     // 使用 SyscallNamespace 常量
        Operation: "some_operation", // snake_case 操作名
        Params:    map[string]any{"param": param},
    })
}
```

---

### 附录 A：检查清单

代码审查时，审查者**应当**使用以下清单逐项确认：

| # | 检查项 | 规则 |
|---|--------|------|
| 1 | 包名是否全小写无下划线 | §2.4 |
| 2 | 导出类型是否 PascalCase | §3.1 |
| 3 | 构造函数是否 New 前缀 | §3.4 |
| 4 | Option 函数是否 With 前缀 | §3.5 |
| 5 | 判断函数是否 Is 前缀 | §3.6 |
| 6 | 错误码是否 Code 前缀 + 十六进制 | §3.7.2 |
| 7 | 哨兵错误是否 Err 前缀 | §3.8 |
| 8 | 接口是否语义名（无 I 前缀） | §3.9 |
| 9 | 是否有编译期接口满足性检查 | §4.3.2 |
| 10 | Option 函数是否零值防御 | §5.2 |
| 11 | 接收者是否类型首字母小写 | §5.3.1 |
| 12 | I/O 方法是否接受 context.Context | §5.4 |
| 13 | 错误是否使用 AgentRTError 体系 | §6.1 |
| 14 | 是否使用工厂函数创建错误 | §6.2 |
| 15 | 并发结构体是否有 sync.RWMutex | §7.1 |
| 16 | 阻塞操作是否检查 ctx.Done() | §7.3 |
| 17 | 测试是否使用表驱动 | §8.1 |
| 18 | 环境变量测试是否 defer 清理 | §8.4 |
| 19 | 公共 API 是否有 godoc + Since 标注 | §9.3-9.4 |
| 20 | 是否引入了外部依赖 | §10.1 |

### 附录 B：Linter 配置建议

推荐的 `golangci-lint` 配置：

```yaml
# .golangci.yml
run:
  go: "1.22"
  timeout: 5m

linters:
  enable:
    - govet
    - staticcheck
    - errcheck
    - gofmt
    - goimports
    - ineffassign
    - typecheck
    - unused
    - gosimple
    - misspell

linters-settings:
  govet:
    enable-all: true
  staticcheck:
    checks: ["all"]

issues:
  max-issues-per-linter: 50
  max-same-issues: 10
```

---

> **文档维护者**: Airymax 架构组
> **下次审查日期**: 2026-09-08

---

## Part II: Airymax Go 安全编码规范

### 目录

1. [总则与安全架构映射](#1-总则与安全架构映射)
2. [输入验证](#2-输入验证)
3. [密码学使用规范](#3-密码学使用规范)
4. [并发安全](#4-并发安全)
5. [错误处理安全](#5-错误处理安全)
6. [依赖安全](#6-依赖安全)
7. [网络安全](#7-网络安全)
8. [资源管理](#8-资源管理)
9. [审计与可观测性](#9-审计与可观测性)
10. [Airymax 模块安全编码示例](#10-agentrt-模块安全编码示例)

---

### 1. 总则与安全架构映射

#### 1.1 Cupolas 四层防护体系

Airymax 采用 **Cupolas 四层防护** 安全架构，每一层对应 Go SDK 中的具体安全机制。所有 Go 代码必须遵循该架构的约束，不得绕过任何一层防护。

| 防护层 | 代号 | 安全目标 | Go SDK 对应机制 |
|--------|------|----------|-----------------|
| D1: Sandbox Isolation | D1 | 插件与核心运行时隔离 | `PluginManager.sandboxEnabled`、`PluginManifest` 资源限制、`PluginState` 状态机 |
| D2: Permission Arbitration | D2 | 操作权限仲裁 | `PluginManifest.Permissions`、`CodePermissionDenied` (0x6001)、`SyscallBinding` 接口 |
| D3: Input Sanitization | D3 | 输入净化与验证 | `utils.SanitizeString()`、`utils.ValidateRequiredString()`、`Config.Validate()` |
| D4: Audit Trail | D4 | 操作审计追踪 | `telemetry.Meter`/`Tracer`、`X-Request-ID` 头、`BaseManager.LogOperation()` |

#### 1.2 安全编码基本原则

- **纵深防御**: 每一层防护独立生效，不得因上层防护存在而省略下层防护
- **最小权限**: 插件仅声明所需权限，默认拒绝一切未声明的能力
- **安全默认**: 所有配置项必须提供安全的默认值（如 `sandboxEnabled: true`）
- **零信任输入**: 所有外部输入（网络响应、环境变量、用户参数）均视为不可信

#### 1.3 架构映射规则

**规则 1.3.1** — D1 沙箱隔离：所有插件必须通过 `PluginManifest` 声明资源上限

❌ **错误**：未设置资源限制的插件清单

```go
manifest := &plugin.PluginManifest{
    PluginID: "my-plugin",
    Name:     "My Plugin",
    // 缺少 MaxMemoryMB / MaxCPUPercent / MaxThreads / TimeoutSeconds
}
```

**正确**：所有资源限制字段必须显式设置

```go
manifest := &plugin.PluginManifest{
    PluginID:       "my-plugin",
    Name:           "My Plugin",
    MaxMemoryMB:    256,
    MaxCPUPercent:  50.0,
    MaxThreads:     4,
    TimeoutSeconds: 30.0,
}
```

**执行机制**: CI 中通过静态分析检查 `PluginManifest` 结构体字面量是否包含所有资源限制字段。

---

**规则 1.3.2** — D2 权限仲裁：插件操作必须通过 `SyscallBinding` 接口，禁止直接访问底层资源

❌ **错误**：插件直接操作 HTTP 客户端

```go
func (p *MyPlugin) OnActivate(ctx map[string]interface{}) error {
    resp, _ := http.Get("http://internal-service/admin/config")
    // 绕过了权限仲裁层
    return nil
}
```

**正确**：通过系统调用接口发起请求，由仲裁层决定是否放行

```go
func (p *MyPlugin) OnActivate(ctx map[string]interface{}) error {
    resp, err := p.binding.Invoke(context.Background(), &syscall.SyscallRequest{
        Namespace: syscall.NamespaceTask,
        Operation: "query",
        Params:    map[string]any{"task_id": "t-001"},
    })
    return err
}
```

**执行机制**: 代码审查禁止插件包内直接导入 `net/http` 等网络包。

---

**规则 1.3.3** — D3 输入净化：所有外部输入必须经过验证或净化后方可使用

❌ **错误**：直接使用用户输入构造查询

```go
func (sm *SessionManager) Get(ctx context.Context, sessionID string) (*types.Session, error) {
    // 未验证 sessionID
    resp, err := sm.api.Get(ctx, fmt.Sprintf("/api/v1/sessions/%s", sessionID))
    // ...
}
```

**正确**：先验证再使用

```go
func (sm *SessionManager) Get(ctx context.Context, sessionID string) (*types.Session, error) {
    if sessionID == "" {
        return nil, agentrt.NewError(agentrt.CodeMissingParameter, "会话ID不能为空", nil)
    }
    resp, err := sm.api.Get(ctx, fmt.Sprintf("/api/v1/sessions/%s", sessionID))
    // ...
}
```

**执行机制**: CI 中通过 linter 规则检查公共方法是否对字符串参数进行空值校验。

---

**规则 1.3.4** — D4 审计追踪：所有关键操作必须记录遥测数据

❌ **错误**：关键操作无审计记录

```go
func (tm *TaskManager) Cancel(ctx context.Context, taskID string) error {
    _, err := tm.api.Post(ctx, fmt.Sprintf("/api/v1/tasks/%s/cancel", taskID), nil)
    return err
}
```

**正确**：操作前后记录遥测

```go
func (tm *TaskManager) Cancel(ctx context.Context, taskID string) error {
    span := tm.telemetry.Tracer.StartSpan("task.cancel")
    defer func() {
        tm.telemetry.Tracer.FinishSpan(span)
        tm.telemetry.Meter.Record("task.cancel.count", 1, map[string]string{"task_id": taskID})
    }()

    _, err := tm.api.Post(ctx, fmt.Sprintf("/api/v1/tasks/%s/cancel", taskID), nil)
    if err != nil {
        span.Status = types.SpanStatusError
    }
    return err
}
```

**执行机制**: 代码审查检查 `Cancel`/`Delete`/`Submit` 等变更类操作是否包含遥测记录。

---

### 2. 输入验证

#### 2.1 参数校验

**规则 2.1.1** — 所有公共 API 的必填参数必须进行非空/有效性校验

SDK 中已提供 `utils.ValidateRequiredString()`、`utils.ValidatePositiveNumber()`、`utils.ValidateNonEmptySlice()` 三个校验函数，必须优先使用。

❌ **错误**：跳过参数校验

```go
func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error) {
    resp, err := tm.api.Post(ctx, "/api/v1/tasks", map[string]interface{}{
        "description": description,
    })
    // ...
}
```

**正确**：使用 SDK 校验工具

```go
func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error) {
    if err := utils.ValidateRequiredString(description, "任务描述"); err != nil {
        return nil, agentrt.NewError(agentrt.CodeMissingParameter, err.Error(), nil)
    }
    resp, err := tm.api.Post(ctx, "/api/v1/tasks", map[string]interface{}{
        "description": description,
    })
    // ...
}
```

**执行机制**: 代码审查 + 单元测试覆盖空值/零值/负值边界。

---

**规则 2.1.2** — 配置对象必须实现 `Validate()` 方法并在构造时调用

参考 `Config.Validate()` 的实现模式：

```go
func (c *Config) Validate() error {
    if c.Endpoint == "" {
        return ErrInvalidEndpoint
    }
    if parsed, err := url.Parse(c.Endpoint); err != nil || (parsed.Scheme != "http" && parsed.Scheme != "https") {
        return NewError(CodeInvalidEndpoint, "端点地址必须以 http:// 或 https:// 开头", nil)
    }
    if c.Timeout <= 0 {
        return NewError(CodeInvalidConfig, "超时时间必须大于零", nil)
    }
    if c.MaxConnections <= 0 {
        return NewError(CodeInvalidConfig, "最大连接数必须大于零", nil)
    }
    return nil
}
```

❌ **错误**：构造配置后不验证

```go
func NewClient(opts ...agentrt.ConfigOption) (*Client, error) {
    config := agentrt.NewConfig(opts...)
    // 未调用 config.Validate()
    return newClientWithConfig(config)
}
```

**正确**：构造后立即验证

```go
func NewClient(opts ...agentrt.ConfigOption) (*Client, error) {
    config := agentrt.NewConfig(opts...)
    if err := config.Validate(); err != nil {
        return nil, err
    }
    return newClientWithConfig(config)
}
```

**执行机制**: 构造函数中 `Validate()` 调用为强制要求，CI 静态分析检查。

---

#### 2.2 JSON 反序列化安全

**规则 2.2.1** — 禁止将未经验证的 JSON 数据直接反序列化为 `interface{}` 后使用

必须通过 `utils.GetString()`、`utils.GetInt64()` 等类型安全提取函数访问反序列化后的数据。

❌ **错误**：直接类型断言，可能 panic

```go
var result map[string]interface{}
json.Unmarshal(body, &result)
taskID := result["task_id"].(string) // 若 key 不存在或类型不匹配将 panic
```

**正确**：使用类型安全提取

```go
var result map[string]interface{}
json.Unmarshal(body, &result)
taskID := utils.GetString(result, "task_id") // 安全返回空字符串
priority := int(utils.GetInt64(result, "priority")) // 安全返回零值
```

**执行机制**: 代码审查禁止在 `map[string]interface{}` 上使用裸类型断言（`.（string)`），必须使用 `utils` 包函数或带 `ok` 的安全断言。

---

**规则 2.2.2** — 反序列化外部 JSON 前必须验证数据完整性

❌ **错误**：未验证响应结构

```go
var apiResp types.APIResponse
json.Unmarshal(respBody, &apiResp)
return &apiResp, nil // 可能返回 Success=false 的响应
```

**正确**：验证后提取

```go
var apiResp types.APIResponse
if err := json.Unmarshal(respBody, &apiResp); err != nil {
    return nil, agentrt.WrapError(agentrt.CodeParseError, "解析响应失败", err)
}
if !apiResp.Success {
    return nil, agentrt.NewError(agentrt.CodeInvalidResponse, apiResp.Message, nil)
}
return &apiResp, nil
```

**执行机制**: 代码审查 + 集成测试验证异常响应处理。

---

#### 2.3 URL 构建安全

**规则 2.3.1** — URL 查询参数必须通过 `url.Values` 编码，禁止手动拼接

SDK 已提供 `utils.BuildURL()` 函数，内部使用 `url.Values.Encode()` 进行安全编码。

❌ **错误**：手动拼接查询参数

```go
path := fmt.Sprintf("/api/v1/tasks?page=%d&page_size=%d", page, pageSize)
// 未对参数进行 URL 编码，特殊字符可能导致注入
```

**正确**：使用 `BuildURL` 或 `url.Values`

```go
path := utils.BuildURL("/api/v1/tasks", map[string]string{
    "page":      fmt.Sprintf("%d", page),
    "page_size": fmt.Sprintf("%d", pageSize),
})
```

**执行机制**: 代码审查禁止 `fmt.Sprintf` 构建含查询参数的 URL。

---

**规则 2.3.2** — URL 路径中的用户输入必须经过净化

❌ **错误**：直接嵌入用户输入到路径

```go
resp, err := sm.api.Get(ctx, fmt.Sprintf("/api/v1/sessions/%s/context/%s", sessionID, key))
// 若 key 包含 "/" 或 ".." 可导致路径遍历
```

**正确**：净化后再嵌入

```go
safeKey := utils.SanitizeString(key)
safeKey = url.PathEscape(safeKey)
resp, err := sm.api.Get(ctx, fmt.Sprintf("/api/v1/sessions/%s/context/%s", sessionID, safeKey))
```

**执行机制**: 代码审查检查路径参数是否经过 `SanitizeString` + `url.PathEscape` 处理。

---

### 3. 密码学使用规范

#### 3.1 随机数生成

**规则 3.1.1** — 所有安全敏感场景必须使用 `crypto/rand`，严禁使用 `math/rand`

SDK 中 `client.generateRequestID()`、`protocol.generateID()`、`utils.GenerateID()` 均使用 `crypto/rand` 作为安全范例。

❌ **错误**：使用 `math/rand` 生成请求 ID 或令牌

```go
import "math/rand"

func generateToken() string {
    return fmt.Sprintf("token-%d", rand.Intn(1000000))
}
```

**正确**：使用 `crypto/rand`

```go
import (
    "crypto/rand"
    "math/big"
)

func generateRequestID() string {
    timestamp := time.Now().UnixNano()
    randomNum, _ := rand.Int(rand.Reader, big.NewInt(1000000))
    return fmt.Sprintf("req-%d-%06d", timestamp, randomNum.Int64())
}
```

**执行机制**: CI 中通过 `go vet` + 自定义 linter 禁止 `math/rand` 导入（测试代码除外，但必须使用 `math/rand/v2` 并标注 `//nolint:mathrand`）。

---

**规则 3.1.2** — `crypto/rand.Read()` 的错误必须检查

❌ **错误**：忽略 `rand.Read` 错误

```go
b := make([]byte, 16)
rand.Read(b) // 忽略了错误返回值
```

**正确**：检查错误

```go
b := make([]byte, 16)
if _, err := rand.Read(b); err != nil {
    return "", agentrt.WrapError(agentrt.CodeInternal, "生成随机数失败", err)
}
```

> **注意**: SDK 中 `utils.GenerateID()` 使用 `_, _ = rand.Read(b)` 忽略了错误，这是一个已知的技术债务，应在后续版本中修复。新代码必须检查错误。

**执行机制**: 代码审查 + `errcheck` linter 检查 `crypto/rand.Read` 返回值。

---

#### 3.2 TLS 配置

**规则 3.2.1** — 生产环境必须使用 TLS 1.2+，禁止降级到 TLS 1.0/1.1

❌ **错误**：允许旧版 TLS

```go
transport := &http.Transport{
    TLSClientConfig: &tls.Config{
        MinVersion: tls.VersionTLS10, // 不安全
    },
}
```

**正确**：强制 TLS 1.2+

```go
transport := &http.Transport{
    TLSClientConfig: &tls.Config{
        MinVersion:         tls.VersionTLS12,
        PreferServerCipherSuites: true,
        CipherSuites: []uint16{
            tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
            tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        },
    },
}
```

**执行机制**: CI 静态分析扫描 `tls.Config` 中 `MinVersion` 设置。

---

**规则 3.2.2** — 禁止跳过证书验证

❌ **错误**：禁用证书验证

```go
transport := &http.Transport{
    TLSClientConfig: &tls.Config{
        InsecureSkipVerify: true, // 绝对禁止
    },
}
```

**正确**：仅在测试中使用自定义 CA，且必须通过配置控制

```go
if config.Debug {
    // 仅在 Debug 模式下允许自定义 CA pool
    certPool := x509.NewCertPool()
    certPool.AppendCertsFromPEM(config.CACert)
    transport.TLSClientConfig = &tls.Config{
        RootCAs:    certPool,
        MinVersion: tls.VersionTLS12,
    }
}
```

**执行机制**: 代码审查禁止 `InsecureSkipVerify: true`，CI 扫描该模式。

---

#### 3.3 密钥管理

**规则 3.3.1** — API Key 不得硬编码在源代码中

SDK 通过 `Config.APIKey` 和环境变量 `AIRY_API_KEY` 传递密钥。

❌ **错误**：硬编码密钥

```go
config := agentrt.NewConfig(
    agentrt.WithAPIKey("sk-abc123def456"), // 硬编码
)
```

**正确**：从环境变量读取

```go
config, err := agentrt.NewConfigFromEnv()
// 或
config := agentrt.NewConfig(
    agentrt.WithAPIKey(os.Getenv("AIRY_API_KEY")),
)
```

**执行机制**: pre-commit hook 扫描 `sk-`、`api_key=` 等密钥模式。

---

**规则 3.3.2** — 密钥不得出现在日志、错误信息或 `String()` 方法中

❌ **错误**：密钥泄露到日志

```go
func (c *Client) String() string {
    return fmt.Sprintf("Client[endpoint=%s, apiKey=%s]", c.config.Endpoint, c.config.APIKey)
}
```

**正确**：脱敏处理

```go
func (c *Client) String() string {
    return fmt.Sprintf("Client[endpoint=%s, timeout=%v]", c.config.Endpoint, c.config.Timeout)
}
```

**执行机制**: 代码审查检查 `String()`、`Error()`、日志格式字符串是否包含 `APIKey` 字段。

---

### 4. 并发安全

#### 4.1 Mutex 使用

**规则 4.1.1** — 共享可变状态必须通过 `sync.RWMutex` 保护，读多写少场景优先使用读锁

SDK 中 `PluginRegistry`、`PluginManager`、`Meter`、`Tracer`、`MockSyscallBinding` 均使用 `sync.RWMutex` 作为范例。

❌ **错误**：无锁保护共享状态

```go
type Cache struct {
    items map[string]string
}

func (c *Cache) Get(key string) string {
    return c.items[key] // 数据竞争
}
```

**正确**：使用 `sync.RWMutex`

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *Cache) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.items[key]
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}
```

**执行机制**: `go test -race` 必须通过，CI 中强制启用竞态检测。

---

**规则 4.1.2** — 禁止在持有锁时执行可能阻塞的操作（I/O、网络请求、长时间计算）

参考 `PluginRegistry.Unregister()` 的实现：先释放写锁，执行 `OnUnload()`，再重新获取写锁。

❌ **错误**：持锁执行回调

```go
func (r *PluginRegistry) Unregister(pluginID string) bool {
    r.mu.Lock()
    defer r.mu.Unlock() // OnUnload 可能耗时很长，锁持有时间过长

    if instance, loaded := r.instances[pluginID]; loaded {
        instance.OnUnload() // 危险：持锁回调
    }
    // ...
}
```

**正确**：释放锁后执行回调

```go
func (r *PluginRegistry) Unregister(pluginID string) bool {
    r.mu.Lock()
    if _, exists := r.factories[pluginID]; !exists {
        r.mu.Unlock()
        return false
    }
    if instance, loaded := r.instances[pluginID]; loaded {
        r.mu.Unlock()              // 先释放锁
        if err := instance.OnUnload(); err != nil {
            instance.OnError(err)
        }
        r.mu.Lock()               // 再获取锁
        delete(r.instances, pluginID)
    }
    delete(r.factories, pluginID)
    delete(r.manifests, pluginID)
    delete(r.states, pluginID)
    r.mu.Unlock()
    return true
}
```

**执行机制**: 代码审查检查持锁区间是否包含 I/O 或回调调用。

---

#### 4.2 Goroutine 安全

**规则 4.2.1** — 启动 goroutine 时必须确保其生命周期可控（可通过 context 取消或 channel 退出）

参考 `TaskManager.WaitForAny()` 的实现：通过 `context.WithTimeout` 和 `done` channel 控制 goroutine 退出。

❌ **错误**：无法退出的 goroutine

```go
go func() {
    for {
        doWork() // 无退出条件，goroutine 泄漏
    }
}()
```

**正确**：通过 context 控制退出

```go
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            doWork()
        }
    }
}()
```

**执行机制**: 代码审查 + `goleak` 测试检测 goroutine 泄漏。

---

**规则 4.2.2** — goroutine 中的 panic 必须被 recover，不得导致整个进程崩溃

❌ **错误**：未 recover 的 goroutine

```go
go func() {
    result := doRiskyWork() // 若 panic，进程崩溃
    ch <- result
}()
```

**正确**：recover 并通过 channel 传递错误

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            errCh <- fmt.Errorf("goroutine panic: %v", r)
        }
    }()
    result, err := doRiskyWork()
    if err != nil {
        errCh <- err
        return
    }
    resultCh <- result
}()
```

**执行机制**: 代码审查检查所有 goroutine 启动点是否包含 `defer recover()`。

---

#### 4.3 Channel 安全

**规则 4.3.1** — 生产者-消费者模式必须确保 channel 关闭安全

参考 `TaskManager.WaitForAll()` 的实现：由唯一的生产者 goroutine（`wg.Wait()` 后）关闭 channel。

❌ **错误**：多个 goroutine 关闭同一 channel

```go
for i, id := range taskIDs {
    go func(idx int, taskID string) {
        result, err := tm.Wait(ctx, taskID, timeout)
        if err != nil {
            close(resultCh) // 多次 close 导致 panic
            return
        }
        resultCh <- result
    }(i, id)
}
```

**正确**：单一 goroutine 负责关闭

```go
resultCh := make(chan indexedResult, len(taskIDs))
var wg sync.WaitGroup

for i, id := range taskIDs {
    wg.Add(1)
    go func(idx int, taskID string) {
        defer wg.Done()
        result, err := tm.Wait(ctx, taskID, timeout)
        resultCh <- indexedResult{index: idx, result: result, err: err}
    }(i, id)
}

go func() {
    wg.Wait()
    close(resultCh) // 仅此一处关闭
}()
```

**执行机制**: 代码审查 + `go vet` 检测 channel 关闭模式。

---

#### 4.4 Context 传播

**规则 4.4.1** — 所有跨函数边界的操作必须接受 `context.Context` 作为第一个参数

SDK 中所有公共 API（`APIClient`、`BaseManager`、`SyscallBinding`）均遵循此模式。

❌ **错误**：不接受 context

```go
func (tm *TaskManager) Submit(description string) (*types.Task, error) {
    resp, err := tm.api.Post(context.Background(), "/api/v1/tasks", body)
    // 无法取消，无法超时
}
```

**正确**：接受并传播 context

```go
func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error) {
    if err := utils.ValidateRequiredString(description, "任务描述"); err != nil {
        return nil, agentrt.NewError(agentrt.CodeMissingParameter, err.Error(), nil)
    }
    resp, err := tm.api.Post(ctx, "/api/v1/tasks", map[string]interface{}{
        "description": description,
    })
    // ...
}
```

**执行机制**: `go vet` + 代码审查检查公共方法签名。

---

**规则 4.4.2** — 禁止在结构体中存储 `context.Context`

❌ **错误**：将 context 存储在结构体中

```go
type TaskManager struct {
    ctx context.Context // 禁止
    api client.APIClient
}
```

**正确**：context 作为方法参数传递

```go
type TaskManager struct {
    api client.APIClient
}

func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error) {
    // ctx 作为参数传递
}
```

**执行机制**: `go vet` + 静态分析检查结构体字段类型。

---

### 5. 错误处理安全

#### 5.1 错误信息脱敏

**规则 5.1.1** — 返回给调用方的错误信息不得包含内部实现细节、文件路径、数据库查询等敏感信息

❌ **错误**：泄露内部信息

```go
return nil, fmt.Errorf("database query failed: SELECT * FROM users WHERE id='%s': connection refused at 10.0.1.5:5432", userID)
```

**正确**：使用 SDK 错误码分类，仅返回通用描述

```go
return nil, agentrt.NewError(agentrt.CodeServerError, "服务端错误", nil)
// 内部错误通过结构化日志记录，不返回给调用方
```

**执行机制**: 代码审查检查错误信息是否包含 IP 地址、SQL 语句、文件路径等模式。

---

#### 5.2 错误码分类

**规则 5.2.1** — 所有错误必须使用 `AgentRTError` 体系，按十六进制分类体系分配错误码

SDK 定义的错误码分类：

| 范围 | 分类 | 示例 |
|------|------|------|
| `0x0xxx` | 通用错误 | `CodeInvalidParameter` (0x0002)、`CodeTimeout` (0x0004) |
| `0x1xxx` | 核心循环错误 | `CodeLoopCreateFailed` (0x1001) |
| `0x2xxx` | 认知层错误 | `CodeCognitionFailed` (0x2001) |
| `0x3xxx` | 执行层错误 | `CodeTaskFailed` (0x3001) |
| `0x4xxx` | 记忆层错误 | `CodeMemoryNotFound` (0x4001) |
| `0x5xxx` | 系统调用错误 | `CodeSyscallError` (0x5002) |
| `0x6xxx` | 安全域错误 | `CodePermissionDenied` (0x6001)、`CodeCorruptedData` (0x6002) |

❌ **错误**：使用裸 `fmt.Errorf`

```go
return nil, fmt.Errorf("plugin '%s' not found", pluginID)
```

**正确**：使用 `AgentRTError` 体系

```go
return nil, agentrt.NewErrorf(agentrt.CodeNotFound, "插件 '%s' 未找到", pluginID)
```

**执行机制**: CI 中通过 linter 禁止在公共 API 中使用 `fmt.Errorf` 返回错误（测试代码除外）。

---

#### 5.3 哨兵错误

**规则 5.3.1** — 预定义的错误状态必须使用哨兵错误（`Err` 前缀变量），支持 `errors.Is` 语义匹配

SDK 已定义完整的哨兵错误集合（如 `ErrNotFound`、`ErrPermissionDenied` 等）。

❌ **错误**：字符串比较判断错误

```go
if err.Error() == "not found" {
    // 脆弱的字符串匹配
}
```

**正确**：使用 `errors.Is` 匹配哨兵错误

```go
if errors.Is(err, agentrt.ErrNotFound) {
    // 类型安全的错误匹配
}
```

---

**规则 5.3.2** — 错误链必须通过 `WrapError` 保留原始错误，支持 `errors.As` 链式追踪

❌ **错误**：丢弃原始错误

```go
return nil, agentrt.NewError(agentrt.CodeNetworkError, "请求失败", nil) // Cause 丢失
```

**正确**：包装原始错误

```go
return nil, agentrt.WrapError(agentrt.CodeNetworkError, "请求执行失败", err) // 保留 Cause
```

**执行机制**: 代码审查检查 `NewError` 的第三个参数是否为 `nil`（当存在原始错误时）。

---

#### 5.4 HTTP 状态码映射

**规则 5.4.1** — HTTP 响应状态码必须通过 `HTTPStatusToError()` 映射为 SDK 错误码

❌ **错误**：直接返回 HTTP 状态码

```go
if resp.StatusCode >= 400 {
    return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, string(body))
}
```

**正确**：使用映射函数

```go
if resp.StatusCode >= 400 {
    lastErr = agentrt.HTTPStatusToError(resp.StatusCode, string(body))
    if !shouldRetry(resp.StatusCode) {
        return nil, lastErr
    }
    continue
}
```

**执行机制**: 代码审查检查 HTTP 客户端代码是否使用 `HTTPStatusToError`。

---

### 6. 依赖安全

#### 6.1 零外部依赖原则

**规则 6.1.1** — Airymax Go SDK 必须保持零外部依赖（仅允许 Go 标准库）

当前 `go.mod` 验证：

```
module github.com/spharx/agentrt/toolkit/go

go 1.22
// 无 require 块 — 零外部依赖
```

❌ **错误**：引入第三方依赖

```go
import (
    "github.com/gin-gonic/gin"         // 第三方 Web 框架
    "github.com/go-resty/resty/v2"      // 第三方 HTTP 客户端
    "github.com/sirupsen/logrus"        // 第三方日志库
)
```

**正确**：仅使用标准库

```go
import (
    "net/http"       // 标准库 HTTP
    "log"            // 标准库日志
    "encoding/json"  // 标准库 JSON
    "crypto/rand"    // 标准库密码学
)
```

**执行机制**: CI 中 `go mod tidy && go mod verify` 检查 `go.sum` 是否为空（仅含标准库校验和）。若 `go.mod` 出现 `require` 块，构建失败。

---

#### 6.2 go.sum 审计

**规则 6.2.1** — `go.sum` 文件变更必须经过安全审计

- 每次合并请求中若 `go.sum` 发生变更，必须由安全负责人审查
- 禁止引入任何 `replace` 指令指向本地路径或非官方仓库

❌ **错误**：使用 replace 指令绕过依赖来源

```go
replace (
    github.com/some/package => ../../../local-copy  // 禁止
    github.com/other/package => github.com/fork/package v0.0.0  // 需审计
)
```

**正确**：无 replace 指令

```
module github.com/spharx/agentrt/toolkit/go
go 1.22
// 无 replace 块
```

**执行机制**: CI 检查 `go.mod` 中是否存在 `replace` 指令，存在则构建失败。

---

#### 6.3 go.mod 规范

**规则 6.3.1** — `go.mod` 必须指定最低兼容 Go 版本，且不得低于 Go 1.22

❌ **错误**：过低的 Go 版本

```
go 1.18
```

**正确**：

```
go 1.22
```

**执行机制**: CI 检查 `go.mod` 中 `go` 指令版本 >= 1.22。

---

**规则 6.3.2** — 模块路径必须与仓库路径一致

```
module github.com/spharx/agentrt/toolkit/go
```

**执行机制**: CI 检查 `module` 路径与仓库 URL 匹配。

---

### 7. 网络安全

#### 7.1 HTTP 客户端安全

**规则 7.1.1** — HTTP 客户端必须设置超时，禁止使用零超时（无限等待）

SDK 中 `Client` 和 `ProtocolClient` 均设置了默认超时：

```go
httpClient: &http.Client{
    Timeout: config.Timeout, // 默认 30s
    Transport: &http.Transport{
        MaxIdleConns:    config.MaxConnections, // 默认 10
        IdleConnTimeout: config.IdleConnTimeout, // 默认 90s
    },
}
```

❌ **错误**：零超时

```go
client := &http.Client{} // Timeout 为零，无限等待
```

**正确**：显式设置超时

```go
client := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:    10,
        IdleConnTimeout: 90 * time.Second,
    },
}
```

**执行机制**: 代码审查检查 `http.Client` 构造是否设置 `Timeout`。

---

**规则 7.1.2** — HTTP 请求必须使用 `http.NewRequestWithContext`，支持取消传播

❌ **错误**：不传播 context

```go
req, err := http.NewRequest(http.MethodGet, url, nil)
```

**正确**：传播 context

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
```

**执行机制**: 代码审查 + `go vet` 检查 `http.NewRequest` 使用（应替换为 `NewRequestWithContext`）。

---

#### 7.2 响应体限制

**规则 7.2.1** — HTTP 响应体必须通过 `io.LimitReader` 限制读取大小，防止内存耗尽攻击

SDK 中 `MaxResponseBodySize` 常量定义了 10MB 上限：

```go
const MaxResponseBodySize = 10 * 1024 * 1024

respBody, readErr := io.ReadAll(io.LimitReader(resp.Body, MaxResponseBodySize))
```

❌ **错误**：无限制读取

```go
body, _ := io.ReadAll(resp.Body) // 恶意服务器可返回无限大响应
```

**正确**：限制读取大小

```go
body, err := io.ReadAll(io.LimitReader(resp.Body, MaxResponseBodySize))
if err != nil {
    return nil, agentrt.WrapError(agentrt.CodeParseError, "读取响应失败", err)
}
```

**执行机制**: 代码审查检查所有 `io.ReadAll(resp.Body)` 是否配合 `io.LimitReader` 使用。

---

> **注意**: `protocol.go` 中的 `DetectProtocol()`、`ListProtocols()`、`TestConnection()`、`GetCapabilities()` 等方法直接使用 `io.ReadAll(resp.Body)` 而未通过 `LimitReader`，这是已知的安全缺陷，必须在后续版本中修复。

#### 7.3 请求超时

**规则 7.3.1** — 重试等待期间必须监听 context 取消信号

参考 `Client.request()` 的实现：

```go
select {
case <-ctx.Done():
    return nil, agentrt.WrapError(agentrt.CodeTimeout, "请求在重试等待中被取消", ctx.Err())
case <-time.After(delay):
}
```

❌ **错误**：重试等待不响应取消

```go
time.Sleep(delay) // 无法通过 context 取消
```

**正确**：使用 select 监听 context

```go
select {
case <-ctx.Done():
    return nil, ctx.Err()
case <-time.After(delay):
}
```

**执行机制**: 代码审查禁止重试逻辑中使用 `time.Sleep`。

---

#### 7.4 重试安全

**规则 7.4.1** — 重试必须使用指数退避 + 随机抖动，防止惊群效应

SDK 中 `calculateBackoff()` 使用 `crypto/rand` 生成抖动：

```go
func calculateBackoff(baseDelay time.Duration, attempt int) time.Duration {
    backoff := float64(baseDelay) * math.Pow(2, float64(attempt-1))
    jitterBig, _ := rand.Int(rand.Reader, big.NewInt(int64(backoff)))
    jitter := time.Duration(jitterBig.Int64())
    return time.Duration(backoff) + jitter
}
```

❌ **错误**：固定间隔重试

```go
for attempt := 0; attempt < maxRetries; attempt++ {
    time.Sleep(1 * time.Second) // 固定间隔，所有客户端同时重试
    retry()
}
```

**正确**：指数退避 + 随机抖动

```go
delay := calculateBackoff(baseDelay, attempt)
select {
case <-ctx.Done():
    return ctx.Err()
case <-time.After(delay):
}
```

**执行机制**: 代码审查检查重试逻辑是否使用指数退避。

---

**规则 7.4.2** — 仅对可重试的错误进行重试（5xx + 429），4xx 客户端错误不得重试

SDK 中 `shouldRetry()` 的实现：

```go
func shouldRetry(statusCode int) bool {
    return statusCode >= 500 || statusCode == http.StatusTooManyRequests
}
```

❌ **错误**：对所有错误重试

```go
for attempt := 0; attempt <= c.config.MaxRetries; attempt++ {
    resp, err := c.httpClient.Do(req)
    if err != nil {
        continue // 对所有错误重试，包括 401/403
    }
}
```

**正确**：仅重试可重试错误

```go
if resp.StatusCode >= 400 {
    lastErr = agentrt.HTTPStatusToError(resp.StatusCode, string(respBody))
    if !shouldRetry(resp.StatusCode) {
        return nil, lastErr // 4xx 不重试
    }
    continue // 仅 5xx 和 429 重试
}
```

**执行机制**: 代码审查检查重试条件是否排除了 4xx。

---

**规则 7.4.3** — 重试时必须重置请求体（对于可寻址的 Body）

参考 `Client.request()` 的实现：

```go
if seeker, ok := req.Body.(io.Seeker); ok && req.Body != nil {
    if _, seekErr := seeker.Seek(0, io.SeekStart); seekErr != nil {
        return nil, agentrt.WrapError(agentrt.CodeNetworkError, "重置请求体失败", seekErr)
    }
}
```

❌ **错误**：重试时不重置 Body

```go
// 使用 bytes.NewReader 创建 Body，重试时 Body 已被读取到末尾
req, _ := http.NewRequestWithContext(ctx, method, url, bytes.NewReader(jsonData))
// 第二次重试时发送空 Body
```

**正确**：重试前重置 Body

```go
if seeker, ok := req.Body.(io.Seeker); ok && req.Body != nil {
    if _, seekErr := seeker.Seek(0, io.SeekStart); seekErr != nil {
        return nil, agentrt.WrapError(agentrt.CodeNetworkError, "重置请求体失败", seekErr)
    }
}
```

**执行机制**: 代码审查检查重试逻辑是否包含 Body 重置。

---

#### 7.5 认证安全

**规则 7.5.1** — API Key 必须通过 `Authorization: Bearer` 头传递，不得通过 URL 参数传递

❌ **错误**：通过 URL 参数传递密钥

```go
url := fmt.Sprintf("/api/v1/tasks?api_key=%s", c.config.APIKey)
// URL 参数会被记录到访问日志、浏览器历史、Referer 头中
```

**正确**：通过 Header 传递

```go
if c.config.APIKey != "" {
    req.Header.Set("Authorization", "Bearer "+c.config.APIKey)
}
```

**执行机制**: 代码审查禁止 URL 中包含 `api_key`、`token`、`secret` 等参数名。

---

### 8. 资源管理

#### 8.1 defer 清理

**规则 8.1.1** — 所有获取的资源（HTTP 响应体、文件句柄、锁）必须通过 `defer` 确保释放

❌ **错误**：未 defer 关闭响应体

```go
resp, err := c.httpClient.Do(req)
if err != nil {
    return nil, err
}
body, _ := io.ReadAll(resp.Body)
// 忘记关闭 resp.Body
```

**正确**：defer 关闭

```go
resp, err := c.httpClient.Do(req)
if err != nil {
    return nil, err
}
defer resp.Body.Close()
body, _ := io.ReadAll(resp.Body)
```

**执行机制**: `go vet` + `errcheck` 检查 `resp.Body` 是否在 `defer` 中关闭。

---

**规则 8.1.2** — HTTP 客户端必须提供 `Close()` 方法以清理空闲连接

参考 `Client.Close()` 的实现：

```go
func (c *Client) Close() error {
    if c.httpClient != nil {
        c.httpClient.CloseIdleConnections()
    }
    return nil
}
```

❌ **错误**：不提供关闭方法

```go
type Client struct {
    httpClient *http.Client
    // 无 Close() 方法，空闲连接不会被释放
}
```

**正确**：实现 `Close()` 方法

```go
func (c *Client) Close() error {
    if c.httpClient != nil {
        c.httpClient.CloseIdleConnections()
    }
    return nil
}
```

**执行机制**: 代码审查检查所有持有 `http.Client` 的类型是否实现 `Close()`。

---

#### 8.2 Goroutine 泄漏预防

**规则 8.2.1** — 使用 `sync.WaitGroup` 等待 goroutine 完成，确保无泄漏

参考 `TaskManager.WaitForAny()` 和 `WaitForAll()` 的实现。

❌ **错误**：启动 goroutine 后不等待

```go
for _, id := range taskIDs {
    go func(taskID string) {
        result, _ := tm.Wait(ctx, taskID, timeout)
        resultCh <- result
    }(id)
}
// 函数返回后 goroutine 仍在运行
```

**正确**：使用 WaitGroup + done channel

```go
var wg sync.WaitGroup
for _, id := range taskIDs {
    wg.Add(1)
    go func(taskID string) {
        defer wg.Done()
        result, err := tm.Wait(ctx, taskID, timeout)
        // ...
    }(id)
}

done := make(chan struct{})
go func() {
    wg.Wait()
    close(done)
}()

select {
case result := <-resultCh:
    return result, nil
case <-done:
    return nil, agentrt.NewError(agentrt.CodeTaskFailed, "所有任务已完成但无结果", nil)
case <-ctx.Done():
    return nil, ctx.Err()
}
```

**执行机制**: 集成测试中使用 `goleak` 检测 goroutine 泄漏。

---

#### 8.3 文件描述符管理

**规则 8.3.1** — 限制 HTTP 连接池大小，防止文件描述符耗尽

SDK 中通过 `Config.MaxConnections` 和 `Transport.MaxIdleConns` 控制：

```go
Transport: &http.Transport{
    MaxIdleConns:    config.MaxConnections, // 默认 10
    IdleConnTimeout: config.IdleConnTimeout, // 默认 90s
}
```

❌ **错误**：无连接限制

```go
client := &http.Client{
    Transport: &http.Transport{
        // 未设置 MaxIdleConns，可能耗尽文件描述符
    },
}
```

**正确**：设置连接上限

```go
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        10,
        IdleConnTimeout:     90 * time.Second,
        DisableCompression:  false,
    },
}
```

**执行机制**: 代码审查检查 `http.Transport` 是否设置 `MaxIdleConns`。

---

**规则 8.3.2** — 遥测数据必须设置上限，防止内存无限增长

SDK 中 `Meter` 和 `Tracer` 均实现了数据点淘汰机制：

```go
// Meter: 超出上限时淘汰最早的数据点
if len(m.metrics[name]) > m.maxDataPoints {
    m.metrics[name] = m.metrics[name][len(m.metrics[name])-m.maxDataPoints:]
}

// Tracer: 超出上限时淘汰最早的区间
if len(t.spans) > t.maxSpans {
    t.spans = t.spans[len(t.spans)-t.maxSpans:]
}
```

❌ **错误**：无上限的遥测数据

```go
type Meter struct {
    metrics map[string][]MetricPoint // 无上限，内存可能无限增长
}
```

**正确**：设置上限并淘汰旧数据

```go
type Meter struct {
    mu            sync.RWMutex
    metrics       map[string][]MetricPoint
    maxDataPoints int // 默认 1000
}

func (m *Meter) Record(name string, value float64, tags map[string]string) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.metrics[name] = append(m.metrics[name], point)
    if len(m.metrics[name]) > m.maxDataPoints {
        m.metrics[name] = m.metrics[name][len(m.metrics[name])-m.maxDataPoints:]
    }
}
```

**执行机制**: 代码审查检查所有缓存/收集器是否设置容量上限。

---

### 9. 审计与可观测性

#### 9.1 请求追踪

**规则 9.1.1** — 所有 HTTP 请求必须携带 `X-Request-ID` 头用于追踪

SDK 中 `Client.request()` 的实现：

```go
req.Header.Set("X-Request-ID", generateRequestID())
```

其中 `generateRequestID()` 使用 `crypto/rand` 生成唯一 ID：

```go
func generateRequestID() string {
    timestamp := time.Now().UnixNano()
    randomNum, _ := rand.Int(rand.Reader, big.NewInt(1000000))
    return fmt.Sprintf("req-%d-%06d", timestamp, randomNum.Int64())
}
```

❌ **错误**：无请求追踪标识

```go
req, _ := http.NewRequestWithContext(ctx, method, url, body)
// 缺少 X-Request-ID
```

**正确**：携带追踪标识

```go
req.Header.Set("X-Request-ID", generateRequestID())
```

**执行机制**: 代码审查检查 HTTP 请求是否设置 `X-Request-ID`。

---

#### 9.2 遥测数据安全

**规则 9.2.1** — 遥测数据不得包含敏感信息（API Key、用户凭证、个人数据）

❌ **错误**：遥测数据包含敏感信息

```go
m.Meter.Record("api.request", 1, map[string]string{
    "api_key": c.config.APIKey, // 泄露密钥
    "user_id": userID,          // 泄露用户标识
})
```

**正确**：仅记录非敏感元数据

```go
m.Meter.Record("api.request", 1, map[string]string{
    "method":    "POST",
    "endpoint":  "/api/v1/tasks",
    "status":    "success",
})
```

**执行机制**: 代码审查检查遥测标签是否包含 `api_key`、`password`、`token` 等敏感字段名。

---

#### 9.3 日志安全

**规则 9.3.1** — 日志不得包含敏感信息，Debug 模式日志必须与生产日志隔离

SDK 使用 `Config.Debug` 和 `Config.LogLevel` 控制日志详细程度。

❌ **错误**：生产日志包含调试信息

```go
log.Printf("Request: endpoint=%s, apiKey=%s, body=%v", endpoint, apiKey, body)
```

**正确**：分级日志，敏感信息仅在 Debug 模式下输出

```go
if c.config.Debug {
    log.Printf("[DEBUG] Request: endpoint=%s, method=%s", endpoint, method)
}
log.Printf("[INFO] Request completed: method=%s, status=%d", method, statusCode)
```

**执行机制**: 代码审查检查日志格式字符串是否包含 `APIKey`、`Password` 等字段。

---

#### 9.4 审计日志完整性

**规则 9.4.1** — 审计日志必须包含操作者、操作类型、目标资源、时间戳和结果

❌ **错误**：审计信息不完整

```go
log.Printf("Task cancelled")
```

**正确**：完整的审计记录

```go
agentrt.GetLogger().Printf("[AUDIT] operation=task.cancel, task_id=%s, result=%v, timestamp=%s",
    taskID, err, time.Now().Format(time.RFC3339))
```

**执行机制**: 代码审查检查变更类操作是否包含完整审计字段。

---

### 10. Airymax 模块安全编码示例

#### 10.1 安全的 HTTP 客户端

综合运用本规范中所有安全规则的完整 HTTP 客户端示例：

```go
package secureclient

import (
    "bytes"
    "context"
    "crypto/rand"
    "encoding/json"
    "fmt"
    "io"
    "math"
    "math/big"
    "net/http"
    "time"

    "github.com/spharx/agentrt/toolkit/go/agentrt"
    "github.com/spharx/agentrt/toolkit/go/agentrt/types"
    "github.com/spharx/agentrt/toolkit/go/agentrt/utils"
)

const SecureMaxResponseBodySize = 10 * 1024 * 1024

type SecureClient struct {
    config     *agentrt.Config
    httpClient *http.Client
    telemetry  *TelemetryWrapper
}

func NewSecureClient(opts ...agentrt.ConfigOption) (*SecureClient, error) {
    config := agentrt.NewConfig(opts...)
    if err := config.Validate(); err != nil { // D3: 输入验证
        return nil, err
    }

    return &SecureClient{
        config: config,
        httpClient: &http.Client{
            Timeout: config.Timeout, // 7.1.1: 设置超时
            Transport: &http.Transport{
                MaxIdleConns:    config.MaxConnections, // 8.3.1: 连接池限制
                IdleConnTimeout: config.IdleConnTimeout,
            },
        },
        telemetry: NewTelemetryWrapper(),
    }, nil
}

func (c *SecureClient) DoRequest(ctx context.Context, method, path string,
    body interface{}) (*types.APIResponse, error) {

    // D4: 审计追踪
    span := c.telemetry.StartSpan(fmt.Sprintf("%s %s", method, path))
    defer func() {
        c.telemetry.FinishSpan(span)
        c.telemetry.Record(fmt.Sprintf("request.%s", method), 1, map[string]string{
            "path": path,
        })
    }()

    // 序列化请求体
    var reqBody io.Reader
    if body != nil {
        jsonData, err := json.Marshal(body)
        if err != nil {
            return nil, agentrt.WrapError(agentrt.CodeParseError, "序列化请求体失败", err) // 5.3.2: 保留原始错误
        }
        reqBody = bytes.NewReader(jsonData)
    }

    // 7.1.2: 使用 NewRequestWithContext 传播 context
    fullURL := c.config.Endpoint + path
    req, err := http.NewRequestWithContext(ctx, method, fullURL, reqBody)
    if err != nil {
        return nil, agentrt.WrapError(agentrt.CodeNetworkError, "创建请求失败", err)
    }

    // 7.5.1: 通过 Header 传递认证
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("User-Agent", c.config.UserAgent)
    req.Header.Set("X-Request-ID", generateSecureRequestID()) // 9.1.1: 请求追踪
    if c.config.APIKey != "" {
        req.Header.Set("Authorization", "Bearer "+c.config.APIKey)
    }

    // 7.4: 重试逻辑
    var lastErr error
    for attempt := 0; attempt <= c.config.MaxRetries; attempt++ {
        if attempt > 0 {
            delay := calculateSecureBackoff(c.config.RetryDelay, attempt) // 7.4.1: 指数退避+抖动
            select {
            case <-ctx.Done(): // 7.3.1: 监听取消信号
                return nil, agentrt.WrapError(agentrt.CodeTimeout, "请求在重试等待中被取消", ctx.Err())
            case <-time.After(delay):
            }

            // 7.4.3: 重置请求体
            if seeker, ok := req.Body.(io.Seeker); ok && req.Body != nil {
                if _, seekErr := seeker.Seek(0, io.SeekStart); seekErr != nil {
                    return nil, agentrt.WrapError(agentrt.CodeNetworkError, "重置请求体失败", seekErr)
                }
            }
        }

        resp, err := c.httpClient.Do(req)
        if err != nil {
            lastErr = agentrt.WrapError(agentrt.CodeNetworkError, "请求执行失败", err)
            if ctx.Err() != nil {
                return nil, agentrt.WrapError(agentrt.CodeTimeout, "请求被取消", ctx.Err())
            }
            continue
        }

        // 8.1.1: defer 关闭响应体
        // 7.2.1: 限制响应体大小
        respBody, readErr := io.ReadAll(io.LimitReader(resp.Body, SecureMaxResponseBodySize))
        resp.Body.Close()

        if readErr != nil {
            lastErr = agentrt.WrapError(agentrt.CodeParseError, "读取响应失败", readErr)
            continue
        }

        if resp.StatusCode >= 400 {
            // 5.4.1: HTTP 状态码映射
            lastErr = agentrt.HTTPStatusToError(resp.StatusCode, string(respBody))
            // 7.4.2: 仅重试可重试错误
            if !shouldSecureRetry(resp.StatusCode) {
                return nil, lastErr
            }
            continue
        }

        var apiResp types.APIResponse
        // 2.2.2: 验证反序列化结果
        if err := json.Unmarshal(respBody, &apiResp); err != nil {
            return nil, agentrt.WrapError(agentrt.CodeParseError, "解析响应失败", err)
        }

        return &apiResp, nil
    }

    span.Status = types.SpanStatusError
    return nil, lastErr
}

// 3.1.1: 使用 crypto/rand
func generateSecureRequestID() string {
    timestamp := time.Now().UnixNano()
    randomNum, err := rand.Int(rand.Reader, big.NewInt(1000000)) // 3.1.2: 检查错误
    if err != nil {
        // 降级方案：仅使用时间戳（比 math/rand 更安全的选择是直接失败）
        return fmt.Sprintf("req-%d-fallback", timestamp)
    }
    return fmt.Sprintf("req-%d-%06d", timestamp, randomNum.Int64())
}

// 7.4.1: 指数退避 + crypto/rand 抖动
func calculateSecureBackoff(baseDelay time.Duration, attempt int) time.Duration {
    backoff := float64(baseDelay) * math.Pow(2, float64(attempt-1))
    jitterBig, _ := rand.Int(rand.Reader, big.NewInt(int64(backoff)))
    jitter := time.Duration(jitterBig.Int64())
    return time.Duration(backoff) + jitter
}

// 7.4.2: 仅重试 5xx 和 429
func shouldSecureRetry(statusCode int) bool {
    return statusCode >= 500 || statusCode == http.StatusTooManyRequests
}

func (c *SecureClient) Close() error { // 8.1.2: 提供关闭方法
    if c.httpClient != nil {
        c.httpClient.CloseIdleConnections()
    }
    return nil
}
```

#### 10.2 安全的插件管理

```go
package secureplugin

import (
    "context"
    "fmt"
    "sync"
    "time"

    "github.com/spharx/agentrt/toolkit/go/agentrt"
    "github.com/spharx/agentrt/toolkit/go/agentrt/plugin"
    "github.com/spharx/agentrt/toolkit/go/agentrt/syscall"
)

// SecurePluginManager 安全的插件管理器封装
type SecurePluginManager struct {
    inner   *plugin.PluginManager
    binding syscall.SyscallBinding // D2: 权限仲裁
}

func NewSecurePluginManager(binding syscall.SyscallBinding) *SecurePluginManager {
    return &SecurePluginManager{
        inner:   plugin.NewPluginManager(), // sandboxEnabled 默认 true (D1)
        binding: binding,
    }
}

// LoadPluginSafely 安全加载插件，强制验证资源限制
func (m *SecurePluginManager) LoadPluginSafely(pluginID string,
    manifest plugin.PluginManifest) (*plugin.PluginInfo, error) {

    // D1: 验证沙箱资源限制
    if manifest.MaxMemoryMB <= 0 {
        return nil, agentrt.NewError(agentrt.CodeInvalidParameter,
            "插件必须设置 MaxMemoryMB", nil)
    }
    if manifest.MaxCPUPercent <= 0 || manifest.MaxCPUPercent > 100 {
        return nil, agentrt.NewError(agentrt.CodeInvalidParameter,
            "插件 MaxCPUPercent 必须在 (0, 100] 范围内", nil)
    }
    if manifest.TimeoutSeconds <= 0 {
        return nil, agentrt.NewError(agentrt.CodeInvalidParameter,
            "插件必须设置 TimeoutSeconds", nil)
    }

    // D3: 净化插件 ID
    pluginID = agentrt.GetLogger().Prefix() // 示例：实际应使用 SanitizeString

    info, err := m.inner.LoadPlugin(pluginID, manifest)
    if err != nil {
        return nil, agentrt.WrapError(agentrt.CodeInternal, "插件加载失败", err)
    }

    return info, nil
}

// ExecuteInSandbox 在沙箱中执行插件操作
func (m *SecurePluginManager) ExecuteInSandbox(ctx context.Context,
    pluginID string, operation string, params map[string]any) (*syscall.SyscallResponse, error) {

    // D2: 通过 SyscallBinding 执行，受权限仲裁约束
    resp, err := m.binding.Invoke(ctx, &syscall.SyscallRequest{
        Namespace: syscall.NamespaceTask,
        Operation: operation,
        Params:    params,
    })
    if err != nil {
        return nil, agentrt.WrapError(agentrt.CodePermissionDenied,
            fmt.Sprintf("插件 %s 操作 %s 被拒绝", pluginID, operation), err)
    }

    return resp, nil
}
```

#### 10.3 安全的系统调用

```go
package securesyscall

import (
    "context"
    "fmt"
    "sync"

    "github.com/spharx/agentrt/toolkit/go/agentrt"
    "github.com/spharx/agentrt/toolkit/go/agentrt/syscall"
    "github.com/spharx/agentrt/toolkit/go/agentrt/types"
    "github.com/spharx/agentrt/toolkit/go/agentrt/utils"
)

// AuditedSyscallBinding 带审计的系统调用绑定
type AuditedSyscallBinding struct {
    inner     syscall.SyscallBinding
    mu        sync.RWMutex
    auditLog  []AuditEntry
    maxAudit  int
}

type AuditEntry struct {
    Namespace syscall.SyscallNamespace
    Operation string
    Timestamp time.Time
    Success   bool
    Error     string
}

func NewAuditedSyscallBinding(inner syscall.SyscallBinding) *AuditedSyscallBinding {
    return &AuditedSyscallBinding{
        inner:    inner,
        auditLog: make([]AuditEntry, 0),
        maxAudit: 1000, // 8.3.2: 设置上限
    }
}

func (b *AuditedSyscallBinding) Invoke(ctx context.Context,
    request *syscall.SyscallRequest) (*syscall.SyscallResponse, error) {

    // D3: 验证请求参数
    if request.Namespace == "" {
        return nil, agentrt.NewError(agentrt.CodeMissingParameter, "命名空间不能为空", nil)
    }
    if err := utils.ValidateRequiredString(request.Operation, "操作名"); err != nil {
        return nil, agentrt.NewError(agentrt.CodeMissingParameter, err.Error(), nil)
    }

    // D4: 记录审计日志
    entry := AuditEntry{
        Namespace: request.Namespace,
        Operation: request.Operation,
        Timestamp: time.Now(),
    }

    resp, err := b.inner.Invoke(ctx, request)
    if err != nil {
        entry.Success = false
        entry.Error = err.Error() // 内部审计可包含详情，不对外暴露
    } else {
        entry.Success = resp.Success
        if !resp.Success {
            entry.Error = resp.Error
        }
    }

    b.mu.Lock()
    b.auditLog = append(b.auditLog, entry)
    if len(b.auditLog) > b.maxAudit { // 8.3.2: 淘汰旧记录
        b.auditLog = b.auditLog[len(b.auditLog)-b.maxAudit:]
    }
    b.mu.Unlock()

    return resp, err
}
```

---

### 附录 A: 规则速查表

| 编号 | 规则 | Cupolas 层 | 严重性 | 执行机制 |
|------|------|-----------|--------|----------|
| 1.3.1 | 插件资源限制声明 | D1 | 高 | 静态分析 |
| 1.3.2 | 插件通过 SyscallBinding 操作 | D2 | 高 | 代码审查 |
| 1.3.3 | 外部输入验证/净化 | D3 | 高 | Linter |
| 1.3.4 | 关键操作审计记录 | D4 | 中 | 代码审查 |
| 2.1.1 | 公共 API 参数校验 | D3 | 高 | 代码审查+测试 |
| 2.1.2 | 配置对象 Validate() | D3 | 高 | 静态分析 |
| 2.2.1 | 类型安全提取 JSON 数据 | D3 | 高 | 代码审查 |
| 2.2.2 | 反序列化数据完整性验证 | D3 | 高 | 代码审查+测试 |
| 2.3.1 | URL 参数通过 url.Values 编码 | D3 | 高 | 代码审查 |
| 2.3.2 | 路径参数净化 | D3 | 高 | 代码审查 |
| 3.1.1 | 使用 crypto/rand | — | 严重 | Linter |
| 3.1.2 | 检查 crypto/rand 错误 | — | 高 | errcheck |
| 3.2.1 | TLS 1.2+ | — | 严重 | 静态分析 |
| 3.2.2 | 禁止 InsecureSkipVerify | — | 严重 | CI 扫描 |
| 3.3.1 | 禁止硬编码密钥 | — | 严重 | pre-commit hook |
| 3.3.2 | 密钥不出现在日志/错误 | — | 高 | 代码审查 |
| 4.1.1 | RWMutex 保护共享状态 | — | 高 | go test -race |
| 4.1.2 | 禁止持锁执行阻塞操作 | — | 高 | 代码审查 |
| 4.2.1 | goroutine 生命周期可控 | — | 高 | goleak |
| 4.2.2 | goroutine 中 recover panic | — | 高 | 代码审查 |
| 4.3.1 | channel 关闭安全 | — | 高 | go vet |
| 4.4.1 | Context 作为第一个参数 | — | 高 | go vet |
| 4.4.2 | 禁止结构体存储 Context | — | 高 | go vet |
| 5.1.1 | 错误信息脱敏 | — | 高 | 代码审查 |
| 5.2.1 | 使用 AgentRTError 体系 | — | 高 | Linter |
| 5.3.1 | 哨兵错误 + errors.Is | — | 中 | 代码审查 |
| 5.3.2 | WrapError 保留原始错误 | — | 高 | 代码审查 |
| 5.4.1 | HTTP 状态码映射 | — | 中 | 代码审查 |
| 6.1.1 | 零外部依赖 | — | 严重 | go mod verify |
| 6.2.1 | go.sum 审计 | — | 高 | 合并请求审查 |
| 6.3.1 | Go 版本 >= 1.22 | — | 中 | CI 检查 |
| 6.3.2 | 模块路径一致 | — | 中 | CI 检查 |
| 7.1.1 | HTTP 客户端设置超时 | — | 高 | 代码审查 |
| 7.1.2 | NewRequestWithContext | — | 高 | go vet |
| 7.2.1 | LimitReader 限制响应体 | — | 严重 | 代码审查 |
| 7.3.1 | 重试等待监听 context | — | 高 | 代码审查 |
| 7.4.1 | 指数退避+随机抖动 | — | 中 | 代码审查 |
| 7.4.2 | 仅重试可重试错误 | — | 高 | 代码审查 |
| 7.4.3 | 重试重置请求体 | — | 中 | 代码审查 |
| 7.5.1 | 密钥通过 Header 传递 | — | 严重 | 代码审查 |
| 8.1.1 | defer 关闭资源 | — | 高 | go vet |
| 8.1.2 | 提供 Close() 方法 | — | 中 | 代码审查 |
| 8.2.1 | WaitGroup 等待 goroutine | — | 高 | goleak |
| 8.3.1 | 限制连接池大小 | — | 中 | 代码审查 |
| 8.3.2 | 遥测数据设置上限 | — | 中 | 代码审查 |
| 9.1.1 | X-Request-ID 追踪 | D4 | 中 | 代码审查 |
| 9.2.1 | 遥测不含敏感信息 | — | 高 | 代码审查 |
| 9.3.1 | 日志分级+脱敏 | — | 高 | 代码审查 |
| 9.4.1 | 审计日志完整性 | D4 | 中 | 代码审查 |

---

### 附录 B: 已知技术债务

以下为当前 SDK 代码中与本规范不符的已知问题，需在后续版本中修复：

| 位置 | 问题 | 对应规则 | 优先级 |
|------|------|----------|--------|
| `utils.GenerateID()` | `rand.Read(b)` 错误被忽略 | 3.1.2 | 高 |
| `protocol.go` 多处 | `io.ReadAll(resp.Body)` 未使用 `LimitReader` | 7.2.1 | 严重 |
| `protocol.go` `DetectProtocol()` | 响应体未限制大小 | 7.2.1 | 严重 |
| `client.calculateBackoff()` | `rand.Int` 错误被忽略 | 3.1.2 | 中 |
| `plugin.PluginManager` | `WithSandboxDisabled()` 选项可关闭沙箱 | 1.3.1 | 中 |

---

*本文档由 Airymax 安全团队维护，最终解释权归 SPHARX Ltd. 所有。*

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
