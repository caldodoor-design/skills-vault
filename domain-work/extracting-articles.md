---
name: extracting-articles
description: URLからWeb記事のクリーンテキスト抽出。reader→trafilatura→curlフォールバックの順に試行。
trigger: 記事/ブログURLのテキスト抽出・保存依頼時
---
# 記事コンテンツ抽出

## ツール優先順
1. **reader** (npm): Mozilla Readabilityベース、ノイズ除去に優秀
2. **trafilatura** (pip): 高精度抽出、`--no-comments --no-tables` オプション
3. **curl + 簡易パース**: `<p>`, `<article>`, `<main>` タグからテキスト抽出

## ワークフロー
```bash
# ツール選定
if command -v reader &>/dev/null; then TOOL="reader"
elif command -v trafilatura &>/dev/null; then TOOL="trafilatura"
else TOOL="fallback"; fi

# 抽出
case $TOOL in
  reader) reader "$URL" > temp.txt; TITLE=$(head -n1 temp.txt | sed 's/^# //') ;;
  trafilatura)
    TITLE=$(trafilatura --URL "$URL" --json | python3 -c "import json,sys;print(json.load(sys.stdin).get('title','Article'))")
    trafilatura --URL "$URL" --output-format txt --no-comments > temp.txt ;;
  fallback) TITLE=$(curl -s "$URL" | grep -oP '<title>\K[^<]+' | head -n1) ;;
esac

# ファイル名整形
FILENAME=$(echo "$TITLE" | tr '/:?"|<>' '-' | cut -c1-80)
mv temp.txt "${FILENAME}.txt"
```

## エラー対応
| 問題 | 対応 |
|------|------|
| ツール未インストール | 別ツールにフォールバック |
| ペイウォール | 「認証が必要」と通知 |
| 抽出ゼロ | JS重いサイトの可能性、フォールバック試行 |
