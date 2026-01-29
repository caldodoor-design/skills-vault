---
name: auto-applying-on-load
description: OCR直後に学習ルールを自動適用し、ユーザー操作なしで学習が効く状態を作る。危険な場合はSafety Gateで止める。
trigger: OCR結果反映直後の自動適用ロジックを実装・修正するとき
---

# S17: AutoApplyOnLoad

> Updated by **Agent Delta** based on `reports/sprint-006/verification_report.md`
> Validated: Sprint 6 Verification

## 1. Purpose
OCR直後に学習ルール（S13）を自動適用し、ユーザーが「ボタンを押さなくても」学習が効く状態を作る。
ただし誤爆は許さず、危険な場合はS18で止める。

## 2. Scope
- 対象：OCR結果が rows に反映された直後の自動適用
- 対象外：手動学習（Teach）自体（Sprint5のS15）
- 対象外：全ページ自動適用の強制（MVPは "現在ページ/現在データ範囲" を優先）

## 3. Inputs / Outputs
### Inputs
- bankId: string
- rows: TransactionRow[]
- rules: LearningRule[]（S2 RuleStoreから取得）
- resolver: S13（bank優先→default）
- safetyGate: S18（危険行は止める）

### Outputs
- updatedRows: TransactionRow[]
- summary:
  - appliedCount: number
  - warningCount: number
  - blockedCount: number
  - changedRowIds: string[]（Undo用）

## 4. Rules
1) 自動適用の対象行は「未確定/空欄」に限定（安全第一）
   - 例：description が null/空 の行のみ
2) ルール解決は S13 を使用（bank→default）
3) 適用前に S18 Safety Gate を必ず通す
4) 自動適用した変更は Undo できるよう変更履歴を記録する（S19）

## 5. DoD
- [ ] OCR直後に自動でdescriptionが埋まる（対象行のみ）
- [ ] 誤爆しそうな行は自動確定されずWARNING/blockedになる
- [ ] summary（applied/warn/blocked）がUIへ渡せる

## 6. Test Cases
- Case1: 学習ルールあり → OCR直後にdescriptionが自動で入る
- Case2: description既に埋まっている行 → 上書きされない
- Case3: bankとdefault両方存在 → bank優先
- Case4: SafetyGateがBLOCK → 自動適用されない

## 7. Anti-Patterns
- 全行を無条件で上書き（事故る）
- Safety Gate を通さず適用（事故る）
- 変更履歴を残さずUndo不能（怖くて使えない）

## 8. Notes
- MVPは「現在ページ範囲だけ」で十分。全ページはSprint7以降で拡張可。
