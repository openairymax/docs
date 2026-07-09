Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）.clang-format 配置与 CI 门禁规范

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）的 `.clang-format` 定制配置与 CI 自动格式化门禁规范。基于 OLK-6.6 `.clang-format`（689 行）提取关键配置项，制定 agentrt-liunx 专属版本，并通过 `make format-check` 在 CI 中强制执行。
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 目的与范围

本文件研究 OLK-6.6 `.clang-format`（689 行配置），提取关键格式化选项，制定 agentrt-liunx 定制版 `.clang-format`，并定义 CI 中的 `make format-check` 门禁流程。所有 agentrt-liunx 内核态 C 文件必须通过 `clang-format` 校验。

### 0.2 OLK-6.6 源码路径

- `.clang-format`（689 行，文件首行 `# SPDX-License-Identifier: GPL-2.0`）
- `.clang-format:2-9` —— 头部说明（要求 clang-format >= 11）
- `.clang-format:55` —— `ColumnLimit: 80`
- `.clang-format:645` —— `IndentWidth: 8`
- `.clang-format:687` —— `TabWidth: 8`
- `.clang-format:688` —— `UseTab: Always`
- `.clang-format:31-46` —— `BraceWrapping` 配置（K&R 风格）
- `.clang-format:48` —— `BreakBeforeBraces: Custom`
- `.clang-format:668` —— `PointerAlignment: Right`
- `.clang-format:71-635` —— `ForEachMacros` 列表（约 565 个宏）

### 0.3 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-liunx（OS 发行版）= 微内核（micro-kernel）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

---

## 1. 关键配置项详解

### 1.1 缩进配置（OS-KER-011 强制）

**OLK-6.6 源码路径**: `.clang-format:645,687,688`

```yaml
IndentWidth: 8			# 缩进宽度 8 字符
TabWidth: 8			# Tab 显示宽度 8
UseTab: Always			# 始终用 Tab（非空格）
ContinuationIndentWidth: 8	# 续行缩进宽度 8
ConstructorInitializerIndentWidth: 8
```

**理由**：8 字符缩进是 Linux 内核传统，作为"复杂度自然惩罚"——嵌套超过 3 层即提示应重构。`UseTab: Always` 确保 Tab 而非空格，配合 `IndentWidth`/`TabWidth` 一致为 8。

### 1.2 行宽配置（OS-KER-012 强制）

**OLK-6.6 源码路径**: `.clang-format:55`

```yaml
ColumnLimit: 80			# 行宽 80 列硬上限
```

**理由**：80 列限制源于终端显示传统，强制函数拆分并允许分屏 diff。`clang-format` 在格式化时自动折行至 80 列内。

### 1.3 大括号配置（K&R 风格）

**OLK-6.6 源码路径**: `.clang-format:31-46,48`

```yaml
BreakBeforeBraces: Custom	# 自定义大括号放置
BraceWrapping:
  AfterClass: false		# class 后不换行
  AfterControlStatement: false	# if/for/while 后不换行
  AfterEnum: false		# enum 后不换行
  AfterFunction: true		# 函数后换行（{ 在新行）
  AfterNamespace: true
  AfterStruct: false		# struct 后不换行
  AfterUnion: false
  BeforeCatch: false		# catch 前不换行
  BeforeElse: false		# else 前不换行
  IndentBraces: false
  SplitEmptyFunction: true
  SplitEmptyRecord: true
```

**理由**：这是 Linux K&R 风格——非函数块（if/for/while/switch）的 `{` 在行尾，函数的 `{` 在新行首。

### 1.4 指针对齐配置

**OLK-6.6 源码路径**: `.clang-format:62,668`

```yaml
DerivePointerAlignment: false	# 不从现有代码推导
PointerAlignment: Right		# * 贴变量名（char *p 而非 char* p）
```

**理由**：`*` 贴变量名避免 `char* a, b` 的歧义（b 不是指针）。

### 1.5 空格配置

**OLK-6.6 源码路径**: `.clang-format:672-685`

```yaml
SpaceAfterCStyleCast: false			# (cast) 后无空格
SpaceAfterTemplateKeyword: true		# template 后有空格
SpaceBeforeAssignmentOperators: true		# = 前有空格
SpaceBeforeCtorInitializerColon: true
SpaceBeforeInheritanceColon: true
SpaceBeforeParens: ControlStatementsExceptForEachMacros	# if/for 后有空格，foreach 宏除外
SpaceBeforeRangeBasedForLoopColon: true
SpaceInEmptyParentheses: false		# () 内无空格
SpacesBeforeTrailingComments: 1		# 注释前 1 个空格
SpacesInAngles: false			# <int> 内无空格
SpacesInContainerLiterals: false
SpacesInCStyleCastParentheses: false
SpacesInParentheses: false		# ( 内无空格
SpacesInSquareBrackets: false		# [ 内无空格
```

### 1.6 列对齐与折行配置

**OLK-6.6 源码路径**: `.clang-format:12-30,47-66,559-666`

```yaml
AlignAfterOpenBracket: Align		# 开括号后对齐续行
AlignConsecutiveAssignments: false	# 不对齐连续赋值
AlignConsecutiveDeclarations: false	# 不对齐连续声明
AlignEscapedNewlines: Left		# 转义换行左对齐
AlignOperands: true			# 操作数对齐
AlignTrailingComments: false		# 不对齐行尾注释
BinPackArguments: true			# 参数可打包多行
BinPackParameters: true
BreakBeforeBinaryOperators: None	# 二元操作符前不换行
BreakBeforeTernaryOperators: false	# 三元操作符前不换行
BreakStringLiterals: false		# 不折断字符串字面量
ReflowComments: false			# 不重排注释
```

### 1.7 Include 排序配置

**OLK-6.6 源码路径**: `.clang-format:637-642`

```yaml
IncludeBlocks: Preserve		# 保留现有 include 块结构
IncludeCategories:
  - Regex: '.*'
    Priority: 1
IncludeIsMainRegex: '(Test)?$'
SortIncludes: false			# 不自动排序 include（保留手动顺序）
SortUsingDeclarations: false
```

**理由**：Linux 内核 include 顺序有特定要求（系统头、内核头、子系统头），自动排序会破坏语义，故禁用。

### 1.8 短语句配置

**OLK-6.6 源码路径**: `.clang-format:20-24`

```yaml
AllowShortBlocksOnASingleLine: false		# 不允许单行块
AllowShortCaseLabelsOnASingleLine: false	# 不允许单行 case
AllowShortFunctionsOnASingleLine: None		# 不允许单行函数
AllowShortIfStatementsOnASingleLine: false	# 不允许单行 if
AllowShortLoopsOnASingleLine: false		# 不允许单行循环
```

### 1.9 ForEachMacros 配置

**OLK-6.6 源码路径**: `.clang-format:71-635`（约 565 个宏）

`ForEachMacros` 列表告知 `clang-format` 哪些宏是 `for_each` 风格的循环宏（如 `list_for_each_entry`），使其在格式化时正确处理。agentrt-liunx 在此基础上需追加自定义宏：

```yaml
ForEachMacros:
  # ... 保留 OLK-6.6 全部 565 个宏 ...
  - 'agentrt_for_each_task'		# agentrt-liunx 自定义
  - 'agentrt_for_each_chan'
  - 'airymax_for_each_token'
```

---

## 2. agentrt-liunx 定制版 .clang-format

### 2.1 完整配置文件

以下是 agentrt-liunx 仓库根目录 `.clang-format` 的完整内容（基于 OLK-6.6 定制）：

```yaml
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
#
# clang-format configuration for agentrt-liunx (AirymaxOS).
# Based on Linux 6.6 .clang-format (OLK-6.6) with agent-specific macros.
# Intended for clang-format >= 11.
#
# See: Documentation/process/clang-format.rst (upstream)
---
AccessModifierOffset: -4
AlignAfterOpenBracket: Align
AlignConsecutiveAssignments: false
AlignConsecutiveDeclarations: false
AlignEscapedNewlines: Left
AlignOperands: true
AlignTrailingComments: false
AllowAllParametersOfDeclarationOnNextLine: false
AllowShortBlocksOnASingleLine: false
AllowShortCaseLabelsOnASingleLine: false
AllowShortFunctionsOnASingleLine: None
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false
AlwaysBreakAfterDefinitionReturnType: None
AlwaysBreakAfterReturnType: None
AlwaysBreakBeforeMultilineStrings: false
AlwaysBreakTemplateDeclarations: false
BinPackArguments: true
BinPackParameters: true
BraceWrapping:
  AfterClass: false
  AfterControlStatement: false
  AfterEnum: false
  AfterFunction: true
  AfterNamespace: true
  AfterObjCDeclaration: false
  AfterStruct: false
  AfterUnion: false
  AfterExternBlock: false
  BeforeCatch: false
  BeforeElse: false
  IndentBraces: false
  SplitEmptyFunction: true
  SplitEmptyRecord: true
  SplitEmptyNamespace: true
BreakBeforeBinaryOperators: None
BreakBeforeBraces: Custom
BreakBeforeInheritanceComma: false
BreakBeforeTernaryOperators: false
BreakConstructorInitializersBeforeComma: false
BreakConstructorInitializers: BeforeComma
BreakAfterJavaFieldAnnotations: false
BreakStringLiterals: false
ColumnLimit: 80
CommentPragmas: '^ IWYU pragma:'
CompactNamespaces: false
ConstructorInitializerAllOnOneLineOrOnePerLine: false
ConstructorInitializerIndentWidth: 8
ContinuationIndentWidth: 8
Cpp11BracedListStyle: false
DerivePointerAlignment: false
DisableFormat: false
ExperimentalAutoDetectBinPacking: false
FixNamespaceComments: false
ForEachMacros:
  # ----- OLK-6.6 inherited (565 macros, abbreviated here) -----
  - 'list_for_each_entry'
  - 'list_for_each_entry_safe'
  - 'hlist_for_each_entry'
  - 'for_each_cpu'
  - 'for_each_online_cpu'
  - 'xa_for_each'
  # ... (full list inherited from OLK-6.6 .clang-format:71-635) ...
  # ----- agentrt-liunx custom -----
  - 'agentrt_for_each_task'
  - 'agentrt_for_each_chan'
  - 'agentrt_for_each_sched'
  - 'airymax_for_each_token'
  - 'airymax_for_each_mem_range'
IncludeBlocks: Preserve
IncludeCategories:
  - Regex: '.*'
    Priority: 1
IncludeIsMainRegex: '(Test)?$'
IndentCaseLabels: false
IndentGotoLabels: false
IndentPPDirectives: None
IndentWidth: 8
IndentWrappedFunctionNames: false
JavaScriptQuotes: Leave
JavaScriptWrapImports: true
KeepEmptyLinesAtTheStartOfBlocks: false
MacroBlockBegin: ''
MacroBlockEnd: ''
MaxEmptyLinesToKeep: 1
NamespaceIndentation: None
ObjCBinPackProtocolList: Auto
ObjCBlockIndentWidth: 8
ObjCSpaceAfterProperty: true
ObjCSpaceBeforeProtocolList: true
PenaltyBreakAssignment: 10
PenaltyBreakBeforeFirstCallParameter: 30
PenaltyBreakComment: 10
PenaltyBreakFirstLessLess: 0
PenaltyBreakString: 10
PenaltyExcessCharacter: 100
PenaltyReturnTypeOnItsOwnLine: 60
PointerAlignment: Right
ReflowComments: false
SortIncludes: false
SortUsingDeclarations: false
SpaceAfterCStyleCast: false
SpaceAfterTemplateKeyword: true
SpaceBeforeAssignmentOperators: true
SpaceBeforeCtorInitializerColon: true
SpaceBeforeInheritanceColon: true
SpaceBeforeParens: ControlStatementsExceptForEachMacros
SpaceBeforeRangeBasedForLoopColon: true
SpaceInEmptyParentheses: false
SpacesBeforeTrailingComments: 1
SpacesInAngles: false
SpacesInContainerLiterals: false
SpacesInCStyleCastParentheses: false
SpacesInParentheses: false
SpacesInSquareBrackets: false
Standard: Cpp03
TabWidth: 8
UseTab: Always
...
```

### 2.2 与 OLK-6.6 的差异

| 配置项 | OLK-6.6 值 | agentrt-liunx 值 | 差异说明 |
|--------|-----------|-----------------|---------|
| `ForEachMacros` | 565 个上游宏 | 565 + 5 个 agent 自定义 | 追加 agentrt 自定义宏 |
| SPDX 标识 | `GPL-2.0` | `AGPL-3.0-or-later OR Apache-2.0` | 许可证变更 |
| 其余所有项 | — | 完全继承 | 无差异 |

---

## 3. CI 门禁：make format-check

### 3.1 规则 OS-STD-FMT-001：format-check 强制门禁

agentrt-liunx CI 在每次合并请求中**必须**运行 `make format-check`，校验所有 C 文件符合 `.clang-format`。任何格式偏差即视为构建失败。

### 3.2 Makefile 集成

在 agentrt-liunx 仓库根 `Makefile` 中添加以下目标：

```makefile
# 文件: Makefile（agentrt-liunx 仓库根）

CLANG_FORMAT ?= clang-format
FORMAT_SOURCES := $(shell find . -name '*.c' -o -name '*.h' | \
	grep -vE '^(./vendor/|./third_party/)')

.PHONY: format-check format-diff format-apply

## format-check: 验证所有 C 文件符合 .clang-format
format-check:
	@echo "  CHECK   clang-format"
	@fail=0; \
	for f in $(FORMAT_SOURCES); do \
		if ! $(CLANG_FORMAT) --dry-run -W error $$f > /dev/null 2>&1; then \
			echo "  FAIL    $$f"; \
			fail=1; \
		fi; \
	done; \
	if [ $$fail -ne 0 ]; then \
		echo "Error: clang-format check failed. Run 'make format-apply' to fix."; \
		exit 1; \
	fi
	@echo "  OK      clang-format"

## format-diff: 显示需要格式化的 diff（不修改文件）
format-diff:
	@for f in $(FORMAT_SOURCES); do \
		$(CLANG_FORMAT) $$f | diff -u $$f - || true; \
	done

## format-apply: 应用 clang-format 自动修复
format-apply:
	@echo "  FORMAT  $(FORMAT_SOURCES)"
	@for f in $(FORMAT_SOURCES); do \
		$(CLANG_FORMAT) -i $$f; \
	done
```

### 3.3 Git pre-commit 钩子（推荐）

开发者可在本地 `.git/hooks/pre-commit` 中安装钩子，在提交前自动检查：

```bash
#!/bin/bash
# 文件: .git/hooks/pre-commit
# 检查暂存的 C 文件格式

CLANG_FORMAT=${CLANG_FORMAT:-clang-format}
exit_code=0

for f in $(git diff --cached --name-only --diff-filter=ACM -- '*.c' '*.h'); do
    if ! git show ":$f" | $CLANG_FORMAT --dry-run -W error - 2>/dev/null; then
        echo "clang-format check failed: $f"
        echo "Run 'clang-format -i $f' to fix."
        exit_code=1
    fi
done

exit $exit_code
```

### 3.4 CI 流水线集成

在 CI 配置文件（如 `.github/workflows/ci.yml` 或 `.gitlab-ci.yml`）中添加 format-check 阶段：

```yaml
# 文件: .github/workflows/ci.yml（示例）
stages:
  - lint
  - build
  - test

format-check:
  stage: lint
  image: clang-15:latest
  script:
    - make format-check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

---

## 4. 常见格式问题与修复

### 4.1 问题 1：空格 vs Tab 混用

**触发**：`CODE_INDENT` (checkpatch) + clang-format 偏差

**修复前**：

```c
static int foo(void)
{
    int x = 0;		/* WRONG: 4 空格 */
    return x;
}
```

**修复后**（`clang-format -i` 自动修复）：

```c
static int foo(void)
{
	int x = 0;		/* OK: Tab */
	return x;
}
```

### 4.2 问题 2：大括号位置错误

**修复前**：

```c
if (x)
{
	foo();
}
```

**修复后**：

```c
if (x) {
	foo();
}
```

### 4.3 问题 3：指针 * 位置

**修复前**：

```c
char* p;			/* WRONG: * 贴类型 */
```

**修复后**：

```c
char *p;			/* OK: * 贴变量名 */
```

### 4.4 问题 4：行宽超限

**修复前**：

```c
static int very_long_function_name(struct very_long_struct_name *param1, struct another_long_struct *param2)
```

**修复后**（clang-format 自动折行对齐到开括号）：

```c
static int very_long_function_name(struct very_long_struct_name *param1,
				   struct another_long_struct *param2)
```

### 4.5 问题 5：函数大括号位置

**修复前**：

```c
int foo(void) {
	return 0;
}
```

**修复后**（函数 `{` 在新行）：

```c
int foo(void)
{
	return 0;
}
```

---

## 5. 与 checkpatch 的协作

### 5.1 职责分工

| 工具 | 检查范围 | 优势 |
|------|---------|------|
| `clang-format` | 纯格式（缩进、空格、大括号、对齐） | 可自动修复 |
| `checkpatch.pl` | 格式 + 语义（如废弃 API、typedef 滥用、宏错误） | 语义感知 |

### 5.2 推荐流水线顺序

```
git commit
   ↓
clang-format --dry-run -W error   (格式门禁)
   ↓ pass
checkpatch.pl --strict            (语义门禁)
   ↓ pass
build with -Wall -Werror          (编译门禁)
   ↓ pass
tests                             (测试门禁)
   ↓ pass
merge allowed
```

### 5.3 互不覆盖的场景

以下场景 `clang-format` 无法检测，必须依赖 `checkpatch`：

- `strcpy` 调用（OS-STD-CODE-010）
- `BUG_ON` 使用（OS-KER-015）
- 新增 `typedef`（OS-KER-013）
- 缺失 SPDX 标识（OS-STD-CODE-040）
- 隐式 fall-through（OS-STD-CODE-015）

---

## 6. 速查表：关键配置项速查

| 配置项 | 值 | OLK-6.6 行号 | 对应规则 |
|--------|-----|------------|---------|
| `ColumnLimit` | 80 | `.clang-format:55` | OS-KER-012 |
| `IndentWidth` | 8 | `.clang-format:645` | OS-KER-011 |
| `TabWidth` | 8 | `.clang-format:687` | OS-KER-011 |
| `UseTab` | Always | `.clang-format:688` | OS-KER-011 |
| `BreakBeforeBraces` | Custom | `.clang-format:48` | OS-STD-CODE-022 |
| `PointerAlignment` | Right | `.clang-format:668` | OS-STD-CODE-026 |
| `SpaceBeforeParens` | ControlStatementsExceptForEachMacros | `.clang-format:677` | OS-STD-CODE-021 |
| `SortIncludes` | false | `.clang-format:670` | OS-STD-CODE-038 |
| `AllowShortFunctionsOnASingleLine` | None | `.clang-format:22` | OS-STD-CODE-022 |
| `ReflowComments` | false | `.clang-format:669` | OS-STD-CODE-036 |
| `BreakStringLiterals` | false | `.clang-format:54` | OS-KER-012 |

---

## 7. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，定义 agentrt-liunx 定制 .clang-format + CI 门禁 | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
