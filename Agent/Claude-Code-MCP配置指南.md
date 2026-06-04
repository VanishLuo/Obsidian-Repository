---
tags: [工具配置, MCP, Claude-Code]
created: 2026-06-04
---

# Claude Code MCP 服务器配置指南

## 常用命令

### 添加 stdio 服务器

```bash
# 格式
claude mcp add <name> -- <command> [args]

# 示例：Playwright
claude mcp add playwright -- npx -y @playwright/mcp@latest

# 示例：yawlabs fetch
claude mcp add yawlabs -- npx -y @yawlabs/fetch-mcp
```

### 指定作用域

```bash
# 项目级（默认）
claude mcp add <name> -- <command>

# 用户级（全局）
claude mcp add <name> -s user -- <command>
```

## 安装 MCP 服务器

```bash
# 安装 playwright
npm install -g @playwright/mcp@latest

# 安装 yawlabs fetch
npm install -g @yawlabs/fetch-mcp
```

## 常用 MCP 服务器

- **playwright** (`@playwright/mcp@latest`) - 浏览器自动化
- **yawlabs-fetch** (`@yawlabs/fetch-mcp`) - HTTP 获取工具

## 配置文件位置

- **用户级**：`~/.claude.json`
- **项目级**：`<project>/.mcp.json`

> Windows 上 `~` = `%USERPROFILE%`（如 `C:\Users\YourName\.claude.json`）

## 管理命令

```bash
claude mcp list                    # 列出所有服务器
claude mcp get <name>              # 查看服务器详情
claude mcp remove <name>           # 删除服务器
claude mcp reset-project-choices   # 重置项目级批准

# 在 Claude Code 会话中
/mcp                               # 打开交互式 MCP 面板
```

## 官方文档

- [MCP 快速入门](https://code.claude.com/docs/en/mcp-quickstart.md)
- [MCP 参考文档](https://code.claude.com/docs/en/mcp.md)
- [Anthropic Directory (MCP 服务器市场)](https://claude.ai/directory)

## 备注

- MCP 服务器名称 `workspace` 是保留的，无法使用
- 工具搜索默认启用（延迟加载），可通过 `ENABLE_TOOL_SEARCH` 环境变量配置
- 项目级服务器首次使用需要批准提示（安全措施）
- 作用域 `-s user` 表示用户级全局配置
