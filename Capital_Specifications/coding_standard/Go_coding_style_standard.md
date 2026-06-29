SPDX-FileCopyrightText: 2026 SPHARX Ltd.
SPDX-License-Identifier: Apache-2.0

# Airymax Go 编码风格规范

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/coding_standard/Go_coding_style_standard.md
---

## 目录

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
11. [Airymax 模块编码示例](#11-agentos-模块编码示例)

---

## 1. 总则与适用范围

### 1.1 目的

本规范定义 Airymax Go SDK 的编码风格标准，确保代码库在命名、结构、错误处理、并发模式和测试策略上保持高度一致性。所有规则均源自项目实际代码中的既定模式，而非理论推导。

### 1.2 适用范围

- **必须**适用于 `agentos/toolkit/go/` 下所有 `.go` 文件
- **必须**适用于新增包、新增类型和新增公共 API
- **应当**适用于项目内其他 Go 代码（工具脚本、代码生成器等可适当放宽）

### 1.3 规范等级

| 关键词 | 含义 |
|--------|------|
| **必须** | 强制遵守，违反将导致 CI 拒绝合并 |
| **禁止** | 强制禁止，违反将导致 CI 拒绝合并 |
| **应当** | 推荐遵守，特殊场景可偏离但须注释说明 |
| **可以** | 可选建议 |

### 1.4 强制执行机制

| 工具 | 用途 |
|------|------|
| `go vet` | 静态分析，检测常见错误 |
| `staticcheck` | 高级静态分析 |
| `golangci-lint` | 聚合 linter，CI 必须通过 |
| `gofmt` / `goimports` | 格式化与导入排序 |
| CI Pipeline | 合并前全量检查 |

---

## 2. 包组织与目录结构

### 2.1 顶层包结构

Airymax Go SDK 采用扁平化子包结构，每个子包对应一个独立职责域：

```
agentos/toolkit/go/agentos/
├── agentos.go          # 包常量 (Version, Author, License) + 全局 Logger
├── config.go           # Config 结构体 + Functional Options + 环境变量构造
├── errors.go           # AgentOSError + ErrorCode + 哨兵错误 + HTTP 映射
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

### 2.2 包划分原则

**必须**按职责单一原则划分包。一个包只做一件事。

❌ **禁止**将不相关的功能混入同一包：

```go
// agentos/helpers.go — 禁止：混合了 HTTP、JSON、时间处理
package agentos

func DoHTTPRequest(...) {}
func ParseJSON(...) {}
func FormatTime(...) {}
```

✅ **必须**按职责拆分到独立子包：

```go
// client/client.go — HTTP 客户端职责
package client

// utils/helpers.go — 通用数据提取职责
package utils

// telemetry/telemetry.go — 可观测性职责
package telemetry
```

### 2.3 包内文件组织

**必须**按类型/功能将代码拆分到不同文件，而非全部塞入单文件：

| 文件 | 内容 |
|------|------|
| `errors.go` | 错误类型、错误码常量、哨兵错误、工厂函数 |
| `config.go` | 配置结构体、Option 函数、环境变量构造 |
| `protocol.go` | 协议客户端、协议类型枚举 |
| `types.go` | 枚举、领域模型、请求/响应结构 |

**禁止**将超过 500 行的代码放在单个文件中（测试文件除外）。

### 2.4 包名规范

**必须**使用全小写、无下划线、无驼峰的简短名词：

```go
package agentos    // ✅ 正确
package client     // ✅ 正确
package types      // ✅ 正确
package modules    // ✅ 正确
package task       // ✅ 正确
package telemetry  // ✅ 正确
package syscall    // ✅ 正确
```

❌ **禁止**以下包名风格：

```go
package agent_os   // 禁止：下划线
package agentOs    // 禁止：驼峰
package agentos_sdk // 禁止：下划线+过长
package util       // 禁止：过于笼统（应为 utils）
```

---

## 3. 命名规范

### 3.1 导出类型：PascalCase

**必须**使用 PascalCase 命名所有导出类型，名词短语优先：

```go
type Client struct{}           // ✅
type TaskManager struct{}      // ✅
type AgentOSError struct{}     // ✅
type Config struct{}           // ✅
type ProtocolType int          // ✅
type ResourceConverter[T any] interface {} // ✅
```

❌ **禁止**：

```go
type client struct{}           // 禁止：导出类型必须大写开头
type Task_Manager struct{}     // 禁止：下划线
type TASKMANAGER struct{}      // 禁止：全大写
```

### 3.2 导出函数：PascalCase

**必须**使用 PascalCase 命名所有导出函数：

```go
func NewClient(...) (*Client, error)       // ✅
func NewConfig(...) *Config                // ✅
func NewTaskManager(...) *TaskManager      // ✅
func IsNetworkError(err error) bool        // ✅
func IsErrorCode(err error, code ErrorCode) bool // ✅
func HTTPStatusToError(...) *AgentOSError  // ✅
```

### 3.3 未导出函数：camelCase

**必须**使用 camelCase 命名所有未导出函数：

```go
func parseTaskFromMap(data map[string]interface{}) *types.Task  // ✅
func buildQueryString(params map[string]string) string          // ✅
func calculateBackoff(base time.Duration, attempt int) time.Duration // ✅
func shouldRetry(statusCode int) bool                           // ✅
func generateRequestID() string                                // ✅
```

❌ **禁止**：

```go
func ParseTaskFromMap(...)   // 禁止：包内辅助函数不应导出
func parse_task_from_map(...) // 禁止：下划线
```

### 3.4 构造函数：New 前缀

**必须**使用 `New` 前缀命名所有构造函数：

```go
func NewClient(opts ...ConfigOption) (*Client, error)        // ✅
func NewConfig(opts ...ConfigOption) *Config                 // ✅
func NewTaskManager(api APIClient) *TaskManager              // ✅
func NewError(code ErrorCode, msg string, cause error) *AgentOSError // ✅
func NewPluginRegistry() *PluginRegistry                     // ✅
func NewMeter() *Meter                                       // ✅
```

当存在多种构造方式时，**必须**使用描述性后缀：

```go
func NewClient(opts ...ConfigOption) (*Client, error)           // ✅ 默认构造
func NewClientWithConfig(config *Config) (*Client, error)       // ✅ 显式配置构造
func NewConfigFromEnv() (*Config, error)                        // ✅ 环境变量构造
func NewProtocolConfigFromEnv() *ProtocolConfig                 // ✅
func NewHTTPSyscallBinding(apiClient APIClient) *HTTPSyscallBinding // ✅
```

❌ **禁止**：

```go
func CreateClient(...)    // 禁止：非 New 前缀
func MakeConfig(...)      // 禁止：非 New 前缀
func ClientNew(...)       // 禁止：后置 New
```

### 3.5 Option 函数：With 前缀

**必须**使用 `With` 前缀命名所有 Functional Option 函数：

```go
func WithEndpoint(endpoint string) ConfigOption        // ✅
func WithTimeout(timeout time.Duration) ConfigOption   // ✅
func WithAPIKey(apiKey string) ConfigOption            // ✅
func WithMaxRetries(maxRetries int) ConfigOption       // ✅
func WithDebug(debug bool) ConfigOption                // ✅
func WithRequestTimeout(timeout time.Duration) RequestOption // ✅
func WithHeader(key, value string) RequestOption       // ✅
func WithQueryParam(key, value string) RequestOption   // ✅
func WithSandboxDisabled() func(*PluginManager)        // ✅
func WithPluginDirectories(dirs []string) func(*PluginManager) // ✅
```

❌ **禁止**：

```go
func SetEndpoint(endpoint string) ConfigOption   // 禁止：非 With 前缀
func EndpointOpt(endpoint string) ConfigOption   // 禁止：非 With 前缀
func With_endpoint(endpoint string) ConfigOption // 禁止：下划线
```

### 3.6 判断函数：Is 前缀

**必须**使用 `Is` 前缀命名所有返回 `bool` 的判断函数：

```go
func IsNetworkError(err error) bool      // ✅
func IsServerError(err error) bool       // ✅
func IsErrorCode(err error, code ErrorCode) bool // ✅
func (s TaskStatus) IsTerminal() bool    // ✅
func (l MemoryLayer) IsValid() bool      // ✅
```

❌ **禁止**：

```go
func CheckNetworkError(err error) bool   // 禁止：非 Is 前缀
func NetworkError(err error) bool        // 禁止：缺少动词
func Is_network_error(err error) bool    // 禁止：下划线
```

### 3.7 常量：PascalCase + 语义分组前缀

#### 3.7.1 包级常量

**必须**使用 PascalCase 命名导出常量：

```go
const (
    Version = "0.1.0"     // ✅
    Author  = "SpharxWorks" // ✅
    License = "MIT"       // ✅
)
```

#### 3.7.2 错误码常量

**必须**使用 `Code` 前缀 + PascalCase 语义名，类型为 `ErrorCode`（底层 `string`），值采用十六进制分类体系：

```go
const (
    CodeSuccess          ErrorCode = "0x0000"  // ✅ 通用
    CodeUnknown          ErrorCode = "0x0001"  // ✅
    CodeInvalidParameter ErrorCode = "0x0002"  // ✅
    CodeNotFound         ErrorCode = "0x0005"  // ✅

    CodeLoopCreateFailed ErrorCode = "0x1001"  // ✅ 核心循环
    CodeCognitionFailed  ErrorCode = "0x2001"  // ✅ 认知层
    CodeTaskFailed       ErrorCode = "0x3001"  // ✅ 执行层
    CodeMemoryNotFound   ErrorCode = "0x4001"  // ✅ 记忆层
    CodeTelemetryError   ErrorCode = "0x5001"  // ✅ 系统调用
    CodePermissionDenied ErrorCode = "0x6001"  // ✅ 安全域
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

#### 3.7.3 枚举常量

**必须**使用类型名作为前缀 + PascalCase 值名：

```go
type TaskStatus string

const (
    TaskStatusPending   TaskStatus = "pending"   // ✅
    TaskStatusRunning   TaskStatus = "running"   // ✅
    TaskStatusCompleted TaskStatus = "completed" // ✅
    TaskStatusFailed    TaskStatus = "failed"    // ✅
    TaskStatusCancelled TaskStatus = "cancelled" // ✅
)

type ProtocolType int

const (
    ProtocolJSONRPC ProtocolType = iota // ✅
    ProtocolMCP                          // ✅
    ProtocolA2A                          // ✅
    ProtocolOpenAI                       // ✅
    ProtocolAutoDetect                   // ✅
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

### 3.8 哨兵错误变量：Err 前缀

**必须**使用 `Err` 前缀命名所有哨兵错误变量，且**必须**使用 `var` 声明：

```go
var (
    ErrNotFound         = NewError(CodeNotFound, "资源未找到", nil)         // ✅
    ErrTimeout          = NewError(CodeTimeout, "操作超时", nil)            // ✅
    ErrInvalidConfig    = NewError(CodeInvalidConfig, "配置无效", nil)      // ✅
    ErrNetworkError     = NewError(CodeNetworkError, "网络错误", nil)       // ✅
    ErrTaskFailed       = NewError(CodeTaskFailed, "任务执行失败", nil)     // ✅
    ErrMemoryNotFound   = NewError(CodeMemoryNotFound, "记忆未找到", nil)   // ✅
    ErrPermissionDenied = NewError(CodePermissionDenied, "权限不足", nil)   // ✅
)
```

❌ **禁止**：

```go
var NotFoundErr = NewError(...)   // 禁止：后置 Err
var ERR_NOT_FOUND = NewError(...) // 禁止：全大写
var notFoundErr = NewError(...)   // 禁止：小写（哨兵错误必须导出）
```

### 3.9 接口命名：语义名

**必须**使用语义名（而非 `I` 前缀或 `Interface` 后缀）命名接口：

```go
type APIClient interface {}            // ✅ 语义名
type BasePlugin interface {}           // ✅ 语义名
type SyscallBinding interface {}       // ✅ 语义名
type ResourceConverter[T any] interface {} // ✅ 泛型接口
type PluginFactory func() BasePlugin   // ✅ 函数类型别名
```

❌ **禁止**：

```go
type IAPIClient interface {}       // 禁止：I 前缀
type APIClientInterface interface {} // 禁止：Interface 后缀
type ClientAPI interface {}        // 禁止：词序颠倒（应为 APIClient）
```

---

## 4. 类型设计

### 4.1 结构体

#### 4.1.1 字段对齐与分组

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

#### 4.1.2 JSON 标签

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

#### 4.1.3 不可变性方法

当方法返回内部状态的副本时，**必须**返回克隆值而非指针：

```go
func (c *Client) GetConfig() *agentos.Config {
    return c.config.Clone() // ✅ 返回克隆，防止外部修改
}
```

❌ **禁止**直接返回内部可变状态的引用：

```go
func (c *Client) GetConfig() *agentos.Config {
    return c.config // 禁止：外部可修改内部状态
}
```

#### 4.1.4 String() 方法

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

### 4.2 枚举类型

**必须**使用 `type` 定义枚举底层类型，**禁止**使用裸 `int`/`string`：

```go
// ✅ 正确：基于 string 的枚举
type TaskStatus string

const (
    TaskStatusPending   TaskStatus = "pending"
    TaskStatusRunning   TaskStatus = "running"
    TaskStatusCompleted TaskStatus = "completed"
)

// ✅ 正确：基于 int 的枚举（iota）
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

### 4.3 接口

#### 4.3.1 接口定义

**必须**在消费侧定义接口，接口应当小而精：

```go
type APIClient interface {
    Get(ctx context.Context, path string, opts ...RequestOption) (*APIResponse, error)
    Post(ctx context.Context, path string, body interface{}, opts ...RequestOption) (*APIResponse, error)
    Put(ctx context.Context, path string, body interface{}, opts ...RequestOption) (*APIResponse, error)
    Delete(ctx context.Context, path string, opts ...RequestOption) (*APIResponse, error)
}
```

#### 4.3.2 接口满足性编译期检查

**必须**在实现类型所在文件中添加编译期接口满足性检查：

```go
var _ APIClient = (*Client)(nil) // ✅ 编译期验证 Client 实现 APIClient
```

❌ **禁止**省略此检查：

```go
// 禁止：缺少编译期接口满足性检查
type Client struct { ... }
```

### 4.4 泛型

#### 4.4.1 泛型基类

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

#### 4.4.2 泛型工具函数

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

## 5. 函数设计

### 5.1 构造函数模式

**必须**遵循以下构造函数模式：

1. 默认构造函数使用 `NewXxx(opts ...Option)` 签名
2. 显式配置构造函数使用 `NewXxxWithConfig(config)` 签名
3. 环境变量构造函数使用 `NewXxxFromEnv()` 签名
4. 内部共享初始化逻辑使用未导出的 `newXxxWithConfig()` 函数

```go
// 公共默认构造
func NewClient(opts ...agentos.ConfigOption) (*Client, error) {
    config := agentos.NewConfig(opts...)
    if err := config.Validate(); err != nil {
        return nil, err
    }
    return newClientWithConfig(config)
}

// 公共显式配置构造
func NewClientWithConfig(config *agentos.Config) (*Client, error) {
    if config == nil {
        config = agentos.DefaultConfig()
    }
    if err := config.Validate(); err != nil {
        return nil, err
    }
    return newClientWithConfig(config)
}

// 内部共享初始化
func newClientWithConfig(config *agentos.Config) (*Client, error) {
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

### 5.2 Functional Options 模式

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
        if endpoint != "" { // ✅ 防御性校验：零值不覆盖默认
            c.Endpoint = endpoint
        }
    }
}

func WithTimeout(timeout time.Duration) ConfigOption {
    return func(c *Config) {
        if timeout > 0 { // ✅ 防御性校验
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
        if timeout > 0 { // ✅ 零值不覆盖
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

### 5.3 方法接收者

#### 5.3.1 接收者命名

**必须**使用类型首字母的小写作为接收者名：

```go
func (c *Client) Get(...) (...)      // ✅ Client → c
func (tm *TaskManager) Submit(...)   // ✅ TaskManager → tm
func (bm *BaseManager[T]) ExecuteGet(...) // ✅ BaseManager → bm
func (r *PluginRegistry) Register(...)    // ✅ PluginRegistry → r
func (m *Meter) Record(...)               // ✅ Meter → m
func (t *Tracer) StartSpan(...)           // ✅ Tracer → t
```

❌ **禁止**：

```go
func (this *Client) Get(...)    // 禁止：this/self
func (self *Client) Get(...)    // 禁止：this/self
func (client *Client) Get(...)  // 禁止：与类型名相同
```

#### 5.3.2 值接收者 vs 指针接收者

**必须**遵循以下规则：

- 结构体方法**必须**使用指针接收者（除非方法不需要修改且结构体极小）
- 枚举类型（基于 string/int）的 `String()` / `Is` 方法**可以**使用值接收者
- 接口方法**应当**使用指针接收者

```go
// ✅ 结构体 → 指针接收者
func (c *Client) Get(...) (*APIResponse, error)

// ✅ 枚举 → 值接收者
func (s TaskStatus) String() string { return string(s) }
func (s TaskStatus) IsTerminal() bool { ... }

// ✅ 枚举 → 值接收者（ProtocolType 底层为 int）
func (p ProtocolType) String() string { ... }
```

#### 5.3.3 接收者一致性

**禁止**同一类型的某些方法用值接收者、另一些用指针接收者。**必须**全部统一为指针接收者（枚举类型除外）。

### 5.4 Context 传播

**必须**在所有涉及 I/O、网络、阻塞等待的公共方法中接受 `context.Context` 作为第一个参数：

```go
func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error) // ✅
func (tm *TaskManager) Wait(ctx context.Context, taskID string, timeout time.Duration) (*types.TaskResult, error) // ✅
func (c *ProtocolClient) SendRequest(ctx context.Context, method string, params map[string]interface{}) ([]byte, error) // ✅
```

❌ **禁止**在 I/O 方法中省略 context：

```go
func (tm *TaskManager) Submit(description string) (*types.Task, error) // 禁止：缺少 ctx
```

### 5.5 可变参数与 Option 链

**必须**使用 `opts ...Option` 可变参数模式替代配置结构体参数：

```go
// ✅ 正确：Functional Options
func NewClient(opts ...agentos.ConfigOption) (*Client, error)

// ✅ 正确：Request Options
func (c *Client) Get(ctx context.Context, path string, opts ...types.RequestOption) (*types.APIResponse, error)
```

❌ **禁止**使用大量 bool/int 参数：

```go
func NewClient(endpoint string, timeout time.Duration, retries int, debug bool, apiKey string) // 禁止
```

---

## 6. 错误处理

### 6.1 AgentOSError 统一错误类型

**必须**使用 `AgentOSError` 作为所有 SDK 错误的统一基类型：

```go
type AgentOSError struct {
    Code    ErrorCode
    Message string
    Cause   error
}
```

**必须**实现 `error`、`Unwrap()`、`Is()` 三个方法：

```go
func (e *AgentOSError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *AgentOSError) Unwrap() error {
    return e.Cause
}

func (e *AgentOSError) Is(target error) bool {
    t, ok := target.(*AgentOSError)
    if !ok {
        return false
    }
    return e.Code == t.Code
}
```

### 6.2 错误工厂函数

**必须**使用以下三个工厂函数创建错误，**禁止**直接构造 `AgentOSError` 结构体：

```go
// 无原因错误
err := agentos.NewError(agentos.CodeNotFound, "资源未找到", nil)          // ✅

// 格式化消息
err := agentos.NewErrorf(agentos.CodeTaskTimeout, "任务 %s 超时", taskID) // ✅

// 包装已有错误
err := agentos.WrapError(agentos.CodeNetworkError, "网络异常", cause)     // ✅
```

❌ **禁止**：

```go
// 禁止：直接构造结构体
err := &agentos.AgentOSError{Code: agentos.CodeNotFound, Message: "未找到"}

// 禁止：使用标准 errors.New
err := errors.New("something failed")

// 禁止：使用 fmt.Errorf
err := fmt.Errorf("something failed: %w", cause)
```

**例外**：在非 SDK 核心包（如 `utils` 内部辅助函数）中，**可以**使用 `fmt.Errorf` 返回简单错误，但该错误最终**必须**在上层被 `WrapError` 包装。

### 6.3 哨兵错误

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
if errors.Is(err, agentos.ErrNotFound) {
    // 处理未找到
}
```

### 6.4 错误分类查询

**必须**使用分类查询函数判断错误类别，**禁止**直接比较错误码字符串：

```go
// ✅ 正确：使用分类查询
if agentos.IsNetworkError(err) {
    // 处理网络错误
}
if agentos.IsServerError(err) {
    // 处理服务端错误
}
if agentos.IsErrorCode(err, agentos.CodeNotFound) {
    // 处理特定错误码
}
```

❌ **禁止**：

```go
// 禁止：直接比较错误码字符串
if err != nil && strings.Contains(err.Error(), "0x0005") { ... }

// 禁止：类型断言后手动比较
if agentErr, ok := err.(*AgentOSError); ok && agentErr.Code == "0x0005" { ... }
```

### 6.5 HTTP 状态码映射

**必须**使用 `HTTPStatusToError()` 将 HTTP 状态码转换为 SDK 错误，**禁止**直接返回 HTTP 错误：

```go
// ✅ 正确
if resp.StatusCode >= 400 {
    lastErr = agentos.HTTPStatusToError(resp.StatusCode, string(respBody))
}
```

❌ **禁止**：

```go
// 禁止：直接返回 HTTP 状态码
return nil, fmt.Errorf("HTTP %d", resp.StatusCode)
```

### 6.6 错误消息语言

**必须**在哨兵错误和面向用户的错误消息中使用中文描述：

```go
ErrNotFound       = NewError(CodeNotFound, "资源未找到", nil)         // ✅
ErrTaskFailed     = NewError(CodeTaskFailed, "任务执行失败", nil)      // ✅
ErrSessionExpired = NewError(CodeSessionExpired, "会话已过期", nil)    // ✅
```

Mock 和测试辅助代码中的错误消息**可以**使用英文：

```go
return nil, agentos.NewError(agentos.CodeNotSupported, "Mock GET handler not configured", nil) // ✅
```

---

## 7. 并发编程

### 7.1 sync.RWMutex

**必须**在包含可变共享状态的结构体中使用 `sync.RWMutex` 保护并发访问：

```go
type PluginRegistry struct {
    mu        sync.RWMutex       // ✅ 互斥锁
    factories map[string]PluginFactory
    instances map[string]BasePlugin
    manifests map[string]PluginManifest
    states    map[string]PluginState
}
```

**必须**遵循读写分离的锁策略：

```go
// ✅ 读操作使用 RLock
func (r *PluginRegistry) Get(pluginID string) (BasePlugin, bool) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    instance, exists := r.instances[pluginID]
    return instance, exists
}

// ✅ 写操作使用 Lock
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

### 7.2 sync.Once

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

### 7.3 Context 传播与取消

**必须**在所有阻塞操作中检查 `ctx.Done()`：

```go
func (tm *TaskManager) Wait(ctx context.Context, taskID string, timeout time.Duration) (*types.TaskResult, error) {
    for {
        // ... 业务逻辑 ...

        select {
        case <-ctx.Done():
            return nil, ctx.Err()           // ✅ 响应上下文取消
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
            return nil, agentos.WrapError(agentos.CodeTimeout, "请求在重试等待中被取消", ctx.Err()) // ✅
        case <-time.After(delay):
        }
    }
    // ...
}
```

### 7.4 Channel 与 Goroutine 安全

**必须**使用缓冲 channel 防止 goroutine 泄漏：

```go
resultCh := make(chan *types.TaskResult, len(taskIDs))  // ✅ 缓冲大小 = goroutine 数
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
    go func(idx int, taskID string) { // ✅ 通过参数捕获
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

### 7.5 defer 与锁释放

**必须**在获取锁后立即使用 `defer` 释放锁：

```go
func (r *PluginRegistry) Register(...) (string, error) {
    r.mu.Lock()
    defer r.mu.Unlock() // ✅ 立即 defer
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

## 8. 测试规范

### 8.1 表驱动测试

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

### 8.2 Mock 模式

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
    return nil, agentos.NewError(agentos.CodeNotSupported, "Mock GET handler not configured", nil)
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

### 8.3 测试函数命名

**必须**使用 `Test<类型>_<方法>_<场景>` 格式命名测试函数：

```go
func TestTaskManager_Submit_Success(t *testing.T)          // ✅
func TestTaskManager_Submit_Empty(t *testing.T)            // ✅
func TestTaskManager_Submit_APIError(t *testing.T)         // ✅
func TestTaskManager_Wait_Timeout(t *testing.T)            // ✅
func TestTaskManager_Wait_ContextCancel(t *testing.T)      // ✅
func TestTaskManager_BatchSubmit_PartialFailure(t *testing.T) // ✅
```

❌ **禁止**：

```go
func TestSubmit(t *testing.T)        // 禁止：缺少类型和场景
func test_task_submit(t *testing.T)  // 禁止：下划线+小写
func TestTaskManagerSubmitSuccess(t *testing.T) // 禁止：缺少分隔符
```

### 8.4 环境变量清理

**必须**在测试中设置环境变量后使用 `defer` 清理：

```go
func TestNewConfigFromEnv(t *testing.T) {
    os.Setenv("AGENTOS_ENDPOINT", "http://env:9999")
    os.Setenv("AGENTOS_TIMEOUT", "45s")
    defer func() {
        os.Unsetenv("AGENTOS_ENDPOINT")
        os.Unsetenv("AGENTOS_TIMEOUT")
    }()

    cfg, err := NewConfigFromEnv()
    // ...
}
```

❌ **禁止**不清理环境变量：

```go
func TestNewConfigFromEnv(t *testing.T) {
    os.Setenv("AGENTOS_ENDPOINT", "http://env:9999")
    // 禁止：缺少 defer 清理，会污染其他测试
}
```

### 8.5 基准测试

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

### 8.6 测试文件位置

**必须**将测试文件放在与被测代码相同的包中（白盒测试）：

```
modules/task/manager.go
modules/task/manager_test.go    // ✅ 同包白盒测试
modules/task/benchmark_test.go  // ✅ 基准测试
```

---

## 9. 文档注释规范

### 9.1 文件头注释

**必须**在每个源文件头部添加文件头注释，包含以下信息：

```go
// Airymax Go SDK - 统一错误体系
// Version: 0.1.0
// Last updated: 2026-04-05
//
// 定义 SDK 的完整错误类型层级、错误码枚举、哨兵错误和 HTTP 状态码映射。
// 所有异常继承自 AgentOSError，支持 errors.Is/As 链式追踪。
// 对应 Python SDK: exceptions.py
```

**应当**包含以下要素：
- 模块中文名
- 版本号
- 最后更新日期
- 功能概述（中文描述）
- 对应的 Python SDK 文件（如有）

### 9.2 SPDX 头

**应当**在需要明确版权的文件头部添加 SPDX 标识：

```go
// SPDX-FileCopyrightText: 2026 SPHARX Ltd.
// SPDX-License-Identifier: Apache-2.0
```

### 9.3 公共 API 注释

**必须**为所有导出类型、函数、常量、变量添加 godoc 注释：

```go
// ErrorCode 表示 Airymax SDK 的错误码类型
// Since: 0.1.0
type ErrorCode string

// AgentOSError 是 SDK 所有错误的统一基类
// Since: 0.1.0
type AgentOSError struct {
    Code    ErrorCode
    Message string
    Cause   error
}

// NewError 创建指定错误码的新错误
// Since: 0.1.0
func NewError(code ErrorCode, message string, cause error) *AgentOSError {
    return &AgentOSError{Code: code, Message: message, Cause: cause}
}

// IsNetworkError 判断是否为网络相关错误
// Since: 0.1.0
func IsNetworkError(err error) bool { ... }
```

### 9.4 版本标注

**必须**为所有公共 API 添加 `Since: x.y.z` 版本标注：

```go
// NewClient 创建新的 API 客户端
// Since: 0.1.0
func NewClient(opts ...agentos.ConfigOption) (*Client, error)
```

### 9.5 中文描述

**必须**在注释中使用中文描述功能和语义，代码标识符保持英文：

```go
// TaskManager 管理任务完整生命周期
type TaskManager struct { ... }

// Submit 提交新的执行任务
func (tm *TaskManager) Submit(ctx context.Context, description string) (*types.Task, error)

// Wait 阻塞等待任务到达终态，支持超时控制和上下文取消
func (tm *TaskManager) Wait(ctx context.Context, taskID string, timeout time.Duration) (*types.TaskResult, error)
```

### 9.6 分隔注释

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

### 9.7 Mock 警告注释

**必须**在 Mock 类型上添加警告注释：

```go
// MockAPIClient 是 APIClient 接口的 Mock 实现，支持通过函数字段自定义行为
// 仅用于测试，不应在生产代码中引用
type MockAPIClient struct { ... }
```

---

## 10. 依赖管理

### 10.1 零外部依赖原则

**必须**保持零外部依赖。`go.mod` 中除模块声明外不得包含任何 `require` 条目：

```go
// go.mod
module github.com/spharx/agentos/toolkit/go

go 1.22
// 无 require — ✅ 正确
```

❌ **禁止**引入任何第三方依赖：

```go
// go.mod
module github.com/spharx/agentos/toolkit/go

go 1.22

require (
    github.com/some/third-party v1.2.3  // 禁止
    go.uber.org/zap v1.27.0              // 禁止
)
```

### 10.2 标准库优先

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

### 10.3 go.mod 规范

**必须**满足以下条件：

1. 模块路径**必须**为 `github.com/spharx/agentos/toolkit/go`
2. Go 版本**必须**为 `1.22` 或更高
3. **禁止**包含任何 `require`、`replace`、`exclude` 指令（除非有架构委员会批准）

### 10.4 内部包引用

**必须**使用完整模块路径引用内部包：

```go
import (
    "github.com/spharx/agentos/toolkit/go/agentos"
    "github.com/spharx/agentos/toolkit/go/agentos/client"
    "github.com/spharx/agentos/toolkit/go/agentos/types"
    "github.com/spharx/agentos/toolkit/go/agentos/utils"
)
```

---

## 11. Airymax 模块编码示例

### 11.1 Manager 模块模板

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

    "github.com/spharx/agentos/toolkit/go/agentos"
    "github.com/spharx/agentos/toolkit/go/agentos/client"
    "github.com/spharx/agentos/toolkit/go/agentos/types"
    "github.com/spharx/agentos/toolkit/go/agentos/utils"
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
        return nil, agentos.NewError(agentos.CodeMissingParameter, err.Error(), nil)
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
        return nil, agentos.NewError(agentos.CodeInvalidResponse, err.Error(), nil)
    }

    return parseXxxFromMap(data), nil
}

// Get 获取指定 XXX 资源的详细信息
func (xm *XxxManager) Get(ctx context.Context, id string) (*types.Xxx, error) {
    if err := utils.ValidateRequiredString(id, "资源ID"); err != nil {
        return nil, agentos.NewError(agentos.CodeMissingParameter, err.Error(), nil)
    }

    resp, err := xm.api.Get(ctx, fmt.Sprintf("/api/v1/xxxs/%s", id))
    if err != nil {
        return nil, err
    }

    data, err := utils.ValidateAndExtractData(resp, "XXX 详情响应格式异常")
    if err != nil {
        return nil, agentos.NewError(agentos.CodeInvalidResponse, err.Error(), nil)
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
        return agentos.NewError(agentos.CodeMissingParameter, err.Error(), nil)
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
        return nil, agentos.NewError(agentos.CodeInvalidResponse, err.Error(), nil)
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

### 11.2 Client 模块模板

以下模板展示如何扩展 API 客户端：

```go
package client

import (
    "context"

    "github.com/spharx/agentos/toolkit/go/agentos"
    "github.com/spharx/agentos/toolkit/go/agentos/types"
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

### 11.3 Plugin 模块模板

以下模板展示如何实现自定义插件：

```go
package myplugin

import (
    "github.com/spharx/agentos/toolkit/go/agentos/plugin"
)

// MyPlugin 自定义插件实现
type MyPlugin struct {
    plugin.BasePluginImpl // ✅ 嵌入默认实现
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
        Version:      "1.0.0",
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

### 11.4 Syscall 便捷类型模板

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
        Namespace: NamespaceXxx,     // ✅ 使用 SyscallNamespace 常量
        Operation: "some_operation", // ✅ snake_case 操作名
        Params:    map[string]any{"param": param},
    })
}
```

---

## 附录 A：检查清单

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
| 13 | 错误是否使用 AgentOSError 体系 | §6.1 |
| 14 | 是否使用工厂函数创建错误 | §6.2 |
| 15 | 并发结构体是否有 sync.RWMutex | §7.1 |
| 16 | 阻塞操作是否检查 ctx.Done() | §7.3 |
| 17 | 测试是否使用表驱动 | §8.1 |
| 18 | 环境变量测试是否 defer 清理 | §8.4 |
| 19 | 公共 API 是否有 godoc + Since 标注 | §9.3-9.4 |
| 20 | 是否引入了外部依赖 | §10.1 |

## 附录 B：Linter 配置建议

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
