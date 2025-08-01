# Serena Stdio 使用指南

## 概述

Serena 默认支持通过 stdio (标准输入输出) 模式运行 MCP 服务器，这是最简单和常用的使用方式。客户端会自动启动和管理 MCP 服务器进程。

## SSE 模式 vs Stdio 模式对比

| 特性         | SSE 模式               | Stdio 模式       |
| ------------ | ---------------------- | ---------------- |
| 服务器控制   | 手动启动/停止          | 客户端自动管理   |
| 调试能力     | 直接查看服务器日志     | 通过客户端日志   |
| 多客户端支持 | 支持多个客户端同时连接 | 一对一绑定       |
| 进程管理     | 独立进程，易于清理     | 可能产生僵尸进程 |
| 网络通信     | HTTP-based             | 标准输入输出流   |

## 快速开始

### 1. 基本配置

Stdio 模式不需要手动启动服务器，客户端会自动启动 MCP 服务器进程。

### 2. 测试安装

```bash
# 测试 Serena MCP 服务器是否可以正常启动
uv run serena-mcp-server --help

# 验证工具可用性
uv run serena-mcp-server --test
```

## 客户端配置

### Claude Code

```bash
# 添加 stdio 模式的 MCP 服务器（全局配置）
claude mcp add serena-stdio uv run serena-mcp-server

# 添加带参数的 stdio 模式服务器
claude mcp add serena-stdio uv run serena-mcp-server --context ide-assistant

# 添加到当前项目
claude mcp add serena-stdio -s project uv run serena-mcp-server --context ide-assistant 

# 添加带项目路径的配置
claude mcp add serena-stdio "uv run --directory /path/to/serena serena-mcp-server --context ide-assistant"
```

### Claude Desktop

编辑 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "serena-stdio": {
      "command": "uv",
      "args": ["run", "serena-mcp-server"],
      "cwd": "/path/to/serena"
    }
  }
}
```

带参数的配置：

```json
{
  "mcpServers": {
    "serena-stdio": {
      "command": "uv",
      "args": [
        "run", 
        "serena-mcp-server", 
        "--context", "ide-assistant",
        "--project", "/path/to/your/project"
      ],
      "cwd": "/path/to/serena"
    }
  }
}
```

### 其他 MCP 客户端 (Cline, Roo-Code, Cursor 等)

在客户端的 MCP 配置中添加：

```json
{
  "name": "serena-stdio",
  "command": "uv",
  "args": ["run", "serena-mcp-server"],
  "cwd": "/path/to/serena"
}
```

或者带参数的配置：

```json
{
  "name": "serena-stdio",
  "command": "uv",
  "args": [
    "run", 
    "serena-mcp-server",
    "--context", "ide-assistant",
    "--project", "/path/to/your/project"
  ],
  "cwd": "/path/to/serena"
}
```

## 高级配置

### 命令行参数详解

```bash
uv run serena-mcp-server --help
```

常用参数：

- `--project <path>`: 指定项目路径
- `--context <context>`: 指定上下文（desktop-app, ide-assistant, agent）
- `--mode <mode>`: 指定模式（planning, editing, interactive 等）
- `--config <path>`: 指定配置文件路径

### 项目特定配置

为特定项目创建配置：

```bash
# 在项目根目录创建 .serena/project.yml
mkdir -p .serena
cat > .serena/project.yml << EOF
name: "my-project"
contexts: ["ide-assistant"]
modes: ["editing", "interactive"]
language_servers:
  python:
    enabled: true
  typescript:
    enabled: true
EOF
```

### 用户全局配置

创建用户配置文件 `~/.serena/serena_config.yml`：

```yaml
default_context: "ide-assistant"
default_modes: ["editing", "interactive"]

projects:
  my-project:
    path: "/path/to/my-project"
    context: "desktop-app"
    modes: ["planning", "editing"]

language_servers:
  python:
    enabled: true
    executable: "pylsp"
  typescript:
    enabled: true
  java:
    enabled: false
```

## 架构图

```ascii
┌─────────────────┐    stdin/stdout   ┌─────────────────┐
│   MCP Client    │ ←──────────────→  │  Serena Server  │
│ (Claude Code,   │   (JSON-RPC)      │  (Stdio Mode)   │
│  Claude Desktop,│                   │                 │
│  Cline, etc.)   │                   │                 │
└─────────────────┘                   └─────────────────┘
        │                                      │
        │ 自动启动/停止                          │ 管理
        │                                      ▼
        │                             ┌─────────────────┐
        └─────────────────────────────│ Language Server │
                                      │   & Project     │
                                      │     Files       │
                                      └─────────────────┘

数据流：
1. Client 启动 → 自动启动 Serena Process
2. Client → JSON-RPC over stdin/stdout → Serena
3. Serena → Language Server → Project Files
4. Results ← JSON-RPC ← Serena ← Language Server
5. Client 退出 → 自动终止 Serena Process
```

## 调试和监控

### 客户端日志

大多数 MCP 客户端会记录与 MCP 服务器的通信日志：

**Claude Code:**
```bash
# 查看详细日志
claude --debug

# 查看 MCP 服务器状态
claude mcp list
```

**Claude Desktop:**
- macOS: `~/Library/Logs/Claude/mcp.log`
- Windows: `%APPDATA%\Claude\logs\mcp.log`
- Linux: `~/.local/share/Claude/logs/mcp.log`

### 服务器调试

在开发模式下可以手动启动服务器进行调试：

```bash
# 手动启动服务器进行调试
uv run serena-mcp-server --project $(pwd) --context ide-assistant

# 然后在另一个终端通过 JSON-RPC 与其交互
echo '{"jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {"protocolVersion": "2024-11-05", "capabilities": {}}}' | uv run serena-mcp-server
```

### 日志输出配置

设置环境变量控制日志级别：

```bash
# 详细日志
export SERENA_LOG_LEVEL=DEBUG

# 仅错误日志
export SERENA_LOG_LEVEL=ERROR

# 启动服务器
uv run serena-mcp-server
```

## 常见问题

### Q: MCP 服务器启动失败？

A: 检查安装和权限：

```bash
# 检查 uv 是否正确安装
uv --version

# 检查 Serena 是否正确安装
uv run serena-mcp-server --version

# 检查权限
ls -la $(which uv)
```

### Q: 客户端连接超时？

A: 检查：

1. 命令路径是否正确
2. 工作目录是否存在
3. 依赖是否完整安装：`uv sync`
4. 系统资源是否充足

### Q: 如何调试通信问题？

A: 

1. 使用客户端的调试模式
2. 检查进程是否正确启动：`ps aux | grep serena`
3. 手动测试服务器：`uv run serena-mcp-server --test`

### Q: 性能问题如何优化？

A:

```bash
# 预索引项目加速启动
uv run index-project

# 使用项目配置减少启动时间
cat > .serena/project.yml << EOF
language_servers:
  # 只启用需要的语言服务器
  python:
    enabled: true
  java:
    enabled: false
EOF
```

### Q: 多项目如何配置？

A: 为每个项目创建独立的 MCP 服务器配置：

```json
{
  "mcpServers": {
    "serena-project-a": {
      "command": "uv",
      "args": ["run", "serena-mcp-server", "--project", "/path/to/project-a"],
      "cwd": "/path/to/serena"
    },
    "serena-project-b": {
      "command": "uv",
      "args": ["run", "serena-mcp-server", "--project", "/path/to/project-b"],
      "cwd": "/path/to/serena"
    }
  }
}
```

## 最佳实践

1. **简单项目**: 使用 stdio 模式，配置简单，客户端自动管理
2. **开发环境**: 使用项目特定的 `.serena/project.yml` 配置
3. **多项目**: 创建多个 MCP 服务器配置，每个绑定特定项目
4. **性能优化**: 预索引项目，只启用必要的语言服务器
5. **调试**: 使用客户端调试模式，检查进程和日志

## 示例配置

### 单项目简单配置

**Claude Desktop (`claude_desktop_config.json`):**

```json
{
  "mcpServers": {
    "serena": {
      "command": "uv",
      "args": ["run", "serena-mcp-server"],
      "cwd": "/Users/Shared/repositories/oraios/serena"
    }
  }
}
```

### 多项目完整配置

**Claude Desktop (`claude_desktop_config.json`):**

```json
{
  "mcpServers": {
    "serena-main": {
      "command": "uv",
      "args": [
        "run", 
        "serena-mcp-server",
        "--context", "ide-assistant",
        "--mode", "editing",
        "--mode", "interactive"
      ],
      "cwd": "/Users/Shared/repositories/oraios/serena"
    },
    "serena-web-project": {
      "command": "uv",
      "args": [
        "run", 
        "serena-mcp-server",
        "--project", "/path/to/web-project",
        "--context", "desktop-app"
      ],
      "cwd": "/Users/Shared/repositories/oraios/serena"
    },
    "serena-python-project": {
      "command": "uv",
      "args": [
        "run", 
        "serena-mcp-server",
        "--project", "/path/to/python-project",
        "--context", "ide-assistant"
      ],
      "cwd": "/Users/Shared/repositories/oraios/serena"
    }
  }
}
```

### 项目配置文件示例

**`.serena/project.yml`:**

```yaml
name: "my-awesome-project"
description: "Full-stack web application"

contexts: ["ide-assistant"]
modes: ["editing", "interactive", "planning"]

language_servers:
  python:
    enabled: true
    executable: "pylsp"
    settings:
      pylsp:
        plugins:
          pycodestyle:
            enabled: false
  typescript:
    enabled: true
  go:
    enabled: false
  java:
    enabled: false

memory:
  auto_save: true
  max_entries: 100

tools:
  file_operations: true
  symbol_operations: true
  memory_operations: true
```

### 启动脚本示例

**`start-serena-stdio.sh`:**

```bash
#!/bin/bash

PROJECT_PATH=${1:-$(pwd)}
CONTEXT=${2:-ide-assistant}
CONFIG_FILE="$PROJECT_PATH/.serena/project.yml"

echo "Testing Serena MCP Server..."
if ! uv run serena-mcp-server --version > /dev/null 2>&1; then
    echo "Error: Serena MCP server not available"
    exit 1
fi

echo "Project: $PROJECT_PATH"
echo "Context: $CONTEXT"

if [ -f "$CONFIG_FILE" ]; then
    echo "Using project config: $CONFIG_FILE"
    ARGS="--config $CONFIG_FILE"
else
    echo "Using default config"
    ARGS="--project $PROJECT_PATH --context $CONTEXT"
fi

echo "Testing server startup..."
echo '{"jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {"protocolVersion": "2024-11-05", "capabilities": {}}}' | \
    uv run serena-mcp-server $ARGS

if [ $? -eq 0 ]; then
    echo "✅ Serena MCP server is ready"
    echo "Add this to your MCP client configuration:"
    echo "Command: uv"
    echo "Args: [\"run\", \"serena-mcp-server\"] + additional args"
    echo "CWD: $(pwd)"
else
    echo "❌ Server test failed"
    exit 1
fi
```

使用方法：

```bash
chmod +x start-serena-stdio.sh
./start-serena-stdio.sh /path/to/project ide-assistant
```

## 总结

Stdio 模式是 Serena 的默认和推荐使用方式，特别适合：

- 日常开发使用
- 客户端自动管理服务器生命周期的场景
- 简单的单用户工作流
- 与现有 IDE 和编辑器集成

相比 SSE 模式，stdio 模式配置更简单，但调试能力稍弱。选择哪种模式取决于你的具体需求和使用场景。
