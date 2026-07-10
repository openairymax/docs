Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax Agent 管理系统调用 API

**最新**: 2026-06-28
**状态**: 维护中
**路径**: OpenAirymax/Docs/30-api/60-syscalls/01-agent.md
**API 版本**: v1.0.0
---

## 概述

Agent 管理系统调用提供 Agent 实例的创建、销毁、调用和查询接口。Agent 遵循 Thinkdual 双思考系统（t1-f 快思考 / t2 慢思考）。所有函数使用 `agentrt_sys_agent_*` 前缀。

### 五维正交原则体现

| 维度 | 原则体现 | 具体实现 |
|------|----------|---------|
| **系统观** | Agent 生命周期的系统管理 | spawn→invoke→terminate 的完整生命周期 |
| **内核观** | 极简的接口契约 | 仅 4 个核心函数 |
| **认知观** | Thinkdual 双思考系统配置 | Agent 规格中配置认知策略 |
| **工程观** | 安全与可观测性内置 | 统一错误码、Cupolas 权限控制 |
| **设计美学** | 简洁的 Agent 抽象 | JSON 规格驱动、一致的命名前缀 |

---

## API 索引

| 函数 | 描述 |
|------|------|
| `agentrt_sys_agent_spawn()` | 创建 Agent 实例 |
| `agentrt_sys_agent_terminate()` | 销毁 Agent 实例 |
| `agentrt_sys_agent_invoke()` | 调用 Agent 执行任务 |
| `agentrt_sys_agent_list()` | 列出所有 Agent |

---

## API 详细说明

### `agentrt_sys_agent_spawn()`

```c
/**
 * @brief 创建 Agent 实例
 * @param agent_spec Agent 规格（JSON 格式）
 * @param out_agent_id 输出参数，返回 Agent ID
 * @return agentrt_error_t (AGENTRT_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_agent_id，通过 agentrt_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AGENTRT_API agentrt_error_t agentrt_sys_agent_spawn(const char *agent_spec,
                                                     char **out_agent_id);
```

**Agent 规格（JSON）示例**：
```json
{
    "name": "customer_service_agent",
    "type": "chat",
    "cognition": {"model": "gpt-4o", "temperature": 0.7},
    "skills": ["file_reader", "data_processor"]
}
```

**使用示例**：
```c
char *agent_id = NULL;
const char *spec = "{\"name\": \"support_agent\", \"type\": \"chat\"}";
agentrt_error_t err = agentrt_sys_agent_spawn(spec, &agent_id);
if (err == AGENTRT_OK) {
    printf("Agent spawned: %s\n", agent_id);
    agentrt_sys_free(agent_id);
}
```

### `agentrt_sys_agent_terminate()`

```c
/**
 * @brief 销毁 Agent 实例
 * @param agent_id Agent ID
 * @return agentrt_error_t (AGENTRT_OK=0 成功，负值=错误)
 *
 * @threadsafe 是
 * @reentrant 否
 */
AGENTRT_API agentrt_error_t agentrt_sys_agent_terminate(const char *agent_id);
```

### `agentrt_sys_agent_invoke()`

```c
/**
 * @brief 调用 Agent 执行任务
 * @param agent_id Agent ID
 * @param input 输入数据
 * @param input_len 输入长度
 * @param out_output 输出参数，返回执行结果
 * @return agentrt_error_t (AGENTRT_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_output，通过 agentrt_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AGENTRT_API agentrt_error_t agentrt_sys_agent_invoke(const char *agent_id,
                                                      const char *input,
                                                      size_t input_len,
                                                      char **out_output);
```

**使用示例**：
```c
char *output = NULL;
const char *input = "请分析这段用户反馈数据";
agentrt_error_t err = agentrt_sys_agent_invoke(
    agent_id, input, strlen(input), &output);
if (err == AGENTRT_OK) {
    printf("Output: %s\n", output);
    agentrt_sys_free(output);
}
```

### `agentrt_sys_agent_list()`

```c
/**
 * @brief 列出所有 Agent 实例
 * @param out_agent_ids 输出参数，返回 Agent ID 数组
 * @param out_count 输出参数，返回 Agent 数量
 * @return agentrt_error_t (AGENTRT_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_agent_ids，通过 agentrt_sys_free() 释放
 * @threadsafe 是
 * @reentrant 否
 */
AGENTRT_API agentrt_error_t agentrt_sys_agent_list(char ***out_agent_ids,
                                                    size_t *out_count);
```

**使用示例**：
```c
char **agent_ids = NULL;
size_t count = 0;
agentrt_error_t err = agentrt_sys_agent_list(&agent_ids, &count);
if (err == AGENTRT_OK) {
    for (size_t i = 0; i < count; i++) {
        printf("[%zu] %s\n", i, agent_ids[i]);
    }
    agentrt_sys_free(agent_ids);
}
```

---

## 注意事项

1. **输出所有权**：所有 `out_*` 参数由调用者通过 `agentrt_sys_free()` 释放。
2. **错误码**：返回值使用 [error.h](../../AgentRT/agentrt/commons/utils/error/include/error.h) 定义的负整数错误码。
3. **系统调用初始化**：使用 Agent 相关 API 前需调用 `agentrt_syscalls_init()`。
4. **Agent 安全**：Agent 操作受 [Cupolas 安全穹顶](../../AgentRT/agentrt/cupolas/include/cupolas.h) 权限控制。

---

## 相关文档

- [系统调用总览](../README.md)
- [任务管理系统调用](task.md)
- [会话管理系统调用](session.md)
- [Skill 管理系统调用](skill.md)
- [架构设计：Thinkdual 双思考系统](../../10-architecture/thinkdual.md)
- [错误码体系](../../AgentRT/agentrt/commons/utils/error/include/error.h)

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
