Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# agentrt-linux（AirymaxOS）整体目录结构设计 v2.0

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）整体目录结构的唯一权威设计文档。本文件定义 1 个管理仓 + 8 个子仓的目录拓扑、文件级职责、跨子仓依赖关系、[SC] 共享契约头文件物理宿主、IRON-9 v3 四层模型落地路径、12 daemon 命名统一规范、ALK 6.6 内核子仓内部结构。所有 agentrt-linux 子仓的目录规划、文件创建、构建配置必须以本文件为唯一权威源。\
> **文档版本**：v2.0（取代 v1.1）\
> **最后更新**：2026-07-18\
> **上级文档**：[agentrt-linux 设计文档总纲](README.md)\
> **作者**：开源极境工程与规范委员会（OpenAirymax Engineering and Standardization Committee）\
> **理论根基**：乔布斯/艾夫"简约就是美"设计哲学 + seL4 Liedtke minimality principle + Linux 6.6 内核基线工程思想\
> **编号权威**：[09-ssot-registry.md §3](../50-engineering-standards/09-ssot-registry.md)\
> **SPDX-License-Identifier**：CC-BY-NC-4.0\
> **SSoT 依赖声明**：本文件引用 IRON-9 v3 四层模型（权威源 [06-iron9-shared-model.md](06-iron9-shared-model.md)）、Unify Design 5 模块（权威源 [10-unify-design.md](10-unify-design.md)）、[SC] 10 头文件物理宿主（权威源 [120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md)）、12 daemon 命名规范（权威源 [90-terminology.md](../50-engineering-standards/90-terminology.md)）、Micro-Supervisor 内核模块（权威源 [09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)）、纯 C LSM 设计（权威源 [07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md)）。

---

## §0 文档元数据与变更记录

### 0.1 版本演进

| 版本 | 日期 | 主要变更 | 状态 |
|------|------|---------|------|
| v1.0 | 2026-07-10 | 初版目录结构，8 子仓 + 文件级树 | 已废弃 |
| v1.1 | 2026-07-14 | 增加 [SC] 头文件宿主说明、daemon 命名表 | 已废弃 |
| **v2.0** | **2026-07-18** | **基于乔布斯/艾夫"简约就是美"哲学重构；9 个设计决策全部解决；§0-§12 完整结构；[SC] 10 头文件清单；IRON-9 v3 四层模型落地；12 daemon 命名统一为 `_d` 后缀；ALK 6.6 内核子仓内部结构；与 agentrt 对比；工程美学自评；SSoT 登记** | **当前权威** |

### 0.2 v1.1 → v2.0 主要差异

1. **结构重整**：从 §1-§6 扩展为 §0-§12 共 13 节，覆盖设计哲学、顶层结构、设计决策、8 子仓详树、[SC] 头文件清单、IRON-9 v3 落地、daemon 命名、跨子仓依赖图、ALK 6.6 内核内部、agentrt 对比、工程美学自评、SSoT 登记。
2. **9 决策闭环**：v1.1 仅解决 5 个问题（A-E），v2.0 补齐 4 个决策（F-I），覆盖 [SC] 头文件补齐、最小可编译骨架、scripts/tools 划分、跨子仓共享代码组织。
3. **daemon 命名统一**：v1.1 存在 `macro_superv` / `logger_daemon` / `config_daemon` 与 `*_d` 混用问题，v2.0 统一为 12 个 `_d` 后缀。
4. **[SC] 10 头文件清单**：v1.1 截断于 ipc.h 第 3 行，v2.0 完整列出 10 头文件的物理宿主、职责、magic、include 路径。
5. **IRON-9 v3 落地**：新增 §6 详细说明 [SC]+[SS]+[IND]+[DSL] 四层在目录结构中的物理映射。
6. **ALK 6.6 内核内部**：新增 §9 详述 Model A 完整 fork 的目录组织（arch/ + include/ + corekern/ + kernel/kernel/superv/ + ...）。
7. **工程美学自评**：新增 §11 基于 5 条简约原则对目录结构进行自评，量化每条原则的达成度。

### 0.3 阅读对象

- 架构师：理解顶层结构与跨子仓依赖
- 子仓维护者：理解本子仓目录拓扑与 [SC] 头文件引用方式
- 内核开发者：理解 ALK 6.6 内核子仓内部结构
- 工具链开发者：理解 scripts/ 与 tools/ 的边界
- CI/CD 工程师：理解 sc-dual-ci.yml 双向校验机制

---

## §1 设计哲学根基：乔布斯/艾夫"简约就是美"

### 1.1 哲学原文

> "Simplicity is the ultimate sophistication."
> —— Leonardo da Vinci（乔布斯引用于 1977 年 Apple II 宣传材料）

> "Design is not just what it looks like and feels like. Design is how it works."
> —— Steve Jobs（2003 年纽约时报访谈）

> "Simplicity is not the absence of clutter, that's a consequence of simplicity. Simplicity is somehow essentially describing the purpose and place of an object and product."
> —— Jony Ive（2013 年时代周刊访谈）

> "A concept is tolerated inside the microkernel only if moving it outside the kernel, i.e., permitting competing implementations, would prevent the implementation of the system's required functionality."
> —— Jochen Liedtke（L4 微内核创始人，seL4 设计思想源头）

### 1.2 哲学要义与工程映射

乔布斯/艾夫的"简约就是美"并非表面极简（删减目录层级），而是**深入理解产品本质后实现的真正简约**。Liedtke minimality principle 与之东西方呼应——内核只保留不可再分的原子机制。本目录结构遵循以下 5 条工程原则：

| 原则 | 乔布斯/艾夫表述 | 工程映射 | 目录结构体现 |
|------|---------------|---------|------------|
| **P1 本质优先** | "深入理解产品的目的与位置" | 每个目录必须能一句话说明其"为什么存在" | 8 子仓均对应一个不可合并的职责域 |
| **P2 原子不可分** | "简约不是删减，而是描述本质" | 目录不可再合并，合并会丧失职责清晰度 | services 内部分层但不拆分（保持 8 子仓） |
| **P3 单一宿主** | "每个对象有唯一目的与位置" | [SC] 头文件唯一物理宿主 kernel/include/uapi/linux/airymax/ | 禁止物理副本，-I 引用 |
| **P4 内外一致** | "看不见的部分应如外观一样精致" | 内部目录结构与对外文档一致 | 子仓公共骨架统一（README/LICENSE/MAINTAINERS/...） |
| **P5 无冗余** | "简约是本质的描述" | 跨子仓共享代码仅 [SC] 层，其余独立 | [IND] 完全独立，无共享库 |

### 1.3 与 seL4 Liedtke minimality 的对应

| seL4 原则 | agentrt-linux 目录映射 |
|----------|----------------------|
| 微内核只保留原子机制 | kernel/kernel/superv/ 只放 Micro-Supervisor（5 个 .c） |
| 服务用户态化 | services/daemons/ 放 12 个 daemon |
| 机制在内核，策略在用户态 | Micro-Supervisor（机制）+ Macro-Supervisor（策略） |
| capability-based security | security/capability/ + security/airy_lsm/ |
| 消息传递通信 | kernel/ipc/ + services/userland/ipc/ |

### 1.4 反模式警示

本设计**明确反对**以下表面简约而实质复杂的做法：

- ❌ 为减少顶层目录数而合并职责不同的子仓（如将 cloudnative 并入 system）
- ❌ 为追求"扁平"而删除公共骨架文件（README/MAINTAINERS/CONTRIBUTING.md）
- ❌ 为"统一"而将 [SC] 头文件物理复制到各子仓（违反 P3 单一宿主）
- ❌ 为"简洁"而在子仓间创建共享库（违反 P5 无冗余，引入跨子仓耦合）
- ❌ 为"快速"而保留桩函数/桩文件（违反 IRON-2 禁止桩函数）

---

## §2 顶层结构

### 2.1 三层仓库拓扑

```
airymaxhub/                          # 伞仓（OpenAirymax 总入口）
├── agentrt/                         # 管理仓 1：用户态微核心运行时
│   ├── atoms/                       # 子仓：原子原语
│   ├── commons/                    # 子仓：公共库
│   ├── cupolas/                    # 子仓：安全穹顶
│   ├── daemons/                    # 子仓：daemon 共享
│   ├── gateway/                    # 子仓：网关
│   ├── protocols/                  # 子仓：协议
│   └── heapstore/                  # 子仓：堆存储
├── agentrt-linux/                   # 管理仓 2：AirymaxOS（本文档对象）
│   ├── kernel/                      # 子仓 1：ALK 6.6 内核
│   ├── services/                    # 子仓 2：用户态服务
│   ├── security/                    # 子仓 3：安全子系统
│   ├── memory/                      # 子仓 4：记忆子系统
│   ├── cognition/                   # 子仓 5：认知子系统
│   ├── cloudnative/                 # 子仓 6：云原生
│   ├── system/                      # 子仓 7：系统基础
│   └── tests-linux/                 # 子仓 8：测试
├── sdk/                             # 管理仓 3：SDK
├── ecosystem/                       # 管理仓 4：生态
└── products/                        # 管理仓 5：产品
```

### 2.2 agentrt-linux 管理仓结构

```
agentrt-linux/                       # 管理仓（独立 git repo，内含 8 个 submodule）
├── .gitmodules                      # 8 个 submodule 登记
├── README.md                        # 管理仓说明（含双许可证描述）
├── README_zh.md                     # 中文说明
├── LICENSE                           # AGPL-3.0-or-later OR Apache-2.0
├── NOTICE                            # 商标与版权声明
├── MAINTAINERS                      # 维护者名单
├── CONTRIBUTING.md                  # 贡献指南
├── CHANGELOG.md                     # 变更日志
├── .clang-format                    # 代码格式（全仓统一）
├── .editorconfig                    # 编辑器配置
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                   # 主 CI
│   │   ├── sc-dual-ci.yml           # [SC] 双向字节级校验（核心）
│   │   └── release.yml              # 发布流水线
│   └── ISSUE_TEMPLATE/
├── docs/                            # 管理仓文档（子仓各有自己的 Documentation/）
│   └── AirymaxOS/                   # 本设计文档体系
├── tools/                           # 跨子仓分析/调试工具（编译产物）
│   ├── ipc-profiler/                # IPC 性能分析器
│   ├── cap-inspector/               # Capability 检查器
│   ├── mem-tier-analyzer/           # 记忆分层分析器
│   └── sched-monitor/               # sched_tac 监控器
├── scripts/                         # 构建/引导/安装脚本
│   ├── build.sh                     # 全量构建
│   ├── bootstrap.sh                 # 开发环境引导
│   ├── install.sh                   # 安装到系统
│   └── gen-version.sh               # 版本号生成
├── kernel/                          # submodule 1（[SC] 头文件物理宿主）
├── services/                        # submodule 2
├── security/                        # submodule 3
├── memory/                          # submodule 4
├── cognition/                       # submodule 5
├── cloudnative/                     # submodule 6
├── system/                          # submodule 7
└── tests-linux/                     # submodule 8
```

### 2.3 8 子仓职责矩阵

| # | 子仓 | 职责一句话 | IRON-9 主层 | 许可证 |
|---|------|----------|-----------|--------|
| 1 | kernel | ALK 6.6 完整 fork + [SC] 头文件宿主 + Micro-Supervisor | [SC]+[IND] | GPL-2.0-only |
| 2 | services | 12 daemon + 用户态 VFS/Net/Drivers/IPC | [SS]+[IND] | AGPL-3.0 OR Apache-2.0 |
| 3 | security | 纯 C LSM + capability + Cupolas | [IND] | AGPL-3.0 OR Apache-2.0 |
| 4 | memory | MemoryRovol L1-L4 + CXL + PMEM + MGLRU | [IND]（含商业 EULA） | SPHARX Commercial EULA v1.0 |
| 5 | cognition | CoreLoopThree + Thinkdual + LLM + Wasm | [IND] | AGPL-3.0 OR Apache-2.0 |
| 6 | cloudnative | K8s CRD + containerd-shim + CNI + agentctl | [IND] | AGPL-3.0 OR Apache-2.0 |
| 7 | system | RPM/dnf + 配置 + shell + 监控 + DevStation | [IND] | AGPL-3.0 OR Apache-2.0 |
| 8 | tests-linux | 单元/集成/形式化/soak/chaos/benchmark | [SS]+[IND] | AGPL-3.0 OR Apache-2.0 |

---

## §3 设计决策（9 个关键问题解决）

本节针对已识别的 9 个关键设计问题，基于乔布斯/艾夫"简约就是美"哲学与 15 项项目硬约束，逐一做出最终决策。每条决策记录问题陈述、备选方案、决策依据、最终选择、落地约束。

### 3.1 决策 A：services 子仓职责过广（问题 1）

**问题陈述**：v1.1 的 services 子仓包含 6 大块（VFS/Net/Drivers/daemons/systemd/IPC），职责过广，违反 P1 本质优先原则。

**备选方案**：
- A1（保留 8 子仓 + 内部分层强化）：services 内部分 `daemons/` + `userland/{vfs,net,drivers,ipc}` + `systemd/`，不拆分子仓
- A2（拆分为 services + userland 2 子仓）：新增 userland 子仓承载 VFS/Net/Drivers，services 仅留 daemons
- A3（拆分为 3 子仓）：services + userland + ipc 三个独立子仓

**决策依据**：
1. 8 子仓结构是项目硬约束（约束 4），不可变更
2. services 的 6 块虽职责不同，但都属于"用户态服务"范畴（P2 原子不可分）
3. 拆分会引入跨子仓 IPC 调用，增加耦合（违反 P5 无冗余）
4. 内部分层即可清晰隔离职责，无需拆分

**最终选择：A1（保留 8 子仓 + 内部分层强化）**

**落地约束**：
- services/daemons/：12 个 daemon，每个独立子目录
- services/userland/vfs/：用户态 VFS 服务（与 kernel/fs 互补）
- services/userland/net/：用户态网络栈
- services/userland/drivers/：用户态驱动（部分驱动用户态化）
- services/userland/ipc/：io_uring IPC 用户态库
- services/systemd/：12 个 systemd unit 文件

### 3.2 决策 B：cloudnative 与 system 边界不清（问题 2）

**问题陈述**：v1.1 中 cloudnative 与 system 职责重叠，如 airymaxmon 归属、agentctl 归属、监控归属不清。

**备选方案**：
- B1（按职责本质划分）：cloudnative = 编排与运行时扩展；system = OS 发行版基础
- B2（按用户面/系统面划分）：cloudnative = 面向 K8s；system = 面向本机
- B3（合并为一个子仓）：cloudsystem 合并

**决策依据**：
1. P1 本质优先：cloudnative 的本质是"将 AirymaxOS 作为 K8s 节点运行"；system 的本质是"OS 发行版基础"
2. K8s CRD/containerd-shim/CNI 是云原生编排面，与 RPM/dnf/shell/DevStation 等 OS 基础设施职责不同
3. airymaxmon 属于 OS 监控（系统面），归 system；agentctl 属于编排入口（编排面），归 cloudnative
4. 合并会导致子仓职责模糊（违反 P2 原子不可分）

**最终选择：B1（按职责本质划分）**

**落地约束**：
- cloudnative/：K8s CRD + containerd-shim + CNI + OCI + agentctl + DPU/IPU + super-node-os + OpenTelemetry
- system/：package-manager（RPM/dnf）+ config + shell + base-libs + monitoring（含 airymaxmon）+ devstation + packaging

### 3.3 决策 C：daemon 命名不一致（问题 3）

**问题陈述**：v1.1 的 12 daemon 命名混乱：`macro_superv` / `logger_daemon` / `config_daemon` 与 `*_d` 混用，违反 P4 内外一致。

**备选方案**：
- C1（统一 `_d` 后缀）：全部 12 daemon 改为 `*_d`
- C2（统一 `_daemon` 后缀）：全部改为 `*_daemon`
- C3（按 Unix 传统保留混合）：保留现状

**决策依据**：
1. P4 内外一致：命名必须统一
2. Unix 传统（syslogd/sshd/cron）与现代 systemd（*.service）均倾向短后缀
3. `_d` 已被多数 daemon 使用（9/12），改造成本最低
4. 90-terminology.md 已登记废弃名：`llm_d→cogn_d`、`tool_d→dev_d`、`market_d→gateway_d`

**最终选择：C1（统一 `_d` 后缀）**

**落地约束**：12 daemon 最终命名（详见 §7）：
| 旧名 | 新名 | systemd unit |
|------|------|-------------|
| macro_superv | **macro_d** | agentrt-macro.service |
| logger_daemon | **logger_d** | agentrt-logger.service |
| config_daemon | **config_d** | agentrt-config.service |
| gateway_d | gateway_d（不变） | agentrt-gateway.service |
| sched_d | sched_d（不变） | agentrt-sched.service |
| vfs_d | vfs_d（不变） | agentrt-vfs.service |
| net_d | net_d（不变） | agentrt-net.service |
| mem_d | mem_d（不变） | agentrt-mem.service |
| cogn_d | cogn_d（不变） | agentrt-cogn.service |
| sec_d | sec_d（不变） | agentrt-sec.service |
| audit_d | audit_d（不变） | agentrt-audit.service |
| dev_d | dev_d（不变） | agentrt-dev.service |

### 3.4 决策 D：daemon 与 agentrt 重叠率仅 16.7%（问题 4）

**问题陈述**：agentrt 有 7 子仓（atoms/commons/cupolas/daemons/gateway/protocols/heapstore），agentrt-linux 有 12 daemon，仅 gateway_d 一项与 agentrt gateway 直接对应（重叠 1/6 ≈ 16.7%），关系不清。

**备选方案**：
- D1（[SS] 语义同源 + [IND] 完全独立）：3 个 daemon 为 [SS]（agentrt 同源 OS 升级），9 个为 [IND]（agentrt-linux 专属）
- D2（全部 [IND]）：12 daemon 完全独立，与 agentrt 无关
- D3（全部 [SS]）：12 daemon 全部为 agentrt 同源

**决策依据**：
1. IRON-9 v3 四层模型要求明确每项的归属层
2. gateway_d/sched_d/cogn_d 在语义上与 agentrt gateway/atoms/sched 同源，是 OS 级升级
3. macro_d/logger_d/config_d/vfs_d/net_d/mem_d/sec_d/audit_d/dev_d 是 OS 专属（agentrt 无对应）
4. P3 单一宿主：[SS] 层语义同源但签名独立演进，不共享代码

**最终选择：D1（[SS] 语义同源 + [IND] 完全独立）**

**落地约束**：
- [SS] 3 daemon：gateway_d / sched_d / cogn_d（与 agentrt 同源 OS 升级）
- [IND] 9 daemon：macro_d / logger_d / config_d / vfs_d / net_d / mem_d / sec_d / audit_d / dev_d（agentrt-linux 专属）

### 3.5 决策 E：8/10 [SC] 头文件缺失（问题 5）

**问题陈述**：v1.1 仅 error.h 实际创建（195 行），其余 9 个 [SC] 头文件缺失，违反 IRON-2 禁止桩文件与 IRON-9 v3 [SC] 层定义。

**备选方案**：
- E1（全量创建 10 头文件）：在 kernel/include/uapi/linux/airymax/ 下创建全部 10 个 [SC] 头文件，0.1.1 内完成
- E2（分批创建）：0.1.1 创建 5 个，1.0.1 创建 5 个
- E3（仅创建 error.h）：保持现状

**决策依据**：
1. IRON-9 v3 [SC] 层定义 10 头文件为"完全共享代码"（约束 1）
2. IRON-2 禁止桩函数/桩文件，但允许"最小可编译实现"
3. 0.1.1 是唯一奠基版本（约束 14），必须完成全部 [SC] 头文件
4. 120-cross-project-code-sharing.md 已提供全部 10 头文件的完整 C 代码

**最终选择：E1（全量创建 10 头文件）**

**落地约束**：10 头文件在 0.1.1 内全部创建于 `kernel/include/uapi/linux/airymax/`（详见 §5）：
1. error.h（A-UEF，~195 行）
2. log_types.h（A-ULP，~80 行）
3. ipc.h（A-IPC，magic 0x41524531，~150 行）
4. sched.h（A-ULS，magic 0x41475453，~120 行）
5. memory_types.h（MemoryRovol，~60 行）
6. security_types.h（capability 41 ID，~100 行）
7. cognition_types.h（A-UCS，airy_q16_t，~70 行）
8. syscalls.h（24 槽位，~90 行）
9. uapi_compat.h（三路桥接，~80 行）
10. lsm_types.h（纯 C LSM 类型，~60 行）

### 3.6 决策 F：7/8 子仓为纯桩（问题 6）

**问题陈述**：v1.1 除 kernel 外，7 个子仓（services/security/memory/cognition/cloudnative/system/tests-linux）均为纯桩目录，无任何可编译代码，违反 IRON-2。

**备选方案**：
- F1（最小可编译骨架）：每子仓必须含 README/LICENSE/NOTICE/MAINTAINERS/.clang-format/.editorconfig/CONTRIBUTING.md/Documentation/ + 构建入口（CMakeLists.txt 或 Kbuild）+ 至少 1 个可编译入口文件（非桩）
- F2（仅骨架文件）：仅 README/LICENSE 等元数据，无代码
- F3（推迟到 1.0.1）：0.1.1 仅 kernel，其余 1.0.1 补齐

**决策依据**：
1. IRON-2 禁止桩函数/桩文件，但允许"最小可编译实现"
2. 0.1.1 是唯一奠基版本（约束 14），必须完成全部子仓骨架
3. P4 内外一致：内部目录结构与对外文档一致
4. 最小可编译骨架 ≠ 桩：必须有真实可编译代码（如 airy_lsm_init() 的实际实现，非空函数）

**最终选择：F1（最小可编译骨架）**

**落地约束**：每子仓必须包含：
```
<submodule>/
├── README.md                    # 职责、边界、依赖
├── LICENSE                      # 子仓特定许可证
├── NOTICE                       # 商标与版权
├── MAINTAINERS                  # 维护者
├── .clang-format                # 代码格式
├── .editorconfig                # 编辑器配置
├── CONTRIBUTING.md              # 贡献指南
├── Documentation/               # 子仓文档
│   └── README.md
├── CMakeLists.txt 或 Kbuild     # 构建入口（可编译）
└── <至少 1 个可编译入口文件>     # 真实代码（非桩）
```

**0.1.1 最小可编译入口文件示例**：
- services/daemons/macro_d/macro_d.c：Macro-Supervisor 用户态裁决主循环（真实实现，非空函数）
- security/airy_lsm/airy_lsm.c：DEFINE_LSM(airy) 注册 + 至少 1 个 LSM_HOOK_INIT
- memory/memoryrovol/mr_l1.c：L1 hot record 分配/释放（真实 alloc_pages + mmap）
- cognition/coreloopthree/clt_main.c：CoreLoopThree kthread 主循环（真实 kthread_run）
- cloudnative/agentctl/agentctl.c：agentctl 命令行入口（真实 main()）
- system/monitoring/airymaxmon.c：airymaxmon 主循环（真实读取 [SC] 状态）
- tests-linux/unit/test_ipc_magic.c：IPC magic 0x41524531 校验（真实断言）

### 3.7 决策 G：[SC] 头文件宿主与 [IND] 引用关系不清（问题 7）

**问题陈述**：v1.1 未明确 [SC] 头文件物理宿主与各子仓的引用关系，导致子仓可能各自复制头文件，违反 P3 单一宿主。

**备选方案**：
- G1（-I 直接引用，禁止物理副本）：所有子仓通过 `-I../kernel/include` 引用 kernel/include/uapi/linux/airymax/
- G2（git submodule 嵌套）：将 kernel/include/uapi/linux/airymax/ 作为独立 submodule 嵌套
- G3（构建时复制）：构建脚本将头文件复制到各子仓

**决策依据**：
1. P3 单一宿主：[SC] 头文件唯一物理宿主 kernel/include/uapi/linux/airymax/
2. OS-IRON-014 已登记：禁止物理副本
3. -I 引用是 C/C++ 标准做法，无运行时开销
4. sc-dual-ci.yml 双向校验确保两端字节相同

**最终选择：G1（-I 直接引用，禁止物理副本）**

**落地约束**：
- CMake 子仓：
  ```cmake
  include_directories(${CMAKE_SOURCE_DIR}/../kernel/include)
  ```
- Kbuild 子仓（security/memory/cognition 内核模块部分）：
  ```makefile
  ccflags-y += -I$(src)/../kernel/include
  ```
- 禁止 `cp kernel/include/uapi/linux/airymax/*.h <submodule>/include/`
- sc-dual-ci.yml 校验：`diff -r kernel/include/uapi/linux/airymax/ agentrt/../kernel/include/uapi/linux/airymax/`

### 3.8 决策 H：scripts/ 与 tools/ 边界不清（问题 8）

**问题陈述**：v1.1 未明确 scripts/ 与 tools/ 的职责划分，易混淆。

**备选方案**：
- H1（按用途划分）：scripts/ = 构建/引导/安装脚本（shell/python）；tools/ = 分析/调试/性能工具（编译产物）
- H2（按调用者划分）：scripts/ = CI 调用；tools/ = 开发者手动调用
- H3（合并为 tools/）：仅保留 tools/

**决策依据**：
1. P1 本质优先：scripts 本质是"自动化流程脚本"；tools 本质是"分析调试工具"
2. P2 原子不可分：两者职责不同，不可合并
3. Unix 传统：scripts/ 放 .sh/.py；tools/ 放可执行二进制
4. CI 调用 scripts/，开发者使用 tools/

**最终选择：H1（按用途划分）**

**落地约束**：
- scripts/（管理仓根目录）：
  - build.sh：全量构建（调用各子仓 CMake/Make）
  - bootstrap.sh：开发环境引导（安装依赖、初始化 submodule）
  - install.sh：安装到系统（/usr/local/bin、/etc/systemd/system）
  - gen-version.sh：版本号生成（从 git tag 推导）
  - sc-verify.sh：[SC] 双向字节级校验（sc-dual-ci.yml 调用）
- tools/（管理仓根目录）：
  - ipc-profiler/：IPC 性能分析器（C，编译为二进制）
  - cap-inspector/：Capability 检查器（C，编译为二进制）
  - mem-tier-analyzer/：记忆分层分析器（C，编译为二进制）
  - sched-monitor/：sched_tac 监控器（C，编译为二进制）

### 3.9 决策 I：跨子仓共享代码组织不清（问题 9）

**问题陈述**：v1.1 未明确跨子仓共享代码的组织方式，可能出现"共享库"导致耦合。

**备选方案**：
- I1（三层共享模型）：[SC] 头文件单一宿主；[SS] API 各子仓独立；[IND] 完全独立；禁止跨子仓共享库
- I2（共享库 libairymax/）：新增 libairymax/ 子仓承载共享代码
- I3（复制到各子仓）：各子仓自行复制所需代码

**决策依据**：
1. P5 无冗余：跨子仓共享代码仅 [SC] 层，其余独立
2. IRON-9 v3 四层模型：[SC] 完全共享，[SS] 语义同源，[IND] 完全独立
3. 共享库会引入跨子仓耦合，违反 P5
4. 复制代码会引入版本漂移，违反 P3 单一宿主

**最终选择：I1（三层共享模型）**

**落地约束**：
1. **[SC] 共享层**：10 头文件物理宿主 kernel/include/uapi/linux/airymax/，各子仓 -I 引用
2. **[SS] 语义同源层**：各子仓独立实现同名 API，签名可独立演进（如 agentrt `airy_ipc_send()` 与 agentrt-linux `airy_ipc_send()` 签名不同但语义同源）
3. **[IND] 完全独立层**：各子仓自有代码，无任何共享
4. **禁止**：
   - 禁止创建 libairymax/ 或类似共享库子仓
   - 禁止子仓间 `#include` 对方非 [SC] 头文件
   - 禁止子仓间链接对方 .o/.a
5. **[DSL] 降级生存层**：[SC] 头文件内 `#ifdef AIRY_SC_FALLBACK` 块，自包含

### 3.10 决策汇总表

| 决策 | 问题 | 选择 | 关键约束 |
|------|------|------|---------|
| A | services 过广 | A1 内部分层 | 保持 8 子仓 |
| B | cloudnative vs system | B1 职责本质 | cloudnative=编排，system=OS 基础 |
| C | daemon 命名 | C1 统一 _d | 12 daemon 全部 *_d |
| D | daemon 与 agentrt 重叠 | D1 [SS]+[IND] | 3 [SS] + 9 [IND] |
| E | [SC] 头文件缺失 | E1 全量创建 | 0.1.1 内 10 头文件 |
| F | 子仓纯桩 | F1 最小可编译骨架 | 非桩，真实代码 |
| G | [SC] 引用关系 | G1 -I 引用 | 禁止物理副本 |
| H | scripts vs tools | H1 按用途 | scripts=流程，tools=分析 |
| I | 跨子仓共享 | I1 三层模型 | 仅 [SC] 共享 |

---

## §4 8 子仓详细目录结构

### 4.1 子仓 1：kernel（ALK 6.6 内核）

> **职责**：Linux 6.6 LTS 完整 fork（Model A，直接写入上游源码树，无 patch 隔离）+ [SC] 10 头文件物理宿主 + Micro-Supervisor 内核模块 + sched_tac 内核侧 + io_uring IPC 内核侧 + airy_lsm 内核侧。
> **IRON-9 主层**：[SC]（10 头文件）+ [IND]（内核专属实现）
> **许可证**：GPL-2.0-only（与 Linux 内核一致）

```
kernel/
├── README.md                        # ALK 6.6 说明 + fork 模型 + [SC] 宿主声明
├── LICENSE                           # GPL-2.0-only
├── NOTICE                            # Linux 内核版权 + Airymax 修改声明
├── MAINTAINERS                      # ALK 维护者 + Airymax 修改点登记
├── .clang-format                    # Linux 内核格式
├── .editorconfig
├── CONTRIBUTING.md                   # 贡献指南（含上游回写策略）
├── Documentation/                    # ALK 文档
│   ├── README.md
│   ├── alk-fork-model.md            # Model A 完整 fork 说明
│   ├── sc-host-manifest.md          # [SC] 10 头文件宿主清单
│   └── upstream-sync.md             # 上游同步策略
│
├── arch/                             # Linux 6.6 原生（Model A 直接写入）
│   ├── x86/
│   │   ├── Makefile                 # 含 -mno-80387 禁浮点
│   │   ├── include/                 # x86 架构相关头文件
│   │   ├── kernel/                   # x86 内核核心
│   │   ├── mm/                       # x86 内存管理
│   │   └── ...
│   ├── arm64/
│   ├── riscv/
│   └── ...
│
├── include/                          # 头文件总目录
│   ├── uapi/                         # Linux UAPI（原生 + Airymax 扩展）
│   │   ├── linux/
│   │   │   ├── sched.h              # SCHED_NORMAL=0...SCHED_DEADLINE=6（原生）
│   │   │   ├── io_uring.h           # IORING_OP_URING_CMD 等（原生，D-8 对齐参考）
│   │   │   ├── airymax/             # ★ [SC] 10 头文件唯一物理宿主（OS-IRON-014，D-8 修复后对齐 OLK 6.6 UAPI 标准路径）
│   │   │   │   ├── error.h          # 1. A-UEF 统一错误码
│   │   │   │   ├── log_types.h      # 2. A-ULP 统一日志类型
│   │   │   │   ├── ipc.h            # 3. A-IPC（magic 0x41524531）
│   │   │   │   ├── sched.h          # 4. A-ULS（magic 0x41475453）
│   │   │   │   ├── memory_types.h   # 5. MemoryRovol L1-L4
│   │   │   │   ├── security_types.h # 6. capability 41 ID
│   │   │   │   ├── cognition_types.h # 7. A-UCS（airy_q16_t）
│   │   │   │   ├── syscalls.h       # 8. 24 槽位系统调用
│   │   │   │   ├── uapi_compat.h    # 9. 三路类型桥接
│   │   │   │   └── lsm_types.h      # 10. 纯 C LSM 类型
│   │   │   └── ...
│   │   └── ...
│   ├── asm-generic/                  # 通用汇编头（原生）
│   ├── kernel/                       # 内核内部头（原生）
│   └── airymax/                      # ★ Airymax 内核内部头（[IND] 独立层，非 UAPI；如 maintainer_types.h / build_types.h / kconfig_types.h）
│
├── corekern/                         # ★ Airymax 内核增强（Model A 直接写入）
│   ├── api/                          # 内核 API 增强
│   │   ├── airy_kern_api.c          # Airymax 内核 API 入口
│   │   └── airy_kern_api.h
│   ├── sched/                       # sched_tac 内核侧
│   │   ├── stc_policy.c             # stc_realtime/interactive/agent/batch 策略
│   │   ├── stc_dispatch.c           # SCHED_DEADLINE/SCHED_FIFO/EEVDF 派发
│   │   ├── stc_mcs_map.c            # seL4 MCS 语义映射
│   │   └── stc_stats.c              # 调度统计
│   ├── ipc/                          # io_uring IPC 内核侧
│   │   ├── airy_uring_cmd.c         # IORING_OP_URING_CMD 处理
│   │   ├── airy_ipc_ring.c          # Ring Buffer 管理
│   │   ├── airy_ipc_fastpath.c      # fastpath（unlikely(ring->frozen)）
│   │   └── airy_ipc_zero_copy.c     # registered buffer + mmap
│   ├── taskflow/                     # Agent 8 态生命周期
│   │   ├── airy_task_lifecycle.c    # INACTIVE→SPAWNING→...→DEAD
│   │   └── airy_task_desc.c         # 任务描述符（magic 0x41475453）
│   ├── memory/                       # 内核内存增强
│   │   ├── airy_mem_alloc.c         # alloc_pages(GFP_KERNEL) + mmap
│   │   └── airy_mem_tier.c          # L1-L4 分层
│   ├── time/                         # 时间服务
│   │   └── airy_time.c
│   ├── object/                      # 内核对象管理
│   │   └── airy_object.c
│   ├── locking/                      # 锁机制
│   │   └── airy_locking.c
│   ├── irq/                          # 中断管理
│   │   └── airy_irq.c
│   └── bpf/                          # eBPF struct_ops（非 [SC] 核心）
│       ├── airy_struct_ops.c        # struct airy_struct_ops_value
│       └── airy_bpf_probe.c         # 可观测性探针
│
├── kernel/kernel/superv/                    # ★ Micro-Supervisor 内核模块（5 个 .c）
│   ├── airy_superv_lsm.c            # DEFINE_LSM(airy) 注册 + LSM_ORDER_MUTABLE
│   ├── airy_cap_check.c             # v1.1 slowpath LSM 钩子（C-S9 失败接管）
│   ├── airy_ipc_freeze.c            # ring->frozen = true（smp_store_release）
│   ├── airy_die_notify.c            # die_notifier（priority INT_MAX）
│   ├── airy_eventfd.c               # eventfd_signal 非阻塞通知
│   └── Kbuild                       # superv 模块构建
│
├── log/                              # 内核日志（A-ULP 内核侧）
│   ├── airy_log_kern.c              # printk 8 级映射
│   ├── airy_log_ring.c             # 128B 记录 Ring Buffer（alloc_pages + mmap）
│   └── airy_log_persist.c           # PMEM 持久化
│
├── ipc/                              # IPC 内核基础设施（与 corekern/ipc 互补）
│   ├── airy_ipc_syscall.c           # 12 系统调用入口
│   └── airy_ipc_capability.c        # Capability 折叠 Native Word 校验
│
├── config/                           # 内核配置
│   └── Kconfig.alk                  # ALK 6.6 配置项
│
├── configs/                          # 预置配置
│   ├── defconfig                     # 默认配置
│   ├── defconfig-agent               # Agent 优化配置
│   └── defconfig-embedded           # 嵌入式配置
│
├── init/                             # Linux 6.6 原生 init
│   ├── main.c                        # 内核入口
│   └── ...
│
├── lib/                              # Linux 6.6 原生 lib
├── machine/                          # 机器相关
├── crypto/                           # Linux 6.6 原生 crypto
├── io_uring/                         # Linux 6.6 原生 io_uring（含 IORING_OP_URING_CMD）
├── fs/                               # Linux 6.6 原生文件系统
├── net/                              # Linux 6.6 原生网络栈
├── block/                            # Linux 6.6 原生块设备
├── drivers/                          # Linux 6.6 原生驱动
│   ├── accel/                        # GPU/NPU 加速器
│   ├── gpu/drm/scheduler/            # drm_sched GPU 调度
│   ├── cxl/                          # CXL 内存
│   ├── pmem/                         # PMEM 持久内存
│   └── ...
├── mm/                               # Linux 6.6 原生内存管理（含 MGLRU）
├── security/                         # Linux 6.6 原生 security 框架
│   └── airy/                         # 纯 C LSM 源码（与 security 子仓同源，此处为内核侧）
├── sound/                            # Linux 6.6 原生声卡
├── certs/                            # 证书
├── samples/                          # 示例
├── usr/                              # 用户空间辅助
├── virt/                             # 虚拟化
├── LICENSES/                         # Linux 内核许可证声明
├── rust/                             # Linux 6.6 Rust 支持（保留，Airymax 不使用）
├── Documentation/                    # Linux 6.6 文档
├── scripts/                          # Linux 6.6 构建脚本
└── tools/                            # Linux 6.6 工具
```

### 4.2 子仓 2：services（用户态服务）

> **职责**：12 daemon + 用户态 VFS/Net/Drivers/IPC + systemd 集成。
> **IRON-9 主层**：[SS]（daemon API 与 agentrt 同源）+ [IND]（VFS/Net/Drivers 独立实现）
> **许可证**：AGPL-3.0-or-later OR Apache-2.0

```
services/
├── README.md
├── LICENSE
├── NOTICE
├── MAINTAINERS
├── .clang-format
├── .editorconfig
├── CONTRIBUTING.md
├── Documentation/
│   ├── README.md
│   ├── daemons.md                   # 12 daemon 详解
│   └── userland.md                  # 用户态子系统
├── CMakeLists.txt                    # services 构建入口
│
├── daemons/                          # ★ 12 daemon（统一 _d 后缀，决策 C1）
│   ├── macro_d/                     # Macro-Supervisor 用户态裁决（[IND]）
│   │   ├── CMakeLists.txt
│   │   ├── macro_d.c                # 主循环（接收 eventfd + 裁决）
│   │   ├── macro_adjudicate.c       # 裁决逻辑（警告/降级/暂停/终止）
│   │   ├── macro_context.c          # 上下文查询
│   │   ├── macro_policy.c           # 策略加载
│   │   ├── macro_event.c            # 事件分发
│   │   ├── macro_state.c            # 状态机
│   │   ├── macro_rpc.c              # JSON-RPC 2.0 接口
│   │   ├── macro_d.h
│   │   └── macro_d_test.c
│   ├── logger_d/                    # 日志服务（[IND]）
│   │   ├── CMakeLists.txt
│   │   ├── logger_d.c               # 主循环
│   │   ├── logger_collect.c         # 日志收集（128B 记录）
│   │   ├── logger_persist.c        # PMEM 持久化
│   │   ├── logger_rotate.c         # 日志轮转
│   │   ├── logger_query.c          # 日志查询
│   │   └── logger_d.h
│   ├── config_d/                    # 配置服务（[IND]）
│   │   ├── CMakeLists.txt
│   │   ├── config_d.c              # 主循环
│   │   ├── config_loader.c         # 配置加载（YAML）
│   │   ├── config_watcher.c        # 配置热加载（inotify）
│   │   ├── config_validate.c       # 配置校验
│   │   └── config_d.h
│   ├── gateway_d/                  # 网关服务（[SS]，agentrt gateway 同源 OS 升级）
│   │   ├── CMakeLists.txt
│   │   ├── gateway_d.c              # 主循环
│   │   ├── gateway_jsonrpc.c       # JSON-RPC 2.0
│   │   ├── gateway_router.c        # 路由
│   │   ├── gateway_auth.c          # 鉴权
│   │   ├── gateway_tls.c           # TLS
│   │   └── gateway_d.h
│   ├── sched_d/                     # 调度服务（[SS]，agentrt atoms/sched 同源 OS 升级）
│   │   ├── CMakeLists.txt
│   │   ├── sched_d.c                # 主循环
│   │   ├── sched_stc.c             # stc_* 策略加载
│   │   ├── sched_policy.c          # 策略管理
│   │   ├── sched_stats.c           # 统计
│   │   ├── sched_cgroup.c          # cgroup cpuset 隔离
│   │   └── sched_d.h
│   ├── vfs_d/                       # VFS 服务（[IND]）
│   │   ├── CMakeLists.txt
│   │   ├── vfs_d.c                  # 主循环
│   │   ├── vfs_ops.c                # VFS 操作
│   │   ├── vfs_cache.c              # 缓存
│   │   ├── vfs_mount.c             # 挂载
│   │   ├── vfs_persist.c            # 持久化
│   │   └── vfs_d.h
│   ├── net_d/                       # 网络服务（[IND]）
│   │   ├── CMakeLists.txt
│   │   ├── net_d.c                  # 主循环
│   │   ├── net_stack.c              # 用户态网络栈
│   │   ├── net_socket.c            # Socket
│   │   ├── net_route.c             # 路由
│   │   ├── net_filter.c            # 过滤
│   │   └── net_d.h
│   ├── mem_d/                       # 记忆服务（[IND]，与 memory 子仓协作）
│   │   ├── CMakeLists.txt
│   │   ├── mem_d.c                  # 主循环
│   │   ├── mem_tier.c               # L1-L4 分层管理
│   │   ├── mem_reclaim.c           # 回收
│   │   ├── mem_cxl.c                # CXL 池化
│   │   ├── mem_pmem.c              # PMEM
│   │   └── mem_d.h
│   ├── cogn_d/                      # 认知服务（[SS]，agentrt atoms/cognition 同源 OS 升级）
│   │   ├── CMakeLists.txt
│   │   ├── cogn_d.c                 # 主循环
│   │   ├── cogn_clt.c              # CoreLoopThree 调度
│   │   ├── cogn_think.c            # Thinkdual 模式管理
│   │   ├── cogn_llm.c              # LLM 推理调度
│   │   ├── cogn_gpu.c              # GPU/NPU 调度
│   │   └── cogn_d.h
│   ├── sec_d/                       # 安全服务（[IND]）
│   │   ├── CMakeLists.txt
│   │   ├── sec_d.c                  # 主循环
│   │   ├── sec_cap.c                # Capability 管理
│   │   ├── sec_policy.c             # 策略
│   │   ├── sec_audit.c             # 审计
│   │   ├── sec_vault.c             # Vault backend
│   │   └── sec_d.h
│   ├── audit_d/                     # 审计服务（[IND]）
│   │   ├── CMakeLists.txt
│   │   ├── audit_d.c                # 主循环
│   │   ├── audit_collect.c         # 审计收集
│   │   ├── audit_chain.c           # 哈希链
│   │   ├── audit_persist.c         # 持久化
│   │   ├── audit_query.c           # 查询
│   │   └── audit_d.h
│   └── dev_d/                       # 设备/工具服务（[IND]）
│       ├── CMakeLists.txt
│       ├── dev_d.c                  # 主循环
│       ├── dev_drv.c                # 用户态驱动
│       ├── dev_tool.c              # 工具管理
│       ├── dev_plugin.c            # 插件
│       ├── dev_discover.c          # 设备发现
│       └── dev_d.h
│
├── userland/                         # 用户态子系统（与 kernel/fs/net/drivers 互补）
│   ├── vfs/                          # 用户态 VFS
│   │   ├── CMakeLists.txt
│   │   ├── uvfs_core.c              # 用户态 VFS 核心
│   │   ├── uvfs_fuse.c              # FUSE 集成
│   │   ├── uvfs_cache.c            # 缓存
│   │   └── uvfs.h
│   ├── net/                          # 用户态网络栈
│   │   ├── CMakeLists.txt
│   │   ├── unet_core.c              # 核心
│   │   ├── unet_dpdk.c             # DPDK 集成
│   │   ├── unet_xdp.c              # XDP
│   │   └── unet.h
│   ├── drivers/                      # 用户态驱动
│   │   ├── CMakeLists.txt
│   │   ├── udrv_core.c              # 核心
│   │   ├── udrv_vfio.c             # VFIO
│   │   └── udrv.h
│   └── ipc/                          # io_uring IPC 用户态库
│       ├── CMakeLists.txt
│       ├── airy_ipc_client.c        # 客户端库
│       ├── airy_ipc_server.c        # 服务端库
│       ├── airy_ipc_ring.c          # Ring Buffer 用户态
│       ├── airy_ipc_zero_copy.c    # 零拷贝
│       └── airy_ipc.h
│
└── systemd/                          # 12 systemd unit 文件
    ├── agentrt-macro.service        # macro_d（启动顺序 1）
    ├── agentrt-gateway.service      # gateway_d（1）
    ├── agentrt-sched.service        # sched_d（1）
    ├── agentrt-logger.service       # logger_d（2）
    ├── agentrt-vfs.service          # vfs_d（2）
    ├── agentrt-net.service          # net_d（2）
    ├── agentrt-cogn.service          # cogn_d（2）
    ├── agentrt-dev.service           # dev_d（3）
    ├── agentrt-config.service        # config_d（3）
    ├── agentrt-mem.service           # mem_d（4）
    ├── agentrt-audit.service         # audit_d（4）
    ├── agentrt-sec.service           # sec_d（5）
    └── agentrt.target                # 整体 target
```

### 4.3 子仓 3：security（安全子系统）

> **职责**：纯 C LSM（DEFINE_LSM(airy)）+ Capability 系统（41 ID + seL4 派生模型）+ Cupolas 安全穹顶 + Integrity + Keys。
> **IRON-9 主层**：[IND]（agentrt-linux 专属安全实现）
> **许可证**：AGPL-3.0-or-later OR Apache-2.0

```
security/
├── README.md
├── LICENSE
├── NOTICE
├── MAINTAINERS
├── .clang-format
├── .editorconfig
├── CONTRIBUTING.md
├── Documentation/
│   ├── README.md
│   ├── lsm-design.md                # 纯 C LSM 设计（引用 07-airy-lsm-design.md）
│   ├── capability-model.md          # Capability 模型
│   └── cupolas.md                   # Cupolas 安全穹顶
├── CMakeLists.txt
│
├── airy_lsm/                         # ★ 纯 C LSM 模块（DEFINE_LSM(airy)）
│   ├── Kbuild
│   ├── airy_lsm.c                   # LSM 注册（DEFINE_LSM(airy) + LSM_ORDER_MUTABLE）
│   ├── airy_lsm_hooks.c             # 250 钩子实现
│   ├── airy_lsm_blob.c              # blob 管理（airy_cred_security 等）
│   ├── airy_lsm_policy.c            # 策略加载
│   └── airy_lsm.h
│
├── capability/                       # Capability 系统（v1.1: agent_caps[1024] 静态数组）
│   ├── Kbuild
│   ├── airy_cap_init.c              # 初始化（agent_caps[1024] 静态数组 + 全局 Epoch）
│   ├── airy_cap_array.c             # agent_caps[1024] 静态数组管理（v1.1 替代 radix_tree 派生树）
│   ├── airy_cap_derive.c            # seL4 CNode 派生（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate）
│   ├── airy_cap_check.c             # slowpath 校验（fastpath C-S9 Badge 内联在 kernel/kernel/superv/airy_cap_check.c）
│   ├── airy_cap_revoke.c            # 撤销（atomic_inc(&airy_cap_global_epoch) O(1) 立即生效）
│   ├── airy_cap_rotate.c            # 轮换（Epoch 更新）
│   └── airy_cap.h
│
├── cupolas/                          # Cupolas 安全穹顶（用户态 API）
│   ├── CMakeLists.txt
│   ├── airy_cupolas_api.c           # airy_cupolas_guard_enter() 等用户态 API
│   ├── airy_cupolas_policy.c        # 策略管理
│   ├── airy_cupolas_verdict.c       # 裁决（ALLOW/DENY/AUDIT/COMPLAIN）
│   └── airy_cupolas.h
│
├── integrity/                        # 完整性
│   ├── CMakeLists.txt
│   ├── airy_integrity_measure.c    # 度量
│   ├── airy_integrity_verify.c     # 验证
│   └── airy_integrity.h
│
├── keys/                             # 密钥管理
│   ├── CMakeLists.txt
│   ├── airy_keyring.c               # 密钥环
│   ├── airy_key_vault.c             # Vault backend
│   └── airy_keys.h
│
├── include/                          # security 子仓内部头文件（非 [SC]）
│   └── security/
│       ├── airy_lsm_internal.h
│       ├── airy_cap_internal.h
│       └── airy_cupolas_internal.h
│
├── tools/                            # security 子仓工具
│   ├── cap-inspector/               # Capability 检查器
│   │   └── cap_inspector.c
│   └── lsm-policy-gen/               # 策略生成器
│       └── policy_gen.c
│
└── tests/                            # security 子仓测试
    ├── unit/
    └── integration/
```

### 4.4 子仓 4：memory（记忆子系统）

> **职责**：MemoryRovol L1-L4 四层记忆 + CXL 内存池化 + PMEM 持久化 + MGLRU 多代 LRU + VFS 持久化 + rovol-kmod。
> **IRON-9 主层**：[IND]（agentrt-linux 专属）
> **许可证**：SPHARX Commercial EULA v1.0（MemoryRovol 闭源商业，特例）

```
memory/
├── README.md                         # 含 MemoryRovol EULA 声明
├── LICENSE-EULA                      # SPHARX Commercial EULA v1.0
├── NOTICE
├── MAINTAINERS
├── .clang-format
├── .editorconfig
├── CONTRIBUTING.md
├── Documentation/
│   ├── README.md
│   ├── memoryrovol.md               # MemoryRovol L1-L4 设计
│   ├── cxl-tiering.md               # CXL 分层
│   └── pmem.md                      # PMEM 持久化
├── CMakeLists.txt
│
├── memoryrovol/                      # ★ MemoryRovol L1-L4（商业 EULA）
│   ├── Kbuild
│   ├── mr_l1.c                      # L1 hot（DRAM）
│   ├── mr_l2.c                      # L2 warm（MGLRU aging）
│   ├── mr_l3.c                      # L3 cold（CXL tier）
│   ├── mr_l4.c                      # L4 persistent（PMEM）
│   ├── mr_forgetting.c              # Forgetting Engine 遗忘机制
│   ├── mr_trace.c                    # TraceID 跟踪
│   ├── mr_audit.c                    # 审计哈希链
│   └── mr.h
│
├── cxl/                              # CXL 内存池化
│   ├── Kbuild
│   ├── cxl_pool.c                   # 池化管理
│   ├── cxl_tier.c                   # 分层（FAST/CXL/PMEM/SSD）
│   ├── cxl_migrate.c                # 迁移
│   └── cxl.h
│
├── pmem/                             # PMEM 持久化
│   ├── Kbuild
│   ├── pmem_ops.c                   # persist/flush/map/unmap
│   ├── pmem_map.c                   # mmap
│   └── pmem.h
│
├── mglru/                            # MGLRU 多代 LRU 增强
│   ├── Kbuild
│   ├── mglru_gen.c                  # 多代管理
│   ├── mglru_aging.c                # 老化策略
│   └── mglru.h
│
├── vfs-persist-kern/                 # VFS 持久化内核模块
│   ├── Kbuild
│   ├── vfs_persist_kern.c
│   └── vfs_persist.h
│
├── rovol-kmod/                       # MemoryRovol 内核模块入口
│   ├── Kbuild
│   ├── rovol_init.c                 # 模块初始化
│   └── rovol.h
│
├── include/                          # memory 子仓内部头文件
│   └── memory/
│       ├── mr_internal.h
│       ├── cxl_internal.h
│       └── pmem_internal.h
│
├── tools/
│   ├── mem-tier-analyzer/           # 记忆分层分析器
│   │   └── mem_tier_analyzer.c
│   └── mr-stat/                      # MemoryRovol 统计
│       └── mr_stat.c
│
└── tests/
    ├── unit/
    └── integration/
```

### 4.5 子仓 5：cognition（认知子系统）

> **职责**：CoreLoopThree 三阶段 + Thinkdual 双模式 + LLM 调度 + Wasm 沙箱 + GPU/NPU 调度 + kthread 实现。
> **IRON-9 主层**：[IND]（agentrt-linux 专属）
> **许可证**：AGPL-3.0-or-later OR Apache-2.0

```
cognition/
├── README.md
├── LICENSE
├── NOTICE
├── MAINTAINERS
├── .clang-format
├── .editorconfig
├── CONTRIBUTING.md
├── Documentation/
│   ├── README.md
│   ├── coreloopthree.md             # CoreLoopThree 设计
│   ├── thinkdual.md                 # Thinkdual 设计
│   └── gpu-npu-sched.md             # GPU/NPU 调度
├── CMakeLists.txt
│
├── coreloopthree/                    # ★ CoreLoopThree 三阶段
│   ├── Kbuild
│   ├── clt_main.c                   # kthread 主循环（PERCEPTION→THINKING→ACTION）
│   ├── clt_phase.c                  # 阶段管理
│   ├── clt_context.c                # 上下文管理（6 字段）
│   ├── clt_cycle.c                  # 周期计数
│   └── clt.h
│
├── thinkdual/                        # Thinkdual 双模式
│   ├── Kbuild
│   ├── td_mode.c                    # SYSTEM1_FAST / SYSTEM2_SLOW
│   ├── td_switch.c                  # 模式切换
│   └── td.h
│
├── llm/                              # LLM 推理调度
│   ├── CMakeLists.txt
│   ├── llm_sched.c                  # 调度（PREFILL/DECODE/SPECULATIVE）
│   ├── llm_token.c                  # Token 效率
│   ├── llm_kv.c                     # KV 缓存
│   └── llm.h
│
├── compute-accel/                    # 计算加速
│   ├── Kbuild
│   ├── accel_gpu.c                  # GPU（drm_sched）
│   ├── accel_npu.c                  # NPU
│   ├── accel_sched.c                # 调度
│   └── accel.h
│
├── wasm-sandbox/                     # Wasm 沙箱
│   ├── CMakeLists.txt
│   ├── wasm_runtime.c               # Wasm 运行时
│   ├── wasm_isolate.c               # 隔离
│   ├── wasm_api.c                   # API
│   └── wasm.h
│
├── kthread/                          # kthread 通信基础设施
│   ├── Kbuild
│   ├── kt_kfifo.c                   # kfifo + wait_event_interruptible
│   ├── kt_comm.c                    # 通信
│   └── kt.h
│
├── include/                          # cognition 子仓内部头文件
│   └── cognition/
│       ├── clt_internal.h
│       ├── td_internal.h
│       └── llm_internal.h
│
├── tools/
│   └── cogn-monitor/                # 认知监控
│       └── cogn_monitor.c
│
└── tests/
    ├── unit/
    └── integration/
```

### 4.6 子仓 6：cloudnative（云原生）

> **职责**：K8s CRD + containerd-shim + CNI + OCI + agentctl + DPU/IPU + super-node OS + OpenTelemetry。
> **IRON-9 主层**：[IND]（agentrt-linux 专属）
> **许可证**：AGPL-3.0-or-later OR Apache-2.0

```
cloudnative/
├── README.md
├── LICENSE
├── NOTICE
├── MAINTAINERS
├── .clang-format
├── .editorconfig
├── CONTRIBUTING.md
├── Documentation/
│   ├── README.md
│   ├── k8s-crd.md                   # K8s CRD 设计
│   ├── containerd-shim.md           # containerd shim
│   └── agentctl.md                  # agentctl
├── CMakeLists.txt
│
├── k8s-crd/                          # K8s CRD
│   ├── CMakeLists.txt
│   ├── api/                          # CRD API
│   │   ├── types.go                 # Agent CRD 类型
│   │   ├── groupversion_info.go
│   │   └── zz_generated.deepcopy.go
│   ├── controllers/                 # 控制器
│   │   ├── agent_controller.go
│   │   └── suite_test.go
│   ├── crd/                          # CRD YAML
│   │   └── agent.airymax.io.yaml
│   └── README.md
│
├── containerd-shim/                  # containerd shim
│   ├── CMakeLists.txt
│   ├── shim_main.c                   # shim 入口
│   ├── shim_runtime.c               # 运行时
│   ├── shim_io.go                    # I/O
│   └── shim.h
│
├── cni/                              # CNI 插件
│   ├── CMakeLists.txt
│   ├── cni_plugin.c                  # 插件入口
│   ├── cni_config.c                  # 配置
│   └── cni.h
│
├── oci/                              # OCI 规范实现
│   ├── CMakeLists.txt
│   ├── oci_runtime.c                # runtime spec
│   ├── oci_image.c                  # image spec
│   └── oci.h
│
├── agentctl/                         # ★ agentctl 命令行
│   ├── CMakeLists.txt
│   ├── agentctl.c                    # 命令行入口（真实 main()）
│   ├── cmd/                          # 子命令
│   │   ├── cmd_agent.c              # agent 子命令
│   │   ├── cmd_sched.c              # sched 子命令
│   │   ├── cmd_mem.c                # mem 子命令
│   │   └── cmd_cogn.c               # cogn 子命令
│   └── agentctl.h
│
├── observability/                    # OpenTelemetry
│   ├── CMakeLists.txt
│   ├── otel_collector.c             # 收集器
│   ├── otel_exporter.c             # 导出器
│   ├── otel_trace.c                 # TraceID
│   └── otel.h
│
├── dpu-ipu/                          # DPU/IPU 支持
│   ├── CMakeLists.txt
│   ├── dpu_offload.c                # 卸载
│   └── dpu.h
│
├── super-node-os/                    # super-node OS
│   ├── CMakeLists.txt
│   ├── sno_main.c                   # 入口
│   ├── sno_scheduler.c              # 调度
│   └── sno.h
│
├── sdk/                              # cloudnative SDK
│   ├── CMakeLists.txt
│   ├── sdk_c.c                      # C SDK
│   ├── sdk_go.go                    # Go SDK
│   └── sdk.h
│
└── tests/
    ├── unit/
    └── integration/
```

### 4.7 子仓 7：system（系统基础）

> **职责**：RPM/dnf 包管理 + 配置工具 + shell + 基础库 + 监控（airymaxmon）+ DevStation + packaging。
> **IRON-9 主层**：[IND]（agentrt-linux 专属）
> **许可证**：AGPL-3.0-or-later OR Apache-2.0

```
system/
├── README.md
├── LICENSE
├── NOTICE
├── MAINTAINERS
├── .clang-format
├── .editorconfig
├── CONTRIBUTING.md
├── Documentation/
│   ├── README.md
│   ├── packaging.md                 # 包管理
│   ├── devstation.md                # DevStation
│   └── airymaxmon.md                # airymaxmon
├── CMakeLists.txt
│
├── package-manager/                  # RPM + dnf
│   ├── CMakeLists.txt
│   ├── rpm/
│   │   ├── airy.spec                # RPM spec
│   │   └── build-rpm.sh
│   ├── dnf/
│   │   ├── dnf.conf
│   │   └── repo/
│   └── README.md
│
├── config/                           # 配置工具
│   ├── CMakeLists.txt
│   ├── airy-config.c                # 配置命令行
│   ├── sysctl/
│   │   ├── 99-airymax.conf          # sysctl 配置
│   │   └── README.md
│   └── config.h
│
├── shell/                            # shell 基础
│   ├── CMakeLists.txt
│   ├── airy-shell.c                 # Airymax shell
│   └── shell.h
│
├── base-libs/                        # 基础库
│   ├── CMakeLists.txt
│   ├── libairybase/
│   │   ├── airy_base.c              # 基础函数
│   │   └── airy_base.h
│   └── libairystring/
│       ├── airy_string.c
│       └── airy_string.h
│
├── monitoring/                       # 监控（含 airymaxmon）
│   ├── CMakeLists.txt
│   ├── airymaxmon.c                  # ★ airymaxmon 主循环（真实读取 [SC] 状态）
│   ├── mon_sched.c                  # sched_tac 统计
│   ├── mon_mem.c                    # MemoryRovol L1-L4
│   ├── mon_cap.c                    # Capability 41 ID
│   ├── mon_clt.c                    # CoreLoopThree 阶段
│   ├── mon_struct_ops.c            # struct_ops 状态机
│   └── airymaxmon.h
│
├── devstation/                       # DevStation（[SS]，agentrt commons 同源）
│   ├── CMakeLists.txt
│   ├── devstation.c                 # 入口
│   ├── ai_assistant.c               # AI 助手（io_uring IPC [SC]）
│   ├── project_gen.c                # 项目生成
│   └── devstation.h
│
├── packaging/                        # 打包
│   ├── CMakeLists.txt
│   ├── iso/
│   │   └── build-iso.sh             # ISO 构建
│   ├── tar/
│   │   └── build-tar.sh
│   └── README.md
│
├── commons/                          # system 公共代码
│   ├── CMakeLists.txt
│   ├── sys_common.c
│   └── sys_common.h
│
├── init/                             # 初始化
│   ├── CMakeLists.txt
│   ├── airy-init.c                  # Airymax init
│   └── init.h
│
└── tests/
    ├── unit/
    └── integration/
```

### 4.8 子仓 8：tests-linux（测试）

> **职责**：单元测试 + 集成测试 + 形式化验证（seL4 Isabelle/HOL + Coq）+ soak + chaos + benchmark + eBPF 验证。
> **IRON-9 主层**：[SS]（与 agentrt tests 语义同源）+ [IND]（agentrt-linux 专属测试）
> **许可证**：AGPL-3.0-or-later OR Apache-2.0

```
tests-linux/
├── README.md
├── LICENSE
├── NOTICE
├── MAINTAINERS
├── .clang-format
├── .editorconfig
├── CONTRIBUTING.md
├── Documentation/
│   ├── README.md
│   ├── formal-verification.md      # 形式化验证标准（seL4 三阶段）
│   └── coverage-targets.md          # 覆盖率目标
├── CMakeLists.txt
│
├── unit/                             # 单元测试（[SS]）
│   ├── CMakeLists.txt
│   ├── test_ipc_magic.c             # IPC magic 0x41524531 校验
│   ├── test_task_desc_magic.c       # 任务描述符 magic 0x41475453
│   ├── test_sched_policy.c          # stc_* 策略
│   ├── test_capability.c            # Capability 41 ID
│   ├── test_verdict.c               # AIRY_VERDICT 4 值
│   ├── test_lsm_hooks.c            # 纯 C LSM 钩子
│   ├── test_memory_types.c          # MemoryRovol L1-L4
│   ├── test_cognition_types.c      # CoreLoopThree + Thinkdual
│   ├── test_syscalls.c             # 12 系统调用
│   └── test_uapi_compat.c           # 三路桥接
│
├── integration/                      # 集成测试（[IND]）
│   ├── CMakeLists.txt
│   ├── test_daemon_startup.c       # 12 daemon 启动顺序
│   ├── test_ipc_end_to_end.c       # IPC 端到端
│   ├── test_superv_freeze.c        # Micro-Supervisor 冻结
│   ├── test_dual_supervisor.c      # 双 Supervisor 模型
│   └── test_systemd_integration.c  # systemd 集成
│
├── kunit/                             # KUnit 内核单元测试
│   ├── Kbuild
│   ├── kunit_sched.c
│   ├── kunit_ipc.c
│   ├── kunit_lsm.c
│   └── kunit_memory.c
│
├── selftests/                        # Linux selftests 扩展
│   ├── Makefile
│   ├── airy_ipc/
│   ├── airy_sched/
│   └── airy_lsm/
│
├── fault-injection/                  # 故障注入
│   ├── CMakeLists.txt
│   ├── fi_cap_fault.c               # Capability 故障
│   ├── fi_ipc_freeze.c              # IPC 冻结
│   ├── fi_vm_fault.c                # VM 故障
│   └── fi_die.c                     # die_notifier
│
├── formal-verification/              # ★ 形式化验证（seL4 三阶段）
│   ├── spec/                         # 阶段 1：形式规约
│   │   ├── ipc_spec.thy             # Isabelle/HOL IPC 规约
│   │   ├── sched_spec.thy           # 调度规约
│   │   ├── cap_spec.thy             # Capability 规约
│   │   └── README.md
│   ├── proof/                       # 阶段 2：形式证明
│   │   ├── ipc_proof.thy
│   │   ├── sched_proof.thy
│   │   ├── cap_proof.thy
│   │   └── README.md
│   └── capdl/                       # 阶段 3：CapDL 系统描述
│       ├── system.capdl
│       └── README.md
│
├── soak/                             # soak 测试（72h 连续）
│   ├── CMakeLists.txt
│   ├── soak_runner.c                # 运行器
│   ├── soak_mem_growth.c            # 内存增长 <5%
│   ├── soak_perf_degrade.c          # 性能降级 <10%
│   └── README.md
│
├── chaos/                            # chaos 测试
│   ├── CMakeLists.txt
│   ├── chaos_kill_daemon.c          # 杀 daemon
│   ├── chaos_net_partition.c        # 网络分区
│   ├── chaos_disk_full.c            # 磁盘满
│   └── README.md
│
├── benchmark/                        # 性能基准
│   ├── CMakeLists.txt
│   ├── bench_ipc_latency.c          # IPC 延迟（~160ns 目标）
│   ├── bench_cap_check.c            # fastpath C-S9 Badge 校验（~10ns）
│   ├── bench_freeze.c              # 冻结（~200ns）
│   ├── bench_eventfd.c             # eventfd（~50ns）
│   ├── bench_frozen_check.c        # frozen 检查（~1ns）
│   └── README.md
│
└── observability-verify/             # eBPF 验证
    ├── CMakeLists.txt
    ├── verify_bpf_probe.c
    └── README.md
```

---

## §5 [SC] 10 头文件清单（物理宿主与引用）

> **权威源**：[120-cross-project-code-sharing.md §2](../50-engineering-standards/120-cross-project-code-sharing.md)、[06-iron9-shared-model.md §2](06-iron9-shared-model.md)、[10-unify-design.md](10-unify-design.md)

### 5.1 物理宿主与引用规则

**唯一物理宿主**：`kernel/include/uapi/linux/airymax/`（10 个头文件全部在此目录）

**引用方式**：
- CMake 子仓：`include_directories(${CMAKE_SOURCE_DIR}/../kernel/include)`
- Kbuild 子仓：`ccflags-y += -I$(src)/../kernel/include`
- **禁止**：物理复制头文件到其他子仓（违反 OS-IRON-014 与 P3 单一宿主）

**双向字节级校验**：`.github/workflows/sc-dual-ci.yml`
```yaml
sc-dual-ci:
  steps:
    - name: verify-sc-byte-identity
      run: |
        diff -r kernel/include/uapi/linux/airymax/ agentrt/../kernel/include/uapi/linux/airymax/
        if [ $? -ne 0 ]; then
          echo "[SC] byte-level identity violated"
          exit 1
        fi
```

### 5.2 10 头文件清单

| # | 头文件 | 模块归属 | magic / 标识 | 物理路径 | 职责 |
|---|--------|---------|-------------|---------|------|
| 1 | `error.h` | A-UEF | — | `kernel/include/uapi/linux/airymax/error.h` | 统一错误码（5 子空间：[-1,-40] POSIX / [-41,-70] IPC / [-71,-100] Capability / [-101,-200] [SC] / [-201,-300] [DSL]）+ Fault 码（0x1000+）+ [DSL] 降级块（38 POSIX 码） |
| 2 | `log_types.h` | A-ULP | — | `kernel/include/uapi/linux/airymax/log_types.h` | 128B 固定日志记录 + 5 级日志枚举 + printk 8 级映射 |
| 3 | `ipc.h` | A-IPC | `0x41524531` ('ARE1') | `kernel/include/uapi/linux/airymax/ipc.h` | IPC 消息头（128B `struct airy_ipc_msg_hdr`，11 字段，D-9 修复后 `__attribute__((aligned(64)))`（移除 packed），`_Static_assert(sizeof==128)` + `_Static_assert(offsetof(capability_badge)==40)`）+ SQE/CQE 操作码与标志位 + Ring 常量（DEF=256, MAX=32768） |
| 4 | `sched.h` | A-ULS | `0x41475453` ('AGTS') | `kernel/include/uapi/linux/airymax/sched.h` | 任务描述符（magic/prio/_pad/vtime）+ `airy_vtime_decay()` inline + AIRY_CAP_MAX_AGENTS=1024 + AIRY_SLICE_DFL=20 + 优先级 0-139 + 权重 1-10000 |
| 5 | `memory_types.h` | MemoryRovol | — | `kernel/include/uapi/linux/airymax/memory_types.h` | L1-L4 enum（HOT/WARM/COLD/PMEM）+ GFP 掩码语义（AIRY_GFP_HOT/WARM/COLD/PMEM） |
| 6 | `security_types.h` | 安全 | — | `kernel/include/uapi/linux/airymax/security_types.h` | POSIX capability 41 ID（CAP_AGENT_SPAWN=41 起）+ LSM 钩子 250 ID + `enum airy_verdict`（ALLOW/DENY/AUDIT/COMPLAIN）+ `enum airy_cap_op`（7 操作：Copy/Mint/Move/Mutate/Revoke/Delete/Rotate）+ `cap_t` = `uint64_t` |
| 7 | `cognition_types.h` | A-UCS | — | `kernel/include/uapi/linux/airymax/cognition_types.h` | `airy_q16_t`（= `__s32`，Q16.16 定点数，因 -mno-80387 禁 FPU）+ `enum airy_cog_phase`（PERCEPT/THINK/ACT）+ `enum airy_think_mode`（FAST/SLOW） |
| 8 | `syscalls.h` | 系统调用 | — | `kernel/include/uapi/linux/airymax/syscalls.h` | v1.1: 4 核心系统调用（AIRY_SYS_CALL=0 ... AIRY_SYS_CLT_NOTIFY=3）+ 20 预留（4-23）= 24 槽位 |
| 9 | `uapi_compat.h` | 桥接 | — | `kernel/include/uapi/linux/airymax/uapi_compat.h` | 三路类型桥接（`#ifdef __KERNEL__` / `#elif defined(__linux__)` / `#else`），确保 [SC] 头文件跨平台逐字节相同编译 |
| 10 | `lsm_types.h` | 安全 | — | `kernel/include/uapi/linux/airymax/lsm_types.h` | `struct airy_lsm_blob` + `airy_capability_check()` 回调原型 + Capability 缓存结构 |

### 5.3 补充共享文件（非 [SC] 核心）

| 文件 | 路径 | 说明 |
|------|------|------|
| `bpf_struct_ops.h` | `kernel/include/uapi/linux/airymax/bpf_struct_ops.h` | BPF struct_ops 状态管理共享结构（`struct airy_struct_ops_value` + `enum airy_struct_ops_state`），两端共享但**不属于** 10 个 [SC] 核心头文件 |

### 5.4 [DSL] 降级生存层

每个 [SC] 头文件内嵌 `#ifdef AIRY_SC_FALLBACK` 降级块，[SC] 损坏时提供最小可运行子集：

| 头文件 | [DSL] 降级内容 |
|--------|--------------|
| `error.h` | 38 个 POSIX 标准错误码（`AIRY_EINVAL` 等） |
| `ipc.h` | 最简 IPC（3 字段消息头 + 2 操作码） |
| `sched.h` | EEVDF 默认调度（无 sched_tac 策略） |
| `log_types.h` | printk 原生（仅 LOG_FATAL + LOG_ERROR） |
| 其他 | 各自最小子集 |

详细设计见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md)。

---

## §6 IRON-9 v3 四层模型目录落地

> **权威源**：[06-iron9-shared-model.md](06-iron9-shared-model.md)、[120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md)

### 6.1 四层定义与目录映射

| 层次 | 缩写 | 共享程度 | 物理目录落地 | 内容 |
|------|------|---------|------------|------|
| **共享契约层** | [SC] | 完全共享代码 | `kernel/include/uapi/linux/airymax/`（10 头文件） | 10 个头文件，逐字节相同，两端共同依赖 |
| **语义同源层** | [SS] | 高层 API 语义同源，签名独立演进 | 各子仓独立实现（如 services/daemons/gateway_d/） | 调度语义、安全模型、IPC 传输、记忆模型 |
| **完全独立层** | [IND] | 完全独立 | 各子仓独立仓库 | 内核驱动/Kbuild（agentrt-linux）；跨平台用户态/SDK 4 语言（agentrt） |
| **降级生存层** | [DSL] | 自包含降级块 | [SC] 头文件内 `#ifdef AIRY_SC_FALLBACK` | [SC] 损坏时的最小可运行子集 |

### 6.2 [SC] 共享契约层落地

**物理宿主**：`kernel/include/uapi/linux/airymax/`（10 头文件，详见 §5）

**两端引用方式**：
- agentrt-linux 子仓：`-I../kernel/include`（决策 G1）
- agentrt 子仓：`-I../agentrt-linux/kernel/include`（通过 git submodule 引用）

**禁止**：
- 物理复制头文件
- 创建 `libairymax/` 共享库子仓
- 在子仓间 `#include` 对方非 [SC] 头文件

### 6.3 [SS] 语义同源层落地

各子仓独立实现同名 API，签名可独立演进：

| [SS] API | agentrt-linux 物理路径 | agentrt 物理路径 | 语义同源点 |
|----------|---------------------|----------------|----------|
| `airy_ipc_send()` | `services/userland/ipc/airy_ipc_client.c` | `atoms/ipc/airy_ipc.c` | [SC] ipc.h（magic 0x41524531） |
| `airy_sched_enqueue()` | `kernel/corekern/sched/stc_dispatch.c` | `atoms/sched/airy_sched.c` | [SC] sched.h（magic 0x41475453） |
| `airy_cap_check()` | `security/capability/airy_cap_check.c` | `cupolas/airy_cupolas.c` | [SC] security_types.h（capability 41 ID） |
| `airy_log_emit()` | `kernel/log/airy_log_kern.c` | `commons/log/airy_log.c` | [SC] log_types.h（128B 记录） |
| `airy_mr_alloc()` | `memory/memoryrovol/mr_l1.c` | `heapstore/airy_mr.c` | [SC] memory_types.h（L1-L4） |
| `airy_cog_phase_set()` | `cognition/coreloopthree/clt_phase.c` | `atoms/cognition/clt.c` | [SC] cognition_types.h（airy_q16_t） |

**12 daemon 中的 [SS]（3 个，决策 D1）**：
- `gateway_d`（agentrt gateway 同源 OS 升级）
- `sched_d`（agentrt atoms/sched 同源 OS 升级）
- `cogn_d`（agentrt atoms/cognition 同源 OS 升级）

### 6.4 [IND] 完全独立层落地

**agentrt-linux [IND]（与 agentrt 完全不共享）**：
- `kernel/`（除 include/uapi/linux/airymax/ 外）：内核驱动、Kbuild、内核 API、systemd、纯 C LSM、eBPF struct_ops、IORING_OP_URING_CMD、alloc_pages+mmap
- `security/airy_lsm/`：DEFINE_LSM(airy) 纯 C LSM
- `security/capability/`：v1.1 `agent_caps[1024]` 静态数组 + Badge 64-bit Native Word（替代 v1.0 radix_tree 派生模型）
- `memory/memoryrovol/`：L1-L4 内核模块
- `cognition/coreloopthree/`：kthread 实现
- `cognition/kthread/`：kfifo + wait_event_interruptible
- `cloudnative/`：K8s/containerd/CNI
- `system/`：RPM/dnf/DevStation

**agentrt [IND]（与 agentrt-linux 完全不共享）**：
- `atoms/`：跨平台用户态原语
- `commons/`：公共库
- `daemons/`：用户态 daemon 共享
- `gateway/`：用户态网关
- `protocols/`：协议
- `heapstore/`：堆存储

### 6.5 [DSL] 降级生存层落地

每个 [SC] 头文件内嵌 `#ifdef AIRY_SC_FALLBACK` 降级块：

```c
/* kernel/include/uapi/linux/airymax/error.h */
#ifndef _AIRY_ERROR_H
#define _AIRY_ERROR_H

#include <airymax/uapi_compat.h>

/* 完整错误码体系（5 子空间 + Fault 码）*/
typedef int32_t airy_err_t;
#define AIRY_EOK 0
#define AIRY_EINVAL (-22)
/* ... 完整定义 ... */

#ifdef AIRY_SC_FALLBACK
/* [DSL] 降级块：仅保留 38 个 POSIX 标准错误码 */
#undef AIRY_EOK
#define AIRY_EOK 0
#define AIRY_EINVAL (-22)
/* ... 38 个 POSIX 码 ... */
#endif /* AIRY_SC_FALLBACK */

#endif /* _AIRY_ERROR_H */
```

详细 [DSL] 设计见 [11-degraded-survival-layer.md](11-degraded-survival-layer.md)。

---

## §7 Daemon 命名统一规范

> **权威源**：[90-terminology.md §二 附录 A](../50-engineering-standards/90-terminology.md)

### 7.1 12 daemon 最终命名（决策 C1 统一 `_d` 后缀）

| # | 二进制名 | systemd unit | 职责 | 启动顺序 | IRON-9 层 | 旧名（废弃） |
|---|---------|-------------|------|---------|----------|-------------|
| 1 | `macro_d` | `agentrt-macro.service` | Macro-Supervisor 用户态裁决 | 1 | [IND] | macro_superv |
| 2 | `gateway_d` | `agentrt-gateway.service` | 网关服务 | 1 | [SS] | — |
| 3 | `sched_d` | `agentrt-sched.service` | 调度服务 | 1 | [SS] | — |
| 4 | `logger_d` | `agentrt-logger.service` | 日志服务 | 2 | [IND] | logger_daemon |
| 5 | `vfs_d` | `agentrt-vfs.service` | VFS 服务 | 2 | [IND] | — |
| 6 | `net_d` | `agentrt-net.service` | 网络服务 | 2 | [IND] | — |
| 7 | `cogn_d` | `agentrt-cogn.service` | 认知服务 | 2 | [SS] | llm_d（废弃） |
| 8 | `dev_d` | `agentrt-dev.service` | 设备/工具服务 | 3 | [IND] | tool_d/drv_d（废弃） |
| 9 | `config_d` | `agentrt-config.service` | 配置服务 | 3 | [IND] | config_daemon |
| 10 | `mem_d` | `agentrt-mem.service` | 记忆服务 | 4 | [IND] | — |
| 11 | `audit_d` | `agentrt-audit.service` | 审计服务 | 4 | [IND] | — |
| 12 | `sec_d` | `agentrt-sec.service` | 安全服务 | 5 | [IND] | — |

### 7.2 命名规则

| 规则 | 说明 |
|------|------|
| 二进制名 | `*_d` 后缀（统一，决策 C1） |
| systemd unit | `agentrt-<name>.service`（前缀 agentrt，与 agentrt 一致） |
| 源码目录 | `services/daemons/<name>_d/` |
| 主源文件 | `<name>_d.c`（如 `macro_d.c`） |
| 头文件 | `<name>_d.h` |
| 启动顺序 | 1（核心三服务）→ 2（基础四服务）→ 3（工具两服务）→ 4（记忆审计两服务）→ 5（安全） |

### 7.3 [SS] 与 agentrt 的对应（决策 D1）

| agentrt-linux daemon | agentrt 对应 | 关系 | [SS] 同源点 |
|---------------------|------------|------|-----------|
| `gateway_d` | `gateway/` | OS 级升级 | JSON-RPC 2.0 协议 + [SC] ipc.h |
| `sched_d` | `atoms/sched/` | OS 级升级 | [SC] sched.h（magic 0x41475453） |
| `cogn_d` | `atoms/cognition/` | OS 级升级 | [SC] cognition_types.h（CoreLoopThree） |

其余 9 daemon（`macro_d` / `logger_d` / `config_d` / `vfs_d` / `net_d` / `mem_d` / `sec_d` / `audit_d` / `dev_d`）为 [IND] 完全独立，agentrt 无对应。

### 7.4 与 Micro-Supervisor 的协作

`macro_d`（Macro-Supervisor 用户态）与 `kernel/kernel/superv/`（Micro-Supervisor 内核态）构成双 Supervisor 模型：

| Supervisor | 执行域 | 物理路径 | 职责 |
|-----------|--------|---------|------|
| Micro-Supervisor | 内核态 | `kernel/kernel/superv/`（5 个 .c） | 冷酷执法：检测 → 冻结 → 返回 AIRY_FAULT_ABNORMAL_CAP → eventfd 通知 |
| Macro-Supervisor | 用户态 | `services/daemons/macro_d/` | 温情裁决：接收 eventfd → 查询上下文 → 裁决（警告/降级/暂停/终止） |

详细设计见 [09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)。

---

## §8 跨子仓依赖图

### 8.1 依赖规则

| 规则 | 说明 |
|------|------|
| R1 | 所有子仓通过 `-I../kernel/include` 引用 [SC] 10 头文件（决策 G1） |
| R2 | 子仓间禁止 `#include` 对方非 [SC] 头文件（决策 I1） |
| R3 | 子仓间禁止链接对方 .o/.a（决策 I1） |
| R4 | 跨子仓通信仅通过 IPC（io_uring）或系统调用 |
| R5 | kernel 是 [SC] 头文件唯一物理宿主，其他子仓仅引用 |

### 8.2 Mermaid 依赖图

```mermaid
graph TD
    subgraph "管理仓 agentrt-linux"
        K[kernel<br/>ALK 6.6 + [SC] 宿主]
        S[services<br/>12 daemon + userland]
        SE[security<br/>纯 C LSM + Capability]
        M[memory<br/>MemoryRovol L1-L4]
        C[cognition<br/>CoreLoopThree + Thinkdual]
        CN[cloudnative<br/>K8s + containerd]
        SY[system<br/>RPM + DevStation + airymaxmon]
        T[tests-linux<br/>单元+集成+形式化]
    end

    subgraph "[SC] 共享层"
        SC[kernel/include/uapi/linux/airymax/<br/>10 头文件]
    end

    subgraph "agentrt（外部）"
        A[agentrt<br/>用户态微核心]
    end

    %% [SC] 引用（-I，禁止物理副本）
    K --> SC
    S -.->|引用| SC
    SE -.->|引用| SC
    M -.->|引用| SC
    C -.->|引用| SC
    CN -.->|引用| SC
    SY -.->|引用| SC
    T -.->|引用| SC
    A -.->|字节级相同| SC

    %% [SS] 语义同源（签名独立演进）
    S -.->|[SS]| A
    C -.->|[SS]| A

    %% 跨子仓协作（通过 IPC/系统调用）
    S -->|IPC| K
    SE -->|系统调用| K
    M -->|系统调用| K
    C -->|kthread| K
    CN -->|IPC| S
    SY -->|读取状态| K

    %% 测试依赖
    T -->|测试| K
    T -->|测试| S
    T -->|测试| SE
    T -->|测试| M
    T -->|测试| C
    T -->|测试| CN
    T -->|测试| SY
```

### 8.3 依赖关系详表

| 源子仓 | 目标子仓 | 依赖类型 | 说明 |
|--------|---------|---------|------|
| kernel | [SC] 10 头文件 | 物理宿主 | 唯一宿主 |
| services | [SC] 10 头文件 | -I 引用 | 决策 G1 |
| services | kernel | IPC + 系统调用 | daemon 通过 io_uring 与内核通信 |
| security | [SC] 10 头文件 | -I 引用 | 决策 G1 |
| security | kernel | 系统调用 + LSM 注册 | airy_lsm 注册到内核 LSM 框架 |
| memory | [SC] 10 头文件 | -I 引用 | 决策 G1 |
| memory | kernel | 内核模块 + 系统调用 | MemoryRovol 作为内核模块加载 |
| cognition | [SC] 10 头文件 | -I 引用 | 决策 G1 |
| cognition | kernel | kthread + 系统调用 | CoreLoopThree kthread 运行于内核 |
| cloudnative | [SC] 10 头文件 | -I 引用 | 决策 G1 |
| cloudnative | services | IPC | agentctl 通过 IPC 与 daemon 通信 |
| system | [SC] 10 头文件 | -I 引用 | 决策 G1 |
| system | kernel | 读取状态 | airymaxmon 读取内核状态 |
| tests-linux | 全部子仓 | 测试 | 测试覆盖所有子仓 |

---

## §9 ALK 6.6 内核子仓内部结构

> **权威源**：[01-system-architecture.md](01-system-architecture.md)、ADR-013（Linux 6.6 LTS 基线）

### 9.1 Model A 完整 fork 策略

**Model A（完整 fork + 直接写入上游源码树）**：
- 不使用 patch 隔离
- 不使用 git subtree
- 直接在 Linux 6.6 LTS 源码树上修改
- Airymax 增强代码集中在 `kernel/corekern/`、`kernel/kernel/superv/`、`kernel/log/`、`kernel/ipc/` 目录
- 修改 Linux 原生代码时在 MAINTAINERS 登记

**禁用项**：
- ❌ sched_ext（6.6 主线不含，使用 sched_tac 替代）
- ❌ SCHED_AGENT 宏定义（避免与内核调度类编号冲突）
- ❌ BPF LSM（使用纯 C LSM）
- ❌ DMA 一致性内存（使用 alloc_pages + mmap）
- ❌ page flipping（使用 IORING_OP_URING_CMD + registered buffer + mmap）
- ❌ 浮点运算（-mno-80387，使用 Q16.16 定点数）

### 9.2 kernel/ 目录详解（Model A）

> **D-8 OLK 6.6 UAPI 路径对齐说明**：OLK 6.6 内核 UAPI 头文件标准路径为 `include/uapi/linux/`（参考 `include/uapi/linux/io_uring.h`、`include/uapi/linux/sched.h`）。Airymax 10 个 [SC] 共享契约头文件属用户态可见的 UAPI（agentrt 用户态与 agentrt-linux 内核双端共享），故物理宿主为 `include/uapi/linux/airymax/`。Airymax 内核内部头文件（`maintainer_types.h`/`build_types.h`/`kconfig_types.h` 等，[IND] 独立层）保留在 `include/airymax/`（非 UAPI）。

| 目录 | 来源 | 说明 |
|------|------|------|
| `arch/` | Linux 6.6 原生 | x86/arm64/riscv 等架构相关 |
| `include/uapi/` | Linux 6.6 原生 | 用户空间 API（含 SCHED_* 定义） |
| `include/uapi/linux/airymax/` | ★ Airymax 新增（D-8 对齐 OLK 6.6 UAPI 标准路径） | [SC] 10 头文件唯一物理宿主（用户态可见，与 agentrt 共享） |
| `include/asm-generic/` | Linux 6.6 原生 | 通用汇编头 |
| `include/kernel/` | Linux 6.6 原生 | 内核内部头 |
| `include/airymax/` | ★ Airymax 新增 | 内核内部头（[IND] 独立层，非 UAPI；如 `maintainer_types.h`/`build_types.h`/`kconfig_types.h`） |
| `corekern/` | ★ Airymax 新增 | 内核增强（api/sched/ipc/taskflow/memory/time/object/locking/irq/bpf） |
| `kernel/kernel/superv/` | ★ Airymax 新增 | Micro-Supervisor（5 个 .c） |
| `log/` | ★ Airymax 新增 | A-ULP 内核侧 |
| `ipc/` | ★ Airymax 新增 | IPC 内核基础设施 |
| `config/` | ★ Airymax 新增 | ALK 配置项 |
| `configs/` | ★ Airymax 新增 | 预置配置 |
| `init/` | Linux 6.6 原生 | 内核入口 |
| `lib/` | Linux 6.6 原生 | 内核 lib |
| `crypto/` | Linux 6.6 原生 | 加密 |
| `io_uring/` | Linux 6.6 原生 | io_uring（含 IORING_OP_URING_CMD） |
| `fs/` | Linux 6.6 原生 | 文件系统 |
| `net/` | Linux 6.6 原生 | 网络栈 |
| `block/` | Linux 6.6 原生 | 块设备 |
| `drivers/` | Linux 6.6 原生 | 驱动（含 accel/gpu/cxl/pmem） |
| `mm/` | Linux 6.6 原生 | 内存管理（含 MGLRU） |
| `security/airy/` | ★ Airymax 新增 | 纯 C LSM 内核侧（与 security 子仓同源） |
| `sound/` | Linux 6.6 原生 | 声卡 |
| `certs/` | Linux 6.6 原生 | 证书 |
| `samples/` | Linux 6.6 原生 | 示例 |
| `usr/` | Linux 6.6 原生 | 用户空间辅助 |
| `virt/` | Linux 6.6 原生 | 虚拟化 |
| `LICENSES/` | Linux 6.6 原生 | 许可证声明 |
| `rust/` | Linux 6.6 原生 | Rust 支持（保留，Airymax 不使用） |
| `Documentation/` | Linux 6.6 原生 + Airymax 增强 | 文档 |
| `scripts/` | Linux 6.6 原生 | 构建脚本 |
| `tools/` | Linux 6.6 原生 | 工具 |

### 9.2.1 include/airymax/ 目录定位

`kernel/include/airymax/` 是 ALK-6.6 的非 [SC] 内部类型头文件目录，与 `kernel/include/uapi/linux/airymax/`（[SC] 共享契约层，UAPI 标准路径）不同：

| 路径 | 定位 | 内容 | 共享范围 |
|------|------|------|---------|
| `include/uapi/linux/airymax/` | [SC] 共享契约层 | 10 个 UAPI 头文件 | agentrt + agentrt-linux 双端共享 |
| `include/airymax/` | 内核内部类型 | build_types.h / kconfig_types.h | 仅 agentrt-linux 内核使用 |

`include/airymax/` 中的文件不属于 [SC] 共享契约层，不参与 IRON-9 v3 的 CI 逐字节校验。

### 9.3 sched_tac 内核侧实现

**路径**：`kernel/corekern/sched/`

**文件**：
- `stc_policy.c`：4 策略枚举实现（stc_realtime/stc_interactive/stc_agent/stc_batch）
- `stc_dispatch.c`：派发到 SCHED_DEADLINE/SCHED_FIFO/EEVDF 原生调度类
- `stc_mcs_map.c`：seL4 MCS 语义映射
- `stc_stats.c`：调度统计

**约束**：
- 禁止 `SCHED_AGENT` 宏定义（约束 7）
- 禁止 `sched_ext`（约束 7）
- 禁止 `AIRY_SCHED_AGENT`（90-terminology.md 已废弃）

### 9.4 Micro-Supervisor 内核模块

**路径**：`kernel/kernel/superv/`（5 个 .c）

**文件**：
1. `airy_superv_lsm.c`：`DEFINE_LSM(airy)` 注册 + `LSM_ORDER_MUTABLE`
2. `airy_cap_check.c`：v1.1 slowpath LSM 钩子（C-S9 失败接管，详见 [07-airy-lsm-design.md §3.3](../110-security/07-airy-lsm-design.md)）
3. `airy_ipc_freeze.c`：`ring->frozen = true`（`smp_store_release`）
4. `airy_die_notify.c`：`die_notifier`（priority `INT_MAX`）
5. `airy_eventfd.c`：`eventfd_signal` 非阻塞通知

**冷酷执法 4 步流程**：
1. 检测（`airy_uring_cmd_check`）
2. 冻结（`airy_ipc_freeze_ring`，`smp_store_release(&ring->frozen, true)`）
3. 返回 Fault 码（`AIRY_FAULT_ABNORMAL_CAP` = 0x1005）
4. eventfd 通知（`airy_eventfd_signal_fault`，~50ns）

**fastpath 零开销**：`unlikely(READ_ONCE(ring->frozen))` ~1ns 正常路径

详细设计见 [09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)。

### 9.5 纯 C LSM 内核侧

**路径**：`kernel/security/airy/`（与 `security/airy_lsm/` 同源）

**关键代码**：
```c
/* kernel/security/airy/airy_lsm.c */
DEFINE_LSM(airy) = {
    .name = "airy",
    .order = LSM_ORDER_MUTABLE,   /* v1.1: 不使用 LSM_ORDER_FIRST（OLK 6.6 仅用于 capabilities），通过 CONFIG_LSM 列表置于 capability 之后 */
};

static struct security_hook_list airy_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(uring_cmd, airy_uring_cmd_check),
    /* ... 250 钩子 ... */
};
```

详细设计见 [07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md)。

---

## §10 与 agentrt 对比

### 10.1 顶层结构对比

| 维度 | agentrt（用户态微核心） | agentrt-linux（AirymaxOS） |
|------|----------------------|--------------------------|
| 定位 | AI Agent 运行时平台工程（跨平台） | 极境智能体操作系统（Linux 发行版） |
| 子仓数 | 7（atoms/commons/cupolas/daemons/gateway/protocols/heapstore） | 8（kernel/services/security/memory/cognition/cloudnative/system/tests-linux） |
| 仓库类型 | 管理仓（伞仓直属） | 管理仓（伞仓直属） |
| 内核依赖 | 无（用户态） | Linux 6.6 LTS fork（ALK 6.6） |
| 跨平台 | Linux/macOS/Windows | Linux only |
| IRON-9 [SC] | 引用方（-I 引用） | 物理宿主（kernel/include/uapi/linux/airymax/） |
| IRON-9 [IND] | 跨平台用户态 + SDK 4 语言 | 内核驱动/Kbuild/内核 API |

### 10.2 [SC] 头文件宿主关系

```
agentrt-linux/kernel/include/uapi/linux/airymax/    ← 唯一物理宿主（10 头文件）
            ↑
            | -I 引用（禁止物理副本）
            |
   +--------+--------+
   |                 |
agentrt/          agentrt-linux/
(引用方)          (各子仓引用)
```

**双向 CI 校验**（`.github/workflows/sc-dual-ci.yml`）：
- 字节级 diff：`diff -r kernel/include/uapi/linux/airymax/ agentrt/../kernel/include/uapi/linux/airymax/`
- 行为契约测试：两端编译相同测试用例，验证行为一致

### 10.3 daemon 对比

| 维度 | agentrt | agentrt-linux |
|------|---------|--------------|
| daemon 数 | 7 子仓含 daemon 共享代码 | 12 daemon（独立进程） |
| 运行方式 | 库形式（链接到应用） | systemd 服务（独立进程） |
| 隔离方式 | 进程内隔离 | systemd unit 隔离（独立地址空间 + capability 限制） |
| [SS] 重叠 | gateway/sched/cognition 3 项 | gateway_d/sched_d/cogn_d 3 项（OS 级升级） |
| 重叠率 | 16.7%（1/6 ≈ gateway） | 25%（3/12） |

### 10.4 许可证对比

| 维度 | agentrt | agentrt-linux |
|------|---------|--------------|
| 开源核心 | AGPL-3.0-or-later OR Apache-2.0 | AGPL-3.0-or-later OR Apache-2.0 |
| 内核 | — | GPL-2.0-only（kernel 子仓） |
| MemoryRovol | — | SPHARX Commercial EULA v1.0（memory 子仓，特例） |
| 文档 | CC-BY-NC-4.0 | CC-BY-NC-4.0 |
| 开放标准 | CC-BY-4.0 | CC-BY-4.0 |

---

## §11 工程美学自评

### 11.1 5 条简约原则达成度

| 原则 | 达成度 | 评估依据 | 改进方向 |
|------|--------|---------|---------|
| **P1 本质优先** | ★★★★★ | 8 子仓均对应不可合并的职责域，每子仓一句话可说明 | 无 |
| **P2 原子不可分** | ★★★★★ | services 内部分层不拆分（保持 8 子仓），保持职责清晰 | 无 |
| **P3 单一宿主** | ★★★★★ | [SC] 10 头文件唯一物理宿主 kernel/include/uapi/linux/airymax/，禁止物理副本 | 无 |
| **P4 内外一致** | ★★★★★ | 子仓公共骨架统一（README/LICENSE/MAINTAINERS/...），daemon 命名统一 `_d`（v2.0 决策已落地，90-terminology.md 已将 macro_superv/logger_daemon/config_daemon 标注为 旧称/禁止使用） | 无 |
| **P5 无冗余** | ★★★★★ | 跨子仓共享代码仅 [SC] 层，[IND] 完全独立，禁止共享库 | 无 |

### 11.2 与 v1.1 的美学提升

| 维度 | v1.1 | v2.0 | 提升 |
|------|------|------|------|
| 子仓数 | 8 | 8（不变，保持原子性） | 无变化（P2 达成） |
| daemon 命名一致性 | 混乱（_superv/_daemon/_d） | 统一 `_d`（12 个） | P4 提升 |
| [SC] 头文件宿主 | 未明确 | kernel/include/uapi/linux/airymax/ 唯一宿主 | P3 提升 |
| 跨子仓共享代码 | 不清 | 三层模型（[SC]/[SS]/[IND]） | P5 提升 |
| 决策闭环 | 5/9 | 9/9（全闭环） | 完整性提升 |
| 工程美学自评 | 无 | 5 条原则量化评估 | 可度量提升 |

### 11.3 反模式自检

| 反模式 | 是否存在 | 说明 |
|--------|---------|------|
| 合并职责不同的子仓 | ❌ 无 | 保持 8 子仓，cloudnative 与 system 分离 |
| 删除公共骨架文件 | ❌ 无 | 每子仓含 README/LICENSE/MAINTAINERS/CONTRIBUTING.md |
| [SC] 头文件物理复制 | ❌ 无 | 决策 G1 明确 -I 引用 |
| 跨子仓共享库 | ❌ 无 | 决策 I1 明确禁止 |
| 桩函数/桩文件 | ❌ 无 | 决策 F1 明确最小可编译骨架（非桩） |
| sched_ext / SCHED_AGENT | ❌ 无 | 使用 sched_tac（约束 7） |
| BPF LSM | ❌ 无 | 使用纯 C LSM（约束 8） |
| DMA 一致性内存 | ❌ 无 | 使用 alloc_pages + mmap（约束 11） |
| page flipping | ❌ 无 | 使用 IORING_OP_URING_CMD + registered buffer + mmap（约束 10） |

---

## §12 SSoT 登记与变更日志

### 12.1 SSoT 登记

| 登记项 | 登记值 | 权威源 | 说明 |
|--------|--------|--------|------|
| 文档编号 | DIR-007 | 本文件 | agentrt-linux 整体目录结构 |
| SSoT 类别 | 架构设计 | [09-ssot-registry.md §3](../50-engineering-standards/09-ssot-registry.md) | 架构设计类 |
| 权威源 | 本文件 | — | agentrt-linux 目录结构唯一权威 |
| 引用文档 | [06-iron9-shared-model.md](06-iron9-shared-model.md) | IRON-9 v3 四层模型 | 权威源 |
| 引用文档 | [10-unify-design.md](10-unify-design.md) | Unify Design 5 模块 | 权威源 |
| 引用文档 | [120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) | [SC] 10 头文件物理宿主 | 权威源 |
| 引用文档 | [90-terminology.md](../50-engineering-standards/90-terminology.md) | 12 daemon 命名 | 权威源 |
| 引用文档 | [09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) | Micro-Supervisor 内核模块 | 权威源 |
| 引用文档 | [07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) | 纯 C LSM 设计 | 权威源 |
| 引用文档 | [11-degraded-survival-layer.md](11-degraded-survival-layer.md) | [DSL] 降级生存层 | 权威源 |

### 12.2 硬约束符合性矩阵

| 约束 # | 约束内容 | 本文档落地章节 | 符合性 |
|--------|---------|--------------|--------|
| 1 | IRON-9 v3 四层模型 | §6 | ✅ |
| 2 | 10 [SC] 核心头文件 | §5 | ✅ |
| 3 | ALK 6.6（Model A 完整 fork） | §9 | ✅ |
| 4 | 8 子仓结构 | §2、§4 | ✅ |
| 5 | [SC] 头文件单一物理宿主 kernel/include/uapi/linux/airymax/ | §5 | ✅ |
| 6 | IPC magic 0x41524531；任务描述符 magic 0x41475453 | §5 | ✅ |
| 7 | sched_tac（不使用 sched_ext） | §9.3 | ✅ |
| 8 | 纯 C LSM（DEFINE_LSM(airy)） | §9.5 | ✅ |
| 9 | AIRY_VERDICT 4 值枚举 | §5 | ✅ |
| 10 | io_uring 零拷贝 IPC | §4.1、§4.2 | ✅ |
| 11 | alloc_pages + mmap | §4.4 | ✅ |
| 12 | 双 Supervisor 模型 | §7.4 | ✅ |
| 13 | 12 daemon | §7.1 | ✅ |
| 14 | 0.1.1 唯一奠基版本 | §0、§3 | ✅ |
| 15 | IRON-1~9 工程规则 | §11 | ✅ |

### 12.3 变更日志

| 日期 | 版本 | 变更内容 | 变更人 |
|------|------|---------|--------|
| 2026-07-10 | v1.0 | 初版目录结构 | 架构委员会 |
| 2026-07-14 | v1.1 | 增加 [SC] 头文件宿主说明、daemon 命名表 | 架构委员会 |
| 2026-07-18 | v2.0 | 基于乔布斯/艾夫"简约就是美"哲学重构；9 决策闭环；§0-§12 完整结构；[SC] 10 头文件清单；IRON-9 v3 落地；12 daemon 命名统一 `_d`；ALK 6.6 内核内部；agentrt 对比；工程美学自评；SSoT 登记 | 架构委员会 |

### 12.4 后续演进路线

| 版本 | 计划内容 | 预计时间 |
|------|---------|---------|
| 0.1.1 | 全部 10 [SC] 头文件创建 + 8 子仓最小可编译骨架 + 12 daemon 实现 | 2026-08 |
| 1.0.1 | 生产就绪 + 29 仓拆分 + 完整生态 | 2026-Q4 |

**禁止**：0.1.2 / 0.2.0 / 0.3.0 / 1.0.0 任何中间过渡版本（约束 14）。

---

## 附录 A：子仓公共骨架清单

每子仓必须包含以下 9 个公共骨架文件（决策 F1）：

| # | 文件 | 用途 | 内容要求 |
|---|------|------|---------|
| 1 | `README.md` | 子仓说明 | 职责、边界、依赖、构建说明 |
| 2 | `LICENSE` | 许可证 | 子仓特定（GPL-2.0 / AGPL-3.0 / Apache-2.0 / EULA） |
| 3 | `NOTICE` | 商标与版权 | 商标声明 + 版权声明 |
| 4 | `MAINTAINERS` | 维护者 | 维护者名单 + 职责范围 |
| 5 | `.clang-format` | 代码格式 | 全仓统一格式 |
| 6 | `.editorconfig` | 编辑器配置 | 全仓统一编辑器配置 |
| 7 | `CONTRIBUTING.md` | 贡献指南 | 提交流程、代码规范、测试要求 |
| 8 | `Documentation/README.md` | 子仓文档索引 | 文档目录与索引 |
| 9 | `CMakeLists.txt` 或 `Kbuild` | 构建入口 | 可编译（非桩） |

## 附录 B：12 daemon 完整源文件清单

| daemon | 源文件数 | 主要文件 |
|--------|---------|---------|
| macro_d | 9 | macro_d.c / macro_adjudicate.c / macro_context.c / macro_policy.c / macro_event.c / macro_state.c / macro_rpc.c / macro_d.h / macro_d_test.c |
| logger_d | 6 | logger_d.c / logger_collect.c / logger_persist.c / logger_rotate.c / logger_query.c / logger_d.h |
| config_d | 5 | config_d.c / config_loader.c / config_watcher.c / config_validate.c / config_d.h |
| gateway_d | 6 | gateway_d.c / gateway_jsonrpc.c / gateway_router.c / gateway_auth.c / gateway_tls.c / gateway_d.h |
| sched_d | 6 | sched_d.c / sched_stc.c / sched_policy.c / sched_stats.c / sched_cgroup.c / sched_d.h |
| vfs_d | 6 | vfs_d.c / vfs_ops.c / vfs_cache.c / vfs_mount.c / vfs_persist.c / vfs_d.h |
| net_d | 6 | net_d.c / net_stack.c / net_socket.c / net_route.c / net_filter.c / net_d.h |
| mem_d | 6 | mem_d.c / mem_tier.c / mem_reclaim.c / mem_cxl.c / mem_pmem.c / mem_d.h |
| cogn_d | 6 | cogn_d.c / cogn_clt.c / cogn_think.c / cogn_llm.c / cogn_gpu.c / cogn_d.h |
| sec_d | 6 | sec_d.c / sec_cap.c / sec_policy.c / sec_audit.c / sec_vault.c / sec_d.h |
| audit_d | 6 | audit_d.c / audit_collect.c / audit_chain.c / audit_persist.c / audit_query.c / audit_d.h |
| dev_d | 6 | dev_d.c / dev_drv.c / dev_tool.c / dev_plugin.c / dev_discover.c / dev_d.h |
| **合计** | **74** | — |

## 附录 C：v1.1 系统调用清单（[SC] syscalls.h，4 核心 + 20 预留 = 24 槽位）

> **v1.1 Capability Folding 决策**：8 个 seL4 风格 IPC 原语（SEND/RECV/NBSEND/NBRECV/REPLYRECV/YIELD/REPLY/NOTIFY）已全部移除，IPC 数据面完全由 io_uring `IORING_OP_URING_CMD` 承载（零 syscall）。

| # | 系统调用 | 编号 | 类别 | 说明 |
|---|---------|------|------|------|
| 1 | `AIRY_SYS_CALL` | 512 | Capability Invocation | sec_d 专属管理入口（Badge 编译/撤销 + LSM_ctl + Wasm_load） |
| 2 | `AIRY_SYS_ROVOL_CTL` | 513 | 控制原语 | MemoryRovol 记忆卷载控制 |
| 3 | `AIRY_SYS_SCHED_CTL` | 514 | 控制原语 | 调度策略配置 |
| 4 | `AIRY_SYS_CLT_NOTIFY` | 515 | 控制原语 | CoreLoopThree 通知 + kthread 注册 |
| 5-24 | 预留 | 516-535 | 预留 | 20 个预留槽位（未来扩展） |

**废弃 syscall 清单**（v1.0 → v1.1 移除）：

| # | 废弃系统调用 | 编号 | 废弃原因 |
|---|------------|------|---------|
| — | `AIRY_SYS_SEND` | 1 | v1.1 Capability Folding：IPC 数据面零 syscall，由 io_uring 承载 |
| — | `AIRY_SYS_RECV` | 2 | 同上 |
| — | `AIRY_SYS_NBSEND` | 3 | 同上 |
| — | `AIRY_SYS_NBRECV` | 4 | 同上 |
| — | `AIRY_SYS_REPLYRECV` | 5 | 同上 |
| — | `AIRY_SYS_YIELD` | 6 | 同上 |
| — | `AIRY_SYS_REPLY` | 7 | 同上 |
| — | `AIRY_SYS_NOTIFY` | 11 | 同上 |

---

**文档结束**

> 本文件是 agentrt-linux（AirymaxOS）整体目录结构的唯一权威源。任何目录规划、文件创建、构建配置必须以本文件为准。变更需经架构委员会审核并登记于 §12.3 变更日志。
>
> **设计哲学声明**：本目录结构遵循乔布斯/艾夫"简约就是美"哲学——深入理解产品本质后实现的真正简约，而非表面极简。每个目录、每个文件都有清晰的职责说明，体现"看不见的部分应如外观一样精致"的工程美学。
