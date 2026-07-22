# Handoff — RPi5 br2rauc + tryboot A/B OTA 项目

> 上次更新: 2026-07-21

## 当前状态

### 项目目标
基于 `br2rauc` (Buildroot external tree) 配合 RPi5 固件原生 tryboot 机制，实现无需 U-Boot 的 A/B 分区 OTA 远程更新系统。

### 已完成

1. **环境搭建**
   - Buildroot 2026.05.1 主树位于 `buildroot/`，已配置为标准 RPi5 构建 (`raspberrypi5_defconfig`)
   - br2rauc external tree 位于 `br2rauc/`，已分析其完整结构和配置
   - 当前 `output/images/` 中已有一次成功构建产物 (sdcard.img, Image, DTBs, rootfs.ext2, boot.vfat)

2. **方案设计文档**
   - [x] `roadmap/tryboot-ab-ota.md` — tryboot 方案完整技术设计 (已存在)
   - [x] `roadmap/AB-OTA.md` — U-Boot A/B 方案设计 (已存在，备选)
   - [x] `roadmap/br2rauc-tryboot-init.md` — **本次新建**，br2rauc + tryboot 完整初始化实施文档

3. **分析完成**
   - br2rauc 现有架构：基于 U-Boot boot-mbr-switch 的 A/B + rescue 分区方案
   - RPi5 defconfig 分析：使用 RPi 固件加载 U-Boot，由 U-Boot 处理 A/B 选择
   - tryboot 方案关键差异：无 U-Boot，双 boot 分区，autoboot.txt + custom backend

### 待完成 (按优先级)

#### P0 — 立即可执行
- [ ] **创建工作目录**：在 `br2rauc/board/raspberrypi5/ab-ota/` 创建 tryboot 配置文件
  - `autoboot.txt`
  - `config.txt` (带 `[boot_partition=N]` 条件)
  - `cmdline_a.txt` / `cmdline_b.txt`
  - `system.conf` (RAUC with custom backend)
  - `rauc-backend.sh` (RAUC custom backend 脚本)
- [ ] **创建 board scripts**：
  - `br2rauc/board/raspberrypi5/post-build.sh`
  - `br2rauc/board/raspberrypi5/post-image.sh`
  - `br2rauc/board/raspberrypi5/genimage.cfg`
  - `br2rauc/board/raspberrypi5/busybox.fragment`
  - `br2rauc/board/raspberrypi5/users`
- [ ] **创建 defconfig**：`br2rauc/configs/raspberrypi5-tryboot-rauc_defconfig`
- [ ] **创建 rootfs overlay**：
  - `rootfs-overlay/etc/rauc/handler.sh` (post-install hook)
  - `rootfs-overlay/etc/systemd/system/rauc-mark-good.service` (auto mark-good)
- [ ] **生成 RAUC 签名密钥**：`cd br2rauc && ./openssl-ca.sh`

#### P1 — 构建验证
- [ ] 使用新 defconfig + br2rauc external tree 构建
- [ ] 检查构建产物 (sdcard.img, update.raucb)
- [ ] 在 RPi5 硬件上验证启动
- [ ] 验证 tryboot 标记设置与读取
- [ ] 验证 RAUC status / install / mark-good 流程

#### P2 — 增强
- [ ] 添加 rescue/recovery 模式 (GPIO 触发)
- [ ] OTA HTTP 服务器搭建
- [ ] 加密更新包支持
- [ ] 硬件 watchdog 完整配置

## 关键文件路径

| 用途 | 路径 |
|---|---|
| Buildroot 主树 | `buildroot/` |
| br2rauc external | `br2rauc/` |
| RPi5 通用 board | `br2rauc/board/raspberrypi/` (现有，共享) |
| RPi5 tryboot board | `br2rauc/board/raspberrypi5/` (需新建) |
| tryboot 配置文件 | `br2rauc/board/raspberrypi5/ab-ota/` |
| RAUC 签名密钥 | `br2rauc/openssl-ca/` (openssl-ca.sh 生成) |
| tryboot 方案设计 | `roadmap/tryboot-ab-ota.md` |
| U-Boot A/B 方案设计 | `roadmap/AB-OTA.md` |
| br2rauc+tryboot 初始化文档 | `roadmap/br2rauc-tryboot-init.md` |
| 工作交接 (本文件) | `handoff.md` |
| AI 会话配置 | `CLAUDE.md` |

## 关键决策

1. **使用 tryboot 而非 U-Boot**：RPi5 的 RP1 芯片在 U-Boot 下无驱动，tryboot 方案完全由 Linux 内核驱动 RP1，启动后网络/USB 全可用
2. **双独立 boot 分区**：每个 slot 拥有独立的 FAT32 boot 分区，包含完整的 RPi 固件文件、内核、DTB 和 cmdline
3. **RAUC custom backend**：使用 shell 脚本实现 RAUC 的 bootloader 接口，操作 `autoboot.txt` 和 dbt 中的 bootloader 信息
4. **MBR 分区表**：使用 MBR (非 GPT)，因为 RPi 固件 `boot_partition` 基于 MBR 分区号

## 下一步建议

从 **P0** 开始：先创建所有 board 配置文件和 defconfig，然后运行构建验证。参见 `roadmap/br2rauc-tryboot-init.md` 第四、五节了解每个文件的完整内容。
