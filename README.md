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

| ディレクトリ | 層 | ファイル数 | 用途 |
|-------------|-----|-----------|------|
| `thinking/` | Foundational | 9 | 思考プロセスの固定（デバッグ、ブレインストーミング、完了前検証等） |
| `domain-guard/` | Partner | 5 | ツールの触り方の標準化（シークレット保護、Webアプリテスト等） |
| `domain-work/` | Enterprise | 60 | 実装テクニック（ドキュメント操作、ファイル整理、テスト修正等） |
| `orchestration/` | Enterprise | 7 | ワークフロー・タスク管理（スキル作成等） |
| `tdd-github-automation/` | Enterprise | 1 | TDD駆動のGitHub自動化 |

## Skills設計原則

- 1 Skill = 1フェーズ = 1ブロック
- SKILL.mdは500行以内
- 詳細はreference/*.mdに逃がす
- 命名規則: 動詞-ing＋ハイフン区切り
