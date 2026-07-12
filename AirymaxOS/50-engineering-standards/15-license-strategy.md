Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）SPDX 许可证策略矩阵

> **⚠️ 本文档已废弃（2026-07-09）**：本卷的许可证策略矩阵与合规执行内容已合并至 [110-spdx-license-compliance.md](./110-spdx-license-compliance.md)。请以 110 卷为权威来源。本卷保留仅作为历史参考，不再维护更新。

> **文档定位**： agentrt-linux（AirymaxOS，极境智能体操作系统）工程标准规范第 9 卷——SPDX 许可证策略。本卷明确 agentrt-linux 全部 8 子仓中各类文件的许可证标识，消解历史三套策略并存且边界未声明的合规风险。\
> **版本**： 0.1.1\
> **最后更新**： 2026-07-09\
> **父文档**： [工程标准规范 README](README.md)\
> **配套文档**： [00-engineering-standards-handbook.md](./00-engineering-standards-handbook.md)（SSoT 索引）、[08-compliance-checklist.md](./08-compliance-checklist.md)（合规检查 STD-CODE-01）\
> **理论根基**： Linux 6.6 内核基线 `COPYING`（GPL-2.0 WITH Linux-syscall-note）+ SPDX 规范

---

## 1. 背景与问题

### 1.1 历史三套策略并存

经审查发现 agentrt-linux 文档与代码中存在三套 SPDX 许可证策略，且各类文件的适用边界未声明：

| 历史策略 | 出现位置 | 适用范围（隐含） |
|---------|---------|----------------|
| `AGPL-3.0-or-later OR Apache-2.0` | 开源设计文档 | 文档 |
| `GPL-2.0-only` | 内核代码示例 | 内核态代码 |
| `Apache-2.0` | 检查脚本示例 | 检查脚本 |

### 1.2 合规风险

- **AGPL-3.0 与 GPL-2.0 不兼容**：AGPL-3.0 基于 GPL-3.0，而 GPL-3.0 与 GPL-2.0 在专利条款与反 DRM 条款上存在不兼容。若同一发行版混合两类许可证代码而无明确边界声明，将构成许可证合规风险。
- **边界未声明**：检查脚本、构建脚本、文档、内核代码、用户态代码各自应使用何种许可证未明确，导致贡献者无法判断自己提交的代码应标注何种 SPDX 标识。

---

## 2. 许可证策略矩阵（5 类文件）

### 2.1 分类与许可证标识

| 类别 | 文件类型 | SPDX 标识 | 适用子仓 | 对齐依据 |
|------|---------|-----------|---------|---------|
| **A 内核态代码** | `.c` / `.h`（内核态） | `GPL-2.0-only WITH Linux-syscall-note` | airymaxos-kernel / security / memory 的内核态部分 | Linux 6.6 内核 `COPYING` |
| **B 用户态代码** | `.c` / `.h` / `.rs` / `.py` / `.ts`（用户态） | `AGPL-3.0-or-later OR Apache-2.0` | services / cognition / cloudnative / system | agentrt 同源用户态策略 |
| **C 文档** | `.md` / `.rst` | `AGPL-3.0-or-later OR Apache-2.0` | 全部 8 子仓 `Documentation/` | 与用户态一致 |
| **D 构建脚本** | `Makefile` / `Kbuild` / `Kconfig` / `.cmake` / `meson.build` | `GPL-2.0` | kernel / security / memory / services | 内核构建系统惯例 |
| **E 检查脚本** | `.sh` / `.py`（检查/CI 脚本） | `Apache-2.0` | 全部 8 子仓 `scripts/` | 轻量许可，便于社区复用 |

### 2.2 各类文件 SPDX 标签示例

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

#### C 文档

```markdown
<!-- SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 -->
Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
```

#### D 构建脚本

```makefile
# SPDX-License-Identifier: GPL-2.0
# Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
```

#### E 检查脚本（适用于检查脚本）

```bash
#!/bin/bash
# SPDX-License-Identifier: Apache-2.0
# Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
```

---

## 3. 许可证边界规则

### 3.1 内核态与用户态边界

| 边界 | 规则 | 说明 |
|------|------|------|
| 内核态 → 用户态 | UAPI 头文件使用 `GPL-2.0-only WITH Linux-syscall-note` | `Linux-syscall-note` 例外允许用户态引用系统调用接口 |
| 用户态 → 内核态 | 用户态代码不链接内核态代码 | 通过 syscall / io_uring / AgentsIPC 消息通信，无代码级链接 |
| [SC] 共享契约层 | 6 个 `include/airymax/*.h` 头文件 | 采用 `GPL-2.0-only WITH Linux-syscall-note`，因含 UAPI 定义；两端引用方式符合各自许可证 |

### 3.2 IRON-9 v2 三层共享模型的许可证约束

| 层次 | 许可证策略 | 说明 |
|------|-----------|------|
| **[SC] 共享契约层** | `GPL-2.0-only WITH Linux-syscall-note` | 6 个头文件含 UAPI 定义，遵循内核 UAPI 许可证 |
| **[SS] 语义同源层** | 各自独立许可证 | agentrt 端用户态用 AGPL-3.0 OR Apache-2.0；agentrt-linux 端内核态用 GPL-2.0 WITH Linux-syscall-note |
| **[IND] 完全独立层** | 各自独立许可证 | 按文件类型适用本矩阵 §2.1 |

### 3.3 构建脚本与检查脚本的区分

| 脚本类型 | 判定标准 | 许可证 |
|---------|---------|--------|
| 构建脚本 | 参与内核/模块编译过程的 Makefile/Kbuild/Kconfig | `GPL-2.0` |
| 检查脚本 | 不参与编译，仅用于 CI 检查/格式校验/禁词扫描的 `.sh`/`.py` | `Apache-2.0` |

> **判定原则**：若脚本被 `make` 调用且其输出参与编译产物，归为构建脚本（D 类）；若脚本仅在 CI 流水线中独立运行且不产生编译产物，归为检查脚本（E 类）。

---

## 4. 合规检查集成

### 4.1 SPDX 标签检查规则更新

[08-compliance-checklist.md](./08-compliance-checklist.md) STD-CODE-01 的 SPDX 检查须按本矩阵 §2.1 的 5 类许可证标识进行分类校验：

| 检查项 | 合格标准 |
|--------|----------|
| 内核态 `.c`/`.h` | 首行含 `// SPDX-License-Identifier: GPL-2.0-only WITH Linux-syscall-note` |
| 用户态 `.c`/`.h`/`.rs` | 首行含 `// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0` |
| 文档 `.md` | 首行或次行含 SPDX 标签（AGPL-3.0 OR Apache-2.0）或 Copyright 头 |
| 构建脚本 `Makefile`/`Kbuild` | 首行含 `# SPDX-License-Identifier: GPL-2.0` |
| 检查脚本 `.sh`/`.py` | 首行含 `# SPDX-License-Identifier: Apache-2.0` |

### 4.2 检查脚本示例

```bash
#!/bin/bash
# check-spdx-classified.sh
# 按 09-license-strategy.md §2.1 五类许可证分类校验 SPDX 标签

KERNEL_DIRS="kernel/ security/ memory/"
USERSPACE_DIRS="services/ cognition/ cloudnative/ system/"

# A 内核态代码
find $KERNEL_DIRS -name "*.c" -o -name "*.h" | while read f; do
    if ! head -1 "$f" | grep -q "GPL-2.0-only WITH Linux-syscall-note"; then
        echo "VIOLATION (kernel code): $f"
    fi
done

# D 构建脚本
find . -name "Makefile" -o -name "Kbuild" -o -name "Kconfig" | while read f; do
    if ! head -1 "$f" | grep -q "GPL-2.0"; then
        echo "VIOLATION (build script): $f"
    fi
done

# E 检查脚本（Apache-2.0）
find scripts/ -name "*.sh" -o -name "*.py" | while read f; do
    if ! head -1 "$f" | grep -q "Apache-2.0"; then
        echo "VIOLATION (check script): $f"
    fi
done
```

---

## 5. 五维原则映射

| 原则 | 落地点 |
|------|--------|
| **K-2 接口契约化** | UAPI 头文件许可证采用 `Linux-syscall-note` 例外，明确接口契约的可引用边界 |
| **E-7 文档即代码** | 许可证策略矩阵本身纳入版本控制，与代码同源审查 |
| **A-4 完美主义** | 三套历史策略并存是"隐藏瑕疵"，本卷消解为统一的 5 类矩阵 |
| **IRON-9 v2 同源且部分代码共享** | [SC] 共享契约层 6 个头文件的许可证遵循内核 UAPI 规范，与 agentrt 端引用方式兼容 |

---

## 6. 相关文档

- [工程标准 README](README.md)：工程标准规范主索引与总纲
- [00-engineering-standards-handbook.md](./00-engineering-standards-handbook.md)：SSoT 索引与 IRON-9 v2 三层共享模型
- [08-compliance-checklist.md](./08-compliance-checklist.md)：规范符合性检查机制（STD-CODE-01 SPDX 检查）
- [01-coding-standards.md](./01-coding-standards.md)：代码规范（SPDX 标签示例）
- Linux 6.6 内核基线 `COPYING`（GPL-2.0 WITH Linux-syscall-note）为同源参考
- SPDX 规范 https://spdx.org/licenses/ 为许可证标识权威源

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-09 | 初始版本：建立 5 类文件许可证策略矩阵，消解三套历史策略并存 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
