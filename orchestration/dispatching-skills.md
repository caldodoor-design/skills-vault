---
name: dispatching-skills
description: タスクのキーワードから関連Skillsを逆引きするテーブル。Main AgentがSubAgentに実装委譲する前に参照し、適切なSkillを選定するために使用。
trigger: SubAgentに実装タスクを委譲する直前
---

# Skills ディスパッチテーブル

## Purpose

タスク内容から関連Skillsを素早く特定するための逆引きテーブル。
Main Agentは実装委譲前に必ずこのテーブルを参照し、該当Skillsをpromptに含める。

---

## キーワード → Skills マッピング

### Kindle関連

| キーワード | Skill | パス |
|-----------|-------|------|
| Kindle, キャプチャ, ページ取得, PDF化 | capturing-kindle-pages | domain-work/capturing-kindle-pages.md |
| Kindle, DOM, セレクタ, 要素取得 | analyzing-kindle-dom-heuristics | domain-work/analyzing-kindle-dom-heuristics.md |
| Kindle, セレクタ, 辞書 | reading-kindle-dom | domain-work/reading-kindle-dom.md |

### Chrome拡張関連

| キーワード | Skill | パス |
|-----------|-------|------|
| Chrome拡張, MV3, Service Worker | building-chrome-ext-mvp | domain-work/building-chrome-ext-mvp.md |
| メッセージング, Popup, Background, Content Script | handling-chrome-extension-messaging | domain-work/handling-chrome-extension-messaging.md |
| Debugger API, キーボード, dispatchKeyEvent | simulating-keyboard-via-debugger | domain-work/simulating-keyboard-via-debugger.md |
| スクリーンショット, リトライ, 失敗 | retrying-screenshot-with-escalation | domain-work/retrying-screenshot-with-escalation.md |
| ローディング, 待機, MutationObserver | detecting-ready-state | domain-work/detecting-ready-state.md |
| Content Script, 準備完了, PING-PONG | detecting-content-script-readiness | domain-work/detecting-content-script-readiness.md |

### 画像処理・メモリ管理

| キーワード | Skill | パス |
|-----------|-------|------|
| OffscreenCanvas, 画像処理, メモリ解放 | capturing-memory-heavy-assets | domain-work/capturing-memory-heavy-assets.md |
| PDF生成, jsPDF, クライアントサイド | orchestrating-client-side-pdf | orchestration/orchestrating-client-side-pdf.md |

### 状態管理

| キーワード | Skill | パス |
|-----------|-------|------|
| Mutex, 競合, chrome.storage | managing-state-mutation-with-mutex | domain-work/managing-state-mutation-with-mutex.md |
| タブ, tabId, 状態管理 | managing-tab-state | domain-work/managing-tab-state.md |
| セッションID, 非同期ループ, 陳腐化 | navigating-pages-with-session-id | domain-work/navigating-pages-with-session-id.md |

### レビュー・品質

| キーワード | Skill | パス |
|-----------|-------|------|
| Codex, レビュー, コード検証 | asking-codex-review | orchestration/asking-codex-review.md |
| タスク完了, 品質ゲート | completing-task-with-contract | orchestration/completing-task-with-contract.md |
| デバッグ, バグ, 根本原因 | debugging-systematically | thinking/debugging-systematically.md |

---

## 使い方

```
1. タスクからキーワードを抽出
2. このテーブルでマッチするSkillを探す
3. 見つかったら該当Skillを読み込む
4. SubAgentのpromptに含める
```

### 例

タスク: 「Kindleのキャプチャ機能でローディング中のスクショを弾きたい」

キーワード: Kindle, キャプチャ, ローディング

マッチするSkills:
- capturing-kindle-pages.md
- detecting-ready-state.md

→ この2つをSubAgentに渡す

---

## テーブル更新ルール

新しいSkillを作成したら、このテーブルにキーワードを追加すること。
登録しないと使われない。
