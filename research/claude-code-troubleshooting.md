# Claude Code 故障排查指南

## 常见问题与解决方案

### 问题1：执行卡住/无响应

**症状**：
- 长时间没有任何输出
- Token使用停止增长
- 命令似乎在执行但无进展

**可能原因**：
1. 任务过于复杂（处理数据量太大）
2. 网络请求超时（访问不可达的域名）
3. 处理数据量过大（超出内存限制）
4. 死循环或无限等待
5. Token耗尽

**诊断步骤**：
```bash
# 检查是否有后台进程在运行
ps aux | grep python
ps aux | grep node

# 检查内存使用
free -h

# 检查磁盘空间
df -h
```

**解决方案**：
1. **立即行动**：
   - 如果可能，停止当前任务
   - 保存已完成的部分到文件
   - 标记当前进度位置

2. **简化任务**：
   ```
   原任务：处理1000个文件
   ↓
   新任务：先处理前10个文件验证逻辑
   ```

3. **检查网络依赖**：
   ```
   是否有curl/wget命令？
   → 改用手动提供数据

   是否有WebFetch调用？
   → 检查URL是否可访问
   ```

4. **减小数据批次**：
   ```python
   # 从
   batch_size = 1000
   # 改为
   batch_size = 100
   ```

5. **添加进度输出**：
   ```python
   for i, item in enumerate(items):
       if i % 100 == 0:
           print(f"Processed {i}/{len(items)}")
       process(item)
   ```

**预防措施**：
- 任务开始前估算执行时间
- 设置合理的超时时间
- 使用TodoWrite工具追踪进度
- 定期保存中间结果

---

### 问题2：网络请求失败

**症状**：
- `curl: (6) Could not resolve host`
- `403 Forbidden`
- `Connection timeout`
- `WebFetch error: Request failed`

**可能原因**：
1. 域名不在白名单中
2. 网络连接不稳定
3. API限流
4. URL格式错误

**诊断步骤**：
```bash
# 测试基本网络连通性
ping -c 3 8.8.8.8

# 测试特定域名
curl -I https://api.anthropic.com
curl -I https://github.com

# 检查URL格式
echo $URL | grep -E '^https?://'
```

**解决方案**：

**情况1：域名不在白名单**
```
错误：curl: (6) Could not resolve host: example.com

解决方案A：手动提供内容
→ 用户在浏览器访问并复制内容
→ 粘贴到文件中
→ 处理本地文件

解决方案B：使用WebFetch工具
→ WebFetch可能有不同的白名单
→ 尝试: WebFetch(url="https://example.com", prompt="...")

解决方案C：使用已知可访问的镜像
→ 如文档镜像、CDN等
```

**情况2：请求超时**
```bash
# 增加超时时间和重试
for i in {1..3}; do
    timeout 30 curl -s "https://api.example.com/data" && break
    echo "Retry $i after $((2**i))s..."
    sleep $((2**i))
done
```

**情况3：API限流**
```python
import time

def rate_limited_request(url, delay=2):
    """限速请求"""
    time.sleep(delay)
    return requests.get(url)

# 批量请求时添加延迟
for url in urls:
    result = rate_limited_request(url)
    # 处理结果
```

**预防措施**：
- 任务开始前测试网络访问
- 准备离线替代方案
- 使用缓存避免重复请求
- 设置合理的重试机制

---

### 问题3：文件操作失败

**症状**：
- `FileNotFoundError`
- `PermissionError`
- `No such file or directory`
- `Read tool error`

**可能原因**：
1. 路径错误（拼写错误、相对路径问题）
2. 文件权限不足
3. 文件格式不支持
4. 文件被占用

**诊断步骤**：
```bash
# 确认当前目录
pwd

# 检查文件是否存在
ls -la path/to/file

# 检查文件权限
ls -l path/to/file

# 检查文件类型
file path/to/file

# 查找文件
find . -name "filename*"
```

**解决方案**：

**情况1：路径错误**
```bash
# 错误示例
cat /home/user/data/file.csv  # 绝对路径可能错误

# 正确做法
# 1. 确认当前目录
pwd
# 输出: /home/user/AI-x-Automation

# 2. 使用相对路径
cat data/file.csv

# 3. 或者先列出目录
ls data/
# 确认文件名后再操作
```

**情况2：权限不足**
```bash
# 检查权限
ls -l file.txt
# 输出: -rw-r--r-- 1 user group 1234 Nov 18 10:00 file.txt

# 如果需要修改权限
chmod +r file.txt  # 添加读权限
chmod +w file.txt  # 添加写权限

# 如果是系统文件，操作用户目录副本
cp /etc/config.conf ~/config.conf
# 然后操作 ~/config.conf
```

**情况3：文件格式不支持**
```python
# 错误：尝试直接编辑.docx文件
# 解决：使用转换库

# 安装库
# pip install python-docx

# 转换处理
from docx import Document
doc = Document('file.docx')
# 提取文本
text = '\n'.join([p.text for p in doc.paragraphs])
# 保存为文本
with open('file.txt', 'w') as f:
    f.write(text)
```

**预防措施**：
- 使用相对路径
- 操作前用ls确认文件存在
- 备份重要文件
- 检查文件格式是否支持

---

### 问题4：输出不符合预期

**症状**：
- 结果格式错误
- 数据缺失或重复
- 逻辑错误
- 数量不匹配

**可能原因**：
1. Prompt描述不够清晰
2. 输入数据格式不符合假设
3. 逻辑理解有误
4. 边界情况未处理

**诊断步骤**：
```bash
# 检查输入数据
head -20 input.csv
wc -l input.csv

# 检查输出数据
head -20 output.csv
wc -l output.csv

# 对比差异
diff input.csv output.csv | head -20

# 检查数据格式
file input.csv
file output.csv
```

**解决方案**：

**步骤1：澄清需求**
```
❌ 模糊需求：
"整理这个CSV文件"

✅ 明确需求：
"CSV文件处理需求：
输入：data/users.csv（包含name, email, date列）
操作：
1. 删除email为空的行
2. 将date从MM/DD/YYYY转为YYYY-MM-DD
3. 按date升序排序
输出：data/users_clean.csv
期望：约5000行 → 约4500行（删除了500个无效行）"
```

**步骤2：提供输出示例**
```
期望输出示例（前3行）：
name,email,date
John Doe,john@example.com,2024-01-15
Jane Smith,jane@example.com,2024-01-16
Bob Johnson,bob@example.com,2024-01-17
```

**步骤3：逐步验证**
```python
# 不要一次性完成所有步骤
# 而是分步验证

# 步骤1：读取并查看数据
import pandas as pd
df = pd.read_csv('input.csv')
print(f"Total rows: {len(df)}")
print(df.head())
# 确认正确后继续

# 步骤2：删除空email
df_filtered = df[df['email'].notna()]
print(f"After filtering: {len(df_filtered)}")
# 确认正确后继续

# 步骤3：转换日期
# ... 依此类推
```

**步骤4：检查边界情况**
```python
# 检查异常值
print("Null values:")
print(df.isnull().sum())

print("\nDuplicate rows:")
print(df.duplicated().sum())

print("\nData types:")
print(df.dtypes)

print("\nSample values:")
print(df.sample(5))
```

**预防措施**：
- 使用明确的Prompt模板
- 提供输入输出示例
- 逐步验证每个操作
- 检查中间结果
- 对比预期数量

---

### 问题5：包/库无法使用

**症状**：
- `ModuleNotFoundError: No module named 'xxx'`
- `ImportError`
- `Command not found`

**可能原因**：
1. 包未安装
2. 包名错误
3. 版本不兼容
4. 依赖冲突

**诊断步骤**：
```bash
# 检查Python包
pip list | grep package_name
pip show package_name

# 检查命令是否存在
which command_name
command_name --version

# 检查Python版本
python --version
python3 --version
```

**解决方案**：

**情况1：包未安装**
```bash
# Python包
pip install package_name

# 如果权限不足
pip install --user package_name

# Node.js包
npm install package_name

# 系统包（如果有权限）
apt-get install package_name
```

**情况2：包名错误**
```bash
# 常见错误
pip install beautifulsoup  # ❌ 错误
pip install beautifulsoup4 # ✅ 正确

pip install sklearn        # ❌ 错误
pip install scikit-learn   # ✅ 正确
```

**情况3：使用内置库替代**
```python
# 避免安装第三方库，使用内置库

# CSV处理
import csv  # 内置，无需安装

# JSON处理
import json  # 内置

# HTTP请求
import urllib.request  # 内置（虽然不如requests好用）

# 日期处理
from datetime import datetime  # 内置
```

**情况4：检查依赖并创建requirements.txt**
```bash
# 生成依赖列表
pip freeze > requirements.txt

# 在新环境安装
pip install -r requirements.txt
```

**预防措施**：
- 任务开始前检查依赖
- 优先使用内置库
- 提供requirements.txt
- 使用虚拟环境隔离

---

### 问题6：Git操作失败

**症状**：
- `fatal: not a git repository`
- `error: failed to push`
- `merge conflict`
- `Permission denied (publickey)`

**可能原因**：
1. 不在Git仓库中
2. 分支名称不符合规范
3. 网络问题
4. 权限问题
5. 冲突未解决

**诊断步骤**：
```bash
# 检查Git状态
git status

# 检查当前分支
git branch

# 检查远程仓库
git remote -v

# 查看最近提交
git log -3 --oneline

# 检查冲突
git diff
```

**解决方案**：

**情况1：分支命名错误**
```bash
# ❌ 错误：不符合命名规范
git checkout -b my-feature
git push origin my-feature
# Error: 分支必须以'claude/'开头

# ✅ 正确：符合规范
git checkout -b claude/my-feature-sessionid
git push -u origin claude/my-feature-sessionid
```

**情况2：Push失败需要重试**
```bash
# 使用指数退避重试
for i in {1..4}; do
    git push -u origin branch-name && break
    echo "Push failed, retry $i after $((2**i))s..."
    sleep $((2**i))
done
```

**情况3：本地更改未提交**
```bash
# 检查未提交的更改
git status

# 提交更改
git add .
git commit -m "Your commit message"

# 然后push
git push
```

**情况4：合并冲突**
```bash
# 查看冲突文件
git status

# 手动解决冲突或取消合并
git merge --abort  # 取消合并

# 或使用策略
git merge -X ours branch-name   # 优先使用当前分支
git merge -X theirs branch-name # 优先使用目标分支
```

**预防措施**：
- 遵循分支命名规范
- 提交前先pull
- 避免复杂的merge/rebase
- 小步提交，频繁push

---

## 应急处理流程

### 当任务卡住时

```
┌─────────────────────────────────┐
│ Step 1: 保存已完成的部分         │
├─────────────────────────────────┤
│ - 要求Claude输出当前进度到文件   │
│ - 保存中间结果                   │
│ - 记录最后成功的步骤             │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 2: 标记断点                 │
├─────────────────────────────────┤
│ - 在文件中标注"进行到XX步骤"     │
│ - 记录待处理项                   │
│ - 保存错误信息（如有）           │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 3: 分析原因                 │
├─────────────────────────────────┤
│ - 任务是否太复杂？               │
│ - 是否有网络请求？               │
│ - 数据量是否过大？               │
│ - 是否Token耗尽？                │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 4: 调整策略                 │
├─────────────────────────────────┤
│ - 简化任务（减小批次）           │
│ - 去除网络依赖                   │
│ - 分段执行                       │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 5: 从断点继续               │
├─────────────────────────────────┤
│ - 加载保存的进度                 │
│ - 继续剩余步骤                   │
│ - 每步验证结果                   │
└─────────────────────────────────┘
```

### 当输出错误时

```
┌─────────────────────────────────┐
│ Step 1: 停止执行                 │
├─────────────────────────────────┤
│ - 不要继续错误的方向             │
│ - 保存错误结果用于分析           │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 2: 检查输入                 │
├─────────────────────────────────┤
│ - 输入数据是否正确？             │
│ - 格式是否符合预期？             │
│ - 是否有异常值？                 │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 3: 检查Prompt               │
├─────────────────────────────────┤
│ - 是否有歧义？                   │
│ - 是否遗漏关键信息？             │
│ - 输出示例是否清晰？             │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 4: 小范围测试               │
├─────────────────────────────────┤
│ - 用1-2个样本验证逻辑            │
│ - 检查每步中间结果               │
│ - 确认逻辑正确                   │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Step 5: 全量执行                 │
├─────────────────────────────────┤
│ - 逻辑正确后处理全部数据         │
│ - 分批执行避免卡顿               │
│ - 保存中间结果                   │
└─────────────────────────────────┘
```

---

## 性能优化清单

### 执行前检查
```
性能检查清单：
┌────────────────────────────────────────────┐
│ □ 任务可以分解为更小的步骤吗？            │
│ □ 是否需要网络请求？能否离线处理？        │
│ □ 数据量是否<100MB？                       │
│ □ 预计执行时间是否<5分钟？                │
│ □ 是否有清晰的输入输出定义？              │
│ □ 是否设置了错误处理？                    │
│ □ 是否准备了中间结果保存位置？            │
└────────────────────────────────────────────┘
```

### 执行中监控
```
监控检查清单：
┌────────────────────────────────────────────┐
│ □ 每个步骤是否有明确输出？                │
│ □ 中间结果是否符合预期？                  │
│ □ 是否出现长时间无响应（>2分钟）？        │
│ □ 错误信息是否及时处理？                  │
│ □ Token使用是否合理？                      │
│ □ 是否需要调整批次大小？                  │
└────────────────────────────────────────────┘
```

### 执行后总结
```
总结检查清单：
┌────────────────────────────────────────────┐
│ □ 最终输出是否正确？                      │
│ □ 数量是否符合预期？                      │
│ □ 哪些步骤可以优化？                      │
│ □ 是否需要记录此workflow？                │
│ □ 失败经验已记录？                        │
│ □ 可复用的脚本已保存？                    │
│ □ 临时文件已清理？                        │
│ □ 下次如何改进？                          │
└────────────────────────────────────────────┘
```

---

## 诊断决策树

```
任务失败？
│
├─ 执行卡住
│  ├─ >10分钟无输出
│  │  → 任务太复杂，需分解
│  │
│  ├─ Token快速增长但无进展
│  │  → 可能在处理大文件，改用分批
│  │
│  └─ 网络请求中
│     → 检查域名白名单，改用手动提供
│
├─ 文件错误
│  ├─ FileNotFoundError
│  │  → ls 确认路径，使用相对路径
│  │
│  ├─ PermissionError
│  │  → 检查权限，操作用户目录
│  │
│  └─ 格式不支持
│     → 使用转换库或API
│
├─ 输出错误
│  ├─ 格式不对
│  │  → 提供输出示例
│  │
│  ├─ 数据缺失
│  │  → 检查输入数据
│  │
│  └─ 逻辑错误
│     → 小样本测试，逐步验证
│
├─ 网络错误
│  ├─ 域名不可达
│  │  → 手动提供数据
│  │
│  ├─ 超时
│  │  → 设置重试机制
│  │
│  └─ 限流
│     → 添加延迟，降低频率
│
└─ 依赖错误
   ├─ 模块未找到
   │  → pip install
   │
   ├─ 命令不存在
   │  → 使用内置替代
   │
   └─ 版本冲突
      → 检查兼容性，使用虚拟环境
```

---

## 求助指南

### 什么时候应该调整策略

**信号1：重复失败**
- 同样的任务卡住超过2次
- 使用相同方法失败3次以上
- **行动**：根本性改变方法，而非微调

**信号2：时间过长**
- 执行时间超过10分钟仍无进展
- 多次尝试总时间>30分钟
- **行动**：考虑任务是否适合Claude Code

**信号3：方法用尽**
- 多次尝试不同Prompt仍失败
- 已尝试所有已知方法
- **行动**：寻求替代工具或方案

**信号4：不适合场景**
- 发现任务需要GUI
- 需要持续运行服务
- 需要处理大规模多媒体
- **行动**：使用更合适的工具

### 替代方案矩阵

| 任务类型 | Claude Code | 替代方案 |
|---------|-------------|----------|
| 复杂计算 | ❌ 资源受限 | 本地Python脚本 |
| 图像处理 | ❌ 不支持 | ImageMagick, Pillow, 在线工具 |
| 视频编辑 | ❌ 不支持 | FFmpeg, 在线工具 |
| 大规模爬虫 | ❌ 限制太多 | Scrapy本地运行 |
| 持续监控 | ❌ 无持久化 | 云服务、Cron job |
| 前端调试 | ⚠️ 受限 | 本地开发环境、在线IDE |
| 机器学习训练 | ❌ 资源不足 | Google Colab, AWS SageMaker |
| 数据库管理 | ⚠️ 临时可以 | DBeaver, MySQL Workbench |

### 何时使用其他工具

**使用本地脚本**：
- 计算密集型任务
- 需要特定库或版本
- 需要重复运行

**使用在线工具**：
- 图像/视频编辑（Canva, Cloudinary）
- PDF操作（iLovePDF, Smallpdf）
- 数据可视化（Tableau Public, Google Data Studio）

**使用云服务**：
- 持续运行任务（AWS Lambda, Google Cloud Functions）
- 大规模数据处理（BigQuery, AWS Glue）
- 机器学习（SageMaker, Vertex AI）

**使用专用工具**：
- API测试（Postman, Insomnia）
- Git GUI（GitKraken, SourceTree）
- 数据库（DBeaver, pgAdmin）

---

## 故障排查快速参考

### 常见错误速查表

| 错误信息 | 可能原因 | 快速修复 |
|---------|---------|---------|
| `FileNotFoundError` | 路径错误 | `ls` 确认路径 |
| `ModuleNotFoundError` | 包未安装 | `pip install xxx` |
| `PermissionError` | 权限不足 | 操作 `~/` 目录 |
| `Connection timeout` | 网络问题 | 手动提供数据 |
| `403 Forbidden` | 域名限制 | 使用WebFetch |
| `Command not found` | 未安装命令 | 检查是否可用 |
| `Memory error` | 数据太大 | 分批处理 |
| `Timeout` | 执行太久 | 分解任务 |
| `git push failed` | 网络/权限 | 重试，检查分支名 |
| `Invalid JSON` | 格式错误 | 验证JSON格式 |

### 快速诊断命令

```bash
# 系统诊断
pwd                    # 当前目录
df -h                  # 磁盘空间
free -h                # 内存使用
date                   # 当前时间

# 文件诊断
ls -lah path/          # 列出文件详细信息
file filename          # 查看文件类型
wc -l filename         # 统计行数
head -20 filename      # 查看前20行

# 网络诊断
ping -c 3 8.8.8.8      # 测试网络
curl -I URL            # 测试URL访问

# Git诊断
git status             # 仓库状态
git branch             # 当前分支
git log -3 --oneline   # 最近提交

# Python诊断
python --version       # Python版本
pip list               # 已安装包
pip show package       # 包详情
```

---

## 记录与改进

### 失败案例模板

```markdown
## 案例 X：[简短描述]

**日期**：YYYY-MM-DD
**任务**：[具体任务描述]
**问题**：[遇到的问题]
**表现**：[具体症状]
**原因**：[根本原因分析]
**尝试方案**：
1. [方案1] - 失败/成功
2. [方案2] - 失败/成功
3. [方案3] - 成功 ✓

**最终解决方案**：[详细解决方法]
**教训**：[经验总结]
**预防**：[以后如何避免]
```

### 改进追踪

保持一个改进日志，记录每次优化：

```markdown
## 改进日志

### 2024-11-18：批处理优化
- **改进前**：一次处理1000个文件，经常超时
- **改进后**：每批100个文件，分10次处理
- **效果**：成功率从40%提升到95%

### 2024-11-17：错误处理增强
- **改进前**：遇到错误就停止
- **改进后**：错误跳过并记录到error.log
- **效果**：处理完整性提升，易于调试
```

---

## 总结

**防止问题的三个原则**：
1. **简单化**：任务尽可能简单，分解为小步骤
2. **可见化**：每步都有输出，进度可追踪
3. **可恢复**：保存中间结果，支持断点续传

**解决问题的三个步骤**：
1. **诊断**：准确识别问题根源
2. **隔离**：缩小问题范围，用小样本测试
3. **修复**：应用解决方案，验证效果

**持续改进**：
- 记录每次失败案例
- 总结成功经验
- 建立个人知识库
- 优化常用流程
