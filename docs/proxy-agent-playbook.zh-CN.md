# 代理代理手册

本手册提供了通过`proxy_config`配置代理行为的复制粘贴工具调用。

当您希望代理快速安全地切换代理范围时，请使用本文档。

## 0. 概要

- **目的：** 为代理范围管理和回滚提供即用型代理工具调用。
- **受众：** 在代理网络中运行ZeroClaw的操作员和维护者。
- **范围：** `proxy_config`操作、模式选择、验证流程和故障排除。
- **非目标：** ZeroClaw运行时行为之外的通用网络调试。

---

## 1. 按意图快速路径

使用本节进行快速操作路由。

### 1.1 仅为ZeroClaw内部流量代理

1. 使用范围`zeroclaw`。
2. 设置`http_proxy`/`https_proxy`或`all_proxy`。
3. 使用`{"action":"get"}`验证。

前往：

- [第4节](#4-模式a--仅为zeroclaw内部代理)

### 1.2 仅为选定服务代理

1. 使用范围`services`。
2. 在`services`中设置具体键或通配符选择器。
3. 使用`{"action":"list_services"}`验证覆盖范围。

前往：

- [第5节](#5-模式b--仅为特定服务代理)

### 1.3 导出进程范围的代理环境变量

1. 使用范围`environment`。
2. 使用`{"action":"apply_env"}`应用。
3. 通过`{"action":"get"}`验证环境快照。

前往：

- [第6节](#6-模式c--为完整进程环境代理)

### 1.4 紧急回滚

1. 禁用代理。
2. 如需要，清除环境导出。
3. 重新检查运行时和环境快照。

前往：

- [第7节](#7-禁用--回滚模式)

---

## 2. 范围决策矩阵

| 范围 | 影响 | 导出环境变量 | 典型用途 |
|------|------|--------------|----------|
| `zeroclaw` | ZeroClaw内部HTTP客户端 | 否 | 正常运行时代理，无进程级副作用 |
| `services` | 仅选定的服务键/选择器 | 否 | 特定提供者/工具/通道的细粒度路由 |
| `environment` | 运行时+进程环境代理变量 | 是 | 需要`HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`的集成 |

---

## 3. 标准安全工作流程

每次代理更改都使用此序列：

1. 检查当前状态。
2. 发现有效的服务键/选择器。
3. 应用目标范围配置。
4. 验证运行时和环境快照。
5. 如果行为不符合预期则回滚。

工具调用：

```json
{"action":"get"}
{"action":"list_services"}
```

---

## 4. 模式A — 仅为ZeroClaw内部代理

当ZeroClaw提供者/通道/工具HTTP流量应使用代理时使用，无需导出进程级代理环境变量。

工具调用：

```json
{"action":"set","enabled":true,"scope":"zeroclaw","http_proxy":"http://127.0.0.1:7890","https_proxy":"http://127.0.0.1:7890","no_proxy":["localhost","127.0.0.1"]}
{"action":"get"}
```

预期行为：

- 运行时代理对ZeroClaw HTTP客户端处于活动状态。
- 不需要`HTTP_PROXY` / `HTTPS_PROXY`进程环境导出。

---

## 5. 模式B — 仅为特定服务代理

当系统的一部分应使用代理时使用（例如特定的提供者/工具/通道）。

### 5.1 针对特定服务

```json
{"action":"set","enabled":true,"scope":"services","services":["provider.openai","tool.http_request","channel.telegram"],"all_proxy":"socks5h://127.0.0.1:1080","no_proxy":["localhost","127.0.0.1",".internal"]}
{"action":"get"}
```

### 5.2 按选择器针对

```json
{"action":"set","enabled":true,"scope":"services","services":["provider.*","tool.*"],"http_proxy":"http://127.0.0.1:7890"}
{"action":"get"}
```

预期行为：

- 仅匹配的服务使用代理。
- 不匹配的服务绕过代理。

---

## 6. 模式C — 为完整进程环境代理

当您故意需要为运行时集成为`HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`NO_PROXY`导出进程环境变量时使用。

### 6.1 配置并应用环境范围

```json
{"action":"set","enabled":true,"scope":"environment","http_proxy":"http://127.0.0.1:7890","https_proxy":"http://127.0.0.1:7890","no_proxy":"localhost,127.0.0.1,.internal"}
{"action":"apply_env"}
{"action":"get"}
```

预期行为：

- 运行时代理处于活动状态。
- 为进程导出了环境变量。

---

## 7. 禁用/回滚模式

### 7.1 禁用代理（默认安全行为）

```json
{"action":"disable"}
{"action":"get"}
```

### 7.2 禁用代理并强制清除环境变量

```json
{"action":"disable","clear_env":true}
{"action":"get"}
```

### 7.3 保持代理启用但仅清除环境导出

```json
{"action":"clear_env"}
{"action":"get"}
```

---

## 8. 常见操作配方

### 8.1 从环境范围代理切换到仅服务代理

```json
{"action":"set","enabled":true,"scope":"services","services":["provider.openai","tool.http_request"],"all_proxy":"socks5://127.0.0.1:1080"}
{"action":"get"}
```

### 8.2 添加一个更多代理服务

```json
{"action":"set","scope":"services","services":["provider.openai","tool.http_request","channel.slack"]}
{"action":"get"}
```

### 8.3 使用选择器重置`services`列表

```json
{"action":"set","scope":"services","services":["provider.*","channel.telegram"]}
{"action":"get"}
```

---

## 9. 故障排除

- 错误：`proxy.scope='services' requires a non-empty proxy.services list`
  - 修复：设置至少一个具体的服务键或选择器。

- 错误：无效的代理URL方案
  - 允许的方案：`http`、`https`、`socks5`、`socks5h`。

- 代理未按预期应用
  - 运行`{"action":"list_services"}`并验证服务名称/选择器。
  - 运行`{"action":"get"}`并检查`runtime_proxy`和`environment`快照值。

---

## 10. 相关文档

- [README.md](./README.md) — 文档索引和分类法。
- [network-deployment.zh-CN.md](./network-deployment.zh-CN.md) — 端到端网络部署和隧道拓扑指导。
- [resource-limits.zh-CN.md](./resource-limits.zh-CN.md) — 网络/工具执行上下文的运行时安全限制。

---

## 11. 维护说明

- **所有者：** 运行时和工具维护者。
- **更新触发：** 新的`proxy_config`操作、代理范围语义或支持的服务选择器更改。
- **最后审查：** 2026-02-18。