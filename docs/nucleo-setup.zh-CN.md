# Nucleo-F401RE上的ZeroClaw — 分步指南

在您的Mac或Linux主机上运行ZeroClaw。通过USB连接Nucleo-F401RE。通过Telegram或CLI控制GPIO（LED、引脚）。

---

## 通过Telegram获取开发板信息（无需固件）

ZeroClaw可以通过USB**无需刷写任何固件**从Nucleo读取芯片信息。向您的Telegram机器人发送消息：

- *"我有什么开发板信息？"*
- *"开发板信息"*
- *"连接了什么硬件？"*
- *"芯片信息"*

代理使用`hardware_board_info`工具返回芯片名称、架构和内存映射。使用`probe`功能时，它通过USB/SWD读取实时数据；否则返回静态数据手册信息。

**配置：** 首先在`config.toml`中添加Nucleo（这样代理就知道要查询哪个开发板）：

```toml
[[peripherals.boards]]
board = "nucleo-f401re"
transport = "serial"
path = "/dev/ttyACM0"
baud = 115200
```

**CLI替代方案：**

```bash
cargo build --features hardware,probe
zeroclaw hardware info
zeroclaw hardware discover
```

---

## 包含内容（无需代码更改）

ZeroClaw包含了Nucleo-F401RE所需的一切：

| 组件 | 位置 | 用途 |
|------|------|------|
| 固件 | `firmware/zeroclaw-nucleo/` | Embassy Rust — USART2（115200），gpio_read，gpio_write |
| 串口外设 | `src/peripherals/serial.rs` | JSON-over-serial协议（与Arduino/ESP32相同） |
| 刷写命令 | `zeroclaw peripheral flash-nucleo` | 构建固件，通过probe-rs刷写 |

协议：换行符分隔的JSON。请求：`{"id":"1","cmd":"gpio_write","args":{"pin":13,"value":1}}`。响应：`{"id":"1","ok":true,"result":"done"}`。

---

## 先决条件

- Nucleo-F401RE开发板
- USB线缆（USB-A到Mini-USB；Nucleo内置ST-Link）
- 用于刷写的：`cargo install probe-rs-tools --locked`（或使用[安装脚本](https://probe.rs/docs/getting-started/installation/)）

---

## 第一阶段：刷写固件

### 1.1 连接Nucleo

1. 通过USB将Nucleo连接到您的Mac/Linux。
2. 开发板显示为USB设备（ST-Link）。现代系统上无需单独驱动程序。

### 1.2 通过ZeroClaw刷写

从zeroclaw仓库根目录：

```bash
zeroclaw peripheral flash-nucleo
```

这会构建`firmware/zeroclaw-nucleo`并通过`probe-rs run --chip STM32F401RETx`刷写。固件在刷写后立即运行。

### 1.3 手动刷写（替代方案）

```bash
cd firmware/zeroclaw-nucleo
cargo build --release --target thumbv7em-none-eabihf
probe-rs run --chip STM32F401RETx target/thumbv7em-none-eabihf/release/zeroclaw-nucleo
```

---

## 第二阶段：查找串口

- **macOS：** `/dev/cu.usbmodem*`或`/dev/tty.usbmodem*`（例如`/dev/cu.usbmodem101`）
- **Linux：** `/dev/ttyACM0`（或插入后检查`dmesg`）

USART2（PA2/PA3）桥接到ST-Link的虚拟COM端口，因此主机看到一个串口设备。

---

## 第三阶段：配置ZeroClaw

添加到`~/.zeroclaw/config.toml`：

```toml
[peripherals]
enabled = true

[[peripherals.boards]]
board = "nucleo-f401re"
transport = "serial"
path = "/dev/cu.usbmodem101"   # 调整为您的端口
baud = 115200
```

---

## 第四阶段：运行和测试

```bash
zeroclaw daemon --host 127.0.0.1 --port 42617
```

或直接使用代理：

```bash
zeroclaw agent --message "打开13号引脚的LED"
```

引脚13 = PA5 = Nucleo-F401RE上的用户LED（LD2）。

---

## 总结：命令

| 步骤 | 命令 |
|------|------|
| 1 | 通过USB连接Nucleo |
| 2 | `cargo install probe-rs-tools --locked` |
| 3 | `zeroclaw peripheral flash-nucleo` |
| 4 | 在config.toml中添加Nucleo（path = 您的串口） |
| 5 | `zeroclaw daemon`或`zeroclaw agent -m "打开LED"` |

---

## 故障排除

- **未识别flash-nucleo** — 从仓库构建：`cargo run --features hardware -- peripheral flash-nucleo`。子命令仅在仓库构建中，不在crates.io安装中。
- **找不到probe-rs** — `cargo install probe-rs-tools --locked`（`probe-rs` crate是库；CLI在`probe-rs-tools`中）
- **未检测到探针** — 确保Nucleo已连接。尝试另一根USB线缆/端口。
- **找不到串口** — 在Linux上，将用户添加到`dialout`：`sudo usermod -a -G dialout $USER`，然后注销/登录。
- **GPIO命令被忽略** — 检查配置中的`path`与您的串口匹配。运行`zeroclaw peripheral list`验证。