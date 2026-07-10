Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）RPM 打包设计详细规范

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）发行版 RPM 包结构与打包规范详细设计\
> **版本**：0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**：2026-07-09\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-linux（AirymaxOS）RPM 打包设计旨在为发行版构建提供可重现、可追溯、可验证的包管理标准。本设计聚焦三大目标：

1. **完整覆盖 8 子仓与 12 daemon**：将 airymaxos-kernel、airymaxos-services 等 8 子仓及其衍生的 12 个 daemon 完整打包为 RPM 包
2. **spec 文件规范统一**：所有 spec 文件遵循 K&R 风格 + Tab=8 缩进 + strscpy + goto out_free_xxx 的内核编码规范
3. **dnf 仓库可重现构建**：所有 RPM 包基于相同源码与构建环境可重现生成（reproducible build），SHA-256 一致

### 1.2 适用范围

- Linux 6.6 内核基线衍生包（kernel / kernel-modules / kernel-headers）
- 12 daemon 用户态服务包（airymaxos-services-*）
- SDK 多语言包（Python / Rust / Go / TypeScript）
- CJK 字体包与 locale 数据包
- 多架构镜像构建（x86_64 / aarch64 / riscv64）

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。所有外部 Linux 发行版统一表述为"主流 Linux 发行版"，禁止使用 openEuler/Euler 字样。RPM 包与 agentrt 之间属于 IRON-9 v2 [IND] 完全独立层。

### 1.4 RPM 包分类矩阵

| 类别 | 包名前缀 | 数量 | 备注 |
|------|----------|------|------|
| 内核 | airymaxos-kernel | 5 | kernel / modules / headers / devel / tools |
| 用户态服务 | airymaxos-services | 12 | gateway_d / llm_d / tool_d 等 |
| 认知运行时 | airymaxos-cognition | 3 | cognition / taskflow / mac |
| 记忆卷载 | airymaxos-memory | 4 | core / regions / forgetting / tiering |
| 安全穹顶 | airymaxos-security | 3 | cupolas / capability / lsm |
| 云原生 | airymaxos-cloudnative | 4 | container / k8s / etcd / cni |
| 系统编排 | airymaxos-system | 5 | systemd-units / init / config / docs / locales |
| 测试框架 | airymaxos-tests-linux | 6 | unit / perf / fuzz / integration / e2e / bench |
| SDK | airymaxos-sdk | 4 | python / rust / go / typescript |
| 字体 | airymaxos-fonts | 5 | cjk-zh-cn / cjk-zh-tw / cjk-ja / cjk-ko / common |
| 总计 | — | 51 | — |

---

## 2. RPM 包结构

### 2.1 顶层包结构

agentrt-linux 发行版顶层分为"内核"、"用户态"、"开发"三大组：

```bash
# 顶层组与依赖关系
airymaxos-base (虚拟组)
├── airymaxos-kernel              # 内核组
│   ├── airymaxos-kernel-core
│   ├── airymaxos-kernel-modules
│   ├── airymaxos-kernel-headers
│   ├── airymaxos-kernel-devel
│   └── airymaxos-kernel-tools
├── airymaxos-services            # 用户态服务组
│   ├── airymaxos-services-gateway
│   ├── airymaxos-services-llm
│   ├── airymaxos-services-tool
│   ├── airymaxos-services-sched
│   ├── airymaxos-services-monit
│   └── ... (共 12 个)
├── airymaxos-cognition           # 认知运行时
├── airymaxos-memory              # 记忆卷载
├── airymaxos-security            # 安全穹顶
├── airymaxos-cloudnative          # 云原生
├── airymaxos-system              # 系统编排
├── airymaxos-fonts                # 字体
└── airymaxos-sdk                  # 开发组（可选）
```

### 2.2 文件系统布局

每个 RPM 包遵循主流 Linux 发行版的文件系统层次标准（FHS）：

```
/usr/lib/airymaxos/
├── kernel/                      # 内核镜像与模块
│   ├── vmlinuz-6.6.0-agentrt
│   ├── initramfs-6.6.0-agentrt.img
│   └── modules/6.6.0-agentrt/
├── services/                    # daemon 二进制
│   ├── gateway_d
│   ├── llm_d
│   ├── tool_d
│   └── ...
├── cognition/                   # 认知运行时库
├── memory/                      # 记忆卷载库
└── security/                    # 安全穹顶库

/usr/lib/systemd/system/
├── gateway_d.service
├── llm_d.service
└── ...

/etc/airymaxos/
├── locale.conf
├── memory_tiering.conf
├── cxl.conf
├── pmem.conf
├── mglru.conf
└── ibus/config.ini

/usr/share/airymaxos/
├── locale/                      # 翻译文件
├── docs/                        # 文档
└── schemas/                     # 配置 schema

/var/lib/airymaxos/
├── memoryrovol/                 # 记忆卷载数据
├── cognition/                   # 认知状态
└── logs/                        # 日志

/var/log/airymaxos/              # 系统日志
```

---

## 3. spec 文件规范

### 3.1 spec 文件头部模板

所有 spec 文件必须包含以下头部信息：

```spec
# airymaxos-kernel.spec —— agentrt-linux 内核包 spec 文件
# SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
# Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

%global agentrt_version 1.0.1
%global agentrt_release 1
%global kernel_version 6.6.0
%global kernel_patchlevel 0
%global sched_ext_version 7

Name:           airymaxos-kernel
Version:        %{agentrt_version}
Release:        %{agentrt_release}%{?dist}
Summary:        agentrt-linux (AirymaxOS) Linux 6.6 kernel with sched_ext
License:        GPL-2.0-only OR AGPL-3.0-or-later
URL:            https://github.com/spharx/agentrt-linux
Source0:        airymaxos-kernel-%{version}.tar.gz
Source1:        airymaxos-kernel.config
Source2:        sched_agent_bpf.c

BuildRequires:  bc, binutils, elfutils-libelf-devel, gcc, make
BuildRequires:  bison, flex, openssl-devel
BuildRequires:  clang, llvm, libbpf-devel
Requires:       airymaxos-system >= %{version}

ExclusiveArch:  x86_64 aarch64 riscv64

%description
The agentrt-linux (AirymaxOS) kernel package contains the Linux 6.6 kernel
with sched_ext framework forward-ported from mainline 6.12+. This enables
the SCHED_AGENT scheduling class for Agent workloads.

%package core
Summary:        agentrt-linux kernel core (vmlinuz + initramfs)
Requires:       airymaxos-kernel-modules = %{version}-%{release}

%package modules
Summary:        agentrt-linux kernel modules

%package headers
Summary:        agentrt-linux kernel headers for development
BuildArch:      noarch

%package devel
Summary:        agentrt-linux kernel development package
Requires:       airymaxos-kernel-headers = %{version}-%{release}

%package tools
Summary:        agentrt-linux kernel tools (perf, bpftool, etc.)
Requires:       airymaxos-kernel-core = %{version}-%{release}
```

### 3.2 %prep 阶段

```spec
%prep
%setup -q -n airymaxos-kernel-%{version}

# 应用 sched_ext 向前移植补丁
%patch0 -p1 -b .sched_ext
%patch1 -p1 -b .sched_agent

# 准备 BPF 程序源码
cp %{SOURCE2} tools/sched_ext/sched_agent_bpf.c

# 复制默认内核配置
cp %{SOURCE1} .config
```

### 3.3 %build 阶段

```spec
%build
# 内核构建（使用 Tab=8 风格 + K&R C 风格）
make -j$(nproc) KCFLAGS="-Werror -Wmissing-prototypes" \
     HOSTCFLAGS="-Werror" \
     olddefconfig

make -j$(nproc) KCFLAGS="-Werror -Wmissing-prototypes" \
     HOSTCFLAGS="-Werror" \
     bzImage modules

# 构建 sched_agent BPF 程序
make -C tools/sched_ext \
     CC=clang \
     LLC=llc \
     sched_agent_bpf.o

# 构建内核工具
make -j$(nproc) -C tools/perf
make -j$(nproc) -C tools/bpf/bpftool
```

### 3.4 %install 阶段

```spec
%install
# 安装内核镜像
mkdir -p %{buildroot}/boot
install -m 644 arch/x86/boot/bzImage \
        %{buildroot}/boot/vmlinuz-%{kernel_version}-agentrt

# 安装内核模块
make INSTALL_MOD_PATH=%{buildroot} INSTALL_MOD_STRIP=1 \
     modules_install

# 安装内核头文件
make INSTALL_HDR_PATH=%{buildroot}/usr \
     headers_install

# 安装 sched_agent BPF 程序
mkdir -p %{buildroot}/usr/lib/airymaxos/sched_ext
install -m 644 tools/sched_ext/sched_agent_bpf.o \
        %{buildroot}/usr/lib/airymaxos/sched_ext/

# 安装内核工具
install -m 755 tools/perf/perf \
        %{buildroot}/usr/bin/airymax-perf
install -m 755 tools/bpf/bpftool/bpftool \
        %{buildroot}/usr/bin/airymax-bpftool

# 安装 systemd unit
mkdir -p %{buildroot}/usr/lib/systemd/system
install -m 644 airymaxos-kernel-load-bpf.service \
        %{buildroot}/usr/lib/systemd/system/
```

### 3.5 %files 段

```spec
%files core
/boot/vmlinuz-%{kernel_version}-agentrt
/boot/initramfs-%{kernel_version}-agentrt.img
%attr(0644, root, root) /usr/lib/airymaxos/sched_ext/sched_agent_bpf.o
/usr/lib/systemd/system/airymaxos-kernel-load-bpf.service

%files modules
/lib/modules/%{kernel_version}-agentrt/

%files headers
/usr/include/linux/
/usr/include/asm-generic/
/usr/include/uapi/

%files devel
/usr/lib/modules/%{kernel_version}-agentrt/build
/usr/lib/modules/%{kernel_version}-agentrt/source

%files tools
/usr/bin/airymax-perf
/usr/bin/airymax-bpftool

%changelog
* Wed Jul 09 2026 SPHARX <dev@spharx.com> - 1.0.1-1
- Initial agentrt-linux (AirymaxOS) 1.0.1 release
- Forward-port sched_ext from Linux 6.12+ to 6.6 baseline
- Add SCHED_AGENT BPF scheduler with Q16.16 vtime
- Add MemoryRovol L1-L4 tiered memory with CXL/PMEM support
```

---

## 4. daemon 包 spec 文件示例

### 4.1 llm_d daemon spec 文件

```spec
# airymaxos-services-llm.spec —— llm_d daemon 包
%global agentrt_version 1.0.1
%global agentrt_release 1

Name:           airymaxos-services-llm
Version:        %{agentrt_version}
Release:        %{agentrt_release}%{?dist}
Summary:        agentrt-linux LLM Inference Daemon (llm_d)
License:        AGPL-3.0-or-later
URL:            https://github.com/spharx/agentrt-linux
Source0:        airymaxos-services-%{version}.tar.gz

BuildRequires:  gcc, make, cmake
BuildRequires:  airymaxos-kernel-headers >= %{version}
BuildRequires:  airymaxos-sdk-python >= %{version}
BuildRequires:  openssl-devel, libcurl-devel, json-c-devel
Requires:       airymaxos-system >= %{version}
Requires:       airymaxos-cognition >= %{version}
Requires:       airymaxos-memory >= %{version}
Requires(post): systemd
Requires(preun): systemd

ExclusiveArch:  x86_64 aarch64

%description
The llm_d daemon provides LLM inference proxy for agentrt-linux (AirymaxOS).
Features include multi-provider routing, token caching, billing, and
SCHED_AGENT-aware task scheduling.

%prep
%setup -q -n airymaxos-services-%{version}

%build
mkdir -p build && cd build
cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr/lib/airymaxos/services \
    -DCMAKE_C_FLAGS="-Wall -Werror -Wmissing-prototypes" \
    -DAIRYMAX_LOCALE_DEFAULT=zh_CN.UTF-8
make -j$(nproc)

%install
cd build
make DESTDIR=%{buildroot} install

# 安装 systemd unit
mkdir -p %{buildroot}/usr/lib/systemd/system
install -m 644 ../systemd/llm_d.service \
        %{buildroot}/usr/lib/systemd/system/

# 安装默认配置
mkdir -p %{buildroot}/etc/airymaxos/services
install -m 644 ../config/llm_d.conf \
        %{buildroot}/etc/airymaxos/services/

# 安装 locale 数据
mkdir -p %{buildroot}/usr/share/locale/zh_CN/LC_MESSAGES
msgfmt ../po/zh_CN.po -o \
        %{buildroot}/usr/share/locale/zh_CN/LC_MESSAGES/airymaxos-llm_d.mo

%post
%systemd_post llm_d.service

%preun
%systemd_preun llm_d.service

%postun
%systemd_postun_with_restart llm_d.service

%files
%defattr(-, root, root, -)
/usr/lib/airymaxos/services/llm_d
/usr/lib/systemd/system/llm_d.service
%config(noreplace) /etc/airymaxos/services/llm_d.conf
/usr/share/locale/zh_CN/LC_MESSAGES/airymaxos-llm_d.mo
/usr/share/locale/en_US/LC_MESSAGES/airymaxos-llm_d.mo
/usr/share/locale/ja_JP/LC_MESSAGES/airymaxos-llm_d.mo
```

---

## 5. dnf 仓库管理

### 5.1 仓库结构

```bash
# agentrt-linux 官方仓库结构
/var/www/airymaxos-repo/
├── 1.0/                          # 1.0 系列
│   ├── 1.0.0/                    # 1.0.0 版本
│   │   ├── BaseOS/
│   │   │   ├── x86_64/
│   │   │   │   ├── Packages/
│   │   │   │   │   ├── airymaxos-kernel-1.0.0-1.x86_64.rpm
│   │   │   │   │   └── ...
│   │   │   │   └── repodata/
│   │   │   │       ├── repomd.xml
│   │   │   │       └── primary.xml.gz
│   │   │   └── aarch64/
│   │   │       └── ...
│   │   ├── AppStream/
│   │   └── updates/
│   └── 1.0.1/                    # 1.0.1 版本
│       └── ...
├── 1.0-lts/                      # 1.0 LTS 长期支持
│   └── ...
└── testing/                      # 测试版
    └── ...
```

### 5.2 客户端 dnf 仓库配置

```ini
# /etc/yum.repos.d/airymaxos.repo —— agentrt-linux dnf 仓库配置

[airymaxos-base]
name=agentrt-linux (AirymaxOS) BaseOS $releasever - $basearch
baseurl=https://repo.airymaxos.org/$releasever/BaseOS/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
repo_gpgcheck=1
metadata_expire=6h
skip_if_unavailable=True

[airymaxos-appstream]
name=agentrt-linux (AirymaxOS) AppStream $releasever - $basearch
baseurl=https://repo.airymaxos.org/$releasever/AppStream/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
metadata_expire=6h

[airymaxos-updates]
name=agentrt-linux (AirymaxOS) Updates $releasever - $basearch
baseurl=https://repo.airymaxos.org/$releasever/updates/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
metadata_expire=1h

[airymaxos-testing]
name=agentrt-linux (AirymaxOS) Testing $releasever - $basearch
baseurl=https://repo.airymaxos.org/testing/$basearch/
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos-testing
```

### 5.3 创建本地仓库

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/create_repo.sh
# 创建本地 dnf 仓库

set -euo pipefail
REPO_DIR=${1:-/var/www/airymaxos-repo/1.0.1}
VERSION=1.0.1

mkdir -p "$REPO_DIR"/{BaseOS,AppStream}/{x86_64,aarch64}

# 复制 RPM 包到对应目录
find /root/rpmbuild/RPMS -name "*.rpm" -exec cp {} \
    "$REPO_DIR/BaseOS/x86_64/" \;

# 生成仓库元数据
for arch in x86_64 aarch64; do
    for stream in BaseOS AppStream; do
        createrepo_c --update \
            "$REPO_DIR/$stream/$arch/"
    done
done

# 签名仓库元数据（GPG）
for arch in x86_64 aarch64; do
    for stream in BaseOS AppStream; do
        cd "$REPO_DIR/$stream/$arch/repodata/"
        gpg --detach-sign --armor repomd.xml
        cd -
    done
done

echo "[完成] 仓库已创建于 $REPO_DIR"
```

---

## 6. 可重现构建

### 6.1 可重现构建配置

```spec
# 可重现构建（reproducible build）配置
%global source_date_epoch 1468923400

%build
# 设置 SOURCE_DATE_EPOCH（确定性时间戳）
export SOURCE_DATE_EPOCH=%{source_date_epoch}

# 禁用构建路径嵌入
export CFLAGS="-ffile-prefix-map=$PWD=."
export KCFLAGS="-ffile-prefix-map=$PWD=."

# 禁用调试信息中的构建路径
make -j$(nproc) KCFLAGS="$KCFLAGS -fdebug-prefix-map=$PWD=." \
     HOSTCFLAGS="$CFLAGS" \
     bzImage modules

# 设置 RPM 构建时间戳为固定值
export TZ=UTC
```

### 6.2 构建环境标准化

```bash
# /etc/airymaxos/buildenv.conf —— 可重现构建环境
# 所有构建必须在相同环境进行

# 构建容器镜像版本
BUILD_IMAGE=airymaxos-buildenv:1.0.1

# 固定编译器版本
GCC_VERSION=13.2.1
CLANG_VERSION=17.0.6
RUSTC_VERSION=1.75.0

# 固定构建参数
export SOURCE_DATE_EPOCH=1468923400
export TZ=UTC
export LANG=C.UTF-8
export LC_ALL=C.UTF-8

# 禁用并行构建的不确定性
export MAKEFLAGS="-j$(nproc) --no-print-directory"
```

### 6.3 可重现构建验证

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/verify_reproducible.sh
# 验证 RPM 包可重现构建

set -euo pipefail
PACKAGE=$1
EXPECTED_SHA256=$2

# 重新构建
rpmbuild -bb /root/rpmbuild/SPECS/$PACKAGE.spec
NEW_RPM=$(find /root/rpmbuild/RPMS -name "$PACKAGE-*.rpm" | head -1)

# 计算 SHA-256
ACTUAL_SHA256=$(sha256sum "$NEW_RPM" | awk '{print $1}')

if [ "$ACTUAL_SHA256" = "$EXPECTED_SHA256" ]; then
    echo "[OK] $PACKAGE 可重现构建验证通过"
    echo "SHA-256: $ACTUAL_SHA256"
    exit 0
else
    echo "[FAIL] $PACKAGE 可重现构建验证失败"
    echo "期望: $EXPECTED_SHA256"
    echo "实际: $ACTUAL_SHA256"
    exit 1
fi
```

---

## 7. 多架构构建

### 7.1 多架构支持矩阵

| 架构 | 优先级 | 状态 | 备注 |
|------|--------|------|------|
| x86_64 | P0 | 主架构 | 默认构建 |
| aarch64 | P0 | 主架构 | ARM 服务器 |
| riscv64 | P1 | 次架构 | RISC-V 实验 |
| ppc64le | P2 | 实验架构 | PowerPC |

### 7.2 多架构构建脚本

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/build_multiarch.sh
# 多架构 RPM 构建

set -euo pipefail
PACKAGE=$1
VERSION=1.0.1

for arch in x86_64 aarch64 riscv64; do
    echo "[构建] $PACKAGE for $arch"
    mock -r airymaxos-$arch-$VERSION \
         --rebuild /root/rpmbuild/SRPMS/$PACKAGE-$VERSION-1.src.rpm \
         --resultdir=/var/tmp/airymaxos-build/$arch/
done

# 验证架构特定包
for arch in x86_64 aarch64 riscv64; do
    RPM=$(find /var/tmp/airymaxos-build/$arch/ \
           -name "$PACKAGE-*.$arch.rpm" | head -1)
    if [ -z "$RPM" ]; then
        echo "[失败] $arch 构建失败"
        exit 1
    fi
    echo "[成功] $arch: $RPM"
done
```

---

## 8. 错误码体系对接

发行版管理错误码纳入 agentrt-linux 统一错误码体系：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AGENTRT_E_PKG_BUILD | -301 | 包构建失败 |
| AGENTRT_E_PKG_INSTALL | -302 | 包安装失败 |
| AGENTRT_E_PKG_DEP | -303 | 依赖冲突 |
| AGENTRT_E_PKG_VERIFY | -304 | GPG 验证失败 |
| AGENTRT_E_PKG_REPRODUCIBLE | -305 | 可重现构建失败 |
| AGENTRT_E_PKG_ARCH | -306 | 架构不匹配 |

集中错误处理示例：

```c
int airymax_pkg_install(const char *package_name)
{
	int ret;

	if (!package_name) {
		ret = -EINVAL;
		goto out_err;
	}

	ret = airymax_pkg_check_dep(package_name);
	if (ret < 0) {
		ret = -AGENTRT_E_PKG_DEP;
		goto out_err;
	}

	ret = airymax_pkg_verify_gpg(package_name);
	if (ret < 0) {
		ret = -AGENTRT_E_PKG_VERIFY;
		goto out_err;
	}

	ret = system("dnf install -y %s", package_name);
	if (ret != 0) {
		ret = -AGENTRT_E_PKG_INSTALL;
		goto out_err;
	}
	return 0;

out_err:
	return ret;
}
```

---

## 9. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **S-4 涌现性管理** | 版本化 RPM 包管理涌现性 |
| **C-2 增量演化** | 语义化版本 + 增量发布 |
| **E-6 错误可追溯** | RPM 包 SHA-256 + GPG 签名 |
| **E-7 文档即代码** | spec 文件即文档 |
| **E-8 可测试性** | 可重现构建验证 |

---

## 10. IRON-9 v2 同源映射

| 组件 | agentrt-linux（[IND]） | agentrt（[IND]） |
|------|--------------------------|------------------|
| RPM 包 | spec 文件、二进制 | 无（用户态运行时） |
| 字体包 | RPM 打包 | 无 |
| locale 数据 | RPM 打包 | 用户态 .mo 文件 |
| 可重现构建 | mock + createrepo_c | 无 |

agentrt-linux 的 RPM 包与 agentrt 完全独立（[IND] 完全独立层），各自演进。

---

## 11. 相关文档

- `190-distribution/README.md`（发行版管理主索引）
- `190-distribution/03-installer-design.md`（安装器设计）
- `190-distribution/04-update-mechanism.md`（更新机制设计）
- `100-operations/README.md`（部署运维）
- `120-development-process/README.md`（开发流程）

---

## 12. 参考材料

- RPM Packaging Guide（Fedora 项目）
- dnf 包管理器文档
- createrepo_c 项目
- Reproducible Builds 项目
- mock 构建工具文档
- 主流 Linux 发行版 RPM 规范

---

> **文档结束** | agentrt-linux（AirymaxOS）RPM 打包设计详细规范 | 1.0.1 开发版本
