---
name: automating-tdd-github
description: GitHub IssueからTDDで実装し、PRを作成するまでの一連のフローを定義する。Issue起点の開発タスクで使う。
---

# TDD GitHub Automation

Issue → テスト設計 → テスト実装 → 機能実装 → テスト実行 → PR作成

## ワークフロー

- [ ] 1. Issue解析（agents/issue-analyzer.md）
- [ ] 2. テストFW検出（agents/test-framework-detector.md）
- [ ] 3. テスト設計（agents/test-designer.md）
- [ ] 4. テスト実装（agents/test-implementer.md）
- [ ] 5. 機能実装（agents/feature-implementer.md）
- [ ] 6. テスト実行（agents/test-runner.md）
- [ ] 7. 並列調整（agents/orchestrator.md）※複数Issue時
- [ ] 8. PR作成（agents/pr-composer.md）

## 原則

- テストを先に書く（Red → Green → Refactor）
- 中間成果物＋バリデーションを必ず挟む
- 1Issue = 1PR
