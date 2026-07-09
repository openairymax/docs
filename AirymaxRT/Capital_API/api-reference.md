# Airymax Gateway API 参考手册

> **P4.3.6** — Airymax Gateway HTTP/WebSocket 端点 API 参考，涵盖 JSON-RPC 2.0 方法、REST API、协议适配器端点、错误码、速率限制与分页。
>
> 版本：Airymax v2.1.0 | Gateway 默认端口：`8080`

---

## 目录

- [1. Gateway 概述](#1-gateway-概述)
- [2. 认证（Authentication）](#2-认证authentication)
- [3. JSON-RPC 2.0 方法](#3-json-rpc-20-方法)
  - [3.1 Task 管理（task.*）](#31-task-管理task)
  - [3.2 Agent 管理（agent.*）](#32-agent-管理agent)
  - [3.3 Memory 管理（memory.*）](#33-memory-管理memory)
  - [3.4 Session 管理（session.*）](#34-session-管理session)
  - [3.5 Skill 管理（skill.*）](#35-skill-管理skill)
  - [3.6 Service 管理（service.*）](#36-service-管理service)
  - [3.7 Config 管理（config.*）](#37-config-管理config)
  - [3.8 系统方法（ping）](#38-系统方法ping)
- [4. REST API 端点](#4-rest-api-端点)
- [5. 协议特定端点](#5-协议特定端点)
  - [5.1 MCP（Model Context Protocol）](#51-mcpmodel-context-protocol)
  - [5.2 A2A（Agent-to-Agent）](#52-a2aagent-to-agent)
  - [5.3 OpenAI 兼容 API](#53-openai-兼容-api)
  - [5.4 OpenJiuwen（二进制协议）](#54-openjiuwen二进制协议)
- [6. 错误码参考](#6-错误码参考)
- [7. 速率限制与分页](#7-速率限制与分页)
- [8. WebSocket 支持](#8-websocket-支持)

---

## 1. Gateway 概述

Airymax Gateway 是 Airymax 平台的统一接入层，负责将外部请求转换为内部系统调用。Gateway 基于 **JSON-RPC 2.0** 协议，默认监听端口 **8080**，同时支持 HTTP 和 WebSocket 两种传输方式。

### 架构定位

```
外部客户端 → Gateway（协议转换层） → agentos/atoms/syscall（系统调用层）
```

Gateway 只做协议转换，零业务逻辑，所有业务能力通过底层 syscall 接口暴露。

### 基本特性

| 特性 | 说明 |
|:-----|:-----|
| **协议** | JSON-RPC 2.0（Request/Response） |
| **默认端口** | `8080`（HTTP），`8081`（WebSocket） |
| **传输方式** | HTTP POST、WebSocket、Stdio |
| **Content-Type** | `application/json` |
| **认证方式** | API Key（Bearer Token / X-API-Key Header） |
| **速率限制** | 令牌桶算法，默认 100 req/s，突发 150 |
| **超时时间** | 默认 30,000 ms |

### 启动 Gateway

```bash
# 使用默认配置启动（HTTP: 8080, WS: 8081）
agentos-gateway

# 指定端口启动
agentos-gateway -h 0.0.0.0 -p 8080 -w 8081

# 从配置文件启动
agentos-gateway -c /etc/agentos/gateway.conf
```

---

## 2. 认证（Authentication）

Gateway 支持通过 API Key 进行请求认证。API Key 通过以下两种方式传递：

### 方式一：Bearer Token（推荐）

```bash
curl -X POST http://localhost:8080/ \
  -H "Authorization: Bearer sk-your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"ping","id":1}'
```

### 方式二：X-API-Key Header

```bash
curl -X POST http://localhost:8080/ \
  -H "X-API-Key: sk-your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"ping","id":1}'
```

### 环境变量

SDK 支持通过环境变量设置认证信息：

```bash
export AGENTOS_ENDPOINT="http://localhost:8080"
export AGENTOS_API_KEY="sk-your-api-key-here"
```

---

## 3. JSON-RPC 2.0 方法

所有 JSON-RPC 2.0 请求通过 `POST /` 端点发送。请求体格式遵循 [JSON-RPC 2.0 规范](https://www.jsonrpc.org/specification)：

```json
{
  "jsonrpc": "2.0",
  "method": "<方法名>",
  "params": { ... },
  "id": 1
}
```

### 3.1 Task 管理（task.*）

#### 3.1.1 task.submit — 提交任务

提交一个异步任务到系统执行。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "task.submit",
  "params": {
    "input": "任务输入数据（JSON 字符串）",
    "timeout_ms": 30000
  },
  "id": 1
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `input` | string | 是 | 任务输入数据，JSON 字符串格式 |
| `timeout_ms` | number | 否 | 超时时间（毫秒），默认 30000 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "task_id": "agentos-0000000000000001-00000001",
    "status": 1,
    "message": "Task accepted and queued"
  },
  "id": 1
}
```

| 字段 | 类型 | 说明 |
|:-----|:-----|:-----|
| `task_id` | string | 任务唯一标识符 |
| `status` | number | 任务状态：1=排队中，2=已完成，3=失败，4=已取消 |
| `message` | string | 状态描述 |

**错误处理：**

| 错误码 | 说明 |
|:-------|:-----|
| `-32602` | Invalid params：`input` 参数缺失或无效 |
| `-32000` | System call failed：底层系统调用失败 |

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "task.submit",
    "params": {
      "input": "{\"action\":\"analyze\",\"data\":\"sample\"}",
      "timeout_ms": 30000
    },
    "id": 1
  }'
```

---

#### 3.1.2 task.status — 查询任务状态

查询指定任务的当前状态。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "task.status",
  "params": {
    "task_id": "agentos-0000000000000001-00000001"
  },
  "id": 2
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `task_id` | string | 是 | 任务唯一标识符 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": 1
  },
  "id": 2
}
```

| 字段 | 类型 | 说明 |
|:-----|:-----|:-----|
| `status` | number | 任务状态：1=排队中，2=已完成，3=失败，4=已取消；-1=不存在 |

**错误处理：**

| 错误码 | 说明 |
|:-------|:-----|
| `-32602` | Invalid params：`task_id` 参数缺失或无效 |
| `-32000` | System call failed：任务不存在 |

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "task.status",
    "params": {"task_id": "agentos-0000000000000001-00000001"},
    "id": 2
  }'
```

---

#### 3.1.3 task.result — 获取任务结果

等待并获取任务的执行结果。此方法会阻塞直到任务完成或超时。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "task.result",
  "params": {
    "task_id": "agentos-0000000000000001-00000001",
    "timeout_ms": 30000
  },
  "id": 3
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `task_id` | string | 是 | 任务唯一标识符 |
| `timeout_ms` | number | 否 | 等待超时时间（毫秒），默认 30000 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "task_id": "agentos-0000000000000001-00000001",
    "status": 2,
    "result": "{\"output\":\"processed\",\"exit_code\":0}"
  },
  "id": 3
}
```

| 字段 | 类型 | 说明 |
|:-----|:-----|:-----|
| `task_id` | string | 任务标识符 |
| `status` | number | 任务状态（2=已完成） |
| `result` | string | 任务执行结果（JSON 字符串） |

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "task.result",
    "params": {
      "task_id": "agentos-0000000000000001-00000001",
      "timeout_ms": 30000
    },
    "id": 3
  }'
```

---

#### 3.1.4 task.cancel — 取消任务

取消一个正在执行或排队中的任务。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "task.cancel",
  "params": {
    "task_id": "agentos-0000000000000001-00000001"
  },
  "id": 4
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "cancelled": true
  },
  "id": 4
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "task.cancel",
    "params": {"task_id": "agentos-0000000000000001-00000001"},
    "id": 4
  }'
```

---

#### 3.1.5 task.list — 列出所有任务

列出系统中所有已提交的任务。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "task.list",
  "params": {
    "limit": 20,
    "offset": 0
  },
  "id": 5
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `limit` | number | 否 | 返回数量上限，默认 20 |
| `offset` | number | 否 | 分页偏移量，默认 0 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "tasks": [
      {
        "task_id": "agentos-0000000000000001-00000001",
        "status": 2,
        "created_at": 1718700000
      }
    ],
    "total": 1
  },
  "id": 5
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "task.list",
    "params": {"limit": 20, "offset": 0},
    "id": 5
  }'
```

---

### 3.2 Agent 管理（agent.*）

#### 3.2.1 agent.list — 列出所有 Agent

列出系统中所有已注册的 Agent。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "agent.list",
  "params": {},
  "id": 1
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent_ids": [
      "agentos-0000000000000001-00000003",
      "agentos-0000000000000001-00000004"
    ],
    "total": 2
  },
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"agent.list","params":{},"id":1}'
```

---

#### 3.2.2 agent.register — 注册 Agent

注册一个新的 Agent 实例到系统中。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "agent.register",
  "params": {
    "agent_spec": "{\"name\":\"my-agent\",\"type\":\"llm\",\"model\":\"gpt-4\"}"
  },
  "id": 2
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `agent_spec` | string | 是 | Agent 规格描述（JSON 字符串），包含 name、type、model 等配置 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent_id": "agentos-0000000000000001-00000005"
  },
  "id": 2
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "agent.register",
    "params": {
      "agent_spec": "{\"name\":\"my-agent\",\"type\":\"llm\",\"model\":\"gpt-4\"}"
    },
    "id": 2
  }'
```

---

#### 3.2.3 agent.info — 查询 Agent 信息

查询指定 Agent 的详细信息或调用 Agent 执行操作。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "agent.info",
  "params": {
    "agent_id": "agentos-0000000000000001-00000005",
    "input": "可选的输入数据"
  },
  "id": 3
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `agent_id` | string | 是 | Agent 唯一标识符 |
| `input` | string | 否 | 传递给 Agent 的输入数据 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent_id": "agentos-0000000000000001-00000005",
    "output": "invocation processed",
    "processing_time_ms": 5.2
  },
  "id": 3
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "agent.info",
    "params": {
      "agent_id": "agentos-0000000000000001-00000005",
      "input": "Hello, agent!"
    },
    "id": 3
  }'
```

---

#### 3.2.4 agent.start — 启动 Agent

启动一个已注册的 Agent 实例。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "agent.start",
  "params": {
    "agent_id": "agentos-0000000000000001-00000005"
  },
  "id": 4
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent_id": "agentos-0000000000000001-00000005",
    "status": "started"
  },
  "id": 4
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "agent.start",
    "params": {"agent_id": "agentos-0000000000000001-00000005"},
    "id": 4
  }'
```

---

#### 3.2.5 agent.stop — 停止 Agent

停止一个正在运行的 Agent 实例。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "agent.stop",
  "params": {
    "agent_id": "agentos-0000000000000001-00000005"
  },
  "id": 5
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "terminated": true
  },
  "id": 5
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "agent.stop",
    "params": {"agent_id": "agentos-0000000000000001-00000005"},
    "id": 5
  }'
```

---

### 3.3 Memory 管理（memory.*）

#### 3.3.1 memory.write — 写入记忆

将数据写入持久化记忆存储。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "memory.write",
  "params": {
    "data": "要存储的记忆数据",
    "metadata": "{\"tags\":[\"important\"],\"category\":\"notes\"}"
  },
  "id": 1
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `data` | string | 是 | 要存储的数据内容 |
| `metadata` | string | 否 | 元数据（JSON 字符串），用于索引和检索 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "record_id": "agentos-0000000000000001-0000000a"
  },
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "memory.write",
    "params": {
      "data": "今天的会议纪要：讨论了Airymax的架构设计",
      "metadata": "{\"tags\":[\"meeting\",\"architecture\"],\"category\":\"work\"}"
    },
    "id": 1
  }'
```

---

#### 3.3.2 memory.search — 搜索记忆

根据查询条件搜索记忆记录。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "memory.search",
  "params": {
    "query": "架构设计",
    "limit": 10
  },
  "id": 2
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `query` | string | 是 | 搜索关键词（在 metadata 中匹配） |
| `limit` | number | 否 | 返回结果数量上限，默认 10 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "results": [
      {
        "record_id": "agentos-0000000000000001-0000000a",
        "score": 1.0
      }
    ],
    "total": 1
  },
  "id": 2
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "memory.search",
    "params": {"query": "架构设计", "limit": 10},
    "id": 2
  }'
```

---

#### 3.3.3 memory.query — 查询记忆详情

根据 record_id 获取记忆记录的完整内容。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "memory.query",
  "params": {
    "record_id": "agentos-0000000000000001-0000000a"
  },
  "id": 3
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `record_id` | string | 是 | 记忆记录唯一标识符 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "data": "今天的会议纪要：讨论了Airymax的架构设计",
    "length": 60
  },
  "id": 3
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "memory.query",
    "params": {"record_id": "agentos-0000000000000001-0000000a"},
    "id": 3
  }'
```

---

### 3.4 Session 管理（session.*）

#### 3.4.1 session.create — 创建会话

创建一个新的会话上下文。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "session.create",
  "params": {
    "metadata": "{\"user\":\"admin\",\"purpose\":\"debugging\"}"
  },
  "id": 1
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `metadata` | string | 否 | 会话元数据（JSON 字符串） |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "session_id": "agentos-0000000000000001-0000000b"
  },
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "session.create",
    "params": {"metadata": "{\"user\":\"admin\"}"},
    "id": 1
  }'
```

---

#### 3.4.2 session.get — 获取会话信息

获取指定会话的详细信息。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "session.get",
  "params": {
    "session_id": "agentos-0000000000000001-0000000b"
  },
  "id": 2
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "session_id": "agentos-0000000000000001-0000000b",
    "metadata": "{\"user\":\"admin\"}",
    "age_seconds": 120.5
  },
  "id": 2
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "session.get",
    "params": {"session_id": "agentos-0000000000000001-0000000b"},
    "id": 2
  }'
```

---

#### 3.4.3 session.delete — 删除会话

关闭并删除一个会话。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "session.delete",
  "params": {
    "session_id": "agentos-0000000000000001-0000000b"
  },
  "id": 3
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "closed": true
  },
  "id": 3
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "session.delete",
    "params": {"session_id": "agentos-0000000000000001-0000000b"},
    "id": 3
  }'
```

---

### 3.5 Skill 管理（skill.*）

#### 3.5.1 skill.load — 加载技能

加载一个技能模块到系统中。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "skill.load",
  "params": {
    "skill_name": "code_review",
    "skill_path": "/opt/agentos/skills/code_review.so",
    "config": "{\"language\":\"python\",\"severity\":\"strict\"}"
  },
  "id": 1
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `skill_name` | string | 是 | 技能名称标识符 |
| `skill_path` | string | 是 | 技能模块文件路径 |
| `config` | string | 否 | 技能配置（JSON 字符串） |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "skill_name": "code_review",
    "status": "loaded",
    "version": "1.0.0"
  },
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "skill.load",
    "params": {
      "skill_name": "code_review",
      "skill_path": "/opt/agentos/skills/code_review.so"
    },
    "id": 1
  }'
```

---

#### 3.5.2 skill.unload — 卸载技能

从系统中卸载一个已加载的技能模块。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "skill.unload",
  "params": {
    "skill_name": "code_review"
  },
  "id": 2
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "skill_name": "code_review",
    "status": "unloaded"
  },
  "id": 2
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "skill.unload",
    "params": {"skill_name": "code_review"},
    "id": 2
  }'
```

---

#### 3.5.3 skill.list — 列出所有技能

列出系统中所有已加载的技能模块。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "skill.list",
  "params": {},
  "id": 3
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "skills": [
      {
        "name": "code_review",
        "version": "1.0.0",
        "status": "loaded"
      },
      {
        "name": "prompt_optimizer",
        "version": "2.1.0",
        "status": "loaded"
      }
    ],
    "total": 2
  },
  "id": 3
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"skill.list","params":{},"id":3}'
```

---

### 3.6 Service 管理（service.*）

#### 3.6.1 service.list — 列出所有服务

列出系统中所有已注册的微服务。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "service.list",
  "params": {},
  "id": 1
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "services": [
      {
        "name": "gateway_d",
        "version": "0.1.0",
        "state": "running",
        "capabilities": ["async", "streaming"]
      }
    ],
    "total": 1
  },
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"service.list","params":{},"id":1}'
```

---

#### 3.6.2 service.status — 查询服务状态

查询指定服务的运行状态和统计信息。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "service.status",
  "params": {
    "service_name": "gateway_d"
  },
  "id": 2
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "service_name": "gateway_d",
    "state": "running",
    "uptime_seconds": 3600.0,
    "request_count": 15000,
    "error_count": 3,
    "current_concurrent": 12
  },
  "id": 2
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "service.status",
    "params": {"service_name": "gateway_d"},
    "id": 2
  }'
```

---

### 3.7 Config 管理（config.*）

#### 3.7.1 config.get — 获取配置项

获取指定的配置项值。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "config.get",
  "params": {
    "key": "gateway.http.port"
  },
  "id": 1
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `key` | string | 是 | 配置项键名（点号分隔层级） |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "key": "gateway.http.port",
    "value": 8080
  },
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "config.get",
    "params": {"key": "gateway.http.port"},
    "id": 1
  }'
```

---

#### 3.7.2 config.set — 设置配置项

动态设置配置项值（运行时生效，不持久化）。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "config.set",
  "params": {
    "key": "gateway.http.timeout_ms",
    "value": 60000
  },
  "id": 2
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `key` | string | 是 | 配置项键名 |
| `value` | any | 是 | 配置项新值 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "key": "gateway.http.timeout_ms",
    "value": 60000,
    "updated": true
  },
  "id": 2
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "config.set",
    "params": {"key": "gateway.http.timeout_ms", "value": 60000},
    "id": 2
  }'
```

---

#### 3.7.3 config.list — 列出所有配置

列出当前所有可用的配置项。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "config.list",
  "params": {
    "prefix": "gateway.http"
  },
  "id": 3
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `prefix` | string | 否 | 配置项前缀过滤，不传则返回全部 |

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "config": {
      "gateway.http.host": "0.0.0.0",
      "gateway.http.port": 8080,
      "gateway.http.enabled": true,
      "gateway.http.max_request_size": 1048576,
      "gateway.http.timeout_ms": 30000
    }
  },
  "id": 3
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "config.list",
    "params": {"prefix": "gateway.http"},
    "id": 3
  }'
```

---

### 3.8 系统方法（ping）

#### ping — 健康检查

检测 Gateway 是否存活。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "ping",
  "params": {},
  "id": 1
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {},
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"ping","id":1}'
```

---

## 4. REST API 端点

除了 JSON-RPC 2.0 接口，Gateway 还提供以下 RESTful 端点：

### 4.1 GET /health — 健康检查

返回 Gateway 的整体健康状态。

**响应格式：**

```json
{
  "status": "ok",
  "version": "0.1.0",
  "uptime_seconds": 3600.5,
  "services": {
    "gateway": "healthy",
    "mcp_server": "healthy",
    "a2a_handler": "healthy",
    "openai_compat": "healthy"
  }
}
```

**curl 示例：**

```bash
curl http://localhost:8080/health
```

---

### 4.2 GET /api/v1/protocols/adapters — 列出协议适配器

返回当前已注册的所有协议适配器信息。

**响应格式：**

```json
{
  "adapters": [
    {
      "name": "JSON-RPC",
      "protocol": "jsonrpc",
      "version": "2.0",
      "status": "active",
      "endpoint": "/"
    },
    {
      "name": "MCP",
      "protocol": "mcp",
      "version": "2024-11-05",
      "status": "active",
      "endpoint": "/mcp"
    },
    {
      "name": "A2A",
      "protocol": "a2a",
      "version": "0.3.0",
      "status": "active",
      "endpoint": "/a2a"
    },
    {
      "name": "OpenAI Compatible",
      "protocol": "openai",
      "version": "1.0",
      "status": "active",
      "endpoint": "/v1/chat/completions"
    },
    {
      "name": "OpenJiuwen",
      "protocol": "openjiuwen",
      "version": "1.0",
      "status": "active",
      "endpoint": "/ojiuwen"
    }
  ]
}
```

**curl 示例：**

```bash
curl http://localhost:8080/health
```

---

### 4.3 GET /api/v1/protocols/stats — 协议统计信息

返回各协议的请求统计信息。

**响应格式：**

```json
{
  "total_requests": 50000,
  "mcp_requests": 12000,
  "a2a_requests": 8000,
  "openai_requests": 25000,
  "jsonrpc_requests": 5000,
  "unknown_requests": 0,
  "route_errors": 12
}
```

**curl 示例：**

```bash
curl http://localhost:8080/metrics
```

---

## 5. 协议特定端点

Gateway 通过智能协议检测自动将请求路由到对应的协议处理器。检测顺序为：**URL 路径 → Content-Type → 请求体内容**。

### 5.1 MCP（Model Context Protocol）

**端点：** `POST /mcp`

MCP（Model Context Protocol）是用于 AI 模型与工具/资源交互的标准协议。Airymax Gateway 实现了 MCP 服务端，支持以下 JSON-RPC 方法：

#### 5.1.1 initialize — 初始化连接

初始化 MCP 客户端与服务端的连接，协商协议版本和能力。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
      "name": "my-mcp-client",
      "version": "1.0.0"
    }
  },
  "id": 1
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {"listChanged": true},
      "resources": {"subscribe": true, "listChanged": true}
    },
    "serverInfo": {
      "name": "agentos-gateway",
      "version": "1.0.0"
    }
  },
  "id": 1
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {"name": "my-client", "version": "1.0.0"}
    },
    "id": 1
  }'
```

---

#### 5.1.2 tools/list — 列出工具

获取所有已注册的 MCP 工具列表。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "params": {},
  "id": 2
}
```

**响应格式：**

```json
{
  "tools": [
    {
      "name": "web_search",
      "description": "Search the web for information"
    },
    {
      "name": "code_executor",
      "description": "Execute code in a sandboxed environment"
    }
  ]
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","params":{},"id":2}'
```

---

#### 5.1.3 tools/call — 调用工具

调用指定的 MCP 工具执行操作。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "web_search",
    "arguments": {
      "query": "Airymax architecture",
      "max_results": 5
    }
  },
  "id": 3
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "搜索结果：Airymax 是一个..."
      }
    ]
  },
  "id": 3
}
```

**错误处理：**

| 错误码 | 说明 |
|:-------|:-----|
| `-32601` | Tool not found：工具未注册 |
| `-32603` | Tool execution failed：工具执行失败 |

**curl 示例：**

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "web_search",
      "arguments": {"query": "Airymax", "max_results": 5}
    },
    "id": 3
  }'
```

---

#### 5.1.4 resources/list — 列出资源

获取所有已注册的 MCP 资源列表。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "resources/list",
  "params": {},
  "id": 4
}
```

**响应格式：**

```json
{
  "resources": [
    {
      "uri": "file:///docs/architecture.md",
      "name": "Architecture Document",
      "description": "Airymax 架构设计文档",
      "mimeType": "text/markdown"
    }
  ]
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"resources/list","params":{},"id":4}'
```

---

#### 5.1.5 resources/read — 读取资源

读取指定 URI 的资源内容。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "resources/read",
  "params": {
    "uri": "file:///docs/architecture.md"
  },
  "id": 5
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "contents": [
      {
        "uri": "file:///docs/architecture.md",
        "mimeType": "text/markdown",
        "text": "# Airymax Architecture\n\n..."
      }
    ]
  },
  "id": 5
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "resources/read",
    "params": {"uri": "file:///docs/architecture.md"},
    "id": 5
  }'
```

---

#### 5.1.6 prompts/list — 列出提示模板

获取所有已注册的提示模板列表。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "prompts/list",
  "params": {},
  "id": 6
}
```

**响应格式：**

```json
{
  "prompts": [
    {
      "name": "code_review",
      "description": "代码审查提示模板",
      "arguments": [
        {
          "name": "language",
          "description": "编程语言",
          "required": true
        }
      ]
    }
  ]
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"prompts/list","params":{},"id":6}'
```

---

#### 5.1.7 prompts/get — 获取提示模板

获取指定提示模板的具体内容。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "language": "python"
    }
  },
  "id": 7
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "description": "代码审查提示模板",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "请审查以下 Python 代码..."
        }
      }
    ]
  },
  "id": 7
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "prompts/get",
    "params": {
      "name": "code_review",
      "arguments": {"language": "python"}
    },
    "id": 7
  }'
```

---

### 5.2 A2A（Agent-to-Agent）

**端点：** `POST /a2a`

A2A（Agent-to-Agent）协议用于 Agent 之间的通信与协作。Airymax Gateway 实现了 A2A 服务端，支持以下方法：

#### 5.2.1 agent.discover — 发现 Agent

获取 Agent 的能力卡片（Agent Card），包含 Agent 的基本信息和能力声明。

**端点：** `GET /a2a/agent-card`

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "agent.discover",
  "params": {
    "agent_url": "http://localhost:8080/a2a"
  },
  "id": 1
}
```

**响应格式：**

```json
{
  "name": "agentos-a2a",
  "version": "0.3.0",
  "url": "http://localhost:8080/a2a",
  "capabilities": {
    "taskExecution": true,
    "streaming": true,
    "pushNotifications": true,
    "negotiation": true,
    "multiTurn": true,
    "stateTransition": true
  },
  "protocolVersion": "0.3.0"
}
```

**curl 示例：**

```bash
# 直接获取 Agent Card
curl http://localhost:8080/a2a/agent-card

# 通过 JSON-RPC 方式
curl -X POST http://localhost:8080/a2a \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "agent.discover",
    "params": {"agent_url": "http://localhost:8080/a2a"},
    "id": 1
  }'
```

---

#### 5.2.2 agent.register — 注册 Agent

在 A2A 网络中注册一个 Agent。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "agent.register",
  "params": {
    "agent_name": "my-agent",
    "agent_version": "1.0.0",
    "agent_url": "http://my-agent:8080/a2a",
    "capabilities": 63
  },
  "id": 2
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent_id": "agentos-0000000000000001-00000010",
    "status": "registered"
  },
  "id": 2
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/a2a \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "agent.register",
    "params": {
      "agent_name": "my-agent",
      "agent_version": "1.0.0",
      "agent_url": "http://my-agent:8080/a2a",
      "capabilities": 63
    },
    "id": 2
  }'
```

---

#### 5.2.3 task.delegate — 委托任务

将一个任务委托给 A2A Agent 执行。

**端点：** `POST /a2a/task`

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "task.delegate",
  "params": {
    "type": "code_review",
    "id": "task-001",
    "input": "{\"file\":\"main.c\",\"language\":\"c\"}"
  },
  "id": 3
}
```

**响应格式：**

```json
{
  "result": {
    "status": "completed",
    "output": "{\"issues\":[],\"score\":95}"
  }
}
```

**错误处理：**

| 错误码 | 说明 |
|:-------|:-----|
| `-32601` | Unknown task type：任务类型未注册 |
| `-32700` | Parse error：请求体格式错误 |

**curl 示例：**

```bash
curl -X POST http://localhost:8080/a2a/task \
  -H "Content-Type: application/json" \
  -d '{
    "type": "code_review",
    "id": "task-001",
    "input": "{\"file\":\"main.c\",\"language\":\"c\"}"
  }'
```

---

#### 5.2.4 negotiate.propose — 协商提议

在 Agent 之间发起协商提议。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "negotiate.propose",
  "params": {
    "proposal": "{\"action\":\"share_resource\",\"resource\":\"gpu-0\"}",
    "target_agent": "agentos-0000000000000001-00000010"
  },
  "id": 4
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "proposal_id": "prop-001",
    "status": "proposed"
  },
  "id": 4
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/a2a \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "negotiate.propose",
    "params": {
      "proposal": "{\"action\":\"share_resource\",\"resource\":\"gpu-0\"}",
      "target_agent": "agentos-0000000000000001-00000010"
    },
    "id": 4
  }'
```

---

#### 5.2.5 consensus.vote — 共识投票

对协商提议进行投票。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "consensus.vote",
  "params": {
    "proposal_id": "prop-001",
    "vote": "approve",
    "reason": "资源分配合理"
  },
  "id": 5
}
```

**响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "proposal_id": "prop-001",
    "vote": "approve",
    "consensus_reached": true
  },
  "id": 5
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:8080/a2a \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "consensus.vote",
    "params": {
      "proposal_id": "prop-001",
      "vote": "approve",
      "reason": "资源分配合理"
    },
    "id": 5
  }'
```

---

### 5.3 OpenAI 兼容 API

**端点：** `POST /v1/chat/completions`、`POST /v1/embeddings`、`GET /v1/models`

Airymax Gateway 提供与 OpenAI API 兼容的接口，允许使用 OpenAI SDK 直接调用 Airymax 服务。

#### 5.3.1 POST /v1/chat/completions — Chat Completions

兼容 OpenAI Chat Completions API。

**请求格式：**

```json
{
  "model": "gpt-4",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Hello! What is Airymax?"}
  ],
  "temperature": 0.7,
  "max_tokens": 4096
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `model` | string | 是 | 模型名称（默认 `gpt-4`） |
| `messages` | array | 是 | 对话消息数组 |
| `temperature` | number | 否 | 采样温度，默认 0.7 |
| `max_tokens` | number | 否 | 最大生成 token 数，默认 4096 |
| `functions` / `tools` | array | 否 | 函数调用定义 |

**响应格式：**

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "created": 1718700000,
  "model": "gpt-4",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Airymax 是 SPHARX 开发的多智能体运行时框架..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 50,
    "total_tokens": 70
  }
}
```

**错误处理：**

| HTTP 状态码 | 说明 |
|:-----------|:-----|
| `400` | Invalid request：请求体为空或格式错误 |
| `429` | Rate limit exceeded：超出速率限制 |
| `500` | LLM call failed：LLM 后端调用失败 |
| `503` | No LLM backend configured：未配置 LLM 后端 |

**curl 示例：**

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-api-key" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {"role": "user", "content": "Hello! What is Airymax?"}
    ],
    "temperature": 0.7,
    "max_tokens": 4096
  }'
```

**Python SDK 用法：**

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8080/v1",
    api_key="sk-your-api-key"
)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response.choices[0].message.content)
```

---

#### 5.3.2 POST /v1/embeddings — Embeddings

兼容 OpenAI Embeddings API。

**请求格式：**

```json
{
  "model": "text-embedding-ada-002",
  "input": "The quick brown fox jumps over the lazy dog"
}
```

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:-----|:-----|
| `model` | string | 是 | 嵌入模型名称 |
| `input` | string / array | 是 | 要嵌入的文本（单个字符串或字符串数组） |

**响应格式：**

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [0.01, 0.02, 0.03, "..."]
    }
  ],
  "model": "text-embedding-ada-002",
  "usage": {
    "prompt_tokens": 8,
    "total_tokens": 8
  }
}
```

**错误处理：**

| HTTP 状态码 | 说明 |
|:-----------|:-----|
| `400` | Invalid request：请求体为空 |
| `500` | Embedding failed：嵌入计算失败 |
| `503` | No embedding backend configured：未配置嵌入后端 |

**curl 示例：**

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-api-key" \
  -d '{
    "model": "text-embedding-ada-002",
    "input": "The quick brown fox jumps over the lazy dog"
  }'
```

---

#### 5.3.3 GET /v1/models — 列出模型

获取可用的模型列表。

**响应格式：**

```json
{
  "object": "list",
  "data": [
    {
      "id": "gpt-4",
      "object": "model",
      "owned_by": "agentos"
    }
  ]
}
```

**curl 示例：**

```bash
curl http://localhost:8080/v1/models \
  -H "Authorization: Bearer sk-your-api-key"
```

---

### 5.4 OpenJiuwen（二进制协议）

**端点：** `POST /ojiuwen`

OpenJiuwen 是 Airymax 支持的自定义二进制协议，用于高性能场景下的 Agent 间通信。请求和响应均使用 `application/octet-stream` 格式。

**特性：**

- 二进制编码，低延迟高吞吐
- 支持消息压缩和加密
- 适用于 Agent 间大量数据传输

**请求格式：** 二进制数据流，Content-Type 为 `application/octet-stream`

**响应格式：** 二进制数据流

**健康检查：**

```bash
curl http://localhost:8080/ojiuwen/health
```

**curl 示例（发送二进制数据）：**

```bash
# 发送二进制文件
curl -X POST http://localhost:8080/ojiuwen \
  -H "Content-Type: application/octet-stream" \
  --data-binary @request.bin \
  -o response.bin
```

---

## 6. 错误码参考

### JSON-RPC 2.0 标准错误码

| 错误码 | 名称 | 说明 |
|:-------|:-----|:-----|
| `-32700` | Parse error | JSON 解析错误 |
| `-32600` | Invalid Request | 请求格式无效（缺少 jsonrpc/method 字段） |
| `-32601` | Method not found | 方法不存在 |
| `-32602` | Invalid params | 参数无效或缺失 |
| `-32603` | Internal error | 内部错误 |
| `-32000` | Server error | 服务端通用错误（含系统调用失败） |

### Gateway 专用错误码

| 错误码 | 枚举值 | 说明 |
|:-------|:-------|:-----|
| `0` | `GATEWAY_SUCCESS` | 成功 |
| `-1` | `GATEWAY_ERROR_INVALID` | 无效参数 |
| `-2` | `GATEWAY_ERROR_MEMORY` | 内存不足 |
| `-3` | `GATEWAY_ERROR_IO` | I/O 错误 |
| `-4` | `GATEWAY_ERROR_TIMEOUT` | 超时 |
| `-5` | `GATEWAY_ERROR_CLOSED` | 连接已关闭 |
| `-6` | `GATEWAY_ERROR_PROTOCOL` | 协议错误 |

### HTTP 状态码

| 状态码 | 说明 | 场景 |
|:-------|:-----|:-----|
| `200` | OK | 请求成功 |
| `400` | Bad Request | 请求格式错误 |
| `401` | Unauthorized | API Key 缺失或无效 |
| `404` | Not Found | 端点或资源不存在 |
| `429` | Too Many Requests | 超出速率限制 |
| `500` | Internal Server Error | 服务器内部错误 |
| `503` | Service Unavailable | 服务不可用（后端未配置） |

### 错误响应示例

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params: task_id required",
    "data": null
  },
  "id": 1
}
```

---

## 7. 速率限制与分页

### 7.1 速率限制（Rate Limiting）

Gateway 使用令牌桶算法实现速率限制，防止 DoS 攻击和资源滥用。

**默认配置：**

| 参数 | 默认值 | 说明 |
|:-----|:-------|:-----|
| `enabled` | `false` | 是否启用速率限制 |
| `requests_per_second` | 100 | 每秒请求数限制 |
| `requests_per_minute` | 6000 | 每分钟请求数限制 |
| `requests_per_hour` | 360000 | 每小时请求数限制 |
| `burst_size` | 150 | 突发容量（令牌桶大小） |
| `cleanup_interval_sec` | 300 | 过期客户端清理间隔（秒） |

**OpenAI 兼容接口速率限制：**

| 参数 | 默认值 | 说明 |
|:-----|:-------|:-----|
| `rate_limit_rpm` | 60 | 每分钟请求数限制 |

**超出限制时的响应：**

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Rate limit exceeded"
  },
  "id": 1
}
```

HTTP 状态码：`429 Too Many Requests`

**响应头：**

| Header | 说明 |
|:-------|:-----|
| `X-RateLimit-Limit` | 限制值 |
| `X-RateLimit-Remaining` | 剩余可用请求数 |
| `X-RateLimit-Reset` | 重置时间（Unix 时间戳） |
| `Retry-After` | 建议重试等待时间（秒） |

### 7.2 分页（Pagination）

支持分页的方法（如 `task.list`、`memory.search`）使用 `limit` 和 `offset` 参数进行分页：

| 参数 | 类型 | 默认值 | 说明 |
|:-----|:-----|:-------|:-----|
| `limit` | number | 20 | 每页返回的最大记录数 |
| `offset` | number | 0 | 分页偏移量（跳过的记录数） |

**分页响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "items": [...],
    "total": 100,
    "limit": 20,
    "offset": 0
  },
  "id": 1
}
```

---

## 8. WebSocket 支持

Airymax Gateway 支持通过 WebSocket 进行 JSON-RPC 2.0 双向通信，适用于需要实时推送、流式响应或长连接的场景。

### 8.1 连接

**WebSocket 端点：** `ws://localhost:8081`（默认端口）

```javascript
// JavaScript 示例
const ws = new WebSocket('ws://localhost:8081');

ws.onopen = () => {
  console.log('WebSocket 连接已建立');
};
```

### 8.2 JSON-RPC over WebSocket

WebSocket 通信完全遵循 JSON-RPC 2.0 协议，每条消息为一个完整的 JSON-RPC 请求或响应。

**发送请求：**

```javascript
// 发送 JSON-RPC 请求
ws.send(JSON.stringify({
  jsonrpc: "2.0",
  method: "ping",
  id: 1
}));

// 提交任务
ws.send(JSON.stringify({
  jsonrpc: "2.0",
  method: "task.submit",
  params: {
    input: "{\"action\":\"analyze\"}",
    timeout_ms: 30000
  },
  id: 2
}));
```

**接收响应：**

```javascript
ws.onmessage = (event) => {
  const response = JSON.parse(event.data);
  
  if (response.id === 1) {
    console.log('Ping 响应:', response.result);
  } else if (response.id === 2) {
    console.log('任务已提交:', response.result.task_id);
  }
  
  // 处理错误响应
  if (response.error) {
    console.error('错误:', response.error.code, response.error.message);
  }
};
```

### 8.3 服务端推送（Notifications）

WebSocket 支持 JSON-RPC 2.0 通知（无需 `id` 字段的请求），用于服务端主动推送事件：

```json
{
  "jsonrpc": "2.0",
  "method": "task.status_changed",
  "params": {
    "task_id": "agentos-0000000000000001-00000001",
    "status": 2
  }
}
```

### 8.4 认证

WebSocket 连接支持通过查询参数传递 API Key：

```javascript
const ws = new WebSocket('ws://localhost:8081?api_key=sk-your-api-key');
```

或在连接建立后通过首个消息发送认证信息：

```javascript
ws.onopen = () => {
  ws.send(JSON.stringify({
    jsonrpc: "2.0",
    method: "auth",
    params: { api_key: "sk-your-api-key" },
    id: 0
  }));
};
```

### 8.5 心跳保活

WebSocket 连接支持通过 `ping` 方法进行心跳检测：

```javascript
// 每 30 秒发送一次心跳
setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ jsonrpc: "2.0", method: "ping", id: Date.now() }));
  }
}, 30000);
```

### 8.6 错误处理与重连

```javascript
let reconnectAttempts = 0;
const maxReconnectAttempts = 5;

function connectWebSocket() {
  const ws = new WebSocket('ws://localhost:8081');

  ws.onclose = (event) => {
    if (reconnectAttempts < maxReconnectAttempts) {
      reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, reconnectAttempts), 30000);
      console.log(`WebSocket 断开，${delay}ms 后重连 (${reconnectAttempts}/${maxReconnectAttempts})`);
      setTimeout(connectWebSocket, delay);
    }
  };

  ws.onerror = (error) => {
    console.error('WebSocket 错误:', error);
  };

  return ws;
}
```

---

## 附录：快速参考

### 常用 curl 命令速查

```bash
# 健康检查
curl http://localhost:8080/health

# 心跳检测
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"ping","id":1}'

# 提交任务
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"task.submit","params":{"input":"{\"action\":\"test\"}"},"id":1}'

# 查询任务状态
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"task.status","params":{"task_id":"YOUR_TASK_ID"},"id":2}'

# OpenAI Chat Completions
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-key" \
  -d '{"model":"gpt-4","messages":[{"role":"user","content":"Hello!"}]}'

# 列出协议适配器
curl http://localhost:8080/health

# 协议统计
curl http://localhost:8080/metrics
```

### 环境变量速查

| 环境变量 | 默认值 | 说明 |
|:---------|:-------|:-----|
| `AGENTOS_ENDPOINT` | `http://127.0.0.1:8080` | Gateway 端点地址 |
| `AGENTOS_API_KEY` | （空） | API 认证密钥 |
| `AGENTOS_MAX_TASKS` | `256` | 最大任务数 |
| `AGENTOS_MAX_RECORDS` | `1024` | 最大记忆记录数 |
| `AGENTOS_MAX_SESSIONS` | `64` | 最大会话数 |
| `AGENTOS_MAX_AGENTS` | `128` | 最大 Agent 数 |
| `AGENTOS_RATE_LIMIT_TABLE_SIZE` | `1021` | 速率限制哈希表大小 |

---

> **文档版本：** P4.3.6 | **最后更新：** 2026-06-18 | **维护团队：** Team-B (Gateway)