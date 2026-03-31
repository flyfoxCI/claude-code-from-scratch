# 第 17 章：错误处理与恢复 — Error Handling

> "健壮的 Agent 不是不犯错，而是知道如何从错误中恢复。"

## 17.1 错误分类

Claude Code 遇到多种类型的错误：

```
错误类型
├── API 错误
│   ├── Rate Limit (429)
│   ├── Auth Error (401)
│   ├── Prompt Too Long (413)
│   └── max_output_tokens
│
├── 工具错误
│   ├── Permission Denied
│   ├── File Not Found
│   ├── Timeout
│   └── Execution Failed
│
├── 用户中断
│   └── Abort Signal
│
└── 内部错误
    └── Bug / Unexpected State
```

## 17.2 API 错误处理

### 17.2.1 错误类型

```typescript
// src/services/api/errors.ts
export class APIError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number,
    public isRetryable: boolean,
  ) {
    super(message)
    this.name = 'APIError'
  }
}

// 常见错误
export const ERROR_CODES = {
  RATE_LIMIT: 'rate_limit',
  AUTH_ERROR: 'authentication_error',
  PROMPT_TOO_LONG: 'prompt_too_long',
  MAX_OUTPUT_TOKENS: 'max_output_tokens',
  INVALID_REQUEST: 'invalid_request',
  SERVER_ERROR: 'server_error',
}
```

### 17.2.2 重试机制

```typescript
// src/services/api/withRetry.ts
async function callWithRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {},
): Promise<T> {
  const {
    maxRetries = 3,
    initialDelay = 1000,
    backoffMultiplier = 2,
    shouldRetry = isRetryableError,
  } = options

  let lastError: Error
  let delay = initialDelay

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error

      if (attempt === maxRetries || !shouldRetry(error)) {
        throw error
      }

      // 指数退避
      await sleep(delay)
      delay *= backoffMultiplier
    }
  }

  throw lastError!
}
```

### 17.2.3 可重试错误判断

```typescript
// src/services/api/errors.ts
export function isRetryableError(error: APIError): boolean {
  // 速率限制
  if (error.code === ERROR_CODES.RATE_LIMIT) {
    return true
  }

  // 服务器错误
  if (error.statusCode >= 500 && error.statusCode < 600) {
    return true
  }

  // 网络错误
  if (error.code === 'NETWORK_ERROR') {
    return true
  }

  return false
}
```

## 17.3 模型回退

### 17.3.1 FallbackTriggeredError

```typescript
// src/services/api/withRetry.ts
export class FallbackTriggeredError extends Error {
  constructor(
    public originalModel: string,
    public fallbackModel: string,
  ) {
    super(`Fallback triggered: ${originalModel} → ${fallbackModel}`)
    this.name = 'FallbackTriggeredError'
  }
}
```

### 17.3.2 回退处理

```typescript
// src/query.ts:894-951
try {
  // 流式调用
  for await (const message of deps.callModel({...})) {
    // 处理消息
  }
} catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    // 切换到备用模型
    currentModel = fallbackModel
    attemptWithFallback = true

    // 清空之前的消息
    yield* yieldMissingToolResultBlocks(
      assistantMessages,
      'Model fallback triggered',
    )
    assistantMessages.length = 0
    toolResults.length = 0
    toolUseBlocks.length = 0
    needsFollowUp = false

    // 丢弃未完成的流式工具执行
    if (streamingToolExecutor) {
      streamingToolExecutor.discard()
      streamingToolExecutor = new StreamingToolExecutor(...)
    }

    // 更新上下文
    toolUseContext.options.mainLoopModel = fallbackModel

    // 发送提示
    yield createSystemMessage(
      `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand`,
      'warning',
    )

    // 重试
    continue
  }
  throw innerError
}
```

## 17.4 用户中断

### 17.4.1 中断信号

```typescript
// src/query.ts:1011-1052
if (toolUseContext.abortController.signal.aborted) {
  // 处理中断
  if (streamingToolExecutor) {
    // 消耗剩余结果
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      if (update.message) {
        yield update.message
      }
    }
  } else {
    // 为未完成的工具生成错误消息
    yield* yieldMissingToolResultBlocks(
      assistantMessages,
      'Interrupted by user',
    )
  }

  // 清理
  if (feature('CHICAGO_MCP') && !toolUseContext.agentId) {
    try {
      await cleanupComputerUseAfterTurn(toolUseContext)
    } catch {
      // 静默失败
    }
  }

  // 发送中断消息（除非是 submit-interrupt）
  if (toolUseContext.abortController.signal.reason !== 'interrupt') {
    yield createUserInterruptionMessage({
      toolUse: true,
    })
  }

  return { reason: 'aborted_streaming' }
}
```

### 17.4.2 中断原因

```typescript
// AbortController 可以有不同的 reason
const abortController = new AbortController()

// 用户按 Ctrl+C
abortController.abort('interrupt')

// 用户提交新消息
abortController.abort('submit')

// 任务完成
abortController.abort('complete')
```

## 17.5 工具错误处理

### 17.5.1 错误分类

```typescript
// src/utils/errors.ts
export class ToolExecutionError extends Error {
  constructor(
    message: string,
    public toolName: string,
    public toolInput: unknown,
    public code: string,
  ) {
    super(message)
    this.name = 'ToolExecutionError'
  }
}

// 常见工具错误码
export const TOOL_ERROR_CODES = {
  PERMISSION_DENIED: 'permission_denied',
  FILE_NOT_FOUND: 'file_not_found',
  TIMEOUT: 'timeout',
  VALIDATION_ERROR: 'validation_error',
  EXECUTION_FAILED: 'execution_failed',
}
```

### 17.5.2 错误处理流程

```typescript
// src/services/tools/toolExecution.ts
async function executeToolWithErrorHandling(
  toolBlock: ToolUseBlock,
  context: ToolUseContext,
): Promise<ToolResult> {
  try {
    const result = await tool.call(
      toolBlock.input,
      context,
    )
    return result
  } catch (error) {
    if (error instanceof ToolExecutionError) {
      return {
        data: {
          error: error.message,
          code: error.code,
        },
      }
    }

    if (error instanceof TimeoutError) {
      return {
        data: {
          error: 'Tool execution timed out',
          code: 'timeout',
        },
      }
    }

    // 未知错误
    return {
      data: {
        error: `Unexpected error: ${error.message}`,
        code: 'unknown',
      },
    }
  }
}
```

## 17.6 停止钩子

### 17.6.1 钩子类型

```typescript
// src/query/stopHooks.ts
interface StopHook {
  id: string
  name: string
  handler: (context: StopHookContext) => Promise<StopHookResult>
  blocking: boolean  // 是否阻塞继续
}

interface StopHookContext {
  messages: Message[]
  toolUseContext: ToolUseContext
  stopReason: string
}

interface StopHookResult {
  preventContinuation: boolean
  blockingErrors: Message[]
  nudgeMessage?: string
}
```

### 17.6.2 执行停止钩子

```typescript
// src/query/stopHooks.ts
async function handleStopHooks(
  messages: Message[],
  assistantMessages: AssistantMessage[],
  // ...
): Promise<StopHookResult> {
  const hooks = await getStopHooks()

  for (const hook of hooks) {
    const result = await hook.handler({
      messages: [...messages, ...assistantMessages],
      toolUseContext,
      stopReason: 'completed',
    })

    if (result.preventContinuation) {
      return result
    }

    if (result.blockingErrors.length > 0) {
      return result
    }
  }

  return { preventContinuation: false, blockingErrors: [] }
}
```

## 17.7 失败记录

### 17.7.1 错误遥测

```typescript
// src/services/analytics/index.ts
export function logError(
  error: Error,
  context: {
    queryChainId?: string
    queryDepth?: number
    toolName?: string
  } = {},
): void {
  logEvent('tengu_error', {
    errorName: error.name,
    errorMessage: error.message,
    stack: error.stack,
    ...context,
  })
}
```

### 17.7.2 失败计数

```typescript
// src/services/compact/autoCompact.ts
interface CompactTrackingState {
  consecutiveFailures: number
  lastFailureAt?: number
}

// 连续失败计数
if (consecutiveFailures >= MAX_CONSECUTIVE_FAILURES) {
  // 放弃自动压缩
  return { error: 'Too many consecutive failures' }
}
```

## 17.8 设计模式

### 17.8.1 错误恢复状态机

```typescript
// 通用错误恢复状态机
class ErrorRecoveryStateMachine {
  state: 'idle' | 'retrying' | 'fallback' | 'failed' = 'idle'

  transitions = {
    idle: {
      onError: 'retrying',
    },
    retrying: {
      onSuccess: 'idle',
      onMaxRetries: 'fallback',
    },
    fallback: {
      onSuccess: 'idle',
      onFailure: 'failed',
    },
    failed: {
      onReset: 'idle',
    },
  }

  dispatch(event: string): void {
    this.state = this.transitions[this.state][event] || this.state
  }
}
```

### 17.8.2 错误包装

```typescript
// 将低层错误转换为高层错误
function wrapError(error: unknown, context: string): Error {
  if (error instanceof APIError) {
    return error
  }

  if (error instanceof Error) {
    return new Error(`${context}: ${error.message}`, { cause: error })
  }

  return new Error(`${context}: ${String(error)}`)
}
```

## 17.9 本章小结

错误处理是 Agent 健壮性的关键：

| 机制 | 说明 |
|------|------|
| `callWithRetry` | 指数退避重试 |
| `FallbackTriggeredError` | 模型回退 |
| `AbortController` | 用户中断 |
| `handleStopHooks` | 停止钩子 |
| `logError` | 错误遥测 |

**核心设计决策**：

1. **错误分类** — 不同错误不同处理
2. **重试机制** — 指数退避
3. **回退策略** — 备用模型
4. **中断处理** — 优雅清理

## 17.10 实践要点

1. **区分错误类型** — 可重试 vs 不可重试
2. **设置重试限制** — 防止无限重试
3. **正确传播错误** — 不要吞掉错误
4. **记录上下文** — 便于调试

---

## 下一步

下一章我们将学习 **权限与安全**，了解如何安全地执行工具。
