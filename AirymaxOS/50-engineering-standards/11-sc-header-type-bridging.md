Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] 类型桥接规范
> **文档定位**：[SC] 共享契约头文件在 Linux 平台（内核态 + 用户态）与第三方平台（macOS/Windows）编译兼容的唯一权威规范\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §4 + [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)\
> **设计依据**：综合修正方案 §4.2.1（A-UEF [SC] 设计）+ §6.2.1 C-04（[SC] 头文件数量不一致修正）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **[SC] 类型桥接规范** 的唯一权威源。类型桥接模型（`#ifdef __KERNEL__` / `#elif defined(__linux__)` / `#else` 三路判定）、`uapi_compat.h` 设计、物理宿主 `kernel/include/uapi/linux/airymax/uapi_compat.h`、CI 三路编译校验均以本文件为唯一权威定义。其余文档只能引用本文件，禁止重新定义 [SC] 头文件跨环境编译兼容策略。
>
> 技术选型声明：整体遵循 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）+ alloc_pages + mmap（不使用 DMA 一致性内存）。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。
>
> **关键设计决策**：[SC] 头文件统一使用 `__u8/__u16/__u32/__u64/__s8/__s16/__s32/__s64` 等 Linux UAPI 风格类型名称，**不引入 `airy_u32` 等自定义别名**。`uapi_compat.h` 在 Linux 平台 `#include <linux/types.h>` 复用系统定义，在第三方平台通过 `typedef` 将 `stdint.h` 类型映射为 UAPI 名称。

---

## 文档信息卡

- **目标读者**：内核开发者、用户态开发者、跨平台集成工程师、CI 维护者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) [SC] 共享契约层、[06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) IRON-9 v3 [SC]/[SS]/[IND] 三层模型、C 预处理器条件编译
- **预计阅读时间**：25 分钟
- **核心概念**：类型桥接、`#ifdef __KERNEL__` / `#elif defined(__linux__)`、`uapi_compat.h`、编译兼容、CI 三路编译
- **复杂度标识**：中级

---

## §1 设计目标：[SC] 头文件在内核态、用户态 Linux、第三方平台编译兼容

### 1.1 问题背景

[SC] 共享契约头文件（物理宿主 `kernel/include/uapi/linux/airymax/`，共 12 个文件：10 个 [SC] 核心 + bpf_struct_ops.h 补充 + syscall.h codegen 产物）由 agentrt（用户态 SDK）与 agentrt-linux（内核态实现）双端逐字节共享。但两端运行在不同编译环境：

| 环境 | 编译器 | 类型系统 | 头文件依赖 |
|------|--------|---------|-----------|
| 内核态（Linux） | GCC（内核构建） | 内核 UAPI 类型（`__u32`、`__u64`） | `<linux/types.h>` |
| 用户态 Linux | GCC/Clang | 用户态类型 + 内核 UAPI 类型 | `<stdint.h>` + `<linux/types.h>` |
| 第三方平台（macOS/Windows） | 任意 C 编译器 | 标准类型（`uint32_t`） | 仅 `<stdint.h>` |

若 [SC] 头文件直接使用某一环境的类型，在其他环境会编译失败。例如直接使用 `struct task_struct` 会导致用户态编译失败；直接在 Linux 平台 `typedef uint64_t __u64` 会与 `<linux/types.h>` 的系统定义冲突（`conflicting types for '__u64'; have 'uint64_t' {aka 'long unsigned int'} vs 'unsigned long long'`）。

### 1.2 设计目标

类型桥接规范的核心目标是：**一份 [SC] 头文件，多环境编译兼容**：

1. **内核态兼容**：使用 Linux 内核 UAPI 类型（`__u32`、`__u64` 等），由 `<linux/types.h>` 提供
2. **用户态 Linux 兼容**：复用同一份 `<linux/types.h>` 提供的 UAPI 类型
3. **第三方平台兼容**：通过 `typedef` 将 `<stdint.h>` 的 `uint8_t/uint16_t/uint32_t/uint64_t` 映射为 `__u8/__u16/__u32/__u64`，保持源码层面的类型名称一致

**核心设计原则**：[SC] 头文件统一使用 `__u8/__u16/__u32/__u64/__s8/__s16/__s32/__s64` 这些 Linux UAPI 风格的类型名称（而非自定义 `airy_*` 别名）。`uapi_compat.h` 的职责是在第三方平台上提供这些类型的定义，使源码无需条件编译即可在多环境编译。

### 1.3 与 IRON-9 v3 的关系

类型桥接对应 IRON-9 v3 的 [SC] 共享契约层——物理宿主唯一，两端逐字节相同（详见 [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §1）。类型桥接是 [SC] 层的核心工程挑战，确保"一份头文件多环境编译"可行。

---

## §2 类型桥接模型

### 2.1 三路条件编译模型

`uapi_compat.h` 使用**三路条件编译**（基于 `__KERNEL__` 与 `__linux__` 宏），根据编译环境选择对应类型来源。三路判定区分内核态、Linux 用户态、非 Linux 用户态：

```c
/* uapi_compat.h 三路类型桥接模型 */
#ifdef __KERNEL__
    /* 路径一：内核态（Linux 内核构建）
     * <linux/types.h> 由内核 UAPI 提供，定义 __u8/__u16/__u32/__u64 等 */
    #include <linux/types.h>
#elif defined(__linux__)
    /* 路径二：Linux 用户态
     * <linux/types.h> 同样提供 __u8/__u16/__u32/__u64 定义
     * 直接复用系统定义，避免 typedef 与系统定义冲突 */
    #include <linux/types.h>
#else
    /* 路径三：第三方平台（macOS/Windows 等）
     * 通过 typedef 将 stdint.h 类型映射为 __u8/__u16/__u32/__u64 */
    #include <stdint.h>
    typedef uint8_t  __u8;
    typedef uint16_t __u16;
    typedef uint32_t __u32;
    typedef uint64_t __u64;
    typedef int8_t   __s8;
    typedef int16_t  __s16;
    typedef int32_t  __s32;
    typedef int64_t  __s64;
#endif
```

### 2.2 类型映射表

| 类型语义 | 内核态（`__KERNEL__`） | Linux 用户态（`__linux__`） | 第三方平台（`#else`） |
|---------|------------------------|----------------------------|---------------------|
| 8位无符号 | `__u8`（`<linux/types.h>` 提供） | `__u8`（`<linux/types.h>` 提供） | `typedef uint8_t __u8` |
| 16位无符号 | `__u16`（`<linux/types.h>` 提供） | `__u16`（`<linux/types.h>` 提供） | `typedef uint16_t __u16` |
| 32位无符号 | `__u32`（`<linux/types.h>` 提供） | `__u32`（`<linux/types.h>` 提供） | `typedef uint32_t __u32` |
| 64位无符号 | `__u64`（`<linux/types.h>` 提供） | `__u64`（`<linux/types.h>` 提供） | `typedef uint64_t __u64` |
| 8位有符号 | `__s8`（`<linux/types.h>` 提供） | `__s8`（`<linux/types.h>` 提供） | `typedef int8_t __s8` |
| 16位有符号 | `__s16`（`<linux/types.h>` 提供） | `__s16`（`<linux/types.h>` 提供） | `typedef int16_t __s16` |
| 32位有符号 | `__s32`（`<linux/types.h>` 提供） | `__s32`（`<linux/types.h>` 提供） | `typedef int32_t __s32` |
| 64位有符号 | `__s64`（`<linux/types.h>` 提供） | `__s64`（`<linux/types.h>` 提供） | `typedef int64_t __s64` |

> **说明**：内核态与 Linux 用户态的类型来源相同（`<linux/types.h>`），但分为两路判定以保持编译环境语义清晰——内核构建通过 `__KERNEL__` 宏触发，用户态构建通过 `__linux__` 宏触发。二者类型定义路径相同，但条件编译入口不同。

### 2.3 预处理器宏判定逻辑

条件编译的判定基于编译器预定义宏 `__KERNEL__` 与 `__linux__`，形成三路判定：

| 宏 | 定义者 | 含义 | 示例环境 |
|----|--------|------|---------|
| `__KERNEL__` | 内核 Kbuild 构建 | 内核态编译 | `make` 内核构建 |
| `__linux__`（无 `__KERNEL__`） | GCC/Clang 预定义 | Linux 用户态编译 | 用户态 GCC/Clang |
| （无 `__linux__`、无 `__KERNEL__`） | — | 第三方平台 | macOS / Windows 等 |

**为何使用三路判定（含 `__KERNEL__`）？** 虽然内核态与 Linux 用户态都通过 `<linux/types.h>` 获取 `__u8/__u16/__u32/__u64` 定义，但使用 `__KERNEL__` 宏区分内核态有两大优势：(1) 语义清晰——CI 可独立验证内核态编译路径，而非假设"用户态通过即代表内核态通过"；(2) 可扩展性——未来若内核态需要额外的类型处理（如内核特有的 `__bitwise__` 类型属性），三路结构可直接扩展而无需重构条件编译骨架。第三方平台通过 `#else` 走 `typedef` 映射路径，与 Linux 平台完全隔离。

### 2.4 类型使用示例

以 `struct airy_log_record` 为例，展示 [SC] 头文件中的实际类型使用方式（**直接使用 `__u32` 等内核 UAPI 类型名称，不使用 `airy_*` 别名**）：

```c
/* kernel/include/uapi/linux/airymax/log_types.h —— 类型使用示例 */
#ifndef _UAPI_AIRYMAX_LOG_TYPES_H
#define _UAPI_AIRYMAX_LOG_TYPES_H

#include <linux/airymax/uapi_compat.h>  /* 类型桥接头文件 */

struct airy_log_record {
    __u32   magic;              /* offset 0:  AIRY_LOG_MAGIC */
    __u16   level;              /* offset 4:  airy_log_level enum */
    __u16   facility;           /* offset 6:  facility code */
    __u64   timestamp_ns;       /* offset 8:  monotonic ns timestamp */
    __u32   caller_id;          /* offset 16: caller identifier */
    __u32   payload_len;        /* offset 20: actual payload length */
    __u8    payload[96];        /* offset 24: log message payload */
    __u8    reserved[8];        /* offset 120: reserved */
} AIRY_ALIGNED(64);

_Static_assert(sizeof(struct airy_log_record) == 128,
	       "airy_log_record must be exactly 128 bytes");

#endif /* _UAPI_AIRYMAX_LOG_TYPES_H */
```

> **关键说明**：[SC] 头文件**统一使用 `__u8/__u16/__u32/__u64/__s8/__s16/__s32/__s64` 类型名称**，不引入 `airy_u32` 等自定义别名。这是因为：
> 1. Linux 内核 UAPI 惯例就是使用 `__u32` 等（参见 `include/uapi/linux/types.h`）
> 2. 减少类型别名层级，降低认知负担
> 3. 与内核原生代码风格一致，便于内核子树采纳

---

## §3 uapi_compat.h 设计：类型桥接的实现头文件

### 3.1 uapi_compat.h 完整实现

`uapi_compat.h` 是类型桥接的实现头文件，物理宿主为 `kernel/include/uapi/linux/airymax/uapi_compat.h`。实际代码内容如下（与 SSoT 源文件逐字节一致）：

```c
/* SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note */
/*
 * Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
 *
 * A-UAPI Compat: three-way type bridge for [SC] shared contract headers.
 *
 * This header provides portable fixed-width integer types for UAPI headers
 * that must compile identically in kernel-space, Linux user-space, and
 * non-Linux platforms (macOS, Windows agentrt).
 *
 *   #ifdef __KERNEL__      -> #include <linux/types.h>   (provides __u32 etc.)
 *   #elif defined(__linux__) -> #include <linux/types.h>  (same UAPI types)
 *   #else                  -> typedef from <stdint.h>    (macOS/Windows)
 */

#ifndef _UAPI_AIRYMAX_UAPI_COMPAT_H
#define _UAPI_AIRYMAX_UAPI_COMPAT_H

#ifdef __KERNEL__
	/* Kernel-space: use Linux kernel UAPI types */
	#include <linux/types.h>
#elif defined(__linux__)
	/* Linux user-space: <linux/types.h> is the authoritative source of
	 * __u32/__u64/... types. Include it directly rather than redefining
	 * with <stdint.h> types, which may differ in type identity (e.g. on
	 * 64-bit, uint64_t = unsigned long vs __u64 = unsigned long long). */
	#include <linux/types.h>
#else
	/* Non-Linux user-space (macOS, Windows): same mapping */
	#include <stdint.h>
	typedef int32_t   __s32;
	typedef uint32_t  __u32;
	typedef int64_t   __s64;
	typedef uint64_t  __u64;
	typedef int16_t   __s16;
	typedef uint16_t  __u16;
	typedef int8_t    __s8;
	typedef uint8_t   __u8;
#endif

/* ─── Struct Alignment Abstraction (OS-IRON-016 sanctioned exception) ──
 *
 * AIRY_ALIGNED(N) provides a compiler-agnostic way to specify struct
 * alignment in UAPI headers without directly using __attribute__.
 *
 * C11's _Alignas cannot be placed after a struct type definition (it
 * only applies to variable declarations), so compiler extensions are
 * unavoidable for struct-level alignment. This macro is the single
 * sanctioned exception to OS-IRON-016's prohibition on __attribute__
 * in UAPI headers — all other __attribute__ uses remain prohibited.
 *
 * Usage (placement after closing brace, same as __attribute__):
 *
 *   struct foo {
 *       ...
 *   } AIRY_ALIGNED(64);
 *
 * Supported compilers:
 *   GCC / Clang: __attribute__((aligned(N)))   [Linux kernel + user-space]
 *   MSVC:        __declspec(align(N))           [Windows user-space, placed
 *                                               before struct keyword via
 *                                               AIRY_ALIGNED_PREFIX]
 *   C11 fallback: _Alignas(N)                   [may not work for struct
 *                                               type definitions]
 *
 * Rationale: Linux 6.6 UAPI headers use __aligned(N) (from
 * include/uapi/linux/types.h) which expands to __attribute__((aligned(N))).
 * AirymaxOS cannot reuse __aligned(N) directly in [SC] headers because
 * macOS/Windows user-space builds do not include <linux/types.h>.
 */
#if defined(__GNUC__) || defined(__clang__)
	#define AIRY_ALIGNED(n) __attribute__((aligned(n)))
#elif defined(_MSC_VER)
	/* MSVC: __declspec(align(N)) must be placed BEFORE the struct keyword.
	 * Use AIRY_ALIGNED_PREFIX(N) struct foo { ... }; for MSVC builds.
	 * For portable code, use AIRY_ALIGNED(N) after the closing brace —
	 * MSVC will silently ignore it (no alignment), which is acceptable
	 * because Windows agentrt uses Clang, not MSVC, for [SC] headers. */
	#define AIRY_ALIGNED(n)
	#define AIRY_ALIGNED_PREFIX(n) __declspec(align(n))
#else
	/* C11 fallback: _Alignas may not enforce struct-level alignment
	 * after a type definition. This is a best-effort fallback. */
	#define AIRY_ALIGNED(n) _Alignas(n)
#endif

/* ─── [DSL] Degraded Survival Layer Fallback Block ──────────────────────
 * When AIRY_SC_FALLBACK is defined, the three-way type bridge collapses
 * to a two-way bridge: __KERNEL__ vs non-__KERNEL__. The non-Linux
 * branch (macOS/Windows) reuses the Linux user-space typedefs because
 * both rely on <stdint.h> with identical fixed-width mappings. This
 * guarantees that agentrt cross-platform builds still compile when the
 * full [SC] contract is degraded. See [DSL] §2.2.
 */
#ifdef AIRY_SC_FALLBACK
	#ifdef __KERNEL__
		/* Kernel branch unchanged — <linux/types.h> is authoritative. */
	#else
		/* All user-space branches collapse to <stdint.h> mapping. */
		#ifndef _AIRY_DSL_UAPI_COMPAT_DONE
			#define _AIRY_DSL_UAPI_COMPAT_DONE
			#include <stdint.h>
			#ifndef __s32
				typedef int32_t   __s32;
			#endif
			#ifndef __u32
				typedef uint32_t  __u32;
			#endif
			#ifndef __s64
				typedef int64_t   __s64;
			#endif
			#ifndef __u64
				typedef uint64_t  __u64;
			#endif
			#ifndef __s16
				typedef int16_t   __s16;
			#endif
			#ifndef __u16
				typedef uint16_t  __u16;
			#endif
			#ifndef __s8
				typedef int8_t    __s8;
			#endif
			#ifndef __u8
				typedef uint8_t   __u8;
			#endif
		#endif
	#endif
	#define AIRY_DSL_UAPI_BRANCHES  2  /* __KERNEL__ vs user-space */

	#warning "AIRY_SC_FALLBACK active: uapi_compat.h degraded to two-way bridge (__KERNEL__ vs user-space)"
#endif /* AIRY_SC_FALLBACK */

#endif /* _UAPI_AIRYMAX_UAPI_COMPAT_H */
```

### 3.2 设计原则：直接使用 UAPI 类型名称

`uapi_compat.h` **不定义 `airy_u32` 等自定义别名**，而是确保 `__u8/__u16/__u32/__u64/__s8/__s16/__s32/__s64` 这些 Linux UAPI 风格类型在所有平台可用。[SC] 头文件直接使用 `__u32` 等类型名称：

| 设计选择 | 采用 | 原因 |
|---------|------|------|
| 类型名称 | `__u32` 等 Linux UAPI 风格 | 与内核 UAPI 惯例一致（`include/uapi/linux/types.h`） |
| 别名层级 | 无（不引入 `airy_u32`） | 减少认知负担，避免别名与原始类型混用 |
| Linux 平台策略 | `#include <linux/types.h>` | 复用系统定义，避免 typedef 冲突 |
| 第三方平台策略 | `typedef uint8_t __u8` 等 | 将 stdint 类型映射为 UAPI 名称，源码无需条件编译 |
| 条件编译宏 | `#ifdef __KERNEL__` / `#elif defined(__linux__)` / `#else` | 三路判定：内核态、Linux 用户态、非 Linux 用户态（macOS/Windows） |

### 3.3 二进制布局一致性

类型桥接的核心保证是**二进制布局一致**——同一结构体在多环境的字节大小、字段偏移完全相同。`__u32` 在 Linux 平台由 `<linux/types.h>` 定义为 `unsigned int`（4 字节），在第三方平台由 `typedef uint32_t __u32` 定义为 `uint32_t`（4 字节），二者大小一致：

```c
/* 验证：多环境 sizeof 与 offsetof 一致 */
struct airy_log_record rec;
_Static_assert(sizeof(rec) == 128, "airy_log_record must be 128B");
_Static_assert(offsetof(struct airy_log_record, magic) == 0, "magic offset");
_Static_assert(offsetof(struct airy_log_record, level) == 4, "level offset");
_Static_assert(offsetof(struct airy_log_record, timestamp_ns) == 8, "ts offset");
```

### 3.4 历史教训：typedef 冲突问题

早期版本曾尝试在 Linux 平台上 `typedef uint64_t __u64`，导致与 `<linux/types.h>` 系统定义冲突：

```
error: conflicting types for '__u64'
  have 'uint64_t' {aka 'long unsigned int'}
  previously declared as 'unsigned long long'
```

**根因**：Linux 平台上 `<asm/types.h>` 已将 `__u64` 定义为 `unsigned long long`（在 64 位平台上），而 `uint64_t` 是 `unsigned long`。虽然二者大小都是 8 字节，但 C 类型系统视为不同类型，触发冲突。

**修复**：Linux 平台直接 `#include <linux/types.h>` 复用系统定义，不进行 `typedef`。当前 `uapi_compat.h` 采用三路条件编译（`#ifdef __KERNEL__` 内核态 / `#elif defined(__linux__)` Linux 用户态 / `#else` 非 Linux 用户态），内核态与 Linux 用户态都通过 `<linux/types.h>` 获取类型定义，非 Linux 平台通过 `<stdint.h>` + typedef 提供等价类型。

---

## §4 物理宿主：kernel/include/uapi/linux/airymax/uapi_compat.h

### 4.1 物理宿主

| 文件 | 路径 | 说明 |
|------|------|------|
| uapi_compat.h | `kernel/include/uapi/linux/airymax/uapi_compat.h` | 类型桥接实现（SSoT） |
| [SC] 核心头文件 | `kernel/include/uapi/linux/airymax/*.h` | 10 个 [SC] 核心头文件（OS-IRON-014），均 #include uapi_compat.h |
| 补充共享文件 | `kernel/include/uapi/linux/airymax/bpf_struct_ops.h` | 1 个补充共享文件（非 [SC] 核心，文件头自声明） |
| codegen 产物 | `kernel/include/uapi/linux/airymax/syscall.h` | codegen 生成产物（非 [SC] 头文件，由 syscall_gen.py 产出） |

### 4.2 与 [SC] 头文件的关系

所有 10 个 [SC] 核心头文件 + bpf_struct_ops.h 补充共享文件都在首行 `#include <linux/airymax/uapi_compat.h>`，确保类型桥接。**所有 [SC] 头文件统一使用 `__u32` 等 UAPI 类型名称**（非 `airy_u32` 别名）：

| 头文件 | 类别 | 用途 | 使用类型 |
|--------|------|------|---------|
| `error.h` | [SC] 核心 | A-UEF 错误码 | `__s32`（返回码，如 `-AIRY_EINVAL`） |
| `log_types.h` | [SC] 核心 | A-ULP 日志类型 | `__u32/__u16/__u64/__u8`（记录字段） |
| `ipc.h` | [SC] 核心 | A-IPC IPC 协议 | `__u32/__u64`（消息头） |
| `sched.h` | [SC] 核心 | A-ULS 调度扩展 | `__u32/__u64`（调度参数） |
| `memory_types.h` | [SC] 核心 | A-UMS 内存类型 | `__u32/__u64`（内存层级、GFP 标志） |
| `security_types.h` | [SC] 核心 | A-USA 安全类型 | `__u64`（cap_t badge）、`__u32`（cap_id） |
| `cognition_types.h` | [SC] 核心 | A-UCG 认知类型 | `__u32/__s32`（Q16.16 定点数） |
| `syscalls.h` | [SC] 核心 | syscall 编号 | `__u32`（编号常量） |
| `uapi_compat.h` | [SC] 核心 | 类型桥接 | （本文件，定义 `__u8` 等） |
| `lsm_types.h` | [SC] 核心 | LSM blob 类型 | `__u32/__u64`（capability slots） |
| `bpf_struct_ops.h` | 补充共享 | eBPF struct_ops | `__u32`（state、refcount） |
| `syscall.h` | codegen 产物 | syscall 入口 | `__u32/__u64`（参数类型） |

> **v4.0 修复说明**：02-P0-17 SSoT 冲突——原表（v1.0.1-fix）将 12 个文件统称为 "[SC] 头文件"，但 SSoT 权威源（`09-ssot-registry.md` OS-IRON-014）明确为 "10 个 [SC] 核心头文件 + bpf_struct_ops.h 补充共享文件"。`bpf_struct_ops.h` 文件头 L5-8 自声明 "NOT a [SC] core header"；`syscall.h` 是 codegen 产物（`@generated` 标记），非 [SC] 共享契约。已修正分类，恢复 SSoT 一致性。

### 4.3 双端共享机制

`uapi_compat.h` 与其他 [SC] 头文件一样，由 agentrt（用户态 SDK）与 agentrt-linux（内核态）双端逐字节共享。CI 通过 `sc-dual-ci.yml` 验证两端一致性：

| 校验项 | CI 脚本 | 频率 |
|--------|---------|------|
| 10+1 头文件存在性 | `sc-dual-ci.yml`（sc-validate job） | 每次 PR |
| 无物理副本 | `sc-dual-ci.yml`（sc-validate job） | 每次 PR |
| 触发 agentrt 镜像 PR | `sc-dual-ci.yml`（sc-trigger-and-await job） | 每次 PR |
| 治理文件完整性 | `mgmt-orchestrator.yml`（file-integrity job） | 每次 PR |

---

## §5 CI 校验：三路编译测试

### 5.1 三路编译测试矩阵

CI 对每个 [SC] 头文件执行三路编译测试，确保内核态、Linux 用户态、第三方平台均编译通过。由于 `uapi_compat.h` 使用 `#ifdef __KERNEL__` / `#elif defined(__linux__)` / `#else` 三路判定，CI 需分别验证三条编译路径：

| 编译路径 | 编译环境 | 编译器 | 包含宏 | 验证目标 |
|---------|---------|--------|--------|---------|
| 路径一（内核态） | 内核 Kbuild 构建 | GCC（内核构建） | `__KERNEL__` + `__linux__` | `<linux/types.h>` 提供 `__u32` 等定义（内核路径） |
| 路径二（Linux 用户态） | 用户态 Linux 构建 | GCC/Clang | `__linux__`（预定义，无 `__KERNEL__`） | `<linux/types.h>` 提供 `__u32` 等定义（用户态路径） |
| 路径三（第三方平台） | 模拟非 Linux 环境 | GCC（`-U__linux__`） | （无 `__linux__`、无 `__KERNEL__`） | `typedef uint32_t __u32` 等映射生效 |

> **内核态编译说明**：内核构建由 `kernel` 子仓的 Kbuild 系统处理。CI 通过 `make C=2` 或独立内核模块编译验证 `__KERNEL__` 路径，确保 [SC] 头文件在内核构建环境下编译通过。这与用户态 Linux 编译路径（路径二）形成互补——二者虽类型来源相同（`<linux/types.h>`），但条件编译入口不同，需独立验证。

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
      - name: Compile as kernel-space
        run: |
          # 内核态编译：__KERNEL__ 由 Kbuild 定义
          # 验证 <linux/types.h> 在内核构建下提供 __u32 等定义
          gcc -D__KERNEL__ -D__linux__ \
              -Ikernel/include -Ikernel/include/uapi \
              -c kernel/include/uapi/linux/airymax/*.h -o /dev/null

  linux-userspace-compile:
    name: Linux userspace path (__linux__)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Compile as Linux userspace
        run: |
          # Linux 用户态编译：__linux__ 由 GCC 预定义，无 __KERNEL__
          # 验证 <linux/types.h> 在用户态下提供 __u32 等定义
          gcc -Ikernel/include -c \
              kernel/include/uapi/linux/airymax/*.h -o /dev/null

  third-party-compile:
    name: Third-party path (no __linux__)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Compile as third-party (macOS/Windows simulation)
        run: |
          # 第三方编译：取消 __linux__ 宏定义，模拟非 Linux 环境
          # 验证 typedef uint32_t __u32 等映射生效
          gcc -U__linux__ -std=c99 \
              -Ikernel/include -c \
              kernel/include/uapi/linux/airymax/*.h -o /dev/null

  type-size-check:
    name: Type size consistency
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verify type sizes
        run: |
          # 验证 __u32 在三路环境都是 4 字节
          cat > /tmp/check.c << 'EOF'
          #include "airymax/uapi_compat.h"
          #include <assert.h>
          int main() {
              assert(sizeof(__u32) == 4);
              assert(sizeof(__u64) == 8);
              assert(sizeof(__u16) == 2);
              assert(sizeof(__u8) == 1);
              assert(sizeof(__s32) == 4);
              assert(sizeof(__s64) == 8);
              return 0;
          }
          EOF
          # 三路编译并运行
          gcc -D__KERNEL__ -D__linux__ -Ikernel/include /tmp/check.c -o /tmp/check_kernel
          gcc -Ikernel/include /tmp/check.c -o /tmp/check_linux
          gcc -U__linux__ -std=c99 -Ikernel/include /tmp/check.c -o /tmp/check_3rd
          /tmp/check_kernel && /tmp/check_linux && /tmp/check_3rd
```

### 5.3 CI 校验清单

| 校验项 | 校验内容 | 失败处理 |
|--------|---------|---------|
| 三路编译通过 | 内核态 + Linux 用户态 + 第三方均无编译错误 | 阻止 PR 合并 |
| 类型大小一致 | `__u32/__u64` 三路环境大小相同 | 阻止 PR 合并 |
| 字段偏移一致 | 结构体字段偏移三路环境相同 | 阻止 PR 合并 |
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
| v1.0 | 2026-07-17 | 初始版本：[SC] 类型桥接规范；三路条件编译模型（`#ifdef __KERNEL__` / `#elif defined(__linux__)` / `#else`）；uapi_compat.h 设计（虚构 airy_* 类型别名）；物理宿主 kernel/include/uapi/linux/airymax/uapi_compat.h；CI 三路编译测试（内核/用户 Linux/第三方）；二进制布局一致性保证（_Static_assert） |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-7 铁律，所有文档版本号统一为 v1.0.1 |
| v1.0.1-fix | 2026-07-26 | **文档-代码策略对立修复**（v3.5 审查 P0）：反映 uapi_compat.h 实际策略——三路条件编译（`#ifdef __KERNEL__` / `#elif defined(__linux__)` / `#else`）、直接使用 `__u32` 等 UAPI 类型名称（非 `airy_u32` 别名）、Linux 平台 `#include <linux/types.h>` 复用系统定义（避免 typedef 冲突）、第三方平台 `typedef uint8_t __u8` 映射；添加 §3.4 历史教训记录 typedef 冲突问题；CI 三路编译（内核态 + Linux 用户态 + 第三方） |
| v1.0.2 | 2026-07-26 | **02-P0-17 SSoT 冲突修复**（v4.0 审查）：§4.1/§4.2 头文件分类修正——原表将 12 个文件统称 "[SC] 头文件" 与 SSoT 权威源（`09-ssot-registry.md` OS-IRON-014 "10 个核心 + bpf_struct_ops.h 补充"）冲突。已修正为三类分类：10 个 [SC] 核心头文件 + bpf_struct_ops.h 补充共享文件（文件头自声明非核心）+ syscall.h codegen 产物（@generated 标记，非 [SC] 契约）。§4.3 CI 校验项 "12 头文件" → "10+1 头文件"。 |
| v1.0.3 | 2026-07-29 | **P0-NEW-04/05/06/07/16 修复**：§2.1/§2.2/§2.3/§5/SSoT 声明由错误的"二路条件编译（不使用 `__KERNEL__`）"修正为与实际 `uapi_compat.h` 代码一致的"三路条件编译（`#ifdef __KERNEL__` / `#elif defined(__linux__)` / `#else`）"；§5 CI 由二路编译改为三路编译（增加内核态 `__KERNEL__` 路径）；§1.1 头文件计数由"共 12 个"细化为"10 个 [SC] 核心 + bpf_struct_ops.h 补充 + syscall.h codegen"；v1.0.1-fix 版本笔记描述由"二路"修正为"三路"。 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] 类型桥接规范 | v1.0.1 | 2026-07-26
