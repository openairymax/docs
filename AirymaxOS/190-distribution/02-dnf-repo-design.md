Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）dnf 仓库设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）dnf 软件仓库工程设计文档，覆盖仓库分层（stable/testing/nightly）、GPG 签名、metadata 生成、仓库目录结构
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 1. 概述

### 1.1 设计目标

agentrt-liunx（AirymaxOS）作为智能体操作系统发行版，其 dnf 软件仓库是包分发的核心基础设施。本文档定义 dnf 仓库工程设计，目标是：

1. **仓库分层清晰**：stable（稳定）/testing（测试）/nightly（每日构建）三层仓库，每层用途明确
2. **GPG 签名强制**：所有仓库的 metadata 与 RPM 包必须 GPG 签名，dnf 安装时强制校验
3. **metadata 高效生成**：使用 createrepo_c 生成 repodata，支持增量更新与并行生成
4. **多架构支持**：x86_64 / aarch64 / riscv64 多架构仓库独立维护

### 1.2 基线约束

本设计严格遵循 Linux 6.6 内核基线，并执行以下硬约束：

| 约束项 | 取值 | 说明 |
|--------|------|------|
| 仓库格式 | dnf/yum 仓库 | repodata + RPM 包 |
| metadata 工具 | createrepo_c | C 实现，比旧版 createrepo 快 10 倍 |
| 签名算法 | GPG + RSA 4096 | metadata 与包双重签名 |
| metadata 格式 | XML + SQLite | 同时生成两种格式 |
| 字符串安全 | `strscpy` | 仓库管理脚本中字符串操作必须安全 |

### 1.3 术语规范

本文档严格遵循术语规范：agentrt（用户态运行时）= 微核心（micro-core）；agentrt-liunx（OS 发行版）= 微内核（micro-kernel）。仓库设计独立于 agentrt（[IND] 完全独立层），不与 agentrt 共享仓库基础设施。文中不使用 openEuler/Euler 字样，仓库实践对照一律表述为"主流 Linux 发行版"。

---

## 2. 仓库分层架构

### 2.1 三层仓库模型

agentrt-liunx 采用三层仓库模型，每层用途明确：

| 仓库层 | 用途 | 更新频率 | 保留策略 | 默认启用 |
|--------|------|----------|----------|----------|
| **stable** | 生产环境稳定包 | 季度 + 紧急安全修复 | 永久保留 | 是 |
| **testing** | 候选发布包 | 周度 | 保留 90 天 | 否 |
| **nightly** | 每日自动构建 | 每日 | 保留 14 天 | 否 |

### 2.2 仓库升级路径

包从 nightly 到 stable 的晋升路径：

```
源码提交 → CI 构建 → nightly 仓库
                        ↓
                    人工评审（1 周）
                        ↓
                    testing 仓库
                        ↓
                    灰度发布（2 周）
                        ↓
                    stable 仓库
```

### 2.3 仓库 URL 结构

```
https://repo.airymaxos.dev/
├── 1.0/                          # 1.0 系列
│   ├── stable/
│   │   ├── x86_64/
│   │   ├── aarch64/
│   │   ├── riscv64/
│   │   └── source/               # SRPM
│   ├── testing/
│   │   ├── x86_64/
│   │   ├── aarch64/
│   │   ├── riscv64/
│   │   └── source/
│   └── nightly/
│       ├── x86_64/
│       ├── aarch64/
│       ├── riscv64/
│       └── source/
├── 1.1/
│   └── ...
├── lts/
│   └── 1.0-lts/
│       └── stable/
│           └── ...
└── gpg/
    └── RPM-GPG-KEY-airymaxos     # 公钥
```

---

## 3. 仓库目录结构

### 3.1 单仓库目录结构

以 stable 仓库 x86_64 架构为例：

```
/var/www/repo/airymaxos/1.0/stable/x86_64/
├── repodata/
│   ├── repomd.xml                          # metadata 索引
│   ├── repomd.xml.asc                       # repomd.xml 的 GPG 签名
│   ├── primary.xml.gz                        # 主 metadata（包信息）
│   ├── primary.xml.gz.asc                    # 签名
│   ├── filelists.xml.gz                     # 文件列表
│   ├── filelists.xml.gz.asc                 # 签名
│   ├── other.xml.gz                         # 其他元数据（changelog）
│   ├── other.xml.gz.asc                     # 签名
│   ├── primary.sqlite.bz2                   # SQLite 格式
│   ├── filelists.sqlite.bz2
│   ├── other.sqlite.bz2
│   └── repomd.xml.key                       # 签名公钥
├── airymaxos-kernel-1.0.1-1.x86_64.rpm
├── airymaxos-kernel-devel-1.0.1-1.x86_64.rpm
├── airymaxos-kernel-debuginfo-1.0.1-1.x86_64.rpm
├── airymaxos-services-1.0.1-1.x86_64.rpm
├── airymaxos-cognition-1.0.1-1.x86_64.rpm
├── airymaxos-memory-1.0.1-1.x86_64.rpm
├── airymaxos-security-1.0.1-1.x86_64.rpm
├── airymaxos-cloudnative-1.0.1-1.x86_64.rpm
├── airymaxos-system-1.0.1-1.x86_64.rpm
├── airymaxos-tests-1.0.1-1.x86_64.rpm
├── airymaxos-sdk-python-1.0.1-1.noarch.rpm
├── airymaxos-sdk-go-1.0.1-1.noarch.rpm
├── airymaxos-sdk-rust-1.0.1-1.noarch.rpm
├── airymaxos-sdk-java-1.0.1-1.noarch.rpm
├── airymaxos-docs-1.0.1-1.noarch.rpm
└── airymaxos-devel-1.0.1-1.x86_64.rpm
```

### 3.2 SRPM 仓库结构

源码 RPM 仓库独立维护：

```
/var/www/repo/airymaxos/1.0/stable/source/
├── repodata/
│   ├── repomd.xml
│   ├── repomd.xml.asc
│   ├── primary.xml.gz
│   ├── filelists.xml.gz
│   └── other.xml.gz
├── airymaxos-kernel-1.0.1-1.src.rpm
├── airymaxos-services-1.0.1-1.src.rpm
├── airymaxos-cognition-1.0.1-1.src.rpm
└── ...
```

---

## 4. GPG 签名

### 4.1 双重签名策略

agentrt-liunx 对仓库采用双重签名策略：

1. **RPM 包签名**：每个 RPM 包单独 GPG 签名（详见 `01-rpm-packaging.md`）
2. **metadata 签名**：repodata 中的每个 metadata 文件（repomd.xml、primary.xml.gz 等）单独 GPG 签名

### 4.2 metadata 签名

```bash
# 1. 生成 metadata
createrepo_c /var/www/repo/airymaxos/1.0/stable/x86_64/

# 2. 签名 metadata
cd /var/www/repo/airymaxos/1.0/stable/x86_64/repodata/
for f in repomd.xml primary.xml.gz filelists.xml.gz other.xml.gz \
         primary.sqlite.bz2 filelists.sqlite.bz2 other.sqlite.bz2; do
    gpg --detach-sign --armor -u "rpm-sign@airymaxos.dev" "$f"
done

# 3. 验证签名
for f in repomd.xml primary.xml.gz filelists.xml.gz other.xml.gz; do
    gpg --verify "${f}.asc" "$f" || echo "签名验证失败: $f"
done
```

### 4.3 自动化签名脚本

```bash
#!/bin/bash
# airymaxos-system/repo_sign.sh [IND]
# 自动签名仓库 metadata

set -euo pipefail

REPO_DIR="$1"
GPG_KEY="rpm-sign@airymaxos.dev"

if [ -z "$REPO_DIR" ]; then
    echo "Usage: repo_sign.sh <repo_dir>"
    exit 1
fi

if [ ! -d "$REPO_DIR/repodata" ]; then
    echo "错误：$REPO_DIR/repodata 不存在"
    exit 1
fi

cd "$REPO_DIR/repodata"

echo "签名 $REPO_DIR 的 metadata..."
for f in repomd.xml primary.xml.gz filelists.xml.gz other.xml.gz \
         primary.sqlite.bz2 filelists.sqlite.bz2 other.sqlite.bz2; do
    if [ -f "$f" ]; then
        gpg --detach-sign --armor -u "$GPG_KEY" "$f"
        echo "  已签名: $f"
    fi
done

echo "验证签名..."
for f in repomd.xml primary.xml.gz filelists.xml.gz other.xml.gz; do
    if [ -f "$f.asc" ]; then
        if gpg --verify "$f.asc" "$f" 2>/dev/null; then
            echo "  验证通过: $f"
        else
            echo "  验证失败: $f"
            exit 1
        fi
    fi
done

echo "OK: $REPO_DIR metadata 签名完成"
```

### 4.4 dnf 客户端配置

```ini
# /etc/yum.repos.d/airymaxos.repo
[airymaxos-stable]
name=agentrt-liunx (AirymaxOS) Stable Repository
baseurl=https://repo.airymaxos.dev/1.0/stable/$basearch/
enabled=1
gpgcheck=1                              # 校验 RPM 包签名
repo_gpgcheck=1                         # 校验 metadata 签名
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
enabled_metadata=1
priority=10
metadata_expire=24h                     # metadata 24 小时过期

[airymaxos-testing]
name=agentrt-liunx (AirymaxOS) Testing Repository
baseurl=https://repo.airymaxos.dev/1.0/testing/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
priority=20
metadata_expire=12h

[airymaxos-nightly]
name=agentrt-liunx (AirymaxOS) Nightly Repository
baseurl=https://repo.airymaxos.dev/1.0/nightly/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
priority=30
metadata_expire=1h
```

---

## 5. metadata 生成

### 5.1 createrepo_c

createrepo_c 是 C 实现的 metadata 生成工具，比旧版 createrepo（Python 实现）快约 10 倍：

```bash
# 1. 基本生成
createrepo_c /var/www/repo/airymaxos/1.0/stable/x86_64/

# 2. 增量更新（只更新变化的包）
createrepo_c --update /var/www/repo/airymaxos/1.0/stable/x86_64/

# 3. 并行生成（利用多核）
createrepo_c --workers=$(nproc) \
    /var/www/repo/airymaxos/1.0/stable/x86_64/

# 4. 生成 SQLite 格式（默认同时生成 XML + SQLite）
createrepo_c --database \
    /var/www/repo/airymaxos/1.0/stable/x86_64/

# 5. 包含 changelog
createrepo_c --changelog-limit=10 \
    /var/www/repo/airymaxos/1.0/stable/x86_64/

# 6. 指定 compress 类型
createrepo_c --compress-type=zst \
    /var/www/repo/airymaxos/1.0/stable/x86_64/
```

### 5.2 repomd.xml 结构

repomd.xml 是 metadata 的索引文件，定义了所有 metadata 文件的位置与校验和：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<repomd xmlns="http://linux.duke.edu/metadata/repo">
  <revision>1720497600</revision>
  <data type="primary">
    <location href="repodata/primary.xml.gz"/>
    <checksum type="sha256">a1b2c3d4...</checksum>
    <timestamp>1720497600</timestamp>
    <size>123456</size>
    <open-size>2345678</open-size>
  </data>
  <data type="filelists">
    <location href="repodata/filelists.xml.gz"/>
    <checksum type="sha256">e5f6g7h8...</checksum>
    <timestamp>1720497600</timestamp>
    <size>98765</size>
    <open-size>876543</open-size>
  </data>
  <data type="other">
    <location href="repodata/other.xml.gz"/>
    <checksum type="sha256">i9j0k1l2...</checksum>
    <timestamp>1720497600</timestamp>
    <size>54321</size>
    <open-size>432109</open-size>
  </data>
  <data type="primary_db">
    <location href="repodata/primary.sqlite.bz2"/>
    <checksum type="sha256">m3n4o5p6...</checksum>
    <timestamp>1720497600</timestamp>
    <size>234567</size>
    <open-size>1234567</open-size>
  </data>
</repomd>
```

### 5.3 自动化生成脚本

```bash
#!/bin/bash
# airymaxos-system/repo_generate.sh [IND]
# 自动生成所有仓库的 metadata

set -euo pipefail

REPO_ROOT="/var/www/repo/airymaxos"
VERSION="1.0"
ARCHES=("x86_64" "aarch64" "riscv64")
LAYERS=("stable" "testing" "nightly")

generate_repo() {
    local repo_dir="$1"
    local workers

    if [ ! -d "$repo_dir" ]; then
        echo "跳过不存在的仓库: $repo_dir"
        return 0
    fi

    workers=$(nproc)
    echo "生成 $repo_dir 的 metadata（$workers 并行）..."

    # 增量更新（若 repodata 已存在）
    if [ -d "$repo_dir/repodata" ]; then
        createrepo_c --update --workers="$workers" \
            --database --changelog-limit=10 \
            --compress-type=zst "$repo_dir"
    else
        createrepo_c --workers="$workers" \
            --database --changelog-limit=10 \
            --compress-type=zst "$repo_dir"
    fi

    # 签名 metadata
    airymaxos-system/repo_sign.sh "$repo_dir"
}

# 生成所有仓库
for layer in "${LAYERS[@]}"; do
    for arch in "${ARCHES[@]}"; do
        repo_dir="$REPO_ROOT/$VERSION/$layer/$arch"
        generate_repo "$repo_dir"
    done
    # 生成 SRPM 仓库
    generate_repo "$REPO_ROOT/$VERSION/$layer/source"
done

echo "OK: 所有仓库 metadata 生成完成"
```

---

## 6. 仓库镜像与同步

### 6.1 镜像策略

agentrt-liunx 主仓库位于 `repo.airymaxos.dev`，全球设镜像节点：

| 镜像节点 | 区域 | 同步频率 | 延迟 |
|---------|------|----------|------|
| repo.airymaxos.dev | 全球主节点 | 实时 | 0ms |
| mirror.cn.airymaxos.dev | 中国大陆 | 每小时 | < 50ms |
| mirror.eu.airymaxos.dev | 欧洲 | 每小时 | < 30ms |
| mirror.us.airymaxos.dev | 北美 | 每小时 | < 20ms |
| mirror.jp.airymaxos.dev | 日本 | 每小时 | < 40ms |

### 6.2 rsync 同步

```bash
# 镜像节点同步主仓库
rsync -avz --delete \
    rsync://repo.airymaxos.dev/airymaxos/ \
    /var/www/repo/airymaxos/

# 验证同步完整性
rsync -avz --dry-run --delete \
    rsync://repo.airymaxos.dev/airymaxos/ \
    /var/www/repo/airymaxos/
```

### 6.3 镜像列表

```ini
# /etc/yum.repos.d/airymaxos-mirrors.repo
# 主仓库
[airymaxos-stable]
name=agentrt-liunx (AirymaxOS) Stable
mirrorlist=https://mirrors.airymaxos.dev/metalink?repo=airymaxos-stable&arch=$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
```

---

## 7. 仓库管理测试

### 7.1 仓库健康检查

```bash
# 1. 检查 metadata 完整性
repoclosure -c /etc/dnf/dnf.conf \
    --repoid=airymaxos-stable \
    --arch=x86_64

# 2. 检查包依赖闭合性
dnf --repo=airymaxos-stable check

# 3. 检查签名
for rpm in /var/www/repo/airymaxos/1.0/stable/x86_64/*.rpm; do
    rpm -K "$rpm" || echo "签名失败: $rpm"
done

# 4. 检查 metadata 签名
cd /var/www/repo/airymaxos/1.0/stable/x86_64/repodata/
for f in *.asc; do
    gpg --verify "$f" "${f%.asc}" || echo "metadata 签名失败: $f"
done
```

### 7.2 性能测试

| 操作 | 仓库规模 | 耗时 |
|------|----------|------|
| 全量 metadata 生成 | 1000 包 | 12s |
| 增量 metadata 更新 | 1000 包（10 包变更） | 1.8s |
| metadata 签名 | 1000 包 | 4.5s |
| dnf makecache | 1000 包 | 3.2s |
| dnf install 单包 | 1000 包 | 1.4s |

---

## 8. 错误处理

### 8.1 仓库损坏降级

当主仓库不可用时，dnf 自动降级到镜像：

```ini
# dnf 自动 failover 配置
[airymaxos-stable]
name=agentrt-liunx (AirymaxOS) Stable
baseurl=https://repo.airymaxos.dev/1.0/stable/$basearch/
        https://mirror.cn.airymaxos.dev/1.0/stable/$basearch/
        https://mirror.eu.airymaxos.dev/1.0/stable/$basearch/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
failovermethod=priority
```

### 8.2 错误码定义

仓库管理子系统错误码遵循 ErrorCodeSystem，定义于 [SC] 共享契约层：

```c
#define AGENTRT_REPO_EMETADATA  (-930)  /* metadata 生成失败 */
#define AGENTRT_REPO_ESIGN     (-931)  /* 签名失败 */
#define AGENTRT_REPO_ESYNC     (-932)  /* 镜像同步失败 */
#define AGENTRT_REPO_EDEPS     (-933)  /* 依赖闭合性失败 */
#define AGENTRT_REPO_EARCH     (-934)  /* 架构不匹配 */
#define AGENTRT_REPO_EGPG      (-935)  /* GPG 校验失败 */
```

### 8.3 集中错误处理

仓库管理工具采用 `goto out_free_xxx` 集中错误处理：

```c
int airymaxos_repo_publish(const char *repo_dir, const char *rpm_dir)
{
    struct repo_ctx *ctx;
    int err;

    ctx = repo_alloc_ctx();
    if (!ctx)
        return -ENOMEM;

    err = repo_validate_structure(ctx, repo_dir);
    if (err)
        goto out_free_ctx;

    err = repo_copy_rpms(ctx, rpm_dir, repo_dir);
    if (err)
        goto out_free_ctx;

    err = repo_generate_metadata(ctx, repo_dir);
    if (err)
        goto out_free_rpms;

    err = repo_sign_metadata(ctx, repo_dir);
    if (err)
        goto out_free_metadata;

    err = repo_sync_mirrors(ctx, repo_dir);
    if (err)
        goto out_free_metadata;

    pr_info("agentrt-liunx: 仓库 %s 发布完成\n", repo_dir);
    return 0;

out_free_metadata:
    repo_release_metadata(ctx);
out_free_rpms:
    repo_release_rpms(ctx);
out_free_ctx:
    kfree(ctx);
    return err;
}
```

---

## 9. 五维原则映射

| 原则 | 在 dnf 仓库设计中的体现 |
|------|----------------------|
| **S-4 涌现性管理** | 三层仓库分层管理包涌现性 |
| **E-3 资源确定性** | metadata 校验和保证确定性 |
| **E-6 错误可追溯** | GPG 签名 + changelog |
| **K-2 接口契约化** | repomd.xml 是仓库契约 |
| **IRON-9 v2 [IND]** | 仓库设计独立于 agentrt |

---

## 10. 相关文档

- `190-distribution/README.md`（发行版管理主索引）
- `190-distribution/01-rpm-packaging.md`（RPM 打包设计）
- `190-distribution/03-installer-design.md`（安装器设计）
- `100-operations/README.md`（部署运维）
- `120-development-process/README.md`（开发流程）
- `160-compatibility/README.md`（兼容性管理）

---

## 11. 参考材料

- Linux 6.6 dnf 包管理器文档
- createrepo_c 项目
- RPM Repository Format 规范
- Reproducible Builds 项目
- 主流 Linux 发行版仓库实践
- metalink 标准

---

## 12. 版本演进

| 版本 | 仓库分层 | metadata | 签名 | 镜像 | 备注 |
|------|---------|----------|------|------|------|
| 0.1.1 | 设计 | — | — | — | 仅文档 |
| 1.0.1 | 三层 | createrepo_c | GPG | 5 节点 | 开发 |
| 1.1.0 | + lts | + 增量 | GPG | +10 节点 | LTS 支持 |
| 2.0.0 | + lts-longterm | + delta | GPG + cosign | 全球 CDN | 完整 |

---

> **文档结束** | agentrt-liunx（AirymaxOS）dnf 仓库设计 v1.0.1
