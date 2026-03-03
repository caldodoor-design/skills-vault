---
name: healing-balance
description: 残高不整合の検出・自動補正提案
trigger: 残高検算・バランスチェック機能の実装・修正時
---

# S6: BalanceSelfHealing

OCR読み取り精度の限界を検算ロジックと自動補正でカバー。

## 検算式
`prevBalance + deposit - withdraw == currentBalance`

## ルール
1. 計算一致 → OK
2. 不一致 → WARNING/ERROR（差額を表示）
3. **Strict Auto-Confirm**: Validation==WARNING & Suggestion存在 & Confidence<0.9の場合のみ自動修正可
4. **High Confidence Safety**: Confidence>=0.9なら不一致でも自動修正禁止（要ユーザー確認）
5. **Swap Logic**: Withdraw/Deposit入替で残高式が成立するなら入替提案

## I/O
- Input: 行データ一式, 直前の確定残高
- Output: VerificationStatus(OK|WARNING|ERROR), SuggestedCorrection

## 注意
- 勝手に数値を書き換えて辻褄を合わせるのは改竄リスク
