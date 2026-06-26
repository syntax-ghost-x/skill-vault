# OpenCode TUI 与 CLI

---

## TUI（终端用户界面）

> 文档来源：https://opencode.ai/docs/zh-cn/tui/

### 启动

```bash
opencode                    # 当前目录
opencode /path/to/project   # 指定目录
```

### 文件引用

使用 `@` 在消息中引用文件（模糊搜索当前工作目录）：
```
How is auth handled in @packages/functions/src/api/index.ts?
```

### Bash 命令

以 `!` 开头的消息会作为 shell 命令执行：
```
!ls -la
```

### TUI 斜杠命令

| 命令 | 快捷键 | 说明 |
|------|--------|------|
| `/connect` | - | 添加提供商 API 密钥 |
| `/compact` 或 `/summarize` | `ctrl+x c` | 压缩当前会话 |
| `/details` | `ctrl+x d` | 切换工具执行详情显示 |
| `/editor` | `ctrl+x e` | 打开外部编辑器编写消息 |
| `/exit` 或 `/quit` 或 `/q` | `ctrl+x q` | 退出 OpenCode |
| `/export` | `ctrl+x x` | 导出对话为 Markdown |
| `/help` | `ctrl+x h` | 显示帮助对话框 |
| `/init` | `ctrl+x i` | 创建/更新 AGENTS.md |
| `/models` | `ctrl+x m` | 列出可用模型 |
| `/new` 或 `/clear` | `ctrl+x n` | 开始新会话 |
| `/redo` | `ctrl+x r` | 重做撤销的消息（需要 Git 仓库） |
| `/sessions` 或 `/resume` 或 `/continue` | `ctrl+x l` | 列出会话并切换 |
| `/share` | `ctrl+x s` | 分享当前会话 |
| `/themes` | `ctrl+x t` | 列出可用主题 |
| `/thinking` | - | 切换思考/推理块的可见性 |
| `/undo` | `ctrl+x u` | 撤销最后一条消息（需要 Git 仓库） |
| `/unshare` | - | 取消分享当前会话 |

> **注意**：`/undo` 和 `/redo` 使用 Git 管理文件更改，项目需要是 Git 仓库。

### 编辑器设置

`/editor` 和 `/export` 命令使用 `EDITOR` 环境变量：
```bash
export EDITOR=vim
export EDITOR="code --wait"  # VS Code 需要 --wait 标志
```

### TUI 配置选项

```json
{
  "tui": {
    "scroll_speed": 3,
    "scroll_acceleration": { "enabled": true }
  }
}
```

---

## CLI（命令行界面）

> 文档来源：https://opencode.ai/docs/zh-cn/cli/

### 基本用法

```bash
opencode                                    # 启动 TUI
opencode run "Explain how closures work"    # 非交互式运行
```

### TUI 标志

| 标志 | 简写 | 说明 |
|------|------|------|
| `--continue` | `-c` | 继续上一个会话 |
| `--session` | `-s` | 指定要继续的会话 ID |
| `--fork` | - | 继续时分叉会话 |
| `--prompt` | - | 要使用的提示词 |
| `--model` | `-m` | 指定模型（格式：provider/model） |
| `--agent` | - | 指定要使用的代理 |
| `--port` | - | 监听端口 |
| `--hostname` | - | 监听主机名 |

### CLI 子命令

#### `opencode agent`
管理代理：
```bash
opencode agent create   # 创建新代理
opencode agent list     # 列出所有代理
```

#### `opencode attach`
连接到已运行的 OpenCode 后端：
```bash
opencode web --port 4096 --hostname 0.0.0.0   # 启动后端
opencode attach http://10.20.30.40:4096        # 连接 TUI
```

#### `opencode auth`
管理提供商凭据：
```bash
opencode auth login     # 添加提供商 API 密钥
opencode auth list      # 列出已认证的提供商
opencode auth logout    # 清除提供商凭据
```
凭据存储在 `~/.local/share/opencode/auth.json`。

#### `opencode github`
管理 GitHub 代理：
```bash
opencode github install   # 在仓库中安装 GitHub 代理
opencode github run       # 运行 GitHub 代理（通常在 CI 中使用）
```

#### `opencode mcp`
管理 MCP 服务器：
```bash
opencode mcp add          # 添加 MCP 服务器
opencode mcp list         # 列出已配置的 MCP 服务器
opencode mcp auth [name]  # 对支持 OAuth 的 MCP 服务器认证
opencode mcp logout [name] # 移除 MCP 服务器的 OAuth 凭据
```

#### `opencode models`
列出可用模型：
```bash
opencode models
```

#### `opencode run`
非交互式运行任务：
```bash
opencode run "Your task description"
```

#### `opencode serve`
启动 OpenCode 服务器（供 Web/API 使用）：
```bash
opencode serve
```

#### `opencode session`
管理会话：
```bash
opencode session list
```

#### `opencode web`
启动 Web 界面：
```bash
opencode web --port 4096
```

#### `opencode upgrade`
升级 OpenCode：
```bash
opencode upgrade
```

### 全局标志

| 标志 | 说明 |
|------|------|
| `--help` | 显示帮助信息 |
| `--version` | 显示版本信息 |

### 环境变量

| 变量 | 说明 |
|------|------|
| `OPENCODE_CONFIG` | 自定义配置文件路径 |
| `OPENCODE_CONFIG_DIR` | 自定义配置目录 |
| `OPENCODE_CONFIG_CONTENT` | 内联配置内容（JSON） |
| `OPENCODE_SERVER_PASSWORD` | 服务器基本认证密码 |
| `OPENCODE_SERVER_USERNAME` | 服务器基本认证用户名 |
