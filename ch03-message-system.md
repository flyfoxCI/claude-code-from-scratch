# 第 3 章：消息系统 — Messages

> "消息是 Agent 记忆的载体。"

## 3.1 消息系统概述

消息系统是 Agent 的记忆核心，它负责：

1. **表示对话历史** — 用户输入、模型输出
2. **携带工具结果** — 工具执行后的返回
3. **传递系统信息** — 错误、警告、进度
4. **支持上下文压缩** — 压缩时需要正确重建消息

```
消息类型
├── UserMessage        # 用户输入
├── AssistantMessage   # 模型输出
├── SystemMessage      # 系统消息
├── AttachmentMessage  # 附件（文件、图片）
└── ProgressMessage    # 进度消息
```

## 3.2 消息类型定义

### 3.2.1 核心消息类型

Claude Code 的消息类型定义在 `src/types/message.js`（从 `@anthropic-ai/sdk` 导入）：

```typescript
// 消息基类
interface Message {
  type: 'user' | 'assistant' | 'system' | 'attachment' | 'progress'
  // ...
}

// 用户消息
interface UserMessage extends Message {
  type: 'user'
  message: {
    role: 'user'
    content: ContentBlock[]
  }
}

// 助手消息
interface AssistantMessage extends Message {
  type: 'assistant'
  message: {
    role: 'assistant'
    content: ContentBlock[]
    usage?: Usage
  }
  // 可能包含工具调用
}
```

### 3.2.2 ContentBlock 类型

```typescript
// 内容块类型
type ContentBlock =
  | { type: 'text'; text: string }                    // 文本
  | { type: 'tool_use'; id: string; name: string; input: object }  // 工具调用
  | { type: 'tool_result'; tool_use_id: string; content: string }   // 工具结果
```

## 3.3 消息创建工具函数

### 3.3.1 创建用户消息

```typescript
// src/utils/messages.ts
export function createUserMessage({
  content,
  agentId,
  isMeta,
  sourceToolAssistantUUID,
  toolUseResult,
}: {
  content: string | ContentBlock[]
  agentId?: AgentId
  isMeta?: boolean
  sourceToolAssistantUUID?: string
  toolUseResult?: string
}): UserMessage {
  const contentBlocks: ContentBlock[] =
    typeof content === 'string'
      ? [{ type: 'text', text: content }]
      : content

  return {
    type: 'user',
    message: {
      role: 'user',
      content: contentBlocks,
    },
    // ... 其他字段
  }
}
```

### 3.3.2 创建系统消息

```typescript
// src/utils/messages.ts
export function createSystemMessage(
  content: string,
  level?: 'info' | 'warning' | 'error',
): SystemMessage {
  return {
    type: 'system',
    level,
    content,
  }
}
```

### 3.3.3 创建工具结果消息

```typescript
// src/utils/messages.ts
export function createToolResultMessage(
  toolUseId: string,
  result: string,
  isError?: boolean,
): UserMessage {
  return {
    type: 'user',
    message: {
      role: 'user',
      content: [
        {
          type: 'tool_result',
          tool_use_id: toolUseId,
          content: result,
          is_error: isError,
        },
      ],
    },
  }
}
```

## 3.4 消息规范化 — normalizeMessagesForAPI

### 3.4.1 为什么需要规范化

不同来源的消息格式可能不同：

```typescript
// 工具执行返回的消息
{
  type: 'tool_result',
  tool_use_id: 'abc123',
  content: 'File content here'
}

// API 期望的格式
{
  type: 'user',
  message: {
    role: 'user',
    content: [{
      type: 'tool_result',
      tool_use_id: 'abc123',
      content: 'File content here'
    }]
  }
}
```

### 3.4.2 规范化函数

```typescript
// src/utils/messages.ts
export function normalizeMessagesForAPI(
  messages: Message[],
  tools: Tools,
): Message[] {
  return messages.map(message => {
    switch (message.type) {
      case 'user':
        return message

      case 'assistant':
        return normalizeAssistantMessage(message)

      case 'attachment':
        return normalizeAttachmentMessage(message)

      default:
        return message
    }
  })
}
```

## 3.5 对话历史管理

### 3.5.1 历史存储

```typescript
// src/history.ts
const HISTORY_FILE = '~/.claude/history.jsonl'

export function getHistory(options?: {
  sessionId?: string
  limit?: number
}): AsyncGenerator<Message> {
  // 从 JSONL 文件读取历史
  // 按时间倒序返回
}
```

### 3.5.2 历史查询

```typescript
// src/history.ts
export function* getHistory(
  sessionId?: string,
  limit?: number,
): Generator<Message> {
  const lines = readLinesSync(HISTORY_FILE)

  // 过滤指定会话
  const filtered = sessionId
    ? lines.filter(line => JSON.parse(line).sessionId === sessionId)
    : lines

  // 倒序返回（最新的在前）
  for (const line of filtered.reverse().slice(0, limit)) {
    yield JSON.parse(line)
  }
}
```

### 3.5.3 粘贴文本处理

```typescript
// src/history.ts
/**
 * 处理粘贴引用，如 [Pasted text #1 +10 lines]
 */
export function expandPastedTextRefs(
  content: string,
  attachments: Attachment[],
): string {
  return content.replace(/\[Pasted text #(\d+)([+-]\d+)? lines?\]/g, (match, index, lineInfo) => {
    const attachment = attachments[parseInt(index) - 1]
    if (!attachment) return match

    if (lineInfo) {
      // 只返回指定行
      const lines = attachment.content.split('\n')
      const [start, end] = parseLineRange(lineInfo, lines.length)
      return lines.slice(start, end).join('\n')
    }

    return attachment.content
  })
}
```

## 3.6 消息与上下文压缩

### 3.6.1 压缩边界

```typescript
// src/utils/messages.ts
export function getMessagesAfterCompactBoundary(
  messages: Message[],
): Message[] {
  const boundaryIndex = messages.findIndex(
    m => m.type === 'system' && m.subtype === 'compact_boundary'
  )

  if (boundaryIndex === -1) {
    return messages
  }

  // 返回压缩边界之后的消息（包括边界消息）
  return messages.slice(boundaryIndex)
}
```

### 3.6.2 压缩后重建

```typescript
// src/services/compact/compact.ts
export function buildPostCompactMessages(
  compactionResult: CompactionResult,
): Message[] {
  const messages: Message[] = []

  // 1. 摘要消息
  messages.push(...compactionResult.summaryMessages)

  // 2. 附件消息
  for (const attachment of compactionResult.attachments) {
    messages.push(createAttachmentMessage(attachment))
  }

  // 3. 钩子结果
  for (const hookResult of compactionResult.hookResults) {
    messages.push(createSystemMessage(hookResult))
  }

  return messages
}
```

## 3.7 附件消息 — AttachmentMessage

### 3.7.1 附件类型

```typescript
// 附件类型
type AttachmentType =
  | 'text_file'           // 文本文件
  | 'image'               // 图片
  | 'edited_text_file'   // 编辑过的文件
  | 'clipboard'           // 剪贴板内容
  | 'web_search'          // 网页搜索结果
```

### 3.7.2 创建附件消息

```typescript
// src/utils/attachments.ts
export function createAttachmentMessage(attachment: Attachment): AttachmentMessage {
  return {
    type: 'attachment',
    attachment: {
      type: attachment.type,
      content: attachment.content,
      name: attachment.name,
      mimeType: attachment.mimeType,
    },
  }
}
```

### 3.7.3 记忆附件

```typescript
// src/utils/attachments.ts
/**
 * 创建记忆附件，用于检索增强
 */
export function createMemoryAttachment(
  memory: Memory,
): Attachment {
  return {
    type: 'memory',
    content: formatMemory(memory),
    source: memory.source,
  }
}
```

## 3.8 消息处理流程

### 3.8.1 完整流程

```
用户输入
    │
    ▼
createUserMessage()
    │
    ▼
normalizeMessagesForAPI()
    │
    ▼
发送给 LLM API
    │
    ▼
LLM 流式响应
    │
    ├── text block ──► 直接 yield
    │
    └── tool_use block
            │
            ▼
        收集到 toolUseBlocks
            │
            ▼
        runTools() 执行工具
            │
            ▼
        createToolResultMessage()
            │
            ▼
        加入 messages 数组
            │
            ▼
        继续下一轮循环或完成
```

### 3.8.2 消息累积

```typescript
// src/query.ts:1715-1717
// 工具执行后，准备下一轮消息
const next: State = {
  messages: [
    ...messagesForQuery,
    ...assistantMessages,
    ...toolResults,
  ],
  // ...
}
```

## 3.9 设计模式

### 3.9.1 消息工厂模式

```typescript
// 创建消息的工厂函数
const MessageFactory = {
  user(content: string): UserMessage {
    return {
      type: 'user',
      message: { role: 'user', content: [{ type: 'text', text: content }] },
    }
  },

  assistant(content: string): AssistantMessage {
    return {
      type: 'assistant',
      message: { role: 'assistant', content: [{ type: 'text', text: content }] },
    }
  },

  toolResult(toolUseId: string, result: string): UserMessage {
    return {
      type: 'user',
      message: {
        role: 'user',
        content: [{
          type: 'tool_result',
          tool_use_id: toolUseId,
          content: result,
        }],
      },
    }
  },
}
```

### 3.9.2 消息变换管道

```typescript
// 消息处理管道
function processMessages(
  messages: Message[],
  transformers: MessageTransformer[],
): Message[] {
  return transformers.reduce(
    (msgs, transformer) => transformer(msgs),
    messages
  )
}

// 使用
const normalized = processMessages(rawMessages, [
  normalizeForAPI,
  removeDuplicates,
  expandPastedRefs,
  addTimestamps,
])
```

## 3.10 本章小结

消息系统是 Agent 记忆的基础：

| 组件 | 职责 |
|------|------|
| `createUserMessage` | 创建用户消息 |
| `normalizeMessagesForAPI` | 格式化消息 |
| `getHistory` | 读取历史 |
| `getMessagesAfterCompactBoundary` | 压缩后消息过滤 |
| `AttachmentMessage` | 附件消息 |

**关键设计决策**：

1. **统一的 ContentBlock** — 文本、工具调用、结果统一表示
2. **消息规范化** — 不同来源消息转为 API 期望格式
3. **历史分离** — 压缩边界之前的消息可以丢弃
4. **附件支持** — 文件、图片作为独立消息类型

## 3.11 实践要点

1. **正确创建消息** — 使用工厂函数避免格式错误
2. **处理粘贴引用** — 用户粘贴代码时的特殊处理
3. **压缩边界意识** — 历史消息可能已被压缩
4. **附件类型** — 不同类型附件有不同渲染方式

---

## 下一步

下一章我们将学习 **工具接口设计**，了解：
- Tool 接口的核心字段
- 输入验证与 Zod Schema
- 工具执行上下文 ToolUseContext
