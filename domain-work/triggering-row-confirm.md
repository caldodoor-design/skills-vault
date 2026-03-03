---
name: triggering-row-confirm
description: Confirmボタン押下で学習処理を発火（唯一のトリガー）
trigger: Confirm押下時の学習発火ロジック・learnRow呼び出しフロー実装時
---

# S26: RowConfirmTrigger

Confirmボタンが学習の唯一の入口。編集だけでは学習しない（誤学習防止）。

## ルール
1. rowAfter == rowBefore なら SKIPPED（学習不要）
2. 学習前にS16 Mislearning Guardrailsを通す
3. ルール保存はBankScoped（S13互換）
4. 汎用語/短すぎ等はBLOCKED（S21整合）
5. 成功 → row.status = CONFIRMED（緑）

## I/O
- Input: rowId, rowBefore, rowAfter, context{bankId, pageIndex, appId}
- Output: status("LEARNED"|"SKIPPED"|"BLOCKED"), reason, ruleId?

## 注意
- 編集だけで勝手に学習 = 誤学習汚染
- BLOCKED理由が見えないと「覚えない」に見えてUX崩壊
