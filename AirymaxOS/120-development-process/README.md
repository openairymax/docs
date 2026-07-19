Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）开发流程设计
> **文档定位**：agentrt-linux（AirymaxOS）开发流程工程体系主索引（代码审查 + 贡献流程 + 版本管理 + 发布流程 + SSoT v2 单一权威源模型 + CI 强制校验）\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[AirymaxOS 总览](../README.md)\
> **同源映射**：agentrt 开发流程 + Linux 6.6 内核开发流程（development-process.rst 8 章）\
> **理论根基**：Linux 内核开发流程 + Airymax S-4 涌现性管理 + C-2 增量演化 + SSoT v2 单一权威源

---

## 1. 模块概述

agentrt-linux 开发流程是代码从设计到主线再到稳定发布的全生命周期规范。它继承 Linux 内核 30+ 年沉淀的开发流程哲学（补丁生命周期 + 维护者层级 + 审查优先 + 稳定版维护 + LTS），并在其上适配现代化的 GitHub PR 工作流（替代邮件 + git send-email）。本目录覆盖四方面职责：

1. **代码审查**：GitHub PR review + CODEOWNERS + DCO bot；每个逻辑变更一个 PR；每个 commit 在序列中点都能编译运行（git bisect 友好）；单次 PR 最多 15 个 commit。
2. **贡献流程**：补丁生命周期 6 阶段（Design → Early review → Wider review → Mainline → Stable release → LTS）；`develop` 分支等价 linux-next 树；`release/*` 分支等价 -stable 树。
3. **版本管理**：`Fixes:` / `Closes:` / `Link:` / `Reviewed-by:` 标签体系；DCO 签名链；8 子仓协同开发（跨仓 PR / 跨仓审查 / 跨仓 CI）。
4. **发布流程**：稳定版维护三种提交流径；每 2 年一个 LTS 版本（5 年支持）；季度评审 agentrt ↔ agentrt-linux 同源 API 漂移。

> **注意**：本模块聚焦 agentrt-linux 特定的开发流程设计。工程标准层面的开发流程规范（规则与编号）详见 [50-engineering-standards/05-development-process.md](../50-engineering-standards/05-development-process.md)。

### 1.1 开发流程分层

| 阶段 | 类型 | 输入 | 输出 | 责任人 |
|------|------|------|------|--------|
| 1. Design | 设计 | 需求/问题 | RFC 文档 | 贡献者 |
| 2. Early review | 早期审查 | RFC | 反馈意见 | 子系统维护者 |
| 3. Wider review | 广泛审查 | 修订 RFC + PR | -next 测试 | 顶级维护者 |
| 4. Mainline | 合并入主线 | PR + 标签 | main 提交 | 总维护者 |
| 5. Stable release | 稳定发布 | main | -stable 分支 | 稳定版团队 |
| 6. LTS | 长期支持 | -stable | LTS 版本 | LTS 维护者 |

### 1.2 agentrt-linux 适配

- **SCM 现代化**：GitHub PR（替代邮件 + git send-email）
- **审查工具**：GitHub PR review + CODEOWNERS + DCO bot
- **标签体系**：保留 `Fixes:` / `Closes:` / `Link:` / `Reviewed-by:` 等
- **预览集成**：`develop` 分支等价 linux-next 树
- **稳定版**：`release/*` 分支等价 -stable 树
- **SSoT v2 单一权威源模型**：每个技术点只允许有一个权威源，CI 强制校验
- **CI 强制校验**：`sc-dual-ci.yml`（[SC] 逐字节校验）+ `ssot-validate.yml`（四层模型归属校验）+ 五大技术选型回归测试

---

## 2. 技术选型声明

agentrt-linux v1.0 开发流程在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度遵循 [AirymaxOS 总览](../README.md) §2 的不可妥协基线。开发流程是五大选型的**流程守护者**——CI 强制校验在每次 PR 提交时阻断任何对五大选型的偏离。五个维度的选型在本目录的具体落地如下：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 在本目录的落地 |
|---|---------|---------|----------------|--------------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 `SCHED_EXT=7` 调度类） | CI 强制校验 `airy_defconfig` 中 `CONFIG_SCHED_EXT` 未启用；引入 sched_ext 的 PR 被 CI 阻断 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | CI 强制校验 `CONFIG_IO_URING=y`；引入 page flipping 代码路径的 PR 被 CI 阻断 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 `airy_lsm` 通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | CI 强制校验 `# CONFIG_BPF_LSM is not set` 与 `CONFIG_SECURITY_AIRY_LSM=y`；引入 BPF LSM 钩子的 PR 被 CI 阻断 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | CI 强制扫描共享内存路径未调用 `dma_alloc_coherent`；引入 DMA 一致性内存的 PR 被 CI 阻断 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | （v2 三层模型升级为 v3 四层模型，新增 [DSL] 降级生存层） | `sc-dual-ci.yml` 双端逐字节校验 10 个 [SC] 头文件；`ssot-validate.yml` 校验四层归属一致性；[SC] 变更必须双向评审 |

### 2.1 SSoT v2 单一权威源模型

agentrt-linux 开发流程遵循 SSoT v2（Single Source of Truth v2）单一权威源模型——每个技术点只允许有一个权威源：

| 技术点 | 权威源 | 物理宿主 |
|--------|--------|---------|
| sched_tac 调度 | [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §1.3 + [30-interfaces/10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) | `kernel/include/uapi/linux/airymax/sched.h` |
| IORING_OP_URING_CMD | [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) | `kernel/include/uapi/linux/airymax/ipc.h` |
| 纯 C LSM | [110-security/07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) | `kernel/include/uapi/linux/airymax/lsm_types.h` |
| alloc_pages + mmap | [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §5.1 | `kernel/include/uapi/linux/airymax/log_types.h` |
| IRON-9 v3 四层模型 | [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) | `kernel/include/uapi/linux/airymax/*.h`（10 个） |
| Airymax Unify Design | [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) | — |

### 2.2 IRON-9 v3 四层模型在开发流程的归属

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 落地文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| [SC] 头文件双向评审流程 | ● | — | — | — | [01-patch-lifecycle.md](01-patch-lifecycle.md) |
| CI 强制校验流水线 | — | — | ● | — | [01-patch-lifecycle.md](01-patch-lifecycle.md) |
| 维护者层级制度 | — | — | ● | — | [02-maintainer-hierarchy.md](02-maintainer-hierarchy.md) |
| 跨端 API 语义同源评审 | — | ● | — | — | [02-maintainer-hierarchy.md](02-maintainer-hierarchy.md) |
| [DSL] 降级块 CI 校验 | — | — | — | ● | [01-patch-lifecycle.md](01-patch-lifecycle.md) |

---

## 3. 文档索引

本目录现有文档（v1.0 范围）：

```
120-development-process/
├── README.md                       # 本文件（v1.0）
├── 01-patch-lifecycle.md           # 补丁生命周期 6 阶段 + CI 强制校验
└── 02-maintainer-hierarchy.md      # 维护者层级制度 + 跨端同源 API 评审
```

| # | 文档 | 版本 | 内容概要 |
|---|------|------|---------|
| — | [README.md](README.md) | v1.0 | 开发流程主索引（本文件，含 SSoT v2 单一权威源模型） |
| 1 | [01-patch-lifecycle.md](01-patch-lifecycle.md) | v1.0 | 补丁生命周期 6 阶段（Design → Early review → Wider review → Mainline → Stable release → LTS）、PR 拆分原则、commit 格式、审查流程、CI 强制校验（`sc-dual-ci.yml` + `ssot-validate.yml` + 五大技术选型回归） |
| 2 | [02-maintainer-hierarchy.md](02-maintainer-hierarchy.md) | v1.0 | 维护者层级制度（Lieutenant System）、CODEOWNERS、DCO bot、8 子仓协同开发、agentrt ↔ agentrt-linux 同源 API 季度评审 |

### 3.1 后续规划文档（1.0.1 版本）

以下文档在 1.0.1 版本完成，不在 v1.0 范围内：

- `03-pull-requests.md`：PR 提交规范（详细）
- `04-code-review.md`：代码审查流程（详细）
- `05-stable-releases.md`：稳定版维护（详细）
- `06-long-term-support.md`：LTS 长期支持（详细）
- `07-contribution-workflow.md`：贡献者工作流
- `08-ci-cd-pipeline.md`：CI/CD 流水线（详细）
- `09-release-process.md`：版本发布流程

---

## 4. Airymax Unify Design 映射

本目录与 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的关系详见 [AirymaxOS 总览](../README.md) §5。开发流程作为横切层，主要承载 **SSoT v2 单一权威源模型** 与 **CI 强制校验**，对五模块的变更流程进行统一治理。

| Unify 模块 | 关系 | 在本目录的体现 |
|-----------|------|--------------|
| **A-UEF** | 治理 | A-UEF 的 [SC] `error.h` 变更必须双向评审（agentrt ↔ agentrt-linux）；CI 强制校验双端逐字节一致 |
| **A-ULP** | 治理 | A-ULP 的 [SC] `log_types.h` 变更必须双向评审；128B 记录格式变更触发 SSoT 评审 |
| **A-UCS** | 治理 | A-UCS 的 [SS] 语义同源配置（sysctl ↔ JSON）变更必须同步两端；`airy_defconfig` 锁定五大选型，偏离即 CI 阻断 |
| **A-ULS** | 治理 | A-ULS 的纯 C LSM 模块变更必须通过安全子系统维护者评审；引入 BPF LSM 的 PR 被 CI 阻断 |
| **A-IPC** | 治理 | A-IPC 的 [SC] `ipc.h` 变更必须双向评审；引入 page flipping 的 PR 被 CI 阻断；`IORING_OP_URING_CMD` 路径变更需性能回归测试 |

### 4.1 SSoT v2 与 CI 强制校验权威源引用

- SSoT v2 注册表：[../50-engineering-standards/09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)
- 工程标准开发流程：[../50-engineering-standards/05-development-process.md](../50-engineering-standards/05-development-process.md)
- 维护者制度：[../50-engineering-standards/07-maintainers-and-governance.md](../50-engineering-standards/07-maintainers-and-governance.md)
- CI/CD 流水线：[../70-build-system/03-ci-cd-pipeline.md](../70-build-system/03-ci-cd-pipeline.md)
- 跨项目代码共享：[../50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md)

---

## 5. 相关文档

- [AirymaxOS 总览](../README.md)：v1.0 技术选型声明 + 20 子目录索引
- [架构设计层](../10-architecture/README.md)：Unify Design 总纲 + IRON-9 v3 四层模型
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md)：Airymax Unify Design 总纲（SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（SSoT）
- [10-architecture/05-adrs.md](../10-architecture/05-adrs.md)：架构决策记录（ADR）
- [50-engineering-standards/05-development-process.md](../50-engineering-standards/05-development-process.md)：开发流程规范层
- [50-engineering-standards/07-maintainers-and-governance.md](../50-engineering-standards/07-maintainers-and-governance.md)：维护者制度
- [50-engineering-standards/09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)：SSoT v2 单一权威源注册表
- [50-engineering-standards/11-sc-header-type-bridging.md](../50-engineering-standards/11-sc-header-type-bridging.md)：[SC] 头文件类型桥接规则
- [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md)：跨项目代码共享
- [70-build-system/03-ci-cd-pipeline.md](../70-build-system/03-ci-cd-pipeline.md)：CI/CD 流水线
- [80-testing/README.md](../80-testing/README.md)：测试体系（CI 中的测试集成）
- [130-roadmap/README.md](../130-roadmap/README.md)：路线图与里程碑
- [160-compatibility/03-upstream-tracking.md](../160-compatibility/03-upstream-tracking.md)：上游跟踪

---

## 6. 参考材料

- Linux 6.6 `Documentation/process/development-process.rst`（开发流程主入口）
- Linux 6.6 `Documentation/process/1.Intro.rst` 至 `8.Conclusion.rst`（8 章）
- Linux 6.6 `Documentation/process/submitting-patches.rst`（提交补丁规范）
- Linux 6.6 `Documentation/process/stable-kernel-rules.rst`（稳定版规则）
- Linux 6.6 `MAINTAINERS`（维护者制度范本）
- GitHub PR 工作流（CODEOWNERS / DCO bot / branch protection）

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，README + 01 + 02 文档奠基，确立补丁生命周期/维护者层级核心机制 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap / IRON-9 v3 四层模型五大技术选型声明（CI 强制校验阻断偏离）；新增 Airymax Unify Design 映射（SSoT v2 单一权威源模型 + CI 强制校验为五模块治理核心）；新增 SSoT v2 权威源清单；文档索引对齐实际目录文件 |

---

> **文档结束** | agentrt-linux 开发流程设计 v1.0 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
