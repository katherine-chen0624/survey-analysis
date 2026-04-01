# 交叉分析详细代码模板

## 使用前必做：二次确认
在执行任何交叉分析前，必须先向使用者确认题目：
> "你说的第X题是'[题目文字]'，第Y题是'[题目文字]'，对吗？确认后我来执行交叉分析。"

---

## 1. 单选 × 单选（基础交叉）

```python
def cross_single_single(df, row_col, col_col, row_options=None, col_options=None):
    """
    两道单选题交叉
    参考Gossip Harbor格式：行=一题选项，列=另一题选项，值=行内占比%
    """
    from scipy.stats import chi2_contingency

    ct = pd.crosstab(df[row_col], df[col_col])
    ct_pct = pd.crosstab(df[row_col], df[col_col], normalize='index') * 100
    ct_pct = ct_pct.round(1)

    # 卡方检验
    try:
        chi2, p, dof, _ = chi2_contingency(ct)
        sig = '***' if p < 0.001 else '**' if p < 0.01 else '*' if p < 0.05 else '不显著'
        print(f"\n卡方检验：χ²={chi2:.2f}, p={p:.4f} {sig}")
    except:
        pass

    print(f"\n=== 交叉分析：{row_col} × {col_col} ===")
    print(f"行内占比(%)：")
    print(ct_pct)
    print(f"\n频次：")
    print(ct)
    return ct, ct_pct
```

---

## 2. 单选 × 11点量表均分对比

```python
def cross_single_11pt(df, group_col, scale_col, group_options=None):
    """
    按单选题分组，对比11点量表的均分（-5后）
    常用场景：不同年龄/性别/游戏经历用户的整体评分对比
    """
    from references.data_cleaning import encode_11pt

    numeric = encode_11pt(df[scale_col])
    df_temp = df.copy()
    df_temp['_score'] = numeric - 5  # -5处理

    result = df_temp.groupby(group_col)['_score'].agg(
        N='count',
        均分=lambda x: round(x.mean(), 2)
    ).reset_index()

    print(f"\n=== {scale_col} 按 {group_col} 分组均分对比 ===")
    print(result.to_string(index=False))
    return result
```

---

## 3. 单选 × 多选（选择率对比）

```python
def cross_single_multi(df, group_col, multi_option_cols, base_col=None):
    """
    按单选题分组，对比多选题各选项的选择率
    常用场景：不同用户群的亮点/痛点选择率对比
    """
    groups = df[group_col].dropna().unique()
    results = {}

    for grp in groups:
        grp_df = df[df[group_col] == grp]
        n = len(grp_df)
        grp_result = {}
        for col in multi_option_cols:
            col_data = grp_df[col]
            if col_data.dtype == object:
                selected = col_data.apply(
                    lambda x: x == 'Selected' or
                             (pd.notna(x) and str(x).strip() not in ['', 'nan', '0'])
                ).sum()
            else:
                selected = (col_data == 1).sum()
            option_text = col.split(': ')[-1] if ': ' in col else col.split('_')[-1]
            grp_result[option_text] = round(selected / n * 100, 1)
        results[f"{grp}(N={n})"] = grp_result

    result_df = pd.DataFrame(results).fillna(0)
    result_df = result_df.sort_values(by=list(results.keys())[0], ascending=False)

    print(f"\n=== 多选题选择率对比（按 {group_col} 分组）===")
    print(result_df.to_string())
    return result_df
```

---

## 4. 题目 × 留存交叉

```python
def cross_with_retention(df, question_col, retention_col, q_type='single'):
    """
    题目 × 留存交叉
    retention_col: '次日登录'/'三日登录'/'五日登录'/'七日登录'/'15日登录'
    q_type: 题目类型（single/multi/scale_11pt/scale_5pt）

    留存值：1=有留存，0或NaN=无留存
    """
    from references.question_types import prepare_retention_group

    df_temp = df.copy()
    df_temp['_retention'] = prepare_retention_group(df_temp, retention_col)

    ret_rate_overall = (pd.to_numeric(df[retention_col], errors='coerce').fillna(0) > 0).mean()
    print(f"\n{retention_col}整体留存率：{ret_rate_overall:.1%}")

    if q_type == 'single':
        # 各选项的留存率
        result = df_temp.groupby(question_col).apply(
            lambda x: pd.to_numeric(x[retention_col], errors='coerce').fillna(0).mean()
        ).reset_index()
        result.columns = [question_col, f'{retention_col}留存率']
        result[f'{retention_col}留存率'] = result[f'{retention_col}留存率'].apply(
            lambda x: f"{x:.1%}"
        )
        result = result.sort_values(f'{retention_col}留存率', ascending=False)
        print(f"\n=== {question_col} × {retention_col} ===")
        print(result.to_string(index=False))
        return result

    elif q_type == 'multi':
        # 选择了该选项 vs 未选择的留存率差异
        print(f"\n=== 多选题各选项 × {retention_col} ===")
        print("（选了该选项的用户留存率 vs 整体留存率）")
        # 需传入具体的option_cols列表，此处为示意
        pass
```

---

## 5. 题目 × 付费交叉

```python
def cross_with_payment(df, question_col, payment_col='累计充值', q_type='single'):
    """
    题目 × 付费交叉
    payment_col: 'LTV3'/'LTV5'/'LTV7'/'LTV30'/'累计充值'
    默认分为：付费用户（>0）/ 非付费用户（=0）
    """
    from references.question_types import prepare_payment_group

    df_temp = df.copy()
    df_temp['_payment_group'] = prepare_payment_group(df_temp, payment_col)

    pay_rate = (pd.to_numeric(df[payment_col], errors='coerce').fillna(0) > 0).mean()
    print(f"\n付费用户占比（{payment_col}>0）：{pay_rate:.1%}")

    if q_type == 'single':
        ct_pct = pd.crosstab(
            df_temp[question_col],
            df_temp['_payment_group'],
            normalize='index'
        ) * 100
        ct_pct = ct_pct.round(1)
        print(f"\n=== {question_col} × 付费分组（{payment_col}）===")
        print(ct_pct.to_string())
        return ct_pct
```

---

## 6. 历史对比列输出

```python
def format_comparison_table(current_data, historical_data, label_current, label_hist):
    """
    生成跨测次对比表
    current_data: 当前测试的统计结果DataFrame
    historical_data: 历史测试的统计结果DataFrame
    """
    merged = current_data.merge(
        historical_data,
        on='选项',
        how='outer',
        suffixes=(f'_{label_current}', f'_{label_hist}')
    )

    current_col = f'占比(%)_{label_current}'
    hist_col = f'占比(%)_{label_hist}'

    if current_col in merged.columns and hist_col in merged.columns:
        merged['差值'] = (merged[current_col] - merged[hist_col]).round(1)
        merged['差值'] = merged['差值'].apply(
            lambda x: f"▲{x:.1f}%" if x > 0 else f"▼{abs(x):.1f}%" if x < 0 else "-"
        )

    return merged
```
