# 第 8 章：Token 预算与成本控制

> "Token 是有限的资源，聪明的 Agent 应该知道何时停止。"

## 8.1 为什么需要 Token 预算？

LLM API 按 token 计费，上下文窗口有容量限制。Claude Code 需要：

1. **成本控制** — 防止意外的高额账单
2. **上下文管理** — 避免超过窗口限制
3. **用户提示** — 告知用户资源使用情况
4. **自动优化** — 在预算耗尽前主动压缩

```
Token 消耗场景
├── 用户输入 ──────────► 累加
├── LLM 输出 ──────────► 累加
├── 工具结果 ──────────► 累加
├── 系统提示词 ────────► 固定成本
└── 工具描述 ──────────► 按使用数量累加
```

## 8.2 Token 计数

### 8.2.1 估算函数

```typescript
// src/utils/tokens.ts
export function tokenCountWithEstimation(
  messages: Message[],
): number {
  let total = 0

  for (const message of messages) {
    // 估算消息 token 数
    total += estimateMessageTokens(message)
  }

  return total
}

function estimateMessageTokens(message: Message): number {
  switch (message.type) {
    case 'user':
      return estimateTextTokens(message.message.content)

    case 'assistant':
      let tokens = estimateTextTokens(message.message.content)
      // 加上 usage 中的精确计数（如果有）
      if (message.message.usage) {
        tokens = message.message.usage.input_tokens
      }
      return tokens

    case 'system':
      return estimateTextTokens(message.content)

    default:
      return 0
  }
}
```

### 8.2.2 精确计数

```typescript
// src/utils/tokens.ts
/**
 * 从 API 响应中获取精确 token 数
 */
export function finalContextTokensFromLastResponse(
  messages: Message[],
): number {
  // 找到最后一条助手消息
  const lastAssistant = messages.filter(m => m.type === 'assistant').at(-1)

  if (!lastAssistant) {
    return 0
  }

  const usage = (lastAssistant as AssistantMessage).message.usage
  if (!usage) {
    return 0
  }

  // 计算总 token 数
  return (
    usage.input_tokens +
    (usage.cache_creation_input_tokens || 0) +
    (usage.cache_read_input_tokens || 0) +
    usage.output_tokens
  )
}
```

## 8.3 预算追踪器

### 8.3.1 创建追踪器

```typescript
// src/query/tokenBudget.ts
export function createBudgetTracker() {
  let totalInputTokens = 0
  let totalOutputTokens = 0
  let totalCacheCreationTokens = 0
  let totalCacheReadTokens = 0
  let continuationCount = 0
  let budgetStartTime = Date.now()

  return {
    addInputTokens: (count: number) => {
      totalInputTokens += count
    },

    addOutputTokens: (count: number) => {
      totalOutputTokens += count
    },

    addCacheCreationTokens: (count: number) => {
      totalCacheCreationTokens += count
    },

    addCacheReadTokens: (count: number) => {
      totalCacheReadTokens += count
    },

    incrementContinuation: () => {
      continuationCount++
    },

    getTotal: () => ({
      input: totalInputTokens,
      output: totalOutputTokens,
      cacheCreation: totalCacheCreationTokens,
      cacheRead: totalCacheReadTokens,
      all: totalInputTokens + totalOutputTokens,
    }),

    getContinuationCount: () => continuationCount,

    getElapsedTime: () => Date.now() - budgetStartTime,
  }
}
```

### 8.3.2 预算检查

```typescript
// src/query/tokenBudget.ts
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  turnBudget: number,
  turnOutputTokens: number,
): BudgetDecision {
  const total = tracker.getTotal()
  const elapsed = tracker.getElapsedTime()

  // 检查是否超过每轮预算
  if (turnOutputTokens > turnBudget) {
    return {
      action: 'warn',
      message: `This turn used ${turnOutputTokens} tokens, exceeding the ${turnBudget} budget.`,
    }
  }

  // 检查是否超过总预算
  const maxBudget = getMaxBudgetUsd() // 从设置获取
  const estimatedCost = calculateCost(total)

  if (estimatedCost > maxBudget) {
    return {
      action: 'stop',
      reason: 'budget_exceeded',
      message: `Budget limit reached. Consider using /compact or /clear.`,
    }
  }

  // 检查是否接近限制（80%）
  if (estimatedCost > maxBudget * 0.8) {
    return {
      action: 'warn',
      message: `Approaching budget limit (${(estimatedCost / maxBudget * 100).toFixed(0)}%).`,
    }
  }

  return { action: 'allow' }
}
```

## 8.4 Turn 级别预算

### 8.4.1 获取当前 Turn 预算

```typescript
// src/bootstrap/state.ts
export function getCurrentTurnTokenBudget(): number {
  // 从环境变量或设置获取
  const envBudget = process.env.CLAUDE_CODE_TURN_BUDGET
  if (envBudget) {
    return parseInt(envBudget, 10)
  }

  // 从 AppState 获取
  const appState = getAppState()
  return appState.settings.maxTurnTokens || DEFAULT_TURN_BUDGET
}

export function getTurnOutputTokens(): number {
  // 从 AppState 获取当前 turn 的输出 token 数
  const appState = getAppState()
  return appState.currentTurnOutputTokens || 0
}
```

### 8.4.2 增量计数

```typescript
// src/bootstrap/state.ts
let currentTurnInputTokens = 0
let currentTurnOutputTokens = 0

export function incrementTurnInputTokens(count: number): void {
  currentTurnInputTokens += count
}

export function incrementTurnOutputTokens(count: number): void {
  currentTurnOutputTokens += count
}

export function resetTurnCounters(): void {
  currentTurnInputTokens = 0
  currentTurnOutputTokens = 0
}
```

## 8.5 预算耗尽处理

### 8.5.1 继续信号

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
        createUserMessage({
          content: decision.nudgeMessage,
          isMeta: true,
        }),
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

### 8.5.2 恢复消息

```typescript
// 预算恢复时发送的消息
const BUDGET_NUDGE_MESSAGES = [
  'You have used a significant amount of tokens. Consider summarizing progress so far.',
  'Token budget is running low. Focus on completing the current task.',
  'Consider using /compact to free up context space.',
]
```

## 8.6 Task Budget (API 级别)

### 8.6.1 API 参数

```typescript
// src/query.ts:181-198
interface QueryParams {
  // ...
  // API task_budget (output_config.task_budget, beta task-budgets-2026-03-13)
  taskBudget?: { total: number }
}

// 传递给 API
const response = await callModel({
  // ...
  options: {
    // ...
    ...(params.taskBudget && {
      taskBudget: {
        total: params.taskBudget.total,
        ...(taskBudgetRemaining !== undefined && {
          remaining: taskBudgetRemaining,
        }),
      },
    }),
  },
})
```

### 8.6.2 跨压缩边界追踪

```typescript
// src/query.ts:504-515
// 压缩前：记录剩余预算
if (params.taskBudget) {
  const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

## 8.7 成本计算

### 8.7.1 模型定价

```typescript
// src/utils/cost.ts
const MODEL_PRICING = {
  'claude-opus-4-5': {
    input: 15,      // $15 / 1M tokens
    output: 75,     // $75 / 1M tokens
    cacheCreation: 18.75,
    cacheRead: 1.5,
  },
  'claude-sonnet-4-5': {
    input: 3,       // $3 / 1M tokens
    output: 15,     // $15 / 1M tokens
    cacheCreation: 3.75,
    cacheRead: 0.3,
  },
  // ...
}
```

### 8.7.2 计算函数

```typescript
// src/utils/cost.ts
export function calculateCost(
  tokens: {
    input: number
    output: number
    cacheCreation?: number
    cacheRead?: number
  },
  model: string,
): number {
  const pricing = MODEL_PRICING[model]
  if (!pricing) {
    throw new Error(`Unknown model: ${model}`)
  }

  const inputCost = (tokens.input / 1_000_000) * pricing.input
  const outputCost = (tokens.output / 1_000_000) * pricing.output
  const cacheCreationCost = tokens.cacheCreation
    ? (tokens.cacheCreation / 1_000_000) * pricing.cacheCreation
    : 0
  const cacheReadCost = tokens.cacheRead
    ? (tokens.cacheRead / 1_000_000) * pricing.cacheRead
    : 0

  return inputCost + outputCost + cacheCreationCost + cacheReadCost
}
```

## 8.8 预算配置

### 8.8.1 用户配置

```typescript
// src/settings.ts
interface Settings {
  // 最大预算（美元）
  maxBudgetUsd?: number

  // 每轮最大 token 数
  maxTurnTokens?: number

  // 是否启用预算警告
  enableBudgetWarnings?: boolean

  // 警告阈值（百分比）
  budgetWarningThreshold?: number
}
```

### 8.8.2 环境变量

```bash
# 环境变量配置
CLAUDE_CODE_MAX_BUDGET=10          # 最多 $10
CLAUDE_CODE_TURN_BUDGET=8000       # 每轮最多 8000 output tokens
CLAUDE_CODE_DISABLE_BUDGET=1       # 禁用预算控制
```

## 8.9 设计模式

### 8.9.1 预算拦截器

```typescript
// 在 API 调用前检查预算
async function callModelWithBudgetCheck(params) {
  const budgetDecision = checkBudget(params)

  if (budgetDecision.action === 'stop') {
    throw new BudgetExceededError(budgetDecision.message)
  }

  if (budgetDecision.action === 'warn') {
    // 在消息前添加警告
    params.messages = [
      createSystemMessage(budgetDecision.message, 'warning'),
      ...params.messages,
    ]
  }

  return callModel(params)
}
```

### 8.9.2 渐进式警告

```typescript
// 不同阶段的警告级别
function getWarningLevel(percentUsed: number): 'info' | 'warn' | 'critical' {
  if (percentUsed < 50) return 'info'
  if (percentUsed < 80) return 'warn'
  return 'critical'
}
```

## 8.10 本章小结

Token 预算管理是 Agent 成本控制的核心：

| 组件 | 职责 |
|------|------|
| `tokenCountWithEstimation` | 估算 token 数 |
| `createBudgetTracker` | 创建预算追踪器 |
| `checkTokenBudget` | 检查预算状态 |
| `calculateCost` | 计算美元成本 |
| `getCurrentTurnTokenBudget` | 获取当前轮预算 |

**核心设计决策**：

1. **双层预算** — Turn 级别 + API 级别
2. **跨压缩追踪** — 压缩边界后继续追踪剩余预算
3. **渐进式警告** — 不同阶段不同提示
4. **可配置** — 环境变量 + 设置文件

## 8.11 实践要点

1. **选择合适的预算值** — 根据任务复杂度设置
2. **监控实际消耗** — 定期检查与估算的差异
3. **设置合理的警告阈值** — 80% 是常用值
4. **考虑压缩成本** — 压缩也消耗 token

---

## 下一步

下一章我们将学习 **自动压缩系统**，了解：
- Auto-Compact 触发和执行
- Reactive-Compact 恢复机制
- Context-Collapse 主动压缩
