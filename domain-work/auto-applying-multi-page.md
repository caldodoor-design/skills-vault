---
name: auto-applying-multi-page
description: 案件（複数ページ）単位で学習ルールの自動適用を実行し、ページ跨ぎの同一clientNorm横展開を自動化する。
trigger: 複数ページにまたがる自動適用ロジックを実装・修正するとき
---

# S20: MultiPageAutoApply

## 1. Purpose
案件（複数ページ）単位で学習ルールの自動適用を実行し、
ページ単体では見落とす「同一clientNormの横展開」を自動化する。
ただし誤爆はS18 Safety Gateで止め、Undo対象（S19）として追跡する。

## 2. Scope
- 対象：OCR後に生成された全ページの TransactionRow 群
- 対象列：description / client / note など「学習で上書きされる」セル
- 非対象：金額、日付、残高（ここはS5/S6と検算が絡むためSprint7では触らない）

## 3. Inputs / Outputs
### Inputs
- pages: Page[]（複数ページの行データ）
- rules: LearningRule[]（S13 resolverで取得した適用候補）
- opts:
  - bankId
  - dryRun (boolean)
  - maxApply (number) 例: 200（安全上限）

### Outputs
- result:
  - appliedCount
  - blockedCount
  - warnedCount
  - perPageSummary: {pageIndex, applied, blocked, warned}[]
  - changes: ChangeRecord[]（Undo用に必要な差分）
  - reasonsTop: string[]（ブロック理由上位）
  - durationMs

## 4. Rules
1) 「手動で確定された行（CONFIRMED）」は絶対に触らない
2) 「IGNORE」は対象外（CSVにも出ない）
3) 適用候補が複数ある場合は S22 ConflictResolver で1つに決める
4) 適用前に必ず S18 AutoApplySafetyGate を通す（危険は止める）
5) 適用対象セルが「すでに同じ値」なら変更しない（ノイズ回避）
6) 変更履歴（before/after）を changes に積む（S19 Undo用）
7) deterministic：同じ入力なら同じ結果（並び順固定）

## 5. DoD
- 3ページ以上の案件で「同一clientNormが全ページに適用」される
- 競合が起きても毎回同じルールが選ばれる（S22連携）
- Safety Gateで止まるものは必ず止まる
- Undo対象changesが生成される（S19と互換）

## 6. Test Cases
- TC1: 3ページに同一clientNormが分散 → 全ページで同一置換が走る
- TC2: 1ページに手動修正（CONFIRMED）混在 → そこだけ触られない
- TC3: 汎用語/短すぎ/曖昧 → Safety GateでBLOCK/WARNING
- TC4: 競合（複数ルール一致） → S22で決定・再現性あり
- TC5: 1000行でも固まらない（maxApply上限で安全）

## 7. Anti-Patterns
- ページ単体で適用して、ページ跨ぎで結果がブレる
- CONFIRMEDを上書きして事故る
- Safety Gateを通さず誤爆する
- changesを積まずUndo不能にする

## 8. Notes
- Sprint6のS17は「OCR直後に自動適用」の入口。Sprint7ではその適用粒度を案件へ拡張する。
- 速度が厳しい場合は dryRun → summary表示 → apply の2段階でもよい（ただし本Sprintではapplyまで完了させる）。
