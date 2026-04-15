---
name: resume-exam-filler
description: 从简历 PDF 中提取信息，批量填入笔试考生信息模板 Excel。当用户提到解析简历、填写考生信息、提取简历数据填表等场景时触发。
---

# 简历考生信息填写

从简历文件夹中读取 PDF 简历，提取关键信息后填入考生信息模板 Excel。

## 工作流程

### 第一步：确认目标目录

用户指定的简历文件夹路径（如 `~/Desktop/简历/工艺工程师`）和模板文件路径（如 `~/Desktop/有鹿智能Java高级开发工程师笔试新增考生模板.xlsx`）。

### 第二步：读取模板结构

用 openpyxl 读取模板，确认列顺序：
```python
import openpyxl
wb = openpyxl.load_workbook('模板.xlsx')
ws = wb.active
for row in ws.iter_rows(min_row=1, max_row=2, values_only=True):
    print(row)
```

### 第三步：提取简历 PDF 文本

用 pdfplumber 批量提取 PDF 内容：
```python
import pdfplumber, os, glob

folder = '简历目录'
files = sorted(glob.glob(os.path.join(folder, '*.pdf')))
results = []
for fp in files:
    text = ''
    with pdfplumber.open(fp) as pdf:
        for page in pdf.pages:
            t = page.extract_text()
            if t: text += t
    results.append((os.path.basename(fp), text))

# 保存供后续分析
with open('/tmp/resumes.txt', 'w') as f:
    for fname, text in results:
        f.write(f'=== {fname} ===\n{text[:6000]}\n\n')
```

### 第四步：提取关键字段

#### 电话号码
- 匹配：`1[3-9]\d[\s\-]?\d{4}[\s\-]?\d{4}` 或 `1[3-9]\d{10}`
- 提取后**统一转为 11 位纯数字**：`re.sub(r'\D', '', phone)`
- 过滤位数不足 11 位的结果

#### 邮箱
- 匹配：`[\w.\-]+@[\w.\-]+\.\w+`
- 清理尾随噪声字符（如 `...@qq.com4` → `...@qq.com`）
- 用 `re.sub(r'(.)\1{2,}', r'\1', email)` 清理字符重复噪声

#### 学历
- 搜索关键词：`大专|本科|硕士|博士|大学本科|硕士研究生`
- 按优先级：博士 > 硕士 > 本科 > 大专
- 标准化：将"大学本科"统一改为"本科"

#### 学校
- 优先从教育背景行提取（格式：`YYYY.YY - YYYY.YY 学校名` 或 `学校名 YYYY - YYYY`）
- PDF 含字符噪声（如`中中国国科科学学院院大大学`）→ 用 `re.sub(r'(.)\1{2,}', r'\1', text)` 清理后再提取
- 手动已知修正表：
  ```python
  SCHOOL_OVERRIDES = {
      '刘先生': '中国科学院大学',
      '李先生': '河海大学',
  }
  ```

#### 姓名提取（重难点）

**优先级 1：文件名字段含真实姓名**
- 文件名格式 `【职位_城市薪资】姓名 X年.pdf`，直接提取"姓名"部分
- **过滤**：若提取结果为"先生/女士/未知"等占位符 → 降级到下述方法

**优先级 2：PDF 文本中定位**
- 噪声 PDF → 先用 `re.sub(r'(.)\1{2,}', r'\1', text)` 清理重复字符
- 从电话号位置往前 150 字符内查找最近的中文姓名
- 匹配 `姓\s*[名甚]:\s*([\u4e00-\u9fa5]{2,4})` 模式（清理后文本中）

**已知姓名替换表**（文件名为"先生/女士"时使用）
```python
KNOWN_NAME_OVERRIDES = {
    '刘先生': '刘锦涛',
    '朱先生': '朱未斌',
    '罗先生': '罗京',
    '李先生': '李子军',
}
```

**姓名全中文校验**：填入前检查姓名是否含中文字符，若为拼音/英文需从 PDF 文本重新提取真实姓名

#### 考生编号
- 优先从文件名提取
- **若用户说明"考生编号与电话号码一致"**，则直接使用电话号码作为考生编号

### 第五步：填入 Excel

```python
import openpyxl
wb = openpyxl.load_workbook('模板.xlsx')
ws = wb.active

# 检查是否已有该姓名（避免重复）
existing = {row[2].value for row in ws.iter_rows(min_row=2, values_only=True)}

for record in candidates:
    if record['name'] not in existing:
        ws.append([试卷名, 考生编号, 姓名, 手机号, 邮箱, 职位, 学历, 学校])

wb.save('模板.xlsx')
```

### 第六步：验证与质检

```python
wb = openpyxl.load_workbook('模板.xlsx')
ws = wb.active
for row in ws.iter_rows(values_only=True):
    print(row)
```

## 数据质量规范

| 字段 | 规范 |
|------|------|
| 电话号码 | 11 位纯数字，如 `17352622556` |
| 邮箱 | 标准格式，清理噪声字符后填入 |
| 学历 | 只能是：大专/本科/硕士/博士 |
| 姓名 | 必须含中文，非中文姓名从 PDF 重新提取 |
| 学校 | 完整校名，清理字符重复噪声 |

## 缺失信息处理

### 标记黄色（仅姓名列）

填表后，识别有缺失字段的候选人，仅将其**姓名列**标黄，无需标记其他列：

```python
from openpyxl.styles import PatternFill
yellow = PatternFill(start_color='FFFF00', end_color='FFFF00', fill_type='solid')

for row in ws.iter_rows(min_row=2):
    for cell in row:
        if not cell.value or str(cell.value).strip() == '':
            row[2].fill = yellow  # 仅姓名列标黄
            break
```

### 打开缺失信息候选人的简历

当存在缺失信息候选人时，**不复制文件**，直接用 `open` 命令同时打开其简历 PDF 供人工补录：

```python
import os, glob

folder = '简历目录'
files = sorted(glob.glob(os.path.join(folder, '*.pdf')))

# 找出缺失信息候选人的简历文件
missing_names = {'任建轩', '孙崇高', '李吉', '段霁航'}
target_files = []
for fp in files:
    for name in missing_names:
        if name in os.path.basename(fp):
            target_files.append(fp)
            break

# 同时打开（macOS）
for fp in target_files:
    os.system(f'open "{fp}"')
```

## 学历字段标准化

| 原文 | 标准 |
|------|------|
| 大学本科 | 本科 |
| 本科 | 本科 |
| 硕士研究生 / 硕士 | 硕士 |
| 博士研究生 / 博士 | 博士 |
| 大专 / 专科 | 大专 |

## 注意事项

- 考生编号：优先从文件名提取；若用户明确说明"考生编号与电话号码一致"，则使用电话作为编号
- PDF 文字识别有噪声时，忽略无关字符，提取有效信息
- 模板文件保存在桌面，与简历文件夹平级
- 缺失信息候选人简历**不复制文件，直接打开原文件**供人工补录
