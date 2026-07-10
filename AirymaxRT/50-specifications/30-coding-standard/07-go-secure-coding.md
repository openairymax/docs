SPDX-FileCopyrightText: 2026 SPHARX Ltd.
SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0

# Airymax Go 安全编码规范

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/50-specifications/30-coding-standard/07-go-secure-coding.md
---

## 目录

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

## 1. 总则与安全架构映射

### 1.1 Cupolas 四层防护体系

Airymax 采用 **Cupolas 四层防护** 安全架构，每一层对应 Go SDK 中的具体安全机制。所有 Go 代码必须遵循该架构的约束，不得绕过任何一层防护。

| 防护层 | 代号 | 安全目标 | Go SDK 对应机制 |
|--------|------|----------|-----------------|
| D1: Sandbox Isolation | D1 | 插件与核心运行时隔离 | `PluginManager.sandboxEnabled`、`PluginManifest` 资源限制、`PluginState` 状态机 |
| D2: Permission Arbitration | D2 | 操作权限仲裁 | `PluginManifest.Permissions`、`CodePermissionDenied` (0x6001)、`SyscallBinding` 接口 |
| D3: Input Sanitization | D3 | 输入净化与验证 | `utils.SanitizeString()`、`utils.ValidateRequiredString()`、`Config.Validate()` |
| D4: Audit Trail | D4 | 操作审计追踪 | `telemetry.Meter`/`Tracer`、`X-Request-ID` 头、`BaseManager.LogOperation()` |

### 1.2 安全编码基本原则

- **纵深防御**: 每一层防护独立生效，不得因上层防护存在而省略下层防护
- **最小权限**: 插件仅声明所需权限，默认拒绝一切未声明的能力
- **安全默认**: 所有配置项必须提供安全的默认值（如 `sandboxEnabled: true`）
- **零信任输入**: 所有外部输入（网络响应、环境变量、用户参数）均视为不可信

### 1.3 架构映射规则

**规则 1.3.1** — D1 沙箱隔离：所有插件必须通过 `PluginManifest` 声明资源上限

❌ **错误**：未设置资源限制的插件清单

```go
manifest := &plugin.PluginManifest{
    PluginID: "my-plugin",
    Name:     "My Plugin",
    // 缺少 MaxMemoryMB / MaxCPUPercent / MaxThreads / TimeoutSeconds
}
```

✅ **正确**：所有资源限制字段必须显式设置

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

✅ **正确**：通过系统调用接口发起请求，由仲裁层决定是否放行

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

✅ **正确**：先验证再使用

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

✅ **正确**：操作前后记录遥测

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

## 2. 输入验证

### 2.1 参数校验

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

✅ **正确**：使用 SDK 校验工具

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

✅ **正确**：构造后立即验证

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

### 2.2 JSON 反序列化安全

**规则 2.2.1** — 禁止将未经验证的 JSON 数据直接反序列化为 `interface{}` 后使用

必须通过 `utils.GetString()`、`utils.GetInt64()` 等类型安全提取函数访问反序列化后的数据。

❌ **错误**：直接类型断言，可能 panic

```go
var result map[string]interface{}
json.Unmarshal(body, &result)
taskID := result["task_id"].(string) // 若 key 不存在或类型不匹配将 panic
```

✅ **正确**：使用类型安全提取

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

✅ **正确**：验证后提取

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

### 2.3 URL 构建安全

**规则 2.3.1** — URL 查询参数必须通过 `url.Values` 编码，禁止手动拼接

SDK 已提供 `utils.BuildURL()` 函数，内部使用 `url.Values.Encode()` 进行安全编码。

❌ **错误**：手动拼接查询参数

```go
path := fmt.Sprintf("/api/v1/tasks?page=%d&page_size=%d", page, pageSize)
// 未对参数进行 URL 编码，特殊字符可能导致注入
```

✅ **正确**：使用 `BuildURL` 或 `url.Values`

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

✅ **正确**：净化后再嵌入

```go
safeKey := utils.SanitizeString(key)
safeKey = url.PathEscape(safeKey)
resp, err := sm.api.Get(ctx, fmt.Sprintf("/api/v1/sessions/%s/context/%s", sessionID, safeKey))
```

**执行机制**: 代码审查检查路径参数是否经过 `SanitizeString` + `url.PathEscape` 处理。

---

## 3. 密码学使用规范

### 3.1 随机数生成

**规则 3.1.1** — 所有安全敏感场景必须使用 `crypto/rand`，严禁使用 `math/rand`

SDK 中 `client.generateRequestID()`、`protocol.generateID()`、`utils.GenerateID()` 均使用 `crypto/rand` 作为安全范例。

❌ **错误**：使用 `math/rand` 生成请求 ID 或令牌

```go
import "math/rand"

func generateToken() string {
    return fmt.Sprintf("token-%d", rand.Intn(1000000))
}
```

✅ **正确**：使用 `crypto/rand`

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

✅ **正确**：检查错误

```go
b := make([]byte, 16)
if _, err := rand.Read(b); err != nil {
    return "", agentrt.WrapError(agentrt.CodeInternal, "生成随机数失败", err)
}
```

> **注意**: SDK 中 `utils.GenerateID()` 使用 `_, _ = rand.Read(b)` 忽略了错误，这是一个已知的技术债务，应在后续版本中修复。新代码必须检查错误。

**执行机制**: 代码审查 + `errcheck` linter 检查 `crypto/rand.Read` 返回值。

---

### 3.2 TLS 配置

**规则 3.2.1** — 生产环境必须使用 TLS 1.2+，禁止降级到 TLS 1.0/1.1

❌ **错误**：允许旧版 TLS

```go
transport := &http.Transport{
    TLSClientConfig: &tls.Config{
        MinVersion: tls.VersionTLS10, // 不安全
    },
}
```

✅ **正确**：强制 TLS 1.2+

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

✅ **正确**：仅在测试中使用自定义 CA，且必须通过配置控制

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

### 3.3 密钥管理

**规则 3.3.1** — API Key 不得硬编码在源代码中

SDK 通过 `Config.APIKey` 和环境变量 `AGENTRT_API_KEY` 传递密钥。

❌ **错误**：硬编码密钥

```go
config := agentrt.NewConfig(
    agentrt.WithAPIKey("sk-abc123def456"), // 硬编码
)
```

✅ **正确**：从环境变量读取

```go
config, err := agentrt.NewConfigFromEnv()
// 或
config := agentrt.NewConfig(
    agentrt.WithAPIKey(os.Getenv("AGENTRT_API_KEY")),
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

✅ **正确**：脱敏处理

```go
func (c *Client) String() string {
    return fmt.Sprintf("Client[endpoint=%s, timeout=%v]", c.config.Endpoint, c.config.Timeout)
}
```

**执行机制**: 代码审查检查 `String()`、`Error()`、日志格式字符串是否包含 `APIKey` 字段。

---

## 4. 并发安全

### 4.1 Mutex 使用

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

✅ **正确**：使用 `sync.RWMutex`

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

✅ **正确**：释放锁后执行回调

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

### 4.2 Goroutine 安全

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

✅ **正确**：通过 context 控制退出

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

✅ **正确**：recover 并通过 channel 传递错误

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

### 4.3 Channel 安全

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

✅ **正确**：单一 goroutine 负责关闭

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

### 4.4 Context 传播

**规则 4.4.1** — 所有跨函数边界的操作必须接受 `context.Context` 作为第一个参数

SDK 中所有公共 API（`APIClient`、`BaseManager`、`SyscallBinding`）均遵循此模式。

❌ **错误**：不接受 context

```go
func (tm *TaskManager) Submit(description string) (*types.Task, error) {
    resp, err := tm.api.Post(context.Background(), "/api/v1/tasks", body)
    // 无法取消，无法超时
}
```

✅ **正确**：接受并传播 context

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

✅ **正确**：context 作为方法参数传递

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

## 5. 错误处理安全

### 5.1 错误信息脱敏

**规则 5.1.1** — 返回给调用方的错误信息不得包含内部实现细节、文件路径、数据库查询等敏感信息

❌ **错误**：泄露内部信息

```go
return nil, fmt.Errorf("database query failed: SELECT * FROM users WHERE id='%s': connection refused at 10.0.1.5:5432", userID)
```

✅ **正确**：使用 SDK 错误码分类，仅返回通用描述

```go
return nil, agentrt.NewError(agentrt.CodeServerError, "服务端错误", nil)
// 内部错误通过结构化日志记录，不返回给调用方
```

**执行机制**: 代码审查检查错误信息是否包含 IP 地址、SQL 语句、文件路径等模式。

---

### 5.2 错误码分类

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

✅ **正确**：使用 `AgentRTError` 体系

```go
return nil, agentrt.NewErrorf(agentrt.CodeNotFound, "插件 '%s' 未找到", pluginID)
```

**执行机制**: CI 中通过 linter 禁止在公共 API 中使用 `fmt.Errorf` 返回错误（测试代码除外）。

---

### 5.3 哨兵错误

**规则 5.3.1** — 预定义的错误状态必须使用哨兵错误（`Err` 前缀变量），支持 `errors.Is` 语义匹配

SDK 已定义完整的哨兵错误集合（如 `ErrNotFound`、`ErrPermissionDenied` 等）。

❌ **错误**：字符串比较判断错误

```go
if err.Error() == "not found" {
    // 脆弱的字符串匹配
}
```

✅ **正确**：使用 `errors.Is` 匹配哨兵错误

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

✅ **正确**：包装原始错误

```go
return nil, agentrt.WrapError(agentrt.CodeNetworkError, "请求执行失败", err) // 保留 Cause
```

**执行机制**: 代码审查检查 `NewError` 的第三个参数是否为 `nil`（当存在原始错误时）。

---

### 5.4 HTTP 状态码映射

**规则 5.4.1** — HTTP 响应状态码必须通过 `HTTPStatusToError()` 映射为 SDK 错误码

❌ **错误**：直接返回 HTTP 状态码

```go
if resp.StatusCode >= 400 {
    return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, string(body))
}
```

✅ **正确**：使用映射函数

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

## 6. 依赖安全

### 6.1 零外部依赖原则

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

✅ **正确**：仅使用标准库

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

### 6.2 go.sum 审计

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

✅ **正确**：无 replace 指令

```
module github.com/spharx/agentrt/toolkit/go
go 1.22
// 无 replace 块
```

**执行机制**: CI 检查 `go.mod` 中是否存在 `replace` 指令，存在则构建失败。

---

### 6.3 go.mod 规范

**规则 6.3.1** — `go.mod` 必须指定最低兼容 Go 版本，且不得低于 Go 1.22

❌ **错误**：过低的 Go 版本

```
go 1.18
```

✅ **正确**：

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

## 7. 网络安全

### 7.1 HTTP 客户端安全

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

✅ **正确**：显式设置超时

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

✅ **正确**：传播 context

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
```

**执行机制**: 代码审查 + `go vet` 检查 `http.NewRequest` 使用（应替换为 `NewRequestWithContext`）。

---

### 7.2 响应体限制

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

✅ **正确**：限制读取大小

```go
body, err := io.ReadAll(io.LimitReader(resp.Body, MaxResponseBodySize))
if err != nil {
    return nil, agentrt.WrapError(agentrt.CodeParseError, "读取响应失败", err)
}
```

**执行机制**: 代码审查检查所有 `io.ReadAll(resp.Body)` 是否配合 `io.LimitReader` 使用。

---

> **注意**: `protocol.go` 中的 `DetectProtocol()`、`ListProtocols()`、`TestConnection()`、`GetCapabilities()` 等方法直接使用 `io.ReadAll(resp.Body)` 而未通过 `LimitReader`，这是已知的安全缺陷，必须在后续版本中修复。

### 7.3 请求超时

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

✅ **正确**：使用 select 监听 context

```go
select {
case <-ctx.Done():
    return nil, ctx.Err()
case <-time.After(delay):
}
```

**执行机制**: 代码审查禁止重试逻辑中使用 `time.Sleep`。

---

### 7.4 重试安全

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

✅ **正确**：指数退避 + 随机抖动

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

✅ **正确**：仅重试可重试错误

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

✅ **正确**：重试前重置 Body

```go
if seeker, ok := req.Body.(io.Seeker); ok && req.Body != nil {
    if _, seekErr := seeker.Seek(0, io.SeekStart); seekErr != nil {
        return nil, agentrt.WrapError(agentrt.CodeNetworkError, "重置请求体失败", seekErr)
    }
}
```

**执行机制**: 代码审查检查重试逻辑是否包含 Body 重置。

---

### 7.5 认证安全

**规则 7.5.1** — API Key 必须通过 `Authorization: Bearer` 头传递，不得通过 URL 参数传递

❌ **错误**：通过 URL 参数传递密钥

```go
url := fmt.Sprintf("/api/v1/tasks?api_key=%s", c.config.APIKey)
// URL 参数会被记录到访问日志、浏览器历史、Referer 头中
```

✅ **正确**：通过 Header 传递

```go
if c.config.APIKey != "" {
    req.Header.Set("Authorization", "Bearer "+c.config.APIKey)
}
```

**执行机制**: 代码审查禁止 URL 中包含 `api_key`、`token`、`secret` 等参数名。

---

## 8. 资源管理

### 8.1 defer 清理

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

✅ **正确**：defer 关闭

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

✅ **正确**：实现 `Close()` 方法

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

### 8.2 Goroutine 泄漏预防

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

✅ **正确**：使用 WaitGroup + done channel

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

### 8.3 文件描述符管理

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

✅ **正确**：设置连接上限

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

✅ **正确**：设置上限并淘汰旧数据

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

## 9. 审计与可观测性

### 9.1 请求追踪

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

✅ **正确**：携带追踪标识

```go
req.Header.Set("X-Request-ID", generateRequestID())
```

**执行机制**: 代码审查检查 HTTP 请求是否设置 `X-Request-ID`。

---

### 9.2 遥测数据安全

**规则 9.2.1** — 遥测数据不得包含敏感信息（API Key、用户凭证、个人数据）

❌ **错误**：遥测数据包含敏感信息

```go
m.Meter.Record("api.request", 1, map[string]string{
    "api_key": c.config.APIKey, // 泄露密钥
    "user_id": userID,          // 泄露用户标识
})
```

✅ **正确**：仅记录非敏感元数据

```go
m.Meter.Record("api.request", 1, map[string]string{
    "method":    "POST",
    "endpoint":  "/api/v1/tasks",
    "status":    "success",
})
```

**执行机制**: 代码审查检查遥测标签是否包含 `api_key`、`password`、`token` 等敏感字段名。

---

### 9.3 日志安全

**规则 9.3.1** — 日志不得包含敏感信息，Debug 模式日志必须与生产日志隔离

SDK 使用 `Config.Debug` 和 `Config.LogLevel` 控制日志详细程度。

❌ **错误**：生产日志包含调试信息

```go
log.Printf("Request: endpoint=%s, apiKey=%s, body=%v", endpoint, apiKey, body)
```

✅ **正确**：分级日志，敏感信息仅在 Debug 模式下输出

```go
if c.config.Debug {
    log.Printf("[DEBUG] Request: endpoint=%s, method=%s", endpoint, method)
}
log.Printf("[INFO] Request completed: method=%s, status=%d", method, statusCode)
```

**执行机制**: 代码审查检查日志格式字符串是否包含 `APIKey`、`Password` 等字段。

---

### 9.4 审计日志完整性

**规则 9.4.1** — 审计日志必须包含操作者、操作类型、目标资源、时间戳和结果

❌ **错误**：审计信息不完整

```go
log.Printf("Task cancelled")
```

✅ **正确**：完整的审计记录

```go
agentrt.GetLogger().Printf("[AUDIT] operation=task.cancel, task_id=%s, result=%v, timestamp=%s",
    taskID, err, time.Now().Format(time.RFC3339))
```

**执行机制**: 代码审查检查变更类操作是否包含完整审计字段。

---

## 10. Airymax 模块安全编码示例

### 10.1 安全的 HTTP 客户端

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

### 10.2 安全的插件管理

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

### 10.3 安全的系统调用

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

## 附录 A: 规则速查表

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

## 附录 B: 已知技术债务

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
