---
name: completing-task-with-contract
description: タスク完了前に品質ゲート（型チェック・ビルド・Codexレビュー）を通過させ、未検証のまま完了させないための手順を定義する。タスクを完了としてマークする直前に使う。
skills: [asking-codex-review]
---

# 品質ゲート付きタスク完了

## Purpose

タスク完了前に品質チェックを必ず通すことで、未検証コードのマージを防ぐ。

---

## When to Use

- タスクが完了したと判断した時
- コード変更を伴うタスクの完了報告前

---

## Workflow

### Step 1: タスク種別を判定

| 種別 | 対象 | ゲートレベル |
|------|------|-------------|
| **Type A** | ドキュメント・設計・分析 | Lightweight |
| **Type B** | コード実装 | Heavyweight |

---

### Step 2: 品質ゲート実行

#### Lightweight Gate (Type A)

- [ ] 変更ファイルに TODO/WIP マーカーが残っていないか確認
- [ ] ドキュメントの内容が正確か確認

#### Heavyweight Gate (Type B)

```bash
# 1. 型チェック
npm run type-check

# 2. ビルド
npm run build

# 3. Codexレビュー
codex exec --skip-git-repo-check --dangerously-bypass-approvals-and-sandbox -C "." "レビュープロンプト"
```

---

### Step 3: 結果判定

#### ❌ NG（いずれかのチェックが失敗）

1. タスクを完了にしない
2. 問題を修正
3. Step 2 を再実行

#### ✅ OK（全チェック通過）

1. 完了報告を作成
2. TaskUpdate でステータスを completed に更新

---

## 完了報告フォーマット

```text
## 完了報告

### 実施内容
（何をしたか簡潔に）

### 変更ファイル
- path/to/file_a
- path/to/file_b

### 検証結果
（実行したコマンドと結果）

### Codexレビュー結果
- Status: OK / Issues Found
- Severity: （該当する場合）
- 対応: （修正した場合はその内容）
```
