# OpenCode 提供商与模型

---

## 提供商

> 文档来源：https://opencode.ai/docs/zh-cn/providers/

OpenCode 使用 AI SDK 和 Models.dev，支持 **75+ LLM 提供商**，也支持本地模型。

### 添加提供商

1. 在 TUI 中运行 `/connect` 命令
2. 选择提供商并输入 API 密钥
3. 凭据存储在 `~/.local/share/opencode/auth.json`

### OpenCode Zen（推荐新用户）

OpenCode Zen 是由 OpenCode 团队测试验证的精选模型列表。
- 运行 `/connect`，选择 opencode，前往 opencode.ai/auth 获取密钥
- 运行 `/models` 查看推荐模型列表

### 主要提供商列表

| 提供商 | 说明 |
|--------|------|
| **Anthropic** | Claude 系列模型，支持 Pro/Max 订阅或 API 密钥 |
| **OpenAI** | GPT 系列模型 |
| **Amazon Bedrock** | AWS 托管模型，支持多种认证方式 |
| **Azure OpenAI** | Azure 托管的 OpenAI 模型 |
| **Google Vertex AI** | Google Cloud 托管模型 |
| **DeepSeek** | DeepSeek 系列模型 |
| **Groq** | 高速推理服务 |
| **Ollama** | 本地运行开源模型 |
| **LM Studio** | 本地模型管理工具 |
| **OpenRouter** | 多提供商聚合路由 |
| **GitHub Copilot** | GitHub 集成模型 |
| **GitLab Duo** | GitLab 集成模型 |
| **Cloudflare AI Gateway** | 统一端点访问多提供商 |
| **xAI** | Grok 系列模型 |

### Amazon Bedrock 配置

```json
{
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "us-east-1",
        "profile": "my-aws-profile",
        "endpoint": "https://bedrock-runtime.us-east-1.vpce-xxxxx.amazonaws.com"
      }
    }
  }
}
```

认证优先级：
1. Bearer Token（`AWS_BEARER_TOKEN_BEDROCK` 或 `/connect`）
2. AWS 凭证链（配置文件、访问密钥、IAM 角色等）

### 自定义提供商（OpenAI 兼容）

```json
{
  "provider": {
    "my-local-llm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My Local LLM",
      "options": {
        "baseURL": "http://127.0.0.1:11434/v1"
      },
      "models": {
        "llama3": {
          "name": "Llama 3"
        }
      }
    }
  }
}
```

### 自定义 Base URL

```json
{
  "provider": {
    "anthropic": {
      "options": {
        "baseURL": "https://api.anthropic.com/v1"
      }
    }
  }
}
```

---

## 模型

> 文档来源：https://opencode.ai/docs/zh-cn/models/

### 设置模型

```json
{
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5"
}
```

`small_model` 用于标题生成等轻量级任务。

### 模型格式

模型 ID 格式为 `provider/model-name`，例如：
- `anthropic/claude-sonnet-4-5`
- `openai/gpt-4o`
- `google/gemini-2.0-flash`

### 本地模型（Ollama）

```json
{
  "provider": {
    "ollama": {
      "options": {
        "baseURL": "http://localhost:11434/api"
      }
    }
  }
}
```

### 提供商特定选项

```json
{
  "provider": {
    "anthropic": {
      "options": {
        "timeout": 600000,
        "setCacheKey": true
      }
    }
  }
}
```

- `timeout`：请求超时（毫秒，默认 300000），设为 `false` 禁用
- `setCacheKey`：始终为该提供商设置缓存键
