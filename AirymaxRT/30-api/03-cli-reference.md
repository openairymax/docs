# Airymax CLI 命令参考手册

> **P4.3.5** — CLI 命令参考手册，涵盖 Airymax 统一命令行工具的所有可用命令。
>
> 版本：Airymax v2.1.0 | CLI 入口：`scripts/dev/cli/agentrt`

---

## 目录

- [1. 概述](#1-概述)
- [2. 环境变量](#2-环境变量)
- [3. Service Management（服务管理）](#3-service-management服务管理)
- [4. Agent Management（Agent 管理）](#4-agent-managementagent-管理)
- [5. Task Management（任务管理）](#5-task-management任务管理)
- [6. Protocol Operations（协议操作）](#6-protocol-operations协议操作)
- [7. Configuration（配置管理）](#7-configuration配置管理)
- [8. Monitoring（监控）](#8-monitoring监控)
- [9. Development（开发工具）](#9-development开发工具)
- [10. Database（数据库操作）](#10-database数据库操作)
- [11. System（系统命令）](#11-system系统命令)

---

## 1. 概述

`agentrt` 是 Airymax 的统一命令行入口，通过 REST API 和 JSON-RPC 2.0 与 Airymax Gateway 通信。所有命令的默认 Gateway 地址为 `http://localhost:8080`。

### 基本用法

```bash
agentrt <command> <subcommand> [options] [arguments]
```

### 全局选项

| 选项 | 说明 |
|:-----|:-----|
| `-h`, `--help` | 显示帮助信息（适用于任何层级） |

直接运行 `agentrt`（不带参数）将显示所有可用命令的概览。

---

## 2. 环境变量

以下环境变量会影响 CLI 的行为：

| 环境变量 | 默认值 | 说明 |
|:---------|:-------|:-----|
| `AGENTRT_ENDPOINT` | `http://localhost:8080` | Airymax Gateway 的 HTTP 地址 |
| `AGENTRT_API_KEY` | （空） | API 认证密钥，设置后将作为 `Authorization: Bearer` 头传递 |
| `AGENTRT_HEAPSTORE_ROOT` | `~/.agentrt/heapstore` | Heapstore 数据根目录，用于 `db` 相关命令 |

### 示例：设置环境变量

```bash
# 连接到远程 Gateway
export AGENTRT_ENDPOINT="http://agentrt.example.com:8080"

# 设置 API 认证
export AGENTRT_API_KEY="sk-xxxxxxxxxxxx"

# 自定义 Heapstore 数据目录
export AGENTRT_HEAPSTORE_ROOT="/data/agentrt/heapstore"
```

---

## 3. Service Management（服务管理）

用于管理 Airymax 平台上的微服务组件的生命周期。

### 3.1 `service list` — 列出所有服务

列出当前注册的所有服务及其运行状态。

**用法：**

```bash
agentrt service list
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Services
============================================================

Name                 Status       Healthy    Port
--------------------------------------------------
gateway              running      ✓          8080
task-scheduler       running      ✓          18791
memory-store         running      ✓          18792
```

**环境变量：** 无额外变量。

---

### 3.2 `service start` — 启动服务

启动指定的服务或所有服务。

**用法：**

```bash
agentrt service start [name]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `name` | 可选 | 要启动的服务名称。省略时启动所有服务。 |

**示例：**

```bash
# 启动所有服务
agentrt service start

# 启动指定服务
agentrt service start agent-manager
```

**预期输出：**

```
→ Starting service 'agent-manager'...
✓ Service 'agent-manager' started successfully
```

**环境变量：** 无额外变量。

---

### 3.3 `service stop` — 停止服务

停止指定的服务或所有服务。

**用法：**

```bash
agentrt service stop [name]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `name` | 可选 | 要停止的服务名称。省略时停止所有服务。 |

**示例：**

```bash
# 停止所有服务
agentrt service stop

# 停止指定服务
agentrt service stop memory-store
```

**预期输出：**

```
→ Stopping service 'memory-store'...
✓ Service 'memory-store' stopped
```

**环境变量：** 无额外变量。

---

### 3.4 `service restart` — 重启服务

重启指定的服务或所有服务。

**用法：**

```bash
agentrt service restart [name]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `name` | 可选 | 要重启的服务名称。省略时重启所有服务。 |

**示例：**

```bash
# 重启所有服务
agentrt service restart

# 重启指定服务
agentrt service restart gateway
```

**预期输出：**

```
→ Restarting service 'gateway'...
✓ Service 'gateway' restarted successfully
```

**环境变量：** 无额外变量。

---

### 3.5 `service status` — 获取服务状态

查看指定服务的详细状态信息。

**用法：**

```bash
agentrt service status [name]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `name` | 可选 | 服务名称。省略时显示所有服务状态。 |

**示例：**

```bash
agentrt service status task-scheduler
```

**预期输出：**

```
============================================================
  Service Status: task-scheduler
============================================================

  Status:      running
  Healthy:     ✓
  Port:        18791
  Uptime:      2h 34m 12s
  Version:     2.1.0
  CPU:         2.3%
  Memory:      128MB / 512MB
```

**环境变量：** 无额外变量。

---

### 3.6 `service health` — 健康检查

对所有服务执行健康检查，调用 Gateway 的 `/health` 端点。

**用法：**

```bash
agentrt service health
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Health Check
============================================================

✓ Gateway is healthy (v2.1.0, uptime: 9240s)
```

**错误输出示例：**

```
============================================================
  Airymax Health Check
============================================================

✗ Health check failed: Connection failed: [Errno 111] Connection refused
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定健康检查的目标地址。

---

## 4. Agent Management（Agent 管理）

用于管理 Airymax 平台上的 AI Agent 实例。

### 4.1 `agent list` — 列出已注册的 Agent

列出所有已注册的 Agent 及其基本信息。

**用法：**

```bash
agentrt agent list
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Agents
============================================================

ID           Name                 Type            Status
-----------------------------------------------------------
a1b2c3d4e5   code-reviewer        llm             idle
f6g7h8i9j0   data-analyzer        tool-calling    running
k1l2m3n4o5   web-researcher       mcp             idle
```

**环境变量：** 无额外变量。

---

### 4.2 `agent register` — 注册新 Agent

注册一个新的 Agent 到平台。

**用法：**

```bash
agentrt agent register <name>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `name` | 必填 | 新 Agent 的名称 |

**示例：**

```bash
agentrt agent register my-custom-agent
```

**预期输出：**

```
→ Registering agent 'my-custom-agent'...
✓ Agent 'my-custom-agent' registered with ID: p6q7r8s9t0
```

**环境变量：** 无额外变量。

---

### 4.3 `agent start` — 启动 Agent

启动一个已注册的 Agent。

**用法：**

```bash
agentrt agent start <id>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `id` | 必填 | Agent 的唯一标识符 |

**示例：**

```bash
agentrt agent start a1b2c3d4e5
```

**预期输出：**

```
→ Starting agent 'a1b2c3d4e5'...
✓ Agent 'a1b2c3d4e5' started successfully
```

**环境变量：** 无额外变量。

---

### 4.4 `agent stop` — 停止 Agent

停止一个正在运行的 Agent。

**用法：**

```bash
agentrt agent stop <id>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `id` | 必填 | Agent 的唯一标识符 |

**示例：**

```bash
agentrt agent stop f6g7h8i9j0
```

**预期输出：**

```
→ Stopping agent 'f6g7h8i9j0'...
✓ Agent 'f6g7h8i9j0' stopped
```

**环境变量：** 无额外变量。

---

### 4.5 `agent info` — 获取 Agent 详细信息

查看指定 Agent 的详细配置和状态信息。

**用法：**

```bash
agentrt agent info <id>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `id` | 必填 | Agent 的唯一标识符 |

**示例：**

```bash
agentrt agent info a1b2c3d4e5
```

**预期输出：**

```
============================================================
  Agent Info: a1b2c3d4e5
============================================================

  Name:        code-reviewer
  Type:        llm
  Status:      idle
  Model:       gpt-4o
  Created:     2026-06-01 10:30:00
  Last Active: 2026-06-18 14:22:15
  Tasks:       42 completed / 0 pending
```

**环境变量：** 无额外变量。

---

## 5. Task Management（任务管理）

用于管理 Airymax 平台上的任务生命周期。

### 5.1 `task list` — 列出任务

列出所有任务及其状态。

**用法：**

```bash
agentrt task list
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Tasks
============================================================

ID           Agent           Status       Progress
-------------------------------------------------
t1a2b3c4d5   a1b2c3d4e5     completed    100%
t6e7f8g9h0   f6g7h8i9j0     running      45%
t1i2j3k4l5   k1l2m3n4o5     pending      0%
```

**环境变量：** 无额外变量。

---

### 5.2 `task submit` — 提交新任务

提交一个新任务到平台。

**用法：**

```bash
agentrt task submit <desc>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `desc` | 必填 | 任务描述（自然语言或 JSON 格式） |

**示例：**

```bash
# 提交自然语言任务
agentrt task submit "Review the latest PR for security vulnerabilities"

# 提交 JSON 格式任务
agentrt task submit '{"type":"code_review","target":"PR#1234","rules":["security","style"]}'
```

**预期输出：**

```
→ Submitting task...
✓ Task submitted successfully
  Task ID:  t1m2n3o4p5
  Status:   pending
```

**环境变量：** 无额外变量。

---

### 5.3 `task status` — 获取任务状态

查看指定任务的当前状态和进度。

**用法：**

```bash
agentrt task status <id>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `id` | 必填 | 任务的唯一标识符 |

**示例：**

```bash
agentrt task status t6e7f8g9h0
```

**预期输出：**

```
============================================================
  Task Status: t6e7f8g9h0
============================================================

  Agent:       data-analyzer (f6g7h8i9j0)
  Status:      running
  Progress:    45%
  Created:     2026-06-18 14:00:00
  Started:     2026-06-18 14:00:05
  Elapsed:     22m 10s
```

**环境变量：** 无额外变量。

---

### 5.4 `task cancel` — 取消任务

取消一个正在运行或等待中的任务。

**用法：**

```bash
agentrt task cancel <id>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `id` | 必填 | 任务的唯一标识符 |

**示例：**

```bash
agentrt task cancel t6e7f8g9h0
```

**预期输出：**

```
→ Cancelling task 't6e7f8g9h0'...
✓ Task 't6e7f8g9h0' cancelled
```

**环境变量：** 无额外变量。

---

### 5.5 `task result` — 获取任务结果

获取已完成任务的结果。

**用法：**

```bash
agentrt task result <id>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `id` | 必填 | 任务的唯一标识符 |

**示例：**

```bash
agentrt task result t1a2b3c4d5
```

**预期输出：**

```
============================================================
  Task Result: t1a2b3c4d5
============================================================

  Status:      completed
  Duration:    1m 34s
  Result:
  {
    "findings": [
      {"severity": "medium", "file": "src/auth.py", "line": 42, "message": "..."},
      {"severity": "low", "file": "src/utils.py", "line": 108, "message": "..."}
    ],
    "summary": "2 issues found (0 critical, 1 medium, 1 low)"
  }
```

**环境变量：** 无额外变量。

---

## 6. Protocol Operations（协议操作）

Airymax 支持多种协议适配器，包括 JSON-RPC 2.0、MCP v1.0、A2A v0.3、OpenAI API Compatible 和 OpenJiuwen（自定义二进制协议）。本节命令用于管理、测试和操作这些协议。

### 6.1 `protocol list` — 列出所有协议适配器

列出当前可用的所有协议适配器。

**用法：**

```bash
agentrt protocol list
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Protocol Adapters
============================================================

Protocol        Version    Status     Endpoint
-----------------------------------------------------
JSONRPC         2.0        active     /jsonrpc
MCP             1.0        active     /mcp
A2A             0.3        active     /a2a
OPENAI          1.0        active     /v1
OPENJIUWEN      1.0        active     /ojiuwen
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定查询的 Gateway 地址。如果 Gateway 不可达，CLI 将显示默认协议列表。

---

### 6.2 `protocol info` — 显示协议详细信息

显示指定协议的版本、端点、传输方式、支持的方法和特性等详细信息。

**用法：**

```bash
agentrt protocol info <proto>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `proto` | 必填 | 协议名称。支持别名：`jsonrpc` / `json-rpc` / `json`、`mcp` / `model-context-protocol`、`a2a` / `agent-to-agent` / `agent2agent`、`openai` / `openai-api` / `openai-compatible`、`openjiuwen` / `ojiuwen` / `ojw` / `binary` |

**示例：**

```bash
agentrt protocol info jsonrpc
```

**示例输出（JSON-RPC）：**

```
============================================================
  Protocol: JSON-RPC 2.0
============================================================

  Version:       2.0
  Endpoint:      /jsonrpc
  Content-Type:  application/json
  Transport:     HTTP/WebSocket

  Methods:
    - task.submit
    - task.status
    - task.result
    - memory.write
    - memory.search
    - session.create
    - skill.load
    - service.list
    - agent.list
    - ping

  Features:
    - batch requests
    - notifications
    - idempotency
    - streaming (SSE)

  Specification: https://www.jsonrpc.org/specification
```

**示例输出（OpenJiuwen）：**

```bash
agentrt protocol info openjiuwen
```

```
============================================================
  Protocol: OpenJiuwen (Custom Binary)
============================================================

  Version:       1.0
  Endpoint:      /ojiuwen
  Content-Type:  application/octet-stream
  Transport:     TCP/WebSocket (binary)

  Binary Format:
    Magic Number: 0x4F4A574D (OJWM)
    Header Size:  24 bytes

  Methods:
    - request
    - response
    - notification
    - heartbeat

  Features:
    - binary protocol
    - CRC32 integrity
    - 24-byte header
    - low latency
    - sequence numbers

  Specification: （内部规范）
```

**环境变量：** 无额外变量。

---

### 6.3 `protocol test` — 测试协议连通性

测试指定协议与 Gateway 的连通性，显示延迟和版本信息。

**用法：**

```bash
agentrt protocol test <proto> [-v|--verbose]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `proto` | 必填 | 协议名称（支持别名，同 `protocol info`） |

**选项：**

| 选项 | 说明 |
|:-----|:-----|
| `-v`, `--verbose` | 显示原始响应 JSON |

**示例：**

```bash
# 基础测试
agentrt protocol test jsonrpc

# 详细模式
agentrt protocol test openai -v
```

**示例输出：**

```
============================================================
  Testing JSON-RPC 2.0
============================================================

→ Connecting to http://localhost:8080/jsonrpc ...
✓ Connected successfully (12ms, version: 2.0)
```

**详细模式输出：**

```
============================================================
  Testing OpenAI API Compatible
============================================================

→ Connecting to http://localhost:8080/v1/models ...
✓ Connected successfully (34ms, version: 1.0)

Raw response:
{
  "object": "list",
  "data": [
    {"id": "gpt-4o", "object": "model", "created": 1700000000, "owned_by": "agentrt"},
    {"id": "gpt-4o-mini", "object": "model", "created": 1700000000, "owned_by": "agentrt"}
  ]
}
```

**错误输出示例：**

```
============================================================
  Testing JSON-RPC 2.0
============================================================

→ Connecting to http://localhost:8080/jsonrpc ...
✗ Connection failed (15000ms)
→ Ensure gateway is running at http://localhost:8080
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定测试的目标地址。

---

### 6.4 `protocol detect` — 检测协议类型

根据输入内容自动检测其所属的协议类型。

**用法：**

```bash
agentrt protocol detect --content <json_string> [--content-type <mime_type>]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `--content`, `-c` | 可选 | 要检测的 JSON 内容字符串。如不指定，则从 stdin 读取。 |
| `--content-type`, `-t` | 可选 | MIME 内容类型提示，可提高检测准确度。 |

**示例：**

```bash
# 检测 JSON-RPC 请求
agentrt protocol detect -c '{"jsonrpc":"2.0","method":"task.submit","params":{},"id":1}'

# 检测 OpenAI 请求
agentrt protocol detect -c '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}],"temperature":0.7}'

# 通过管道从 stdin 输入
echo '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"web_search"}}' | agentrt protocol detect
```

**示例输出：**

```
============================================================
  Protocol Detection
============================================================

  Input size:     98 bytes
  Content-Type:   (not specified)

  Detected:      JSON-RPC (95.0% confidence)

  Alternatives:
    MCP             [████████████████████████████░░] 90.0%
```

**内容类型映射：**

| Content-Type | 检测协议 |
|:-------------|:---------|
| `application/json-rpc` | JSON-RPC (98%) |
| `application/mcp` | MCP (97%) |
| `application/a2a` | A2A (96%) |
| `application/openai+json` | OpenAI (95%) |
| `application/octet-stream` | OpenJiuwen (90%) |

**环境变量：** 无额外变量。

---

### 6.5 `protocol send` — 发送协议消息

通过指定协议发送一个方法调用，并显示响应。

**用法：**

```bash
agentrt protocol send <proto> <method> [params]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `proto` | 必填 | 目标协议（支持别名，同 `protocol info`） |
| `method` | 必填 | 方法名或端点路径。对于 JSON-RPC/MCP 为方法名，对于 OpenAI 为 API 路径（如 `chat/completions`），对于 A2A 为消息类型。 |
| `params` | 可选 | JSON 格式的参数字符串 |

**示例：**

```bash
# 通过 JSON-RPC 发送 ping
agentrt protocol send jsonrpc ping

# 通过 JSON-RPC 提交任务
agentrt protocol send jsonrpc task.submit '{"description":"Analyze code quality"}'

# 通过 OpenAI 协议发送聊天请求
agentrt protocol send openai chat/completions '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'

# 通过 A2A 发送发现请求
agentrt protocol send a2a discovery_request
```

**示例输出：**

```
============================================================
  Sending via JSON-RPC 2.0
============================================================

→ Method: ping

Response:
{
  "jsonrpc": "2.0",
  "result": {
    "message": "pong",
    "timestamp": "2026-06-18T14:30:00Z"
  },
  "id": "cli-a1b2c3d4"
}
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定消息发送的目标 Gateway。

---

### 6.6 `protocol stats` — 显示协议统计信息

显示各协议的使用统计信息，包括请求量、错误率、延迟等。

**用法：**

```bash
agentrt protocol stats
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Protocol Statistics
============================================================

  Total Requests:   15,234
  Transforms:       1,208
  Transform Errors: 3
  Avg Latency:      23.5ms

  By Protocol:
  Protocol        Requests   Errors    Share
  --------------- ---------- -------- --------
  JSONRPC             6,521       12    42.8%
  OPENAI              4,102        5    26.9%
  MCP                 2,891        8    19.0%
  A2A                 1,420        2     9.3%
  OPENJIUWEN            300        1     2.0%
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定查询统计信息的目标 Gateway。如果 Gateway 不可达，将显示零值统计。

---

### 6.7 `protocol transform` — 协议间转换

演示两种协议之间的请求格式转换。

**用法：**

```bash
agentrt protocol transform <src> <tgt> [data]
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `src` | 必填 | 源协议（支持别名） |
| `tgt` | 必填 | 目标协议（支持别名） |
| `data` | 可选 | 自定义输入数据，用于演示转换效果 |

**支持的转换对：**

| 源 → 目标 | 说明 |
|:----------|:-----|
| JSON-RPC → MCP | `tool.execute` → `tools/call`（params → arguments 映射） |
| JSON-RPC → OpenAI | `llm.chat` → `chat/completions`（prompt → messages 映射） |
| JSON-RPC → A2A | `task.delegate` → `task_delegation` 类型 |
| MCP → JSON-RPC | `tools/call` → `tool.execute`（arguments → params 映射） |
| OpenAI → JSON-RPC | `chat/completions` → `llm.chat`（messages → prompt 提取） |
| JSON-RPC ↔ OpenJiuwen | 双向转换 |
| 其他组合 | 通过统一 transformer pipeline 支持 |

**示例：**

```bash
# 演示 JSON-RPC 到 OpenAI 的转换
agentrt protocol transform jsonrpc openai

# 演示 JSON-RPC 到 MCP 的转换
agentrt protocol transform jsonrpc mcp

# 带自定义数据
agentrt protocol transform openai jsonrpc '{"model":"gpt-4o","messages":[{"role":"user","content":"Hi"}]}'
```

**示例输出（JSON-RPC → MCP）：**

```
============================================================
  Protocol Transform: JSONRPC → MCP
============================================================

  Sample Input (JSONRPC):
  {
      "jsonrpc": "2.0",
      "method": "tool.execute",
      "params": {
          "name": "web_search",
          "query": "Airymax"
      },
      "id": 1
  }

  Expected Transformation:
  tool.execute → tools/call (params → arguments mapping)

  Available transformations:
    ✓ JSON-RPC ↔ MCP
    ✓ JSON-RPC ↔ A2A
    ✓ JSON-RPC ↔ OpenAI
      JSON-RPC ↔ OpenJiuwen
      MCP ↔ JSON-RPC
      A2A ↔ JSON-RPC
      OpenAI ↔ JSON-RPC
      OpenJiuwen ↔ JSON-RPC
```

**环境变量：** 无额外变量。

---

### 6.8 `protocol capabilities` — 显示协议能力

显示指定协议支持的所有方法、特性和模型。

**用法：**

```bash
agentrt protocol capabilities <proto>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `proto` | 必填 | 协议名称（支持别名，同 `protocol info`） |

**示例：**

```bash
agentrt protocol capabilities openai
```

**示例输出：**

```
============================================================
  Capabilities: OpenAI API Compatible
============================================================

  Supported Methods (4):
     1. /v1/chat/completions
     2. /v1/embeddings
     3. /v1/models
     4. /v1/completions

  Features (4):
    - chat completions
    - embeddings
    - streaming (SSE)
    - function calling

  Available Models (6):
    - gpt-4o
    - gpt-4o-mini
    - gpt-4-turbo
    - text-embedding-ada-002
    - text-embedding-3-small
    - text-embedding-3-large

  Transport Details:
    Endpoint:     http://localhost:8080/v1
    Content-Type: application/json
    Transport:    HTTP (SSE)
```

**环境变量：** `AGENTRT_ENDPOINT` — 影响显示的端点地址。

---

## 7. Configuration（配置管理）

用于读取和修改 Airymax 运行时配置。

### 7.1 `config get` — 获取配置值

获取指定配置键的值。通过 JSON-RPC `config.get` 方法查询。

**用法：**

```bash
agentrt config get <key>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `key` | 必填 | 配置键名 |

**示例：**

```bash
agentrt config get gateway.port
agentrt config get memory.max_size
agentrt config get logging.level
```

**示例输出：**

```
gateway.port = 8080
```

**错误输出示例：**

```
✗ Failed to get config 'gateway.port': Config key not found
```

**环境变量：** 无额外变量。

---

### 7.2 `config set` — 设置配置值

设置指定配置键的值。通过 JSON-RPC `config.set` 方法更新。

**用法：**

```bash
agentrt config set <key> <value>
```

**参数：**

| 参数 | 类型 | 说明 |
|:-----|:-----|:-----|
| `key` | 必填 | 配置键名 |
| `value` | 必填 | 配置值 |

**示例：**

```bash
agentrt config set logging.level debug
agentrt config set gateway.max_connections 1000
agentrt config set memory.ttl 3600
```

**示例输出：**

```
✓ Set logging.level = debug
```

**环境变量：** 无额外变量。

---

### 7.3 `config list` — 列出所有配置

列出所有当前的配置项。

**用法：**

```bash
agentrt config list
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Configuration
============================================================

  gateway.port:             8080
  gateway.host:             0.0.0.0
  gateway.max_connections:  100
  logging.level:            info
  logging.format:           json
  memory.max_size:          1073741824
  memory.ttl:               7200
  task.max_concurrent:      10
  task.default_timeout:     300
```

**环境变量：** 无额外变量。

---

### 7.4 `config reload` — 重新加载配置

重新加载配置文件，使更改生效而无需重启 Gateway。

**用法：**

```bash
agentrt config reload
```

**选项/参数：** 无

**示例输出：**

```
→ Reloading configuration...
✓ Configuration reloaded successfully
```

**环境变量：** 无额外变量。

---

## 8. Monitoring（监控）

用于查看 Airymax 平台的运行时指标和告警。

### 8.1 `monitor metrics` — 查看指标

显示当前运行时指标，包括请求量、延迟、错误率、资源使用率等。

**用法：**

```bash
agentrt monitor metrics
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Metrics
============================================================

  Gateway:
    Requests/sec:     42.3
    Avg Latency:      23.5ms
    P99 Latency:      145.2ms
    Error Rate:       0.12%

  Memory Store:
    Cache Hit Rate:   94.7%
    Total Entries:    12,456
    Memory Usage:     256MB / 1GB

  Task Scheduler:
    Active Tasks:     3
    Queued Tasks:     7
    Completed (24h):  1,234
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定查询指标的目标 Gateway。

---

### 8.2 `monitor alerts` — 查看活跃告警

显示当前活跃的告警。

**用法：**

```bash
agentrt monitor alerts
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Active Alerts
============================================================

  ⚠ WARNING  gateway.latency_p99 > 200ms (current: 245ms)
  ⚠ WARNING  memory.usage > 80% (current: 85%)
  ✓ No critical alerts

  Total: 2 active alerts
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定查询告警的目标 Gateway。

---

### 8.3 `monitor dashboard` — 打开监控面板

打开 Grafana 监控面板的 URL。

**用法：**

```bash
agentrt monitor dashboard
```

**选项/参数：** 无

**示例输出：**

```
→ Opening Grafana dashboard...
  URL: http://localhost:3000/d/agentrt/agentrt-overview
```

**环境变量：** 无额外变量。

---

## 9. Development（开发工具）

用于 Airymax 项目的开发工作流。

### 9.1 `dev init` — 初始化开发环境

初始化 Airymax 的开发环境，包括依赖安装、子模块初始化和构建目录配置。

**用法：**

```bash
agentrt dev init
```

**选项/参数：** 无

**示例输出：**

```
→ Initializing development environment...
✓ CMake 3.20 found
✓ GCC 11 found
✓ Python 3.10 found
✓ Dependencies installed
✓ Submodules initialized
✓ Build directory configured
✓ Development environment ready
```

**环境变量：** 无额外变量。

---

### 9.2 `dev build` — 构建项目

编译 Airymax 项目。

**用法：**

```bash
agentrt dev build
```

**选项/参数：** 无

**示例输出：**

```
→ Building Airymax...
  Build type: Release
  Generator:  Ninja
  Parallel:   8 jobs

[100/100] Linking CXX executable agentrt-gateway
✓ Build completed successfully (2m 34s)
```

**环境变量：** 无额外变量。

---

### 9.3 `dev test` — 运行测试

执行测试套件。

**用法：**

```bash
agentrt dev test
```

**选项/参数：** 无

**示例输出：**

```
→ Running tests...
  Test suite: All

[==========] Running 156 tests from 23 test suites.
[==========] 156 tests passed (12.3s total)

✓ All tests passed
```

**环境变量：** 无额外变量。

---

### 9.4 `dev doctor` — 运行诊断

检查开发环境中的工具链是否齐全，诊断 Gateway 是否可达。

**用法：**

```bash
agentrt dev doctor
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax Diagnostics
============================================================

Tool                      Status
-----------------------------------
✓ CMake                    Found
✓ C Compiler (GCC)         Found
✓ C++ Compiler (G++)       Found
✓ Python 3                 Found
⚠ Docker                   Not found (optional)
⚠ Node.js                  Not found (optional)
⚠ Rust/Cargo               Not found (optional)
✓ Gateway                  Running (http://localhost:8080)

✓ All required tools available
```

**部分缺失示例：**

```
============================================================
  Airymax Diagnostics
============================================================

Tool                      Status
-----------------------------------
✗ CMake                    Missing (required)
✓ C Compiler (GCC)         Found
✓ C++ Compiler (G++)       Found
✓ Python 3                 Found
⚠ Docker                   Not found (optional)
⚠ Gateway                  Not running

✗ 1 required tool(s) missing
```

**检查项说明：**

| 工具 | 是否必需 | 说明 |
|:-----|:---------|:-----|
| CMake | 是 | 构建系统 |
| GCC (C Compiler) | 是 | C 编译器 |
| G++ (C++ Compiler) | 是 | C++ 编译器 |
| Python 3 | 是 | Python 运行时 |
| Docker | 否 | 容器化部署 |
| Node.js | 否 | TypeScript SDK |
| Rust/Cargo | 否 | Rust SDK |
| Gateway | — | Airymax Gateway 是否在运行 |

**环境变量：** `AGENTRT_ENDPOINT` — 决定 Gateway 健康检查的目标地址。

---

### 9.5 `dev shell` — 打开开发 Shell

打开一个配置好 Airymax 开发环境的 Shell 会话。

**用法：**

```bash
agentrt dev shell
```

**选项/参数：** 无

**示例输出：**

```
→ Opening development shell...
  Source tree:  /home/user/agentrt
  Build dir:    /home/user/AgentRT-build
  PATH:         .../scripts/dev/cli:...

[agentrt-dev] $
```

**环境变量：** 无额外变量。

---

## 10. Database（数据库操作）

用于管理 Heapstore 的 Schema 迁移和数据备份。

### 10.1 `db migrate` — 执行 Schema 迁移

执行 Heapstore 的 Schema 版本迁移。默认从 v1 升级到 v2。

**用法：**

```bash
agentrt db migrate [--target=<version>]
```

**选项：**

| 选项 | 默认值 | 说明 |
|:-----|:-------|:-----|
| `--target` | `v2` | 目标 Schema 版本 |

**示例：**

```bash
# 迁移到默认版本 v2
agentrt db migrate

# 迁移到指定版本
agentrt db migrate --target=v2
```

**示例输出（需要迁移）：**

```
============================================================
  Airymax Database Migration
============================================================

→ Target version: v2
→ Data directory: /home/user/.agentrt/heapstore
→ Current version: v1 (1)
→ Running forward migration (v1 → v2)...
✓ Migration completed successfully (2340ms)
→ Records migrated: 12,456
→ Backup saved at: /home/user/.agentrt/heapstore/backups/v1_20260618_143000

  Step                                     Records      Time
  ---------------------------------------- -------- ----------
  add_metadata_fields                           456       890ms
  add_embedding_column                         2000      1200ms
  create_search_index                         10000       250ms
```

**示例输出（无需迁移）：**

```
============================================================
  Airymax Database Migration
============================================================

→ Target version: v2
→ Data directory: /home/user/.agentrt/heapstore
→ Current version: v2 (2)
✓ Schema is already up to date. No migration needed.
```

**环境变量：**

| 环境变量 | 说明 |
|:---------|:-----|
| `AGENTRT_HEAPSTORE_ROOT` | 指定 Heapstore 数据目录（默认：`~/.agentrt/heapstore`） |

---

### 10.2 `db migrate-status` — 查看迁移状态

查看当前 Heapstore 的 Schema 版本和迁移状态。

**用法：**

```bash
agentrt db migrate-status
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Heapstore Migration Status
============================================================

→ Data directory: /home/user/.agentrt/heapstore

  Current version:  v2 (2)
  Latest version:   v2 (2)
  Status:           ✓ Up to date

  Backups:
    - v1_20260618_143000 (2026-06-18T14:30:00)
    - v1_20260617_090000 (2026-06-17T09:00:00)
```

**需要迁移时：**

```
  Current version:  v1 (1)
  Latest version:   v2 (2)
  Status:           ⚠ Migration needed
```

**环境变量：**

| 环境变量 | 说明 |
|:---------|:-----|
| `AGENTRT_HEAPSTORE_ROOT` | 指定 Heapstore 数据目录（默认：`~/.agentrt/heapstore`） |

---

### 10.3 `db migrate-rollback` — 回滚 Schema 迁移

将 Schema 回滚到上一个版本。出于安全考虑，必须使用 `--force` 标志。

**用法：**

```bash
agentrt db migrate-rollback --force
```

**选项：**

| 选项 | 说明 |
|:-----|:-----|
| `--force` | **必填**。确认回滚操作。回滚将丢弃新版本中新增的字段。 |

**示例：**

```bash
agentrt db migrate-rollback --force
```

**示例输出：**

```
============================================================
  Heapstore Migration Rollback
============================================================

⚠ Rolling back schema version...
→ Data directory: /home/user/.agentrt/heapstore
✓ Rollback completed (1890ms)
→ Records affected: 12,456
→ Fields removed: 3
→ Backup saved at: /home/user/.agentrt/heapstore/backups/v2_20260618_150000

  Step                                     Records   Fields
  ---------------------------------------- -------- --------
  remove_metadata_fields                       456        3
  remove_embedding_column                     2000        1
  drop_search_index                          10000        1
```

**未使用 --force 时的输出：**

```
============================================================
  Heapstore Migration Rollback
============================================================

✗ Rollback requires --force flag for safety.
→ This will discard new fields added in v2.
→ Usage: agentrt db migrate-rollback --force
```

**环境变量：**

| 环境变量 | 说明 |
|:---------|:-----|
| `AGENTRT_HEAPSTORE_ROOT` | 指定 Heapstore 数据目录（默认：`~/.agentrt/heapstore`） |

---

### 10.4 `db backup` — 创建数据备份

创建 Heapstore 数据的完整备份。

**用法：**

```bash
agentrt db backup
```

**选项/参数：** 无

**示例：**

```bash
agentrt db backup
```

**示例输出：**

```
============================================================
  Heapstore Data Backup
============================================================

→ Data directory: /home/user/.agentrt/heapstore
✓ Backup created: /home/user/.agentrt/heapstore/backups/manual_20260618_143000
```

**环境变量：**

| 环境变量 | 说明 |
|:---------|:-----|
| `AGENTRT_HEAPSTORE_ROOT` | 指定 Heapstore 数据目录（默认：`~/.agentrt/heapstore`） |

---

## 11. System（系统命令）

用于查看 Airymax 的版本和系统状态。

### 11.1 `version` — 显示版本

显示 Airymax CLI 和 Gateway 的版本信息。

**用法：**

```bash
agentrt version
```

**选项/参数：** 无

**示例输出：**

```
Airymax v2.1.0
  Gateway: http://localhost:8080
  Protocols: JSON-RPC 2.0, MCP v1.0, A2A v0.3, OpenAI API, OpenJiuwen (binary)
```

**环境变量：** `AGENTRT_ENDPOINT` — 影响显示的 Gateway 地址。

---

### 11.2 `status` — 显示系统状态

显示 Airymax Gateway 和所有服务的当前状态。通过调用 `/health` 端点获取信息。

**用法：**

```bash
agentrt status
```

**选项/参数：** 无

**示例输出：**

```
============================================================
  Airymax System Status
============================================================

✓ Gateway: healthy (v2.1.0)
  ✓ gateway: healthy
  ✓ agent-manager: healthy
  ✓ task-scheduler: healthy
  ✓ memory-store: healthy
```

**Gateway 不可达时：**

```
============================================================
  Airymax System Status
============================================================

✗ Gateway unreachable: Connection failed: [Errno 111] Connection refused
→ Expected at: http://localhost:8080
```

**环境变量：** `AGENTRT_ENDPOINT` — 决定状态查询的目标地址。

---

### 11.3 `help` — 显示帮助

显示所有可用命令的概览或特定命令的帮助信息。

**用法：**

```bash
# 显示所有命令概览
agentrt help

# 显示特定命令的帮助（等同于 -h 选项）
agentrt service --help
agentrt protocol --help
agentrt db --help
```

**选项/参数：** 无

**示例输出：**

```
usage: agentrt [-h] {service,agent,task,protocol,config,monitor,dev,db,version,status,help} ...

Airymax Unified CLI - 统一命令行工具

positional arguments:
  {service,agent,task,protocol,config,monitor,dev,db,version,status,help}
                        Available commands
    service             Service management
    agent               Agent management
    task                Task management
    protocol            Protocol operations
    config              Configuration management
    monitor             Monitoring
    dev                 Development tools
    db                  Database operations
    version             Show version
    status              Show system status
    help                Show help

optional arguments:
  -h, --help            show this help message and exit
```

**环境变量：** 无额外变量。

---

## 附录 A：退出码

| 退出码 | 含义 |
|:-------|:-----|
| `0` | 命令执行成功 |
| `1` | 命令执行失败（错误信息已输出到 stderr） |

---

## 附录 B：协议别名速查表

| 命令中使用的别名 | 解析为 |
|:-----------------|:-------|
| `jsonrpc`, `json-rpc`, `json` | `jsonrpc` |
| `mcp`, `model-context-protocol` | `mcp` |
| `a2a`, `agent-to-agent`, `agent2agent` | `a2a` |
| `openai`, `openai-api`, `openai-compatible` | `openai` |
| `openjiuwen`, `ojiuwen`, `ojw`, `binary` | `openjiuwen` |

---

## 附录 C：环境变量速查表

| 环境变量 | 默认值 | 影响的命令 |
|:---------|:-------|:-----------|
| `AGENTRT_ENDPOINT` | `http://localhost:8080` | 所有需要与 Gateway 通信的命令 |
| `AGENTRT_API_KEY` | （空） | 所有需要认证的 API 请求 |
| `AGENTRT_HEAPSTORE_ROOT` | `~/.agentrt/heapstore` | `db migrate`, `db migrate-status`, `db migrate-rollback`, `db backup` |