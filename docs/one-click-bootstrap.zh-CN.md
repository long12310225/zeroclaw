# 一键引导

本页定义了安装和初始化ZeroClaw的最快支持路径。

最后验证：**2026年2月20日**。

## 选项0：Homebrew（macOS/Linuxbrew）

```bash
brew install zeroclaw
```

## 选项A（推荐）：克隆 + 本地脚本

```bash
git clone https://github.com/zeroclaw-labs/zeroclaw.git
cd zeroclaw
./bootstrap.sh
```

默认情况下它的作用：

1. `cargo build --release --locked`
2. `cargo install --path . --force --locked`

### 资源预检和预构建流程

源码构建通常至少需要：

- **2 GB RAM + 交换分区**
- **6 GB可用磁盘**

当资源受限时，引导现在会首先尝试预构建二进制文件。

```bash
./bootstrap.sh --prefer-prebuilt
```

要仅要求二进制安装并在没有兼容发布资产时失败：

```bash
./bootstrap.sh --prebuilt-only
```

要绕过预构建流程并强制源码编译：

```bash
./bootstrap.sh --force-source-build
```

## 双模式引导

默认行为是**仅应用**（构建/安装ZeroClaw）并期望现有的Rust工具链。

对于全新机器，显式启用环境引导：

```bash
./bootstrap.sh --install-system-deps --install-rust
```

说明：

- `--install-system-deps`安装编译器/构建先决条件（可能需要`sudo`）。
- `--install-rust`在缺少时通过`rustup`安装Rust。
- `--prefer-prebuilt`首先尝试发布二进制下载，然后回退到源码构建。
- `--prebuilt-only`禁用源码回退。
- `--force-source-build`完全禁用预构建流程。

## 选项B：远程一行命令

```bash
curl -fsSL https://raw.githubusercontent.com/zeroclaw-labs/zeroclaw/main/scripts/bootstrap.sh | bash
```

对于高安全环境，首选选项A，这样您可以在执行前查看脚本。

遗留兼容性：

```bash
curl -fsSL https://raw.githubusercontent.com/zeroclaw-labs/zeroclaw/main/scripts/install.sh | bash
```

此遗留端点优先转发到`scripts/bootstrap.sh`，如果在该修订版中不可用则回退到遗留源码安装。

如果您在仓库检出外运行选项B，引导脚本会自动克隆临时工作区，构建，安装，然后清理它。

## 可选入门模式

### 容器化入门（Docker）

```bash
./bootstrap.sh --docker
```

这会构建本地ZeroClaw镜像并在容器内启动入门，同时
将配置/工作区持久化到`./.zeroclaw-docker`。

容器CLI默认为`docker`。如果Docker CLI不可用且`podman`存在，
引导会自动回退到`podman`。您也可以显式设置`ZEROCLAW_CONTAINER_CLI`
（例如：`ZEROCLAW_CONTAINER_CLI=podman ./bootstrap.sh --docker`）。

对于Podman，引导使用`--userns keep-id`和`:Z`卷标签，因此
工作区/配置挂载在容器内保持可写。

如果您添加`--skip-build`，引导会跳过本地镜像构建。它首先尝试本地
Docker标签（`ZEROCLAW_DOCKER_IMAGE`，默认：`zeroclaw-bootstrap:local`）；如果缺失，
它会拉取`ghcr.io/zeroclaw-labs/zeroclaw:latest`并在运行前本地标记。

### 快速入门（非交互式）

```bash
./bootstrap.sh --onboard --api-key "sk-..." --provider openrouter
```

或使用环境变量：

```bash
ZEROCLAW_API_KEY="sk-..." ZEROCLAW_PROVIDER="openrouter" ./bootstrap.sh --onboard
```

### 交互式入门

```bash
./bootstrap.sh --interactive-onboard
```

## 有用标志

- `--install-system-deps`
- `--install-rust`
- `--skip-build`（在`--docker`模式中：如果存在则使用本地镜像，否则拉取`ghcr.io/zeroclaw-labs/zeroclaw:latest`）
- `--skip-install`
- `--provider <id>`

查看所有选项：

```bash
./bootstrap.sh --help
```

## 相关文档

- [README.zh-CN.md](../README.zh-CN.md)
- [commands-reference.zh-CN.md](commands-reference.zh-CN.md)
- [providers-reference.zh-CN.md](providers-reference.zh-CN.md)
- [channels-reference.zh-CN.md](channels-reference.zh-CN.md)