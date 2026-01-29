---
name: storing-rules
description: 学習ルールの保存・更新・削除ロジックを一元管理し、データストア移行を容易にするSkill。ルールのCRUD操作やBank Scope分離が必要な場面で使用する。
trigger: 学習ルールのCRUD操作（追加・更新・削除・取得）を実装または修正するとき。RuleStoreやLearningServiceのデータ操作に関わるコード変更時。
---

# Skill: RuleStore (S2) v1.1

## 1. Purpose
学習ルールの保存・更新・削除ロジックを1箇所に集約し、将来的なデータストア移行（JSON → Supabase等）を容易にする。

## 2. Scope
- `LearningService` クラスのデータ操作メソッド
- データソース (`learning_examples.json`)

## 3. Inputs / Outputs
- `getRules(bankId)`: `LearningRule[]`
- `addRule(rule, rawText?)`: `boolean` (success/fail)
- `deleteRule(id)`: `boolean` (success/fail)

## 4. Rules
1. **正規化キー**: ルールの重複判定には `clientNorm` (= `normalized(client)`) を使用する。
2. **Bank Scope**: `bankId` ごとにデータを分離・フィルタリングする。指定なしは `default` として扱う。
    - 特例: `bankId = 'ALL'` の場合はフィルタせず全データを返す（Debug/Admin用）。
3. **Upsert**: 同一 `clientNorm` が既存の場合は更新、なければ新規追加する。
4. **ID削除**: 削除は一意な `id` をキーに行う。

## 5. DoD
- 同一の `client` (正規化後) を追加した場合、既存レコードが更新され、重複登録されないこと。
- 異なる `bankId` であれば、同じ `client` でも別ルールとして保存されること。
- `deleteRule` で指定したIDのルールのみが削除され、同名の別ルールが消えないこと。

## 6. Test Cases
- Add `ABC` (Bank A) -> Add `ABC` (Bank A) -> Count should be 1.
- Add `ABC` (Bank A) -> Add `ABC` (Bank B) -> Count should be 2 (1 for A, 1 for B).
- Delete `ABC` (ID: 123) -> `ABC` (ID: 456) remains.

## 7. Anti-Patterns
- UIコンポーネントやAPIルートで直接 `fs` モジュールを使用してJSONを読み書きすること。
- `description` をキーにして重複判定を行うこと（以前のバグ）。

## 8. Notes
- 現在はローカルJSONファイルを使用しているが、この `Skill` の内部実装を差し替えるだけで DB 移行が完了するように設計する。
