---
name: guarding-secrets
description: シークレットの安全な管理ルール（Varlock使用）
---
# シークレット保護

シークレットをターミナル出力・コンテキスト・ログ・git・エラーメッセージに露出させない。

## NG/OKパターン

```bash
# NG
echo $SECRET_KEY
cat .env
printenv | grep API
curl -H "Authorization: Bearer sk_live_xxx" ...

# OK
varlock load --quiet && echo "Secrets validated"
cat .env.schema          # スキーマ（値なし）は安全
varlock run -- <cmd>     # 環境変数付き実行
```

## .env.schema

値を含まないスキーマファイル。安全に読める。

```bash
# @defaultSensitive=true @defaultRequired=infer
# @type=enum(development,staging,production) @sensitive=false
NODE_ENV=development
# @type=url @required
DATABASE_URL=
# @type=string(startsWith=sk_) @required @sensitive
STRIPE_SECRET_KEY=
```

| アノテーション | 効果 |
|---------------|------|
| `@sensitive` | 全出力でマスク |
| `@sensitive=false` | ログに表示可 |
| `@defaultSensitive=true` | 全変数をデフォルト機密 |

## シークレット更新依頼時

Claudeはシークレットを直接変更しない。ユーザーに.env手動更新 or シークレットマネージャー使用を案内し、`varlock load` でバリデーション。
