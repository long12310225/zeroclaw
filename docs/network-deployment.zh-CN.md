# 网络部署 —— 树莓派和本地网络上的ZeroClaw

本文档涵盖了在树莓派或本地网络上的其他主机上部署ZeroClaw，包括Telegram和可选的webhook通道。

---

## 1. 概述

| 模式 | 需要入站端口？ | 使用场景 |
|------|----------------|----------|
| **Telegram轮询** | 否 | ZeroClaw轮询Telegram API；可在任何地方工作 |
| **Matrix同步（包括E2EE）** | 否 | ZeroClaw通过Matrix客户端API同步；不需要入站webhook |
| **Discord/Slack** | 否 | 相同 —— 仅出站 |
| **Nostr** | 否 | 通过WebSocket连接到中继；仅出站 |
| **网关webhook** | 是 | POST /webhook, /whatsapp, /linq, /nextcloud-talk需要公共URL |
| **网关配对** | 是 | 如果您通过网关联接客户端 |
| **Alpine/OpenRC服务** | 否 | Alpine Linux上的系统范围后台服务 |

**关键：** Telegram、Discord、Slack和Nostr使用**出站连接** —— ZeroClaw连接到外部服务器/中继。不需要端口转发或公共IP。

---

## 2. 树莓派上的ZeroClaw

### 2.1 先决条件

- 树莓派（3/4/5）配备树莓派操作系统
- USB外设（Arduino、Nucleo）如果使用串口传输
- 可选：`rppal`用于本地GPIO（`peripheral-rpi`功能）

### 2.2 安装

```bash
# 为RPi构建（或从主机交叉编译）
cargo build --release --features hardware

# 或通过您喜欢的方法安装
```

### 2.3 配置

编辑`~/.zeroclaw/config.toml`：

```toml
[peripherals]
enabled = true

[[peripherals.boards]]
board = "rpi-gpio"
transport = "native"

# 或通过USB的Arduino
[[peripherals.boards]]
board = "arduino-uno"
transport = "serial"
path = "/dev/ttyACM0"
baud = 115200

[channels_config.telegram]
bot_token = "YOUR_BOT_TOKEN"
allowed_users = []

[gateway]
host = "127.0.0.1"
port = 42617
allow_public_bind = false
```

### 2.4 运行守护进程（仅本地）

```bash
zeroclaw daemon --host 127.0.0.1 --port 42617
```

- 网关绑定到`127.0.0.1` —— 无法从其他机器访问
- Telegram通道工作：ZeroClaw轮询Telegram API（出站）
- 不需要防火墙或端口转发

---

## 3. 绑定到0.0.0.0（本地网络）

要允许LAN上的其他设备访问网关（例如用于配对或webhook）：

### 3.1 选项A：明确选择加入

```toml
[gateway]
host = "0.0.0.0"
port = 42617
allow_public_bind = true
```

```bash
zeroclaw daemon --host 0.0.0.0 --port 42617
```

**安全：** `allow_public_bind = true`将网关暴露给您的本地网络。仅在受信任的LAN上使用。

### 3.2 选项B：隧道（推荐用于Webhook）

如果您需要**公共URL**（例如WhatsApp webhook、外部客户端）：

1. 在本地主机上运行网关：
   ```bash
   zeroclaw daemon --host 127.0.0.1 --port 42617
   ```

2. 启动隧道：
   ```toml
   [tunnel]
   provider = "tailscale"   # 或"ngrok"、"cloudflare"
   ```
   或使用`zeroclaw tunnel`（参见隧道文档）。

3. ZeroClaw将拒绝`0.0.0.0`，除非`allow_public_bind = true`或激活了隧道。

---

## 4. Telegram轮询（无入站端口）

Telegram默认使用**长轮询**：

- ZeroClaw调用`https://api.telegram.org/bot{token}/getUpdates`
- 不需要入站端口或公共IP
- 可在NAT后、RPi上、家庭实验室中工作

**配置：**

```toml
[channels_config.telegram]
bot_token = "YOUR_BOT_TOKEN"
allowed_users = []            # 默认拒绝，明确绑定身份
```

运行`zeroclaw daemon` —— Telegram通道自动启动。

要在运行时批准一个Telegram账户：

```bash
zeroclaw channel bind-telegram <IDENTITY>
```

`<IDENTITY>`可以是数字Telegram用户ID或用户名（不含`@`）。

### 4.1 单轮询器规则（重要）

Telegram Bot API `getUpdates`每个机器人令牌仅支持一个活动轮询器。

- 为同一令牌保持一个运行时实例（推荐：`zeroclaw daemon`服务）。
- 不要同时运行`cargo run -- channel start`或另一个机器人进程。

如果您遇到此错误：

`Conflict: terminated by other getUpdates request`

您遇到了轮询冲突。停止额外实例并仅重启一个守护进程。

---

## 5. Webhook通道（WhatsApp、Nextcloud Talk、自定义）

基于Webhook的通道需要**公共URL**，以便Meta（WhatsApp）或您的客户端可以POST事件。

### 5.1 Tailscale Funnel

```toml
[tunnel]
provider = "tailscale"
```

Tailscale Funnel通过`*.ts.net` URL暴露您的网关。无需端口转发。

### 5.2 ngrok

```toml
[tunnel]
provider = "ngrok"
```

或手动运行ngrok：
```bash
ngrok http 42617
# 为您的webhook使用HTTPS URL
```

### 5.3 Cloudflare Tunnel

配置Cloudflare Tunnel以转发到`127.0.0.1:42617`，然后将您的webhook URL设置为隧道的公共主机名。

---

## 6. 检查清单：RPi部署

- [ ] 使用`--features hardware`构建（如果使用本地GPIO则加上`peripheral-rpi`）
- [ ] 配置`[peripherals]`和`[channels_config.telegram]`
- [ ] 运行`zeroclaw daemon --host 127.0.0.1 --port 42617`（Telegram无需0.0.0.0即可工作）
- [ ] 对于LAN访问：`--host 0.0.0.0` + 配置中的`allow_public_bind = true`
- [ ] 对于webhook：使用Tailscale、ngrok或Cloudflare隧道

---

## 7. OpenRC（Alpine Linux服务）

ZeroClaw支持Alpine Linux和其他使用OpenRC init系统的发行版。OpenRC服务运行**系统范围**，需要root/sudo。

### 7.1 先决条件

- Alpine Linux（或其他基于OpenRC的发行版）
- Root或sudo访问权限
- 专用的`zeroclaw`系统用户（安装期间创建）

### 7.2 安装服务

```bash
# 安装服务（在Alpine上自动检测OpenRC）
sudo zeroclaw service install
```

这会创建：
- 初始化脚本：`/etc/init.d/zeroclaw`
- 配置目录：`/etc/zeroclaw/`
- 日志目录：`/var/log/zeroclaw/`

### 7.3 配置

通常不需要手动复制配置。

`sudo zeroclaw service install`自动准备`/etc/zeroclaw`，从您的用户设置迁移现有的运行时状态（如果可用），并为`zeroclaw`服务用户设置所有权/权限。

如果没有可迁移的先前运行时状态，请在启动服务之前创建`/etc/zeroclaw/config.toml`。

### 7.4 启用和启动

```bash
# 添加到默认运行级别
sudo rc-update add zeroclaw default

# 启动服务
sudo rc-service zeroclaw start

# 检查状态
sudo rc-service zeroclaw status
```

### 7.5 管理服务

| 命令 | 描述 |
|------|------|
| `sudo rc-service zeroclaw start` | 启动守护进程 |
| `sudo rc-service zeroclaw stop` | 停止守护进程 |
| `sudo rc-service zeroclaw status` | 检查服务状态 |
| `sudo rc-service zeroclaw restart` | 重启守护进程 |
| `sudo zeroclaw service status` | ZeroClaw状态包装器（使用`/etc/zeroclaw`配置） |

### 7.6 日志

OpenRC将日志路由到：

| 日志 | 路径 |
|------|------|
| 访问/标准输出 | `/var/log/zeroclaw/access.log` |
| 错误/标准错误 | `/var/log/zeroclaw/error.log` |

查看日志：

```bash
sudo tail -f /var/log/zeroclaw/error.log
```

### 7.7 卸载

```bash
# 停止并从运行级别移除
sudo rc-service zeroclaw stop
sudo rc-update del zeroclaw default

# 移除初始化脚本
sudo zeroclaw service uninstall
```

### 7.8 注意事项

- OpenRC仅**系统范围**（无用户级服务）
- 所有服务操作都需要`sudo`或root
- 服务以`zeroclaw:zeroclaw`用户运行（最低权限）
- 配置必须位于`/etc/zeroclaw/config.toml`（init脚本中的显式路径）
- 如果`zeroclaw`用户不存在，安装将失败并提供创建说明

### 7.9 检查清单：Alpine/OpenRC部署

- [ ] 安装：`sudo zeroclaw service install`
- [ ] 启用：`sudo rc-update add zeroclaw default`
- [ ] 启动：`sudo rc-service zeroclaw start`
- [ ] 验证：`sudo rc-service zeroclaw status`
- [ ] 检查日志：`/var/log/zeroclaw/error.log`

---

## 8. 参考资料

- [channels-reference.zh-CN.md](./channels-reference.zh-CN.md) —— 通道配置概述
- [matrix-e2ee-guide.zh-CN.md](./matrix-e2ee-guide.zh-CN.md) —— Matrix设置和加密房间故障排除
- [hardware-peripherals-design.zh-CN.md](./hardware-peripherals-design.zh-CN.md) —— 外设设计
- [adding-boards-and-tools.zh-CN.md](./adding-boards-and-tools.zh-CN.md) —— 硬件设置和添加开发板