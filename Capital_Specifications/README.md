Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

---
copyright: "Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved."
slogan: "From data intelligence emerges."
title: "Airymax 技术规范"
version: "Doc V2.3"
last_updated: "2026-07-04"
status: "production_ready"
review_due: "2026-07-31"
theoretical_basis: "工程两论、五维正交系统、Thinkdual 双思考系统、ARE Standards 开放标准体系"
target_audience: "开发者/架构师/安全专家"
prerequisites: "了解软件开发流程，熟悉技术规范概念"
estimated_reading_time: "1.5小时"
core_concepts: "契约规范, 编码标准, 项目管理, 集成标准, 开放标准, 术语统一, 安全合规"
---

# Airymax 技术规范

**最新**: 2026-07-04
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/README.md

## 📋 文档信息卡
- **目标读者**: 开发者/架构师/安全专家
- **前置知识**: 了解软件开发流程，熟悉技术规范概念
- **预计阅读时间**: 1小时
- **核心概念**: 契约规范, 编码标准, 项目管理, 集成标准, 术语统一, 安全合规
- **文档状态**: 🟢 生产就绪
- **复杂度标识**: ⭐⭐ 中级

## 🎯 概述

Airymax 技术规范体系是项目开发、测试、部署和维护的权威标准。本规范体系基于工程控制论、系统工程和 Thinkdual 双思考系统，确保系统的稳定性、安全性和可演进性。

### 规范体系结构

```
┌─────────────────────────────────────────┐
│         Airymax 技术规范体系              │
├─────────────────────────────────────────┤
│  📋 契约规范 (Contract Specifications)   │
│  • Agent 契约 • Skill 契约 • 协议契约      │
│  • 系统调用契约 • 日志格式契约              │
├─────────────────────────────────────────┤
│  💻 编码规范 (Coding Standards)          │
│  • C/C++/Go/Rust/Python/JS 编码规范      │
│  • 安全编码指南 • 日志/审计标准            │
├─────────────────────────────────────────┤
│  📊 项目管理 (Project ERP)               │
│  • 错误码参考 • 资源管理表 • SBOM          │
│  • 模块需求手册                           │
├─────────────────────────────────────────┤
│  🔗 集成标准 (Integration Standards)     │
│  • 集成规范 • 生态伙伴计划 • 标准贡献       │
├─────────────────────────────────────────┤
│  📚 术语与索引 (Terminology & Index)     │
│  • 统一术语表 • 契约术语索引                │
└─────────────────────────────────────────┘
```

---

## 📋 契约规范 (Contract Specifications)

契约规范定义了 Airymax 各组件间的交互协议，确保系统的可组合性和可替换性。

### 1. Agent 契约

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [agent_contract.md](agentos_contract/agent_contract.md) | v1.3.0 | ✅ 生产就绪 | Agent 接口定义、生命周期、能力描述 |
| [agent_contract_schema.json](agentos_contract/agent_contract_schema.json) | v1.3.0 | ✅ 生产就绪 | Agent 契约 JSON Schema 验证文件 |

**核心要求**:
- 所有 Agent 必须实现契约定义的接口
- Agent 能力必须通过契约声明
- 契约版本必须与运行时兼容

### 2. Skill 契约

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [skill_contract.md](agentos_contract/skill_contract.md) | v1.3.0 | ✅ 生产就绪 | Skill 接口定义、输入输出、错误处理 |
| [skill_contract_schema.json](agentos_contract/skill_contract_schema.json) | v1.3.0 | ✅ 生产就绪 | Skill 契约 JSON Schema 验证文件 |

**核心要求**:
- Skill 必须声明输入输出格式
- 错误处理必须遵循统一规范
- 性能指标必须可观测

### 3. 通信协议

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [protocol_contract.md](agentos_contract/protocol_contract.md) | v1.3.0 | ✅ 生产就绪 | 组件间通信协议、消息格式、序列化 |

**核心要求**:
- 支持 JSON-RPC 2.0 协议
- 消息必须包含 TraceID
- 错误响应必须标准化

### 4. 系统调用 API

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [syscall_api_contract.md](agentos_contract/syscall_api_contract.md) | v1.3.0 | ✅ 生产就绪 | 系统调用接口定义、参数、返回值 |

**核心要求**:
- 所有系统调用必须文档化
- 参数验证必须严格
- 错误码必须统一

### 5. 日志格式

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [logging_format.md](agentos_contract/logging_format.md) | v1.3.0 | ✅ 生产就绪 | 结构化日志格式、字段定义、级别控制 |

**核心要求**:
- 日志必须结构化 (JSON)
- 必须包含 TraceID 贯穿
- 日志级别必须可配置

---

## 💻 编码规范 (Coding Standards)

编码规范确保代码质量、可维护性和安全性。

### 1. 语言编码风格

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [C_coding_style_standard.md](coding_standard/C_coding_style_standard.md) | v1.3.0 | ✅ 生产就绪 | C 语言编码风格、命名约定、代码组织 |
| [Cpp_coding_style_standard.md](coding_standard/Cpp_coding_style_standard.md) | v1.3.0 | ✅ 生产就绪 | C++ 编码风格、命名约定、代码组织 |
| [Go_coding_style_standard.md](coding_standard/Go_coding_style_standard.md) | v1.3.0 | ✅ 生产就绪 | Go 编码风格、命名约定、代码组织 |
| [Python_coding_style_standard.md](coding_standard/Python_coding_style_standard.md) | v1.3.0 | ✅ 生产就绪 | Python 编码风格、类型注解、异步编程 |
| [JavaScript_coding_style_standard.md](coding_standard/JavaScript_coding_style_standard.md) | v1.3.0 | ✅ 生产就绪 | JavaScript/TypeScript 编码规范 |
| [Rust_coding_style_standard.md](coding_standard/Rust_coding_style_standard.md) | v1.3.0 | ✅ 生产就绪 | Rust 编码风格、所有权、生命周期 |
| [Code_comment_template.md](coding_standard/Code_comment_template.md) | v1.3.0 | ✅ 生产就绪 | 代码注释模板、Doxygen 规范 |

### 2. 安全编码规范

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [C_Cpp_secure_coding_standard.md](coding_standard/C_Cpp_secure_coding_standard.md) | v1.3.0 | ✅ 生产就绪 | C/C++ 安全编码规范、漏洞防护 |
| [Go_secure_coding_standard.md](coding_standard/Go_secure_coding_standard.md) | v1.3.0 | ✅ 生产就绪 | Go 安全编码规范、并发安全 |
| [Java_secure_coding_standard.md](coding_standard/Java_secure_coding_standard.md) | v1.3.0 | ✅ 生产就绪 | Java 安全编码规范、输入验证 |
| [Rust_secure_coding_standard.md](coding_standard/Rust_secure_coding_standard.md) | v1.3.0 | ✅ 生产就绪 | Rust 安全编码规范、内存安全 |
| [Security_design_standard.md](coding_standard/Security_design_standard.md) | v1.3.0 | ✅ 生产就绪 | 安全设计原则、威胁建模 |

### 3. 日志与审计标准

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [Log_standard.md](coding_standard/Log_standard.md) | v1.3.0 | ✅ 生产就绪 | 日志记录最佳实践、性能优化 |
| [Config_Audit_Log_standard.md](coding_standard/Config_Audit_Log_standard.md) | v1.3.0 | ✅ 生产就绪 | 配置变更审计日志格式规范 |

**核心原则**:
- 日志级别必须合理使用
- 敏感信息必须脱敏
- 配置变更必须可审计

---

## 📊 项目管理 (Project ERP)

项目管理规范确保项目的可维护性和可追溯性。

### 1. 错误码参考

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [error_code_reference.md](project_erp/error_code_reference.md) | v1.3.0 | ✅ 生产就绪 | 统一错误码体系、分类、含义 |

**核心要求**:
- 错误码必须唯一
- 错误描述必须清晰
- 错误传播必须可追溯

**双错误码体系**:
Airymax 采用双错误码体系：
- **C 负整数体系（首要）**: 定义于 `agentos/commons/utils/error/include/error.h`，C 内核和 daemon 层必须使用（如 `AGENTOS_EINVAL=-2`）
- **SDK 十六进制体系（次要）**: 定义于 `error_code_reference.md`，SDK 和外部接口使用（如 `0x0000-0x7FFF` 分段）

> 详见 [TERMINOLOGY.md](TERMINOLOGY.md) 中"错误码体系"条目

### 2. 资源管理表

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [resource_management_table.md](project_erp/resource_management_table.md) | v1.3.0 | ✅ 生产就绪 | 项目资源清单、依赖关系、许可证 |

**核心要求**:
- 所有依赖必须记录
- 许可证必须兼容
- 版本必须明确

### 3. 软件物料清单 (SBOM)

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [SBOM.md](project_erp/SBOM.md) | v1.3.0 | ✅ 生产就绪 | 软件组成分析、安全漏洞扫描 |

**核心要求**:
- SBOM 必须自动生成
- 漏洞必须定期扫描
- 许可证必须合规

### 4. 模块需求手册

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [manuals_module_requirements.md](project_erp/manuals_module_requirements.md) | v1.3.0 | ✅ 生产就绪 | 各模块需求规范、功能边界、接口约定 |

---

## 🔗 集成标准 (Integration Standards)

集成标准定义了 Airymax 与外部系统的交互规范、生态合作和技术贡献路径。

### 1. 集成规范

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [INTEGRATION_STANDARD.md](integration_standards/INTEGRATION_STANDARD.md) | v1.3.0 | ✅ 生产就绪 | 第三方系统集成规范、适配器标准 |
| [README.md](integration_standards/README.md) | v1.3.0 | ✅ 生产就绪 | 集成标准目录、快速导航 |

### 2. 生态合作

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [ecosystem_partnership.md](integration_standards/ecosystem_partnership.md) | v1.3.0 | ✅ 生产就绪 | 生态伙伴计划、合作模式、权益说明 |

### 3. 标准贡献

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [standards_contribution.md](integration_standards/standards_contribution.md) | v1.3.0 | ✅ 生产就绪 | MCP/A2A 等标准参与路径、技术方案贡献 |

---

## 🌐 开放标准 (Open Standards / ARE Standards)

开放标准定义 Airymax 推动的"行业底座"开放标准体系（ARE Standards — Agent Runtime Environment Standards），目标是定义 Agent Runtime Environment 的三层标准接口，使第三方可独立实现兼容的 Agent 运行时。**版权归属人：SPHARX Ltd.** | 许可证：AGPL v3 + Apache 2.0 双许可证（SPDX: `AGPL-3.0-or-later OR Apache-2.0`）。

### 标准分层架构

```
┌─────────────────────────────────────────────────┐
│            ARE Standards 三层标准                  │
├─────────────────────────────────────────────────┤
│  L3 安全与治理                                    │
│  • Cupolas 权限引擎 • 五级沙箱 • 统一错误码       │
│  • 审计日志 • 双许可证体系 • SBOM                  │
├─────────────────────────────────────────────────┤
│  L2 服务通信协议                                  │
│  • IPC 消息头（128 字节）• JSON-RPC 命名空间       │
│  • 服务发现多后端 • trace_id 贯穿                  │
├─────────────────────────────────────────────────┤
│  L1 核心运行时接口                                │
│  • 微核心原语（IPC/Mem/Task/Time/Sync）           │
│  • 生命周期 • ops 注入机制 • 一致性测试            │
└─────────────────────────────────────────────────┘
```

### 1. ARE Standards 总览

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [are_standards/README.md](are_standards/README.md) | v0.1.0-draft | 📝 草案 | ARE Standards 三层标准总览、独立仓库可行性、标准化路线图 |
| [are_standards/L1_runtime_interface.md](are_standards/L1_runtime_interface.md) | v0.1.0-draft | 📝 草案 | L1 核心运行时接口规范（微核心原语 + ops 注入） |
| [are_standards/L2_service_protocol.md](are_standards/L2_service_protocol.md) | v0.1.0-draft | 📝 草案 | L2 服务通信协议规范（IPC 消息头 + JSON-RPC 命名空间） |
| [are_standards/L3_security_governance.md](are_standards/L3_security_governance.md) | v0.1.0-draft | 📝 草案 | L3 安全与治理规范（Cupolas + 错误码 + 双许可证） |

**核心要求**:
- 三层标准接口使第三方可独立实现兼容 Agent 运行时
- 参考实现保留在 Airymax 主仓库，标准文档独立发布
- 标准化路线：草案（v0.1.1）→ 试用（v0.2.0）→ 候选（v0.3.0）→ 正式（v1.0.0）

### 2. SDK 标准规范

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [sdk_standard/README.md](sdk_standard/README.md) | v0.1.0-draft | 📝 草案 | SDK 双层 API（Manager + 嵌套资源）+ Cognition/Safety/Tool/Chat 客户端覆盖 |

**核心要求**:
- 双层 API：Manager 层（向后兼容）+ 嵌套资源 API 层（对齐 OpenAI `/v1/chat/completions`）
- 四语言 SDK（Rust/Python/Go/TypeScript）同步实现
- 显式覆盖 CognitionClient/SafetyClient/ToolClient/ChatClient

### 3. IPC Bus 消息头规范

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [ipc_standard/README.md](ipc_standard/README.md) | v0.1.0-draft | 📝 草案 | `are_ipc_message_header_t` 128 字节统一消息头 + 开放标准推动路径 |

**核心要求**:
- 统一消息头 128 字节（magic/version/trace_id/correlation_id/source/target）
- magic=0x41524531 ("ARE1") 校验
- 开放标准推动：v0.1.1 文档 → v0.2.0 参考实现 → v0.3.0 IETF 草案 → v1.0.0 正式

### 4. 服务发现规范

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [service_discovery_standard/README.md](service_discovery_standard/README.md) | v0.1.0-draft | 📝 草案 | 多后端适配器架构（shm/dns-sd/consul/etcd/k8s）+ 统一接口 |

**核心要求**:
- 统一适配器接口 `are_svc_discovery_adapter_t`
- 五种后端：SHM（高性能本地）/ DNS-SD（局域网）/ Consul（生产）/ etcd（K8s 原生）/ K8s（原生 Service）
- 配置选择：`agentos.yaml` 的 `service_discovery.backend` 字段

### 5. JSON-RPC API 规范

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [rpc_api_standard/README.md](rpc_api_standard/README.md) | v0.1.0-draft | 📝 草案 | 强制 `<daemon>.<method>` 命名空间 + 12 daemon 方法清单 + 第三方实现指南 |

**核心要求**:
- 强制命名空间格式 `<daemon>.<method>`（如 `llm.complete`、`tool.execute`）
- 12 个 daemon 命名空间：`llm.`/`tool.`/`sched.`/`mem.`/`agent.`/`a2a.`/`mcp.`/`hook.`/`plugin.`/`cupolas.`/`monit.`/`market.`
- 消除扁平方法名冲突（如 `register_agent`/`health_check` 跨 daemon 冲突）

### 6. 错误码规范

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [error_code_standard/README.md](error_code_standard/README.md) | v0.1.0-draft | 📝 草案 | 统一错误码分段 + 权威源唯一性 + Cupolas 错误码完全统一 |

**核心要求**:
- 权威源唯一：`agentos/commons/utils/error/include/error.h` 为唯一定义源
- 分段规范：通用 -1~-99 / 系统 -100~-199 / 内核 -200~-299 / 服务 -300~-399 / LLM -400~-499 / 执行 -500~-599 / 记忆 -600~-699 / 安全 -700~-799 / 协议 -800~-899
- 消除三套并行错误码系统（commons / cupolas enum / cupolas 本地 #define）

### 独立仓库可行性

**结论**: 高度可行。
- `Docs/` 本身已是独立 git submodule（`git@atomgit.com:openairymax/docs.git`）
- `Capital_Specifications/are_standards/` 作为标准容器，未来可独立发布为 `are-standards` 仓库
- 参考实现保留在 Airymax 主仓库，标准文档独立发布
- 治理模型：SPHARX Ltd. 主导标准制定 + 社区贡献流程

---

## 📚 术语与索引 (Terminology & Index)

### 1. 统一术语表

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [TERMINOLOGY.md](TERMINOLOGY.md) | v1.3.0 | ✅ 生产就绪 | Airymax 统一术语定义、交叉引用 |

**核心要求**:
- 术语定义必须准确
- 交叉引用必须完整
- 版本必须同步更新

### 2. 术语索引

| 文档 | 版本 | 状态 | 描述 |
|------|------|------|------|
| [glossary_index.md](agentos_contract/glossary_index.md) | v1.3.0 | ✅ 生产就绪 | 契约术语索引、快速查找 |

**核心要求**:
- 索引必须按字母顺序
- 链接必须有效
- 更新必须及时

---

## 🔧 规范实施与验证

### 1. 规范实施流程

```
设计阶段 → 编码阶段 → 测试阶段 → 部署阶段
    ↓          ↓          ↓          ↓
契约设计 → 代码审查 → 契约测试 → 合规检查
    ↓          ↓          ↓          ↓
架构评审 → 静态分析 → 集成测试 → 安全审计
```

### 2. 验证工具

```bash
# 契约验证
python scripts/validate_contract.py --contract agent_contract.json

# 代码规范检查
python scripts/lint_code.py --lang c --path agentos/atoms/

# 安全扫描
python scripts/security_scan.py --sbom sbom.json

# 文档验证
python scripts/validate_docs.py --check-links
```

### 3. 合规检查清单

**契约合规**:
- [ ] 所有接口是否实现契约定义？
- [ ] 错误处理是否遵循规范？
- [ ] 性能指标是否可观测？

**编码合规**:
- [ ] 代码是否通过静态分析？
- [ ] 安全漏洞是否修复？
- [ ] 许可证是否合规？

**文档合规**:
- [ ] 术语使用是否一致？
- [ ] 交叉引用是否有效？
- [ ] 版本信息是否准确？

---

## 📈 规范演进机制

### 1. 版本管理

规范版本与 Airymax 主版本同步：

- **MAJOR 版本**: 不兼容的规范变更
- **MINOR 版本**: 向后兼容的功能新增
- **PATCH 版本**: 问题修复和澄清

### 2. 变更流程

1. **变更提案**: 提交规范变更请求
2. **影响评估**: 评估变更对现有系统的影响
3. **架构评审**: 架构委员会评审变更
4. **试点实施**: 在小范围试点新规范
5. **全面推广**: 试点成功后全面推广

### 3. 兼容性保证

| 变更类型 | 兼容性保证 | 通知要求 |
|---------|-----------|---------|
| **新增功能** | 完全兼容 | 提前 1 个月通知 |
| **功能增强** | 完全兼容 | 提前 2 周通知 |
| **行为变更** | 可能不兼容 | 提前 3 个月通知 |
| **接口废弃** | 不兼容 | 提前 6 个月通知 |

---

## 🛡️ 安全与合规

### 1. 安全要求

- **输入验证**: 所有用户输入必须验证
- **输出编码**: 所有输出必须适当编码
- **错误处理**: 错误信息不得泄露敏感数据
- **访问控制**: 必须实现最小权限原则

### 2. 合规要求

- **许可证合规**: 所有组件许可证必须兼容
- **数据保护**: 必须遵守数据保护法规
- **审计追踪**: 所有操作必须可审计
- **安全更新**: 必须定期应用安全更新

### 3. 安全审查

```bash
# 定期安全审查
python scripts/security_review.py --full

# 依赖漏洞扫描
python scripts/dependency_scan.py --update

# 许可证合规检查
python scripts/license_compliance.py --report
```

---

## 🤝 贡献规范

### 1. 贡献流程

1. **发现问题**: 识别规范不完善或错误之处
2. **提交提案**: 提交规范变更提案
3. **讨论评审**: 参与社区讨论和架构评审
4. **实现验证**: 实现变更并通过测试
5. **文档更新**: 更新相关文档

### 2. 提案模板

```markdown
# 规范变更提案

## 问题描述
[描述需要解决的问题]

## 变更内容
[描述具体的规范变更]

## 影响分析
- 对现有系统的影响
- 兼容性考虑
- 迁移路径

## 实施计划
- 时间安排
- 资源需求
- 测试方案

## 参考资料
- 相关规范文档
- 技术背景
- 类似案例
```

### 3. 评审标准

- **技术正确性**: 变更是否技术正确？
- **实用性**: 变更是否解决实际问题？
- **兼容性**: 变更是否保持向后兼容？
- **可维护性**: 变更是否易于维护？

---

## 📞 支持与反馈

### 1. 问题报告

- **GitHub Issues**: [规范问题报告](https://github.com/SpharxTeam/AgentRT/issues)
- **邮件列表**: specifications@agentos.io
- **Discord**: #specifications 频道

### 2. 技术支持

- **规范解释**: standards@agentos.io
- **合规咨询**: compliance@agentos.io
- **安全报告**: security@agentos.io

### 3. 培训资源

- **规范培训**: 每月一次的规范培训会议
- **在线课程**: Airymax 规范体系在线课程
- **认证考试**: Airymax 规范认证考试

---

## 📚 相关文档

- [架构设计原则](../ARCHITECTURAL_PRINCIPLES.md)
- [编码规范总览](./coding_standard/README.md)
- [术语表](TERMINOLOGY.md)
- [集成标准导航](./integration_standards/README.md)
- [API 参考](../Capital_API/README.md)

---

## 📝 版本历史

| 版本 | 日期 | 作者 | 变更说明 |
|------|------|------|----------|
| Doc V2.3 | 2026-07-04 | SPHARX Ltd. | 新增"开放标准 (ARE Standards)"章节，覆盖 6 个标准子目录（are_standards/sdk_standard/ipc_standard/service_discovery_standard/rpc_api_standard/error_code_standard）+ L1/L2/L3 三层标准架构；统一许可证为 AGPL v3 + Apache 2.0 双许可证（SPDX: `AGPL-3.0-or-later OR Apache-2.0`），版权人 SPHARX Ltd. |
| Doc V2.2 | 2026-06-08 | Airymax 规范委员会 | 统一版本号至 v1.3.0，新增双错误码体系说明，新增 Rust/Go 编码风格与安全编码标准 |
| Doc V2.1 | 2026-06-07 | Airymax 规范委员会 | 修正所有失效链接，新增集成标准章节，整理散落文件 |
| Doc V2.0 | 2026-03-31 | Airymax Team | 更新规范版本，完善内容结构 |
| Doc V2.0 | 2026-03-23 | Airymax 规范委员会 | 初始版本，建立技术规范体系 |

---

**最后更新**: 2026-07-04
**维护者**: SPHARX Ltd.

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
*"From data intelligence emerges."*
