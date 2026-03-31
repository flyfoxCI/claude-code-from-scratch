# 第 2 章：核心循环 — Query Loop

> "Query Loop 是 Agent 的心脏，每一下跳动都是一次 LLM 调用加上可能的工具执行。"

## 2.1 循环的本质

Agent 的核心是一个不断重复的循环：

```
用户输入 → LLM 思考 → 需要工具？
                    │
         ┌──────────┴──────────┐
         │ 否                 │ 是
         ▼                   ▼
      输出结果            执行工具
                              │
                              ▼
                         返回 LLM 继续思考
                              │
                              ▼
                    [重复直到无需工具]
```

Claude Code 的 `query.ts` 实现了这个循环，它是一个 **1730 行的 AsyncGenerator**。

## 2.2 AsyncGenerator 模式

### 2.2.1 什么是 AsyncGenerator？

AsyncGenerator 是 JavaScript 的异步迭代器，它允许：

1. **异步产生值** — `yield` 可以是 Promise
2. **流式处理** — 不需要等待全部完成
3. **保持状态** — 迭代器内部维护位置

```typescript
// 一个简单的 AsyncGenerator 示例
async function* countToThree() {
  for (let i = 1; i <= 3; i++) {
    await sleep(100)  // 异步等待
    yield i            // 产生值
  }
}

// 使用
for await (const num of countToThree()) {
  console.log(num)  // 1, 2, 3 (每 100ms 一个)
}
```

### 2.2.2 Claude Code 的 query 函数签名

```typescript
// src/query.ts:219-239
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent          // 流式事件
  | RequestStartEvent    // 请求开始
  | Message              // 消息
  | TombstoneMessage     // 删除标记
  | ToolUseSummaryMessage, // 工具摘要
  Terminal               // 终止状态
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)

  // 只有在 queryLoop 正常返回时才会执行这里
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

**类型参数解析**：

| 类型 | 说明 |
|------|------|
| `StreamEvent` | 模型流式输出的事件 |
| `Message` | 完整的消息（用户/助手/系统） |
| `TombstoneMessage` | 标记删除的消息 |
| `ToolUseSummaryMessage` | 工具使用的 AI 摘要 |
| `Terminal` | 循环终止原因 |

### 2.2.3 yield* 的作用

`yield*` 是委托迭代，它把控制权委托给另一个 generator：

```typescript
// src/query.ts:230
const terminal = yield* queryLoop(params, consumedCommandUuids)
```

这意味着 `query()` 本身不实现循环逻辑，而是把调用委托给 `queryLoop()`。

## 2.3 queryLoop 详解

### 2.3.1 状态管理

循环内部维护一个 `State` 对象：

```typescript
// src/query.ts:204-217
type State = {
  messages: Message[]                           // 对话历史
  toolUseContext: ToolUseContext                // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 压缩状态
  maxOutputTokensRecoveryCount: number          // 输出 token 恢复计数
  hasAttemptedReactiveCompact: boolean          // 是否尝试过响应式压缩
  maxOutputTokensOverride: number | undefined  // 最大输出 token 覆盖
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined           // 停止钩子是否激活
  turnCount: number                             // 当前轮次
  transition: Continue | undefined              // 继续原因
}
```

**关键设计**：状态在循环顶部解构，修改时创建新对象：

```typescript
// src/query.ts:307-321
while (true) {
  let { toolUseContext } = state
  const {
    messages,
    autoCompactTracking,
    maxOutputTokensRecoveryCount,
    // ...
  } = state
  // ...
}
```

### 2.3.2 循环体结构

```typescript
// src/query.ts:306-1728
// eslint-disable-next-line no-constant-condition
while (true) {
  // ===== 阶段 1：预处理 =====
  // 1.1 工具发现预取
  // 1.2 Token 预算检查
  // 1.3 上下文压缩检查

  // ===== 阶段 2：LLM 调用 =====
  yield { type: 'stream_request_start' }
  for await (const message of deps.callModel({...})) {
    // 处理流式消息
    // 收集 tool_use 块
  }

  // ===== 阶段 3：工具执行 =====
  for await (const update of runTools(toolUseBlocks, ...)) {
    yield update.message
  }

  // ===== 阶段 4：继续或终止 =====
  if (!needsFollowUp) {
    return { reason: 'completed' }
  }

  // ===== 阶段 5：准备下一轮 =====
  state = {
    ...state,
    messages: [...messages, ...assistantMessages, ...toolResults],
    turnCount: turnCount + 1,
  }
}
```

### 2.3.3 工具调用检测

LLM 返回后，如何知道需要调用工具？

```typescript
// src/query.ts:551-558
const assistantMessages: AssistantMessage[] = []
const toolUseBlocks: ToolUseBlock[] = []
let needsFollowUp = false

// 在流式处理中收集
for await (const message of deps.callModel({...})) {
  if (message.type === 'assistant') {
    assistantMessages.push(message)

    const msgToolUseBlocks = message.message.content.filter(
      content => content.type === 'tool_use',
    ) as ToolUseBlock[]

    if (msgToolUseBlocks.length > 0) {
      toolUseBlocks.push(...msgToolUseBlocks)
      needsFollowUp = true  // 标记需要工具执行
    }
  }
}
```

## 2.4 工具执行

### 2.4.1 runTools 函数

```typescript
// src/query.ts:1380-1408
const toolUpdates = runTools(
  toolUseBlocks,
  assistantMessages,
  canUseTool,
  toolUseContext,
)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message  // 产生工具结果消息
  }
  if (update.newContext) {
    updatedToolUseContext = update.newContext
  }
}
```

### 2.4.2 工具执行器 (StreamingToolExecutor)

现代 Claude Code 使用 `StreamingToolExecutor` 支持流式工具执行：

```typescript
// src/query.ts:561-568
const streamingToolExecutor = new StreamingToolExecutor(
  toolUseContext.options.tools,
  canUseTool,
  toolUseContext,
)

// 在流式处理中添加工具
for await (const message of deps.callModel({...})) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

**好处**：
- 工具可以在模型还在输出时就启动执行
- 减少等待时间
- 支持进度报告

## 2.5 错误处理与恢复

### 2.5.1 模型回退机制

当主要模型失败时，自动切换到备用模型：

```typescript
// src/query.ts:894-951
try {
  // ... 流式调用
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

    // 重试
    continue
  }
  throw innerError
}
```

### 2.5.2 输出 Token 恢复

当输出被截断时，自动重试并增加 token 限制：

```typescript
// src/query.ts:1188-1252
if (isWithheldMaxOutputTokens(lastMessage)) {
  // 第一次尝试：使用默认的 8k limit
  // 如果失败，进入 escalate 逻辑

  if (maxOutputTokensOverride === undefined) {
    // 升级到 64k
    logEvent('tengu_max_tokens_escalate', {
      escalatedTo: ESCALATED_MAX_TOKENS,
    })
    state = {
      ...state,
      maxOutputTokensOverride: ESCALATED_MAX_TOKENS,
      transition: { reason: 'max_output_tokens_escalate' },
    }
    continue
  }

  // 如果还是失败，发送恢复消息
  const recoveryMessage = createUserMessage({
    content: `Output token limit hit. Resume directly — no apology, no recap.
Pick up mid-thought if that is where the cut happened.
Break remaining work into smaller pieces.`,
    isMeta: true,
  })

  state = {
    ...state,
    messages: [...messages, ...assistantMessages, recoveryMessage],
    maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
  }
  continue
}
```

### 2.5.3 中断处理

用户可以随时中断 Agent 的执行：

```typescript
// src/query.ts:1011-1052
if (toolUseContext.abortController.signal.aborted) {
  if (streamingToolExecutor) {
    // 消耗剩余结果
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      if (update.message) {
        yield update.message
      }
    }
  } else {
    // 为每个 tool_use 生成错误消息
    yield* yieldMissingToolResultBlocks(
      assistantMessages,
      'Interrupted by user',
    )
  }

  return { reason: 'aborted_streaming' }
}
```

## 2.6 停止条件与钩子

### 2.6.1 正常停止

```typescript
// src/query.ts:1062-1357
if (!needsFollowUp) {
  // 没有工具调用 = 正常完成

  // 1. 处理停止钩子
  const stopHookResult = yield* handleStopHooks(...)

  if (stopHookResult.preventContinuation) {
    return { reason: 'stop_hook_prevented' }
  }

  if (stopHookResult.blockingErrors.length > 0) {
    // 有阻塞错误，重试
    state = {
      ...state,
      messages: [...messages, ...assistantMessages, ...stopHookResult.blockingErrors],
      stopHookActive: true,
    }
    continue
  }

  // 2. Token 预算检查
  if (feature('TOKEN_BUDGET')) {
    const decision = checkTokenBudget(budgetTracker!, ...)

    if (decision.action === 'continue') {
      // 预算未耗尽，发送提示继续
      state = {
        ...state,
        messages: [...messages, createUserMessage({ content: decision.nudgeMessage })],
      }
      continue
    }

    if (decision.completionEvent) {
      // 预算耗尽，记录事件
      logEvent('tengu_token_budget_completed', {...})
    }
  }

  // 3. 正常完成
  return { reason: 'completed' }
}
```

### 2.6.2 工具执行后的继续

```typescript
// src/query.ts:1714-1727
const next: State = {
  messages: [...messages, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: turnCount + 1,
  // ...
}
state = next
// 继续下一轮循环
```

## 2.7 实战：追踪一次完整的 Query

让我们追踪一次完整的执行流程：

```
1. 用户输入: "帮我创建一个 React 组件"
   │
   ▼
2. queryLoop 开始
   │
   ▼
3. 预处理
   - 检查 token 预算
   - 应用 microcompact
   │
   ▼
4. 调用 LLM
   yield { type: 'stream_request_start' }
   │
   ▼
5. 流式接收响应
   assistant message:
   "我会帮你创建这个组件。首先让我检查一下项目结构。"
   │
   ▼
6. 检测到 tool_use
   tool_use { name: "Bash", input: { command: "ls src/components" } }
   │
   ▼
7. 工具执行
   for await (const update of runTools(...)) {
     yield update.message  // 产生工具结果
   }
   │
   ▼
8. 工具结果加入消息
   messages = [...messages, assistant, tool_result]
   │
   ▼
9. 继续循环
   turnCount = 2
   │
   ▼
10. 再次调用 LLM
    "项目结构已检查。现在让我创建组件..."
    │
    ▼
11. 再次检测 tool_use
    tool_use { name: "FileWrite", input: { path: "src/components/Button.tsx", content: "..." } }
    │
    ▼
12. 执行文件写入
    │
    ▼
13. 继续循环
    │
    ▼
14. LLM 返回最终响应
    "我已经创建了 Button.tsx 组件，包含以下功能..."
    │
    ▼
15. 无需工具
    return { reason: 'completed' }
```

## 2.8 本章小结

Query Loop 是 Claude Code 的核心：

| 组件 | 职责 |
|------|------|
| `query()` | AsyncGenerator 入口 |
| `queryLoop()` | 实际循环实现 |
| `State` | 跨迭代状态管理 |
| `runTools()` | 工具执行 |
| `StreamingToolExecutor` | 流式工具执行 |

**关键设计决策**：

1. **AsyncGenerator** — 流式响应 + 状态保持
2. **显式状态** — 状态外部化，可序列化
3. **多阶段恢复** — 多种错误恢复策略
4. **流式工具执行** — 减少等待时间

## 2.9 实践要点

1. **理解 State 的流转**：每次 `continue` 都会创建新的 State
2. **追踪 needsFollowUp**：这是判断是否需要工具执行的关键
3. **善用 yield**：`yield` 不仅是返回值，更是暂停点
4. **错误恢复思维**：每种错误都有对应的恢复策略

---

## 下一步

下一章我们将学习 **消息系统**，了解 Agent 如何组织和管理对话历史。这包括：
- 消息类型体系
- 消息的创建与转换
- 上下文压缩后的消息重建
