---
name: extracting-articles
description: URLからWeb記事のクリーンなテキストを抽出し、広告やナビゲーション等のノイズを除去して保存する。ユーザーが記事やブログのURL指定で内容抽出を求めた時に使用。
trigger: ユーザーが記事/ブログのURLを提示し、テキスト抽出・保存を依頼した時
---

# 記事コンテンツ抽出

Web記事やブログ投稿から本文を抽出し、ナビゲーション・広告・ニュースレター登録等のノイズを除去してクリーンなテキストとして保存する。

## 発動条件

- 記事/ブログのURLを提示してテキスト内容を求めた時
- 「この記事をダウンロードして」と言われた時
- 「URLからコンテンツを抽出して」と言われた時
- 「このブログ記事をテキストとして保存して」と言われた時

## 処理フロー

1. **ツールの存在確認**（reader または trafilatura）
2. **記事のダウンロードと抽出**（最適なツールを使用）
3. **コンテンツ整形**（余分な空白除去、フォーマット調整）
4. **ファイル保存**（記事タイトルをファイル名に使用）
5. **保存先の確認とプレビュー表示**

## ツール確認（優先順）

### 方法1: reader（推奨 - Mozilla Readability ベース）

```bash
command -v reader
```

### 方法2: trafilatura（Python ベース）

```bash
command -v trafilatura
```

### 方法3: フォールバック（curl + 簡易パース）

上記ツールが無い場合、基本的な curl + テキスト抽出で対応（精度は劣る）

> ツールのインストール手順は元リポジトリを参照:
> https://github.com/nicehash/tapestry (article-extractor)

## 抽出方法

### reader 使用時（大半の記事に最適）

```bash
reader "URL" > article.txt
```

- Mozilla Readability アルゴリズムベース
- ノイズ除去に優秀、記事構造を保持

### trafilatura 使用時（ブログ/ニュースに最適）

```bash
trafilatura --URL "URL" --output-format txt > article.txt
trafilatura --URL "URL" --output-format txt --no-comments --no-tables > article.txt
```

- 高精度抽出、多言語対応
- `--no-comments`: コメントセクションをスキップ
- `--no-tables`: データテーブルをスキップ
- `--precision`: 精度優先
- `--recall`: より多くのコンテンツを抽出

### フォールバック（curl + 簡易パース）

```bash
curl -s "URL" | python3 -c "
from html.parser import HTMLParser
import sys

class ArticleExtractor(HTMLParser):
    def __init__(self):
        super().__init__()
        self.in_content = False
        self.content = []
        self.skip_tags = {'script', 'style', 'nav', 'header', 'footer', 'aside'}
    def handle_starttag(self, tag, attrs):
        if tag not in self.skip_tags:
            if tag in {'p', 'article', 'main', 'h1', 'h2', 'h3'}:
                self.in_content = True
    def handle_data(self, data):
        if self.in_content and data.strip():
            self.content.append(data.strip())
    def get_content(self):
        return '\n\n'.join(self.content)

parser = ArticleExtractor()
parser.feed(sys.stdin.read())
print(parser.get_content())
" > article.txt
```

## タイトル取得

```bash
# reader の場合
TITLE=$(reader "URL" | head -n 1 | sed 's/^# //')

# trafilatura の場合
TITLE=$(trafilatura --URL "URL" --json | python3 -c "import json, sys; print(json.load(sys.stdin)['title'])")

# curl フォールバック
TITLE=$(curl -s "URL" | grep -oP '<title>\K[^<]+' | sed 's/ - .*//' | sed 's/ | .*//')
```

## ファイル名の生成

```bash
FILENAME=$(echo "$TITLE" | tr '/' '-' | tr ':' '-' | tr '?' '' | tr '"' '' | tr '<>' '' | tr '|' '-' | cut -c 1-80 | sed 's/ *$//')
FILENAME="${FILENAME}.txt"
```

## 完全ワークフロー

```bash
ARTICLE_URL="https://example.com/article"

# ツール選定
if command -v reader &> /dev/null; then
    TOOL="reader"
elif command -v trafilatura &> /dev/null; then
    TOOL="trafilatura"
else
    TOOL="fallback"
fi

# 抽出実行
case $TOOL in
    reader)
        reader "$ARTICLE_URL" > temp_article.txt
        TITLE=$(head -n 1 temp_article.txt | sed 's/^# //')
        ;;
    trafilatura)
        METADATA=$(trafilatura --URL "$ARTICLE_URL" --json)
        TITLE=$(echo "$METADATA" | python3 -c "import json, sys; print(json.load(sys.stdin).get('title', 'Article'))")
        trafilatura --URL "$ARTICLE_URL" --output-format txt --no-comments > temp_article.txt
        ;;
    fallback)
        TITLE=$(curl -s "$ARTICLE_URL" | grep -oP '<title>\K[^<]+' | head -n 1)
        TITLE=${TITLE%% - *}
        ;;
esac

# ファイル名整形と保存
FILENAME=$(echo "$TITLE" | tr '/' '-' | tr ':' '-' | tr '?' '' | tr '"' '' | cut -c 1-80)
mv temp_article.txt "${FILENAME}.txt"

echo "抽出完了: $TITLE"
echo "保存先: ${FILENAME}.txt"
head -n 10 "${FILENAME}.txt"
```

## エラー対応

| 問題 | 対応 |
|------|------|
| ツール未インストール | 別ツールにフォールバック。インストール案内を表示 |
| ペイウォール/ログイン必須 | 「認証が必要なため抽出不可」と通知 |
| 無効なURL | URL形式を確認、リダイレクト有無を試行 |
| コンテンツ抽出ゼロ | JSが重いサイトの可能性。フォールバック試行 |
| タイトルに特殊文字 | ファイルシステム互換にクリーニング |

## 出力内容

**保存されるもの**: 記事タイトル、著者（取得可能時）、本文、見出し

**除去されるもの**: ナビゲーション、広告、ニュースレター登録、サイドバー、コメント、SNSボタン、Cookie通知

## 抽出後の表示

1. 「抽出完了: [記事タイトル]」
2. 「保存先: [ファイル名]」
3. プレビュー（先頭10-15行）
4. ファイルサイズと保存場所