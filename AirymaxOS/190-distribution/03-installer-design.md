Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）安装器设计

> **文档定位**: agentrt-linux（AirymaxOS，极境智能体操作系统）系统安装器工程设计文档，覆盖分区方案、引导加载器配置、最小安装 vs 完整安装、kickstart 配置
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 1. 概述

### 1.1 设计目标

agentrt-linux（AirymaxOS）作为智能体操作系统发行版，其安装器是用户首次接触产品的入口。本文档定义安装器工程设计，目标是：

1. **三模式安装**：最小安装（Server）/ 完整安装（Desktop）/ 自定义安装（Expert）三种模式覆盖不同场景
2. **kickstart 自动化**：支持 kickstart 配置文件实现无人值守安装，适用于云环境批量部署
3. **分区智能**：根据用户选择自动生成分区方案，支持 Btrfs 子卷、LVM、RAID
4. **引导加载器安全**：GRUB + Secure Boot + TPM 度量，确保引导链可信

### 1.2 基线约束

本设计严格遵循 Linux 6.6 内核基线，并执行以下硬约束：

| 约束项 | 取值 | 说明 |
|--------|------|------|
| 安装器框架 | Anaconda 衍生 | 主流 Linux 发行版标准 |
| 文件系统 | Btrfs（默认）/ XFS（备选） | Btrfs 支持子卷与快照 |
| 引导加载器 | GRUB 2.06+ | 支持 Secure Boot |
| 内核版本 | Linux 6.6 | agentrt-linux 1.0.1 锁定基线 |
| 字符串安全 | `strscpy` | 安装器代码中字符串操作必须安全 |

### 1.3 术语规范

本文档严格遵循术语规范：agentrt（用户态运行时）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）。安装器设计独立于 agentrt（[IND] 完全独立层），不与 agentrt 共享安装器代码。文中不使用 openEuler/Euler 字样，安装器实践对照一律表述为"主流 Linux 发行版"。

---

## 2. 三种安装模式

### 2.1 模式概览

| 模式 | 用途 | 安装大小 | 安装包数 | 安装时间 |
|------|------|----------|----------|----------|
| **最小安装（Minimal）** | 服务器/容器宿主 | 1.2GB | 280 | 8 分钟 |
| **完整安装（Full）** | 桌面/开发工作站 | 4.5GB | 1240 | 22 分钟 |
| **自定义安装（Expert）** | 高级用户 | 可变 | 用户选择 | 可变 |

### 2.2 最小安装包列表

最小安装包含运行 agentrt-linux 所需的最小包集：

```
最小安装包组：
  - @core                          # 基础系统
  - @base                          # 基础工具
  - airymaxos-kernel               # 内核
  - airymaxos-bootloader           # 引导加载器
  - airymaxos-initramfs            # initramfs
  - airymaxos-services             # 12 个 Daemon（最小子集）
  - airymaxos-security             # Cupolas 安全穹顶
  - airymaxos-system               # 系统编排工具
  - systemd                       # 服务管理
  - dnf                           # 包管理器
  - openssh-server                # SSH 服务
  - NetworkManager                # 网络管理
  - firewalld                     # 防火墙
```

### 2.3 完整安装包列表

完整安装在最小安装基础上增加：

```
完整安装额外包组：
  - @desktop                      # 桌面环境（GNOME）
  - @development-tools            # 开发工具
  - airymaxos-cognition           # CoreLoopThree 认知运行时
  - airymaxos-memory              # MemoryRovol 记忆卷载
  - airymaxos-cloudnative         # K8s/容器集成
  - airymaxos-tests-linux               # 基准测试套件
  - airymaxos-docs                # 双语文档
  - airymaxos-sdk-python          # Python SDK
  - airymaxos-sdk-go              # Go SDK
  - airymaxos-sdk-rust            # Rust SDK
  - airymaxos-sdk-java            # Java SDK
  - airymaxos-devel               # 开发头文件
  - vim, emacs                    # 编辑器
  - git, make, cmake              # 版本控制与构建
```

---

## 3. 分区方案

### 3.1 默认分区方案

agentrt-linux 默认采用 Btrfs 子卷方案，支持快照回滚：

| 分区 | 挂载点 | 文件系统 | 大小 | 说明 |
|------|--------|----------|------|------|
| `/dev/sda1` | `/boot/efi` | FAT32 | 200MB | EFI 系统分区（UEFI） |
| `/dev/sda2` | `/boot` | XFS | 1GB | 内核与 initramfs |
| `/dev/sda3` | `/` | Btrfs | 剩余空间 | 根分区（含子卷） |

### 3.2 Btrfs 子卷布局

```
/dev/sda3 (Btrfs)
├── @                          # 根子卷 → /
│   ├── usr/
│   ├── etc/
│   ├── var/
│   └── ...
├── @home                      # 用户家目录 → /home
├── @var                       # 可变数据 → /var
├── @opt                       # 第三方软件 → /opt
├── @srv                       # 服务数据 → /srv
├── @tmp                       # 临时文件 → /tmp
└── @snapshots                 # 快照 → /.snapshots
    ├── pre-install/            # 安装前快照
    └── post-install/           # 安装后快照
```

### 3.3 服务器分区方案（LVM）

服务器场景推荐 LVM 方案，便于动态调整分区大小：

| 逻辑卷 | 挂载点 | 文件系统 | 大小 |
|--------|--------|----------|------|
| `lv_root` | `/` | XFS | 20GB |
| `lv_home` | `/home` | XFS | 10GB |
| `lv_var` | `/var` | XFS | 10GB |
| `lv_var_lib_agentrt` | `/var/lib/agentrt` | Btrfs | 剩余空间 |
| `lv_swap` | swap | swap | 8GB |

### 3.4 PMEM 持久化分区

当系统配有 PMEM 设备时，MemoryRovol L1 原始卷独占 PMEM 分区：

| 分区 | 挂载点 | 文件系统 | 大小 | 说明 |
|------|--------|----------|------|------|
| `/dev/pmem0` | `/var/lib/agentrt/memory_rovol/L1_raw` | XFS (DAX) | 800GB | MemoryRovol L1 |

---

## 4. 引导加载器配置

### 4.1 GRUB 配置

agentrt-linux 使用 GRUB 2.06+ 作为引导加载器，支持 UEFI 与 BIOS 两种模式：

```
/boot/efi/EFI/airymaxos/
├── grubx64.efi                 # GRUB EFI 主程序
├── grub.cfg                    # GRUB 配置
├── MokManager.efi              # Secure Boot 密钥管理
└── shimx64.efi                 # Secure Boot shim
```

### 4.2 GRUB 配置文件

```bash
# /boot/efi/EFI/airymaxos/grub.cfg
set default="0"
set timeout=5

menuentry "agentrt-linux (AirymaxOS) 1.0.1" {
    linux /boot/vmlinuz-1.0.1-1 root=/dev/mapper/airymaxos-root \
        ro crashkernel=auto rd.lvm.lv=airymaxos/root \
        rd.lvm.lv=airymaxos/swap \
        rhgb quiet \
        agentrt.sched_ext=1 \
        agentrt.io_uring_batch=64 \
        agentrt.mglru_min_ttl_ms=10000 \
        systemd.machine_id=cgroup
    initrd /boot/initramfs-1.0.1-1.img
}

menuentry "agentrt-linux (AirymaxOS) 1.0.1 (Recovery Mode)" {
    linux /boot/vmlinuz-1.0.1-1 root=/dev/mapper/airymaxos-root \
        ro single \
        agentrt.sched_ext=0 \
        systemd.unit=rescue.target
    initrd /boot/initramfs-1.0.1-1.img
}
```

### 4.3 Secure Boot 配置

agentrt-linux 启用 Secure Boot，所有内核模块与 GRUB EFI 程序必须签名：

```bash
# 1. 生成 Secure Boot 密钥
openssl req -new -x509 -newkey rsa:2048 \
    -subj "/CN=AirymaxOS Platform Key/" \
    -keyout PK.key -out PK.crt -days 3650 -nodes

openssl req -new -x509 -newkey rsa:2048 \
    -subj "/CN=AirymaxOS Key Exchange Key/" \
    -keyout KEK.key -out KEK.crt -days 3650 -nodes

openssl req -new -x509 -newkey rsa:2048 \
    -subj "/CN=AirymaxOS Kernel Signing Key/" \
    -keyout db.key -out db.crt -days 3650 -nodes

# 2. 签名内核
sbsign --key db.key --cert db.crt \
    --output /boot/vmlinuz-1.0.1-1.signed \
    /boot/vmlinuz-1.0.1-1

# 3. 注册密钥到 UEFI 固件
mokutil --import PK.crt
mokutil --import KEK.crt
mokutil --import db.crt
```

### 4.4 TPM 度量

agentrt-linux 在引导阶段启用 TPM 2.0 度量，记录引导链各阶段哈希：

```bash
# 查看 TPM 度量值
tpm2_pcrread -L sha256:0,1,2,3,4,5,6,7

# PCR 寄存器含义：
# PCR0:  UEFI 固件
# PCR1:  UEFI 配置
# PCR2:  Option ROM
# PCR3:  Option ROM 配置
# PCR4:  引导加载器（GRUB）
# PCR5:  GPT 分区表
# PCR6:  电源状态
# PCR7:  Secure Boot 状态
```

---

## 5. kickstart 配置

### 5.1 最小安装 kickstart

```bash
# airymaxos-minimal.ks [IND]
# agentrt-linux (AirymaxOS) 最小安装 kickstart 配置

# 安装类型
install
cdrom

# 语言与键盘
lang en_US.UTF-8
keyboard us

# 时区
timezone Asia/Shanghai --isUtc

# 网络
network --bootproto=dhcp --device=eth0 --activate
network --hostname=agentrt-server

# Root 密码（加密）
rootpw --iscrypted $6$rounds=5000$saltsalt$hashhashhash

# 普通用户
user --name=agentrt --password=$6$rounds=5000$saltsalt$hashhashhash \
    --groups=wheel --shell=/bin/bash

# 分区方案
# 清空磁盘
clearpart --all --initlabel

# Btrfs 子卷方案
part /boot/efi --fstype=efi --size=200
part /boot --fstype=xfs --size=1024
part btrfs.01 --fstype=btrfs --size=1 --grow

btrfs none --label=airymaxos btrfs.01
btrfs / --subvol --name=@ btrfs.01
btrfs /home --subvol --name=@home btrfs.01
btrfs /var --subvol --name=@var btrfs.01
btrfs /opt --subvol --name=@opt btrfs.01
btrfs /srv --subvol --name=@srv btrfs.01
btrfs /tmp --subvol --name=@tmp btrfs.01
btrfs /.snapshots --subvol --name=@snapshots btrfs.01

# 引导加载器
bootloader --location=mbr --boot-drive=sda \
    --append="agentrt.sched_ext=1 agentrt.io_uring_batch=64"

# 软件包选择
%packages
@core
@base
airymaxos-kernel
airymaxos-bootloader
airymaxos-initramfs
airymaxos-services
airymaxos-security
airymaxos-system
systemd
dnf
openssh-server
NetworkManager
firewalld
%end

# 安装后脚本
%post --log=/var/log/airymaxos-install.log

# 启用 agentrt-linux 服务
systemctl enable gateway_d.service
systemctl enable llm_d.service
systemctl enable memory_d.service
systemctl enable sched_d.service
systemctl enable airymaxos-sched-agent.service

# 配置 dnf 仓库
cat > /etc/yum.repos.d/airymaxos.repo <<'EOF'
[airymaxos-stable]
name=agentrt-linux (AirymaxOS) Stable Repository
baseurl=https://repo.airymaxos.dev/1.0/stable/$basearch/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
EOF

# 导入 GPG 公钥
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos

# 创建安装后快照
btrfs subvolume snapshot / /.snapshots/post-install/$(date +%Y%m%d)

# 配置 MGLRU
echo "y" > /sys/kernel/mm/lru_gen/enabled
echo "10000" > /sys/kernel/mm/lru_gen/min_ttl_ms

echo "agentrt-linux 最小安装完成" >> /var/log/airymaxos-install.log
%end

# 重启
reboot
```

### 5.2 完整安装 kickstart

```bash
# airymaxos-full.ks [IND]
# agentrt-linux (AirymaxOS) 完整安装 kickstart 配置

install
cdrom

lang en_US.UTF-8
keyboard us
timezone Asia/Shanghai --isUtc

network --bootproto=dhcp --device=eth0 --activate
network --hostname=agentrt-workstation

rootpw --iscrypted $6$rounds=5000$saltsalt$hashhashhash

user --name=agentrt --password=$6$rounds=5000$saltsalt$hashhashhash \
    --groups=wheel --shell=/bin/bash

# 分区方案（LVM）
clearpart --all --initlabel

part /boot/efi --fstype=efi --size=200
part /boot --fstype=xfs --size=1024
part pv.01 --size=1 --grow

volgroup airymaxos pv.01
logvol / --vgname=airymaxos --size=20480 --name=root --fstype=xfs
logvol /home --vgname=airymaxos --size=10240 --name=home --fstype=xfs
logvol /var --vgname=airymaxos --size=10240 --name=var --fstype=xfs
logvol /var/lib/agentrt --vgname=airymaxos --size=1 --grow \
    --name=agentrt --fstype=btrfs
logvol swap --vgname=airymaxos --size=8192 --name=swap

# 引导加载器
bootloader --location=mbr --boot-drive=sda \
    --append="agentrt.sched_ext=1 agentrt.io_uring_batch=64 \
              agentrt.mglru_min_ttl_ms=10000"

# 软件包选择（完整）
%packages
@core
@base
@desktop
@development-tools
airymaxos-kernel
airymaxos-bootloader
airymaxos-initramfs
airymaxos-services
airymaxos-cognition
airymaxos-memory
airymaxos-security
airymaxos-cloudnative
airymaxos-system
airymaxos-tests-linux
airymaxos-docs
airymaxos-sdk-python
airymaxos-sdk-go
airymaxos-sdk-rust
airymaxos-sdk-java
airymaxos-devel
systemd
dnf
openssh-server
NetworkManager
firewalld
vim
emacs
git
make
cmake
%end

%post --log=/var/log/airymaxos-install.log
# 启用所有 agentrt-linux 服务
systemctl enable gateway_d.service
systemctl enable llm_d.service
systemctl enable tool_d.service
systemctl enable market_d.service
systemctl enable sched_d.service
systemctl enable monit_d.service
systemctl enable channel_d.service
systemctl enable observe_d.service
systemctl enable notify_d.service
systemctl enable info_d.service
systemctl enable hook_d.service
systemctl enable plugin_d.service
systemctl enable airymaxos-sched-agent.service

# 配置 dnf 仓库
cat > /etc/yum.repos.d/airymaxos.repo <<'EOF'
[airymaxos-stable]
name=agentrt-linux (AirymaxOS) Stable Repository
baseurl=https://repo.airymaxos.dev/1.0/stable/$basearch/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos
EOF

rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos

# 配置 MemoryRovol
mkdir -p /var/lib/agentrt/memory_rovol/L1_raw
mkdir -p /var/lib/agentrt/memory_rovol/L2_feat
mkdir -p /var/lib/agentrt/memory_rovol/L3_str
mkdir -p /var/lib/agentrt/memory_rovol/L4_pat

echo "y" > /sys/kernel/mm/lru_gen/enabled
echo "10000" > /sys/kernel/mm/lru_gen/min_ttl_ms

echo "agentrt-linux 完整安装完成" >> /var/log/airymaxos-install.log
%end

reboot
```

### 5.3 PMEM 专用 kickstart 片段

当系统配有 PMEM 设备时，kickstart 须包含 PMEM 分区配置：

```bash
# PMEM 分区配置片段
%post

# 1. 识别 PMEM 设备
PMEM_DEV=$(ndctl list -R | jq -r '.[0].dev' | sed 's/namespace/pmem/')

if [ -n "$PMEM_DEV" ]; then
    # 2. 格式化为 DAX 文件系统
    mkfs.xfs -m dax=indev /dev/${PMEM_DEV}p1

    # 3. 挂载 MemoryRovol L1
    mkdir -p /var/lib/agentrt/memory_rovol/L1_raw
    echo "/dev/${PMEM_DEV}p1 /var/lib/agentrt/memory_rovol/L1_raw xfs dax 0 0" \
        >> /etc/fstab
    mount /var/lib/agentrt/memory_rovol/L1_raw

    # 4. 验证 DAX 启用
    xfs_io -c "statx -v" /var/lib/agentrt/memory_rovol/L1_raw \
        | grep STATX_ATTR_DAX
fi
%end
```

---

## 6. 安装流程

### 6.1 安装阶段

```
┌──────────────────────────────────────────────────────────┐
│                  agentrt-linux 安装流程                    │
└──────────────────────────────────────────────────────────┘

  ┌─────────────────┐
  │ 1. 引导 ISO      │
  │   (UEFI/BIOS)   │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 2. 加载内核      │
  │   vmlinuz +     │
  │   initramfs     │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 3. 启动安装器    │
  │   (Anaconda)    │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 4. 选择安装模式  │
  │   最小/完整/自定义│
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 5. 配置分区      │
  │   Btrfs/LVM/RAID│
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 6. 配置网络      │
  │   DHCP/静态      │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 7. 配置用户      │
  │   root + 普通用户│
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 8. 安装包        │
  │   dnf install   │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 9. 安装后配置    │
  │   GRUB + 服务   │
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 10. 重启         │
  │   进入新系统     │
  └─────────────────┘
```

### 6.2 安装日志

安装日志位于 `/var/log/airymaxos-install.log`：

```
2026-07-09 10:00:00 [INFO] agentrt-linux 安装器启动
2026-07-09 10:00:01 [INFO] 安装模式: 完整安装
2026-07-09 10:00:02 [INFO] 分区方案: Btrfs 子卷
2026-07-09 10:00:05 [INFO] 网络: DHCP, eth0
2026-07-09 10:00:10 [INFO] 用户配置完成
2026-07-09 10:00:15 [INFO] 开始安装包（1240 个）
2026-07-09 10:08:30 [INFO] 包安装完成
2026-07-09 10:08:35 [INFO] GRUB 配置完成
2026-07-09 10:08:40 [INFO] 服务启用: gateway_d, llm_d, ...
2026-07-09 10:08:45 [INFO] dnf 仓库配置完成
2026-07-09 10:08:50 [INFO] MGLRU 配置完成
2026-07-09 10:08:55 [INFO] 安装后快照创建完成
2026-07-09 10:09:00 [INFO] agentrt-linux 完整安装完成
2026-07-09 10:09:05 [INFO] 准备重启
```

---

## 7. ISO 镜像构建

### 7.1 ISO 构建流程

```bash
# 1. 准备 ISO 根目录
mkdir -p /var/lib/airymaxos-iso/

# 2. 复制 GRUB 与 isolinux
mkdir -p /var/lib/airymaxos-iso/EFI/BOOT/
mkdir -p /var/lib/airymaxos-iso/isolinux/

cp /boot/efi/EFI/airymaxos/grubx64.efi \
    /var/lib/airymaxos-iso/EFI/BOOT/BOOTX64.EFI
cp /usr/share/syslinux/isolinux.bin \
    /var/lib/airymaxos-iso/isolinux/

# 3. 复制内核与 initramfs
cp /boot/vmlinuz-1.0.1-1 \
    /var/lib/airymaxos-iso/isolinux/vmlinuz
cp /boot/initramfs-1.0.1-1.img \
    /var/lib/airymaxos-iso/isolinux/initrd.img

# 4. 复制 agentrt-linux 仓库（用于离线安装）
cp -r /var/www/repo/airymaxos/1.0/stable/x86_64/ \
    /var/lib/airymaxos-iso/packages/

# 5. 生成 ISO
mkisofs -o airymaxos-1.0.1-x86_64.iso \
    -b isolinux/isolinux.bin \
    -c isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    -eltorito-alt-boot \
    -e EFI/BOOT/BOOTX64.EFI \
    -no-emul-boot \
    -V "agentrt-linux 1.0.1" \
    -R -J -T \
    /var/lib/airymaxos-iso/
```

### 7.2 ISO 校验

```bash
# 1. 计算 SHA256
sha256sum airymaxos-1.0.1-x86_64.iso > airymaxos-1.0.1-x86_64.iso.sha256

# 2. GPG 签名 ISO
gpg --detach-sign --armor -u "iso-sign@airymaxos.dev" \
    airymaxos-1.0.1-x86_64.iso

# 3. 验证签名
gpg --verify airymaxos-1.0.1-x86_64.iso.asc airymaxos-1.0.1-x86_64.iso
```

---

## 8. 安装测试

### 8.1 测试矩阵

| 测试场景 | 安装模式 | 架构 | 文件系统 | Secure Boot | TPM | 通过标准 |
|---------|----------|------|----------|-------------|-----|----------|
| 最小安装 | Minimal | x86_64 | Btrfs | 关 | 无 | 引导成功 |
| 最小安装 | Minimal | aarch64 | Btrfs | 关 | 无 | 引导成功 |
| 完整安装 | Full | x86_64 | LVM | 开 | 2.0 | 引导 + 服务启动 |
| 完整安装 | Full | aarch64 | Btrfs | 开 | 2.0 | 引导 + 服务启动 |
| PMEM 安装 | Full | x86_64 | Btrfs + DAX | 开 | 2.0 | MemoryRovol L1 启用 |
| kickstart | Minimal | x86_64 | Btrfs | 关 | 无 | 无人值守完成 |

### 8.2 自动化测试

```bash
#!/bin/bash
# tests-linux/install/test_install.sh [IND]
# 自动化安装测试

set -euo pipefail

ISO_PATH="$1"
TEST_VM_NAME="airymaxos-test-$$"

if [ -z "$ISO_PATH" ]; then
    echo "Usage: test_install.sh <iso_path>"
    exit 1
fi

# 1. 创建测试 VM
virt-install \
    --name "$TEST_VM_NAME" \
    --memory 4096 \
    --vcpus 2 \
    --disk size=40 \
    --cdrom "$ISO_PATH" \
    --os-variant fedora38 \
    --noautoconsole \
    --wait 30

# 2. 等待安装完成（最长 30 分钟）
TIMEOUT=1800
ELAPSED=0
while [ $ELAPSED -lt $TIMEOUT ]; do
    STATE=$(virsh domstate "$TEST_VM_NAME" 2>/dev/null || echo "unknown")
    if [ "$STATE" = "shut off" ]; then
        echo "安装完成，VM 已关机"
        break
    fi
    sleep 30
    ELAPSED=$((ELAPSED + 30))
    echo "等待安装完成... ${ELAPSED}s"
done

# 3. 启动 VM 验证
virsh start "$TEST_VM_NAME"
sleep 30

# 4. SSH 验证
ssh -o StrictHostKeyChecking=no agentrt@localhost \
    "uname -r && systemctl is-active gateway_d" || exit 1

# 5. 清理
virsh destroy "$TEST_VM_NAME"
virsh undefine "$TEST_VM_NAME"

echo "OK: 安装测试通过"
```

---

## 9. 错误处理

### 9.1 安装失败处理

| 错误场景 | 处理方式 |
|---------|---------|
| 磁盘空间不足 | 提示用户调整分区 |
| 网络不可用 | 切换到离线安装（使用 ISO 内置包） |
| Secure Boot 校验失败 | 提示用户禁用 Secure Boot 或注册密钥 |
| 包签名验证失败 | 中止安装，提示仓库损坏 |
| GRUB 安装失败 | 回滚分区变更，提示用户手动安装 |

### 9.2 错误码定义

安装器子系统错误码遵循 ErrorCodeSystem，定义于 [SC] 共享契约层：

```c
#define AGENTRT_INST_EDISK      (-940)  /* 磁盘错误 */
#define AGENTRT_INST_EPART      (-941)  /* 分区错误 */
#define AGENTRT_INST_EBOOT      (-942)  /* 引导加载器错误 */
#define AGENTRT_INST_ENET       (-943)  /* 网络错误 */
#define AGENTRT_INST_EPKG       (-944)  /* 包安装错误 */
#define AGENTRT_INST_ESIGN      (-945)  /* 签名校验失败 */
#define AGENTRT_INST_ESECURE    (-946)  /* Secure Boot 错误 */
#define AGENTRT_INST_ETPM       (-947)  /* TPM 错误 */
#define AGENTRT_INST_EPMEM      (-948)  /* PMEM 配置错误 */
```

### 9.3 集中错误处理

安装器代码采用 `goto out_free_xxx` 集中错误处理：

```c
int airymaxos_installer_run(struct install_config *cfg)
{
    struct install_ctx *ctx;
    int err;

    ctx = installer_alloc_ctx();
    if (!ctx)
        return -ENOMEM;

    err = installer_init_network(ctx, cfg);
    if (err)
        goto out_free_ctx;

    err = installer_partition_disk(ctx, cfg);
    if (err)
        goto out_free_network;

    err = installer_install_packages(ctx, cfg);
    if (err)
        goto out_free_partition;

    err = installer_configure_grub(ctx, cfg);
    if (err)
        goto out_free_packages;

    err = installer_enable_services(ctx, cfg);
    if (err)
        goto out_free_grub;

    err = installer_create_snapshot(ctx);
    if (err)
        goto out_free_services;

    pr_info("agentrt-linux: 安装完成\n");
    return 0;

out_free_services:
    installer_release_services(ctx);
out_free_grub:
    installer_release_grub(ctx);
out_free_packages:
    installer_release_packages(ctx);
out_free_partition:
    installer_release_partition(ctx);
out_free_network:
    installer_release_network(ctx);
out_free_ctx:
    kfree(ctx);
    return err;
}
```

---

## 10. 五维原则映射

| 原则 | 在安装器设计中的体现 |
|------|---------------------|
| **S-4 涌现性管理** | 三种安装模式管理用户场景 |
| **E-3 资源确定性** | 分区方案确定性 |
| **E-6 错误可追溯** | 安装日志全程记录 |
| **K-2 接口契约化** | kickstart 是安装契约 |
| **IRON-9 v2 [IND]** | 安装器独立于 agentrt |

---

## 11. 相关文档

- `190-distribution/README.md`（发行版管理主索引）
- `190-distribution/01-rpm-packaging.md`（RPM 打包设计）
- `190-distribution/02-dnf-repo-design.md`（dnf 仓库设计）
- `100-operations/README.md`（部署运维）
- `110-security/README.md`（Secure Boot 与 TPM）
- `170-performance/02-memory-performance.md`（PMEM 分区）

---

## 12. 参考材料

- Linux 6.6 Anaconda 安装器
- kickstart 语法规范
- GRUB 2 手册
- UEFI Secure Boot 规范
- TPM 2.0 规范
- Btrfs 文件系统手册
- 主流 Linux 发行版安装器实践

---

## 13. 版本演进

| 版本 | 安装模式 | 分区 | 引导 | kickstart | 备注 |
|------|---------|------|------|-----------|------|
| 0.1.1 | 设计 | — | — | — | 仅文档 |
| 1.0.1 | 三模式 | Btrfs/LVM | GRUB + SB | 完整 | 开发 |
| 1.1.0 | + Cloud | + RAID | + TPM | + 模板 | 云环境 |
| 2.0.0 | + Container | + OverlayFS | + Measured | + AI 辅助 | 完整 |

---

> **文档结束** | agentrt-linux（AirymaxOS）安装器设计 v1.0.1
