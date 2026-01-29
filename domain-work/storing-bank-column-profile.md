---
name: storing-bank-column-profile
description: 銀行ごとの列設定（columnMap）をbankId単位で保存・取得・削除し、人間の補正値を次回以降の解析に再利用する。MANUAL補正はAUTO推定より常に優先される。
trigger: 銀行別のcolumnMapを保存・取得・管理する処理を実装するとき、またはMANUAL/AUTO優先度ロジックに関わるとき。
---

# S24: BankColumnProfileStore

## 1. Purpose
銀行ごとの列設定（columnMap）を保存し、次回以降のページ解析を安定させる。
"人間の補正"が最終正解なので、補正値をbankIdスコープで再利用する。

## 2. Scope
- 対象：bankId単位のcolumnMap
- 保存先：IndexedDB（ローカル）または既存ストア方針に準拠
- 非対象：クラウド保存（セキュリティ優先）

## 3. Inputs / Outputs
### Inputs
- bankId: string
- columnMap
- meta: {updatedAt, source: "AUTO" | "MANUAL"}

### Outputs
- getProfile(bankId) -> columnMap | null
- upsertProfile(bankId, columnMap) -> ok
- deleteProfile(bankId) -> ok

## 4. Rules
1) bankIdスコープで混線させない
2) MANUALはAUTOより優先（人間が正）
3) 古いAUTOは上書きしてよいが、MANUALは勝手に消さない
4) columnMapの整合（0〜1、x0<x1）は保存前に検証

## 5. DoD
- save/load/delete が動作
- bankId違いで混線しない
- MANUAL優先が保証される

## 6. Test Cases
- TC1: bankId=Aに保存 → Aでロードできる
- TC2: bankId=Bは影響なし
- TC3: MANUAL保存 → AUTO推定より優先
- TC4: 不正columnMapは保存できない

## 7. Anti-Patterns
- bankId無視で混ざる
- MANUALをAUTOで上書きする
- 保存データが壊れて全ページ崩壊する

## 8. Notes
- "育つ"はここ（人間補正の蓄積）で実現する。
