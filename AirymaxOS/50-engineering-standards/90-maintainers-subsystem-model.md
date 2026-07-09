Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）MAINTAINERS 子系统维护者模式

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）的 MAINTAINERS 子系统维护者制度规范。基于 OLK-6.6 `MAINTAINERS` 文件格式（M/S/F/T/R/L/Q/B/C/P/N/K/X 等 13 类字段），制定 agentrt-liunx 8 子仓的 MAINTAINERS 模板、维护者层级、CODEOWNERS 映射。
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 0. 文档说明

### 0.1 目的与范围

本文件研究 OLK-6.6 `MAINTAINERS` 文件（734KB，约 7000 行）的字段格式与维护者制度，制定 agentrt-liunx 8 子仓的 MAINTAINERS 文件模板、维护者层级制度、以及与 GitHub/GitLab CODEOWNERS 的映射。

### 0.2 OLK-6.6 源码路径

- `MAINTAINERS`（734KB，顶层文件）
- `MAINTAINERS:1-61` —— 字段说明（"Descriptions of section entries and preferred order"）
- `MAINTAINERS:7-60` —— M/R/L/S/W/Q/B/C/P/T/F/X/N/K 字段定义
- `scripts/get_maintainer.pl` —— 维护者查询脚本（基于 MAINTAINERS）

### 0.3 术语约束

- agentrt（用户态）= 微核心（micro-core）；agentrt-liunx（OS 发行版）= 微内核（micro-kernel）
- 禁止使用特定上游发行版名称，统一用"主流 Linux 发行版"

---

## 1. OLK-6.6 MAINTAINERS 字段格式

### 1.1 字段定义（OLK-6.6 源码行号）

OLK-6.6 `MAINTAINERS` 文件采用 13 类字段标记子系统。每类字段含义如下（OLK-6.6 源码 `MAINTAINERS:7-60`）：

| 字段 | 全称 | OLK-6.6 行号 | 含义 |
|------|------|------------|------|
| `M:` | Mail | `MAINTAINERS:7` | 主维护者邮箱 `FullName <address@domain>` |
| `R:` | Reviewer | `MAINTAINERS:8` | 指定审查者（应被 CC） |
| `L:` | List | `MAINTAINERS:10` | 相关邮件列表 |
| `S:` | Status | `MAINTAINERS:11-20` | 维护状态：Supported/Maintained/Odd Fixes/Orphan/Obsolete |
| `W:` | Web | `MAINTAINERS:21` | 状态/信息网页 |
| `Q:` | Patchwork | `MAINTAINERS:22` | Patchwork 补丁追踪网址 |
| `B:` | Bug | `MAINTAINERS:23-24` | 缺陷追踪 URI |
| `C:` | Chat | `MAINTAINERS:25-26` | 聊天协议/服务器/频道（如 `irc://server/channel`） |
| `P:` | Profile | `MAINTAINERS:27-30` | 子系统提交者档案文档 URI |
| `T:` | Tree | `MAINTAINERS:31-32` | SCM 树类型与位置（git/hg/quilt/stgit/topgit） |
| `F:` | Files | `MAINTAINERS:33-38` | 文件/目录通配符（尾斜杠含子目录） |
| `X:` | Excluded | `MAINTAINERS:39-44` | 排除的文件/目录（优先于 F:） |
| `N:` | Regex | `MAINTAINERS:45-53` | 文件/目录正则模式（触发 git log 查询） |
| `K:` | Keyword | `MAINTAINERS:54-60` | 内容正则（perl 扩展），匹配补丁/文件内容 |

### 1.2 字段优先序（OLK-6.6 规定）

OLK-6.6 规定字段推荐顺序（`MAINTAINERS:4`）：

```
M: / R: / L: / S: / W: / Q: / B: / C: / P: / T: / F: / X: / N: / K:
```

### 1.3 维护状态 S: 五级（OLK-6.6 行号 `MAINTAINERS:11-20`）

| 状态 | OLK-6.6 行号 | 含义 | agentrt-liunx 适用 |
|------|------------|------|-------------------|
| `Supported` | `MAINTAINERS:12` | 有人付费维护 | 商业支持子仓 |
| `Maintained` | `MAINTAINERS:13` | 有人实际维护 | 大部分子仓 |
| `Odd Fixes` | `MAINTAINERS:14-15` | 有维护者但时间有限 | 实验性子仓 |
| `Orphan` | `MAINTAINERS:16-17` | 无当前维护者 | 不应存在 |
| `Obsolete` | `MAINTAINERS:18-20` | 旧代码，已被替代 | 废弃模块 |

### 1.4 OLK-6.6 示例（参考结构）

**OLK-6.6 源码路径**: `MAINTAINERS:71-75`（3com vortex 示例）

```
3C59X NETWORK DRIVER
M:	Steffen Klassert <klassert@kernel.org>
L:	netdev@vger.kernel.org
S:	Odd Fixes
F:	Documentation/networking/device_drivers/ethernet/3com/vortex.rst
F:	drivers/net/ethernet/3com/3c59x.c
```

**OLK-6.6 源码路径**: `MAINTAINERS:112-128`（cfg80211 多字段示例，含 T: git 树）

```
CFG80211 WIRELESS FRAMEWORK
M:	Johannes Berg <johannes@sipsolutions.de>
L:	linux-wireless@vger.kernel.org
S:	Maintained
W:	https://wireless.wiki.kernel.org/
Q:	https://patchwork.kernel.org/project/linux-wireless/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/wireless/wireless.git
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/wireless/wireless-next.git
F:	Documentation/driver-api/80211/cfg80211.rst
F:	include/net/cfg80211.h
F:	net/wireless/
```

---

## 2. agentrt-liunx 8 子仓划分

### 2.1 子仓清单

agentrt-liunx 仓库划分为 8 个子系统（subsystem），每个子系统对应一个 MAINTAINERS 条目：

| 编号 | 子系统 | 路径前缀 | 职责 | IRON-9 v2 层 |
|------|--------|---------|------|-------------|
| 1 | airymax-shared-contract | `include/airymax/` | [SC] 6 个共享契约头文件 | [SC] |
| 2 | airymax-kernel-core | `kernel/airymax/` | 内核核心（调度器、IPC、记忆） | [IND] |
| 3 | airymax-drivers | `drivers/airymax/` | 加速器/GPU/NPU 驱动 | [IND] |
| 4 | airymax-security | `security/airymax/` | Cupolas LSM 安全模块 | [IND] |
| 5 | airymax-bpf | `kernel/bpf/airymax/` | eBPF struct_ops 调度扩展 | [IND] |
| 6 | airymax-memory | `mm/airymax/` | MemoryRovol 记忆管理 | [IND] |
| 7 | airymax-cognition | `kernel/cognition/` | CoreLoopThree 认知循环 | [IND] |
| 8 | airymax-build | `build/`, `scripts/` | 构建系统、Kbuild、CI | [IND] |

### 2.2 子系统与 IRON-9 v2 层的对应

| 子系统 | [SC] 共享契约 | [SS] 语义同源 | [IND] 完全独立 |
|--------|-------------|-------------|--------------|
| airymax-shared-contract | ✓（6 个头文件） | — | — |
| airymax-kernel-core | — | ✓（调度/IPC 语义同 agentrt） | ✓（内核实现） |
| airymax-drivers | — | — | ✓ |
| airymax-security | — | ✓（Cupolas 安全语义同源） | ✓（LSM 实现） |
| airymax-bpf | — | — | ✓ |
| airymax-memory | — | ✓（MemoryRovol 语义同源） | ✓（内核实现） |
| airymax-cognition | — | ✓（CoreLoopThree 语义同源） | ✓（内核实现） |
| airymax-build | — | — | ✓ |

---

## 3. agentrt-liunx MAINTAINERS 模板

### 3.1 完整 MAINTAINERS 文件

以下是 agentrt-liunx 仓库根 `MAINTAINERS` 文件的完整模板：

```
List of maintainers
===================

agentrt-liunx (AirymaxOS) subsystem maintainers. Field format
inherited from Linux 6.6 MAINTAINERS (OLK-6.6). See
Documentation/process/maintainers.rst for field semantics.

Descriptions of section entries and preferred order
---------------------------------------------------

	M: *Mail* patches to: FullName <address@domain>
	R: Designated *Reviewer*: FullName <address@domain>
	L: *Mailing list* relevant to this area
	S: *Status*: Supported/Maintained/Odd Fixes/Orphan/Obsolete
	W: *Web-page* with status/info
	Q: *Patchwork* web based patch tracking site
	B: URI for where to file *bugs*
	C: URI for *chat* protocol, server and channel
	P: Subsystem *Profile* document URI
	T: *SCM* tree type and location (git)
	F: *Files* and directories wildcard patterns
	X: *Excluded* files and directories
	N: Files and directories *Regex* patterns
	K: *Content regex* (perl) pattern match in a patch or file

Preferred order: M: R: L: S: W: Q: B: C: P: T: F: X: N: K:

----------------------------------------------------------------------

AIRYMAX SHARED CONTRACT LAYER ([SC])
M:	SPHARX Engineering <eng@airymaxos.com>
R:	Agent Runtime Lead <agentrt-lead@airymaxos.com>
L:	airymax-dev@airymaxos.com
S:	Supported
W:	https://airymaxos.com/docs/standards/120-cross-project-code-sharing
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	include/airymax/bpf_struct_ops.h
F:	include/airymax/memory_types.h
F:	include/airymax/security_types.h
F:	include/airymax/cognition_types.h
F:	include/airymax/sched.h
F:	include/airymax/ipc.h
K:	\b(AIRYMAX_IPC_MAGIC|AIRYMAX_TASK_MAGIC|airymax_q16_t)\b
# Note: [SC] changes require bidirectional CI validation
# (agentrt + agentrt-liunx). See 120-cross-project-code-sharing.md.

AIRYMAX KERNEL CORE
M:	SPHARX Engineering <eng@airymaxos.com>
R:	Kernel Core Lead <kernel-core@airymaxos.com>
L:	airymax-dev@airymaxos.com
S:	Supported
W:	https://airymaxos.com/docs/subsystems/kernel-core
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues?label=kernel-core
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	kernel/airymax/sched/
F:	kernel/airymax/ipc/
F:	kernel/airymax/task.c
X:	kernel/airymax/ipc/legacy/	# legacy compat, separate maintainer
K:	\b(agentrt_sched_|agentrt_ipc_)\b

AIRYMAX DRIVERS (ACCELERATOR / GPU / NPU)
M:	Driver Engineering <drivers@airymaxos.com>
R:	Accelerator Lead <accel@airymaxos.com>
L:	airymax-drivers@airymaxos.com
S:	Maintained
W:	https://airymaxos.com/docs/subsystems/drivers
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues?label=drivers
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	drivers/airymax/accel/
F:	drivers/airymax/gpu/
F:	drivers/airymax/npu/
K:	\b(airymax_accel_|airymax_gpu_|airymax_npu_)\b

AIRYMAX SECURITY (CUPOLAS LSM)
M:	Security Engineering <security@airymaxos.com>
R:	Cupolas Lead <cupolas@airymaxos.com>
L:	airymax-security@airymaxos.com
S:	Supported
W:	https://airymaxos.com/docs/subsystems/security
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues?label=security
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	security/airymax/
F:	include/uapi/linux/airymax_security.h
K:	\b(airymax_security_|cupolas_)\b

AIRYMAX BPF (STRUCT_OPS SCHEDULER EXTENSION)
M:	BPF Engineering <bpf@airymaxos.com>
R:	BPF Lead <bpf-lead@airymaxos.com>
L:	airymax-bpf@airymaxos.com
S:	Maintained
W:	https://airymaxos.com/docs/subsystems/bpf
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues?label=bpf
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	kernel/bpf/airymax/
F:	tools/airymax/bpf/
K:	\b(airymax_bpf_|airymax_struct_ops)\b

AIRYMAX MEMORY (MEMORYROVOL)
M:	Memory Engineering <memory@airymaxos.com>
R:	Memory Lead <memory-lead@airymaxos.com>
L:	airymax-memory@airymaxos.com
S:	Maintained
W:	https://airymaxos.com/docs/subsystems/memory
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues?label=memory
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	mm/airymax/
F:	include/airymax/memory_types.h	# shared contract
K:	\b(airymax_mem_|memoryrovol_)\b

AIRYMAX COGNITION (CORELOOPTHREE)
M:	Cognition Engineering <cognition@airymaxos.com>
R:	CoreLoop Lead <coreloop@airymaxos.com>
L:	airymax-cognition@airymaxos.com
S:	Maintained
W:	https://airymaxos.com/docs/subsystems/cognition
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues?label=cognition
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	kernel/cognition/
F:	include/airymax/cognition_types.h	# shared contract
K:	\b(airymax_cog_|coreloopthree_|thinkdual_)\b

AIRYMAX BUILD SYSTEM AND CI
M:	Release Engineering <release@airymaxos.com>
R:	Build Lead <build@airymaxos.com>
L:	airymax-build@airymaxos.com
S:	Supported
W:	https://airymaxos.com/docs/subsystems/build
B:	https://gitlab.airymaxos.com/agentrt-liunx/-/issues?label=build
T:	git git@gitlab.airymaxos.com:agentrt-liunx.git
F:	build/
F:	scripts/airymax/
F:	.clang-format
F:	Makefile
K:	\b(KBUILD_|AIRYMAX_BUILD_)\b
```

### 3.2 字段使用约定

agentrt-liunx 对 MAINTAINERS 字段的额外约定：

| 字段 | agentrt-liunx 约定 |
|------|------------------|
| `M:` | 每个子系统**必须**至少 1 个主维护者；邮箱用团队邮箱 |
| `R:` | 每个子系统**必须**至少 1 个指定审查者；与 M: 不同人 |
| `L:` | 邮件列表使用 `airymax-<subsystem>@airymaxos.com` 命名 |
| `S:` | 生产子系统 `Supported`；实验子系统 `Maintained` 或 `Odd Fixes` |
| `B:` | 缺陷追踪必须指向 GitLab Issue，含 `?label=` 过滤 |
| `T:` | git 树统一指向 `git@gitlab.airymaxos.com:agentrt-liunx.git` |
| `F:` | 文件通配符**必须**精确到子目录或具体文件，禁止整仓 `F: .` |
| `X:` | 排除项用于"同子系统但不同维护者"的子目录（如 legacy） |
| `K:` | 内容正则用于匹配子系统专有标识符（前缀） |

---

## 4. 维护者层级制度

### 4.1 三级维护者层级

agentrt-liunx 采用三级维护者层级，对应 Linux 内核的 maintainer / subsystem maintainer / Linus 模型：

| 层级 | 角色 | 职责 | agentrt-liunx 实例 |
|------|------|------|------------------|
| L1 | 子系统维护者 | 单个子系统的代码审查与合并 | airymax-security M: |
| L2 | 集成维护者 | 跨子系统集成、冲突解决、发布分支管理 | release@airymaxos.com |
| L3 | 总维护者 | 最终合并决策、架构仲裁、[SC] 变更审批 | SPHARX Engineering |

### 4.2 [SC] 共享契约层的特殊审批流程

[SC] 层（6 个头文件）的变更必须经过**双向 CI 校验 + 总维护者审批**：

```
开发者提交 [SC] 变更 MR
   ↓
agentrt-liunx CI（本仓 CI）
   ↓ pass
agentrt CI（触发镜像 PR，验证 agentrt 编译通过）
   ↓ pass
L1 子系统维护者（airymax-shared-contract M:）审查
   ↓ approve
L3 总维护者（SPHARX Engineering）最终审批
   ↓ approve
合并至 main
```

### 4.3 维护者任命与轮换

- **任命**：新维护者由 L3 总维护者任命，需在 MAINTAINERS 文件登记邮箱与姓名
- **轮换**：维护者离任需在 MAINTAINERS 中更新 `M:` 字段，并通知 `L:` 邮件列表
- **临时维护**：`Odd Fixes` 状态的子系统可指定临时维护者，标注 `(interim)`

---

## 5. CODEOWNERS 映射

### 5.1 GitHub/GitLab CODEOWNERS 格式

GitHub 与 GitLab 使用 `.CODEOWNERS` 文件（语法相似但路径规则不同）实现自动审查者指派。agentrt-liunx 将 MAINTAINERS 的 `F:` + `M:` 映射为 CODEOWNERS。

### 5.2 CODEOWNERS 文件模板

以下是 agentrt-liunx 仓库根 `.github/CODEOWNERS`（或 GitLab `CODEOWNERS`）的模板：

```
# CODEOWNERS for agentrt-liunx (AirymaxOS)
# Auto-generated from MAINTAINERS. Do not edit manually;
# run 'make codeowners-sync' to regenerate.
# See 90-maintainers-subsystem-model.md for field semantics.

# [SC] Shared Contract Layer (bidirectional CI required)
/include/airymax/bpf_struct_ops.h    @airymaxos/eng @airymaxos/agentrt-lead
/include/airymax/memory_types.h      @airymaxos/eng @airymaxos/memory-lead
/include/airymax/security_types.h    @airymaxos/eng @airymaxos/cupolas
/include/airymax/cognition_types.h   @airymaxos/eng @airymaxos/coreloop
/include/airymax/sched.h             @airymaxos/eng @airymaxos/kernel-core
/include/airymax/ipc.h               @airymaxos/eng @airymaxos/kernel-core

# Kernel Core
/kernel/airymax/sched/               @airymaxos/kernel-core
/kernel/airymax/ipc/                @airymaxos/kernel-core
/kernel/airymax/task.c              @airymaxos/kernel-core

# Drivers
/drivers/airymax/accel/             @airymaxos/drivers
/drivers/airymax/gpu/               @airymaxos/drivers
/drivers/airymax/npu/               @airymaxos/drivers

# Security (Cupolas LSM)
/security/airymax/                  @airymaxos/security @airymaxos/cupolas

# BPF
/kernel/bpf/airymax/                @airymaxos/bpf
/tools/airymax/bpf/                 @airymaxos/bpf

# Memory (MemoryRovol)
/mm/airymax/                        @airymaxos/memory

# Cognition (CoreLoopThree)
/kernel/cognition/                  @airymaxos/cognition

# Build System and CI
/build/                             @airymaxos/release
/scripts/airymax/                  @airymaxos/release
/.clang-format                      @airymaxos/release @airymaxos/build
/Makefile                           @airymaxos/release @airymaxos/build

# Fallback: any file not matched above requires L3 approval
*                                   @airymaxos/eng
```

### 5.3 MAINTAINERS ↔ CODEOWNERS 同步机制

为避免 MAINTAINERS 与 CODEOWNERS 不一致，agentrt-liunx 提供 `make codeowners-sync` 目标自动同步：

```makefile
# 文件: Makefile（agentrt-liunx 仓库根）

.PHONY: codeowners-sync codeowners-check

## codeowners-sync: 从 MAINTAINERS 生成 CODEOWNERS
codeowners-sync:
	@echo "  SYNC    CODEOWNERS from MAINTAINERS"
	@./scripts/airymax/maintainers_to_codeowners.pl \
		MAINTAINERS > .github/CODEOWNERS

## codeowners-check: 验证 CODEOWNERS 与 MAINTAINERS 一致
codeowners-check:
	@echo "  CHECK   CODEOWNERS consistency"
	@if ! diff -q .github/CODEOWNERS \
		<(./scripts/airymax/maintainers_to_codeowners.pl MAINTAINERS) \
		> /dev/null 2>&1; then \
		echo "Error: CODEOWNERS out of sync. Run 'make codeowners-sync'."; \
		exit 1; \
	fi
	@echo "  OK      CODEOWNERS consistent"
```

CI 中 `codeowners-check` 作为强制门禁，不一致即拒绝合并。

---

## 6. get_maintainer.pl 集成

### 6.1 OLK-6.6 提供的查询脚本

OLK-6.6 提供 `scripts/get_maintainer.pl`，根据补丁修改的文件查询 MAINTAINERS，输出应 CC 的维护者邮箱列表。agentrt-liunx 直接复用此脚本：

```bash
# 查询补丁应 CC 的维护者
$ ./scripts/get_maintainer.pl --no-rolestats --no-n \
    --git --git-fallback --git-min-percent=67 \
    patches/0001-agentrt-ipc-add-batch-send.patch
```

### 6.2 git send-email 集成

提交补丁时自动调用 `get_maintainer.pl` 生成 `--to` / `--cc` 列表：

```bash
$ git format-patch -1 HEAD -o patches/
$ ./scripts/get_maintainer.pl --to --cc patches/0001-*.patch \
    | xargs git send-email
```

---

## 7. 速查表：字段速查

| 字段 | 含义 | agentrt-liunx 用法 | OLK-6.6 行号 |
|------|------|------------------|------------|
| `M:` | 主维护者邮箱 | 团队邮箱 `eng@airymaxos.com` | `MAINTAINERS:7` |
| `R:` | 审查者邮箱 | 子系统 lead 邮箱 | `MAINTAINERS:8` |
| `L:` | 邮件列表 | `airymax-<sub>@airymaxos.com` | `MAINTAINERS:10` |
| `S:` | 维护状态 | Supported/Maintained/Odd Fixes | `MAINTAINERS:11-20` |
| `B:` | 缺陷追踪 | GitLab Issue URL | `MAINTAINERS:23-24` |
| `T:` | git 树 | git@gitlab.airymaxos.com:agentrt-liunx.git | `MAINTAINERS:31-32` |
| `F:` | 文件通配符 | 精确到子目录 | `MAINTAINERS:33-38` |
| `X:` | 排除文件 | legacy 子目录 | `MAINTAINERS:39-44` |
| `K:` | 内容正则 | 子系统专有前缀 | `MAINTAINERS:54-60` |

---

## 8. 历史与变更记录

| 日期 | 版本 | 变更摘要 | 责任人 |
|------|------|---------|--------|
| 2026-07-09 | 0.1.1 | 初始创建，8 子仓 MAINTAINERS 模板 + CODEOWNERS 映射 | SPHARX 工程标准组 |

---

> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
