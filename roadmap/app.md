# Buildroot + RPi5 应用程序开发指南

## 概述

在 Buildroot 框架中开发应用程序有三种方式，按复杂度递增排列：

| 方式 | 适用场景 | 迭代速度 |
|---|---|---|
| rootfs overlay（推荐开发阶段） | 脚本、预编译二进制、配置文件 | 最快，`make` 即可 |
| Buildroot package | 需要从源码编译、管理依赖 | 中等 |
| BR2_EXTERNAL 外部树 | 独立维护的复杂项目 | 结构最清晰 |

---

## 方式一：rootfs overlay（开发阶段首选）

### 原理

Buildroot 构建 rootfs 后，会把 `BR2_ROOTFS_OVERLAY` 指向的目录**直接覆盖**到 rootfs 上。

当前配置：`BR2_ROOTFS_OVERLAY="board/raspberrypi5/overlay"`

### 目录结构

```
board/raspberrypi5/overlay/
├── etc/
│   ├── init.d/
│   │   ├── S01font          ← 开机自启脚本（已有）
│   │   └── S99myapp         ← 你的应用（S 开头 = 自动执行）
│   └── wpa_supplicant.conf  ← WiFi 配置（已有）
├── usr/
│   └── bin/
│       └── myapp            ← 你的可执行文件
├── home/
│   └── app/
│       ├── main.py          ← Python 应用
│       └── lib/             ← 应用依赖
└── root/
    └── .profile             ← 自动登录后执行的命令
```

### Hello World 示例

**1. 创建 shell 脚本应用：**

```bash
mkdir -p board/raspberrypi5/overlay/usr/bin

cat > board/raspberrypi5/overlay/usr/bin/hello <<'EOF'
#!/bin/sh
echo "Hello from RPi5!"
echo "Current time: $(date)"
echo "IP address: $(ip addr show eth0 2>/dev/null | grep 'inet ' || echo 'no network')"
EOF

chmod +x board/raspberrypi5/overlay/usr/bin/hello
```

**2. 创建开机自启动脚本：**

```bash
mkdir -p board/raspberrypi5/overlay/etc/init.d

cat > board/raspberrypi5/overlay/etc/init.d/S99hello <<'EOF'
#!/bin/sh
# S99hello - 开机自动启动 hello 应用

case "$1" in
  start)
    echo "Starting hello app..."
    /usr/bin/hello > /tmp/hello.log 2>&1 &
    ;;
  stop)
    killall hello 2>/dev/null
    ;;
  *)
    echo "Usage: $0 {start|stop}"
    ;;
esac
EOF

chmod +x board/raspberrypi5/overlay/etc/init.d/S99hello
```

**3. 重新构建 rootfs（只需几秒）：**

```bash
make
# 或者只重建 rootfs:
make target-finalize
```

> **注意**：如果 overlay 中的脚本使用了 `#!/bin/bash` 但 rootfs 只有 `/bin/sh`（BusyBox），
> 脚本会失败。BusyBox 的 ash 兼容 POSIX sh，用 `#!/bin/sh` 即可。

---

## 方式二：Buildroot Package（从源码编译）

### 适用场景

- C/C++ 程序需要交叉编译
- 需要链接特定的库
- 需要在 Buildroot 中管理依赖关系

### 最小 Package 示例

**1. 创建 package 目录：**

```
package/myapp/
├── Config.in          ← Kconfig 菜单定义
└── myapp.mk           ← 构建规则
```

**2. Config.in：**

```
config BR2_PACKAGE_MYAPP
    bool "myapp"
    help
      My custom application for RPi5.
```

**3. myapp.mk（本地源码）：**

```makefile
################################################################################
#
# myapp
#
################################################################################

MYAPP_VERSION = 1.0
MYAPP_SITE = $(TOPDIR)/../myapp-source   # 源码路径（Buildroot 外）
MYAPP_SITE_METHOD = local
MYAPP_LICENSE = MIT

define MYAPP_BUILD_CMDS
    $(TARGET_MAKE_ENV) $(MAKE) $(TARGET_CONFIGURE_OPTS) -C $(@D)
endef

define MYAPP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/myapp $(TARGET_DIR)/usr/bin/myapp
endef

$(eval $(generic-package))
```

**4. 源码目录结构（`../myapp-source/`）：**

```
myapp-source/
├── main.c
└── Makefile
```

**5. main.c：**

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello from myapp!\n");
    return 0;
}
```

**6. Makefile（注意：用 Buildroot 的变量，不用硬编码 CC）：**

```makefile
.PHONY: all clean

all: myapp

myapp: main.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o myapp main.c

clean:
	rm -f myapp
```

**7. 启用 package：**

```bash
# 注册 Config.in（只需做一次）
echo 'source "package/myapp/Config.in"' >> package/Config.in

# menuconfig 勾选
make menuconfig
# Target packages → myapp
```

### Python Package 示例

```makefile
# package/myapp/myapp.mk
MYAPP_VERSION = 1.0
MYAPP_SITE = $(TOPDIR)/../myapp-source
MYAPP_SITE_METHOD = local
MYAPP_LICENSE = MIT

define MYAPP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/main.py $(TARGET_DIR)/usr/bin/myapp
endef

$(eval $(python-package))
```

---

## 方式三：BR2_EXTERNAL 外部树

适用场景：项目独立维护，和 Buildroot 主线分离，有自己的 defconfig、board 文件、package。

### 结构

```
myproject/
├── external.desc        ← BR2_EXTERNAL 标识文件
├── external.mk
├── Config.in            ← 顶层 Kconfig
├── board/
│   └── myboard/
│       ├── overlay/
│       └── post-build.sh
├── configs/
│   └── myboard_defconfig
└── package/
    └── myapp/
        ├── Config.in
        └── myapp.mk
```

### 使用

```bash
make BR2_EXTERNAL=/path/to/myproject myboard_defconfig
make
```

参考项目：`/home/young/Embeded_Linux/buildroot/br2rauc`

---

## 开发工作流

### 日常开发循环

```
修改应用源码
    ↓
make                        ← 增量编译（只重建变化部分，几秒到几分钟）
    ↓
# 然后根据改动的文件类型选择：
    ↓
[overlay 文件改动] → make 即可（不需要重新烧卡，overlay 直接打进 rootfs）
    ↓
[package 源码改动] → make myapp-rebuild && make
    ↓
烧录测试
pv output/images/sdcard.img | sudo dd of=/dev/sdX bs=4M conv=fsync
```

### 快速迭代技巧

**1. overlay 方式最快**：把编译好的二进制直接放到 overlay 目录，`make` 后烧录

**2. 交叉编译后放入 overlay（无需 Buildroot 重编）：**

```bash
# 用 Buildroot 的工具链直接交叉编译
TOOLCHAIN=output/host/bin/aarch64-linux-

${TOOLCHAIN}gcc -o myapp main.c

# 放到 overlay
cp myapp board/raspberrypi5/overlay/usr/bin/
make
```

**3. 只更新 rootfs 不重建 Image：**

```bash
make target-finalize        # 只重做 rootfs
# 然后手动替换 SD 卡的 rootfs 分区（ext4）
```

---

## 常见应用场景

### 场景 1：Python GUI 应用（PyQt5）

**依赖**：Python3 + PyQt5 已启用（menuconfig: `Target packages → Interpreter languages → python3` + Qt5）

```python
# board/raspberrypi5/overlay/usr/bin/myapp.py
import sys
from PyQt5.QtWidgets import QApplication, QLabel, QWidget, QVBoxLayout
from PyQt5.QtCore import Qt

def main():
    app = QApplication(sys.argv)
    win = QWidget()
    win.setWindowTitle("My App")
    label = QLabel("Hello RPi5!")
    label.setAlignment(Qt.AlignCenter)
    layout = QVBoxLayout()
    layout.addWidget(label)
    win.setLayout(layout)
    win.showFullScreen()
    sys.exit(app.exec_())

if __name__ == "__main__":
    main()
```

**开机自启脚本**：

```bash
# board/raspberrypi5/overlay/etc/init.d/S99pyapp
#!/bin/sh
case "$1" in
  start)
    echo "Starting PyQt5 app..."
    export QT_QPA_PLATFORM=eglfs          ← Qt5 在无 X11 下的后端
    /usr/bin/python3 /usr/bin/myapp.py &
    ;;
  stop)
    killall python3 2>/dev/null
    ;;
esac
```

### 场景 2：C 语言系统服务（守护进程）

```c
// 需要链接 pthread
// 交叉编译:
// output/host/bin/aarch64-linux-gcc -o mydaemon mydaemon.c -lpthread
```

### 场景 3：Web 管理后台（Python Flask + 轻量 HTTP）

1. 在 menuconfig 中启用 `python3` + `python-flask`
2. 应用代码放入 overlay
3. 开机自启脚本启动 Flask 服务

---

## 依赖管理

### 查看包是否已启用

```bash
# JSON 格式列出所有启用的包
make show-info | jq 'keys'

# 搜索特定包
make show-info | jq 'with_entries(select(.key | test("python|qt"))) | keys'
```

### 添加 Buildroot 已支持的包

```bash
make menuconfig
# 在 Target packages 中搜索并勾选
```

### 添加 Buildroot 未收录的 Python 包

```bash
# 扫描 PyPI 自动生成 Buildroot package
./utils/scanpypi <package-name> -o package/
```

### 查看依赖树

```bash
make <pkg>-show-depends     # 前向依赖
make <pkg>-show-rdepends    # 反向依赖（谁依赖它）
```

---

## 常用命令速查

| 操作 | 命令 |
|---|---|
| 只重建 rootfs | `make target-finalize` |
| 重建某个包 | `make <pkg>-rebuild` |
| 清理某个包 | `make <pkg>-dirclean` |
| 查看所有启用的包 | `make show-info` |
| 增量构建 | `make` |
| 完整 sdcard.img | `make` |
| 烧录 SD 卡 | `pv output/images/sdcard.img \| sudo dd of=/dev/sdX bs=4M conv=fsync` |
| 进入 U-Boot 命令行 | 上电时按 USB 键盘任意键（或串口按任意键） |
| 查看启动日志 | `dmesg \| less` |

---

## 总结：开发应用的最小步骤

1. **把应用放到 overlay**：`board/raspberrypi5/overlay/usr/bin/` 或 `/etc/init.d/`
2. **添加依赖**：`make menuconfig` 勾选需要的库
3. **设置自启**：overlay 中创建 `/etc/init.d/S99xxx` 脚本
4. **构建**：`make`
5. **烧录**：`pv output/images/sdcard.img | sudo dd of=/dev/sdX bs=4M conv=fsync`
