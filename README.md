# Skills Vault

プロジェクト横断で共有するSkillsの保管庫。
各プロジェクトからgit submoduleとして参照する。

## 使い方

```bash
# プロジェクトにsubmoduleとして追加
git submodule add https://github.com/caldodoor-design/skills-vault.git .claude/skills

# 既存プロジェクトでsubmoduleを取得
git submodule update --init --recursive
```

## 構造

| ディレクトリ | 層 | 用途 |
|-------------|-----|------|
| `thinking/` | Foundational | 思考プロセスの固定 |
| `domain-guard/` | Partner | ツールの触り方の標準化 |
| `orchestration/` | Enterprise | ワークフロー・タスク管理 |
| `tdd-github-automation/` | Enterprise | TDD駆動のGitHub自動化 |

## Skills設計原則

- 1 Skill = 1フェーズ = 1ブロック
- SKILL.mdは500行以内
- 詳細はreference/*.mdに逃がす
- 命名規則: 動詞-ing＋ハイフン区切り
