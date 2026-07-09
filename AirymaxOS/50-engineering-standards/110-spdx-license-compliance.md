Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）SPDX 许可证策略与合规规范

> **文档定位**: agentrt-linux（AirymaxOS，极境智能体操作系统）的 SPDX 许可证策略矩阵与合规强制规范。本卷合并原 09-license-strategy.md（策略矩阵）与 110-spdx-license-compliance.md（合规执行），消解两文档间许可证字符串不一致的合规风险，建立统一的 5 类文件许可证分类体系。
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **父文档**: [工程标准规范 README](README.md)
> **配套文档**: [00-engineering-standards-handbook.md](./00-engineering-standards-handbook.md)（SSoT 索引）、[08-compliance-checklist.md](./08-compliance-checklist.md)（合规检查 STD-CODE-01）
> **理论根基**: Linux 6.6 内核基线 `COPYING`（GPL-2.0 WITH Linux-syscall-note）+ SPDX 规范
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 背景：历史三套策略并存

审查报告 TS-P0-15 发现 agentrt-linux 文档与代码中存在三套 SPDX 许可证策略，且各类文件的适用边界未声明：

| 历史策略 | 出现位置 | 适用范围（隐含） |
|---------|---------|----------------|
| `AGPL-3.0-or-later OR Apache-2.0` | 开源设计文档 | 文档 |
| `GPL-2.0-only` | 内核代码示例 | 内核态代码 |
| `Apache-2.0` | 检查脚本示例 | 检查脚本 |

**合规风险**：AGPL-3.0 与 GPL-2.0 在专利条款与反 DRM 条款上存在不兼容。若同一发行版混合两类许可证代码而无明确边界声明，将构成许可证合规风险。此外，110-spdx-license-compliance.md 曾规定所有文件统一使用 `AGPL-3.0-or-later OR Apache-2.0`，与 09-license-strategy.md 的 5 类分类策略冲突——内核态代码必须遵循 Linux 6.6 内核 GPL-2.0 基线，不可统一使用 AGPL-3.0。

### 0.2 合并决策

本卷合并 09-license-strategy.md（策略矩阵）与 110-spdx-license-compliance.md（合规执行），以 09 的 5 类分类策略为权威源，参考 OLK-6.6 `COPYING`（GPL-2.0 WITH Linux-syscall-note）建立统一的许可证体系。原 09-license-strategy.md 已废弃，所有引用重定向至本卷。

### 0.3 OLK-6.6 源码路径

- `COPYING`（OLK-6.6 顶层）：GPL-2.0 WITH Linux-syscall-note 许可证全文
- `scripts/checkpatch.pl:3772-3807`：`SPDX_LICENSE_TAG` 规则检查
- `scripts/checkpatch.pl:3773-3777`：SPDX 注释样式检查（`.[chsS]` 文件需用对应注释符）
- `scripts/checkpatch.pl:3780-3783`：缺失或格式错误的 SPDX 标识检查
- `scripts/checkpatch.pl:3784-3789`：SPDX 许可证合法性校验（在 `LICENSES/` 中查询）
- `LICENSES/`（OLK-6.6 顶层目录）：许可证文本存放目录

### 0.4 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

---

## 1. 许可证策略矩阵（5 类文件）

### 1.1 规则 OS-STD-SPDX-001：按文件类型分类使用许可证

agentrt-linux 仓库中所有文件按以下 5 类分类，每类使用对应的 SPDX 许可证标识：

| 类别 | 文件类型 | SPDX 标识 | 适用子仓 | 对齐依据 |
|------|---------|-----------|---------|---------|
| **A 内核态代码** | `.c` / `.h`（内核态） | `GPL-2.0-only WITH Linux-syscall-note` | airymaxos-kernel / security / memory 的内核态部分 | Linux 6.6 内核 `COPYING` |
| **B 用户态代码** | `.c` / `.h` / `.rs` / `.py` / `.ts`（用户态） | `AGPL-3.0-or-later OR Apache-2.0` | services / cognition / cloudnative / system | agentrt 同源用户态策略 |
| **C 文档** | `.md` / `.rst` | `AGPL-3.0-or-later OR Apache-2.0` | 全部 8 子仓 `Documentation/` | 与用户态一致 |
| **D 构建脚本** | `Makefile` / `Kbuild` / `Kconfig` / `.cmake` / `meson.build` | `GPL-2.0` | kernel / security / memory / services | 内核构建系统惯例 |
| **E 检查脚本** | `.sh` / `.py`（CI/检查脚本） | `Apache-2.0` | 全部 8 子仓 `scripts/` | 轻量许可，便于社区复用 |

### 1.2 各类文件 SPDX 标签示例

#### A 内核态代码

```c
// SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note
/*
 * airymaxos_ipc.c - agentrt-linux AgentsIPC 内核实现
 *
 * Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
 */
```

> **说明**：`WITH Linux-syscall-note` 例外允许 UAPI 头文件中的系统调用接口定义以更宽松方式被用户态引用，对齐 Linux 6.6 内核 `COPYING` 的 `Linux-syscall-note` 例外声明。

#### B 用户态代码

```c
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
/* Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved. */
```

```rust
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
//! agentrt-linux Rust kernel module: Airymax task accounting.
```

#### C 文档

```markdown
Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）文档标题

> **文档定位**: ...
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
```

#### D 构建脚本

```makefile
# SPDX-License-Identifier: GPL-2.0
# Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
```

#### E 检查脚本

```bash
#!/bin/bash
# SPDX-License-Identifier: Apache-2.0
# Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
```

---

## 2. 许可证边界规则

### 2.1 内核态与用户态边界

| 边界 | 规则 | 说明 |
|------|------|------|
| 内核态 → 用户态 | UAPI 头文件使用 `GPL-2.0-only WITH Linux-syscall-note` | `Linux-syscall-note` 例外允许用户态引用系统调用接口 |
| 用户态 → 内核态 | 用户态代码不链接内核态代码 | 通过 syscall / io_uring / AgentsIPC 消息通信，无代码级链接 |
| [SC] 共享契约层 | 6 个 `include/airymax/*.h` 头文件 | 采用 `GPL-2.0-only WITH Linux-syscall-note`，因含 UAPI 定义；两端引用方式符合各自许可证 |

### 2.2 IRON-9 v2 三层共享模型的许可证约束

| 层次 | 许可证策略 | 说明 |
|------|-----------|------|
| **[SC] 共享契约层** | `GPL-2.0-only WITH Linux-syscall-note` | 6 个头文件含 UAPI 定义，遵循内核 UAPI 许可证 |
| **[SS] 语义同源层** | 各自独立许可证 | agentrt 端用户态用 AGPL-3.0 OR Apache-2.0；agentrt-linux 端内核态用 GPL-2.0 WITH Linux-syscall-note |
| **[IND] 完全独立层** | 各自独立许可证 | 按文件类型适用本矩阵 §1.1 |

### 2.3 构建脚本与检查脚本的区分

| 脚本类型 | 判定标准 | 许可证 |
|---------|---------|--------|
| 构建脚本 | 参与内核/模块编译过程的 Makefile/Kbuild/Kconfig | `GPL-2.0` |
| 检查脚本 | 不参与编译，仅用于 CI 检查/格式校验/禁词扫描的 `.sh`/`.py` | `Apache-2.0` |

> **判定原则**：若脚本被 `make` 调用且其输出参与编译产物，归为构建脚本（D 类）；若脚本仅在 CI 流水线中独立运行且不产生编译产物，归为检查脚本（E 类）。

---

## 3. SPDX 标识格式规则

### 3.1 规则 OS-STD-SPDX-002：注释样式正确性

SPDX 标识**必须**使用与文件类型匹配的注释样式：

**OLK-6.6 源码路径**: `scripts/checkpatch.pl:3772-3778`

| 文件类型 | 正确注释样式 | 错误注释样式 |
|---------|------------|------------|
| `.c`/`.h`/`.S` | `/* SPDX-License-Identifier: ... */` | `// SPDX-...`（禁止） |
| `.py` | `# SPDX-License-Identifier: ...` | `"""SPDX-..."""`（禁止） |
| `.sh`/`Makefile` | `# SPDX-License-Identifier: ...` | `// SPDX-...`（禁止） |
| `.md`/`.rst` | `SPDX-License-Identifier: ...`（无注释符，在文档元数据区） | `<!-- SPDX-... -->`（禁止） |
| `.rs` | `// SPDX-License-Identifier: ...` | — |

### 3.2 规则 OS-STD-SPDX-003：SPDX 标识必须在首行

所有 `.c`/`.h`/`.S`/`.py`/`.sh`/`Makefile`/`Kconfig`/`.rs` 文件的 SPDX 标识**必须**在文件首行。`.md`/`.rst` 文档文件的 SPDX 标识应在文档元数据区（标题后、正文前）。

### 3.3 违规示例

#### 缺失 SPDX 标识

```c
/*
 * agentrt_ipc.c - agentrt-linux IPC channel implementation
 * WRONG: 缺 SPDX-License-Identifier 行
 */
```

#### 错误注释样式

```c
// SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note
/* WRONG: .c 文件应用 /* */ 块注释，非 // */
```

#### 错误许可证字符串（内核态代码）

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
/* WRONG: 内核态代码必须用 GPL-2.0-only WITH Linux-syscall-note */
```

#### 错误许可证字符串（用户态代码）

```c
/* SPDX-License-Identifier: GPL-2.0-only */
/* WRONG: 用户态代码应用 AGPL-3.0-or-later OR Apache-2.0 */
```

---

## 4. CI 分类检查脚本

### 4.1 spdx-check-classified.sh 脚本

agentrt-linux CI 使用以下脚本按 5 类许可证分类校验所有文件：

```bash
#!/bin/bash
# SPDX-License-Identifier: Apache-2.0
#
# spdx-check-classified.sh: 按 110-spdx-license-compliance.md §1.1 五类许可证分类校验 SPDX 标签
# 参考 OLK-6.6 COPYING（GPL-2.0 WITH Linux-syscall-note）
# Exits non-zero if any file is missing or has incorrect SPDX tag.

set -euo pipefail

REPO_ROOT="${1:-$(git rev-parse --show-toplevel)}"
EXIT_CODE=0

# A 类：内核态代码（.c/.h）→ GPL-2.0-only WITH Linux-syscall-note
KERNEL_DIRS="airymaxos-kernel/ airymaxos-security/ airymaxos-memory/"

# B 类：用户态代码（.c/.h/.rs/.py/.ts）→ AGPL-3.0-or-later OR Apache-2.0
USERSPACE_DIRS="airymaxos-services/ airymaxos-cognition/ airymaxos-cloudnative/ airymaxos-system/"

# D 类：构建脚本（Makefile/Kbuild/Kconfig）→ GPL-2.0
# E 类：检查脚本（scripts/ 下的 .sh/.py）→ Apache-2.0

check_kernel_code() {
    local file="$1"
    local first_line
    first_line=$(head -n 1 "$file")
    if ! echo "$first_line" | grep -q "GPL-2.0-only WITH Linux-syscall-note"; then
        echo "FAIL (A-内核态代码): $file"
        echo "  expected: GPL-2.0-only WITH Linux-syscall-note"
        echo "  actual:   $first_line"
        EXIT_CODE=1
    fi
}

check_userspace_code() {
    local file="$1"
    local first_line
    first_line=$(head -n 1 "$file")
    if ! echo "$first_line" | grep -q "AGPL-3.0-or-later OR Apache-2.0"; then
        echo "FAIL (B-用户态代码): $file"
        echo "  expected: AGPL-3.0-or-later OR Apache-2.0"
        echo "  actual:   $first_line"
        EXIT_CODE=1
    fi
}

check_build_script() {
    local file="$1"
    local first_line
    first_line=$(head -n 1 "$file")
    if ! echo "$first_line" | grep -q "GPL-2.0"; then
        echo "FAIL (D-构建脚本): $file"
        echo "  expected: GPL-2.0"
        echo "  actual:   $first_line"
        EXIT_CODE=1
    fi
}

check_check_script() {
    local file="$1"
    local first_line
    first_line=$(head -n 1 "$file")
    if ! echo "$first_line" | grep -q "Apache-2.0"; then
        echo "FAIL (E-检查脚本): $file"
        echo "  expected: Apache-2.0"
        echo "  actual:   $first_line"
        EXIT_CODE=1
    fi
}

check_document() {
    local file="$1"
    local first_two_lines
    first_two_lines=$(head -n 3 "$file")
    if ! echo "$first_two_lines" | grep -q "AGPL-3.0-or-later OR Apache-2.0"; then
        echo "FAIL (C-文档): $file"
        echo "  expected: AGPL-3.0-or-later OR Apache-2.0 in header"
        EXIT_CODE=1
    fi
}

cd "$REPO_ROOT"

# A 内核态代码
for dir in $KERNEL_DIRS; do
    if [ -d "$dir" ]; then
        find "$dir" -name "*.c" -o -name "*.h" | while read -r f; do
            check_kernel_code "$f"
        done
    fi
done

# B 用户态代码
for dir in $USERSPACE_DIRS; do
    if [ -d "$dir" ]; then
        find "$dir" -name "*.c" -o -name "*.h" -o -name "*.rs" -o -name "*.py" -o -name "*.ts" | while read -r f; do
            check_userspace_code "$f"
        done
    fi
done

# C 文档
find . -path "*/Documentation/*" -name "*.md" -o -path "*/Documentation/*" -name "*.rst" | while read -r f; do
    check_document "$f"
done

# D 构建脚本
find . -name "Makefile" -o -name "Kbuild" -o -name "Kconfig" -o -name "*.cmake" -o -name "meson.build" | while read -r f; do
    check_build_script "$f"
done

# E 检查脚本（scripts/ 下的 .sh/.py）
find scripts/ -name "*.sh" -o -name "*.py" 2>/dev/null | while read -r f; do
    check_check_script "$f"
done

if [[ $EXIT_CODE -ne 0 ]]; then
    echo ""
    echo "Error: SPDX-License-Identifier check failed."
    echo "See: 50-engineering-standards/110-spdx-license-compliance.md"
    exit 1
fi

echo "OK: all files have correct SPDX-License-Identifier (5-class matrix)."
exit 0
```

### 4.2 CI 集成

```yaml
# 文件: .github/workflows/ci.yml
spdx-check:
  stage: lint
  image: bash:latest
  script:
    - ./scripts/airymax/spdx-check-classified.sh
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

### 4.3 Makefile 集成

```makefile
# SPDX-License-Identifier: GPL-2.0
# 文件: Makefile（agentrt-linux 仓库根）

.PHONY: spdx-check

## spdx-check: 按 5 类许可证分类校验所有文件 SPDX 合规
spdx-check:
	@echo "  CHECK   SPDX-License-Identifier (5-class matrix)"
	@./scripts/airymax/spdx-check-classified.sh
```

---

## 5. 合规检查集成

### 5.1 SPDX 标签检查规则

[08-compliance-checklist.md](./08-compliance-checklist.md) STD-CODE-01 的 SPDX 检查须按本卷 §1.1 的 5 类许可证标识进行分类校验：

| 检查项 | 合格标准 |
|--------|----------|
| 内核态 `.c`/`.h` | 首行含 `// SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note` |
| 用户态 `.c`/`.h`/`.rs` | 首行含 `// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` |
| 文档 `.md` | 文档元数据区含 SPDX 标签（AGPL-3.0 OR Apache-2.0）或 Copyright 头 |
| 构建脚本 `Makefile`/`Kbuild` | 首行含 `# SPDX-License-Identifier: GPL-2.0` |
| 检查脚本 `.sh`/`.py` | 首行含 `# SPDX-License-Identifier: Apache-2.0` |

---

## 6. 例外与豁免

### 6.1 第三方代码

第三方代码（`vendor/`、`third_party/`）保留其原始 SPDX 标识，不强制本卷 5 类分类。但必须在 `THIRD_PARTY_LICENSES.md` 中登记：

```markdown
| 路径 | 原始许可证 | 来源 |
|------|----------|------|
| vendor/libbpf/ | LGPL-2.1-or-later | github.com/libbpf/libbpf |
```

### 6.2 自动生成文件

由工具自动生成的文件（如 `scripts/airymax/gen_headers.py` 输出的 `.h`）若无法在首行插入 SPDX，需在生成器中添加 SPDX 注释行，或在 `.spdx-ignore` 文件中豁免：

```
# 文件: .spdx-ignore
# 格式: <文件路径> <豁免理由>

# 自动生成头文件
include/airymax/generated/version.h 由 gen_version.py 生成
```

---

## 7. 与 checkpatch SPDX_LICENSE_TAG 的协作

### 7.1 checkpatch 校验范围

OLK-6.6 `scripts/checkpatch.pl:3772-3807` 的 `SPDX_LICENSE_TAG` 规则校验：

1. SPDX 注释样式与文件类型匹配（`.c`/`.h`/`.S` 用 `/* */`）
2. SPDX 标识存在且格式正确
3. 许可证字符串在 `LICENSES/` 目录中合法

### 7.2 LICENSES 目录结构

agentrt-linux 在 `LICENSES/` 目录中需包含以下许可证文本文件，供 checkpatch 校验合法性：

```
LICENSES/
├── preferred/
│   ├── GPL-2.0-only.txt             # GPL-2.0 全文（内核态代码）
│   ├── AGPL-3.0-or-later.txt        # AGPL-3.0+ 全文（用户态代码 + 文档）
│   └── Apache-2.0.txt               # Apache-2.0 全文（检查脚本 + 双许可）
├── exceptions/
│   └── Linux-syscall-note           # UAPI 例外条款
└── deprecated/
    └── ...                          # 废弃许可证（不应使用）
```

### 7.3 双重校验

agentrt-linux CI 同时运行：

1. `spdx-check-classified.sh`（本规范定义）—— 按 5 类分类校验许可证字符串
2. `checkpatch.pl`（OLK-6.6）—— 校验注释样式与许可证合法性

两者皆通过方可合并。

---

## 8. 速查表：SPDX 标识速查

| 文件类型 | 类别 | SPDX 标识 | OLK-6.6 checkpatch 行号 |
|---------|------|-----------|------------------------|
| 内核态 `.c` | A | `/* SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note */` | `checkpatch.pl:3773` |
| 内核态 `.h` | A | `/* SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note */` | `checkpatch.pl:3773` |
| 内核态 `.S` | A | `/* SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note */` | `checkpatch.pl:3773` |
| 用户态 `.c`/`.h` | B | `/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */` | `checkpatch.pl:3773` |
| 用户态 `.rs` | B | `// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |
| 用户态 `.py`/`.ts` | B | `# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` | — |
| 文档 `.md`/`.rst` | C | `SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0`（元数据区） | — |
| `Makefile`/`Kbuild` | D | `# SPDX-License-Identifier: GPL-2.0` | — |
| `Kconfig`/`.cmake` | D | `# SPDX-License-Identifier: GPL-2.0` | — |
| 检查脚本 `.sh` | E | `# SPDX-License-Identifier: Apache-2.0` | — |
| 检查脚本 `.py` | E | `# SPDX-License-Identifier: Apache-2.0` | — |

---

## 9. 五维原则映射

| 原则 | 落地点 |
|------|--------|
| **K-2 接口契约化** | UAPI 头文件许可证采用 `Linux-syscall-note` 例外，明确接口契约的可引用边界 |
| **E-7 文档即代码** | 许可证策略矩阵本身纳入版本控制，与代码同源审查 |
| **A-4 完美主义** | 三套历史策略并存与两文档间许可证字符串不一致是"隐藏瑕疵"，本卷消解为统一的 5 类矩阵 |
| **IRON-9 v2 同源且部分代码共享** | [SC] 共享契约层 6 个头文件的许可证遵循内核 UAPI 规范，与 agentrt 端引用方式兼容 |

---

## 10. 相关文档

- [工程标准 README](README.md)：工程标准规范主索引与总纲
- [00-engineering-standards-handbook.md](./00-engineering-standards-handbook.md)：SSoT 索引与 IRON-9 v2 三层共享模型
- [08-compliance-checklist.md](./08-compliance-checklist.md)：规范符合性检查机制（STD-CODE-01 SPDX 检查）
- [01-coding-standards.md](./01-coding-standards.md)：代码规范（SPDX 标签示例）
- Linux 6.6 内核基线 `COPYING`（GPL-2.0 WITH Linux-syscall-note）为同源参考
- SPDX 规范 https://spdx.org/licenses/ 为许可证标识权威源

---

## 11. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建：合并 09-license-strategy.md 与 110-spdx-license-compliance.md，建立统一 5 类许可证分类体系，消解两文档间许可证字符串不一致（内核态代码 AGPL-3.0 → GPL-2.0 WITH Linux-syscall-note） | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0