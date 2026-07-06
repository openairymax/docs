# {API 名称} API 参考

> Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

**版本**: {版本号}  
**状态**: {草案/已发布}  
**最后更新**: {YYYY-MM-DD}

---

## 1. API 概述

{API 的简要概述，包括用途和适用场景}

### 1.1 基础信息

| 项目 | 值 |
|------|-----|
| 协议 | {HTTP/WebSocket/JSON-RPC 2.0} |
| 基础路径 | {如 /api/v1/task} |
| 认证方式 | {Bearer Token / API Key / 无} |
| 内容类型 | application/json |

### 1.2 通用响应格式

```json
{
  "code": 0,
  "message": "success",
  "data": { },
  "request_id": "uuid-v4"
}
```

### 1.3 错误码

| 错误码 | 含义 | HTTP 状态码 |
|--------|------|:-----------:|
| 0 | 成功 | 200 |
| 1001 | 参数无效 | 400 |
| 1002 | 未授权 | 401 |
| 2001 | 资源不存在 | 404 |
| 5000 | 内部错误 | 500 |

---

## 2. 接口列表

### 2.1 {接口名称 1}

{接口描述}

**请求**

| 方法 | 路径 |
|------|------|
| {GET/POST/PUT/DELETE} | {/api/v1/resource} |

**请求参数**

| 参数名 | 类型 | 必填 | 描述 |
|--------|------|:----:|------|
| param1 | string | 是 | 参数 1 描述 |
| param2 | integer | 否 | 参数 2 描述，默认 0 |

**请求示例**

```bash
curl -X POST https://api.airymax.dev/api/v1/resource \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"param1": "value1", "param2": 100}'
```

**响应参数**

| 字段名 | 类型 | 描述 |
|--------|------|------|
| field1 | string | 字段 1 描述 |
| field2 | integer | 字段 2 描述 |

**响应示例**

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "field1": "value1",
    "field2": 100
  }
}
```

### 2.2 {接口名称 2}

{按上述格式继续}

---

## 3. 数据模型

### 3.1 {模型名称 1}

| 字段名 | 类型 | 必填 | 描述 |
|--------|------|:----:|------|
| id | string | 是 | 唯一标识符 |
| name | string | 是 | 名称 |
| created_at | integer | 是 | 创建时间戳（毫秒） |

---

## 4. 限制与配额

| 限制项 | 值 |
|--------|-----|
| 每秒请求数 | 100 |
| 单次返回最大条数 | 1000 |
| 请求超时 | 30s |

---

## 5. 变更记录

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0.0 | {YYYY-MM-DD} | 初始版本 |

---

**© 2025-2026 SPHARX Ltd. All Rights Reserved.**
