Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 记忆设计文档

> **文档定位**：agentrt-linux（AirymaxOS）记忆设计文档（memory，极境记忆&存储）\
> **文档版本**：v1.0.1\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **核心约束**：IRON-9 v3 同源且部分代码共享——与 agentrt 用户态 memoryrovol 通过 \[SC] 共享契约层 + \[SS] 语义同源层协作，\[IND] 内核态 CXL/PMEM/VFS 持久化实现独立\
> **子仓编号**：04\
> **子仓代号**：极境记忆（Airymax Memory）\
> **设计基准**：MemoryRovol 内核态 + CXL 内存分层 + PMEM 持久化 + MGLRU 多代回收\
> **同源 agentrt**：heapstore + memoryrovol（MemoryRovol）\
> **横切关注点**：记忆卷载贯穿调度（记忆迁移感知）、IPC（快照传递）、eBPF（回收追踪）、安全（记忆加密）4 大数据流

***

## 目录

- [1. 子仓职责](#1-子仓职责)
- [2. 同源关系（IRON-9 v3 四层共享模型）](#2-同源关系iron-9-v3-四层共享模型)
- [3. 目录结构](#3-目录结构)
- [4. 核心特性](#4-核心特性)
  - [4.8 mem_d daemon 设计（12 daemon 之一）](#48-mem_d-daemon-设计12-daemon-之一)
  - [4.9 io_uring 工程规范与三独立数据区](#49-io_uring-工程规范与三独立数据区)
  - [4.10 Badge 在 MemoryRovol 中的应用](#410-badge-在-memoryrovol-中的应用)
  - [4.11 ADR 引用](#411-adr-引用)
- [5. 微内核思想体现](#5-微内核思想体现)
- [6. IRON-9 v3 四层共享模型落地](#6-iron-9-v3-四层共享模型落地)
- [7. agentrt-linux 工程基线](#7-agentrt-linux-工程基线)
- [8. 前沿理论参考](#8-前沿理论参考)
- [9. 与其他子仓的协作](#9-与其他子仓的协作)
- [10. 里程碑（M0-M8）](#10-里程碑m0-m8)
- [11. agentrt 一致性检查](#11-agentrt-一致性检查)
- [12. 相关文档](#12-相关文档)
- [13. 参考](#13-参考)
- [14. 变更历史](#14-变更历史)

***

## 1. 子仓职责

`memory` 是 agentrt-linux（AirymaxOS）的记忆与存储子仓，承担以下核心职责：

1. **MemoryRovol 内核态实现 \[SS]**：将 agentrt 的 MemoryRovol（记忆卷载）升级为内核态实现，提供 Agent 记忆的持久化与卷载能力。L1-L4 层级枚举 `airy_mem_level` \[SC] 与 agentrt 共享。
2. **CXL 内存分层与池化 \[IND]**：利用 2026 年 CXL 3.0 硬件普及，实现内存分层与池化。
3. **持久化内存（PMEM）\[IND]**：基于 PMEM 提供非易失性内存支持。PMEM 设备驱动与持久化接口实现属 \[IND] 独立层（SSoT 头文件中仅以 `AIRY_MEM_PMEM` 枚举值与 `AIRY_GFP_PMEM` 标志形式出现，详见 §4.3）。
4. **MGLRU \[SS]**：利用 Linux 6.6 多代 LRU 改进，优化内存回收策略。`AIRY_GFP_*` tier 标志与 `AIRY_PAGE_CLASS_*` 页面分类 \[SC] 与 agentrt 共享。
5. **VFS 持久化层 \[IND]**：为 `services/vfs` 提供持久化后端。
6. **userfaultfd 用户态缺页处理 \[SS]**：支持用户态缺页处理，用于记忆迁移与快照。
7. **透明巨页（THP）优化 \[IND]**：利用 Linux 6.6 THP 改进提升大页性能。

### 1.1 横切关注点声明

记忆卷载贯穿 agentrt-linux 全部 4 大数据流：

| 数据流      | 记忆切入点                               | 同源标注   |
| -------- | ----------------------------------- | ------ |
| 调度数据流    | 记忆迁移感知——迁移期间调整调度优先级                 | \[SS]  |
| IPC 数据流  | 快照传递——MemoryRovol 快照通过 io\_uring 传递 | \[SS]  |
| eBPF 数据流 | MGLRU 回收追踪——BPF 追踪 aging/eviction   | \[SS]  |
| 安全数据流    | 记忆加密——MemoryRovol 加密与 TEE 保护        | \[IND] |

***

## 2. 同源关系（IRON-9 v3 四层共享模型）

依据 IRON-9 v3 决策，agentrt（用户态 memoryrovol）与 agentrt-linux（内核态 memory）通过 v3 四层共享模型（[SC] 共享契约 + [SS] 同源签名 + [IND] 独立 + [DSL] 降级生存）协作：

| 层次               | 共享程度          | 记忆子系统内容                                                                                                                  | 组织方式                             |
| ---------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------ | -------------------------------- |
| **\[SC] 共享契约层**  | 完全共享代码        | `airy_mem_level` L1-L4 层级枚举、`AIRY_GFP_*` 4 个 tier 标志、`AIRY_PAGE_CLASS_*` 4 个页面分类标志、`#ifdef AIRY_SC_FALLBACK` 降级块 | `include/uapi/linux/airymax/memory_types.h`（10 个 [SC] 头文件之一） |
| **\[SS] 语义同源层**  | 操作模式同源，函数签名独立 | `rovol_snapshot()`、`rovol_restore()`、`rovol_migrate()`、`rovol_compress()`、MGLRU aging/eviction 语义、userfaultfd 处理接口 等 6 项 | 各自独立实现                           |
| **\[IND] 完全独立层** | 完全独立          | CXL 设备驱动、PMEM 设备驱动、VFS 持久化层实现、THP 优化实现、zswap/zram 集成                                                                     | 各自独立仓库                           |
| **\[DSL] 降级生存层** | 降级模式生存        | `#ifdef AIRY_SC_FALLBACK` 降级块位于 `memory_types.h` 底部——L1-L4 tier 折叠为 L1 hot 单层、`AIRY_GFP_*` 折叠为 `AIRY_GFP_HOT`、`AIRY_PAGE_CLASS_*` 折叠为 `AIRY_PAGE_CLASS_ANON`（详见 §6.4 SSoT 实际 DSL 块） | 每个 \[SC] 头文件底部 `#ifdef AIRY_SC_FALLBACK` 块 |

### 2.1 维度对比

| 维度       | agentrt（heapstore + memoryrovol） | agentrt-linux（memory）           | 同源标注   |
| -------- | -------------------------------- | ------------------------------- | ------ |
| 记忆存储     | heapstore（用户态）                   | MemoryRovol 内核态 + heapstore 用户态 | \[SS]  |
| 记忆卷载     | MemoryRovol（用户态）                 | MemoryRovol 内核态实现               | \[SS]  |
| 持久化      | 文件系统                             | PMEM + CXL + VFS 持久化层           | \[IND] |
| 分层       | 用户态分层                            | CXL 内存分层 + MGLRU                | \[IND] |
| L1-L4 层级枚举 | 用户态枚举（直接 include SSoT）            | 内核态枚举（直接 include SSoT）         | \[SC]  |
| `AIRY_GFP_*` tier 标志 | 应用层标志                            | 内核分配标志                          | \[SC]  |
| `AIRY_PAGE_CLASS_*` 页面分类 | 应用层标志                            | 内核页面分类标志                        | \[SC]  |
| PMEM 设备驱动 | 不实现                              | 内核 nvdimm 驱动                    | \[IND] |

### 2.2 同源传承要点

- 保留 agentrt MemoryRovol 的"记忆卷载"语义（snapshot + restore）\[SS]。
- 保留 heapstore 的"记忆存储"抽象 \[SS]。
- `airy_mem_level` L1-L4 层级枚举 \[SC] 共享，确保两端记忆层级语义一致。
- `AIRY_GFP_*` tier 标志与 `AIRY_PAGE_CLASS_*` 页面分类 \[SC] 共享，便于用户态代码移植到内核态。
- PMEM 设备驱动与持久化接口实现属 \[IND] 独立层（SSoT 中无对应共享结构）。
- 升级为内核态实现，利用 CXL/PMEM 硬件加速 \[IND]。

***

## 3. 目录结构

```
memory/
├── memoryrovol/            # MemoryRovol 内核态实现（记忆卷载）[SS]
├── cxl/                    # CXL 内存分层与池化 [IND]
├── pmem/                   # 持久化内存 [IND]
├── mglru/                  # MGLRU（Linux 6.6 多代 LRU）[SS]
├── vfs-persist/            # VFS 持久化层 [IND]
├── userfaultfd/           # 用户态缺页处理 [SS]
├── thp/                    # 透明巨页优化 [IND]
└── docs/
```

### 3.1 memoryrovol/（MemoryRovol 内核态实现）\[SS]

参考 agentrt MemoryRovol 设计，L1-L4 层级枚举 `airy_mem_level` \[SC] 共享：

- `rovol-kmod`：内核模块，提供记忆卷载系统调用 \[SS]。
- `snapshot`：记忆快照（基于 fork + COW）\[SS]。
- `restore`：记忆恢复（基于 mmap + userfaultfd）\[SS]。
- `migrate`：记忆迁移（跨节点、跨 CXL 设备）\[SS]。
- `compress`：记忆压缩（zswap、zram 集成）\[IND]。
- `encrypt`：记忆加密（与 `security` 协作）\[IND]。

### 3.2 cxl/（CXL 内存分层与池化）\[IND]

基于 **CXL 3.0** 规格：

- `cxl-type2`：CXL Type 2 设备支持（缓存一致内存）。
- `cxl-type3`：CXL Type 3 设备支持（内存扩展）。
- `tiering`：内存分层策略（FAST/CXL/PMEM tier）。
- `pooling`：内存池化（跨节点共享）。
- `hotplug`：CXL 内存热插拔。

### 3.3 pmem/（持久化内存）\[IND]

PMEM 设备驱动与持久化接口实现属 \[IND] 独立层（SSoT 头文件中仅以 `AIRY_MEM_PMEM` 枚举值与 `AIRY_GFP_PMEM` 标志形式出现，无共享 PMEM 接口结构）：

- `pmem-driver`：PMEM 设备驱动（nvdimm）。
- `dax`：DAX（Direct Access）模式，绕过 page cache。
- `fsdax`：文件系统 DAX（ext4-dax、xfs-dax）。
- `devdax`：设备 DAX（字符设备模式）。

### 3.4 mglru/（MGLRU）\[SS]

利用 **Linux 6.6** MGLRU 改进，`AIRY_GFP_*` tier 标志与 `AIRY_PAGE_CLASS_*` 页面分类 \[SC] 共享：

- `multi-gen-lru`：多代 LRU 回收策略 \[SS]。
- `aging`：老化策略（按代标记页面）\[SS]。
- `eviction`：逐出策略（按代逐出）\[SS]。
- `workingset-protection`：工作集保护 \[SS]。

### 3.5 vfs-persist/（VFS 持久化层）\[IND]

为 `services/vfs` 提供持久化后端：

- `backends/`：后端实现（PMEM、CXL、SSD、HDD）。
- `journal`：日志系统（WAL）。
- `snapshot`：文件系统快照。
- `dedup`：去重。
- `compress`：压缩（zstd、lz4）。

### 3.6 userfaultfd/（用户态缺页处理）\[SS]

- `uffd-handler`：用户态缺页处理框架 \[SS]。
- `live-migration`：进程实时迁移 \[SS]。
- `snapshot`：进程快照（与 MemoryRovol 协作）\[SS]。
- `postcopy`：post-copy 迁移策略 \[SS]。

### 3.7 thp/（透明巨页优化）\[IND]

利用 **Linux 6.6** THP 改进：

- `hugepages`：大页分配策略。
- `khugepaged`：大页合并守护进程。
- `madvise`：madvise 策略（MADV\_HUGEPAGE）。
- `shmem`：shmem 大页支持。

#### 3.8 组件架构图

```mermaid
graph TD
    subgraph SC["[SC] 共享契约层 include/uapi/linux/airymax/memory_types.h"]
        MEMLEVEL[airy_mem_level 枚举<br/>HOT/WARM/COLD/PMEM]
        GFP[AIRY_GFP_* 4 个 tier 标志<br/>HOT/WARM/COLD/PMEM]
        PAGECLASS[AIRY_PAGE_CLASS_* 4 个页面分类<br/>ANON/FILE/SHMEM/AGENT]
        DSL[AIRY_SC_FALLBACK 降级块]
    end
    subgraph SS["[SS] 语义同源层"]
        MEMROVOL[MemoryRovol 内核态实现]
        MGLRU[MGLRU aging / eviction]
        RECLAIM[memory.reclaim 主动回收]
        UFFD[userfaultfd 记忆迁移]
    end
    subgraph IND["[IND] 独立层"]
        CXL[CXL 内存分层与池化]
        PMEM[PMEM nvdimm 驱动]
        VFSPERSIST[VFS 持久化层]
        THP[透明巨页优化]
    end
    MEMLEVEL --> MEMROVOL
    MEMLEVEL --> CXL
    MEMLEVEL --> PMEM
    GFP --> MGLRU
    PAGECLASS --> MGLRU
    DSL --> MEMROVOL
    MEMROVOL --> RECLAIM
    MEMROVOL --> UFFD
    CXL --> THP
```

***

## 4. 核心特性

### 4.1 MemoryRovol 内核态实现（同源）\[SS]

**记忆卷载语义** \[SS]——操作模式同源（概念操作一致），函数签名因抽象层级不同而独立：

- `rovol_snapshot(pid)`：对指定进程创建记忆快照 \[SS]。
- `rovol_restore(snapshot_id)`：从快照恢复记忆 \[SS]。
- `rovol_migrate(pid, target_node)`：迁移进程记忆至目标节点 \[SS]。
- `rovol_compress(snapshot_id)`：压缩快照 \[SS]。

**MemoryRovol L1-L4 层级定义** \[SC]（`include/uapi/linux/airymax/memory_types.h`）：

> **SSoT 声明**：以下为 `include/uapi/linux/airymax/memory_types.h` 中实际定义的内容，agentrt 用户态与 agentrt-linux 内核态两端直接 include 共享。SSoT 头文件**仅定义 L1-L4 层级枚举 `airy_mem_level`，不定义任何 L1-L4 Record/Block/Entry/Persistent 结构体**——具体记忆记录的数据结构由各端在 \[SS]/\[IND] 层独立定义。

```c
/* 来源：include/uapi/linux/airymax/memory_types.h */
/* MemoryRoVol types — [SC] shared contract header.
 * L1-L4 memory tiering definitions and GFP mask semantics.
 */

/* ─── Memory Tier Levels ─────────────────────────────────────────────── */
enum airy_mem_level {
	AIRY_MEM_HOT    = 0,   /* L1: HBM/DDR hot tier */
	AIRY_MEM_WARM   = 1,   /* L2: DDR warm tier */
	AIRY_MEM_COLD   = 2,   /* L3: CXL/NVMe cold tier */
	AIRY_MEM_PMEM   = 3,   /* L4: PMEM persistent tier */
	AIRY_MEM_LEVEL_MAX
};
```

> **层级语义映射**：SSoT 中 L1-L4 仅作为 `airy_mem_level` 枚举的层级标识符，对应 MemoryRovol 的四层记忆语义：
>
> | 枚举值 | 层级 | MemoryRovol 语义 | 物理后端 |
> | --- | --- | --- | --- |
> | `AIRY_MEM_HOT` | L1 | 实时记忆层（高频读写） | DRAM / HBM |
> | `AIRY_MEM_WARM` | L2 | 短期记忆层（MGLRU aging） | DDR |
> | `AIRY_MEM_COLD` | L3 | 长期记忆层（低频读写） | CXL / NVMe |
> | `AIRY_MEM_PMEM` | L4 | 持久记忆层（持久化） | PMEM |
>
> **说明性示例，非 SSoT 定义**：L1-L4 各层级具体记忆记录的数据结构（含 trace_id / generation / access_count / checksum 等字段）由 agentrt 与 agentrt-linux 在 \[SS]/\[IND] 层各自独立定义，**不在 `memory_types.h` 中共享**。本节早期版本曾虚构 `airy_mr_l1_record_t` / `airy_mr_l2_block_t` / `airy_mr_l3_entry_t` / `airy_mr_l4_persistent_t` 四个结构体并误标为 \[SC] 共享，已在 v1.0.1-fix 移除——SSoT 头文件中并不存在这些类型。

**实现机制** \[IND]：

- 快照基于 fork + COW（用户空间）或 fork + userfaultfd（内核空间）。
- 恢复基于 mmap + userfaultfd 按需加载。
- 迁移基于 userfaultfd post-copy。

### 4.2 CXL 内存分层与池化（2026 硬件普及）\[IND]

**CXL 3.0 规格**（2026 硬件普及）：

- cache coherent：缓存一致性，简化编程模型。
- multi-host：多主机共享内存池。
- switching：CXL switch 支持复杂拓扑。

**分层策略** \[IND]：

| Tier | 设备         | 延迟      | 用途  | 对应 MemoryRovol  |
| ---- | ---------- | ------- | --- | --------------- |
| FAST | DRAM       | \~100ns | 热数据 | L1 实时记忆 \[SC]   |
| CXL  | CXL memory | \~200ns | 温数据 | L2/L3 中长期 \[SS] |
| PMEM | 持久内存       | \~300ns | 持久化 | L4 持久 \[SC]     |
| SSD  | NVMe SSD   | \~10μs  | 冷数据 | 归档 \[IND]       |

**池化** \[IND]：

- 多节点共享 CXL 内存池。
- 动态分配/释放内存至不同节点。
- 故障切换（节点宕机时内存迁移）。

### 4.3 PMEM 持久化内存 \[IND]

> **SSoT 声明**：`include/uapi/linux/airymax/memory_types.h` 中**并未定义** `airy_pmem_ops_t` 结构体或任何 PMEM 操作回调接口。本节早期版本曾虚构 `airy_pmem_ops_t`（含 persist/flush/map/unmap 四函数指针）并误标为 \[SC] 共享，已在 v1.0.1-fix 移除——SSoT 中 PMEM 仅以 `AIRY_MEM_PMEM` 枚举值（L4 持久层级）与 `AIRY_GFP_PMEM` 分配标志（0x08）形式出现。PMEM 设备驱动与持久化接口实现属 \[IND] 独立层，由 agentrt-linux 内核态 `pmem/` 目录独立实现（详见 §3.3）。

**SSoT 中与 PMEM 相关的实际定义** \[SC]（`include/uapi/linux/airymax/memory_types.h`）：

```c
/* 来源：include/uapi/linux/airymax/memory_types.h */
/* L4 持久层级枚举值 */
enum airy_mem_level {
	/* ... */
	AIRY_MEM_PMEM   = 3,   /* L4: PMEM persistent tier */
	/* ... */
};

/* L4 PMEM tier 分配标志 */
#define AIRY_GFP_PMEM   0x08   /* Allocate from PMEM tier */
```

**特性** \[IND]：

- 非易失性：断电后数据保留。
- 字节寻址：像内存一样访问。
- 低延迟：\~300ns（比 SSD 快 30 倍）。

**应用** \[IND]：

- Agent 记忆持久化（MemoryRovol L4 后端）。
- 文件系统元数据（DAX 模式）。
- 日志系统（WAL）。

### 4.4 MGLRU（Linux 6.6 多代 LRU）\[SS]

**GFP 掩码语义** \[SC]（`include/uapi/linux/airymax/memory_types.h`）：

> **SSoT 声明**：以下为 `include/uapi/linux/airymax/memory_types.h` 中实际定义的 GFP 掩码宏，agentrt 用户态与 agentrt-linux 内核态两端直接 include 共享。SSoT 中**仅定义 4 个 tier 选择标志**（HOT/WARM/COLD/PMEM），用于在 MemoryRovol L1-L4 层级间选择分配目标。本节早期版本曾虚构 `AIRY_GFP_IO` / `AIRY_GFP_FS` / `AIRY_GFP_RECLAIM` / `AIRY_GFP_KSWAPD` / `AIRY_GFP_HIGH` / `AIRY_GFP_NOWARN` / `AIRY_GFP_ZERO` 等 7 个仿 Linux 内核 GFP 标志并误标为 \[SC] 共享，已在 v1.0.1-fix 移除——这些标志属 Linux 内核原生 GFP 体系，不在 `memory_types.h` 共享契约层中重复定义。

```c
/* 来源：include/uapi/linux/airymax/memory_types.h */
/* ─── GFP Mask Semantics for MemoryRoVol ──────────────────────────────── */
#define AIRY_GFP_HOT    0x01   /* Allocate from hot tier   (L1: HBM/DDR) */
#define AIRY_GFP_WARM   0x02   /* Allocate from warm tier  (L2: DDR)      */
#define AIRY_GFP_COLD   0x04   /* Allocate from cold tier  (L3: CXL/NVMe) */
#define AIRY_GFP_PMEM   0x08   /* Allocate from PMEM tier  (L4: PMEM)     */
```

> **SSoT 中另定义的页面分类宏**（`include/uapi/linux/airymax/memory_types.h` 同文件）：
>
> ```c
> /* 来源：include/uapi/linux/airymax/memory_types.h */
> /* ─── Memory Page Classification ──────────────────────────────────────── */
> #define AIRY_PAGE_CLASS_ANON     0x01  /* Anonymous page */
> #define AIRY_PAGE_CLASS_FILE     0x02  /* File-backed page */
> #define AIRY_PAGE_CLASS_SHMEM    0x04  /* Shared memory page */
> #define AIRY_PAGE_CLASS_AGENT    0x08  /* Agent-private page */
> ```
>
> **说明性示例，非 SSoT 定义**：Linux 原生 GFP 体系（`GFP_KERNEL` / `GFP_ATOMIC` / `__GFP_IO` / `__GFP_FS` / `__GFP_RECLAIM` / `__GFP_KSWAPD` 等）由 `include/linux/gfp.h` 提供，agentrt-linux 内核态实现直接复用，**不在 `memory_types.h` 中重复定义**。`AIRY_GFP_*` 仅作为 MemoryRovol 跨端共享的 tier 选择语义，与 Linux GFP 体系正交。

**改进** \[SS]：

- 多代回收：页面按代分组，按代逐出。
- 工作集保护：识别并保护活跃工作集。
- 更优的内存压力应对。

**配置** \[IND]：

```
echo y > /sys/kernel/mm/lru_gen/enabled
echo 1000 > /sys/kernel/mm/lru_gen/max_seq
```

### 4.5 VFS 持久化层 \[IND]

**多后端支持** \[IND]：

- PMEM：高性能持久化。
- CXL：可共享持久化。
- SSD：大容量持久化。
- HDD：归档持久化。

**特性** \[IND]：

- 写前日志（WAL）：保证崩溃一致性。
- 快照：文件系统级快照。
- 去重：块级去重。
- 压缩：zstd/lz4 压缩。

### 4.6 userfaultfd 用户态缺页处理 \[SS]

**用例** \[SS]：

- 进程实时迁移：将进程从一节点迁移至另一节点。
- 进程快照：创建进程记忆快照。
- 按需加载：仅在访问时加载页面。
- 惰性恢复：从快照惰性恢复。

**API** \[SS]：

```c
struct uffdio_api api = { .api = UFFD_API };
ioctl(uffd, UFFDIO_API, &api);
/* 注册缺页处理区域 */
struct uffdio_register reg = {
    .range = { .start = addr, .len = size },
    .mode = UFFDIO_REGISTER_MODE_MISSING,
};
ioctl(uffd, UFFDIO_REGISTER, &reg);
```

### 4.7 透明巨页（THP）优化（Linux 6.6）\[IND]

**Linux 6.6 改进** \[IND]：

- 更激进的 khugepaged 合并策略。
- shmem 大页支持改进。
- madvise 行为更可预测。
- 减少 THP 抖动。

**配置** \[IND]：

```
echo always > /sys/kernel/mm/transparent_hugepage/enabled
echo madvise > /sys/kernel/mm/transparent_hugepage/shmem_enabled
```

### 4.8 mem_d daemon 设计（12 daemon 之一）

`mem_d` 是 AirymaxOS 用户态 **12 daemon** 之一，承担记忆卷载管理职责。daemon 命名后缀 `_d`（详见 [01-kernel.md §14.2](01-kernel.md)）。

| 维度 | 设计 |
| --- | --- |
| 职责 | MemoryRovol L1-L4 快照/恢复/迁移管理 |
| 内核切入点 | userfaultfd + MGLRU + CXL bus |
| 控制通道 | `airy_sys_rovol_ctl`（编号 1）——记忆卷载控制 syscall |
| 数据通道 | io_uring `IORING_OP_URING_CMD`（SQE128 模式，cmd 扩展至 80 字节） |
| daemon 命名 | `mem_d`（`_d` 后缀，12 daemon 命名约定） |

**`airy_sys_rovol_ctl`（编号 1）控制接口**：

```c
/* mem_d 通过 airy_sys_rovol_ctl 控制记忆卷载——4 核心 syscall 之一 */
enum airy_rovol_opcode {
    AIRY_ROVOL_SNAPSHOT   = 1,  /* 创建记忆快照 [SS] */
    AIRY_ROVOL_RESTORE    = 2,  /* 从快照恢复 [SS] */
    AIRY_ROVOL_MIGRATE    = 3,  /* 跨节点/跨 tier 迁移 [SS] */
    AIRY_ROVOL_COMPRESS   = 4,  /* 压缩快照 [SS] */
    AIRY_ROVOL_REGISTER   = 5,  /* 注册 io_uring registered buffer [IND] */
    AIRY_ROVOL_QUERY_TIER = 6,  /* 查询页面所在 tier [IND] */
};

int airy_sys_rovol_ctl(enum airy_rovol_opcode op,
                       struct airy_rovol_args __user *args);
```

> **mem_d 与 sec_d 协作**：mem_d 的所有记忆卷载操作受 `sec_d` 颁发的 Badge 权限控制（详见 §4.10）。`airy_sys_rovol_ctl` 入口校验调用方 Badge 的 Perms 位段是否含 `AIRY_CAP_PERM_ROVOL_*` 权限位。

### 4.9 io_uring 工程规范与三独立数据区

#### 4.9.1 io_uring 工程规范

OLK 6.6 内核基线下，记忆卷载数据面完全由 io_uring 承载（v1.0.1 Capability Folding 后 8 个 seL4 风格 IPC 原语 syscall 已全部移除）：

| 规范 | 说明 |
| --- | --- |
| `io_uring_cmd_to_pdu(cmd, pdu_type)` | OLK 6.6 安全宏，访问 `struct io_uring_cmd` 的 `pdu[32]` 字段（仅内联存储 cmd 前 32 字节副本） |
| `io_uring_cmd_done(cmd, ret, res2, issue_flags)` | OLK 6.6 标准 4 参数完成接口（禁止使用旧版 2 参数变体） |
| `security_uring_cmd(struct io_uring_cmd *ioucmd)` | OLK 6.6 LSM 钩子，**单参数**——airy_lsm 通过 `LSM_ORDER_MUTABLE` 注册叠加 Badge 审计 |
| SQE128 模式 | `IORING_SETUP_SQE128`——SQE 64B→128B，cmd 16B→80B，承载 `airy_ipc_cmd` |
| UAPI 路径 | `include/uapi/linux/airymax/`（标准路径） |

**mem_d 使用 `io_uring_cmd_to_pdu()` 示例**：

```c
/* mem_d io_uring cmd 处理——访问 pdu[32] 字段 */
static int mem_d_uring_cmd_handler(struct io_uring_cmd *ioucmd)
{
    /* pdu[32] 仅承载前 32 字节副本——含 rovol_opcode + agent_id */
    struct airy_rovol_cmd *cmd = io_uring_cmd_to_pdu(ioucmd, struct airy_rovol_cmd);
    int ret;

    /* fastpath C-S9 Badge 校验已在 io_uring issue 路径内联完成 */
    switch (cmd->rovol_opcode) {
    case AIRY_ROVOL_SNAPSHOT:
        ret = mem_d_do_snapshot(cmd->agent_id, cmd->snapshot_id);
        break;
    case AIRY_ROVOL_RESTORE:
        ret = mem_d_do_restore(cmd->agent_id, cmd->snapshot_id);
        break;
    /* ... */
    }

    /* OLK 6.6 标准 4 参数完成接口 */
    io_uring_cmd_done(ioucmd, ret, 0, 0);
    return 0;
}
```

#### 4.9.2 三独立数据区

mem_d 运行时维护**三个相互独立的数据区**，物理隔离避免相互污染：

| 数据区 | 大小 | 用途 | 分配方式 | 写者 |
| --- | --- | --- | --- | --- |
| **`agent_caps[1024]` 静态数组** | 128KB | capability Badge 存储（与 sec_d 共享全局实例） | 内核静态分配（`__aligned(64)` slot） | sec_d 唯一写者（mem_d 只读） |
| **`airy_ipc_ring` kfifo** | 默认 1MB | io_uring SQ/CQ ring + IPC 消息传递 | `kfifo_alloc()` 内核动态分配 | mem_d 内核侧 + io_uring issue 路径 |
| **io_uring registered buffer** | 默认 64MB | MemoryRovol L1 实时记忆数据零拷贝区 | `alloc_pages()` + `mmap()` 用户态映射 | mem_d 用户态 + 内核零拷贝读 |

> **三独立数据区设计原则**：capability（控制面）+ IPC ring（消息面）+ registered buffer（数据面）物理隔离，避免数据面负载影响控制面 Badge 校验延迟。这是 seL4 "机制与策略分离"思想在记忆子仓的体现（ADR-014）。

#### 4.9.3 io_uring registered buffer + alloc_pages + mmap（替代 page flipping）

MemoryRovol L1 实时记忆数据通过 **io_uring registered buffer** 零拷贝传递，替代传统 page flipping 方案：

| 维度 | 传统 page flipping | io_uring registered buffer |
| --- | --- | --- |
| 内存分配 | `alloc_page()` 单页 + `vm_insert_page()` | `alloc_pages()` 批量 + `mmap()` 用户态映射 |
| DMA 一致性 | 需要 DMA 一致性内存（`dma_alloc_coherent`） | **不使用 DMA 一致性内存**——纯 CPU 内存 + userfaultfd |
| 零拷贝 | 页表翻转（`mmap()` + `remap_pfn_range()`） | registered buffer 固定页 + io_uring 直接 DMA（如有设备） |
| 权限校验 | 每次翻转需 `cap_capable()` | 注册时一次性 Badge 校验 + 运行时 fastpath C-S9 内联 |
| 用户态访问 | 翻转后用户态可见 | `mmap()` 持久映射，无需翻转 |

**registered buffer 注册流程**（mem_d 启动时）：

```c
/* mem_d 启动时注册 io_uring registered buffer——alloc_pages + mmap 模式 */
int mem_d_register_ring_buffer(int ring_fd, size_t size)
{
    struct page **pages;
    void *vaddr;
    int nr_pages, ret;

    nr_pages = (size + PAGE_SIZE - 1) >> PAGE_SHIFT;
    pages = kvmalloc_array(nr_pages, sizeof(*pages), GFP_KERNEL);
    if (!pages)
        return -ENOMEM;

    /* alloc_pages 批量分配——不使用 DMA 一致性内存 */
    ret = alloc_pages_bulk_array(GFP_KERNEL | __GFP_ZERO, nr_pages, pages);
    if (ret != nr_pages)
        goto out_free_pages;

    /* vmap 映射到内核虚拟地址空间 */
    vaddr = vmap(pages, nr_pages, VM_MAP, PAGE_KERNEL);
    if (!vaddr)
        goto out_free_pages;

    /* mmap 映射到用户态——持久映射，无需 page flipping */
    /* ...vm_area_struct 处理 + remap_pfn_range... */

    /* 注册到 io_uring registered buffer 表 */
    ret = io_uring_register_buffers(ring_fd, /* ... */);

out_free_pages:
    kvmfree(pages);
    return ret;
}
```

> **关键约束**：registered buffer 使用 `alloc_pages()` + `mmap()` 模式，**不使用 DMA 一致性内存**（`dma_alloc_coherent()`）。MemoryRovol L1 数据通过 userfaultfd 按需加载，无需设备 DMA 一致性保证。该约束源自 OLK 6.6 工程规范——DMA 一致性内存资源稀缺，仅在设备驱动场景使用。
>
> **OLK 6.6 smart grid 扩展说明**：OLK 6.6 的 `__alloc_pages_mpol(gfp, order, mpol, ilx, nid, use_smart_grid)` 相比 vanilla 主线多出第 6 个参数 `bool use_smart_grid`，这是 openEuler 特有扩展。agentrt-linux 不依赖此扩展，MemoryRovol 内核态使用标准 `alloc_pages()` / `alloc_pages_bulk_array()` 接口。

### 4.10 Badge 在 MemoryRovol 中的应用

MemoryRovol L1-L4 分层访问受 `sec_d` 颁发的 Badge 权限控制。`mem_d` 在执行 `airy_sys_rovol_ctl` 操作时，fastpath C-S9 内联校验 Badge Perms 位段：

| MemoryRovol 操作 | Badge Perms 位 | 校验位置 | 失败错误码 |
| --- | --- | --- | --- |
| L1 实时记忆读写 | `AIRY_CAP_PERM_ROVOL_L1_RW` | fastpath C-S9.PERMS 内联 | `-AIRY_ECAP_PERM = -81` |
| L2 短期记忆读写 | `AIRY_CAP_PERM_ROVOL_L2_RW` | fastpath C-S9.PERMS 内联 | `-AIRY_ECAP_PERM = -81` |
| L3 长期记忆只读 | `AIRY_CAP_PERM_ROVOL_L3_RO` | fastpath C-S9.PERMS 内联 | `-AIRY_ECAP_PERM = -81` |
| L4 持久记忆读写 | `AIRY_CAP_PERM_ROVOL_L4_RW` | fastpath C-S9.PERMS 内联 | `-AIRY_ECAP_PERM = -81` |
| 跨 tier 迁移 | `AIRY_CAP_PERM_ROVOL_MIGRATE` | slowpath airy_lsm 钩子 | `-AIRY_ECAP_PERM = -81` |
| 跨节点迁移 | `AIRY_CAP_PERM_ROVOL_MIGRATE_NODE` | slowpath airy_lsm + gateway_d | `-AIRY_ECAP_PERM = -81` |
| 快照/恢复 | `AIRY_CAP_PERM_ROVOL_SNAPSHOT` | fastpath C-S9.PERMS 内联 | `-AIRY_ECAP_PERM = -81` |

**Badge Epoch 失效场景**：

- `sec_d` 执行 `airy_cap_epoch_bump(agent_id)` 撤销时（K9-1 per-agent 主要机制），该 Agent 所有飞行中的 MemoryRovol 操作 fastpath C-S9.EPOCH 校验失败，返回 `-AIRY_ECAP_FROZEN = -82`（capability 冻结）。
- `mem_d` 收到 `-82` 后中止当前操作，通知 `macro_d` 重启受影响 Agent。
- 跨节点迁移时，`gateway_d` gossip 100ms 内同步补充性 `airy_cap_global_epoch`（UNFREEZE 用），确保 peer 节点 Badge 一致失效。

> **设计原则**：MemoryRovol 分层访问的 Badge 校验遵循"机制在内核（fastpath C-S9），策略在用户态（sec_d 编译 Badge）"原则（ADR-014 seL4 思想借鉴）。mem_d 不自行决定权限，仅执行 Badge 校验结果。

### 4.11 ADR 引用

| ADR | 标题 | 在本子仓的体现 |
| --- | --- | --- |
| [ADR-012](../10-architecture/05-adrs.md#adr-012) | 微内核化改造技术路线确认（基于 Linux 改造 + seL4 思想） | MemoryRovol 内核态实现基于 Linux 6.6 userfaultfd + MGLRU + CXL bus 改造，非从零实现 |
| [ADR-016](../10-architecture/05-adrs.md#adr-016) | 版本基线锁定（1.x.x 锁定 Linux 6.6） | OLK 6.6 MGLRU + userfaultfd + THP + CXL bus + ZONE_DEVICE + DAX |
| [ADR-014](../10-architecture/05-adrs.md#adr-014) | **微内核设计思想来源单一化（仅 seL4，不引入 Zircon/Minix3）** | **机制与策略分离原则（MemoryRovol 机制在内核，策略在 mem_d 用户态）；三独立数据区物理隔离借鉴 seL4 capability 空间隔离思想；Badge 分层访问控制借鉴 seL4 capability-based security；不引入 Zircon VMO rights/Minix3 多服务器模型** |

***

## 5. 微内核思想体现

### 5.1 记忆作为独立服务

遵循微内核"机制在内核，策略在用户态"原则（Liedtke minimality principle）：

- 内核提供 MemoryRovol 机制（snapshot、restore、migrate）\[SS]。
- 记忆管理策略（何时快照、何时迁移）在用户态 daemon（`macro_d`）\[SS]。
- L1-L4 数据结构 \[SC] 两端共享，确保记忆层级语义一致。

### 5.2 内存分层解耦

- 内存分层策略在用户态（与 `cognition` 协作）\[IND]。
- 内核仅提供分层机制（CXL、PMEM、DRAM tier）\[IND]。
- GFP 掩码 \[SC] 两端共享，统一分配语义。

### 5.3 最小内核介入

- userfaultfd 让用户态处理缺页，减少内核介入 \[SS]。
- DAX 模式绕过 page cache，减少内核介入 \[IND]。
- 符合微内核"最小化特权态代码"原则。

***

## 6. IRON-9 v3 四层共享模型落地

### 6.1 \[SC] 共享契约层——`include/uapi/linux/airymax/memory_types.h`

本头文件完全共享代码，agentrt 用户态与 agentrt-linux 内核态两端直接 include。内容清单（**以 SSoT 头文件实际定义为唯一依据**）：

| 内容 | 类别 | 说明 |
| --- | --- | --- |
| `enum airy_mem_level` | 枚举 | L1-L4 层级定义：`AIRY_MEM_HOT=0` / `AIRY_MEM_WARM=1` / `AIRY_MEM_COLD=2` / `AIRY_MEM_PMEM=3` / `AIRY_MEM_LEVEL_MAX` |
| `AIRY_GFP_HOT` / `AIRY_GFP_WARM` / `AIRY_GFP_COLD` / `AIRY_GFP_PMEM` | 宏 | 4 个 tier 分配标志（0x01/0x02/0x04/0x08） |
| `AIRY_PAGE_CLASS_ANON` / `AIRY_PAGE_CLASS_FILE` / `AIRY_PAGE_CLASS_SHMEM` / `AIRY_PAGE_CLASS_AGENT` | 宏 | 4 个页面分类标志（0x01/0x02/0x04/0x08） |
| `#ifdef AIRY_SC_FALLBACK` 降级块 | DSL 块 | L1-L4 降级为 L1 hot 单层 + GFP/page class 折叠（详见 §6.4） |

> **本节早期版本虚构内容已移除**（v1.0.1-fix）：原表曾列出 `airy_mr_l1_record_t` / `airy_mr_l2_block_t` / `airy_mr_l3_entry_t` / `airy_mr_l4_persistent_t` 四个结构体、`AIRY_GFP_*` 7 个仿 Linux 内核 GFP 标志、`airy_pmem_ops_t` 持久化接口结构，**这些均不在 SSoT 头文件中实际定义**，已全部移除。

### 6.2 \[SS] 语义同源层——6 项 API 映射

操作模式同源（概念操作一致），函数签名因抽象层级不同而独立：

| 序号 | API                  | 语义     | agentrt 实现   | agentrt-linux 实现         |
| -- | -------------------- | ------ | ------------ | ------------------------ |
| 1  | `rovol_snapshot()`   | 创建记忆快照 | 用户态 fork+COW | 内核 fork+userfaultfd      |
| 2  | `rovol_restore()`    | 恢复记忆   | 用户态 mmap     | 内核 mmap+userfaultfd      |
| 3  | `rovol_migrate()`    | 迁移记忆   | 用户态迁移        | 内核 userfaultfd post-copy |
| 4  | `rovol_compress()`   | 压缩快照   | 用户态 zstd     | 内核 zswap/zram            |
| 5  | MGLRU aging/eviction | 代际回收语义 | 用户态模拟        | 内核 `lru_gen_folio`       |
| 6  | userfaultfd 处理       | 缺页处理   | 用户态 handler  | 内核 uffd 框架               |

### 6.3 \[IND] 完全独立层——5 项独立实现

| 序号 | 内容            | 不共享原因                          |
| -- | ------------- | ------------------------------ |
| 1  | CXL 设备驱动      | 硬件驱动仅 agentrt-linux 内核态        |
| 2  | PMEM 设备驱动     | nvdimm 驱动仅 agentrt-linux 内核态   |
| 3  | VFS 持久化层实现    | 文件系统后端仅 agentrt-linux          |
| 4  | THP 优化实现      | khugepaged 仅 agentrt-linux 内核态 |
| 5  | zswap/zram 集成 | 内核压缩框架仅 agentrt-linux          |

### 6.4 \[DSL] 降级生存层——`#ifdef AIRY_SC_FALLBACK` 降级块

依据 IRON-9 v3 决策，`memory_types.h` 底部存在 `#ifdef AIRY_SC_FALLBACK` 降级块，保证 \[SC] 共享契约在内核态 \[SC] 头文件不可用或被裁剪时仍能维持降级生存模式。降级块内容**以 SSoT 头文件实际定义为唯一依据**：

```c
/* 来源：include/uapi/linux/airymax/memory_types.h */
#ifdef AIRY_SC_FALLBACK
	/* All tiers collapse to L1 hot. */
	#define AIRY_DSL_MEM_LEVEL   AIRY_MEM_HOT
	#define AIRY_DSL_MEM_TIERS   1  /* Only L1 retained */

	/* All GFP flags collapse to HOT. */
	#define AIRY_DSL_GFP_HOT     AIRY_GFP_HOT
	#define AIRY_DSL_GFP_WARM    AIRY_GFP_HOT
	#define AIRY_DSL_GFP_COLD    AIRY_GFP_HOT
	#define AIRY_DSL_GFP_PMEM    AIRY_GFP_HOT

	/* All page classes collapse to ANON. */
	#define AIRY_DSL_PAGE_CLASS_ANON   AIRY_PAGE_CLASS_ANON
	#define AIRY_DSL_PAGE_CLASS_FILE   AIRY_PAGE_CLASS_ANON
	#define AIRY_DSL_PAGE_CLASS_SHMEM  AIRY_PAGE_CLASS_ANON
	#define AIRY_DSL_PAGE_CLASS_AGENT  AIRY_PAGE_CLASS_ANON

	#warning "AIRY_SC_FALLBACK active: memory_types.h degraded to L1 hot tier only, MemoryRoVol L2-L4 unavailable"
#endif /* AIRY_SC_FALLBACK */
```

依据 SSoT 实际 DSL 块，记忆子仓的 \[DSL] 降级策略：

| 序号 | 降级项 | 正常模式（SSoT 定义） | \[DSL] 降级模式（SSoT `AIRY_SC_FALLBACK` 块） |
| -- | ---- | ------ | ------------- |
| 1 | L1-L4 tier 枚举 | `airy_mem_level` 完整枚举（HOT/WARM/COLD/PMEM） | `AIRY_DSL_MEM_LEVEL = AIRY_MEM_HOT`，`AIRY_DSL_MEM_TIERS = 1`（仅 L1 保留） |
| 2 | GFP tier 选择标志 | `AIRY_GFP_HOT/WARM/COLD/PMEM` 4 标志 | `AIRY_DSL_GFP_*` 全部折叠为 `AIRY_GFP_HOT`（仅热层级分配） |
| 3 | 页面分类标志 | `AIRY_PAGE_CLASS_ANON/FILE/SHMEM/AGENT` 4 标志 | `AIRY_DSL_PAGE_CLASS_*` 全部折叠为 `AIRY_PAGE_CLASS_ANON`（仅匿名页） |
| 4 | 编译期告警 | 无 | `#warning` 提示 `MemoryRoVol L2-L4 unavailable` |
| 5 | MemoryRovol 实现 | 内核态实现 + CXL/PMEM tier 分层 | 降级为 agentrt 用户态 heapstore 实现（仅 DRAM 单层）——属 \[IND] 层降级，不在 SSoT 中 |
| 6 | `alloc_pages` + `mmap` | 内核 registered buffer + 零拷贝 | 仍可使用，仅 tier 标志退化为 HOT——属 \[IND] 层降级 |

> **\[DSL] 设计原则**：降级生存不等于功能完整——SSoT DSL 块仅折叠 \[SC] 共享契约层的 tier/GFP/page class 标志至 L1 hot 单层；CXL/PMEM tier 退化、零拷贝退化为拷贝等 \[IND] 层降级由各端独立处理。降级模式下性能显著下降，但保证 agentrt 用户态代码可在不支持 agentrt-linux 内核的平台上运行（兼容性优先）。详见 [DSL] §2.2 和 §4.1（SSoT 头文件注释引用）。

### 6.5 跨态协作流

```mermaid
sequenceDiagram
    participant AGENT as Agent 进程
    participant ROV_U as agentrt MemoryRovol (用户态)
    participant IPC as io_uring / AgentsIPC
    participant ROV_K as agentrt-linux MemoryRovol kthread (内核态)
    participant MGLRU as MGLRU 回收
    participant PMEM as PMEM/CXL 设备

    AGENT->>ROV_U: 发起记忆操作
    ROV_U->>ROV_U: 用户态 L1/L2 操作 [SC] memory_types.h
    ROV_U->>IPC: 提交快照/迁移请求 [SS]
    IPC->>ROV_K: io_uring 提交
    ROV_K->>ROV_K: 内核态 L3/L4 操作 [IND]
    ROV_K->>MGLRU: aging/eviction [SS]
    MGLRU->>PMEM: 回收至 CXL/PMEM tier [IND]
    PMEM-->>MGLRU: 完成
    MGLRU-->>ROV_K: 回收结果
    ROV_K-->>IPC: 返回
    IPC-->>ROV_U: 返回
    ROV_U-->>AGENT: 记忆操作结果
```

***

## 7. agentrt-linux 工程基线

- **agentrt-linux 内存管理**：MGLRU、CXL、THP 等特性贡献。
- **agentrt-linux 内存分层**：内存分层策略基线。
- **agentrt-linux PMEM**：持久化内存支持。
- **agentrt-linux CXL**：CXL 设备支持。
- **Linux 6.6 内核基线**：MGLRU + userfaultfd + THP + CXL bus + ZONE\_DEVICE + DAX。

### 7.1 五维正交 24 原则映射

| 原则                      | 在本模块的体现                                |
| ----------------------- | -------------------------------------- |
| **E-1 安全内生**            | 记忆加密 + PMEM 完整性校验 + TEE 保护             |
| **K-3 服务隔离**            | MemoryRovol 独立 kthread + memcg 隔离      |
| **K-4 可插拔策略**           | 内存分层策略可配置 + VFS 后端可插拔                  |
| **IRON-9 v3 同源且部分代码共享** | \[SC] 共享契约层 + \[SS] 语义同源层 + \[IND] 独立层 |
| **A-4 完美主义**            | CXL 内存池化 + PMEM 持久化 + MGLRU 多代回收       |

***

## 8. 前沿理论参考

| 理论                           | 来源              | 应用                     | 同源标注   |
| ---------------------------- | --------------- | ---------------------- | ------ |
| Liedtke minimality principle | Liedtke SOSP'95 | 微内核最小化原则——机制在内核，策略在用户态 | \[SS]  |
| CXL 3.0                      | CXL Consortium  | 内存分层与池化                | \[IND] |
| PMEM                         | Intel           | 持久化内存                  | \[IND] |
| MGLRU                        | Linux 6.6       | 多代 LRU 回收——代际模型        | \[SS]  |
| userfaultfd                  | Linux 4.x+      | 用户态缺页处理                | \[SS]  |
| Linux 6.6 THP                | Linux 6.6       | 透明巨页                   | \[IND] |
| DAX                          | Linux           | 直接访问模式                 | \[IND] |
| zswap/zram                   | Linux           | 内存压缩                   | \[IND] |
| MemoryRovol                  | agentrt         | 记忆卷载——L1-L4 分层         | \[SC]  |

***

## 9. 与其他子仓的协作

### 9.1 与子仓协作

| 协作子仓          | 协作内容                           | 同源标注           |
| ------------- | ------------------------------ | -------------- |
| `kernel`      | 提供 MemoryRovol、CXL、MGLRU 内核实现  | \[SS] + \[IND] |
| `services`    | 提供 VFS 持久化层、MemoryRovol 用户态服务  | \[SS]          |
| `security`    | 提供记忆加密、TEE 保护                  | \[IND]         |
| `cognition`   | 提供 Agent 记忆管理、CoreLoopThree 协作 | \[SS]          |
| `cloudnative` | 提供容器记忆卷载、迁移                    | \[IND]         |
| `system`      | 提供内存监控工具                       | \[SS]          |
| `tests-linux` | 内存测试、Soak Test                 | \[SS]          |

### 9.2 与 12 daemon 协作

AirymaxOS 用户态 **12 daemon**（daemon 命名后缀统一为 `_d`，**无例外**，v2.0 决策 C1；12 daemon 完整名单以 [10-user-supervisor-daemon.md §1.3](10-user-supervisor-daemon.md) 为 SSoT）与记忆子仓的协作关系（详见 [01-kernel.md §14.2](01-kernel.md)）：

| Daemon | 职责 | 与记忆子仓的协作 | 数据通道 |
| --- | --- | --- | --- |
| `mem_d` | 记忆卷载管理（MemoryRovol L1-L4） | **记忆子仓核心 daemon**——通过 `airy_sys_rovol_ctl`（编号 1）控制记忆卷载 | io_uring + userfaultfd |
| `sec_d` | capability 编译/撤销 + Badge 生命周期 | 为 mem_d 颁发 `AIRY_CAP_PERM_ROVOL_*` Badge 权限位；fastpath C-S9 校验 | `airy_sys_call`（编号 0）+ agent_caps[] |
| `cogn_d` | 认知循环调度（CoreLoopThree） | CoreLoopThree 各阶段切换时同步记忆状态（perception→thinking→action） | `airy_sys_clt_notify`（编号 3） |
| `gateway_d` | 跨节点 IPC | 跨节点记忆迁移通过 `gateway_d` 转发 + gossip 100ms Epoch 同步 | io_uring + gRPC/QUIC |
| `logger_d` | 统一日志（128B 记录） | 接收 MemoryRovol 操作日志（快照/恢复/迁移事件） | char dev `/dev/airy_log` |
| `macro_d` | 宏观监管 | 监控 mem_d 心跳（systemd watchdog），Badge Epoch 失效时重启 Agent | systemd watchdog |
| `audit_d` | 审计哈希链 | 审计 MemoryRovol 跨 tier/跨节点迁移操作 | eBPF ringbuf |
| `sched_d` | sched_tac 策略守护 | 记忆迁移感知——迁移期间调整调度优先级 | `airy_sys_sched_ctl`（编号 2） |
| `dev_d` | 设备驱动用户态化 | CXL/PMEM 设备通过 dev_d 用户态驱动注册 | io_uring + VFIO |
| `net_d` | 网络栈用户态化 | 跨节点记忆迁移数据传输 | io_uring + VFIO |
| `vfs_d` | VFS 用户态化 | MemoryRovol 快照通过 vfs_d 持久化至文件系统 | io_uring + VFIO |
| `config_d` | 统一配置管理 | MGLRU/CXL tier/THP 等运行时参数热更新 | sysfs + procfs |

> **mem_d 是 12 daemon 的记忆核心**：所有记忆卷载操作（快照/恢复/迁移/压缩）均经 `mem_d` 通过 `airy_sys_rovol_ctl`（编号 1）处理。mem_d 与 sec_d 协作完成 Badge 权限校验，与 cogn_d 协作完成 CoreLoopThree 阶段同步，与 gateway_d 协作完成跨节点迁移。

***

## 10. 里程碑（M0-M8）

| 阶段 | 目标                                           | 时间      | 同源标注   |
| -- | -------------------------------------------- | ------- | ------ |
| M0 | 设计基线锁定 + 工程标准就绪 + 内核改造框架就位（本模块设计文档）                              | 2026-07 | —      |
| M1 | \[SC] `include/uapi/linux/airymax/memory_types.h` 共享契约层 | 2026 Q3 | \[SC]  |
| M2 | MemoryRovol 内核态实现 + L1-L4 层级枚举（基于 `airy_mem_level`）| 2026 Q3 | \[SS]  |
| M3 | MGLRU 集成 + aging/eviction 策略                 | 2026 Q4 | \[SS]  |
| M4 | userfaultfd 缺页处理框架 + 迁移                      | 2026 Q4 | \[SS]  |
| M5 | PMEM 持久化内存支持 + DAX                           | 2027 Q1 | \[IND] |
| M6 | CXL 内存分层（Phase 1）+ tiering                   | 2027 Q1 | \[IND] |
| M7 | CXL 内存池化 + VFS 持久化层                          | 2027 Q2 | \[IND] |
| M8 | THP 优化 + zswap/zram 集成                       | 2027 Q2 | \[IND] |

### 10.1 0.1.1 版本范围

仅完成 M0（设计基线锁定 + 工程标准就绪 + 内核改造框架就位）+ M1（\[SC] 共享契约层头文件占位）。不含内核/OS 代码实施。

### 10.2 1.0.1 版本范围

完成 M2-M8 全部里程碑，并实施记忆工程标准。

***

## 11. agentrt 一致性检查

对 agentrt heapstore + memoryrovol 设计进行一致性检查，确认两端在 IRON-9 v3 四层共享模型下无冲突（**以 SSoT 头文件 `include/uapi/linux/airymax/memory_types.h` 实际定义为唯一依据**）：

| 序号 | 检查项 | agentrt 状态 | agentrt-linux 状态 | 结论 |
| -- | --- | --- | --- | --- |
| 1  | `airy_mem_level` 枚举一致性 | `AIRY_MEM_HOT/WARM/COLD/PMEM/LEVEL_MAX` | 同（直接 include SSoT） | PASS \[SC] |
| 2  | L1-L4 层级语义一致性 | HOT=L1 / WARM=L2 / COLD=L3 / PMEM=L4 | 同 | PASS \[SC] |
| 3  | `AIRY_GFP_HOT/WARM/COLD/PMEM` 4 标志一致性 | 0x01/0x02/0x04/0x08 | 同（直接 include SSoT） | PASS \[SC] |
| 4  | `AIRY_PAGE_CLASS_ANON/FILE/SHMEM/AGENT` 4 标志一致性 | 0x01/0x02/0x04/0x08 | 同（直接 include SSoT） | PASS \[SC] |
| 5  | `AIRY_SC_FALLBACK` DSL 块一致性 | tier 折叠至 HOT / page class 折叠至 ANON | 同 | PASS \[DSL] |
| 6  | `rovol_snapshot()` 语义等价性 | 用户态 fork+COW | 内核 fork+userfaultfd | PASS \[SS] |
| 7  | `rovol_restore()` 语义等价性 | 用户态 mmap | 内核 mmap+userfaultfd | PASS \[SS] |
| 8  | `rovol_migrate()` 语义等价性 | 用户态迁移 | 内核 userfaultfd post-copy | PASS \[SS] |
| 9  | `rovol_compress()` 语义等价性 | 用户态 zstd | 内核 zswap/zram | PASS \[SS] |
| 10 | MGLRU aging/eviction 语义一致性 | 用户态模拟 | 内核 `lru_gen_folio` | PASS \[SS] |
| 11 | userfaultfd 处理语义等价性 | 用户态 handler | 内核 uffd 框架 | PASS \[SS] |
| 12 | CXL/PMEM/VFS 独立性 | 不实现 | 内核态实现 | PASS \[IND] |
| 13 | THP/zswap 独立性 | 不实现 | 内核态实现 | PASS \[IND] |
| 14 | MGLRU Bloom 过滤器独立性 | 不使用 | 内核态使用（可选优化） | PASS \[IND] |

> **本节早期版本虚构检查项已移除**（v1.0.1-fix）：原表曾列出对 `airy_mr_l1_record_t` / `airy_mr_l2_block_t` / `airy_mr_l3_entry_t` / `airy_mr_l4_persistent_t` 四个虚构结构体的一致性检查（条目 1-4）、对 7 个虚构 `AIRY_GFP_*` 标志的检查（条目 5）、对虚构 `airy_pmem_ops_t` 接口的检查（条目 6）——这些类型**均不在 SSoT 头文件中实际定义**，相关检查项已全部移除并替换为对 SSoT 实际定义内容的检查。

**结论**：agentrt heapstore + memoryrovol 设计无需修改。14 项检查全部 PASS，两端在 \[SC]/\[SS]/\[IND]/\[DSL] v3 四层共享模型下完全一致。

***

## 12. 相关文档

- `40-dataflows/02-memory-flow.md`（记忆卷载数据流设计）
- `50-engineering-standards/01-coding-standards.md`（记忆编码规范）
- `80-testing/` 内存测试文档
- `90-observability/README.md`（内存监控）
- [01-kernel.md §14.2](01-kernel.md)（12 daemon 内核切入点）
- [03-security.md](03-security.md)（Badge 权限 + sec_d 协作 + 记忆加密）
- [30-interfaces/01-syscalls.md](../30-interfaces/01-syscalls.md)（`airy_sys_rovol_ctl` 编号 1 + 24 槽位 syscall 表）
- [30-interfaces/07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md)（io_uring SQE128 模式 + `io_uring_cmd_to_pdu()` + `io_uring_cmd_done()` 4 参数）
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)（`AIRY_ECAP_FROZEN = -82` + `AIRY_ECAP_PERM = -81`）
- [10-architecture/05-adrs.md#adr-012](../10-architecture/05-adrs.md)（ADR-012 微内核化改造技术路线）
- [10-architecture/05-adrs.md#adr-014](../10-architecture/05-adrs.md)（ADR-014 微内核设计思想来源单一化——仅 seL4）
- agentrt heapstore + memoryrovol 设计文档（同源 \[SC]/\[SS]）

***

## 13. 参考

- **ADR-014: 微内核设计思想来源单一化（仅 seL4，不引入 Zircon/Minix3）**——机制与策略分离 + capability-based security 思想来源 SSoT，详见 [10-architecture/05-adrs.md#adr-014](../10-architecture/05-adrs.md)
- **ADR-012: 微内核化改造技术路线确认（基于 Linux 改造 + seL4 思想，非从零开发）**——MemoryRovol 内核态实现基于 Linux 6.6 改造的定位依据
- CXL 3.0 规格（CXL Consortium）
- Linux 6.6 MGLRU 文档（`mm/vmscan.c` + `include/linux/mmzone.h`）
- Linux 6.6 THP 文档
- Linux 6.6 userfaultfd 文档（`mm/userfaultfd.c`）
- Linux 6.6 DAX 文档（`fs/dax.c`）
- Linux 6.6 CXL bus 文档（`drivers/cxl/`）
- Linux 6.6 ZONE\_DEVICE 文档（`include/linux/mmzone.h`）
- Linux 6.6 `include/uapi/linux/io_uring.h`（`io_uring_cmd_to_pdu()` 安全宏 + `io_uring_cmd_done()` 4 参数 + `IORING_SETUP_SQE128` + registered buffer 接口）
- Linux 6.6 `mm/page_alloc.c`（`alloc_pages()` + `alloc_pages_bulk_array()` 接口）
- PMEM 文档（Intel）
- agentrt-linux 内存管理文档
- agentrt heapstore + memoryrovol 设计文档
- Liedtke SOSP'95（微内核最小化原则，seL4 思想来源之一，ADR-014）

***

## 14. 变更历史

| 版本 | 日期 | 变更内容 |
| --- | --- | --- |
| v1.0.1 | 2026-07 | 初始版本——MemoryRovol 内核态设计、IRON-9 v3 四层共享模型落地、12 daemon 协作 |
| v1.0.1-fix | 2026-07-26 | **SSoT 对齐修复**——对照 `include/uapi/linux/airymax/memory_types.h` 实际定义，移除文档中所有虚构内容，恢复 SSoT 单一真相源地位。具体修复如下： |

### 14.1 v1.0.1-fix (2026-07-26) 修复详情

**SSoT 头文件实际定义清单**（`include/uapi/linux/airymax/memory_types.h`）：

- `enum airy_mem_level`：L1-L4 层级枚举（`AIRY_MEM_HOT/WARM/COLD/PMEM/LEVEL_MAX`）
- `AIRY_GFP_HOT/WARM/COLD/PMEM` 4 个 tier 分配标志宏（0x01/0x02/0x04/0x08）
- `AIRY_PAGE_CLASS_ANON/FILE/SHMEM/AGENT` 4 个页面分类标志宏（0x01/0x02/0x04/0x08）
- `#ifdef AIRY_SC_FALLBACK` 降级块（tier 折叠至 HOT、page class 折叠至 ANON、`#warning` 告警）

**移除的虚构内容**：

1. **虚构 L1-L4 Record 结构体**（原 §4.1，行 225-261）：移除虚构的 `airy_mr_l1_record_t` / `airy_mr_l2_block_t` / `airy_mr_l3_entry_t` / `airy_mr_l4_persistent_t` 四个结构体——SSoT 头文件中并不存在这些类型，L1-L4 仅作为 `airy_mem_level` 枚举的层级标识符存在。替换为 SSoT 实际定义的 `airy_mem_level` 枚举与层级语义映射表，并明确标注 L1-L4 各层级具体记忆记录的数据结构属 \[SS]/\[IND] 层独立定义。

2. **虚构 `airy_pmem_ops_t` 持久化接口**（原 §4.3，行 296-304）：移除虚构的 `airy_pmem_ops_t` 结构体（含 persist/flush/map/unmap 四函数指针）——SSoT 头文件中并无任何 PMEM 操作回调接口。替换为 SSoT 中 PMEM 相关的实际定义（`AIRY_MEM_PMEM` 枚举值 + `AIRY_GFP_PMEM` 分配标志），并明确标注 PMEM 设备驱动与持久化接口实现属 \[IND] 独立层。

3. **虚构 7 个仿 Linux GFP 标志**（原 §4.4，行 322-331）：移除虚构的 `AIRY_GFP_IO` / `AIRY_GFP_FS` / `AIRY_GFP_RECLAIM` / `AIRY_GFP_KSWAPD` / `AIRY_GFP_HIGH` / `AIRY_GFP_NOWARN` / `AIRY_GFP_ZERO` 等 7 个仿 Linux 内核 GFP 标志——这些标志属 Linux 内核原生 GFP 体系（`include/linux/gfp.h`），不在 `memory_types.h` 共享契约层中重复定义。替换为 SSoT 实际定义的 4 个 tier 选择标志，并补充 SSoT 中另定义的 4 个 `AIRY_PAGE_CLASS_*` 页面分类宏。

4. **虚构 §6.1 内容清单**：原表曾列出虚构结构体与接口条目，已全部移除并替换为 SSoT 头文件实际定义的 4 类内容清单。

5. **虚构 §6.4 DSL 降级表**：原表描述的"MemoryRovol L1-L4 数据结构降级为 heapstore"、"PMEM 持久化接口降级为 fsync"、"MGLRU/userfaultfd/io_uring 降级" 等均不属 SSoT DSL 块内容——SSoT DSL 块仅折叠 \[SC] 共享契约层的 tier/GFP/page class 标志至 L1 hot 单层。替换为 SSoT 实际 DSL 块代码与降级策略表。

6. **虚构 §11 一致性检查**：原表条目 1-6 全部基于虚构结构/接口，已移除并替换为对 SSoT 实际定义内容（`airy_mem_level` 枚举、`AIRY_GFP_*` 标志、`AIRY_PAGE_CLASS_*` 标志、`AIRY_SC_FALLBACK` DSL 块）的检查。

7. **虚构 §3.8 mermaid 图节点**：原图节点 `L1/L2/L3/L4 Record 数据结构` 与 `PMEM 持久化接口` 已移除，替换为 SSoT 实际节点（`airy_mem_level` 枚举 / `AIRY_GFP_*` 标志 / `AIRY_PAGE_CLASS_*` 标志 / `AIRY_SC_FALLBACK` 降级块）。

8. **散落虚构引用**：修正 §1 子仓职责、§2 维度对比表、§2.2 同源传承要点、§3.1/§3.3/§3.4 节描述、§10 里程碑 M2 等位置对 "L1-L4 数据结构" / "PMEM 持久化接口 [SC]" / "GFP 掩码语义" 等虚构表述。

**修复原则**：

- 所有 SSoT 实际定义的代码示例均明确标注来源 `/* 来源：include/uapi/linux/airymax/memory_types.h */`
- 每处修复位置添加 `> **SSoT 声明**` 引用块，明确标注 SSoT 头文件路径
- 说明性示例（非 SSoT 定义）明确标注 "**说明性示例，非 SSoT 定义**"
- 虚构类型完全移除，不在文档中保留为"示例"
- 保留文档教学性质，但所有 SSoT 引用必须与头文件实际定义一致

***

> **文档结束** | v1.0.1-fix | IRON-9 v3 同源且部分代码共享 | 记忆卷载贯穿 4 大数据流 | 0.1.1 = 文档体系完成 | SSoT 对齐修复对照 `include/uapi/linux/airymax/memory_types.h`

