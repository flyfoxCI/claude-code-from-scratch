# 第 10 章：响应式压缩 — Reactive Systems

> "错误不应该是终点，而应该是恢复的起点。"

## 10.1 响应式设计概述

Claude Code 不仅仅是被动地压缩，它还实现了**响应式压缩**——当遇到错误时，能够智能地恢复：

```
传统方式：
API Error (413) ──► 表面错误 ──► 用户看到 "Context too long"
                      └──► 结束

响应式方式：
API Error (413) ──► withhold 错误 ──► 尝试压缩 ──►
    │
    ├── 成功 ──► 继续执行
    │
    └── 失败 ──► 表面错误
```

## 10.2 错误 withhold 机制

### 10.2.1 什么是 withhold？

Withhold = 暂时不向用户展示错误，给系统恢复机会。

```typescript
// src/query.ts:788-822
let withheld = false

// 检查是否应该 withhold 此错误
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(
    message,
    isPromptTooLongMessage,
    querySource,
  )) {
    withheld = true
  }
}

if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}

// Media recovery
if (
  mediaRecoveryEnabled &&
  reactiveCompact?.isWithheldMediaSizeError(message)
) {
  withheld = true
}

// max_output_tokens 错误
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}

// 如果不 withhold，yield 给用户
if (!withheld) {
  yield yieldMessage
}

// 无论是否 withhold，都添加到消息列表
if (message.type === 'assistant') {
  assistantMessages.push(message)
}
```

### 10.2.2 Withhold 判断逻辑

```typescript
// src/services/compact/reactiveCompact.ts
export function isWithheldPromptTooLong(
  message: StreamEvent,
): boolean {
  // 1. 必须是 assistant 消息
  if (message.type !== 'assistant') {
    return false
  }

  // 2. 必须是 API 错误消息
  if (!message.isApiErrorMessage) {
    return false
  }

  // 3. 错误类型必须是 prompt_too_long
  if (!isPromptTooLongMessage(message)) {
    return false
  }

  // 4. 检查特性开关
  if (!feature('REACTIVE_COMPACT')) {
    return false
  }

  return true
}
```

## 10.3 恢复尝试

### 10.3.1 Reactive-Compact 恢复流程

```typescript
// src/query.ts:1119-1166
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  // 尝试响应式压缩
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
    // 压缩成功！
    const postCompactMessages = buildPostCompactMessages(compacted)

    state = {
      messages: postCompactMessages,
      hasAttemptedReactiveCompact: true,
      // ...
    }
    continue  // 继续循环
  }

  // 恢复失败，表面错误
  yield lastMessage
}
```

### 10.3.2 恢复策略

```typescript
// src/services/compact/reactiveCompact.ts
export async function tryReactiveCompact(
  params: ReactiveCompactParams,
): Promise<CompactedMessages | null> {
  const { hasAttempted, messages } = params

  // 第一阶段：上下文折叠排水
  if (!hasAttempted) {
    const collapsed = tryContextCollapse(messages)
    if (collapsed && isUnderLimit(collapsed)) {
      return collapsed
    }
  }

  // 第二阶段：全量压缩
  if (!hasAttempted) {
    const compacted = await fullCompact(messages)
    if (compacted && isUnderLimit(compacted)) {
      return compacted
    }
  }

  // 第三阶段：强制压缩（保留最少）
  const forced = await forcedCompact(messages)
  if (forced) {
    return forced
  }

  return null  // 所有恢复都失败
}
```

## 10.4 媒体恢复

### 10.4.1 媒体大小错误

```typescript
// 图片、PDF 等大文件可能导致错误
if (
  mediaRecoveryEnabled &&
  reactiveCompact?.isWithheldMediaSizeError(message)
) {
  withheld = true
}
```

### 10.4.2 媒体恢复策略

```typescript
// src/services/compact/reactiveCompact.ts
export async function tryMediaRecovery(
  messages: Message[],
): Promise<CompactedMessages | null> {
  // 1. 找到包含媒体附件的消息
  const mediaMessages = messages.filter(m =>
    hasLargeMediaAttachment(m)
  )

  // 2. 移除或压缩媒体
  const recovered: Message[] = []
  for (const msg of messages) {
    if (isLargeMediaMessage(msg)) {
      // 替换为文本描述
      recovered.push({
        ...msg,
        content: `[Media content removed: ${getMediaDescription(msg)}]`,
      })
    } else {
      recovered.push(msg)
    }
  }

  return recovered
}
```

## 10.5 Context-Collapse 集成

### 10.5.1 排水（Drain）机制

```typescript
// src/services/contextCollapse/index.ts
export function recoverFromOverflow(
  messages: Message[],
  querySource: string,
): { messages: Message[]; committed: number } {
  if (!isContextCollapseEnabled()) {
    return { messages, committed: 0 }
  }

  // 获取已暂存但未提交的折叠
  const stagedCollapses = getStagedCollapses()

  if (stagedCollapses.length === 0) {
    return { messages, committed: 0 }
  }

  // 提交暂存的折叠
  const collapsedMessages: Message[] = []

  for (const msg of messages) {
    if (isStagedCollapse(msg)) {
      // 将暂存的折叠转换为实际折叠
      collapsedMessages.push(commitStagedCollapse(msg))
    } else {
      collapsedMessages.push(msg)
    }
  }

  return {
    messages: collapsedMessages,
    committed: stagedCollapses.length,
  }
}
```

### 10.5.2 与 Reactive-Compact 配合

```typescript
// src/query.ts:1086-1117
if (isWithheld413) {
  // 首先尝试上下文折叠排水
  if (
    feature('CONTEXT_COLLAPSE') &&
    contextCollapse &&
    state.transition?.reason !== 'collapse_drain_retry'
  ) {
    const drained = contextCollapse.recoverFromOverflow(
      messagesForQuery,
      querySource,
    )

    if (drained.committed > 0) {
      const next: State = {
        messages: drained.messages,
        transition: {
          reason: 'collapse_drain_retry',
          committed: drained.committed,
        },
        // ...
      }
      state = next
      continue  // 重试
    }
  }
}
```

## 10.6 max_output_tokens 恢复

### 10.6.1 输出截断恢复

```typescript
// src/query.ts:1188-1252
if (isWithheldMaxOutputTokens(lastMessage)) {
  // 第一阶段：升级到更大的输出限制
  if (
    capEnabled &&
    maxOutputTokensOverride === undefined &&
    !process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS
  ) {
    logEvent('tengu_max_tokens_escalate', {
      escalatedTo: ESCALATED_MAX_TOKENS,
    })

    state = {
      maxOutputTokensOverride: ESCALATED_MAX_TOKENS,
      transition: { reason: 'max_output_tokens_escalate' },
      // ...
    }
    continue
  }

  // 第二阶段：发送恢复消息
  if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    const recoveryMessage = createUserMessage({
      content: `Output token limit hit. Resume directly — no apology, no recap of what you were doing.
Pick up mid-thought if that is where the cut happened.
Break remaining work into smaller pieces.`,
      isMeta: true,
    })

    state = {
      messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
      maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
      transition: { reason: 'max_output_tokens_recovery', attempt: maxOutputTokensRecoveryCount + 1 },
    }
    continue
  }

  // 恢复失败，表面错误
  yield lastMessage
}
```

### 10.6.2 恢复限制

```typescript
// 防止无限重试
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3
```

## 10.7 错误表面时机

### 10.7.1 何时表面错误

```typescript
// 所有恢复都失败后，才表面错误
if (
  isWithheld413 &&
  !reactiveCompact?.canRecover() &&
  !contextCollapse?.canRecover()
) {
  yield lastMessage  // 表面错误
  return { reason: 'prompt_too_long' }
}
```

### 10.7.2 跳过 Stop Hooks

```typescript
// API 错误时，跳过 stop hooks
// 因为 hook 可能会再次触发同样的错误
if (lastMessage?.isApiErrorMessage) {
  void executeStopFailureHooks(lastMessage, toolUseContext)
  return { reason: 'completed' }
}
```

## 10.8 设计模式

### 10.8.1 错误恢复状态机

```typescript
// 错误恢复状态机
const ErrorRecoveryMachine = {
  state: 'idle',

  transitions: {
    error: {
      withhold: ['withheld'],
      surface: ['failed'],
    },
    withheld: {
      recover: ['recovering'],
      surface: ['failed'],
    },
    recovering: {
      success: ['idle'],
      retry: ['recovering'],
      fail: ['failed'],
    },
  },

  dispatch(event) {
    const nextState = this.transitions[this.state][event]
    if (nextState) {
      this.state = nextState
    }
  },
}
```

### 10.8.2 预算回退

```typescript
// 跨压缩边界传递剩余预算
function carryOverBudget(
  previousBudget: number,
  consumedBeforeCompact: number,
): number {
  return Math.max(0, previousBudget - consumedBeforeCompact)
}
```

## 10.9 本章小结

响应式压缩是 Claude Code 的智能错误恢复机制：

| 机制 | 说明 |
|------|------|
| `withhold` | 暂时隐藏错误，等待恢复 |
| `recoverFromOverflow` | Context-Collapse 排水 |
| `tryReactiveCompact` | 多阶段压缩恢复 |
| `tryMediaRecovery` | 媒体文件恢复 |
| `maxOutputTokensRecovery` | 输出截断恢复 |

**核心设计决策**：

1. **错误 withhold** — 不急于表面错误
2. **多阶段恢复** — 从轻量到重量级
3. **有限重试** — 防止无限循环
4. **跨边界追踪** — 压缩后继续追踪预算

## 10.10 实践要点

1. **理解 withhold** — 错误不一定是终点
2. **多阶段策略** — 从简单到复杂
3. **设置重试限制** — 防止死循环
4. **正确处理媒体** — 大文件需要特殊处理

---

## 下一步

下一章我们将学习 **MCP 协议集成**，了解如何扩展 Agent 的工具能力。
