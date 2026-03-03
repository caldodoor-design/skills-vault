---
name: resolving-bank-scoped-rules
description: 学習ルールを銀行スコープで安全に解決（bank→default→なし）
trigger: 学習ルールの解決・適用順序管理の実装・修正時
---

# S13: BankScopedRuleResolver

学習ルール（clientNorm→description）を銀行スコープ（bankId）で安全に解決。

## 優先順位（固定）
1. bankId一致ルール
2. defaultルール
3. なし（description=null）

## ルール
1. 一致判定はclientNormの**完全一致**（normalize後）
2. "ALL"スコープは閲覧専用。適用系ロジックには使わない
3. 複数ヒット時は最新timestamp/ID優先で固定
4. 該当なし → description=null

## I/O
- Input: bankId, clientText, rules[]
- Output: resolved{description, matchedRuleId, matchedScope("BANK"|"DEFAULT"|"NONE")}, confidence, reason

## 注意
- 部分一致で適用して誤爆しない
- bankId無視して全ルールからヒットさせない
