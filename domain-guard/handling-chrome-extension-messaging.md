---
name: handling-chrome-extension-messaging
description: Chrome拡張のPopup/Background/Content Script間メッセージ通信パターン
skills: [building-chrome-ext-mvp, implementing-chrome-ping-pong]
---
# Chrome Extension Messaging

## 通信パターン
| From | To | API |
|------|----|-----|
| Popup → Background | `chrome.runtime.sendMessage` |
| Background → Content | `chrome.tabs.sendMessage` |
| Content → Background | `chrome.runtime.sendMessage` |

## 型定義

```typescript
export type MessageType = "START_CAPTURE" | "STOP_CAPTURE" | "TURN_PAGE" | "PAGE_TURNED" | "PING" | "CAPTURE_COMPLETE" | "CAPTURE_ERROR";
export interface Message { type: MessageType; payload?: unknown; }
export interface MessageResponse { success: boolean; error?: string; data?: unknown; }
```

## 送信ヘルパー

```typescript
// Background → Content
export async function sendMessageToTab(
  tabId: number, message: Message, options?: { frameId?: number }
): Promise<MessageResponse> {
  try {
    const response = await chrome.tabs.sendMessage(tabId, message, { frameId: options?.frameId ?? 0 });
    return response ?? { success: false, error: "No response" };
  } catch (error) { return { success: false, error: String(error) }; }
}
```

## 非同期応答: `return true` 必須

```typescript
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === "ASYNC_ACTION") {
    doAsyncWork().then(result => sendResponse({ success: true, data: result }));
    return true; // チャンネルを開いたままにする
  }
});
```

## iframe対策
```typescript
const IS_TOP_FRAME = window.top === window.self;
// Top Frameでのみ処理したいメッセージはIS_TOP_FRAMEでフィルタ
```

## 注意
- 相手スクリプト未ロード時はサイレント失敗 → `PING`でreadiness確認
- `sendResponse` 呼び忘れ → 送信側がハング
- Sender検証: `sender.id !== chrome.runtime.id` で拒否
