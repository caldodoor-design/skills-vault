---
name: operating-pdf
description: PDFの作成・結合・分割・テキスト/テーブル抽出・暗号化。pypdf/pdfplumber/reportlab使用。
---
# PDF操作ガイド

## pypdf - 基本操作
```python
from pypdf import PdfReader, PdfWriter
reader = PdfReader("doc.pdf")
text = "".join(p.extract_text() for p in reader.pages)  # テキスト抽出
```
| 操作 | コード |
|------|--------|
| 結合 | `writer.add_page(page)` で各PDFのページ追加 |
| 分割 | 1ページずつ別PdfWriterでファイル出力 |
| 回転 | `page.rotate(90)` |
| 透かし | `page.merge_page(watermark_page)` |
| 暗号化 | `writer.encrypt("user_pw", "owner_pw")` |

## pdfplumber - テーブル抽出
```python
import pdfplumber
with pdfplumber.open("doc.pdf") as pdf:
    for page in pdf.pages:
        tables = page.extract_tables()  # list[list[list[str]]]
        # DataFrameへ: pd.DataFrame(table[1:], columns=table[0])
```

## reportlab - PDF作成
```python
from reportlab.platypus import SimpleDocTemplate, Paragraph
from reportlab.lib.styles import getSampleStyleSheet
doc = SimpleDocTemplate("report.pdf")
doc.build([Paragraph("テキスト", getSampleStyleSheet()['Normal'])])
```

## CLI
```bash
pdftotext input.pdf output.txt          # テキスト抽出
pdftotext -layout input.pdf output.txt  # レイアウト保持
qpdf --empty --pages f1.pdf f2.pdf -- merged.pdf  # 結合
pdfimages -j input.pdf prefix           # 画像抽出
```
