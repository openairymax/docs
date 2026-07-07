# Airymax Documentation Center — 文档中心

> **路径**: `docs/` | **版本**: 0.1.1 | **最后更新**: 2026-07-07

## 概述

`docs/` 是 Airymax 平台的统一文档中心，包含两个项目的完整文档体系：

- **[AirymaxAgentRT](AirymaxAgentRT/README.md)** — 极境智能体运行底座平台工程（agentrt）的设计文档、API 参考、开发指南和编码规范
- **[AirymaxAgentOS](AirymaxAgentOS/README.md)** — 极境智能体操作系统（agentrt-liunx（AirymaxOS））的架构设计、子仓设计草案和工程标准

## 目录结构

```
docs/
├── README.md                              # 本文件（文档中心入口）
├── LICENSE                                # 许可证
├── NOTICE                                 # 版权声明
│
├── AirymaxAgentRT/                        # 极境智能体运行底座平台工程文档
│   ├── README.md                          # AgentRT 文档总览
│   ├── ARCHITECTURAL_PRINCIPLES.md        # 24 条 S/K/C/E/A 五维正交设计原则
│   ├── Basic_Theories/                    # 基础理论（MCIS/认知层/记忆层/设计原则）
│   ├── Capital_Architecture/              # 核心架构（MicroCoreRT/CoreLoopThree/MemoryRovol/Syscall）
│   ├── Capital_API/                       # API 参考（syscall/daemon/toolkit/algorithms）
│   ├── Capital_Guides/                    # 开发指南（快速入门/构建/部署/创建Agent/排错）
│   ├── Capital_Specifications/            # 规范标准（编码规范/契约/ARE标准/集成标准）
│   ├── Paper_Academic/                    # 学术论文
│   └── Source_Other/                      # 其他资源
│
└── AirymaxAgentOS/                        # 极境智能体操作系统文档
    ├── README.md                          # AirymaxOS 文档总览（8 子仓 + 19 模块）
    ├── 00-requirements/                   # 需求分析（业务需求/功能需求/非功能需求）
    ├── 10-architecture/                   # 架构设计（系统架构/五维原则/微内核策略/工程基线/ADR）
    ├── 20-modules/                        # 子仓设计（kernel/services/security/memory/cognition/cloudnative/system/tests）
    ├── 30-interfaces/                     # 接口设计（syscalls/IPC协议/SDK API/编码规范）
    ├── 40-dataflows/                      # 数据流设计（认知/IPC/内存/调度）
    ├── 50-engineering-standards/          # 工程标准（编码规范/格式/风格/哲学/开发流程/工具链/治理/合规检查）
    ├── 70-build-system/                   # 构建系统（KBuild/KConfig）
    ├── 90-observability/                  # 可观测性（ftrace/eBPF probes）
    ├── 100-operations/                    # 运维体系（部署/配置）
    ├── 110-security/                      # 安全体系（LSM/Landlock）
    ├── 130-roadmap/                       # 路线图（策略/里程碑/资源估算/依赖图/风险/验收）
    ├── 140-application-development/       # 应用开发
    ├── 150-cloudnative/                   # 云原生
    ├── 160-compatibility/                 # 兼容性
    ├── 170-performance/                   # 性能工程
    ├── 180-i18n/                          # 国际化
    ├── 190-distribution/                  # 发行版管理
    └── Capital_Specifications/            # 规范（术语/编码规范/ERP）
```

## 两个项目的关系

| 维度 | AirymaxAgentRT | AirymaxAgentOS |
|------|---------------|----------------|
| **定位** | AI Agent 运行时平台工程 | 智能体操作系统发行版 |
| **英文名** | AirymaxAgentRT | AirymaxOS |
| **代码仓库** | agentrt/（管理仓 + 7 叶子仓） | agentrt-linux/（管理仓 + 8 叶子仓） |
| **内核基线** | 跨平台用户态（Linux/macOS/Windows） | Linux 6.6 内核 |
| **关系** | 独立运行时 | 与 agentrt 同源，基于 Linux 内核 |

两个项目共享 Airymax 设计理念（MicroCoreRT / AgentsIPC / Cupolas / MemoryRovol / CoreLoopThree），通过 IRON-9 v2 同源且部分代码共享原则保持互操作。

## 快速导航

| 角色 | 推荐阅读路径 |
|------|-------------|
| **新手入门** | AirymaxAgentRT → Capital_Guides/getting_started.md → configuration_guide.md |
| **开发者** | AirymaxAgentRT → Capital_API/ → Capital_Specifications/coding_standard/ |
| **架构师** | AirymaxAgentRT → Basic_Theories/ → Capital_Architecture/ → ARCHITECTURAL_PRINCIPLES.md |
| **运维工程师** | AirymaxAgentRT → Capital_Guides/deployment_guide.md → monitoring_guide.md |
| **OS 开发者** | AirymaxAgentOS → 10-architecture/ → 20-modules/ → 50-engineering-standards/ |

## 许可证

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

双许可证：**AGPL-3.0-or-later OR Apache-2.0**（SPDX: `AGPL-3.0-or-later OR Apache-2.0`）。详见 [LICENSE](LICENSE)。

---

> **文档结束** | 0.1.1（Airymax 统一文档中心入口）
