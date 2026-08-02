Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# 工作大厅 (Work Hall)：任务图注册 / 状态看板 / 查询 / 取消
> **文档定位**：工作大厅 (Work Hall) 机制架构详解\
> **最后更新**：2026-08-02\
> **上级文档**：[AirymaxAgentRT 文档中心](../README_zh.md)

---

## 1. 概述

**工作大厅（Work Hall）** 是任务图的宿主与执行中枢：认知引擎产出的任务计划（`airy_task_plan_t`）经 **Plan→TaskFlow DAG 适配层** 转换为 `taskflow_workflow_t` 后在此注册、提交执行。大厅对外提供统一看板（execution 状态/进度）、查询、取消、等待与 orchestration ops 注入（`airy_orch_ops_t`），使行动层 agents 按工作大厅中的任务图工作。

工作大厅遵循 **机制/策略分离**：大厅是机制（任务图执行与状态管理）；任务图内容与 handler 绑定是策略（由 plan 适配层与调用方注入）。

## 2. 架构位置

```
用户大任务指令
      │
      ▼
┌───────────────────────┐
│ 认知管线（CoreLoopThree）│
│  Phase 0 拆解           │
│  [GCCP 意图完备确认]     │  ← 07-gccp.md
│  Phase 1 规划           │  → airy_task_plan_t
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Plan→DAG 适配层         │
│  airy_plan_to_workflow │  → taskflow_workflow_t
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 工作大厅 (Work Hall)    │  ★ 本组件 ★
│  注册 / 看板 / 取消 / 等待│
│  ops 注入 (airy_orch_ops_t)│
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 行动层 agents           │
│  agent_d spawn/invoke  │  → ecosystem/agents 真实执行
└───────────────────────┘
```

## 3. 核心职责

| 职责 | 说明 |
|------|------|
| **任务图注册** | 提交 workflow 时自动注册未绑定的节点 handler |
| **状态看板** | 执行实例快照（state/progress/时间戳），支持查询与列表 |
| **执行控制** | 取消执行实例、等待完成并获取结果 |
| **Agent 驱动** | 节点 handler 路由到 `agent_d`（daemon_rpc_call spawn/invoke），真正驱动 `ecosystem/agents` 下的 Python/Rust agents |
| **Ops 注入** | 实现 `airy_orch_ops_t`（schedule/cancel/list/get_status/wait）并注入全局 ops_injection 表 |

## 4. 公共 API

头文件：`agentrt/atoms/coreloopthree/include/work_hall.h`（版本 `AIRY_WORK_HALL_VERSION "1.0.0"`）

| API | 功能 | 线程安全 |
|-----|------|----------|
| `airy_work_hall_create()` | 创建大厅（绑定 core loop） | 否 |
| `airy_work_hall_destroy()` | 销毁大厅 | 否 |
| `airy_work_hall_submit()` | 提交工作流执行（自动注册 handler + 登记看板） | 是 |
| `airy_work_hall_status()` | 查询执行状态（看板） | 是 |
| `airy_work_hall_list()` | 列出全部执行实例 | 是 |
| `airy_work_hall_cancel()` | 取消执行实例 | 是 |
| `airy_work_hall_wait()` | 等待完成并获取结果 | 是 |
| `airy_work_hall_bind_ops()` | 注入 orchestration ops 表 | — |
| `airy_work_hall_agent_id()` | 角色 → agent_id（自动 spawn 缓存） | 是 |
| `airy_work_hall_entry_free()` / `list_free()` | 释放看板条目（数组） | — |

**核心数据结构**：

- `airy_work_hall_config_t` — `max_concurrent`（默认 8）、`default_timeout_ms`（默认 30000）、`agent_d_socket`（agent_d Unix socket 路径）。
- `airy_work_hall_entry_t` — 看板条目快照：`execution_id/workflow_id/workflow_name/state/progress/started_at/completed_at/task_id`。
- 状态取值：`pending/ready/running/waiting/completed/failed/canceled/skipped/retrying`。

## 5. Plan → TaskFlow DAG 适配层

头文件：`agentrt/atoms/coreloopthree/include/plan_to_dag.h`

| API | 功能 |
|-----|------|
| `airy_plan_to_workflow()` | 任务计划 → 工作流定义（节点/边/入口转换） |
| `airy_workflow_free()` | 释放适配层生成的工作流副本 |

**转换规则**：

- 计划节点 → 工作流节点（`type=TASK`，`handler=task_node_handler_name`）
- 节点依赖（`depends_on`）→ 工作流边（source=依赖 ID，target=节点 ID）
- 入口点（`entry_points`）→ `initial_node_id`
- **handler 规范化**：节点 handler 无 `agent:` 前缀时自动补全，确保所有任务节点由工作大厅的 agent 路由 handler 接管

## 6. Agent 驱动机制

工作大厅提交工作流时，为 `task_handler_name` 尚未注册的节点自动绑定 agent 路由 handler（`user_data` 含 role 上下文）：

```
workflow 节点（handler="agent:coding"）
      │
      ▼ wh_agent_handler
┌──────────────────────────┐
│ daemon_rpc_call spawn     │ → agent_d 启动 coding agent
│ daemon_rpc_call invoke    │ → 向 agent 下发任务/上下文
└──────────────────────────┘
      │
      ▼
ecosystem/agents 下真实执行（Python/Rust）
```

- `wh_register_handlers` 仅接管 `agent:` 前缀 handler，不影响其他预注册 handler。
- `agent_d` 不可用时，`airy_work_hall_agent_id` 返回 `AIRY_EUNAVAILABLE`，执行走降级路径。

## 7. 与 CoreLoopThree 的接线

- **执行引擎**：复用 loop 内 taskflow 引擎（`airy_loop_submit_dag` 等）。
- **取消扩展**：`airy_loop_dag_cancel()`（`loop.h` 新增）将取消请求转发至 `taskflow_engine_cancel()`，供大厅取消执行实例。
- **Ops 注入**：`airy_work_hall_bind_ops()` 实现 `airy_orch_ops_t`（schedule/cancel/list/get_status/wait）并注入全局 ops_injection 表，接通此前无调用点的 `airy_orch_ops_t` 断链。

## 8. 产品化形态

`agentrt/tools/airy_cli`（交互式 CLI 产品入口）完成完整闭环：

```
用户自然语言大任务指令
  → GCCP 意图完备确认（推理 + 四问）
  → 认知管线规划（Phase 0-1）
  → Plan → TaskFlow DAG 适配
  → 工作大厅提交/看板轮询/结果获取
  → agent_d 驱动 ecosystem/agents 真实执行
```

## 9. 测试

`atoms/coreloopthree/tests/unit/test_gccp_workhall.c`（6/6 通过，ASAN 复验无 double-free/UAF）：

- 工作大厅基础闭环：create → 注册 handler → submit → wait → status
- 工作大厅列表与取消：list 非空；cancel 更新看板状态

## 10. 相关文档

- [CoreLoopThree 认知循环运行时](02-coreloopthree.md) — 大厅宿主所在认知管线
- [目标完备确认协议 (GCCP)](07-gccp.md) — 提交任务图前的意图完备确认
- [CoreLoopThree DAG 集成](../140-application-development/26-coreloopthree-dag-integration.md) — DAG 工作流集成指南

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
