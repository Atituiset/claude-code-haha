# 工具系统深度解析

> 本文从源码层面详解 Claude Code 的 55+ Agent 工具，包括工具定义、注册、并发编排、权限集成和每个核心工具的实现原理。

## 1. 工具定义 (Tool.ts)

### 1.1 核心类型

```typescript
// src/Tool.ts (792 行)

// 工具权限上下文
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode                    // 'default' | 'auto' | 'bypassPermissions' | 'plan' | 'dontAsk'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource    // 按来源的 allow 规则
  alwaysDenyRules: ToolPermissionRulesBySource     // 按来源的 deny 规则
  alwaysAskRules: ToolPermissionRulesBySource      // 按来源的 ask 规则
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean           // headless/async 时
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>

// 工具使用上下文
type ToolUseContext = {
  options: {
    commands: Command[]                   // 可用命令
    debug: boolean
    mainLoopModel: string                 // 主循环模型
    tools: Tool[]                         // 可用工具
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPClientMap              // MCP 客户端
    mcpResources: MCPResourceMap          // MCP 资源
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinition[]
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    querySource?: string
    refreshTools?: () => Promise<void>
  }
  abortController: AbortController        // 中止控制
  readFileState: FileStateCache           // 文件读取状态缓存
  getAppState(): AppState                 // 读取应用状态
  setAppState(f: (prev: AppState) => AppState): void
  handleElicitation?: (...) => Promise<ElicitationResult>
  addNotification?: (notification) => void
  localDenialTracking?: DenialTrackingState
  // ... 更多字段
}
```

### 1.2 工具接口

```typescript
interface Tool {
  name: string                           // 工具名（如 "Bash", "Edit"）
  description: string                    // 工具描述（发送给 LLM）
  input_schema: ToolInputJSONSchema      // JSON Schema 输入定义
  annotations?: ToolAnnotation           // 工具注解

  // 权限检查方法
  checkPermissions?(input, context): ToolPermissionResult

  // 执行方法
  call(input, context): Promise<ToolResult>

  // 自动分类器输入
  toAutoClassifierInput?(input): string
}

interface ToolAnnotation {
  title?: string                         // 显示标题
  description?: string                   // UI 描述
  kind?: 'information' | 'action' | 'warning'
  bypassA11y?: boolean
  skipConfirmation?: boolean
}
```

## 2. 工具注册 (tools.ts)

### 2.1 注册流程

```typescript
// src/tools.ts
function getTools(permissionContext, workspace): Tool[] {
  const tools: Tool[] = []

  // 1. 基础工具 (13+ 核心 + 特性门控)
  tools.push(...getAllBaseTools())

  // 2. MCP 工具 (动态加载)
  tools.push(...getMcpTools())

  // 3. Swarm 工具 (如果启用)
  if (isAgentSwarmsEnabled()) {
    tools.push(...getSwarmTools())
  }

  return tools
}
```

### 2.2 工具清单 (55+)

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | FileReadTool, FileEditTool, FileWriteTool, NotebookEditTool, GlobTool, GrepTool | 读写和搜索文件 |
| **执行** | BashTool, PowerShellTool, TerminalCaptureTool, MonitorTool | 命令执行和监控 |
| **Agent/团队** | AgentTool, TeamCreateTool, TeamDeleteTool, SendMessageTool, ListPeersTool | 多 Agent 协作 |
| **任务系统** | TaskCreateTool, TaskGetTool, TaskListTool, TaskOutputTool, TaskStopTool, TaskUpdateTool | 后台任务管理 |
| **MCP** | MCPTool, McpAuthTool, ListMcpResourcesTool, ReadMcpResourceTool | MCP 协议工具 |
| **Web** | WebFetchTool, WebBrowserTool, WebSearchTool | 网络访问 |
| **知识** | ToolSearchTool, DiscoverSkillsTool, SkillTool, LSPTool | 工具和 Skill 发现 |
| **会话** | REPLTool, SnipTool, CtxInspectTool, BriefTool, EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool, VerifyPlanExecutionTool | 会话和工作流控制 |
| **通信** | AskUserQuestionTool, PushNotificationTool, SendUserFileTool | 用户交互 |
| **工作流** | WorkflowTool, ScheduleCronTool, SubscribePRTool, SuggestBackgroundPRTool | 自动化和 CI/CD |
| **配置** | ConfigTool, TodoWriteTool, SleepTool, SyntheticOutputTool, ReviewArtifactTool, RemoteTriggerTool, TungstenTool, OverflowTestTool | 配置和测试 |

## 3. 工具编排

### 3.1 并发分区

```typescript
// src/services/tools/toolOrchestration.ts

function partitionByConcurrency(blocks: ToolUseBlock[]): [safe[], exclusive[]] {
  // 并发安全工具：可以并行执行
  //   - FileReadTool, GlobTool, GrepTool, WebFetchTool 等
  // 独占工具：必须串行执行
  //   - BashTool, FileEditTool, FileWriteTool 等

  return blocks.reduce(([safe, excl], block) => {
    const tool = findTool(block.name)
    if (tool.exclusive) {
      excl.push(block)
    } else {
      safe.push(block)
    }
    return [safe, excl]
  }, [[], []])
}
```

### 3.2 流式工具执行

```typescript
// src/services/tools/StreamingToolExecutor.ts (~350 行)

// 在 LLM 还在流式输出时，已完成的 tool_use block 可以提前执行
// 而不是等待整个消息完成

class StreamingToolExecutor {
  private pendingExecutions: Set<Promise<ToolResult>> = new Set()

  onContentBlockStop(block: ContentBlock): void {
    if (block.type === 'tool_use' && this.isReadyToExecute(block)) {
      // 立即开始执行，不等消息完成
      this.pendingExecutions.add(
        this.runToolUse(block)
          .then(result => {
            this.pendingExecutions.delete(...)
            return result
          })
      )
    }
  }

  async waitForAll(): Promise<ToolResult[]> {
    return Promise.all([...this.pendingExecutions])
  }
}
```

## 4. 核心工具详解

### 4.1 AgentTool (最复杂)

```typescript
// src/tools/AgentTool/ 目录结构
AgentTool/
├── index.ts              // 工具定义
├── prompt.ts             // LLM 描述
├── builtInAgents.ts      // 6 个内置 Agent 类型
├── agentMemory.ts        // Agent 记忆管理
├── agentMemorySnapshot.ts // 记忆快照
├── resumeAgent.ts        // 恢复暂停的 Agent
├── forkSubagent.ts       // Fork 子 Agent
├── runAgent.ts           // Agent 执行逻辑
├── agentDisplay.ts       // Agent 显示
├── agentColorManager.ts  // Agent 颜色管理
└── agentToolUtils.ts     // 工具函数
```

**6 个内置 Agent 类型：**

| Agent | 功能 | 适用场景 |
|-------|------|----------|
| `generalPurpose` | 通用助手 | 默认 Agent |
| `claudeCodeGuide` | Claude Code 指南 | 使用帮助 |
| `explore` | 代码探索 | 快速搜索 |
| `plan` | 规划 | 制定计划 |
| `verification` | 验证 | 检查结果 |
| `statuslineSetup` | 状态栏设置 | 终端配置 |

**Agent 执行流程：**

```
AgentTool.call(input)
  → resolveAgentDefinition(input.agent)   // 解析 Agent 定义
  → createAgentSession(agentDef)           // 创建子 Agent 会话
  → runAgent(prompt, session)              // 执行 Agent
  → 返回 Agent 输出作为工具结果
```

**Agent 可用工具限制**（真实集合，`src/constants/tools.ts:36-72`）：

```typescript
// 所有子 Agent 均不可用（ALL_AGENT_DISALLOWED_TOOLS）
TASK_OUTPUT, EXIT_PLAN_MODE_V2, ENTER_PLAN_MODE,
AGENT,        // ant 构建例外：允许嵌套 Agent
ASK_USER_QUESTION, TASK_STOP,
WORKFLOW      // feature 门控，防递归执行

// 自定义 Agent 额外禁用（CUSTOM_AGENT_DISALLOWED_TOOLS = 上面 + 更多）

// 异步 Agent 允许的工具（ASYNC_AGENT_ALLOWED_TOOLS，白名单模式）
FILE_READ, WEB_SEARCH, TODO_WRITE, GREP, WEB_FETCH,
GLOB, SHELL_*(Bash/PowerShell 等), FILE_EDIT, FILE_WRITE, NOTEBOOK_EDIT,
SKILL, SYNTHETIC_OUTPUT, TOOL_SEARCH, ENTER_WORKTREE, EXIT_WORKTREE
```

### 4.2 BashTool

```typescript
// src/tools/BashTool/

// 执行 Shell 命令
// 特点：
// - 支持沙箱执行 (@anthropic-ai/sandbox-runtime)
// - 超时控制 (默认 60000ms)
// - 工作目录限制
// - 环境变量清理
// - 实时输出流
// - 后台进程管理
// - PowerShell 支持 (Windows)
```

### 4.3 FileEditTool

```typescript
// src/tools/FileEditTool/

// "保持目标"语义的文件编辑
// 支持的操作：
// - 单行替换 (old_string → new_string)
// - 多行替换
// - 正则替换
// - 干运行预览 (dry_run: true)
// - 多重编辑 (MultiEditTool)
// - Notebook 编辑 (NotebookEditTool)
// - 补丁应用 (apply_patch)

// 安全机制：
// - old_string 唯一性检查
// - 文件备份 (用于会话回退)
// - 路径安全检查
```

### 4.4 TaskTool 系统

```typescript
// 6 个任务管理工具

TaskCreateTool   // 创建后台任务
TaskGetTool       // 获取任务详情
TaskListTool      // 列出所有任务
TaskOutputTool    // 获取任务输出
TaskStopTool      // 停止任务
TaskUpdateTool    // 更新任务状态

// 后台任务允许 Agent 在后台执行长时间操作
// 主线程可以继续处理其他请求
```

### 4.5 WebFetchTool / WebSearchTool

```typescript
// WebFetchTool: 获取网页内容
// - Turndown 库将 HTML 转为 Markdown
// - XSS 清理
// - 图片内联 (base64)
// - 超时控制

// WebSearchTool: 网络搜索
// - 支持多个搜索后端
// - 结果排序和过滤
```

### 4.6 VimTool

```typescript
// src/vim/ (5 个模块)
// 纯状态机实现的 Vim 模拟器

// 支持：
// - Normal/Insert/Visual 模式
// - 基本移动 (h,j,k,l,w,b,e)
// - 编辑操作 (dd,yy,p,u)
// - 搜索 (/pattern)
// - 不支持 ex 命令和宏
```

## 5. 工具权限集成

### 5.1 工具级权限检查

```typescript
// 每个工具可以定义自己的 checkPermissions 方法
interface Tool {
  checkPermissions?(
    input: ToolInput,
    context: ToolPermissionContext,
  ): ToolPermissionResult
}

// 结果类型：
type ToolPermissionResult =
  | { result: 'allow' }                   // 允许
  | { result: 'deny', reason: string }    // 拒绝
  | { result: 'ask', message: string }    // 询问用户
  | { result: 'passthrough' }             // 透传到模式决策

// 示例：BashTool 的 checkPermissions
function checkBashPermissions(input, context) {
  // 1. 检查命令是否在 deny 列表
  // 2. 检查沙箱模式
  // 3. 检查 acceptEdits 模式
  // 4. 返回相应的权限结果
}
```

### 5.2 自动分类器输入

```typescript
// 每个工具定义 toAutoClassifierInput 方法
// 用于 AI 分类器判断是否安全执行

interface Tool {
  toAutoClassifierInput?(input: ToolInput): string
}

// 示例：
// BashTool.toAutoClassifierInput({ command: "ls -la" }) → "Bash ls -la"
// FileEditTool.toAutoClassifierInput({ file_path: "/tmp/test.ts" }) → "Edit /tmp/test.ts"
// MCPTool.toAutoClassifierInput({ ... }) → "mcp__server__tool ..."
```

## 6. 工具调用协议

### 6.1 Anthropic API 格式

```json
// 请求中的工具定义
{
  "tools": [
    {
      "name": "Bash",
      "description": "Execute shell commands...",
      "input_schema": {
        "type": "object",
        "properties": {
          "command": { "type": "string" },
          "timeout": { "type": "number", "default": 60000 }
        },
        "required": ["command"]
      }
    }
  ]
}

// 响应中的工具调用
{
  "content": [
    { "type": "text", "text": "I'll list the files for you." },
    {
      "type": "tool_use",
      "id": "toolu_01ABC",
      "name": "Bash",
      "input": { "command": "ls -la" }
    }
  ],
  "stop_reason": "tool_use"
}

// 工具结果返回
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01ABC",
      "content": "total 32\ndrwxr-xr-x 4 user user 4096 ...",
      "is_error": false
    }
  ]
}
```

## 7. 如何开发自己的 Agent 工具

### 7.1 最小工具实现

```typescript
const myTool: Tool = {
  name: 'MyTool',
  description: 'A custom agent tool',
  input_schema: {
    type: 'object',
    properties: {
      input: { type: 'string' }
    },
    required: ['input'],
  },
  async call(input, context) {
    const result = await doSomething(input.input)
    return { type: 'text', text: result }
  },
}
```

### 7.2 带权限检查的工具

```typescript
const myTool: Tool = {
  name: 'MyDangerousTool',
  description: 'A tool that needs permission',
  input_schema: { ... },
  checkPermissions(input, context) {
    if (input.action === 'delete') {
      return { result: 'ask', message: 'This will delete data. Allow?' }
    }
    return { result: 'allow' }
  },
  toAutoClassifierInput(input) {
    return `MyDangerousTool ${input.action}`
  },
  async call(input, context) {
    // ...
  },
}
```

### 7.3 工具开发最佳实践

| 实践 | 原因 |
|------|------|
| 定义 `checkPermissions` | 安全第一 |
| 定义 `toAutoClassifierInput` | 支持 auto 模式 |
| 使用 `AbortController` | 支持取消 |
| 限制输出长度 | 避免 token 浪费 |
| 处理错误返回 `is_error: true` | 让 LLM 知道失败原因 |
| 标记 `exclusive` | 避免并发冲突 |
| 备份修改的文件 | 支持会话回退 |

## 8. 关键文件索引

| 文件 | 功能 |
|------|------|
| `src/Tool.ts` | Tool 类型定义、ToolPermissionContext、ToolUseContext |
| `src/tools.ts` | 工具注册中心 |
| `src/services/tools/toolOrchestration.ts` | 并发/独占分区执行 |
| `src/services/tools/toolExecution.ts` | 单工具执行（权限+hooks+遥测） |
| `src/services/tools/StreamingToolExecutor.ts` | 流式工具提前执行 |
| `src/tools/AgentTool/` | 子 Agent 工具（最复杂） |
| `src/tools/BashTool/` | Shell 执行 |
| `src/tools/FileReadTool/` | 文件读取 |
| `src/tools/FileEditTool/` | 文件编辑 |
| `src/tools/FileWriteTool/` | 文件写入 |
| `src/tools/GlobTool/` | 文件模式匹配 |
| `src/tools/GrepTool/` | 内容搜索 |
| `src/tools/WebFetchTool/` | 网页获取 |
| `src/tools/WebSearchTool/` | 网络搜索 |
| `src/tools/TaskTool/` | 任务系统 (6 个工具) |
| `src/tools/MCPTool/` | MCP 工具封装 |
| `src/tools/SkillTool/` | Skill 调用 |
| `src/tools/AskUserQuestionTool/` | 用户交互 |
| `src/tools/TeamCreateTool/` | 团队创建 |
| `src/tools/SendMessageTool/` | Agent 间通信 |
| `src/constants/tools.ts` | 工具名常量和限制列表 |
