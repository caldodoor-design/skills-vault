---
name: creating-context-capsule
description: コンテキストリセット前にセッション状態を.claude/memory/に保存する手順を定義する。コンテキストが肥大化した時やセッション終了時に使う。
---

# Context Capsule 作成手順

## Purpose

コンテキストリセットやセッション終了前に、作業状態を `.claude/memory/` に保存して次回の継続を可能にする。

---

## When to Use

- コンテキストが肥大化してリセットが必要な時
- セッション終了時
- 長時間タスクの区切りで状態保存したい時
- ユーザーから明示的に要求された時

---

## Workflow

### Step 1: sessions.md を更新

**Location:** `.claude/memory/sessions.md`

以下を追記する：

```markdown
## YYYY-MM-DD セッションN

### 実施内容
- （何をしたか箇条書き）

### 変更ファイル
- path/to/file_a
- path/to/file_b

### 変更規模
- Nファイル変更、-X行 / +Y行

### 未完了・申し送り
- （次回に引き継ぐべきこと）
```

---

### Step 2: decisions.md を更新（方針決定があった場合のみ）

**Location:** `.claude/memory/decisions.md`

新たな方針・アーキテクチャ判断があれば追記する。

---

### Step 3: 完了メッセージ

```
Context Capsule 保存完了
- .claude/memory/sessions.md 更新済み
- .claude/memory/decisions.md 更新済み（該当があれば）
```

---

## Important Rules

1. sessions.md は追記のみ（過去の記録を編集しない）
2. decisions.md は方針変更があった場合のみ追記
3. 簡潔に書く（コンテキストウィンドウは配列 — 長いログは次回のトークンを食う）
