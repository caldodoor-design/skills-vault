---
name: detecting-content-script-readiness
description: PING-PONGによるContent Script準備完了検出と2段階待機の起動シーケンス
---
# Content Script準備確認（PING方式）

## パターン
```typescript
// PING（即座に判定、retryCount:0）
export async function isContentScriptReady(tabId: number, frameId = 0): Promise<boolean> {
  const result = await sendMessageToTab(tabId, { type: "PING" }, { frameId, retryCount: 0 });
  return result.success;
}

// Content Script側PONG（トップフレームのみ応答）
chrome.runtime.onMessage.addListener((msg, _sender, sendResponse) => {
  if (msg.type === "PING") {
    if (window.top !== window.self) return false; // iframe無視
    sendResponse({ success: true });
    return false; // 同期レスポンス
  }
  return false;
});
```

## 2段階待機
```typescript
async function startCapture(tabId: number): Promise<MessageResponse> {
  if (!await isContentScriptReady(tabId, 0)) {
    await delay(1000); // CS インジェクション待ち（通常数百ms）
    if (!await isContentScriptReady(tabId, 0)) {
      return { success: false, error: "Content script not available." };
    }
  }
  // 処理開始...
}
```

## ルール
- PINGは`retryCount: 0`（呼び出し元の待機と二重にならない）
- トップフレームのみ応答（iframe応答で誤判定を防ぐ）
- 2段階とも失敗→ユーザーにわかりやすいエラー
- `executeScript`成功≠メッセージリスナー登録完了（タイムラグあり）
