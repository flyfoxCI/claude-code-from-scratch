# 第 7 章：实战 BashTool — 构建生产级工具

> "BashTool 是 Claude Code 最复杂的内置工具，也是理解工具构建的最佳范例。"

## 7.1 概述

BashTool 允许 Agent 执行 shell 命令，是 Agent 与操作系统交互的主要窗口。它的实现包含：

- **命令解析与分类** — 判断命令是否只读
- **权限管理** — 用户确认机制
- **结果渲染** — UI 显示优化
- **进度报告** — 长时间运行的命令
- **错误处理** — 各种异常场景
- **沙箱支持** — 安全隔离

## 7.2 命令分类系统

### 7.2.1 为什么需要分类？

LLM 需要知道一个命令是否会修改系统，以便：
1. **选择是否并行执行**
2. **决定是否需要权限确认**
3. **在 UI 中显示合适的图标**

### 7.2.2 分类定义

```typescript
// src/tools/BashTool/BashTool.tsx:54-82

// 搜索命令 — 网络搜索、文件内容搜索
const BASH_SEARCH_COMMANDS = new Set([
  'find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis'
])

// 读取命令 — 查看文件内容
const BASH_READ_COMMANDS = new Set([
  'cat', 'head', 'tail', 'less', 'more',
  // 分析命令
  'wc', 'stat', 'file', 'strings',
  // 数据处理
  'jq', 'awk', 'cut', 'sort', 'uniq', 'tr'
])

// 目录列表命令
const BASH_LIST_COMMANDS = new Set(['ls', 'tree', 'du'])

// 语义中性命令 — 不影响命令的读写性质
const BASH_SEMANTIC_NEUTRAL_COMMANDS = new Set([
  'echo', 'printf', 'true', 'false', ':'  // bash no-op
])

// 静默命令 — 成功时无输出
const BASH_SILENT_COMMANDS = new Set([
  'mv', 'cp', 'rm', 'mkdir', 'rmdir',
  'chmod', 'chown', 'chgrp', 'touch', 'ln', 'cd',
  'export', 'unset', 'wait'
])
```

### 7.2.3 isSearchOrReadBashCommand 实现

这是 BashTool 的核心算法之一：

```typescript
// src/tools/BashTool/BashTool.tsx:95-172
export function isSearchOrReadBashCommand(command: string): {
  isSearch: boolean
  isRead: boolean
  isList: boolean
} {
  // 1. 解析命令，支持管道和操作符
  let partsWithOperators: string[]
  try {
    partsWithOperators = splitCommandWithOperators(command)
  } catch {
    // 解析失败 = 非只读命令
    return { isSearch: false, isRead: false, isList: false }
  }

  if (partsWithOperators.length === 0) {
    return { isSearch: false, isRead: false, isList: false }
  }

  let hasSearch = false
  let hasRead = false
  let hasList = false
  let hasNonNeutralCommand = false

  // 2. 遍历命令部分
  for (const part of partsWithOperators) {
    // 跳过操作符和重定向
    if (part === '>' || part === '>>' || part === '>&') continue
    if (part === '||' || part === '&&' || part === '|' || part === ';') continue

    const baseCommand = part.trim().split(/\s+/)[0]
    if (!baseCommand) continue

    // 3. 语义中性命令跳过
    if (BASH_SEMANTIC_NEUTRAL_COMMANDS.has(baseCommand)) continue

    hasNonNeutralCommand = true

    // 4. 检查命令类别
    const isPartSearch = BASH_SEARCH_COMMANDS.has(baseCommand)
    const isPartRead = BASH_READ_COMMANDS.has(baseCommand)
    const isPartList = BASH_LIST_COMMANDS.has(baseCommand)

    // 5. 如果任何一个不是只读，整个命令就不是只读
    if (!isPartSearch && !isPartRead && !isPartList) {
      return { isSearch: false, isRead: false, isList: false }
    }

    if (isPartSearch) hasSearch = true
    if (isPartRead) hasRead = true
    if (isPartList) hasList = true
  }

  // 只有纯输出/状态命令（如 echo "hello"）不算只读
  if (!hasNonNeutralCommand) {
    return { isSearch: false, isRead: false, isList: false }
  }

  return { isSearch: hasSearch, isRead: hasRead, isList: hasList }
}
```

**关键洞察**：

1. **管道分析** — `cat file | grep pattern` 是只读的
2. **操作符处理** — `&&`, `||`, `;` 连接的命令需要全部只读
3. **重定向跳过** — `>` 目标不是命令，不影响分类
4. **语义中性** — `echo` 等命令在管道中被忽略

## 7.3 权限检查

### 7.3.1 权限规则

```typescript
// src/tools/BashTool/bashPermissions.ts
import { bashToolHasPermission, commandHasAnyCd } from './bashPermissions'

async function call(input, context, canUseTool, parentMessage) {
  // 1. 检查是否有缓存的权限决策
  const cachedDecision = context.toolDecisions?.get(input.command)
  if (cachedDecision) {
    if (cachedDecision.decision === 'accept') {
      return executeCommand(input, context)
    }
    // 否则需要重新请求权限
  }

  // 2. 检查权限规则
  const hasPermission = await bashToolHasPermission({
    command: input.command,
    context,
  })

  if (!hasPermission) {
    // 请求用户确认
    const result = await canUseTool({
      toolName: 'Bash',
      toolInput: input,
    })
    // ...
  }
}
```

### 7.3.2 命令前缀匹配

```typescript
// src/tools/BashTool/bashPermissions.ts
function matchWildcardPattern(pattern: string, command: string): boolean {
  // 支持通配符模式
  // 如 "git *" 匹配所有 git 命令
}

function permissionRuleExtractPrefix(rule: string): string {
  // 提取规则前缀
  // 如 "git commit" -> "git"
}
```

## 7.4 执行与监控

### 7.4.1 本地 Shell 任务

Claude Code 使用 `LocalShellTask` 管理长时间运行的命令：

```typescript
// src/tools/BashTool/BashTool.tsx
import {
  spawnShellTask,
  registerForeground,
  backgroundExistingForegroundTask,
} from '../../tasks/LocalShellTask/LocalShellTask.js'

// 启动后台任务
const task = spawnShellTask({
  command: input.command,
  workingDirectory: input.workingDirectory || cwd,
  timeoutMs: getTimeout(input.timeout),
  onOutput: (data) => {
    // 实时收集输出
    outputAccumulator.append(data)
  },
  signal: context.abortController.signal,
})

// 注册前台任务
registerForeground(task)

// 或者后台运行
if (shouldBackground) {
  backgroundExistingForegroundTask(task)
}
```

### 7.4.2 进度报告

```typescript
// 长时间运行的命令，每 2 秒报告一次进度
const PROGRESS_THRESHOLD_MS = 2000

let lastProgressReport = Date.now()
let startTime = Date.now()

onProgress?.({
  toolUseID: context.toolUseId!,
  data: {
    type: 'bash_progress',
    partialOutput: truncate(output, PREVIEW_SIZE_BYTES),
    runningTimeMs: Date.now() - startTime,
  },
})
```

## 7.5 结果处理

### 7.5.1 结果累积

```typescript
// src/utils/stringUtils.ts
export class EndTruncatingAccumulator {
  private content: string = ''
  private maxLength: number

  append(chunk: string): void {
    if (this.content.length < this.maxLength) {
      this.content += chunk
    }
  }

  getContent(): string {
    return this.content
  }
}
```

### 7.5.2 结果截断

```typescript
// src/tools/BashTool/utils.ts
const TOOL_RESULT_MAX_LENGTH = 100_000  // 100KB

export function truncateOutput(output: string): {
  truncated: boolean
  content: string
  preview: string
} {
  if (output.length <= TOOL_RESULT_MAX_LENGTH) {
    return { truncated: false, content: output, preview: output }
  }

  return {
    truncated: true,
    content: output.slice(0, TOOL_RESULT_MAX_LENGTH),
    preview: output.slice(0, PREVIEW_SIZE_BYTES) + '...[truncated]',
  }
}
```

### 7.5.3 错误解释

```typescript
// src/tools/BashTool/commandSemantics.ts
export function interpretCommandResult(
  result: ExecResult,
): { isError: boolean; message: string } {
  //ENOENT - 文件不存在
  if (result.exitCode === 127) {
    return { isError: true, message: 'Command not found' }
  }

  // 权限拒绝
  if (result.exitCode === 126) {
    return { isError: true, message: 'Permission denied' }
  }

  // 其他非零退出码
  if (result.exitCode !== 0) {
    return { isError: true, message: `Exit code: ${result.exitCode}` }
  }

  return { isError: false, message: '' }
}
```

## 7.6 UI 渲染

### 7.6.1 渲染函数

```typescript
// src/tools/BashTool/UI.ts
export function renderToolUseMessage(input: BashInput): string {
  const truncated = truncateCommand(input.command)
  return `<bash>${escapeHtml(truncated)}</bash>`
}

export function renderToolResultMessage(
  result: BashResult,
  options: RenderOptions,
): string {
  if (result.exitCode === 0 && isSilentCommand(result.command)) {
    return `<bash_result>Done</bash_result>`
  }

  if (result.truncated) {
    return `<bash_result>
Command output truncated (${result.truncatedSize} bytes removed)
${result.preview}
</bash_result>`
  }

  return `<bash_result>${escapeHtml(result.stdout)}</bash_result>`
}
```

### 7.6.2 搜索结果折叠

```typescript
// 如果是搜索命令，显示简洁摘要
if (isSearch) {
  const lineCount = result.stdout.split('\n').length
  return `<bash_result>Found ${lineCount} matches</bash_result>`
}

// 如果是读取命令，显示行数
if (isRead) {
  const lineCount = result.stdout.split('\n').length
  return `<bash_result>${lineCount} lines</bash_result>`
}
```

## 7.7 完整示例

让我们看一个简化的完整调用流程：

```typescript
// BashTool.call 的简化版本
async function call(input, context, canUseTool, parentMessage, onProgress) {
  // 1. 权限检查
  const hasPermission = await bashToolHasPermission({
    command: input.command,
    context,
  })

  if (!hasPermission) {
    const canUse = await canUseTool({ toolName: 'Bash', toolInput: input })
    if (!can.use) {
      return { data: { error: 'Permission denied' } }
    }
  }

  // 2. 启动任务
  const task = spawnShellTask({
    command: input.command,
    workingDirectory: input.workingDirectory,
    timeoutMs: getTimeout(input.timeout),
    signal: context.abortController.signal,
  })

  // 3. 收集输出
  const accumulator = new EndTruncatingAccumulator(TOOL_RESULT_MAX_LENGTH)

  task.onOutput((data) => {
    accumulator.append(data)
    onProgress?.({ toolUseID: context.toolUseId!, data: { partialOutput: data } })
  })

  // 4. 等待完成
  const result = await task.wait()

  // 5. 解释结果
  const interpreted = interpretCommandResult(result)

  return {
    data: {
      stdout: accumulator.getContent(),
      stderr: result.stderr,
      exitCode: result.exitCode,
      interpreted,
    },
    newMessages: interpreted.isError
      ? [createSystemMessage(interpreted.message, 'warning')]
      : [],
  }
}
```

## 7.8 设计模式总结

### 7.8.1 命令语义分析模式

```typescript
// 核心模式：组合多个分类器
function analyzeCommand(command: string): CommandAnalysis {
  return {
    isRead: isReadCommand(command),
    isWrite: isWriteCommand(command),
    isSearch: isSearchCommand(command),
    isDangerous: isDangerousCommand(command),
  }
}

// 使用：多个条件组合判断
if (analysis.isRead && !analysis.isDangerous) {
  // 允许并行执行
}
```

### 7.8.2 进度延迟报告模式

```typescript
// 只有超过阈值才报告，避免噪音
const PROGRESS_THRESHOLD_MS = 2000

let lastReport = 0
onOutput(data) {
  const now = Date.now()
  if (now - lastReport > PROGRESS_THRESHOLD_MS) {
    reportProgress(data)
    lastReport = now
  }
}
```

### 7.8.3 结果累积与截断模式

```typescript
// 无限累积但限制总长度
class TruncatingAccumulator {
  private content: string = ''
  private readonly maxLength = 100_000

  append(chunk: string) {
    if (this.content.length < this.maxLength) {
      this.content += chunk.slice(0, this.maxLength - this.content.length)
    }
  }
}
```

## 7.9 本章小结

BashTool 展示了构建生产级工具的关键要素：

| 要素 | 实现 |
|------|------|
| 命令分类 | `isSearchOrReadBashCommand` 分析命令语义 |
| 权限管理 | `bashToolHasPermission` + `canUseTool` |
| 进度报告 | `onProgress` 回调 + 延迟阈值 |
| 结果处理 | 累积器 + 截断 + 预览生成 |
| 错误解释 | `interpretCommandResult` 语义化错误 |
| UI 渲染 | 分离的渲染函数 |

**核心原则**：

1. **分类要准确** — LLM 依赖这些信息做决策
2. **渐进式反馈** — 用户需要知道长时间运行命令的进展
3. **优雅截断** — 避免大输出撑爆上下文
4. **语义错误** — 不仅仅报告退出码，要解释含义

## 7.10 实践要点

1. **实现 `isSearchOrReadBashCommand`** — 这是最容易被低估的函数
2. **使用 `EndTruncatingAccumulator`** — 避免内存问题
3. **进度阈值** — 2 秒是合理的默认值
4. **处理中止信号** — 用户可能随时取消

---

## 下一步

下一章我们将学习 **工具执行引擎**，了解：
- 工具如何并行执行
- 读写工具的分离策略
- 流式工具执行器
