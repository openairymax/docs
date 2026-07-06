# AirymaxOS 记忆设计文档（airymaxos-memory，极境记忆）

> 子仓编号：04
> 子仓代号：极境记忆（Airymax Memory）
> 文档版本：v1.0（2026-07-06）
> 设计基准：记忆持久化 + CXL + PMEM
> 同源 agentrt：heapstore + memoryrovol（MemoryRovol）

---

## 1. 子仓职责

`airymaxos-memory` 是 AirymaxOS 的记忆与存储子仓，承担以下核心职责：

1. **MemoryRovol 内核态实现**：将 agentrt 的 MemoryRovol（记忆卷载）升级为内核态实现，提供 Agent 记忆的持久化与卷载能力。
2. **CXL 内存分层与池化**：利用 2026 年 CXL 3.0 硬件普及，实现内存分层与池化。
3. **持久化内存（PMEM）**：基于 PMEM 提供非易失性内存支持。
4. **MGLRU**：利用 Linux 6.6 多代 LRU 改进，优化内存回收策略。
5. **VFS 持久化层**：为 `airymaxos-services/vfs` 提供持久化后端。
6. **userfaultfd 用户态缺页处理**：支持用户态缺页处理，用于记忆迁移与快照。
7. **透明巨页（THP）优化**：利用 Linux 6.6 THP 改进提升大页性能。

---

## 2. 同源关系

| 维度 | agentrt（heapstore + memoryrovol） | AirymaxOS（airymaxos-memory） |
|------|-----------------------------------|------------------------------|
| 记忆存储 | heapstore（用户态） | MemoryRovol 内核态 + heapstore 用户态 |
| 记忆卷载 | MemoryRovol（用户态） | MemoryRovol 内核态实现 |
| 持久化 | 文件系统 | PMEM + CXL + VFS 持久化层 |
| 分层 | 用户态分层 | CXL 内存分层 + MGLRU |

**同源传承要点**：
- 保留 agentrt MemoryRovol 的"记忆卷载"语义（snapshot + restore）。
- 保留 heapstore 的"记忆存储"抽象。
- 升级为内核态实现，利用 CXL/PMEM 硬件加速。

---

## 3. 目录结构

```
airymaxos-memory/
├── memoryrovol/            # MemoryRovol 内核态实现（记忆卷载）
├── cxl/                    # CXL 内存分层与池化
├── pmem/                   # 持久化内存
├── mglru/                  # MGLRU（Linux 6.6 多代 LRU）
├── vfs-persist/            # VFS 持久化层
├── userfaultfd/           # 用户态缺页处理
├── thp/                    # 透明巨页优化
└── docs/
```

### 3.1 memoryrovol/（MemoryRovol 内核态实现）

参考 agentrt MemoryRovol 设计：
- `rovol-kmod`：内核模块，提供记忆卷载系统调用。
- `snapshot`：记忆快照（基于 fork + COW）。
- `restore`：记忆恢复（基于 mmap + userfaultfd）。
- `migrate`：记忆迁移（跨节点、跨 CXL 设备）。
- `compress`：记忆压缩（zswap、zram 集成）。
- `encrypt`：记忆加密（与 `airymaxos-security` 协作）。

### 3.2 cxl/（CXL 内存分层与池化）

基于 **CXL 3.0** 规格：
- `cxl-type2`：CXL Type 2 设备支持（缓存一致内存）。
- `cxl-type3`：CXL Type 3 设备支持（内存扩展）。
- `tiering`：内存分层策略（FAST/SSLOW/CXL tier）。
- `pooling`：内存池化（跨节点共享）。
- `hotplug`：CXL 内存热插拔。

### 3.3 pmem/（持久化内存）

- `pmem-driver`：PMEM 设备驱动（nvdimm）。
- `dax`：DAX（Direct Access）模式，绕过 page cache。
- `fsdax`：文件系统 DAX（ext4-dax、xfs-dax）。
- `devdax`：设备 DAX（字符设备模式）。

### 3.4 mglru/（MGLRU）

利用 **Linux 6.6** MGLRU 改进：
- `multi-gen-lru`：多代 LRU 回收策略。
- ` aging`：老化策略（按代标记页面）。
- `eviction`：逐出策略（按代逐出）。
- `workingset-protection`：工作集保护。

### 3.5 vfs-persist/（VFS 持久化层）

为 `airymaxos-services/vfs` 提供持久化后端：
- `backends/`：后端实现（PMEM、CXL、SSD、HDD）。
- `journal`：日志系统（WAL）。
- `snapshot`：文件系统快照。
- `dedup`：去重。
- `compress`：压缩（zstd、lz4）。

### 3.6 userfaultfd/（用户态缺页处理）

- `uffd-handler`：用户态缺页处理框架。
- `live-migration`：进程实时迁移。
- `snapshot`：进程快照（与 MemoryRovol 协作）。
- `postcopy`：post-copy 迁移策略。

### 3.7 thp/（透明巨页优化）

利用 **Linux 6.6** THP 改进：
- `hugepages`：大页分配策略。
- `khugepaged`：大页合并守护进程。
- `madvise`：madvise 策略（MADV_HUGEPAGE）。
- `shmem`：shmem 大页支持。

---

## 4. 核心特性

### 4.1 MemoryRovol 内核态实现（同源）

**记忆卷载语义**：
- `rovol_snapshot(pid)`：对指定进程创建记忆快照。
- `rovol_restore(snapshot_id)`：从快照恢复记忆。
- `rovol_migrate(pid, target_node)`：迁移进程记忆至目标节点。
- `rovol_compress(snapshot_id)`：压缩快照。

**实现机制**：
- 快照基于 fork + COW（用户空间）或 fork + userfaultfd（内核空间）。
- 恢复基于 mmap + userfaultfd 按需加载。
- 迁移基于 userfaultfd post-copy。

**与 agentrt 同源**：保留 agentrt MemoryRovol 的 API 兼容性，但底层实现升级为内核态。

### 4.2 CXL 内存分层与池化（2026 硬件普及）

**CXL 3.0 规格**（2026 硬件普及）：
-  cache coherent：缓存一致性，简化编程模型。
- multi-host：多主机共享内存池。
- switching：CXL switch 支持复杂拓扑。

**分层策略**：
| Tier | 设备 | 延迟 | 用途 |
|------|------|------|------|
| FAST | DRAM | ~100ns | 热数据 |
| CXL | CXL memory | ~200ns | 温数据 |
| PMEM | 持久内存 | ~300ns | 持久化 |
| SSD | NVMe SSD | ~10μs | 冷数据 |

**池化**：
- 多节点共享 CXL 内存池。
- 动态分配/释放内存至不同节点。
- 故障切换（节点宕机时内存迁移）。

### 4.3 PMEM 持久化内存

**特性**：
- 非易失性：断电后数据保留。
- 字节寻址：像内存一样访问。
- 低延迟：~300ns（比 SSD 快 30 倍）。

**应用**：
- Agent 记忆持久化（MemoryRovol 后端）。
- 文件系统元数据（DAX 模式）。
- 日志系统（WAL）。

### 4.4 MGLRU（Linux 6.6 多代 LRU）

**改进**：
- 多代回收：页面按代分组，按代逐出。
- 工作集保护：识别并保护活跃工作集。
- 更优的内存压力应对。

**配置**：
```
echo y > /sys/kernel/mm/lru_gen/enabled
echo 1000 > /sys/kernel/mm/lru_gen/max_seq
```

### 4.5 VFS 持久化层

**多后端支持**：
- PMEM：高性能持久化。
- CXL：可共享持久化。
- SSD：大容量持久化。
- HDD：归档持久化。

**特性**：
- 写前日志（WAL）：保证崩溃一致性。
- 快照：文件系统级快照。
- 去重：块级去重。
- 压缩：zstd/lz4 压缩。

### 4.6 userfaultfd 用户态缺页处理

**用例**：
- 进程实时迁移：将进程从一节点迁移至另一节点。
- 进程快照：创建进程记忆快照。
- 按需加载：仅在访问时加载页面。
- 惰性恢复：从快照惰性恢复。

**API**：
```c
struct uffdio_api api = { .api = UFFD_API };
ioctl(uffd, UFFDIO_API, &api);
// 注册缺页处理区域
struct uffdio_register reg = {
    .range = { .start = addr, .len = size },
    .mode = UFFDIO_REGISTER_MODE_MISSING,
};
ioctl(uffd, UFFDIO_REGISTER, &reg);
```

### 4.7 透明巨页（THP）优化（Linux 6.6）

**Linux 6.6 改进**：
- 更激进的 khugepaged 合并策略。
- shmem 大页支持改进。
- madvise 行为更可预测。
- 减少 THP 抖动。

**配置**：
```
echo always > /sys/kernel/mm/transparent_hugepage/enabled
echo madvise > /sys/kernel/mm/transparent_hugepage/shmem_enabled
```

---

## 5. 微内核思想体现

### 5.1 记忆作为独立服务

遵循微内核"机制在内核，策略在用户态"原则：
- 内核提供 MemoryRovol 机制（snapshot、restore、migrate）。
- 记忆管理策略（何时快照、何时迁移）在用户态 daemon（`memoryrovol_d`）。

### 5.2 内存分层解耦

- 内存分层策略在用户态（与 `airymaxos-cognition` 协作）。
- 内核仅提供分层机制（CXL、PMEM、DRAM tier）。

### 5.3 最小内核介入

- userfaultfd 让用户态处理缺页，减少内核介入。
- DAX 模式绕过 page cache，减少内核介入。

---

## 6. AirymaxOS 工程基线

- **AirymaxOS 内存管理**：MGLRU、CXL、THP 等特性贡献。
- **AirymaxOS 内存分层**：内存分层策略基线。
- **AirymaxOS PMEM**：持久化内存支持。
- **AirymaxOS CXL**：CXL 设备支持。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| CXL 3.0 | CXL Consortium | 内存分层与池化 |
| PMEM | Intel | 持久化内存 |
| MGLRU | Linux 6.6 | 多代 LRU 回收 |
| userfaultfd | Linux 4.x+ | 用户态缺页处理 |
| Linux 6.6 THP | Linux 6.6 | 透明巨页 |
| DAX | Linux | 直接访问模式 |
| zswap/zram | Linux | 内存压缩 |
| MemoryRovol | agentrt | 记忆卷载 |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-kernel` | 提供 MemoryRovol、CXL、MGLRU 内核实现 |
| `airymaxos-services` | 提供 VFS 持久化层、MemoryRovol 用户态服务 |
| `airymaxos-security` | 提供记忆加密、TEE 保护 |
| `airymaxos-cognition` | 提供 Agent 记忆管理、CoreLoopThree 协作 |
| `airymaxos-cloudnative` | 提供容器记忆卷载、迁移 |
| `airymaxos-system` | 提供内存监控工具 |
| `airymaxos-tests` | 内存测试、Soak Test |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | MemoryRovol 内核态实现 | 2026 Q3 |
| M2 | MGLRU 集成 + THP 优化 | 2026 Q4 |
| M3 | userfaultfd 缺页处理框架 | 2027 Q1 |
| M4 | PMEM 持久化内存支持 | 2027 Q2 |
| M5 | CXL 内存分层（Phase 1） | 2027 Q3 |
| M6 | CXL 内存池化 + VFS 持久化层 | 2027 Q4 |

---

## 10. 参考

- CXL 3.0 规格（CXL Consortium）
- Linux 6.6 MGLRU 文档
- Linux 6.6 THP 文档
- userfaultfd 文档
- DAX 文档
- PMEM 文档（Intel）
- AirymaxOS 内存管理文档
- agentrt heapstore + memoryrovol 设计文档
