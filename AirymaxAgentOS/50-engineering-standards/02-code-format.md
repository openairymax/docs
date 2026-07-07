Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）代码格式标准

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）工程标准规范第 2 卷——代码格式。本卷规定缩进、行宽、大括号、空格、switch/case、空行与多语句等机械格式规则，并以"配置即代码"形式由 `.clang-format` / `.rustfmt.toml` / `.pyproject.toml` / `.prettierrc` 强制执行。
> **版本**: 1.0.1（开发）
> **最后更新**: 2026-07-06
> **同源映射**: `docs/ARCHITECTURAL_PRINCIPLES.md`（五维正交 24 原则）+ Linux 6.6 内核基线 `Documentation/process/coding-style.rst` + `.clang-format`（689 行 + 560 ForEachMacros）
> **理论根基**: Linux 6.6 内核基线工程思想 + Airymax 体系并行论（Multibody Cybernetic Intelligent System）

---

## 0. 章节定位

本卷回答"代码长什么样"——纯机械的格式规则。它不讨论"该怎么写代码"（01 代码规范）、"该怎么思考"（03 代码风格）或"工程体系怎么建立"（04 工程思想），只规定视觉层面的可由工具 100% 自动化的规则。agentrt-liunx 的格式哲学是：

> **配置即代码（OS-STD-201）**：每一条格式规则必须有对应的格式化工具配置项；规则若无法被工具强制，等同于不存在。

- **上游依赖**：04 工程思想定义"为什么"——双层稳定性、可读性优先、cache 影响是头等大事；本卷定义"长什么样"。
- **下游依赖**：06 工具链与自动化定义"格式检查在哪一层执行"——预提交钩子（Layer 3）+ CI 门禁（Layer 4）必须运行 `make format-check`；本卷定义"被检查的内容"。

本卷所有强制规则均赋予 **OS-KER**（内核工程）/ **OS-STD**（标准）编号，与 07 维护者制度与治理的"规则编号注册表"对齐。

### 0.1 关键术语

| 术语 | 定义 |
|------|------|
| Tab 宽度 | Tab 字符在显示时占用的列数；内核 C 代码统一为 8 |
| K&R 风格 | Kernighan & Ritchie 在《The C Programming Language》中确立的大括号放置风格 |
| 行尾空白 | 行末 Tab 或空格序列；agentrt-liunx 全部禁用 |
| 后继行 | 被断行的长行的续行；必须显著短于父行并右移 |
| 配置即代码 | 格式规则与格式化工具配置项一一对应，无人工裁量空间 |

---

## 1. 缩进规范

缩进是控制流块边界的视觉表达。agentrt-liunx 内核 C 代码严格沿用 Linux 6.6 内核基线 8 字符 Tab——这是 30+ 年沉淀的可读性经验，任何"4 空格更美观"的论调在内核上下文中都是噪声。

### 1.1 C 内核代码：Tab，8 字符宽（OS-KER-001）

**OS-KER-001**：airymaxos-kernel 与 airymaxos-security 全部 C 代码使用 Tab 缩进，Tab 宽度 8 字符；禁止用空格缩进。理由有二：

1. **易读**：连续看屏幕 20 小时后，8 字符缩进让块边界更易辨认。
2. **警告嵌套过深**：超过 3 层缩进即提示函数该重构；4 空格会容忍 6 层而不显眼，等同放弃警告。

```c
/* 正确：Tab 缩进，块边界清晰 */
int airymaxos_ipc_recv(struct agentrt_ipc_msg *msg)
{
	if (msg->hdr.flags & AGENTRT_IPC_F_NONBLOCK) {
		if (!try_wait_for_packet(msg)) {
			return -EAGAIN;
		}
	}
	return decode_payload(msg);
}
```

```c
/* 错误：4 空格缩进（隐藏了 4 层嵌套的坏味道） */
int airymaxos_ipc_recv(struct agentrt_ipc_msg *msg)
{
    if (msg->hdr.flags & AGENTRT_IPC_F_NONBLOCK) {
        if (!try_wait_for_packet(msg)) {
            return -EAGAIN;
        }
    }
    return decode_payload(msg);
}
```

### 1.2 Rust：4 空格（OS-STD-202）

**OS-STD-202**：Rust 代码（airymaxos-cognition、airymaxos-cloudnative、内核中的 Rust 模块）使用 4 空格缩进，由 `.rustfmt.toml` 强制。Rust 不沿用 Tab，因为 rustfmt 的默认值即如此，社区惯例已固化。

```rust
// 正确：4 空格
pub async fn submit_task(&self, desc: TaskDesc) -> Result<i32, Error> {
    let req = self.encode_request(desc)?;
    self.transport.send(req).await
}
```

### 1.3 Python：4 空格（PEP 8）（OS-STD-203）

**OS-STD-203**：Python 代码（SDK、DevStation 脚本）遵循 PEP 8，4 空格缩进，禁止 Tab。由 `black` + `isort` 强制。

### 1.4 TypeScript：2 空格（OS-STD-204）

**OS-STD-204**：TypeScript / JavaScript 代码（airymaxos-cloudnative 前端、SDK TS 包）使用 2 空格缩进，由 `.prettierrc` 强制。前端社区惯例固化，与 Prettier 默认值一致。

### 1.5 Kconfig 与文档例外

**OS-KER-002**：仅在注释、文档、Kconfig 中允许混用 Tab 与空格；Kconfig 中 `config` 下的字段以 1 个 Tab 缩进，`help` 文本以 Tab + 2 空格缩进。

```kconfig
config AIRYMAXOS_IPC
	bool "agentrt-liunx AgentsIPC subsystem"
	depends on NET
	help
	  Enable the agentrt-liunx AgentsIPC subsystem (128B fixed header).
	  Required by all agent workloads.
```

### 1.6 行尾禁止空白（OS-KER-003）

**OS-KER-003**：任何源代码行尾禁止 Tab 或空格序列。Git 在 `git diff` 中会以红色背景提示；CI 检查 `git diff --check` 任何违规即拒绝合并。

---

## 2. 行长度限制

### 2.1 C 首选 80 列（OS-KER-004）

**OS-KER-004**：C 内核代码单行首选 ≤80 列；超过 80 列的语句应拆分为合理块，除非"超过 80 列显著提升可读性且不隐藏信息"。

### 2.2 用户可见字符串禁止断行（OS-KER-005）

**OS-KER-005**：用户可见字符串（`printk` / `pr_err` / `dev_warn` 等消息）**禁止断行**。断行会破坏 `grep` 检索——线上故障排查时靠 `dmesg | grep` 定位是基本操作。

```c
/* 正确：字符串保持单行，即使超过 80 列 */
pr_err("airymaxos-cognition: dispatcher refused task %llu due to capability mismatch (have=%#llx, want=%#llx)\n",
       task->id, have_caps, want_caps);

/* 错误：字符串断行破坏 grep */
pr_err("airymaxos-cognition: dispatcher refused task %llu due to "
       "capability mismatch\n", task->id);
```

### 2.3 其他语言 100-120 列（OS-STD-205）

**OS-STD-205**：Rust / Python / TypeScript 单行 ≤100 列；TypeScript 在 JSX 模板场景放宽至 ≤120 列。各语言的 `rustfmt` / `black` / `prettier` 配置项与该限制对齐。

### 2.4 后继行应大幅短于父行（OS-KER-006）

**OS-KER-006**：被断行的长行的后继行必须显著短于父行并向右对齐；常见做法是后继行与函数开括号对齐。

```c
/* 正确：后继行与开括号对齐 */
int airymaxos_dispatcher_select(struct agentrt_task_desc *desc,
                                u32 capability_mask,
                                u64 deadline_ns)
{
	...
}
```

---

## 3. 大括号位置（K&R 风格）

K&R 大括号放置是 Linux 6.6 内核基线的工程惯例，agentrt-liunx 完全继承。其核心价值是**最小化空行占用**——在 25 行终端屏幕时代确立，至今仍是代码密度与可读性的最佳平衡。

### 3.1 非函数语句块：开括号在行末（OS-KER-007）

**OS-KER-007**：所有非函数语句块（`if` / `switch` / `for` / `while` / `do`）开括号置于控制语句行末，闭括号置于下一行行首。

```c
if (x == y) {
	do_a();
	do_b();
}
```

### 3.2 函数例外：开括号在下一行行首（OS-KER-008）

**OS-KER-008**：函数定义的开括号独占一行，置于行首。函数与语句块的不一致是 K&R 的有意设计——函数不可嵌套，独占一行让其更显眼。

```c
int airymaxos_ipc_send(struct agentrt_ipc_msg *msg)
{
	return do_send(msg);
}
```

### 3.3 do-while 与 if-else 例外

闭括号独占一行，**除非**其后紧跟同一语句的延续（`while` 终结 do 循环、`else` 续接 if）：

```c
do {
	body_of_do_loop();
} while (condition);

if (x == y) {
	..
} else if (x > y) {
	...
} else {
	....
}
```

### 3.4 单语句与多语句的大括号规则（OS-KER-009）

**OS-KER-009**：单语句不需要大括号；但**任一分支是多语句则两分支都必须加大括号**——对称性是代码可读性的根基。

```c
/* 正确：两分支都为单语句，无需大括号 */
if (condition)
	do_this();
else
	do_that();

/* 错误：if 分支多语句而 else 单语句，不对称 */
if (condition) {
	do_this();
	do_that();
} else
	otherwise();

/* 正确：if 分支多语句时 else 也加大括号 */
if (condition) {
	do_this();
	do_that();
} else {
	otherwise();
}
```

---

## 4. 空格使用

### 4.1 关键字后空格（OS-KER-010）

**OS-KER-010**：`if` / `switch` / `case` / `for` / `do` / `while` 等关键字后必须加空格。

### 4.2 例外：`sizeof` / `typeof` / `__attribute__` 不加空格（OS-KER-011）

**OS-KER-011**：`sizeof` / `typeof` / `alignof` / `__attribute__` / `defined` 视作函数式关键字，**不加空格**。

```c
/* 正确 */
s = sizeof(struct file);
ret = __builtin_expect(!!err, 0);

/* 错误 */
s = sizeof (struct file);
```

### 4.3 指针声明 `*` 贴名字不贴类型（OS-KER-012）

**OS-KER-012**：指针声明中 `*` 紧贴变量名而非类型名。这是 C 语言指针语义本质的体现——指针属于变量，不属于类型。

```c
/* 正确 */
char *airymaxos_banner;
unsigned long long memparse(char *ptr, char **retptr);
char *match_strdup(substring_t *s);

/* 错误：`*` 贴类型 */
char* airymaxos_banner;
```

### 4.4 二元/三元运算符两侧加空格，一元运算符后无空格（OS-KER-013）

**OS-KER-013**：`=  +  -  <  >  *  /  %  |  &  ^  <=  >=  ==  !=  ?  :` 两侧加空格；一元 `&  *  +  -  ~  !` 后无空格；前后缀 `++  --` 紧贴操作数；结构成员 `.` / `->` 周围无空格。

```c
/* 正确 */
a = b + c;
*p = ~x & y;
if (!err && (flags & MASK) == VAL)
	r = cond ? x : y;
s->field++;

/* 错误 */
a=b+c;
*p = ~ x&y;
if( !err&&(flags&MASK)==VAL )
```

### 4.5 括号内禁止空格（OS-KER-014）

**OS-KER-014**：括号内侧禁止空格，无论是函数参数还是表达式。

```c
/* 正确 */
s = sizeof(struct file);
airymaxos_ipc_send(msg, flags);

/* 错误 */
s = sizeof( struct file );
airymaxos_ipc_send( msg, flags );
```

### 4.6 行尾禁止空白（重申）

`OS-KER-003` 在第 1 章已声明。此处重申：行尾空白是空格滥用的最坏形态——它既不传递信息，又会在 `git diff --check` 触发噪声。`MaxEmptyLinesToKeep: 1`（见 §9）即为其工具层防线。

---

## 5. switch/case 格式

### 5.1 case 标签与 switch 对齐（OS-KER-015）

**OS-KER-015**：`case` 标签与 `switch` 对齐到同一列，**禁止双缩进**——`case` 本身就是 `switch` 的下属，缩进一层即可表达语义。

```c
switch (suffix) {
case 'G':
case 'g':
	mem <<= 30;
	break;
case 'K':
case 'k':
	mem <<= 10;
	fallthrough;
default:
	break;
}
```

### 5.2 fallthrough 必须显式标注（OS-KER-016）

**OS-KER-016**：`switch` 中故意 fallthrough 必须 `fallthrough;` 显式标注。`-Wimplicit-fallthrough` 警告会捕捉遗漏。

---

## 6. 空行使用

### 6.1 函数之间用 1 个空行分隔（OS-KER-017）

**OS-KER-017**：源文件中函数之间恰好用 1 个空行分隔；`EXPORT_SYMBOL` 紧跟在函数闭括号行之后。

```c
int system_is_up(void)
{
	return system_state == SYSTEM_RUNNING;
}
EXPORT_SYMBOL(system_is_up);

int system_is_down(void)
{
	return system_state != SYSTEM_RUNNING;
}
```

### 6.2 禁止函数内多余空行（OS-KER-018）

**OS-KER-018**：函数体内禁止使用空行分隔"逻辑段落"——逻辑段落应当被抽出为独立 helper（详见 03 代码风格 §3）。函数内空行只允许出现在变量声明区与可执行语句区之间。

### 6.3 `MaxEmptyLinesToKeep: 1`（OS-STD-206）

**OS-STD-206**：连续空行最多保留 1 行；`.clang-format` 的 `MaxEmptyLinesToKeep: 1` 配置项强制执行；CI 会对超过 1 行的连续空行拒绝合并。

---

## 7. 多语句同行禁令

### 7.1 禁止单行多语句（OS-KER-019）

**OS-KER-019**：禁止单行放置多条语句。该禁令源自一个朴素判断：把多语句塞在一行，等于在向读者隐藏什么。

```c
/* 错误：单行多语句，且无大括号 */
if (condition) do_this;

/* 正确 */
if (condition)
	do_this();
```

### 7.2 禁止用逗号逃避大括号（OS-KER-020）

**OS-KER-020**：禁止用逗号运算符把多条语句塞进 `if` 单语句体，逃避大括号约束。

```c
/* 错误：逗号逃避大括号 */
if (condition)
	do_this(), do_that();

/* 正确 */
if (condition) {
	do_this();
	do_that();
}
```

### 7.3 禁止单行多赋值（OS-KER-021）

**OS-KER-021**：禁止一行内多个赋值；每个赋值独占一行。agentrt-liunx 内核代码风格简单——避免技巧性表达。

```c
/* 错误 */
a = 1; b = 2; c = 3;

/* 正确 */
a = 1;
b = 2;
c = 3;
```

---

## 8. 配置即代码（强制）

### 8.1 `.clang-format`（C/C++，OS-STD-207）

**OS-STD-207**：airymaxos-kernel / airymaxos-security / airymaxos-services 仓库根目录必须维护 `.clang-format`（基于 Linux 6.6 内核基线 689 行版本 + ForEachMacros 列表 + agentrt-liunx 专属宏扩展）。

### 8.2 `.rustfmt.toml`（Rust，OS-STD-208）

**OS-STD-208**：airymaxos-cognition / airymaxos-cloudnative / 内核 Rust 模块仓库根目录必须维护 `.rustfmt.toml`，edition 2021，`hard_tabs = false`（4 空格），`max_width = 100`。

### 8.3 `.pyproject.toml`（Python，OS-STD-209）

**OS-STD-209**：Python 仓库（SDK / DevStation）必须维护 `.pyproject.toml`，配置 `black` `line-length = 100`、`isort` `profile = "black"`、`mypy` `strict = true`。

### 8.4 `.prettierrc`（TypeScript，OS-STD-210）

**OS-STD-210**：TypeScript 仓库必须维护 `.prettierrc`，`printWidth = 120`，`tabWidth = 2`，`singleQuote = true`，`trailingComma = "all"`。

### 8.5 CI 强制 `make format-check`（OS-STD-211）

**OS-STD-211**：CI 流水线第 4 层（CI 门禁，详见 06 卷 §4）必须执行 `make format-check`；任何格式违规直接拒绝 PR 合并，无人工豁免通道。修复方式唯一：本地运行 `make format` 自动修复后再提交。

---

## 9. clang-format 关键配置项

agentrt-liunx `.clang-format` 在 Linux 6.6 内核基线 689 行配置基础上扩展。下表列出关键项及其与第 1-7 章规则的对应关系：

| 配置项 | 值 | 对应规则 | 说明 |
|--------|-----|----------|------|
| `Language` | `Cpp` | — | 主语言 |
| `BasedOnStyle` | `LLVM` | — | 基线 |
| `ColumnLimit` | `80` | OS-KER-004 | C 行宽首选 80 |
| `IndentWidth` | `8` | OS-KER-001 | 缩进宽度 |
| `TabWidth` | `8` | OS-KER-001 | Tab 显示宽度 |
| `UseTab` | `Always` | OS-KER-001 | 强制 Tab |
| `ContinuationIndentWidth` | `8` | OS-KER-006 | 后继行缩进 |
| `BreakBeforeBraces` | `Custom` | OS-KER-007/008 | K&R 风格 |
| `BraceWrapping.AfterFunction` | `true` | OS-KER-008 | 函数开括号独占行 |
| `BraceWrapping.AfterControlStatement` | `false` | OS-KER-007 | 控制语句开括号行末 |
| `BraceWrapping.BeforeElse` | `false` | OS-KER-007 | else 紧贴闭括号 |
| `PointerAlignment` | `Right` | OS-KER-012 | `*` 贴名字 |
| `SpaceAfterCStyleCast` | `false` | OS-KER-014 | 括号内禁空格 |
| `SpaceBeforeParens` | `ControlStatements` | OS-KER-010/011 | 关键字后空格、`sizeof` 无 |
| `AlignAfterOpenBracket` | `Align` | OS-KER-006 | 后继行对齐 |
| `AllowShortIfStatementsOnASingleLine` | `false` | OS-KER-019 | 禁单行多语句 |
| `AllowShortBlocksOnASingleLine` | `false` | OS-KER-019 | 同上 |
| `AllowShortLoopsOnASingleLine` | `false` | OS-KER-019 | 同上 |
| `MaxEmptyLinesToKeep` | `1` | OS-STD-206 | 连续空行最多 1 |
| `BreakStringLiterals` | `false` | OS-KER-005 | 禁止断字符串 |
| `ForEachMacros` | 560 项 | — | 内核 for_each 宏列表，agentrt-liunx 扩展 `agentrt_for_each_*` |

---

## 10. 五维原则映射

代码格式卷是 Airymax 五维正交 24 原则在视觉层面的最小切片——大多数原则在格式层不直接体现，但下列原则在此卷有具体落地点：

| 原则 | 落地点 | 对应规则 |
|------|--------|----------|
| **A-1 极简主义** | K&R 大括号最小化空行；连续空行最多 1 | OS-KER-007、OS-KER-017、OS-STD-206 |
| **A-2 细节关注** | 行尾禁止空白；指针声明贴名；括号内禁空格 | OS-KER-003、OS-KER-012、OS-KER-014 |
| **A-4 完美主义** | 配置即代码——格式规则必须可被工具 100% 强制 | OS-STD-201、OS-STD-211 |
| **E-4 跨平台一致性** | 多语言各自规范（C/Rust/Python/TS），无强制跨语言一致 | OS-STD-202~205 |
| **E-7 文档即代码** | `.clang-format` 等配置文件纳入版本控制，与代码同源审查 | OS-STD-207~210 |
| **K-2 接口契约化** | 用户可见字符串禁止断行——保证 grep 契约可执行 | OS-KER-005 |

---

## 11. 相关文档

- [工程标准 README](README.md)：工程标准规范主索引与总纲
- [01 代码规范](01-coding-standards.md)：命名、函数、注释、类型、错误处理
- [03 代码风格](03-code-style.md)：模块化、抽象层次、防御性、性能、工具链使用风格
- [06 工具链与自动化](06-toolchain-and-automation.md)：7 层验证中格式检查所在的 Layer 3 / Layer 4
- [10-architecture/02-five-dimensional-principles.md](../10-architecture/02-five-dimensional-principles.md)：五维正交 24 原则完整定义
- Linux 6.6 内核基线 `Documentation/process/coding-style.rst` 与 `.clang-format`（689 行 + 560 ForEachMacros）为同源参考

---

## 12. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-06 | 初始版本（含 §1-§10 全部规则 + clang-format 配置表 + 五维映射） |
| 1.0.1 | TBD | 与代码实现同步验证；ForEachMacros 扩展至包含 agentrt-liunx 专属宏 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
