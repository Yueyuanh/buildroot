# SD 卡烧录指南

## 概述

Buildroot 编译产物 `output/images/sdcard.img` 是一个完整的 SD 卡镜像，包含所有启动和运行所需的内容。本文覆盖从 PC 烧录到 RPi5 上电的完整操作。

## 快速操作

sudo apt install pv -y

```bash
# 1. 插卡 — 确认设备
lsblk

# 2. 卸载所有已挂载分区
sudo umount /dev/sda1

# 3. 烧录

sudo dd if=output/images/sdcard.img of=/dev/sda bs=4M status=progress conv=fsync

# 带进度条
pv output/images/sdcard.img | sudo dd of=/dev/sda bs=4M conv=fsync

# 4. 同步 + 弹出
sync
sudo eject /dev/sda
```

> ⚠️ `of=/dev/sdX` 写的是**整个设备**，不是分区。写错设备会毁掉你的系统盘。

---

## 详细步骤

### 1. 确认 SD 卡设备

插入读卡器前后各运行一次 `lsblk`，**新增的那个设备**就是 SD 卡：

```bash
# 插卡前
$ lsblk
sda      8:0    1  29.7G  0 disk
└─sda1   8:1    1  29.7G  0 part /run/media/young/XXXX
#                    ↑ 看到挂载点说明是 USB 存储设备
```

判断技巧：
- **SIZE** 接近你的卡容量（32G 卡显示 ~29.7G）
- **TYPE** 是 `disk`，下面挂着 `part`
- **MOUNTPOINTS** 有 `/run/media/...` 说明系统已自动挂载
- **`RM`** (Removable) 值为 1 表示可移动设备

也可用 `fdisk` 确认：

```bash
sudo fdisk -l /dev/sda
# Disk /dev/sda: 29.72 GiB ...
# Disk model: STORAGE DEVICE   ← 典型的 USB 读卡器标识
```

### 2. 卸载

SD 卡插入后 Linux 可能会自动挂载已有分区。烧录前**必须卸载所有分区**：

```bash
# 查看挂载
mount | grep sdX

# 逐个卸载
sudo umount /dev/sda1

# 或一键卸载所有
sudo umount /dev/sda*
```

### 3. dd 烧录

```bash
sudo dd if=output/images/sdcard.img of=/dev/sda bs=4M status=progress conv=fsync
```

| 参数 | 含义 |
|------|------|
| `if=` | 输入文件 (Buildroot 生成的镜像) |
| `of=` | 输出设备 (整个 SD 卡，不带数字) |
| `bs=4M` | 块大小 4MB，加速写入 |
| `status=progress` | 显示实时进度 |
| `conv=fsync` | 写入完成后同步元数据，确保数据真正落盘 |

输出示例：

```
$ sudo dd if=output/images/sdcard.img of=/dev/sda bs=4M status=progress conv=fsync
153092096 bytes (153 MB, 146 MiB) copied, 35 s, 4.4 MB/s
36+1 records in
36+1 records out
153092096 bytes (153 MB, 146 MiB) copied, 35.2119 s, 4.3 MB/s
```

### 4. 同步 + 安全弹出

```bash
sync                     # 确保内核缓冲区全部写入
sudo eject /dev/sda      # 安全弹出 (或直接拔卡)
```

> 不要忽略 `sync`。直接拔卡可能导致写入不完整，系统无法启动。

---

## 镜像内容

烧录后 SD 卡有两个分区：

```
/dev/sda
├── sda1: FAT32 (boot, 32MB)      ← 启动分区
│   ├── start4.elf                ← RPi5 GPU 固件
│   ├── fixup4.dat
│   ├── config.txt                ← kernel=Image
│   ├── cmdline.txt               ← root=/dev/mmcblk0p2 ...
│   ├── Image                     ← Linux 内核
│   ├── *.dtb                     ← 设备树文件
│   └── overlays/                 ← 设备树 overlay
│
└── sda2: ext4 (rootfs, ~120MB)   ← 根文件系统
    ├── bin/  -> busybox
    ├── sbin/ -> busybox
    ├── etc/
    ├── lib/
    ├── usr/
    └── ...
```

`fdisk -l` 可以验证分区：

```bash
sudo fdisk -l /dev/sda
# Device     Boot Start    End Sectors  Size Id Type
# /dev/sda1  *     2048  67583   65536   32M  c W95 FAT32 (LBA)
# /dev/sda2        67584 313343  245760  120M 83 Linux
```

---

## 跨平台工具

### Linux (推荐)

```bash
# 标准方式
sudo dd if=sdcard.img of=/dev/sdX bs=4M status=progress conv=fsync

# 或使用 bmap-tools (更快，跳过空白区域)
sudo bmaptool copy sdcard.img /dev/sdX
```

### macOS

```bash
# 先找设备
diskutil list
# /dev/disk4 (external, physical)  ← 找到外部磁盘

# 卸载
diskutil unmountDisk /dev/disk4

# 烧录 (macOS 的 dd 比较慢，用 rdisk 加速)
sudo dd if=sdcard.img of=/dev/rdisk4 bs=4m status=progress
```

### Windows

推荐使用 **Rufus** 或 **balenaEtcher**（有 GUI，不用写命令）：

1. 打开 balenaEtcher
2. 选 `sdcard.img`
3. 选 SD 卡
4. 点 Flash

---

## 验证烧录

### 校验数据完整性

```bash
# 比较写入的内容和镜像文件
sudo dd if=/dev/sda bs=4M count=37 | md5sum
dd if=output/images/sdcard.img bs=4M count=37 | md5sum
# 两个 MD5 应该一致

# 或直接用 cmp (读到第一处差异就停)
sudo cmp -n $(stat -c%s output/images/sdcard.img) /dev/sda output/images/sdcard.img
# 无输出 = 完全一致
```

### 查看分区

```bash
sudo fdisk -l /dev/sda
# 应该看到两个分区: FAT32 (bootable) + Linux (ext4)
```

### 挂载检查内容

```bash
sudo mkdir -p /mnt/sd-boot /mnt/sd-root
sudo mount /dev/sda1 /mnt/sd-boot
sudo mount /dev/sda2 /mnt/sd-root

ls /mnt/sd-boot/
# start4.elf  config.txt  Image  *.dtb ...

ls /mnt/sd-root/
# bin  etc  lib  sbin  usr  var ...

sudo umount /mnt/sd-boot /mnt/sd-root
```

---

## 常见问题

### "Permission denied" 写 SD 卡

```bash
# 确认设备路径正确
lsblk

# 部分 USB 读卡器有物理写保护开关，检查是否拨到 lock
```

### dd 写入很慢 (>10 分钟)

```bash
# 用更大的块大小
sudo dd if=sdcard.img of=/dev/sda bs=16M status=progress conv=fsync

# 或者先确认 USB 口速率
lsusb -t
# 如果是 USB 2.0 接口 (480M)，速度上限 ~40MB/s
```

### 烧录后 SD 卡容量变小

烧录的镜像是 153MB，但 SD 卡是 32GB。烧录后 SD 卡显示只用了 ~150MB，剩余空间未分配，**这是正常的**。

可以用 `gparted` 或 `fdisk` 扩容 rootfs 分区：

```bash
sudo gparted /dev/sda
# 右键 sda2 → Resize/Move → 拉到最大 → Apply
```

### 烧录后 Windows 不识别

Windows 不认识 ext4 分区，会提示"需要格式化"——**不要格式化**，那是你的 rootfs。FAT32 的 boot 分区可以正常在 Windows 中浏览。

### 烧录后 RPi5 不启动

见 [NFS 开发调试指南](./nfs-debug-guide.md) 中的故障排查部分。

---

## 开发阶段建议

SD 卡烧一次就够。之后的日常开发走 NFS + TFTP 避免反复烧卡，详见 [NFS 开发调试指南](./nfs-debug-guide.md)。
