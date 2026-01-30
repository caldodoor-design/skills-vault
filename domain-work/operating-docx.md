---
name: operating-docx
description: Word文書(.docx)の作成・編集・分析。テキスト抽出、変更履歴、コメント、フォーマット保持に対応。
trigger: Word文書の作成、編集、テキスト抽出、変更履歴操作が必要な時
---

# DOCX操作ガイド

> **元リポジトリ**: https://github.com/anthropics/skills
> **scripts依存**: ooxml/scripts/* は元リポジトリを参照

## ワークフロー判断ツリー

### 読み取り・分析
テキスト抽出またはRaw XMLアクセスを使用

### 新規文書作成
docx-jsワークフローを使用

### 既存文書編集
- **自分の文書 + 簡単な変更** → 基本OOXML編集
- **他者の文書** → Redliningワークフロー（推奨）
- **法務・学術・ビジネス文書** → Redliningワークフロー（必須）

## テキスト抽出

```bash
# pandocでMarkdownに変換（変更履歴付き）
pandoc --track-changes=all path-to-file.docx -o output.md
```

## Raw XMLアクセス

コメント、複雑なフォーマット、埋め込みメディア、メタデータの操作に必要。

```bash
# .docxを展開
python ooxml/scripts/unpack.py <office_file> <output_directory>
```

### 主要ファイル構造
- `word/document.xml` - メイン文書
- `word/comments.xml` - コメント
- `word/media/` - 埋め込み画像・メディア
- 変更履歴: `<w:ins>`（挿入）、`<w:del>`（削除）タグ

## 新規文書作成（docx-js）

1. docx-jsの参照ドキュメントを読む
2. JavaScript/TypeScriptでDocument, Paragraph, TextRunを使用
3. `Packer.toBuffer()` で.docxエクスポート

## Redliningワークフロー（変更履歴付き編集）

**原則: 最小限の正確な編集** - 変更箇所のみマークし、未変更テキストはそのまま残す。

```python
# NG - 文全体を置換
'<w:del>..全文..</w:del><w:ins>..全文..</w:ins>'

# OK - 変更箇所のみマーク
'<w:r>The term is </w:r><w:del>30</w:del><w:ins>60</w:ins><w:r> days.</w:r>'
```

### 手順
1. pandocでMarkdown変換し現在の内容を確認
2. 変更を特定しバッチ（3-10件）にグループ化
3. ooxml.mdを読み、文書を展開
4. バッチごとにスクリプトで変更を実装
5. 再パック: `python ooxml/scripts/pack.py <dir> <file.docx>`
6. 最終検証: pandocで変換し全変更が正しく適用されたか確認

## 画像への変換

```bash
# DOCX → PDF → JPEG
soffice --headless --convert-to pdf document.docx
pdftoppm -jpeg -r 150 document.pdf page
```

## 依存関係

- **pandoc**: テキスト抽出用
- **docx**: `npm install -g docx`（新規文書作成）
- **LibreOffice**: PDF変換
- **Poppler**: `poppler-utils`（PDF→画像変換）
