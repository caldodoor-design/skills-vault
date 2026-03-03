---
name: auto-applying-multi-page
description: 案件（複数ページ）単位で学習ルールを自動適用
trigger: 複数ページにまたがる自動適用ロジック実装・修正時
---

# S20: MultiPageAutoApply

案件（複数ページ）単位で学習ルールを自動適用し、ページ跨ぎの同一clientNorm横展開を自動化。

## ルール
1. CONFIRMED行は絶対に触らない
2. IGNORE行は対象外
3. 適用候補が複数ある場合はS22 ConflictResolverで1つに決める
4. 適用前にS18 AutoApplySafetyGateを通す
5. 既に同じ値なら変更しない（ノイズ回避）
6. 変更履歴（before/after）をchangesに積む（S19 Undo用）
7. deterministic：同一入力 = 同一結果

## I/O
- Input: pages[], rules[], bankId, dryRun, maxApply(例:200)
- Output: appliedCount, blockedCount, warnedCount, perPageSummary[], changes[], durationMs

## 注意
- 速度が厳しければ dryRun→summary表示→apply の2段階でもOK
