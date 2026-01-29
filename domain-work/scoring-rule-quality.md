---
name: scoring-rule-quality
description: 学習ルールの危険度を定量化（qualityScore）し、誤爆しやすいルールを自動適用から抑制するスコアリングスキル。
trigger: ルールの品質評価やスコアリングロジックを実装・修正するとき、またはS18/S22の判断材料を検討するとき
---

# S21: RuleQualityScorer

## 1. Purpose
学習ルールの「危険度」を定量化し、誤爆しやすいルールを自動適用から抑制する。
（ルールが増えるほど事故りやすいので"劣化防止"が目的）

## 2. Scope
- 対象：LearningRule（clientNorm -> description など）
- 出力：qualityScore（0.0〜1.0）と理由（reasons）
- 使いどころ：S18 Safety Gate と S22 ConflictResolver の判断材料

## 3. Inputs / Outputs
### Inputs
- rule: LearningRule
- context:
  - bankId
  - stats（任意）：applyCount / undoCount / lastUsedAt 等（取れる範囲で）

### Outputs
- quality:
  - score: number (0.0 - 1.0)
  - level: "ALLOW" | "WARN" | "BLOCK"
  - reasons: string[]

## 4. Rules
スコアは減点方式でよい（理由が説明できることが最優先）

### BLOCK条件（即BLOCK推奨）
- clientNormが短すぎ（例: 1〜2文字）
- 数字/記号だらけ（例: "12345", "***"）
- 汎用語のみ（例: "振込", "振替", "ATM" 単体）
- undoCountが一定以上（例: 2回以上Undoされてる）

### WARN条件（自動適用は抑制 or 要注意）
- 両義語（振込/振替/手数料/利息 等）が含まれる
- 一致が部分一致のみ（曖昧）
- bankスコープがdefaultのみで広がりすぎる

### 目安の閾値
- score < 0.35 -> BLOCK
- 0.35 <= score < 0.60 -> WARN
- score >= 0.60 -> ALLOW

## 5. DoD
- ルールに対して score/level/reasons が返る
- level判定がS18/S22に接続できる
- 同じ入力なら同じ評価（再現性）

## 6. Test Cases
- TC1: "振込" 単体 -> BLOCK
- TC2: "A社"（短いが固有） -> WARN or ALLOW（理由が出る）
- TC3: 数字列 -> BLOCK
- TC4: undoCount=3 -> BLOCK
- TC5: bank固有の長いclientNorm -> ALLOW

## 7. Anti-Patterns
- スコアは出るが理由が出ず、ユーザーが納得できない
- ルールを全部BLOCKして何も効かない
- 評価がランダムでブレる

## 8. Notes
- 重要なのは"賢さ"より"安全側に倒す説明可能な判断"。
- 迷ったら WARN に落として Human-in-the-loop を維持する。
