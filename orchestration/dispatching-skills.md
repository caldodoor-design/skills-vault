---
name: dispatching-skills
description: キーワードからSkillを逆引きするディスパッチテーブル
---
# Skills ディスパッチテーブル

タスクからキーワード抽出 → テーブルでマッチ → 該当SkillをSubAgentに渡す。

## Kindle関連
| キーワード | Skill |
|-----------|-------|
| Kindle, キャプチャ, PDF化 | capturing-kindle-pages |
| Kindle, DOM, セレクタ | analyzing-kindle-dom-heuristics |
| Kindle, セレクタ辞書 | reading-kindle-dom |

## Chrome拡張関連
| キーワード | Skill |
|-----------|-------|
| Chrome拡張, MV3, Service Worker | building-chrome-ext-mvp |
| メッセージング, Popup/Background/Content | handling-chrome-extension-messaging |
| Debugger API, dispatchKeyEvent | simulating-keyboard-via-debugger |
| スクリーンショット, リトライ | retrying-screenshot-with-escalation |
| ローディング, MutationObserver | detecting-ready-state |
| Content Script, PING-PONG | detecting-content-script-readiness |

## 画像・メモリ・PDF
| キーワード | Skill |
|-----------|-------|
| OffscreenCanvas, メモリ解放 | capturing-memory-heavy-assets |
| PDF生成, jsPDF | orchestrating-client-side-pdf |

## 状態管理
| キーワード | Skill |
|-----------|-------|
| Mutex, chrome.storage競合 | managing-state-mutation-with-mutex |
| タブ, tabId | managing-tab-state |
| セッションID, 非同期ループ | navigating-pages-with-session-id |

## レビュー・品質
| キーワード | Skill |
|-----------|-------|
| Codex, レビュー | asking-codex-review |
| タスク完了, 品質ゲート | completing-task-with-contract |
| デバッグ, 根本原因 | debugging-systematically |

新しいSkill作成時はこのテーブルにも追加すること。
