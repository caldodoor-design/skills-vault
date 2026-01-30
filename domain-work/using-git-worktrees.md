---
name: using-git-worktrees
description: Git Worktreeで隔離ワークスペースを作成する。フィーチャー隔離や実装計画の実行前に使用。
trigger: フィーチャー作業の隔離、実装計画の実行開始時
---

# Git Worktree の使用

## 概要

Git Worktreeは同一リポジトリを共有する隔離ワークスペースを作成し、ブランチ切り替えなしで複数ブランチの同時作業を可能にする。

## ディレクトリ選択プロセス

優先順位: 既存ディレクトリ > CLAUDE.md設定 > ユーザー確認

1. .worktrees/ または worktrees/ の存在確認（両方あれば .worktrees 優先）
2. CLAUDE.md にworktreeディレクトリ設定があればそれを使用
3. なければユーザーに選択肢を提示

## 安全性検証

プロジェクトローカルの場合、git check-ignore でignore確認必須。
未ignoreなら .gitignore に追加してコミットしてから進む。

## 作成手順

1. プロジェクト名検出
2. git worktree add で作成
3. プロジェクトセットアップ自動検出・実行
4. テストでクリーンベースライン検証
5. 配置場所報告

## クイックリファレンス

| 状況 | アクション |
|------|-----------|
| .worktrees/ が存在 | 使用（ignore確認） |
| worktrees/ が存在 | 使用（ignore確認） |
| 両方存在 | .worktrees/ を使用 |
| どちらも存在しない | CLAUDE.md確認後ユーザーに質問 |
| ignore未設定 | .gitignoreに追加+コミット |
| テスト失敗 | 報告+確認 |

## よくあるミス

- ignore検証スキップでgit status汚染
- ディレクトリ配置の想定（優先順位に従う）
- テスト失敗時の無断続行
- セットアップコマンドのハードコード

## 連携

- finishing-development-branch と組み合わせて使用

## 元リポジトリ

https://github.com/obra/superpowers