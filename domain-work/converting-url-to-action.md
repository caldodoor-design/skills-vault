---
name: converting-url-to-action
description: URLからコンテンツタイプ（YouTube/記事/PDF）を自動検出し、抽出してアクションプランを作成する統合ワークフロー。「tapestry URL」「weave URL」等のトリガーで発動する。
trigger: ユーザーが「tapestry URL」「weave URL」「URLからプランを作って」「これをアクションに変換して」等と依頼した時
---

# URL → アクションプラン変換（Tapestry）

URLからコンテンツタイプを自動検出し、適切なスキルで抽出した上で、Ship-Learn-Next アクションプランを自動生成する統合ワークフロー。

## 発動条件

- 「tapestry [URL]」と言われた時
- 「weave [URL]」と言われた時
- 「URLからプランを作って」と言われた時
- 「これをアクションに変換して」と言われた時
- 「この内容から学んで実装して」と言われた時

**キーワード**: tapestry, weave, プラン, アクション, 抽出してプラン

> 元リポジトリ: https://github.com/nicehash/tapestry

## 処理フロー

1. **URLタイプを検出**（YouTube / 記事 / PDF）
2. **適切なスキルでコンテンツ抽出**
3. **アクションプランを自動作成**
4. **コンテンツファイルとプランファイルの両方を保存**
5. **サマリーをユーザーに表示**

## URL検出ロジック

### YouTube動画

検出パターン:
- `youtube.com/watch?v=`
- `youtu.be/`
- `youtube.com/shorts/`
- `m.youtube.com/watch?v=`

→ `extracting-youtube-transcript` スキルを使用

### Web記事/ブログ

検出パターン:
- `http://` または `https://`（YouTube/PDFを除く）
- medium.com, substack.com, dev.to 等

→ `extracting-articles` スキルを使用

### PDFドキュメント

検出パターン:
- URLが `.pdf` で終わる
- `Content-Type: application/pdf` を返す

→ ダウンロードしてテキスト抽出

## ステップ1: コンテンツタイプ検出

```bash
URL="$1"

if [[ "$URL" =~ youtube\.com/watch || "$URL" =~ youtu\.be/ || "$URL" =~ youtube\.com/shorts ]]; then
    CONTENT_TYPE="youtube"
elif [[ "$URL" =~ \.pdf$ ]]; then
    CONTENT_TYPE="pdf"
elif curl -sI "$URL" | grep -i "Content-Type: application/pdf" > /dev/null; then
    CONTENT_TYPE="pdf"
else
    CONTENT_TYPE="article"
fi

echo "検出結果: $CONTENT_TYPE"
```

## ステップ2: コンテンツ抽出

### YouTube動画の場合

```bash
if ! command -v yt-dlp &> /dev/null; then
    echo "yt-dlp が必要です"
    exit 1
fi

VIDEO_TITLE=$(yt-dlp --print "%(title)s" "$URL" | tr '/' '_' | tr ':' '-' | tr '?' '' | tr '"' '')
yt-dlp --write-auto-sub --skip-download --sub-langs en --output "temp_transcript" "$URL"

# VTTをプレーンテキストに変換（重複除去）
python3 -c "
import sys, re
seen = set()
with open('temp_transcript.en.vtt', 'r') as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith('WEBVTT') and not line.startswith('Kind:') and not line.startswith('Language:') and '-->' not in line:
            clean = re.sub('<[^>]*>', '', line)
            clean = clean.replace('&amp;', '&').replace('&gt;', '>').replace('&lt;', '<')
            if clean and clean not in seen:
                print(clean)
                seen.add(clean)
" > "${VIDEO_TITLE}.txt"
rm -f temp_transcript.en.vtt
CONTENT_FILE="${VIDEO_TITLE}.txt"
```

### 記事/ブログの場合

```bash
if command -v reader &> /dev/null; then
    reader "$URL" > temp_article.txt
    ARTICLE_TITLE=$(head -n 1 temp_article.txt | sed 's/^# //')
elif command -v trafilatura &> /dev/null; then
    METADATA=$(trafilatura --URL "$URL" --json)
    ARTICLE_TITLE=$(echo "$METADATA" | python3 -c "import json, sys; print(json.load(sys.stdin).get('title', 'Article'))")
    trafilatura --URL "$URL" --output-format txt --no-comments > temp_article.txt
else
    ARTICLE_TITLE=$(curl -s "$URL" | grep -oP '<title>\K[^<]+' | head -n 1)
    ARTICLE_TITLE=${ARTICLE_TITLE%% - *}
fi

FILENAME=$(echo "$ARTICLE_TITLE" | tr '/' '-' | tr ':' '-' | tr '?' '' | cut -c 1-80)
mv temp_article.txt "${FILENAME}.txt"
CONTENT_FILE="${FILENAME}.txt"
```

### PDFの場合

```bash
PDF_FILENAME=$(basename "$URL")
curl -L -o "$PDF_FILENAME" "$URL"

if command -v pdftotext &> /dev/null; then
    pdftotext "$PDF_FILENAME" "${PDF_FILENAME%.pdf}.txt"
    CONTENT_FILE="${PDF_FILENAME%.pdf}.txt"
else
    echo "pdftotext が見つかりません。poppler のインストールが必要です。"
    CONTENT_FILE="$PDF_FILENAME"
fi
```

## ステップ3: アクションプラン作成

抽出コンテンツから以下を生成:
- 実行可能な教訓の抽出（要約ではなく）
- 4-8週間のクエスト定義
- Rep 1（今週出荷可能なもの）の作成
- Rep 2-5（段階的なイテレーション）の設計
- `Ship-Learn-Next Plan - [クエストタイトル].md` として保存

## ステップ4: 結果表示

```
Tapestry ワークフロー完了！

コンテンツ抽出:
  [コンテンツタイプ]: [タイトル]
  保存先: [filename.txt]
  [X] 語抽出

アクションプラン作成:
  クエスト: [クエストタイトル]
  保存先: Ship-Learn-Next Plan - [タイトル].md

あなたのクエスト: [一行サマリー]
Rep 1（今週）: [Rep 1 の目標]

Rep 1 はいつ出荷しますか？
```

## エラー対応

| 問題 | 対応 |
|------|------|
| 未対応のURLタイプ | 記事抽出をフォールバックで試行 |
| コンテンツ抽出失敗 | URL到達性確認、別手法試行 |
| ツール未インストール | 可能なら自動インストール、不可なら案内表示 |
| 空コンテンツ | 抽出失敗時はプラン作成をスキップ |

## 依存関係

| 用途 | ツール |
|------|--------|
| YouTube | yt-dlp, Python 3 |
| 記事 | reader (npm) または trafilatura (pip)、フォールバック: curl |
| PDF | curl, pdftotext (poppler) |
| プラン作成 | 追加依存なし |

## 思想

**Tapestry は学習コンテンツをアクションに織り上げる。**

統合ワークフローにより、コンテンツを消費するだけで終わらず、必ず実装プランを作成する。受動的な学習を能動的な構築に変換する。

抽出 → プラン → 出荷 → 学習 → 次へ。