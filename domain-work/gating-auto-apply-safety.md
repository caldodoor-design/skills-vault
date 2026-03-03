---
name: gating-auto-apply-safety
description: 自動適用の危険ケースを止めるSafety Gate
trigger: 自動適用の安全判定ロジック実装・修正時
---

# S18: AutoApplySafetyGate

自動適用前に通す安全判定。Human-in-the-loopの思想を守る。

## BLOCK条件
1. resolvedDescription が null（該当なし）
2. clientText が汎用語/短すぎ（S16相当）
3. 両義語で方向が確定しない（S14相当）
4. nearBoundary=true（テンプレ信頼度低）

## WARNING
- 自動適用はしないが、ユーザーに確認推奨

## ALLOW
- 上記条件を全てクリアした場合のみ

## I/O
- Input: row, bankId, clientText, resolvedDescription, directionSignal{hasWithdraw,hasDeposit,isAmbiguousKeyword}, templateSignal{templateUsed,nearBoundary}
- Output: verdict("ALLOW"|"WARNING"|"BLOCK"), reason

## 注意
- BLOCKを強めにしておくのが正解。後から緩めるのは簡単
- reasonは必ず返す（現場が納得できるように）
