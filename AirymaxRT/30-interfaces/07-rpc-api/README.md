# Airymax Daemon JSON-RPC API 规范
> **文档定位**：Airymax Daemon JSON-RPC API 规范\
> **文档版本**：v0.1.0-draft | **状态**: 草案 | **日期**: 2026-07-04\
> **版权归属人**：SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）

---

## 1. 现状问题（已验证）

### 1.1 扁平方法名 + 跨 daemon 冲突（CONFIRMED）

实际注册的方法名（无命名空间前缀）：

| Daemon | 方法名 | 冲突 |
|--------|--------|------|
| llm_d | `complete`、`complete_stream` | — |
| tool_d | `register`、`list_tools`、`get_tool`、`execute_tool` | — |
| sched_d | `register_agent`、`schedule_task`、`get_stats`、`health_check` | ⚠️ 与 market_d 冲突 |
| market_d | `register_agent`、`search_agents`、`install_agent`、`register_skill`、`search_skills`、`health_check` | ⚠️ 与 sched_d 冲突 |
| monit_d | `record_metric`、`get_metrics`、`trigger_alert`、`get_alerts`、`health_check`、`generate_report` | ⚠️ 与 sched_d/market_d 冲突 |
| gateway_d | `gateway.forward` | ✅ 唯一规范命名 |

### 1.2 README 示例与代码不一致
- `daemon/common/README.md:383-384` 示例 `method_dispatcher_register(disp, "llm.generate", ...)`
- 但 `llm_d/src/main.c:657-658` 实际注册 `"complete"`（无 `llm.` 前缀）

### 1.3 隐式命名空间
- `svc_common.c:2117-2118` 运行时 `snprintf(rpc_method, ..., "%s.%s", service_name, method)` 拼接
- 非显式注册，第三方无法从规范文档推导实际方法名

## 2. 标准规范

### 2.1 强制命名空间（ARE-RPC-01）

所有 JSON-RPC 方法必须采用 `<daemon>.<method>` 格式：

| Daemon | 命名空间 | 方法示例 |
|--------|---------|---------|
| llm_d | `llm.` | `llm.complete`、`llm.complete_stream`、`llm.list_models` |
| tool_d | `tool.` | `tool.register`、`tool.list`、`tool.get`、`tool.execute` |
| sched_d | `sched.` | `sched.register_agent`、`sched.schedule_task`、`sched.get_stats`、`sched.health_check` |
| market_d | `market.` | `market.register_agent`、`market.search_agents`、`market.install_agent`、`market.register_skill`、`market.search_skills`、`market.health_check` |
| monit_d | `monit.` | `monit.record_metric`、`monit.get_metrics`、`monit.trigger_alert`、`monit.get_alerts`、`monit.health_check`、`monit.generate_report` |
| gateway_d | `gateway.` | `gateway.forward`（已规范） |
| hook_d | `hook.` | `hook.register`、`hook.trigger`、`hook.list` |
| channel_d | `channel.` | `channel.send`、`channel.receive`、`channel.list` |
| notify_d | `notify.` | `notify.send`、`notify.subscribe`、`notify.unsubscribe` |
| observe_d | `observe.` | `observe.start`、`observe.stop`、`observe.get_trace` |
| info_d | `info.` | `info.get_version`、`info.get_config`、`info.health_check` |
| plugin_d | `plugin.` | `plugin.load`、`plugin.unload`、`plugin.list` |

### 2.2 跨 daemon 共享方法处理

`health_check` 等通用方法允许跨 daemon 重复，但必须带命名空间前缀（`sched.health_check` ≠ `market.health_check`）。

### 2.3 显式注册规范（ARE-RPC-02）

禁止使用 `snprintf` 运行时拼接命名空间。所有方法必须显式注册完整命名空间名：

```c
// 正确
method_dispatcher_register(disp, "llm.complete", handler_complete);
method_dispatcher_register(disp, "llm.complete_stream", handler_stream);

// 禁止
method_dispatcher_register(disp, "complete", handler_complete);
// 运行时 snprintf 拼接 "llm.complete"
```

### 2.4 第三方实现指南

本文档定义每个 daemon 的：
- 方法名（含命名空间）
- 请求参数（JSON Schema）
- 响应格式（JSON Schema）
- 错误码

第三方可基于本文档独立实现兼容 daemon。

## 3. 修复任务

详见 [0.1.1详细任务清单](../../../docs-closed/agentrt/_design_0.1.1/03-detailed-task-list.md) P0.24（JSON-RPC 命名空间规范，16h）

---

<div align="center">

**© 2025-2026 SPHARX Ltd.**

</div>
