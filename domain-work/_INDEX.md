# Skills Index (Smart Passbook Engine)

このフォルダは「技能（Skill）の仕様」を資産として蓄積する場所。
実装は変わっても、Skillの契約（I/F・DoD・Anti-Patterns）はここで固定する。

---

## ✅ Core Skills（学習）
- S1 NormalizeClientText
- S2 RuleStore
- S3 PromptContextBuilder
- S4 RulesManagerUX

## ✅ Core Skills（整合・安全）
- S5 DirectionDecision
- S6 BalanceSelfHealing

## ✅ Sprint 3（銀行テンプレ）
- S7 BankProfileStore
- S8 ColumnTemplateApplier
- S9 BankTemplateUX

---

## ✅ 運用ルール（重要）
- Sprintの検証レポートは `.agent/reports/sprint-XXX/verification_report.md`
- Skill更新は **verification_report.md の事実**のみで行う（推測で仕様変更禁止）
- “壊れない”が最優先。迷ったら WARNING で止める。
