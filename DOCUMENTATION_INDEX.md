# 📚 claude-code-acp 项目文档索引

> 本项目现有完整的多层次文档生态，帮助你快速上手和深入理解 claude-code-acp。

## 🚀 快速开始（5 分钟）

**第一次接触这个项目？**

1. 阅读 **[CLAUDE.md](./CLAUDE.md)** - 快速参考指南
2. 运行命令：
   ```bash
   npm install
   npm run build
   npm run test
   ```
3. 完成！现在你可以开始开发了

---

## 📖 完整文档指南

### 1. **CLAUDE.md**
**⏱️ 阅读时间：10-15 分钟**

**这是什么：** 为 Claude Code 开发者准备的快速参考和开发指南

**包含内容：**
- 常用命令速查表（build, lint, test）
- 项目结构和文件说明
- 5 层架构简要概述
- 会话和权限管理说明
- 调试技巧

**何时阅读：**
- ✅ 刚加入项目时
- ✅ 需要快速查找命令时
- ✅ 想要快速了解项目时

**[👉 阅读 CLAUDE.md](./CLAUDE.md)**

---

### 2. **PROJECT_DETAILED_ANALYSIS.md**
**⏱️ 阅读时间：45-60 分钟**

**这是什么：** 项目的深度技术分析和完整解析

**包含内容：**
- 项目概述和目标
- 2533 行代码结构分析
- **5 层协议适配架构详解**
  - Layer 1: ACP 代理（820 行）
  - Layer 2: MCP 服务器（928 行）
  - Layer 3: 工具转换（540 行）
  - Layer 4: 工具结果转换
  - Layer 5: Plan 映射
- 3 个典型场景的消息流
- 权限系统深度解析
- 缓存策略详解
- 启动流程分解
- 20+ 个常见问题解答

**何时阅读：**
- ✅ 要修改核心逻辑时
- ✅ 要添加新功能时
- ✅ 想深入理解系统时

**[👉 阅读 PROJECT_DETAILED_ANALYSIS.md](./PROJECT_DETAILED_ANALYSIS.md)**

---

### 3. **ARCHITECTURE_DIAGRAMS.md**
**⏱️ 阅读时间：30-40 分钟**

**这是什么：** 包含 15+ 流程图和可视化的架构指南

**包含内容：**
- 全局架构图
- 会话生命周期图
- Prompt 处理流程
- Query 迭代循环（最关键）
- 工具调用和结果处理
- 权限系统状态机
- 文件操作缓存流程
- MCP 服务器架构
- 流式响应架构
- 后台终端管理
- **完整的 T0-T19 会话示例**（最直观）
- 并发安全分析

**何时阅读：**
- ✅ 第一次学习架构时
- ✅ 需要可视化理解时
- ✅ 要调试问题时

**[👉 阅读 ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md)**

---

### 4. **DOCUMENTATION_GUIDE.md**
**⏱️ 阅读时间：15-20 分钟**

**这是什么：** 文档导航和学习指南

**包含内容：**
- **4 条学习路径**
  - 路径 A: 快速入门（20 分钟）
  - 路径 B: 深度学习（2 小时）
  - 路径 C: 贡献者级别（4 小时）
  - 路径 D: 系统设计师级别（全天）
- 按任务快速查找表
- 文档内容对比矩阵
- 学习成果检查清单
- 常见学习问题解答

**何时阅读：**
- ✅ 第一次接触时
- ✅ 不知道读什么时
- ✅ 需要导航时

**[👉 阅读 DOCUMENTATION_GUIDE.md](./DOCUMENTATION_GUIDE.md)**

---

### 5. **DOCUMENTATION_SUMMARY.txt**
**⏱️ 阅读时间：5 分钟**

**这是什么：** 快速总结和检查清单

**包含内容：**
- 所有文档的快速总结
- 核心概念一览
- 文档统计数据
- 推荐阅读顺序
- 快速开始指令

**何时阅读：**
- ✅ 需要快速总览时
- ✅ 选择阅读路径时

**[👉 阅读 DOCUMENTATION_SUMMARY.txt](./DOCUMENTATION_SUMMARY.txt)**

---

## 🎯 按需求快速查找

### 我想要...

**快速了解项目**
→ [CLAUDE.md](./CLAUDE.md) - 项目概述部分

**学会如何开发**
→ [CLAUDE.md](./CLAUDE.md) - 常用命令部分

**理解整个系统**
→ [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md) - 全局架构图

**深入学习实现细节**
→ [PROJECT_DETAILED_ANALYSIS.md](./PROJECT_DETAILED_ANALYSIS.md) - 5 层架构部分

**看懂消息流**
→ [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md) - Query 迭代循环

**理解权限系统**
→ [PROJECT_DETAILED_ANALYSIS.md](./PROJECT_DETAILED_ANALYSIS.md) - 权限系统详解
→ [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md) - 权限状态机

**学会处理文件**
→ [PROJECT_DETAILED_ANALYSIS.md](./PROJECT_DETAILED_ANALYSIS.md) - Layer 2 工具

**了解缓存策略**
→ [PROJECT_DETAILED_ANALYSIS.md](./PROJECT_DETAILED_ANALYSIS.md) - 数据缓存策略
→ [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md) - 文件操作缓存

**修复一个 bug**
→ [CLAUDE.md](./CLAUDE.md) - 测试部分
→ [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md) - 查看相关流程图

**添加新功能**
→ [DOCUMENTATION_GUIDE.md](./DOCUMENTATION_GUIDE.md) - 路径 C
→ [PROJECT_DETAILED_ANALYSIS.md](./PROJECT_DETAILED_ANALYSIS.md) - 扩展点部分

**向团队讲解项目**
→ [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md) - 所有流程图

**获得答案**
→ [PROJECT_DETAILED_ANALYSIS.md](./PROJECT_DETAILED_ANALYSIS.md) - 常见问题

---

## 📊 文档对比表

| 特点 | CLAUDE.md | ANALYSIS.md | DIAGRAMS.md | GUIDE.md |
|------|:---------:|:-----------:|:-----------:|:--------:|
| 快速性 | ⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐ |
| 深度性 | ⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐ |
| 可视性 | ⭐ | ⭐ | ⭐⭐⭐ | ⭐ |
| 实用性 | ⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐ |
| 完整性 | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

---

## 🎓 推荐学习路径

### 路径 1: 快速入门（20 分钟）
```
CLAUDE.md (10 min)
    ↓
运行项目 (10 min)
    ↓
✅ 能编译和测试
```

### 路径 2: 基础理解（2 小时）
```
DOCUMENTATION_GUIDE.md - 路径介绍 (10 min)
    ↓
ARCHITECTURE_DIAGRAMS.md - 全局架构 (20 min)
    ↓
PROJECT_DETAILED_ANALYSIS.md - 第 1-3 层 (60 min)
    ↓
ARCHITECTURE_DIAGRAMS.md - 完整示例 (20 min)
    ↓
✅ 理解整个系统
```

### 路径 3: 深度掌握（4 小时）
```
完成路径 2 (2 h)
    ↓
PROJECT_DETAILED_ANALYSIS.md - 全部 (80 min)
    ↓
ARCHITECTURE_DIAGRAMS.md - 权限/缓存 (40 min)
    ↓
✅ 准备贡献
```

### 路径 4: 专家级别（全天）
```
完成路径 3 (4 h)
    ↓
源代码深入研究 (2-4 h)
    ↓
实验和改进 (2-4 h)
    ↓
✅ 能设计和教学
```

---

## 📈 文档统计

- **总大小**：~85 KB
- **总行数**：~2,850 行
- **文档数**：5 个
- **覆盖代码行**：2,533 行（100%）
- **包含流程图**：15+
- **包含示例**：完整的 T0-T19 会话示例

---

## 💡 核心概念速览

### ACP 适配器
这个项目是一个 **ACP (Agent Client Protocol) 适配器**，让任何支持 ACP 的编辑器（Zed、Emacs、Neovim 等）都能使用 Claude Code。

### 5 层架构
```
Layer 1: ACP 代理          ← 处理 ACP 协议
Layer 2: MCP 服务器        ← 提供工具
Layer 3: 工具转换          ← 格式转换
Layer 4: 结果转换          ← 输出格式
Layer 5: Plan 映射         ← TodoWrite 转换
```

### 关键设计
- **Query 迭代**：异步生成器驱动对话
- **会话隔离**：每个用户完全独立
- **权限模式**：4 种灵活选择
- **缓存策略**：工具和文件缓存
- **流式响应**：实时推送更新

---

## 🔗 快速链接

**项目资源：**
- 📝 README.md - 项目简介
- 🔧 package.json - 依赖配置
- 💻 src/ - 源代码目录

**外部资源：**
- 🌐 [ACP 规范](https://agentclientprotocol.com)
- 🤖 [Claude Code 文档](https://docs.anthropic.com)
- 🔌 [MCP 协议](https://modelcontextprotocol.io)

---

## ✅ 文档检查清单

**基础知识：**
- [ ] 理解 ACP 是什么
- [ ] 理解项目为什么存在
- [ ] 了解 5 层架构
- [ ] 能运行项目

**核心理解：**
- [ ] 理解会话生命周期
- [ ] 理解消息流
- [ ] 理解权限系统
- [ ] 理解工具处理

**高级知识：**
- [ ] 理解缓存策略
- [ ] 理解流式处理
- [ ] 理解并发管理
- [ ] 能修改和扩展代码

---

## 📞 获取帮助

1. **文档内搜索** - 在文档中查找关键字
2. **查看代码示例** - 每个文档都有具体示例
3. **参考 Q&A** - PROJECT_DETAILED_ANALYSIS.md 有常见问题
4. **研究源代码** - src/ 目录有详细注释
5. **提交 issue** - GitHub Issues 获取社区帮助

---

## 🎉 欢迎开始！

选择上面的任何文档开始阅读，根据你的需求选择合适的学习路径。

**建议：** 如果你不确定从哪里开始，先读 [DOCUMENTATION_GUIDE.md](./DOCUMENTATION_GUIDE.md)，它会指导你选择最合适的路径。

祝你学习愉快！🚀

---

**最后更新：** 2025-11-04
