---
name: analyzing-issues
description: GitHub Issueの要件を解析し、実装すべきタスクを構造化する。Issue起点の開発開始時に使う。
model: sonnet
---

# Issue解析

## 入力
- GitHub Issue URL or 番号

## 手順
1. Issueのタイトル・本文・ラベル・コメントを取得
2. 受け入れ条件（Acceptance Criteria）を抽出
3. 影響範囲（ファイル・モジュール）を特定
4. テスト可能な要件に分解

## 出力
- 構造化された要件リスト
- 影響ファイル一覧
- テストケース候補
