# agentrt-liunx（AirymaxOS）工程基线

> **文档定位**: agentrt-liunx（AirymaxOS）工程基线（Engineering Baseline）的完整定义与落地规范
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **父文档**: [架构设计](README.md)

---

## 1. 工程基线概述

### 1.1 什么是工程基线

**工程基线**（Engineering Baseline）是 agentrt-liunx 在工程实践层面统一遵循的最小完备规范集合。它锁定内核版本、架构支持、包管理、安全模型、系统服务、AI 原生能力、治理模式与版本节奏等关键工程维度，为 8 子仓的协同开发、跨团队协作与跨版本演进提供不可动摇的参考基准。

工程基线存在的核心价值：

| 价值 | 说明 |
|------|------|
| **统一表述** | 所有文档、代码注释、对外材料统一使用基线术语，杜绝表述漂移 |
| **统一节奏** | 内核、用户态、测试、文档按同一版本节奏推进，避免错位 |
| **统一边界** | 明确哪些特性属于 1.0.1 基线、哪些属于下一代基线，防止误引（见 IRON-10 / BAN-361） |
| **统一兼容性** | 对外声明一致的兼容性边界，便于上层应用与硬件生态对接 |

### 1.2 工程基线的四条铁律

| 铁律 | 内容 | 关联工程纪律 |
|------|------|--------------|
| **基线锁定** | 内核基线锁定 Linux 6.6，禁止引用未来内核专属特性作为 6.6 原生能力 | IRON-10 / BAN-361 |
| **表述纯净** | 所有文档使用 agentrt-liunx 自身工程语言表述，不引入任何外部参考来源措辞 | ACC-OS04 |
| **同源约束** | 与 agentrt 同源且部分代码共享（IRON-9 v2），基线落地遵循 IRON-9 v2 | IRON-9 v2 |
| **演进可控** | 基线演进必须通过 ADR 评审，禁止未经评审的基线漂移 | ADR 流程 |

### 1.3 工程基线与三大支柱的关系

agentrt-liunx 架构设计建立在三大支柱之上：微内核设计思想、agentrt-liunx 工程基线、Airymax 同源性。工程基线作为其中之一，提供"如何对接生态与工程实践"的标准：

- **微内核设计思想** 提供"如何设计内核"的方法论
- **agentrt-liunx 工程基线** 提供"如何对接生态、如何表述工程"的基线（本文档）
- **Airymax 同源性** 提供"如何与 agentrt 协同"的语义约束

---

## 2. 内核基线

### 2.1 内核版本锁定

agentrt-liunx 1.0.1 锁定 **Linux 6.6 内核基线** 作为系统基础。内核版本选择直接影响硬件兼容性、生态兼容性、新特性支持与长期支持周期。

| 基线版本 | Linux 内核 | agentrt-liunx 定位 |
|----------|------------|-----------------|
| 当前基线 | Linux 6.6 内核基线 | 1.0.1 基础内核 |
| 增强基线 | Linux 6.6 内核基线（SP3 增强） | AI 原生能力增强 |
| 下一代基线 | Linux 6.15+ 内核 | 具身智能 / 超节点沙箱演进 |

### 2.2 基线内核特性映射

Linux 6.6 内核基线提供以下原生特性，全部纳入 agentrt-liunx 工程基线：

| 特性 | 内核版本 | 来源 | agentrt-liunx 落地 |
|------|----------|------|----------------|
| EEVDF 调度器 | Linux 6.6 原生 | 主线 | airymaxos-kernel 调度核心 |
| Rust 实验性支持 | Linux 6.6 | 主线 | airymaxos-kernel 安全驱动 |
| MGLRU（多代 LRU） | Linux 6.6 | 主线 | airymaxos-memory 冷热分层 |
| XFS 在线 fsck | Linux 6.6 | 主线 | airymaxos-services 文件系统 |
| eBPF kfunc + dynamic pointer | Linux 6.6 原生 | 主线 | airymaxos-security 可观测性 |
| io_uring 异步 I/O 改进 | Linux 6.6 | 主线 | airymaxos-kernel + services IPC |
| BPF + io_uring 集成 | Linux 6.6 | 主线 | airymaxos-security 策略可插拔 |
| sched_ext（SCHED_AGENT） | agentrt-liunx 内核增强 | 主线 6.12+ | airymaxos-kernel 用户态调度 |

### 2.3 内核增强策略

对于主线 6.12+ 才稳定但 agentrt-liunx 1.0.1 必需的特性（如 sched_ext），采用 **agentrt-liunx 内核增强** 策略引入，而非等待主线 6.6 原生支持：

```
Linux 6.6 原生特性（EEVDF / Rust / MGLRU / XFS fsck / eBPF kfunc / io_uring）
    +
agentrt-liunx 内核增强（sched_ext 等，主线 6.12+ 特性向前移植）
    =
agentrt-liunx 1.0.1 完整内核基线
```

**增强边界**：仅增强 agentrt-liunx 微内核化改造与 AI 原生能力所必需的特性，不引入与基线目标无关的主线特性。

### 2.4 内核基线的工程纪律

| 纪律 | 内容 |
|------|------|
| 禁止误引 | 禁止引用 6.7+ 主线特性作为 6.6 原生能力（IRON-10 / BAN-361 / ACC-OS04） |
| 统一表述 | 所有文档统一使用"Linux 6.6 内核基线"表述 |
| 增强透明 | 所有 agentrt-liunx 内核增强必须标注"主线 X.Y+ 特性增强"，禁止伪称为原生 |
| 演进评审 | 基线升级（如 6.6 → 6.15+）必须通过 ADR 评审 |

---

## 3. 架构支持基线

### 3.1 多架构支持

agentrt-liunx 工程基线支持以下 CPU 架构：

| 架构 | 厂商 / 实现 | 优先级 |
|------|-------------|--------|
| x86 | Intel / AMD | 1.0.1 优先支持 |
| ARM | 鲲鹏 / 飞腾 / Ampere | 1.0.1 优先支持 |
| RISC-V | 通用 RISC-V | 后续扩展 |
| LoongArch | 龙芯 | 后续扩展 |

### 3.2 异构内存支持

| 内存类型 | 内核特性 | agentrt-liunx 落地 |
|----------|----------|----------------|
| CXL 内存池化 | Linux 6.6 CXL 支持 | airymaxos-memory L1 跨节点池化 |
| PMEM 持久化内存 | Linux PMEM 支持 | airymaxos-memory 持久化层 |
| MGLRU 多代 LRU | Linux 6.6 MGLRU | airymaxos-memory 冷热数据分代回收 |

---

## 4. 子仓治理组基线

### 4.1 治理组对应表

agentrt-liunx 采用 **子仓治理组**（Sub-repository Governance Group）模式组织工程协作。每个子仓对应一个治理组，负责技术决策与代码审查：

| agentrt-liunx 治理组 | 对应子仓 | 核心职责 |
|------------------|----------|----------|
| Kernel 治理组 | airymaxos-kernel | 内核 + 微内核化改造 |
| Base Systems 治理组 | airymaxos-services + airymaxos-system | 用户态服务 + 系统工具 |
| Security 治理组 | airymaxos-security | 安全子系统 + 国密 |
| Memory 治理组 | airymaxos-memory | 记忆持久化 + 异构内存 |
| Cognition 治理组 | airymaxos-cognition | 认知运行时 + 超节点沙箱 |
| Cloud Native 治理组 | airymaxos-cloudnative | K8s + containerd + 超节点 OS |
| QA 治理组 | airymaxos-tests | 集成测试框架 + 形式化验证 |
| Embedded 治理组 | airymaxos-cognition（部分） | 具身智能支持 |

### 4.2 治理模式

agentrt-liunx 子仓治理模式遵循以下原则：

1. **每个子仓对应一个治理组**：治理组负责技术决策和代码审查
2. **社区开放**：欢迎外部贡献，治理组负责审查与合入
3. **能力域清晰**：8 子仓严格按微内核能力域划分，无功能重叠
4. **层次纪律**：治理组之间的依赖遵循 S-2 层次分解原则（kernel → services/security/memory/cognition → cloudnative → system → tests）
5. **独立演进**：每个子仓可独立测试和演进，通过接口契约协同

---

## 5. AI 原生能力基线

agentrt-liunx 作为 Agentic OS，AI 原生能力是工程基线的核心组成。以下能力构成 agentrt-liunx 1.0.1 ~ 2.0+ 的 AI 原生演进路线。

### 5.1 认知中枢系统

**agentrt-liunx 认知中枢**（Cognitive Hub）是 airymaxos-cognition 子仓的核心能力，对应 agentrt CoreLoopThree 同源语义：

| 维度 | 规格 |
|------|------|
| 实现语言 | Rust |
| 内存占用 | 仅 25M |
| 启动时间 | 0.1 秒极速启动 |
| Token 成本管控 | 可精细管控 Token 成本 |
| 回滚能力 | 支持毫秒级回滚 |
| 内核集成 | CoreLoopThree kthread（内核线程） |

**agentrt-liunx 落地**：airymaxos-cognition 实现 agentrt-liunx 认知中枢，与 agentrt 的 CoreLoopThree 同源，作为内核态认知运行时。

### 5.2 超节点 OS

**agentrt-liunx 超节点 OS** 是 airymaxos-cloudnative 子仓的核心能力，支持大规模 Agent 容器化部署：

| 维度 | 规格 |
|------|------|
| AI 故障定位 | AI 实现开箱即用和故障定位 |
| RPC 时延 | RPC 时延下降 20% |
| 异构互联 | 通过超节点异构统一互联底座 UMDK |
| 容器通信 | 实现大规模容器低时延通信 |

**agentrt-liunx 落地**：airymaxos-cloudnative 实现超节点 OS，支持大规模 Agent 容器化部署与跨节点低时延协同。

### 5.3 超节点沙箱

**agentrt-liunx 超节点沙箱** 是 airymaxos-cognition 子仓的 Agent 快速冷启动能力：

| 维度 | 规格 |
|------|------|
| 镜像快照 | 软硬协同优化镜像快照 |
| 懒加载 | 远端懒加载、分层按需加载 |
| 冷启动 | 大幅缩短冷启动耗时 |
| 基线版本 | 下一代内核基线（Linux 6.15+ 内核） |

**agentrt-liunx 落地**：airymaxos-cognition 实现超节点沙箱，支持 Agent 快速冷启动，与 Wasm 3.0 沙箱运行时集成（ADR-008）。

### 5.4 Token 能效框架

**agentrt-liunx Token 能效框架**（Token Energy Efficiency Framework）实现 Token 消耗的边际递减：

| 层级 | 组件 | 职责 |
|------|------|------|
| 网关层 | KVC-Gateway | 精准调度 Token 请求 |
| 缓存层 | LMCache | 多级内存池化复用 |
| 响应层 | Bifrost | 弹性响应与负载自适应 |

**agentrt-liunx 落地**：airymaxos-cognition 实现 Token 能效优化，让 Token 消耗走向边际递减，支撑 Agentic OS 的成本可控性。

### 5.5 具身智能 Claw

**agentrt-liunx 具身智能** 支持 Agent 与物理世界的交互：

| 维度 | 规格 |
|------|------|
| 开箱即用 | 行业首个开箱即用的具身智能解决方案 |
| 系统融合 | 融合 AI 与 ROS 系统 |
| 场景切换 | 支持真实/仿真场景无缝切换 |
| 基线版本 | 下一代内核基线（Linux 6.15+ 内核） |

**agentrt-liunx 落地**：airymaxos-cognition 实现具身智能支持，由 Embedded 治理组协同推进。

### 5.6 Agentic AI 架构演进

agentrt-liunx 对标 Agentic OS 三阶段演进路线：

| 阶段 | 名称 | 特征 | agentrt-liunx 定位 |
|------|------|------|-----------------|
| 阶段 1 | 资源抽象 | 传统 OS | 不在此阶段 |
| 阶段 2 | 异构融合 | AI 原生 OS | 1.0.1 部分覆盖 |
| 阶段 3 | 意图协同 | Agentic OS | agentrt-liunx 直接对标此阶段 |

**agentrt-liunx 落地**：agentrt-liunx 直接对标第三阶段"意图协同"，作为 Agentic OS，认知中枢 + 超节点 OS + Token 能效框架共同构成意图协同的工程基座。

---

## 6. 技术规格基线

### 6.1 包管理基线

| 维度 | agentrt-liunx 基线 |
|------|----------------|
| 包格式 | RPM |
| 包管理器 | dnf |
| 软件源 | repo.airymaxos.org |
| 构建工具 | agentrt-liunx 构建工具链 |

### 6.2 安全基线

| 维度 | agentrt-liunx 基线 |
|------|----------------|
| 强制访问控制 | SELinux + capability（seL4 风格） |
| 国密算法 | SM2 / SM3 / SM4 |
| 漏洞响应 | agentrt-liunx-SA |
| 审计 | auditd + eBPF 可观测性 |
| 机密计算 | TEE / SGX |
| 审计哈希链 | SHA-256 哈希链不可篡改日志 |
| 运行时保护 | Seccomp + CFI（控制流完整性） |

### 6.3 系统服务基线

| 维度 | agentrt-liunx 基线 |
|------|----------------|
| 初始化系统 | systemd |
| 服务管理 | agentctl（兼容 systemctl） |
| 日志 | journald + Airymax 日志系统（结构化 + trace_id） |
| 网络 | NetworkManager + eBPF 网络 |
| 命令行 | agentctl（兼容 kubectl 语法） |

---

## 7. 工程规范基线

### 7.1 编码规范

agentrt-liunx 工程基线要求所有代码遵循 `docs/Capital_Specifications/coding_standard/` 中的 15 个规范文件，覆盖 C / Rust / Python / Go / TypeScript 等语言。

### 7.2 接口契约规范

所有跨模块交互必须通过明确定义的接口进行，接口契约通过 `30-interfaces/` 文档定义：

| 契约载体 | 位置 | 内容 |
|----------|------|------|
| syscall 接口 | `30-interfaces/01-syscalls.md` | 系统调用编号、签名、参数方向、所有权 |
| IPC 协议 | `30-interfaces/02-ipc-protocol.md` | 128B 消息头 + 5 种 payload 协议 |
| SDK API | `30-interfaces/03-sdk-api.md` | Python/Rust/Go/TS 四语言 SDK |
| 编码规范 | `30-interfaces/` 全部 5 文档 | syscall + IPC + SDK + 编码规范 |

### 7.3 工程铁律

agentrt-liunx 工程基线受以下工程铁律约束：

| 铁律 | 内容 |
|------|------|
| IRON-9 | agentrt 和 agentrt-liunx 同源且部分代码共享（IRON-9 v2：共享契约层代码完全共享 + 语义同源层 API 同源实现独立 + 完全独立层各自独立） |
| IRON-10 | 内核基线锁定，禁止引用未来内核专属特性作为 6.6 原生能力 |
| BAN-361 | 禁止未来内核特性误引 |
| ACC-OS04 | Grep 扫描未来内核误引残留（每次发布执行） |

---

## 8. 版本管理基线

### 8.1 版本节奏

agentrt-liunx 采用两类版本节奏：

| 版本类型 | 发布周期 | 支持周期 |
|----------|----------|----------|
| LTS 版本 | 2 年发布周期 | 4 年支持 |
| 创新版本 | 6 个月发布周期 | 滚动更新 |
| Update 版本 | 每周发布 | 持续 |

### 8.2 agentrt-liunx 版本规划

| 版本 | 日期 | 定位 |
|------|------|------|
| 0.1.1 | 2026 | 仓库占位 |
| 1.0.1 | 2027 | 首个开发版本（Linux 6.6 内核基线） |
| 后续 | - | 按 LTS + 创新版本模式演进 |

### 8.3 内核基线演进

| agentrt-liunx 版本 | 内核基线 | 关键能力 |
|----------------|----------|----------|
| 1.0.1 | Linux 6.6 内核基线 | sched_ext + eBPF kfunc + io_uring + 微内核化改造阶段 1 |
| 1.0.2+ | Linux 6.6 内核基线 | 微内核化改造阶段 2（VFS / 网络栈 / 驱动用户态化） |
| 2.0+ | Linux 6.6 内核基线 / 下一代内核基线 | 完整微内核化 + 超节点沙箱 + 具身智能 |

---

## 9. 文档体系基线

### 9.1 文档类型

| 文档类型 | agentrt-liunx 基线 |
|----------|----------------|
| 技术白皮书 | agentrt-liunx 技术白皮书 |
| 安装指南 | agentrt-liunx 安装指南 |
| 管理员指南 | agentrt-liunx 管理员指南 |
| 开发者指南 | agentrt-liunx 开发者指南 |
| API 参考 | docs.airymaxos.org |
| 安全公告 | agentrt-liunx SA |

### 9.2 文档层级

agentrt-liunx 设计文档采用 19 模块三层体系，与根 [README.md](../README.md) 导航保持一致：

| 层级 | 目录范围 | 模块数 | 0.1.1 状态 | 内容 |
|------|---------|--------|-----------|------|
| **核心设计层** | `00-requirements/` ~ `40-dataflows/` | 5 模块 | 28 文档 ✅ 完成 | 需求 + 架构 + 模块 + 接口 + 数据流 |
| **工程标准与实施层** | `50-engineering-standards/` ~ `130-roadmap/` | 9 模块 | 36 文档 ✅ 完成 | 工程标准 + 驱动 + 构建 + 测试 + 可观测 + 运维 + 安全 + 流程 + 路线图 |
| **延伸层** | `140-application-development/` ~ `190-distribution/` | 5 模块 | 6 README 占位 | 应用开发 + 云原生 + 兼容性 + 性能 + 国际化 + 发行版 |

**0.1.1 合计 ~64 文档**（核心 28 + 实施层 36 + 延伸层 6 README 占位 + 根 README 等），**1.0.1 扩展到 ~140 文档**。

详细模块导航见根 [README.md](../README.md) §4 文档体系结构。

### 9.3 文档即代码

文档与代码同步更新（E-7 文档即代码原则）：

1. 文档作为代码的一部分进行版本控制
2. 文档变更与代码变更同 PR 提交
3. 过时文档比没有文档更糟糕，必须及时更新
4. 所有公共 API 必须有 Doxygen 契约注释

---

## 10. 兼容性声明

agentrt-liunx 工程基线声明以下兼容性：

- ✅ 兼容企业级 Linux 生态的 RPM 包格式
- ✅ 兼容企业级 Linux 生态的 dnf 包管理器
- ✅ 兼容企业级 Linux 生态的 systemd 服务管理
- ✅ 兼容企业级 Linux 生态的安全模块（SELinux）
- ✅ 兼容企业级 Linux 生态的国密算法（SM2/SM3/SM4）
- ✅ 兼容企业级 Linux 生态的架构支持（x86 / ARM / RISC-V / LoongArch）
- ✅ 基于 Linux 6.6 内核基线，与企业级 Linux 内核同源
- ✅ 兼容 agentrt-liunx 超节点 OS 设计（大规模容器低时延通信）

---

## 11. 基线验证

### 11.1 基线符合性检查

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 内核基线表述 | ACC-OS04（Grep 扫描禁词与误引残留） | 每次发布 |
| 接口契约完整性 | scripts/check_api_complexity.sh | 每次 PR |
| 层次依赖检查 | scripts/check_layer_deps.py | 每次 PR |
| 文档同步检查 | CI 流水线文档校验 | 每次 PR |
| 测试覆盖率 | CI 流水线 lcov | 每次 PR |

### 11.2 基线违反处理

当发现基线违反时，按以下流程处理：

1. **记录违规**：在代码审查中标注违反的基线项
2. **评估影响**：分析违规对系统的影响程度
3. **制定方案**：提出符合基线的替代方案
4. **实施整改**：修改代码以符合基线
5. **验证合规**：重新检查基线符合性
6. **ADR 记录**：严重违规需创建 ADR 记录决策

---

## 12. 相关文档

- [架构设计 README](README.md)：架构设计层总览
- [系统架构](01-system-architecture.md)：agentrt-liunx 系统架构总览
- [五维正交原则](02-five-dimensional-principles.md)：五维正交 24 原则落地映射
- [微内核策略](03-microkernel-strategy.md)：微内核化改造策略
- [架构决策记录](05-adrs.md)：10 个核心 ADR
- [架构原则](../../ARCHITECTURAL_PRINCIPLES.md)：五维正交 24 原则的完整定义

---

## 13. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-06 | 初始版本（含工程基线完整定义） |
| 1.0.1 | 2027-XX-XX | 首个开发版本（与代码实现同步验证） |

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
