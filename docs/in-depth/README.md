# Claude Code Haha 文档索引

## 快速开始

- [安装指南](../README.md) - 项目安装和环境配置

## 深度文档

### 执行流程与架构

| 文档 | 描述 |
|------|------|
| [01-execution-flow](01-execution-flow.md) | 完整的程序执行流程，从入口脚本到 REPL 启动 |
| [02-ink-tui-engine](02-ink-tui-engine.md) | Ink TUI 渲染引擎的原理和使用 |
| [03-tools-system](03-tools-system.md) | 工具系统的注册、执行和权限管理 |
| [04-multi-agent](04-multi-agent.md) | 多 Agent 系统的架构和通信机制 |
| [05-mcp-service](05-mcp-service.md) | MCP (Model Context Protocol) 服务详解 |
| [06-state-management](06-state-management.md) | 状态管理、数据流和持久化 |
| [07-commands-and-skills](07-commands-and-skills.md) | 斜杠命令和 Skills 系统 |
| [08-api-and-query](08-api-and-query.md) | API 层和查询执行流程 |

## 学习路径

### 初级路线（了解基本运行原理）

1. [执行流程](./01-execution-flow.md) - 了解程序如何启动
2. [TUI 引擎](./02-ink-tui-engine.md) - 了解 UI 如何渲染
3. [工具系统](./03-tools-system.md) - 了解 Agent 如何与外界交互

### 高级路线（深入理解扩展机制）

4. [Agent 系统](./04-multi-agent.md) - 了解多 Agent 协作
5. [MCP 服务](./05-mcp-service.md) - 了解外部扩展协议
6. [状态管理](./06-state-management.md) - 了解数据流和持久化

### 专家路线（掌握自定义扩展）

7. [命令和 Skills](./07-commands-and-skills.md) - 扩展命令和 Skill
8. [API 层](./08-api-and-query.md) - 理解 API 调用和优化
