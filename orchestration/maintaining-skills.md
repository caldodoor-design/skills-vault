---
name: maintaining-skills
description: Skillsの品質チェック・肥大化検知・重複検出
---
# Skillsメンテナンス

## チェック項目
- 500行超のSkill → reference/*.mdに分割
- 重複内容 → 統合検討
- Claudeが既に知っている内容 → 削除
- descriptionは「何をするか」＋「いつ使うか」が明確か
- 1 Skill = 1フェーズ = 1ブロックか

## 手順
1. `.claude/skills/` 配下の全ファイル一覧取得
2. 行数チェック（500行超過検知）
3. フロントマターのdescription品質確認
4. 重複目視チェック
5. 問題あればAskUserQuestionで報告
