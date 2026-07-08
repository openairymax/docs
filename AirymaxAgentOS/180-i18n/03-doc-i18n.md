Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）文档国际化设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）文档国际化工程设计文档，覆盖中英双语文档维护机制、README.md / README_zh.md 同步策略、术语表（TERMINOLOGY.md）治理、文档翻译流程
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 1. 概述

### 1.1 设计目标

agentrt-liunx（AirymaxOS）作为面向全球开发者和用户的智能体操作系统发行版，其文档体系是知识传递的核心载体。本文档定义文档国际化工程设计，目标是：

1. **中英双语强制**：所有设计文档、API 文档、用户指南必须同时维护中文与英文两个版本
2. **README.md / README_zh.md 同步**：英文为权威源（SSoT），中文为翻译镜像，二者内容必须同步
3. **术语表治理**：TERMINOLOGY.md 是术语唯一权威，所有文档必须遵循术语规范
4. **文档翻译流程化**：从源文档到翻译文档的全流程自动化，CI 校验保证同步

### 1.2 基线约束

本设计严格遵循 Linux 6.6 内核基线，并执行以下硬约束：

| 约束项 | 取值 | 说明 |
|--------|------|------|
| 权威源语言 | 英文 | README.md 为 SSoT，README_zh.md 为翻译 |
| 文档编码 | UTF-8 | 强制，与 locale 设计一致 |
| 术语权威 | TERMINOLOGY.md | 术语唯一权威，所有文档必须引用 |
| 字符串安全 | `strscpy` | 文档中代码示例的字符串操作必须用 `strscpy` |
| 文档同源 | IRON-9 v2 [SS] 语义同源层 | 文档结构与 agentrt 同源，实现独立 |

### 1.3 术语规范

本文档严格遵循术语规范：agentrt（用户态运行时）= 微核心（micro-core）；agentrt-liunx（OS 发行版）= 微内核（micro-kernel）。文档结构共享于 [SS] 语义同源层，但内容独立（agentrt-liunx 文档聚焦 OS 发行版视角）。文中不使用 openEuler/Euler 字样，文档体系对照一律表述为"主流 Linux 发行版"。

---

## 2. 双语文档维护机制

### 2.1 双语文件命名规范

agentrt-liunx 文档体系采用"英文权威 + 中文翻译"的命名规范：

```
docs/AirymaxAgentOS/
├── README.md                # 英文（权威源 SSoT）
├── README_zh.md             # 中文（翻译镜像）
├── 10-architecture/
│   ├── README.md            # 英文
│   ├── README_zh.md         # 中文
│   ├── 01-system-architecture.md
│   ├── 01-system-architecture_zh.md
│   └── ...
├── 170-performance/
│   ├── README.md
│   ├── README_zh.md
│   ├── 01-scheduling-performance.md
│   ├── 01-scheduling-performance_zh.md
│   └── ...
└── ...
```

### 2.2 命名规则

| 文件类型 | 英文命名 | 中文命名 | 权威源 |
|---------|----------|----------|--------|
| README | `README.md` | `README_zh.md` | 英文 |
| 设计文档 | `NN-name.md` | `NN-name_zh.md` | 英文 |
| API 文档 | `api.md` | `api_zh.md` | 英文 |
| 用户指南 | `guide.md` | `guide_zh.md` | 英文 |
| 术语表 | `TERMINOLOGY.md` | （单一双语版） | 双语合体 |

### 2.3 双语文档头部模板

每个双语文档必须包含统一的头部，标注语言与权威源：

英文文档头部：

```markdown
Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx (AirymaxOS) Document Title

> **Document Position**: ...
> **Version**: 0.1.1 (doc system complete) / 1.0.1 (development)
> **Last Updated**: 2026-07-09
> **Theoretical Foundation**: Linux 6.6 kernel baseline + seL4 microkernel + Airymax parallelism theory
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
> **Language**: English (SSoT)
> **Translation**: README_zh.md (Chinese mirror)
```

中文文档头部：

```markdown
Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）文档标题

> **文档定位**: ...
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
> **语言**: 中文（翻译镜像）
> **权威源**: README.md（英文 SSoT）
```

---

## 3. README.md / README_zh.md 同步策略

### 3.1 同步原则

英文 README.md 是权威源（SSoT），任何内容变更必须先发生在英文版，再同步到中文版：

```
变更流程：
  1. 开发者修改 README.md（英文）
  2. CI 检测到 README.md 变更，标记 README_zh.md 为 "outdated"
  3. 翻译者在 7 天内同步 README_zh.md
  4. CI 校验 README_zh.md 结构与 README.md 一致
  5. PR 合并时双语版本同步更新
```

### 3.2 同步校验脚本

agentrt-liunx 提供 `airymaxos-doc-sync-check` 工具校验双语文档同步：

```python
#!/usr/bin/env python3
# airymaxos-system/doc_sync_check.py [IND]
"""校验英文与中文文档的结构同步性"""

import re
import sys
from pathlib import Path

HEADER_PATTERN = re.compile(r'^> \*\*([^*]+)\*\*: (.+)$', re.MULTILINE)
SECTION_PATTERN = re.compile(r'^##+ (.+)$', re.MULTILINE)

def extract_structure(doc_path):
    """提取文档结构（标题层级 + 头部字段）"""
    text = Path(doc_path).read_text(encoding="utf-8")
    headers = HEADER_PATTERN.findall(text)
    sections = SECTION_PATTERN.findall(text)
    return {
        "headers": dict(headers),
        "sections": sections,
    }

def check_sync(en_path, zh_path):
    """校验中英文文档同步"""
    en = extract_structure(en_path)
    zh = extract_structure(zh_path)

    errors = []

    # 1. 头部字段必须一致（除 Language 和权威源标注）
    en_header_keys = set(en["headers"].keys()) - {"Language", "Translation"}
    zh_header_keys = set(zh["headers"].keys()) - {"Language", "权威源"}
    if en_header_keys != zh_header_keys:
        errors.append(f"头部字段不一致: EN={en_header_keys}, ZH={zh_header_keys}")

    # 2. 章节标题数量必须一致
    if len(en["sections"]) != len(zh["sections"]):
        errors.append(
            f"章节数不一致: EN={len(en['sections'])}, ZH={len(zh['sections'])}"
        )

    return errors

def main():
    if len(sys.argv) != 3:
        print("Usage: doc_sync_check.py <en.md> <zh.md>")
        return 1

    errors = check_sync(sys.argv[1], sys.argv[2])
    if errors:
        print("FAIL: 文档未同步")
        for e in errors:
            print(f"  - {e}")
        return 1

    print("OK: 文档已同步")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

### 3.3 CI 集成

```yaml
# .github/workflows/doc-sync.yml
name: Doc Sync Check
on:
  pull_request:
    paths:
      - 'docs/**/*.md'
jobs:
  sync-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Find changed docs
        id: changed
        run: |
          git diff --name-only origin/main HEAD \
            | grep -E '^docs/.*\.md$' \
            | grep -v '_zh\.md$' \
            > changed_en_docs.txt
      - name: Check sync for each changed doc
        run: |
          while read en_doc; do
            zh_doc="${en_doc%.md}_zh.md"
            if [ -f "$zh_doc" ]; then
              python3 airymaxos-system/doc_sync_check.py \
                "$en_doc" "$zh_doc"
            fi
          done < changed_en_docs.txt
```

### 3.4 同步延迟容忍

| 文档类型 | 同步延迟容忍 | 超期处理 |
|---------|------------|---------|
| README.md | 7 天 | PR 阻塞合并 |
| 设计文档 | 14 天 | 标记 `translation:outdated` |
| API 文档 | 3 天 | PR 阻塞合并 |
| 用户指南 | 7 天 | 标记 `translation:outdated` |

---

## 4. 术语表（TERMINOLOGY.md）治理

### 4.1 术语表角色

TERMINOLOGY.md 是 agentrt-liunx 规范体系的唯一权威术语参考。当不同文档间存在术语冲突时，以 TERMINOLOGY.md 为准。该文件采用双语合体形式（单文件包含中英对照），避免双语同步问题。

### 4.2 术语表结构

TERMINOLOGY.md 的固定结构：

```
一、编制说明
  - 目的
  - 使用规则

二、标准计算机名词（OS 领域通用术语）
  - IPC / 进程间通信
  - Daemon / 用户态服务进程
  - Sandbox / 沙箱
  - Microkernel / 微内核
  - ...

三、agentrt-liunx 特有架构术语
  - agentrt-liunx（AirymaxOS / 极境智能体操作系统）
  - MicroCoreRT / 微核心运行时
  - AgentsIPC / Agent 进程间通信
  - Cupolas / 安全穹顶
  - MemoryRovol / 记忆卷载
  - CoreLoopThree / 认知三阶段循环
  - Thinkdual / 双思考系统
  - SCHED_AGENT / Agent 调度类
  - IRON-9 v2 / 同源且部分代码共享
  - ...

四、标准化名称映射总表
  - 标准中文 | 标准英文 | 代码前缀 | IRON-9 v2 层次 | 禁止使用的旧称

五、命名规范速查
  - 函数命名
  - 结构体命名
  - 枚举命名
  - 文件/目录命名

六、附录
  - Daemon 服务完整列表
  - 协议适配层完整列表
  - 五维正交24原则编号参考
```

### 4.3 术语治理流程

术语变更须通过架构团队评审：

```
术语变更流程：
  1. 提交术语变更提案（GitHub Issue）
  2. 架构团队评审（5 个工作日内）
  3. 评审通过 → 修改 TERMINOLOGY.md
  4. CI 校验所有文档是否引用了被禁用的旧称
  5. 通知所有文档维护者更新
```

### 4.4 术语合规校验

agentrt-liunx 提供 `airymaxos-term-check` 工具校验文档术语合规：

```python
#!/usr/bin/env python3
# airymaxos-system/term_check.py [IND]
"""校验文档术语合规性"""

import re
import sys
from pathlib import Path

# 禁止使用的术语 → 标准术语
BANNED_TERMS = {
    "openEuler": "主流 Linux 发行版",
    "Euler": "主流 Linux 发行版",
    "微内核原语": "微核心原语",  # 描述 agentrt 时
    "记忆漩涡引擎": "MemoryRovol（记忆卷载）",
    "三层循环运行时": "CoreLoopThree（认知三阶段循环）",
    "三层一体架构": "CoreLoopThree（认知三阶段循环）",
    "认知双思系统": "Thinkdual（双思考系统）",
    "Multi-Agent System": "MAC（多智能体协作框架）",
    "时间切片推理": "TimeSliceInfer（分时推理框架）",
    "IRON-9": "IRON-9 v2（同源且部分代码共享）",
}

def check_terms(doc_path):
    """检查文档是否使用了禁止术语"""
    text = Path(doc_path).read_text(encoding="utf-8")
    violations = []

    for banned, standard in BANNED_TERMS.items():
        if re.search(re.escape(banned), text):
            violations.append({
                "banned": banned,
                "standard": standard,
            })

    return violations

def main():
    if len(sys.argv) < 2:
        print("Usage: term_check.py <doc.md> [doc2.md ...]")
        return 1

    has_violation = False
    for doc in sys.argv[1:]:
        violations = check_terms(doc)
        if violations:
            has_violation = True
            print(f"FAIL: {doc} 发现禁止术语")
            for v in violations:
                print(f'  - "{v["banned"]}" → 应使用 "{v["standard"]}"')

    if not has_violation:
        print("OK: 术语合规")

    return 1 if has_violation else 0

if __name__ == "__main__":
    sys.exit(main())
```

### 4.5 CI 术语校验集成

```yaml
# .github/workflows/term-check.yml
name: Terminology Check
on:
  pull_request:
    paths:
      - 'docs/**/*.md'
      - '**/*.c'
      - '**/*.h'
jobs:
  term-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check terminology in docs
        run: |
          find docs/ -name "*.md" -exec \
            python3 airymaxos-system/term_check.py {} +
      - name: Check terminology in code comments
        run: |
          find . -name "*.c" -o -name "*.h" | \
            xargs python3 airymaxos-system/term_check.py
```

---

## 5. 文档翻译流程

### 5.1 翻译流程图

```
┌──────────────────────────────────────────────────────────┐
│                  文档翻译流程                              │
└──────────────────────────────────────────────────────────┘

  ┌─────────────────┐
  │ 1. 修改英文文档  │
  │   README.md     │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 2. 提交 PR       │
  │  (含英文变更)    │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 3. CI 自动检测   │
  │  README_zh.md   │
  │  标记 outdated  │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 4. 翻译者认领    │
  │  (7 天内)       │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 5. 翻译 README_zh│
  │  遵循术语表      │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 6. CI 同步校验   │
  │  结构一致性      │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 7. CI 术语校验   │
  │  禁止术语检测    │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 8. 评审合并      │
  │  双语版本同步    │
  └─────────────────┘
```

### 5.2 翻译规范

#### 5.2.1 标题翻译

| 英文 | 中文 | 说明 |
|------|------|------|
| `# Title` | `# 标题` | 一级标题保留 |
| `## Section` | `## 章节` | 二级标题保留 |
| `### Subsection` | `### 子章节` | 三级标题保留 |
| `**Bold**` | `**粗体**` | 保留 Markdown 格式 |
| `` `code` `` | `` `代码` `` | 代码块保留原文 |

#### 5.2.2 代码块翻译

代码块中的注释须翻译，代码本身保留原文：

```c
/* 英文原文 */
int main(void)
{
    /* Initialize the scheduler */
    return sched_init();
}
```

```c
/* 中文翻译（注释翻译，代码保留） */
int main(void)
{
    /* 初始化调度器 */
    return sched_init();
}
```

#### 5.2.3 术语处理

| 情况 | 处理方式 | 示例 |
|------|---------|------|
| 有标准中文名 | 用中文名 | MicroCoreRT → 微核心运行时 |
| 无标准中文名 | 保留英文 | sched_ext → sched_ext |
| 代码标识符 | 保留原文 | `agentrt_error_t` → `agentrt_error_t` |
| 文件路径 | 保留原文 | `include/airymax/error.h` → `include/airymax/error.h` |

### 5.3 翻译质量评分

agentrt-liunx 文档翻译质量按以下维度评分：

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 术语准确 | 30% | 100% 遵循 TERMINOLOGY.md |
| 结构同步 | 25% | 章节结构完全一致 |
| 表达流畅 | 20% | 中文表达自然 |
| 代码完整 | 15% | 代码块无遗漏 |
| 格式规范 | 10% | Markdown 格式正确 |

总分 ≥ 90 分为合格，< 90 分须返工。

---

## 6. 文档国际化测试

### 6.1 测试套件

文档国际化测试位于 `airymaxos-tests/docs/`：

| 测试名 | 描述 | 通过标准 |
|--------|------|----------|
| `test_doc_bilingual` | 双语完整性 | 所有 .md 都有 _zh.md |
| `test_doc_sync` | 同步性 | 中英结构一致 |
| `test_term_compliance` | 术语合规 | 无禁止术语 |
| `test_doc_encoding` | 编码 | UTF-8 |
| `test_doc_header` | 头部 | 头部模板完整 |

### 6.2 双语完整性测试

```python
#!/usr/bin/env python3
# airymaxos-tests/docs/test_bilingual.py [IND]
"""测试所有英文文档都有中文翻译"""

from pathlib import Path
import sys

def find_untranslated_docs(docs_dir):
    """查找没有中文翻译的英文文档"""
    docs_path = Path(docs_dir)
    untranslated = []

    for en_doc in docs_path.rglob("*.md"):
        if en_doc.name.endswith("_zh.md"):
            continue  # 跳过中文文档
        if en_doc.name == "TERMINOLOGY.md":
            continue  # 术语表双语合体

        zh_doc = en_doc.with_name(en_doc.stem + "_zh.md")
        if not zh_doc.exists():
            untranslated.append(str(en_doc))

    return untranslated

def main():
    untranslated = find_untranslated_docs("docs/AirymaxAgentOS")
    if untranslated:
        print(f"FAIL: {len(untranslated)} 个文档缺少中文翻译")
        for doc in untranslated:
            print(f"  - {doc}")
        return 1

    print("OK: 所有文档都有中文翻译")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

### 6.3 编码测试

```python
#!/usr/bin/env python3
# airymaxos-tests/docs/test_encoding.py [IND]
"""测试所有文档都是 UTF-8 编码"""

from pathlib import Path
import sys

def check_encoding(docs_dir):
    bad_encoding = []
    docs_path = Path(docs_dir)

    for doc in docs_path.rglob("*.md"):
        try:
            doc.read_text(encoding="utf-8")
        except UnicodeDecodeError:
            bad_encoding.append(str(doc))

    return bad_encoding

def main():
    bad = check_encoding("docs/AirymaxAgentOS")
    if bad:
        print(f"FAIL: {len(bad)} 个文档非 UTF-8 编码")
        for doc in bad:
            print(f"  - {doc}")
        return 1

    print("OK: 所有文档均为 UTF-8 编码")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

---

## 7. 错误处理

### 7.1 翻译缺失降级

当某文档的中文翻译缺失时，系统自动显示英文原文并标记 `translation:missing`：

```markdown
> ⚠️ **翻译缺失**: 本文档暂无中文翻译，显示英文原文。
> 如需翻译，请在 [GitHub Issue](link) 提交翻译请求。
```

### 7.2 术语冲突处理

当文档中使用了与 TERMINOLOGY.md 冲突的术语时，CI 阻塞 PR 合并：

```bash
$ python3 airymaxos-system/term_check.py docs/example.md
FAIL: docs/example.md 发现禁止术语
  - "openEuler" → 应使用 "主流 Linux 发行版"
  - "微内核原语" → 应使用 "微核心原语"
```

### 7.3 错误码定义

文档 i18n 子系统错误码遵循 ErrorCodeSystem，定义于 [SC] 共享契约层：

```c
#define AGENTRT_DOC_ENOTRANSLATED  (-910)  /* 文档未翻译 */
#define AGENTRT_DOC_EOUTOFSYNC    (-911)  /* 中英文不同步 */
#define AGENTRT_DOC_ETERM         (-912)  /* 术语冲突 */
#define AGENTRT_DOC_EENCODING    (-913)  /* 非 UTF-8 编码 */
#define AGENTRT_DOC_EHEADER      (-914)  /* 头部模板缺失 */
```

### 7.4 集中错误处理

文档校验工具采用 `goto out_free_xxx` 集中错误处理：

```c
int agentrt_doc_validate(const char *doc_path)
{
    struct doc_ctx *ctx;
    int err;

    ctx = doc_alloc_ctx();
    if (!ctx)
        return -ENOMEM;

    err = doc_load(ctx, doc_path);
    if (err)
        goto out_free_ctx;

    err = doc_check_header(ctx);
    if (err)
        goto out_free_doc;

    err = doc_check_term(ctx);
    if (err)
        goto out_free_doc;

    err = doc_check_sync(ctx);
    if (err)
        goto out_free_doc;

    pr_info("agentrt-liunx: 文档 %s 校验通过\n", doc_path);
    return 0;

out_free_doc:
    doc_release(ctx);
out_free_ctx:
    kfree(ctx);
    return err;
}
```

---

## 8. 五维原则映射

| 原则 | 在文档 i18n 中的体现 |
|------|---------------------|
| **A-3 人文关怀** | 多语言文档支持全球开发者 |
| **E-7 文档即代码** | 文档版本控制 + CI 校验 |
| **E-8 可测试性** | 双语完整性 + 同步性测试 |
| **K-2 接口契约化** | 头部模板是文档契约 |
| **IRON-9 v2 [SS]** | 文档结构与 agentrt 同源 |

---

## 9. 相关文档

- `180-i18n/README.md`（国际化主索引）
- `180-i18n/01-locale-design.md`（区域设置设计）
- `180-i18n/02-error-message-i18n.md`（错误消息国际化）
- `TERMINOLOGY.md`（统一术语表）
- `50-engineering-standards/07-maintainers-and-governance.md`（多语言社区治理）
- `50-engineering-standards/README.md`（工程标准含文档规范）

---

## 10. 参考材料

- Linux 6.6 内核文档双语实践
- GNU gettext 文档翻译指南
- Markdown 国际化最佳实践
- 主流 Linux 发行版文档 i18n 实践
- agentrt 文档结构规范（[SS] 语义同源层）

---

## 11. 版本演进

| 版本 | 双语策略 | 术语治理 | CI 校验 | 翻译流程 | 备注 |
|------|---------|---------|---------|---------|------|
| 0.1.1 | 设计 | TERMINOLOGY | — | — | 仅文档 |
| 1.0.1 | 强制双语 | SSoT | 同步 + 术语 | 流程化 | 开发 |
| 1.1.0 | + 日文 | SSoT | + 翻译质量评分 | + 自动翻译辅助 | 扩展语言 |
| 2.0.0 | 全语言 | SSoT | + 实时同步 | + AI 辅助翻译 | 完整 i18n |

---

> **文档结束** | agentrt-liunx（AirymaxOS）文档国际化设计 v1.0.1
