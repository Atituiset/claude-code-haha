# 多 Agent 系统详解

## 1. 概述

Claude Code 支持多 Agent 协同工作，允许主 Agent 启动子 Agent 处理复杂任务。

## 2. Agent 类型

### 2.1 内置 Agent

6 个内置 Agent 类型（`src/tools/AgentTool/builtInAgents.ts`，详见第 15/16 章）：`generalPurpose`、`claudeCodeGuide`、`explore`、`plan`、`verification`、`statuslineSetup`。主 Agent（用户交互）不在此列，由 REPL 会话直接承载。

### 2.2 自定义 Agent

用户可在 `.claude/agents/`（项目级）或 `~/.claude/agents/`（用户级）定义自定义 Agent，格式为 **Markdown + frontmatter**（`src/tools/AgentTool/loadAgentsDir.ts`）：

```markdown
---
name: code-reviewer
description: A specialized agent for reviewing code
tools: Read, Grep, Glob      # 可选：未指定时继承主 Agent 工具集
model: opus                  # 可选：模型别名或完整 ID
---

You are a code reviewer. Analyze the provided code...
```

> 注意：没有 `temperature`/`max_tokens` frontmatter 字段，也不支持 `.json`/`.ts` 格式的 Agent 定义文件。

## 3. Agent 加载 (`src/tools/AgentTool/loadAgentsDir.ts`)

```typescript
// getAgentDefinitionsWithOverrides()（memoized）按来源合并并按优先级覆盖：
// 1. 内置 Agent（builtInAgents.ts，6 个类型）
// 2. 用户全局（~/.claude/agents/*.md）
// 3. 项目（.claude/agents/*.md）
// 4. 插件 / 托管来源
// 同名定义按来源优先级高者覆盖低者
```

加载时会解析 frontmatter（name/description/tools/model/skills 等），并校验看起来像 Agent 定义的文件（有 name 字段的 frontmatter），详见 `loadMarkdownFilesForSubdir('agents', cwd)`（`loadAgentsDir.ts:308`）。

## 4. Agent 通信架构

```
                    ┌─────────────────┐
                    │     User        │
                    │   (Terminal)    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Main Agent    │
                    │  (Root/Parent)  │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
┌────────▼────────┐  ┌───────▼────────┐  ┌───────▼────────┐
│  Teammate 1     │  │  Teammate 2    │  │ Custom Agent  │
│  (sub-agent)    │  │  (sub-agent)   │  │  (user-def)   │
└─────────────────┘  └───────────────┘  └───────────────┘
         │                   │                   │
         └───────────────────┴───────────────────┘
                             │
                    ┌────────▼────────┐
                    │  UDS Messaging  │
                    │    Server      │
                    └────────────────┘
```

## 5. 消息传递

### 5.1 UDS (Unix Domain Socket) 消息

当 `feature('UDS_INBOX')` 启用时，Agent 间通过 UDS 通信：

```typescript
// src/utils/udsMessaging.js
export async function startUdsMessaging(
  socketPath: string,
  options: { isExplicit: boolean },
): Promise<void> {
  const server = new UnixSocketServer(socketPath)
  
  server.on('connection', (socket) => {
    socket.on('data', async (data) => {
      const message = JSON.parse(data.toString())
      await handleMessage(message)
    })
  })
  
  await server.listen()
}
```

### 5.2 消息类型

Swarm 通信采用**邮箱模型**（mailbox）而非直连消息结构：每个 Agent 有自己的 inbox 目录/队列，核心操作是：

- **SendMailboxMessage** — 向指定队友的邮箱投递消息
- **PollInbox** — 轮询自己的邮箱收取新消息
- **ListPeers** — 列出当前连接的 Agent

（源码入口：`src/utils/swarm/`、`src/utils/udsMessaging.ts`、`src/tools/SendMessageTool/`、`src/tools/ListPeersTool/`）

### 5.3 SendMessageTool

用于 Agent 间发送邮箱消息（真实工具见 `src/tools/SendMessageTool/`；无 `urgency` 字段，消息投递到目标 Agent 的 inbox，由其 `PollInbox` 轮询接收）：

```typescript
const SendMessageTool: Tool = {
  name: 'SendMessage',
  description: 'Send a message to a teammate',
  input_schema: {
    type: 'object',
    properties: {
      recipient: { type: 'string', description: 'Target agent' },
      message: { type: 'string', description: 'Message content' },
      // 实际字段以 src/tools/SendMessageTool/prompt.ts 为准
    },
    required: ['recipient', 'message'],
  },
  async call(input, context) {
    // 将消息写入目标 Agent 的邮箱（mailbox）目录
    // 目标 Agent 通过 PollInbox 类工具/轮询器读取
  },
}
```

## 6. Teammate 系统

当 `isAgentSwarmsEnabled()` 返回 `true` 时启用：

### 6.1 Teammate 快照

```typescript
// src/utils/swarm/backends/teammateModeSnapshot.js
export function captureTeammateModeSnapshot(): void {
  const snapshot = {
    agents: getActiveAgents(),
    messages: getRecentMessages(100),
    context: captureCurrentContext(),
    timestamp: Date.now(),
  }
  
  // 保存到磁盘供恢复
  writeFileSync(
    path.join(getSessionDir(), 'teammate-snapshot.json'),
    JSON.stringify(snapshot),
  )
}
```

### 6.2 Teammate Prompt 补充

```typescript
// src/utils/swarm/teammatePromptAddendum.js
export function getTeammatePromptAddendum(
  teammateId: string,
): string {
  const mainContext = getCurrentMainAgentContext()
  const activeAgents = getActiveTeammates()
  
  return `
Current team state:
- Main agent: ${mainContext.id}
- Active teammates: ${activeAgents.map(a => a.id).join(', ')}

Your ID: ${teammateId}

Recent context from main agent:
${mainContext.recentMessages.slice(-5).join('\n')}
  `
}
```

## 7. Agent 创建流程

### 7.1 启动子 Agent

真实流程（`src/tools/AgentTool/AgentTool.tsx` + `runAgent.ts`，详见第 15/16 章）：

```
AgentTool.call(input)
  → 解析 Agent 定义（内置 / .claude/agents/*.md）
  → runAgent(): 为子 Agent 构造受限的 ToolUseContext
      （工具集按 agent.tools 过滤，剔除 ALL_AGENT_DISALLOWED_TOOLS）
  → 子 Agent 跑独立 query() 循环
  → 输出作为 ToolResult 返回主 Agent
```

### 7.2 会话管理

子 Agent 会话与任务系统对接（async agent 的输出通过 `TaskOutput` 类工具在后台获取；`TaskStop` 可中止）。没有书中早期虚构的 `AgentSession` 接口——异步生命周期由 `src/tasks/` 的 TaskState 管理。

## 8. 并发控制

### 8.1 并发相关约束

多 Agent 并发不使用书中早期虚构的 `CLAUDE_CODE_MAX_CONCURRENT_AGENTS` 批处理调度。真实约束分散在：

- **工具级并发**：`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`（默认 10，见 `toolOrchestration.ts:7`）限制并发工具批
- **Swarm Worker**：由 `runAgent.ts` / `workerAgent.ts` 驱动独立 Agent Loop，各自受 Token 预算（`query/tokenBudget.ts`）与 `--max-budget-usd` 等限制保护
- **任务系统**：后台任务（TaskCreate 等）由 `src/tasks/` 管理，Agent 主线程可继续处理输入

### 8.2 资源限制

真实限额以 Token/成本为主（内存/CPU 时间配额不存在于源码中）：

- `--max-budget-usd`：CLI 参数，限制 API 花费（`src/main.tsx`）
- `--max-turns`：非交互模式下的最大 agentic 轮数
- `TOKEN_BUDGET` 特性：BudgetTracker（90% 完成阈值 / 500 token 递减阈值，`query/tokenBudget.ts:3-4`）

## 9. Swarm 模式

### 9.1 启用条件（`src/utils/agentSwarmsEnabled.ts:24`）

```typescript
export function isAgentSwarmsEnabled(): boolean {
  // Ant 构建：始终启用
  if (process.env.USER_TYPE === 'ant') return true

  // 外部构建需同时满足：
  // 1. 显式开启：CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 环境变量
  //    或 --agent-teams CLI flag
  // 2. GrowthBook 门控 'tengu_amber_flint'（killswitch）
  // ...
}
```

### 9.2 协调模式 (`COORDINATOR_MODE`)

当 `feature('COORDINATOR_MODE')` 启用时，主 Agent 可以充当协调者（真实实现见 `src/coordinator/coordinatorMode.ts` 与 `workerAgent.ts`）：协调者把自身工具集限制为任务管理 + Agent + 通信（见第 16 章 COORDINATOR_MODE_ALLOWED_TOOLS），Worker 走独立 Agent Loop 并通过邮箱/权限桥接与 Leader 交互。

## 10. 状态同步

### 10.1 真实机制

没有书中早期版本的"广播 StateUpdate + 三策略冲突解决"实现。状态同步的真实载体：

- **团队目录**：`~/.claude/teams/<team>/`（config.json + inboxes/ + subagents/），teamWatcher（`src/server/services/teamWatcher.ts`）每 3 秒轮询并广播到 WebSocket 客户端
- **会话元数据**：`notifySessionMetadataChanged` 把权限模式等变更同步给 CCR/SDK（`src/state/onChangeAppState.ts`）
- **Agent 记忆**：`agentMemorySnapshot.ts` 支持合并/替换/保留策略（跨会话持久化，见第 16 章）

## 11. 调试和监控

### 11.1 日志

工具与 Agent 生命周期事件通过 `logEvent` 遥测打点（`src/services/analytics/`），配合 `--debug` / `--debug-to-stderr` 可在本地观察 agent 启动、工具调用、权限决策等事件流。

### 11.2 追踪

没有 `CLAUDE_CODE_TRACE_AGENTS` 环境变量；调试手段是：

```bash
# 调试输出
claude-haha --debug
claude-haha --debug-to-stderr
# MCP 调试
claude-haha --mcp-debug
```

（更多启动参数见 `claude-haha --help` 或 `src/main.tsx` 的 Commander 定义。）
```
