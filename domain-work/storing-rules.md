---
name: storing-rules
description: 学習ルールのCRUD一元管理
trigger: 学習ルールのCRUD操作・データストア実装・修正時
---

# S2: RuleStore

学習ルールの保存・更新・削除を1箇所に集約。

## API
- `getRules(bankId)` → LearningRule[]
- `addRule(rule, rawText?)` → boolean
- `deleteRule(id)` → boolean

## ルール
1. 重複判定はclientNorm（normalize後）を使用
2. bankIdごとにデータ分離（指定なし→"default"）
3. bankId="ALL"はフィルタせず全データ返す（Debug/Admin用）
4. 同一clientNormはUpsert（更新）、なければ新規追加
5. 削除は一意なidをキーに実行

## 注意
- descriptionをキーにした重複判定は禁止（旧バグ）
- UIやAPIルートで直接fsを使わない（ストア経由）
- 現在はローカルJSON、将来DB移行時はストア内部だけ差替え
