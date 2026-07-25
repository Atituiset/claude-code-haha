# 状态管理与数据流详解

## 1. 概述

Claude Code 使用多种状态管理机制：
- **Bootstrap 状态**：全局基础状态（会话 ID、项目根目录等）
- **App 状态**：UI 和交互状态（消息、工具调用结果等）
- **持久化状态**：通过文件系统存储的配置和会话数据

## 2. Bootstrap 状态 (`src/bootstrap/state.js`)

### 2.1 核心状态变量

```typescript
// src/bootstrap/state.js
interface BootstrapState {
  // 会话管理
  sessionId: SessionId
  projectRoot: string
  originalCwd: string
  
  // 模型配置
  mainLoopModelOverride: string | null
  sdkBetas: string[]
  
  // 客户端类型
  clientType: ClientType
  
  // 标志位
  isInteractive: boolean
  isRemoteMode: boolean
  isNonInteractiveSession: boolean
  
  // API 配置
  directConnectServerUrl: string | null
  
  // 权限模式
  sessionBypassPermissionsMode: PermissionMode | null
}
```

### 2.2 状态访问函数

```typescript
// 获取状态
export function getSessionId(): SessionId
export function getProjectRoot(): string
export function getCwd(): string
export function getMainLoopModel(): string

// 设置状态
export function setSessionId(id: SessionId): void
export function setProjectRoot(root: string): void
export function setCwd(cwd: string): void
export function switchSession(id: SessionId): void

// 批量设置
export function setCwdState(cwd: string, projectRoot: string): void
export function setOriginalCwd(cwd: string): void
```

### 2.3 会话切换

```typescript
export function switchSession(id: SessionId): void {
  const previousId = currentState.sessionId
  
  // 1. 保存当前会话状态
  saveSessionState(previousId)
  
  // 2. 切换会话 ID
  currentState.sessionId = id
  
  // 3. 加载新会话状态
  loadSessionState(id)
  
  // 4. 通知状态变化
  broadcastSessionChange({
    previousId,
    newId: id,
  })
}
```

## 3. App 状态 (`src/state/AppStateStore.js`)

### 3.1 AppState 接口

```typescript
export interface AppState {
  // 消息历史
  messages: Message[]
  
  // 推测性执行
  speculation: Speculation | null
  
  // 可用工具
  tools: Tool[]
  
  // MCP 相关
  mcpClients: Map<string, McpClient>
  mcpTools: Tool[]
  
  // 权限请求
  permissionRequests: PermissionRequest[]
  
  // 通知
  notifications: Notification[]
  
  // 会话成本
  sessionCost: number
  costThreshold: number
  
  // UI 状态
  inputValue: string
  selectedAgentId: string | null
  vimMode: 'normal' | 'insert'
  
  // 主题
  theme: ThemeName
  themeSettings: ThemeSettings
  
  // 性能指标
  fps: number
  fpsAverage: number
  fpsLow1Pct: number
}
```

### 3.2 状态存储 (`Zustand`)

```typescript
// src/state/store.js
import { createStore } from './store.js'

export const appStore = createStore<AppState>((set, get) => ({
  // 初始状态
  messages: [],
  speculation: null,
  tools: [],
  mcpClients: new Map(),
  mcpTools: [],
  permissionRequests: [],
  notifications: [],
  sessionCost: 0,
  costThreshold: 100,
  inputValue: '',
  selectedAgentId: null,
  vimMode: 'insert',
  theme: 'dark',
  themeSettings: defaultThemeSettings,
  fps: 0,
  fpsAverage: 0,
  fpsLow1Pct: 0,
  
  // 操作
  addMessage: (message: Message) => set(state => ({
    messages: [...state.messages, message],
  })),
  
  setSpeculation: (speculation: Speculation | null) => set({
    speculation,
  }),
  
  addPermissionRequest: (request: PermissionRequest) => set(state => ({
    permissionRequests: [...state.permissionRequests, request],
  })),
  
  // ...
}))
```

### 3.3 React Hooks

```typescript
// src/state/AppState.js
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useStore(appStore)
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState()),
  )
}

export function useSetAppState(): {
  addMessage: (message: Message) => void
  setSpeculation: (speculation: Speculation | null) => void
  // ...
} {
  return {
    addMessage: (message) => appStore.getState().addMessage(message),
    setSpeculation: (speculation) => appStore.getState().setSpeculation(speculation),
    // ...
  }
}
```

### 3.4 状态变化监听 (`src/state/onChangeAppState.js`)

```typescript
export function onChangeAppState(
  partial: Partial<AppState> | ((prev: AppState) => Partial<AppState>),
  shouldRender = true,
): void {
  // 1. 更新状态
  appStore.setState(partial)
  
  // 2. 触发渲染更新
  if (shouldRender) {
    requestAnimationFrame(() => {
      // 触发 Ink 重新渲染
      inkRoot.render(<App />)
    })
  }
  
  // 3. 持久化（如需要）
  if (shouldPersist(partial)) {
    saveAppStateSnapshot(appStore.getState())
  }
}
```

## 4. 持久化状态

### 4.1 会话存储 (`src/utils/sessionStorage.js`)

```typescript
interface SessionData {
  sessionId: string
  messages: Message[]
  model: string
  cost: number
  startedAt: number
  lastActivityAt: number
  title?: string
  customTitle?: string
}

// 保存会话
export function saveSession(session: SessionData): void {
  const sessionsDir = path.join(getSessionDir())
  ensureDirExists(sessionsDir)
  
  const filePath = path.join(sessionsDir, `${session.sessionId}.json`)
  writeFileSync(filePath, JSON.stringify(session))
}

// 加载会话
export function loadSession(sessionId: string): SessionData | null {
  const filePath = path.join(getSessionDir(), `${sessionId}.json`)
  
  if (!existsSync(filePath)) {
    return null
  }
  
  return JSON.parse(readFileSync(filePath, 'utf8'))
}

// 搜索会话
export function searchSessionsByCustomTitle(title: string): SessionData[] {
  const sessionsDir = getSessionDir()
  const files = readdirSync(sessionsDir).filter(f => f.endsWith('.json'))
  
  return files
    .map(f => JSON.parse(readFileSync(path.join(sessionsDir, f), 'utf8')))
    .filter(s => s.customTitle?.includes(title))
}
```

### 4.2 配置存储 (`src/utils/config.js`)

```typescript
interface GlobalConfig {
  version: string
  lastReleaseNotesSeen: string
  permissions: PermissionConfig
  themes: ThemeSettings
  agentSwarmsEnabled: boolean
}

export function getGlobalConfig(): GlobalConfig {
  const configPath = path.join(getGlobalConfigDir(), 'settings.json')
  
  if (!existsSync(configPath)) {
    return defaultConfig
  }
  
  return JSON.parse(readFileSync(configPath, 'utf8'))
}

export function saveGlobalConfig(config: Partial<GlobalConfig>): void {
  const current = getGlobalConfig()
  const updated = { ...current, ...config }
  
  writeFileSync(
    getGlobalConfigPath(),
    JSON.stringify(updated, null, 2),
  )
}
```

### 4.3 项目配置

```typescript
interface ProjectConfig {
  projectRoot: string
  lastSessionId: string | null
  lastCost?: number
  lastDuration?: number
  lastLinesAdded?: number
  lastLinesRemoved?: number
}

export function getCurrentProjectConfig(): ProjectConfig {
  const configPath = path.join(getProjectRoot(), '.claude', 'settings.json')
  
  if (!existsSync(configPath)) {
    return defaultProjectConfig
  }
  
  return JSON.parse(readFileSync(configPath, 'utf8'))
}

export function saveProjectConfig(config: Partial<ProjectConfig>): void {
  const current = getCurrentProjectConfig()
  const updated = { ...current, ...config }
  
  writeFileSync(
    path.join(getProjectRoot(), '.claude', 'settings.json'),
    JSON.stringify(updated, null, 2),
  )
}
```

## 5. 数据流

### 5.1 用户输入流程

```
User Input (Terminal)
       │
       ▼
┌─────────────────┐
│  REPL.tsx       │  ← 捕获键盘输入
│  useInput       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  inputValue     │  ← 更新状态
│  (AppState)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  query.ts       │  ← 构建 API 请求
│  buildQuery     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  API            │  ← 调用 Anthropic API
│  messages.create│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  handleResponse │  ← 处理工具调用/文本响应
│  (query.ts)     │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌───────┐
│ Tools │ │ Text  │
│.run   │ │ Response │
└───┬───┘ └───┬───┘
    │         │
    ▼         ▼
┌─────────────────┐
│  addMessage     │  ← 更新消息列表
│  (AppState)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  onChangeAppState│ ← 触发重新渲染
│  requestAnimation│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  REPL.tsx       │  ← Ink 重新渲染消息列表
│  (re-render)    │
└─────────────────┘
```

### 5.2 工具执行流程

```
API Response (tool_use)
         │
         ▼
┌─────────────────┐
│  runTools()     │  ← 工具编排
│  toolOrchestration│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  findTool()     │  ← 查找工具实现
│  (tools.ts)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  tool.handler() │  ← 执行工具
│  (e.g. BashTool)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  result         │  ← 工具执行结果
│  content        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  API Continue   │  ← 将结果发回 API
│  messages.create│
└─────────────────┘
```

### 5.3 权限请求流程

```
Tool Execution Request
         │
         ▼ (requiresPermissions)
┌─────────────────┐
│  checkPermission │  ← 检查权限
│  (permissionSetup)│
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌───────────┐
│Granted│ │ Needs     │
│       │ │ Approval  │
└───────┘ └─────┬─────┘
                │
                ▼
        ┌─────────────────┐
        │  addPermission  │
        │  Request        │
        │  (AppState)     │
        └────────┬────────┘
                │
                ▼
        ┌─────────────────┐
        │  showPermission │
        │  Dialog         │
        │  (REPL.tsx)     │
        └────────┬────────┘
                │
                ▼
        ┌─────────────────┐
        │  User Decision  │
        │  (allow/deny)   │
        └────────┬────────┘
                │
         ┌──────┴──────┐
         │             │
         ▼             ▼
   ┌─────────┐  ┌─────────┐
   │ Executed │  │ Blocked  │
   │ + Save   │  │ + Log   │
   │ Permission│ │         │
   └─────────┘  └─────────┘
```

## 6. 状态初始化

### 6.1 应用启动

```typescript
async function initializeApp(): Promise<void> {
  // 1. 初始化 Bootstrap 状态
  setCwd(initialCwd)
  setProjectRoot(initialCwd)
  setSessionId(generateSessionId())
  
  // 2. 加载持久化配置
  enableConfigs()
  loadGlobalConfig()
  loadProjectConfig()
  
  // 3. 初始化 App 状态
  const defaultState = getDefaultAppState()
  appStore.setState(defaultState)
  
  // 4. 加载历史会话（如需恢复）
  if (options.resumeSession) {
    await loadConversationForResume(options.sessionId)
  }
  
  // 5. 初始化 MCP 服务器
  await startMcpServers()
  
  // 6. 初始化工具列表
  const tools = await getTools()
  appStore.setState({ tools })
}
```

### 6.2 会话恢复

```typescript
async function loadConversationForResume(
  sessionId: string,
): Promise<void> {
  // 1. 加载会话数据
  const sessionData = loadSession(sessionId)
  
  if (!sessionData) {
    throw new Error(`Session not found: ${sessionId}`)
  }
  
  // 2. 恢复状态
  switchSession(sessionId)
  
  // 3. 恢复消息历史
  appStore.setState({
    messages: sessionData.messages,
    sessionCost: sessionData.cost,
  })
  
  // 4. 恢复模型配置
  setMainLoopModelOverride(sessionData.model)
}
```

## 7. 性能指标追踪

### 7.1 FPS 追踪 (`src/utils/fpsTracker.js`)

```typescript
interface FpsMetrics {
  current: number
  average: number
  low1Pct: number
  samples: number[]
}

export class FpsTracker {
  private samples: number[] = []
  private lastFrame: number = 0
  
  tick(): void {
    const now = performance.now()
    const delta = now - this.lastFrame
    this.lastFrame = now
    
    const fps = 1000 / delta
    this.samples.push(fps)
    
    // 保持最近 300 帧
    if (this.samples.length > 300) {
      this.samples.shift()
    }
  }
  
  getMetrics(): FpsMetrics {
    const sorted = [...this.samples].sort((a, b) => a - b)
    
    return {
      current: sorted[sorted.length - 1],
      average: sorted.reduce((a, b) => a + b) / sorted.length,
      low1Pct: sorted[Math.floor(sorted.length * 0.01)],
      samples: this.samples.length,
    }
  }
}
```

### 7.2 成本追踪

```typescript
interface CostMetrics {
  inputTokens: number
  outputTokens: number
  cacheCreationTokens: number
  cacheReadTokens: number
  totalCost: number
}

export function trackApiCost(usage: Usage): void {
  const state = appStore.getState()
  
  const metrics: CostMetrics = {
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
    cacheCreationTokens: usage.cache_creation_input_tokens ?? 0,
    cacheReadTokens: usage.cache_read_input_tokens ?? 0,
    totalCost: calculateCost(usage),
  }
  
  // 更新 App 状态
  appStore.setState(state => ({
    sessionCost: state.sessionCost + metrics.totalCost,
  }))
  
  // 保存到会话数据
  saveSessionCost(metrics)
}
```

## 8. 状态调试

### 8.1 状态检查命令

```bash
# 查看当前会话状态
claude-haha session info

# 查看历史会话
claude-haha session list

# 查看配置
claude-haha config show
```

### 8.2 状态导出

```typescript
export function exportStateDump(): string {
  return JSON.stringify({
    bootstrap: getBootstrapState(),
    app: appStore.getState(),
    config: {
      global: getGlobalConfig(),
      project: getCurrentProjectConfig(),
    },
    sessions: listSessions().map(loadSession),
  }, null, 2)
}
```
