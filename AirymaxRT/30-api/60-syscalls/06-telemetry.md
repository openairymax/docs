Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 可观测性系统调用 API

**最新**: 2026-06-28
**状态**: 维护中
**路径**: OpenAirymax/Docs/30-api/60-syscalls/06-telemetry.md
**API 版本**: v1.0.0
---

## 概述

可观测性系统调用提供系统指标和链路追踪数据的查询接口。全部使用 `agentrt_sys_telemetry_*` 前缀，返回 JSON 格式数据。

> **说明**: 日志写入、指标记录、追踪 span 创建等写操作不通过 syscall 层暴露，而是由各 daemon（observe_d、monit_d）和内部子系统直接处理。syscall 层仅提供读取接口。

### 五维正交原则体现

| 维度 | 原则体现 | 具体实现 |
|------|----------|---------|
| **系统观** | 可观测性数据的标准化读取 | 统一的 JSON 输出格式，支持 Prometheus 兼容 |
| **内核观** | 极简的读取契约 | 仅 2 个核心函数 |
| **认知观** | 认知过程透明化 | 链路追踪支持 trace_id 查询 |
| **工程观** | 可观测性内置 | 结构化数据输出，明确的所有权语义 |
| **设计美学** | 简洁的查询接口 | 一致的前缀、清晰的错误返回 |

---

## API 索引

| 函数 | 描述 |
|------|------|
| `agentrt_sys_telemetry_metrics()` | 获取系统指标（JSON 格式） |
| `agentrt_sys_telemetry_traces()` | 获取链路追踪跨度列表 |

---

## API 详细说明

### `agentrt_sys_telemetry_metrics()`

```c
/**
 * @brief 获取系统指标
 * @param out_metrics 输出参数，返回指标数据（JSON 格式）
 * @return agentrt_error_t (AGENTRT_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_metrics，通过 agentrt_sys_free() 释放
 * @threadsafe 是
 * @reentrant 是
 */
AGENTRT_API agentrt_error_t agentrt_sys_telemetry_metrics(char **out_metrics);
```

**使用示例**：
```c
char *metrics = NULL;
agentrt_error_t err = agentrt_sys_telemetry_metrics(&metrics);
if (err == AGENTRT_OK) {
    printf("Metrics: %s\n", metrics);
    agentrt_sys_free(metrics);
}
```

### `agentrt_sys_telemetry_traces()`

```c
/**
 * @brief 获取链路追踪跨度列表
 * @param trace_id 追踪 ID
 * @param out_spans 输出参数，返回跨度列表（JSON 数组格式）
 * @return agentrt_error_t (AGENTRT_OK=0 成功，负值=错误)
 *
 * @ownership 调用者拥有 *out_spans，通过 agentrt_sys_free() 释放
 * @threadsafe 是
 * @reentrant 是
 */
AGENTRT_API agentrt_error_t agentrt_sys_telemetry_traces(const char *trace_id,
                                                          char **out_spans);
```

**使用示例**：
```c
char *spans = NULL;
agentrt_error_t err = agentrt_sys_telemetry_traces("trace_abc123", &spans);
if (err == AGENTRT_OK) {
    printf("Spans: %s\n", spans);
    agentrt_sys_free(spans);
}
```

---

## 注意事项

1. **输出所有权**：所有 `out_*` 参数由调用者通过 `agentrt_sys_free()` 释放。
2. **错误码**：返回值使用 [error.h](../../AgentRT/agentrt/commons/utils/error/include/error.h) 定义的负整数错误码（`AGENTRT_OK=0`, `AGENTRT_ERR_INVALID_PARAM=-2` 等）。
3. **可观测性后端**：指标和追踪的实际采集、存储、导出一由 [observe_d](../../AgentRT/agentrt/daemon/observe_d/) 负责，具体日志格式见 [日志格式规范](../../50-specifications/10-contracts/05-logging-format.md)。

---

## 相关文档

- [系统调用总览](../README.md)
- [错误码体系](../../AgentRT/agentrt/commons/utils/error/include/error.h)
- [日志格式规范](../../50-specifications/10-contracts/05-logging-format.md)

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
