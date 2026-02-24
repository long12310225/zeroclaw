# 自定义提供者配置

ZeroClaw支持OpenAI兼容和Anthropic兼容提供者的自定义API端点。

## 提供者类型

### OpenAI兼容端点（`custom:`）

对于实现OpenAI API格式的服务：

```toml
default_provider = "custom:https://your-api.com"
api_key = "your-api-key"
default_model = "your-model-name"
```

### Anthropic兼容端点（`anthropic-custom:`）

对于实现Anthropic API格式的服务：

```toml
default_provider = "anthropic-custom:https://your-api.com"
api_key = "your-api-key"
default_model = "your-model-name"
```

## 配置方法

### 配置文件

编辑`~/.zeroclaw/config.toml`：

```toml
api_key = "your-api-key"
default_provider = "anthropic-custom:https://api.example.com"
default_model = "claude-sonnet-4-6"
```

### 环境变量

对于`custom:`和`anthropic-custom:`提供者，使用通用密钥环境变量：

```bash
export API_KEY="your-api-key"
# 或：export ZEROCLAW_API_KEY="your-api-key"
zeroclaw agent
```

## llama.cpp服务器（推荐的本地设置）

ZeroClaw包含`llama-server`的一流本地提供者：

- 提供者ID：`llamacpp`（别名：`llama.cpp`）
- 默认端点：`http://localhost:8080/v1`
- API密钥是可选的，除非`llama-server`启动时带有`--api-key`

启动本地服务器（示例）：

```bash
llama-server -hf ggml-org/gpt-oss-20b-GGUF --jinja -c 133000 --host 127.0.0.1 --port 8033
```

然后配置ZeroClaw：

```toml
default_provider = "llamacpp"
api_url = "http://127.0.0.1:8033/v1"
default_model = "ggml-org/gpt-oss-20b-GGUF"
default_temperature = 0.7
```

快速验证：

```bash
zeroclaw models refresh --provider llamacpp
zeroclaw agent -m "hello"
```

此流程不需要导出`ZEROCLAW_API_KEY=dummy`。

## SGLang服务器

ZeroClaw包含[SGLang](https://github.com/sgl-project/sglang)的一流本地提供者：

- 提供者ID：`sglang`
- 默认端点：`http://localhost:30000/v1`
- API密钥是可选的，除非服务器需要认证

启动本地服务器（示例）：

```bash
python -m sglang.launch_server --model meta-llama/Llama-3.1-8B-Instruct --port 30000
```

然后配置ZeroClaw：

```toml
default_provider = "sglang"
default_model = "meta-llama/Llama-3.1-8B-Instruct"
default_temperature = 0.7
```

快速验证：

```bash
zeroclaw models refresh --provider sglang
zeroclaw agent -m "hello"
```

此流程不需要导出`ZEROCLAW_API_KEY=dummy`。

## vLLM服务器

ZeroClaw包含[vLLM](https://docs.vllm.ai/)的一流本地提供者：

- 提供者ID：`vllm`
- 默认端点：`http://localhost:8000/v1`
- API密钥是可选的，除非服务器需要认证

启动本地服务器（示例）：

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct
```

然后配置ZeroClaw：

```toml
default_provider = "vllm"
default_model = "meta-llama/Llama-3.1-8B-Instruct"
default_temperature = 0.7
```

快速验证：

```bash
zeroclaw models refresh --provider vllm
zeroclaw agent -m "hello"
```

此流程不需要导出`ZEROCLAW_API_KEY=dummy`。

## 测试配置

验证您的自定义端点：

```bash
# 交互模式
zeroclaw agent

# 单消息测试
zeroclaw agent -m "test message"
```

## 故障排除

### 认证错误

- 验证API密钥是否正确
- 检查端点URL格式（必须包含`http://`或`https://`）
- 确保端点可从您的网络访问

### 找不到模型

- 确认模型名称与提供者的可用模型匹配
- 查看提供者文档以获取确切的模型标识符
- 确保端点和模型系列匹配。某些自定义网关只暴露模型的子集。
- 从您配置的相同端点和密钥验证可用模型：

```bash
curl -sS https://your-api.com/models \
  -H "Authorization: Bearer $API_KEY"
```

- 如果网关未实现`/models`，发送最小聊天请求并检查提供者返回的模型错误文本。

### 连接问题

- 测试端点可达性：`curl -I https://your-api.com`
- 验证防火墙/代理设置
- 检查提供者状态页面

## 示例

### 本地LLM服务器（通用自定义端点）

```toml
default_provider = "custom:http://localhost:8080/v1"
api_key = "your-api-key-if-required"
default_model = "local-model"
```

### 企业代理

```toml
default_provider = "anthropic-custom:https://llm-proxy.corp.example.com"
api_key = "internal-token"
```

### 云提供者网关

```toml
default_provider = "custom:https://gateway.cloud-provider.com/v1"
api_key = "gateway-api-key"
default_model = "gpt-4"
```