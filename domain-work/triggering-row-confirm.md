---
name: triggering-row-confirm
description: ユーザーが行を確定（Confirm）した瞬間にlearnRow学習処理を発火させる。編集だけでは学習せず、Confirmボタンが学習の唯一のトリガーとなる。
trigger: 行の確定ボタン（Confirm）押下時の学習発火ロジック、またはlearnRowの呼び出しフローを実装・修正するとき。
---

# S26: RowConfirmTrigger

## 1. Purpose
ユーザーが「この行は正解」と確定（Confirm）した瞬間に、学習処理（learnRow）を確実に発火させる。
単に編集しただけでは学習しない（誤学習防止）。Confirmがトリガーの唯一の入口。

## 2. Scope
- 対象：行単位（TransactionRow）
- 対象UI：行右端のConfirmボタン
- 非対象：自動適用（S17/S20）による確定（人間確認がないため）

## 3. Inputs / Outputs
### Inputs
- rowId
- rowBefore (OCR原文)
- rowAfter (ユーザー修正後)
- context: { bankId, pageIndex, appId }

### Outputs
- result:
  - status: "LEARNED" | "SKIPPED" | "BLOCKED"
  - reason: string
  - ruleId?: string

## 4. Rules
1) rowAfter と rowBefore が同一なら SKIPPED（学習不要）
2) 学習前に Mislearning Guardrails（S16）を必ず通す
3) ルール保存は BankScoped（S13互換）を優先
4) 汎用語/短すぎ/数字だけ等は BLOCKED（S21と整合）
5) 成功したら row.status = CONFIRMED（緑）

## 5. DoD
- Confirm押下で learnRow が必ず呼ばれる
- 学習がSKIPPED/BLOCKEDの場合でもUIが説明できる
- 確定の再現性（同じ入力で同じ判定）

## 6. Test Cases
- TC1: OCRと修正が違う → LEARNED
- TC2: 同一 → SKIPPED
- TC3: "振込"単体修正 → BLOCKED
- TC4: ルール保存後にruleIdが返る

## 7. Anti-Patterns
- 編集だけで勝手に学習する（誤学習汚染）
- ConfirmがあってもlearnRowが呼ばれない
- BLOCKED理由が見えず「覚えない」に見える

## 8. Notes
- 学習の入口は必ずConfirm（Confirm）に統一する。
