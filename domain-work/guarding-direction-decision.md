---
name: guarding-direction-decision
description: 入出金の誤爆を減らし曖昧ケースをWARNINGに落とすガード
trigger: 入出金方向判定ロジック・両義語の扱い実装・修正時
---

# S14: DirectionDecisionGuard

入出金の誤爆を減らし、曖昧なケースは必ずWARNINGに落として人間確認へ回す。

## ルール
1. withdraw金額のみ → CONFIRM_WITHDRAW
2. deposit金額のみ → CONFIRM_DEPOSIT
3. 両方存在 or 両方不在 → WARNING_AMBIGUOUS
4. 両義語（振込/振替/ATM/手数料など）→ WARNING（確定しない）
5. 反転提案は「提案止まり」（自動採用しない）

## I/O
- Input: rowCandidate, signals{hasWithdraw,hasDeposit,descriptionLabel,clientText,bankId,templateUsed}, optionalRuleHint
- Output: decision("CONFIRM_WITHDRAW"|"CONFIRM_DEPOSIT"|"WARNING_AMBIGUOUS"|"NO_DECISION"), warningReason?

## 注意
- 方向判定は「安全第一」。確定しないことが正義の場面が多い
