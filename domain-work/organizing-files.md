---
name: organizing-files
description: ファイル・フォルダの整理自動化。重複検出、構造提案、コンテキストに基づくスマートな分類・リネーム・クリーンアップ。
trigger: ファイル整理、フォルダ構造の改善、重複検出、ダウンロードフォルダの整理が必要な時
---

# ファイル整理スキル

ファイル・フォルダを論理的に整理し、デジタルワークスペースを整頓するアシスタント。

> **元リポジトリ**: https://github.com/ComposioHQ/awesome-claude-skills

## 手順

### 1. スコープの理解

確認事項:
- 対象ディレクトリ（Downloads、Documents、ホームフォルダ全体？）
- 主な問題（ファイルが見つからない、重複、構造がない？）
- 避けるべきファイル・フォルダ（進行中プロジェクト、機密データ？）
- 整理の積極度（保守的 vs 包括的？）

### 2. 現状分析

```bash
ls -la [target_directory]
du -sh [target_directory]/* | sort -rh | head -20
find [target_directory] -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn
```

サマリ: 総ファイル数、ファイル種別内訳、サイズ分布、日付範囲

### 3. 整理パターンの特定

**種類別**: Documents, Images, Videos, Archives, Code, Spreadsheets
**目的別**: 仕事 vs 個人, アクティブ vs アーカイブ, プロジェクト別
**日付別**: 年/月, 四半期

### 4. 重複検出

```bash
# ハッシュで完全一致を検出
find [directory] -type f -exec md5 {} \; | sort | uniq -d

# 同名ファイル検出
find [directory] -type f -printf '%f\n' | sort | uniq -d
```

各重複セット:
- 全ファイルパスを表示
- サイズ・更新日を表示
- 保持推奨を提示
- **削除前に必ず確認**

### 5. 整理計画の提示

変更前に計画を提示:
- 現状サマリ
- 提案するフォルダ構造
- 実行する変更の一覧
- 判断が必要なファイルのリスト
- ユーザー承認を得てから実行

### 6. 実行

```bash
mkdir -p "path/to/new/folders"
mv "old/path/file.pdf" "new/path/file.pdf"
```

**ルール**:
- 削除前に必ず確認
- 全移動をログに記録
- 元の更新日を保持
- ファイル名衝突を適切に処理
- 予期しない状況では停止して確認

### 7. 完了サマリ

- 作成フォルダ数、整理ファイル数、削減容量
- 新しいフォルダツリー表示
- メンテナンス推奨（週次: 新規ダウンロード整理、月次: プロジェクトアーカイブ）

## ファイル命名ベストプラクティス

- 日付を含める: `2024-10-17-meeting-notes.md`
- 具体的に: `q3-financial-report.xlsx`
- スペースを避ける（ハイフンかアンダースコア）
- ダウンロードの重複番号を除去: `document-final-v2 (1).pdf` → `document.pdf`

## アーカイブの判断基準

- 6ヶ月以上未使用のプロジェクト
- 完了済みの参照可能な作業
- 削除を躊躇するファイル（まずアーカイブ）
