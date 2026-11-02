# Tina-Linux 快速导读手册

> 分析对象：`/home/hzen/Tina-Linux`（mangopi-sbc 维护的全志 F133/T113/D1 系列 Tina SDK），重点：**整体架构与构建流程**。以下结论均基于对源码、构建脚本与打包脚本的实际阅读，非 README 转述。

---

## 1. 项目速览 (TL;DR)

- **一句话定位**：一套面向全志 F133/T113/D1-H 系列 SoC 的**嵌入式 Linux 发行版 SDK**，基于 OpenWrt/buildroot 体系深度魔改，一条命令即可产出可直接烧录到 TF 卡 / eMMC / NAND 的量产固件（`.img`）。

- **核心技术栈**：
  - **OpenWrt 风格构建系统**（Kconfig + Make + ipkg 包管理，`build/main.mk`/`toplevel.mk`）
  - **Linux 5.4.61 全志魔改内核**（`lichee/linux-5.4` git 子模块）
  - **U-Boot 2018 + SPL/boot0 引导链**（`lichee/brandy-2.0/` 子模块）
  - **musl libc + BusyBox** 根文件系统（squashfs），GCC 6.4-linaro 预编译交叉工具链
  - **全志私有打包工具链** `pack-bintools`（`dragon`/`script`/`update_boot0`/`fsbuild` 等 30+ 个 host 工具）

- **适用场景**：
  - ✅ F133/T113/D1 系列板卡定制（内核裁剪、dts、量产固件、OTA、rootfs 裁减）
  - ✅ 产线打包（`dragon` 合成 PhoenixSuit/PhoenixCard 固件）、安全启动签名
  - ❌ 不适合：桌面/服务器发行版、与 OpenWrt 标准树直接做上层功能移植（改了很多，不是标准 OpenWrt）

---

## 2. 架构与目录解析

### 核心目录结构

```
Tina-Linux/
├── build/                    # 构建体系核心（OpenWrt 魔改版）
│   ├── envsetup.sh           # 命令入口：lunch/make/mboot/pack/get_build_var
│   ├── main.mk / toplevel.mk # OpenWrt "world" 构建主控（defconfig→编译→镜像）
│   ├── target.mk / kernel-build.mk / kernel-defaults.mk  # 内核构建框架
│   └── product.mk / product-config.mk                    # lunch 产品解析
├── device/config/chips/      # ★ 芯片/板级配置（全志风格）
│   └── t113/configs/
│       ├── default/          #   芯片默认配置（boot_package.cfg 等）
│       ├── mq_r/             #   ★ 板级目录：BoardConfig.mk / sys_config.fex /
│       │                     #     sys_partition.fex / env.cfg / board.dts /
│       │                     #     linux→linux-5.4(config-5.4) / boot-resource(bootlogo)
│       └── bingpi_m2/        #   新板卡（configs/config-* 分 kernel/tina/uboot 三份）
│   └── t113/bin/             #   boot0/u-boot/fes1/optee 二进制（mboot 产物 or 预编译）
├── lichee/
│   ├── linux-5.4/            # ★ 内核 git 子模块（全志 5.4 fork，board.dts 在此落位）
│   └── brandy-2.0/           # ★ U-Boot/SPL 子模块：u-boot-2018/ + spl-pub/ + build.sh
├── target/allwinner/         # ★ OpenWrt 板级工程
│   ├── generic/              #   通用层（image/Makefile 组装 boot.img/rootfs.img）
│   ├── t113-common/          #   T113 芯片公共（CPU/rootfs 镜像格式配置）
│   └── t113-mq_r/            #   ★ 板工程：defconfig / Makefile / BoardConfig.mk /
│                             #     TinaProducts.mk / busybox-init-base-files/（rootfs 定制）
├── package/                  # OpenWrt 包配方（git 子模块）
├── scripts/pack_img.sh       # ★ 固件打包总控（2154 行，dragon 调用方）
├── tools/                    # 68 个 host 构建工具（含 pack-bintools）
├── prebuilt/gcc/             # 预编译交叉工具链（arm/aarch64/riscv）
├── dl/                       # 下载的源码压缩包缓存（779 个）
└── out/<board>/              # 构建产物：image/、boot.img、rootfs.img、tina_*.img
```

### 数据流向 / 构建执行流

```
source build/envsetup.sh
   │  lunch (选 t113-mq_r)
   │   ├─ 导出 TARGET_PRODUCT/PLATFORM/BOARD/PLAN、LICHEE_CHIP/BOARD_CONFIG_DIR 等
   │   └─ fix_after_lunch：建符号链接  configs/mq_r/linux → linux-5.4
   ▼
make  (OpenWrt world)
   ├─ defconfig：cp target/allwinner/t113-mq_r/defconfig → .config（gitignored 产物）
   ├─ prepare：host 工具 → 工具链 → 内核
   │    └─ 内核：ln -s lichee/linux-5.4 → out/.../compile_dir/.../linux-5.4
   │         cp 板级 linux/config-5.4 + board.dts → 内核树，make zImage dtbs
   ├─ package：busybox + 各包 → 合并进 rootfs（squashfs）
   └─ image：target/allwinner/generic/image 产出 boot.img（zImage）与 rootfs.img
   ▼
mboot (必须单独执行！)
   └─ lichee/brandy-2.0/build.sh -p sun8iw20p1 -o uboot/spl
         → 产出 u-boot-sun8iw20p1.bin / boot0_sdcard_sun8iw20p1.bin → device/config/chips/t113/bin/
   ▼
pack → scripts/pack_img.sh
   ├─ do_prepare  : 汇集 板级fex配置 + bin/引导件 + boot.img/rootfs.img 到 out/<board>/image/
   ├─ do_common   : script(fex→bin) + update_boot0/update_uboot + fsbuild(boot-resource.fex)
   │                + dragonsecboot(boot_package.fex) + mkenv(env.fex)
   ├─ do_pack_tina: boot.fex→boot.img、rootfs.fex→rootfs.img 链接入目录
   ├─ do_finish   : update_mbr 生成分区表（sunxi_mbr.fex）
   └─ do_dragon   : dragon image.cfg sys_partition_for_dragon.fex
         → out/t113-mq_r/tina_t113-mq_r_uart3.img
   ▼
PhoenixCard（TF 卡量产） / PhoenixSuit（USB 烧录）
```

**启动链**：`boot0(SPL, SDMMC_CARD)` → `boot_package.fex(u-boot+optee+dtb)` → 读 env.cfg（`setargs_*`/`bootcmd`）→ 从 boot 分区读 zImage → 挂 squashfs rootfs。

---

## 3. 核心模块拆解

| 模块 | 核心职责 |
|---|---|
| **build/envsetup.sh** | 命令面。`lunch` 解析产品/平台/板级三要素并导出构建环境；`m`=make、`mboot`/`mboot0`=引导链、`pack`=打包、`get_build_var`=查询构建变量 |
| **build/main.mk / toplevel.mk** | OpenWrt world 主控。`defconfig` 目标把 `target/allwinner/<board>/defconfig` 灌入根 `.config`，随后按 `prepare→world→image` 编排全链路 |
| **build/target.mk + kernel-build.mk** | 内核构建框架。`find_kernel_config` 按 `DEVICE_PLATFORM_DIR/config-5.4`（即板级 `linux/` 目录）解析内核配置；`Kernel/Prepare` 用符号链接**原地编译** `lichee/linux-5.4`；编译期把板级 `board.dts` 拷贝进 `arch/arm/boot/dts/` |
| **lichee/brandy-2.0/build.sh** | U-Boot/SPL 构建。按 `-p <platform> -o uboot\|spl` 调 `make <defconfig>`，`LICHEE_BRANDY_DEFCONF`（板级 BoardConfig.mk 里 `sun8iw20p1_defconfig`）决定具体 defconfig |
| **scripts/pack_img.sh** | 固件打包总控（细节见第 2 节流程）。核心三件套：**`dragon` + `image.cfg` + `sys_partition.fex`** |
| **target/allwinner/<board>/** | 板级工程。`defconfig`=OpenWrt 全量配置（含 CONFIG_PACKAGE_* 包开关）、`Makefile` 声明 ARCH/CPU/内核版本、`busybox-init-base-files/` 直接落入 rootfs 的定制文件 |
| **device/config/chips/<chip>/configs/<board>/** | 全志私有板级配置。`sys_config.fex`=引脚/介质/uart 配置、`sys_partition.fex`=烧录分区表、`env.cfg`=u-boot 环境、`linux/config-5.4`=内核配置、`boot-resource/bootlogo.bmp`=开机 logo |

---

## 4. 快速上手指南

### 环境依赖

```sh
# Ubuntu 18.04/20.04 x86_64（务必支持 32 位库，dragon 是 ia32 ELF）
sudo apt-get install build-essential subversion git-core libncurses5-dev zlib1g-dev \
  gawk flex quilt libssl-dev xsltproc libxml-parser-perl mercurial bzr ecj cvs unzip \
  lib32z1 lib32z1-dev lib32stdc++6 libstdc++6 libmpc-dev libgmp-dev -y
```

### 构建与运行

```sh
git clone https://github.com/mangopi-sbc/Tina-Linux.git && cd Tina-Linux/
git submodule update --init --recursive        # 拉取 package/u-boot/kernel 子模块
# 另需按 README 下载静态资源：prebuilt.tar.gz（工具链）、dl.tar（源码包）、
# lichee/brandy-2.0/tools/toolchain/ 下的 arm 与 riscv 交叉工具链

source build/envsetup.sh
lunch                    # 选号：4=f133_evb1, 5=f133_mq_r, 6=t113_evb1, 7=t113_mq_r
make                     # 内核 + rootfs（首次数小时，.config 已生成可并行 make -jN）
mboot                    # 构建 u-boot + boot0（不执行则用 bin/ 里预编译的旧引导件）
pack                     # 打包 → out/t113-mq_r/tina_t113-mq_r_uart3.img
```

> 注意：镜像名 `tina_<board>_uart<N>.img` 中的 `<N>` 来自 `sys_config.fex` 的 `uart_debug_port`（mq_r=3 → `uart3`；README 里 d1-mq_pro 是 uart0）。

### 最小化 Demo（改板级 → 重新出固件）

```sh
# 1) 改内核配置
vim device/config/chips/t113/configs/mq_r/linux-5.4/config-5.4
# 2) 改板级配置（引脚/介质/uart）
vim device/config/chips/t113/configs/mq_r/sys_config.fex
# 3) 改 rootfs 里落地文件（如开机脚本/驱动模块）
vim target/allwinner/t113-mq_r/busybox-init-base-files/etc/init.d/rc.modules
# 4) 增量重建并打包
make -j$(nproc) && mboot && pack
# 5) 产物
ls -l out/t113-mq_r/tina_t113-mq_r_uart3.img   # → PhoenixCard 烧 TF 卡
```

---

## 5. 避坑与进阶提示

1. **`make` 不会编译 u-boot/boot0**。`make` 只出内核+rootfs+boot.img；必须先 `mboot` 再 `pack`，否则 `pack` 会拿 `device/config/chips/t113/bin/` 里**预编译的旧 boot0/u-boot** 打包。改过 dts/分区/介质后忘了 `mboot`，会出现"烧了镜像引导不了"。

2. **根目录 `.config` 是 gitignored 构建产物**（真源是 `target/allwinner/<board>/defconfig`）。切板子必须重新 `lunch` + `make`（defconfig 目标会 `cp defconfig .config`），不要手工编辑 `.config`；板级 `defconfig` 反而**被 git 跟踪**，改包配置应改它。

3. **`linux` 目录是 lunch 时建的符号链接**（`configs/mq_r/linux → linux-5.4`，`fix_after_lunch` 创建）。内核配置解析走 `find_kernel_config` 只看 `linux/config-5.4` 这个路径；想切内核版本要重 `lunch`，别手动删链接。

4. **子模块是"嵌套仓库"**：`lichee/linux-5.4`、`u-boot-2018`、`package` 都是 git submodule。git status 里的 ` m lichee/linux-5.4` 表示子模块内部有改动——内核/uboot 的修改要 `cd` 进子模块提交，父仓库只记录子模块指针。

5. **mq_r 的 env.cfg 默认 `bootcmd=run setargs_nand boot_normal`（NAND 启动），但 `sys_config.fex` 的 `storage_type=1` 是 SD**。烧 TF 卡时若启动异常，把 env.cfg 改成 `bootcmd=run setargs_mmc boot_normal` 再打包（或用 `fw_printenv`/u-boot 里改环境）。

6. **wifi 驱动是预编译落地文件**：`target/allwinner/t113-mq_r/busybox-init-base-files/lib/modules/5.4.61/r8188eu.ko`（untracked）由 `rc.modules` 手动 `insmod`，并 `wpa_supplicant -D wext` + `udhcpc` 自动连网。**换内核版本后该 .ko 需重新适配**（内核 vermagic 不匹配会加载失败）。

7. **打包环境注意 32 位兼容**：`dragon` 是 ia32 动态链接 ELF，缺 `lib32z1`/`lib32stdc++6` 会直接跑不起来；64 位环境下这也正是最常见的 `pack` 失败原因。

8. 进阶：`CONFIG_SUPPORT_PACK_OUT_OF_TINA=y` 时 `pack` 会额外产出 `out/<board>/aw_pack_src/`，交付客户后对方仅靠 `dragon image.cfg sys_partition.fex` 三件套即可自行出固件（`scripts/independently_pack/aw_pack.sh`）。

---

**一句话总结**：这套 SDK 的哲学是"**三层配置 + 一条命令**"——`target/allwinner/` 定 OpenWrt 层（包/rootfs）、`device/config/chips/` 定全志层（引脚/分区/引导/内核配置）、`lichee/` 定引导与内核源码；`make → mboot → pack` 三连把三层合成 PhoenixCard 可直接烧录的 `tina_<board>_uart<N>.img`。入手时抓住"改板级配置 → 重 lunch → make/mboot/pack"这条回路，比深入任何单个 mk 文件都更有效。
