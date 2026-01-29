# Project Skills Map

このディレクトリには、プロジェクトで実装される主要な「Skill（機能モジュール）」の仕様定義書 (`SKILL.md`) が格納されています。
実装に着手する前に、各Skillの定義を参照し、ルールと完了条件 (DoD) を確認してください。

## Sprint 1: Learning & Persistence (Core)

| Skill ID | Name | Description | Status |
|---|---|---|---|
| **S1** | [NormalizeClientText](./normalize-client-text/SKILL.md) | 表記ゆれ吸収（NFKC, 法人格除去） | Defined |
| **S2** | [RuleStore](./rule-store/SKILL.md) | ルール保存・管理（JSON/DB抽象化） | Defined |
| **S3** | [PromptContextBuilder](./prompt-context-builder/SKILL.md) | OCR用Few-shotプロンプト生成 | Defined |
| **S4** | [RulesManagerUX](./rules-manager-ux/SKILL.md) | ルール管理UI・フィルタ・削除 | Defined |

## Sprint 2: Logic & Automation (Intelligence)

| Skill ID | Name | Description | Status |
|---|---|---|---|
| **S5** | [DirectionDecision](./direction-decision/SKILL.md) | 入出金列の自動判定・反転 | Planned |
| **S6** | [BalanceSelfHealing](./balance-self-healing/SKILL.md) | 残高検算と自動補正 | Planned |

## 運用ルール
- 新しい複雑なロジックを追加する場合は、まずここにフォルダを作成し、`SKILL.md` を定義してから実装すること。
- `SKILL.md` は実装の「真実のソース」として扱い、仕様変更時はここを更新すること。
