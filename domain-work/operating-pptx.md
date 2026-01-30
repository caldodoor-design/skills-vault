---
name: operating-pptx
description: PowerPointプレゼンテーション(.pptx)の作成・編集・分析。テンプレート活用、デザイン原則、OOXML編集、サムネイル生成。
trigger: PowerPointプレゼンテーションの作成、編集、テンプレート操作が必要な時
---

# PPTX操作ガイド

> **元リポジトリ**: https://github.com/anthropics/skills
> **scripts依存**: html2pptx.js, thumbnail.py, rearrange.py, replace.py等は元リポジトリを参照

## テキスト抽出

```bash
python -m markitdown path-to-file.pptx
```

## Raw XMLアクセス

```bash
python ooxml/scripts/unpack.py <pptx_file> <output_dir>
```

### 主要ファイル構造
- `ppt/presentation.xml` - プレゼンテーションメタデータ
- `ppt/slides/slide{N}.xml` - 各スライド
- `ppt/notesSlides/notesSlide{N}.xml` - スピーカーノート
- `ppt/slideLayouts/` - レイアウトテンプレート
- `ppt/slideMasters/` - マスタースライド
- `ppt/theme/` - テーマ・スタイル情報
- `ppt/media/` - 画像・メディア

## 新規プレゼン作成（テンプレートなし）

html2pptxワークフロー: HTMLスライドをPowerPointに変換。

### デザイン原則

1. **主題に合うデザインを選択** - 内容・トーン・業界を考慮
2. **Webセーフフォントのみ使用**: Arial, Helvetica, Georgia, Verdana等
3. **明確な視覚階層**: サイズ、太さ、色で階層を表現
4. **コントラスト確保**: テキストの可読性を担保

### カラーパレット例（一部）
- Classic Blue: `#1C2833`, `#2E4053`, `#AAB7B8`, `#F4F6F6`
- Teal & Coral: `#5EA8A7`, `#277884`, `#FE4447`, `#FFFFFF`
- Black & Gold: `#BF9A4A`, `#000000`, `#F4F6F6`

### レイアウトルール
- チャート・テーブル: **2カラム**（推奨）またはフルスライド
- テキストの下にチャートを縦積み**禁止**

### ワークフロー
1. html2pptx.mdを読む
2. 各スライドのHTMLファイル作成（720pt × 405pt for 16:9）
3. html2pptx.jsでPowerPointに変換
4. サムネイルで視覚検証 → 問題あれば修正

## テンプレートを使った作成

1. テンプレートのテキスト抽出＋サムネイルグリッド作成
2. テンプレート分析 → インベントリファイル作成
3. コンテンツに合うレイアウト選択 → アウトライン作成
4. `rearrange.py` でスライド複製・並べ替え
5. `inventory.py` でテキスト抽出
6. 置換テキストJSON作成
7. `replace.py` で適用

## 既存プレゼンの編集

1. ooxml.mdを読む
2. 展開: `python ooxml/scripts/unpack.py <file> <dir>`
3. XMLファイル編集
4. バリデーション: `python ooxml/scripts/validate.py <dir> --original <file>`
5. 再パック: `python ooxml/scripts/pack.py <dir> <file>`

## サムネイルグリッド作成

```bash
python scripts/thumbnail.py template.pptx [output_prefix]
# デフォルト: 5列、最大30スライド/グリッド
# カスタム: --cols 4（3-6の範囲）
```

## 画像変換

```bash
soffice --headless --convert-to pdf template.pptx
pdftoppm -jpeg -r 150 template.pdf slide
```
