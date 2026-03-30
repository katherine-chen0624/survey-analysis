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

## 0. 答题时长计算（无效样本剔除，所有平台通用）

⚠️ 时间字段可能是 datetime 对象或字符串，必须兼容两种格式，不得假设类型。

```python
from datetime import datetime as dt

def parse_dt(v):
    """兼容 datetime 对象和字符串两种格式"""
    if isinstance(v, dt):
        return v
    if isinstance(v, str):
        try:
            return dt.strptime(v.strip()[:19], '%Y-%m-%d %H:%M:%S')
        except:
            return None
    return None

def get_secs(row, start_col, end_col):
    """计算答题时长（秒）"""
    s = parse_dt(row[start_col])
    e = parse_dt(row[end_col])
    if s and e:
        return abs((e - s).total_seconds())
    return None

# 使用示例：
df['答题时长(秒)'] = df.apply(
    lambda row: get_secs(row, '开始答题时间', '结束答题时间'), axis=1
)

# ⚠️ 过滤后必须校验（防止静默失败）：
n_raw = len(df)
df_valid = df[df['答题时长(秒)'] >= 60]
n_invalid = n_raw - len(df_valid)

if n_invalid == 0 and n_raw > 20:
    # 过滤完全未生效，大概率是时间字段格式异常
    raise ValueError(
        f"⚠️ 时长过滤未生效（N_INVALID=0，N_RAW={n_raw}），"
        f"时间字段格式异常，请检查开始/结束时间列的实际内容。"
        f"禁止在未确认过滤生效的情况下继续执行。"
    )

print(f"原始数据：{n_raw} 份，剔除无效（<60秒）：{n_invalid} 份，有效N：{len(df_valid)}")
```

**各平台时间列名对照：**
| 平台 | 开始时间列 | 结束时间列 |
|-----|----------|----------|
| 自研平台 | `开始答题时间` | `结束答题时间` |
| 问卷星 | 用`所用时间`列直接提取秒数 | — |
| SurveyMonkey | `Start Date` | `End Date` |
| yiyo | `Time Started(UTC)` | `Time Finished(UTC)` |

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

## 2. 问卷星 — 多选题解析（重要：特殊分隔符）

⚠️ **问卷星多选题数据格式与其他平台完全不同！**

问卷星的多选题把所有选中选项合并在**一个单元格**里，
用特殊字符 `┋`（U+250B，不是普通竖线 | ）分隔，例如：
```
MOBA类（王者荣耀...）┋MMORPG类（杖剑传说...）┋沙盒类（我的世界...）
```

**禁止把整行字符串当作一个选项统计！必须先用 `┋` 切分。**

⚠️ **问卷星导出时可能出现编码损坏**，导致选项文字中出现乱码字符（如 `◇◇`、`??`），
匹配失败的残片**不得单独列出**，应归入 Other 或静默丢弃。

```python
import re
import unicodedata

def clean_option_text(s):
    """
    清洗选项文字：
    1. 去除乱码字符（替换字符 U+FFFD、未知字符等）
    2. 统一全半角括号
    3. 去除多余空格
    """
    # 去除替换字符和控制字符
    s = re.sub(r'[\ufffd\x00-\x1f\x7f]', '', s)
    # 统一全角括号为半角（可选，视题目文件格式而定）
    s = s.replace('（', '(').replace('）', ')')
    s = s.strip()
    return s

def match_option(s, question_options, threshold=0.6):
    """
    多级匹配：精确 → 包含 → 前缀 → 编辑距离 → 静默丢弃
    threshold: 最短前缀匹配字符数比例
    """
    s_clean = clean_option_text(s)

    # 第一级：精确匹配
    for opt in question_options:
        if s_clean == clean_option_text(opt):
            return opt

    # 第二级：包含匹配（s 包含在 opt 里，或 opt 包含在 s 里）
    for opt in question_options:
        opt_clean = clean_option_text(opt)
        if s_clean in opt_clean or opt_clean in s_clean:
            return opt

    # 第三级：前缀匹配（取前N个字符匹配，容忍乱码导致的截断）
    prefix_len = max(6, int(len(s_clean) * threshold))
    s_prefix = s_clean[:prefix_len]
    for opt in question_options:
        opt_clean = clean_option_text(opt)
        if opt_clean.startswith(s_prefix) or s_clean.startswith(opt_clean[:prefix_len]):
            return opt

    # 第四级：编辑距离匹配（容忍1-2个字符的差异，如"3时以内"vs"3小时以内"）
    def edit_distance(a, b):
        """计算两个字符串的编辑距离（Levenshtein）"""
        m, n = len(a), len(b)
        dp = list(range(n + 1))
        for i in range(1, m + 1):
            prev = dp[0]
            dp[0] = i
            for j in range(1, n + 1):
                temp = dp[j]
                if a[i-1] == b[j-1]:
                    dp[j] = prev
                else:
                    dp[j] = 1 + min(prev, dp[j], dp[j-1])
                prev = temp
        return dp[n]

    # 编辑距离阈值：字符串越短容忍度越低，越长容忍度越高
    best_match = None
    best_dist = float('inf')
    for opt in question_options:
        opt_clean = clean_option_text(opt)
        max_len = max(len(s_clean), len(opt_clean))
        # 允许的最大编辑距离：短字符串(≤8)允许1个，长字符串允许2个
        max_dist = 1 if max_len <= 8 else 2
        dist = edit_distance(s_clean, opt_clean)
        if dist <= max_dist and dist < best_dist:
            best_dist = dist
            best_match = opt

    if best_match:
        return best_match

    # 所有匹配失败 → 返回 None，不单独列出
    return None


def parse_wenjuanxing_multi(series, question_options):
    """
    问卷星多选题解析（含乱码容错）
    series: 该题数据列（每格是用┋分隔的多个选项）
    question_options: 从问卷题目文件获取的标准选项列表
    """
    SEPARATOR = '┋'  # 问卷星专用分隔符（U+250B）

    counts = {opt: 0 for opt in question_options}
    n_answered = 0
    unmatched = []  # 记录匹配失败的选项（用于调试）

    for val in series.dropna():
        selected = [s.strip() for s in str(val).split(SEPARATOR) if s.strip()]
        if selected:
            n_answered += 1
            for s in selected:
                matched = match_option(s, question_options)
                if matched:
                    counts[matched] += 1
                else:
                    unmatched.append(s)  # 静默丢弃，不列入结果

    if unmatched:
        print(f"⚠️ 匹配失败的选项（已丢弃，共{len(unmatched)}条）：{set(unmatched)}")

    return counts, n_answered

# 使用：
# counts, n = parse_wenjuanxing_multi(df['Q1'], q1_options)
# 选择率 = counts[opt] / n * 100
```

---

## 2b. 问卷星 — 单选题数字还原

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
    other_col: 选择了Other的标记列
    other_text_col: Other的填写内容列

    ⚠️ 重要：Other选项的值不一定是单字母！
    自研平台普通选项是 A./B. 等字母，
    但 Other__ 列的值可能是：
      - 字母（如 H.）
      - 直接是填写的文字内容
      - 或者与 other_text_col 同列
    所以不能只用字母判断，要用"非空"判断是否选中
    """
    # 统计选择Other的人数
    # 不论值是字母还是文字，非空即代表选中
    if df[other_col].dtype == object:
        # 自研平台/SurveyMonkey：非空=选中（包含字母值和文字内容）
        other_count = df[other_col].notna().sum()
        # 排除明确的空占位符
        other_count = df[other_col].apply(
            lambda x: pd.notna(x) and str(x).strip() not in ['', 'nan', 'None', '0']
        ).sum()
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
        'texts': texts.tolist()
    }
```
