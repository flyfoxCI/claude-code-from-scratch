# 第 4 章：工具接口设计 — The Tool Interface

> "工具是 Agent 感知和改变世界的手臂。"

## 4.1 工具在 Agent 中的角色

在 Claude Code 的架构中，工具（Tools）不是可选的附加组件，而是 Agent 能力不可分割的一部分。LLM 本身只能生成文本，但通过工具，它可以：

- **读取文件系统** — 了解项目结构、代码内容
- **执行命令** — 运行测试、构建、部署
- **搜索网络** — 获取最新信息
- **调用 API** — 与外部服务交互
- **启动子 Agent** — 分解复杂任务

```
┌─────────────────────────────────────────────────────────┐
│                      LLM (大脑)                          │
│   "用户要我创建一个按钮组件"                              │
│   "我需要先看看现有的组件风格"                             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼ tool_use
┌─────────────────────────────────────────────────────────┐
│                   Tool Layer (手臂)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ FileRead │  │   Bash   │  │ GlobTool │              │
│  │  Tool    │  │   Tool   │  │          │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

## 4.2 Tool 接口定义

### 4.2.1 核心接口结构

Claude Code 的工具接口定义在 `src/Tool.ts:362-420`：

```typescript
// src/Tool.ts:362-420
export type Tool<
  Input extends AnyObject = AnyObject,  // 输入类型
  Output = unknown,                       // 输出类型
  P extends ToolProgressData = ToolProgressData, // 进度类型
> = {
  // === 标识 ===
  name: string

  // === 核心方法 ===
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  description(
    input: z.infer<Input>,
    options: {
      isNonInteractiveSession: boolean
      toolPermissionContext: ToolPermissionContext
      tools: Tools
    },
  ): Promise<string>

  // === Schema ===
  readonly inputSchema: Input
  outputSchema?: z.ZodType<unknown>

  // === 安全相关 ===
  isConcurrencySafe(input: z.infer<Input>): boolean
  isEnabled(): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean

  // === 元数据 ===
  aliases?: string[]                    //向后兼容的别名
  searchHint?: string                   //工具搜索关键词
  interruptBehavior?(): 'cancel' | 'block'  //中断行为
  requiresUserInteraction?(): boolean
}
```

### 4.2.2 关键字段解析

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 工具唯一标识，如 `"Bash"`, `"FileRead"` |
| `call` | `Function` | 工具的实际执行逻辑 |
| `description` | `Function` | 动态生成描述，供 LLM 理解何时使用 |
| `inputSchema` | `Zod Schema` | 输入参数验证 |
| `isConcurrencySafe` | `Function` | 是否可以与其他工具并行执行 |
| `isReadOnly` | `Function` | 是否只读操作 |
| `isDestructive` | `Function` | 是否为破坏性操作 |

## 4.3 输入验证 — Zod Schema

Claude Code 使用 [Zod](https://zod.dev) 进行运行时类型验证。

### 4.3.1 为什么用 Zod？

1. **TypeScript 集成** — 类型推导，自动完成
2. **运行时验证** — 防止无效输入
3. **LLM 友好的错误信息** — 可以生成用户友好的错误提示
4. **Schema 转换** — 轻松转换为 JSON Schema 给 LLM

### 4.3.2 BashTool 的输入 Schema

```typescript
// src/tools/BashTool/BashTool.tsx
import { z } from 'zod/v4'

const BashTool = buildTool({
  name: BASH_TOOL_NAME,

  inputSchema: z.object({
    command: z.string()
      .describe('The bash command to execute'),
    timeout: z.number().optional()
      .describe('Timeout in milliseconds'),
    workingDirectory: z.string().optional(),
  }),

  description: async (input, options) => {
    // 动态生成描述
    return `Execute a shell command: ${input.command}`
  },
})
```

### 4.3.3 FileReadTool 的 Schema

```typescript
// src/tools/FileReadTool/FileReadTool.tsx
inputSchema: z.object({
  file_path: z.string()
    .describe('Path to the file to read'),
  offset: z.number().optional()
    .describe('Line number to start reading from'),
  limit: z.number().optional()
    .describe('Maximum number of lines to read'),
  show_line_numbers: z.boolean().optional()
    .describe('Whether to show line numbers'),
}),
```

## 4.4 工具执行上下文 — ToolUseContext

### 4.4.1 上下文的作用

工具不是孤立执行的，它需要访问：

- **当前消息历史** — 了解对话状态
- **AbortController** — 支持取消
- **AppState** — 全局状态
- **文件缓存** — 追踪已读取的文件
- **权限上下文** — 权限检查

### 4.4.2 ToolUseContext 结构

```typescript
// src/Tool.ts:158-300
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools                    // 可用工具列表
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    refreshTools?: () => Tools      // 动态刷新工具
  }

  abortController: AbortController   // 支持取消
  readFileState: FileStateCache     // 文件读取缓存

  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void

  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setResponseLength: (f: (prev: number) => number) => void

  agentId?: AgentId                 // 子 Agent ID
  agentType?: string                // 子 Agent 类型
  messages: Message[]                // 当前消息历史

  // ... 更多字段
}
```

### 4.4.3 实际使用示例

```typescript
// 在工具中访问上下文
async function call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress,
): Promise<ToolResult<Output>> {
  // 读取全局状态
  const appState = context.getAppState()
  const settings = appState.settings

  // 检查是否取消
  if (context.abortController.signal.aborted) {
    throw new Error('Operation cancelled')
  }

  // 报告进度
  onProgress?.({
    toolUseID: context.toolUseId!,
    data: { type: 'progress', message: 'Reading file...' },
  })

  // 更新文件读取状态
  context.readFileState.recordRead(filePath)
}
```

## 4.5 工具结果 — ToolResult

### 5.5.1 返回结构

```typescript
// src/Tool.ts:321-336
export type ToolResult<T> = {
  data: T                          // 工具返回的数据

  // 可选：产生额外的消息
  newMessages?: (
    | UserMessage
    | AssistantMessage
    | AttachmentMessage
    | SystemMessage
  )[]

  // 可选：修改工具执行上下文
  contextModifier?: (context: ToolUseContext) => ToolUseContext

  // MCP 协议元数据
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

### 5.5.2 示例：返回结果和消息

```typescript
// BashTool 的执行结果
const result = await exec(command, { timeout })

return {
  data: {
    stdout: result.stdout,
    stderr: result.stderr,
    exitCode: result.exitCode,
  },
  newMessages: result.exitCode !== 0 ? [
    createSystemMessage(`Command failed with exit code ${result.exitCode}`, 'warning')
  ] : [],
}
```

## 4.6 工具元数据 — 描述与分类

### 4.6.1 动态描述

工具的 `description` 方法返回的字符串会出现在系统提示词中：

```typescript
// src/Tool.ts:386-393
description(
  input: z.infer<Input>,
  options: {
    isNonInteractiveSession: boolean
    toolPermissionContext: ToolPermissionContext
    tools: Tools
  },
): Promise<string>
```

**示例：不同输入，不同描述**

```typescript
// FileEditTool
description: async (input, options) => {
  if (input.replaceAll) {
    return `Replace all occurrences of "${input.old_string}" with "${input.new_string}" in ${input.file_path}`
  }
  return `Edit a specific section in ${input.file_path}:
- old_string: "${input.old_string}" (the exact text to find)
- new_string: "${input.new_string}" (the replacement text)`
}
```

### 4.6.2 搜索提示

```typescript
// src/Tool.ts:377-378
searchHint?: string  // "3–10 words, no trailing period"
```

用于 LLM 在工具被 defer 时搜索定位：

```typescript
// WebSearchTool
searchHint: 'web search internet research'
```

## 4.7 安全性与权限

### 4.7.1 并发安全 — isConcurrencySafe

判断工具是否可以并行执行：

```typescript
// FileReadTool
isConcurrencySafe: () => true  // 读取是安全的

// BashTool
isConcurrencySafe(input) {
  return isSearchOrReadBashCommand(input.command).isSearch ||
         isSearchOrReadBashCommand(input.command).isRead
}
```

**并发安全 = 可以与其他工具同时执行**

### 4.7.2 读写分类 — isReadOnly

```typescript
// BashTool
isReadOnly(input): boolean {
  return isSearchOrReadBashCommand(input.command).isSearch ||
         isSearchOrReadBashCommand(input.command).isRead ||
         isSearchOrReadBashCommand(input.command).isList
}
```

**读写分类影响 UI 显示和权限要求**

### 4.7.3 破坏性操作 — isDestructive

```typescript
// FileEditTool
isDestructive: (input) => {
  return input.old_string.length > 0 && input.new_string.length === 0
}
```

**破坏性工具会触发额外的确认提示**

## 4.8 进度报告 — Progress

### 4.8.1 进度回调

```typescript
// src/Tool.ts:338-340
export type ToolCallProgress<P extends ToolProgressData = ToolProgressData> = (
  progress: ToolProgress<P>,
) => void

// src/Tool.ts:307-310
export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}
```

### 4.8.2 BashTool 的进度报告

```typescript
// src/tools/BashTool/BashTool.tsx
onProgress?.({
  toolUseID: context.toolUseId!,
  data: {
    type: 'bash_progress',
    partialOutput: currentOutput,
    // ...
  },
})
```

### 4.8.3 进度类型

```typescript
// src/types/tools.ts
type BashProgress = {
  type: 'bash_progress'
  partialOutput: string
  exitCode?: number
}

type WebSearchProgress = {
  type: 'websearch_progress'
  resultsSoFar: WebSearchResult[]
}

type MCPProgress = {
  type: 'mcp_progress'
  message: string
}
```

## 4.9 工具构建器 — buildTool

### 4.9.1 简化工具创建

Claude Code 提供了一个 `buildTool` 辅助函数：

```typescript
// src/Tool.ts - buildTool 函数
export function buildTool<T extends Tool>(def: T): T {
  // 提供默认值和标准化处理
  return {
    interruptBehavior: () => 'block',
    isConcurrencySafe: () => false,
    isReadOnly: () => false,
    isEnabled: () => true,
    ...def,
  }
}
```

### 4.9.2 使用示例

```typescript
// src/tools/GlobTool/GlobTool.tsx
export const GlobTool = buildTool({
  name: 'Glob',

  inputSchema: z.object({
    pattern: z.string()
      .describe('Glob pattern to match files (e.g., "**/*.ts")'),
    baseDirectory: z.string().optional(),
  }),

  isConcurrencySafe: () => true,
  isReadOnly: () => true,

  async call(input, context, canUseTool, parentMessage, onProgress) {
    const files = await glob(input.pattern, {
      cwd: input.baseDirectory || context.options.workingDirectory,
    })
    return { data: { files } }
  },
})
```

## 4.10 工具查找 — findToolByName

### 4.10.1 按名称查找

```typescript
// src/Tool.ts:358-360
export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}

// src/Tool.ts:348-353
export function toolMatchesName(
  tool: { name: string; aliases?: string[] },
  name: string,
): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}
```

### 4.10.2 别名支持

```typescript
// 可以给工具添加别名
const OldNameTool = buildTool({
  name: 'NewName',
  aliases: ['OldName', 'LegacyName'],  // 向后兼容
  // ...
})
```

## 4.11 本章小结

Tool 接口是 Claude Code 工具系统的核心契约：

| 接口字段 | 作用 |
|---------|------|
| `name` | 唯一标识 |
| `call` | 执行逻辑 |
| `description` | LLM 理解接口 |
| `inputSchema` | 输入验证 |
| `isConcurrencySafe` | 并行策略 |
| `isReadOnly` | 读写分类 |
| `ToolUseContext` | 执行环境 |

**设计亮点**：

1. **Zod Schema** — 运行时验证 + TypeScript 推导
2. **ToolUseContext** — 统一的执行环境访问
3. **Progress 报告** — 支持长时间操作的进度更新
4. **元数据丰富** — description, searchHint 帮助 LLM 决策

## 4.12 实践要点

1. **Schema 描述要清晰** — LLM 依赖这些描述理解工具用途
2. **正确实现 isConcurrencySafe** — 影响并行执行效率
3. **善用 progress 回调** — 长操作应该有进度反馈
4. **处理中止信号** — 检查 `context.abortController.signal`

---

## 下一步

下一章我们将学习 **工具注册与发现**，了解：
- 30+ 工具如何组织
- 特性开关与条件加载
- 工具的分类体系
