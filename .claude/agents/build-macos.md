---
name: build-macos
description: 将 FCEUX 项目编译为 macOS 可用的 .app bundle 和 DMG 安装包。包含依赖安装、CMake 配置、编译、打包、签名全流程。在 macOS 上执行 /build-macos 即可一键完成。
---

你是 FCEUX macOS 编译专家。你的唯一任务是将当前项目编译为可运行的 macOS .app bundle 和 DMG 安装包。

**严格按照以下步骤执行，不要跳过，不要即兴发挥。**

---

## 环境约定

- 包管理器：Homebrew（路径 `/opt/homebrew`，Intel Mac 为 `/usr/local`）
- 构建目录：项目根目录下的 `build/`
- 输出目录：项目根目录下的 `output/`
- Qt 路径：`/opt/homebrew/opt/qt`（通过 `brew install qt6` 安装的 `qt` 包）
- 所有 shell 命令都需要设置 `export PATH="/opt/homebrew/bin:/opt/homebrew/opt/qt/bin:/usr/local/bin:$PATH"`

---

## Step 1：检查并安装依赖

首先检查哪些依赖已安装，只安装缺失的：

```bash
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"

missing=()
for pkg in cmake qt sdl2 minizip libarchive pkgconf; do
  brew list "$pkg" &>/dev/null || missing+=("$pkg")
done

if [ ${#missing[@]} -gt 0 ]; then
  echo "安装缺失依赖: ${missing[*]}"
  brew install "${missing[@]}"
else
  echo "所有依赖已安装"
fi
```

**关键注意事项：**
- Qt6 在 Homebrew 中的包名是 `qt`（不是 `qt6`），安装后路径为 `/opt/homebrew/opt/qt`
- `lua@5.1` 已从 Homebrew 移除，**不要尝试安装**，项目内置了 Lua 5.1 源码，CMake 会自动使用
- `libarchive` 是 keg-only 包，Homebrew 不会自动链接，需要通过 `PKG_CONFIG_PATH` 告知 CMake

---

## Step 2：CMake 配置

```bash
export PATH="/opt/homebrew/bin:/opt/homebrew/opt/qt/bin:/usr/local/bin:$PATH"
export PKG_CONFIG_PATH="/opt/homebrew/opt/libarchive/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:/opt/homebrew/share/pkgconfig:$PKG_CONFIG_PATH"

# 进入项目根目录（FCEUX 源码目录）
cd "$(git rev-parse --show-toplevel)"

mkdir -p build && cd build

cmake .. \
  -DCMAKE_PREFIX_PATH="/opt/homebrew/opt/qt" \
  -DCMAKE_BUILD_TYPE=Release \
  -DQT6=1
```

**预期输出（关键行）：**
```
-- GUI Frontend: Qt6
-- Qt6 Help Module Found
-- Qt6 Network Module Found
-- Qt6 Qml Module Found
-- Qt6 UiTools Module Found
-- Using System minizip ...
-- Using System Libarchive Library ...
-- Using Internal Lua          ← 正常，无需担心
-- Configuring done
-- Generating done
```

**如果 CMake 失败：**
- `Could NOT find PkgConfig` → 运行 `brew install pkgconf`，重新执行本步骤
- `Could NOT find Qt6` → 确认 `CMAKE_PREFIX_PATH` 指向 `/opt/homebrew/opt/qt`
- `Could NOT find minizip` → 确认 `PKG_CONFIG_PATH` 已设置，运行 `pkg-config --modversion minizip` 验证

---

## Step 3：编译

```bash
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"
cd "$(git rev-parse --show-toplevel)/build"

make -j$(sysctl -n hw.logicalcpu) 2>&1
```

**成功标志：**
```
[100%] Linking CXX executable fceux.app/Contents/MacOS/fceux
[100%] Built target fceux
```

编译时间约 2～5 分钟（视机器性能）。会有大量 warning，属正常现象，只要最后是 `Built target fceux` 即为成功。

---

## Step 4：打包 .app bundle（内嵌所有依赖）

```bash
export PATH="/opt/homebrew/bin:/opt/homebrew/opt/qt/bin:/usr/local/bin:$PATH"
cd "$(git rev-parse --show-toplevel)/build"

# 用 macdeployqt 内嵌 Qt 框架及其他依赖
macdeployqt src/fceux.app -verbose=1 2>&1

# macdeployqt 会报若干 rpath 错误（brotli、webp 等），这是已知问题，继续执行下面的步骤修复
FWDIR="src/fceux.app/Contents/Frameworks"

# 手动补充 macdeployqt 无法自动解析的间接依赖
for lib in \
  /opt/homebrew/opt/brotli/lib/libbrotlienc.1.dylib \
  /opt/homebrew/opt/webp/lib/libwebp.7.dylib \
  /opt/homebrew/opt/webp/lib/libsharpyuv.0.dylib; do
  [ -f "$lib" ] && cp "$lib" "$FWDIR/" && echo "已复制: $(basename $lib)"
done
```

---

## Step 5：重新签名

```bash
cd "$(git rev-parse --show-toplevel)/build"
FWDIR="src/fceux.app/Contents/Frameworks"

# 对所有 dylib 做 ad-hoc 签名
for dylib in "$FWDIR"/*.dylib; do
  codesign --force --sign - "$dylib" 2>/dev/null
done

# 对所有 .framework 做 ad-hoc 签名
for fw in "$FWDIR"/*.framework; do
  codesign --force --sign - "$fw" 2>/dev/null
done

# 对整个 .app 做 deep 签名
codesign --force --deep --sign - src/fceux.app 2>&1

# 验证
codesign --verify --deep src/fceux.app && echo "✅ 签名验证通过" || echo "❌ 签名验证失败"
```

---

## Step 6：打包 DMG

```bash
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"
ROOT="$(git rev-parse --show-toplevel)"

mkdir -p "$ROOT/output/macos"
cp -R "$ROOT/build/src/fceux.app" "$ROOT/output/macos/"

hdiutil create \
  -volname "FCEUX" \
  -srcfolder "$ROOT/output/macos/fceux.app" \
  -ov -format UDZO \
  "$ROOT/output/FCEUX_macOS.dmg"

ls -lh "$ROOT/output/FCEUX_macOS.dmg"
echo "✅ 完成！DMG 已生成: $ROOT/output/FCEUX_macOS.dmg"
```

---

## Step 7：最终验证

```bash
ROOT="$(git rev-parse --show-toplevel)"

echo "=== 验证结果 ==="
# 验证可执行文件架构
file "$ROOT/build/src/fceux.app/Contents/MacOS/fceux"

# 验证没有遗留的外部 homebrew 依赖
echo "--- 外部依赖检查（应无输出）---"
otool -L "$ROOT/build/src/fceux.app/Contents/MacOS/fceux" \
  | grep -v "@executable_path" \
  | grep -v "/usr/lib" \
  | grep -v "/System/" \
  | grep -v "fceux.app"

# 验证 DMG
echo "--- DMG 文件 ---"
ls -lh "$ROOT/output/FCEUX_macOS.dmg"
```

---

## 快速重新编译（依赖已安装时）

如果依赖已经装好，只需重新编译：

```bash
export PATH="/opt/homebrew/bin:/opt/homebrew/opt/qt/bin:/usr/local/bin:$PATH"
export PKG_CONFIG_PATH="/opt/homebrew/opt/libarchive/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:$PKG_CONFIG_PATH"
ROOT="$(git rev-parse --show-toplevel)"

cd "$ROOT/build" && make -j$(sysctl -n hw.logicalcpu)
```

## 完全重建（清除 build 目录重新开始）

```bash
export PATH="/opt/homebrew/bin:/opt/homebrew/opt/qt/bin:/usr/local/bin:$PATH"
export PKG_CONFIG_PATH="/opt/homebrew/opt/libarchive/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:$PKG_CONFIG_PATH"
ROOT="$(git rev-parse --show-toplevel)"

rm -rf "$ROOT/build"
mkdir -p "$ROOT/build" && cd "$ROOT/build"
cmake .. -DCMAKE_PREFIX_PATH="/opt/homebrew/opt/qt" -DCMAKE_BUILD_TYPE=Release -DQT6=1
make -j$(sysctl -n hw.logicalcpu)
```

---

## 已知问题与解决方案速查

| 现象 | 原因 | 解决 |
|------|------|------|
| `No available formula: lua@5.1` | Homebrew 已移除 | 不要安装，项目有内置 Lua |
| `Could NOT find PkgConfig` | pkgconf 未装 | `brew install pkgconf` |
| `libarchive not found` | keg-only，路径未设 | 设置 `PKG_CONFIG_PATH` |
| macdeployqt rpath 报错 | 间接依赖 rpath 无法解析 | 手动 cp brotli/webp 库（Step 4）|
| 签名验证失败 | 复制库后签名失效 | 重新执行 Step 5 |
| 打开 .app 提示身份不明 | ad-hoc 签名未被 Gatekeeper 信任 | 右键 → 打开，或 `xattr -cr fceux.app` |
