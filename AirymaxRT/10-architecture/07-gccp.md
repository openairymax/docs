Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# 目标完备确认协议 (GCCP)：用户意图完备确认与目标建模
> **文档定位**：目标完备确认协议 (Goal Complete Confirmation Protocol, GCCP) 架构详解\
> **最后更新**：2026-08-02\
> **上级文档**：[AirymaxAgentRT 文档中心](../README_zh.md)

---

## 1. 概述

**GCCP（Goal Complete Confirmation Protocol，目标完备确认协议）** 是 CoreLoopThree 认知管线中用于**用户意图完备确认**的交互式两阶段协议。它基于最优控制与数值优化的用户意图完备确认方法（CCP-V2.0），在意图解析与任务规划之间插入"目标完备确认"阶段：当用户输入大任务集（复杂多步）自然语言指令时，协议先推理、再向用户询问结构化问题，把模糊输入收敛为**可验证的目标模型**，使后续的规划（Phase 1）、执行-验证（Phase 2）、审计（Phase 3）与目标对齐（Phase 4）基于明确的验收标准执行。

GCCP 遵循 **机制/策略分离**：协议本身是机制（何时提问、如何补全目标）；向用户展示问题、收集回答是策略，由产品层通过交互回调实现。

> **理论依据**：GCCP 的四问经庞特里亚金最小值原理（PMP）映射到最优控制问题——`Φ`（目标终态）与 `x₀`（初始状态）构成边界条件，`U`（约束集合）限定可行域，`w`（受众权重）决定代价泛函的方向，`Q5` 提供终端验证判据。

## 2. 五问模型

| 问题 | 领域 | 最优控制映射 | 说明 |
|------|------|-------------|------|
| Q1 **终点**（目标终态是什么？） | `goal_endpoint` | Φ 目标终态 | 明确的完成状态描述 |
| Q2 **起点**（现状/已具备条件？） | `goal_start` | x₀ 初始状态 | 初始条件与已有资产 |
| Q3 **卡点**（路径上的约束/障碍？） | `goal_bottleneck` | U 约束集合 | 限制可行路径的约束 |
| Q4 **受众**（谁是受益者/验收者？） | `goal_audience` | w 受众权重与偏好 | 验收方与偏好方向 |
| Q5 **完成判据**（如何验证完成？） | `goal_verify` | 终端验证判据 | 可验证的验收条件 |

## 3. 两阶段交互协议

```
用户输入（大任务集指令）
      │
      ▼
┌─────────────────────────────────────────────┐
│ Phase 1: airy_gccp_probe() — 目标探测        │
│   LLM 驱动：分析输入 → 产出初步目标 (prefill) │
│   与待询问问题集                              │
│   · 输入已完备 → need_interaction=0          │
│   · 输入不完备 → need_interaction=1（四问）    │
│   LLM 不可用 → 启发式降级（固定四问）          │
└──────────────────┬──────────────────────────┘
                   ▼
        ┌────────────────────┐
        │ 产品层交互回调       │  ← airy_gccp_interact_cb_t
        │ （CLI/UI 向用户提问）│
        └─────────┬──────────┘
                  ▼ 用户回答 JSON
┌─────────────────────────────────────────────┐
│ Phase 2: airy_gccp_confirm() — 目标确认       │
│   用户回答融合补全完整目标                     │
│   · 回答缺失 + LLM 可用 → 单次补全            │
│   · 两者皆不可用 → 启发式确认 (DEGRADED)      │
└──────────────────┬──────────────────────────┘
                   ▼
        airy_gccp_goal_t（五问 + 置信度 + 状态）
```

**状态机**：

| 状态 | 含义 | 触发条件 |
|------|------|----------|
| `CONFIRMED` | 已确认 | 四问 + Q5 完整，置信度 ≥ 0.6 |
| `DEGRADED` | 降级确认 | 无交互/LLM 不可用，启发式补全 |
| `AMBIGUOUS` | 歧义 | 置信度 < 0.6，需澄清（intent_flags |= 0x10） |

## 4. 在 CoreLoopThree 中的插接位置

GCCP 确认阶段插入在**认知引擎 Phase 0（拆解）之后、Phase 1（规划）之前**（`engine.c`）：

```
Phase 0 拆解 → [GCCP 确认阶段] → Phase 1 规划 → Phase 2 执行-验证 → Phase 3 审计 → Phase 4 目标对齐
```

插接行为：

- **目标挂载**：确认结果存入 `intent.intent_gccp_goal`（`airy_intent_t` 新增字段，OWNER 语义，由 `airy_intent_free` 释放）。
- **工作记忆**：目标 JSON 写入 thinking chain working memory 键 `gccp_goal`，供 Phase 3 审计与 Phase 4 对齐读取。
- **反馈事件**：确认成功后触发 `trigger_feedback("intent_confirmed")`。
- **歧义标记**：`AMBIGUOUS` 状态在 intent 标志位中置位，供上层决策。

## 5. 公共 API

头文件：`agentrt/atoms/coreloopthree/include/gccp.h`（版本 `AIRY_GCCP_VERSION "1.0.0"`）

| API | 功能 |
|-----|------|
| `airy_gccp_probe()` | 阶段一：目标探测（推理产出 prefill + 问题集） |
| `airy_gccp_confirm()` | 阶段二：目标确认（回答融合补全） |
| `airy_gccp_probe_free()` | 释放探测结果 |
| `airy_gccp_goal_free()` | 释放目标结构 |
| `airy_gccp_status_str()` | 状态可读描述 |
| `airy_gccp_goal_to_json()` | 目标序列化为 JSON |

**核心数据结构**：

- `airy_gccp_goal_t` — 五问字段（`goal_endpoint/goal_start/goal_bottleneck/goal_audience/goal_verify`）+ `confidence` + `status` + `raw_prompt`。
- `airy_gccp_question_t` — 面向用户的问题（`id/question/hint/required`）。
- `airy_gccp_probe_t` — 探测结果（`prefill/questions/question_count/need_interaction`）。
- `airy_gccp_interact_cb_t` — 产品层交互回调，返回回答 JSON（key 与问题 ID 对应）。

**认知引擎 setter**（`cognition.h`）：

- `airy_cognition_set_gccp_enabled()` — 启用/禁用 GCCP 确认阶段。
- `airy_cognition_set_gccp_interact()` — 注入产品层交互回调。

## 6. 降级路径

GCCP 在任何依赖不可用时保证认知管线不中断：

| 场景 | 行为 |
|------|------|
| LLM 不可用 | 启发式探测：`need_interaction=1` + 固定四问，prefill 用输入全文 |
| 交互回调未注入 | 跳过交互，直接以输入单次确认 |
| 用户放弃交互（回调返回 NULL） | 以输入单次确认 |
| LLM 不可用 + 无回答 | 启发式确认，`status=DEGRADED` |

## 7. 产品化形态

`agentrt/tools/airy_cli`（交互式 CLI 产品入口）实现 GCCP 交互回调：向用户展示四问（必答标记 + 提示），收集回答并以 JSON 返回引擎。`llm_d` 未启动时自动降级为启发式确认。

## 8. 测试

`atoms/coreloopthree/tests/unit/test_gccp_workhall.c`：

- 启发式探测（`need_interaction=1` + 四问）
- 启发式确认（`status=DEGRADED` + 置信度设置）
- 目标 JSON 序列化（含 endpoint/status）

不依赖 `llm_d`/`agent_d` 守护进程，走启发式/降级路径。

## 9. 相关文档

- [CoreLoopThree 认知循环运行时](02-coreloopthree.md) — GCCP 所在认知管线的总体架构
- [工作大厅 (Work Hall)](08-work-hall.md) — 确认后的任务图注册与执行宿主
- [CoreLoopThree DAG 集成](../140-application-development/26-coreloopthree-dag-integration.md) — DAG 工作流集成
- [双思考系统设计](../00-requirements/02-cognition-design-cn.md) — Thinkdual 认知架构
- [基于目标的相对逻辑准确判定协议 (GRAD)](09-grad.md) — 逻辑锁定层（G 锚定下的计划级批判循环）

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
