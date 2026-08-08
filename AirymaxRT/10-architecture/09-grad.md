Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# 基于目标的相对逻辑准确判定协议 (GRAD)：计划级批判循环架构详解

> **文档定位**：基于目标的相对逻辑准确判定协议 (Goal-oriented Relative Accuracy Determination, GRAD) 架构详解\
> **最后更新**：2026-08-08\
> **上级文档**：[AirymaxAgentRT 文档中心](../README_zh.md)

***

## 1. 概述

**GRAD（Goal-oriented Relative Accuracy Determination，基于目标的相对逻辑准确判定协议）** 是 CoreLoopThree 认知管线中**计划级批判循环**的核心协议。在 Airymax 的落地形态中，批判循环的验证对象从"文本语义单元"升级为"DAG 计划（结构化图纸）"——这与 GCCP（目标完备确认，07-gccp.md）共同构成"事实-逻辑双锁决策模型"：

- **GCCP（事实层）**：处理"不确定性消除"——通过四问（落地五问）将无限可能映射为唯一客观目标 $G$。
- **GRAD（逻辑层）**：处理"计算冗余削减"——通过差分迭代与结构化验证，将错误的候选路径修剪至唯一可行的可达路径 $P^\*$。

GRAD 遵循 **机制/策略分离**：协议是机制（何时验证、如何判定、如何修正）；计划生成与语境终裁是策略（由模型 A/B 回调注入）。

> **数学定位（本章核心）**：GRAD V3.0 以四大数学支柱为设计依据——**计算复杂性理论**（生成 NP 难 vs 验证 P，非对称设计）、**霍尔逻辑**（因果充分性）、**图论**（偏序/无环性/良基性收敛）、**贝叶斯决策论**（语境终裁）、**信息论**（差分熵削减）。其核心升级为**差分熵削减机制（Differential Entropy Reduction）**，将多轮迭代的总验证成本从 $O(M \times N)$（$M$ 为迭代轮次，$N$ 为单次完整图纸规模）重构为 $O(N + \sum_{k=1}^{M} |\Delta_k|)$，其中 $|\Delta_k| \ll N$ 为第 $k$ 轮增量补丁的规模。第 10 章给出完整的停机与成本收敛定理及构造性证明。

***

## 2. 设计决策：放弃文本批判循环，全面采用 GRAD

**决策（2026-08-02）**：放弃批判循环当前验证"文本"的方式，全面采用 GRAD——批判循环的验证对象是 **DAG 计划（结构化图纸）**，而非 LLM 生成的文本。

| 维度      | 原 TC3 文本批判循环                    | GRAD 计划级批判循环                           |
| ------- | ------------------------------- | -------------------------------------- |
| 验证对象    | 文本语义单元（su\_stream\_detector 切分） | DAG 计划（airy\_task\_plan\_t + GRAD 元数据） |
| 模型 C 验证 | LLM 对抗式批评（JSON 评分）              | 确定性四验（E-01\~E-04，零生成 Token）+ LLM 语义复核  |
| 收敛机制    | 修正文本直到评分达标                      | 增量补丁直到 Δ\_k → 0                        |
| 成本模型    | O(M × N) 全量文本重验                 | O(N + M·Δ\_max) 差分验证                   |
| 目标锚定    | 无（仅原始任务文本）                      | GCCP 五元组 G 强锚定（公理 I）                   |

***

## 3. 核心公理系统（V3.0 形式化定义）

GRAD V3.0 以三条公理为形式化基础，每条公理给出数学表述与 Airymax 落地映射：

| 公理编号 | 公理名称 | 数学表述 | Airymax 落地映射 |
| :--- | :--- | :--- | :--- |
| **公理 I** | **目标锚定性** | 设系统终态为 $G$，流程 $P$ 有效当且仅当 $\Pr\left(\text{Exec}(P, S_0) \to G\right) > \theta$，其中 $\theta$ 为预设置信阈值（$0 < \theta < 1$）。 | GCCP 目标 $G$ 注入 `airy_gccp_goal_t`，模型 A prompt 与 B 终裁均带 $G$；为收敛提供边界条件，禁止优化过程中的逻辑漂移。 |
| **公理 II** | **相对判定性** | 判定函数 $\Phi(P) = f(P, H, R)$，其中 $H$ 为历史语境（信息集），$R$ 为资源约束向量。 | 模型 B 终裁结合 $G$ 与四验报告（贝叶斯先验）；$R$ 对应资源预算向量 `R_total`。 |
| **公理 III** | **差分收敛性** | 设第 $k$ 轮图纸为 $D_k$，补丁为 $\Delta_k = D_k \setminus D_{k-1}$（对称差集）。系统仅需验证 $\Delta_k$ 及其一阶逻辑闭包（相邻节点）。若 $\lim_{k \to \infty} \|\Delta_k\| = 0$，则系统必然在有限步内停机。 | `airy_grad_verify_scope()` 差分范围验证；未变更节点视为"已证定理"，免除重复推导。 |

***

## 4. 数学基础：四大理论支柱

### 4.1 计算复杂性理论（生成 ≠ 验证）

- **数学依据**：在计算复杂性中，"生成"一个可行解（搜索解空间）通常属于 **NP 难（NP-hard）** 或 **PSPACE** 问题，而"验证"一个给定解是否正确属于 **P** 或 **co-NP** 问题。
- **GRAD 映射**：模型 A（大参数）承担生成任务（指数级发散）；模型 C（中参数）承担验证任务（多项式级收敛）。这种非对称设计在数学上被证明能最大化系统可靠性，因为验证算法天然比生成算法更容易逼近真值。

### 4.2 霍尔逻辑（Hoare Logic，用于因果充分性）

- **数学定义**：对于流程中的任意步骤 $S$，定义霍尔三元组为 $\{P\} \ S \ \{Q\}$，其中 $P$ 为前置条件，$Q$ 为后置条件。
- **判定准则**：逻辑正确的充要条件是 $P \Rightarrow Q$（前置条件必须强到足以推导出后置条件）。
- **GRAD 应用**：模型 C 检查 DAG 中相邻步骤的输入输出（Artifacts），若 $\text{Output}(S_i) \not\subseteq \text{Input}(S_{i+1})$，则判定为逻辑断裂（E-01）。

### 4.3 图论（偏序集与有向无环图）

- **数学依据**：任务流程必须是一个 **DAG（有向无环图）**，对应数学上的**偏序集（Poset）**。
- **判定准则**：若图中存在环 $\text{Cycle}$，则系统死锁（Deadlock）。根据**拓扑排序定理**，一个 DAG 必然存在至少一个拓扑序。若模型 C 无法对当前图纸执行拓扑排序（Kahn 算法），则判定为逻辑死锁（E-02）。
- **收敛性证明（良基性）**：每一次修正在数学上对应向 DAG 中添加一条"覆盖关系（Covering Relation）"的边，这严格减少了偏序集中"未覆盖元素对"的数量，保证修正过程在有限步内终止。

### 4.4 贝叶斯决策论（语境终裁的数学本质）

- **数学依据**：设用户真实隐性需求为 $T$，模型 B 拥有观测证据集 $D$（即历史对话）。模型 B 的终裁等价于计算后验概率 $P(T | D)$。
- **判定准则**：模型 C 给出的硬逻辑判断 $L$ 只是一个先验信号。模型 B 将 $L$ 与自身的后验概率 $P(T|D)$ 进行加权融合。当融合后的置信度低于预设阈值（如 0.6）时，系统强制驳回。

### 4.5 信息论基础（差分熵与冗余消除）

- **定义**：设完整图纸的熵为 $H(D)$。在迭代修正中，相邻两轮图纸的互信息为 $I(D_k; D_{k-1})$。
- **优化依据**：当系统趋于逻辑收敛时，条件熵 $H(D_k \mid D_{k-1}) \to 0$。这意味着增量补丁 $\Delta_k$ 的信息量趋近于零。
- **推论**：强制验证器每轮读取全量 $D_k$ 在信息论上是冗余的。V3.0 规定验证器仅处理条件熵所代表的新增信息子集，从而在数学上消除重复计算（对应第 8 章差分熵削减机制）。

***

## 5. 三权分立架构（模型 A / B / C）

| 角色 | Airymax 槽位 | 参数量级特征 | 核心职能 | 数学角色映射 | Token 优化策略 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **模型 A（生成者）** | t2（`tc3_s2_model`） | 超大（高推理成本） | **构造性规划**。接收 GCCP 锁定的目标 $G$ 和驳回补丁，构造/修正初始 DAG。 | **解空间内的启发式搜索函数** $\mathcal{G}(S_0, G) \to \text{DAG}$ | **分层生成 + 增量输出**。首轮仅输出骨架层（< 10% 全量）；后续仅输出补丁 $\Delta_k$，禁止全量重绘。 |
| **模型 C（逻辑验证者）** | t1-p（`tc3_s1_expert_model`） | 中大（中等推理成本） | **硬性逻辑筛查**。基于"流程逻辑四验"进行静态推导，剔除结构上必然不通的方案。 | **形式化证明检查器**（检查霍尔三元组与 DAG 可达性） | **结构化截断验证**。仅读取 `inputs`/`outputs`/`cost` 字段；死锁检查交由确定性算法（Kahn 拓扑排序），零生成式 Token 消耗。 |
| **模型 B（语境终裁者）** | t1-f（`tc3_s1_verify_model`） | 中等，高频交互 | **软性语境仲裁**。结合长期对话历史，对 C 的判决进行终审，签发最终核准意见。 | **贝叶斯元验证器** $\text{Judgement} = \arg\max P(\text{Valid} \mid \text{Logic}, \text{History})$ | **向量化终裁**。历史对话预编码为固定 Embedding；终裁时仅做标量比较与余弦相似度计算，避免长文本拼接。 |

> **模型自选**：t2/t1-f/t1-p 使用何种 LLM 由用户配置（`airy_cognition_set_tc3_models()` + 环境变量 `AIRY_MODEL_T2/T1F/T1P`），非固定厂商。模型 C 的"零生成式 Token"指**四验动作**本身是确定性算法；语义终裁仍可用 LLM。

***

## 6. 流程逻辑四验（模型 C 执行标准）

模型 C 必须对 DAG（或增量补丁）中的相关节点执行以下四项严格的数学检查。**所有检查仅依赖结构化元数据，不依赖自然语言描述**。

| 检验维度 | 命名代号 | 数学判定法则 | 数据依赖（Token 轻量化） | 异常触发条件 |
| :--- | :--- | :--- | :--- | :--- |
| **因果充分性** | **E-01** | $\forall v_i \in \text{scope},\ \text{Inputs}(v_i) \subseteq \bigcup_{j < i} \text{Outputs}(v_j)$ | 仅读取 `inputs`/`outputs` 类型签名数组 | 若缺失依赖，报错 **E-01（逻辑断裂）** |
| **时间偏序性** | **E-02** | 子图 $G' = (V', E')$ 必须满足**无环性**。由 **Kahn 拓扑排序算法**判定，零生成式 Token 消耗。 | 仅读取 `step_id` 与边关系邻接表 | 若存在环，报错 **E-02（死锁）** |
| **资源守恒性** | **E-03** | $\sum_{i \in \text{scope}} \text{cost}(v_i) \leq R_{\text{total}}$（逐分量比较） | 仅读取 `cost` 数值字段（`cost_time_ms`/`cost_mem_mb`） | 若任一维度超限，报错 **E-03（资源崩溃）** |
| **不变式保持** | **E-04** | 设 $I(G)$ 为目标 $G$ 的关键特征谓词。必须证明 $\forall v_i \in \text{scope},\ \text{Execute}(v_i) \to I(G)$ 成立。 | 仅读取节点预设的 `invariant_guard` 布尔标签 | 若某步骤会破坏 $G$ 的定义，报错 **E-04（目的漂移）**（语义复核） |

**保守策略**：元数据未知（inputs/outputs 为 NULL、cost 为 0）时对应验项跳过，仅在元数据充分时严格判定。

***

## 7. 闭环运行时序（差分迭代算法）

系统严格遵循以下步骤，其中 $k$ 为迭代计数器：

- **Step 0（目标注入）**：输入 GCCP 锁定的目标 $G$ 及全量预算向量 $R_{\text{total}}$（或采用 Phase 1 seed plan）。
- **Step 1（骨架生成）**：模型 A 生成骨架计划 $\text{Skeleton}_0$。此阶段仅输出宏观元节点（如"环境搭建"、"编译"），消耗代价 $\alpha \cdot \mathcal{C}(N)$，其中 $0 < \alpha \ll 1$，$N$ 为全量图纸规模。
- **Step 2（骨架验证）**：模型 C 对骨架执行"验 I（因果）"和"验 II（偏序）"。若失败，生成结构补丁 $\text{Patch}_C^0$，跳转 Step 5；若通过，进入 Step 3。
- **Step 3（细节增量展开）**：模型 A 按拓扑序展开一个未完成的元节点，生成补丁 $\Delta_k$，其中 $|\Delta_k| \ll N$。
- **Step 4（增量逻辑验证）**：模型 C 仅读取 $\Delta_k$ 及其前后各 2 层邻居节点，执行完整四验。若失败，生成结构补丁 $\text{Patch}_C^k$。
- **Step 5（语境终裁与驳回）**：模型 B 接收当前补丁，结合缓存的用户 Embedding 进行终裁。若发现资源超限或习惯冲突，生成语境补丁 $\text{Patch}_B^k$。
- **Step 6（修正迭代）**：模型 A 接收 $\text{Patch}_C^k$ 或 $\text{Patch}_B^k$，仅修改受影响节点，生成 $\Delta_{k+1}$。令 $k = k+1$，返回 Step 3。
- **Step 7（终止条件）**：当所有元节点展开完毕，且模型 C 与模型 B 均签发"通过"时，系统停机，输出 $D_{\text{final}}$。

***

## 8. 差分熵削减机制（V3.0 核心）

`airy_grad_verify_scope()` 接收上一轮驳回补丁的 `affected_scope`（变更节点 ID），计算**一阶闭包**（变更节点 + 依赖前驱 + 直接后继），仅对闭包内节点执行 E-01 严格检查。未变更节点视为"已证定理"。

- **信息论依据**：当系统趋于逻辑收敛时，条件熵 $H(D_k \mid D_{k-1}) \to 0$，增量补丁 $\Delta_k$ 的信息量趋近于零；验证器仅处理条件熵所代表的新增信息子集，从数学上消除重复计算。
- **成本模型**：验证成本从 $O(M \times N)$（全量重验）降至 $O(N + M \cdot \Delta_{\max})$，其中 $\Delta_{\max} = \max_k |\Delta_k| \ll N$。

***

## 9. 数据接口

### 9.1 增量补丁（Delta Patch）规范

模型 A 严禁输出全量 JSON，必须输出以下格式（`full_snapshot_omitted: true` 表明差分协议生效）：

```json
{
  "base_version_hash": "a1b2c3d4e5",
  "iteration_round": 2,
  "patches": [
    {
      "op": "add",
      "node": {
        "step_id": "S_07",
        "inputs": ["Compiled_Binary.exe"],
        "outputs": ["Installer.msi"],
        "cost": {"time_min": 15, "memory_GB": 4},
        "invariant_guard": true
      },
      "dependencies": ["S_04"]
    },
    {
      "op": "modify",
      "target_id": "S_03",
      "delta_content": {
        "cost": {"time_min": 30}
      }
    }
  ],
  "full_snapshot_omitted": true
}
```

### 9.2 计划 JSON（模型 A 输出，含 GRAD 元数据）

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

### 9.3 驳回补丁（模型 C/B 输出）

当模型 C 或 B 驳回时，必须返回标准化的"逻辑补丁"，包含受影响的局部范围，以便模型 A 精准定位：

```json
{
  "patch_id": "P_02",
  "source": "C_Model",
  "error_code": "E-01",
  "affected_scope": ["S_05", "S_06"],
  "missing_artifact": "Compiled_Binary.exe",
  "suggestion": "请在 S_04 与 S_05 之间插入'执行编译命令'节点。"
}
```

***

## 10. 收敛性保证与停机证明

**定理（GRAD V3.0 停机与成本收敛）**：设全量图纸规模为 $N$，系统设定最大迭代轮次为 $M_{\max}$（有限常数）。在 V3.0 差分协议下，总计算代价 $T$ 存在上界 $\mathcal{C}(N) + (M_{\max} - 1) \cdot \mathcal{C}(\Delta_{\max})$，其中 $\Delta_{\max} = \max_{k} |\Delta_k|$。

**证明（构造性）**：

1. **首轮有界性**：首轮骨架生成消耗 $\alpha \mathcal{C}(N)$，其中 $0 < \alpha < 1$。若骨架被驳回，浪费仅 $\alpha \mathcal{C}(N)$，远小于全量重绘。
2. **补丁单调递减引理**：在目标 $G$ 固定的有穷状态空间中，每轮修正后，未修改节点集合随迭代单调递增。因此，新增或修改的节点数量 $|\Delta_k|$ 严格递减，直至为 0。即存在 $\Delta_{\max} \ll N$，使得 $\forall k,\ |\Delta_k| \leq \Delta_{\max}$。
3. **最坏情况上界**：即使取最坏情况，总代价 $T \leq \alpha \mathcal{C}(N) + \sum_{k=2}^{M_{\max}} \mathcal{C}(\Delta_{\max})$。由于 $M_{\max}$ 是预设常数，$\sum \mathcal{C}(\Delta_{\max}) = \mathrm{const} \times \mathcal{C}(\Delta_{\max})$。
4. **比较结论**：相比原始全量迭代 $M_{\max} \cdot \mathcal{C}(N)$，在 $\Delta_{\max} \ll N$ 的工程假设下，$T$ 实现从 $O(M \cdot N)$ 到 $O(N + M \cdot \Delta_{\max})$ 的结构性降级。**系统必然在 $M_{\max}$ 步内停机**（超时则强转人工介入）。

***

## 11. 异常处理与容灾机制（V3.0 新增）

| 异常现象 | 数学判定条件 | 系统响应策略 |
| :--- | :--- | :--- |
| **补丁震荡（Oscillation）** | 连续两轮补丁来回修改同一节点，即 $\Delta_{k+1} \approx -\Delta_k$ | 系统自动锁死该节点，将其标记为"人工待定区"，模型 A 不再拥有修改权限，降级为线性执行模式。 |
| **补丁熵增（Entropy Increase）** | 当前补丁尺寸 $|\Delta_{k+1}| \geq |\Delta_k|$（不缩反扩） | 系统拒绝该补丁，强制回滚至上一稳定版本，并向模型 A 发送"冲突规避"信号，要求重新生成。 |
| **B 终裁置信度不足** | 余弦相似度得分低于预设阈值 $\theta_B$（如 0.55） | 模型 B 放弃终裁权，默认采纳模型 C 的硬逻辑判定（保守安全策略）。 |
| **超时强制终止** | 迭代轮次 $k > M_{\max}$ | 系统强制输出当前评分最高的 DAG，并标注"需人工复核"的最高风险等级。 |

***

## 12. 双思考工作空间（决策链留痕）

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

***

## 13. 公共 API

头文件：`agentrt/atoms/coreloopthree/src/cognition/critique/grad_verifier.h`、`grad_coordinator.h`、`grad_llm_callbacks.h`

| API                                      | 功能                             |
| ---------------------------------------- | ------------------------------ |
| `airy_grad_verify_plan()`                | 全量四验（E-01/E-02/E-03 + E-04 统计） |
| `airy_grad_verify_scope()`               | 差分四验（Δ\_k 及一阶闭包）               |
| `airy_grad_coordinator_create/destroy()` | GRAD 协调器生命周期                   |
| `airy_grad_coordinator_execute()`        | 计划级批判循环（含 seed plan 与收敛判定）     |
| `airy_grad_build_patch()`                | 构造驳回补丁（GRAD §9.3 格式）           |
| `grad_llm_s2_plan()`                     | 模型 A 计划生成回调（LLM + 保守降级）        |
| `grad_llm_s1_arbiter()`                  | 模型 B 语境终裁回调（LLM + 保守采纳 C）      |
| `airy_cognition_set_grad_enabled()`      | 引擎级开关（cognition.h）             |

***

## 14. 在 CoreLoopThree 中的插接位置

```
Phase 0 拆解 → [GCCP 确认阶段] → Phase 1 规划 → [GRAD 计划级批判循环] →
Phase 2 执行（GRAD 启用时跳过文本级批判循环）→ Phase 3 审计 → Phase 4 目标对齐
```

- **插接行为**：Phase 1 规划产出 `airy_task_plan_t` 后，若 `enable_grad` 且 LLM 可用，创建 GRAD 协调器执行四验/终裁/修正；收敛后采用收敛计划。
- **Phase 2 降级**：GRAD 启用时，Phase 2 文本级批判循环（TC3 t2/t1-f/t1-p 文本循环）被跳过（`!enable_grad` 时回退，向后兼容）。
- **分级思考**：非任务集对话仅启用 t1-f 轻量验证（`AIRY_CHAT_T1F_VERIFY=1`）；任务集启用全量 GRAD。

***

## 15. 测试

`atoms/coreloopthree/tests/unit/test_grad.c`（8/8 通过，ASan 无泄漏）：

- E-02 死锁检测（Kahn 环检测）
- E-01 因果充分性（类型签名覆盖）
- E-03 资源守恒（成本预算）
- 协调器 seed 收敛（不修改 seed）
- 协调器驳回 → 模型 A 重新生成 → 收敛
- 驳回补丁构造（GRAD §9.3 格式）
- 差分范围验证（全量/闭包/非法节点降级）

***

## 16. 总结：与 GCCP 的关系及 V3.0 客观定位

本协议 GRAD V3.0 与事实锁定协议 GCCP 共同构成了"人类-人工智能协同决策"的完整科学框架：

- **GCCP（事实层）**：处理"不确定性消除"——通过四问将无限可能映射为唯一的客观目标 $G$。（数学映射：逆问题正则化）
- **GRAD V3.0（逻辑层）**：处理"计算冗余削减"——通过差分迭代与结构化验证，将错误的候选路径修剪至唯一可行的可达路径 $P^*$。（数学映射：带剪枝的梯度下降）

**V3.0 核心价值声明**：本协议在数学上**不承诺具体的百分比节省**（因节省率依赖具体任务的 $N$ 与 $\Delta_k$ 分布），但**严格承诺了复杂度的结构性优化**：将验证成本从随迭代轮次线性增长 $O(M \cdot N)$ 优化为 $O(N + M \cdot \Delta_{\max})$，其中 $\Delta_{\max} \ll N$。

**最终协同效应**：当 GCCP 锁定了事实 $G$，且 GRAD V3.0 锁定了逻辑 $P^*$ 后，系统便从"概率推测阶段"进入"确定性执行阶段"。此即 **"事实-逻辑双锁决策模型（Fact-Logic Twin Lock Decision Model）"**。

***

## 17. 相关文档

- [目标完备确认协议 (GCCP)](07-gccp.md) — 事实锁定层（G 的来源，基于最优控制与数值优化 CCP-V2.0）
- [认知循环运行时 (CoreLoopThree)](02-coreloopthree.md) — GRAD 所在认知管线
- [工作大厅 (Work Hall)](08-work-hall.md) — 收敛计划的执行宿主
- [双思考系统设计](../00-requirements/02-cognition-design-cn.md) — Thinkdual 认知架构

***

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
