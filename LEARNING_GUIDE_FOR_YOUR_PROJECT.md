# 🚀 学习 claude-code-acp 实现代码编辑功能 - 完整指南

## 你的项目目标

**在 Electron/Tauri 桌面应用中**，通过聊天界面**集成 Claude Code** 来修改本地项目文件。

架构愿景：
```
┌─────────────────────────────────────────┐
│       你的 AI 聊天客户端                 │
│      (Electron/Tauri)                   │
├─────────────────────────────────────────┤
│  聊天界面 + 代码编辑面板                 │
├─────────────────────────────────────────┤
│  你实现的适配层 (参考 claude-code-acp)   │
│  └─ 处理协议转换                        │
│  └─ 管理工具和权限                      │
├─────────────────────────────────────────┤
│  Claude Code SDK / Claude API            │
│  (后端 LLM 推理)                        │
└─────────────────────────────────────────┘
```

---

## 📚 推荐学习路径 (5 天集中学习)

### Day 1: 基础理解 (3 小时)

**目标**：理解 claude-code-acp 的整体架构

**学习资源：**
1. 阅读 **DOCUMENTATION_INDEX.md** (5 min)
   - 快速了解有哪些文档

2. 阅读 **CLAUDE.md** (15 min)
   - 了解项目目的和快速命令
   - 理解 5 层架构概念

3. 看 **ARCHITECTURE_DIAGRAMS.md** (45 min)
   - 全局架构图
   - 会话生命周期图
   - Prompt 处理流程
   - Query 迭代循环（最重要）

4. 运行项目 (30 min)
   ```bash
   cd claude-code-acp
   npm install
   npm run build
   npm run test
   ```

5. 浏览源代码结构 (45 min)
   ```bash
   # 快速了解代码组织
   tree src -L 2
   # 阅读关键文件的开头注释
   head -50 src/acp-agent.ts
   ```

**检查点：**
- [ ] 理解 ACP 是什么
- [ ] 理解 5 层架构
- [ ] 能运行项目
- [ ] 知道主要文件的位置

---

### Day 2: 深入核心 (4 小时)

**目标**：理解核心代理和消息流

**学习资源：**

1. 阅读 **PROJECT_DETAILED_ANALYSIS.md** - 第 1 层 (60 min)
   ```
   关键点：
   - ClaudeAcpAgent class
   - initialize() 方法
   - newSession() 方法
   - prompt() 方法 ⭐⭐⭐ (最重要)
   ```

   **重点理解：**
   - 会话如何创建和存储
   - 用户输入如何转换为 SDK 格式
   - SDK 消息如何转换回 ACP 格式
   - 错误处理和取消机制

2. 阅读 **ARCHITECTURE_DIAGRAMS.md** - Prompt 处理流程 (30 min)
   ```
   可视化理解：
   - ACP PromptRequest 的结构
   - promptToClaude() 如何转换
   - 支持的内容类型 (文本、图片、资源)
   ```

3. 阅读 **ARCHITECTURE_DIAGRAMS.md** - Query 迭代循环 (45 min)
   ```
   关键理解：
   - while (true) { await query.next() } 循环
   - 6 种消息类型的处理
   - toAcpNotifications() 的工作原理
   - 流式消息和完整消息的区别
   ```

4. 代码追踪 (45 min)
   ```bash
   # 在编辑器中打开 acp-agent.ts
   # 追踪 prompt() 方法的完整流程：
   - 第 314 行：promptToClaude()
   - 第 324 行：query.next() 循环
   - 第 332-437 行：消息处理 switch
   ```

**检查点：**
- [ ] 理解 ClaudeAcpAgent 的 6 个主要方法
- [ ] 理解消息循环如何工作
- [ ] 能画出 Prompt 的转换流程
- [ ] 理解流式响应的机制

---

### Day 3: 工具和权限 (4 小时)

**目标**：理解工具系统和权限管理

**学习资源：**

1. 阅读 **PROJECT_DETAILED_ANALYSIS.md** - 第 2 层 (60 min)
   ```
   6 个核心工具：
   - Read (读文件)
   - Write (写文件)
   - Edit (编辑文件) ⭐ 最复杂
   - Bash (执行命令)
   - BashOutput (获取输出)
   - KillShell (杀进程)
   ```

   **重点：**
   - 每个工具的输入输出格式
   - Edit 工具如何计算修改位置
   - 权限检查流程
   - 文件大小限制

2. 阅读 **PROJECT_DETAILED_ANALYSIS.md** - 权限系统 (45 min)
   ```
   4 种权限模式：
   - default (每个工具都需确认)
   - acceptEdits (自动接受编辑)
   - plan (只读模式)
   - bypassPermissions (完全自动)
   ```

   **关键问题：**
   - 如何切换权限模式
   - 权限提示如何显示
   - Root 用户的特殊处理

3. 阅读 **ARCHITECTURE_DIAGRAMS.md** - 工具调用流程 (45 min)
   ```
   流程图展示：
   - tool_use 块的生成
   - toolUseCache 如何跟踪
   - tool_result 的返回
   - 权限系统的介入点
   ```

4. 阅读 **ARCHITECTURE_DIAGRAMS.md** - 权限状态机 (30 min)
   ```
   理解：
   - 每个权限模式的决策树
   - 权限提示的时机
   - 用户交互的流程
   ```

**检查点：**
- [ ] 能列出 6 个工具及其用途
- [ ] 理解 Edit 工具的工作原理
- [ ] 理解 4 种权限模式的区别
- [ ] 知道权限检查的位置

---

### Day 4: 缓存和实用工具 (3 小时)

**目标**：理解性能优化和辅助功能

**学习资源：**

1. 阅读 **PROJECT_DETAILED_ANALYSIS.md** - 工具转换层 (45 min)
   ```
   理解：
   - toolInfoFromToolUse() 如何工作
   - toolUpdateFromToolResult() 的转换
   - planEntries() 如何处理 TodoWrite
   ```

2. 阅读 **PROJECT_DETAILED_ANALYSIS.md** - 缓存策略 (45 min)
   ```
   关键缓存：
   - toolUseCache: 跟踪工具调用
   - fileContentCache: 缓存文件内容
   - backgroundTerminals: 后台进程管理
   ```

3. 阅读 **ARCHITECTURE_DIAGRAMS.md** - 文件操作缓存 (30 min)
   ```
   可视化理解缓存的更新时机
   ```

4. 阅读 **PROJECT_DETAILED_ANALYSIS.md** - 实用工具层 (15 min)
   ```
   了解：
   - Pushable<T> 异步队列
   - 流适配器
   - 字节限制
   ```

**检查点：**
- [ ] 理解工具缓存的必要性
- [ ] 理解文件缓存的更新规则
- [ ] 知道后台进程如何管理

---

### Day 5: 完整理解和代码实验 (4 小时)

**目标**：全面理解，准备自己的实现

**学习资源：**

1. 阅读 **ARCHITECTURE_DIAGRAMS.md** - 完整会话示例 (60 min)
   ```
   T0-T19 时间线展示：
   - initialize → newSession → prompt 的完整流程
   - 每一步的状态变化
   - UI 在各个阶段的显示
   ```

   **边看边思考：**
   - 你的应用需要在哪里显示进度
   - 你需要实现哪些 UI 组件
   - 哪些状态需要保存

2. 阅读 **DOCUMENTATION_GUIDE.md** - Q&A 部分 (30 min)
   ```
   解答常见疑问，填补理解空白
   ```

3. 源代码深度阅读 (90 min)
   ```
   按以下顺序阅读完整的源文件：

   a) src/utils.ts (194 行) - 快速了解
      - Pushable<T> 实现
      - 流转换逻辑
      - 字节限制算法

   b) src/tools.ts (540 行) - 重点阅读
      - toolInfoFromToolUse() 的各种情况
      - 理解工具转换逻辑

   c) src/acp-agent.ts (820 行) - 逐行阅读
      - 特别关注 prompt() 方法
      - 理解每个 switch 分支

   d) src/mcp-server.ts (928 行) - 理解工具实现
      - 每个 registerTool() 的完整逻辑
      - 权限检查的实现
   ```

4. 实验项目 (30 min)
   ```bash
   # 在本地运行 claude-code-acp
   # 尝试修改一些参数，看看效果
   # 例如：改变权限模式、改变文件大小限制等
   npm run dev
   ```

**检查点：**
- [ ] 能完整描述一个会话的全生命周期
- [ ] 理解系统中的每个关键组件
- [ ] 知道要实现什么和如何实现

---

## 🎯 关键学习点总结

### 必须深入理解

1. **Query 迭代模式**
   - 这是整个系统的核心驱动机制
   - 位置：`acp-agent.ts` 第 324 行 `while (true)`
   - 为什么？异步生成器如何驱动对话流

2. **消息转换流程**
   - ACP → SDK → Claude → SDK → ACP
   - 位置：`acp-agent.ts` 第 561、644 行
   - 涉及的函数：`promptToClaude()`, `toAcpNotifications()`, `streamEventToAcpNotifications()`

3. **权限检查机制**
   - 4 种模式如何工作
   - 位置：`acp-agent.ts` 第 205-217 行, `mcp-server.ts` 第 181-189 行
   - 为什么？安全和用户体验

4. **工具执行流程**
   - tool_use → 权限检查 → 执行 → tool_result
   - 位置：`mcp-server.ts` 工具 handlers
   - 为什么？这是实现代码修改的核心

5. **缓存策略**
   - 什么时候更新，什么时候查询
   - 位置：`acp-agent.ts` 中的三个缓存
   - 为什么？性能优化和一致性

### 应该了解

6. **MCP 服务器架构**
7. **流式响应处理**
8. **后台终端管理**
9. **启动流程**
10. **错误处理**

### 参考即可

11. **测试用例**
12. **边界情况处理**
13. **根用户特殊处理**

---

## 💻 在你的项目中实现

### 第一阶段：基础框架 (1 周)

```typescript
// 你的项目结构（参考 claude-code-acp）
src/
├── adapters/
│  ├── ElectronAdapter.ts      // 替代 acp-agent.ts (Electron 特定)
│  ├── MessageHandler.ts        // 消息转换 (参考 acp-agent.ts)
│  └── ToolManager.ts           // 工具管理 (参考 mcp-server.ts)
├── tools/
│  ├── FileEditor.ts            // Read/Write/Edit
│  ├── CommandRunner.ts         // Bash/BashOutput/KillShell
│  └── ToolRegistry.ts          // 工具注册
├── permissions/
│  ├── PermissionManager.ts     // 权限控制
│  └── PermissionUI.ts          // 权限提示 UI
└── core/
   ├── SessionManager.ts         // 会话管理
   └── Messaging.ts              // 消息处理
```

### 第二阶段：核心实现 (2 周)

1. **实现 SessionManager**
   - 参考：`acp-agent.ts` 的 sessions 管理
   - 管理 sessionId、Query 对象、权限模式

2. **实现 MessageHandler**
   - 参考：`acp-agent.ts` 的 `promptToClaude()` 和 `toAcpNotifications()`
   - 处理消息的入出转换

3. **实现 6 个工具**
   - 参考：`mcp-server.ts` 的每个工具实现
   - 特别注意 Edit 工具的复杂逻辑

4. **实现权限系统**
   - 参考：`acp-agent.ts` 第 205-217 行
   - 4 种权限模式的切换

### 第三阶段：UI 和集成 (1 周)

1. **聊天界面集成**
   - 显示工具调用（title, kind, content）
   - 实时流式响应显示
   - 工具结果展示

2. **代码编辑面板**
   - 显示正在编辑的文件
   - 显示 diff 预览
   - 批准/拒绝编辑按钮

3. **权限提示 UI**
   - 显示权限请求
   - 允许用户选择权限模式

---

## 📖 具体代码参考映射

当你在实现时，直接参考这些位置：

| 你要实现什么 | 参考位置 | 文件 | 行号 |
|------------|--------|------|------|
| 会话管理 | Session type | acp-agent.ts | 60-65 |
| 用户消息转换 | promptToClaude() | acp-agent.ts | 561-638 |
| SDK 消息处理 | toAcpNotifications() | acp-agent.ts | 644-776 |
| 权限模式 | Permission modes | acp-agent.ts | 205-217 |
| Read 工具 | registerTool("Read") | mcp-server.ts | 52-157 |
| Edit 工具 | registerTool("Edit") | mcp-server.ts | 224-303 |
| Bash 工具 | registerTool("Bash") | mcp-server.ts | 307-399 |
| 工具转换 | toolInfoFromToolUse() | tools.ts | 26-... |
| 后台进程 | backgroundTerminals | acp-agent.ts | 100 |
| 缓存管理 | fileContentCache | acp-agent.ts | 99 |

---

## 🔧 实现技巧和注意事项

### 1. 消息处理的异步流程
```typescript
// 参考 acp-agent.ts 第 324 行的模式
while (true) {
  const { value: message, done } = await query.next();

  // 不要忘记处理 done
  if (done || !message) break;

  // 根据消息类型处理
  switch (message.type) {
    case "stream_event":
      // 实时流式更新
      break;
    case "tool_use":
      // 检查权限，然后执行
      break;
    case "result":
      // 对话结束
      break;
  }
}
```

### 2. 工具执行和权限检查
```typescript
// 参考 mcp-server.ts 的工具 handlers
async (input, extra) => {
  // 1. 检查会话存在
  const session = agent.sessions[sessionId];
  if (!session) return error();

  // 2. 检查权限（在 SDK 中处理，你需要代理权限请求）
  const permissionResult = await requestPermission(input);
  if (!permissionResult.approved) return error();

  // 3. 执行工具
  const result = await executeTool(input);

  // 4. 返回结果
  return { content: [{ type: "text", text: result }] };
}
```

### 3. 缓存更新策略
```typescript
// 参考 acp-agent.ts 的缓存逻辑
// Read（部分）- 不更新缓存
readTextFile({ path, line, limit })

// Read（完整）- 更新缓存
fileContentCache[path] = content;

// Edit - 自动更新缓存
fileContentCache[path] = newContent;

// Write - 同步更新缓存
fileContentCache[path] = content;
```

### 4. 权限系统的实现
```typescript
// 参考 acp-agent.ts 第 205-217 行
const permissionMode = "default"; // 或其他模式

if (permissionMode === "plan") {
  // 拒绝所有工具执行
  return { error: "Plan mode is read-only" };
}

if (permissionMode === "bypassPermissions") {
  // 直接执行
  return await executeTool(input);
}

if (permissionMode === "acceptEdits" && isEditTool) {
  // 自动接受编辑工具
  return await executeTool(input);
}

if (permissionMode === "default") {
  // 显示权限提示，等待用户
  return await promptUser(input);
}
```

---

## 📋 学习清单

### Week 1 - 基础理解
- [ ] 阅读所有 6 份文档
- [ ] 运行 claude-code-acp 项目
- [ ] 理解 5 层架构
- [ ] 理解 6 个工具
- [ ] 理解权限系统

### Week 2 - 深入学习
- [ ] 理解消息流的每一步
- [ ] 理解工具执行流程
- [ ] 理解缓存策略
- [ ] 理解权限检查机制
- [ ] 能够手绘系统流程图

### Week 3 - 代码阅读
- [ ] 逐行阅读 acp-agent.ts
- [ ] 逐行阅读 mcp-server.ts
- [ ] 理解每个工具的实现
- [ ] 理解工具转换逻辑
- [ ] 能够解释任何一段代码

### Week 4 - 实现规划
- [ ] 确定你的技术栈（Electron/Tauri + 什么 UI 框架）
- [ ] 设计你的项目结构
- [ ] 列出需要实现的核心组件
- [ ] 创建代码框架
- [ ] 开始实现第一个工具

---

## 💡 关键问题和答案

**Q: 我需要实现 ACP 协议吗？**
A: 你的应用内部不需要完全实现 ACP（那是用来连接外部客户端的）。你只需要参考它的**消息格式和转换逻辑**。

**Q: 我需要使用 Claude Code SDK 吗？**
A: 是的，这是最重要的。SDK 提供了：
- `Query` 对象（迭代式对话）
- 工具管理
- 权限控制
- 流式响应

**Q: 我可以简化权限系统吗？**
A: 可以，但建议实现至少这两个模式：
- `default`：询问用户
- `acceptEdits`：自动接受编辑

**Q: 文件缓存真的必要吗？**
A: 是的，特别是对于 Edit 工具。它需要原始内容来计算修改位置。

**Q: 后台终端支持重要吗？**
A: 如果你要支持长运行的命令（npm install、git clone 等），是重要的。否则可以先简化。

---

## 🎓 学习成果期望

完成这个学习计划后，你应该能够：

✅ 解释 claude-code-acp 的整个工作原理
✅ 在你的项目中实现类似的功能
✅ 设计和实现工具系统
✅ 实现权限管理
✅ 处理流式响应
✅ 调试协议级别的问题
✅ 优化性能（缓存、流式处理）
✅ 扩展到新的工具和功能

---

## 📞 实施过程中遇到问题时

1. **回到文档查询**
   - DOCUMENTATION_GUIDE.md 的"按任务查询"
   - PROJECT_DETAILED_ANALYSIS.md 的 Q&A

2. **查看源代码**
   - 根据表格定位到具体位置
   - 阅读代码注释

3. **运行和调试**
   - 在 claude-code-acp 项目中进行小实验
   - 添加日志了解流程

4. **参考官方文档**
   - Claude Code SDK: https://docs.anthropic.com
   - ACP: https://agentclientprotocol.com

---

## 🚀 下一步行动

1. **今天**
   - [ ] 阅读这个指南
   - [ ] 开始 Day 1 的学习

2. **这周**
   - [ ] 完成 Day 1-3 的学习
   - [ ] 运行项目实验

3. **下周**
   - [ ] 完成 Day 4-5 的学习
   - [ ] 规划你的项目实现

4. **第三周开始**
   - [ ] 开始实现你的项目

祝你学习和开发成功！🎓🚀
