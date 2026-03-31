# 第 13 章：团队协作 — Team System

> "团队是多 Agent 协作的高级形式，Agent 之间可以相互通信。"

## 13.1 团队系统概述

团队系统是比协调者模式更复杂的多 Agent 协作机制：

```
┌─────────────────────────────────────────────────┐
│                    Team                          │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────┐│
│  │  Agent A    │◄─┼─► Agent B  │  │  Agent C  ││
│  │  (Leader)   │  │  (Member)  │◄─┼─►(Member) ││
│  └─────────────┘  └─────────────┘  └───────────┘│
│         │                                    │
│         └──────────► Scratchpad ◄────────────┘│
└─────────────────────────────────────────────────┘
```

### 13.1.1 与协调者模式的区别

| 方面 | 协调者模式 | 团队系统 |
|------|-----------|-----------|
| 通信方式 | Worker 向协调者单向报告 | Agent 之间双向通信 |
| 关系 | 协调者-Worker | 对等成员 |
| 消息 | SendMessage 发送给协调者 | 任意 Agent 之间 |
| 创建 | 协调者启动 Worker | 团队创建时指定 |

## 13.2 TeamCreateTool

### 13.2.1 工具定义

```typescript
// src/tools/TeamCreateTool/TeamCreateTool.ts
export const TeamCreateTool = buildTool({
  name: TEAM_CREATE_TOOL_NAME,  // "TeamCreate"

  inputSchema: z.object({
    name: z.string()
      .describe('Team name'),
    agents: z.array(z.object({
      name: z.string()
        .describe('Agent name'),
      type: z.string()
        .describe('Agent type (e.g., "code", "review", "research")'),
      role: z.string().optional()
        .describe('Agent role description'),
    })).describe('Array of agent configurations'),
    scratchpadDir: z.string().optional()
      .describe('Directory for cross-agent shared knowledge'),
  }),

  async call(input, context) {
    // 创建团队
    const team = await createTeam({
      name: input.name,
      agents: input.agents.map(agent => ({
        id: uuid(),
        name: agent.name,
        type: agent.type,
        role: agent.role,
      })),
      scratchpadDir: input.scratchpadDir,
    })

    return {
      data: {
        teamId: team.id,
        agents: team.agents.map(a => ({ id: a.id, name: a.name })),
      },
    }
  },
})
```

### 13.2.2 团队创建

```typescript
// src/team/createTeam.ts
interface CreateTeamParams {
  name: string
  agents: Array<{
    id: string
    name: string
    type: string
    role?: string
  }>
  scratchpadDir?: string
}

export async function createTeam(params: CreateTeamParams): Promise<Team> {
  // 1. 创建团队记录
  const team: Team = {
    id: uuid(),
    name: params.name,
    agents: [],
    createdAt: Date.now(),
  }

  // 2. 创建 Agent 实例
  for (const agentConfig of params.agents) {
    const agent = await createTeamAgent({
      ...agentConfig,
      teamId: team.id,
    })
    team.agents.push(agent)
  }

  // 3. 设置共享存储
  if (params.scratchpadDir) {
    team.scratchpadDir = params.scratchpadDir
  }

  // 4. 注册团队
  await registerTeam(team)

  return team
}
```

## 13.3 TeamDeleteTool

### 13.3.1 工具定义

```typescript
// src/tools/TeamDeleteTool/TeamDeleteTool.ts
export const TeamDeleteTool = buildTool({
  name: TEAM_DELETE_TOOL_NAME,  // "TeamDelete"

  inputSchema: z.object({
    teamId: z.string()
      .describe('Team ID to delete'),
    reason: z.string().optional()
      .describe('Reason for deletion'),
  }),

  async call(input, context) {
    // 1. 停止所有团队 Agent
    await stopAllTeamAgents(input.teamId)

    // 2. 清理资源
    await cleanupTeamResources(input.teamId)

    // 3. 注销团队
    await unregisterTeam(input.teamId)

    return {
      data: {
        teamId: input.teamId,
        deleted: true,
      },
    }
  },
})
```

## 13.4 SendMessageTool

### 13.4.1 工具定义

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts
export const SendMessageTool = buildTool({
  name: SEND_MESSAGE_TOOL_NAME,  // "SendMessage"

  inputSchema: z.object({
    to: z.string()
      .describe('Agent ID to send message to'),
    message: z.string()
      .describe('Message content'),
    priority: z.enum(['normal', 'high']).optional()
      .describe('Message priority'),
  }),

  async call(input, context) {
    // 1. 验证接收者存在
    const recipient = await getAgentById(input.to)
    if (!recipient) {
      return { data: { error: 'Agent not found' } }
    }

    // 2. 发送消息
    const message = await sendMessageToAgent({
      to: input.to,
      content: input.message,
      from: context.agentId,
      priority: input.priority,
    })

    return {
      data: {
        messageId: message.id,
        delivered: true,
      },
    }
  },
})
```

### 13.4.2 消息传递

```typescript
// src/team/messagePassing.ts
interface TeamMessage {
  id: string
  from: string
  to: string
  content: string
  timestamp: number
  priority: 'normal' | 'high'
}

export async function sendMessageToAgent(params: {
  to: string
  content: string
  from: string
  priority?: 'normal' | 'high'
}): Promise<TeamMessage> {
  const message: TeamMessage = {
    id: uuid(),
    from: params.from,
    to: params.to,
    content: params.content,
    timestamp: Date.now(),
    priority: params.priority || 'normal',
  }

  // 添加到接收者队列
  await addToMessageQueue(message)

  return message
}
```

## 13.5 团队 Agent 通信

### 13.5.1 消息队列

```typescript
// src/team/messageQueue.ts
class TeamMessageQueue {
  private queues: Map<string, TeamMessage[]> = new Map()

  add(message: TeamMessage): void {
    const queue = this.queues.get(message.to) || []
    queue.push(message)
    this.queues.set(message.to, queue)
  }

  getNext(agentId: string): TeamMessage | null {
    const queue = this.queues.get(agentId)
    if (!queue || queue.length === 0) {
      return null
    }
    return queue.shift()!
  }

  peek(agentId: string): TeamMessage | null {
    return this.queues.get(agentId)?.[0] || null
  }
}
```

### 13.5.2 消息作为用户输入

```typescript
// 当 Agent 拉取消息时，转换为用户消息格式
async function pollForMessages(agentId: string): Promise<UserMessage | null> {
  const message = messageQueue.getNext(agentId)

  if (!message) {
    return null
  }

  return {
    type: 'user',
    message: {
      role: 'user',
      content: [{
        type: 'text',
        text: `[Message from ${message.from}]: ${message.content}`,
      }],
    },
    metadata: {
      messageId: message.id,
      priority: message.priority,
    },
  }
}
```

## 13.6 Scratchpad 共享

### 13.6.1 共享存储

```typescript
// src/team/scratchpad.ts
interface ScratchpadStore {
  read(key: string): Promise<string | null>
  write(key: string, value: string): Promise<void>
  delete(key: string): Promise<void>
  list(): Promise<string[]>
}

export function createScratchpad(dir: string): ScratchpadStore {
  return {
    async read(key) {
      const path = `${dir}/${key}`
      return fileExists(path) ? readFile(path) : null
    },

    async write(key, value) {
      const path = `${dir}/${key}`
      await writeFile(path, value)
    },

    async delete(key) {
      const path = `${dir}/${key}`
      await deleteFile(path)
    },

    async list() {
      return listFiles(dir)
    },
  }
}
```

### 13.6.2 在 Agent 中使用

```typescript
// Agent 可以读写 scratchpad
const scratchpad = createScratchpad(team.scratchpadDir)

// 写入
await scratchpad.write('api-design.md', '# API Design\n\n...')

// 读取
const design = await scratchpad.read('api-design.md')
```

## 13.7 团队生命周期

### 13.7.1 状态管理

```typescript
// src/team/teamState.ts
interface TeamState {
  teams: Map<string, Team>
  agentTeams: Map<string, string>  // agentId -> teamId
}

export const teamState: TeamState = {
  teams: new Map(),
  agentTeams: new Map(),
}

export function getTeam(teamId: string): Team | undefined {
  return teamState.teams.get(teamId)
}

export function getAgentTeam(agentId: string): Team | undefined {
  const teamId = teamState.agentTeams.get(agentId)
  return teamId ? teamState.teams.get(teamId) : undefined
}
```

### 13.7.2 清理

```typescript
// 团队删除时清理所有资源
async function cleanupTeam(teamId: string): Promise<void> {
  const team = teamState.teams.get(teamId)
  if (!team) return

  // 1. 停止所有 Agent
  for (const agent of team.agents) {
    await stopAgent(agent.id)
  }

  // 2. 清理消息队列
  for (const agent of team.agents) {
    messageQueue.remove(agent.id)
  }

  // 3. 清理 scratchpad
  if (team.scratchpadDir) {
    await deleteDirectory(team.scratchpadDir)
  }

  // 4. 从状态中移除
  for (const agent of team.agents) {
    teamState.agentTeams.delete(agent.id)
  }
  teamState.teams.delete(teamId)
}
```

## 13.8 设计模式

### 13.8.1 团队工厂

```typescript
// 创建团队的工厂函数
async function createSoftwareTeam(
  project: ProjectSpec,
): Promise<Team> {
  return createTeam({
    name: `${project.name} Team`,
    agents: [
      { name: 'Tech Lead', type: 'coordinator', role: 'Coordinate all work' },
      { name: 'Backend Dev', type: 'code', role: 'Implement backend' },
      { name: 'Frontend Dev', type: 'code', role: 'Implement frontend' },
      { name: 'QA Engineer', type: 'review', role: 'Test and review' },
    ],
    scratchpadDir: `/tmp/${project.id}/scratchpad`,
  })
}
```

### 13.8.2 消息协议

```typescript
// 定义 Agent 之间的消息协议
const MessageProtocol = {
  // 请求
  REQUEST_REVIEW: 'REQUEST_REVIEW',
  REPORT_PROGRESS: 'REPORT_PROGRESS',
  REQUEST_HELP: 'REQUEST_HELP',

  // 响应
  REVIEW_COMPLETE: 'REVIEW_COMPLETE',
  HELP_PROVIDED: 'HELP_PROVIDED',

  // 解析
  parse(content: string): ProtocolMessage {
    const [type, ...body] = content.split(':')
    return { type, payload: body.join(':') }
  },
}
```

## 13.9 本章小结

团队系统提供了完整的多 Agent 协作框架：

| 组件 | 职责 |
|------|------|
| `TeamCreateTool` | 创建团队 |
| `TeamDeleteTool` | 删除团队 |
| `SendMessageTool` | Agent 间通信 |
| `ScratchpadStore` | 共享存储 |
| `TeamMessageQueue` | 消息队列 |

**核心设计决策**：

1. **双向通信** — Agent 之间可以直接通信
2. **共享存储** — Scratchpad 实现知识共享
3. **消息队列** — 异步通信机制
4. **生命周期管理** — 完整的创建/删除流程

## 13.10 实践要点

1. **团队规模** — 不要创建过多 Agent（一般 3-5 个）
2. **角色定义** — 每个 Agent 应该有清晰的职责
3. **消息协议** — 定义清晰的消息格式避免歧义
4. **清理** — 完成任务后及时删除团队

---

## 下一步

下一章我们将学习 **MCP 协议集成**，了解如何通过 Model Context Protocol 扩展工具能力。
