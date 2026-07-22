# RPi5 A/B 启动分区 + OTA 自动更新方案

## 概述

在已搭建的 NFS + TFTP 开发环境之上，为**最终产品部署**引入 A/B 分区和 OTA 自动更新能力。核心目标：

- **A/B 双槽位**：两套完整的启动镜像（A 和 B），更新失败自动回滚
- **OTA 更新**：通过网络下载更新包，安装到非活跃槽位，验证成功后切换
- **原子切换**：要么完整切换到新版本，要么保持旧版本不变，不存在中间态

选型：使用 **RAUC** (Robust Auto-Update Controller) + **U-Boot** 实现。

## 整体架构

```
┌─────────────────────────────────────────────────────┐
│                    OTA 服务器 (HTTP/HTTPS)            │
│                  firmware.example.com                │
│                   ──── update.raucb ────→            │
└─────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│                   Raspberry Pi 5                     │
│                                                     │
│  ┌──────────┐    ┌──────────┐    ┌───────────────┐  │
│  │ RAUC     │    │ U-Boot   │    │ SD 卡 (GPT)   │  │
│  │ 客户端   │◄──►│ boot.scr │◄──►│ boot (FAT32) │  │
│  │          │    │ env      │    │ rootfs_a     │  │
│  │ 下载+验 │    │ slot选择 │    │ rootfs_b     │  │
│  │ 证+安装 │    │ 回滚逻辑 │    │ data (持久)  │  │
│  └──────────┘    └──────────┘    └───────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 第一步：SD 卡分区方案

### 分区布局 (GPT)

```
/dev/mmcblk0
├── p1: boot      (FAT32, 128MB)   — RPi 固件 + U-Boot + 内核/DTB
├── p2: rootfs_a  (ext4, 512MB)    — 槽位 A 根文件系统
├── p3: rootfs_b  (ext4, 512MB)    — 槽位 B 根文件系统
└── p4: data      (ext4, 剩余空间)  — 持久化数据 (配置、日志、数据库)
```

### 为什么 boot 分区是共享的

RPi5 的启动链 `EEPROM → start4.elf` 只能从第一个 FAT 分区读取固件和 `config.txt`，无法做真正的双 boot 分区。但这对 A/B 方案影响很小：

- **boot 分区极少更新**：`start4.elf`、`config.txt`、`u-boot.bin` 几乎不变
- **内核文件以 A/B 命名区分**：`Image.A` / `Image.B`，`bcm2712-rpi-5-b.A.dtb` / `bcm2712-rpi-5-b.B.dtb`
- **U-Boot 根据 slot 状态选择对应的内核文件**

```
boot 分区内容:
├── start4.elf
├── fixup4.dat
├── config.txt
├── u-boot.bin
├── boot.scr                ← U-Boot 启动脚本 (带 A/B 逻辑)
├── uboot.env               ← U-Boot 环境变量 (存 slot 状态)
├── Image.A  Image.B        ← 两个槽位的内核
├── *.A.dtb  *.B.dtb        ← 两个槽位的设备树
└── (RAUC 会把对应 slot 的 kernel/dtb 拷贝到 boot 分区)
```

> RAUC 支持将 slot 的不同成员装在**不同位置**——一个 slot 可以有 FAT boot 上的内核文件 + ext4 上的 rootfs。

---

## 第二步：RAUC 简介

RAUC 由 Pengutronix 开发，专为嵌入式 Linux 设计的更新框架。核心概念：

### 术语

| 概念 | 说明 |
|---|---|
| **Bundle** (`.raucb`) | 更新包，一个签名的 tar 归档，包含新版本的全部文件 |
| **Slot** | 一个可启动的系统副本。系统中至少有两个 slot (A/B) |
| **Slot member** | 每个 slot 由多个成员组成（如 kernel image + rootfs 分区） |
| **Manifest** (`manifest.raucm`) | 描述更新包内容、目标 slot、版本信息的文件 |
| **Compatible** | 防止误刷：设备和更新包必须有相同的 compatible 字符串 |

### 更新流程

```
1. 开发机构建:
   make → 生成 rootfs.ext4 + Image + DTB
   rauc bundle → 签名打包为 update.raucb

2. 上传到 HTTP 服务器

3. RPi5 侧:
   rauc install http://server/update.raucb
   ├── 1. 下载 + 验证签名
   ├── 2. 写入非活跃 slot (rootfs_b + kernel)
   ├── 3. 标记新 slot 为 "待验证"
   └── 4. 重启

4. U-Boot:
   ├── 检查 boot status
   ├── 尝试启动新 slot
   └── 如果内核 panic 或启动失败 → 标记失败 → 回滚到旧 slot
```

---

## 第三步：Buildroot 配置

基于已有的 `raspberrypi5_defconfig`，追加以下配置。也可用 `make menuconfig` 逐项设置。

### 3.1 启用必需组件

```bash
cat >> .config <<'EOF'
# ====== GPT 分区表 ======
BR2_PACKAGE_HOST_GENIMAGE=y
BR2_PACKAGE_HOST_GPTFDISK=y

# ====== RAUC ======
BR2_PACKAGE_RAUC=y
BR2_PACKAGE_RAUC_NETWORK=y
BR2_PACKAGE_RAUC_GPT=y

# ====== U-Boot 环境变量工具 ======
BR2_PACKAGE_UBOOT_TOOLS=y
BR2_PACKAGE_UBOOT_TOOLS_FW_PRINTENV=y

# ====== 额外工具 ======
BR2_PACKAGE_UTIL_LINUX=y
BR2_PACKAGE_UTIL_LINUX_LIBS=y
BR2_PACKAGE_UTIL_LINUX_LIBFDISK=y
BR2_PACKAGE_LIBCURL=y

# ====== 双份 rootfs ======
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
BR2_TARGET_ROOTFS_EXT2_SIZE="512M"
# 不压缩 tar rootfs（NFS 调试用）
BR2_TARGET_ROOTFS_TAR=y
EOF
```

### 3.2 开启 U-Boot bootcount 支持

```
Bootloaders  --->
  [*] U-Boot
      ...
      [*]   Enable bootcount
      [*]   Enable boot limit
```

或修改 `.config`：

```bash
cat >> .config <<'EOF'
BR2_TARGET_UBOOT_BOOTCOUNT=y
BR2_TARGET_UBOOT_BOOTCOUNT_LIMIT=y
EOF
```

### 3.3 确认多份 rootfs 会被构建

需要在 Buildroot 中构建两个 rootfs。一种方式是使用两个 rootfs overlay，但 Buildroot 原生不直接支持"两份 rootfs"。**推荐做法**：

- 构建一次 rootfs，`rauc bundle` 时打包同一份（更新包是一样的内容，只是安装到不同的 slot）
- A/B 是两个**目标槽位**，不是构建时区分

---

## 第四步：创建 U-Boot A/B 启动脚本

```bash
cat > board/raspberrypi5/boot.scr.ab.txt <<'EOF'
# U-Boot A/B boot script for RPi5
# 根据 RAUC 写入的状态选择启动槽位

# 加载环境变量 (从 boot 分区 FAT)
load ${devtype} ${devnum}:1 ${loadaddr} uboot.env
env import -t ${loadaddr} ${filesize}

# 如果没有环境变量或首次启动，设置默认值
test -n "${BOOT_ORDER}" || setenv BOOT_ORDER "A B"
test -n "${BOOT_A_LEFT}" || setenv BOOT_A_LEFT 3
test -n "${BOOT_B_LEFT}" || setenv BOOT_B_LEFT 3

echo "=== A/B Boot Selection ==="
echo "Boot order : ${BOOT_ORDER}"
echo "Slot A tries left : ${BOOT_A_LEFT}"
echo "Slot B tries left : ${BOOT_B_LEFT}"

# 按 BOOT_ORDER 尝试启动
setenv boot_success 0

for BOOT_SLOT in ${BOOT_ORDER}; do
    if test "${BOOT_SLOT}" = "A"; then
        if test ${BOOT_A_LEFT} -gt 0; then
            echo ">>> Trying slot A <<<"
            setexpr BOOT_A_LEFT ${BOOT_A_LEFT} - 1

            # 保存更新的尝试次数
            setenv BOOT_SLOT_CURRENT A
            env save

            # 加载 slot A 的内核和 DTB
            load ${devtype} ${devnum}:1 ${kernel_addr_r} Image.A
            load ${devtype} ${devnum}:1 ${fdt_addr_r} broadcom/bcm2712-rpi-5-b.A.dtb

            # 指向 slot A 的 rootfs
            setenv bootargs console=ttyAMA10,115200 console=tty1 root=/dev/mmcblk0p2 rootwait

            booti ${kernel_addr_r} - ${fdt_addr_r}
        fi
    fi

    if test "${BOOT_SLOT}" = "B"; then
        if test ${BOOT_B_LEFT} -gt 0; then
            echo ">>> Trying slot B <<<"
            setexpr BOOT_B_LEFT ${BOOT_B_LEFT} - 1

            setenv BOOT_SLOT_CURRENT B
            env save

            load ${devtype} ${devnum}:1 ${kernel_addr_r} Image.B
            load ${devtype} ${devnum}:1 ${fdt_addr_r} broadcom/bcm2712-rpi-5-b.B.dtb

            setenv bootargs console=ttyAMA10,115200 console=tty1 root=/dev/mmcblk0p3 rootwait

            booti ${kernel_addr_r} - ${fdt_addr_r}
        fi
    fi
done

# 全部槽位都失败 → 进入 U-Boot 命令行
echo "!!! ALL BOOT SLOTS EXHAUSTED !!!"
echo "Falling back to U-Boot console"
EOF
```

### 启动逻辑说明

```
BOOT_ORDER = "A B"
BOOT_A_LEFT = 3     ← slot A 最多尝试 3 次
BOOT_B_LEFT = 3     ← slot B 最多尝试 3 次

流程:
1. 先试 A, BOOT_A_LEFT 减 1
2. 内核启动 → 用户空间正常运行 → RAUC 标记 boot successful
   → BOOT_A_LEFT 重置为 3
3. 如果内核 panic 或 watchdog 触发 → 重启
   → U-Boot 再次尝试，如果 A 仍失败 → 尝试 B
   → B 启动成功 → 系统恢复正常
4. 如果 A 和 B 全部耗尽 → 进入 U-Boot 命令行等待手动恢复
```

---

## 第五步：RAUC 配置

### 5.1 系统级配置 `/etc/rauc/system.conf`

```bash
cat > board/raspberrypi5/overlay/etc/rauc/system.conf <<'EOF'
[system]
compatible=RaspberryPi5-Buildroot
bootloader=uboot

# RAUC 管理的 slot 定义
[slot.rootfs.0]
device=/dev/mmcblk0p2
type=ext4
bootname=A

[slot.rootfs.1]
device=/dev/mmcblk0p3
type=ext4
bootname=B
EOF
```

### 5.2 密钥生成（开发机，只做一次）

```bash
# 创建开发用的 CA 和签名密钥
mkdir -p board/raspberrypi5/rauc-keys
cd board/raspberrypi5/rauc-keys

# 生成 CA 证书
openssl req -x509 -newkey rsa:4096 -nodes -keyout ca.key.pem \
    -out ca.cert.pem -days 3650 \
    -subj "/O=RAUC Development/CN=Development CA"

# 生成签名密钥
openssl req -newkey rsa:4096 -nodes -keyout signing.key.pem \
    -out signing.csr.pem \
    -subj "/O=RAUC Development/CN=Development Signing"

# 用 CA 签名
openssl x509 -req -in signing.csr.pem -CA ca.cert.pem -CAkey ca.key.pem \
    -out signing.cert.pem -days 3650 -CAcreateserial

cd -
```

### 5.3 RAUC 密钥配置 `/etc/rauc/ca.cert.pem`

在 Buildroot rootfs overlay 中包含 CA 证书：

```
Target packages  --->
  [*] RAUC
      [*]   network support

System configuration  --->
  Root filesystem overlay directories
    → board/raspberrypi5/overlay
```

并设置正确的 overlay 目录结构：

```
board/raspberrypi5/overlay/
└── etc/
    └── rauc/
        ├── system.conf         ← 上面创建的 RAUC 配置
        └── ca.cert.pem         ← CA 公钥 (用于验证更新包签名)
```

### 5.4 更新包清单模板 (manifest.raucm)

在开发机上创建，每次构建时自动生成：

```bash
cat > board/raspberrypi5/manifest.raucm.template <<'EOF'
[update]
compatible=RaspberryPi5-Buildroot
version=${VERSION}
description=Buildroot A/B OTA update

[image.rootfs]
filename=rootfs.ext4

[file.kernel-A]
filename=Image
dest=Image.A

[file.dtb-A]
filename=bcm2712-rpi-5-b.dtb
dest=broadcom/bcm2712-rpi-5-b.A.dtb

[file.kernel-B]
filename=Image
dest=Image.B

[file.dtb-B]
filename=bcm2712-rpi-5-b.dtb
dest=broadcom/bcm2712-rpi-5-b.B.dtb
EOF
```

---

## 第六步：构建 + 生成更新包

### 6.1 完整构建脚本

```bash
cat > scripts/build-ab-ota.sh <<'SCRIPT'
#!/bin/bash
set -e

BUILDROOT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
OUTPUT="${BUILDROOT_DIR}/output/images"
VERSION=$(date +%Y%m%d-%H%M%S)
RAUC_KEYS="${BUILDROOT_DIR}/board/raspberrypi5/rauc-keys"

cd "${BUILDROOT_DIR}"

echo "=== 1. 构建 Buildroot ==="
make -j$(nproc)

echo "=== 2. 生成 RAUC 更新 manifest ==="
sed "s/\${VERSION}/${VERSION}/" \
    board/raspberrypi5/manifest.raucm.template \
    > "${OUTPUT}/manifest.raucm"

echo "=== 3. 打包签名 RAUC bundle ==="
cd "${OUTPUT}"
rauc bundle \
    --cert="${RAUC_KEYS}/signing.cert.pem" \
    --key="${RAUC_KEYS}/signing.key.pem" \
    . \
    "update-${VERSION}.raucb"

echo "=== 4. 完成 ==="
echo "Update bundle: ${OUTPUT}/update-${VERSION}.raucb"
echo "Upload to: scp update-${VERSION}.raucb user@firmware.example.com:/var/www/ota/"
SCRIPT

chmod +x scripts/build-ab-ota.sh
```

### 6.2 首次烧录 SD 卡

```bash
# 使用 genimage 生成包含 GPT 分区布局的完整镜像
# 然后 dd 到 SD 卡
sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M status=progress conv=fsync
```

---

## 第七步：部署 + 更新操作

### 7.1 设备端手动更新

```bash
# 从本地文件更新
rauc install /tmp/update.raucb

# 从 HTTP 服务器下载并更新
rauc install http://firmware.example.com/ota/update-latest.raucb

# 查看当前状态
rauc status
```

### 7.2 设备端查看 slot 状态

```bash
# rauc status 输出示例
=== System Info ===
Compatible:  RaspberryPi5-Buildroot
Booted from: A (rootfs.0)

=== Booted Slot ===
Slot: rootfs.0
State: good

=== Slots ===
Slot rootfs.0 (A):
  State: good         ← 当前运行，验证通过
  Device: /dev/mmcblk0p2
  
Slot rootfs.1 (B):
  State: inactive     ← 空闲中，等待下次更新
  Device: /dev/mmcblk0p3
```

### 7.3 自动标记 good

RAUC 不会自动标记启动成功，需要在系统启动后由用户空间**显式确认**。在 `post-build.sh` 中加入自动标记逻辑：

```bash
# 在 /etc/init.d/S99rauc-mark-good 或 systemd service 中:
rauc status mark-good
```

> 必须在启动完成后显式调用 `rauc status mark-good`，否则多次启动失败后 U-Boot 会认为该 slot 损坏并回滚。

---

## 第八步：完整更新流程示意图

```
┌──────────────────────────────────────────────────────────────┐
│ 当前: Slot A 运行中, Slot B 闲置                              │
│                                                              │
│ 1. rauc install http://server/update.raucb                   │
│    └→ 下载 bundle → 验证签名 → 写入 slot B                    │
│       (rootfs 写入 /dev/mmcblk0p3, 内核拷到 boot 分区)        │
│                                                              │
│ 2. 重启                                                      │
│    └→ U-Boot 检测 BOOT_ORDER="A B"                           │
│       BOOT_B_LEFT 从 3 减为 2                                │
│       尝试加载 Image.B + /dev/mmcblk0p3                      │
│                                                              │
│ 3. 内核启动                                                  │
│    ├─ 成功 → rauc status mark-good                          │
│    │         BOOT_B_LEFT 重置为 3                            │
│    │         新版本进入 "good" 状态                           │
│    │                                                        │
│    └─ 失败 (panic/watchdog) → 自动重启                       │
│              U-Boot: BOOT_B_LEFT = 2, 再试                   │
│              如果 B 连续失败 3 次                             │
│              → 回滚到 A: 加载 Image.A + /dev/mmcblk0p2      │
│              → 系统用旧版本恢复运行                           │
└──────────────────────────────────────────────────────────────┘
```

---

## 故障排查

### 更新后无限重启

```bash
# 在 U-Boot 命令行手动选择旧 slot:
U-Boot> setenv BOOT_ORDER "A"      # 强制从 A 启动
U-Boot> setenv BOOT_A_LEFT 3
U-Boot> setenv BOOT_B_LEFT 0       # 禁用 B
U-Boot> saveenv
U-Boot> boot
```

### 签名验证失败

```bash
# 检查 CA 证书是否已正确安装
cat /etc/rauc/ca.cert.pem
# 确认 bundle 签名证书由同一 CA 签发
rauc info update.raucb | grep -A5 "Certificate"
```

### 查看 U-Boot 环境变量状态

```bash
# Linux 用户空间:
fw_printenv
# 输出: BOOT_ORDER, BOOT_A_LEFT, BOOT_B_LEFT, ...

# U-Boot 命令行:
U-Boot> printenv BOOT_ORDER
U-Boot> printenv BOOT_A_LEFT
```

---

## 与 NFS 开发环境的关系

```
开发阶段:
  NFS rootfs + TFTP 内核   ← 快速迭代，不需要烧卡
  (参考 board/raspberrypi5/nfs-boot-guide.md)

产品部署:
  A/B + RAUC OTA            ← 稳定可靠，支持远程更新
  (本文档)

过渡: 开发机 make → 测试 RAUC bundle → 部署到测试设备验证 OTA
```

两个方案共用同一套 Buildroot 和 U-Boot 配置，通过不同的 `boot.scr` 和 `cmdline.txt` 切换模式。

---

## 参考链接

- [RAUC 官方文档](https://rauc.readthedocs.io/)
- [RAUC U-Boot 集成](https://rauc.readthedocs.io/en/latest/integration.html#u-boot)
- [U-Boot Bootcount 文档](https://docs.u-boot.org/en/latest/usage/bootcount.html)
- [Buildroot RAUC 包](https://git.buildroot.net/buildroot/tree/package/rauc)
