---
name: detecting-content-script-readiness
description: PING-PONGメッセージによるContent Scriptの準備完了検出と、2段階待機によるグレースフルな起動シーケンス。
trigger: Background ScriptからContent Scriptへの通信開始前に、Content Scriptがロード済みか確認する必要があるとき。
---

# PING方式によるContent Script準備確認

## Purpose

Chrome拡張のBackground ScriptがContent Scriptにメッセージを送る前に、Content Scriptが実際にロードされ応答可能な状態かを確認する。Content Script未準備のまま処理を開始すると、全てのメッセージが失敗する。

## Context

以下の状況でContent Scriptが未準備になる:
1. 拡張インストール直後（タブ再読込前）
2. ページ遷移直後（Content Scriptのインジェクションが完了していない）
3. chrome.scripting.executeScriptが失敗した場合

単純に「メッセージを送って失敗したらエラー」ではユーザー体験が悪い。1秒待つだけで解決するケースが多いため、2段階の待機を設ける。

## Pattern

### 1. PING関数（即座に結果を返す）

```typescript
// lib/messaging.ts
export async function isContentScriptReady(
  tabId: number,
  frameId: number = 0
): Promise<boolean> {
  const result = await sendMessageToTab(
    tabId,
    { type: "PING" },
    { frameId, retryCount: 0 }  // リトライなし = 即座に判定
  );
  return result.success;
}
```

**なぜ`retryCount: 0`か**: PINGの目的は「今この瞬間に準備できているか」を知ること。リトライすると判定が遅れ、呼び出し元の待機ロジックと二重待機になる。

### 2. Content Script側のPONG応答

```typescript
// content/index.ts
const IS_TOP_FRAME = window.top === window.self;

chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  if (message.type === "PING") {
    if (!IS_TOP_FRAME) {
      return false; // iframeではPINGを無視
    }
    sendResponse({ success: true }); // PONG
    return false; // 同期レスポンス（非同期不要）
  }
  return false;
});
```

**なぜトップフレームのみか**: iframeのContent Scriptが応答すると、トップフレーム未準備なのに「準備完了」と誤判定する。

### 3. 2段階待機による起動シーケンス

```typescript
// background/index.ts
const TOP_FRAME_ID = 0;

async function startCapture(tabId: number): Promise<MessageResponse> {
  // 第1段階: 即座にチェック
  console.log("[Background] Checking content script readiness...");
  const isReady = await isContentScriptReady(tabId, TOP_FRAME_ID);

  if (!isReady) {
    // 第2段階: 1秒待ってから再チェック
    console.warn("[Background] Content script not ready, waiting...");
    await delay(1000);
    const isReadyAfterWait = await isContentScriptReady(tabId, TOP_FRAME_ID);

    if (!isReadyAfterWait) {
      // 2回失敗 → ユーザーにエラーを返す
      return {
        success: false,
        error: "Content script not available. Please ensure you are on a Kindle Cloud Reader page.",
      };
    }
  }

  // Content Script準備完了 → 以降の処理を開始
  // ...
}
```

### シーケンス図

```
Background                Content Script
    |                          |
    |--- PING (retryCount:0) ->|
    |                          |
    |<-- { success: true } ----|  (準備完了)
    |                          |
    |  → 処理開始              |

    --- または ---

    |--- PING (retryCount:0) ->|
    |                          |  (未ロード)
    |<-- error ----------------|
    |                          |
    |  [1秒待機]               |
    |                          |  (ロード完了)
    |--- PING (retryCount:0) ->|
    |<-- { success: true } ----|
    |                          |
    |  → 処理開始              |
```

## Rules

- PINGは`retryCount: 0`で送る。リトライは呼び出し元の責務。
- Content ScriptはPINGに対して`IS_TOP_FRAME`チェックを行い、トップフレームのみ応答する。
- 2段階待機の間隔は1秒。Content Scriptのインジェクションは通常数百msで完了するため、1秒あれば十分。
- 2段階とも失敗した場合は、ユーザーに分かりやすいエラーメッセージを返す（技術的な内容ではなく「正しいページにいるか確認してください」）。

## Anti-Patterns

- PINGにリトライを付ける（呼び出し元の待機と二重になり、最悪3回×500ms×2 = 3秒の無駄な待機が発生）
- Content Scriptの準備確認をスキップして直接メッセージを送る（初回メッセージが失敗し、そのメッセージの内容が失われる）
- `chrome.scripting.executeScript`の成功をもって準備完了と判断する（スクリプト注入とメッセージリスナー登録は非同期で、タイムラグがある）
- iframeのContent ScriptにもPONGさせる（トップフレーム未準備の誤判定を招く）

## Notes

- PINGメッセージは最も軽量なメッセージ。ペイロードなし、同期レスポンス、副作用なし。
- この2段階パターンは「楽観的チェック → 猶予期間 → 確定判定」という汎用パターン。ヘルスチェック全般に応用可能。
- Content Scriptが`return false`で同期レスポンスするのはChrome拡張の仕様。`return true`にすると非同期レスポンスモードになり、タイムアウトの扱いが変わる。
