---
name: managing-workflow
description: 計画確認→実装→検証→レビュー→報告の必須ワークフローと報告フォーマットを定義する。全てのコード変更タスクで常に従うべきフローとして使う。
skills: [asking-codex-review, completing-task-with-contract]
---

# 必須ワークフロー (Strict Flow)

Claude Codeが従うべき作業フロー。

---

## 基本フロー

1. **計画確認**: 指示された計画・DoDを確認
2. **実装**: コード修正を実行
3. **検証**: ビルド・テストを実行
4. **レビュー**: Codexでレビュー実施
5. **報告**: レビュー結果を含めて報告

---

## 堅牢なワークフロー (Decoupled)

フリーズ回避のため、以下は**別々のステップ**として分割実行：

1. **Step 1: 実装 (Modify)** - コード修正のみ
2. **Step 2: ビルド (Build)** - `npm run build` 等を単独実行
3. **Step 3: レビュー (Review)** - `codex exec` でレビュー

「修正してビルドしてレビューして」という一括指示は避ける。

---

## 報告フォーマット

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

### DoD達成状況
- [x] 条件1
- [x] 条件2

### 備考
（注意点や申し送り事項）
```
