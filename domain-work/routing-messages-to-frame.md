---
name: routing-messages-to-frame
description: Chrome拡張でフレーム指定メッセージ送信と接続エラーのみ選択的リトライ
---
# フレーム指定メッセージルーティング

## コア実装
```typescript
const TOP_FRAME_ID = 0;
const DEFAULT_RETRY = 3, RETRY_DELAY = 500;

export async function sendMessageToTab<T, R>(
  tabId: number, message: Message<T>,
  { frameId = 0, retryCount = DEFAULT_RETRY, retryDelayMs = RETRY_DELAY } = {}
): Promise<MessageResponse<R>> {
  for (let attempt = 0; attempt <= retryCount; attempt++) {
    const result = await new Promise<MessageResponse<R>>((resolve) => {
      chrome.tabs.sendMessage(tabId, message, { frameId }, (response) => {
        if (chrome.runtime.lastError) { resolve({ success: false, error: chrome.runtime.lastError.message }); return; }
        resolve(response ?? { success: true });
      });
    });
    // 成功 or 接続エラー以外 → 即返却
    if (result.success || !result.error?.includes("Could not establish connection")) return result;
    if (attempt < retryCount) await new Promise(r => setTimeout(r, retryDelayMs));
  }
  return { success: false, error: "Could not establish connection after retries" };
}
```

## ルール
- `frameId=0` = トップフレーム（`TOP_FRAME_ID`定数で定義）
- **Selective Retry**: `"Could not establish connection"`のみリトライ。権限エラー等はリトライしない
- Content Script側も`window.top === window.self`で二重フィルタ
- `chrome.runtime.lastError`はcallback内で必ずチェック
- 500ms × 3回 = 最大1.5秒（CSロードは通常100-300ms）
