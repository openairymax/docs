Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 极境内核 ALK-6.6 总体架构规范

> **文档定位**：极境内核（Airymax Linux Kernel 6.6，ALK-6.6）的总体架构权威定义文档。本文档是 ALK-6.6 物理形态、改造方式、微内核化体现、关键设计决策的唯一权威源，所有子文档必须引用本文档，禁止重新定义。
>
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **文档编号**：ALK-ARCH-00\
> **权威等级**：SSoT（Single Source of Truth）\
> **核心约束**：OS-ARCH-001（不修改 Linux 6.6 主线源码，仅增量添加）\
> **设计依据**：ADR-012（微内核化改造技术路线）+ ADR-013（版本基线锁定 Linux 6.6 LTS）+ ADR-014（seL4 唯一思想来源）+ ADR-016（Linux 6.6 LTS 版本基线正式确立）+ IRON-9 v3（四层共享模型）

***

## 目录

- [§1 文档 scope 与权威声明](#§1-文档-scope-与权威声明)
- [§2 内核基线锁定](#§2-内核基线锁定)
- [§3 物理形态定义](#§3-物理形态定义)
- [§4 改造方式与工程约束](#§4-改造方式与工程约束)
- [§5 微内核化体现（seL4 思想借鉴）](#§5-微内核化体现sel4-思想借鉴)
- [§6 关键设计决策](#§6-关键设计决策)
- [§7 技术选型四大支柱](#§7-技术选型四大支柱)
- [§8 编译与配置](#§8-编译与配置)
- [§9 相关文档](#§9-相关文档)
- [§10 变更历史](#§10-变更历史)
- [§11 极境内核具体能力详解](#§11-极境内核具体能力详解)

***

## §1 文档 scope 与权威声明

### 1.1 scope

本文档回答两个工程问题：

1. **ALK-6.6 长什么样子？**——物理形态定义（§3）
2. **如何对 Linux 6.6 进行 seL4 思想借鉴的微内核化改造？**——改造方式与工程约束（§4-§5）

本文档**不涵盖**以下内容（由对应权威文档负责）：

- 内核模块详细设计 → [20-modules/01-kernel.md](../20-modules/01-kernel.md)
- IRON-9 v3 四层模型 → [06-iron9-shared-model.md](06-iron9-shared-model.md)
- 目录结构详尽展开 → [07-directory-structure.md](07-directory-structure.md)
- Unify Design → [10-unify-design.md](10-unify-design.md)
- Capability 模型 → [110-security/03-capability-model.md](../110-security/03-capability-model.md)
- 纯 C LSM 设计 → [110-security/07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md)
- Micro-Supervisor → [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)

### 1.2 权威声明

> **SSoT 声明**：本文件是 **极境内核 ALK-6.6 总体架构** 的唯一权威源。ALK-6.6 的物理形态、改造方式、6 大改造点、4 大技术支柱、seL4 机制映射、关键设计决策（sched\_ext 不采用、纯 C LSM 采用、io\_uring 采用）均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 ALK-6.6 总体架构。

***

## §2 内核基线锁定

### 2.1 基线版本

| 维度      | 值                       | 来源                                             |
| ------- | ----------------------- | ---------------------------------------------- |
| 内核版本    | Linux 6.6.144 LTS       | `Makefile` VERSION=6 PATCHLEVEL=6 SUBLEVEL=144 |
| 版本名称    | Pinguïn Aangedreven     | `Makefile` NAME 字段                             |
| 基线性质    | vanilla 主线稳定版（非 OLK 变种） | ADR-013 / ADR-016                              |
| Fork 方式 | 完整 git clone，不修改主线源码    | OS-ARCH-001                                    |
| LTS 状态  | Long-Term Support 分支    | kernel.org LTS                                 |

### 2.2 基线选择理由

**决策**：ALK-6.6 基于 vanilla 6.6.144 LTS，不基于 OLK-6.6。

| 理由      | 说明                                          |
| ------- | ------------------------------------------- |
| 上游 SSoT | vanilla 主线是 Single Source of Truth，OLK 是衍生品 |
| 补丁透明    | vanilla 无闭源/定制补丁，所有修改可审计                    |
| 版本新鲜    | 6.6.144 含上游累计 bugfix，OLK-6.6 停在 6.6.0       |
| 形式化验证友好 | vanilla 行为可预测，无衍生补丁引入的语义不确定性                |
| 约束兼容    | vanilla 基线满足 OS-ARCH-001"不修改主线源码"约束         |

### 2.3 基线锁定约束

- **IRON-1**：禁止修改 Linux 6.6 主线源码（`arch/ block/ crypto/ drivers/ fs/ include/linux/ init/ kernel/ lib/ mm/ net/ security/selinux/ security/apparmor/ ...` 等全部上游目录）
- **IRON-2**：禁止绕开源码修改约束（不允许通过 patches/ quilt 等方式间接修改）
- **IRON-3**：所有 Airymax 增强代码必须在新增目录中（`security/airy/`、`kernel/kernel/superv/`、`include/uapi/linux/airymax/` 等）
- **ADR-013**：版本基线锁定 Linux 6.6 LTS，禁止升级到 6.7+ 或降级到 6.5-

***

## §3 物理形态定义

### 3.1 ALK-6.6 等式定义

```
ALK-6.6
  = Linux 6.6.144 LTS 完整 fork（不修改主线源码）
  + 新增 [SC] 共享契约层 10 个 UAPI 头文件
      物理位置：kernel/include/uapi/linux/airymax/
      内容：error.h / log_types.h / ipc.h / sched.h / memory_types.h /
            security_types.h / cognition_types.h / syscalls.h /
            uapi_compat.h / lsm_types.h
  + 新增 security/airy/ 纯 C LSM 模块（13 文件）
      物理位置：kernel/security/airy/
      内容：airy_lsm.c + airy_cap_init.c + airy_cap_array.c +
            airy_cap_derive.c + airy_cap_check.c + airy_cap_revoke.c +
            airy_cap_rotate.c + airy_superv_lsm.c + airy_ipc_freeze.c +
            airy_die_notify.c + airy_eventfd.c + airy_cap.h + Kbuild
  + 新增 kernel/kernel/superv/ Micro-Supervisor 内核模块（5 文件 + Kbuild）
      物理位置：kernel/kernel/superv/（位于 Linux 源码根的 kernel/ 子目录下）
      内容：airy_cap_check.c + airy_die_notify.c + airy_eventfd.c +
            airy_ipc_freeze.c + airy_superv_lsm.c + Kbuild
  + 新增 kernel/kernel/syscalls/ 4 个 Airymax syscall（编号 454-457）
      物理位置：kernel/kernel/syscalls/airy_syscalls.c +
                kernel/include/uapi/asm-generic/unistd.h 扩展
      内容：airy_sys_call + airy_sys_rovol_ctl +
            airy_sys_sched_ctl + airy_sys_clt_notify
      说明：避开 x32 历史占用区间 512-547，选用 454-457 空闲编号
  + 新增 kernel/kernel/corekern/ 微核心抽象层（10 子目录，32 文件）
      物理位置：kernel/kernel/corekern/
      内容：api/ sched/ ipc/ taskflow/ memory/ time/ object/
            locking/ irq/ bpf/（详见 §11.1）
  + 新增 kernel/kernel/log/ A-ULP 内核侧（3 文件 + Kbuild）
      物理位置：kernel/kernel/log/
      内容：airy_log_kern.c + airy_log_ring.c + airy_log_persist.c
  + 新增 kernel/kernel/ipc/ A-IPC 内核基础设施（2 文件 + Kbuild）
      物理位置：kernel/kernel/ipc/
      内容：airy_ipc_capability.c + airy_ipc_syscall.c
  + 新增 configs/ 预置配置（3 fragment）
      物理位置：kernel/configs/
      内容：defconfig + defconfig-agent + defconfig-embedded
  + 新增 config/Kconfig.alk ALK 顶层 Kconfig 入口
      物理位置：kernel/config/Kconfig.alk（顶层 Kconfig 通过 source 引用）
  + 新增 airy_defconfig fragment
      物理位置：kernel/arch/{x86,arm64}/configs/airy_defconfig
  + 新增非 [SC] 内部类型头文件
      物理位置：kernel/include/airymax/
      内容：build_types.h + kconfig_types.h
  + 复用 io_uring IORING_OP_URING_CMD（不新增 opcode）
  + 复用 SCHED_DEADLINE / SCHED_FIFO / EEVDF（不新增调度类）
  + 复用 MGLRU（mm/vmscan.c）/ userfaultfd / alloc_pages（不修改 mm/）
```

### 3.2 目录树（增量内容，严格按物理路径）

```
agentrt-linux/kernel/                              # repo 根（Linux 6.6.144 LTS fork）
│
├── 【Linux 6.6 主线源码（不修改，OS-ARCH-001 约束）】
│   ├── arch/           block/         certs/        crypto/
│   ├── drivers/        fs/            init/         ipc/
│   ├── kernel/         lib/           mm/           net/
│   ├── security/       sound/         tools/        usr/
│   ├── virt/           io_uring/      include/
│   ├── Kbuild  Kconfig  Makefile  COPYING  CREDITS  MAINTAINERS
│   └── ...
│
├── 【Airymax 增量内容（新增，不修改主线）】
│   │
│   ├── airy_core.c                                 # ALK 最小可启动模块入口
│   │                                               # （Phase 0 里程碑交付物）
│   │
│   ├── include/uapi/linux/airymax/                 # ★ [SC] 共享契约层（10 头文件）
│   │   ├── error.h                                 #   A-UEF 错误码 / Fault 码
│   │   ├── log_types.h                             #   A-ULP 日志类型
│   │   ├── ipc.h                                   #   A-IPC 128B 消息头 Layout C v4
│   │   ├── sched.h                                 #   A-ULS 调度类型
│   │   ├── memory_types.h                          #   MemoryRovol L1-L4 数据结构
│   │   ├── security_types.h                        #   Capability 41 ID + Badge 64-bit
│   │   ├── cognition_types.h                       #   CoreLoopThree 阶段枚举
│   │   ├── syscalls.h                              #   4 核心 + 20 预留 syscall
│   │   ├── uapi_compat.h                           #   UAPI 兼容层
│   │   └── lsm_types.h                             #   纯 C LSM 类型契约
│   │
│   ├── include/airymax/                            # 非 [SC] 内部类型
│   │   ├── build_types.h                           #   构建派生类型
│   │   └── kconfig_types.h                         #   Kconfig 派生类型
│   │
│   ├── security/airy/                              # ★ 纯 C LSM 模块（13 文件）
│   │   ├── Kbuild                                  #   构建配置
│   │   ├── airy_lsm.c                              #   DEFINE_LSM(airy) 注册 + 250 钩子
│   │   ├── airy_cap_init.c                         #   agent_caps[1024] 初始化
│   │   ├── airy_cap_array.c                        #   静态数组管理（v1.1 替代 radix_tree）
│   │   ├── airy_cap_derive.c                       #   seL4 CNode 7 派生操作
│   │   ├── airy_cap_check.c                        #   slowpath 校验
│   │   ├── airy_cap_revoke.c                       #   atomic_inc(&global_epoch) O(1) 撤销
│   │   ├── airy_cap_rotate.c                       #   per-Agent Epoch 轮换
│   │   ├── airy_superv_lsm.c                       #   Micro-Supervisor LSM 钩子注册
│   │   ├── airy_ipc_freeze.c                       #   IPC Ring 冻结
│   │   ├── airy_die_notify.c                       #   die_notifier 注册（priority INT_MAX）
│   │   ├── airy_eventfd.c                          #   eventfd 通知 Macro-Supervisor
│   │   └── airy_cap.h                              #   capability 内部头文件
│   │
│   ├── kernel/kernel/superv/                       # ★ Micro-Supervisor（5 .c + Kbuild）
│   │   ├── Kbuild                                  #   构建配置
│   │   ├── airy_cap_check.c                        #   fastpath C-S9 Badge 校验（~10ns）
│   │   ├── airy_die_notify.c                       #   die_notifier 注册
│   │   ├── airy_eventfd.c                          #   eventfd 通知 Macro-Supervisor
│   │   ├── airy_ipc_freeze.c                       #   IPC Ring 冻结
│   │   └── airy_superv_lsm.c                       #   Micro-Supervisor LSM 钩子
│   │
│   ├── kernel/kernel/syscalls/airy_syscalls.c      # ★ 4 新增 syscall（454-457）
│   │
│   ├── arch/{x86,arm64}/configs/airy_defconfig     # ★ defconfig fragment
│   │
│   ├── kernel/kernel/corekern/                     # ★ 微核心抽象层（10 子目录）
│   │   ├── Kbuild                                  #   顶层聚合（obj-y += api/ sched/ ...）
│   │   ├── api/                                    #   聚合初始化入口
│   │   │   ├── airy_kern_api.c                     #     late_initcall(airy_kern_api_init)
│   │   │   └── airy_kern_api.h
│   │   ├── sched/                                  #   sched_tac 4 策略 + 派发
│   │   │   ├── stc_policy.c                        #     STC_POLICY_REALTIME/INTERACTIVE/AGENT/BATCH
│   │   │   ├── stc_dispatch.c                      #     派发到 SCHED_DEADLINE/FIFO/EEVDF
│   │   │   ├── stc_mcs_map.c                       #     seL4 MCS 语义映射
│   │   │   └── stc_stats.c                         #     atomic_t 统计
│   │   ├── ipc/                                    #   IPC 内核原语（独立于 kernel/ipc/）
│   │   │   ├── airy_uring_cmd.c                    #     IORING_OP_URING_CMD 处理
│   │   │   ├── airy_ipc_ring.c                     #     Ring Buffer 管理
│   │   │   ├── airy_ipc_fastpath.c                 #     unlikely(READ_ONCE(ring->frozen)) 快速路径
│   │   │   ├── airy_ipc_zero_copy.c                #     vm_insert_pages 零拷贝
│   │   │   └── airy_ipc_internal.h                 #     共享 struct airy_ipc_ring
│   │   ├── taskflow/                               #   8 态生命周期状态机
│   │   │   ├── airy_task_lifecycle.c               #     INACTIVE→SPAWNING→READY→RUNNING→...→DEAD
│   │   │   └── airy_task_desc.c                    #     magic 0x41475453 校验
│   │   ├── memory/                                 #   L1-L4 分层内存
│   │   │   ├── airy_mem_alloc.c                    #     alloc_pages + mmap
│   │   │   └── airy_mem_tier.c                     #     AIRY_MEM_TIER_L1/L2/L3/L4
│   │   ├── time/                                   #   高精度时间戳
│   │   │   └── airy_time.c                         #     ktime_get_ns 包装
│   │   ├── object/                                 #   内核对象管理
│   │   │   └── airy_object.c                       #     DEFINE_SPINLOCK + LIST_HEAD
│   │   ├── locking/                                #   自旋锁包装
│   │   │   └── airy_locking.c                      #     spin_lock_nested 包装
│   │   ├── irq/                                    #   IRQ 注册/释放
│   │   │   └── airy_irq.c                          #     request_irq/free_irq
│   │   └── bpf/                                    #   BPF 兼容层（无 BPF 依赖）
│   │       ├── airy_struct_ops.c                   #     struct airy_struct_ops_value
│   │       └── airy_bpf_probe.c                    #     pr_info("%pV") 替代 trace_bpf_trace_printk
│   │
│   ├── kernel/kernel/log/                          # ★ A-ULP 内核侧（3 文件 + Kbuild）
│   │   ├── Kbuild
│   │   ├── airy_log_kern.c                         #   printk 8 级→A-ULP 5 级映射
│   │   ├── airy_log_ring.c                         #   alloc_pages + 128B 记录 Ring Buffer
│   │   └── airy_log_persist.c                      #   filp_open + kernel_write PMEM 持久化
│   │
│   ├── kernel/kernel/ipc/                          # ★ A-IPC 内核基础设施（2 文件 + Kbuild）
│   │   ├── Kbuild
│   │   ├── airy_ipc_capability.h                   #   airy_cap_badge_ok_fast() 3 次 READ_ONCE
│   │   ├── airy_ipc_capability.c                   #   atomic64_t airy_cap_global_epoch
│   │   └── airy_ipc_syscall.c                      #   airy_ipc_syscall_dispatch() 12 op 分发
│   │
│   ├── configs/                                    # ★ 预置配置（3 fragment）
│   │   ├── defconfig                               #   默认 fragment（5 个 CONFIG_AIRY_*）
│   │   ├── defconfig-agent                         #   Agent 优化（CGROUP/DRM_SCHED/CXL/THP）
│   │   └── defconfig-embedded                      #   最小化（禁用 AIRY_LOG/AIRY_IPC/BPF/CGROUPS）
│   │
│   ├── config/Kconfig.alk                          # ALK Kconfig 入口（顶层 Kconfig 通过 source 引用）
│   └── configs/airy_defconfig                      # 顶层 defconfig 副本
│
└── 【Airymax 文档元数据】
    ├── AIRYMAX_KERNEL_FILES.txt
    ├── LICENSE  NOTICE
    └── README_ALK.md
```

### 3.3 与 vanilla Linux 6.6 的差异矩阵

| #  | 维度                                            | vanilla 6.6.144 | ALK-6.6                      |      修改主线？      |
| -- | --------------------------------------------- | --------------- | ---------------------------- | :-------------: |
| 1  | `include/uapi/linux/airymax/`                 | 不存在             | 新增 10 头文件                    |        否        |
| 2  | `include/airymax/`                            | 不存在             | 新增 2 内部类型                    |        否        |
| 3  | `security/airy/`                              | 不存在             | 新增 13 文件纯 C LSM              |        否        |
| 4  | `kernel/kernel/superv/`                       | 不存在             | 新增 5+Kbuild Micro-Supervisor |        否        |
| 5  | `kernel/kernel/syscalls/airy_syscalls.c`      | 不存在             | 新增 4 syscall（编号 454-457）    |        否        |
| 6  | `kernel/kernel/corekern/`                     | 不存在             | 新增 10 子目录 32 文件微核心抽象层     |        否        |
| 7  | `kernel/kernel/log/`                          | 不存在             | 新增 3 文件 A-ULP 内核侧           |        否        |
| 8  | `kernel/kernel/ipc/`                          | 不存在             | 新增 2 文件 A-IPC 内核基础设施         |        否        |
| 9  | `configs/`                                    | 不存在             | 新增 3 defconfig fragment       |        否        |
| 10 | `arch/*/configs/airy_defconfig`               | 不存在             | 新增 defconfig fragment         |        否        |
| 11 | `include/uapi/asm-generic/unistd.h`           | vanilla 原样      | 扩展 4 syscall 编号 454-457      | **是**（扩展，不修改已有） |
| 12 | `kernel/`、`mm/`、`net/`、`drivers/` 等           | vanilla 原样      | 不修改                          |        否        |
| 13 | `security/selinux/`、`apparmor/`、`landlock/` 等 | vanilla 原样      | 不修改                          |        否        |
| 14 | `io_uring/`                                   | vanilla 原样      | 复用 IORING\_OP\_URING\_CMD    |        否        |

**唯一的主线文件扩展**：`include/uapi/asm-generic/unistd.h` 新增 4 个 syscall 编号条目（454-457，避开 x32 历史占用区间 512-547），不修改已有条目。这是 Linux 内核标准的 syscall 注册方式，符合上游规范。

***

## §4 改造方式与工程约束

### 4.1 核心原则

**OS-ARCH-001 硬约束**：不修改任何 Linux 6.6 主线源码，仅增量添加。

ALK-6.6 不是"魔改 Linux 内核"，而是"在 Linux 6.6 LTS 之上增量添加 agentrt-linux 所需的内核能力"。所有 Airymax 代码必须在新增目录中，与主线源码物理隔离。

### 4.2 10 大改造点

| #  | 改造点              | 改造方式                                            | 物理位置                                                                              |        修改主线？        | 依赖      |
| -- | ---------------- | ----------------------------------------------- | --------------------------------------------------------------------------------- | :-----------------: | ------- |
| 1  | \[SC] 共享契约层      | 新增 10 个 UAPI 头文件                                | `include/uapi/linux/airymax/`                                                     |          否          | 无       |
| 2  | 纯 C LSM 模块       | 新增 `airy_lsm` 模块（编译为 .ko 或内置）                   | `security/airy/`                                                                  |          否          | 改造点 1   |
| 3  | Micro-Supervisor | 新增内核模块（冷酷执法）                                    | `kernel/kernel/superv/`                                                           |          否          | 改造点 2   |
| 4  | Capability 系统    | `agent_caps[1024]` 静态数组管理（集成在 `security/airy/`） | `security/airy/`                                                                  |          否          | 改造点 2   |
| 5  | 4 个新增 syscall    | 新增 syscall 入口（454-457，避开 x32 历史区间 512-547）       | `kernel/kernel/syscalls/airy_syscalls.c` + `include/uapi/asm-generic/unistd.h` 扩展 | **是**（仅扩展 unistd.h） | 改造点 1   |
| 6  | corekern 微核心抽象层  | 新增 10 子目录 32 文件                                 | `kernel/kernel/corekern/`                                                         |          否          | 改造点 1   |
| 7  | A-ULP 内核侧        | 新增 3 文件 + Kbuild                                | `kernel/kernel/log/`                                                              |          否          | 改造点 1   |
| 8  | A-IPC 内核基础设施      | 新增 2 文件 + Kbuild                                | `kernel/kernel/ipc/`                                                              |          否          | 改造点 1   |
| 9  | airy\_defconfig  | 新增 3 个 defconfig fragment                        | `configs/{defconfig,defconfig-agent,defconfig-embedded}` + `arch/{x86,arm64}/configs/airy_defconfig` | 否 | 改造点 1-8 |
| 10 | ALK Kconfig 入口   | 顶层 Kconfig 通过 source 引用 `config/Kconfig.alk`     | `config/Kconfig.alk` + `Kconfig` 第 34 行 `source` 引用                              |          否          | 改造点 1-8 |

### 4.3 工程约束矩阵

| 约束 ID       | 约束内容               | 强制等级 | 验证方法                                          |
| ----------- | ------------------ | ---- | --------------------------------------------- |
| OS-ARCH-001 | 不修改 Linux 6.6 主线源码 | 强制   | `git diff -- mainline/` 必须为空                  |
| OS-ARCH-002 | 所有 Airymax 代码在新增目录 | 强制   | 目录白名单审计                                       |
| OS-ARCH-003 | \[SC] 头文件单一物理宿主    | 强制   | `find . -name 'ipc.h' -path '*/airymax/*'` 唯一 |
| IRON-1      | 禁止新特性（0.1.1 奠基版本）  | 强制   | TSC 评审                                        |
| IRON-2      | 禁止桩函数 / 桩文件        | 强制   | 静态分析                                          |
| IRON-3      | 禁止绕开功能限制的编译集成      | 强制   | CI 构建测试                                       |
| IRON-4      | 禁止简化功能             | 强制   | 代码审查                                          |
| IRON-5      | 禁止未实现的功能设计         | 强制   | 设计-实现一致性审计                                    |

***

## §5 微内核化体现（seL4 思想借鉴）

### 5.1 seL4 机制映射

依据 ADR-014（seL4 唯一思想来源），ALK-6.6 借鉴 seL4 的以下机制：

| seL4 机制          | seL4 源码位置                   | ALK-6.6 实现                                 | ALK-6.6 物理载体                                 | 借鉴程度 |
| ---------------- | --------------------------- | ------------------------------------------ | -------------------------------------------- | ---- |
| CNode Capability | `src/object/cnode.c`        | `agent_caps[1024]` 静态数组 + Badge 64-bit     | `security/airy/`                             | 机制借鉴 |
| CNode 派生操作（7 种）  | `src/object/cnode.c`        | Copy/Mint/Move/Mutate/Revoke/Delete/Rotate | `security/airy/airy_cap_derive.c`            | 完整借鉴 |
| handleFault()    | `src/api/faults.c`          | Micro-Supervisor 冷酷执法                      | `kernel/kernel/superv/`                      | 模式借鉴 |
| Endpoint IPC     | `src/object/endpoint.c`     | io\_uring + 128B 消息头 Layout C v4           | 复用 `io_uring/` + \[SC] ipc.h                 | 语义借鉴 |
| Notification 位图  | `src/object/notification.c` | eventfd + 位图聚合                             | `kernel/kernel/superv/airy_eventfd.c`        | 语义借鉴 |
| MCS 调度上下文        | `src/object/schedcontext.c` | sched\_tac（SCHED\_DEADLINE 映射）             | 复用 `kernel/sched/deadline.c`                 | 映射借鉴 |
| MCS Refill 循环缓冲  | `src/object/schedcontext.c` | SCHED\_DEADLINE `runtime`/`period` 补充机制    | 复用 `kernel/sched/deadline.c`                 | 语义借鉴 |
| Double Fault 处理  | `src/api/faults.c`          | Macro-Supervisor 用户态二级裁决                   | `services/daemons/macro_d/`                  | 模式借鉴 |
| 服务用户态化           | seL4 用户态服务                  | 12 daemon 用户态运行                            | `services/` 子仓                               | 架构借鉴 |
| 机制/策略分离          | seL4 内核机制 + 用户态策略           | 内核冷酷执法 + 用户态温情裁决                           | `kernel/kernel/superv/` + `services/macro_d` | 哲学借鉴 |
| fastpath         | `src/fastpath`              | C-S0\~C-S12 校验链 + C-S9 Badge 内联            | fastpath C-S9 `airy_cap_badge_ok()`          | 模式借鉴 |

### 5.2 Liedtke minimality principle 落地

**Liedtke 原则**（ES-SEL4-01）：只有当某个概念移出微内核会导致系统无法实现所需功能时，才容忍它留在微内核内。

**ALK-6.6 诠释**：ALK-6.6 不是从零开发微内核（ADR-012），而是基于 Linux 6.6 内核进行微内核化改造。因此"极简"不意味着将 Linux 缩减到 1.44 万行，而是遵循以下体量控制策略：

| 维度            | seL4 基线                                       | ALK-6.6 预算                 | 控制策略                                  |
| ------------- | --------------------------------------------- | -------------------------- | ------------------------------------- |
| 内核核心代码        | \~1.44 万行 C                                   | 5-10 万行（含 agent 调度原语）      | 每新增子系统需论证"无法在用户态安全实现"                 |
| 微内核化改造补丁      | —                                             | 控制在 2 万行以内                 | VFS / 网络栈 / 驱动用户态化补丁                  |
| \[SC] 共享契约层   | —                                             | 10 个头文件（IRON-9 v3）         | 单一物理宿主，禁止重复定义                         |
| Syscall 数量    | 7（非 MCS）/ 11（MCS）                             | 4 核心 + 20 预留 = 24 槽位       | Capability Folding 后 IPC 数据面零 syscall |
| Capability 操作 | 7（Copy/Mint/Move/Mutate/Revoke/Delete/Rotate） | 7（与 seL4 完全对齐）             | 完整借鉴 seL4 CNode                       |
| LSM 钩子覆盖      | —                                             | 250（对齐 OLK-6.6 纯 C LSM 模式） | 移除 BPF 专属钩子                           |

### 5.3 体量审计机制

每版本发布前执行代码体量审计，消除"蠕变增胖"（kernel bloat）。新增子系统需向 TSC 提交"无法在用户态安全实现"论证报告。

***

## §6 关键设计决策

### 6.1 决策 1：为什么使用 sched\_tac 而非 sched\_ext？

**背景**：OLK-6.6 已 backport sched\_ext（`kernel/sched/ext.c`，215KB，`SCHED_EXT = 7`，`CONFIG_SCHED_CLASS_EXT`）。vanilla 6.6.144 主线不含 sched\_ext（6.12 才合入主线）。

**决策**：**不采用 sched\_ext，采用 sched\_tac**（SCHED\_DEADLINE/SCHED\_FIFO/EEVDF + cgroup cpuset 隔离 + seL4 MCS 语义映射）。

**理由**（4 条，按重要性排序）：

| # | 理由                      | 证据                                                                                                       |
| - | ----------------------- | -------------------------------------------------------------------------------------------------------- |
| 1 | **BPF 依赖与纯 C LSM 原则冲突** | sched\_ext 依赖 `BPF_SYSCALL && BPF_JIT && DEBUG_INFO_BTF`（`kernel/Kconfig.preempt:138`），与 H5 纯 C LSM 原则冲突 |
| 2 | **x86 默认禁用**            | `include/linux/sched.h:1663-1667` 注释明确："SCHED\_EXT is disabled by default on x86"                        |
| 3 | **形式化验证可行性**            | BPF verifier 在调度路径的语义不确定性影响形式化验证可行性                                                                      |
| 4 | **体系一致性**               | sched\_ext 的 BPF struct\_ops 模型与 agentrt-linux 纯 C 体系不一致                                                 |

**详见**：[30-interfaces/10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) §1.3

### 6.2 决策 2：为什么使用纯 C LSM 而非 BPF LSM？

**决策**：采用纯 C LSM 模块（`security/airy/`），不使用 BPF LSM。

**理由**：

1. **对齐 openEuler 纯 C 模式**：OLK-6.6 的 SELinux/AppArmor/Landlock/Tomoyo 全部纯 C 实现，不使用 BPF LSM
2. **Landlock 源码验证**：OLK-6.6 的 `security/landlock/` 全部纯 C，无 BPF 依赖（使用 `landlock_create_ruleset` / `landlock_add_rule` / `landlock_restrict_self` 三个专用 syscall，不走 BPF 路径）
3. **形式化验证友好**：纯 C 代码可被静态分析与形式化验证工具处理，BPF verifier 的语义不确定性阻碍验证
4. **fastpath 性能**：纯 C fastpath \~160ns，无 BPF 间接调用开销

**详见**：[110-security/07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md) §1

### 6.3 决策 3：为什么使用 io\_uring 而非新增 IPC syscall？

**决策**：复用 io\_uring `IORING_OP_URING_CMD` 承载 IPC 数据面，不新增 IPC syscall。

**理由**：

1. **零 syscall**：io\_uring 提交队列用户态直接写，内核态消费，无 syscall 开销
2. **零拷贝**：registered buffer + mmap 实现零拷贝
3. **不新增 opcode**：复用 `IORING_OP_URING_CMD`，通过 `cmd_op` 子命令区分操作
4. **异步 I/O 基础设施**：io\_uring 已是 Linux 6.6 标准 async I/O 接口

**详见**：[30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md) §4.4

### 6.4 决策 4：为什么 Capability 系统集成在 security/airy/ 而非独立 security/capability/？

**决策**：`agent_caps[1024]` 静态数组管理集成在 `security/airy/` 内，不创建独立 `security/capability/` 目录。

**理由**：

1. **LSM 钩子与 Capability 校验紧耦合**：fastpath C-S9 Badge 校验是 LSM 钩子的一部分，拆分会导致跨目录调用
2. **单一模块编译单元**：`airy_lsm` 作为一个 LSM 模块统一编译，Capability 是其子功能
3. **seL4 CNode 哲学**：seL4 的 CNode 也是内核对象的一部分，不独立于内核安全机制

**注**：本决策修正了早期方案中"创建 `security/capability/` 独立目录"的设计。

***

## §7 技术选型四大支柱

| 支柱  | 选型                                            | 复用/新增 | 理由                          | 物理载体               |
| --- | --------------------------------------------- | ----- | --------------------------- | ------------------ |
| 调度  | sched\_tac（SCHED\_DEADLINE/SCHED\_FIFO/EEVDF） | 复用    | 不新增调度类，不依赖 BPF              | 复用 `kernel/sched/` |
| IPC | io\_uring `IORING_OP_URING_CMD`               | 复用    | 不新增 opcode，零拷贝，零 syscall    | 复用 `io_uring/`     |
| 安全  | 纯 C LSM（airy\_lsm）                            | 新增    | 对齐 openEuler 纯 C 模式，形式化验证友好 | `security/airy/`   |
| 内存  | alloc\_pages + mmap                           | 复用    | x86\_64 默认缓存一致，无需 DMA 一致性内存 | 复用 `mm/`           |

***

## §8 编译与配置

### 8.1 airy\_defconfig

```kconfig
# ===== Airymax ALK-6.6 defconfig fragment =====
# 物理位置：arch/{x86,arm64}/configs/airy_defconfig

# --- 纯 C LSM 模块 ---
CONFIG_SECURITY_AIRY=y
# CONFIG_BPF_LSM is not set

# --- io_uring ---
CONFIG_IO_URING=y

# --- 调度（sched_tac）---
CONFIG_SCHED_DEADLINE=y
# CONFIG_SCHED_CLASS_EXT is not set

# --- 内存（MGLRU + userfaultfd）---
CONFIG_LRU_GEN=y
CONFIG_LRU_GEN_ENABLED=y
CONFIG_USERFAULTFD=y
CONFIG_TRANSPARENT_HUGEPAGE=y

# --- Airymax syscall ---
CONFIG_AIRY_SYSCALL=y
```

### 8.2 Kconfig

| 文件                      | 作用                                            |
| ----------------------- | --------------------------------------------- |
| `security/airy/Kconfig` | 定义 `CONFIG_SECURITY_AIRY` 纯 C LSM 配置          |
| `security/Kconfig`      | `CONFIG_LSM` 默认值添加 `airy`（位于 `capability` 之后） |
| `config/Kconfig.alk`    | ALK 顶层 Kconfig 入口，聚合所有 Airymax 配置             |

### 8.3 构建命令

```bash
# 构建 ALK-6.6（x86_64）
make ARCH=x86_64 airy_defconfig
make ARCH=x86_64 -j$(nproc)

# 构建 ALK-6.6（arm64）
make ARCH=arm64 airy_defconfig
make ARCH=arm64 -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu-
```

***

## §9 相关文档

### 9.1 上游权威文档

| 文档                                                     | 标题                | 关系            |
| ------------------------------------------------------ | ----------------- | ------------- |
| [06-iron9-shared-model.md](06-iron9-shared-model.md)   | IRON-9 v3 四层共享模型  | \[SC] 共享契约层定义 |
| [07-directory-structure.md](07-directory-structure.md) | 目录结构 v2.0         | 详尽目录展开        |
| [10-unify-design.md](10-unify-design.md)               | Unify Design v1.1 | 统一设计总纲        |

### 9.2 模块详细设计文档

| 文档                                                                                      | 标题               | 关系      |
| --------------------------------------------------------------------------------------- | ---------------- | ------- |
| [20-modules/01-kernel.md](../20-modules/01-kernel.md)                                   | 内核模块详细设计         | 内核子系统展开 |
| [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md) | Micro-Supervisor | 冷酷执法机制  |
| [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md)   | Macro-Supervisor | 温情裁决机制  |

### 9.3 安全子系统工程文档

| 文档                                                                                | 标题            | 关系         |
| --------------------------------------------------------------------------------- | ------------- | ---------- |
| [110-security/03-capability-model.md](../110-security/03-capability-model.md)     | Capability 模型 | 7 派生操作定义   |
| [110-security/07-airy-lsm-design.md](../110-security/07-airy-lsm-design.md)       | 纯 C LSM 设计    | 250 钩子覆盖   |
| [110-security/06-io-uring-hardening.md](../110-security/06-io-uring-hardening.md) | io\_uring 加固  | opcode 白名单 |

### 9.4 接口契约文档

| 文档                                                                                  | 标题         | 关系            |
| ----------------------------------------------------------------------------------- | ---------- | ------------- |
| [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md)                     | Syscall 契约 | 4 核心 syscall  |
| [30-interfaces/02-ipc-protocol.md](../30-interfaces/02-ipc-protocol.md)             | IPC 协议     | 128B 消息头      |
| [30-interfaces/10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md) | 调度扩展       | sched\_tac 定义 |

***

## §10 变更历史

| 版本   | 日期         | 变更内容                                                            | 作者           |
| ---- | ---------- | --------------------------------------------------------------- | ------------ |
| v1.0 | 2026-07-20 | 初始版本：ALK-6.6 总体架构权威定义；物理形态、改造方式、6 大改造点、4 大技术支柱、seL4 机制映射、关键设计决策 | SPHARX 工程标准组 |
| v1.1 | 2026-07-20 | 精简：移除非奠基内容（OLK-6.6 关系论述、工程基线审计），章节由 12 节精简至 10 节                     | SPHARX 工程标准组 |
| v1.2 | 2026-07-21 | 全面更新：syscall 编号 512-515→454-457（避开 x32 历史区间）；superv 路径修正（kernel/kernel/superv/）；改造点扩展为 10 项；新增 §11 极境内核具体能力详解（corekern/log/ipc/configs 全景） | SPHARX 工程标准组 |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） | — |

***

## §11 极境内核具体能力详解

> **本节定位**：v1.2 新增。本节展开 ALK-6.6 极境内核的**具体能力清单**，按物理载体（corekern/log/ipc/configs）逐项描述每个子系统的实现要点、关键函数、设计权衡。所有描述与 `agentrt-linux/kernel/` 实际源码一一对应，是设计-实现一致性审计的依据。

### 11.1 corekern 微核心抽象层（10 子目录，32 文件）

`kernel/kernel/corekern/` 是 ALK-6.6 内核增强的核心载体，通过 `obj-y` 编译进 vmlinux（由 `CONFIG_AIRY_COREKERN=y` 控制）。10 个子目录按"机制/策略分离"原则组织，机制由 corekern 提供，策略由用户态 daemon 决定。

#### 11.1.1 api/ — 聚合初始化入口

| 文件                     | 能力                                                          |
| ---------------------- | ----------------------------------------------------------- |
| `airy_kern_api.c`      | `late_initcall(airy_kern_api_init)` 内核启动后期初始化入口，打印版本 banner |
| `airy_kern_api.h`      | 公开 `airy_kern_api_init()` 原型                                 |

**关键函数**：`airy_kern_api_init()` 依次调用 `airy_locking_init()`（最先初始化，其他子系统依赖自旋锁包装），打印 `airy corekern: api ready` 完成启动公告。

#### 11.1.2 sched/ — sched_tac 4 策略派发

| 文件                   | 能力                                                              |
| -------------------- | --------------------------------------------------------------- |
| `stc_policy.c`       | 定义 4 策略枚举 `STC_POLICY_REALTIME/INTERACTIVE/AGENT/BATCH`，与 [SC] sched.h 的 `AIRY_SCHED_POLICY_*` 宏值一一对应 |
| `stc_dispatch.c`     | 策略派发：`stc_realtime→SCHED_DEADLINE`、`stc_interactive→SCHED_FIFO`、`stc_agent→EEVDF`、`stc_batch→SCHED_BATCH` |
| `stc_mcs_map.c`      | seL4 MCS（Mixed-Criticality System）调度上下文语义映射，`runtime`/`period`/`deadline` 三元组对应 SCHED_DEADLINE 参数 |
| `stc_stats.c`        | `atomic_t` 统计各策略的派发次数、切换次数、错过截止时间次数                              |

**关键决策**：不采用 sched_ext（理由详见 §6.1），不新增调度类，全部复用 Linux 6.6 原生 SCHED_DEADLINE/SCHED_FIFO/EEVDF/SCHED_BATCH。

#### 11.1.3 ipc/ — IPC 内核原语（独立于 kernel/ipc/）

| 文件                       | 能力                                                              |
| ------------------------ | --------------------------------------------------------------- |
| `airy_uring_cmd.c`       | `IORING_OP_URING_CMD` 处理入口，按 `cmd_op` 子命令分发到 fastpath/zero-copy 路径 |
| `airy_ipc_ring.c`        | Ring Buffer 管理：`airy_ipc_ring_post()` 写入消息，`airy_ipc_ring_pop()` 读取消息 |
| `airy_ipc_fastpath.c`    | 热路径发送：`unlikely(READ_ONCE(ring->frozen))` 单次检查后委托到 `airy_ipc_ring_post()` |
| `airy_ipc_zero_copy.c`   | 零拷贝路径：`airy_ipc_zero_copy_register()` 注册页对齐缓冲区，`airy_ipc_zero_copy_map()` 通过 `vm_insert_pages()` 映射到用户 VMA |
| `airy_ipc_internal.h`    | 共享内部头：定义 `struct airy_ipc_ring`，含 `frozen` 标志、生产者/消费者索引       |

**关键约束**：禁用 DMA 一致性内存（OS-ARCH-001），仅使用 `alloc_pages + mmap`；零拷贝要求 base/len 必须 `PAGE_ALIGNED`。

#### 11.1.4 taskflow/ — 8 态生命周期状态机

| 文件                       | 能力                                                              |
| ------------------------ | --------------------------------------------------------------- |
| `airy_task_lifecycle.c`  | 定义 8 态枚举：`INACTIVE→SPAWNING→READY→RUNNING→BLOCKED→SUSPENDED→TERMINATING→DEAD`；`airy_task_state_transition()` 强制合法转换（如 DEAD 终态不可转出） |
| `airy_task_desc.c`       | 任务描述符管理：magic `0x41475453`（'AGTS'）校验，与 IPC magic `0x41524531`（'ARE1'）区分 |

**关键决策**：任务描述符独立 magic，避免与 IPC 消息头混淆（IRON-9 v3 同源但独立原则）。

#### 11.1.5 memory/ — L1-L4 分层内存

| 文件                  | 能力                                                              |
| ------------------- | --------------------------------------------------------------- |
| `airy_mem_alloc.c`  | `alloc_pages + mmap` 包装：按 tier 选择 alloc_flags，映射到用户 VMA       |
| `airy_mem_tier.c`   | 4 tier 枚举：`AIRY_MEM_TIER_L1`（HBM/DDR hot）、`L2`（DDR warm）、`L3`（CXL/NVMe cold）、`L4`（PMEM persistent）；`airy_mem_tier_of(gfp)` 按 `AIRY_GFP_HOT/WARM/COLD/PMEM` 低 4 位分类 |

**关键设计**：per-tier size shift 表（L1=0/L2=3/L3=6/L4=9）允许批量化分配，L4 一次分配 512 页（2MB）以减少 PMEM 写放大。`GFP_KERNEL` 默认归入 L2 warm tier。

#### 11.1.6 time/ — 高精度时间戳

| 文件             | 能力                                  |
| -------------- | ----------------------------------- |
| `airy_time.c`  | `ktime_get_ns()` 包装，提供单调时钟纳秒级时间戳    |

#### 11.1.7 object/ — 内核对象管理

| 文件                | 能力                                                          |
| ----------------- | ----------------------------------------------------------- |
| `airy_object.c`   | `DEFINE_SPINLOCK + LIST_HEAD` 维护内核对象链表，提供 `airy_object_add/del/find` 操作 |

#### 11.1.8 locking/ — 自旋锁包装

| 文件                  | 能力                                                          |
| ------------------- | ----------------------------------------------------------- |
| `airy_locking.c`    | `spin_lock_nested()` 包装（避免 lockdep 误报），`airy_locking_init()` 初始化（最先被 api/ 调用） |

#### 11.1.9 irq/ — IRQ 注册/释放

| 文件             | 能力                                                          |
| -------------- | ----------------------------------------------------------- |
| `airy_irq.c`   | `request_irq()`/`free_irq()` 包装，提供 Airymax 设备的 IRQ 注册接口     |

#### 11.1.10 bpf/ — BPF 兼容层（无 BPF 依赖）

| 文件                       | 能力                                                              |
| ------------------------ | --------------------------------------------------------------- |
| `airy_struct_ops.c`      | 定义 `struct airy_struct_ops_value`，为 BPF struct_ops 提供 C 结构体对等物（不依赖 BPF） |
| `airy_bpf_probe.c`       | `pr_info("%pV")` 替代 `trace_bpf_trace_printk()`，避免 BPF verifier 语义不确定性 |

**关键决策**：H5 纯 C LSM 原则禁止 BPF 依赖，但保留兼容层用于未来可选的 BPF 探针（非默认路径）。

### 11.2 log — A-ULP 内核侧（3 文件 + Kbuild）

`kernel/kernel/log/` 实现 Airymax Unified Logging Plane（A-ULP）的内核侧桥接。

| 文件                    | 能力                                                              |
| --------------------- | --------------------------------------------------------------- |
| `airy_log_kern.c`     | printk 8 级 → A-ULP 5 级映射表：`LOGLEVEL_EMERG/ALERT/CRIT→FATAL`、`ERR→ERROR`、`WARNING→WARN`、`NOTICE/INFO→INFO`、`DEBUG→DEBUG`；`airy_log_emit()` 通过 `vprintk_emit()` 转发到标准 printk ring；`late_initcall` 公告就绪 |
| `airy_log_ring.c`     | `alloc_pages` 分配 128B 记录的 SPSC（单生产者单消费者）Ring Buffer            |
| `airy_log_persist.c`  | `filp_open + kernel_write` 将日志记录写入 PMEM 路径（默认 `/pmem/airy/log`）实现持久化 |

**关键设计**：A-ULP 5 级与 syslog 8 级的映射是多对一关系（3 个 printk 级别映射到 1 个 A-ULP FATAL），保留 A-ULP 简化的同时不丢失 printk 原生语义。

### 11.3 ipc — A-IPC 内核基础设施（2 文件 + Kbuild）

`kernel/kernel/ipc/` 提供 A-IPC 的内核侧 Capability Folding 后端和 syscall 派发辅助。

| 文件                          | 能力                                                              |
| --------------------------- | --------------------------------------------------------------- |
| `airy_ipc_capability.h`     | `airy_cap_badge_ok_fast()` 内联函数：3 次 `READ_ONCE` 读取 cacheline-aligned slot 的 epoch/randtag/perms |
| `airy_ipc_capability.c`     | `airy_cap_badge_verify()` slowpath：校验 badge epoch==slot epoch==global epoch、randtag 匹配、perms 包含期望权限；`airy_cap_epoch_bump()` O(1) 撤销：`atomic64_inc_return(&airy_cap_global_epoch)` |
| `airy_ipc_syscall.c`        | `airy_ipc_syscall_dispatch()` 12 op switch 分发辅助函数（不重复定义 SYSCALL_DEFINE，实际 syscall 已在 `kernel/syscalls/airy_syscalls.c` 中定义） |

**关键设计**：

1. **Capability Folding 双层校验**：fastpath 内联 `airy_cap_badge_ok_fast()` 仅 3 次 READ_ONCE（~10ns），slowpath `airy_cap_badge_verify()` 增加 global epoch 校验捕获 O(1) 撤销
2. **Badge 编码**：`Badge = Epoch<<48 | RandomTag<<16 | Perms`，48 位 Epoch、32 位 RandomTag、16 位 Perms
3. **O(1) 撤销**：单次 `atomic64_inc_return` 使所有旧 epoch 的 badge 失效，无需遍历 slot 表
4. **`__cacheline_aligned`**：slot 表强制缓存行对齐，避免 fastpath 读取时的 false sharing
5. **避免 syscall 重复定义**：4 个 SYSCALL_DEFINE 已在 `kernel/syscalls/airy_syscalls.c` 中定义（编号 454-457），本文件仅提供 dispatch 辅助

### 11.4 configs — 预置配置（3 fragment）

`kernel/configs/` 提供 3 个 defconfig fragment，覆盖不同部署场景。

| 文件                  | 适用场景         | 关键配置项                                                       |
| ------------------- | ------------ | ----------------------------------------------------------- |
| `defconfig`         | 默认基线         | `CONFIG_SECURITY_AIRY=y` + 5 个 `CONFIG_AIRY_*` + `IO_URING` + `BPF` |
| `defconfig-agent`   | Agent 优化部署   | 默认基线 + `CGROUPS` + `DRM_SCHED` + `CXL` + `TRANSPARENT_HUGEPAGE` |
| `defconfig-embedded` | 嵌入式最小化 | 默认基线 - `AIRY_LOG` - `AIRY_IPC` - `BPF` - `CGROUPS`（节省 ROM/RAM） |

**使用方式**：

```bash
make ARCH=x86_64 defconfig
./scripts/kconfig/merge_config.sh -m .config configs/defconfig-agent
make ARCH=x86_64 olddefconfig
make ARCH=x86_64 -j$(nproc)
```

### 11.5 能力清单汇总

| #  | 子系统              | 物理位置                          | 文件数 | 关键能力                                                      |
| -- | ---------------- | ----------------------------- | --- | --------------------------------------------------------- |
| 1  | corekern/api     | `kernel/kernel/corekern/api/` | 2   | late_initcall 聚合初始化                                       |
| 2  | corekern/sched   | `kernel/kernel/corekern/sched/` | 4   | sched_tac 4 策略 + seL4 MCS 映射                              |
| 3  | corekern/ipc     | `kernel/kernel/corekern/ipc/` | 5   | uring_cmd + Ring Buffer + fastpath + zero-copy             |
| 4  | corekern/taskflow | `kernel/kernel/corekern/taskflow/` | 2   | 8 态生命周期 + magic 0x41475453 校验                              |
| 5  | corekern/memory  | `kernel/kernel/corekern/memory/` | 2   | L1-L4 分层 + per-tier size shift                            |
| 6  | corekern/time    | `kernel/kernel/corekern/time/` | 1   | ktime_get_ns 高精度时间戳                                       |
| 7  | corekern/object  | `kernel/kernel/corekern/object/` | 1   | SPINLOCK + LIST_HEAD 内核对象管理                                |
| 8  | corekern/locking | `kernel/kernel/corekern/locking/` | 1   | spin_lock_nested 包装                                       |
| 9  | corekern/irq     | `kernel/kernel/corekern/irq/` | 1   | request_irq/free_irq 包装                                   |
| 10 | corekern/bpf     | `kernel/kernel/corekern/bpf/` | 2   | struct_ops C 结构体 + pr_info("%pV") 替代 BPF trace            |
| 11 | log              | `kernel/kernel/log/`          | 3   | printk 8→5 级映射 + 128B SPSC ring + PMEM 持久化                |
| 12 | ipc              | `kernel/kernel/ipc/`          | 2   | Capability Folding slowpath + 12 op dispatch              |
| 13 | configs          | `kernel/configs/`             | 3   | defconfig / defconfig-agent / defconfig-embedded          |
| **合计** |              |                               | **29** |                                                          |

**注**：本表不含 Kbuild 文件（每个子目录另含 1 个 Kbuild，共 13 个 Kbuild，总文件数 29+13=42，与 §3.1 等式定义一致）。

### 11.6 性能关键路径

#### 11.6.1 IPC fastpath（~10ns）

```
用户态 airy_sys_call(454)
  → kernel/syscalls/airy_syscalls.c: SYSCALL_DEFINE2(airy_sys_call)
    → airy_ipc_syscall_dispatch()              [kernel/ipc/airy_ipc_syscall.c]
      → airy_cap_badge_ok_fast()               [airy_ipc_capability.h, 内联]
          3 次 READ_ONCE(slot.epoch/randtag/perms)
      → airy_ipc_fastpath_send()               [corekern/ipc/airy_ipc_fastpath.c]
          unlikely(READ_ONCE(ring->frozen))    [单次分支预测]
          → airy_ipc_ring_post()               [corekern/ipc/airy_ipc_ring.c]
```

**性能保证**：

1. fastpath 内联：3 次 READ_ONCE（cacheline-aligned slot，无 false sharing）
2. 单次 unlikely 分支：frozen 标志位检查
3. 无锁 SPSC Ring：生产者/消费者索引无需原子操作（单线程写入）

#### 11.6.2 Capability O(1) 撤销

```
airy_cap_epoch_bump()                          [kernel/ipc/airy_ipc_capability.c]
  atomic64_inc_return(&airy_cap_global_epoch)  [单次原子操作]
  → 所有旧 epoch 的 badge 在下一次 airy_cap_badge_verify() 时失效
```

**复杂度**：O(1)（不依赖 slot 表大小，1024 个 agent 的撤销与 1 个 agent 的撤销耗时相同）。

***

© 2025-2026 SPHARX Ltd. All Rights Reserved. | 极境内核 ALK-6.6 总体架构规范 | v1.0.1 | 2026-07-21
