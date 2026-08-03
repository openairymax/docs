Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# 基于目标的相对逻辑准确判定协议 (GRAD)：计划级批判循环架构详解

> **文档定位**：基于目标的相对逻辑准确判定协议 (Goal-oriented Relative Accuracy Determination, GRAD) 架构详解\
> **最后更新**：2026-08-02\
> **上级文档**：[AirymaxAgentRT 文档中心](../README_zh.md)

***

## 1. 概述

**GRAD（Goal-oriented Relative Accuracy Determination，基于目标的相对逻辑准确判定协议）** 是 CoreLoopThree 认知管线中**计划级批判循环**的核心协议。在 Airymax 的落地形态中，批判循环的验证对象从"文本语义单元"升级为"DAG 计划（结构化图纸）"——这与 GCCP（目标完备确认，07-gccp.md）共同构成"事实-逻辑双锁决策模型"：

- **GCCP（事实层）**：处理"不确定性消除"——通过四问将无限可能映射为唯一客观目标 $G$。
- **GRAD（逻辑层）**：处理"计算冗余削减"——通过差分迭代与结构化验证，将错误的候选路径修剪至唯一可行的可达路径 $P^\*$。

GRAD 遵循 **机制/策略分离**：协议是机制（何时验证、如何判定、如何修正）；计划生成与语境终裁是策略（由模型 A/B 回调注入）。

> **理论依据**：GRAD 以计算复杂性理论（生成 NP 难 vs 验证 P）为不对称设计基础，以霍尔逻辑（因果充分性）、图论（偏序/无环性）、贝叶斯决策论（语境终裁）、信息论（差分熵削减）为四大数学支柱。

## 2. 设计决策：放弃文本批判循环，全面采用 GRAD

**决策（2026-08-02）**：放弃批判循环当前验证"文本"的方式，全面采用 GRAD——批判循环的验证对象是 **DAG 计划（结构化图纸）**，而非 LLM 生成的文本。

| 维度      | 原 TC3 文本批判循环                    | GRAD 计划级批判循环                           |
| ------- | ------------------------------- | -------------------------------------- |
| 验证对象    | 文本语义单元（su\_stream\_detector 切分） | DAG 计划（airy\_task\_plan\_t + GRAD 元数据） |
| 模型 C 验证 | LLM 对抗式批评（JSON 评分）              | 确定性四验（E-01\~E-04，零生成 Token）+ LLM 语义复核  |
| 收敛机制    | 修正文本直到评分达标                      | 增量补丁直到 Δ\_k → 0                        |
| 成本模型    | O(M × N) 全量文本重验                 | O(N + M·Δ\_max) 差分验证                   |
| 目标锚定    | 无（仅原始任务文本）                      | GCCP 五元组 G 强锚定（公理 I）                   |

## 3. 核心公理系统（Airymax 化）

| 公理编号       | 名称    | 表述                               | 落地映射                                                   |
| :--------- | :---- | :------------------------------- | :----------------------------------------------------- |
| **公理 I**   | 目标锚定性 | 计划 P 有效当且仅当 Exec(P,S₀)→G 的概率超过阈值 | GCCP 目标 G 注入 `airy_gccp_goal_t`，模型 A prompt 与 B 终裁均带 G |
| **公理 II**  | 相对判定性 | 判定函数 Φ(P)=f(P,H,R)，H 为历史语境       | 模型 B 终裁结合 G 与四验报告（贝叶斯先验）                               |
| **公理 III** | 差分收敛性 | 系统仅验证 Δ\_k 及其一阶闭包；Δ\_k→0 必然有限步停机 | `airy_grad_verify_scope()` 差分范围验证                      |

## 4. 三权分立架构（模型 A / B / C）

| 角色              | Airymax 槽位                  | 核心职能                          | Token 优化                                                   |
| :-------------- | :-------------------------- | :---------------------------- | :--------------------------------------------------------- |
| **模型 A（生成者）**   | t2（`tc3_s2_model`）          | 构造/修正 DAG 计划（Skeleton → 增量展开） | 首轮骨架，后续仅补丁 Δ\_k                                            |
| **模型 C（逻辑验证者）** | t1-p（`tc3_s1_expert_model`） | 确定性四验（E-01\~E-04）             | 仅读 inputs/outputs/cost/invariant\_guard；死锁用 Kahn 算法零 Token |
| **模型 B（语境终裁者）** | t1-f（`tc3_s1_verify_model`） | 结合 G 与 C 的报告签发最终核准            | 标量比较 + 余弦相似度，避免长文本拼接                                       |

> **模型自选**：t2/t1-f/t1-p 使用何种 LLM 由用户配置（`airy_cognition_set_tc3_models()` + 环境变量 `AIRY_MODEL_T2/T1F/T1P`），非固定厂商。模型 C 的"零生成式 Token"指**四验动作**本身是确定性算法；语义终裁仍可用 LLM。

## 5. 流程逻辑四验（模型 C 执行标准）

| 检验维度  | 代号       | 数学判定法则                    | 数据依赖                         | 异常             |
| :---- | :------- | :------------------------ | :--------------------------- | :------------- |
| 因果充分性 | **E-01** | Inputs(vᵢ) ⊆ ∪Outputs(前驱) | `inputs`/`outputs` 类型签名      | **逻辑断裂**       |
| 时间偏序性 | **E-02** | 子图无环（Kahn 拓扑排序，零 Token）   | 依赖邻接表                        | **死锁**         |
| 资源守恒性 | **E-03** | Σcost(vᵢ) ≤ R\_total（逐分量） | `cost_time_ms`/`cost_mem_mb` | **资源崩溃**       |
| 不变式保持 | **E-04** | invariant\_guard 节点不破坏 G  | `invariant_guard` 标签         | **目的漂移**（语义复核） |

**保守策略**：元数据未知（inputs/outputs 为 NULL、cost 为 0）时对应验项跳过，仅在元数据充分时严格判定。

## 6. 闭环运行时序（差分迭代算法）

```
Step 0  目标注入：GCCP 锁定的 G + 资源预算 R_total
Step 1  模型 A 生成骨架计划（或采用 Phase 1 seed plan）
Step 2  模型 C 四验骨架（round 0 全量验证）
Step 3  通过 → 模型 B 语境终裁
Step 4  通过 → 收敛停机（D_final）；驳回 → 构造补丁 Patch
Step 5  模型 A 按补丁修正 → 生成 Δ_k
Step 6  模型 C 差分四验（仅 Δ_k 及一阶闭包）
Step 7  循环直至收敛或达到 M_max
```

### 差分熵削减（V3.0 §2.5）

`airy_grad_verify_scope()` 接收上一轮驳回补丁的 `affected_scope`（变更节点 ID），计算**一阶闭包**（变更节点 + 依赖前驱 + 直接后继），仅对闭包内节点执行 E-01 严格检查。未变更节点视为"已证定理"，验证成本从 O(M×N) 降至 O(N + M·Δ\_max)。

## 7. 数据接口

### 7.1 计划 JSON（模型 A 输出，含 GRAD 元数据）

```json
{
  "nodes": [
    {"id":"S_01","goal":"定义需求","role":"creator","depends":[],
     "inputs":[],"outputs":["requirements_spec"],
     "cost_time_ms":500,"cost_mem_mb":16,"invariant_guard":false}
  ],
  "entry":["S_01"]
}
```

### 7.2 驳回补丁（模型 C/B 输出，GRAD §6.2）

```json
{
  "patch_id": "P_01", "source": "C_Model", "error_code": "E-01",
  "affected_scope": ["S_05"], "missing_artifact": "Compiled_Binary.exe",
  "suggestion": "insert a producer node or fix the input signature"
}
```

## 8. 双思考工作空间（决策链留痕）

所有工作有迹可循：每次 GRAD 执行在工作空间目录落盘原始文档与决策链日志。

```
$AIRY_RUNTIME_DIR/workspace/<plan_id>/
├── t2/plan_round_N.json      # 模型 A 各轮计划原始 JSON（原始文档保留）
├── c_verify/round_N.json     # 模型 C 四验报告
├── b_arbiter/round_N.json    # 模型 B 终裁意见
├── trace/chain.jsonl         # 决策链日志（JSONL，逐行追加）
└── goal.json                 # GCCP 目标快照
```

决策链 JSONL 每行格式：`{"event":"b_arbiter|s2_plan|c_verify|patch","ts":<ns>,"data":<payload>}`。

## 9. 公共 API

头文件：`agentrt/atoms/coreloopthree/src/cognition/critique/grad_verifier.h`、`grad_coordinator.h`、`grad_llm_callbacks.h`

| API                                      | 功能                             |
| ---------------------------------------- | ------------------------------ |
| `airy_grad_verify_plan()`                | 全量四验（E-01/E-02/E-03 + E-04 统计） |
| `airy_grad_verify_scope()`               | 差分四验（Δ\_k 及一阶闭包）               |
| `airy_grad_coordinator_create/destroy()` | GRAD 协调器生命周期                   |
| `airy_grad_coordinator_execute()`        | 计划级批判循环（含 seed plan 与收敛判定）     |
| `airy_grad_build_patch()`                | 构造驳回补丁（GRAD §6.2）              |
| `grad_llm_s2_plan()`                     | 模型 A 计划生成回调（LLM + 保守降级）        |
| `grad_llm_s1_arbiter()`                  | 模型 B 语境终裁回调（LLM + 保守采纳 C）      |
| `airy_cognition_set_grad_enabled()`      | 引擎级开关（cognition.h）             |

## 10. 在 CoreLoopThree 中的插接位置

```
Phase 0 拆解 → [GCCP 确认阶段] → Phase 1 规划 → [GRAD 计划级批判循环] →
Phase 2 执行（GRAD 启用时跳过文本级批判循环）→ Phase 3 审计 → Phase 4 目标对齐
```

- **插接行为**：Phase 1 规划产出 `airy_task_plan_t` 后，若 `enable_grad` 且 LLM 可用，创建 GRAD 协调器执行四验/终裁/修正；收敛后采用收敛计划。
- **Phase 2 降级**：GRAD 启用时，Phase 2 文本级批判循环（TC3 t2/t1-f/t1-p 文本循环）被跳过（`!enable_grad` 时回退，向后兼容）。
- **分级思考**：非任务集对话仅启用 t1-f 轻量验证（`AIRY_CHAT_T1F_VERIFY=1`）；任务集启用全量 GRAD。

## 11. 测试

`atoms/coreloopthree/tests/unit/test_grad.c`（8/8 通过，ASan 无泄漏）：

- E-02 死锁检测（Kahn 环检测）
- E-01 因果充分性（类型签名覆盖）
- E-03 资源守恒（成本预算）
- 协调器 seed 收敛（不修改 seed）
- 协调器驳回 → 模型 A 重新生成 → 收敛
- 驳回补丁构造（GRAD §6.2 格式）
- 差分范围验证（全量/闭包/非法节点降级）

## 12. 相关文档

- [目标完备确认协议 (GCCP)](07-gccp.md) — 事实锁定层（G 的来源）
- [认知循环运行时 (CoreLoopThree)](02-coreloopthree.md) — GRAD 所在认知管线
- [工作大厅 (Work Hall)](08-work-hall.md) — 收敛计划的执行宿主
- [双思考系统设计](../00-requirements/02-cognition-design-cn.md) — Thinkdual 认知架构

***

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
