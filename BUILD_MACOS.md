# FCEUX macOS 编译指南

本文档记录了在 macOS（Apple Silicon / Intel）上从源码编译 FCEUX NES 模拟器的完整步骤。
编译环境：macOS 26.5 (Sequoia)，Apple Silicon (arm64)，Homebrew 5.x。

---

## 目录

1. [前置要求](#1-前置要求)
2. [安装依赖](#2-安装依赖)
3. [CMake 配置](#3-cmake-配置)
4. [编译](#4-编译)
5. [打包 .app bundle](#5-打包-app-bundle)
6. [打包 DMG 安装包](#6-打包-dmg-安装包)
7. [常见问题排查](#7-常见问题排查)
8. [依赖版本参考](#8-依赖版本参考)

---

## 1. 前置要求

### 必需工具

| 工具 | 说明 |
|------|------|
| Xcode Command Line Tools | 提供 Clang 编译器和系统头文件 |
| Homebrew | macOS 包管理器 |

### 安装 Xcode Command Line Tools

```bash
xcode-select --install
```

执行后会弹出图形化安装窗口，按提示完成安装。验证：

```bash
xcode-select -p
# 应输出类似：/Library/Developer/CommandLineTools
```

### 安装 Homebrew

如果尚未安装 Homebrew：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Apple Silicon 安装完成后，需要将 Homebrew 加入 PATH（写入 `~/.zshrc`）：

```bash
echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

---

## 2. 安装依赖

### 必需依赖

```bash
brew install cmake qt6 sdl2 minizip libarchive pkgconf
```

说明：

| 包名 | 用途 |
|------|------|
| `cmake` | 构建系统生成器 |
| `qt6` | GUI 前端框架（Qt6，包含 Widgets、OpenGL、Network、Qml、UiTools 等模块） |
| `sdl2` | 音频/输入/窗口后端 |
| `minizip` | ZIP 文件支持（必需） |
| `libarchive` | 多格式归档支持（可选，但推荐，开启后支持更多 ROM 格式） |
| `pkgconf` | pkg-config 工具，用于 CMake 查找非标准路径的库 |

> **注意**：`libarchive` 是 keg-only 包（macOS 自带同名库），Homebrew 不会自动建立符号链接。
> 需要手动设置 `PKG_CONFIG_PATH`（见下方 CMake 配置步骤）。

### 可选依赖（视需要安装）

```bash
# 视频录制支持（ffmpeg + 编码器）
brew install ffmpeg x264 x265

# Lua 5.4 脚本支持（项目内置了 Lua 5.1，无需强制安装）
# brew install lua@5.4
```

---

## 3. CMake 配置

```bash
# 进入项目根目录
cd ~/code/github/fceux   # 替换为你实际的路径

# 创建并进入 build 目录
mkdir -p build && cd build

# 设置 PKG_CONFIG_PATH（让 CMake 找到 keg-only 的 libarchive）
export PKG_CONFIG_PATH="/opt/homebrew/opt/libarchive/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:/opt/homebrew/share/pkgconfig:$PKG_CONFIG_PATH"

# 设置 Qt6 路径
export QT6_DIR="/opt/homebrew/opt/qt"

# 运行 CMake 配置（Release 模式，使用 Qt6）
cmake .. \
  -DCMAKE_PREFIX_PATH="$QT6_DIR" \
  -DCMAKE_BUILD_TYPE=Release \
  -DQT6=1
```

### CMake 配置输出说明

配置成功后，你应该看到类似以下输出：

```
-- GUI Frontend: Qt6
-- Qt6 Help Module Found
-- Qt6 Network Module Found
-- Qt6 Qml Module Found
-- Qt6 UiTools Module Found
-- Found ZLIB: ...
-- Using System minizip 1.3.2
-- Using System Libarchive Library 3.8.7
-- Found sdl2, version 2.32.10
-- Using Internal Lua          ← 使用项目内置的 Lua 5.1（正常）
-- Configuring done
-- Generating done
-- Build files have been written to: .../build
```

### CMake 参数说明

| 参数 | 说明 |
|------|------|
| `-DCMAKE_PREFIX_PATH` | Qt6 安装路径，让 CMake 的 find_package 能找到 Qt |
| `-DCMAKE_BUILD_TYPE=Release` | 编译优化版本；调试用 `Debug` |
| `-DQT6=1` | 强制使用 Qt6（不设置则自动检测，先找 Qt6 再找 Qt5） |
| `-DQT6=0` 或 `-DQT=5` | 强制使用 Qt5（需要单独安装 `qt@5`） |

---

## 4. 编译

```bash
# 确保在 build 目录内
# 使用所有 CPU 核心并行编译（大幅加速）
make -j$(sysctl -n hw.logicalcpu)
```

编译成功后，末尾应输出：

```
[100%] Linking CXX executable fceux.app/Contents/MacOS/fceux
Copying OS X content src/fceux.app/Contents/Resources/fceux.icns
[100%] Built target fceux
```

生成的可执行文件位于：

```
build/src/fceux.app/Contents/MacOS/fceux
```

---

## 5. 打包 .app bundle

编译完成后，`fceux.app` 还依赖 Homebrew 中的动态库。使用 `macdeployqt` 将所有依赖内嵌进 `.app`，使其可以在没有安装 Homebrew 的机器上运行。

```bash
export PATH="/opt/homebrew/bin:/opt/homebrew/opt/qt/bin:$PATH"

# 用 macdeployqt 内嵌 Qt 框架和其他依赖
macdeployqt build/src/fceux.app -verbose=1
```

### 修复 macdeployqt 可能遗漏的库

`macdeployqt` 有时无法自动解析某些 rpath 库（如 brotli、webp）。
需要手动补充：

```bash
FRAMEWORKS_DIR="build/src/fceux.app/Contents/Frameworks"

# 补充 macdeployqt 可能遗漏的库
for lib in \
  /opt/homebrew/opt/brotli/lib/libbrotlienc.1.dylib \
  /opt/homebrew/opt/webp/lib/libwebp.7.dylib \
  /opt/homebrew/opt/webp/lib/libsharpyuv.0.dylib; do
  [ -f "$lib" ] && cp "$lib" "$FRAMEWORKS_DIR/"
done
```

### ad-hoc 签名（本地开发用）

```bash
# 对 .app 内所有 dylib 重新签名
for dylib in "build/src/fceux.app/Contents/Frameworks/"*.dylib; do
  codesign --force --sign - "$dylib"
done

# 对整个 .app 做 deep 签名
codesign --force --deep --sign - build/src/fceux.app

# 验证
codesign --verify --deep build/src/fceux.app && echo "签名验证成功"
```

> **说明**：`--sign -` 表示 ad-hoc 签名（无开发者证书），适用于本地运行。
> 如果要分发给其他 Mac 用户，需要使用 Apple Developer 账号的代码签名证书。

### 验证无外部依赖

```bash
# 这条命令应该没有输出（表示没有指向 /opt/homebrew 的外部依赖）
otool -L build/src/fceux.app/Contents/MacOS/fceux \
  | grep -v "@executable_path" \
  | grep -v "/usr/lib" \
  | grep -v "/System/"
```

---

## 6. 打包 DMG 安装包

```bash
# 将 .app 复制到输出目录
mkdir -p output/macos
cp -R build/src/fceux.app output/macos/

# 创建 DMG
hdiutil create \
  -volname "FCEUX" \
  -srcfolder output/macos/fceux.app \
  -ov -format UDZO \
  output/FCEUX_macOS.dmg

echo "DMG 生成于: output/FCEUX_macOS.dmg"
```

---

## 7. 常见问题排查

### 问题：`Could NOT find PkgConfig`

**原因**：`pkg-config` / `pkgconf` 未安装。

**解决**：
```bash
brew install pkgconf
```

---

### 问题：`No available formula with the name "lua@5.1"`

**原因**：Homebrew 已移除 `lua@5.1`。

**解决**：不需要安装，项目内置了 Lua 5.1 源码，CMake 配置时会自动使用 Internal Lua。
CMake 输出 `Using Internal Lua` 是正常现象。

---

### 问题：`CMake Error: Could NOT find minizip`

**原因**：`minizip` 未安装，或 `PKG_CONFIG_PATH` 未正确设置。

**解决**：
```bash
brew install minizip
export PKG_CONFIG_PATH="/opt/homebrew/lib/pkgconfig:$PKG_CONFIG_PATH"
```

---

### 问题：`libarchive` 找不到

**原因**：`libarchive` 是 keg-only，不在默认 pkg-config 路径中。

**解决**：执行 CMake 前先设置：
```bash
export PKG_CONFIG_PATH="/opt/homebrew/opt/libarchive/lib/pkgconfig:$PKG_CONFIG_PATH"
```

---

### 问题：`macdeployqt` 报 rpath 错误

**原因**：`macdeployqt` 无法通过 rpath 找到某些间接依赖（如 brotli、webp）。

**解决**：手动将缺失的库复制到 `Frameworks` 目录（见第 5 步"修复遗漏的库"）。
这类错误不影响 `.app` 本身运行，主要影响 `macdeployqt` 的 `codesign` 验证步骤。

---

### 问题：双击 .app 提示"无法打开，因为来自身份不明的开发者"

**解决**：
- 方法一（推荐临时使用）：右键点击 `.app` → 选择"打开" → 点击"打开"按钮。
- 方法二：在终端执行：
  ```bash
  xattr -cr /path/to/fceux.app
  ```

---

### 问题：Intel Mac 编译

在 Intel Mac（x86_64）上步骤完全相同。Homebrew 路径为 `/usr/local` 而非 `/opt/homebrew`，替换相应路径即可：

```bash
export PKG_CONFIG_PATH="/usr/local/opt/libarchive/lib/pkgconfig:/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH"
export QT6_DIR="/usr/local/opt/qt"
```

---

## 8. 依赖版本参考

以下是编写本文档时使用的实际版本（2026年6月）：

| 工具/库 | 版本 |
|---------|------|
| macOS | 26.5 (Sequoia) |
| 架构 | arm64 (Apple Silicon) |
| Xcode CLT | AppleClang 21.0.0 |
| cmake | 4.3.3 |
| qt (Qt6) | 6.11.1 |
| sdl2 | 2.32.10 |
| minizip | 1.3.2_1 |
| libarchive | 3.8.7 |
| pkgconf | 2.5.1 |

---

## 快速参考：完整命令序列

以下是从零开始到生成 DMG 的完整命令（假设 Xcode CLT 和 Homebrew 已安装）：

```bash
# 1. 安装依赖
brew install cmake qt6 sdl2 minizip libarchive pkgconf

# 2. 进入项目，创建 build 目录
cd ~/code/github/fceux
mkdir -p build && cd build

# 3. CMake 配置
export PKG_CONFIG_PATH="/opt/homebrew/opt/libarchive/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:$PKG_CONFIG_PATH"
cmake .. \
  -DCMAKE_PREFIX_PATH="/opt/homebrew/opt/qt" \
  -DCMAKE_BUILD_TYPE=Release \
  -DQT6=1

# 4. 编译
make -j$(sysctl -n hw.logicalcpu)

# 5. 打包依赖
export PATH="/opt/homebrew/opt/qt/bin:$PATH"
macdeployqt src/fceux.app -verbose=1

# 5a. 补充遗漏的库（如有需要）
FWDIR="src/fceux.app/Contents/Frameworks"
for lib in \
  /opt/homebrew/opt/brotli/lib/libbrotlienc.1.dylib \
  /opt/homebrew/opt/webp/lib/libwebp.7.dylib \
  /opt/homebrew/opt/webp/lib/libsharpyuv.0.dylib; do
  [ -f "$lib" ] && cp "$lib" "$FWDIR/"
done

# 5b. 重新签名
for dylib in "$FWDIR"/*.dylib; do codesign --force --sign - "$dylib"; done
codesign --force --deep --sign - src/fceux.app

# 6. 打包 DMG
cd ..
mkdir -p output/macos
cp -R build/src/fceux.app output/macos/
hdiutil create \
  -volname "FCEUX" \
  -srcfolder output/macos/fceux.app \
  -ov -format UDZO \
  output/FCEUX_macOS.dmg

echo "完成！DMG 位于: output/FCEUX_macOS.dmg"
```
