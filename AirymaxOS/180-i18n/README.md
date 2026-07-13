Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）国际化与本地化设计

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）国际化与本地化工程体系主索引\
> **版本**：0.1.1\
> **最后更新**：2026-07-13\
> **优先级**：P2（5 文档）\
> **同源映射**：agentrt 多语言错误码 + Linux 6.6 locale / gettext / iconv\
> **理论根基**：软件国际化哲学 + Airymax A-3 人文关怀 + E-7 文档即代码

> **审查状态**：Wave 2 v2 源码级深读审查完成（Phase A/B/C/D），国际化与本地化已通过审查（D7 修复涉及 `02-encoding-spec.md`：陈旧注释路径 `ipc_msg_hdr.h` → `ipc.h` 已修复）

---

## 1. 模块定位

agentrt-linux 国际化与本地化体系是面向全球开发者和用户的工程保障。它继承 Linux 发行版 30+ 年沉淀的国际化哲学（locale + gettext + iconv + Unicode），并在其上扩展智能体操作系统专属的多语言 Agent 提示词、多区域记忆卷载、多语言文档同步等。

国际化不是「翻译」的代名词，而是「文化适配」的工程体系。agentrt-linux 国际化的核心理念是：**系统界面必须支持多语言，Agent 提示词必须支持多文化，文档必须多语言同步**。这要求从设计阶段就将国际化纳入考量，而非在发布末期才进行翻译。

### 1.1 国际化分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 字符编码 | UTF-8 / UTF-16 / GB18030 | 字符集 |
| L2 | 区域设置 | locale + LC_* | 区域格式 |
| L3 | 消息国际化 | gettext + .po / .mo | 翻译 |
| L4 | 输入法 | IBus / Fcitx | 输入 |
| **L5** | **Agent 提示词** | **agentrt-linux 专属** | **多语言提示词** |
| **L6** | **记忆卷载区域** | **agentrt-linux 专属** | **多区域记忆** |

### 1.2 agentrt-linux 扩展

- **多语言 Agent 提示词**：Agent 系统提示词支持中文 / 英文 / 日文等
- **多区域记忆卷载**：记忆卷载支持区域差异化（如中文知识库 / 英文知识库）
- **多语言文档同步**：所有文档英文 / 中文双版本同步（README + README_zh）
- **国密算法合规**：中文场景下的国密算法支持

---

## 2. 核心国际化机制

### 2.1 locale 与字符编码

```bash
# 查看当前 locale
locale

# 设置中文 locale
export LANG=zh_CN.UTF-8
export LC_ALL=zh_CN.UTF-8

# 支持的 locale
locale -a | grep zh
```

### 2.2 gettext 消息国际化

```c
#include <libintl.h>
#include <locale.h>

int main() {
    setlocale(LC_ALL, "");
    bindtextdomain("airymaxos", "/usr/share/locale");
    textdomain("airymaxos");
    
    printf(gettext("agentrt-linux started\n"));
    return 0;
}
```

### 2.3 多语言文档同步

每个文档必须包含英文（README.md）和中文（README_zh.md）两个版本：

```
docs/AirymaxOS/50-engineering-standards/
├── README.md          # 英文版
├── README_zh.md       # 中文版
├── 01-coding-standards.md
├── 01-coding-standards_zh.md
└── ...
```

---

## 3. 文档索引

```
180-i18n/
├── README.md                       # 本文件
├── 01-locale-design.md             # Locale 区域设置设计 
├── 02-encoding-spec.md             # 字符编码规范与 UTF-8 处理 
├── 03-error-message-i18n.md        # 错误消息国际化 
├── 04-cjk-support.md              # CJK 中文/日文/韩文支持 
└── 05-doc-i18n.md                 # 文档国际化与双语同步 
```

### 3.1 0.1.1 版本范围

完成 README + 01-locale-design.md（Locale 区域设置 + 环境变量）+ 02-encoding-spec.md（UTF-8 处理 + 编码转换）+ 03-error-message-i18n.md（错误码注册表 SSoT + 多语言消息映射）+ 04-cjk-support.md（CJK 显示与输入支持）+ 05-doc-i18n.md（中英双语文档维护机制 + 术语表治理）。

### 3.2 1.0.1 版本范围

实施国际化工程标准，补充 Agent 提示词多语言支持与社区翻译流程。

### 3.3 D7 修复说明（Wave 2 v2 Phase D）

`02-encoding-spec.md` 中曾存在陈旧注释路径引用：L533 处将 IPC 128B 消息头定义的注释路径错误标注为 `ipc_msg_hdr.h`，实际应为 `ipc.h`。该问题在 Wave 2 v2 Phase D（D7，对应 C-A07）中已修复：

| 修复项 | 修复前 | 修复后 | 状态 |
|--------|--------|--------|------|
| 陈旧注释路径 | `ipc_msg_hdr.h` | `ipc.h` | 已修复 |

此修复与 `170-performance/03-ipc-performance.md` 中的 5 处同源陈旧路径（D7 同批修复）保持一致，确保全文档体系对 IPC 消息头物理宿主的引用统一为 `ipc.h`。

---

## 4. agentrt-linux 专属扩展

### 4.1 多语言 Agent 提示词

```python
from agentrt import CognitionClient

client = CognitionClient(language="zh-CN")
result = client.process(prompt="你好")
# 系统提示词自动适配中文场景
```

### 4.2 多区域记忆卷载

```yaml
# 记忆卷载区域配置
memoryRovol:
  regions:
    - name: zh-CN
      knowledgeBase: /data/memory/zh-CN
      language: zh-CN
    - name: en-US
      knowledgeBase: /data/memory/en-US
      language: en-US
```

### 4.3 国密算法合规

中文场景下的国密算法支持：

| 算法 | 用途 | 场景 |
|------|------|------|
| SM2 | 数字签名 | 中文合规 |
| SM3 | 密码哈希 | 中文合规 |
| SM4 | 对称加密 | 中文合规 |

### 4.4 同源 agentrt 国际化

agentrt 的多语言错误码与 agentrt-linux 同源：
- agentrt 用户态：`airy_strerror()` 多语言错误码
- agentrt-linux 内核态：`printk` + gettext 多语言内核消息
- 两端通过同源错误码保持一致

### 4.5 IRON-9 v2 同源且部分代码共享

国际化体系遵循 IRON-9 原则：
- agentrt 国际化（用户态运行时多语言）
- agentrt-linux 国际化（OS 级 + Agent 提示词）
- 两端独立演进，但通过同源错误码保持互操作

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **A-3 人文关怀** | 多语言支持全球开发者 |
| **E-7 文档即代码** | 多语言文档同步 |
| **K-2 接口契约化** | 国际化接口契约 |
| **E-8 可测试性** | 多语言测试覆盖 |
| **IRON-9 v2 同源且部分代码共享** | 与 agentrt 多语言同源 |

---

## 6. 相关文档

- `50-engineering-standards/01-coding-standards.md`（编码规范含国际化）
- `50-engineering-standards/07-maintainers-and-governance.md`（多语言社区治理）
- `110-security/README.md`（国密算法合规）
- `140-application-development/README.md`（多语言 SDK）

---

## 7. 参考材料

- Linux 6.6 locale 子系统
- gettext 项目
- Unicode 标准
- 国密算法标准（GM/T 系列）
- agentrt 多语言错误码实现

---

> **文档结束** | 5 文档
