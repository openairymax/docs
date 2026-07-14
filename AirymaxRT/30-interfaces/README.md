Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# AirymaxAgentRT 接口文档

**最新更新**: 2026-07-11
**状态**: 维护中
**路径**: `docs/AirymaxRT/30-interfaces/README.md`

---

## 📚 文档导航

本目录包含 agentrt 的全部对外接口文档，涵盖系统调用 API、运行时标准接口（ARE L1-L3）、IPC 协议、RPC API、服务发现、核心算法、多语言 SDK 等。

---

### 核心接口

- [**Gateway API 参考**](02-api-reference.md) — HTTP/WebSocket 端点 API（JSON-RPC 2.0 方法、REST 端点、协议适配器、错误码、速率限制）
- [**CLI 命令参考**](03-cli-reference.md) — 统一命令行工具完整命令参考
- [**Doxygen 指南**](01-doxygen-guide.md) — C 代码 Doxygen 文档生成指南

### 运行时标准接口（ARE L1-L3）

- [**L1 运行时接口**](05-runtime-interfaces/01-l1-runtime-interface.md) — 可移植运行时接口（IPC / 内存 / 同步 / 任务）
- [**L2 服务协议**](05-runtime-interfaces/02-l2-service-protocol.md) — 服务层协议（Agent / Skill / Memory / Session）
- [**L3 安全治理**](05-runtime-interfaces/03-l3-security-governance.md) — 安全治理接口（沙箱 / 权限 / 审计）

### 通信协议

- [**IPC 协议**](06-ipc/README.md) — 128B 消息头、payload 类型、请求-响应/发布-订阅模式
- [**RPC API**](07-rpc-api/README.md) — JSON-RPC 2.0 接口规范
- [**服务发现**](08-service-discovery/README.md) — 服务注册与发现机制

### 系统 API

- [**CoreLoop API**](20-core/01-coreloop-api.md) — 核心循环 API（创建/销毁/提交/等待）
- [**Daemon API**](30-daemon/01-api-documentation.md) — Daemon 接口文档
- [**Gateway API**](30-daemon/02-gateway-api.md) — Gateway 服务 API

**系统调用 API**：

| 模块 | 文档 |
|------|------|
| Agent 管理 | [01-agent.md](60-syscalls/01-agent.md) |
| Memory 管理 | [02-memory.md](60-syscalls/02-memory.md) |
| Session 管理 | [03-session.md](60-syscalls/03-session.md) |
| Skill 管理 | [04-skill.md](60-syscalls/04-skill.md) |
| Task 管理 | [05-task.md](60-syscalls/05-task.md) |
| 遥测 | [06-telemetry.md](60-syscalls/06-telemetry.md) |

### 多语言 SDK

| 语言 | 文档 |
|------|------|
| Go | [README.md](70-toolkit/10-go/README.md) |
| Python | [README.md](70-toolkit/20-python/README.md) |
| Rust | [README.md](70-toolkit/30-rust/README.md) |
| TypeScript | [README.md](70-toolkit/40-typescript/README.md) |

- [**协议指南**](70-toolkit/01-protocol-guide.md) — 协议适配器开发指南
- [**协议快速入门**](70-toolkit/02-protocol-quickstart.md) — 协议快速集成

### 算法与示例

- [**核心算法**](10-algorithms/README.md) — 文档处理 / 搜索索引 / 质量验证 / 性能优化
- [**Docker 集成**](40-docker/README.md) — 容器化部署完整方案
- [**快速入门**](50-examples/01-quickstart.md) — 5 分钟从零到 Hello World

---

## 📖 API 设计原则

### 错误处理约定

所有 API 函数返回 `airy_err_t` 类型（`int32_t`）：

> **SSoT 声明**：错误码权威定义位于 `include/airymax/error.h`（[SC] 共享契约层，agentrt 与 agentrt-linux 共享同一物理头文件）。类型 `airy_err_t = int32_t` 定义于 `airy_types.h:41`。本表与 `docs/AirymaxRT/50-engineering-standards/120-cross-project-code-sharing.md` §2.1（13 个 `AIRY_E*` 宏，对齐 POSIX errno 负值）逐字节一致。

| 错误码 | 值 | 含义 |
|--------|---|------|
| `AIRY_EOK` | 0 | 成功 |
| `AIRY_EPERM` | -1 | 权限不足（对齐 POSIX EPERM） |
| `AIRY_ENOENT` | -2 | 不存在（对齐 POSIX ENOENT） |
| `AIRY_ENOMEM` | -12 | 内存不足（对齐 POSIX ENOMEM） |
| `AIRY_EINVAL` | -22 | 无效参数（对齐 POSIX EINVAL） |
| `AIRY_EBUSY` | -16 | 资源忙（对齐 POSIX EBUSY） |
| `AIRY_ETIMEDOUT` | -110 | 操作超时（对齐 POSIX ETIMEDOUT） |

### 内存管理约定

- **分配**: 使用 `AIRY_MALLOC`, `AIRY_CALLOC`, `AIRY_STRDUP`
- **释放**: 使用 `AIRY_FREE`
- **规则**: 谁分配谁释放（除非文档明确说明转移所有权）

### 线程安全约定

- 所有标记为 `_t` 的句柄类型都是线程安全的
- 内部使用互斥锁保护共享状态
- 回调函数会在独立的线程上下文中调用

### 命名规范

- 函数: `<module>_<action>_<object>`（如 `airy_loop_create`）
- 类型: `<module>_<object>_t`（如 `airy_core_loop_t`）
- 常量: `MODULE_CONSTANT_NAME`（如 `AIRY_EOK`）
- 枚举: `MODULE_TYPE_VALUE`（如 `TASK_STATUS_PENDING`）

> **详见**: [命名规范](../50-engineering-standards/10-coding-style/naming_conventions.md)

---

## 🔗 相关文档

- [架构设计](../10-architecture/) — 系统架构与设计原则
- [模块设计](../20-modules/) — 契约规范
- [工程标准](../50-engineering-standards/) — 共享工程标准
- [应用开发](../140-application-development/) — 开发者指南

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
