# 第 1 章：Agent 概述与源码环境搭建

> "Agent 是 LLM 的下一次进化——从被动响应到主动执行。"

## 1.1 什么是 Agent？

在传统的 LLM 使用模式中，模型接收一个输入，返回一个输出。这是一种**请求-响应**模式，模型本身不持有状态，不执行任何操作。

**Agent** 改变了这个范式。一个 Agent 包含：

1. **一个 LLM** — 决策中心
2. **工具 (Tools)** — 与外部世界交互的能力
3. **记忆 (Memory)** — 跨交互保持状态
4. **循环 (Loop)** — 持续的"思考-执行-反馈"机制

```
┌─────────────────────────────────────────────────────────┐
│                      USER INPUT                          │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                      LLM (思考)                          │
│   "用户想要什么？我需要调用什么工具？"                      │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                     TOOL CALL                            │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│   │ Bash    │  │ File    │  │ Web     │                │
│   │ Tool    │  │ Tools   │  │ Search  │  ...           │
│   └─────────┘  └─────────┘  └─────────┘                │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                   TOOL RESULTS                           │
│   "文件已读取" "命令执行完成" "搜索结果：..."              │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ───► 返回 LLM 继续思考
                              │
                              ▼
                      [重复直到完成]
```

## 1.2 Claude Code 架构概览

Claude Code 是一个生产级别的 Agent，本节从宏观角度理解它的架构。

### 1.2.1 源码目录结构

```
src/
├── main.tsx                    # 应用入口
├── query.ts                    # 核心 Agent 循环 (Query Loop)
├── QueryEngine.ts              # 查询引擎
├── Tool.ts                     # 工具接口定义
├── tools.ts                    # 工具注册
├── tools/                      # 工具实现
│   ├── AgentTool/              # 子 Agent 工具
│   ├── BashTool/               # Shell 执行工具
│   ├── FileEditTool/           # 文件编辑工具
│   ├── FileReadTool/           # 文件读取工具
│   ├── MCPTool/                # MCP 协议工具
│   └── ...                     # 30+ 工具
├── services/
│   ├── mcp/                    # MCP 协议客户端
│   ├── compact/                 # 上下文压缩服务
│   └── tools/                  # 工具执行服务
├── coordinator/                # 协调者模式
├── state/                      # 状态管理
├── entrypoints/                # CLI 入口
└── utils/                      # 工具函数
```

### 1.2.2 核心组件关系

```
┌──────────────────────────────────────────────────────────────┐
│                         main.tsx                              │
│                    (应用入口，初始化)                           │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                       QueryEngine.ts                          │
│               (管理单个对话会话的生命周期)                      │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                        query.ts                              │
│                 (核心 AsyncGenerator 循环)                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  while (true) {                       │    │
│  │    1. 调用 LLM API                                   │    │
│  │    2. 处理工具调用                                   │    │
│  │    3. 管理上下文压缩                                 │    │
│  │    4. 检查停止条件                                   │    │
│  │  }                                                   │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   BashTool      │  │  FileTools      │  │  AgentTool     │
│   (Shell执行)    │  │  (文件操作)      │  │  (子Agent)     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                    MCP Client (services/mcp/)                │
│              (Model Context Protocol 集成)                   │
└──────────────────────────────────────────────────────────────┘
```

## 1.3 源码环境搭建

### 1.3.1 获取源码

Claude Code 的源码已从 npm 包中提取到本地：

```bash
# 源码位置
ls ~/code/cc-code/src
```

### 1.3.2 关键文件速查

| 功能 | 文件路径 | 核心内容 |
|------|---------|---------|
| 入口 | `src/main.tsx` | 应用初始化、配置加载 |
| 循环 | `src/query.ts` | 1730 行，核心 AsyncGenerator |
| 工具 | `src/Tool.ts` | Tool 接口定义 |
| 注册 | `src/tools.ts` | 30+ 工具的注册 |
| MCP | `src/services/mcp/client.ts` | 119KB，MCP 客户端 |

### 1.3.3 调试技巧

**1. 启用详细日志**

Claude Code 使用 `verbose` 标志控制日志输出：

```typescript
// src/state/AppState.tsx
const verbose = state.verbose || false
```

**2. 查看 Query Loop 的检查点**

源码中使用 `queryCheckpoint` 进行性能追踪：

```typescript
// src/query.ts:339
queryCheckpoint('query_fn_entry')        // 函数入口
queryCheckpoint('query_api_streaming_start') // API 调用开始
queryCheckpoint('query_tool_execution_start') // 工具执行开始
```

**3. 追踪工具执行**

工具执行有详细的日志：

```typescript
// src/query.ts:1366-1378
if (streamingToolExecutor) {
  logEvent('tengu_streaming_tool_execution_used', {
    tool_count: toolUseBlocks.length,
    queryChainId: queryChainIdForAnalytics,
    queryDepth: queryTracking.depth,
  })
}
```

## 1.4 Claude Code 的设计哲学

通过源码分析，我们可以看到 Claude Code 的几个关键设计决策：

### 1.4.1 AsyncGenerator 作为核心模式

```typescript
// src/query.ts:219-239
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent      // 流式事件
  | RequestStartEvent
  | Message          // 消息
  | TombstoneMessage // 删除标记
  | ToolUseSummaryMessage, // 工具摘要
  Terminal          // 终止状态
>
```

**为什么选择 AsyncGenerator？**

1. **流式响应** — 模型输出可以边生成边 yield
2. **状态保持** — 循环状态在 generator 内部维护
3. **可控执行** — 调用者可以控制迭代节奏
4. **内存效率** — 不需要一次性加载所有数据

### 1.4.2 状态外部化

Claude Code 没有把状态藏在函数内部，而是显式地管理：

```typescript
// src/query.ts:204-217
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

**好处**：
- 状态可以被序列化和恢复
- 便于测试和调试
- 支持中断和继续

### 1.4.3 特性开关 (Feature Flags)

```typescript
// 使用 bun:bundle 的特性开关
import { feature } from 'bun:bundle'

const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null
```

这允许：
- A/B 测试新功能
- 按环境启用/禁用特性
- Dead code elimination 减小包体积

## 1.5 本章小结

本章我们：

1. **理解 Agent 是什么** — LLM + 工具 + 记忆 + 循环
2. **看懂 Claude Code 架构** — 目录结构、核心组件关系
3. **搭建开发环境** — 源码位置、调试技巧
4. **领悟设计哲学** — AsyncGenerator、状态外部化、特性开关

## 1.6 实践要点

1. **理解 AsyncGenerator**：这是 Claude Code 的核心模式，建议先熟悉这个模式
2. **善用源码索引**：本节提供的文件路径是理解源码的关键
3. **追踪状态变化**：使用 `queryCheckpoint` 理解循环中的状态流转
4. **特性开关思维**：理解这种设计如何支持快速迭代

---

## 下一步

下一章我们将深入 **Query Loop**，这是 Agent 的核心循环。我们将看到：
- 如何用 AsyncGenerator 实现流式处理
- 状态如何在循环中传递
- 工具调用如何与模型交互
