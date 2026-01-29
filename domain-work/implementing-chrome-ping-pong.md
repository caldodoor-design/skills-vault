---
name: implementing-chrome-ping-pong
description: MV3 Service Workerの早期終了を防ぐPing-Pong方式（Signal Echo）のアーキテクチャと実装パターンを定義する。バックグラウンドで長時間実行が必要な拡張機能の実装時に使う。
skills: [handling-chrome-extension-messaging, managing-tab-state]
---

# Chrome Extension Ping-Pong Architecture (Signal Echo)

ブラウザ拡張機能において、Service Worker (Background) の早期終了を防ぎ、バックグラウンドでの安定した長時間実行を実現するためのアーキテクチャ。

## 課題

MV3 (Manifest V3) の Service Worker は、アイドル状態が続くと数分（最短30秒程度）でブラウザによって終了させられます。`setTimeout` によるループは「アイドル状態」とみなされ、ポップアップを閉じると停止するリスクが高いです。

## 解決策: Ping-Pong 方式

Service Worker と Content Script の間でメッセージを投げ合うことで、Service Worker の「アクティビティ」を継続させ、終了タイマーをリセットし続けます。

### フロー

1. **Ping**: Service Worker が Content Script に `TRIGGER_COMMAND` を送信。
2. **Action**: Content Script がページ上で処理（例：DOM操作、ページ内遷移）を実行。
3. **Echo (Pong)**: 処理完了後、Content Script が Service Worker に `ACTION_COMPLETED` を送信。
4. **Loop**: Service Worker がこのメッセージを受信したことで「起機」し、次のステップへ。

### メリット

- `chrome.alarms` の最短1分という制約を回避し、高速なループ（秒単位）が可能。
- ユーザー操作（ポップアップ）から独立し、タブが開いている限り継続可能。
- 外部API呼び出しやキャプチャなどの重量級処理を「Echo受信タイミング」で行うことで、リソース管理が容易。

## 実装例 (messaging.ts)

```typescript
type MessageType = "TRIGGER_ACTION" | "ACTION_DONE";
```

## 実装例 (Background)

```typescript
chrome.runtime.onMessage.addListener((message) => {
  if (message.type === "ACTION_DONE" && state.active) {
    // キャプチャ等の実処理を実行
    doHeavyWork();
    // 次のPingを送る
    sendPingToContentScript();
  }
});
```
