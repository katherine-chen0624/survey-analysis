# Excel报告输出模板

## 整体输出逻辑

**所有内容输出到同一个 Sheet，从上到下依次排列：**

```python
import pandas as pd
from openpyxl import load_workbook
from openpyxl.styles import Font, PatternFill, Alignment
from openpyxl.utils import get_column_letter

def export_survey_report(analysis_results, project_name, test_round, n_raw,
                          n_invalid_empty, n_invalid_short, output_path=None,
                          historical_data=None):
    """
    生成标准问卷分析Excel报告（单Sheet版本）

    n_raw: 剔除全空行后的真实问卷份数
    n_invalid_empty: 全空行数量（导出时产生的空白行）
    n_invalid_short: 答题时长<60秒的剔除数量
    """
    if output_path is None:
        output_path = f'/mnt/user-data/outputs/{project_name}_{test_round}_问卷分析报告.xlsx'

    all_rows = []

    # ① 样本说明
    all_rows.append([f'{project_name} {test_round} — 问卷分析报告'])
    all_rows.append([])
    all_rows.append(['【样本说明】'])
    all_rows.append(['原始数据（去除空白行后）', f'{n_raw} 份'])
    all_rows.append(['剔除无效样本（<60秒）', f'{n_invalid_short} 份'])
    all_rows.append(['有效样本 N', n_raw - n_invalid_short])
    all_rows.append([])

    # ② 关键洞察摘要
    all_rows.append(['【关键洞察摘要】'])
    all_rows.append(['（由AI分析自动生成）'])
    all_rows.append([])

    # ③ 宏观指标
    all_rows.append(['【宏观指标】'])
    all_rows.append(['指标', '数值', '说明'])
    for metric_name, metric_value, metric_note in analysis_results.get('宏观指标', []):
        row = [metric_name, metric_value, metric_note]
        if historical_data and metric_name in historical_data:
            hist_val = historical_data[metric_name]
            diff = round(metric_value - hist_val, 2) if isinstance(metric_value, (int, float)) else '-'
            row.extend([hist_val, f'▲{diff}' if diff > 0 else f'▼{abs(diff)}'])
        all_rows.append(row)
    all_rows.append([])

    # ④ 各题详细数据（依次往下，不分Sheet）
    all_rows.append(['【各题详细数据】'])
    all_rows.append([])

    for item in analysis_results.get('各题数据', []):
        q_text = item.get('question', '')
        q_type = item.get('type', '')
        n = item.get('n', 0)
        result_df = item.get('data')

        # 题目标题行
        all_rows.append([f"▌ {q_text}（有效N={n}）"])

        if q_type in ('single_choice', 'multi_choice'):
            col_name = '占比' if q_type == 'single_choice' else '选择率'
            all_rows.append(['选项', col_name, '人数'])
            for _, row in result_df.iterrows():
                # ⚠️ 格式强制：选择率/占比必须是带%的字符串，人数必须是整数
                # 列顺序严格对应标题：选项文字 | 选择率(%) | 人数(整数)
                raw_pct = row.get(col_name, '')
                if isinstance(raw_pct, (int, float)):
                    pct_str = f"{raw_pct:.1f}%"  # 裸数字强制转为百分比字符串
                else:
                    pct_str = str(raw_pct) if '%' in str(raw_pct) else f"{raw_pct}%"
                count = int(row['人数']) if pd.notna(row['人数']) else 0
                all_rows.append([row['选项'], pct_str, count])
            if q_type == 'multi_choice':
                all_rows.append([f'（多选题，基数N={n}，各选择率之和可能超过100%）'])

            # ⚠️ Other 填答归纳：必须在所有选项数据展示完毕后追加，不得插在选项中间
            other_texts = item.get('other_texts', [])
            if other_texts:
                all_rows.append([f'↓ Other填答归纳（N={len(other_texts)}）'])
                if len(other_texts) >= 5:
                    # ≥5条做主题归纳
                    for theme in item.get('other_themes', other_texts):
                        all_rows.append([f'  • {theme}'])
                else:
                    # <5条展示原文
                    for t in other_texts:
                        all_rows.append([f'  • "{t}"'])

        elif q_type == '5pt_scale':
            all_rows.append([f'均分（1~5）：{item.get("score")}'])
            all_rows.append(['分值', '选项文字', '占比(%)', '人数'])
            for _, row in result_df.iterrows():
                all_rows.append([row.get('分值', ''), row['选项'], row['占比(%)'], row['人数']])

        elif q_type == 'matrix_5pt':
            all_rows.append(['维度', '均分(1~5)', '1%', '2%', '3%', '4%', '5%', '有效N'])
            for matrix_row in item.get('rows', []):
                all_rows.append([
                    matrix_row['维度'], matrix_row['均分'],
                    matrix_row.get('1%', ''), matrix_row.get('2%', ''),
                    matrix_row.get('3%', ''), matrix_row.get('4%', ''),
                    matrix_row.get('5%', ''), matrix_row.get('n', '')
                ])

        elif q_type == 'NPS':
            all_rows.append([f'NPS：{item.get("nps")}%'])
            all_rows.append([f'推荐者(8-10)：{item.get("promoters")}人（{item.get("promoters_pct")}%）'])
            all_rows.append([f'中立(5-7)：{item.get("neutral")}人（{item.get("neutral_pct")}%）'])
            all_rows.append([f'贬损者(0-4)：{item.get("detractors")}人（{item.get("detractors_pct")}%）'])

        elif q_type == 'open_text':
            all_rows.append([f'有效回答：{n} 条'])
            for theme in item.get('themes', []):
                all_rows.append([f'  • {theme}'])
            if item.get('keywords'):
                all_rows.append([f'高频词汇：{item.get("keywords")}'])

        # 题目之间空行分隔
        all_rows.append([])
        all_rows.append([])

    # 写入Excel（单Sheet）
    df_output = pd.DataFrame(all_rows)
    with pd.ExcelWriter(output_path, engine='openpyxl') as writer:
        df_output.to_excel(writer, sheet_name='问卷分析报告', index=False, header=False)

    # 应用样式
    apply_single_sheet_style(output_path)
    print(f"✓ 报告已生成：{output_path}")
    return output_path


def apply_single_sheet_style(output_path):
    """单Sheet报告样式"""
    wb = load_workbook(output_path)
    ws = wb.active

    title_font = Font(bold=True, size=13)
    section_font = Font(bold=True, size=11)
    question_font = Font(bold=True, size=10)
    header_fill = PatternFill(start_color='D9D9D9', end_color='D9D9D9', fill_type='solid')
    section_fill = PatternFill(start_color='F2F2F2', end_color='F2F2F2', fill_type='solid')

    for row in ws.iter_rows():
        for cell in row:
            cell.alignment = Alignment(wrap_text=True, vertical='center')
            if cell.value:
                val = str(cell.value)
                # 报告标题
                if '问卷分析报告' in val and cell.column == 1 and cell.row == 1:
                    cell.font = title_font
                # 大区标题（【样本说明】等）
                elif val.startswith('【') and val.endswith('】'):
                    cell.font = section_font
                    cell.fill = section_fill
                # 题目标题（▌开头）
                elif val.startswith('▌'):
                    cell.font = question_font
                    cell.fill = header_fill

    # 自动调整列宽
    for col in ws.columns:
        max_length = max(
            (len(str(cell.value)) for cell in col if cell.value), default=10
        )
        ws.column_dimensions[get_column_letter(col[0].column)].width = min(max_length + 4, 60)

    wb.save(output_path)
```

---

## 报告顶部关键洞察摘要生成提示

生成洞察摘要时，遵循以下原则：
1. **3-5句话**，不超过200字
2. **数据驱动**：每句话必须有具体数字支撑
3. **层次清晰**：宏观指标 → 核心亮点/痛点 → 关键发现 → 对比变化（如有）
4. **区分结论和假设**：确定的用"X%的用户..."，推测的用"初步来看..."

示例格式：
```
1. 整体评分X.X分（-5~5），NPS XX%，续玩意愿X.X分，[高于/低于/持平]上测。
2. 最主要吸引点为[选项]（XX%），其次是[选项]（XX%）；最主要痛点为[选项]（XX%）。
3. [某一关键发现，如：高留存用户中，选择了[亮点]的占比（XX%）显著高于整体（XX%）]
4. [与历史对比的关键变化，如：[指标]较上测提升/下降X个百分点]（有历史数据时才写）
5. [用户画像关键特征，如：超X成用户有Y经历，Z群体评价普遍高于整体]
```
