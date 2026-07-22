# Buildroot 可视化库与开机自启指南

## 概述

Buildroot 支持在嵌入式 Linux 上运行图形界面程序，RPi5 有 HDMI 输出，可以直接接显示器。本文覆盖：可用 GUI 库、Display Server 选择、Python 绑定、以及如何开机自启你的 UI 程序。

## 开机自启：在哪里写

不管用什么 UI 库，自启的方式是一样的，取决于你用的 init 系统。

### 当前 init 系统确认

```bash
grep BR2_INIT /home/young/Embeded_Linux/buildroot/buildroot/.config
# BR2_INIT_BUSYBOX=y   ← 你的 RPi5 用的是 BusyBox init
# BR2_INIT_SYSTEMD is not set
```

### BusyBox init 方式（你当前的）

创建一个 init 脚本放到 SD 卡的 `/etc/init.d/` 下，文件名格式 `S<NN><name>`（NN 是执行顺序）：

```bash
cat > board/raspberrypi5/overlay/etc/init.d/S99myapp <<'EOF'
#!/bin/sh

case "$1" in
  start)
    echo "Starting my UI application..."
    # 你的程序：Python 程序或其他
    /usr/bin/my-ui-app &
    ;;
  stop)
    killall my-ui-app
    ;;
  restart)
    $0 stop
    sleep 1
    $0 start
    ;;
  *)
    echo "Usage: $0 {start|stop|restart}"
    exit 1
esac
EOF

chmod +x board/raspberrypi5/overlay/etc/init.d/S99myapp
```

| 文件名 | 执行时机 |
|--------|---------|
| `S01xxx` | 最早（网络还没起来） |
| `S30xxx` | 网络刚通 |
| `S50xxx` | 大部分服务已就绪 |
| `S99xxx` | **最后**（你的 UI 应该放这里，所有依赖已就绪） |

### systemd 方式（如果切换到 systemd）

```
board/raspberrypi5/overlay/etc/systemd/system/myapp.service:

[Unit]
Description=My UI Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/my-ui-app
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## 显示架构层级

在嵌入式 Linux 上，图形程序有几种运行层级：

```
Level 0:  /dev/fb0 直接写帧缓冲         ← 最底层，无窗口系统
Level 1:  SDL2/Qt 直接渲染到 DRM/KMS    ← 全屏应用，单进程
Level 2:  Wayland 合成器 (Weston)        ← 多窗口，硬件加速
Level 3:  X11 Server                     ← 完整桌面级，要求最高
```

对 RPi5 而言，上面四种 Buildroot **全部支持**。

---

## 可选方案对比

### 方案 A：直接 Framebuffer (最简单)

```
你的 Python/C 程序 → /dev/fb0 → HDMI 输出
```

- **无需任何 display server**，程序直接画像素到屏幕
- 使用 `python-pillow` 写 `/dev/fb0`，或用 C 的 SDL2 全屏模式
- 适合：固定布局仪表盘、简单 UI、无窗口需求

### 方案 B：Qt5/Qt6 + eglfs (推荐)

```
Qt 程序 → eglfs (DRM/KMS) → GPU → HDMI
```

- **无需 display server**，Qt 直接通过 DRM/KMS 驱动 GPU
- 硬件加速、流畅动画、专业的 UI 控件
- **Python 可用**（PyQt5）
- 适合：车载中控、工业 HMI、信息亭

### 方案 C：Weston + Wayland (现代)

```
你的程序 → Wayland 协议 → Weston 合成器 → DRM/KMS → GPU → HDMI
```

- 多窗口支持，现代架构
- 触摸屏、多屏都可管理
- 适合：带多个应用切换的产品

### 方案 D：X11 (传统桌面级)

```
你的程序 → X11 Server → DRM → GPU → HDMI
```

- 成熟稳定，工具链丰富
- 开销大，不适合资源受限场景
- 适合：产品原型、对兼容性要求高的场景

---

## Buildroot 中可用的 GUI 库

### 从 Python 使用

| 库 | Buildroot 包 | 显示方式 | Python 绑定 | 适用场景 |
|----|-------------|---------|------------|---------|
| **Qt5 + PyQt5** | `qt5` + `python-pyqt5` | eglfs/wayland/x11 | ✅ python-pyqt5 | 专业 GUI，完整控件库 |
| **GTK3** | `libgtk3` + `python-gobject` | wayland/x11 | ✅ python-gobject + python-pycairo | 桌面风格应用 |
| **Cairo** | `cairo` + `python-pycairo` | fb/sdl2/wayland/x11 | ✅ python-pycairo | 2D 绘图，图表 |
| **Pillow** | `python-pillow` | 直接写 /dev/fb0 | ✅ 原生 Python | 简单图片显示 |
| **Matplotlib** | `python-matplotlib` | 依赖后端(GTK/Qt/SDL) | ✅ 原生 Python | 数据图表 |

> Tkinter、Kivy、PyGame **不在 Buildroot 中**。PyQt5 是唯一完整可用的 Python GUI 框架。

### 从 C/C++ 使用

| 库 | Buildroot 包 | 显示方式 | 资源占用 | 适用场景 |
|----|-------------|---------|---------|---------|
| **Qt5** | `qt5` | eglfs/wayland/x11 | ~30MB | 企业级 GUI |
| **Qt6** | `qt6` | eglfs/wayland/x11 | ~35MB | Qt5 升级版 |
| **GTK3** | `libgtk3` | wayland/x11 | ~20MB | 桌面应用 |
| **GTK4** | `libgtk4` | wayland | ~25MB | 新一代 GTK |
| **SDL2** | `sdl2` | DRM/fb/x11 | ~2MB | 全屏游戏/简单 UI |
| **FLTK** | `fltk` | fb/x11 | ~2MB | 轻量级 UI |
| **EFL** | `efl` | fb/wayland | ~15MB | Enlightenment 生态 |
| **lvgl** | ❌ 不在 Buildroot 中 | - | - | （需手动添加包） |

---

## 推荐方案：Qt5 + PyQt5 + eglfs

这是 RPi5 上最适合 "Python GUI 程序" 的组合：

```
┌──────────────────────────┐
│   Python 3               │
│   └── PyQt5 (python-pyqt5)│
│        └── Qt5 (QtWidgets/QML) │
│             └── eglfs    │  ← Qt 的 EGL 全屏后端
│                  └── Mesa (GPU 驱动) │
│                       └── DRM/KMS  │
│                            └── HDMI │
└──────────────────────────┘
```

### 在 menuconfig 中启用

```
Target packages --->
  Interpreter languages --->
    [*] python3
    External python modules --->
      [*] python-pyqt5

  Graphic libraries and applications --->
    [*] qt5
        [*]   eglfs support
        [*]   widgets module        ← 传统控件 (QPushButton, QLabel...)
        [*]   quick module          ← QML 声明式 UI
        [*]   jpeg, png, gif support

  Libraries --->
    Graphics --->
      [*] mesa3d
          [*]   DRI driver for vc4  ← RPi5 VideoCore GPU
      [*] libdrm
```

### Python 示例程序

```python
#!/usr/bin/env python3
# my-ui-app.py — Qt5 全屏 UI 示例

import sys
from PyQt5.QtWidgets import QApplication, QLabel, QWidget, QVBoxLayout
from PyQt5.QtCore import Qt

app = QApplication(sys.argv)

# 创建全屏窗口
window = QWidget()
window.setWindowTitle("My RPi5 App")
window.showFullScreen()   # 全屏显示

# 简单布局
layout = QVBoxLayout()
label = QLabel("Hello Raspberry Pi 5!")
label.setAlignment(Qt.AlignCenter)
label.setStyleSheet("font-size: 48px; color: white; background-color: black;")
layout.addWidget(label)
window.setLayout(layout)

sys.exit(app.exec_())
```

### 自启配置

将程序放入 rootfs，配合 init 脚本：

```bash
# overlay/etc/init.d/S99myui
#!/bin/sh

export QT_QPA_PLATFORM=eglfs         # ← 关键：告诉 Qt 用 eglfs 直接渲染
export QT_QPA_EGLFS_PHYS_WIDTH=192   # 可选：告诉 Qt 你的屏幕物理尺寸(mm)
export QT_QPA_EGLFS_PHYS_HEIGHT=108

case "$1" in
  start)
    echo "Starting UI..."
    /usr/bin/python3 /opt/myapp/my-ui-app.py &
    ;;
  stop)
    killall python3
    ;;
  *)
    echo "Usage: $0 {start|stop}"
esac
```

---

## 其他方案的快速对比

### 如果你要做的是仪表盘/静态画面

→ **方案 A**: Python + Pillow 直接写 `/dev/fb0`，零依赖，几分钟上手。

### 如果你要做的是现代触屏应用

→ **方案 B**: Qt5 + PyQt5 + eglfs，最成熟稳定。

### 如果你要同时跑多个应用窗口

→ **方案 C**: Weston + Wayland + Qt5。

### 如果你要把已有的桌面 Linux 程序移植过来

→ **方案 D**: X11 + GTK3/Qt5，兼容性最好。

---

## 常用 Qt 环境变量速查

```bash
export QT_QPA_PLATFORM=eglfs         # 全屏 DRM/KMS，无窗口管理器
export QT_QPA_PLATFORM=wayland       # 走 Wayland
export QT_QPA_PLATFORM=xcb           # 走 X11
export QT_QPA_EGLFS_FORCE888=1       # 强制 RGB888（部分屏幕需要）
export QT_QPA_EGLFS_KMS_CONFIG=file  # 指定 KMS 配置
export QT_QPA_EGLFS_NO_LIBINPUT=1    # 禁用 libinput（使用 tslib 代替）
```

---

## 参考

- [Qt Embedded Linux 文档](https://doc.qt.io/qt-5/embedded-linux.html)
- [Qt eglfs 后端](https://doc.qt.io/qt-5/embedded-linux.html#eglfs)
- [Buildroot 手册 — Qt5](https://buildroot.org/manual.html#qt5)
