---
name: applying-column-template
description: BankProfileの比率に基づいてページを垂直分割し、OCRトークンを適切な列に割り当てるSkill。列テンプレート適用やトークンの列アサイン処理で使用する。
trigger: 列テンプレートの適用・トークン割り当て・列境界判定を実装・修正するとき。ColumnTemplateApplierや列割り当てロジックに関わるコード変更時。
---

# S8: ColumnTemplateApplier

> Updated by **Agent Delta** based on `reports/sprint-003/verification_report.md` & `reports/sprint-004/verification_report.md`
> Validated: Sprint 3 & 4 Verification

## 1. Purpose
`BankProfile` (S7) で定義された比率に基づいて、ページを垂直分割し、OCRのトークン（文字）を適切な列（Date, Deposit, Withdraw, etc.）に割り当てる。

## 2. Scope
- 入力：ページの全トークン、Column Ratios
- 出力：列割り当て済みの TransactionRow 配列

## 3. Inputs / Outputs
### Inputs
- tokens: Token[]
- ratios: ColumnRatios

### Outputs
- rows: TransactionRow[]
- warnings: ValidationWarning[]

## 4. Rules
1. **Strict Banding**: 定義された列範囲（X座標）に中心点が含まれるトークンをその列に割り当てる。
2. **Straddle Warning**: トークンが列境界を跨いでおり、どちらの列か判定が際どい場合（Overlap < 70%）、必ず `WARNING` を発生させる。
3. **Boundary Confidence (S11)**: 列境界付近のトークンで信頼度が低い（Confidence < 0.2）場合も WARNING を出す。
4. **Data Integrity**: どの列にも属さないトークン（はみ出し）がある場合も警告する。

## 5. DoD (Definition of Done)
- テンプレート適用により、ヘッダー行がなくてもデータが正しく列に収まる。
- 境界線上の文字に対しては警告が出る（見逃し防止）。

## 6. Test Cases
| Case | Scenario | Expected | Result |
|---|---|---|---|
| A | Token fits in column | Assigned Correctly | PASS |
| B | Token straddles boundary | WARNING Triggered | PASS |

## 7. Anti-Patterns
- 境界線上の文字を勝手にどちらかに割り振って、警告を出さずに処理を進める（金額ミスの原因）。
