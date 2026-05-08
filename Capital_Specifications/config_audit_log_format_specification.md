# AgentOS 配置变更审计日志格式规范

**文档版本**: V1.0  
**创建日期**: 2026-04-05  
**适用范围**: AgentOS Manager模块  
**Schema版本**: config-audit-log.schema.json (Draft-07)  

---

## 目录

1. [概述](#1-概述)
2. [日志格式定义](#2-日志格式定义)
3. [字段详细说明](#3-字段详细说明)
4. [动作类型定义](#4-动作类型定义)
5. [使用示例](#5-使用示例)
6. [最佳实践](#6-最佳实践)
7. [工具与集成](#7-工具与集成)
8. [常见问题](#8-常见问题)

---

## 1. 概述

### 1.1 目的

配置变更审计日志用于记录AgentOS Manager模块中所有配置文件的变更操作，提供：

- **可追溯性**: 完整记录谁在什么时间做了什么变更
- **合规性**: 满足安全审计和合规要求
- **故障排查**: 快速定位配置问题根源
- **变更回滚**: 支持基于审计日志的配置回滚

### 1.2 设计原则

遵循 **E-2 可观测性原则** 和 **E-1 安全内生原则**：

- ✅ **完整性**: 记录所有关键信息，无遗漏
- ✅ **不可篡改**: 日志采用加密存储（AES-256-GCM）
- ✅ **结构化**: JSON格式，便于解析和查询
- ✅ **标准化**: 遵循JSON Schema Draft-07规范

---

## 2. 日志格式定义

### 2.1 整体结构

审计日志采用 **JSON数组** 格式，每个数组元素为一条独立的审计日志条目：

```json
[
  {
    "timestamp": "2026-04-05T10:30:00.123456Z",
    "action": "CHANGE",
    "config_file": "kernel/settings.yaml",
    "operator": { ... },
    "changes": [ ... ],
    "checksum": { ... },
    "metadata": { ... },
    "result": { ... }
  }
]
```

### 2.2 必需字段

每条审计日志 **必须** 包含以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `timestamp` | string | ISO 8601格式的时间戳 |
| `action` | string | 审计动作类型 |
| `config_file` | string | 配置文件相对路径 |
| `operator` | object | 操作者信息 |
| `checksum` | object | 文件校验和 |

---

## 3. 字段详细说明

### 3.1 timestamp（时间戳）

**格式**: ISO 8601 (RFC 3339)  
**示例**: `"2026-04-05T10:30:00.123456Z"`

- 必须使用UTC时间（以`Z`结尾）
- 精确到微秒（6位小数）
- 格式: `YYYY-MM-DDTHH:MM:SS.ffffffZ`

### 3.2 action（动作类型）

| 值 | 说明 | 触发场景 |
|---|------|---------|
| `LOAD` | 配置加载 | 系统启动、手动加载 |
| `RELOAD` | 热更新重载 | 文件监听器检测到变更 |
| `CHANGE` | 配置变更 | 用户/API修改配置 |
| `ROLLBACK` | 回滚 | 验证失败、手动回滚 |
| `VALIDATE` | Schema验证 | CI/CD、定期检查 |
| `EXPORT` | 导出 | 备份、迁移 |
| `IMPORT` | 导入 | 部署、恢复 |

### 3.3 config_file（配置文件路径）

**格式**: 相对路径（相对于`manager/`目录）

常见配置文件：
- `kernel/settings.yaml` - 内核配置
- `model/model.yaml` - 模型配置
- `security/policy.yaml` - 安全策略
- `agent/registry.yaml` - Agent注册表

### 3.4 operator（操作者信息）

```json
{
  "type": "user | system | ci_cd",
  "identity": "操作者标识",
  "ip_address": "IP地址（可选）",
  "session_id": "会话ID（可选）"
}
```

| 类型 | 说明 | identity示例 |
|------|------|-------------|
| `user` | 真实用户 | admin, devops_engineer |
| `system` | 系统组件 | config-manager, file-watcher |
| `ci_cd` | CI/CD流水线 | github-actions, jenkins |

### 3.5 changes（变更详情）

```json
{
  "changes": [
    {
      "path": "YAML路径表达式",
      "old_value": "旧值",
      "new_value": "新值",
      "field_type": "字段类型（可选）"
    }
  ]
}
```

### 3.6 checksum（校验和）

```json
{
  "checksum": {
    "algorithm": "sha256",
    "before": "变更前的SHA-256哈希",
    "after": "变更后的SHA-256哈希"
  }
}
```

### 3.7 metadata（扩展元数据）

| 字段 | 类型 | 说明 |
|------|------|------|
| `environment` | string | 运行环境 |
| `version` | string | Manager模块版本号 |
| `correlation_id` | string | 关联ID |
| `source` | string | 变更来源 |
| `reason` | string | 变更原因说明 |
| `approved_by` | string | 审批人 |
| `ticket_id` | string | 关联工单ID |

### 3.8 result（操作结果）

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | boolean | 操作是否成功 |
| `error_code` | string | 错误码 |
| `error_message` | string | 错误消息 |
| `duration_ms` | integer | 操作耗时（毫秒） |

---

## 4. 最佳实践

### 4.1 日志记录原则

1. **及时记录**: 变更发生后立即记录，不要延迟
2. **完整记录**: 包含所有必需字段
3. **准确记录**: 确保时间戳、哈希值信息准确
4. **安全存储**: 使用AES-256-GCM加密存储

### 4.2 日志保留策略

```yaml
audit:
  enabled: true
  log_path: "audit/config_audit.log"
  events:
    - config.load
    - config.reload
    - config.change
    - config.rollback
    - config.validate
    - config.export
    - config.import
  retention_days: 90
```

---

## 5. 工具与集成

| 工具 | 路径 | 用途 |
|------|------|------|
| 审计日志生成器 | `tools/audit_log_generator.py` | 生成测试日志 |
| Schema验证器 | `tests/test_schema_validation.py` | 验证配置文件 |

---

## 附录

### A. 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|---------|
| V1.0 | 2026-04-05 | 初始版本 |

---

**文档维护**: AgentOS Team  
**最后更新**: 2026-04-05  

© 2026 SPHARX Ltd. All Rights Reserved.