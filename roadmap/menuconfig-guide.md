# Buildroot `make menuconfig` 配置指南

## 概述

`make menuconfig` 是 Buildroot 的配置界面，基于 Linux 内核的 Kconfig 系统。所有配置最终写入 `.config` 文件，`make` 时生效。整个界面大约包含 **2700+ 个软件包** 可供选择。

### 基础操作

| 按键 | 功能 |
|------|------|
| `↑↓←→` | 导航 |
| `Enter` | 进入子菜单 / 确认 |
| `Space` | 选择/取消 (`[*]` / `[ ]`) |
| `Y` / `N` | 启用 / 禁用 |
| `M` | 模块化编译 (Buildroot 不支持，忽略) |
| `/` | 搜索配置项 |
| `?` | 查看帮助 |
| `Esc Esc` | 返回上级 |
| `Tab` | 在 Save/Exit 间切换 |

---

## 一级菜单总览

```
Buildroot Configuration
├── Target options                    ← 目标架构选择
├── Build options                     ← 构建行为控制
├── Toolchain                         ← 交叉编译工具链
├── System configuration              ← 系统全局设置
├── Kernel                            ← Linux 内核
├── Target packages                   ← 用户空间软件包 (2700+)
│   ├── Audio and video               ← 音视频应用
│   ├── Compressors                   ← 压缩工具
│   ├── Debugging / profiling         ← 调试分析
│   ├── Development tools             ← 开发工具
│   ├── Filesystem / flash utils     ← 文件系统工具
│   ├── Graphic libraries / apps      ← 图形/UI
│   ├── Hardware handling             ← 硬件相关
│   ├── Interpreter languages         ← 脚本/编程语言
│   ├── Libraries                     ← 库文件 (含 14 个子类)
│   ├── Networking applications       ← 网络应用
│   ├── Package managers              ← 包管理器
│   ├── Security                      ← 安全工具
│   ├── Shell and utilities           ← Shell 和实用程序
│   ├── System tools                  ← 系统工具
│   └── Text editors and viewers      ← 编辑器/查看器
├── Filesystem images                 ← 根文件系统镜像类型
├── Bootloaders                       ← 引导加载程序
├── Host utilities                    ← 主机端工具
└── Legacy config options             ← 已废弃的配置项
```

---

## 逐级详解

### 1. Target options — 目标架构

```
Target options --->
  Target Architecture         ← ARM (little endian) / AArch64 / x86_64 / RISC-V ...
  Target Architecture Variant ← Cortex-A76, Cortex-A53 ...
  Target ABI                  ← EABI, EABIhf, ...
  Floating point strategy     ← VFPv4 / NEON / soft-float
  ARM instruction set         ← ARM / Thumb2
```

**你的 RPi5**：`AArch64 (64-bit)` + `Cortex-A76`。选错架构编译出来的程序在目标板上**跑不了**。

---

### 2. Build options — 构建行为

```
Build options --->
  Commands --->                  ← 下载工具路径 (wget, git, svn...) 一般不动
  Mirrors and Download locations ← 源码下载镜像地址
  Advanced --->
    (output)   Destination directory   ← 构建输出目录
    [ ]        Enable compiler cache   ← ccache 加速重复编译
    [*]        parallel jobs          ← 并行编译
    [*]        per-package directories ← 隔离每个包的构建 (重要)
```

| 关键选项 | 建议 |
|----------|------|
| `ccache` | 开启后重复编译快 3-5 倍，适合频繁 `make clean` 后重编 |
| `per-package directories` | 建议开启，解决包间文件冲突的常见问题 |
| `strip command` | 默认 strip 减小体积，调试时选 `none` |
| `gcc optimization` | 默认 `size`，开发时可选 `speed` 或 `0` |

---

### 3. Toolchain — 交叉编译工具链

```
Toolchain --->
  Toolchain type:
    ( ) Buildroot toolchain          ← 让 Buildroot 自己编译
    (•) External toolchain           ← 用预编译的工具链 (推荐新手)
    ( ) External custom toolchain    ← 用自定义工具链
```

**你的 RPi5 用的是**：`External toolchain → Bootlin → aarch64 glibc stable`。

| 类型 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Buildroot toolchain | 可定制 C 库版本、内核头文件版本 | **编译 30-60 分钟** | 需要精确控制工具链 |
| External 预编译 | **秒级就绪**，下载即用 | 不可定制 | 标准开发、快速原型 |
| External 自定义 | 完全控制 | 需自行准备工具链 | 厂商提供的特定工具链 |

> C 库选择：`glibc` 功能最全、体积大；`musl` 体积小、适合小型设备；`uClibc-ng` 更小但兼容性差。

---

### 4. System configuration — 系统全局设置

```
System configuration --->
  Root FS skeleton              ← 默认 skeleton (基础目录结构)
  System hostname               ← 主机名 (默认 buildroot)
  System banner                 ← 登录欢迎语
  Passwords encoding            ← sha-256 / sha-512 / md5
  Init system                   ← BusyBox init / systemd / OpenRC / none
  /dev management               ← devtmpfs (默认) / mdev / eudev
  Root password                 ← 为空表示无密码登录
  Port to run getty on          ← 串口控制台
  (eth0)  Network interface     ← 网络接口名，DHCP 配置
  Custom scripts to run         ← 构建后/镜像生成后执行的自定义脚本
```

| 关键项 | RPi5 当前设置 | 说明 |
|--------|--------------|------|
| Init system | BusyBox (默认) | 极简，适合无桌面系统 |
| /dev management | devtmpfs (默认) | 动态设备节点 |
| Network interface | `eth0` via DHCP | 自动获取 IP |
| Root password | 空 | 无需密码登录 |

> 如果改成 `systemd`，需要同时勾选 `udev`，并获得更完整的 systemd 生态（journald, systemd-networkd 等）。

---

### 5. Kernel — Linux 内核

```
Kernel --->
  [*] Linux Kernel
      Kernel version
        ( ) Latest stable
        (•) Custom tarball   ← RPi5 用的是 GitHub 上的 RPi 内核
        ( ) Custom Git
        ( ) Custom Mercurial
      Kernel configuration
        ( ) Use an in-tree defconfig ← bcm2712 (RPi5 的)
        ( ) Use a custom config file
      [*] Build a Device Tree Blob (DTB)
      [*] Needs host OpenSSL
```

你的 RPi5 用的是 Raspberry Pi 官方内核 (`github.com/raspberrypi/linux`)，defconfig 为 `bcm2712`，设备树为 `broadcom/bcm2712-rpi-5-b`。

---

### 6. Target packages — 用户空间软件包

这是最大的区域，按功能分为 **40+ 个子菜单**。以下列出最常用的：

#### 6.1 网络应用

```
Networking applications --->
  [ ] connman              ← 网络连接管理器
  [*] openssh              ← SSH 远程登录 (强烈推荐)
  [ ] wget                 ← 命令行下载
  [ ] curl                 ← HTTP/HTTPS 客户端
  [ ] ntp                  ← 时间同步
  [ ] rsync                ← 文件同步
  [ ] tcpdump              ← 网络抓包
  [ ] iperf3               ← 网络性能测试
  [ ] mosquitto            ← MQTT Broker (物联网常用)
```

#### 6.2 脚本/编程语言

```
Interpreter languages --->
  [*] python3              ← Python
  [ ] lua                  ← Lua
  [ ] nodejs               ← Node.js
  [ ] perl                 ← Perl
  [ ] php                  ← PHP
  [ ] ruby                 ← Ruby
```

> 选 Python 后会自动出现 **External python modules** 子菜单，可以选 `numpy`, `requests`, `flask`, `pyserial` 等数百个模块。

#### 6.3 系统工具

```
System tools --->
  [ ] coreutils            ← 完整版 GNU 工具 (ls, cp...)，替代 BusyBox 版本
  [ ] util-linux           ← 完整版 mount, fdisk, blkid... 替代 BusyBox 版本
  [ ] sudo                 ← sudo 命令
```

#### 6.4 调试工具

```
Debugging, profiling and benchmark --->
  [ ] gdb                  ← GDB 调试器
  [ ] strace               ← 系统调用跟踪
  [ ] valgrind             ← 内存泄漏检测
  [ ] htop                 ← 交互式进程查看
  [ ] iozone               ← 磁盘性能
  [ ] lmbench              ← 系统基准测试
```

#### 6.5 硬件相关

```
Hardware handling --->
  Firmware --->
    [*] rpi-firmware       ← Raspberry Pi GPU 固件 (必选)
  [ ] i2c-tools            ← I2C 总线工具
  [ ] spi-tools            ← SPI 总线工具
  [ ] gpio-tools           ← GPIO 操作
  [ ] rpi-userland         ← RPi 专用工具 (vcgencmd, raspistill 等)
```

#### 6.6 图形界面

```
Graphic libraries and applications --->
  [ ] X.org X Window System   ← 完整 X11
  [ ] weston                  ← Wayland 合成器
  [ ] Qt5 / Qt6               ← Qt 图形框架
  [ ] libgtk3                 ← GTK3 图形库
```

> RPi5 支持 HDMI 输出。如果要接屏幕，至少需要 `weston` (Wayland) 或 X11 + 一个桌面环境。

---

### 7. Filesystem images — 根文件系统镜像

```
Filesystem images --->
  [*] ext2/3/4 root filesystem    ← 生成 ext4 镜像 (你当前的)
  [ ] tar the root filesystem     ← 生成 rootfs.tar (NFS 开发需要)
  [ ] cpio the root filesystem    ← initramfs 用
  [ ] squashfs                    ← 压缩只读文件系统
  [ ] ubifs                       ← 用于裸 NAND Flash
```

你的 RPi5 当前只开了 ext4 (120MB)。建议至少也勾选 `tar`，用于 NFS 部署。

---

### 8. Bootloaders — 引导加载程序

```
Bootloaders --->
  [ ] U-Boot            ← 开源 bootloader
  [ ] Barebox           ← U-Boot 替代品
  [ ] GRUB2             ← x86 引导
  [ ] rpi-firmware      ← (在 Hardware handling → Firmware 下，不在这里)
```

---

### 9. Host utilities — 主机端工具

这些是构建过程中在**开发机上**运行的工具，不会进入目标 rootfs：

```
Host utilities --->
  [*] host dosfstools    ← 创建 FAT 分区 (RPi 需要)
  [*] host genimage      ← 生成 SD 卡镜像
  [*] host mtoools        ← 操作 FAT 分区
  [*] host kmod           ← 内核模块工具
```

这些你的 defconfig 已经勾选了，不用动。

---

## 开发阶段推荐配置

基于你当前的 `raspberrypi5_defconfig`，建议补上这些：

```
Target packages --->
  Networking applications --->
    [*] openssh                     ← SSH 登录 (必须)
    [*] wget                        ← 下载
    [*] iproute2                    ← ip 命令
    [*] tcpdump                     ← 抓包调试

  Compressors and decompressors --->
    [*] unzip
    [*] zstd

  Debugging, profiling --->
    [*] strace                      ← 调试神器
    [*] htop                        ← 进程监控

  Interpreter languages --->
    [*] python3                     ← Python，按需选模块

  Hardware handling --->
    [*] rpi-userland                ← vcgencmd 等 RPi 工具
    [*] i2c-tools
    [*] spi-tools

  System tools --->
    [*] coreutils                   ← 完整 GNU 工具 (可选)
    [*] sudo                        (可选)

Filesystem images --->
  [*] tar the root filesystem       ← NFS 开发需要
  (512M) exact size                 ← 调大一些

Bootloaders --->
  [*] U-Boot                        ← 后续 A/B + OTA 需要 (参考 roadmap/U-Boot.md)
```

加上这些后，rootfs 大约 200-300MB，建议把 ext4 大小调到 **512MB** 或 **1GB**。

---

## 搜索技巧

`make menuconfig` 中按 `/` 可以全局搜索：

```
搜索 "openssh" → 显示:
  Symbol: BR2_PACKAGE_OPENSSH
  Prompt: openssh
  Location: Target packages → Networking applications
```

如果你知道包名但不知道在哪个菜单，这是最快的方式。

---

## 配置检查

改完配置后，运行以下命令确保无冲突：

```bash
make olddefconfig     # 解析配置依赖，自动处理新增/移除的选项
```

如果所有符号都正确解析了（无交互询问），说明配置是完整的。然后 `make -j$(nproc)` 构建。

---

## 参考

- [Buildroot 官方手册 — Configuration](https://buildroot.org/manual.html#configure)
- [Buildroot 软件包列表](https://packages.buildroot.org/)
