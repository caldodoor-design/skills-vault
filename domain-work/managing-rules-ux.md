---
name: managing-rules-ux
description: 学習ルール管理・整理UI
trigger: RulesManagerのフィルタリング・削除・表示ロジック実装時
---

# S4: RulesManagerUX

学習ルールを直感的に管理・整理するUI。

## ルール
1. **Filtering**: Default / Current Bank / ALL のスコープ切り替え
2. ALL表示時はbankId列/バッジで識別可能にする
3. **Deletion**: 特定IDに対して動作、その行だけを消す
4. 削除後はトースト通知でフィードバック

## 注意
- 削除確認なしで即削除は禁止
- ALLモードは明示的な選択のみ許可
