Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax Agent 运行时标准  

> **文档定位**: Airymax 开放标准
> **版本**: 1.0（开放标准草案）
> **最后更新**: 2026-07-09
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 一、范围与目的

本标准定义 AI Agent 运行时（Airymax Agent Runtime）的开放接口契约，覆盖 Agent 生命周期、Agent 元数据格式、Agent 能力声明、Token 预算契约四大主题。

本标准面向：

- **运行时实现者**：基于本标准实现兼容的 Agent 运行时（用户态运行时、操作系统发行版）。
- **Agent 作者**：编写可在任何兼容运行时上运行的 Agent。
- **工具链开发者**：构建 Agent 调试器、监控器、注册中心等工具。
- **集成方**：将第三方 Agent 平台接入 Airymax 生态。

本标准仅规定**接口契约与语义**，不规定实现细节。实现者可自由选择编程语言、运行时载体、调度策略、内存模型。

### 1.1 术语

| 术语 | 含义 |
|------|------|
| Agent | 一个具备感知、思考、行动能力的智能体单元 |
| 运行时（Runtime） | 加载、调度、执行 Agent 的载体 |
| 微核心（micro-core） | 用户态极简运行时核心（agentrt 定位） |
| 微内核（micro-kernel） | 操作系统发行版的内核极简化形态（agentrt-linux 定位） |
| Token 预算 | Agent 一次执行可用 LLM token 上限 |
| 能力声明（Capability Manifest） | Agent 声明其需要的资源、权限、依赖的元数据 |

### 1.2 规范用词

本标准沿用 RFC 2119 用词：

- **必须**（MUST）：实现者强制遵守。
- **不得**（MUST NOT）：实现者禁止行为。
- **应当**（SHOULD）：强烈建议，可有合理例外。
- **不应当**（SHOULD NOT）：强烈不建议。
- **可以**（MAY）：可选行为。

---

## 二、Agent 生命周期标准

### 2.1 生命周期状态机

任何兼容运行时**必须**实现以下 6 状态机：

| 状态 | 枚举值 | 含义 | 允许的转移 |
|------|--------|------|-----------|
| `AGENT_STATE_CREATED` | 0 | 已创建，未注册 | → REGISTERED / DESTROYED |
| `AGENT_STATE_REGISTERED` | 1 | 已注册，未调度 | → SCHEDULED / DESTROYED |
| `AGENT_STATE_SCHEDULED` | 2 | 已调度，待执行 | → RUNNING / SUSPENDED / DESTROYED |
| `AGENT_STATE_RUNNING` | 3 | 正在执行 | → SUSPENDED / TERMINATED / REGISTERED |
| `AGENT_STATE_SUSPENDED` | 4 | 已挂起 | → SCHEDULED / TERMINATED |
| `AGENT_STATE_TERMINATED` | 5 | 已终止（正常） | → DESTROYED |
| `AGENT_STATE_DESTROYED` | 6 | 已销毁，资源已回收 | 终态 |

枚举值（`AOS-STD-RT-001`）：

```c
typedef enum {
    AGENT_STATE_CREATED     = 0,
    AGENT_STATE_REGISTERED  = 1,
    AGENT_STATE_SCHEDULED   = 2,
    AGENT_STATE_RUNNING     = 3,
    AGENT_STATE_SUSPENDED   = 4,
    AGENT_STATE_TERMINATED  = 5,
    AGENT_STATE_DESTROYED   = 6,
} airy_agent_state_t;
```

枚举值**永久稳定**（AOS-STD-RT-002，L1 稳定级），新状态只能追加到 `AGENT_STATE_DESTROYED` 之前。

### 2.2 状态转移规则

| 转移 | 触发接口 | 约束 |
|------|---------|------|
| CREATED → REGISTERED | `airy_agent_register` | 必须提供合法元数据 |
| REGISTERED → SCHEDULED | `airy_agent_schedule` | 必须通过能力检查 |
| SCHEDULED → RUNNING | 调度器选中 | 自动 |
| RUNNING → SUSPENDED | `airy_agent_suspend` | 必须保存上下文 |
| SUSPENDED → SCHEDULED | `airy_agent_resume` | 必须恢复上下文 |
| RUNNING → TERMINATED | Agent 自然结束 或 `airy_agent_terminate` | 必须清理临时资源 |
| 任意 → DESTROYED | `airy_agent_destroy` | 必须回收全部资源 |

非法转移**必须**返回 `AIRY_EINVAL`（AOS-STD-RT-003）。

### 2.3 生命周期接口

兼容运行时**必须**提供以下 6 个核心接口（AOS-STD-RT-004 ~ AOS-STD-RT-009）：

```c
/**
 * airy_agent_create - 创建 Agent 实例
 * @meta:        Agent 元数据（JSON 编码，见第 3 节）
 * @meta_len:    元数据字节长度
 * @agent_out:   输出：Agent 句柄
 *
 * 创建后 Agent 处于 CREATED 状态，尚未注册到运行时。
 *
 * 返回值:
 *   0:                  成功
 *   -AIRY_EINVAL:   元数据格式非法
 *   -AIRY_ENOMEM:   内存不足
 *   -AIRY_EPERM:    权限不足
 */
int airy_agent_create(const uint8_t *meta, size_t meta_len,
                         airy_agent_t *agent_out);

/**
 * airy_agent_register - 注册 Agent 到运行时
 * @agent:        Agent 句柄
 *
 * 将 Agent 注册到运行时注册表，使其可被调度。
 * 注册前必须通过能力检查（见第 4 节）。
 *
 * 返回值:
 *   0:                  成功
 *   -AIRY_EINVAL:   Agent 处于非法状态
 *   -AIRY_EEXIST:   Agent ID 已存在
 *   -AIRY_EPERM:    能力检查失败
 */
int airy_agent_register(airy_agent_t agent);

/**
 * airy_agent_schedule - 调度 Agent
 * @agent:        Agent 句柄
 * @priority:     优先级（0-139，对齐 SCHED_AGENT）
 * @budget:       Token 预算（见第 5 节）
 *
 * 返回值:
 *   0:                  成功
 *   -AIRY_EINVAL:   参数非法
 *   -AIRY_EBUSY:    调度资源已满（达到 MAC_MAX_AGENTS）
 */
int airy_agent_schedule(airy_agent_t agent, uint32_t priority,
                           airy_token_budget_t budget);

/**
 * airy_agent_suspend - 挂起 Agent
 * @agent:        Agent 句柄
 *
 * 返回值:
 *   0:                  成功
 *   -AIRY_EINVAL:   Agent 未运行
 */
int airy_agent_suspend(airy_agent_t agent);

/**
 * airy_agent_resume - 恢复 Agent
 * @agent:        Agent 句柄
 *
 * 返回值:
 *   0:                  成功
 *   -AIRY_EINVAL:   Agent 未挂起
 */
int airy_agent_resume(airy_agent_t agent);

/**
 * airy_agent_destroy - 销毁 Agent
 * @agent:        Agent 句柄
 *
 * 销毁后 Agent 句柄失效，所有资源被回收。
 *
 * 返回值:
 *   0:                  成功
 *   -AIRY_EINVAL:   Agent 句柄非法
 */
int airy_agent_destroy(airy_agent_t agent);
```

接口签名**永久稳定**（AOS-STD-RT-004 ~ AOS-STD-RT-009，L1 稳定级）。

---

## 三、Agent 元数据格式标准

### 3.1 元数据 JSON Schema

Agent 元数据使用 JSON 编码，遵循以下 JSON Schema（AOS-STD-RT-010）：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://docs.airymax.org/schemas/agent-meta.v1.json",
  "title": "Airymax Agent Metadata v1.0",
  "type": "object",
  "required": ["api_version", "agent_id", "name", "version", "runtime", "capabilities"],
  "properties": {
    "api_version": {
      "type": "string",
      "const": "airymax/v1",
      "description": "元数据 API 版本，固定为 airymax/v1"
    },
    "agent_id": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]{1,62}[a-z0-9]$",
      "description": "Agent 唯一标识符，小写字母数字与连字符"
    },
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 128,
      "description": "Agent 显示名称"
    },
    "version": {
      "type": "string",
      "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$",
      "description": "Agent 语义化版本 MAJOR.MINOR.PATCH"
    },
    "runtime": {
      "type": "object",
      "required": ["entry", "language"],
      "properties": {
        "entry":     { "type": "string", "description": "入口符号或文件路径" },
        "language":  { "enum": ["c", "rust", "python", "typescript", "wasm"] },
        "min_memory_mb": { "type": "integer", "minimum": 1 },
        "min_cpu_percent": { "type": "integer", "minimum": 1, "maximum": 100 }
      }
    },
    "capabilities": {
      "type": "array",
      "items": { "$ref": "#/$defs/capability" },
      "description": "Agent 能力声明清单，见第 4 节"
    },
    "token_budget": {
      "type": "object",
      "description": "Token 预算契约，见第 5 节",
      "properties": {
        "hard_limit":   { "type": "integer", "minimum": 0 },
        "soft_limit":   { "type": "integer", "minimum": 0 },
        "replenish_rate": { "type": "integer", "minimum": 0 }
      }
    },
    "dependencies": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["agent_id", "version_range"],
        "properties": {
          "agent_id":      { "type": "string" },
          "version_range": { "type": "string", "description": "semver range" }
        }
      }
    },
    "sandbox_level": {
      "enum": [0, 1, 2, 3, 4],
      "description": "沙箱级别，对齐 03 安全模型标准"
    }
  },
  "$defs": {
    "capability": {
      "type": "object",
      "required": ["domain", "actions"],
      "properties": {
        "domain":  { "type": "string", "description": "能力域，如 ipc/file/net/mem" },
        "actions": {
          "type": "array",
          "items": { "type": "string" },
          "description": "允许的动作列表"
        },
        "constraints": {
          "type": "object",
          "description": "可选的附加约束键值对"
        }
      }
    }
  }
}
```

### 3.2 字段约束

| 字段 | 约束 | 标准条目 |
|------|------|---------|
| `api_version` | **必须**为 `airymax/v1` | AOS-STD-RT-011 |
| `agent_id` | 全局唯一，**必须**满足正则 | AOS-STD-RT-012 |
| `version` | **必须**为语义化版本 | AOS-STD-RT-013 |
| `capabilities` | **必须**非空数组 | AOS-STD-RT-014 |
| `token_budget.hard_limit` | 若存在，**必须** >= `soft_limit` | AOS-STD-RT-015 |
| `sandbox_level` | 若省略，默认 2（容器隔离） | AOS-STD-RT-016 |

未在 Schema 中定义的字段**可以**出现，运行时**必须**忽略未知字段而非报错（AOS-STD-RT-017，前向兼容）。

### 3.3 元数据示例

```json
{
  "api_version": "airymax/v1",
  "agent_id": "com.spharx.demo.weather",
  "name": "Weather Agent",
  "version": "1.2.0",
  "runtime": {
    "entry": "weather_agent_main",
    "language": "c",
    "min_memory_mb": 64,
    "min_cpu_percent": 10
  },
  "capabilities": [
    { "domain": "net",   "actions": ["http_get"], "constraints": { "hosts": ["api.weather.example"] } },
    { "domain": "file",  "actions": ["read"],     "constraints": { "paths": ["/var/cache/weather/"] } }
  ],
  "token_budget": {
    "hard_limit": 8192,
    "soft_limit": 4096,
    "replenish_rate": 100
  },
  "sandbox_level": 2
}
```

---

## 四、Agent 能力声明标准

### 4.1 能力声明模型

Agent 通过能力声明（Capability Manifest）告知运行时其所需的资源访问权限。运行时根据声明授予 capability 令牌（详见 03 安全模型标准）。

能力声明由三要素构成（AOS-STD-RT-018）：

| 要素 | 含义 | 示例 |
|------|------|------|
| `domain` | 能力域 | `ipc` / `file` / `net` / `mem` / `sched` / `cog` |
| `actions` | 允许动作 | `read` / `write` / `send` / `recv` |
| `constraints` | 附加约束 | 路径前缀、主机白名单、最大字节 |

### 4.2 能力域枚举

能力域**必须**取自以下枚举（AOS-STD-RT-019，L1 稳定级）：

```c
typedef enum {
    AGENT_CAP_DOMAIN_IPC   = 0,
    AGENT_CAP_DOMAIN_FILE  = 1,
    AGENT_CAP_DOMAIN_NET   = 2,
    AGENT_CAP_DOMAIN_MEM   = 3,
    AGENT_CAP_DOMAIN_SCHED = 4,
    AGENT_CAP_DOMAIN_COG   = 5,
    AGENT_CAP_DOMAIN_SEC   = 6,
    AGENT_CAP_DOMAIN_TOOL  = 7,
} airy_cap_domain_t;
```

新增能力域**必须**追加到枚举末尾，不得修改既有值。

### 4.3 动作语义

| 动作 | 适用域 | 语义 |
|------|--------|------|
| `read` | file / mem / cog | 读取数据 |
| `write` | file / mem / cog | 写入数据 |
| `send` | ipc / net | 发送消息 |
| `recv` | ipc / net | 接收消息 |
| `execute` | tool / cog | 执行工具或推理 |
| `mint` | sec | 派生 capability（高权限） |
| `revoke` | sec | 撤销 capability（高权限） |
| `schedule` | sched | 调度其他 Agent |

运行时**必须**根据声明的动作授予对应 capability 令牌，**不得**授予未声明的动作（AOS-STD-RT-020，最小权限原则）。

### 4.4 约束语义

`constraints` 为键值对，运行时**必须**在授予 capability 时将约束写入令牌的 `permissions` 掩码或扩展字段。常用约束键：

| 约束键 | 类型 | 语义 |
|--------|------|------|
| `paths` | 字符串数组 | 文件访问路径前缀白名单 |
| `hosts` | 字符串数组 | 网络访问主机白名单 |
| `ports` | 整数数组 | 网络端口白名单 |
| `max_bytes` | 整数 | 单次操作最大字节数 |
| `max_rate` | 整数 | 每秒最大调用次数 |
| `ttl_seconds` | 整数 | capability 生存时间 |

约束冲突时，运行时**应当**取最严格值（AOS-STD-RT-021）。

---

## 五、Token 预算契约标准

### 5.1 预算模型

Token 预算约束 Agent 一次执行可消耗的 LLM token 数量，防止资源耗尽。预算模型包含三要素（AOS-STD-RT-022）：

| 要素 | 含义 |
|------|------|
| `hard_limit` | 硬上限，超过**必须**终止 Agent 并返回 `AIRY_EBUDGET` |
| `soft_limit` | 软阈值，超过**应当**触发告警但允许继续 |
| `replenish_rate` | 每秒补充的 token 数（令牌桶算法） |

### 5.2 预算数据结构

```c
typedef struct {
    uint64_t hard_limit;       /* 硬上限，0 表示无限制 */
    uint64_t soft_limit;      /* 软阈值，0 表示无告警 */
    uint32_t replenish_rate;  /* 每秒补充 token 数 */
    uint64_t consumed;        /* 已消耗 token 数（运行时维护） */
    uint64_t remaining;       /* 剩余 token 数（运行时维护） */
} airy_token_budget_t;
```

### 5.3 预算扣减规则

运行时在每次 LLM 推理调用后**必须**扣减预算（AOS-STD-RT-023）：

1. 扣减量 = `实际消耗 token 数`。
2. 若 `remaining` 扣减后 < 0，**必须**终止 Agent 并返回 `AIRY_EBUDGET`（错误码 -1004 区段，详见 03 标准）。
3. 若 `remaining` < `soft_limit`，**应当**触发 `AGENT_EVENT_BUDGET_LOW` 事件。
4. 令牌桶补充由调度器在每秒节拍执行，补充量 = `replenish_rate`。

### 5.4 预算继承

子 Agent 通过 `airy_agent_create` 创建时**可以**继承父 Agent 的预算（AOS-STD-RT-024）：

- 子预算的 `hard_limit` **不得**超过父预算的剩余 `remaining`。
- 子预算消耗计入父预算 `consumed`。
- 父预算耗尽时，所有子 Agent **必须**被终止。

预算继承形成树状结构，根节点为系统级预算池。

---

## 六、标准条目注册表

本标准定义的条目汇总：

| 编号 | 名称 | 稳定级 |
|------|------|--------|
| AOS-STD-RT-001 | Agent 生命周期状态枚举 | L1 |
| AOS-STD-RT-002 | 状态枚举值永久稳定 | L1 |
| AOS-STD-RT-003 | 非法状态转移返回 EINVAL | L1 |
| AOS-STD-RT-004 | airy_agent_create 接口 | L1 |
| AOS-STD-RT-005 | airy_agent_register 接口 | L1 |
| AOS-STD-RT-006 | airy_agent_schedule 接口 | L1 |
| AOS-STD-RT-007 | airy_agent_suspend 接口 | L1 |
| AOS-STD-RT-008 | airy_agent_resume 接口 | L1 |
| AOS-STD-RT-009 | airy_agent_destroy 接口 | L1 |
| AOS-STD-RT-010 | Agent 元数据 JSON Schema | L2 |
| AOS-STD-RT-011 | api_version 固定值 | L1 |
| AOS-STD-RT-012 | agent_id 唯一性 | L1 |
| AOS-STD-RT-013 | version 语义化 | L1 |
| AOS-STD-RT-014 | capabilities 非空 | L1 |
| AOS-STD-RT-015 | hard_limit >= soft_limit | L1 |
| AOS-STD-RT-016 | sandbox_level 默认 2 | L2 |
| AOS-STD-RT-017 | 未知字段前向兼容 | L1 |
| AOS-STD-RT-018 | 能力声明三要素 | L1 |
| AOS-STD-RT-019 | 能力域枚举 | L1 |
| AOS-STD-RT-020 | 最小权限授予 | L1 |
| AOS-STD-RT-021 | 约束取最严格值 | L2 |
| AOS-STD-RT-022 | Token 预算三要素 | L1 |
| AOS-STD-RT-023 | 预算扣减规则 | L1 |
| AOS-STD-RT-024 | 预算继承树 | L2 |

---

## 七、最小实现示例

### 7.1 Agent 元数据（weather-agent.json）

```json
{
  "api_version": "airymax/v1",
  "agent_id": "com.spharx.demo.weather",
  "name": "Weather Agent",
  "version": "1.0.0",
  "runtime": {
    "entry": "weather_agent_main",
    "language": "c"
  },
  "capabilities": [
    { "domain": "net",  "actions": ["http_get"], "constraints": { "hosts": ["api.weather.example"] } },
    { "domain": "ipc",  "actions": ["send", "recv"] }
  ],
  "token_budget": {
    "hard_limit": 4096,
    "soft_limit": 3072,
    "replenish_rate": 50
  },
  "sandbox_level": 2
}
```

### 7.2 Agent C 伪代码

```c
#include "airymax/agent_runtime.h"
#include "airymax/ipc.h"
#include <stdio.h>

/* Agent 入口函数，签名由运行时约定 */
int weather_agent_main(airy_agent_t self, int argc, char **argv)
{
    /* 1. 获取自身能力令牌（运行时根据元数据自动授予） */
    are_cap_t net_cap, ipc_cap;
    if (airy_cap_lookup(self, AGENT_CAP_DOMAIN_NET,  &net_cap)  != 0) return -1;
    if (airy_cap_lookup(self, AGENT_CAP_DOMAIN_IPC, &ipc_cap) != 0) return -1;

    /* 2. 通过能力令牌发起 HTTP 请求（受约束：仅访问 api.weather.example） */
    char response[4096];
    int rc = airy_http_get(net_cap, "/v1/forecast?city=beijing",
                              response, sizeof(response));
    if (rc < 0) {
        return rc;  /* 错误码透传 */
    }

    /* 3. 通过 IPC 把结果发回调度方 */
    struct airy_ipc_msg_hdr hdr = {0};
    hdr.magic        = AIRY_IPC_MAGIC;       /* 0x41524531 */
    hdr.opcode       = AIRY_IPC_OP_SEND;     /* SEND 操作码 */
    hdr.flags        = AIRY_IPC_F_NOREPLY;   /* 单向通知，无需响应 */
    hdr.src_task     = airy_self_task(self); /* 运行时填充，用户态不可伪造 */
    hdr.dst_task     = 0;                    /* 0 表示广播 */
    hdr.payload_len  = (uint32_t)rc;
    /* payload_type（0x0003 EVENT）由 payload 首字段携带，不在消息头中 */
    rc = airy_ipc_send(ipc_cap, &hdr, response, (size_t)rc);
    return rc;
}
```

### 7.3 运行时加载流程

```c
/* 运行时侧伪代码 */
int load_and_run_weather_agent(void)
{
    uint8_t meta[4096];
    size_t meta_len = read_file("weather-agent.json", meta, sizeof(meta));
    airy_agent_t agent;
    airy_token_budget_t budget = { .hard_limit = 4096, .soft_limit = 3072,
                                      .replenish_rate = 50 };

    int rc = airy_agent_create(meta, meta_len, &agent);          /* CREATED */
    if (rc) return rc;
    rc = airy_agent_register(agent);                            /* REGISTERED */
    if (rc) goto destroy;
    rc = airy_agent_schedule(agent, 100 /* 交互响应 */, budget);/* SCHEDULED */
    if (rc) goto destroy;

    /* 运行时调度器选中后进入 RUNNING，执行 weather_agent_main */
    /* 完成后自动进入 TERMINATED */
destroy:
    airy_agent_destroy(agent);
    return rc;
}
```

---

## 八、兼容性测试要求

实现本标准的运行时**必须**通过以下测试用例（详细测试代码随测试套件发布）：

| 测试 ID | 测试内容 | 通过条件 |
|---------|---------|---------|
| RT-T-001 | 6 状态全路径覆盖 | 所有合法转移成功，所有非法转移返回 EINVAL |
| RT-T-002 | 元数据 Schema 校验 | 缺少必填字段返回 EINVAL |
| RT-T-003 | agent_id 唯一性 | 重复注册返回 EEXIST |
| RT-T-004 | 最小权限授予 | Agent 无法访问未声明的能力 |
| RT-T-005 | Token 硬上限 | 超过 hard_limit 终止并返回 EBUDGET |
| RT-T-006 | 预算继承 | 子预算耗尽不影响父预算，父耗尽终止所有子 |
| RT-T-007 | 未知字段忽略 | 元数据含未知字段不报错 |
| RT-T-008 | 沙箱默认级 | 未指定 sandbox_level 时默认 2 |

---

## 九、错误码

本标准使用的错误码取自 Airymax 统一错误码体系（详见 03 安全模型标准第 4 节）：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| `AIRY_EOK` | 0 | 成功 |
| `AIRY_EINVAL` | -2 | 参数非法（含非法状态转移） |
| `AIRY_ENOMEM` | -3 | 内存不足 |
| `AIRY_EPERM` | -22 | 权限不足 |
| `AIRY_EEXIST` | -17 | 已存在 |
| `AIRY_EBUSY` | -16 | 资源忙 |
| `AIRY_EBUDGET` | -1004 | Token 预算耗尽 |

---

## 十、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-07-09 | 初始草案：生命周期 6 状态、元数据 JSON Schema、能力声明、Token 预算契约 |

---

## 十一、参考文献

- [Airymax 开放标准体系总览](./README.md)
- [Airymax IPC 协议标准](./02-airy-ipc-protocol-standard.md)
- [Airymax 安全模型标准](./03-airy-security-model-standard.md)
- [Airymax 调度标准](./06-airy-scheduling-standard.md)
- JSON Schema Draft 2020-12：https://json-schema.org/
- RFC 2119：https://www.rfc-editor.org/rfc/rfc2119
- 语义化版本：https://semver.org/

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
