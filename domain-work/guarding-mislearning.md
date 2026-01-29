---
name: guarding-mislearning
description: ゴミ学習（短すぎ・汎用語・数字だらけ・曖昧語）を学習対象から排除し、ルールDBを汚さないためのバリデーションガード。
trigger: Teach前の入力バリデーションや学習ブロック条件を実装・修正するとき
---

# S16: MislearningGuardrails

> Updated by **Agent Delta** based on `reports/sprint-005/verification_report.md`
> Validated: Sprint 5 Verification

## 1. Purpose
ゴミ学習（短すぎ・汎用語・数字だらけ・曖昧語）を学習対象から排除し、ルールDBを汚さない。

## 2. Scope
- 対象：Teach前の入力バリデーション（保存前に止める）
- 対象外：学習済みルールのクレンジング（後処理は別途）

## 3. Inputs / Outputs
### Inputs
- clientText: string（通帳記載）
- descriptionLabel: string（摘要カテゴリ）
- bankId: string

### Outputs
- allow: boolean
- reason?: string（allow=false の理由：UI表示用）
- severity?: "INFO" | "WARNING" | "BLOCK"

## 4. Rules
最低限のブロック条件（MVP）：
1) 文字数が短すぎる（例：2〜3文字以下）
2) 汎用語のみ（例：振込/振替/入金/出金/ATM/手数料/繰越/合計）
3) 数字が多すぎる（例：数字比率が高い）
4) 空文字・空白だけ

判定結果：
- BLOCK：学習不可（理由表示）
- WARNING：学習できるが注意（MVPはBLOCKだけでも可）

## 5. DoD
- [ ] "振込"単体で学習できない（BLOCK）
- [ ] 短すぎる文字列は学習できない（BLOCK）
- [ ] ブロック理由がUIに出る
- [ ] DBが汚れない（保存前に止める）

## 6. Test Cases
- Case1: "振込" → BLOCK（汎用語）
- Case2: "AB" → BLOCK（短すぎ）
- Case3: "ｼｬｶｲﾎｹﾝﾘｮｳ" → allow=true
- Case4: "123456" → BLOCK（数字だらけ）

## 7. Anti-Patterns
- 学習後に弾く（DBが汚れる）
- 汎用語を許す（爆発的に誤爆が増える）
- 理由なしで弾いて現場が困る

## 8. Notes
- 汎用語リストは将来、銀行別に分けてもよい（ただしMVPは共通で十分）
- "弾く強さ"は後から緩められるが、汚れたDBは戻すのが大変
