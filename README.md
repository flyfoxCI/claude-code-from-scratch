# Claude Agent From Scratch

**基于真实源码的 Agent 开发实战指南**

---

## 书籍简介

本书通过解析 Claude Code 的完整源码，系统性地讲解如何从零构建一个生产级别的 AI Agent。

### 目标读者

- 有意开发 AI Agent 的工程师
- 需要理解 Agent 架构和实现细节的开发者
- 对 LLM 应用开发有经验的开发者

### 前置知识

- 熟悉 TypeScript/JavaScript
- 了解 LLM 的基本概念
- 有 API 调用经验更佳

---

## 书籍特色

### 1. 源码驱动

所有概念和实现都来源于 **真实运行的代码**，不是纸上谈兵。

```
你将看到：
├── query.ts 的 1730 行核心循环实现
├── Tool.ts 的工具接口定义
├── BashTool 的完整实现
├── MCP 客户端的 119KB 代码
└── 等等...
```

### 2. 渐进式学习

从简单到复杂，逐步构建理解：

```
第 1 部分：基础
├── 第 1 章：Agent 概述
├── 第 2 章：核心循环  ← 从这里开始
├── 第 3 章：消息系统
│
第 2 部分：工具系统
├── 第 4 章：工具接口  ← 核心抽象
├── 第 5 章：工具注册
├── 第 6 章：工具执行
├── 第 7 章：BashTool  ← 实战范例
│
第 3 部分：上下文管理
├── 第 8-10 章：各种压缩策略
│
第 4 部分：多 Agent
├── 第 11-13 章：多 Agent 协作
│
...
```

### 3. 实用导向

每章包含 **实践要点**：

```markdown
## 实践要点

1. **理解 AsyncGenerator** — 这是 Claude Code 的核心模式
2. **善用源码索引** — 文件路径是理解源码的关键
3. **追踪状态变化** — 使用 queryCheckpoint 理解循环
```

---

## 目录结构

```
book/
├── README.md                    # 本文件
├── book-outline.md              # 详细大纲
└── chapters/                    # 章节内容
    ├── ch01-agent-overview.md   # 第 1 章：Agent 概述与源码环境
    ├── ch02-query-loop.md        # 第 2 章：核心循环
    ├── ch03-message-system.md    # 第 3 章：消息系统
    ├── ch04-tool-interface.md    # 第 4 章：工具接口设计
    ├── ch05-tool-registration.md # 第 5 章：工具注册与发现
    ├── ch06-tool-orchestration.md # 第 6 章：工具执行引擎
    ├── ch07-bash-tool.md         # 第 7 章：BashTool 实战
    ├── ch08-token-budget.md      # 第 8 章：Token 预算与成本控制
    ├── ch09-auto-compact.md      # 第 9 章：自动压缩系统
    ├── ch10-reactive-compact.md  # 第 10 章：响应式压缩
    ├── ch11-multi-agent.md       # 第 11 章：子 Agent 与 AgentTool
    ├── ch12-coordinator-mode.md  # 第 12 章：协调者模式
    ├── ch13-team-system.md       # 第 13 章：团队协作
    ├── ch14-mcp-integration.md   # 第 14 章：MCP 协议集成
    ├── ch15-plugin-system.md     # 第 15 章：插件系统
    ├── ch16-skills-system.md     # 第 16 章：Skills 系统
    ├── ch17-error-handling.md    # 第 17 章：错误处理与恢复
    ├── ch18-permissions.md       # 第 18 章：权限与安全
    ├── ch19-state-management.md  # 第 19 章：状态管理架构
    ├── ch20-cli-repl.md          # 第 20 章：CLI 与 REPL
    └── appendix.md                # 附录
```

---

## 快速开始

### 1. 搭建环境

```bash
# 查看源码
ls ~/code/cc-code/src

# 关键文件
# ├── query.ts         # 核心循环
# ├── Tool.ts          # 工具接口
# ├── tools.ts         # 工具注册
# └── tools/           # 工具实现
```

### 2. 阅读路线

**推荐阅读顺序**：

1. **第 1 章** — 理解整体架构
2. **第 2 章** — 掌握核心循环（最重要）
3. **第 4 章** — 理解工具抽象
4. **第 7 章** — 看一个完整实现
5. **第 11 章** — 多 Agent 协作
6. 其他章节 — 按需深入

### 3. 源码索引

| 功能 | 文件 |
|------|------|
| 核心循环 | `src/query.ts` |
| 工具接口 | `src/Tool.ts` |
| 工具注册 | `src/tools.ts` |
| BashTool | `src/tools/BashTool/BashTool.tsx` |
| 文件工具 | `src/tools/FileReadTool/` |
| AgentTool | `src/tools/AgentTool/` |
| MCP 客户端 | `src/services/mcp/client.ts` |
| 状态管理 | `src/state/AppStateStore.ts` |
| 协调者模式 | `src/coordinator/coordinatorMode.ts` |

---

## 核心概念

### Agent 循环

```
while (true) {
  1. 预处理 — 上下文管理、Token 检查
  2. LLM 调用 — 发送消息、接收流式响应
  3. 工具执行 — 检测 tool_use、执行工具
  4. 继续或终止 — 检查停止条件
}
```

### 工具接口

```typescript
interface Tool<Input, Output> {
  name: string
  call(args: Input, context: ToolUseContext): Promise<ToolResult<Output>>
  description(args: Input): string
  inputSchema: z.ZodSchema
  isConcurrencySafe(args: Input): boolean
  isReadOnly(args: Input): boolean
}
```

### 多 Agent

```
Coordinator
    ├── Worker 1 (并行)
    ├── Worker 2 (并行)
    └── Worker 3 (并行)
```

---

## 章节预览

### 第 2 章：核心循环

**核心源码**：

```typescript
// src/query.ts:219-239
export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | Message, Terminal> {
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  return terminal
}
```

**关键洞察**：
- AsyncGenerator 实现流式响应
- 状态外部化，可序列化
- 多阶段错误恢复

### 第 4 章：工具接口

**核心源码**：

```typescript
// src/Tool.ts:362-420
export type Tool<Input, Output> = {
  name: string
  call(args, context, canUseTool, parentMessage): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  readonly inputSchema: Input
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
}
```

**关键洞察**：
- Zod Schema 验证输入
- ToolUseContext 统一执行环境
- 丰富的元数据支持 LLM 决策

### 第 7 章：BashTool

**核心源码**：

```typescript
// src/tools/BashTool/BashTool.tsx:95-172
export function isSearchOrReadBashCommand(command: string) {
  // 分析命令语义
  // 区分读写操作
}
```

**关键洞察**：
- 命令分类系统
- 权限管理
- 进度报告
- 结果截断

---

## 设计模式

### 1. AsyncGenerator 模式

```typescript
// 状态保持 + 流式处理
async function* agentLoop() {
  let state = initialState
  while (true) {
    state = yield* process(state)
  }
}
```

### 2. 工具策略模式

```typescript
// 统一接口，多种实现
const tool = buildTool({
  name: 'Read',
  inputSchema: z.object({ path: z.string() }),
  async call(input, context) { /* 实现 */ }
})
```

### 3. Store 响应式模式

```typescript
// 简单状态管理
function createStore(initial) {
  let state = initial
  const listeners = new Set()

  return {
    getState: () => state,
    setState: (updater) => {
      state = updater(state)
      listeners.forEach(l => l())
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    }
  }
}
```

### 4. Feature Flag 模式

```typescript
// 条件加载
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

---

## 相关资源

- [Claude Code 源码](../src/)
- [Zod 文档](https://zod.dev)
- [AsyncGenerator MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncGenerator)

---

## 贡献

本书所有内容基于 Claude Code 公开源码编写。发现错误或有改进建议？欢迎提交 Issue 或 PR。

---

*本书基于 Claude Code 源码编写，源码版权属于 Anthropic PBC。*
