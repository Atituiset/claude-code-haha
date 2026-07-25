# Claude Code Haha 源码深度分析

## 目录

1. [项目概述](#1-项目概述)
2. [执行流程详解](#2-执行流程详解)
   - [2.1 入口脚本 (bin/claude-haha)](#21-入口脚本-binclaude-haha)
   - [2.2 CLI 引导器 (src/entrypoints/cli.tsx)](#22-cli-引导器-srcentrypointsclitsx)
   - [2.3 主程序 (src/main.tsx)](#23-主程序-srcmaintsx)
   - [2.4 初始化 (src/setup.ts)](#24-初始化-srcsetupts)
   - [2.5 REPL 启动器 (src/replLauncher.tsx)](#25-repl-启动器-srcrepllaunchertsx)
3. [TUI 渲染引擎](#3-tui-渲染引擎)
4. [工具系统](#4-工具系统)
5. [Agent 系统](#5-agent-系统)
6. [MCP 服务](#6-mcp-服务)
7. [状态管理](#7-状态管理)

---

## 1. 项目概述

Claude Code Haha 是基于 2026 年 3 月从 Anthropic npm registry 泄露的 Claude Code 源码修复的**本地可运行版本**。

### 技术栈

| 类别 | 技术 |
|------|------|
| 运行时 | Bun |
| 语言 | TypeScript |
| 终端 UI | React + Ink |
| CLI 解析 | Commander.js |
| API | Anthropic SDK |
| 协议 | MCP, LSP |

### 项目结构

```
bin/claude-haha              # Shell 入口脚本
preload.ts                   # Bun preload（设置 MACRO 全局变量）
src/
├── entrypoints/
│   ├── cli.tsx              # CLI 引导器（处理特殊 flags）
│   └── init.ts              # 初始化逻辑
├── main.tsx                 # 主程序（Commander.js + React/Ink）
├── setup.ts                 # 启动初始化
├── replLauncher.tsx         # REPL 启动器
├── screens/
│   └── REPL.tsx             # 主交互界面
├── ink/                     # Ink 终端渲染引擎
├── components/              # UI 组件
├── tools/                   # Agent 工具
├── commands/                # 斜杠命令
├── skills/                  # Skill 系统
├── services/                # 服务层
│   ├── api/                 # API 客户端
│   ├── mcp/                 # MCP 协议
│   └── analytics/           # 分析/遥测
├── state/                   # 状态管理
└── utils/                   # 工具函数
```

---

## 2. 执行流程详解

### 2.1 入口脚本 (bin/claude-haha)

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT_DIR"

# 降级模式：简单 readline REPL，无 Ink TUI
if [[ "${CLAUDE_CODE_FORCE_RECOVERY_CLI:-0}" == "1" ]]; then
  exec bun --env-file=.env ./src/localRecoveryCli.ts "$@"
fi

# 默认：完整 CLI + Ink TUI
exec bun --env-file=.env ./src/entrypoints/cli.tsx "$@"
```

**关键逻辑：**
1. 切换到项目根目录
2. 检查 `CLAUDE_CODE_FORCE_RECOVERY_CLI` 环境变量
   - 为 `1` 时：启动降级 Recovery CLI（`localRecoveryCli.ts`）
   - 默认：启动完整 CLI（`cli.tsx`）

### 2.2 CLI 引导器 (src/entrypoints/cli.tsx)

`cli.tsx` 是**快速路径路由器**，在加载重型模块前检测特殊 flags。

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2)

  // 快速路径 1: --version / -v（零模块加载）
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`)
    return
  }

  // 各种特殊 flags 的快速路径...
  // --dump-system-prompt, --claude-in-chrome-mcp, --computer-use-mcp
  // --daemon-worker, remote-control, daemon, bg sessions, templates, etc.

  // 默认：加载完整 CLI
  const { main: cliMain } = await import('../main.js')
  await cliMain()
}
```

**检测的特殊 flags：**

| Flag | 功能 | 文件 |
|------|------|------|
| `--version` / `-v` | 输出版本号 | 直接返回 |
| `--dump-system-prompt` | 输出系统提示词 | constants/prompts.js |
| `--claude-in-chrome-mcp` | Chrome MCP 服务器 | utils/claudeInChrome/mcpServer.js |
| `--chrome-native-host` | Chrome 原生主机 | utils/claudeInChrome/chromeNativeHost.js |
| `--computer-use-mcp` | 计算机使用 MCP | utils/computerUse/mcpServer.js |
| `--daemon-worker=<kind>` | Daemon 工作进程 | daemon/workerRegistry.js |
| `remote-control/rc/remote/sync/bridge` | 网桥模式 | bridge/bridgeMain.js |
| `daemon` | 长驻守护进程 | daemon/main.js |
| `ps/logs/attach/kill` | 会话管理 | cli/bg.js |
| `new/list/reply` | 模板任务 | cli/handlers/templateJobs.js |
| `--worktree --tmux` | Tmux 工作树 | utils/worktree.js |

### 2.3 主程序 (src/main.tsx)

`main.tsx` 是核心初始化文件，负责：
1. 设置启动性能分析（`startupProfiler.js`）
2. 启动 MDM 原始读取（macOS 移动设备管理设置）
3. 启动 Keychain 预取（macOS 安全存储）
4. 初始化 Commander.js 命令行解析
5. 调用 `setup()` 进行初始化
6. 启动 REPL

**核心流程：**

```typescript
// 1. 启动性能分析
profileCheckpoint('main_tsx_entry')
startMdmRawRead()
startKeychainPrefetch()

// 2. 解析命令行参数（Commander.js）
const program = new Command()
program
  .name('claude')
  .option('-p, --print', '无头模式')
  .option('--model <model>', '指定模型')
  // ...

// 3. 调用 setup() 初始化
await setup(cwd, permissionMode, allowDangerouslySkipPermissions, ...)

// 4. 启动 REPL
const repl = await launchRepl(options)
```

### 2.4 初始化 (src/setup.ts)

`setup()` 函数执行大量初始化工作：

```typescript
export async function setup(
  cwd: string,
  permissionMode: PermissionMode,
  allowDangerouslySkipPermissions: boolean,
  worktreeEnabled: boolean,
  worktreeName: string | undefined,
  tmuxEnabled: boolean,
  customSessionId?: string | null,
  worktreePRNumber?: number,
  messagingSocketPath?: string,
): Promise<void>
```

**主要步骤：**

1. **Node.js 版本检查**（需 >= 18）
2. **启动 UDS 消息服务器**（`feature('UDS_INBOX')`）
3. **终端备份恢复**（iTerm2、Terminal.app）
4. **设置工作目录**（`setCwd()`, `setProjectRoot()`）
5. **捕获 Hooks 配置快照**
6. **处理 Worktree 创建**（如需）
7. **初始化后台任务**
   - `initSessionMemory()`
   - `lockCurrentVersion()`
   - 插件预取
   - Hook 加载
8. **预取数据**
   - API Key 预取
   - 发布说明检查
   - 最近的活跃会话
9. **安全检查**
   - 权限模式验证
   - Docker/沙箱检测

### 2.5 REPL 启动器 (src/replLauncher.tsx)

```typescript
export async function launchRepl(options: ReplLaunchOptions): Promise<Instance> {
  // 1. 创建 Ink 根节点
  const root = await createRoot(options)

  // 2. 渲染 App 组件
  await render(
    createElement(App, {
      sessionId,
      initialMessage,
      permissionMode,
      // ...
    }),
    root,
  )

  return root
}
```

---

## 3. TUI 渲染引擎

项目使用 [Ink](https://github.com/vadimdemedes/ink) 作为终端 UI 渲染引擎。

### Ink 核心概念

Ink 是 React 的终端渲染版本，使用相同的组件模型：

```typescript
import { Box, Text, useInput } from 'ink'

function MyComponent() {
  useInput((input, key) => {
    if (key.return) {
      // 处理回车键
    }
  })

  return (
    <Box flexDirection="column">
      <Text>Hello, Terminal!</Text>
    </Box>
  )
}
```

### 核心组件 (`src/ink.ts`)

| 导出 | 用途 |
|------|------|
| `render()` | 渲染 React 节点到终端 |
| `createRoot()` | 创建 Ink 根节点 |
| `Box` | 容器组件（类似 Flexbox） |
| `Text` | 文本组件 |
| `useInput` | 键盘输入 Hook |
| `useStdin` | 标准输入 Hook |
| `useApp` | 应用上下文 Hook |
| `ThemeProvider` | 主题提供器 |

### REPL 界面 (`src/screens/REPL.tsx`)

主交互界面，负责：
- 消息列表显示
- 输入提示符
- 键盘快捷键处理
- Vim 模式支持
- 权限请求展示
- 费用阈值提醒

---

## 4. 工具系统

工具系统允许 Agent 执行各种操作（bash 命令、文件编辑、搜索等）。

### 工具注册 (`src/tools.ts`)

```typescript
export function getTools(permissionContext: PermissionContext): Tool[] {
  return [
    ...getAllBaseTools(),
    ...getMcpTools(),
    ...getSwarmTools(),
  ]
}

function getAllBaseTools(): Tool[] {
  return [
    AgentTool,
    BashTool,
    GrepTool,
    GlobTool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    WebFetchTool,
    WebSearchTool,
    TaskCreateTool,
    TaskListTool,
    TaskGetTool,
    TaskUpdateTool,
    SkillTool,
    MCPTool,
    // ...
  ]
}
```

### 核心工具

| 工具 | 功能 | 关键文件 |
|------|------|----------|
| `BashTool` | 执行 Shell 命令 | tools/BashTool/ |
| `FileReadTool` | 读取文件 | tools/FileReadTool/ |
| `FileEditTool` | 编辑文件 | tools/FileEditTool/ |
| `FileWriteTool` | 写入文件 | tools/FileWriteTool/ |
| `GlobTool` | 文件模式匹配 | tools/GlobTool/ |
| `GrepTool` | 内容搜索 | tools/GrepTool/ |
| `WebFetchTool` | 获取网页 | tools/WebFetchTool/ |
| `WebSearchTool` | 网络搜索 | tools/WebSearchTool/ |
| `TaskTool` | 任务管理 | tools/TaskTool/ |
| `AgentTool` | Agent 协调 | tools/AgentTool/ |
| `SkillTool` | Skill 调用 | tools/SkillTool/ |
| `MCPTool` | MCP 协议工具 | tools/MCPTool/ |

### 工具执行 (`src/services/tools/toolOrchestration.js`)

```typescript
export async function runTools(
  tools: Tool[],
  inputs: ToolInput[],
  context: ExecutionContext,
): Promise<ToolResult[]> {
  const results: ToolResult[] = []

  for (const input of inputs) {
    const tool = tools.find(t => t.name === input.name)
    if (!tool) {
      throw new Error(`Tool not found: ${input.name}`)
    }

    const result = await tool.execute(input.params, context)
    results.push(result)
  }

  return results
}
```

---

## 5. Agent 系统

### 多 Agent 架构

Claude Code 支持多个 Agent 协同工作：

```
User
  └─> Main Agent (Root)
        ├─> Teammate Agent 1
        ├─> Teammate Agent 2
        └─> Custom Agents
```

### Agent 定义

Agent 定义存储在 `.claude/agents/` 目录：

```json
{
  "name": "my-agent",
  "description": "Agent description",
  "instructions": "Agent instructions...",
  "tools": ["BashTool", "Read"],
  "model": "claude-opus-4-7-20251120"
}
```

### Agent 加载 (`src/tools/AgentTool/loadAgentsDir.js`)

```typescript
export function getAgentDefinitionsWithOverrides(): AgentDefinition[] {
  const bundled = getBundledAgents()
  const custom = loadCustomAgents()
  return [...bundled, ...custom]
}
```

### Agent 通信

- **UDS 消息传递**：Unix Domain Socket 用于同机器上的 Agent 间通信
- **SendMessageTool**：Agent 间发送消息
- **ListPeersTool**：列出已连接的 Agent

---

## 6. MCP 服务

MCP（Model Context Protocol）允许连接外部数据源和工具。

### MCP 客户端 (`src/services/mcp/client.ts`)

```typescript
export async function getMcpToolsCommandsAndResources(
  servers: McpServerConfig[],
): Promise<McpToolsAndResources> {
  const clients = await Promise.all(
    servers.map(config => createMcpClient(config))
  )

  return {
    tools: clients.flatMap(c => c.tools),
    commands: clients.flatMap(c => c.commands),
    resources: clients.flatMap(c => c.resources),
  }
}
```

### MCP 配置

MCP 服务器在 `.claude/mcp.json` 中配置：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    }
  }
}
```

### 支持的传输类型

| 传输类型 | 用途 |
|---------|------|
| StdioClientTransport | 标准 I/O（本地进程） |
| SSEClientTransport | Server-Sent Events |
| StreamableHTTPClientTransport | HTTP 流 |
| WebSocketTransport | WebSocket 连接 |

---

## 7. 状态管理

### Bootstrap 状态 (`src/bootstrap/state.js`)

```typescript
interface BootstrapState {
  sessionId: SessionId
  projectRoot: string
  originalCwd: string
  modelOverride: string | null
  mainThreadAgentType: AgentType
  // ...
}
```

### App 状态 (`src/state/AppStateStore.js`)

```typescript
export interface AppState {
  messages: Message[]
  speculation: Speculation | null
  tools: Tool[]
  permissionRequests: PermissionRequest[]
  notifications: Notification[]
  // ...
}

export function getDefaultAppState(): AppState {
  return {
    messages: [],
    speculation: null,
    tools: [],
    permissionRequests: [],
    notifications: [],
    // ...
  }
}
```

---

## 附录：关键文件索引

| 文件路径 | 功能 |
|---------|------|
| `bin/claude-haha` | Shell 入口脚本 |
| `preload.ts` | 设置 MACRO 全局变量 |
| `src/entrypoints/cli.tsx` | CLI 引导器（快速路径路由） |
| `src/main.tsx` | 主程序（初始化 + 命令解析） |
| `src/setup.ts` | 启动初始化逻辑 |
| `src/replLauncher.tsx` | REPL 启动器 |
| `src/screens/REPL.tsx` | 主交互界面 |
| `src/ink.ts` | Ink 渲染引擎导出 |
| `src/ink/` | Ink 核心实现 |
| `src/tools.ts` | 工具注册 |
| `src/tools/*/` | 各工具实现 |
| `src/services/mcp/client.ts` | MCP 客户端 |
| `src/bootstrap/state.js` | Bootstrap 状态 |
| `src/state/AppStateStore.js` | App 状态存储 |
| `src/commands.js` | 斜杠命令 |
| `src/query.ts` | 查询执行逻辑 |
| `src/Tool.ts` | Tool 类型定义 |
