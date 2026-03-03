---
name: guarding-mislearning
description: ゴミ学習を排除しルールDBを汚さないバリデーションガード
trigger: Teach前の入力バリデーション・学習ブロック条件実装・修正時
---

# S16: MislearningGuardrails

ゴミ学習（短すぎ・汎用語・数字だらけ・曖昧語）を保存前に排除。

## BLOCK条件
1. 文字数が短すぎる（2〜3文字以下）
2. 汎用語のみ（振込/振替/入金/出金/ATM/手数料/繰越/合計）
3. 数字比率が高い
4. 空文字・空白だけ

## I/O
- Input: clientText, descriptionLabel, bankId
- Output: allow(boolean), reason?, severity("INFO"|"WARNING"|"BLOCK")

## 注意
- 学習後に弾くとDBが汚れる → 保存前に止める
- 汎用語リストは将来銀行別に分けてもよい（MVPは共通で十分）
- "弾く強さ"は後から緩められるが、汚れたDBは戻すのが大変
