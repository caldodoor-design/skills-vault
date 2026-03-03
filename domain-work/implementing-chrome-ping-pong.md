---
name: implementing-chrome-ping-pong
description: MV3 Service Worker早期終了を防ぐPing-Pong方式
trigger: バックグラウンド長時間実行が必要な拡張機能の実装時
---

# Chrome Extension Ping-Pong (Signal Echo)

MV3 Service Workerの早期終了を防ぐ。Service WorkerとContent Script間でメッセージを投げ合い、アクティビティを継続。

## フロー
1. **Ping**: SW → Content Script に `TRIGGER_COMMAND` 送信
2. **Action**: CS がページ上で処理実行
3. **Echo**: CS → SW に `ACTION_COMPLETED` 送信
4. **Loop**: SW が受信して次のステップへ

## メリット
- `chrome.alarms`の最短1分制約を回避（秒単位ループ可能）
- ポップアップから独立、タブが開いている限り継続
- 重量級処理をEcho受信タイミングで実行

```typescript
chrome.runtime.onMessage.addListener((message) => {
  if (message.type === "ACTION_DONE" && state.active) {
    doHeavyWork();
    sendPingToContentScript();
  }
});
```
