---
name: resolving-bank-scoped-rules
description: 学習ルール（clientNorm -> description）を銀行スコープ（bankId）で安全に解決し、適用優先順位（bank → default → なし）を固定して誤混線を防ぐ。
trigger: 学習ルールの解決（resolve）や適用順序の管理を実装・修正するとき
---

# S13: BankScopedRuleResolver

> Updated by **Agent Delta** based on `reports/sprint-005/verification_report.md`
> Validated: Sprint 5 Verification

## 1. Purpose
学習ルール（clientNorm -> description）を「銀行スコープ（bankId）」で安全に解決し、誤混線を防ぐ。
適用優先順位を固定し、Defaultルールも活かしつつ誤爆を避ける。

## 2. Scope
- 対象：学習ルールの解決（resolve）と適用順序の管理
- 対象外：ルールの保存そのもの（S2 RuleStore側）
- 対象外：曖昧一致（部分一致・正規表現）は MVPでは採用しない

## 3. Inputs / Outputs
### Inputs
- bankId: string（例: "default" / "sumitomo" / "yucho"）
- clientText: string（通帳記載文字）
- rules: LearningRule[]（S2 RuleStoreの取得結果）

### Outputs
- resolved: { description: string | null, matchedRuleId?: string, matchedScope?: "BANK" | "DEFAULT" | "NONE" }
- confidence: number（0.0〜1.0、MVPは固定でも可）
- reason: string（適用理由のログ用）

## 4. Rules
1) ルール適用順序は固定：
   - bankId一致ルール → defaultルール → なし
2) 一致判定は clientNorm の完全一致（normalize後）
3) "ALL" スコープは閲覧専用。適用系ロジックには使わない
4) ルールが複数ヒットする場合（基本起きない設計）：
   - 最新timestamp優先（またはID優先）を明示して固定
5) 該当なしの場合は description=null を返す

## 5. DoD
- [ ] bankIdが違うルールが適用されない
- [ ] defaultルールは未登録銀行でも効く
- [ ] "ALL" 表示で混在しても適用処理に混ざらない
- [ ] resolveの結果に matchedScope と reason が付く

## 6. Test Cases
- Case1: bankId="A" にのみ存在するルールは bankId="B" では適用されない
- Case2: defaultルールのみ存在 → bankId="A" でも適用される（matchedScope=DEFAULT）
- Case3: bankルールとdefault両方存在 → bankルールが優先（matchedScope=BANK）
- Case4: 該当なし → description=null（matchedScope=NONE）

## 7. Anti-Patterns
- 部分一致で適用して誤爆（例："振込" だけで全部を同一視）
- bankId無視して全ルールからヒットさせる
- "ALL" 表示用の一覧をそのまま適用に流用する

## 8. Notes
- MVPでは resolve を純関数（関数単体）で実装しやすい構造にする
- 将来：regexルールを入れる場合は Guardrails（S16）とセットで導入する
