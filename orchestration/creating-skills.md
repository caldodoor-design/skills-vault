---
name: creating-skills
description: スキル作成の5段階プロセスとProgressive Disclosure設計
---
# スキル作成ガイド

## 核心原則
- **Claudeが持っていない情報だけ追加**。各トークンの価値を問う
- 自由度はタスクの脆弱性で調整（高: テキスト指示、中: 疑似コード、低: 特定スクリプト）

## 構成
```
skill-name/
├── SKILL.md（必須: YAMLフロントマター + Markdown本文）
└── バンドルリソース（任意）
    ├── scripts/     - 繰り返し必要なコード
    ├── references/  - 作業中に参照するドキュメント
    └── assets/      - テンプレート等
```

## Progressive Disclosure（3段階ローディング）
1. **メタデータ**（name + description）→ 常にコンテキスト内（~100語）
2. **SKILL.md本文** → トリガー時にロード（5,000語以下）
3. **バンドルリソース** → 必要時にClaude判断でロード

SKILL.md本文は**500行以下**。超過時は別ファイルに分割。

## フロントマター
- `name`: スキル名
- `description`: **トリガーの主要メカニズム**。何をするか＋いつ使うかを含める

## 作成プロセス
1. 具体例で使用パターンを把握
2. 再利用コンテンツを計画（scripts/references/assets）
3. テンプレート生成
4. 編集・実装（別のClaudeインスタンスが使うことを意識）
5. 実使用でイテレーション
