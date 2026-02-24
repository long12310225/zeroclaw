# ZeroClaw沙箱策略

> ⚠️ **状态：提案/路线图**
>
> 本文档描述了提议的方法，可能包含假设的命令或配置。
> 有关当前运行时行为，请参见[config-reference.zh-CN.md](config-reference.zh-CN.md)、[operations-runbook.zh-CN.md](operations-runbook.zh-CN.md)和[troubleshooting.zh-CN.md](troubleshooting.zh-CN.md)。

## 问题
ZeroClaw目前具有应用层安全（允许列表、路径阻止、命令注入保护），但缺乏操作系统级别的隔离。如果攻击者在允许列表上，他们可以使用zeroclaw的用户权限运行任何允许的命令。

## 提议的解决方案

### 选项1：Firejail集成（推荐用于Linux）
Firejail提供用户空间沙箱，开销最小。

```rust
// src/security/firejail.rs
use std::process::Command;

pub struct FirejailSandbox {
    enabled: bool,
}

impl FirejailSandbox {
    pub fn new() -> Self {
        let enabled = which::which("firejail").is_ok();
        Self { enabled }
    }

    pub fn wrap_command(&self, cmd: &mut Command) -> &mut Command {
        if !self.enabled {
            return cmd;
        }

        // Firejail用沙箱包装任何命令
        let mut jail = Command::new("firejail");
        jail.args([
            "--private=home",           // 新的主目录
            "--private-dev",            // 最小的/dev
            "--nosound",                // 无音频
            "--no3d",                   // 无3D加速
            "--novideo",                // 无视频设备
            "--nowheel",                // 无输入设备
            "--notv",                   // 无电视设备
            "--noprofile",              // 跳过配置文件加载
            "--quiet",                  // 抑制警告
        ]);

        // 附加原始命令
        if let Some(program) = cmd.get_program().to_str() {
            jail.arg(program);
```