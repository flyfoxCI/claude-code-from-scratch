# 第 18 章：权限与安全 — Permissions

> "工具可以改变世界，所以需要权限控制。"

## 18.1 权限系统概述

Claude Code 的权限系统确保工具执行是安全的：

```
┌─────────────────────────────────────────────────────────┐
│                     Permission System                     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐  │
│  │              ToolPermissionContext               │  │
│  │  - mode: 'default' | 'plan' | 'bypass'         │  │
│  │  - alwaysAllowRules: {...}                       │  │
│  │  - alwaysDenyRules: {...}                        │  │
│  │  - alwaysAskRules: {...}                         │  │
│  └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Permission Check                        │
│                                                         │
│  alwaysAllow? ──► YES ──► Execute                       │
│       │                                                    │
│       ▼                                                   │
│  alwaysDeny? ──► YES ──► Deny                           │
│       │                                                    │
│       ▼                                                   │
│  alwaysAsk? ──► YES ──► Request User                      │
│       │                                                    │
│       ▼                                                   │
│  Default Mode ──► Request if needed                       │
└─────────────────────────────────────────────────────────┘
```

## 18.2 权限上下文

### 18.2.1 类型定义

```typescript
// src/types/permissions.ts
export type PermissionMode =
  | 'default'    // 标准权限模式
  | 'plan'      // Plan 模式（只读）
  | 'bypass'    // 完全绕过

export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode

  // 规则
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource

  // 能力
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean

  // 特殊模式
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

### 18.2.2 创建空上下文

```typescript
// src/Tool.ts:140-148
export const getEmptyToolPermissionContext: () => ToolPermissionContext =
  () => ({
    mode: 'default',
    additionalWorkingDirectories: new Map(),
    alwaysAllowRules: {},
    alwaysDenyRules: {},
    alwaysAskRules: {},
    isBypassPermissionsModeAvailable: false,
  })
```

## 18.3 权限规则

### 18.3.1 规则类型

```typescript
// src/types/permissions.ts
export type ToolPermissionRulesBySource = {
  [source: string]: ToolPermissionRule
}

export interface ToolPermissionRule {
  // 允许的条件
  allow?: {
    commands?: string[]      // bash 命令前缀
    paths?: string[]         // 文件路径模式
    tools?: string[]         // 工具名称
  }

  // 拒绝的条件
  deny?: {
    commands?: string[]
    paths?: string[]
    tools?: string[]
  }

  // 认证信息
  auth?: {
    type: 'api_key' | 'oauth' | 'basic'
    config: Record<string, string>
  }
}
```

### 18.3.2 规则示例

```typescript
// 用户配置
{
  "permissions": {
    "alwaysAllow": {
      "Read": {
        "paths": ["${cwd}/src/**", "${cwd}/tests/**"]
      },
      "Bash": {
        "commands": ["git status", "git log", "npm test", "npm run *"]
      }
    },
    "alwaysDeny": {
      "Bash": {
        "commands": ["rm -rf /*", "dd if=*"]
      }
    }
  }
}
```

## 18.4 权限检查

### 18.4.1 检查函数

```typescript
// src/utils/permissions/permissions.ts
export function checkToolPermission(
  toolName: string,
  toolInput: unknown,
  context: ToolPermissionContext,
): PermissionResult {
  const toolRules = context.alwaysAllowRules[toolName]

  // 1. 检查 alwaysAllow
  if (toolRules?.allow) {
    if (matchesRule(toolInput, toolRules.allow)) {
      return { allowed: true, reason: 'always_allow' }
    }
  }

  // 2. 检查 alwaysDeny
  const denyRules = context.alwaysDenyRules[toolName]
  if (denyRules?.deny) {
    if (matchesRule(toolInput, denyRules.deny)) {
      return { allowed: false, reason: 'always_deny' }
    }
  }

  // 3. 检查 alwaysAsk
  const askRules = context.alwaysAskRules[toolName]
  if (askRules?.allow) {
    if (matchesRule(toolInput, askRules.allow)) {
      return { allowed: false, reason: 'always_ask', requiresAuth: true }
    }
  }

  // 4. 根据模式决定
  if (context.mode === 'bypass') {
    return { allowed: true, reason: 'bypass_mode' }
  }

  if (context.mode === 'plan') {
    // Plan 模式下只允许读操作
    return { allowed: false, reason: 'plan_mode_readonly' }
  }

  return { allowed: false, reason: 'needs_confirmation' }
}
```

### 18.4.2 规则匹配

```typescript
// src/utils/permissions/matching.ts
function matchesRule(
  input: unknown,
  rule: { commands?: string[]; paths?: string[] },
): boolean {
  if (typeof input === 'object' && input !== null) {
    // 检查命令
    if ('command' in input && rule.commands) {
      const command = String(input.command)
      for (const pattern of rule.commands) {
        if (matchesPattern(command, pattern)) {
          return true
        }
      }
    }

    // 检查路径
    if ('path' in input && rule.paths) {
      const path = String(input.path)
      for (const pattern of rule.paths) {
        if (matchesPattern(path, expandVariables(pattern))) {
          return true
        }
      }
    }
  }

  return false
}

function matchesPattern(str: string, pattern: string): boolean {
  // 支持通配符
  if (pattern === '*') return true
  if (pattern.endsWith('/*')) {
    const prefix = pattern.slice(0, -2)
    return str.startsWith(prefix)
  }
  if (pattern.includes('*')) {
    const regex = new RegExp('^' + pattern.replace(/\*/g, '.*') + '$')
    return regex.test(str)
  }
  return str === pattern
}
```

## 18.5 用户确认

### 18.5.1 确认请求

```typescript
// src/hooks/useCanUseTool.ts
export type CanUseToolFn = (params: {
  toolName: string
  toolInput: unknown
}) => Promise<{ use: boolean; reason?: string }>

// 实现
async function canUseTool(params: { toolName: string; toolInput: unknown }):
  Promise<{ use: boolean; reason?: string }> {
  const result = checkToolPermission(params.toolName, params.toolInput, context)

  if (result.allowed) {
    return { use: true, reason: result.reason }
  }

  if (result.requiresAuth) {
    // 需要认证
    return { use: false, reason: 'Authentication required' }
  }

  // 请求用户确认
  return showPermissionDialog(params.toolName, params.toolInput)
}
```

### 18.5.2 确认对话框

```typescript
// 显示确认对话框
async function showPermissionDialog(
  toolName: string,
  toolInput: unknown,
): Promise<{ use: boolean }> {
  const tool = findToolByName(tools, toolName)

  return new Promise((resolve) => {
    // 创建对话框
    const dialog = createDialog({
      title: `Run ${toolName}?`,
      content: renderToolConfirmation(tool, toolInput),
      buttons: [
        { label: 'Allow', value: true, primary: true },
        { label: 'Deny', value: false },
        { label: 'Always Allow', value: 'always' },
        { label: 'Always Deny', value: 'never' },
      ],
    })

    dialog.on('result', (result) => {
      if (result === 'always') {
        addAlwaysAllowRule(toolName, toolInput)
        resolve({ use: true })
      } else if (result === 'never') {
        addAlwaysDenyRule(toolName, toolInput)
        resolve({ use: false })
      } else {
        resolve({ use: result })
      }
    })
  })
}
```

## 18.6 权限模式

### 18.6.1 模式切换

```typescript
// src/state/AppState.tsx
interface AppState {
  toolPermissionContext: ToolPermissionContext
}

// 切换到 bypass 模式
function enableBypassMode(): void {
  appState.setState(state => ({
    toolPermissionContext: {
      ...state.toolPermissionContext,
      mode: 'bypass',
    },
  }))
}

// 切换到 plan 模式
function enablePlanMode(): void {
  appState.setState(state => ({
    toolPermissionContext: {
      ...state.toolPermissionContext,
      prePlanMode: state.toolPermissionContext.mode,
      mode: 'plan',
    },
  }))
}

// 退出 plan 模式
function disablePlanMode(): void {
  appState.setState(state => ({
    toolPermissionContext: {
      ...state.toolPermissionContext,
      mode: state.toolPermissionContext.prePlanMode || 'default',
      prePlanMode: undefined,
    },
  }))
}
```

## 18.7 拒绝追踪

### 18.7.1 追踪状态

```typescript
// src/utils/permissions/denialTracking.ts
interface DenialTrackingState {
  denials: Map<string, number>  // toolName -> denial count
  lastDenialAt: Map<string, number>  // toolName -> timestamp
}

const DENIAL_THRESHOLD = 3  // 超过 3 次拒绝后降低权限
const DENIAL_WINDOW_MS = 5 * 60 * 1000  // 5 分钟窗口
```

### 18.7.2 追踪更新

```typescript
// 当工具被拒绝时
function recordDenial(toolName: string): void {
  const now = Date.now()
  const state = appState.localDenialTracking || { denials: new Map(), lastDenialAt: new Map() }

  const count = (state.denials.get(toolName) || 0) + 1
  state.denials.set(toolName, count)
  state.lastDenialAt.set(toolName, now)

  // 检查是否超过阈值
  if (count >= DENIAL_THRESHOLD) {
    // 降低该工具的权限
    downgradeToolPermission(toolName)
  }
}
```

## 18.8 工作目录权限

### 18.8.1 额外工作目录

```typescript
// src/types/permissions.ts
interface AdditionalWorkingDirectory {
  path: string
  readOnly: boolean
  requireConfirmation: boolean
}

// 添加额外工作目录
function addAdditionalWorkingDirectory(
  path: string,
  options: { readOnly?: boolean; requireConfirmation?: boolean } = {},
): void {
  appState.setState(state => ({
    toolPermissionContext: {
      ...state.toolPermissionContext,
      additionalWorkingDirectories: new Map(
        state.toolPermissionContext.additionalWorkingDirectories
      ).set(path, { path, ...options }),
    },
  }))
}
```

## 18.9 设计模式

### 18.9.1 权限中间件

```typescript
// 工具调用权限包装
function withPermission<T>(tool: Tool<T>): Tool<T> {
  return {
    ...tool,
    async call(input, context, canUseTool, parentMessage) {
      // 检查权限
      const result = await canUseTool({ toolName: tool.name, toolInput: input })

      if (!result.use) {
        return {
          data: { error: result.reason || 'Permission denied' },
        }
      }

      // 执行工具
      return tool.call(input, context, canUseTool, parentMessage)
    },
  }
}
```

### 18.9.2 规则引擎

```typescript
// 简单的规则引擎
class PermissionEngine {
  private rules: PermissionRule[] = []

  addRule(rule: PermissionRule): void {
    this.rules.push(rule)
  }

  evaluate(
    toolName: string,
    input: unknown,
    context: ToolPermissionContext,
  ): PermissionResult {
    // 按优先级评估规则
    for (const rule of this.rules.sort((a, b) => b.priority - a.priority)) {
      const result = rule.evaluate(toolName, input, context)
      if (result.match) {
        return result
      }
    }

    // 默认拒绝
    return { allowed: false, reason: 'no_rule_matched' }
  }
}
```

## 18.10 本章小结

权限系统是 Agent 安全执行的基础：

| 组件 | 职责 |
|------|------|
| `ToolPermissionContext` | 权限配置 |
| `checkToolPermission` | 权限检查 |
| `showPermissionDialog` | 用户确认 |
| `denialTracking` | 拒绝追踪 |
| `PermissionMode` | 权限模式切换 |

**核心设计决策**：

1. **规则匹配** — 支持通配符和模式
2. **多层检查** — alwaysAllow → alwaysDeny → alwaysAsk → default
3. **拒绝追踪** — 多次拒绝后降低权限
4. **模式切换** — 支持 bypass、plan 等模式

## 18.11 实践要点

1. **最小权限** — 默认拒绝，按需开放
2. **路径模式** — 使用通配符简化规则
3. **追踪拒绝** — 识别用户不信任的工具
4. **模式选择** — 根据任务选择合适的权限模式

---

## 下一步

下一章我们将学习 **状态管理架构**，了解 Claude Code 如何管理应用状态。
