# API 层与查询执行流程详解

## 1. 概述

Claude Code 通过 HTTP API 与 Anthropic 的 Claude 模型通信。API 层封装了所有 API 调用，包括请求构建、重试机制、错误处理等。

## 2. API 客户端 (`src/services/api/`)

### 2.1 客户端初始化

```typescript
// src/services/api/client.ts
import Anthropic from '@anthropic-ai/sdk'

export function createApiClient(config: ApiClientConfig): Anthropic {
  return new Anthropic({
    apiKey: config.apiKey,
    baseURL: config.baseURL,
    timeout: config.timeout ?? 600_000,
    maxRetries: config.maxRetries ?? 3,
  })
}
```

### 2.2 配置来源

```typescript
function getApiConfig(): ApiClientConfig {
  return {
    apiKey: process.env.ANTHROPIC_API_KEY ?? process.env.ANTHROPIC_AUTH_TOKEN!,
    baseURL: process.env.ANTHROPIC_BASE_URL ?? 'https://api.anthropic.com',
    timeout: parseInt(process.env.API_TIMEOUT_MS ?? '600000', 10),
    maxRetries: 3,
  }
}
```

## 3. 查询执行 (`src/query.ts`)

### 3.1 主查询函数

```typescript
export async function executeQuery(
  params: QueryParams,
): Promise<QueryResult> {
  const { messages, tools, systemPrompt, model, options } = params
  
  // 1. 准备 API 请求
  const request: MessageCreateParams = {
    model: model ?? getMainLoopModel(),
    messages: buildMessages(messages),
    system: buildSystemPrompt(systemPrompt),
    tools: buildTools(tools),
    max_tokens: options?.maxTokens ?? 4096,
  }
  
  // 2. 执行 API 调用
  const response = await apiClient.messages.create(request)
  
  // 3. 处理响应
  const result = processApiResponse(response)
  
  // 4. 如有工具调用，执行工具
  if (result.toolCalls.length > 0) {
    const toolResults = await executeToolCalls(result.toolCalls, tools)
    return {
      content: result.content,
      toolResults,
      stopReason: result.stopReason,
    }
  }
  
  return result
}
```

### 3.2 消息构建

```typescript
function buildMessages(conversation: Message[]): MessageParam[] {
  return conversation.map(msg => ({
    role: msg.role,
    content: buildMessageContent(msg),
  }))
}

function buildMessageContent(message: Message): ContentBlockParam {
  if (typeof message.content === 'string') {
    return message.content
  }
  
  // 处理多模态内容
  return message.content.map(block => {
    switch (block.type) {
      case 'text':
        return block.text
      
      case 'image':
        return {
          type: 'image',
          source: {
            type: 'base64',
            media_type: block.mimeType,
            data: block.data,
          },
        }
      
      default:
        return block
    }
  })
}
```

### 3.3 系统提示词构建

```typescript
async function buildSystemPrompt(
  basePrompt: string | null,
): Promise<string | null> {
  const sections: string[] = []
  
  // 1. 基础系统提示词
  const baseSystem = await getSystemPrompt()
  sections.push(baseSystem)
  
  // 2. 项目上下文
  const projectContext = await getProjectContext()
  sections.push(projectContext)
  
  // 3. 用户提供的补充
  if (basePrompt) {
    sections.push(basePrompt)
  }
  
  return sections.join('\n\n')
}
```

## 4. 工具调用执行 (`src/services/tools/toolOrchestration.js`)

### 4.1 工具编排器

```typescript
export async function runTools(
  toolInputs: ToolInput[],
  context: ExecutionContext,
): Promise<ToolResult[]> {
  const results: ToolResult[] = []
  
  // 1. 按并行组分割工具调用
  const parallelGroups = groupByParallelism(toolInputs)
  
  for (const group of parallelGroups) {
    if (group.parallel) {
      // 并行执行
      const groupResults = await Promise.all(
        group.inputs.map(input => executeSingleTool(input, context))
      )
      results.push(...groupResults)
    } else {
      // 串行执行
      for (const input of group.inputs) {
        const result = await executeSingleTool(input, context)
        results.push(result)
      }
    }
  }
  
  return results
}
```

### 4.2 单个工具执行

```typescript
async function executeSingleTool(
  input: ToolInput,
  context: ExecutionContext,
): Promise<ToolResult> {
  const tool = findTool(input.name, context.tools)
  
  if (!tool) {
    return {
      tool_use_id: input.tool_use_id,
      content: `Tool not found: ${input.name}`,
      is_error: true,
    }
  }
  
  // 检查权限
  if (tool.requiresPermissions) {
    const granted = await checkPermission(tool.name, context.permissionContext)
    if (!granted) {
      return {
        tool_use_id: input.tool_use_id,
        content: `Permission denied for tool: ${input.name}`,
        is_error: true,
      }
    }
  }
  
  try {
    const result = await tool.handler(input, context)
    return result
  } catch (error) {
    return {
      tool_use_id: input.tool_use_id,
      content: `Error executing ${input.name}: ${error.message}`,
      is_error: true,
    }
  }
}
```

### 4.3 工具调用分组

```typescript
interface ToolGroup {
  parallel: boolean
  inputs: ToolInput[]
}

function groupByParallelism(inputs: ToolInput[]): ToolGroup[] {
  // 1. 分离需要并行和需要串行的工具
  const parallelInputs: ToolInput[] = []
  const serialInputs: ToolInput[] = []
  
  for (const input of inputs) {
    if (input.parallel && !input.requiresSequentialExecution) {
      parallelInputs.push(input)
    } else {
      serialInputs.push(input)
    }
  }
  
  const groups: ToolGroup[] = []
  
  if (parallelInputs.length > 0) {
    groups.push({ parallel: true, inputs: parallelInputs })
  }
  
  for (const input of serialInputs) {
    groups.push({ parallel: false, inputs: [input] })
  }
  
  return groups
}
```

## 5. 推测执行 (Speculation)

### 5.1 概念

推测执行允许 Claude Code 在用户确认前"预执行"工具调用，提供更流畅的体验：

```
User: "Create a new file"
         │
         ▼
┌─────────────────┐
│  API Response   │  ← 模型返回工具调用
│  (speculative)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Speculation    │  ← 显示推测状态
│  State         │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Execute Tool   │  ← 执行工具（可隐藏）
│  (in background)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  User Confirm   │  ← 显示结果
│  (or cancel)    │
└─────────────────┘
```

### 5.2 推测状态

```typescript
interface Speculation {
  toolName: string
  toolParams: Record<string, unknown>
  status: 'pending' | 'executing' | 'completed' | 'cancelled'
  result?: ToolResult
  startTime: number
}
```

### 5.3 推测执行流程

```typescript
async function executeWithSpeculation(
  toolInputs: ToolInput[],
  context: ExecutionContext,
): Promise<QueryResult> {
  const speculation: Speculation[] = toolInputs.map(input => ({
    toolName: input.name,
    toolParams: input.params,
    status: 'pending',
    startTime: Date.now(),
  }))
  
  // 1. 更新状态为推测中
  updateAppState({ speculation })
  
  // 2. 后台执行工具
  const toolResults = await runTools(toolInputs, context)
  
  // 3. 清除推测状态
  updateAppState({ speculation: null })
  
  // 4. 返回结果
  return {
    toolResults,
    speculation: undefined,
  }
}
```

## 6. 重试机制 (`src/services/api/withRetry.ts`)

### 6.1 重试策略

```typescript
interface RetryConfig {
  maxRetries: number
  initialDelayMs: number
  maxDelayMs: number
  backoffMultiplier: number
  retryableErrors: string[]
}

const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxRetries: 3,
  initialDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
  retryableErrors: [
    'ECONNRESET',
    'ETIMEDOUT',
    '429',
    '500',
    '502',
    '503',
    '504',
  ],
}
```

### 6.2 重试实现

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  config: RetryConfig = DEFAULT_RETRY_CONFIG,
): Promise<T> {
  let lastError: Error | undefined
  let delay = config.initialDelayMs
  
  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error
      
      if (!isRetryable(error, config.retryableErrors)) {
        throw error
      }
      
      if (attempt < config.maxRetries) {
        await sleep(delay)
        delay = Math.min(delay * config.backoffMultiplier, config.maxDelayMs)
      }
    }
  }
  
  throw lastError
}
```

### 6.3 错误判断

```typescript
function isRetryable(error: Error, retryableErrors: string[]): boolean {
  // 1. 检查错误码
  if (retryableErrors.includes(getErrorCode(error))) {
    return true
  }
  
  // 2. 检查 HTTP 状态码
  if (error instanceof ApiError) {
    if (retryableErrors.includes(String(error.status))) {
      return true
    }
  }
  
  // 3. 检查速率限制
  if (error instanceof RateLimitError) {
    return true
  }
  
  return false
}
```

## 7. 错误处理 (`src/services/api/errors.ts`)

### 7.1 错误类型

```typescript
class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public code?: string,
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

class RateLimitError extends ApiError {
  constructor(
    public retryAfterMs: number,
  ) {
    super(429, 'Rate limit exceeded')
    this.name = 'RateLimitError'
  }
}

class AuthenticationError extends ApiError {
  constructor() {
    super(401, 'Authentication failed')
    this.name = 'AuthenticationError'
  }
}

class ValidationError extends ApiError {
  constructor(
    public validationErrors: ValidationError[],
  ) {
    super(400, 'Invalid request')
    this.name = 'ValidationError'
  }
}
```

### 7.2 错误处理流程

```typescript
async function handleApiError(
  error: Error,
  context: ErrorContext,
): Promise<ErrorResult> {
  if (error instanceof AuthenticationError) {
    // 清除认证信息，提示重新登录
    clearAuthTokens()
    return {
      type: 'auth_error',
      message: 'Please login again: /login',
      action: 'reauthenticate',
    }
  }
  
  if (error instanceof RateLimitError) {
    // 显示速率限制信息
    return {
      type: 'rate_limit',
      message: `Rate limit exceeded. Retry after ${error.retryAfterMs / 1000}s`,
      action: 'wait',
      retryAfter: error.retryAfterMs,
    }
  }
  
  if (error instanceof ValidationError) {
    // 显示验证错误详情
    return {
      type: 'validation_error',
      message: formatValidationErrors(error.validationErrors),
      action: 'fix_input',
    }
  }
  
  // 未知错误
  logError(error)
  return {
    type: 'unknown_error',
    message: error.message,
    action: 'retry',
  }
}
```

## 8. 上下文窗口管理

### 8.1 上下文窗口限制

```typescript
interface ContextWindow {
  model: string
  maxTokens: number
  currentTokens: number
  remainingTokens: number
}

function getContextWindow(model: string): ContextWindow {
  const limits = {
    'claude-opus-4-7-20251120': { maxTokens: 200000 },
    'claude-sonnet-4-7-20251120': { maxTokens: 200000 },
    'claude-haiku-4-7-20251120': { maxTokens: 200000 },
  }
  
  return limits[model] ?? { maxTokens: 100000 }
}
```

### 8.2 上下文压缩

```typescript
interface CompactOptions {
  targetTokens: number
  preserveRecent: number
  strategy: 'summary' | 'truncate' | 'mixed'
}

async function compactContext(
  messages: Message[],
  options: CompactOptions,
): Promise<Message[]> {
  const currentTokens = await countTokens(messages)
  
  if (currentTokens <= options.targetTokens) {
    return messages
  }
  
  // 1. 保留最近的 N 条消息
  const recentMessages = messages.slice(-options.preserveRecent)
  
  // 2. 压缩旧消息
  const olderMessages = messages.slice(0, -options.preserveRecent)
  const compactedOlder = await compactMessages(olderMessages, {
    targetTokens: options.targetTokens - countTokens(recentMessages),
    strategy: options.strategy,
  })
  
  return [...compactedOlder, ...recentMessages]
}

async function compactMessages(
  messages: Message[],
  options: { targetTokens: number; strategy: string },
): Promise<Message[]> {
  switch (options.strategy) {
    case 'summary':
      return await summarizeMessages(messages, options.targetTokens)
    
    case 'truncate':
      return truncateMessages(messages, options.targetTokens)
    
    case 'mixed':
      return await mixedCompact(messages, options.targetTokens)
  }
}
```

## 9. 流式响应 (`src/services/api/stream.ts`)

### 9.1 流式处理

```typescript
async function* streamQuery(
  request: MessageCreateParams,
): AsyncGenerator<StreamEvent> {
  const response = await apiClient.messages.stream(request)
  
  for await (const event of response) {
    yield parseStreamEvent(event)
  }
}

type StreamEvent =
  | { type: 'content_block_delta'; content: string }
  | { type: 'content_block_stop' }
  | { type: 'message_delta'; usage: Usage }
  | { type: 'error'; error: Error }
```

### 9.2 流式事件处理

```typescript
async function handleStream(
  events: AsyncGenerator<StreamEvent>,
  context: StreamContext,
): Promise<void> {
  let currentBlock: string = ''
  
  for await (const event of events) {
    switch (event.type) {
      case 'content_block_delta':
        currentBlock += event.content
        // 更新 UI
        updateStreamingOutput(currentBlock)
        break
      
      case 'content_block_stop':
        // 完成当前块
        finalizeContentBlock(currentBlock)
        currentBlock = ''
        break
      
      case 'message_delta':
        // 更新使用量统计
        updateUsageStats(event.usage)
        break
      
      case 'error':
        // 处理错误
        handleStreamError(event.error)
        break
    }
  }
}
```

## 10. 请求追踪

### 10.1 请求 ID 生成

```typescript
function generateRequestId(): string {
  const timestamp = Date.now().toString(36)
  const random = Math.random().toString(36).substring(2, 10)
  return `req_${timestamp}_${random}`
}
```

### 10.2 请求日志

```typescript
interface RequestLog {
  requestId: string
  model: string
  messageCount: number
  toolCallCount: number
  startTime: number
  endTime?: number
  duration?: number
  status: 'pending' | 'completed' | 'failed'
  error?: string
}

function logRequest(request: RequestLog): void {
  const logsDir = path.join(getSessionDir(), 'logs')
  ensureDirExists(logsDir)
  
  const logFile = path.join(logsDir, 'requests.jsonl')
  appendFileSync(logFile, JSON.stringify(request) + '\n')
}
```

## 11. 查询配置 (`src/query/config.ts`)

### 11.1 查询配置构建

```typescript
interface QueryConfig {
  model: string
  temperature: number
  topP: number
  topK: number
  maxTokens: number
  stopSequences: string[]
  tools: Tool[]
}

export function buildQueryConfig(
  options: QueryOptions,
): QueryConfig {
  return {
    model: options.model ?? getMainLoopModel(),
    temperature: options.temperature ?? 1.0,
    topP: options.topP ?? 1.0,
    topK: options.topK ?? 250,
    maxTokens: options.maxTokens ?? 4096,
    stopSequences: options.stopSequences ?? [],
    tools: options.tools ?? [],
  }
}
```

### 11.2 模型选择逻辑

```typescript
function selectModel(context: ModelSelectionContext): string {
  // 1. 优先使用用户指定的模型
  if (context.userSpecifiedModel) {
    return context.userSpecifiedModel
  }
  
  // 2. 根据任务类型选择
  if (context.taskType === 'simple') {
    return getDefaultHaikuModel()
  }
  
  if (context.taskType === 'complex') {
    return getDefaultOpusModel()
  }
  
  // 3. 默认使用 Sonnet
  return getDefaultSonnetModel()
}
```
