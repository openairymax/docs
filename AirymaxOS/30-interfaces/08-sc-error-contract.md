Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# [SC] error.h 二进制契约
> **文档定位**：UEF（统一错误码与故障定义体系）模块的 [SC] 共享契约权威定义\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §4\
> **设计依据**：[15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §2.3（错误码 SSoT）+ §4.2.1（UEF 设计）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax 错误码与故障码体系** 的唯一权威源。Error 码空间分配、Fault 码空间分配、`AIRY_E*` 命名风格、`AIRY_FAULT_*` 命名风格、[DSL] 降级块均以本文件为唯一权威定义。物理宿主为 `kernel/include/airymax/error.h`，agentrt 与 agentrt-linux 物理共享同一份头文件，逐字节相同。
>
> 废弃风格声明：`AIRY_ERR_*`（详细前缀）已废弃，全部迁移为 `AIRY_E*`；`EAIRY_*`（errno 风格前缀）已废弃。本契约遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) 的技术选型（方案 C-Prime + 纯 C LSM 不使用 BPF LSM + IORING_OP_URING_CMD + registered buffer + mmap 不使用 page flipping + alloc_pages + mmap 不使用 DMA 一致性内存）。

---

## 文档信息卡

- **目标读者**：agentrt-linux 内核开发者、agentrt 用户态开发者、CI 维护者
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) UEF 模块定位、[06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) [SC] 层概念
- **预计阅读时间**：25 分钟
- **核心概念**：Error（负数可恢复）、Fault（正数 0x1000+ 不可恢复）、[SC] 契约、CI 逐字节校验
- **复杂度标识**：中级

---

## §1 契约概述：UEF 模块的 [SC] 共享契约

UEF（Unified Error & Fault，统一错误码与故障定义体系）是 Airymax Unify Design 的基础模块，其 [SC] 共享契约头文件 `error.h` 是整个体系最底层的数据契约。

### 1.1 契约范围

本契约定义以下内容，agentrt 与 agentrt-linux 双端必须逐字节一致：

1. `airy_err_t` 类型定义（错误码类型）
2. `AIRY_E*` 错误码宏（Error 码，负数空间）
3. `AIRY_FAULT_*` 故障码宏（Fault 码，正数 0x1000+ 空间）
4. `AIRY_LOG_MAGIC` 等相关魔数（仅与错误码语义绑定的魔数）
5. `#ifdef AIRY_SC_FALLBACK` 降级块

### 1.2 物理宿主

| 属性 | 值 |
|------|-----|
| 物理路径 | `kernel/include/airymax/error.h` |
| 共享层级 | [SC] 共享契约层（IRON-9 v3 第一层） |
| 共享方式 | 物理共享，逐字节相同 |
| CI 校验 | `sc-dual-ci.yml` 逐字节校验 |
| agentrt 引用方式 | `-I` 编译选项 + `#include <airymax/error.h>` |

### 1.3 类型约束

遵循 [SC] 共享契约层通用约束：

- 使用内核 UAPI 类型 `int32_t` / `__s32`（不使用 `int`）
- 禁止使用 `float` 类型
- 禁止依赖任何非 UAPI 头文件
- 使用 `__attribute__((packed))` 仅在需要紧凑布局时（错误码宏不涉及结构体，无需 packed）

---

## §2 Error 码空间分配

Error 码占据负数空间 `[-300, -1]`，按来源分 5 个子空间，每个子空间有明确的值域、来源与语义。

### 2.1 Error 码空间总表

| 子空间 | 值域 | 来源 | 数量 | 命名风格 |
|--------|------|------|------|---------|
| POSIX 码 | `[-1, -40]` | 对齐 Linux errno | 40 | `AIRY_E*`（对齐 `E*` errno 名） |
| IPC 码 | `[-41, -70]` | UIPF 协议层 | 30 | `AIRY_EIPC_*` |
| Capability 码 | `[-71, -100]` | 安全子系统 | 30 | `AIRY_ECAP_*` |
| [SC] 码 | `[-101, -200]` | [SC] 共享契约层 | 100 | `AIRY_ESC_*` |
| [DSL] 码 | `[-201, -300]` | [DSL] 降级生存层 | 100 | `AIRY_EDSL_*` |

### 2.2 POSIX 码 `[-1, -40]`

对齐 Linux 6.6 标准 errno，前 40 个为 POSIX 兼容码。完整清单见 `kernel/include/airymax/error.h`，部分关键码：

```c
#define AIRY_EOK            0       /* 成功（非负数，特殊） */
#define AIRY_EPERM          (-1)    /* 权限不足 */
#define AIRY_ENOENT         (-2)    /* 不存在 */
#define AIRY_EINTR          (-4)    /* 被信号中断 */
#define AIRY_EIO            (-5)    /* I/O 错误 */
#define AIRY_ENOMEM         (-12)   /* 内存不足 */
#define AIRY_EACCES         (-13)   /* 拒绝访问 */
#define AIRY_EFAULT         (-14)   /* 错误地址 */
#define AIRY_EBUSY          (-16)   /* 设备忙 */
#define AIRY_EEXIST         (-17)   /* 已存在 */
#define AIRY_EINVAL         (-22)   /* 参数无效 */
#define AIRY_ENOSPC         (-28)   /* 空间不足 */
#define AIRY_EPIPE          (-32)   /* 管道破裂 */
#define AIRY_ENOSYS         (-38)   /* 功能未实现 */
#define AIRY_ENOTEMPTY      (-39)   /* 目录非空 */
#define AIRY_ELOOP          (-40)   /* 符号链接循环 */
```

### 2.3 IPC 码 `[-41, -70]`

UIPF 协议层专用错误码，覆盖 IPC 协议、Ring Buffer、io_uring_cmd 等场景：

```c
#define AIRY_EIPC_RING_FULL     (-41)   /* Ring Buffer 已满 */
#define AIRY_EIPC_RING_EMPTY    (-42)   /* Ring Buffer 为空 */
#define AIRY_EIPC_MAGIC         (-43)   /* magic 校验失败 */
#define AIRY_EIPC_OPCODE        (-44)   /* 未知 opcode */
#define AIRY_EIPC_FROZEN        (-45)   /* IPC 队列已冻结 */
#define AIRY_EIPC_PAYLOAD       (-46)   /* payload 长度非法 */
#define AIRY_EIPC_REGISTERED    (-47)   /* buffer 未注册 */
#define AIRY_EIPC_URING_CMD     (-48)   /* io_uring_cmd 失败 */
/* ... 共 30 个 IPC 码，[-41, -70] */
```

### 2.4 Capability 码 `[-71, -100]`

安全子系统专用错误码，覆盖 Capability 校验、纯 C LSM 检查等场景：

```c
#define AIRY_ECAP_MISSING       (-71)   /* Capability 缺失 */
#define AIRY_ECAP_REVOKED       (-72)   /* Capability 已撤销 */
#define AIRY_ECAP_EXPIRED       (-73)   /* Capability 已过期 */
#define AIRY_ECAP_MISMATCH      (-74)   /* Capability 不匹配 */
#define AIRY_ECAP_LSM_DENIED    (-75)   /* 纯 C LSM 拒绝 */
#define AIRY_ECAP_RADIX_MISS    (-76)   /* radix tree 查找失败 */
#define AIRY_ECAP_STATIC_KEY    (-77)   /* static_key 禁用 */
/* ... 共 30 个 Capability 码，[-71, -100] */
```

### 2.5 [SC] 码 `[-101, -200]`

[SC] 共享契约层专用错误码，覆盖配置版本、头文件校验等场景：

```c
#define AIRY_ESC_CFGVERSION     (-101)  /* 配置版本不匹配 */
#define AIRY_ESC_HASH_MISMATCH  (-102)  /* [SC] 头文件 hash 不匹配 */
#define AIRY_ESC_FALLBACK       (-103)  /* 已降级到 [DSL] 模式 */
#define AIRY_ESC_DUAL_CI        (-104)  /* 双端 CI 校验失败 */
#define AIRY_ESC_UAPI_TYPE      (-105)  /* UAPI 类型违规 */
#define AIRY_ESC_FLOAT_FORBIDDEN (-106) /* 禁止使用 float */
#define AIRY_ESC_PACKED_REQUIRED (-107) /* 缺少 __packed */
/* ... 共 100 个 [SC] 码，[-101, -200] */
```

### 2.6 [DSL] 码 `[-201, -300]`

[DSL] 降级生存层专用错误码，覆盖降级模式下的特殊场景：

```c
#define AIRY_EDSL_ACTIVE        (-201)  /* [DSL] 降级模式已激活 */
#define AIRY_EDSL_FAULT_DISABLED (-202) /* Fault 码已禁用 */
#define AIRY_EDSL_RING_UNAVAILABLE (-203) /* Ring Buffer 不可用 */
#define AIRY_EDSL_PRINTK_FALLBACK (-204) /* 已回退到 printk 原生 */
#define AIRY_EDSL_EEVDF_ONLY    (-205)  /* 仅 EEVDF 调度可用 */
#define AIRY_EDSL_CAP_MINIMAL   (-206)  /* 仅 POSIX capability 可用 */
/* ... 共 100 个 [DSL] 码，[-201, -300] */
```

---

## §3 Fault 码空间分配

Fault 码占据正数空间 `[0x1000, 0x1FFF]`，从 `0x1000` 起步以避免与 errno 正值（如 `EPERM=1`）冲突。每个 Fault 对应一个不可恢复的异常场景，由 Micro-Supervisor 直接接管。

### 3.1 Fault 码定义

```c
/* kernel/include/airymax/error.h —— Fault 码（正数 0x1000+，不可恢复） */
#define AIRY_FAULT_CAP_FAULT        0x1001   /* Capability 故障 */
#define AIRY_FAULT_VM_FAULT         0x1002   /* 虚拟内存故障 */
#define AIRY_FAULT_IPC_FAULT        0x1003   /* IPC 故障 */
#define AIRY_FAULT_TIMEOUT          0x1004   /* 超时故障 */
#define AIRY_FAULT_ABNORMAL_CAP     0x1005   /* 异常 Capability */
/* 预留 0x1006 ~ 0x1FFF */
```

### 3.2 Fault 码触发与处置

| Fault 码 | 触发场景 | Micro-Supervisor 处置 | 后续路径 |
|---------|---------|---------------------|---------|
| `AIRY_FAULT_CAP_FAULT` | 检测到伪造/越权 Capability | 立即冻结 IPC 队列 + eventfd 通知 | Macro-Supervisor 裁决 |
| `AIRY_FAULT_VM_FAULT` | 共享页映射损坏 | 立即标记 Ring 不可用 | 回退到 printk_safe |
| `AIRY_FAULT_IPC_FAULT` | Ring Buffer 元数据损坏 | 立即冻结对应 Ring | 重启 Ring（保留消息） |
| `AIRY_FAULT_TIMEOUT` | Agent 心跳超时且未响应冻结 | 强制 STOPPED 态 | Macro-Supervisor 裁决 |
| `AIRY_FAULT_ABNORMAL_CAP` | Capability 树完整性校验失败 | 立即终止 Agent（SIGKILL） | 进入 DEAD 态 |

### 3.3 Error 与 Fault 的关系

Error 与 Fault 在码空间上严格不重叠（Error 负数，Fault 正数 0x1000+），调用方可通过符号判断区分：

```c
static inline bool airy_is_fault(airy_err_t err) {
    return err >= 0x1000 && err <= 0x1FFF;
}
static inline bool airy_is_error(airy_err_t err) {
    return err < 0;
}
```

函数正常返回 `AIRY_EOK`（0），可恢复错误返回负数 Error，不可恢复故障返回正数 Fault 并触发 Fault Handler。

---

## §4 CI 逐字节校验：sc-dual-ci.yml 验证两端 error.h 相同

### 4.1 校验机制

`sc-dual-ci.yml` 是 [SC] 双端一致性 CI 工作流，对 `error.h` 进行逐字节校验：

```yaml
# .github/workflows/sc-dual-ci.yml
name: [SC] Dual CI
on: [push, pull_request]
jobs:
  sc-error-h-validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: spharx/agentrt-linux
          path: agentrt-linux
      - uses: actions/checkout@v4
        with:
          repository: spharx/agentrt
          path: agentrt
      - name: Verify error.h byte-for-byte identical
        run: |
          diff agentrt-linux/kernel/include/airymax/error.h \
               agentrt/commons/include/airymax/error.h
          if [ $? -ne 0 ]; then
            echo "::error::error.h dual-end mismatch detected"
            exit 1
          fi
      - name: Verify [DSL] fallback block hash
        run: |
          scripts/airymax/check-uapi-compiler-agnostic.sh \
            agentrt-linux/kernel/include/airymax/error.h
```

### 4.2 校验失败的处理

当 `sc-dual-ci.yml` 校验失败时：

1. CI 自动设置 `AIRY_SC_FALLBACK` 并重新构建（验证降级路径可编译）
2. CI 阻止合并，要求双端维护者同步评审
3. 修复后重新触发 CI，校验通过方可合并

### 4.3 编译器无关性校验

除逐字节校验外，`error.h` 还需通过编译器无关性校验（`check-uapi-compiler-agnostic.sh`），确保在 GCC / Clang / 不同版本下布局一致。校验项包括：

- 无 `sizeof` 依赖的结构体
- 无编译器扩展（`__attribute__` 仅限 `packed`）
- 无 `enum` 大小假设（错误码全部用 `#define`，不用 `enum`）

---

## §5 [DSL] 降级块：#ifdef AIRY_SC_FALLBACK → 仅保留 38 个 POSIX 码

### 5.1 降级块设计

`error.h` 底部包含 `#ifdef AIRY_SC_FALLBACK` 降级块，详细设计见 [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2。降级块生效时：

- **保留**：38 个 POSIX errno 负值（`AIRY_EPERM=-1` ~ `AIRY_ELOOP=-40`）
- **保留**：1 个配置版本码 `AIRY_ESC_CFGVERSION=-101`（降级块内别名 `AIRY_ECFGVERSION`）
- **禁用**：IPC 码、Capability 码、[SC] 扩展码、[DSL] 码
- **禁用**：全部 Fault 码（降级模式下故障统一走 Panic）

### 5.2 降级块代码骨架

```c
/* kernel/include/airymax/error.h —— 文件底部 */
#ifdef AIRY_SC_FALLBACK
#ifndef _AIRY_ERROR_FALLBACK_H
#define _AIRY_ERROR_FALLBACK_H
/* 仅保留 38 个 POSIX 码 + AIRY_ECFGVERSION，详见 11-degraded-survival-layer.md §2.1 */
#define AIRY_EPERM          (-1)
/* ... */
#define AIRY_ELOOP          (-40)
#define AIRY_ECFGVERSION    (-101)   /* [DSL] 唯一保留的非 POSIX 码 */
#warning "AIRY_SC_FALLBACK active: only 38 POSIX codes + AIRY_ECFGVERSION available"
#endif /* _AIRY_ERROR_FALLBACK_H */
#endif /* AIRY_SC_FALLBACK */
```

### 5.3 降级块的独立性

降级块设计为**自包含**——不依赖 `error.h` 主体的任何符号。即使 `error.h` 主体损坏，只要降级块本身完整（通过 `.fallback_hashes` 校验），系统仍可降级启动。

---

## §6 物理宿主：kernel/include/airymax/error.h

### 6.1 文件位置

| 属性 | 值 |
|------|-----|
| 内核侧路径 | `agentrt-linux/kernel/include/airymax/error.h` |
| agentrt 侧路径 | `agentrt/commons/include/airymax/error.h`（符号链接或 git submodule 引用内核侧物理文件） |
| 实际物理文件 | 仅一份，位于 `agentrt-linux/kernel/include/airymax/error.h` |
| 文件编码 | UTF-8 without BOM |
| 换行符 | LF（Unix） |

### 6.2 文件结构

`error.h` 文件结构自上而下：

1. 版权头 + 文件说明注释
2. 头文件守卫 `#ifndef _AIRY_ERROR_H`
3. `airy_err_t` 类型定义
4. POSIX 码 `[-1, -40]`
5. IPC 码 `[-41, -70]`
6. Capability 码 `[-71, -100]`
7. [SC] 码 `[-101, -200]`
8. [DSL] 码 `[-201, -300]`
9. Fault 码 `[0x1000, 0x1FFF]`
10. 内联辅助函数 `airy_is_fault()` / `airy_is_error()`
11. `#ifdef AIRY_SC_FALLBACK` 降级块
12. 头文件守卫结束 `#endif`

### 6.3 变更流程

`error.h` 的任何变更必须遵循 [SC] 共享契约变更流程：

1. 在本文件（`08-sc-error-contract.md`）提出变更提案
2. agentrt 与 agentrt-linux 双方维护者评审
3. 同步修改 `kernel/include/airymax/error.h` 物理头文件
4. `sc-dual-ci.yml` 双端校验通过
5. 更新本契约文档版本号

---

## §7 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §4 —— UEF 模块总纲
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §2 —— [DSL] 降级块机制
- [06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md) §2 —— [SC] 共享契约层
- [02-ipc-protocol.md](02-ipc-protocol.md) —— IPC 码使用场景
- [03-capability-model.md](../110-security/03-capability-model.md) —— Capability 码使用场景
- [15-comprehensive-correction-plan.md](../../docs-closed/agentrt-linux/00-reviews/_review_v2.2/15-comprehensive-correction-plan.md) §2.3 / §4.2.1 —— 设计依据

---

## §8 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：UEF [SC] error.h 二进制契约；Error 码 5 子空间分配（POSIX/IPC/Capability/[SC]/[DSL]）；Fault 码 0x1000+ 分配；CI 逐字节校验；[DSL] 降级块；物理宿主 `kernel/include/airymax/error.h` |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | [SC] error.h 二进制契约 | v1.0 | 2026-07-17
