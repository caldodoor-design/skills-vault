---
name: guarding-secrets
description: 環境変数・シークレットの安全な管理ルール。.envの直接読み取り禁止、マスク出力、スキーマ定義によるバリデーション。
trigger: シークレット、APIキー、.env、環境変数、認証情報を扱う時
---

# シークレット保護スキル

環境変数に含まれるシークレットを安全に管理するためのルールと手順。

> **元リポジトリ**: https://github.com/wrsmith108/varlock-claude-skill
> **Varlockドキュメント**: https://varlock.dev

## 基本原則: シークレットは絶対に露出させない

シークレットが以下に表示されてはならない:
- ターミナル出力
- Claudeの入出力コンテキスト
- ログファイルやトレース
- gitコミットやdiff
- エラーメッセージ

---

## セキュリティルール

### ルール1: シークレットをechoしない

```bash
# NG - シークレットがコンテキストに露出
echo $CLERK_SECRET_KEY
cat .env | grep SECRET
printenv | grep API

# OK - 値を表示せずバリデーション
varlock load --quiet && echo "Secrets validated"
```

### ルール2: .envを直接読まない

```bash
# NG - 全シークレットが露出
cat .env
less .env
# Readツールで.envファイルを開く

# OK - スキーマ（値なし）を読む
cat .env.schema
varlock load  # マスク値を表示
```

### ルール3: バリデーションにはVarlockを使う

```bash
# NG - エラーメッセージにシークレットが含まれる
test -n "$API_KEY" && echo "Key: $API_KEY"

# OK - Varlockがマスク表示
varlock load
# 出力: API_KEY sensitive ... masked
```

### ルール4: コマンドにシークレットを含めない

```bash
# NG - コマンド履歴にシークレットが残る
curl -H "Authorization: Bearer sk_live_xxx" https://api.example.com

# OK - 環境変数を使用
curl -H "Authorization: Bearer $API_KEY" https://api.example.com
```

---

## スキーマファイル: .env.schema

スキーマは各変数の型・バリデーション・機密性を定義する。値は含まないため安全に読める。

```bash
# グローバルデフォルト
# @defaultSensitive=true @defaultRequired=infer

# アプリケーション設定（非機密）
# @type=enum(development,staging,production) @sensitive=false
NODE_ENV=development

# @type=port @sensitive=false
PORT=3000

# データベース（機密）
# @type=url @required
DATABASE_URL=

# APIキー（機密）
# @type=string(startsWith=sk_) @required @sensitive
STRIPE_SECRET_KEY=
```

### セキュリティアノテーション

| アノテーション | 効果 | 用途 |
|---------------|------|------|
| `@sensitive` | 全出力でマスク | APIキー、パスワード、トークン |
| `@sensitive=false` | ログに表示可 | 公開キー、非機密設定 |
| `@defaultSensitive=true` | 全変数をデフォルト機密 | 高セキュリティプロジェクト |

---

## 安全なコマンド一覧

| タスク | 安全なコマンド |
|--------|---------------|
| 全変数バリデーション | `varlock load` |
| サイレントバリデーション | `varlock load --quiet` |
| 環境変数付き実行 | `varlock run -- <cmd>` |
| スキーマ確認 | `cat .env.schema` |
| 特定変数チェック | `varlock load \| grep VAR_NAME` |

| 禁止コマンド | 理由 |
|-------------|------|
| `cat .env` | 全シークレットが露出 |
| `echo $SECRET` | Claudeコンテキストに露出 |
| `printenv \| grep` | マッチしたシークレットが露出 |
| Readツールで.envを開く | シークレットがコンテキストに入る |

---

## よくあるシナリオの対応

### 「APIキーが設定されているか確認して」

```bash
# OK
varlock load 2>&1 | grep "API_KEY"
# NG
echo $API_KEY
```

### 「シークレットを更新して」

Claudeはシークレットを直接変更しない。ユーザーに以下を案内:
1. .envファイルを手動で更新
2. またはシークレットマネージャー（1Password等）で更新
3. `varlock load` でバリデーション

### 「.envファイルを見せて」

.envファイルは直接読まない。代替手段を案内:
- `varlock load` でマスク値を確認
- `cat .env.schema` でスキーマを確認

---

## 新規プロジェクトチェックリスト

- [ ] `.env.schema` を作成し全変数を定義
- [ ] 全シークレットに `@sensitive` を付与
- [ ] スキーマヘッダーに `@defaultSensitive=true` を追加
- [ ] `.env` を `.gitignore` に追加
- [ ] `.env.schema` をバージョン管理にコミット
- [ ] CI/CDにバリデーションを追加
- [ ] Claudeセッションで `cat .env` や `echo $SECRET` を絶対に使わない
