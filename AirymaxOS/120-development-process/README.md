Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）开发流程设计

> **文档定位**： agentrt-linux（AirymaxOS）开发流程工程体系主索引
> **版本**： 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**： 2026-07-06
> **同源映射**： agentrt 开发流程 + Linux 6.6 内核开发流程（development-process.rst 8 章）
> **理论根基**： Linux 内核开发流程 + Airymax S-4 涌现性管理 + C-2 增量演化

---

## 1. 模块定位

agentrt-linux 开发流程是代码从设计到主线再到稳定发布的全生命周期规范。它继承 Linux 内核 30+ 年沉淀的开发流程哲学（补丁生命周期 + 维护者层级 + 审查优先 + 稳定版维护 + LTS），并在其上适配现代化的 GitHub PR 工作流（替代邮件 + git send-email）。

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
- **标签体系**：保留 `Fixes:`/`Closes:`/`Link:`/`Reviewed-by:` 等
- **预览集成**：`develop` 分支等价 linux-next 树
- **稳定版**：`release/*` 分支等价 -stable 树

---

## 2. 核心开发流程

### 2.1 补丁生命周期（6 阶段）

```
Design → Early review → Wider review → Mainline → Stable release → Long-term maintenance
```

每个阶段都有明确的输入、输出、责任人和 SLA。

### 2.2 维护者层级制度（Lieutenant System）

```
普通贡献者
    ↓ 提交 PR
子系统维护者（airymaxos-kernel、airymaxos-services 等）
    ↓ pull 请求
顶级子系统维护者（airymaxos 总维护者）
    ↓ pull 请求
agentrt-linux 总维护者（BDFL 角色）
```

### 2.3 -next 树等价物

`develop` 分支作为预览集成分支：
- 所有应进入下一 merge window 的 PR 必须先入 develop
- develop 持续集成测试，暴露冲突与 regression
- develop 等价 linux-next 树

### 2.4 补丁提交规范

#### PR 拆分原则
- 每个**逻辑变更**一个 PR
- 不混合不同类型改动（bug fix / 性能优化 / API 改动 / 新驱动）
- 每个 PR 必须能独立审查、独立验证
- 每个 commit 在序列中点都应能编译运行（git bisect 友好）
- 单次 PR 最多 15 个 commit

#### Commit 格式
```
subsystem: summary phrase

Description body (75 columns wide):
- Describe the problem
- Describe the user-visible impact
- Quantify optimizations and trade-offs
- Use imperative mood

Signed-off-by: Author Name <author@example.com>
Reviewed-by: Reviewer Name <reviewer@example.com>

---
Additional comments (not in changelog)

Actual diff
```

### 2.5 审查流程

- 必须响应每条审查意见
- 不同意需解释技术理由
- 忽略审查是致命错误
- Andrew Morton 建议：每条未导致代码改动的审查意见都应转化为代码注释

### 2.6 稳定版维护

- `release/0.1.x`、`release/1.0.x` 等价 -stable 树
- 三种提交流径：
  - Option 1：主线 commit 描述中加 `Cc: stable@airymaxos.org`
  - Option 2：commit 已 mainline 后请求 stable 团队捡取
  - Option 3：提交等价于已 mainline 的补丁给 stable 团队

### 2.7 长期支持（LTS）

- 每 2 年一个 LTS 版本
- LTS 版本获更长支持（5 年）
- LTS 维护者需持续负责

---

## 3. 文档索引

```
120-development-process/
├── README.md                       # 本文件
├── 01-patch-lifecycle.md           # 补丁生命周期 6 阶段
├── 02-maintainer-hierarchy.md      # 维护者层级制度
├── 03-pull-requests.md             # PR 提交规范
├── 04-code-review.md               # 代码审查流程
├── 05-stable-releases.md           # 稳定版维护
├── 06-long-term-support.md         # LTS 长期支持
├── 07-contribution-workflow.md     # 贡献者工作流
├── 08-ci-cd-pipeline.md            # CI/CD 流水线
└── 09-release-process.md           # 版本发布流程
```

### 3.1 0.1.1 版本范围

0.1.1 完成 README + 01 + 02 文档（3 文档奠基），确立开发流程设计框架与补丁生命周期/维护者层级核心机制。其余 7 文档（03-pull-requests 至 09-release-process）在 1.0.1 版本完成。

### 3.2 1.0.1 版本范围

完成剩余 7 文档（03-pull-requests 至 09-release-process），实施开发流程工程标准，进行代码级验证。

---

## 4. agentrt-linux 专属扩展

### 4.1 8 子仓协同开发流程

8 个子仓（kernel/services/security/memory/cognition/cloudnative/system/tests）的协同开发：
- **跨仓 PR**：通过 submodule 更新触发上游仓 PR
- **跨仓审查**：CODEOWNERS 跨仓引用
- **跨仓 CI**：触发上下游仓的 CI 测试

### 4.2 同源 agentrt 协同

agentrt 与 agentrt-linux 的同源 API（MicroCoreRT/AgentsIPC/Cupolas/MemoryRovol/CoreLoopThree）变更流程：
- **RFC 双向同步**：agentrt 端 RFC 必须同步到 agentrt-linux 端，反之亦然
- **兼容性测试**：变更必须通过两端兼容性测试
- **季度评审**：季度评审同源 API 漂移

### 4.3 IRON-9 v2 同源且部分代码共享

开发流程遵循 IRON-9 原则：
- agentrt 开发流程（用户态运行时规范）
- agentrt-linux 开发流程（内核发行版规范）
- 两端独立演进，但通过同源 API 保持互操作

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **S-4 涌现性管理** | 渐进式开发 + Merge Window + RC |
| **C-2 增量演化** | git bisect 友好 + commit 序列中点可编译 |
| **E-6 错误可追溯** | Fixes/Closes/Link 标签 + DCO 链条 |
| **A-3 人文关怀** | 审查礼仪 + 不烧桥管理哲学 |
| **IRON-9 v2 同源且部分代码共享** | 与 agentrt 开发流程并行 |

---

## 6. 相关文档

- `50-engineering-standards/05-development-process.md`（开发流程规范层）
- `50-engineering-standards/07-maintainers-and-governance.md`（维护者制度）
- `130-roadmap/README.md`（路线图与里程碑）
- `50-engineering-standards/06-toolchain-and-automation.md`（CI/CD）

---

## 7. 参考材料

- Linux 6.6 `Documentation/process/development-process.rst`（开发流程主入口）
- Linux 6.6 `Documentation/process/1.Intro.rst` 至 `8.Conclusion.rst`（8 章）
- Linux 6.6 `Documentation/process/submitting-patches.rst`（提交补丁规范）
- Linux 6.6 `Documentation/process/stable-kernel-rules.rst`（稳定版规则）
- Linux 6.6 `MAINTAINERS`（维护者制度范本）

---

> **文档结束** | 0.1.1 P0 优先完成 README + 01 + 02
