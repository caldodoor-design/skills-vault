---
name: wrapping-callback-to-promise
description: Chrome APIのcallback→Promiseラッパー。Generic型で型安全、chrome.runtime.lastErrorハンドリング付き。
---
# Callback→Promise変換

## 基本パターン
```typescript
async function sendDebuggerCommand<T>(tabId: number, method: string, params?: Record<string, unknown>): Promise<T> {
  return new Promise((resolve, reject) => {
    chrome.debugger.sendCommand({ tabId }, method, params, (result) => {
      if (chrome.runtime.lastError) { reject(new Error(chrome.runtime.lastError.message)); return; }
      resolve(result as T);
    });
  });
}

// 使用例
const result = await sendDebuggerCommand<{ data: string }>(tabId, "Page.captureScreenshot", { format: "png" });
```

## ルール
- callback内で`chrome.runtime.lastError`を必ずチェック（未チェック→コンソール警告）
- reject後に即`return`（resolve/reject二重呼び出し防止）
- Generic `<T>` で戻り値の型を呼び出し側で明示
- callback未呼び出しリスクがある場合は`withTimeout()`と組み合わせ

## 適用対象
- `chrome.debugger.sendCommand`（Promise版なし）
- `chrome.tabs.sendMessage`（frameId指定時）
- その他callback形式のchrome.* API
