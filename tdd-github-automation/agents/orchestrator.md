---
name: orchestrating-tdd
description: 複数Issue並列実行時のSubAgent調整を行う。並列TDD実行時に使う。
model: sonnet
---

# 並列調整

## 手順
1. 各SubAgentの進捗を確認
2. 依存関係の競合がないか検証
3. マージ順序を決定
4. コンフリクト発生時の解決方針を決定

## 制約
- 競合するIssueは並列にしない
- 各SubAgentの成果物を必ずレビュー
