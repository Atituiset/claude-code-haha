# Ink TUI 渲染引擎详解

## 1. 概述

Claude Code 使用 [Ink](https://github.com/vadimdemedes/ink) 作为终端 UI 渲染引擎。Ink 是 React 的终端版本，允许使用 React 组件构建交互式 TUI。

## 2. 核心架构

### 2.1 Ink 与 React 的区别

| 特性 | React Web | Ink 终端 |
|------|----------|---------|
| 渲染目标 | DOM | 终端字符渲染 |
| 布局引擎 | CSS Flexbox | Yoga Layout (C++) |
| 事件 | DOM Events | Keyboard/Mouse Input |
| 更新机制 | Virtual DOM | Full re-render (controlled) |
| 测量 | CSSOM |Measured dimensions |

### 2.2 核心文件

```
src/ink/
├── root.ts           # Ink Root 创建和管理
├── render.ts         # 主渲染函数
├── dom.ts            # DOM 元素类型定义
├── components/       # 内置组件
│   ├── Box.tsx       # 容器组件
│   ├── Text.tsx      # 文本组件
│   ├── Button.tsx    # 按钮组件
│   └── ...
├── hooks/            # Ink Hooks
│   ├── use-input.ts  # 键盘输入
│   ├── use-stdin.ts  # 标准输入
│   ├── use-app.ts    # 应用上下文
│   └── ...
├── events/           # 事件系统
│   ├── input-event.ts
│   ├── click-event.ts
│   └── ...
├── focus.ts          # 焦点管理
├── frame.ts          # 帧管理
└── termio/           # 终端 I/O
```

## 3. 入口点 (`src/ink.ts`)

```typescript
import { render, createRoot, Box, Text, useInput, useStdin } from './ink/index.js'

// 渲染组件
const root = await createRoot()
await render(<MyComponent />, root)

// 或使用快捷方式
await render(<MyComponent />, process.stdout)
```

## 4. 核心组件

### 4.1 Box 组件

Box 是基础容器组件，类似于 CSS Flexbox：

```typescript
import { Box, Text } from 'ink'

function Layout() {
  return (
    <Box flexDirection="column" gap={1}>
      <Box>
        <Text>Header</Text>
      </Box>
      <Box flexGrow={1}>
        <Text>Content</Text>
      </Box>
      <Box>
        <Text>Footer</Text>
      </Box>
    </Box>
  )
}
```

**常用属性：**

| 属性 | 类型 | 描述 |
|------|------|------|
| `flexDirection` | `row` \| `column` | 主轴方向 |
| `gap` | `number` | 元素间距 |
| `flexGrow` | `number` | 扩展比例 |
| `flexShrink` | `number` | 收缩比例 |
| `width` | `number` \| `string` | 宽度 |
| `height` | `number` \| `string` | 高度 |
| `padding` | `number` | 内边距 |
| `margin` | `number` | 外边距 |
| `borderStyle` | `BorderStyle` | 边框样式 |
| `backgroundColor` | `string` | 背景色 |

### 4.2 Text 组件

文本组件，支持颜色和样式：

```typescript
import { Box, Text } from 'ink'

function StyledText() {
  return (
    <Box>
      <Text color="green" bold>
        Success!
      </Text>
      <Text color="red" dim>
        Error!
      </Text>
      <Text underline inverse>
        Important
      </Text>
    </Box>
  )
}
```

**常用属性：**

| 属性 | 类型 | 描述 |
|------|------|------|
| `color` | `string` | 文本颜色 |
| `backgroundColor` | `string` | 背景色 |
| `bold` | `boolean` | 加粗 |
| `dim` | `boolean` | 暗淡 |
| `italic` | `boolean` | 斜体 |
| `underline` | `boolean` | 下划线 |
| `inverse` | `boolean` | 反色 |

### 4.3 自定义组件

可以使用 `React.Component` 或函数组件：

```typescript
import { Box, Text } from 'ink'
import React from 'react'

// 函数组件
function Message({ text, sender }: { text: string; sender: 'user' | 'assistant' }) {
  return (
    <Box justifyContent={sender === 'user' ? 'flex-end' : 'flex-start'}>
      <Box
        borderStyle="round"
        padding={1}
        backgroundColor={sender === 'user' ? 'cyan' : 'gray'}
      >
        <Text color="white">{text}</Text>
      </Box>
    </Box>
  )
}
```

## 5. Hooks

### 5.1 useInput

处理键盘输入：

```typescript
import { useInput } from 'ink'

function InputHandler() {
  useInput((input, key) => {
    // 输入的字符
    console.log('Input:', input)
    
    // 功能键
    if (key.return) {
      console.log('Enter pressed')
    }
    if (key.escape) {
      console.log('Escape pressed')
    }
    if (key.ctrl && input === 'c') {
      console.log('Ctrl+C')
    }
    
    // 方向键
    if (key.upArrow) {}
    if (key.downArrow) {}
  })
  
  return <Text>Press any key...</Text>
}
```

**key 对象属性：**

| 属性 | 类型 |
|------|------|
| `return` | `boolean` | 回车 |
| `escape` | `boolean` | ESC |
| `ctrl` | `boolean` | Ctrl 修饰键 |
| `shift` | `boolean` | Shift 修饰键 |
| `alt` | `boolean` | Alt 修饰键 |
| `upArrow/downArrow/leftArrow/rightArrow` | `boolean` | 方向键 |
| `backspace` | `boolean` | 退格 |
| `tab` | `boolean` | Tab |

### 5.2 useStdin

访问标准输入：

```typescript
import { useStdin } from 'ink'

function StdinReader() {
  const { stdin, isRawModeSupported, setRawMode } = useStdin()
  
  // stdin 是标准输入流，可监听数据事件（实际项目里用 useInput 更常见）
  stdin?.on('data', (data) => {
    console.log('Received:', data.toString())
  })
  
  return <Text>Reading stdin...</Text>
}
```

### 5.3 useApp

访问应用上下文（注意：实际只暴露 `exit`，没有 `stdout`）：

```typescript
// src/ink/hooks/use-app.ts
const useApp = () => useContext(AppContext)
// AppContext = { exit: (error?: Error) => void }
```

### 5.4 useAnimationFrame

用于需要每帧更新的场景（实现位于 `src/ink/hooks/use-animation-frame.ts`）：

```typescript
import { useAnimationFrame } from 'ink'

function AnimatedClock() {
  const [time, setTime] = useState(new Date())
  
  useAnimationFrame((deltaTime) => {
    setTime(new Date())
  })
  
  return <Text>{time.toISOString()}</Text>
}
```

## 6. 焦点管理

Ink 支持焦点管理，允许组件获取焦点响应键盘事件：

```typescript
import { FocusManager, Box, Text, useInput } from 'ink'

function FocusableInput() {
  const [value, setValue] = useState('')
  
  useInput((input) => {
    setValue(prev => prev + input)
  })
  
  return (
    <Box 
      borderStyle="round" 
      padding={1}
      justifyContent="center"
    >
      <Text>Input: {value}_</Text>
    </Box>
  )
}

function App() {
  return (
    <FocusManager>
      <FocusableInput />
      <FocusableInput />
    </FocusManager>
  )
}
```

## 7. 主题系统

Claude Code 在 `src/components/design-system/` 中实现了主题系统：

```typescript
import { ThemeProvider, useTheme } from 'ink'

function ThemedComponent() {
  const theme = useTheme()
  
  return (
    <Box 
      backgroundColor={theme.colors.background}
      padding={2}
    >
      <Text color={theme.colors.text}>
        Themed Text
      </Text>
    </Box>
  )
}

function App() {
  return (
    <ThemeProvider theme={customTheme}>
      <ThemedComponent />
    </ThemeProvider>
  )
}
```

## 8. 与 React Web 的主要区别

### 8.1 渲染方式

**React Web:**
```jsx
// Virtual DOM diffing，批量更新到真实 DOM
setState({ count: 1 }) // 触发重新渲染，自动 diff
```

**Ink:**
```tsx
// 同样是 React reconciler：setState 触发 re-render，
// Ink 把输出 diff 到终端字符网格（而非 DOM）
// 应用代码不需要手动调用 render()
const root = await createRoot()
await render(<App />, root)
// 之后组件内 setState 即可自动重绘
```

### 8.2 生命周期

**React Web:** `componentDidMount` -> `componentDidUpdate` -> `componentWillUnmount`

**Ink:** 同样基于 React reconciler（`src/ink/reconciler.ts`），生命周期与 React 一致；函数组件使用 `useEffect`：

```typescript
import { useEffect } from 'react'

function MyComponent() {
  useEffect(() => {
    console.log('Mounted')
    return () => console.log('Unmounted')
  }, [])
  
  return <Text>Hello</Text>
}
```

### 8.3 样式系统

**React Web:** CSS、Styled Components、Emotion 等

**Ink:** 内联属性、CSS-like：

```typescript
<Box
  flexDirection="column"
  padding={2}
  borderStyle="round"
  backgroundColor="blue"
/>
```

## 9. Claude Code 中的 TUI 架构

REPL 的实际组件树（`src/screens/REPL.tsx`，~5000 行）比示意结构复杂得多，核心包括：

```
src/screens/REPL.tsx
  ├─> App (components/App.tsx, FpsMetricsProvider 等包裹)
  ├─> VirtualMessageList (虚拟滚动消息列表)
  ├─> PromptInput (components/PromptInput/, 输入框 + 各类选择器)
  ├─> TeammateViewHeader (队友视图)
  └─> 各类对话框 (权限/主题/模型选择等, components/*Dialog.tsx)
```

### 9.1 状态响应式更新

TUI 侧通过 React hooks + 自研轻量 store 订阅状态（`src/state/store.ts` 提供 `createStore`）：

```typescript
// 组件中通过 useAppState 订阅
function MessageList() {
  const messages = useAppState(s => s.messages)
  
  return (
    <Box flexDirection="column">
      {messages.map(msg => (
        <MessageItem key={msg.id} message={msg} />
      ))}
    </Box>
  )
}
```

状态写入的副作用集中在 `onChangeAppState`（`src/state/onChangeAppState.ts:43`）：

```typescript
// src/state/onChangeAppState.ts
export function onChangeAppState({
  newState,
  oldState,
}: {
  newState: AppState
  oldState: AppState
}) {
  // 权限模式变化 → 通知 CCR 外部元数据与 SDK 状态流
  // 模型变化 → 写回用户设置
  // expandedView 变化 → 持久化 showExpandedTodos / showSpinnerTree
  // ... 其余 diff 副作用
}
```

它不是"手动触发 Ink 重渲染"的函数——渲染由 React reconciler 的正常订阅机制驱动。

## 10. 调试技巧

### 10.1 查看渲染树

```tsx
<Box debug>
  <Text>Inspect me</Text>
</Box>
```

### 10.2 测量组件

`measureElement` 是同步方法，通过 ref 调用（不是 await）：

```typescript
import { measureElement } from '../ink.js'

function MyComponent() {
  const ref = useRef<DOMElement>(null)
  useEffect(() => {
    const dimensions = measureElement(ref.current)
    // → { width: number, height: number }
  }, [])
  return <Box ref={ref}>...</Box>
}
```
