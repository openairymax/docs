Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）checkpatch 规则映射表

> **文档定位**: agentrt-linux（AirymaxOS，极境智能体操作系统）checkpatch 规则与编码规范的映射表。提取 OLK-6.6 `scripts/checkpatch.pl` 中的 ERROR/WARNING/CHK 规则，映射到 agentrt-linux 规则编号体系（OS-STD-CODE-NNN），供 CI 门禁、Code Review 与开发者自查使用。
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 目的与范围

本文件研究 OLK-6.6 `scripts/checkpatch.pl`（239KB，约 7800 行）中定义的所有补丁检查规则，提取 ≥30 条 ERROR/WARNING 类型规则，建立到 agentrt-linux 规则编号体系的映射。

每条规则包含以下字段：

- **checkpatch 规则名**：`scripts/checkpatch.pl` 中的 `ERROR("RULE_NAME")` / `WARN("RULE_NAME")` / `CHK("RULE_NAME")` 调用标识
- **级别**：ERROR / WARNING / CHECK
- **检查内容**：规则触发条件的中文化描述
- **OLK-6.6 源码行号**：`scripts/checkpatch.pl:行号` 格式
- **agentrt-linux 规则编号**：OS-STD-CODE-NNN
- **示例**：触发该规则的代码片段

> **交叉引用**：本文件是 checkpatch 规则到 agentrt-linux 规则编号的逐条映射表（规则映射层）。checkpatch 在 7 层验证体系中的框架定位（第 2 层静态分析 + 第 3 层预提交 + 第 4 层 CI 门禁）、三级报告策略、CI 调用方式与退出码处理，详见 [06-toolchain-and-automation.md](06-toolchain-and-automation.md) §5.1（checkpatch.pl）与 §1.2-§1.4（7 层验证体系第 2-4 层）。.clang-format 配置项与格式规则定义，详见 [02-code-format.md](02-code-format.md) 与 [80-clang-format-enforcement.md](80-clang-format-enforcement.md)（clang-format 与 checkpatch 的协作流水线顺序见 80 §5）。

### 0.2 checkpatch 规则分级约定

agentrt-linux 对 checkpatch 规则的分级处理：

| checkpatch 级别 | agentrt-linux 处理策略 |
|----------------|---------------------|
| **ERROR** | CI 强制门禁，0 容忍，违反即拒合并 |
| **WARNING** | CI 强制门禁，0 容忍（升级为 ERROR），违反即拒合并 |
| **CHECK** | Code Review 提示项，不阻塞合并但建议修复 |

### 0.3 规则编号体系

agentrt-linux 编码规则编号采用 `OS-STD-CODE-NNN` 三段式：

- `OS`：归属 agentrt-linux（OS 发行版）
- `STD-CODE`：编码标准 - 代码规则子类
- `NNN`：三位数字序号（001-999）

---

## 1. 字符串与内存安全规则

### 规则 1：STRCPY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `STRCPY` |
| **级别** | WARNING |
| **检查内容** | 检测 `strcpy()` 调用，建议改用 `strscpy`。`strcpy` 无边界检查，可导致线性溢出。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7029-7031` |
| **agentrt-linux 规则编号** | OS-STD-CODE-010 |
| **示例** | `strcpy(dst, src);` → 触发 "Prefer strscpy over strcpy" |

### 规则 2：STRLCPY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `STRLCPY` |
| **级别** | WARNING |
| **检查内容** | 检测 `strlcpy()` 调用，建议改用 `strscpy`。`strlcpy` 多余读源且可能溢出。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7035-7037` |
| **agentrt-linux 规则编号** | OS-STD-CODE-010 |
| **示例** | `strlcpy(dst, src, len);` → 触发 "Prefer strscpy over strlcpy" |

### 规则 3：STRNCPY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `STRNCPY` |
| **级别** | WARNING |
| **检查内容** | 检测 `strncpy()` 调用，建议改用 `strscpy`/`strscpy_pad`/`__nonstring`。`strncpy` 不保证 NUL 终止。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7041-7043` |
| **agentrt-linux 规则编号** | OS-STD-CODE-010 |
| **示例** | `strncpy(dst, src, n);` → 触发 "Prefer strscpy, strscpy_pad, or __nonstring" |

### 规则 4：MEMSET

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MEMSET` |
| **级别** | ERROR / WARNING |
| **检查内容** | 检测 `memset(p, 0, sizeof(*p))` 模式，建议改用 `kzalloc` 或显式零初始化。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6978-6981` |
| **agentrt-linux 规则编号** | OS-STD-CODE-018 |
| **示例** | `p = kmalloc(...); memset(p, 0, sizeof(*p));` → 建议用 `kzalloc` |

### 规则 5：ALLOC_WITH_MULTIPLY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ALLOC_WITH_MULTIPLY` |
| **级别** | WARNING |
| **检查内容** | 检测 `kmalloc(n * sizeof(...))` 模式，建议改用 `kmalloc_array`/`kcalloc`（含溢出检查）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7255` |
| **agentrt-linux 规则编号** | OS-STD-CODE-012 |
| **示例** | `kmalloc(n * sizeof(struct x), GFP)` → 建议用 `kcalloc(n, sizeof(*p), GFP)` |

### 规则 6：ALLOC_SIZEOF_STRUCT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ALLOC_SIZEOF_STRUCT` |
| **级别** | CHECK |
| **检查内容** | 检测 `kmalloc(sizeof(struct xxx), ...)` 模式，建议改用 `sizeof(*p)`。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7229` |
| **agentrt-linux 规则编号** | OS-STD-CODE-012 |
| **示例** | `kmalloc(struct foo, GFP)` → 建议用 `kmalloc(sizeof(*p), GFP)` |

### 规则 7：KREALLOC_ARG_REUSE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `KREALLOC_ARG_REUSE` |
| **级别** | WARNING |
| **检查内容** | 检测 `p = krealloc(p, ...)` 模式，重用参数 p 在失败时会泄漏原 p。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7268` |
| **agentrt-linux 规则编号** | OS-STD-CODE-019 |
| **示例** | `p = krealloc(p, n, GFP);` → 应用临时变量 |

---

## 2. 缩进与空格规则

### 规则 8：CODE_INDENT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `CODE_INDENT` |
| **级别** | ERROR |
| **检查内容** | 检测用空格而非 Tab 进行代码缩进。代码必须用 Tab 缩进。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:3916` |
| **agentrt-linux 规则编号** | OS-KER-011 |
| **示例** | `    foo();`（4 空格） → 触发 "code indent should use tabs where possible" |

### 规则 9：SPACE_BEFORE_TAB

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPACE_BEFORE_TAB` |
| **级别** | WARNING |
| **检查内容** | 检测 Tab 前有空格的混合缩进（如 `\t  \t`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:3926` |
| **agentrt-linux 规则编号** | OS-KER-011 |
| **示例** | `  \tfoo` → 触发 "please, no space before tabs" |

### 规则 10：TABSTOP

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TABSTOP` |
| **级别** | WARNING |
| **检查内容** | 检测行内出现 Tab 字符（用于代码对齐位置之外）。Tab 仅用于行首缩进。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:3967` |
| **agentrt-linux 规则编号** | OS-KER-011 |
| **示例** | `printk("a\tb");` 中 Tab 用于对齐 → 触发警告 |

### 规则 11：TRAILING_WHITESPACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TRAILING_WHITESPACE` |
| **级别** | ERROR |
| **检查内容** | 检测行尾空白字符（空格或 Tab）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:3599` |
| **agentrt-linux 规则编号** | OS-STD-CODE-020 |
| **示例** | `foo();  ` → 触发 "trailing whitespace" |

### 规则 12：SPACING

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPACING` |
| **级别** | ERROR / WARNING |
| **检查内容** | 检测关键字后缺空格、`sizeof` 后多余空格、逗号后缺空格、`(`后多余空格等多种空格规范。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4967,5014,5192-5348,5444-5491,5665-5708`（多处） |
| **agentrt-linux 规则编号** | OS-STD-CODE-021 |
| **示例** | `if(x)` → 触发 "space required after the 'if' keyword" |

### 规则 13：LEADING_SPACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `LEADING_SPACE` |
| **级别** | WARNING |
| **检查内容** | 检测文件首行有空格而非 Tab 缩进。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4161` |
| **agentrt-linux 规则编号** | OS-KER-011 |
| **示例** | 文件首行 `    #include` → 触发警告 |

---

## 3. 大括号与控制流规则

### 规则 14：OPEN_BRACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `OPEN_BRACE` |
| **级别** | ERROR |
| **检查内容** | 检测大括号位置错误：非函数块（if/for/while/switch）的 `{` 应在行尾，函数的 `{` 应在新行首。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4361,4568,4931,4950,7204` |
| **agentrt-linux 规则编号** | OS-STD-CODE-022 |
| **示例** | `if (x)\n{` → 触发 "open brace '{' following if go on the same line" |

### 规则 15：BRACKET_SPACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `BRACKET_SPACE` |
| **级别** | ERROR |
| **检查内容** | 检测数组声明 `[` 前多余空格（如 `char buf [8]`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:5054` |
| **agentrt-linux 规则编号** | OS-STD-CODE-021 |
| **示例** | `char buf [8]` → 触发 "space prohibited before open square bracket" |

### 规则 16：SWITCH_CASE_INDENT_LEVEL

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SWITCH_CASE_INDENT_LEVEL` |
| **级别** | ERROR |
| **检查内容** | 检测 `switch` 与 `case` 标签未对齐到同一列（应同列而非 case 多缩进一级）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4326` |
| **agentrt-linux 规则编号** | OS-STD-CODE-023 |
| **示例** | `switch` 后 `case` 多缩进一级 → 触发错误 |

### 规则 17：ASSIGN_IN_IF

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ASSIGN_IN_IF` |
| **级别** | ERROR |
| **检查内容** | 检测 `if ((a = b))` 在 if 条件中赋值的反模式。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:5708` |
| **agentrt-linux 规则编号** | OS-STD-CODE-024 |
| **示例** | `if ((ret = foo()))` → 建议拆分为 `ret = foo(); if (ret)` |

### 规则 18：TRAILING_STATEMENTS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TRAILING_STATEMENTS` |
| **级别** | ERROR |
| **检查内容** | 检测 `if/while` 后同行跟语句（如 `if (x) foo();`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:5754,5792,5798,5809` |
| **agentrt-linux 规则编号** | OS-STD-CODE-025 |
| **示例** | `if (x) do_this();` → 触发 "trailing statements should be on next line" |

### 规则 19：DEFAULT_NO_BREAK

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DEFAULT_NO_BREAK` |
| **级别** | WARNING |
| **检查内容** | 检测 `switch` 的 `case` 末尾缺少 `break`/`fallthrough`/`return`/`goto`/`continue`。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7344` |
| **agentrt-linux 规则编号** | OS-STD-CODE-015 |
| **示例** | `case A: foo(); case B:` → 触发 "break/return/goto/continue/fallthrough expected" |

### 规则 20：ELSE_AFTER_BRACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `ELSE_AFTER_BRACE` |
| **级别** | ERROR |
| **检查内容** | 检测 `}` 与 `else` 未同行（`}\nelse` 应为 `} else`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:5817` |
| **agentrt-linux 规则编号** | OS-STD-CODE-022 |
| **示例** | `}\nelse` → 触发 "else should follow close brace" |

### 规则 21：WHILE_AFTER_BRACE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `WHILE_AFTER_BRACE` |
| **级别** | ERROR |
| **检查内容** | 检测 `do { ... }\n while` 中 `while` 未与 `}` 同行（应为 `} while`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:5843` |
| **agentrt-linux 规则编号** | OS-STD-CODE-022 |
| **示例** | `}\nwhile (x);` → 触发 "while should follow close brace" |

---

## 4. 类型与声明规则

### 规则 22：NEW_TYPEDEFS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `NEW_TYPEDEFS` |
| **级别** | WARNING |
| **检查内容** | 检测新增 `typedef`，禁止为结构体/指针新增 typedef。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4783-4790` |
| **agentrt-linux 规则编号** | OS-KER-013 |
| **示例** | `typedef struct { ... } foo_t;` → 触发 "do not add new typedefs" |

### 规则 23：POINTER_LOCATION

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `POINTER_LOCATION` |
| **级别** | ERROR |
| **检查内容** | 检测 `*` 紧贴类型而非变量名（如 `char* p` 应为 `char *p`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4808,4835` |
| **agentrt-linux 规则编号** | OS-STD-CODE-026 |
| **示例** | `char* p` → 触发 "\"foo* bar\" should be \"foo *bar\"" |

### 规则 24：GLOBAL_INITIALISERS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `GLOBAL_INITIALISERS` |
| **级别** | ERROR |
| **检查内容** | 检测全局变量使用位置初始化器而非指定初始化器。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4662` |
| **agentrt-linux 规则编号** | OS-STD-CODE-014 |
| **示例** | `int arr[] = {1, 2, 3};` → 建议用 `{ [0] = 1, ... }` |

### 规则 25：INITIALISED_STATIC

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `INITIALISED_STATIC` |
| **级别** | ERROR |
| **检查内容** | 检测 `static` 变量初始化为 0/NULL（BSS 段已默认零初始化，无需显式）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4670` |
| **agentrt-linux 规则编号** | OS-STD-CODE-027 |
| **示例** | `static int x = 0;` → 触发 "do not initialise statics to 0" |

### 规则 26：AVOID_EXTERNS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `AVOID_EXTERNS` |
| **级别** | WARNING / CHECK |
| **检查内容** | 检测函数原型使用 `extern` 关键字（多余且使行变长）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7121,7141,7160,7168` |
| **agentrt-linux 规则编号** | OS-STD-CODE-028 |
| **示例** | `extern int foo(void);` → 触发 "function prototypes should not be extern" |

### 规则 27：FUNCTION_ARGUMENTS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `FUNCTION_ARGUMENTS` |
| **级别** | WARNING |
| **检查内容** | 检测函数原型参数无名称（参数应含描述性名称以助可读性）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7146,7180` |
| **agentrt-linux 规则编号** | OS-STD-CODE-029 |
| **示例** | `int foo(int, char*);` → 建议用 `int foo(int count, char *name)` |

### 规则 28：VOLATILE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `VOLATILE` |
| **级别** | WARNING |
| **检查内容** | 检测 `volatile` 使用，建议改用 `READ_ONCE`/`WRITE_ONCE`/屏障。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6275` |
| **agentrt-linux 规则编号** | OS-STD-CODE-030 |
| **示例** | `volatile int flag;` → 触发 "Use of volatile is usually wrong" |

---

## 5. 宏与预处理规则

### 规则 29：MULTISTATEMENT_MACRO_USE_DO_WHILE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MULTISTATEMENT_MACRO_USE_DO_WHILE` |
| **级别** | ERROR |
| **检查内容** | 检测多语句宏未用 `do { ... } while (0)` 包裹（避免 if 嵌套错乱）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6016,6019,6022` |
| **agentrt-linux 规则编号** | OS-STD-CODE-031 |
| **示例** | `#define FOO(a) a++; b--;` → 建议用 `do { a++; b--; } while (0)` |

### 规则 30：COMPLEX_MACRO

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `COMPLEX_MACRO` |
| **级别** | ERROR |
| **检查内容** | 检测宏内运算未加括号（参数与表达式都需括号保护优先级）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6022` |
| **agentrt-linux 规则编号** | OS-STD-CODE-032 |
| **示例** | `#define X(a) a + 1` → 建议用 `#define X(a) ((a) + 1)` |

### 规则 31：MACRO_WITH_FLOW_CONTROL

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MACRO_WITH_FLOW_CONTROL` |
| **级别** | WARNING |
| **检查内容** | 检测宏内含 `return`/`break`/`goto` 等控制流（破坏调用方控制流预期）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6073` |
| **agentrt-linux 规则编号** | OS-STD-CODE-033 |
| **示例** | `#define FOO(x) if (x < 0) return -1;` → 建议用 inline 函数 |

### 规则 32：TRAILING_SEMICOLON

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `TRAILING_SEMICOLON` |
| **级别** | WARNING |
| **检查内容** | 检测宏定义末尾多余分号（导致 `if (cond) MACRO;` 错乱）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6131` |
| **agentrt-linux 规则编号** | OS-STD-CODE-034 |
| **示例** | `#define FOO(x) do_bar(x);` → 建议去掉末尾分号 |

### 规则 33：MACRO_ARG_PRECEDENCE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MACRO_ARG_PRECEDENCE` |
| **级别** | CHECK |
| **检查内容** | 检测宏参数在表达式中未加括号（优先级风险）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6062` |
| **agentrt-linux 规则编号** | OS-STD-CODE-035 |
| **示例** | `#define X(a) a * 2` → 建议用 `(a) * 2` |

---

## 6. 注释与文档规则

### 规则 34：BLOCK_COMMENT_STYLE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `BLOCK_COMMENT_STYLE` |
| **级别** | WARNING |
| **检查内容** | 检测多行块注释样式不符（首行应几乎为空，左侧星号列对齐）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4038,4047,4070` |
| **agentrt-linux 规则编号** | OS-STD-CODE-036 |
| **示例** | `/* line1\n * line2 */` → 建议用 `/*\n * line1\n */` |

### 规则 35：NETWORKING_BLOCK_COMMENT_STYLE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `NETWORKING_BLOCK_COMMENT_STYLE` |
| **级别** | WARNING |
| **检查内容** | `net/` 与 `drivers/net/` 下文件的多行注释首行不应为空（特殊风格）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4028` |
| **agentrt-linux 规则编号** | OS-STD-CODE-036 |
| **示例** | net/ 下 `/*\n * foo */` → 建议用 `/* foo\n */` |

### 规则 36：C99_COMMENTS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `C99_COMMENTS` |
| **级别** | ERROR |
| **检查内容** | 检测 `//` 单行注释（部分场景禁止，建议用 `/* */`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4601` |
| **agentrt-linux 规则编号** | OS-STD-CODE-037 |
| **示例** | UAPI 头文件中 `// comment` → 触发 "do not use C99 // comments" |

---

## 7. 头文件与许可证规则

### 规则 37：MALFORMED_INCLUDE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MALFORMED_INCLUDE` |
| **级别** | ERROR |
| **检查内容** | 检测 `#include` 语法错误（如 `#include <foo.h` 缺 `>`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4590` |
| **agentrt-linux 规则编号** | OS-STD-CODE-038 |
| **示例** | `#include <foo.h` → 触发 "malformed #include" |

### 规则 38：UAPI_INCLUDE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `UAPI_INCLUDE` |
| **级别** | ERROR |
| **检查内容** | 检测内核内部代码包含 `include/uapi/` 头文件（应包含 `include/linux/`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4594` |
| **agentrt-linux 规则编号** | OS-STD-CODE-039 |
| **示例** | 内核代码 `#include <uapi/linux/foo.h>` → 建议用 `<linux/foo.h>` |

### 规则 39：SPDX_LICENSE_TAG

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPDX_LICENSE_TAG` |
| **级别** | WARNING |
| **检查内容** | 检测 SPDX 许可证标识缺失、格式错误或不被 LICENSES/ 支持。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:3776,3782,3787,3803,3812,3823` |
| **agentrt-linux 规则编号** | OS-STD-CODE-040 |
| **示例** | `// SPDX-License-Identifier: GPL-2.0` 在 .c 文件中 → 建议用 `/* */` 注释 |

---

## 8. 杂项与质量规则

### 规则 40：DEPRECATED_API

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DEPRECATED_API` |
| **级别** | ERROR / WARNING |
| **检查内容** | 检测调用 `deprecated_apis` 表中的废弃 API（如 `kmap`/`synchronize_sched`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6456-6458,7424` |
| **agentrt-linux 规则编号** | OS-STD-CODE-041 |
| **示例** | `kmap(page)` → 建议用 `kmap_local_page(page)` |

### 规则 41：DEEP_INDENTATION

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DEEP_INDENTATION` |
| **级别** | WARNING |
| **检查内容** | 检测续行过度缩进（>6 级）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4339` |
| **agentrt-linux 规则编号** | OS-STD-CODE-042 |
| **示例** | 续行缩进 8 级 → 触发警告 |

### 规则 42：SUSPECT_CODE_INDENT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SUSPECT_CODE_INDENT` |
| **级别** | WARNING |
| **检查内容** | 检测 if/else 块内代码缩进与条件不一致（混合 Tab/空格的典型症状）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:4474` |
| **agentrt-linux 规则编号** | OS-KER-011 |
| **示例** | `if (x)\n        foo();`（空格而非 Tab） → 触发 "suspect code indent" |

### 规则 43：RETURN_PARENTHESES

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `RETURN_PARENTHESES` |
| **级别** | ERROR |
| **检查内容** | 检测 `return (x);` 多余括号。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:5595` |
| **agentrt-linux 规则编号** | OS-STD-CODE-043 |
| **示例** | `return (ret);` → 建议用 `return ret;` |

### 规则 44：USE_NEGATIVE_ERRNO

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `USE_NEGATIVE_ERRNO` |
| **级别** | WARNING |
| **检查内容** | 检测 `return ENOENT;`（应返回负 errno `-ENOENT`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:5663` |
| **agentrt-linux 规则编号** | OS-STD-CODE-044 |
| **示例** | `return ENOENT;` → 建议用 `return -ENOENT;` |

### 规则 45：DATE_TIME

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DATE_TIME` |
| **级别** | ERROR |
| **检查内容** | 检测代码中嵌入 `__DATE__`/`__TIME__`（导致不可重现构建）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7359` |
| **agentrt-linux 规则编号** | OS-STD-CODE-045 |
| **示例** | `pr_info("built at %s %s", __DATE__, __TIME__);` → 触发错误 |

### 规则 46：EXECUTE_PERMISSIONS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `EXECUTE_PERMISSIONS` |
| **级别** | ERROR |
| **检查内容** | 检测源码文件（.c/.h）被设为可执行权限。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:2957` |
| **agentrt-linux 规则编号** | OS-STD-CODE-046 |
| **示例** | `-rwxr-xr-x foo.c` → 触发 "do not set execute permissions for source files" |

### 规则 47：DOS_LINE_ENDINGS

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `DOS_LINE_ENDINGS` |
| **级别** | ERROR |
| **检查内容** | 检测 CRLF（\r\n）行尾，必须用 LF（\n）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:3592` |
| **agentrt-linux 规则编号** | OS-STD-CODE-047 |
| **示例** | 文件含 `\r\n` → 触发 "DOS line endings" |

### 规则 48：MISSING_EOF_NEWLINE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MISSING_EOF_NEWLINE` |
| **级别** | WARNING |
| **检查内容** | 检测文件末尾无换行符（POSIX 要求文件以换行结尾）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:3893` |
| **agentrt-linux 规则编号** | OS-STD-CODE-048 |
| **示例** | 文件末行无 `\n` → 触发 "Missing a newline at the end of the file" |

### 规则 49：SPLIT_STRING

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `SPLIT_STRING` |
| **级别** | WARNING |
| **检查内容** | 检测字符串字面量被折断（如 `"foo"\n"bar"`）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6286` |
| **agentrt-linux 规则编号** | OS-KER-012 |
| **示例** | `pr_info("foo " "bar");` → 触发 "quoted string split across lines" |

### 规则 50：FLEXIBLE_ARRAY

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `FLEXIBLE_ARRAY` |
| **级别** | ERROR |
| **检查内容** | 检测结构体末尾使用零长/单元素数组，应改用 C99 柔性数组 `items[]`。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7479` |
| **agentrt-linux 规则编号** | OS-STD-CODE-049 |
| **示例** | `struct x { int n; struct y items[1]; };` → 建议用 `items[]` |

### 规则 51：MEMORY_BARRIER

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `MEMORY_BARRIER` |
| **级别** | WARNING |
| **检查内容** | 检测 `smp_mb()`/`smp_rmb()`/`smp_wmb()`，建议用 `smp_store_release`/`smp_load_acquire`。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6695,6706` |
| **agentrt-linux 规则编号** | OS-STD-CODE-050 |
| **示例** | `smp_mb();` → 建议用 `smp_store_release()` |

### 规则 52：INLINE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `INLINE` |
| **级别** | WARNING |
| **检查内容** | 检测 `inline` 滥用（>3 行函数不建议 inline）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6757` |
| **agentrt-linux 规则编号** | OS-STD-CODE-051 |
| **示例** | 大函数加 `inline` → 触发 "inline is usually wrong" |

### 规则 53：LIKELY_MISUSE

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `LIKELY_MISUSE` |
| **级别** | WARNING |
| **检查内容** | 检测 `likely()`/`unlikely()` 滥用（不应在普通条件分支上用）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7461,7488` |
| **agentrt-linux 规则编号** | OS-STD-CODE-052 |
| **示例** | `if (likely(x > 0))` → 触发 "likely/unlikely misuse" |

### 规则 54：CONST_STRUCT

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `CONST_STRUCT` |
| **级别** | WARNING |
| **检查内容** | 检测 `static struct foo` 应为 `static const struct foo`（只读结构体应 const 化）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:7433` |
| **agentrt-linux 规则编号** | OS-STD-CODE-053 |
| **示例** | `static struct foo default = {...};` → 建议用 `static const struct foo` |

### 规则 55：INLINE_LOCATION

| 字段 | 内容 |
|------|------|
| **checkpatch 规则名** | `INLINE_LOCATION` |
| **级别** | ERROR |
| **检查内容** | 检测 `inline` 函数定义在 .c 文件而非 .h 文件（应放头文件以便共享）。 |
| **OLK-6.6 源码行号** | `scripts/checkpatch.pl:6750` |
| **agentrt-linux 规则编号** | OS-STD-CODE-054 |
| **示例** | `.c` 文件中 `static inline int foo(void) {...}` 且被多文件用 → 触发错误 |

---

## 9. 规则汇总与统计

### 9.1 按级别统计

| 级别 | 规则数 | 占比 |
|------|--------|------|
| ERROR | 23 | 41.8% |
| WARNING | 26 | 47.3% |
| CHECK | 6 | 10.9% |
| **总计** | **55** | **100%** |

### 9.2 按主题分类统计

| 主题 | 规则数 | agentrt-linux 规则编号区间 |
|------|--------|--------------------------|
| 字符串与内存安全 | 7 | OS-STD-CODE-010/012/018/019 |
| 缩进与空格 | 6 | OS-KER-011 / OS-STD-CODE-020/021 |
| 大括号与控制流 | 8 | OS-STD-CODE-015/022-025 |
| 类型与声明 | 7 | OS-KER-013 / OS-STD-CODE-014/026-030 |
| 宏与预处理 | 5 | OS-STD-CODE-031-035 |
| 注释与文档 | 3 | OS-STD-CODE-036/037 |
| 头文件与许可证 | 3 | OS-STD-CODE-038-040 |
| 杂项与质量 | 16 | OS-STD-CODE-041-054 |

### 9.3 与 12 条强化规则的交叉引用

下表展示 checkpatch 规则如何支撑 `10-coding-style/C_coding_style_standard_supplement.md` 的 12 条强化规则：

| 强化规则编号 | 强化规则名 | 对应 checkpatch 规则 |
|------------|----------|---------------------|
| OS-KER-011 | Tab-8 缩进强制 | CODE_INDENT, SPACE_BEFORE_TAB, TABSTOP, LEADING_SPACE, SUSPECT_CODE_INDENT |
| OS-KER-009 | goto 集中错误处理 | （间接，无直接 checkpatch 规则） |
| OS-KER-008 | bool 仅用于返回值与栈变量 | （间接，无直接 checkpatch 规则） |
| OS-KER-007 | 内核态禁 float | （由 -mno-80387 编译选项强制，无 checkpatch 规则） |
| OS-KER-012 | 行宽 80 列硬上限 | SPLIT_STRING（部分） |
| OS-STD-CODE-014 | 指定初始化器强制 | GLOBAL_INITIALISERS |
| OS-STD-CODE-015 | fallthrough; 显式穿透 | DEFAULT_NO_BREAK |
| OS-KER-013 | 禁止结构体 typedef | NEW_TYPEDEFS |
| OS-KER-014 | 零警告门禁 | （checkpatch 自身 0 ERROR/WARNING 退出码） |
| OS-STD-CODE-010 | strscpy 强制替代 | STRCPY, STRLCPY, STRNCPY |
| OS-KER-015 | 禁止 BUG()/BUG_ON() | （间接，由 deprecated API 登记） |
| OS-STD-CODE-012 | sizeof(*p) 替代 sizeof(struct) | ALLOC_SIZEOF_STRUCT, ALLOC_WITH_MULTIPLY |

---

## 10. CI 集成与门禁配置

### 10.1 checkpatch CI 调用

agentrt-linux CI 在每次合并请求中按以下方式调用 checkpatch：

```bash
# 基准路径（agentrt-linux 仓库根）
$ ./scripts/checkpatch.pl --no-tree --no-summary --no-tree \
    --strict --codespell --min-conf-desc-length=1 \
    --max-line-length=80 \
    $(git diff origin/main...HEAD --name-only)
```

关键参数说明：

- `--strict`：启用额外检查（CHK 级别提升为 WARNING）
- `--max-line-length=80`：行宽 80 列硬上限（对应 `.clang-format: ColumnLimit: 80`）
- `--no-tree`：不依赖内核源码树
- `--codespell`：启用拼写检查

### 10.2 退出码处理

| 退出码 | 含义 | agentrt-linux 处理 |
|--------|------|------------------|
| 0 | 0 ERROR / 0 WARNING | 通过，允许合并 |
| 1 | ≥1 ERROR | 拒绝合并 |
| 2 | 0 ERROR / ≥1 WARNING | 拒绝合并（WARNING 升级为 ERROR） |

### 10.3 白名单机制

部分场景需豁免 checkpatch 规则，需在 `.checkpatch-ignore` 文件中登记并经维护者批准：

```
# 文件: .checkpatch-ignore
# 格式: <文件路径> <规则名> <豁免理由>

# 例: 第三方贡献的兼容层
drivers/airymax/legacy/compat.c STRCPY 已知遗留代码，迁移中
```

---

## 11. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，覆盖 55 条 checkpatch 规则映射 | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
