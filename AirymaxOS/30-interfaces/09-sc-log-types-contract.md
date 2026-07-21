Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] log_types.h 二进制契约
> **文档定位**：A-ULP（统一日志与打印系统）模块的 [SC] 共享契约权威定义\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5\
> **设计依据**：综合修正方案 §2.3（日志类型 SSoT）+ §4.2.2（A-ULP 设计）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax 日志类型与 128B 记录格式** 的唯一权威源。`LOG_*` 日志级别枚举、128B 固定记录格式、`AIRY_LOG_MAGIC` 魔数、printk 8 级映射均以本文件为唯一权威定义。物理宿主为 `kernel/include/uapi/linux/airymax/log_types.h`，agentrt 与 agentrt-linux 物理共享同一份头文件，逐字节相同。
>
> 废弃风格声明：`AIRY_LOG_*`（详细前缀）已废弃，全部迁移为 `LOG_*`。本契约遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) 的技术选型（sched_tac + 纯 C LSM 不使用 BPF LSM + IORING_OP_URING_CMD + registered buffer + mmap 不使用 page flipping + alloc_pages + mmap 不使用 DMA 一致性内存）。

---

## 文档信息卡

- **目标读者**：agentrt-linux 内核开发者、Logger Daemon 开发者、agentrt 用户态开发者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) A-ULP 模块定位、[06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) [SC] 层概念
- **预计阅读时间**：20 分钟
- **核心概念**：128B 固定记录、`LOG_*` 5 级枚举、printk 8 级映射、Fastpath 禁止格式化
- **复杂度标识**：中级

---

## §1 契约概述：A-ULP 模块的 [SC] 共享契约

A-ULP（Unified Logging and Printk Subsystem，统一日志与打印系统）是 Airymax Unify Design 的观测主干模块，其 [SC] 共享契约头文件 `log_types.h` 定义了双端共享的日志记录格式与级别枚举。

### 1.1 契约范围

本契约定义以下内容，agentrt 与 agentrt-linux 双端必须逐字节一致：

1. `AIRY_LOG_MAGIC` 魔数
2. `airy_log_level` 枚举（5 级日志级别）
3. `airy_log_record` 结构体（128B 固定记录格式）
4. `airy_log_facility` 设施编号枚举
5. printk 8 级映射宏
6. `#ifdef AIRY_SC_FALLBACK` 降级块

### 1.2 物理宿主

| 属性 | 值 |
|------|-----|
| 物理路径 | `kernel/include/uapi/linux/airymax/log_types.h` |
| 共享层级 | [SC] 共享契约层（IRON-9 v3 第一层） |
| 共享方式 | 物理共享，逐字节相同 |
| CI 校验 | `sc-dual-ci.yml` 逐字节校验 |
| 关联实现 | `kernel/log/airy_log_ring.c`（内核侧）+ `services/daemons/logger_d/`（用户态） |

### 1.3 设计原则

A-ULP 遵循"Fastpath 禁止格式化"原则——128B 记录的 `payload` 字段存储**原始二进制**，不在内核侧进行 `sprintf` 格式化。格式化由用户态 Logger Daemon 异步完成。这一原则确保内核 fastpath 性能（~50-100ns），详见 [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md)。

---

## §2 128B 固定记录格式

### 2.1 记录布局

A-ULP 采用 128 字节固定长度日志记录，对齐 cache line（64B 的 2 倍），确保单次 `memcpy` 完成写入，避免跨 cache line 的拆分开销。

```c
/* kernel/include/uapi/linux/airymax/log_types.h —— [SC] 共享契约层，逐字节共享 */
#define AIRY_LOG_MAGIC    0x414C4F47u   /* 'ALOG' —— 日志记录 magic */

struct airy_log_record {
    __u32   magic;          /* offset  0, 4B: AIRY_LOG_MAGIC (0x414C4F47) */
    __u16   level;          /* offset  4, 2B: 日志级别（airy_log_level） */
    __u16   facility;       /* offset  6, 2B: 设施编号（airy_log_facility） */
    __u64   timestamp_ns;   /* offset  8, 8B: 纳秒时间戳（CLOCK_MONOTONIC） */
    __u32   caller_id;      /* offset 16, 4B: 调用者 ID（pid / kmod id） */
    __u32   payload_len;    /* offset 20, 4B: payload 有效长度（≤96） */
    __u8    payload[96];    /* offset 24, 96B: 原始二进制 payload（不格式化） */
    __u64   reserved;       /* offset 120, 8B: 预留对齐 + 校验位 */
} __attribute__((aligned(64)));
/* sizeof(struct airy_log_record) == 128 */
```

### 2.2 字段详解

| 字段 | 偏移 | 长度 | 类型 | 说明 |
|------|------|------|------|------|
| `magic` | 0 | 4B | `__u32` | `AIRY_LOG_MAGIC = 0x414C4F47`（ASCII 'A''L''O''G'），接收方必须校验 |
| `level` | 4 | 2B | `__u16` | 日志级别，取值见 `airy_log_level` 枚举（0-4） |
| `facility` | 6 | 2B | `__u16` | 设施编号，标识日志来源（内核模块/daemon），取值见 `airy_log_facility` 枚举 |
| `timestamp_ns` | 8 | 8B | `__u64` | 纳秒时间戳，时钟源 `CLOCK_MONOTONIC`（禁用 `CLOCK_REALTIME`，避免跳变） |
| `caller_id` | 16 | 4B | `__u32` | 调用者 ID：内核侧为 kmod id，用户态为 pid |
| `payload_len` | 20 | 4B | `__u32` | payload 有效长度，最大 96（超出截断） |
| `payload[96]` | 24 | 96B | `__u8[]` | 原始二进制 payload，由 Logger Daemon 解析格式 |
| `reserved` | 120 | 8B | `__u64` | 预留对齐 + 可选校验位（高 16 位存 CRC，低 48 位保留） |

### 2.3 关键约束

1. **结构体名称**：`airy_log_record`（**不是** `airy_log_entry` 或 `airymaxos_log_record_t`）
2. **对齐方式**：`__attribute__((aligned(64)))`（D-9 修复后禁用 `__attribute__((packed))`，参考 OLK 6.6 内核协议头自然对齐）
3. **类型**：使用内核 UAPI 类型 `__u32`/`__u16`/`__u64`/`__u8`（**不是** `uint32_t` 等）
4. **总大小**：严格 128 字节，`sizeof(struct airy_log_record) == 128`
5. **payload 禁止格式化**：内核 fastpath 仅 `memcpy` 原始二进制到 `payload`，禁止 `sprintf`/`snprintf`
6. **时钟源**：`CLOCK_MONOTONIC`，禁用 `CLOCK_REALTIME`

### 2.4 reserved 字段的 CRC 选项

`reserved` 高 16 位可选存储 payload 的 CRC16 校验值。当 `AIRY_LOG_OPT_CRC` 编译选项启用时，写入方计算 CRC 并填入；读取方校验 CRC，不匹配则丢弃该记录。降级模式下 CRC 校验关闭。

---

## §3 5 级日志枚举

### 3.1 airy_log_level 枚举

A-ULP 定义 5 级日志枚举，从最详细到最严重：

```c
/* kernel/include/uapi/linux/airymax/log_types.h */
enum airy_log_level {
    LOG_DEBUG   = 0,    /* 调试信息（最详细，生产关闭） */
    LOG_INFO    = 1,    /* 常规信息 */
    LOG_WARN    = 2,    /* 警告（可恢复异常） */
    LOG_ERROR   = 3,    /* 错误（可恢复但需关注） */
    LOG_FATAL   = 4,    /* 致命（不可恢复，触发 Panic/Fault） */
};
```

### 3.2 级别语义

| 级别 | 值 | 语义 | 内核默认输出 | Logger Daemon 默认落盘 |
|------|-----|------|------------|---------------------|
| `LOG_DEBUG` | 0 | 调试信息 | 否 | 否 |
| `LOG_INFO` | 1 | 常规信息 | 是 | 是 |
| `LOG_WARN` | 2 | 警告 | 是 | 是 |
| `LOG_ERROR` | 3 | 错误 | 是 | 是 |
| `LOG_FATAL` | 4 | 致命 | 是（+Panic） | 是（+告警） |

### 3.3 [DSL] 降级模式

降级模式下（`AIRY_SC_FALLBACK` 激活），仅保留 `LOG_FATAL` + `LOG_ERROR` 两级，其余级别被预处理为空操作，直接走 `printk` 原生路径。详见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §4.1.4。

### 3.4 设施编号 airy_log_facility

设施编号标识日志来源，便于 Logger Daemon 按模块过滤：

```c
/* kernel/include/uapi/linux/airymax/log_types.h */
enum airy_log_facility {
    AIRY_FAC_KERNEL      = 0,     /* 内核主体 */
    AIRY_FAC_IPC         = 1,     /* A-IPC：IPC 子系统 */
    AIRY_FAC_SCHED       = 2,     /* A-ULS：调度子系统 */
    AIRY_FAC_SUPERV      = 3,     /* A-ULS：Micro-Supervisor */
    AIRY_FAC_LOG         = 4,     /* A-ULP：日志子系统自身 */
    AIRY_FAC_CONFIG      = 5,     /* A-UCS：配置管理 */
    AIRY_FAC_SECURITY    = 6,     /* 纯 C LSM 安全子系统 */
    AIRY_FAC_MEMORY      = 7,     /* 记忆子系统 */
    AIRY_FAC_COGNITION   = 8,     /* 认知子系统 */
    AIRY_FAC_MACRO_SUPV  = 16,    /* Macro-Supervisor（用户态） */
    AIRY_FAC_LOGGER_D    = 17,    /* Logger Daemon（用户态） */
    AIRY_FAC_CONFIG_D    = 18,    /* Config Daemon（用户态） */
    /* 用户态 daemon 从 16 起 */
};
```

---

## §4 printk 8 级映射：KERN_EMERG~KERN_DEBUG → LOG_FATAL~LOG_DEBUG

### 4.1 映射表

Linux 6.6 的 `printk` 定义 8 级日志级别（`KERN_EMERG` ~ `KERN_DEBUG`），A-ULP 将其映射到 5 级 `airy_log_level`：

| printk 级别 | 值 | A-ULP 级别 | 映射规则 |
|------------|-----|----------|---------|
| `KERN_EMERG` | 0 | `LOG_FATAL` | 系统不可用 → 致命 |
| `KERN_ALERT` | 1 | `LOG_FATAL` | 必须立即行动 → 致命 |
| `KERN_CRIT` | 2 | `LOG_FATAL` | 临界条件 → 致命 |
| `KERN_ERR` | 3 | `LOG_ERROR` | 错误条件 → 错误 |
| `KERN_WARNING` | 4 | `LOG_WARN` | 警告条件 → 警告 |
| `KERN_NOTICE` | 5 | `LOG_INFO` | 正常但重要 → 常规信息 |
| `KERN_INFO` | 6 | `LOG_INFO` | 信息 → 常规信息 |
| `KERN_DEBUG` | 7 | `LOG_DEBUG` | 调试 → 调试 |

### 4.2 映射宏

```c
/* kernel/include/uapi/linux/airymax/log_types.h —— printk 8 级映射 */
#define AIRY_PRINTK_TO_LEVEL(lvl)  (                 \
    (lvl) <= KERN_CRIT      ? LOG_FATAL  :           \
    (lvl) == KERN_ERR       ? LOG_ERROR  :           \
    (lvl) == KERN_WARNING   ? LOG_WARN   :           \
    (lvl) <= KERN_INFO      ? LOG_INFO   :           \
                              LOG_DEBUG)
```

### 4.3 printk 桥接

`kernel/log/airy_printk_bridge.c` 实现 printk → Ring Buffer 的桥接：拦截 `printk` 写入，转换为 `airy_log_record` 后写入 Ring Buffer。Panic 时回退到 `printk_safe` 原生路径（详见 [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) §6）。

---

## §5 物理宿主：kernel/include/uapi/linux/airymax/log_types.h

### 5.1 文件位置

| 属性 | 值 |
|------|-----|
| 内核侧路径 | `agentrt-linux/kernel/include/uapi/linux/airymax/log_types.h` |
| agentrt 侧路径 | `agentrt/commons/include/uapi/linux/airymax/log_types.h`（引用同一物理文件） |
| 实际物理文件 | 仅一份，位于 `agentrt-linux/kernel/include/uapi/linux/airymax/log_types.h` |
| 文件编码 | UTF-8 without BOM |
| 换行符 | LF（Unix） |

### 5.2 文件结构

`log_types.h` 文件结构自上而下：

1. 版权头 + 文件说明注释
2. 头文件守卫 `#ifndef _AIRY_LOG_TYPES_H`
3. `AIRY_LOG_MAGIC` 魔数定义
4. `airy_log_level` 枚举（5 级）
5. `airy_log_facility` 枚举
6. `airy_log_record` 结构体（128B）
7. printk 8 级映射宏
8. 内联辅助函数 `airy_log_level_valid()` 等
9. `#ifdef AIRY_SC_FALLBACK` 降级块（仅 `LOG_FATAL` + `LOG_ERROR`）
10. 头文件守卫结束 `#endif`

### 5.3 变更流程

`log_types.h` 的任何变更必须遵循 [SC] 共享契约变更流程：

1. 在本文件（`09-sc-log-types-contract.md`）提出变更提案
2. agentrt 与 agentrt-linux 双方维护者评审
3. 同步修改 `kernel/include/uapi/linux/airymax/log_types.h` 物理头文件
4. `sc-dual-ci.yml` 双端校验通过
5. 更新本契约文档版本号

### 5.4 与 128B IPC 消息头的关系

`airy_log_record`（128B）与 `airy_ipc_msg_hdr`（128B）长度相同但语义不同——前者是日志记录，后者是 IPC 消息头。两者使用不同的 magic（`AIRY_LOG_MAGIC=0x414C4F47` vs `AIRY_IPC_MAGIC=0x41524531`）避免混淆，且物理宿主分别在 `log_types.h` 与 `ipc.h`。禁止将两者复用同一结构体。

---

## §6 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §5 —— A-ULP 模块总纲
- [40-dataflows/05-ring-buffer-logging.md](../40-dataflows/05-ring-buffer-logging.md) —— 零拷贝 Ring Buffer 日志数据流
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §4.1.4 —— [DSL] 降级日志子集
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §2 —— [SC] 共享契约层
- [02-ipc-protocol.md](02-ipc-protocol.md) —— IPC 128B 消息头（与日志记录对比）
- 综合修正方案 §2.3 / §4.2.2 —— 设计依据

---

## §7 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：A-ULP [SC] log_types.h 二进制契约；128B 固定记录格式（magic/level/facility/timestamp/caller_id/payload/reserved）；5 级日志枚举（LOG_DEBUG~LOG_FATAL）；printk 8 级映射；物理宿主 `kernel/include/uapi/linux/airymax/log_types.h` |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-8 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] log_types.h 二进制契约 | v1.0.1 | 2026-07-21
