# 程序执行流程

> 本章梳理从 `claude-haha` 命令到 REPL 就绪的完整启动链路，并给出后续各章的地图。

## 1. 项目概述

Claude Code Haha 是基于 2026 年 3 月从 Anthropic npm registry 泄露的 Claude Code 源码修复的**本地可运行版本**。

### 技术栈

| 类别 | 技术 |
|------|------|
| 运行时 | Bun |
| 语言 | TypeScript（少量 `.js` 遗留） |
| 终端 UI | React + Ink（定制版） |
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
│   ├── init.ts              # 13 步完整初始化
│   └── mcp.ts               # MCP Server 模式入口
├── main.tsx                 # 主程序（Commander.js + React/Ink）
├── setup.ts                 # 启动初始化
├── replLauncher.tsx         # REPL 启动器
├── screens/
│   └── REPL.tsx             # 主交互界面（~5000 行）
├── ink/                     # Ink 终端渲染引擎（定制实现）
├── ink.ts                   # Ink 导出薄封装
├── components/              # UI 组件
├── tools/                   # Agent 工具（50+ 个目录）
├── tools.ts                 # 工具注册中心
├── commands/                # 斜杠命令实现
├── commands.ts              # 命令注册表
├── skills/                  # Skill 系统
├── services/
│   ├── api/                 # API 客户端（claude.ts, client.ts, withRetry.ts）
│   ├── mcp/                 # MCP 协议
│   └── tools/               # 工具编排/执行
├── state/                   # App 状态（AppStateStore.ts）
├── bootstrap/state.ts       # 全局可变单例状态
├── query.ts                 # Agent 主循环 async generator
└── utils/                   # 工具函数（含 permissions/, model/）
```

> 注意：本仓库 TypeScript 文件为 `.ts/.tsx`，文档中早期版本写 `.js` 的路径均已修正。

---

## 2. 执行流程详解

### 2.1 入口脚本 (bin/claude-haha)

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
# 记录调用者工作目录为环境变量
export CALLER_DIR="${CALLER_DIR:-$(pwd -W 2>/dev/null || pwd)}"
cd "$ROOT_DIR"

# Desktop/Web 服务器作为父进程启动 CLI 时跳过 .env 加载（避免陈旧的
# provider key 覆盖已激活的 provider 配置）
if [[ "${CC_HAHA_SKIP_DOTENV:-0}" == "1" ]]; then
  ENV_FILE_FLAG="--env-file=/dev/null"
elif [[ -f .env ]]; then
  ENV_FILE_FLAG="--env-file=.env"
else
  ENV_FILE_FLAG=""
fi

# 降级模式：简单 readline REPL，无 Ink TUI
if [[ "${CLAUDE_CODE_FORCE_RECOVERY_CLI:-0}" == "1" ]]; then
  exec bun $ENV_FILE_FLAG ./src/localRecoveryCli.ts "$@"
fi

# 默认：完整 CLI + Ink TUI
exec bun $ENV_FILE_FLAG ./src/entrypoints/cli.tsx "$@"
```

**关键逻辑：**
1. 记录 `CALLER_DIR`（调用者目录）并切换到项目根目录
2. 按 `CC_HAHA_SKIP_DOTENV` / `.env` 存在性决定环境加载方式
3. 检查 `CLAUDE_CODE_FORCE_RECOVERY_CLI`
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

`setup()` 函数执行大量初始化工作（真实签名见 `src/setup.ts`，含 cwd、permissionMode、worktree、tmux、customSessionId 等参数）：

**主要步骤（源码注释编号）：**

1. **Node.js 版本检查**（需 >= 18；Bun 运行时同样满足该检查）
2. **启动 UDS 消息服务器**（Swarm/teammate 通信，非固定 feature 门控）
3. **捕获 Hooks 配置快照**
4. **启动文件变更监听器**（hook 触发器）
5. **定位 git root，设置项目根目录**（`setCwd()`, `setProjectRoot()`）
6. **处理 Worktree 创建**（如需）
7. **后台家务任务**（并行、非阻塞）
8. **初始化会话记忆**（`initSessionMemory()`）
9. **终端备份恢复**（Apple Terminal、iTerm2）
10. **锁定当前版本**（原生安装器场景）

> 注意：早期版本文档把"API Key 预取、发布说明检查、Docker 检测"列为 setup 步骤，实际这些发生在 `main.tsx` 的 action handler 与 `entrypoints/init.ts` 的 13 步初始化中（见第 14 章）。

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

`getTools(permissionContext)` 只返回内置工具（真实签名见 `src/tools.ts:272`）：

```typescript
export const getTools = (permissionContext: ToolPermissionContext): Tools => {
  // Simple 模式：仅 Bash / Read / Edit（CLAUDE_CODE_SIMPLE=1）
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) { /* ... */ }

  const specialTools = new Set([
    ListMcpResourcesTool.name,
    ReadMcpResourceTool.name,
    SYNTHETIC_OUTPUT_TOOL_NAME,
  ])

  const tools = getAllBaseTools().filter(tool => !specialTools.has(tool.name))
  // 过滤 deny 规则命中的工具，再按 isEnabled() 过滤
  return allowedTools.filter(...)
}
```

MCP 工具不在这里注册，而是由 `assembleToolPool`（`src/tools.ts:338` 附近）把 `getTools()` 结果与 MCP 工具按名字去重合并（内置优先），供 REPL（`useMergedTools`）和子 Agent 运行时共用。

### 核心工具（`src/tools/` 目录）

| 工具 | 功能 | 关键文件 |
|------|------|----------|
| `BashTool` | 执行 Shell 命令 | tools/BashTool/ |
| `FileReadTool`（工具名 `Read`） | 读取文件 | tools/FileReadTool/ |
| `FileEditTool`（工具名 `Edit`） | 编辑文件 | tools/FileEditTool/ |
| `FileWriteTool`（工具名 `Write`） | 写入文件 | tools/FileWriteTool/ |
| `GlobTool` | 文件模式匹配 | tools/GlobTool/ |
| `GrepTool` | 内容搜索 | tools/GrepTool/ |
| `WebFetchTool` | 获取网页 | tools/WebFetchTool/ |
| `WebSearchTool` | 网络搜索 | tools/WebSearchTool/ |
| Task 系列（`TaskCreate/Get/List/Output/Stop/Update`） | 后台任务管理 | tools/Task*Tool/ |
| `AgentTool` | 子 Agent | tools/AgentTool/ |
| `SkillTool`（工具名 `Skill`） | Skill 调用 | tools/SkillTool/ |
| `MCPTool` | MCP 协议工具 | tools/MCPTool/ |
| `TodoWriteTool` | 待办管理 | tools/TodoWriteTool/ |

### 工具编排 (`src/services/tools/toolOrchestration.ts`)

真实实现不是简单的 find+execute 循环，而是按"连续并发安全批"分区的 async generator（详见第 15 章）：

```typescript
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate, void, void> {
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(...)) {
    if (isConcurrencySafe) {
      // 只读批并发执行（上限 CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY，默认 10）
      yield* runToolsConcurrently(blocks, ...)
    } else {
      // 写操作批串行执行
      yield* runToolsSerially(blocks, ...)
    }
  }
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

Agent 定义存储在 `.claude/agents/`（项目级）与 `~/.claude/agents/`（用户级）目录，格式是 **Markdown + frontmatter**（不是 JSON，见 `src/tools/AgentTool/loadAgentsDir.ts`）：

```markdown
---
name: my-agent
description: Agent description
tools: Read, Grep, Glob
model: opus
---

Agent instructions...
```

### Agent 加载 (`src/tools/AgentTool/loadAgentsDir.ts`)

`getAgentDefinitionsWithOverrides()`（memoized）合并：内置 Agent（`builtInAgents.ts`，6 个类型）+ 用户/项目/插件/托管来源的自定义 Agent，同名按来源优先级覆盖。

### Agent 通信

- **UDS 消息传递**：Unix Domain Socket 用于同机器上的 Agent 间通信
- **SendMessageTool**：Agent 间邮箱消息
- **ListPeersTool**：列出已连接的 Agent

---

## 6. MCP 服务

MCP（Model Context Protocol）允许连接外部数据源和工具。

### MCP 配置

MCP 服务器按作用域分层配置（`src/services/mcp/config.ts`、`utils.ts:268`）：

| 作用域 | 文件 |
|--------|------|
| project | `.mcp.json`（项目根，可提交到 git 共享） |
| user | `~/.claude.json`（mcpServers 键） |
| local | `~/.claude.json`（带 project 标记，私有） |
| enterprise | `managed-mcp.json`（托管策略） |
| claude.ai | 云端连接器（自动同步） |

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

### Bootstrap 状态 (`src/bootstrap/state.ts`)

全局可变单例（`src/bootstrap/state.ts`，~1781 行、约 260 个字段），每字段配套 `get*/set*` 访问器：

```typescript
const State = {
  sessionId: SessionId
  originalCwd: string
  projectRoot: string
  clientType: 'cli'
  mainLoopModelOverride: string | undefined
  mainThreadAgentType: string | undefined
  // ... 数百个字段
}
```

### App 状态 (`src/state/AppStateStore.ts`)

`AppState` 是 DeepImmutable 类型（`src/state/AppStateStore.ts:89`），关键字段包括：

```typescript
export type AppState = DeepImmutable<{
  messages: Message[]
  tools: Tool[]
  speculation: SpeculationState        // 提示建议的推测执行状态
  notifications: { ... }
  toolPermissionContext: ToolPermissionContext
  tasks: ...
  // ...
}>
```

状态写入经由 `setAppState`，变更 diff 由 `onChangeAppState`（`src/state/onChangeAppState.ts:43`）消费——它负责权限模式同步、模型设置持久化等副作用。

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
| `src/services/mcp/config.ts` | MCP 配置解析 |
| `src/bootstrap/state.ts` | Bootstrap 状态单例 |
| `src/state/AppStateStore.ts` | App 状态类型与默认值 |
| `src/state/onChangeAppState.ts` | 状态变更副作用 |
| `src/commands.ts` | 斜杠命令注册表 |
| `src/query.ts` | Agent 主循环 |
| `src/Tool.ts` | Tool 类型定义 |
