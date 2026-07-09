Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# Airymax 记忆卷载标准

> **文档定位**: Airymax 开放标准
> **版本**: 1.0（开放标准草案）
> **最后更新**: 2026-07-09
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 一、范围与目的

本标准定义 AI Agent 记忆管理的开放契约，覆盖：

- MemoryRovol L1-L4 四层分级标准
- GFP 掩码语义标准
- PMEM 持久化接口标准
- 记忆遗忘机制标准

本标准面向：

- **运行时实现者**：实现兼容的记忆管理子系统（用户态堆存储、内核 CXL+MGLRU 等）。
- **Agent 作者**：使用标准 API 存取记忆。
- **工具链开发者**：构建记忆迁移、归档、检索工具。
- **跨运行时互通方**：在不同实现间交换记忆卷。

本标准遵循 IRON-9 v2 [SC] 共享契约层——L1-L4 数据结构、GFP 掩码、PMEM 接口在 agentrt 与 agentrt-liunx 之间完全共享。

### 1.1 术语

| 术语 | 含义 |
|------|------|
| MemoryRovol | Airymax 记忆卷载，四层渐进式记忆架构 |
| L1 工作记忆 | 当前活跃的上下文窗口 |
| L2 短期记忆 | 近期会话与特征缓存 |
| L3 长期记忆 | 结构化知识与向量索引 |
| L4 持久化记忆 | 冷归档与模式层 |
| GFP | Get Free Pages 掩码，记忆分配语义标记 |
| PMEM | Persistent Memory，持久化记忆接口 |
| 遗忘机制 | 基于艾宾浩斯曲线的记忆衰减 |

### 1.2 规范用词

沿用 RFC 2119 用词。

---

## 二、MemoryRovol L1-L4 分级标准

### 2.1 四层分级模型

| 层级 | 名称 | 语义 | 访问延迟 | 实现载体（示例） |
|------|------|------|---------|------------------|
| **L1** | 工作记忆 | 当前认知循环活跃上下文窗口 | < 1 μs | 内存（堆或栈） |
| **L2** | 短期记忆 | 近期会话特征、临时向量缓存 | 1 μs ~ 1 ms | 内存 + 索引 |
| **L3** | 长期记忆 | 结构化知识图谱、向量检索库 | 1 ms ~ 100 ms | SSD + 索引 |
| **L4** | 持久化记忆 | 冷归档、模式挖掘结果、原始数据 | > 100 ms | HDD / 对象存储 |

层级语义**永久稳定**（AOS-STD-MEM-001，L1 稳定级）；每层职责不可变更，但实现载体可演进。

### 2.2 数据流方向

```
原始输入 ──> L1（活跃）──> L2（特征化）──> L3（结构化）──> L4（持久化）
                                                              │
   检索 <──── L1 <──── L2 <──── L3 <─────────────────────────────┘
```

- **写入**：逐层提炼，从原始数据到深层模式。
- **检索**：按需从 L4 反向加载到 L1，分时推理（TimeSliceInfer）。

### 2.3 L1 工作记忆标准

L1 是当前认知循环的活跃上下文窗口，**必须**满足：

- **必须**支持 O(1) 随机读写（AOS-STD-MEM-002）。
- **必须**在 Agent 切换时整体保存/恢复（AOS-STD-MEM-003）。
- **应当**使用连续内存布局以减少 cache miss。
- 容量上限由运行时配置，**建议** 256 KB ~ 4 MB。

### 2.4 L2 短期记忆标准

L2 缓存近期会话特征，**必须**满足：

- **必须**支持基于 key 的检索（AOS-STD-MEM-004）。
- **必须**在 TTL 过期后自动迁移到 L3（AOS-STD-MEM-005）。
- **应当**维护热点索引以加速检索。
- 容量上限**建议** 100 MB ~ 1 GB。

### 2.5 L3 长期记忆标准

L3 存储结构化知识与向量索引，**必须**满足：

- **必须**支持 HNSW 向量检索 + BM25 文本检索 + 混合融合（AOS-STD-MEM-006）。
- **必须**维护知识图谱与 VSA（Vector Symbolic Architecture）绑定/解绑算子（AOS-STD-MEM-007）。
- **应当**支持增量更新而非全量重建。
- 容量上限**建议** 10 GB ~ 1 TB。

### 2.6 L4 持久化记忆标准

L4 存储冷归档与模式挖掘结果，**必须**满足：

- **必须**使用 SHA-256 哈希 + ZSTD 压缩，append-only 写入（AOS-STD-MEM-008）。
- **必须**支持 HDBSCAN 聚类 + 0D 持久同调 + 模式挖掘（AOS-STD-MEM-009）。
- **应当**支持对象存储后端（S3 兼容）。
- 容量无上限。

---

## 三、L1-L4 数据结构 C 定义

### 3.1 通用记忆条目

```c
#define AGENTRT_MEM_MAGIC  0x4d454d31u  /* 'MEM1' */

typedef struct {
    uint32_t magic;          /* 偏移 0:  魔数 0x4d454d31 'MEM1' */
    uint16_t version;        /* 偏移 4:  版本 0x0100 */
    uint16_t layer;          /* 偏移 6:  层级 1-4 */
    uint32_t flags;          /* 偏移 8:  GFP 掩码 */
    uint32_t key_len;        /* 偏移 12: key 长度 */
    uint32_t value_len;      /* 偏移 16: value 长度 */
    uint64_t created_ns;     /* 偏移 24: 创建纳秒时间戳 */
    uint64_t accessed_ns;    /* 偏移 32: 最后访问时间戳 */
    uint64_t access_count;   /* 偏移 40: 访问次数 */
    uint64_t expire_ns;      /* 偏移 48: 过期时间（0=永不过期） */
    uint8_t  reserved[8];    /* 偏移 56: 保留 */
    /* key[key_len] 紧跟其后 */
    /* value[value_len] 紧跟 key 之后 */
} agentrt_mem_entry_hdr_t;  /* 头部 64 字节 */

_Static_assert(sizeof(agentrt_mem_entry_hdr_t) == 64,
               "agentrt_mem_entry_hdr_t must be 64 bytes");
```

记忆条目头部**永久稳定**（AOS-STD-MEM-010，L1 稳定级）。

### 3.2 L1 工作记忆结构

```c
typedef struct {
    uint64_t agent_id;                /* 所属 Agent */
    uint32_t capacity_bytes;          /* 容量上限 */
    uint32_t used_bytes;              /* 已用字节 */
    uint32_t entry_count;             /* 条目数 */
    uint32_t reserved;
    agentrt_mem_entry_hdr_t *entries; /* 条目指针数组 */
    void    *backing_buffer;          /* 连续内存缓冲 */
} agentrt_mem_l1_t;
```

### 3.3 L2 短期记忆结构

```c
typedef struct {
    uint64_t agent_id;
    uint32_t capacity_bytes;
    uint32_t entry_count;
    /* 哈希索引：key -> entry */
    struct agentrt_mem_l2_index *index;
    /* 热点 LRU 链表 */
    struct agentrt_mem_l2_lru   *lru;
    /* TTL 优先队列（按 expire_ns 排序） */
    struct agentrt_mem_l2_ttl  *ttl_queue;
} agentrt_mem_l2_t;
```

### 3.4 L3 长期记忆结构

```c
typedef struct {
    /* 向量索引（HNSW） */
    struct agentrt_hnsw_index *vectors;
    /* 文本索引（BM25） */
    struct agentrt_bm25_index *texts;
    /* 知识图谱 */
    struct agentrt_kg_graph    *knowledge_graph;
    /* VSA 绑定算子表 */
    struct agentrt_vsa_table  *vsa;
} agentrt_mem_l3_t;
```

### 3.5 L4 持久化记忆结构

```c
typedef struct {
    /* 原始卷（append-only） */
    int      raw_fd;             /* 原始卷文件描述符 */
    char     raw_path[256];
    /* 模式层（聚类结果） */
    struct agentrt_hdbscan_result *clusters;
    struct agentrt_0d_persistence  *persistence;
    /* 后端类型 */
    enum { AGENTRT_MEM_BACKEND_FILE, AGENTRT_MEM_BACKEND_S3 } backend;
} agentrt_mem_l4_t;
```

---

## 四、GFP 掩码语义标准

### 4.1 掩码位定义

GFP（Get Free Pages）掩码是记忆分配时的语义标记，**永久稳定**（AOS-STD-MEM-011，L1 稳定级）：

| 位 | 名称 | 值 | 语义 | 调度优先级影响 |
|----|------|-----|------|---------------|
| 0 | `AGENTRT_GFP_L1` | 0x00000001 | 分配 L1 工作记忆 | 实时（不可阻塞） |
| 1 | `AGENTRT_GFP_L2` | 0x00000002 | 分配 L2 短期记忆 | 高（可短暂阻塞） |
| 2 | `AGENTRT_GFP_L3` | 0x00000004 | 分配 L3 长期记忆 | 普通 |
| 3 | `AGENTRT_GFP_L4` | 0x00000008 | 分配 L4 持久化记忆 | 后台 |
| 4 | `AGENTRT_GFP_NOWAIT` | 0x00000010 | 不等待，内存不足立即返回 | — |
| 5 | `AGENTRT_GFP_PERSIST` | 0x00000020 | 持久化分配（写入 PMEM） | — |
| 6 | `AGENTRT_GFP_COMPRESS` | 0x00000040 | 启用 ZSTD 压缩 | — |
| 7 | `AGENTRT_GFP_ENCRYPT` | 0x00000080 | 启用加密（SM4） | — |
| 8 | `AGENTRT_GFP_ATOMIC` | 0x00000100 | 原子分配（不可中断） | — |
| 9 | `AGENTRT_GFP_ZERO` | 0x00000200 | 分配后清零 | — |
| 10 | `AGENTRT_GFP_RECLAIMABLE` | 0x00000400 | 可回收（压力时自动释放） | — |
| 11-15 | 保留 | — | **必须**为 0 | — |

### 4.2 掩码组合规则

| 组合 | 语义 | 约束 |
|------|------|------|
| `L1 \| NOWAIT` | 实时 L1 分配，不足立即失败 | **必须**用于认知循环关键路径 |
| `L3 \| PERSIST` | L3 持久化分配 | **必须**同步写入 PMEM |
| `L4 \| COMPRESS \| ENCRYPT` | L4 加密压缩归档 | **必须**先压缩后加密 |
| `L2 \| RECLAIMABLE` | L2 可回收分配 | 内存压力时优先释放 |

层级位（L1-L4）**互斥**（AOS-STD-MEM-012），同一次分配**不得**同时设置多个层级位。

### 4.3 GFP 接口

```c
/**
 * agentrt_mem_alloc - 按 GFP 掩码分配记忆
 * @gfp:    GFP 掩码
 * @size:   字节数
 *
 * 返回值:
 *   非 NULL: 成功，返回记忆条目指针
 *   NULL:    失败（NOWAIT 时内存不足，或参数非法）
 */
void *agentrt_mem_alloc(uint32_t gfp, size_t size);

/**
 * agentrt_mem_free - 释放记忆
 * @ptr: agentrt_mem_alloc 返回的指针
 */
void agentrt_mem_free(void *ptr);
```

---

## 五、PMEM 持久化接口标准

### 5.1 PMEM 接口语义

PMEM（Persistent Memory）接口用于 L3/L4 持久化记忆，**必须**保证写入持久性（断电不丢失）。接口语义：

| 接口 | 语义 | 持久性保证 |
|------|------|-----------|
| `agentrt_pmem_write` | 写入持久化记忆 | 调用返回时数据已落盘 |
| `agentrt_pmem_read` | 读取持久化记忆 | — |
| `agentrt_pmem_flush` | 刷新缓存到持久介质 | 显式 barrier |
| `agentrt_pmem_sync` | 同步所有未落盘写入 | 全局 barrier |

### 5.2 PMEM 接口定义

```c
/**
 * agentrt_pmem_write - 写入持久化记忆
 * @backend:   PMEM 后端句柄
 * @key:       键（用于去重）
 * @key_len:   键长度
 * @value:     值
 * @value_len: 值长度
 * @gfp:       GFP 掩码（L3 或 L4 + 可选 PERSIST/COMPRESS/ENCRYPT）
 *
 * 写入操作为 append-only：相同 key 不会覆盖，而是新增版本。
 * 旧版本由遗忘机制回收。
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   参数无效
 *   -AGENTRT_ENOMEM:   持久化介质已满
 *   -AGENTRT_EIO:      I/O 错误
 */
int agentrt_pmem_write(agentrt_pmem_t backend,
                       const void *key, size_t key_len,
                       const void *value, size_t value_len,
                       uint32_t gfp);

/**
 * agentrt_pmem_read - 读取持久化记忆
 * @backend:   PMEM 后端句柄
 * @key:       键
 * @key_len:   键长度
 * @value:     输出缓冲区
 * @value_len: 输入：缓冲区大小；输出：实际读取字节数
 *
 * 读取最新版本。
 */
int agentrt_pmem_read(agentrt_pmem_t backend,
                      const void *key, size_t key_len,
                      void *value, size_t *value_len);

/**
 * agentrt_pmem_flush - 刷新到持久介质
 */
int agentrt_pmem_flush(agentrt_pmem_t backend);

/**
 * agentrt_pmem_sync - 全局同步
 */
int agentrt_pmem_sync(void);
```

接口签名**永久稳定**（AOS-STD-MEM-013 ~ AOS-STD-MEM-016，L1 稳定级）。

### 5.3 PMEM 后端抽象

PMEM 后端**可以**是：

- 文件系统（XFS / ext4，启用 O_DIRECT + fsync）
- 持久内存设备（/dev/pmem0，使用 mmap + msync）
- 对象存储（S3 兼容，异步上传）
- 远程块存储（iSCSI / NVMe-oF）

后端类型由实现决定，**不属于**本标准约束范围。但所有后端**必须**满足持久性保证（AOS-STD-MEM-017）。

---

## 六、记忆遗忘机制标准

### 6.1 遗忘公式

遗忘机制基于艾宾浩斯遗忘曲线（AOS-STD-MEM-018，L1 稳定级）：

```
R(t) = e^(-t/τ)
```

其中：
- `R(t)`：t 时刻记忆保留率（0 ~ 1）
- `t`：自最后访问以来经过的时间（秒）
- `τ`：时间常数，默认 7 天（604800 秒），可配置

当 `R(t)` 低于阈值 `R_min`（默认 0.1）时，记忆条目**必须**被迁移到下一层或归档删除。

### 6.2 遗忘策略枚举

```c
typedef enum {
    AGENTRT_FORGET_NONE          = 0,  /* 不遗忘（仅 L1） */
    AGENTRT_FORGET_EBBINGHAUS    = 1,  /* 艾宾浩斯曲线（默认） */
    AGENTRT_FORGET_LINEAR        = 2,  /* 线性衰减 */
    AGENTRT_FORGET_ACCESS_BASED  = 3,  /* 基于访问次数 */
} agentrt_forget_strategy_t;
```

枚举值**永久稳定**（AOS-STD-MEM-019，L1 稳定级）。

### 6.3 遗忘策略语义

| 策略 | 公式 | 迁移时机 |
|------|------|---------|
| NONE | — | 不迁移（L1 专属） |
| EBBINGHAUS | `R(t) = e^(-t/τ)` | `R(t) < R_min` |
| LINEAR | `R(t) = max(0, 1 - t/T)` | `R(t) = 0` |
| ACCESS_BASED | `R = max(0, 1 - 1/(access_count+1))` | `R < R_min` |

### 6.4 遗忘迁移路径

| 源层 | 目标层 | 触发 |
|------|--------|------|
| L1 | L2 | Agent 切换 / L1 容量不足 |
| L2 | L3 | TTL 过期 / `R(t) < R_min` |
| L3 | L4 | 容量不足 / 长期未访问 |
| L4 | 删除 | `R(t) < R_min` 且容量不足 |

迁移**必须**保持数据完整性（AOS-STD-MEM-020，L1 稳定级）——迁移过程中**不得**丢失数据，**必须**先写入目标层再删除源层。

### 6.5 遗忘接口

```c
/**
 * agentrt_forget_run - 执行遗忘机制
 * @strategy:  遗忘策略
 * @tau:       时间常数（秒），0 使用默认 604800
 * @r_min:     保留率阈值，0 使用默认 0.1
 *
 * 扫描所有记忆条目，按策略计算保留率，
 * 迁移或删除低于阈值的条目。
 *
 * 返回值:
 *   >= 0: 处理的条目数
 *   <  0: 错误码
 */
int agentrt_forget_run(agentrt_forget_strategy_t strategy,
                       uint64_t tau, double r_min);
```

---

## 七、记忆存取 API

### 7.1 统一存取接口

```c
/**
 * agentrt_mem_store - 存储记忆到指定层
 * @layer:   目标层级（1-4）
 * @gfp:     GFP 掩码（必须包含对应层级位）
 * @key:     键
 * @key_len: 键长度
 * @value:   值
 * @value_len: 值长度
 * @ttl:     生存时间（秒），0 表示永不过期
 *
 * 返回值:
 *   0:                  成功
 *   -AGENTRT_EINVAL:   参数非法（层级与 gfp 不匹配）
 *   -AGENTRT_ENOMEM:   内存不足
 */
int agentrt_mem_store(uint16_t layer, uint32_t gfp,
                      const void *key, size_t key_len,
                      const void *value, size_t value_len,
                      uint64_t ttl);

/**
 * agentrt_mem_load - 从指定层加载记忆
 * @layer:     源层级
 * @key:       键
 * @key_len:   键长度
 * @value:     输出缓冲区
 * @value_len: 输入：缓冲区大小；输出：实际读取字节数
 */
int agentrt_mem_load(uint16_t layer,
                     const void *key, size_t key_len,
                     void *value, size_t *value_len);

/**
 * agentrt_mem_search - 在指定层检索
 * @layer:   层级
 * @query:   查询（向量或文本）
 * @query_len: 查询长度
 * @k:       返回 top-k 结果
 * @results:  输出：结果数组
 * @result_count: 输入：数组容量；输出：实际结果数
 */
int agentrt_mem_search(uint16_t layer,
                       const void *query, size_t query_len,
                       uint32_t k,
                       agentrt_mem_result_t *results,
                       uint32_t *result_count);
```

### 7.2 跨层迁移接口

```c
/**
 * agentrt_mem_migrate - 跨层迁移记忆
 * @from_layer: 源层
 * @to_layer:   目标层
 * @key:        键
 * @key_len:    键长度
 *
 * 迁移：写入目标层 + 删除源层（原子操作）。
 */
int agentrt_mem_migrate(uint16_t from_layer, uint16_t to_layer,
                        const void *key, size_t key_len);
```

---

## 八、标准条目注册表

| 编号 | 名称 | 稳定级 |
|------|------|--------|
| AOS-STD-MEM-001 | L1-L4 四层分级 | L1 |
| AOS-STD-MEM-002 | L1 O(1) 读写 | L1 |
| AOS-STD-MEM-003 | L1 Agent 切换保存恢复 | L1 |
| AOS-STD-MEM-004 | L2 key 检索 | L1 |
| AOS-STD-MEM-005 | L2 TTL 迁移 L3 | L1 |
| AOS-STD-MEM-006 | L3 HNSW+BM25+融合 | L1 |
| AOS-STD-MEM-007 | L3 KG+VSA | L1 |
| AOS-STD-MEM-008 | L4 SHA-256+ZSTD append-only | L1 |
| AOS-STD-MEM-009 | L4 HDBSCAN+0D+模式挖掘 | L1 |
| AOS-STD-MEM-010 | 记忆条目头 64 字节 | L1 |
| AOS-STD-MEM-011 | GFP 掩码位定义 | L1 |
| AOS-STD-MEM-012 | 层级位互斥 | L1 |
| AOS-STD-MEM-013 | pmem_write 接口 | L1 |
| AOS-STD-MEM-014 | pmem_read 接口 | L1 |
| AOS-STD-MEM-015 | pmem_flush 接口 | L1 |
| AOS-STD-MEM-016 | pmem_sync 接口 | L1 |
| AOS-STD-MEM-017 | PMEM 持久性保证 | L1 |
| AOS-STD-MEM-018 | 艾宾浩斯遗忘公式 | L1 |
| AOS-STD-MEM-019 | 遗忘策略枚举 | L1 |
| AOS-STD-MEM-020 | 迁移原子性 | L1 |
| AOS-STD-MEM-021 | 记忆条目 magic 0x4d454d31 | L1 |

---

## 九、错误码

| 错误码 | 数值 | 含义 |
|--------|------|------|
| `AGENTRT_OK` | 0 | 成功 |
| `AGENTRT_EINVAL` | -2 | 参数非法（层级与 gfp 不匹配） |
| `AGENTRT_ENOMEM` | -1100 | 内存不足 |
| `AGENTRT_ENOENT` | -2 | 记忆条目不存在 |
| `AGENTRT_EMSGSIZE` | -1109 | 记忆条目超过容量 |
| `AGENTRT_EIO` | -5 | I/O 错误（PMEM） |
| `AGENTRT_EPERSIST` | -1101 | 持久化失败 |

---

## 十、兼容性测试要求

| 测试 ID | 测试内容 | 通过条件 |
|---------|---------|---------|
| MEM-T-001 | L1 O(1) 读写 | 读写延迟 < 1 μs |
| MEM-T-002 | L2 TTL 迁移 | TTL 过期后条目出现在 L3 |
| MEM-T-003 | L3 HNSW 检索 | top-k 结果按余弦相似度排序 |
| MEM-T-004 | L4 append-only | 重复 key 不覆盖，新增版本 |
| MEM-T-005 | GFP 层级互斥 | 同时设置 L1\|L2 返回 EINVAL |
| MEM-T-006 | PMEM 持久性 | 断电后数据可读 |
| MEM-T-007 | 遗忘公式 | R(t) < R_min 时条目被迁移 |
| MEM-T-008 | 迁移原子性 | 迁移中断不丢数据 |
| MEM-T-009 | 条目头布局 | 64 字节，magic 匹配 |

---

## 十一、最小示例

### 11.1 L1 工作记忆存取

```c
/* 存储当前对话上下文到 L1 */
const char *key   = "session-42-context";
const char *value = "{\"role\":\"user\",\"content\":\"北京天气如何\"}";
agentrt_mem_store(/* layer */ 1,
                  /* gfp   */ AGENTRT_GFP_L1 | AGENTRT_GFP_NOWAIT,
                  key, strlen(key),
                  value, strlen(value),
                  /* ttl */ 0);

/* 加载 */
char buf[4096];
size_t len = sizeof(buf);
agentrt_mem_load(1, key, strlen(key), buf, &len);
```

### 11.2 L3 向量检索

```c
float query_vec[768] = { /* 嵌入向量 */ };
agentrt_mem_result_t results[10];
uint32_t result_count = 10;
agentrt_mem_search(/* layer */ 3,
                   query_vec, sizeof(query_vec),
                   /* k */ 10,
                   results, &result_count);
/* results 按 cosine 相似度降序排列 */
```

### 11.3 PMEM 持久化归档

```c
agentrt_pmem_t pmem = agentrt_pmem_open("/var/lib/airymax/pmem/l4.raw");
const char *key   = "doc-2026-07-09-001";
const char *value = "原始文档内容...";
agentrt_pmem_write(pmem,
                   key, strlen(key),
                   value, strlen(value),
                   AGENTRT_GFP_L4 | AGENTRT_GFP_PERSIST |
                   AGENTRT_GFP_COMPRESS | AGENTRT_GFP_ENCRYPT);
agentrt_pmem_flush(pmem);  /* 确保落盘 */
```

### 11.4 遗忘机制运行

```c
/* 每日运行艾宾浩斯遗忘 */
int processed = agentrt_forget_run(AGENTRT_FORGET_EBBINGHAUS,
                                    /* tau */ 604800,  /* 7 天 */
                                    /* r_min */ 0.1);
printf("已处理 %d 条记忆\n", processed);
```

---

## 十二、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-07-09 | 初始草案：L1-L4 分级、GFP 掩码、PMEM 接口、艾宾浩斯遗忘机制 |

---

## 十三、参考文献

- [Airymax 开放标准体系总览](./README.md)
- [Airymax 认知循环标准](./05-airymax-cognition-loop-standard.md)
- HNSW 算法：https://arxiv.org/abs/1603.09320
- BM25：https://en.wikipedia.org/wiki/Okapi_BM25
- ZSTD 压缩：https://github.com/facebook/zstd
- HDBSCAN：https://hdbscan.readthedocs.io/
- 艾宾浩斯遗忘曲线：https://en.wikipedia.org/wiki/Forgetting_curve

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
