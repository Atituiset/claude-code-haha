# 工具系统详解

## 1. 概述

工具系统是 Claude Code 的核心，允许 Agent 与外部世界交互。工具可以是命令执行、文件操作、网络请求等各种能力。

## 2. 工具类型

### 2.1 内置工具

| 工具 | 功能 | 源码位置 |
|------|------|----------|
| `AgentTool` | 启动子 Agent | `src/tools/AgentTool/` |
| `BashTool` | 执行 Shell 命令 | `src/tools/BashTool/` |
| `Read` | 读取文件 | `src/tools/FileReadTool/` |
| `Edit` | 编辑文件 | `src/tools/FileEditTool/` |
| `Write` | 写入文件 | `src/tools/FileWriteTool/` |
| `Glob` | 文件模式匹配 | `src/tools/GlobTool/` |
| `Grep` | 内容搜索 | `src/tools/GrepTool/` |
| `WebFetch` | 获取网页 | `src/tools/WebFetchTool/` |
| `WebSearch` | 网络搜索 | `src/tools/WebSearchTool/` |
| `TaskCreate` | 创建任务 | `src/tools/TaskTool/` |
| `TaskList` | 列出任务 | `src/tools/TaskTool/` |
| `TaskGet` | 获取任务详情 | `src/tools/TaskTool/` |
| `TaskUpdate` | 更新任务 | `src/tools/TaskTool/` |
| `Skill` | 调用 Skill | `src/tools/SkillTool/` |
| `MCPTool` | MCP 工具封装 | `src/tools/MCPTool/` |
| `EnterPlanMode` | 进入计划模式 | - |
| `ExitPlanMode` | 退出计划模式 | - |
| `EnterWorktree` | 进入工作树 | - |
| `ExitWorktree` | 退出工作树 | - |
| `Sleep` | 延迟执行 | - |
| `Brief` | 摘要生成 | - |

### 2.2 工具定义 (`src/Tool.ts`)

```typescript
export interface Tool {
  name: string
  description: string
  input_schema: ToolInputJSONSchema
  annotations?: ToolAnnotation
  handler?: ToolHandler
  
  // 权限相关
  dangerous?: boolean
  requiresPermissions?: boolean
}

export interface ToolAnnotation {
  title?: string
  description?: string
  kind?: 'information' | 'action' | 'warning'
  bypassA11y?: boolean
  skipConfirmation?: boolean
}

export interface ToolInputJSONSchema {
  type: 'object'
  properties?: Record<string, JSONSchemaProperty>
  required?: string[]
  definitions?: Record<string, JSONSchemaProperty>
}
```

## 3. 工具注册

### 3.1 注册流程 (`src/tools.ts`)

```typescript
export function getTools(
  permissionContext: PermissionContext,
  workspace: Workspace | null,
): Tool[] {
  const tools: Tool[] = []
  
  // 1. 添加基础工具
  tools.push(...getAllBaseTools())
  
  // 2. 添加 MCP 工具
  tools.push(...getMcpTools())
  
  // 3. 添加 Agent 相关工具
  if (isAgentSwarmsEnabled()) {
    tools.push(...getSwarmTools())
  }
  
  // 4. 根据权限过滤
  return filterToolsByPermissions(tools, permissionContext)
}
```

### 3.2 权限过滤

```typescript
function filterToolsByPermissions(
  tools: Tool[],
  permissionContext: PermissionContext,
): Tool[] {
  return tools.filter(tool => {
    // 跳过需要权限但未授权的工具
    if (tool.requiresPermissions && !permissionContext.hasPermission(tool.name)) {
      return false
    }
    
    // 跳过危险工具
    if (tool.dangerous && !permissionContext.allowDangerousTools) {
      return false
    }
    
    return true
  })
}
```

## 4. 工具执行

### 4.1 执行入口 (`src/query.ts`)

```typescript
export async function executeQuery(
  context: QueryContext,
): Promise<QueryResult> {
  // 1. 获取工具列表
  const tools = getTools(context.permissionContext, context.workspace)
  
  // 2. 准备工具输入
  const toolInputs = parseToolCalls(apiResponse.tool_calls)
  
  // 3. 执行工具
  const toolResults = await runTools(tools, toolInputs, context)
  
  // 4. 返回结果
  return {
    content: toolResults.map(result => ({
      type: 'tool_result',
      tool_use_id: result.tool_use_id,
      content: result.content,
    })),
    is_complete: false,
  }
}
```

### 4.2 工具编排 (`src/services/tools/toolOrchestration.js`)

```typescript
export async function runTools(
  tools: Tool[],
  inputs: ToolInput[],
  context: ExecutionContext,
): Promise<ToolResult[]> {
  const results: ToolResult[] = []
  
  for (const input of inputs) {
    // 1. 查找工具
    const tool = tools.find(t => t.name === input.name)
    if (!tool) {
      results.push({
        tool_use_id: input.tool_use_id,
        content: `Error: Tool not found: ${input.name}`,
        is_error: true,
      })
      continue
    }
    
    // 2. 执行工具（可并行或串行）
    if (input.parallel) {
      // 并行执行
      const parallelResults = await Promise.all(
        inputs.filter(i => i.name === input.name).map(i => tool.handler!(i, context))
      )
      results.push(...parallelResults)
    } else {
      // 串行执行
      const result = await tool.handler!(input, context)
      results.push(result)
    }
  }
  
  return results
}
```

## 5. 核心工具详解

### 5.1 BashTool (`src/tools/BashTool/`)

执行 Shell 命令：

```typescript
const BashTool: Tool = {
  name: 'Bash',
  description: 'Execute shell commands',
  input_schema: {
    type: 'object',
    properties: {
      command: {
        type: 'string',
        description: 'The shell command to execute',
      },
      timeout: {
        type: 'number',
        description: 'Timeout in milliseconds',
        default: 60000,
      },
      workdir: {
        type: 'string',
        description: 'Working directory',
      },
    },
    required: ['command'],
  },
  annotations: {
    kind: 'action',
    title: 'Run Command',
  },
  handler: async (input, context) => {
    const result = await executeBash(input.command, {
      cwd: input.workdir ?? context.cwd,
      timeout: input.timeout,
      env: context.env,
    })
    
    return {
      tool_use_id: input.tool_use_id,
      content: result.stdout + result.stderr,
      metadata: {
        exit_code: result.exit_code,
        duration_ms: result.duration_ms,
      },
    }
  },
}
```

**安全机制：**
- 命令白名单过滤
- 超时控制
- 工作目录限制
- 环境变量清理

### 5.2 FileEditTool (`src/tools/FileEditTool/`)

文件编辑使用"保持目标"的语义：

```typescript
const FileEditTool: Tool = {
  name: 'Edit',
  description: 'Make edits to a file',
  input_schema: {
    type: 'object',
    properties: {
      file_path: { type: 'string' },
      old_string: { type: 'string' },
      new_string: { type: 'string' },
      dry_run: { type: 'boolean', default: false },
    },
    required: ['file_path', 'old_string', 'new_string'],
  },
  handler: async (input, context) => {
    // 1. 读取文件
    const content = await readFile(input.file_path)
    
    // 2. 验证 old_string 存在
    if (!content.includes(input.old_string)) {
      return {
        tool_use_id: input.tool_use_id,
        content: `Error: old_string not found in file`,
        is_error: true,
      }
    }
    
    // 3. 执行替换
    const newContent = content.replace(old_string, new_string)
    
    // 4. 写入文件
    await writeFile(input.file_path, newContent)
    
    return {
      tool_use_id: input.tool_use_id,
      content: `Edited ${input.file_path}`,
    }
  },
}
```

### 5.3 GrepTool (`src/tools/GrepTool/`)

使用 Ripgrep 进行内容搜索：

```typescript
const GrepTool: Tool = {
  name: 'Grep',
  description: 'Search file contents using ripgrep',
  input_schema: {
    type: 'object',
    properties: {
      pattern: { type: 'string' },
      path: { type: 'string' },
      case_sensitive: { type: 'boolean', default: false },
      recursive: { type: 'boolean', default: true },
      context_lines: { type: 'number', default: 0 },
    },
    required: ['pattern'],
  },
  handler: async (input, context) => {
    const args = [
      input.pattern,
      input.path ?? '.',
      '--json',
      input.case_sensitive ? '' : '-i',
      input.recursive ? '-r' : '',
    ].filter(Boolean)
    
    const result = await spawn('rg', args)
    const matches = parseRipgrepJson(result.stdout)
    
    return {
      tool_use_id: input.tool_use_id,
      content: JSON.stringify(matches),
    }
  },
}
```

### 5.4 AgentTool (`src/tools/AgentTool/`)

启动子 Agent：

```typescript
const AgentTool: Tool = {
  name: 'Agent',
  description: 'Start a subsidiary agent',
  input_schema: {
    type: 'object',
    properties: {
      model: { type: 'string' },
      agent: { type: 'string' },
      prompt: { type: 'string' },
      max_tokens: { type: 'number' },
    },
    required: ['prompt'],
  },
  handler: async (input, context) => {
    // 1. 解析 Agent 定义
    const agentDef = resolveAgentDefinition(input.agent)
    
    // 2. 创建子 Agent 会话
    const session = await createAgentSession({
      ...agentDef,
      parentContext: context,
      model: input.model ?? agentDef.model,
    })
    
    // 3. 执行查询
    const result = await session.query(input.prompt)
    
    return {
      tool_use_id: input.tool_use_id,
      content: result.content,
    }
  },
}
```

## 6. MCP 工具 (`src/tools/MCPTool/`)

MCP 工具是对 Model Context Protocol 服务器工具的封装：

```typescript
export class MCPTool implements Tool {
  name: string
  description: string
  input_schema: ToolInputJSONSchema
  
  constructor(
    private serverName: string,
    private mcpTool: McpServerTool,
  ) {
    this.name = `mcp_${serverName}_${mcpTool.name}`
    this.description = mcpTool.description
    this.input_schema = mcpTool.inputSchema
  }
  
  async handler(input: ToolInput, context: ExecutionContext): Promise<ToolResult> {
    // 1. 获取 MCP 客户端
    const client = getMcpClient(this.serverName)
    
    // 2. 调用工具
    const result = await client.callTool(this.mcpTool.name, input.params)
    
    return {
      tool_use_id: input.tool_use_id,
      content: formatToolResult(result),
    }
  }
}
```

## 7. 工具权限

### 7.1 权限模式

| 模式 | 描述 | 行为 |
|------|------|------|
| `auto` | 自动模式 | 首次使用时请求权限 |
| `bypass` | 绕过模式 | 所有工具直接执行 |
| `haiku` | Haiku 模式 | 限制性最强的模式 |
| `localRecovery` | 本地恢复 | 最小权限 |

### 7.2 权限请求

```typescript
interface PermissionRequest {
  tool: Tool
  params: Record<string, unknown>
  reason: string
  userConfirmation?: Promise<boolean>
}
```

### 7.3 权限存储

权限决策存储在 `.claude/settings.json`：

```json
{
  "permissions": {
    "toolPermissions": {
      "Bash": {
        "allowed": true,
        "lastAllowed": "2026-01-15T10:30:00Z"
      },
      "Read": {
        "allowed": true,
        "lastAllowed": "2026-01-15T10:30:00Z"
      }
    }
  }
}
```

## 8. 工具调用协议

### 8.1 Anthropic API 格式

Anthropic API 使用 `tool_calls` 格式：

```typescript
// API 请求
{
  model: 'claude-opus-4-7-20251120',
  max_tokens: 4096,
  tools: [
    {
      name: 'Bash',
      description: 'Execute shell commands',
      input_schema: { ... }
    }
  ],
  messages: [
    { role: 'user', content: 'List files in current directory' }
  ]
}

// API 响应
{
  content: [
    {
      type: 'text',
      text: 'Here are the files...'
    },
    {
      type: 'tool_use',
      id: 'toolu_xxx',
      name: 'Bash',
      input: { command: 'ls -la' }
    }
  ],
  stop_reason: 'tool_use'
}
```

### 8.2 工具结果返回

```typescript
// 继续请求
{
  messages: [
    { role: 'user', content: 'List files in current directory' },
    { role: 'assistant', content: null, tool_calls: [...] },
    {
      role: 'user',
      content: [
        {
          type: 'tool_result',
          tool_use_id: 'toolu_xxx',
          content: 'total 32\ndrwxr-xr-x  4 user user 4096 Jan 15 10:30 .\n...'
        }
      ]
    }
  ]
}
```

## 9. 自定义工具

用户可以在 `.claude/tools/` 目录添加自定义工具：

```typescript
// .claude/tools/myCustomTool.ts
export const myCustomTool = {
  name: 'MyCustom',
  description: 'A custom tool',
  input_schema: {
    type: 'object',
    properties: {
      input: { type: 'string' }
    },
    required: ['input']
  },
  handler: async (input, context) => {
    // 自定义逻辑
    return {
      tool_use_id: input.tool_use_id,
      content: `Processed: ${input.input}`,
    }
  }
}
```

## 10. 调试工具

### 10.1 工具日志

```typescript
// 启用工具调试
process.env.CLAUDE_CODE_TOOL_DEBUG = '1'
```

### 10.2 工具追踪

```typescript
import { traceTool } from '../utils/trace.js'

async function tracedHandler(input, context) {
  return traceTool(input.name, () => handler(input, context))
}
```
