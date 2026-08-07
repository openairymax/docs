Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）错误消息国际化设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）错误消息国际化工程设计文档，覆盖错误码（airy_err_t）与本地化消息分离设计、错误码注册表 SSoT、多语言消息映射\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 概述

### 1.1 设计目标

agentrt-linux（AirymaxOS）作为面向全球开发者和用户的智能体操作系统发行版，其错误消息体系是开发者体验与可追溯性的关键。本文档定义错误消息国际化工程设计，目标是：

1. **错误码与消息分离**：错误码（`airy_err_t`）作为机器可读契约，本地化消息作为人类可读呈现，二者严格分离
2. **错误码注册表 SSoT**：所有错误码唯一定义于 `include/uapi/linux/airymax/error.h`（IRON-9 v3 [SC] 共享契约层），任何文档、代码不得另起定义
3. **多语言消息映射**：每个错误码映射到多语言消息表，支持 zh_CN、en_US、ja_JP 等 locale
4. **跨端错误码同源**：agentrt-linux 内核态与 agentrt 用户态运行时错误码完全一致，实现跨端错误可追溯

### 1.2 基线约束

本设计严格遵循 Linux 6.6 内核基线，并执行以下硬约束：

| 约束项 | 取值 | 说明 |
|--------|------|------|
| 错误码体系 | 双错误码体系 | C 负整数（首要）+ SDK 十六进制（次要） |
| 错误码 SSoT | `include/uapi/linux/airymax/error.h` | 唯一定义源，[SC] 共享契约层 |
| 内核态消息 | 英文 | `printk` 永远英文，错误码通过 errno 暴露 |
| 用户态本地化 | gettext | 错误码 → msgid 映射，由 gettext 查找本地化消息 |
| 字符串安全 | `strscpy` | 错误消息拷贝必须用 `strscpy` |

### 1.3 术语规范

本文档严格遵循术语规范：agentrt（用户态运行时）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）。错误码定义共享于 [SC] 共享契约层（两端完全共享代码），错误消息呈现独立于两端实现。文中不使用 主流 Linux 发行版/主流 Linux 发行版 字样，错误码体系对照一律表述为"主流 Linux 发行版"。

---

## 2. 双错误码体系

### 2.1 体系概览

agentrt-linux 采用双错误码体系，分别适用于不同场景：

| 体系 | 适用场景 | 格式 | 权威源 |
|------|----------|------|--------|
| **C 错误码体系**（首要） | C 内核和 Daemon 层 | `AIRY_EOK=0`、`AIRY_EINVAL=5`（**正数幅值**，返回 `-AIRY_EINVAL` 产生负值） | `include/uapi/linux/airymax/error.h` |
| **SDK 十六进制体系**（次要，规划态） | SDK 和外部接口 | `0x0000`-`0x7FFF` 分段（设计规划，未在 [SC] 注册） | `include/uapi/linux/airymax/error.h` |

### 2.2 C 错误码体系分段（正数幅值）

C 错误码体系按子系统分段，定义于 [SC] 共享契约层：

```c
/* include/uapi/linux/airymax/error.h [SC] — 错误码 SSoT（单一数据源）
 * airy_err_t 为 __s32；AIRY_E* 常量为正数幅值（如 POSIX errno），
 * 调用方返回 -AIRY_E* 产生负错误值。完整定义含 [DSL] AIRY_SC_FALLBACK
 * 降级块（38 个 POSIX 码映射到 5 核心码），本节仅摘录分段结构。
 */
typedef __s32 airy_err_t;

/* 成功 */
#define AIRY_EOK              0

/* POSIX 对齐段（1-40，正数幅值） */
#define AIRY_EACCES           1     /* Operation not permitted */
#define AIRY_EEXIST           2     /* File exists */
#define AIRY_EFAULT           3     /* Bad address */
#define AIRY_EINTR            4     /* Interrupted system call */
#define AIRY_EINVAL           5     /* Invalid argument */
#define AIRY_EIO              6     /* I/O error */
#define AIRY_EISDIR           7     /* Is a directory */
#define AIRY_ENOENT           8     /* No such file or directory */
#define AIRY_ENOMEM           9     /* Out of memory */
#define AIRY_ENOSPC           10    /* No space left on device */
#define AIRY_ENOTSUP          11    /* Operation not supported */
#define AIRY_EPERM            12    /* Operation not permitted (POSIX) */
#define AIRY_ERANGE           13    /* Result too large */
#define AIRY_EBUSY            16    /* Device or resource busy */
#define AIRY_ECANCELED        19    /* Operation canceled */
#define AIRY_EAGAIN           35    /* Try again */

/* 子系统段（41-240，正数幅值）：
 *   IPC 41-70 / Capability 71-100 / Config 101-120 /
 *   Sched+Lifecycle 121-140 / Mem 141-160 / Cognition 161-180 /
 *   Log 181-200 / Object 201-220 / Syscall 221-240
 */
#define AIRY_EIPC_FROZEN      53    /* 示例：IPC 段（Ring 冻结） */
#define AIRY_ESCHED_DEADLINE  123   /* 示例：Sched 段（Deadline missed） */
#define AIRY_EMEM_OOM         148   /* 示例：Mem 段（agent-scoped OOM） */

/* 保留段（241-300）：未来子系统（含 i18n 专用码）在此申请分配，
 * 需同步更新 30-interfaces/08-sc-error-contract.md。
 * 注意：i18n 错误码（locale/encoding 相关）尚未在 [SC] 注册 ——
 * 标注为"待 [SC] 注册"，禁止在文档中虚构 -900/-901 等负值 i18n 段。
 * 原文档虚构的 -1~-15 负值基础段、-100~-899 各子系统负值段
 * （AIRY_SYS_*/AIRY_KERN_*/AIRY_SVC_*/AIRY_LLM_*/AIRY_EXEC_*/
 * AIRY_MEM_*/AIRY_SEC_*/AIRY_PROTO_*，含 AIRY_KERN_ENODIE=-210）
 * 与 -1000~-1099 发行版段在 [SC] error.h 中均不存在，已废弃。
 */
```

> **修正说明**：旧文档将 `AIRY_KERN_ENODIE` 等符号定为 -200 段负值，与性能文档中虚构的 `AIRY_E_SCHED_TIMEOUT=-210` 数值冲突；[SC] error.h 的 A-ULS Sched/Lifecycle 段（121-140）与超节点相关错误码尚未注册，上述虚构符号一律废弃，以 error.h 实际定义为准。

### 2.3 SDK 十六进制体系分段

> **规划态声明**：以下 SDK 十六进制分段（`0x0000`-`0x7FFF`）为 **SDK 层设计规划**，未在 [SC] `error.h` 中注册；待注册时需分配独立命名空间并在 `30-interfaces/08-sc-error-contract.md` 登记，不得与 C 错误码段（1-300）混用。

SDK 十六进制体系用于 SDK 和外部接口（Python/Go/Rust/Java SDK）：

```c
/* include/uapi/linux/airymax/error.h [SC] （续） */

/* SDK 错误码（0x0000 - 0x7FFF） */
#define AIRY_ERROR_OK             0x0000
#define AIRY_ERROR_GENERIC        0x0001
#define AIRY_ERROR_INVALID_PARAM  0x0002

/* 通用段 0x0000 - 0x0FFF */
#define AIRY_ERROR_SYS_BASE       0x0100

/* 内核段 0x1000 - 0x1FFF */
#define AIRY_ERROR_KERN_BASE      0x1000

/* 服务段 0x2000 - 0x2FFF */
#define AIRY_ERROR_SVC_BASE       0x2000

/* LLM 段 0x3000 - 0x3FFF */
#define AIRY_ERROR_LLM_BASE       0x3000

/* 安全段 0x4000 - 0x4FFF */
#define AIRY_ERROR_SEC_BASE       0x4000
```

### 2.4 双体系映射

> **规划态声明**：以下映射函数为**设计示意**，所引符号（`AIRY_EGENERIC` / `AIRY_KERN_EBPF` / `AIRY_SVC_EGATEWAY` 等）在 [SC] `error.h` 中**不存在**，待 SDK 十六进制体系在 [SC] 注册后按实际符号实现：

双错误码体系之间的映射函数定义于 [SC] 共享契约层：

```c
/* include/uapi/linux/airymax/error.h [SC] （续） */

static inline u16 airy_err_c_to_sdk(airy_err_t err)
{
    if (err >= 0)
        return AIRY_ERROR_OK;

    /* C 负整数 → SDK 十六进制 */
    switch (err) {
    case AIRY_EGENERIC:        return AIRY_ERROR_GENERIC;
    case AIRY_EINVAL:          return AIRY_ERROR_INVALID_PARAM;
    case AIRY_KERN_EBPF:       return AIRY_ERROR_KERN_BASE + 0x00;
    case AIRY_KERN_ESCHED:     return AIRY_ERROR_KERN_BASE + 0x01;
    case AIRY_KERN_EIPC:       return AIRY_ERROR_KERN_BASE + 0x02;
    case AIRY_SVC_EGATEWAY:    return AIRY_ERROR_SVC_BASE + 0x00;
    case AIRY_SVC_ELLM:        return AIRY_ERROR_SVC_BASE + 0x01;
    /* ... 完整映射表 */
    default:                      return AIRY_ERROR_GENERIC;
    }
}

static inline airy_err_t airy_err_sdk_to_c(u16 sdk_err)
{
    /* SDK 十六进制 → C 负整数（反向映射） */
    switch (sdk_err) {
    case AIRY_ERROR_OK:           return AIRY_EOK;
    case AIRY_ERROR_GENERIC:      return AIRY_EGENERIC;
    case AIRY_ERROR_INVALID_PARAM: return AIRY_EINVAL;
    /* ... 完整映射表 */
    default:                       return AIRY_EGENERIC;
    }
}
```

### 2.5 体系使用规范

| 场景 | 体系 | 示例 |
|------|------|------|
| C 内核代码 | C 错误码（正数幅值） | `return -AIRY_ENOENT;`（-8） |
| Daemon C 代码 | C 错误码（正数幅值） | `return -AIRY_EIPC_TIMEOUT;`（-52） |
| Python SDK | SDK 十六进制 | `raise AgentrtError(0x1002)` |
| Go SDK | SDK 十六进制 | `return AgentrtErrKernIPC` |
| Rust SDK | SDK 十六进制 | `AgentrtErr::KernIpc` |
| Java SDK | SDK 十六进制 | `throw new AgentrtException(0x1002)` |
| JSON-RPC 错误响应 | SDK 十六进制 | `{"code": 4098, "message": "..."}` |

**禁止**：C 内核代码中使用十六进制错误码；SDK 中使用负整数错误码。

---

## 3. 错误码注册表 SSoT

### 3.1 SSoT 原则

Single Source of Truth（SSoT）原则要求：所有错误码唯一定义于 `include/uapi/linux/airymax/error.h`。任何文档、代码、SDK 不得另起定义，必须引用此权威源。

```
权威源（SSoT）：
  include/uapi/linux/airymax/error.h  [SC] 共享契约层
       ↓
  生成器（airymaxos-error-gen）
       ↓
  ┌─────────────────────────────────────────┐
  ↓              ↓              ↓           ↓
内核 .o        Daemon .o     Python SDK   Go SDK
                            Rust SDK      Java SDK
                            JSON-RPC     .po/.mo 翻译
```

### 3.2 错误码生成器

agentrt-linux 提供 `airymaxos-error-gen` 工具，从 `include/uapi/linux/airymax/error.h` 自动生成各语言的错误码绑定：

```python
#!/usr/bin/env python3
# system/error_gen.py [IND]
"""从 include/uapi/linux/airymax/error.h 生成各语言错误码绑定"""

import re
import sys
from pathlib import Path

ERROR_HEADER = "include/uapi/linux/airymax/error.h"

PATTERN = re.compile(
    r'#define\s+(AIRY_[A-Z_]+)\s+(\(?-?\d+\)?)\s*/\*\s*(.*?)\s*\*/'
)

def parse_errors(header_path):
    errors = []
    with open(header_path) as f:
        for line in f:
            m = PATTERN.match(line.strip())
            if m:
                name, value, desc = m.groups()
                errors.append({
                    "name": name,
                    "value": int(value.strip("()")),
                    "desc": desc,
                })
    return errors

def gen_python(errors, out_path):
    """生成 Python 错误码枚举"""
    with open(out_path, "w") as f:
        f.write("# AUTO-GENERATED. DO NOT EDIT.\n")
        f.write("# Source: include/uapi/linux/airymax/error.h [SC]\n\n")
        f.write("from enum import IntEnum\n\n")
        f.write("class AgentrtError(IntEnum):\n")
        for e in errors:
            f.write(f'    {e["name"]} = {e["value"]}  # {e["desc"]}\n')

def gen_po(errors, out_path):
    """生成 .po 模板的错误码 msgid"""
    with open(out_path, "w") as f:
        f.write('# AUTO-GENERATED. DO NOT EDIT.\n')
        f.write('msgid ""\n')
        f.write('msgstr ""\n')
        f.write('"Content-Type: text/plain; charset=UTF-8\\n"\n\n')
        for e in errors:
            f.write(f'# {e["desc"]}\n')
            f.write(f'msgid "{e["name"]}"\n')
            f.write('msgstr ""\n\n')

if __name__ == "__main__":
    errors = parse_errors(ERROR_HEADER)
    gen_python(errors, "agentrt/sdk/python/agentrt/errors.py")
    gen_po(errors, "po/airy_errors.pot")
    print(f"生成 {len(errors)} 个错误码绑定")
```

### 3.3 生成产物示例

Python SDK 自动生成的错误码：

```python
# agentrt/sdk/python/agentrt/errors.py
# AUTO-GENERATED. DO NOT EDIT.
# Source: include/uapi/linux/airymax/error.h [SC]

from enum import IntEnum

class AgentrtError(IntEnum):
    AIRY_EOK = 0  # 成功
    AIRY_EINVAL = 5  # 参数非法（示意值：正数幅值，调用方返回 -AIRY_EINVAL）
    AIRY_ENOMEM = 9  # 内存不足（示意值）
    AIRY_KERN_EBPF = -200  # BPF 程序加载失败
    AIRY_KERN_ESCHED = -201  # 调度器错误
    AIRY_IPC_ETIMEDOUT = -802  # IPC 通信超时
    # ...
```

---

## 4. 错误码与本地化消息分离设计

> **规划态声明**：§4.2/§4.3 示例中的 `AIRY_EGENERIC` / `AIRY_KERN_EBPF` / `AIRY_SVC_EGATEWAY` / `AIRY_LLM_EPROVIDER` / `AIRY_MEM_EROVOL` / `AIRY_SEC_ECAP` / `AIRY_IPC_ETIMEDOUT` / `AIRY_I18N_ELOCALE` 等符号在 [SC] `error.h` 中**不存在**（仅 §2 摘录的 POSIX 段 1-40 与子系统段 41-240 为实际定义），以下代码为**设计示意**；`AIRY_I18N_ELOCALE` 等 i18n 错误码**待 [SC] 注册**（保留段 241-300）。

### 4.1 分离架构

错误码（机器契约）与本地化消息（人类呈现）严格分离：

```
┌─────────────────────────────────────────────────┐
│  错误码（SSoT）                                  │
│  include/uapi/linux/airymax/error.h [SC]                   │
│  AIRY_IPC_ETIMEDOUT = -802                   │
└────────────────────┬────────────────────────────┘
                     │
                     ↓
         ┌───────────────────────┐
         │  错误码 → msgid 映射   │
         │  airy_err_to_msgid│
         └───────────┬───────────┘
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
   en_US.po      zh_CN.po     ja_JP.po
   msgid         msgid         msgid
   msgstr(英)    msgstr(中)    msgstr(日)
        ↓            ↓            ↓
   en_US.mo     zh_CN.mo     ja_JP.mo
        ↓            ↓            ↓
   运行时按 locale 选择 .mo 文件
```

### 4.2 错误码 → msgid 映射

```c
/* services/common/error_msgid.c [IND] */
#include <airymax/error.h>

const char *airy_err_to_msgid(airy_err_t err)
{
    switch (err) {
    case AIRY_EOK:               return "AIRY_EOK";
    case AIRY_EGENERIC:         return "AIRY_EGENERIC";
    case AIRY_EINVAL:           return "AIRY_EINVAL";
    case AIRY_ENOMEM:           return "AIRY_ENOMEM";
    case AIRY_ENOSYS:           return "AIRY_ENOSYS";
    case AIRY_KERN_EBPF:        return "AIRY_KERN_EBPF";
    case AIRY_KERN_ESCHED:      return "AIRY_KERN_ESCHED";
    case AIRY_KERN_EIPC:        return "AIRY_KERN_EIPC";
    case AIRY_SVC_EGATEWAY:     return "AIRY_SVC_EGATEWAY";
    case AIRY_SVC_ELLM:         return "AIRY_SVC_ELLM";
    case AIRY_LLM_EPROVIDER:    return "AIRY_LLM_EPROVIDER";
    case AIRY_LLM_ECONTEXT:     return "AIRY_LLM_ECONTEXT";
    case AIRY_MEM_EROVOL:       return "AIRY_MEM_EROVOL";
    case AIRY_SEC_ECAP:         return "AIRY_SEC_ECAP";
    case AIRY_IPC_ETIMEDOUT:    return "AIRY_IPC_ETIMEDOUT";
    case AIRY_I18N_ELOCALE:     return "AIRY_I18N_ELOCALE";
    /* ... 完整映射表，由 error_gen 自动生成 */
    default:                       return "AIRY_EGENERIC";
    }
}
```

### 4.3 错误码 → 本地化消息

```c
/* services/common/error_i18n.c [IND] */
#include <libintl.h>
#include <airymax/error.h>

const char *airy_strerror_i18n(airy_err_t err)
{
    const char *msgid;

    msgid = airy_err_to_msgid(err);
    if (!msgid)
        return dgettext("airymaxos", "AIRY_EGENERIC");

    /* gettext 查找当前 locale 的翻译 */
    return dgettext("airymaxos", msgid);
}

/* 带参数的错误消息格式化 */
int airy_strerror_r(airy_err_t err, char *buf, size_t len)
{
    const char *msg;
    int written;

    if (!buf || len == 0)
        return -EINVAL;

    msg = airy_strerror_i18n(err);
    /* strscpy 安全拷贝，保证 NUL 终止 */
    written = strscpy(buf, msg, len);
    if (written < 0)
        return -E2BIG;

    return 0;
}
```

---

## 5. 多语言消息映射表

### 5.1 错误码多语言映射表

以下为关键错误码的多语言消息映射表（节选）：

| 错误码 | en_US | zh_CN | ja_JP |
|--------|-------|-------|-------|
| AIRY_EOK | Success | 成功 | 成功 |
| AIRY_EGENERIC | Generic error | 通用错误 | 汎用エラー |
| AIRY_EINVAL | Invalid parameter | 参数非法 | パラメータが無効です |
| AIRY_ENOMEM | Out of memory | 内存不足 | メモリ不足 |
| AIRY_KERN_EBPF | BPF program load failed | BPF 程序加载失败 | BPF プログラムのロードに失敗 |
| AIRY_KERN_ESCHED | Scheduler error | 调度器错误 | スケジューラエラー |
| AIRY_KERN_EIPC | IPC failure | IPC 通信失败 | IPC 通信失敗 |
| AIRY_SVC_EGATEWAY | Gateway service error | 网关服务错误 | ゲートウェイサービスエラー |
| AIRY_SVC_ELLM | LLM service error | LLM 服务错误 | LLM サービスエラー |
| AIRY_LLM_EPROVIDER | LLM provider error | LLM 提供商错误 | LLM プロバイダエラー |
| AIRY_LLM_ECONTEXT | Context too long | 上下文过长 | コンテキストが長すぎます |
| AIRY_MEM_EROVOL | MemoryRovol error | 记忆卷载错误 | MemoryRovol エラー |
| AIRY_SEC_ECAP | Capability denied | capability 权限拒绝 | capability 権限拒否 |
| AIRY_IPC_ETIMEDOUT | IPC timeout | IPC 通信超时 | IPC 通信タイムアウト |
| AIRY_I18N_ELOCALE | Locale unavailable | locale 不可用 | locale が利用不可 |

### 5.2 zh_CN .po 文件片段

```po
# agentrt-linux 错误消息中文翻译
# Copyright (c) 2025-2026 SPHARX Ltd.
#
msgid ""
msgstr ""
"Project-Id-Version: agentrt-linux 1.0.1\n"
"Language: zh_CN\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"

# 成功
msgid "AIRY_EOK"
msgstr "成功"

# 通用错误
msgid "AIRY_EGENERIC"
msgstr "通用错误"

msgid "AIRY_EINVAL"
msgstr "参数非法"

msgid "AIRY_ENOMEM"
msgstr "内存不足"

# 内核错误
msgid "AIRY_KERN_EBPF"
msgstr "BPF 程序加载失败"

msgid "AIRY_KERN_ESCHED"
msgstr "调度器错误"

msgid "AIRY_KERN_EIPC"
msgstr "IPC 通信失败"

# 服务错误
msgid "AIRY_SVC_EGATEWAY"
msgstr "网关服务错误"

msgid "AIRY_SVC_ELLM"
msgstr "LLM 服务错误"

# LLM 推理错误
msgid "AIRY_LLM_EPROVIDER"
msgstr "LLM 提供商错误"

msgid "AIRY_LLM_ECONTEXT"
msgstr "上下文过长"

# 记忆错误
msgid "AIRY_MEM_EROVOL"
msgstr "记忆卷载错误"

# 安全错误
msgid "AIRY_SEC_ECAP"
msgstr "capability 权限拒绝"

# 协议错误
msgid "AIRY_IPC_ETIMEDOUT"
msgstr "IPC 通信超时"

# i18n 错误
msgid "AIRY_I18N_ELOCALE"
msgstr "locale 不可用"
```

### 5.3 en_US .po 文件片段

```po
# agentrt-linux error messages English translation
# Copyright (c) 2025-2026 SPHARX Ltd.
#
msgid ""
msgstr ""
"Project-Id-Version: agentrt-linux 1.0.1\n"
"Language: en_US\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"

msgid "AIRY_EOK"
msgstr "Success"

msgid "AIRY_EGENERIC"
msgstr "Generic error"

msgid "AIRY_EINVAL"
msgstr "Invalid parameter"

msgid "AIRY_ENOMEM"
msgstr "Out of memory"

msgid "AIRY_KERN_EBPF"
msgstr "BPF program load failed"

msgid "AIRY_KERN_ESCHED"
msgstr "Scheduler error"

msgid "AIRY_IPC_ETIMEDOUT"
msgstr "IPC timeout"

msgid "AIRY_I18N_ELOCALE"
msgstr "Locale unavailable"
```

---

## 6. 跨端错误码同源

### 6.1 IRON-9 v3 [SC] 共享

错误码定义位于 IRON-9 v3 [SC] 共享契约层，agentrt-linux 内核态与 agentrt 用户态运行时完全共享同一份 `include/uapi/linux/airymax/error.h`：

```
agentrt 仓库:
  include/uapi/linux/airymax/error.h  ←──┐
                               │ 同一份代码
agentrt-linux 仓库:           │ （symlink 或 git submodule）
  include/uapi/linux/airymax/error.h  ←──┘
```

### 6.2 [SC] 层变更流程

错误码属于 [SC] 共享契约层，变更须通过双向 CI 校验：

```bash
# 1. 修改 include/uapi/linux/airymax/error.h（任一仓库）
vim include/uapi/linux/airymax/error.h
# 新增：#define AIRY_NEW_ERR  (-999)

# 2. agentrt 仓库 CI 校验
cd /path/to/agentrt && make ci-check-error-codes

# 3. agentrt-linux 仓库 CI 校验
cd /path/to/agentrt-linux && make ci-check-error-codes

# 4. 双向同步校验（必须两端通过）
make ci-cross-validate
```

### 6.3 错误码可追溯

每个错误码的来源、修改历史、影响范围全部可追溯：

```bash
# 查询错误码定义来源
airymaxos-error-trace AIRY_IPC_ETIMEDOUT

# 输出：
# 错误码：AIRY_IPC_ETIMEDOUT
# 值：-802
# 定义文件：include/uapi/linux/airymax/error.h:42
# 引入版本：1.0.1
# 引入提交：a1b2c3d4 ("ipc: 新增超时错误码")
# 影响范围：
#   - kernel/ipc/ (5 处)
#   - services/gateway_d/ (3 处)
#   - agentrt/sdk/python/agentrt/errors.py (1 处)
#   - po/airy_errors.pot (1 处 msgid)
```

---

## 7. 错误消息测试

### 7.1 测试套件

错误消息 i18n 测试位于 `tests-linux/i18n/`：

| 测试名 | 描述 | 通过标准 |
|--------|------|----------|
| `test_error_code_ssoT` | 错误码 SSoT | 所有错误码唯一定义于 error.h |
| `test_error_msgid_mapping` | 错误码 → msgid | 所有错误码都有对应 msgid |
| `test_error_translation_complete` | 翻译完整性 | 所有 locale 的 .po 无 untranslated |
| `test_error_strscpy_safety` | strscpy 安全 | 错误消息拷贝用 strscpy |
| `test_error_cross_validate` | 跨端校验 | agentrt 与 agentrt-linux 错误码一致 |

### 7.2 SSoT 校验测试

```c
/* tests-linux/i18n/test_error_ssot.c [IND] */
#include <airymax/error.h>
#include <assert.h>

static void test_no_duplicate_definition(void)
{
    /* 每个错误码值必须唯一 */
    assert(AIRY_EOK == 0);
    assert(AIRY_EINVAL == -1);
    assert(AIRY_ENOMEM == -2);
    assert(AIRY_KERN_EBPF == -200);
    assert(AIRY_IPC_ETIMEDOUT == -802);
    /* ... 全量校验 */
}

static void test_msgid_mapping(void)
{
    const char *msgid;

    msgid = airy_err_to_msgid(AIRY_EOK);
    assert(strcmp(msgid, "AIRY_EOK") == 0);

    msgid = airy_err_to_msgid(AIRY_IPC_ETIMEDOUT);
    assert(strcmp(msgid, "AIRY_IPC_ETIMEDOUT") == 0);

    /* 未知错误码降级到 AIRY_EGENERIC */
    msgid = airy_err_to_msgid(-9999);
    assert(strcmp(msgid, "AIRY_EGENERIC") == 0);
}

int main(void)
{
    test_no_duplicate_definition();
    test_msgid_mapping();
    printf("OK: 错误码 SSoT 测试通过\n");
    return 0;
}
```

---

## 8. 错误处理与降级

### 8.1 .mo 文件缺失降级

当某 locale 的 .mo 文件缺失时，gettext 自动返回 msgid（即错误码名称），保证可读性：

```c
/* 即使 .mo 缺失，仍返回有意义的字符串 */
const char *msg = airy_strerror_i18n(AIRY_IPC_ETIMEDOUT);
/* 若 zh_CN.mo 缺失，msg = "AIRY_IPC_ETIMEDOUT"（msgid passthrough） */
```

### 8.2 错误码注册失败

```c
int airy_err_register_new(airy_err_t new_err, const char *msgid)
{
    int err;

    if (!msgid) {
        WARN(1, "错误码注册失败：msgid 为空");
        return -EINVAL;
    }

    err = error_registry_add(new_err, msgid);
    if (err) {
        WARN(1, "错误码 %d 注册失败：%d", new_err, err);
        return err;
    }

    return 0;
}
```

### 8.3 集中错误处理

错误码注册表初始化采用 `goto out_free_xxx` 集中错误处理：

```c
int airy_err_registry_init(struct error_registry_cfg *cfg)
{
    struct error_registry *reg;
    int err;

    reg = kzalloc(sizeof(*reg), GFP_KERNEL);
    if (!reg)
        return -ENOMEM;

    err = registry_init_hashtable(reg, cfg);
    if (err)
        goto out_free_reg;

    err = registry_load_from_header(reg, "include/uapi/linux/airymax/error.h");
    if (err)
        goto out_free_hashtable;

    err = registry_load_translations(reg, cfg->locale);
    if (err)
        goto out_free_header;

    pr_info("agentrt-linux: 错误码注册表初始化完成\n");
    return 0;

out_free_header:
    registry_release_header(reg);
out_free_hashtable:
    registry_release_hashtable(reg);
out_free_reg:
    kfree(reg);
    return err;
}
```

---

## 9. 五维原则映射

| 原则 | 在错误消息 i18n 中的体现 |
|------|------------------------|
| **E-6 错误可追溯** | 每个错误码可追溯定义来源与影响范围 |
| **K-2 接口契约化** | 错误码是跨端硬契约（[SC] 层） |
| **E-7 文档即代码** | .po 文件版本控制，error_gen 自动生成 |
| **E-8 可测试性** | SSoT 校验 + 翻译完整性测试 |
| **IRON-9 v3 [SC]** | 错误码定义两端完全共享 |

---

## 10. 相关文档

- `180-i18n/README.md`（国际化主索引）
- `180-i18n/01-locale-design.md`（区域设置设计）
- `180-i18n/05-doc-i18n.md`（文档国际化）
- `90-observability/README.md`（错误链路追踪）
- `50-engineering-standards/01-coding-standards.md`（错误处理规范）
- `140-application-development/README.md`（SDK 错误码绑定）

---

## 11. 参考材料

- Linux 6.6 errno 定义（`include/uapi/asm-generic/errno-base.h`）
- GNU gettext 错误消息实践
- JSON-RPC 2.0 错误对象规范
- agentrt 错误码定义（[SC] 共享契约层）
- 主流 Linux 发行版错误消息实践

---

## 12. 版本演进

| 版本 | 错误码体系 | SSoT | 多语言 | 跨端同源 | 备注 |
|------|-----------|------|--------|---------|------|
| 0.1.1 | 设计 | error.h | — | — | 仅文档 |
| 1.0.1 | 双体系 | SSoT | zh/en/ja | [SC] 共享 | 开发 |
| 1.1.0 | + 扩展段 | SSoT | + ko/de/fr | [SC] 共享 | 扩展语言 |
| 2.0.0 | + 动态注册 | SSoT | 全语言 | [SC] 共享 | 动态错误码 |

---

> **文档结束** | agentrt-linux（AirymaxOS）错误消息国际化设计 v1.0.1
