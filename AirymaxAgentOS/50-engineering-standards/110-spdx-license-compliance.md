Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）SPDX 许可证标识强制规范

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）的 SPDX 许可证标识强制规范。所有 `.c`/`.h`/`.md`/`.rst`/`.py`/`.sh`/`Makefile`/`Kconfig` 文件必须含 `SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` 标识，含检查脚本与违规/合规示例。
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 目的与范围

本文件定义 agentrt-liunx 仓库中所有源代码、文档、脚本文件的 SPDX 许可证标识强制要求。所有文件**必须**在首行（或首行注释内）含 `SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` 标识。CI 通过扫描脚本强制校验，违规即拒绝合并。

### 0.2 OLK-6.6 源码路径

- `scripts/checkpatch.pl:3772-3807` —— `SPDX_LICENSE_TAG` 规则检查
- `scripts/checkpatch.pl:3773-3777` —— SPDX 注释样式检查（`.[chsS]` 文件需用对应注释符）
- `scripts/checkpatch.pl:3780-3783` —— 缺失或格式错误的 SPDX 标识检查
- `scripts/checkpatch.pl:3784-3789` —— SPDX 许可证合法性校验（在 LICENSES/ 中查询）
- `LICENSES/`（OLK-6.6 顶层目录）—— 许可证文本存放目录

### 0.3 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-liunx（OS 发行版）= 微内核（micro-kernel）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

---

## 1. SPDX 标识强制规则

### 1.1 规则 OS-STD-SPDX-001：所有文件必须含 SPDX 标识

agentrt-liunx 仓库中以下文件类型**必须**在文件首行（或首行注释内）含 SPDX 标识：

| 文件扩展名 | SPDX 标识位置 | 注释样式 | OLK-6.6 checkpatch 行号 |
|-----------|-------------|---------|------------------------|
| `.c` | 首行 | `/* */` 块注释 | `checkpatch.pl:3773` |
| `.h` | 首行 | `/* */` 块注释 | `checkpatch.pl:3773` |
| `.S`（汇编） | 首行 | `/* */` 块注释 | `checkpatch.pl:3773` |
| `.s` | 首行 | `/* */` 块注释 | `checkpatch.pl:3773` |
| `.md` / `.rst` | 首行 | 无注释符，直接文本 | — |
| `.py` | 首行 | `#` 行注释 | — |
| `.sh` | 首行 | `#` 行注释 | — |
| `Makefile` / `Kconfig` | 首行 | `#` 行注释 | — |
| `.yaml` / `.yml` | 首行 | `#` 行注释 | — |
| `.rs`（Rust） | 首行 | `//` 行注释 | — |

### 1.2 规则 OS-STD-SPDX-002：统一许可证字符串

agentrt-liunx 仓库**统一**使用以下 SPDX 许可证字符串：

```
SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
```

**许可证选择理由**：

- **AGPL-3.0-or-later**：保障网络服务场景下的源代码披露义务，与 agentrt-liunx 作为发行版的开放性一致
- **Apache-2.0**：提供更宽松的兼容路径，便于与 Apache 生态（如 Rust crates）集成
- **OR**：双许可，使用者可选择任一许可

### 1.3 规则 OS-STD-SPDX-003：注释样式正确性

SPDX 标识**必须**使用与文件类型匹配的注释样式：

**OLK-6.6 源码路径**: `scripts/checkpatch.pl:3772-3778`

| 文件类型 | 正确注释样式 | 错误注释样式 |
|---------|------------|------------|
| `.c`/`.h`/`.S` | `/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */` | `// SPDX-...`（禁止） |
| `.py` | `# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | `"""SPDX-..."""`（禁止） |
| `.sh`/`Makefile` | `# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | `// SPDX-...`（禁止） |
| `.md`/`.rst` | `SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0`（无注释符） | `<!-- SPDX-... -->`（禁止） |

---

## 2. 合规示例

### 2.1 C 文件合规示例

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
/*
 * agentrt_ipc.c - agentrt-liunx IPC channel implementation
 *
 * Implements the kernel-side IPC channel for agentrt-liunx.
 * The protocol is defined in include/airymax/ipc.h ([SC] contract).
 */

#include <linux/module.h>
#include <airymax/ipc.h>
```

### 2.2 头文件合规示例

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
#ifndef _AIRYMAX_IPC_H
#define _AIRYMAX_IPC_H

#include <linux/types.h>

struct agentrt_ipc_msg_hdr {
	__u32	magic;		/* 0x41524531 'ARE1' */
	/* ... */
};

#endif /* _AIRYMAX_IPC_H */
```

### 2.3 Python 脚本合规示例

```python
#!/usr/bin/env python3
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
"""agentrt-liunx CI helper: validate SPDX tags across the repo."""

import sys
import re
```

### 2.4 Shell 脚本合规示例

```bash
#!/bin/bash
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
#
# agentrt-liunx build script: trigger kernel + userspace build.

set -euo pipefail
```

### 2.5 Makefile 合规示例

```makefile
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
#
# agentrt-liunx top-level Makefile.

obj-y += kernel/airymax/
obj-y += drivers/airymax/
```

### 2.6 Markdown 文档合规示例

```markdown
Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）文档标题

> **文档定位**: ...
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

正文内容...
```

### 2.7 YAML 配置合规示例

```yaml
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
#
# CI configuration for agentrt-liunx.

stages:
  - lint
  - build
```

### 2.8 Rust 文件合规示例

```rust
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
//! agentrt-liunx Rust kernel module: Airymax task accounting.

use kernel::prelude::*;
```

---

## 3. 违规示例

### 3.1 缺失 SPDX 标识

```c
/*
 * agentrt_ipc.c - agentrt-liunx IPC channel implementation
 * WRONG: 缺 SPDX-License-Identifier 行
 */

#include <linux/module.h>
```

### 3.2 错误注释样式

```c
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
/* WRONG: .c 文件应用 /* */ 块注释，非 // */

#include <linux/module.h>
```

```python
"""SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0"""
# WRONG: .py 文件应用 # 行注释
```

### 3.3 错误许可证字符串

```c
/* SPDX-License-Identifier: GPL-2.0 */
/* WRONG: agentrt-liunx 统一用 AGPL-3.0-or-later OR Apache-2.0 */

#include <linux/module.h>
```

```c
/* SPDX-License-Identifier: AGPL-3.0 */
/* WRONG: 应用 AGPL-3.0-or-later（含 -or-later 后缀） */
```

### 3.4 SPDX 标识不在首行

```c
/*
 * agentrt_ipc.c - agentrt-liunx IPC channel implementation
 */
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
/* WRONG: SPDX 必须在首行 */

#include <linux/module.h>
```

### 3.5 文档文件用注释符包裹

```markdown
<!-- SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 -->
# 文档标题
<!-- WRONG: .md 文件应直接写 SPDX，不用 HTML 注释 -->
```

---

## 4. CI 检查脚本

### 4.1 spdx-check.sh 脚本

agentrt-liunx CI 使用以下脚本扫描所有文件，校验 SPDX 标识存在性与正确性：

```bash
#!/bin/bash
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
#
# spdx-check.sh: Validate SPDX-License-Identifier tags in agentrt-liunx repo.
# Exits non-zero if any file is missing or has malformed SPDX tag.

set -euo pipefail

REPO_ROOT="${1:-$(git rev-parse --show-toplevel)}"
EXPECTED="SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0"
EXIT_CODE=0

# 文件类型 → 期望的 SPDX 行正则
declare -A SPDX_PATTERNS=(
	["c"]='^/\* SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0 \*/$'
	["h"]='^/\* SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0 \*/$'
	["S"]='^/\* SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0 \*/$'
	["py"]='^# SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
	["sh"]='^# SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
	["md"]='^SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
	["rst"]='^SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
	["yaml"]='^# SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
	["yml"]='^# SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
	["rs"]='^// SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
)

# Makefile / Kconfig（无扩展名但需匹配）
declare -A FILENAME_PATTERNS=(
	["Makefile"]='^# SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
	["Kconfig"]='^# SPDX-License-Identifier: AGPL-3\.0-or-later OR Apache-2\.0$'
)

check_file() {
	local file="$1"
	local first_line
	first_line=$(head -n 1 "$file")

	local pattern=""

	# 先按文件名匹配（Makefile/Kconfig）
	local base
	base=$(basename "$file")
	if [[ -n "${FILENAME_PATTERNS[$base]:-}" ]]; then
		pattern="${FILENAME_PATTERNS[$base]}"
	else
		# 按扩展名匹配
		local ext="${file##*.}"
		pattern="${SPDX_PATTERNS[$ext]:-}"
	fi

	if [[ -z "$pattern" ]]; then
		# 未定义模式的文件类型：跳过（如二进制、.gitignore）
		return 0
	fi

	if ! [[ "$first_line" =~ $pattern ]]; then
		echo "FAIL: $file"
		echo "  expected first line: $EXPECTED"
		echo "  actual first line:   $first_line"
		EXIT_CODE=1
	fi
}

# 遍历仓库中所有受检文件
cd "$REPO_ROOT"
while IFS= read -r file; do
	# 跳过 vendor / third_party / .git
	case "$file" in
		vendor/*|third_party/*|.git/*) continue ;;
	esac
	check_file "$file"
done < <(git ls-files '*.c' '*.h' '*.S' '*.py' '*.sh' '*.md' '*.rst' \
	'*.yaml' '*.yml' '*.rs' 'Makefile' 'Kconfig' '*/Makefile' '*/Kconfig')

if [[ $EXIT_CODE -ne 0 ]]; then
	echo ""
	echo "Error: SPDX-License-Identifier check failed."
	echo "See: 110-spdx-license-compliance.md"
	exit 1
fi

echo "OK: all files have correct SPDX-License-Identifier."
exit 0
```

### 4.2 CI 集成

在 CI 配置中添加 spdx-check 阶段：

```yaml
# 文件: .github/workflows/ci.yml
spdx-check:
  stage: lint
  image: bash:latest
  script:
    - ./scripts/airymax/spdx-check.sh
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

### 4.3 Makefile 集成

```makefile
# 文件: Makefile（agentrt-liunx 仓库根）

.PHONY: spdx-check

## spdx-check: 验证所有文件 SPDX-License-Identifier 合规
spdx-check:
	@echo "  CHECK   SPDX-License-Identifier"
	@./scripts/airymax/spdx-check.sh
```

### 4.4 git pre-commit 钩子

```bash
#!/bin/bash
# 文件: .git/hooks/pre-commit
# 仅检查暂存文件

REPO_ROOT=$(git rev-parse --show-toplevel)
EXIT_CODE=0

for file in $(git diff --cached --name-only --diff-filter=ACM); do
	case "$file" in
		vendor/*|third_party/*|.git/*) continue ;;
	esac
	"$REPO_ROOT/scripts/airymax/spdx-check.sh" "$REPO_ROOT" \
		"$file" || EXIT_CODE=1
done

exit $EXIT_CODE
```

---

## 5. 例外与豁免

### 5.1 第三方代码

第三方代码（`vendor/`、`third_party/`）保留其原始 SPDX 标识，不强制 AGPL/Apache。但必须在 `THIRD_PARTY_LICENSES.md` 中登记：

```markdown
| 路径 | 原始许可证 | 来源 |
|------|----------|------|
| vendor/libbpf/ | LGPL-2.1-or-later | github.com/libbpf/libbpf |
```

### 5.2 自动生成文件

由工具自动生成的文件（如 `scripts/airymax/gen_headers.py` 输出的 `.h`）若无法在首行插入 SPDX，需在生成器中添加 SPDX 注释行，或在 `.spdx-ignore` 文件中豁免：

```
# 文件: .spdx-ignore
# 格式: <文件路径> <豁免理由>

# 自动生成头文件
include/airymax/generated/version.h 由 gen_version.py 生成
```

---

## 6. 与 checkpatch SPDX_LICENSE_TAG 的协作

### 6.1 checkpatch 校验范围

OLK-6.6 `scripts/checkpatch.pl:3772-3807` 的 `SPDX_LICENSE_TAG` 规则校验：

1. SPDX 注释样式与文件类型匹配（`.c`/`.h`/`.S` 用 `/* */`）
2. SPDX 标识存在且格式正确
3. 许可证字符串在 `LICENSES/` 目录中合法

agentrt-liunx 在 `LICENSES/` 目录中需添加 `AGPL-3.0-or-later.txt` 与 `Apache-2.0.txt` 两个许可证文本文件，供 checkpatch 校验合法性。

### 6.2 LICENSES 目录结构

```
LICENSES/
├── preferred/
│   ├── AGPL-3.0-or-later.txt    # AGPL-3.0+ 全文
│   └── Apache-2.0.txt           # Apache-2.0 全文
├── exceptions/
│   └── ...                      # 例外条款
└── deprecated/
    └── ...                      # 废弃许可证（不应使用）
```

### 6.3 双重校验

agentrt-liunx CI 同时运行：

1. `spdx-check.sh`（本规范定义）—— 校验许可证字符串为 `AGPL-3.0-or-later OR Apache-2.0`
2. `checkpatch.pl`（OLK-6.6）—— 校验注释样式与许可证合法性

两者皆通过方可合并。

---

## 7. 速查表：SPDX 标识速查

| 文件类型 | 首行 SPDX 样式 | OLK-6.6 checkpatch 行号 |
|---------|--------------|------------------------|
| `.c` | `/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */` | `checkpatch.pl:3773` |
| `.h` | `/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */` | `checkpatch.pl:3773` |
| `.S` | `/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */` | `checkpatch.pl:3773` |
| `.py` | `# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |
| `.sh` | `# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |
| `Makefile` | `# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |
| `.md` | `SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |
| `.yaml` | `# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |
| `.rs` | `// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |

---

## 8. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，定义 SPDX 强制规范 + 检查脚本 | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
