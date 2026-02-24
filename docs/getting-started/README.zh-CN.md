# 入门文档

用于首次设置和快速入门。

## 开始路径

1. 主要概述和快速开始：[../../README.zh-CN.md](../../README.zh-CN.md)
2. 一键设置和双重引导模式：[../one-click-bootstrap.zh-CN.md](../one-click-bootstrap.zh-CN.md)
3. 按任务查找命令：[../commands-reference.zh-CN.md](../commands-reference.zh-CN.md)

## 选择您的路径

| 场景 | 命令 |
|------|------|
| 我有API密钥，想要最快设置 | `zeroclaw onboard --api-key sk-... --provider openrouter` |
| 我想要引导提示 | `zeroclaw onboard --interactive` |
| 配置已存在，只需修复通道 | `zeroclaw onboard --channels-only` |
| 配置已存在，我故意想要完全覆盖 | `zeroclaw onboard --force` |
| 使用订阅认证 | 参见[订阅认证](../../README.zh-CN.md#subscription-auth-openai-codex--claude-code) |

## 入门和验证

- 快速入门：`zeroclaw onboard --api-key "sk-..." --provider openrouter`
- 交互式入门：`zeroclaw onboard --interactive`
- 现有配置保护：重新运行需要明确确认（或在非交互式流程中使用`--force`）
- Ollama云模型（`:cloud`）需要远程`api_url`和API密钥（例如`api_url = "https://ollama.com"`）。
- 验证环境：`zeroclaw status` + `zeroclaw doctor`

## 下一步

- 运行时操作：[../operations/README.zh-CN.md](../operations/README.zh-CN.md)
- 参考目录：[../reference/README.zh-CN.md](../reference/README.zh-CN.md)