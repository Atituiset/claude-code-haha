# Claude Code Haha 深度文档索引

## 快速开始

- [安装指南](../index.md) - 项目安装和环境配置

## 基础文档

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

## 深度文档（源码级分析）

| 文档 | 描述 |
|------|------|
| [09-agent-loop-deep-dive](09-agent-loop-deep-dive.md) | **Agent 主循环深度解析**：状态机、流式 API、工具编排、Token 预算、消息压缩 |
| [10-permission-system-deep-dive](10-permission-system-deep-dive.md) | **权限系统深度解析**：8/5 步管线、AI 分类器、4 路竞争、文件系统安全 |
| [11-provider-model-system](11-provider-model-system.md) | **Provider/模型系统深度解析**：多 Provider 路由、思考模式、缓存控制、成本追踪 |
| [12-mcp-skills-system](12-mcp-skills-system.md) | **MCP 与 Skills 系统深度解析**：4 种传输、OAuth、Elicitation、Skill 加载、插件 |
| [13-server-desktop-architecture](13-server-desktop-architecture.md) | **Server/Desktop 架构深度解析**：28+ 服务、Tauri 集成、WebSocket 协议、IM 适配器 |
| [14-cli-tui-startup](14-cli-tui-startup.md) | **CLI 启动与 TUI 深度解析**：13 步初始化、和弦按键、18 上下文 80+ 动作、命令系统 |
| [15-tools-system-deep-dive](15-tools-system-deep-dive.md) | **工具系统深度解析**：55+ 工具、并发编排、流式执行、6 个内置 Agent |
| [16-multi-agent-hooks](16-multi-agent-hooks.md) | **多 Agent 与 Hooks 深度解析**：Swarm 通信、Agent 生命周期、27 个 Hook 事件、插件 |

## 学习路径

### 初级路线（了解基本运行原理）

1. [执行流程](./01-execution-flow.md) - 了解程序如何启动
2. [TUI 引擎](./02-ink-tui-engine.md) - 了解 UI 如何渲染
3. [工具系统](./03-tools-system.md) - 了解 Agent 如何与外界交互

### 高级路线（深入理解扩展机制）

4. [Agent 系统](./04-multi-agent.md) - 了解多 Agent 协作
5. [MCP 服务](./05-mcp-service.md) - 了解外部扩展协议
6. [状态管理](./06-state-management.md) - 了解数据流和持久化

### 专家路线（源码级理解）

7. [Agent 主循环](./09-agent-loop-deep-dive.md) - **理解 Agent 运行的核心引擎**
8. [权限系统](./10-permission-system-deep-dive.md) - **理解 Agent 安全执行的关键**
9. [Provider/模型](./11-provider-model-system.md) - **理解多 LLM 接入的架构**
10. [MCP/Skills](./12-mcp-skills-system.md) - **理解扩展能力的核心**
11. [Server/Desktop](./13-server-desktop-architecture.md) - **理解服务端和桌面端架构**
12. [CLI/TUI](./14-cli-tui-startup.md) - **理解终端交互的底层机制**
13. [工具系统](./15-tools-system-deep-dive.md) - **理解 55+ 工具的实现**
14. [多 Agent/Hooks](./16-multi-agent-hooks.md) - **理解协作和事件系统**

### 自建 Agent 路线（从零实现）

1. [Agent 主循环](./09-agent-loop-deep-dive.md) → 最小 async generator 循环
2. [工具系统](./15-tools-system-deep-dive.md) → 工具定义和执行
3. [权限系统](./10-permission-system-deep-dive.md) → 安全决策管线
4. [Provider/模型](./11-provider-model-system.md) → 多 Provider 路由
5. [MCP/Skills](./12-mcp-skills-system.md) → 外部扩展
6. [Server/Desktop](./13-server-desktop-architecture.md) → 服务端架构
