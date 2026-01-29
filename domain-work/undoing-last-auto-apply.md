---
name: undoing-last-auto-apply
description: 自動適用（S17）の直前変更をワンアクションで取り消し、心理安全性を確保するUndoスキル。
trigger: 自動適用のUndo機能を実装・修正するとき、または変更履歴の管理方法を検討するとき
---

# S19: UndoLastAutoApply

> Updated by **Agent Delta** based on `reports/sprint-006/verification_report.md`
> Validated: Sprint 6 Verification

## 1. Purpose
自動適用（S17）が怖くない状態を作る。
直前の自動適用だけをワンアクションで取り消せるようにする（心理安全性）。

## 2. Scope
- 対象：直前の "自動適用で変更された部分" のみを元に戻す
- 対象外：ユーザー手修正の巻き戻し（戻さない）
- 対象外：複数世代のUndo（MVPは直前1回でOK）

## 3. Inputs / Outputs
### Inputs
- lastAutoApplySnapshot:
  - changedRowIds: string[]
  - beforeValues: Map(rowId -> previous row state) もしくは patch diff
- currentRows: TransactionRow[]

### Outputs
- restoredRows: TransactionRow[]
- restoredCount: number

## 4. Rules
1) Undo対象は "直前の auto apply で変更された行/セル" のみ
2) Undo後もユーザー手修正履歴は保持（消さない）
3) UndoはUIボタン1つで実行できる状態にする

## 5. DoD
- [ ] 自動適用後にUndoできる
- [ ] Undoしてもユーザー手修正は消えない
- [ ] restoredCountがUIに表示できる

## 6. Test Cases
- Case1: 自動適用で12行変更 → Undoで12行戻る
- Case2: 自動適用後にユーザーが1行手修正 → Undoしても手修正は残る
- Case3: 変更なしの状態 → Undoは何もしない（安全）

## 7. Anti-Patterns
- 全部巻き戻す（ユーザー修正まで消える）
- Undoできない（怖くて自動化が使われない）
- 何が戻ったか分からない（現場不安）

## 8. Notes
- 最初は "before snapshot方式" が実装が簡単で堅牢
- diff方式は将来最適化でOK
