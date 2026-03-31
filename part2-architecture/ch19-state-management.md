# 第 19 章：状态管理架构 — State Management

> "没有状态，Agent 就是无根之木。"

## 19.1 状态系统概述

Claude Code 使用简单的 pub/sub 模式管理状态：

```
┌─────────────────────────────────────────────────────────┐
│                      Store<T>                           │
│  ┌─────────────────────────────────────────────────┐  │
│  │  state: T                                       │  │
│  │  listeners: Set<() => void>                    │  │
│  └─────────────────────────────────────────────────┘  │
│                          │                              │
│         ┌─────────────────┼─────────────────┐          │
│         ▼                 ▼                 ▼          │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐    │
│  │ getState() │    │ setState()│    │ subscribe()│    │
│  └────────────┘    └────────────┘    └────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## 19.2 Store 实现

### 19.2.1 核心接口

```typescript
// src/state/store.ts
export interface Store<T> {
  getState(): T
  setState(updater: (prev: T) => T): void
  subscribe(listener: () => void): () => void
}

export function createStore<T>(initial: T): Store<T> {
  let state = initial
  const listeners = new Set<() => void>()

  return {
    getState: () => state,

    setState: (updater) => {
      const next = typeof updater === 'function'
        ? (updater as (prev: T) => T)(state)
        : updater

      if (next !== state) {
        state = next
        // 通知所有监听器
        listeners.forEach(listener => listener())
      }
    },

    subscribe: (listener) => {
      listeners.add(listener)
      // 返回取消订阅函数
      return () => listeners.delete(listener)
    },
  }
}
```

### 19.2.2 使用示例

```typescript
// 创建 store
const counterStore = createStore({ count: 0 })

// 读取状态
console.log(counterStore.getState())  // { count: 0 }

// 更新状态
counterStore.setState(prev => ({ count: prev.count + 1 }))
console.log(counterStore.getState())  // { count: 1 }

// 订阅变更
const unsubscribe = counterStore.subscribe(() => {
  console.log('State changed:', counterStore.getState())
})

counterStore.setState(prev => ({ count: prev.count + 1 }))
// 输出: "State changed: { count: 2 }"

// 取消订阅
unsubscribe()
```

## 19.3 AppState

### 19.3.1 AppState 结构

```typescript
// src/state/AppState.tsx
interface AppState {
  // 设置
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting

  // 任务
  tasks: { [taskId: string]: TaskState }

  // MCP
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }

  // 插件
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: {...}
  }

  // 权限
  toolPermissionContext: ToolPermissionContext

  // UI
  // ...
}
```

### 19.3.2 AppStateStore

```typescript
// src/state/AppStateStore.ts
export const appStateStore = createStore<AppState>({
  settings: defaultSettings,
  verbose: false,
  mainLoopModel: 'claude-sonnet-4-5',
  tasks: {},
  mcp: {
    clients: [],
    tools: [],
    commands: [],
    resources: {},
    pluginReconnectKey: 0,
  },
  plugins: {
    enabled: [],
    disabled: [],
    commands: [],
    errors: [],
    installationStatus: {
      installing: [],
      updating: [],
      failed: new Map(),
    },
  },
  toolPermissionContext: getEmptyToolPermissionContext(),
})
```

## 19.4 React 集成

### 19.4.1 Context Provider

```typescript
// src/state/AppState.tsx
const AppStateContext = React.createContext<Store<AppState> | null>(null)

export function AppStateProvider({ children }: { children: React.ReactNode }) {
  return (
    <AppStateContext.Provider value={appStateStore}>
      {children}
    </AppStateContext.Provider>
  )
}
```

### 19.4.2 useAppState Hook

```typescript
// src/state/AppState.tsx
export function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useContext(AppStateContext)!

  // 使用 selector 选择需要的状态
  const [selected, setSelected] = useState(() => selector(store.getState()))

  useEffect(() => {
    // 订阅变更，只在 selected 变化时更新
    return store.subscribe(() => {
      const next = selector(store.getState())
      setSelected(prev => prev !== next ? next : prev)
    })
  }, [store, selector])

  return selected
}

// 使用示例
function TaskCounter() {
  const taskCount = useAppState(state => Object.keys(state.tasks).length)
  return <div>Tasks: {taskCount}</div>
}
```

### 19.4.3 useSetAppState

```typescript
// src/state/AppState.tsx
export function useSetAppState(): (updater: (prev: AppState) => AppState) => void {
  const store = useContext(AppStateContext)!
  return store.setState.bind(store)
}

// 使用示例
function AddTaskButton() {
  const setState = useSetAppState()

  const addTask = () => {
    setState(prev => ({
      tasks: {
        ...prev.tasks,
        [uuid()]: { status: 'pending' },
      },
    }))
  }

  return <button onClick={addTask}>Add Task</button>
}
```

## 19.5 状态隔离

### 19.5.1 子 Agent 状态

子 Agent 的状态是隔离的：

```typescript
// src/query/createSubagentContext.ts
function createSubagentContext(
  parentContext: ToolUseContext,
): ToolUseContext {
  // 子 Agent 有独立的 AppState 前缀
  const subAgentState = createStore({
    ...cloneDeep(parentContext.getAppState()),
    agentId: uuid(),
  })

  return {
    ...cloneDeep(parentContext),

    // 子 Agent 状态隔离
    getAppState: () => subAgentState.getState(),
    setAppState: (updater) => {
      // 更新只影响子 Agent
      subAgentState.setState(updater)
    },
  }
}
```

### 19.5.2 异步 Agent 状态

异步 Agent 的状态更新可能丢失（因为 setAppState 是 noop）：

```typescript
// 对于异步 Agent，使用特殊处理
if (toolUseContext.setAppStateForTasks) {
  // 使用任务专用的状态更新
  toolUseContext.setAppStateForTasks(prev => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [taskId]: newTaskState,
    },
  }))
}
```

## 19.6 状态持久化

### 19.6.1 设置持久化

```typescript
// src/state/settings.ts
async function loadSettings(): Promise<SettingsJson> {
  const configPath = getConfigPath()

  if (await fileExists(configPath)) {
    const content = await readFile(configPath)
    return JSON.parse(content)
  }

  return defaultSettings
}

async function saveSettings(settings: SettingsJson): Promise<void> {
  const configPath = getConfigPath()
  await writeFile(configPath, JSON.stringify(settings, null, 2))
}
```

### 19.6.2 状态变更监听

```typescript
// 监听设置变更并持久化
appStateStore.subscribe(() => {
  const settings = appStateStore.getState().settings

  // 防抖保存
  debounce(() => {
    saveSettings(settings)
  }, 1000)()
})
```

## 19.7 状态层级

### 19.7.1 多个 Store

```typescript
// 按功能分离 store
const settingsStore = createStore(defaultSettings)
const tasksStore = createStore({})
const mcpStore = createStore({ clients: [], tools: [] })

// 或者组合成一个根 store
const rootStore = createStore({
  settings: defaultSettings,
  tasks: {},
  mcp: { clients: [], tools: [] },
})
```

### 19.7.2 模块化状态

```typescript
// src/state/modules/settings.ts
export const settingsModule = {
  store: createStore(defaultSettings),

  updateSettings: (updates: Partial<Settings>) => {
    settingsModule.store.setState(prev => ({ ...prev, ...updates }))
  },

  resetSettings: () => {
    settingsModule.store.setState(defaultSettings)
  },
}
```

## 19.8 设计模式

### 19.8.1 不可变更新

```typescript
// 始终创建新对象，不修改原对象
store.setState(prev => ({
  ...prev,
  nested: {
    ...prev.nested,
    value: newValue,
  },
}))
```

### 19.8.2 选择器模式

```typescript
// 使用选择器避免不必要的重渲染
const TaskList = () => {
  // 只订阅 tasks 变化
  const tasks = useAppState(state => state.tasks)
  const verbose = useAppState(state => state.verbose)  // 单独订阅

  return (
    <div>
      {verbose && <TaskCounter tasks={tasks} />}
      {tasks.map(task => <TaskItem key={task.id} task={task} />)}
    </div>
  )
}
```

### 19.8.3 中间件模式

```typescript
// 状态更新中间件
function withLogging<T>(store: Store<T>): Store<T> {
  return {
    ...store,
    setState: (updater) => {
      console.log('State updating...')
      const start = performance.now()
      store.setState(updater)
      console.log(`State updated in ${performance.now() - start}ms`)
    },
  }
}

function withPersistence<T>(store: Store<T>, key: string): Store<T> {
  return {
    ...store,
    setState: (updater) => {
      store.setState(updater)
      localStorage.setItem(key, JSON.stringify(store.getState()))
    },
  }
}
```

## 19.9 本章小结

状态管理是 Claude Code 架构的基础：

| 组件 | 职责 |
|------|------|
| `createStore` | 创建状态存储 |
| `Store<T>` | 状态接口 |
| `AppStateStore` | 全局状态 |
| `useAppState` | React Hook |
| `subscribe` | 变更订阅 |

**核心设计决策**：

1. **简单 pub/sub** — 不依赖外部库
2. **不可变更新** — 始终创建新对象
3. **选择器** — 避免不必要重渲染
4. **状态隔离** — 子 Agent 有独立状态

## 19.10 实践要点

1. **不可变性** — 永远不直接修改状态
2. **细粒度订阅** — 使用选择器
3. **防抖保存** — 避免频繁写入
4. **状态隔离** — 子 Agent 不能影响父状态

---

## 下一步

下一章我们将学习 **CLI 与 REPL**，了解 Claude Code 的命令行界面。
