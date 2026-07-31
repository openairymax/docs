Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# ALK-6.6 内核总体架构（SSoT）

> **文档定位**：agentrt-linux（AirymaxOS）ALK-6.6（AirymaxOS Linux Kernel 6.6）内核总体架构的唯一权威定义文档。本文件定义 ALK-6.6 的物理形态、内核基线、改造点清单、技术支柱、syscall 架构、子系统概要与微内核化改造路径。所有涉及 ALK-6.6 内核架构的文档必须以本文件为准\
> **文档版本**：v1.0.1（v3.1 重建）\
> **最后更新**：2026-07-30\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：Linux 6.6 LTS 内核基线 + seL4 微内核工程思想（ADR-014）\
> **编号权威**：[09-ssot-registry.md §3](../50-engineering-standards/09-ssot-registry.md)\
> **SPDX-License-Identifier**：CC-BY-NC-4.0

***

## SSoT 声明

> **单一权威源声明**：本文件是 **ALK-6.6 内核总体架构** 的唯一权威源。ALK-6.6 等式、内核基线版本、改造点清单、技术支柱清单、syscall 编号段（548-571）、6 类内核职责边界均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 ALK-6.6 物理形态与改造范围。
>
> **子主题权威源引用（免责声明）**：本文件作为总体架构入口，对子主题仅提供概述。以下子主题以各自 SSoT 文档为唯一权威，本文件概述如与子 SSoT 冲突，以子 SSoT 为准：
>
> | 子主题                   | 唯一权威 SSoT 文档                                                                                          |
> | --------------------- | ----------------------------------------------------------------------------------------------------- |
> | Agent 8 态生命周期         | [10-sc-sched-extension.md §2](../30-interfaces/10-sc-sched-extension.md)                              |
> | syscall 编号与 op-dispatch | [07-syscall-registry.md](../140-application-development/07-syscall-registry.md)                       |
> | LSM 框架与排序             | [01-lsm-framework.md](../110-security/01-lsm-framework.md)                                            |
> | Capability 安全模型       | [03-capability-model.md](../110-security/03-capability-model.md)                                      |
> | IPC 协议与 fastpath      | [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)                                             |
> | 微内核化改造路径              | [03-microkernel-strategy.md](03-microkernel-strategy.md)                                              |
> | 内核目录结构详细              | [07-directory-structure.md §9](07-directory-structure.md)                                             |
> | 内核子仓详细设计              | [20-modules/01-kernel.md](../20-modules/01-kernel.md)                                                 |
> | 规则编号注册                | [09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)                                |

***

## §1 文档定位与权威声明

### 1.1 文档作用

本文件是 ALK-6.6 内核总体架构的**入口文档**，提供以下权威定义：

| 定义项          | 权威性  | 详细文档                                                                            |
| ------------ | ---- | ------------------------------------------------------------------------------- |
| ALK-6.6 物理形态 | 唯一权威 | —                                                                               |
| 内核基线版本锁定     | 唯一权威 | [04-engineering-baseline.md](04-engineering-baseline.md)                        |
| 改造点清单        | 唯一权威 | —                                                                               |
| 技术支柱清单       | 唯一权威 | —                                                                               |
| syscall 编号段  | 引用权威 | [07-syscall-registry.md](../140-application-development/07-syscall-registry.md) |
| 微内核化改造路径     | 唯一权威 | [03-microkernel-strategy.md](03-microkernel-strategy.md)                        |
| 内核目录结构       | 引用权威 | [07-directory-structure.md](07-directory-structure.md) §9                       |
| 内核子仓详细设计     | 引用权威 | [20-modules/01-kernel.md](../20-modules/01-kernel.md)                           |

### 1.2 与其他文档的关系

```
00-alk-kernel-overview.md（本文件，总体架构入口）
  ├── 03-microkernel-strategy.md（微内核设计思想详解）
  ├── 06-iron9-shared-model.md（IRON-9 v3 四层共享模型）
  ├── 10-unify-design.md（Airymax Unify Design 五模块总纲）
  ├── 07-directory-structure.md（整体目录结构，§9 含 ALK 内核内部）
  ├── 20-modules/01-kernel.md（内核子仓详细设计）
  ├── 30-interfaces/01-syscalls.md（syscall 接口设计）
  ├── 30-interfaces/02-ipc-protocol.md（IPC 协议设计）
  ├── 110-security/01-lsm-framework.md（LSM 框架详解）
  ├── 110-security/03-capability-model.md（Capability 安全模型）
  └── 140-application-development/07-syscall-registry.md（syscall 编号 SSoT）
```

***

## §2 内核基线锁定

### 2.1 基线版本

| 维度        | 值                                       | 决策依据    |
| --------- | --------------------------------------- | ------- |
| 内核版本      | Linux 6.6 LTS                           | ADR-013 |
| 基线类型      | **vanilla**（非变种）                        | ADR-013 |
| 版本锁定策略    | 1.x.x 锁定 Linux 6.6 / 2.x.x 锁定 Linux 7.1 | ADR-016 |
| seL4 理论基础 | seL4 微内核工程思想                            | ADR-014 |
| 技术路线      | 基于 Linux 内核微内核化改造，非从零开发                 | ADR-012 |

### 2.2 选择 Linux 6.6 的理由

1. **LTS 长期支持**：Linux 6.6 是 LTS 版本，维护周期至 2026 年底+，适合生产就绪目标
2. **io\_uring 成熟**：6.6 的 io\_uring 子系统已成熟，支持 `IORING_OP_URING_CMD` + registered buffer + SQPOLL
3. **sched\_ext 不在 6.6 基线**：sched\_ext 于 Linux 6.12 合入主线，6.6 LTS 基线（vanilla）不含 sched\_ext。agentrt-linux 锁定 6.6 基线自然不依赖 sched\_ext；即便未来升级基线，仍坚持 sched\_tac 路线（H5 纯 C LSM 约束，sched\_ext 依赖 BPF\_SYSCALL/BPF\_JIT/DEBUG\_INFO\_BTF）
4. **Landlock 纯 C 实现**：6.6 的 Landlock 是纯 C 实现，使用专用 syscall 而非 BPF 程序，符合 H5 约束
5. **EEVDF 调度器**：6.6 引入 EEVDF（Earliest Eligible Virtual Deadline First）调度类，sched\_tac 可组合使用

### 2.3 不使用 sched\_ext 的理由

> **前瞻性约束**：6.6 LTS 基线不含 sched\_ext（sched\_ext 于 6.12 合入主线）。以下为即便未来升级基线也拒绝采纳 sched\_ext 的设计理由，sched\_tac 路线不可妥协。

| # | 理由                                                       | 代码证据                                                                 |
| - | -------------------------------------------------------- | -------------------------------------------------------------------- |
| 1 | sched\_ext 依赖 `BPF_SYSCALL && BPF_JIT && DEBUG_INFO_BTF` | Linux 6.12+ `kernel/Kconfig.preempt` CONFIG\_SCHED\_EXT              |
| 2 | 与 H5 纯 C LSM 硬约束冲突                                       | [03-capability-model.md §13](../110-security/03-capability-model.md) |
| 3 | BPF verifier 在调度路径的语义不确定性影响形式化验证                         | [03-microkernel-strategy.md §7](03-microkernel-strategy.md)          |
| 4 | sched\_ext 的 BPF struct\_ops 模型与纯 C 体系不一致                | [01-lsm-framework.md §7](../110-security/01-lsm-framework.md)        |

***

## §3 ALK-6.6 物理形态定义

### 3.1 ALK-6.6 等式

```
ALK-6.6 = Linux 6.6 LTS (vanilla)                       [IRON-7 基线]
         + 国产硬件驱动层（LAYER，复用 openEuler 硬件适配）  [架构可选，非核心子系统]
         + agentrt-linux 微内核化改造补丁（≤ 2 万行）        [核心子系统改造]
         + [SC] 共享契约层（10 个头文件）
         + 4 核心 syscall（548-551）+ op-dispatch
         + io_uring 数据面（IORING_OP_URING_CMD）
         + 纯 C LSM（airy_lsm，H5 硬约束）
         + Micro-Supervisor（kernel/superv/，4 个 .c）
         + Capability 静态数组（agent_caps[1024]，128KB）
```

> **层序说明**：国产硬件驱动层（复用openEuler的硬件适配能力）与微内核化改造补丁相互正交——驱动层仅触及 `drivers/`、`arch/` 硬件适配，不触及调度/安全/IPC/内存核心子系统；微内核化改造不依赖驱动层，两者可独立构建。核心子系统基线纯净性由 IRON-7 保障（详见 §3.5）。

### 3.2 内核目录结构概要

ALK-6.6 在 Linux 6.6 原生目录树基础上新增以下 agentrt-linux 专属目录：

```
kernel/                          # Linux 6.6 内核源码树
├── include/
│   ├── uapi/linux/airymax/      # [SC] 共享契约层（10 头文件 + syscall.xml）
│   │   ├── error.h              # A-UEF 统一错误码
│   │   ├── log_types.h          # A-ULP 统一日志类型
│   │   ├── ipc.h                # IPC magic + 128B 消息头
│   │   ├── sched.h              # sched_tac 约束 + 任务描述符
│   │   ├── memory_types.h       # MemoryRovol L1-L4 数据结构
│   │   ├── security_types.h     # 44 cap（41 POSIX + 3 Airymax）+ Badge 访问宏
│   │   ├── cognition_types.h    # 三阶段枚举
│   │   ├── syscalls.h           # 4 核心 syscall 编号（548-551）
│   │   ├── uapi_compat.h        # __KERNEL__/__linux__ 桥接
│   │   ├── lsm_types.h          # DEFINE_LSM(airy) 骨架
│   │   └── syscall.xml          # codegen 契约源（R-01）
│   └── airymax/                 # 内核内部头文件
├── kernel/
│   ├── corekern/                # 核心内核改造（api/bpf/ipc/irq/locking/memory/object/sched/taskflow/time）
│   └── superv/                  # Micro-Supervisor（4 个 .c + Kbuild）
│       ├── airy_cap_check_superv.c    # Capability 检查（superv 侧）
│       ├── airy_eventfd_superv.c      # eventfd 机制
│       ├── airy_ipc_freeze_superv.c   # IPC 冻结
│       └── airy_superv_init.c         # Micro-Supervisor 初始化入口
├── security/
│   └── airy/                    # Airy LSM（纯 C，H5 硬约束）
│       ├── Kbuild               # 构建配置
│       ├── Kconfig              # 配置选项
│       ├── airy_cap.h           # Capability 头文件
│       ├── airy_lsm.c           # LSM 主入口（DEFINE_LSM(airy)）
│       ├── airy_cap_array.c     # agent_caps[1024] 静态数组
│       ├── airy_cap_check.c     # fastpath C-S9 Badge 校验
│       ├── airy_cap_derive.c    # Capability 派生（Copy/Mint/MintCopy）
│       ├── airy_cap_rotate.c    # Capability 轮换
│       ├── airy_die_notify.c    # Agent 死亡通知（security 侧）
│       ├── airy_eventfd.c       # eventfd（security 侧）
│       ├── airy_ipc_freeze.c    # IPC 冻结（security 侧）
│       └── airy_superv_lsm.c    # Supervisor LSM（security 侧）
├── arch/x86/entry/syscalls/
│   └── syscall_64.tbl           # 新增段 548-551（4 核心 syscall）
└── include/uapi/asm-generic/
    └── unistd.h                 # __NR_airy_sys_call 548 ~ __NR_airy_sys_clt_notify 551
```

> **完整目录结构**：详见 [07-directory-structure.md §9](07-directory-structure.md)。

### 3.3 改造点清单

ALK-6.6 对 Linux 6.6 vanilla 的改造点清单：

| #      | 改造区域                                     | 改造内容                                    | 新增/修改  | 代码预估           |
| ------ | ---------------------------------------- | --------------------------------------- | ------ | -------------- |
| 1      | `include/uapi/linux/airymax/`            | \[SC] 共享契约层 10 头文件 + syscall.xml        | 新增     | \~2,000 行      |
| 2      | `include/uapi/asm-generic/unistd.h`      | 新增 4 核心 syscall 编号（548-551）             | 修改     | \~10 行         |
| 3      | `arch/x86/entry/syscalls/syscall_64.tbl` | 新增 4 核心 syscall 表项                      | 修改     | \~4 行          |
| 4      | `kernel/superv/`                         | Micro-Supervisor（4 个 .c + Kbuild）       | 新增     | \~3,000 行      |
| 5      | `kernel/corekern/`                       | 核心内核改造（调度/IPC/内存/capability 原语）         | 新增     | \~15,000 行     |
| 6      | `security/airy/`                         | Airy LSM（纯 C，H5 硬约束）                    | 新增     | \~5,000 行      |
| 7      | `include/airymax/`                       | 内核内部头文件                                 | 新增     | \~3,000 行      |
| 8      | io\_uring 子系统                            | `IORING_OP_URING_CMD` 语义映射 + opcode 白名单 | 修改     | \~2,000 行      |
| 9      | sysctl                                   | 三级紧急刹车机制（io\_uring 控制）                  | 修改     | \~500 行        |
| **合计** | <br />                                   | <br />                                  | <br /> | **\~30,514 行** |

> **体量约束**：微内核化改造补丁总量控制在 ≤ 2 万行核心代码 + \~1 万行头文件/构建配置。每新增子系统需论证"无法在用户态安全实现"（Liedtke minimality principle）。

### 3.4 技术支柱清单

ALK-6.6 的 6 大技术支柱：

| # | 技术支柱                | seL4 借鉴                         | Linux 6.6 落地                                                            | 硬约束   |
| - | ------------------- | ------------------------------- | ----------------------------------------------------------------------- | ----- |
| 1 | **Capability 安全模型** | CNode + MDB 派生树                 | 纯 C LSM + `agent_caps[1024]` 静态数组 + Badge 64-bit Native Word            | H1-H6 |
| 2 | **IPC 消息传递**        | Endpoint 三态状态机                  | io\_uring `IORING_OP_URING_CMD` + 128B 定长消息头 + fastpath C-S9 Badge 内联校验 | H1-H6 |
| 3 | **调度原语**            | priority + round-robin + domain | sched\_tac（SCHED\_DEADLINE / SCHED\_FIFO / EEVDF 调度类组合 + seL4 MCS 语义映射） | —     |
| 4 | **地址空间管理**          | VSpace + Page Table             | Linux mm/（保留原生）                                                         | —     |
| 5 | **中断/异常处理**         | Interrupt 对象 + fault handler    | Linux IRQ + fault handler（保留原生）                                         | —     |
| 6 | **内存 Retype**       | Untyped → Typed 转换              | buddy allocator + Retype 层（新增）                                          | —     |

### 3.5 国产硬件驱动层（LAYER 方案）

#### 3.5.1 定位

ALK-6.6 在 vanilla 基线之上叠加**国产硬件驱动层**——该驱动层**复用openEuler的硬件适配能力**，以广泛兼容国产主流硬件（鲲鹏/飞腾/海光/申威/昇腾）。驱动层与微内核化改造正交，仅触及 `drivers/`、`arch/` 硬件适配，**不触及调度/安全/IPC/内存核心子系统**，核心子系统基线纯净性由 IRON-7 保障。对齐语境详见 [README.md 参考声明](../README.md)。

#### 3.5.2 物理布局

```
kernel/                          # ALK-6.6 源码树
├── arch/sw_64/                  # 申威 SW_64 架构（vanilla 不含，完整导入）
├── patches/hw-vendor/           # 国产硬件驱动补丁（LAYER 提取，按架构组织）
│   ├── x86/                     # x86 厂商驱动（海光/阿里等）
│   ├── arm64/                   # arm64 厂商驱动（鲲鹏/飞腾等）
│   └── sw_64/                   # sw_64 厂商驱动（申威）
└── configs/
    ├── hw-vendor_x86.config     # x86 硬件 CONFIG 碎片
    ├── hw-vendor_arm64.config   # arm64 硬件 CONFIG 碎片
    └── hw-vendor_sw64.config    # sw_64 硬件 CONFIG 碎片
```

#### 3.5.3 提取边界

| 类别 | 提取 | 说明 |
|------|------|------|
| 厂商 NIC 驱动 | 是 | 鲲鹏 hns3、华为 hinic3 等 |
| 厂商 PMU 驱动 | 是 | 海思 PMU、阿里 DRW PMU 等 |
| 厂商 cpufreq 驱动 | 是 | 申威 sunway-cpufreq 等 |
| 总线/检测驱动 | 是 | UnifiedBus、ROH、CPU SDC 检测、XCU 等 |
| Vendor Hooks | 是（仅硬件相关） | `drivers/hooks/` 中硬件相关 tracepoint 钩子 |
| 完整架构 | 是 | `arch/sw_64/`（vanilla 不含） |

#### 3.5.4 排除清单（核心子系统纯净性保障）

| 排除项 | 原因 |
|--------|------|
| QoS 网格调度（grid）+ soft\_domain + smt\_qos + bpf\_sched | 与 sched\_tac 冲突，不可共存 |
| security/ 全部修改 | Airymax 采用 vanilla security + airy\_lsm（H5 纯 C LSM） |
| 上游 LTS 变种的版本构建系统（Makefile.oever 类） | Airymax 自有 airy\_defconfig + configs/defconfig\{,-agent,-embedded\} |
| KABI 兼容标记 | 保持 vanilla 头文件洁净，Airymax 无 KABI 约束 |
| sched\_ext | 6.6 基线不含；H5 纯 C 约束禁止（详见 §2.3） |

#### 3.5.5 一致性约束

1. **IRON-7 不变**：驱动层为硬件适配附加层，vanilla linux-stable 6.6.145 仍为唯一核心基线
2. **正交性**：驱动层补丁不得修改 `kernel/sched/`、`security/`、`kernel/corekern/`、`kernel/superv/`、`include/uapi/linux/airymax/` 任何文件
3. **可剥离**：驱动层为架构可选项，不应用驱动层时 ALK-6.6 仍可在 vanilla 支持的架构上构建运行
4. **开源披露政策**：硬件适配语境下允许在开源文档中明确表述"复用openEuler的硬件适配能力"（BAN-INTERNAL-002 硬件适配语境豁免）；驱动层提取的逐文件差异数据仍属内部参考，不进入开源文档

***

## §4 Syscall 架构（v1.0.1 Capability Folding）

### 4.1 架构概览

v1.0.1 采用 **Capability Folding 单平面架构**——控制面仅保留 4 个低频管理 syscall，IPC 数据传递与能力校验全部折叠到 io\_uring 数据面 fastpath：

```
┌─────────────────────────────────────────────────────────────────┐
│                    v1.0.1 Capability Folding                     │
├──────────────────────┬──────────────────────────────────────────┤
│    控制面（4 syscall）  │      数据面（io_uring，零 syscall）        │
│                      │                                          │
│  548 airy_sys_call   │  IORING_OP_URING_CMD                     │
│      (sec_d 专属管理)  │    ├── AIRY_URING_CMD_IPC_SEND           │
│  549 airy_sys_rovol_ctl│    ├── AIRY_URING_CMD_IPC_RECV           │
│      (记忆卷载控制)     │    ├── AIRY_URING_CMD_IPC_SEND_BATCH     │
│  550 airy_sys_sched_ctl│    ├── AIRY_URING_CMD_IPC_CANCEL         │
│      (调度策略配置)     │    ├── AIRY_URING_CMD_IPC_FREEZE         │
│  551 airy_sys_clt_notify│    ├── AIRY_URING_CMD_IPC_CAP_REQUEST   │
│      (阶段通知+kthread)│    └── AIRY_URING_CMD_IPC_CAP_RESPONSE   │
│                      │                                          │
│  op-dispatch 分派     │  fastpath C-S9 Badge 内联校验（~10ns）    │
└──────────────────────┴──────────────────────────────────────────┘
```

### 4.2 4 核心 syscall（编号 548-551）

| 编号      | 符号                    | 分类                    | 功能                           | op-dispatch                                                                                                                                   |
| ------- | --------------------- | --------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| 548     | `AIRY_SYS_CALL`       | Capability Invocation | sec\_d 专属管理入口                | COMPILE\_BADGE / REVOKE\_BADGE / LSM\_CTL / WASM\_LOAD / LSM\_AUDIT\_QUERY / CAP\_INSPECT / CAP\_TRANSFER                                     |
| 549     | `AIRY_SYS_ROVOL_CTL`  | 控制原语                  | MemoryRovol L1-L4 控制         | SNAPSHOT / RESTORE / MIGRATE / TIER\_SET / TIER\_GET / MGLRU\_CONFIG / LIST / DELETE / DEMOTE / PROMOTE                                       |
| 550     | `AIRY_SYS_SCHED_CTL`  | 控制原语                  | sched\_tac 调度策略配置            | SET\_POLICY / GET\_POLICY / REGISTER\_BPF / GET\_LATENCY / SET\_PRIORITY / GET\_VTIME / YIELD                                                 |
| 551     | `AIRY_SYS_CLT_NOTIFY` | 控制原语                  | CoreLoopThree 阶段通知 + kthread | PHASE\_NOTIFY / REGISTER\_KTHREAD / UNREGISTER\_KTHREAD / SET\_MODE / GET\_METRICS / WASM\_LOAD\_MODULE / WASM\_UNLOAD\_MODULE / WASM\_INVOKE |
| 552-571 | 预留                    | —                     | 未来扩展                         | —                                                                                                                                             |

> **编号权威**：[07-syscall-registry.md](../140-application-development/07-syscall-registry.md) 是 syscall 编号的唯一 SSoT。编号 548 起始，避开 x86\_64 x32 历史遗留区域 512-547。

### 4.3 io\_uring 数据面

IPC 数据传递完全由 io\_uring `IORING_OP_URING_CMD` + `cmd_op` 区分操作承载，**零 syscall**：

- **fastpath C-S9**：`airy_cap_badge_ok()` 内联 Badge 校验（\~10ns，3 个 READ\_ONCE + 位运算 + 比较）
- **registered buffer**：预注册内存区域，零拷贝传输
- **opcode 白名单**：仅允许 `IORING_OP_URING_CMD` / `IORING_OP_NOP`，其余 opcode 禁止
- **三级紧急刹车**：sysctl 控制 io\_uring 可用性（0=可用 / 1=仅 CAP\_SYS\_ADMIN / 2=完全禁用）

> **详细设计**：详见 [02-ipc-protocol.md §4.4](../30-interfaces/02-ipc-protocol.md)。

***

## §5 Capability 安全模型概要

### 5.1 模型选型

agentrt-linux 选择 **seL4 风格 + POSIX 混合** capability 模型：

| 维度    | ACL        | POSIX capability | seL4 capability | agentrt-linux       |
| ----- | ---------- | ---------------- | --------------- | ------------------- |
| 权限传递  | 通过组/用户     | 进程位图             | 不可伪造令牌          | seL4（Agent 间传递）     |
| 权限撤销  | 困难（遍历 ACL） | 不支持运行时           | 递归级联撤销          | seL4（O(1) Epoch 撤销） |
| 权限粒度  | 对象级        | 41 个粗粒度 cap      | 对象 + 权限掩码       | seL4 + POSIX 混合     |
| 形式化验证 | 无          | 无                | 有（seL4 已验证）     | seL4（参考验证方法）        |

### 5.2 核心数据结构

| 数据结构                 | 说明                                  | 物理宿主                             |
| -------------------- | ----------------------------------- | -------------------------------- |
| `struct airy_cnode`  | Capability Node（单个 cap 槽位）          | `security/airy/airy_cap.h`       |
| `struct airy_cspace` | Agent 的 Capability Space            | `security/airy/airy_cap.h`       |
| `agent_caps[1024]`   | 内核静态数组（128KB，per-Agent 索引）          | `security/airy/airy_cap_array.c` |
| Badge 64-bit         | Epoch<<48 \| RandomTag<<16 \| Perms | `ipc.h` offset 40-47             |

### 5.3 7 种 CNode 操作原语

| 操作     | 语义                    | 可撤销性     |
| ------ | --------------------- | -------- |
| Copy   | 复制 cap（父子派生）          | 可 Revoke |
| Mint   | 派生 cap（可限制权限 + badge） | 可 Revoke |
| Move   | 移动 cap（源→目标）          | 不可逆      |
| Mutate | 修改 cap（权限 + badge）    | 不可逆      |
| Revoke | 递归撤销子 cap             | 不可逆      |
| Delete | 删除单个 cap              | 不可逆      |
| Rotate | 轮换 cap（Badge 刷新）      | 不可逆      |

### 5.4 Badge 撤销机制（三层互补）

agentrt-linux 采用**三层互补**的 Badge/capability 撤销机制，覆盖不同粒度：

| 层级 | 机制 | 复杂度 | 触发场景 | 代码位置 |
|------|------|--------|---------|---------|
| ① REVOKE CNode 操作 | `airy_cap_revoke_subtree()` 沿 MDB 派生树（左孩子右兄弟）递归撤销子 cap | O(N)（典型深度 ≤3） | 撤销某 cap 的所有派生子 cap | [airy_cap_derive.c:92-117](../../agentrt-linux/kernel/security/airy/airy_cap_derive.c#L92-L117) |
| ② per-agent epoch bump | `airy_cap_epoch_bump(agent_id)` 递增 slot_epoch，该 Agent 旧 Badge 立即失效 | O(1) | 撤销某 Agent 的所有 Badge（K9-1 主要撤销机制） | `security/airy/airy_cap_array.c` |
| ③ cancelBadgedSends | `airy_ipc_cancel_badged_sends(ring, badge)` 扫描 ring 中待发送消息，匹配 badge 者将 magic 置零使其被跳过 | O(N)（N=ring 深度） | 撤销指定 Badge 的待发送 IPC 消息 | [airy_ipc_ring.c:159-207](../../agentrt-linux/kernel/kernel/corekern/ipc/airy_ipc_ring.c#L159-L207) |

**seL4 对比**：seL4 的 `cteRevoke` 需递归遍历 MDB 派生树，复杂度 O(N)；agentrt-linux 的 ②层将复杂度从撤销方的 O(N) 转移到被撤销方的重新申请（O(1)），①层保留 seL4 风格的递归撤销语义，③层补齐 seL4 `endpoint.c:476-489` 的 `cancelBadgedSends` 对等能力。

> **详细设计**：详见 [03-capability-model.md](../110-security/03-capability-model.md) 与 [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)。

***

## §6 IPC 子系统概要

### 6.1 架构

| 维度       | seL4 实现                        | agentrt-linux 实现                     |
| -------- | ------------------------------ | ------------------------------------ |
| 通信模型     | Endpoint 三态状态机（Idle/Send/Recv） | io\_uring SQ/CQ ring 异步消息            |
| 消息头      | MessageInfo 紧凑编码               | 128B 定长消息头（Layout C v4）              |
| 消息传递     | MR（物理寄存器）+ IPC Buffer          | io\_uring SQE 字段 + registered buffer |
| Fastpath | POINT OF NO RETURN 原子提交        | C-S9 Badge 内联校验（\~10ns）              |
| 零拷贝      | copyMRs 跨架构消息复制                | io\_uring 语义映射零拷贝                    |

### 6.2 IPC magic

| magic   | 十六进制         | ASCII  | 用途       | 共享层   |
| ------- | ------------ | ------ | -------- | ----- |
| IPC 消息头 | `0x41524531` | 'ARE1' | IPC 协议识别 | \[SC] |
| 任务描述符   | `0x41475453` | 'AGTS' | 任务描述符识别  | \[SC] |

### 6.3 128B 定长消息头（Layout C v4）

```
offset  0-3:   magic (0x41524531 'ARE1')
offset  4-5:   opcode (AIRY_IPC_OP_*)
offset  6-7:   flags (AIRY_IPC_FLAG_*)
offset  8-15:  trace_id (64-bit)
offset 16-23:  timestamp_ns (64-bit monotonic)
offset 24-31:  src_task (64-bit)
offset 32-39:  dst_task (64-bit)
offset 40-47:  capability_badge (64-bit Native Word)
offset 48-51:  payload_len (32-bit)
offset 52-55:  crc32 (32-bit)
offset 56-127: reserved[72]
```

> **SSoT**：字段布局以 [SC] `include/uapi/linux/airymax/ipc.h` 中 `struct airy_ipc_msg_hdr` 为权威源。详细设计见 [02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)。

***

## §7 调度子系统概要

### 7.1 sched\_tac 框架

sched\_tac（sched\_tac = **sched**uling **t**hrough **a**gent **c**lasses）是 agentrt-linux 的调度策略框架：

| 维度           | 说明                                              |
| ------------ | ----------------------------------------------- |
| 调度类组合        | SCHED\_DEADLINE + SCHED\_FIFO + EEVDF           |
| 语义映射         | seL4 MCS（Mixed-Criticality System）语义映射          |
| 策略位置         | 用户态（stc\_agent 策略枚举 + 策略 daemon）                |
| 机制位置         | 内核态（调度类组合 + cgroup cpuset 隔离）                   |
| 非 sched\_ext | 不使用 sched\_ext（H5 纯 C LSM 约束，sched\_ext 依赖 BPF） |

### 7.2 Agent 8 态生命周期

> **SSoT 对齐**：本节为概述，Agent 8 态生命周期的唯一权威源为 [10-sc-sched-extension.md §2](../30-interfaces/10-sc-sched-extension.md)。本节枚举与状态迁移必须与 SSoT 保持一致，任何变更必须先在 SSoT 中执行。

| 状态         | 说明                  | Linux 进程状态                | 调度策略                | 触发者                                  |
| ------------ | ------------------- | ------------------------ | ------------------- | ------------------------------------ |
| INACTIVE     | 进程不存在，等待 fork       | —                        | —                   | Macro-Supervisor `fork()`            |
| SPAWNING     | fork/exec 中，未就绪     | TASK_RUNNING（过渡）         | SCHED_NORMAL        | exec 完成 → READY                      |
| READY        | 在运行队列等待，可被调度        | TASK_RUNNING             | SCHED_DEADLINE      | Macro-Supervisor `sched_setattr()`   |
| RUNNING      | 正在 CPU 执行           | TASK_RUNNING             | SCHED_DEADLINE      | 调度器挑选                                |
| BLOCKED      | 等待 IPC/IO           | TASK_INTERRUPTIBLE       | —                   | IPC/IO 等待                            |
| STOPPING     | SIGSTOP 发送中，正在冻结 IPC | 过渡态                      | —                   | Micro-Supervisor `kill(SIGSTOP)`     |
| STOPPED      | 已冻结，待裁决             | TASK_STOPPED             | —                   | SIGSTOP 生效                           |
| DEAD         | 等待 waitpid 回收       | EXIT_ZOMBIE              | —                   | `exit()` / `kill(SIGKILL)`           |

> **设计要点**：sched_tac 核心成果——8 态与 Linux 进程状态天然映射，无需新增内核调度器状态，仅复用 SCHED_DEADLINE/SCHED_FIFO/EEVDF。状态迁移由 Macro-Supervisor 驱动，Micro-Supervisor 仅在检测到异常时触发 RUNNING → STOPPING 的强制迁移。

> **详细设计**：详见 [10-sc-sched-extension.md §2](../30-interfaces/10-sc-sched-extension.md)（SSoT）与 [20-modules/01-kernel.md §7](../20-modules/01-kernel.md)。

***

## §8 内存子系统概要

### 8.1 MemoryRovol L1-L4 分层

| 层级 | 存储介质           | 访问延迟    | 用途    |
| -- | -------------- | ------- | ----- |
| L1 | DRAM（热数据）      | \~100ns | 当前工作集 |
| L2 | DRAM（冷数据）      | \~100ns | 近期工作集 |
| L3 | PMEM / CXL 内存  | \~300ns | 记忆快照  |
| L4 | 持久存储（SSD/NVMe） | \~10μs  | 长期记忆  |

### 8.2 内存管理机制

| 机制     | Linux 6.6 原生    | agentrt-linux 增强                                         |
| ------ | --------------- | -------------------------------------------------------- |
| 物理内存分配 | buddy allocator | 保留 + Retype 层（新增）                                        |
| 页面回收   | LRU / MGLRU     | MGLRU 配置（`AIRY_ROVOL_MGLRU_CONFIG`）                      |
| 内存分层   | 默认              | CXL 内存分层（`AIRY_ROVOL_TIER_SET/GET`）                      |
| 记忆快照   | —               | userfaultfd + MemoryRovol（`AIRY_ROVOL_SNAPSHOT/RESTORE`） |
| 跨节点迁移  | —               | MemoryRovol 迁移（`AIRY_ROVOL_MIGRATE`）                     |

> **详细设计**：详见 [20-modules/01-kernel.md §8](../20-modules/01-kernel.md)。

***

## §9 安全子系统概要

### 9.1 安全体系分层

| 层级     | 类型                        | 机制                      | 优先级                   |
| ------ | ------------------------- | ----------------------- | --------------------- |
| L1     | LSM 框架                    | `security_hook_heads`   | —                     |
| L2     | Landlock                  | 用户态沙箱                   | after L3              |
| **L3** | **capability（airy\_lsm）** | **seL4 风格 + POSIX，纯 C** | **LSM\_ORDER\_MUTABLE**（CONFIG\_LSM 置于 capability 之后第一位） |
| L4     | 代码完整性验证                  | 编译期校验 + CBMC 形式化验证（H5，不依赖 BPF） | 独立                    |
| L7     | Cupolas                   | Agent 行为约束              | 基于 capability         |

### 9.2 H5 纯 C LSM 硬约束

**H5**：agentrt-linux 的 airy\_lsm 是**纯 C 实现**，禁止依赖 BPF：

| 禁止依赖             | 理由                   |
| ---------------- | -------------------- |
| `BPF_SYSCALL`    | BPF 系统调用引入用户态可编程内核入口 |
| `BPF_JIT`        | JIT 编译引入代码生成不确定性     |
| `DEBUG_INFO_BTF` | BTF 依赖 BPF 基础设施      |

**允许使用**：Landlock（纯 C 实现的 LSM，使用专用 syscall 而非 BPF 程序）。

### 9.3 fastpath / slowpath 严格分工

| 路径           | 执行内容                                     | 禁止                           |
| ------------ | ---------------------------------------- | ---------------------------- |
| **fastpath** | C-S9 Badge 内联校验（\~10ns）                  | 禁止调用 LSM 钩子                  |
| **slowpath** | LSM 策略裁决（airy\_lsm + Landlock + Cupolas） | 禁止再次调用 `airy_cap_badge_ok()` |

> **单快照语义**：fastpath 与 slowpath 严格禁止重复校验，维护单快照语义避免 TOCTOU 漏洞。

> **详细设计**：详见 [01-lsm-framework.md](../110-security/01-lsm-framework.md) 与 [03-capability-model.md](../110-security/03-capability-model.md)。

***

## §10 IRON-9 v3 四层模型落地

### 10.1 四层模型

| 层次               | 共享程度             | 内核内容                                         | 物理宿主                          |
| ---------------- | ---------------- | -------------------------------------------- | ----------------------------- |
| **\[SC] 共享契约层**  | 完全共享代码           | 10 个头文件                                      | `include/uapi/linux/airymax/` |
| **\[SS] 语义同源层**  | 高层 API 语义同源      | sched\_tac / io\_uring 等 30+ 项（struct\_ops 仅用于可观测性，非核心架构，H5 约束） | 各自独立实现                        |
| **\[IND] 完全独立层** | 完全独立             | syscall 表注册 / LSM 钩子 / KABI / Kbuild         | 内核态专属                         |
| **\[DSL] 降级生存层** | \[SC] 损坏时最小可运行子集 | 每个 \[SC] 头文件底部 `#ifdef AIRY_SC_FALLBACK` 降级块 | 自包含                           |

### 10.2 \[SC] 10 头文件清单

| #  | 头文件                 | 职责           | magic / 关键定义                    |
| -- | ------------------- | ------------ | ------------------------------- |
| 1  | `error.h`           | A-UEF 统一错误码  | 38 POSIX 码 + `AIRY_ECFGVERSION` |
| 2  | `log_types.h`       | A-ULP 统一日志类型 | 128B 固定记录 + 5 级日志               |
| 3  | `ipc.h`             | IPC 协议       | magic `0x41524531` + 128B 消息头   |
| 4  | `sched.h`           | 调度约束         | magic `0x41475453` + 任务描述符      |
| 5  | `memory_types.h`    | 内存类型         | MemoryRovol L1-L4               |
| 6  | `security_types.h`  | 安全类型         | 44 cap（41 POSIX + 3 Airymax）+ Badge 访问宏 |
| 7  | `cognition_types.h` | 认知类型         | 三阶段枚举                           |
| 8  | `syscalls.h`        | Syscall 编号   | 4 核心（548-551）                   |
| 9  | `uapi_compat.h`     | UAPI 兼容      | `__KERNEL__` / `__linux__` 桥接   |
| 10 | `lsm_types.h`       | LSM 类型       | `DEFINE_LSM(airy)` 骨架           |

> **详细设计**：详见 [06-iron9-shared-model.md](06-iron9-shared-model.md) 与 [120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md)。

***

## §11 内核代码体量预算

### 11.1 体量控制策略

| 代码区域                         | seL4 基线    | agentrt-linux 预算 | 控制策略                  |
| ---------------------------- | ---------- | ---------------- | --------------------- |
| 内核核心（调度/IPC/capability/内存原语） | \~14,400 行 | 5-10 万行          | 每新增子系统需论证"无法在用户态安全实现" |
| 微内核化改造补丁                     | —          | ≤ 2 万行           | VFS / 网络栈 / 驱动用户态化补丁  |
| \[SC] 共享契约层                  | —          | 10 个头文件          | IRON-9 v3 单一数据源       |
| \[SS] 语义同源层                  | —          | 30+ 项高层 API 语义   | 实现独立，语义同源             |
| \[IND] 独立层                   | —          | 15+ 项            | 内核态专属                 |

### 11.2 审计机制

- **每版本发布前**执行代码体量审计，消除"蠕变增胖"（kernel bloat）
- **新增子系统**需向 TSC 提交"无法在用户态安全实现"论证报告
- **CBMC 形式化验证**：仅针对 fastpath 核心 100 行代码（`airy_cap_badge_ok` / `airy_ipc_validate` 等），结合 KCSAN 运行时检测交叉验证

***

## §12 微内核化改造路径

### 12.1 三阶段改造

| 阶段   | 版本     | 改造内容                                                             | seL4 借鉴                         |
| ---- | ------ | ---------------------------------------------------------------- | ------------------------------- |
| 阶段 1 | 1.x.x  | sched\_tac 调度框架 + io\_uring IPC + 纯 C LSM capability 层引入（H5）     | capability 单一模型（ES-SEL4-05\~09） |
| 阶段 2 | 1.0.1 后续小版本 | VFS 部分用户态化 + 网络栈部分用户态化（DPDK / AF\_XDP）+ 驱动框架用户态化（VFIO / libvfio） | 服务用户态化（ES-SEL4-29\~31）          |
| 阶段 3 | 2.x.x  | 大部分系统服务用户态化 + 完整 capability 安全模型 + 形式化验证（部分核心模块）+ Linux 7.1 基线升级 | 形式化验证预留（ES-SEL4-16\~20）         |

### 12.2 下沉到用户态的 Linux 子系统

| Linux 子系统   | 下沉策略                    | 用户态服务                               | 迁移阶段   |
| ----------- | ----------------------- | ----------------------------------- | ------ |
| VFS（具体文件系统） | 保留 VFS 框架在内核，具体 FS 下沉   | services VFS server                 | 1.0.1 后续小版本 |
| 网络栈（TCP/IP） | 保留 socket 层在内核，协议栈下沉    | services net server（DPDK / AF\_XDP） | 1.0.1 后续小版本 |
| 设备驱动        | 通过 VFIO / libvfio 下沉    | services driver server              | 1.0.1 后续小版本 |
| POSIX 接口    | 通过用户态 POSIX server 兼容   | services POSIX server               | 2.x.x  |
| 信号管道        | eventfd / signalfd 等效替代 | services signal server              | 2.x.x  |

> **详细设计**：详见 [03-microkernel-strategy.md §9](03-microkernel-strategy.md)。

***

## §13 变更历史

| 版本           | 日期             | 主要变更                                                                                                                                                                                                  |
| ------------ | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v1.0.1       | 2026-07-10     | 初版 ALK-6.6 总体架构文档，基于 Linux 6.6 + seL4 微内核化改造路线                                                                                                                                                        |
| v1.0.1       | 2026-07-18     | Capability Folding 单平面架构落地；syscall 12→4 精简（编号 548-551）；fastpath C-S9 Badge 内联校验；H1-H6 硬约束建立；纯 C LSM（H5）确认                                                                                             |
| v1.0.1       | 2026-07-22     | P0-014 修复：syscall 编号统一为 548-551（避开 x86\_64 x32 历史遗留区域 512-547）；codegen 管道 + ABI 检查工具落地                                                                                                                |
| **v3.1（重建）** | **2026-07-23** | **文档完全重建（原文件被意外清空）。基于当前整体方案重写，对齐 v1.0.1 Capability Folding 架构、548-551 syscall 编号、纯 C LSM（H5）、sched\_tac 调度框架。修复 P0-K02（eBPF kfunc → 纯 C LSM）。新增 §2.3 不使用 sched\_ext 的理由、§9.3 fastpath/slowpath 严格分工** |

***

## §14 相关文档

| 文档                                                                                                                          | 关系                         |
| --------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| [01-system-architecture.md](01-system-architecture.md)                                                                      | agentrt-linux 系统架构总览       |
| [03-microkernel-strategy.md](03-microkernel-strategy.md)                                                                    | 微内核设计思想详解                  |
| [06-iron9-shared-model.md](06-iron9-shared-model.md)                                                                        | IRON-9 v3 四层共享模型           |
| [07-directory-structure.md](07-directory-structure.md)                                                                      | 整体目录结构设计                   |
| [10-unify-design.md](10-unify-design.md)                                                                                    | Airymax Unify Design 五模块总纲 |
| [20-modules/01-kernel.md](../20-modules/01-kernel.md)                                                                       | 内核子仓详细设计                   |
| [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md)                                                             | syscall 接口设计               |
| [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)                                                     | IPC 协议设计                   |
| [110-security/01-lsm-framework.md](../110-security/01-lsm-framework.md)                                                     | LSM 框架详解                   |
| [110-security/03-capability-model.md](../110-security/03-capability-model.md)                                               | Capability 安全模型            |
| [140-application-development/07-syscall-registry.md](../140-application-development/07-syscall-registry.md)                 | syscall 编号 SSoT            |
| [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) | 跨项目代码共享                    |

***

## §15 参考文献

1. seL4 微内核源代码（`seL4-master`），verified microkernel
2. Linux 6.6 内核源代码（vanilla），`kernel.org`
3. Jochen Liedtke, "On μ-Kernel Construction", SOSP 1995
4. seL4 CAVEATS.md — verification scope declaration
5. openEuler 24.03 LTS / 26.03 标准规范
6. Steve Jobs / Jony Ive 设计哲学（简约就是美）

