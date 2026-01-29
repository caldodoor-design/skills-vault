---
name: wrapping-callback-to-promise
description: Chrome APIのcallbackスタイルをPromiseでラップし、Generic型パラメータで型安全に扱うパターン。chrome.runtime.lastErrorのハンドリングを含む。
trigger: chrome.debugger.sendCommandやchrome.tabs.sendMessage等のcallback APIをasync/awaitで使いたいとき
---

# Callback→Promise変換パターン

## Purpose

Chrome拡張APIの多くはcallback形式（MV3でも一部）。これをPromiseでラップすることで、async/awaitの構文で扱え、try-catchによるエラーハンドリングが統一できる。TypeScriptのGeneric型パラメータを使って戻り値の型も安全に扱う。

## Context

- chrome.debugger.sendCommand はcallback形式のみ（Promise版なし）
- chrome.tabs.sendMessage はPromise版があるが、frameId指定時にcallbackが必要な場合がある
- chrome.runtime.lastError は callback内でチェックしないと「Unchecked runtime.lastError」警告が出る
- 複数のAPI呼び出しを順次実行する場合、callback地獄を避けたい

## Pattern

### 基本パターン：Generic型パラメータ付きラッパー

```typescript
async function sendDebuggerCommand<T>(
  tabId: number,
  method: string,
  params?: Record<string, unknown>
): Promise<T> {
  const target: DebuggerTarget = { tabId };

  return new Promise((resolve, reject) => {
    chrome.debugger.sendCommand(target, method, params, (result) => {
      // ★ 重要: chrome.runtime.lastError を必ずチェック
      if (chrome.runtime.lastError) {
        reject(new Error(chrome.runtime.lastError.message));
        return;
      }
      resolve(result as T);
    });
  });
}
```

### 使用例

```typescript
// 型パラメータで戻り値の型を指定
interface CaptureScreenshotResult {
  data: string; // Base64 encoded image
}

const result = await sendDebuggerCommand<CaptureScreenshotResult>(
  tabId,
  "Page.captureScreenshot",
  { format: "png", captureBeyondViewport: false }
);
// result.data は string 型として型安全にアクセスできる
```

### 別の適用例：chrome.tabs.sendMessage

```typescript
async function sendMessageToTab<T>(
  tabId: number,
  message: Message,
  options?: { frameId?: number }
): Promise<T> {
  return new Promise((resolve, reject) => {
    const callback = (response: T) => {
      if (chrome.runtime.lastError) {
        reject(new Error(chrome.runtime.lastError.message));
        return;
      }
      resolve(response);
    };

    if (options?.frameId !== undefined) {
      chrome.tabs.sendMessage(tabId, message, { frameId: options.frameId }, callback);
    } else {
      chrome.tabs.sendMessage(tabId, message, callback);
    }
  });
}
```

### なぜこのパターンか

1. **chrome.runtime.lastError**: callbackの中でしかチェックできない。Promiseのrejectに変換することでtry-catchで統一的に扱える
2. **Generic型**: `sendDebuggerCommand<T>` の `T` により、Chrome DevTools Protocolの各コマンドの戻り値型を呼び出し側で指定できる
3. **as T**: Chrome APIの戻り値は `object | undefined` 型であることが多く、キャストが必要。ラッパー内に閉じ込めることで呼び出し側のコードがクリーンになる

## Rules

- callback内で`chrome.runtime.lastError`を必ずチェックすること（未チェックだとコンソール警告が出る）
- lastErrorチェック後は即座にreturnすること（resolve/rejectの二重呼び出し防止）
- Generic型パラメータ `<T>` を使い、呼び出し側で戻り値の型を明示すること
- callbackが呼ばれない可能性がある場合は、withTimeout()と組み合わせること

## Anti-Patterns

- **chrome.runtime.lastErrorをチェックしない**: 「Unchecked runtime.lastError」警告が出続ける
- **rejectの後にreturnを書かない**: resolveとrejectが両方呼ばれる
- **any型で戻す**: 型安全性が失われ、呼び出し側でランタイムエラーになる
- **Promiseラッパーの外でtry-catchする**: callback内のエラーはPromiseのrejectで伝搬するため、ラッパー外のtry-catchでは捕捉できない

## Notes

- MV3のchrome.* APIの多くはPromise版が追加されているが、chrome.debugger.sendCommandなど一部は未対応
- Chrome DevTools Protocolのコマンド型定義は `@anthropic-ai/devtools-protocol` や `devtools-protocol` パッケージから取得可能
- このパターンはNode.jsの `util.promisify` に相当するが、chrome.runtime.lastErrorの処理が追加されている点が異なる
