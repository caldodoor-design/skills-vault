---
name: applying-batch-on-confirm
description: Confirm直後に同一摘要の他行を即時一括修正する
trigger: Confirm後の一括適用ロジック実装・修正時
---

# S27: InteractiveBatchApplyOnConfirm

Confirm直後に学習ルールを使って「同一摘要の他行」を即時一括修正。

## ルール
1. S15 InteractiveBatchApply を再利用
2. 変更対象は CONFIRMED以外（未確定）のみ
3. S18 Safety Gateを必ず通す
4. 変更行には "AI_APPLIED"（青）を付ける
5. maxApply上限（例：200）を必ず守る
6. 件数をUIメッセージで提示

## I/O
- Input: confirmedRow, allVisibleRows, rules, bankId
- Output: appliedCount, blockedCount, warnedCount, changedRowIds[], summaryMessage

## 注意
- "次回リロードで効く"では遅い。作業中に効くのが価値
- Undo（S19）で戻せる差分を積む
