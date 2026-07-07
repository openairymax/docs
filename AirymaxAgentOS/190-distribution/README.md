Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）发行版管理设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）发行版管理工程体系主索引
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **优先级**: P2（0.1.1 仅创建 README 占位，1.0.1 完成 5 文档）
> **同源映射**: agentrt 版本管理 + Linux 6.6 发行版工程（RPM / dnf / ISO / 镜像）
> **理论根基**: Linux 发行版管理哲学 + Airymax S-4 涌现性管理 + C-2 增量演化

---

## 1. 模块定位

agentrt-liunx 发行版管理体系是操作系统作为产品发布的工程保障。它继承 Linux 发行版 30+ 年沉淀的发行版管理哲学（版本号 + RPM 包 + ISO 镜像 + 仓库管理 + 签名验证），并在其上扩展智能体操作系统专属的 Agent 应用商店、记忆卷载分发、多架构镜像等。

发行版管理不是「打包」的代名词，而是「版本化、可追溯、可验证」的工程体系。agentrt-liunx 发行版管理的核心理念是：**每个版本必须可追溯，每个包必须可验证，每个镜像必须可重现构建**。这要求从源码到发布的全链路工程化，而非手工打包。

### 1.1 发行版分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 版本号 | 语义化版本 + LTS 策略 | 版本管理 |
| L2 | 包格式 | RPM + dnf | 包管理 |
| L3 | 镜像构建 | ISO + QCOW2 + Docker | 镜像分发 |
| L4 | 仓库管理 | dnf 仓库 + 签名 | 软件仓库 |
| **L5** | **Agent 应用商店** | **agentrt-liunx 专属** | **Agent 应用分发** |
| **L6** | **记忆卷载分发** | **agentrt-liunx 专属** | **记忆卷载包** |

### 1.2 agentrt-liunx 扩展

- **Agent 应用商店**：Agent 应用作为 RPM 包分发
- **记忆卷载分发**：记忆卷载作为独立包分发
- **多架构镜像**：x86_64 / aarch64 / riscv64 多架构
- **可重现构建**：所有镜像可重现构建（reproducible build）

---

## 2. 核心发行版机制

### 2.1 版本号策略

| 版本类型 | 格式 | 生命周期 | 示例 |
|---------|------|---------|------|
| 文档体系版本 | 0.1.1 | 短期 | 0.1.1（文档体系完成） |
| 正式版本 | X.Y.Z | 2 年 | 1.0.1 |
| LTS 版本 | X.0 LTS | 5 年 | 1.0 LTS / 3.0 LTS |
| 维护版本 | X.Y.Z | 季度 | 1.0.1 / 1.0.2 |

### 2.2 RPM 包格式

```bash
# 构建内核 RPM
rpmbuild -bb airymaxos-kernel.spec

# 包命名规范
airymaxos-kernel-1.0.1-1.x86_64.rpm
airymaxos-services-1.0.1-1.x86_64.rpm
airymaxos-sdk-python-1.0.1-1.x86_64.rpm
```

### 2.3 ISO 镜像构建

```bash
# 构建 ISO 镜像
mkisofs -o airymaxos-1.0.1-x86_64.iso \
    -b isolinux/isolinux.bin \
    -c isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    airymaxos-root/
```

### 2.4 可重现构建

```bash
# 验证镜像可重现
sha256sum airymaxos-1.0.1-x86_64.iso
# 应与官方发布的 SHA256 一致
```

---

## 3. 文档索引

```
190-distribution/
├── README.md                       # 本文件
├── 01-versioning.md                # 版本号策略
├── 02-rpm-packaging.md             # RPM 打包规范
├── 03-iso-building.md              # ISO 镜像构建
├── 04-agent-store.md               # Agent 应用商店
└── 05-reproducible-build.md        # 可重现构建
```

### 3.1 0.1.1 版本范围

仅完成 README 占位（P2 优先级，0.1.1 不开发具体文档）。

### 3.2 1.0.1 版本范围

完成全部 5 文档，并实施发行版管理工程标准。

---

## 4. agentrt-liunx 专属扩展

### 4.1 Agent 应用商店

Agent 应用作为 RPM 包分发：

```bash
# 安装 Agent 应用
dnf install agent-cognition-v1.0.1

# Agent 应用元数据
Name: agent-cognition
Version: 1.0.1
Release: 1
License: AGPLv3+
Requires: airymaxos-sdk-python >= 1.0.1
```

### 4.2 记忆卷载分发

记忆卷载作为独立包分发：

```bash
# 安装记忆卷载包
dnf install memoryrovol-zh-cn-knowledge-1.0.1

# 记忆卷载包元数据
Name: memoryrovol-zh-cn-knowledge
Version: 1.0.1
Release: 1
Requires: airymaxos-memory >= 1.0.1
```

### 4.3 多架构镜像

| 架构 | 状态 | 优先级 |
|------|------|--------|
| x86_64 | 主架构 | P0 |
| aarch64 | 主架构 | P0 |
| riscv64 | 次架构 | P1 |
| ppc64le | 实验架构 | P2 |

### 4.4 同源 agentrt 版本管理

agentrt 的版本管理与 agentrt-liunx 同源：
- agentrt 用户态：语义化版本 + 0.1.1 唯一奠基版本
- agentrt-liunx 内核态：0.1.1 文档体系完成 + 1.0.1 正式开发
- 两端版本号协同但独立

### 4.5 IRON-9 v2 同源且部分代码共享

发行版管理遵循 IRON-9 原则：
- agentrt 版本管理（用户态运行时版本）
- agentrt-liunx 版本管理（OS 级 + 发行版版本）
- 两端独立演进，但通过同源版本号策略保持协同

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **S-4 涌现性管理** | 版本化演进管理涌现性 |
| **C-2 增量演化** | 语义化版本 + 增量发布 |
| **E-6 错误可追溯** | 版本可追溯 + 签名验证 |
| **E-7 文档即代码** | 发行说明文档化 |
| **IRON-9 v2 同源且部分代码共享** | 与 agentrt 版本管理同源 |

---

## 6. 相关文档

- `100-operations/README.md`（部署运维）
- `120-development-process/README.md`（开发流程 + 稳定版维护）
- `160-compatibility/README.md`（兼容性管理）
- `50-engineering-standards/05-development-process.md`（开发流程规范）
- `50-engineering-standards/07-maintainers-and-governance.md`（维护者治理）

---

## 7. 参考材料

- Linux 6.6 RPM 打包规范
- dnf 包管理器
- ISO 9660 镜像标准
- Reproducible Builds 项目
- agentrt 版本管理规范

---

> **文档结束** | 0.1.1 P2 占位，1.0.1 完成 5 文档
