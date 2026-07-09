# Airymax Documentation Center — 文档中心

> **路径**: `docs/` | **版本**: 0.1.1 | **最后更新**: 2026-07-09

## 概述

`docs/` 是 Airymax 平台的统一文档中心，包含四个项目的完整文档体系：

- **[AirymaxRT](AirymaxRT/README.md)** — 极境智能体运行底座平台工程（agentrt）的设计文档、API 参考、开发指南和编码规范
- **[AirymaxOS](AirymaxOS/README.md)** — 极境智能体操作系统（agentrt-linux）的架构设计、子仓设计草案和工程标准
- **[OpenStandards](OpenStandards/README.md)** — Airymax 开放标准体系（运行时标准/IPC协议/安全模型/MemoryRovol/认知循环/调度标准）
- **[Publications](Publications/README.md)** — Airymax 研究成果发布中心（期刊论文/会议论文/技术报告/预印本）

## 目录结构

```
docs/
├── README.md                              # 本文件（文档中心入口）
├── LICENSE                                # 许可证
├── NOTICE                                 # 版权声明
│
├── AirymaxRT/                             # 极境智能体运行底座平台工程文档
│   ├── README.md                          # AirymaxRT 文档总览
│   ├── README_zh.md                       # AirymaxRT 中文文档总览
│   ├── 10-terminology.md                     # 术语表
│   ├── 00-architectural-principles.md        # 24 条 S/K/C/E/A 五维正交设计原则
│   ├── 00-basic-theories/                    # 基础理论（MCIS/认知层/记忆层/设计原则）
│   ├── 10-architecture/              # 核心架构（MicroCoreRT/CoreLoopThree/MemoryRovol/Syscall）
│   ├── 30-api/                       # API 参考（syscall/daemon/toolkit/algorithms）
│   ├── 60-guides/                    # 开发指南（快速入门/构建/部署/创建Agent/排错）
│   ├── 50-specifications/            # 规范标准（编码规范/契约/ARE标准/集成标准）
│   └── 90-references/                      # 其他资源
│
├── AirymaxOS/                             # 极境智能体操作系统文档
│   ├── README.md                          # AirymaxOS 文档总览（8 子仓 + 19 模块）
│   ├── 10-terminology.md                     # 术语表
│   ├── 00-requirements/                   # 需求分析（业务需求/功能需求/非功能需求）
│   ├── 10-architecture/                   # 架构设计（系统架构/五维原则/微内核策略/工程基线/ADR）
│   ├── 20-modules/                        # 子仓设计（kernel/services/security/memory/cognition/cloudnative/system/tests）
│   ├── 30-interfaces/                     # 接口设计（syscalls/IPC协议/SDK API/编码规范）
│   ├── 40-dataflows/                      # 数据流设计（认知/IPC/内存/调度）
│   ├── 50-engineering-standards/          # 工程标准（编码规范/格式/风格/哲学/开发流程/工具链/治理/合规检查）
│   ├── 60-driver-model/                   # 驱动模型（设备模型/平台驱动）
│   ├── 70-build-system/                   # 构建系统（KBuild/KConfig）
│   ├── 80-testing/                        # 测试体系（KUnit/kselftest）
│   ├── 90-observability/                  # 可观测性（ftrace/eBPF probes）
│   ├── 100-operations/                    # 运维体系（部署/配置）
│   ├── 110-security/                      # 安全体系（LSM/Landlock）
│   ├── 120-development-process/           # 开发流程（补丁生命周期/维护者层级）
│   ├── 130-roadmap/                       # 路线图（策略/里程碑/资源估算/依赖图/风险/验收）
│   ├── 140-application-development/       # 应用开发
│   ├── 150-cloudnative/                   # 云原生
│   ├── 160-compatibility/                 # 兼容性
│   ├── 170-performance/                   # 性能工程
│   ├── 180-i18n/                          # 国际化
│   └── 190-distribution/                  # 发行版管理
│
├── OpenStandards/                         # Airymax 开放标准体系
│   ├── README.md                          # 开放标准总览
│   ├── 01-airymax-agent-runtime-standard.md    # 智能体运行时标准
│   ├── 02-airymax-ipc-protocol-standard.md     # IPC 协议标准
│   ├── 03-airymax-security-model-standard.md   # 安全模型标准
│   ├── 04-airymax-memory-rovol-standard.md    # MemoryRovol 标准
│   ├── 05-airymax-cognition-loop-standard.md  # 认知循环标准
│   └── 06-airymax-scheduling-standard.md      # 调度标准
│
└── Publications/                          # Airymax 研究成果发布中心
    ├── README.md                          # 研究成果发布中心入口
    ├── Journals/                          # 期刊论文
    ├── Conferences/                       # 会议论文
    ├── Technical_Reports/                 # 技术报告
    └── Preprints/                         # 预印本（arXiv）
```

## 四个项目的关系

| 维度 | AirymaxRT | AirymaxOS | OpenStandards | Publications |
|------|----------|-----------|---------------|--------------|
| **定位** | AI Agent 运行时平台工程 | 智能体操作系统发行版 | 开放标准体系 | 研究成果发布中心 |
| **英文名** | AirymaxRT | AirymaxOS | OpenStandards | Publications |
| **代码仓库** | agentrt/（管理仓 + 7 叶子仓） | agentrt-linux/（管理仓 + 8 叶子仓） | docs/OpenStandards/ | docs/Publications/ |
| **内核基线** | 跨平台用户态（Linux/macOS/Windows） | Linux 6.6 内核 | - | - |
| **关系** | 独立运行时 | 与 agentrt 同源，基于 Linux 内核 | AirymaxRT 与 AirymaxOS 共享的标准契约 | AirymaxRT 与 AirymaxOS 的研究产出 |

四个项目共享 Airymax 设计理念（MicroCoreRT / AgentsIPC / Cupolas / MemoryRovol / CoreLoopThree），通过 IRON-9 v2 同源且部分代码共享原则保持互操作。  
OpenStandards 定义了 AirymaxRT 与 AirymaxOS 之间的共享契约层，Publications 集中管理两个项目的学术研究成果。  

## 快速导航

| 角色 | 推荐阅读路径 |
|------|-------------|
| **新手入门** | AirymaxRT → 60-guides/01-getting-started.md → configuration_guide.md |
| **开发者** | AirymaxRT → 30-api/ → 50-specifications/30-coding-standard/ |
| **架构师** | AirymaxRT → 00-basic-theories/ → 10-architecture/ → 00-architectural-principles.md |
| **运维工程师** | AirymaxRT → 60-guides/03-deployment-guide.md → monitoring_guide.md |
| **OS 开发者** | AirymaxOS → 10-architecture/ → 20-modules/ → 50-engineering-standards/ |
| **标准制定者** | OpenStandards/ → 01-airymax-agent-runtime-standard.md |
| **研究人员** | Publications/ → Journals/ / Conferences/ / Preprints/ |

## 许可证

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

双许可证：**AGPL-3.0-or-later OR Apache-2.0**（SPDX: `AGPL-3.0-or-later OR Apache-2.0`）。详见 [LICENSE](LICENSE)。

---

> **文档结束** | 0.1.1（Airymax 统一文档中心入口）
