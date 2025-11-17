# 定制方案设计

本文档包含三个不同层级的灵感管理自动化方案设计。

---

## 方案概览

- **方案A**: 轻量级方案（Claude Code + Markdown）
- **方案B**: 混合方案（第三方工具 + Claude Code）
- **方案C**: 深度自动化方案（自建脚本 + API集成）

---

## 方案A：Claude Code + Markdown（轻量级）

### 核心理念

用Claude Code处理内容，Markdown存储知识，Git版本管理，无需额外工具或服务。完全基于本地文件系统，保持数据完全可控。

### 工作流程（5步）

1. **收集** - 发现灵感时，复制URL或内容文本到Claude Code
2. **处理** - 使用预设Prompt模板，Claude自动提取：标题、摘要、关键要点、标签、应用场景
3. **归档** - Claude自动生成Markdown文件，保存到 `inspiration/YYYY-MM/topic-name.md`
4. **索引** - 自动更新 `inspiration/index.md`，添加新条目（按时间和主题）
5. **回顾** - 每周运行回顾Prompt，Claude生成「本周灵感总结」到 `inspiration/weekly/YYYY-WW.md`

### 所需文件结构

```
inspiration/
├── index.md                          # 主索引（按主题分类）
├── chronological-index.md            # 时间线索引（按日期）
├── 2024-11/                          # 按月归档
│   ├── ai-automation-tools.md
│   ├── code-architecture-patterns.md
│   └── workflow-optimization.md
├── 2024-12/
│   └── ...
├── weekly/                           # 每周总结
│   ├── 2024-W46.md
│   └── 2024-W47.md
└── templates/
    ├── inspiration-template.md       # 灵感条目模板
    └── weekly-summary-template.md    # 周总结模板
```

### 核心Prompt模板

**日常收集Prompt：**

```
我发现了一个灵感素材：

[粘贴URL或内容描述]

请帮我：
1. 提取核心信息（50-100字摘要）
2. 提炼3-5个关键要点（bullet points）
3. 生成3-5个关键词标签
4. 标注适用场景（如：AI工具开发、工作流优化、技术架构等）
5. 建议后续行动（如有）

然后按以下模板格式生成Markdown文件：

---
标题: [自动提取]
来源: [URL]
日期: 2024-11-17
类型: [文章/视频/工具/对话]
标签: #tag1 #tag2 #tag3

## 核心内容
[摘要]

## 关键要点
- 要点1
- 要点2
- 要点3

## 应用场景
- 场景1: [具体描述]
- 场景2: [具体描述]

## 后续行动
- [ ] 行动项1（如有）
---

请将文件保存到 `inspiration/2024-11/[自动命名].md`，
并在 `inspiration/index.md` 中添加索引条目。

执行后告诉我文件路径。
```

**每周回顾Prompt：**

```
请分析 `inspiration/2024-11/` 目录下本周新增的灵感条目。

生成每周总结，包括：
1. 本周收集了多少条灵感
2. Top 3 最有价值的发现
3. 主题分布（按标签统计）
4. 推荐优先行动的3个后续任务

保存到 `inspiration/weekly/2024-W47.md`
```

### 优点（3个）

✅ **完全免费** - 零月度成本，只依赖已有工具（Claude Code + Git）

✅ **完全可控** - 数据存储在本地，Markdown纯文本格式，永不锁定，可随时迁移

✅ **无缝集成** - 与现有开发workflow完全融合，在Claude Code中直接操作，无需切换工具

### 缺点（3个）

❌ **需要手动触发** - 每次收集灵感需要手动复制内容到Claude Code并运行Prompt

❌ **没有自动导入** - 无浏览器插件快速保存，无法从RSS/邮箱自动抓取

❌ **需要自律维护** - 依赖个人习惯，容易因忘记而中断

### 预估工作量

- **初始搭建**：2小时
  - 创建文件结构：15分钟
  - 设计模板和Prompt：1小时
  - 测试优化：45分钟

- **日常使用**：每条灵感30秒-1分钟
  - 复制内容：5秒
  - 粘贴Prompt：5秒
  - Claude处理：10-20秒
  - 确认保存：10秒

- **每周维护**：10-15分钟
  - 运行周总结Prompt：5分钟
  - 审查并调整：5-10分钟
