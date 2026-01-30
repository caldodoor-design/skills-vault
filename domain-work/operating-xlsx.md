---
name: operating-xlsx
description: Excelスプレッドシート(.xlsx)の作成・編集・分析。数式、フォーマット、データ分析、財務モデル対応。
trigger: Excelファイルの作成、編集、データ分析、数式操作が必要な時
---

# XLSX操作ガイド

> **元リポジトリ**: https://github.com/anthropics/skills
> **scripts依存**: recalc.py等は元リポジトリを参照

## 重要: 数式を使い、値をハードコードしない

```python
# NG - Pythonで計算してハードコード
total = df['Sales'].sum()
sheet['B10'] = total

# OK - Excel数式を使用
sheet['B10'] = '=SUM(B2:B9)'
sheet['C5'] = '=(C4-C2)/C2'
sheet['D20'] = '=AVERAGE(D2:D19)'
```

## データ読み取り・分析（pandas）

```python
import pandas as pd

df = pd.read_excel('file.xlsx')
all_sheets = pd.read_excel('file.xlsx', sheet_name=None)
df.describe()
df.to_excel('output.xlsx', index=False)
```

## 新規Excelファイル作成（openpyxl）

```python
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

wb = Workbook()
sheet = wb.active
sheet['A1'] = 'データ'
sheet['B2'] = '=SUM(A1:A10)'
sheet['A1'].font = Font(bold=True, color='FF0000')
sheet['A1'].fill = PatternFill('solid', start_color='FFFF00')
sheet.column_dimensions['A'].width = 20
wb.save('output.xlsx')
```

## 既存ファイル編集

```python
from openpyxl import load_workbook

wb = load_workbook('existing.xlsx')
sheet = wb.active
sheet['A1'] = '新しい値'
sheet.insert_rows(2)
new_sheet = wb.create_sheet('NewSheet')
wb.save('modified.xlsx')
```

## 数式の再計算

```bash
python recalc.py output.xlsx
```

エラーが返された場合（`#REF!`, `#DIV/0!`等）は修正して再実行。

## 財務モデルの規約

### 色コーディング
- **青文字**: ハードコード入力値（ユーザーが変更する数値）
- **黒文字**: 全ての数式・計算
- **緑文字**: 同ワークブック内の他シートからのリンク
- **赤文字**: 外部ファイルへのリンク
- **黄背景**: 注意が必要な前提値

### 数値フォーマット
- **年**: テキスト形式（"2024"、"2,024"にしない）
- **通貨**: `$#,##0`、ヘッダーに単位記載（"Revenue ($mm)"）
- **ゼロ**: "-"表示（`$#,##0;($#,##0);-`）
- **パーセント**: `0.0%`（小数1位）
- **負数**: 括弧表示 `(123)` マイナス記号 `-123` ではなく

### 数式ルール
- 前提値は別セルに配置し、数式からセル参照
- `=B5*(1+$B$6)` を使い `=B5*1.05` としない
- ハードコード値にはソースコメントを付記

## ライブラリ選択

| 用途 | ツール |
|------|--------|
| データ分析・一括操作 | pandas |
| 数式・フォーマット・Excel固有機能 | openpyxl |

## 注意点

- セルインデックスは1始まり（row=1, column=1 = A1）
- `data_only=True` で開いて保存すると数式が値に置換され永久消失
- 大規模ファイル: `read_only=True` / `write_only=True` を使用
