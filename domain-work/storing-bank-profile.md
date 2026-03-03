---
name: storing-bank-profile
description: 銀行IDごとの列テンプレート（Column Profile）永続化
trigger: 銀行プロファイルの保存・取得・フォールバック処理実装時
---

# S7: BankProfileStore

銀行IDごとの列テンプレート（Column Ratios）を永続化し、OCR揺らぎに依存しない安定した列推定を提供。

## ルール
1. **Default Fallback**: 未登録bankIdは必ずDefault Profileを返す（クラッシュ防止）
2. **Ratio Normalization**: 保存・取得時にColumn Ratios合計を1.0に自動正規化
3. **Isolation**: bankIdごとに完全分離
4. **Validation**: NaN/負数/ゼロはDefault Profileにフォールバック

## API
- `getProfile(bankId)` → BankProfile
- `upsertProfile(bankId, profile)` → void

## 注意
- profileが無い場合にnull返すとUIクラッシュ → 必ずDefaultで返す
- 合計>1.0の比率をそのまま適用すると列はみ出し
