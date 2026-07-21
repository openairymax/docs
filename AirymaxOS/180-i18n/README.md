Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 国际化与本地化设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）国际化与本地化工程体系主索引——locale、错误消息、日志格式与多语言支持\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[AirymaxOS 总览](../README.md)

---

## 1. 模块概述

`180-i18n/` 是 AirymaxOS 文档体系中**面向全球开发者和用户的工程保障**。它继承 Linux 发行版 30+ 年沉淀的国际化哲学（locale + gettext + iconv + Unicode），并在其上扩展智能体操作系统专属的多语言 Agent 提示词、多区域记忆卷载、多语言文档同步、国密算法合规等。

国际化不是「翻译」的代名词，而是「文化适配」的工程体系。AirymaxOS 国际化的核心理念是：**系统界面必须支持多语言，Agent 提示词必须支持多文化，文档必须多语言同步**。这要求从设计阶段就将国际化纳入考量，而非在发布末期才进行翻译。

本模块承担五项核心职责：

1. **Locale 区域设置**：locale + LC_* 环境变量 + 区域格式（日期/时间/数字/货币）。
2. **字符编码规范**：UTF-8 处理 + 编码转换 + GB18030 中文合规。
3. **错误消息国际化**：错误码注册表 SSoT + 多语言消息映射（A-UEF）。
4. **日志格式国际化**：128B 日志记录格式 + 多语言日志格式化（A-ULP）。
5. **CJK 支持与文档国际化**：中文/日文/韩文显示与输入 + 中英双语文档同步。

---

## 2. 技术选型声明

agentrt-linux v1.0 在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度做出**不可妥协**的技术选型。本目录所有国际化设计必须以本声明为基线——多语言错误消息与日志格式化不得引入 sched_ext / page flipping / BPF LSM / DMA 一致性内存相关依赖。

| # | 技术维度 | 选定方案 | 明确不采用 | 选定理由 |
|---|---------|---------|-----------|---------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类，通过 `nice` / `sched_setattr` / `sched_setaffinity` 注入 Agent 感知策略，**不定义新的调度类宏** | **不使用 sched_ext**（不引入 eBPF 调度器、不依赖 `SCHED_EXT=7`、不使用 SCHED_AGENT 宏） | 保留 Linux 6.6 主线调度器稳定性，国际化不影响调度路径，ABI 稳定 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输，registered buffer + mmap 共享页 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | 多语言消息传输复用 IPC 通道，IORING_OP_URING_CMD 在 Linux 6.6 已稳定 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 Linux Security Module（`airy_lsm`），通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | 纯 C LSM 保证安全策略可审计，国际化不引入 BPF 依赖 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `remap_pfn_range` 映射到用户态地址空间 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | 多语言消息缓冲基于 alloc_pages + mmap，跨架构语义一致 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | v2 三层模型（升级为 v3，新增 [DSL] 降级生存层） | [DSL] 降级生存层确保 [SC] 损坏时系统仍可降级运行，多语言消息类型安全 |

> 详细技术选型声明见上级文档 [AirymaxOS 总览](../README.md) §2 与 [`10-architecture/10-unify-design.md`](../10-architecture/10-unify-design.md) SSoT 声明。

---

## 3. 文档索引

`180-i18n/` 由 6 个文档构成（含本 README）：

```
180-i18n/
├── README.md                       # 本文件 — 国际化主索引（v1.0）
├── 01-locale-design.md             # Locale 区域设置设计 + 环境变量（v1.0）
├── 02-encoding-spec.md             # 字符编码规范 + UTF-8 处理 + 编码转换（v1.0）
├── 03-error-message-i18n.md        # 错误消息国际化 + 错误码注册表 SSoT（v1.0）
├── 04-cjk-support.md              # CJK 中文/日文/韩文显示与输入支持（v1.0）
└── 05-doc-i18n.md                 # 文档国际化 + 中英双语同步 + 术语表治理（v1.0）
```

### 3.1 各文档定位

| 文档 | 核心问题 | 主要产物 |
|------|---------|---------|
| README.md | 国际化体系全貌？ | 6 层分层 + A-UEF/A-ULP 映射 + 技术选型声明 |
| 01-locale-design.md | 区域怎么设置？ | Locale + LC_* 环境变量 + 区域格式 |
| 02-encoding-spec.md | 编码怎么处理？ | UTF-8 处理 + 编码转换 + GB18030 合规 |
| 03-error-message-i18n.md | 错误消息怎么多语言？ | 错误码注册表 SSoT + 多语言消息映射（A-UEF） |
| 04-cjk-support.md | CJK 怎么支持？ | 中文/日文/韩文显示 + 输入法（IBus/Fcitx） |
| 05-doc-i18n.md | 文档怎么双语同步？ | 中英双语文档 + 术语表治理 |

### 3.2 国际化分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 字符编码 | UTF-8 / UTF-16 / GB18030 | 字符集 |
| L2 | 区域设置 | locale + LC_* | 区域格式 |
| L3 | 消息国际化 | gettext + .po / .mo | 翻译 |
| L4 | 输入法 | IBus / Fcitx | 输入 |
| **L5** | **Agent 提示词** | **AirymaxOS 专属** | **多语言提示词** |
| **L6** | **记忆卷载区域** | **AirymaxOS 专属** | **多区域记忆** |

### 3.3 错误消息国际化（A-UEF SSoT）

A-UEF 的错误码注册表是多语言消息映射的 SSoT，错误码值（`AIRY_E*`）与语言无关，消息文本通过 gettext 动态映射：

```c
#include <libintl.h>
#include <locale.h>

setlocale(LC_ALL, "");
bindtextdomain("airymaxos", "/usr/share/locale");
textdomain("airymaxos");

/* 错误码值与语言无关，消息文本动态映射 */
printf(gettext("agentrt-linux started: %s\n"), airy_strerror(err));
```

---

## 4. Airymax Unify Design 映射

Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）在国际化体系中的关系如下。国际化是 A-UEF 错误消息 i18n 与 A-ULP 日志格式 i18n 的主要落地维度。

| Unify 模块 | 全称 | 在本目录的体现 | i18n 范围 |
|-----------|------|--------------|----------|
| **A-UEF** | Unified Error and Fault Framework | **核心**：错误消息国际化、`AIRY_E*` 错误码多语言消息映射、错误码注册表 SSoT（语言无关值 + 动态文本） | 错误消息 i18n |
| **A-ULP** | Unified Logging and Printk Subsystem | **核心**：日志格式国际化、128B 日志记录 payload 多语言格式化、Logger Daemon 异步 gettext | 日志格式 i18n |
| **A-UCS** | Unified Configuration Subsystem | 配置项多语言描述、locale 配置热重载、sysctl/JSON 区域设置同步 | 配置 i18n |
| **A-ULS** | Unified Lifecycle Supervision Framework | Agent 8 态生命周期状态多语言描述、故障通知消息 i18n | 状态消息 i18n |
| **A-IPC** | Unified Airymax IPC Fabric | IPC 错误响应消息 i18n、128B 消息头 payload 多语言透传 | IPC 消息 i18n |

### 4.1 A-UEF 错误消息 i18n

A-UEF 的错误码采用**值与文本分离**设计——错误码值（`AIRY_E*` 负数空间 + `AIRY_FAULT_*` 正数空间）与语言无关，多语言消息通过 gettext 动态映射：

| 码空间 | 值域 | i18n 机制 | 多语言消息 |
|--------|------|---------|-----------|
| Error（可恢复） | `[-300, -1]` | `airy_strerror(err)` + gettext | 中文/英文/日文等 |
| Fault（不可恢复） | `[0x1000, 0x1FFF]` | Fault Handler 通知 + gettext | 中文/英文/日文等 |

错误码值通过 [SC] `error.h` 共享契约两端一致（语言无关），消息文本各自独立实现（[IND] 层）。

### 4.2 A-ULP 日志格式 i18n

A-ULP 的 128B 日志记录采用**payload 与格式化分离**设计——内核 fastpath 仅写入 128B 原始二进制 payload（语言无关），多语言格式化由用户态 Logger Daemon 异步完成：

```
内核 fastpath（≤100ns，语言无关）   用户态 Logger Daemon（异步，i18n）
┌──────────────────────────┐        ┌──────────────────────────┐
│ 1. reserve 128B slot     │        │ 1. mmap 读取 Ring Buffer │
│ 2. memcpy 128B raw       │ ──eventfd──▶ 2. sprintf + gettext │
│ 3. commit 原子索引        │        │ 3. 多语言格式化 + 过滤   │
│ 4. eventfd_signal        │        │ 4. 落盘 + 轮转           │
└──────────────────────────┘        └──────────────────────────┘
```

日志内存基于 **alloc_pages + mmap**（**不使用 DMA 一致性内存**），128B 记录格式通过 [SC] `log_types.h` 共享契约两端一致。

### 4.3 同源 agentrt 国际化（IRON-9 v3）

国际化遵循 **IRON-9 v3 四层模型**与 agentrt 同源：[SC] 共享错误码值与日志记录格式（语言无关），[SS] 语义同源（`airy_strerror()` 签名一致），[DSL] 降级生存，[IND] 各自独立实现多语言消息文本（agentrt 用户态 gettext ↔ AirymaxOS 内核 printk + gettext）。

---

## 5. 相关文档

### 5.1 上级与架构文档
- [AirymaxOS 总览](../README.md) —— 文档体系顶层纲领（v1.0）
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) —— Airymax Unify Design 总纲（五模块 SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 四层模型

### 5.2 接口与数据流文档
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— A-UEF [SC] error.h 契约（错误码 SSoT）
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— A-ULP [SC] log_types.h 契约（128B 日志格式）
- [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— Ring Buffer 日志（A-ULP）
- [40-dataflows/06-logger-daemon-design.md](../40-dataflows/06-logger-daemon-design.md) —— Logger Daemon（异步格式化）

### 5.3 关联模块文档
- [110-security/README.md](../110-security/README.md) —— 安全加固（国密算法合规：SM2/SM3/SM4）
- [140-application-development/README.md](../140-application-development/README.md) —— Agent 应用开发（多语言 SDK）
- [50-engineering-standards/01-coding-standards.md](../50-engineering-standards/01-coding-standards.md) —— 编码规范（含国际化）
- [50-engineering-standards/07-maintainers-and-governance.md](../50-engineering-standards/07-maintainers-and-governance.md) —— 多语言社区治理

### 5.4 参考材料
- Linux 6.6 locale 子系统
- gettext 项目 + Unicode 标准
- 国密算法标准（GM/T 系列）

---

## 6. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，5/5 文档完成（locale + 编码 + 错误消息 + CJK + 文档 i18n） |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac 技术选型声明（不使用 sched_ext）；IORING_OP_URING_CMD（不使用 page flipping）；纯 C LSM（不使用 BPF LSM）；alloc_pages + mmap（不使用 DMA 一致性内存）；IRON-9 v3 四层模型（新增 [DSL] 降级生存层）；新增 Airymax Unify Design 五模块映射（A-UEF 错误消息 i18n / A-ULP 日志格式 i18n） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

> **文档结束** | 国际化模块 v1.0 | 共 6 文档 | 维护者：开源极境工程与规范委员会 | 系统界面必须支持多语言
