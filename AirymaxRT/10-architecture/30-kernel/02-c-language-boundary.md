# C 语言边界定义 (P4.4.3)

## 1. C 语言边界定义 (C Language Boundary)

C 语言在 Airymax 中承载系统最核心、最底层的基础设施。C 语言边界内的模块构成系统的"骨骼"与"神经"，所有其他语言通过 FFI 层与 C 核心交互。

### 1.1 C 语言职责范围

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              C Language Boundary                                  │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    Agent Runtime Execution Engine                        │   │
│   │                                                                         │   │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │   │
│   │   │CoreKern  │  │CoreLoop  │  │Cupola    │  │Task Queue│  │Context  │  │   │
│   │   │微核心调度 │  │认知循环  │  │安全穹顶  │  │任务队列  │  │上下文   │  │   │
│   │   └──────────┘  └──────────┘  └──────────┘  └──────────┘  └─────────┘  │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                       Critical Infrastructure                            │   │
│   │                                                                         │   │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐           │   │
│   │   │ IPC Bus  │  │Memory    │  │HeapStore │  │MemoryRovol   │           │   │
│   │   │ 进程间   │  │Pool      │  │ 堆存储   │  │ 四层记忆     │           │   │
│   │   │ 通信总线 │  │ 内存池   │  │          │  │ 管理系统     │           │   │
│   │   └──────────┘  └──────────┘  └──────────┘  └──────────────┘           │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                         FFI Bridge                                       │   │
│   │                                                                         │   │
│   │              C Headers ──▶ Auto-generated Bindings                       │   │
│   │                     │                                                   │   │
│   │         ┌───────────┼───────────┬───────────────┐                       │   │
│   │         ▼           ▼           ▼               ▼                       │   │
│   │      Python      Rust         Go          TypeScript                     │   │
│   │      (pyo3/     (bindgen/    (cgo/        (napi-rs/                     │   │
│   │       ctypes)    cbindgen)   cgo)          node-api)                     │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Agent 运行时执行引擎 (Agent Runtime Execution Engine)

Agent 运行时执行引擎是 Airymax 的"心脏"，全部由 C 语言实现：

| 模块 | 说明 |
|------|------|
| **CoreKern** | MicroCoreRT 微核心调度器，负责系统级调度、任务路由、上下文管理。所有服务请求的中央仲裁者。 |
| **CoreLoopThree** | 核心认知循环，实现 Cognition → Planning → Action 三阶段流水线。 |
| **Cupola** | 安全穹顶框架，可插拔的服务加载与生命周期管理。 |
| **Task Queue** | 任务队列，基于优先级的多级调度队列。 |
| **Context Manager** | 上下文管理器，维护 Agent 会话状态与上下文窗口。 |

### 1.3 关键基础设施 (Critical Infrastructure)

以下基础设施模块必须在 C 语言边界内实现，因为它们直接管理内存、进程间通信等系统级资源：

| 模块 | 说明 |
|------|------|
| **IPC Bus** | 进程间通信总线，基于共享内存 (shared memory) 和消息队列 (message queue) 实现零拷贝通信。 |
| **Memory Pool** | 内存池，预分配内存块，减少 malloc/free 开销，避免内存碎片化。 |
| **HeapStore** | 堆存储引擎，为 Agent 提供持久化数据的键值存储。 |
| **MemoryRovol** | 四层记忆管理系统 (Working → Short-term → Long-term → Archival)，管理 token budget 与 memory budget。 |

### 1.4 FFI Bridge (Foreign Function Interface)

FFI Bridge 是 C 语言边界与上层语言的唯一通道：

- **C Headers 是唯一的数据结构定义源** (single source of truth)
- 通过自动化工具从 C 头文件生成各语言绑定：
  - Python: `pyo3` / `ctypes` / `cffi`
  - Rust: `bindgen` / `cbindgen`
  - Go: `cgo`
  - TypeScript: `napi-rs` / `node-api`
- FFI 层仅负责数据序列化/反序列化与函数调用转发，不包含任何业务逻辑

---

## 2. 为什么 C 是核心语言 (Why C is the Core Language)

### 2.1 Performance (性能)

- **零成本抽象 (Zero-cost abstractions)**：C 语言不引入运行时开销，所有抽象在编译期完成。
- **直接内存控制 (Direct memory control)**：精确控制内存布局与访问模式，对于 AI 工作负载中的 token budget 和 memory budget 管理至关重要。
- **无 GC 停顿 (No GC pauses)**：Agent 实时推理场景下，GC 停顿不可接受。C 语言的手动内存管理保证确定性延迟。

### 2.2 Portability (可移植性)

- **裸机运行 (Bare metal)**：C 运行时极简，可在无操作系统的嵌入式环境运行。
- **容器化部署 (Containers)**：最小镜像体积，冷启动时间 < 10ms。
- **嵌入式系统 (Embedded systems)**：支持 ARM Cortex-M、RISC-V 等 MCU 平台。
- **WASM 目标**：C 代码可编译为 WebAssembly，在浏览器和边缘节点运行。

### 2.3 Stability (稳定性)

- **C ABI 是通用 FFI 标准 (Universal FFI standard)**：几乎所有编程语言都支持调用 C ABI。Python 的 `ctypes`、Rust 的 `extern "C"`、Go 的 `cgo`、Node.js 的 `napi` 均基于 C ABI。
- **ABI 向后兼容 (ABI backward compatibility)**：C ABI 稳定，不会因编译器版本升级而破坏二进制兼容性。
- **长期支持 (Long-term support)**：C 语言标准 (C11/C17/C23) 演进保守，核心代码可维护数十年。

### 2.4 Control (控制力)

- **精确的内存管理 (Precise memory management)**：手动管理 token budget、memory budget、context window 等 AI 工作负载特有的资源约束。
- **确定性行为 (Deterministic behavior)**：无 JIT 编译、无运行时优化导致的性能波动。
- **系统级能力 (System-level capabilities)**：直接操作 `mmap`、`sched_setaffinity` (CPU affinity)、`mlock` (memory locking) 等系统调用。

---

## 3. Rust 职责 (Rust's Responsibilities)

Rust 在 Airymax 中负责面向用户和开发者的交互层，利用其内存安全特性提供可靠的开发体验：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Rust Boundary                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────────────┐   ┌──────────────────┐                      │
│   │   CLI / TUI      │   │   Plugin SDK      │                      │
│   │                   │   │                    │                      │
│   │ • agentrt CLI     │   │ • Plugin trait     │                      │
│   │ • agentrt tui     │   │ • Sandbox API      │                      │
│   │ • 终端仪表盘      │   │ • 安全隔离         │                      │
│   │ • 实时监控        │   │ • 类型安全         │                      │
│   │                   │   │                    │                      │
│   └────────┬──────────┘   └────────┬───────────┘                      │
│            │                       │                                  │
│            └───────────┬───────────┘                                  │
│                        │                                              │
│                   FFI (C ABI)                                         │
│                        │                                              │
│                        ▼                                              │
│                  C Core Engine                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.1 CLI / TUI (命令行界面与终端 UI)

- **agentrt CLI**：命令行管理工具，提供启动、停止、监控、配置等操作。
- **agentrt TUI**：终端仪表盘，实时显示 Agent 运行状态、资源消耗、token 使用量等指标。
- 利用 Rust 的 `clap`、`ratatui` 等生态库构建高性能终端应用。

### 3.2 Plugin SDK (插件开发套件)

- **Plugin trait**：定义插件接口，提供类型安全的插件开发体验。
- **Sandbox API**：基于 WASM 或进程隔离的插件沙箱，防止插件崩溃影响核心系统。
- **Memory safety without GC**：Rust 的所有权模型保证插件内存安全，无需 GC，适合与 C 核心的 FFI 交互。

---

## 4. Python 职责 (Python's Responsibilities)

Python 在 Airymax 中负责编排层和实验层，利用其生态丰富性和开发效率：

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Python Boundary                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────────────┐   ┌──────────────────┐                      │
│   │  Manager Layer   │   │    OpenLab       │                      │
│   │                   │   │                    │                      │
│   │ • 服务编排        │   │ • Prompt 调优      │                      │
│   │ • 配置管理        │   │ • A/B Testing      │                      │
│   │ • 部署脚本        │   │ • 实验追踪         │                      │
│   │ • 健康检查        │   │ • 评估基准         │                      │
│   │                   │   │                    │                      │
│   └────────┬──────────┘   └────────┬───────────┘                      │
│            │                       │                                  │
│            └───────────┬───────────┘                                  │
│                        │                                              │
│                   FFI (C ABI)                                         │
│                        │                                              │
│                        ▼                                              │
│                  C Core Engine                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.1 Manager Layer (编排层)

- **服务编排 (Orchestration)**：管理多个 Agent 实例的生命周期，负责启动、停止、扩缩容。
- **配置管理 (Configuration)**：YAML/TOML 配置文件的解析、验证与分发。
- **部署脚本 (Deployment scripts)**：Docker Compose、Kubernetes Helm Chart 的生成与管理。

### 4.2 OpenLab (实验层)

- **Prompt 调优 (Prompt optimization)**：利用 Python 生态 (如 `dspy`、`langchain`) 进行 prompt 工程。
- **A/B Testing**：对比不同 prompt 策略的效果，自动化评估与选择。
- **实验追踪 (Experiment tracking)**：记录每次实验的参数、指标与结果，支持 MLflow / WandB 集成。

### 4.3 CI/CD Scripts

- 构建流水线脚本 (Build pipeline scripts)
- 自动化测试编排 (Automated test orchestration)
- 发布管理 (Release management)

---

## 5. Go 职责 (Go's Responsibilities)

Go 在 Airymax 中负责网络密集型服务，利用其并发模型和网络库优势：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Go Boundary                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌────────────────────────────────────────────────────────────┐    │
│   │                      Gateway Service                        │    │
│   │                                                             │    │
│   │   • HTTP/1.1, HTTP/2, HTTP/3 支持                          │    │
│   │   • WebSocket 长连接管理 (百万级并发)                        │    │
│   │   • JSON-RPC / MCP / A2A 协议适配                           │    │
│   │   • 请求路由、限流、鉴权、负载均衡                            │    │
│   │   • TLS 终止 (TLS termination)                              │    │
│   │                                                             │    │
│   └────────────────────────────────────────────────────────────┘    │
│                                │                                     │
│                           FFI (C ABI)                                │
│                                │                                     │
│                                ▼                                     │
│                          C Core Engine                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.1 Gateway Service (网关服务)

- **HTTP/WebSocket 处理**：Go 的 goroutine 模型天然适合处理海量并发连接，每个连接一个 goroutine 开销极低。
- **协议适配**：将外部 JSON-RPC、MCP (Model Context Protocol)、A2A (Agent-to-Agent) 协议转换为内部 IPC 消息。
- **网络层关注点**：
  - 请求路由 (Request routing)
  - 速率限制 (Rate limiting)
  - 鉴权与授权 (Authentication & Authorization)
  - 负载均衡 (Load balancing)
  - TLS 终止 (TLS termination)

### 5.2 网络密集型服务 (Network-heavy Services)

- 任何以网络 I/O 为主要瓶颈的服务均应使用 Go 实现。
- 避免在 Go 层实现任何核心业务逻辑 (如认知循环、记忆管理)。

---

## 6. TypeScript 职责 (TypeScript's Responsibilities)

TypeScript 在 Airymax 中负责 Web 前端和前端集成 SDK：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     TypeScript Boundary                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────────────────────┐   ┌──────────────────────────┐       │
│   │    Web Dashboard         │   │   Frontend SDK           │       │
│   │                          │   │                          │       │
│   │ • Agent 状态监控面板     │   │ • @agentrt/sdk           │       │
│   │ • 实时日志查看           │   │ • React/Vue 组件库       │       │
│   │ • 配置管理界面           │   │ • WebSocket 客户端       │       │
│   │ • 可视化工作流编辑器     │   │ • 类型安全的 API 调用    │       │
│   │                          │   │                          │       │
│   └────────────┬─────────────┘   └────────────┬─────────────┘       │
│                │                              │                      │
│                │     HTTP / WebSocket         │                      │
│                └──────────────┬───────────────┘                      │
│                               │                                      │
│                               ▼                                      │
│                        Go Gateway                                    │
│                               │                                      │
│                          FFI (C ABI)                                 │
│                               │                                      │
│                               ▼                                      │
│                         C Core Engine                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.1 Web Dashboard

- **Agent 状态监控面板**：实时展示 Agent 运行状态、资源使用、token 消耗等。
- **实时日志查看**：基于 WebSocket 的实时日志流。
- **配置管理界面**：可视化的配置文件编辑与管理。
- **可视化工作流编辑器**：拖拽式 Agent 工作流编排。

### 6.2 Frontend SDK

- **@agentrt/sdk**：npm 包，提供类型安全的 Airymax API 调用。
- **React / Vue 组件库**：预构建的 UI 组件，方便前端集成。
- **WebSocket 客户端**：与 Gateway 的 WebSocket 连接管理。

---

## 7. 跨语言调用规则 (Cross-language Calling Rules)

### 7.1 核心原则

```
                            ┌─────────────────┐
                            │   C Headers      │
                            │  (Single Source  │
                            │   of Truth)      │
                            └────────┬────────┘
                                     │
                              Auto-generation
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
   │  libagentrt-sys  │   │  libagentrt-ffi  │   │  agentrt.h       │
   │  (Rust -sys      │   │  (Python C       │   │  (Go cgo /       │
   │   crate)         │   │   extension)     │   │   TypeScript     │
   │                  │   │                  │   │   napi-rs)       │
   └────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
            │                      │                      │
            ▼                      ▼                      ▼
   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
   │  agentrt-rs      │   │  agentrt-py      │   │  @agentrt/sdk    │
   │  (Safe Rust      │   │  (Pythonic       │   │  (TS/JS          │
   │   wrapper)       │   │   wrapper)       │   │   wrapper)       │
   └──────────────────┘   └──────────────────┘   └──────────────────┘
```

### 7.2 规则清单

| 规则 | 说明 |
|------|------|
| **Rule 1: C 是唯一数据源** | 所有核心数据结构 (task、message、context、memory entry 等) 在 C 头文件中定义，不允许在其他语言中重复定义。 |
| **Rule 2: FFI 层自动生成** | 通过 `cbindgen`、`bindgen`、`pyo3` 等工具从 C 头文件自动生成绑定代码，禁止手写 FFI 绑定。 |
| **Rule 3: SDK 封装 FFI** | 各语言的 SDK (Python agentrt-py、Rust agentrt-rs、TS @agentrt/sdk) 封装裸 FFI 调用，提供 idiomatic API。 |
| **Rule 4: 禁止业务逻辑重复** | 任何业务逻辑不得在多个语言中重复实现。C 核心是唯一实现源，上层语言仅做 thin wrapper。 |
| **Rule 5: 单向依赖** | 上层语言依赖 C 核心，C 核心不依赖任何上层语言。依赖方向必须单向。 |
| **Rule 6: 错误传播** | C 核心通过错误码 (error code) 返回错误，FFI 层将错误码转换为各语言原生异常/错误类型。 |

### 7.3 调用链示例

```
TypeScript SDK                    Go Gateway                   C Core
     │                                │                          │
     │  HTTP POST /api/agent/run      │                          │
     │───────────────────────────────▶│                          │
     │                                │                          │
     │                                │  CGO call: agentrt_run() │
     │                                │─────────────────────────▶│
     │                                │                          │
     │                                │                          │
     │                                │  return: task_id, status │
     │                                │◀─────────────────────────│
     │                                │                          │
     │  Response: { taskId, status }  │                          │
     │◀───────────────────────────────│                          │
     │                                │                          │
```

---

## 8. 边界违规示例 (Boundary Violation Examples)

### 8.1 禁止事项 (DON'T)

| 违规行为 | 错误原因 | 正确做法 |
|----------|----------|----------|
| **在 Python 中实现内存管理** | 内存管理是 C 核心的职责，Python 的 GC 无法精确控制 AI 工作负载的 token budget 和 memory budget。 | 通过 FFI 调用 C 的 Memory Pool / MemoryRovol API。 |
| **在 Go 中实现认知循环** | 认知循环 (Cognition → Planning → Action) 是 Agent 的核心执行逻辑，必须由 C 实现以保证性能和确定性。 | Go Gateway 仅负责协议转换，将请求转发给 C CoreLoop。 |
| **绕过 FFI 层直接通信** | 绕过 FFI 层意味着绕过了类型检查、错误传播、版本兼容性等保障机制。 | 所有跨语言调用必须经过 FFI Bridge。 |
| **在 Rust 中重复定义数据结构** | 违反 "C 是唯一数据源" 原则，导致数据结构不一致和同步问题。 | 通过 `bindgen` 从 C 头文件自动生成 Rust 结构体。 |
| **在 TypeScript 中实现任务调度** | 任务调度涉及优先级、抢占、资源隔离等底层机制，前端 JavaScript 无法胜任。 | 通过 WebSocket 将调度指令发送给 C CoreKern。 |
| **在 Python 中实现 IPC 通信** | IPC 通信基于共享内存和信号量，Python 的 GIL 和内存模型不适合此类底层操作。 | 通过 FFI 调用 C 的 IPC Bus API。 |

### 8.2 正确模式 (Correct Patterns)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Correct Layering                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   TypeScript ──── UI / Frontend SDK ──── (HTTP/WS) ────┐            │
│                                                         │            │
│   Go ──────────── Gateway / Network ──── (HTTP/WS) ────┤            │
│                                                         │            │
│   Python ──────── Manager / OpenLab ──── (FFI) ────────┤            │
│                                                         │            │
│   Rust ────────── CLI / Plugin SDK ────── (FFI) ────────┤            │
│                                                         │            │
│                                          ┌──────────────▼────────┐  │
│                                          │     FFI Bridge         │  │
│                                          │  (Auto-generated from  │  │
│                                          │   C Headers)           │  │
│                                          └──────────────┬────────┘  │
│                                                         │            │
│                                          ┌──────────────▼────────┐  │
│                                          │     C Core Engine      │  │
│                                          │                        │  │
│                                          │  • CoreKern            │  │
│                                          │  • CoreLoopThree       │  │
│                                          │  • IPC Bus             │  │
│                                          │  • Memory Pool         │  │
│                                          │  • HeapStore           │  │
│                                          │  • MemoryRovol         │  │
│                                          └────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.3 设计决策树

当需要实现新功能时，按以下决策树选择实现语言：

```
新功能需求
    │
    ├── 是否涉及 Agent 核心执行逻辑？
    │       └── 是 ──▶ C (CoreLoop / CoreKern)
    │
    ├── 是否涉及内存/资源管理？
    │       └── 是 ──▶ C (Memory Pool / MemoryRovol / HeapStore)
    │
    ├── 是否涉及进程间通信？
    │       └── 是 ──▶ C (IPC Bus)
    │
    ├── 是否涉及网络协议处理？
    │       └── 是 ──▶ Go (Gateway)
    │
    ├── 是否涉及用户交互界面？
    │       ├── 终端 ──▶ Rust (CLI/TUI)
    │       └── Web  ──▶ TypeScript (Dashboard)
    │
    ├── 是否涉及编排/实验/脚本？
    │       └── 是 ──▶ Python (Manager / OpenLab)
    │
    └── 是否涉及插件开发体验？
            └── 是 ──▶ Rust (Plugin SDK)
```

---

## 9. 版本兼容性承诺

| 边界 | 兼容性承诺 |
|------|-----------|
| C ABI | 主版本内向后兼容，结构体尾部追加字段，不删除/重排已有字段 |
| FFI 绑定 | 自动生成，与 C 头文件同步更新，CI 中验证绑定一致性 |
| SDK API | 语义化版本 (SemVer)，主版本升级需提供迁移指南 |

---

*文档版本: 1.0 | 最后更新: 2026-06-18 | 对应规格: P4.4.3*