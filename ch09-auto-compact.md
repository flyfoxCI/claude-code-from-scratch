# 第 9 章：自动压缩系统 — Context Management

> "上下文窗口是有限的，但对话可以很长。"

## 9.1 问题：上下文窗口限制

LLM 有上下文窗口限制（如 200K tokens）。当对话很长时：

```
问题：对话历史持续增长
├── 第 1 轮：用户问题 (1K) + LLM 响应 (2K)
├── 第 2 轮：+ 历史 (3K) + 新问题 (1K) + LLM 响应 (5K)
├── 第 3 轮：+ 历史 (8K) + 新问题 (2K) + ...
└── ...
         ↓
最终：历史 > 上下文窗口 → 无法继续
```

Claude Code 的解决方案：**自动压缩 (Auto-Compact)**

## 9.2 压缩策略概述

Claude Code 实现了几种压缩策略：

| 策略 | 说明 | 触发条件 |
|------|------|---------|
| Auto-Compact | 自动压缩历史 | Token 超阈值 |
| Reactive-Compact | 响应式压缩 | API 返回 413 错误 |
| Microcompact | 微压缩 | 每个 API 调用后 |
| Context-Collapse | 上下文折叠 | 特征开关启用 |

## 9.3 Auto-Compact

### 9.3.1 触发检查

```typescript
// src/query.ts:628-648
if (
  !compactionResult &&
  querySource !== 'compact' &&
  querySource !== 'session_memory' &&
  !(
    reactiveCompact?.isReactiveCompactEnabled() && isAutoCompactEnabled()
  ) &&
  !collapseOwnsIt
) {
  const { isAtBlockingLimit } = calculateTokenWarningState(
    tokenCountWithEstimation(messagesForQuery) - snipTokensFreed,
    toolUseContext.options.mainLoopModel,
  )

  if (isAtBlockingLimit) {
    yield createAssistantAPIErrorMessage({
      content: PROMPT_TOO_LONG_ERROR_MESSAGE,
      error: 'invalid_request',
    })
    return { reason: 'blocking_limit' }
  }
}
```

### 9.3.2 压缩执行

```typescript
// src/query.ts:454-543
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    forkContextMessages: messagesForQuery,
  },
  querySource,
  tracking,
  snipTokensFreed,
)

// 如果有压缩结果
if (compactionResult) {
  // 构建压缩后的消息
  const postCompactMessages = buildPostCompactMessages(compactionResult)

  for (const message of postCompactMessages) {
    yield message  // 通知压缩发生
  }

  // 使用压缩后的消息继续
  messagesForQuery = postCompactMessages

  // 重置跟踪状态
  tracking = {
    compacted: true,
    turnId: deps.uuid(),
    turnCounter: 0,
    consecutiveFailures: 0,
  }
}
```

## 9.4 压缩消息构建

### 9.4.1 buildPostCompactMessages

```typescript
// src/services/compact/compact.ts
export function buildPostCompactMessages(
  compactionResult: CompactionResult,
): Message[] {
  const messages: Message[] = []

  // 1. 添加摘要消息
  for (const summaryMsg of compactionResult.summaryMessages) {
    messages.push(summaryMsg)
  }

  // 2. 添加附件（保留重要上下文）
  for (const attachment of compactionResult.attachments) {
    messages.push(createAttachmentMessage(attachment))
  }

  // 3. 添加钩子结果
  for (const hookResult of compactionResult.hookResults) {
    messages.push(createSystemMessage(hookResult, 'compact'))
  }

  return messages
}
```

### 9.4.2 摘要消息

```typescript
// src/services/compact/summarize.ts
interface SummaryMessage {
  type: 'system'
  role: 'system'
  content: string  // 包含压缩前的关键信息摘要
}

function createSummaryMessage(
  originalMessages: Message[],
  tokenBudget: number,
): SummaryMessage {
  // 生成摘要，保留关键信息
  const summary = summarizeConversation(originalMessages, {
    maxTokens: tokenBudget,
    preserve: ['user_requests', 'key_decisions', 'file_changes'],
  })

  return {
    type: 'system',
    role: 'system',
    content: `Previous conversation summary:
${summary}`,
  }
}
```

## 9.5 Reactive-Compact

### 9.5.1 概念

Reactive-Compact 是一种**被动**压缩策略。当 API 返回 `413 Payload Too Large` 错误时触发。

### 9.5.2 错误 withhold

```typescript
// src/query.ts:788-822
let withheld = false

// 检查是否应该 withhold 错误
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, ...)) {
    withheld = true
  }
}

if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}

// 如果是 withhold 的错误，不 yield 给用户
if (!withheld) {
  yield yieldMessage
}

// 继续收集到 assistantMessages（用于后续恢复）
if (message.type === 'assistant') {
  assistantMessages.push(message)
}
```

### 9.5.3 恢复尝试

```typescript
// src/query.ts:1119-1166
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,
    querySource,
    aborted: toolUseContext.abortController.signal.aborted,
    messages: messagesForQuery,
    cacheSafeParams: {
      systemPrompt,
      userContext,
      systemContext,
      toolUseContext,
      forkContextMessages: messagesForQuery,
    },
  })

  if (compacted) {
    // 压缩成功，使用新消息继续
    const postCompactMessages = buildPostCompactMessages(compacted)
    state = {
      ...state,
      messages: postCompactMessages,
      hasAttemptedReactiveCompact: true,
    }
    continue
  }

  // 恢复失败，表面错误
  yield lastMessage
}
```

## 9.6 Token 预算

### 9.6.1 预算追踪

```typescript
// src/query/tokenBudget.ts
export function createBudgetTracker() {
  let totalInputTokens = 0
  let totalOutputTokens = 0
  let continuationCount = 0

  return {
    addInputTokens: (count: number) => {
      totalInputTokens += count
    },
    addOutputTokens: (count: number) => {
      totalOutputTokens += count
    },
    getTotal: () => ({
      input: totalInputTokens,
      output: totalOutputTokens,
      all: totalInputTokens + totalOutputTokens,
    }),
    getContinuationCount: () => continuationCount,
  }
}
```

### 9.6.2 预算检查

```typescript
// src/query.ts:1308-1355
if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!,
    toolUseContext.agentId,
    getCurrentTurnTokenBudget(),
    getTurnOutputTokens(),
  )

  if (decision.action === 'continue') {
    // 预算未耗尽，发送提示继续
    incrementBudgetContinuationCount()
    logForDebugging(`Token budget continuation #${decision.continuationCount}`)

    state = {
      ...state,
      messages: [
        ...state.messages,
        createUserMessage({ content: decision.nudgeMessage, isMeta: true }),
      ],
      transition: { reason: 'token_budget_continuation' },
    }
    continue
  }

  if (decision.completionEvent) {
    // 预算耗尽，记录完成事件
    logEvent('tengu_token_budget_completed', {
      ...decision.completionEvent,
      queryChainId: queryChainIdForAnalytics,
      queryDepth: queryTracking.depth,
    })
  }
}
```

## 9.7 上下文折叠 — Context Collapse

### 9.7.1 概念

Context-Collapse 是一种**主动**压缩策略，在达到限制之前就进行压缩：

```
正常流程：
[消息1] [消息2] [消息3] ... [消息N]
                                    ↓
                              触发压缩
                                    ↓
                              [摘要] [近N条]

Context-Collapse 流程：
[消息1] [消息2] [消息3] ... [消息N-2]
         ↓                    ↓
       折叠                  保留
         ↓                    ↓
    [旧摘要] ... [消息N-1] [消息N]
```

### 9.7.2 实现

```typescript
// src/services/contextCollapse/index.ts
export function applyCollapsesIfNeeded(
  messages: Message[],
  context: ToolUseContext,
  querySource: string,
): CollapseResult {
  if (!isContextCollapseEnabled()) {
    return { messages, committed: 0 }
  }

  // 检查是否需要压缩
  const tokenCount = tokenCountWithEstimation(messages)
  const threshold = getCollapseThreshold()

  if (tokenCount < threshold) {
    return { messages, committed: 0 }
  }

  // 执行折叠
  const collapsed = collapseMessages(messages, {
    preserveRecent: PRESERVE_RECENT_COUNT,  // 保留最近 N 条
    maxTokensToFree: threshold * 0.3,       // 释放 30% 空间
  })

  return {
    messages: collapsed,
    committed: collapsed.length,
  }
}
```

## 9.8 设计模式

### 9.8.1 压缩策略组合

```typescript
// 组合多种压缩策略
async function compactWithStrategy(
  messages: Message[],
  context: ToolUseContext,
): Promise<CompactResult | null> {
  // 1. 尝试主动压缩
  const proactive = await tryProactiveCompact(messages, context)
  if (proactive) return proactive

  // 2. 尝试微压缩
  const micro = await tryMicrocompact(messages, context)
  if (micro) return micro

  // 3. 尝试上下文折叠
  const collapsed = tryContextCollapse(messages, context)
  if (collapsed) return collapsed

  // 4. 都失败
  return null
}
```

### 9.8.2 延迟执行

```typescript
// 延迟压缩直到确认需要
const compactionResult = feature('CONTEXT_COLLAPSE')
  ? await contextCollapse.applyCollapsesIfNeeded(messages, ...)
  : null

// 只有在确认后才替换消息
if (compactionResult && compactionResult.committed > 0) {
  messages = compactionResult.messages
}
```

## 9.9 本章小结

上下文管理是 Claude Code 能处理长对话的关键：

| 策略 | 触发方式 | 适用场景 |
|------|---------|---------|
| Auto-Compact | Token 超阈值 | 常规长对话 |
| Reactive-Compact | API 413 错误 | 突发大请求 |
| Microcompact | 每个 API 调用 | 持续优化 |
| Context-Collapse | 主动预压缩 | 高频长对话 |

**核心思想**：
1. **不要等到爆满** — 提前压缩有更多选择
2. **保留关键信息** — 摘要 + 最近消息
3. **可恢复** — 压缩失败要有降级方案
4. **透明** — 用户应该感知不到压缩

## 9.10 实践要点

1. **选择合适的阈值** — 太频繁影响质量，太少可能爆
2. **保留关键上下文** — 用户意图、关键决策
3. **监控压缩效果** — 压缩后 LLM 还能正确继续吗？
4. **处理压缩失败** — 降级策略是什么？

---

## 下一步

下一章我们将学习 **MCP 协议集成**，了解如何扩展 Claude Code 的工具能力。
