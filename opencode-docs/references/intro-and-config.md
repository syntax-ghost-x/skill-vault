# OpenCode 简介与配置

> 文档来源：https://opencode.ai/docs/zh-cn/

---

## 简介

**OpenCode** 是一个开源的 AI 编码代理，提供终端界面（TUI）、桌面应用和 IDE 扩展等多种使用方式。

### 前提条件

- 一款现代终端模拟器（WezTerm、Alacritty、Ghostty、Kitty 等）
- 你想使用的 LLM 提供商的 API 密钥

### 安装

```bash
# 最简单的方式：安装脚本
curl -fsSL https://opencode.ai/install | bash

# 使用 Node.js
npm install -g opencode-ai
# 或 bun install -g opencode-ai
# 或 pnpm install -g opencode-ai

# macOS/Linux 使用 Homebrew
brew install anomalyco/tap/opencode

# Arch Linux
sudo pacman -S opencode

# Windows 使用 Chocolatey
choco install opencode

# Windows 使用 Scoop
scoop install opencode

# 使用 Docker
docker run -it --rm ghcr.io/anomalyco/opencode
```

### 配置提供商

```
/connect
```
在 TUI 中运行 `/connect` 命令，选择提供商，前往 opencode.ai/auth 获取 API 密钥。

### 初始化项目

```bash
cd /path/to/project
opencode
/init
```

`/init` 会分析项目并在根目录创建 `AGENTS.md` 文件，建议将其提交到 Git。

### 基本使用

- **提问**：直接输入问题，用 `@` 引用文件（模糊搜索）
- **计划模式**：按 `Tab` 切换到计划模式，不会修改文件，只给出建议
- **构建模式**：再次按 `Tab` 切换回构建模式，执行修改
- **撤销**：`/undo` 撤销最后一次修改（需要 Git 仓库）
- **重做**：`/redo` 重做撤销的修改
- **分享**：`/share` 生成对话分享链接

---

## 配置

> 文档来源：https://opencode.ai/docs/zh-cn/config/

OpenCode 使用 JSON/JSONC 格式的配置文件。

### 配置文件格式

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",
  "autoupdate": true,
  "server": {
    "port": 4096
  }
}
```

### 配置文件位置与优先级

配置文件**合并**而非替换，优先级从低到高：

1. **远程配置**（`.well-known/opencode`）- 组织默认值
2. **全局配置**（`~/.config/opencode/opencode.json`）- 用户偏好
3. **自定义配置**（`OPENCODE_CONFIG` 环境变量）
4. **项目配置**（项目根目录的 `opencode.json`）- 最高优先级
5. **`.opencode` 目录** - 代理、命令、插件
6. **内联配置**（`OPENCODE_CONFIG_CONTENT` 环境变量）

### 主要配置项

#### TUI 配置
```json
{
  "tui": {
    "scroll_speed": 3,
    "scroll_acceleration": { "enabled": true },
    "diff_style": "auto"
  }
}
```

#### 服务器配置
```json
{
  "server": {
    "port": 4096,
    "hostname": "0.0.0.0",
    "mdns": true,
    "cors": ["http://localhost:5173"]
  }
}
```

#### 模型配置
```json
{
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",
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

#### 工具配置（禁用某些工具）
```json
{
  "tools": {
    "write": false,
    "bash": false
  }
}
```

#### 权限配置
```json
{
  "permission": {
    "edit": "ask",
    "bash": "ask"
  }
}
```

#### 自动更新
```json
{
  "autoupdate": false
}
```
设为 `"notify"` 则只通知不自动更新。

#### 上下文压缩
```json
{
  "compaction": {
    "auto": true,
    "prune": true,
    "reserved": 10000
  }
}
```

#### 分享设置
```json
{
  "share": "manual"
}
```
可选值：`"manual"`（默认）、`"auto"`、`"disabled"`

#### 禁用/启用提供商
```json
{
  "disabled_providers": ["openai", "gemini"],
  "enabled_providers": ["anthropic", "openai"]
}
```
注意：`disabled_providers` 优先于 `enabled_providers`。

#### 环境变量引用
```json
{
  "model": "{env:OPENCODE_MODEL}"
}
```

### 自定义配置路径
```bash
export OPENCODE_CONFIG=/path/to/custom-config.json
export OPENCODE_CONFIG_DIR=/path/to/config-directory
```
