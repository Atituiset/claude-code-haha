# 斜杠命令与 Skills 系统

## 1. 斜杠命令 (`src/commands.js`)

### 1.1 命令注册

```typescript
export interface Command {
  name: string
  description: string
  options?: CommandOption[]
  handler: (args: CommandArgs) => Promise<void> | void
  aliases?: string[]
}

export function getCommands(projectRoot: string): Command[] {
  const commands: Command[] = []
  
  // 1. 核心命令（始终可用）
  commands.push(...getCoreCommands())
  
  // 2. MCP 服务器贡献的命令
  commands.push(...getMcpCommands())
  
  // 3. 插件贡献的命令
  commands.push(...getPluginCommands())
  
  // 4. Skill 贡献的命令
  commands.push(...getSkillCommands())
  
  return commands
}
```

### 1.2 核心命令

| 命令 | 描述 | 功能 |
|------|------|------|
| `/login` | 登录 | 启动 OAuth 登录流程 |
| `/logout` | 登出 | 清除认证信息 |
| `/session` | 会话管理 | 列出、切换、删除会话 |
| `/model` | 模型切换 | 切换当前使用的模型 |
| `/cost` | 成本显示 | 显示当前会话成本 |
| `/clear` | 清除 | 清除当前对话历史 |
| `/help` | 帮助 | 显示帮助信息 |
| `/exit` | 退出 | 退出 Claude Code |
| `/commit` | 提交 | 启动 git commit 流程 |
| `/review` | 代码审查 | 启动代码审查流程 |
| `/test` | 测试 | 运行测试 |
| `/debug` | 调试 | 开启调试模式 |

### 1.3 命令处理器

```typescript
async function handleCommand(
  input: string,
  context: CommandContext,
): Promise<CommandResult> {
  // 1. 解析命令
  const { command, args } = parseCommand(input)
  
  // 2. 查找命令
  const cmd = findCommand(command)
  
  if (!cmd) {
    return { error: `Unknown command: ${command}` }
  }
  
  // 3. 检查前置条件
  if (cmd.requiresAuth && !context.isAuthenticated) {
    return { error: 'Please login first: /login' }
  }
  
  if (cmd.requiresProject && !context.hasProject) {
    return { error: 'This command must be run in a project directory' }
  }
  
  // 4. 执行命令
  try {
    const result = await cmd.handler(args, context)
    return { result }
  } catch (error) {
    return { error: error.message }
  }
}
```

### 1.4 内置命令实现

#### `/session` 命令

```typescript
const sessionCommand: Command = {
  name: 'session',
  description: 'Manage sessions',
  options: [
    { name: 'list', short: 'l', description: 'List all sessions' },
    { name: 'switch', short: 's', description: 'Switch to a session' },
    { name: 'delete', short: 'd', description: 'Delete a session' },
    { name: 'export', short: 'e', description: 'Export session data' },
  ],
  
  async handler(args, context) {
    if (args.list) {
      const sessions = listSessions()
      console.table(sessions.map(s => ({
        id: s.sessionId,
        title: s.title || 'Untitled',
        cost: `$${s.cost.toFixed(4)}`,
        lastActivity: new Date(s.lastActivityAt).toLocaleString(),
      })))
      return
    }
    
    if (args.switch) {
      await switchSession(args.switch)
      console.log(`Switched to session: ${args.switch}`)
      return
    }
    
    if (args.delete) {
      deleteSession(args.delete)
      console.log(`Deleted session: ${args.delete}`)
      return
    }
    
    // 显示帮助
    console.log(this.description)
  },
}
```

#### `/commit` 命令

```typescript
const commitCommand: Command = {
  name: 'commit',
  description: 'Create a git commit',
  
  async handler(args, context) {
    // 1. 检查 git 状态
    const status = await gitStatus()
    
    if (status.isClean) {
      console.log('Nothing to commit')
      return
    }
    
    // 2. 获取变更文件列表
    const files = status.changedFiles
    
    // 3. 检查是否有 Linter 可用
    const linter = await detectLinter(files)
    
    // 4. 自动修复（如启用）
    if (linter && context.autoFix) {
      await linter.fix(files)
    }
    
    // 5. 生成 commit 消息
    const commitMessage = await generateCommitMessage(files, context)
    
    // 6. 显示确认
    console.log(`Files to commit: ${files.join(', ')}`)
    console.log(`Commit message: ${commitMessage}`)
    
    const confirmed = await confirm('Proceed with commit?')
    
    if (confirmed) {
      await gitCommit(commitMessage, files)
      console.log('Committed successfully')
    }
  },
}
```

## 2. Skills 系统 (`src/skills/`)

### 2.1 概念

Skills 是一种可扩展的能力模块，允许：
- 添加新的命令
- 定义复杂的 workflow
- 封装常用的操作序列

### 2.2 Skill 定义

```typescript
// src/skills/types.ts
interface Skill {
  id: string
  name: string
  description: string
  version: string
  
  // 触发方式
  triggers?: SkillTrigger[]
  
  // 执行入口
  execute: (context: SkillContext) => Promise<SkillResult>
  
  // 生命周期钩子
  onStart?: () => void
  onEnd?: () => void
  
  // 权限要求
  requiredPermissions?: Permission[]
}

interface SkillTrigger {
  type: 'slash' | 'keyword' | 'context'
  pattern: string | RegExp
}
```

### 2.3 Skill 目录结构

```
.claude/skills/
├── skill-name/
│   ├── skill.json        # Skill 元数据
│   ├── src/
│   │   └── index.ts     # 执行入口
│   ├── commands/         # 贡献的命令
│   ├── tools/            # 贡献的工具
│   └── resources/        # 资源文件
```

### 2.4 Skill 元数据格式

```json
{
  "id": "code-review",
  "name": "Code Review",
  "description": "Automated code review for pull requests",
  "version": "1.0.0",
  "triggers": [
    { "type": "slash", "pattern": "/review" },
    { "type": "keyword", "pattern": "review this" }
  ],
  "requiredPermissions": ["git.read", "git.write"]
}
```

### 2.5 Skill 加载 (`src/skills/bundled/index.js`)

```typescript
export function initBundledSkills(): void {
  // 1. 注册内置 Skills
  registerSkill({
    id: 'verify',
    name: 'Verify',
    description: 'Verify plan execution',
    execute: verifySkillExecute,
  })
  
  registerSkill({
    id: 'update',
    name: 'Update',
    description: 'Check and apply updates',
    execute: updateSkillExecute,
  })
  
  // 2. 加载用户 Skills
  const userSkills = loadUserSkills()
  for (const skill of userSkills) {
    registerSkill(skill)
  }
  
  // 3. 触发 Skill 加载完成事件
  logEvent('skills_loaded', { count: getRegisteredSkills().length })
}
```

### 2.6 Skill 执行 (`src/tools/SkillTool/`)

```typescript
const SkillTool: Tool = {
  name: 'Skill',
  description: 'Invoke a skill',
  input_schema: {
    type: 'object',
    properties: {
      skill: { type: 'string', description: 'Skill ID to invoke' },
      args: { type: 'object', description: 'Arguments for the skill' },
    },
    required: ['skill'],
  },
  
  async handler(input, context) {
    const skill = getSkill(input.skill)
    
    if (!skill) {
      return {
        tool_use_id: input.tool_use_id,
        content: `Skill not found: ${input.skill}`,
        is_error: true,
      }
    }
    
    // 1. 执行前置钩子
    if (skill.onStart) {
      await skill.onStart()
    }
    
    // 2. 执行 Skill
    const result = await skill.execute({
      args: input.args,
      context,
    })
    
    // 3. 执行后置钩子
    if (skill.onEnd) {
      await skill.onEnd()
    }
    
    return {
      tool_use_id: input.tool_use_id,
      content: result.content,
    }
  },
}
```

## 3. 内置 Skill 详解

### 3.1 Verify Skill

验证计划执行：

```typescript
const verifySkill: Skill = {
  id: 'verify',
  name: 'Verify',
  description: 'Verify that a plan is being executed correctly',
  
  triggers: [
    { type: 'slash', pattern: '/verify' },
    { type: 'keyword', pattern: 'verify this' },
  ],
  
  async execute(context) {
    // 1. 获取当前对话上下文
    const messages = context.getMessages()
    const plan = extractPlanFromContext(messages)
    
    if (!plan) {
      return {
        content: 'No active plan found to verify',
      }
    }
    
    // 2. 检查计划执行状态
    const status = await checkPlanStatus(plan)
    
    // 3. 生成验证报告
    const report = {
      plan: plan.description,
      status: status.isComplete ? 'complete' : 'in_progress',
      completedSteps: status.completedSteps,
      remainingSteps: status.remainingSteps,
      issues: status.issues,
    }
    
    return {
      content: formatVerificationReport(report),
    }
  },
}
```

### 3.2 Update Skill

检查和应用更新：

```typescript
const updateSkill: Skill = {
  id: 'update',
  name: 'Update',
  description: 'Check for and apply Claude Code updates',
  
  async execute(context) {
    // 1. 检查当前版本
    const currentVersion = getCurrentVersion()
    
    // 2. 检查最新版本
    const latestVersion = await checkLatestVersion()
    
    if (!needsUpdate(currentVersion, latestVersion)) {
      return {
        content: `You are running the latest version: ${currentVersion}`,
      }
    }
    
    // 3. 下载并应用更新
    const downloaded = await downloadUpdate(latestVersion)
    await applyUpdate(downloaded)
    
    return {
      content: `Updated to version ${latestVersion}. Please restart Claude Code.`,
    }
  },
}
```

## 4. 命令执行流程

```
User Input: "/commit --all"
         │
         ▼
┌─────────────────┐
│  REPL.tsx       │  ← 捕获输入
│  parseInput     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  isCommand()    │  ← 检测是否为命令
│  input.startsWith('/') │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
  ┌─────┐  ┌──────────┐
  │ Yes │  │ No       │
  └──┬──┘  └────┬─────┘
     │          │
     ▼          ▼
┌─────────┐  ┌─────────┐
│ Command │  │ Query   │
│ Handler│  │ Handler │
└────┬───┘  └────┬────┘
     │          │
     ▼          ▼
┌─────────┐  ┌─────────┐
│ Execute │  │ API     │
│ Command │  │ Query   │
└────┬───┘  └────┬────┘
     │          │
     └────┬─────┘
          ▼
   ┌─────────────────┐
   │  Result Output  │
   │  (to terminal) │
   └─────────────────┘
```

## 5. Skill 与命令的协作

### 5.1 命令调用 Skill

```typescript
const reviewCommand: Command = {
  name: 'review',
  description: 'Code review',
  
  async handler(args, context) {
    // 使用 Skill 工具调用 code-review skill
    const result = await context.tools.Skill.execute({
      skill: 'code-review',
      args: { files: args.files },
    })
    
    console.log(result.content)
  },
}
```

### 5.2 Skill 调用命令

```typescript
const codeReviewSkill: Skill = {
  id: 'code-review',
  name: 'Code Review',
  
  async execute(context) {
    // 使用 BashTool 运行 eslint
    const eslintResult = await context.tools.Bash.execute({
      command: 'npx eslint . --format json',
    })
    
    // 使用 ReadTool 读取问题文件
    const issues = JSON.parse(eslintResult.content)
    
    // 生成审查报告
    return {
      content: formatReviewReport(issues),
    }
  },
}
```

## 6. 自定义命令开发

### 6.1 创建自定义命令

```typescript
// .claude/commands/myCommand.ts
export const myCommand = {
  name: 'mycommand',
  description: 'My custom command',
  aliases: ['mc', 'my'],
  
  async handler(args: string[], context: CommandContext) {
    console.log('Hello from my custom command!')
    console.log('Arguments:', args)
    console.log('Current directory:', context.cwd)
    
    return { success: true }
  },
}
```

### 6.2 注册命令

```typescript
// 在 .claude/commands/index.ts 中注册
import { myCommand } from './myCommand.js'

export function getCustomCommands(): Command[] {
  return [
    myCommand,
    // 添加更多命令...
  ]
}
```

## 7. 命令行补全

### 7.1 补全规则

```typescript
export interface CompletionRule {
  command: string
  options: CompletionOption[]
  subcommands?: CompletionRule[]
}

const completionRules: CompletionRule[] = [
  {
    command: 'session',
    options: [
      { name: 'list', description: 'List all sessions' },
      { name: 'switch', description: 'Switch session' },
    ],
    subcommands: [
      {
        command: 'switch',
        options: listSessionIds(), // 动态补全
      },
    ],
  },
]
```

### 7.2 补全执行

```typescript
export function getCompletions(
  input: string,
  context: CompletionContext,
): string[] {
  const parts = input.split(' ')
  const last = parts[parts.length - 1]
  
  if (parts.length === 1) {
    // 命令名补全
    return getCommandCompletions(last)
  }
  
  if (parts.length === 2) {
    // 命令选项补全
    return getOptionCompletions(parts[0], last)
  }
  
  // 子命令补全
  return getSubcommandCompletions(parts[0], parts[1], last)
}
```
