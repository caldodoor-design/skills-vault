---
name: storing-bank-column-profile
description: 銀行別columnMapの保存・取得（MANUAL優先）
trigger: 銀行別columnMapの保存・取得・MANUAL/AUTO優先度ロジック実装時
---

# S24: BankColumnProfileStore

銀行ごとの列設定（columnMap）をbankId単位で保存し、人間の補正値を再利用。

## ルール
1. bankIdスコープで混線させない
2. MANUAL > AUTO（人間が正）
3. 古いAUTOは上書き可、MANUALは勝手に消さない
4. columnMapの整合（0〜1、x0<x1）は保存前に検証

## API
- `getProfile(bankId)` → columnMap | null
- `upsertProfile(bankId, columnMap, meta{source})` → ok
- `deleteProfile(bankId)` → ok

## 注意
- 保存先: IndexedDB（ローカル）。クラウド保存はセキュリティ上しない
