Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）兼容性设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）兼容性工程体系主索引
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **优先级**: P1（0.1.1 仅创建 README 占位，1.0.1 完成 5 文档）
> **同源映射**: agentrt ABI 稳定性 + Linux 6.6 KABI / ABI 兼容性体系
> **理论根基**: Linux 内核兼容性哲学 + Airymax K-2 接口契约化 + C-2 增量演化

---

## 1. 模块定位

agentrt-liunx 兼容性体系是系统长期演进的工程保障。它继承 Linux 内核 30+ 年沉淀的兼容性哲学（用户空间 ABI 永不破坏 + 内核内部 API 不保证稳定 + KABI 内核符号兼容性），并在其上扩展智能体操作系统专属的 AgentsIPC 协议兼容性、SDK 跨版本兼容性、Agent 应用迁移兼容性等。

兼容性是操作系统的生命线。agentrt-liunx 作为智能体操作系统发行版，必须承诺：在 0.1.1 上编写的 Agent 应用，到 1.0.1、3.0 LTS、5.0 LTS 上仍能正常运行；在 1.0.1 上开发的 Agent 应用，能通过兼容层在 0.1.1 上降级运行。这种双向兼容承诺是 IRON-9 v2 同源且部分代码共享原则的延伸——agentrt 应用级接口与 agentrt-liunx OS 级接口同源且部分代码共享演进。

### 1.1 兼容性分层

| 层级 | 类型 | 范围 | 承诺 |
|------|------|------|------|
| L1 | 用户空间 ABI | 系统调用 + ioctl + procfs + sysfs | 永不破坏（OS-IRON-001） |
| L2 | 内核 KABI | EXPORT_SYMBOL 导出符号 | 主版本内兼容 |
| L3 | AgentsIPC 协议 | 128B 消息头 + 5 种 payload | 语义版本化演进 |
| L4 | SDK API | Python / Rust / Go / TS | 语义版本号 |
| **L5** | **Agent 应用迁移** | **agentrt-liunx 专属** | **0.1.1 ↔ 1.0.1 双向** |
| **L6** | **跨发行版兼容** | **agentrt-liunx 专属** | **与主流 Linux 发行版** |

### 1.2 agentrt-liunx 扩展

- **Agent 应用迁移**：0.1.1 ↔ 1.0.1 之间 Agent 应用平滑迁移
- **跨发行版兼容**：与主流 Linux 发行版（Ubuntu / Debian / Fedora）的二进制兼容性
- **KABI 白名单**：内核导出符号的白名单管理
- **AgentsIPC 版本协商**：协议版本运行时协商机制

---

## 2. 核心兼容性机制

### 2.1 用户空间 ABI 永不破坏

**OS-IRON-001**：一旦接口导出至用户空间，必须永久支持。

```c
// airymaxos-kernel/include/uapi/agentrt/syscall.h
// 系统调用编号一旦分配，永不复用
#define AGENTRT_SYS_COGNITION_PROCESS  1001  // 1.0.1 引入，永久支持
#define AGENTRT_SYS_MEMORY_ROVOL_GET   1002  // 1.0.1 引入，永久支持
```

### 2.2 内核 KABI 管理

内核导出符号的 KABI（Kernel Application Binary Interface）管理：

```bash
# 检查 KABI 兼容性
make CHECK_KABI=1 modules

# KABI 白名单
# kernel/abi/whitelist-airymaxos
```

### 2.3 AgentsIPC 版本协商

```c
struct agentrt_ipc_header {
    uint16_t version;  // 协议版本
    // ...
};

// 接收方检查版本
if (header.version > SUPPORTED_VERSION) {
    return -AGENTRT_ENOTSUP;  // 触发发送方降级
}
```

### 2.4 SDK 语义版本

四语言 SDK 遵循语义化版本：

| 版本变更 | 含义 | 兼容性 |
|---------|------|--------|
| MAJOR | 不兼容变更 | 需迁移 |
| MINOR | 向后兼容新增 | 自动兼容 |
| PATCH | 缺陷修复 | 完全兼容 |

---

## 3. 文档索引

```
160-compatibility/
├── README.md                       # 本文件
├── 01-abi-stability.md             # 用户空间 ABI 稳定性
├── 02-kabi-management.md           # 内核 KABI 管理
├── 03-ipc-versioning.md            # AgentsIPC 版本协商
├── 04-sdk-compatibility.md        # SDK 跨版本兼容性
└── 05-cross-distro.md             # 跨发行版兼容性
```

### 3.1 0.1.1 版本范围

仅完成 README 占位（P1 优先级，0.1.1 不开发具体文档）。

### 3.2 1.0.1 版本范围

完成全部 5 文档，并实施兼容性工程标准。

---

## 4. agentrt-liunx 专属扩展

### 4.1 4 层接口稳定性分级

| 层级 | 接口类型 | 稳定性 | 变更流程 |
|------|---------|--------|---------|
| L1 | Agent 应用 API | 极稳定 | RFC + ABI 审查 + 6 个月宽限期 |
| L2 | AgentsIPC 协议 | 中等稳定 | 季度评审 + 兼容性测试 |
| L3 | 内核子系统 API | 可重构 | 补丁序列修复所有调用点 |
| L4 | 内部实现 | 完全自由 | 无约束 |

### 4.2 Agent 应用迁移

| 方向 | 兼容性 | 机制 |
|------|--------|------|
| 0.1.1 → 1.0.1 | 向前兼容 | ABI 永不破坏 |
| 1.0.1 → 0.1.1 | 向后兼容 | 降级运行（功能受限） |

### 4.3 跨发行版兼容

| 发行版 | 二进制兼容 | 说明 |
|--------|-----------|------|
| Ubuntu 22.04+ | ✓ | glibc 2.35+ |
| Debian 12+ | ✓ | glibc 2.36+ |
| Fedora 36+ | ✓ | glibc 2.35+ |
| RHEL 9+ | ✓ | glibc 2.34+ |

### 4.4 同源 agentrt 兼容性

agentrt 的 ABI 稳定性与 agentrt-liunx 同源：
- agentrt 用户态：`agentrt_*` API 稳定
- agentrt-liunx 内核态：`AGENTRT_SYS_*` 系统调用稳定
- 两端通过同源语义保持兼容

### 4.5 IRON-9 v2 同源且部分代码共享

兼容性体系遵循 IRON-9 原则：
- agentrt 兼容性（用户态运行时 ABI）
- agentrt-liunx 兼容性（OS 级 ABI + KABI）
- 两端独立演进，但通过同源 API 保持互操作

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **K-2 接口契约化** | 4 层接口稳定性分级 |
| **C-2 增量演化** | 语义版本号 + 弃用流程 |
| **E-6 错误可追溯** | ABI 变更追溯 |
| **E-7 文档即代码** | ABI 文档化（Documentation/ABI/） |
| **IRON-9 v2 同源且部分代码共享** | 与 agentrt ABI 同源 |

---

## 6. 相关文档

- `50-engineering-standards/04-engineering-philosophy.md`（双层稳定性哲学）
- `50-engineering-standards/01-coding-standards.md`（ABI 编码规范）
- `30-interfaces/01-syscalls.md`（系统调用接口）
- `30-interfaces/02-ipc-protocol.md`（AgentsIPC 协议）
- `120-development-process/README.md`（稳定版维护）

---

## 7. 参考材料

- Linux 6.6 `Documentation/ABI/`（ABI 文档）
- Linux 6.6 KABI 子系统
- Linux 内核 LTS 兼容性实践
- 语义化版本规范（SemVer）
- agentrt ABI 稳定性规范

---

> **文档结束** | 0.1.1 P1 占位，1.0.1 完成 5 文档
