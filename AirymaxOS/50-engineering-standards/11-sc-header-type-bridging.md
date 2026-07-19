Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] 三路类型桥接规范
> **文档定位**：[SC] 共享契约头文件在内核态、用户态、第三方三环境编译兼容的唯一权威规范\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §4 + [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)\
> **设计依据**：综合修正方案 §4.2.1（A-UEF [SC] 设计）+ §6.2.1 C-04（[SC] 头文件数量不一致修正）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **[SC] 三路类型桥接规范** 的唯一权威源。三路类型桥接模型（`#ifdef __KERNEL__` / `#ifdef __linux__` / `#else`）、`uapi_compat.h` 设计、物理宿主 `kernel/include/uapi/linux/airymax/uapi_compat.h`、CI 三路编译校验均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 [SC] 头文件跨环境编译兼容策略。
>
> 技术选型声明：整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

---

## 文档信息卡

- **目标读者**：内核开发者、用户态开发者、跨平台集成工程师、CI 维护者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) [SC] 共享契约层、[06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) IRON-9 v3 [SC]/[SS]/[IND] 三层模型、C 预处理器条件编译
- **预计阅读时间**：25 分钟
- **核心概念**：三路类型桥接、`#ifdef __KERNEL__`、`uapi_compat.h`、编译兼容、CI 三路编译
- **复杂度标识**：中级

---

## §1 设计目标：[SC] 头文件在内核态、用户态、第三方三环境编译兼容

### 1.1 问题背景

[SC] 共享契约头文件（物理宿主 `kernel/include/uapi/linux/airymax/`，共 10 个）由 agentrt（用户态 SDK）与 agentrt-linux（内核态实现）双端逐字节共享。但两端运行在不同编译环境：

| 环境 | 编译器 | 类型系统 | 头文件依赖 |
|------|--------|---------|-----------|
| 内核态 | GCC（内核构建） | 内核类型（`struct task_struct`、`__u32`） | `<linux/types.h>` |
| 用户态 Linux | GCC/Clang | 用户态类型（`uint32_t`） | `<stdint.h>`、`<linux/types.h>` |
| 第三方平台 | 任意 C 编译器 | 标准类型（`uint32_t`） | 仅 `<stdint.h>` |

若 [SC] 头文件直接使用某一环境的类型，在其他环境会编译失败。例如直接使用 `struct task_struct` 会导致用户态编译失败。

### 1.2 设计目标

三路类型桥接规范的核心目标是：**一份 [SC] 头文件，三环境编译兼容**：

1. **内核态兼容**：使用内核类型（`__u32`、`struct task_struct` 等）
2. **用户态 Linux 兼容**：使用用户态类型（`uint32_t`）
3. **第三方平台兼容**：仅依赖标准 C 类型（`stdint.h`）

### 1.3 与 IRON-9 v3 的关系

三路类型桥接对应 IRON-9 v3 的 [SC] 共享契约层——物理宿主唯一，两端逐字节相同（详见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §1）。类型桥接是 [SC] 层的核心工程挑战，确保"一份头文件多环境编译"可行。

---

## §2 三路类型桥接模型

### 2.1 三路条件编译模型

[SC] 头文件使用三层条件编译，根据编译环境选择对应类型：

```c
/* 三路类型桥接模型 */
#ifdef __KERNEL__
    /* 路径一：内核态 —— 使用内核类型 */
    #include <linux/types.h>
    /* 如 __u32, __u64, struct task_struct 等 */
#elif defined(__linux__)
    /* 路径二：用户态 Linux —— 使用 Linux 用户态类型 */
    #include <stdint.h>
    #include <linux/types.h>
    /* 如 uint32_t, uint64_t 等 */
#else
    /* 路径三：第三方平台 —— 仅使用标准 C 类型 */
    #include <stdint.h>
    /* 如 uint32_t, uint64_t 等 */
#endif
```

### 2.2 三路映射表

| 类型语义 | 路径一（`__KERNEL__`） | 路径二（`__linux__`） | 路径三（`#else`） |
|---------|---------------------|---------------------|------------------|
| 32位无符号 | `__u32` | `uint32_t` | `uint32_t` |
| 64位无符号 | `__u64` | `uint64_t` | `uint64_t` |
| 16位无符号 | `__u16` | `uint16_t` | `uint16_t` |
| 8位无符号 | `__u8` | `uint8_t` | `uint8_t` |
| 进程标识 | `struct task_struct *` | `pid_t` | `int32_t` |
| 布尔 | `bool`（`<linux/types.h>`） | `bool`（`<stdbool.h>`） | `int` |

### 2.3 预处理器宏判定逻辑

条件编译的判定基于编译器预定义宏：

| 宏 | 定义者 | 含义 | 示例环境 |
|----|--------|------|---------|
| `__KERNEL__` | 内核构建系统 | 内核态编译 | `make` 内核构建 |
| `__linux__` | GCC/Clang 预定义 | Linux 平台（用户态） | 用户态 GCC 编译 |
| （无以上宏） | — | 第三方平台 | 非 Linux 的 C 编译器 |

判定优先级：`__KERNEL__` > `__linux__` > `#else`。这确保内核态优先识别，即使内核态也会定义 `__linux__`。

### 2.4 类型桥接示例

以 `struct airy_log_record` 为例，展示三路桥接：

```c
/* kernel/include/uapi/linux/airymax/log_types.h —— 三路类型桥接示例 */
#ifndef _AIRYM_LOG_TYPES_H
#define _AIRYM_LOG_TYPES_H

#include "uapi_compat.h"  /* 三路类型桥接头文件 */

struct airy_log_record {
    airy_u32  magic;          /* 路径一: __u32, 路径二/三: uint32_t */
    airy_u16  level;
    airy_u16  facility;
    airy_u64  timestamp_ns;
    airy_u32  caller_id;
    airy_u32  payload_len;
    char      payload[96];
    airy_u64  reserved;
} __attribute__((aligned(64)));

#endif /* _AIRYM_LOG_TYPES_H */
```

---

## §3 uapi_compat.h 设计：三路类型桥接的实现头文件

### 3.1 uapi_compat.h 完整设计

`uapi_compat.h` 是三路类型桥接的实现头文件，定义统一的类型别名：

```c
/* kernel/include/uapi/linux/airymax/uapi_compat.h —— 三路类型桥接 */
#ifndef _AIRYM_UAPI_COMPAT_H
#define _AIRYM_UAPI_COMPAT_H

/*
 * 三路类型桥接：内核态 / 用户态 Linux / 第三方平台
 *
 * 判定优先级：__KERNEL__ > __linux__ > else
 * 确保内核态优先识别（内核态也会定义 __linux__）。
 */

/* ───────── 路径一：内核态 ───────── */
#ifdef __KERNEL__
#include <linux/types.h>
#include <linux/compiler.h>

/* 内核态：直接使用内核类型 */
typedef __u32   airy_u32;
typedef __u64   airy_u64;
typedef __u16   airy_u16;
typedef __u8    airy_u8;
typedef __s32   airy_s32;
typedef __s64   airy_s64;
typedef bool    airy_bool;

/* 内核态独有类型（用户态不需要） */
struct task_struct;  /* 前向声明 */

/* ───────── 路径二：用户态 Linux ───────── */
#elif defined(__linux__)
#include <stdint.h>
#include <stdbool.h>
#include <linux/types.h>  /* 用户态也可访问 <linux/types.h> */

/* 用户态 Linux：使用 stdint + linux/types 双重保障 */
typedef uint32_t  airy_u32;
typedef uint64_t  airy_u64;
typedef uint16_t  airy_u16;
typedef uint8_t   airy_u8;
typedef int32_t   airy_s32;
typedef int64_t   airy_s64;
typedef bool      airy_bool;

/* ───────── 路径三：第三方平台 ───────── */
#else
#include <stdint.h>
#include <stdbool.h>

/* 第三方平台：仅依赖标准 C（stdint.h） */
typedef uint32_t  airy_u32;
typedef uint64_t  airy_u64;
typedef uint16_t  airy_u16;
typedef uint8_t   airy_u8;
typedef int32_t   airy_s32;
typedef int64_t   airy_s64;
typedef bool      airy_bool;

#endif /* __KERNEL__ / __linux__ / else */

/* ───────── 通用定义（三路共享） ───────── */

/* 结构体对齐宏：确保三环境二进制布局一致 */
#ifdef __KERNEL__
#define AIRY_PACKED  /* D-9: 禁用 __attribute__((packed))，改用自然对齐 */
#define AIRY_ALIGNED(x) __attribute__((aligned(x)))
#else
#define AIRY_PACKED  /* D-9: 禁用 __attribute__((packed))，改用自然对齐 */
#define AIRY_ALIGNED(x) __attribute__((aligned(x)))
#endif

/* 常量宏：三路通用 */
#define AIRY_PACKED_STRUCT struct AIRY_PACKED

#endif /* _AIRYM_UAPI_COMPAT_H */
```

### 3.2 类型别名设计原则

`uapi_compat.h` 定义统一的 `airy_*` 类型别名，[SC] 头文件统一使用别名而非原始类型：

| 别名 | 路径一（内核） | 路径二（用户 Linux） | 路径三（第三方） |
|------|--------------|---------------------|-----------------|
| `airy_u32` | `__u32` | `uint32_t` | `uint32_t` |
| `airy_u64` | `__u64` | `uint64_t` | `uint64_t` |
| `airy_u16` | `__u16` | `uint16_t` | `uint16_t` |
| `airy_u8` | `__u8` | `uint8_t` | `uint8_t` |
| `airy_bool` | `bool` | `bool` | `bool` |

### 3.3 二进制布局一致性

三路桥接的核心保证是**二进制布局一致**——同一结构体在三环境的字节大小、字段偏移完全相同：

```c
/* 验证：三环境 sizeof 与 offsetof 一致 */
struct airy_log_record rec;
_Static_assert(sizeof(rec) == 128, "airy_log_record must be 128B");
_Static_assert(offsetof(struct airy_log_record, magic) == 0, "magic offset");
_Static_assert(offsetof(struct airy_log_record, level) == 4, "level offset");
_Static_assert(offsetof(struct airy_log_record, timestamp_ns) == 8, "ts offset");
```

---

## §4 物理宿主：kernel/include/uapi/linux/airymax/uapi_compat.h

### 4.1 物理宿主

| 文件 | 路径 | 说明 |
|------|------|------|
| uapi_compat.h | `kernel/include/uapi/linux/airymax/uapi_compat.h` | 三路类型桥接实现 |
| [SC] 头文件 | `kernel/include/uapi/linux/airymax/*.h` | 10 个 [SC] 头文件，均 #include uapi_compat.h |

### 4.2 与 10 个 [SC] 头文件的关系

所有 10 个 [SC] 头文件都必须在首行 `#include "uapi_compat.h"`，确保类型桥接：

| [SC] 头文件 | 用途 | 使用类型 |
|------------|------|---------|
| `error.h` | A-UEF 错误码 | `airy_s32`（返回码） |
| `log_types.h` | A-ULP 日志类型 | `airy_u32/u16/u64`（记录字段） |
| `ipc.h` | A-IPC IPC 协议 | `airy_u32/u64`（消息头） |
| `sched.h` | A-ULS 调度扩展 | `airy_u32/u64`（调度参数） |
| `config.h` | A-UCS 配置 | `airy_u32`（配置常量） |
| `superv.h` | A-ULS 监管 | `airy_u32`（Fault 码） |
| `cap.h` | Capability | `airy_u32/u64`（cap_id/badge） |
| `ring.h` | Ring Buffer | `airy_u64`（head/tail 索引） |
| `uapi_compat.h` | 类型桥接 | （本文件） |
| `version.h` | 版本号 | `airy_u32` |

### 4.3 双端共享机制

`uapi_compat.h` 与其他 [SC] 头文件一样，由 agentrt 与 agentrt-linux 双端逐字节共享。CI 通过 `sc-dual-ci.yml` 验证两端一致性：

| 校验项 | CI 脚本 | 频率 |
|--------|---------|------|
| 逐字节对比 | `sc-dual-ci.yml` | 每次 PR |
| 类型大小校验 | `type-size-check.c` | 每次 PR |
| 字段偏移校验 | `offsetof-check.c` | 每次 PR |
| 三路编译 | `tri-compile-ci.yml` | 每次 PR |

---

## §5 CI 校验：三路编译测试

### 5.1 三路编译测试矩阵

CI 对每个 [SC] 头文件执行三路编译测试，确保三环境均编译通过：

| 编译路径 | 编译环境 | 编译器 | 包含宏 |
|---------|---------|--------|--------|
| 路径一（内核） | 内核构建 | GCC（内核配置） | `__KERNEL__` |
| 路径二（用户 Linux） | 用户态构建 | GCC/Clang | `__linux__`（预定义） |
| 路径三（第三方） | 纯标准 C | 任意 C99 编译器 | （无特殊宏） |

### 5.2 CI 脚本示例

```yaml
# .github/workflows/sc-tri-compile-ci.yml —— 三路编译 CI
name: [SC] Tri-Compile CI
on: [pull_request]

jobs:
  kernel-compile:
    name: Kernel path (__KERNEL__)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Compile as kernel
        run: |
          # 内核态编译：定义 __KERNEL__
          gcc -D__KERNEL__ -Ikernel/include -c \
              kernel/include/uapi/linux/airymax/*.h -o /dev/null

  userspace-linux-compile:
    name: Userspace Linux path (__linux__)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Compile as userspace Linux
        run: |
          # 用户态 Linux 编译：__linux__ 由 GCC 预定义
          gcc -Ikernel/include -c \
              kernel/include/uapi/linux/airymax/*.h -o /dev/null

  third-party-compile:
    name: Third-party path (no macros)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Compile as third-party
        run: |
          # 第三方编译：不定义任何特殊宏
          # 仅依赖 stdint.h，模拟非 Linux 环境
          gcc -U__linux__ -U__KERNEL__ -std=c99 \
              -Ikernel/include -c \
              kernel/include/uapi/linux/airymax/*.h -o /dev/null

  type-size-check:
    name: Type size consistency
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verify type sizes
        run: |
          # 验证 airy_u32 在三环境都是 4 字节
          cat > /tmp/check.c << 'EOF'
          #include "airymax/uapi_compat.h"
          #include <assert.h>
          int main() {
              assert(sizeof(airy_u32) == 4);
              assert(sizeof(airy_u64) == 8);
              assert(sizeof(airy_u16) == 2);
              assert(sizeof(airy_u8) == 1);
              return 0;
          }
          EOF
          # 三路编译并运行
          gcc -D__KERNEL__ -Ikernel/include /tmp/check.c -o /tmp/check_kern
          gcc -Ikernel/include /tmp/check.c -o /tmp/check_linux
          gcc -U__linux__ -std=c99 -Ikernel/include /tmp/check.c -o /tmp/check_3rd
          /tmp/check_kern && /tmp/check_linux && /tmp/check_3rd
```

### 5.3 CI 校验清单

| 校验项 | 校验内容 | 失败处理 |
|--------|---------|---------|
| 三路编译通过 | 三环境均无编译错误 | 阻止 PR 合并 |
| 类型大小一致 | `airy_u32/u64` 三环境大小相同 | 阻止 PR 合并 |
| 字段偏移一致 | 结构体字段偏移三环境相同 | 阻止 PR 合并 |
| 逐字节对比 | agentrt 与 agentrt-linux 一致 | 阻止 PR 合并 |

---

## §6 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §4 —— A-UEF [SC] 共享契约
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) —— IRON-9 v3 [SC]/[SS]/[IND] 三层模型
- [08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) —— A-UEF [SC] error.h 契约（使用 uapi_compat.h）
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— A-ULP [SC] log_types.h 契约
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2 —— [DSL] 降级块（不依赖 uapi_compat.h）
- 综合修正方案 §4.2.1 / §6.2.1 C-04 —— 设计依据

---

## §7 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：[SC] 三路类型桥接规范；三路条件编译模型（`#ifdef __KERNEL__` / `#ifdef __linux__` / `#else`）；uapi_compat.h 设计（统一 airy_* 类型别名）；物理宿主 kernel/include/uapi/linux/airymax/uapi_compat.h；CI 三路编译测试（内核/用户 Linux/第三方）；二进制布局一致性保证（_Static_assert） |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] 三路类型桥接规范 | v1.0 | 2026-07-17
