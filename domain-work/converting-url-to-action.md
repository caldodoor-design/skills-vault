---
name: converting-url-to-action
description: URLからコンテンツタイプ（YouTube/記事/PDF）を自動検出→抽出→Ship-Learn-Nextアクションプラン作成
trigger: 「tapestry URL」「weave URL」「URLからプランを作って」
---
# URL → アクションプラン変換（Tapestry）

## フロー
1. URLタイプ検出 → 2. コンテンツ抽出 → 3. アクションプラン作成 → 4. 保存・表示

## URL検出
```bash
if [[ "$URL" =~ youtube\.com/watch || "$URL" =~ youtu\.be/ || "$URL" =~ youtube\.com/shorts ]]; then
    CONTENT_TYPE="youtube"
elif [[ "$URL" =~ \.pdf$ ]] || curl -sI "$URL" | grep -qi "Content-Type: application/pdf"; then
    CONTENT_TYPE="pdf"
else
    CONTENT_TYPE="article"
fi
```

## 抽出方法
| タイプ | ツール | フォールバック |
|--------|--------|---------------|
| YouTube | `yt-dlp --write-auto-sub` → VTT→テキスト変換（重複除去） | - |
| 記事 | `reader` (npm) | `trafilatura` (pip) → `curl` + title抽出 |
| PDF | `curl -L` + `pdftotext` (poppler) | - |

## アクションプラン作成
抽出コンテンツから:
- 実行可能な教訓を抽出（要約ではなく）
- 4-8週間のクエスト定義
- Rep 1（今週出荷可能）〜 Rep 5 の段階設計
- `Ship-Learn-Next Plan - [タイトル].md` として保存

## エラー対応
| 問題 | 対応 |
|------|------|
| 未対応URL | 記事抽出でフォールバック |
| 抽出失敗 | URL到達性確認、別手法試行 |
| ツール未インストール | 自動インストールまたは案内 |
| 空コンテンツ | プラン作成スキップ |
