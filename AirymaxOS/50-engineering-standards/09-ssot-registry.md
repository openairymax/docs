Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax 全局规则 SSoT 注册表
> **文档定位**：Airymax 全项目（agentrt 用户态运行时 + AirymaxOS/agentrt-linux 操作系统）全部编号、规则、命名的**唯一权威来源（Single Source of Truth）**。任何文档不得私自定义规则编号；所有规则编号必须在此注册表中登记后方可使用。\
> **文档版本**：0.2.0\
> **最后更新**：2026-07-17\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **权威性声明**：本注册表是 Airymax 全部规则编号的 SSoT。当任何文档与本注册表冲突时，以本注册表为准。各主题文档可包含规则的详细说明与代码示例，但必须引用本注册表中的编号，**禁止在主题文档中私自定义新编号**。新增规则编号必须通过 RFC 流程在本注册表登记。  

---

## 目录

- [第 1 章 注册表使用规则](#第-1-章-注册表使用规则)
- [第 2 章 OS-IRON 工程铁律](#第-2-章-os-iron-工程铁律)
- [第 3 章 OS-KER 内核工程规则](#第-3-章-os-ker-内核工程规则)
- [第 4 章 OS-STD 标准规则](#第-4-章-os-std-标准规则)
- [第 5 章 OS-BAN 禁止规则](#第-5-章-os-ban-禁止规则)
- [第 6 章 OS-ACC 验收标准](#第-6-章-os-acc-验收标准)
- [第 7 章 OS-ABI 接口稳定性](#第-7-章-os-abi-接口稳定性)
- [第 8 章 命名规范](#第-8-章-命名规范)
- [第 9 章 OS-SEC 安全工程规则](#第-9-章-os-sec-安全工程规则)
- [第 10 章 OS-ARCH 架构工程规则](#第-10-章-os-arch-架构工程规则)
- [第 11 章 OS-IFACE 接口工程规则](#第-11-章-os-iface-接口工程规则)
- [第 12 章 OS-TEST 测试工程规则](#第-12-章-os-test-测试工程规则)
- [第 13 章 其他 OS-\* 工程规则扩展登记](#第-13-章-其他-os--工程规则扩展登记)
- [第 14 章 agentrt 规则编号体系](#第-14-章-agentrt-规则编号体系)

***

## 第 1 章 注册表使用规则

### 1.1 核心原则

**每个技术点只能有一个权威编号。** 本注册表是全部规则编号的唯一分配者；主题文档是规则详细说明的载体，但编号的分配权仅属于本注册表。本注册表覆盖两套同源但独立的编号体系：AirymaxOS `OS-*` 前缀规则（第 2-13 章）与 agentrt 无 `OS-` 前缀规则（第 14 章）。

### 1.2 使用规则

| 规则         | 说明                                                                                                   |
| ---------- | ---------------------------------------------------------------------------------------------------- |
| **编号分配权**  | 所有编号（AirymaxOS `OS-*` + agentrt 无前缀）的分配权仅属于本注册表。主题文档不得自行创建新编号。                                       |
| **引用义务**   | 主题文档使用规则编号时，必须引用本注册表中已登记的编号。若该编号不存在，须先通过 RFC 流程在本注册表登记。                                              |
| **详细说明**   | 主题文档可包含规则的详细说明、代码示例、checkpatch 映射等，但编号须与本注册表一致。                                                      |
| **冲突解决**   | 当任何文档与本注册表冲突时，以本注册表为准。                                                                               |
| **语义去重**   | 同一技术点在多个文档中出现时，应使用同一个编号（交叉引用），而非各自定义新编号。                                                             |
| **编号体系隔离** | AirymaxOS `OS-*` 编号与 agentrt 无前缀编号属于不同编号空间，禁止混用（如禁止将 `OS-IRON-001` 与 `IRON-1` 视为同一编号）。跨体系引用时须标注来源体系。 |

### 1.3 编号格式

本注册表包含两套编号格式：

**AirymaxOS 编号格式**（第 2-13 章）：

```
OS-<前缀>-<子域>-<NNN>
```

| 前缀   | 子域   | 说明                  |
| ---- | ---- | ------------------- |
| IRON | —    | 工程铁律（不可妥协）          |
| KER  | —    | 内核工程规则              |
| STD  | CODE | 编码规范                |
| STD  | FMT  | 格式规范                |
| STD  | STY  | 风格规范                |
| STD  | GOV  | 治理规范                |
| STD  | TOOL | 工具链与自动化规范（7 层验证体系）  |
| STD  | PROD | 开发流程规范（补丁生命周期/审查响应） |
| BAN  | —    | 禁止规则                |
| ACC  | —    | 验收标准                |
| ABI  | —    | 接口稳定性               |

**agentrt 编号格式**（第 14 章，无 `OS-` 前缀）：

```
<前缀>-<NN>
```

agentrt 17 类规则前缀详见 [第 14 章](#第-14-章-agentrt-规则编号体系)。

### 1.4 新增编号流程

1. 在本注册表对应章节追加新条目（编号、规则简述、权威定义文档）
2. 在主题文档中引用该编号并展开详细说明
3. 通过 PR 审查确认编号唯一性与语义无重复

### 1.5 登记方式与验证方法（2026-07-15 补全）

本注册表采用两种登记方式：

| 登记方式     | 适用场景                    | 格式                                 | 验证方法               |
| -------- | ----------------------- | ---------------------------------- | ------------------ |
| **全量登记** | 规则数量少（≤30 条）或语义重要性高     | 逐条列出编号 + 规则简述 + 权威定义               | grep 提取具体编号字符串直接比对 |
| **摘要登记** | 规则数量多（>30 条）或属同一子域的连续编号 | 列出编号区间（如 `OS-OPS-001~041`）+ 权威定义文档 | 须将区间表达式展开为具体编号后比对  |

**摘要登记覆盖声明**：以下子节采用摘要登记方式，其覆盖的完整编号范围以区间表达式声明。区间内的所有编号视为已在本注册表登记，主题文档可直接引用：

| 子节                 | 覆盖区间                                  | 编号数   | 权威定义                                   |
| ------------------ | ------------------------------------- | ----- | -------------------------------------- |
| §4.1 OS-STD-CODE   | OS-STD-CODE-001\~059                  | 58    | 01-coding-standards.md                 |
| §4.5 OS-STD-RUST   | OS-STD-030\~057                       | 28    | 10-coding-style/Rust\_coding\_style.md |
| §4.6 OS-STD-TOOL   | OS-STD-TOOL-001\~161                  | \~161 | 05-development-process.md Part II      |
| §4.6.1 OS-STD-PROD | OS-STD-PROD-001\~158                  | \~158 | 05-development-process.md              |
| §4.9 OS-STD-DOC    | OS-STD-DOC-001\~014                   | 14    | 01-coding-standards.md Part IV         |
| §6 OS-ACC          | OS-ACC-001\~110                       | 110   | 130-roadmap/06-acceptance-criteria.md  |
| §13.1 OS-OPS       | OS-OPS-001\~041 + OS-OPS-101\~140     | \~81  | 100-operations/                        |
| §13.2 OS-IPC       | OS-IPC-001\~010                       | 10    | 40-dataflows/03-ipc-flow\.md           |
| §13.3 OS-CHK-DOC   | OS-CHK-DOC-01\~08, 18\~19             | 10    | 05-development-process.md              |
| §13.4 OS-CHK-CODE  | OS-CHK-CODE-01\~05, 08                | 6     | 05-development-process.md              |
| §13.5 OS-CHK-IRON  | OS-CHK-IRON-01\~04, 18                | 5     | 05-development-process.md              |
| §13.6 OS-DEV       | OS-DEV-101\~194 + OS-DEV-201\~282     | \~60  | 120-development-process/               |
| §13.7 OS-BUILD     | OS-BUILD-001\~016 + OS-BUILD-021\~028 | 24    | 70-build-system/                       |
| §13.8 OS-DRV       | OS-DRV-001\~035                       | \~26  | 60-driver-model/                       |
| §13.9 OS-OBS       | OS-OBS-001\~022                       | 22    | 90-observability/                      |
| §13.10 OS-MM       | OS-MM-001\~002                        | 2     | 40-dataflows/02-memory-flow\.md        |
| §13.11 OS-TST      | OS-TST-013\~016                       | 4     | 80-testing/01-kunit-framework.md       |

**OS-STD 裸编号全量覆盖声明**：除上述子前缀编号外，主题文档中使用的 `OS-STD-NNN` 裸编号（无子前缀，如 `OS-STD-001`、`OS-STD-101`）覆盖 `OS-STD-001~218` 全部 135 条，权威定义散布于 `01-coding-standards.md` 各 Part 及 `05-development-process.md`。本注册表 §4 各子节已登记部分代表性裸编号（如 §4.2 末尾 OS-STD-010/011、§4.5 OS-STD-030\~057、§4.2 OS-STD-201\~211 迁移映射），其余裸编号通过本声明覆盖。

**验证工具约定**：自动化验证脚本须将区间表达式（如 `OS-OPS-001~041`）展开为具体编号列表后，再与主题文档使用的编号比对。仅用 grep 提取 `OS-*-NNN` 字符串的简单比对会误报摘要登记的编号为"未登记"。

### 1.6 SSoT 三层模型（Tier 0 / Tier 1 / Tier 2）

> **模型定位**：Airymax 全项目 SSoT 按"权威强度"划分为三层（Tier 0 / Tier 1 / Tier 2），每层对应不同的变更门槛与 CI 校验强度。本模型是 §1.1"每个技术点只能有一个权威编号"的组织化落地，解决"权威源众多但强度不一"的治理问题。

| 层级 | 名称 | 权威强度 | 变更门槛 | CI 校验 | 覆盖范围 |
|------|------|---------|---------|---------|---------|
| **Tier 0** | 不可变权威源 | 最高——一经定义永不破坏 | 需 TSC 全体通过 + 跨端双向评审 + ADR 记录 | `sc-dual-ci.yml` 逐字节哈希锁定 | 用户空间 ABI（OS-IRON-001）、[SC] 共享契约物理宿主（`kernel/include/uapi/linux/airymax/*.h`）、IPC magic、日志 magic、128B 消息头布局 |
| **Tier 1** | 强权威源 | 高——变更需 RFC + 双向评审 | RFC 流程 + 本注册表登记 + 主题文档同步 | `ssot-validate.yml` 一致性校验 | 错误码枚举、日志级别枚举、capability 41 ID、调度参数、IRON-9 四层模型、Unify Design 5 模块边界、[DSL] 降级块、内核基线（Linux 6.6 / 7.1）、Agent 8 态生命周期 |
| **Tier 2** | 常规权威源 | 中——变更需 PR 评审 | PR 评审 + 本注册表登记 | `ssot-validate.yml` 编号唯一性校验 | 本注册表第 2-13 章登记的 `OS-*` 规则编号、第 14 章 agentrt 17 类规则编号 |

**层级判定规则**：

1. **Tier 0 判定**：技术点进入二进制 ABI / 物理共享头文件 / 跨端 magic 值 → Tier 0，变更等同 ABI 破坏，受 OS-IRON-001 约束
2. **Tier 1 判定**：技术点是跨端语义契约（枚举、模型、模块边界）但不直接进 ABI → Tier 1
3. **Tier 2 判定**：技术点是单端工程规则（编码规范、验收标准等） → Tier 2

**层间关系**：Tier 0 是 Tier 1 的物理承载（Tier 1 的语义经 Tier 0 的 [SC] 头文件物化）；Tier 1 是 Tier 2 的语义约束源（Tier 2 规则不得违反 Tier 1 契约）。冲突时优先级：Tier 0 > Tier 1 > Tier 2。

### 1.7 技术点单一权威源清单

> **清单定位**：下表登记 Airymax 全项目核心技术点的**唯一权威源**。每个技术点只能有一行登记，主题文档引用时必须指向此处的权威文档。新增技术点权威源须通过 RFC 在本表登记。

| # | 技术点 | Tier | 唯一权威源文档 | 物理宿主 / 编号载体 |
|---|--------|------|--------------|-------------------|
| 1 | 错误码（`AIRY_E*`）+ 故障码（`AIRY_FAULT_*`） | Tier 0 | [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) | [SC] `error.h`（`kernel/include/uapi/linux/airymax/error.h`） |
| 2 | 日志类型（`LOG_*` 5 级 + 128B 记录 + printk 8 级映射） | Tier 0 | [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) | [SC] `log_types.h` |
| 3 | IPC 协议（128B 消息头 + magic 0x41524531 + 操作码） | Tier 0 | [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) | [SC] `ipc.h` |
| 4 | 调度策略（sched_tac：SCHED_DEADLINE/SCHED_FIFO/EEVDF） | Tier 1 | [10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) | [SC] `sched.h` |
| 5 | 安全模型（capability 41 ID + LSM 钩子 + Cupolas blob） | Tier 1 | [03-capability-model.md](../110-security/03-capability-model.md) | [SC] `security_types.h` + `lsm_types.h` |
| 6 | 微内核策略（seL4 思想分布落地 + 机制策略分离） | Tier 1 | [03-microkernel-strategy.md](../10-architecture/03-microkernel-strategy.md) | OS-ARCH-005 / OS-ARCH-006 |
| 7 | IRON-9 模型（[SC]/[SS]/[IND]/[DSL] 四层） | Tier 1 | [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) | OS-IRON-008 / OS-IRON-014 |
| 8 | Unify Design（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC 5 模块） | Tier 1 | [10-unify-design.md](../10-architecture/10-unify-design.md) | — |
| 9 | [DSL] 降级生存层（[SC] 损坏时最小可运行子集） | Tier 1 | [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) | 每个 [SC] 头文件 `#ifdef AIRY_SC_FALLBACK` 块 |
| 10 | 内核基线（1.x.x Linux 6.6 LTS / 2.x.x Linux 7.1） | Tier 1 | [04-engineering-baseline.md](../10-architecture/04-engineering-baseline.md) | ADR-013 |
| 11 | Agent 8 态生命周期 | Tier 1 | [01-agent-lifecycle.md](../140-application-development/01-agent-lifecycle.md) | [SC] `cognition_types.h` |

**使用规则**：

- 主题文档定义上述技术点的细节时，**必须**引用本表登记的权威源，禁止重新定义
- 本表新增技术点须标注 Tier 层级与物理宿主；Tier 0 技术点变更触发 `sc-dual-ci.yml` 哈希重算
- 跨技术点的复合设计（如 IPC + capability + 调度）须交叉引用多个权威源，不得在单一文档中合并定义

### 1.8 CI 强制校验机制

> **机制定位**：SSoT 三层模型的强制力由 CI 工作流保障。下列 workflow 在每个 PR 上运行，违反任一校验即阻断合入。CI 校验是 §1.6 三层模型与 §1.7 权威源清单的自动化执行手段。

| # | CI Workflow | 校验内容 | 覆盖层级 | 失败动作 |
|---|------------|---------|---------|---------|
| 1 | `ci-kernel.yml` | kernel Kbuild 18 配置矩阵 + checkpatch + sparse + Coccinelle + KUnit + kselftest | Tier 0/1 | 阻断合入 |
| 2 | `sc-dual-ci.yml` | agentrt 与 AirymaxOS 两端 [SC] 头文件逐字节一致 + 编译器无关性 + Tier 0 权威源逐字节哈希校验（[SC] 10 头文件 + magic 值）+ 禁止新增 `AIRY_LOG_*` 调用 / `AIRY_ERR_*` 引用（对齐日志收敛/错误码迁移阶段 3+） | Tier 0/1 | 阻断合入，Tier 0 变更要求 TSC 评审 |
| 3 | `ssot-validate.yml` | §1.7 权威源清单引用一致性 + [SC] 头文件数量为 10 + 每个 [SC] 有 [DSL] 降级块 + [SS] 语义映射文档存在 + [IND] 实现未泄露到 [SC]/[SS] + Unify Design 5 模块边界一致性 + sched_tac 调度选型（禁 sched_ext）+ IPC 零拷贝选型（禁 page flipping）+ 内存选型（禁 DMA 一致性内存用于日志/IPC） | Tier 0/1 | 阻断合入 |
| 4 | `mgmt-orchestrator.yml` | SSoT 规则 ID 校验 + 本注册表登记的 `OS-*` / agentrt 编号唯一性 + 主题文档引用编号均在注册表覆盖范围内 + 文件完整性（8 子仓 submodule 指针 + [SC] 6+2 头文件物理存在）+ 子仓 CI 状态聚合 + 文档格式（markdownlint + 版权头 + 链接有效性） | Tier 1/2 | 阻断合入 |
| 5 | `nightly.yml` | develop 夜间构建（L5/L6 连接/协议验证）+ 性能回归检测 | — | 标记回退 |
| 6 | `release.yml` | release tag 流水线（L7 发布验证）+ ABI 兼容性校验 | — | 阻断发布 |

**CI 与三层模型的映射**：

- **Tier 0**：workflow #2 哈希锁定 + 双端逐字节校验，变更门槛最高
- **Tier 1**：workflow #2 / #3 一致性 + 选型约束 + 禁止引用校验
- **Tier 2**：workflow #4 编号唯一性校验

**例外与豁免**：Tier 0 变更（如 [SC] 头文件演进）须经 TSC 评审通过后在 PR 中标注 `ssot-tier0-exception` 标签，CI 临时放行哈希校验，但合入后须立即重算 Tier 0 哈希基线。任何豁免须在附录 A 变更日志登记。

***

## 第 2 章 OS-IRON 工程铁律

> **权威定义文档**：04-engineering-philosophy.md

| 编号          | 规则                                                                                   | 权威定义                                     | 五维映射      |
| ----------- | ------------------------------------------------------------------------------------ | ---------------------------------------- | --------- |
| OS-IRON-001 | 用户空间 ABI 永不破坏                                                                        | 04 §2.1                                  | K-2 / E-7 |
| OS-IRON-002 | 内核内部 API 不保证稳定；改动者必须修复所有调用点                                                          | 04 §2.2                                  | K-1 / C-2 |
| OS-IRON-003 | 策略与机制分离                                                                              | 04 §3                                    | K-4       |
| OS-IRON-004 | 渐进式开发，补丁自包含                                                                          | 04 §4 / 05 开发流程                          | S-4 / C-2 |
| OS-IRON-005 | 审查优先文化                                                                               | 04 §6 / 07 §6                            | S-3 / S-1 |
| OS-IRON-006 | 不破坏用户空间（regression 零容忍）                                                              | 04 §9 / 05 开发流程                          | E-6       |
| OS-IRON-007 | DCO 强制                                                                               | 07 §5                                    | E-6       |
| OS-IRON-008 | \[SC] 共享契约层双向 CI 验证                                                                  | 120 跨项目代码共享                              | E-6 / E-8 |
| OS-IRON-009 | 代码共享边界：仅 agentrt ↔ AirymaxOS                                                         | 120 / 07 §1.4                            | —         |
| OS-IRON-010 | "Linux 6.6 为基、seL4 为鉴"工程取向                                                           | 04 §12                                   | —         |
| OS-IRON-011 | 双源边界声明（01Reference/ 仅本地参考）                                                           | 04 §12                                   | —         |
| OS-IRON-012 | seL4 借鉴仅限架构层（ES-SEL4-1\~5）                                                           | 04 §12                                   | —         |
| OS-IRON-013 | 8 子仓独立 git 仓库 + submodule                                                            | 04 §13                                   | S-2       |
| OS-IRON-014 | \[SC] 共享契约层 10 个核心头文件单一数据源（禁止物理副本）——10 个 [SC] 头文件物理宿主在 `kernel/include/uapi/linux/airymax/`，其他子仓通过 -I 引用（bpf\_struct\_ops.h 为补充共享文件，非 [SC] 核心）                   | 120 跨项目代码共享                              | E-7       |
| OS-IRON-015 | 编号管理元规则——OS-KER / OS-STD / OS-OBS / OS-DRV 等所有规则编号一经分配不得复用；废弃规则标记 `DEPRECATED` 但保留编号 | 90-observability/02-ebpf-probes.md §14.2 | S-1       |

### 2.1 ES-SEL4 编号范围声明

> **声明目的**：开源文档 `10-architecture/03-microkernel-strategy.md` 在引用 seL4 工程思想时使用了 ES-SEL4-21\~43 等扩展编号，但 SSoT 注册表此前的 ES-SEL4 登记仅覆盖 ES-SEL4-1\~5（经 OS-IRON-012 引用）。为消除编号范围歧义，特此声明 ES-SEL4 编号的三段作用域。

| 编号段            | 作用域     | 登记状态                | 权威定义                                                     |
| -------------- | ------- | ------------------- | -------------------------------------------------------- |
| ES-SEL4-1\~5   | 架构层强制借鉴 | 已注册（OS-IRON-012 引用） | 04-engineering-philosophy.md §12 / 闭源总纲 §12.6            |
| ES-SEL4-6\~10  | P2 增强建议 | 已注册（闭源总纲附录 D）       | 闭源总纲附录 D                                                 |
| ES-SEL4-11\~43 | 研究范围编号  | 未注册为工程标准            | `_research_0.2.5/01-sel4-deep-analysis.md`（研究范围，非工程标准范围） |

> **使用约定**：ES-SEL4-11\~43 属于 seL4 源码深度研读的研究范围编号（研究文件共提炼 54 条 ES-SEL4 工程思想），非工程标准范围。在开源文档中引用 ES-SEL4-11\~43 时，必须标注为"研究范围参考"，不得作为强制工程规则引用。

***

## 第 3 章 OS-KER 内核工程规则

> **编号区间说明**：001-010 由 01-coding-standards.md 权威定义；011-021 由 C\_Cpp\_coding\_style.md Part I 权威定义；022-029 由 02-kconfig-system.md 权威定义；046 由 C\_Cpp\_coding\_style.md Part I 权威定义；053-056 由 01-coding-standards.md 权威定义；060-069 由 04-engineering-philosophy.md 权威定义；071-085 由各主题文档权威定义；086-155 为历史冲突消解后的唯一编号（2026-07-12 全量消解）。（2026-07-13 更新：C\_coding\_style\_standard.md 已合并入 C\_Cpp\_coding\_style.md Part I）

### 3.1 01-coding-standards.md 权威定义（001-010, 053, 056）

| 编号         | 规则                                                                                                                                                                                                       | 章节      |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| OS-KER-001 | goto 集中出口模式                                                                                                                                                                                              | 01 §7.1 |
| OS-KER-002 | 禁止 #if 桩函数模式（由 IRON-1 涵盖）                                                                                                                                                                                | SSoT 内部 |
| OS-KER-003 | 条件编译必须用 IS\_ENABLED()                                                                                                                                                                                    | SSoT 内部 |
| OS-KER-004 | 分级标签按分配逆序释放                                                                                                                                                                                              | 01 §7.2 |
| OS-KER-005 | \[已废弃] 原定义为"C 内核代码 Tab 8 字符缩进"，因格式规则重新组织（内容迁移至 OS-STD-FMT-001 / 闭源 OS-FMT-001）而废弃                                                                                                                        | —       |
| OS-KER-006 | volatile 仅 4 种例外                                                                                                                                                                                         | 01 §6.3 |
| OS-KER-007 | \[已废弃] 原定义为"内核态禁 float，强制 q16.16 定点"，于 2026-07-12 迁移至 OS-STD-010（与 01-coding-standards.md Part II §0.2.1 迁移映射表一致）                                                                                        | —       |
| OS-KER-008 | \[已废弃] 原定义为"函数例外：开括号在下一行行首（K\&R）"，于 2026-07-12 与 OS-STD-047 合并迁移至 OS-STD-FMT-009（与 01-coding-standards.md Part II §0.2.1 迁移映射表一致）                                                                        | —       |
| OS-KER-009 | 忙等循环必须调用 cpu\_relax()                                                                                                                                                                                    | 01 §8.1 |
| OS-KER-010 | 受保护数据结构须声明 magic 字段并注册                                                                                                                                                                                   | 01 §8.2 |
| OS-KER-011 | \[已废弃] 原定义为空格规则之一（关键字后空格 / sizeof 无空格 / 括号内禁空格等），于 2026-07-12 与 OS-KER-010/014 合并迁移至 OS-STD-FMT-005（空格规则，合并 3 条为 1 条）。迁移映射见 01-coding-standards.md Part II §0.2.1。                                       | —       |
| OS-KER-012 | \[已废弃] 原定义为"指针声明 `*` 贴名字"（指针对齐），于 2026-07-12 单独迁移至 OS-STD-FMT-006（指针对齐，§4.3 单独条目）。迁移映射见 01-coding-standards.md Part II §0.2.1 L736。                                                                      | —       |
| OS-KER-013 | \[已废弃] 原定义为"禁止结构体 typedef"（typedef 仅 5 种例外），于 2026-07-12 单独迁移至 OS-STD-CODE-020（NEW\_TYPEDEFS）。迁移映射见 01-coding-standards.md Part II §0.2.1 + 变更日志 L2747 + C\_Cpp\_coding\_style.md L817/L1154/L1399 三方确认。 | —       |
| OS-KER-014 | \[已废弃] 原定义为空格规则之一（运算符两侧加空格 / 一元运算符后无空格），于 2026-07-12 与 OS-KER-010/011 合并迁移至 OS-STD-FMT-005（空格规则，合并 3 条为 1 条）。迁移映射见 01-coding-standards.md Part II §0.2.1。                                                | —       |
| OS-KER-053 | 共享数据必须用同步原语保护                                                                                                                                                                                            | 01 §6.4 |
| OS-KER-056 | I/O 内存必须通过 accessor 访问                                                                                                                                                                                   | 01 §6.5 |

### 3.2 C\_Cpp\_coding\_style.md Part I 权威定义（011-021, 046, 070）

> **合并说明（2026-07-13）**：原 `C_coding_style_standard.md` 已合并入 `C_Cpp_coding_style.md` Part I。下表章节引用由 `C_coding_style §X` 更新为 `C_Cpp_coding_style Part I §X`。

| 编号         | 规则                                             | 章节                               |
| ---------- | ---------------------------------------------- | -------------------------------- |
| OS-KER-015 | 函数一屏可读（≤80×24），局部变量 ≤5-10                      | C\_Cpp\_coding\_style Part I §3  |
| OS-KER-016 | GFP 选型：GFP\_KERNEL / GFP\_ATOMIC / GFP\_NOWAIT | C\_Cpp\_coding\_style Part I §4  |
| OS-KER-017 | 释放内存后指针置 NULL                                  | C\_Cpp\_coding\_style Part I §5  |
| OS-KER-018 | krealloc 用临时变量接收                               | C\_Cpp\_coding\_style Part I §6  |
| OS-KER-019 | 锁选型：mutex / spinlock / rwlock / RCU            | C\_Cpp\_coding\_style Part I §7  |
| OS-KER-020 | RCU 读写端规则                                      | C\_Cpp\_coding\_style Part I §8  |
| OS-KER-021 | \[SC] 共享契约层代码约束                                | C\_Cpp\_coding\_style Part I §9  |
| OS-KER-046 | 锁设计在数据结构设计阶段完成                                 | C\_Cpp\_coding\_style Part I §10 |
| OS-KER-070 | \[SS] 语义同源层 API 签名约束                           | C\_Cpp\_coding\_style Part I §11 |

> **注**：历史 OS-KER-011\~014（空格规则 / 指针对齐 / typedef 规则 / 空格规则）已于 2026-07-12 迁移：OS-KER-011/014 → OS-STD-FMT-005（合并）、OS-KER-012 → OS-STD-FMT-006（单独）、OS-KER-013 → OS-STD-CODE-020（单独）。显式 \[已废弃] 条目见 §3.1（2026-07-15 补全，消除历史注释与显式条目不一致）。迁移映射见 01-coding-standards.md Part II §0.2.1。

### 3.3 02-kconfig-system.md 权威定义（022-029）

| 编号         | 规则                               | 章节            |
| ---------- | -------------------------------- | ------------- |
| OS-KER-022 | 互斥决策必须用 `choice`                 | 02-kconfig §2 |
| OS-KER-023 | `select` 目标的依赖须同时满足              | 02-kconfig §3 |
| OS-KER-024 | 可模块化子系统必须用 `tristate`            | 02-kconfig §4 |
| OS-KER-025 | `obj-$(CONFIG_*)` 的宏须有 config 声明 | 02-kconfig §5 |
| OS-KER-026 | 新增 Kconfig 须被父 Kconfig source    | 02-kconfig §6 |
| OS-KER-027 | Kconfig.airymaxos 仅聚合跨子系统特性      | 02-kconfig §7 |
| OS-KER-028 | 新 choice/menuconfig 须用工具实际验证     | 02-kconfig §8 |
| OS-KER-029 | airymaxos-base.config 变更经评审      | 02-kconfig §9 |
| OS-KER-030 | 内核态配置必须以 Kconfig 为唯一描述手段         | 02-kconfig §1 |

### 3.4 04-engineering-philosophy.md 权威定义（060-069）

| 编号         | 规则                        | 章节      |
| ---------- | ------------------------- | ------- |
| OS-KER-060 | API 改动者必须修复所有调用点          | 04 §2.2 |
| OS-KER-061 | 内核不内置策略（IRON-003 子规则）     | 04 §3.2 |
| OS-KER-062 | 补丁序列中点可编译（IRON-004 子规则）   | 04 §4.3 |
| OS-KER-063 | 逻辑变更拆分为可编译补丁序列            | 04 §4.3 |
| OS-KER-064 | 补丁顺序反映逻辑依赖                | 04 §4.3 |
| OS-KER-065 | bug 修复补丁必须附带回归测试          | 04 §5.4 |
| OS-KER-066 | bug 修复复杂度与严重性匹配           | 04 §5.4 |
| OS-KER-067 | bug 修复补丁必须含 Fixes: 标签     | 04 §7.1 |
| OS-KER-068 | 补丁必须含 Signed-off-by DCO 链 | 04 §7.4 |
| OS-KER-069 | 新 /proc 条目必须文档化           | 04 §8.1 |

### 3.5 主题文档权威定义（054-055, 071-085）

| 编号         | 规则                                                      | 权威定义文档                                  |
| ---------- | ------------------------------------------------------- | --------------------------------------- |
| OS-KER-054 | ruleset 是策略载体，运行时不可变（Landlock）                          | 110-security/02-landlock-sandbox.md     |
| OS-KER-055 | Agent 跟踪必须用 trace\_array\_create() 独立 instance          | 90-observability/01-ftrace-framework.md |
| OS-KER-071 | 系统级测试与 KUnit 契约测试共享同一契约文档                               | 80-testing/02-kselftest.md              |
| OS-KER-072 | Landlock 网络规则为可选增强，缺失时回退默认拒绝                            | 110-security/02-landlock-sandbox.md     |
| OS-KER-073 | register\_ftrace\_function 须设 SAVE\_REGS\_IF\_SUPPORTED | 90-observability/01-ftrace-framework.md |
| OS-KER-074 | L1 白盒单元测试必须用 KUnit                                      | 80-testing/01-kunit-framework.md        |
| OS-KER-075 | KUnit 套件命名空间必须正交                                        | 80-testing/01-kunit-framework.md        |
| OS-KER-076 | built-in.a 用 ar cDPrST 确定顺序                             | 70-build/01-kbuild-system.md            |
| OS-KER-077 | 生成物规则用 if\_changed 系列                                   | 70-build/01-kbuild-system.md            |
| OS-KER-078 | scripts\_basic/syncconfig 先于 descending                 | 70-build/01-kbuild-system.md            |
| OS-KER-079 | Landlock 三 syscall 是稳定 ABI                              | 110-security/02-landlock-sandbox.md     |
| OS-KER-080 | 每个 Agent 必须施加独立 Landlock 域                              | 110-security/02-landlock-sandbox.md     |
| OS-KER-081 | no\_new\_privs 是 Agent 启动前必设标志                          | 110-security/02-landlock-sandbox.md     |
| OS-KER-082 | Agent tracepoint print fmt 须以 agent\_id=0x%04x 起始       | 90-observability/01-ftrace-framework.md |
| OS-KER-083 | kernel 须导出 Agent 路径函数符号至 kallsyms                       | 90-observability/01-ftrace-framework.md |
| OS-KER-084 | defconfig 须开 CONFIG\_KALLSYMS 等                         | 90-observability/01-ftrace-framework.md |
| OS-KER-085 | 禁止 KUnit 与 kselftest 混编                                 | 80-testing/02-kselftest.md              |
| OS-KER-222 | 内存毒化保护——POISON\_FREE/POISON\_INUSE/POISON\_END + SLUB RedZone 三层防御体系 | 10-coding-style/C\_Cpp\_coding\_style.md §8.5 |
| OS-KER-223 | kmemleak 泄漏检测标注——自定义分配器注册、静态单例 not\_leak、零开销条件编译 | 10-coding-style/C\_Cpp\_coding\_style.md §8.6 |
| OS-KER-224 | DEFINE\_FREE / \_\_free() 自动资源释放，编译器保证作用域退出清理 | 10-coding-style/C\_Cpp\_coding\_style.md §7.4 |
| OS-KER-225 | 校验优先于修改——所有输入校验/权限检查/边界检查置于副作用之前，校验失败零副作用零泄漏。源于 seL4 `decodeUntypedInvocation()` 工程实践 | 10-coding-style/C\_Cpp\_coding\_style.md §7.0 |
| OS-KER-226 | kref\_put release 回调必须通过 container_of 获取外层结构体，禁止直接传 kfree。源于 Linux 6.6 内核基线 kref.h L49-55 注释声明 | 10-coding-style/C\_Cpp\_coding\_style.md §9.0 |
| OS-KER-227 | kmem_cache 选型标准——固定大小 + 高频分配(>100次) + 需调试支持的对象必须使用专用 kmem_cache，通用 kmalloc 仅用于临时/一次性/变长分配 | 10-coding-style/C\_Cpp\_coding\_style.md §8.7 |
| OS-KER-228 | 长循环抢占点规则——预计耗时>1ms或迭代>1000次的循环体必须周期性调用 cond_resched()。源于 seL4 preemptionPoint() 模式 | 10-coding-style/C\_Cpp\_coding\_style.md §8.8 |
| OS-KER-229 | 编译期断言与函数属性——必须利用 BUILD_BUG_ON/static_assert/\_Static\_assert 将不变式检测前移到编译阶段；不返回函数必须标注 \_\_noreturn；头文件依赖最小化 | 10-coding-style/C\_Cpp\_coding\_style.md §6.5 |

### 3.6 历史冲突消解编号（086-155，2026-07-12 全量消解）

> 以下编号于 2026-07-12 全量消解时分配，替代各主题文档中私自重用的 001-052 编号。消解原则：每个技术点获得全局唯一编号，杜绝跨文档冲突。

#### 3.6.1 110-security/01-lsm-framework.md（086-089）

| 编号         | 规则                                             | 原编号        |
| ---------- | ---------------------------------------------- | ---------- |
| OS-KER-086 | security\_hook\_heads 由 \_\_ro\_after\_init 保护 | OS-KER-001 |
| OS-KER-087 | exclusive LSM 互斥语义不可绕过                         | OS-KER-002 |
| OS-KER-088 | CONFIG\_LSM 顺序编译期固化，运行时不可追加                    | OS-KER-003 |
| OS-KER-089 | MicroCoreRT 锁定 Cupolas 钩子白名单                   | OS-KER-004 |

#### 3.6.2 70-build/01-kbuild-system.md（090-093）

| 编号         | 规则                            | 原编号        |
| ---------- | ----------------------------- | ---------- |
| OS-KER-090 | 顶层 Kbuild descending 顺序与基线一致  | OS-KER-001 |
| OS-KER-091 | prepare 阶段先于 descending       | OS-KER-002 |
| OS-KER-092 | obj-\* 用 obj-$(CONFIG\_\*) 门控 | OS-KER-003 |
| OS-KER-093 | 子系统 Makefile 含 SPDX 标识        | OS-KER-004 |

#### 3.6.3 80-testing/01-kunit-framework.md（094-097）

| 编号         | 规则                                  | 原编号        |
| ---------- | ----------------------------------- | ---------- |
| OS-KER-094 | 测试函数禁止直接访问 struct kunit 私有字段        | OS-KER-001 |
| OS-KER-095 | 扩展套件以 airy\_\* 前缀命名                 | OS-KER-002 |
| OS-KER-096 | 禁止默认开启 CONFIG\_KUNIT\_EXAMPLE\_TEST | OS-KER-003 |
| OS-KER-097 | 测试文件与被测源码同目录同名加 \_test 后缀           | OS-KER-004 |

#### 3.6.4 80-testing/02-kselftest.md（098-100）

| 编号         | 规则                       | 原编号        |
| ---------- | ------------------------ | ---------- |
| OS-KER-098 | 子目录禁止引入对上游测试集源码的依赖       | OS-KER-010 |
| OS-KER-099 | L2 系统级用例必须用 kselftest 实现 | OS-KER-011 |
| OS-KER-100 | kselftest 子目录命名空间必须正交    | OS-KER-012 |

#### 3.6.5 90-observability/01-ftrace-framework.md（101-105）

| 编号         | 规则                                        | 原编号        |
| ---------- | ----------------------------------------- | ---------- |
| OS-KER-101 | defconfig 必须开启 CONFIG\_FUNCTION\_TRACER 等 | OS-KER-001 |
| OS-KER-102 | init 阶段完成 tracefs 挂载，失败 panic             | OS-KER-002 |
| OS-KER-103 | ring buffer 默认大小 1408 KB/CPU              | OS-KER-003 |
| OS-KER-104 | 为 Agent 核心路径打 tracepoint                  | OS-KER-004 |
| OS-KER-105 | Agent 跟踪代码体积 < 8 KB                       | OS-KER-010 |

#### 3.6.6 90-observability/02-ebpf-probes.md（106-117）

| 编号         | 规则                                                  | 原编号        |
| ---------- | --------------------------------------------------- | ---------- |
| OS-KER-106 | defconfig 须开 CONFIG\_BPF 等 5 项                      | OS-KER-011 |
| OS-KER-107 | 须启用 CONFIG\_BPF\_JIT\_ALWAYS\_ON                    | OS-KER-012 |
| OS-KER-108 | 为 Agent 核心结构导出 BTF                                  | OS-KER-013 |
| OS-KER-109 | 注册 bpf\_agent\_decision\_get 等 kfunc                | OS-KER-014 |
| OS-KER-110 | 须开 CONFIG\_BPF\_VERIFIER\_STATE\_MARK               | OS-KER-015 |
| OS-KER-111 | Agent 核心路径函数支持 kprobe                               | OS-KER-016 |
| OS-KER-112 | 须支持 bpf(BPF\_LINK\_CREATE)                          | OS-KER-017 |
| OS-KER-113 | /sys/kernel/agentrt/token\_usage 导出 per-Agent Token | OS-KER-018 |
| OS-KER-114 | 四个函数符号导出至 kallsyms                                  | OS-KER-019 |
| OS-KER-115 | BPF 程序 .o 体积 < 64 KB，指令数 < 1 万                      | OS-KER-020 |
| OS-KER-116 | register\_bpf\_struct\_ops 注册 sched\_agent\_ops     | OS-KER-021 |
| OS-KER-117 | SCHED\_AGENT fallback 机制                            | OS-KER-022 |

#### 3.6.7 50-engineering-standards/01-coding-standards.md Part III（118-148）

| 编号         | 规则                          | 原编号        |
| ---------- | --------------------------- | ---------- |
| OS-KER-118 | 禁止"未来扩展"式抽象                 | OS-KER-022 |
| OS-KER-119 | 不维护跨平台 HAL（E-4 原则）          | OS-KER-023 |
| OS-KER-120 | 内核只提供能力，策略外移                | OS-KER-024 |
| OS-KER-121 | 策略可在不重编译内核下替换               | OS-KER-025 |
| OS-KER-122 | SCHED\_AGENT 基于 sched\_ext  | OS-KER-026 |
| OS-KER-123 | 函数长度一屏可读                    | OS-KER-027 |
| OS-KER-124 | 相信编译器，不写 \_\_always\_inline | OS-KER-028 |
| OS-KER-125 | 禁止 .c 内 #ifdef 切割函数体        | OS-KER-029 |
| OS-KER-126 | IS\_ENABLED 优先于 #ifdef      | OS-KER-030 |
| OS-KER-127 | 引用计数强制                      | OS-KER-031 |
| OS-KER-128 | 锁与引用计数职责不同                  | OS-KER-032 |
| OS-KER-129 | C99 灵活数组成员                  | OS-KER-033 |
| OS-KER-130 | bool 合并为 bitfield           | OS-KER-034 |
| OS-KER-131 | 内核内部 API 不保证稳定              | OS-KER-035 |
| OS-KER-132 | 用户空间 ABI 永久稳定               | OS-KER-036 |
| OS-KER-133 | 抽象有成本                       | OS-KER-037 |
| OS-KER-134 | 错误路径必须有测试                   | OS-KER-038 |
| OS-KER-135 | 用户可控长度参数须溢出检查               | OS-KER-039 |
| OS-KER-136 | 错误路径分级标签                    | OS-KER-040 |
| OS-KER-137 | WARN\_ON\_ONCE 禁止 WARN\_ON  | OS-KER-041 |
| OS-KER-138 | >3 行函数不应 inline             | OS-KER-042 |
| OS-KER-139 | 数据结构 cache line 对齐          | OS-KER-043 |
| OS-KER-140 | sizeof(\*p) 强制              | OS-KER-044 |
| OS-KER-141 | 禁止单独 printk，用分级宏            | OS-KER-045 |
| OS-KER-142 | 锁设计在数据结构阶段完成                | OS-KER-046 |
| OS-KER-143 | 热路径必须用 C 实现                 | OS-KER-047 |
| OS-KER-144 | Rust 仅用于安全敏感非热路径            | OS-KER-048 |
| OS-KER-145 | -Wall -Wextra -Werror       | OS-KER-049 |
| OS-KER-146 | make C=1 sparse 检查          | OS-KER-050 |
| OS-KER-147 | CONFIG\_LOCKDEP 强制          | OS-KER-051 |
| OS-KER-148 | fault injection 框架          | OS-KER-052 |

#### 3.6.8 60-driver-model（149-151）

| 编号         | 规则                                     | 权威定义文档                                | 原编号        |
| ---------- | -------------------------------------- | ------------------------------------- | ---------- |
| OS-KER-149 | device/driver/bus 模型不得引入用户态 daemon 强依赖 | 60-driver-model/01-device-model.md    | OS-KER-030 |
| OS-KER-150 | platform driver remove 回调通知用户态 daemon  | 60-driver-model/02-platform-driver.md | OS-KER-040 |
| OS-KER-151 | probe 中 printk 级别限制                    | 60-driver-model/02-platform-driver.md | OS-KER-041 |

#### 3.6.9 70-build/02-kconfig-system.md（155）

| 编号         | 规则                           | 原编号        |
| ---------- | ---------------------------- | ---------- |
| OS-KER-155 | config 名字必须以 AIRY\_ 或子系统前缀开头 | OS-KER-021 |

### 3.7 120-development-process 独立编号（211, 221）

| 编号         | 规则                                         | 权威定义文档                                             |
| ---------- | ------------------------------------------ | -------------------------------------------------- |
| OS-KER-211 | kernel 子仓 ABI 改动须知会 system 子仓维护者           | 120-development-process/02-maintainer-hierarchy.md |
| OS-KER-221 | AgentsIPC/MicroCoreRT 补丁须顶级维护者 Reviewed-by | 120-development-process/02-maintainer-hierarchy.md |

***

## 第 4 章 OS-STD 标准规则

### 4.1 OS-STD-CODE 编码规范

> **权威定义文档**：01-coding-standards.md

| 编号              | 规则                                                                                                                                                  |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| OS-STD-CODE-001 | 全局函数/变量描述性命名，禁止无意义缩写                                                                                                                                |
| OS-STD-CODE-002 | goto 集中出口，分级标签按分配逆序释放                                                                                                                               |
| OS-STD-CODE-003 | 禁止 strcpy/strncpy/strlcpy，强制 strscpy                                                                                                                |
| OS-STD-CODE-004 | 编译期必须 0 警告 0 错误                                                                                                                                     |
| OS-STD-CODE-005 | 代码重复 3 处以上应抽出到 lib/ 或 commons/ 子模块                                                                                                                  |
| OS-STD-CODE-006 | agentrt-linux 4 层接口稳定性分级（内核内部 / 用户 ABI / \[SC] / \[SS]）                                                                                             |
| OS-STD-CODE-007 | 跨语言 FFI 边界必须显式声明（extern "C" / ctypes / N-API）                                                                                                       |
| OS-STD-CODE-020 | typedef 仅 5 种例外（禁止结构体 typedef；原 OS-KER-013，2026-07-12 迁移）。权威定义：01-coding-standards.md §6.1 + C\_Cpp\_coding\_style.md §8 + checkpatch NEW\_TYPEDEFS |

### 4.2 OS-STD-FMT 格式规范

> **权威定义文档**：01-coding-standards.md Part II（30 条：FMT-001\~030）
>
> **登记说明（2026-07-15 补全）**：原 §4.2 仅登记 5 条代表性编号，OS-STD-FMT-001\~029 未显式登记，违反 SSoT §1.2"所有编号的分配权仅属于本注册表"。本次补全全量 30 条编号登记，消除 P0 级 SSoT 违规。完整描述见 [01-coding-standards.md Part II §0.2.1 迁移映射表](./01-coding-standards.md) 及正文 §1-§8。

| 编号             | 规则                                           | 原编号                   | 章节                          |
| -------------- | -------------------------------------------- | --------------------- | --------------------------- |
| OS-STD-FMT-001 | Tab 缩进，8 字符宽                                 | OS-KER-005            | Part II §1.1                |
| OS-STD-FMT-002 | C 代码首选 80 列                                  | OS-KER-004            | Part II §2.1                |
| OS-STD-FMT-003 | 非函数语句块开括号在行末（K\&R）                           | OS-STD-047            | Part II §3.1                |
| OS-STD-FMT-004 | 单语句与多语句的大括号规则                                | OS-KER-009            | Part II §3.4                |
| OS-STD-FMT-005 | 空格规则（关键字后空格 / sizeof 无空格 / 运算符空格 / 括号内禁空格）   | OS-KER-010/011/014 合并 | Part II §4.1/§4.2/§4.4/§4.5 |
| OS-STD-FMT-006 | 指针对齐（指针声明 `*` 贴名字不贴类型）                       | OS-KER-012            | Part II §4.3                |
| OS-STD-FMT-007 | 行尾禁止空白                                       | OS-KER-003            | Part II §1.6                |
| OS-STD-FMT-008 | 用户可见字符串禁止断行                                  | OS-KER-005（双义消解）      | Part II §2.2                |
| OS-STD-FMT-009 | 函数例外：开括号在下一行行首                               | OS-KER-008            | Part II §3.2                |
| OS-STD-FMT-010 | case 标签与 switch 对齐                           | OS-KER-015            | Part II §5.1                |
| OS-STD-FMT-011 | fallthrough 必须显式标注                           | OS-KER-016            | Part II §5.2                |
| OS-STD-FMT-012 | 函数之间用 1 个空行分隔                                | OS-KER-017            | Part II §6.1                |
| OS-STD-FMT-013 | 禁止函数内多余空行                                    | OS-KER-018            | Part II §6.2                |
| OS-STD-FMT-014 | 禁止用逗号逃避大括号 / 禁止单行多赋值                         | OS-KER-020/021 合并     | Part II §7.2/§7.3           |
| OS-STD-FMT-015 | 后继行应大幅短于父行                                   | OS-KER-006            | Part II §2.4                |
| OS-STD-FMT-016 | Kconfig Tab 缩进（DANGEROUS 标注）                 | OS-KER-002（02 §1.5）   | Part II §1.5                |
| OS-STD-FMT-017 | （预留/待确认）                                     | —                     | —                           |
| OS-STD-FMT-018 | （预留/待确认）                                     | —                     | —                           |
| OS-STD-FMT-019 | （预留/待确认）                                     | —                     | —                           |
| OS-STD-FMT-020 | 多语言缩进规则（Python/Rust/TS）                      | OS-STD-202            | Part II §1.2-§1.4           |
| OS-STD-FMT-021 | 多语言缩进规则（续）                                   | OS-STD-203            | Part II §1.2-§1.4           |
| OS-STD-FMT-022 | 多语言行宽规则                                      | OS-STD-204            | Part II §2.3                |
| OS-STD-FMT-023 | 多语言行宽规则（续）                                   | OS-STD-205            | Part II §2.3                |
| OS-STD-FMT-024 | 连续空行最多保留 1 行（注册表别名，C 语言 SSoT 定义为 OS-STD-206） | OS-STD-206            | Part II §6.3                |
| OS-STD-FMT-025 | 配置即代码工具链维护（clang-format）                     | OS-STD-207            | Part II §8.1                |
| OS-STD-FMT-026 | 配置即代码工具链维护（续）                                | OS-STD-208            | Part II §8.2                |
| OS-STD-FMT-027 | 配置即代码工具链维护（续）                                | OS-STD-209            | Part II §8.3                |
| OS-STD-FMT-028 | 配置即代码工具链维护（续）                                | OS-STD-210            | Part II §8.4                |
| OS-STD-FMT-029 | 配置即代码工具链维护（续）                                | OS-STD-211            | Part II §8.5                |
| OS-STD-FMT-030 | 配置即代码——每条格式规则必须有对应格式化工具配置项                   | OS-STD-201            | Part II §0                  |

> **历史遗留裸编号说明**：以下裸编号（无 FMT/CODE 子域前缀）为历史遗留，保留以兼容现有引用：
>
> | 编号              | 规则                                                                                                                                                                                                                      | 归属说明                                                                                                                                         |
> | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
> | OS-STD-010      | 内核态禁 float，强制 airy\_q16\_t（2026-07-12 从 OS-KER-007 迁移）                                                                                                                                                                  | 语义属 CODE 子域，保留裸编号兼容                                                                                                                          |
> | OS-STD-011      | bool 仅用于返回值与栈变量                                                                                                                                                                                                         | 语义属 CODE 子域，保留裸编号兼容。**修正（2026-07-15）**：原记录"从 OS-KER-008 迁移"错误——OS-KER-008 迁移至 OS-STD-FMT-009（K\&R 函数开括号），非 OS-STD-011。OS-STD-011 为独立规则，无迁移源。 |
> | OS-STD-CODE-031 | snake\_case 命名规则（函数名/变量名 snake\_case，常量宏 UPPER\_SNAKE\_CASE，结构体不 typedef）—— 10-coding-style/C\_Cpp\_coding\_style.md §3.2 权威定义                                                                                          | 属 CODE 子域，登记于此兼容历史引用                                                                                                                         |
> | OS-STD-CODE-032 | 内核内部用 u8/u32 及有符号变体；UAPI 结构体用 \_\_u8/\_\_u32（2026-07-15 从 OS-STD-CODE-031 让位消解——OS-STD-CODE-031 被 C\_Cpp\_coding\_style.md §3.2 snake\_case 命名规则占用，整数类型选择规则让位至 OS-STD-CODE-032；OS-STD-CODE-0xx 系列避免子系统 OS-STD-0xx 跨域冲突） | 属 CODE 子域，登记于此兼容历史引用                                                                                                                         |

### 4.3 OS-STD-STY 风格规范

> **权威定义文档**：01-coding-standards.md Part III §8.1-§8.5（OS-STD-STY-001\~004 定义于 §8.1-§8.4；OS-STD-STY-005\~012 品味哲学定义于 §8.5，源自 Linux 6.6 内核基线 源码深度提炼）

| 编号             | 规则                                                                      |
| -------------- | ----------------------------------------------------------------------- |
| OS-STD-STY-001 | 反过度抽象，不预先抽象                                                             |
| OS-STD-STY-002 | 引用计数强制，锁不替代引用计数                                                         |
| OS-STD-STY-003 | agentrt-linux 分层职责：热路径 C / 慢路径 Rust / Agent Python+TS                   |
| OS-STD-STY-004 | Agent 工作负载考量——Token 能效与记忆卷载（L1-L4 分层卷载）                                 |
| OS-STD-STY-005 | 信任读者——暴露而非封装（Linux 6.6 内核基线 `mutex.c` L60-73 owner 位编码）                        |
| OS-STD-STY-006 | 短函数——一屏可读（≤80×24），强制逻辑分解（Linux 6.6 内核基线 `mutex.c` L845）                        |
| OS-STD-STY-007 | 简洁文件头——SPDX 优先，禁止冗长版权声明块（Linux 6.6 内核基线 `mutex.c` L1-20）                       |
| OS-STD-STY-008 | EXPORT\_SYMBOL 紧跟定义——导出即声明（Linux 6.6 内核基线 `mutex.c` L46-58）                    |
| OS-STD-STY-009 | include 语义分组——禁止字母序自动排序，checkpatch 不检查字母序（Linux 6.6 内核基线 `mutex.c` L21-35）     |
| OS-STD-STY-010 | `__` 前缀标注内部接口——C 无访问控制下的约定（Linux 6.6 内核基线 `mutex.c` L80 `__mutex_owner`）       |
| OS-STD-STY-011 | `__read_mostly` 缓存优化——mostly-read 数据分离段（Linux 6.6 内核基线 `core.c` L168/178/7786） |
| OS-STD-STY-012 | `__randomize_layout` 结构体布局随机化——安全敏感结构体加固（Linux 6.6 内核基线 `sched.h` L658）        |

### 4.4 OS-STD-GOV 治理规范

> **权威定义文档**：07-maintainers-and-governance.md

| 编号             | 规则                                                       |
| -------------- | -------------------------------------------------------- |
| OS-STD-GOV-001 | MAINTAINERS 文件为子系统归属单一事实源                                |
| OS-STD-GOV-002 | 每子仓须有主维护者与至少 1 名审查者                                      |
| OS-STD-GOV-003 | 状态字段须如实标注                                                |
| OS-STD-GOV-004 | Co-developed-by 必须紧跟对应 Signed-off-by                     |
| OS-STD-GOV-005 | 审查意见必须逐条响应                                               |
| OS-STD-GOV-006 | 不响应审查是致命错误                                               |
| OS-STD-GOV-007 | Andrew Morton 规则：未导致改动的审查意见转化为代码注释                       |
| OS-STD-GOV-008 | 信任链分层不可越级提交                                              |
| OS-STD-GOV-009 | 组织 6 级成熟度 Level 2 为底线目标                                  |
| OS-STD-GOV-010 | 涉及 MicroCoreRT 内核适配的补丁，必须额外经 system 子仓顶级维护者会签，因其影响同源 ABI |

### 4.5 OS-STD-RUST Rust 编码规范

> **权威定义文档**：`10-coding-style/Rust_coding_style.md`（Part I 风格 + Part II 安全编码 + Part III 驱动开发）

| 编号         | 规则                                         | 所在 Part       |
| ---------- | ------------------------------------------ | ------------- |
| OS-STD-030 | 所有权模型——每个资源唯一所有者，move 语义                   | Part I §2.1   |
| OS-STD-031 | 借用而非克隆——优先 `&T`/`&mut T`                   | Part I §2.2   |
| OS-STD-032 | 生命周期显式标注                                   | Part I §2.3   |
| OS-STD-033 | snake\_case 命名约定                           | Part I §3.1   |
| OS-STD-034 | `airy_` 前缀隔离                               | Part I §3.2   |
| OS-STD-035 | 模块命名与文件名一致                                 | Part I §3.3   |
| OS-STD-036 | 禁止 panic/unwrap/expect                     | Part I §5.1   |
| OS-STD-037 | `?` 运算符错误传播                                | Part I §5.2   |
| OS-STD-038 | `Option<T>` 代替哨兵值                          | Part I §5.3   |
| OS-STD-039 | 使用 kernel crate 安全抽象                       | Part I §6.1   |
| OS-STD-040 | KBox/KVec 内存分配                             | Part I §6.2   |
| OS-STD-041 | kernel::sync 同步原语                          | Part I §6.3   |
| OS-STD-042 | Pin<T> 不可移动保证                              | Part I §7.1   |
| OS-STD-043 | pin\_init! 宏初始化                            | Part I §7.2   |
| OS-STD-044 | extern "C" FFI 边界                          | Part I §8.1   |
| OS-STD-045 | FFI 类型映射——`core::ffi`/`kernel::ffi` C 兼容类型 | Part I §8.2   |
| OS-STD-046 | FFI 所有权语义——`#[ownership]` 标注               | Part I §8.3   |
| OS-STD-050 | 驱动注册模型（kernel::module! 宏）                  | Part III §1.1 |
| OS-STD-051 | 驱动 Trait 约定                                | Part III §1.2 |
| OS-STD-052 | 驱动 unsafe 审计标准（SAFETY 注释）                  | Part III §2.1 |
| OS-STD-053 | 硬件 MMIO 寄存器安全访问                            | Part III §2.2 |
| OS-STD-054 | DMA 内存安全（DmaObject）                        | Part III §2.3 |
| OS-STD-055 | Kbuild Rust 集成                             | Part III §3.1 |
| OS-STD-056 | Kconfig 配置（CONFIG\_AIRY\_\*\_RUST）         | Part III §3.2 |
| OS-STD-057 | KUnit Rust 驱动测试                            | Part III §4.1 |

### 4.6 OS-STD-TOOL 工具链规则编号段声明

> **权威定义文档**：`05-development-process.md` Part II（§0.2 声明历史 `OS-STD-101~158` 迁移至 `OS-STD-TOOL-101~158`，本注册表登记编号段归属）

`OS-STD-TOOL-NNN`（TOOL = Toolchain）编号段承载 7 层自动化验证体系的工具链规则（编译期/静态分析/预提交/CI 门禁/连接验证/协议验证/发布验证）。本编号段与 `OS-STD-PROD-NNN`（PROD = Process/Development，权威定义于 `05-development-process.md`）隔离，消除历史上 `OS-STD-101~158` 在两卷中的语义冲突。

**编号段归属**：

| 编号段                  | 权威定义文档                            | 内容范围                     | 7 层验证对应 |
| -------------------- | --------------------------------- | ------------------------ | ------- |
| OS-STD-TOOL-001\~099 | 05-development-process.md Part II | 通用工具链规则（编译/格式/检查）        | 第 1-3 层 |
| OS-STD-TOOL-101\~158 | 05-development-process.md Part II | CI/CD 矩阵 + 覆盖率门槛 + 发布流水线 | 第 4-7 层 |
| OS-STD-PROD-001\~099 | 05-development-process.md         | 开发流程规则（补丁生命周期/审查响应）      | 流程层     |
| OS-STD-PROD-101\~158 | 05-development-process.md         | 开发流程高级规则（RFC/早期审查/维护者响应） | 流程层     |

**关键规则登记**：

| 编号                  | 规则                                                 | 权威定义                     |
| ------------------- | -------------------------------------------------- | ------------------------ |
| OS-STD-TOOL-081     | sparse `make C=2` 检查 + W 分级体系（W=1 本地 / W=2 CI）     | 06 §3.1 + §3.1.1         |
| OS-STD-TOOL-081-CI  | CI 强制 `make W=2 C=2` 检查新增代码                        | 06 §3.1.1                |
| OS-STD-TOOL-081-REL | 发布前 `make W=2 C=2` 全量审计                            | 06 §3.1.1                |
| OS-STD-TOOL-121     | 内核子系统行覆盖率 ≥80%                                     | 06 §7.3                  |
| OS-STD-TOOL-122     | Agent SDK 行覆盖率 ≥80%                                | 06 §7.3                  |
| OS-STD-TOOL-124     | 关键路径覆盖率 100%                                       | 06 §7.4                  |
| OS-STD-TOOL-160     | checkpatch.pl --strict 每次提交前必须通过                   | 01-coding-standards §9.5 |
| OS-STD-TOOL-161     | 多配置构建（defconfig + allnoconfig + allmodconfig 三者皆绿） | 01-coding-standards §9.6 |

> **历史编号等价声明**：05-development-process.md Part II 正文中现存的历史编号 `OS-STD-002~158` 与目标 `OS-STD-TOOL-002~158` 等价有效，规则效力以 Part II 正文为准。CI/CD 脚本与 GitHub Actions workflow 实施时统一替换为 `OS-STD-TOOL-NNN`。

### 4.6.1 OS-STD-PROD 开发流程规则登记

> **权威定义文档**：`05-development-process.md`
>
> **迁移说明（2026-07-14）**：原 `05-development-process.md` 中的裸 `OS-STD-2xx` 编号（与 `01-coding-standards.md` 的 `OS-STD-2xx` 冲突）已全部迁移至 `OS-STD-PROD-NNN` 子域编号，消除编号三重冲突。

| 编号              | 规则                                                      | 权威定义     |
| --------------- | ------------------------------------------------------- | -------- |
| OS-STD-PROD-001 | LTS 版本必须每季度发布一个维护版本                                     | 05 §7.3  |
| OS-STD-PROD-002 | LTS 维护者离职需提前 6 个月通知                                     | 05 §7.3  |
| OS-STD-PROD-011 | 从 L1 晋升 L2 需至少 5 个被合并的 PR，且无 regression                 | 05 §8.3  |
| OS-STD-PROD-012 | 从 L2 晋升 L3 需至少 20 个被合并的 PR + 维护者面试                      | 05 §8.3  |
| OS-STD-PROD-021 | 每个子仓必须维护一份 MAINTAINERS.md                               | 05 §9.3  |
| OS-STD-PROD-031 | 所有 commit 必须用 `git commit -s` 添加 DCO 签名                 | 05 §10.1 |
| OS-STD-PROD-032 | 禁止 `git push --force` 到 `main`/`develop`/`release/*` 分支 | 05 §10.1 |
| OS-STD-PROD-033 | 所有 PR 必须通过 GitHub Actions 全部检查才能合并                      | 05 §10.2 |
| OS-STD-PROD-034 | PR 状态变更须同步到 GitHub Projects 看板                          | 05 §10.3 |

### 4.6 族索引登记（已废除）

> **废除说明**（2026-07-12 SSoT 重新设计）：原 9 个族索引文件（200/201 根目录 + 200-206 子目录）已物理删除。族索引模式被证明是"文档越来越多"的根源之一——9 个文件 \~1,260 行仅是导航链接堆叠，无独立规则定义价值。所有规则编号登记统一由本注册表 §2-§7 承载，不再通过族索引中间层导航。

### 4.7 子目录 README SSoT 依赖登记

> 以下 5 个子目录 README 已在文件头部追加 SSoT 依赖声明，声明其规则编号权威为 09-ssot-registry.md §3。子目录 README 本身不定义新规则编号，仅作为子目录内文件的索引入口。（族索引已废除，2026-07-12 SSoT 重新设计）

| 子目录 README                        | SSoT 依赖声明内容 | 特定权威引用                                           | 登记日期       |
| --------------------------------- | ----------- | ------------------------------------------------ | ---------- |
| `10-coding-style/README.md`       | 编号权威 09 §3  | coding\_conventions.md Part IV §4.4（模块前缀）        | 2026-07-12 |
| `20-contracts/README.md`          | 编号权威 09 §3  | contracts.md Part III（日志 SSoT）                   | 2026-07-12 |
| `30-runtime-interfaces/README.md` | 编号权威 09 §3  | coding\_conventions.md Part IV §1.10（`are_*` 前缀） | 2026-07-12 |
| `40-integration/README.md`        | 编号权威 09 §3  | IRON-9 v3 四层模型（\[SC]/\[SS]/\[IND]）               | 2026-07-12 |
| `50-project-erp/README.md`        | 编号权威 09 §3  | IRON-9 v3 工程铁律                                   | 2026-07-12 |

### 4.8 OS-FMT 闭源总纲代码格式规则

> **权威定义文档**：闭源总纲（`agentrt-linux基本工程标准规范.md`）§3 代码格式
>
> **来源说明**：OS-FMT 为闭源总纲定义的格式子域前缀（FMT = Format），承载 C 内核代码格式规则。本节将其登记到 SSoT 注册表以建立跨开源/闭源的编号一致性。注意：本节 OS-FMT-001\~007（闭源总纲 §3）与 §4.2 OS-STD-FMT（开源 `01-coding-standards.md` Part II，29 条）为两套独立编号系列——前者源自闭源总纲，后者源自开源编码标准；二者内容有重叠但编号体系不同，引用时须明确区分。

| 编号         | 规则                                                              | 章节定位      |
| ---------- | --------------------------------------------------------------- | --------- |
| OS-FMT-001 | C 内核代码使用 Tab 缩进，Tab 宽度为 8 字符                                    | 闭源总纲 §3.1 |
| OS-FMT-002 | C 代码首选 80 列行宽                                                   | 闭源总纲 §3.2 |
| OS-FMT-003 | 使用 K\&R 风格大括号（非函数块左括号放行末，函数定义左括号新起一行）                           | 闭源总纲 §3.3 |
| OS-FMT-004 | 即使只有一条语句，也使用大括号包围                                               | 闭源总纲 §3.3 |
| OS-FMT-005 | 空格规则（关键字后加空格 / 括号内禁止空格 / 二元运算符两侧加空格 / 一元运算符不加空格 / 指针声明 `*` 贴名字） | 闭源总纲 §3.4 |
| OS-FMT-006 | 行尾禁止空白字符                                                        | 闭源总纲 §3.5 |
| OS-FMT-007 | 所有 C/C++ 代码必须通过 `.clang-format` 配置自动格式化                         | 闭源总纲 §3.6 |

> **历史关联**：OS-FMT-001（Tab 8 字符缩进）原属 OS-KER-005，因格式规则重新组织而迁移；OS-KER-005 已标记为已废弃（见 §3.1）。

### 4.9 OS-STD-DOC 文档规范（kernel-doc）

> **权威定义文档**：`10-coding-style/01-coding-standards.md` Part IV（kernel-doc 规范）
>
> **登记说明（2026-07-15 补全）**：原 `01-coding-standards.md` Part IV 中使用的 `OS-STD-DOC-001~014` 编号未在 SSoT 登记，违反 SSoT §1.2"所有编号的分配权仅属于本注册表"。本次补全全量 14 条编号登记，消除 P0 级 SSoT 违规。

| 编号             | 规则                       | 章节           |
| -------------- | ------------------------ | ------------ |
| OS-STD-DOC-001 | `/** */` 起始标记强制          | Part IV §1.1 |
| OS-STD-DOC-002 | 短描述（short description）强制 | Part IV §1.2 |
| OS-STD-DOC-003 | 函数 kernel-doc 字段顺序       | Part IV §2.1 |
| OS-STD-DOC-004 | `@arg:` 参数描述强制           | Part IV §2.2 |
| OS-STD-DOC-005 | `Return:` 返回值强制          | Part IV §2.3 |
| OS-STD-DOC-006 | `Context:` 上下文约束强制       | Part IV §2.4 |
| OS-STD-DOC-007 | struct kernel-doc 字段顺序   | Part IV §3.1 |
| OS-STD-DOC-008 | 嵌套成员引用                   | Part IV §3.2 |
| OS-STD-DOC-009 | enum kernel-doc 字段顺序     | Part IV §4.1 |
| OS-STD-DOC-010 | 函数指针参数描述                 | Part IV §5.1 |
| OS-STD-DOC-011 | 仅允许 typedef 的 kernel-doc | Part IV §6.1 |
| OS-STD-DOC-012 | 交叉引用标记                   | Part IV §7.1 |
| OS-STD-DOC-013 | 文件级 kernel-doc 注释        | Part IV §8.1 |
| OS-STD-DOC-014 | kernel-doc CI 校验         | Part IV §9.1 |

### 4.10 OS-STD-SEC 安全编码规范

> **权威定义文档**：`110-security/02-landlock-sandbox.md`
>
> **登记说明（2026-07-15 补全）**：原 `02-landlock-sandbox.md` 中使用的 `OS-STD-SEC-010~011` 编号未在 SSoT 登记，违反 SSoT §1.2。本次补全登记。
>
> **编号空间隔离声明**：OS-STD-SEC-NNN（本节，安全编码规范子域）与 OS-SEC-NNN（§9 OS-SEC 安全工程规则）属于不同编号空间，禁止混用。OS-STD-SEC 覆盖 Landlock 沙箱编码规范；OS-SEC 覆盖安全工程全局规则。

| 编号             | 规则                                               | 章节                       |
| -------------- | ------------------------------------------------ | ------------------------ |
| OS-STD-SEC-010 | `is_initialized()` 失败时必须返回 `-EOPNOTSUPP`，清晰告知用户态 | 02-landlock-sandbox §A-1 |
| OS-STD-SEC-011 | 沙箱施加失败必须中止 Agent 启动，禁止降级运行                       | 02-landlock-sandbox §A-4 |

### 4.11 OS-STD-SPDX 许可证标识规范

> **权威定义文档**：`05-development-process.md` Part IV（SPDX 许可证合规）
>
> **登记说明（2026-07-15 补全）**：原 `05-development-process.md` Part IV 中使用的 `OS-STD-SPDX-001~003` 编号未在 SSoT 登记，违反 SSoT §1.2。本次补全登记。

| 编号              | 规则           | 章节           |
| --------------- | ------------ | ------------ |
| OS-STD-SPDX-001 | 按文件类型分类使用许可证 | Part IV §1.1 |
| OS-STD-SPDX-002 | 注释样式正确性      | Part IV §3.1 |
| OS-STD-SPDX-003 | SPDX 标识必须在首行 | Part IV §3.2 |

### 4.12 OS-STD-TEST 测试规范

> **权威定义文档**：`80-testing/02-kselftest.md`
>
> **登记说明（2026-07-15 补全）**：原 `02-kselftest.md` 中使用的 `OS-STD-TEST-010~011` 编号未在 SSoT 登记，违反 SSoT §1.2。本次补全登记。
>
> **编号空间隔离声明**：OS-STD-TEST-NNN（本节，测试编码规范子域）与 OS-TEST-NNN（§12 OS-TEST 测试工程规则）属于不同编号空间，禁止混用。

| 编号              | 规则                                                                                  | 章节           |
| --------------- | ----------------------------------------------------------------------------------- | ------------ |
| OS-STD-TEST-010 | 每个 `TEST_F` 必须有对应 `FIXTURE_SETUP` 与 `FIXTURE_TEARDOWN`；资源在 setup 中获取，在 teardown 中释放 | 02-kselftest |
| OS-STD-TEST-011 | CI 总时长不得超过 60 分钟（与 06-toolchain-and-automation 的 OS-STD-033 对齐）                     | 02-kselftest |

***

## 第 5 章 OS-BAN 禁止规则

> **权威定义文档**：01-coding-standards.md

| 编号         | 规则                                          |
| ---------- | ------------------------------------------- |
| OS-BAN-001 | 禁止敏感术语 master/slave、blacklist/whitelist     |
| OS-BAN-002 | 禁止 BUG()/BUG\_ON()，改用 WARN\_ON\_ONCE()      |
| OS-BAN-003 | sizeof(\*p) / kmalloc\_array / struct\_size |
| OS-BAN-004 | 禁止 strcpy/strncpy/strlcpy，强制 strscpy        |
| OS-BAN-005 | 禁止 %p 默认输出（须哈希化）                            |
| OS-BAN-006 | 禁止匈牙利命名法                                    |
| OS-BAN-007 | 禁止函数声明 extern                               |
| OS-BAN-008 | 禁止自动排序 include                              |
| OS-BAN-009 | 禁止五类危险宏                                     |
| OS-BAN-010 | 开源文档禁词                                      |
| OS-BAN-011 | 禁词违规处罚                                      |

***

## 第 6 章 OS-ACC 验收标准

> **权威定义文档**：130-roadmap/06-acceptance-criteria.md
>
> **SSoT 对齐说明（v0.2.5 已消解）**：历史 00-handbook §3.7 与 07-governance §10 中定义的 OS-ACC-001\~005 与 130-roadmap 中的 OS-ACC 编号存在语义冲突（同一编号指代不同验收项）。现已完成消解：本注册表 §7 确认 130-roadmap/06-acceptance-criteria.md 为 OS-ACC 的唯一权威定义文档；00-handbook §3.7 已改为引用本注册表 §7（见 00-handbook L151）；07-governance §10.6 已改为引用本注册表 §7（见 07-governance L645）。OS-ACC 编号体系单源权威已建立。
>
> **摘要登记覆盖声明（2026-07-15 补全）**：OS-ACC-001\~110 全部 110 条验收标准采用摘要登记方式，覆盖范围见 §1.5 摘要登记覆盖声明表。权威定义为 `130-roadmap/06-acceptance-criteria.md`。本节不逐条列举，避免与权威定义文档内容重复；编号分配权属于本注册表，详细说明由权威定义文档承载。

***

## 第 7 章 OS-ABI 接口稳定性

> **权威定义文档**：04-engineering-philosophy.md §6.2

| 编号         | 规则                                             |
| ---------- | ---------------------------------------------- |
| OS-ABI-001 | 4 层接口稳定性分级：L1 永久 / L2 弃用流程 / L3 版本协商 / L4 完全自由 |

***

## 第 8 章 命名规范

> **权威定义文档**：10-coding-style/coding\_conventions.md Part IV §4.4

模块前缀（`are_` / `airy_` / `airy_core_` / `airy_` 及模块子前缀）的唯一权威来源为 coding\_conventions.md Part IV §4.4。本注册表不重复注册模块前缀；前缀注册、变更与新增须遵循 coding\_conventions.md Part IV §4.4.6 的 RFC 流程。（2026-07-13 更新：naming\_conventions.md 已合并入 coding\_conventions.md Part IV）

### 8.1 术语表登记

| 文件                  | 内容范围                               | 权威范围      | 迁移说明                                                   |
| ------------------- | ---------------------------------- | --------- | ------------------------------------------------------ |
| `90-terminology.md` | Airymax 全项目术语表（原 `TERMINOLOGY.md`） | 术语定义 SSoT | 2026-07-12 从 AirymaxOS 根目录移入 50-engineering-standards/ |

### 8.2 废弃文档登记

| 文件                               | 原内容                            | 当前状态                        | 重定向目标                                | 废弃日期       |
| -------------------------------- | ------------------------------ | --------------------------- | ------------------------------------ | ---------- |
| `15-license-strategy.md`         | 许可证策略（200 行）                   | 已物理删除（2026-07-12 SSoT 重新设计） | `05-development-process.md` Part IV  | 2026-07-12 |
| `02-code-format.md`              | 代码格式标准（547 行）                  | 已物理删除（2026-07-13 顶层文件合并）    | `01-coding-standards.md` Part II     | 2026-07-13 |
| `03-code-style.md`               | 代码风格标准（489 行）                  | 已物理删除（2026-07-13 顶层文件合并）    | `01-coding-standards.md` Part III    | 2026-07-13 |
| `60-checkpatch-rule-map.md`      | checkpatch 规则映射表（788 行）        | 已物理删除（2026-07-13 顶层文件合并）    | `01-coding-standards.md` Part IV     | 2026-07-13 |
| `70-kernel-doc-standard.md`      | kernel-doc 注释规范（552 行）         | 已物理删除（2026-07-13 顶层文件合并）    | `01-coding-standards.md` Part V      | 2026-07-13 |
| `80-clang-format-enforcement.md` | .clang-format 配置与 CI 门禁（592 行） | 已物理删除（2026-07-13 顶层文件合并）    | `01-coding-standards.md` Part VI     | 2026-07-13 |
| `06-toolchain-and-automation.md` | 工具链与自动化（764 行）                 | 已物理删除（2026-07-13 顶层文件合并）    | `05-development-process.md` Part II  | 2026-07-13 |
| `08-compliance-checklist.md`     | 规范符合性检查（756 行）                 | 已物理删除（2026-07-13 顶层文件合并）    | `05-development-process.md` Part III | 2026-07-13 |
| `110-spdx-license-compliance.md` | SPDX 许可证策略与合规（492 行）           | 已物理删除（2026-07-13 顶层文件合并）    | `05-development-process.md` Part IV  | 2026-07-13 |
| `100-deprecated-api-registry.md` | 废弃 API 动态清单（642 行）             | 已物理删除（2026-07-13 废弃文档清理）    | （无重定向，内容已废弃）                         | 2026-07-13 |

### 8.3 文档计数三作用域模型

> **登记目的**：工程文档在不同上下文中使用不同的文档计数口径，历史上有 \~79 / \~85 / \~122 三种数字并存且未在 SSoT 注册表中声明其作用域，导致计数混淆。特此登记三作用域模型以消除歧义。

| 计数       | 作用域   | 上下文               | 说明                              |
| -------- | ----- | ----------------- | ------------------------------- |
| \~79 文档  | P0 基线 | "文档体系完成"上下文       | P0 级文档基线，核心工程标准与架构文档集合          |
| \~85 文档  | 工程基线  | "0.1.1 合计/总产出"上下文 | 0.1.1 版本工程产出合计，含 P0 基线 + 扩展工程文档 |
| \~122 文档 | 全量展开  | "全部三大支柱"上下文       | 全量文档展开（全部三大支柱），含研究/分析/审查等辅助文档   |

> **使用约定**：引用文档计数时必须标注所属作用域（P0 基线 / 工程基线 / 全量展开），禁止脱离作用域上下文使用裸数字。

***

## 第 9 章 OS-SEC 安全工程规则

> **权威定义文档**：10-coding-style/C\_Cpp\_coding\_style.md Part III（C/C++ 安全编码）+ 10-coding-style/Rust\_coding\_style.md §4-§7（Rust 安全编码）+ 110-security/01-lsm-framework.md + 110-security/02-landlock-sandbox.md（LSM/Cupolas 安全规则）
>
> **编号空间分配（2026-07-14 消解三重冲突）**：历史 OS-SEC 编号在三个文档体系中有三套完全不同的定义（LSM 安全规则 / C 安全编码 / Rust 安全编码），导致同一编号指代完全不同的规则。本节消解此三重冲突，按编号段分配子域：
>
> | 编号段             | 子域               | 权威定义文档                            | 迁移偏移 |
> | --------------- | ---------------- | --------------------------------- | ---- |
> | OS-SEC-001\~099 | LSM/Cupolas 安全规则 | 110-security/                     | 不变   |
> | OS-SEC-100\~199 | C/C++ 安全编码规则     | C\_Cpp\_coding\_style.md Part III | +100 |
> | OS-SEC-200\~299 | Rust 安全编码规则      | Rust\_coding\_style.md §4-§7      | +200 |

### 9.1 LSM/Cupolas 安全规则（OS-SEC-001\~099）

> **权威定义文档**：110-security/01-lsm-framework.md + 110-security/02-landlock-sandbox.md

| 编号         | 规则                                                        | 权威定义                   |
| ---------- | --------------------------------------------------------- | ---------------------- |
| OS-SEC-001 | LSM 钩子内置每一关键路径，不可外挂补丁替代                                   | 01-lsm-framework.md    |
| OS-SEC-002 | 钩子链表 RCU 不变量需通过形式化检查                                      | 01-lsm-framework.md    |
| OS-SEC-003 | Cupolas 必须在 capability 之后初始化                              | 01-lsm-framework.md    |
| OS-SEC-004 | 沙箱能力内置内核，Agent 无需特权即可施加                                   | 02-landlock-sandbox.md |
| OS-SEC-005 | 域的不可逆性需通过形式化检查                                            | 02-landlock-sandbox.md |
| OS-SEC-006 | 沙箱施加前后必须通过 AgentsIPC 上报事件                                 | 02-landlock-sandbox.md |
| OS-SEC-007 | Landlock 域不可替代 capability，必须与之共存                          | 02-landlock-sandbox.md |
| OS-SEC-008 | Cupolas 钩子回调必须返回 \[SC] 4 值枚举之一（ALLOW/DENY/AUDIT/COMPLAIN）  | 01-lsm-framework.md    |
| OS-SEC-009 | COMPLAIN 裁决必须通过 A-IPC 询问 daemon，5 秒超时后回退 ALLOW 并记录 AUDIT | 01-lsm-framework.md    |
| OS-SEC-010 | 审计事件必须包含 agent\_id 字段，缺失 agent\_id 的事件必须丢弃并告警             | 01-lsm-framework.md    |
| OS-SEC-011 | Cupolas 7 子系统钩子映射必须与 \[SC] 钩子 ID 一一对应，新增子系统必须扩展映射表        | 01-lsm-framework.md    |
| OS-SEC-012 | Agent 沙箱必须至少 3 层域叠加（L0 全局 + L1 角色 + L2 实例），禁止少于 3 层       | 02-landlock-sandbox.md |
| OS-SEC-013 | 访问掩码合并必须用交集语义（final\_mask = L0 & L1 & L2），禁止并集或最宽优先       | 02-landlock-sandbox.md |
| OS-SEC-014 | landlock\_restrict\_self 必须在 Agent 进程 fork() 后立即调用，禁止延迟施加 | 02-landlock-sandbox.md |
| OS-SEC-015 | 沙箱创建失败必须终止 Agent 启动，禁止降级运行                                | 02-landlock-sandbox.md |
| OS-SEC-016 | v1.1: Capability Badge 校验由 fastpath C-S9 内联完成，LSM 钩子不在正常路径上重复执行 capability 校验 | 01-lsm-framework.md    |
| OS-SEC-017 | v1.1: LSM 钩子（`security_uring_cmd`）仅在 fastpath C-S9 失败时被调用，做策略裁决与冷酷执法 | 01-lsm-framework.md    |

### 9.2 C/C++ 安全编码规则（OS-SEC-100\~199）

> **权威定义文档**：C\_Cpp\_coding\_style.md Part III
> **迁移说明（2026-07-14）**：原 OS-SEC-010\~027 迁移至 OS-SEC-110\~127（+100 偏移），消除与 LSM 安全规则 OS-SEC-001\~099 的编号冲突。

| 编号         | 规则                                                           | 权威定义                 | 原编号        |
| ---------- | ------------------------------------------------------------ | -------------------- | ---------- |
| OS-SEC-110 | 边界检查原则——所有涉及用户可控长度参数的缓冲区操作必须进行边界检查                           | C\_Cpp Part III §1.1 | OS-SEC-010 |
| OS-SEC-111 | 安全内存复制函数——内核态与用户态之间使用 copy\_from\_user/copy\_to\_user        | C\_Cpp Part III §1.3 | OS-SEC-011 |
| OS-SEC-112 | 整数溢出检测——使用 check\_add\_overflow/check\_mul\_overflow 等溢出安全函数 | C\_Cpp Part III §2.1 | OS-SEC-012 |
| OS-SEC-113 | 类型转换安全——避免隐式类型转换导致的数据截断                                      | C\_Cpp Part III §2.2 | OS-SEC-013 |
| OS-SEC-114 | 符号问题——避免有符号/无符号混用                                            | C\_Cpp Part III §2.3 | OS-SEC-014 |
| OS-SEC-115 | 禁止用户可控格式化字符串                                                 | C\_Cpp Part III §3.1 | OS-SEC-015 |
| OS-SEC-116 | 格式化函数 \_\_printf 注解                                          | C\_Cpp Part III §3.2 | OS-SEC-016 |
| OS-SEC-117 | NULL 检查——外部指针在解引用前必须进行 NULL 检查                               | C\_Cpp Part III §4.1 | OS-SEC-017 |
| OS-SEC-118 | 悬挂指针防护——释放内存后立即置 NULL，使用 kfree\_sensitive                    | C\_Cpp Part III §4.2 | OS-SEC-018 |
| OS-SEC-119 | 数据竞争防护——共享数据必须通过同步原语保护，KCSAN 检测                              | C\_Cpp Part III §5.1 | OS-SEC-019 |
| OS-SEC-120 | 死锁预防——多锁获取遵循固定锁顺序                                            | C\_Cpp Part III §5.2 | OS-SEC-020 |
| OS-SEC-121 | capability 检查——安全敏感操作必须通过 Cupolas capability 系统检查            | C\_Cpp Part III §6.1 | OS-SEC-021 |
| OS-SEC-122 | capability 令牌管理——安全通道传递，禁止日志打印原始值                            | C\_Cpp Part III §6.2 | OS-SEC-022 |
| OS-SEC-123 | 所有用户态输入必须验证（长度/范围/类型/格式）                                     | C\_Cpp Part III §7.1 | OS-SEC-023 |
| OS-SEC-124 | 系统调用参数验证模板——UAPI 结构体字段验证不遗漏                                  | C\_Cpp Part III §7.2 | OS-SEC-024 |
| OS-SEC-125 | copy\_to\_user 敏感数据——确保不含内核地址/未初始化内存                         | C\_Cpp Part III §8.1 | OS-SEC-025 |
| OS-SEC-126 | 内核地址保护——禁止向用户态泄露内核地址                                         | C\_Cpp Part III §8.2 | OS-SEC-026 |
| OS-SEC-127 | 敏感数据清零——使用 memzero\_explicit/kfree\_sensitive                | C\_Cpp Part III §8.3 | OS-SEC-027 |

### 9.3 Rust 安全编码规则（OS-SEC-200\~299）

> **权威定义文档**：Rust\_coding\_style.md §4-§7
> **迁移说明（2026-07-14）**：原 OS-SEC-001\~003 迁移至 OS-SEC-201\~203，原 OS-SEC-030\~047 迁移至 OS-SEC-230\~247（+200 偏移），消除与 LSM 安全规则 OS-SEC-001\~099 的编号冲突。

| 编号         | 规则                                        | 权威定义      | 原编号        |
| ---------- | ----------------------------------------- | --------- | ---------- |
| OS-SEC-201 | unsafe 代码块最小化——仅包裹真正需要 unsafe 的操作         | Rust §4.1 | OS-SEC-001 |
| OS-SEC-202 | SAFETY 注释文档化——每个 unsafe 块包含 // SAFETY: 注释 | Rust §4.2 | OS-SEC-002 |
| OS-SEC-203 | unsafe 代码审查——至少两名资深维护者审查                  | Rust §4.3 | OS-SEC-003 |
| OS-SEC-230 | unsafe 审计五步法                              | Rust §1.2 | OS-SEC-030 |
| OS-SEC-231 | unsafe 代码审查清单                             | Rust §1.3 | OS-SEC-031 |
| OS-SEC-232 | 借用检查器利用——静态验证并发安全性                        | Rust §2.2 | OS-SEC-032 |
| OS-SEC-233 | 智能指针与 RAII——资源生命周期自动管理                    | Rust §2.3 | OS-SEC-033 |
| OS-SEC-234 | Send 与 Sync trait——跨线程传递/共享类型约束           | Rust §3.1 | OS-SEC-034 |
| OS-SEC-235 | Mutex 与 RwLock——编译器保证锁释放                  | Rust §3.2 | OS-SEC-035 |
| OS-SEC-236 | 原子操作——使用 AtomicU32/AtomicBool 替代锁         | Rust §3.3 | OS-SEC-036 |
| OS-SEC-237 | DMA 缓冲区安全——硬件直接访问不受 MMU 保护                | Rust §4.1 | OS-SEC-037 |
| OS-SEC-238 | MMIO 寄存器访问——通过 accessor 函数，禁止裸指针          | Rust §4.2 | OS-SEC-038 |
| OS-SEC-239 | 裸指针与内核链表——使用 kernel::list 安全封装            | Rust §4.3 | OS-SEC-039 |
| OS-SEC-240 | C 调用约定与 ABI 稳定性——extern "C" 声明            | Rust §5.1 | OS-SEC-040 |
| OS-SEC-241 | 类型布局兼容性——#\[repr(C, packed)] 确保与 C 布局兼容   | Rust §5.2 | OS-SEC-041 |
| OS-SEC-242 | 所有权穿越 FFI 边界——明确约定所有权转移                   | Rust §5.3 | OS-SEC-042 |
| OS-SEC-243 | cargo audit——CRITICAL 级漏洞阻断合并             | Rust §6.1 | OS-SEC-043 |
| OS-SEC-244 | 依赖审查——新增依赖必须审查内容/许可证/安全                   | Rust §6.2 | OS-SEC-044 |
| OS-SEC-245 | 最小依赖原则——能用标准库不引入外部 crate                  | Rust §6.3 | OS-SEC-045 |
| OS-SEC-246 | Kani 验证器——Rust 形式化验证                      | Rust §7.1 | OS-SEC-046 |
| OS-SEC-247 | Creusot 验证器——Rust 规范验证                    | Rust §7.2 | OS-SEC-047 |

### 9.4 跨子系统 OS-STD-0xx 编号登记（2026-07-15 消解跨域冲突）

> **背景**：原 OS-STD-0xx 编号被 110-security/60-driver-model/90-observability/05-development-process 多个子系统文档独立占用，造成跨域编号冲突。2026-07-15 消解：110-security 保留 OS-STD-001\~009 权威定义；05-development-process 保留 OS-STD-012 权威定义；60-driver-model 改用 OS-STD-DRV-0xx 系列；90-observability 改用 OS-STD-OBS-0xx 系列；OS-STD-017 编号管理元规则提升为 OS-IRON-015（因 OS-IRON-013 已被"8 子仓 submodule"占用而改用 OS-IRON-015）。

| 编号             | 规则                                                                                             | 权威定义文档                                                  |
| -------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| OS-STD-001     | blob 大小变化必须配套 `BUILD_BUG_ON` 校验                                                                | 110-security/01-lsm-framework.md §C-1                   |
| OS-STD-002     | 钩子回调签名严格由 `LSM_HOOK` 宏决定                                                                       | 110-security/01-lsm-framework.md §C-2                   |
| OS-STD-003     | blob 生命周期由 kmem\_cache 自动管理                                                                    | 110-security/01-lsm-framework.md §C-3                   |
| OS-STD-004     | `init_debug` 默认开启，便于启动审计                                                                       | 110-security/01-lsm-framework.md §A-1                   |
| OS-STD-005     | 拒绝路径必须通过 AgentsIPC 上报可读原因                                                                      | 110-security/01-lsm-framework.md §A-3                   |
| OS-STD-006     | ABI 结构体大小变化必须配套 `BUILD_BUG_ON` 校验                                                              | 110-security/02-landlock-sandbox.md §C-1                |
| OS-STD-007     | 用户缓冲区必须通过 `copy_min_struct_from_user` 校验                                                       | 110-security/02-landlock-sandbox.md §C-2                |
| OS-STD-008     | ruleset FD 通过引用计数管理生命周期，关闭即释放                                                                  | 110-security/02-landlock-sandbox.md §C-3                |
| OS-STD-009     | 域绑定后红黑树冻结为不可变                                                                                  | 110-security/02-landlock-sandbox.md §C-5                |
| OS-STD-012     | 新增代码引入新的 sparse/Smatch 警告禁止合并                                                                  | 50-engineering-standards/05-development-process.md §1.2 |
| OS-STD-DRV-012 | sysfs `show` 回调必须使用 `sysfs_emit()` 而非 `sprintf()`/`snprintf()`                                 | 60-driver-model/01-device-model.md §5.3                 |
| OS-STD-DRV-013 | `show` 回调返回值是写入的字节数（含换行符）                                                                      | 60-driver-model/01-device-model.md §5.3                 |
| OS-STD-DRV-014 | `device_register()` 后必须调用 `put_device()` 平衡引用计数                                                | 60-driver-model/01-device-model.md §6.2                 |
| OS-STD-DRV-015 | driver 必须在 `MODULE_AUTHOR`/`MODULE_DESCRIPTION`/`MODULE_LICENSE` 三处填充完整元数据                     | 60-driver-model/01-device-model.md §6.4                 |
| OS-STD-OBS-010 | BPF 程序复杂度（指令数）不得超过验证器上限（Linux 6.6 默认 100 万指令）；超限程序必须拆分为多个小程序                                   | 90-observability/02-ebpf-probes.md                      |
| OS-STD-OBS-011 | BPF 程序中所有 dynptr 在退出前必须调用 `submit` 或 `discard`，验证器检测到未释放 dynptr 必须拒绝加载                         | 90-observability/02-ebpf-probes.md                      |
| OS-STD-OBS-012 | BPF 程序加载失败时必须记录验证器日志，含程序名、失败指令偏移、错误原因三要素                                                       | 90-observability/02-ebpf-probes.md §6.3                 |
| OS-STD-OBS-013 | BPF map 创建必须显式设置 `max_entries`，ringbuf ≥1MB，hash/array ≥1024 条目                                | 90-observability/02-ebpf-probes.md §8.3                 |
| OS-STD-OBS-014 | BPF 程序 .o 文件必须随 airymaxos-kernel 发布包分发                                                         | 90-observability/02-ebpf-probes.md §11.2                |
| OS-STD-OBS-015 | agentrt-linux eBPF 与 agentrt 共享 `include/uapi/linux/airymax/bpf_struct_ops.h`，struct\_ops state 枚举值两端必须一致 | 90-observability/02-ebpf-probes.md §13.2                |
| OS-STD-OBS-016 | 文档中引用的 BPF 程序类型、map 类型、kfunc 名必须与 Linux 6.6 内核基线 `include/uapi/linux/bpf.h` 保持一致               | 90-observability/02-ebpf-probes.md §14.1                |

> **交叉引用说明**：原 OS-STD-017"编号管理元规则"于 2026-07-15 提升为 OS-IRON-015，权威登记已迁移至 [§2 IRON 工程铁律](#第-2-章-iron-工程铁律不可妥协)。此处保留交叉引用以追溯迁移路径，避免在 §4.4 表格内重复登记。

> **编号空间声明**：
>
> - `OS-STD-001~011`：110-security + 05-development-process + 工程标准总纲（已登记）
> - `OS-STD-012`：构建规则权威定义（05-development-process）
> - `OS-STD-013~029`：保留扩展空间
> - `OS-STD-030~099`：Rust 编码规则（见 §4.5）
> - `OS-STD-DRV-0xx`：驱动模型子域规则（60-driver-model 权威定义）
> - `OS-STD-OBS-0xx`：可观测性子域规则（90-observability 权威定义）
> - `OS-STD-CODE-0xx`：编码规范子域规则（见 §4.1）
> - `OS-STD-FMT-0xx`：格式规范子域规则（见 §4.2）
> - `OS-STD-STY-0xx`：风格规范子域规则（见 §4.3）

***

## 第 10 章 OS-ARCH 架构工程规则

> **权威定义文档**：10-architecture/01-system-architecture.md + 02-five-dimensional-principles.md + 03-microkernel-strategy.md + 04-engineering-baseline.md + 05-adrs.md
>
> **登记背景（2026-07-15）**：历史 OS-ARCH-001\~010 共 10 个编号在 10-architecture/ 5 个主题文档中使用，但完全未在 SSoT 登记构成 P0 级 SSoT 违规。本节摘要登记，消除违规。
>
> **编号空间声明**：OS-ARCH-001\~010 为 AirymaxOS 架构工程规则系列，权威定义散布于 10-architecture/ 各主题文档；新增编号须通过 RFC 流程在本表登记后方可使用。

| 编号          | 规则摘要                                                                                                                                                                                | 权威定义                                                      |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| OS-ARCH-001 | agentrt-linux 与 agentrt 的同源关系遵循 IRON-9 v3 四层共享模型——\[SC] 共享契约层完全共享代码（10 个核心头文件，bpf\_struct\_ops.h 为补充共享文件非 [SC] 核心）、\[SS] 语义同源层高层 API 语义同源、\[IND] 完全独立层各自独立、\[DSL] 降级生存层提供 [SC] 损坏时的最小可运行子集；禁止在双端之间引入适配层或兼容别名层                                   | 10-architecture/01-system-architecture.md §8 L240         |
| OS-ARCH-002 | 跨态协作遵循"零适配层天然契合"原则——通过 \[SC] 共享契约层头文件直接对接，不生成 `airy_compat_aliases.h` 或任何兼容层；Linux 平台可选启用 agentrt-linux 内核加速路径（ops 注入机制，非强制），macOS/Windows 仅走用户态路径                                  | 10-architecture/01-system-architecture.md §8 L330         |
| OS-ARCH-003 | 五维正交 24 原则在 agentrt（用户态）与 agentrt-linux（内核态）间遵循 IRON-9 v3 四层共享模型——S/K/C/E/A 五维跨态同源，认知观契约经 \[SC] 共享，内核观实现经 \[IND] 各自独立，禁止为五维原则引入双端适配别名层                                              | 10-architecture/02-five-dimensional-principles.md §9 L568 |
| OS-ARCH-004 | 五维原则跨态同源遵循"零适配层天然契合"——A 设计美学与 C 认知观经 \[SC] 头文件直接对接，不生成 `five_dim_compat.h` 或任何兼容别名层；K 内核观因平台差异落入 \[IND]，两端各自演进                                                                      | 10-architecture/02-five-dimensional-principles.md §9 L647 |
| OS-ARCH-005 | seL4 微内核设计思想在 agentrt-linux 三层模型中分布落地——capability 模型经 \[SC] `security_types.h` 共享契约，IPC 消息传递经 \[SC] `ipc.h` + \[SS] 同源 API 双层落地，形式化验证经 \[IND] 各自独立实现，禁止将 seL4 验证代码混入共享契约层           | 10-architecture/03-microkernel-strategy.md §10 L584       |
| OS-ARCH-006 | 微内核化改造遵循"seL4 思想分布落地"原则——capability/IPC 契约经 \[SC] 共享保证双端一致，形式化验证落入 \[IND] 由 tests-linux 独立维护，io\_uring 仅用于用户态-内核态通信、kthread 间改用 kfifo + wait\_event\_interruptible                  | 10-architecture/03-microkernel-strategy.md §10 L656       |
| OS-ARCH-007 | 工程基线在 agentrt（CMake/libc/POSIX）与 agentrt-linux（Kbuild/Kconfig/Linux 6.6）间遵循 IRON-9 v3 四层共享模型——编码契约经 \[SC] 共享，构建系统与平台适配落入 \[IND] 各自独立，禁止在双端工程基线间引入构建兼容垫片                             | 10-architecture/04-engineering-baseline.md §8 L581        |
| OS-ARCH-008 | 工程基线跨态协作遵循"契约共享、构建独立"原则——头文件编码契约经 \[SC] 直接共享，CMake 与 Kbuild 经 \[SS] 风格同源但工具链独立落入 \[IND]，禁止生成 `build_compat shim` 或构建兼容垫片                                                            | 10-architecture/04-engineering-baseline.md §8 L654        |
| OS-ARCH-009 | 14 个 ADR 中涉及 agentrt 同源关系的 6 个核心 ADR（ADR-003/004/005/006/007/010）遵循 IRON-9 v3 四层共享模型分布——capability/IPC/认知/记忆契约经 \[SC] 共享，8 子仓与 7 模块经 \[SS] 同源，构建与平台适配经 \[IND] 独立                    | 10-architecture/05-adrs.md §12 L1070                      |
| OS-ARCH-010 | ADR 同源关系汇总遵循"头文件契约共享、ADR-003/010 双端映射、构建与验证独立"原则——ADR-004/005/006/007 经 \[SC] 共享契约，ADR-003/010 经 \[SS] 双端同源映射，io\_uring 仅用于用户态-内核态通信、kthread 间改用 kfifo + wait\_event\_interruptible | 10-architecture/05-adrs.md §12 L1156                      |

***

## 第 11 章 OS-IFACE 接口工程规则

> **权威定义文档**：30-interfaces/01-syscalls.md + 02-ipc-protocol.md + 03-sdk-api.md + 04-coding-standard.md + 05-syscall-semantic-mapping.md
>
> **登记背景（2026-07-15）**：历史 OS-IFACE-001\~010 共 10 个编号在 30-interfaces/ 5 个主题文档中使用，但完全未在 SSoT 登记构成 P0 级 SSoT 违规。本节摘要登记，消除违规。
>
> **编号空间声明**：OS-IFACE-001\~010 为 AirymaxOS 接口工程规则系列，覆盖 syscall/IPC/SDK/编码规范/语义映射五个子域；新增编号须通过 RFC 流程在本表登记后方可使用。

| 编号           | 规则摘要                                                                                                                                                                                                                           | 权威定义                                                 |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- |
| OS-IFACE-001 | 系统调用接口遵循 IRON-9 v3 四层共享模型——agentrt 用户态 `syscalls.h` 与 agentrt-linux 内核 `airy_syscalls.h` 的编号、签名、错误码通过 \[SC] 共享契约层头文件同源；syscall 表注册、`SYSCALL_DEFINE*` 宏、capability 守卫实现各自独立；禁止在用户态与内核态之间引入 syscall 号映射表或编号转换层                   | 30-interfaces/01-syscalls.md §8 L483                 |
| OS-IFACE-002 | 系统调用编号 0-9 段在 agentrt 用户态（`syscalls.h` 宏定义）与 agentrt-linux 内核态（`syscall_64.tbl` 表项）保持二进制一致——同一编号在两侧语义完全相同，禁止在用户态引入编号重映射；`AIRY_E*` 错误码在两侧同源，用户态通过 `airy_errno_to_linux()` 互转，但 syscall 返回值本身不转换                                 | 30-interfaces/01-syscalls.md §8 L562                 |
| OS-IFACE-003 | IPC 协议遵循 IRON-9 v3 四层共享模型——128B 消息头结构、magic 0x41524531（'ARE1'）、5 种 payload type 通过 \[SC] 共享契约层 `ipc.h` 完全共享；io\_uring 操作码与传输实现各自独立；禁止在 agentrt 与 agentrt-linux 之间引入协议转换层或字节序适配层                                                | 30-interfaces/02-ipc-protocol.md §8 L456             |
| OS-IFACE-004 | IPC 协议的 128B 消息头布局（`struct airy_ipc_msg_hdr`）与 magic 0x41524531（'ARE1'）在 agentrt 与 agentrt-linux 间完全共享同一份 `ipc.h`——agentrt 在 agentrt-linux 上运行时无需任何协议转换或字节序适配，消息头字段布局二进制兼容                                                     | 30-interfaces/02-ipc-protocol.md §8 L526             |
| OS-IFACE-005 | SDK API 遵循 IRON-9 v3 四层共享模型——SDK 层签名同源（同一份源码两端编译，构建期条件编译切换传输层），其他层（系统调用层等）仅语义同源、签名因抽象层级不同而独立演进；底层系统调用通过 \[SC] 共享契约层头文件同源；传输层与构建链各自独立；禁止在 SDK 层引入平台判定或后端切换的适配层                                                                  | 30-interfaces/03-sdk-api.md §9 L512                  |
| OS-IFACE-006 | 4 语言 SDK 的 `AirymaxClient` 及 4 嵌套客户端（CognitionClient/SafetyClient/ToolClient/ChatClient）API 签名在 agentrt 与 agentrt-linux 上一致——SDK 层签名同源（同一份 SDK 源码两端编译运行，底层传输由构建期条件编译切换）；此一致性仅限 SDK 层，系统调用层等签名因抽象层级不同而独立演进、仅概念操作语义同源；禁止运行时平台判定  | 30-interfaces/03-sdk-api.md §9 L583                  |
| OS-IFACE-007 | 编码规范遵循 IRON-9 v3 四层共享模型——命名前缀（`airy_`）、类型命名（`snake_case_t`）、Doxygen 注释、错误码（`AIRY_E*`）通过 \[SC] 共享契约层头文件同源；构建系统与缩进风格各自独立；禁止在 agentrt 与 agentrt-linux 之间引入风格转换工具或 lint 规则映射层                                                      | 30-interfaces/04-coding-standard.md §10 L487         |
| OS-IFACE-008 | 编码规范在 agentrt（用户态，4 空格 + 1TBS + 100 列）与 agentrt-linux 内核态（Linux 6.6 内核基线 Tab 8 + K\&R + 80 列 + errno+goto）间保持命名同源而风格独立——\[SC] 共享头文件统一采用 `airy_` 前缀 + `snake_case_t` + Doxygen + `AIRY_E*`，在两侧编译期通过 `-I` 引用同一份源码，禁止生成风格转换中间文件   | 30-interfaces/04-coding-standard.md §10 L555         |
| OS-IFACE-009 | 系统调用语义映射遵循 IRON-9 v3 四层共享模型——agentrt 高层 API（JSON-RPC 统一入口）与 agentrt-linux 编号 syscall（512-631）在 \[SS] 语义同源层保持概念操作一致，签名因抽象层级不同而独立演进；SDK 层（AirymaxClient 4 语言）签名确实同源（同一份源码两端编译）；禁止在系统调用层引入签名转换层或 JSON-to-struct 适配层，签名同源仅限于 SDK 层 | 30-interfaces/05-syscall-semantic-mapping.md §8 L332 |
| OS-IFACE-010 | SDK 层签名同源是 IRON-9 v3 在系统调用领域的唯一签名同源点——agentrt 与 agentrt-linux 的系统调用层签名因抽象层级不同而独立演进，仅 SDK 层（AirymaxClient 4 语言）通过同一份源码两端编译实现签名同源；禁止在系统调用层引入签名转换层、JSON-to-struct 适配层或编号映射表来强制签名同源                                                | 30-interfaces/05-syscall-semantic-mapping.md §8 L425 |

***

## 第 12 章 OS-TEST 测试工程规则

> **权威定义文档**：80-testing/01-kunit-framework.md + 80-testing/02-kselftest.md
>
> **登记背景（2026-07-15）**：历史 OS-TEST-001\~022 共 22 个编号在 80-testing/ 2 个主题文档中使用，但完全未在 SSoT 登记构成 P0 级 SSoT 违规。本节摘要登记，消除违规。
>
> **编号空间声明**：OS-TEST-001\~022 为 AirymaxOS 测试工程规则系列，覆盖 KUnit 单元测试（001-012）与 kselftest 系统级测试（013-022）两个子域；新增编号须通过 RFC 流程在本表登记后方可使用。

| 编号          | 规则摘要                                                                                                                        | 权威定义                                      |
| ----------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| OS-TEST-001 | 所有 agentrt-linux 内核模块必须提供至少一个 KUnit 测试套件；无 KUnit 测试的模块禁止合入 `kernel` 主分支                                                     | 80-testing/01-kunit-framework.md §3 L74   |
| OS-TEST-002 | KUnit 测试默认在 UML 上运行；若测试依赖硬件特性，必须通过 `kunit_mark_skipped()` 在非 UML 平台显式跳过并标注原因                                                | 80-testing/01-kunit-framework.md §3 L76   |
| OS-TEST-003 | `init`/`exit` 用于每用例的资源分配/释放；`suite_init`/`suite_exit` 用于跨用例共享资源；若资源由 KUnit 托管（`kunit_kmalloc` 等），无需手写 `exit`，违反此规则需在评审中说明理由 | 80-testing/01-kunit-framework.md §4 L94   |
| OS-TEST-004 | 所有依赖前置资源（内存分配、句柄获取）的断言必须用 `ASSERT_*`；断言失败会导致后续代码 NULL 解引用时必须用 `ASSERT_*`；普通业务断言用 `EXPECT_*` 以收集尽可能多的失败信息                    | 80-testing/01-kunit-framework.md §5 L137  |
| OS-TEST-005 | 所有接受枚举/标志位/数值区间入参的内核接口必须有参数化测试覆盖；参数集必须包含边界值（0、最小、最大、非法值各一例）                                                                 | 80-testing/01-kunit-framework.md §6 L171  |
| OS-TEST-006 | 参数生成器不得返回栈上地址；必须返回静态数组元素或堆分配（后者需在 `exit` 中释放）                                                                               | 80-testing/01-kunit-framework.md §6 L173  |
| OS-TEST-007 | 每个 `kunit_suite` 的 `name` 字段在编译单元内必须唯一；跨子仓的命名冲突由 CI 第 7 层（单元测试层）静态检查报告                                                      | 80-testing/01-kunit-framework.md §7 L212  |
| OS-TEST-008 | 测试中所有堆分配必须经 `kunit_kmalloc`/`kunit_kzalloc` 托管；禁止裸 `kmalloc` 后忘记 `kfree`，导致内存泄漏被 kmemleak 误报为产品缺陷                           | 80-testing/01-kunit-framework.md §7 L216  |
| OS-TEST-009 | 测试禁止调用 `BUG()`/`panic()`；致命错误必须经 `KUNIT_ASSERT_*` 经 try-catch 退出；`KUNIT_FAIL()` 之后禁止再有副作用代码                                 | 80-testing/01-kunit-framework.md §8 L280  |
| OS-TEST-010 | CI 必须解析 TAP 输出并上传至测试报告系统；`not ok` 与 `# SKIP` 必须分别计数，禁止用单一 pass/fail 掩盖跳过                                                    | 80-testing/01-kunit-framework.md §9 L322  |
| OS-TEST-011 | 所有 agentrt-linux Agent SDK 接口必须有 KUnit 契约测试；契约测试覆盖正常路径 + 至少 1 个异常路径 + 至少 1 个边界路径                                            | 80-testing/01-kunit-framework.md §10 L382 |
| OS-TEST-012 | PR 引入新 KUnit 套件时，CI 必须对比 PR 前后的 TAP 用例数；新增套件必须使 agentrt-linux 总用例数单调递增（不可因重构而静默删除套件）                                        | 80-testing/01-kunit-framework.md §11 L430 |
| OS-TEST-013 | 所有 agentrt-linux 内核特性必须有对应的 kselftest 子系统测试；特性无 kselftest 覆盖时，PR 评审必须显式标注"ksft 豁免理由"                                        | 80-testing/02-kselftest.md §3 L81         |
| OS-TEST-014 | kselftest 默认在 QEMU 上运行（CI 容器无法访问真实硬件）；测试若依赖真实硬件特性，必须用 `ksft_test_result_skip()` 在虚拟环境显式跳过并标注原因                              | 80-testing/02-kselftest.md §3 L83         |
| OS-TEST-015 | 测试退出码必须严格使用 `ksft_exit_pass()`/`ksft_exit_fail()`/`ksft_exit_skip()`；禁止用裸 `exit(0)`/`exit(1)`，破坏 CI 解析                      | 80-testing/02-kselftest.md §4 L118        |
| OS-TEST-016 | MicroCoreRT 调度算法的每个策略（FIFO/RR/DEADLINE）必须有 sched\_ext 风格系统测试；时间片测量允许 ±5% 容差（QEMU 时钟精度限制）                                    | 80-testing/02-kselftest.md §6 L212        |
| OS-TEST-017 | KUnit 测试与 kselftest 测试不可相互替代；每个 agentrt-linux 子系统必须同时有 KUnit 白盒测试（覆盖函数）与 kselftest 系统级测试（覆盖系统调用接口）                          | 80-testing/02-kselftest.md §7 L258        |
| OS-TEST-018 | agentrt-linux CI 必须提供 root 与非 root 两套 kselftest 运行矩阵；非 root 测试集标记在 `airy_enabled_targets.list` 第 2 列                        | 80-testing/02-kselftest.md §8 L298        |
| OS-TEST-019 | 所有 agentrt-linux 扩展测试在 `settings` 文件中必须声明 `timeout` 字段；CI 默认超时 60 秒，超时测试由 CI 标记为 `KSFT_FAIL`                                | 80-testing/02-kselftest.md §8 L300        |
| OS-TEST-020 | agentrt-linux Agent SDK 的每个公共 ioctl 必须有 kselftest 系统级测试；契约覆盖正常路径 + `EINVAL`/`ENOMEM`/`EBUSY` 各一例异常路径                        | 80-testing/02-kselftest.md §10 L415       |
| OS-TEST-021 | CI 必须解析 kselftest TAP 输出并上传至测试报告系统；新增/删除测试集必须使总用例数单调变化，PR 评审需显式确认                                                           | 80-testing/02-kselftest.md §11 L570       |
| OS-TEST-022 | flaky 测试（同一 PR 重复运行结果不一致）必须在 `airy_flaky_baseline.md` 登记并限期修复；未修复的 flaky 测试由 CI 标记为 `KSFT_XFAIL`                            | 80-testing/02-kselftest.md §12 L582       |

***

## 第 13 章 其他 OS-\* 工程规则扩展登记

> **登记背景（2026-07-15）**：深层审查发现 OS-OPS-NNN（100-operations/ 2 文档）/ OS-IPC-NNN（40-dataflows/03-ipc-flow\.md）/ OS-CHK-DOC-NNN + OS-CHK-CODE-NNN + OS-CHK-IRON-NNN（50-engineering-standards/05-development-process.md）共 5 个 `OS-*` 子前缀在主题文档中使用但完全未在 SSoT 登记，构成 P0 级 SSoT 违规（违反 §1.2"编号分配权"）。本章摘要登记或全量登记这 5 个子前缀共 \~96 条规则，消除违规。
>
> **编号空间声明**：OS-OPS / OS-IPC / OS-CHK-DOC / OS-CHK-CODE / OS-CHK-IRON 为 AirymaxOS 工程规则的扩展子前缀，分别覆盖运维部署/IPC 传输/文档检查/代码检查/IRON 规则检查五个子域；新增编号须通过 RFC 流程在本表登记后方可使用。

### 13.1 OS-OPS 运维工程规则（OS-OPS-001\~041 + OS-OPS-101\~139）

> **权威定义文档**：100-operations/01-deployment.md（001-041）+ 100-operations/02-configuration.md（101-139）
>
> **登记方式**：OS-OPS 共 \~75 条规则，数量较多，本节按编号区间摘要登记。完整规则清单以 `100-operations/` 两份主题文档为唯一权威来源。

| 编号区间            | 子域                      | 数量 | 权威定义                   | 关键规则摘要                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------- | ----------------------- | -- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| OS-OPS-001\~003 | 部署产物与版本锁                | 3  | 01-deployment §1       | 001 GPG 签名强制 / 002 部署可重放 / 003 kernel-services 版本锁强一致                                                                                                                                                                                                                                                                                                                                                                                                                          |
| OS-OPS-004\~008 | RPM/ISO 签名与校验           | 5  | 01-deployment §2-§3    | 004 RPM GPG 签名 / 005 develop 仓库禁用生产 / 006 dnf versionlock 三包一致 / 007 staging 验证前置 / 008 ISO SHA-256+GPG 双校验                                                                                                                                                                                                                                                                                                                                                                    |
| OS-OPS-009\~013 | Kickstart/PXE/firstboot | 5  | 01-deployment §4-§5    | 009 rootpw SHA-512 哈希 / 010 systemctl enable agentrt.target / 011 PXE 签名链路 / 012 PXE 仓库快照一致 / 013 firstboot sysctl+LSM                                                                                                                                                                                                                                                                                                                                                         |
| OS-OPS-014\~017 | systemd daemons 启动      | 4  | 01-deployment §6       | 014 WatchdogSec / 015 daemon 间 AgentsIPC 强制 / 016 12 daemons 一次性升级 / 017 DevStation loopback 限制                                                                                                                                                                                                                                                                                                                                                                                |
| OS-OPS-018\~023 | 升级/回滚/快照                | 6  | 01-deployment §7-§8    | 018 staging+7 天回滚 / 019 download 后 reboot / 020 IPC 协议 L2 稳定性 / 021 根分区快照 7 天 / 022 回滚后记忆卷一致性 / 023 dnf history rollback 配置不回滚                                                                                                                                                                                                                                                                                                                                                 |
| OS-OPS-024\~028 | 协议契约与重启策略               | 5  | 01-deployment §9       | 024 daemon-SDK 协议版本一致 / 025 RestartSec=5s / 026 WatchdogSec=30s / 027 依赖图无环 / 028 agentrt.target After=sysinit                                                                                                                                                                                                                                                                                                                                                                 |
| OS-OPS-029\~032 | RPM 元数据与权限              | 4  | 01-deployment §2       | 029 %changelog 引用提交哈希 / 030 Requires 版本声明 / 031 %config(noreplace) / 032 文件权限 0755/0644/0600                                                                                                                                                                                                                                                                                                                                                                                   |
| OS-OPS-033\~041 | 仓库/ISO/initramfs/兼容矩阵   | 9  | 01-deployment §3-§10   | 033 gpgcheck+repo\_gpgcheck / 034 ISO 仓库快照固定 / 035 ISO BIOS+UEFI / 036 /var/lib/agentrt 独立分区 / 037 agentrt.target 强依赖 sysinit / 038 MemoryMax+TasksMax / 039 DevStation AgentsIPC / 040 early-init microcorert\_probe / 041 跨大版本兼容矩阵                                                                                                                                                                                                                                           |
| OS-OPS-101\~118 | 配置层级/sysctl/daemon conf | 18 | 02-configuration §1-§5 | 101 L3-L7 配置层级 / 102 可验证/可回滚/可审计 / 103 vm.max\_map\_count≥262144 / 104 kernel.msgmni≥32768 / 105 sysctl %config(noreplace) / 106 sysctl 三时机应用 / 107 /etc/agentrt/\*.conf 权限 0640 / 108 systemd drop-in daemon-reload / 109 MemoryMax 不低于默认 / 110 daemon 启动校验配置 / 111 airy\_ipc.conf %config(noreplace) / 112 AIRY\_DATA\_DIR=/var/lib/agentrt / 113 AIRY\_IPC\_BASE\_PORT 一致性 / 114 config-check 时机 / 115 告警路由 audit\_d / 116 /etc/agentrt/ git 管理 / 117 dry-run 前置 / 118 审计日志 |
| OS-OPS-122\~135 | 密钥/sysctl 命名/drop-in 验证 | 14 | 02-configuration §3-§7 | 122 密钥引用禁止明文 / 123 sysctl 通过命令/接口 / 124 sysctl 文件命名 NN-name.conf / 125 禁止重复 sysctl 声明 / 126 daemon 独立 conf / 127 agentrt.conf 全局配置 / 128 conf.d 数字前缀 / 129 禁止修改包提供单元 / 130 systemd-analyze verify / 131 header\_size==128 / 132 环境变量禁密钥 / 133 EnvironmentFile 加载 / 134 config-check 测试覆盖 / 135 配置仓库提交原因                                                                                                                                                                        |
| OS-OPS-138\~139 | MicroCoreRT sysctl 集中管理 | 2  | 02-configuration §1    | 138 MicroCoreRT sysctl 集中 / 139 kernel.airy\_\* 文档登记                                                                                                                                                                                                                                                                                                                                                                                                                           |

> **注**：OS-OPS-119\~121、136\~137 为预留编号（未在主题文档中使用），遵循 OS-IRON-015"编号一经分配不得复用"——预留编号保持空缺，不得重新分配给其他规则。

### 13.2 OS-IPC IPC 工程规则（OS-IPC-001\~010）

> **权威定义文档**：40-dataflows/03-ipc-flow\.md

| 编号         | 规则摘要                                                                          | 权威定义                |
| ---------- | ----------------------------------------------------------------------------- | ------------------- |
| OS-IPC-001 | SQ/CQ ring 单生产者-单消费者所有权契约——daemon 只写 sq.tail+cq.head，内核只写 sq.head+cq.tail     | 03-ipc-flow §1 L61  |
| OS-IPC-002 | SQPOLL 模式 sq\_thread\_idle 超时调优——高频 IPC 5-10s，低延迟 100ms                       | 03-ipc-flow §2 L268 |
| OS-IPC-003 | DEFER\_TASKRUN + SINGLE\_ISSUER 组合是 daemon 间 IPC 推荐配置（\[SS] 语义同源层）            | 03-ipc-flow §2 L292 |
| OS-IPC-004 | registered buffer 的 bvec\[] scatter-gather 模型支持非连续物理页零拷贝传输                    | 03-ipc-flow §3 L342 |
| OS-IPC-005 | IOSQE\_BUFFER\_SELECT + IOSQE\_CQE\_SKIP\_SUCCESS 组合降低 CQE 开销                 | 03-ipc-flow §3 L370 |
| OS-IPC-006 | IORING\_OP\_MSG\_RING 的 trylock + punt-to-io-wq 死锁避免策略                        | 03-ipc-flow §4 L540 |
| OS-IPC-007 | IORING\_MSG\_SEND\_FD 模式支持 daemon 间传递固定 fd（memfd/socket fd）                   | 03-ipc-flow §4 L542 |
| OS-IPC-008 | SEND\_ZC 双 CQE 模型（数据发送完成 + 缓冲区回收）用于跨节点零拷贝                                     | 03-ipc-flow §5 L610 |
| OS-IPC-009 | 必须显式启用 11 个 IORING\_SETUP\_\* 标志，二选一 COOP\_TASKRUN/DEFER\_TASKRUN，禁止 6 个不适用标志 | 03-ipc-flow §2 L160 |
| OS-IPC-010 | 5 种 payload 协议（应用语义层）与 6 个 io\_uring OP（传输机制层）正交映射——禁止跨层嵌入                    | 03-ipc-flow §6 L407 |

### 13.3 OS-CHK-DOC 文档检查规则（OS-CHK-DOC-01\~08, 18\~19）

> **权威定义文档**：50-engineering-standards/05-development-process.md Part III §5

| 编号            | 规则摘要                                                                         | 权威定义                    | 优先级 |
| ------------- | ---------------------------------------------------------------------------- | ----------------------- | --- |
| OS-CHK-DOC-01 | 所有 Markdown 文档必须以 Copyright 头部开头                                             | 05-dev-process §5 L1458 | P0  |
| OS-CHK-DOC-02 | 开源文档严禁出现禁词（完整禁词清单维护于内部配置文件）                                                  | 05-dev-process §5 L1477 | P0  |
| OS-CHK-DOC-03 | 深度修订的子系统设计文档必须包含 IRON-9 v3 四层共享模型标注                                          | 05-dev-process §5 L1513 | P0  |
| OS-CHK-DOC-04 | 所有引用 \[SC] 共享契约层的文档必须与 10 个头文件清单一致                                            | 05-dev-process §5 L1544 | P0  |
| OS-CHK-DOC-05 | 所有 Markdown 内部链接必须指向存在的文件                                                    | 05-dev-process §5 L1561 | P1  |
| OS-CHK-DOC-06 | 设计文档行数应在合理范围内                                                                | 05-dev-process §5 L1581 | P1  |
| OS-CHK-DOC-07 | 深度修订的文档应包含至少 2 个 Mermaid 图表                                                  | 05-dev-process §5 L1594 | P1  |
| OS-CHK-DOC-08 | 深度修订的文档应引用"五维正交 24 原则"至少 2 次                                                 | 05-dev-process §5 L1611 | P2  |
| OS-CHK-DOC-18 | \[SC] 共享契约层 syscall 编号定义必须采用 seL4 syscall.xml 式单一来源管理（R-04，1.0.1 增补，依赖 R-01） | 05-dev-process §5 L1627 | P2  |
| OS-CHK-DOC-19 | \[SC] 共享契约层位域结构体定义应参考 seL4 structures.bf 模式（R-04，1.0.1 增补，依赖 R-02）           | 05-dev-process §5 L1664 | P2  |

> **注**：OS-CHK-DOC-09\~17 为预留编号（未在主题文档中使用），遵循 OS-IRON-015"编号一经分配不得复用"。

### 13.4 OS-CHK-CODE 代码检查规则（OS-CHK-CODE-01\~05, 08）

> **权威定义文档**：50-engineering-standards/05-development-process.md Part III §5

| 编号             | 规则摘要                                       | 权威定义                    |
| -------------- | ------------------------------------------ | ----------------------- |
| OS-CHK-CODE-01 | 所有 C/C++/Rust 源文件必须包含 SPDX 许可证标签           | 05-dev-process §5 L1707 |
| OS-CHK-CODE-02 | 函数与变量命名遵循 01-coding-standards.md §3.1      | 05-dev-process §5 L1733 |
| OS-CHK-CODE-03 | 所有运行时日志必须使用核心日志系统                          | 05-dev-process §5 L1756 |
| OS-CHK-CODE-04 | 遵循 Linux 内核 goto 集中错误处理范式                  | 05-dev-process §5 L1779 |
| OS-CHK-CODE-05 | CMake 脚本遵循 cmake/airy\_print.cmake 统一打印系统  | 05-dev-process §5 L1822 |
| OS-CHK-CODE-08 | 代码中对 seL4 的借鉴仅限 ES-SEL4-1\~5 架构层，不得借鉴编码风格层 | 05-dev-process §5 L1833 |

> **注**：OS-CHK-CODE-06\~07、09+ 为预留编号（未在主题文档中使用），遵循 OS-IRON-015"编号一经分配不得复用"。

### 13.5 OS-CHK-IRON IRON 规则检查（OS-CHK-IRON-01\~04, 18）

> **权威定义文档**：50-engineering-standards/05-development-process.md Part III §5

| 编号             | 规则摘要                                                                           | 权威定义                    |
| -------------- | ------------------------------------------------------------------------------ | ----------------------- |
| OS-CHK-IRON-01 | \[SC] 共享契约层 10 个头文件必须 agentrt 与 agentrt-linux 完全一致                              | 05-dev-process §5 L1879 |
| OS-CHK-IRON-02 | \[SS] 语义同源层：SDK 层 API 签名应与 agentrt 同源；系统调用层签名独立演进，仅要求概念操作语义同源                  | 05-dev-process §5 L1907 |
| OS-CHK-IRON-03 | \[IND] 独立层不应引用对方仓库的内部实现                                                        | 05-dev-process §5 L1913 |
| OS-CHK-IRON-04 | agentrt-linux 工程思想与上游 Linux 保持一致性                                              | 05-dev-process §5 L1929 |
| OS-CHK-IRON-18 | agentrt-linux 采用 MAINTAINERS 文件范式（Linux 6.6 基线 14 字段 + 3 新增字段），非 seL4 TSC 集中治理 | 05-dev-process §5 L1943 |

> **注**：OS-CHK-IRON-05\~17 为预留编号（未在主题文档中使用），遵循 OS-IRON-015"编号一经分配不得复用"。

### 13.6 OS-DEV 开发流程工程规则（OS-DEV-101\~194 + OS-DEV-201\~282）

> **权威定义文档**：120-development-process/01-patch-lifecycle.md（101-194, 241-243）+ 120-development-process/02-maintainer-hierarchy.md（201-202, 241-243, 251-282）
>
> **登记方式**：OS-DEV 共 \~60 条规则，按补丁生命周期阶段与维护者层级两个维度组织，本节按编号区间摘要登记。完整规则清单以 `120-development-process/` 两份主题文档为唯一权威来源。

| 编号区间            | 子域                                   | 数量 | 权威定义                                         |
| --------------- | ------------------------------------ | -- | -------------------------------------------- |
| OS-DEV-101\~103 | RFC/Design 阶段                        | 3  | 01-patch-lifecycle §1                        |
| OS-DEV-111\~115 | Early Review 早期审查                    | 5  | 01-patch-lifecycle §2                        |
| OS-DEV-121\~124 | Wider Review/develop 分支              | 4  | 01-patch-lifecycle §3                        |
| OS-DEV-131\~134 | Mainline 合并入主线                       | 4  | 01-patch-lifecycle §4                        |
| OS-DEV-141\~144 | Stable Release 稳定发布                  | 4  | 01-patch-lifecycle §5                        |
| OS-DEV-151\~154 | Long-term Maintenance 长期维护           | 4  | 01-patch-lifecycle §6                        |
| OS-DEV-161\~163 | 回退/重试机制                              | 3  | 01-patch-lifecycle §7                        |
| OS-DEV-171\~173 | 跨仓 PR 管理                             | 3  | 01-patch-lifecycle §8                        |
| OS-DEV-181\~184 | Commit 格式与 DCO 签名                    | 4  | 01-patch-lifecycle §9                        |
| OS-DEV-191\~194 | 跨仓合并顺序与 submodule                    | 4  | 01-patch-lifecycle §10                       |
| OS-DEV-201\~202 | 信任链传递规则                              | 2  | 02-maintainer-hierarchy §2                   |
| OS-DEV-241\~243 | 审查质量规则（橡皮图章/NAK 技术理由/撤销 Reviewed-by） | 3  | 01-patch-lifecycle + 02-maintainer-hierarchy |
| OS-DEV-251\~254 | 维护者层级晋升                              | 4  | 02-maintainer-hierarchy §3                   |
| OS-DEV-261\~264 | 维护者离职与接班                             | 4  | 02-maintainer-hierarchy §4                   |
| OS-DEV-271\~272 | Emeritus 荣誉退休                        | 2  | 02-maintainer-hierarchy §5                   |
| OS-DEV-281\~282 | Orphan 区域管理                          | 2  | 02-maintainer-hierarchy §6                   |

> **注**：OS-DEV-001\~100 段未使用（规则从 101 起）；OS-DEV-204\~240、245\~250、255\~260、265\~270、273\~280、283+ 为预留编号，遵循 OS-IRON-015。

### 13.7 OS-BUILD 构建系统工程规则（OS-BUILD-001\~016 + OS-BUILD-021\~028）

> **权威定义文档**：70-build-system/01-kbuild-system.md（001-016）+ 70-build-system/02-kconfig-system.md（021-028）

| 编号区间              | 子域                      | 数量 | 权威定义                    | 关键规则摘要                                                                                                                                                                                                      |
| ----------------- | ----------------------- | -- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OS-BUILD-001\~012 | Kbuild descending 与构建规则 | 12 | 01-kbuild-system §1-§8  | 001 顶层 Kbuild descending / 002 子目录 obj-y 声明 / 003 obj-y/obj-m 互斥 / 010 filechk version.h / 011 built-in.a 链接 / 012 CI 三配置验证                                                                                 |
| OS-BUILD-013\~016 | 8 子仓构建编排                | 4  | 01-kbuild-system §9     | 013 分层依赖顺序 / 014 工具链版本锁定 / 015 构建产物签名 manifest / 016 交叉构建配置矩阵                                                                                                                                               |
| OS-BUILD-021\~028 | Kconfig 配置系统规则          | 8  | 02-kconfig-system §1-§9 | 021 CONFIG\_\* 由 Kconfig 求解 / 022 obj-$(CONFIG\_\*) 引用 / 023 source 相对路径 / 024 AIRYMAXOS\_ 前缀 / 025 olddefconfig 验证 / 026 airymaxos-base.config / 027 allmodconfig/allnoconfig CI / 028 airy\_defconfig 一致性 |

> **注**：OS-BUILD-017\~020、029+ 为预留编号（未在主题文档中使用），遵循 OS-IRON-015。

### 13.8 OS-DRV 驱动模型工程规则（OS-DRV-001\~035）

> **权威定义文档**：60-driver-model/01-device-model.md（001-010）+ 60-driver-model/02-platform-driver.md（020-035）
>
> **编号空间隔离声明**：OS-DRV-NNN（本节，驱动模型工程规则）与 OS-STD-DRV-NNN（§4 OS-STD 子域，从 OS-STD-012\~015 迁移而来）属于不同编号空间，禁止混用。OS-DRV-001\~035 覆盖设备模型与 platform driver 全生命周期规则；OS-STD-DRV-012\~015 覆盖从 OS-STD-0xx 迁移的驱动模型编码规范。

| 编号区间            | 子域                   | 数量 | 权威定义                     | 关键规则摘要                                                                                                                                                                           |
| --------------- | -------------------- | -- | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OS-DRV-001\~010 | 设备模型核心规则             | 10 | 01-device-model §1-§6    | 001 内核态设备模型二进制兼容 / 002 用户态扩展不引入新总线 / 003 match 回调 / 004 probe 错误码 / 005 remove\_new / 006 remove 资源释放 / 007 OF sentinel / 008 OF/ACPI 等价 / 009 dev\_groups 电源检查 / 010 release 回调 |
| OS-DRV-020\~028 | Platform driver 核心规则 | 9  | 02-platform-driver §1-§5 | 020 SoC platform 总线 / 021 PCI/USB 适配层 / 022 probe 前置初始化 / 023\~028 probe/remove 生命周期与资源管理                                                                                        |
| OS-DRV-030\~035 | 资源托管与 PM 运行时         | 6  | 02-platform-driver §6-§8 | 030 devm\_add\_action\_or\_reset / 031 runtime\_suspend/resume / 032 pm\_runtime\_enable / 033 跨内核/用户态托管链 / 034 devm\_request\_irq 顺序 / 035 pm\_runtime 配对                       |

> **注**：OS-DRV-011\~019、029、036+ 为预留编号（未在主题文档中使用），遵循 OS-IRON-015。
>
> **约束别名登记（2026-07-15 补全）**：`OS-DRV-IRON-PLAT`（非数字编号，约束标识符）——`60-driver-model/02-platform-driver.md` §515 中使用的约束别名，语义为"agentrt daemons 与 agentrt-linux platform driver 不直接共享 platform.c 头文件；`agent_id` 字段语义两端一致（Agent 实例 ↔ 内核平台设备匹配键），由 `include/uapi/linux/airymax/sched.h` 间接锁定"。此为 IRON-9 v3 [IND] 完全独立层的约束声明，登记为 OS-DRV-NNN 编号空间的别名约束，不分配新数字编号。

### 13.9 OS-OBS 可观测性工程规则（OS-OBS-001\~022）

> **权威定义文档**：90-observability/01-ftrace-framework.md（001-010）+ 90-observability/02-ebpf-probes.md（011-022）
>
> **编号空间隔离声明**：OS-OBS-NNN（本节，可观测性工程规则）与 OS-STD-OBS-NNN（§4 OS-STD 子域，从 OS-STD-012\~016 迁移而来）属于不同编号空间，禁止混用。OS-OBS-001\~022 覆盖 ftrace 与 eBPF 工程规则；OS-STD-OBS-012\~016 覆盖从 OS-STD-0xx 迁移的可观测性编码规范。

| 编号区间            | 子域          | 数量 | 权威定义                       | 关键规则摘要                                                                                                                                                          |
| --------------- | ----------- | -- | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OS-OBS-001\~010 | ftrace 框架规则 | 10 | 01-ftrace-framework §1-§10 | 001 ftrace 强制基线 / 002 tracefs root 权限 / 003 available\_tracers / 004 tracer 切换 / 005 Agent 行为追踪 4 位标志 / 006\~010 tracefs 配置与输出规范                                |
| OS-OBS-011\~022 | eBPF 探针规则   | 12 | 02-ebpf-probes §1-§10      | 011 eBPF 强制基线 / 012 CONFIG\_DEBUG\_INFO\_BTF / 013 kfunc 注册 / 014\~019 BPF 程序类型与 map / 020 ringbuf 大小 / 021 struct\_ops SCHED\_AGENT / 022 struct\_ops state 读取 |

> **注**：OS-OBS-023+ 为预留编号（未在主题文档中使用），遵循 OS-IRON-015。

### 13.10 OS-MM 记忆管理工程规则（OS-MM-001\~002）

> **权威定义文档**：`40-dataflows/02-memory-flow.md`
>
> **登记说明（2026-07-15 补全）**：原 `02-memory-flow.md` 中使用的 `OS-MM-001~002` 编号未在 SSoT 登记，违反 SSoT §1.2"所有编号的分配权仅属于本注册表"。本次补全全量登记，消除 P0 级 SSoT 违规。

| 编号        | 规则                                                                                                                                                    | 章节                |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| OS-MM-001 | `include/uapi/linux/airymax/memory_types.h` 是 MemoryRovol 数据结构的 \[SC] 共享契约层——agentrt 用户态与 agentrt-linux 内核态通过此头文件共享 L1-L4 数据结构定义、GFP 掩码语义、PMEM 持久化接口，避免双份定义导致不一致 | 02-memory-flow §1 |
| OS-MM-002 | 五维原则在记忆卷载数据流中的落地必须经过工程规范委员会审查，新增设计不得违反已落地的原则                                                                                                          | 02-memory-flow §2 |

> **注**：OS-MM-003+ 为预留编号（未在主题文档中使用），遵循 OS-IRON-015。

### 13.11 OS-TST 测试工程规则补充（OS-TST-013\~016）

> **权威定义文档**：`80-testing/01-kunit-framework.md`
>
> **登记说明（2026-07-15 补全）**：原 `01-kunit-framework.md` 中使用的 `OS-TST-013~016` 编号未在 SSoT 登记，违反 SSoT §1.2。本次补全登记。
>
> **编号空间隔离声明**：OS-TST-NNN（本节，测试工程规则补充）与 OS-TEST-NNN（§12 OS-TEST 测试工程规则）及 OS-STD-TEST-NNN（§4.12 测试编码规范子域）属于不同编号空间，禁止混用。OS-TST 承载 KUnit 框架下的 Agent 行为契约测试补充规则；OS-TEST 承载测试工程全局规则；OS-STD-TEST 承载测试编码规范。

| 编号         | 规则                                                                                                                                                                                                   | 章节                 |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| OS-TST-013 | 每种 Agent 角色（LLM/TOOL/PLAN/OBSERVE）必须有完整的 8 行为测试用例覆盖（probe/bind/dispatch/suspend/resume/terminate/token\_exhaust/comm\_failure），共 32 个用例；缺失任一行为测试视为契约覆盖不完整，禁止合入                                       | 01-kunit-framework |
| OS-TST-014 | token\_exhaust 行为测试必须验证 `token_handler` 被调用且 `TOKEN_EVENT_EXHAUST` 事件触发；测试通过 mock 注入 `mock_token_handler_call_count()` 计数器验证回调执行，禁止仅断言返回值不验证副作用                                                      | 01-kunit-framework |
| OS-TST-015 | comm\_failure 行为测试必须验证 5 秒超时回退——通过 `ktime_get()` 计时断言回退耗时在 \[4900ms, 5100ms] 区间内，且回退后 Agent 进入降级模式（OBSERVE 进入本地缓存模式、PLAN 回退上一节点）；禁止用 `msleep(5000)` 硬等待替代真实超时检测                                      | 01-kunit-framework |
| OS-TST-016 | Agent 行为契约测试套件必须以 `airy_agent_contract` 前缀命名（如 `airy_agent_contract_llm`），遵循 OS-KER-095 命名规范；套件注册到 `.kunit_test_suites` ELF section，由 CI 第 7 层 `kunit.filter_glob=airy_agent_contract_*` 单独运行以隔离命名空间 | 01-kunit-framework |

> **注**：OS-TST-001\~012、017+ 为预留编号（未在主题文档中使用），遵循 OS-IRON-015。

***

## 第 14 章 agentrt 规则编号体系

> **权威定义文档**：agentrt 工程标准规范手册 v29.0（2026-07-15）
>
> **适用范围**：agentrt 用户态运行时全部模块（atoms/cupolas/daemons/gateway/heapstore/protocols/commons/sdk/ecosystem）。不适用于 AirymaxOS（agentrt-linux）8 子仓——后者使用 `OS-*` 前缀规则（第 2-13 章）。
>
> **编号体系隔离声明**：agentrt 规则编号不使用 `OS-` 前缀（如 `IRON-1`、`BAN-001`），与 AirymaxOS `OS-*` 编号属于不同编号空间。两套体系遵循 IRON-9 v3 四层共享模型同源但独立，禁止编号混用。
>
> **17 类规则总览**：

| #  | 前缀       | 类别              | 编号区间                                     | 数量  | 权威定义章节                                 | 登记方式        |
| -- | -------- | --------------- | ---------------------------------------- | --- | -------------------------------------- | ----------- |
| 1  | IRON     | 工程铁律            | IRON-1\~10                               | 10  | §35.10 + §36.1 + §38.1 + §38.10        | §14.1 全量登记  |
| 2  | BAN      | 禁止规则            | BAN-001\~361                             | 361 | §3-§38 各章                              | §14.2 摘要登记  |
| 3  | STD      | 开放标准规范          | STD-01\~08                               | 8   | §34.1\~§34.8                           | §14.3 全量登记  |
| 4  | ACC      | 验收标准            | ACC-001\~149 + ACC-SP01\~SP12 + ACC-OS04 | 149 | 130-roadmap/06-acceptance-criteria.md  | §14.4 摘要登记  |
| 5  | FOUND    | 奠基性版本工程规范       | FOUND-01\~10                             | 10  | §35.1\~§35.10                          | §14.5 全量登记  |
| 6  | SPLIT    | 38 仓 git 拆分工程规范 | SPLIT-01\~08                             | 8   | §36.2\~§36.9                           | §14.6 全量登记  |
| 7  | PROD     | 生产就绪工程规范        | PROD-01\~06                              | 6   | §37.1\~§37.6                           | §14.7 全量登记  |
| 8  | ARC      | 微内核架构规范         | ARC-01\~08                               | 8   | §33.1\~§33.8                           | §14.8 全量登记  |
| 9  | LC       | 开源许可证规范         | LC-01\~08                                | 8   | §32.1\~§32.8                           | §14.9 全量登记  |
| 10 | PRT      | 统一打印系统规范        | PRT-01\~18                               | 18  | §29.2\~§29.4                           | §14.10 摘要登记 |
| 11 | LOG      | 统一日志系统规范        | LOG-01\~26                               | 26  | §30.1\~§30.6                           | §14.11 摘要登记 |
| 12 | PATH-BAN | 路径硬编码禁止规则       | PATH-BAN-1\~5                            | 5   | §3.5                                   | §14.12 全量登记 |
| 13 | L        | 历史教训            | L-01\~53                                 | 53  | §2.1\~§2.2                             | §14.13 摘要登记 |
| 14 | CROSS    | 跨平台规范           | CROSS-01\~06                             | 6   | §4.6                                   | §14.14 全量登记 |
| 15 | REQ      | 需求规则            | REQ-01\~08                               | 8   | §4.7                                   | §14.15 全量登记 |
| 16 | W        | 工作任务            | W01-W30                                  | 30  | 130-roadmap/01-development-strategy.md | §14.16 摘要登记 |
| 17 | SP       | 29 仓拆分工作        | SP01-SP37                                | 37  | 130-roadmap/01-development-strategy.md | §14.17 摘要登记 |

### 14.1 agentrt IRON 工程铁律（IRON-1\~10）

> **权威定义文档**：`agentrt工程标准规范手册.md` §35.10（IRON-1\~5）+ §36.1（IRON-6\~8）+ §38.1（IRON-9）+ §38.10（IRON-10）

| 编号          | 规则                                                                                                     | 定义章节   |
| ----------- | ------------------------------------------------------------------------------------------------------ | ------ |
| **IRON-1**  | 禁止新特性：0.1.1 不引入任何新功能，仅修复 + 集成 + 标准化                                                                    | §35.10 |
| **IRON-2**  | 禁止桩函数/桩文件：所有功能必须真实实现，禁止 no-op / TODO / FIXME                                                           | §35.10 |
| **IRON-3**  | 禁止绕开功能限制的编译/集成：禁止 `#ifdef DISABLE_*` 绕过检查                                                              | §35.10 |
| **IRON-4**  | 禁止简化功能：不允许"为了 0.1.1 先简化，1.0.1 再完善"                                                                     | §35.10 |
| **IRON-5**  | 禁止未实现的功能设计：文档中的设计必须有对应实现                                                                               | §35.10 |
| **IRON-6**  | 禁止跨层耦合残留：0.1.1 拆仓前必须完成 5 处跨层耦合解耦（SP02-SP06），任何残留即拆仓失败                                                  | §36.1  |
| **IRON-7**  | 禁止版本过渡：0.1.1 之后直接 1.0.1，禁止 0.1.2/0.2.0/0.3.0/1.0.0 中间版本，禁止兼容别名头/兼容层/双轨制                                | §36.1  |
| **IRON-8**  | 禁止生产就绪降级：SP32-SP37 生产就绪 6 项全部必须达标，未达标禁止发布 0.1.1                                                        | §36.1  |
| **IRON-9**  | agentrt 和 AirymaxOS 同源且部分代码共享（IRON-9 v3）：三层共享模型 \[SC]+\[SS]+\[IND]，agentrt 保持跨平台用户态纯净，AirymaxOS 是完整发行版 | §38.1  |
| **IRON-10** | 禁止引用 Linux 7.0 专属特性作为 6.6 原生能力：AirymaxOS 基于 Linux 6.6，禁止 PREEMPT\_LAZY/Rust 正式转正/MGLRU 2.0 等误引用        | §38.10 |

### 14.2 agentrt BAN 禁止规则（BAN-001\~361）

> **权威定义文档**：`agentrt工程标准规范手册.md` §3-§38 各章
>
> **登记方式**：BAN 规则共 361 条，数量庞大，本节仅登记分类摘要与编号区间。完整规则清单以 `agentrt工程标准规范手册.md` 第二十章（BAN 规则完整汇总）为唯一权威来源。

| 编号区间         | 类别                                                       | 数量 | 权威定义章节        |
| ------------ | -------------------------------------------------------- | -- | ------------- |
| BAN-001\~030 | 内存管理（禁止裸 malloc/free、溢出检查、null 终止）                       | 30 | §4.1          |
| BAN-031\~059 | 其他编码规范                                                   | 29 | §4.x          |
| BAN-060\~072 | 日志系统（统一 SVC\_LOG\_*/AIRY\_LOG\_*、禁止 printf）              | 13 | §4.3          |
| BAN-073\~085 | 错误处理（禁止 return -1、错误上下文传播）                               | 13 | §4.2          |
| BAN-080\~099 | Python 规范（禁止 print()、裸 except、assert）                    | 20 | §5.1          |
| BAN-100\~110 | 桩函数禁止                                                    | 11 | §4.4          |
| BAN-111\~150 | 其他编码规范                                                   | 40 | §4.x          |
| BAN-151\~162 | 内存安全（溢出检查、strncpy null 终止、memcpy 前置校验）                   | 12 | §4.1          |
| BAN-163\~190 | 其他编码规范                                                   | 28 | §4.x          |
| BAN-191\~245 | v0.1.1 新增规则（CI/CD/安全/测试/配置/构建）                           | 55 | §3-§14        |
| BAN-246\~290 | v0.1.1 扩展（内存安全增强/部署工程/跨层耦合）                              | 45 | §4.1+§28.3    |
| BAN-291\~310 | 统一打印/日志 BAN 规则                                           | 20 | §31           |
| BAN-311\~327 | 微内核架构/开放标准 BAN 规则                                        | 17 | §33+§34       |
| BAN-328\~340 | v3.5 唯一奠基版本 BAN 规则（CoreKern 精简/taskflow/SDK/许可证/静态分析）    | 13 | §35.11        |
| BAN-341\~356 | v4.0 38 仓拆分 BAN 规则（改名彻底/拆分顺序/构建/生产就绪）                    | 16 | §11.6+§36+§37 |
| BAN-357\~361 | v4.1 AirymaxOS BAN 规则（仓库占位/文档分离/内核态污染/子仓命名/Linux 7.0 特性） | 5  | §38.9+§38.10  |

### 14.3 agentrt STD 开放标准规范（STD-01\~08）

> **权威定义文档**：`agentrt工程标准规范手册.md` §34.1\~§34.8

| 编号         | 规则                                                                                                                   | 定义章节  |
| ---------- | -------------------------------------------------------------------------------------------------------------------- | ----- |
| **STD-01** | SDK 双层 API 规范：Manager 层（向后兼容）+ 嵌套资源 API 层（对齐 OpenAI），四语言 SDK 同步实现 CognitionClient/SafetyClient/ToolClient/ChatClient | §34.1 |
| **STD-02** | IPC Bus 统一消息头规范：所有 IPC 消息必须使用 128 字节 `struct airy_ipc_msg_hdr`（\[SC] 共享契约层，magic 0x41524531 'ARE1'）                  | §34.2 |
| **STD-03** | 服务发现多后端适配器规范：支持 etcd/Consul/内置 KV 多后端，统一 `airy_service_resolver_t` 接口                                                | §34.3 |
| **STD-04** | JSON-RPC 方法命名空间规范：`airymax.<domain>.<method>` 命名空间隔离                                                                 | §34.4 |
| **STD-05** | 统一错误码规范：`airy_err_t`（int32\_t）统一错误码体系，分段注册                                                                           | §34.5 |
| **STD-06** | ARE Standards 三层标准：L1 C ABI / L2 语言原生包装 / L3 协议层（独立版本号，不违反 IRON-7）                                                   | §34.6 |
| **STD-07** | 独立仓库可行性：agentrt 可作为独立 git 仓库克隆构建，无外部硬依赖                                                                              | §34.7 |
| **STD-08** | 开放标准 BAN 规则：禁止违反 STD-01\~07 的实现，由 CI 强制                                                                              | §34.8 |

### 14.4 agentrt ACC 验收标准（149 项）

> **权威定义文档**：`agentrt工程标准规范手册.md` §37.6（PROD-06 发布验收）+ §11.6.2（改名验证）+ §38.10（AirymaxOS 兼容性）+ 各主题章节
>
> **登记方式**：ACC 验收标准共 149 项，按类别使用不同前缀（非连续编号）。agentrt 手册 §37.6 明确：149 项 = ACC-FOUND01\~08（8 项奠基验收）+ ACC-SP01\~SP12（12 项仓库拆分验收）+ 其他 129 项基础验收（含 ACC-OP/ACC-LOG 等类别前缀）。ACC-OS04 为 v4.2 新增的 AirymaxOS 兼容性特殊编号。本节仅登记分类摘要，完整验收清单以 `agentrt工程标准规范手册.md` §37.6 为唯一权威来源。

| 编号前缀              | 类别                                                       | 数量      | 权威定义章节                           |
| ----------------- | -------------------------------------------------------- | ------- | -------------------------------- |
| ACC-FOUND01\~08   | 奠基验收（CoreKern 精简 + taskflow 修复 + SDK + 双许可证 + 静态分析 + 铁律） | 8       | §35 + §37.6                      |
| ACC-SP01\~SP12    | 29 仓拆分验收（改名 + 拆分顺序 + 构建系统 + submodule + CI）              | 12      | §11.6.2 + §37.6                  |
| ACC-OS04          | AirymaxOS Linux 6.6 内核兼容性验证（v4.2 新增，无 Linux 7.0 专属特性误引）  | 1       | §38.10                           |
| ACC-OP/ACC-LOG/其他 | 基础验收（CI/CD + 日志 + 内存 + 安全 + IPC + 调度等各主题验收，按类别前缀分组）      | 128     | 各主题章节                            |
| **总计**            | <br />                                                   | **149** | §37.6 PROD-06 强制 149/149 全部 PASS |

> **注**：ACC-OS04 为 v4.2 新增特殊编号，计入 149 项总数（8+12+128+1=149）。BAN-355 禁止 149 项验收有 1 项 FAIL，发布前必须 149/149 全部 PASS。

### 14.5 agentrt FOUND 奠基性版本工程规范（FOUND-01\~10）

> **权威定义文档**：`agentrt工程标准规范手册.md` §35.1\~§35.10

| 编号           | 规则                                                                                        | 定义章节   |
| ------------ | ----------------------------------------------------------------------------------------- | ------ |
| **FOUND-01** | CoreKern 精简方案 C：从 7,522 行精简至 \~4,000 行（±200 容差），0.1.1 内完成                                 | §35.1  |
| **FOUND-02** | CoreKern 6 类 BUG 修复规范：必须在 0.1.1 内全部修复，ASan/LSan 全量验证                                      | §35.2  |
| **FOUND-03** | taskflow 双引擎协同规范：保留在 atoms，按 A/B 类语义重新分类，0.1.1 内采用双引擎协同方案                                 | §35.3  |
| **FOUND-04** | taskflow 6 类 BUG 修复规范：必须在 0.1.1 内全部修复，5 个测试文件 ctest 全绿                                    | §35.4  |
| **FOUND-05** | SDK 双层 API 4 语言规范：0.1.1 内完成 Python + Rust + Go + TypeScript 4 语言 × 4 嵌套资源 API，共 16 个嵌套客户端 | §35.5  |
| **FOUND-06** | SDK L1 C ABI 接口规范：4 语言 SDK 必须严格遵循 L1 C ABI 接口规范，CTS 一致性测试套件全绿                             | §35.6  |
| **FOUND-07** | AGPL v3 + Apache 2.0 双许可证规范：全项目 `AGPL-3.0-or-later OR Apache-2.0`，版权人 SPHARX Ltd.         | §35.7  |
| **FOUND-08** | 静态分析基线规范：lizard v1.23.0 + 重复检测器基线阈值，CCN>100 函数数 ≤ 5，重复率 < 15%                             | §35.8  |
| **FOUND-09** | 静态分析操作规范：先基线后优化，禁止无基线的盲目拆分                                                                | §35.9  |
| **FOUND-10** | 唯一奠基版本工程铁律：0.1.1 后直接 1.0.1，无中间过渡版本（IRON-1\~10 集合载体）                                       | §35.10 |

### 14.6 agentrt SPLIT 38 仓 git 拆分工程规范（SPLIT-01\~08）

> **权威定义文档**：`agentrt工程标准规范手册.md` §36.2\~§36.9

| 编号           | 规则                                                                                                                                              | 定义章节  |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| **SPLIT-01** | 38 仓目录结构工程规范：1 伞仓 + 5 管理仓 + 29 叶子仓 + 3 顶层仓，结构必须与 §11.7.1 完全一致                                                                                   | §36.2 |
| **SPLIT-02** | 仓库命名工程规范：C 层叶子仓裸名 + SDK 子仓 `sdk-xx` 命名 + 全部目录名小写                                                                                                | §36.3 |
| **SPLIT-03** | git 历史保留工程规范：agentrt 管理仓继承原 AgentRT 仓库 `git@atomgit.com:openairymax/agentos.git` 保留全部 git 历史                                                    | §36.4 |
| **SPLIT-04** | 7 阶段拆分顺序工程规范：必须按 SP01-SP07 顺序拆分，跨仓依赖未就绪禁止拆分                                                                                                     | §36.5 |
| **SPLIT-05** | 5 处跨层耦合解耦工程规范：coreloopthree←daemons/common + coreloopthree←cupolas + coreloopthree←daemons/vfs\_d + protocols←daemons/common + cupolas←gateway | §36.6 |
| **SPLIT-06** | 拆分验证工程规范：4 构建系统（CMake + Python + Rust + Go）全部通过 + submodule 一致性 + CI 集成                                                                         | §36.7 |
| **SPLIT-07** | submodule 同步与 CI 集成工程规范：`.gitmodules` 全部裸名 + feature/official-hubs-01 分支 + 跨仓 CI 触发                                                             | §36.8 |
| **SPLIT-08** | 跨平台构建工程规范：Linux/macOS/Windows 三平台构建脚本 + CMake 跨平台抽象                                                                                             | §36.9 |

### 14.7 agentrt PROD 生产就绪工程规范（PROD-01\~06）

> **权威定义文档**：`agentrt工程标准规范手册.md` §37.1\~§37.6

| 编号          | 规则                                                              | 定义章节  |
| ----------- | --------------------------------------------------------------- | ----- |
| **PROD-01** | CI/CD 全仓流水线规范：38 仓全部接入 GitHub Actions，9 矩阵（3 架构 × 3 配置）         | §37.1 |
| **PROD-02** | SBOM 供应链安全规范：SPDX SBOM 生成 + 依赖完整性校验 + 漏洞扫描                      | §37.2 |
| **PROD-03** | 跨平台二进制打包规范：Linux（deb/rpm/tar）+ macOS（pkg/tar）+ Windows（msi/zip） | §37.3 |
| **PROD-04** | 性能 Soak Test 规范：分层执行 L1（PR 4h）/ L2（周级 24h）/ L3（发布前 72h）         | §37.4 |
| **PROD-05** | 安全审计规范：静态分析 + 依赖扫描 + 密钥泄露检测 + 容器镜像扫描                            | §37.5 |
| **PROD-06** | 发布验收规范：GPG 签名 + 模块签名 + SHA-256 校验 + 149 项 ACC 全部 PASS           | §37.6 |

### 14.8 agentrt ARC 微内核架构规范（ARC-01\~08）

> **权威定义文档**：`agentrt工程标准规范手册.md` §33.1\~§33.8

| 编号         | 规则                                                                                         | 定义章节  |
| ---------- | ------------------------------------------------------------------------------------------ | ----- |
| **ARC-01** | atoms 层语义边界定义：三类模块（真正微核心原语 / 应用语义层 / 商业隔离层），物理位置不变，通过 ops 接口隔离                             | §33.1 |
| **ARC-02** | 反向依赖禁止：atoms 层禁止依赖任何上层模块（daemons/gateway/heapstore 等）                                      | §33.2 |
| **ARC-03** | atoms 可独立编译：`cmake --build build --target agentrt_atoms_standalone` 必须成功，构建产物不含 daemons 符号 | §33.3 |
| **ARC-04** | 层级倒置修复——接口反转（方案 A）：通过 ops 抽象表 + daemons init 时注入消除反向依赖，atoms 文件物理位置不变                      | §33.4 |
| **ARC-05** | airy\_core\_init() 初始化链完整：必须初始化所有必要 atoms 子系统，通过适配器注入点接受 daemons 层初始化                      | §33.5 |
| **ARC-06** | 微内核遵循性验证（方案 A）：atoms 层必须满足 5 项验证标准（无反向依赖 / 可独立编译 / 接口隔离 / 初始化链完整 / ops 注入降级）               | §33.6 |
| **ARC-07** | 三权分立与合约文件强化：同一文件只属于一个团队，交叉点通过接口契约解耦                                                        | §33.7 |
| **ARC-08** | 架构规范 BAN 规则：禁止 atoms 反向依赖 / 禁止跨层编译绑定 / 禁止绕过 ops 注入                                         | §33.8 |

### 14.9 agentrt LC 开源许可证规范（LC-01\~08）

> **权威定义文档**：`agentrt工程标准规范手册.md` §32.1\~§32.8

| 编号        | 规则                                                                                      | 定义章节  |
| --------- | --------------------------------------------------------------------------------------- | ----- |
| **LC-01** | 双许可证体系总览：OpenAirymax 主仓库内所有源代码统一使用 `AGPL-3.0-or-later OR Apache-2.0`，禁止引入第三种许可证         | §32.1 |
| **LC-02** | SPDX 标识符强制规范：所有源码文件头部必须包含 SPDX 标识符，禁止单一许可证 / 禁止 All Rights Reserved / 禁止省略 Copyright 字段 | §32.2 |
| **LC-03** | 子目录许可证路由：主仓库内所有子目录统一使用双许可证（不再区分运行时核心与工具链）                                               | §32.3 |
| **LC-04** | 许可证矛盾消除规范：禁止 SPHARX-Proprietary / CC-BY-SA-4.0 作为主仓库内文件 SPDX 标识符                        | §32.4 |
| **LC-05** | 非源码文件许可证字段：配置文件/文档/数据文件也必须包含许可证声明                                                       | §32.5 |
| **LC-06** | CI 许可证合规扫描：`[LICENSE-SCAN]` 自动扫描全部文件 SPDX 标识符，违规阻断合并                                    | §32.6 |
| **LC-07** | 贡献者许可协议 CLA：所有贡献者必须签署 CLA，DCO + Signed-off-by 强制                                        | §32.7 |
| **LC-08** | MemoryRovol 独立闭源仓库隔离规范：MemoryRovol 不在双许可证体系内，由自有商业 EULA 治理                              | §32.8 |

### 14.10 agentrt PRT 统一打印系统规范（PRT-01\~18）

> **权威定义文档**：`agentrt工程标准规范手册.md` §29.2\~§29.4
>
> **登记方式**：PRT 规则共 18 条，本节登记关键规则摘要。完整规则以 §29 为唯一权威来源。

| 编号区间       | 类别                                                             | 数量 | 权威定义  |
| ---------- | -------------------------------------------------------------- | -- | ----- |
| PRT-01\~08 | 运行时统一打印 API 规范（`airy_print()` 系列 + 分级打印 + 格式化约束）               | 8  | §29.2 |
| PRT-09\~15 | 违规打印整改清单（禁止 printf/fprintf/puts 等裸打印函数）                        | 7  | §29.3 |
| PRT-16\~18 | 遗留模块迁移规范（protocols/atoms/memory/heapstore\_migration 模块打印统一迁移） | 3  | §29.4 |

### 14.11 agentrt LOG 统一日志系统规范（LOG-01\~26）

> **权威定义文档**：`agentrt工程标准规范手册.md` §30.1\~§30.6
>
> **登记方式**：LOG 规则共 26 条，本节登记关键规则摘要。完整规则以 §30 为唯一权威来源。

| 编号区间       | 类别                                                             | 数量 | 权威定义  |
| ---------- | -------------------------------------------------------------- | -- | ----- |
| LOG-01\~05 | 核心 API 规范（`airy_log()` 系列 + 7 级日志 + 结构化字段）                     | 5  | §30.1 |
| LOG-06\~10 | 兼容层规范（旧 SVC\_LOG\_*/AIRY\_LOG\_* 兼容映射）                         | 5  | §30.2 |
| LOG-11\~18 | 关键环节增强日志覆盖（IPC/调度/内存/安全/Plugin/Hook/LLM/SSE 路径日志）              | 8  | §30.3 |
| LOG-19\~20 | 日志配置与运维规范（日志级别运行时配置 + 日志轮转）                                    | 2  | §30.4 |
| LOG-21\~25 | v3.2 新增日志系统加固规范（异步日志连通 + JSON 格式化 + syslog 后端 + 时间轮转 + 监控数据采集） | 5  | §30.5 |
| LOG-26     | v3.4 新增全功能统一打印/日志系统定义（统一率 98.2% + 打印完整度 82% + 日志完整度 62%）       | 1  | §30.6 |

### 14.12 agentrt PATH-BAN 路径硬编码禁止规则（PATH-BAN-1\~5）

> **权威定义文档**：`agentrt工程标准规范手册.md` §3.5

| 编号             | 规则                                                                    | 检测方式                                             |
| -------------- | --------------------------------------------------------------------- | ------------------------------------------------ |
| **PATH-BAN-1** | 禁止在源代码中硬编码形如 `/home/<user>/`、`C:\Users\<user>\`、`D:\SPHARX-CN\` 的绝对路径 | CI grep 扫描（`quality-gate.sh` 的 `[PATH-SCAN]` 步骤） |
| **PATH-BAN-2** | 所有运行时路径必须通过 `agentrt.yaml` 配置或 `commons/path_manager.h` 获取，禁止硬编码      | Code review                                      |
| **PATH-BAN-3** | CMake 跨子仓库引用必须使用 `get_filename_component` 相对推导，不得硬编码 `../agentrt/` 等  | CI CMake 审计                                      |
| **PATH-BAN-4** | 禁止在 git commit 中包含本机用户名或主机名                                           | pre-commit hook                                  |
| **PATH-BAN-5** | `.env` 和 `.env.example` 中的路径必须使用相对路径或环境变量占位符                          | pre-commit hook                                  |

### 14.13 agentrt L 历史教训（L-01\~53）

> **权威定义文档**：`agentrt工程标准规范手册.md` §2.1\~§2.2
>
> **登记方式**：历史教训共 53 条（L-01\~L-38 为 v0.1.0 继承，L-39\~L-53 为 v0.1.1 新增），本节仅登记分类摘要。完整教训清单以 §2 为唯一权威来源。每条教训对应至少一条 BAN/IRON/ACC 规则。

| 编号区间       | 类别                                                                                                                            | 数量 | 权威定义 |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------- | -- | ---- |
| L-01\~L-38 | v0.1.0 教训继承（宣称完成/CI 全绿假象/集成谎言/桩函数/内存安全等）                                                                                      | 38 | §2.1 |
| L-39\~L-53 | v0.1.1 新增教训（源码审计/阶段依赖/验收覆盖/非交叉原则/Hook fail-open/LLM 错误分类/SecretRef/Plugin 权限/记忆原子性/SSE 背压/.gitignore/改名同步/SDK 包名/任务状态一致/品牌统一） | 15 | §2.2 |

### 14.14 agentrt CROSS 跨平台规范（CROSS-01\~06）

> **权威定义文档**：`agentrt工程标准规范手册.md` §4.6

| 编号           | 规则                                                       | 定义章节 |
| ------------ | -------------------------------------------------------- | ---- |
| **CROSS-01** | 禁止直接使用 POSIX 线程 API：必须通过 `commons/platform/thread.h` 抽象层 | §4.6 |
| **CROSS-02** | 禁止使用 C99 VLA 变长数组：必须使用固定大小数组或动态分配                        | §4.6 |
| **CROSS-03** | 禁止直接使用 POSIX 时间函数：必须通过 `commons/platform/time.h` 抽象层     | §4.6 |
| **CROSS-04** | 禁止使用 GCC 扩展语法：必须使用标准 C11/C17                             | §4.6 |
| **CROSS-05** | Windows 头文件保护：`#ifdef _WIN32` 隔离 Windows 特定代码            | §4.6 |
| **CROSS-06** | MSVC /WX 级别零警告：Windows 构建必须零警告零错误                        | §4.6 |

### 14.15 agentrt REQ 需求规则（REQ-01\~08）

> **权威定义文档**：`agentrt工程标准规范手册.md` §4.7

| 编号         | 规则                                                                                          | 定义章节 |
| ---------- | ------------------------------------------------------------------------------------------- | ---- |
| **REQ-01** | 所有公共 API 必须使用 Doxygen 风格注释，明确 ownership、线程语义、错误语义                                           | §4.7 |
| **REQ-02** | 返回值优先使用 `airy_err_t`（int32\_t），或返回指针（失败返回 NULL 并记录日志）                                       | §4.7 |
| **REQ-03** | 所有资源按 create/destroy 成对管理，API 文档中明确释放责任                                                     | §4.7 |
| **REQ-04** | 优先使用 C11 atomics（`stdatomic.h`），避免遗留 `_sync*`                                               | §4.7 |
| **REQ-05** | 热路径优先无锁/原子实现，并提供锁保护的安全回退（System 2）                                                          | §4.7 |
| **REQ-06** | 日志格式使用结构化 JSON，最小字段：timestamp/level/module/function/trace\_id/request\_id/message/err\_code | §4.7 |
| **REQ-07** | 追踪采用 W3C Trace Context（`traceparent`），请求入口生成 `trace_id`                                     | §4.7 |
| **REQ-08** | 指标采用 Prometheus exposition 格式，提供 `/metrics`、`/healthz`、`/readyz` 端点                         | §4.7 |

### 14.16 agentrt W 工作任务（W01-W30）

> **权威定义文档**：agentrt 130-roadmap/01-development-strategy.md
>
> **登记方式**：W 工作任务共 30 项（W01-W30），是 0.1.1 唯一奠基版本的核心工作包分解。本节仅登记分类摘要。完整任务清单以 `130-roadmap/01-development-strategy.md` 为唯一权威来源。

| 编号区间    | 类别                                                         | 数量 | 权威定义                                   |
| ------- | ---------------------------------------------------------- | -- | -------------------------------------- |
| W01-W10 | 核心引擎工作（atoms/corekern 精简 + taskflow 修复 + CoreLoopThree 集成） | 10 | 130-roadmap/01-development-strategy.md |
| W11-W20 | 生态层工作（SDK 4 语言 + gateway + protocols + ecosystem）          | 10 | 130-roadmap/01-development-strategy.md |
| W21-W30 | 基础设施工作（CI/CD + 测试 + 部署 + 文档 + MemoryRovol）                 | 10 | 130-roadmap/01-development-strategy.md |

### 14.17 agentrt SP 29 仓拆分工作（SP01-SP37）

> **权威定义文档**：agentrt 130-roadmap/01-development-strategy.md
>
> **登记方式**：SP 工作包共 37 项（SP01-SP37），是 0.1.1 三大支柱之"29 仓拆分 + 生产就绪"的工作分解。本节仅登记分类摘要。完整任务清单以 `130-roadmap/01-development-strategy.md` 为唯一权威来源。

| 编号区间      | 类别                                                                                                                                           | 数量 | 权威定义                                   |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -- | -------------------------------------- |
| SP01-SP06 | 5 处跨层耦合解耦（coreloopthree←daemons/common + coreloopthree←cupolas + coreloopthree←daemons/vfs\_d + protocols←daemons/common + cupolas←gateway） | 6  | 130-roadmap/01-development-strategy.md |
| SP07-SP31 | 29 仓 git 拆分（7 阶段拆分顺序 + submodule 同步 + CI 集成）                                                                                                 | 25 | 130-roadmap/01-development-strategy.md |
| SP32-SP37 | 生产就绪 6 项（CI/CD 全仓 + SBOM + 跨平台打包 + Soak Test + 安全审计 + 发布验收）                                                                                  | 6  | 130-roadmap/01-development-strategy.md |

### 14.18 agentrt 规则编号新增流程

agentrt 规则编号新增须遵循以下流程（与 AirymaxOS `OS-*` 编号流程一致）：

1. **RFC**：在 GitHub Discussions 发起 RFC，说明规则编号、规则文本、来源文档、适用范围。
2. **评审**：经至少一名顶级子系统维护者审查，公示 14 天接受异议。
3. **注册**：通过评审后，由总维护者将规则写入 `agentrt工程标准规范手册.md` 对应章节，并在本注册表第 14 章对应小节追加登记条目。

> **双编号体系同步声明**：当 agentrt 新增规则与 AirymaxOS 存在语义同源关系时，须同时在 AirymaxOS `OS-*` 编号体系中登记对应规则（通过 IRON-9 v3 [SS] 语义同源层关联），确保两端编号体系语义一致。

***

## 附录 A 变更日志

| 日期         | 版本    | 变更内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | 作者           |
| ---------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| 2026-07-12 | 0.1.1 | 创建本注册表，替代碎片化注册表（00 §3 + 07 §10.3）；完成 OS-KER 001-155 全量消解（086-155 为新分配编号，消除 42 处跨文档冲突）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **v1.0-P0 修复：Linux 6.6 内核基线代码品味规则化——内存毒化、kmemleak 标注、DEFINE_FREE 自动释放**——(1) 新增 OS-KER-222（内存毒化保护）：POISON_FREE(0x6b)/POISON_INUSE(0x5a)/POISON_END(0xa5) 三层毒化 + SLUB RedZone(0xbb/0xcc) 越界检测。(2) 新增 OS-KER-223（kmemleak 泄漏检测标注）：自定义分配器注册、静态单例 not_leak。(3) 新增 OS-KER-224（DEFINE_FREE 自动资源释放）。C_Cpp_coding_style.md 新增 §7.4、§8.5、§8.6、§8.7 | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **v1.0-P0 修复：seL4 "校验优先于修改"原则规则化**——新增 OS-KER-225（校验优先于修改）。所有可能产生副作用的函数，必须将输入校验/权限检查/边界检查置于所有副作用（kmalloc、写寄存器、获取锁、注册表）之前。源于 seL4 `decodeUntypedInvocation()` 工程实践（11 项校验全部完成后方执行 Retype）。C_Cpp_coding_style.md 新增 §7.0。与 OS-KER-001 goto 集中出口形成"校验阶段直接返回 + 执行阶段 goto 清理"的完整错误处理策略 | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **v1.0-P0 修复：P1 残留规则化——kref、kmem_cache 选型、长循环抢占点、GFP 扩展**——(1) 新增 OS-KER-226（kref_put container_of 规范）：release 回调必须 container_of 获取外层结构体，禁止直接传 kfree。(2) 新增 OS-KER-227（kmem_cache 选型标准）：固定大小+高频(>100次)+调试支持的对象必须使用专用 kmem_cache。(3) 新增 OS-KER-228（长循环抢占点）：>1ms或>1000次迭代必须插入 cond_resched()，提供 airy_untyped_reset_children 示例。(4) OS-KER-016 扩展：§8.4.1 __GFP_NOWARN 回退路径 + §8.4.2 GFP_KERNEL_ACCOUNT 用户空间核算 + GFP 六项选型表。(5) C_Cpp_coding_style.md 新增 §8.4.1-8.4.2、§8.7-§8.8（原 §8.7 重编号为 §8.9）、§9.0 | SPHARX 工程标准组 |
| 2026-07-16 | 0.1.1 | **自主化修正 + 编译期断言规则化**——(1) 全面清除 C_Cpp_coding_style.md 中 Linux 6.6 内核基线/openEuler 引用（8 处），统一为"Linux 6.6 内核基线"自主化表述。(2) 新增 OS-KER-229（编译期断言与函数属性）：BUILD_BUG_ON/static_assert/_Static_assert 编译期不变式 + __noreturn 属性 + 头文件依赖最小化。C_Cpp_coding_style.md 新增 §6.5。(3) 修复 §7.5 编号重复（"错误处理流程总览"改为 §7.6）。(4) 新增四 Part 章节导航。(5) 09-ssot-registry.yaml 移至 agentrt-linux/ 管理仓根 | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | 第二轮 SSoT 整合：创建 3 个族索引文件（01-ssot-code-standards-index.md / 05-ssot-process-toolchain-index.md / 10-coding-style/00-ssot-c-cpp-index.md）；09-license-strategy.md 降级为 stub 重定向至 110-spdx-license-compliance.md；TERMINOLOGY.md 移入本模块并重命名为 90-terminology.md                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B1 其他语言族索引扩展：创建 3 个语言族索引（10-coding-style/00-ssot-rust-index.md / 00-ssot-go-index.md / 00-ssot-python-js-java-index.md）；发现 Rust 文件 OS-STD-030\~035 + OS-SEC 编号登记缺口（OS-STD-030\~046 已于 2026-07-14 完整登记至 §4.5；OS-SEC 三重冲突已于 2026-07-14 消解，见 §9 OS-SEC 安全工程规则——编号空间分配 001\~099 LSM / 100\~199 C / 200\~299 Rust）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B2 横切关注点族索引：创建 3 个横切族索引（10-coding-style/00-ssot-docs-index.md 文档族 / 00-ssot-logging-index.md 日志族 / 00-ssot-security-index.md 安全设计族）；横切族索引与语言族索引形成正交叉，同一 secure\_coding 文件同时属于语言族和安全族                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B3 子目录 README 登记到 SSoT：5 个子目录 README（10/20/30/40/50）头部追加 SSoT 依赖声明（编号权威 09 §3 + 族索引导航 §4.5）；09-ssot-registry.md 新增 §4.6 子目录 README SSoT 依赖登记表；20-contracts/README.md 修复父文档路径（../../README.md → ../00-engineering-standards-handbook.md）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B4 扩展文档 SSoT 依赖声明：6 个扩展文档（60/70/80/100/110/120）头部追加编号权威字段（09 §3）+ SSoT 依赖声明块；80 已有 SSoT 声明，追加 09 §3 引用；110 声明为 SPDX 许可证策略唯一 SSoT；120 声明 IRON-9 v3 四层模型 + IPC magic 权威引用                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B5 第二轮全模块审查：11 维度（D1-D11）全部 PASS；63 文件计数一致；§4.5 族索引 9 条 + §4.6 子目录 README 5 条；发现并修复 2 处格式不一致——(1) 80-clang-format-enforcement.md "SSoT 声明" → "SSoT 依赖声明"（措辞统一）；(2) 10-coding-style/README.md §9 编号冲突（"版本历史"重编为 §11）；Rust 编号缺口已文档化，留待后续消解                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | B6 第二轮 V1-V12 验证：12/12 项全部 PASS；V1 编号唯一性（Rust 缺口已文档化）+ V2 SSoT 三层一致 + V3 §4.5 九条/§4.6 五条 + V4 90-terminology 登记 + V5 stub 正确 + V6 0 活跃旧路径 + V7 README 9 族索引引用 + V8 B1-B5 五条 changelog + V9 9/9 族索引文件存在 + V10 §4.6 五条 README + V11 6/6 扩展文档 SSoT 声明 + V12 9/9 族索引格式一致；第二阶段整合工作完成                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | Wave 2 NEW 命名整理：消除数字前缀冲突——根目录 3 组（01/05/09）+ 子目录 7 文件（00）；SSoT 族索引文件统一使用 200 段唯一编号（200-206 + 200-201 根目录）；09-license-strategy.md → 15-license-strategy.md（09-ssot-registry.md 保留 09 前缀体现权威地位）；§4.5 族索引登记表 + §4.6 子目录 README + §8.2 废弃文档登记同步更新                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | SPHARX 工程标准组 |
| 2026-07-12 | 0.1.1 | **SSoT 重新设计**：取消三层镜像模型（00 §3 + 07 §10 改为引用链接，消除 \~448 行重复）；删除 9 个族索引文件（200/201 根目录 + 200-206 子目录）+ 1 个 stub 文件（15-license-strategy.md 物理删除）；§4.5 族索引登记表整节废除；§4.6 子目录 README 登记更新（移除族索引导航引用）；§8.2 废弃文档登记更新（15-license-strategy.md 标记为已物理删除）；文件数从 63 减至 53。**设计原则**：只保留 09-ssot-registry.md 一个 SSoT 文档，废弃文档全部删除，文档不能越来越多，要精要准确                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | SPHARX 工程标准组 |
| 2026-07-13 | 0.1.1 | 10-coding-style 文件合并：16 份源文件合并为 5 份合并文档（C\_Cpp\_coding\_style.md / Rust\_coding\_style.md / Go\_coding\_style.md / scripting\_coding\_style.md / coding\_conventions.md），10-coding-style 文件数从 17 减至 6（含 README）；本注册表同步更新：§3 编号区间说明 + §3.2 章节标题与引用（C\_coding\_style\_standard.md → C\_Cpp\_coding\_style.md Part I）+ §4.7 子目录 README 登记表（naming\_conventions.md → coding\_conventions.md Part IV）+ §8 命名规范权威文档引用（naming\_conventions.md §4.4/§4.4.6/§1.10 → coding\_conventions.md Part IV §4.4/§4.4.6/§1.10）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | SPHARX 工程标准组 |
| 2026-07-13 | 0.1.1 | **顶层文件合并（B 组）**：8 份源文件合并为 2 份合并文档 + 1 份废弃文档删除。`01-coding-standards.md` 合并 5 份源文件（02-code-format→Part II / 03-code-style→Part III / 60-checkpatch-rule-map→Part IV / 70-kernel-doc-standard→Part V / 80-clang-format-enforcement→Part VI），692→3667 行；`05-development-process.md` 合并 3 份源文件（06-toolchain-and-automation→Part II / 08-compliance-checklist→Part III / 110-spdx-license-compliance→Part IV），666→2685 行；`100-deprecated-api-registry.md` 废弃删除（642 行）。本注册表同步更新：§3.6.7 标题（03-code-style→01 Part III）+ §4.2 OS-STD-FMT 权威文档（02→01 Part II）+ §4.3 OS-STD-STY 权威文档（03→01 Part III）+ §4.5 OS-STD-TOOL 权威文档与编号段归属表（06→05 Part II）+ §4.5 历史编号等价声明（06→05 Part II）+ §8.2 废弃文档登记（新增 9 条 + 15-license 重定向至 05 Part IV）。版本规则 IRON-8 验证通过（0 处违规）                                                                                                                                                                                                                                                                                                                                                                       | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v2.0-P1-01 修复：新增 §4.8 OS-FMT 闭源总纲代码格式规则章节，登记 OS-FMT-001\~007（Tab 8 缩进 / 80 列行宽 / K\&R 大括号 / 单语句大括号 / 空格规则 / 行尾禁止空白 / clang-format 配置）；规则源自闭源总纲 §3.1-3.6                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v2.0-P1-03 修复：新增 §2.1 ES-SEL4 编号范围声明，澄清三段作用域——ES-SEL4-1\~5 架构层强制借鉴（已注册 OS-IRON-012）/ ES-SEL4-6\~10 P2 增强建议（已注册闭源总纲附录 D）/ ES-SEL4-11\~43 研究范围编号（`_research_0.2.5/01-sel4-deep-analysis.md` 研究范围，非工程标准范围，开源文档引用时须标注"研究范围参考"）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v1.0-P1-02 修复：OS-KER-005 跳号补齐——标记为已废弃，原定义为"C 内核代码 Tab 8 字符缩进"，因格式规则重新组织（内容迁移至 OS-STD-FMT-001 / 闭源 OS-FMT-001）而废弃                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v1.0-P2-04 修复：新增 §8.3 文档计数三作用域模型登记——\~79 文档（P0 基线，"文档体系完成"上下文）/ \~85 文档（工程基线，"0.1.1 合计/总产出"上下文）/ \~122 文档（全量展开，"全部三大支柱"上下文）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | v1.0-P2-05 修复：附录 A 变更日志补齐本次修复会话（2026-07-14）全部条目——OS-FMT-001\~007 注册 / ES-SEL4-11\~43 研究范围声明 / OS-KER-005 废弃 / 文档计数三作用域模型注册                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **P0 修复：消解 OS-SEC 三重编号冲突**——历史 OS-SEC 编号在 110-security/（LSM/Cupolas 安全规则 001\~015）、C\_Cpp\_coding\_style.md（C 安全编码 010\~027）、Rust\_coding\_style.md（Rust 安全编码 001\~003+030\~047）三处有完全不同的定义且未在 SSoT 登记。编号空间重新分配：OS-SEC-001\~099 LSM 安全规则（不变）/ OS-SEC-100\~199 C 安全编码（+100 偏移）/ OS-SEC-200\~299 Rust 安全编码（+200 偏移）；新增 §9 OS-SEC 安全工程规则章节登记全部 54 个编号；同步修改 C\_Cpp\_coding\_style.md 18 个编号（010\~027→110\~127）+ Rust\_coding\_style.md 21 个编号（001\~003→201\~203, 030\~047→230\~247）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：补全 Rust OS-STD-045\~046 登记**——§4.5 OS-STD-RUST 表新增 OS-STD-045（FFI 类型映射——`core::ffi`/`kernel::ffi` C 兼容类型，Part I §8.2）+ OS-STD-046（FFI 所有权语义——`#[ownership]` 标注，Part I §8.3）；Rust 编码规则 OS-STD-030\~046 全部 17 项完整登记，消除 Rust 编号登记缺口                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：文档化 Linux 6.6 内核基线品味哲学 8 项隐性智慧**——01-coding-standards.md 新增 §8.5 章节（OS-STD-STY-005\~012）：信任读者/短函数/简洁文件头/EXPORT\_SYMBOL 紧跟/include 语义分组/`__` 前缀/`__read_mostly`/`__randomize_layout`；每项含 Linux 6.6 内核基线源码证据（`mutex.c`/`core.c`/`sched.h`）+ 好坏代码示例；SSoT §4.3 同步登记 8 个新编号；§10 五维映射表更新 A-1/A-2/A-4/E-1/E-3 映射                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：消解 ES-SEL4-9/10 状态分类模糊**——闭源总纲 §12.8 引言文本区分"ES-SEL4-7/9/10 已在 0.1.1 阶段提前落地（已落地）"与"ES-SEL4-6/8 作为 1.0.1 阶段增量待落地（待落地）"；表格新增"状态"列明确标注每项落地状态，消除"P2 增强建议在 1.0.1 阶段增量落地"引言与"已落地"正文之间的语义冲突                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P1 修复：补齐 OS-KER-007/008 跳号废弃说明**——§3.1 新增 OS-KER-007（原"内核态禁 float，强制 q16.16 定点"，2026-07-12 迁移至 OS-STD-010）+ OS-KER-008（原"函数例外：开括号在下一行行首（K\&R）"，2026-07-12 与 OS-STD-047 合并迁移至 OS-STD-FMT-009）废弃说明条目；与 01-coding-standards.md Part II §0.2.1 迁移映射表一致，消除 OS-KER-001\~010 区间的跳号未标注问题，SSoT 内部一致性完整                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P0 修复：消除 DEFER\_INIT 设计层面延期至 1.0.1 的表述**——40-dataflows/03-ipc-flow\.md §2.5 io\_uring SETUP 标志表 L151/L158/L160 共 3 处"留待 1.0.1 评估"表述违反"不能有任何设计往后迁移"硬约束。修复为明确版本路线声明：`DEFER_INIT` 设计决策已在 0.1.1 完成（0.1.1 阶段不启用——Linux 6.6 基线不支持 ADR-001；1.0.1 阶段启用——升级 Linux 7.1 后 ADR-013），属版本路线规划非设计延期。OS-IPC-009 规则同步更新，启用策略分类由"延后启用"改为"版本路线"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P2 修复：清除 tests 简写残留**——在指代 tests-linux 子仓的上下文中，将 "tests" 简写改为全称 "tests-linux"。修复 10 个文件共 15 处：120-development-process/01-patch-lifecycle.md（L27/L389 共 3 处）、02-maintainer-hierarchy.md（L179）、README.md（L150）、10-architecture/04-engineering-baseline.md（L145）、05-adrs.md（L212/L234/L251）、130-roadmap/01-development-strategy.md（L182）、02-milestones-and-timeline.md（L113）、00-requirements/README.md（L17）、50-engineering-standards/00-engineering-standards-handbook.md（L55）、05-development-process.md（L174）、50-project-erp/README.md（L24）、50-project-erp/project\_erp.md（L1518）。验证 grep 确认零残留                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | SPHARX 工程标准组 |
| 2026-07-14 | 0.1.1 | **v1.0-P0 修复：消解 OS-STD-010 编号三重含义冲突**——01-coding-standards.md 中 OS-STD-010 存在三重含义：(1) L329 §6.2 整数类型选择 (u8/u32 vs \_\_u8/\_\_u32)；(2) L778 §0.2.1 迁移映射表"内核态禁 float"（从 OS-KER-007 迁移）；(3) L785 typedef 限制（从 OS-KER-003 扩展）。冲突消解：OS-STD-010 权威含义确定为"内核态禁 float，强制 airy\_q16\_t"（与 SSoT §3.1 + C\_Cpp\_coding\_style.md 一致）；整数类型选择 + typedef 限制迁移至新编号 OS-STD-CODE-032（初稿曾用 OS-STD-012，因 OS-STD-012 已被 60-driver-model/90-observability/05-development-process 三子系统跨域占用而放弃）；05-development-process.md L1861-1862 的 typedef 引用同步修正为 OS-STD-CODE-020                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **v1.0-P0 修复：消解 OS-STD-CODE-031 命名冲突**——OS-STD-CODE-031 被 01-coding-standards.md §6.2（整数类型选择，2026-07-14 从 OS-STD-010 让位引入）与 10-coding-style/C\_Cpp\_coding\_style.md §3.2（snake\_case 命名规则，更早存在）双重占用。消解方案：OS-STD-CODE-031 权威定义确定为 snake\_case 命名规则（C\_Cpp\_coding\_style.md §3.2 权威定义源）；整数类型选择规则让位至 OS-STD-CODE-032。修改 4 处：01-coding-standards.md §6.2 L329 规则定义 + §0 速查表 L656 + §0.2.1 迁移映射表 L784/L785（共 4 处 OS-STD-CODE-031→OS-STD-CODE-032）；09-ssot-registry.md §4.2 L349 新增 OS-STD-CODE-031（snake\_case 命名规则）与 OS-STD-CODE-032（整数类型选择）双编号登记                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **深审维度 C 修复：ES-SEL4-11\~43 标注 + §8.4 前瞻性预留免责声明**——(1) fix5: 10-architecture/04-engineering-baseline.md §8.4 追加前瞻性预留免责声明，将 PREEMPT\_LAZY / Rust 正式转正 / MGLRU 2.0 / XFS 自愈合 / eBPF 签名验证等 Linux 7.x 特性明确标注为 2.x.x 前瞻性预留（对齐 BAN-361/IRON-10 禁词规则）。(2) fix6/7/8: 03-microkernel-strategy.md + 20-modules/01-kernel.md + 05-development-process.md 三份文档统一追加 ES-SEL4 编号作用域声明，所有 ES-SEL4-11\~43 引用统一标注"研究范围参考"（对齐 SSoT §2.1 L127 使用约定）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **深审 SSoT 章节重编号 + §13 新增 11 个子节**——(1) SSoT 章节重编号：原 §13 agentrt → §14（18 个子章节标题 + 17 行总览表格 + 3 处"第 10 章"引用 + L60 编号格式 + L1143 流程引用全部更新）。(2) §13 新增 11 个子节登记此前未登记的 OS-\* 前缀：§13.1 OS-OPS（\~81 条）/ §13.2 OS-IPC（10 条）/ §13.3 OS-CHK-DOC（10 条）/ §13.4 OS-CHK-CODE（6 条）/ §13.5 OS-CHK-IRON（5 条）/ §13.6 OS-DEV（\~60 条）/ §13.7 OS-BUILD（24 条）/ §13.8 OS-DRV（\~26 条，含 OS-DRV-IRON-PLAT 约束别名登记）/ §13.9 OS-OBS（22 条）/ §13.10 OS-MM（2 条）/ §13.11 OS-TST（4 条）。OS-DRV/OS-OBS 编号空间隔离声明追加（与 OS-STD-DRV/OS-STD-OBS 区分）。共登记 \~252 条此前未登记规则                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **深审 auditB 修复：§4 新增 4 个 OS-STD 子域登记**——(1) §4.9 OS-STD-DOC 文档规范（kernel-doc，14 条，OS-STD-DOC-001\~014，权威定义 01-coding-standards.md Part IV）。(2) §4.10 OS-STD-SEC 安全编码规范（2 条，OS-STD-SEC-010\~011，权威定义 110-security/02-landlock-sandbox.md，含编号空间隔离声明与 §9 OS-SEC 区分）。(3) §4.11 OS-STD-SPDX 许可证标识规范（3 条，OS-STD-SPDX-001\~003，权威定义 05-development-process.md Part IV）。(4) §4.12 OS-STD-TEST 测试规范（2 条，OS-STD-TEST-010\~011，权威定义 80-testing/02-kselftest.md，含编号空间隔离声明与 §12 OS-TEST 区分）。补全 OS-STD-GOV-010 + OS-STD-OBS-010/011 登记                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **深审 auditD 修复：§1.5 登记方式与验证方法 + §6 OS-ACC 摘要登记声明 + 范围扩展**——(1) §1.5 新增"登记方式与验证方法"章节：全量登记 vs 摘要登记双模式定义 + 17 子节摘要登记覆盖声明表（共 \~924 条编号）+ OS-STD 裸编号全量覆盖声明（OS-STD-001\~218）+ 验证工具约定（区间表达式展开后比对）。(2) §6 OS-ACC 追加摘要登记覆盖声明（OS-ACC-001\~110 全部 110 条）。(3) §1.5 表格 OS-OPS 范围扩展 139→140 / OS-STD-TOOL 范围扩展 158→161 / OS-STD-GOV 范围扩展 9→10 / OS-STD-OBS 范围扩展 12\~16→10\~16。(4) auditD 验证通过：924 个主题文档使用编号全部在 SSoT 声明覆盖范围内，0 未覆盖                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **v1.0-P0 修复：消解跨子系统 OS-STD-0xx 编号冲突**——OS-STD-0xx 编号被 4 个子系统文档独立占用未在 SSoT 登记，造成系统性 SSoT 违规：(1) OS-STD-001\~009 被 110-security 两个文件重复定义（LSM/Landlock）；(2) OS-STD-012 被 60-driver-model（sysfs\_emit）/90-observability（BPF 验证器日志）/05-development-process（sparse 警告）三处独立占用；(3) OS-STD-013/014/015 被 60-driver-model 与 90-observability 跨域冲突；(4) OS-STD-016/017 被 90-observability 独立占用。消解方案：① 110-security 保留 OS-STD-001\~009 权威定义并登记 SSoT；② 05-development-process 保留 OS-STD-012 权威定义（构建规则）并登记 SSoT；③ 60-driver-model 的 OS-STD-012\~015 → OS-STD-DRV-012\~015（驱动模型子域）；④ 90-observability 的 OS-STD-012\~016 → OS-STD-OBS-012\~016（可观测性子域）；⑤ OS-STD-017（编号管理元规则）提升为 OS-IRON-013（全局元规则归入 IRON 系列）。新增 §9.4 跨子系统 OS-STD-0xx 编号登记章节登记全部 20 个编号；§9.4 末尾新增编号空间声明明确各前缀子域归属。修改文件：60-driver-model/01-device-model.md（5 处 OS-STD-0xx→OS-STD-DRV-0xx）、90-observability/02-ebpf-probes.md（6 处 OS-STD-0xx→OS-STD-OBS-0xx + 1 处 OS-STD-017→OS-IRON-013）、09-ssot-registry.md（新增 §9.4 章节 + 编号空间声明）                                                                                                                                       | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **v2.0-P0 修复：消解 OS-IRON-013 双重登记冲突**——2026-07-15 提升 OS-STD-017 → OS-IRON-013 时，未检测到 §2 表格 L91 已登记"OS-IRON-013 = 8 子仓独立 git 仓库 + submodule"（来源 04 §13），导致同一编号被两条规则同时占用：(1) §2 L91 = 8 子仓 submodule（正确登记，与 04 工程思想 + 00 手册镜像一致）；(2) §4.4 OS-STD-OBS 表格 L697 = 编号管理元规则（重复登记）。消解方案：保留 §2 L91 的"8 子仓 submodule"含义不变，将 L697 的"编号管理元规则"重新编号为 **OS-IRON-015**（紧接 OS-IRON-014 之后，连续编号），§2 表格 L92 之后新增 OS-IRON-015 登记行；§4.4 OS-STD-OBS 表格 L697 的重复表格行改为"交叉引用说明"块（不再以表格行形式登记），消除同一编号在两处表格重复登记的 SSoT 违规。本次修复与上一条记录（2026-07-15 v1.0-P0）的对应关系：上一条记录中的"⑤ OS-STD-017 提升为 OS-IRON-013"实际执行为"提升为 OS-IRON-015"（因 OS-IRON-013 已被占用），上一条记录的描述文本不再修改但以本条记录为准                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **SSoT 升级：AirymaxOS → Airymax 全局**——标题从"AirymaxOS 全局规则编号注册表"升级为"Airymax 全局规则 SSoT 注册表"；文档定位扩展为"Airymax 全项目（agentrt 用户态运行时 + AirymaxOS/agentrt-linux 操作系统）"；头部新增双编号体系声明（AirymaxOS `OS-*` 前缀第 2-9 章 + agentrt 无前缀第 10 章）+ IRON-9 v3 四层共享模型引用；§1.1-§1.3 扩展核心原则/使用规则/编号格式覆盖双编号体系，§1.2 新增"编号体系隔离"规则禁止 `OS-IRON-001` 与 `IRON-1` 混用，§1.3 新增 agentrt 编号格式 `<前缀>-<NN>` 声明；新增第 10 章 agentrt 规则编号体系（§10.1-§10.18 共 18 节），全量登记 17 类规则：IRON-1\~10（10 条全量）/BAN-001\~361（摘要 16 区间）/STD-01\~08（8 条全量，含 STD-02 IPC 128 字节消息头）/ACC 149 项（摘要 6 区间）/FOUND-01\~10（10 条全量）/SPLIT-01\~08（8 条全量）/PROD-01\~06（6 条全量）/ARC-01\~08（8 条全量，含方案 A 接口反转）/LC-01\~08（8 条全量，含 AGPL+Apache 双许可证）/PRT-01\~18（摘要 3 区间）/LOG-01\~26（摘要 6 区间）/PATH-BAN-1\~5（5 条全量）/L-01\~53（摘要 2 区间）/CROSS-01\~06（6 条全量）/REQ-01\~08（8 条全量）/W01-W30（摘要 3 区间）/SP01-SP37（摘要 3 区间）；小类全量登记 10 类共 77 条（IRON 10/STD 8/FOUND 10/SPLIT 8/PROD 6/ARC 8/LC 8/PATH-BAN 5/CROSS 6/REQ 8）+ 大类摘要登记 7 类共 674 条（BAN 361/ACC 149/PRT 18/LOG 26/L 53/W 30/SP 37）= 总计 17 类 751 条 agentrt 规则；§10.18 新增 agentrt 规则编号新增流程（RFC→评审→注册三步）+ 双编号体系同步声明（agentrt 新增规则与 AirymaxOS 存在语义同源关系时须同时登记 `OS-*` 编号） | SPHARX 工程标准组 |
| 2026-07-15 | 0.1.1 | **P0 修复：补登 42 个未在 SSoT 登记的规则编号 + 重编号原 §10 → §13**——深层审查发现 OS-ARCH-001\~010（10-architecture/ 5 文档）/ OS-IFACE-001\~010（30-interfaces/ 5 文档）/ OS-TEST-001\~022（80-testing/ 2 文档 + 10-architecture/07-directory-structure.md L329 引用 OS-TEST-010）共 42 个 `OS-*` 前缀规则编号在主题文档中使用但完全未在 SSoT 登记，构成 P0 级 SSoT 违规（违反 §1.2"编号分配权"——所有编号分配权仅属于 SSoT）。修复方案：① 新增 §10 OS-ARCH 架构工程规则章节摘要登记 OS-ARCH-001\~010；② 新增 §11 OS-IFACE 接口工程规则章节摘要登记 OS-IFACE-001\~010；③ 新增 §12 OS-TEST 测试工程规则章节摘要登记 OS-TEST-001\~022；④ 原 §10 agentrt 规则编号体系重编号为 §13（§10.1-§10.18 共 18 个子章节标题 + 17 个总览表引用全部同步更新）；⑤ 头部双编号体系声明更新为"AirymaxOS 第 2-12 章 + agentrt 第 13 章"；⑥ TOC 同步追加 §10/§11/§12/§13 四个条目；⑦ 00-engineering-standards-handbook.md §3 章节引用列表同步追加 §10-§13。本次修复使 AirymaxOS `OS-*` 编号体系（OS-IRON/OS-KER/OS-STD/OS-BAN/OS-ACC/OS-ABI/OS-SEC/OS-ARCH/OS-IFACE/OS-TEST 共 10 个系列）与 agentrt 17 类无前缀编号体系（IRON/BAN/STD/ACC/FOUND/SPLIT/PROD/ARC/LC/PRT/LOG/PATH-BAN/L/CROSS/REQ/W/SP）在 SSoT 中获得完全登记覆盖，根除"主题文档使用未登记编号"的系统性 SSoT 违规                                                                                                                               | SPHARX 工程标准组 |

| 2026-07-17 | 0.2.0 | **v0.2.0 升级：SSoT 三层模型 + 技术点单一权威源清单 + CI 强制校验机制**——(1) 新增 §1.6 SSoT 三层模型（Tier 0 不可变权威源 / Tier 1 强权威源 / Tier 2 常规权威源），定义层级判定规则与层间优先级。(2) 新增 §1.7 技术点单一权威源清单，登记 11 个核心技术点（错误码/日志类型/IPC 协议/调度策略/安全模型/微内核策略/IRON-9 模型/Unify Design/[DSL] 降级层/内核基线/SCHED_AGENT）的 Tier 层级与唯一权威文档。(3) 新增 §1.8 CI 强制校验机制，登记 6 个 CI workflow（ci-kernel/sc-dual-ci/ssot-validate/mgmt-orchestrator/nightly/release）覆盖三层模型强制力。本次升级配套创建 P2 级文档：20-modules/12-logger-daemon-module.md（Logger Daemon 模块设计）+ 20-modules/13-printk-bridge.md（printk 桥接设计）+ agentrt 50-engineering-standards/11-log-convergence-plan.md（日志收敛计划）+ 12-error-code-migration-plan.md（错误码迁移计划）；升级 03-microkernel-strategy.md / 07-ipc-fastpath.md / 03-capability-model.md 至 v1.0（新增附录：Unify Design 5 模块融入 + sched_tac + [DSL] 降级层 + Capability 缓存/Reconciliation/MDB 完整性约束） | SPHARX 工程标准组 |
