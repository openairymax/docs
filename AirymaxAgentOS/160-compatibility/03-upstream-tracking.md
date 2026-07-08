Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 上游追踪策略

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）兼容性体系核心子文档，定义追踪 Linux 6.6 上游 stable 补丁、merge window 策略与 CVE backport 流程
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
> **同源映射**: Linux 6.6 LTS 内核维护流程（IRON-9 v2 [IND] 完全独立层，上游追踪为 agentrt-liunx 专属策略）
> **IRON-9 v2 层次**: [IND] 完全独立层（上游追踪为 agentrt-liunx 作为 OS 发行版的专属维护策略）

---

## 1. 设计目标与上游追踪原则

### 1.1 设计目标

agentrt-liunx（AirymaxOS）基于 Linux 6.6 内核基线开发，必须持续追踪上游 stable 补丁，确保安全漏洞及时修复、缺陷及时修补、硬件兼容性持续扩展。上游追踪策略达成以下工程目标：

1. **及时安全响应**：高危 CVE 在 7 天内 backport，中危 30 天，低危 90 天
2. **稳定基线锁定**：仅追踪 Linux 6.6 stable 分支，不盲目跟随 mainline
3. **merge window 纪律**：每 2 周一次 minor merge，每季度一次 major merge
4. **补丁序列化管理**：所有 backport 补丁通过 git cherry-pick + 冲突解决
5. **回归测试保障**：每次 merge 后运行完整测试套件，确保不破坏 agentrt-liunx 专属扩展

### 1.2 上游追踪原则

agentrt-liunx 遵循以下上游追踪原则：

| 原则 | 说明 |
|------|------|
| 基线锁定 | 锁定 Linux 6.6.x stable，不追随 mainline 主线 |
| 选择性 backport | 仅 backport 安全补丁与关键缺陷修复 |
| 冲突优先解决 | 与 agentrt-liunx 专属补丁冲突时，优先保留上游修复 |
| 测试前置 | 任何 backport 必须经测试套件验证方可合入 |
| 文档同步 | 每次合入更新 CHANGELOG 与安全公告 |

### 1.3 上游分支模型

```
Linux 上游仓库（kernel.org）
    |
    +-- mainline（主线，6.7+）
    |
    +-- linux-6.6.y（stable 分支，6.6.x）
         |
         +-- agentrt-liunx-6.6（agentrt-liunx 维护分支）
              |
              +-- agentrt-liunx-6.6-stable（发布分支）
```

---

## 2. Linux 6.6 stable 补丁追踪

### 2.1 stable 补丁来源

agentrt-liunx 追踪以下上游补丁来源：

| 来源 | URL/路径 | 频率 | 内容 |
|------|----------|------|------|
| Linux 6.6 stable | git.kernel.org/.../stable-queue | 每周 | 缺陷修复 |
| CVE 公告 | cve.mitre.org / NVD | 不定 | 安全漏洞 |
| Linux security | kernel.org/security | 不定 | 安全公告 |
| Greg KH 签名公告 | 邮件列表 | 每周 | stable 发布 |

### 2.2 补丁分类与优先级

| 优先级 | 补丁类型 | 响应时间 | 例子 |
|--------|----------|----------|------|
| P0 | 高危 CVE（RCE/提权） | 7 天内 | CVE-2024-xxxx |
| P1 | 中危 CVE（DoS/信息泄露） | 30 天内 | CVE-2024-xxxx |
| P2 | 低危 CVE | 90 天内 | CVE-2024-xxxx |
| P3 | 关键缺陷修复 | 下次 merge window | 内存泄漏 |
| P4 | 一般缺陷修复 | 季度 merge | 边界条件 |
| P5 | 硬件兼容性 | 季度 merge | 新设备驱动 |

### 2.3 补丁筛选流程

```bash
#!/bin/bash
# scripts/upstream/track-stable.sh
# 追踪 Linux 6.6 stable 补丁

set -e

UPSTREAM_REMOTE="https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git"
UPSTREAM_BRANCH="linux-6.6.y"
LOCAL_BRANCH="agentrt-liunx-6.6"

# 1. 同步上游 stable 分支
git fetch $UPSTREAM_REMOTE $UPSTREAM_BRANCH
git fetch $UPSTREAM_BRANCH

# 2. 获取上游新增补丁
LAST_MERGE=$(cat .upstream-last-merge)
NEW_COMMITS=$(git log --oneline $LAST_MERGE..$UPSTREAM_BRANCH)

# 3. 分类补丁
for commit in $NEW_COMMITS; do
    subject=$(git log --format=%s -1 $commit)

    # CVE 补丁
    if echo "$subject" | grep -qE "CVE-[0-9]+-[0-9]+"; then
        echo "CVE: $commit $subject" >> patches-cve.txt
    # 安全相关
    elif echo "$subject" | grep -qiE "security|vulnerab|exploit"; then
        echo "SECURITY: $commit $subject" >> patches-security.txt
    # 缺陷修复
    elif echo "$subject" | grep -qiE "fix|bug|crash|leak"; then
        echo "FIX: $commit $subject" >> patches-fix.txt
    # 其他
    else
        echo "OTHER: $commit $subject" >> patches-other.txt
    fi
done

# 4. 生成补丁评审报告
echo "=== 补丁评审报告 ==="
echo "CVE 补丁: $(wc -l < patches-cve.txt)"
echo "安全补丁: $(wc -l < patches-security.txt)"
echo "缺陷修复: $(wc -l < patches-fix.txt)"
echo "其他补丁: $(wc -l < patches-other.txt)"
```

---

## 3. merge window 策略

### 3.1 merge window 节奏

agentrt-liunx 采用两级 merge window 节奏：

| 类型 | 频率 | 范围 | 测试要求 |
|------|------|------|----------|
| minor merge | 每 2 周 | P0/P1 安全补丁 | 回归测试 + 安全测试 |
| major merge | 每季度 | P2-P5 全部补丁 | 完整测试套件 |

### 3.2 minor merge 流程

```
[1] 补丁筛选（周一）
   ├── 从 stable-queue 拉取 P0/P1 补丁
   └── 生成补丁清单
        |
        v
[2] backport（周二-周三）
   ├── git cherry-pick 每个补丁
   ├── 解决与 agentrt-liunx 专属补丁的冲突
   └── 标注冲突解决说明
        |
        v
[3] 编译验证（周三）
   ├── 编译内核 + 8 子仓
   └── 编译失败则回退
        |
        v
[4] 回归测试（周三-周四）
   ├── 运行 LTP + agentrt 专属测试
   ├── 运行安全测试（CVE 复现）
   └── 测试失败则回退
        |
        v
[5] 代码审查（周四-周五）
   ├── 内核维护者审查
   └── 通过后合入
        |
        v
[6] 发布（周五）
   ├── 更新 CHANGELOG
   ├── 发布安全公告（如适用）
   └── 更新 .upstream-last-merge
```

### 3.3 major merge 流程

```
[1] 季度补丁汇总（季度首月第 1 周）
   ├── 汇总 3 个月内的 P2-P5 补丁
   └── 生成季度补丁清单
        |
        v
[2] 批量 backport（第 2-3 周）
   ├── 按子系统分组 backport
   ├── 解决冲突
   └── 生成补丁序列
        |
        v
[3] 完整测试（第 4-6 周）
   ├── LTP 完整套件
   ├── agentrt-liunx 8 子仓测试
   ├── POSIX 兼容性测试（PCTS）
   ├── ABI 兼容性检查
   ├── 性能基准测试
   └── 长期稳定性测试（72 小时）
        |
        v
[4] 发布候选（第 7 周）
   ├── 发布 RC1 供社区测试
   ├── 收集反馈
   └── 修复发现的问题
        |
        v
[5] 正式发布（第 8 周）
   ├── 发布 stable 版本
   ├── 更新文档
   └── 发布发布公告
```

### 3.4 merge window 纪律

| 纪律 | 说明 |
|------|------|
| 冻结期 | 发布前 1 周冻结新补丁合入 |
| 回滚机制 | 任何阶段失败，git revert 回退至上一稳定点 |
| 双签制度 | 至少 2 名维护者签字方可合入 |
| 文档同步 | CHANGELOG 与代码同步更新 |

---

## 4. backport 流程

### 4.1 backport 原则

| 原则 | 说明 |
|------|------|
| 最小化修改 | 仅修改必要部分，不引入无关变更 |
| 保留 commit message | 保留上游 commit 信息 + 标注 backport |
| 冲突透明 | 冲突解决必须详细说明 |
| 测试验证 | 每个 backport 必须经测试验证 |

### 4.2 backport 标注规范

```bash
# backport 后的 commit message 格式
[BACKPORT] <原始 commit 标题>

原 commit: <原 commit hash>
原分支: linux-6.6.y
backport 至: agentrt-liunx-6.6

冲突说明:
- <文件路径>: <冲突原因>
- <解决方式>

测试:
- [x] 编译通过
- [x] LTP 相关用例通过
- [x] agentrt-liunx 专属测试通过

Signed-off-by: <backport 者>
Reviewed-by: <审查者>
```

### 4.3 冲突解决示例

```bash
# git cherry-pick 上游补丁
git cherry-pick <upstream_commit>

# 冲突发生
# CONFLICT (content): Merge conflict in kernel/sched/core.c

# 查看冲突
git diff

# 解决冲突：保留上游修复 + 保留 agentrt-liunx 专属扩展
# 例如：上游修复了 sched_core 中的内存泄漏
#       agentrt-liunx 在同一文件扩展了 SCHED_AGENT
# 解决方式：应用上游修复 + 保留 SCHED_AGENT 代码

# 标记冲突已解决
git add kernel/sched/core.c

# 继续 cherry-pick
git cherry-pick --continue

# 编辑 commit message，添加 backport 标注
```

### 4.4 自动化 backport 工具

```bash
#!/bin/bash
# scripts/upstream/auto-backport.sh
# 自动化 backport 工具

set -e

COMMIT=$1
BRANCH="agentrt-liunx-6.6"

if [ -z "$COMMIT" ]; then
    echo "Usage: $0 <commit-hash>"
    exit 1
fi

# 1. 尝试 cherry-pick
if git cherry-pick "$COMMIT"; then
    echo "Backport succeeded without conflicts"
    exit 0
fi

# 2. 冲突检测
CONFLICTS=$(git diff --name-only --diff-filter=U)
echo "Conflicts in:"
echo "$CONFLICTS"

# 3. 自动解决简单冲突（仅上下文偏移）
for file in $CONFLICTS; do
    # 检查是否仅为上下文偏移
    if git diff "$file" | grep -qE "^[+-].*AGENTRT_"; then
        echo "Agentrt-specific conflict in $file, manual resolve required"
        exit 1
    fi
done

# 4. 复杂冲突需要人工解决
echo "Complex conflicts, manual resolve required"
echo "After resolving, run: git cherry-pick --continue"
```

---

## 5. CVE 补丁 backport 完整流程

### 5.1 CVE 响应流程示例

以 CVE-2024-XXXX（假设为内核 netfilter 提权漏洞）为例，展示从上游到 agentrt-liunx 的完整 backport 流程：

```
[第 0 天] CVE 公告
   ├── NVD 发布 CVE-2024-XXXX
   ├── CVSS 评分 8.5（高危）
   ├── 影响：本地提权至 root
   └── 影响版本：Linux 6.6.x < 6.6.45
        |
        v
[第 1 天] 影响评估
   ├── 确认 agentrt-liunx 受影响
   ├── 检查上游修复补丁是否已发布
   ├── 上游修复 commit: a1b2c3d4
   └── 标记为 P0 优先级
        |
        v
[第 2-3 天] backport
   ├── git cherry-pick a1b2c3d4
   ├── 冲突检测（net/netfilter/ 目录）
   ├── 解决冲突（如有）
   │   ├── agentrt-liunx 在 netfilter 扩展了 Cupolas 钩子
   │   └── 保留上游修复 + 保留 Cupolas 钩子
   ├── 编译验证
   └── 生成 backport commit
        |
        v
[第 4-5 天] 测试验证
   ├── 编写 CVE 复现测试用例
   ├── 运行 PoC 验证漏洞已修复
   ├── 运行 LTP netfilter 测试
   ├── 运行 agentrt-liunx 网络测试
   ├── 运行 Cupolas 安全测试
   └── 确认无回归
        |
        v
[第 6 天] 代码审查
   ├── 内核维护者审查 backport
   ├── 安全官审查冲突解决
   └── 双签通过
        |
        v
[第 7 天] 发布
   ├── 发布 agentrt-liunx-6.6.45-airymax.1
   ├── 发布安全公告（AIRYMAX-SA-2024-XX）
   ├── 更新 CVE 影响文档
   ├── 通知用户升级
   └── 提交至 CVE 数据库（已修复标记）
```

### 5.2 CVE 复现测试

```c
/* airymaxos-tests/security/test_cve_2024_xxxx.c
 * CVE-2024-XXXX 复现与验证测试 */
#include <stdio.h>
#include <sys/socket.h>
#include <linux/netfilter_ipv4.h>

int test_cve_2024_xxxx_fixed(void)
{
	int sock;
	struct nf_sockaddr addr = {
		.port = 0,
		/* 构造恶意参数触发漏洞 */
	};

	sock = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
	if (sock < 0)
		return -1;

	/* 尝试触发漏洞 */
	int ret = bind(sock, (struct sockaddr *)&addr, sizeof(addr));

	/* 验证：若已修复，bind 应返回 -EINVAL */
	if (ret == -1 && errno == EINVAL) {
		printf("[PASS] CVE-2024-XXXX 已修复\n");
		close(sock);
		return 0;
	}

	/* 若漏洞存在，bind 可能成功（错误行为） */
	printf("[FAIL] CVE-2024-XXXX 仍可触发\n");
	close(sock);
	return -1;
}

int main(void)
{
	return test_cve_2024_xxxx_fixed();
}
```

### 5.3 安全公告模板

```markdown
# AIRYMAX-SA-2024-XX: Linux netfilter 提权漏洞

## 漏洞信息
- CVE ID: CVE-2024-XXXX
- CVSS 评分: 8.5（高危）
- 影响版本: agentrt-liunx < 6.6.45-airymax.1
- 修复版本: agentrt-liunx 6.6.45-airymax.1

## 漏洞描述
Linux 内核 netfilter 子系统存在本地提权漏洞...
（详见 CVE 公告）

## 影响
本地低权限用户可利用此漏洞提权至 root...

## 修复方案
升级至 agentrt-liunx 6.6.45-airymax.1 或更高版本

## 补丁来源
上游修复 commit: a1b2c3d4
backport commit: e5f6g7h8

## 致谢
感谢上游 Linux 内核安全团队...
```

---

## 6. agentrt-liunx 专属补丁保护

### 6.1 专属补丁清单

agentrt-liunx 在 Linux 6.6 基线上有大量专属补丁，需在 merge 时保护：

| 补丁类别 | 估计数量 | 冲突风险 |
|----------|----------|----------|
| SCHED_AGENT 调度类 | ~50 补丁 | 高（与 sched/ 冲突） |
| Cupolas 安全穹顶 | ~80 补丁 | 中（与 security/ 冲突） |
| MemoryRovol 记忆卷载 | ~60 补丁 | 中（与 mm/ 冲突） |
| AgentsIPC io_uring | ~40 补丁 | 高（与 io_uring/ 冲突） |
| capability 系统 | ~30 补丁 | 低 |

### 6.2 冲突预防机制

```bash
# scripts/upstream/check-conflict-risk.sh
# 预测上游补丁与 agentrt-liunx 专属补丁的冲突风险

UPSTREAM_COMMIT=$1

# 获取上游补丁修改的文件列表
CHANGED_FILES=$(git show --name-only --format= $UPSTREAM_COMMIT)

# 检查每个文件是否被 agentrt-liunx 专属补丁修改过
for file in $CHANGED_FILES; do
    # 检查该文件是否有 agentrt-liunx 专属修改
    if git log --oneline agentrt-liunx-6.6 -- $file | \
       grep -qiE "agentrt|airymax|cupolas|memoryrov|sched_agent"; then
        echo "HIGH RISK: $file (has agentrt-liunx changes)"
    fi
done
```

### 6.3 专属补丁回归测试

每次 merge 后运行 agentrt-liunx 专属测试套件：

```bash
# scripts/test/airymaxos-regression.sh
# agentrt-liunx 专属回归测试

set -e

echo "=== SCHED_AGENT 测试 ==="
./test_sched_agent --all

echo "=== Cupolas 安全测试 ==="
./test_cupolas --all

echo "=== MemoryRovol 测试 ==="
./test_memoryrovol --all

echo "=== AgentsIPC 测试 ==="
./test_agentsipc --all

echo "=== capability 系统测试 ==="
./test_capability --all

echo "=== ABI 兼容性测试 ==="
./test_abi_compat --old-version=6.6.44 --new-version=6.6.45

echo "=== POSIX 兼容性测试 ==="
./test_posix_compat --suite=PCTS

echo "=== 全部测试通过 ==="
```

---

## 7. 监控与告警

### 7.1 上游监控

```bash
# 每日检查上游 stable 发布
#!/bin/bash
# scripts/upstream/daily-check.sh

LATEST=$(curl -s https://www.kernel.org/finger_banner | \
        grep "stable:" | head -1 | awk '{print $NF}')
CURRENT=$(cat .upstream-last-version)

if [ "$LATEST" != "$CURRENT" ]; then
    echo "ALERT: New stable release: $LATEST (current: $CURRENT)"
    # 发送告警至维护团队
    mail -s "Linux 6.6 stable release $LATEST" \
         airymax-kernel-maintainers@spharx.dev << EOF
New Linux 6.6 stable release available: $LATEST
Current agentrt-liunx version: $CURRENT
Please review and plan backport.
EOF
fi
```

### 7.2 CVE 监控

```bash
# 每日检查 CVE 公告
#!/bin/bash
# scripts/upstream/cve-check.sh

# 获取 Linux 内核相关 CVE
CVE_LIST=$(curl -s "https://cve.circl.lu/api/search/linux/kernel" | \
           jq '.[] | select(.cvss > 7.0) | .id')

for cve in $CVE_LIST; do
    # 检查是否已处理
    if grep -q "$cve" .cve-processed; then
        continue
    fi

    # 评估是否影响 Linux 6.6
    if check_cve_affects_6_6 "$cve"; then
        echo "URGENT: $cve affects Linux 6.6"
        # 发送紧急告警
    fi
done
```

---

## 8. 版本号规范

### 8.1 版本号格式

agentrt-liunx 采用以下版本号格式：

```
<kernel_version>-airymax.<airymax_release>[.<patch>]

示例：
6.6.45-airymax.1          # 基于 Linux 6.6.45 的第 1 次发布
6.6.45-airymax.1.1         # 第 1 次发布的第 1 个补丁
6.6.45-airymax.2           # 基于 Linux 6.6.45 的第 2 次发布
```

### 8.2 版本号语义

| 版本段 | 含义 | 变更触发 |
|--------|------|----------|
| `6.6.45` | 上游 Linux stable 版本 | 上游 minor merge |
| `airymax.1` | agentrt-liunx 发布号 | 每次 stable 发布 |
| `.1` | 补丁号 | 安全补丁或紧急修复 |

---

## 9. 测试与验证

### 9.1 merge 验证矩阵

| 测试维度 | minor merge | major merge |
|----------|--------------|-------------|
| 编译 | 必须 | 必须 |
| LTP | 相关用例 | 完整套件 |
| agentrt 专属测试 | 必须 | 必须 |
| POSIX 兼容性 | 不要求 | 必须 |
| ABI 兼容性 | 不要求 | 必须 |
| 性能基准 | 不要求 | 必须 |
| 长期稳定性 | 不要求 | 72 小时 |

### 9.2 回滚测试

```bash
# 验证回滚机制
./test_rollback --from=6.6.45-airymax.1 --to=6.6.44-airymax.5
# 预期：成功回滚，数据完整
```

---

## 10. 相关文档

- `160-compatibility/README.md`（兼容性主索引）
- `160-compatibility/01-abi-stability.md`（ABI 稳定性设计）
- `160-compatibility/02-posix-compat.md`（POSIX 兼容性设计）
- `50-engineering-standards/05-development-process.md`（开发流程）
- `120-development-process/README.md`（稳定版维护）

---

## 11. 参考材料

- Linux 6.6 stable 仓库（git.kernel.org）
- Linux 内核安全公告（kernel.org/security）
- CVE 数据库（cve.mitre.org / NVD）
- Greg KH stable kernel 发布公告
- Linux 内核 backport 指南
- git cherry-pick 文档

---

> **文档结束** | agentrt-liunx（AirymaxOS）上游追踪策略 v0.1.1
> 遵循 IRON-9 v2 [IND] 完全独立层（上游追踪为 agentrt-liunx 专属维护策略）
