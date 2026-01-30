---
name: organizing-invoices
description: 請求書・領収書の自動整理。ファイル読み取り→情報抽出→統一命名→フォルダ分類→CSV出力で確定申告準備。
trigger: 請求書・領収書の整理、経費管理、確定申告準備、インボイス整理が必要な時
---

# 請求書整理スキル

散在する請求書・領収書を、税務申告に対応した整理済みファイルシステムに変換する。

> **元リポジトリ**: https://github.com/ComposioHQ/awesome-claude-skills

## 処理内容

1. **情報抽出**: PDF・画像から取引先名、請求番号、日付、金額、品目を抽出
2. **統一命名**: `YYYY-MM-DD 取引先 - Invoice - 品目.pdf` 形式にリネーム
3. **フォルダ分類**: 取引先別、経費カテゴリ別、期間別、税区分別に分類
4. **CSV出力**: 全請求書情報を一覧化し会計ソフト連携・経理共有に対応

## 手順

### 1. フォルダスキャン

```bash
find . -type f \( -name "*.pdf" -o -name "*.jpg" -o -name "*.png" \) -print
```

ファイル数、種別、日付範囲を報告。

### 2. 情報抽出

**PDFから**:
- テキスト抽出で請求書内容を読み取り
- パターン: "Invoice Date:", "Invoice #:", "Amount Due:", "Total:"

**画像から**:
- 可視テキストを読み取り
- 取引先名（通常上部）、日付、合計金額を特定

**判別不能な場合**:
- ファイル名の手がかりを使用
- ファイルの作成/更新日をフォールバック
- 手動レビュー用にフラグ

### 3. 整理戦略の選択

ユーザーに確認（未指定の場合）:
1. **取引先別**: Adobe/, Amazon/, Stripe/
2. **カテゴリ別**: Software/, Office/, Travel/
3. **日付別**: 2024/Q1/, 2024/Q2/
4. **税区分別**: 控除対象/, 個人/
5. デフォルト: 年/カテゴリ/取引先

### 4. 統一ファイル名

```
YYYY-MM-DD 取引先 - Invoice - 品目.ext
```

例:
- `2024-03-15 Adobe - Invoice - Creative Cloud.pdf`
- `2024-01-10 Amazon - Receipt - Office Supplies.pdf`

### 5. 実行

計画を提示し承認後に実行:

```bash
mkdir -p "Invoices/2024/Software/Adobe"
cp "original.pdf" "Invoices/2024/Software/Adobe/2024-03-15 Adobe - Invoice - Creative Cloud.pdf"
```

### 6. CSV出力

```csv
Date,Vendor,Invoice Number,Description,Amount,Category,File Path
2024-03-15,Adobe,INV-12345,Creative Cloud,52.99,Software,Invoices/2024/...
```

用途: 会計ソフトへのインポート、経理への共有、経費レポート、確定申告

### 7. 完了サマリ

- 処理件数、日付範囲、合計金額、取引先数
- 新しいフォルダ構造
- レビューが必要なファイル一覧
- CSV出力ファイルの場所

## 整理パターン例

### 取引先別（シンプル）
```
Invoices/
├── Adobe/
├── Amazon/
└── Google/
```

### 年+カテゴリ（税務対応）
```
Invoices/
├── 2023/
│   ├── Software/
│   ├── Hardware/
│   └── Travel/
└── 2024/
```

### 税区分別（経理向け）
```
Invoices/
├── 控除対象/
│   ├── Software/
│   └── Services/
├── 一部控除/
│   └── 交際費/
└── 個人/
```

## 特殊ケース対応

- **情報不足**: 手動レビューフォルダに分類、ファイル更新日をフォールバック
- **重複請求書**: ハッシュ比較、高品質版を保持
- **複数ページ**: 必要に応じてPDF結合

## ヒント

- 月次で整理する（年次でなく）
- 元ファイルをバックアップしてから整理
- 領収書は7年間保管（監査期間）
- 金額をCSVに含める（予算管理に有用）
