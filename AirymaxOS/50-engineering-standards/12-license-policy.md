# Airymax 许可证策略规范（SSoT）

> **文档定位**: Airymax 工程标准 / 许可证治理
> **版本**: v1.0
> **最后更新**: 2026-07-21
> **SPDX-License-Identifier**: CC-BY-NC-4.0
> **状态**: 正式生效（取代历史"三许可证体系"和"双许可证体系"的所有遗留描述）

---

## 一、目的与范围

本文档是 Airymax 项目的**许可证策略单一真相来源**（Single Source of Truth, SSoT）。
所有子仓库、子目录、源代码文件、文档的许可证选择必须以本文档为准。

**适用范围**:
- airymaxhub 伞仓及其下所有子仓库（29 仓拆分后的全部仓库）
- 所有源代码文件（C/C++/Header/Python/Rust/Go/TypeScript/JavaScript/Shell/Bash）
- 所有文档文件（Markdown/HTML/PDF）
- 所有构建配置文件（CMake/Cargo/package.json/pyproject.toml/vcpkg.json）
- 所有 manifest/schema/contract JSON 文件

**历史背景**:
- 2026-07-03: 提出 Open Core + Fair Source 分层策略
- 2026-07-04 上午: 建立"三许可证体系"（AGPL+Apache+Proprietary）
- 2026-07-04 下午: 简化为"双许可证体系"（AGPL+Apache）
- 2026-07-04 晚间: MemoryRovol 改为独立 EULA（选项 C）
- 2026-07-18: 本文确立最终的 5 层许可证分层策略（Open Core 模式回归）

---

## 二、5 层许可证分层策略

Airymax 采用 **Open Core + Fair Source** 分层许可模式，按内容性质将项目分为 5 层：

```
┌────────────────────────────────────────────────────────────────────┐
│ L1 开放标准层（CC-BY-4.0）                                          │  最自由
│   完全自由，允许任何第三方商业或非商业实现                            │
├────────────────────────────────────────────────────────────────────┤
│ L2 文档层（CC-BY-NC-4.0）                                           │
│   自由分发与修改，禁止商业使用                                        │
├────────────────────────────────────────────────────────────────────┤
│ L3 开源核心层（AGPL-3.0-or-later OR Apache-2.0 双许可证）           │
│   Open Core 开源核心，双许可证由用户选择                              │
├────────────────────────────────────────────────────────────────────┤
│ L4 内核层（GPL-2.0-only）                                           │
│   与 Linux 内核兼容，强制 GPL v2                                    │
├────────────────────────────────────────────────────────────────────┤
│ L5 增值闭源层（SPHARX Ltd. 商业 EULA）                              │  最严格
│   独立闭源商业产品，需要商业许可证                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 三、各层详细规范

### 3.1 L1 开放标准层

| 属性 | 值 |
|------|-----|
| **许可证** | Creative Commons Attribution 4.0 International |
| **SPDX 标识符** | `CC-BY-4.0` |
| **覆盖路径** | `docs/OpenStandards/` |
| **LICENSE 文件** | `docs/OpenStandards/LICENSE` |
| **目的** | 让任何第三方（商业或非商业）可以自由实现 Airymax 标准 |

**权利与义务**:
- 允许商业使用、修改、分发、衍生作品
- 唯一要求：保留版权声明并标注 SPHARX Ltd. 为原作者
- 不强制衍生作品开源

**为什么选择 CC-BY-4.0**:
- 行业标准实践（W3C、IETF RFC、Linux Standard Base）
- 不受 copyleft 限制，鼓励最广泛采纳
- 与"成为行业标准"的战略目标一致

**覆盖内容**:
- `docs/OpenStandards/01-airy-agent-runtime-standard.md`
- `docs/OpenStandards/02-airy-ipc-protocol-standard.md`
- `docs/OpenStandards/03-airy-security-model-standard.md`
- `docs/OpenStandards/04-airy-memory-rovol-standard.md`
- `docs/OpenStandards/05-airy-cognition-loop-standard.md`
- `docs/OpenStandards/06-airy-scheduling-standard.md`
- `docs/OpenStandards/07-airy-syscall-standard.md`
- `docs/OpenStandards/README.md`

---

### 3.2 L2 文档层

| 属性 | 值 |
|------|-----|
| **许可证** | Creative Commons Attribution-NonCommercial 4.0 International |
| **SPDX 标识符** | `CC-BY-NC-4.0` |
| **覆盖路径** | `docs/`（除 `docs/OpenStandards/`） |
| **LICENSE 文件** | `docs/LICENSE` |
| **目的** | 保护 SPHARX Ltd. 工程文档版权，允许非商业自由使用 |

**权利与义务**:
- 允许非商业使用、修改、分发、衍生作品
- 禁止商业使用（需要单独商业许可证）
- 保留版权声明并标注 SPHARX Ltd. 为原作者

**覆盖内容**:
- `docs/AirymaxOS/`（工程标准、设计文档）
- `docs/AirymaxRT/`（运行时文档）
- `docs/Publications/`（论文、白皮书）
- `docs/` 下其他非 OpenStandards 文档

**商业使用申请**:
- 联系 SPHARX Ltd. 获取商业文档许可证
- 邮箱: licensing@spharx.example.com（占位符，请替换为真实邮箱）

---

### 3.3 L3 开源核心层

| 属性 | 值 |
|------|-----|
| **许可证** | GNU Affero General Public License v3 or later **OR** Apache License 2.0 |
| **SPDX 标识符** | `AGPL-3.0-or-later OR Apache-2.0` |
| **LICENSE 文件** | 各子仓根目录的 LICENSE 文件（924 行双许可证全文） |
| **目的** | Open Core 开源核心，由用户选择 AGPL 或 Apache |

**许可证选择指南**:

| 使用场景 | 推荐选择 | 理由 |
|----------|----------|------|
| 网络服务/SaaS 部署 | AGPL v3 | 触发网络服务条款，强制衍生作品开源 |
| 商业闭源集成 | Apache 2.0 | 允许闭源衍生作品，专利保护完善 |
| 个人学习与研究 | 任一 | 两者均允许 |
| 二次开发开源项目 | AGPL v3 | copyleft 保护衍生作品也开源 |
| 企业内部工具 | Apache 2.0 | 允许企业内部闭源使用 |

**覆盖路径（26 个子仓）**:

| 仓库组 | 子仓 |
|--------|------|
| agentrt | atoms, commons, cupolas, daemons, gateway, heapstore, protocols, (agentrt 根) |
| agentrt-linux (除 kernel) | cloudnative, cognition, memory, security, services, system, tests-linux, (agentrt-linux 根) |
| sdk | cli, sdk-go, sdk-python, sdk-rust, sdk-typescript, tui, (sdk 根) |
| ecosystem | examples, manager, openlab, prompts, skills, (ecosystem 根) |
| products (除 memoryrovol) | docker, desktop, (products 根) |
| 其他 | devtools |

**源代码 SPDX 标签格式**:

```c
/* SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0 */
```

```python
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
```

```rust
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
```

```go
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
```

```typescript
// SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
```

**特殊情况：agentrt-linux 内核模块混合代码**

由于 agentrt-linux 其他子仓（services/security/cognition 等）可能混合包含：
1. **用户态代码**：使用双许可证 `AGPL-3.0-or-later OR Apache-2.0`
2. **内核模块代码**（`.ko` 文件或包含 `#include <linux/module.h>`）：必须使用 `GPL-2.0-only`

逐文件检查规则：
- 若源代码包含 `#include <linux/module.h>`、`MODULE_LICENSE("GPL")`、`module_init()`、`module_exit()` 等内核模块标识 → 使用 `GPL-2.0-only`
- 否则 → 使用双许可证

---

### 3.4 L4 内核层

| 属性 | 值 |
|------|-----|
| **许可证** | GNU General Public License v2 only |
| **SPDX 标识符** | `GPL-2.0-only` |
| **覆盖路径** | `agentrt-linux/kernel/` |
| **LICENSE 文件** | `agentrt-linux/kernel/LICENSE` |
| **目的** | 与 Linux 内核（6.6 LTS / 7.1）许可证保持兼容 |

**为什么必须使用 GPL-2.0-only**:

1. **Linux 内核许可证**：Linux 内核使用 GPL-2.0-only（不含 "or later"）
2. **派生作品强制**：任何派生自 Linux 内核或与内核链接的代码必须使用 GPL-2.0-only
3. **不兼容许可证**：
   - AGPL v3 基于 GPL v3，与 GPL v2 **不兼容**
   - Apache 2.0 包含专利条款，GPL v2 **不接受**

**Linux Syscall Exception（库例外）**:

作为特殊例外，本内核子系统的版权持有者授予许可：
- 将代码与标准 C 库和用户态系统调用接口链接
- 分发此类组合

此例外适用于本目录下的所有内核模块、头文件和源文件。

**源代码 SPDX 标签格式**:

```c
/* SPDX-License-Identifier: GPL-2.0-only */
```

**上游归属声明**:
- Linux 内核版权归 Linus Torvalds 及全球贡献者所有（1991-2024）
- 上游 Linux 内核的完整版权和许可证声明保留在源代码树中

---

### 3.5 L5 增值闭源层

| 属性 | 值 |
|------|-----|
| **许可证** | SPHARX Ltd. End User License Agreement for MemoryRovol |
| **SPDX 标识符** | `LicenseRef-SPHARX-MemoryRovol-EULA-1.0` |
| **覆盖路径** | `products/memoryrovol/` |
| **LICENSE 文件** | `products/memoryrovol/LICENSE`（224 行 EULA 全文） |
| **目的** | 商业增值闭源产品，Open Core 增值模块 |

**授权层级**:

| 层级 | 用途 | 价格策略 |
|------|------|----------|
| Trial | 30 天试用 | 免费 |
| Pro | 单实例生产部署 | 付费 |
| Enterprise | 多实例部署 | 付费 |
| Enterprise+ | 无限部署 + 优先支持 | 付费 |

**重要说明**:
- MemoryRovol **不属于** OpenAirymax 双许可证体系
- 使用 MemoryRovol 需要单独的商业许可证密钥
- MemoryRovol 源代码 SPDX 标签使用 `LicenseRef-SPHARX-MemoryRovol-EULA-1.0`

---

## 四、LICENSE 文件位置清单

### 4.1 完整 LICENSE 文件清单（37 个）

| # | 路径 | 层 | SPDX 标识符 |
|---|------|----|----|
| 1 | `airymaxhub/LICENSE` | 伞仓 | （策略概述文件） |
| 2 | `docs/OpenStandards/LICENSE` | L1 | `CC-BY-4.0` |
| 3 | `docs/LICENSE` | L2 | `CC-BY-NC-4.0` |
| 4 | `agentrt/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 5 | `agentrt/atoms/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 6 | `agentrt/commons/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 7 | `agentrt/cupolas/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 8 | `agentrt/daemons/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 9 | `agentrt/gateway/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 10 | `agentrt/heapstore/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 11 | `agentrt/protocols/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 12 | `sdk/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 13 | `sdk/cli/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 14 | `sdk/sdk-go/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 15 | `sdk/sdk-python/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 16 | `sdk/sdk-rust/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 17 | `sdk/sdk-typescript/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 18 | `sdk/tui/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 19 | `ecosystem/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 20 | `ecosystem/examples/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 21 | `ecosystem/manager/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 22 | `ecosystem/openlab/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 23 | `ecosystem/prompts/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 24 | `ecosystem/skills/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 25 | `products/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 26 | `products/docker/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 27 | `products/desktop/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 28 | `agentrt-linux/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 29 | `agentrt-linux/cloudnative/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 30 | `agentrt-linux/cognition/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 31 | `agentrt-linux/memory/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 32 | `agentrt-linux/security/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 33 | `agentrt-linux/services/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 34 | `agentrt-linux/system/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 35 | `agentrt-linux/tests-linux/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 36 | `devtools/LICENSE` | L3 | `AGPL-3.0-or-later OR Apache-2.0` |
| 37 | `agentrt-linux/kernel/LICENSE` | **L4** | `GPL-2.0-only` |
| 38 | `products/memoryrovol/LICENSE` | **L5** | `LicenseRef-SPHARX-MemoryRovol-EULA-1.0` |

---

## 五、源代码 SPDX 标签要求

### 5.1 强制要求

**所有源代码文件**必须在文件首行添加 SPDX-License-Identifier 标签。

### 5.2 标签格式

| 语言 | 格式 |
|------|------|
| C / C++ / Header | `/* SPDX-License-Identifier: <SPDX_ID> */` |
| Python / Shell / Bash | `# SPDX-License-Identifier: <SPDX_ID>` |
| Rust / Go | `// SPDX-License-Identifier: <SPDX_ID>` |
| TypeScript / JavaScript | `// SPDX-License-Identifier: <SPDX_ID>` |

### 5.3 SPDX 标识符选择

| 文件位置 | SPDX 标识符 |
|----------|-------------|
| `docs/OpenStandards/` 下的源代码（若有） | `CC-BY-4.0` |
| `docs/` 下的源代码（若有） | `CC-BY-NC-4.0` |
| `agentrt/`, `sdk/`, `ecosystem/`, `products/{docker,desktop}/`, `devtools/` 下 | `AGPL-3.0-or-later OR Apache-2.0` |
| `agentrt-linux/kernel/` 下 | `GPL-2.0-only` |
| `agentrt-linux/{其他子仓}` 用户态代码 | `AGPL-3.0-or-later OR Apache-2.0` |
| `agentrt-linux/{其他子仓}` 内核模块代码（含 `#include <linux/module.h>`） | `GPL-2.0-only` |
| `products/memoryrovol/` 下 | `LicenseRef-SPHARX-MemoryRovol-EULA-1.0` |

### 5.4 检查命令

```bash
# 检查所有源代码文件是否包含 SPDX 标签
find . -type f \( -name "*.c" -o -name "*.h" -o -name "*.py" -o -name "*.rs" \
  -o -name "*.go" -o -name "*.ts" -o -name "*.js" \) \
  -not -path "*/node_modules/*" -not -path "*/target/*" | while IFS= read -r f; do
  grep -q "SPDX-License-Identifier" "$f" || echo "MISSING: $f"
done
```

---

## 六、构建配置文件许可证要求

### 6.1 Cargo.toml（Rust）

```toml
license = "AGPL-3.0-or-later OR Apache-2.0"
authors = ["SPHARX Ltd."]
```

### 6.2 package.json（Node.js）

```json
{
  "license": "AGPL-3.0-or-later OR Apache-2.0",
  "author": "SPHARX Ltd."
}
```

### 6.3 pyproject.toml（Python）

```toml
[project]
license = "AGPL-3.0-or-later OR Apache-2.0"
authors = [{name = "SPHARX Ltd."}]
classifiers = [
  "License :: OSI Approved :: GNU Affero General Public License v3 or later (AGPLv3+)",
  "License :: OSI Approved :: Apache Software License"
]
```

### 6.4 vcpkg.json（C++）

```json
{
  "license": "AGPL-3.0-or-later OR Apache-2.0"
}
```

### 6.5 CMakeLists.txt

```cmake
# 在 CMake 中声明项目许可证
project(AirymaxXXX VERSION 0.1.1
  DESCRIPTION "Airymax XXX module"
  HOMEPAGE_URL "https://github.com/SpharxTeam/Airymax"
  LANGUAGES C)
set(PROJECT_LICENSE "AGPL-3.0-or-later OR Apache-2.0")
```

---

## 七、第三方组件许可证处理

### 7.1 引用第三方组件的要求

1. **保留原始许可证**：第三方组件必须保留其原始的 LICENSE 文件和版权声明
2. **ACKNOWLEDGMENTS.md**：在主仓库的 `ACKNOWLEDGMENTS.md` 中声明使用的第三方组件及其许可证
3. **NOTICE 文件**：在有 NOTICE 文件的子仓中，补充第三方组件的版权声明

### 7.2 兼容性矩阵

| 第三方许可证 | 与 L3 双许可证兼容？ | 与 L4 GPL-2.0-only 兼容？ |
|--------------|---------------------|---------------------------|
| MIT | ✅ | ✅ |
| BSD-2-Clause | ✅ | ✅ |
| BSD-3-Clause | ✅ | ✅ |
| Apache 2.0 | ✅ | ❌（专利条款冲突） |
| LGPL-2.1 | ✅ | ✅ |
| LGPL-3.0 | ✅ | ❌ |
| MPL 2.0 | ✅ | ✅ |
| GPL-2.0-only | ✅ | ✅ |
| GPL-2.0-or-later | ✅ | ✅ |
| GPL-3.0-only | ✅ | ❌ |
| AGPL-3.0-or-later | ✅ | ❌ |
| Proprietary | ❌ | ❌ |

### 7.3 不兼容许可证处理

若必须引用与目标层不兼容的第三方许可证：
1. **隔离目录**：将第三方代码放在 `third_party/<component>/` 隔离目录
2. **保留原始 LICENSE**：完整保留第三方 LICENSE 文件
3. **静态链接警告**：在构建系统中标记，避免静态链接到主代码
4. **文档声明**：在 ACKNOWLEDGMENTS.md 中明确说明兼容性问题

---

## 八、商标使用政策

### 8.1 SPHARX Ltd. 商标清单

以下商标归 SPHARX Ltd. 所有：
- **Airymax**
- **AgentRT**
- **MicroCoreRT**
- **CoreLoopThree**
- **Thinkdual**
- **Cupolas**
- **MemoryRovol**
- **TimeSliceInfer**
- **SystemicLLM**
- **AgentsIPC**
- **AirymaxOS**

### 8.2 商标使用规则

| 使用场景 | 是否允许 | 要求 |
|----------|----------|------|
| 描述产品基于 Airymax | ✅ | 保留商标声明 |
| 推广基于 Airymax 的衍生产品 | ❌ | 需要商标授权 |
| 在兼容实现中使用 Airymax 名称 | ❌ | 需要商标授权 |
| 学术研究引用 | ✅ | 标注商标所有者 |
| 商业产品中使用 Airymax 商标 | ❌ | 需要商标授权 |

### 8.3 商标授权申请

联系邮箱: licensing@spharx.example.com（占位符，请替换为真实邮箱）

---

## 九、验收标准（LC-01 ~ LC-12）

### 9.1 LICENSE 文件验收

| 编号 | 描述 | 验证方法 |
|------|------|----------|
| **LC-01** | airymaxhub/LICENSE 存在并包含 5 层策略概述 | `ls airymaxhub/LICENSE` |
| **LC-02** | docs/OpenStandards/LICENSE 为 CC-BY-4.0 | `head -1 docs/OpenStandards/LICENSE` |
| **LC-03** | docs/LICENSE 为 CC-BY-NC-4.0 | `head -1 docs/LICENSE` |
| **LC-04** | agentrt-linux/kernel/LICENSE 为 GPL-2.0-only | `head -10 agentrt-linux/kernel/LICENSE \| grep SPDX` |
| **LC-05** | products/memoryrovol/LICENSE 为 SPHARX EULA | `head -1 products/memoryrovol/LICENSE` |
| **LC-06** | L3 层 33 个 LICENSE 文件均为双许可证 | `find ... -name LICENSE -exec grep -l "AGPL-3.0-or-later OR Apache-2.0" {} \;` |

### 9.2 源代码 SPDX 标签验收

| 编号 | 描述 | 验证方法 |
|------|------|----------|
| **LC-07** | agentrt/ 所有源代码包含 SPDX 标签 | 见 5.4 检查命令 |
| **LC-08** | agentrt-linux/kernel/ 所有源代码 SPDX 为 GPL-2.0-only | `grep -r "SPDX-License-Identifier" agentrt-linux/kernel/` |
| **LC-09** | agentrt-linux/{其他子仓} 用户态代码 SPDX 为双许可证 | 逐文件检查 |
| **LC-10** | sdk/, products/{docker,desktop}/, ecosystem/, devtools/ 源代码 SPDX 为双许可证 | 见 5.4 检查命令 |

### 9.3 文档与配置验收

| 编号 | 描述 | 验证方法 |
|------|------|----------|
| **LC-11** | 所有 Cargo.toml/package.json/pyproject.toml/vcpkg.json license 字段统一 | `grep -r "license" --include="*.toml" --include="*.json"` |
| **LC-12** | README.md 和 README_zh.md 反映 5 层许可证分层 | 阅读 README |

---

## 十、历史决策与变迁记录

| 日期 | 决策 | 状态 |
|------|------|------|
| 2026-07-03 | 提出 Open Core + Fair Source 分层策略 | 已被本文取代 |
| 2026-07-04 上午 | 建立"三许可证体系"（AGPL+Apache+Proprietary） | 已被本文取代 |
| 2026-07-04 下午 | 简化为"双许可证体系"（AGPL+Apache） | 部分保留（L3 层） |
| 2026-07-04 晚间 | MemoryRovol 改为独立 EULA（选项 C） | 保留（L5 层） |
| 2026-07-18 | 确立 5 层许可证分层策略（本文） | **正式生效** |

---

## 十一、参考文献

- [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/legalcode)
- [Creative Commons Attribution-NonCommercial 4.0 International](https://creativecommons.org/licenses/by-nc/4.0/legalcode)
- [GNU Affero General Public License v3](https://www.gnu.org/licenses/agpl-3.0.html)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [GNU General Public License v2](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
- [SPDX License List](https://spdx.org/licenses/)
- [Open Core 模式 - Wikipedia](https://en.wikipedia.org/wiki/Open-core_model)

---

## 十二、联系与反馈

**文档维护者**: SPHARX Ltd. - Airymax Team
**许可证申请**: licensing@spharx.example.com（占位符，请替换为真实邮箱）
**问题反馈**: 通过 GitHub Issues 提交

**变更记录**:
- v1.0 (2026-07-18): 初版，确立 5 层许可证分层策略

---

Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
SPDX-License-Identifier: CC-BY-NC-4.0
