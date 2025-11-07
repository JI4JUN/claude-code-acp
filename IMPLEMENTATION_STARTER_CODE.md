# 你的项目实现起点 - 代码框架示例

这个文件展示了如何在你的 Electron/Tauri 项目中应用 claude-code-acp 的设计模式。

## 文件结构

```
your-ai-chat-client/
├── src/
│  ├── adapters/
│  │  ├── ClaudeCodeAdapter.ts       # 核心适配层（类似 acp-agent.ts）
│  │  ├── MessageTransformer.ts      # 消息转换（类似 acp-agent.ts 的 promptToClaude）
│  │  └── ToolManager.ts             # 工具管理（类似 mcp-server.ts）
│  ├── tools/
│  │  ├── FileEditorTool.ts          # Read/Write/Edit 工具
│  │  ├── CommandExecutorTool.ts     # Bash/BashOutput/KillShell 工具
│  │  └── ToolBase.ts                # 基类
│  ├── permissions/
│  │  ├── PermissionManager.ts       # 权限控制
│  │  └── PermissionPrompt.ts        # UI 反馈
│  ├── types/
│  │  ├── session.ts                 # 会话类型
│  │  ├── tools.ts                   # 工具类型
│  │  └── messages.ts                # 消息类型
│  └── utils/
│     ├── cache.ts                   # 缓存管理
│     └── validators.ts              # 验证工具
└── electron/
   └── preload.ts / main.ts          # Electron 主进程
```

## 核心代码示例

### 1. 会话类型定义 (types/session.ts)

```typescript
// 参考 acp-agent.ts 第 60-85 行

import { Query } from '@anthropic-ai/claude-agent-sdk';

export type PermissionMode = 'default' | 'acceptEdits' | 'plan' | 'bypassPermissions';

export interface Session {
  id: string;
  query: Query;                          // Claude Code SDK 的核心对象
  userInput: AsyncIterable<UserMessage>; // 用户输入队列
  cancelled: boolean;
  permissionMode: PermissionMode;
  createdAt: Date;
}

export interface UserMessage {
  type: 'user';
  message: {
    role: 'user';
    content: ContentBlock[];
  };
}

export type ContentBlock =
  | { type: 'text'; text: string }
  | { type: 'image'; source: { type: 'base64' | 'url'; data?: string; url?: string; media_type?: string } }
  | { type: 'resource'; resource: { uri: string; text: string } };

export interface ToolCall {
  type: 'tool_call' | 'server_tool_use' | 'mcp_tool_use';
  id: string;
  name: string;
  input: Record<string, any>;
}

export interface ToolResult {
  toolUseId: string;
  content: string;
  isError?: boolean;
}
```

### 2. Claude Code 适配器 (adapters/ClaudeCodeAdapter.ts)

```typescript
// 参考 acp-agent.ts 的 ClaudeAcpAgent class

import { query, Query, PermissionMode, Options } from '@anthropic-ai/claude-agent-sdk';
import { Session, UserMessage, ToolCall } from '../types/session';
import { MessageTransformer } from './MessageTransformer';
import { ToolManager } from './ToolManager';
import { PermissionManager } from '../permissions/PermissionManager';

export class ClaudeCodeAdapter {
  private sessions: Map<string, Session> = new Map();
  private toolUseCache: Map<string, ToolCall> = new Map();
  private fileContentCache: Map<string, string> = new Map();
  private permissionManager: PermissionManager;
  private toolManager: ToolManager;
  private messageTransformer: MessageTransformer;

  constructor() {
    this.permissionManager = new PermissionManager();
    this.toolManager = new ToolManager();
    this.messageTransformer = new MessageTransformer();
  }

  /**
   * 创建新会话
   * 参考：acp-agent.ts newSession() 方法 (第 138-307 行)
   */
  async createSession(cwd: string): Promise<string> {
    const sessionId = this.generateId();

    // 创建输入队列（参考 utils.ts 的 Pushable<T>）
    const userInputQueue = new AsyncQueue<UserMessage>();

    // 配置 SDK 选项
    const options: Options = {
      cwd,
      includePartialMessages: true,
      mcpServers: {}, // 你的项目不需要额外的 MCP 服务器
      systemPrompt: { type: 'preset', preset: 'claude_code' },
      permissionMode: 'default',
      // 其他选项...
    };

    // 创建 Query 对象（核心）
    const q = query({
      prompt: userInputQueue,
      options,
    });

    // 保存会话
    const session: Session = {
      id: sessionId,
      query: q,
      userInput: userInputQueue,
      cancelled: false,
      permissionMode: 'default',
      createdAt: new Date(),
    };

    this.sessions.set(sessionId, session);
    return sessionId;
  }

  /**
   * 发送消息并处理响应
   * 参考：acp-agent.ts prompt() 方法 (第 314-439 行)
   */
  async handleUserMessage(
    sessionId: string,
    userPrompt: string,
    onUpdate: (update: any) => void
  ): Promise<void> {
    const session = this.sessions.get(sessionId);
    if (!session) throw new Error('Session not found');

    // 将用户输入转换为 SDK 格式
    const sdkMessage = this.messageTransformer.promptToSdk(userPrompt);

    // 推入用户消息队列
    (session.userInput as AsyncQueue<UserMessage>).push(sdkMessage);

    // 主消息循环（参考 acp-agent.ts 第 324-437 行）
    while (true) {
      const { value: message, done } = await session.query.next();

      if (done || !message) {
        if (session.cancelled) {
          onUpdate({ type: 'cancelled' });
        }
        break;
      }

      // 处理不同类型的消息
      switch (message.type) {
        case 'system':
          // 系统消息 (init, compact_boundary 等)
          break;

        case 'stream_event':
          // 实时流事件
          this.handleStreamEvent(message, sessionId, onUpdate);
          break;

        case 'result':
          // 对话结束
          if (message.subtype === 'success') {
            onUpdate({ type: 'complete', reason: 'success' });
          } else {
            onUpdate({ type: 'error', reason: message.subtype });
          }
          break;

        case 'assistant':
        case 'user':
          // 处理内容块
          const notifications = this.messageTransformer.sdkMessageToNotifications(
            message.message.content,
            message.message.role,
            this.toolUseCache,
            this.fileContentCache
          );

          for (const notification of notifications) {
            onUpdate(notification);

            // 如果是工具调用，检查权限
            if (notification.type === 'tool_call') {
              const approved = await this.permissionManager.checkPermission(
                notification,
                session.permissionMode
              );

              if (!approved) {
                onUpdate({
                  type: 'tool_rejected',
                  toolCallId: notification.toolCallId,
                });
              }
            }
          }
          break;
      }
    }
  }

  /**
   * 处理实时流事件
   * 参考：acp-agent.ts streamEventToAcpNotifications() (第 778-812 行)
   */
  private handleStreamEvent(
    message: any,
    sessionId: string,
    onUpdate: (update: any) => void
  ): void {
    const event = message.event;

    switch (event.type) {
      case 'content_block_start':
      case 'content_block_delta':
        // 转换为客户端格式并发送
        const notification = this.messageTransformer.streamEventToNotification(
          event,
          message.message.role
        );
        onUpdate(notification);
        break;

      case 'message_stop':
      case 'content_block_stop':
        // 结束信号
        break;
    }
  }

  /**
   * 切换权限模式
   * 参考：acp-agent.ts setSessionMode() (第 456-479 行)
   */
  async setPermissionMode(
    sessionId: string,
    mode: PermissionMode
  ): Promise<void> {
    const session = this.sessions.get(sessionId);
    if (!session) throw new Error('Session not found');

    session.permissionMode = mode;
    await session.query.setPermissionMode(mode);
  }

  /**
   * 取消对话
   * 参考：acp-agent.ts cancel() (第 441-447 行)
   */
  async cancelSession(sessionId: string): Promise<void> {
    const session = this.sessions.get(sessionId);
    if (!session) throw new Error('Session not found');

    session.cancelled = true;
    await session.query.interrupt();
  }

  /**
   * 读取文件
   * 参考：acp-agent.ts readTextFile() (第 481-487 行)
   */
  async readFile(sessionId: string, path: string): Promise<string> {
    const session = this.sessions.get(sessionId);
    if (!session) throw new Error('Session not found');

    // 如果有缓存，先返回缓存
    if (this.fileContentCache.has(path)) {
      return this.fileContentCache.get(path)!;
    }

    // 否则读取文件（实际实现）
    const fs = await import('fs').then(m => m.promises);
    const content = await fs.readFile(path, 'utf-8');

    // 更新缓存
    this.fileContentCache.set(path, content);

    return content;
  }

  /**
   * 写入文件
   * 参考：acp-agent.ts writeTextFile() (第 489-493 行)
   */
  async writeFile(sessionId: string, path: string, content: string): Promise<void> {
    const fs = await import('fs').then(m => m.promises);
    await fs.writeFile(path, content, 'utf-8');

    // 更新缓存
    this.fileContentCache.set(path, content);
  }

  private generateId(): string {
    return `session_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// AsyncQueue 实现 (参考 utils.ts 的 Pushable<T>)
class AsyncQueue<T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private resolvers: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;

  push(item: T) {
    if (this.resolvers.length > 0) {
      const resolve = this.resolvers.shift()!;
      resolve({ value: item, done: false });
    } else {
      this.queue.push(item);
    }
  }

  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;
      } else if (this.done) {
        break;
      } else {
        await new Promise<void>((resolve) => {
          this.resolvers.push((result) => {
            if (!result.done) {
              resolve();
            }
          });
        });
      }
    }
  }
}
```

### 3. 消息转换器 (adapters/MessageTransformer.ts)

```typescript
// 参考 acp-agent.ts 的 promptToClaude() 和 toAcpNotifications()

export class MessageTransformer {
  /**
   * 将用户输入转换为 SDK 格式
   * 参考：acp-agent.ts promptToClaude() (第 561-638 行)
   */
  promptToSdk(userPrompt: string): UserMessage {
    return {
      type: 'user',
      message: {
        role: 'user',
        content: [
          {
            type: 'text',
            text: userPrompt,
          },
        ],
      },
    };
  }

  /**
   * 将 SDK 消息转换为客户端格式
   * 参考：acp-agent.ts toAcpNotifications() (第 644-776 行)
   */
  sdkMessageToNotifications(
    content: any,
    role: 'user' | 'assistant',
    toolUseCache: Map<string, ToolCall>,
    fileContentCache: Map<string, string>
  ): any[] {
    const notifications = [];

    if (typeof content === 'string') {
      // 纯文本消息
      notifications.push({
        type: 'text',
        role,
        text: content,
      });
      return notifications;
    }

    // 处理内容块数组
    for (const chunk of content) {
      switch (chunk.type) {
        case 'text':
        case 'text_delta':
          notifications.push({
            type: 'text',
            role,
            text: chunk.text,
          });
          break;

        case 'image':
          notifications.push({
            type: 'image',
            role,
            source: chunk.source,
          });
          break;

        case 'tool_use':
        case 'server_tool_use':
        case 'mcp_tool_use':
          // 缓存工具使用
          toolUseCache.set(chunk.id, {
            type: chunk.type,
            id: chunk.id,
            name: chunk.name,
            input: chunk.input,
          });

          // 转换为客户端格式
          notifications.push({
            type: 'tool_call',
            toolCallId: chunk.id,
            toolName: chunk.name,
            input: chunk.input,
          });
          break;

        case 'tool_result':
          // 工具结果
          const originalTool = toolUseCache.get(chunk.tool_use_id);
          if (originalTool) {
            notifications.push({
              type: 'tool_result',
              toolCallId: chunk.tool_use_id,
              toolName: originalTool.name,
              result: chunk.content,
              isError: chunk.is_error || false,
            });
          }
          break;
      }
    }

    return notifications;
  }

  /**
   * 处理流事件
   * 参考：acp-agent.ts streamEventToAcpNotifications() (第 778-812 行)
   */
  streamEventToNotification(event: any, role: 'user' | 'assistant'): any {
    switch (event.type) {
      case 'content_block_start':
        return {
          type: 'stream_start',
          blockType: event.content_block.type,
        };

      case 'content_block_delta':
        return {
          type: 'stream_delta',
          delta: event.delta,
        };

      case 'content_block_stop':
        return {
          type: 'stream_stop',
        };

      default:
        return null;
    }
  }
}
```

### 4. 工具管理器 (adapters/ToolManager.ts)

```typescript
// 参考 mcp-server.ts 的工具注册

export class ToolManager {
  private tools: Map<string, ToolHandler> = new Map();

  constructor() {
    this.registerDefaultTools();
  }

  private registerDefaultTools(): void {
    // Read 工具
    this.registerTool('Read', {
      description: 'Read file contents',
      inputSchema: {
        file_path: 'string',
        offset: 'number?',
        limit: 'number?',
      },
      handler: async (input) => {
        // 实现文件读取（参考 mcp-server.ts 第 95-155 行）
      },
    });

    // Write 工具
    this.registerTool('Write', {
      description: 'Write file contents',
      inputSchema: {
        file_path: 'string',
        content: 'string',
      },
      handler: async (input) => {
        // 实现文件写入（参考 mcp-server.ts 第 189-221 行）
      },
    });

    // Edit 工具
    this.registerTool('Edit', {
      description: 'Edit file with string replacement',
      inputSchema: {
        file_path: 'string',
        old_string: 'string',
        new_string: 'string',
        replace_all: 'boolean?',
      },
      handler: async (input) => {
        // 实现文件编辑（参考 mcp-server.ts 第 260-302 行）
      },
    });

    // Bash 工具
    this.registerTool('Bash', {
      description: 'Execute bash command',
      inputSchema: {
        command: 'string',
        timeout: 'number?',
        description: 'string?',
        run_in_background: 'boolean?',
      },
      handler: async (input) => {
        // 实现命令执行（参考 mcp-server.ts 第 341-450 行）
      },
    });
  }

  private registerTool(name: string, handler: ToolHandler): void {
    this.tools.set(name, handler);
  }

  async executeTool(name: string, input: any): Promise<any> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Unknown tool: ${name}`);

    return tool.handler(input);
  }
}

interface ToolHandler {
  description: string;
  inputSchema: Record<string, string>;
  handler: (input: any) => Promise<any>;
}
```

### 5. 权限管理器 (permissions/PermissionManager.ts)

```typescript
// 参考 acp-agent.ts 第 205-217 行和 456-479 行

export type PermissionMode = 'default' | 'acceptEdits' | 'plan' | 'bypassPermissions';

export class PermissionManager {
  private currentMode: PermissionMode = 'default';

  /**
   * 检查工具是否被允许
   */
  async checkPermission(
    toolCall: any,
    mode: PermissionMode = this.currentMode
  ): Promise<boolean> {
    // 参考权限状态机（ARCHITECTURE_DIAGRAMS.md）

    if (mode === 'plan') {
      // 只读模式，拒绝所有写入
      if (['Write', 'Edit', 'Bash'].includes(toolCall.toolName)) {
        return false;
      }
    }

    if (mode === 'bypassPermissions') {
      // 跳过所有权限检查
      return true;
    }

    if (mode === 'acceptEdits') {
      // 自动接受编辑工具
      if (['Write', 'Edit'].includes(toolCall.toolName)) {
        return true;
      }
    }

    if (mode === 'default') {
      // 默认模式，需要提示用户
      return await this.promptUser(toolCall);
    }

    return false;
  }

  private async promptUser(toolCall: any): Promise<boolean> {
    // 这应该触发 UI 中的权限提示
    // 返回用户的选择（批准或拒绝）
    return new Promise((resolve) => {
      // 发送事件到 UI，等待用户响应
      window.electronAPI?.promptPermission(toolCall).then(resolve);
    });
  }

  setMode(mode: PermissionMode): void {
    this.currentMode = mode;
  }

  getMode(): PermissionMode {
    return this.currentMode;
  }
}
```

## 在 React/Vue 组件中使用

### React 示例

```tsx
import React, { useState, useRef } from 'react';
import { ClaudeCodeAdapter } from './adapters/ClaudeCodeAdapter';

export const ChatWithCodeEditor = () => {
  const [messages, setMessages] = useState<any[]>([]);
  const [sessionId, setSessionId] = useState<string | null>(null);
  const adapterRef = useRef(new ClaudeCodeAdapter());

  const initializeSession = async (projectPath: string) => {
    const sid = await adapterRef.current.createSession(projectPath);
    setSessionId(sid);
  };

  const handleSendMessage = async (userMessage: string) => {
    if (!sessionId) return;

    adapterRef.current.handleUserMessage(sessionId, userMessage, (update) => {
      // 处理各种更新类型
      switch (update.type) {
        case 'text':
          setMessages((prev) => [
            ...prev,
            { type: 'text', role: update.role, content: update.text },
          ]);
          break;

        case 'tool_call':
          setMessages((prev) => [
            ...prev,
            {
              type: 'tool_call',
              toolCallId: update.toolCallId,
              toolName: update.toolName,
              input: update.input,
            },
          ]);
          // 显示权限提示
          showPermissionPrompt(update);
          break;

        case 'tool_result':
          setMessages((prev) => [
            ...prev,
            {
              type: 'tool_result',
              toolCallId: update.toolCallId,
              result: update.result,
            },
          ]);
          break;

        case 'complete':
          // 对话结束
          break;
      }
    });
  };

  const showPermissionPrompt = (toolCall: any) => {
    // 在 UI 中显示权限提示
    // 等待用户批准或拒绝
  };

  return (
    <div>
      <input
        type="text"
        placeholder="Enter your message..."
        onKeyPress={(e) => {
          if (e.key === 'Enter') {
            handleSendMessage(e.currentTarget.value);
          }
        }}
      />
      {messages.map((msg, idx) => (
        <div key={idx}>
          {msg.type === 'text' && <p>{msg.content}</p>}
          {msg.type === 'tool_call' && (
            <div>
              <strong>{msg.toolName}</strong>
              <pre>{JSON.stringify(msg.input, null, 2)}</pre>
              <button onClick={() => approveToolCall(msg.toolCallId)}>
                Approve
              </button>
            </div>
          )}
        </div>
      ))}
    </div>
  );
};
```

## 关键实现点总结

1. **SessionManager**: 管理多个会话和其 Query 对象
2. **MessageTransformer**: SDK 消息 ↔ UI 消息转换
3. **ToolManager**: 工具的注册和执行
4. **PermissionManager**: 权限检查和模式管理
5. **FileCache**: 文件内容缓存
6. **EventEmitter**: 向 UI 推送更新

## 下一步

1. 根据这个框架创建你的项目结构
2. 逐步实现每个模块
3. 参考 claude-code-acp 的源代码处理复杂逻辑
4. 添加你的 UI 层

祝实现顺利！
