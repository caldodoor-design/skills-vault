---
name: applying-batch-on-confirm
description: Confirm確定の直後に、保存された学習ルールを使って同一摘要の他行を即時一括修正する。作業中にリアルタイムで効果を発揮し、作業体験を劇的に改善する。
trigger: Confirm後の一括適用ロジック、同一摘要行への即時反映処理、またはSafety Gate連携を実装・修正するとき。
---

# S27: InteractiveBatchApplyOnConfirm

## 1. Purpose
Confirm確定の直後に、今保存された学習ルールを使って「同一摘要の他行」を即時に一括修正し、作業体験を劇的に改善する。

## 2. Scope
- 対象：現在画面に表示されている行（ページ/案件）
- 対象：同一 clientNorm / 同一摘要キー（既存正規化ルールに従う）
- 非対象：Safety GateでBLOCKされる行

## 3. Inputs / Outputs
### Inputs
- confirmedRow
- allVisibleRows
- rules (最新。保存直後のルールを含む)
- context: { bankId }

### Outputs
- result:
  - appliedCount
  - blockedCount
  - warnedCount
  - changedRowIds: string[]
  - summaryMessage: string

## 4. Rules
1) まずS15 Interactive Batch Apply を呼ぶ（再利用）
2) 変更対象は CONFIRMED以外（未確定）のみ
3) Safety Gate（S18）を必ず通す
4) 変更した行には "AI_APPLIED"（青）を付ける
5) maxApply上限を必ず守る（例：200）
6) 何件変えたかをUIメッセージで提示する

## 5. DoD
- Confirm押下直後に同一摘要行が即時に更新される
- BLOCK/WARNは止まり、理由が残る
- Undo（S19）で戻せる差分を積む

## 6. Test Cases
- TC1: 同一摘要が10件 → appliedCount=9
- TC2: CONFIRMED行は触らない
- TC3: 危険行はSafety GateでBLOCK
- TC4: Undoで元に戻せる

## 7. Anti-Patterns
- Confirm押しても何も変わらない（体験崩壊）
- CONFIRMEDを上書きする事故
- Safety Gateを通さず誤爆する

## 8. Notes
- "次回リロードで効く"では遅い。作業中に効くのが価値。
