---
name: asking-codex-review
description: Codex CLIを使ったコードレビュー・デバッグ・生成の呼び出し方法と結果の解釈方法を定義する。コード変更後のレビュー実施時に使う。
skills: [completing-task-with-contract]
---

# ask-codex Skill

Codex CLIを使用してコーディング支援を受けるためのスキル。

## 使用目的

- セカンドオピニオンの取得
- コード生成
- デバッグ支援
- コーディングタスクの委任
- コードレビュー

## コマンド構文

```bash
codex exec --skip-git-repo-check -C '.' 'PROMPT'
```

### パラメータ説明

| パラメータ | 説明 |
|-----------|------|
| `exec` | Codexを実行モードで起動 |
| `--skip-git-repo-check` | **必須** - 'Not inside trusted directory' エラーを回避 |
| `-C '.'` | カレントディレクトリをワーキングディレクトリとして指定 |
| `'PROMPT'` | Codexに送信するプロンプト（シングルクォートで囲む） |

## 重要な注意事項

### MCP使用禁止

MCP (Model Context Protocol) 経由でのCodex呼び出しは**禁止**。フリーズのリスクがあるため、必ず `codex exec` コマンドを使用すること。

### --skip-git-repo-check フラグ

このフラグは必須。省略すると以下のエラーが発生する可能性がある：
```
Error: Not inside trusted directory
```

## 使用例

### コードレビュー依頼

```bash
codex exec --skip-git-repo-check -C '.' '以下のファイルをレビューしてください。対象: src/components/Button.tsx 重点チェック項目: セキュリティ、エラーハンドリング'
```

### デバッグ支援

```bash
codex exec --skip-git-repo-check -C '.' 'この関数のバグを特定してください: [コード or ファイルパス]'
```

### コード生成

```bash
codex exec --skip-git-repo-check -C '.' 'ユーザー認証のためのミドルウェアを作成してください'
```

### Antigravity専用: 日本語ドキュメントの検証

日本語ドキュメントのロジックを検証する際、以下の指示をプロンプトに加えることで、環境依存の文字化け（mojibake）に起因する誤判定を回避できる。

```bash
codex exec --skip-git-repo-check -C '.' "Review .agent/skills/FILE_NAME.md. This is a Japanese document. Focus strictly on the logic in CODE blocks only. If you see encoding issues in the prose, IGNORE them. Reply with Issues: 0件 if code logic is correct."
```

## レビュー結果の解釈

Codexからのレビュー結果は以下のステータスで返される：

- **OK ✅** - 問題なし
- **Issues Found ⚠️** - 問題あり（Severity: Minor/Major/Blocker）

### 問題発見時の対応

| Severity | 対応 |
|----------|------|
| Minor | 修正してから報告（再レビュー不要） |
| Major | 修正後、再度Codexにレビュー依頼 |
| Blocker | 即座にAntigravityへエスカレーション |

---

## Windows環境でのファイルアクセス

### 問題

Windows環境では、標準のサンドボックスオプション（`-s workspace-write`, `--full-auto`）を使用しても、Codexがファイルにアクセスできない場合がある。

```
sandbox: read-only
`powershell ...` rejected: blocked by policy
```

### サンドボックスオプション一覧

| オプション | 説明 | Windows対応 |
|-----------|------|-------------|
| `-s read-only` | 読み取り専用（デフォルト） | ❌ ブロック |
| `-s workspace-write` | ワークスペース読み書き | ❌ ブロック |
| `--full-auto` | 自動実行モード | ❌ ブロック |
| `--dangerously-bypass-approvals-and-sandbox` | サンドボックス完全無効化 | ✅ 動作 |

### 解決策：ファイルアクセスが必要な場合

```bash
codex exec --skip-git-repo-check --dangerously-bypass-approvals-and-sandbox -C "." "プロンプト"
```

### ⚠️ セキュリティ警告

`--dangerously-bypass-approvals-and-sandbox` はサンドボックスを**完全に無効化**する。以下の用途のみに限定すること：

- ✅ コードレビュー（読み取り専用操作）
- ✅ ファイル分析
- ❌ 信頼できないコードの実行は禁止

### 代替策：コードを直接渡す

ファイルアクセスを避けたい場合は、プロンプトにコードを直接埋め込む：

```bash
codex exec --skip-git-repo-check -C "." "Review this code: [コードをここに貼り付け]"
```