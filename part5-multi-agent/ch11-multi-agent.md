# 第 11 章：子 Agent 与 AgentTool

> "一个 Agent 可以启动其他 Agent，这是构建复杂任务分解的基础。"

## 11.1 多 Agent 架构概述

当任务过于复杂时，单一 Agent 可能无法有效处理。多 Agent 架构允许：

1. **任务分解** — 将大任务拆分为子任务
2. **并行执行** — 多个子 Agent 同时处理不同任务
3. **专业分工** — 不同 Agent 擅长不同领域
4. **上下文隔离** — 子 Agent 有独立的上下文

```
┌─────────────────────────────────────────────────────────┐
│                    Coordinator (协调者)                   │
│   "用户想要重构整个后端代码"                                │
└─────────────────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Worker1 │ │ Worker2 │ │ Worker3 │
   │ (API层)  │ │ (数据库) │ │ (测试)   │
   └─────────┘ └─────────┘ └─────────┘
```

## 11.2 AgentTool 接口

### 11.2.1 工具定义

```typescript
// src/tools/AgentTool/AgentTool.tsx
export const AgentTool = buildTool({
  name: AGENT_TOOL_NAME,  // "Agent"

  inputSchema: z.object({
    // Agent 类型
    type: z.enum(['task', 'code', 'review', 'debug']).optional(),

    // 指令
    prompt: z.string()
      .describe('The instruction for the agent'),

    // 可选：指定模型
    model: z.string().optional(),

    // 可选：工作目录
    workingDirectory: z.string().optional(),

    // 可选：最大轮次
    maxTurns: z.number().optional(),
  }),

  // ...
})
```

### 11.2.2 执行签名

```typescript
// AgentTool.call 签名
async function call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<AgentToolProgress>,
): Promise<ToolResult<AgentResult>>
```

## 11.3 子 Agent 生命周期

### 11.3.1 创建上下文

```typescript
// src/query/createSubagentContext.ts
async function createSubagentContext({
  parentContext,
  agentId,
  agentType,
  input,
}: {
  parentContext: ToolUseContext
  agentId: AgentId
  agentType: string
  input: AgentInput
}): Promise<ToolUseContext> {
  // 1. 生成新的 UUID
  const subAgentId = agentId || uuid()

  // 2. 克隆父上下文（隔离但共享部分状态）
  const subagentContext: ToolUseContext = {
    ...cloneDeep(parentContext),

    // 3. 新的 AbortController（子 Agent 可独立中止）
    abortController: new AbortController(),

    // 4. 独立的文件读取状态（不污染父 Agent）
    readFileState: new FileStateCache(),

    // 5. 子 Agent 标记
    agentId: subAgentId,
    agentType: agentType,

    // 6. 继承的工具配置
    options: {
      ...parentContext.options,
      // 可以覆盖工作目录
      workingDirectory: input.workingDirectory || parentContext.options.workingDirectory,
    },
  }

  return subagentContext
}
```

### 11.3.2 启动子 Agent

```typescript
// src/tools/AgentTool/AgentTool.tsx
async function call(input, context, canUseTool, parentMessage, onProgress) {
  // 1. 创建子 Agent 上下文
  const subagentContext = await createSubagentContext({
    parentContext: context,
    agentId: context.agentId,  // 传递父 agentId
    agentType: input.type,
    input,
  })

  // 2. 启动查询循环
  const agentQuery = query({
    messages: [
      createUserMessage({
        content: input.prompt,
        agentId: subagentContext.agentId,
      }),
    ],
    systemPrompt: getAgentSystemPrompt(input.type),
    toolUseContext: subagentContext,
    maxTurns: input.maxTurns,
    // ...
  })

  // 3. 收集结果
  const messages: Message[] = []
  for await (const event of agentQuery) {
    if (event.type === 'message') {
      messages.push(event)
    }
    // 报告进度
    onProgress?.({
      toolUseID: context.toolUseId!,
      data: { type: 'agent_progress', message: event },
    })
  }

  // 4. 提取最终响应
  const finalMessage = messages.filter(m => m.type === 'assistant').at(-1)

  return {
    data: {
      response: finalMessage?.message?.content || '',
      messages,
    },
  }
}
```

## 11.4 子 Agent 隔离机制

### 11.4.1 为什么需要隔离？

1. **防止状态污染** — 子 Agent 的文件读取不应影响父 Agent
2. **独立中止** — 可以单独停止某个子 Agent
3. **资源限制** — 子 Agent 有独立的超时和轮次限制
4. **权限控制** — 子 Agent 可能需要不同的权限配置

### 11.4.2 隔离的内容

```typescript
// 隔离的资源
{
  abortController: new AbortController(),     // 独立的信号控制
  readFileState: new FileStateCache(),         // 独立的文件追踪
  agentId: uuid(),                             // 独立的 ID
  messages: [],                                 // 独立的消息历史
}

// 共享的资源（克隆传递）
{
  toolPermissionContext: cloneDeep(parent),     // 共享权限配置
  settings: parent.settings,                    // 共享设置
  mcpClients: parent.mcpClients,               // 共享 MCP 连接
}
```

## 11.5 消息传递

### 11.5.1 父子通信

```typescript
// 父 Agent 可以发送消息给子 Agent
const response = await sendMessageToAgent({
  agentId: subAgentId,
  message: '请更新 API 文档',
})

// 子 Agent 的结果通过消息传递回来
interface AgentMessage {
  type: 'agent_response'
  agentId: AgentId
  content: string
  messages: Message[]
}
```

### 11.5.2 结果收集

```typescript
// src/tools/AgentTool/resultCollection.ts
async function collectAgentResults(
  agentQuery: AsyncGenerator<StreamEvent>,
): Promise<AgentResult> {
  const messages: Message[] = []
  const toolResults: ToolResult[] = []

  for await (const event of agentQuery) {
    switch (event.type) {
      case 'message':
        messages.push(event.message)
        break
      case 'tool_use':
        // 处理工具调用
        break
      case 'progress':
        // 报告进度
        break
    }
  }

  return {
    response: extractFinalResponse(messages),
    messages,
    toolResults,
  }
}
```

## 11.6 Agent 类型

### 11.6.1 内置类型

```typescript
// src/tools/AgentTool/agentTypes.ts
export const AGENT_TYPES = {
  task: {
    name: 'Task Agent',
    description: 'General purpose task execution',
    systemPrompt: 'You are a task agent. Complete the given task...',
  },
  code: {
    name: 'Code Agent',
    description: 'Specialized in code changes',
    systemPrompt: 'You are a code agent. Focus on implementing...',
  },
  review: {
    name: 'Review Agent',
    description: 'Code review specialist',
    systemPrompt: 'You are a review agent. Analyze code quality...',
  },
  debug: {
    name: 'Debug Agent',
    description: 'Debugging specialist',
    systemPrompt: 'You are a debug agent. Investigate and fix...',
  },
}
```

### 11.6.2 自定义系统提示词

```typescript
// 通过 input.customSystemPrompt 自定义
const agentQuery = query({
  // ...
  customSystemPrompt: input.customSystemPrompt,
})
```

## 11.7 协调者模式 — Coordinator Mode

### 11.7.1 什么是协调者模式？

当启用 `COORDINATOR_MODE` 时，主 Agent 变成协调者，专门负责：

1. **分解任务** — 将用户请求拆分为子任务
2. **分配工作** — 将子任务分配给 Worker Agent
3. **监控进度** — 跟踪子 Agent 的执行状态
4. **聚合结果** — 整合子 Agent 的输出

### 11.7.2 配置

```typescript
// src/coordinator/coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

### 11.7.3 协调者系统提示词

```typescript
// src/coordinator/coordinatorMode.ts:111-148
export function getCoordinatorSystemPrompt(): string {
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
- **SendMessage** - Continue an existing worker
- **TaskStop** - Stop a running worker

## 3. Delegation Strategy

Do NOT use one worker to check on another. Workers will notify you when they are done.
Do NOT use workers to trivially report file contents or run commands. Give them higher-level tasks.
`
}
```

## 11.8 Worker 工具限制

### 11.8.1 简单模式

```typescript
// 仅允许基础工具
const workerTools = [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
```

### 11.8.2 完整模式

```typescript
// 允许所有异步工具
const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
```

### 11.8.3 工具过滤

```typescript
// src/coordinator/coordinatorMode.ts:29-34
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])

// 内部工具对 Worker 不可见
const visibleTools = workerTools.filter(
  tool => !INTERNAL_WORKER_TOOLS.has(tool)
)
```

## 11.9 团队系统 — Team

### 11.9.1 TeamCreateTool

```typescript
// src/tools/TeamCreateTool/TeamCreateTool.ts
export const TeamCreateTool = buildTool({
  name: 'TeamCreate',

  inputSchema: z.object({
    name: z.string(),
    agents: z.array(z.object({
      name: z.string(),
      type: z.string(),
      capabilities: z.array(z.string()),
    })),
  }),

  async call(input, context) {
    // 创建团队
    const team = await createTeam({
      name: input.name,
      agents: input.agents.map(a => ({
        id: uuid(),
        name: a.name,
        type: a.type,
        capabilities: a.capabilities,
      })),
    })

    return { data: { teamId: team.id } }
  },
})
```

### 11.9.2 SendMessageTool

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts
export const SendMessageTool = buildTool({
  name: 'SendMessage',

  inputSchema: z.object({
    to: z.string().describe('Agent ID to send message to'),
    message: z.string(),
  }),

  async call(input, context) {
    // 向指定 Agent 发送消息
    const response = await sendMessage({
      to: input.to,
      message: input.message,
    })

    return { data: { response } }
  },
})
```

## 11.10 设计模式总结

### 11.10.1 Agent 工厂模式

```typescript
// 创建子 Agent 的标准流程
async function createAgent(
  type: AgentType,
  prompt: string,
  options?: AgentOptions,
): Promise<AgentContext> {
  const context = await createSubagentContext({
    parentContext: getCurrentContext(),
    agentId: uuid(),
    agentType: type,
  })

  const systemPrompt = getAgentSystemPrompt(type)

  return {
    context,
    query: query({
      messages: [createUserMessage({ content: prompt })],
      systemPrompt,
      toolUseContext: context,
      ...options,
    }),
  }
}
```

### 11.10.2 结果聚合模式

```typescript
// 聚合多个 Agent 的结果
async function aggregateResults(
  agentQueries: AsyncGenerator<StreamEvent>[],
): Promise<AggregatedResult> {
  const results = await Promise.all(
    agentQueries.map(collectAgentResults)
  )

  return {
    responses: results.map(r => r.response),
    allMessages: results.flatMap(r => r.messages),
    summary: synthesize(results.map(r => r.response)),
  }
}
```

### 11.10.3 监督者模式

```typescript
// 监督子 Agent 执行
async function supervise(
  agentQuery: AsyncGenerator<StreamEvent>,
  onProgress: (event: StreamEvent) => void,
): Promise<SupervisionResult> {
  let turnCount = 0
  let lastMessage: Message | null = null

  for await (const event of agentQuery) {
    onProgress(event)
    turnCount++

    if (event.type === 'message') {
      lastMessage = event.message
    }

    // 超时检查
    if (turnCount > MAX_TURNS) {
      return { status: 'timeout', lastMessage }
    }
  }

  return { status: 'completed', lastMessage }
}
```

## 11.11 本章小结

AgentTool 实现了多 Agent 架构的基础：

| 组件 | 职责 |
|------|------|
| `createSubagentContext` | 创建隔离的子 Agent 上下文 |
| `AgentTool` | 启动和管理子 Agent |
| `Coordinator Mode` | 主 Agent 作为协调者 |
| `Team System` | 多 Agent 协作框架 |

**关键设计决策**：

1. **上下文隔离** — 独立的 AbortController、FileStateCache
2. **消息传递** — 通过消息系统通信
3. **工具限制** — Worker 只能访问必要的工具
4. **协调者模式** — 主 Agent 专门负责协调

## 11.12 实践要点

1. **何时拆分** — 任务可并行执行或需要不同专业知识时
2. **隔离程度** — 根据需求选择共享或隔离资源
3. **通信协议** — 定义清晰的 Agent 间消息格式
4. **错误处理** — 子 Agent 失败时协调者的恢复策略

---

## 下一步

下一章我们将学习 **MCP 协议集成**，了解：
- Model Context Protocol 客户端实现
- 多传输协议支持
- MCP 工具如何包装为 Claude Code 工具
