---
name: operating-pptx
description: PowerPoint(.pptx)の作成・編集・分析。テンプレート活用、OOXML編集、サムネイル生成。
---
# PPTX操作ガイド

## テキスト抽出
```bash
python -m markitdown path-to-file.pptx
```

## Raw XML
```bash
python ooxml/scripts/unpack.py <file> <dir>  # 展開
python ooxml/scripts/pack.py <dir> <file>    # 再パック
```
主要: `ppt/slides/slide{N}.xml`, `ppt/slideMasters/`, `ppt/theme/`

## 新規作成（html2pptx）
1. HTMLスライド作成（720pt × 405pt for 16:9）
2. html2pptx.jsで変換
3. サムネイルで検証

**デザイン原則**: Webセーフフォントのみ、明確な視覚階層、チャートは2カラム推奨

## テンプレート使用
1. テキスト抽出+サムネイル→分析→インベントリ
2. レイアウト選択→`rearrange.py`で複製・並替
3. `replace.py`でテキスト置換

## 画像変換
```bash
soffice --headless --convert-to pdf template.pptx
pdftoppm -jpeg -r 150 template.pdf slide
```
