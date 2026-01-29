---
name: managing-state-mutation-with-mutex
description: Chrome拡張Service WorkerでPromiseチェーン式Mutexを用いてchrome.storage.localへの競合状態を防ぐパターン。インメモリ状態と永続化の二層管理を含む。
trigger: chrome.storage.localへの読み書きが複数箇所から非同期に行われるとき、または Service Worker で状態管理が必要なとき
---

# Promiseチェーン式Mutexによる状態管理

## Purpose

Chrome拡張のService Workerでは、複数のイベントリスナー（onMessage, onDetach等）が同時に状態を読み書きする。chrome.storage.localは非同期APIであり、Read-Modify-Writeの間に別のリスナーが割り込むとrace conditionが発生する。このSkillはPromiseチェーンによるシリアライゼーションでこれを防ぐ。

## Context

- chrome.storage.localへの書き込みが複数箇所から非同期に行われる
- Service Workerは任意のタイミングで休止・再起動されるため、インメモリ状態だけでは不十分
- Web Workerでは`SharedArrayBuffer`+`Atomics`が使えるがService Workerでは制約がある
- `navigator.locks`も選択肢だがPromiseチェーンのほうがシンプルで十分

## Pattern

### Dual-layer State Management（インメモリ + storage永続化）

```typescript
// 1. インメモリ状態：同期アクセス用
let memoryState: CaptureState = {
  isCapturing: false,
  tabId: null,
  // ...
};

// 2. MutexとなるPromiseチェーン
let stateUpdatePromise: Promise<void> = Promise.resolve();

// 3. 状態読み取り：常にインメモリから同期的に返す
function getState(): CaptureState {
  return { ...memoryState }; // スプレッドでコピーを返す（外部からの変更を防ぐ）
}

// 4. 状態書き込み：Promiseチェーンでシリアライズ
async function setState(state: Partial<CaptureState>): Promise<void> {
  stateUpdatePromise = stateUpdatePromise
    .then(async () => {
      // インメモリ状態を先に更新（即座に反映）
      memoryState = { ...memoryState, ...state };
      // storage永続化（Service Worker再起動時の復旧用）
      await chrome.storage.local.set({ [STORAGE_KEY]: memoryState });
    })
    .catch((error) => {
      console.error("[Background] State persistence failed:", error);
      // インメモリ状態は更新済み → 永続化失敗でもデッドロックしない
    });
  await stateUpdatePromise;
}
```

### 起動時の状態復元

```typescript
let stateReadyResolve: () => void;
const stateReady: Promise<void> = new Promise((resolve) => {
  stateReadyResolve = resolve;
});

async function initializeState(): Promise<void> {
  try {
    const result = await chrome.storage.local.get(STORAGE_KEY);
    if (result[STORAGE_KEY]) {
      memoryState = result[STORAGE_KEY];
    }
  } catch (error) {
    console.error("[Background] Failed to load state from storage:", error);
  }
  stateReadyResolve(); // イベントリスナーにstate準備完了を通知
}

// イベントリスナー側：状態初期化を待ってから処理
async function handleSomeEvent(): Promise<void> {
  await stateReady; // ← 初期化完了まで待機
  const state = getState();
  // ...
}
```

### なぜこのパターンか

1. **Promiseチェーンがロックの代わり**: `stateUpdatePromise = stateUpdatePromise.then(...)` により、前のthenが完了するまで次のthenは実行されない。これが自然なFIFOキューになる
2. **インメモリ優先**: getState()が同期的に返せるため、読み取り側はawait不要
3. **catch必須**: もしcatchがないと、一度rejectされたPromiseチェーンは以降全てのthenをスキップする → **デッドロック**

## Rules

- `getState()`は必ずスプレッド構文でコピーを返すこと（外部からの直接変更を防ぐ）
- `setState()`のPromiseチェーンには必ず`.catch()`を付けること（デッドロック防止）
- イベントリスナーの冒頭で`await stateReady`を呼ぶこと（初期化前のアクセス防止）
- storage永続化が失敗してもインメモリ状態は更新済みとし、処理を継続すること

## Anti-Patterns

- **chrome.storage.localを直接Read-Modify-Write**: getしてからsetするまでの間に別のリスナーが割り込む
- **Promiseチェーンのcatchを省略**: 一度の永続化エラーでチェーン全体が死ぬ
- **getState()で参照を返す**: `return memoryState` は外部から直接変更される危険がある
- **resetState()でsetState()を使う**: 全フィールドリセットの場合はPromiseチェーンに直接接続してアトミックに行う

## Notes

- Service Workerは30秒〜5分のアイドルで停止されるため、storage永続化は必須
- `navigator.locks` Web APIも代替手段だが、Service Worker環境では挙動が不安定な報告がある
- このパターンは単一プロセス（1つのService Worker）を前提としている。複数コンテキストからの同時アクセスには`navigator.locks`が適切
