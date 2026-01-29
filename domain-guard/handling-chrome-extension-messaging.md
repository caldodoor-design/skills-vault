---
name: handling-chrome-extension-messaging
description: Chrome拡張のPopup/Background/Content Script間メッセージ通信の型定義・送信ヘルパー・非同期応答パターンを定義する。拡張機能のメッセージング設計・実装時に使う。
skills: [building-chrome-ext-mvp, implementing-chrome-ping-pong]
---

# Chrome Extension Messaging Architecture

Chrome拡張機能における Popup / Background (Service Worker) / Content Script 間のメッセージ通信を設計・実装するための標準ガイドライン。

## 通信パターン (Communication Patterns)

| From | To | API | 用途 |
|------|----|-----|------|
| Popup | Background | `chrome.runtime.sendMessage` | コマンド発行（Start, Stop等） |
| Popup | Content | `chrome.tabs.sendMessage` | 直接指示（通常は非推奨） |
| Background | Content | `chrome.tabs.sendMessage` | ページ操作指示 |
| Content | Background | `chrome.runtime.sendMessage` | イベント通知、結果報告 |

## メッセージ型定義の推奨 (Type-Safe Messaging)

TypeScriptを使用する場合、すべてのメッセージを統一した型で管理することを推奨する。

```typescript
// lib/messaging.ts
export type MessageType =
  | "START_CAPTURE"
  | "STOP_CAPTURE"
  | "TURN_PAGE"
  | "PAGE_TURNED"
  | "PING"
  | "CAPTURE_COMPLETE"
  | "CAPTURE_ERROR";

export interface Message {
  type: MessageType;
  payload?: unknown;
}

export interface MessageResponse {
  success: boolean;
  error?: string;
  data?: unknown;
}
```

## 送信ヘルパー関数

### Background → Content Script

```typescript
export async function sendMessageToTab(
  tabId: number,
  message: Message,
  options?: { frameId?: number; retryCount?: number }
): Promise<MessageResponse> {
  const frameId = options?.frameId ?? 0; // 0 = Top frame
  try {
    const response = await chrome.tabs.sendMessage(tabId, message, { frameId });
    return response ?? { success: false, error: "No response" };
  } catch (error) {
    return { success: false, error: String(error) };
  }
}
```

### Content Script → Background

```typescript
export async function sendMessage(message: Message): Promise<MessageResponse> {
  return new Promise((resolve) => {
    chrome.runtime.sendMessage(message, (response) => {
      // chrome.runtime.lastError をチェック
      if (chrome.runtime.lastError) {
        resolve({ success: false, error: chrome.runtime.lastError.message });
        return;
      }
      resolve(response ?? { success: false, error: "No response" });
    });
  });
}
```

> **Note**: Promise APIは Manifest V3 および Chrome 99+ で利用可能です。それ以前の環境ではコールバックパターンを使用してください。

## Sender検証 (Sender Validation)

メッセージが信頼できる送信元からのものかを検証することで、セキュリティを向上させます。

```typescript
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  // 自身の拡張機能からのメッセージかを確認
  if (sender.id !== chrome.runtime.id) {
    console.warn("Rejected message from unknown extension");
    return false;
  }

  // 特定のタブからのメッセージのみ受け付ける場合
  if (!sender.tab || sender.frameId !== 0) {
    console.warn("Rejected message from non-top-frame");
    return false;
  }

  // ... 処理 ...
});
```


## 非同期応答のベストプラクティス

メッセージリスナー内で非同期処理を行う場合、**`return true`** を返してチャンネルを開いたままにする必要がある。

```typescript
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === "ASYNC_ACTION") {
    doAsyncWork().then((result) => {
      sendResponse({ success: true, data: result });
    });
    return true; // ← 重要: チャンネルを開いたままにする
  }
  return false;
});
```

## iframe対策

Content Scriptはiframe内でも実行される場合があるため、Top Frameでのみ処理したいメッセージは明示的にフィルタリングする。

```typescript
const IS_TOP_FRAME = window.top === window.self;

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (!IS_TOP_FRAME) {
    return false; // iframe内では無視
  }
  // ...
});
```

## 注意点

- **サイレントエラー**: 相手先のスクリプトがロードされていない場合、エラーなく失敗することがある。`PING` メッセージでreadiness（準備完了）を確認するのが堅牢。
- **sendResponse の呼び出し忘れ**: 非同期処理で `sendResponse` を呼び出さないと、送信側がタイムアウトまたはハングする。
