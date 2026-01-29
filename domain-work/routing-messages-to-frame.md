---
name: routing-messages-to-frame
description: Chrome拡張でフレーム指定メッセージを送信し、接続エラー時のみ選択的リトライを行うルーティングパターン。
trigger: Background ScriptからContent Script（特定フレーム）へメッセージを送信する処理を実装するとき。
---

# フレーム指定メッセージルーティング

## Purpose

Chrome拡張でタブにメッセージを送る際、iframeが存在すると全フレームのContent Scriptが受信してしまう。トップフレームのみに確実にメッセージを届け、かつContent Script未準備の一時的エラーにはリトライで対応する。

## Context

Kindle Cloud Readerのページにはiframeが存在する。Background Service WorkerからContent Scriptへメッセージを送ると:
- `frameId`未指定: 全フレームのContent Scriptが受信 → iframe内のスクリプトが誤動作
- Content Script未ロード時: `"Could not establish connection"` エラー → リトライで回復可能
- 権限エラー等: リトライしても無駄

## Pattern

### 1. フレーム指定メッセージ送信（Selective Retry）

```typescript
const DEFAULT_RETRY_COUNT = 3;
const DEFAULT_RETRY_DELAY_MS = 500;

export async function sendMessageToTab<T = unknown, R = unknown>(
  tabId: number,
  message: Message<T>,
  options: SendMessageToTabOptions = {}
): Promise<MessageResponse<R>> {
  const {
    frameId = 0, // デフォルトでトップフレーム
    retryCount = DEFAULT_RETRY_COUNT,
    retryDelayMs = DEFAULT_RETRY_DELAY_MS,
  } = options;

  const sendOptions: chrome.tabs.MessageSendOptions = {};
  if (frameId !== undefined) {
    sendOptions.frameId = frameId;
  }

  for (let attempt = 0; attempt <= retryCount; attempt++) {
    const result = await new Promise<MessageResponse<R>>((resolve) => {
      chrome.tabs.sendMessage(tabId, message, sendOptions, (response) => {
        if (chrome.runtime.lastError) {
          resolve({ success: false, error: chrome.runtime.lastError.message });
          return;
        }
        resolve(response ?? { success: true });
      });
    });

    // 成功、または接続エラー以外 → 即座に返す
    if (result.success || !result.error?.includes("Could not establish connection")) {
      return result;
    }

    // 接続エラーのみリトライ
    if (attempt < retryCount) {
      console.log(`[Messaging] Retry ${attempt + 1}/${retryCount} for tab ${tabId}, frame ${frameId}`);
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs));
    }
  }

  return { success: false, error: "Could not establish connection after retries" };
}
```

### 2. Content Script側のフレームフィルタリング

```typescript
// content/index.ts
const IS_TOP_FRAME = window.top === window.self;

chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  if (message.type === "PING") {
    if (!IS_TOP_FRAME) {
      return false; // iframeではPINGを無視
    }
    sendResponse({ success: true });
    return false;
  }
  return false;
});
```

### 3. 呼び出し側での使用例

```typescript
const TOP_FRAME_ID = 0;

// 通常のメッセージ送信（リトライあり）
const response = await sendMessageToTab(
  tabId,
  { type: "NOTIFY_COMPLETE", payload: { imageCount } },
  { frameId: TOP_FRAME_ID }
);

// PING送信（リトライなし = 即座に結果を知りたい）
const result = await sendMessageToTab(
  tabId,
  { type: "PING" },
  { frameId: TOP_FRAME_ID, retryCount: 0 }
);
```

## Rules

- `frameId = 0`はChrome APIでトップフレームを意味する。この定数を`TOP_FRAME_ID`として定義し、マジックナンバーを避ける。
- リトライは`"Could not establish connection"`エラーのみ。権限エラーや無効タブエラーはリトライしない（Selective Retry）。
- Content Script側でも`IS_TOP_FRAME`チェックを行い、**送信側と受信側の両方**でフレームフィルタリングする（二重防御）。
- `chrome.runtime.lastError`はコールバック内で必ずチェックする。チェックしないとChromeがコンソールにエラーを出す。

## Anti-Patterns

- `frameId`を省略してブロードキャスト送信する（全フレームのContent Scriptが反応してしまう）
- 全エラーに対してリトライする（権限エラーに3回リトライしても時間の無駄）
- Content Script側でフレームチェックを省略する（`frameId`指定が効かないEdgeケースに対応できない）
- `chrome.runtime.lastError`をチェックせずにレスポンスを使う（undefinedアクセスでクラッシュ）

## Notes

- `frameId = 0`（トップフレーム）はChromeの仕様。サブフレームは1以上の動的IDが割り振られる。
- `chrome.tabs.sendMessage`のコールバックパターンをPromiseでラップしている。`sendResponse`ではなく`resolve`で結果を返す設計。
- `retryDelayMs`は固定500ms。Content Scriptのロードは通常100-300msで完了するため、500ms × 3回 = 最大1.5秒の待機で十分。
