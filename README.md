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

## 核心图表

| 图表 | 说明 |
|------|------|
| [Agent 核心循环](diagrams/agent-loop.svg) | Query Loop 的四阶段执行模型 |
| [架构概览](diagrams/architecture-overview.svg) | Claude Code 整体架构组件 |
| [工具系统架构](diagrams/tool-system-hierarchy.svg) | 43个内置工具 + MCP 扩展 |
| [多代理模式](diagrams/multi-agent-modes.svg) | 单代理/协调者/团队协作模式 |
| [上下文管理](diagrams/context-management.svg) | Token 预算与自动压缩 |
| [状态管理](diagrams/state-management.svg) | AppStateStore 与 TaskStore |

---

## 书籍结构（9 部分）

```
claude-code-from-scratch/
├── diagrams/                      # 架构图表
│   ├── agent-loop.svg            # Agent 核心循环
│   ├── architecture-overview.svg # 整体架构
│   ├── tool-system-hierarchy.svg # 工具系统
│   ├── multi-agent-modes.svg     # 多代理模式
│   ├── context-management.svg    # 上下文管理
│   └── state-management.svg      # 状态管理
│
├── part1-introduction/           # 第一部分：认识 Claude Code
│   └── ch01-agent-overview.md
│
├── part2-architecture/           # 第二部分：架构设计
│   ├── ch02-query-loop.md       # 核心循环
│   ├── ch03-message-system.md    # 消息系统
│   ├── ch19-state-management.md  # 状态管理
│   └── ch20-cli-repl.md          # CLI/REPL
│
├── part3-tool-system/            # 第三部分：工具系统
│   ├── ch04-tool-interface.md    # 工具接口
│   ├── ch05-tool-registration.md # 工具注册
│   ├── ch06-tool-orchestration.md # 工具编排
│   └── ch07-bash-tool.md         # BashTool
│
├── part4-context-management/     # 第四部分：上下文管理
│   ├── ch08-token-budget.md      # Token 预算
│   ├── ch09-auto-compact.md      # 自动压缩
│   └── ch10-reactive-compact.md   # 响应式压缩
│
├── part5-multi-agent/            # 第五部分：多代理
│   ├── ch11-multi-agent.md       # 子 Agent
│   ├── ch12-coordinator-mode.md  # 协调者模式
│   └── ch13-team-system.md       # 团队协作
│
├── part6-extensions/            # 第六部分：扩展系统
│   ├── ch14-mcp-integration.md   # MCP 协议
│   ├── ch15-plugin-system.md     # 插件系统
│   └── ch16-skills-system.md     # Skills
│
├── part7-infrastructure/        # 第七部分：基础设施
│   └── ch17-error-handling.md    # 错误处理
│
├── part8-security/              # 第八部分：安全与权限
│   └── ch18-permissions.md       # 权限模型
│
└── part9-design-philosophy/     # 第九部分：设计哲学
    └── appendix.md               # 总结与展望
```

---

## 快速导航

### 按主题查找

| 主题 | 章节 | 图表 |
|------|------|------|
| Agent 核心循环 | ch02 | [agent-loop.svg](diagrams/agent-loop.svg) |
| 工具系统 | ch04-07 | [tool-system-hierarchy.svg](diagrams/tool-system-hierarchy.svg) |
| 上下文压缩 | ch08-10 | [context-management.svg](diagrams/context-management.svg) |
| 多代理协作 | ch11-13 | [multi-agent-modes.svg](diagrams/multi-agent-modes.svg) |
| 状态管理 | ch19 | [state-management.svg](diagrams/state-management.svg) |

### 推荐阅读路线

1. **入门** → 第 1 章 + 架构概览图
2. **核心** → 第 2 章 (Query Loop) + agent-loop.svg
3. **工具** → 第 4-7 章 + tool-system-hierarchy.svg
4. **进阶** → 第 11-13 章 (多代理) + multi-agent-modes.svg

---

## 源码索引

| 功能 | 文件 |
|------|------|
| 核心循环 | `src/query.ts` |
| 工具接口 | `src/Tool.ts` |
| 工具注册 | `src/tools.ts` |
| BashTool | `src/tools/BashTool/BashTool.tsx` |
| AgentTool | `src/tools/AgentTool/` |
| MCP 客户端 | `src/services/mcp/client.ts` |
| 状态管理 | `src/state/AppStateStore.ts` |
| 协调者模式 | `src/coordinator/coordinatorMode.ts` |

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
