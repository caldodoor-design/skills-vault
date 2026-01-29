---
name: building-chrome-ext-mvp
description: Chrome拡張機能（MV3）の堅牢なプロジェクト構成とService Worker/Messaging/ログの実装パターンを定義する。Chrome拡張の新規構築やアーキテクチャ設計時に使う。
skills: [handling-chrome-extension-messaging, implementing-chrome-ping-pong]
---

# Chrome拡張(MV3) 堅牢な構成テンプレート

## 目的

Chrome拡張機能（Manifest V3）の堅牢なプロジェクト構成とベストプラクティスを定義する。

---

## ディレクトリ構成

```
src/
├── background/
│   └── service-worker.ts    # Service Worker エントリポイント
├── content/
│   └── index.ts             # Content Script エントリポイント
├── popup/
│   ├── popup.html
│   ├── popup.ts
│   └── popup.css
├── lib/
│   ├── messaging.ts         # メッセージング共通ライブラリ
│   └── storage.ts           # ストレージ操作ユーティリティ
└── manifest.json
```

---

## Background: Service Worker

### 基本原則

- **イベント駆動**: Service Workerは必要時のみ起動し、アイドル時は終了する
- **常駐不可**: `chrome.runtime.onStartup` や永続的な接続に依存しない設計とする
- **状態保持**: グローバル変数での状態保持は禁止。`chrome.storage` を使用すること

### 実装パターン

```typescript
// ❌ 禁止: グローバル変数での状態管理
let cachedData = {};

// ✅ 推奨: chrome.storage による状態管理
chrome.storage.local.get(['cachedData'], (result) => {
  const data = result.cachedData || {};
});
```

### ライフサイクル注意点

- Service Workerは30秒〜5分でアイドル終了する
- 長時間処理は `chrome.alarms` でスケジューリングすること
- `chrome.runtime.onInstalled` でのみ初期化処理を行う

---

## Messaging: Promiseベースラッパー

### 必須: sendMessageAsync の実装

全てのメッセージングは以下のPromiseベースラッパーを通じて行うこと。

```typescript
// lib/messaging.ts

export interface Message {
  type: string;
  payload?: unknown;
}

export interface MessageResponse<T = unknown> {
  success: boolean;
  data?: T;
  error?: string;
}

export function sendMessageAsync<T = unknown>(
  message: Message
): Promise<MessageResponse<T>> {
  return new Promise((resolve, reject) => {
    chrome.runtime.sendMessage(message, (response: MessageResponse<T>) => {
      if (chrome.runtime.lastError) {
        reject(new Error(chrome.runtime.lastError.message));
        return;
      }
      resolve(response);
    });
  });
}

export function sendTabMessageAsync<T = unknown>(
  tabId: number,
  message: Message
): Promise<MessageResponse<T>> {
  return new Promise((resolve, reject) => {
    chrome.tabs.sendMessage(tabId, message, (response: MessageResponse<T>) => {
      if (chrome.runtime.lastError) {
        reject(new Error(chrome.runtime.lastError.message));
        return;
      }
      resolve(response);
    });
  });
}
```

### 使用例

```typescript
// Content Script から Background へ
const response = await sendMessageAsync<{ url: string }>({
  type: 'GET_CURRENT_URL',
});

if (response.success) {
  console.log(response.data?.url);
}
```

---

## Debugging: ログ出力先定義

### 各コンテキストのログ確認方法

| コンテキスト | ログ出力先 | 確認方法 |
|-------------|-----------|---------|
| Background (Service Worker) | Service Worker コンソール | `chrome://extensions` → 拡張機能の「Service Worker」リンク |
| Content Script | ページのDevTools コンソール | 対象ページでF12 → Console |
| Popup | Popup専用DevTools | Popupを右クリック →「検証」 |

### 推奨: 統一ログフォーマット

```typescript
// lib/logger.ts

type LogLevel = 'DEBUG' | 'INFO' | 'WARN' | 'ERROR';

function log(level: LogLevel, context: string, message: string, data?: unknown): void {
  const timestamp = new Date().toISOString();
  const prefix = `[${timestamp}][${level}][${context}]`;

  if (data !== undefined) {
    console.log(prefix, message, data);
  } else {
    console.log(prefix, message);
  }
}

export const Logger = {
  debug: (ctx: string, msg: string, data?: unknown) => log('DEBUG', ctx, msg, data),
  info: (ctx: string, msg: string, data?: unknown) => log('INFO', ctx, msg, data),
  warn: (ctx: string, msg: string, data?: unknown) => log('WARN', ctx, msg, data),
  error: (ctx: string, msg: string, data?: unknown) => log('ERROR', ctx, msg, data),
};
```

---

## manifest.json テンプレート

```json
{
  "manifest_version": 3,
  "name": "Extension Name",
  "version": "1.0.0",
  "description": "Extension description",
  "permissions": [],
  "host_permissions": [],
  "background": {
    "service_worker": "src/background/service-worker.js",
    "type": "module"
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["src/content/index.js"],
      "run_at": "document_idle"
    }
  ],
  "action": {
    "default_popup": "src/popup/popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  }
}
```
