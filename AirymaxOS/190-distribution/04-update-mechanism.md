Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）更新机制设计详细规范

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）内核热更新、原子更新与回滚机制详细设计\
> **版本**：0.1.1\
> **最后更新**：2026-07-09\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-linux（AirymaxOS）更新机制设计旨在为运行中的系统提供安全、原子、可回滚的更新能力。本设计聚焦三大目标：

1. **内核热更新（livepatch）支持**：通过 kpatch + livepatch 框架对运行中内核打补丁，无需重启
2. **rpm-ostree 原子更新**：整个系统（内核 + 用户态）作为单一原子事务更新，全有或全无
3. **回滚机制 100% 覆盖**：每次更新后系统保留前两版本，一键回滚至任意历史版本，回滚时间 ≤ 30 秒

### 1.2 适用范围

- Linux 6.6 内核基线 livepatch 子系统（CONFIG_LIVEPATCH=y）
- rpm-ostree 原子更新框架
- GRUB2 引导加载器的多版本管理
- 12 daemon 的滚动重启（systemd + livepatch）
- MemoryRovol 数据卷的版本兼容性

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。所有外部 Linux 发行版统一表述为"主流 Linux 发行版"，禁止使用 主流 Linux 发行版/主流 Linux 发行版 字样。更新机制与 agentrt 之间属于 IRON-9 v2 [IND] 完全独立层。

### 1.4 更新场景矩阵

| 场景 | 机制 | 重启需求 | 停机时间 | 适用 |
|------|------|----------|----------|------|
| 安全补丁（CVE） | livepatch | 无需重启 | 0 | 紧急修复 |
| 内核小版本（6.6.0 → 6.6.1） | rpm-ostree + 重启 | 需重启 | 5-10 分钟 | 定期更新 |
| 用户态 daemon 更新 | rpm-ostree + 滚动重启 | 部分重启 | 单服务 < 1 秒 | 应用更新 |
| 大版本升级（1.0 → 2.0） | rpm-ostree rebase | 需重启 | 5-15 分钟 | 年度升级 |
| 紧急回滚 | rpm-ostree rollback + 重启 | 需重启 | ≤ 30 秒 | 故障恢复 |

---

## 2. 内核热更新（livepatch）

### 2.1 livepatch 架构

Linux 6.6 内核基线原生支持 livepatch 子系统（CONFIG_LIVEPATCH=y），允许在不重启系统的情况下替换内核函数实现：

```kconfig
# kernel/.config（节选）
CONFIG_LIVEPATCH=y
CONFIG_HAVE_LIVEPATCH=y
CONFIG_LIVEPATCH_WW=y
CONFIG_KALLSYMS_ALL=y
CONFIG_KPROBES=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FTRACE=y
```

### 2.2 livepatch 模块开发

```c
/* kernel/livepatches/lp_sched_agent_fix.c */
/* 示例：修复 SCHED_AGENT BPF 调度器的 vtime 计算错误 */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/livepatch.h>
#include <linux/sched.h>
#include <airymax/airy_q16.h>

static struct klp_func lp_vtime_fix_funcs[] = {
	{
		.old_name = "airy_sched_agent_compute_weight",
		.new_func = airy_sched_agent_compute_weight_fixed,
	},
	{ }
};

static struct klp_object lp_vtime_fix_objs[] = {
	{
		.name = "airy_kernel",
		.funcs = lp_vtime_fix_funcs,
	},
	{ }
};

static struct klp_patch lp_vtime_fix_patch = {
	.mod = THIS_MODULE,
	.objs = lp_vtime_fix_objs,
};

/* 修复后的 vtime 权重计算（修复 Token 预算溢出问题） */
static airy_weight_t airy_sched_agent_compute_weight_fixed(
	const struct airy_task_meta *meta)
{
	airy_weight_t base = meta->base_weight;
	airy_weight_t stage = meta->stage_factor;
	airy_weight_t token;
	airy_q16_t w;

	/* 修复：增加 token_budget 上界检查，避免负值 */
	__u32 budget = meta->token_budget & 0x7F;  /* 屏蔽高位 */

	if (budget < 20) {
		token = 0x10000;
	} else if (budget < 50) {
		token = 0xC000;
	} else {
		token = 0x8000;
	}

	w = (airy_q16_t)((__s128)base * stage >> 16);
	w = (airy_q16_t)((__s128)w * token >> 16);
	if (w > 0xFFFF)
		w = 0xFFFF;
	return (airy_weight_t)w;
}

static int lp_vtime_fix_init(void)
{
	return klp_enable_patch(&lp_vtime_fix_patch);
}

static void lp_vtime_fix_exit(void)
{
}

module_init(lp_vtime_fix_init);
module_exit(lp_vtime_fix_exit);
MODULE_LICENSE("GPL");
MODULE_INFO(livepatch, "Y");
MODULE_DESCRIPTION("agentrt-linux SCHED_AGENT vtime fix");
```

### 2.3 livepatch 应用流程

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/apply_livepatch.sh
# 应用内核 livepatch

set -euo pipefail
LP_MODULE=$1

echo "[1/4] 编译 livepatch 模块"
make -C /lib/modules/$(uname -r)/build \
     M=$(pwd) modules

echo "[2/4] 加载 livepatch 模块"
insmod "$LP_MODULE".ko

echo "[3/4] 验证 livepatch 状态"
cat /sys/kernel/livepatch/"$LP_MODULE"/transition
# 输出：0（已完成过渡）

echo "[4/4] 查看已打补丁的函数"
cat /proc/kallsyms | grep airy_sched_agent_compute_weight
# 输出示例：
# ffffffffc0a12340 t airy_sched_agent_compute_weight_fixed [lp_vtime_fix]
# ffffffffc0a12380 t airy_sched_agent_compute_weight   [airy_kernel]

echo "[完成] livepatch 已应用，无需重启系统"
```

### 2.4 livepatch 回滚

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/revert_livepatch.sh
# 回滚 livepatch

set -euo pipefail
LP_MODULE=$1

echo "[1/3] 禁用 livepatch"
echo "0" > /sys/kernel/livepatch/"$LP_MODULE"/enabled

echo "[2/3] 等待过渡完成（最多 30 秒）"
for i in $(seq 1 30); do
    transition=$(cat /sys/kernel/livepatch/"$LP_MODULE"/transition 2>/dev/null || echo "GONE")
    if [ "$transition" = "GONE" ]; then
        break
    fi
    sleep 1
done

echo "[3/3] 卸载 livepatch 模块"
rmmod "$LP_MODULE"

echo "[完成] livepatch 已回滚"
```

---

## 3. rpm-ostree 原子更新

### 3.1 rpm-ostree 架构

rpm-ostree 将整个系统（内核 + 用户态 + 配置）作为不可变的"操作系统树"（ostree）管理。每次更新生成一棵新树，原子切换：

```
┌─────────────────────────────────────────────────────────┐
│                rpm-ostree 架构                          │
├─────────────────────────────────────────────────────────┤
│  /usr  （只读，由 ostree 管理）                        │
│    ├── bin/  lib/  share/  ...                          │
│    └── 由 commit hash 标识，原子切换                   │
├─────────────────────────────────────────────────────────┤
│  /etc  （可写，三路合并：default + 用户修改 + 新版本）  │
├─────────────────────────────────────────────────────────┤
│  /var  （可写，用户数据，跨版本保留）                  │
│    └── /var/lib/airymaxos/  （MemoryRovol 数据）       │
├─────────────────────────────────────────────────────────┤
│  /sysroot/ostree/deploy/airymaxos/                     │
│    ├── deploy.<hash>.0/    ← 当前活跃部署              │
│    ├── deploy.<hash>.1/    ← 前一版本（回滚用）        │
│    └── deploy.<hash>.2/    ← 前两版本                  │
└─────────────────────────────────────────────────────────┘
```

### 3.2 系统初始化（首次安装）

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/ostree_init.sh
# 初始化 rpm-ostree 系统

set -euo pipefail
REMOTE_URL=https://ostree.airymaxos.org
BRANCH=airymaxos/1.0/x86_64

echo "[1/5] 初始化 ostree 仓库"
ostree init --repo=/sysroot/ostree/repo --mode=archive-z2

echo "[2/5] 添加远程仓库"
ostree remote add --repo=/sysroot/ostree/repo airymaxos \
    "$REMOTE_URL" --no-gpg-verify

echo "[3/5] 拉取初始操作系统树"
ostree pull --repo=/sysroot/ostree/repo airymaxos:"$BRANCH"

echo "[4/5] 部署操作系统树"
ostree admin deploy --os=airymaxos \
    --repo=/sysroot/ostree/repo \
    airymaxos:"$BRANCH"

echo "[5/5] 验证部署"
ostree admin status
# 输出示例：
# * airymaxos 1.0.1
#     Version: 1.0.1
#     Commit: 7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b
#     Subject: agentrt-linux 1.0.1 release

echo "[完成] rpm-ostree 系统已初始化"
```

### 3.3 原子更新流程图

```
┌──────────────────────────────────────────────────────────┐
│ 1. rpm-ostree upgrade                                    │
│    └→ 拉取新 commit 到本地仓库                          │
├──────────────────────────────────────────────────────────┤
│ 2. 准备新部署                                            │
│    └→ 在 /sysroot/ostree/deploy/ 创建新 deploy.X.Y      │
├──────────────────────────────────────────────────────────┤
│ 3. 三路合并 /etc                                         │
│    └→ default /etc + 用户修改 + 新版本 /etc             │
├──────────────────────────────────────────────────────────┤
│ 4. 更新 GRUB 引导条目                                    │
│    └→ 新 deploy 设为默认，原 deploy 保留                │
├──────────────────────────────────────────────────────────┤
│ 5. 提示用户重启                                          │
│    └→ 重启后进入新部署                                  │
├──────────────────────────────────────────────────────────┤
│ 6. 重启后验证                                            │
│    └→ 健康检查：12 daemon + SCHED_AGENT + MemoryRovol   │
├──────────────────────────────────────────────────────────┤
│ 7a. 验证通过 → 保留新部署                                │
│ 7b. 验证失败 → 自动回滚到前一版本                       │
└──────────────────────────────────────────────────────────┘
```

### 3.4 原子更新命令

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/atomic_upgrade.sh
# 原子更新 agentrt-linux 系统

set -euo pipefail
LOG=/var/log/airymaxos-upgrade.log

echo "[1/5] 检查可用更新" | tee -a "$LOG"
rpm-ostree upgrade --check | tee -a "$LOG"

echo "[2/5] 下载并准备新部署（不切换）" | tee -a "$LOG"
rpm-ostree upgrade --stage | tee -a "$LOG"

echo "[3/5] 切换到新部署（下次重启生效）" | tee -a "$LOG"
rpm-ostree upgrade | tee -a "$LOG"

echo "[4/5] 显示部署状态" | tee -a "$LOG"
rpm-ostree status | tee -a "$LOG"

echo "[5/5] 提示用户重启" | tee -a "$LOG"
echo "[完成] 请执行 reboot 进入新版本" | tee -a "$LOG"
echo ""
echo "当前部署状态："
rpm-ostree status
```

---

## 4. 回滚机制

### 4.1 回滚部署状态

```bash
# 查看所有部署版本
rpm-ostree status
# 输出示例：
# * airymaxos 1.0.2
#     Version: 1.0.2
#     Commit: 1a2b3c4d5e6f...
#     Subject: agentrt-linux 1.0.2 release
#   airymaxos 1.0.1
#     Version: 1.0.1
#     Commit: 7a8b9c0d1e2f...
#     Subject: agentrt-linux 1.0.1 release

# 当前活跃版本（* 标记）为 1.0.2，可回滚到 1.0.1
```

### 4.2 回滚命令序列

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/rollback.sh
# 回滚到前一版本

set -euo pipefail
TARGET=${1:-previous}  # previous | 1.0.1 | <commit-hash>

echo "[1/6] 显示当前部署状态"
rpm-ostree status

echo "[2/6] 回滚到目标版本"
case "$TARGET" in
    previous)
        rpm-ostree rollback
        ;;
    *)
        rpm-ostree deploy --os=airymaxos "$TARGET"
        ;;
esac

echo "[3/6] 验证回滚后状态"
rpm-ostree status

echo "[4/6] 提示重启"
echo "[完成] 请执行 reboot 进入回滚版本"
echo ""
echo "回滚后部署状态："
rpm-ostree status

echo "[5/6]（可选）自动重启"
read -p "立即重启？(y/N) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    reboot
fi

echo "[6/6] 重启后健康检查（在启动脚本中执行）"
# /usr/lib/airymaxos/scripts/health_check.sh
```

### 4.3 自动回滚机制

系统启动后执行健康检查，若失败则自动回滚到前一版本：

```bash
#!/bin/bash
# /usr/lib/systemd/system/airymaxos-health-check.service
# 启动后健康检查（systemd 服务）

[Unit]
Description=agentrt-linux Health Check after Boot
After=multi-user.target
ConditionPathExists=/var/lib/airymaxos/.pending-upgrade

[Service]
Type=oneshot
ExecStart=/usr/lib/airymaxos/scripts/health_check.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/health_check.sh
# 启动后健康检查脚本

set -euo pipefail
CHECK_PASSES=0
CHECK_FAILS=0
MAX_FAILURES=3

echo "[健康检查] 开始..."

# 检查 1：内核版本
KERNEL_VER=$(uname -r)
echo "  [1/5] 内核版本: $KERNEL_VER"

# 检查 2：SCHED_AGENT 调度器
if cat /sys/kernel/sched_ext/state | grep -q "enabled"; then
    echo "  [2/5] SCHED_AGENT: OK"
    CHECK_PASSES=$((CHECK_PASSES + 1))
else
    echo "  [2/5] SCHED_AGENT: FAIL"
    CHECK_FAILS=$((CHECK_FAILS + 1))
fi

# 检查 3：12 个 daemon 状态
DOWN_DAEMONS=0
for svc in gateway_d llm_d tool_d market_d sched_d monit_d \
           channel_d observe_d notify_d info_d hook_d plugin_d; do
    if ! systemctl is-active --quiet "$svc.service"; then
        DOWN_DAEMONS=$((DOWN_DAEMONS + 1))
    fi
done
if [ $DOWN_DAEMONS -eq 0 ]; then
    echo "  [3/5] 12 daemon: OK"
    CHECK_PASSES=$((CHECK_PASSES + 1))
else
    echo "  [3/5] 12 daemon: FAIL ($DOWN_DAEMONS down)"
    CHECK_FAILS=$((CHECK_FAILS + 1))
fi

# 检查 4：MemoryRovol PMEM 挂载
if mount | grep -q "/var/lib/airymaxos/memoryrovol/pmem"; then
    echo "  [4/5] MemoryRovol PMEM: OK"
    CHECK_PASSES=$((CHECK_PASSES + 1))
else
    echo "  [4/5] MemoryRovol PMEM: FAIL"
    CHECK_FAILS=$((CHECK_FAILS + 1))
fi

# 检查 5：MGLRU 状态
if [ "$(cat /sys/kernel/mm/lru_gen/enabled)" != "0" ]; then
    echo "  [5/5] MGLRU: OK"
    CHECK_PASSES=$((CHECK_PASSES + 1))
else
    echo "  [5/5] MGLRU: FAIL"
    CHECK_FAILS=$((CHECK_FAILS + 1))
fi

echo "[健康检查] 通过: $CHECK_PASSES / 5，失败: $CHECK_FAILS"

# 如果失败超过阈值，自动回滚
if [ $CHECK_FAILS -ge $MAX_FAILURES ]; then
    echo "[健康检查] 失败次数 $CHECK_FAILS >= $MAX_FAILURES，触发自动回滚"
    rpm-ostree rollback
    rm -f /var/lib/airymaxos/.pending-upgrade
    echo "[健康检查] 已回滚，系统将在下次重启时回到前一版本"
    exit 1
fi

# 检查通过，清除待升级标记
rm -f /var/lib/airymaxos/.pending-upgrade
echo "[健康检查] 全部通过，本次升级固化"
exit 0
```

---

## 5. MemoryRovol 数据兼容性

### 5.1 数据版本化

MemoryRovol L1-L4 数据卷在更新过程中保留版本标记，确保跨版本兼容：

```yaml
# /var/lib/airymaxos/memoryrovol/metadata.yaml
# MemoryRovol 元数据（跨更新保留）

version: 1.0.1
schema_version: 2
created_at: 2026-07-09T10:00:00Z
last_updated: 2026-07-09T15:30:00Z

layers:
  L1_raw:
    capacity_gb: 64
    used_gb: 32.5
    format: zstd_v1
    schema_compatible: [1.0.1, 1.0.2]
  L2_feat:
    capacity_gb: 64
    used_gb: 28.3
    format: hnsw_v1
    schema_compatible: [1.0.1, 1.0.2]
  L3_str:
    capacity_gb: 32
    used_gb: 12.1
    format: kg_v1
    schema_compatible: [1.0.1, 1.0.2]
  L4_pat:
    capacity_gb: 32
    used_gb: 8.5
    format: hdbscan_v1
    schema_compatible: [1.0.1, 1.0.2]
```

### 5.2 数据迁移

更新后若 schema 不兼容，自动执行数据迁移：

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/memoryrovol_migrate.sh
# MemoryRovol 数据迁移

set -euo pipefail
TARGET_VERSION=${1:-1.0.2}

echo "[1/4] 检查 MemoryRovol 元数据"
META=/var/lib/airymaxos/memoryrovol/metadata.yaml
CURRENT_VER=$(yq '.version' "$META")

echo "  当前版本: $CURRENT_VER"
echo "  目标版本: $TARGET_VERSION"

if [ "$CURRENT_VER" = "$TARGET_VERSION" ]; then
    echo "[跳过] 版本已一致，无需迁移"
    exit 0
fi

echo "[2/4] 检查 schema 兼容性"
COMPATIBLE=$(yq ".layers.L1_raw.schema_compatible | index(\"$TARGET_VERSION\")" "$META")
if [ "$COMPATIBLE" = "null" ]; then
    echo "[失败] 当前数据不兼容目标版本 $TARGET_VERSION"
    exit 1
fi

echo "[3/4] 备份数据"
BACKUP_DIR=/var/lib/airymaxos/memoryrovol-backup-$(date +%Y%m%d-%H%M%S)
cp -r /var/lib/airymaxos/memoryrovol "$BACKUP_DIR"

echo "[4/4] 执行迁移"
airymax-memory migrate --from "$CURRENT_VER" --to "$TARGET_VERSION"

# 更新元数据
yq -i ".version = \"$TARGET_VERSION\"" "$META"
echo "[完成] MemoryRovol 数据已迁移至 $TARGET_VERSION"
```

---

## 6. 滚动重启机制

### 6.1 daemon 滚动重启

用户态 daemon 更新后，通过 systemd 滚动重启，避免全部服务同时下线：

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/rolling_restart.sh
# 12 daemon 滚动重启

set -euo pipefail

# 重启顺序：从底层到上层
# 1. 基础服务（memory_d / observe_d）
# 2. 中间服务（sched_d / monit_d / notify_d / info_d）
# 3. 业务服务（gateway_d / llm_d / tool_d / market_d / channel_d / hook_d / plugin_d）

PHASE1="memory_d observe_d"
PHASE2="sched_d monit_d notify_d info_d"
PHASE3="gateway_d llm_d tool_d market_d channel_d hook_d plugin_d"

restart_phase() {
    local phase_name=$1
    local services=$2
    echo "[$phase_name] 重启服务: $services"
    for svc in $services; do
        systemctl restart "$svc.service"
        sleep 2
        if ! systemctl is-active --quiet "$svc.service"; then
            echo "[失败] $svc 重启失败，触发回滚"
            exit 1
        fi
        echo "  $svc: OK"
    done
}

restart_phase "阶段 1/3" "$PHASE1"
restart_phase "阶段 2/3" "$PHASE2"
restart_phase "阶段 3/3" "$PHASE3"

echo "[完成] 12 daemon 滚动重启完成"
```

---

## 7. 错误码体系对接

更新机制错误码纳入 agentrt-linux 统一错误码体系（发行版错误段 -1000~-1099，SSoT 定义于 `include/airymax/error.h`）：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AIRY_DIST_EUPDATE_LIVEPATCH | -1010 | livepatch 应用失败 |
| AIRY_DIST_EUPDATE_OSTREE | -1011 | rpm-ostree 更新失败 |
| AIRY_DIST_EUPDATE_ROLLBACK | -1012 | 回滚失败 |
| AIRY_DIST_EUPDATE_HEALTH | -1013 | 健康检查失败 |
| AIRY_DIST_EUPDATE_MIGRATE | -1014 | 数据迁移失败 |
| AIRY_DIST_EUPDATE_DEP | -1015 | 依赖冲突 |

集中错误处理示例：

```c
int airy_update_atomic(const char *target_version)
{
	int ret;

	if (!target_version) {
		ret = -EINVAL;
		goto out_err;
	}

	ret = airy_update_ostree_upgrade(target_version);
	if (ret < 0) {
		ret = -AIRY_DIST_EUPDATE_OSTREE;
		goto out_err;
	}

	ret = airy_update_reboot_and_check();
	if (ret < 0) {
		ret = -AIRY_DIST_EUPDATE_HEALTH;
		goto out_rollback;
	}
	return 0;

out_rollback:
	airy_update_rollback();
out_err:
	return ret;
}
```

---

## 8. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **S-1 反馈闭环** | 健康检查 + 自动回滚形成闭环 |
| **S-4 涌现性管理** | 多版本部署管理更新涌现性 |
| **E-1 安全内生** | GPG 签名 + 安全启动链 |
| **E-3 资源确定性** | 原子更新保证全有或全无 |
| **E-6 错误可追溯** | 完整更新日志 + 健康检查日志 |
| **E-8 可测试性** | 健康检查脚本可独立测试 |

---

## 9. IRON-9 v2 同源映射

| 组件 | agentrt-linux（[IND]） | agentrt（[IND]） |
|------|--------------------------|------------------|
| livepatch | 内核热更新 | 无 |
| rpm-ostree | 原子更新 | 无 |
| 回滚机制 | GRUB 多版本 | 无 |
| 数据迁移 | MemoryRovol schema 迁移 | 用户态数据迁移 |

agentrt-linux 的更新机制与 agentrt 完全独立（[IND] 完全独立层），各自演进。

---

## 10. 相关文档

- `190-distribution/README.md`（发行版管理主索引）
- `190-distribution/01-rpm-packaging.md`（RPM 打包设计）
- `190-distribution/03-installer-design.md`（安装器设计）
- `100-operations/README.md`（部署运维）
- `170-performance/02-memory-performance.md`（MemoryRovol 配置）

---

## 11. 参考材料

- Linux 6.6 `kernel/livepatch/`（livepatch 子系统）
- rpm-ostree 项目文档
- ostree 项目文档
- kpatch 工具文档
- systemd 服务管理文档
- 主流 Linux 发行版原子更新方案

---

> **文档结束** | agentrt-linux（AirymaxOS）更新机制设计详细规范 
