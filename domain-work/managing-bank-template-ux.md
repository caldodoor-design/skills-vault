---
name: managing-bank-template-ux
description: ユーザーが銀行ごとのテンプレートを直感的に切り替え・適用結果を確認できるUIを提供するSkill。Bank ID入力による列テンプレートの即時反映UIで使用する。
trigger: 銀行テンプレート切り替えUIやBank ID入力による即時プレビュー機能を実装・修正するとき。BankTemplateUXやSplitViewのBank ID関連UIに関わるコード変更時。
---

# S9: BankTemplateUX

> Updated by **Agent Delta** based on `reports/sprint-003/verification_report.md`
> Validated: Sprint 3 Verification

## 1. Purpose
ユーザーが直感的に「銀行ごとのテンプレート」を切り替え、適用結果を確認できるUIを提供する。
設定の複雑さを隠し、「銀行名を入れるだけ」で最適化される体験を目指す。

## 2. Scope
- 対象コンポーネント: `SplitView` toolbar / header
- 対象操作: Bank ID Input, Apply Button

## 3. Inputs / Outputs
- Input: User types `bankId`
- Output: Instant UI update (table re-render with new columns)

## 4. Rules
1. **Instant Feedback**: Bank ID を変更したら（または適用ボタンを押したら）、即座にテーブルの列割当を再計算して表示する。
2. **Visual Guide**: 現在適用されている列の境界線（ガイドライン）を画像上にオーバーレイ表示すると尚良い（Optional but recommended）。

## 5. DoD (Definition of Done)
- ツールバーで Bank ID を入力できる。
- 変更するとデータの並びが変わる。

## 6. Test Cases
| Case | Scenario | Expected | Result |
|---|---|---|---|
| A | Change Bank ID | Table updates immediately | PASS |

## 7. Anti-Patterns
- IDを変えても再解析ボタンを押すまで何も変わらない（フィードバック遅延）。
