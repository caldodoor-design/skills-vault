---
name: building-chrome-ext-mvp
description: Chrome MV3拡張のプロジェクト構成とService Worker/Messaging/ログパターン
skills: [handling-chrome-extension-messaging, implementing-chrome-ping-pong]
---
# Chrome拡張(MV3) 構成テンプレート

## ディレクトリ構成
```
src/
├── background/service-worker.ts
├── content/index.ts
├── popup/ (popup.html, popup.ts, popup.css)
├── lib/ (messaging.ts, storage.ts)
└── manifest.json
```

## Service Worker 基本原則
- **イベント駆動**: アイドル時は終了（30秒〜5分）
- **状態保持禁止**: グローバル変数NG → `chrome.storage` を使用
- 長時間処理は `chrome.alarms` でスケジューリング
- 初期化は `chrome.runtime.onInstalled` でのみ

## Messaging: Promiseベースラッパー
```typescript
export interface Message { type: string; payload?: unknown; }
export interface MessageResponse<T = unknown> { success: boolean; data?: T; error?: string; }

export function sendMessageAsync<T>(message: Message): Promise<MessageResponse<T>> {
  return new Promise((resolve, reject) => {
    chrome.runtime.sendMessage(message, (response: MessageResponse<T>) => {
      if (chrome.runtime.lastError) { reject(new Error(chrome.runtime.lastError.message)); return; }
      resolve(response);
    });
  });
}
```
`sendTabMessageAsync` も同パターン（`chrome.tabs.sendMessage`）。

## ログ出力先
| コンテキスト | 確認方法 |
|-------------|---------|
| Background | `chrome://extensions` → SW リンク |
| Content Script | 対象ページのDevTools Console |
| Popup | Popup右クリック →「検証」 |

統一フォーマット: `[timestamp][LEVEL][context] message`
