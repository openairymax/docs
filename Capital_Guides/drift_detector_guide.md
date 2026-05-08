# AgentOS 配置漂移检测器使用指南

**版本**: V1.0  
**创建日期**: 2026-04-05  
**工具路径**: `agentos/manager/tools/drift_detector.py`  

---

## 目录

1. [概述](#1-概述)
2. [快速开始](#2-快速开始)
3. [功能特性](#3-功能特性)
4. [使用示例](#4-使用示例)
5. [输出格式](#5-输出格式)
6. [集成到CI/CD](#6-集成到cd)
7. [最佳实践](#7-最佳实践)
8. [故障排查](#8-故障排查)

---

## 1. 概述

### 1.1 什么是配置漂移检测？

**配置漂移** (Configuration Drift) 是指运行环境中的配置文件逐渐偏离预期基线状态的现象。这通常由以下原因引起：

- 手动修改配置文件
- 自动化脚本意外更改
- 部署流程不一致
- 紧急修复后未同步到基线

### 1.2 为什么需要检测配置漂移？

- ✅ **安全性**: 防止未授权的安全配置变更
- ✅ **稳定性**: 避免因配置不一致导致的服务中断
- ✅ **合规性**: 满足审计和合规要求
- ✅ **可追溯性**: 追踪配置变更历史

### 1.3 漂移检测器功能

| 功能 | 说明 |
|------|------|
| **基线创建** | 创建配置文件的SHA-256哈希清单 |
| **漂移检测** | 检测文件的修改、删除、新增 |
| **严重程度分级** | 根据文件重要性自动分级（CRITICAL/WARNING/INFO） |
| **报告生成** | 生成JSON和Markdown格式的详细报告 |
| **CI/CD集成** | 支持在CI流水线中自动检测 |

---

## 2. 快速开始

### 2.1 基本用法

```bash
cd agentos/manager/tools

# 创建基线（第一次使用）
python drift_detector.py --action create-baseline

# 检测漂移
python drift_detector.py --action detect

# 一键执行（创建基线 + 检测）
python drift_detector.py --action both
```

### 2.2 输出示例

```
======================================================================
Drift Report Summary:
  Scan Time: 2026-04-05T10:30:00.123456Z
  Config Directory: /path/to/manager
  Baseline Created: 2026-04-05T09:00:00.000000Z
  Total Files Scanned: 15
  Drifted Files: 2 (13.3%)
  Unchanged Files: 13
  Severity Breakdown:
    - Critical: 1
    - Warning: 0
    - Info: 1
======================================================================
```

---

## 3. 功能特性

### 3.1 文件分类

#### 🔴 CRITICAL（严重）
- `security/policy.yaml` - 安全策略
- `security/permission_rules.yaml` - 权限规则
- `model/model.yaml` - 模型配置
- `kernel/settings.yaml` - 内核配置
- `manager_management.yaml` - Manager管理配置

#### 🟡 WARNING（警告）
- `agent/registry.yaml` - Agent注册表
- `skill/registry.yaml` - 技能注册表
- `logging/manager.yaml` - 日志配置
- `sanitizer/sanitizer_rules.json` - 输入净化规则

#### 🔵 INFO（信息）
- 其他配置文件

### 3.2 检测类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **MODIFIED** | 文件内容被修改 | 配置参数变更 |
| **DELETED** | 文件被删除 | 配置文件丢失 |
| **ADDED** | 新增文件 | 未授权的文件 |

### 3.3 忽略规则

自动忽略以下文件：
- `*.pyc`, `__pycache__/` - Python编译文件
- `.git/`, `.gitignore` - Git相关文件
- `*.log` - 日志文件
- `node_modules/` - Node模块
- `.env*` - 环境变量文件
- `*.tmp`, `*.bak`, `*.swp` - 临时文件

---

## 4. 使用示例

### 4.1 创建基线

```bash
# 基本用法
python drift_detector.py --action create-baseline

# 指定配置目录
python drift_detector.py --config-dir /path/to/config --action create-baseline

# 详细模式
python drift_detector.py --action create-baseline --verbose
```

### 4.2 检测漂移

```bash
# 基本检测
python drift_detector.py --action detect

# 详细模式
python drift_detector.py --action detect --verbose

# 输出到指定文件
python drift_detector.py --action detect --output drift_report.json

# 同时输出Markdown格式
python drift_detector.py --action detect --output report.json --output-md report.md
```

### 4.3 CI/CD集成

```bash
# 在CI中运行，检测到漂移时返回错误码
python drift_detector.py --action detect --fail-on-drift
```

---

## 5. 集成到CI/CD

### 5.1 GitHub Actions

```yaml
- name: Check configuration drift
  run: |
    cd agentos/manager/tools
    python drift_detector.py --action detect --fail-on-drift --output drift_report.json
    
- name: Upload drift report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: drift-report
    path: agentos/manager/tools/drift_report.json
```

### 5.2 Jenkins Pipeline

```groovy
stage('Configuration Drift Check') {
    steps {
        sh '''
            cd agentos/manager/tools
            python drift_detector.py --action detect --fail-on-drift
        '''
    }
}
```

---

## 6. 最佳实践

### 6.1 基线管理

1. **初始基线**: 在部署后立即创建基线
2. **定期更新**: 每次配置变更后更新基线
3. **版本控制**: 将基线文件纳入版本控制

### 6.2 检测频率

| 环境 | 频率 | 说明 |
|------|------|------|
| **开发环境** | 每天1次 | 避免干扰开发 |
| **测试环境** | 每次部署后 | 确保一致性 |
| **预发布环境** | 每4小时 | 中等频率 |
| **生产环境** | 每30分钟 | 高频率监控 |

### 6.3 响应流程

```
检测到漂移 → 生成报告 → 评估严重程度
    ↓
CRITICAL: 立即响应、回滚配置、通知负责人
WARNING:  24小时内响应、评估影响、通知团队  
INFO:     记录并跟踪、定期审查
```

---

## 7. 故障排查

### 7.1 常见问题

**Q1: 提示"Baseline not found"**
```bash
python drift_detector.py --action create-baseline
```

**Q2: 检测不到预期的漂移**
```bash
# 重新创建基线
python drift_detector.py --action create-baseline
# 检查文件是否被忽略
python drift_detector.py --action detect --verbose
```

---

## 附录

### A. 命令行参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--config-dir` | 配置目录路径 | `../manager` |
| `--action` | 执行动作 | `both` |
| `--output` | JSON输出路径 | `drift_report.json` |
| `--output-md` | Markdown输出路径 | 自动生成 |
| `--verbose` | 详细模式 | `false` |
| `--fail-on-drift` | 检测到漂移时返回错误码 | `false` |

### B. 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|---------|
| V1.0 | 2026-04-05 | 初始版本 |

---

**文档维护**: AgentOS Team  
**最后更新**: 2026-04-05  

© 2026 SPHARX Ltd. All Rights Reserved.