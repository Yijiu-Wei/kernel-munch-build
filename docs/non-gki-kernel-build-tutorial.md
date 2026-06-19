# 非GKI内核编译完全教程

> 适用目标：Android 设备自定义内核编译，支持 KernelSU / SUSFS / APatch 等
> 最后更新：2026-06-20

---

## 目录

1. [GKI 与非 GKI 的区别](#1-gki-与非-gki-的区别)
2. [准备工作](#2-准备工作)
3. [获取内核源码](#3-获取内核源码)
4. [获取工具链](#4-获取工具链)
5. [配置内核（Defconfig）](#5-配置内核defconfig)
6. [编译内核](#6-编译内核)
7. [打包 boot.img](#7-打包-bootimg)
8. [刷入设备](#8-刷入设备)
9. [集成 KernelSU](#9-集成-kernelsu)
10. [集成 SUSFS](#10-集成-susfs)
11. [使用 GitHub Actions 云编译](#11-使用-github-actions-云编译)
12. [常见问题与排错](#12-常见问题与排错)
13. [参考资料](#13-参考资料)

---

## 1. GKI 与非 GKI 的区别

### 1.1 什么是 GKI？

**GKI（Generic Kernel Image）** 是 Google 从 Android 12 开始推行的通用内核映像方案，旨在解决 Android 内核碎片化问题。

| 内核类型 | 版本范围 | 说明 |
|---------|---------|------|
| **真正 Non-GKI** | ≤ 4.14 | 完全碎片化，无统一编译方法 |
| **GKI 1.0 / QGKI** | 4.19 ~ 5.4 | 半标准化，5.4 称为 QGKI |
| **GKI 2.0** | 5.10+ | Android 12+ 强制要求 |

### 1.2 判断你的设备使用什么内核

```bash
# 在设备上执行（需要 root）
adb shell "uname -a"
adb shell "zcat /proc/config.gz | grep CONFIG_ANDROID"
```

- 如果 `CONFIG_ANDROID` 存在但不是 GKI 结构 → **Non-GKI**
- 内核版本 ≤ 4.14 → **Non-GKI**
- 内核版本 4.19 ~ 5.4 → **GKI 1.0（仍按非GKI方式处理）**
- 内核版本 ≥ 5.10 + 设备是 Android 12+ → **GKI 2.0**（不在本教程范围）

### 1.3 非GKI内核的特点

- **高度设备相关**：每个 SoC/厂商有各自的 kernel tree、驱动补丁、defconfig
- **没有统一编译方法**：高通、联发科、Exynos、麒麟 各有差异
- **工具链要求不同**：老内核（≤4.14）通常需要 Clang + GCC 4.9 组合
- **碎片化严重**：同一 SoC 的不同设备可能使用不同的内核分支

---

## 2. 准备工作

### 2.1 硬件要求

| 项目 | 最低要求 | 推荐 |
|------|---------|------|
| CPU | 4 核 | 8 核或以上 |
| 内存 | 8 GB | 16 GB 或以上 |
| 磁盘 | 50 GB 空闲 | 100 GB+（SSD） |
| 网络 | 宽带 | 宽带 |

### 2.2 软件环境

**方案 A：本地 Linux（推荐）**

```bash
# Ubuntu 22.04 / 24.04
sudo apt update && sudo apt install -y \
    git git-lfs wget curl \
    build-essential flex bison \
    libssl-dev libncurses-dev \
    python3 python3-pip \
    cpio bc lz4 zstd \
    p7zip-full unzip xz-utils \
    ccache

# 安装 repo 工具
curl -sS https://storage.googleapis.com/git-repo-downloads/repo > /usr/local/bin/repo
chmod a+x /usr/local/bin/repo
```

**方案 B：WSL2（Windows）**

```powershell
# 在 PowerShell（管理员）中
wsl --install -d Ubuntu-22.04
wsl --set-version Ubuntu-22.04 2
```

**方案 C：云编译（推荐新手）**

使用 GitHub Actions 进行云编译，无需本地环境（见第 11 章）。

---

## 3. 获取内核源码

### 3.1 从厂商开源仓库获取

各厂商的内核源码发布地址：

| 厂商 | 源码地址 | 搜索技巧 |
|------|---------|---------|
| **小米/红米** | `github.com/MiCode/Xiaomi_Kernel_OpenSource` | 按设备代号找分支 |
| **OPPO/一加** | `github.com/OneplusOSS` | 搜索 `android_kernel_oneplus_` |
| **三星** | `opensource.samsung.com` | 需解析 release 编号 |
| **高通参考** | `github.com/LineageOS` | 搜索 `android_kernel_` |
| **联发科** | 极少开源 | 搜索 `kernel-4.19` / `kernel-5.4` 分支 |

### 3.2 搜索内核源码的通用方法

```bash
# 方法 1：按设备代号搜索
# 假设设备代号是 "olive"
# 在 GitHub 搜索：
#   kernel olive
#   android_kernel_xiaomi_olive
#   kernel sdm439  (SoC 名)
#   kernel msm8953 (SoC 代号)

# 方法 2：从 LineageOS 设备树找
# 查看设备的 device tree 中的 BoardConfig.mk
wget -qO- https://raw.githubusercontent.com/LineageOS/android_device_X_Y_Z/lineage-XX/BoardConfig.mk \
    | grep TARGET_KERNEL_CONFIG

# 方法 3：从 LineageOS 内核仓库
# 格式：android_kernel_制造商_SoC
# 例如：https://github.com/LineageOS/android_kernel_xiaomi_sdm660
```

### 3.3 从 boot.img 提取内核配置

```bash
# 需要 root 的设备
adb shell "zcat /proc/config.gz" > device_defconfig

# 或从 boot.img 提取
# 先使用 unpack_bootimg.py 或 magiskboot 解包
python3 mkbootimg/unpack_bootimg.py --boot boot.img --out boot_extract
cd boot_extract
# 解压内核
cat kernel | gunzip > kernel_elf
# 提取配置
scripts/extract-ikconfig kernel_elf > extracted_defconfig
```

### 3.4 克隆内核源码

```bash
git clone <你的内核源码地址>
cd <内核目录>

# 查看分支
git branch -a

# 如果使用 LineageOS 源码
git clone https://github.com/LineageOS/android_kernel_xiaomi_sdm660 -b lineage-18.1
```

---

## 4. 获取工具链

### 4.1 工具链选择指南

| 内核版本 | 推荐编译器 | 说明 |
|---------|-----------|------|
| **5.4+** | 仅 Clang | LLVM=1 LLVM_IAS=1 |
| **4.19** | Clang（+ GCC 备用） | 大部分可用 Clang |
| **4.14** | Clang + GCC 4.9 | 需要 GCC 的交叉编译器 |
| **4.9 及以下** | GCC 4.9（或 Clang 旧版） | 老内核兼容性差 |

### 4.2 获取 Clang

```bash
# 方案 1：使用 Google 官方 Clang（推荐）
# Android S：Clang 12.0.5 (r416183b)
# Android T：Clang 14.0.6 (r450784d)
# Android U：Clang 15.0.1 (r458507)
# Android V：Clang 17+

git clone https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86 \
    -b master-kernel-build-2022 --depth 1

# 方案 2：使用 Proton Clang（社区优化，兼容性好）
git clone https://github.com/kdrag0n/proton-clang --depth 1

# 方案 3：使用 AOSP 预编译 Clang（推荐给 GKI）
wget https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/master-kernel-build-2022/clang-r487747c.tar.gz
tar -xzf clang-r487747c.tar.gz -C clang-r487747c/
```

### 4.3 获取 GCC 交叉编译器（老内核需要）

```bash
# 从 AOSP 获取
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9 -b master --depth 1
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9 -b master --depth 1

# 从 Linaro 获取（备选）
wget https://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
tar -xf gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
```

### 4.4 工具链路径设置

```bash
# 设置环境变量（根据你的实际路径修改）

# Clang 路径
export CLANG_PATH=/path/to/prebuilts/clang/host/linux-x86/clang-r450784d/bin
export PATH=$CLANG_PATH:$PATH

# GCC 交叉编译器路径（老内核需要）
export CROSS_COMPILE=/path/to/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-
export CROSS_COMPILE_ARM32=/path/to/arm-linux-androideabi-4.9/bin/arm-linux-androidkernel-

# 如果使用 Linaro 工具链
# export CROSS_COMPILE=/path/to/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-

# 线程数
export THREADS=$(nproc --all)
```

---

## 5. 配置内核（Defconfig）

### 5.1 查找 defconfig

内核的配置文件通常位于以下位置：

```
arch/arm64/configs/<defconfig_name>      # 标准位置
arch/arm64/configs/vendor/<defconfig>    # 厂商自定义位置
arch/arm64/configs/debug/<defconfig>     # 调试配置
```

查找你的 defconfig：

```bash
# 方法 1：从 BoardConfig.mk 查找
grep -r "TARGET_KERNEL_CONFIG" device/

# 方法 2：列出所有 arm64 的 defconfig
ls arch/arm64/configs/ | grep -E "(your_device|vendor)"
ls arch/arm64/configs/vendor/ 2>/dev/null

# 方法 3：常见的命名模式
# gki_defconfig, msm8940_defconfig, sdm660_defconfig, ...
# vendor/xxxx-perf_defconfig, vendor/xxxx_defconfig
```

### 5.2 生成 .config

```bash
# 定义架构
export ARCH=arm64

# 方法 1：直接使用 defconfig（最常用）
make ARCH=arm64 <defconfig_name>

make ARCH=arm64 vendor/sdm660-perf_defconfig
# 或
make ARCH=arm64 O=out vendor/sdm660-perf_defconfig

# 方法 2：使用 O=out（推荐，将编译产物放在 out/ 目录）
# 这样便于清理：rm -rf out 即可
make O=out ARCH=arm64 <defconfig_name>
```

### 5.3 修改内核配置

```bash
# 方法 1：menuconfig（交互式菜单，推荐）
make O=out ARCH=arm64 menuconfig

# 方法 2：直接编辑 .config（适用于简单修改）
# 启用：CONFIG_XXX=y
# 模块：CONFIG_XXX=m
# 禁用：# CONFIG_XXX is not set
vim out/.config

# 方法 3：用 scripts/config 工具（适合脚本化）
# 启用一个选项
scripts/config --file out/.config -e CONFIG_KPROBES
# 禁用一个选项
scripts/config --file out/.config -d CONFIG_SECURITY_DMESG_RESTRICT
# 设置字符串选项
scripts/config --file out/.config --set-str CONFIG_LOCALVERSION "-custom"
# 应用修改
make O=out ARCH=arm64 olddefconfig
```

### 5.4 常用推荐配置（用于 KernelSU 等）

```bash
# 以下配置通常建议启用
scripts/config --file out/.config -e CONFIG_KPROBES
scripts/config --file out/.config -e CONFIG_HAVE_KPROBES
scripts/config --file out/.config -e CONFIG_KPROBE_EVENTS
scripts/config --file out/.config -e CONFIG_NETFILTER_XT_MATCH_COMMENT
scripts/config --file out/.config -e CONFIG_NETFILTER_XT_MATCH_CONNTRACK
scripts/config --file out/.config -e CONFIG_NETFILTER_XT_TARGET_TPROXY

# 可选：安全相关
scripts/config --file out/.config -d CONFIG_SECURITY_DMESG_RESTRICT
scripts/config --file out/.config -e CONFIG_DEBUG_INFO
scripts/config --file out/.config -e CONFIG_IKHEADERS

# 应用修改后重新生成
make O=out ARCH=arm64 olddefconfig
```

### 5.5 从设备提取配置并进行对比

```bash
# 从设备提取当前配置
adb shell "zcat /proc/config.gz" > running_config

# 拷贝你的 .config 到当前目录
cp out/.config my_config

# 使用 diff 对比差异（安装 diffconfig 工具）
# 内核源码中自带的脚本
scripts/diffconfig running_config my_config

# 这将显示你相比设备原厂配置所做的修改
```

### 5.6 保存修改后的 defconfig

```bash
# 将修改后的配置保存回 defconfig
make O=out ARCH=arm64 savedefconfig
# 生成的文件在 out/defconfig

# 覆盖原 defconfig
cp out/defconfig arch/arm64/configs/<你的defconfig>
```

---

## 6. 编译内核

### 6.1 完整编译命令

```bash
# ===== 基础环境设置 =====
export ARCH=arm64
export SUBARCH=arm64
export CLANG_PATH=/path/to/clang/bin
export PATH=$CLANG_PATH:$PATH

# GCC 交叉编译器（视内核版本选择）
export CROSS_COMPILE=/path/to/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-

# ===== 新内核（4.19+）使用纯 Clang =====
make O=out ARCH=arm64 <defconfig>
make O=out \
    ARCH=arm64 \
    CC=clang \
    LD=ld.lld \
    AR=llvm-ar \
    NM=llvm-nm \
    OBJCOPY=llvm-objcopy \
    OBJDUMP=llvm-objdump \
    READELF=llvm-readelf \
    STRIP=llvm-strip \
    LLVM=1 \
    LLVM_IAS=1 \
    -j$(nproc --all)

# ===== 老内核（≤4.14）使用 Clang + GCC =====
make O=out ARCH=arm64 <defconfig>
make O=out \
    ARCH=arm64 \
    CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=aarch64-linux-androidkernel- \
    CROSS_COMPILE_ARM32=arm-linux-androidkernel- \
    -j$(nproc --all)
```

### 6.2 常见编译目标

```bash
# 编译内核镜像
make ... Image.gz                    # gzip 压缩的内核
make ... Image.gz-dtb               # 内嵌 DTB 的内核（最常见）
make ... Image.lz4                  # lz4 压缩的内核（更快解压）
make ... Image.lzma                 # lzma 压缩（更小体积）
make ... Image.xz                   # xz 压缩
make ... zImage                     # 传统方式

# 编译内核 + DTBO
make ... dtbo.img                   # DTBO 分区镜像

# 仅编译 DTB
make ... dtbs                       # 所有设备树
make ... <specific>.dtb            # 单个设备树

# 编译内核模块
make ... modules                    # 编译所有模块
make ... modules_install INSTALL_MOD_PATH=./modules  # 安装模块

# 编译 vendor_dlkm
make ... vendor_dlkm                # vendor DLKM 镜像
```

### 6.3 加速编译

```bash
# 使用 ccache（大幅加速重复编译）
export USE_CCACHE=1
export CCACHE_DIR=/path/to/ccache
ccache -M 50G  # 设置缓存大小

# 首次编译后，后续增量编译会快很多
make O=out ... -j$(nproc --all)

# 使用 Thin LTO（比完全 LTO 快）
# 在 menuconfig 中设置：
#   CONFIG_LTO_CLANG_THIN=y
# 或在 .config 中启用

# 增量编译（仅修改了的部分）
make O=out ... -j$(nproc --all)
```

### 6.4 编译输出

编译完成后，内核映像位于以下位置：

```
out/arch/arm64/boot/
├── Image                   # 未压缩的内核
├── Image.gz                # gzip 压缩
├── Image.gz-dtb            # gzip 压缩 + DTB（最常见输出）
├── Image.lz4               # lz4 压缩
├── Image.lz4-dtb           # lz4 压缩 + DTB
├── dts/                    # 编译好的 DTB 文件
└── dtbo.img                # DTBO 镜像
```

---

## 7. 打包 boot.img

### 7.1 方法 A：使用 mkbootimg（推荐完整打包）

```bash
# 从 AOSP 获取 mkbootimg
git clone https://android.googlesource.com/platform/system/tools/mkbootimg
cd mkbootimg

# 从原厂 boot.img 获取参数
python3 unpack_bootimg.py --boot /path/to/stock/boot.img --out boot_extract

# 查看解包后的参数
cat boot_extract/boot_info.json

# 使用相同参数打包新 boot.img
python3 mkbootimg.py \
    --kernel /path/to/compiled/Image.gz-dtb \
    --ramdisk boot_extract/ramdisk \
    --base 0x80000000 \
    --kernel_offset 0x00008000 \
    --ramdisk_offset 0x01000000 \
    --tags_offset 0x00000100 \
    --pagesize 2048 \
    --cmdline "console=ttyMSM0,115200n8 androidboot.hardware=qcom ..." \
    --header_version 1 \
    -o new-boot.img
```

### 7.2 方法 B：使用 magiskboot 替换内核（简单可靠）

```bash
# 获取 MagiskBoot
# 可以从 Magisk APK 中提取，或使用预编译版本
# 推荐使用 AnyKernel3 的 tools/magiskboot（简化版）

# 1. 解包原厂 boot.img
/path/to/magiskboot unpack /path/to/stock/boot.img

# 解包后得到：
#   kernel     - 原内核
#   ramdisk.cpio - ramdisk
#   dtb        - 设备树（可选）

# 2. 替换内核
cp /path/to/compiled/Image.gz-dtb kernel

# 3. 重新打包
/path/to/magiskboot repack /path/to/stock/boot.img

# 生成：new-boot.img
```

### 7.3 方法 C：使用 AnyKernel3（闪存脚本 + 内核包，推荐）

[AnyKernel3](https://github.com/osm0sis/AnyKernel3) 是一个闪存模板，将内核打包为可刷新的 ZIP 包。

```bash
# 1. 克隆 AnyKernel3
git clone https://github.com/osm0sis/AnyKernel3
cd AnyKernel3

# 2. 替换内核文件
cp /path/to/Image.gz-dtb Image.gz-dtb

# 3. 编辑 anykernel.sh（根据设备修改）
#    - 设置 block 分区路径
#    - 设置 is_slot_device
#    - 设置 device_names

# 4. 打包为 ZIP
zip -r9 ../kernel-anykernel3.zip * -x .git README.md *placeholder
```

典型的 `anykernel.sh` 示例：

```bash
# 设备名称
device.name1=beryllium
device.name2=beryllium-user
device.name3=
device.name4=

# 分区路径（Magisk 会自动检测时可以不设置）
block=/dev/block/bootdevice/by-name/boot

# slot 设备（AB 分区）
is_slot_device=0

# ramdisk 压缩格式
ramdisk_compression=gzip
```

### 7.4 方法 D：直接替换 boot.img 中的内核（Rockchip 方法）

某些 SoC（如 Rockchip）提供了直接替换 boot.img 内核的 make 目标：

```bash
make O=out ARCH=arm64 BOOT_IMG=../boot.img rk356x-boot.img -j$(nproc)
```

---

## 8. 刷入设备

### 8.1 使用 fastboot 刷入

```bash
# 重启到 bootloader
adb reboot bootloader

# 或手动进入 fastboot
# 关机后 音量- + 电源

# 刷入 boot.img
fastboot flash boot new-boot.img

# 如果使用 AB 分区
fastboot flash boot_a new-boot.img
fastboot flash boot_b new-boot.img

# 如果使用 boot_ramdisk
fastboot flash boot_ramdisk new-boot.img

# 清除数据（可选，如果你遇到启动问题）
fastboot erase misc
fastboot reboot
```

### 8.2 刷入 AnyKernel3 ZIP（需要自定义 Recovery）

```bash
# 将 ZIP 推送至设备
adb push kernel-anykernel3.zip /sdcard/

# 重启到 Recovery
adb reboot recovery

# 在 Recovery 中：
# 1. 选择 "Install" / "安装"
# 2. 选择 kernel-anykernel3.zip
# 3. 滑动刷入
# 4. 重启系统
```

### 8.3 注意事项

- **始终先备份原厂 boot.img**：
  ```bash
  dd if=/dev/block/bootdevice/by-name/boot of=/sdcard/stock_boot.img
  ```
- **确认你的设备是否使用 AB 分区**（`fastboot getvar current-slot`）
- **确认 boot.img 头版本**（`header_version`，用 `unpack_bootimg` 查看）
- 如果刷入后卡第一屏或无限重启，fastboot 刷回原厂 boot.img

---

## 9. 集成 KernelSU

### 9.1 什么是 KernelSU

[KernelSU](https://github.com/tiann/KernelSU) 是一个基于内核的 root 方案，通过内核模块实现 root 权限管理。对于非 GKI 内核，有两种集成方式：

### 9.2 方式 A：Kprobe 模式（推荐，适用于 4.14+）

```bash
# 1. 在编译环境中克隆 KernelSU
git clone https://github.com/tiann/KernelSU --depth=1

# 2. 运行集成脚本
cd KernelSU
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main

# 或指定版本
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s v0.9.5

# 3. 手动集成（如果脚本失败）
cd /path/to/kernel-source
ln -sf /path/to/KernelSU/kernel drivers/ksu

# 编辑 drivers/Makefile，添加：
echo "obj-y += ksu/" >> drivers/Makefile

# 4. 在内核配置中启用：
#    CONFIG_KSU=y
#    CONFIG_KPROBES=y
#    CONFIG_HAVE_KPROBES=y
#    CONFIG_KPROBE_EVENTS=y
```

### 9.3 方式 B：手动 Hook 模式（适用于 ≤4.9 老内核）

对于不支持 Kprobe 的老内核，需要手动修改内核源码：

```bash
# 1. 同样克隆 KernelSU
git clone https://github.com/tiann/KernelSU --depth=1

# 2. 复制 hook 文件
cp -r KernelSU/kernel drivers/ksu

# 3. 修改以下内核关键文件：

# a) fs/exec.c - 在 do_execveat_common 函数中添加：
#    #include <linux/ksu.h>
#    在返回值前添加：ksu_handle_execveat_common(...)

# b) fs/open.c - 在 do_faccessat 函数中添加：
#    #include <linux/ksu.h>
#    ksu_handle_faccessat(...)

# c) drivers/input/input.c - 在 input_handle_event 函数中添加：
#    ksu_handle_input_handle_event(...)
```

### 9.4 使用 KernelSU-Next（增强版）

[KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next) 是 KernelSU 的社区增强版：

```bash
# 克隆 KernelSU-Next
git clone https://github.com/KernelSU-Next/KernelSU-Next --depth=1

# 集成方式同上，替换路径即可
ln -sf /path/to/KernelSU-Next/kernel drivers/ksu
```

### 9.5 验证 KernelSU 是否集成成功

编译完成后，检查内核配置：

```bash
grep CONFIG_KSU out/.config
# 应该看到：CONFIG_KSU=y

grep CONFIG_KPROBES out/.config
# 应该看到：CONFIG_KPROBES=y
```

刷入后，安装 KernelSU Manager 应用验证。

---

## 10. 集成 SUSFS

### 10.1 什么是 SUSFS

[SUSFS](https://git.uwu.systems/KernelSU-Next/SUSFS) 是 KernelSU-Next 项目的一部分，提供更高级的隐藏功能（隐藏 root、BL 状态等）。

### 10.2 集成 SUSFS

```bash
# 1. 下载 SUSFS 补丁
git clone https://git.uwu.systems/KernelSU-Next/SUSFS.git --depth=1

# 2. 应用补丁（根据你的内核版本）
cd /path/to/kernel-source

# 方式一：使用 SUSFS 提供的 apply_susfs_patch.sh（如果存在）
./SUSFS/apply_susfs_patch.sh

# 方式二：手动打补丁
# 查看 SUSFS/patches/ 目录中的补丁文件
# 使用 patch 命令应用
for patch in /path/to/SUSFS/patches/*.patch; do
    patch -p1 < "$patch"
done

# 3. 复制 SUSFS 内核模块到内核源码树
cp -r /path/to/SUSFS/kernel drivers/susfs
# 编辑 drivers/Makefile，添加：
echo "obj-y += susfs/" >> drivers/Makefile

# 4. 在内核配置中启用 SUSFS
scripts/config --file out/.config -e CONFIG_KSU_SUSFS
scripts/config --file out/.config -e CONFIG_KSU_SUSFS_HAS_MISC

# 确保也启用了：
scripts/config --file out/.config -e CONFIG_KSU
scripts/config --file out/.config -e CONFIG_KPROBES

make O=out ARCH=arm64 olddefconfig
```

### 10.3 SUSFS 常用配置选项

```bash
# SUSFS 功能配置（在 menuconfig 中的位置）
CONFIG_KSU_SUSFS=y                    # 主开关
CONFIG_KSU_SUSFS_HAS_MISC=y          # 杂项功能
CONFIG_KSU_SUSFS_SUS_PATH=y          # 路径隐藏
CONFIG_KSU_SUSFS_SUS_MOUNT=y         # 挂载隐藏
CONFIG_KSU_SUSFS_AUTO_ADD_GENERIC=y  # 自动添加通用隐藏规则
CONFIG_KSU_SUSFS_SUS_KSTAT=y         # kstat 隐藏
CONFIG_KSU_SUSFS_SUS_BINDER=y        # binder 隐藏
```

---

## 11. 使用 GitHub Actions 云编译

### 11.1 推荐模板

| 项目 | 说明 | 链接 |
|------|------|------|
| **kernel_build_action** | 通用 Kbuild Action，支持 KSU/APatch | [dabao1955/kernel_build_action](https://github.com/dabao1955/kernel_build_action) |
| **LXC_KernelSU_Action** | 支持多内核版本 + 自定义 Clang/GCC | [tappsoft/LXC_KernelSU_Action](https://github.com/tappsoft/LXC_KernelSU_Action) |
| **NonGKI_Kernel_Build_mix2s** | 带 KSU + SUSFS 自动编译 | [NootFond/NonGKI_Kernel_Build_mix2s](https://github.com/NootFond/NonGKI_Kernel_Build_mix2s) |
| **KernelSU-Next-Builder** | 一键生成非GKI内核 + KernelSU-Next | [nikobakinec/KernelSU-Next-Builder-](https://github.com/nikobakinec/KernelSU-Next-Builder-) |

### 11.2 工作流配置示例

这里是一个完整的 GitHub Actions 工作流示例（基于 kernel_build_action）：

```yaml
name: Build Non-GKI Kernel
on:
  workflow_dispatch:
    inputs:
      KERNEL_SOURCE:
        description: '内核源码 Git 地址'
        required: true
        default: 'https://github.com/LineageOS/android_kernel_xiaomi_sdm660'
      KERNEL_BRANCH:
        description: '内核分支'
        required: true
        default: 'lineage-18.1'
      DEFCONFIG_NAME:
        description: 'Defconfig 名称'
        required: true
        default: 'vendor/sdm660-perf_defconfig'
      CLANG_VERSION:
        description: 'Clang 版本'
        required: true
        default: 'clang-r450784d'
      ENABLE_KSU:
        description: '启用 KernelSU'
        type: boolean
        default: true
      ENABLE_SUSFS:
        description: '启用 SUSFS'
        type: boolean
        default: false

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt update
          sudo apt install -y git wget curl \
            build-essential flex bison cpio bc \
            libssl-dev libncurses-dev python3 \
            lz4 zstd ccache

      - name: Clone kernel source
        run: |
          git clone ${{ github.event.inputs.KERNEL_SOURCE }} \
            --depth=1 -b ${{ github.event.inputs.KERNEL_BRANCH }} kernel
          cd kernel

      - name: Setup toolchain
        run: |
          mkdir -p toolchain
          # 下载 Clang
          cd toolchain
          wget https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/master-kernel-build-2022/${{ github.event.inputs.CLANG_VERSION }}.tar.gz
          mkdir clang && tar -xf *.tar.gz -C clang/

          # 下载 GCC 4.9
          git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9 --depth=1

      - name: Configure kernel
        run: |
          cd kernel
          export ARCH=arm64
          export PATH=$(pwd)/../toolchain/clang/bin:$PATH
          export CROSS_COMPILE=$(pwd)/../toolchain/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-

          make O=out ARCH=arm64 CC=clang \
            CLANG_TRIPLE=aarch64-linux-gnu- \
            ${{ github.event.inputs.DEFCONFIG_NAME }}

      - name: Integrate KernelSU
        if: ${{ github.event.inputs.ENABLE_KSU == 'true' }}
        run: |
          cd kernel
          git clone https://github.com/tiann/KernelSU --depth=1
          cd KernelSU
          curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main
          cd ..
          # 启用 KSU 相关配置
          scripts/config --file out/.config -e CONFIG_KSU
          scripts/config --file out/.config -e CONFIG_KPROBES
          scripts/config --file out/.config -e CONFIG_HAVE_KPROBES
          make O=out ARCH=arm64 olddefconfig

      - name: Build kernel
        run: |
          cd kernel
          export ARCH=arm64
          export PATH=$(pwd)/../toolchain/clang/bin:$PATH
          export CROSS_COMPILE=$(pwd)/../toolchain/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-

          make O=out ARCH=arm64 \
            CC=clang CLANG_TRIPLE=aarch64-linux-gnu- \
            CROSS_COMPILE=$CROSS_COMPILE \
            -j$(nproc --all)

      - name: Package with AnyKernel3
        run: |
          git clone https://github.com/osm0sis/AnyKernel3
          cp kernel/out/arch/arm64/boot/Image.gz-dtb AnyKernel3/
          cd AnyKernel3
          zip -r9 ../kernel-anykernel3.zip * -x .git README.md

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: non-gki-kernel
          path: kernel-anykernel3.zip
```

### 11.3 使用 Action 模板快速开始

```yaml
# 在你的仓库中创建 .github/workflows/build-kernel.yml
name: Build Non-GKI Kernel

on:
  workflow_dispatch:
    inputs:
      KERNEL_SOURCE:
        description: 'Kernel Source URL'
        required: true

jobs:
  build:
    uses: dabao1955/kernel_build_action/.github/workflows/build.yml@master
    with:
      KERNEL_SOURCE: ${{ github.event.inputs.KERNEL_SOURCE }}
      ENABLE_KSU: true
    secrets:
      TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
```

---

## 12. 常见问题与排错

### 12.1 编译错误

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `arm-linux-gnueabi-: not found` | 缺少 32 位交叉编译器 | 设置 `CROSS_COMPILE_ARM32`，或安装对应 GCC |
| `undefined reference to xxx` | KSU/SUSFS 补丁冲突或版本不匹配 | 使用匹配的 KernelSU 分支 + SUSFS 补丁集 |
| `implicit function declaration` | 内核版本太老，某些函数未定义 | 切换到手动 Hook 模式，或使用旧版 KernelSU |
| `ld.lld: error: undefined symbol` | 链接器问题 | 尝试使用 `LD=ld.bfd` 代替 `LD=ld.lld` |
| `scripts/config: line xxx: not found` | 没有在正确的目录运行 | 确保在内核源码根目录运行，并且已安装 `bc` |
| `No rule to make target 'debian/canonical-certs.pem'` | 证书文件缺失 | 在 .config 中禁用：`CONFIG_SYSTEM_TRUSTED_KEYS=""` |
| `cc1: error: unknown -mno-outline-atomics` | Clang 版本不兼容 | 更换 Clang 版本或使用 GCC |
| `KASAN: null-ptr-deref` | 内核运行时崩溃 | 检查 defconfig 是否遗漏了必要的 SoC 驱动配置 |

### 12.2 启动问题

| 现象 | 可能原因 | 解决方案 |
|------|---------|---------|
| **卡在第一屏（bootlogo）** | 内核无法启动 | 检查 DTB 是否正确，确认 defconfig 是否完整 |
| **无限重启（bootloop）** | 内核或驱动问题 | 用 fastboot 刷回原厂 boot.img |
| **触屏/ WiFi 不工作** | 缺少相关驱动 | 检查 defconfig 中对应驱动是否启用 |
| **KernelSU Manager 显示未安装** | KSU 未正确集成 | 确认 `CONFIG_KSU=y`，检查 /proc/kallsyms |
| **SUSFS 功能不可用** | 补丁未正确应用 | 检查 SUSFS 配置是否启用，补丁是否适配内核版本 |
| **无法解密 data 分区** | FBE/FDE 密钥问题 | 确保 `CONFIG_FS_ENCRYPTION=y`，检查密钥描述符 |

### 12.3 提取内核日志

```bash
# 正常启动后
adb shell dmesg > kernel.log

# 设备无法完全启动时（通过 bootloader 模式）
# 使用 serial console（如果有）
# 或查看 last_kmsg
adb shell cat /proc/last_kmsg > last_kmsg.log

# KernelSU 相关日志
adb shell dmesg | grep -i ksu
adb shell dmesg | grep -i susfs
adb shell lsmod  # 查看已加载模块
```

### 12.4 调试建议

```bash
# 1. 启用更详细的日志
scripts/config --file out/.config -e CONFIG_DEBUG_KERNEL
scripts/config --file out/.config -e CONFIG_DYNAMIC_DEBUG

# 2. 保持串口输出
# 在 cmdline 中添加：
#   console=ttyMSM0,115200n8
#   (根据你的 SoC 调整)

# 3. 保留未压缩的内核（方便调试）
# 镜像在 out/arch/arm64/boot/Image（未压缩）

# 4. 使用 GDB 调试（如有条件）
# 编译时添加：
make O=out ... -g
```

---

## 13. 参考资料

### 官方文档

- [AOSP Kernel Build Guide](https://source.android.com/docs/setup/build/building-kernels) — Google 官方内核编译文档
- [Android Generic Kernel Image (GKI)](https://source.android.com/docs/core/architecture/kernel) — GKI 架构说明
- [KernelSU 官方文档](https://kernelsu.org/guide/how-to-integrate-for-non-gki.html) — KernelSU 非 GKI 集成指南

### 社区资源

- [KernelSU GitHub](https://github.com/tiann/KernelSU) — KernelSU 主项目
- [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next) — KernelSU 社区增强版
- [SUSFS](https://git.uwu.systems/KernelSU-Next/SUSFS) — SUSFS 隐藏功能模块
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3) — 通用内核刷写模板
- [kernel_build_action](https://github.com/dabao1955/kernel_build_action) — GitHub Actions 通用构建模板
- [Proton Clang](https://github.com/kdrag0n/proton-clang) — 社区 Clang 工具链
- [Android-Kernel-Tutorials](https://github.com/st-rnd/ravindu644_Android-Kernel-Tutorials) — 设备内核编译新手教程（英文）

### 中文资源

- [酷安：从零开始云构建你的非GKI内核](https://www.coolapk.com/feed/68364847) — 小白向图文教程
- [Android内核编译基础教程](https://github.com/feichaixiaobai/Android-kernel-make-teach/discussions/1) — Ubuntu 环境编译 Clang/GCC 入门
- [非GKI设备内核集成KernelSU完全指南](https://blog.gitcode.com/e443bf73aa2e0a2869f8c10fdd6b5234.html) — 从问题诊断到编译验证
- [NetHunter 内核编译指南](https://github.com/akabul0us/So_You_Want_To_Build_A_Nethunter_Kernel) — 面向 NetHunter 的内核构建教程

### 工具下载

| 工具 | 用途 | 地址 |
|------|------|------|
| Google Clang 预编译 | 编译器 | [Android Clang](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/) |
| GCC 4.9 交叉编译器 | 老内核编译器 | [AOSP GCC 4.9](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/) |
| mkbootimg | boot.img 打包工具 | [AOSP mkbootimg](https://android.googlesource.com/platform/system/tools/mkbootimg) |
| MagiskBoot | boot.img 解包/重包 | [Magisk](https://github.com/topjohnwu/Magisk) 中包含 |
| AnyKernel3 | 刷写 ZIP 模板 | [AnyKernel3](https://github.com/osm0sis/AnyKernel3) |
| Android Image Kitchen | boot.img 解包工具（Win） | [XDA 原帖](https://forum.xda-developers.com/showthread.php?t=2073775) |

---

## 快速参考命令速查表

```bash
# ===== 完整编译流程（4.19+ 内核，纯 Clang） =====
export PATH=/path/to/clang/bin:$PATH
export ARCH=arm64
export CROSS_COMPILE=/path/to/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-
make O=out ARCH=arm64 vendor/sdm660-perf_defconfig
make O=out ARCH=arm64 CC=clang LD=ld.lld LLVM=1 LLVM_IAS=1 -j$(nproc) Image.gz-dtb

# ===== 完整编译流程（≤4.14 内核，Clang + GCC） =====
export PATH=/path/to/clang/bin:$PATH
export ARCH=arm64
export CROSS_COMPILE=/path/to/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-
export CROSS_COMPILE_ARM32=/path/to/arm-linux-androideabi-4.9/bin/arm-linux-androidkernel-
make O=out ARCH=arm64 CC=clang CLANG_TRIPLE=aarch64-linux-gnu- <defconfig>
make O=out ARCH=arm64 CC=clang CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=$CROSS_COMPILE -j$(nproc)

# ===== KernelSU 集成 =====
git clone https://github.com/tiann/KernelSU --depth=1
cd KernelSU && curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash

# ===== boot.img 打包 =====
/path/to/magiskboot unpack stock_boot.img
cp Image.gz-dtb kernel
/path/to/magiskboot repack stock_boot.img  # → new-boot.img

# ===== 刷入 =====
fastboot flash boot new-boot.img
fastboot reboot
```

---

> **免责声明**：编译自定义内核可能导致设备变砖、数据丢失或保修失效。请始终做好备份，并确保你了解如何在紧急情况下恢复设备（fastboot 刷回原厂 boot.img）。本教程仅提供技术指导，不承担任何因操作不当造成的损失。
