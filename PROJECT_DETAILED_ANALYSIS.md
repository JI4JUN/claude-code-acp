# claude-code-acp 项目详细分析

## 项目概述

**claude-code-acp** 是一个 **ACP (Agent Client Protocol)** 服务器适配器，用于在 ACP 兼容的客户端（如 Zed、Emacs、Neovim、marimo notebook）中使用 Claude Code。

本项目充当一个**协议适配层**，将 ACP 协议请求转换为 Claude Code SDK 理解的格式，反之亦然。

### 核心目标
- 为任何支持 ACP 的编辑器提供 Claude Code 的功能
- 维护 ACP 协议兼容性
- 管理用户权限和工具执行
- 支持实时流式响应和交互式终端

---

## 项目结构（2533 行代码）

```
src/
├── index.ts                      (26 行)   - CLI 入口点
├── lib.ts                        (25 行)   - 库导出接口
├── acp-agent.ts                 (820 行)  - ACP 代理核心实现 ⭐
├── mcp-server.ts                (928 行)  - MCP 服务器和工具定义 ⭐⭐
├── tools.ts                     (540 行)  - 工具转换和翻译逻辑
├── utils.ts                     (194 行)  - 实用工具函数
└── tests/                                 - 测试文件
    ├── acp-agent.test.ts
    ├── extract-lines.test.ts
    └── replace-and-calculate-location.test.ts
```

---

## 核心架构：5 层协议适配

### 第 1 层：ACP 代理实现 (`acp-agent.ts` - 820 行)

#### 类定义
```typescript
export class ClaudeAcpAgent implements Agent
```

这是项目的**心脏**，实现了 ACP 的 `Agent` 接口，定义了 6 个关键方法：

#### 1. `initialize()` - 初始化握手
```
客户端 → initialize() → 返回能力和版本信息
```

**返回的能力声明：**
- `promptCapabilities`: 支持图片和嵌入上下文
- `mcpCapabilities`: 支持 HTTP 和 SSE 连接
- `authMethods`: 登录方式

```typescript
// 代码示例
{
  protocolVersion: 1,
  agentCapabilities: {
    promptCapabilities: { image: true, embeddedContext: true },
    mcpCapabilities: { http: true, sse: true }
  },
  agentInfo: {
    name: "@zed-industries/claude-code-acp",
    title: "Claude Code",
    version: "0.9.0"
  }
}
```

#### 2. `newSession()` - 创建新会话
这是**最复杂的方法**（~70 行），完成以下工作：

**会话初始化流程：**
```
newSession(params)
    ↓
创建 sessionId (UUID v7)
    ↓
初始化 MCP 服务器配置
    ├─ 转换客户端提供的 MCP 服务器配置
    ├─ 创建 ACP MCP 服务器（提供文件/终端工具）
    └─ 创建权限 HTTP 服务器
    ↓
创建 Claude Code Query 对象
    ├─ 配置系统提示（claude_code preset）
    ├─ 配置权限模式
    └─ 配置工具白/黑名单
    ↓
保存会话状态
    ↓
返回 sessionId + 可用模型 + 权限模式
```

**关键状态结构：**
```typescript
type Session = {
  query: Query;                    // Claude Code SDK 核心
  input: Pushable<SDKUserMessage>; // 用户消息流
  cancelled: boolean;              // 取消标志
  permissionMode: PermissionMode;  // "default" | "acceptEdits" | "bypassPermissions" | "plan"
};

// Agent 维护
this.sessions[sessionId] = { query, input, cancelled, permissionMode };
```

**权限模式对比：**

| 模式 | 行为 | 使用场景 |
|------|------|---------|
| `default` | 首次使用工具时提示 | 高度控制 |
| `acceptEdits` | 自动接受文件编辑权限 | 迭代开发 |
| `plan` | 只读模式，无工具执行 | 分析和规划 |
| `bypassPermissions` | 跳过所有权限提示 | 完全自动化（非 root） |

#### 3. `prompt()` - 核心对话循环 ⭐⭐⭐

这是**最关键的方法**（~125 行），实现了 ACP-SDK 的双向消息流：

```
prompt(PromptRequest)
    ↓
step 1: 转换 ACP 格式 → SDK 格式
    input.push(promptToClaude(params))
    ↓
step 2: 启动对话循环
    while (true) {
        message = await query.next()
        switch (message.type) {
            case "system":      // init, compact_boundary, hook_response
            case "result":      // 对话结束
            case "stream_event": // 实时流
            case "user":        // 用户消息
            case "assistant":   // AI 响应
        }
    }
    ↓
step 3: 将 SDK 响应 → ACP 格式
    toAcpNotifications() / streamEventToAcpNotifications()
    ↓
step 4: 通过 sessionUpdate() 发送给客户端
    await this.client.sessionUpdate(notification)
```

**消息类型处理流程：**

```
SDK 消息类型                      → 处理逻辑
────────────────────────────────────────────────
system/init                       → 忽略
system/compact_boundary           → 忽略
system/hook_response              → TODO（标准化处理）
result/success                    → 返回 stopReason: "end_turn"
result/error_*                    → 返回错误 stopReason
stream_event/*                    → 转换为 ACP 通知
assistant/user 消息               → 提取内容块转换格式
```

**文本流处理（streaming）：**
```
SDK 返回 SDKPartialAssistantMessage
    ↓
解析 content_block_start/content_block_delta
    ↓
转换为 ACP agent_message_chunk 通知
    ↓
实时发送给客户端（支持文本、图片、思考块）
```

#### 4. `cancel()` - 取消操作
```typescript
async cancel(params: CancelNotification): Promise<void> {
  this.sessions[params.sessionId].cancelled = true;
  await this.sessions[params.sessionId].query.interrupt();
}
```

#### 5. `setSessionModel()` - 切换模型
将模型 ID 转发给 Claude Code SDK。

#### 6. `setSessionMode()` - 改变权限模式
```typescript
switch (params.modeId) {
  case "default":
  case "acceptEdits":
  case "bypassPermissions":
  case "plan":
    await query.setPermissionMode(params.modeId);
}
```

---

### 第 2 层：MCP 服务器和工具 (`mcp-server.ts` - 928 行)

#### 架构
```
Claude Code SDK
    ↓
MCP 服务器（this project）
    ├─ createMcpServer()           // SDK 直接实例化
    └─ createPermissionMcpServer() // HTTP 服务器（权限管理）
```

#### 工具命名约定
```typescript
const SERVER_PREFIX = "mcp__acp__";

toolNames = {
  read:       "mcp__acp__Read",      // 文件读取
  edit:       "mcp__acp__Edit",      // 文件编辑
  write:      "mcp__acp__Write",     // 文件写入
  bash:       "mcp__acp__Bash",      // 执行命令
  killShell:  "mcp__acp__KillShell", // 杀死进程
  bashOutput: "mcp__acp__BashOutput" // 获取终端输出
};
```

**为什么使用前缀？**
- 避免与 Claude Code 内置工具冲突
- 允许同时存在多个工具实现
- 明确指示哪些工具由 ACP 提供

#### Tool 1: Read （文件读取）

```typescript
registerTool("Read", {
  description: "读取项目中的给定文件内容",
  inputSchema: {
    file_path: string,     // 绝对路径
    offset?: number,       // 起始行（默认 1）
    limit?: number         // 读取行数（默认 2000）
  }
})

// 实现逻辑
async (input) => {
  1. 获取会话
  2. 调用 agent.readTextFile()
     └─ 这反过来调用 ACP 客户端的读取能力
  3. 应用字节限制（最大 50KB）
  4. 返回内容 + 元数据
     └─ 如果被截断：包含续读信息
}
```

**关键机制：字节限制**
```typescript
result = extractLinesWithByteLimit(content, 50000)
// 返回:
// {
//   content: "file content...",
//   wasLimited: boolean,
//   linesRead: number
// }
```

#### Tool 2: Write （文件写入）

```typescript
registerTool("Write", {
  inputSchema: {
    file_path: string,
    content: string
  }
})

// 约束：
// - 必须首先读取文件（缓存检查）
// - 总是覆盖整个文件
// - 存储在 fileContentCache[path]
```

#### Tool 3: Edit （精确替换）

```typescript
registerTool("Edit", {
  inputSchema: {
    file_path: string,
    old_string: string,    // 要替换的文本
    new_string: string,    // 替换为
    replace_all?: boolean  // 全局替换
  }
})

// 关键步骤：
1. 读取当前文件内容
2. 使用 replaceAndCalculateLocation() 执行替换
   ├─ 验证 old_string 唯一性（若不则报错）
   ├─ 计算受影响的行号
   └─ 返回 { newContent, lineNumbers }
3. 生成 diff 补丁
4. 写入文件
5. 返回 patch 给客户端
```

#### Tool 4: Bash （命令执行）

```typescript
registerTool("Bash", {
  inputSchema: {
    command: string,
    timeout?: number = 120000,
    description?: string,
    run_in_background?: boolean
  }
})

// 执行流程：
if (run_in_background) {
  // 创建后台终端
  1. handle = await client.createTerminal({ command, ... })
  2. agent.backgroundTerminals[handle.id] = {
       handle,
       status: "started",
       lastOutput: null
     }
  3. 启动 Promise.race():
     ├─ handle.waitForExit()    // 正常退出
     ├─ abortPromise            // 取消信号
     └─ sleep(timeout)          // 超时
  4. 返回 { shellId: handle.id }
} else {
  // 前台执行（简化）
  // 等待完成后返回结果
}
```

**后台终端状态机：**
```
BackgroundTerminal =
  | { handle, status: "started", lastOutput }     // 运行中
  | { status: "exited"|"aborted"|"killed"|"timedOut", pendingOutput }  // 已完成
```

#### Tool 5 & 6: BashOutput + KillShell

```typescript
// BashOutput - 获取后台进程输出
registerTool("BashOutput", {
  inputSchema: { bash_id: string }
})

// KillShell - 杀死后台进程
registerTool("KillShell", {
  inputSchema: { shell_id: string }
})
```

---

### 第 3 层：工具转换 (`tools.ts` - 540 行)

这一层负责**将工具信息转换为 ACP 格式**。

#### 关键函数

```typescript
export function toolInfoFromToolUse(
  toolUse: any,
  cachedFileContent: { [key: string]: string }
): ToolInfo {
  // 将 SDK 工具使用 → ACP ToolCallContent

  switch (name) {
    case "Task":           // 代理自主任务
    case "NotebookEdit":   // Jupyter 编辑
    case "Bash":           // 命令执行
    case "BashOutput":     // 获取输出
    case toolNames.read:   // 读取文件
    case toolNames.edit:   // 编辑文件
    case toolNames.write:  // 写入文件
    case "Glob":           // 文件搜索
    case "Grep":           // 内容搜索
    // ...
  }

  return ToolInfo {
    title: string;                    // 显示标题
    kind: "think"|"read"|"edit"|...;  // 工具分类
    content: ToolCallContent[];       // 包含的内容
    locations?: ToolCallLocation[];   // 涉及的文件位置
  }
}
```

**ToolCallContent 类型示例：**
```typescript
// 文本内容
{ type: "content", content: { type: "text", text: "..." } }

// 差异展示
{ type: "diff", path: "file.ts", oldText: "a", newText: "b" }

// 终端
{ type: "terminal", terminalId: "..." }
```

#### 工具类型映射示例

| SDK 工具 | ACP 类型 | 标题模板 |
|---------|---------|--------|
| Task | think | 输入的 description |
| Read | read | Read file.ts (1-100) |
| Edit | edit | Edit `file.ts` |
| Write | edit | Write file.ts |
| Bash | execute | `npm install` |
| BashOutput | execute | Tail Logs |

---

### 第 4 层：工具结果转换

```typescript
export function toolUpdateFromToolResult(
  result: ToolResultBlockParam | ...,
  toolUse: any
): ToolUpdate {
  // 将工具结果转换为 ACP 格式
  // 处理错误、输出内容等
}

// 返回值：
{
  title?: string;           // 可选的标题更新
  content?: ToolCallContent[];  // 结果内容
  locations?: ToolCallLocation[];  // 位置更新
}
```

---

### 第 5 层：工具结果 → Plan 映射

```typescript
export function planEntries(
  data: { todos: ClaudePlanEntry[] }
): PlanEntry[] {
  // 当 Claude 使用 TodoWrite 工具时
  // 将 todos 转换为 ACP plan 格式
}

export type ClaudePlanEntry = {
  content: string;       // 任务描述
  activeForm: string;    // 进行中时的文本
  status: "pending"|"in_progress"|"completed";
};
```

---

## 消息流详解

### 场景 1：用户发起提示

```
客户端
  │
  ├─ prompt(PromptRequest)
  │  └─ content: [
  │      { type: "text", text: "Write a function..." },
  │      { type: "image", data: "...", mimeType: "..." },
  │      { type: "resource", resource: { uri, text } }
  │    ]
  │
  └─→ ACP Agent.prompt()
      │
      ├─ promptToClaude() 转换
      │  └─ 返回 SDKUserMessage
      │      {
      │        type: "user",
      │        message: {
      │          role: "user",
      │          content: [
      │            { type: "text", text: "..." },
      │            { type: "image", source: { type: "base64"|"url", ... } }
      │          ]
      │        }
      │      }
      │
      └─→ query.next() 处理
          │
          └─→ AI 生成响应
```

### 场景 2：工具调用和结果

```
SDK/Claude 决定调用工具
  │
  ├─ 返回 tool_use 块
  │  {
  │    type: "tool_use",
  │    id: "...",
  │    name: "mcp__acp__Read",
  │    input: { file_path: "..." }
  │  }
  │
  └─→ toAcpNotifications() 转换
      │
      ├─ toolUseCache["<id>"] = tool_use
      │
      └─ 返回 SessionNotification:
          {
            sessionId,
            update: {
              sessionUpdate: "tool_call",
              toolCallId: "<id>",
              rawInput: { file_path: "..." },
              status: "pending",
              title: "Read file.ts",
              kind: "read",
              ...
            }
          }

客户端收到通知，显示工具调用
  │
  └─→ 用户审查 (可选)
      │
      └─→ tool_result 返回
          {
            type: "tool_result",
            tool_use_id: "<id>",
            content: "file content..."
          }

SDK 继续处理
  │
  └─→ 可能生成新的工具调用或最终响应
```

### 场景 3：实时流响应

```
SDK 启用 includePartialMessages: true
  │
  └─→ 返回 SDKPartialAssistantMessage
      {
        type: "stream_event",
        event: {
          type: "content_block_start",
          content_block: {
            type: "text",
            text: ""
          }
        }
      }

streamEventToAcpNotifications() 处理
  │
  ├─ content_block_start
  │  └─ 返回初始块通知
  │
  ├─ content_block_delta
  │  └─ 文本流更新，如：
  │     {
  │       type: "text_delta",
  │       text: "Hello"
  │     }
  │
  └─ message_stop
     └─ 结束

客户端实时显示文本流 ✨
```

---

## 权限系统详解

### 权限检查流程

```
Tool 被调用
  │
  ├─ 如果 permissionMode == "bypassPermissions"
  │  └─ 直接执行 (跳过权限检查)
  │
  ├─ 如果 permissionMode == "plan"
  │  └─ 拒绝执行 (只读)
  │
  ├─ 如果 permissionMode == "default"
  │  └─ 显示权限提示
  │      │
  │      ├─ 用户批准
  │      │  └─ 执行并缓存决策
  │      │
  │      └─ 用户拒绝
  │         └─ 返回错误
  │
  └─ 如果 permissionMode == "acceptEdits"
     └─ 自动批准文件编辑，其他工具仍提示
```

### Root 用户处理

```typescript
const IS_ROOT = (process.geteuid?.() ?? process.getuid?.()) === 0;

// 在非 root 环境才提供 bypassPermissions
if (!IS_ROOT) {
  availableModes.push({
    id: "bypassPermissions",
    name: "Bypass Permissions",
    description: "跳过所有权限提示"
  });
}

// 原因：Claude Code 在 root 模式下无法启动
```

---

## 实用工具层 (`utils.ts`)

### 1. Pushable<T> - 异步迭代队列

```typescript
export class Pushable<T> implements AsyncIterable<T> {
  // 目的：连接推送式 API 和异步迭代器消费者

  push(item: T) {
    if (resolvers.length > 0) {
      // 有等待的消费者
      resolve({ value: item, done: false });
    } else {
      // 缓冲项目
      queue.push(item);
    }
  }

  [Symbol.asyncIterator]() {
    // 允许 for-await-of 消费
  }
}

// 使用场景：
const input = new Pushable<SDKUserMessage>();
// 推送用户消息
input.push({ type: "user", message: {...} });
// SDK query 迭代消费
for await (const msg of input) { ... }
```

### 2. 流转换器

```typescript
// Node.js Stream → Web Stream 适配
nodeToWebWritable(process.stdout)  // Node Writable → Web WritableStream
nodeToWebReadable(process.stdin)   // Node Readable → Web ReadableStream

// 为什么需要？
// - ACP SDK 使用 Web Streams API
// - Node.js 使用传统流 API
// - ndJsonStream() 需要 Web Streams
```

### 3. 文件内容限制

```typescript
export function extractLinesWithByteLimit(
  fullContent: string,
  maxContentLength: number
): ExtractLinesResult {
  // 逐行遍历，维护字节计数
  // 防止超过限制

  return {
    content: "...",      // 截断的内容
    wasLimited: boolean, // 是否被截断
    linesRead: number    // 实际行数
  };
}

// 约束：
// - 最多 50KB (UTF-16 代码单元)
// - 最多 2000 行（默认）
// - 支持分页读取
```

### 4. 环境设置加载

```typescript
export interface ManagedSettings {
  permissions?: {
    allow?: string[];   // 允许的工具
    deny?: string[];    // 拒绝的工具
  };
  env?: Record<string, string>;
}

// 从平台特定路径加载
// macOS:   ~/Library/Application Support/ClaudeCode/managed-settings.json
// Linux:   /etc/claude-code/managed-settings.json
// Windows: C:\ProgramData\ClaudeCode\managed-settings.json
```

---

## 数据缓存策略

### 工具使用缓存

```typescript
toolUseCache: {
  [toolUseId]: {
    type: "tool_use" | "server_tool_use" | "mcp_tool_use",
    id: string,
    name: string,
    input: any
  }
}

// 作用：
// - 当工具结果返回时，关联到原始工具调用
// - 用于生成准确的 ACP 通知
```

### 文件内容缓存

```typescript
fileContentCache: {
  [filePath]: string  // 文件最新内容
}

// 作用：
// - Edit 工具需要快速查询文件内容
// - 避免重复读取网络文件
// - 计算编辑位置（toolInfoFromToolUse）
```

**缓存更新时机：**
```
readTextFile()  → fileContentCache[path] 不更新 (参数限制)
readTextFile()  → fileContentCache[path] 更新 (完整读取)
writeTextFile() → fileContentCache[path] = content (同步更新)
```

---

## 命令处理

### Slash Commands 过滤

```typescript
async function getAvailableSlashCommands(query: Query): Promise<AvailableCommand[]> {
  const commands = await query.supportedCommands();

  const UNSUPPORTED_COMMANDS = [
    "context",
    "cost",
    "login",
    "logout",
    "output-style:new",
    "release-notes",
    "todos"
  ];

  // 过滤原因：
  // - "login"/"logout": ACP 有自己的认证流程
  // - "todos": 需要特殊处理（转换为 plan）
  // - "output-style": 不相关
  // - "context"/"cost": 在 ACP 中无意义
}
```

### MCP 命令支持

```
用户输入: /mcp:server:command args
  │
  └─→ promptToClaude() 转换
      │
      ├─ 识别模式: /mcp:([^:\s]+):(\S+)(\s+.*)?$/
      │
      └─ 转换为: /server:command (MCP) args
         (SDK 可识别的格式)
```

---

## 特殊情况处理

### 1. 登录要求检测

```typescript
if (message.result.includes("Please run /login")) {
  throw RequestError.authRequired();
}
```

**原因：** Claude Code SDK 会在认证失效时返回这个消息，需要向客户端报告认证要求。

### 2. 合成消息过滤

```typescript
if (message.type === "user" && typeof message.message.content === "string") {
  // SDK 生成的纯文本用户消息，不转发给客户端
  break;
}
```

### 3. 本地命令输出处理

```typescript
if (message.message.content.includes("<local-command-stdout>")) {
  console.log(...);  // 输出到 stderr (不中断 ACP)
  break;
}
```

### 4. 思考块提取

```typescript
// 流式消息处理
case "thinking":
case "thinking_delta":
  update = {
    sessionUpdate: "agent_thought_chunk",
    content: { type: "text", text: chunk.thinking }
  };
```

### 5. 索引偏移转换

```
ACP 索引（0-based） ← → SDK 索引（1-based）

示例：
input.offset = 5      // ACP: 第 5 行起
→ line: 5             // SDK: 传递 5
input.offset = 0      // ACP: 从开始
→ line: undefined     // SDK: 省略
```

---

## 启动流程详解

### 1. 入口点 (`index.ts`)

```typescript
#!/usr/bin/env node

// Step 1: 加载托管设置
const managedSettings = loadManagedSettings();
if (managedSettings) {
  applyEnvironmentSettings(managedSettings);
}

// Step 2: 重定向 console (避免干扰 ACP)
console.log = console.error;
console.info = console.error;
// ... 其他重定向

// Step 3: 处理未捕获的异常
process.on("unhandledRejection", (reason, promise) => {
  console.error(...);
});

// Step 4: 启动 ACP
import { runAcp } from "./acp-agent.js";
runAcp();

// Step 5: 保持进程活动
process.stdin.resume();
```

### 2. ACP 初始化 (`runAcp()`)

```typescript
export function runAcp() {
  // Web Stream 适配
  const input = nodeToWebWritable(process.stdout);
  const output = nodeToWebReadable(process.stdin);

  // ndjson 流处理
  const stream = ndJsonStream(input, output);

  // 创建 ACP 代理
  new AgentSideConnection(
    (client) => new ClaudeAcpAgent(client),
    stream
  );
}
```

**协议初始化（交互）：**
```
客户端                       服务器
  │                            │
  ├─ initialize request ───────→ initialize()
  │                            │
  ←─── initialize response ──── ┤
  │                            │
  ├─ newSession request ───────→ newSession()
  │                            │
  ←─── newSession response ──── ┤
  │                            │
  ├─ prompt request ──────────→ prompt()
  │                            │
  ←─── sessionUpdate ────────── ┤ (多个)
  │                            │
  ├─ prompt response ←────────── ┤
```

---

## 库使用方式 (`lib.ts`)

除了 CLI，还可以作为库导入：

```typescript
// 导出主要组件
export { ClaudeAcpAgent, runAcp, toAcpNotifications, ... } from "./acp-agent.js";

// 在其他 Node.js 应用中使用
import { ClaudeAcpAgent, runAcp } from "@zed-industries/claude-code-acp";

// 自定义启动
const agentSideConnection = new AgentSideConnection(
  (client) => new ClaudeAcpAgent(client),
  customStream
);
```

---

## 测试覆盖 (`src/tests/`)

### 1. `acp-agent.test.ts` - 核心代理测试
- 会话管理
- 消息转换
- 权限模式切换

### 2. `extract-lines.test.ts` - 文件读取限制
- 字节限制逻辑
- 行号计算
- UTF-16 编码处理

### 3. `replace-and-calculate-location.test.ts` - 编辑位置计算
- 字符串替换准确性
- 受影响行号追踪
- 正则表达式支持

**运行测试：**
```bash
npm test                    # 监视模式
npm run test:run           # 单次运行
npm run test:coverage      # 覆盖率报告
RUN_INTEGRATION_TESTS=true npm run test:integration
```

---

## 性能考虑

### 1. 流式处理
- 启用了 `includePartialMessages: true`
- 避免等待完整响应
- 支持实时 UI 更新

### 2. 缓存策略
- 工具使用缓存避免重复查询
- 文件内容缓存加速编辑

### 3. 大文件处理
- 50KB 字节限制防止过度加载
- 支持分页读取（offset/limit）
- 行号计数优化

### 4. 并发会话
- 每个会话独立的 Query 对象
- sessionId 隔离状态
- 后台终端独立管理

---

## 安全考虑

### 1. 权限模式隔离
```
plan 模式        ← 纯分析，无副作用
default 模式     ← 每个工具都要确认
acceptEdits 模式 ← 自动编辑，但不执行命令
bypassPermissions ← 完全自动化（谨慎使用）
```

### 2. Root 用户保护
- `bypassPermissions` 在 root 禁用
- 防止 Claude Code 意外崩溃

### 3. 文件内容验证
- Edit 工具强制要求先读取文件
- 防止盲目编辑

### 4. 恶意代码检测提示
```typescript
export const SYSTEM_REMINDER = `
<system-reminder>
Whenever you read a file, you should consider whether it looks malicious.
If it does, you MUST refuse to improve or augment the code.
</system-reminder>
`;
```

---

## 扩展点

### 1. 自定义工具
```typescript
// 在 createMcpServer() 中添加
server.registerTool("CustomTool", {
  inputSchema: {...},
  handler: async (input) => {...}
});
```

### 2. 自定义权限提示
```typescript
// createPermissionMcpServer() 中实现自定义逻辑
```

### 3. 额外的 MCP 服务器
```typescript
// newSession() 中配置
mcpServers["custom"] = {
  type: "stdio"|"http",
  command?: "...",
  url?: "..."
};
```

---

## 常见问题

### Q: 为什么需要 ACP 适配层？
A: ACP 是标准协议，而 Claude Code SDK 是专有 API。适配层让任何 ACP 客户端都能使用 Claude Code。

### Q: 工具缓存的目的是什么？
A: 当工具返回结果时，需要知道是哪个工具调用。缓存维护 id → 工具信息的映射。

### Q: 为什么文件读取有字节限制？
A: 防止 API 超载、内存溢出，以及保持响应时间。大文件可以分页读取。

### Q: Root 用户为何不能使用 bypassPermissions？
A: 因为 Claude Code 在 root 模式下启动会失败。这是 SDK 的限制。

### Q: 流式消息如何工作？
A: SDK 在生成响应时实时发出流事件，适配层立即转换为 ACP 通知发送给客户端。

---

## 总结

**claude-code-acp** 是一个精心设计的协议适配器，通过 5 层架构实现了：

1. **ACP 代理实现** - 完整的协议处理
2. **MCP 工具集合** - 文件和终端操作
3. **工具转换** - SDK ↔ ACP 格式转换
4. **权限管理** - 灵活的权限模式
5. **工具函数库** - 流处理、缓存、限制

核心强度：
- ✅ 清晰的分层架构
- ✅ 完善的错误处理
- ✅ 灵活的权限系统
- ✅ 高效的性能优化
- ✅ 广泛的协议支持

对于想要深入了解 AI 代理架构的开发者来说，这是一个极好的参考项目。
