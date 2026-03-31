# 第 6 章：工具执行引擎 — Tool Orchestration

> "工具执行不是简单的顺序调用，而是智能的并行与调度。"

## 6.1 工具执行概述

当 LLM 返回 `tool_use` 块时，Claude Code 需要：

1. **解析工具调用** — 提取工具名称和参数
2. **验证参数** — 使用 Zod Schema 验证
3. **检查权限** — 用户是否允许此操作
4. **执行工具** — 串行或并行
5. **收集结果** — 返回给 LLM 继续

### 6.1.1 执行策略

```
LLM 返回: tool_use [
  { name: "Read", input: { path: "a.txt" } },
  { name: "Read", input: { path: "b.txt" } },
  { name: "Bash", input: { command: "ls" } }
]

执行策略：
- Read(a.txt) ──┐
- Read(b.txt) ──┼── 并行 (都是 isConcurrencySafe)
                │
                └──► Bash(ls) ──► 串行 (非并发安全)
```

## 6.2 工具编排 — runTools

### 6.2.1 主入口函数

```typescript
// src/services/tools/toolOrchestration.ts
export async function* runTools(
  toolUseBlocks: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<ToolUpdate> {
  // 1. 分离并发安全和并发不安全的工具
  const { concurrencySafe, nonConcurrent } = partitionTools(toolUseBlocks)

  // 2. 并行执行安全的工具
  const safeResults = await executeConcurrently(
    concurrencySafe,
    toolUseContext,
    canUseTool,
  )

  // 3. 串行执行不安全的工具
  for (const tool of nonConcurrent) {
    const result = await executeSerially(tool, toolUseContext, canUseTool)
    yield* result
  }

  // 4. 合并结果
  yield* safeResults
}
```

### 6.2.2 工具分区

```typescript
// src/services/tools/toolOrchestration.ts
function partitionTools(
  toolBlocks: ToolUseBlock[],
): { concurrencySafe: ToolUseBlock[]; nonConcurrent: ToolUseBlock[] } {
  return toolBlocks.reduce(
    (acc, block) => {
      const tool = findToolByName(toolUseContext.options.tools, block.name)
      if (tool?.isConcurrencySafe?.(block.input)) {
        acc.concurrencySafe.push(block)
      } else {
        acc.nonConcurrent.push(block)
      }
      return acc
    },
    { concurrencySafe: [], nonConcurrent: [] }
  )
}
```

## 6.3 流式工具执行器 — StreamingToolExecutor

### 6.3.1 为什么需要流式执行？

传统方式：等所有工具执行完再返回
```typescript
// 传统方式
const results = []
for (const tool of tools) {
  const result = await tool.call()  // 等待完成
  results.push(result)
}
return results
```

流式方式：工具执行时就开始返回
```typescript
// 流式方式
for (const tool of tools) {
  const executor = startTool(tool)  // 立即开始
  executor.onProgress((progress) => {
    yield { type: 'progress', ...progress }  // 实时报告
  })
  executor.onComplete((result) => {
    yield { type: 'result', ...result }
  })
}
```

### 6.3.2 StreamingToolExecutor 实现

```typescript
// src/services/tools/StreamingToolExecutor.ts
export class StreamingToolExecutor {
  private pendingTools: Map<string, ToolUseBlock> = new Map()
  private completedTools: Map<string, ToolResult> = new Map()
  private progressCallbacks: Map<string, ToolCallProgress[]> = new Map()

  addTool(toolBlock: ToolUseBlock, assistantMessage: AssistantMessage): void {
    const tool = findToolByName(this.tools, toolBlock.name)

    if (tool?.isConcurrencySafe?.(toolBlock.input)) {
      // 并发执行
      this.executeConcurrently(toolBlock, tool)
    } else {
      // 串行执行
      this.executeSerially(toolBlock, tool)
    }
  }

  private async executeConcurrently(
    toolBlock: ToolUseBlock,
    tool: Tool,
  ): Promise<void> {
    const promise = tool.call(
      toolBlock.input,
      this.context,
      this.canUseTool,
      this.parentMessage,
      (progress) => this.onToolProgress(toolBlock.id, progress),
    )

    // 不 await，直接返回 promise
    promise.then((result) => {
      this.completedTools.set(toolBlock.id, result)
      this.pendingTools.delete(toolBlock.id)
    })
  }

  getCompletedResults(): ToolResult[] {
    return Array.from(this.completedTools.values())
  }
}
```

### 6.3.3 进度回调

```typescript
// src/services/tools/StreamingToolExecutor.ts
private onToolProgress(
  toolUseId: string,
  progress: ToolProgress,
): void {
  // 产生进度消息
  this.emit({
    type: 'progress',
    toolUseId,
    progress,
  })
}
```

## 6.4 权限检查

### 6.4.1 canUseTool 回调

```typescript
// src/hooks/useCanUseTool.ts
export type CanUseToolFn = (params: {
  toolName: string
  toolInput: unknown
}) => Promise<{ use: boolean; reason?: string }>
```

### 6.4.2 权限检查流程

```typescript
// src/services/tools/toolExecution.ts
async function executeTool(
  tool: Tool,
  toolBlock: ToolUseBlock,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
): Promise<ToolResult> {
  // 1. 检查是否有缓存决策
  const cached = context.toolDecisions?.get(toolBlock.id)
  if (cached) {
    if (cached.decision === 'reject') {
      return { data: { error: 'Permission denied' } }
    }
  }

  // 2. 检查工具权限规则
  const permissionContext = context.options.toolPermissionContext
  if (permissionContext.alwaysAllowRules[tool.name]) {
    return tool.call(toolBlock.input, context)
  }

  if (permissionContext.alwaysDenyRules[tool.name]) {
    return { data: { error: 'Always denied' } }
  }

  // 3. 请求权限
  const result = await canUseTool({
    toolName: tool.name,
    toolInput: toolBlock.input,
  })

  if (!result.use) {
    // 记录拒绝
    context.toolDecisions?.set(toolBlock.id, {
      source: 'user',
      decision: 'reject',
      timestamp: Date.now(),
    })
    return { data: { error: result.reason || 'Permission denied' } }
  }

  // 4. 执行工具
  return tool.call(toolBlock.input, context)
}
```

## 6.5 工具执行上下文更新

### 6.5.1 执行后更新上下文

```typescript
// src/services/tools/toolOrchestration.ts
export function updateToolUseContext(
  context: ToolUseContext,
  toolResults: ToolResult[],
): ToolUseContext {
  // 1. 记录文件读取状态
  const readFileState = context.readFileState.clone()
  for (const result of toolResults) {
    if (result.type === 'file_read') {
      readFileState.recordRead(result.path)
    }
  }

  // 2. 更新其他状态
  return {
    ...context,
    readFileState,
    // 可能还有其他需要更新的状态
  }
}
```

### 6.5.2 结果影响上下文

```typescript
// 某些工具结果可能改变后续工具的行为
interface ToolResult<T> {
  data: T
  newMessages?: Message[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
}

// 使用
for (const update of toolUpdates) {
  if (update.newContext) {
    // 工具执行修改了上下文
    updatedToolUseContext = update.newContext
  }
}
```

## 6.6 错误处理

### 6.6.1 工具执行错误

```typescript
// src/services/tools/toolExecution.ts
async function executeToolWithErrorHandling(
  toolBlock: ToolUseBlock,
): Promise<ToolResult> {
  try {
    const result = await tool.call(...)
    return result
  } catch (error) {
    // 错误分类
    if (error instanceof ToolNotFoundError) {
      return {
        data: { error: `Tool '${toolBlock.name}' not found` },
      }
    }

    if (error instanceof ValidationError) {
      return {
        data: { error: `Invalid input: ${error.message}` },
      }
    }

    if (error instanceof PermissionError) {
      return {
        data: { error: `Permission denied: ${error.message}` },
      }
    }

    // 未知错误
    return {
      data: { error: `Tool execution failed: ${error.message}` },
    }
  }
}
```

### 6.6.2 超时处理

```typescript
// src/services/tools/toolExecution.ts
async function executeWithTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number,
): Promise<T> {
  return Promise.race([
    promise,
    new Promise<T>((_, reject) =>
      setTimeout(() => reject(new Error('Tool execution timeout')), timeoutMs)
    ),
  ])
}
```

## 6.7 中断处理

### 6.7.1 AbortController 集成

```typescript
// src/services/tools/toolExecution.ts
async function executeTool(
  toolBlock: ToolUseBlock,
  context: ToolUseContext,
): Promise<ToolResult> {
  // 检查中断信号
  if (context.abortController.signal.aborted) {
    return {
      data: {
        error: 'Interrupted',
        isInterrupted: true,
      },
    }
  }

  // 注册中断监听
  const handler = () => {
    // 清理资源
    cleanup()
  }
  context.abortController.signal.addEventListener('abort', handler)

  try {
    return await tool.call(toolBlock.input, context)
  } finally {
    context.abortController.signal.removeEventListener('abort', handler)
  }
}
```

### 6.7.2 部分结果

```typescript
// 如果中断时某些工具已完成，返回部分结果
const partialResults = completedTools.map(tool => ({
  toolUseId: tool.id,
  result: tool.result,
  status: 'completed' as const,
}))

const interruptedTools = pendingTools.map(tool => ({
  toolUseId: tool.id,
  status: 'interrupted' as const,
}))

return {
  results: [...partialResults, ...interruptedTools],
  hasPartialResults: true,
}
```

## 6.8 工具结果缓存

### 6.8.1 结果去重

```typescript
// src/services/tools/toolResultStorage.ts
export function useCachedToolResult(
  toolBlock: ToolUseBlock,
  cache: ToolResultCache,
): ToolResult | null {
  const cacheKey = generateCacheKey(toolBlock.name, toolBlock.input)

  const cached = cache.get(cacheKey)
  if (cached) {
    // 验证缓存是否仍然有效
    if (isCacheValid(cached)) {
      return cached.result
    }
  }

  return null
}
```

### 6.8.2 缓存键生成

```typescript
// src/services/tools/toolResultStorage.ts
function generateCacheKey(toolName: string, input: object): string {
  return `${toolName}:${JSON.stringify(input, Object.keys(input).sort())}`
}
```

## 6.9 设计模式

### 6.9.1 执行器模式

```typescript
// 统一的工具执行接口
interface ToolExecutor {
  addTool(toolBlock: ToolUseBlock): void
  execute(): AsyncGenerator<ToolResult>
  getCompletedResults(): ToolResult[]
  discard(): void
}

// 实现串行执行器
class SerialExecutor implements ToolExecutor {
  async *execute() {
    for (const tool of this.tools) {
      yield await this.executeOne(tool)
    }
  }
}

// 实现并行执行器
class ConcurrentExecutor implements ToolExecutor {
  async *execute() {
    const promises = this.tools.map(t => this.executeOne(t))
    for (const promise of promises) {
      yield await promise
    }
  }
}
```

### 6.9.2 管道模式

```typescript
// 工具执行管道
const executionPipeline = pipe(
  validateInput,      // 1. 验证输入
  checkPermission,    // 2. 检查权限
  recordMetrics,      // 3. 记录指标
  execute,            // 4. 执行
  handleResult,       // 5. 处理结果
  updateCache,        // 6. 更新缓存
)

for await (const result of executionPipeline(toolBlock)) {
  yield result
}
```

## 6.10 本章小结

工具执行引擎负责工具的智能调度：

| 组件 | 职责 |
|------|------|
| `runTools` | 主入口，分离并发/串行 |
| `StreamingToolExecutor` | 流式执行和进度报告 |
| `partitionTools` | 按并发安全性分区 |
| `executeWithTimeout` | 超时控制 |
| `ToolResultCache` | 结果缓存 |

**核心设计决策**：

1. **并发安全分离** — 读操作并行，写操作串行
2. **流式执行** — 减少 LLM 等待时间
3. **进度回调** — 长时间任务可以报告进度
4. **中断支持** — AbortController 集成

## 6.11 实践要点

1. **正确实现 isConcurrencySafe** — 影响并行效率
2. **处理中断信号** — 用户可能随时取消
3. **超时控制** — 防止工具无限挂起
4. **结果缓存** — 相同调用直接返回缓存

---

## 下一步

下一章我们将通过 **BashTool 实战**，看一个完整工具的实现细节。
