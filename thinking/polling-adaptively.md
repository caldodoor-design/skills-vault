---
name: polling-adaptively
description: バックグラウンドコマンドの適応型ポーリング（トークン節約）
---
# 適応型ポーリング

1回目: `WaitDurationSeconds: 30`
2回目: `WaitDurationSeconds: 60`
3回目以降: `WaitDurationSeconds: 120`

短いタスク（ビルド等）は30秒で検知、長いタスク（レビュー等）は不要なAPI呼び出しを削減。
