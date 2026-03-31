# 第 20 章：CLI 与 REPL — Command Line Interface

> "CLI 是用户与 Agent 交互的窗口。"

## 20.1 CLI 架构概述

Claude Code 的 CLI 分为两个阶段：

```
┌─────────────────────────────────────────────────────────┐
│                   Bootstrap Phase                        │
│            (src/entrypoints/cli.tsx)                      │
│  - 解析命令行参数                                          │
│  - 处理快速命令 (--version, --help)                       │
│  - 路由到正确的入口                                       │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    Main Phase                            │
│              (src/main.tsx / REPL)                       │
│  - 初始化应用                                            │
│  - 启动交互循环                                          │
└─────────────────────────────────────────────────────────┘
```

## 20.2 快速命令处理

### 20.2.1 Bootstrap CLI

```typescript
// src/entrypoints/cli.tsx
async function bootstrap(args: string[]): Promise<void> {
  const [command, ...rest] = args

  // 快速命令：不需要加载完整应用
  switch (command) {
    case '--version':
    case '-v':
      console.log(getVersion())
      return

    case '--help':
    case '-h':
      printHelp()
      return

    case '--dump-system-prompt':
      await dumpSystemPrompt()
      return

    case '--claude-in-chrome-mcp':
      await runChromeMcpMode()
      return

    case 'remote-control':
    case 'bridge':
      await runRemoteControlMode(rest)
      return

    case 'daemon':
      await runDaemonMode(rest)
      return

    case 'ps':
      await listSessions()
      return

    case 'logs':
      await showLogs(rest)
      return

    case 'attach':
      await attachToSession(rest)
      return

    case 'kill':
      await killSession(rest)
      return
  }

  // 不是快速命令，启动完整应用
  await main(args)
}
```

### 20.2.2 命令行参数

```bash
# 版本
claude --version
claude -v

# 帮助
claude --help

# 转储系统提示词
claude --dump-system-prompt

# 会话管理
claude ps
claude logs <session-id>
claude attach <session-id>
claude kill <session-id>

# 启动新会话
claude "帮我写一个 React 组件"
```

## 20.3 主入口

### 20.3.1 main 函数

```typescript
// src/main.tsx
export async function main(args: string[]): Promise<void> {
  // 1. 初始化配置
  const config = await loadConfig()

  // 2. 初始化遥测
  await initTelemetry()

  // 3. 初始化 GrowthBook
  await initGrowthBook()

  // 4. 初始化设置
  const settings = await loadSettings()

  // 5. 检查会话恢复
  if (config.resumeSession) {
    await resumeSession(config.sessionId)
  }

  // 6. 启动 REPL 或 headless 模式
  if (config.headless) {
    await runHeadless(config)
  } else {
    await replLauncher.launch()
  }
}
```

## 20.4 REPL 循环

### 20.4.1 REPL 启动

```typescript
// src/replLauncher.tsx
export async function launch(): Promise<void> {
  // 1. 初始化状态
  await initAppState()

  // 2. 初始化 MCP 服务器
  await initMcpClients()

  // 3. 初始化插件
  await loadPlugins()

  // 4. 启动 REPL 循环
  await runReplLoop()
}
```

### 20.4.2 REPL 循环

```typescript
// src/repl.ts
async function runReplLoop(): Promise<void> {
  const reader = createInterface({
    input: process.stdin,
    output: process.stdout,
  })

  while (true) {
    try {
      // 1. 显示提示符
      const prompt = getPrompt()
      const input = await reader.question(prompt)

      // 2. 处理输入
      if (input === '/exit' || input === '/quit') {
        break
      }

      if (input.startsWith('/')) {
        await handleSlashCommand(input)
      } else {
        await processUserInput(input)
      }

    } catch (error) {
      console.error('Error:', error.message)
    }
  }

  reader.close()
}
```

## 20.5 命令处理

### 20.5.1 斜杠命令

```typescript
// src/commands.ts
interface Command {
  name: string
  description: string
  aliases?: string[]
  handler: (args: string[]) => Promise<void>
  completions?: string[]  // 自动补全
}

const commands: Command[] = [
  {
    name: '/help',
    description: 'Show help',
    aliases: ['/h', '/?'],
    handler: async () => {
      printHelp()
    },
  },
  {
    name: '/clear',
    description: 'Clear conversation',
    aliases: ['/cl'],
    handler: async () => {
      await clearConversation()
    },
  },
  {
    name: '/compact',
    description: 'Compact context',
    aliases: ['/co'],
    handler: async () => {
      await compactContext()
    },
  },
  {
    name: '/model',
    description: 'Switch model',
    aliases: ['/m'],
    handler: async (args) => {
      await switchModel(args[0])
    },
  },
  // ... 更多命令
]
```

### 20.5.2 命令执行

```typescript
// src/repl.ts
async function handleSlashCommand(input: string): Promise<void> {
  const [command, ...args] = input.split(' ')
  const trimmed = command.trim()

  const cmd = commands.find(
    c => c.name === trimmed || c.aliases?.includes(trimmed)
  )

  if (!cmd) {
    console.error(`Unknown command: ${trimmed}`)
    return
  }

  await cmd.handler(args)
}
```

## 20.6 输出渲染

### 20.6.1 终端渲染

```typescript
// src/cli/print.ts
export function printAssistantMessage(message: AssistantMessage): void {
  for (const block of message.message.content) {
    switch (block.type) {
      case 'text':
        printText(block.text)
        break

      case 'tool_use':
        printToolUse(block)
        break

      case 'thinking':
        printThinking(block.thinking)
        break
    }
  }
}

export function printToolUse(tool: ToolUseBlock): void {
  const icon = getToolIcon(tool.name)
  console.log(`${icon} ${tool.name}`)

  if (tool.input) {
    console.log(formatJSON(tool.input))
  }
}
```

### 20.6.2 颜色支持

```typescript
// src/utils/colors.ts
const colors = {
  reset: '\x1b[0m',
  bold: '\x1b[1m',
  dim: '\x1b[2m',

  black: '\x1b[30m',
  red: '\x1b[31m',
  green: '\x1b[32m',
  yellow: '\x1b[33m',
  blue: '\x1b[34m',
  cyan: '\x1b[36m',

  bgBlack: '\x1b[40m',
  bgGreen: '\x1b[42m',
}

export function color(text: string, color: string): string {
  return `${colors[color]}${text}${colors.reset}`
}
```

## 20.7 自动补全

### 20.7.1 Tab 补全

```typescript
// src/cli/complete.ts
export function setupCompletions(): void {
  const completer = (line: string): [string[], string] => {
    // 斜杠命令补全
    if (line.startsWith('/')) {
      const input = line.slice(1)
      const matches = commands
        .filter(c => c.name.startsWith(input) || c.aliases?.some(a => a.startsWith(input)))
        .map(c => '/' + c.name)
      return [matches, line]
    }

    // 文件路径补全
    if (line.includes(' ')) {
      const [command, ...args] = line.split(' ')
      if (command === 'Read' || command === 'Edit') {
        return completeFilePath(args.join(' '))
      }
    }

    return [[], line]
  }

  // 设置 readline 补全
  ;(process.stdin as any).setCompletions(completions)
}
```

## 20.8 会话管理

### 20.8.1 列出会话

```typescript
// src/cli/sessions.ts
export async function listSessions(): Promise<void> {
  const sessions = await getSessions()

  console.log('Active sessions:')
  for (const session of sessions) {
    const status = session.active ? 'active' : 'background'
    const time = formatDuration(Date.now() - session.startedAt)
    console.log(`  ${session.id} [${status}] ${time}`)
  }
}
```

### 20.8.2 附加到会话

```typescript
// src/cli/sessions.ts
export async function attachToSession(sessionId: string): Promise<void> {
  const session = await getSession(sessionId)

  if (!session) {
    console.error(`Session not found: ${sessionId}`)
    return
  }

  if (!session.active) {
    console.error(`Session is not active: ${sessionId}`)
    return
  }

  // 切换终端到该会话
  await attachTerminal(session)
}
```

## 20.9 信号处理

### 20.9.1 Ctrl+C 处理

```typescript
// src/cli/signals.ts
export function setupSignalHandlers(): void {
  // Ctrl+C - 中断当前操作
  process.on('SIGINT', () => {
    if (currentOperation) {
      abortCurrentOperation()
    } else {
      // 退出确认
      console.log('\nQuit? (y/n)')
      // 等待用户确认
    }
  })

  // Ctrl+Z - 后台挂起
  process.on('SIGTSTP', () => {
    suspendToBackground()
  })

  // 窗口大小变化
  process.on('SIGWINCH', () => {
    updateTerminalSize()
  })
}
```

## 20.10 设计模式

### 20.10.1 REPL 模板

```typescript
class REPL {
  private running = false

  async start(): Promise<void> {
    this.running = true
    await this.initialize()

    while (this.running) {
      try {
        const input = await this.read()
        const output = await this.evaluate(input)
        await this.print(output)
      } catch (error) {
        await this.handleError(error)
      }
    }

    await this.cleanup()
  }

  async stop(): Promise<void> {
    this.running = false
  }
}
```

### 20.10.2 命令注册

```typescript
// 注册命令的装饰器
function command(name: string, description: string) {
  return function (target: any, propertyKey: string) {
    commands.push({
      name,
      description,
      handler: target[propertyKey],
    })
  }
}

class MyCommands {
  @command('/hello', 'Say hello')
  async hello(args: string[]) {
    console.log('Hello!')
  }
}
```

## 20.11 本章小结

CLI 是 Claude Code 与用户交互的主要界面：

| 组件 | 职责 |
|------|------|
| `bootstrap` | 快速命令处理 |
| `main` | 主入口初始化 |
| `replLauncher` | REPL 启动 |
| `commands` | 斜杠命令注册 |
| `print` | 输出渲染 |
| `sessions` | 会话管理 |

**核心设计决策**：

1. **快速命令** — 避免加载完整应用
2. **模块化** — bootstrap 和 main 分离
3. **REPL 循环** — 交互式命令处理
4. **会话管理** — 支持后台和恢复

## 20.12 实践要点

1. **快速响应** — 简单命令不应该加载完整应用
2. **优雅退出** — 处理信号，正确清理资源
3. **用户友好** — 提供有用的错误信息和帮助
4. **会话持久化** — 支持后台和恢复

---

## 下一步

现在让我们进入附录部分，总结源码索引、设计模式和下一步学习路径。
