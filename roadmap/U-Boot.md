# RPi5 + U-Boot 启动配置指南

## 概述

Raspberry Pi 5 使用 U-Boot 作为中间 bootloader，提供可编程的启动流程，为后续 A/B 分区 OTA 升级做准备。

**当前状态（已验证）**：内核成功启动，HDMI 终端正常显示。

---

## RPi5 启动流程

```
上电
  └→ ROM → EEPROM → start4.elf (闭源固件，必须保留)
       └→ 读 config.txt
            kernel=u-boot.bin        ← 固件把 U-Boot 当成"内核"加载
            arm_64bit=1              ← RPi5 强制 64 位
            enable_uart=1            ← 启用 UART（仅 Debug 口）
       └→ 固件加载 DTB、应用 overlay、修改 DTB（含 simple-framebuffer）
       └→ 固件把修改后的 DTB 放在高位内存，指针传给 U-Boot
       └→ 启动 ARM 核，跳到 U-Boot 入口
                              └→ U-Boot 初始化
                                   └→ 执行 boot.scr
                                        ├→ fdt move ${fdt_addr} ${fdt_addr_r}  ← 关键！
                                        ├→ fdt get value bootargs /chosen bootargs
                                        ├→ load mmc 0:1 ${kernel_addr_r} Image
                                        └→ booti ${kernel_addr_r} - ${fdt_addr_r}
                                             └→ Linux 内核启动
```

---

## 遇到的 Bug 及修复（踩坑记录）

### Bug 1：DTB 路径错误

**现象**：U-Boot logo 显示，内核不启动

**原因**：`boot.scr.txt` 中 DTB 路径写的 `broadcom/bcm2712-rpi-5-b.dtb`，但 FAT 分区上文件在根目录 `bcm2712-rpi-5-b.dtb`

**修复**：去掉 `broadcom/` 前缀
```
# 错误
load mmc 0:1 ${fdt_addr_r} broadcom/bcm2712-rpi-5-b.dtb

# 正确
load mmc 0:1 ${fdt_addr_r} bcm2712-rpi-5-b.dtb
```

### Bug 2：Image 不在 boot 分区

**现象**：改 `kernel=Image` 直接启动时报 "boot error: code 7"

**原因**：`post-image.sh` 只根据 `config.txt` 的 `kernel=u-boot.bin` 把 `u-boot.bin` 加入 boot 分区，`Image` 被遗漏

**修复**：在 `genimage.cfg.in` 中显式添加 `"Image"`

### Bug 3：固件 DTB 在高位内存，内核无法访问（RPi5 专有）

**现象**：U-Boot logo 显示，内核永远不启动，屏幕永远卡在 logo

**原因**：RPi5 固件把修改后的 DTB 放在高位内存（接近 RAM 顶部），U-Boot 的
`booti ${kernel_addr_r} - ${fdt_addr}` 直接传这个地址给内核，但内核无法访问该地址

**社区确认**：[RPi 5 rev 1.1 stuck on uboot logo](https://forums.raspberrypi.com) —
多位用户报告同样的问题，在 RPi4 上正常工作的配置在 RPi5 上卡 logo

**修复**：先用 `fdt move` 把 DTB 搬到 `fdt_addr_r` (0x05600000) 安全地址

```bash
# 关键命令 — 搬移固件 DTB 到安全地址
fdt addr ${fdt_addr}
fdt move ${fdt_addr} ${fdt_addr_r}

# 然后从安全地址启动
booti ${kernel_addr_r} - ${fdt_addr_r}
```

### Bug 4：bcm2708_fb 与 simple-framebuffer 冲突导致内核 panic

**现象**：内核启动后立即崩溃：

```
bcm2708_fb_probe → register_framebuffer → fbcon_prepare_logo → bitfill_aligned
Kernel panic - not syncing: Attempted to kill init!
```

**原因**：内核的 `CONFIG_FB_BCM2708`（旧版 RPi 显示驱动）和固件 DTB 中的
`simple-framebuffer` 节点同时生效，两个驱动抢同一块显存，`bcm2708_fb` 访问了
已被释放/重映射的内存地址导致崩溃

**修复**：添加内核 fragment 禁用 `CONFIG_FB_BCM2708`：

```
# board/raspberrypi5/linux-no-bcm2708fb.fragment
# CONFIG_FB_BCM2708 is not set
```

### Bug 5：缺少 arm_64bit=1

**现象**：`kernel=Image` 启动时报 "boot error: code 7"

**原因**：RPi5 64 位内核需要 `arm_64bit=1` 在 `config.txt` 中显式声明

### Bug 6：RPi5 的 GPIO 串口（GPIO 14/15）在 U-Boot 下不可用

**现象**：UART 转 TTL 模块接 GPIO 14/15，串口完全无输出

**原因**：RPi5 的 GPIO 通过 RP1 芯片（PCIe 南桥）连接，U-Boot 2026.07
没有 RP1 驱动。GPIO 14/15 的 UART、以太网、USB 在 U-Boot 下均不可用。

**结论**：RPi5 + U-Boot 下调试必须用 **Debug UART**（两个 HDMI 口之间的 3 针接口），
或通过 HDMI 显示观察。TFTP/NFS 网络启动在 RPi5 的 U-Boot 下不可行。

### Bug 7：覆盖固件的 bootargs

**现象**：测试期间多次尝试手动 `setenv bootargs`，导致固件自动添加的平台参数
（`coherent_pool`、`vc_mem`）丢失

**修复**：不用 `setenv bootargs`，直接从固件 DTB 读取：

```bash
fdt get value bootargs /chosen bootargs
```

固件 DTB 中的 `/chosen/bootargs` 已包含完整的参数（cmdline.txt 内容 +
固件追加的 `coherent_pool=1M vc_mem.mem_base=0x3ec00000 vc_mem.mem_size=0x40000000`）。

---

## 最终配置文件

### 1. config_5_uboot.txt（RPi 固件配置）

文件：`board/raspberrypi5/config_5_uboot.txt`

```
enable_uart=1
kernel=u-boot.bin
disable_overscan=1
arm_64bit=1
```

### 2. boot.scr.txt（U-Boot 启动脚本）

文件：`board/raspberrypi5/boot.scr.txt`

```bash
# U-Boot boot script for RPi5 — SD 卡独立启动
# 已验证在 RPi5 上正常工作

# 1. 把固件 DTB 从高位内存搬到安全地址（RPi5 必须！）
fdt addr ${fdt_addr}
fdt move ${fdt_addr} ${fdt_addr_r}

# 2. 从固件 DTB 中读取完整的 bootargs
#    （含 coherent_pool, vc_mem 等固件追加的平台参数）
fdt get value bootargs /chosen bootargs

# 3. 从 SD 卡加载内核
load mmc 0:1 ${kernel_addr_r} Image

# 4. 用安全地址的 DTB 启动
booti ${kernel_addr_r} - ${fdt_addr_r}
```

### 3. genimage.cfg.in（SD 卡镜像布局）

文件：`board/raspberrypi5/genimage.cfg.in`

```
image boot.vfat {
    vfat {
        files = {
#BOOT_FILES#
            "boot.scr",
            "Image",
        }
    }
    size = 32M
}

image sdcard.img {
    hdimage { }
    partition boot {
        partition-type = 0xC
        bootable = "true"
        image = "boot.vfat"
    }
    partition rootfs {
        partition-type = 0x83
        image = "rootfs.ext4"
    }
}
```

> **关键**：`"Image"` 必须显式写在 `genimage.cfg.in` 中，因为 `post-image.sh`
> 只根据 `config.txt` 的 `kernel=u-boot.bin` 自动添加 `u-boot.bin`，
> 不会自动添加 `Image`。

### 4. 内核配置片段

**linux-no-bcm2708fb.fragment**（禁用冲突的旧版显示驱动）：

```
# CONFIG_FB_BCM2708 is not set
```

**linux-display.fragment**（为 VC4 DRM 预留 CMA 内存）：

```
CONFIG_CMA_SIZE_MBYTES=64
```

**linux-console-font.fragment**（控制台字体支持）：

```
CONFIG_FONTS=y
CONFIG_FONT_8x16=y
CONFIG_FONT_SUN12x22=y
CONFIG_FONT_TER16x32=y
```

### 5. Buildroot .config 关键项

```
# U-Boot
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="rpi_arm64"
BR2_PACKAGE_HOST_UBOOT_TOOLS=y
BR2_PACKAGE_HOST_UBOOT_TOOLS_BOOT_SCRIPT=y
BR2_PACKAGE_HOST_UBOOT_TOOLS_BOOT_SCRIPT_SOURCE="board/raspberrypi5/boot.scr.txt"

# rpi-firmware
BR2_PACKAGE_RPI_FIRMWARE=y
BR2_PACKAGE_RPI_FIRMWARE_CONFIG_FILE="board/raspberrypi5/config_5_uboot.txt"

# 内核配置片段
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES="board/raspberrypi/linux-4k-page-size.fragment board/raspberrypi5/linux-console-font.fragment board/raspberrypi5/linux-display.fragment board/raspberrypi5/linux-no-bcm2708fb.fragment"
```

---

## 构建与烧录

```bash
# 构建
make -j$(nproc)

# 烧录 SD 卡
pv output/images/sdcard.img | sudo dd of=/dev/sdX bs=4M conv=fsync
sync && sudo eject /dev/sdX
```

---

## SD 卡分区内容

```
分区1 (FAT32, 32MB, bootable):
├── u-boot.bin              ← config.txt: kernel=u-boot.bin
├── boot.scr                ← U-Boot 启动脚本 (mkimage 编译)
├── Image                   ← Linux 内核 (arm64 Image)
├── bcm2712-rpi-5-b.dtb     ← 设备树
├── bcm2712d0-rpi-5-b.dtb
├── bcm2712-rpi-500.dtb
├── start4.elf              ← RPi 固件 (rpi-firmware 包)
├── fixup4.dat
├── config.txt              ← 内容来自 config_5_uboot.txt
├── cmdline.txt             ← 备用 (U-Boot 模式下由固件 DTB 传参)
└── overlays/               ← DT overlay 文件

分区2 (ext4):
└── rootfs 根文件系统
```

---

## RPi5 + U-Boot 已知限制

| 功能 | 状态 | 原因 |
|---|---|---|
| SD 卡启动 | ✅ 正常 | - |
| HDMI 显示 | ✅ 正常 | simple-framebuffer + VC4 DRM |
| 以太网 | ❌ 不可用 | RP1 芯片无 U-Boot 驱动 |
| USB | ❌ 不可用 | RP1 芯片无 U-Boot 驱动 |
| GPIO 14/15 串口 | ❌ 不可用 | 经过 RP1，U-Boot 无驱动 |
| Debug UART (3-pin) | ✅ 可用 | BCM2712 内置 PL011，直连 |
| TFTP 网络启动 | ❌ 不可行 | 无网络驱动 |
| NFS rootfs | ❌ 不可行 | 无网络驱动 |

> **结论**：RPi5 的 U-Boot 目前只能用于 SD 卡本地启动。如果需要网络启动，
> 需等 U-Boot 上游完善 RP1 芯片驱动。

---

## 调试手段

由于 RPi5 GPIO 串口在 U-Boot 下不可用，调试方法有限：

1. **HDMI 显示** — 内核启动后可看到 console 输出和 login 提示
2. **Debug UART** — 两个 HDMI 口之间的 3 针接口，接 USB-TTL 模块（波特率 115200）
3. **内核 panic 日志** — 内核崩溃时会打印到 HDMI，拍照记录

---

## 后续：A/B OTA 升级

U-Boot 环境变量支持 `saveenv` 持久化，配合 `boot.scr` 中的 A/B 分区选择逻辑，
可实现：

- 双 rootfs 分区（A/B）
- 启动计数与回滚
- RAUC 集成

参见 [AB-OTA.md](AB-OTA.md)。

---

## 参考

- [U-Boot RPi 板级文档](https://docs.u-boot.org/en/latest/board/broadcom/raspberrypi.html)
- [Buildroot RPi5 defconfig](https://github.com/buildroot/buildroot/tree/master/configs/raspberrypi5_defconfig)
- br2rauc 项目：Buildroot + RAUC + U-Boot on RPi5 参考实现
- RPi 论坛 U-Boot RPi5 讨论：[RPi 5 rev 1.1 stuck on uboot logo](https://forums.raspberrypi.com)
