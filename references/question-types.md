# 各题型统计分析详细代码

## 1. 单选题统计

```python
def analyze_single_choice(series, question_options=None, is_demographic=False):
    """
    单选题统计
    is_demographic: 是否为人口统计类题目（年龄/性别/职业/学历/收入/地区）
                   True→不倒序，False→按频率倒序
    """
    counts = series.value_counts(dropna=True)
    total = counts.sum()
    pct = (counts / total * 100).round(1)

    result = pd.DataFrame({
        '选项': counts.index,
        '人数': counts.values,
        '占比(%)': pct.values
    })

    if not is_demographic:
        result = result.sort_values('占比(%)', ascending=False)

    result['有效N'] = total
    return result
```

---

## 2. 多选题统计

```python
def analyze_multi_choice(df, option_cols, base_n=None):
    """
    多选题统计
    option_cols: 该多选题的所有选项列名列表
    base_n: 基数（有效回答人数），默认为总行数

    自研平台：有字母值=选中
    问卷星：1=选中
    SurveyMonkey：有文字=选中
    yiyo：'Selected'=选中
    """
    if base_n is None:
        base_n = len(df)

    results = []
    for col in option_cols:
        col_data = df[col]

        # 根据数据类型判断是否选中
        if col_data.dtype == object:
            # 自研平台/SurveyMonkey/yiyo
            selected = col_data.apply(
                lambda x: x == 'Selected' or  # yiyo
                         (pd.notna(x) and str(x).strip() not in ['', 'nan', '0'])
            ).sum()
        else:
            # 问卷星
            selected = (col_data == 1).sum()

        # 从列名中提取选项文字
        option_text = col.split(': ')[-1] if ': ' in col else col.split('_')[-1]

        results.append({
            '选项': option_text,
            '人数': int(selected),
            '选择率(%)': round(selected / base_n * 100, 1)
        })

    result_df = pd.DataFrame(results)
    result_df = result_df.sort_values('选择率(%)', ascending=False)
    result_df['基数N'] = base_n
    return result_df
```

---

## 3. 11点量表统计

```python
def analyze_11pt_scale(series, is_nps=False):
    """
    11点量表完整统计
    is_nps: 是否为NPS题（含recommend/推荐语义）
    """
    from references.data_cleaning import encode_11pt, calc_11pt_score, calc_nps

    # 先编码为数字
    numeric = encode_11pt(series)
    valid = numeric.dropna()
    n = len(valid)

    if is_nps:
        nps, promoters, detractors, total = calc_nps(valid)
        return {
            'type': 'NPS',
            'NPS%': nps,
            '推荐者(8-10)': promoters,
            '中立(5-7)': int((valid.between(5, 7)).sum()),
            '贬损者(0-4)': detractors,
            '有效N': total,
            # 每分值占比（内部记录，报告只展示NPS%）
            'distribution': _calc_distribution(valid, range(11))
        }
    else:
        score = calc_11pt_score(valid)
        return {
            'type': '11pt_scale',
            '得分(-5~5)': score,
            '有效N': n,
            # 每分值占比（内部记录，报告宏观区只展示最终得分）
            'distribution': _calc_distribution(valid, range(11))
        }

def _calc_distribution(numeric_series, value_range):
    """计算各分值占比"""
    n = len(numeric_series)
    dist = {}
    for v in value_range:
        count = (numeric_series == v).sum()
        dist[v] = {'count': int(count), 'pct': round(count / n * 100, 1)}
    return dist
```

---

## 4. 五分量表统计

```python
def analyze_5pt_scale(series, question_options=None):
    """
    五分量表统计
    既展示均分，又展示每分值占比
    """
    from references.data_cleaning import encode_5pt, calc_5pt_score

    numeric = encode_5pt(series)
    valid = numeric.dropna()
    n = len(valid)
    score = calc_5pt_score(valid)

    # 每分值分布
    distribution = []
    labels = question_options if question_options else ['1', '2', '3', '4', '5']
    for i, label in enumerate(labels, 1):
        count = (valid == i).sum()
        distribution.append({
            '选项': f"{i}（{label}）" if question_options else str(i),
            '人数': int(count),
            '占比(%)': round(count / n * 100, 1)
        })

    return {
        'type': '5pt_scale',
        '均分(1~5)': score,
        '有效N': n,
        'distribution': pd.DataFrame(distribution)
    }

def analyze_matrix_5pt(df, row_items, col_options):
    """
    矩阵单选题（每行独立统计）
    row_items: 行维度列表（如各个功能名称）
    col_options: 列维度（1-5分的标签）
    """
    results = []
    for item, col in zip(row_items, [c for c in df.columns if '...' in c]):
        result = analyze_5pt_scale(df[col])
        result['维度'] = item
        results.append(result)
    return results
```

---

## 5. 填空题统计

```python
def analyze_open_text(series, col_name=''):
    """填空题基础统计"""
    valid = series.dropna().astype(str)
    valid = valid[~valid.isin(['', '(空)', 'nan', 'None', 'N/A'])]
    n = len(valid)

    print(f"\n📝 填空题：{col_name}")
    print(f"  有效回答数：{n}")
    print(f"  平均字数：{valid.str.len().mean():.0f}")

    # 简单关键词频统计（无需额外库）
    all_text = ' '.join(valid.tolist())
    # 返回原始文本供AI归纳分析
    return {
        'type': 'open_text',
        '有效N': n,
        'texts': valid.tolist(),
        'summary_hint': '请对以上文字做主题归类，分正面/负面/建议三类'
    }
```

---

## 6. 留存 & 付费交叉分析专用处理

```python
def prepare_retention_group(df, retention_col):
    """
    将留存列转换为分组标签
    留存列值通常为 1（有留存）/ 0（无留存）或 NaN
    """
    col = df[retention_col].copy()
    # 统一处理为0/1
    col = pd.to_numeric(col, errors='coerce').fillna(0).astype(int)
    label_map = {1: f'{retention_col}✓', 0: f'{retention_col}✗'}
    return col.map(label_map)

def prepare_payment_group(df, payment_col, bins=None):
    """
    将付费列转换为分组标签（付费/非付费，或按金额分层）
    默认分两组：付费（>0）/ 非付费（=0或NaN）
    """
    col = pd.to_numeric(df[payment_col], errors='coerce').fillna(0)
    if bins is None:
        # 默认二分：付费 vs 非付费
        return col.apply(lambda x: '付费用户' if x > 0 else '非付费用户')
    else:
        # 自定义分层
        labels = [f'{bins[i]}-{bins[i+1]}' for i in range(len(bins)-1)]
        return pd.cut(col, bins=bins, labels=labels)
```
