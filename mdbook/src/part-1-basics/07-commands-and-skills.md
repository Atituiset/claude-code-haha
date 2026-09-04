## 1. 斜杠命令 (`src/commands.ts`)

### 1.1 命令注册

真实 Command 类型和注册表见 `src/commands.ts`（752 行）与 `src/types/command.ts`。命令分三种类型：

```typescript
type Command = {
  type: 'local' | 'local-jsx' | 'prompt'
  name: string                  // 如 'help', 'compact'
  description: string
  aliases: string[]             // 如 help 的 ['h']
  handler?: (args) => void      // local 类型：直接执行
  getPromptForCommand?: (args) => string  // prompt 类型：展开为提示词
  isEnabled?: () => boolean
  source?: 'builtin' | 'skill' | 'plugin'
  loadedFrom?: string
  kind?: string
}
```

命令来源（`getCommands` memoized）：
1. 内置命令（静态导入 ~70 个 + 特性门控）
2. Skill 贡献（用户/项目/插件 Skills 转为 prompt 命令）
3. MCP 贡献（MCP prompts 命令）
4. 插件命令

### 1.2 核心命令（实际存在）

| 命令 | 描述 | 说明 |
|------|------|------|
| `/help` | 帮助 | 列出所有可用命令 |
| `/clear` | 清除 | 清除当前对话历史 |
| `/compact` | 压缩 | 手动触发上下文压缩 |
| `/cost` | 成本 | 显示当前会话成本 |
| `/model` | 模型 | 切换模型（/model opus 等） |
| `/resume` | 恢复 | 恢复历史会话（也有 CLI `--resume`） |
| `/theme`、`/color` | 主题 | 切换终端主题 |
| `/vim` | Vim | 开关 Vim 输入模式 |
| `/export` | 导出 | 导出会话 |
| `/agents` | Agent | 管理 Agent/团队 |
| `/mcp` | MCP | 管理 MCP 服务器 |
| `/doctor` | 诊断 | 诊断持久化状态 |
| `/exit` | 退出 | 退出 CLI |
| `/status` | 状态 | 显示版本/账号/工作区信息 |
| `/feedback` | 反馈 | 提交反馈 |
| `/bug` | 报障 | 提交 bug 报告 |

（完整 90+ 命令见 `src/commands.ts` 与 `src/commands/` 目录；另有 ant 内部命令如 `commit`，不在外部版本帮助中显示。）

### 1.3 命令分发

REPL 捕获输入后（`src/screens/REPL.tsx:3158`）以 `/` 前缀判定命令，按 name/alias 匹配：

```typescript
if (!speculationAccept && input.trim().startsWith('/')) {
  const matchingCommand = commands.find(cmd =>
    isCommandEnabled(cmd) &&
    (cmd.name === commandName ||
     cmd.aliases?.includes(commandName) ||
     getCommandName(cmd) === commandName))
  // prompt 型：把命令展开为提示词走正常 query 流程
  // local 型：直接调用 handler
}
```

安全白名单控制远程/bridge 场景下可用的命令（`REMOTE_SAFE_COMMANDS`、`BRIDGE_SAFE_COMMANDS`，见第 14 章）。

### 1.4 `/commit` 命令（prompt 型示例）

`/commit` 是 ant 内部命令（`src/commands/commit.ts`），本质是一段精心设计的提示词，把 git 上下文（status/diff/branch/log 通过 `!` 反引号命令内联）连同安全协议（never amend、never skip hooks、不提交疑似密钥文件等）交给模型执行：

```typescript
const ALLOWED_TOOLS = [
  'Bash(git add:*)',
  'Bash(git status:*)',
  'Bash(git commit:*)',
]

function getPromptContent(): string {
  return `## Context
- Current git status: !\`git status\`
- Current git diff: !\`git diff HEAD\`
- Recent commits: !\`git log --oneline -10\`
## Git Safety Protocol
- NEVER update the git config
- NEVER use git commit --amend unless explicitly requested
- Do not commit files that likely contain secrets
...
## Your task
1. Analyze changes and draft a commit message following repo style
2. Stage relevant files and commit using HEREDOC syntax`
}
```

没有独立的 linter 自动修复环节——linter/测试等由模型按需通过 Bash 工具调用。

## 2. Skills 系统 (`src/skills/`)
## 2. Skills 系统 (`src/skills/`)

### 2.1 概念

Skills 是一种可扩展的能力模块，允许：
- 添加新的命令
- 定义复杂的 workflow
- 封装常用的操作序列

### 2.2 Skill 定义（Markdown + frontmatter）

Skill 是**声明式**的能力扩展：一个目录（或单文件）Markdown，正文即指令内容（真实解析见 `src/skills/loadSkillsDir.ts:197-260`）：

```markdown
---
name: code-review
description: 执行代码审查，检查安全、性能和可维护性问题
when_to_use: 当用户要求审查代码或 PR 时
allowed-tools: Bash, Read, Grep, Glob   # 可选：限制可用工具
model: opus                              # 可选：指定模型
disable-model-invocation: false          # 可选：是否禁止模型主动调用
argument-hint: "[files]"                 # 可选：参数提示
context: fork                            # 可选：fork 上下文执行
---

# 代码审查 Skill

你是一个专业的代码审查专家。请按照以下步骤进行审查：
...
```

没有 `triggers`/`version`/`kind` 字段，也没有 `execute()` 程序入口——Skill 本质是"带元数据的提示词包"，由 SkillTool/命令系统在调用时注入。

### 2.3 Skill 目录结构

```
.claude/skills/
  ├── my-skill/
  │   ├── SKILL.md           # 主文件（或与目录同名 .md）
  │   └── references/        # 可选：参考文件（按需加载）
  ├── code-review.md         # 单文件 Skill
```

### 2.4 Skill 加载来源与优先级

`loadSkillsDir.ts`（memoized）按来源合并去重（后加载覆盖先加载）：

1. 内置 Skills（`src/skills/bundled/`：verify、keybindings、remember、loop 等）
2. 用户全局（`~/.claude/skills/`）
3. 项目（`.claude/skills/`）
4. 插件 Skills
5. 托管 Skills（企业策略）

### 2.5 Skill 执行

Skill 以两种方式触达模型：

- **命令方式**：用户输入 `/skill-name`（Skill 注册为 prompt 型命令，正文作为提示词展开）
- **SkillTool 方式**（工具名 `Skill`，`src/tools/SkillTool/SkillTool.ts:331`）：模型根据 description 判断适用场景，主动调用并注入 Skill 内容

内置 bundled Skills 走 `src/skills/bundledSkills.ts` 的 `registerBundledSkill()` 程序化注册。

### 2.6 MCP→Skill 转换

MCP 服务器的 prompts 会被转换为 Skill/命令（`src/skills/mcpSkillBuilders.ts`），MCP tools 命名空间 `mcp__server__tool` 一并进入工具池。

## 3. 内置 Skills 示例

`src/skills/bundled/` 内置了若干 Skills：`verify`（校验计划执行）、`keybindings`（快捷键管理）、`remember`（记忆）、`loop`、`hunter`、`simplify` 等。它们与用户 Skills 走完全相同的加载与调用机制，只是注册来源为 `bundled`。

（早期版本文档描述的 verify/update Skill 的 `execute()` 流程是虚构的——真实 Skill 没有 JS 执行入口。）

## 4. 命令执行流程

```
User Input: "/compact"
         │
         ▼
┌─────────────────┐
│  REPL.tsx       │  ← input.trim().startsWith('/') 判定
└────────┬────────┘
         │
    ┌────┴─────┐
    ▼          ▼
 匹配命令   未匹配 → 作为普通消息发给模型
    │
    ├─ prompt 型 → 展开提示词 → 走 query() 流程
    └─ local 型  → 直接调用 handler
```

## 5. Skill 与命令的协作

Skill 与命令是**同一机制的两个视角**：

- 用户视角：`/skill-name`（命令）
- 模型视角：`Skill` 工具按 description 匹配调用

Skill 正文注入对话后，模型通过常规工具（Bash/Read/Grep 等，受 `allowed-tools` 限制）完成实际工作。没有 `context.tools.Skill.execute()` 这类 JS API——Skill 内容只是提示词，执行全靠 Agent Loop 的工具调用。

## 6. 自定义命令开发

自定义命令的正确途径是 **Skill**（prompt 型命令）——没有 `.claude/commands/*.ts` 代码注册机制：

```markdown
<!-- .claude/skills/deploy.md -->
---
name: deploy
description: 部署当前项目到预发环境
argument-hint: "[env]"
allowed-tools: Bash(npm run deploy:*)
---

请将当前项目部署到 $ARGUMENTS 环境，步骤：
1. 运行构建
2. 执行部署脚本
3. 验证健康检查
```

保存后即可 `/deploy staging` 调用。frontmatter 与命令行为见第 12 章 Skills 深度解析。

## 7. 命令行补全

REPL 输入 `/` 时由 typeahead（`src/hooks/useTypeahead.tsx`）提供命令/文件名建议；命令的 `argument-hint` frontmatter 决定参数提示文案。补全候选来自 `getCommands()`（含 Skills/MCP/插件命令），无独立的补全规则配置文件。
