# AVD KernelSU Builder

基于 GitHub Actions 自动构建适用于 Android Virtual Device (AVD) 的 KernelSU 内核。

> 方法来源：[5ec1cff 的博客](https://5ec1cff.github.io/my-blog/2024/01/31/avd-ksu2/)

## 原理

AVD 内核版本号中 `-ab` 后的数字对应 Android CI 的构建号。通过该构建号可以从 [ci.android.com](https://ci.android.com) 获取 `BUILD_INFO`，其中包含所有仓库的精确 git revision。使用相同源码 + KernelSU 补丁重新编译，即可得到兼容 AVD 的 KernelSU 内核。

```
6.12.69-android16-6-g214d1615c480-ab15053784
│        │         │             │
│        │         │             └─ CI 构建号 (BUILD_NUMBER)
│        │         └─ 内核源码提交 hash
│        └─ 内核分支后缀
└─ 内核版本
```

## 使用方法

### 1. 获取 AVD 内核信息

```bash
adb root
adb shell uname -r
# 示例: 6.12.69-android16-6-g214d1615c480-ab15053784
```

从版本号中提取：
- **构建号**: `15053784`（`-ab` 后面的数字）
- **内核分支**: 从 [BUILD_INFO](https://ci.android.com/builds/submitted/15053784/kernel_virt_x86_64/latest/view/BUILD_INFO) 的 `repo-init-branch` 字段获取

### 2. 触发构建

1. Fork 本仓库
2. 进入 Actions → "Build KernelSU for AVD x86_64" → "Run workflow"
3. 填入参数：
   - `kernel_branch`: BUILD_INFO 中的 `repo-init-branch`（如 `aosp_kernel-common-android16-6.12`）
   - `build_number`: 从 `uname -r` 提取的数字（如 `15053784`）
   - `ksu_version`: KernelSU 版本（如 `v3.2.5`）
4. 等待构建完成（约 1-3 小时）

### 3. 替换 AVD 内核

构建完成后，从 Actions → Artifacts 下载 `kernelsu-avd-x86_64`。

```powershell
# AVD 内核文件位置
$avdDir = "$env:LOCALAPPDATA\Android\Sdk\system-images\android-36\google_apis\x86_64"

# 备份原 boot.img
Copy-Item "$avdDir\boot.img" "$avdDir\boot.img.bak"

# 替换为 KernelSU 内核
Copy-Item .\dist\boot.img "$avdDir\boot.img"
```

### 4. 验证

```bash
adb root
adb shell su -c "id"
# 预期: uid=0(root) ... context=u:r:su:s0

adb shell su -c "ksud --version"
```

## 前置条件

- Android Studio 已安装，AVD 已创建
- 系统镜像选择 **Google APIs**（userdebug 构建，`adb root` 可用）
- GitHub 账号（用于 Actions 构建）

## 常见问题

### 构建超时

GitHub Actions 免费账户限制 6 小时。内核编译通常 1-3 小时，超时可考虑付费 runner。

### boot.img 启动卡 logo

1. 确认 BUILD_INFO 构建号与 `uname -r` 中的 `-ab` 完全一致
2. 尝试只替换 bzImage 而非整个 boot.img
3. 某些 AVD 镜像可能有 AVB 校验，换一个镜像版本试试

### CI 构建页面 404

构建号可能对应其他目标。访问 `https://ci.android.com/builds/submitted/<构建号>` 查看所有可用目标。常见目标：
- `kernel_virt_x86_64` — AVD x86_64（最常见）
- `kernel_virt_arm64` — AVD ARM64

## 参考

- [5ec1cff: 在 AVD 上使用 KernelSU](https://5ec1cff.github.io/my-blog/2024/01/16/avd-ksu/)
- [5ec1cff: 在 AVD 上使用 KernelSU - 第二回](https://5ec1cff.github.io/my-blog/2024/01/31/avd-ksu2/)
- [Android CI](https://ci.android.com)
- [KernelSU](https://github.com/tiann/KernelSU)
- [KernelSU GKI 安装文档](https://kernelsu.org/zh_CN/guide/installation.html)
