# bl-mt798x — Agent Guide

U-Boot + ARM Trusted Firmware build wrapper for MediaTek MT7981/MT7986 router SoCs.
Fork of [hanwckf/bl-mt798x](https://github.com/hanwckf/bl-mt798x).

## Build

**Prerequisites (Linux host):**
```bash
sudo apt install gcc-aarch64-linux-gnu build-essential flex bison libssl-dev device-tree-compiler
```

**Build a board:**
```bash
SOC=mt7981 BOARD=360t7 ./build.sh
SOC=mt7986 BOARD=redmi_ax6000 MULTI_LAYOUT=1 ./build.sh
SOC=mt7981 BOARD=cmcc_rax3000m-emmc ./build.sh
```

Required env vars: `SOC` (`mt7981`|`mt7986`), `BOARD` (board name, matches defconfig suffix).
Optional: `MULTI_LAYOUT=1` (selects `*_multi_layout_defconfig`), `FIXED_MTDPARTS=0` (disable fixed mtdparts, default=1 for NAND).

## Repo Layout

| Path | Purpose |
|---|---|
| `build.sh` | **Single entry point.** Do not invoke u-boot or ATF make directly. |
| `uboot-mtk-20230718-09eda825/` | **Default** u-boot (2023.07 based + MediaTek patches + bootmenu/failsafe) |
| `uboot-mtk-20220606/` | Older u-boot (2022.06), **not default** |
| `atf-20231013-0ea67d76a/` | **Default** ATF. `configs` is a symlink → `../atf-20220606-637ba581b/configs` |
| `atf-20220606-637ba581b/` | Older ATF, contains the actual `configs/` directory. Both ATF versions share configs. |
| `output/` | Build artifacts (`*-fip.bin`, optionally `*-bl2.bin`). Gitignored. |

## Build Flow (build.sh)

1. Checks `SOC`, `BOARD` env vars, cross-compiler (`aarch64-linux-gnu-gcc`), Python 3
2. Verifies defconfig exists in both u-boot and ATF config dirs
3. eMMC boards → fixedparts=0, multilayout=0 always (no MTD partitions)
4. NAND boards → defaults to fixedparts=1, multilayout=0; MULTI_LAYOUT=1 switches to `*_multi_layout_defconfig`
5. **Step 1: Build u-boot** — copies defconfig to `.config`, optionally appends fixed-mtdparts, runs `make olddefconfig && make -j$(nproc) all`
6. Copies `u-boot.bin` → ATF dir
7. **Step 2: Build ATF** — `make -f makefile <defconfig>` then `make -f makefile clean && make -f makefile all`
8. Copies `build/{soc}/release/fip.bin` → `output/{soc}_{board}[-fixed-parts][-multi-layout]-fip.bin`
9. If config has `CONFIG_TARGET_ALL_NO_SEC_BOOT=y`, also copies `bl2.img` → `output/`

## Config Files

**u-boot defconfigs:** `uboot-mtk-20230718-09eda825/configs/mt798{1,6}_{boardname}[_multi_layout].defconfig`
**ATF defconfigs:** `atf-20220606-637ba581b/configs/mt798{1,6}_{boardname}.defconfig`

Both must exist with matching board name for a build to succeed.

### ATF defconfig keys
- `CONFIG_PLAT_MT7981=y` / `CONFIG_PLAT_MT7986=y` → selects SoC platform
- `CONFIG_FLASH_DEVICE_SPIM_NAND=y` / `CONFIG_FLASH_DEVICE_EMMC=y` → flash type
- `CONFIG_TARGET_FIP_NO_SEC_BOOT=y` → FIP without secure boot
- `CONFIG_TARGET_ALL_NO_SEC_BOOT=y` → also produces `bl2.img`
- `CONFIG_BGA=y` / `CONFIG_LOG_LEVEL_INFO=y`

### u-boot defconfig keys of interest
- `CONFIG_MEDIATEK_BOOTMENU=y` + `CONFIG_MEDIATEK_BOOTMENU_DELAY=3` — custom boot menu
- `CONFIG_MTK_WEB_FAILSAFE=y` — web recovery UI (source in `failsafe/`)
- `CONFIG_MTK_UBI_SUPPORT=y` — UBI flash image support
- `CONFIG_ENABLE_NAND_NMBM=y` — NAND bad block management
- `CONFIG_MTK_UPGRADE_BL2_VERIFY=y` — BL2 upgrade verification
- `CONFIG_DEFAULT_DEVICE_TREE` — sets the board-specific DTS
- `CONFIG_CMD_GL_BTN=y` — GPIO button command (used for reset/failsafe)
- `CONFIG_ENV_IS_IN_MTD=y` — environment stored on MTD partition

## Custom Features (MediaTek additions to upstream u-boot)

- **Bootmenu** (`board/mediatek/Kconfig`, `cmd/bootmenu.c`): Menu-driven firmware upgrade/boot. Device type configurable: MTD, MTD_LEGACY, EMMC.
- **Web Failsafe** (`failsafe/`): Embedded HTTP server for recovery. Activated on boot failure or reset button.
- **NMBM** (`cmd/nmbm.c`): NAND bad block management for SPI-NAND.
- **Multi-layout**: Alternate partition table scheme for OpenWrt firmware compatibility (stock vs. immortalwrt 112m vs. QWRT on wr30u, etc.)

## DTS / Device Tree

Board DTS files are in `uboot-mtk-20230718-09eda825/arch/arm/dts/`. The `CONFIG_DEFAULT_DEVICE_TREE` in defconfig selects which one.

## Git Conventions

- Single `master` branch
- Origin: `https://github.com/yufanpin/bl-mt798x.git`
- Upstream: `https://github.com/Kasazaki4090/bl-mt798x.git` (forked from hanwckf)
- Commits are in Chinese/English mix — standard conventional commits not expected

## Common Gotchas

- **Never run `make` directly in u-boot or ATF dirs** — always use `build.sh`. The build script orchestrates the two-stage build (u-boot → ATF) and copies binaries between them.
- **ATF-20231013's `configs/` is a symlink** to the older ATF configs dir. If adding a new board, the defconfig must go into `atf-20220606-637ba581b/configs/` (the real directory).
- **eMMC boards skip MTD partition logic** — `CONFIG_FLASH_DEVICE_EMMC=y` disables fixed-mtdparts and multilayout entirely.
- **Adding a new board** requires both a u-boot defconfig AND an ATF defconfig with the same `<soc>_<boardname>` prefix.
- **Python 3 is required** by the build script (checks with `command -v python3`).
- **Output directory** is `output/` at repo root. Binaries are named with pattern: `{SOC}_{BOARD}[-fixed-parts][-multi-layout]-fip.bin`.
