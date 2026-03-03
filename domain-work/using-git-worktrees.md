---
name: using-git-worktrees
description: Git Worktreeで隔離ワークスペース作成
trigger: フィーチャー隔離・実装計画の実行開始時
---

# Git Worktree

## ディレクトリ優先順位
1. .worktrees/ の存在確認（両方あれば .worktrees 優先）
2. worktrees/ の存在確認
3. CLAUDE.md設定
4. ユーザーに質問

## 手順
1. プロジェクト名検出
2. `git worktree add` で作成
3. セットアップ自動検出・実行
4. テストでクリーンベースライン検証
5. 配置場所報告

## 注意
- `git check-ignore` でignore確認必須。未ignoreなら .gitignore に追加してコミット
- テスト失敗時は報告+確認（無断続行禁止）
- finishing-development-branch と組み合わせて使用
