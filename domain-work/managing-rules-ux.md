---
name: managing-rules-ux
description: ユーザーが直感的に学習ルールを管理・整理できるUIを提供し、誤った学習データの混入を防ぐSkill。RulesManagerコンポーネントのフィルタリングや削除操作で使用する。
trigger: 学習ルール管理UIのフィルタリング・削除・表示ロジックを実装・修正するとき。RulesManager.tsxに関わるコード変更時。
---

# Skill: RulesManagerUX (S4) v1.1

## 1. Purpose
現場のユーザーが直感的に学習ルールを管理・整理でき、誤った学習データの混入や適用を防ぐ。

## 2. Scope
- `RulesManager.tsx` コンポーネント

## 3. Inputs / Outputs
- **Input**: User interactions (Filter change, Delete click)
- **Output**: API calls (`DELETE`, `GET`), UI updates

## 4. Rules
1. **Filtering**: `Default` / `Current Bank` / `ALL` のスコープ切り替えを提供する。
2. **List Item**: `All` 表示時は、どの銀行のルールかわかるように `bankId` 列を表示する（またはバッジ）。
3. **Deletion**: 削除ボタンは特定の `ID` に対して動作し、リストからその行だけを消す。
4. **Feedback**: 削除後、トースト通知等でフィードバックを行う。

## 5. DoD
- ユーザーが現在の銀行以外のルールを誤って削除しにくいUIになっていること。
- 意図した行だけが削除されること。
- フィルタリングが正しく機能し、表示件数が変わること。

## 6. Test Cases
- Select `Current Bank` -> Show only current bank rules.
- Click Delete on Item A -> Item A disappears, similar Item B (different ID/Bank) remains.

## 7. Anti-Patterns
- 削除確認なしで即削除する。
- フィルタリング機能がなく、全ルールが混在して表示される（ALLモードは明示的な選択のみ許可する）。

## 8. Notes
- Sprint 1 では基本的なCRUDとフィルタリングを実装する。
