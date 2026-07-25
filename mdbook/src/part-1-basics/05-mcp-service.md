# MCP (Model Context Protocol) 服务详解

## 1. 概述

MCP (Model Context Protocol) 是一种标准协议，允许 Claude Code 连接外部数据源和工具。MCP 服务器提供工具、资源和提示，扩展 Agent 的能力。

## 2. MCP 架构

```
                    ┌─────────────────┐
                    │   Claude Code   │
                    │   (MCP Client)  │
                    └────────┬────────┘
                             │ MCP Protocol
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼────────┐ ┌───▼──────┐ ┌────▼────────┐
     │ MCP Server 1   │ │MCP Srv 2 │ │ MCP Server 3 │
     │ (File System)  │ │ (Git)    │ │  (Custom)    │
     └────────────────┘ └─────────┘ └─────────────┘
```

## 3. MCP 配置

### 3.1 配置文件格式

MCP 服务器在 `.claude/mcp.json` 中配置：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/path/to/directory"
      ]
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "."]
    },
    "custom": {
      "command": "node",
      "args": ["/path/to/server.js"],
      "env": {
        "API_KEY": "secret"
      }
    }
  }
}
```

### 3.2 环境变量引用

```json
{
  "mcpServers": {
    "custom": {
      "command": "node",
      "args": ["/path/to/server.js"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}",
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

## 4. MCP 客户端 (`src/services/mcp/client.ts`)

### 4.1 核心结构

```typescript
export class McpClient {
  private transport: ClientTransport
  private client: StreamableHTTPClientTransport | StdioClientTransport
  
  constructor(
    private name: string,
    private config: McpServerConfig,
  ) {
    this.transport = this.createTransport(config)
  }
  
  private createTransport(config: McpServerConfig): ClientTransport {
    switch (config.type) {
      case 'stdio':
        return new StdioClientTransport({
          command: config.command,
          args: config.args,
          env: config.env,
        })
      
      case 'sse':
        return new SSEClientTransport(new URL(config.url))
      
      case 'streamable-http':
        return new StreamableHTTPClientTransport(new URL(config.url))
      
      case 'websocket':
        return new WebSocketTransport(new URL(config.url))
      
      default:
        throw new Error(`Unknown transport type: ${config.type}`)
    }
  }
}
```

### 4.2 工具获取

```typescript
export async function getMcpToolsCommandsAndResources(
  servers: McpServerConfig[],
): Promise<McpToolsAndResources> {
  // 1. 创建所有 MCP 客户端
  const clients = await Promise.all(
    servers.map(config => createMcpClient(config))
  )
  
  // 2. 并行获取工具、命令、资源
  const results = await Promise.all(
    clients.map(async client => ({
      client,
      tools: await client.listTools(),
      commands: await client.listCommands(),
      resources: await client.listResources(),
    }))
  )
  
  // 3. 汇总结果
  return {
    tools: results.flatMap(r => r.tools.map(t => ({ ...t, server: r.client.name }))),
    commands: results.flatMap(r => r.commands.map(c => ({ ...c, server: r.client.name }))),
    resources: results.flatMap(r => r.resources.map(r => ({ ...r, server: r.client.name }))),
  }
}
```

### 4.3 工具调用

```typescript
export async function callMcpTool(
  serverName: string,
  toolName: string,
  arguments: Record<string, unknown>,
): Promise<ToolResult> {
  const client = getMcpClient(serverName)
  
  if (!client) {
    throw new Error(`MCP server not found: ${serverName}`)
  }
  
  const result = await client.callTool(toolName, arguments)
  
  return {
    content: formatResult(result),
    is_error: result.isError,
  }
}
```

## 5. MCP 工具封装 (`src/tools/MCPTool/`)

```typescript
// MCPTool.ts
export class MCPTool implements Tool {
  name: string
  description: string
  input_schema: ToolInputJSONSchema
  
  constructor(
    private serverName: string,
    private mcpTool: McpServerTool,
  ) {
    this.name = `mcp_${serverName}_${mcpTool.name}`
    this.description = mcpTool.description ?? `MCP tool: ${mcpTool.name}`
    this.input_schema = mcpTool.inputSchema
  }
  
  async handler(input: ToolInput, context: ExecutionContext): Promise<ToolResult> {
    // 1. 获取 MCP 客户端
    const client = getMcpClient(this.serverName)
    
    if (!client) {
      return {
        tool_use_id: input.tool_use_id,
        content: `Error: MCP server '${this.serverName}' not available`,
        is_error: true,
      }
    }
    
    // 2. 调用 MCP 工具
    try {
      const result = await client.callTool(this.mcpTool.name, input.params)
      
      return {
        tool_use_id: input.tool_use_id,
        content: formatToolResult(result),
      }
    } catch (error) {
      return {
        tool_use_id: input.tool_use_id,
        content: `Error: ${error.message}`,
        is_error: true,
      }
    }
  }
}
```

## 6. 传输类型

### 6.1 Stdio (标准 I/O)

用于本地进程通信：

```typescript
const transport = new StdioClientTransport({
  command: 'npx',
  args: ['-y', '@modelcontextprotocol/server-filesystem', '/tmp'],
  env: { ... },
})

await transport.start()
```

### 6.2 SSE (Server-Sent Events)

用于 HTTP 长连接：

```typescript
const transport = new SSEClientTransport(
  new URL('https://example.com/mcp'),
  { authorization: 'Bearer token' }
)

await transport.start()
```

### 6.3 Streamable HTTP

用于 HTTP 流式响应：

```typescript
const transport = new StreamableHTTPClientTransport(
  new URL('https://example.com/mcp/stream'),
  { authorization: 'Bearer token' }
)

await transport.start()
```

### 6.4 WebSocket

用于双向实时通信：

```typescript
const transport = new WebSocketTransport(
  new URL('wss://example.com/mcp'),
  { headers: { authorization: 'Bearer token' } }
)

await transport.start()
```

## 7. 资源管理

### 7.1 资源列表

```typescript
interface McpResource {
  uri: string
  name: string
  description?: string
  mimeType?: string
}

async function listResources(client: McpClient): Promise<McpResource[]> {
  const response = await client.request({
    method: 'resources/list',
    params: {},
  })
  
  return response.resources
}
```

### 7.2 资源读取

```typescript
async function readResource(
  client: McpClient,
  uri: string,
): Promise<string> {
  const response = await client.request({
    method: 'resources/read',
    params: { uri },
  })
  
  return response.contents[0].text
}
```

### 7.3 资源订阅

```typescript
// 订阅资源变化通知
client.subscribe({
  method: 'resources/subscribe',
  params: { uri: 'file:///path/to/file' },
})

client.on('notification', (notification) => {
  if (notification.method === 'resources/updated') {
    console.log('Resource updated:', notification.params.uri)
  }
})
```

## 8. MCP 配置解析 (`src/services/mcp/config.ts`)

### 8.1 加载配置

```typescript
export async function getAllMcpConfigs(): Promise<McpServerConfig[]> {
  const configs: McpServerConfig[] = []
  
  // 1. 加载项目级配置
  const projectConfig = await loadMcpConfig('.claude/mcp.json')
  if (projectConfig) {
    configs.push(...projectConfig.mcpServers)
  }
  
  // 2. 加载全局配置
  const globalConfig = await loadMcpConfig('~/.claude/mcp.json')
  if (globalConfig) {
    configs.push(...globalConfig.mcpServers)
  }
  
  // 3. 合并环境变量覆盖
  const envConfigs = parseMcpEnvVars()
  configs.push(...envConfigs)
  
  // 4. 按策略过滤
  return filterMcpServersByPolicy(configs)
}
```

### 8.2 策略过滤

```typescript
export function filterMcpServersByPolicy(
  servers: McpServerConfig[],
): McpServerConfig[] {
  // 1. 获取策略限制
  const policy = getCurrentPolicy()
  
  if (!policy.allowMcpServers) {
    // 完全禁用
    return []
  }
  
  if (policy.allowedMcpServers) {
    // 白名单模式
    return servers.filter(s => policy.allowedMcpServers!.includes(s.name))
  }
  
  if (policy.blockedMcpServers) {
    // 黑名单模式
    return servers.filter(s => !policy.blockedMcpServers!.includes(s.name))
  }
  
  return servers
}
```

## 9. MCP 生命周期

### 9.1 启动

```typescript
async function startMcpServers(): Promise<void> {
  const configs = await getAllMcpConfigs()
  
  for (const config of configs) {
    try {
      const client = await createMcpClient(config)
      await client.connect()
      
      // 保存到全局客户端映射
      setMcpClient(config.name, client)
      
      console.log(`MCP server '${config.name}' started`)
    } catch (error) {
      console.error(`Failed to start MCP server '${config.name}':`, error)
    }
  }
}
```

### 9.2 健康检查

```typescript
async function checkMcpServerHealth(
  serverName: string,
): Promise<HealthStatus> {
  const client = getMcpClient(serverName)
  
  if (!client) {
    return { status: 'not_found' }
  }
  
  try {
    const start = Date.now()
    await client.ping()
    const latency = Date.now() - start
    
    return {
      status: 'healthy',
      latency_ms: latency,
    }
  } catch {
    return {
      status: 'unhealthy',
      error: 'Ping failed',
    }
  }
}
```

### 9.3 重连

```typescript
class McpClientWithReconnect {
  private retryCount = 0
  private maxRetries = 3
  
  async connect(): Promise<void> {
    try {
      await this.client.connect()
      this.retryCount = 0
    } catch (error) {
      if (this.retryCount < this.maxRetries) {
        this.retryCount++
        await this.reconnect()
      } else {
        throw error
      }
    }
  }
  
  private async reconnect(): Promise<void> {
    await new Promise(r => setTimeout(r, 1000 * this.retryCount))
    await this.connect()
  }
}
```

## 10. 错误处理

### 10.1 错误类型

```typescript
enum McpErrorCode {
  CONNECTION_FAILED = 'CONNECTION_FAILED',
  TOOL_NOT_FOUND = 'TOOL_NOT_FOUND',
  INVALID_PARAMETERS = 'INVALID_PARAMETERS',
  TIMEOUT = 'TIMEOUT',
  SERVER_ERROR = 'SERVER_ERROR',
}

class McpError extends Error {
  constructor(
    public code: McpErrorCode,
    message: string,
    public serverName?: string,
  ) {
    super(message)
    this.name = 'McpError'
  }
}
```

### 10.2 重试逻辑

```typescript
async function callMcpToolWithRetry(
  serverName: string,
  toolName: string,
  args: Record<string, unknown>,
  maxRetries = 3,
): Promise<ToolResult> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await callMcpTool(serverName, toolName, args)
    } catch (error) {
      if (attempt === maxRetries - 1) {
        throw error
      }
      
      if (isRetryableError(error)) {
        await sleep(1000 * Math.pow(2, attempt))
        continue
      }
      
      throw error
    }
  }
}
```

## 11. 官方 MCP 服务器

### 11.1 文件系统服务器

```bash
npx -y @modelcontextprotocol/server-filesystem /path/to/directory
```

### 11.2 Git 服务器

```bash
uvx mcp-server-git --repository /path/to/repo
```

### 11.3 PostgreSQL 服务器

```bash
npx -y @modelcontextprotocol/server-postgresql postgresql://localhost/mydb
```

## 12. 安全考虑

### 12.1 权限限制

```typescript
function checkMcpPermissions(
  serverName: string,
  toolName: string,
): boolean {
  const policy = getCurrentPolicy()
  
  // 检查是否允许使用 MCP
  if (!policy.allowMcpServers) {
    return false
  }
  
  // 检查服务器白名单
  if (policy.allowedMcpServers) {
    return policy.allowedMcpServers.includes(serverName)
  }
  
  return true
}
```

### 12.2 环境变量隔离

```typescript
function sanitizeMcpEnvVars(
  env: Record<string, string>,
  allowedVars: string[],
): Record<string, string> {
  const sanitized: Record<string, string> = {}
  
  for (const key of allowedVars) {
    if (env[key] !== undefined) {
      sanitized[key] = env[key]
    }
  }
  
  return sanitized
}
```
