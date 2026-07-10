Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）规范符合性检查机制

> **文档定位**： agentrt-linux（AirymaxOS）工程标准规范符合性检查清单与自动化验证机制\
> **版本**： 0.1.1（文档体系完成）\
> **最后更新**： 2026-07-07\
> **父文档**： [工程标准规范 README](README.md)\
> **核心约束**： IRON-9 v2 同源且部分代码共享——[SC] 共享契约层 6 个头文件落地于 `include/airymax/`

---

## 1. 检查机制定位

### 1.1 目的

建立系统化的规范符合性检查机制，对所有设计文档和代码实现进行标准化审查，减少因不规范导致的潜在技术风险。本检查机制是用户决策指南第 4 点的直接落地：

> "制定并严格执行 agentrt-linux 工程标准规范；对所有设计文档和代码实现进行标准化审查；建立规范符合性检查机制，减少因不规范导致的潜在技术风险"

### 1.2 检查范围

| 范围 | 路径 | 检查类型 | 优先级 |
|------|------|----------|--------|
| 开源设计文档 | `docs/AirymaxOS/` | 文档标准化 | P0 |
| 闭源参考文档 | 内部闭源文档路径 | 文档标准化（放宽禁词） | P1 |
| C 源码 | `atoms/` `daemons/` `commons/` `memoryrovol/` | 代码标准化 | P0 |
| CMake 构建系统 | `cmake/` `CMakeLists.txt` | 构建标准化 | P1 |
| 头文件 | `include/airymax/` | [SC] 契约层一致性 | P0 |

### 1.3 检查分级

| 级别 | 含义 | 阻断 CI | 处理时限 |
|------|------|---------|----------|
| **P0-CRITICAL** | 阻断性违规（禁词/缺失 Copyright/IRON-9 v2 不一致） | ✅ 是 | 立即修复 |
| **P1-MAJOR** | 重大违规（链接断裂/行数超标/缺少 Mermaid） | ✅ 是 | 24h 内修复 |
| **P2-MINOR** | 轻微违规（格式不规范/注释缺失） | ❌ 否 | 下次迭代修复 |

---

## 2. 文档标准化审查清单

### 2.1 Copyright 头部检查

**规则 STD-DOC-01**：所有 Markdown 文档必须以 Copyright 头部开头。

```markdown
Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
```

**检查脚本**：
```bash
#!/bin/bash
# check-copyright.sh
find docs/AirymaxOS/ -name "*.md" -exec grep -L "^Copyright (c) 2025-2026 SPHARX" {} \;
# 退出码 0 = 全部通过，非 0 = 有文件缺失 Copyright
```

**适用范围**：`docs/AirymaxOS/` 全部 `.md` 文件（含 README.md）。
**例外**：内部闭源文档可选 Copyright（建议有但不阻断）。

### 2.2 禁词检查（开源文档）

**规则 STD-DOC-02**：开源文档中严禁出现以下禁词（完整禁词清单维护于内部配置文件，此处仅给出描述性说明）。

| 禁词类别 | 替代表述 | 说明 |
|----------|----------|------|
| 内核发行版名称（含大小写/空格变体） | "内核发行版" / "行业标杆发行版" | 避免直接引用特定发行版 |
| 集成测试框架代号 | "集成测试框架" | 避免直接引用 |
| 内部代号（K 前缀） | — | 避免内部代号泄露 |
| 旧术语（中枢类） | "认知循环" / "CoreLoopThree" | 使用 Airymax 术语 |

> **禁词清单获取**：完整禁词清单（含大小写变体）存储于内部配置文件，检查脚本从该文件读取。

**检查脚本**：
```bash
#!/bin/bash
# check-forbidden-words.sh
# 禁词清单从闭源配置文件读取
FORBIDDEN_FILE="内部配置文件路径"  # 闭源禁词清单配置（仅授权人员访问实际路径）
if [ ! -f "$FORBIDDEN_FILE" ]; then
    echo "FATAL: Forbidden words config not found: $FORBIDDEN_FILE"
    exit 2
fi
FORBIDDEN=$(cat "$FORBIDDEN_FILE" | grep -v '^#' | tr '\n' '|' | sed 's/|$//')
VIOLATIONS=$(grep -rniE "$FORBIDDEN" docs/AirymaxOS/ --include="*.md" | wc -l)
if [ "$VIOLATIONS" -ne 0 ]; then
    echo "ERROR: $VIOLATIONS forbidden word(s) found in open-source docs"
    grep -rniE "$FORBIDDEN" docs/AirymaxOS/ --include="*.md"
    exit 1
fi
echo "OK: 0 forbidden words in open-source docs"
```

**适用范围**：`docs/AirymaxOS/` 全部 `.md` 文件。
**例外**：内部闭源文档允许引用（但路径引用不应泄露到开源文档）。

### 2.3 IRON-9 v2 三层标注检查

**规则 STD-DOC-03**：深度修订的子系统设计文档必须包含 IRON-9 v2 三层共享模型标注。

**检查项**：

| 检查项 | 合格标准 | 检查方法 |
|--------|----------|----------|
| §2 三层模型表 | 包含 [SC]/[SS]/[IND] 三行表 | grep "\[SC\]" + grep "\[SS\]" + grep "\[IND\]" |
| [SC] 头文件引用 | 引用 `include/airymax/*.h` | grep "include/airymax/" |
| [SS] API 映射表 | 包含 API 签名映射 | grep "\[SS\].*API" |
| [IND] 独立实现表 | 包含独立实现清单 | grep "\[IND\]" |
| agentrt 一致性检查 | 包含 15 项检查表 | grep "agentrt 一致性检查" |
| 里程碑 M0-M8 | 包含 M0-M8 里程碑 | grep "M0.*M8" |

**检查脚本**：
```bash
#!/bin/bash
# check-iron9-v2.sh
for doc in docs/AirymaxOS/20-modules/0[1-8]-*.md; do
    SC=$(grep -c "\[SC\]" "$doc")
    SS=$(grep -c "\[SS\]" "$doc")
    IND=$(grep -c "\[IND\]" "$doc")
    if [ "$SC" -eq 0 ] || [ "$SS" -eq 0 ] || [ "$IND" -eq 0 ]; then
        echo "WARN: $doc missing IRON-9 v2 annotations (SC=$SC SS=$SS IND=$IND)"
    fi
done
```

**适用范围**：`docs/AirymaxOS/20-modules/0[1-8]-*.md`（8 个模块设计文档）。

### 2.4 [SC] 头文件清单一致性检查

**规则 STD-DOC-04**：所有引用 [SC] 共享契约层的文档必须与 6 个头文件清单一致。

**6 个 [SC] 头文件清单**（权威清单，维护于 `project_memory.md`，详见 [00-engineering-standards-handbook.md §4.2](./00-engineering-standards-handbook.md)）：

| # | 头文件 | 子系统 | 内容摘要 |
|---|--------|--------|----------|
| 1 | `include/airymax/bpf_struct_ops.h` | eBPF | struct_ops 状态机 + common_value |
| 2 | `include/airymax/memory_types.h` | 记忆 | MemoryRovol L1-L4 + GFP 掩码 + PMEM 接口 |
| 3 | `include/airymax/security_types.h` | 安全 | capability 38 ID + LSM 254 ID + Cupolas blob + 派生模型 + Vault + 裁决 4 值 |
| 4 | `include/airymax/cognition_types.h` | 认知 | CoreLoopThree 阶段 + Thinkdual 模式 + LLM 推理阶段 + 上下文 + 能效 + GPU/NPU |
| 5 | `include/airymax/sched.h` | 调度 | SCHED_EXT 约束 + 任务描述符（'AGTS'）+ vtime + 优先级 + SLICE_DFL |
| 6 | `include/airymax/ipc.h` | IPC | IPC magic（'ARE1'）+ 128B 消息头 + SQE/CQE 操作码 |

**检查方法**：grep `include/airymax/` 引用，核对是否与上述 6 个头文件一致。

### 2.5 链接完整性检查

**规则 STD-DOC-05**：所有 Markdown 内部链接必须指向存在的文件。

**检查脚本**：
```bash
#!/bin/bash
# check-links.sh
find docs/AirymaxOS/ -name "*.md" -exec grep -oP '\[.*?\]\((?!http|#|mailto)([^)]+)\)' {} \; | \
  while IFS= read -r match; do
    link=$(echo "$match" | grep -oP '(?<=\()[^)]+(?=\))')
    # 解析相对路径
    dir=$(dirname "$doc")
    target="$dir/$link"
    if [ ! -f "$target" ]; then
      echo "BROKEN LINK: $doc -> $link"
    fi
  done
```

### 2.6 文档行数检查

**规则 STD-DOC-06**：设计文档行数应在合理范围内。

| 文档类型 | 目标行数 | 上限 | 下限 |
|----------|----------|------|------|
| 模块设计文档（20-modules/） | 400-600 | 700 | 300 |
| 数据流文档（40-dataflows/） | 500-800 | 900 | 400 |
| 测试文档（80-testing/） | 400-500 | 550 | 350 |
| 工程标准文档（50-engineering-standards/） | 500-1000 | 1200 | 400 |
| 闭源源码映射（02-olk66-source-mapping/） | 600-1200 | 1500 | 500 |
| 闭源技术规范（01-tech-reference/） | 700-1000 | 1200 | 600 |

### 2.7 Mermaid 图表检查

**规则 STD-DOC-07**：深度修订的文档应包含至少 2 个 Mermaid 图表。

**检查脚本**：
```bash
#!/bin/bash
# check-mermaid.sh
for doc in docs/AirymaxOS/20-modules/0[1-8]-*.md \
           docs/AirymaxOS/40-dataflows/0[1-4]-*.md; do
    MERMAID=$(grep -c '```mermaid' "$doc")
    if [ "$MERMAID" -lt 2 ]; then
        echo "WARN: $doc has only $MERMAID Mermaid diagrams (need >= 2)"
    fi
done
```

### 2.8 "五维正交 24 原则"引用检查

**规则 STD-DOC-08**：深度修订的文档应引用"五维正交 24 原则"至少 2 次。

**检查脚本**：
```bash
#!/bin/bash
# check-principles.sh
for doc in docs/AirymaxOS/80-testing/0[1-2]-*.md; do
    COUNT=$(grep -c "五维正交" "$doc")
    if [ "$COUNT" -lt 2 ]; then
        echo "WARN: $doc references '五维正交' only $COUNT times (need >= 2)"
    fi
done
```

---

## 3. 代码标准化审查清单

### 3.1 SPDX 许可证标签检查

**规则 STD-CODE-01**：所有 C/C++/Rust 源文件必须包含 SPDX 许可证标签。

**合格格式**：
```c
// SPDX-License-Identifier: Apache-2.0
/* Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved. */
```

> **适用范围说明**：`Apache-2.0` 适用于检查脚本（CI/格式校验/禁词扫描等 `.sh`/`.py` 脚本）。内核态代码使用 `GPL-2.0-only WITH Linux-syscall-note`，用户态代码与文档使用 `AGPL-3.0-or-later OR Apache-2.0`，构建脚本使用 `GPL-2.0`。完整 5 类文件许可证策略矩阵见 [110-spdx-license-compliance.md](./110-spdx-license-compliance.md)（原 09-license-strategy.md 已合并至此）。

**检查脚本**：
```bash
#!/bin/bash
# check-spdx.sh
find atoms/ daemons/ commons/ memoryrovol/ -name "*.c" -o -name "*.h" | \
  while read f; do
    if ! head -1 "$f" | grep -q "SPDX-License-Identifier"; then
      echo "MISSING SPDX: $f"
    fi
  done
```

**目标覆盖率**：≥ 95%（0.1.1 版本目标），100%（1.0.1 版本目标）。

### 3.2 命名规范检查

**规则 STD-CODE-02**：函数与变量命名遵循 [01-coding-standards.md](01-coding-standards.md) §3.1。

| 前缀 | 用途 | 示例 |
|------|------|------|
| `agentrt_` | agentrt 同源 API（[SS] 语义同源层） | `agentrt_ipc_send()` |
| `airymaxos_` | agentrt-linux 专属 API（[IND] 独立层） | `airymaxos_kthread_create()` |
| `AGENTRT_` | agentrt 宏/常量（[SC] 共享层） | `AGENTRT_API` / `AGENTRT_SYS_*` |
| `AIRYMAXOS_` | agentrt-linux 宏/常量（[IND] 独立层） | `AIRYMAXOS_CONFIG_*` |

**检查脚本**：
```bash
#!/bin/bash
# check-naming.sh
# 检查 agentrt_ 前缀是否仅用于 [SS] 同源 API
grep -rn "^void\|^int\|^static.*(" atoms/ daemons/ | \
  grep -v "agentrt_\|airymaxos_\|static inline\|main(" | \
  head -20
```

### 3.3 日志系统使用检查

**规则 STD-CODE-03**：所有运行时日志必须使用核心日志系统。

| 禁止 | 允许 |
|------|------|
| `fprintf(stderr, ...)` | `log_write(LOG_LEVEL_INFO, ...)` |
| `printf(...)` | `log_write_va(LOG_LEVEL_DEBUG, ...)` |
| `puts(...)` | `agentrt_print_info(...)`（构建期） |

**ANSI 颜色编码**：

| 级别 | 颜色 | 宏 |
|------|------|----|
| INFO | 蓝色 | `\033[34m` |
| WARN | 黄色 | `\033[33m` |
| ERROR | 红色 | `\033[31m` |
| FATAL | 品红 | `\033[35m` |
| DEBUG | 灰色 | `\033[90m` |
| OK | 绿色 | `\033[32m` |

**时间戳**：必须使用 `CLOCK_REALTIME` 对齐北京时间。

### 3.4 错误处理范式检查

**规则 STD-CODE-04**：遵循 Linux 内核 `goto` 集中错误处理范式。

**合格模式**：
```c
int function(...)
{
    int ret;
    struct resource *a, *b, *c;

    a = alloc_resource_a();
    if (!a)
        return -ENOMEM;

    b = alloc_resource_b();
    if (!b) {
        ret = -ENOMEM;
        goto out_free_a;
    }

    c = alloc_resource_c();
    if (!c) {
        ret = -ENOMEM;
        goto out_free_b;
    }

    /* ... 正常逻辑 ... */
    return 0;

out_free_b:
    free_resource_b(b);
out_free_a:
    free_resource_a(a);
    return ret;
}
```

**禁止模式**：
- `BUG()` / `BUG_ON()` → 改用 `WARN()` / `WARN_ON_ONCE()`
- `strcpy` / `strncpy` / `strlcpy` → 改用 `strscpy`
- `strcpy` / `sprintf` → 改用 `strscpy` / `snprintf`

### 3.5 CMake 构建系统检查

**规则 STD-CODE-05**：CMake 脚本遵循 `cmake/agentrt_print.cmake` 统一打印系统。

| 检查项 | 合格标准 |
|--------|----------|
| 打印函数 | 使用 `agentrt_print_ok/info/warn/error/fatal/debug/section` 而非 `message()` |
| 时间戳 | 使用 `_AGENTRT_TIMESTAMP` 宏生成 |
| ANSI 颜色 | 通过 `_AGENTRT_C_*` 变量，支持 `AGENTRT_BUILD_COLOR` 环境变量控制 |
| SPDX | CMake 文件头含 SPDX-License-Identifier |

---

## 4. IRON-9 v2 三层共享模型检查

### 4.1 [SC] 共享契约层一致性检查

**规则 STD-IRON-01**：[SC] 共享契约层的 6 个头文件必须 agentrt 与 agentrt-linux 完全一致。

**检查方法**：
```bash
#!/bin/bash
# check-sc-consistency.sh
SC_HEADERS="
    include/airymax/bpf_struct_ops.h
    include/airymax/memory_types.h
    include/airymax/security_types.h
    include/airymax/cognition_types.h
    include/airymax/sched.h
    include/airymax/ipc.h
"

for h in $SC_HEADERS; do
    # 对比 agentrt 仓库与 agentrt-linux 仓库的同名文件
    diff "agentrt/$h" "airymaxos/$h" > /dev/null 2>&1
    if [ $? -ne 0 ]; then
        echo "SC MISMATCH: $h differs between agentrt and agentrt-linux"
    fi
done
```

**合格标准**：6 个头文件 MD5 哈希完全一致。

### 4.2 [SS] 语义同源层 API 签名检查

**规则 STD-IRON-02**：[SS] 语义同源层的 API 签名必须一致（函数名、参数类型、返回值）。

**检查方法**：使用 `diff` 对比 agentrt 与 agentrt-linux 的头文件 API 声明部分。

### 4.3 [IND] 独立层隔离检查

**规则 STD-IRON-03**：[IND] 独立层不应引用对方仓库的内部实现。

**检查方法**：
```bash
#!/bin/bash
# check-ind-isolation.sh
# agentrt-linux（AirymaxOS）不应引用 agentrt 内部实现
grep -rn "#include.*agentrt/" airymaxos-kernel/ 2>/dev/null
# agentrt 不应引用 agentrt-linux 内部实现
grep -rn "#include.*airymaxos/" agentrt/ 2>/dev/null
```

**合格标准**：0 处违规。

### 4.4 主流发行版兼容性检查

**规则 STD-IRON-04**：agentrt-linux 工程思想与上游 Linux 保持一致性。

**检查项**：

| 检查项 | 验证方法 | 频率 |
|--------|----------|------|
| kthread 协议一致性 | 核对 `kthread_create/stop/park` API 签名 | 每次内核基线升级 |
| accel 接入模式一致性 | 核对 `drivers/accel/` 框架模式 | 每次加速器驱动新增 |
| drm_sched 模式一致性 | 核对 `drm_sched_main` 主循环结构 | 每次调度器修改 |
| Kconfig 风格一致性 | 核对 `CONFIG_` 前缀命名 | 每次新增配置项 |
| 调度类编号不冲突 | 核对 `SCHED_AGENT` 编号 | 每次调度类变更 |

---

## 5. 自动化检查集成

### 5.1 CI/CD 检查流水线

**规则 STD-CI-01**：CI/CD 流水线必须包含规范符合性检查阶段。

```yaml
# .github/workflows/compliance-check.yml
stages:
  - name: doc-standard-check
    steps:
      - run: scripts/check-copyright.sh
      - run: scripts/check-forbidden-words.sh
      - run: scripts/check-iron9-v2.sh
      - run: scripts/check-links.sh
      - run: scripts/check-mermaid.sh

  - name: code-standard-check
    steps:
      - run: scripts/check-spdx.sh
      - run: scripts/check-naming.sh
      - run: scripts/check-logging.sh

  - name: iron9-consistency-check
    steps:
      - run: scripts/check-sc-consistency.sh
      - run: scripts/check-ind-isolation.sh
      - run: scripts/check-distro-compat.sh
```

### 5.2 检查脚本目录结构

```
scripts/
├── check-copyright.sh           # STD-DOC-01 Copyright 头部检查
├── check-forbidden-words.sh     # STD-DOC-02 禁词检查
├── check-iron9-v2.sh            # STD-DOC-03 IRON-9 v2 标注检查
├── check-sc-consistency.sh      # STD-DOC-04 [SC] 头文件清单检查
├── check-links.sh               # STD-DOC-05 链接完整性检查
├── check-line-count.sh          # STD-DOC-06 文档行数检查
├── check-mermaid.sh             # STD-DOC-07 Mermaid 图表检查
├── check-principles.sh          # STD-DOC-08 五维原则引用检查
├── check-spdx.sh                # STD-CODE-01 SPDX 标签检查
├── check-naming.sh              # STD-CODE-02 命名规范检查
├── check-logging.sh             # STD-CODE-03 日志系统检查
├── check-error-handling.sh      # STD-CODE-04 错误处理范式检查
├── check-cmake.sh               # STD-CODE-05 CMake 构建系统检查
├── check-sc-consistency.sh      # STD-IRON-01 [SC] 一致性检查
├── check-ind-isolation.sh       # STD-IRON-03 [IND] 隔离检查
└── check-distro-compat.sh        # STD-IRON-04 主流发行版兼容性检查
```

### 5.3 检查报告格式

**合格报告**：
```
[2026-07-07 18:30:00] [  OK  ] STD-DOC-01 Copyright check: 64/64 files pass
[2026-07-07 18:30:01] [  OK  ] STD-DOC-02 Forbidden words: 0 violations
[2026-07-07 18:30:02] [  OK  ] STD-DOC-03 IRON-9 v2 annotations: 8/8 modules pass
[2026-07-07 18:30:03] [  OK  ] STD-DOC-05 Link integrity: 0 broken links
[2026-07-07 18:30:04] [  OK  ] STD-IRON-01 [SC] consistency: 6/6 headers identical
```

**违规报告**：
```
[2026-07-07 18:30:00] [ERROR] STD-DOC-02 Forbidden words: 2 violations
  - docs/AirymaxOS/20-modules/03-security.md:454: <禁词类别: 内核发行版名称>
  - docs/AirymaxOS/20-modules/05-cognition.md:599: <禁词类别: 内核发行版名称>
[2026-07-07 18:30:01] [ERROR] STD-DOC-05 Link integrity: 1 broken link
  - docs/AirymaxOS/20-modules/README.md:210 -> 09-obsolete.md (not found)
```

---

## 6. 规范编号体系

### 6.1 文档规范编号（STD-DOC-*）

| 编号 | 规则名称 | 检查脚本 | 优先级 |
|------|----------|----------|--------|
| STD-DOC-01 | Copyright 头部 | check-copyright.sh | P0 |
| STD-DOC-02 | 禁词检查（开源文档） | check-forbidden-words.sh | P0 |
| STD-DOC-03 | IRON-9 v2 三层标注 | check-iron9-v2.sh | P0 |
| STD-DOC-04 | [SC] 头文件清单一致性 | check-sc-consistency.sh | P0 |
| STD-DOC-05 | 链接完整性 | check-links.sh | P1 |
| STD-DOC-06 | 文档行数 | check-line-count.sh | P1 |
| STD-DOC-07 | Mermaid 图表 | check-mermaid.sh | P1 |
| STD-DOC-08 | 五维原则引用 | check-principles.sh | P2 |

### 6.2 代码规范编号（STD-CODE-*）

| 编号 | 规则名称 | 检查脚本 | 优先级 |
|------|----------|----------|--------|
| STD-CODE-01 | SPDX 许可证标签 | check-spdx.sh | P0 |
| STD-CODE-02 | 命名规范 | check-naming.sh | P1 |
| STD-CODE-03 | 日志系统使用 | check-logging.sh | P1 |
| STD-CODE-04 | 错误处理范式 | check-error-handling.sh | P1 |
| STD-CODE-05 | CMake 构建系统 | check-cmake.sh | P1 |

### 6.3 IRON-9 v2 规范编号（STD-IRON-*）

| 编号 | 规则名称 | 检查脚本 | 优先级 |
|------|----------|----------|--------|
| STD-IRON-01 | [SC] 共享契约层一致性 | check-sc-consistency.sh | P0 |
| STD-IRON-02 | [SS] 语义同源层 API 签名 | — | P1 |
| STD-IRON-03 | [IND] 独立层隔离 | check-ind-isolation.sh | P0 |
| STD-IRON-04 | 主流发行版兼容性 | check-distro-compat.sh | P1 |

---

## 7. 检查机制实施路线

### 7.1 0.1.1 版本（奠基）

- [x] 建立规范编号体系（STD-DOC/STD-CODE/STD-IRON）
- [x] 定义检查脚本目录结构
- [ ] 编写全部 16 个检查脚本
- [ ] 集成到 CI/CD 流水线
- [ ] 完成首次全量检查并修复违规

### 7.2 1.0.1 版本（开发）

- [ ] 检查脚本覆盖率 100%
- [ ] CI/CD 强制阻断 P0/P1 违规
- [ ] 自动生成检查报告（每日 + 每次提交）
- [ ] 与 7 层自动化验证集成（详见 [06-toolchain-and-automation.md](06-toolchain-and-automation.md)）

### 7.3 检查频率

| 检查类型 | 频率 | 触发 |
|----------|------|------|
| 禁词检查 | 每次提交 | git pre-commit hook + CI |
| Copyright 检查 | 每次提交 | CI |
| IRON-9 v2 标注检查 | 每次提交 | CI |
| 链接完整性检查 | 每日 | 定时任务 + CI |
| [SC] 一致性检查 | 每次提交 | CI（双向：agentrt + agentrt-linux） |
| 主流发行版兼容性检查 | 每次内核基线升级 | 手动 + CI |
| 全量检查 | 每日 | 定时任务 |

---

## 8. 相关文档

- [工程标准规范 README](README.md)——主索引与总纲
- [01-coding-standards.md](01-coding-standards.md)——代码规范（语义规则 SSoT）
- [02-code-format.md](02-code-format.md)——代码格式（格式规则 SSoT）
- [06-toolchain-and-automation.md](06-toolchain-and-automation.md)——工具链与自动化（7 层验证）
- [07-maintainers-and-governance.md](07-maintainers-and-governance.md)——维护者制度与治理
- [10-coding-style/](10-coding-style/)——编码规范子目录（C/Rust 编码风格与安全编码，引用 01/02 SSoT）

---

## 9. 文档变更记录

| 版本 | 日期 | 变更内容 | 变更人 |
|---|---|---|---|
| 0.1.1 | 2026-07-07 | 初始版本，建立规范符合性检查机制：8 项 STD-DOC + 5 项 STD-CODE + 4 项 STD-IRON 规则，16 个检查脚本规范，CI/CD 集成方案 | 工程规范委员会 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
