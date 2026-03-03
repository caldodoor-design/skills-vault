---
name: operating-docx
description: Word(.docx)の作成・編集・分析。テキスト抽出、変更履歴(Redlining)、OOXML編集。
---
# DOCX操作ガイド

## 判断ツリー
- **読み取り**: `pandoc --track-changes=all file.docx -o output.md`
- **新規作成**: docx-js (Document, Paragraph, TextRun → Packer.toBuffer())
- **既存編集(自分の)**: OOXML直接編集
- **既存編集(他者の)**: Redliningワークフロー（変更履歴付き）

## Raw XML
```bash
python ooxml/scripts/unpack.py <file> <dir>  # 展開
python ooxml/scripts/pack.py <dir> <file>    # 再パック
```
主要: `word/document.xml`, `word/comments.xml`, `word/media/`

## Redlining（変更履歴付き編集）
**原則: 変更箇所のみマーク**
```xml
<!-- OK: 変更箇所のみ -->
<w:r>The term is </w:r><w:del>30</w:del><w:ins>60</w:ins><w:r> days.</w:r>
<!-- NG: 文全体を置換 -->
```
手順: pandoc確認→変更特定→展開→バッチ編集→再パック→pandoc検証

## 画像変換
```bash
soffice --headless --convert-to pdf doc.docx && pdftoppm -jpeg -r 150 doc.pdf page
```
