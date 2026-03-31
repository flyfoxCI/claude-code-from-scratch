# 第 14 章：MCP 协议集成 — Model Context Protocol

> "MCP 让 Agent 能够连接任何外部工具和数据源。"

## 14.1 MCP 概述

Model Context Protocol (MCP) 是一种标准化协议，允许 AI 应用连接外部工具和数据源。

```
┌─────────────────────────────────────────────────────────────┐
│                     Claude Code                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                   MCP Client                          │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐               │  │
│  │  │ stdio   │  │   SSE   │  │ HTTP   │  ...          │  │
│  │  │Transport│  │Transport│  │Transport│              │  │
│  │  └────┬────┘  └────┬────┘  └────┬────┘               │  │
│  │       └────────────┴─────────────┘                   │  │
│  │                      │                                 │  │
│  │                      ▼                                 │  │
│  │  ┌─────────────────────────────────────────────────┐ │  │
│  │  │              MCP Tools                           │ │  │
│  │  │  (wrapMcpTool 包装为 Claude Code Tool)          │ │  │
│  │  └─────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │filesystem│   │  GitHub │   │  Slack  │
   │ Server  │   │ Server  │   │ Server  │
   └─────────┘   └─────────┘   └─────────┘
```

## 14.2 MCP 客户端架构

### 14.2.1 客户端主类

```typescript
// src/services/mcp/client.ts
export class MCPClient {
  private connections: Map<string, MCPServerConnection> = new Map()
  private tools: Map<string, Tool> = new Map()

  async connect(config: MCPServerConfig): Promise<void> {
    const connection = await this.createConnection(config)
    this.connections.set(config.name, connection)

    // 加载服务器提供的工具
    const serverTools = await connection.listTools()
    for (const tool of serverTools) {
      const wrapped = this.wrapMcpTool(connection, tool)
      this.tools.set(wrapped.name, wrapped)
    }
  }

  async disconnect(name: string): Promise<void> {
    const connection = this.connections.get(name)
    if (connection) {
      await connection.close()
      this.connections.delete(name)
    }
  }

  getTools(): Tools {
    return Array.from(this.tools.values())
  }
}
```

### 14.2.2 连接状态

```typescript
// src/services/mcp/types.ts
export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer

interface ConnectedMCPServer {
  type: 'connected'
  name: string
  transport: Transport
  tools: MCPTool[]
  resources: ServerResource[]
}

interface FailedMCPServer {
  type: 'failed'
  name: string
  error: string
  retryAt?: number
}

interface PendingMCPServer {
  type: 'pending'
  name: string
}
```

## 14.3 传输协议

### 14.3.1 支持的传输类型

```typescript
// src/services/mcp/types.ts
export type Transport = 'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk'

// stdio: 标准输入输出（本地进程）
// SSE: Server-Sent Events
// HTTP: REST API
// WebSocket: 双向通信
// SDK: 使用 MCP SDK
```

### 14.3.2 Stdio 传输

```typescript
// src/services/mcp/transport/stdio.ts
export class StdioTransport implements Transport {
  private process: ChildProcess

  async connect(command: string, args: string[]): Promise<void> {
    this.process = spawn(command, args, {
      stdio: ['pipe', 'pipe', 'pipe'],
    })

    this.process.stdout.on('data', (data) => {
      this.handleMessage(JSON.parse(data.toString()))
    })

    this.process.stderr.on('data', (data) => {
      this.handleError(data.toString())
    })
  }

  async send(message: MCPMessage): Promise<void> {
    this.process.stdin.write(JSON.stringify(message) + '\n')
  }

  async close(): Promise<void> {
    this.process.kill()
  }
}
```

### 14.3.3 SSE 传输

```typescript
// src/services/mcp/transport/sse.ts
export class SSETransport implements Transport {
  private eventSource: EventSource
  private controller: ReadableStreamController

  async connect(url: string, headers?: Record<string, string>): Promise<void> {
    this.eventSource = new EventSource(url, { headers })

    this.eventSource.onmessage = (event) => {
      const message = JSON.parse(event.data)
      this.handleMessage(message)
    }

    this.eventSource.onerror = (error) => {
      this.handleError(error)
    }
  }

  async send(message: MCPMessage): Promise<void> {
    await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message),
    })
  }
}
```

## 14.4 MCP 工具包装

### 14.4.1 包装为 Claude Code 工具

```typescript
// src/services/mcp/client.ts
export function wrapMcpTool(
  connection: MCPServerConnection,
  mcpTool: MCPTool,
): Tool {
  // 构建工具名称
  const toolName = buildMcpToolName(connection.name, mcpTool.name)

  return buildTool({
    name: toolName,
    inputSchema: convertJsonSchemaToZod(mcpTool.inputSchema),

    async call(input, context, canUseTool) {
      try {
        const result = await connection.callTool(mcpTool.name, input)

        // 处理结果
        return {
          data: result,
          mcpMeta: {
            _meta: result._meta,
            structuredContent: result.structuredContent,
          },
        }
      } catch (error) {
        // 错误处理
        return {
          data: { error: error.message },
        }
      }
    },

    description: () => mcpTool.description || `MCP tool: ${mcpTool.name}`,

    isMcp: true,
    isConcurrencySafe: () => mcpTool.canCallConcurrently ?? true,
    isReadOnly: () => mcpTool.readOnly ?? false,
  })
}
```

### 14.4.2 工具名称

```typescript
// src/tools/MCPTool/utils.ts
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `mcp__${serverName}__${toolName}`
}

// 示例
// mcp__filesystem__read_file
// mcp__github__create_issue
// mcp__slack__send_message
```

## 14.5 MCP 资源配置

### 14.5.1 资源列表

```typescript
// src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts
export const ListMcpResourcesTool = buildTool({
  name: 'ListMcpResources',

  inputSchema: z.object({
    server: z.string().optional()
      .describe('Server name to list resources from (all servers if not specified)'),
  }),

  async call(input, context) {
    const servers = input.server
      ? [getServer(input.server)]
      : getAllServers()

    const resources: Record<string, ServerResource[]> = {}

    for (const server of servers) {
      if (server.type === 'connected') {
        resources[server.name] = server.resources
      }
    }

    return { data: { resources } }
  },
})
```

### 14.5.2 读取资源

```typescript
// src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts
export const ReadMcpResourceTool = buildTool({
  name: 'ReadMcpResource',

  inputSchema: z.object({
    uri: z.string()
      .describe('Resource URI to read'),
  }),

  async call(input, context) {
    // 解析 URI 获取服务器和资源
    const { serverName, resourceName } = parseResourceUri(input.uri)

    const server = getServer(serverName)
    if (!server || server.type !== 'connected') {
      return { data: { error: 'Server not found' } }
    }

    const resource = server.resources.find(r => r.uri === input.uri)
    if (!resource) {
      return { data: { error: 'Resource not found' } }
    }

    // 读取资源内容
    const content = await server.readResource(input.uri)

    return {
      data: {
        contents: [{
          uri: input.uri,
          mimeType: resource.mimeType,
          content,
        }],
      },
    }
  },
})
```

## 14.6 服务器配置

### 14.6.1 配置类型

```typescript
// src/services/mcp/types.ts
interface McpStdioServerConfig {
  type: 'stdio'
  name: string
  command: string
  args?: string[]
  env?: Record<string, string>
}

interface McpSSEServerConfig {
  type: 'sse'
  name: string
  url: string
  headers?: Record<string, string>
}

interface McpHTTPServerConfig {
  type: 'http'
  name: string
  url: string
  headers?: Record<string, string>
}

interface McpWebSocketServerConfig {
  type: 'ws'
  name: string
  url: string
}
```

### 14.6.2 OAuth 支持

```typescript
// src/services/mcp/types.ts
interface McpOAuthConfig {
  clientId: string
  clientSecret: string
  authUrl: string
  tokenUrl: string
  scopes?: string[]
}

interface NeedsAuthMCPServer {
  type: 'needs_auth'
  name: string
  authConfig: McpOAuthConfig
}
```

## 14.7 错误处理

### 14.7.1 连接失败

```typescript
// src/services/mcp/client.ts
async connect(config: MCPServerConfig): Promise<void> {
  try {
    const connection = await this.createConnection(config)
    this.connections.set(config.name, connection)
  } catch (error) {
    // 记录失败状态
    this.connections.set(config.name, {
      type: 'failed',
      name: config.name,
      error: error.message,
      retryAt: Date.now() + RETRY_DELAY_MS,
    })

    // 触发重试
    this.scheduleRetry(config.name)
  }
}
```

### 14.7.2 工具调用失败

```typescript
// src/tools/MCPTool/MCPTool.ts
async call(input, context) {
  try {
    const result = await this.connection.callTool(this.mcpTool.name, input)

    // 处理 MCP 错误码
    if (result.isError) {
      return {
        data: {
          error: result.error || 'Tool execution failed',
          code: result.code,
        },
      }
    }

    return { data: result }
  } catch (error) {
    // 转换错误为 Claude Code 格式
    if (error.code === 'TOOL_NOT_FOUND') {
      return { data: { error: `MCP tool not found: ${this.mcpTool.name}` } }
    }

    if (error.code === 'INVALID_INPUT') {
      return { data: { error: `Invalid input: ${error.message}` } }
    }

    return { data: { error: `MCP error: ${error.message}` } }
  }
}
```

## 14.8 设计模式

### 14.8.1 MCP 工厂

```typescript
// 创建 MCP 连接的工厂
class MCPConnectionFactory {
  static async create(config: MCPServerConfig): Promise<MCPServerConnection> {
    switch (config.type) {
      case 'stdio':
        return new StdioTransport().connect(config.command, config.args)

      case 'sse':
        return new SSETransport().connect(config.url, config.headers)

      case 'http':
        return new HTTPTransport().connect(config.url, config.headers)

      case 'ws':
        return new WebSocketTransport().connect(config.url)

      default:
        throw new Error(`Unsupported transport: ${config.type}`)
    }
  }
}
```

### 14.8.2 工具缓存

```typescript
// 缓存 MCP 工具避免重复加载
class MCPToolCache {
  private cache: Map<string, { tools: Tool[]; timestamp: number }> = new Map()
  private ttl = 5 * 60 * 1000  // 5 分钟

  get(serverName: string): Tool[] | null {
    const cached = this.cache.get(serverName)
    if (!cached) return null

    if (Date.now() - cached.timestamp > this.ttl) {
      this.cache.delete(serverName)
      return null
    }

    return cached.tools
  }

  set(serverName: string, tools: Tool[]): void {
    this.cache.set(serverName, {
      tools,
      timestamp: Date.now(),
    })
  }
}
```

## 14.9 本章小结

MCP 协议让 Claude Code 能够连接无限的外部工具：

| 组件 | 职责 |
|------|------|
| `MCPClient` | MCP 连接管理 |
| `StdioTransport` | 本地进程通信 |
| `SSETransport` | Server-Sent Events |
| `wrapMcpTool` | 工具包装 |
| `MCPToolCache` | 工具缓存 |

**核心设计决策**：

1. **统一传输** — 支持多种传输协议
2. **状态管理** — 连接状态跟踪
3. **工具包装** — MCP 工具转换为 Claude Code 工具
4. **错误恢复** — 失败状态和重试机制

## 14.10 实践要点

1. **选择传输** — 本地工具用 stdio，远程用 SSE/HTTP
2. **错误处理** — MCP 工具调用要处理各种错误码
3. **缓存** — 避免频繁加载工具
4. **OAuth** — 需要认证的服务器配置 OAuth

---

## 下一步

下一章我们将学习 **插件系统**，了解如何扩展 Claude Code 的功能。
