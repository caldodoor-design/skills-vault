---
name: synchronizing-terminal-io
description: 対話型ターミナル（Claude Code, REPL等）の入力待ち状態をプロンプトパターンで検知し、適応型ポーリングで完了判定を行うテクニック集。
trigger: ターミナルの出力同期や入力待ち状態の検知が必要な時、対話型ツールとの自動連携を実装する時
---

# Terminal Sync Techniques for Antigravity

Antigravityがターミナル上の対話型ツール（Claude Code, REPL等）と確実に同期するためのテクニック集。

## 課題

- 対話型ターミナルの出力は非同期で、いつ「入力待ち」になったか判別しにくい。
- `command_status` の出力が切り詰められ（truncated）、プロンプトが見えないことがある。
- 入力を送っても改行がなければ実行されない。

## プロンプト検知パターン

ターミナルが入力待ち状態であることを示す典型的なパターン：

| ツール | プロンプトパターン |
|--------|-------------------|
| Claude Code | `>`, `>`, `?`, `waiting...`, `waiting for input` |
| PowerShell | `PS C:\...>` |
| Bash | `$`, `#` |
| Python REPL | `>>>` |

```typescript
// 入力待ちプロンプトを検知する正規表現
const READY_PATTERNS = [
  /❯\s*$/m,                // Claude Code
  /\?\s*$/m,               // 質問プロンプト
  /waiting\.\.\.\s*$/mi,    // 待機中
  /waiting for input\s*$/mi,
  /^>\s*$/m,               // 一般的な REPL (> で始まる行)
  /^PS [^>\r\n]*>\s*$/m,    // PowerShell
  /^[\w\.\-]+@[\w\.\-]+:.* \s*[\$#]\s*$/m, // Linux Bash (スペース入り)
  /^[^\r\n]* \s*[\$#]\s*$/m, // 一般的な Bash/Zsh (スペース入り)
  />>>\s*$/m,              // Python REPL
];

function isTerminalReady(output: string): boolean {
  return READY_PATTERNS.some(pattern => pattern.test(output));
}
```

## 入力送信の確実化

`send_command_input` を使用する際の必須ルール：

1. **必ず末尾に改行 (`\n`) を含める**
2. **空改行の追加送信**（念のため `\n` のみを追送）

## 完了判定ワークフロー

### 判定ロジック

| ステータス | プロンプト検知 | 判定 | 次のアクション |
|-----------|---------------|------|---------------|
| `DONE` | - | **完了** | 結果を処理、次のタスクへ |
| `RUNNING` | **検出** | **完了（入力待ち）** | 入力待ち状態として扱い、次のコマンドを送信可能 |
| `RUNNING` | 未検出 | **継続** | 適応型ポーリングで再確認 |

### 実装フロー

1. `send_command_input` でコマンド送信 (末尾に `\n` を含む)
2. `command_status` (Wait: 30秒) で確認
3. `Status == DONE` なら完了。結果を処理。
4. `Status == RUNNING` かつ `READY_PATTERNS` 一致なら完了（入力待ち）。次のコマンド送信可能。
5. `Status == RUNNING` かつプロンプトなしならポーリング継続 (次は60秒待機)
6. `command_status` (Wait: 60秒) で再確認
7. 以降 120秒間隔で継続

## 適応型ポーリング (Adaptive Polling)

| ポーリング回数 | WaitDurationSeconds |
|---------------|---------------------|
| 1回目 | 30秒 |
| 2回目 | 60秒 |
| 3回目以降 | 120秒 |

**ルール**: ステータスが `RUNNING` である限り、この間隔でポーリングを継続する。

## トラブルシューティング

- **出力が truncated で見えない**: `OutputCharacterCount` を増やして再取得するか、`read_terminal` を使用。
- **コマンドが実行されない**: 改行 (`\n`) が含まれているか確認。
- **複数行プロンプト**: ZshやPowerShellの `>>` といった継続入力プロンプトは、現在の `READY_PATTERNS` では判定を誤る可能性があります。必要に応じてパターンを追加してください。
- **フリーズ**: 一括指示（「修正してビルドしてレビューして」）を避け、ステップを分割する。
