# Raspberry Pi 5 — NFS + TFTP 开发调试指南

## 概述

传统开发方式是每次 `make` 后用 `dd` 烧录整个 `sdcard.img` 到 SD 卡。这在调试阶段效率极低（写入慢、磨损 SD 卡、无法快速迭代）。本文档介绍 NFS rootfs + TFTP 内核的远程启动方案，实现：

```
make 编译 → 复制内核到 /tftpboot → 复制 rootfs 到 /nfsroot → 板子重启 = 生效
```

SD 卡只烧一次，之后**完全不需要碰它**。

## 整体架构

```
┌─────────────────────────┐        ┌─────────────────────┐
│     开发机 (PC)          │        │   Raspberry Pi 5    │
│                         │        │                     │
│  TFTP 服务器 ─────────────┼──────── →  加载内核 Image    │
│  192.168.2.1            │ TFTP   │  192.168.2.x (DHCP) │
│                         │        │                     │
│  NFS 服务器 ──────────────┼──────── →  挂载 rootfs       │
│  /nfsroot/rpi5          │ NFS    │  / (over NFS)       │
│                         │        │                     │
│  以太网 ──────────────────┼────────── 以太网              │
└─────────────────────────┘        └─────────────────────┘
```

**前提条件**：
- 开发机和 RPi5 通过以太网连接（直连或经交换机，Pi 5 有千兆网口）
- RPi5 有一张 SD 卡（只需要放一次 boot 分区）
- Buildroot 已能正常构建 (`make raspberrypi5_defconfig && make`)

---

## 第一步：开发机安装 TFTP + NFS 服务

```bash
sudo apt update
sudo apt install -y tftpd-hpa nfs-kernel-server
```

### 配置 TFTP

```bash
# 创建 TFTP 根目录
sudo mkdir -p /tftpboot
sudo chmod 777 /tftpboot

# 编辑 /etc/default/tftpd-hpa
sudo tee /etc/default/tftpd-hpa <<'EOF'
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure --create"
EOF

sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa
```

### 配置 NFS

```bash
# 创建 NFS root 目录
sudo mkdir -p /nfsroot/rpi5
sudo chmod 777 /nfsroot/rpi5

# 导出 NFS 目录
echo "/nfsroot/rpi5 *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -ra

# 确认导出成功
sudo exportfs -v
```

### 确认开发机有线网卡 IP

```bash
ip addr show
# 找到连接 RPi5 的网口 IP，假设为 192.168.2.1
# 如果直连 Pi，建议设置静态 IP：
#   编辑 NetworkManager 或 /etc/network/interfaces
#   设置为 192.168.2.1/24
```

> 记下这个 IP，后面 NFS 和 TFTP 地址都用它。本文档以 `192.168.2.1` 为例，请替换为你的实际 IP。

---

## 第二步：准备 NFS 启动用的 cmdline

在 Buildroot 源码树中创建 NFS 专用内核命令行文件：

```bash
cat > board/raspberrypi5/cmdline_5_nfs.txt <<'EOF'
root=/dev/nfs nfsroot=192.168.2.1:/nfsroot/rpi5,vers=4,tcp rw ip=dhcp console=tty1 console=ttyAMA10,115200
EOF
```

**参数说明**：

| 参数 | 含义 |
|------|------|
| `root=/dev/nfs` | 告知内核 rootfs 来自 NFS（不需要实际的 /dev/nfs 设备） |
| `nfsroot=192.168.2.1:/nfsroot/rpi5` | NFS 服务器 IP 和导出路径 |
| `vers=4,tcp` | 使用 NFSv4 + TCP（比默认的 v3+UDP 更稳定） |
| `rw` | 可读写挂载（调试需要修改文件） |
| `ip=dhcp` | RPi5 通过 DHCP 获取 IP |
| `console=tty1` | HDMI 输出控制台 |
| `console=ttyAMA10,115200` | UART 串口控制台（GPIO 14/15, 115200bps） |

---

## 第三步：修改 Buildroot 配置

```bash
make menuconfig
```

需要调整的关键配置：

### 3.1 启用 NFS root 支持

```
System configuration  --->
  [*] remount root filesystem read-write during boot
```

### 3.2 生成 tar 格式 rootfs（方便解压到 NFS 目录）

```
Filesystem images  --->
  [*] tar the root filesystem
```

> 原有的 ext2/ext4 可以保留，不影响。

### 3.3 修改 cmdline 为 NFS 版本

```
Target packages  --->
  Hardware handling  --->
    Firmware  --->
      [*] rpi-firmware
        (board/raspberrypi5/cmdline_5_nfs.txt) Command line file
```

或者直接编辑 `.config`：

```bash
sed -i 's|BR2_PACKAGE_RPI_FIRMWARE_CMDLINE_FILE="board/raspberrypi5/cmdline_5.txt"|BR2_PACKAGE_RPI_FIRMWARE_CMDLINE_FILE="board/raspberrypi5/cmdline_5_nfs.txt"|' .config
```

### 3.4 (可选) 增大 rootfs

调试期间会多装工具，建议把 rootfs 从默认 120MB 调大：

```
Filesystem images  --->
  (512M) exact size
```

---

## 第四步：构建

```bash
# 确认配置无冲突
make olddefconfig

# 构建
make -j$(nproc)
```

构建完成后 `output/images/` 下关键文件：

```
output/images/
├── Image                          # 内核镜像
├── broadcom/
│   ├── bcm2712-rpi-5-b.dtb        # 设备树
│   ├── bcm2712d0-rpi-5-b.dtb
│   └── bcm2712-rpi-500.dtb
├── rootfs.tar                     # rootfs 归档
├── sdcard.img                     # 完整 SD 卡镜像（用于首次烧录）
└── rpi-firmware/
    ├── config.txt
    ├── cmdline.txt                 # 用的是 NFS 版
    ├── start4.elf
    ├── fixup4.dat
    └── ...
```

---

## 第五步：部署到开发机

每次 `make` 完成后执行：

```bash
#!/bin/bash
# 将此脚本保存为 deploy.sh 在 Buildroot 根目录
set -e
NFSROOT="/nfsroot/rpi5"
TFTPROOT="/tftpboot"
IMAGES="output/images"

echo "=== 部署内核到 TFTP ==="
sudo cp ${IMAGES}/Image ${TFTPROOT}/
sudo cp ${IMAGES}/broadcom/*.dtb ${TFTPROOT}/

echo "=== 部署 rootfs 到 NFS ==="
sudo rm -rf ${NFSROOT}/*
sudo tar xf ${IMAGES}/rootfs.tar -C ${NFSROOT}/
sync

echo "=== 完成 ==="
echo "重启 RPi5: ssh root@<pi-ip> reboot  或  按 reset 键"
```

```bash
chmod +x deploy.sh
./deploy.sh
```

---

## 第六步：首次烧录 SD 卡（只做一次）

SD 卡只需要一个 FAT 启动分区。可以直接用 Buildroot 生成的 `sdcard.img` 烧一次：

```bash
# 插入读卡器，确认设备名
lsblk

# 烧录（替换 sdX 为实际设备）
sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M status=progress conv=fsync
```

烧完后 SD 卡结构：

```
SD 卡
├── 分区1 (FAT32, bootable)
│   ├── config.txt           → kernel=Image
│   ├── cmdline.txt          → root=/dev/nfs ...
│   ├── start4.elf
│   ├── fixup4.dat
│   ├── Image
│   └── *.dtb
└── 分区2 (ext4)              → 不会被用到（NFS 替代了它）
```

> 之后除非需要更新 bootloader EEPROM 或修改 config.txt，不要再碰这张 SD 卡。

---

## 第七步：RPi5 上电启动

```
1. SD 卡插入 Pi
2. 以太网线连接 PC ↔ Pi
3. 开发机确保 TFTP 和 NFS 服务运行中
4. Pi 上电

启动日志 (串口/HMDI)：
  Bootloader: start4.elf → 读取 config.txt → 加载 Image
  Kernel:     通过 TFTP 下载 Image → 通过 DHCP 获取 IP → 挂载 NFS root
  Userspace:  /sbin/init (BusyBox) 启动
```

---

## 第八步：日常开发循环

```
修改源码/配置
    ↓
make -j$(nproc)       ← 在开发机上编译
    ↓
./deploy.sh           ← 复制到 TFTP + NFS
    ↓
ssh root@<pi-ip> reboot   ← 或直接按板子 reset
    ↓
验证改动
```

整个循环 **不到 2 分钟**（主要耗时在增量编译）。

---

## 故障排查

### Pi 卡在彩虹屏/无输出

- 检查 `config.txt` 中 `kernel=Image` 是否正确
- 确认 SD 卡 FAT 分区中有 `start4.elf`、`fixup4.dat`
- 检查串口线连接 (GPIO 14=TX, 15=RX, GND)，波特率 115200

### Kernel panic: VFS: Unable to mount root fs

```
原因：内核无法通过 NFS 挂载 rootfs
```
- **检查物理连接**：`ip addr` 确认开发机和 Pi 在同一网络
- **检查 NFS 服务**：`sudo exportfs -v` 确认 `/nfsroot/rpi5` 已导出
- **检查防火墙**：`sudo ufw allow 2049` (NFS) 和 `sudo ufw allow 69/udp` (TFTP)
- **检查 cmdline.txt**：`nfsroot=` 中的 IP 是否为开发机地址
- **NFS 版本**：如果 `vers=4` 失败尝试改为 `vers=3`

### TFTP timeout

```
原因：TFTP 服务未运行或被防火墙拦截
```
```bash
sudo systemctl status tftpd-hpa
sudo ufw allow 69/udp
```

### NFS 挂载只读

确认 `/etc/exports` 中有 `rw` 选项，且 Buildroot 配置了：
```
BR2_SYSTEM_REMOUNT_ROOT_RW=y
```

### 开发机 IP 变化

如果你的开发机 IP 变了（从 `192.168.2.1` 变成了其他），需要更新 `cmdline_5_nfs.txt` 中的 `nfsroot=` 地址，然后重新 `make` 并重新制作 SD 卡或只更新 SD 卡上的 `cmdline.txt`。

---

## 进阶技巧

### 不重新烧卡更新内核

SD 卡上的内核也可以走 TFTP 下载，在 `config.txt` 中加一行就行（不需要每次烧卡）：

```
# 这样甚至 SD 卡上都不需要有 Image 文件，省掉每次 dd
# 但前提是 RPi5 的 bootloader 支持 TFTP — 
# 实际上 RPi5 的 bootloader 不支持 TFTP 直接加载内核
# Image 还是要在 SD 卡 FAT 分区上
```

> **注意**：RPi5 的启动流程是 `EEPROM → start4.elf → config.txt → 加载 kernel`。`start4.elf` 阶段不支持 TFTP。所以内核仍需放在 SD 卡的 FAT 分区，或者改用 U-Boot 作为中间 bootloader（后面单开文档讲）。

### 跳过 update-alternatives 警告

Buildroot 启动时会检查 uutils `install` 兼容性。每次打开新终端也可能需要：

```bash
# 一劳永逸（如果系统常用 Buildroot）
sudo update-alternatives --install /usr/bin/install install /usr/bin/gnuinstall 100
```

### 使用 USB 转以太网适配器

如果开发机只有一个网口（连外网），可以：
- 给 Pi 加一个 USB 转以太网（Pi 5 本身已有千兆口）
- 开发机用 WiFi 连外网，有线口直连 Pi
- 或者用 DHCP 服务器给 Pi 分配 IP

---

## 参考链接

- [Buildroot 官方手册 — NFS rootfs](https://buildroot.org/manual.html#_using_live_cd_rom_or_nfs_boot)
- [RPi 内核参数文档](https://www.kernel.org/doc/html/latest/admin-guide/nfs/nfsroot.html)
- [RPi config.txt 文档](https://www.raspberrypi.com/documentation/computers/config_txt.html)
