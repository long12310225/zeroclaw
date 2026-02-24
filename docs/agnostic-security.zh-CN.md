# 无关安全：对可移植性零影响

> ⚠️ **状态：提案/路线图**
>
> 本文档描述了提议的方法，可能包含假设的命令或配置。
> 有关当前运行时行为，请参见[config-reference.md](config-reference.zh-CN.md)、[operations-runbook.md](operations-runbook.zh-CN.md)和[troubleshooting.md](troubleshooting.zh-CN.md)。

## 核心问题：安全功能会破坏...
1. ❓ 快速交叉编译构建？
2. ❓ 可插拔架构（交换任何组件）？
3. ❓ 硬件无关性（ARM、x86、RISC-V）？
4. ❓ 小型硬件支持（<5MB RAM，$10开发板）？

**答案：全部否定** —— 安全设计为**可选功能标志**，具有**平台特定的条件编译**。

---

## 1. 构建速度：功能门控安全

### Cargo.toml：安全功能位于功能之后

```toml
[features]
default = ["basic-security"]

# 基础安全（始终开启，零开销）
basic-security = []

# 平台特定沙箱（按平台选择加入）
sandbox-landlock = []   # 仅限Linux
sandbox-firejail = []  # 仅限Linux
sandbox-bubblewrap = []# macOS/Linux
sandbox-docker = []    # 所有平台（重量级）

# 完整安全套件（用于生产构建）
security-full = [
    "basic-security",
    "sandbox-landlock",
    "resource-monitoring",
    "audit-logging",
]

# 资源和审计监控
resource-monitoring = []
audit-logging = []

# 开发构建（最快，无额外依赖）
dev = []
```

### 构建命令（选择您的配置文件）

```bash
# 超快开发构建（无安全附加功能）
cargo build --profile dev

# 发布构建与基础安全（默认）
cargo build --release
# → 包含：允许列表、路径阻止、注入保护
# → 排除：Landlock、Firejail、审计日志

# 具有完整安全性的生产构建
cargo build --release --features security-full
# → 包含：所有功能

# 仅平台特定沙箱
cargo build --release --features sandbox-landlock  # Linux
cargo build --release --features sandbox-docker    # 所有平台
```

### 条件编译：禁用时零开销

```rust
// src/security/mod.rs

#[cfg(feature = "sandbox-landlock")]
mod landlock;
#[cfg(feature = "sandbox-landlock")]
pub use landlock::LandlockSandbox;

#[cfg(feature = "sandbox-firejail")]
mod firejail;
#[cfg(feature = "sandbox-firejail")]
pub use firejail::FirejailSandbox;

// 始终包含的基础安全（无功能标志）
pub mod policy;  // 允许列表、路径阻止、注入保护
```

**结果**：当功能被禁用时，代码甚至不会被编译 —— **零二进制膨胀**。

---

## 2. 可插拔架构：安全也是一个Trait

### 安全后端Trait（像其他一切一样可交换）

```rust
// src/security/traits.rs

#[async_trait]
pub trait Sandbox: Send + Sync {
    /// 用沙箱保护包装命令
    fn wrap_command(&self, cmd: &mut std::process::Command) -> std::io::Result<()>;

    /// 检查沙箱在此平台上是否可用
    fn is_available(&self) -> bool;

    /// 人类可读的名称
    fn name(&self) -> &str;
}

// 无操作沙箱（始终可用）
pub struct NoopSandbox;

impl Sandbox for NoopSandbox {
    fn wrap_command(&self, _cmd: &mut std::process::Command) -> std::io::Result<()> {
        Ok(())  // 原样传递
    }

    fn is_available(&self) -> bool { true }
    fn name(&self) -> &str { "none" }
}
```

### 工厂模式：基于功能自动选择

```rust
// src/security/factory.rs

pub fn create_sandbox() -> Box<dyn Sandbox> {
    #[cfg(feature = "sandbox-landlock")]
    {
        if LandlockSandbox::is_available() {
            return Box::new(LandlockSandbox::new());
        }
    }

    #[cfg(feature = "sandbox-firejail")]
    {
        if FirejailSandbox::is_available() {
            return Box::new(FirejailSandbox::new());
        }
    }

    #[cfg(feature = "sandbox-bubblewrap")]
    {
        if BubblewrapSandbox::is_available() {
            return Box::new(BubblewrapSandbox::new());
        }
    }

    #[cfg(feature = "sandbox-docker")]
    {
        if DockerSandbox::is_available() {
            return Box::new(DockerSandbox::new());
        }
    }

    // 回退：始终可用
    Box::new(NoopSandbox)
}
```

**就像提供者、通道和内存一样 —— 安全也是可插拔的！**

---

## 3. 硬件无关性：相同二进制，不同平台

### 跨平台行为矩阵

| 平台 | 构建于 | 运行时行为 |
|------|--------|------------|
| **Linux ARM**（树莓派） | ✅ 是 | Landlock → 无（优雅降级） |
| **Linux x86_64** | ✅ 是 | Landlock → Firejail → 无 |
| **macOS ARM**（M1/M2） | ✅ 是 | Bubblewrap → 无 |
| **macOS x86_64** | ✅ 是 | Bubblewrap → 无 |
| **Windows ARM** | ✅ 是 | 无（应用层） |
| **Windows x86_64** | ✅ 是 | 无（应用层） |
| **RISC-V Linux** | ✅ 是 | Landlock → 无 |

### 工作原理：运行时检测

```rust
// src/security/detect.rs

impl SandboxingStrategy {
    /// 在运行时选择最佳可用沙箱
    pub fn detect() -> SandboxingStrategy {
        #[cfg(target_os = "linux")]
        {
            // 首先尝试Landlock（内核功能检测）
            if Self::probe_landlock() {
                return SandboxingStrategy::Landlock;
            }

            // 尝试Firejail（用户空间工具检测）
            if Self::probe_firejail() {
                return SandboxingStrategy::Firejail;
            }
        }

        #[cfg(target_os = "macos")]
        {
            if Self::probe_bubblewrap() {
                return SandboxingStrategy::Bubblewrap;
            }
        }

        // 始终可用的回退
        SandboxingStrategy::ApplicationLayer
    }
}
```

**相同的二进制在各处运行** —— 它只是根据可用内容调整其保护级别。

---

## 4. 小型硬件：内存影响分析

### 二进制大小影响（估计）

| 功能 | 代码大小 | RAM开销 | 状态 |
|------|----------|---------|------|
| **基础ZeroClaw** | 3.4MB | <5MB | ✅ 当前 |
| **+ Landlock** | +50KB | +100KB | ✅ Linux 5.13+ |
| **+ Firejail包装器** | +20KB | +0KB（外部） | ✅ Linux + firejail |
| **+ 内存监控** | +30KB | +50KB | ✅ 所有平台 |
| **+ 审计日志** | +40KB | +200KB（缓冲） | ✅ 所有平台 |
| **完整安全** | +140KB | +350KB | ✅ 仍<6MB总计 |

### $10硬件兼容性

| 硬件 | RAM | ZeroClaw（基础） | ZeroClaw（完整安全） | 状态 |
|------|-----|------------------|----------------------|------|
| **树莓派Zero** | 512MB | ✅ 2% | ✅ 2.5% | 工作正常 |
| **Orange Pi Zero** | 512MB | ✅ 2% | ✅ 2.5% | 工作正常 |
| **NanoPi NEO** | 256MB | ✅ 4% | ✅ 5% | 工作正常 |
| **C.H.I.P.** | 512MB | ✅ 2% | ✅ 2.5% | 工作正常 |
| **Rock64** | 1GB | ✅ 1% | ✅ 1.2% | 工作正常 |

**即使具有完整安全性，ZeroClaw在$10开发板上使用的RAM也<5%。**

---

## 5. 无关交换：一切都保持可插拔

### ZeroClaw的核心承诺：交换任何东西

```rust
// 提供者（已经可插拔）
Box<dyn Provider>

// 通道（已经可插拔）
Box<dyn Channel>

// 内存（已经可插拔）
Box<dyn MemoryBackend>

// 隧道（已经可插拔）
Box<dyn Tunnel>

// 现在也是：安全（新可插拔）
Box<dyn Sandbox>
Box<dyn Auditor>
Box<dyn ResourceMonitor>
```

### 通过配置交换安全后端

```toml
# 不使用沙箱（最快，仅应用层）
[security.sandbox]
backend = "none"

# 使用Landlock（Linux内核LSM，原生）
[security.sandbox]
backend = "landlock"

# 使用Firejail（用户空间，需要安装firejail）
[security.sandbox]
backend = "firejail"

# 使用Docker（最重，最隔离）
[security.sandbox]
backend = "docker"
```

**就像将OpenAI交换为Gemini，或将SQLite交换为PostgreSQL一样。**

---

## 6. 依赖影响：最小新增依赖

### 当前依赖（供参考）
```
reqwest, tokio, serde, anyhow, uuid, chrono, rusqlite,
axum, tracing, opentelemetry, ...
```

### 安全功能依赖

| 功能 | 新依赖 | 平台 |
|------|--------|------|
| **Landlock** | `landlock` crate（纯Rust） | 仅限Linux |
| **Firejail** | 无（外部二进制） | 仅限Linux |
| **Bubblewrap** | 无（外部二进制） | macOS/Linux |
| **Docker** | `bollard` crate（Docker API） | 所有平台 |
| **内存监控** | 无（std::alloc） | 所有平台 |
| **审计日志** | 无（已有hmac/sha2） | 所有平台 |

**结果**：大多数功能添加**零新的Rust依赖** —— 它们要么：
1. 使用纯Rust crate（landlock）
2. 包装外部二进制（Firejail、Bubblewrap）
3. 使用现有依赖（hmac、sha2已在Cargo.toml中）

---

## 总结：核心价值主张保持不变

| 价值主张 | 之前 | 之后（带安全） | 状态 |
|----------|------|----------------|------|
| **<5MB RAM** | ✅ <5MB | ✅ <6MB（最坏情况） | ✅ 保持 |
| **<10ms启动** | ✅ <10ms | ✅ <15ms（检测） | ✅ 保持 |
| **3.4MB二进制** | ✅ 3.4MB | ✅ 3.5MB（带所有功能） | ✅ 保持 |
| **ARM + x86 + RISC-V** | ✅ 全部 | ✅ 全部 | ✅ 保持 |
| **$10硬件** | ✅ 工作 | ✅ 工作 | ✅ 保持 |
| **一切可插拔** | ✅ 是 | ✅ 是（安全也一样） | ✅ 增强 |
| **跨平台** | ✅ 是 | ✅ 是 | ✅ 保持 |

---

## 关键：功能标志 + 条件编译

```bash
# 开发者构建（最快，无额外功能）
cargo build --profile dev

# 标准发布（您当前的构建）
cargo build --release

# 具有完整安全性的生产版本
cargo build --release --features security-full

# 针对特定硬件
cargo build --release --target aarch64-unknown-linux-gnu  # 树莓派
cargo build --release --target riscv64gc-unknown-linux-gnu # RISC-V
cargo build --release --target armv7-unknown-linux-gnueabihf  # ARMv7
```

**每个目标、每个平台、每个用例 —— 仍然快速、仍然小巧、仍然无关。**