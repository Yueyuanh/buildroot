# NFS + TFTP 开发调试指南

## 概述

开发阶段避免反复烧 SD 卡。内核通过 TFTP 下发，rootfs 通过 NFS 挂载。一次配置后，`make → deploy → reboot` 即可验证改动，全程不需要碰 SD 卡。

与 [board/raspberrypi5/nfs-boot-guide.md](../board/raspberrypi5/nfs-boot-guide.md) 互为补充 — 那篇讲配置步骤，这篇聚焦**调试和故障排查**。

## 网络拓扑

```
┌────────────────────────┐          ┌────────────────────┐
│      开发机 (PC)        │          │      RPi5          │
│                        │          │                    │
│  有线网口 enp1s0        │          │  eth0              │
│  192.168.2.1/24        │◄─网线───►│  192.168.2.x (DHCP)│
│                        │          │                    │
│  TFTP :69/udp          │          │                    │
│  NFS  :2049/tcp        │          │                    │
│  ─────────────────     │          │  ───────────────   │
│  /tftpboot/            │  TFTP    │  下载 Image + DTB  │
│    Image               │─────────►│                    │
│    *.dtb               │          │                    │
│                        │          │                    │
│  /nfsroot/rpi5/        │   NFS    │  挂载为 /          │
│    bin/ etc/ lib/ ...  │◄─────────│  运行用户空间程序   │
└────────────────────────┘          └────────────────────┘
```

### 连接方式

| 方式 | 说明 |
|------|------|
| **直连** | 开发机和 RPi5 用网线直接连接，最简单可靠 |
| **经交换机** | 都插到同一个交换机/路由器上 |
| **开发机双网卡** | WiFi 上外网 + 有线口直连 Pi（推荐） |

> 推荐双网卡：WiFi 连外网（查资料、下载），有线口直连 Pi（没有其他设备干扰）。

---

## 第一步：开发机服务配置

### 1.1 设置有线网卡静态 IP

```bash
# 查看有线网卡名
ip link show | grep -E "^[0-9]+: e"

# 假设网卡名为 enp1s0，设置静态 IP
sudo ip addr add 192.168.2.1/24 dev enp1s0
sudo ip link set enp1s0 up

# 确认
ip addr show enp1s0
# inet 192.168.2.1/24
```

**永久生效** (NetworkManager)：

```bash
nmcli con add type ethernet ifname enp1s0 con-name pi-direct ip4 192.168.2.1/24
nmcli con up pi-direct
```

### 1.2 TFTP 服务

```bash
sudo apt install -y tftpd-hpa

sudo mkdir -p /tftpboot
sudo chmod 777 /tftpboot

# 配置文件
sudo tee /etc/default/tftpd-hpa <<'EOF'
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure --create"
EOF

sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa
```

**验证 TFTP 服务**：

```bash
# 放一个测试文件
echo "hello" > /tftpboot/test.txt

# 从本机测试
tftp localhost -c get test.txt
cat test.txt
# hello

# 查看服务状态
sudo systemctl status tftpd-hpa
```

### 1.3 NFS 服务

```bash
sudo apt install -y nfs-kernel-server

sudo mkdir -p /nfsroot/rpi5
sudo chmod 777 /nfsroot/rpi5

# 导出路径
echo "/nfsroot/rpi5 *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -ra

# 确认
sudo exportfs -v
# /nfsroot/rpi5 <world>(rw,sync,wdelay,...)
```

**验证 NFS 服务**：

```bash
# 本机挂载测试
sudo mkdir -p /mnt/nfs-test
sudo mount -t nfs 127.0.0.1:/nfsroot/rpi5 /mnt/nfs-test

echo "hello" | sudo tee /mnt/nfs-test/test.txt
cat /nfsroot/rpi5/test.txt
# hello

sudo umount /mnt/nfs-test
```

### 1.4 DNSMasq (集成 DHCP + TFTP，可选)

如果你的网络没有 DHCP 服务器，RPi5 没法自动获取 IP。用 dnsmasq 同时提供 DHCP 和 TFTP：

```bash
sudo apt install -y dnsmasq

sudo tee /etc/dnsmasq.conf <<'EOF'
# 监听网卡
interface=enp1s0
bind-interfaces

# DHCP 范围 (分配给设备)
dhcp-range=192.168.2.100,192.168.2.200,12h

# TFTP
enable-tftp
tftp-root=/tftpboot
EOF

sudo systemctl restart dnsmasq
```

---

## 第二步：准备 RPi5 SD 卡

需要一张**包含 NFS 启动参数**的 SD 卡。详见 [NFS Boot Guide](../board/raspberrypi5/nfs-boot-guide.md)。

关键修改点：

**cmdline.txt 里的 NFS 启动参数**：

```
root=/dev/nfs nfsroot=192.168.2.1:/nfsroot/rpi5,vers=4,tcp rw ip=dhcp console=tty1 console=ttyAMA10,115200
```

> 根据实际情况替换 IP。如果用了 U-Boot，cmdline 在 `boot.scr.txt` 里通过 `bootargs` 传递。

---

## 第三步：部署脚本

每次 `make` 完成后执行，把产物从 `output/images/` 推到 TFTP 和 NFS 目录：

```bash
cat > scripts/deploy-nfs.sh <<'SCRIPT'
#!/bin/bash
set -e

IMAGES="output/images"
TFTP="/tftpboot"
NFS="/nfsroot/rpi5"
SERVER_IP="192.168.2.1"

echo "=== Deploy to TFTP ==="
sudo cp ${IMAGES}/Image ${TFTP}/
for dtb in ${IMAGES}/broadcom/*.dtb; do
    sudo cp "$dtb" ${TFTP}/broadcom/
done
echo "  Kernel + DTB → ${TFTP}"

echo "=== Deploy to NFS ==="
sudo rm -rf ${NFS}/*
sudo tar xf ${IMAGES}/rootfs.tar -C ${NFS}/
echo "  Rootfs → ${NFS}"

echo ""
echo "=== Done ==="
echo "  ssh root@${SERVER_IP%%.*}.x reboot   OR   press RPi5 reset"
SCRIPT

chmod +x scripts/deploy-nfs.sh
```

---

## 第四步：上电启动 + 监控

### 4.1 串口监控（推荐）

RPi5 的 Debug UART 在 GPIO 14 (TX) / GPIO 15 (RX)：

```
RPi5 GPIO Header (J8):
 
 1  2               Pin 6:  GND
 3  4               Pin 8:  TX (GPIO14)
 5  6               Pin 10: RX (GPIO15)
 7  8
 9 10
... ...

USB-TTL 连接:
  GND → Pi Pin 6
  RX  → Pi Pin 8
  TX  → Pi Pin 10
```

> RPi5 的调试串口设备是 `ttyAMA10`，不是传统 RPi 的 `ttyAMA0`。

```bash
# 查看串口输出 (波特率 115200)
sudo screen /dev/ttyUSB0 115200
# 或
sudo minicom -D /dev/ttyUSB0 -b 115200
# 或
sudo picocom -b 115200 /dev/ttyUSB0
```

### 4.2 HDMI 监控

不用串口线就接 HDMI 显示器 + USB 键盘，同样能看到启动日志和控制台。

### 4.3 网络验证（启动后）

```bash
# 查看 DHCP 分配的 IP
sudo journalctl -u tftpd-hpa | tail
# 或在路由器/DHCP 服务器上看 lease

# ping 测试
ping 192.168.2.100   # 替换为 Pi 实际获取的 IP
```

---

## 故障排查

### Pi 完全不启动（彩虹屏/无输出）

| 可能原因 | 检查方法 |
|----------|----------|
| SD 卡坏了 | 换个卡重新烧 |
| config.txt 错误 | `kernel=Image` 指向的文件存在吗？ |
| start4.elf 缺失 | FAT 分区根目录有 start4.elf 吗？ |
| 供电不足 | RPi5 需要 5V/3A+ 的电源，确认适配器规格 |

### 内核启动了但停在 "Waiting for root device"

```
[    3.xxx] Waiting for root device /dev/nfs...
[   10.xxx] VFS: Unable to mount root fs via NFS, trying floppy.
```

**原因**：内核无法挂载 NFS rootfs。

排查步骤：

```bash
# 1. 开发机 NFS 服务在运行吗？
sudo systemctl status nfs-kernel-server
sudo exportfs -v

# 2. 防火墙有没有拦截？(NFSv4 用 2049/tcp, mountd/rpcbind 用 111)
sudo ufw allow 2049
sudo ufw allow 111
sudo ufw status

# 3. cmdline.txt 里 nfsroot= 的 IP 是开发机 IP 吗？
#    注意：如果开发机 IP 变了，必须更新 cmdline.txt

# 4. 可以尝试挂载到本机测试
sudo mount -t nfs 192.168.2.1:/nfsroot/rpi5 /mnt
ls /mnt/bin/
sudo umount /mnt

# 5. NFS 版本兼容性：试试 vers=3 替代 vers=4
#    cmdline.txt: ...,vers=3,tcp,...
```

### TFTP 超时

```
TFTP from server 192.168.2.1; our IP address is 192.168.2.x
Filename 'Image'
Load address: 0x80000
Loading: T T T T T    ← T 表示 Timeout
```

```bash
# 1. TFTP 服务运行？
sudo systemctl status tftpd-hpa

# 2. 文件在吗？
ls -la /tftpboot/Image

# 3. 权限？
sudo chmod 644 /tftpboot/*

# 4. 防火墙？
sudo ufw allow 69/udp

# 5. 本机测试
tftp 127.0.0.1 -c get Image
```

### NFS mount 后文件只读

```
# ls: Read-only file system
```

**原因**：`/etc/exports` 里忘了 `rw` 选项，或者 Buildroot 配置没开 remount rw。

```bash
# 检查导出选项
sudo exportfs -v | grep rpi5
# 应该包含 rw

# Buildroot 配置
grep REMOUNT .config
# BR2_SYSTEM_REMOUNT_ROOT_RW=y
```

### 开发机 IP 变了，Pi 找不到 NFS

如果开发机从 `192.168.2.1` 变成了 `192.168.2.100`：

```bash
# 短暂修复：更新 cmdline 里 nfsroot= 的 IP
# 长期修复：给开发机有线网卡设静态 IP
```

### 多个 Pi 同时开发怎么办

```bash
# 每个 Pi 用独立的 NFS 目录
sudo mkdir -p /nfsroot/rpi5-dev1
sudo mkdir -p /nfsroot/rpi5-dev2

sudo tee -a /etc/exports <<'EOF'
/nfsroot/rpi5-dev1 *(rw,sync,no_root_squash,no_subtree_check)
/nfsroot/rpi5-dev2 *(rw,sync,no_root_squash,no_subtree_check)
EOF
sudo exportfs -ra

# 每个 Pi 的 cmdline.txt 指向各自的 nfsroot
# Pi 1: nfsroot=192.168.2.1:/nfsroot/rpi5-dev1
# Pi 2: nfsroot=192.168.2.1:/nfsroot/rpi5-dev2
```

---

## 开发循环

标准操作流程（从改代码到看到结果）：

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. 修改代码/配置                                        │
│       ↓                                                 │
│  2. make -j$(nproc)    ← 增量编译，通常 30s-3min         │
│       ↓                                                 │
│  3. ./scripts/deploy-nfs.sh    ← 5-10s                  │
│       ↓                                                 │
│  4. ssh root@<pi-ip> reboot    ← 3-5s Pi 重启           │
│       ↓                                                 │
│  5. 等启动完成 (15-30s)                                  │
│       ↓                                                 │
│  6. ssh root@<pi-ip>   ← 验证改动                       │
│                                                         │
│  总耗时: ~1-3 分钟                                       │
└─────────────────────────────────────────────────────────┘
```

vs. SD 卡方案（改代码 → dd 烧卡 → 拔卡 → 插卡 → 开机 = 5-10 分钟）。

---

## 快速检查清单

开发环境排障口诀 — 按顺序查：

```
1. 网线插了吗？    → ip link show enp1s0, 确认状态 UP
2. IP 对吗？       → ip addr show enp1s0, 确认是 192.168.2.1/24
3. TFTP 起了吗？   → sudo systemctl status tftpd-hpa
4. NFS 起了吗？    → sudo exportfs -v, 确认 /nfsroot/rpi5 在列表中
5. 防火墙关了吗？  → sudo ufw status (或 sudo ufw disable 快速关闭)
6. Pi 有电吗？     → 看绿色 LED, 或串口有无输出
```

---

## 参考

- [NFS Boot 配置指南](../board/raspberrypi5/nfs-boot-guide.md)
- [内核 NFS root 文档](https://www.kernel.org/doc/html/latest/admin-guide/nfs/nfsroot.html)
- [RPi5 串口引脚](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#serial-port)
