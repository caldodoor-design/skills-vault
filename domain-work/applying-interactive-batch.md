---
name: applying-interactive-batch
description: 学習（Teach）した瞬間に同一clientNormの行へ一括適用を提案し、承認後に反映することで現場の入力工数を削減する。
trigger: 学習ボタン押下後のページ内一括適用提案を実装・修正するとき
---

# S15: InteractiveBatchApply

> Updated by **Agent Delta** based on `reports/sprint-005/verification_report.md`
> Validated: Sprint 5 Verification

## 1. Purpose
学習（Teach）した瞬間に、同一 clientNorm の行へ一括適用を提案し、現場の入力工数を削減する。

## 2. Scope
- 対象：学習ボタン押下 → 保存 → ページ内一括適用提案 → 承認で反映
- 対象外：全ページへの自動適用（Sprint6以降で検討）
- 対象外：ルール管理画面の全件一括適用（RulesManager側）

## 3. Inputs / Outputs
### Inputs
- currentPageRows: TransactionRow[]
- taughtRule:
  - clientText: string
  - descriptionLabel: string
  - bankId: string
- normalizeFn: (text:string) => string

### Outputs
- affectedCount: number
- applyProposal:
  - message: string（例: "同じ取引先が 8 件あります。適用しますか？"）
  - apply: () => void（承認時の更新処理）
  - cancel: () => void

## 4. Rules
1) 一括対象は「同一 clientNorm」一致のみ
2) "既に別のdescriptionが確定済み"の行は上書きしない（安全）
3) IGNORE行には適用しない
4) 必ず確認（ユーザー承認）を挟む

## 5. DoD
- [ ] 学習ボタン押下後に affectedCount が出る
- [ ] 承認した時だけ一括反映される
- [ ] 既確定・IGNOREは安全に除外される

## 6. Test Cases
- Case1: 同一clientNormが複数 → count>0、承認で反映
- Case2: count=0 → 提案なし
- Case3: 既確定が混ざる → 上書きされない
- Case4: IGNORE行 → 反映されない

## 7. Anti-Patterns
- 承認なしで勝手に全置換
- 部分一致で広範囲に誤爆
- IGNOREにも反映してデータが汚れる

## 8. Notes
- "効いてる感" を最優先：count表示は必須
- UIはトースト/ダイアログどちらでも良いが、導線は短く
