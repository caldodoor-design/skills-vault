---
name: storing-bank-profile
description: 銀行IDごとの列テンプレート（Column Profile）を永続化し、OCR揺らぎに依存しない安定した列推定を提供するSkill。銀行プロファイルの保存・取得やデフォルトフォールバックで使用する。
trigger: 銀行プロファイル（Column Ratios）の保存・取得・フォールバック処理を実装・修正するとき。BankProfileStoreやgetProfile/upsertProfileに関わるコード変更時。
---

# S7: BankProfileStore

> Updated by **Agent Delta** based on `reports/sprint-003/verification_report.md` & `reports/sprint-004/verification_report.md`
> Validated: Sprint 3 & 4 Verification

## 1. Purpose
銀行ID（bankId）ごとの列テンプレート（Column Profile）を保存し、次回以降の列推定を安定化させる。
OCRの揺らぎに依存せず、人間が定義した「正しい比率」を適用するための基盤。

## 2. Scope
- 対象：銀行プロファイル（Column Ratios）の永続化と取得
- 保存先：LocalStorage / IndexedDB (MVP)
- フォールバック：未登録銀行に対する Default Profile の提供

## 3. Inputs / Outputs
### Inputs
- bankId: string
- profile: BankProfile (Optional for upsert)

### Outputs
- getProfile(bankId) -> BankProfile
- upsertProfile(bankId, profile) -> void

## 4. Rules
1. **Default Fallback**: 指定された `bankId` が存在しない場合、必ずシステムの `Default Profile` を返す（クラッシュ防止）。
2. **Ratio Normalization**: 保存時および取得時に、Column Ratios の合計が `1.0` になるように自動正規化する。
   - 例: `0.1, 0.1, 0.1` -> `0.33, 0.33, 0.33`
3. **Isolation**: 異なる `bankId` のプロファイルは完全に分離して保存する。
4. **Validation (S10)**: 比率に `NaN` や負数、ゼロが含まれる場合、保存せずに `Default Profile` にフォールバックする（破損防止）。

## 5. DoD (Definition of Done)
- 未登録の `bankId` を入力してもエラーにならず、デフォルト設定で動作する。
- 比率設定が保存され、リロード後も維持される。

## 6. Test Cases
| Case | Senario | Expected | Result |
|---|---|---|---|
| A | Unregistered Bank ID | Return Default Profile | PASS |
| B | Ratio Sum != 1.0 | Auto Normalize | PASS |

## 7. Anti-Patterns
- プロファイルが見つからない場合に `null` を返してUIをクラッシュさせる。
- 合計が `1.0` を超える比率をそのまま適用して、列がはみ出す。
