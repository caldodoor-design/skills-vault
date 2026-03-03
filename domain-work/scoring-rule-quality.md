---
name: scoring-rule-quality
description: 学習ルールの危険度を定量化し誤爆を抑制
trigger: ルール品質評価・スコアリングロジック実装・修正時
---

# S21: RuleQualityScorer

学習ルールの危険度を定量化（qualityScore）し、誤爆しやすいルールを自動適用から抑制。

## 減点方式スコアリング

### BLOCK条件（score < 0.35）
- clientNormが短すぎ（1〜2文字）
- 数字/記号だらけ
- 汎用語のみ（振込/振替/ATM単体）
- undoCount >= 2

### WARN条件（0.35 <= score < 0.60）
- 両義語（振込/振替/手数料/利息等）含む
- 部分一致のみ
- defaultスコープで広がりすぎる

### ALLOW（score >= 0.60）

## I/O
- Input: rule, bankId, stats?{applyCount,undoCount,lastUsedAt}
- Output: score(0.0-1.0), level("ALLOW"|"WARN"|"BLOCK"), reasons[]

## 注意
- 迷ったらWARNに落としてHuman-in-the-loopを維持
- S18 Safety GateとS22 ConflictResolverの判断材料として使用
