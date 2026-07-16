Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）安装器设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）系统安装器工程设计文档，覆盖安装模式、分区方案、引导加载器配置、kickstart 自动化、ISO/QCOW2 镜像构建、cloud-init 云定制、安装测试与错误处理\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-09\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 概述

### 1.1 设计目标

agentrt-linux（AirymaxOS）作为智能体操作系统发行版，其安装器是用户首次接触产品的入口。本文档定义安装器工程设计，聚焦以下目标：

1. **三模式安装**：最小安装（Minimal）/ 完整安装（Full）/ 自定义安装（Expert）三种模式覆盖不同场景
2. **多场景覆盖**：支持裸机安装（ISO）、虚拟机部署（QCOW2）、容器化部署（Docker）、云镜像部署（cloud-init）、PXE 网络安装
3. **kickstart 自动化 100%**：所有安装场景均可通过 kickstart 文件自动化，无人值守完成
4. **分区智能**：根据用户选择自动生成分区方案，支持 Btrfs 子卷、LVM、RAID，预留 PMEM 持久化区域与 CXL 内存区域
5. **引导加载器安全**：GRUB + Secure Boot + TPM 2.0 度量，确保引导链可信

### 1.2 基线约束

本设计严格遵循 Linux 6.6 内核基线，并执行以下硬约束：

| 约束项 | 取值 | 说明 |
|--------|------|------|
| 安装器框架 | Anaconda 衍生 | 主流 Linux 发行版标准 |
| 文件系统 | Btrfs（默认）/ XFS（备选） | Btrfs 支持子卷与快照回滚 |
| 引导加载器 | GRUB 2.06+ | 支持 Secure Boot |
| 内核版本 | Linux 6.6 | agentrt-linux 1.0.1 锁定基线 |
| 字符串安全 | `strscpy` | 安装器代码中字符串操作必须安全（OS-BAN-004） |

### 1.3 术语规范

本文档严格遵循术语规范：agentrt（用户态运行时）= 微核心（micro-core）；agentrt-linux（OS 发行版）= 微内核（micro-kernel）。安装器设计独立于 agentrt（[IND] 完全独立层），不与 agentrt 共享安装器代码。文中不使用 主流 Linux 发行版/主流 Linux 发行版 字样，安装器实践对照一律表述为"主流 Linux 发行版"。

### 1.4 安装场景矩阵

| 场景 | 介质 | 自动化方式 | 适用 | 优先级 |
|------|------|------------|------|--------|
| 裸机安装 | ISO | kickstart | 物理服务器 | P0 |
| 虚拟机部署 | QCOW2 | cloud-init | KVM / VMware | P0 |
| 容器化部署 | Docker 镜像 | Dockerfile | K8s / Podman | P0 |
| 云镜像部署 | raw 镜像 | cloud-init | AWS / 阿里云 | P1 |
| PXE 网络安装 | PXE + kickstart | 大规模部署 | 数据中心 | P1 |

### 1.5 安装器组件

```bash
# 安装器主要组件
/usr/lib/airymaxos/installer/
├── anaconda-airymaxos        # 定制 Anaconda 安装器
├── pykickstart-airymaxos     # kickstart 解析器
├── airymaxos-dracut-modules  # dracut 模块（initramfs 生成）
├── airymaxos-iso-builder     # ISO 构建工具
├── airymaxos-qcow2-builder   # QCOW2 构建工具
└── airymaxos-cloud-init      # cloud-init 配置模板
```

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
  - airymaxos-tests-linux         # 基准测试套件
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

### 3.1 默认分区方案（Btrfs 子卷）

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

### 3.3 服务器分区方案（LVM + PMEM）

服务器场景推荐 LVM 方案，便于动态调整分区大小，并预留 PMEM 持久化区域：

| 逻辑卷 | 挂载点 | 文件系统 | 大小 |
|--------|--------|----------|------|
| `lv_root` | `/` | XFS | 100GB |
| `lv_home` | `/home` | XFS | 50GB |
| `lv_var` | `/var` | XFS | 200GB |
| `lv_var_lib_agentrt` | `/var/lib/agentrt` | Btrfs | 剩余空间 |
| `lv_swap` | swap | swap | 16GB |
| `lv_log` | `/var/log` | XFS | 20GB |
| `lv_tmp` | `/tmp` | XFS | 20GB |

### 3.4 PMEM 持久化分区

当系统配有 PMEM 设备时，MemoryRovol L1 原始卷独占 PMEM 分区：

| 分区 | 挂载点 | 文件系统 | 大小 | 说明 |
|------|--------|----------|------|------|
| `/dev/pmem0` | `/var/lib/agentrt/memory_rovol/L1_raw` | XFS (DAX) | 800GB | MemoryRovol L1 |

PMEM 与 CXL 自动检测脚本：

```bash
#!/bin/bash
# /usr/lib/airymaxos/installer/scripts/detect_pmem.sh
# 检测 PMEM 设备并配置 DAX 挂载

set -euo pipefail

PMEM_MOUNT=/var/lib/airymaxos/memoryrovol/pmem

# 检测 PMEM 区域
for region in $(ndctl list -R 2>/dev/null | jq -r '.[].dev'); do
    echo "[检测] PMEM 区域: $region"
    NS_DEV=$(ndctl create-namespace -r "$region" -m fsdax -f 2>/dev/null \
             | jq -r '.dev' || echo "")

    if [ -n "$NS_DEV" ] && [ -e "/dev/$NS_DEV" ]; then
        echo "[创建] DAX 命名空间: $NS_DEV"
        mkfs.xfs -m dax=inode -f "/dev/$NS_DEV"
        mkdir -p "$PMEM_MOUNT"
        echo "/dev/$NS_DEV $PMEM_MOUNT xfs dax 0 0" >> /etc/fstab
        mount "$PMEM_MOUNT"
    fi
done

# 检测 CXL 内存
for memdev in $(cxl list -M 2>/dev/null | jq -r '.[].memdev'); do
    echo "[检测] CXL 内存设备: $memdev"
done
```

### 3.5 大页（hugepage）分区配置

```bash
# /etc/airymaxos/sysctl.d/99-hugepage.conf
# 为 IPC 零拷贝区域预留 2MB 大页

vm.nr_hugepages = 1024

# 启用透明大页
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# 大页挂载点
hugetlbfs /var/lib/airymaxos/hugepages hugetlbfs mode=1770,gid=1000 0 0
```

---

## 4. 引导加载器配置

### 4.1 GRUB 目录结构

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
# /etc/default/grub —— agentrt-linux GRUB2 默认配置

GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="agentrt-linux (AirymaxOS)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_TERMINAL="console"
GRUB_CMDLINE_LINUX="crashkernel=auto \
    sched_ext.enable=1 \
    agentrt.memoryrovol=1 \
    agentrt.cxl=1 \
    agentrt.mglru=1 \
    transparent_hugepage=always \
    rd.driver.pre=cxl_acpi,cxl_mem \
    intel_iommu=on iommu=pt \
    loglevel=7"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
GRUB_ENABLE_CRYPTODISK=y
```

### 4.3 GRUB 菜单条目

```bash
# /boot/grub2/grub.cfg（自动生成）

menuentry "agentrt-linux (AirymaxOS) 1.0.1 (Linux 6.6.0-agentrt)" {
    set root=(hd0,gpt2)
    linux /boot/vmlinuz-6.6.0-agentrt \
        root=/dev/mapper/airymaxos--vg-root \
        ro crashkernel=auto \
        sched_ext.enable=1 \
        agentrt.memoryrovol=1 \
        agentrt.cxl=1 \
        agentrt.mglru=1 \
        transparent_hugepage=always \
        rd.driver.pre=cxl_acpi,cxl_mem \
        intel_iommu=on iommu=pt \
        loglevel=7
    initrd /boot/initramfs-6.6.0-agentrt.img
}

menuentry "agentrt-linux (AirymaxOS) 1.0.1 (recovery mode)" {
    set root=(hd0,gpt2)
    linux /boot/vmlinuz-6.6.0-agentrt \
        root=/dev/mapper/airymaxos--vg-root \
        ro single systemd.unit=rescue.target
    initrd /boot/initramfs-6.6.0-agentrt.img
}
```

### 4.4 Secure Boot 配置

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

# 4. 验证 shim 链
mokutil --list-enrolled | grep "AIRYMAXOS"
```

### 4.5 TPM 2.0 度量

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

## 5. kickstart 自动安装配置

### 5.1 最小安装 kickstart（Btrfs 子卷方案）

```kickstart
# /usr/lib/airymaxos/installer/airymaxos-minimal.ks
# agentrt-linux (AirymaxOS) 最小安装 kickstart 配置

install
cdrom

lang en_US.UTF-8
keyboard us
timezone Asia/Shanghai --isUtc

network --bootproto=dhcp --device=eth0 --activate
network --hostname=agentrt-server

rootpw --iscrypted $6$rounds=5000$saltsalt$hashhashhash
user --name=agentrt --password=$6$rounds=5000$saltsalt$hashhashhash \
    --groups=wheel --shell=/bin/bash

# 分区方案：Btrfs 子卷
clearpart --all --initlabel --disklabel=gpt
part /boot/efi --fstype=efi --size=200
part /boot --fstype=xfs --size=1024
part btrfs.01 --fstype=btrfs --size=1 --grow

btrfs none --label=airymaxos btrfs.01
btrfs / --subvol --name=@ btrfs.01
btrfs /home --subvol --name=@home btrfs.01
btrfs /var --subvol --name=@var btrfs.01
btrfs /opt --subvol --name=@opt btrfs.01
btrfs /.snapshots --subvol --name=@snapshots btrfs.01

bootloader --location=mbr --boot-drive=sda \
    --append="sched_ext.enable=1 agentrt.memoryrovol=1"

%packages --nobase
@core
@base
airymaxos-kernel-core
airymaxos-kernel-modules
airymaxos-system-init
%end

%post --log=/var/log/airymaxos-install.log
systemctl enable airymaxos-kernel-load-bpf.service
btrfs subvolume snapshot / /.snapshots/post-install/$(date +%Y%m%d)
%end

reboot
```

### 5.2 完整安装 kickstart（LVM 方案）

```kickstart
# /usr/lib/airymaxos/installer/airymaxos-full.ks
# agentrt-linux (AirymaxOS) 完整安装 kickstart 配置

install
cdrom

lang zh_CN.UTF-8
keyboard us
timezone Asia/Shanghai --isUtc

network --bootproto=dhcp --device=eth0 --activate
network --hostname=agentrt-workstation

rootpw --iscrypted $6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
user --name=airymax --password=$6$yyyyyyyyyyyy --groups=wheel,airymaxos

# 分区方案：LVM + PMEM 预留
clearpart --all --initlabel --disklabel=gpt
part /boot/efi --fstype=efi --size=600
part /boot --fstype=ext4 --size=1024
part pv.01 --size=1 --grow

volgroup airymaxos-vg --pesize=4096 pv.01
logvol / --vgname=airymaxos-vg --name=root --fstype=xfs --size=102400
logvol /home --vgname=airymaxos-vg --name=home --fstype=xfs --size=51200
logvol /var --vgname=airymaxos-vg --name=var --fstype=xfs --size=204800
logvol /var/lib/airymaxos --vgname=airymaxos-vg --name=airymaxos --fstype=xfs --size=102400
logvol swap --vgname=airymaxos-vg --name=swap --fstype=swap --size=16384

bootloader --location=mbr --append="crashkernel=auto rd.agentrt.sched_ext=1 rd.agentrt.memoryrovol=1"

%packages
@airymaxos-base
@airymaxos-kernel
@airymaxos-services
@airymaxos-cognition
@airymaxos-memory
@airymaxos-security
@airymaxos-system
airymaxos-kernel-core
airymaxos-kernel-modules
airymaxos-kernel-tools
airymaxos-cognition-core
airymaxos-memory-core
airymaxos-security-cupolas
airymaxos-sdk-python
airymaxos-sdk-rust
airymaxos-sdk-go
airymaxos-sdk-java
# CJK 字体与输入法
airymaxos-fonts-cjk-zh-cn
airymaxos-fonts-cjk-ja
ibus
ibus-libpinyin
-exclude
-kernel-debug
%end

%post --log=/var/log/airymaxos-install.log

# 启用 SCHED_AGENT BPF 调度器
systemctl enable airymaxos-kernel-load-bpf.service

# 启用 12 个 daemon
for svc in gateway_d llm_d tool_d market_d sched_d monit_d \
           channel_d observe_d notify_d info_d hook_d plugin_d; do
    systemctl enable "$svc.service"
done

# 配置默认 locale 与时区
localectl set-locale LANG=zh_CN.UTF-8
timedatectl set-timezone Asia/Shanghai

# 启用 MemoryRovol PMEM 挂载点
mkdir -p /var/lib/airymaxos/memoryrovol/pmem
if [ -e /dev/pmem0 ]; then
    mkfs.xfs -m dax=inode -f /dev/pmem0
    echo "/dev/pmem0 /var/lib/airymaxos/memoryrovol/pmem xfs dax 0 0" >> /etc/fstab
fi

# 启用 MGLRU 多代 LRU
echo "y" > /sys/kernel/mm/lru_gen/enabled
echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms

# 配置大页
echo "vm.nr_hugepages = 1024" > /etc/sysctl.d/99-airymaxos-hugepage.conf
echo "hugetlbfs /var/lib/airymaxos/hugepages hugetlbfs mode=1770,gid=1000 0 0" >> /etc/fstab
mkdir -p /var/lib/airymaxos/hugepages

%end

reboot
```

### 5.3 PMEM 专用 kickstart 片段

当系统配有 PMEM 设备时，kickstart 须包含 PMEM 分区配置：

```bash
# PMEM 分区配置片段
%post

PMEM_DEV=$(ndctl list -R | jq -r '.[0].dev' | sed 's/namespace/pmem/')

if [ -n "$PMEM_DEV" ]; then
    mkfs.xfs -m dax=indev /dev/${PMEM_DEV}p1
    mkdir -p /var/lib/agentrt/memory_rovol/L1_raw
    echo "/dev/${PMEM_DEV}p1 /var/lib/agentrt/memory_rovol/L1_raw xfs dax 0 0" \
        >> /etc/fstab
    mount /var/lib/agentrt/memory_rovol/L1_raw
    xfs_io -c "statx -v" /var/lib/agentrt/memory_rovol/L1_raw \
        | grep STATX_ATTR_DAX
fi
%end
```

---

## 6. 安装流程

### 6.1 安装流程图

```
┌───────────────────────────────────────────────────────────────┐
│ 1. 引导加载（BIOS/UEFI）                                       │
│    └→ 加载 airymaxos-installer kernel + initramfs             │
├───────────────────────────────────────────────────────────────┤
│ 2. Anaconda 启动                                              │
│    └→ 读取 kickstart 配置（无人值守）/ 进入 TUI（交互式）      │
├───────────────────────────────────────────────────────────────┤
│ 3. 硬件检测                                                   │
│    └→ CPU / 内存 / 磁盘 / NIC / NUMA / CXL / PMEM 检测        │
├───────────────────────────────────────────────────────────────┤
│ 4. 选择安装模式                                               │
│    └→ 最小 / 完整 / 自定义                                    │
├───────────────────────────────────────────────────────────────┤
│ 5. 分区与文件系统                                             │
│    └→ 创建 Btrfs 子卷 / LVM + PMEM DAX 挂载点                │
├───────────────────────────────────────────────────────────────┤
│ 6. RPM 包安装                                                │
│    └→ dnf install --installroot=/mnt/sysimage airymaxos-*     │
├───────────────────────────────────────────────────────────────┤
│ 7. 引导加载器安装                                             │
│    └→ grub2-install + grub2-mkconfig + Secure Boot 签名       │
├───────────────────────────────────────────────────────────────┤
│ 8. 后置配置                                                   │
│    └→ locale + 时区 + 用户 + 网络 + SCHED_AGENT BPF + 快照    │
├───────────────────────────────────────────────────────────────┤
│ 9. 重启进入系统                                              │
│    └→ 加载 airymaxos-kernel + 启动 systemd + 启动 12 daemon   │
└───────────────────────────────────────────────────────────────┘
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
```

---

## 7. ISO 与 QCOW2 镜像构建

### 7.1 ISO 构建脚本

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/build_iso.sh
# 构建 agentrt-linux 安装 ISO 镜像

set -euo pipefail
VERSION=${1:-1.0.1}
ARCH=${2:-x86_64}
WORK_DIR=/var/tmp/airymaxos-iso
ISO_NAME=airymaxos-${VERSION}-${ARCH}.iso

echo "[1/6] 准备工作目录"
rm -rf "$WORK_DIR"
mkdir -p "$WORK_DIR"/{LiveOS,isolinux,images,Packages,repodata,EFI/BOOT}

echo "[2/6] 复制内核与 initramfs"
cp /boot/vmlinuz-6.6.0-agentrt "$WORK_DIR/isolinux/vmlinuz"
cp /boot/initramfs-6.6.0-agentrt.img "$WORK_DIR/isolinux/initrd.img"

echo "[3/6] 复制 kickstart 与 GRUB EFI"
cp /usr/lib/airymaxos/installer/airymaxos-default.ks "$WORK_DIR/isolinux/ks.cfg"
cp /boot/efi/EFI/airymaxos/grubx64.efi "$WORK_DIR/EFI/BOOT/BOOTX64.EFI"

echo "[4/6] 创建 isolinux 配置"
cat > "$WORK_DIR/isolinux/isolinux.cfg" <<EOF
DEFAULT airymaxos
PROMPT 0
TIMEOUT 30

LABEL airymaxos
    MENU LABEL agentrt-linux (AirymaxOS) ${VERSION}
    KERNEL /isolinux/vmlinuz
    APPEND initrd=/isolinux/initrd.img \
        inst.stage2=hd:LABEL=AIRYMAXOS \
        inst.ks=hd:LABEL=AIRYMAXOS:/isolinux/ks.cfg \
        sched_ext.enable=1 agentrt.memoryrovol=1
EOF

echo "[5/6] 复制 RPM 包并生成仓库"
find /root/rpmbuild/RPMS -name "*.rpm" -exec cp {} "$WORK_DIR/Packages/" \;
createrepo_c "$WORK_DIR/Packages/"
cp /etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos "$WORK_DIR/"

echo "[6/6] 生成 ISO 镜像"
mkisofs -o "$ISO_NAME" \
    -b isolinux/isolinux.bin \
    -c isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    -eltorito-alt-boot \
    -e EFI/BOOT/BOOTX64.EFI \
    -no-emul-boot \
    -J -R -V AIRYMAXOS -T \
    "$WORK_DIR/"

implantisomd5 "$ISO_NAME"
sha256sum "$ISO_NAME" > "${ISO_NAME}.sha256"

# GPG 签名 ISO
gpg --detach-sign --armor -u "iso-sign@airymaxos.dev" "$ISO_NAME"

echo "[完成] ISO 镜像已生成: $ISO_NAME"
```

### 7.2 QCOW2 镜像构建

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/build_qcow2.sh
# 构建 agentrt-linux 虚拟机 QCOW2 镜像

set -euo pipefail
VERSION=${1:-1.0.1}
ARCH=${2:-x86_64}
IMG_NAME=airymaxos-${VERSION}-${ARCH}.qcow2
SIZE=20G

echo "[1/4] 创建 QCOW2 镜像（${SIZE}）"
qemu-img create -f qcow2 "$IMG_NAME" "$SIZE"

echo "[2/4] 启动虚拟机执行 kickstart 安装"
qemu-system-${ARCH} \
    -m 4096 -smp 4 \
    -drive file="$IMG_NAME",format=qcow2,if=virtio \
    -cdrom airymaxos-${VERSION}-${ARCH}.iso \
    -kernel /boot/vmlinuz-6.6.0-agentrt \
    -initrd /boot/initramfs-6.6.0-agentrt.img \
    -append "inst.ks=file:/isolinux/ks.cfg \
             console=ttyS0 \
             sched_ext.enable=1" \
    -nographic \
    -serial mon:stdio

echo "[3/4] 安装 cloud-init（用于云镜像定制）"
virt-customize -a "$IMG_NAME" --install cloud-init

echo "[4/4] 压缩镜像"
qemu-img convert -O qcow2 -c "$IMG_NAME" "${IMG_NAME}.compressed"
mv "${IMG_NAME}.compressed" "$IMG_NAME"

echo "[完成] QCOW2 镜像已生成: $IMG_NAME"
qemu-img info "$IMG_NAME"
```

---

## 8. cloud-init 云镜像定制

### 8.1 默认 cloud-init 配置

```yaml
# /etc/cloud/cloud.cfg.d/99-airymaxos.cfg
# agentrt-linux cloud-init 配置

system_info:
  default_user:
    name: airymax
    groups: [wheel, airymaxos]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash

cloud_init_modules:
  - seed_random
  - bootcmd
  - write_files
  - growpart
  - resizefs
  - disk_setup
  - mounts
  - set_hostname
  - update_hostname
  - update_etc_hosts
  - rsyslog
  - users_groups
  - ssh

cloud_config_modules:
  - locale
  - set_passwords
  - ssh-import-id
  - package_update_upgrade_install
  - timezone
  - runcmd

locale: zh_CN.UTF-8
timezone: Asia/Shanghai

packages:
  - airymaxos-services-gateway
  - airymaxos-services-llm
  - airymaxos-services-tool
  - airymaxos-cognition-core

runcmd:
  - systemctl enable airymaxos-kernel-load-bpf.service
  - systemctl enable gateway_d.service
  - systemctl enable llm_d.service
  - systemctl enable tool_d.service
  - |
    echo "y" > /sys/kernel/mm/lru_gen/enabled
    echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms
```

### 8.2 用户自定义 cloud-init 配置

```yaml
# user-data 示例（用户上传到云平台）
#cloud-config
hostname: agentrt-node-01
fqdn: agentrt-node-01.example.com

users:
  - name: airymax
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: wheel,airymaxos
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAA... user@example.com

write_files:
  - path: /etc/airymaxos/services/llm_d.conf
    content: |
      [llm]
      provider = openai
      api_key = sk-xxxxxxxxxxxx
      locale = zh_CN.UTF-8

runcmd:
  - echo "agentrt-linux (AirymaxOS) cloud-init 完成"
  - systemctl restart llm_d.service
```

---

## 9. 安装测试

### 9.1 测试矩阵

| 测试场景 | 安装模式 | 架构 | 文件系统 | Secure Boot | TPM | 通过标准 |
|---------|----------|------|----------|-------------|-----|----------|
| 最小安装 | Minimal | x86_64 | Btrfs | 关 | 无 | 引导成功 |
| 最小安装 | Minimal | aarch64 | Btrfs | 关 | 无 | 引导成功 |
| 完整安装 | Full | x86_64 | LVM | 开 | 2.0 | 引导 + 服务启动 |
| 完整安装 | Full | aarch64 | Btrfs | 开 | 2.0 | 引导 + 服务启动 |
| PMEM 安装 | Full | x86_64 | Btrfs + DAX | 开 | 2.0 | MemoryRovol L1 启用 |
| kickstart | Minimal | x86_64 | Btrfs | 关 | 无 | 无人值守完成 |
| QCOW2 部署 | Minimal | x86_64 | Btrfs | 关 | 无 | cloud-init 完成 |

### 9.2 自动化安装测试

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

## 10. 错误处理

### 10.1 安装失败处理

| 错误场景 | 处理方式 |
|---------|---------|
| 磁盘空间不足 | 提示用户调整分区 |
| 网络不可用 | 切换到离线安装（使用 ISO 内置包） |
| Secure Boot 校验失败 | 提示用户禁用 Secure Boot 或注册密钥 |
| 包签名验证失败 | 中止安装，提示仓库损坏 |
| GRUB 安装失败 | 回滚分区变更，提示用户手动安装 |
| PMEM 配置失败 | 跳过 PMEM 挂载，继续安装（降级模式） |

### 10.2 错误码定义

安装器子系统错误码遵循 ErrorCodeSystem，定义于 [SC] 共享契约层：

```c
#define AIRY_INST_EDISK      (-940)  /* 磁盘错误 */
#define AIRY_INST_EPART      (-941)  /* 分区错误 */
#define AIRY_INST_EBOOT      (-942)  /* 引导加载器错误 */
#define AIRY_INST_ENET       (-943)  /* 网络错误 */
#define AIRY_INST_EPKG      (-944)  /* 包安装错误 */
#define AIRY_INST_ESIGN     (-945)  /* 签名校验失败 */
#define AIRY_INST_ESECURE   (-946)  /* Secure Boot 错误 */
#define AIRY_INST_ETPM      (-947)  /* TPM 错误 */
#define AIRY_INST_EPMEM     (-948)  /* PMEM 配置错误 */
#define AIRY_INST_EKICKSTART (-949) /* kickstart 解析错误 */
```

### 10.3 集中错误处理

安装器代码采用 `goto out_free_xxx` 集中错误处理（OS-KER-003/004）：

```c
int airy_installer_run(struct install_config *cfg)
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

## 11. 五维原则映射

| 原则 | 在安装器设计中的体现 |
|------|---------------------|
| **S-4 涌现性管理** | 三种安装模式 + 五种部署场景管理用户场景涌现性 |
| **C-2 增量演化** | kickstart 模板按版本演进 |
| **E-1 安全内生** | Secure Boot + GPG 签名 + TPM 度量默认启用 |
| **E-3 资源确定性** | PMEM/CXL/大页在安装期即配置完成 |
| **E-6 错误可追溯** | 安装日志全程记录 + 分级错误码 |
| **E-7 文档即代码** | kickstart 配置即安装契约文档 |
| **E-8 可测试性** | 自动化无人值守安装 + 测试矩阵 |
| **K-2 接口契约化** | kickstart 是安装接口契约 |
| **IRON-9 v2 [IND]** | 安装器独立于 agentrt |

---

## 12. IRON-9 v2 同源映射

| 组件 | agentrt-linux（[IND]） | agentrt（[IND]） |
|------|--------------------------|------------------|
| 安装器 | Anaconda + kickstart | 无（用户态运行时） |
| 引导加载器 | GRUB2 / systemd-boot | 无 |
| cloud-init | 完整支持 | 无 |
| ISO/QCOW2 | 完整支持 | 无 |
| Secure Boot | 完整支持 | 无 |
| TPM 度量 | 完整支持 | 无 |

agentrt-linux 的安装器与 agentrt 完全独立（[IND] 完全独立层），各自演进。

---

## 13. 相关文档

- `190-distribution/README.md`（发行版管理主索引）
- `190-distribution/01-rpm-packaging.md`（RPM 打包设计）
- `190-distribution/02-dnf-repo-design.md`（dnf 仓库设计）
- `190-distribution/04-update-mechanism.md`（更新机制设计）
- `100-operations/README.md`（部署运维）
- `110-security/README.md`（Secure Boot 与 TPM）
- `170-performance/02-memory-performance.md`（PMEM/CXL 分区）

---

## 14. 参考材料

- Anaconda 安装器文档
- pykickstart 项目
- GRUB 2 手册
- UEFI Secure Boot 规范
- TPM 2.0 规范
- Btrfs 文件系统手册
- cloud-init 项目
- mkisofs / xorriso 文档
- qemu-img 工具文档
- 主流 Linux 发行版安装器实践

---

## 15. 版本演进

| 版本 | 安装模式 | 分区 | 引导 | kickstart | 备注 |
|------|---------|------|------|-----------|------|
| 0.1.1 | 设计 | — | — | — | 仅文档 |
| 1.0.1 | 三模式 | Btrfs/LVM | GRUB + SB + TPM | 完整 | 开发 |
| 1.1.0 | + Cloud | + RAID | + Measured Boot | + 模板 | 云环境 |
| 2.0.0 | + Container | + OverlayFS | + AI 辅助 | + 智能配置 | 完整 |

---

> **文档结束** | agentrt-linux（AirymaxOS）安装器设计 v1.0.1
