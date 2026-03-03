---
name: resolving-conflicts
description: 複数ルール競合を再現性ある優先順位で解決
trigger: 複数ルール競合の解決ロジック実装・修正時
---

# S22: ConflictResolver

1つのセル/行に複数ルールが刺さる場合、再現性ある優先順位で1つを選ぶ。

## 優先順位（上が強い）
1. bankScope: BANK > DEFAULT
2. matchType: EXACT > PARTIAL
3. quality.level: ALLOW > WARN > BLOCK
4. quality.score: 高い方
5. tie-breaker: ruleId（辞書順で安定化）

※ 全候補BLOCKなら null を返す

## I/O
- Input: candidates[]{rule, matchType, bankScope, quality}, bankId
- Output: selected(LearningRuleMatch|null), reason, ranked[]

## 注意
- 同一入力で常に同一結果（deterministic）
- BLOCK候補は自動適用されない
- S20 MultiPageAutoApply内で必ず使う
