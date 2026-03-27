# 四类数据源详细读取代码

## 1. 自研平台（CSV / XLSX）

```python
import pandas as pd
import numpy as np

def load_inhouse(filepath):
    """
    自研平台数据读取
    结构：前2行为说明信息，第3行为真正列名，第4行起为数据
    系统字段：编号/开始结束时间/游戏id/user id/open id/
              LTV3/LTV5/LTV7/LTV30/累计充值/创号日期/
              最近一次登录日期/次日登录/三日登录/五日登录/七日登录/15日登录/region
    """
    if filepath.endswith('.csv'):
        for enc in ['utf-8', 'utf-8-sig', 'gbk', 'gb2312']:
            try:
                df_raw = pd.read_csv(filepath, encoding=enc, skiprows=1, header=0)
                break
            except:
                continue
    else:
        df_raw = pd.read_excel(filepath, sheet_name=0, skiprows=1, header=0)

    # 第0行是真正的列名
    real_cols = df_raw.iloc[0].tolist()
    df = df_raw.iloc[1:].copy()
    df.columns = real_cols
    df = df.reset_index(drop=True)

    print(f"✓ 自研平台数据读取成功，有效样本：{len(df)} 份，共 {len(df.columns)} 列")
    print(f"  系统字段：{list(df.columns[:19])}")
    print(f"  问卷题目列数：{len(df.columns) - 19}")
    return df

# 系统字段列表（供后续区分题目列使用）
INHOUSE_SYSTEM_COLS = [
    '编号', '开始答题时间', '结束答题时间', '游戏id', 'user id', 'open id',
    'LTV3', 'LTV5', 'LTV7', 'LTV30', '累计充值', '创号日期',
    '最近一次登录日期', '次日登录', '三日登录', '五日登录', '七日登录', '15日登录', 'region'
]

# 留存字段（用于留存交叉分析）
RETENTION_COLS = ['次日登录', '三日登录', '五日登录', '七日登录', '15日登录']

# 付费字段（用于付费交叉分析）
PAYMENT_COLS = ['LTV3', 'LTV5', 'LTV7', 'LTV30', '累计充值']
```

---

## 2. 问卷星（XLSX）

```python
def load_wenjuanxing(filepath):
    """
    问卷星数据读取
    结构：第1行即为列名，无额外说明行
    系统字段：序号/提交答卷时间/所用时间/来源/来源详情/来自IP（前6列）
    多选题：每选项一列，1=选中，0=未选
    单选题：数字编码（1/2/3...对应第几个选项），需结合问卷题目还原文字
    量表题：直接是数字分值
    填空题：文字内容
    """
    df = pd.read_excel(filepath, sheet_name=0)
    print(f"✓ 问卷星数据读取成功，有效样本：{len(df)} 份，共 {len(df.columns)} 列")
    return df

WENJUANXING_SYSTEM_COLS = ['序号', '提交答卷时间', '所用时间', '来源', '来源详情', '来自IP']
```

---

## 3. SurveyMonkey（XLSX）

```python
def load_surveymonkey(filepath):
    """
    SurveyMonkey数据读取
    结构：第0行为空行，第1行起为真正数据
    系统字段：Respondent ID/Collector ID/Start Date/End Date/
              IP Address/Email Address/First Name/Last Name/Custom Data 1（前9列）
    多选题：每选项一列（Unnamed列），选了=选项文字，未选=NaN
    单选题：直接存选项文字
    """
    df_raw = pd.read_excel(filepath, sheet_name=0)
    # 跳过第0行空行
    df = df_raw.iloc[1:].copy().reset_index(drop=True)
    print(f"✓ SurveyMonkey数据读取成功，有效样本：{len(df)} 份，共 {len(df.columns)} 列")
    return df

SM_SYSTEM_COLS = [
    'Respondent ID', 'Collector ID', 'Start Date', 'End Date',
    'IP Address', 'Email Address', 'First Name', 'Last Name', 'Custom Data 1'
]
```

---

## 4. yiyo供应商（XLSX）

```python
def load_yiyo(filepath):
    """
    yiyo数据读取
    直接读取"文本数据"sheet
    系统字段：transid/Time Started(UTC)/Time Finished(UTC)/Country/US State/US County（前6列）
    多选题：题目_选项名 各自一列，Selected=选中，Not Selected=未选
    单选题：选项文字，_oe结尾列存填空内容
    列名格式：Q1: 题目文字_选项文字
    """
    df = pd.read_excel(filepath, sheet_name='文本数据')
    print(f"✓ yiyo数据读取成功，有效样本：{len(df)} 份，共 {len(df.columns)} 列")
    return df

YIYO_SYSTEM_COLS = [
    'transid', 'Time Started(UTC)', 'Time Finished(UTC)',
    'Country', 'US State', 'US County'
]
```

---

## 5. 未知格式处理流程

```python
def detect_source_type(filepath):
    """自动识别数据来源类型"""
    try:
        if filepath.endswith('.csv'):
            df = pd.read_csv(filepath, nrows=5, encoding='utf-8')
        else:
            df = pd.read_excel(filepath, nrows=5)

        all_cols = ' '.join([str(c) for c in df.columns] +
                            [str(v) for v in df.iloc[0].tolist()])

        if any(k in all_cols for k in ['游戏id', 'LTV3', '累计充值', '次日登录']):
            return 'inhouse'
        elif any(k in all_cols for k in ['提交答卷时间', '来自IP', '所用时间']):
            return 'wenjuanxing'
        elif any(k in all_cols for k in ['Respondent ID', 'Collector ID']):
            return 'surveymonkey'
        elif any(k in all_cols for k in ['transid', 'Time Started(UTC)']):
            return 'yiyo'
        else:
            return 'unknown'
    except Exception as e:
        return 'unknown'
```

**unknown时的处理步骤：**
1. 打印前5行数据结构给使用者看
2. 询问："这份数据中，多选题的值是怎么存储的？单选题是存文字还是数字？"
3. 使用者无法回答时，根据以下规则猜测并告知假设：
   - 值只有0/1 → 猜测多选题0/1格式
   - 值是A./B./C. → 猜测自研平台格式
   - 值是Selected/Not Selected → 猜测yiyo格式
   - 值是选项文字 → 猜测SurveyMonkey格式
