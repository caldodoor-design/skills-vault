---
name: asking-codex-review
description: Codex CLIでコードレビュー・デバッグ・生成を行う手順
skills: [completing-task-with-contract]
---
# ask-codex Skill

## コマンド構文

```bash
codex exec --skip-git-repo-check -C '.' 'PROMPT'
```

- `--skip-git-repo-check`: **必須**（'Not inside trusted directory'エラー回避）
- MCP経由の呼び出しは**禁止**（フリーズリスク）

## Windows環境でファイルアクセスが必要な場合

```bash
codex exec --skip-git-repo-check --dangerously-bypass-approvals-and-sandbox -C "." "プロンプト"
```
`--dangerously-bypass-approvals-and-sandbox` はコードレビュー等の読み取り専用操作のみに限定。

## 日本語ドキュメント検証時

```bash
codex exec --skip-git-repo-check -C '.' "Review FILE.md. This is a Japanese document. Focus strictly on CODE blocks only. If encoding issues in prose, IGNORE them. Reply with Issues: 0件 if correct."
```

## レビュー結果の対応
| Severity | 対応 |
|----------|------|
| Minor | 修正して報告（再レビュー不要） |
| Major | 修正後、再レビュー |
| Blocker | ユーザーへエスカレーション |
