Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# AirymaxAgentRT 文档中心
> **文档定位**：AirymaxAgentRT 文档中心\
> **最后更新**：2026-07-11

---

## 📚 文档导航

AirymaxAgentRT（极境智能体运行底座平台工程）是用户态 AI Agent 运行时，定位为**微核心**（micro-core）。本文档中心覆盖 agentrt 的需求规格、系统架构、模块设计、对外接口和应用开发。

> **工程标准**: agentrt 与 agentrt-linux 共用一套工程标准，物理宿主在 [`docs/AirymaxOS/50-engineering-standards/`](../AirymaxOS/50-engineering-standards/)，通过符号链接 `50-engineering-standards` 引用。

---

## 📖 文档目录

### 1️⃣ 需求规格（00-requirements）

agentrt 的功能需求、非功能需求与业务约束：

- [**需求规格总览**](00-requirements/) — agentrt 功能/非功能/业务需求

### 2️⃣ 系统架构（10-architecture）

agentrt 的设计哲学、架构原则与核心组件设计：

- [**架构原则**](10-architecture/) — 五维正交 24 原则、微核心架构设计
- [**CoreLoopThree**](10-architecture/) — 认知→执行→记忆三阶段循环
- [**MemoryRovol**](10-architecture/) — L1→L2→L3→L4 四层记忆卷载
- [**IPC 通信**](10-architecture/) — 进程间通信架构
- [**Syscall 架构**](10-architecture/) — 用户态/内核态统一接口

### 3️⃣ 模块设计（20-modules）

agentrt 各子模块的设计文档（atoms / commons / cupolas / daemons / gateway / heapstore / protocols）：

- [**模块设计总览**](20-modules/) — 7 个子仓模块的"是什么"概览

### 4️⃣ 对外接口（30-interfaces）

agentrt 的 UAPI、ABI、SDK 公共接口设计：

- [**接口设计总览**](30-interfaces/) — 系统调用、IPC 协议、RPC API、服务发现
- [**IPC 协议**](30-interfaces/) — 128B 消息头、payload 类型、请求-响应/发布-订阅
- [**RPC API**](30-interfaces/07-rpc-api/) — JSON-RPC 2.0 接口规范
- [**服务发现**](30-interfaces/) — 服务注册与发现机制

### 5️⃣ 工程标准（50-engineering-standards，共享）

agentrt 与 agentrt-linux 共用的工程标准规范（符号链接至 AirymaxOS）：

- [**工程标准手册**](50-engineering-standards/00-engineering-standards-handbook.md) — 总览与导航
- [**编码风格**](50-engineering-standards/10-coding-style/) — C/C++/Rust/Go/Python/JS 编码规范
- [**跨项目代码共享**](50-engineering-standards/120-cross-project-code-sharing.md) — IRON-9 v2 [SC] 共享契约层
- [**合规检查**](50-engineering-standards/05-development-process.md) — 17 项检查规则（Part III 合规检查清单）

### 6️⃣ 应用开发（140-application-development）

Agent 开发者指南、SDK、CLI 工具与部署：

- [**SDK 开发**](140-application-development/) — Python/Go/Rust/TypeScript 多语言 SDK
- [**Agent 开发**](140-application-development/) — Agent 创建与开发工作流
- [**CLI 工具**](140-application-development/) — 统一 CLI 命令参考
- [**部署指南**](140-application-development/) — 生产部署最佳实践

---

## 🏗️ 目录结构

```
docs/AirymaxRT/
├── README.md                          本文件
├── TERMINOLOGY.md                     agentrt 术语表
├── 00-requirements/                   需求规格
├── 10-architecture/                   系统架构
├── 20-modules/                        模块设计
├── 30-interfaces/                     对外接口
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

---

## 🔗 相关文档

- [AirymaxOS 文档中心](../AirymaxOS/README.md) — agentrt-linux 智能体操作系统文档
- [Airymax 开放标准](../OpenStandards/README.md) — 7 主题开放标准体系
- [工程标准手册](50-engineering-standards/00-engineering-standards-handbook.md) — 共享工程标准

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
