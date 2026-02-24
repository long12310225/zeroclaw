# 通道参考

本文档是ZeroClaw中通道配置的权威参考。

对于加密的Matrix房间，请 также阅读专门的运行手册：
- [Matrix E2EE指南](./matrix-e2ee-guide.zh-CN.md)

## 快速路径

- 需要按通道的完整配置参考：跳转到[每通道配置示例](#4-每通道配置示例)。
- 需要无响应诊断流程：跳转到[故障排除检查清单](#6-故障排除检查清单)。
- 需要Matrix加密房间帮助：使用[Matrix E2EE指南](./matrix-e2ee-guide.zh-CN.md)。
- 需要Nextcloud Talk机器人设置：使用[Nextcloud Talk设置](./nextcloud-talk-setup.zh-CN.md)。
- 需要部署/网络假设（轮询vs webhook）：使用[网络部署](./network-deployment.zh-CN.md)。

## 常见问题：Matrix设置通过但无回复

这是最常见的症状（与问题#499同类）。按顺序检查这些：

1. **允许列表不匹配**：`allowed_users`不包括发送者（或为空）。
2. **错误的房间目标**：机器人未加入到配置的`room_id`/别名目标房间。
3. **令牌/账户不匹配**：令牌有效但属于另一个Matrix账户。
4. **E2EE设备身份差距**：`whoami`不返回`device_id`且配置未提供一个。
5. **密钥共享/信任差距**：房间密钥未共享给机器人设备，因此无法解密加密事件。
6. **陈旧的运行时状态**：配置已更改但`zeroclaw daemon`未重启。

---

## 1. 配置命名空间

所有通道设置都位于`~/.zeroclaw/config.toml`中的`channels_config`下。

```toml
[channels_config]
cli = true
```

通过创建其子表来启用每个通道（例如，`[channels_config.telegram]`）。

## 聊天内运行时模型切换（Telegram/Discord）

运行`zeroclaw channel start`（或守护进程模式）时，Telegram和Discord现在支持发送者范围的运行时切换：

- `/models` — 显示可用的提供者和当前选择
- `/models <provider>` — 为当前发送者会话切换提供者
- `/model` — 显示当前模型和缓存的模型ID（如果可用）
- `/model <model-id>` — 为当前发送者会话切换模型

注意事项：

- 切换仅清除该发送者的内存对话历史，以避免跨模型上下文污染。