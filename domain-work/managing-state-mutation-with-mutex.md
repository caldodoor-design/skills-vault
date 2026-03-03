---
name: managing-state-mutation-with-mutex
description: Promiseチェーン式Mutexでchrome.storage.localの競合を防ぐ。インメモリ+永続化の二層状態管理。
---
# Promiseチェーン式Mutex状態管理

## パターン
```typescript
let memoryState: CaptureState = { isCapturing: false, tabId: null };
let stateUpdatePromise: Promise<void> = Promise.resolve();

function getState(): CaptureState { return { ...memoryState }; } // コピーを返す

async function setState(state: Partial<CaptureState>): Promise<void> {
  stateUpdatePromise = stateUpdatePromise
    .then(async () => {
      memoryState = { ...memoryState, ...state };
      await chrome.storage.local.set({ [KEY]: memoryState });
    })
    .catch((e) => console.error("Persistence failed:", e)); // catch必須→デッドロック防止
  await stateUpdatePromise;
}
```

## 起動時復元
```typescript
const stateReady = new Promise<void>((resolve) => { stateReadyResolve = resolve; });
async function initializeState() {
  const result = await chrome.storage.local.get(KEY);
  if (result[KEY]) memoryState = result[KEY];
  stateReadyResolve();
}
// イベントリスナー冒頭で `await stateReady;`
```

## ルール
- `getState()`はスプレッドでコピーを返す（外部変更防止）
- Promiseチェーンに`.catch()`必須（1回のrejectでチェーン全体が死ぬ）
- `await stateReady`で初期化完了を待つ
- storage永続化失敗でもインメモリは更新済み→処理続行
- SW停止に備えてstorage永続化は必須（30秒〜5分でアイドル停止）
