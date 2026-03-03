---
name: deciding-direction
description: 入出金列の誤判定・列逆転を検出・防止
trigger: 入出金方向判定や残高整合性チェックの実装・修正時
---

# S5: DirectionDecision

入金・出金列の誤判定やレイアウト崩れによる列逆転を検出・防止。

## ルール
1. **残高整合性優先**: `前残高 - 出金 + 入金 = 当残高` が成立するパターンを正とする
2. **キーワード判定**: 「振込」「利息」等は補助ヒントのみ（Confidence 0.5-0.7）
3. **Safety**: どちらとも取れる場合はWARNING、自動決定しない

## Confidence基準
- 残高計算一致: 0.9
- キーワードのみ: 0.5-0.7
- 両列に値あり: AMBIGUOUS (0.0)

## I/O
- Input: withdraw, deposit, prevBalance, currentBalance, description
- Output: Direction(WITHDRAW|DEPOSIT|AMBIGUOUS), Confidence, Suggestion

## 注意
- 残高計算を無視してキーワードだけで列入替は禁止
