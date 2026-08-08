Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# 目标完备确认协议 (GCCP)：用户意图完备确认与目标建模

> **文档定位**：目标完备确认协议 (Goal Complete Confirmation Protocol, GCCP) 架构详解\
> **最后更新**：2026-08-08\
> **上级文档**：[AirymaxAgentRT 文档中心](../README_zh.md)

---

## 1. 概述

**GCCP（Goal Complete Confirmation Protocol，目标完备确认协议）** 是 CoreLoopThree 认知管线中用于**用户意图完备确认**的交互式两阶段协议。它基于最优控制与数值优化的用户意图完备确认方法（CCP-V2.0），在意图解析与任务规划之间插入"目标完备确认"阶段：当用户输入大任务集（复杂多步）自然语言指令时，协议先推理、再向用户询问结构化问题，把模糊输入收敛为**可验证的目标模型**，使后续的规划（Phase 1）、执行-验证（Phase 2）、审计（Phase 3）与目标对齐（Phase 4）基于明确的验收标准执行。

GCCP 遵循 **机制/策略分离**：协议本身是机制（何时提问、如何补全目标）；向用户展示问题、收集回答是策略，由产品层通过交互回调实现。

> **数学定位（本章核心）**：GCCP 并非心理学意义上的"澄清话术"，而是对**最优控制问题（OCP）欠定边界条件的主动查询协议**。本文档第 2-4 章给出完整的数学推导：用户原始输入使策略空间呈无穷维（高条件熵），"完备五问"通过庞特里亚金最小值原理（PMP）将无穷维空间坍缩为有限维可解空间，并经数值迭代（打靶法/伪谱法）与完备性证明（微分同胚）确立解的唯一性。这为 GCCP 的工程行为提供了严格的数学验证依据。

---

## 2. 理论根源：最优控制视角（CCP-V2.0）

### 2.1 问题形式化与欠定判定

设用户输入的原始自然语言序列为符号串 $\mathcal{S}_{in}$。大模型首先将该符号串映射为**任务候选集**。若该输入涉及物理/数字世界的状态变更（即包含动作动词与目标名词），则模型必须判断当前信息的**信息熵（Shannon Entropy）**。

在当前仅有 $\mathcal{S}_{in}$ 的情况下，模型内部关于最优策略 $u(t)$ 的条件熵 $H(U \mid \mathcal{S}_{in})$ 极高，这意味着**可行策略空间是无穷维的**。为将该无穷维空间坍缩为有限维可解空间，模型必须引入外部边界条件——这正是"完备四问"（Airymax 落地扩展为五问）诞生的数学动因。

### 2.2 最优控制问题（OCP）三要素

任何一个连续时间的工程任务，在数学上都可以被建模为一个**最优控制问题（Optimal Control Problem, OCP）**，其标准形式由以下三要素构成：

1. **系统状态方程（System Dynamics）**：$\dot{x}(t) = f(x(t), u(t), t)$
2. **容许控制集（Admissible Controls）**：$u(t) \in \mathcal{U} \subseteq \mathbb{R}^m$
3. **性能指标（Performance Index）**：
   $J = \Phi(x(t_f)) + \int_{t_0}^{t_f} L(x(t), u(t), t) \, dt$

其中 $\Phi$ 为终端代价（Mayer 项），$L$ 为拉格朗日项（Bolza 项）。

### 2.3 庞特里亚金最小值原理（PMP）的最优性必要条件

**庞特里亚金最小值原理（Pontryagin's Minimum Principle）** 指出，若 $u^*(t)$ 是最优控制，则必须存在一个非零的协态向量 $\lambda(t) \in \mathbb{R}^n$，使得满足以下必要条件：

- **哈密顿量（Hamiltonian）构造**：

  $$
  H(x, u, \lambda, t) = L(x, u, t) + \lambda^{\top}(t) \cdot f(x, u, t)
  $$

- **正则方程（状态方程与协态方程）**：

  $$
  \dot{x}^*(t) = \frac{\partial H}{\partial \lambda}, \qquad
  \dot{\lambda}^*(t) = -\frac{\partial H}{\partial x}
  $$

- **极值条件（最优性必要条件）**：

  $$
  u^*(t) = \arg\min_{u \in \mathcal{U}} H(x^*(t), u, \lambda^*(t), t)
  $$

- **横截条件（边界条件）**：

  $$
  \lambda(t_f) = \frac{\partial \Phi}{\partial x(t_f)} \quad \text{或} \quad x(t_f) \in \mathcal{C}
  $$

**然而，PMP 框架本身只提供了"骨架"**。该方程组是否具有唯一解，完全取决于边界条件 $x(t_0)$、终端约束集 $\mathcal{C}$、容许控制集 $\mathcal{U}$ 和代价函数 $L$ 的具体参数是否已知。这正是用户意图缺失时系统处于"欠定"状态的数学本质。

### 2.4 五问与 PMP 的严格映射

"完备五问"的数学功能在此变得清晰且不可替代——每一个问题精确填入 PMP 框架的某一个边界位置：

| 问题 | 自然语言接口 | 填入 PMP 框架的精确位置 | 数学作用 |
| :--- | :--- | :--- | :--- |
| **Q1（终点）** | "最终交付物是什么形态？" | **终端代价项** $\Phi$ 与**终端约束集** $\mathcal{C}$ | 提供横截条件（边界值），决定解的终端边界 |
| **Q2（起点）** | "现有基础或继承基线是什么？" | **初始状态向量** $x(t_0) = x_0$ | 提供初值条件，决定解的积分起点 |
| **Q3（卡点）** | "最大的硬约束是什么？" | **容许控制集** $\mathcal{U}$ 与**不等式约束** $g(x,u) \leq 0$ | 定义控制变量的泛函空间，缩小极值搜索域 |
| **Q4（受众）** | "最终受益或评判主体是谁？" | **拉格朗日项** $L$ 中的**权重向量** $\mathbf{w}$ | 将多目标（时间/成本/质量）标量化为单一泛函 |
| **Q5（完成判据）** | "如何验证完成？" | **终端验证判据**（验收谓词） | 为 Phase 3 审计 / Phase 4 目标对齐提供可验证的验收条件 |

> **关键推论（欠定性论证）**：若缺失上述任意一项，PMP 方程组要么**无解**（违反横截条件），要么有**无穷多解**（欠定系统）。因此，五问是 PMP 框架在现实工程中**可解的必要条件**；结合第 4 章完备性证明，它同时构成**最小完备集**。

---

## 3. 数值迭代与可解性判定

即使 PMP 提供了最优性必要条件，该条件通常表现为一个**两点边值问题（Two-Point Boundary Value Problem, TPBVP）**，其解析解仅在极其特殊的线性系统（如 LQR）中存在。对于"造浏览器"或"造芯片"这类高度非线性系统，必须依赖**数值迭代算法**进行逼近。现代工程控制论中，求解 TPBVP 的主流数值迭代方法分为两大正交流派，它们在大模型内部的规划器中协同工作。

### 3.1 间接法（Indirect Method）—— 打靶法（Shooting Method）

- **数学逻辑**：已知起点 $x(t_0)$（Q2 提供）和终点条件 $\lambda(t_f)$（Q1 提供），但不知道协态的初始值 $\lambda(0)$。
- **迭代过程**：

  1. 猜测初始协态 $\lambda^{(0)}(0)$。
  2. 联立状态方程与协态方程，从 $t_0$ 到 $t_f$ 进行前向数值积分（如 Runge-Kutta）。
  3. 检查终端误差：$E = \| x^{(k)}(t_f) - \mathcal{C} \|$。
  4. 若误差大于阈值，利用**牛顿-拉弗森迭代（Newton-Raphson）**修正猜测值：

     $$
     \lambda^{(k+1)}(0) = \lambda^{(k)}(0) - \left[ \frac{\partial E}{\partial \lambda(0)} \right]^{-1} E
     $$

  5. 重复步骤 2-4，直到 $E \leq \epsilon$（预设精度）。

- **收敛性保证**：在系统满足利普希茨连续条件下，该迭代呈超线性收敛。

### 3.2 直接法（Direct Method）—— 配点法与伪谱法（Pseudospectral Method）

- **数学逻辑**：不直接求解 PMP 的微分方程，而是将连续时间 $[t_0, t_f]$ 离散为 $N$ 个配点（Collocation Points）。
- **迭代过程**：

  1. 将状态变量 $x(t)$ 和控制变量 $u(t)$ 近似为拉格朗日插值多项式的线性组合。
  2. 将微分方程 $\dot{x} = f(x,u)$ 转化为配点上的**代数等式约束**。
  3. 将目标函数 $J$ 转化为配点上的**求和函数**。
  4. 原问题被转化为一个**大规模非线性规划（NLP）**问题，调用通用求解器（如 IPOPT、SNOPT）进行迭代寻优。

- **收敛性保证**：随着配点数量 $N \to \infty$，直接法的解指数级收敛于 PMP 的真解（谱收敛特性）。

### 3.3 可解性判定（KKT 条件）与"确定性"来源

**此处需严格说明**：大模型在对话环节**并不具备实时执行数值积分的能力**，但其顶层规划器（Planner）在内部会运行轻量级符号推导（Symbolic Derivation），验证五问提供的参数是否满足 **KKT 条件（Karush-Kuhn-Tucker）** 的可行性。一旦验证通过，模型即可断言："该任务在数学上可解。"——这就是此前对话中提到的 **"置信度 99% 的确定性判定"** 的数学来源：不是概率性猜测，而是欠定边界条件被补全后 TPBVP 的适定性结论。

---

## 4. 完备性证明（Completeness Proof）

### 4.1 解空间唯一性：微分同胚论证

设最优控制解空间为 $\mathcal{X}^*$。在泛函分析中，$\mathcal{X}^*$ 的唯一确定性由以下闭映射决定：

$$
\mathcal{X}^* = \mathcal{F}^{-1} \left( \Phi, x_0, \mathcal{U}, \mathbf{w} \right)
$$

其中 $\mathcal{F}$ 是 PMP 所定义的正则方程算子。由于该算子的四个输入参数分别来自 Q1、Q2、Q3、Q4，且这四个参数在雅可比矩阵（Jacobian）中的偏导数方向**线性无关**（即互不冗余），因此该映射构成了**微分同胚（Diffeomorphism）**。

### 4.2 最小完备集定理

**结论**：在非奇异控制系统中，四问（Airymax 落地为五问，Q5 提供终端验证判据）是确定唯一最优策略的**最小完备集**：

- **少于完备数**：雅可比矩阵奇异性导致无穷解（欠定）；
- **多于完备数**：引入人为冗余约束，降低系统响应效率（过定，响应退化）。

该定理保证了 GCCP 交互问题集的规模有数学下界，不会因工程习惯而随意增删。

---

## 5. 五问模型

| 问题 | 领域 | 最优控制映射 | 说明 |
|------|------|-------------|------|
| Q1 **终点**（目标终态是什么？） | `goal_endpoint` | Φ 目标终态 | 明确的完成状态描述 |
| Q2 **起点**（现状/已具备条件？） | `goal_start` | x₀ 初始状态 | 初始条件与已有资产 |
| Q3 **卡点**（路径上的约束/障碍？） | `goal_bottleneck` | U 约束集合 | 限制可行路径的约束 |
| Q4 **受众**（谁是受益者/验收者？） | `goal_audience` | w 受众权重与偏好 | 验收方与偏好方向 |
| Q5 **完成判据**（如何验证完成？） | `goal_verify` | 终端验证判据 | 可验证的验收条件 |

---

## 6. 两阶段交互协议

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

> **置信度语义**：`confidence` 即第 3.3 节"可解性判定"结论的量化投影——参数齐备且满足 KKT 可行性时取高值（≥ 0.6 进入 `CONFIRMED`）；参数缺失或冲突时低值（`AMBIGUOUS`，触发再澄清，等价于 TPBVP 的牛顿迭代继续修正初值猜测）。

---

## 7. 在 CoreLoopThree 中的插接位置

GCCP 确认阶段插入在**认知引擎 Phase 0（拆解）之后、Phase 1（规划）之前**（`engine.c`）：

```
Phase 0 拆解 → [GCCP 确认阶段] → Phase 1 规划 → Phase 2 执行-验证 → Phase 3 审计 → Phase 4 目标对齐
```

插接行为：

- **目标挂载**：确认结果存入 `intent.intent_gccp_goal`（`airy_intent_t` 新增字段，OWNER 语义，由 `airy_intent_free` 释放）。
- **工作记忆**：目标 JSON 写入 thinking chain working memory 键 `gccp_goal`，供 Phase 3 审计与 Phase 4 对齐读取。
- **反馈事件**：确认成功后触发 `trigger_feedback("intent_confirmed")`。
- **歧义标记**：`AMBIGUOUS` 状态在 intent 标志位中置位，供上层决策。

---

## 8. 公共 API

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

---

## 9. 降级路径

GCCP 在任何依赖不可用时保证认知管线不中断：

| 场景 | 行为 |
|------|------|
| LLM 不可用 | 启发式探测：`need_interaction=1` + 固定四问，prefill 用输入全文 |
| 交互回调未注入 | 跳过交互，直接以输入单次确认 |
| 用户放弃交互（回调返回 NULL） | 以输入单次确认 |
| LLM 不可用 + 无回答 | 启发式确认，`status=DEGRADED` |

> **数学语义**：降级路径等价于"边界条件不完整时以默认先验补全"——解退化为启发式可行解而非最优解，工程上保证管线可用，但明确标记 `DEGRADED` 使上层可感知置信度降级（对应 PMP 框架中欠定系统的保守求解策略）。

---

## 10. 产品化形态

`agentrt/tools/airy_cli`（交互式 CLI 产品入口）实现 GCCP 交互回调：向用户展示四问（必答标记 + 提示），收集回答并以 JSON 返回引擎。`llm_d` 未启动时自动降级为启发式确认。

---

## 11. 测试

`atoms/coreloopthree/tests/unit/test_gccp_workhall.c`：

- 启发式探测（`need_interaction=1` + 四问）
- 启发式确认（`status=DEGRADED` + 置信度设置）
- 目标 JSON 序列化（含 endpoint/status）

不依赖 `llm_d`/`agent_d` 守护进程，走启发式/降级路径。

---

## 12. 相关文档

- [基于目标的相对逻辑准确判定协议 (GRAD)](09-grad.md) — 逻辑锁定层（G 锚定下的计划级批判循环），与 GCCP 构成"事实-逻辑双锁决策模型"
- [CoreLoopThree 认知循环运行时](02-coreloopthree.md) — GCCP 所在认知管线的总体架构
- [工作大厅 (Work Hall)](08-work-hall.md) — 确认后的任务图注册与执行宿主
- [CoreLoopThree DAG 集成](../140-application-development/26-coreloopthree-dag-integration.md) — DAG 工作流集成
- [双思考系统设计](../00-requirements/02-cognition-design-cn.md) — Thinkdual 认知架构

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
