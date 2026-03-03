---
name: completing-task-with-contract
description: タスク完了前の品質ゲート（型チェック・ビルド）手順
skills: [asking-codex-review]
---
# 品質ゲート付きタスク完了

## タスク種別
| 種別 | 対象 | ゲート |
|------|------|--------|
| Type A | ドキュメント・設計 | Lightweight |
| Type B | コード実装 | Heavyweight |

## Lightweight Gate (Type A)
- TODO/WIPマーカーが残っていないか確認

## Heavyweight Gate (Type B)
```bash
npm run type-check
npm run build
```

## 結果判定
- **NG**: タスク完了にしない → 修正 → 再実行
- **OK**: 完了報告作成 → TaskUpdateでcompleted

## 完了報告フォーマット
```
### 実施内容
### 変更ファイル
### 検証結果（実行コマンドと結果）
```
