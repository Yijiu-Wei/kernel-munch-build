# Redmi K40S (munch) 非GKI内核编译教程

> **设备**：Redmi K40S / POCO F4  
> **代号**：munch (munchin)  
> **SoC**：Qualcomm SM8250-AC (Snapdragon 870)  
> **内核**：Linux 4.19.325 (CLO rebased)  
> **目标系统**：HyperOS / MIUI / AOSP Android 14  
> **集成组件**：KernelSU + DroidSpaces  
> **最后更新**：2026-06-20

---

## 📖 目录

1. [准备工作与资源](#1-准备工作与资源)
2. [获取内核源码](#2-获取内核源码)
3. [了解内核配置结构](#3-了解内核配置结构)
4. [搭建编译环境](#4-搭建编译环境)
5. [配置内核](#5-配置内核)
6. [集成 KernelSU](#6-集成-kernelsu)
7. [集成 DroidSpaces](#7-集成-droidspaces)
8. [编译内核](#8-编译内核)
9. [打包 boot.img / AnyKernel3](#9-打包-bootimg--anykernel3)
10. [刷入设备](#10-刷入设备)
11. [验证](#11-验证)
12. [GitHub Actions 自动化云编译](#12-github-actions-自动化云编译)
13. [常见问题与排错](#13-常见问题与排错)
14. [参考资料](#14-参考资料)

---

## 1. 准备工作与资源

### 1.1 设备信息速查

```bash
# 确认你的设备
adb shell getprop ro.product.board        # → msmnile (SM8250)
adb shell getprop ro.product.device       # → munch
adb shell getprop ro.build.version.sdk    # → 34 (Android 14)
adb shell uname -r                        # → 4.19.157-perf+
```

### 1.2 所需资源一览

| 资源 | 地址 |
|------|------|
| **内核源码**（推荐） | [yspbwx2010/kernel_xiaomi_sm8250_mod](https://github.com/yspbwx2010/kernel_xiaomi_sm8250_mod) — `android14-lineage22-mod` 分支 |
| **备选内核源码** | [Dowsion-dev/mi10s](https://github.com/Dowsion-dev/mi10s) |
| **KernelSU** | [tiann/KernelSU](https://github.com/tiann/KernelSU) |
| **DroidSpaces** | [ravindu644/Droidspaces-OSS](https://github.com/ravindu644/Droidspaces-OSS) |
| **AnyKernel3** | [osm0sis/AnyKernel3](https://github.com/osm0sis/AnyKernel3) |
| **Clang 工具链** | [Google Clang (AOSP)](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/) |
| **GCC 4.9 交叉编译器** | [AOSP GCC 4.9](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/) |

---

## 2. 获取内核源码

### 2.1 克隆推荐的内核源码

```bash
# 克隆 yspbwx2010 的修改版内核（推荐，已修复1%电量bug，内置KernelSU支持）
git clone https://github.com/yspbwx2010/kernel_xiaomi_sm8250_mod \
    -b android14-lineage22-mod \
    --depth=1 \
    kernel_xiaomi_sm8250

cd kernel_xiaomi_sm8250
```

> 注意：该仓库已于 2026-04-05 归档，但源码仍可正常使用。

### 2.2 备选：使用其他内核源码

```bash
# 备选 1：Dowsion-dev 的内核（支持更多USB串口驱动）
git clone https://github.com/Dowsion-dev/mi10s -b main --depth=1

# 备选 2：crDroid 的 munch 专用内核
git clone https://github.com/crdroidandroid/android_kernel_xiaomi_sm8250-munch -b 14.0 --depth=1

# 备选 3：官方 LineageOS 内核（已归档）
git clone https://github.com/UtsavBalar1231/kernel_xiaomi_sm8250 -b fourteen --depth=1
```

### 2.3 确认内核版本

```bash
head -5 Makefile
# 输出：
# VERSION = 4
# PATCHLEVEL = 19
# SUBLEVEL = 325
# NAME = "People's Front"

# 确认 defconfig 存在
ls arch/arm64/configs/munch_defconfig
```

---

## 3. 了解内核配置结构

该内核使用**分层配置**结构，理解这一点很重要：

```
arch/arm64/configs/
├── munch_defconfig          ← 主 defconfig（munch 设备）
├── munch_stock-defconfig    ← 原厂风格配置（备用）
├── vendor/
│   └── xiaomi/
│       ├── sm8250-common.config  ← SM8250 平台公共配置
│       ├── munch.config          ← munch 设备专有配置
│       ├── alioth.config
│       └── ...
├── AOSP_defconfig
└── gki_defconfig
```

`munch_defconfig` 是**最终完整的 defconfig**，它已经包含了 vendor 片段中的相关选项。

### 当前配置中已包含的关键功能

该内核源码已经预置了很多功能，检查 `munch_defconfig`：

```bash
grep -E "(CONFIG_KSU|CONFIG_KPROBES|CONFIG_OVERLAY_FS|CONFIG_NAMESPACES|CONFIG_CGROUP)" \
    arch/arm64/configs/munch_defconfig
```

你应该能看到这些已经很完善——因为 yspbwx2010 的仓库**已经集成了 KernelSU 支持**（部分版本）。我们后续要加上 DroidSpaces 所需的配置。

---

## 4. 搭建编译环境

### 4.1 安装系统依赖

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
```

### 4.2 下载 Clang 工具链

Redmi K40S 的 4.19 内核推荐使用 **Clang 14+**（Android T 以上版本）：

```bash
mkdir -p toolchain && cd toolchain

# 方式 1：下载 Google 官方 Clang 14 (r450784d) — 推荐
wget https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/master-kernel-build-2022/clang-r450784d.tar.gz
mkdir clang && tar -xf clang-r450784d.tar.gz -C clang/

# 方式 2：使用 Proton Clang（社区优化版）
git clone https://github.com/kdrag0n/proton-clang --depth=1

# 方式 3：下载 Google 官方 Clang 15 (r458507)
wget https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/master-kernel-build-2022/clang-r458507.tar.gz
mkdir clang15 && tar -xf clang-r458507.tar.gz -C clang15/
```

### 4.3 下载 GCC 4.9 交叉编译器（老内核兼容）

```bash
# 虽然 4.19 可以使用纯 Clang，但 SM8250 的内核需要 GCC 交叉编译器
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9 --depth=1

# 32位交叉编译器（某些驱动需要）
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9 --depth=1

cd ..
```

### 4.4 设置编译环境变量

创建一个环境设置脚本 `env.sh`，方便重复使用：

```bash
cat > env.sh << 'EOF'
#!/bin/bash
# Redmi K40S (munch) 内核编译环境

export ARCH=arm64
export SUBARCH=arm64

# 工具链路径（根据你的实际路径修改！）
export CLANG_PATH=$(pwd)/toolchain/clang/bin
export PATH=$CLANG_PATH:$PATH

# GCC 交叉编译器
export CROSS_COMPILE=$(pwd)/toolchain/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-

# 如果使用 Proton Clang 或 Clang 15，修改上面的 CLANG_PATH 即可

# 线程数
export THREADS=$(nproc --all)

# ccache 加速
export USE_CCACHE=1
export CCACHE_DIR=$(pwd)/.ccache

echo "环境已设置：ARCH=$ARCH, CLANG_PATH=$CLANG_PATH, THREADS=$THREADS"
EOF

chmod +x env.sh
```

---

## 5. 配置内核

### 5.1 生成基础 .config

```bash
# 加载环境变量
source env.sh

# 使用 munch_defconfig 生成 .config
make O=out ARCH=arm64 CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    munch_defconfig
```

### 5.2 对比原厂配置（可选）

如果你希望对比设备当前的运行配置：

```bash
# 从设备提取当前配置
adb shell "zcat /proc/config.gz" > running_config

# 使用内核源码中的 diffconfig 工具对比
scripts/diffconfig running_config out/.config > config_diff.txt

# 查看差异
head -50 config_diff.txt
```

### 5.3 使用 menuconfig 查看/修改配置

```bash
make O=out ARCH=arm64 CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    menuconfig
```

在 menuconfig 中可以检查以下关键配置是否已启用：

- **KernelSU** → `CONFIG_KSU`（如果源码已预集成）
- **命名空间** → `CONFIG_NAMESPACES`, `CONFIG_PID_NS` 等
- **CGroup** → `CONFIG_CGROUPS`, `CONFIG_CGROUP_DEVICE` 等
- **OverlayFS** → `CONFIG_OVERLAY_FS`
- **Netfilter** → `CONFIG_NETFILTER`, `CONFIG_NF_NAT` 等

### 5.4 保存修改后的配置

```bash
# 保存为新的 defconfig
make O=out ARCH=arm64 savedefconfig
cp out/defconfig arch/arm64/configs/munch_defconfig
```

---

## 6. 集成 KernelSU

### 6.1 检查是否已集成

yspbwx2010 的仓库可能已经预集成了 KernelSU。检查：

```bash
# 检查是否有 KernelSU 的源文件
ls -la drivers/ksu/ 2>/dev/null

# 检查 defconfig 中是否已启用
grep CONFIG_KSU arch/arm64/configs/munch_defconfig
grep CONFIG_KPROBES arch/arm64/configs/munch_defconfig
```

如果已经存在 `drivers/ksu/` 目录并且 `CONFIG_KSU=y`，可以跳过此章节。

### 6.2 全新集成 KernelSU

```bash
# 1. 克隆 KernelSU
cd kernel_xiaomi_sm8250
git clone https://github.com/tiann/KernelSU --depth=1

# 2. 运行集成脚本
cd KernelSU
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main
cd ..

# 3. 如果脚本失败，手动集成
ln -sf $(pwd)/KernelSU/kernel drivers/ksu

# 确保 drivers/Makefile 包含：
echo "obj-y += ksu/" >> drivers/Makefile
```

### 6.3 启用 KernelSU 配置

将以下选项添加到 `munch_defconfig` 或使用 menuconfig 启用：

```bash
scripts/config --file out/.config -e CONFIG_KSU

# 确保 Kprobe 已启用（KernelSU 依赖）
scripts/config --file out/.config -e CONFIG_KPROBES
scripts/config --file out/.config -e CONFIG_HAVE_KPROBES
scripts/config --file out/.config -e CONFIG_KPROBE_EVENTS

# 重新生成配置
make O=out ARCH=arm64 olddefconfig
```

如果需要检查 KernelSU 是否真的工作，可以在编译后执行：

```bash
grep CONFIG_KSU out/.config
# 必须输出：CONFIG_KSU=y
```

---

## 7. 集成 DroidSpaces

### 7.1 DroidSpaces 内核要求

DroidSpaces 需要内核开启**命名空间（Namespaces）、CGroup、Seccomp、OverlayFS** 等功能。对于 4.19 非 GKI 内核，官方文档称"添加配置片段即可即插即用"。

### 7.2 添加 DroidSpaces 必选配置

将以下 DroidSpaces 配置片段追加到你的 defconfig 或直接修改 `.config`：

```bash
# 创建一个 DroidSpaces 配置片段文件
cat > droidspaces.config << 'EOF'
# ===== DroidSpaces 必选内核配置 (Non-GKI 4.19) =====
# IPC 与系统控制
CONFIG_SYSCTL=y
CONFIG_SYSVIPC=y
CONFIG_POSIX_MQUEUE=y

# 命名空间（核心容器支持 — 缺一不可！）
CONFIG_NAMESPACES=y
CONFIG_PID_NS=y
CONFIG_UTS_NS=y
CONFIG_IPC_NS=y

# Seccomp（安全沙箱）
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y

# CGroup（资源隔离）
CONFIG_CGROUPS=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_PIDS=y
CONFIG_MEMCG=y
CONFIG_CGROUP_SCHED=y
CONFIG_FAIR_GROUP_SCHED=y
CONFIG_CGROUP_FREEZER=y
CONFIG_CGROUP_NET_PRIO=y

# 设备与文件系统
CONFIG_DEVTMPFS=y
CONFIG_OVERLAY_FS=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_TMPFS_XATTR=y

# 固件加载
CONFIG_FW_LOADER=y
CONFIG_FW_LOADER_USER_HELPER=y
CONFIG_FW_LOADER_COMPRESS=y

# 网络命名空间与 NAT（网络隔离支持）
CONFIG_NET_NS=y
CONFIG_VETH=y
CONFIG_BRIDGE=y
CONFIG_NETFILTER=y
CONFIG_BRIDGE_NETFILTER=y
CONFIG_NETFILTER_ADVANCED=y
CONFIG_NF_CONNTRACK=y
CONFIG_IP_NF_IPTABLES=y
CONFIG_IP_NF_FILTER=y
CONFIG_NF_NAT=y
CONFIG_NF_TABLES=y
CONFIG_IP_NF_TARGET_MASQUERADE=y
CONFIG_NETFILTER_XT_TARGET_MASQUERADE=y
CONFIG_NETFILTER_XT_TARGET_TCPMSS=y
CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
CONFIG_NF_CONNTRACK_NETLINK=y
CONFIG_NF_NAT_REDIRECT=y
CONFIG_IP_ADVANCED_ROUTER=y
CONFIG_IP_MULTIPLE_TABLES=y

# 传统内核兼容层（4.19 需要）
CONFIG_NF_CONNTRACK_IPV4=y
CONFIG_NF_NAT_IPV4=y
CONFIG_IP_NF_NAT=y

EOF

# 将 DroidSpaces 配置片段合并到当前 .config
# 方法 1：直接追加到 defconfig
cat droidspaces.config >> arch/arm64/configs/munch_defconfig

# 方法 2：或者写入临时文件并合并
# cat droidspaces.config >> out/.config

# 重新生成配置
make O=out ARCH=arm64 olddefconfig
```

### 7.3 禁用 Android Paranoid Network

DroidSpaces 明确要求**禁用** `CONFIG_ANDROID_PARANOID_NETWORK`，否则容器内无法联网：

```bash
scripts/config --file out/.config -d CONFIG_ANDROID_PARANOID_NETWORK

# 重新生成
make O=out ARCH=arm64 olddefconfig
```

### 7.4 应用 DroidSpaces 内核补丁

DroidSpaces 为非 GKI 内核提供了两个必要补丁：

#### 补丁 1：修复 xt_qtaguid 内核崩溃

```bash
# 下载补丁
wget -O patches/fix_xt_qtaguid_panic.patch \
    https://raw.githubusercontent.com/ravindu644/Droidspaces-OSS/master/Documentation/resources/kernel-patches/non-GKI/01.fix_kernel_panic_in_xt_qtaguid.patch

# 应用补丁
patch -p1 < patches/fix_xt_qtaguid_panic.patch
```

此补丁修复了在 LXC/DroidSpaces 环境下，在不活跃网络接口上调用 `dev_get_stats` 导致的内核崩溃问题。它修改 `net/netfilter/xt_qtaguid.c`，在不活跃接口上直接使用空统计结构体而不是尝试获取统计信息。

#### 补丁 2：修复 CGroup 文件前缀处理

```bash
# 下载补丁
wget -O patches/fix_cgroup_prefix.patch \
    "https://raw.githubusercontent.com/ravindu644/Droidspaces-OSS/master/Documentation/resources/kernel-patches/non-GKI/02.fix_restore%20cgroup%20file%20prefix%20handling%20.patch"

# 应用补丁
patch -p1 < patches/fix_cgroup_prefix.patch
```

此补丁修改 `kernel/cgroup/cgroup.c`，当 cgroup 根节点设置了 `CGRP_ROOT_NOPREFIX` 标志时，创建 `"子系统名.文件名"` 格式的符号链接，恢复旧版本内核的 cgroup 文件命名行为，确保 DroidSpaces/LXC 的兼容性。

### 7.5 验证 DroidSpaces 配置完整性

```bash
# 检查关键配置
echo "=== DroidSpaces 关键配置检查 ==="
for cfg in \
    CONFIG_NAMESPACES CONFIG_PID_NS CONFIG_UTS_NS CONFIG_IPC_NS \
    CONFIG_CGROUPS CONFIG_CGROUP_DEVICE CONFIG_CGROUP_PIDS \
    CONFIG_MEMCG CONFIG_DEVTMPFS CONFIG_OVERLAY_FS \
    CONFIG_SECCOMP CONFIG_SECCOMP_FILTER \
    CONFIG_NET_NS CONFIG_VETH CONFIG_BRIDGE \
    CONFIG_NF_NAT CONFIG_IP_NF_IPTABLES \
    CONFIG_POSIX_MQUEUE CONFIG_SYSVIPC \
    CONFIG_TMPFS_POSIX_ACL CONFIG_TMPFS_XATTR; do
    if grep -q "${cfg}=y" out/.config; then
        echo "✅ $cfg"
    else
        echo "❌ $cfg — 缺少！"
    fi
done

# 检查必须禁用的配置
echo ""
echo "=== 必须禁用的配置检查 ==="
if grep -q "CONFIG_ANDROID_PARANOID_NETWORK=y" out/.config; then
    echo "❌ CONFIG_ANDROID_PARANOID_NETWORK — 仍未禁用！容器内将无法联网"
else
    echo "✅ CONFIG_ANDROID_PARANOID_NETWORK 已禁用"
fi
```

---

## 8. 编译内核

### 8.1 一次性完整编译

```bash
source env.sh

# 清理之前的编译（如果是首次编译可以跳过）
make O=out clean 2>/dev/null
make O=out mrproper 2>/dev/null

# Step 1: 配置
make O=out ARCH=arm64 CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    munch_defconfig

# 如果还没有应用 DroidSpaces 配置，现在应用
cat droidspaces.config >> out/.config 2>/dev/null
make O=out ARCH=arm64 olddefconfig

# Step 2: 编译内核
make O=out ARCH=arm64 \
    CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    -j$(nproc --all) \
    Image.gz-dtb 2>&1 | tee build.log
```

### 8.2 分步编译（调试用）

```bash
source env.sh

# 仅编译内核映像（不编译 modules、dtbo 等）
make O=out ARCH=arm64 CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    -j$(nproc --all) \
    Image.gz-dtb

# 单独编译 dtbo（如果需要）
make O=out ARCH=arm64 CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    -j$(nproc --all) \
    dtbo.img

# 编译内核模块（如果需要）
make O=out ARCH=arm64 CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    -j$(nproc --all) \
    modules
```

### 8.3 编译输出

编译成功后，产物位于：

```
out/arch/arm64/boot/
├── Image                   # 未压缩内核 (~38MB)
├── Image.gz                # gzip 压缩 (~14MB)
├── Image.gz-dtb            # ✅ 内嵌 DTB 的内核（我们需要的）
└── dts/                    # 编译好的设备树
```

### 8.4 编译加速技巧

```bash
# 使用 ccache（首次编译后大幅加速）
export USE_CCACHE=1
export CCACHE_DIR=$(pwd)/.ccache
ccache -M 20G   # 设置 20GB 缓存

# 增量编译（仅编译修改的部分）
make O=out ... -j$(nproc --all)

# 只编译特定文件（调试驱动时）
make O=out ... drivers/usb/dwc3/
```

---

## 9. 打包 boot.img / AnyKernel3

### 9.1 方法 A：使用 AnyKernel3（推荐）

AnyKernel3 会生成一个 ZIP 包，可在 Recovery 中刷入：

```bash
# 1. 克隆 AnyKernel3
cd ..
git clone https://github.com/osm0sis/AnyKernel3
cd AnyKernel3

# 2. 复制编译好的内核
cp ../kernel_xiaomi_sm8250/out/arch/arm64/boot/Image.gz-dtb Image.gz-dtb

# 3. 编辑 anykernel.sh — 设置 munch 设备
cat > anykernel.sh << 'EOF'
# Redmi K40S / POCO F4 (munch)
device.name1=munch
device.name2=munchin
device.name3=RedmiK40S
device.name4=POCOF4
block=/dev/block/bootdevice/by-name/boot
is_slot_device=auto
ramdisk_compression=gzip
EOF

# 4. 打包为 ZIP
zip -r9 ../RedmiK40S-Kernel-AnyKernel3.zip * -x .git README.md *placeholder

# 回到内核目录
cd ../kernel_xiaomi_sm8250/
```

### 9.2 方法 B：使用 MagiskBoot 打包 boot.img

需要有**原厂 boot.img**：

```bash
# 1. 从手机提取 boot.img
adb shell "su -c 'dd if=/dev/block/bootdevice/by-name/boot of=/sdcard/stock_boot.img'"
adb pull /sdcard/stock_boot.img

# 2. 获取 MagiskBoot
# 可以从 Magisk APK 中提取
# 或从 AnyKernel3 中使用
cp ../AnyKernel3/tools/magiskboot ./

# 3. 解包
./magiskboot unpack stock_boot.img
# 解包后你会看到：
#   kernel       - 原厂内核
#   ramdisk.cpio - ramdisk
#   dtb          - 设备树（可选）

# 4. 替换内核
cp out/arch/arm64/boot/Image.gz-dtb kernel

# 5. 重新打包
./magiskboot repack stock_boot.img
# 生成：new-boot.img
```

### 9.3 方法 C：使用 mkbootimg（精确控制）

```bash
# 获取 mkbootimg
git clone https://android.googlesource.com/platform/system/tools/mkbootimg -b master --depth=1

# 从原厂 boot.img 提取参数
python3 mkbootimg/unpack_bootimg.py --boot stock_boot.img --out boot_extract
cat boot_extract/boot_info.json

# 重新打包（使用提取的参数）
python3 mkbootimg/mkbootimg.py \
    --kernel out/arch/arm64/boot/Image.gz-dtb \
    --ramdisk boot_extract/ramdisk \
    --base 0x80000000 \
    --kernel_offset 0x00008000 \
    --ramdisk_offset 0x01000000 \
    --tags_offset 0x00000100 \
    --pagesize 4096 \
    --header_version 2 \
    --cmdline "console=ttyMSM0,115200n8 earlycon=msm_geni_serial,0x888000 androidboot.hardware=qcom androidboot.console=ttyMSM0 video=vfb:640x400,bpp=32,memsize=3072000 msm_display.dsi_dsc0=1 msm_display.dsi_dsc1=1 cgroup_disable=pressure ramoops_memreserve=4M" \
    -o new-boot.img
```

---

## 10. 刷入设备

### 10.1 备份原厂 boot.img（非常重要！）

```bash
adb shell "su -c 'dd if=/dev/block/bootdevice/by-name/boot of=/sdcard/stock_boot.img'"
adb pull /sdcard/stock_boot.img stock_boot_backup.img
# 保存好这个文件，刷机失败时用来恢复
```

### 10.2 方式 A：使用 Fastboot 刷入 boot.img

```bash
# 重启到 bootloader
adb reboot bootloader

# 确认 fastboot 连接
fastboot devices

# 检查分区方案
fastboot getvar current-slot  # 如果是 AB 分区，会有 _a / _b

# 如果支持 AB 分区（PixelExperience / crDroid 等新 ROM）
fastboot flash boot_a new-boot.img
fastboot flash boot_b new-boot.img

# 如果是 A-only 分区（MIUI / HyperOS）
fastboot flash boot new-boot.img

# 重启
fastboot reboot
```

### 10.3 方式 B：使用 Recovery 刷入 AnyKernel3 ZIP

```bash
# 将 ZIP 推送到设备
adb push RedmiK40S-Kernel-AnyKernel3.zip /sdcard/

# 重启到 Recovery
adb reboot recovery

# 在 Recovery 中：
# 1. 点击 "Install" / "安装"
# 2. 选择 RedmiK40S-Kernel-AnyKernel3.zip
# 3. 滑动刷入
# 4. 重启系统
```

### 10.4 如果设备无法启动

```bash
# 重启到 bootloader
# 关机后：音量- + 电源键

# 刷回原厂 boot.img
fastboot flash boot stock_boot_backup.img
# 或
fastboot flash boot_a stock_boot_backup.img
fastboot flash boot_b stock_boot_backup.img

fastboot reboot
```

---

## 11. 验证

### 11.1 验证内核版本

```bash
# 正常启动后
adb shell "uname -a"
# 应显示 4.19.325-perf+ 或类似

adb shell "cat /proc/version"
```

### 11.2 验证 KernelSU

```bash
# 方法 1：检查 ksu 模块是否加载
adb shell "lsmod | grep ksu"
adb shell "cat /proc/kallsyms | grep ksu"

# 方法 2：安装 KernelSU Manager 应用
# 下载地址：https://github.com/tiann/KernelSU/releases
# 打开应用，应显示 "KernelSU 已安装"
```

### 11.3 验证 DroidSpaces

```bash
# 方法 1：使用 DroidSpaces 应用检查
# 下载 DroidSpaces APK
# 打开 → 设置 → 需求
# 应用会自动检查内核兼容性

# 方法 2：手动检查内核配置
adb shell "zcat /proc/config.gz" | grep -E \
    "(CONFIG_NAMESPACES|CONFIG_PID_NS|CONFIG_CGROUP_DEVICE|CONFIG_OVERLAY_FS|CONFIG_SECCOMP|CONFIG_NET_NS)"

# 方法 3：尝试创建容器
# 安装 DroidSpaces 后，创建一个测试容器
# 如果容器能正常启动并分配 IP，说明一切正常
```

### 11.4 验证系统稳定性

```bash
# 检查内核日志是否有异常
adb shell "dmesg | grep -iE 'error|panic|oops|bug'"

# 测试基础功能
# - WiFi 连接
# - 蓝牙
# - 触屏
# - 电话/移动数据
# - 指纹
# - 充电/电池显示（检查1%电量bug是否修复）
```

---

## 12. GitHub Actions 自动化云编译

### 12.1 完整云编译工作流

在 GitHub 上创建一个仓库，将内核源码推送上去，然后添加以下工作流文件：

```yaml
# .github/workflows/build-kernel-munch.yml
name: Build Kernel for Redmi K40S (munch)

on:
  workflow_dispatch:
    inputs:
      ENABLE_DROIDSPACES:
        description: '集成 DroidSpaces'
        type: boolean
        default: true
      ENABLE_KSU:
        description: '集成 KernelSU'
        type: boolean
        default: true

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout kernel source
        uses: actions/checkout@v4
        with:
          repository: yspbwx2010/kernel_xiaomi_sm8250_mod
          ref: android14-lineage22-mod

      - name: Install dependencies
        run: |
          sudo apt update
          sudo apt install -y git wget curl \
            build-essential flex bison cpio bc \
            libssl-dev libncurses-dev python3 \
            lz4 zstd ccache

      - name: Setup toolchain
        run: |
          mkdir -p toolchain && cd toolchain
          # Clang 14
          wget -q https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/master-kernel-build-2022/clang-r450784d.tar.gz
          mkdir clang && tar -xf clang-r450784d.tar.gz -C clang/
          # GCC 4.9
          git clone -q https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9 --depth=1
          cd ..

      - name: Configure kernel
        run: |
          export ARCH=arm64
          export PATH=$(pwd)/toolchain/clang/bin:$PATH
          export CROSS_COMPILE=$(pwd)/toolchain/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-
          make O=out ARCH=arm64 CC=clang \
            CLANG_TRIPLE=aarch64-linux-gnu- \
            CROSS_COMPILE=$CROSS_COMPILE \
            munch_defconfig

      - name: Integrate KernelSU
        if: ${{ github.event.inputs.ENABLE_KSU == 'true' }}
        run: |
          git clone https://github.com/tiann/KernelSU --depth=1
          cd KernelSU
          curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main
          cd ..
          scripts/config --file out/.config -e CONFIG_KSU
          make O=out ARCH=arm64 olddefconfig

      - name: Integrate DroidSpaces
        if: ${{ github.event.inputs.ENABLE_DROIDSPACES == 'true' }}
        run: |
          # 添加 DroidSpaces 配置
          cat >> out/.config << 'DSEOF'
          CONFIG_SYSVIPC=y
          CONFIG_POSIX_MQUEUE=y
          CONFIG_CGROUP_PIDS=y
          CONFIG_CGROUP_NET_PRIO=y
          CONFIG_TMPFS_POSIX_ACL=y
          CONFIG_TMPFS_XATTR=y
          CONFIG_BRIDGE_NETFILTER=y
          CONFIG_IP_ADVANCED_ROUTER=y
          CONFIG_IP_MULTIPLE_TABLES=y
          CONFIG_NETFILTER_XT_TARGET_TCPMSS=y
          CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
          CONFIG_NF_CONNTRACK_NETLINK=y
          CONFIG_NF_NAT_REDIRECT=y
          CONFIG_NF_NAT_IPV4=y
          CONFIG_NF_CONNTRACK_IPV4=y
          CONFIG_IP_NF_NAT=y
DSEOF
          scripts/config --file out/.config -d CONFIG_ANDROID_PARANOID_NETWORK
          make O=out ARCH=arm64 olddefconfig

          # 应用 DroidSpaces 补丁
          wget -q https://raw.githubusercontent.com/ravindu644/Droidspaces-OSS/master/Documentation/resources/kernel-patches/non-GKI/01.fix_kernel_panic_in_xt_qtaguid.patch
          wget -q "https://raw.githubusercontent.com/ravindu644/Droidspaces-OSS/master/Documentation/resources/kernel-patches/non-GKI/02.fix_restore%20cgroup%20file%20prefix%20handling%20.patch"
          patch -p1 < 01.fix_kernel_panic_in_xt_qtaguid.patch
          patch -p1 < 02.fix_restore\ cgroup\ file\ prefix\ handling\ .patch

      - name: Build kernel
        run: |
          export ARCH=arm64
          export PATH=$(pwd)/toolchain/clang/bin:$PATH
          export CROSS_COMPILE=$(pwd)/toolchain/aarch64-linux-android-4.9/bin/aarch64-linux-androidkernel-
          make O=out ARCH=arm64 \
            CC=clang CLANG_TRIPLE=aarch64-linux-gnu- \
            CROSS_COMPILE=$CROSS_COMPILE \
            -j$(nproc --all) \
            Image.gz-dtb 2>&1 | tee build.log

      - name: Package with AnyKernel3
        run: |
          git clone https://github.com/osm0sis/AnyKernel3
          cp out/arch/arm64/boot/Image.gz-dtb AnyKernel3/
          cat > AnyKernel3/anykernel.sh << 'AKEOF'
device.name1=munch
device.name2=munchin
block=/dev/block/bootdevice/by-name/boot
is_slot_device=auto
ramdisk_compression=gzip
AKEOF
          cd AnyKernel3
          zip -r9 ../RedmiK40S-Kernel-DroidSpaces.zip * -x .git README.md *placeholder
          cd ..

      - name: Upload kernel
        uses: actions/upload-artifact@v4
        with:
          name: RedmiK40S-Kernel
          path: |
            RedmiK40S-Kernel-DroidSpaces.zip
            build.log
```

### 12.2 使用现有 Action 模板快速开始

如果你不想手动维护工作流，也可以直接使用社区维护的 Action：

```yaml
name: Build munch Kernel
on: [workflow_dispatch]

jobs:
  build:
    uses: dabao1955/kernel_build_action/.github/workflows/build.yml@master
    with:
      KERNEL_SOURCE: 'https://github.com/yspbwx2010/kernel_xiaomi_sm8250_mod'
      KERNEL_BRANCH: 'android14-lineage22-mod'
      DEFCONFIG_NAME: 'munch_defconfig'
      CLANG_VERSION: 'clang-r450784d'
      ENABLE_KSU: true
      # DroidSpaces 部分可能需要在 fork 中自行添加
```

---

## 13. 常见问题与排错

### 13.1 编译错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `fatal error: linux/ksu.h: No such file or directory` | KernelSU 未正确集成 | 检查 `drivers/ksu/` 是否存在，重跑 setup.sh |
| `undefined reference to ksu_handle_execveat_common` | KernelSU hook 未打上 | 重新运行 KernelSU 集成脚本 |
| `patch: **** Can't find file to patch` | 补丁路径不对 | 确保在内核源码根目录运行 `patch -p1` |
| `scripts/config: line 42: bc: command not found` | 缺少 bc | `sudo apt install bc` |
| `ld.lld: error: undefined symbol: __mod_member_use` | LLD 链接问题 | 添加 `LD=ld.bfd` 或使用 `LD=arm-linux-gnueabi-ld` |
| `No rule to make target 'debian/canonical-certs.pem'` | 签名证书缺失 | 在 .config 中设置 `CONFIG_SYSTEM_TRUSTED_KEYS=""` |
| `1% battery bug` | 电量计驱动问题 | 使用 yspbwx2010 的 mod 内核（已修复） |
| `DroidSpaces 容器无法联网` | `CONFIG_ANDROID_PARANOID_NETWORK` 未禁用 | 检查并禁用此选项 |

### 13.2 启动问题

| 现象 | 原因 | 解决 |
|------|------|------|
| **卡在 Redmi logo** | 内核崩溃或 DTB 问题 | 检查 `Image.gz-dtb` 是否成功生成；刷回原厂 boot.img |
| **无限重启（bootloop）** | 配置缺失关键驱动 | 用 fastboot 刷回原厂 boot.img |
| **WiFi 打不开** | WLAN 驱动问题 | 检查 `CONFIG_QCA_CLD_WLAN=y` 和 `CNSS_QCA6390` |
| **触屏不工作** | 触控驱动问题 | 检查 `CONFIG_TOUCHSCREEN_FOCALTECH_3658U=y` |
| **KernelSU Manager 显示未安装** | KSU 未正确编译 | 检查 `grep CONFIG_KSU out/.config` |
| **DroidSpaces 需求检查失败** | 缺少命名空间或 cgroup 配置 | 重新检查第 7.5 节的配置验证脚本 |

### 13.3 获取调试信息

```bash
# 内核日志
adb shell dmesg > dmesg.log
adb shell dmesg | grep -iE "ksu|droidspaces|container"

# last_kmsg（如果设备重启后无法完全启动）
adb shell cat /proc/last_kmsg > last_kmsg.log

# KernelSU 状态
adb shell lsmod | grep ksu
adb shell cat /proc/kallsyms | grep ksu

# 检查当前内核配置
adb shell "zcat /proc/config.gz" > running_config
grep CONFIG_KSU running_config
grep CONFIG_NAMESPACES running_config
```

### 13.4 恢复引导

```bash
# 如果刷入后无法启动：
# 1. 进入 fastboot（音量- + 电源）
fastboot flash boot stock_boot_backup.img
fastboot reboot

# 2. 如果连 fastboot 都进不去：
#    - EDL 模式（需要小米授权账号）
#    - 使用 Miflash 线刷完整 ROM
```

---

## 14. 参考资料

### 14.1 本教程使用的主要项目

| 项目 | 说明 | 链接 |
|------|------|------|
| **kernel_xiaomi_sm8250_mod** | 本教程用的内核源码 | [GitHub](https://github.com/yspbwx2010/kernel_xiaomi_sm8250_mod) |
| **DroidSpaces-OSS** | 容器运行时 | [GitHub](https://github.com/ravindu644/Droidspaces-OSS) |
| **KernelSU** | 内核级 Root 方案 | [GitHub](https://github.com/tiann/KernelSU) |
| **AnyKernel3** | 通用卡刷包模板 | [GitHub](https://github.com/osm0sis/AnyKernel3) |

### 14.2 相关内核项目

| 项目 | 说明 | 链接 |
|------|------|------|
| **kernel_xiaomi_sm8250** (UtsavBalar1231) | 上游原始内核 | [GitHub](https://github.com/UtsavBalar1231/kernel_xiaomi_sm8250) |
| **mi10s** (Dowsion-dev) | 备选内核 | [GitHub](https://github.com/Dowsion-dev/mi10s) |
| **android_kernel_xiaomi_sm8250-munch** (crDroid) | crDroid 维护的内核 | [GitHub](https://github.com/crdroidandroid/android_kernel_xiaomi_sm8250-munch) |
| **E404R Kernel** | munch 第三方内核 | [GitHub](https://github.com/kvsnr113/e404_kernel_releases) |

### 14.3 工具与文档

| 工具 | 用途 | 链接 |
|------|------|------|
| Google Clang 预编译 | 内核编译器 | [AOSP Clang](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/) |
| GCC 4.9 交叉编译器 | 老内核兼容 | [AOSP GCC 4.9](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/) |
| mkbootimg | boot.img 打包 | [AOSP mkbootimg](https://android.googlesource.com/platform/system/tools/mkbootimg) |
| kernel_build_action | GitHub Actions 云编译模板 | [dabao1955/kernel_build_action](https://github.com/dabao1955/kernel_build_action) |
| Android-Kernel-Tutorials | 通用内核编译教程 | [ravindu644/Android-Kernel-Tutorials](https://github.com/ravindu644/Android-Kernel-Tutorials) |

### 14.4 社区讨论

- **酷安**：搜索"Redmi K40S 内核"或"munch kernel"
- **XDA**：[POCO F4 / Redmi K40S 论坛](https://forum.xda-developers.com/f/poco-f4.12483/)
- **Telegram**：搜索 `@munch_updates` 或 `@XiaomiSM8250`
- **小米开源站**：[MiCode](https://github.com/MiCode/Xiaomi_Kernel_OpenSource)（搜索 munch）

---

## 附录 A：一键编译脚本

将以下内容保存为 `build-munch.sh`，然后 `chmod +x build-munch.sh && ./build-munch.sh`：

```bash
#!/bin/bash
# Redmi K40S (munch) 一键编译脚本
# 支持 KernelSU + DroidSpaces
set -e

# ===== 配置部分（根据你的环境修改）=====
KERNEL_DIR=$(pwd)
CLANG_DIR="$KERNEL_DIR/toolchain/clang"
GCC_DIR="$KERNEL_DIR/toolchain/aarch64-linux-android-4.9"
THREADS=$(nproc --all)
ENABLE_KSU=true
ENABLE_DROIDSPACES=true

# ===== 颜色输出 =====
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'
info() { echo -e "${GREEN}[INFO]${NC} $1"; }
warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
err() { echo -e "${RED}[ERROR]${NC} $1"; exit 1; }

# ===== 检查环境 =====
[ -f "Makefile" ] || err "请在内核源码根目录运行此脚本"
[ -d "$CLANG_DIR" ] || warn "Clang 未找到: $CLANG_DIR"
[ -d "$GCC_DIR" ] || warn "GCC 未找到: $GCC_DIR"
[ -f "arch/arm64/configs/munch_defconfig" ] || err "找不到 munch_defconfig"

# ===== 设置环境变量 =====
export ARCH=arm64
export PATH=$CLANG_DIR/bin:$PATH
export CROSS_COMPILE=$GCC_DIR/bin/aarch64-linux-androidkernel-
export CC=clang

info "=== 环境设置 ==="
info "内核目录: $KERNEL_DIR"
info "工具链: $(clang --version | head -1)"
info "线程数: $THREADS"

# ===== Step 1: 配置 =====
info "=== Step 1: 配置内核 ==="
make O=out ARCH=arm64 CC=clang \
    CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    munch_defconfig

# ===== Step 2: KernelSU =====
if [ "$ENABLE_KSU" = true ]; then
    info "=== Step 2: 集成 KernelSU ==="
    if [ ! -d "KernelSU" ]; then
        git clone https://github.com/tiann/KernelSU --depth=1
    fi
    if [ ! -d "drivers/ksu" ]; then
        cd KernelSU
        curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main
        cd ..
    fi
    scripts/config --file out/.config -e CONFIG_KSU
    make O=out ARCH=arm64 olddefconfig
    info "KernelSU 已集成"
fi

# ===== Step 3: DroidSpaces =====
if [ "$ENABLE_DROIDSPACES" = true ]; then
    info "=== Step 3: 集成 DroidSpaces ==="

    # 添加配置
    cat >> out/.config << 'EOF'
CONFIG_SYSVIPC=y
CONFIG_POSIX_MQUEUE=y
CONFIG_CGROUP_PIDS=y
CONFIG_CGROUP_NET_PRIO=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_TMPFS_XATTR=y
CONFIG_BRIDGE_NETFILTER=y
CONFIG_IP_ADVANCED_ROUTER=y
CONFIG_IP_MULTIPLE_TABLES=y
CONFIG_NETFILTER_XT_TARGET_TCPMSS=y
CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
CONFIG_NF_CONNTRACK_NETLINK=y
CONFIG_NF_NAT_REDIRECT=y
CONFIG_NF_NAT_IPV4=y
CONFIG_NF_CONNTRACK_IPV4=y
CONFIG_IP_NF_NAT=y
EOF
    scripts/config --file out/.config -d CONFIG_ANDROID_PARANOID_NETWORK
    make O=out ARCH=arm64 olddefconfig

    # 应用补丁
    mkdir -p patches
    [ -f "patches/01_fix_xt_qtaguid.patch" ] || \
        wget -q -O patches/01_fix_xt_qtaguid.patch \
        https://raw.githubusercontent.com/ravindu644/Droidspaces-OSS/master/Documentation/resources/kernel-patches/non-GKI/01.fix_kernel_panic_in_xt_qtaguid.patch
    [ -f "patches/02_fix_cgroup_prefix.patch" ] || \
        wget -q -O patches/02_fix_cgroup_prefix.patch \
        "https://raw.githubusercontent.com/ravindu644/Droidspaces-OSS/master/Documentation/resources/kernel-patches/non-GKI/02.fix_restore%20cgroup%20file%20prefix%20handling%20.patch"

    patch -p1 -N < patches/01_fix_xt_qtaguid.patch 2>/dev/null || true
    patch -p1 -N < patches/02_fix_cgroup_prefix.patch 2>/dev/null || true

    info "DroidSpaces 已集成"
fi

# ===== Step 4: 编译 =====
info "=== Step 4: 编译内核 (这可能需要 10-30 分钟) ==="
START_TIME=$(date +%s)
make O=out ARCH=arm64 \
    CC=clang CLANG_TRIPLE=aarch64-linux-gnu- \
    CROSS_COMPILE=$CROSS_COMPILE \
    -j$THREADS \
    Image.gz-dtb 2>&1 | tee build.log
END_TIME=$(date +%s)

if [ ${PIPESTATUS[0]} -eq 0 ]; then
    info "=== 编译成功！耗时 $((END_TIME - START_TIME)) 秒 ==="
    ls -lh out/arch/arm64/boot/Image.gz-dtb
else
    err "编译失败！请查看 build.log"
fi

# ===== Step 5: 打包 AnyKernel3 =====
info "=== Step 5: 打包 AnyKernel3 ==="
if [ ! -d "AnyKernel3" ]; then
    git clone https://github.com/osm0sis/AnyKernel3 --depth=1
fi
cp out/arch/arm64/boot/Image.gz-dtb AnyKernel3/
cat > AnyKernel3/anykernel.sh << 'EOF'
device.name1=munch
device.name2=munchin
block=/dev/block/bootdevice/by-name/boot
is_slot_device=auto
ramdisk_compression=gzip
EOF
cd AnyKernel3
zip -r9 "../RedmiK40S-Kernel-$(date +%Y%m%d-%H%M).zip" * -x .git README.md *placeholder
cd ..

info "=== 全部完成！==="
info "AnyKernel3 包：$(ls -t RedmiK40S-Kernel-*.zip | head -1)"
info "刷入方式：adb reboot recovery → 安装 ZIP → 重启"
```

---

> **免责声明**：编译自定义内核可能导致设备变砖、数据丢失或保修失效。请始终做好备份。本教程仅提供技术指导，不承担任何因操作不当造成的损失。
