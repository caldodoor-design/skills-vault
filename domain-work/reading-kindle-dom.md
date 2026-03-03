---
name: reading-kindle-dom
description: Kindle Cloud Readerの主要DOM要素セレクタ辞書
skills: [analyzing-kindle-dom-heuristics]
---
# Kindle Cloud Reader DOM セレクタ

| 要素 | セレクタ | 備考 |
|------|---------|------|
| コンテンツIframe | `iframe[id^='column_']` | `#kr-renderer`内、複数に分割描画 |
| Iframe内画像 | `img` | 各Iframe内の実体 |
| 次ページ | `#kr-chevron-right` | コンテナ: `.kr-chevron-container-right` |
| 前ページ | `#kr-chevron-left` | 対称構造 |
| ローディング | `.loader` | `display: none`で非表示化を監視 |
| ページ番号 | （未調査） | |
| 書籍タイトル | （未調査） | |

## 注意
- DOM構造はKindleアップデートで変更される可能性
- 動的ハッシュ付きクラス名には属性セレクタを検討
