---
name: resolving-conflicts
description: 1つのセル/行に複数ルールが刺さる場合、再現性ある優先順位で1つを選び、誤爆とブレを減らすコンフリクト解決スキル。
trigger: 複数ルール競合の解決ロジックを実装・修正するとき、またはS20内での競合処理を検討するとき
---

# S22: ConflictResolver

## 1. Purpose
1つのセル/行に複数ルールが刺さる場合、再現性ある優先順位で1つを選ぶ。
（誤爆とブレを減らし、必ず同じ決定を返す）

## 2. Scope
- 対象：候補ルールが複数存在する状況（exact/partial混在含む）
- 出力：選ばれたルール + デバッグ理由

## 3. Inputs / Outputs
### Inputs
- candidates: LearningRuleMatch[]
  - rule
  - matchType: "EXACT" | "PARTIAL"
  - bankScope: "BANK" | "DEFAULT"
  - quality (S21結果)
- context:
  - bankId

### Outputs
- resolved:
  - selected: LearningRuleMatch | null
  - reason: string
  - ranked: LearningRuleMatch[]（デバッグ用）

## 4. Rules
優先順位（上が強い）

1) bankScope: BANK > DEFAULT
2) matchType: EXACT > PARTIAL
3) quality.level: ALLOW > WARN > BLOCK
4) quality.score: 高い方
5) tie-breaker: ruleId で安定化（辞書順など）

※ BLOCK は原則選ばない（全候補BLOCKなら null を返す）

## 5. DoD
- 同一入力で常に同一ルールが選ばれる
- 競合解決理由が reason に残る
- BLOCK候補は自動適用されない

## 6. Test Cases
- TC1: BANK+EXACT vs DEFAULT+EXACT -> BANKが勝つ
- TC2: BANK+PARTIAL vs BANK+EXACT -> EXACTが勝つ
- TC3: ALLOW vs WARN -> ALLOWが勝つ
- TC4: 全部BLOCK -> null
- TC5: 完全同点 -> ruleIdで固定

## 7. Anti-Patterns
- 競合があるたびに結果が変わる（順序依存）
- BLOCK候補が選ばれて誤爆する
- reasonが無くブラックボックス化する

## 8. Notes
- S20 MultiPageAutoApply 内で必ず使う（競合を放置しない）
- reason/ranked は verification_report にも貼れるようにする
