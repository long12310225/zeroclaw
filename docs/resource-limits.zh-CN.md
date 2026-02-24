# ZeroClaw资源限制

> ⚠️ **状态：提案/路线图**
>
> 本文档描述了提议的方法，可能包含假设的命令或配置。
> 有关当前运行时行为，请参见[config-reference.zh-CN.md](config-reference.zh-CN.md)、[operations-runbook.zh-CN.md](operations-runbook.zh-CN.md)和[troubleshooting.zh-CN.md](troubleshooting.zh-CN.md)。

## 问题
ZeroClaw有限速（20个操作/小时）但没有资源上限。失控的代理可能会：
- 耗尽可用内存
- 使CPU达到100%
- 用日志/输出填满磁盘

---

## 提议的解决方案

### 选项1：cgroups v2（Linux，推荐）
自动为zeroclaw创建一个带有限制的cgroup。

```bash
# 创建带有限制的systemd服务
[Service]
MemoryMax=512M
CPUQuota=100%
IOReadBandwidthMax=/dev/sda 10M
IOWriteBandwidthMax=/dev/sda 10M
TasksMax=100
```

### 选项2：tokio::task::死锁检测
防止任务饥饿。

```rust
use tokio::time::{timeout, Duration};

pub async fn execute_with_timeout<F, T>(
    fut: F,
    cpu_time_limit: Duration,
    memory_limit: usize,
) -> Result<T>
where
    F: Future<Output = Result<T>>,
{
    // CPU超时
    timeout(cpu_time_limit, fut).await?
}
```

### 选项3：内存监控
跟踪堆使用情况，如果超过限制则终止。

```rust
use std::alloc::{GlobalAlloc, Layout, System};

struct LimitedAllocator<A> {
    inner: A,
    max_bytes: usize,
    used: std::sync::atomic::AtomicUsize,
}

unsafe impl<A: GlobalAlloc> GlobalAlloc for LimitedAllocator<A> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let current = self.used.fetch_add(layout.size(), std::sync::atomic::Ordering::Relaxed);
        if current + layout.size() > self.max_bytes {
            std::process::abort();
        }
        self.inner.alloc(layout)
    }
}
```

---

## 配置模式

```toml
[resources]
# 内存限制（以MB为单位）
max_memory_mb = 512
max_memory_per_command_mb = 128

# CPU限制
max_cpu_percent = 50
max_cpu_time_seconds = 60

# 磁盘I/O限制
max_log_size_mb = 100
max_temp_storage_mb = 500

# 进程限制
max_subprocesses = 10
max_open_files = 100
```

---

## 实施优先级

| 阶段 | 功能 | 工作量 | 影响 |
|------|------|--------|------|
| **P0** | 内存监控 + 终止 | 低 | 高 |
| **P1** | 每个命令的CPU超时 | 低 | 高 |
| **P2** | cgroups集成（Linux） | 中等 | 很高 |
| **P3** | 磁盘I/O限制 | 中等 | 中等 |