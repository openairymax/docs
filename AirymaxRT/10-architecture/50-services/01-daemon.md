# Airymax Daemon 模块系统架构与使用指南
> **文档定位**：Airymax Daemon 模块系统架构与使用指南\
> **最后更新**：2026-06-09

---

## 模块概述

Airymax Daemon 模块是用户空间用户态服务集合，负责提供 MicroCoreRT 微核心架构中的高级服务功能。每个用户态服务作为独立进程运行，通过 Unix Socket 或 TCP 与其他组件通信。

---

## 服务清单

> **注**: 本文档 V7.0 原列 6 个核心服务。经源码审计确认实际存在 **12 个独立服务进程**（详见 TERMINOLOGY.md v3.0 裁决 #4，P4.8.2 补遗新增 hook_d / plugin_d）。以下为完整清单：

| 服务 | 端口 (TCP) | Unix Socket | 功能 |
|------|-----------|-------------|------|
| **gateway_d** | HTTP:8080 | 可选 | 统一入口网关，HTTP/WebSocket/Stdio → JSON-RPC 转换 |
| **llm_d** | — | /var/run/agentrt/llm.sock | 大语言模型推理服务，支持 OpenAI/Anthropic/Google/DeepSeek/本地模型 |
| **tool_d** | — | /var/run/agentrt/tool.sock | 工具注册、发现、执行管理，沙箱执行、超时控制、权限验证 |
| **market_d** | — | /var/run/agentrt/market.sock | Agent/Skill 应用市场，注册、搜索、安装、版本管理 |
| **sched_d** | — | /var/run/agentrt/sched.sock | 智能任务调度与负载均衡，轮询、加权、优先级队列、ML 调度 |
| **monit_d** | HTTP:9090 | /var/run/agentrt/monit.sock | 系统监控与告警，OpenTelemetry 兼容格式 |
| **channel_d** | — | /var/run/agentrt/channel.sock | 通道服务，跨组件通信通道管理 |
| **observe_d** | — | /var/run/agentrt/observe.sock | 观测服务，系统状态观测与数据采集 |
| **notify_d** | — | /var/run/agentrt/notify.sock | 通知服务，事件通知与消息推送 |
| **info_d** | — | /var/run/agentrt/info.sock | 信息服务，元数据查询与系统信息暴露 |
| **hook_d** | — | /var/run/agentrt/hook.sock | 全局 Hook 生命周期管理：注册、触发、卸载、审计；支持 8 种 Hook 类型（PRE/POST_EXEC、PRE/POST_LLM、PRE/POST_TOOL、ON_ERROR、ON_MEMORY_EVOLVE）；按优先级顺序执行，聚合决策（ABORT>RETRY>MODIFY>SKIP>CONTINUE）；PRE 阶段安全护栏前置拦截；委托 hook_registry/hook_executor/hook_interceptor/hook_timeout 模块化实现 |
| **plugin_d** | — | /var/run/agentrt/plugin.sock | 动态插件加载/卸载/生命周期管理：通过 dlopen/dlsym 加载动态库，状态机 UNLOADED→LOADED→INITIALIZED→RUNNING→ERROR/DISABLED；支持 4 种插件类型（TOOL_PROVIDER、PROTOCOL_ADAPTER、MEMORY_PROVIDER、HOOK_EXTENSION）；线程安全注册表（读写锁，最大 64 插件）；启动时自动扫描 ecosystem/plugins/ 经权限校验后加载 |

---

## 公共库组件

位于 `common/` 目录，供所有服务共享：

| 组件文件 | 功能 | 使用率 |
|---------|------|--------|
| jsonrpc_helpers.h/c | JSON-RPC 2.0 响应构建 | 100% |
| method_dispatcher.h/c | 方法路由分发器 | 67% (8/12) |
| param_validator.h/c | 参数验证工具集 | 67% (8/12) |
| log_sanitizer.h/c | 日志安全脱敏 | 按需 |
| svc_common.h | 公共类型和宏定义 | 100% |
| svc_logger.h/c | 统一日志接口 | 100% |
| platform.h | 跨平台抽象层 | 100% |

> **注**: hook_d 与 plugin_d 当前仅完成 IPC Bus 通道引导（`daemon_bootstrap_ipc_start`），未通过 `method_dispatcher_register` 注册具名 JSON-RPC 方法处理器，故 method_dispatcher / param_validator 使用率为 8/12。这两个 daemon 的本地 C 层服务 API（`hook_service_*` / `plugin_service_*`）已完整实现，RPC 方法暴露为后续补全项（P4.8.2 已记录）。

---

## 架构设计模式

### 1. Method Dispatcher Pattern（方法分发器模式）
- 替代 if-else 链式路由
- 降低圈复杂度从平均 14 → 4.2
- 支持运行时动态注册

### 2. Wrapper/Adapter Pattern（适配器模式）
- 将原有 handle_* 函数适配为统一 method_fn 签名
- 保持向后兼容性

### 3. Macro-Based Response（宏响应模式）
- JSONRPC_SEND_ERROR / JSONRPC_SEND_SUCCESS
- 消除重复的 build→send→free 三行组合

---

## 代码质量指标 (V7)

| 指标 | 当前值 | 目标 | 状态 |
|------|--------|------|------|
| 代码重复率 | < 1% | < 5% | ✅ |
| 平均圈复杂度 | 4.2 | < 5 | ✅ |
| 公共库集成率 | 96% | > 90% | ✅ |
| Doxygen 覆盖率 | 95% | - | ✅ |
| 静态分析 | 0 Critical, 0 Warning | 0 | ✅ |

---

## 编译与部署

```bash
# 构建所有服务
cd agentrt/daemon
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)

# 单独构建某个服务
make llm_d
make tool_d

# 安装到系统路径
sudo make install

# 启动服务（示例）
./llm_d --config /etc/agentrt/llm.conf
./tool_d -p 8081
```

---

## 测试

```bash
# 运行单元测试
ctest --output-on-failure

# 手动测试 JSON-RPC 接口
echo '{"jsonrpc":"2.0","method":"health_check","id":1}' \
  | socat - UNIX-CONNECT:/var/run/agentrt/llm.sock
```

---

## 维护指南

1. **添加新服务**: 复制现有服务模板，实现 handle_* 函数
2. **添加新方法**: 在 register_rpc_methods() 中调用 method_dispatcher_register()
3. **修改公共库**: 确保向后兼容，更新此文档
4. **性能优化**: 使用 param_validator 替代手动 cJSON 解析

---

## 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|---------|
| V8.0 | 2026-07-04 | P4.8.2 补遗：daemon 清单从 10 个修正为 12 个，新增 hook_d（全局 Hook 生命周期管理，8 种 Hook 类型）与 plugin_d（动态插件加载/卸载，4 种插件类型）的完整架构说明；公共库使用率同步更新为 8/12 |
| V7.0 | 2026-04-06 | 生产级优化：统一响应宏、参数验证集成、代码风格标准化 |
| V6.0 | 2026-04-05 | V6 全面检查通过 (S+/96.5分) |
| V5.0 | 2026-04-05 | param_validator + log_sanitizer 实现 |
| V4.0 | 2026-04-05 | method_dispatcher 100% 集成 |
| V3.0 | 2026-04-05 | jsonrpc_helpers 全面迁移 |
| V2.0 | 2026-04-04 | 初始重构版本 |
| V1.0 | 2026-04-03 | 原始版本 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.