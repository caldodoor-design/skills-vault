---
name: navigating-pages-with-session-id
description: UUIDベースのセッションIDで非同期ループの陳腐化を検出し、Ping-Pong状態遷移とタイムアウトで安全にページ遷移を管理するパターン。
trigger: 非同期ループで状態が途中で変わる可能性があるとき、またはContent ScriptとService Workerの間でPing-Pongメッセージングが必要なとき
---

# Session IDパターンによる非同期ページ遷移の安全管理

## Purpose

キャプチャループはawaitを多数含む長時間の非同期処理。その途中でユーザーが「停止」→「再開始」すると、古いループと新しいループが同時に動作する。Session IDで各ループを一意に識別し、古いループ（stale iteration）を安全に停止させる。

## Context

- キャプチャループ：スクリーンショット撮影 → ページ遷移 → 待機 → 次のスクリーンショット... を繰り返す
- ループの各ステップにawaitがあり、その間に状態が変わる可能性がある
- 停止→再開始で2つのループが並行動作するとrace conditionが発生
- Content ScriptとService Worker間のメッセージングにも遅延がある

## Pattern

### Session ID生成と検証

```typescript
function generateSessionId(): string {
  return `${Date.now()}-${Math.random().toString(36).substring(2, 9)}`;
}

// キャプチャ開始時にセッションIDを生成
const sessionId = generateSessionId();
await setState({
  isCapturing: true,
  sessionId,
  // ...
});

// ループ開始（sessionIdを引数で渡す）
captureLoop(sessionId).catch((error) => {
  finishCapture("error", error.message);
});
```

### Stale Loop Iteration検出

```typescript
async function captureLoop(sessionId: string): Promise<void> {
  const state = getState();

  // ★ ループの冒頭で必ずセッションIDを検証
  if (state.sessionId !== sessionId) {
    console.log("[Background] Capture loop detected stale session, stopping");
    return; // 古いループは静かに終了
  }

  // ... スクリーンショット撮影（重い非同期処理）...

  // ★ 非同期処理の後に再検証（awaitの間に停止された可能性）
  const latestState = getState();
  if (!latestState.isCapturing || latestState.sessionId !== sessionId) {
    console.log("[Background] Capture stopped or session changed during processing");
    return;
  }

  // ... ページ遷移処理 ...

  // ★ delayの後にも再検証
  await delay(PAGE_TURN_DELAY_MS);
  const stateAfterDelay = getState();
  if (!stateAfterDelay.isCapturing || stateAfterDelay.sessionId !== sessionId) {
    return;
  }

  // 次のイテレーションへ（再帰ではなくPing-Pong経由）
}
```

**重要**: 全てのawaitの後にセッションIDを再検証する。検証ポイントが1つでも欠けると、停止後に古いループが1ステップ余分に実行されてしまう。

### Ping-Pong状態遷移フロー

```
Service Worker                    Content Script
     │                                  │
     │  TURN_PAGE ───────────────────►  │
     │  (awaitingPageTurn = true)        │
     │                                  │  キー入力実行
     │                                  │  スピナー表示検出
     │                                  │  スピナー消失検出
     │  ◄─────────────── PAGE_TURNED    │
     │  (awaitingPageTurn = false)       │  (pageIndex付き)
     │                                  │
     │  captureLoop(sessionId)続行       │
```

### awaitingPageTurnフラグによる重複防止

```typescript
async function handlePageTurned(payload?: { pageIndex?: number }): Promise<void> {
  await stateReady;
  const state = getState();

  // Guard 1: キャプチャ中でない
  if (!state.isCapturing || !state.sessionId) return;

  // Guard 2: PAGE_TURNEDを待っていない（重複メッセージの除外）
  if (!state.awaitingPageTurn) return;

  // Guard 3: expectedPageとの一致確認（順序保証）
  if (payload?.pageIndex !== undefined && payload.pageIndex !== state.expectedPage) {
    return; // 期待していないページ番号→無視
  }

  await setState({ awaitingPageTurn: false });
  await delay(CAPTURE_DELAY_MS); // ページ安定化待ち

  // セッション再確認してからループ続行
  const latestState = getState();
  if (!latestState.isCapturing || latestState.sessionId !== state.sessionId) return;

  await captureLoop(state.sessionId);
}
```

### 二段タイムアウト管理

```
TURN_PAGE送信
  │
  ├── PAGE_TURN_TIMEOUT (30秒)
  │   Content Scriptがメッセージに応答しない → エラー終了
  │
  └── Content Scriptが応答（ページ遷移開始）
      │
      └── PAGE_TURNED_TIMEOUT (60秒)
          Content ScriptからPAGE_TURNEDが返ってこない → エラー終了
```

```typescript
// TURN_PAGE送信時：Content Scriptの応答タイムアウト
const response = await withTimeout(
  sendMessageToTab(tabId, { type: "TURN_PAGE" }, { frameId: TOP_FRAME_ID }),
  PAGE_TURN_TIMEOUT_MS,      // 30秒
  "Page turn timed out - content script did not respond"
);

// PAGE_TURNED待機タイムアウト（別管理）
function startPageTurnedTimeout(sessionId: string): void {
  clearPageTurnedTimeout();
  pageTurnedTimeoutId = setTimeout(async () => {
    const state = getState();
    // セッションIDとawaitingPageTurnを検証してから終了
    if (state.isCapturing && state.sessionId === sessionId && state.awaitingPageTurn) {
      await finishCapture("error", "PAGE_TURNED timeout");
    }
  }, PAGE_TURNED_TIMEOUT_MS);   // 60秒
}
```

**なぜ二段か**: TURN_PAGEへの応答（「ページ遷移を開始した」）と、PAGE_TURNED（「ページ遷移が完了した」）は別のイベント。前者は数秒だが、後者はネットワーク状況によって数十秒かかる場合がある。

## Rules

- 全てのawaitポイントの後にsessionIdを再検証すること
- staleなループは`return`で静かに終了させること（エラーではない）
- awaitingPageTurnフラグで重複PAGE_TURNEDメッセージを除外すること
- expectedPageで順序の整合性を検証すること
- タイムアウトのcallback内でもsessionIdとisCapturingを検証すること

## Anti-Patterns

- **sessionIdチェックをループ冒頭だけに置く**: awaitの間に状態が変わる
- **staleなループでfinishCapture()を呼ぶ**: 新しいセッションを巻き込んで終了してしまう
- **awaitingPageTurnフラグなしでPAGE_TURNEDを処理**: 重複メッセージでループが2回進む
- **タイムアウトを1段だけにする**: 「応答あり・完了なし」の状態を検出できない
- **clearTimeout忘れ**: 正常完了後にタイムアウトが発火してエラー終了する

## Notes

- `generateSessionId()`はDate.now() + ランダム文字列。UUIDv4ほどの一意性は不要（同一Service Worker内での識別のみ）
- PAGE_TURN_TIMEOUT_MS (30秒) はContent Scriptの応答時間。PAGE_TURNED_TIMEOUT_MS (60秒) はページ描画完了の時間。Kindleの画像ロード速度に依存する
- Ping-Pong方式はContent Scriptが「スピナーの表示→消失」を検出してPAGE_TURNEDを送る設計。単純なdelay待ちでは不安定
- MAX_CONSECUTIVE_FAILURES (5回) で連続失敗時にループを強制終了する安全弁もある
