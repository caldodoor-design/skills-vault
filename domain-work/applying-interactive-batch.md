---
name: applying-interactive-batch
description: 学習した瞬間に同一clientNormの行へ一括適用を提案
trigger: 学習ボタン押下後のページ内一括適用提案実装・修正時
---

# S15: InteractiveBatchApply

学習（Teach）した瞬間に同一clientNormの行へ一括適用を提案し、承認後に反映。

## ルール
1. 対象は「同一clientNorm」一致のみ
2. 既にdescription確定済みの行は上書きしない
3. IGNORE行は対象外
4. 必ずユーザー承認を挟む

## I/O
- Input: currentPageRows[], taughtRule{clientText, descriptionLabel, bankId}, normalizeFn
- Output: affectedCount, applyProposal{message, apply(), cancel()}

## 注意
- "効いてる感"を最優先：count表示は必須
- 承認なしで勝手に全置換は禁止
