# 第 15 章：插件系统 — Plugin System

> "插件让 Claude Code 可以无限扩展。"

## 15.1 插件系统概述

插件系统允许开发者扩展 Claude Code 的功能：

```
┌─────────────────────────────────────────────────────────┐
│                   Claude Code Core                      │
│  ┌─────────────────────────────────────────────────┐  │
│  │              Plugin Manager                       │  │
│  │  - Load/Unload plugins                         │  │
│  │  - Plugin lifecycle                            │  │
│  │  - Sandboxing                                  │  │
│  └─────────────────────────────────────────────────┘  │
│         │                    │                    │
└─────────┼────────────────────┼────────────────────┘
          │                    │
          ▼                    ▼
┌─────────────────┐    ┌─────────────────┐
│    Plugin A     │    │    Plugin B     │
│  - Custom Tools │    │  - New Commands │
│  - Event Hooks  │    │  - UI Extension │
└─────────────────┘    └─────────────────┘
```

### 15.1.2 与 MCP 的区别

| 方面 | MCP | 插件 |
|------|-----|------|
| 协议 | 标准化的 MCP 协议 | 自定义 API |
| 部署 | 独立进程 | 打包在 Claude Code 中 |
| 复杂度 | 适合简单工具 | 适合复杂功能 |
| 工具注册 | 自动发现 | 手动注册 |

## 15.2 插件结构

### 15.2.1 插件定义

```typescript
// src/plugins/types.ts
interface Plugin {
  id: string
  name: string
  version: string
  description: string
  author: string

  // 生命周期
  onLoad?: () => Promise<void>
  onUnload?: () => Promise<void>

  // 扩展点
  tools?: Tool[]          // 注册工具
  commands?: Command[]    // 注册命令
  hooks?: Hook[]          // 注册钩子
  providers?: Provider[]  // 注册数据提供者
}
```

### 15.2.2 插件清单

```typescript
// plugin.json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "main": "dist/index.js",
  "manifest": {
    "tools": ["./tools"],
    "commands": ["./commands"],
    "hooks": ["./hooks"]
  }
}
```

## 15.3 插件加载

### 15.3.1 加载管理器

```typescript
// src/plugins/PluginManager.ts
export class PluginManager {
  private plugins: Map<string, Plugin> = new Map()
  private enabledPlugins: Set<string> = new Set()

  async loadPlugin(pluginPath: string): Promise<void> {
    // 1. 读取插件清单
    const manifest = await readPluginManifest(pluginPath)

    // 2. 加载插件代码
    const plugin = await import(pluginPath)

    // 3. 验证插件接口
    if (!this.validatePlugin(plugin)) {
      throw new Error(`Invalid plugin: ${manifest.name}`)
    }

    // 4. 注册插件
    this.plugins.set(manifest.id, plugin)

    // 5. 调用 onLoad
    if (plugin.onLoad) {
      await plugin.onLoad()
    }

    // 6. 如果启用，注册扩展
    if (this.enabledPlugins.has(manifest.id)) {
      await this.enablePlugin(manifest.id)
    }
  }

  async unloadPlugin(pluginId: string): Promise<void> {
    const plugin = this.plugins.get(pluginId)
    if (!plugin) return

    // 1. 禁用插件
    await this.disablePlugin(pluginId)

    // 2. 调用 onUnload
    if (plugin.onUnload) {
      await plugin.onUnload()
    }

    // 3. 移除插件
    this.plugins.delete(pluginId)
  }
}
```

### 15.3.2 启用插件

```typescript
// src/plugins/PluginManager.ts
async enablePlugin(pluginId: string): Promise<void> {
  const plugin = this.plugins.get(pluginId)
  if (!plugin) return

  // 注册工具
  if (plugin.tools) {
    for (const tool of plugin.tools) {
      await toolRegistry.register(tool)
    }
  }

  // 注册命令
  if (plugin.commands) {
    for (const command of plugin.commands) {
      await commandRegistry.register(command)
    }
  }

  // 注册钩子
  if (plugin.hooks) {
    for (const hook of plugin.hooks) {
      await hookRegistry.register(hook)
    }
  }

  this.enabledPlugins.add(pluginId)
}
```

## 15.4 插件命令

### 15.4.1 命令注册

```typescript
// src/plugins/commands.ts
interface PluginCommand {
  name: string
  description: string
  handler: (args: string[]) => Promise<void>
  aliases?: string[]
}

// 注册命令
export function registerCommand(command: PluginCommand): void {
  commands.push(command)
}

// 执行命令
export async function executeCommand(
  name: string,
  args: string[],
): Promise<void> {
  const command = commands.find(
    c => c.name === name || c.aliases?.includes(name)
  )

  if (!command) {
    throw new Error(`Command not found: ${name}`)
  }

  await command.handler(args)
}
```

### 15.4.2 斜杠命令

```typescript
// 插件可以注册 /command
const myCommand: PluginCommand = {
  name: 'my-plugin',
  description: 'Run my plugin command',
  aliases: ['mp'],
  handler: async (args) => {
    // 处理命令
    console.log('My plugin command:', args)
  },
}
```

## 15.5 插件钩子

### 15.5.1 钩子类型

```typescript
// src/plugins/hooks.ts
type HookType =
  | 'pre_tool_call'      // 工具调用前
  | 'post_tool_call'     // 工具调用后
  | 'pre_message'        // 消息处理前
  | 'post_message'       // 消息处理后
  | 'on_error'           // 错误发生时
  | 'on_compact'         // 上下文压缩时

interface Hook {
  type: HookType
  handler: HookHandler
  priority?: number  // 执行优先级
}

type HookHandler = (
  context: HookContext,
  next: () => Promise<void>,
) => Promise<void>
```

### 15.5.2 钩子实现

```typescript
// 工具调用前钩子
const preToolCallHook: Hook = {
  type: 'pre_tool_call',
  priority: 10,  // 较低优先级，先执行
  handler: async (context, next) => {
    console.log(`Calling tool: ${context.toolName}`)

    // 可以修改参数
    if (context.toolName === 'Bash') {
      context.toolInput.command = `set -e\n${context.toolInput.command}`
    }

    // 或者阻止执行
    if (context.toolName === 'DeleteEverything') {
      throw new Error('Not allowed!')
    }

    // 继续执行
    await next()
  },
}
```

## 15.6 插件状态

### 15.6.1 AppState 中的插件状态

```typescript
// src/state/AppState.tsx
interface AppState {
  // ...
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: {
      installing: string[]
      updating: string[]
      failed: Map<string, string>
    }
  }
}
```

### 15.6.2 插件错误追踪

```typescript
// src/plugins/errorTracking.ts
interface PluginError {
  pluginId: string
  pluginVersion: string
  error: string
  stack?: string
  timestamp: number
  context?: Record<string, unknown>
}

export function recordPluginError(error: PluginError): void {
  // 添加到 AppState
  appState.setState(state => ({
    plugins: {
      ...state.plugins,
      errors: [...state.plugins.errors, error],
    },
  }))

  // 发送到遥测
  logEvent('plugin_error', error)
}
```

## 15.7 插件安装

### 15.7.1 安装流程

```typescript
// src/plugins/installer.ts
async function installPlugin(source: string): Promise<void> {
  // 1. 解析来源
  const parsed = parsePluginSource(source)

  // 2. 下载插件
  const pluginPath = await downloadPlugin(parsed)

  // 3. 验证插件
  const manifest = await readPluginManifest(pluginPath)
  if (!validateManifest(manifest)) {
    throw new Error('Invalid plugin manifest')
  }

  // 4. 安装依赖
  if (manifest.dependencies) {
    await installDependencies(pluginPath, manifest.dependencies)
  }

  // 5. 加载插件
  await pluginManager.loadPlugin(pluginPath)

  // 6. 启用插件
  await pluginManager.enablePlugin(manifest.id)
}
```

### 15.7.2 安装来源

```typescript
// 支持的来源类型
type PluginSource =
  | { type: 'npm', name: string, version?: string }      // npm 包
  | { type: 'git', url: string, ref?: string }            // Git 仓库
  | { type: 'local', path: string }                       // 本地路径
  | { type: 'url', url: string }                          // 直接 URL
```

## 15.8 插件沙箱

### 15.8.1 隔离机制

```typescript
// src/plugins/sandbox.ts
function createSandbox(plugin: Plugin): Sandbox {
  return {
    // 限制可访问的模块
    allowedModules: [
      'fs',
      'path',
      'crypto',
      // 只允许特定的 Node.js 模块
    ],

    // 限制可执行的操作
    allowedOperations: [
      'file:read',
      'file:write',
      'http:fetch',
      // ...
    ],

    // 超时限制
    timeout: 30000,  // 30 秒

    // 内存限制
    maxMemory: '100MB',
  }
}
```

## 15.9 设计模式

### 15.9.1 插件模板

```typescript
// 插件开发模板
export function createPlugin(config: PluginConfig): Plugin {
  return {
    id: config.id,
    name: config.name,
    version: config.version,

    async onLoad() {
      // 初始化
    },

    async onUnload() {
      // 清理
    },

    tools: [
      // 注册工具
    ],

    commands: [
      // 注册命令
    ],

    hooks: [
      // 注册钩子
    ],
  }
}
```

## 15.10 本章小结

插件系统提供了强大的扩展能力：

| 组件 | 职责 |
|------|------|
| `PluginManager` | 插件加载/卸载 |
| `PluginLoader` | 代码动态加载 |
| `HookRegistry` | 钩子注册 |
| `CommandRegistry` | 命令注册 |
| `Sandbox` | 安全隔离 |

**核心设计决策**：

1. **生命周期** — onLoad/onUnload 钩子
2. **扩展点** — 工具、命令、钩子
3. **沙箱隔离** — 限制危险操作
4. **状态追踪** — 错误和安装状态

## 15.11 实践要点

1. **遵循接口** — 确保插件符合 Plugin 接口
2. **错误处理** — 插件代码应该有完善的错误处理
3. **清理资源** — onUnload 中清理所有资源
4. **沙箱限制** — 不要依赖沙箱未限制的操作

---

## 下一步

下一章我们将学习 **Skills 系统**，了解可组合的技能机制。
