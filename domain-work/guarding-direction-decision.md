---
name: guarding-direction-decision
description: 入出金（withdraw/deposit）の誤爆を減らし、曖昧なケースをWARNINGに落として人間確認へ回す方向判定ガード。
trigger: 入出金の方向判定ロジックを実装・修正するとき、または両義語の扱いを検討するとき
---

# S14: DirectionDecisionGuard

> Updated by **Agent Delta** based on `reports/sprint-005/verification_report.md`
> Validated: Sprint 5 Verification

## 1. Purpose
入出金（withdraw/deposit）の誤爆を減らし、曖昧なケースは必ず WARNING に落として人間確認へ回す。

## 2. Scope
- 対象：入出金の確定/抑制/警告の判断
- 対象外：OCR自体の精度改善（Gemini側の話）
- 対象外：残高整合性の最終修復（balance-self-healing側）

## 3. Inputs / Outputs
### Inputs
- rowCandidate: TransactionRow（暫定行）
- signals:
  - hasWithdrawAmount: boolean
  - hasDepositAmount: boolean
  - descriptionLabel: string | null（摘要カテゴリ）
  - clientText: string（通帳記載）
  - bankId: string
  - templateUsed: boolean（Virtual Grid/Template fallback有無）
- optionalRuleHint:
  - preferredDirection?: "WITHDRAW" | "DEPOSIT" | "UNKNOWN"

### Outputs
- decision: "CONFIRM_WITHDRAW" | "CONFIRM_DEPOSIT" | "WARNING_AMBIGUOUS" | "NO_DECISION"
- warningReason?: string（例: "AMBIGUOUS_KEYWORD", "BOTH_AMOUNT_PRESENT"）

## 4. Rules
1) withdraw金額のみ存在 → withdraw確定
2) deposit金額のみ存在 → deposit確定
3) 両方存在 or 両方不在 → WARNING_AMBIGUOUS
4) 両義語（振込/振替/ATM/手数料など）は確定しない（WARNING）
5) 反転提案は「自動採用しない」：提案止まり（Human-in-the-Loop）

## 5. DoD
- [ ] 両義語で勝手に方向確定しない
- [ ] 両方金額がある場合は必ず WARNING
- [ ] 反転は自動採用されず、提案として扱われる

## 6. Test Cases
- Case1: withdrawのみ数値 → CONFIRM_WITHDRAW
- Case2: depositのみ数値 → CONFIRM_DEPOSIT
- Case3: 振込 + どちらにも数値（or なし） → WARNING_AMBIGUOUS
- Case4: templateUsed=true かつ境界近傍 → WARNING（安全側に倒す）

## 7. Anti-Patterns
- キーワードだけで方向確定（危険）
- 反転修正を即時自動反映（事故る）
- WARNINGを出さずに silent に確定する

## 8. Notes
- 方向判定は「安全第一」。確定しないことが正義の場面が多い
- 将来：銀行ごとの方向規則を入れる場合も WARNING制御は維持する
