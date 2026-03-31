# 第 16 章：Skills 系统 — Skills

> "Skills 是可复用的 Agent 行为单元。"

## 16.1 Skills 概述

Skills 是预定义的工作流程，可以被 Agent 调用：

```
┌─────────────────────────────────────────────────────────┐
│                     Agent                               │
│  需要执行一个复杂的 git commit 工作流                      │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Skill Tool                            │
│  skill: "commit"                                        │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   /commit Skill                         │
│  1. Run git status                                     │
│  2. Run git diff                                       │
│  3. Generate commit message                            │
│  4. Confirm with user                                  │
│  5. Execute git commit                                │
└─────────────────────────────────────────────────────────┘
```

### 16.1.1 Skills vs 工具

| 方面 | 工具 | Skills |
|------|------|--------|
| 复杂度 | 单一操作 | 多步骤工作流 |
| 用户交互 | 一般无 | 可以有确认步骤 |
| 执行方式 | 同步 | 可能异步 |
| 可组合性 | 有限 | 可以组合多个工具 |

## 16.2 Skill 定义

### 16.2.1 Skill 结构

```typescript
// src/skills/types.ts
interface Skill {
  id: string
  name: string
  description: string
  category: SkillCategory
  triggers: string[]  // 触发词

  // 执行
  execute: (context: SkillContext) => Promise<SkillResult>

  // 元数据
  version: string
  author?: string
  examples?: string[]
}

type SkillCategory =
  | 'git'
  | 'code'
  | 'debug'
  | 'test'
  | 'deploy'
  | 'custom'
```

### 16.2.2 Skill 上下文

```typescript
// src/skills/types.ts
interface SkillContext {
  agentContext: ToolUseContext
  input: string
  attachments: Attachment[]
  signal: AbortSignal
  progress: (message: string) => void
}
```

## 16.3 Skill 注册

### 16.3.1 注册表

```typescript
// src/skills/SkillRegistry.ts
class SkillRegistry {
  private skills: Map<string, Skill> = new Map()

  register(skill: Skill): void {
    // 注册主名称
    this.skills.set(skill.name, skill)

    // 注册触发词
    for (const trigger of skill.triggers) {
      this.skills.set(trigger, skill)
    }
  }

  get(nameOrTrigger: string): Skill | undefined {
    return this.skills.get(nameOrTrigger)
  }

  getByCategory(category: SkillCategory): Skill[] {
    return Array.from(this.skills.values())
      .filter(s => s.category === category)
  }

  list(): Skill[] {
    // 返回去重后的技能列表
    const seen = new Set<string>()
    return Array.from(this.skills.values())
      .filter(s => {
        if (seen.has(s.id)) return false
        seen.add(s.id)
        return true
      })
  }
}
```

### 16.3.2 目录扫描

```typescript
// src/skills/loader.ts
async function loadSkillsFromDirectory(dir: string): Promise<Skill[]> {
  const skills: Skill[] = []
  const entries = await readdir(dir)

  for (const entry of entries) {
    const path = join(dir, entry)

    if ((await stat(path)).isDirectory()) {
      // 检查是否是 skill 目录
      const skillFile = join(path, 'skill.ts')
      if (await exists(skillFile)) {
        const skill = await import(skillFile)
        skills.push(skill.default)
      }
    }
  }

  return skills
}
```

## 16.4 Skill 执行

### 16.4.1 SkillTool

```typescript
// src/tools/SkillTool/SkillTool.ts
export const SkillTool = buildTool({
  name: 'Skill',

  inputSchema: z.object({
    name: z.string()
      .describe('Name of the skill to execute'),
    args: z.record(z.string()).optional()
      .describe('Arguments to pass to the skill'),
  }),

  async call(input, context, canUseTool) {
    // 1. 查找 skill
    const skill = skillRegistry.get(input.name)
    if (!skill) {
      return { data: { error: `Skill not found: ${input.name}` } }
    }

    // 2. 创建执行上下文
    const skillContext: SkillContext = {
      agentContext: context,
      input: input.args?.input || '',
      attachments: [],
      signal: context.abortController.signal,
      progress: (message) => {
        // 报告进度
      },
    }

    // 3. 执行 skill
    try {
      const result = await skill.execute(skillContext)

      return {
        data: {
          success: true,
          result: result.output,
          logs: result.logs,
        },
      }
    } catch (error) {
      return {
        data: {
          success: false,
          error: error.message,
        },
      }
    }
  },
})
```

### 16.4.2 执行结果

```typescript
// src/skills/types.ts
interface SkillResult {
  output: string
  logs: SkillLog[]
  artifacts?: Artifact[]
  error?: string
}

interface SkillLog {
  timestamp: number
  level: 'info' | 'warn' | 'error'
  message: string
}
```

## 16.5 内置 Skills

### 16.5.1 Commit Skill

```typescript
// src/skills/commit/skill.ts
export const commitSkill: Skill = {
  id: 'commit',
  name: 'commit',
  description: 'Create a git commit with a good commit message',
  category: 'git',
  triggers: ['commit', '/commit', 'git commit'],

  async execute(context) {
    const logs: SkillLog[] = []

    // 1. 检查 git status
    logs.push({ timestamp: Date.now(), level: 'info', message: 'Checking git status...' })
    const status = await runTool('Bash', { command: 'git status --short' })

    if (!status.stdout.trim()) {
      return { output: 'No changes to commit', logs }
    }

    // 2. 查看 diff
    logs.push({ timestamp: Date.now(), level: 'info', message: 'Viewing changes...' })
    const diff = await runTool('Bash', { command: 'git diff --staged' })

    // 3. 生成 commit message
    logs.push({ timestamp: Date.now(), level: 'info', message: 'Generating commit message...' })
    const commitMessage = await generateCommitMessage(diff)

    // 4. 请求用户确认
    const confirmed = await context.progress(`Proposed commit message:\n\n${commitMessage}\n\nConfirm? (y/n)`)

    if (!confirmed) {
      return { output: 'Commit cancelled', logs }
    }

    // 5. 执行 commit
    logs.push({ timestamp: Date.now(), level: 'info', message: 'Creating commit...' })
    await runTool('Bash', { command: `git commit -m "${commitMessage}"` })

    return { output: 'Commit created successfully', logs }
  },
}
```

### 16.5.2 Verify Skill

```typescript
// src/skills/verify/skill.ts
export const verifySkill: Skill = {
  id: 'verify',
  name: 'verify',
  description: 'Verify code changes are correct',
  category: 'code',
  triggers: ['verify', '/verify'],

  async execute(context) {
    const logs: SkillLog[] = []

    // 1. 运行测试
    logs.push({ timestamp: Date.now(), level: 'info', message: 'Running tests...' })
    const testResult = await runTool('Bash', { command: 'npm test' })

    // 2. 检查 lint
    logs.push({ timestamp: Date.now(), level: 'info', message: 'Running linter...' })
    const lintResult = await runTool('Bash', { command: 'npm run lint' })

    // 3. 汇总结果
    const success = testResult.exitCode === 0 && lintResult.exitCode === 0

    return {
      output: success ? 'All checks passed' : 'Some checks failed',
      logs,
      artifacts: [
        { type: 'test', content: testResult.stdout },
        { type: 'lint', content: lintResult.stdout },
      ],
    }
  },
}
```

## 16.6 Skill 搜索

### 16.6.1 发现机制

```typescript
// src/services/skillSearch/discovery.ts
export async function discoverSkills(
  messages: Message[],
  context: ToolUseContext,
): Promise<DiscoveredSkill[]> {
  // 1. 分析最近的消息
  const recentContent = extractRecentContent(messages)

  // 2. 搜索匹配的 skills
  const matches = await searchSkills(recentContent)

  // 3. 排序和过滤
  return matches
    .filter(m => m.score > THRESHOLD)
    .sort((a, b) => b.score - a.score)
}
```

### 16.6.2 预取

```typescript
// src/services/skillSearch/prefetch.ts
export function startSkillDiscoveryPrefetch(
  null,  // 第一个参数未使用
  messages: Message[],
  context: ToolUseContext,
): PendingPrefetch {
  // 在后台运行发现，不阻塞主流程
  const promise = discoverSkills(messages, context)

  return {
    promise,
    settledAt: null,  // 尚未 settle
  }
}
```

## 16.7 Skill 组合

### 16.7.1 顺序执行

```typescript
// 执行多个 skills
async function executeSkillsSequentially(
  skills: Skill[],
  context: SkillContext,
): Promise<SkillResult[]> {
  const results: SkillResult[] = []

  for (const skill of skills) {
    const result = await skill.execute(context)
    results.push(result)

    // 如果失败，停止执行
    if (!result.success) {
      break
    }
  }

  return results
}
```

### 16.7.2 条件执行

```typescript
// 根据条件选择 skill
const skill = conditions
  .reduce(
    (acc, { condition, skill }) => acc === null && condition() ? skill : acc,
    null
  )
```

## 16.8 设计模式

### 16.8.1 Skill 工厂

```typescript
// 创建 skill 的辅助函数
function createSkill(config: SkillConfig): Skill {
  return {
    id: config.id,
    name: config.name,
    description: config.description,
    category: config.category,
    triggers: config.triggers,

    async execute(context) {
      const logs: SkillLog[] = []

      try {
        for (const step of config.steps) {
          logs.push({ timestamp: Date.now(), level: 'info', message: step.description })
          await step.run(context)
        }

        return { output: config.successMessage, logs }
      } catch (error) {
        return { output: '', logs, error: error.message }
      }
    },
  }
}
```

## 16.9 本章小结

Skills 提供了可复用的 Agent 行为：

| 组件 | 职责 |
|------|------|
| `SkillRegistry` | Skill 注册表 |
| `SkillTool` | 执行 Skill 的工具 |
| `SkillContext` | 执行上下文 |
| `discoverSkills` | Skill 发现 |

**核心设计决策**：

1. **触发词** — 多种方式激活 Skill
2. **进度报告** — 可以与用户交互
3. **结果日志** — 记录执行过程
4. **可组合** — 多个 Skill 可以组合

## 16.10 实践要点

1. **选择合适的粒度** — 不要把简单操作做成 Skill
2. **错误处理** — Skill 应该优雅地处理错误
3. **用户确认** — 重要操作需要确认
4. **日志记录** — 记录执行的每一步

---

## 下一步

下一章我们将学习 **错误处理与恢复**，了解 Agent 如何处理各种错误。
