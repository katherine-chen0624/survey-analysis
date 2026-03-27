# Excel报告输出模板

## 整体输出逻辑

```python
import pandas as pd
from openpyxl import load_workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side
from openpyxl.utils import get_column_letter

def export_survey_report(analysis_results, project_name, test_round, n_total,
                          output_path=None, historical_data=None):
    """
    生成标准问卷分析Excel报告
    
    analysis_results: dict，key=分组名称，value=该分组的统计结果列表
    project_name: 项目名称（如'W3'、'SP'）
    test_round: 测试轮次（如'海外首测'、'二测'）
    n_total: 有效样本量
    historical_data: 历史数据（用于对比列），可为None
    """
    if output_path is None:
        output_path = f'/mnt/user-data/outputs/{project_name}_{test_round}_问卷分析报告.xlsx'

    with pd.ExcelWriter(output_path, engine='openpyxl') as writer:

        # Sheet 1：宏观指标总览
        _write_macro_sheet(writer, analysis_results, project_name,
                           test_round, n_total, historical_data)

        # Sheet 2+：各题组详细数据
        for group_name, group_data in analysis_results.items():
            if group_name == '宏观指标':
                continue
            sheet_name = group_name[:31]  # Excel限制31字符
            _write_detail_sheet(writer, sheet_name, group_data,
                                n_total, historical_data)

    print(f"✓ 报告已生成：{output_path}")
    return output_path


def _write_macro_sheet(writer, results, project, round_name, n, hist=None):
    """宏观指标Sheet"""
    rows = []

    # 标题行
    header = [f'{project} {round_name}（N={n}）']
    if hist:
        header.append(f"历史对比（N={hist['n']}）")
        header.append('差值')
    rows.append(header)

    # 宏观指标区
    rows.append(['宏观指标'])
    for metric_name, metric_value in results.get('宏观指标', {}).items():
        row = [metric_name, metric_value]
        if hist and metric_name in hist.get('core_metrics', {}):
            hist_val = hist['core_metrics'][metric_name]
            diff = round(metric_value - hist_val, 2) if isinstance(metric_value, (int, float)) else '-'
            row.extend([hist_val, diff])
        rows.append(row)

    df_macro = pd.DataFrame(rows)
    df_macro.to_excel(writer, sheet_name='宏观指标总览', index=False, header=False)


def _write_detail_sheet(writer, sheet_name, group_data, n, hist=None):
    """各题详细数据Sheet"""
    all_rows = []

    for item in group_data:
        q_text = item.get('question', '')
        q_type = item.get('type', '')
        result_df = item.get('data')

        # 题目标题行
        all_rows.append([f"Q: {q_text}（有效N={item.get('n', n)}）"])

        if q_type in ('single_choice', 'multi_choice'):
            # 标准频次表
            all_rows.append(['选项', '占比(%)', '人数'])
            for _, row in result_df.iterrows():
                all_rows.append([row['选项'], row.get('占比(%)', row.get('选择率(%)')), row['人数']])

        elif q_type == '5pt_scale':
            # 均分 + 分值分布
            all_rows.append([f"均分：{item.get('score')}"])
            all_rows.append(['分值', '占比(%)', '人数'])
            for _, row in result_df.iterrows():
                all_rows.append([row['选项'], row['占比(%)'], row['人数']])

        elif q_type == 'matrix_5pt':
            # 矩阵：每行一个维度
            all_rows.append(['维度', '均分(1~5)', '1占比', '2占比', '3占比', '4占比', '5占比'])
            for matrix_item in item.get('rows', []):
                row_data = [matrix_item['维度'], matrix_item['均分(1~5)']]
                for i in range(1, 6):
                    row_data.append(matrix_item['distribution'].get(i, {}).get('pct', '-'))
                all_rows.append(row_data)

        elif q_type == 'open_text':
            all_rows.append([f"有效回答：{item.get('n')} 条"])
            all_rows.append(['主要内容归纳：'])
            for theme in item.get('themes', []):
                all_rows.append([f"  • {theme}"])

        # 空行分隔
        all_rows.append([])

    df_detail = pd.DataFrame(all_rows)
    df_detail.to_excel(writer, sheet_name=sheet_name, index=False, header=False)


def apply_report_style(output_path):
    """
    对生成的Excel应用基础样式
    标题行：加粗 + 浅灰背景
    数据行：正常
    宏观指标：加粗
    """
    wb = load_workbook(output_path)
    
    header_fill = PatternFill(start_color='D9D9D9', end_color='D9D9D9', fill_type='solid')
    section_fill = PatternFill(start_color='F2F2F2', end_color='F2F2F2', fill_type='solid')
    bold_font = Font(bold=True)
    
    for ws in wb.worksheets:
        for row in ws.iter_rows():
            for cell in row:
                cell.alignment = Alignment(wrap_text=True, vertical='center')
                
                # 检测题目行（以Q:开头）
                if cell.value and str(cell.value).startswith('Q:'):
                    cell.font = bold_font
                    cell.fill = section_fill
                    
                # 检测表头行（选项/占比/人数）
                if cell.value in ('选项', '占比(%)', '人数', '选择率(%)', '分值', '维度', '均分(1~5)'):
                    cell.font = bold_font
                    cell.fill = header_fill
        
        # 自动调整列宽
        for col in ws.columns:
            max_length = max(
                (len(str(cell.value)) for cell in col if cell.value), default=10
            )
            ws.column_dimensions[get_column_letter(col[0].column)].width = min(max_length + 4, 50)
    
    wb.save(output_path)
    return output_path
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
