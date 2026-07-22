# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Session Start — Read Handoff First

**IMPORTANT**: At the start of every session, before doing anything else, read `handoff.md` in the project root to pick up where the previous session left off. After completing each task, update `handoff.md` with the current status, completed items, and next steps.

## Overview

This is a workspace containing two related projects for Raspberry Pi 5 embedded Linux development:

1. **Buildroot 2026.05.1** at `https://github.com/Yueyuanh/buildroot.git` — the main build system, customized to support:
   - **NXP QorIQ Layerscape** embedded platforms (upstream tracking)
   - **Raspberry Pi 5** with br2rauc + tryboot A/B OTA (active development)

2. **br2rauc** at `../br2rauc/` — a Buildroot external tree providing RAUC-based A/B OTA update infrastructure, originally for RPi CM4/4B, being adapted for RPi5 tryboot.

### Active Project: RPi5 br2rauc + tryboot A/B OTA

The current active work is adapting br2rauc to use RPi5 firmware's native `tryboot` mechanism (instead of U-Boot) for A/B partition OTA updates. See `roadmap/br2rauc-tryboot-init.md` for the full initialization guide, `roadmap/tryboot-ab-ota.md` for the technical design, and `handoff.md` for current work status.

Currently supported Layerscape targets:
- `ls1046a-rdb` — LS1046A Reference Design Board
- `ls1046a-frwy` — LS1046A Freeway Board
- `ls1043a-rdb` — LS1043A Reference Design Board
- `ls1028ardb` — LS1028A Reference Design Board

All NXP-forked components (rcw, atf, u-boot, linux) are pinned to the **Linux Factory tag `lf-6.18.20-2.0.0`** sourced from the `github.com/nxp-qoriq` organization.

## Building

```bash
# Configure for a Layerscape board:
make ls1046a-rdb_defconfig    # (or ls1046a-frwy_defconfig, ls1043a-rdb_defconfig, ls1028ardb_defconfig)

# Customize if needed:
make menuconfig

# Build (use -jN to parallelize):
make
```

Output images land in `output/images/` and typically include `sdcard.img`, `Image`, device trees (`.dtb`), `bl2_sd.pbl`, `fip.bin`, `u-boot.bin`, `rootfs.ext4`, and `PBL.bin`.

## Key Buildroot Conventions

- Upstream documentation: `docs/manual/` (generate with `make manual-text` or use `make manual-html`). Online: https://buildroot.org/docs.html
- `make list-defconfigs` — list all available defconfigs.
- `make linux-menuconfig` / `make uboot-menuconfig` — reconfigure individual components after the initial build.
- `make <pkg>-dirclean` / `make <pkg>-rebuild` — clean/rebuild individual packages.
- Graph utilities: `make graph-build-time`, `make graph-depends` for build time/complexity visualization.
- The `.gitlab-ci.yml` at the repo root is the upstream CI; use `support/scripts/generate-gitlab-ci-yml` to regenerate.
- `support/scripts/genimage.sh` — the post-image script referenced by all Layerscape board configs for generating `sdcard.img`.
- Hash files for each NXP-forked tarball are maintained in `board/freescale/<board>/patches/<component>/<component>.hash` (not in the package directory).

## Architecture: Layerscape Board Layout

Each Layerscape board follows this structure under `board/freescale/<board>/`:

```
board/freescale/ls1046a-rdb/
  genimage.cfg          — SD card partition layout (genimage tool config)
  patches/
    linux/linux.hash    — sha256 hash for the NXP linux tarball
    uboot/uboot.hash    — sha256 hash for the NXP u-boot tarball
    arm-trusted-firmware/arm-trusted-firmware.hash  — hash for NXP ATF tarball
    linux-headers/      — patches for kernel headers if needed
  readme.txt            — board-specific build/boot instructions, DIP switch settings
  rootfs_overlay/
    boot/extlinux/extlinux.conf  — boot config (kernel, dtb, cmdline)
```

## Architecture: NXP Custom Packages

Custom packages live in `package/` and are specific to NXP Layerscape platforms:

| Package | Kind | Purpose |
|---|---|---|
| `qoriq-rcw` | Host | Reset Configuration Word — builds PBL.bin from .rcw sources; supports intree and custom paths |
| `qoriq-fm-ucode` | Target (blob) | Frame Manager microcode binary for LS104x platforms |
| `qoriq-mc-binary` | Target (blob) | Management Complex firmware for DPAA2 platforms (LS208x/LX216x) |
| `qoriq-mc-utils` | Target | Userspace tools for Management Complex (restool, etc.) |
| `qoriq-ddr-phy-binary` | Target (blob) | DDR PHY training firmware |
| `qoriq-firmware-inphi` | Target (blob) | Inphi 25G/40G SerDes firmware |
| `qoriq-cadence-dp-firmware` | Target (blob) | Cadence DisplayPort firmware for LS1028A |
| `qoriq-restool` | Target | REST-based DPAA2 resource management tool |
| `fmlib` | Target | Frame Manager library |
| `fmc` | Target | Frame Manager Configuration tool |
| `nxp-mwifiex` | Target | NXP Wi-Fi driver |
| `nxp-bt-wifi-firmware` | Target (blob) | NXP Bluetooth/Wi-Fi firmware binaries |

Each package follows the standard Buildroot `package/<name>/` layout: `Config.in`, `<name>.mk`, `<name>.hash`.

## Boot Flow (Layerscape)

The Layerscape boot chain is:

1. **POR (Power-On Reset)** → loads **PBL** (Pre-Boot Loader, built from RCW by `host-qoriq-rcw`)
2. **PBL** → loads **BL2** (`bl2_sd.pbl` from ATF)
3. **BL2** → loads **FIP** (`fip.bin`, a Firmware Image Package containing BL31 + BL32 + BL33)
4. **FIP** → loads **U-Boot** (as BL33) → loads **Linux** kernel + device tree + rootfs

Key ATF build variables in defconfigs:
- `BR2_TARGET_ARM_TRUSTED_FIRMWARE_FIP=y` — build Firmware Image Package
- `BR2_TARGET_ARM_TRUSTED_FIRMWARE_UBOOT_AS_BL33=y` — include U-Boot as BL33 in FIP
- `BR2_TARGET_ARM_TRUSTED_FIRMWARE_RCW=y` — integrate RCW/PBL into the build
- `BR2_TARGET_ARM_TRUSTED_FIRMWARE_ADDITIONAL_VARIABLES="BOOT_MODE=sd"` — build for SD boot
- `BR2_TARGET_ARM_TRUSTED_FIRMWARE_IMAGES` — list of output images (fip.bin, bl2_sd.pbl)

## SD Card Image Layout

Defined by `genimage.cfg`. Typical Layerscape layout:

| Partition | Offset | Content |
|---|---|---|
| FSBL | 4K | `bl2_sd.pbl` |
| SSBL (FIP) | 1M | `fip.bin` |
| FMAN ucode | 9M | FM microcode binary (LS104x only) |
| rootfs | 16M | ext4 root filesystem |

## Bumping BSP Versions

When a new NXP Linux Factory release comes out, the BSP tag must be updated across all these locations consistently:

1. **Defconfigs** — `BR2_LINUX_KERNEL_CUSTOM_TARBALL_LOCATION`, `BR2_TARGET_ARM_TRUSTED_FIRMWARE_CUSTOM_TARBALL_LOCATION`, `BR2_TARGET_UBOOT_CUSTOM_TARBALL_LOCATION` — all reference the tag in `$(call github,nxp-qoriq,<repo>,<tag>)`
2. **Package .mk files** — `QORIQ_RCW_VERSION`, `QORIQ_FM_UCODE_VERSION`, `FMLIB_VERSION`, `FMC_VERSION`, `QORIQ_DDR_PHY_BINARY_VERSION`, etc.
3. **Board patch hash files** — `board/freescale/<board>/patches/<component>/<component>.hash` must be updated with the new tarball SHA256
4. **Config.in.legacy** — if renaming/removing config options

## External Dependencies

NXP component tarballs are fetched from GitHub (`github.com/nxp-qoriq`). All package versions reference the same Linux Factory tag. The `BR2_DOWNLOAD_FORCE_CHECK_HASHES=y` setting in defconfigs enforces hash verification.

## RPi5 br2rauc + tryboot A/B OTA Project

### Workspace Layout

```
~/Embeded_Linux/my_projects/rpi5/
├── br2rauc/                          ← Buildroot external tree (RAUC A/B OTA)
│   ├── board/raspberrypi/            ← Shared RPi board files (existing)
│   ├── board/raspberrypi5/           ← RPi5 tryboot-specific files (TO BE CREATED)
│   │   └── ab-ota/                   ← autoboot.txt, config.txt, system.conf, etc.
│   ├── configs/                      ← Defconfig files
│   │   ├── raspberrypi5-rauc_defconfig        ← Existing U-Boot based config
│   │   └── raspberrypi5-tryboot-rauc_defconfig ← New tryboot config (TO BE CREATED)
│   ├── openssl-ca/                   ← RAUC signing keys (generated)
│   └── linux.fragment                ← Kernel config fragment (verity + squashfs)
│
└── buildroot/                        ← Buildroot 2026.05.1 main tree
    ├── output/                       ← Current build output (standard rpi5 config)
    ├── roadmap/                      ← Design documents
    │   ├── br2rauc-tryboot-init.md   ← Full initialization guide
    │   ├── tryboot-ab-ota.md         ← tryboot technical design
    │   └── AB-OTA.md                 ← U-Boot A/B alternative design
    ├── handoff.md                    ← Session-to-session work handoff
    └── CLAUDE.md                     ← This file
```

### Key Differences: U-Boot vs tryboot

| Aspect | U-Boot (existing) | tryboot (new) |
|---|---|---|
| Bootloader | U-Boot `rpi_arm64` | None (RPi firmware direct) |
| Boot partition | 1 shared FAT | 2 independent FAT (one per slot) |
| A/B selection | U-Boot boot.scr + env | autoboot.txt + tryboot flag |
| Boot retry | bootcount (multi-retry) | one-shot (single attempt) |
| RAUC backend | `bootloader=uboot` (built-in) | `bootloader=custom` (shell script) |
| U-Boot tools needed | Yes (fw_printenv, etc.) | No |

### Tryboot A/B OTA Task Tracking

See `handoff.md` for the current status and prioritized task list. See `roadmap/br2rauc-tryboot-init.md` for the complete implementation guide with all file contents.

### Quick Build Commands (tryboot config — once defconfig is created)

```bash
cd ~/Embeded_Linux/my_projects/rpi5
make -C buildroot/ BR2_EXTERNAL=../br2rauc O=../output-rpi5-rauc raspberrypi5-tryboot-rauc_defconfig
cd output-rpi5-rauc
make -j$(nproc)
```
