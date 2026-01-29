---
name: deciding-direction
description: PDF/画像OCR特有の入金・出金列の誤判定やレイアウト崩れによる列逆転を検出・防止するSkill。残高整合性チェックやDirection判定ロジックの実装時に使用する。
trigger: 入出金の方向判定（WITHDRAW/DEPOSIT）や残高整合性チェックを実装・修正するとき。DirectionDecisionやLogicEngineの方向判定に関わるコード変更時。
---

# S5: DirectionDecision

> Updated by **Agent Delta** based on `reports/sprint-002/verification_report.md`
> Validated: Sprint 2 Verification

## 1. Purpose
PDFや画像OCR特有の「入金・出金列の誤判定」や「レイアウト崩れによる逆転」を防止・検出する。

## 2. Scope
- `LogicEngine` または専用の判定サービス
- データパース時の後処理

## 3. Inputs / Outputs
- **Inputs**: `withdraw` (value), `deposit` (value), `prevBalance`, `currentBalance`, `description` (hint)
- **Outputs**: `Direction` (WITHDRAW | DEPOSIT | AMBIGUOUS), `Confidence`, `Suggestion`

## 4. Rules
1. **残高整合性優先**: `前残高 - 出金 + 入金 = 当残高` が成立するパターンを正とする。
2. **キーワード判定**: 「振込」「利息」などが、通常どちらの列に来るべきかのヒントとして利用する（補助）。
3. **Safety**: どちらとも取れる場合（計算が合わない、かつキーワードもない）は `WARNING` を出し、自動決定しない。

### Verified Constraints (Sprint 2)
- **Confidence Threshold**: Direction suggestion is `AMBIGUOUS` (0.0) if both columns have values.
- **Balance Logic**: `prevBalance - currentBalance` > 1 implies Withdraw (Confidence 0.9).
- **Keyword**: Only used as weak hint (Confidence 0.5-0.7).
2. **キーワード判定**: 「振込」「利息」などが、通常どちらの列に来るべきかのヒントとして利用する（補助）。
3. **Safety**: どちらとも取れる場合（計算が合わない、かつキーワードもない）は `WARNING` を出し、自動決定しない。

### Updated via Verification Report
- **Confidence Threshold**: Direction suggestion is `AMBIGUOUS` (0.0) if both columns have values.
- **Balance Logic**: `prevBalance - currentBalance` > 1 implies Withdraw (Confidence 0.9).
- **Keyword**: Only used as weak hint (Confidence 0.5-0.7).

## 5. DoD
- 入出金列が逆転しているケースで、残高計算に基づいて修正提案が出ること。
- 「振替」のような両義的な言葉だけで安易に反転しないこと。

## 6. Test Cases
- Case: Dep=0, Wit=1000, Bal decreases by 1000 -> Confirmed WITHDRAW.
- Case: Dep=1000, Wit=0, Bal decreases by 1000 -> Inferred WITHDRAW (Column Shift suspected).

## 7. Anti-Patterns
- 残高計算を無視して、キーワードだけで列を入れ替えること（OCRの数値ミスと競合して混乱する）。

## 8. Notes
- Sprint 2 での実装予定。
