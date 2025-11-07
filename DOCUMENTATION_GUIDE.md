# 📚 claude-code-acp 学习指南

本文档集合为 **claude-code-acp** 项目提供了完整的学习材料。选择适合你的学习路径开始吧！

---

## 📖 文档概览

### 1. **CLAUDE.md** (⭐ 开发者必读)
**文件大小：** ~7KB | **阅读时间：** 10 分钟

**适合人群：** 想要在项目中工作的开发者

**内容：**
- 项目概述
- 常用命令（build, lint, test）
- 5 层架构简要说明
- 会话和状态管理
- 权限和工具处理
- 实现细节提示

**何时阅读：**
- ✅ 刚加入项目时
- ✅ 要运行构建或测试时
- ✅ 需要快速架构参考时

**快速链接：**
```bash
# 常用命令（摘自 CLAUDE.md）
npm run build              # 编译 TypeScript
npm run dev                # 开发模式
npm run lint && npm run format:check  # 检查代码质量
npm run test               # 运行测试
```

---

### 2. **PROJECT_DETAILED_ANALYSIS.md** (⭐⭐ 深度学习)
**文件大小：** ~26KB | **阅读时间：** 45-60 分钟

**适合人群：** 想深入理解项目架构的开发者、系统设计师

**内容：**
- 完整项目概述（目标、用途）
- 5 层架构详细解析
  - Layer 1: ACP 代理核心（820 行）
  - Layer 2: MCP 服务器（928 行）
  - Layer 3: 工具转换（540 行）
  - Layer 4: 工具结果转换
  - Layer 5: 工具结果 → Plan 映射
- 消息流详解（3 个典型场景）
- 权限系统详解
- 实用工具层分析
- 数据缓存策略
- 命令处理
- 特殊情况处理
- 启动流程详解
- 库使用方式
- 测试覆盖说明
- 性能考虑
- 安全考虑
- 扩展点说明
- 常见问题解答

**何时阅读：**
- ✅ 要修改核心逻辑时
- ✅ 要添加新功能时
- ✅ 要优化性能时
- ✅ 学习系统设计时

**关键章节：**
```
1. 项目概述 (2 分钟)
   ↓
2. 项目结构 (3 分钟)
   ↓
3. 核心架构：5 层协议适配 (35 分钟) ⭐⭐⭐
   ├─ 第 1 层：ACP 代理
   ├─ 第 2 层：MCP 服务器
   ├─ 第 3 层：工具转换
   ├─ 第 4 层：工具结果
   └─ 第 5 层：Plan 映射
   ↓
4. 消息流详解 (10 分钟)
   ↓
5. 权限系统 (5 分钟)
   ↓
6. 缓存、命令、启动流程 (5 分钟)
```

---

### 3. **ARCHITECTURE_DIAGRAMS.md** (⭐⭐⭐ 可视化学习)
**文件大小：** ~32KB | **阅读时间：** 30-40 分钟

**适合人群：** 喜欢图表的人、系统架构师

**内容：**
- 全局架构图
- 会话生命周期图
- Prompt 处理流程图
- Query 迭代循环流程图
- 工具调用和结果处理流程图
- 权限系统状态机图
- 文件操作缓存流程图
- MCP 服务器架构图
- 流式响应架构图
- 后台终端管理流程图
- 完整会话示例（T0-T19 时间线）
- 并发安全考虑点

**何时阅读：**
- ✅ 第一次学习项目架构时
- ✅ 要理解数据流时
- ✅ 要调试问题时
- ✅ 向他人解释项目时

**推荐阅读顺序：**
1. 全局架构 (2 分钟)
2. 会话生命周期 (3 分钟)
3. Prompt 处理流程 (5 分钟) ← 最重要
4. Query 迭代循环 (10 分钟) ← 最关键
5. 工具调用和结果 (8 分钟) ← 核心机制
6. 权限系统状态机 (5 分钟)
7. 完整会话示例 (15 分钟) ← 最直观

---

## 🗺️ 学习路径

### 路径 A: 快速入门（20 分钟）
```
新开发者 / 快速了解
    ↓
1. 阅读 README.md（项目介绍）
2. 阅读 CLAUDE.md（开发指南）
3. 运行 npm run build && npm run test
    ↓
✅ 能够编译、测试和基本开发
```

### 路径 B: 深度学习（2 小时）
```
想要全面理解
    ↓
1. 快速入门（路径 A）
2. 看 ARCHITECTURE_DIAGRAMS.md 中的全局架构
3. 读 PROJECT_DETAILED_ANALYSIS.md 第 1-3 层
4. 看 ARCHITECTURE_DIAGRAMS.md 中的 Query 循环
5. 读 PROJECT_DETAILED_ANALYSIS.md 消息流详解
6. 看完整会话示例
    ↓
✅ 理解整个系统如何工作
```

### 路径 C: 贡献者级别（4 小时）
```
要参与项目贡献
    ↓
1. 完成路径 B
2. 阅读 PROJECT_DETAILED_ANALYSIS.md 全文
3. 研究 ARCHITECTURE_DIAGRAMS.md 中的权限、缓存、状态机
4. 查看源代码注释：
   - src/acp-agent.ts (prompt 方法)
   - src/mcp-server.ts (工具定义)
5. 运行测试并理解每个测试
6. 创建一个小的改进或修复作为练习
    ↓
✅ 准备好对项目做出重大贡献
```

### 路径 D: 系统设计师级别（全天）
```
学习为了教学/架构决策
    ↓
1. 完成路径 C
2. 研究所有三个文档的关键部分
3. 对比 ACP vs MCP vs SDK 的角色
4. 分析权限系统的设计决策
5. 考虑可能的改进和扩展
6. 研究其他 ACP/MCP 实现如何处理类似问题
7. 创建架构文档或技术演讲稿
    ↓
✅ 能够解释、改进和扩展系统
```

---

## 🎯 按任务查找信息

### 我要...

#### 🔧 运行项目
→ **CLAUDE.md** → "Common Commands" 部分

#### 🐛 修复一个 bug
→ **CLAUDE.md** → "Testing" 部分
→ **PROJECT_DETAILED_ANALYSIS.md** → 相关章节
→ **ARCHITECTURE_DIAGRAMS.md** → 相关流程图

#### ✨ 添加新功能
1. **CLAUDE.md** → "Architecture Overview"
2. **PROJECT_DETAILED_ANALYSIS.md** → "核心架构：5 层协议适配"
3. **ARCHITECTURE_DIAGRAMS.md** → 相关流程图
4. **CLAUDE.md** → "Debugging"

#### 🎓 理解权限系统
→ **PROJECT_DETAILED_ANALYSIS.md** → "权限系统详解" 部分
→ **ARCHITECTURE_DIAGRAMS.md** → "权限系统状态机"

#### 💾 处理文件操作
→ **PROJECT_DETAILED_ANALYSIS.md** → "第 2 层：MCP 服务器和工具" (Tool 1-3)
→ **ARCHITECTURE_DIAGRAMS.md** → "文件操作缓存"

#### 🖥️ 执行终端命令
→ **PROJECT_DETAILED_ANALYSIS.md** → "第 2 层：MCP 服务器和工具" (Tool 4-6)
→ **ARCHITECTURE_DIAGRAMS.md** → "后台终端管理"

#### 📨 理解消息流
→ **PROJECT_DETAILED_ANALYSIS.md** → "消息流详解"
→ **ARCHITECTURE_DIAGRAMS.md** → "Prompt 处理流程" 和 "Query 迭代循环"

#### 🔐 安全性考虑
→ **PROJECT_DETAILED_ANALYSIS.md** → "安全考虑"
→ **PROJECT_DETAILED_ANALYSIS.md** → "权限系统详解"

#### 📈 性能优化
→ **PROJECT_DETAILED_ANALYSIS.md** → "性能考虑"
→ **ARCHITECTURE_DIAGRAMS.md** → "文件操作缓存"

#### 🔌 添加 MCP 服务器支持
→ **PROJECT_DETAILED_ANALYSIS.md** → "扩展点"
→ **CLAUDE.md** → "Architecture Overview" → Layer 2

#### 👥 向团队讲解项目
→ **ARCHITECTURE_DIAGRAMS.md** → 所有流程图
→ **PROJECT_DETAILED_ANALYSIS.md** → 完整会话示例 (T0-T19)

---

## 📊 文档内容矩阵

| 主题 | CLAUDE.md | ANALYSIS.md | DIAGRAMS.md |
|------|:---------:|:-----------:|:-----------:|
| 项目概述 | ✅ | ✅⭐ | ✅ |
| 快速命令 | ✅⭐ | - | - |
| 架构分层 | ✅ | ✅⭐⭐⭐ | ✅⭐ |
| ACP 代理 | ✅ | ✅⭐⭐ | ✅⭐ |
| MCP 服务器 | ✅ | ✅⭐⭐ | ✅ |
| 消息流 | - | ✅⭐⭐ | ✅⭐⭐⭐ |
| 权限系统 | ✅ | ✅⭐⭐ | ✅⭐⭐ |
| 工具处理 | ✅ | ✅⭐⭐⭐ | ✅⭐ |
| 缓存策略 | ✅ | ✅⭐ | ✅⭐ |
| 会话管理 | ✅ | ✅⭐ | ✅⭐ |
| 启动流程 | - | ✅⭐ | ✅ |
| 测试指南 | ✅⭐ | ✅ | - |
| 调试技巧 | ✅⭐ | ✅ | - |
| 扩展点 | ✅ | ✅⭐ | - |
| Q&A | - | ✅⭐⭐ | - |
| 图表示例 | - | - | ✅⭐⭐⭐ |

**图例：** ✅ = 包含 | ⭐ = 深入 | ⭐⭐ = 非常深入 | ⭐⭐⭐ = 最深入

---

## 🎬 从代码到概念

### 我有一段代码，想要理解它

#### 例 1: `ClaudeAcpAgent.prompt()` 方法 (acp-agent.ts:314-439)

```
查找流程：
1. CLAUDE.md → "Architecture Overview" → "Layer 1: ACP Agent"
2. PROJECT_DETAILED_ANALYSIS.md → "第 1 层" → "prompt() 方法"
   (读取该方法的完整解析)
3. ARCHITECTURE_DIAGRAMS.md → "Prompt 处理流程"
   (查看流程图)
4. ARCHITECTURE_DIAGRAMS.md → "Query 迭代循环"
   (深入消息处理)
5. 回到源代码，现在理解每一行
```

#### 例 2: `registerTool("Edit")` 的实现 (mcp-server.ts:224-303)

```
查找流程：
1. CLAUDE.md → "Architecture Overview" → "Layer 2: MCP Server"
2. PROJECT_DETAILED_ANALYSIS.md → "第 2 层" → "Tool 3: Edit"
3. ARCHITECTURE_DIAGRAMS.md → "工具调用和结果处理"
4. 查看相关的 replaceAndCalculateLocation() 实现
5. 看 PROJECT_DETAILED_ANALYSIS.md → "特殊情况处理"
```

#### 例 3: 权限检查流程 (acp-agent.ts:205-217, mcp-server.ts:181-189)

```
查找流程：
1. PROJECT_DETAILED_ANALYSIS.md → "权限系统详解"
2. ARCHITECTURE_DIAGRAMS.md → "权限系统状态机"
3. CLAUDE.md → "Permission and Tool Handling"
4. 回到代码，追踪 permissionMode 的流转
```

---

## 💡 常见学习问题

### Q: 我应该先读哪个文档？
**A:**
- **新手**：从 CLAUDE.md 开始，然后看 README.md
- **有经验的开发者**：ARCHITECTURE_DIAGRAMS.md (全局视图) + PROJECT_DETAILED_ANALYSIS.md (细节)
- **架构师**：PROJECT_DETAILED_ANALYSIS.md (全部) + ARCHITECTURE_DIAGRAMS.md (深化理解)

### Q: 这三个文档有重复吗？
**A:** 有意的设计，每个文档有不同的：
- **CLAUDE.md**：实用、命令、快速参考
- **PROJECT_DETAILED_ANALYSIS.md**：深入、理论、完整解释
- **ARCHITECTURE_DIAGRAMS.md**：可视化、流程、具体示例

可以按需要只读部分，或全读深化理解。

### Q: 读这些文档需要多长时间？
**A:**
- 快速浏览：5-10 分钟（CLAUDE.md）
- 学习开发：30-45 分钟（CLAUDE.md + DIAGRAMS.md）
- 深度学习：2-3 小时（全部）
- 专家级别：4-6 小时（全部 + 源代码研究）

### Q: 我可以离线阅读吗？
**A:** 是的！所有文档都是 Markdown，可以离线查看。建议克隆仓库：
```bash
git clone https://github.com/zed-industries/claude-code-acp.git
# 然后在编辑器中打开 .md 文件
```

### Q: 这些文档会过时吗？
**A:** 我已尝试写得足够抽象和通用：
- CLAUDE.md 遵循代码更新保持最新
- PROJECT_DETAILED_ANALYSIS.md 架构相对稳定，只需偶尔更新
- ARCHITECTURE_DIAGRAMS.md 仅在主要架构变化时更新

建议定期检查 CHANGELOG.md 了解变化。

---

## 🚀 下一步

### 现在你已经了解了项目，可以：

1. **运行和测试**
   ```bash
   npm install
   npm run build
   npm run test
   ```

2. **探索源代码**
   ```
   src/
   ├── index.ts (26 行) → 入口点
   ├── acp-agent.ts (820 行) → 核心代理
   ├── mcp-server.ts (928 行) → 工具定义
   └── tools.ts (540 行) → 转换逻辑
   ```

3. **实践修改**
   - 在 test 文件中添加新的测试用例
   - 尝试添加一个新工具
   - 改进权限提示消息

4. **参与社区**
   - 提交 issue 报告 bug
   - 提交 PR 改进代码
   - 在讨论中分享见解

---

## 📞 获取帮助

如果有任何问题：

1. **查找关键词**：在文档中搜索你的问题
2. **查看 Q&A**：PROJECT_DETAILED_ANALYSIS.md 的"常见问题"部分
3. **查看源代码注释**：源代码中的注释往往能回答为什么
4. **提交 issue**：GitHub Issues 获取社区帮助

---

## 📄 文档元数据

| 文档 | 行数 | 字数 | 深度 | 实用性 | 最后更新 |
|------|---:|---:|:---:|:---:|---:|
| CLAUDE.md | ~200 | ~2K | 中等 | 高 | 2025-11-04 |
| PROJECT_DETAILED_ANALYSIS.md | ~700 | ~13K | 深 | 中等 | 2025-11-04 |
| ARCHITECTURE_DIAGRAMS.md | ~500 | ~15K | 深 | 高 | 2025-11-04 |
| **总计** | ~1400 | ~30K | - | - | - |

---

**祝学习愉快！🎓**

如果这个文档对你有帮助，请考虑：
- ⭐ 给项目点个 star
- 🔗 分享给其他开发者
- 💬 在讨论中提供反馈
