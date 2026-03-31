# 附录 A：源码速查表

## A.1 核心模块

| 功能 | 源码位置 | 核心行数 |
|------|---------|---------|
| 核心循环 | `src/query.ts` | 1730 行 |
| 查询引擎 | `src/QueryEngine.ts` | ~500 行 |
| 工具接口 | `src/Tool.ts` | ~500 行 |
| 工具注册 | `src/tools.ts` | ~350 行 |
| 消息系统 | `src/utils/messages.ts` | ~300 行 |

## A.2 工具实现

| 工具 | 源码位置 | 复杂度 |
|------|---------|--------|
| BashTool | `src/tools/BashTool/BashTool.tsx` | 高 (700+ 行) |
| FileReadTool | `src/tools/FileReadTool/` | 中 |
| FileEditTool | `src/tools/FileEditTool/` | 高 |
| AgentTool | `src/tools/AgentTool/` | 高 |
| GlobTool | `src/tools/GlobTool/` | 低 |
| GrepTool | `src/tools/GrepTool/` | 中 |
| WebSearchTool | `src/tools/WebSearchTool/` | 中 |

## A.3 服务层

| 服务 | 源码位置 | 描述 |
|------|---------|------|
| MCP 客户端 | `src/services/mcp/client.ts` | 119KB，最大的单一文件 |
| 工具编排 | `src/services/tools/toolOrchestration.ts` | 工具执行调度 |
| 流式工具执行 | `src/services/tools/StreamingToolExecutor.ts` | 工具流式执行 |
| 自动压缩 | `src/services/compact/autoCompact.ts` | 上下文压缩 |
| 响应式压缩 | `src/services/compact/reactiveCompact.ts` | 错误恢复压缩 |

## A.4 状态与存储

| 功能 | 源码位置 |
|------|---------|
| Store 实现 | `src/state/store.ts` |
| AppState | `src/state/AppState.tsx` |
| 历史管理 | `src/history.ts` |

## A.5 多 Agent

| 功能 | 源码位置 |
|------|---------|
| 协调者模式 | `src/coordinator/coordinatorMode.ts` |
| 子 Agent 创建 | `src/query/createSubagentContext.ts` |
| 团队管理 | `src/team/` |

## A.6 入口点

| 入口 | 源码位置 | 描述 |
|------|---------|------|
| CLI 引导 | `src/entrypoints/cli.tsx` | 快速命令路由 |
| 主入口 | `src/main.tsx` | 应用初始化 |
| REPL 启动 | `src/replLauncher.tsx` | 交互循环 |

---

# 附录 B：关键设计模式

## B.1 AsyncGenerator 模式

**用途**：实现流式响应和状态保持

```typescript
// 核心模式
async function* agentLoop(initialState: State) {
  let state = initialState

  while (true) {
    // 1. 发送请求
    yield { type: 'request_start' }

    // 2. 流式处理
    for await (const event of stream) {
      yield event
      state = updateState(state, event)
    }

    // 3. 检查停止条件
    if (shouldStop(state)) {
      return state
    }
  }
}
```

**适用场景**：
- 需要边处理边返回结果
- 状态需要在迭代间保持
- 处理可中断的操作

## B.2 Store 模式

**用途**：简单的响应式状态管理

```typescript
// 核心模式
function createStore<T>(initial: T) {
  let state = initial
  const listeners = new Set<() => void>()

  return {
    getState: () => state,
    setState: (updater) => {
      state = typeof updater === 'function' ? updater(state) : updater
      listeners.forEach(l => l())
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

**适用场景**：
- 需要响应式更新 UI
- 状态需要在多处共享
- 简单的状态管理需求

## B.3 Tool 策略模式

**用途**：统一接口，多种实现

```typescript
// 核心模式
const tool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({ ... }),

  async call(input, context, canUseTool) {
    // 实现
  },

  isConcurrencySafe: () => true,
  isReadOnly: () => false,
})
```

**适用场景**：
- 定义 Agent 可用的工具
- 需要统一的行为接口
- 工具需要元数据描述

## B.4 Feature Flag 模式

**用途**：条件加载和 A/B 测试

```typescript
// 核心模式
const ExpensiveTool = feature('EXPENSIVE_FEATURE')
  ? require('./ExpensiveTool.js').ExpensiveTool
  : null

// 使用
const tools = [
  BasicTool,
  ...(ExpensiveTool ? [ExpensiveTool] : []),
]
```

**适用场景**：
- 按环境启用/禁用功能
- A/B 测试
- 减少生产包大小

## B.5 命令分类模式

**用途**：区分操作类型，决定执行策略

```typescript
// 核心模式
function classifyCommand(command: string): Classification {
  if (READ_COMMANDS.has(command)) return 'read'
  if (WRITE_COMMANDS.has(command)) return 'write'
  if (SEARCH_COMMANDS.has(command)) return 'search'
  return 'other'
}

// 使用
const classification = classifyCommand(input)
if (classification === 'read' && allowParallel) {
  // 并行执行
} else {
  // 串行执行
}
```

**适用场景**：
- 根据操作类型决定执行策略
- UI 显示优化
- 权限控制

---

# 附录 C：下一步学习路径

## C.1 深入特定模块

### 1. Query Loop 深入
- `src/query.ts` 完整源码
- 实现自定义的 Agent 循环
- 添加新的错误恢复策略

### 2. 工具开发
- 在 `src/tools/` 添加新工具
- 实现完整的工具测试
- 参与开源工具开发

### 3. MCP 集成
- `src/services/mcp/client.ts` 源码
- 开发 MCP 服务器
- 连接自定义数据源

## C.2 构建自己的 Agent

### 1. 最小 Agent 模板

```typescript
// minimal-agent.ts
import { createStore } from './state/store'

interface State {
  messages: Message[]
  running: boolean
}

const store = createStore<State>({
  messages: [],
  running: true,
})

async function* agentLoop() {
  while (store.getState().running) {
    const state = store.getState()

    // 1. 生成请求
    const response = await callModel(state.messages)

    // 2.  Yield 流式响应
    for await (const chunk of response.stream) {
      yield chunk
    }

    // 3. 检查工具调用
    if (response.toolUse) {
      const result = await executeTool(response.toolUse)
      store.setState(prev => ({
        messages: [...prev.messages, result],
      }))
    }
  }
}
```

### 2. 添加工具

```typescript
const MyTool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({ input: z.string() }),

  async call(input, context) {
    // 实现
    return { data: { result: input.input } }
  },

  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

### 3. 添加状态持久化

```typescript
// 保存状态
async function saveState(state: State) {
  await writeFile('state.json', JSON.stringify(state))
}

// 加载状态
async function loadState(): Promise<State> {
  const data = await readFile('state.json')
  return JSON.parse(data)
}
```

## C.3 贡献开源

### 1. Claude Code 贡献
- 官方仓库：待定
- 提交 Issues
- Pull Requests

### 2. MCP 服务器开发
- 实现标准 MCP 接口
- 发布到 npm

### 3. 工具生态
- 开发并发布工具包
- 文档和测试

---

# 附录 D：常见问题

## D.1 如何调试 Query Loop？

```typescript
// 启用详细日志
export CLAUDE_CODE_VERBOSE=1

// 使用 queryCheckpoint
queryCheckpoint('debug_point')

// 在关键位置添加日志
console.log('State:', JSON.stringify(state, null, 2))
```

## D.2 如何添加新工具？

1. 在 `src/tools/` 创建目录
2. 实现工具类
3. 在 `src/tools.ts` 注册
4. 添加测试

## D.3 如何处理长上下文？

```typescript
// 启用自动压缩
export CLAUDE_CODE_AUTO_COMPACT=1

// 手动压缩
await runCompact()
```

## D.4 如何自定义权限？

```typescript
// 在配置文件中
{
  "permissions": {
    "alwaysAllow": {
      "Bash": {
        "commands": ["git *", "npm test"]
      }
    }
  }
}
```

---

# 附录 E：参考资源

## E.1 内部资源

- 源码：`~/code/cc-code/src/`
- 书籍大纲：`book/book-outline.md`
- 章节：`book/chapters/`

## E.2 外部资源

- [Zod 文档](https://zod.dev)
- [TypeScript 手册](https://www.typescriptlang.org/docs/)
- [AsyncGenerator MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncGenerator)
- [Model Context Protocol](https://modelcontextprotocol.io)

---

*本书基于 Claude Code 源码编写，源码版权属于 Anthropic PBC。*
