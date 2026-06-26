# SUBSYSTEM_manager — 配置与告警管理

**版本**: v0.1.0  
**状态**: 稳定  
**路径**: `agentos/daemon/common/` (config_manager.h + alert_manager.h)  
**负责人**: Team C (基础设施)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

manager 子系统是 Airymax 的统一配置管理与智能告警系统，为所有守护进程和模块提供配置中心化管理和运行时告警监控能力。

### 1.1 核心职责

#### 配置管理 (config_manager.h)

| 职责 | 说明 |
|------|------|
| **多格式配置** | 支持 JSON / YAML / INI / ENV 多种配置格式 |
| **配置热更新** | 文件监视 + 回调通知，无需重启服务 |
| **配置版本控制** | 变更历史记录 + 回滚到指定版本 |
| **环境差异化** | dev / staging / prod 环境配置隔离 |
| **配置校验** | 自定义校验器，配置写入前验证 |
| **跨进程同步** | 基于共享内存的跨进程配置同步 |
| **命名空间** | 按模块隔离配置命名空间 |

#### 告警管理 (alert_manager.h)

| 职责 | 说明 |
|------|------|
| **多级告警** | INFO / WARNING / CRITICAL / EMERGENCY 四级告警 |
| **告警规则引擎** | 阈值 / 趋势 / 组合条件 / 异常检测 四种规则 |
| **告警抑制** | 重复告警去重与抑制 |
| **多通道通知** | 日志 / 回调 / Webhook / 文件 四种通知通道 |
| **告警升级** | 长时间未处理的告警自动升级 |
| **熔断器联动** | 与 circuit_breaker 集成 |
| **服务发现联动** | 与 service_discovery 集成 |

### 1.2 不在范围内的职责

- 不负责服务注册与发现（由 service_discovery 负责）
- 不负责指标收集（由 monit_d 负责）
- 不负责日志系统（由 logging 模块负责）

---

## 2. 输入/输出接口

### 2.1 配置管理接口 (config_manager.h)

#### 生命周期

```c
int cm_init(const cm_config_t *config);
void cm_shutdown(void);
```

#### 配置读写

```c
const char *cm_get(const char *key, const char *default_value);
int64_t cm_get_int(const char *key, int64_t default_value);
double cm_get_double(const char *key, double default_value);
bool cm_get_bool(const char *key, bool default_value);
int cm_set(const char *key, const char *value, const char *source);
int cm_set_namespaced(const char *namespace_, const char *key,
                      const char *value, const char *source);
```

#### 配置加载

```c
int cm_load_json(const char *path, const char *namespace_);  // 支持 JSON/YAML/INI
int cm_load_env(const char *prefix, const char *namespace_);
int cm_load_args(int argc, char **argv);
```

#### 热更新与监视

```c
int cm_watch(const char *key_pattern, cm_change_callback_t callback, void *user_data);
int cm_unwatch(const char *key_pattern, cm_change_callback_t callback);
int cm_reload(void);
```

#### 校验与版本控制

```c
int cm_register_validator(const char *key_pattern, cm_validator_t validator);
int cm_validate_all(void);
int cm_get_history(const char *key, cm_change_record_t *records,
                   uint32_t max_count, uint32_t *found_count);
int cm_rollback(const char *key, uint64_t version);
```

#### 环境与导出

```c
const char *cm_get_environment(void);
int cm_set_environment(const char *env);
int cm_load_environment_config(const char *env);
char *cm_export_json(const char *namespace_);
```

### 2.2 告警管理接口 (alert_manager.h)

#### 生命周期

```c
int am_init(const am_config_t *config);
void am_shutdown(void);
```

#### 规则管理

```c
int am_add_rule(const am_rule_t *rule);
int am_remove_rule(const char *name);
int am_set_rule_enabled(const char *name, bool enabled);
```

#### 告警触发

```c
int am_fire(const char *name, am_level_t level, const char *message,
            const char *source, const char *labels);
int am_resolve(const char *name);
int am_acknowledge(const char *name);
```

#### 指标评估

```c
int am_record_metric(const char *metric_name, double value);
int am_evaluate(const char *metric_name, double value);
int am_evaluate_all(void);
```

#### 通知与查询

```c
int am_register_channel(const am_channel_t *channel);
int am_register_callback(am_alert_callback_t callback, void *user_data, am_level_t min_level);
int am_get_active_alerts(am_alert_t *alerts, uint32_t max_count, uint32_t *found_count);
int am_get_alerts_by_level(am_level_t level, am_alert_t *alerts,
                           uint32_t max_count, uint32_t *found_count);
uint32_t am_active_alert_count(void);
```

### 2.3 告警级别

| 级别 | 枚举值 | 说明 |
|------|--------|------|
| Info | `AM_LEVEL_INFO` | 信息通知 |
| Warning | `AM_LEVEL_WARNING` | 警告 |
| Critical | `AM_LEVEL_CRITICAL` | 严重 |
| Emergency | `AM_LEVEL_EMERGENCY` | 紧急 |

### 2.4 告警状态

| 状态 | 枚举值 | 说明 |
|------|--------|------|
| Pending | `AM_STATE_PENDING` | 待处理 |
| Firing | `AM_STATE_FIRING` | 触发中 |
| Resolved | `AM_STATE_RESOLVED` | 已解决 |
| Suppressed | `AM_STATE_SUPPRESSED` | 已抑制 |
| Acknowledged | `AM_STATE_ACKNOWLEDGED` | 已确认 |

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **corekern** | 强依赖 | 内存管理、错误码 |
| **daemon/common** | 强依赖 | 共享服务框架 |
| **circuit_breaker** | 可选依赖 | 告警触发熔断 |
| **service_discovery** | 可选依赖 | 告警联动服务发现 |

### 3.2 被依赖方

- **所有 daemon**: 通过 `cm_get()` / `cm_set()` 读写配置
- **gateway_d**: 通过 `POST /api/v1/config/reload` 触发配置重载
- **monit_d**: 告警指标上报
- **CLI**: 通过 `agentrt config reload` 触发配置重载

### 3.3 配置加载优先级

```
1. 命令行参数 (cm_load_args)          ← 最高优先级
2. 环境变量 (cm_load_env)             ← 次高优先级
3. 环境配置文件 (cm_load_environment_config)
4. 基础配置文件 (cm_load_json)        ← 最低优先级
```

---

## 4. 配置参数

### 4.1 配置管理器配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `base_path` | 由调用者指定 | 配置基础路径 |
| `environment` | "dev" | 当前环境 |
| `watch_interval_ms` | 由调用者指定 | 文件监视间隔 |
| `max_history` | 64 | 最大历史记录数 |
| `enable_hot_reload` | true | 启用热更新 |
| `enable_validation` | true | 启用配置校验 |
| `enable_cross_process_sync` | false | 启用跨进程同步 |

### 4.2 常量限制

| 常量 | 值 | 说明 |
|------|-----|------|
| `CM_MAX_KEY_LEN` | 128 | 最大键长度 |
| `CM_MAX_VALUE_LEN` | 2048 | 最大值长度 |
| `CM_MAX_ENTRIES` | 512 | 最大条目数 |
| `CM_MAX_WATCHERS` | 32 | 最大监视器数 |
| `CM_MAX_HISTORY` | 64 | 最大历史记录 |
| `AM_MAX_RULES` | 64 | 最大告警规则数 |
| `AM_MAX_ACTIVE_ALERTS` | 256 | 最大活跃告警数 |

---

## 5. 线程安全与并发

| 接口 | 线程安全 | 说明 |
|------|----------|------|
| `cm_init` / `cm_shutdown` | 否 | 初始化/关闭时单线程 |
| `cm_get` / `cm_get_int` 等 | 是 | 只读操作线程安全 |
| `cm_set` | 是 | 写操作线程安全 |
| `cm_watch` / `cm_unwatch` | 是 | 回调注册线程安全 |
| `am_fire` / `am_resolve` | 是 | 告警操作线程安全 |
| `am_evaluate_all` | 否 | 评估操作需串行 |

---

## 6. 相关文档

- [配置指南](../../Capital_Guides/configuration.md)
- [监控运维指南](../../Capital_Guides/monitoring.md)
- [Manager 集成标准](../../Capital_Specifications/integration_standards/INTEGRATION_STANDARD.md)