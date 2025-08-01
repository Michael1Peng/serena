# Serena SSE (Server-Sent Events) 使用指南

## 概述

Serena 支持通过 SSE (Server-Sent Events) 模式运行 MCP 服务器，相比传统的 stdio 模式，SSE 模式提供了更好的控制和调试体验。

## SSE 模式 vs Stdio 模式对比

| 特性         | SSE 模式               | Stdio 模式       |
| ------------ | ---------------------- | ---------------- |
| 服务器控制   | 手动启动/停止          | 客户端自动管理   |
| 调试能力     | 直接查看服务器日志     | 通过客户端日志   |
| 多客户端支持 | 支持多个客户端同时连接 | 一对一绑定       |
| 进程管理     | 独立进程，易于清理     | 可能产生僵尸进程 |
| 网络通信     | HTTP-based             | 标准输入输出流   |

## 快速开始

### 1. 启动 SSE 服务器

```bash
# 基本启动命令
uv run serena start-mcp-server --transport sse --port 9121

# 指定项目路径启动
uv run --directory /Users/Shared/repositories/oraios/serena serena start-mcp-server --transport sse --port 9121

# 带项目和上下文的启动
uv run serena start-mcp-server \
  --transport sse \
  --port 9121 \
  --context ide-assistant \
  --project $(pwd)

# 指定配置文件启动
uv run serena start-mcp-server \
  --transport sse \
  --port 9121 \
  --config ~/.serena/serena_config.yml
```

成功启动后，你应该看到类似输出：

```
INFO: Serena MCP Server started in SSE mode on http://localhost:9121/sse
INFO: Dashboard available at http://localhost:24282/dashboard/index.html
```

### 2. 连接地址

- **MCP 连接地址**: `http://localhost:9121/sse`
- **Web 仪表盘**: `http://localhost:24282/dashboard/index.html`

## 客户端配置

### Claude Code

```bash
# 添加 SSE 模式的 MCP 服务器
claude mcp add --transport sse sse-server
claude mcp add serena-sse --transport sse http://localhost:9121/sse

# 添加 SSE 模式的 MCP 服务器到当前项目
claude mcp add serena-sse --transport sse http://localhost:9121/sse -s project
```

### Claude Desktop

编辑 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "serena-sse": {
      "transport": "sse",
      "url": "http://localhost:9121/sse"
    }
  }
}
```

### 其他 MCP 客户端 (Cline, Roo-Code, Cursor 等)

在客户端的 MCP 配置中添加：

```json
{
  "name": "serena-sse",
  "transport": "sse",
  "url": "http://localhost:9121/sse"
}
```

## 高级配置

### 命令行参数详解

```bash
uv run serena start-mcp-server --help
```

常用参数：

- `--transport sse`: 指定使用 SSE 传输模式
- `--port <port>`: 指定端口号（默认随机端口）
- `--host <host>`: 指定绑定地址（默认 localhost）
- `--project <path>`: 指定项目路径
- `--context <context>`: 指定上下文（desktop-app, ide-assistant, agent）
- `--mode <mode>`: 指定模式（planning, editing, interactive 等）

### 多项目配置示例

为不同项目启动不同的 SSE 服务器：

```bash
# 项目 A - 端口 9121
uv run serena start-mcp-server \
  --transport sse --port 9121 \
  --project /path/to/project-a \
  --context ide-assistant

# 项目 B - 端口 9122
uv run serena start-mcp-server \
  --transport sse --port 9122 \
  --project /path/to/project-b \
  --context desktop-app
```

## 架构图

```ascii
┌─────────────────┐    HTTP/SSE     ┌─────────────────┐
│   MCP Client    │ ←──────────────→ │  Serena Server  │
│ (Claude Code,   │    Port 9121    │   (SSE Mode)    │
│  Claude Desktop,│                 │                 │
│  Cline, etc.)   │                 │                 │
└─────────────────┘                 └─────────────────┘
                                             │
                                             │ 管理
                                             ▼
                                    ┌─────────────────┐
                                    │ Language Server │
                                    │   & Project     │
                                    │     Files       │
                                    └─────────────────┘

数据流：
1. Client → HTTP Request → Serena SSE Server
2. Serena → Language Server → Project Files
3. Results ← SSE Response ← Serena ← Language Server
```

## 调试和监控

### 服务器日志

SSE 模式下，服务器日志直接输出到终端，便于实时调试：

```bash
# 启动时可以看到详细日志
uv run serena start-mcp-server --transport sse --port 9121
```

### Web 仪表盘

访问 `http://localhost:24282/dashboard/index.html` 查看：

- 工具使用统计
- 实时日志
- 服务器状态
- 关闭服务器功能

### 健康检查

```bash
# 检查服务器是否正常运行
curl http://localhost:9121/health

# 检查 SSE 端点
curl -H "Accept: text/event-stream" http://localhost:9121/sse
```

## 常见问题

### Q: SSE 服务器无法启动？

A: 检查端口是否被占用：

```bash
lsof -i :9121
```

### Q: 客户端连接失败？

A: 确认：

1. 服务器正在运行
2. 端口号正确
3. 防火墙设置
4. URL 格式: `http://localhost:9121/sse`

### Q: 如何优雅关闭服务器？

A:

- 终端: `Ctrl+C`
- Web 仪表盘: 点击关闭按钮
- 命令: `pkill -f "serena start-mcp-server"`

### Q: 支持 HTTPS 吗？

A: 当前版本不直接支持 HTTPS，可通过反向代理（如 nginx）实现。

## 最佳实践

1. **开发环境**: 使用 SSE 模式便于调试
2. **生产环境**: 可考虑使用进程管理器（如 PM2）管理 SSE 服务器
3. **多项目**: 为每个项目使用不同端口的 SSE 服务器
4. **日志管理**: 将 SSE 服务器输出重定向到日志文件
5. **监控**: 定期检查服务器健康状态

## 示例脚本

### 启动脚本 (start-serena-sse.sh)

```bash
#!/bin/bash
PROJECT_PATH=${1:-$(pwd)}
PORT=${2:-9121}
CONTEXT=${3:-ide-assistant}

echo "Starting Serena SSE server..."
echo "Project: $PROJECT_PATH"
echo "Port: $PORT"
echo "Context: $CONTEXT"

uv run serena start-mcp-server \
  --transport sse \
  --port $PORT \
  --project "$PROJECT_PATH" \
  --context $CONTEXT
```

使用方法：

```bash
chmod +x start-serena-sse.sh
./start-serena-sse.sh /path/to/project 9121 ide-assistant
```

### 批量启动脚本

```bash
#!/bin/bash
# start-multiple-serena.sh

projects=(
  "/path/to/project-a:9121:ide-assistant"
  "/path/to/project-b:9122:desktop-app"
  "/path/to/project-c:9123:agent"
)

for project in "${projects[@]}"; do
  IFS=':' read -r path port context <<< "$project"
  echo "Starting Serena for $path on port $port"

  uv run serena start-mcp-server \
    --transport sse \
    --port $port \
    --project "$path" \
    --context $context &
done

wait
```

## 总结

SSE 模式为 Serena 提供了更灵活的部署和调试选项，特别适合：

- 开发和调试场景
- 需要精确控制服务器生命周期的情况
- 多项目并行开发
- 与多个客户端同时协作

选择 SSE 还是 stdio 模式取决于你的具体使用场景和偏好。
