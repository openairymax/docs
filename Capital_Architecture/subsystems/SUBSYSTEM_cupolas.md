# SUBSYSTEM_cupolas — 安全穹顶

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/cupolas/`  
**负责人**: Team A (核心引擎)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

cupolas 是 Airymax 的安全穹顶模块，为所有 Agent 操作提供运行时安全防护。每个 Agent 在穹顶内以最小权限运行，所有操作经过安全裁决。

### 1.1 核心职责

| 职责 | 模块 | 说明 |
|------|------|------|
| **权限裁决引擎** | `permission/` | 基于规则的最小权限访问控制 (RBAC) |
| **输入净化器** | `sanitizer/` | 注入攻击防护（SQL/命令/脚本注入） |
| **审计日志** | `audit/` | 全量操作追踪与合规审计 |
| **虚拟工位** | `workbench/` | 代理隔离执行环境 |
| **熔断器** | `circuit_breaker.h` | 异常行为检测与自动熔断 |
| **安全守卫** | `guards/` | SafetyGuard 守卫链框架 |
| **运行时保护** | `cupolas_runtime_protection.h` | 内存保护、栈保护、ASLR |
| **网络安全** | `cupolas_network_security.h` | TLS、DNS、HTTP 安全过滤 |
| **签名验证** | `cupolas_signature.h` | 代码签名与完整性校验 |
| **密钥保险库** | `cupolas_vault.h` | 凭据安全存储与访问 |
| **授权管理** | `cupolas_entitlements.h` | Agent 能力授权声明 |
| **零信任集成** | `zero_trust_integration.h` | 零信任架构适配 |
| **动态策略引擎** | `dynamic_policy_engine.h` | 运行时动态策略更新 |

### 1.2 设计原则

- **Security by Default**: 每个 Agent 默认在穹顶内以最小权限运行
- **High Performance**: 权限缓存 + 异步审计写入
- **Cross-Platform**: Windows / Linux / macOS

### 1.3 不在范围内的职责

- 不负责网络防火墙（操作系统级别）
- 不负责数据加密算法实现（使用系统库）
- 不负责身份认证（由网关层处理）

---

## 2. 输入/输出接口

### 2.1 初始化与清理

```c
int cupolas_init(const char *config_path, agentos_error_t *error);
void cupolas_cleanup(void);
const char *cupolas_version(void);
```

### 2.2 权限管理

| 函数 | 说明 |
|------|------|
| `cupolas_check_permission(agent_id, action, resource, context)` | 检查权限（1=允许, 0=拒绝, <0=错误） |
| `cupolas_add_permission_rule(agent_id, action, resource, allow, priority)` | 添加权限规则 |
| `cupolas_clear_permission_cache()` | 清除权限缓存 |

**权限动作**: `"read"`, `"write"`, `"execute"`

### 2.3 输入净化

| 函数 | 说明 |
|------|------|
| `cupolas_sanitize_input(input, output, output_size)` | 净化输入字符串 |

### 2.4 命令执行

```c
int cupolas_execute_command(const char *command, char *const argv[],
                            int *exit_code,
                            char *stdout_buf, size_t stdout_size,
                            char *stderr_buf, size_t stderr_size);
```

### 2.5 审计

| 函数 | 说明 |
|------|------|
| `cupolas_flush_audit_log()` | 刷新审计日志到存储 |

### 2.6 安全子模块（独立引入）

| 头文件 | 功能 |
|--------|------|
| `cupolas_signature.h` | 代码签名与完整性校验 |
| `cupolas_vault.h` | 凭据安全存储 |
| `cupolas_entitlements.h` | Agent 能力授权声明 |
| `cupolas_runtime_protection.h` | 运行时内存保护 |
| `cupolas_network_security.h` | 网络安全过滤 |
| `guard_core.h` | SafetyGuard 守卫核心 |
| `guard_integration.h` | SafetyGuard 集成接口 |

### 2.7 错误码别名

cupolas 提供与 corekern 错误码兼容的别名（注意：corekern 定义 `AGENTOS_SUCCESS`，cupolas 将其别名为 `cupolas_OK`）：

```c
#define cupolas_OK               AGENTOS_SUCCESS       // corekern 的 AGENTOS_SUCCESS
#define cupolas_ERROR_UNKNOWN    AGENTOS_ERR_UNKNOWN
#define cupolas_ERROR_INVALID_ARG AGENTOS_ERR_INVALID_PARAM
#define cupolas_ERROR_NO_MEMORY  AGENTOS_ERR_OUT_OF_MEMORY
#define cupolas_ERROR_PERMISSION AGENTOS_ERR_PERMISSION_DENIED
#define cupolas_ERROR_TIMEOUT    AGENTOS_ERR_TIMEOUT
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | 错误码 (`error.h`)、内存管理 |
| **commons** | 强依赖 | 类型定义 (`agentos_types.h`) |
| **libyaml** | 可选 | YAML 配置解析（`yaml_minimal.h`） |

### 3.2 被依赖方

- **tool_d**: 通过 `safety_guard_bridge.h` 桥接安全守卫，工具执行前进行权限检查
- **hook_d**: 拦截型 Hook 通过 `hook_interceptor` 调用 SafetyGuard 守卫链
- **gateway_d**: 输入净化、网络安全过滤
- **syscall**: 系统调用层的安全沙箱

### 3.3 安全集成架构

```
┌──────────────────────────────────────────────────┐
│                    Agent 操作                      │
└──────────────────────┬───────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
┌─────────────────┐          ┌──────────────┐
│  cupolas         │          │ SafetyGuard   │
│  权限裁决引擎    │◄─────────│ 守卫链        │
│  Permission      │  hook_d  │ Guard Chain   │
└────────┬────────┘          └──────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌────────┐
│ 净化器 │ │ 审计   │
└───────┘ └────────┘
```

---

## 4. 配置参数

### 4.1 初始化配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `config_path` | NULL | 配置文件路径（NULL 使用默认配置） |

### 4.2 权限规则配置

| 规则字段 | 说明 |
|----------|------|
| `agent_id` | Agent ID 模式（`"*"` 通配） |
| `action` | 动作模式（`"read"`, `"write"`, `"execute"`） |
| `resource` | 资源路径（支持 glob 模式） |
| `allow` | 1=允许, 0=拒绝 |
| `priority` | 优先级（数值越高越优先） |

### 4.3 编译选项

| 选项 | 说明 |
|------|------|
| `BUILD_CUPOLAS` | 是否编译安全穹顶模块（默认 ON） |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `cupolas_init` | 是 | 多次调用只有首次成功 |
| `cupolas_cleanup` | 是 | 阻塞直到所有操作完成 |
| `cupolas_check_permission` | 是 | 内部使用锁保护 |
| `cupolas_add_permission_rule` | 是 | 线程安全 |
| `cupolas_sanitize_input` | 是 | 可重入 |
| `cupolas_execute_command` | 否 | 每个工位实例单线程 |
| `cupolas_flush_audit_log` | 是 | 线程安全 |

---

## 6. 相关文档

- [安全设计规范](../../Capital_Specifications/coding_standard/Security_design_standard.md)
- [C/C++ 安全编码规范](../../Capital_Specifications/coding_standard/C_Cpp_secure_coding_standard.md)
- [系统调用架构](../syscall.md)