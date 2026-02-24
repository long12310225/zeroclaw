# ZeroClaw故障排除

本指南重点关注常见的设置/运行时故障和快速解决路径。

最后验证：**2026年2月20日**。

## 安装/引导

### 找不到`cargo`

症状：

- 引导退出并显示`cargo is not installed`

修复：

```bash
./bootstrap.sh --install-rust
```

或从<https://rustup.rs/>安装。

### 缺少系统构建依赖

症状：

- 由于编译器或`pkg-config`问题导致构建失败

修复：

```bash
./bootstrap.sh --install-system-deps
```

### 在低RAM/低磁盘主机上构建失败

症状：

- `cargo build --release`被终止（`signal: 9`、OOM杀手或`cannot allocate memory`）
- 添加交换分区后构建崩溃，因为磁盘空间耗尽

为什么会发生这种情况：

- 运行时内存（常见操作<5MB）与编译时内存不同。
- 完整源码构建可能需要**2 GB RAM + 交换分区**和**6+ GB可用磁盘**。
- 在小型磁盘上启用交换分区可以避免RAM OOM，但仍可能因磁盘耗尽而失败。

受限机器的首选路径：

```bash
./bootstrap.sh --prefer-prebuilt
```