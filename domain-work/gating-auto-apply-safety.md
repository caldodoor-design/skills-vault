---
name: gating-auto-apply-safety
description: 自動適用が危険なケースを確実に止めるSafety Gate。Human-in-the-loopの思想を守り、危険時は停止・警告を担保する。
trigger: 自動適用の安全判定ロジックを実装・修正するとき、またはBLOCK/WARNING条件を追加するとき
---

# S18: AutoApplySafetyGate

> Updated by **Agent Delta** based on `reports/sprint-006/verification_report.md`
> Validated: Sprint 6 Verification

## 1. Purpose
自動適用が危険なケースを確実に止める。
Human-in-the-loopの思想を守り、「危ない時は止まる・警告を出す」を担保する。

## 2. Scope
- 対象：S17の自動適用直前に通す安全判定
- 対象外：手動編集の制限（ユーザーは最終決定者）
- 対象外：残高整合性の最終修復（別スキル）

## 3. Inputs / Outputs
### Inputs
- row: TransactionRow
- bankId: string
- clientText: string
- resolvedDescription: string | null（S13の結果）
- directionSignal:
  - hasWithdrawAmount: boolean
  - hasDepositAmount: boolean
  - isAmbiguousKeyword: boolean（振込/振替など）
- templateSignal:
  - templateUsed: boolean
  - nearBoundary: boolean（境界近い/信頼度低）

### Outputs
- verdict: "ALLOW" | "WARNING" | "BLOCK"
- reason: string（UI/ログに表示）

## 4. Rules
BLOCK（自動適用禁止）条件（最低限）：
1) resolvedDescription が null（該当なし）
2) clientText が汎用語/短すぎ（S16相当）
3) 両義語で方向が確定しない（S14相当で WARNING/曖昧）
4) nearBoundary=true など、テンプレ信頼度が低い

WARNING：
- 自動適用はしないが、ユーザーに確認推奨として残す（行ステータスWARNING）

ALLOW：
- 条件を満たす場合のみ自動適用OK

## 5. DoD
- [ ] 危険行が自動確定されない
- [ ] reason が必ず返り、現場が納得できる
- [ ] "止めること" が正義として機能する

## 6. Test Cases
- Case1: clientText="振込" → BLOCK
- Case2: nearBoundary=true → WARNING/BLOCK
- Case3: resolvedDescription=null → BLOCK
- Case4: 明確なclientText + bank一致 → ALLOW

## 7. Anti-Patterns
- なんでもALLOW（事故る）
- reasonを返さない（現場が混乱）
- WARNINGなのに自動反映する（思想崩壊）

## 8. Notes
- BLOCKを強めにしておくのが正解。後から緩めるのは簡単。
- "止めたログ" は改善の宝（汎用語リストの育成につながる）
