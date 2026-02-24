# Arduino Uno Q上的ZeroClaw — 分步指南

在Arduino Uno Q的Linux端运行ZeroClaw。Telegram通过WiFi工作；GPIO控制使用Bridge（需要最小的应用实验室应用程序）。

---

## 包含内容（无需代码更改）

ZeroClaw包含了Arduino Uno Q所需的一切。**克隆仓库并遵循本指南 —— 无需补丁或自定义代码。**

| 组件 | 位置 | 用途 |
|------|------|------|
| Bridge应用 | `firmware/zeroclaw-uno-q-bridge/` | MCU草图 + Python套接字服务器（端口9999）用于GPIO |
| Bridge工具 | `src/peripherals/uno_q_bridge.rs` | 通过TCP与Bridge通信的`gpio_read` / `gpio_write`工具 |
| 设置命令 | `src/peripherals/uno_q_setup.rs` | `zeroclaw peripheral setup-uno-q` 通过scp + arduino-app-cli部署Bridge |
| 配置模式 | `board = "arduino-uno-q"`, `transport = "bridge"` | 在`config.toml`中支持 |

使用`--features hardware`构建以包含Uno Q支持。

---

## 先决条件

- 已配置WiFi的Arduino Uno Q
- 在Mac上安装了Arduino App Lab（用于初始设置和部署）
- LLM的API密钥（OpenRouter等）

---

## 第一阶段：初始Uno Q设置（一次性）

### 1.1 通过App Lab配置Uno Q

1. 下载[Arduino App Lab](https://docs.arduino.cc/software/app-lab/)（Linux上的AppImage）。
2. 通过USB连接Uno Q，开机。
3. 打开App Lab，连接到开发板。
4. 遵循设置向导：
   - 设置用户名和密码（用于SSH）
   - 配置WiFi（SSID、密码）
   - 应用任何固件更新
5. 记下显示的IP地址（例如`arduino@192.168.1.42`）或稍后通过App Lab终端中的`ip addr show`找到它。

### 1.2 验证SSH访问

```bash
ssh arduino@<UNO_Q_IP>
# 输入您设置的密码
```

---

## 第二阶段：在Uno Q上安装ZeroClaw

### 选项A：在设备上构建（更简单，约20-40分钟）

```bash
# SSH进入Uno Q
ssh arduino@<UNO_Q_IP>

# 安装Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env

# 安装构建依赖（Debian）
sudo apt-get update
sudo apt-get install -y pkg-config libssl-dev

# 克隆zeroclaw（或scp您的项目）
git clone https://github.com/theonlyhennygod/zeroclaw.git
cd zeroclaw

# 构建（在Uno Q上需要约15-30分钟）
cargo build --release --features hardware

# 安装
sudo cp target/release/zeroclaw /usr/local/bin/
```

### 选项B：在Mac上交叉编译（更快）

```bash
# 在您的Mac上 — 添加aarch64目标
rustup target add aarch64-unknown-linux-gnu

# 安装交叉编译器（macOS；链接所需）
brew tap messense/macos-cross-toolchains
brew install aarch64-unknown-linux-gnu

# 构建
CC_aarch64_unknown_linux_gnu=aarch64-unknown-linux-gnu-gcc cargo build --release --target aarch64-unknown-linux-gnu --features hardware

# 复制到Uno Q
scp target/aarch64-unknown-linux-gnu/release/zeroclaw arduino@<UNO_Q_IP>:~/
ssh arduino@<UNO_Q_IP> "sudo mv ~/zeroclaw /usr/local/bin/"
```

如果交叉编译失败，请使用选项A并在设备上构建。

---

## 第三阶段：配置ZeroClaw

### 3.1 运行Onboard（或手动创建配置）

```bash
ssh arduino@<UNO_Q_IP>

# 快速配置
zeroclaw onboard --api-key YOUR_OPENROUTER_KEY --provider openrouter

# 或手动创建配置
mkdir -p ~/.zeroclaw/workspace
nano ~/.zeroclaw/config.toml
```

### 3.2 最小config.toml

```toml
api_key = "YOUR_OPENROUTER_API_KEY"
default_provider = "openrouter"
default_model = "anthropic/claude-sonnet-4-6"

[peripherals]
enabled = false
# 通过Bridge的GPIO需要第四阶段

[channels_config.telegram]
bot_token = "YOUR_TELEGRAM_BOT_TOKEN"
allowed_users = ["*"]

[gateway]
host = "127.0.0.1"
port = 42617
allow_public_bind = false

[agent]
compact_context = true
```

---

## 第四阶段：运行ZeroClaw守护进程

```bash
ssh arduino@<UNO_Q_IP>

# 运行守护进程（Telegram轮询通过WiFi工作）
zeroclaw daemon --host 127.0.0.1 --port 42617
```

**此时：** Telegram聊天工作正常。向您的机器人发送消息 —— ZeroClaw会回复。还没有GPIO功能。

---

## 第五阶段：通过Bridge使用GPIO（ZeroClaw处理）

ZeroClaw包含了Bridge应用和设置命令。

### 5.1 部署Bridge应用

**从您的Mac**（带有zeroclaw仓库）：
```bash
zeroclaw peripheral setup-uno-q --host 192.168.0.48
```

**从Uno Q**（已SSH登录）：
```bash
zeroclaw peripheral setup-uno-q
```

这会将Bridge应用复制到`~/ArduinoApps/zeroclaw-uno-q-bridge`并启动它。

### 5.2 添加到config.toml

```toml
[peripherals]
enabled = true

[[peripherals.boards]]
board = "arduino-uno-q"
transport = "bridge"
```

### 5.3 运行ZeroClaw

```bash
zeroclaw daemon --host 127.0.0.1 --port 42617
```

现在当您向Telegram机器人发送*"打开LED"*或*"设置引脚13为高电平"*这样的消息时，ZeroClaw会通过Bridge使用`gpio_write`。

---

## 总结：从开始到结束的命令

| 步骤 | 命令 |
|------|------|
| 1 | 在App Lab中配置Uno Q（WiFi、SSH） |
| 2 | `ssh arduino@<IP>` |
| 3 | `curl -sSf https://sh.rustup.rs \| sh -s -- -y && source ~/.cargo/env` |
| 4 | `sudo apt-get install -y pkg-config libssl-dev` |
| 5 | `git clone https://github.com/theonlyhennygod/zeroclaw.git && cd zeroclaw` |
| 6 | `cargo build --release --features hardware` |
| 7 | `zeroclaw onboard --api-key KEY --provider openrouter` |
| 8 | 编辑`~/.zeroclaw/config.toml`（添加Telegram bot_token） |
| 9 | `zeroclaw daemon --host 127.0.0.1 --port 42617` |
| 10 | 向您的Telegram机器人发送消息 —— 它会回复 |

---

## 故障排除

- **"command not found: zeroclaw"** — 使用完整路径：`/usr/local/bin/zeroclaw`或确保`~/.cargo/bin`在PATH中。
- **Telegram无响应** — 检查bot_token、allowed_users，并确保Uno Q有互联网连接（WiFi）。
- **内存不足** — 保持功能精简（Uno Q使用`--features hardware`）；考虑使用`compact_context = true`。
- **GPIO命令被忽略** — 确保Bridge应用正在运行（`zeroclaw peripheral setup-uno-q`部署并启动它）。配置必须有`board = "arduino-uno-q"`和`transport = "bridge"`。
- **LLM提供者（GLM/智谱）** — 使用`default_provider = "glm"`或`"zhipu"`，在环境变量或配置中设置`GLM_API_KEY`。ZeroClaw使用正确的v4端点。