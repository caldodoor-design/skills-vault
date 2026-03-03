---
name: auto-applying-on-load
description: OCR直後に学習ルールを自動適用
trigger: OCR結果反映直後の自動適用ロジック実装・修正時
---

# S17: AutoApplyOnLoad

OCR直後に学習ルール（S13）を自動適用し、ユーザー操作なしで学習が効く状態を作る。

## ルール
1. 対象は「未確定/空欄」行のみ（description が null/空）
2. ルール解決はS13使用（bank→default）
3. 適用前にS18 Safety Gateを必ず通す
4. 変更履歴を記録（S19 Undo用）

## I/O
- Input: bankId, rows[], rules[], resolver(S13), safetyGate(S18)
- Output: updatedRows[], appliedCount, warningCount, blockedCount, changedRowIds[]

## 注意
- 全行無条件上書きは禁止
- MVPは「現在ページ範囲だけ」で十分。全ページはSprint7以降
