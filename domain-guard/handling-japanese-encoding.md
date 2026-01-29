---
name: handling-japanese-encoding
description: Windows PowerShell環境でのUTF-8/CP932文字化け防止策とCodexレビュー時の日本語対策を定義する。日本語ファイルの読み書きやCodexに日本語ドキュメントを渡す時に使う。
skills: [asking-codex-review]
---

# Japanese Encoding Best Practices for Antigravity

Windows PowerShell環境で日本語ファイルを扱う際の文字化け（mojibake）を防ぐためのベストプラクティス集。

## 1. 根本原因

| 環境 | デフォルトエンコーディング |
|------|---------------------------|
| Windows PowerShell | CP932 (Shift-JIS) |
| PowerShell Core 7+ | UTF-8 |
| Codex CLI | PowerShell経由で読み取り → CP932として解釈 |

UTF-8で保存された日本語ファイルを、CP932として読み取ると文字化けが発生する。

## 2. 解決策

### 2.1 ファイル作成時

**推奨: Antigravityの `write_to_file` ツールを使用**

```
このツールは内部的にUTF-8で書き込むため、PowerShellを介さずに正しいエンコーディングでファイルを作成できる。
```

**PowerShellを使う場合:**

```powershell
# コンソールエンコーディングをUTF-8に設定
chcp 65001
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# UTF-8 (BOMなし) で書き込み
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("path/to/file.md", $content, $utf8NoBom)

# UTF-8 (BOM付き) で書き込み - 一部のツールはBOMを必要とする
$utf8WithBom = New-Object System.Text.UTF8Encoding $true
[System.IO.File]::WriteAllText("path/to/file.md", $content, $utf8WithBom)
```

### 2.2 ファイル読み取り時

```powershell
# 明示的にUTF-8で読み取り
Get-Content -Path "path/to/file.md" -Encoding UTF8
```

### 2.3 Codexレビュー時の対策

Codexはファイルを読み取る際にPowerShellを使用する可能性があり、これが文字化けの原因となる。

**対策A: 英語版で論理検証**
1. 英語版の一時ファイル (`.en.md`) を作成
2. Codexでロジック検証 (Issues: 0件を確認)
3. 検証済みロジックを日本語版に移植
4. 一時ファイルを削除

**対策B: コードブロックのみ検証を依頼**
```
Codexへの指示例:
"This is a Japanese document. If you see encoding issues in the prose, 
ignore them and focus on the CODE blocks only."
```

## 3. チェックリスト

- [ ] `write_to_file` ツールを優先使用
- [ ] PowerShell使用時は `chcp 65001` を先に実行
- [ ] ファイル読み取りは `-Encoding UTF8` を指定
- [ ] Codexレビュー時は「コードのみ検証」を明示

## 4. トラブルシューティング

| 症状 | 原因 | 解決策 |
|------|------|--------|
| `縺ｮ` `繧` などの文字列 | UTF-8をCP932として読み取り | `-Encoding UTF8` を指定 |
| `?????` | 文字コード範囲外 | BOM付きUTF-8で保存 |
| 一部のみ文字化け | here-string内でのエンコーディング混在 | ファイルから読み込むか `write_to_file` を使用 |
