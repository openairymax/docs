Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# AirymaxAgentRT 文档中心

**语言**: [English](README.md) | 简体中文

**最新更新**: 2026-07-11
**状态**: 维护中
**路径**: `docs/AirymaxRT/README_zh.md`

---

## 📚 文档导航

AirymaxAgentRT（极境智能体运行底座平台工程）是用户态 AI Agent 运行时，定位为**微核心**（micro-core）。本文档中心覆盖 agentrt 的需求规格、系统架构、模块设计、对外接口和应用开发。

> **工程标准**: agentrt 与 agentrt-linux 共用一套工程标准，物理宿主在 [`docs/AirymaxOS/50-engineering-standards/`](../AirymaxOS/50-engineering-standards/)，通过符号链接 `50-engineering-standards` 引用。

---

## 📖 文档目录

### 1️⃣ 需求规格（00-requirements）

agentrt 的功能需求、非功能需求与业务约束（中英双语）：

- [**体系并行论 (MCIS)**](00-requirements/01-mcis-cn.md) — 多体控制智能系统（[English](00-requirements/01-mcis.md)）
- [**认知层设计**](00-requirements/02-cognition-design-cn.md) — 双思考系统与三层架构（[English](00-requirements/02-cognition-design.md)）
- [**记忆层设计**](00-requirements/03-memory-design-cn.md) — 四层记忆卷载模型（[English](00-requirements/03-memory-design.md)）
- [**设计原则**](00-requirements/04-design-principles-cn.md) — 五维正交设计原则导论（[English](00-requirements/04-design-principles.md)）

### 2️⃣ 系统架构（10-architecture）

agentrt 的设计哲学、架构原则与核心组件设计：

- [**架构设计原则**](10-architecture/00-architectural-principles.md) — S/K/C/E/A 五维正交 24 原则
- [**系统总览**](10-architecture/01-system-architecture.md) — 分层架构与组件总览
- [**CoreLoopThree**](10-architecture/02-coreloopthree.md) — 认知→执行→记忆三阶段循环
- [**MemoryRovol**](10-architecture/03-memoryrovol.md) — L1→L2→L3→L4 四层记忆卷载
- [**MicroCoreRT**](10-architecture/04-microcorert.md) — 微核心架构（K-1~K-4 原则）
- [**Syscall 架构**](10-architecture/05-syscall.md) — 用户态/内核态统一接口
- [**IRON-9 v2 共享模型**](10-architecture/06-iron9-shared-model.md) — 跨端共享契约层
- [**IPC 通信机制**](10-architecture/30-kernel/01-ipc.md) — 进程间通信架构
- [**C 语言边界**](10-architecture/30-kernel/02-c-language-boundary.md) — C 核心职责与 FFI 边界
- [**设计哲学**](10-architecture/40-philosophy/01-design-philosophy.md) — 体系并行与五维正交设计体系
- [**Daemon 服务**](10-architecture/50-services/01-daemon.md) — 用户态服务架构
- [**日志系统**](10-architecture/50-services/02-logging.md) — 跨语言可观测性

### 3️⃣ 模块设计（20-modules）

agentrt 各子模块的契约规范：

- [**Agent 契约**](20-modules/10-contracts/01-agent-contract.md) — Agent 能力描述规范
- [**Skill 契约**](20-modules/10-contracts/02-skill-contract.md) — Skill 技能描述规范
- [**通信协议规范**](20-modules/10-contracts/03-protocol-contract.md) — HTTP/WS/Stdio 网关 + JSON-RPC 2.0
- [**系统调用 API 规范**](20-modules/10-contracts/04-syscall-api-contract.md) — 系统调用接口契约
- [**日志格式规范**](20-modules/10-contracts/05-logging-format.md) — 结构化 JSON 日志格式
- [**术语索引**](20-modules/10-contracts/06-glossary-index.md) — 术语表与快速索引

### 4️⃣ 对外接口（30-interfaces）

agentrt 的 UAPI、ABI、SDK 公共接口设计：

- [**接口文档总览**](30-interfaces/README.md) — API 层次结构
- [**Gateway API 参考**](30-interfaces/02-api-reference.md) — HTTP/WebSocket 端点 API
- [**CLI 命令参考**](30-interfaces/03-cli-reference.md) — 统一命令行工具
- [**L1-L3 运行时接口**](30-interfaces/05-runtime-interfaces/README.md) — ARE 标准接口（L1 运行时 / L2 服务协议 / L3 安全治理）
- [**IPC 协议**](30-interfaces/06-ipc/README.md) — 128B 消息头、payload 类型、请求-响应/发布-订阅
- [**RPC API**](30-interfaces/07-rpc-api/README.md) — JSON-RPC 2.0 接口规范
- [**服务发现**](30-interfaces/08-service-discovery/README.md) — 服务注册与发现机制
- [**核心算法**](30-interfaces/10-algorithms/README.md) — 文档处理/搜索索引/质量验证
- [**CoreLoop API**](30-interfaces/20-core/01-coreloop-api.md) — 核心循环 API
- [**Daemon API**](30-interfaces/30-daemon/01-api-documentation.md) — Daemon 接口文档
- [**Docker 集成**](30-interfaces/40-docker/README.md) — 容器化部署
- [**快速入门**](30-interfaces/50-examples/01-quickstart.md) — 5 分钟从零到 Hello World
- **系统调用 API**：[Agent](30-interfaces/60-syscalls/01-agent.md) / [Memory](30-interfaces/60-syscalls/02-memory.md) / [Session](30-interfaces/60-syscalls/03-session.md) / [Skill](30-interfaces/60-syscalls/04-skill.md) / [Task](30-interfaces/60-syscalls/05-task.md) / [Telemetry](30-interfaces/60-syscalls/06-telemetry.md)
- **多语言 SDK**：[Go](30-interfaces/70-toolkit/10-go/README.md) / [Python](30-interfaces/70-toolkit/20-python/README.md) / [Rust](30-interfaces/70-toolkit/30-rust/README.md) / [TypeScript](30-interfaces/70-toolkit/40-typescript/README.md)

### 5️⃣ 工程标准（50-engineering-standards，共享）

agentrt 与 agentrt-linux 共用的工程标准规范（符号链接至 AirymaxOS）：

- [**工程标准手册**](50-engineering-standards/00-engineering-standards-handbook.md) — 总览与导航
- [**编码风格**](50-engineering-standards/10-coding-style/) — C/C++/Rust/Go/Python/JS 编码规范
- [**跨项目代码共享**](50-engineering-standards/120-cross-project-code-sharing.md) — IRON-9 v2 [SC] 共享契约层
- [**合规检查**](50-engineering-standards/05-development-process.md) — 17 项检查规则（Part III 合规检查清单）

### 6️⃣ 应用开发（140-application-development）

Agent 开发者指南、SDK、CLI 工具与部署：

- [**快速开始**](140-application-development/01-getting-started.md) — 5 分钟从零到 Hello World
- [**构建指南**](140-application-development/02-build-guide.md) — 构建系统与编译选项
- [**部署指南**](140-application-development/03-deployment-guide.md) — 生产环境部署最佳实践
- [**配置指南**](140-application-development/04-configuration-guide.md) — 完整配置选项说明
- [**监控运维**](140-application-development/07-monitoring-guide.md) — Prometheus + Grafana 监控栈
- [**性能调优**](140-application-development/09-performance-tuning.md) — 内核参数与性能优化
- [**迁移指南**](140-application-development/10-migration-guide.md) — 版本升级与数据迁移
- [**备份恢复**](140-application-development/11-backup-recovery.md) — 数据备份与灾难恢复
- [**安全网关**](140-application-development/12-security-gateway.md) — 安全网关配置
- [**安全加固**](140-application-development/13-security-hardening.md) — 安全加固指南
- [**测试指南**](140-application-development/14-testing-guide.md) — 单元/集成/E2E 测试
- [**测试规范**](140-application-development/15-testing-standards.md) — 测试分层与覆盖率标准
- [**CI/CD 流水线**](140-application-development/16-ci-cd-pipelines.md) — 持续集成与部署
- [**Plugin SDK 教程**](140-application-development/17-plugin-sdk-tutorial.md) — Plugin SDK 完整开发指南
- [**协议集成**](140-application-development/18-protocol-integration.md) — 多协议适配与路由
- [**Prompt 工程**](140-application-development/19-prompt-engineering.md) — Prompt 模板/注入/调优
- [**Manager 开发**](140-application-development/20-manager-development.md) — Manager 模块开发
- [**最佳实践**](140-application-development/21-best-practices.md) — 开发最佳实践
- [**已知问题**](140-application-development/22-known-issues.md) — 已知问题与解决方案
- [**故障排查 FAQ**](140-application-development/23-troubleshooting-faq.md) — 高频问题排查
- [**创建 Agent**](140-application-development/24-create-agent.md) — Agent 开发完整流程
- [**创建 Skill**](140-application-development/25-create-skill.md) — Skill 开发完整流程
- [**CoreLoopThree DAG 集成**](140-application-development/26-coreloopthree-dag-integration.md) — DAG 工作流集成

---

## 🏗️ 目录结构

```
docs/AirymaxRT/
├── README.md                          本文件（英文版）
├── README_zh.md                       本文件（中文版）
├── TERMINOLOGY.md                     agentrt 术语表
├── 00-requirements/                   需求规格（中英双语）
├── 10-architecture/                   系统架构
│   ├── 20-engineering/                工程实践
│   ├── 30-kernel/                     内核子系统
│   ├── 40-philosophy/                 设计哲学
│   ├── 50-services/                   服务子系统
│   └── 60-diagrams/                   架构图（drawio）
├── 20-modules/                        模块设计
│   └── 10-contracts/                  契约规范
├── 30-interfaces/                     对外接口
│   ├── 05-runtime-interfaces/        L1-L3 运行时接口
│   ├── 06-ipc/                        IPC 标准
│   ├── 07-rpc-api/                   RPC API 标准
│   ├── 08-service-discovery/         服务发现
│   ├── 10-algorithms/                核心算法
│   ├── 20-core/                      核心 API
│   ├── 30-daemon/                    Daemon API
│   ├── 40-docker/                    Docker 集成
│   ├── 50-examples/                  示例
│   ├── 60-syscalls/                  系统调用 API
│   └── 70-toolkit/                   多语言 SDK
├── 50-engineering-standards/          工程标准（符号链接 → AirymaxOS）
└── 140-application-development/       应用开发
```

> **设计原则**: AirymaxRT 仅保留 agentrt 自身有实际内容的目录。不创建仅用于镜像 AirymaxOS 的空占位目录。agentrt 与 agentrt-linux 共用 `50-engineering-standards` 工程标准层。

---

## 📋 快速入口

| 角色 | 推荐阅读路径 | 预估时间 |
|------|-------------|---------|
| **开发者** | 架构原则 → 模块设计 → 接口设计 → SDK 开发 | 2 小时 |
| **架构师** | 架构原则 → 微核心架构 → CoreLoopThree → MemoryRovol | 3 小时 |
| **Agent 作者** | 应用开发 → SDK 开发 → Agent 创建 | 1.5 小时 |
| **运维工程师** | 部署指南 → 监控运维 → 故障排查 | 1.5 小时 |

---

## 📊 文档质量标准

Airymax 文档遵循**完美主义原则 (A-4)**：

- **完整性**：每个公共 API 都有文档
- **准确性**：示例代码可运行，配置参数经过验证
- **易读性**：使用清晰的语言，避免过度技术化
- **可操作性**：每个指南都提供分步操作说明

---

## 🔗 相关文档

- [AirymaxOS 文档中心](../AirymaxOS/README.md) — agentrt-linux 智能体操作系统文档
- [Airymax 开放标准](../OpenStandards/README.md) — 7 主题开放标准体系
- [工程标准手册](50-engineering-standards/00-engineering-standards-handbook.md) — 共享工程标准
- [统一术语表](TERMINOLOGY.md) — agentrt 术语表

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
