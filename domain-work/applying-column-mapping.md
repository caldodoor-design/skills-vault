---
name: applying-column-mapping
description: columnMapでOCRトークンを各セルに割り当てる
trigger: OCRトークンの列割り当て処理実装・修正時
---

# S25: ApplyColumnMapping

S23/S24のcolumnMapを使い、OCRトークンを各セル（withdraw/deposit/description/balance等）へ割り当て。

## ルール
1. tokenはcolumnMapのX範囲で列へ割り当て
2. どの列にも入らないtokenはunassignedに送る（WARNING）
3. 割り当てが弱い場合はセルconfidenceを下げる
4. deterministic：同一入力 = 同一割り当て

## I/O
- Input: pageTokens[], columnMap, rowBands, strict flag
- Output: TransactionRow[], assignedCount, unassignedCount, warnings[]

## 注意
- 目的は「列の安定化」。WARNINGが増えるのは安全側でOK
- "無言で捨てる"は禁止（監査不能）
