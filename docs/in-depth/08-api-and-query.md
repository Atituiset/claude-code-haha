# API 层与查询执行流程详解

## 1. 概述

Claude Code 通过 HTTP API 与 Anthropic 的 Claude 模型通信。API 层封装了所有 API 调用，包括请求构建、重试机制、错误处理等。

## 2. API 客户端 (`src/services/api/`)

### 2.1 客户端初始化

真实入口是 `getAnthropicClient()`（`src/services/api/client.ts:112`），按 Provider 分支构造（详见第 11 章 Provider 路由）：

```typescript
export async function getAnthropicClient({ apiKey }: { apiKey?: string } = {}): Promise<Anthropic> {
  // 依次检查：CLAUDE_CODE_USE_BEDROCK → Bedrock SDK
  //           CLAUDE_CODE_USE_FOUNDRY → Foundry（Azure）
  //           CLAUDE_CODE_USE_VERTEX  → Vertex（GoogleAuth）
  //           OpenAI Responses 模型 → OpenAI 兼容客户端
  //           默认 → new Anthropic({ apiKey: resolveAnthropicClientApiKey(...) })
  // timeout: API_TIMEOUT_MS（默认 600s）
}
```

### 2.2 认证来源（`client.ts:96`）

```typescript
// ANTHROPIC_AUTH_TOKEN 存在且无显式 key/API_KEY → 走 auth token 路径
// 显式 apiKey > ANTHROPIC_API_KEY > OAuth token（Claude.ai 订阅）
//   > apiKeyHelper（settings 配置的辅助程序）
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

## 5. 推测执行 (Speculation / Prompt Suggestion)

### 5.1 真实概念

推测执行**不是**"用户确认前预执行工具调用"，而是 **Prompt Suggestion 机制**（`src/services/PromptSuggestion/speculation.ts`）：在用户输入前，对预测的下一轮提示词跑一个完整的后台 Agent Loop（speculative decoding 的 Agent 版）：

```
上一轮对话结束
         │
         ▼
预测用户下一输入（提示建议）
         │
         ▼
后台 speculative query loop（受限工具集）
  - 只读工具直接执行
  - 写工具（Edit/Write/NotebookEdit）写入 overlay 目录
         │
         ▼
用户实际输入 == 建议？
  ├─ 是 → 接受：overlay 写入复制回主工作区，跳过整轮计算
  └─ 否 → 丢弃：删除 overlay，正常执行
```

### 5.2 关键约束（`speculation.ts:15-28`）

```typescript
const MAX_SPECULATION_TURNS = 20        // 最多 20 轮
const MAX_SPECULATION_MESSAGES = 100

const WRITE_TOOLS = new Set(['Edit', 'Write', 'NotebookEdit'])
// 写操作进入 overlay（临时目录），接受前不影响真实工作区
const SAFE_READ_ONLY_TOOLS = new Set([
  'Read', 'Glob', 'Grep', 'ToolSearch', 'LSP', 'TaskGet', 'TaskList',
])
```

### 5.3 状态机（`src/state/AppStateStore.ts:58`）

```typescript
export type SpeculationState =
  | { status: 'idle' }          // 空闲
  | { status: 'active', ... }   // 建议/执行/等待接受
  | { status: '...', ... }      // 详见源码
// 会话累计节省时间记录于 speculationSessionTimeSavedMs
```

## 6. 重试机制 (`src/services/api/withRetry.ts`)

真实的 `withRetry` 是一个高阶 async generator（逐事件透传流式响应，同时处理重试、认证降级、Fast Mode 冷却等），核心要点（`withRetry.ts:50-730`）：

### 6.1 重试判定要点

- **重试源**：SDK 的 `APIConnectionError`、`APIError`（429/500 级）、`overloaded_error`（含 529，通过消息内容探测，`withRetry.ts:615-622`）
- **认证特殊路径**：401 触发 `handleOAuth401Error`（OAuth token 刷新），AWS/GCP 凭证错误有独立清理逻辑（`clearAwsCredentialsCache` 等）
- **Fast Mode 冷却**：429/overload 可能触发 fast mode cooldown 而非直接重试
- **持久模式**：429/529 始终可重试并绕过订阅者门控（`withRetry.ts:702`）
- 重试间退避为指数增长（`sleep`），并伴随 `tengu_api_*` 遥测事件（如 `tengu_api_custom_529_overloaded_error`）

### 6.2 结构示意

```typescript
async function* withRetry(messages, fn, config) {
  // for attempt in retries:
  //   try:
  //     for await (const event of await fn()) yield event   // 透传流
  //   catch (error):
  //     if (!isRetryableError(error)) throw
  //     handleOAuth401 / credentials 清理 / fastMode cooldown
  //     await sleep(backoff)
}
```

> 注意：不存在文档早期版本虚构的 `DEFAULT_RETRY_CONFIG`/`retryableErrors` 字符串数组——可重试性由 SDK 错误类型与状态码分支逻辑决定。

## 7. 错误处理 (`src/services/api/errors.ts`)

### 7.1 真实结构

错误类型直接复用 SDK 的 `APIError`/`APIConnectionError`/`APIConnectionTimeoutError`，`errors.ts` 不定义自定义错误类层级，而是提供**面向用户的错误消息构造**（`REPEATED_529_ERROR_MESSAGE` 等）与错误→AssistantMessage 转换：

```typescript
// src/services/api/errors.ts
import { APIConnectionError, APIConnectionTimeoutError, APIError }
  from '@anthropic-ai/sdk'

// 关键职责：
// - 401 → OAuth/订阅诊断（getClaudeAIOAuthTokens / isClaudeAISubscriber）
// - 413/429/500/529 → 人类可读的重试/降级提示
// - 图片/PDF 超限（ImageResizeError、API_PDF_MAX_PAGES）→ 指导性错误
// - 生成 createAssistantAPIErrorMessage 消息注入对话
```

### 7.2 认证降级路径

认证错误（401）在 `withRetry.ts` 中走专门路径：`handleOAuth401Error` 尝试刷新 token；失败后按订阅状态（Claude.ai subscriber / enterprise）给出相应提示；AWS/GCP 凭证缓存被清理以在重试时重新获取。

## 8. 上下文窗口管理

### 8.1 上下文窗口（真实来源 `src/utils/model/modelContextWindows.ts`）

```typescript
const DIRECT_MODEL_CONTEXT_WINDOWS: Record<string, number> = {
  'claude-opus-4-7': 1_000_000,
  'claude-sonnet-4-6': 200_000,
  'claude-haiku-4-5': 200_000,
  'deepseek-v4-pro': 1_000_000,
  'kimi-k2.6': 262_144,
  'glm-5': 200_000,
  // ...
}
// 另有 PATTERN_MODEL_CONTEXT_WINDOWS 按 provider 前缀匹配
// 可用 CLAUDE_CODE_MODEL_CONTEXT_WINDOWS 环境变量覆盖
// 上限 10M、下限 16K（MODEL_CONTEXT_WINDOW_MIN/MAX）
```

### 8.2 多级压缩（真实层级）

压缩不是 `strategy: 'summary'|'truncate'|'mixed'` 单一策略，而是**四级递进**（`src/query.ts:397-468`、`src/services/compact/autoCompact.ts`）：

| 层级 | 触发时机 | 做什么 |
|------|----------|--------|
| Micro-compact（partialCompact） | 每轮循环前检查 | 对最旧消息段落做部分压缩 |
| Tool-call collapse | autocompact 之前 | 折叠冗余 tool_use/tool_result 对（若已低于阈值则 autocompact 为 no-op） |
| Auto-compact | token 阈值触发 | 调 LLM 生成对话摘要替换旧消息，保留最近上下文 |
| Emergency/reactive compact | API 报 prompt-too-long | 立即压缩后重试 |

`deps.microcompact` / `deps.autocompact` 通过 `productionDeps()` 注入（详见第 09 章 QueryDeps）。

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

### 11.1 QueryConfig 不可变快照（真实结构）

QueryConfig 不含 temperature/topP 等采样参数——它是**运行时门控快照**（`src/query/config.ts:29`），每次 `query()` 调用时创建一次，防止循环中途门控变化：

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean   // tengu_streaming_tool_execution2
    emitToolUseSummaries: boolean     // CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES
    isAnt: boolean                    // USER_TYPE === 'ant'
    fastModeEnabled: boolean          // CLAUDE_CODE_DISABLE_FAST_MODE 反相
  }
}
```

（模型、工具、maxTokens、thinking 等查询参数在 `QueryParams` / `ToolUseContext.options` 上传递。）

### 11.2 模型选择链（真实优先级）

```
--model CLI 参数 / SDK 指定
  > 会话内 /model 切换（mainLoopModelOverride）
  > ANTHROPIC_MODEL 环境变量
  > 用户设置（settings.model）/ 项目设置
  > 产品默认（Sonnet 4.6；Opus 别名 → 4.7）
```

别名解析见 `src/utils/model/model.ts:488-502`（sonnet/opus/haiku/best/opusplan，`[1m]` 后缀表示 1M 上下文），默认模型见 `getDefaultOpusModel/getDefaultSonnetModel`（`model.ts:110-140`）。
