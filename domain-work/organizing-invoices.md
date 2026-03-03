---
name: organizing-invoices
description: 請求書・領収書の自動整理（情報抽出→統一命名→フォルダ分類→CSV出力）
trigger: 請求書・領収書の整理、経費管理、確定申告準備時
---
# 請求書整理

## フロー
1. PDF/画像スキャン → 2. 情報抽出（取引先・日付・金額） → 3. 統一命名 → 4. フォルダ分類 → 5. CSV出力

## 命名規則
```
YYYY-MM-DD 取引先 - Invoice - 品目.ext
```

## 分類パターン
| パターン | 構造 |
|---------|------|
| 取引先別 | Adobe/, Amazon/, Google/ |
| 年+カテゴリ | 2024/Software/, 2024/Travel/ |
| 税区分別 | 控除対象/Software/, 個人/ |
| デフォルト | 年/カテゴリ/取引先 |

## CSV出力
```csv
Date,Vendor,Invoice Number,Description,Amount,Category,File Path
```

## 特殊ケース
- **情報不足**: ファイル更新日フォールバック、手動レビューフォルダへ
- **重複**: ハッシュ比較、高品質版を保持
- 領収書は7年間保管（監査期間）
