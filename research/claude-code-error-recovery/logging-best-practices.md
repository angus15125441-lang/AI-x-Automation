# 日志记录与审计最佳实践

## 日志记录层级

### Level 1: 最小日志(适用于简单任务)

**记录内容**:
- 任务开始/结束时间
- 主要操作摘要(3-5条)
- 最终输出文件列表
- 是否成功(✅/❌)

**实现方式** - 提示词示例:
```markdown
请完成:{任务描述}

日志要求:
在 logs/{task-name}-{date}.md 中记录:
- 开始时间: {自动记录当前时间}
- 结束时间: {完成时记录}
- 主要操作:
  1. [操作1描述]
  2. [操作2描述]
  3. ...
- 输出文件: [文件路径列表]
- 状态: ✅成功 / ❌失败
```

**适用场景**:
- 单文件编辑
- 简单脚本执行
- 无网络依赖的任务
- 预计执行时间<5分钟

### Level 2: 标准日志(推荐)

**记录内容**:
- **元信息**: 任务名、分支名、开始/结束时间
- **阶段记录**: 每个阶段的输入来源、处理步骤、输出结果、耗时
- **关键决策点**: 为什么选择某个方案、备选方案是什么
- **错误上下文**: 错误发生时的完整状态、恢复方式
- **工具调用**: 使用了哪些工具(Grep/Read/Bash)、调用原因
- **最终交付**: 完成清单、未完成项说明

**实现方式** - 提示词示例:
```markdown
请完成:{任务描述}

日志要求:
在 logs/{task-name}-execution-log.md 创建执行日志,结构:

---
# 执行日志: {任务名}

## 元信息
- 任务ID: {唯一标识}
- 开始时间: {YYYY-MM-DD HH:mm:ss}
- 结束时间: {YYYY-MM-DD HH:mm:ss}
- 执行分支: {branch-name}
- 执行者: Claude Code

## 执行记录

### 阶段1: {阶段名}
**输入**: {来源/前置条件}
**处理步骤**:
1. [步骤1] - 工具:{Grep/Read/...}
2. [步骤2] - 决策:{为什么这样做}
3. ...

**输出**: {结果摘要 + 文件路径}
**耗时**: {X分钟}
**状态**: ✅成功 / ⚠️部分成功 / ❌失败

### 阶段2: {阶段名}
[重复以上结构]

## 错误记录(如有)
### 错误1
- 发生时间: {HH:mm:ss}
- 错误类型: {参考error-types-and-recovery.md分类}
- 症状描述: {详细说明}
- 触发操作: {正在执行什么}
- 恢复方式: {采取了什么策略}
- 是否成功: ✅/❌

## 最终交付
**已完成**:
- [x] 文件1: /path/to/file1
- [x] 文件2: /path/to/file2

**未完成**:
- [ ] 项目X - 原因:{说明}

**备注**: {其他重要信息}
---

在每个阶段结束时,立即更新此日志文件。
```

**适用场景** (推荐默认使用):
- 网络搜索>2次
- 多阶段任务(>3个阶段)
- 预计执行时间>10分钟
- 生产环境部署
- 需要团队协作的任务

### Level 3: 详细审计(复杂任务)

**记录内容**(Level 2基础上增加):
- **完整决策树**: 每个分支点的考虑因素、权衡、最终选择
- **工具调用明细**: 每次工具调用的参数、返回值、耗时
- **中间状态快照**: 关键时刻的变量值、文件内容hash
- **OpenTelemetry追踪**: 使用标准协议记录spans和traces
- **性能指标**: Token使用量、API调用次数、延迟分布
- **可复现性信息**: 环境变量、依赖版本、随机种子

**实现方式** - 集成OpenTelemetry:

**提示词示例**:
```markdown
请完成:{任务描述}

审计要求:
1. 启用详细日志模式: --verbose --debug
2. 使用OpenTelemetry追踪,输出到 logs/{task-name}-otel-trace.json
3. 在 logs/{task-name}-audit-log.md 记录:

---
# 详细审计日志: {任务名}

## 执行环境
- Claude Code版本: {version}
- 操作系统: {OS}
- 工作目录: {pwd}
- Git commit: {hash}
- 环境变量: {关键变量}

## Trace概览
- Trace ID: {otel-trace-id}
- 根Span: {root-span-name}
- 总耗时: {duration}
- 总Spans数: {count}

## 决策树

### 决策点1: {决策名}
**时间**: {HH:mm:ss}
**背景**: {为什么需要做决策}
**选项**:
- 选项A: {描述} - 优势:{...} - 劣势:{...}
- 选项B: {描述} - 优势:{...} - 劣势:{...}

**最终选择**: 选项A
**理由**: {详细说明}
**影响**: {对后续流程的影响}

### 决策点2: ...
[重复结构]

## 工具调用明细

### Span 1: Grep搜索
- Span ID: {id}
- 父Span: {parent-id}
- 开始时间: {timestamp}
- 参数:
  ```json
  {
    "pattern": "error recovery",
    "path": "src/",
    "output_mode": "files_with_matches"
  }
  ```
- 返回值: [10个匹配文件]
- 耗时: 234ms
- 状态: success

### Span 2: Read文件
[重复结构]

## 性能指标
- Token使用:
  * 输入: 12,543 tokens
  * 输出: 8,921 tokens
  * 缓存命中: 45%
- API调用:
  * 总次数: 23
  * 平均延迟: 1.2s
  * 最大延迟: 3.5s
- 工具调用分布:
  * Read: 15次
  * Grep: 5次
  * Edit: 8次
  * Bash: 3次

## 中间状态快照

### 快照1: 阶段1完成时
- 时间: {timestamp}
- 文件状态:
  * error-types.md: hash=abc123, size=5.2KB
  * logging.md: hash=def456, size=3.1KB
- Git状态: 2 files staged
- 待处理队列: [阶段2, 阶段3]

## 可复现性信息
- 随机种子: {如有}
- API端点: {endpoint-url}
- 模型版本: claude-sonnet-4-5-20250929
- 提示词模板版本: v2.3.1
- 依赖版本: {package.json snapshot}

---
```

**工具集成** - 使用监控平台:
- **SigNoz**: `export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317`
- **Honeycomb**: `export HONEYCOMB_API_KEY=...`
- **Grafana**: `export GRAFANA_LOKI_URL=...`

**适用场景**:
- 生产环境关键任务
- 需要后续优化的工作流
- 安全审计需求
- 研究性质的任务(分析Agent行为)
- 多Agent协作系统

## 日志存储策略

### 文件命名规范
```
logs/
├── {task-type}/
│   ├── {task-name}-{YYYY-MM-DD}-execution.md    # 标准日志
│   ├── {task-name}-{YYYY-MM-DD}-audit.md        # 详细审计
│   ├── {task-name}-{YYYY-MM-DD}-otel-trace.json # OpenTelemetry追踪
│   └── {task-name}-{YYYY-MM-DD}-errors.md       # 错误专项记录
└── archive/
    └── {YYYY-MM}/                                # 按月归档
```

**示例**:
```
logs/
├── research/
│   ├── claude-code-error-recovery-2025-01-15-execution.md
│   └── claude-code-error-recovery-2025-01-15-audit.md
├── feature-development/
│   ├── user-auth-2025-01-14-execution.md
│   └── user-auth-2025-01-14-errors.md
└── archive/
    └── 2024-12/
```

### 目录结构最佳实践
- **按任务类型分类**: research/ feature-development/ debugging/ refactoring/
- **按月归档**: 超过30天的日志移入archive/
- **错误专项**: 有错误发生时创建独立的errors.md便于后续分析
- **trace文件分离**: JSON格式的trace单独存储,避免污染markdown可读性

### 自动化日志记录方法

#### 方法1: 提示词模板自动化
在每个提示词中嵌入日志记录指令:
```markdown
【日志记录】(自动执行)
在开始执行前,创建 logs/{task-type}/{task-name}-{date}-execution.md
在每个阶段开始时记录: "正在执行:[阶段名]"
在每个阶段结束时更新: "已完成:[阶段名],输出:[摘要]"
在遇到错误时立即记录到错误章节
```

#### 方法2: 使用Claude Code Hooks
配置session-start hook自动初始化日志:
```bash
# .claude/hooks/session-start.sh
#!/bin/bash
DATE=$(date +%Y-%m-%d)
mkdir -p logs/$(date +%Y-%m)
echo "# Session Log - $DATE" > logs/$(date +%Y-%m)/session-$DATE.md
echo "Session log initialized at: logs/$(date +%Y-%m)/session-$DATE.md"
```

#### 方法3: 集成OpenTelemetry自动追踪
在环境中启用自动追踪:
```bash
export OTEL_SERVICE_NAME="claude-code-agent"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
export OTEL_TRACES_EXPORTER="otlp"
export OTEL_LOGS_EXPORTER="otlp"
```

Claude Code原生支持OpenTelemetry,自动记录:
- API请求/响应
- 工具调用(Read/Write/Grep/Bash)
- Token使用量
- 错误和异常

### 日志清理与维护策略
- **保留期**:
  - 最近30天: 保留所有日志
  - 30-90天: 保留标准日志,删除详细审计
  - >90天: 仅保留错误日志和关键任务审计
- **压缩**: 归档日志使用gzip压缩节省空间
- **索引**: 维护 logs/INDEX.md 记录所有任务的快速索引

## 实战案例

### 案例1: 生产环境竞态条件调试
**场景**: 某遗留代码仓库存在潜伏数月的竞态条件bug,导致偶发性数据不一致

**出现的问题**:
- 错误类型: 逻辑错误 - 并发访问冲突
- 症状: 随机出现的数据覆盖,无稳定复现路径
- 传统调试耗时: 预计数小时人工审查代码

**日志如何帮助定位**:
1. 使用CLAUDE_LEGACY.md提供代码库上下文
2. Claude分析时记录了关键决策点:
   ```
   决策: 检查所有数据库操作是否有事务保护
   理由: 竞态条件通常出现在非原子性操作中
   ```
3. 在日志中标记了3个可疑文件和具体代码行
4. 工具调用记录显示:
   ```
   Grep "database.update" → 15个匹配
   Read suspicious_file.py:45-67 → 发现无事务包装
   ```

**恢复过程**:
1. Claude建议将操作包装在事务中
2. 自动生成测试用例验证并发安全性
3. 在日志中记录了修复前后的行为对比

**结果**: 将数小时调试压缩至几分钟,日志记录了完整的分析路径供团队学习

**来源**: Anthropic官方案例研究 - 安全工程团队实践

### 案例2: 多步骤工作流追踪与优化
**场景**: 设计团队使用Claude Code自动化Figma设计文件到代码的转换流程

**出现的问题**:
- 错误类型: 输出错误 - 部分组件未生成
- 症状: 10个组件中随机有2-3个缺失
- 难点: 不知道在哪个环节丢失

**日志如何帮助定位**:
使用OpenTelemetry + SigNoz追踪完整流程:
```
Trace: design-to-code-conversion
├─ Span 1: parse-figma-file (234ms) ✅
├─ Span 2: extract-components (567ms) ✅
│   ├─ Span 2.1: component-1 (45ms) ✅
│   ├─ Span 2.2: component-2 (52ms) ✅
│   └─ Span 2.3: component-3 (48ms) ❌ Error: timeout
├─ Span 3: generate-code (1.2s) ⚠️ 部分成功
└─ Span 4: run-tests (890ms) ❌ 失败
```

发现Span 2.3超时导致组件3未被提取,影响后续生成

**恢复过程**:
1. 在日志中定位到具体超时的组件类型(复杂嵌套组件)
2. 调整提示词,增加该类型组件的处理时间
3. 添加检查点: 每解析5个组件保存一次中间结果
4. 重新执行,日志显示所有组件成功提取

**结果**: 成功率从70%提升至98%,通过日志分析识别了性能瓶颈

**来源**: 产品设计团队生产工作流,详见Anthropic团队使用案例

### 案例3: 测试驱动开发的自然检查点
**场景**: 开发新功能,要求每步都有测试覆盖

**出现的问题**:
- 错误类型: 执行错误 - 测试失败但不知道是代码还是测试有问题
- 症状: 5个测试中3个失败
- 困惑: 失败是预期的(功能未实现)还是意外的(测试写错了)

**日志如何帮助定位**:
使用标准日志记录每个测试-实现循环:
```markdown
## 阶段1: 功能A - 用户认证
### 步骤1: 编写测试
- 输入: 需求文档
- 输出: test_user_auth.py (5个测试用例)
- 预期状态: 全部失败(功能未实现)

### 步骤2: 实现功能
- 输入: 测试用例
- 输出: user_auth.py
- 测试结果: 5/5通过 ✅

### 步骤3: 边缘情况测试
- 新增: 2个边缘情况测试
- 结果: 1/2失败 ❌
- 错误: test_invalid_token - 预期AssertionError,实际ValueError
```

日志清晰显示:在步骤3发现代码逻辑问题(应抛出AssertionError但抛出了ValueError)

**恢复过程**:
1. 检查日志中的预期vs实际对比
2. 修复异常类型
3. 重跑测试,更新日志: 2/2通过 ✅
4. Commit with message: "修复token验证异常类型(见log行X)"

**结果**: 日志成为开发节奏的自然组成部分,每个检查点都有清晰记录

**来源**: Claude Code最佳实践 - TDD工作流推荐

### 案例4: API速率限制的优雅降级
**场景**: 大批量数据处理任务触发API速率限制

**出现的问题**:
- 错误类型: 执行错误 - API 429速率限制
- 症状: 处理100个文件时在第23个失败
- 传统处理: 手动等待后从头开始(浪费前22个的工作)

**日志如何帮助定位**:
标准日志记录了每个文件的处理状态:
```markdown
## 批次1(文件1-20): ✅ 完成
## 批次2(文件21-40):
- 文件21: ✅
- 文件22: ✅
- 文件23: ❌ 错误429 Rate Limit
  - 时间: 14:23:45
  - X-RateLimit-Reset: 14:25:00(还需等待75秒)
  - 已完成: 22/100
  - 待处理: 78/100
```

**恢复过程**:
1. 日志显示已完成22个,从文件23开始继续
2. Claude检测到日志中的速率限制记录,自动调整:
   ```
   决策: 启用节流模式
   新策略: 每批5个文件,间隔15秒
   预计剩余时间: 78文件 * 3秒/文件 + 15次间隔 = 6分钟
   ```
3. 在日志中记录策略切换点
4. 继续执行,最终完成100/100

**结果**: 零损失恢复,日志中的X-RateLimit信息指导了最优等待时间

**来源**: 社区最佳实践 - 处理API限制的标准模式

### 案例5: 安全工程的Runbook自动生成与调试
**场景**: 安全团队需要为新服务创建故障排查手册,并用其调试实际生产问题

**出现的问题**:
- 错误类型: 逻辑错误 - 文档不完整导致排查路径遗漏
- 症状: 按Runbook操作无法定位某类故障
- 挑战: 不知道是Runbook有问题还是故障本身特殊

**日志如何帮助定位**:
使用详细审计记录Runbook生成+应用过程:
```markdown
## Runbook生成日志
### 输入源:
1. 服务文档: docs/service-A.md
2. 历史工单: tickets/2024-Q4/*.json (34个)
3. 监控告警规则: monitoring/alerts.yml

### 生成决策:
决策1: 故障分类
- 基于34个历史工单,识别出5大类故障
- 网络问题: 12个 → 占比35% → 纳入Runbook
- 配置错误: 8个 → 占比23% → 纳入Runbook
- 数据库超时: 7个 → 占比20% → 纳入Runbook
- 内存泄漏: 5个 → 占比15% → 纳入Runbook
- 其他: 2个 → 占比6% → 暂不纳入

## Runbook应用日志(生产调试)
### 问题: 服务响应缓慢
步骤1: 检查网络连接 → 正常 ✅
步骤2: 检查配置文件 → 正常 ✅
步骤3: 检查数据库延迟 → 异常 ❌ (平均延迟3.2s,正常<100ms)
  → 定位成功,问题类型:数据库超时

### 反馈改进:
发现: 步骤3未包含"检查数据库连接池状态"
原因: 历史工单中未明确记录此步骤
改进: 在Runbook中补充连接池检查
更新: runbook-v1.1.md,日志记录变更依据
```

**恢复过程**:
1. 日志中记录了Runbook的生成依据(34个工单)
2. 应用日志显示步骤3成功定位问题
3. 发现遗漏步骤后,日志记录了改进理由
4. 版本化Runbook,日志成为演进历史

**结果**: Runbook成为"活文档",日志追踪其有效性并指导迭代

**来源**: Anthropic安全工程团队,官方案例研究

## 日志记录检查清单

在开始任务前,根据任务复杂度选择日志级别:

### 简单任务(Level 1)
- [ ] 创建日志文件: `logs/{task-name}-{date}.md`
- [ ] 记录开始时间
- [ ] 列出主要操作步骤(3-5条)
- [ ] 记录输出文件列表
- [ ] 标记最终状态(✅/❌)

### 标准任务(Level 2) - 推荐
- [ ] 创建结构化日志: 包含元信息、执行记录、错误记录、最终交付
- [ ] 每个阶段记录: 输入、处理步骤、输出、耗时、状态
- [ ] 关键决策点记录理由
- [ ] 错误发生时立即记录上下文
- [ ] 每阶段结束后更新日志

### 复杂任务(Level 3)
- [ ] 启用OpenTelemetry追踪
- [ ] 配置监控后端(SigNoz/Honeycomb/Grafana)
- [ ] 记录完整决策树
- [ ] 记录所有工具调用的参数和返回值
- [ ] 创建中间状态快照
- [ ] 记录性能指标(token/延迟/调用次数)
- [ ] 保存可复现性信息(环境/版本/种子)

## 日志分析与优化

### 定期回顾(每周/每月)
1. **识别重复错误模式**:
   - 统计错误类型分布
   - 找出Top 3高频错误
   - 为高频错误创建预防模板

2. **优化提示词**:
   - 对比成功vs失败任务的日志
   - 提取成功任务的关键指令
   - 将优化后的提示词版本化

3. **性能基准**:
   - 记录相似任务的平均耗时
   - 识别异常慢的环节
   - 优化工具选择和执行顺序

### 构建知识库
将日志转化为可复用的知识:
```
knowledge-base/
├── error-patterns/
│   ├── api-timeout-recovery.md (基于10次相似错误的总结)
│   └── git-conflict-resolution.md
├── optimization-insights/
│   ├── search-efficiency.md (Grep vs Bash grep性能对比)
│   └── batch-processing.md (最优批次大小分析)
└── prompt-templates/
    ├── research-task-v2.1.md (基于5次成功任务优化)
    └── feature-dev-v1.3.md
```

### 日志驱动的持续改进循环
```
执行任务 → 记录日志 → 分析日志 → 优化提示词 → 再次执行
   ↑                                                    ↓
   └────────────────── 性能提升/错误减少 ←───────────────┘
```
