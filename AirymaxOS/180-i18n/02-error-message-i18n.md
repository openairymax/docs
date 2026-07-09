Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）错误消息国际化设计

> **文档定位**: agentrt-linux（AirymaxOS，极境智能体操作系统）错误消息国际化工程设计文档，覆盖错误码（agentrt_error_t）与本地化消息分离设计、错误码注册表 SSoT、多语言消息映射
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 1. 概述

### 1.1 设计目标

agentrt-linux（AirymaxOS）作为面向全球开发者和用户的智能体操作系统发行版，其错误消息体系是开发者体验与可追溯性的关键。本文档定义错误消息国际化工程设计，目标是：

1. **错误码与消息分离**：错误码（`agentrt_error_t`）作为机器可读契约，本地化消息作为人类可读呈现，二者严格分离
2. **错误码注册表 SSoT**：所有错误码唯一定义于 `include/airymax/error.h`（IRON-9 v2 [SC] 共享契约层），任何文档、代码不得另起定义
3. **多语言消息映射**：每个错误码映射到多语言消息表，支持 zh_CN、en_US、ja_JP 等 locale
4. **跨端错误码同源**：agentrt-linux 内核态与 agentrt 用户态运行时错误码完全一致，实现跨端错误可追溯

### 1.2 基线约束

本设计严格遵循 Linux 6.6 内核基线，并执行以下硬约束：

| 约束项 | 取值 | 说明 |
|--------|------|------|
| 错误码体系 | 双错误码体系 | C 负整数（首要）+ SDK 十六进制（次要） |
| 错误码 SSoT | `include/airymax/error.h` | 唯一定义源，[SC] 共享契约层 |
| 内核态消息 | 英文 | `printk` 永远英文，错误码通过 errno 暴露 |
| 用户态本地化 | gettext | 错误码 → msgid 映射，由 gettext 查找本地化消息 |
| 字符串安全 | `strscpy` | 错误消息拷贝必须用 `strscpy` |

### 1.3 术语规范

本文档严格遵循术语规范：agentrt（用户态运行时）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）。错误码定义共享于 [SC] 共享契约层（两端完全共享代码），错误消息呈现独立于两端实现。文中不使用 openEuler/Euler 字样，错误码体系对照一律表述为"主流 Linux 发行版"。

---

## 2. 双错误码体系

### 2.1 体系概览

agentrt-linux 采用双错误码体系，分别适用于不同场景：

| 体系 | 适用场景 | 格式 | 权威源 |
|------|----------|------|--------|
| **C 负整数体系**（首要） | C 内核和 Daemon 层 | `AGENTRT_OK=0`、`AGENTRT_EINVAL=-2` | `include/airymax/error.h` |
| **SDK 十六进制体系**（次要） | SDK 和外部接口 | `0x0000`-`0x7FFF` 分段 | `include/airymax/error.h` |

### 2.2 C 负整数体系分段

C 负整数体系按子系统分段，定义于 [SC] 共享契约层：

```c
/* include/airymax/error.h [SC] */
#ifndef _AIRYMAX_ERROR_H
#define _AIRYMAX_ERROR_H

#include <linux/types.h>

typedef int agentrt_error_t;

/* 成功 */
#define AGENTRT_OK               0

/* 通用错误（-1 ~ -99） */
#define AGENTRT_EGENERIC        (-1)
#define AGENTRT_EINVAL          (-2)
#define AGENTRT_ENOMEM          (-3)
#define AGENTRT_ENOSYS          (-4)
#define AGENTRT_ENOENT          (-5)
#define AGENTRT_EACCES          (-6)
#define AGENTRT_EBUSY           (-7)
#define AGENTRT_ETIMEDOUT       (-8)
#define AGENTRT_EIO             (-9)

/* 系统级错误（-100 ~ -199） */
#define AGENTRT_SYS_EBOOT       (-100)
#define AGENTRT_SYS_ESHUTDOWN   (-101)
#define AGENTRT_SYS_ESERVICE    (-102)
#define AGENTRT_SYS_ECONFIG     (-103)

/* 内核错误（-200 ~ -299） */
#define AGENTRT_KERN_EBPF       (-200)
#define AGENTRT_KERN_ESCHED     (-201)
#define AGENTRT_KERN_EIPC       (-202)
#define AGENTRT_KERN_EMEM       (-203)

/* 服务错误（-300 ~ -399） */
#define AGENTRT_SVC_EGATEWAY    (-300)
#define AGENTRT_SVC_ELLM        (-301)
#define AGENTRT_SVC_ETOOL       (-302)
#define AGENTRT_SVC_ESCHED      (-303)

/* LLM 推理错误（-400 ~ -499） */
#define AGENTRT_LLM_EPROVIDER   (-400)
#define AGENTRT_LLM_ECONTEXT    (-401)
#define AGENTRT_LLM_ETOKEN      (-402)
#define AGENTRT_LLM_ERATE       (-403)

/* 执行错误（-500 ~ -599） */
#define AGENTRT_EXEC_ETASK      (-500)
#define AGENTRT_EXEC_ECOMPENSATE (-501)

/* 记忆错误（-600 ~ -699） */
#define AGENTRT_MEM_EROVOL      (-600)
#define AGENTRT_MEM_ESWAP       (-601)
#define AGENTRT_MEM_EFORGET     (-602)

/* 安全错误（-700 ~ -799） */
#define AGENTRT_SEC_ECAP        (-700)
#define AGENTRT_SEC_ESANDBOX    (-701)
#define AGENTRT_SEC_EAUDIT      (-702)

/* 协议错误（-800 ~ -899） */
#define AGENTRT_PROTO_EMCP      (-800)
#define AGENTRT_PROTO_EA2A      (-801)

/* i18n 错误（-900 ~ -999） */
#define AGENTRT_I18N_ELOCALE    (-900)
#define AGENTRT_I18N_EENCODING  (-901)

#endif /* _AIRYMAX_ERROR_H */
```

### 2.3 SDK 十六进制体系分段

SDK 十六进制体系用于 SDK 和外部接口（Python/Go/Rust/Java SDK）：

```c
/* include/airymax/error.h [SC] （续） */

/* SDK 错误码（0x0000 - 0x7FFF） */
#define AGENTRT_ERR_OK             0x0000
#define AGENTRT_ERR_GENERIC        0x0001
#define AGENTRT_ERR_INVALID_PARAM  0x0002

/* 通用段 0x0000 - 0x0FFF */
#define AGENTRT_ERR_SYS_BASE       0x0100

/* 内核段 0x1000 - 0x1FFF */
#define AGENTRT_ERR_KERN_BASE      0x1000

/* 服务段 0x2000 - 0x2FFF */
#define AGENTRT_ERR_SVC_BASE       0x2000

/* LLM 段 0x3000 - 0x3FFF */
#define AGENTRT_ERR_LLM_BASE       0x3000

/* 安全段 0x4000 - 0x4FFF */
#define AGENTRT_ERR_SEC_BASE       0x4000
```

### 2.4 双体系映射

双错误码体系之间的映射函数定义于 [SC] 共享契约层：

```c
/* include/airymax/error.h [SC] （续） */

static inline u16 agentrt_err_c_to_sdk(agentrt_error_t err)
{
    if (err >= 0)
        return AGENTRT_ERR_OK;

    /* C 负整数 → SDK 十六进制 */
    switch (err) {
    case AGENTRT_EGENERIC:        return AGENTRT_ERR_GENERIC;
    case AGENTRT_EINVAL:          return AGENTRT_ERR_INVALID_PARAM;
    case AGENTRT_KERN_EBPF:       return AGENTRT_ERR_KERN_BASE + 0x00;
    case AGENTRT_KERN_ESCHED:     return AGENTRT_ERR_KERN_BASE + 0x01;
    case AGENTRT_KERN_EIPC:       return AGENTRT_ERR_KERN_BASE + 0x02;
    case AGENTRT_SVC_EGATEWAY:    return AGENTRT_ERR_SVC_BASE + 0x00;
    case AGENTRT_SVC_ELLM:        return AGENTRT_ERR_SVC_BASE + 0x01;
    /* ... 完整映射表 */
    default:                      return AGENTRT_ERR_GENERIC;
    }
}

static inline agentrt_error_t agentrt_err_sdk_to_c(u16 sdk_err)
{
    /* SDK 十六进制 → C 负整数（反向映射） */
    switch (sdk_err) {
    case AGENTRT_ERR_OK:           return AGENTRT_OK;
    case AGENTRT_ERR_GENERIC:      return AGENTRT_EGENERIC;
    case AGENTRT_ERR_INVALID_PARAM: return AGENTRT_EINVAL;
    /* ... 完整映射表 */
    default:                       return AGENTRT_EGENERIC;
    }
}
```

### 2.5 体系使用规范

| 场景 | 体系 | 示例 |
|------|------|------|
| C 内核代码 | C 负整数 | `return -ENOENT;` |
| Daemon C 代码 | C 负整数 | `return AGENTRT_IPC_ETIMEDOUT;` |
| Python SDK | SDK 十六进制 | `raise AgentrtError(0x1002)` |
| Go SDK | SDK 十六进制 | `return AgentrtErrKernIPC` |
| Rust SDK | SDK 十六进制 | `AgentrtErr::KernIpc` |
| Java SDK | SDK 十六进制 | `throw new AgentrtException(0x1002)` |
| JSON-RPC 错误响应 | SDK 十六进制 | `{"code": 4098, "message": "..."}` |

**禁止**：C 内核代码中使用十六进制错误码；SDK 中使用负整数错误码。

---

## 3. 错误码注册表 SSoT

### 3.1 SSoT 原则

Single Source of Truth（SSoT）原则要求：所有错误码唯一定义于 `include/airymax/error.h`。任何文档、代码、SDK 不得另起定义，必须引用此权威源。

```
权威源（SSoT）：
  include/airymax/error.h  [SC] 共享契约层
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

agentrt-linux 提供 `airymaxos-error-gen` 工具，从 `include/airymax/error.h` 自动生成各语言的错误码绑定：

```python
#!/usr/bin/env python3
# airymaxos-system/error_gen.py [IND]
"""从 include/airymax/error.h 生成各语言错误码绑定"""

import re
import sys
from pathlib import Path

ERROR_HEADER = "include/airymax/error.h"

PATTERN = re.compile(
    r'#define\s+(AGENTRT_[A-Z_]+)\s+(\(?-?\d+\)?)\s*/\*\s*(.*?)\s*\*/'
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
        f.write("# Source: include/airymax/error.h [SC]\n\n")
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
    gen_python(errors, "airymaxos_sdk/python/agentrt/errors.py")
    gen_po(errors, "po/airymaxos_errors.pot")
    print(f"生成 {len(errors)} 个错误码绑定")
```

### 3.3 生成产物示例

Python SDK 自动生成的错误码：

```python
# airymaxos_sdk/python/agentrt/errors.py
# AUTO-GENERATED. DO NOT EDIT.
# Source: include/airymax/error.h [SC]

from enum import IntEnum

class AgentrtError(IntEnum):
    AGENTRT_OK = 0  # 成功
    AGENTRT_EGENERIC = -1  # 通用错误
    AGENTRT_EINVAL = -2  # 参数非法
    AGENTRT_ENOMEM = -3  # 内存不足
    AGENTRT_KERN_EBPF = -200  # BPF 程序加载失败
    AGENTRT_KERN_ESCHED = -201  # 调度器错误
    AGENTRT_IPC_ETIMEDOUT = -802  # IPC 通信超时
    # ...
```

---

## 4. 错误码与本地化消息分离设计

### 4.1 分离架构

错误码（机器契约）与本地化消息（人类呈现）严格分离：

```
┌─────────────────────────────────────────────────┐
│  错误码（SSoT）                                  │
│  include/airymax/error.h [SC]                   │
│  AGENTRT_IPC_ETIMEDOUT = -802                   │
└────────────────────┬────────────────────────────┘
                     │
                     ↓
         ┌───────────────────────┐
         │  错误码 → msgid 映射   │
         │  agentrt_error_to_msgid│
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
/* airymaxos-services/common/error_msgid.c [IND] */
#include <airymax/error.h>

const char *agentrt_error_to_msgid(agentrt_error_t err)
{
    switch (err) {
    case AGENTRT_OK:               return "AGENTRT_OK";
    case AGENTRT_EGENERIC:         return "AGENTRT_EGENERIC";
    case AGENTRT_EINVAL:           return "AGENTRT_EINVAL";
    case AGENTRT_ENOMEM:           return "AGENTRT_ENOMEM";
    case AGENTRT_ENOSYS:           return "AGENTRT_ENOSYS";
    case AGENTRT_KERN_EBPF:        return "AGENTRT_KERN_EBPF";
    case AGENTRT_KERN_ESCHED:      return "AGENTRT_KERN_ESCHED";
    case AGENTRT_KERN_EIPC:        return "AGENTRT_KERN_EIPC";
    case AGENTRT_SVC_EGATEWAY:     return "AGENTRT_SVC_EGATEWAY";
    case AGENTRT_SVC_ELLM:         return "AGENTRT_SVC_ELLM";
    case AGENTRT_LLM_EPROVIDER:    return "AGENTRT_LLM_EPROVIDER";
    case AGENTRT_LLM_ECONTEXT:     return "AGENTRT_LLM_ECONTEXT";
    case AGENTRT_MEM_EROVOL:       return "AGENTRT_MEM_EROVOL";
    case AGENTRT_SEC_ECAP:         return "AGENTRT_SEC_ECAP";
    case AGENTRT_IPC_ETIMEDOUT:    return "AGENTRT_IPC_ETIMEDOUT";
    case AGENTRT_I18N_ELOCALE:     return "AGENTRT_I18N_ELOCALE";
    /* ... 完整映射表，由 error_gen 自动生成 */
    default:                       return "AGENTRT_EGENERIC";
    }
}
```

### 4.3 错误码 → 本地化消息

```c
/* airymaxos-services/common/error_i18n.c [IND] */
#include <libintl.h>
#include <airymax/error.h>

const char *agentrt_strerror_i18n(agentrt_error_t err)
{
    const char *msgid;

    msgid = agentrt_error_to_msgid(err);
    if (!msgid)
        return dgettext("airymaxos", "AGENTRT_EGENERIC");

    /* gettext 查找当前 locale 的翻译 */
    return dgettext("airymaxos", msgid);
}

/* 带参数的错误消息格式化 */
int agentrt_strerror_r(agentrt_error_t err, char *buf, size_t len)
{
    const char *msg;
    int written;

    if (!buf || len == 0)
        return -EINVAL;

    msg = agentrt_strerror_i18n(err);
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
| AGENTRT_OK | Success | 成功 | 成功 |
| AGENTRT_EGENERIC | Generic error | 通用错误 | 汎用エラー |
| AGENTRT_EINVAL | Invalid parameter | 参数非法 | パラメータが無効です |
| AGENTRT_ENOMEM | Out of memory | 内存不足 | メモリ不足 |
| AGENTRT_KERN_EBPF | BPF program load failed | BPF 程序加载失败 | BPF プログラムのロードに失敗 |
| AGENTRT_KERN_ESCHED | Scheduler error | 调度器错误 | スケジューラエラー |
| AGENTRT_KERN_EIPC | IPC failure | IPC 通信失败 | IPC 通信失敗 |
| AGENTRT_SVC_EGATEWAY | Gateway service error | 网关服务错误 | ゲートウェイサービスエラー |
| AGENTRT_SVC_ELLM | LLM service error | LLM 服务错误 | LLM サービスエラー |
| AGENTRT_LLM_EPROVIDER | LLM provider error | LLM 提供商错误 | LLM プロバイダエラー |
| AGENTRT_LLM_ECONTEXT | Context too long | 上下文过长 | コンテキストが長すぎます |
| AGENTRT_MEM_EROVOL | MemoryRovol error | 记忆卷载错误 | MemoryRovol エラー |
| AGENTRT_SEC_ECAP | Capability denied | capability 权限拒绝 | capability 権限拒否 |
| AGENTRT_IPC_ETIMEDOUT | IPC timeout | IPC 通信超时 | IPC 通信タイムアウト |
| AGENTRT_I18N_ELOCALE | Locale unavailable | locale 不可用 | locale が利用不可 |

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
msgid "AGENTRT_OK"
msgstr "成功"

# 通用错误
msgid "AGENTRT_EGENERIC"
msgstr "通用错误"

msgid "AGENTRT_EINVAL"
msgstr "参数非法"

msgid "AGENTRT_ENOMEM"
msgstr "内存不足"

# 内核错误
msgid "AGENTRT_KERN_EBPF"
msgstr "BPF 程序加载失败"

msgid "AGENTRT_KERN_ESCHED"
msgstr "调度器错误"

msgid "AGENTRT_KERN_EIPC"
msgstr "IPC 通信失败"

# 服务错误
msgid "AGENTRT_SVC_EGATEWAY"
msgstr "网关服务错误"

msgid "AGENTRT_SVC_ELLM"
msgstr "LLM 服务错误"

# LLM 推理错误
msgid "AGENTRT_LLM_EPROVIDER"
msgstr "LLM 提供商错误"

msgid "AGENTRT_LLM_ECONTEXT"
msgstr "上下文过长"

# 记忆错误
msgid "AGENTRT_MEM_EROVOL"
msgstr "记忆卷载错误"

# 安全错误
msgid "AGENTRT_SEC_ECAP"
msgstr "capability 权限拒绝"

# 协议错误
msgid "AGENTRT_IPC_ETIMEDOUT"
msgstr "IPC 通信超时"

# i18n 错误
msgid "AGENTRT_I18N_ELOCALE"
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

msgid "AGENTRT_OK"
msgstr "Success"

msgid "AGENTRT_EGENERIC"
msgstr "Generic error"

msgid "AGENTRT_EINVAL"
msgstr "Invalid parameter"

msgid "AGENTRT_ENOMEM"
msgstr "Out of memory"

msgid "AGENTRT_KERN_EBPF"
msgstr "BPF program load failed"

msgid "AGENTRT_KERN_ESCHED"
msgstr "Scheduler error"

msgid "AGENTRT_IPC_ETIMEDOUT"
msgstr "IPC timeout"

msgid "AGENTRT_I18N_ELOCALE"
msgstr "Locale unavailable"
```

---

## 6. 跨端错误码同源

### 6.1 IRON-9 v2 [SC] 共享

错误码定义位于 IRON-9 v2 [SC] 共享契约层，agentrt-linux 内核态与 agentrt 用户态运行时完全共享同一份 `include/airymax/error.h`：

```
agentrt 仓库:
  include/airymax/error.h  ←──┐
                               │ 同一份代码
agentrt-linux 仓库:           │ （symlink 或 git submodule）
  include/airymax/error.h  ←──┘
```

### 6.2 [SC] 层变更流程

错误码属于 [SC] 共享契约层，变更须通过双向 CI 校验：

```bash
# 1. 修改 include/airymax/error.h（任一仓库）
vim include/airymax/error.h
# 新增：#define AGENTRT_NEW_ERR  (-999)

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
airymaxos-error-trace AGENTRT_IPC_ETIMEDOUT

# 输出：
# 错误码：AGENTRT_IPC_ETIMEDOUT
# 值：-802
# 定义文件：include/airymax/error.h:42
# 引入版本：1.0.1
# 引入提交：a1b2c3d4 ("ipc: 新增超时错误码")
# 影响范围：
#   - airymaxos-kernel/ipc/ (5 处)
#   - airymaxos-services/gateway_d/ (3 处)
#   - airymaxos_sdk/python/agentrt/errors.py (1 处)
#   - po/airymaxos_errors.pot (1 处 msgid)
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
    assert(AGENTRT_OK == 0);
    assert(AGENTRT_EGENERIC == -1);
    assert(AGENTRT_EINVAL == -2);
    assert(AGENTRT_KERN_EBPF == -200);
    assert(AGENTRT_IPC_ETIMEDOUT == -802);
    /* ... 全量校验 */
}

static void test_msgid_mapping(void)
{
    const char *msgid;

    msgid = agentrt_error_to_msgid(AGENTRT_OK);
    assert(strcmp(msgid, "AGENTRT_OK") == 0);

    msgid = agentrt_error_to_msgid(AGENTRT_IPC_ETIMEDOUT);
    assert(strcmp(msgid, "AGENTRT_IPC_ETIMEDOUT") == 0);

    /* 未知错误码降级到 AGENTRT_EGENERIC */
    msgid = agentrt_error_to_msgid(-9999);
    assert(strcmp(msgid, "AGENTRT_EGENERIC") == 0);
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
const char *msg = agentrt_strerror_i18n(AGENTRT_IPC_ETIMEDOUT);
/* 若 zh_CN.mo 缺失，msg = "AGENTRT_IPC_ETIMEDOUT"（msgid passthrough） */
```

### 8.2 错误码注册失败

```c
int agentrt_error_register_new(agentrt_error_t new_err, const char *msgid)
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
int agentrt_error_registry_init(struct error_registry_cfg *cfg)
{
    struct error_registry *reg;
    int err;

    reg = kzalloc(sizeof(*reg), GFP_KERNEL);
    if (!reg)
        return -ENOMEM;

    err = registry_init_hashtable(reg, cfg);
    if (err)
        goto out_free_reg;

    err = registry_load_from_header(reg, "include/airymax/error.h");
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
| **IRON-9 v2 [SC]** | 错误码定义两端完全共享 |

---

## 10. 相关文档

- `180-i18n/README.md`（国际化主索引）
- `180-i18n/01-locale-design.md`（区域设置设计）
- `180-i18n/03-doc-i18n.md`（文档国际化）
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
