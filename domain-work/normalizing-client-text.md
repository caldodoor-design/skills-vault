---
name: normalizing-client-text
description: 表記ゆれ（前株/後株、半角カナ、全角英数、空白）を吸収し、学習データのキーマッチング率を向上させる正規化Skill。OCR結果の取引先名を統一キーに変換する際に使用する。
trigger: 取引先名の正規化・名寄せ処理を実装または修正するとき。NormalizeClientTextやclientNormに関わるコード変更時。
---

# S1: NormalizeClientText

> Updated by **Agent Delta** based on `reports/sprint-1/verification_report.md`
> Validated: Sprint 1 Verification

## 1. Purpose
表記ゆれ（前株/後株、半角カナ、全角英数、空白の有無）を吸収し、学習データのキーマッチング率を向上させる。

## 2. Scope
- `LearningService.normalizeClientText` メソッド
- 学習データの `client` 保存時および照合時
- OCR結果の `client` 照合時

## 3. Inputs / Outputs
- **Input**: `text` (string) - 通帳OCRの生の取引先名
- **Output**: `normalizedText` (string) - 正規化された同一視用キー

## 4. Rules
1. **NFKC正規化**: 全角英数→半角、半角カナ→全角、記号分解（㈱→(株)）を行う。
2. **法人格除去**: `株式会社`, `有限会社`, `合同会社`, `(株)`, `(有)` などの法人格文字列を一律削除する。
3. **空白正規化**: 連続する空白は半角スペース1つに置換し、前後の空白は削除する（Trim）。
4. **英字小文字化**: 大文字小文字の差異を無視して正規化する（`.toLowerCase()`）。

## 5. DoD (Definition of Done)
以下のテストケースが全て意図通りに動作すること。

## 6. Test Cases
| Input | Output | 判定 |
|---|---|---|
| `（株）ABC` | `abc` | 同一 |
| `ABC(株)` | `abc` | 同一 |
| `株式会社ABC` | `abc` | 同一 |
| `ABC` | `abc` | 同一 |
| `ｼｬｶｲﾎｹﾝﾘｮｳ` | `シャカイホケンリョウ` | 同一 |
| `シャカイホケンリョウ` | `シャカイホケンリョウ` | 同一 |
| `ABC 12月分` | `abc 12月分` | **別** (数字・月度は維持) |
| `ABC 11月分` | `abc 11月分` | **別** |
| `三井住友` | `三井住友` | 同一 |
| `三井　住友` | `三井 住友` | 同一 |
| `固定資産税` | `固定資産税` | PASS (Sprint 1 Verified) |

### Added v1.1 (Verified in Sprint 1)
| Input | Output | Result |
|---|---|---|
| `固定資産税` | `固定資産税` | PASS (No Change) |
| `ｼｬｶｲﾎｹﾝﾘｮｳ` | `シャカイホケンリョウ` | PASS (Kana Norm) |
| `（株）ＡＢＣ` | `abc` | PASS (Corp removal + Lowercase) |

## 4. Known Issues
- コンソール出力の文字化け。
- UI上の「学習影響件数」の表示は今回は未実装。

## 5. Conclusion
Sprint 1 Goal 達成。Sprint 2 (Logic & Automation) へ移行可能。

## 7. Anti-Patterns
- **過剰な削除**: `12月` や `No.1` などの識別子まで削除してしまい、異なる明細を同一視させてしまうこと。
- **法人格の誤爆**: `株式会社` という名前の個人商店など（レアケースだが、原則は削除でOK）。

## 8. Notes
- 空白処理について: 当初は「全削除」案もあったが、`ABC 12` と `ABC 1` のようなケースで `ABC12` `ABC1` となり、区切りが消えるリスクがあるため、`Verify` フェーズでは「連続空白→1つ」を採用している。
