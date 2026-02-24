# Android设置

ZeroClaw为Android设备提供预构建的二进制文件。

## 支持的架构

| 目标 | Android版本 | 设备 |
|------|-------------|------|
| `armv7-linux-androideabi` | Android 4.1+ (API 16+) | 较旧的32位手机（Galaxy S3等） |
| `aarch64-linux-android` | Android 5.0+ (API 21+) | 现代64位手机 |

## 通过Termux安装

在Android上运行ZeroClaw的最简单方法是通过[Termux](https://termux.dev/)。

### 1. 安装Termux

从[F-Droid](https://f-droid.org/packages/com.termux/)下载（推荐）或GitHub发布版本。

> ⚠️ **注意：** Play商店版本已过时且不受支持。

### 2. 下载ZeroClaw

```bash
# 检查您的架构
uname -m
# aarch64 = 64位, armv7l/armv8l = 32位

# 下载适当的二进制文件
# 对于64位（aarch64）：
curl -LO https://github.com/zeroclaw-labs/zeroclaw/releases/latest/download/zeroclaw-aarch64-linux-android.tar.gz
tar xzf zeroclaw-aarch64-linux-android.tar.gz

# 对于32位（armv7）：
curl -LO https://github.com/zeroclaw-labs/zeroclaw/releases/latest/download/zeroclaw-armv7-linux-androideabi.tar.gz
tar xzf zeroclaw-armv7-linux-androideabi.tar.gz
```

### 3. 安装和运行

```bash
chmod +x zeroclaw
mv zeroclaw $PREFIX/bin/

# 验证安装
zeroclaw --version

# 运行设置
zeroclaw onboard
```

## 通过ADB直接安装

对于想要在Termux之外运行ZeroClaw的高级用户：

```bash
# 从您的计算机使用ADB
adb push zeroclaw /data/local/tmp/
adb shell chmod +x /data/local/tmp/zeroclaw
adb shell /data/local/tmp/zeroclaw --version
```

> ⚠️ 在Termux之外运行需要root设备或特定权限才能获得完整功能。

## Android上的限制

- **无systemd：** 使用Termux的`termux-services`进行守护进程模式
- **存储访问：** 需要Termux存储权限（`termux-setup-storage`）
- **网络：** 某些功能可能需要Android VPN权限进行本地绑定

## 从源代码构建

要自己为Android构建：

```bash
# 安装Android NDK
# 添加目标
rustup target add armv7-linux-androideabi aarch64-linux-android

# 设置NDK路径
export ANDROID_NDK_HOME=/path/to/ndk
export PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH

# 构建
cargo build --release --target armv7-linux-androideabi
cargo build --release --target aarch64-linux-android
```

## 故障排除

### "Permission denied"
```bash
chmod +x zeroclaw
```

### "not found"或链接器错误
确保您为设备下载了正确的架构。

### 旧版Android（4.x）
使用带有API级别16+的`armv7-linux-androideabi`构建。