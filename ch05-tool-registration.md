# 第 5 章：工具注册与发现 — Tool Registration

> "30+ 工具如何有序地注册到系统中？"

## 5.1 工具注册概述

Claude Code 内置了 **30+ 工具**，它们通过统一的注册机制被发现和使用。

### 5.1.1 工具分类

```
Tools
├── 核心工具
│   ├── BashTool        # Shell 执行
│   ├── FileReadTool    # 文件读取
│   ├── FileEditTool    # 文件编辑
│   └── FileWriteTool   # 文件写入
│
├── 搜索工具
│   ├── GlobTool        # 文件模式匹配
│   ├── GrepTool        # 内容搜索
│   ├── WebSearchTool   # 网络搜索
│   └── WebFetchTool    # 网页获取
│
├── 任务工具
│   ├── TaskCreateTool  # 创建任务
│   ├── TaskListTool    # 列出任务
│   ├── TaskUpdateTool  # 更新任务
│   └── TaskStopTool    # 停止任务
│
├── Agent 工具
│   ├── AgentTool       # 启动子 Agent
│   ├── TeamCreateTool  # 创建团队
│   └── SendMessageTool # 发送消息
│
├── 杂项工具
│   ├── SkillTool       # 技能调用
│   ├── ExitPlanModeTool
│   ├── EnterPlanModeTool
│   └── ...
│
└── MCP 工具 (动态)
    ├── mcp__filesystem__read_file
    ├── mcp__filesystem__write_file
    └── ...
```

## 5.2 工具注册 — getAllBaseTools

### 5.2.1 注册函数

```typescript
// src/tools.ts:193-210
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // ... 更多工具
  ]
}
```

### 5.2.2 完整的工具列表

```typescript
// src/tools.ts:193-260
export function getAllBaseTools(): Tools {
  return [
    // === 核心工具 ===
    AgentTool,
    TaskOutputTool,
    BashTool,

    // === 文件工具 ===
    FileEditTool,
    FileReadTool,
    FileWriteTool,
    GlobTool,
    NotebookEditTool,

    // === 搜索工具 ===
    GrepTool,
    WebSearchTool,
    WebFetchTool,

    // === 任务工具 ===
    TaskCreateTool,
    TaskGetTool,
    TaskListTool,
    TaskStopTool,
    TaskUpdateTool,

    // === 开发者工具 ===
    ExitPlanModeV2Tool,
    ExitPlanModeTool,
    EnterPlanModeTool,
    EnterWorktreeTool,
    ExitWorktreeTool,
    ConfigTool,

    // === 特殊工具 ===
    BriefTool,
    AskUserUserQuestionTool,
    LSPTool,
    ToolSearchTool,

    // === MCP 资源工具 ===
    ListMcpResourcesTool,
    ReadMcpResourceTool,

    // === 条件加载的工具 ===
    ...(REPLTool ? [REPLTool] : []),
    ...(SleepTool ? [SleepTool] : []),
    ...(MonitorTool ? [MonitorTool] : []),
    ...(WebBrowserTool ? [WebBrowserTool] : []),
    // ... 更多条件工具
  ]
}
```

## 5.3 条件加载 — Feature Flags

### 5.3.1 为什么需要条件加载？

1. **减少包体积** — 生产环境不需要测试工具
2. **A/B 测试** — 新功能只对部分用户开放
3. **环境差异** — CLI 和 SDK 需要不同工具

### 5.3.2 实现方式

```typescript
// src/tools.ts:14-52
// 使用 Bun 的 feature() 进行条件加载
import { feature } from 'bun:bundle'

// SleepTool - 只在特定特性开启时加载
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null

// MonitorTool - 通过 MONITOR_TOOL 特性开启
const MonitorTool = feature('MONITOR_TOOL')
  ? require('./tools/MonitorTool/MonitorTool.js').MonitorTool
  : null

// WebBrowserTool - 通过 WEB_BROWSER_TOOL 特性开启
const WebBrowserTool = feature('WEB_BROWSER_TOOL')
  ? require('./tools/WebBrowserTool/WebBrowserTool.js').WebBrowserTool
  : null

// Cron 工具 - AGENT_TRIGGERS 特性
const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      require('./tools/ScheduleCronTool/CronDeleteTool.js').CronDeleteTool,
      require('./tools/ScheduleCronTool/CronListTool.js').CronListTool,
    ]
  : []
```

### 5.3.3 Dead Code Elimination

Bun 的 `feature()` 实现是 **编译时** 条件：

```typescript
// 当特性关闭时，整个模块被剔除
if (feature('SOME_FEATURE')) {
  // 这段代码在最终产物中完全不存在
}
```

**好处**：
- 零运行时开销
- 减小产物大小
- 支持多版本共存

## 5.4 工具预设 — Tool Presets

### 5.4.1 预设定义

```typescript
// src/tools.ts:161-171
export const TOOL_PRESETS = ['default'] as const
export type ToolPreset = (typeof TOOL_PRESETS)[number]

export function parseToolPreset(preset: string): ToolPreset | null {
  const presetString = preset.toLowerCase()
  if (!TOOL_PRESETS.includes(presetString as ToolPreset)) {
    return null
  }
  return presetString as ToolPreset
}
```

### 5.4.2 获取预设工具

```typescript
// src/tools.ts:179-183
export function getToolsForDefaultPreset(): string[] {
  const tools = getAllBaseTools()
  const isEnabled = tools.map(tool => tool.isEnabled())
  return tools.filter((_, i) => isEnabled[i]).map(tool => tool.name)
}
```

## 5.5 工具过滤

### 5.5.1 按来源过滤

```typescript
// src/constants/tools.ts
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  // Agent 不能使用的工具
  'Agent',      // 防止嵌套 Agent
  'TeamCreate', // 团队管理由协调者负责
  // ...
])

export const CUSTOM_AGENT_DISALLOWED_TOOLS = new Set([
  // 自定义 Agent 额外禁止的工具
])
```

### 5.5.2 按权限过滤

```typescript
// src/utils/permissions/permissions.ts
export function filterToolsByPermission(
  tools: Tools,
  permissionContext: ToolPermissionContext,
): Tools {
  return tools.filter(tool => {
    // 检查是否总是被拒绝
    if (permissionContext.alwaysDenyRules[tool.name]) {
      return false
    }

    // 检查是否总是允许
    if (permissionContext.alwaysAllowRules[tool.name]) {
      return true
    }

    // 其他工具需要权限确认
    return true
  })
}
```

### 5.5.3 协调者模式过滤

```typescript
// src/constants/tools.ts
export const COORDINATOR_MODE_ALLOWED_TOOLS = new Set([
  'Agent',
  'SendMessage',
  'TaskStop',
  'TaskList',
  'Bash',
  'Read',
  'Edit',
  // ... Worker 可用的工具
])
```

## 5.6 工具发现

### 5.6.1 按名称查找

```typescript
// src/Tool.ts:358-360
export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}

// src/Tool.ts:348-353
export function toolMatchesName(
  tool: { name: string; aliases?: string[] },
  name: string,
): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}
```

### 5.6.2 工具搜索 (ToolSearch)

```typescript
// src/tools/ToolSearchTool/ToolSearchTool.ts
// 当工具数量太多时，LLM 可以先搜索工具

export const ToolSearchTool = buildTool({
  name: 'ToolSearch',

  inputSchema: z.object({
    query: z.string()
      .describe('Search query for tools'),
  }),

  async call(input, context) {
    // 搜索匹配的工具
    const results = searchTools(input.query, context.options.tools)
    return {
      data: {
        tools: results.map(t => ({
          name: t.name,
          description: t.description,
        })),
      },
    }
  },
})
```

### 5.6.3 延迟加载 (Deferred Tools)

```typescript
// src/Tool.ts:442
/**
 * 当 true 时，此工具通过 defer_loading 发送，
 * 需要先通过 ToolSearch 找到后才能调用。
 */
readonly shouldDefer?: boolean

// 例如，某些大型或不常用的工具
const HeavyTool = buildTool({
  name: 'HeavyTool',
  shouldDefer: true,  // 延迟加载
  // ...
})
```

## 5.7 MCP 工具集成

### 5.7.1 动态注册

MCP 服务器提供的工具是**动态注册**的：

```typescript
// src/services/mcp/client.ts
export class MCPClient {
  async loadTools(): Promise<Tool[]> {
    const tools = await this.server.listTools()
    return tools.map(tool => this.wrapMcpTool(tool))
  }

  wrapMcpTool(mcpTool: MCPTool): Tool {
    return buildTool({
      name: buildMcpToolName(this.serverName, mcpTool.name),
      inputSchema: mcpTool.inputSchema,
      isMcp: true,

      async call(input, context) {
        const result = await this.server.callTool(mcpTool.name, input)
        return { data: result }
      },
    })
  }
}
```

### 5.7.2 工具名称

```typescript
// src/tools/MCPTool/utils.ts
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `mcp__${serverName}__${toolName}`
}

// 示例
// mcp__filesystem__read_file
// mcp__github__create_issue
```

## 5.8 工具刷新

### 5.8.1 动态刷新

某些工具可能在运行时发生变化：

```typescript
// src/query.ts:1659-1671
// 刷新工具列表，以便新的 MCP 服务器可用
if (updatedToolUseContext.options.refreshTools) {
  const refreshedTools = updatedToolUseContext.options.refreshTools()
  if (refreshedTools !== updatedToolUseContext.options.tools) {
    updatedToolUseContext = {
      ...updatedToolUseContext,
      options: {
        ...updatedToolUseContext.options,
        tools: refreshedTools,
      },
    }
  }
}
```

### 5.8.2 刷新回调

```typescript
// src/Tool.ts:177-178
/**
 * 可选的回调，获取最新工具列表
 * 例如：MCP 服务器连接后的动态工具
 */
refreshTools?: () => Tools
```

## 5.9 设计模式

### 5.9.1 插件式注册

```typescript
// 工具注册表模式
class ToolRegistry {
  private tools: Map<string, Tool> = new Map()

  register(tool: Tool): void {
    this.tools.set(tool.name, tool)
  }

  get(name: string): Tool | undefined {
    return this.tools.get(name)
  }

  getAll(): Tools {
    return Array.from(this.tools.values())
  }
}

// 使用
const registry = new ToolRegistry()
registry.register(BashTool)
registry.register(FileReadTool)
// ...
```

### 5.9.2 中间件模式

```typescript
// 工具调用包装
function withLogging<T>(tool: Tool<T>): Tool<T> {
  return {
    ...tool,
    async call(args, context, canUseTool, parentMessage, onProgress) {
      console.log(`Calling tool: ${tool.name}`)
      const result = await tool.call(args, context, canUseTool, parentMessage, onProgress)
      console.log(`Tool ${tool.name} returned`)
      return result
    },
  }
}

// 权限检查包装
function withPermission<T>(tool: Tool<T>, rules: PermissionRules): Tool<T> {
  return {
    ...tool,
    async call(args, context, canUseTool, parentMessage, onProgress) {
      const hasPermission = await checkPermission(tool.name, args, rules)
      if (!hasPermission) {
        const canUse = await canUseTool({ toolName: tool.name, toolInput: args })
        if (!canUse) {
          throw new Error('Permission denied')
        }
      }
      return tool.call(args, context, canUseTool, parentMessage, onProgress)
    },
  }
}
```

## 5.10 本章小结

工具注册是 Claude Code 工具系统的组织机制：

| 机制 | 实现 |
|------|------|
| 统一注册 | `getAllBaseTools()` |
| 条件加载 | `feature()` 编译时开关 |
| 按权限过滤 | `filterToolsByPermission()` |
| 名称匹配 | `toolMatchesName()` 支持别名 |
| MCP 动态工具 | `wrapMcpTool()` 包装 |

**关键设计决策**：

1. **集中注册** — 所有工具在 `getAllBaseTools()` 列出
2. **条件编译** — 使用 `feature()` 减少产物
3. **延迟加载** — 大型工具可以标记 `shouldDefer`
4. **动态刷新** — 支持 MCP 等动态工具

## 5.11 实践要点

1. **添加新工具** — 在 `getAllBaseTools()` 添加
2. **条件加载** — 使用 `feature('FEATURE_NAME')`
3. **别名支持** — 通过 `aliases` 字段添加
4. **权限规则** — 在 `alwaysAllowRules` 等配置

---

## 下一步

下一章我们将学习 **工具执行引擎**，了解：
- 工具如何并行执行
- 读写工具的分离策略
- 流式工具执行器
