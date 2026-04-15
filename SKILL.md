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
import pdfplumber, os

folder = '简历目录'
files = sorted(os.listdir(folder))
for f in files:
    if f.endswith('.pdf'):
        with pdfplumber.open(os.path.join(folder, f)) as pdf:
            for page in pdf.pages:
                text = page.extract_text()
                if text:
                    print(text[:3000])
```

### 第四步：提取关键字段

从文本中正则匹配以下字段：
- **电话**：`1[3-9]\d{9}` 或 `1[3-9]\d{1,4}[-\s]?\d{4}[-\s]?\d{4}`，提取后统一转为 11 位纯数字
- **邮箱**：`[\w.-]+@[\w.-]+\.\w+`
- **学历**：搜索"大专|本科|硕士|博士|大学本科|硕士研究生"，按"博士 > 硕士 > 本科 > 大专"优先级取最高
- **学校**：教育背景行中提取
- **考生编号**：从文件名或模板要求中获取，通常文件名包含（如 `J476526`）；**若用户说明考生编号与电话号码一致，则用电话作为考生编号**

#### 姓名提取（重难点）

PDF 简历常有字符噪声导致姓名识别困难，按以下优先级处理：

**优先级 1：文件名字段含真实姓名**
- 文件名格式 `【职位_城市薪资】姓名 X年.pdf`，直接提取"姓名"部分
- **过滤**：若提取结果为"先生/女士/未知"等占位符 → 降级到下述方法

**优先级 2：PDF 文本中定位**
- 噪声 PDF（含重复字符如"基基本本"、双字节噪声）→ 先用 `re.sub(r'(.)\1{2,}', r'\1', text)` 清理
- 从电话号位置往前 150 字符内查找最近的中文姓名
- 匹配 `姓\s*[名甚]:\s*([\u4e00-\u9fa5]{2,4})` 模式（清理后文本中）

**已知姓名替换表**（文件名为"先生/女士"时使用）
```python
KNOWN_OVERRIDES = {
    '刘先生': '刘锦涛',
    '朱先生': '朱未斌',
    '罗先生': '罗京',
    '李先生': '李子军',
}
```

### 第五步：填入 Excel

```python
import openpyxl
wb = openpyxl.load_workbook('模板.xlsx')
ws = wb.active

# 删除原有示例行（如有）
# ws.delete_rows(2)

data = [
    ('试卷名称', '考生编号', '姓名', '手机号', '邮箱', '职位', '学历', '学校'),
    # ... 每位考生一行
]
for row_data in data:
    ws.append(row_data)

wb.save('模板.xlsx')
```

### 第六步：验证结果

```python
wb = openpyxl.load_workbook('模板.xlsx')
ws = wb.active
for row in ws.iter_rows(values_only=True):
    print(row)
```

## 学历字段标准化

将简历原文中的学历描述统一转换为标准格式：

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
