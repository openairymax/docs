Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# ABI 稳定性设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）兼容性体系核心子文档，定义 4 层接口稳定性分级、ABI 审查流程与弃用声明机制\
> **文档版本**：v1.0.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：agentrt ABI 稳定性（IRON-9 v3 [SC] 共享契约层共享 syscall 编号 + [SS] 语义同源层 SDK 层签名同源）\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **IRON-9 v3 层次**：[SC] 共享契约层（syscall 编号、错误码、消息头结构）+ [SS] 语义同源层（SDK 层签名同源）+ [IND] 完全独立层（内部实现）

---

## 1. 设计目标与 ABI 哲学

### 1.1 设计目标

ABI（Application Binary Interface）稳定性是操作系统的生命线。agentrt-linux（AirymaxOS）承诺：在 0.1.1 上编译的 Agent 应用二进制，到 1.0.1、3.0 LTS、5.0 LTS 上无需重新编译即可运行。ABI 稳定性设计达成以下工程目标：

1. **永不破坏承诺**：L1 层用户空间 ABI 一旦导出，永久支持（OS-IRON-001）
2. **分级稳定性**：4 层接口按稳定性分级，匹配不同变更频率
3. **弃用流程化**：任何弃用必须经 6 个月宽限期，确保生态迁移
4. **版本协商**：运行时版本协商机制，支持新客户端连接旧服务
5. **ABI 审查制度化**：所有 ABI 变更须经架构团队审查

### 1.2 双层稳定性哲学

agentrt-linux 继承 Linux 内核的双层稳定性哲学：

| 层次 | 稳定性承诺 | 依据 |
|------|------------|------|
| 用户空间 ABI | 永不破坏 | Linux 内核 30 年实践 |
| 内核内部 API | 可自由重构 | Linus "we don't break userspace" 但内部可重构 |

agentrt-linux 在此基础上扩展为 4 层分级，覆盖 Agent 操作系统特有的接口类型。

### 1.3 与 agentrt 同源关系

agentrt 用户态运行时的 ABI 稳定性与本设计遵循 IRON-9 v3：

| 层次 | 共享内容 | agentrt-linux | agentrt |
|------|----------|---------------|---------|
| [SC] | syscall 编号、错误码、IPC 消息头 | 完全共享 | 完全共享 |
| [SS] | SDK API 签名 | SDK 层签名同源 | SDK 层签名同源 |
| [IND] | 内部实现 | 各自独立 | 各自独立 |

---

## 2. 4 层接口稳定性分级

### 2.1 分级总览

| 层级 | 接口类型 | 稳定性 | 变更流程 | 例子 |
|------|----------|--------|----------|------|
| **L1** | 用户空间 ABI | 永久稳定 | RFC + ABI 审查 + 6 月宽限期 | syscall、ioctl、procfs |
| **L2** | AgentsIPC 协议 | 中等稳定 | 季度评审 + 兼容性测试 | 128B 消息头字段 |
| **L3** | 内核子系统 API | 可重构 | 补丁序列修复所有调用点 | 内核 EXPORT_SYMBOL |
| **L4** | 内部实现 | 完全自由 | 无约束 | static 函数、私有结构 |

### 2.2 L1 层：永久稳定的用户空间 ABI

L1 层接口遵循「永不破坏」承诺：

```c
/* include/uapi/linux/airymax/syscalls.h（[SC] 共享契约层 SSoT）
 * 一旦 syscall 编号分配，永久不可复用。
 * 4 核心系统调用 + 20 预留槽位，编号 548-571（避开 x32 历史区域 512-547） */
#define AIRY_SYS_CALL            548   /* IPC send/recv（永久支持） */
#define AIRY_SYS_ROVOL_CTL       549   /* MemoryRoVol control（永久支持） */
#define AIRY_SYS_SCHED_CTL       550   /* Scheduler control（永久支持） */
#define AIRY_SYS_CLT_NOTIFY      551   /* Cognition lifecycle notify（永久支持） */
/* 预留槽位 552-571（20 个），用于未来扩展，未分配前不得占用 */
```

> **SSoT 声明**：syscall 编号的单一数据源为 `include/uapi/linux/airymax/syscalls.h`，与 `arch/x86/entry/syscalls/syscall_64.tbl`、`asm-generic/unistd.h` 保持三方一致（ABI 铁律）。本节不再就地重定义编号，仅引用 SSoT 源。参见 `140-application-development/07-syscall-registry.md`。

L1 层变更规则：
- **禁止**：删除已有 syscall 编号、改变参数语义、改变返回值含义
- **允许**：新增 syscall 编号（追加至预留槽位 552-571，再向后扩展）、新增字段（必须可空字段）
- **弃用**：经 6 个月宽限期后，syscall 编号保留占位（永不复用）但实现可移除

### 2.3 L2 层：中等稳定的 AgentsIPC 协议

L2 层接口允许向后兼容的扩展：

```c
/* include/uapi/linux/airymax/ipc.h（[SC] 共享契约层，物理宿主 kernel/include/uapi/linux/airymax/ipc.h） */
/* IPC 128B 消息头定义见 [SC] 共享契约层（SSoT），不就地重定义 */
#include <airymax/ipc.h>
/* 结构体名称：struct airy_ipc_msg_hdr（Layout C，物理宿主见
 * 50-engineering-standards/120-cross-project-code-sharing.md §Layout C） */
```

> **SSoT 声明**：本节 IPC 128B 消息头不再就地重定义，以 `include/uapi/linux/airymax/ipc.h`（物理宿主见 `50-engineering-standards/120-cross-project-code-sharing.md` §Layout C）为单一数据源。结构体名称为 `struct airy_ipc_msg_hdr`（Layout C）。

L2 层变更规则：
- **禁止**：改变 128B 消息头大小、改变已用字段语义
- **允许**：新增 payload 类型（追加 enum 值）、使用 reserved 字段
- **版本协商**：`version` 字段支持运行时协商

### 2.4 L3 层：可重构的内核子系统 API

L3 层接口允许自由重构，但需修复所有调用点：

```c
/* 内核内部 API（非 EXPORT_SYMBOL），可自由重构 */
static int airy_internal_sched_adjust(struct airy_agent *agent);
/* 重构为： */
static int airy_internal_sched_adjust_v2(struct airy_agent *agent,
					     uint32_t new_prio);
/* 必须修复所有调用点，无外部影响 */
```

L3 层变更规则：
- **允许**：自由重构（重命名、改变签名、改变实现）
- **要求**：同一补丁序列内修复所有调用点
- **禁止**：影响 L1/L2 层接口语义

### 2.5 L4 层：完全自由的内部实现

L4 层是文件内 static 函数、私有数据结构，完全无约束：

```c
/* 文件内 static 函数，完全自由 */
static inline void airy_update_timestamp(struct airy_agent *a)
{
	a->last_update = ktime_get();
}
```

---

## 3. ABI 审查流程

### 3.1 工程规范委员会

agentrt-linux 设立 工程规范委员会（ABI Review Board, ARB）：

| 角色 | 职责 | 人数 |
|------|------|------|
| 架构师 | ABI 设计原则把控 | 2 |
| 内核维护者 | 内核 KABI 评估 | 2 |
| SDK 维护者 | SDK 兼容性评估 | 1 |
| 安全官 | 安全影响评估 | 1 |
| 兼容性测试 | 自动化测试验证 | 1 |

### 3.2 审查流程

```
[1] 提交 RFC（含 ABI 变更说明）
       |
       v
[2] ARB 初审（7 天内反馈）
       |
       v
[3] 公开评审（社区 14 天讨论）
       |
       v
[4] ARB 终审（通过/拒绝/要求修改）
       |
       v
[5] 实现 + 兼容性测试
       |
       v
[6] 合入主线（如弃用，启动 6 月宽限期）
```

### 3.3 RFC 模板

ABI 变更 RFC 必须包含以下内容：

```markdown
# RFC: <ABI 变更标题>

## 变更摘要
<一句话描述>

## 变更类型
- [ ] L1 新增（追加 syscall）
- [ ] L1 弃用（6 月宽限期）
- [ ] L2 协议扩展（向后兼容）
- [ ] L3 重构（修复调用点）

## 影响分析
- 用户空间影响：<描述>
- SDK 影响：<描述>
- 向后兼容：<是/否，机制>

## 弃用计划（如适用）
- 弃用版本：<版本>
- 移除版本：<版本>
- 迁移指南：<链接>

## 兼容性测试
- 测试用例：<描述>
- 回归测试：<描述>
```

---

## 4. 弃用声明机制

### 4.1 弃用声明语法

弃用通过内核注释 + 编译期警告 + 运行时日志三重声明。**当前 0.1.1/1.0.1 尚无任何已弃用 syscall**（4 核心 syscall 548-551 自 v1.0.1 引入，永久支持）。以下为假设性示例，展示未来若需弃用某个 syscall 时的声明语法：

```c
/* include/uapi/linux/airymax/syscalls.h（[SC] 共享契约层 SSoT）
 * 弃用声明范例 — 当前 4 核心 syscall 无一进入弃用流程，此为模板 */
/**
 * AIRY_SYS_<NAME> - <描述>（假设性弃用范例）
 *
 * @deprecated since <X.Y.Z>, use AIRY_SYS_<REPLACEMENT> instead
 * @scheduled_removal: <X.Y.Z+1>（6 个月后）
 * @migration_guide: docs/migration/<name>.md
 *
 * 弃用原因：<具体原因>
 */
/* 注：syscall 编号一旦分配，永不复用。弃用后实现可移除，但编号
 * 保留占位（参见 §2.2 L1 层规则）。当前 4 核心 syscall (548-551)
 * 均处于活跃状态，无弃用计划。 */

/* 内核实现侧的弃用标记（范例，非实际代码） */
__attribute__((deprecated("use <replacement> instead")))
int airy_sys_<name>(struct airy_<name>_req __user *req);
```

### 4.2 运行时弃用警告

```c
/* 内核 syscall 入口检测弃用接口调用（范例，当前无弃用 syscall） */
SYSCALL_DEFINE1(airy_<name>, struct airy_<name>_req __user *, req)
{
	static atomic_t warn_count = ATOMIC_INIT(0);

	/* 限频警告（每 Agent 每小时最多 1 次） */
	if (atomic_inc_return(&warn_count) % 3600 == 0) {
		pr_warn("agentrt: AIRY_SYS_<NAME> is deprecated "
			"since <X.Y.Z>, use AIRY_SYS_<REPLACEMENT>. "
			"Will be removed in <X.Y.Z+1>. "
			"Migration: docs/migration/<name>.md\n");
	}

	return do_<name>(req);
}
```

### 4.3 弃用登记表

所有弃用接口登记在 `Documentation/ABI/airymaxos-deprecated.md`。**当前（v1.0.1）无任何已弃用接口**，下表为模板示例：

| 接口 | 弃用版本 | 移除版本 | 替代接口 | 迁移指南 |
|------|----------|----------|----------|----------|
| _（暂无已弃用接口，4 核心 syscall 548-551 自 v1.0.1 起永久支持）_ | — | — | — | — |

---

## 5. syscall 从弃用到移除的 6 个月流程

### 5.1 完整流程示例

**当前 v1.0.1 无任何进入弃用流程的 syscall**（4 核心 syscall 548-551 均为活跃状态）。以下为假设性范例，展示未来若某 syscall 进入弃用流程时的完整 6 个月时间线：

```
月份 0：声明弃用
  ├── ARB 通过弃用 RFC
  ├── 内核注释标记 @deprecated
  ├── 编译警告启用（-Wdeprecated）
  ├── 运行时警告启用（限频）
  ├── 文档更新（Documentation/ABI/airymaxos-deprecated.md）
  └── 通知 SDK 团队（四语言 SDK 标记弃用）

月份 1-2：生态迁移期
  ├── SDK 释放新版本（默认使用新接口）
  ├── 官方 Agent 应用迁移至新接口
  ├── 兼容性测试套件更新
  └── 社区公告（博客 + 邮件列表）

月份 3-4：监测期
  ├── 监控弃用接口调用频率（Prometheus 指标）
  ├── 若调用频率仍高，延长宽限期
  ├── 联系高频调用方协助迁移
  └── 发布迁移进度报告

月份 5：最后警告期
  ├── 运行时警告升级（每次调用都警告）
  ├── 编译警告升级（-Werror=deprecated）
  ├── 发布「即将移除」公告
  └── 提供 shim 兼容层（如必要）

月份 6：移除
  ├── 内核移除 syscall 实现
  ├── syscall 编号保留占位（永不复用，参见 §2.2 L1 规则）
  ├── 文档标记为「已移除」
  ├── 发布移除公告
  └── 提供迁移完成报告
```

### 5.2 弃用追踪指标

| 指标 | 0 月 | 3 月 | 6 月 | 移除条件 |
|------|------|------|------|----------|
| 弃用接口调用次数 | 10000/天 | 1000/天 | <100/天 | <100/天方可移除 |
| 调用方 Agent 数 | 50 | 10 | <5 | <5 方可移除 |
| SDK 默认使用新接口 | 否 | 是 | 是 | 是 |

### 5.3 编译期弃用警告

```c
/* 用户态头文件中的弃用标记（范例，当前无弃用接口） */
#include <linux/compiler.h>

#define AIRY_DEPRECATED(since, replacement) \
	__attribute__((deprecated("since " since ", use " replacement " instead")))

/* 范例：若 AIRY_SYS_<OLD> 弃用，则其用户态包装函数标记如下 */
AIRY_DEPRECATED("<X.Y.Z>", "airy_sys_<replacement>")
int airy_sys_<old>(struct airy_<old>_req *req);

/* 用户编译时若调用此函数，触发警告：
 * warning: 'airy_sys_<old>' is deprecated:
 *          since <X.Y.Z>, use airy_sys_<replacement> instead
 */
```

---

## 6. 版本协商机制

### 6.1 AgentsIPC 版本协商

AgentsIPC 协议通过 `version` 字段支持运行时版本协商：

```c
/**
 * airy_ipc_negotiate_version - 版本协商
 * @client_version: 客户端支持的版本
 *
 * 返回双方共同支持的最高版本，或 -AIRY_ENOTSUP
 */
uint16_t airy_ipc_negotiate_version(uint16_t client_version)
{
	uint16_t server_version = AIRY_IPC_VERSION_CURRENT;  /* 服务端版本 */

	/* 客户端版本高于服务端：服务端尝试降级 */
	if (client_version > server_version) {
		if (client_version <= AIRY_IPC_VERSION_MAX_COMPAT)
			return server_version;  /* 服务端版本，客户端降级 */
		return -AIRY_ENOTSUP;
	}

	/* 客户端版本低于服务端：服务端尝试兼容 */
	if (client_version < AIRY_IPC_VERSION_MIN_COMPAT)
		return -AIRY_ENOTSUP;

	/* 使用客户端版本（向后兼容） */
	return client_version;
}
```

### 6.2 版本兼容矩阵

| 服务端版本 | 客户端版本 | 协商结果 |
|------------|------------|----------|
| 1.0 | 1.0 | 1.0 |
| 1.0 | 1.1 | 1.0（服务端降级） |
| 1.1 | 1.0 | 1.0（向后兼容） |
| 1.0 | 2.0 | -ENOTSUP（不兼容） |

### 6.3 特性探测

**当前 v1.0.1 仅有 4 核心 syscall (548-551)，无独立的特性探测 syscall**（`AIRY_SYS_FEATURE_QUERY` 不存在，syscalls.h SSoT 中未定义）。特性探测通过以下两种合规机制实现：

1. **IPC 版本协商（推荐）**：通过 `AIRY_SYS_CALL` (548) 发送 IPC 探测消息，服务端在响应中返回支持的特性位图与版本。
2. **/proc/airymax/features 读取**：用户态读取 procfs 节点获取特性列表（参见 `30-interfaces/02-procfs.md`）。

```c
/* 特性探测通过 IPC 协议层完成（非独立 syscall） */
#include <linux/airymax/syscalls.h>  /* SSoT: AIRY_SYS_CALL = 548 */
#include <linux/airymax/ipc.h>       /* SSoT: IPC 消息头 */

/* 客户端发送 FEATURE_QUERY IPC 消息（payload type = FEATURE_QUERY） */
struct airy_ipc_msg_hdr hdr = {
	.magic    = AIRY_IPC_MAGIC,
	.version  = AIRY_IPC_VERSION_CURRENT,
	.opcode   = AIRY_IPC_OP_FEATURE_QUERY,  /* IPC 层 opcode，非 syscall */
	.payload_len = 0,
};

int ret = syscall(AIRY_SYS_CALL, &hdr, NULL, 0);
if (ret == 0) {
	/* 响应 payload 中包含特性位图与版本，解析后启用对应特性 */
} else {
	/* 降级至 [DSL] 或最小功能集 */
}
```

> **SSoT 声明**：特性探测走 IPC 协议层（`AIRY_IPC_OP_FEATURE_QUERY` opcode），不引入独立 syscall。这避免了 syscall 编号膨胀，与 syscalls.h SSoT（4 核心 + 20 预留）一致。

---

## 7. ABI 测试与验证

### 7.1 ABI 快照测试

每个版本发布前生成 ABI 快照，与上一版本对比：

```bash
# 生成 ABI 快照
make CHECK_ABI=1 abidump

# 对比两个版本的 ABI 快照（IRON-8: 0.1.1 后直接过渡到 1.0.1）
abi-compliance-checker -l agentrt-linux \
	-old abidump-1.0.1.xml \
	-new abidump-<X.Y.Z>.xml

# 输出兼容性报告
# - Compatible：向后兼容
# - Incompatible：不兼容（必须修复）
```

### 7.2 兼容性测试矩阵

| 测试维度 | 验证方法 | 通过标准 |
|----------|----------|----------|
| syscall 编号 | 编号表对比 | 无删除、无复用 |
| syscall 签名 | 参数类型对比 | 类型不变 |
| struct 布局 | offsetof 对比 | 已有字段偏移不变 |
| enum 值 | 枚举值对比 | 已有值不变 |
| 错误码 | 错误码表对比 | 已有错误码不变 |

### 7.3 回归测试套件

```bash
# 运行 ABI 兼容性回归测试（IRON-8: 0.1.1 → 1.0.1 直跳，无中间版本）
./test_abi_compat --old-version=1.0.1 --new-version=<X.Y.Z>

# 运行弃用接口回归测试（当前无弃用接口，套件为占位）
./test_deprecated --expect-warnings
```

---

## 8. KABI 管理

### 8.1 KABI 白名单

内核导出符号（EXPORT_SYMBOL）通过白名单管理：

```bash
# 检查 KABI 兼容性
make CHECK_KABI=1 modules

# KABI 白名单文件
# kernel/abi/whitelist-airymaxos
# 格式：<symbol_name> <checksum> <version>
```

### 8.2 KABI 变更处理

| 变更类型 | 处理方式 |
|----------|----------|
| 新增导出符号 | 追加至白名单 |
| 删除导出符号 | 禁止（需先弃用） |
| 改变函数签名 | 禁止（需新增符号） |

---

## 9. 相关文档

- `160-compatibility/README.md`（兼容性主索引）
- `160-compatibility/02-posix-compat.md`（POSIX 兼容性设计）
- `160-compatibility/03-upstream-tracking.md`（上游追踪策略）
- `30-interfaces/01-syscalls.md`（系统调用接口）
- `50-engineering-standards/04-engineering-philosophy.md`（双层稳定性哲学）

---

## 10. 参考材料

- Linux 6.6 `Documentation/ABI/`（ABI 文档规范）
- Linux 内核 KABI 子系统
- Linux 内核「we don't break userspace」原则
- 语义化版本规范（SemVer）
- libabigail 项目（ABI 兼容性检查）
- abi-compliance-checker 工具
- agentrt ABI 稳定性规范（IRON-9 v3 [SC] 同源）

---

> **文档结束** | agentrt-linux（AirymaxOS）ABI 稳定性设计 v1.0.1
> 遵循 IRON-9 v3 [SC] 共享契约层 + [SS] 语义同源层与 agentrt ABI 同源
