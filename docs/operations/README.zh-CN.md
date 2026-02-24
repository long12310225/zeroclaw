# 运营和部署文档

适用于在持久性或类生产环境中运行ZeroClaw的操作员。

## 核心运营

- 日常运行手册：[../operations-runbook.zh-CN.md](../operations-runbook.zh-CN.md)
- 发布运行手册：[../release-process.zh-CN.md](../release-process.zh-CN.md)
- 故障排除矩阵：[../troubleshooting.zh-CN.md](../troubleshooting.zh-CN.md)
- 安全网络/网关部署：[../network-deployment.zh-CN.md](../network-deployment.zh-CN.md)
- Mattermost设置（通道特定）：[../mattermost-setup.zh-CN.md](../mattermost-setup.zh-CN.md)

## 常见流程

1. 验证运行时（`status`、`doctor`、`channel doctor`）
2. 一次应用一个配置更改
3. 重启服务/守护进程
4. 验证通道和网关健康状况
5. 如果行为倒退则快速回滚

## 相关

- 配置参考：[../config-reference.zh-CN.md](../config-reference.zh-CN.md)
- 安全集合：[../security/README.zh-CN.md](../security/README.zh-CN.md)