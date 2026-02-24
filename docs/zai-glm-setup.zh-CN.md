# Z.AI GLM设置

ZeroClaw通过OpenAI兼容端点支持Z.AI的GLM模型。
本指南涵盖了与当前ZeroClaw提供者行为匹配的实际设置选项。

## 概述

ZeroClaw开箱即用地支持这些Z.AI别名和端点：

| 别名 | 端点 | 说明 |
|------|------|------|
| `zai` | `https://api.z.ai/api/coding/paas/v4` | 全球端点 |
| `zai-cn` | `https://open.bigmodel.cn/api/paas/v4` | 中国端点 |

如果您需要自定义基础URL，请参见`docs/custom-providers.zh-CN.md`。

## 设置

### 快速开始

```bash
zeroclaw onboard \
  --provider "zai" \
  --api-key "YOUR_ZAI_API_KEY"
```

### 手动配置

编辑`~/.zeroclaw/config.toml`：

```toml
api_key = "YOUR_ZAI_API_KEY"
default_provider = "zai"
default_model = "glm-5"
default_temperature = 0.7
```

## 可用模型

| 模型 | 描述 |
|------|------|
| `glm-5` | 入门默认；最强推理能力 |
| `glm-4.7` | 强大的通用质量 |
| `glm-4.6` | 平衡基线 |
| `glm-4.5-air` | 低延迟选项 |

模型可用性可能因账户/地区而异，所以有疑问时请使用`/models` API。

## 验证设置

### 使用curl测试

```bash
# 测试OpenAI兼容端点
curl -X POST "https://api.z.ai/api/coding/paas/v4/chat/completions" \
  -H "Authorization: Bearer YOUR_ZAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-5",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

预期响应：
```json
{
  "choices": [{
    "message": {
      "content": "Hello! How can I help you today?",
      "role": "assistant"
    }
  }]
}
```

### 使用ZeroClaw CLI测试

```bash
# 直接测试代理
echo "Hello" | zeroclaw agent

# 检查状态
zeroclaw status
```

## 环境变量

添加到您的`.env`文件：

```bash
# Z.AI API密钥
ZAI_API_KEY=your-id.secret

# 可选的通用密钥（许多提供者使用）
# API_KEY=your-id.secret
```

密钥格式为`id.secret`（例如：`abc123.xyz789`）。

## 故障排除

### 速率限制

**症状：** `rate_limited`错误

**解决方案：**
- 等待并重试
- 检查您的Z.AI套餐限制
- 尝试`glm-4.5-air`以获得更低延迟和更高配额容忍度

### 认证错误

**症状：** 401或403错误

**解决方案：**
- 验证您的API密钥格式为`id.secret`
- 检查密钥是否已过期
- 确保密钥中没有多余的空格

### 找不到模型

**症状：** 模型不可用错误

**解决方案：**
- 列出可用模型：
```bash
curl -s "https://api.z.ai/api/coding/paas/v4/models" \
  -H "Authorization: Bearer YOUR_ZAI_API_KEY" | jq '.data[].id'
```

## 获取API密钥

1. 访问[Z.AI](https://z.ai)
2. 注册Coding套餐
3. 从仪表板生成API密钥
4. 密钥格式：`id.secret`（例如，`abc123.xyz789`）

## 相关文档

- [ZeroClaw README](../README.zh-CN.md)
- [自定义提供者端点](./custom-providers.zh-CN.md)
- [贡献指南](../CONTRIBUTING.zh-CN.md)