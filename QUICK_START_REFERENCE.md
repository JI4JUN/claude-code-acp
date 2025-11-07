# 🎯 快速参考 - 5 分钟速览

## 你的学习目标

在 Electron/Tauri 聊天客户端中实现类似 Claude Code 的代码编辑功能。

参考项目：**claude-code-acp** (一个 ACP 适配器，实现了相同的功能)

---

## 核心架构 (3 层)

```
┌──────────────────────────────────┐
│    你的聊天 UI (React/Vue)       │  ← 显示消息、工具调用、权限提示
├──────────────────────────────────┤
│  适配层 (你要实现的)             │  ← 消息转换、权限管理、工具执行
│  - MessageTransformer            │
│  - ToolManager                   │
│  - PermissionManager             │
├──────────────────────────────────┤
│  Claude Code SDK                 │  ← 对话驱动、流式响应、工具调用
└──────────────────────────────────┘
```

---

## 最关键的 3 个概念

### 1️⃣ Query 迭代循环 ⭐⭐⭐

这是系统的**核心驱动机制**。

```typescript
// claude-code-acp 中的实现 (acp-agent.ts 第 324-437 行)
while (true) {
  const { value: message, done } = await query.next();

  if (done) break;

  // 处理 6 种消息类型
  switch (message.type) {
    case 'system':        // 初始化消息
    case 'result':        // 对话结束
    case 'stream_event':  // 实时流
    case 'user':          // 用户消息
    case 'assistant':     // AI 响应
  }
}
```

你需要实现的：
- 将用户输入推入 query
- 循环处理 query 返回的消息
- 将消息转换为 UI 能显示的格式

### 2️⃣ 工具执行流程 ⭐⭐

工具是 Claude 修改代码的方式。

```
user → "修改 src/main.ts" →
  ↓
Claude → tool_use (Edit) →
  ↓
你的工具执行 →
  ↓
tool_result → Claude 继续 →
  ↓
对话结束
```

6 个工具：
- **Read** - 读文件
- **Write** - 写文件
- **Edit** - 编辑文件（最复杂）
- **Bash** - 执行命令
- **BashOutput** - 获取输出
- **KillShell** - 杀进程

### 3️⃣ 权限管理 ⭐

保护用户不让 Claude 随意修改文件。

```
4 种模式：
- default         ← 每个工具都问用户
- acceptEdits     ← 自动接受编辑
- plan            ← 只读模式
- bypassPermissions ← 完全自动（谨慎）
```

---

## 文件对应关系

当你实现时，参考这些映射：

| 你要实现 | 参考 claude-code-acp | 行号 |
|--------|-------------------|------|
| 会话管理 | acp-agent.ts | 60-108 |
| 消息转换 | acp-agent.ts | 561-812 |
| Query 循环 | acp-agent.ts | 324-437 |
| 权限系统 | acp-agent.ts | 205-217, 456-479 |
| 读文件工具 | mcp-server.ts | 52-157 |
| 编辑文件工具 | mcp-server.ts | 224-303 |
| 执行命令工具 | mcp-server.ts | 307-450 |
| 工具转换 | tools.ts | 全部 |

---

## 5 步实现计划

### Week 1: 学习 (5 天，每天 1-2 小时)

```
Day 1: 理解基础
  ✓ 读 CLAUDE.md
  ✓ 看 ARCHITECTURE_DIAGRAMS.md 的全局架构
  ✓ 运行 claude-code-acp

Day 2: 理解核心
  ✓ 读 PROJECT_DETAILED_ANALYSIS.md 第 1 层
  ✓ 看 Query 迭代循环图
  ✓ 理解消息流

Day 3: 理解工具
  ✓ 读 PROJECT_DETAILED_ANALYSIS.md 第 2 层
  ✓ 看工具调用流程图
  ✓ 理解权限系统

Day 4: 理解缓存
  ✓ 读 PROJECT_DETAILED_ANALYSIS.md 缓存策略
  ✓ 理解性能优化

Day 5: 完整理解
  ✓ 看 T0-T19 完整会话示例
  ✓ 回顾源代码
  ✓ 规划你的实现
```

### Week 2-3: 实现 (10 天)

```
Day 6-7: 基础框架
  ✓ 创建项目结构
  ✓ 实现 Session 类型
  ✓ 实现 ClaudeCodeAdapter (核心)

Day 8-9: 工具实现
  ✓ 实现 6 个工具
  ✓ 实现文件缓存
  ✓ 实现权限管理

Day 10-15: UI 集成
  ✓ 聊天界面集成
  ✓ 代码编辑面板
  ✓ 权限提示 UI
  ✓ 测试和调试
```

---

## 代码框架（复制粘贴起点）

### 最小可行的 Session 管理

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

class SessionManager {
  private sessions = new Map();

  async createSession(projectPath: string) {
    const sessionId = `session_${Date.now()}`;

    // 创建输入队列
    const inputQueue = this.createAsyncQueue();

    // 创建 Query 对象（核心！）
    const q = query({
      prompt: inputQueue,
      options: {
        cwd: projectPath,
        includePartialMessages: true,
        systemPrompt: { type: 'preset', preset: 'claude_code' }
      }
    });

    // 保存会话
    this.sessions.set(sessionId, { q, inputQueue });

    return sessionId;
  }

  async sendMessage(sessionId: string, userText: string) {
    const session = this.sessions.get(sessionId);

    // 推入用户消息
    session.inputQueue.push({
      type: 'user',
      message: {
        role: 'user',
        content: [{ type: 'text', text: userText }]
      }
    });

    // 处理响应（核心循环！）
    while (true) {
      const { value: message, done } = await session.q.next();
      if (done) break;

      // 对每个消息类型做出反应
      this.handleMessage(message);
    }
  }

  private handleMessage(message: any) {
    switch (message.type) {
      case 'stream_event':
        // 实时显示流式响应
        break;
      case 'assistant':
        // 处理 AI 消息
        const blocks = message.message.content;
        for (const block of blocks) {
          if (block.type === 'text') {
            // 显示文本
          }
          if (block.type === 'tool_use') {
            // 显示工具调用，执行工具
            this.executeTool(block);
          }
        }
        break;
      case 'result':
        // 对话结束
        break;
    }
  }

  private async executeTool(toolUse: any) {
    const { name, id, input } = toolUse;

    let result: any;

    switch (name) {
      case 'Read':
        // 读文件：input.file_path
        break;
      case 'Write':
        // 写文件：input.file_path, input.content
        break;
      case 'Edit':
        // 编辑文件：input.file_path, input.old_string, input.new_string
        break;
      case 'Bash':
        // 执行命令：input.command
        break;
    }

    // SDK 会自动将结果发回，但你需要推入响应
    // 这取决于 SDK 的设计
  }

  private createAsyncQueue() {
    const queue = [];
    const resolvers = [];

    return {
      push: (item: any) => {
        if (resolvers.length > 0) {
          resolvers.shift()({ value: item, done: false });
        } else {
          queue.push(item);
        }
      },
      [Symbol.asyncIterator]: () => ({
        next: () => {
          if (queue.length > 0) {
            return Promise.resolve({ value: queue.shift(), done: false });
          }
          return new Promise(resolve => {
            resolvers.push(resolve);
          });
        }
      })
    };
  }
}
```

---

## 关键文件速查

### claude-code-acp 中最重要的代码位置

| 功能 | 文件 | 行数 | 必读？ |
|------|------|------|--------|
| 核心代理类 | acp-agent.ts | 93-494 | ⭐⭐⭐ |
| 消息循环 | acp-agent.ts | 324-437 | ⭐⭐⭐ |
| 工具定义 | mcp-server.ts | 43-600 | ⭐⭐⭐ |
| Read 工具 | mcp-server.ts | 52-157 | ⭐⭐ |
| Edit 工具 | mcp-server.ts | 224-303 | ⭐⭐⭐ |
| 权限系统 | acp-agent.ts | 205-217 | ⭐⭐ |
| 工具转换 | tools.ts | 26-200 | ⭐⭐ |
| 异步队列 | utils.ts | 9-47 | ⭐ |

---

## 最常见的错误

❌ **错误 1**: 不理解 Query 循环如何工作
→ 反复阅读 acp-agent.ts 第 324 行的 while 循环

❌ **错误 2**: 不知道工具如何返回结果给 Claude
→ 研究 tool_use 和 tool_result 的交互

❌ **错误 3**: 权限检查实现不当
→ 参考 acp-agent.ts 第 205-217 行的权限检查点

❌ **错误 4**: 文件缓存一致性问题
→ 理解何时更新缓存（ARCHITECTURE_DIAGRAMS.md）

❌ **错误 5**: UI 没有反应权限模式
→ 实现权限提示 UI，让用户能选择权限模式

---

## 调试技巧

### 1. 添加日志跟踪消息流

```typescript
console.log('📩 Received message:', message.type, message.subtype);
```

### 2. 在 Claude Code SDK 中启用调试

```typescript
const options = {
  // ...
  stderr: (err) => console.error('SDK Error:', err)
};
```

### 3. 使用浏览器 DevTools

- Network 标签查看 API 调用
- Console 查看日志
- 在 Session 创建时设置断点

### 4. 与 claude-code-acp 对比

- 运行 claude-code-acp 作为参考实现
- 在相同的输入下对比行为
- 使用 strace 或类似工具查看系统调用

---

## 资源快速链接

📚 **你已经有的文档** (在 claude-code-acp 目录)
- DOCUMENTATION_INDEX.md - 入口
- CLAUDE.md - 快速参考
- PROJECT_DETAILED_ANALYSIS.md - 深度分析
- ARCHITECTURE_DIAGRAMS.md - 流程图
- LEARNING_GUIDE_FOR_YOUR_PROJECT.md - 学习计划

📖 **官方文档**
- https://docs.anthropic.com - Claude Code SDK
- https://agentclientprotocol.com - ACP 规范

🔗 **相关项目**
- github.com/zed-industries/claude-code-acp - 参考实现

---

## 今天就开始！

```bash
# Step 1: 阅读文档
# 从 DOCUMENTATION_INDEX.md 开始 (5 分钟)

# Step 2: 了解项目
cd claude-code-acp
npm run build
npm run test

# Step 3: 查看源代码
# 从 src/acp-agent.ts 开始 (重点看 prompt() 方法)

# Step 4: 规划你的实现
# 根据 IMPLEMENTATION_STARTER_CODE.md 创建框架

# Step 5: 开始编码
# 实现 SessionManager → MessageTransformer → Tools
```

---

**预期时间表**：
- 学习：5 天 (每天 1-2 小时)
- 实现：10-15 天 (每天 2-4 小时)
- **总计**：3-4 周就能有一个可用的原型

祝你成功！🚀

有问题？
- 查阅 PROJECT_DETAILED_ANALYSIS.md 的 Q&A 部分
- 查看 ARCHITECTURE_DIAGRAMS.md 的流程图
- 查找 LEARNING_GUIDE_FOR_YOUR_PROJECT.md 的相应部分
