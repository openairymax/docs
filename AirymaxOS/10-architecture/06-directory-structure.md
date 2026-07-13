Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）系统性目录结构设计

> **文档定位**：agentrt-linux（AirymaxOS）源码目录结构 SSoT\
> **版本**：0.1.1（文档体系完成）/ 1.0.1 M1（代码落地）\
> **最后更新**：2026-07-13\
> **父文档**：[10-architecture/README.md](README.md)\
> **来源**：Wave 2 v2 Phase C §7 系统性目录结构设计建议（基于 B1 OLK-6.6 深读 + B2 seL4 深读 + 8 子仓架构）\
> **铁律依据**：IRON-9 v2 三层模型 + ES-OLK-1~13 工程思想 + ADR-014 微内核来源单一化

---

## 1. 设计原则

agentrt-linux 源码目录结构遵循五大设计原则，每条原则都有源码级标杆依据：

| # | 原则 | 来源 | 标杆证据 |
|---|------|------|---------|
| 1 | ES-OLK-1 关注点分离 | OLK-6.6 `mm/`/`kernel/`/`fs/`/`net/` 分离 | B1 §1.1 + OLK-6.6 顶层目录 |
| 2 | ES-OLK-2 架构正交 | OLK-6.6 `arch/` 是唯一架构相关根目录 | B1 §1.2 + `arch/x86/`/`arch/arm64/`/`arch/riscv/` |
| 3 | ES-OLK-3/6 UAPI 分离 | OLK-6.6 `include/uapi/` 是用户空间 ABI 唯一来源 | B1 §1.3 + `include/uapi/linux/` |
| 4 | seL4 单编译单元（可选） | seL4 `kernel_all.c` 拼接模式 | B2 §1.1 + `CMakeLists.txt:355-363` |
| 5 | IRON-9 v2 三层模型 | [SC] 共享契约层物理隔离 | `50-engineering-standards/120-cross-project-code-sharing.md` §1.1 |

**关键约束**：
- [SC] 头文件必须 Tab 8 缩进（OLK-6.6 §1，对齐 `01-kernel.md` 源码品味）
- [SC] 头文件必须最小 typedef（OLK-6.6 §5，仅 5 种例外）
- [SC] 头文件变更必须双向 CI 校验（IRON-9 v2）
- [SC] 头文件物理宿主为 `120-cross-project-code-sharing.md` SSoT

> **代码组织模型（v0.2.4 裁决）**：AirymaxOS 采用"**直接写入上游源码树**"模型 —— 所有 Airymax 代码直接放置在 Linux 6.6 内核源码树对应子系统中（如 `kernel/airy_syscall.c`、`mm/airy_rovol.c`），而非通过 `patches/` 隔离层管理。这与 openEuler OLK-6.6 的完整 fork 实践一致（参考 `mm/dynamic_pool.c` 直接写入）。`20-modules/01-kernel.md` 中展示的 `patches/` 目录树为早期设计稿的概念分组，**不作为物理落地路径**。开发者应以下方树为准。

---

## 2. 顶层目录结构

```
agentrt-linux/
├── arch/                          # 架构相关代码（ES-OLK-2）
│   ├── x86/
│   │   ├── include/
│   │   ├── kernel/
│   │   └── mm/
│   ├── arm64/
│   │   ├── include/
│   │   ├── kernel/
│   │   └── mm/
│   └── riscv/
│       ├── include/
│       ├── kernel/
│       └── mm/
│
├── kernel/                        # 内核核心子系统
│   ├── airy_syscall.c            # syscall 分发（12 核心 + 12 预留）
│   ├── airy_cspace.c             # capability 空间（CNode 7 操作）
│   ├── airy_cnode.c              # CNode 操作实现
│   ├── airy_endpoint.c           # IPC endpoint（3 状态机 Idle/Send/Recv）
│   ├── airy_notification.c       # Notification（3 状态 + bitwise OR badge）
│   ├── airy_thread.c             # TCB 管理（含 reply/caller slot）
│   ├── airy_sched.c              # 调度器（EEVDF + sched_ext）
│   ├── airy_fastpath.c           # Fastpath（POINT OF NO RETURN 模式）
│   ├── airy_zombie.c             # Zombie 能力增量清理
│   └── airy_preemption.c         # 抢占点（work-unit 计数器）
│
├── mm/                            # 内存管理子系统
│   ├── airy_rovol.c              # MemoryRovol L1-L4
│   ├── airy_mglru.c              # MGLRU 适配
│   ├── airy_cxl.c                # CXL 内存池化
│   └── airy_page.c               # 页面管理
│
├── ipc/                           # IPC 子系统
│   ├── airy_ipc_msg.c            # IPC 消息处理（Layout C 128B）
│   ├── airy_ipc_transfer.c       # IPC 消息传输
│   └── airy_io_uring.c           # io_uring 数据面（零拷贝优化）
│
├── security/                      # 安全子系统
│   ├── airy_capability.c         # capability 系统（MDB 派生树）
│   ├── airy_lsm.c                # LSM 钩子（250 个）
│   └── airy_cupolas.c            # Cupolas blob
│
├── cognition/                     # 认知子系统
│   ├── airy_clt.c                # CoreLoopThree
│   ├── airy_kthread.c            # kthread 管理（kfifo 通信）
│   └── airy_wasm.c               # Wasm 模块加载
│
├── include/                       # 内核内部头文件
│   ├── uapi/                      # 用户空间 ABI（ES-OLK-3/6）
│   │   ├── airymax/              # [SC] 共享契约层 6 头文件
│   │   │   ├── syscalls.h        # [SC] 12 核心 syscall 编号
│   │   │   ├── ipc.h             # [SC] IPC magic + 128B 消息头
│   │   │   ├── sched.h           # [SC] SCHED_EXT + 任务描述符 magic
│   │   │   ├── security_types.h  # [SC] POSIX capability + LSM 钩子
│   │   │   ├── memory_types.h    # [SC] MemoryRovol L1-L4 + GFP 掩码
│   │   │   └── cognition_types.h # [SC] CoreLoopThree + LLM 推理
│   │   └── airy_syscall.xml      # syscall 编号 XML 源（C-C04）
│   ├── kernel/
│   │   ├── airy_cspace.h
│   │   ├── airy_cnode.h
│   │   ├── airy_endpoint.h
│   │   ├── airy_notification.h
│   │   ├── airy_thread.h
│   │   └── airy_fastpath.h
│   ├── mm/
│   ├── ipc/
│   ├── security/
│   └── cognition/
│
├── tools/                         # 工具链
│   ├── bitfield_gen.py            # 位域 DSL 生成器（C-C02，fork seL4）
│   ├── syscall_header_gen.py     # syscall 头文件生成器（C-C04）
│   └── checkpatch.pl             # 代码规范检查（80 偏好 / 100 硬阈值）
│
├── scripts/                       # 构建脚本
│   ├── Kbuild
│   ├── Kconfig
│   ├── Makefile
│   ├── Makefile.oever             # openEuler 扩展（C-D01）
│   └── missing-syscalls           # syscall 完整性检查（C-D04）
│
├── Documentation/                 # 文档（ES-OLK-13 文档即代码）
│   ├── process/
│   │   ├── coding-style.rst
│   │   ├── submitting-patches.rst
│   │   └── stable-kernel-rules.rst
│   └── admin-guide/
│
├── MAINTAINERS                    # 维护者制度（ES-OLK-8）
├── Kconfig                        # 顶层配置（ES-OLK-4/7）
├── Makefile                       # 顶层构建
├── .clang-format                  # 格式化规范（80 列偏好）
└── .rustfmt.toml                  # Rust 格式化（C-D05）
```

---

## 3. [SC] 共享契约层物理隔离

### 3.1 物理宿主

`include/uapi/airymax/` 目录是 [SC] 共享契约层的物理宿主，6 个头文件：

| 头文件 | 内容 | 共享对象 | 物理宿主 SSoT |
|--------|------|---------|---------------|
| `syscalls.h` | 12 核心 syscall 编号 + 12 预留槽位 | agentrt + agentrt-linux | `120-cross-project-code-sharing.md` §2.3 |
| `ipc.h` | IPC magic 0x41524531 + 128B 消息头（Layout C） | agentrt + agentrt-linux | `120-cross-project-code-sharing.md` §2.7 |
| `sched.h` | SCHED_EXT + 任务描述符 magic 0x41475453 | agentrt + agentrt-linux | `120-cross-project-code-sharing.md` §2.6 |
| `security_types.h` | POSIX capability + LSM 钩子 + Cupolas blob | agentrt + agentrt-linux | `120-cross-project-code-sharing.md` §2.4 |
| `memory_types.h` | MemoryRovol L1-L4 + GFP 掩码 | agentrt + agentrt-linux | `120-cross-project-code-sharing.md` §2.5 |
| `cognition_types.h` | CoreLoopThree + Thinkdual + LLM 推理 | agentrt + agentrt-linux | `120-cross-project-code-sharing.md` §2.8 |

### 3.2 [SC] 头文件铁律

1. **逐字节相同**：agentrt 与 agentrt-linux 两端 [SC] 头文件必须逐字节一致（IRON-9 v2）。
2. **Tab 8 缩进**：[SC] 头文件必须 Tab 8 缩进（对齐 OLK-6.6 §1，对齐 `.clang-format IndentWidth: 8`）。
3. **最小 typedef**：[SC] 头文件必须最小 typedef（对齐 OLK-6.6 §5，仅 5 种例外：`airy_vtime_t`/`airy_cap_op`/`airy_struct_ops_state`/`airy_ipc_msg_type`/`airy_err_t`）。
4. **双向 CI 校验**：[SC] 头文件变更必须通过 agentrt 与 agentrt-linux 双向 CI 校验。
5. **SSoT 物理宿主**：[SC] 头文件物理宿主为 `50-engineering-standards/120-cross-project-code-sharing.md`，本目录头文件为物理落地。
6. **变更流程**：任何 [SC] 变更必须先更新 SSoT，再双向同步两端代码 + CI 验证。

### 3.3 [SS] 语义同源层与 [IND] 独立层

| 层 | 物理位置 | 共享方式 | 变更约束 |
|----|---------|---------|---------|
| [SC] 共享契约层 | `include/uapi/airymax/*.h` | 逐字节相同 | 双向 CI |
| [SS] 语义同源层 | `include/kernel/airy_*.h` + agentrt 对应头 | API 签名独立演进，语义一致 | 文档化语义契约 |
| [IND] 独立层 | `kernel/`/`mm/`/`ipc/`/`security/`/`cognition/` 实现 | 完全独立 | 无约束（遵循 OLK-6.6 代码品味） |

---

## 4. 与 AirymaxOS 8 子仓的映射

agentrt-linux 8 子仓架构（详见 [10-architecture/README.md §2](README.md)）与源码目录的映射：

| AirymaxOS 子仓 | 源码目录 | 物理范围 | 说明 |
|---------------|---------|---------|------|
| kernel | `kernel/` + `arch/` + `include/kernel/` + `include/uapi/` | 内核核心 + 架构相关 + UAPI | 微内核改造落点 |
| services | `ipc/` + `cognition/` 部分 + 用户态守护进程目录（独立） | IPC 子系统 + 认知服务 + 12 daemons | 服务子系统 |
| security | `security/` + `include/security/` | capability + LSM + Cupolas | 安全子系统 |
| memory | `mm/` + `include/mm/` | MemoryRovol + CXL + MGLRU | 内存管理 |
| cognition | `cognition/` + `include/cognition/` | CoreLoopThree kthread + Wasm | 认知子系统 |
| cloudnative | （独立目录，1.0.1 M2+ 评估） | K8s + containerd shim | 云原生适配 |
| system | `Documentation/` + `MAINTAINERS` + `Kconfig` + `Makefile` + `scripts/` | 系统集成 + 构建系统 | 系统集成 |
| tests-linux | `tools/testing/` | 测试框架 | 测试 |

### 4.1 子仓间的依赖关系

```
kernel (L2)
  ↑
  ├── services (L3)
  ├── security (L3)
  ├── memory (L3)
  └── cognition (L4)
        ↑
        ├── cloudnative (L5)
        └── system (L6)
              ↑
              └── tests-linux (L7)
```

**层次纪律**（S-2）：
1. 每层只依赖其直接下层的抽象接口，禁止越级访问
2. 同层之间通过 IPC 通信，禁止直接函数调用
3. 层次之间的接口契约通过 `30-interfaces/` 文档定义
4. 任何新增跨层依赖必须通过 ADR 评审

---

## 5. ES-OLK 工程思想映射

### 5.1 ES-OLK-1~13 落地证据

| ES-OLK | 工程思想 | 落地目录 | 落地证据 |
|--------|---------|---------|---------|
| ES-OLK-1 | 关注点分离 | `kernel/`/`mm/`/`ipc/`/`security/`/`cognition/` 顶层分离 | 5 大子系统目录独立 |
| ES-OLK-2 | 架构正交 | `arch/{x86,arm64,riscv}/` | 唯一架构相关根目录 |
| ES-OLK-3 | UAPI 分离 | `include/uapi/airymax/` | 用户空间 ABI 唯一来源 |
| ES-OLK-4 | 特性可配置 | `Kconfig` + `scripts/Kconfig` | 顶层配置入口 |
| ES-OLK-5 | 条件编译规范化 | `#ifdef CONFIG_*` 配置项 | 与 Kconfig 配合 |
| ES-OLK-6 | UAPI 物理隔离 | `include/uapi/` 物理独立目录 | 与 `include/kernel/` 分离 |
| ES-OLK-7 | 声明式配置 + 命令式构建分离 | `Kconfig`（声明）+ `Makefile`（命令）+ `Makefile.oever`（openEuler 扩展，C-D01） | 三层分离 |
| ES-OLK-8 | 维护者制度 | `MAINTAINERS` | 文件级维护者登记 |
| ES-OLK-9 | 规范可执行 | `tools/checkpatch.pl` + `.clang-format` | 80 偏好 / 100 硬阈值（C-D07） |
| ES-OLK-10 | regression 零容忍 | `tools/testing/`（tests-linux 子仓） | 回归测试 |
| ES-OLK-11 | ABI 永久稳定 | `include/uapi/airymax/syscalls.h` | syscall 编号 MAJOR 版本内不可变更 |
| ES-OLK-12 | 运行时可加载 | `scripts/Kbuild` 模块构建 | 模块化 |
| ES-OLK-13 | 文档即代码 | `Documentation/process/*.rst` | 与源码同仓库 |

### 5.2 OLK-6.6 工程实践对齐（B1 新发现）

| OLK-6.6 实践 | agentrt-linux 落地 | 差距编号 | 优先级 |
|-------------|------------------|---------|--------|
| `Makefile.oever` openEuler 扩展 | `scripts/Makefile.oever` | C-D01 | P1 |
| `lib/Kconfig.openeuler` openEuler Kconfig 扩展 | `scripts/Kconfig` | C-D02 | P1 |
| `arch/*/include/asm/atomic.h` SHA1 校验 | 待引入 | C-D03 | P2 |
| `scripts/missing-syscalls` syscall 完整性检查 | `scripts/missing-syscalls` | C-D04 | P2 |
| `.rustfmt.toml` Rust 格式化 | `.rustfmt.toml` | C-D05 | P2 |
| `EXPORT_TRACEPOINT_SYMBOL_GPL` tracepoint 导出 | 待引入 | C-D06 | P2 |
| `checkpatch.pl max_line_length=100` 100 列硬阈值 | `tools/checkpatch.pl` | C-D07 | P1 |
| `lockdep_assert_*_held` 锁断言 | 待引入 | C-D08 | P2 |

---

## 6. seL4 微内核思想映射

### 6.1 seL4 设计模式对齐（B2 验证）

| seL4 设计模式 | agentrt-linux 落地 | 差距编号 | 优先级 |
|-------------|------------------|---------|--------|
| Liedtke 最小性原则 | `kernel/airy_*.c` 微核心原语 | — | ✅ 已对齐 |
| Capability 系统（CNode 7 操作 + MDB 派生树） | `kernel/airy_cspace.c` + `security/airy_capability.c` | C-C08 | P2 |
| Endpoint 3 状态机（`EPState_Idle/Send/Recv`） | `kernel/airy_endpoint.c` | — | ✅ 已对齐 |
| Reply 原子性（`doReplyTransfer` 临界区串行） | `kernel/airy_endpoint.c` `airy_ipc_reply()` | C-C07 | P2 |
| Notification（bitwise OR badge + 3 状态） | `kernel/airy_notification.c` | C-C01 | P1 |
| Fastpath（POINT OF NO RETURN + 12 项前置检查） | `kernel/airy_fastpath.c` | — | ✅ 已对齐（D2 修复） |
| Zombie 能力增量清理（`reduceZombie` + preemptionPoint） | `kernel/airy_zombie.c` | C-C05 | P2 |
| 极简错误码（11 种 + 哨兵 + 内部 exception 分层） | `airy_err_t`（v0.2.5 已收敛：`typedef int32_t airy_err_t`，登记于 cognition_types.h [SC] 头文件，详见 120-cross-project-code-sharing.md §2.1） | C-C03 | ✅ 已收敛 |
| 单编译单元（`kernel_all.c` 拼接 + `-ffreestanding -nostdinc`） | 待评估 | C-C06 | P2 |
| 位域 DSL（`bitfield_gen.py` + Isabelle/HOL 同源） | `tools/bitfield_gen.py`（fork seL4，移除 HOL 部分） | C-C02 | P1 |
| syscall XML 化（`syscall.xml` + 生成脚本） | `include/uapi/airy_syscall.xml` + `tools/syscall_header_gen.py` | C-C04 | P1 |
| TCB 内嵌 reply slot（`tcbCaller`/`tcbReply`） | `kernel/airy_thread.c`（待引入） | C-C07 | P2 |

### 6.2 6 项新发现 seL4 设计模式（B2 新发现）

| seL4 设计模式 | 来源 | agentrt-linux 落地建议 | 优先级 |
|-------------|------|---------------------|--------|
| prune + 生成头文件分离构建 | seL4 `tools/bitfield_gen.py` | 1.0.1 M2+ 评估 | P2 |
| preemptionPoint 精细化（work-unit 计数器） | seL4 `src/kernel/thread.c` | `kernel/airy_preemption.c` 吸收 | P1 |
| 三态 IRQ 状态机（IRQInactive/IRQSignal/IRQTimer/IRQIPI） | seL4 `include/object/structures.h` | 1.0.1 M2+ 评估 | P2 |
| 调度 bitmap 二级索引 + 倒序优化 | seL4 `src/kernel/sched.c` | 1.0.1 M3+ 评估 | P2 |
| `field_ptr(N)` 类型标记技巧 | seL4 `include/object/structures.h` | 1.0.1 M2+ 评估 | P2 |
| Haskell error 注释模式 | seL4 assert 注释 | 0.1.1 文档层吸收 | P1 |

---

## 7. 文件命名规范

### 7.1 命名约定

| 类型 | 命名规则 | 示例 |
|------|---------|------|
| 内核源文件 | `airy_<子系统>_<功能>.c` | `airy_ipc_msg.c`、`airy_endpoint.c` |
| 内核头文件 | `airy_<子系统>_<功能>.h` | `airy_cspace.h`、`airy_endpoint.h` |
| [SC] UAPI 头文件 | `<语义域>.h`（无 `airy_` 前缀） | `syscalls.h`、`ipc.h`、`sched.h` |
| 架构相关文件 | `arch/<arch>/<子系统>/` | `arch/x86/kernel/airy_syscall.c` |
| 工具脚本 | `<工具名>_<类型>.py`/`.pl` | `bitfield_gen.py`、`checkpatch.pl` |
| 构建脚本 | `<构建系统>.<扩展>` | `Makefile`、`Kconfig`、`Kbuild`、`Makefile.oever` |

### 7.2 函数前缀规范

- **内核函数**：`airy_<子系统>_<功能>()`，例如 `airy_ipc_send()`、`airy_endpoint_recv()`
- **[SC] 共享函数**：`airy_<语义域>_<功能>()`，例如 `airy_vtime_decay()`（agentrt 与 agentrt-linux 共享）
- **禁止前缀**：`airymaxos_*`（已废弃，详见 project_memory）、`agentrt_*`（已改名）、`agentos_*`（已废弃）

---

## 8. 与现有文档的关系

### 8.1 文档体系映射

| 文档 | 目录结构关注点 |
|------|-------------|
| [10-architecture/README.md](README.md) | 8 子仓架构 + 7 层架构模型 |
| [10-architecture/01-system-architecture.md](01-system-architecture.md) | 系统架构总览 |
| [10-architecture/03-microkernel-strategy.md](03-microkernel-strategy.md) | 微内核化改造策略 |
| [10-architecture/04-engineering-baseline.md](04-engineering-baseline.md) | 工程基线（Linux 6.6 内核基线锁定） |
| [20-modules/](../20-modules/) | 8 子仓详细模块设计 |
| [30-interfaces/](../30-interfaces/) | syscall + IPC + SDK + 编码规范 |
| [40-dataflows/](../40-dataflows/) | 4 大数据流（认知 + 记忆 + IPC + 调度） |
| [50-engineering-standards/120-cross-project-code-sharing.md](../50-engineering-standards/120-cross-project-code-sharing.md) | [SC] 共享契约层 SSoT |
| [50-engineering-standards/09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md) | 全局规则编号 SSoT |

### 8.2 本文档的 SSoT 范围

本文档是 agentrt-linux 源码目录结构的 SSoT，所有子目录设计、文件命名、[SC] 物理宿主归属以本文档为准。其他文档涉及目录结构时引用本文档，禁止重新定义。

---

## 9. 实施路线图

### 9.1 0.1.1（文档体系完成）

- [x] 本文档创建（Wave 2 v2 Phase D D9）
- [x] 目录结构设计经 Phase B/C 源码深读验证
- [x] [SC] 6 头文件物理宿主确认（`include/uapi/airymax/`）
- [x] ES-OLK-1~13 落地证据映射
- [x] seL4 设计模式对齐状态评估

### 9.2 1.0.1 M1（代码开发启动）

- [ ] `kernel/` 子系统骨架代码填充（10 个 `.c` 文件）
- [ ] `include/uapi/airymax/` 6 头文件物理落地
- [ ] `arch/{x86,arm64,riscv}/` 三架构骨架
- [ ] `scripts/` 构建系统（`Makefile`/`Kconfig`/`Kbuild`/`Makefile.oever`）
- [ ] `tools/checkpatch.pl` + `.clang-format` 落地
- [ ] `MAINTAINERS` 文件级维护者登记

### 9.3 1.0.1 M2+（深度借鉴）

- [ ] `tools/bitfield_gen.py` fork seL4 + 移除 HOL 部分（C-C02）
- [ ] `include/uapi/airy_syscall.xml` + `tools/syscall_header_gen.py`（C-C04）
- [ ] `kernel/airy_notification.c` 重新设计（3 状态 + bitwise OR badge，C-C01）
- [ ] `kernel/airy_zombie.c` 增量清理算法（C-C05）
- [ ] TCB 内嵌 reply slot（C-C07）
- [ ] MDB 派生树增强（C-C08）
- [ ] CapRights 4 位掩码（C-C09）

---

## 10. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本（Wave 2 v2 Phase D D9 创建，基于 Phase C §7 建议） |
| 1.0.1 M1 | 2027-XX-XX | M1 代码开发启动，骨架代码填充 |
| 1.0.1 M2+ | 2027-XX-XX | seL4 深度借鉴（bitfield DSL / Notification 重设计 / Zombie 清理等） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
