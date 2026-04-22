# 多 Agent 系统详解

## 1. 概述

Claude Code 支持多 Agent 协同工作，允许主 Agent 启动子 Agent 处理复杂任务。

## 2. Agent 类型

### 2.1 内置 Agent

| Agent | 用途 | 源码 |
|------|------|------|
| `main` | 主 Agent（用户交互） | - |
| `claude-sonnet` | Sonnet 级别 Agent | - |
| `claude-haiku` | Haiku 级别 Agent | - |

### 2.2 自定义 Agent

用户可在 `.claude/agents/` 定义自定义 Agent：

```json
{
  "name": "code-reviewer",
  "description": "A specialized agent for reviewing code",
  "instructions": "You are a code reviewer. Analyze the provided code...",
  "tools": ["BashTool", "Read", "Grep"],
  "model": "claude-opus-4-7-20251120",
  "temperature": 0.5,
  "max_tokens": 4096
}
```

## 3. Agent 加载 (`src/tools/AgentTool/loadAgentsDir.js`)

```typescript
export function getAgentDefinitionsWithOverrides(): AgentDefinition[] {
  // 1. 加载内置 Agent
  const bundled = getBundledAgents()
  
  // 2. 加载用户自定义 Agent
  const custom = loadCustomAgents()
  
  // 3. 应用环境变量覆盖
  const overrides = getEnvironmentAgentOverrides()
  
  return [...bundled, ...custom, ...overrides]
}

function loadCustomAgents(): AgentDefinition[] {
  const agentsDir = path.join(getProjectRoot(), '.claude', 'agents')
  
  if (!existsSync(agentsDir)) {
    return []
  }
  
  const files = readdirSync(agentsDir)
    .filter(f => f.endsWith('.json') || f.endsWith('.ts'))
  
  return files.map(file => {
    if (file.endsWith('.json')) {
      return JSON.parse(readFileSync(file, 'utf8'))
    } else {
      return loadAgentFromTS(file)
    }
  })
}
```

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

```typescript
interface AgentMessage {
  type: 'query' | 'response' | 'tool_result' | 'error'
  from: string  // Agent ID
  to: string    // Agent ID
  payload: unknown
  id: string    // Message ID
  timestamp: number
}
```

### 5.3 SendMessageTool

用于 Agent 间发送消息：

```typescript
const SendMessageTool: Tool = {
  name: 'SendMessage',
  description: 'Send a message to another agent',
  input_schema: {
    type: 'object',
    properties: {
      recipient: { type: 'string', description: 'Agent name or ID' },
      message: { type: 'string', description: 'Message content' },
      urgency: { type: 'string', enum: ['normal', 'high'] },
    },
    required: ['recipient', 'message'],
  },
  handler: async (input, context) => {
    const message: AgentMessage = {
      type: 'query',
      from: context.agentId,
      to: input.recipient,
      payload: { content: input.message, urgency: input.urgency },
      id: generateId(),
      timestamp: Date.now(),
    }
    
    await sendViaUds(message)
    
    return {
      tool_use_id: input.tool_use_id,
      content: `Message sent to ${input.recipient}`,
    }
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

```typescript
async function spawnAgent(
  config: AgentConfig,
  prompt: string,
): Promise<AgentSession> {
  // 1. 解析 Agent 定义
  const definition = resolveAgentDefinition(config.agent ?? 'default')
  
  // 2. 创建会话
  const session = createAgentSession({
    id: generateAgentId(),
    definition,
    parentSession: config.parentSession,
    workspace: config.workspace,
  })
  
  // 3. 执行初始 prompt
  const result = await session.query(prompt)
  
  return session
}
```

### 7.2 会话管理

```typescript
interface AgentSession {
  id: string
  agentId: string
  parentId: string | null
  state: 'active' | 'completed' | 'failed'
  
  query(prompt: string): Promise<QueryResult>
  sendMessage(message: AgentMessage): Promise<void>
  terminate(): Promise<void>
}
```

## 8. 并发控制

### 8.1 最大并发数

```typescript
const MAX_CONCURRENT_AGENTS = parseInt(
  process.env.CLAUDE_CODE_MAX_CONCURRENT_AGENTS ?? '5',
  10
)

async function queryWithConcurrency(
  agents: Agent[],
  prompt: string,
): Promise<AgentResult[]> {
  const batches = chunkArray(agents, MAX_CONCURRENT_AGENTS)
  const results: AgentResult[] = []
  
  for (const batch of batches) {
    const batchResults = await Promise.all(
      batch.map(agent => agent.query(prompt))
    )
    results.push(...batchResults)
  }
  
  return results
}
```

### 8.2 资源限制

```typescript
interface AgentResourceLimits {
  maxMemoryMB: number
  maxCPUTimeMs: number
  maxTools: number
  timeoutMs: number
}

const DEFAULT_LIMITS: AgentResourceLimits = {
  maxMemoryMB: 512,
  maxCPUTimeMs: 300000,
  maxTools: 100,
  timeoutMs: 600000,
}
```

## 9. Swarm 模式

### 9.1 启用条件

```typescript
function isAgentSwarmsEnabled(): boolean {
  // 1. 检查环境变量
  if (process.env.CLAUDE_CODE_AGENT_SWARMS === '1') {
    return true
  }
  
  // 2. 检查 GrowthBook 功能开关
  if (feature('AGENT_SWARMS')) {
    return true
  }
  
  // 3. 检查配置文件
  const config = getGlobalConfig()
  return config.agentSwarmsEnabled ?? false
}
```

### 9.2 协调模式 (`COORDINATOR_MODE`)

当 `feature('COORDINATOR_MODE')` 启用时，主 Agent 可以充当协调者：

```typescript
// src/coordinator/coordinatorMode.js
export async function runCoordinatorMode(
  task: string,
  agents: AgentDefinition[],
): Promise<void> {
  // 1. 分解任务为子任务
  const subTasks = await decomposeTask(task)
  
  // 2. 分配子任务给 Agent
  const assignments = await assignTasksToAgents(subTasks, agents)
  
  // 3. 收集结果
  const results = await Promise.all(
    assignments.map(({ agent, task }) => agent.query(task))
  )
  
  // 4. 综合结果
  const finalResult = await synthesizeResults(results)
  
  // 5. 返回给用户
  return finalResult
}
```

## 10. 状态同步

### 10.1 主 Agent 状态广播

```typescript
async function broadcastStateUpdate(
  update: StateUpdate,
): Promise<void> {
  const activeSessions = getActiveSessions()
  
  await Promise.all(
    activeSessions
      .filter(s => s.id !== getMainAgentId())
      .map(s => s.receiveUpdate(update))
  )
}
```

### 10.2 冲突解决

```typescript
interface ConflictResolution {
  strategy: 'latest-wins' | 'merge' | 'parent-wins'
  mergeFields?: string[]
}

function resolveConflict(
  local: AgentState,
  remote: AgentState,
  strategy: ConflictResolution,
): AgentState {
  switch (strategy.strategy) {
    case 'latest-wins':
      return remote.timestamp > local.timestamp ? remote : local
    
    case 'merge':
      return {
        ...local,
        ...remote,
        messages: [...local.messages, ...remote.messages],
      }
    
    case 'parent-wins':
      return local
  }
}
```

## 11. 调试和监控

### 11.1 日志

```typescript
logEvent('agent_spawned', {
  agent_id: session.id,
  parent_id: session.parentId,
  definition: session.definition.name,
})

logEvent('agent_completed', {
  agent_id: session.id,
  duration_ms: Date.now() - session.startTime,
  messages_count: session.messages.length,
})
```

### 11.2 追踪

```typescript
// 启用 Agent 追踪
process.env.CLAUDE_CODE_TRACE_AGENTS = '1'

async function tracedQuery(session, prompt) {
  const traceId = generateTraceId()
  console.log(`[TRACE:${traceId}] Starting query`)
  
  try {
    const result = await session.query(prompt)
    console.log(`[TRACE:${traceId}] Completed: ${result.content.length} chars`)
    return result
  } catch (error) {
    console.log(`[TRACE:${traceId}] Failed: ${error.message}`)
    throw error
  }
}
```
