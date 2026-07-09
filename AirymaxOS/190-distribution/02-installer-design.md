Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）安装器设计详细规范

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）安装流程、分区方案与引导加载器配置详细设计
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-liunx（AirymaxOS）安装器设计旨在为多种部署场景提供可靠、自动化、可定制的安装能力。本设计聚焦三大目标：

1. **多场景覆盖**：支持裸机安装（ISO）、虚拟机部署（QCOW2）、容器化部署（Docker）、云镜像部署（cloud-init）
2. **kickstart 自动化 100% 覆盖**：所有安装场景均可通过 kickstart 文件自动化，无人值守完成
3. **MemoryRovol 友好的分区方案**：预留 PMEM 持久化区域、CXL 内存区域、大页挂载点，安装期即配置完成

### 1.2 适用范围

- 安装介质：ISO（livecd-iso-to-disk）、QCOW2（虚拟机镜像）、Docker 镜像、raw 镜像
- 引导加载器：GRUB2（BIOS / UEFI）、systemd-boot（UEFI）、shim + GRUB2（安全启动）
- 文件系统：XFS（默认 root）、ext4（boot）、FAT32（EFI 系统分区）、DAX（PMEM）
- 分区方案：单盘 LVM、RAID1、RAID10、CXL/PMEM 混合方案
- 部署架构：x86_64（BIOS + UEFI）、aarch64（UEFI）

### 1.3 术语规范

本设计严格遵守 agentrt-liunx 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-liunx（OS 发行版）称为**微内核**（micro-kernel）。所有外部 Linux 发行版统一表述为"主流 Linux 发行版"，禁止使用 openEuler/Euler 字样。安装器与 agentrt 之间属于 IRON-9 v2 [IND] 完全独立层。

### 1.4 安装场景矩阵

| 场景 | 介质 | 自动化方式 | 适用 | 优先级 |
|------|------|------------|------|--------|
| 裸机安装 | ISO | kickstart | 物理服务器 | P0 |
| 虚拟机部署 | QCOW2 | cloud-init | KVM / VMware | P0 |
| 容器化部署 | Docker 镜像 | Dockerfile | K8s / Podman | P0 |
| 云镜像部署 | raw 镜像 | cloud-init | AWS / 阿里云 | P1 |
| PXE 网络安装 | PXE + kickstart | 大规模部署 | 数据中心 | P1 |

---

## 2. 安装流程总体设计

### 2.1 安装流程图

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
│ 4. 分区与文件系统                                             │
│    └→ 创建 LVM 分区 + XFS + PMEM DAX 挂载点                   │
├───────────────────────────────────────────────────────────────┤
│ 5. RPM 包安装                                                │
│    └→ dnf install --installroot=/mnt/sysimage airymaxos-*     │
├───────────────────────────────────────────────────────────────┤
│ 6. 引导加载器安装                                             │
│    └→ grub2-install + grub2-mkconfig                          │
├───────────────────────────────────────────────────────────────┤
│ 7. 后置配置                                                   │
│    └→ locale + 时区 + 用户 + 网络 + SCHED_AGENT BPF           │
├───────────────────────────────────────────────────────────────┤
│ 8. 重启进入系统                                              │
│    └→ 加载 airymaxos-kernel + 启动 systemd + 启动 12 daemon   │
└───────────────────────────────────────────────────────────────┘
```

### 2.2 安装器组件

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

## 3. 分区方案

### 3.1 默认分区方案（裸机，1TB 磁盘）

```bash
# 默认分区方案（基于 LVM + XFS）
# 适用：1TB NVMe SSD，4GB+ PMEM（可选）

# 分区表（GPT）
Disk /dev/nvme0n1: 1.00 TiB

# 分区 1：EFI 系统分区（UEFI）/ boot 分区（BIOS）
/dev/nvme0n1p1  600 MB   FAT32   /boot/efi  (UEFI)  或 /boot (BIOS)

# 分区 2：boot 分区
/dev/nvme0n1p2  1 GB     ext4    /boot

# 分区 3：LVM 物理卷
/dev/nvme0n1p3  rest     LVM PV  → airymaxos-vg

# LVM 逻辑卷
airymaxos-vg/root     100 GB   XFS    /
airymaxos-vg/home     50 GB    XFS    /home
airymaxos-vg/var      200 GB   XFS    /var
airymaxos-vg/opt      100 GB   XFS    /opt
airymaxos-vg/airymaxos  100 GB XFS    /var/lib/airymaxos   # 记忆卷载数据
airymaxos-vg/swap      16 GB    swap   [SWAP]
airymaxos-vg/log       20 GB    XFS    /var/log
airymaxos-vg/tmp       20 GB    XFS    /tmp
airymaxos-vg/free      rest     (空闲)

# PMEM 设备（如果存在）
/dev/pmem0  32 GB   XFS (DAX)   /var/lib/airymaxos/memoryrovol/pmem
```

### 3.2 PMEM 与 CXL 自动检测分区

```bash
# /usr/lib/airymaxos/installer/scripts/detect_pmem.sh
# 检测 PMEM 设备并配置 DAX 挂载

#!/bin/bash
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

### 3.3 大页（hugepage）分区配置

```bash
# /etc/airymaxos/sysctl.d/99-hugepage.conf
# 为 IPC 零拷贝区域预留 2MB 大页

# 2MB 大页数量（按内存大小自动计算）
vm.nr_hugepages = 1024

# 启用透明大页
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# 大页挂载点
hugetlbfs /var/lib/airymaxos/hugepages hugetlbfs mode=1770,gid=1000 0 0
```

---

## 4. kickstart 自动安装配置

### 4.1 完整 kickstart 文件示例

```kickstart
# /usr/lib/airymaxos/installer/airymaxos-default.ks
# agentrt-liunx (AirymaxOS) 默认 kickstart 配置
# Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# === 基本设置 ===
# 安装类型（最小化 / 服务器 / 工作站）
text
install
cdrom

# 键盘与语言
keyboard us
lang zh_CN.UTF-8

# 时区
timezone Asia/Shanghai --isUtc

# === 网络配置 ===
# DHCP 自动获取
network --bootproto=dhcp --device=eth0 --activate
# 静态 IP（可选）
# network --bootproto=static --device=eth0 \
#         --ip=192.168.1.100 --netmask=255.255.255.0 \
#         --gateway=192.168.1.1 --nameserver=8.8.8.8

# === 用户与密码 ===
# root 密码（哈希值，使用 openssl passwd -6 生成）
rootpw --iscrypted $6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 普通用户
user --name=airymax --password=$6$yyyyyyyyyyyy --groups=wheel,airymaxos

# === 分区方案 ===
# 清空磁盘并创建 GPT 分区表
clearpart --all --initlabel --disklabel=gpt

# 引导加载器
bootloader --location=mbr --append="crashkernel=auto rd.agentrt.sched_ext=1 rd.agentrt.memoryrovol=1"

# 自动分区（LVM 方案）
autopart --type=lvm --fstype=xfs --nohome

# 或者手动分区（更精细控制）
# part /boot/efi --fstype=efi --size=600
# part /boot --fstype=ext4 --size=1024
# part pv.01 --size=1 --grow
# volgroup airymaxos-vg --pesize=4096 pv.01
# logvol / --vgname=airymaxos-vg --name=root --fstype=xfs --size=102400
# logvol /var --vgname=airymaxos-vg --name=var --fstype=xfs --size=204800
# logvol /var/lib/airymaxos --vgname=airymaxos-vg --name=airymaxos --fstype=xfs --size=102400
# logvol swap --vgname=airymaxos-vg --name=swap --fstype=swap --size=16384

# === 软件包选择 ===
# 安装 agentrt-liunx 核心包组
%packages
@airymaxos-base
@airymaxos-kernel
@airymaxos-services
@airymaxos-cognition
@airymaxos-memory
@airymaxos-security
@airymaxos-system

# 显式包含的包
airymaxos-kernel-core
airymaxos-kernel-modules
airymaxos-kernel-tools
airymaxos-services-gateway
airymaxos-services-llm
airymaxos-services-tool
airymaxos-cognition-core
airymaxos-memory-core
airymaxos-security-cupolas
airymaxos-system-init

# CJK 字体
airymaxos-fonts-cjk-zh-cn
airymaxos-fonts-cjk-ja
airymaxos-fonts-cjk-ko

# IBus 输入法
ibus
ibus-libpinyin
ibus-anthy
ibus-hangul

# 排除的包
-exclude
-kernel-debug
-kernel-debug-devel
%end

# === 安装后脚本 ===
%post --log=/var/log/airymaxos-install.log

# 启用 SCHED_AGENT BPF 调度器
systemctl enable airymaxos-kernel-load-bpf.service

# 启用 12 个 daemon
for svc in gateway_d llm_d tool_d market_d sched_d monit_d \
           channel_d observe_d notify_d info_d hook_d plugin_d; do
    systemctl enable "$svc.service"
done

# 配置默认 locale
localectl set-locale LANG=zh_CN.UTF-8
localectl set-keymap us

# 配置时区
timedatectl set-timezone Asia/Shanghai

# 启用 MemoryRovol PMEM 挂载点
mkdir -p /var/lib/airymaxos/memoryrovol/pmem
if [ -e /dev/pmem0 ]; then
    mkfs.xfs -m dax=inode -f /dev/pmem0
    echo "/dev/pmem0 /var/lib/airymaxos/memoryrovol/pmem xfs dax 0 0" >> /etc/fstab
fi

# 启用 MGLRU
echo "y" > /sys/kernel/mm/lru_gen/enabled
echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms

# 配置大页
echo "vm.nr_hugepages = 1024" > /etc/sysctl.d/99-airymaxos-hugepage.conf
echo "hugetlbfs /var/lib/airymaxos/hugepages hugetlbfs mode=1770,gid=1000 0 0" >> /etc/fstab
mkdir -p /var/lib/airymaxos/hugepages

# 配置 GRUB 内核参数
GRUB_CMDLINE="crashkernel=auto sched_ext.enable=1 \
    agentrt.memoryrovol=1 agentrt.cxl=1 agentrt.mglru=1 \
    transparent_hugepage=always"
sed -i "s/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX=\"$GRUB_CMDLINE\"/" \
    /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg

%end

# === 安装完成动作 ===
reboot
```

### 4.2 最小化安装 kickstart

```kickstart
# /usr/lib/airymaxos/installer/airymaxos-minimal.ks
# agentrt-liunx 最小化安装（仅内核 + 基础服务）

text
install
cdrom

keyboard us
lang en_US.UTF-8
timezone UTC --isUtc

network --bootproto=dhcp --device=eth0 --activate
rootpw --iscrypted $6$minimalpasswordhash

clearpart --all --initlabel --disklabel=gpt
autopart --type=lvm --fstype=xfs --nohome

bootloader --location=mbr --append="rd.agentrt.sched_ext=1"

%packages --nobase
@airymaxos-base
airymaxos-kernel-core
airymaxos-kernel-modules
airymaxos-system-init
%end

%post --log=/var/log/airymaxos-install.log
systemctl enable airymaxos-kernel-load-bpf.service
%end

reboot
```

---

## 5. 引导加载器配置

### 5.1 GRUB2 配置（UEFI）

```bash
# /etc/default/grub —— agentrt-liunx GRUB2 默认配置

GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="agentrt-liunx (AirymaxOS)"
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

# 安全启动配置
GRUB_ENABLE_CRYPTODISK=y
```

### 5.2 GRUB2 菜单条目

```bash
# /boot/grub2/grub.cfg（自动生成）
# 主菜单条目示例

menuentry "agentrt-liunx (AirymaxOS) 1.0.1 (Linux 6.6.0-agentrt)" {
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

menuentry "agentrt-liunx (AirymaxOS) 1.0.1 (recovery mode)" {
    set root=(hd0,gpt2)
    linux /boot/vmlinuz-6.6.0-agentrt \
        root=/dev/mapper/airymaxos--vg-root \
        ro single systemd.unit=rescue.target
    initrd /boot/initramfs-6.6.0-agentrt.img
}
```

### 5.3 安全启动配置

```bash
# 安全启动（Secure Boot）配置

# 1. 安装 shim + MOK（Machine Owner Key）
dnf install shim mokutil

# 2. 注册 SPHARX MOK 公钥
mokutil --import /etc/pki/airymaxos/Spharx-MOK.crt

# 3. 验证 shim 链
mokutil --list-enrolled | grep "SPHARX"

# 4. 签名内核镜像
sbsign --key /etc/pki/airymaxos/Spharx-MOK.key \
       --cert /etc/pki/airymaxos/Spharx-MOK.crt \
       --output /boot/vmlinuz-6.6.0-agentrt.signed \
       /boot/vmlinuz-6.6.0-agentrt

# 5. 更新 GRUB 启动安全启动
echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

## 6. ISO 镜像构建

### 6.1 ISO 构建脚本

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/build_iso.sh
# 构建 agentrt-liunx 安装 ISO 镜像

set -euo pipefail
VERSION=${1:-1.0.1}
ARCH=${2:-x86_64}
WORK_DIR=/var/tmp/airymaxos-iso
ISO_NAME=airymaxos-${VERSION}-${ARCH}.iso

echo "[1/6] 准备工作目录"
rm -rf "$WORK_DIR"
mkdir -p "$WORK_DIR"/{LiveOS,isolinux,images,Packages,repodata}

echo "[2/6] 复制内核与 initramfs"
cp /boot/vmlinuz-6.6.0-agentrt "$WORK_DIR/isolinux/vmlinuz"
cp /boot/initramfs-6.6.0-agentrt.img "$WORK_DIR/isolinux/initrd.img"

echo "[3/6] 复制 kickstart 配置"
cp /usr/lib/airymaxos/installer/airymaxos-default.ks \
   "$WORK_DIR/isolinux/ks.cfg"

echo "[4/6] 创建 isolinux 配置"
cat > "$WORK_DIR/isolinux/isolinux.cfg" <<EOF
DEFAULT airymaxos
PROMPT 0
TIMEOUT 30

LABEL airymaxos
    MENU LABEL agentrt-liunx (AirymaxOS) ${VERSION}
    KERNEL /isolinux/vmlinuz
    APPEND initrd=/isolinux/initrd.img \
        inst.stage2=hd:LABEL=AIRYMAXOS \
        inst.ks=hd:LABEL=AIRYMAXOS:/isolinux/ks.cfg \
        sched_ext.enable=1 agentrt.memoryrovol=1

LABEL airymaxos-rescue
    MENU LABEL agentrt-liunx Rescue Mode
    KERNEL /isolinux/vmlinuz
    APPEND initrd=/isolinux/initrd.img \
        inst.stage2=hd:LABEL=AIRYMAXOS rescue
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
    -J -R -V AIRYMAXOS \
    -T \
    "$WORK_DIR/"

# 嵌入 MD5 校验
implantisomd5 "$ISO_NAME"

echo "[完成] ISO 镜像已生成: $ISO_NAME"
sha256sum "$ISO_NAME"
```

### 6.2 QCOW2 镜像构建

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/build_qcow2.sh
# 构建 agentrt-liunx 虚拟机 QCOW2 镜像

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

## 7. cloud-init 配置

### 7.1 cloud-init 配置文件

```yaml
# /etc/cloud/cloud.cfg.d/99-airymaxos.cfg
# agentrt-liunx cloud-init 配置

# 默认用户
system_info:
  default_user:
    name: airymax
    groups: [wheel, airymaxos]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash

# 启用的模块
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

# 默认 locale
locale: zh_CN.UTF-8
timezone: Asia/Shanghai

# 默认安装的包
packages:
  - airymaxos-services-gateway
  - airymaxos-services-llm
  - airymaxos-services-tool
  - airymaxos-cognition-core

# 启用服务
runcmd:
  - systemctl enable airymaxos-kernel-load-bpf.service
  - systemctl enable gateway_d.service
  - systemctl enable llm_d.service
  - systemctl enable tool_d.service
  - |
    # 配置 MGLRU
    echo "y" > /sys/kernel/mm/lru_gen/enabled
    echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms
```

### 7.2 用户自定义 cloud-init 配置

```yaml
# user-data 示例（用户上传到云平台）
#cloud-config
hostname: agentrt-node-01
fqdn: agentrt-node-01.example.com

users:
  - name: airymax
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: wheel, airymaxos
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
  - echo "agentrt-liunx (AirymaxOS) cloud-init 完成"
  - systemctl restart llm_d.service
```

---

## 8. 错误码体系对接

安装器错误码纳入 agentrt-liunx 统一错误码体系：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AGENTRT_E_INSTALL_DISK | -310 | 磁盘分区错误 |
| AGENTRT_E_INSTALL_PKG | -311 | 包安装失败 |
| AGENTRT_E_INSTALL_BOOTLOADER | -312 | 引导加载器安装失败 |
| AGENTRT_E_INSTALL_KICKSTART | -313 | kickstart 解析错误 |
| AGENTRT_E_INSTALL_NETWORK | -314 | 网络配置失败 |
| AGENTRT_E_INSTALL_PMEM | -315 | PMEM 配置失败 |

集中错误处理示例：

```c
int airymax_install_run(const char *ks_path)
{
	int ret;

	if (!ks_path) {
		ret = -EINVAL;
		goto out_err;
	}

	ret = airymax_install_parse_kickstart(ks_path);
	if (ret < 0) {
		ret = -AGENTRT_E_INSTALL_KICKSTART;
		goto out_err;
	}

	ret = airymax_install_partition_disk();
	if (ret < 0) {
		ret = -AGENTRT_E_INSTALL_DISK;
		goto out_err;
	}

	ret = airymax_install_packages();
	if (ret < 0) {
		ret = -AGENTRT_E_INSTALL_PKG;
		goto out_err;
	}

	ret = airymax_install_bootloader();
	if (ret < 0) {
		ret = -AGENTRT_E_INSTALL_BOOTLOADER;
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
| **S-4 涌现性管理** | 多场景安装方案管理发行版涌现性 |
| **C-2 增量演化** | kickstart 模板按版本演进 |
| **E-1 安全内生** | 安全启动 + GPG 签名默认启用 |
| **E-3 资源确定性** | PMEM/CXL/大页在安装期即配置 |
| **E-7 文档即代码** | kickstart 配置即文档 |
| **E-8 可测试性** | 自动化无人值守安装 |

---

## 10. IRON-9 v2 同源映射

| 组件 | agentrt-liunx（[IND]） | agentrt（[IND]） |
|------|--------------------------|------------------|
| 安装器 | Anaconda + kickstart | 无（用户态运行时） |
| 引导加载器 | GRUB2 / systemd-boot | 无 |
| cloud-init | 完整支持 | 无 |
| ISO/QCOW2 | 完整支持 | 无 |

agentrt-liunx 的安装器与 agentrt 完全独立（[IND] 完全独立层），各自演进。

---

## 11. 相关文档

- `190-distribution/README.md`（发行版管理主索引）
- `190-distribution/01-rpm-packaging.md`（RPM 打包设计）
- `190-distribution/03-update-mechanism.md`（更新机制设计）
- `100-operations/README.md`（部署运维）
- `170-performance/02-memory-performance.md`（PMEM/CXL 配置）

---

## 12. 参考材料

- Anaconda 安装器文档
- pykickstart 项目
- GRUB2 文档
- cloud-init 项目
- mkisofs / xorriso 文档
- qemu-img 工具文档
- 主流 Linux 发行版安装规范

---

> **文档结束** | agentrt-liunx（AirymaxOS）安装器设计详细规范 | 1.0.1 开发版本
