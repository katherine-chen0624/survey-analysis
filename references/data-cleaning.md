# 数据清洗与编码详细规则

## 通用预处理

```python
def preprocess(df, system_cols):
    """通用预处理：去除全空行、分离系统字段和题目字段"""
    df = df.dropna(how='all').reset_index(drop=True)
    survey_cols = [c for c in df.columns if c not in system_cols]
    return df, survey_cols
```

---

## 1. 自研平台 — 题目列解析

自研平台多选题的列名格式：`题号.题目文字: 选项文字`
单选/量表题的列名格式：`题号.题目文字`

```python
def parse_inhouse_question(col_name, value, question_options):
    """
    自研平台答案解码
    value: 'A.' / 'B.' / 'C.' ... 或 NaN
    question_options: 该题的选项列表（从问卷题目文件获取）
    """
    if pd.isna(value):
        return None
    letter = str(value).strip().rstrip('.')
    idx = ord(letter.upper()) - ord('A')  # A=0, B=1, C=2...
    if 0 <= idx < len(question_options):
        return question_options[idx]
    return value  # 无法映射时保留原值

def decode_inhouse_scale_11pt(value):
    """11点量表：A→0, B→1 ... K→10"""
    if pd.isna(value):
        return None
    letter = str(value).strip().rstrip('.')
    idx = ord(letter.upper()) - ord('A')
    return idx if 0 <= idx <= 10 else None

def decode_inhouse_scale_5pt(value, question_options):
    """五分量表：A→1, B→2, C→3, D→4, E→5（或按选项顺序）"""
    if pd.isna(value):
        return None
    letter = str(value).strip().rstrip('.')
    idx = ord(letter.upper()) - ord('A')
    return idx + 1 if 0 <= idx <= 4 else None
```

---

## 2. 问卷星 — 单选题数字还原

```python
def decode_wenjuanxing_single(value, question_options):
    """
    问卷星单选题：数字→对应选项文字
    value: 1/2/3/4/5...（对应第几个选项）
    """
    if pd.isna(value):
        return None
    try:
        idx = int(float(value)) - 1  # 1-based → 0-based
        if 0 <= idx < len(question_options):
            return question_options[idx]
        return value
    except:
        return value
```

---

## 3. 11点量表完整编码

```python
LETTER_TO_NUM = {chr(ord('A') + i): i for i in range(11)}  # A=0...K=10

def encode_11pt(series):
    """将A-K转换为0-10"""
    def convert(v):
        if pd.isna(v):
            return None
        s = str(v).strip().rstrip('.').upper()
        return LETTER_TO_NUM.get(s, None)
    return series.apply(convert)

def calc_11pt_score(series):
    """11点量表均分后-5，结果范围-5~5"""
    numeric = pd.to_numeric(series, errors='coerce').dropna()
    if len(numeric) == 0:
        return None
    return round(numeric.mean() - 5, 2)

def calc_nps(series):
    """
    NPS计算
    推荐者：8-10分
    贬损者：0-4分
    NPS% = (推荐者人数 - 贬损者人数) / 总人数
    """
    numeric = pd.to_numeric(series, errors='coerce').dropna()
    n = len(numeric)
    if n == 0:
        return None, None, None, None
    promoters = (numeric >= 8).sum()
    detractors = (numeric <= 4).sum()
    nps = round((promoters - detractors) / n * 100, 1)
    return nps, int(promoters), int(detractors), n
```

---

## 4. 五分量表编码

```python
# 常见文字→数字映射
LIKERT_5_MAP = {
    # 满意度
    '非常满意': 5, '比较满意': 4, '一般': 3, '比较不满意': 2, '非常不满意': 1,
    'Very satisfied': 5, 'Satisfied': 4, 'Neutral': 3,
    'Dissatisfied': 2, 'Very dissatisfied': 1,
    # 喜好度
    '非常喜欢': 5, '比较喜欢': 4, '不喜欢也不讨厌': 3, '比较不喜欢': 2, '非常不喜欢': 1,
    # 同意度
    '非常同意': 5, '比较同意': 4, '中立': 3, '比较不同意': 2, '非常不同意': 1,
    # 重要性
    '非常重要': 5, '比较重要': 4, '一般重要': 3, '不太重要': 2, '完全不重要': 1,
    # 数字直接映射
    '1': 1, '2': 2, '3': 3, '4': 4, '5': 5,
    '1（非常不满意）': 1, '5（非常满意）': 5,
}

def encode_5pt(series, custom_map=None):
    """五分量表编码，支持自定义映射"""
    mapping = {**LIKERT_5_MAP, **(custom_map or {})}
    def convert(v):
        if pd.isna(v):
            return None
        # 尝试直接转数字
        try:
            n = float(v)
            if 1 <= n <= 5:
                return int(n)
        except:
            pass
        # 尝试文字映射
        return mapping.get(str(v).strip(), None)
    return series.apply(convert)

def calc_5pt_score(series):
    """五分量表均分（不-5）"""
    numeric = pd.to_numeric(series, errors='coerce').dropna()
    if len(numeric) == 0:
        return None
    return round(numeric.mean(), 2)
```

---

## 5. Other__ 双列处理

```python
def process_other_column(df, other_col, other_text_col):
    """
    处理Other__双列
    other_col: 选择了Other的标记列（自研平台是字母，问卷星是1/0）
    other_text_col: Other的填写内容列
    """
    # 统计选择Other的人数
    if df[other_col].dtype == object:
        # 自研平台：有字母值=选了
        other_count = df[other_col].notna().sum()
    else:
        # 问卷星：1=选了
        other_count = (df[other_col] == 1).sum()

    total = len(df)
    other_pct = round(other_count / total * 100, 1)

    # 收集填写内容（过滤空值和占位符）
    texts = df[other_text_col].dropna().astype(str)
    texts = texts[~texts.isin(['', '(空)', 'nan', 'None', 'N/A'])]

    return {
        'count': int(other_count),
        'pct': other_pct,
        'texts': texts.tolist()  # 供后续文字归纳
    }
```
