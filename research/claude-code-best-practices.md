# Claude Code 最佳实践

## 任务设计原则

### 1. 分解大任务
**为什么重要**：避免超时、卡顿，便于调试和恢复

❌ **错误示例**：
```
「请分析这100个网页并生成报告」
```

✅ **正确示例**：
```
步骤1：分析前10个网页，保存到temp/batch1.json
步骤2：分析第11-20个网页，保存到temp/batch2.json
...
步骤N：汇总所有结果到final_report.md
```

**分解策略**：
- 单个步骤<5分钟
- 每步有明确的输入输出
- 保存中间结果到文件
- 可以从任意步骤恢复

### 2. 明确输入输出
**为什么重要**：减少误解，提高效率，便于验证

❌ **错误示例**：
```
「帮我整理一下文件」
```

✅ **正确示例**：
```
输入：data/raw/ 目录下的所有CSV文件
操作：
  1. 合并为单个文件
  2. 去除重复行
  3. 按日期列排序
输出：data/processed/merged.csv
格式：保留原有列，添加processed_at时间戳列
```

**关键要素**：
- 明确文件路径（使用相对路径）
- 指定文件格式
- 说明数据结构
- 定义输出格式
- 说明特殊要求

### 3. 逐步验证
**为什么重要**：及时发现问题，避免返工

❌ **错误示例**：
```
一次性执行10步操作，最后发现第2步就错了
```

✅ **正确示例**：
```
执行步骤1 → 检查输出 → 确认正确
执行步骤2 → 检查输出 → 确认正确
...
```

**验证检查点**：
- 每步完成后查看输出文件
- 检查文件大小是否合理
- 抽样查看数据质量
- 验证数据格式正确性
- 确认数量符合预期

### 4. 控制复杂度
**为什么重要**：避免超时、内存溢出、Token耗尽

❌ **错误示例**：
```
单个命令处理1000个文件
```

✅ **正确示例**：
```
每次处理50-100个文件，分10批执行
```

**复杂度指标**：
- 文件数量：<100个/批
- 数据行数：<10000行/批
- 文件大小：<100MB/批
- 处理时间：<5分钟/步
- 循环次数：<1000次

### 5. 保存中间结果
**为什么重要**：避免重复计算，支持断点恢复

❌ **错误示例**：
```python
# 所有处理都在内存中
data = load_all_files()
processed = process(data)
final = finalize(processed)
save(final)
```

✅ **正确示例**：
```python
# 每步保存
data = load_all_files()
save_json(data, 'temp/step1_loaded.json')

processed = process(data)
save_json(processed, 'temp/step2_processed.json')

final = finalize(processed)
save_json(final, 'output/final_result.json')
```

**中间文件管理**：
- 使用temp/目录存放临时文件
- 清晰的文件命名（step1_xxx.json）
- 定期清理temp/目录
- 重要中间结果保留备份

## Prompt编写技巧

### 明确指令结构

**标准模板**：
```
请帮我完成以下任务：

**目标**：[1句话描述期望达成的结果]

**输入**：
- 文件：[具体路径]
- 格式：[数据格式说明]
- 数量：[预期数量]

**处理步骤**：
1. [具体步骤1 - 动词开头]
2. [具体步骤2 - 动词开头]
3. [具体步骤3 - 动词开头]

**输出**：
- 文件：[具体路径]
- 格式：[数据格式说明]
- 要求：[特殊格式要求]

**注意事项**：
- [边界情况处理]
- [错误处理策略]

请逐步执行，每步完成后等待我确认。
```

**实例**：
```
请帮我完成以下任务：

**目标**：清洗用户数据CSV文件

**输入**：
- 文件：data/users_raw.csv
- 格式：包含name, email, signup_date, status列
- 数量：约5000行

**处理步骤**：
1. 读取CSV文件
2. 删除email列为空的行
3. 将signup_date转换为YYYY-MM-DD格式
4. 删除重复的email记录（保留最早的）
5. 按signup_date排序

**输出**：
- 文件：data/users_clean.csv
- 格式：同输入，但数据已清洗
- 要求：保留原有列顺序

**注意事项**：
- 如果name为空，保留该行但name设为"Unknown"
- 输出处理统计（删除了多少重复项等）

请逐步执行，每步完成后告知进度。
```

### 避免模糊表达

| ❌ 模糊表达 | ✅ 明确表达 |
|----------|----------|
| 整理一下 | 按字母顺序排序 |
| 优化一下 | 删除重复项并压缩文件大小 |
| 改进一下 | 添加错误处理和日志记录 |
| 美化一下 | 添加Markdown格式和代码高亮 |
| 修复问题 | 修复XSS漏洞（转义用户输入） |
| 加个功能 | 添加导出CSV功能 |

**明确性检查清单**：
- [ ] 使用具体动词（排序、删除、添加、转换）
- [ ] 指定具体数量（前10个、所有、每批50个）
- [ ] 明确文件路径（避免"那个文件"）
- [ ] 说明预期结果（而非"看起来更好"）
- [ ] 提供示例（特别是格式转换时）

### 指定错误处理

**错误处理模板**：
```
如果遇到以下情况：
- [情况1] → [处理方式]
- [情况2] → [处理方式]
- [情况3] → [处理方式]
```

**实例**：
```
如果遇到以下情况：
- 文件不存在 → 跳过并记录到error.log
- 格式错误（缺少必需列） → 输出到invalid.csv
- 网络超时 → 重试3次，失败后跳过并记录
- 数据验证失败 → 保留原值并标记flag列为"需人工审核"
```

**常见错误场景**：
1. 文件不存在/路径错误
2. 格式不符合预期
3. 数据类型错误
4. 网络请求失败
5. 权限不足
6. 磁盘空间不足
7. 依赖包未安装

## 文件操作技巧

### 1. 使用相对路径
**为什么**：跨平台兼容，便于项目迁移

✅ **推荐**：
```
research/data/file.csv
./temp/output.json
../config/settings.yaml
```

❌ **不推荐**：
```
/Users/username/projects/research/data/file.csv
C:\Users\username\Desktop\file.csv
```

### 2. 创建清晰的目录结构
**标准项目结构**：
```
project/
├── input/          # 原始输入文件（只读）
├── output/         # 最终输出文件
├── temp/           # 临时中间文件
├── scripts/        # 生成的脚本
├── logs/           # 日志文件
├── config/         # 配置文件
└── docs/           # 文档
```

**使用场景示例**：
```
data_processing/
├── input/
│   └── raw_data.csv          # 原始数据
├── output/
│   ├── clean_data.csv        # 清洗后数据
│   └── report.md             # 分析报告
├── temp/
│   ├── step1_loaded.json     # 步骤1输出
│   └── step2_filtered.json   # 步骤2输出
└── scripts/
    └── process.py            # 处理脚本
```

### 3. 命名规范
**文件命名最佳实践**：
```
✅ 好的命名：
- 2024-11-18-user-analysis.csv
- step1_data_loading.json
- final_report_v2.md
- config_production.yaml

❌ 差的命名：
- 结果1.csv
- 新文件.txt
- data.json（太通用）
- temp123.csv（无意义）
```

**命名规则**：
- 使用小写字母和连字符
- 包含日期（YYYY-MM-DD格式）
- 说明内容或用途
- 版本号如需要（v1, v2）
- 避免空格和特殊字符
- 使用英文（便于脚本处理）

### 4. 及时清理
**清理策略**：
```bash
# 每个任务完成后
rm -rf temp/*

# 保留最近的临时文件
find temp/ -mtime +7 -delete

# 压缩旧输出
tar -czf archive/output_2024-11.tar.gz output/
```

**清理检查清单**：
- [ ] 删除temp/目录内容
- [ ] 归档旧的output文件
- [ ] 清理日志文件（保留最近7天）
- [ ] 删除重复的备份文件
- [ ] 检查磁盘空间使用

## 防止卡顿的技巧

### 1. 控制搜索次数
**策略**：
- 单个任务网络搜索不超过3次
- 优先使用已知信息和本地文档
- 批量搜索合并为一次
- 搜索结果保存到文件供后续使用

**示例**：
```
❌ 错误方式：
- 搜索"Python CSV读取"
- 搜索"Python CSV写入"
- 搜索"Python CSV去重"
- 搜索"Python CSV排序"
（4次搜索）

✅ 正确方式：
- 搜索"Python CSV处理完整教程"
- 保存结果到temp/csv_guide.md
- 后续参考本地文件
（1次搜索）
```

### 2. 分批处理数据
**分批策略**：
```python
# ❌ 一次处理所有
all_data = read_all_files()  # 可能10000+行
process(all_data)

# ✅ 分批处理
batch_size = 100
for i in range(0, len(files), batch_size):
    batch = files[i:i+batch_size]
    process_batch(batch)
    save_intermediate_result(f'temp/batch_{i}.json')
```

**分批大小建议**：
- CSV文件：每批1000-5000行
- JSON文件：每批50-100个文件
- 图片文件：每批20-50个
- API请求：每批10-20个

### 3. 设置超时和重试
**Python示例**：
```python
import time
import requests

def safe_fetch(url, max_retries=3, timeout=10):
    """带重试的网络请求"""
    for i in range(max_retries):
        try:
            response = requests.get(url, timeout=timeout)
            response.raise_for_status()
            return response
        except Exception as e:
            if i == max_retries - 1:
                print(f"Failed after {max_retries} attempts: {e}")
                raise
            wait_time = 2 ** i  # 指数退避
            print(f"Retry {i+1} after {wait_time}s...")
            time.sleep(wait_time)
```

**Bash示例**：
```bash
# 重试机制
for i in {1..3}; do
    curl -f -s "https://api.example.com/data" && break
    [ $i -lt 3 ] && sleep $((2**i))
done
```

### 4. 使用流式处理
**大文件处理**：
```python
# ❌ 全部加载到内存
with open('huge.json') as f:
    data = json.load(f)  # 可能几百MB
    process(data)

# ✅ 流式读取（适用于JSON Lines格式）
with open('huge.jsonl') as f:
    for line in f:
        item = json.loads(line)
        process(item)

# ✅ 分块读取CSV
import pandas as pd
for chunk in pd.read_csv('huge.csv', chunksize=1000):
    process(chunk)
    save_chunk(chunk)
```

## 效率提升技巧

### 1. 复用代码片段
**创建代码库**：
```
scripts/
├── common/
│   ├── csv_utils.py      # CSV处理工具
│   ├── json_utils.py     # JSON处理工具
│   └── file_utils.py     # 文件操作工具
├── templates/
│   ├── data_processing.py
│   └── report_generator.py
└── snippets/
    ├── retry_logic.py
    └── batch_processor.py
```

**复用示例**：
```python
# scripts/common/csv_utils.py
def clean_csv(input_path, output_path, required_columns):
    """标准CSV清洗流程"""
    # 可复用的清洗逻辑
    pass

# 在新任务中复用
from scripts.common.csv_utils import clean_csv
clean_csv('data/raw.csv', 'data/clean.csv', ['name', 'email'])
```

### 2. 使用模板
**任务Prompt模板库**：
```
prompts/
├── file_processing/
│   ├── csv_cleaning.md
│   ├── json_transform.md
│   └── batch_rename.md
├── data_analysis/
│   ├── basic_stats.md
│   └── report_generation.md
└── content_generation/
    ├── readme_template.md
    └── api_docs_template.md
```

**模板示例**（csv_cleaning.md）：
```markdown
# CSV清洗任务模板

**目标**：清洗[文件名]

**输入**：
- 文件：[路径]
- 格式：CSV，包含[列名]

**步骤**：
1. 读取CSV
2. 删除[条件]的行
3. 转换[列名]格式为[目标格式]
4. 去重基于[列名]
5. 排序基于[列名]

**输出**：
- 文件：[路径]
- 统计信息：处理前后行数对比
```

### 3. 建立个人知识库
**知识库结构**：
```
knowledge/
├── claude_code/
│   ├── capabilities.md     # 能力清单
│   ├── limitations.md      # 限制清单
│   └── tips.md             # 技巧总结
├── common_tasks/
│   ├── data_cleaning.md
│   ├── api_integration.md
│   └── git_workflow.md
└── troubleshooting/
    ├── error_solutions.md
    └── failed_cases.md
```

### 4. 记录失败案例
**失败案例模板**：
```markdown
# 失败案例记录

## 案例1：大文件处理卡顿
- **日期**：2024-11-18
- **任务**：处理500MB的CSV文件
- **问题**：读取文件时卡顿，最终超时
- **原因**：文件过大，一次性加载内存
- **解决**：改用分批读取，每批5000行
- **教训**：>100MB文件必须分批处理
```

## 常见错误与避免

### 错误1：任务描述过于笼统
**问题表现**：
- Claude询问很多细节
- 结果不符合预期
- 需要多次修改

**解决方案**：
```
使用5W1H框架：
- What：具体要做什么
- Why：为什么要这样做（帮助理解意图）
- Who：数据来源/使用者
- When：什么时候执行/数据时间范围
- Where：文件位置
- How：具体步骤和方法
```

### 错误2：一次性操作太多
**问题表现**：
- 执行卡住
- 部分完成后出错
- 难以定位问题

**解决方案**：
```
分解规则：
- 单步<5分钟
- 单批<100个文件
- 单文件<100MB
- 总步骤<10步
```

### 错误3：依赖外部网络资源
**问题表现**：
- 网络请求失败
- 超时错误
- 无法访问目标网站

**解决方案**：
```
Plan A：使用WebFetch/WebSearch（如可用）
Plan B：手动下载后提供文件
Plan C：使用白名单内的备用资源
Plan D：生成脚本供用户本地执行
```

### 错误4：没有备份原始文件
**问题表现**：
- 操作失误后无法恢复
- 数据丢失
- 需要重新获取数据

**解决方案**：
```bash
# 操作前备份
cp -r data/原始/ backup/原始_20241118/

# 或使用Git
git add data/
git commit -m "保存修改前的状态"

# 操作完成后验证
# 如果失败，从备份恢复
```

### 错误5：路径错误
**问题表现**：
- 文件找不到
- 读取错误
- 写入位置错误

**解决方案**：
```bash
# 操作前确认
ls data/raw/        # 确认目录存在
ls data/raw/*.csv   # 确认文件存在
pwd                 # 确认当前目录

# 使用相对路径
./data/raw/file.csv

# 避免硬编码
INPUT_DIR="data/raw"
OUTPUT_DIR="data/processed"
```

## 高级技巧

### 1. 并行处理（受限）
虽然工具调用是串行的，但可以在单个命令中实现伪并行：

```bash
# 多个独立命令并行
(process1 file1.csv > output1.txt) & \
(process2 file2.csv > output2.txt) & \
(process3 file3.csv > output3.txt) & \
wait

# Python多进程
python -c "
from multiprocessing import Pool
def process_file(f):
    # 处理逻辑
    pass

with Pool(4) as p:
    p.map(process_file, files)
"
```

### 2. 使用Task工具探索
**何时使用Task工具**：
- 不确定文件位置
- 需要理解代码库结构
- 开放式搜索

**示例**：
```
不要：直接Grep搜索"user authentication"
要做：使用Task工具（Explore agent）探索认证相关代码

Task prompt：
"探索这个代码库中用户认证的实现方式，
找到相关文件、函数和配置，thoroughness: medium"
```

### 3. 增量式开发
**策略**：
```
第1轮：最小可行版本
- 基本功能
- 简单验证
- 无错误处理

第2轮：增强功能
- 添加错误处理
- 优化性能
- 添加日志

第3轮：完善细节
- 边界情况处理
- 文档注释
- 测试用例
```

### 4. 使用TodoWrite追踪
**TodoWrite最佳实践**：
```
1. 任务启动时创建todo列表
2. 每个待办具体可执行
3. 包含验收标准
4. 标记为in_progress前确认理解
5. 完成后立即标记completed
6. 遇到阻塞时添加新todo说明问题
```

## 检查清单

### 任务开始前
- [ ] 任务可以在<10分钟内完成？（否则需分解）
- [ ] 输入输出路径明确？
- [ ] 有清晰的验收标准？
- [ ] 错误处理策略明确？
- [ ] 已备份重要文件？
- [ ] 网络依赖最小化？

### 执行过程中
- [ ] 每步输出是否符合预期？
- [ ] 中间结果已保存？
- [ ] 遇到错误是否及时调整？
- [ ] Token使用是否合理？
- [ ] 是否有意外的长时间等待？

### 任务完成后
- [ ] 最终输出是否正确？
- [ ] 临时文件已清理？
- [ ] 可复用的脚本已保存？
- [ ] 失败经验已记录？
- [ ] Git提交信息清晰？
- [ ] 文档已更新（如需要）？
