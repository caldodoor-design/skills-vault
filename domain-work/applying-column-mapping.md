---
name: applying-column-mapping
description: columnMapを使ってOCRトークンを各セル（withdraw/deposit/description/balance等）に割り当て、TransactionRowの品質を安定させる。割り当て不能なトークンはWARNINGとして可視化する。
trigger: OCRトークンをcolumnMapに基づいて行・セルへ割り当てる処理を実装・修正するとき。
---

# S25: ApplyColumnMapping

## 1. Purpose
S23で推定/ S24で取得したcolumnMapを使い、
OCR tokens を各セル（withdraw/deposit/description/balance等）へ割り当てて
TransactionRowの品質を安定させる。

## 2. Scope
- 対象：ページ内tokens -> 行/セル割り当て
- 出力：TransactionRow[]（セルのbbox/confidence含む）
- 非対象：残高検算や入出金方向確定（S5/S6が担当）

## 3. Inputs / Outputs
### Inputs
- pageTokens: token[]
- columnMap
- rowBands: 行のY範囲推定（既存ロジック or 簡易でもOK）
- options:
  - strict: boolean

### Outputs
- rows: TransactionRow[]
- statusSummary:
  - assignedCount
  - unassignedCount
  - warnings: string[]

## 4. Rules
1) tokenは columnMap のX範囲で列へ割り当てる
2) どの列にも入らないtokenは misc or unassigned に送る（WARNING）
3) 割り当てが弱い場合はセルconfidenceを下げる（後段でWARNINGに）
4) deterministic：同じ入力は同じ割り当て

## 5. DoD
- 列割り当てが安定する
- どこにも入らないtokenが可視化される（warning）
- 同一入力で同一結果

## 6. Test Cases
- TC1: 正しいcolumnMapで期待列に入る
- TC2: columnMapがズレたらWARNINGが出る
- TC3: tokenが境界上 → strictならWARNING
- TC4: 同一入力で同一出力

## 7. Anti-Patterns
- "無言で捨てる"（監査不能）
- ランダム割り当てでブレる
- WARNING無しで誤った列に入れる

## 8. Notes
- 目的は「列の安定化」。黄色が増えるのは安全側でOK。
