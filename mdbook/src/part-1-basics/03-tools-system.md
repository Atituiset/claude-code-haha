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

真实签名只接收 permissionContext（`src/tools.ts:272`），MCP 工具的合并发生在 `assembleToolPool`：

```typescript
export const getTools = (
  permissionContext: ToolPermissionContext,
): Tools => {
  // Simple 模式: 仅 Bash/Read/Edit
  const tools = getAllBaseTools()   // 静态导入 + 特性门控条件 require
  // 过滤 deny 规则 + isEnabled()
  return filterToolsByDenyRules(tools, permissionContext).filter(...)
}

// MCP 合并入口（src/tools.ts:338 附近）
// 1. getTools() 获取内置工具
// 2. MCP 工具按 deny 规则过滤
// 3. 按工具名去重（内置优先）
// 供 REPL 的 useMergedTools 和子 Agent 运行时共用
```

许多工具是特性门控的条件加载（如 `SleepTool` 需要 `PROACTIVE`/`KAIROS`，cron 系列需要 `AGENT_TRIGGERS`），详见 `src/tools.ts` 顶部。

### 3.2 权限过滤

工具级过滤只剔除被 deny 规则命中的工具；**单次调用**的 allow/ask 决策发生在执行期（`toolExecution.ts` 调 `hasPermissionsToUseTool`），不是注册期：

```typescript
// src/tools.ts:264 附近
function filterToolsByDenyRules<T extends { name: string }>(
  tools: readonly T[],
  permissionContext: ToolPermissionContext,
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
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

### 4.2 工具编排 (`src/services/tools/toolOrchestration.ts`)

真实实现按"连续并发安全批"分区执行，且是流式 async generator（逐个 yield 消息更新，而非攒齐返回）：

```typescript
// src/services/tools/toolOrchestration.ts:19
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate, void, void> {
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(
    toolUseMessages,
    toolUseContext,
  )) {
    if (isConcurrencySafe) {
      // 连续的并发安全（只读）工具 → 并发批
      yield* runToolsConcurrently(blocks, ...)
    } else {
      // 非并发安全（写）工具 → 串行批
      yield* runToolsSerially(blocks, ...)
    }
  }
}
```

并发判定不依赖调用里写 `parallel` 字段，而是每个工具实现 `isConcurrencySafe(input)`（如 Read/Glob/Grep 返回 true；Bash/Edit/Write 返回 false），见 `partitionToolCalls`（`toolOrchestration.ts:100`）。并发上限为 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`（默认 10）。

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

使用 Ripgrep 进行内容搜索（真实输入参数以 `src/tools/GrepTool/prompt.ts` 为准，支持正则、文件过滤、多路径、输出模式等字段）：

```typescript
const GrepTool: Tool = {
  name: 'Grep',
  description: 'Search file contents using ripgrep',
  input_schema: {
    type: 'object',
    properties: {
      pattern: { type: 'string', description: '正则表达式' },
      path: { type: 'array', items: { type: 'string' }, description: '搜索路径（可为多个）' },
      glob: { type: 'string', description: '文件名过滤' },
      output_mode: { type: 'string', enum: ['content', 'files_with_matches', 'count'] },
      // ...
    },
    required: ['pattern'],
  },
  // call() 内部 spawn 'rg'，解析输出后返回匹配结果
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

MCP 工具是对 Model Context Protocol 服务器工具的封装。命名格式为**双下划线** `mcp__{serverName}__{toolName}`（见 `src/services/mcp/mcpStringUtils.ts:40`）：

```typescript
// src/tools/MCPTool/MCPTool.ts（结构示意，字段以源码为准）
class MCPTool {
  name = `mcp__${serverName}__${mcpTool.name}`   // 双下划线分隔
  description = mcpTool.description
  input_schema = mcpTool.inputSchema

  async call(input, context) {
    // 通过 MCPConnectionManager 获取对应 server 的 client
    const client = getMcpClientForServer(this.serverName)
    const result = await client.callTool({
      name: this.mcpTool.name,
      arguments: input,
    })
    return formatToolResult(result)
  }
}
```

## 7. 工具权限

### 7.1 权限模式

真实模式集合见 `src/types/permissions.ts:16-38`（外部可见模式 + 门控内部模式）：

| 模式 | 描述 | 行为 |
|------|------|------|
| `default` | 默认模式 | 需要权限的操作询问用户 |
| `acceptEdits` | 接受编辑 | 工作目录内的文件编辑自动放行 |
| `plan` | 计划模式 | 只读探索，等价于受限执行 |
| `bypassPermissions` | 绕过模式 | 跳过大部分权限检查（安全规则仍执行） |
| `dontAsk` | 不询问 | 所有 'ask' 转为 'deny'（非交互场景） |
| `auto` | 自动模式 | AI 分类器自动决策（`TRANSCRIPT_CLASSIFIER` 特性门控） |

### 7.2 权限请求

权限决策的完整管线（规则检查 → 模式决策 → 分类器 → 4 路竞争）在第 10 章展开。

### 7.3 权限规则存储

权限规则存储在 `~/.claude/settings.json` / 项目 `.claude/settings.json` 的 `permissions` 键（真实 schema 见 `src/utils/settings/types.ts:42`）：

```json
{
  "permissions": {
    "allow": ["Bash(git status:*)", "Read(~/.zshrc)"],
    "deny": ["Bash(rm:*)"],
    "ask": ["Bash(npm publish:*)"],
    "defaultMode": "acceptEdits"
  }
}
```

规则格式为 `ToolName`、`ToolName(specifier)` 或 MCP 的 `mcp__server`、`mcp__server__tool`。

## 8. 工具调用协议

### 8.1 Anthropic API 格式

Anthropic API 使用 `tool_calls` 格式：

```typescript
// API 请求
{
  model: 'claude-sonnet-4-6',   // 示例；别名见 src/utils/model/aliases.ts
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

CLI 本身没有 `.claude/tools/` 用户自定义工具目录——扩展 Agent 能力的三条真实途径是：

1. **MCP 服务器**：在 `.mcp.json`（项目）或 `~/.claude.json`（用户）配置，工具以 `mcp__server__tool` 暴露
2. **Skills**：`.claude/skills/` 下 Markdown + frontmatter，可带 `allowed-tools`（见第 12 章）
3. **插件**：`~/.claude/plugins/<name>/` 提供 skills/mcp/hooks/commands 组合（见第 16 章）

## 10. 调试工具

### 10.1 工具调试

调试相关开关（部分为实验性，随版本演进，以 `--help`/源码为准）：

```bash
# 启用调试日志输出到 stderr
claude-haha --debug-to-stderr

# 写入指定调试日志文件
claude-haha --debug-file /tmp/cc-debug.log

# MCP 调试模式
claude-haha --mcp-debug
```

### 10.2 工具追踪

遥测事件贯穿工具生命周期（`src/services/tools/toolExecution.ts` 内置）：工具开始/结束/失败都会打 `logEvent`，配合 `--debug` 可观察每次工具调用的权限决策、hook 执行与耗时。
