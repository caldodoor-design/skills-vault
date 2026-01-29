---
name: healing-balance
description: OCR読み取り精度の限界を検算ロジックと自動補正でカバーし、残高不整合の検出・修正提案を行うSkill。残高検証やバランスチェック機能の実装時に使用する。
trigger: 残高の検算・自動補正・バランスチェックを実装・修正するとき。BalanceSelfHealingやvalidateRow、recalculateBalancesに関わるコード変更時。
---

# S6: BalanceSelfHealing

> Updated by **Agent Delta** based on `reports/sprint-002/verification_report.md`
> Validated: Sprint 2 Verification

## 1. Purpose
OCR読み取り精度の限界を、計算ロジックによる検算と自動補正でカバーし、ユーザーの手修正負担を減らす。

## 2. Scope
- `LogicEngine.validateRow` および `recalculateBalances`

## 3. Inputs / Outputs
- **Inputs**: 行データ一式、直前の確定残高
- **Outputs**: `VerificationStatus` (OK | WARNING | ERROR), `SuggestedCorrection`

## 4. Rules
1. **検算式**: `prevBalance + deposit - withdraw == currentBalance`
2. **Strict Auto-Confirm**:
   - 計算が完全に一致する
   - かつ、OCR自体のConfidenceが一定以下（または不明瞭）の場合
   - 数値が「1」と「l」の誤読など、よくあるパターンである場合、自動修正を適用してもよい（要慎重設計）。
3. **Warning**: 計算が合わない場合は必ずユーザーに警告を表示する。

### Verified Constraints (Sprint 2)
- **Strict Auto-Confirm**: Only if `Validation == WARNING` AND `Suggestion Exists` AND `TargetCell.Confidence < 0.9`.
- **High Confidence Safety**: If `Confidence >= 0.9`, do NOT auto-fix even if balance overlaps (require user confirmation).
- **Swap Logic**: Reverse `Withdraw` and `Deposit` if it satisfies balance equation.

## 5. DoD
- 計算が合う行は緑色（OK）になり、ユーザーが確認する必要がない状態になること。
- 計算が合わない行は赤/黄色になり、どこが間違っているか（差額など）が示されること。

## 6. Test Cases
- Prev=1000, Dep=500, Wit=0, Curr=1500 -> OK.
- Prev=1000, Dep=500, Wit=0, Curr=1400 -> ERROR (Diff -100).

## 7. Anti-Patterns
- 勝手に数値を書き換えて、結果的に辻褄を合わせること（事実と異なる改竄のリスク）。
- 自動補正を過信して、ユーザー確認をスキップすること。

## 8. Notes
- Sprint 2 での実装予定。誤読パターンの辞書（`0` <-> `O` など）と組み合わせると強力。
