# 在 AVD 上使用 KernelSU（GitHub Actions 自动构建）

> 基于 5ec1cff 的方法：从 Android CI 获取 AVD 内核精确源码，重新编译并集成 KernelSU。
> 使用 GitHub Actions 完成编译，无需本地 Linux 环境。

## 原理

AVD 的内核版本号包含构建信息：

```
6.12.69-android16-6-g214d1615c480-ab15053784
│        │         │             │
│        │         │             └─ CI 构建号 (BUILD_NUMBER)
│        │         └─ 内核源码提交 hash (12位)
│        └─ 内核分支后缀
└─ 内核版本
```

通过 CI 构建号可以找到该次构建的完整 `BUILD_INFO`，其中包含所有仓库的精确 revision。基于相同源码 + KernelSU patch 重新编译，即可得到兼容 AVD 的 KernelSU 内核。

## 前置条件

- Android Studio 已安装，AVD 已创建
- 系统镜像选择 **Google APIs**（userdebug 构建，`adb root` 可用）
- GitHub 账号（用于 Actions 构建）

## 步骤

### 1. 获取 AVD 内核版本

```powershell
adb root
adb shell uname -r
# 示例输出: 6.12.69-android16-6-g214d1615c480-ab15053784
```

### 2. 提取 CI 构建号

从版本号中取 `-ab` 后面的数字：

```
6.12.69-android16-6-g214d1615c480-ab15053784
                                  ^^^^^^^^^^
                                  构建号: 15053784
```

### 3. 确认 CI 构建存在

浏览器打开（替换构建号）：

```
https://ci.android.com/builds/submitted/15053784/kernel_virt_x86_64/latest
```

如果页面 404，说明构建号对应的目标不是 `kernel_virt_x86_64`，尝试：
- `kernel_virt_x86_64` — AVD x86_64（最常见）
- `kernel_virt_arm64` — AVD ARM64

找到后，下载 `BUILD_INFO` 文件：

```
https://ci.android.com/builds/submitted/15053784/kernel_virt_x86_64/latest/view/BUILD_INFO
```

### 4. 确定内核分支

BUILD_INFO 中 `repo-init-branch` 字段即为内核分支名：

```json
"repo-init-branch": "aosp_kernel-common-android16-6.12"
```

记下这个值，后续 Actions 需要。

同时记录 `parsed_manifest` 中各仓库的 `revision`，最重要的是：

| 仓库 | path | 用途 |
|------|------|------|
| `kernel/common` | `common` | 内核主源码 |
| `kernel/build` | `build/kernel` | 构建脚本 |
| `kernel/common-modules/virtual-device` | `common-modules/virtual-device` | 虚拟设备模块 |
| `kernel/configs` | `kernel/configs` | 内核配置 |
| `platform/prebuilts/clang/host/linux-x86` | `prebuilts/clang/host/linux-x86` | 编译器 |

### 5. 创建 GitHub 仓库

1. 创建新仓库（如 `avd-kernelsu-builder`）
2. 不需要配置 Secrets，所有操作都是公开的 AOSP 仓库

### 6. 创建 Actions Workflow

在仓库中创建 `.github/workflows/build.yml`：

```yaml
name: Build KernelSU for AVD x86_64

on:
  workflow_dispatch:
    inputs:
      kernel_branch:
        description: '内核分支 (从 BUILD_INFO 的 repo-init-branch 获取)'
        required: true
        default: 'aosp_kernel-common-android16-6.12'
        type: string
      build_number:
        description: 'CI 构建号 (从 uname -r 的 -ab 后面获取)'
        required: true
        default: '15053784'
        type: string
      ksu_version:
        description: 'KernelSU 版本'
        required: true
        default: 'v3.2.5'
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 360

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y bc bison flex libelf-dev libssl-dev \
            liblz4-tool cpio lz4

      - name: Setup Bazelisk
        run: |
          wget -q https://github.com/bazelbuild/bazelisk/releases/download/v1.19.0/bazelisk-linux-amd64 \
            -O /usr/local/bin/bazel
          chmod +x /usr/local/bin/bazel
          bazel --version

      - name: Fetch BUILD_INFO
        run: |
          BID=${{ inputs.build_number }}
          TARGET="kernel_virt_x86_64"

          echo "Fetching BUILD_INFO for build $BID ($TARGET)..."

          wget -q "https://ci.android.com/builds/submitted/${BID}/${TARGET}/latest/view/BUILD_INFO" \
            -O BUILD_INFO.json

          # 提取各仓库的 revision
          python3 << 'PYEOF'
          import json

          with open("BUILD_INFO.json") as f:
              data = json.load(f)

          projects = data["parsed_manifest"]["projects"]
          revisions = {}
          for p in projects:
              revisions[p["path"]] = {
                  "name": p["name"],
                  "revision": p["revision"]
              }
              print(f"  {p['path']}: {p['revision'][:12]}")

          with open("revisions.json", "w") as f:
              json.dump(revisions, f, indent=2)

          print(f"\nFound {len(revisions)} repositories")
          PYEOF

      - name: Clone kernel source
        run: |
          python3 << 'PYEOF'
          import json, subprocess, os

          with open("revisions.json") as f:
              revisions = json.load(f)

          # 仓库列表（按依赖顺序）
          repos = [
              ("kernel/common", "common"),
              ("kernel/build", "build/kernel"),
              ("kernel/common-modules/virtual-device", "common-modules/virtual-device"),
              ("kernel/configs", "kernel/configs"),
              ("kernel/tests", "kernel/tests"),
              ("platform/prebuilts/clang/host/linux-x86", "prebuilts/clang/host/linux-x86"),
              ("platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8",
               "prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8"),
              ("platform/prebuilts/build-tools", "prebuilts/build-tools"),
              ("platform/prebuilts/clang-tools", "prebuilts/clang-tools"),
              ("platform/prebuilts/bazel/linux-x86_64", "prebuilts/bazel/linux-x86_64"),
              ("platform/prebuilts/jdk/jdk11", "prebuilts/jdk/jdk11"),
              ("toolchain/prebuilts/ndk/r23", "prebuilts/ndk-r23"),
              ("platform/external/bazel-skylib", "external/bazel-skylib"),
              ("platform/build/bazel_common_rules", "build/bazel_common_rules"),
              ("platform/external/stardoc", "external/stardoc"),
              ("platform/external/python/absl-py", "external/python/absl-py"),
              ("kernel/manifest", "kernel/manifest"),
              ("kernel/prebuilts/build-tools", "prebuilts/kernel-build-tools"),
              ("platform/system/tools/mkbootimg", "tools/mkbootimg"),
          ]

          for repo_name, local_path in repos:
              if local_path not in revisions:
                  print(f"  SKIP {local_path} (not in manifest)")
                  continue

              rev = revisions[local_path]["revision"]
              url = f"https://android.googlesource.com/{repo_name}"
              parent = os.path.dirname(local_path)
              if parent:
                  os.makedirs(parent, exist_ok=True)

              print(f"  Clone {repo_name} -> {local_path} ({rev[:12]})...")
              subprocess.run(["git", "clone", "--depth=1", url, local_path], check=True)
              subprocess.run(["git", "-C", local_path, "fetch", "origin", rev], check=True)
              subprocess.run(["git", "-C", local_path, "checkout", rev], check=True)
              print(f"    OK")

          print("\nAll repositories cloned.")
          PYEOF

      - name: Apply KernelSU patches
        run: |
          echo "Applying KernelSU ${{ inputs.ksu_version }}..."

          cd common
          curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" \
            | bash -s ${{ inputs.ksu_version }}

          echo "KernelSU applied. Checking drivers/kernelsu/..."
          ls -la drivers/kernelsu/

      - name: Enable KernelSU in defconfig
        run: |
          cd common

          GKI_CONFIG="arch/x86/configs/gki_defconfig"
          VIRT_CONFIG="../common-modules/virtual-device/android/virtual_device_x86_64.fragment"

          # 在 GKI defconfig 中启用 KSU
          if [ -f "$GKI_CONFIG" ] && ! grep -q "CONFIG_KSU" "$GKI_CONFIG"; then
            echo "" >> "$GKI_CONFIG"
            echo "# KernelSU" >> "$GKI_CONFIG"
            echo "CONFIG_KSU=y" >> "$GKI_CONFIG"
            echo "CONFIG_KSU_ALLOWLIST=y" >> "$GKI_CONFIG"
            echo "Added KSU to gki_defconfig"
          fi

          # 在 virtual device fragment 中启用
          if [ -f "$VIRT_CONFIG" ] && ! grep -q "CONFIG_KSU" "$VIRT_CONFIG"; then
            echo "" >> "$VIRT_CONFIG"
            echo "# KernelSU" >> "$VIRT_CONFIG"
            echo "CONFIG_KSU=y" >> "$VIRT_CONFIG"
            echo "Added KSU to virtual_device fragment"
          fi

      - name: Build kernel
        run: |
          echo "Building virtual device kernel (this takes 1-3 hours)..."

          # 创建符号链接
          ln -sf build/kernel/kleaf/bazel.sh tools/bazel
          ln -sf build/kernel/kleaf/bazel.WORKSPACE WORKSPACE
          mkdir -p build

          tools/bazel run \
            --verbose_failures \
            --jobs=$(nproc) \
            --make_jobs=$(nproc) \
            --config=fast \
            --config=stamp \
            --lto=none \
            //common-modules/virtual-device:virtual_device_x86_64_dist \
            -- --dist_dir=$PWD/dist --flat

          echo "Build complete!"
          ls -lh dist/

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: kernelsu-avd-x86_64
          path: |
            dist/boot.img
            dist/bzImage
            dist/initramfs.img
          retention-days: 30
```

### 7. 触发构建

1. Push 仓库到 GitHub
2. Actions → "Build KernelSU for AVD x86_64" → "Run workflow"
3. 填入参数：
   - `kernel_branch`: BUILD_INFO 中的 `repo-init-branch`
   - `build_number`: 从 `uname -r` 提取的数字
   - `ksu_version`: `v3.2.5`
4. 等待构建完成（约 1-3 小时）

### 8. 下载并使用

构建完成后，从 Actions → Artifacts 下载 `kernelsu-avd-x86_64`，解压得到 `boot.img`。

```powershell
# AVD 内核文件位于 SDK 系统镜像目录
$avdDir = "$env:LOCALAPPDATA\Android\Sdk\system-images\android-36\google_apis\x86_64"

# 备份原 boot.img
copy "$avdDir\boot.img" "$avdDir\boot.img.bak"

# 替换
copy .\dist\boot.img "$avdDir\boot.img"

# 启动 AVD（在 AVD Manager 中 Cold Boot）
```

### 9. 验证 KernelSU

```powershell
adb root
adb shell su -c "id"
# 预期: uid=0(root) ... context=u:r:su:s0

adb shell su -c "ksud --version"
```

### 10. 安装 TeeForge-CD 模块

```powershell
# 打包模块
./package.sh

# 推送到 AVD 安装
adb push out/teeforge_cd-*.zip /sdcard/

# 在 KernelSU Manager 中安装，或手动部署：
adb shell mkdir -p /data/adb/modules/teeforge_cd
adb push out/build/teeforge_cd/* /data/adb/modules/teeforge_cd/
adb shell chmod 755 /data/adb/modules/teeforge_cd/teeforge

# 测试
adb shell /data/adb/modules/teeforge_cd/teeforge --help
adb shell /data/adb/modules/teeforge_cd/teeforge --rootdetect
```

## AVD 内核文件位置

```
%LOCALAPPDATA%\Android\Sdk\system-images\android-<API>\google_apis\x86_64\
├── boot.img          ← 替换这个
├── system.img
├── vendor.img
└── ...
```

查找命令：

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Android\Sdk\system-images" -Recurse -Filter "boot.img" | Select FullName
```

## 常见问题

### 构建超时

GitHub Actions 免费账户限制 6 小时。内核编译通常 1-3 小时，超时可考虑付费 runner。

### boot.img 启动卡 logo

1. 确认 BUILD_INFO 构建号与 `uname -r` 中的 `-ab` 完全一致
2. 尝试只替换 bzImage 而非整个 boot.img
3. 某些 AVD 镜像可能有 AVB 校验，换一个镜像版本试试

### insmod 失败 (Exec format error)

架构不匹配。确认 `adb shell uname -m` 输出 `x86_64`，.ko 也必须是 x86_64 编译的。

### CI 构建页面 404

构建号可能对应其他目标。访问 `https://ci.android.com/builds/submitted/<构建号>` 查看所有可用目标。

## 参考

- [5ec1cff: 在 AVD 上使用 KernelSU - 第二回](https://5ec1cff.github.io/my-blog/2024/01/31/avd-ksu2/)
- [Android CI](https://ci.android.com)
- [KernelSU Releases](https://github.com/tiann/KernelSU/releases)
- [KernelSU GKI 安装文档](https://kernelsu.org/zh_CN/guide/installation.html)
