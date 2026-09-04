# 状态管理与数据流详解

## 1. 概述

Claude Code 使用多种状态管理机制：
- **Bootstrap 状态**：全局基础状态（会话 ID、项目根目录等）
- **App 状态**：UI 和交互状态（消息、工具调用结果等）
- **持久化状态**：通过文件系统存储的配置和会话数据

## 2. Bootstrap 状态 (`src/bootstrap/state.ts`)

### 2.1 核心状态变量

```typescript
// src/bootstrap/state.ts
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

真实实现（`src/bootstrap/state.ts:468`）非常轻量——只换 ID 与目录并发出信号，没有"保存/加载会话状态"步骤：

```typescript
export function switchSession(
  sessionId: SessionId,
  projectDir: string | null = null,
): void {
  // 清理旧会话的 plan-slug 缓存，保持 Map 有界
  STATE.planSlugCache.delete(STATE.sessionId)
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  sessionSwitched.emit(sessionId)   // 订阅者（如 concurrentSessions）自行同步
}

// 订阅接口
export const onSessionSwitch = sessionSwitched.subscribe
```

## 3. App 状态 (`src/state/AppStateStore.ts`)

### 3.1 AppState 接口

真实 AppState 是 `DeepImmutable` 类型的扁平结构（`src/state/AppStateStore.ts:89`），核心字段：

```typescript
export type AppState = DeepImmutable<{
  messages: Message[]                    // 消息历史
  speculation: SpeculationState          // 提示建议的推测执行状态
  tools: Tool[]                          // 可用工具
  mcp: { ... }                           // MCP 服务器/命令/资源视图
  toolPermissionContext: ToolPermissionContext
  tasks: ...                              // 后台任务/队友任务表
  notifications: { ... }
  mainLoopModel: ModelSetting            // 当前模型
  // vimMode/主题等 UI 偏好存于全局配置，不在 AppState
  // fps 指标由 src/utils/fpsTracker.ts 独立采集（FpsMetricsProvider）
}>

export function getDefaultAppState(): AppState { /* ... */ }
```

### 3.2 状态存储（自研轻量 store）

注意：CLI 的状态层**不是 Zustand**（Zustand 只在 Desktop 用）。`src/state/store.ts` 提供自研 `createStore`；写入 API 是 `setAppState(fn)`，组件侧通过 `useAppState`（`src/state/AppState.tsx`）以 selector 订阅。

```typescript
// src/state/store.ts
export function createStore<T>(/* ... */) {
  // getState / setState / subscribe
}
```

### 3.3 React Hooks

```typescript
// src/state/AppState.tsx 提供 useAppState(selector)
// 基于 React useSyncExternalStore 订阅 store 变化
const messages = useAppState(s => s.messages)

// 写入通过 setAppState（任意路径），不是逐字段的 setter 对象
setAppState(prev => ({ ...prev, messages: [...prev.messages, msg] }))
```

### 3.4 状态变化副作用 (`src/state/onChangeAppState.ts`)

`onChangeAppState({ newState, oldState })` 消费每次 diff（真实签名 `src/state/onChangeAppState.ts:43`）：

```typescript
export function onChangeAppState({
  newState,
  oldState,
}: {
  newState: AppState
  oldState: AppState
}) {
  // 权限模式变化 → 通知 CCR external_metadata 与 SDK 状态流
  //   （统一 choke point，替代散落各处的手动 notify）
  // mainLoopModel 变化 → 写回用户设置 / bootstrap override
  // expandedView 变化 → 持久化 showExpandedTodos / showSpinnerTree
  // ...
}
```

它不负责"触发 Ink 重渲染"——渲染由 React 订阅机制自动驱动，也没有 `shouldPersist(partial)` 快照逻辑。

## 4. 持久化状态

### 4.1 会话存储 (`src/utils/sessionStorage.ts`)

会话以 **JSONL 转录文件**持久化（`sessionStorage.ts:204`）：`~/.claude/projects/<项目目录编码>/<sessionId>.jsonl`；子 Agent 有独立转录（`agent-<agentId>.jsonl`）。关键函数包括 `getTranscriptPathForSession`、`searchSessionsByCustomTitle`（按自定义标题搜索历史会话，供 /resume 使用）等。

```typescript
// src/utils/sessionStorage.ts:204
return join(projectDir, `${getSessionId()}.jsonl`)
// 子 Agent 转录
return join(base, `agent-${agentId}.jsonl`)
```

### 4.2 全局配置 (`src/utils/config.ts`)

`~/.claude.json` 是全局配置文件（注意不是 `~/.claude/settings.json`）。真实 `GlobalConfig`（`config.ts:183`）字段很多，代表性的包括：

```typescript
export type GlobalConfig = {
  projects?: Record<string, ProjectConfig>   // 按项目路径索引
  numStartups: number
  installMethod?: InstallMethod
  autoUpdates?: boolean
  userID?: string
  theme: ThemeSetting
  hasCompletedOnboarding?: boolean
  lastReleaseNotesSeen?: string
  mcpServers?: Record<string, McpServerConfig>   // 用户级 MCP
  preferredNotifChannel: NotificationChannel
  // ... 更多
}

// 读取/写入带合并语义（saveGlobalConfig 接收函数式更新）
export function getGlobalConfig(): GlobalConfig
export function saveGlobalConfig(
  update: GlobalConfig | ((current: GlobalConfig) => GlobalConfig),
): void
```

### 4.3 项目配置

项目配置**不在** `.claude/settings.json`，而是全局配置中按项目路径索引的 `projects` 键（`config.ts:76`）。真实字段：

```typescript
export type ProjectConfig = {
  allowedTools: string[]
  mcpContextUris: string[]
  lastCost?: number
  lastDuration?: number
  lastLinesAdded?: number
  lastLinesRemoved?: number
  lastSessionId?: string          // /resume 默认目标
  lastFpsAverage?: number
  lastFpsLow1Pct?: number
  hasTrustDialogAccepted?: boolean
  projectOnboardingSeenCount: number
  // ... 更多
}
```

另有独立的 `~/.claude/settings.json` / 项目 `.claude/settings.json`（settings 层，含 permissions/hooks/model 等，见 `src/utils/settings/`），与 GlobalConfig 分层管理。

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

### 6.1 应用启动（真实次序）

```
bin/claude-haha
  → entrypoints/cli.tsx（快速路径路由）
  → main.tsx action handler
      → init()           // 13 步初始化（enableConfigs 等，见第 14 章）
      → setup(cwd, ...)   // 工作目录/UDS/hooks 快照
      → showSetupScreens // 信任对话框/入职/MCP 审批
      → launchRepl       // 渲染 App + REPL
```

REPL 挂载后，`useMergedTools` 等 hook 拉取工具池（内置 + MCP），写入 AppState。

### 6.2 会话恢复

`--resume` / `/resume` 流程（`src/commands/resume/resume.tsx`）：从 `~/.claude/projects/.../<sessionId>.jsonl` 加载转录（`loadFullLog` 等），`switchSession(sessionId, projectDir)` 切换 bootstrap 状态，消息历史回填 AppState。成本与 token 计数由 `src/cost-tracker.ts` 在使用侧累加。

## 7. 性能指标追踪

### 7.1 FPS 追踪 (`src/utils/fpsTracker.ts`)

注意路径是 `.ts`。提供 `FpsMetrics`（current/average/low1Pct），由 `FpsMetricsProvider`（`src/components/App.tsx`）注入组件树，会话结束时写入 ProjectConfig 的 `lastFpsAverage/lastFpsLow1Pct`。

### 7.2 成本追踪 (`src/cost-tracker.ts`, 381 行)

成本数据存于 **bootstrap state 计数器**（totalCostUSD、totalInputTokens 等），不是 AppState 字段：

```typescript
getTotalCostUSD()        // 总成本 (USD)
getModelUsage()           // 按模型的使用量
getTotalInputTokens()
getTotalOutputTokens()
```

会话结束时 `saveCurrentSessionCosts()` 写入 ProjectConfig（lastCost/lastModelUsage 等），恢复会话时 `restoreCostStateForSession()` 回读。

## 8. 状态调试

### 8.1 检查命令

```bash
# 恢复选择器（列出历史会话）
claude-haha --resume
claude-haha -r            # 恢复最近一次会话

# 非交互输出（观察单轮 query 的状态流转）
claude-haha -p "..." --output-format stream-json
```

### 8.2 状态观察

没有 `session info` 子命令或 `exportStateDump`。调试时可用：

- `--debug` / `--debug-to-stderr`：观察权限模式切换、工具决策、hook 执行等事件
- `--output-format stream-json`：结构化输出每轮消息与 usage
