# Server、Desktop 与通信架构深度解析

> 本文详解 Claude Code 的本地 Server 架构、Desktop 桌面端设计，以及 CLI/Server/Desktop 之间的通信机制。

## 1. Server 架构

### 1.1 概述

Server 是一个基于 Bun HTTP 的本地服务器，同时提供 REST API 和 WebSocket 通信，用于连接 CLI 和 Desktop 应用。

```typescript
// src/server/index.ts
const server = Bun.serve({
  port: process.env.SERVER_PORT ?? 3456,
  fetch(req, server) {
    const url = new URL(req.url)

    // WebSocket 升级
    if (url.pathname === '/ws') {
      return server.upgrade(req, { data: { ... } })
    }

    // REST API 路由
    return router.handle(req)
  },
  websocket: wsHandler,
})
```

### 1.2 服务清单

Server 实现了 28+ 个服务模块：

| 服务 | 文件 | 功能 |
|------|------|------|
| **会话回退** | `sessionRewindService.ts` | 预览/执行会话回退，恢复文件备份 |
| **搜索** | `searchService.ts` | 工作区文件搜索 + 会话历史搜索 |
| **工作树启动** | `repositoryLaunchService.ts` | Git worktree 会话启动器 |
| **工作区浏览器** | `workspaceService.ts` | 文件树、文件内容、diff 查看 |
| **诊断** | `doctorService.ts` | 持久化目标诊断（只读，不修复） |
| **Computer Use** | `computerUseApprovalService.ts` | MCP Computer Use 权限桥接 |
| **OpenAI OAuth** | `hahaOpenAIOAuthService.ts` | OpenAI/Codex OAuth 管理 |
| **H5 访问** | `h5AccessService.ts` | 移动端 Token 门控访问 |
| **团队监控** | `teamWatcher.ts` | 实时团队变更广播 |
| **MCP 预检** | `mcpHostPreflight.ts` | MCP 服务器命令验证 |
| **托管设置** | `managedSettingsService.ts` | 企业设置 CRUD |
| **存储迁移** | `persistentStorageMigrations.ts` | 存储格式升级 |
| **OAuth** | `hahaOAuthService.ts` | PKCE OAuth 流程 |
| **会话管理** | `sessionService.ts` | 会话列表、创建、恢复 |
| **消息服务** | `messageService.ts` | 消息发送和查询 |
| **权限服务** | `permissionService.ts` | 权限请求/决策桥接 |
| **工具服务** | `toolService.ts` | 工具列表和调用 |

### 1.3 会话回退服务

```typescript
// src/server/services/sessionRewindService.ts (1152 行)

// 预览回退：计算需要移除的消息 + 代码差异
async function previewRewind(sessionId: string, turnIndex: number) {
  // 1. 找到目标轮次之后的所有消息
  const messagesToRemove = getMessagesAfterTurn(sessionId, turnIndex)

  // 2. 计算文件差异
  // 优先从 ~/.claude/file-history/{sessionId}/ 读取备份
  // 回退到从 tool_use 条目提取 diff（解析 write/edit/multiedit/notebookedit/apply_patch）
  const codeDiff = await computeCodeDiff(sessionId, turnIndex)

  return { messagesToRemove, codeDiff }
}

// 执行回退：停止会话，恢复文件，裁剪 JSONL
async function executeRewind(sessionId: string, turnIndex: number) {
  // 1. 停止活跃会话
  await stopActiveSession(sessionId)

  // 2. 从备份恢复文件
  await restoreFileBackups(sessionId, turnIndex)

  // 3. 裁剪 JSONL 会话文件
  await trimSessionJsonl(sessionId, turnIndex)
}
```

### 1.4 搜索服务

```typescript
// src/server/services/searchService.ts (453 行)

// 双重搜索：
// 1. 工作区文件搜索 (ripgrep > grep > 可移植文件系统降级)
// 2. 会话历史搜索 (遍历 ~/.claude/projects/ JSONL 文件)

async function search(query: string, options: SearchOptions) {
  const [fileResults, sessionResults] = await Promise.all([
    searchWorkspaceFiles(query, options),  // ripgrep 优先
    searchSessionHistory(query, options),   // JSONL 文件搜索
  ])

  return { files: fileResults, sessions: sessionResults }
}
```

### 1.5 工作区浏览器服务

```typescript
// src/server/services/workspaceService.ts (1670 行)

// getStatus(): Git 状态 + 会话文件变更合并
// readFile(): 文本/图片/二进制检测，1MB 预览限制
// readTree(): 目录列表（隐藏文件排除）
// getDiff(): 会话 diff > file-history diff > git diff（三级降级）

// 会话文件变更提取：解析 tool_use 条目
// (write, edit, multiedit, notebookedit, apply_patch)
```

### 1.6 Doctor 诊断服务

```typescript
// src/server/services/doctorService.ts (509 行)

// 检查所有持久化目标：
// - 用户设置、提供商配置、适配器配置
// - Skills、Teams、Plugins
// - MCP 配置、OAuth tokens
// - 会话 JSONL 文件

// 所有项目标记为 protected: true —— 修复是纯预览，永不修改
// 路径消毒：~ 替换主目录，<project> 替换项目根
```

### 1.7 H5 访问服务

```typescript
// src/server/services/h5AccessService.ts (411 行)

// Token 门控的移动端 Web 访问
// Token 存储为 SHA-256 哈希在托管设置中
// 自动发现 LAN IP (CLAUDE_H5_AUTO_PUBLIC_URL=1)
// Origin 白名单验证（无通配符，仅 http/https）
// 三级 URL 解析：环境变量 > 自动 LAN > 已存储
```

## 2. Desktop 架构

### 2.1 技术栈

```
Desktop App
├── Frontend: React 19 + Zustand + i18n (EN/ZH)
├── Native:   Tauri (Rust)
├── Build:    Vite
├── Test:     Vitest + Testing Library (jsdom)
└── Communication: WebSocket → Server → CLI
```

### 2.2 目录结构

```
desktop/
├── src/
│   ├── api/              # API 客户端（与 Server 通信）
│   ├── components/       # 共享 UI 组件
│   ├── stores/           # Zustand 状态管理 (20 个 stores)
│   ├── hooks/            # React hooks
│   ├── i18n/             # 国际化 (EN/ZH)
│   ├── types/            # TypeScript 类型
│   ├── utils/            # 工具函数
│   └── __tests__/        # 测试
├── src-tauri/
│   ├── src/              # Rust 代码
│   │   ├── sidecar.rs    # Sidecar 进程管理
│   │   ├── pty.rs        # PTY 集成
│   │   └── notifications.rs  # macOS ObjC 通知
│   ├── Cargo.toml
│   └── tauri.conf.json
└── scripts/
    └── build-macos-arm64.sh  # Apple Silicon 构建
```

### 2.3 状态管理 (Zustand Stores)

Desktop 使用 20+ 个 Zustand stores 管理状态：

| Store | 功能 |
|-------|------|
| `sessionStore` | 会话列表、当前会话 |
| `messageStore` | 消息列表、流式更新 |
| `permissionStore` | 权限请求队列 |
| `toolStore` | 工具列表和状态 |
| `settingsStore` | 用户设置 |
| `themeStore` | 主题配置 |
| `mcpStore` | MCP 服务器状态 |
| `teamStore` | 团队和队友状态 |
| `workspaceStore` | 工作区文件树 |
| `diffStore` | 文件 diff 视图 |
| `searchStore` | 搜索状态 |
| `costStore` | 成本追踪 |

### 2.4 Tauri 集成

```rust
// desktop/src-tauri/src/sidecar.rs
// Sidecar 生命周期管理：
// 1. 启动 CLI 进程作为 sidecar
// 2. 管理标准 I/O
// 3. 监控进程健康
// 4. 处理崩溃重启

// desktop/src-tauri/src/pty.rs
// PTY (伪终端) 集成：
// 为 Bash 工具提供完整的终端模拟

// desktop/src-tauri/src/notifications.rs
// macOS 原生通知：
// 使用 Objective-C 桥接发送系统通知
```

### 2.5 Sidecar 构建与打包

```bash
# Sidecar 构建流程
bun run build:sidecars
# → 编译 Rust sidecar 二进制
# → 放置在 src-tauri/binaries/

# macOS ARM64 构建入口
desktop/scripts/build-macos-arm64.sh
# → 输出到 desktop/build-artifacts/macos-arm64/
```

## 3. 通信架构

### 3.1 三方通信模型

```mermaid
flowchart LR
    A[CLI / Agent Loop] <-->|WebSocket| B[Local Server]
    B <-->|WebSocket| C[Desktop App]
    B <-->|WebSocket| D[Mobile H5]
    B <-->|Permission Bridge| E[CCR / claude.ai]
    A <-->|UDS Socket| F[Swarm Workers]
```

### 3.2 Bridge 通信

#### 环境型 Bridge (CCR v1)

```typescript
// 基于 CLAUDE_CODE_REMOTE=1 环境变量
// CLI 作为远程会话运行在 CCR 容器中
// 通过 HTTP 长轮询与 claude.ai 通信
```

#### 无环境型 Bridge (CCR v2)

```typescript
// 更新的协议，不依赖特定环境变量
// 支持 WebSocket 双向通信
// 权限请求/决策通过 bridge 传递
```

### 3.3 WebSocket 协议

```typescript
// 消息类型
type WsMessage =
  | { type: 'session_update', sessionId, data }
  | { type: 'message', sessionId, message }
  | { type: 'permission_request', requestId, tool, input }
  | { type: 'permission_response', requestId, decision }
  | { type: 'tool_start', sessionId, toolName }
  | { type: 'tool_result', sessionId, result }
  | { type: 'team_update', teamId, data }
  | { type: 'computer_use_permission_request', requestId, ... }
  | { type: 'notification', ... }
```

### 3.4 UDS (Unix Domain Socket) 通信

```typescript
// 用于同机器上的 Agent 间通信
// setup() 中启动 UDS 消息服务器

// Swarm Worker 通过 UDS 发送/接收消息：
// - SendMailboxMessage: 向队友邮箱发送消息
// - PollInbox: 轮询自己的邮箱
// - ListPeers: 列出已连接的 Agent
```

### 3.5 Channel 中继

```typescript
// 通过 MCP 通知将权限请求发送到 IM 渠道
// Telegram、iMessage 等
// 回复是纯 yes/no（无 updatedInput）
// 即发即弃，优雅降级
```

## 4. Adapters (IM 适配器)

### 4.1 架构

```
adapters/
├── telegram/     # Telegram Bot 适配器
├── feishu/       # 飞书 Bot 适配器
├── wechat/       # 企业微信适配器
├── dingtalk/     # 钉钉适配器
└── shared/       # 共享代码
```

### 4.2 Sidecar 隔离模型

```typescript
// 每个 IM 适配器作为独立 sidecar 进程运行
// 与主 CLI 进程隔离，避免互相影响

// 热重启：适配器崩溃后自动重启
// 通信：通过标准 I/O 或 WebSocket 与 Server 通信
```

### 4.3 适配器工作流

```mermaid
sequenceDiagram
    participant User as IM 用户
    participant Adapter as IM 适配器
    participant Server as Local Server
    participant CLI as Agent Loop

    User->>Adapter: 发送消息 (如 "帮我写个函数")
    Adapter->>Server: POST /api/message
    Server->>CLI: WebSocket 消息转发
    CLI->>CLI: Agent Loop 处理
    CLI->>Server: 流式结果
    Server->>Adapter: WebSocket 结果推送
    Adapter->>User: 回复消息 (IM 格式)
```

## 5. 团队监控系统

```typescript
// src/server/services/teamWatcher.ts (335 行)

// 每 3 秒轮询 ~/.claude/teams/ 目录
// 检测事件：
// - team_created: 新团队目录出现
// - team_update: config.json 修改
// - team_deleted: 团队目录消失

// 广播到所有已连接的 WebSocket 客户端
// 从以下位置发现成员：
// - config.json (团队配置)
// - inboxes/ 目录 (消息邮箱)
// - subagents/ 目录 (子代理)

// 部分JSON恢复：通过正则提取 agentId/name/isActive 字段
```

## 6. 如何构建自己的 Agent 服务端

### 6.1 最小 Server

```typescript
const server = Bun.serve({
  port: 3456,
  fetch(req) {
    // REST API
    return new Response(JSON.stringify({ status: 'ok' }))
  },
  websocket: {
    message(ws, msg) {
      // WebSocket 消息处理
      ws.send(JSON.stringify({ echo: msg }))
    }
  }
})
```

### 6.2 生产级增强

| 增强项 | 作用 | 复杂度 |
|--------|------|--------|
| WebSocket 双向通信 | 实时更新 | 中 |
| 权限桥接 | 远程权限决策 | 高 |
| 会话回退 | 撤销 Agent 操作 | 高 |
| 工作区浏览器 | 文件查看/编辑 | 中 |
| IM 适配器 | 多渠道接入 | 高 |
| Tauri 原生集成 | 桌面应用 | 高 |
| H5 访问 | 移动端 | 中 |
| 团队监控 | 多 Agent 管理 | 中 |
| Doctor 诊断 | 健康检查 | 低 |

## 7. 关键文件索引

| 文件 | 行数 | 功能 |
|------|------|------|
| `src/server/index.ts` | - | Bun HTTP + WS 服务器 |
| `src/server/services/sessionRewindService.ts` | 1152 | 会话回退 |
| `src/server/services/searchService.ts` | 453 | 双重搜索 |
| `src/server/services/repositoryLaunchService.ts` | 643 | Worktree 启动 |
| `src/server/services/workspaceService.ts` | 1670 | 工作区浏览器 |
| `src/server/services/doctorService.ts` | 509 | 诊断 |
| `src/server/services/computerUseApprovalService.ts` | 73 | Computer Use 权限 |
| `src/server/services/hahaOpenAIOAuthService.ts` | 208 | OpenAI OAuth |
| `src/server/services/h5AccessService.ts` | 411 | H5 访问 |
| `src/server/services/teamWatcher.ts` | 335 | 团队监控 |
| `src/server/services/mcpHostPreflight.ts` | 166 | MCP 预检 |
| `src/server/services/managedSettingsService.ts` | 83 | 托管设置 |
| `src/server/services/persistentStorageMigrations.ts` | 194 | 存储迁移 |
| `desktop/src/` | - | React UI 代码 |
| `desktop/src-tauri/` | - | Tauri 原生集成 |
| `adapters/` | - | IM 适配器 |
