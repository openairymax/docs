Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax 文档中心  

**语言:** [English](README.md) | 简体中文

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/README.md
---

## 📚 文档导航

欢迎使用Airymax智能体操作系统文档中心。本文档采用分层组织结构，从入门到精通，满足不同层次的需求。

### 🎯 快速入口

| 角色 | 推荐阅读路径 | 预计时间 |
|------|------------|---------|
| **初学者** | 快速开始 → 安装指南 → 配置指南 | 30分钟 |
| **开发者** | API参考 → 编码规范 → 创建Agent/Skill | 2小时 |
| **运维工程师** | 部署指南 → 监控运维 → 故障排查 | 1.5小时 |
| **架构师** | 设计原则 → 微核心架构 → 核心循环三层 | 3小时 |

---

## 📖 文档分类

### 1️⃣ 基础设计 (Basic Theories)

Airymax 的设计基石，理解设计哲学的必读材料：

- [**体系并行 (MCIS)**](00-basic-theories/01-mcis-cn.md) — 多体控制智能系统（[English](00-basic-theories/01-mcis.md)）
- [**认知层设计**](00-basic-theories/02-cognition-design-cn.md) — 双思考系统与三层架构（[English](00-basic-theories/02-cognition-design.md)）
- [**记忆层设计**](00-basic-theories/03-memory-design-cn.md) — 四层记忆卷载模型（[English](00-basic-theories/03-memory-design.md)）
- [**设计原则**](00-basic-theories/04-design-principles-cn.md) — 五维正交设计原则导论（[English](00-basic-theories/04-design-principles.md)）
- [**设计哲学**](10-architecture/40-philosophy/01-design-philosophy.md) — 体系并行与五维正交设计体系详述
- [**架构设计原则完整版**](00-architectural-principles.md) — S/K/C/E/A五维24条原则详述

---

### 2️⃣ 入门指南 (Guides)

面向新用户的引导式教程，帮助快速上手：

- [**快速开始**](60-guides/01-getting-started.md) — 5分钟从零到Hello World
- [**配置指南**](60-guides/04-configuration-guide.md) — 完整的配置选项说明
- [**部署指南**](60-guides/03-deployment-guide.md) — 生产环境部署最佳实践（Docker/K8s/监控/安全）
- [**创建Agent**](60-guides/24-create-agent.md) — Agent开发完整流程
- [**创建Skill**](60-guides/25-create-skill.md) — Skill开发完整流程
- [**Plugin SDK教程**](60-guides/17-plugin-sdk-tutorial.md) — Plugin SDK完整开发指南
- [**Prompt工程指南**](60-guides/19-prompt-engineering.md) — Prompt模板/注入/调优全生命周期
- [**迁移指南**](60-guides/10-migration-guide.md) — 版本升级与数据迁移
- [**构建指南**](60-guides/02-build-guide.md) — 构建系统与编译选项
- [**测试规范**](60-guides/15-testing-standards.md) — 测试分层与覆盖率标准
- [**性能调优**](60-guides/09-performance-tuning.md) — 内核参数与性能优化
- [**协议集成**](60-guides/18-protocol-integration.md) — 多协议适配与路由

---

### 3️⃣ 架构设计 (Architecture)

深入理解Airymax的设计哲学和技术实现：

- [**微核心架构**](10-architecture/04-microcorert.md) — K-1~K-4 原则的实现
- [**核心循环三层**](10-architecture/02-coreloopthree.md) — Cognition→Execution→Memory
- [**记忆卷载系统**](10-architecture/03-memoryrovol.md) — L1→L2→L3→L4 四层记忆
- [**IPC通信机制**](10-architecture/30-kernel/01-ipc.md) — Binder/Channel/Buffer 进程间通信
- [**系统调用架构**](10-architecture/05-syscall.md) — 用户态与内核态的统一接口
- [**日志系统架构**](10-architecture/50-services/02-logging.md) — 跨语言可观测性与动态反馈调节
- [**架构概述**](10-architecture/01-system-architecture.md) — 系统总览与分层架构图
- [**C语言边界定义**](10-architecture/30-kernel/02-c-language-boundary.md) — C核心职责范围与FFI边界

---

### 4️⃣ API 参考 (API Reference)

完整的接口文档，包含请求示例和响应格式：

**系统调用 API**：

- [**API总览**](30-api/README.md) — API层次结构与设计哲学
- [**Gateway API参考**](30-api/02-api-reference.md) — HTTP/WebSocket端点API参考
- [**CLI命令参考**](30-api/03-cli-reference.md) — 统一命令行工具完整命令
- [**任务管理 API**](30-api/60-syscalls/05-task.md) — submit/query/wait/cancel 任务全生命周期
- [**记忆管理 API**](30-api/60-syscalls/02-memory.md) — write/query/evolve/forget 四层记忆操作
- [**会话管理 API**](30-api/60-syscalls/03-session.md) — create/get/close/list 会话管理
- [**可观测性 API**](30-api/60-syscalls/06-telemetry.md) — metrics/traces 遥测接口
- [**Agent管理 API**](30-api/60-syscalls/01-agent.md) — spawn/terminate/invoke Agent管理

**多语言 SDK**：

- [**Python SDK**](30-api/70-toolkit/20-python/README.md) — Python语言绑定API
- [**Go SDK**](30-api/70-toolkit/10-go/README.md) — Go语言绑定API
- [**Rust SDK**](30-api/70-toolkit/30-rust/README.md) — Rust语言绑定API
- [**TypeScript SDK**](30-api/70-toolkit/40-typescript/README.md) — TypeScript语言绑定API

**核心算法**：

- [**算法实现文档**](30-api/10-algorithms/README.md) — 文档处理/搜索索引/质量验证/性能优化核心算法

---

### 5️⃣ 开发者指南 (Development)

参与Airymax开发的必备知识：

**贡献与测试**：

- [**贡献指南**](../agentrt/CONTRIBUTING.md) — 提交PR的完整流程
- [**测试指南**](60-guides/14-testing-guide.md) — 单元测试、集成测试、E2E测试

**编码规范**：

- [**C编码风格规范**](50-specifications/30-coding-standard/01-c-coding-style.md) — C语言命名/函数/错误处理/并发/安全规范
- [**C++编码风格规范**](50-specifications/30-coding-standard/05-cpp-coding-style.md) — C++语言编码规范
- [**Python编码风格规范**](50-specifications/30-coding-standard/12-python-coding-style.md) — Python类型设计/异步/错误处理规范
- [**JavaScript编码风格规范**](50-specifications/30-coding-standard/08-javascript-coding-style.md) — JavaScript/TypeScript编码规范
- [**命名规范**](50-specifications/30-coding-standard/11-naming-conventions.md) — 组件/文件/函数/类型/常量命名约定
- [**代码注释模板**](50-specifications/30-coding-standard/03-code-comment-template.md) — Doxygen/docstring注释规范
- [**日志打印规范**](50-specifications/30-coding-standard/10-log-standard.md) — 日志级别/内容/格式/质量规范

**安全编码**：

- [**安全设计规范**](50-specifications/30-coding-standard/15-security-design.md) — D1~D4四层防护/加密/认证/隐私保护
- [**C/C++安全编码规范**](50-specifications/30-coding-standard/02-c-cpp-secure-coding.md) — C/C++安全编码实践
- [**Java安全编码规范**](50-specifications/30-coding-standard/09-java-secure-coding.md) — Java安全编码实践

---

### 6️⃣ 运维手册 (Operations)

生产环境的运维保障：

- [**Docker部署**](../Docker/README.md) — 容器化部署完整方案
- [**监控运维**](60-guides/07-monitoring-guide.md) — Prometheus+Grafana监控栈
- [**备份恢复**](60-guides/11-backup-recovery.md) — 数据备份与灾难恢复
- [**内核调优**](60-guides/09-performance-tuning.md) — 内核参数调优与性能优化

---

### 7️⃣ 故障排查 (Troubleshooting)

常见问题及解决方案：

- [**常见问题FAQ**](60-guides/23-troubleshooting-faq.md) — 故障排查与高频问题
- [**错误诊断**](60-guides/08-diagnosis-guide.md) — 日志分析与问题定位
- [**已知问题**](60-guides/22-known-issues.md) — 已知Bug及临时解决方案

---

### 8️⃣ 规范与契约 (Specifications)

Airymax项目的标准化规范体系：

**契约规范**：

- [**契约规范总览**](50-specifications/10-contracts/06-glossary-index.md) — 术语表与快速索引
- [**Agent契约**](50-specifications/10-contracts/01-agent-contract.md) — Agent能力描述规范（[Schema](50-specifications/10-contracts/01-agent-contract-schema.json)）
- [**Skill契约**](50-specifications/10-contracts/02-skill-contract.md) — Skill技能描述规范（[Schema](50-specifications/10-contracts/02-skill-contract-schema.json)）
- [**通信协议规范**](50-specifications/10-contracts/03-protocol-contract.md) — HTTP/WS/Stdio网关+JSON-RPC 2.0
- [**系统调用API规范**](50-specifications/10-contracts/04-syscall-api-contract.md) — 系统调用接口契约
- [**日志格式规范**](50-specifications/10-contracts/05-logging-format.md) — 结构化JSON日志格式

**集成标准**：

- [**集成标准总览**](50-specifications/50-integration/README.md) — 模块间集成标准索引
- [**Manager配置集成标准**](50-specifications/50-integration/01-integration-standard.md) — Manager模块与统一配置库集成

**项目管理**：

- [**错误码参考**](50-specifications/70-project-erp/02-error-code-reference.md) — 完整的错误码定义和处理建议
- [**资源管理表**](50-specifications/70-project-erp/04-resource-management-table.md) — 资源创建/释放/所有权规范
- [**软件物料清单 (SBOM)**](50-specifications/70-project-erp/01-sbom.md) — 组件/依赖/许可证/安全信息
- [**模块功能需求**](50-specifications/70-project-erp/03-manuals-module-requirements.md) — manuals模块需求与技术规范

**术语**：

- [**统一术语表**](50-specifications/10-terminology.md) — Airymax统一术语定义

---

### 9️⃣ 白皮书与模板 (White Paper & Templates)

- [**技术白皮书**](90-references/01-white-paper.md) — Airymax官方技术白皮书（中/英）
- [**文档模板**](90-references/02-template.md) — 通用文档模板
- [**API文档模板**](90-references/03-template-api.md) — API文档编写模板
- [**指南文档模板**](90-references/04-template-guide.md) — 指南文档编写模板

---

### 🔟 参考资料 (References)

补充材料和外部链接：

- [**统一术语表**](50-specifications/10-terminology.md) — 统一术语定义
- [**变更日志**](../CHANGELOG.md) — 版本更新历史
- [**许可证**](../agentrt/LICENSE) — AGPL v3 + Apache 2.0 双许可证全文

---

## 🔍 文档使用技巧

### 搜索功能

使用 `Ctrl+F` 或 `Cmd+F` 在当前页面搜索关键词。

跨页面搜索可以使用以下命令：

```bash
# 在所有Markdown文件中搜索"IPC"
grep -r "IPC" docs/ --include="*.md"
```

### 文档反馈

发现文档错误或有改进建议？

1. 在对应文档页面点击右上角的 **编辑此页** 按钮
2. 直接修改并提交PR
3. 或者在 [AtomGit Issues](https://atomgit.com/openairymax/docs/issues) 反馈

### 版本选择

本文档始终与最新稳定版代码同步。如需查看历史版本文档：

```bash
# 切换到指定版本的文档
git checkout v1.0.0 -- docs/
```

---

## 📊 文档统计

| 类别 | 文档数量 | 总字数(估) | 最后更新 |
|------|---------|-----------|---------|
| 基础理论 | 9篇 | ~35,000字 | 2026-04-09 |
| 入门指南 | 7篇 | ~25,000字 | 2026-04-09 |
| 架构设计 | 6篇 | ~40,000字 | 2026-04-09 |
| API参考 | 11篇 | ~45,000字 | 2026-04-09 |
| 开发者指南 | 10篇 | ~40,000字 | 2026-04-09 |
| 运维手册 | 6篇 | ~30,000字 | 2026-04-09 |
| 故障排查 | 3篇 | ~12,000字 | 2026-04-09 |
| 规范与契约 | 14篇 | ~55,000字 | 2026-04-09 |
| 白皮书与模板 | 4篇 | ~15,000字 | 2026-04-09 |
| 参考资料 | 3篇 | ~8,000字 | 2026-04-09 |

**总计**: 73篇文档，约305,000字

---

## 📂 文档目录结构

```
docs/
├── 00-architectural-principles.md   # 五维正交设计原则完整版
├── 00-basic-theories/               # 基础设计（中/英双语）
│   ├── 01-mcis.md                   # 体系并行论（英文）
│   ├── 01-mcis-cn.md                # 体系并行论（中文）
│   ├── 02-cognition-design.md       # 认知层设计（英文）
│   ├── 02-cognition-design-cn.md    # 认知层设计（中文）
│   ├── 03-memory-design.md          # 记忆层设计（英文）
│   ├── 03-memory-design-cn.md       # 记忆层设计（中文）
│   ├── 04-design-principles.md      # 系统设计原则（英文）
│   └── 04-design-principles-cn.md   # 系统设计原则（中文）
├── 10-architecture/                 # 架构设计
│   ├── 01-system-architecture.md    # 系统总览与分层架构图
│   ├── 02-coreloopthree.md          # 三层认知循环
│   ├── 03-memoryrovol.md            # 记忆卷载系统
│   ├── 04-microcorert.md            # 微核心 (MicroCoreRT) 架构详解
│   ├── 05-syscall.md                # 系统调用架构
│   ├── 20-engineering/              # 工程实践
│   │   ├── 01-testing.md            # 测试架构
│   │   └── 02-toolkit.md            # 工具链设计
│   ├── 30-kernel/                   # 内核子系统
│   │   ├── 01-ipc.md                # IPC 通信机制
│   │   └── 02-c-language-boundary.md # C 语言边界
│   ├── 40-philosophy/               # 设计哲学
│   │   └── 01-design-philosophy.md  # 体系并行与五维正交设计体系
│   ├── 50-services/                 # 服务子系统
│   │   ├── 01-daemon.md             # Daemon 用户态服务
│   │   └── 02-logging.md            # 日志系统架构
│   └── 60-diagrams/                 # 架构图（drawio）
│       ├── 01-coreloopthree-flow.drawio
│       ├── 02-memoryrovol-layers.drawio
│       └── 03-overall-architecture.drawio
├── 10-terminology.md                # 统一术语表
├── 30-api/                          # API参考
│   ├── README.md
│   ├── 01-doxygen-guide.md          # Doxygen 文档指南
│   ├── 02-api-reference.md          # Gateway API 参考手册
│   ├── 03-cli-reference.md          # CLI 命令参考手册
│   ├── 10-algorithms/               # 核心算法
│   │   └── README.md
│   ├── 20-core/                     # 核心 API
│   │   └── 01-coreloop-api.md       # CoreLoop API
│   ├── 30-daemon/                   # Daemon API
│   │   ├── 01-api-documentation.md  # Daemon API 文档
│   │   └── 02-gateway-api.md        # Gateway API
│   ├── 40-docker/                   # Docker 集成
│   │   └── README.md
│   ├── 50-examples/                 # 示例
│   │   └── 01-quickstart.md         # 快速入门
│   ├── 60-syscalls/                 # 系统调用 API
│   │   ├── 01-agent.md
│   │   ├── 02-memory.md
│   │   ├── 03-session.md
│   │   ├── 04-skill.md
│   │   ├── 05-task.md
│   │   └── 06-telemetry.md
│   └── 70-toolkit/                  # 多语言 SDK
│       ├── 01-protocol-guide.md     # 协议指南
│       ├── 02-protocol-quickstart.md # 协议快速入门
│       ├── 10-go/README.md
│       ├── 20-python/README.md
│       ├── 30-rust/README.md
│       └── 40-typescript/README.md
├── 50-specifications/               # 规范与契约
│   ├── README.md
│   ├── 10-contracts/                # 契约规范
│   │   ├── 01-agent-contract.md
│   │   ├── 02-skill-contract.md
│   │   ├── 03-protocol-contract.md
│   │   ├── 04-syscall-api-contract.md
│   │   ├── 05-logging-format.md
│   │   ├── 06-glossary-index.md
│   │   ├── 01-agent-contract-schema.json
│   │   └── 02-skill-contract-schema.json
│   ├── 20-are-standards/            # ARE 标准 (L1-L3)
│   │   ├── README.md
│   │   ├── 01-l1-runtime-interface.md
│   │   ├── 02-l2-service-protocol.md
│   │   └── 03-l3-security-governance.md
│   ├── 30-coding-standard/          # 编码规范
│   │   ├── 01-c-coding-style.md
│   │   ├── 02-c-cpp-secure-coding.md
│   │   ├── 03-code-comment-template.md
│   │   ├── 04-config-audit-log.md
│   │   ├── 05-cpp-coding-style.md
│   │   ├── 06-go-coding-style.md
│   │   ├── 07-go-secure-coding.md
│   │   ├── 08-javascript-coding-style.md
│   │   ├── 09-java-secure-coding.md
│   │   ├── 10-log-standard.md
│   │   ├── 11-naming-conventions.md
│   │   ├── 12-python-coding-style.md
│   │   ├── 13-rust-coding-style.md
│   │   ├── 14-rust-secure-coding.md
│   │   └── 15-security-design.md
│   ├── 40-error-code/               # 错误码标准
│   │   └── README.md
│   ├── 50-integration/              # 集成标准
│   │   ├── README.md
│   │   ├── 01-integration-standard.md
│   │   ├── 02-ecosystem-partnership.md
│   │   └── 03-standards-contribution.md
│   ├── 60-ipc/                      # IPC 标准
│   │   └── README.md
│   ├── 70-project-erp/              # 项目管理
│   │   ├── 01-sbom.md
│   │   ├── 02-error-code-reference.md
│   │   ├── 03-manuals-module-requirements.md
│   │   └── 04-resource-management-table.md
│   ├── 80-rpc-api/                  # RPC API 标准
│   │   └── README.md
│   ├── 90-sdk/                      # SDK 标准
│   │   └── README.md
│   └── 95-service-discovery/        # 服务发现标准
│       └── README.md
├── 60-guides/                       # 入门与运维指南
│   ├── 01-getting-started.md
│   ├── 02-build-guide.md
│   ├── 03-deployment-guide.md
│   ├── 04-configuration-guide.md
│   ├── 05-config-change-process.md
│   ├── 06-config-drift-detector.md
│   ├── 07-monitoring-guide.md
│   ├── 08-diagnosis-guide.md
│   ├── 09-performance-tuning.md
│   ├── 10-migration-guide.md
│   ├── 11-backup-recovery.md
│   ├── 12-security-gateway.md
│   ├── 13-security-hardening.md
│   ├── 14-testing-guide.md
│   ├── 15-testing-standards.md
│   ├── 16-ci-cd-pipelines.md
│   ├── 17-plugin-sdk-tutorial.md
│   ├── 18-protocol-integration.md
│   ├── 19-prompt-engineering.md
│   ├── 20-manager-development.md
│   ├── 21-best-practices.md
│   ├── 22-known-issues.md
│   ├── 23-troubleshooting-faq.md
│   ├── 24-create-agent.md
│   ├── 25-create-skill.md
│   └── 26-coreloopthree-dag-integration.md
├── 90-references/                   # 其他资源
│   └── README.md
├── README.md                        # 英文文档入口
└── README_zh.md                     # 中文文档入口（本文档）
```

---

## 🎯 文档质量标准

Airymax文档遵循**完美主义原则 (A-4)**：

✅ **完整性**：每个公共API都有文档  
✅ **准确性**：示例代码可运行，配置参数经过验证  
✅ **及时性**：代码变更后24小时内同步更新文档  
✅ **易读性**：使用清晰的语言，避免过度技术化  
✅ **可操作性**：每个指南都提供分步操作说明  

---

## 📞 联系方式


---

**© 2025-2026 SPHARX Ltd. All Rights Reserved.**
