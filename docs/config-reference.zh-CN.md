# ZeroClaw配置参考（面向运维人员）

这是常见配置部分和默认值的高信号参考。

最后验证：**2026年2月21日**。

启动时的配置路径解析：

1. `ZEROCLAW_WORKSPACE`覆盖（如果设置）
2. 持久化的`~/.zeroclaw/active_workspace.toml`标记（如果存在）
3. 默认的`~/.zeroclaw/config.toml`

ZeroClaw在启动时以`INFO`级别记录解析的配置：

- `Config loaded`包含字段：`path`、`workspace`、`source`、`initialized`

模式导出命令：

- `zeroclaw config schema`（将JSON Schema草案2020-12打印到stdout）

## 核心键

| 键 | 默认值 | 说明 |
|----|--------|------|
| `default_provider` | `openrouter` | 提供者ID或别名 |
| `default_model` | `anthropic/claude-sonnet-4-6` | 通过所选提供者路由的模型 |
| `default_temperature` | `0.7` | 模型温度 |

## `[observability]`

| 键 | 默认值 | 用途 |
|----|--------|------|
| `backend` | `none` | 可观察性后端：`none`、`noop`、`log`、`prometheus`、`otel`、`opentelemetry`或`otlp` |
| `otel_endpoint` | `http://localhost:4318` | 当后端为`otel`时使用的OTLP HTTP端点 |
| `otel_service_name` | `zeroclaw` | 发送到OTLP收集器的服务名称 |
| `runtime_trace_mode` | `none` | 运行时跟踪存储模式：`none`、`rolling`或`full` |
| `runtime_trace_path` | `state/runtime-trace.jsonl` | 运行时跟踪JSONL路径（相对于工作区，除非是绝对路径） |
| `runtime_trace_max_entries` | `200` | 当`runtime_trace_mode = "rolling"`时保留的最大事件数 |

说明：

- `backend = "otel"`使用带有阻塞导出客户端的OTLP HTTP导出，因此可以从非Tokio上下文中安全地发出跨度和指标。
- 别名值`opentelemetry`和`otlp`映射到相同的OTel后端。
- 运行时跟踪旨在调试工具调用失败和格式错误的模型工具负载。它们可能包含模型输出文本，因此在共享主机上默认保持禁用。
- 查询运行时跟踪：
  - `zeroclaw doctor traces --limit 20`
  - `zeroclaw doctor traces --event tool_call_result --contains \"error\"`
  - `zeroclaw doctor traces --id <trace-id>`