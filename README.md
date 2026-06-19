# 使用 GitHub Actions 云编译 Redmi K40S 内核

> 本仓库使用 GitHub Actions 自动编译 Redmi K40S (munch) 内核，集成 KernelSU + DroidSpaces。

## 🚀 使用方法

### 1. 推送代码到 GitHub

在 GitHub 创建一个新仓库，然后推送：

```bash
# 初始化 git（如未初始化）
git init
git add .
git commit -m "Initial commit"

# 关联远程仓库
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git branch -M main
git push -u origin main
```

### 2. 触发编译

**方法一：GitHub 网页端**
1. 打开你的 GitHub 仓库
2. 点击 **Actions** 标签页
3. 左侧选择 **Build Kernel**
4. 点击 **Run workflow** 按钮
5. 配置参数：
   - **target_devices** → `munch`（Redmi K40S）
   - **compiled_system** → `MIUI`（HyperOS/MIUI）或 `AOSP`
   - **kernelsu_variant** → 推荐 `SukiSU-Ultra`（功能最全）
   - **kpm_enable** → `enable`
   - **droidspaces_enable** → `enable`（✅ 启用容器支持）
6. 点击 **Run workflow**
7. 等待约 30-60 分钟编译完成

**方法二：手动触发**
```bash
gh workflow run Build_Kernel.yml \
  -f target_devices=munch \
  -f compiled_system=MIUI \
  -f kernelsu_variant=SukiSU-Ultra \
  -f kpm_enable=enable \
  -f droidspaces_enable=enable
```

### 3. 下载编译产物

编译完成后：
1. 进入 Actions 页面，点击刚刚完成的 workflow run
2. 在 **Artifacts** 区域下载 `Kernel-munch.zip`
3. 解压后得到 `.zip` 的 AnyKernel3 卡刷包

### 4. 刷入设备

```bash
# 推送到手机
adb push Kernel-munch-xxx.zip /sdcard/

# 重启到 Recovery
adb reboot recovery

# 在 Recovery 中选 Install → 选择 ZIP → 滑动刷入 → 重启
```

---

## ⚙️ 可选参数说明

| 参数 | 选项 | 说明 |
|------|------|------|
| **target_devices** | `munch` / `alioth` / `all` | 目标设备，选 `munch` 是红米 K40S |
| **compiled_system** | `MIUI` / `AOSP` | 适用系统，MIUI 包含 HyperOS |
| **kernelsu_variant** | `SukiSU-Ultra` / `RKSU` / `KernelSU` / `None` | Root 方案 |
| **kpm_enable** | `enable` / `disable` | KPM 模块支持 |
| **droidspaces_enable** | `enable` / `disable` | ✅ DroidSpaces 容器支持 |
| **kernel_version** | 自定义文本 | 内核版本号后缀 |
| **build_type** | `Release` / `Dev` | 构建类型 |

---

## 🧩 集成说明

### KernelSU
- **SukiSU-Ultra**：功能最全，支持 SUSFS 隐藏 + KPM 模块
- **RKSU**：基于 KernelSU，支持 SUSFS
- **KernelSU**：原生 KernelSU

### DroidSpaces
启用后会：
- 开启命名空间（PID/Net/IPC/UTS/User NS）
- 开启 CGroup 全套支持
- 开启 Seccomp、OverlayFS、Netfilter NAT
- **禁用** `ANDROID_PARANOID_NETWORK`（否则容器无法联网）
- 应用两个内核补丁：
  - `xt_qtaguid` 内核崩溃修复
  - CGroup 文件前缀兼容性修复

### 内核源码
基于 [yspbwx2010/kernel_xiaomi_sm8250_mod](https://github.com/yspbwx2010/kernel_xiaomi_sm8250_mod) `android14-lineage22-mod` 分支，内核版本 **4.19.325**，已修复 1% 电量 bug。

---

## 📦 输出产物

- AnyKernel3 卡刷包（`.zip`）— 可在 Recovery 中直接刷入
- 刷入前请**备份原厂 boot.img**

---

## 🔧 本地编译（备选）

如果不想用 GitHub Actions，也可以在本地编译：

```bash
# Ubuntu 22.04+
sudo apt install git wget curl build-essential flex bison cpio bc libssl-dev libncurses-dev lz4 zstd ccache

# 克隆内核源码
git clone https://github.com/yspbwx2010/kernel_xiaomi_sm8250_mod.git -b android14-lineage22-mod --depth=1
cd kernel_xiaomi_sm8250_mod

# 下载工具链（参考 workflow 中的步骤）

# 编译
export ARCH=arm64
make O=out ARCH=arm64 CC=clang CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=... munch_defconfig
make O=out ARCH=arm64 CC=clang ... -j$(nproc) Image.gz-dtb
```

---

## ⚠️ 免责声明

刷机有风险，操作需谨慎！请确保：
1. 已备份原厂 boot.img
2. 了解如何在紧急情况下恢复设备（fastboot 刷回原厂 boot.img）
3. 本教程提供的内核仅供学习研究使用
