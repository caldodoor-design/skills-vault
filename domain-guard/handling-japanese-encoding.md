---
name: handling-japanese-encoding
description: Windows PowerShellでのUTF-8/CP932文字化け防止策
skills: [asking-codex-review]
---
# Japanese Encoding

## 根本原因
Windows PowerShell = CP932、PowerShell Core 7+ = UTF-8。UTF-8ファイルをCP932で読むと文字化け。

## 対策

**ファイル作成**: Claude CodeのWriteツール推奨（内部UTF-8）。

PowerShell使用時:
```powershell
chcp 65001
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("path/to/file.md", $content, $utf8NoBom)
```

**ファイル読み取り**: `Get-Content -Path "file" -Encoding UTF8`

**Codexレビュー時**: 英語版一時ファイルでロジック検証、またはCodexに「コードブロックのみ検証」と指示。

## トラブルシューティング
| 症状 | 原因 | 解決策 |
|------|------|--------|
| `縺ｮ` `繧` | UTF-8→CP932誤読 | `-Encoding UTF8` 指定 |
| `?????` | 文字コード範囲外 | BOM付きUTF-8で保存 |
