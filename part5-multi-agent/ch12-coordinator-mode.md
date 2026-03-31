# 第 12 章：协调者模式 — Coordinator Mode

> "一个协调者指挥多个工作者，这才是真正的多 Agent 协作。"

## 12.1 协调者模式概述

协调者模式是 Claude Code 的**多 Agent 协作架构**。主 Agent 变成协调者，将任务分配给多个 Worker Agent。

### 12.1.1 核心概念

```
用户请求
    │
    ▼
┌─────────────────────────────────────┐
│           Coordinator (协调者)         │
│  - 理解用户意图                       │
│  - 分解任务                          │
│  - 分配给 Workers                     │
│  - 聚合结果                          │
└─────────────────────────────────────┘
    │
    ├──► Worker 1 (API层重构)
    ├──► Worker 2 (数据库优化)
    └──► Worker 3 (测试覆盖)
```

### 12.1.2 与普通 Agent 的区别

| 方面 | 普通 Agent | 协调者模式 |
|------|-----------|-----------|
| 工具 | 使用全部工具 | 使用 Agent/SendMessage/TaskStop |
| 目标 | 直接完成用户请求 | 协调 Workers 完成 |
| 子任务 | 不使用或很少使用 | 频繁使用 |
| 消息来源 | 用户输入 | Worker 结果 |

## 12.2 启用协调者模式

### 12.2.1 特性开关

```typescript
// src/coordinator/coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  // 1. 检查特性开关是否启用
  if (feature('COORDINATOR_MODE')) {
    // 2. 检查环境变量
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

### 12.2.2 环境变量配置

```bash
# 启用协调者模式
export CLAUDE_CODE_COORDINATOR_MODE=1

# 简单模式（只允许基础工具）
export CLAUDE_CODE_SIMPLE=1
```

## 12.3 协调者系统提示词

### 12.3.1 核心提示词

```typescript
// src/coordinator/coordinatorMode.ts:111-148
export function getCoordinatorSystemPrompt(): string {
  const workerCapabilities = isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
    ? 'Workers have access to Bash, Read, and Edit tools, plus MCP tools from configured MCP servers.'
    : 'Workers have access to standard tools, MCP tools from configured MCP servers, and project skills via the Skill tool. Delegate skill invocations (e.g. /commit, /verify) to workers.'

  return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role

You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that you can handle without tools

Every message you send is to the user. Worker results and system notifications are internal signals, not conversation partners — never thank or acknowledge them. Summarize new information for the user as it arrives.

## 2. Your Tools

- **Agent** - Spawn a new worker
- **SendMessage** - Continue an existing worker (send a follow-up to its \`to\` agent ID)
- **TaskStop** - Stop a running worker
- **subscribe_pr_activity / unsubscribe_pr_activity** (if available) - Subscribe to GitHub PR events

## 3. Your Capabilities

${workerCapabilities}
`
}
```

### 12.3.2 工作者上下文

```typescript
// src/coordinator/coordinatorMode.ts:80-109
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  if (!isCoordinatorMode()) {
    return {}
  }

  const workerTools = isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
    ? [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME].sort().join(', ')
    : Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
        .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
        .sort()
        .join(', ')

  let content = `Workers spawned via the Agent tool have access to these tools: ${workerTools}`

  if (mcpClients.length > 0) {
    const serverNames = mcpClients.map(c => c.name).join(', ')
    content += `\n\nWorkers also have access to MCP tools from connected MCP servers: ${serverNames}`
  }

  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers can read and write here without permission prompts. Use this for durable cross-worker knowledge.`
  }

  return { workerToolsContext: content }
}
```

## 12.4 Worker 工具限制

### 12.4.1 简单模式

```typescript
// 简单模式：只允许基础读写工具
const workerTools = [
  BASH_TOOL_NAME,      // "Bash"
  FILE_READ_TOOL_NAME, // "Read"
  FILE_EDIT_TOOL_NAME, // "Edit"
]
```

### 12.4.2 完整模式

```typescript
// 完整模式：允许所有异步工具
const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
```

### 12.4.3 内部工具过滤

```typescript
// 这些工具对 Worker 不可见
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,    // "TeamCreate"
  TEAM_DELETE_TOOL_NAME,    // "TeamDelete"
  SEND_MESSAGE_TOOL_NAME,   // "SendMessage"
  SYNTHETIC_OUTPUT_TOOL_NAME, // "SyntheticOutput"
])

// Worker 不能创建团队或发送跨团队消息
const visibleTools = workerTools.filter(
  tool => !INTERNAL_WORKER_TOOLS.has(tool)
)
```

## 12.5 Worker 执行

### 12.5.1 启动 Worker

```typescript
// src/tools/AgentTool/AgentTool.tsx
async function call(input, context, canUseTool, parentMessage, onProgress) {
  // 1. 创建 Worker 上下文
  const workerContext = await createWorkerContext({
    parentContext: context,
    agentType: 'worker',
    allowedTools: getWorkerTools(),  // 应用工具限制
  })

  // 2. 添加 Worker 标记
  const workerMessage = createUserMessage({
    content: input.prompt,
    agentId: workerContext.agentId,
  })

  // 3. 执行 Worker
  const results = await executeWorker({
    context: workerContext,
    message: workerMessage,
    maxTurns: input.maxTurns,
  })

  return {
    data: {
      workerId: workerContext.agentId,
      results: results,
    },
  }
}
```

### 12.5.2 结果收集

```typescript
// src/tools/AgentTool/resultCollection.ts
async function collectWorkerResults(
  workerQuery: AsyncGenerator<StreamEvent>,
): Promise<WorkerResult> {
  const messages: Message[] = []
  const toolCalls: ToolCall[] = []

  for await (const event of workerQuery) {
    switch (event.type) {
      case 'message':
        messages.push(event.message)
        break
      case 'tool_use':
        toolCalls.push(event.tool)
        break
      case 'progress':
        // 报告进度给协调者
        onProgress?.(event)
        break
    }
  }

  return {
    workerId: context.agentId,
    messages,
    toolCalls,
    finalResponse: extractFinalResponse(messages),
  }
}
```

## 12.6 任务通知机制

### 12.6.1 Worker 完成通知

```typescript
// Worker 完成后，发送 task-notification
interface TaskNotification {
  type: 'task-notification'
  taskId: string
  status: 'completed' | 'failed' | 'stopped'
  result?: string
  error?: string
}
```

### 12.6.2 协调者处理

```typescript
// src/coordinator/coordinatorMode.ts
// 协调者系统提示词中定义：
/*
Worker results arrive as **user-role messages** containing \<task-notification\> XML.

Format:
\`\`\`xml
<task-notification>
<task-id>{agentId}</task-id>
<status>{completed|failed|stopped}</status>
<result>...</result>
</task-notification>
\`\`\`
*/
```

## 12.7 会话模式匹配

### 12.7.1 恢复时模式检查

```typescript
// src/coordinator/coordinatorMode.ts:49-78
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  if (!sessionMode) {
    return undefined  // 旧会话，没有模式记录
  }

  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'

  if (currentIsCoordinator === sessionIsCoordinator) {
    return undefined  // 模式匹配
  }

  // 模式不匹配，需要切换
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }

  return sessionIsCoordinator
    ? 'Entered coordinator mode to match resumed session.'
    : 'Exited coordinator mode to match resumed session.'
}
```

## 12.8 设计模式

### 12.8.1 协调者模式模板

```typescript
class Coordinator {
  async coordinate(task: string): Promise<Result> {
    // 1. 理解任务
    const subtasks = await this.decompose(task)

    // 2. 启动 Workers
    const workers = await Promise.all(
      subtasks.map(subtask => this.spawnWorker(subtask))
    )

    // 3. 收集结果
    const results = await Promise.all(
      workers.map(worker => worker.waitForCompletion())
    )

    // 4. 聚合结果
    return this.synthesize(results)
  }
}
```

### 12.8.2 工具限制模式

```typescript
// 在子 Agent 中限制工具
function createWorkerContext(options: {
  parentContext: ToolUseContext
  allowedTools: string[]
}): ToolUseContext {
  return {
    ...cloneDeep(parentContext),
    options: {
      ...parentContext.options,
      tools: parentContext.options.tools.filter(
        tool => options.allowedTools.includes(tool.name)
      ),
    },
  }
}
```

## 12.9 本章小结

协调者模式实现了真正的多 Agent 协作：

| 组件 | 职责 |
|------|------|
| `isCoordinatorMode` | 检查是否启用 |
| `getCoordinatorSystemPrompt` | 协调者系统提示词 |
| `getWorkerTools` | Worker 工具限制 |
| `matchSessionMode` | 会话恢复时模式匹配 |

**核心设计决策**：

1. **工具限制** — Worker 只能使用必要的工具
2. **结果通知** — 通过 XML 格式的任务通知
3. **会话匹配** — 恢复时自动匹配协调者模式
4. **Scratchpad** — 跨 Worker 共享知识的机制

## 12.10 实践要点

1. **何时使用** — 任务可并行分解且需要协调时
2. **工具粒度** — 根据任务复杂度选择简单/完整模式
3. **结果聚合** — 协调者负责合成 Worker 输出
4. **错误处理** — Worker 失败时协调者的恢复策略

---

## 下一步

下一章我们将学习 **团队系统**，了解更复杂的多 Agent 协作机制。
