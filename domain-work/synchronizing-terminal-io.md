---
name: synchronizing-terminal-io
description: 対話型ターミナルの入力待ち検知と適応型ポーリングによる完了判定
---
# Terminal Sync Techniques

## プロンプト検知パターン
```typescript
const READY_PATTERNS = [
  /❯\s*$/m,              // Claude Code
  /\?\s*$/m,             // 質問プロンプト
  /waiting\.\.\.\s*$/mi, // 待機中
  /^PS [^>\r\n]*>\s*$/m, // PowerShell
  />>>\s*$/m,            // Python REPL
  /[\$#]\s*$/m,          // Bash/Zsh
];
```

## 完了判定
| ステータス | プロンプト検知 | 判定 |
|-----------|--------------|------|
| DONE | - | 完了 |
| RUNNING | 検出 | 完了（入力待ち） |
| RUNNING | 未検出 | ポーリング継続 |

## 適応型ポーリング
1回目: 30秒 → 2回目: 60秒 → 3回目以降: 120秒

## ルール
- コマンド送信時は末尾に`\n`必須
- 出力truncated時は`OutputCharacterCount`を増やして再取得
- 一括指示を避けステップ分割
