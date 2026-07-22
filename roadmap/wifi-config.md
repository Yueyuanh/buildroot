# RPi5 WiFi 配置指南

## 概述

Raspberry Pi 5 板载 Broadcom/Cypress WiFi + 蓝牙芯片，通过 SDIO 接口连接。Buildroot 下启用 WiFi 需要三个层面的配置：**内核驱动**、**固件文件**、**用户空间管理工具**。

## RPi5 WiFi 硬件

```
SoC: BCM2712
WiFi: Cypress CYW43455 (或类似型号)
接口: SDIO (mmc)
内核驱动: brcmfmac (Broadcom FullMAC)
固件:   brcmfmac43455-sdio.bin + 对应的 .txt NVRAM 文件
        brcmfmac43455-sdio.clm_blob
```

---

## 第一步：内核驱动

RPi5 的官方内核 (`bcm2712_defconfig`) **默认已启用** `brcmfmac`。确认一下：

```bash
# 检查当前内核配置
grep -E "CFG80211|BRCMFMAC" output/build/linux-*/.config 2>/dev/null | head -5
```

如果缺少，创建 kernel config fragment：

```bash
cat > board/raspberrypi5/linux-wifi.fragment <<'EOF'
CONFIG_CFG80211=y
CONFIG_WLAN=y
CONFIG_NET_WIRELESS=y
CONFIG_WIRELESS=y
CONFIG_BRCMFMAC=y
CONFIG_BRCMFMAC_SDIO=y
CONFIG_CRYPTO_SHA256=y
CONFIG_CRYPTO_AES=y
EOF
```

然后在 `menuconfig` 中加上此 fragment：

```
Kernel  --->
  Additional configuration fragment files:
    board/raspberrypi/linux-4k-page-size.fragment
    board/raspberrypi5/linux-console-font.fragment
    board/raspberrypi5/linux-wifi.fragment             ← 新增
```

> 如果 WiFi 内核模块是 M（模块），还需要 `CONFIG_BRCMFMAC_SDIO=y` + `CONFIG_BRCMFMAC=y` 确保编入内核，避免启动后手动 `modprobe`。

---

## 第二步：固件

RPi5 的 WiFi 固件由 `linux-firmware` 包提供：

```
make menuconfig
```

```
Target packages  --->
  Hardware handling  --->
    Firmware  --->
      [*] linux-firmware
          [*]   Broadcom BRCM bcm43xxx
          [*]   Cypress CY cyw43xxx
```

等价 `.config` 配置：

```bash
cat >> .config <<'EOF'
BR2_PACKAGE_LINUX_FIRMWARE=y
BR2_PACKAGE_LINUX_FIRMWARE_BRCM_BCM43XXX=y
BR2_PACKAGE_LINUX_FIRMWARE_CYPRESS_CYW43XXX=y
EOF
```

安装后的固件路径：

```
/lib/firmware/brcm/
├── brcmfmac43455-sdio.bin          ← WiFi 固件
├── brcmfmac43455-sdio.txt          ← NVRAM 配置
├── brcmfmac43455-sdio.clm_blob     ← CLM 校准数据
└── brcmfmac43455-sdio.raspberrypi,5-model-b.txt
```

---

## 第三步：用户空间管理工具

### 3.1 wpa_supplicant（推荐，最常用）

```
Target packages  --->
  Networking applications  --->
    [*] wpa_supplicant
        [*]   nl80211 support       ← 现代 Linux WiFi 驱动标准接口
        [*]   Install wpa_cli       ← 命令行管理工具
        [*]   Install wpa_passphrase ← 生成加密密码
```

### 3.2 iwd（轻量替代，Intel 出品）

```
Target packages  --->
  Networking applications  --->
    [*] iwd
```

iwd 比 wpa_supplicant 更轻、启动更快，但社区成熟度略低。新手建议先用 wpa_supplicant。

### 3.3 命令行工具（辅助）

```
Target packages  --->
  Networking applications  --->
    [*] iw                         ← 替代 wireless-tools，查询 WiFi 信息
    [*] iproute2                   ← ip 命令，替代 ifconfig/route
```

---

## 第四步：配置文件

### 4.1 wpa_supplicant.conf

```bash
mkdir -p board/raspberrypi5/overlay/etc

cat > board/raspberrypi5/overlay/etc/wpa_supplicant.conf <<'EOF'
# /etc/wpa_supplicant.conf
ctrl_interface=/var/run/wpa_supplicant
ap_scan=1
update_config=1

# 你的 WiFi 网络
network={
    ssid="MyWiFi"
    psk="mypassword"
    key_mgmt=WPA-PSK
}
EOF
```

> 用 `wpa_passphrase SSID PASSWORD` 可以生成加密的 psk，避免明文存密码：

```bash
wpa_passphrase "MyWiFi" "mypassword"
# network={
#     ssid="MyWiFi"
#     #psk="mypassword"
#     psk=5e884898da28047151d0e56f8dc62...  ← 使用这行替代明文 psk
# }
```

### 4.2 多网络配置（手机热点 + 家里 WiFi）

```bash
cat > board/raspberrypi5/overlay/etc/wpa_supplicant.conf <<'EOF'
ctrl_interface=/var/run/wpa_supplicant
ap_scan=1

# 优先级：数字越大越优先
network={
    ssid="HomeWiFi"
    psk="home_password"
    priority=10
}

network={
    ssid="iPhone Hotspot"
    psk="hotspot_password"
    priority=5
}

# 开放网络
network={
    ssid="FreeWiFi"
    key_mgmt=NONE
    priority=1
}
EOF
```

### 4.3 网络接口配置（BusyBox ifupdown）

```bash
cat > board/raspberrypi5/overlay/etc/network/interfaces <<'EOF'
# loopback
auto lo
iface lo inet loopback

# 有线 — DHCP
auto eth0
iface eth0 inet dhcp

# WiFi — DHCP，启动时自动连
auto wlan0
iface wlan0 inet dhcp
    pre-up wpa_supplicant -B -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf
    post-down killall -q wpa_supplicant
EOF
```

### 4.4 静态 IP（可选）

```bash
# 不用 DHCP，手动指定
auto wlan0
iface wlan0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    pre-up wpa_supplicant -B -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf
    post-down killall -q wpa_supplicant
```

---

## 第五步：Overlay 使配置生效

确认 `.config` 中 overlay 路径包含自定义文件：

```bash
grep BR2_ROOTFS_OVERLAY .config
# BR2_ROOTFS_OVERLAY="board/raspberrypi5/overlay"
```

overlay 目录结构：

```
board/raspberrypi5/overlay/
└── etc/
    ├── wpa_supplicant.conf       ← WiFi 密码配置
    └── network/
        └── interfaces            ← 网络接口启动配置
```

---

## 第六步：构建 + 验证

```bash
make -j$(nproc)
```

启动后在 RPi5 上验证：

```bash
# 1. 检查网卡是否识别
ip link show
# 应该看到 wlan0

# 2. 扫描 WiFi
iw dev wlan0 scan | grep SSID

# 3. 查看 IP
ip addr show wlan0
# inet 192.168.x.x/24

# 4. 查看连接状态
wpa_cli status
# wpa_state=COMPLETED

# 5. 测通
ping 8.8.8.8
```

---

## 常见问题

### wlan0 找不到

```bash
# 检查内核模块是否加载
lsmod | grep brcm

# 检查固件是否安装
ls /lib/firmware/brcm/brcmfmac43455*

# 手动加载驱动
modprobe brcmfmac

# 查看内核日志
dmesg | grep -i "brcm\|wlan\|firmware"
```

### 连接被拒绝 / 密码错误

```bash
# 前台运行 wpa_supplicant 调试
killall wpa_supplicant
wpa_supplicant -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf -dd
# 看详细日志，定位是驱动问题还是密码问题
```

### WiFi 频繁断连

```bash
# wpa_supplicant.conf 加入省电关闭
network={
    ssid="MyWiFi"
    psk="password"
    # 关闭 WiFi 省电模式
}

# 或手动关闭省电
iw dev wlan0 set power_save off
```

### 启动后 WiFi 不自动连

```bash
# 检查 /etc/network/interfaces 是否有 auto wlan0
grep "auto wlan0" /etc/network/interfaces

# 手动启动
ifup wlan0
```

---

## WiFi + 有线双网络

如果需要 WiFi 和有线同时工作，注意**不要配两个默认网关**。通常做法：有线优先走外网，WiFi 作备用：

```bash
# /etc/network/interfaces
auto eth0
iface eth0 inet dhcp
    metric 10               ← 优先级高（数字小）

auto wlan0
iface wlan0 inet dhcp
    metric 20               ← 优先级低
    pre-up wpa_supplicant -B -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf
    post-down killall -q wpa_supplicant
```

> BusyBox 的 `ifupdown` 支持 `metric`，内核 >= 4.x 已自带支持。

---

## 一键配置脚本

把上面的所有配置整合，快速启用 WiFi：

```bash
cat > scripts/setup-wifi.sh <<'SCRIPT'
#!/bin/bash
# 一键配置 RPi5 WiFi

CONFIG=".config"

echo "=== 1. 启用 linux-firmware ==="
cat >> ${CONFIG} <<'EOF'
BR2_PACKAGE_LINUX_FIRMWARE=y
BR2_PACKAGE_LINUX_FIRMWARE_BRCM_BCM43XXX=y
BR2_PACKAGE_LINUX_FIRMWARE_CYPRESS_CYW43XXX=y
EOF

echo "=== 2. 启用 wpa_supplicant ==="
cat >> ${CONFIG} <<'EOF'
BR2_PACKAGE_WPA_SUPPLICANT=y
BR2_PACKAGE_WPA_SUPPLICANT_NL80211=y
BR2_PACKAGE_WPA_SUPPLICANT_CLI=y
BR2_PACKAGE_WPA_SUPPLICANT_PASSPHRASE=y
EOF

echo "=== 3. 启用 iw + iproute2 ==="
cat >> ${CONFIG} <<'EOF'
BR2_PACKAGE_IW=y
BR2_PACKAGE_IPROUTE2=y
EOF

echo "=== 4. 配置 overlay 文件 ==="
mkdir -p board/raspberrypi5/overlay/etc/network

cat > board/raspberrypi5/overlay/etc/wpa_supplicant.conf <<'WPA'
ctrl_interface=/var/run/wpa_supplicant
ap_scan=1

network={
    ssid="YOUR_SSID"
    psk="YOUR_PASSWORD"
}
WPA

cat > board/raspberrypi5/overlay/etc/network/interfaces <<'NET'
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto wlan0
iface wlan0 inet dhcp
    pre-up wpa_supplicant -B -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf
    post-down killall -q wpa_supplicant
NET

echo ""
echo "=== 完成 ==="
echo "请编辑 board/raspberrypi5/overlay/etc/wpa_supplicant.conf 填入你的 WiFi 密码"
echo "然后: make -j\$(nproc)"
SCRIPT

chmod +x scripts/setup-wifi.sh
```

---

## wpa_supplicant vs iwd 对比

| | wpa_supplicant | iwd |
|---|---|---|
| 诞生年份 | 2003 | 2018 |
| 依赖 | libnl | D-Bus + ELL |
| rootfs 增量 | ~2MB | ~3MB |
| 启动速度 | 慢 | 快 |
| 配置格式 | 自定义（易读） | 自定义 |
| 成熟度 | 行业标准 | 较新 |
| 建议 | **新手推荐** | 追求精简可尝试 |

---

## 参考

- [wpa_supplicant 配置文件说明](https://w1.fi/cgit/hostap/plain/wpa_supplicant/wpa_supplicant.conf)
- [RPi Linux WiFi 驱动](https://github.com/raspberrypi/linux/tree/rpi-6.6.y/drivers/net/wireless/broadcom/brcm80211)
- [iw 命令用法](https://wireless.wiki.kernel.org/en/users/documentation/iw)
