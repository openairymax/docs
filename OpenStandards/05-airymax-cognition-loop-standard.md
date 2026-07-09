Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax 认知循环标准

> **文档定位**: Airymax 开放标准
> **版本**: 1.0（开放标准草案）
> **最后更新**: 2026-07-09
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 一、范围与目的

本标准定义 AI Agent 认知循环的开放契约，覆盖：

- CoreLoopThree 三阶段标准（PERCEPTION / THINKING / ACTION）
- Thinkdual 双模式标准（SYSTEM1_FAST / SYSTEM2_SLOW）
- LLM 推理阶段标准（PREFILL / DECODE / SPECULATIVE）
- Token 能效指标标准

本标准面向：

- **运行时实现者**：实现兼容的认知循环运行时（用户态线程、内核 kthread 等）。
- **Agent 作者**：编写符合三阶段状态机的认知逻辑。
- **LLM 推理引擎开发者**：对接 PREFILL/DECODE/SPECULATIVE 推理阶段。
- **能效优化方**：基于 Token 能效指标优化推理成本。

本标准遵循 IRON-9 v2 [SC] 共享契约层——CoreLoopThree 阶段枚举、Thinkdual 模式枚举、LLM 推理阶段枚举在 agentrt 与 agentrt-liunx 之间完全共享。

### 1.1 术语

| 术语 | 含义 |
|------|------|
| CoreLoopThree | 认知三阶段循环：感知-思考-行动 |
| PERCEPTION | 感知阶段：解析输入、提取意图 |
| THINKING | 思考阶段：规划、推理、决策 |
| ACTION | 行动阶段：执行、调用工具、产出结果 |
| Thinkdual | 双思考系统：快思考 + 慢思考 |
| SYSTEM1_FAST | 快思考：直觉式、低延迟 |
| SYSTEM2_SLOW | 慢思考：深度推理、高延迟 |
| PREFILL | LLM 预填充阶段：处理 prompt |
| DECODE | LLM 解码阶段：生成 token |
| SPECULATIVE | LLM 投机解码：草稿模型 + 验证 |

### 1.2 规范用词

沿用 RFC 2119 用词。

---

## 二、CoreLoopThree 三阶段标准

### 2.1 三阶段模型

CoreLoopThree 定义认知循环的三个不可分割阶段：

| 阶段 | 枚举值 | 职责 | 输入 | 输出 |
|------|--------|------|------|------|
| PERCEPTION | 0 | 感知：解析输入、提取意图、加载上下文 | 原始输入 + L1 记忆 | 意图对象 + 上下文窗口 |
| THINKING | 1 | 思考：规划、推理、决策 | 意图对象 + 上下文 | 行动计划 |
| ACTION | 2 | 行动：执行计划、调用工具、产出结果 | 行动计划 | 执行结果 + L1 更新 |

枚举值**永久稳定**（AOS-STD-COG-001，L1 稳定级）：

```c
typedef enum {
    AGENTRT_COG_PHASE_PERCEPTION = 0,
    AGENTRT_COG_PHASE_THINKING   = 1,
    AGENTRT_COG_PHASE_ACTION      = 2,
} agentrt_cog_phase_t;
```

### 2.2 状态机

认知循环遵循以下状态机（AOS-STD-COG-002，L1 稳定级）：

```
                 ┌─────────────────────────────┐
                 │                             │
                 ▼                             │
        ┌────────────────┐    ┌──────────┐    │
──> 0 │  PERCEPTION    │ -> │ THINKING │ -> │
        └────────────────┘    └──────────┘    │
                                  │            │
                                  ▼            │
                              ┌────────┐       │
                              │ ACTION │ ──────┘
                              └────────┘
                                  │
                                  ▼
                              (循环结束 / 终止)
```

转移规则：

| 转移 | 触发 | 约束 |
|------|------|------|
| → PERCEPTION | 循环启动 | **必须**为初始状态 |
| PERCEPTION → THINKING | 感知完成 | **必须**产出意图对象 |
| THINKING → ACTION | 思考完成 | **必须**产出行动计划 |
| ACTION → PERCEPTION | 循环继续 | **可选**，根据行动计划决定 |
| 任意 → 终止 | Token 预算耗尽 / 显式终止 | **必须**清理上下文 |

非法转移**必须**返回 `AGENTRT_EINVAL`（AOS-STD-COG-003）。

### 2.3 CoreLoopThree 上下文结构

```c
#define AGENTRT_COG_MAGIC 0x434f4731u  /* 'COG1' */

typedef struct {
    uint32_t magic;                 /* 偏移 0:  魔数 0x434f4731 'COG1' */
    uint16_t version;                /* 偏移 4:  版本 0x0100 */
    uint16_t current_phase;          /* 偏移 6:  当前阶段（枚举值） */
    uint32_t iteration;              /* 偏移 8:  循环迭代次数 */
    uint32_t reserved1;
    uint64_t agent_id;               /* 偏移 16: Agent ID */
    uint64_t trace_id;               /* 偏移 24: 链路追踪 ID */
    uint64_t started_ns;             /* 偏移 32: 循环启动时间 */
    uint64_t deadline_ns;            /* 偏移 40: 截止时间（0=无） */
    /* Token 预算 */
    uint64_t token_consumed;         /* 偏移 48: 已消耗 token */
    uint64_t token_budget;           /* 偏移 56: 预算上限 */
    /* Thinkdual 模式 */
    uint16_t think_mode;             /* 偏移 64: 当前思考模式 */
    uint16_t llm_phase;              /* 偏移 66: LLM 推理阶段 */
    uint32_t reserved2;
    /* 能效指标 */
    uint64_t prefill_tokens;         /* 偏移 72: prefill token 数 */
    uint64_t decode_tokens;          /* 偏移 80: decode token 数 */
    uint64_t speculative_tokens;     /* 偏移 88: 投机 token 数 */
    uint64_t energy_uj;              /* 偏移 96: 能耗（微焦耳） */
    uint8_t  reserved[24];           /* 偏移 104: 保留 */
} agentrt_cog_context_t;             /* 总长 128 字节 */

_Static_assert(sizeof(agentrt_cog_context_t) == 128,
               "agentrt_cog_context_t must be 128 bytes");
```

上下文结构**永久稳定**（AOS-STD-COG-004，L1 稳定级）。

---

## 三、Thinkdual 双模式标准

### 3.1 双模式模型

Thinkdual 实现双思考系统，灵感来自 Kahneman 的 System 1 / System 2：

| 模式 | 枚举值 | 特征 | 延迟目标 | 典型场景 |
|------|--------|------|---------|---------|
| SYSTEM1_FAST | 0 | 快思考：直觉式、低延迟、低成本 | < 100 ms | 简单问答、事实核查 |
| SYSTEM2_SLOW | 1 | 慢思考：深度推理、高延迟、高成本 | < 5 s | 复杂规划、反思调整 |

枚举值**永久稳定**（AOS-STD-COG-005，L1 稳定级）：

```c
typedef enum {
    AGENTRT_THINK_SYSTEM1_FAST = 0,
    AGENTRT_THINK_SYSTEM2_SLOW = 1,
} agentrt_think_mode_t;
```

### 3.2 模式选择规则

运行时**必须**在 THINKING 阶段开始时选择思考模式（AOS-STD-COG-006）：

| 条件 | 选择 | 依据 |
|------|------|------|
| 意图复杂度低 + 上下文 < 4K token | SYSTEM1_FAST | 简单任务用快思考 |
| 意图复杂度高 或 上下文 > 32K token | SYSTEM2_SLOW | 复杂任务用慢思考 |
| Token 预算 < 500 | SYSTEM1_FAST | 预算紧张用快思考 |
| 显式标记 `force_slow=true` | SYSTEM2_SLOW | 用户强制慢思考 |
| 安全相关操作 | SYSTEM2_SLOW | 安全操作强制慢思考 |

模式选择规则**应当**可由策略注入覆盖（AOS-STD-COG-007）。

### 3.3 模式切换

THINKING 阶段内**可以**切换模式（AOS-STD-COG-008）：

- SYSTEM1 → SYSTEM2：快思考发现复杂度超预期时升级。
- SYSTEM2 → SYSTEM1：慢思考产出初步结论后用快思考验证。

切换**必须**记录审计（AOS-STD-COG-009）；切换不重置 `token_consumed`。

---

## 四、LLM 推理阶段标准

### 4.1 三阶段模型

LLM 推理分为三个阶段：

| 阶段 | 枚举值 | 职责 | Token 计费 |
|------|--------|------|-----------|
| PREFILL | 0 | 处理 prompt，计算 KV cache | 按 prompt token 数 |
| DECODE | 1 | 自回归生成 token | 按生成 token 数 |
| SPECULATIVE | 2 | 投机解码：草稿模型生成 + 主模型验证 | 草稿 token + 验证 token |

枚举值**永久稳定**（AOS-STD-COG-010，L1 稳定级）：

```c
typedef enum {
    AGENTRT_LLM_PHASE_PREFILL     = 0,
    AGENTRT_LLM_PHASE_DECODE      = 1,
    AGENTRT_LLM_PHASE_SPECULATIVE = 2,
} agentrt_llm_phase_t;
```

### 4.2 阶段语义

#### 4.2.1 PREFILL 阶段

PREFILL 阶段处理输入 prompt，构建 KV cache。运行时**必须**：

- **必须**统计 prompt token 数并扣减预算（AOS-STD-COG-011）。
- **应当**支持 prefix caching（相同 prompt 前缀复用 KV cache）。
- **应当**在 prompt > 32K 时启用长上下文优化（如 RingAttention）。

#### 4.2.2 DECODE 阶段

DECODE 阶段自回归生成 token。运行时**必须**：

- **必须**逐 token 扣减预算（AOS-STD-COG-012）。
- **必须**在预算耗尽时终止生成。
- **应当**支持 batching（多个请求共享 decode 步骤）。
- **应当**支持流式输出（每 N 个 token 产出一次）。

#### 4.2.3 SPECULATIVE 阶段

SPECULATIVE 阶段使用草稿模型 + 主模型验证。运行时**必须**：

- **必须**统计草稿 token 数与验证 token 数（AOS-STD-COG-013）。
- **应当**在草稿接受率 > 50% 时启用投机解码。
- **应当**在草稿接受率 < 20% 时关闭投机解码（退化为纯 DECODE）。

### 4.3 阶段转移

```
PREFILL -> DECODE -> (可选 SPECULATIVE) -> DECODE -> ... -> 终止
```

转移规则：

| 转移 | 触发 | 约束 |
|------|------|------|
| PREFILL → DECODE | prompt 处理完成 | **必须**构建 KV cache |
| DECODE → SPECULATIVE | 启用投机解码 | **必须**有草稿模型 |
| SPECULATIVE → DECODE | 草稿验证完成 | 接受的 token 计入 decode_tokens |
| 任意 → 终止 | 预算耗尽 / EOS | — |

---

## 五、Token 能效指标标准

### 5.1 指标定义

| 指标 | 计算公式 | 单位 | 含义 |
|------|---------|------|------|
| Token 吞吐量 | `decode_tokens / elapsed_s` | token/s | 解码速度 |
| Prefill 效率 | `prefill_tokens / prefill_time_s` | token/s | 预填充速度 |
| 投机接受率 | `accepted_spec_tokens / total_spec_tokens` | 0 ~ 1 | 草稿模型准确度 |
| 能效比 | `total_tokens / energy_uj * 1e6` | token/J | 单位能耗产出的 token |
| 成本效率 | `total_tokens / cost_usd` | token/$ | 单位成本产出的 token |
| 首字延迟 | `first_token_time - request_time` | ms | TTFT |

### 5.2 能效指标结构

```c
typedef struct {
    uint64_t total_tokens;          /* 总 token 数（prefill + decode + speculative） */
    uint64_t prefill_tokens;
    uint64_t decode_tokens;
    uint64_t speculative_tokens;
    uint64_t accepted_spec_tokens;  /* 投机接受 token 数 */
    uint64_t rejected_spec_tokens;  /* 投机拒绝 token 数 */
    uint64_t elapsed_ns;            /* 总耗时（纳秒） */
    uint64_t energy_uj;             /* 总能耗（微焦耳） */
    uint64_t cost_micro_usd;         /* 总成本（微美元） */
    uint64_t first_token_ns;        /* 首字延迟（纳秒） */
} agentrt_token_efficiency_t;
```

### 5.3 指标上报

运行时**必须**在认知循环结束后上报能效指标（AOS-STD-COG-014，L1 稳定级）：

```c
/**
 * agentrt_cog_report_efficiency - 上报能效指标
 * @ctx:        CoreLoopThree 上下文
 * @efficiency: 能效指标
 *
 * 返回值:
 *   0: 成功
 */
int agentrt_cog_report_efficiency(const agentrt_cog_context_t *ctx,
                                   const agentrt_token_efficiency_t *efficiency);
```

### 5.4 指标阈值（建议）

| 指标 | 建议阈值 | 说明 |
|------|---------|------|
| Token 吞吐量 | > 50 token/s | SYSTEM2_SLOW |
| Token 吞吐量 | > 500 token/s | SYSTEM1_FAST |
| 首字延迟 | < 500 ms | 交互场景 |
| 投机接受率 | > 0.5 | 启用投机解码阈值 |
| 能效比 | > 1000 token/J | GPU 推理 |

阈值**不**作为合规判定依据，仅供实现参考（AOS-STD-COG-015）。

---

## 六、CoreLoopThree 接口

### 6.1 循环控制接口

```c
/**
 * agentrt_cog_loop_init - 初始化认知循环上下文
 * @ctx:         上下文（输出）
 * @agent_id:    Agent ID
 * @token_budget: Token 预算上限
 * @deadline_ns: 截止时间（0=无）
 *
 * 初始化后 current_phase = PERCEPTION。
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   参数无效
 */
int agentrt_cog_loop_init(agentrt_cog_context_t *ctx,
                          uint64_t agent_id,
                          uint64_t token_budget,
                          uint64_t deadline_ns);

/**
 * agentrt_cog_loop_step - 推进一个阶段
 * @ctx:     上下文
 * @input:   阶段输入
 * @output:  阶段输出
 *
 * 根据当前阶段调用对应的处理函数。
 * 处理完成后自动转移到下一阶段。
 *
 * 返回值:
 *   0:                  阶段完成
 *   -AGENTRT_EINVAL:   非法状态
 *   -AGENTRT_EBUDGET:  Token 预算耗尽
 *   -AGENTRT_ETIMEOUT: 超过 deadline
 */
int agentrt_cog_loop_step(agentrt_cog_context_t *ctx,
                          const void *input, void *output);

/**
 * agentrt_cog_loop_finalize - 终止循环
 * @ctx: 上下文
 */
int agentrt_cog_loop_finalize(agentrt_cog_context_t *ctx);
```

### 6.2 阶段处理函数注册

```c
typedef int (*agentrt_cog_phase_handler_t)(agentrt_cog_context_t *ctx,
                                            const void *input,
                                            void *output);

/**
 * agentrt_cog_register_phase - 注册阶段处理函数
 * @phase:    阶段
 * @handler:  处理函数
 */
int agentrt_cog_register_phase(agentrt_cog_phase_t phase,
                                agentrt_cog_phase_handler_t handler);
```

---

## 七、标准条目注册表

| 编号 | 名称 | 稳定级 |
|------|------|--------|
| AOS-STD-COG-001 | CoreLoopThree 三阶段枚举 | L1 |
| AOS-STD-COG-002 | 状态机转移规则 | L1 |
| AOS-STD-COG-003 | 非法转移返回 EINVAL | L1 |
| AOS-STD-COG-004 | 上下文结构 128 字节 | L1 |
| AOS-STD-COG-005 | Thinkdual 双模式枚举 | L1 |
| AOS-STD-COG-006 | 模式选择规则 | L2 |
| AOS-STD-COG-007 | 策略可覆盖 | L2 |
| AOS-STD-COG-008 | 模式切换 | L2 |
| AOS-STD-COG-009 | 切换审计 | L1 |
| AOS-STD-COG-010 | LLM 推理阶段枚举 | L1 |
| AOS-STD-COG-011 | PREFILL 预算扣减 | L1 |
| AOS-STD-COG-012 | DECODE 预算扣减 | L1 |
| AOS-STD-COG-013 | SPECULATIVE 统计 | L1 |
| AOS-STD-COG-014 | 能效指标上报 | L1 |
| AOS-STD-COG-015 | 阈值非合规依据 | L2 |
| AOS-STD-COG-016 | 上下文 magic 0x434f4731 | L1 |

---

## 八、错误码

| 错误码 | 数值 | 含义 |
|--------|------|------|
| `AGENTRT_OK` | 0 | 成功 |
| `AGENTRT_EINVAL` | -2 | 参数非法（含非法状态转移） |
| `AGENTRT_EBUDGET` | -1004 | Token 预算耗尽 |
| `AGENTRT_ETIMEOUT` | -1004 | 超过 deadline |
| `AGENTRT_ECog_TIMEOUT` | -1701 | 认知处理超时 |
| `AGENTRT_EMODEL` | -1702 | 模型调用失败 |
| `AGENTRT_EPHASE` | -1703 | 阶段处理函数未注册 |

---

## 九、兼容性测试要求

| 测试 ID | 测试内容 | 通过条件 |
|---------|---------|---------|
| COG-T-001 | 三阶段枚举 | 枚举值精确匹配 |
| COG-T-002 | 状态机 | 合法转移成功，非法转移返回 EINVAL |
| COG-T-003 | 上下文布局 | 128 字节，magic 匹配 |
| COG-T-004 | 双模式选择 | 简单任务选 SYSTEM1，复杂选 SYSTEM2 |
| COG-T-005 | PREFILL 预算扣减 | prompt token 数正确扣减 |
| COG-T-006 | DECODE 预算扣减 | 逐 token 扣减，耗尽终止 |
| COG-T-007 | SPECULATIVE 统计 | 草稿/接受/拒绝 token 数正确 |
| COG-T-008 | 能效指标上报 | 上报后指标可查询 |
| COG-T-009 | 模式切换审计 | 切换产生审计记录 |

---

## 十、最小示例

### 10.1 三阶段状态机实现

```c
/* 注册阶段处理函数 */
agentrt_cog_register_phase(AGENTRT_COG_PHASE_PERCEPTION, handle_perception);
agentrt_cog_register_phase(AGENTRT_COG_PHASE_THINKING,   handle_thinking);
agentrt_cog_register_phase(AGENTRT_COG_PHASE_ACTION,     handle_action);

/* 运行认知循环 */
agentrt_cog_context_t ctx;
agentrt_cog_loop_init(&ctx,
                      /* agent_id */ 42,
                      /* budget */ 4096,
                      /* deadline */ 0);

const char *user_input = "北京天气如何？";
void *input = (void *)user_input;
void *output = NULL;

while (ctx.current_phase != /* terminated */ 99) {
    int rc = agentrt_cog_loop_step(&ctx, input, output);
    if (rc == -AGENTRT_EBUDGET) {
        fprintf(stderr, "Token 预算耗尽\n");
        break;
    }
    if (rc == -AGENTRT_ETIMEOUT) {
        fprintf(stderr, "超时\n");
        break;
    }
    /* output 作为下一阶段 input */
    input = output;
}

agentrt_cog_loop_finalize(&ctx);
```

### 10.2 Thinkdual 模式选择

```c
int handle_thinking(agentrt_cog_context_t *ctx, const void *input, void *output)
{
    /* 根据意图复杂度选择模式 */
    intent_t *intent = (intent_t *)input;
    if (intent->complexity < 0.3 && ctx->token_budget - ctx->token_consumed > 500) {
        ctx->think_mode = AGENTRT_THINK_SYSTEM1_FAST;
    } else {
        ctx->think_mode = AGENTRT_THINK_SYSTEM2_SLOW;
    }

    /* 调用 LLM 推理 */
    llm_request_t req = { .prompt = intent->prompt, .mode = ctx->think_mode };
    llm_response_t resp;
    int rc = llm_infer(&req, &resp);

    /* 扣减预算 */
    ctx->token_consumed += resp.prefill_tokens + resp.decode_tokens;
    ctx->prefill_tokens  += resp.prefill_tokens;
    ctx->decode_tokens   += resp.decode_tokens;

    /* 产出行动计划 */
    plan_t *plan = parse_plan(resp.text);
    *(plan_t **)output = plan;
    return 0;
}
```

### 10.3 能效指标上报

```c
agentrt_token_efficiency_t eff = {0};
eff.total_tokens       = ctx.prefill_tokens + ctx.decode_tokens + ctx.speculative_tokens;
eff.prefill_tokens     = ctx.prefill_tokens;
eff.decode_tokens      = ctx.decode_tokens;
eff.speculative_tokens = ctx.speculative_tokens;
eff.elapsed_ns         = now_ns() - ctx.started_ns;
eff.first_token_ns     = first_token_time - ctx.started_ns;

/* 投机接受率统计（假设接受 80/100） */
eff.accepted_spec_tokens = 80;
eff.rejected_spec_tokens = 20;

agentrt_cog_report_efficiency(&ctx, &eff);
```

---

## 十一、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-07-09 | 初始草案：CoreLoopThree 三阶段、Thinkdual 双模式、LLM 推理三阶段、Token 能效指标 |

---

## 十二、参考文献

- [Airymax 开放标准体系总览](./README.md)
- [Airymax 记忆卷载标准](./04-airymax-memory-rovol-standard.md)
- [Airymax 调度标准](./06-airymax-scheduling-standard.md)
- Kahneman, D. (2011). Thinking, Fast and Slow.
- Speculative Decoding：https://arxiv.org/abs/2211.17192
- vLLM PagedAttention：https://arxiv.org/abs/2309.06180

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
