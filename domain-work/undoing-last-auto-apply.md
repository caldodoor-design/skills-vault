---
name: undoing-last-auto-apply
description: 自動適用の直前変更をワンアクションで取り消し
trigger: 自動適用のUndo機能・変更履歴管理の実装・修正時
---

# S19: UndoLastAutoApply

自動適用が怖くない状態を作る。直前の自動適用だけをワンアクションで取り消し（心理安全性）。

## ルール
1. 対象は「直前のauto applyで変更された行/セル」のみ
2. ユーザー手修正履歴は保持（消さない）
3. UIボタン1つで実行可能

## I/O
- Input: lastAutoApplySnapshot{changedRowIds[], beforeValues}, currentRows[]
- Output: restoredRows[], restoredCount

## 注意
- 全部巻き戻すとユーザー修正まで消える → 対象限定が必須
- MVPは直前1回のUndoで十分。before snapshot方式が簡単で堅牢
