# OpenCode 工具、代理与 MCP 服务器

---

## 工具

> 文档来源：https://opencode.ai/docs/zh-cn/tools/

工具允许 LLM 在代码库中执行操作。默认情况下，所有工具都是**启用**的，无需权限即可运行。

### 内置工具列表

| 工具 | 说明 | 权限控制字段 |
|------|------|------------|
| `bash` | 执行 shell 命令（npm install、git status 等） | `bash` |
| `edit` | 通过精确字符串替换修改现有文件 | `edit` |
| `write` | 创建新文件或覆盖现有文件 | `edit`（同 edit） |
| `read` | 读取文件内容，支持指定行范围 | `read` |
| `grep` | 使用正则表达式搜索文件内容 | `grep` |
| `glob` | 通过 glob 模式匹配查找文件 | `glob` |
| `lsp` | 与 LSP 服务器交互（实验性，需设置 `OPENCODE_EXPERIMENTAL_LSP_TOOL=true`） | `lsp` |
| `patch` | 对文件应用补丁 | `edit`（同 edit） |
| `skill` | 加载 SKILL.md 文件并返回内容 | `skill` |
| `todowrite` | 管理待办事项列表（默认对子代理禁用） | `todowrite` |
| `webfetch` | 获取网页内容 | `webfetch` |
| `websearch` | 网络搜索（需要 OpenCode 提供商或 `OPENCODE_ENABLE_EXA=1`） | `websearch` |
| `question` | 在执行过程中向用户提问 | `question` |

> **提示**：`websearch` 用于发现信息，`webfetch` 用于从特定 URL 获取内容。

### 工具权限配置

```json
{
  "permission": {
    "edit": "deny",
    "bash": "ask",
    "webfetch": "allow"
  }
}
```

权限值：`"allow"`（允许）、`"ask"`（需审批）、`"deny"`（禁用）

使用通配符控制 MCP 工具：
```json
{
  "permission": {
    "mymcp_*": "ask"
  }
}
```

### 忽略模式

`grep` 和 `glob` 底层使用 ripgrep，默认遵循 `.gitignore`。要包含被忽略的文件，在项目根目录创建 `.ignore` 文件：
```
!node_modules/
!dist/
!build/
```

---

## 代理

> 文档来源：https://opencode.ai/docs/zh-cn/agents/

### 代理类型

**主代理（Primary）**：直接交互的主要助手，用 `Tab` 键切换。
**子代理（Subagent）**：由主代理调用或通过 `@` 提及手动调用的专业助手。

### 内置代理

| 代理 | 类型 | 说明 |
|------|------|------|
| **Build** | 主代理 | 默认代理，启用所有工具，用于完整开发工作 |
| **Plan** | 主代理 | 受限代理，`edit` 和 `bash` 默认需要审批，用于分析规划不修改代码 |
| **General** | 子代理 | 通用代理，完整工具访问（todo 除外），可并行运行多个工作单元 |
| **Explore** | 子代理 | 快速只读代理，无法修改文件，用于探索代码库 |
| **Scout** | 子代理 | 只读代理，用于外部文档和依赖研究，可克隆仓库到缓存 |
| **Compaction** | 系统 | 隐藏，自动压缩长上下文 |
| **Title** | 系统 | 隐藏，自动生成会话标题 |
| **Summary** | 系统 | 隐藏，自动创建会话摘要 |

### 使用代理

```
# 切换主代理
Tab 键

# 调用子代理
@general help me search for this function
@explore find all files using authentication
```

### 配置代理（JSON 方式）

```jsonc
{
  "agent": {
    "code-reviewer": {
      "description": "Reviews code for best practices",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-5",
      "prompt": "You are a code reviewer. Focus on security, performance, and maintainability.",
      "temperature": 0.1,
      "tools": {
        "write": false,
        "edit": false
      },
      "permission": {
        "bash": "deny"
      }
    }
  }
}
```

### 配置代理（Markdown 方式）

放在 `~/.config/opencode/agents/` 或 `.opencode/agents/` 目录：

```markdown
---
description: Reviews code for quality and best practices
mode: subagent
model: anthropic/claude-sonnet-4-5
temperature: 0.1
tools:
  write: false
  edit: false
  bash: false
---

You are in code review mode. Focus on:
- Code quality and best practices
- Potential bugs and edge cases
```

### 代理配置选项

| 选项 | 说明 |
|------|------|
| `description` | 代理功能描述（必填） |
| `mode` | `primary`、`subagent` 或 `all`（默认 `all`） |
| `model` | 覆盖使用的模型 |
| `prompt` | 自定义系统提示词或文件引用 `{file:./prompts/xxx.txt}` |
| `temperature` | 响应随机性（0.0-1.0，0=确定性，1=创造性） |
| `steps` | 最大代理迭代次数 |
| `tools` | 启用/禁用特定工具 |
| `permission` | 工具权限配置 |
| `color` | UI 颜色（十六进制或主题色） |
| `hidden` | 从 `@` 自动补全菜单隐藏（仅子代理） |
| `disable` | 禁用代理 |
| `top_p` | 响应多样性控制（0.0-1.0） |

### 创建代理

```bash
opencode agent create
```

---

## MCP 服务器

> 文档来源：https://opencode.ai/docs/zh-cn/mcp-servers/

MCP（Model Context Protocol）允许为 OpenCode 添加外部工具。

> **注意**：MCP 服务器会占用上下文空间，请谨慎选择启用哪些服务器。

### 本地 MCP 服务器

```jsonc
{
  "mcp": {
    "my-local-mcp": {
      "type": "local",
      "command": ["npx", "-y", "my-mcp-command"],
      "enabled": true,
      "environment": {
        "MY_ENV_VAR": "value"
      }
    }
  }
}
```

### 远程 MCP 服务器

```json
{
  "mcp": {
    "my-remote-mcp": {
      "type": "remote",
      "url": "https://my-mcp-server.com",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer {env:MY_API_KEY}"
      }
    }
  }
}
```

### OAuth 认证

OpenCode 自动处理 OAuth 认证：
```bash
opencode mcp auth my-oauth-server   # 手动触发认证
opencode mcp list                   # 查看认证状态
opencode mcp logout my-oauth-server # 删除凭据
```

Token 存储在 `~/.local/share/opencode/mcp-auth.json`。

### 常用 MCP 示例

**Sentry**：
```json
{
  "mcp": {
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "oauth": {}
    }
  }
}
```

**Context7（文档搜索）**：
```json
{
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp"
    }
  }
}
```

**Grep by Vercel（GitHub 代码搜索）**：
```json
{
  "mcp": {
    "gh_grep": {
      "type": "remote",
      "url": "https://mcp.grep.app"
    }
  }
}
```

### 按代理管理 MCP

```json
{
  "tools": { "my-mcp*": false },
  "agent": {
    "my-agent": {
      "tools": { "my-mcp*": true }
    }
  }
}
```

### MCP 工具 Glob 模式

MCP 工具以服务器名称为前缀，例如服务器 `sentry` 的工具为 `sentry_*`：
```json
{
  "tools": { "sentry_*": false }
}
```
