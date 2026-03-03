---
name: managing-bank-template-ux
description: 銀行テンプレート切替えの即時反映UI
trigger: 銀行テンプレート切替UIやBank ID入力の即時プレビュー実装時
---

# S9: BankTemplateUX

銀行名を入れるだけで最適化される体験。

## ルール
1. **Instant Feedback**: Bank ID変更で即座にテーブル列割当を再計算・表示
2. **Visual Guide**: 列境界線を画像上にオーバーレイ表示（推奨）

## I/O
- Input: ユーザーがbankIdを入力
- Output: テーブル即時再描画（新しい列割り当て）

## 注意
- IDを変えても再解析ボタンを押すまで何も変わらないのはアンチパターン
