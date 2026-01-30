---
name: operating-pdf
description: PDFの作成・結合・分割・テキスト抽出・テーブル抽出・暗号化。pypdf、pdfplumber、reportlabを使用。
trigger: PDFの作成、編集、結合、分割、テキスト抽出、テーブル抽出が必要な時
---

# PDF操作ガイド

> **元リポジトリ**: https://github.com/anthropics/skills
> **scripts依存**: 実行スクリプトは元リポジトリを参照

## Pythonライブラリ

### pypdf - 基本操作

#### 読み取り・テキスト抽出
```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("document.pdf")
print(f"ページ数: {len(reader.pages)}")

text = ""
for page in reader.pages:
    text += page.extract_text()
```

#### PDF結合
```python
writer = PdfWriter()
for pdf_file in ["doc1.pdf", "doc2.pdf", "doc3.pdf"]:
    reader = PdfReader(pdf_file)
    for page in reader.pages:
        writer.add_page(page)

with open("merged.pdf", "wb") as output:
    writer.write(output)
```

#### PDF分割
```python
reader = PdfReader("input.pdf")
for i, page in enumerate(reader.pages):
    writer = PdfWriter()
    writer.add_page(page)
    with open(f"page_{i+1}.pdf", "wb") as output:
        writer.write(output)
```

#### メタデータ抽出
```python
reader = PdfReader("document.pdf")
meta = reader.metadata
print(f"タイトル: {meta.title}, 著者: {meta.author}")
```

#### ページ回転
```python
reader = PdfReader("input.pdf")
writer = PdfWriter()
page = reader.pages[0]
page.rotate(90)
writer.add_page(page)
with open("rotated.pdf", "wb") as output:
    writer.write(output)
```

### pdfplumber - テキスト・テーブル抽出

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page in pdf.pages:
        text = page.extract_text()
        tables = page.extract_tables()
        for table in tables:
            for row in table:
                print(row)
```

#### テーブルをDataFrameに変換
```python
import pandas as pd

with pdfplumber.open("document.pdf") as pdf:
    all_tables = []
    for page in pdf.pages:
        tables = page.extract_tables()
        for table in tables:
            if table:
                df = pd.DataFrame(table[1:], columns=table[0])
                all_tables.append(df)
    if all_tables:
        combined_df = pd.concat(all_tables, ignore_index=True)
        combined_df.to_excel("extracted_tables.xlsx", index=False)
```

### reportlab - PDF作成

```python
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, PageBreak
from reportlab.lib.styles import getSampleStyleSheet

doc = SimpleDocTemplate("report.pdf", pagesize=letter)
styles = getSampleStyleSheet()
story = []
story.append(Paragraph("レポートタイトル", styles['Title']))
story.append(Spacer(1, 12))
story.append(Paragraph("本文テキスト " * 20, styles['Normal']))
doc.build(story)
```

## コマンドラインツール

```bash
# テキスト抽出（poppler-utils）
pdftotext input.pdf output.txt
pdftotext -layout input.pdf output.txt  # レイアウト保持

# PDF結合（qpdf）
qpdf --empty --pages file1.pdf file2.pdf -- merged.pdf

# ページ分割
qpdf input.pdf --pages . 1-5 -- pages1-5.pdf

# 画像抽出
pdfimages -j input.pdf output_prefix
```

## その他の操作

#### 透かし追加
```python
watermark = PdfReader("watermark.pdf").pages[0]
reader = PdfReader("document.pdf")
writer = PdfWriter()
for page in reader.pages:
    page.merge_page(watermark)
    writer.add_page(page)
with open("watermarked.pdf", "wb") as output:
    writer.write(output)
```

#### パスワード保護
```python
writer.encrypt("userpassword", "ownerpassword")
with open("encrypted.pdf", "wb") as output:
    writer.write(output)
```

## クイックリファレンス

| タスク | ツール | コマンド/コード |
|--------|--------|----------------|
| 結合 | pypdf | `writer.add_page(page)` |
| 分割 | pypdf | 1ページずつファイル出力 |
| テキスト抽出 | pdfplumber | `page.extract_text()` |
| テーブル抽出 | pdfplumber | `page.extract_tables()` |
| PDF作成 | reportlab | Canvas or Platypus |
| OCR（スキャン） | pytesseract | pdf2imageで画像化後OCR |
