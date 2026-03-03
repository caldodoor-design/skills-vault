---
name: operating-xlsx
description: Excel(.xlsx)の作成・編集・分析。数式・フォーマット・財務モデル対応。openpyxl/pandas使用。
---
# XLSX操作ガイド

## 鉄則: 数式を使い、値をハードコードしない
```python
sheet['B10'] = '=SUM(B2:B9)'      # OK
sheet['C5'] = '=(C4-C2)/C2'       # OK
# sheet['B10'] = total             # NG
```

## データ分析（pandas）
```python
df = pd.read_excel('file.xlsx')
all_sheets = pd.read_excel('file.xlsx', sheet_name=None)
df.to_excel('output.xlsx', index=False)
```

## 作成・編集（openpyxl）
```python
from openpyxl import Workbook, load_workbook
from openpyxl.styles import Font, PatternFill
wb = Workbook()  # or load_workbook('existing.xlsx')
sheet = wb.active
sheet['A1'] = 'データ'
sheet['A1'].font = Font(bold=True)
wb.save('output.xlsx')
```

## 財務モデル規約
- **色**: 青文字=入力値、黒=数式、緑=シート間リンク、赤=外部リンク
- **通貨**: `$#,##0`、負数は括弧`(123)`
- **前提値**: 別セルに配置し数式からセル参照

## 注意
- セルインデックスは1始まり
- `data_only=True`で保存すると数式が値に置換され消失
- 大規模: `read_only=True` / `write_only=True`
